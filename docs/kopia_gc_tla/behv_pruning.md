---
layout: default
title: Behaviour pruning using Modus Tollens
parent: TLA+ for Kopia's GC
nav_order: 11
---

Assume a disjoint set of behaviours X and Y allowed by the system. If we are able to establish a fact such as - if the safety condition breaks in Y => it will break in X, we can avoid captures the behaviours in Y in the specification. This is because if the safety condition doesn't break in X, we can be sure that it doesn't break in Y as well. This is just an application of Modus Tollens.

We know that safety will be violated only in behaviours with garbage collection where a content is marked deleted which is still referenced by a snapshot. The index compaction process after GC only sets it in stone that a content is lost forever.

**Refined safety condition** - For every content id present in a snapshot in the repository, an index entry with location of content data can be found after which there is no later entry for the index entry with the deleted flag as True.

So, our safety condition can be set up as this invariant - the latest index entry for a content id used by a snapshot shouldn't be marked deleted.

Let us now consider the follwoing:
	Y = the set of behaviours which include garbage collection and index compaction steps
	X = the set of behaviours which include garbage collection but no index compaction steps.

It is clear that if the invariant breaks in Y, it will break in X as well, since index compaction doesn't nothing more than remove the content's index entries forever. So we can prune our the behaviour space to be checked to exclude the behaviours in Y. Hence, we can remove index compaction processes and index_blob_id from the state variables (since these were used only by index compaction).

Below is the new set of state variables and steps after this change.

	index_blobs_list,
	(*  <<
			[index\_entries\_list |-> <<[content_id |-> 0, deleted |-> TRUE/FALSE, timestamp |-> 0],
	    	  [content_id |-> 1, deleted |-> TRUE/FALSE, timestamp |-> 0]>>],
			[index\_entries\_list |-> <<[content_id |-> 0, deleted |-> TRUE/FALSE, timestamp |-> 0],
	    	  [content_id |-> 3, deleted |-> TRUE/FALSE, timestamp |-> 0]>>]
	    >>
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
	      local_index_blob_list |-> <<
	        [content_id |-> 6, deleted |-> FALSE, timestamp |-> 0]
	      >>
	      start_timestamp |-> 1]: 2,
	      ...
	    }
	*)
	
	gcs_processes_bag,
	(* {
	        [local_snapshot_entries_bag |-> a bag of snapshot processes as above
	         local_index_blob_list |-> <<[content_id |-> 1, deleted |-> TRUE, timestamp |-> 2, data_blob_id |-> 0]>>
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
