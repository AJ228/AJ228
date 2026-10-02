# CloudFS: A Hybrid Cloud-Backed File System

**Tools:** C++, FUSE, AWS (S3) · **Context:** CMU Pittsburgh, Storage Systems coursework, Fall 2025

## Overview

CloudFS is a FUSE-based file system that sits between the Linux VFS layer and cloud storage,
giving applications a normal-looking local filesystem backed transparently by the cloud
underneath. The project builds up in four layers: size-based placement across local SSD and
cloud storage, block-level deduplication to cut cloud costs, point-in-time snapshots for backup
and recovery, and a local cache to reduce the cost of repeatedly fetching cloud data. I built all
four layers solo, implementing the FUSE operation handlers and each layer's design from there.

## Tiering: local SSD vs. cloud

Every file write starts on local disk. Below a configurable size threshold, a file stays local —
there's no benefit to involving the cloud for something that small, and doing so would only add
latency and per-request cost. Above that threshold, the file is migrated to cloud storage once
closed, and the local copy is replaced with a proxy: a stub that holds the file's metadata
(size, timestamps, permissions) so that operations like `stat` or a directory listing never need
to touch the cloud just to report attributes. Only opening the file for its actual contents
triggers a fetch. That split — metadata stays cheap and local, only content fetches touch the
network — is what keeps directory operations on a cloud-backed tree close to local speed, and
it's also what the assignment's own correctness tests check directly: reading metadata alone
(e.g. `ls -laR`) isn't allowed to cause any cloud access at all.

## Deduplication

Large files heading to the cloud often share content with each other — the same library
checked into multiple projects, files that differ by only a few bytes after an edit. Naive
whole-file hashing only catches exact duplicates and breaks the moment a single byte is
prepended or appended, since that shifts every fixed-size block boundary that follows it. The
project uses Rabin fingerprinting instead: a rolling hash over the file's byte stream marks
segment boundaries based on content rather than fixed offsets, so two files that share a chunk
of content still produce matching segments even if the shared data isn't aligned the same way in
both files. Each segment is hashed, and a segment is only uploaded if its hash hasn't been seen
before; otherwise the file just references the existing one.

Average segment size is a real trade-off here, and one the project explicitly asks you to reason
about: a larger average segment means less metadata and fewer lookups, but a smaller one catches
more duplication since you're comparing at a finer grain. I tuned this against a realistic mixed
workload rather than picking a default and moving on, since the right answer depends on how much
of the data set is actually redundant and at what granularity.

The harder part of dedup isn't segmenting — it's safe reclamation. A segment can be referenced by
several files, so deleting one file can't simply delete its segments out from under whatever else
still points to them. I used a reference-counting scheme tied to the segment lookup table:
incrementing on new references, decrementing on deletion, and only freeing a segment's cloud
storage once its count hits zero. Getting this right under concurrent-looking edit patterns
(delete and re-add touching the same segment) was the part of the assignment I spent the most
time testing carefully.

## Snapshots

The snapshot layer adds point-in-time backup and restore, exposed through an `ioctl` interface on
a special `.snapshot` file in the FUSE root — `SNAPSHOT`, `RESTORE`, `DELETE`, `INSTALL`,
`UNINSTALL`, and `LIST`, each identified by a timestamp. Taking a snapshot doesn't mean copying
the whole file system to the cloud: file data above the migration threshold is already in the
cloud and already assumed reliable, so a snapshot only needs to capture the local SSD's small
files and metadata, plus bump the reference count on every cloud segment the snapshot now
depends on — the same reference-counting structure used for deduplication reused here to keep
a snapshot's segments alive even if the live file system later deletes its own reference to them.
Restoring undoes all changes since that point (and discards any later snapshots); installing lets
a snapshot be browsed read-only without restoring or disturbing the live file system, similar in
spirit to a mount.

## Caching

The last layer uses spare SSD capacity to cache cloud-backed data locally, purely to cut the cost
of repeatedly fetching the same segments from the cloud rather than for reliability — cloud
providers bill per request and per byte transferred, and transfer cost specifically is
significantly more expensive per byte than storage or a single request, so avoiding repeat
fetches of hot segments has an outsized effect on total cost. The cache operates at the same
Rabin-segment granularity as deduplication, is persistent and write-back (surviving remounts, and
captured correctly by the snapshot layer rather than silently dropped), and uses an LRU eviction
policy to decide what to keep when the fixed-size cache fills up.

## Output

The end deliverable was a working FUSE file system passing the project's correctness and cost
tests at each layer: size-based placement with metadata operations staying entirely on the SSD,
Rabin-based deduplication measurably reducing cloud storage consumption, a snapshot `ioctl`
interface supporting the full set of operations with correct reference-count bookkeeping, and an
LRU-backed cache reducing the cloud cost of a repeated-access workload — plus a final report
explaining the design trade-offs and cost/performance evaluation behind each layer.

## What I'd do differently

The reference-counting logic is a single point that every delete, dedup check, and snapshot
operation has to go through correctly — there's no room for two operations racing on the same
segment's count. The project's own single-threaded FUSE model sidesteps real concurrency, but if I
extended this past the course's scope, that bookkeeping is the piece I'd want to make more
explicitly robust under real concurrent access, since it's the part most likely to break outside
the single-writer test setup the course used.
