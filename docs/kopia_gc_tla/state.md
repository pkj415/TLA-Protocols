---
layout: default
title: State variables
parent: TLA+ for Kopia's GC
nav_order: 6
---

To start off, we have the following state variables in the specification (every state variable name has the type as the suffix in case it is a set/bag/list/record). Bags and lists (also called sequences) can have duplicate elements.  Records are just a mapping from field names to values.

1. **data\_blobs\_list** - the list of data blobs in the repository. Each data blob in the list is a record having -
	* **data_blob\_id** - a blob ID (required so that any index entry can reference it)
	* **content\_ids\_list** - a list of content IDs (we represent only content IDs because the actual data is not relevant to the specification)


2. **index\_blobs\_list** - the list of index blobs in the repository. Each index blob in the list is -
   * **index\_blob\_id** - a blob ID (required so that an index compaction entry can keep track of which index blobs have been deleted and which are to be deleted)
	* **index\_entries\_list** - a list of index entry records with each record having - the content id, the deleted flag, timestamp and the location of the content (just the data blob id is enough as the offset doesn't matter to the specification i.e., if a data blob with the blob id exists, whatever be the offset, the content will be found. This is because the data blob either exists fully or not at all)

3. **snapshot\_processes\_bag** - a bag of snapshot creation processes. Each snapshot creation process has -
	* **status** - if it is "in_progress", "completed" or "deleted". Note that we don't separately have a state variable for snapshots. They are just represented by snapshot processes which have status "completed". This is done just for syntactic simplicity of the specification.
   * **content\_ids\_to\_be\_written\_list** - a list of content ids of contents that the snapshot process will write in the same order as this list and post which it can be marked complete.
   * **pointer\_to\_next\_content\_to\_pack** - an integer represeting the next content id to be packed by the snapshot.
   * **local\_index\_entries\_set** - a set populated from the point in time view of list of index blobs read at the start of snapshotting process (TriggerSnapshot step), later this is updated on every flush of index blob by the snapshot. Note that we use a set as the division and ordering of the content IDs in index blobs doens't matter.
   * **local\_data\_blob\_record** - has the same format as a data blob record (which was in data\_blobs\_list), only without the blob\_id. A blob_\id is assigned by the step in which the local data blob is being flushed.
   * **local\_index\_blob\_list** - a list of index entries representing the next index blob with contents that haven't been flushed to the global list of index_blobs; once flushed, this will be reset to empty set.
   * **start_timestamp** - start time of the snapshot creation process.

4. **gc\_processes\_bag** - a bag of GC processes. Each GC process has -
	* **local\_snapshot\_entries\_bag** - a bag representing point in time view of snapshots (i.e., completed snapshot processes) in the repository, populated when the GC process is triggered.
	* **local\_index\_blob\_list** - a list of index entries representing the local index blob of deletion entries that can be flushed to global list of index blobs. Once flushed this will be reset to an empty list.
	* **local\_index\_entries\_set** - a set populated from the point in time view of list of index blobs read at the start of GC process (TriggerGC step), later this is updated on every flush of index blob by the GC. Note that we use a set as the division and ordering of the content IDs in index blobs doens't matter to snapshot creation processes

5. **data\_blob\_compaction\_processes\_bag** - a bag of data blob compaction processes with each element having -
	* **local\_data\_blob\_ids\_list** - a point in time copy of the list of data blob IDs read when this process is triggered
	* **local\_index\_entries\_set** - a set populated from the point in time view of list of index blobs read at the startup, later this is updated on every flush of index blob by the process. Note that we use a set as the division and ordering of the content IDs in index blobs doens't matter.
	* **local\_data\_blob\_record** - has the same format as a data blob, just without the blob\_id. It represents the local data blob to which contents are packed waiting to be flushed in one go as a blob
	* **local\_index\_blob\_list** - a list of index entries representing the local index blob which is flushed periodicially similar to as done in a snapshot process
	* **current\_blob\_content\_pointer** - data blob compaction reads contents from data blobs in order and flushed new data and index blobs similar to snapshot creation. So, we need a pointer to denote which contents are already written either to the local data blob/ or flushed so that the step to write more contents knows where it left of earlier. The pointer will be a tuple of two values - the blob id in the local list of data blob IDs and the content id in the list of contents in that blob. A data blob in the repository is skipped from compaction if it has no contents which are dangling (i.e., not referenced by any index entry).
	* **data\_blobs\_ids\_to\_be\_deleted\_set** - a set of data blob IDs that have to be deleted once all merged data blobs have been written after a full sweep of the data blobs.

6. **index\_blob\_compaction\_processes\_bag** - a bag of index blob compaction processes with each element having -
   * **local\_index\_blobs\_list** - a point in time copy of the list of index blobs read when this process is triggered
   * **index\_blobs\_ids\_to\_be\_deleted\_set** - a set of index blob IDs that have to be deleted by this process.
   * **new_index_written** - a flag to indicate if the merged index has been written. If so, the process can go ahead to delete index blobs which were compacted.
   * **local\_index\_blob\_list** - a list of index entries representing the merged index blob. Index compaction results in a single flush of one large index blob and so doesn't need a pointer (as needed in data blob compaction process) to keep track of which index blobs it read and flushed as a new index blob so far from the local\_index\_blobs\_list.

7. **logical\_timestamp** - a monotonically increasing counter starting from 1
8. **next\_blob\_id** - a monotonically increasing counter which can be used by a snapshot/data blob compaction process to assign a blob id so that no two data blobs have the same blob id.
9. **next\_index\_id** - a monotonically increasing counter which can be used by a snapshot/index blob compaction process to assign a blob id so that no two index blobs have the same blob id.

A bag is used for the processes because there might be two processes of the same type running simulatenously with the same local state. For example, two snapshot processes running simulatneously might have started with the intention to write the same set of contents and might have as of some state in the behaviour written the same set of index & data blobs and have the same values for local state of the processes.
