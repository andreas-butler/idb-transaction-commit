# Explainer
Documentation & FAQ of IndexedDB databases enumeration function. 
**Please file an issue @ [https://github.com/w3c/IndexedDB/issues](#https://github.com/w3c/IndexedDB/issues) if you have any feedback :)**

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**
- [What?](#what)
- [Why?](#why)
- [How?](#how)
- [Potential Issues](#potential-issues)
- [Use Cases and Example Code](#use-cases-and-example-code)
  - [Writeonly Mode](#writeonly-mode)
  - [Page Lifecycle](#page-lifecycle)
  - [Best Practice](#best-practice)
- [Transaction commit consistency and guarantees](#transaction-commit-consistency-and-guarantees)
- [Spec changes](#spec-changes)
- [Future features](#future-features)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->
# What?
IndexedDB’ transaction.commit() functionality will provide an explicit API for requesting that an IndexedDB transaction be committed. 

At present an IndexedDB transaction is auto-committed when it is determined by script that it is no longer possible to kick the transaction from an inactive state to an active one. A transaction may transition from inactive to active only within the scope of any callbacks belonging to any requests made on the transaction, which is the same scope in which new requests may be made on the transaction. Thus, as long as callbacks from previous transaction requests themselves make new requests (for example, in the event that the results of a previous query are used to construct a secondary query), the lifetime of the transaction will be extended and the transaction will not be committed. 

Once all callbacks owned by a transaction’s requests resolve such that no new requests have been made on the transaction, then the transaction has arrived in a state whereby it is impossible for it to reattain ‘active’ status, and so the transaction transitions into the ‘commiting’ state and a signal is sent denoting that it is time to flush the results of the transaction to disk. This flow is illustrated below in Figure 1.

<p align="center">
<img src="https://github.com/andreas-butler/idb-transaction-commit/blob/master/pics/idb_autocommit_curr.png?raw=true" alt="hi" />
<br/>
Figure 1: An example control flow illustrating the transaction autocommit functionality.
</p>

Under the new architecture, the explicit commit() call is added to the transaction API. When this call is invoked on an active transaction, then the transaction will be forced into the ‘committing’ state, and thus no new requests will be permitted to be made on it (even in any callbacks belonging to previous requests). Attempts at making new requests on the transaction after commit() is called will throw a DOMException. We see the difference between the current autocommiting implementation and the new explicit commit() call in Figure 2 below.

<p align="center">
<img src="https://github.com/andreas-butler/idb-transaction-commit/blob/master/pics/idb_commit_new.png?raw=true" alt="hi" />
<br/>
Figure 2: How the previous control flow differs in the case of explicit commit().
</p>

# Why?
Under the previous architecture, before a transaction could be fully committed and data flushed to disk, it was necessary for script to verify that no request callbacks themselves made new requests on the transaction. This meant that flushing data to disk required waiting for all pending callbacks to resolve completely. 

IndexedDB’s transaction.commit() will allow developers the flexibility to announce the fact that they do not intend to make any new requests on a transaction object and that it can be committed as soon as possible. Under these conditions, the task of flushing data to disk does not have to wait until a signal from the front end declares that all callbacks associated with the transaction have resolved; it only has to wait until all transaction requests have been processed completely.

The primary benefit of this change is increasing the throughput of writing data to disk. The throughput increase is evident in comparing Figure 2 to Figure 1. In Figure 2 one sees that time required by the round trip of the response of the final request and the commit signal it ultimately returns upon completion of its callback is saved by the explicit commit().

This is particularly useful for the case of loading a database, when large amounts of data are being written to disk via many transactions with no intention of making any follow up queries on that data, and the case of writing data to disk under some time constraint. The latter situation is encountered, for example, when the browser informs a tab that is about to be killed (because it has been inactive for a period of time and will thus be killed to ease memory usage) and so the tab must save its state to disk as fast as possible so that it can be reloaded again in the event that a user navigates back to it.

A secondary benefit exists that is less concrete but nevertheless still important. By allowing developers to explicitly call commit() on transactions, the IndexedDB API begins to adopt the feel of a more conventional database API.

# How?

# Potential Issues
IndexedDB initially shipped solely with an autocommit functionality. Developers did not have the control to declare themselves finished with a transaction that the new explicit commit() API call affords them. It is thus reasonable to ask why the initial ship of indexedDB did not include an explicit commit() call and whether adding one could cause potential issues. The short answer to the first question is ‘simplicity’ and the short answer to the second question is ‘not really’.

IndexedDB shipped with autocommit initially to encourage short-lived transactions and to prevent bugs involving leaving uncommited transactions hanging around blocking future transactions from being opened on a database. Developers of indexedDB were concerned about web developers forgetting to call commit on transactions or holding transactions open too long, thus hindering the experience of web users. Because adding the explicit commit() call will not also entail removing autocommit, the autocommit feature will still exist to prevent dangling transactions and to close transactions as soon as they are no longer requestable by script.

The primary concern about adding commit() is developer confusion, specifically confusion regarding whether and when commit() should be called. Adding commit() to the API may obfuscate the fact that indexedDB still has an auto-commit functionality. Developers may incorrectly believe that they must call commit() on all transactions lest they leave them dangling in the database. While this is a rather benign misunderstanding as it encourages always calling commit() and thus always increasing the throughput of the final requests made on a database, it is not ideal for the behaviour of indexedDB to seem at all unintuitive or opaque.

In general there might be concern regarding the fact that the explicit commit() is strictly better than indexedDB’s autocommit functionality (increasing throughput at no expense to the developer, who themselves have the best understanding of when they are completely finished with a transaction). As a result, calling commit() commit at the end of every transaction may very well become best practice, indicating that the autocommit functionality is superfluous. The best argument for the continued existence of the autocommit functionality is that it serves as a failsafe for any event in which a developer does not call commit() explicitly, in which case the autocommit will commit the transaction and prevent it from dangling.

There have been questions regarding whether or not some procedure for resurfacing commit errors that occur after an explicit commit call is made will be included with commit(). Without such error reporting, there could be cases when commit() is used for saving tab state, say in the case of freezing, when errors may occur on requests against the transaction that are never reported back to the front end because the page is killed before information regarding the errors propagates back to script. In these situations it may be useful to let script know of such errors the next time the page is loaded, otherwise information is lost.

# Use Cases and Example Code

## Writeonly mode: Loading a lot of data into an indexedDB database
When a user initially creates an indexedDB database, they may desire to load a large amount of data into the database without any intention of making any secondary requests beyond this large ‘put’ operation. In this event, to ensure loading occurs as fast as possible, the developer can call commit() after issuing all their ‘puts’.

```javascript
let database_info_enumeration = await window.indexedDB.databases();
database_info_enumeration.forEach(function(info) {
  indexedDB.deleteDatabase(info.name);
});
```
## Page Lifecycle: Writing the state of a page to IndexedDB as fast as possible
The Page Lifecycle API is an API heavily involved in alleviating power and memory tolls on users running many web applications at once. Central to this task is efficiently tracking and managing pages as they transition in and out of active and inactive states.

The process of ‘freezing’ a tab involves recognizing when a tab has been inactive for a long period of time, marking it for ‘freezing’ (letting it know that the browser is getting ready to kill its process to save on memory), and then subsequently killing it. When the tab is informed that it is going to be killed, it has a limited window of time to save its state to disk so that in the event that a user renavigates to that tab, it can be allocated a new process that can be loaded back into the state that the previous process was in when it was killed.

Currently ‘freezing’ and other such functionalities under the umbrella of the ‘lifecycle’ API cannot reliably use indexedDB to save state because there is a risk that the transaction responsible for saving state to disk will not commit before the page is killed. This unreliability arises because the page must be alive after the ‘put’ requests return to the front end in order to issue the commit signal after all callbacks have resolved. Instead there is a pattern of developers saving state to localStorage. This unreliability is illustrated below in Figure 3.

<p align="center">
<img src="https://github.com/andreas-butler/idb-transaction-commit/blob/master/pics/idb_autocommit_save_state.png?raw=true" alt="hi" />
<br/>
Figure 3: An example of the unreliability of using the current autocommitting implementation of indexedDB for saving page state during the freezing process.
</p>

With the addition of commit() there is no longer a need for the page to still be alive after the ‘put’ requests are issued when saving state because, assuming commit() was appropriately called, the resolution of callbacks no longer mediates the flushing of data to disk, and instead it may be safely assumed that data can be saved with no performance implications. This fact is illustrated below in Figure 4.

<p align="center">
<img src="https://github.com/andreas-butler/idb-transaction-commit/blob/master/pics/idb_commit_state_Save.png?raw=true" alt="hi" />
<br/>
Figure 4: An example of how the above situation is made more reliable with the addition of the explicit commit() API call.
</p>

By providing a reliable commit() option for indexedDB in the case of saving page state, developers will no longer have to rely on saving to localStorage, which will have a positive impact on website performance and input latency.

```javascript
let database_info_enumeration = await window.indexedDB.databases();
let ui_list = [];
database_info_enumeration.forEach(function(info) {
  ui_list.push(info.name);
});
```

## Best Practice: It’s just a good idea
When a developer knows that they have made the last request on an open transaction, calling commit() is strictly beneficial to them. Their data will be written to disk faster and they will still receive responses from the final requests they have made against the transaction and so can still perform operations in script in reaction to the final requests being processed. Therefore whenever a developer is certain that a request is the final one they will make against a transaction, best practice will likely become that they call commit explicitly on the transaction.

# Transaction commit() consistency and guarantees
Transaction commit() 
## Guarantees
// talk about how even if the page crashes after the requests are issued, the stuff will still 
// commit? Or will there be a further issue because of the oncomplete not being adequately 
// received?
## Consistency
// talk about what happens in the event of a writing error (cache full) and how abort will be called?

# Spec changes
See [https://github.com/w3c/IndexedDB/pull/240/files#diff-ec9cfa5f3f35ec1f84feb2e59686c34dR2369](#https://github.com/w3c/IndexedDB/pull/240/files#diff-ec9cfa5f3f35ec1f84feb2e59686c34dR2369)

# Future Features

commit() 2
