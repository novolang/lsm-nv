# lsm-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

A log-structured merge tree, as arithmetic over blocks somebody else
stores.  A sorted memtable, an SSTable format, a write-ahead log, a
manifest of levels, a compaction planner and a merge iterator — and not
one file handle anywhere in it.

It is the `core` half of a storage engine.  The `host` half — the files,
the fsync, the background compaction thread, the snapshot registry —
is a separate package and is described at the end.

Six modules, and a reader should know which one they are on.

| surface | module | reach for it when |
| --- | --- | --- |
| the **entry and the order** | `lsmmem` | anything. Start here |
| the **table** | `lsmsst` | you are writing or reading an SSTable |
| the **log** | `lsmwal` | you are writing or recovering a WAL |
| the **levels** | `lsmlevel` | you are deciding what to compact |
| the **merge** | `lsmmerge` | you are compacting, scanning or reading |
| the **faults** | `lsmerror` | something would not decode |

## Adding it, and checking it

```bash
novo pkg add lsm-nv            # into your novo.toml
novo pkg build                 # type- and effect-check the package
novo test --isolate tests/lsmmem_tests.nv
```

`novo test` is red today and that is the point of the release: every
assertion fails with `not implemented: lsm-nv.<module>.<fn>`.  They
turn green one at a time as bodies land.

## The one example that will work

```novo
use lsmmem
use lsmsst

// Flush a memtable to an SSTable, into a buffer the caller owns.
fn flush(m: lsmmem.LsmMemtable, o: lsmmem.LsmOptions) -> [u8] [io]
    match lsmsst.write_table([], lsmmem.entries(m), o)
        Err(f) => []
        Ok(out) =>
            // out.footer.keys is the table's range, for the manifest.
            // out.bytes is what the host writes to a file.
            out.bytes
```

Nothing in that opens anything.

## The load-bearing interface

`lsmmem.compare_entries` — **by key ascending, then by sequence number
DESCENDING.**

```novo norun:pseudo
pub fn compare_entries(a: LsmEntry, b: LsmEntry) -> Int []
```

One comparator, public, and every other module written against it: the
memtable's insert, the SSTable's block layout, the merge iterator's
choice, and the compaction's drop decisions.

Why it is worth publishing rather than burying.  A key in an LSM is not
one value — it is a **sequence of versions** spread across the memtable
and every level, and the newest one wins.  Every correctness bug an LSM
has is a version-ordering bug: a merge that answered the older of two
versions; a read that stopped at level 1 while level 0 held a newer
write; a compaction that dropped a tombstone and resurrected a key
deleted a year ago.  Each of those is this comparator applied
inconsistently in one place, and the only defence is that there is
exactly one of it and it has a name.

The **descending** half is the part people write ascending.  With
sequence descending, the first entry for a key *is* the current one, so
every reader is "find the key, take the first" and no reader scans to
the end of a run.  Ascending would make every one of them scan, and the
one that forgot would answer a stale value with nothing to show for it.

The second load-bearing rule is **a tombstone may only be dropped at
the bottom level**.  A compaction that dropped one at level 2
resurrects the key from the copy at level 3 it did not read — silently,
minutes later, to a key somebody deleted.  So the decision is
`lsmlevel.plan_may_drop_tombstones`, it is a function of the plan, and
`lsmmerge.merge` takes the answer as an **argument** rather than
deciding for itself.  A merge cannot get it wrong without a caller
having written `true`.

## The reader asks, the caller performs

`lsmsst.reader` answers `LsmNeedLength`, then `LsmNeedBytes(table, at,
len)`, and the caller feeds it.  Nothing here reads a file, so the same
reader serves a file on disk, a blob in object storage and a fixture a
test built in memory — the only thing that differs is who answers the
request.

That is btree-nv's `PageRequest` shape, for the same reason, and the
two are deliberately separate types: a B-tree asks for fixed-size
**pages** and an SSTable asks for variable-length byte **ranges**, and
one enum covering both would have a field that is meaningless in half
its uses.

The table format is LevelDB's, and its one unusual decision is the
reason it works over a network: a **fixed-size footer at the end**.  A
reader that knows the file's length knows exactly which bytes to fetch
first, in one request, having read nothing.

## The merge is fed, not pulling

`lsmmerge` holds one entry per source and answers `LsmMergeWant(i)`
when a source runs dry.  A pulling iterator would have needed every
source to be a trait with an effect row, and a `core` function that
calls one inherits that row and leaves its budget (SPEC § 5.6).  Feed
and drain is the shape that holds — and it also lets a caller fetch
four sources concurrently, which a pulling iterator forces into series.

Source order **is** precedence: source 0 is newest.

## The write-ahead log shares pager-nv's shape and not its code

pager-nv is `host` and this is `core`, so the two cannot be related by
a dependency.  What they share is a design, and it is worth naming
because its three decisions are each individually easy to get wrong:

- **A generation, not a truncation.**  The header carries two salts and
  every record repeats them.  A checkpoint bumps the salts rather than
  emptying the file, so there is no window in which the log is gone and
  the tables are not yet durable.
- **A cumulative checksum.**  Each record's folds in the one before it,
  so a record lifted out of another log and dropped in is detected.  A
  per-record checksum accepts it.
- **The commit marker is in the record.**  A separate commit record
  would leave a state — batch written, marker not — that is
  indistinguishable from a torn tail and has to be handled the same
  way, so it buys nothing and costs a write.

Where it differs: pager-nv logs **pages** and this logs **entries**,
and the checksum is crc-nv's CRC-32C rather than SQLite's rolling pair,
because `lsmsst` already checksums with crc-nv.

**A torn tail is not an error.**  The ordinary way a crash ends a log
is half a record.  `lsmerror.ends_recovery` answers true for that and
for a stale generation, so a recovery that treated either as corruption
— and discarded every committed write in the log — cannot be written by
accident.

## Level 0 is different, and the package says so once

Level 0's tables **overlap**, because each one is a memtable flushed
whole and two flushes cover the same keys.  So a read that misses the
memtable consults *every* level-0 table, and level 0's compaction
trigger is a table **count** rather than a byte size.  Every deeper
level's tables are disjoint: a read consults at most one, and the
trigger is bytes.

Two rules one field apart, so `lsmlevel.is_overlapping` is the
predicate and `level == 0` is not written in six places.
`lsmlevel.read_path` answers the whole consult order in one call, which
is where a reader would otherwise forget.

## What the host half does

The row above this one on the grid.  novokv is the consumer it is
shaped for, and reading novokv says what that store keeps and what it
would hand over:

**What novokv keeps.**  One shard per CPU core, each owning a slice of
the keyspace outright, reached from one loop on one thread — so there
is no lock anywhere in it and no atomic on the hot path.  Requests for
another shard's keys are forwarded over the cell mesh.  Time is a
parameter rather than a call, so its TTL tests run in microseconds.
None of that changes: the ownership model is the reason novokv is fast
and it is orthogonal to where the bytes live.

**What it would hand to the host half over this package.**  Today
novokv's store is a `HashMap` and nothing survives a restart; it is a
cache, and its own comments say so.  The host half would give it: a
file per table and a log, opened and fsynced; a background thread that
runs the plans `lsmlevel.pick` answers; a snapshot registry that turns
live readers into the `oldest_snapshot` a plan carries; and the block
cache that decides which `LsmNeedBytes` are answered from memory.  All
four are effects, all four are the machine rather than the algorithm,
and none of them is in this package.

**What would have to change in novokv.**  Its `apply` takes a map and
returns bytes to send; over this package it would take a memtable and a
manifest and might additionally return a request for a byte range — the
same shape, one more variant in the reply.  Expiry stays lazy: a TTL
becomes a tombstone written at the sequence the key expires, which is
what every LSM-backed cache does and what keeps the read path from
consulting a clock.

## Why `core`

Nothing here opens a file, syncs one, takes a lock or starts a thread.
A writer appends to a caller's buffer; a reader answers a request; a
compaction answers a **plan** rather than running one.

That last one is what makes a compaction policy testable: a test builds
a manifest, asks for a plan, and asserts on the answer — with no file,
no thread and no timing in it.  A planner that also ran the compaction
could only be tested by running it.

No device claim.  A memtable is a list of entries each holding two heap
byte strings, and a bloom filter at ten bits per key is more memory
than a microcontroller has.  A device that logs to flash wants a ring
buffer, which is bbqueue-nv.

## Dependencies

Two, both `core`: **crc-nv** for every block, footer and log record's
checksum, and **varint-nv** for every length and offset.  Both are
tables that should exist once on this registry — a CRC table
transcribed twice differs in one entry, and a varint written twice is
two decoders that disagree about a 64-bit boundary, which only appears
on the large values nobody's fixtures have.

## Ports

[LevelDB](https://github.com/google/leveldb) and
[RocksDB](https://github.com/facebook/rocksdb) are the reference
implementations.  The table format is LevelDB's, the level sizing and
the defaults are both projects' where they agree, and the WAL's shape
is pager-nv's, which is SQLite's.

## Licence

Apache-2.0.
