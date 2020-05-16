---
layout: default
title: Ignore data blobs and data blob
parent: TLA+ for Kopia's GC
nav_order: 9
---

We have already started with a very minimal representation of a repository and the various processes. But we will now go further.

**Safety condition** - Content data for every content id present in a snapshot in the repository can be found.

The state in the spec can ignore the list of data blobs and only maintain the list of index blobs. Safety is violated when content for a content id present in some snapshot is no longer found in the repository. This can happen if the global list of index blobs doesn't contain an index entry to find the content or if the index contains an entry for the content id but the referenced data blob doesn't contain the content or doesn't exist. We know that that the later scenario is not an issue because blob compaction doesn't delete a content from a blob until the corresponding index entry referencing it is not present in the repository.

In a nut shell, it can't happen that an index entry was referencing a content in some data blob and that content got deleted because of data blob compaction. And so, we don't have to consider data blobs and data blob compaction in our specification. Just index blobs and their interaction with snapshot, gc and index blob compaction processes suffice. We know that safety can be violated in Kopia only in a scenario where index blob compaction occurs after gc (with the right edge case where a snapshot running simultaneously with GC referenced the content). Data blob compaction is not the culprit.

**Refined safety condition** - For every content id present in a snapshot in the repository, an index entry with location of content data can be found.

Also, given we are ignoring data blobs, the index entries in the list of index blobs don't need to include references to data blob ids (if an index entry exists, the corresponding data blob will surely exist). Note that removing data blobs from the specification reduces the state space because now the following two states from the previous specificion of last section are the same state in the refined specification -
	one in which a snapshot process wrote 4 contents A, B, C, D arranged in data blobs as A in the first data blob and B, C, D in the second one
	and another state in which a snapshot process wrote 3 contents A, B, C, D arranged in data blobs as A, B in the first data blob and C, D in the second one

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
	      content_ids_to_be_written_list |-> <<5, 6, 7, 8, 9>>,
	      pointer\_to\_next\_content\_to\_pack |-> 1,
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
