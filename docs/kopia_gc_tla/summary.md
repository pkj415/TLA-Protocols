---
layout: default
title: Tieing it all up
parent: Abstraction for Vanilla GC
nav_order: 7
---

Below is a brief representation of our state variables and steps. I have given a sample for each state variable (this syntax is not stricly follwing TLA+ syntax rules, I am just trying to represent everything succinctly). \<\< \>\> is notation to represent a list. Sets and bags are both representated with {} (bags also have a number alongside the element). Records are of the form \[field1 \|-\> value, field2 \|-\> value ...]

	data_blobs_list,
	(* <<
			[data_blob_id |-> 0, content_ids_list |-> <<0, 1, 2>>],
			[data_blob_id |-> 1, content_ids_list |-> <<3, 1, 5>>]
	   >>
	*)

	data_blobs_list,
	(* <<
			[data_blob_id |-> 0, content_ids_list |-> <<0, 1, 2>>],
			[data_blob_id |-> 1, content_ids_list |-> <<3, 1, 5>>]
	   >>
	*)
	
	index_blobs_list,
	(*  <<
			[index_blob_id |-> 0,
			 index\_entries\_list |-> <<[content_id |-> 0, deleted |-> TRUE/FALSE, timestamp |-> 0, data_blob_id |-> 0],
	    	  [content_id |-> 1, deleted |-> TRUE/FALSE, timestamp |-> 0, data_blob_id |-> 0]>>],
			[index_blob_id |-> 2,
			 index\_entries\_list |-> <<[content_id |-> 0, deleted |-> TRUE/FALSE, timestamp |-> 0, data_blob_id |-> 0],
	    	  [content_id |-> 3, deleted |-> TRUE/FALSE, timestamp |-> 0, data_blob_id |-> 1]>>]
	    >>
	*)
	
	snapshot_processess_bag,
	(* 
	    {
	     [status |-> "in_progress",
	      content_ids_to_be_written_list |-> <<5, 6, 7, 8, 9>>,
	      pointer\_to\_next\_content\_to\_pack |-> 1,
	      local_index_entries_set |-> {[content_id |-> 5, deleted |-> FALSE, timestamp |-> 0, data_blob_id |-> 3],
	        [content_id |-> 1, deleted |-> FALSE, timestamp |-> 0, data_blob_id |-> 0],
	        [content_id |-> 3, deleted |-> FALSE, timestamp |-> 0, data_blob_id |-> 1]}
	      local_data_blob_record |-> [content_ids_list |-> <<8, 9>>],
	      local_index_blob_list |-> <<
	        [content_id |-> 6, deleted |-> FALSE, timestamp |-> 0, data_blob_id |-> 3]
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
	           [content_id |-> 1, deleted |-> FALSE, timestamp |-> 0, data_blob_id |-> 0],
	           [content_id |-> 3, deleted |-> FALSE, timestamp |-> 0, data_blob_id |-> 1]
	         }
	        ]: 2,
	        ...
	    }
	*)
	
	data_blob_compaction_processes_bag,
	(* {
	        [local\_data\_blob\_ids\_list |-> <<0, 1, 3>>,
	         local_index_entries_set |-> {
	           [content_id |-> 1, deleted |-> FALSE, timestamp |-> 0, data_blob_id |-> 0],
	           [content_id |-> 3, deleted |-> FALSE, timestamp |-> 0, data_blob_id |-> 1]
	         }
	         local\_data\_blob\_record |-> [content_ids_list |-> <<1, 3>>],
	         local\_index\_blob\_list |-> <<
	            [content_id |-> 1, deleted |-> FALSE, timestamp |-> 0, data_blob_id |-> 3]
	            [content_id |-> 3, deleted |-> FALSE, timestamp |-> 0, data_blob_id |-> 3]
	      		>>,
	         current\_blob\_content\_pointer |-> <<1, 3>>, \* blob id, content id
	         data\_blobs\_ids\_to\_be\_deleted\_set|-> {}
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
	
	For data blob compaction -
		TriggerDataBlobCompaction
		PackContents
		FlushLocalDataBlob
		FlushLocalIndexBlob
		DeleteDataBlob)

