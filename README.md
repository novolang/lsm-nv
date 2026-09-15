# lsm-nv

A log-structured merge tree is a key-value store that never updates a
record where it lies. A write goes into a sorted table in memory, that
table is written out whole when it fills, and the tables are merged
into larger ones in the background. It was described in
[The Log-Structured Merge-Tree](https://www.cs.umb.edu/~poneil/lsmtree.pdf)
(O'Neil, Cheng, Gawlick and O'Neil, Acta Informatica, 1996), and
[LevelDB](https://github.com/google/leveldb) and
[RocksDB](https://github.com/facebook/rocksdb) are the implementations
everything since has followed. This package is the arithmetic of one,
and nothing else: it opens no file, starts no thread and takes no
lock.

**Status: NOT IMPLEMENTED — interface only.** Every function is
declared with its full signature, but every body is a `todo()` that
panics when called. The package is published so its design can be
reviewed and depended on before it is implemented. Version 0.1.0 will
be the first working release.

## What it is

An **entry** is one version of one key: the key, a value, a
**sequence number** saying when the version happened, and what it
does. A **put** sets the key. A **tombstone** removes it, and it is an
entry like any other rather than an absence, so that it can travel
through the store and cancel the older copies it meets. A **range
tombstone** removes every key from its own key up to an exclusive end,
in one entry rather than one per key.

A key is therefore not one value but a sequence of versions, and the
newest one wins. **The newest one is the one with the highest sequence
number**, and finding it is the whole difficulty of the design.

The writes that have not been written out yet sit in a **memtable**, a
sorted list of entries in memory. When the memtable reaches its size
it is **frozen** and written out as an **SSTable**, a sorted file of
entries that is never modified afterwards. Tables are arranged in
**levels**, numbered from 0, each level about ten times the size of
the one above. A **compaction** merges tables from one level into the
next, and that merge is where older versions and finished tombstones
are dropped.

A **write-ahead log** (WAL) holds the entries that are in the memtable
and not yet in a table, so that a crash loses nothing. **Recovery**
reads the log back into a memtable at startup.

Reading a key means consulting the memtable, then level 0, then each
level below, and stopping at the first entry found. Two things make
that cheap. A table's **bloom filter** answers "this key is certainly
not in this table" from a few bits per key, so most tables are never
read. A table's **index block** names, for each of its **data
blocks**, a key that sorts at or after everything in it, so a lookup
fetches one block rather than the file.

**Level 0 is different from every other level.** Its tables overlap,
because each one is a memtable written out whole and two of them cover
the same keys. So a read that misses the memtable consults every
level-0 table, and level 0 is compacted on a table count rather than a
byte size. Every deeper level's tables are disjoint, a read consults
at most one of them, and the trigger is bytes.

**Nothing in this package performs any input or output.** A writer
appends to a buffer the caller owns. A reader says which bytes of
which table it wants and waits for them. A compaction planner answers
a plan rather than running one. The same code therefore serves a store
on a disk, a store in object storage and a test with three lists in
memory, and the only thing that differs is who answers the requests.

`lsmmem.default_options()` carries LevelDB's defaults, which are also
RocksDB's where the two agree.

| Setting | Default | What it decides |
| --- | --- | --- |
| `memtable_bytes` | 4 MB | How much is flushed at once, and how much is at risk in a crash |
| `block_bytes` | 4 KB | How many bytes a point lookup fetches |
| `restart_interval` | 16 keys | How often prefix compression restarts inside a block |
| `bloom_bits_per_key` | 10 | About a one per cent false positive rate; 0 is no filter |
| `l0_trigger` | 4 tables | How many level-0 tables before a compaction is due |
| `level_multiplier` | 10 | How much larger each level is than the one above |
| `base_level_bytes` | 64 MB | The byte budget of level 1 |
| `max_key_len`, `max_value_len` | — | A key or value longer than these is refused |

An SSTable is laid out smallest to largest, and the format is
LevelDB's.

| Part | Contents |
| --- | --- |
| Data block | Entries, prefix-compressed within a restart group, then the restart offsets, their count, and a 4-byte CRC |
| Bloom block | `bloom_bits_per_key` bits per key, or absent |
| Index block | One entry per data block: a separator key, the block's offset and its length, in the data-block encoding |
| Footer | Fixed size, last in the file: where the index and bloom blocks are, the entry count, the key and sequence ranges, a format version, a CRC and a magic number |

**The footer is a fixed size and it is at the end.** A reader that
knows the file's length knows exactly which bytes to ask for first, in
one request, having read nothing.

## Install

```
novo pkg add lsm-nv
```

## Example

```novo
use std.list
use std.bytes
use lsmmem
use lsmerror
use lsmsst

fn main() [io]
    // The sizes the store runs under: a 4 MB memtable, 4 KB blocks,
    // ten bloom bits per key, and the rest of LevelDB's defaults.
    let opts = lsmmem.default_options()

    // Two writes, each with the sequence number the store gave it.
    let m0 = lsmmem.memtable()
    match lsmmem.insert(m0, lsmmem.put(bytes.to_byte_list(bytes.from_str("alpha")),
                                       bytes.to_byte_list(bytes.from_str("1")), 1))
        Err(f) => println(lsmerror.message(f))
        Ok(m1) =>
            match lsmmem.insert(m1, lsmmem.tombstone(bytes.to_byte_list(bytes.from_str("beta")), 2))
                Err(f) => println(lsmerror.message(f))
                Ok(m2) =>
                    // Fix the contents, then write them into a buffer
                    // the caller owns. Nothing here opens a file.
                    let frozen = lsmmem.freeze(m2)
                    match lsmsst.write_table([], lsmmem.entries(frozen), opts)
                        Err(f) => println(lsmerror.message(f))
                        Ok(w)  =>
                            // `w.bytes` is what the host writes to a file.
                            // `w.footer` is what it records in the manifest.
                            println("${list.len(w.bytes)} bytes, ${w.footer.entries} entries")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: lsm-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `lsmmem` | The entry, the one comparator every other module is written against, the memtable, the options, and the key range. |
| `lsmsst` | The SSTable: a writer that appends to a buffer, a reader that asks for byte ranges, the footer, the index, and the bloom filter. |
| `lsmwal` | The write-ahead log: its header, its record format, its cumulative checksum, and the recovery that stops in the right place. |
| `lsmlevel` | The manifest of tables by level, the byte budgets, and the planner that answers which tables to compact next. |
| `lsmmerge` | One ordered stream out of several, with the version and tombstone rules applied exactly once. |
| `lsmerror` | Why a run of bytes is not the table it claimed to be, and which of those reasons a recovery should stop at rather than refuse. |

## How to choose an entry point

**Writing a table.** `lsmsst.write_table` takes every entry at once
and is what a memtable flush uses. `lsmsst.writer`, `add` and `finish`
are the same thing one entry at a time, for a compaction whose input
arrives from a merge. `lsmsst.table_len` counts the bytes the write
will produce without producing them, for a caller pre-allocating a
buffer or checking that the output fits.

**Reading a table.** `lsmsst.reader` asks and the caller performs:
`wants` says what it needs, `feed_length` and `feed` hand it over, and
`is_open` is true once the footer and the index are in. After that,
`bloom_says_absent` settles most lookups with no read at all,
`block_for` names the one block a key could be in, and
`blocks_for_range` names the blocks a scan covers.

**Merging.** `lsmmerge.merge` is the state machine: it answers
`LsmMergeWant(i)` when source `i` runs dry, and the caller feeds that
source and steps again. Use it when the sources are files, because the
caller can then fetch several at once. `lsmmerge.merge_all` is the same
merge over lists already in memory, which is what a test and a
memtable-only read use.

**Planning a compaction.** `lsmlevel.pick` answers the compaction to
run next. `pick_with` is the same with a snapshot boundary the caller
supplies. `plan_range` is a compaction over a key range the caller
chose. A plan with no inputs means there is nothing to do.

## The rules a user needs

1. **There is one comparator: by key ascending, then by sequence
   number descending.** `lsmmem.compare_entries` is it. With sequence
   descending the first entry for a key is the current one, so every
   reader is "find the key, take the first" and no reader has to scan
   a run of versions to its end.
2. **Keys are compared unsigned, byte by byte, shorter first on a
   prefix.** `lsmmem.compare_keys` is that comparison on its own, for
   the manifest and the planner, which compare keys without having
   entries. Nothing here interprets a key, so a stored order does not
   depend on anyone's locale. An empty key is refused with
   `LsmEmptyKey`.
3. **A tombstone is an entry, not an absence.** `lsmmem.get` answers
   the entry, tombstone included. A caller that treated a tombstone
   and a miss alike would stop the read at the memtable on a delete,
   which is right, and stop it there on a miss, which is wrong.
4. **A tombstone may only be dropped at the bottom level.** A
   compaction that dropped one higher up resurrects the key from the
   older copy at a level it did not read, silently and minutes later.
   `lsmlevel.plan_may_drop_tombstones` decides, and `lsmmerge.merge`
   takes the answer as an argument rather than deciding for itself.
5. **Source order is precedence, and source 0 is the newest.** Two
   entries with the same key and the same sequence number can only be
   one write replayed into two places, and the earlier source wins.
   Every other tie goes to the comparator in rule 1.
6. **`at_seq` is what makes a read a snapshot.** A reader holding a
   sequence number sees the store as it was, and a write that landed
   afterwards is invisible to it. Pass the store's current sequence
   for an ordinary read.
7. **A compaction may drop an older version only when no live reader
   sits between it and the newer one.** `LsmPlan.oldest_snapshot`
   carries that boundary. A store with no readers passes its current
   sequence, and everything older than the newest version goes.
8. **Level 0's tables overlap and every other level's do not.**
   `lsmlevel.is_overlapping` is the predicate, and
   `lsmlevel.read_path` answers the whole consult order for a key in
   one call.
9. **A torn tail is not corruption.** Half a record is the ordinary
   way a crash ends a log. `lsmerror.ends_recovery` is true for a torn
   record and for a stale generation, and false for everything else. A
   recovery that refused the log in either case would discard every
   committed write in it.
10. **A checkpoint bumps the log's salts; it does not truncate the
    log.** Every record repeats the header's two salts, and recovery
    stops at the first record whose salts are stale. There is no
    window in which the log has been emptied and the tables are not
    yet durable. `lsmwal.truncate_to` says where the good records end.
11. **A WAL record's checksum folds in the record before it.** A
    record lifted out of another log and dropped into this one is
    detected. A per-record checksum would accept it.
12. **A frozen memtable refuses writes with `LsmMemtableFrozen`.** A
    flush must see fixed contents, or the table it writes will not
    match its own footer. Freeze, swap in a fresh memtable, then
    flush.
13. **`LsmRange` includes both ends; a scan range does not.** A
    table's range is the smallest and largest keys it actually holds.
    A scan takes a `from` and an exclusive `to`, and an empty `to`
    means the end of the keyspace. The two are never the same type.
14. **A fault names a table id and a byte offset, never a path.** This
    package never sees a filename. The caller maps the id back. A
    block CRC failure carries the block's offset and length, so a
    store that can re-fetch one block does not have to discard the
    file.

## What is not included

- **Any input or output.** No file is opened, no byte is read, no
  thread is started and no lock is taken. A reader asks and the caller
  performs.
- **A block cache.** Which `LsmNeedBytes` are answered from memory is
  the caller's decision, and a cache is state that outlives any one
  read.
- **A snapshot registry.** `LsmPlan.oldest_snapshot` is a number the
  caller supplies. Tracking which readers are live is the store's
  bookkeeping.
- **Compression.** Blocks are stored as they are written. A codec
  between the block encoder and the buffer is an addition that changes
  the format version, and it is not in this release.
- **A device build.** A memtable holds two heap byte strings per
  entry, and a bloom filter at ten bits per key is more memory than a
  microcontroller has. A device that logs to flash wants a ring
  buffer, which is
  [bbqueue-nv](https://novo-lang.org/packages/bbqueue-nv).
- **A merge operator.** A write here sets or removes a key. Read-
  modify-write on the store's side, as RocksDB's merge operators do,
  would need the store to interpret values, and nothing here
  interprets a value.
- **Column families and secondary indexes.** One keyspace, one
  ordering.

## Related packages

- [btree-nv](https://novo-lang.org/packages/btree-nv) is the other
  storage shape: an ordered map updated in place, in fixed-size pages.
  Take it for a read-heavy store whose writes are scattered. Take this
  one for a write-heavy store, where appending and merging later costs
  less than updating a page per write.
- [pager-nv](https://novo-lang.org/packages/pager-nv) is the page file
  and its write-ahead log, for the B-tree. Its log has the same three
  decisions as the one here: a generation rather than a truncation, a
  cumulative checksum, and the commit marker inside the record. It
  logs pages where this logs entries.
- [sqlite-nv](https://novo-lang.org/packages/sqlite-nv) reads and
  writes a real SQLite database file, which is a B-tree and not this.
- [crc-nv](https://novo-lang.org/packages/crc-nv) is the CRC-32C every
  block, footer and log record here is checksummed with. It is a
  dependency rather than a copy, because a CRC table transcribed twice
  is a CRC table that differs in one entry.
- [varint-nv](https://novo-lang.org/packages/varint-nv) is every
  length and offset in a block. Two variable-length integer decoders
  written separately disagree at the 64-bit boundary, which only shows
  on values nobody's fixtures have.
- `std.collections` in the standard library has `OrderedMap`, an
  ordered map in memory with nothing underneath it. Take it when the
  whole collection fits in memory and nothing has to survive the
  process.

## Tests

```bash
novo test tests/lsmmem_tests.nv     # 14 tests: the order, the memtable, the ranges
novo test tests/lsmsst_tests.nv     # 14 tests: the table format and the reader protocol
novo test tests/lsmwal_tests.nv     # 12 tests: the log, its checksum and its recovery
novo test tests/lsmlevel_tests.nv   # 13 tests: the manifest and the compaction planner
novo test tests/lsmmerge_tests.nv   # 13 tests: the merge and the version rules
```

The reference implementations are LevelDB and RocksDB. The table
format is LevelDB's, and the level sizing and the defaults are both
projects' where they agree. No test opens a file: a reader is fed a
fixture the test built, a merge is fed lists, and a planner is asked
about a manifest the test wrote out. The suite asserts that the
sequence half of the order is descending, that a tombstone survives a
compaction above the bottom level, that a read consults every level-0
table and one table per level below, that a torn record ends recovery
rather than failing it, that a record from a previous generation ends
it too, and that a frozen memtable refuses a write.

The tests compile today and fail at run, each on the
`not implemented: lsm-nv.<module>.<fn>` panic that is its body. That is
the expected state of an interface release. They turn green one at a
time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `lsmmem.default_options` | no |
| `lsmmem.compare_entries`, `.compare_keys`, `.keys_eq`, `.prefix_end`, `.separator` | no |
| `lsmmem.put`, `.tombstone`, `.range_tombstone`, `.is_tombstone`, `.range_covers`, `.entry_bytes` | no |
| `lsmmem.memtable`, `.insert`, `.insert_batch`, `.get`, `.scan`, `.should_flush`, `.freeze` | no |
| `lsmmem.entries`, `.count`, `.key_range` | no |
| `lsmmem.range`, `.ranges_overlap`, `.ranges_union`, `.range_holds` | no |
| `lsmsst.writer`, `.add`, `.finish`, `.write_table`, `.table_len` | no |
| `lsmsst.footer_size`, `.magic`, `.format_version`, `.block_crc` | no |
| `lsmsst.reader`, `.wants`, `.feed_length`, `.feed`, `.is_open`, `.footer` | no |
| `lsmsst.block_for`, `.block_request`, `.decode_block`, `.find_in_block` | no |
| `lsmsst.bloom_says_absent`, `.blocks_for_range`, `.block_count`, `.index_of` | no |
| `lsmsst.bloom_bytes`, `.bloom_hash`, `.varint_len`, `.decode_varint` | no |
| `lsmwal.header_size`, `.record_header_size`, `.new_header`, `.checkpoint` | no |
| `lsmwal.write_header`, `.read_header`, `.write_record`, `.write_batch`, `.batch_len` | no |
| `lsmwal.recovery`, `.replay`, `.is_done`, `.committed`, `.stopped_by`, `.truncate_to` | no |
| `lsmwal.record_crc`, `.fold_checksum`, `.is_current`, `.read_record` | no |
| `lsmlevel.manifest`, `.table_ref`, `.add_table`, `.remove_table`, `.apply`, `.tables_at` | no |
| `lsmlevel.is_overlapping`, `.tables_holding`, `.read_path` | no |
| `lsmlevel.level_bytes`, `.level_budget`, `.level_pressure`, `.total_bytes`, `.read_amplification` | no |
| `lsmlevel.pick`, `.pick_with`, `.plan_range`, `.plan_is_empty`, `.plan_bytes` | no |
| `lsmlevel.plan_may_drop_tombstones`, `.plans_conflict`, `.reason_text` | no |
| `lsmmerge.merge`, `.merge_for`, `.step`, `.feed`, `.close`, `.is_done`, `.waiting_for` | no |
| `lsmmerge.would_drop`, `.drop_reason`, `.merge_all`, `.newest_of`, `.is_visible` | no |
| `lsmerror.fault`, `.kind_name`, `.message`, `.is_corruption`, `.ends_recovery` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
