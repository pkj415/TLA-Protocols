---
layout: default
title: Garbage collection, index/data blob compaction
parent: TLA+ for Kopia's GC
nav_order: 4

gc_process_images:
  - image: /assets/images/images.016.png
  - image: /assets/images/images.017.png
  - image: /assets/images/images.018.png
  - image: /assets/images/images.019.png
  - image: /assets/images/images.020.png
  - image: /assets/images/images.021.png
  - image: /assets/images/images.022.png
  - image: /assets/images/images.023.png
  - image: /assets/images/images.024.png
---

Garbage collection of unused contents (either due to duplicate copies of a content or due to the fact that no snapshot metadata refers to a content) in Kopia is not trivial. This is due two reasons -

1. Blobs are immutable which requires GC to rewrite them without the contents (or index entries) which are not needed anymore, followed by deletion of the old blobs.
2. Even as we rewrite blobs while keeping only necessary information, it might happen that another snapshot process which started simultaneously with the garbage collection  process, reuses a content while it is being considered for deletetion.

The strategy of vanilla garbage collection in Kopia is described next for both cases when a content is not needed anymore -

1. Duplicate copies of the content exist and all except one copy can be removed.
2. The content is not being referenced by any snapshot.

**Content with duplicate copies** - In this case, the content is removed in two stages - i. index blob compaction ii. data blob compaction. This is without garbage collection process' intervention. If you notice, the content C47 is present twice in the blob store and has two index entries. The older index entry and it's corresponding content data are removed. The older entry will be cleaned up by index compaction and its corresponding content will be cleaned up by data blob compaction. This presents no issue, as restoration of any snapshot which references the content will remain unhindered as a copy of the content is still present which can be referenced using the index entry which is left untouched. Both index and data blob compaction are periodically scheduled background tasks and they are explained in more detail later.

**Content not referenced by an snapshot** - In this case, the content is removed in three stages - i. content is marked deleted by the GC process (more on what "marking" deleted means shortly) ii. all index entries for the content are removed by index blob compaction iii. the actual content is deleted via data blob compaction. From the previous example of snapshotting, the content C48 is not referenced by any snapshot metadata. This will be cleaned by being marked as deleted by the GC process which first collects a point in time view of snapshots and list of index blobs. It then sweeps through all snapshots to find content IDs which are referenced by any snapshot and then sweeps through the list of index blobs to find index entries of content ids which are are not referenced by any snapshots. Marking a content as deleted involves adding an index entry for the content id with the deleted flag as True, a timestamp, and a pointer to the data blob same as the previous index entry. Later, the next scheduled index compaction process will remove all the index entries for C48 including and older than the deleted entry. Post this blob compaction will remove all contents which are not referenced by an index entry. Also, to avoid cases where GC marks content written by a snapshot creation process in progress as deleted, GC ensures that it marks contents which are older than MaxSnapshotTime.

In a nutshell, any content which is no longer referenced by a snapshot is marked deleted by GC so that its index entries can later be removed completely by index compaction and content data can later be removed completely by blob compaction similar to the previous case of cleaning up duplicate contents. The delete marker is just an indication that not even a single copy of the content before the deleted entry timestamp is to be kept in the system. And any later snapshot creation is expected to rewrite the content (the GC can't retain the content because it can't anticipate when such a snapshot creation will occur and keeping the content around for this possibility is just a waste of space. It is beneficial to rewrite the content later instead of wasting space to possibly avoid writing the content later).

You might have guessed what might be wrong here - there might exist a snapshot process that started before the GC process. which read the global list of index blobs to create its local set of index entries and then reused the content. This is an issue because if later we try to read the snapshot (possibly for restoring it), the content might not be found via any index entry as it would have been removed by index compaction (the GC running in parallel with the snapshot would have marked the content deleted). Note that before index compaction, we can still find the content data by looking at the index entries. But this is only if we get lucky that the scheduled index compaction process doesn't delete the index entries before we try to read the content. Also, even if we get lucky, eventually index compaction will remove the index entries and we will not be able to read all contents of the snapshot later.

{% include gc_process.html height="70" unit="%" duration="7" %}

##### How index blob compaction works -

If there is an index entry for a content that marks it deleted, then all earlier index entries for the content (including the one that marks it deleted) can be removed. If there are multiple index entries for undeleted contents, the older ones can be removed. In case there exist two index entries for a content with the same timestamp (this can happen in case two snapshot processes independently write the content at the same time), either of the index entries can be removed.

Since blobs are immutable in Kopia, index compaction of a list of index blobs X involves creating a new index blob without index entries which are to be removed from the entries in X. Once the new index blob is created, the index blobs in X can be deleted. The removal is done post flushing of the new index blob so that no content specified in snapshots becomes unreachable.

##### How data blob compaction works -

Data blob compaction sweeps through all contents in all data blobs in the repository and deletes any contents which are not referenced by any index entry in the index blobs. Contents can't be removed by shifting contents in a data blob as a blob is immutable. The only way to compact blobs is to find blobs which contain contents which are no longer needed (i.e., not referenced by any index entry) and then merge them into a smaller number of blobs without the unneeded contents. Also, a new set of indices will have to be written to point to the new contents. Post this, the old data blobs can be removed. Note that after writing new index blobs, there will be duplicate old entries for many contents in the old index blobs and this will be taken care of by index compaction. The process of writing data and index blobs is similar to data of a snapshot creation process. The data blob compaction process maintains a local data and index blob which are periodically filled and flushed.
