---
layout: default
title: Choose data structures of state variables wisely
parent: TLA+ for Kopia's GC
nav_order: 12
---

This is just a special case of avoiding semantically distinct states.

**Convert index\_blobs\_list to index\_entries\_set -** Since the split/ordering information of index entries in index blobs and the ordering information of index entries within blobs is never used now in any step (index compaction is gone! it would required the split information to delete index entries in blob units), we don't have to maintain a global list of index blobs either. We also don't have to maintain a bag instead of a set as no step relies to more than 1 index entry in the global set of index entries.

**Convert local\_index\_blob\_list to local\_index\_entries\_to\_flush\_set -** Note that since we are maintaining just a global set of index entries now, we can convert local\_index\_blob\_list in snapshot and GC process records to a set. We still have to maintain these fields even though splitting information is not maintained in the global set of index entries because the index entries are flushed to the global index in batches and we still have to mimic that behaviour. We can't mimic that by writing a subset of unwritten contents to the index because the index entries might have different timestamps and they get assigned only the logical timestamp which is incremented via a designated step in the specification (as an aside I tried to write the specification without having a logical timestamp field and writing a batch of index entries into the global index while mimicing the behaviour that the contents might be written at different timestamps by assigning increasing timestamp values to index entries; but the specification became hard to interpret).

To conclude, use unordered data structures such as bags and sets where ever possible in case the ordering information is not used in any step. This helps reduce the state space.

Below is the new set of state variables and steps after this change.

	index_entries_set,
	(*  {
			[index\_entries\_list |-> <<[content_id |-> 0, deleted |-> TRUE/FALSE, timestamp |-> 0],
	    	  [content_id |-> 1, deleted |-> TRUE/FALSE, timestamp |-> 0]>>],
			[index\_entries\_list |-> <<[content_id |-> 0, deleted |-> TRUE/FALSE, timestamp |-> 0],
	    	  [content_id |-> 3, deleted |-> TRUE/FALSE, timestamp |-> 0]>>]
	    }
	*)
	
	snapshot_processess_bag,
	(*
	    {
	     [status |-> "in_progress",
	      content_ids_written_list |-> <<5, 6, 7, 8, 9>>,
	      local_index_entries_set |-> {[content_id |-> 5, deleted |-> FALSE, timestamp |-> 0, data_blob_id |-> 3],
	        [content_id |-> 1, deleted |-> FALSE, timestamp |-> 0],
	        [content_id |-> 3, deleted |-> FALSE, timestamp |-> 0]}
	      local_data_blob_record |-> [content_ids_list |-> <<8, 9>>],
	      local\_index\_entries\_to\_flush\_set |-> {
	        [content_id |-> 6, deleted |-> FALSE, timestamp |-> 0]
	      }
	      start_timestamp |-> 1]: 2,
	      ...
	    }
	*)
	
	gcs_processes_bag,
	(* {
	        [local_snapshot_entries_bag |-> a bag of snapshot processes as above
	         local\_index\_entries\_to\_flush\_set |-> {[content_id |-> 1, deleted |-> TRUE, timestamp |-> 2, data_blob_id |-> 0]}
	         local_index_entries_set |-> {
	           [content_id |-> 1, deleted |-> FALSE, timestamp |-> 0],
	           [content_id |-> 3, deleted |-> FALSE, timestamp |-> 0]
	         }
	        ]: 2,
	        ...
	    }
	*)
	
	logical_timestamp
	
	Steps -
	
	TickTimeForward
	
	For snapshot processing -
		TriggerSnapshot
		PackContents
		FlushLocalDataBlob
		FlushLocalIndexBlob
		MarkComplete
		DeleteSnapshot
	
	For GC processing -
		TriggerGC
		MarkContentDeleted
		FlushLocalIndexBlob

