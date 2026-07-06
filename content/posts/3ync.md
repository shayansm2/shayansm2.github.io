+++
date = '2026-07-06T10:11:18+03:30'
title = 'Building 3ync: A Local-First, S3-Compatible Sync Engine'
showToC = true
TocOpen = true
[cover]
image = "https://substackcdn.com/image/fetch/$s_!QqFj!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F93d8ebbd-4064-4f40-8b5d-07e449a7487a_1392x738.png"
+++
## The Problem and Old Solutions

I previously came up with a simple local-first solution to sync my Obsidian files between my desktop and phone, which I wrote about in this [blog](https://shayansm.substack.com/p/syncing-your-local-first-notes-with). The way it worked was like this:

![previous design](https://substackcdn.com/image/fetch/$s_!4lAD!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Ffd4c556e-f92e-4928-8b50-fefc9e75a129_908x646.png)

The local storage on my devices (desktop or mobile) was periodically syncing files with the cloud. This cloud storage could be any object storage, or out-of-the-box solutions like Google Drive, Dropbox, etc.

However, it had a major flaw: the syncing method was simple, but fragile. Imagine a scenario where the latest version on the cloud is `v1`. Both mobile and desktop change the contents simultaneously offline, bumping their local states to `v2` and `v3`. In this case, standard syncing causes data loss depending on the order of the sync schedules. If the desktop syncs first, `v2` gets overwritten and lost when the mobile syncs `v3`.

One way to mitigate this is simply to decrease the sync interval. Faster syncs reduce the window of time where two devices can diverge from the remote server's main version. (Keep in mind, these personal notes have only one user, and I can only use one device at a time. If we were building real-time multi-user collaboration, that is a completely different story).
The other, much safer way to achieve this is to implement a robust sync engine that intelligently determines what to keep and what to overwrite. That is why I wrote 3ync, an S3-compatible sync engine. But before explaining how it works under the hood, let’s explore the literature and context of the problem through the lens of my favorite book, _Designing Data-Intensive Applications_.

![DDIA](https://substackcdn.com/image/fetch/$s_!7wty!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fbece2111-5efe-4c1b-abfa-0f6b95929bd3_500x660.png)

## The Theory: Local-First as a Distributed System
_Note: The concepts below are largely heavily inspired by Chapter 6 (Replication) of DDIA. If you haven't read it, I highly encourage you to check it out—it's an amazing read. You can also check out my other blog on [database index internals](https://shayansm.substack.com/p/database-index-internals), which covers Chapter 4 of the same book._

Replication is a foundational concept in designing stateful distributed systems. It enables higher availability, performance, and fault tolerance. The three common replication architectures are:
1. **Single-Leader Replication:** The most commonly used approach. If a database needs to handle high read loads or be fault-tolerant via failover mechanisms, this is the simplest and standard solution.
2. **Multi-Leader Replication:** Often used for replication across massive geographic regions, or—importantly for us—**local-first replication**.
3. **Leaderless Replication:** Nodes accept writes directly without a designated leader

In our Obsidian setup, each local database (the local file system on the phone or desktop) acts as a "leader" because it actively accepts write requests from the user offline. The flow of syncing these changes from one local device to a remote server (and indirectly to other local devices) is fundamentally a replication process.

Because we have multiple local DBs (leaders) syncing (replicating) their data with one another, the local-first sync problem officially maps to **multi-leader replication**. **Sync engines** are the crucial piece of software that handles this multi-leader replication in the background, allowing the user to seamlessly read and write data in the foreground without waiting for network latency.

One of the biggest hurdles multi-leader replication must address is **conflict resolution**. How do leaders agree when they both edit the same file? Here are the common strategies:

1. **Conflict avoidance:** If you can ensure that all writes for a particular record go through the same leader, conflicts cannot occur. Useful for geo-replicated systems, but pointless for local-first sync engines.
2. **Last Write Wins (LWW):** One of the simplest solutions. The system simply keeps the most recent write based on a timestamp and discards the others. The downside? It can cause data loss.
3. **Manual conflict resolution:** Prompting the user to fix the clash directly, similar to resolving a Git merge conflict.
4. **Automatic conflict resolution:** Algorithms that diff the initial and final states of conflicting writes, then try to mathematically apply both. (For example, if the initial state was 50, and node A subtracts 4 to make it 46, while Node B subtracts 2 to make it 48, the engine resolves the final state to 44).
5. **Conflict-Free Replicated Data Types (CRDTs).**
6. **Operational Transformation (OT).**

Due to the deep complexity of CRDTs and OT, I won't dissect them here. I highly recommend reading Chapter 6 of _Designing Data-Intensive Applications_, or reviewing Chapter 8 of Dr. Martin Kleppmann's _Distributed Systems_ course at the University of Cambridge.

## The 3ync Engine Concept and Architecture
I decided to write a S3-compatible sync engine, named **3ync**, that syncs S3 buckets with one another. Each object in a bucket is a "record," and each bucket acts as a leader database.

First, let's look at the architecture of how this is applied to my Obsidian notes:

![current 3ync design](https://substackcdn.com/image/fetch/$s_!QqFj!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F93d8ebbd-4064-4f40-8b5d-07e449a7487a_1392x738.png)

Both the desktop and phone push their local changes to their own dedicated cloud buckets. The 3ync engine (currently running server-side) then kicks in to sync a "Main" bucket with the desktop bucket and mobile bucket. Finally, the desktop and mobile pull from the Main bucket to update their local files

For conflict resolution, I opted for **Last Write Wins (LWW)** on an object level. To solve LWW's famous "data loss" problem, I built in an automatic backup mechanism. If 3ync overwrites a file, a backup of the deleted file is saved to `.3ync/backups/{timestamp

_Note: Currently, this sync evaluates the entire object, not the granular text content inside the Markdown files like a CRDT or OT would. If two concurrent writes happen on entirely different lines of the same file, the file with the highest timestamp simply wins, and the other is backed up and removed._

## How LWW Interacts with S3 Metadata
During a sync, if an object exists in both the source bucket and replica bucket, 3ync first checks their `ETag` (the object's hash) to see if the contents are identical. If the ETags differ, we choose the object with the most recent `last_modified` timestamp.

The problem gets much more complicated if a file exists in one bucket but is missing in the other. Because we don't immediately know if the object was just created in Bucket A, or if it was recently deleted from Bucket B.
To track this state safely, 3ync writes a `last_update` timestamp to a `.3ync/metadata.json` file inside the bucket after every successful sync run. This acts as our anchor to determine if a missing file implies a "create" or a "delete" event.
Here is the pseudocode of the core LWW sync logic:
```

```FUNCTION LWW_SYNC(source, replica)
    CASE

        Source missing:
            Replica older than bucket_metadata.lastUpdate?
                YES → Do nothing (File was previously deleted in source)
                NO  → Copy object from replica to source (New file in replica)

        Replica missing:
            Source newer than bucket_metadata.lastUpdate?
                YES → Do nothing (New file in source)
                NO  → Backup source object
                      Delete source object

        Both exist:
            Objects identical (ETags match)?
                YES → Do nothing

            Source newer?
                YES → Do nothing

            Replica newer:
                YES → Backup source object
                      Replace source object with replica object

END FUNCTION
```
You can check out the actual Go implementation [here](https://github.com/shayansm2/3ync/blob/main/sync_engine.go). I’ve also covered these edge cases thoroughly in my integration tests using [Testcontainers](https://testcontainers.com/), which you can view [here](https://github.com/shayansm2/3ync/blob/main/sync_engine_test.go).

## Current Limitations and Future Improvements
1. **Storage Overhead:** Because 3ync strictly utilizes separate buckets per device, the storage footprint is roughly 3 times larger than the actual note contents.
2. **Server-Side Bottleneck:** Currently, the 3ync engine runs as a server-side process syncing cloud buckets together. The ideal architecture would move the 3ync runtime directly to the client side. This would eliminate the need for device-specific cloud buckets, allowing the client to sync directly with one central server bucket.

![future-design](https://substackcdn.com/image/fetch/$s_!901T!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fe019891e-6e1a-421f-a614-97df98e9b1cb_894x634.png)

3. **No Garbage Collection:** There is no automated cleanup process for the `.3ync/backups` folder. Right now, pruning old backup files is completely manual.
4. **Platform Lock-In:** 3ync is strictly S3-compatible and does not yet work natively with local file systems directly or native APIs for Google Drive/Dropbox.

Thanks for reading! I’d love to hear your thoughts, and I would be incredibly grateful if you left a star on the [3ync repo](https://github.com/shayansm2/3ync/tree/main) if you found the concept helpful.

## References
1. _Designing Data-Intensive Applications_ by Dr. Martin Kleppmann
2. _Distributed Systems_ course at the University of Cambridge by Dr. Martin Kleppmann
