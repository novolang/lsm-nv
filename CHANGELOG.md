# Changelog

All notable changes to lsm-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.3 — 2026-09-25

The package builds with novo 0.11.  Every body is still `todo()`.

- The lock file moves crc-nv 0.1.4 to 0.1.5, leb128-nv 0.1.5 to 0.1.7,
  varint-nv 0.1.4 to 0.1.6 and zigzag-nv 0.1.4 to 0.1.5.  leb128-nv
  0.1.5 and varint-nv 0.1.4 write into lists through names that are not
  declared `var`, which novo 0.11 refuses (E2038), so this package did
  not build with novo 0.11 against them.  No requirement in the manifest
  changed.

## 0.0.2 — 2026-09-15

- README rewritten to the package README style guide (docs/writing-a-readme.md).
- `LsmFault` now declares the `impl Error` its own `Result` positions
  require.  `Result<T, E>` has carried the bound `E: Error` since SPEC
  § 3.4, and the compiler enforced it only when `E` was declared in the
  module that named it — so `Result<_, lsmerror.LsmFault>` was accepted
  across modules with no impl anywhere.  The impl is the signature this
  package always meant; nothing else about the interface changed.

## 0.0.1 — 2026-09-11

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `lsmmem` — `LsmEntry` with a sequence and a kind, `compare_entries`
  (the package's one comparator), the memtable as a sorted value,
  `LsmRange` inclusive at both ends, and `separator` / `prefix_end`,
  the two key computations an index and a prefix scan need.
- `lsmsst` — the LevelDB table format: a writer that refuses an
  out-of-order entry, a reader that answers `LsmNeedBytes` instead of
  reading, block and range lookups, and `bloom_says_absent`, whose
  name says which way the asymmetry goes.
- `lsmwal` — a header with salts, records with a cumulative checksum
  and the commit marker inside them, and a recovery that stops at a
  torn tail and keeps everything committed before it.
- `lsmlevel` — the manifest, `read_path`, the level budgets, and
  `pick`, which answers a plan and runs nothing.
- `lsmmerge` — a fed merge over sources in precedence order, with
  `would_drop` published so the invisible decision is testable.
- `lsmerror` — faults that name a block rather than a file, and
  `ends_recovery`, which separates a torn log from a corrupt one.

### Known

- **The order is by key ascending, then by sequence DESCENDING**, and
  it is one public function because every LSM correctness bug is that
  comparator applied inconsistently somewhere.
- **A tombstone may only be dropped at the bottom level**, and the
  decision is `lsmlevel.plan_may_drop_tombstones`, passed to
  `lsmmerge.merge` as an argument so the merge cannot decide it.
- **The reader asks and the caller performs**, in byte ranges rather
  than pages — deliberately a different type from btree-nv's
  `PageRequest`, because a page and a range are different shapes.
- **The merge is fed, not pulling**: a pulling iterator would need a
  trait with an effect row, which a `core` function cannot call and
  stay inside its budget.
- **A torn tail and a stale generation end recovery and are not
  corruption**; `ends_recovery` exists so treating them as corruption —
  and losing every committed write in the log — cannot be written by
  accident.
- **Level 0 overlaps and every other level does not**, so its trigger
  is a table count and theirs is bytes; `is_overlapping` is the
  predicate and `read_path` answers the consult order.
- **A compaction planner answers a plan and runs nothing**, which is
  what makes a policy testable without a file, a thread or a clock.
- **The WAL shape is pager-nv's and the dependency is not**: pager-nv
  is `host`, so the three decisions are named in `lsmwal`'s module
  comment rather than imported.
- **Two `core` dependencies**, crc-nv and varint-nv.
- **No device claim**: a memtable is heap byte strings and a bloom
  filter is more memory than a microcontroller has.
