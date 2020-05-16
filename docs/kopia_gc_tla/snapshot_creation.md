---
layout: default
title: Snapshot creation
parent: TLA+ for Kopia's GC
nav_order: 3

snapshot_process_images:
  - image: /assets/images/images.002.png
  - image: /assets/images/images.003.png
  - image: /assets/images/images.004.png
  - image: /assets/images/images.005.png
  - image: /assets/images/images.006.png
  - image: /assets/images/images.007.png
  - image: /assets/images/images.008.png
  - image: /assets/images/images.009.png
  - image: /assets/images/images.010.png
  - image: /assets/images/images.011.png
  - image: /assets/images/images.012.png
  - image: /assets/images/images.013.png
  - image: /assets/images/images.014.png
  - image: /assets/images/images.015.png
---

The process of snapshot creation in Kopia writes filesystem data (under a user specified root directory) to a repository after splitting the data into objects, where each object is inturn  split into contents. We can ignore this detail and assume that a snapshot process starts up with the intention to write a set of contents (and the corresponding index entries to find those contents) to the repository. The contents are written to data blobs and index entries are periodically flushed, one index blob at a time.

When a process to create a snapshot is triggered, it reads a point in time copy of the index blobs on start up before performing any tasks and stores all the index entries from all index blobs in an unordered set (the information about division of index entries into different index blobs and ordering of index entries in each blob is not needed and hence is not maintained in the local copy of index entries). This local copy of index entries helps in deduplication of data i.e., if some data has already been written to the repository, the snapshotting process can avoid writing the data again (also called reusing the content). This is done by checking if there exists an index entry for the content id (which is a hash of the content data which the snapshotting process intends to write). Note that only those contents are visible to the snapshotting process which were flushed before a point in time view of the index blobs was stored in the local index entries set of the snapshotting process. To search any content, a process searches its local set of index entries to find the entry with the latest timestamp for the content id being searched (the older index entries for a given content id could also be used but they aren't because - in case the latest entry has the delete flag set as True, it implies that the content has already been marked for deletion by the GC process and the snapshot should rewrite the content; this will become more clear in the GC process section).

As part of snapshotting, when data blobs are written to the repository, they are added to the global list of blobs maintained in the repository. When index blobs are flushed, they are added to the global list of index blobs of the repository. They are also added to the local set of index entries which were populated on startup so that the process can search for the contents it had already written and reuse them.

**NOTE -** I will usually say "write" to mean flushing of data/index blob to the repository and "pack/append" to be adding a content/index entry to a data/index blob.

Completion of a snapshot process results in some metadata entry written atomically to the repository which points to the content ids which make up the snapshot. A snapshot process can complete within a time limit as specified by the MaxSnapshotTime constant. On completion, the snapshot metadata is written to the repository atomically (in some way which is unimportant to us). In case a snapshot process doesn't complete within the time limit, it can still write contents and flush indices to the repository. However, it cannot mark itself completed (i.e., the snapshot metadata can't be written).

**NOTE -** When we say snapshot in this document later, we mean the snapshot metadata which defines it. Without snapshot metadata written in the repository, the snapshot doesn't exist for the outside world even if a snapshot creation process has written all the data to the repository but failed just before writing the snapshot metadata to the repository.

A snapshot can also later be deleted. Deletion of a snapshot results in the metadata entry of the snapshot being deleted.

{% include snapshot_process.html height="70" unit="%" duration="7" %}

The image slide show walks through a sample scenario. The starting state of the repository has no data blobs or index blobs. There are two snapshot creation processess which are started one after the other. The first snapshot process S1 intends to write some contents {C45, C46, C47} and the process starts at timestamp 0. It writes one data blob with contents C45 & C46 and writes an index blob with the corresponding index entries. Later another snapshot process S2 which intends to write the contents {C45, C46, C47, C48} is triggered and it reads a point in time view of index entries from the index blob. It finds contents C45 & C46 already in the local index entries set and so doesn't add them to the local data blob for writing. However, after this the first snapshot process S1 writes another data blob with content C47, flushes the index blob and marks itself completed. But S2 isn't aware of this new index entry as the local set of index entries are populated only at the start of snapshotting process. So it writes a data blob with the content C47 & C48. C47 is now present in two data blobs with two different index entries in separate index blobs pointing to the respective data blob contents. Having two copies of the same content just consumes space and is useless. Post this snapshot S2 is deleted and so content C48 is useless as well. It is the task of the garbage collection module (in conjunction with index compaction and blob compation) to delete any unused contents & corresponding index entries to free space.
