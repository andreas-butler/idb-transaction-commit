# IDB Transaction Explicit Commit Explained
Documentation & FAQ of the IndexedDB transaction explicit commit() API. 
**Please file an issue @ [https://github.com/w3c/IndexedDB/issues](#https://github.com/w3c/IndexedDB/issues) if you have any feedback :)**

Last updated date: 10/10/2018

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**
- [What’s all this then?](#what’s-all-this-then)
  - [Goals](#goals)
  - [Non-goals](#non-goals)
- [Getting started and example code](#getting-started-and-example-code)
- [Key scenarios](#key-scenarios)
  - [Scenario 1: Loading a database](#scenario-1\:-loading-a-database) 
  - [Scenario 2: Page lifecycle](#scenario-2\:-page-lifecycle)
- [Use Cases and Example Code](#use-cases-and-example-code)
  - [Writeonly Mode](#writeonly-mode)
  - [Page Lifecycle](#page-lifecycle)
  - [Best Practice](#best-practice)
- [Potential Issues](#potential-issues)
  - [Why an explicit commit function was not initially shipped](#why-an-explicit-commit-function-was-not-initially-shipped)
  - [Potential developer confusion](#potential-developer-confusion)
  - [Possibility of obsolescing autocommit](#possibility-of-obsolescing-autocommit)
- [Spec changes](#spec-changes)
- [Considered alternatives](#considered-alternatives)
- [Future features](#future-features)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->
# What’s all this then?
IndexedDB’ transaction.commit() functionality will provide an explicit API for requesting that an IndexedDB transaction be committed. 

At present an IndexedDB transaction is auto-committed when it is determined by script that it is no longer possible to kick the transaction from an inactive state to an active one. A transaction may transition from inactive to active only within the scope of any callbacks belonging to any requests made on the transaction, which is the same scope in which new requests may be made on the transaction. Thus, as long as callbacks from previous transaction requests themselves make new requests (for example, in the event that the results of a previous query are used to construct a secondary query), the lifetime of the transaction will be extended and the transaction will not be committed. Otherwise, if all associated callbacks have resolved and there are no new requests against a transaction, script will detect this fact and autocommit the transaction, flushing any changes made by the transaction to a database to disk. This behaviour can be seen in Figure 1.

<p align="center">
<img src="https://github.com/andreas-butler/idb-transaction-commit/blob/master/pics/idb_autocommit_curr.png?raw=true" alt="hi" />
<br/>
Figure 1: An example control flow illustrating the transaction autocommit functionality.
</p>

With the addition of the explicit commit() call to the IDBTransaction API, the strict requirement for script alone to determine programmatically when a transaction is ready to be committed is removed. When the explicit commit() call is invoked on an active transaction, the transaction will be forced into the ‘committing’ state, and thus no new requests will be permitted to be made on it (even in any callbacks belonging to previous requests). Attempts at making new requests on the transaction after commit() is called will throw a DOMException. We see the difference between the current autocommiting implementation and the new explicit commit() call in Figure 2 below.

<p align="center">
<img src="https://github.com/andreas-butler/idb-transaction-commit/blob/master/pics/idb_commit_new.png?raw=true" alt="hi" />
<br/>
Figure 2: How the previous control flow differs in the case of explicit commit().
</p>

## Goals
The primary benefit of this change is increasing the throughput of writing data to disk.

Under the previous architecture, before a transaction could be fully committed and data flushed to disk, it was necessary for script to verify that no request callbacks themselves made new requests on the transaction. This meant that flushing data to disk required waiting for all pending callbacks to resolve completely. 

IndexedDB’s transaction.commit() will allow developers the flexibility to announce the fact that they do not intend to make any new requests on a transaction object and that it can be committed as soon as possible. Under these conditions, the task of flushing data to disk does not have to wait until a signal from the front end declares that all callbacks associated with the transaction have resolved; it only has to wait until all transaction requests have been processed completely by the database task queue. The fact that this final signal roundtrip between the database task queue and the page task queue is no longer necessary is evidenced by its lack in Figure 2 relative to Figure 1.

Increasing the throughput of writing data to disk for indexedDB is particularly useful for the case of loading a database, when large amounts of data are being written to disk via many transactions with no intention of making any follow up queries on that data, and the case of writing data to disk under some time constraint. The latter situation is encountered, for example, when the browser informs a tab that is about to be killed (because it has been inactive for a period of time and will thus be killed to ease memory usage) and so the tab must save its state to disk as fast as possible so that it can be reloaded again in the event that a user navigates back to it. This case is explained in greater detail in the section [Page Lifcycle](#page-lifecycle) below.

## Non-goals

# Getting started / example code
Using the IDBTransaction.commit() API call will be simple and intuitive and introduce very little additional code relative to what developers are already accustomed to writing. Developers will simply call commit() on any transaction that they know they are finished requesting and are ready to commit to the database.

```javascript
// We go through a simple example in which a transaction is made
// that puts some data into an object store, and then commits it.
let data = get_some_data(); // helper function elsewhere defined
let db;

// Connect to the database
let openRequest = indexedDB.open(['myDatabase']);
openRequest.onsuccess = function(event) {
  db = openRequest.result;
};

let txn = db.transaction(['myDatabase'], 'readwrite');
txn.onsuccess = function(event) {
  console.log("Successfully wrote data”);
}
txn.onerror = function(event) {
  console.log("Unsuccessfully wrote data”);
}

let objectStore = txn.objectStore('myObjectStore');
objectStore.put(data.key, data.value);

// Here we call the explicit commit.
txn.commit();
```

# Key scenarios

## Scenario 1: Loading a database
Oftentimes a user may desire to load a large amount of data into a database without any intention of making any secondary requests beyond this large ‘put’ operation. In this event, to ensure loading occurs as fast as possible, the developer can call commit() after issuing all their ‘puts’.

```javascript
// Here we collect a bunch of data to load into a database,
// using multiple transactions. Because commit() will speed up the
// flushing of each individual transaction, throughput increases
// dramatically.
let data_chunks = get_data_to_load(); // helper function elsewhere defined
let db;

// Connect to the database
let openRequest = indexedDB.open(['myDatabase']);
openRequest.onsuccess = function(event) {
  db = openRequest.result;
};

// Make all the transactions for the different data chunks
// and commit them.
data_chunks.forEach(async function(chunk) {
  let txn = db.transaction(['myDatabase'], 'readwrite');
  txn.onsuccess = function(event) {
    console.log("Successfully wrote chunk: " + chunk.num);
  }
  txn.onerror = function(event) {
    console.log("Unsuccessfully wrote chunk: " + chunk.num);
  }
  
  let objectStore = txn.objectStore('myDatabase');
  chunk.data.forEach(function(datum) {
    objectStore.put(datum.key, datum.value);
  });
  
  // Here we call the explicit commit.
  txn.commit();
});
```
## Scenario 2: Page Lifecycle
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
<img src="https://github.com/andreas-butler/idb-transaction-commit/blob/master/pics/idb_commit_save_state.png?raw=true" alt="hi" />
<br/>
Figure 4: An example of how the above situation is made more reliable with the addition of the explicit commit() API call.
</p>

By providing a reliable commit() option for indexedDB in the case of saving page state, developers will no longer have to rely on saving to localStorage, which will have a positive impact on website performance and input latency.

```javascript
function onShutdownSignal(event) {
  let pageState = getCurrentPageState(); // helper function elsewhere defined
  let db;

  // Connect to the database
  let openRequest = indexedDB.open(['pageStateDB']);
  openRequest.onsuccess = function(event) {
    db = openRequest.result;
  };

  // Make the transactions for saving state.
  let txn = db.transaction(['pageStateDB'], 'readwrite');
  txn.onsuccess = function(event) {
    console.log("Successfully wrote state.");
  }
  txn.onerror = function(event) {
    console.log("Unsuccessfully wrote state.");
  }

  // Define the puts for the transaction
  let objectStore = txn.objectStore('pageStateDB');
  pageState.forEach(function(pieceOfState) {
    objectStore.put(pieceOfState.key, pieceOfState.value);
  });

  // Here we call the explicit commit.
  txn.commit();
}
```

## Scenario 3: Best Practice
When a developer knows that they have made the last request on an open transaction, calling commit() is strictly beneficial to them. Their data will be written to disk faster and they will still receive responses from the final requests they have made against the transaction and so can still perform operations in script in reaction to the final requests being processed. Therefore whenever a developer is certain that a request is the final one they will make against a transaction, best practice will likely become that they call commit explicitly on the transaction.

```javascript
let txn;

// Do most anything with the transaction here.

// When the transaction is ready, commit it explicitly.
txn.commit()
```

# Potential Issues
IndexedDB initially shipped solely with an autocommit functionality. Developers did not have the control to declare themselves finished with a transaction that the new explicit commit() API call affords them. It is thus reasonable to ask why the initial ship of indexedDB did not include an explicit commit() call and whether adding one could cause potential issues. The short answer to the first question is ‘simplicity’ and the short answer to the second question is ‘not really’.

## Why an explicit commit function was not initially shipped
The first iteration of the indexedDB spec shipped with autocommit-only functionality to encourage short-lived transactions and to prevent bugs involving leaving uncommited transactions hanging around blocking future transactions from being opened on a database. Developers of indexedDB were concerned about web developers forgetting to call commit on transactions or holding transactions open too long, thus hindering the experience of web users. Because adding the explicit commit() call will not also entail removing autocommit, the autocommit feature will still exist to prevent dangling transactions and to close transactions as soon as they are no longer requestable by script.

## Potential developer confusion
The primary concern about adding commit() is developer confusion, specifically confusion regarding whether and when commit() should be called. Adding commit() to the API may obfuscate the fact that indexedDB still has an auto-commit functionality. Developers may incorrectly believe that they must call commit() on all transactions lest they leave them dangling in the database. While this is a rather benign misunderstanding as it encourages always calling commit() and thus always increasing the throughput of the final requests made on a database, it is not ideal for the behaviour of indexedDB to seem at all unintuitive or opaque.

## Possibility of obsolescing autocommit
In general there might be concern regarding the fact that the explicit commit() is strictly better than indexedDB’s autocommit functionality (increasing throughput at no expense to the developer, who themselves have the best understanding of when they are completely finished with a transaction). As a result, calling commit() commit at the end of every transaction may very well become best practice, indicating that the autocommit functionality is superfluous. The best argument for the continued existence of the autocommit functionality is that it serves as a failsafe for any event in which a developer does not call commit() explicitly, in which case the autocommit will commit the transaction and prevent it from dangling.

Obviously even if calling commit() always becomes best practice, autocommit will not likely be removed.

# Spec changes
See [https://github.com/w3c/IndexedDB/pull/242](https://github.com/w3c/IndexedDB/pull/242)

# Considered alternatives
## API call to ‘commit when ready’
There was discussion regarding implementing a commit method that would allow a developer to extend the life of a transaction as they saw fit. Such a scheme would allow for an easier method of doing asynchronous work upon which a transaction depended between initializing the transaction and committing it. At present it is difficult to have the results of asynchronous work affect a transaction because of how autocommitting detects when a transaction has no work left to do (essentially script would likely figure that a transaction could be committed while it was waiting for async work to be done because it would have no immediate requests open against it). Such an implementation of commit was put aside for the time being because of the potential complications it could cause if developers don’t carefully manage their open database transactions.

# Potential future features
## Resurfacing commit errors
In the future there may be additional support for resurfacing commit errors that may occur when a page has been shutdown in the middle of a transaction being committed and is thus not around to receive the errors from the database task queue. Such an implementation would involve saving any errors that occur during committing so that if the page is loaded again at another time it can be delivered the errors that occurred previously and handle them.

# References & acknowledgements
Thanks to Josh Bell for doing the spec work and helping explain commit().
Thanks to Daniel Murphy for his explanations as well.
Thanks to Shubhie Panicker and Philip Walton for their explanation of how commit() helps the Lifecycle API.
Thanks to 

