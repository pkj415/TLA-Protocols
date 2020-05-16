---
layout: default
title: Steps in the spec
parent: TLA+ for Kopia's GC
nav_order: 8
---

Let's know discuss the final peice, the steps we need to have in the specification -

1. **TickTimeForward** - just increment the logical\_timestamp. This step is always enabled i.e., has no preconditions.
2. **For snapshot processing** -
   1. **TriggerSnapshot** -
	   * Precondition - cardinality of snapshot\_processes\_bag < NumSnapshots
	   * Action - add new snapshot process record to the bag of snapshot processes. Populate content\_ids\_to\_be\_written\_list with a permutation of a subset of possbile content IDs (i.e., 0 to NumContentIDs-1). Populate local\_index\_entries\_set based on index\_blobs\_list. Set timestamp as logical_timestamp.
   2. **PackContents** -
      * Precondition - snapshot hasn't yet packed/ written all contents it was supposed to write (i.e., pointer\_to\_next\_content\_to\_pack < length of content\_ids\_to\_be\_written\_list)
      * Action - choose some contents from content\_ids\_to\_be\_written\_list (starting at pointer\_to\_next\_content\_to\_pack) to be packed in the same timestamp (and which are not in local\_index\_entries\_set as an undeleted latest entry) and append them to local\_data\_blob\_record.
   3. **FlushLocalDataBlob** -
      * Precondition - local\_data\_blob\_list is not empty
      * Action - append local\_data\_blob\_list to data\_blobs\_list and append corresponding index entries to local\_index\_blob\_list
   4. **FlushLocalIndexBlob** -
      * Precondition - local\_index\_blob\_list is not empty
      * Action - append the local\_index\_blob\_list blob to the global index\_blob\_list if the local\_data\_blob\_list is empty (so that index blob is flushed only for all contents already flushed to the repository). Add entries in local\_index\_blob\_list to local\_index\_entries\_set.
   5. **MarkComplete** -
      * Precondition - all contents have been written with the timelimit (i.e., pointer\_to\_next\_content\_to\_pack = length of content\_ids\_to\_be\_written\_list and local\_data\_blob\_list is empty and logical_timestamp < start_timestamp + MaxSnapshotTime)
      * Action - mark the status of the snapshot as complete
   6. **DeleteSnapshot** -
      * Precondition - snapshot is in complete status
      * Action - change status to deleted
3. **For garbage collection processing** -
	1. **TriggerGC** -
	   * Precondition - cardinality of gc\_processes\_bag < NumGCs
	   * Action - add a new gc process record to the global bag. Populate local\_snapshot\_entries\_bag with snapshot\_processes\_bag. Populate local\_index\_entries\_set based on entries in index\_blobs\_list.
   2. **MarkContentDeleted** -
      * Precondition - based on the local copy of snapshots and index blobs, the GC knows which content IDs are to be marked deleted. If there are content IDs which haven't been marked deleted, but need to be deleted, the precondition is True.
      * Action - Mark deleted some contents which are yet to be deleted. This involves adding the delete index entries to the local\_index\_blob\_list. Also add the entries to local\_index\_entries\_set.
   3. **FlushLocalIndexBlob** -
      * Precondition - local\_index\_blob\_list is not empty.
      * Action - append local\_index\_blob\_list to index\_blobs\_list and reset local\_index\_blob\_list to empty list.
4. **For index blob compaction processing** -
	1. **TriggerIndexBlobCompcation** -
	   * Precondition - cardinality of index\_blob\_compaction\_processes\_bag < NumIndexBlobCompactions
	   * Action - add an index compaction process record to the global bag. Populate local\_index\_blobs\_list with a copy of index\_blobs\_list. Set new\_index\_written to False.
	2. **FlushMergedIndexBlob** -
	   * Precondition - new\_index\_written is False.
	   * Action - Flush the new index based on compaction (it is easy to compute which index blobs are to be compacted, skipping the details ehre). Set new\_index\_written to True. Populate index\_blobs\_ids\_to\_be\_deleted\_set based on the compaction process.
	3. **DeleteOldIndexBlob** -
	   * Precondition - new\_index\_written is True and there exists an index blob in index\_blobs\_ids\_to\_be\_deleted\_set.
	   * Action - Remove such a blob from index\_blobs\_list and remove the index blob id from index\_blobs\_ids\_to\_be\_deleted\_set.
5. **For data blob compaction processing** -
   1. **TriggerDataBlobCompaction** -
      * Precondition - cardinality of data\_blob\_compaction\_processes\_bag < NumDataBlobCompactions
      * Action - add a data blob compaction process record to the global bag. Populate local\_index\_blob\_ids\_list with a copy of blob\_ids from index\_blobs\_list. Populate local\_index\_entries\_set based on index\_blobs\_list. Set data\_blobs\_written to False.
	2. **PackContents** -
		* Precondition - current\_blob\_content\_pointer hasn't reached the last content in the last data blob in local\_index\_blob\_ids\_list.
		* Action - add the content to the local\_data\_blob\_record. Add the data blob id to data\_blobs\_ids\_to\_be\_deleted\_set in case it is not already added.
	3. **FlushLocalDataBlob** - Similar to the snapshot process
	4. **FlushLocalIndexBlob** - Similar to the snapshot process. If it was the last index to be flushed, mark data\_blobs\_written as True.
	5. **DeleteDataBlob** -
	   * Precondition - data\_blobs\_ids\_to\_be\_deleted is not empty and data\_blobs\_written is True
	   * Action - delete a data blob from data\_blobs\_ids\_to\_be\_deleted\_set and remove from data\_blobs\_ids\_to\_be\_deleted\_set.

Note that we don't have a "failed" state for any snapshot process. In the actual implementation, any snapshot that has not completed within its timelimit can stay in progress for anytime after the timelimit and still write contents, or it can fail. But we haven't explicitly specified a step to fail the snapshot because we will still captures behaviour where a snapshot in progress after the timelimit never takes a step to write any content and this is as good as a failed snapshot process.
