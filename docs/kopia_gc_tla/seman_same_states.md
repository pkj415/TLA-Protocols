---
layout: default
title: Avoiding semantically same, distinct states
parent: TLA+ for Kopia's GC
nav_order: 10
---

I will usually call two states in TLA+ to be semantically equivalent and that means - the states might differ in some variable(s) and no temporal properties to be checked or sequence of steps that follow depend on this difference in values of the variable(s).

As per our current set of state variables in the specification, we can have the following two distinct states with MaxSnapshotTime as 2 and with all other state variables which are not specified below as having the same values -


	logical_timetamp = 3
	snapshot_processes_bag = {[
	  status |-> "in_progress", \* Equivalent to a failed snapshot as logical_timestamp > start_timestamp + MaxSnapshotTime
	  content_ids_to_be_written_list |-> <<1, 2, 3, 4>>,
	  pointer\_to\_next\_content\_to\_pack |-> 3,
	  local_index_entries_set |-> {[content_id |-> 1, deleted |-> False, timestamp |-> 0], [content_id |-> 2, deleted |-> False, timestamp |-> 0]},
	  local_data_blob_record |-> [content_ids_list |-> <<>>],
	  local_index_blob_list |-> <<>>,
	  index_blobs_to_be_flushed |-> <<>>,
	  start_timestamp |-> 0]: 1}
	
		VS
	
	logical_timestamp = 3
	snapshot_processes = {[
	  status |-> "in_progress", \* Equivalent to a failed snapshot as logical_timestamp > start_timestamp + MaxSnapshotTime
	  content_ids_to_be_written_list |-> <<1, 2, 3>>,
	  pointer\_to\_next\_content\_to\_pack |-> 3,
	  local_index_entries_set |-> {[content_id |-> 1, deleted |-> False, timestamp |-> 0], [content_id |-> 2, deleted |-> False, timestamp |-> 0]},
	  local_data_blob_record |-> [content_ids_list |-> <<>>],
	  local_index_blob_list |-> <<>>,
	  start_timestamp |-> 0]: 1}


Both states differ only in their set of content_ids_to_be_written_list and there might be a lot of behaviours (let's denote the set by X) in which the step of packing/writing further contents in these snapshots will not be taken. Also, no other steps other than packing/writing further contents in these snapshots depends on content\_ids\_to\_be\_written\_list. So the two states are semantically the same for the behaviours in X. The behaviours where further contents are written allow more contents to be written in both states (3, 4 can be written in the first and just 3 can be written in the second) but it can be seen that every behaviour possible in the second state will be covered in the first state as well with the difference in the field content_ids_to_be_written_list which is not used in any invariant.

Since the field content_ids_to_be_written_list just serves the purpose of deciding contents to be written for a snapshot on triggering (and the order in which they would be written) and then writing those contents in batches of blobs, we can get rid of this field and allow any snapshot process to write any content from the set of possible contents while being allowed to complete at any point in time. Making the snapshot process completion step as always enabled allows us to still capture all behaviours where different snapshots intend to write different sets of contents (in different orders) before completion.

We still have to maintain what contents the snapshot wrote along the way and so maintain a field content_ids_written_list instead. Contents are added to this list as they are written so that if the snapshot completes, we will know which contents it is composed of.

Below is the refined set of state variables and steps after this change.

	index_blobs_list,
	(*  <<
			[index_blob_id |-> 0,
			 index\_entries\_list |-> <<[content_id |-> 0, deleted |-> TRUE/FALSE, timestamp |-> 0],
	    	  [content_id |-> 1, deleted |-> TRUE/FALSE, timestamp |-> 0]>>],
			[index_blob_id |-> 2,
			 index\_entries\_list |-> <<[content_id |-> 0, deleted |-> TRUE/FALSE, timestamp |-> 0],
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

	index_blob_compaction_processes_bag,
	(* {
	        [local_index_blobs_list |-> <<
				[index_blob_id |-> 0,
				 index\_entries\_list |-> <<[content_id |-> 0, deleted |-> TRUE/FALSE, timestamp |-> 0, data_blob_id |-> 0],
		    	  [content_id |-> 1, deleted |-> TRUE/FALSE, timestamp |-> 0, data_blob_id |-> 0]>>],
				[index_blob_id |-> 2,
				 index\_entries\_list |-> <<[content_id |-> 0, deleted |-> TRUE/FALSE, timestamp |-> 0, data_blob_id |-> 0],
		    	  [content_id |-> 3, deleted |-> TRUE/FALSE, timestamp |-> 0, data_blob_id |-> 1]>>]
		      >>,
	         index\_blobs\_ids\_to\_be\_deleted\_set |-> {},
	         new_index_written |->True,
	         local\_index\_blob\_list |-> <<>>
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
	
	For index blob compaction -
		TriggerIndexBlobCompcation
		FlushMergedIndexBlob
		DeleteOldIndexBlob
