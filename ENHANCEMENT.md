# Per-statement compute and storage metering

This fork adds one thing to SQLite: after every SQL statement, it tells you
what that statement cost. Upstream SQLite has no billing or metering
interface — it is an in-process library with no notion of a tenant or an
account — and what it does have is a scattering of counters and quotas that
a caller has to assemble into an answer. This makes the answer a first-class
event.

Branch `cu+su`, on top of upstream 3.54.0 (`8364e34cb`).

| commit | what |
|---|---|
| `5077e3aae` | the metering event, compute units, storage deltas, rollback semantics |
| `f663bd3f9` | charge compute for the page write-backs a rollback performs |

## Using it

```c
static int onMeter(unsigned t, void *pArg, void *pStmt, void *pX){
  const sqlite3_meter *m = (const sqlite3_meter*)pX;
  printf("%s: cu=%lld  storage %+lld bytes\n",
         sqlite3_sql((sqlite3_stmt*)pStmt), m->cu, m->suLiveDelta);
  return 0;
}

sqlite3_trace_v2(db, SQLITE_TRACE_METER, onMeter, 0);
```

The callback fires as each statement finishes. `pX` points to an
`sqlite3_meter`, valid for the duration of the call:

| field | meaning |
|---|---|
| `cu` | weighted compute units |
| `nVmStep` | virtual machine instructions executed |
| `nPageRead` | pages read from disk |
| `nPageWrite` | pages written to disk |
| `nPageUndo` | pages written back to undo a change |
| `suAlloc` | bytes the database file occupies afterwards |
| `suAllocDelta` | signed change in that figure |
| `suLive` | bytes holding actual data afterwards |
| `suLiveDelta` | signed change in that figure |

Totals sum over the main database and every attached database.

From the shell:

```
sqlite> .trace stdout --meter
sqlite> DELETE FROM t WHERE a > 100;
DELETE FROM t WHERE a > 100;
  -- cu 2408 (vm 1208, read 0, write 6, undo 0)
  -- su live -323584 -> 94208, alloc +0 -> 417792
```

## Two kinds of storage

`suAlloc` counts every page in the database file, so it tracks what the
filesystem has handed out. `suLive` counts only pages holding data, so
freeing space shows up immediately even though the file has not shrunk.
A `DELETE` moves one and not the other; a `VACUUM` does the reverse:

| statement | suLiveDelta | suAllocDelta |
|---|---|---|
| `INSERT` 2000 rows of ~800 bytes | +1638400 | +1638400 |
| `SELECT count(*)` | 0 | 0 |
| `DELETE` 75% of the rows | **−1228800** | 0 |
| `VACUUM` | 0 | **−1228800** |

Both figures were checked against `PRAGMA page_count`, `PRAGMA page_size`
and `PRAGMA freelist_count` and agree exactly, under five configurations:
rollback journal with `auto_vacuum` set to `NONE`, `FULL` and `INCREMENTAL`,
WAL mode, and a database held in memory.

## The compute cost model

```
cu = nVmStep   * SQLITE_CU_WEIGHT_VMSTEP      (default 1)
   + nPageRead * SQLITE_CU_WEIGHT_PAGEREAD    (default 100)
   + (nPageWrite + nPageUndo) * SQLITE_CU_WEIGHT_PAGEWRITE  (default 200)
```

The defaults treat a page read as worth a hundred instructions and a page
write as worth two hundred, on the grounds that I/O rather than bytecode
dominates the real cost of a query. The weights are compile-time options,
and every raw component is reported next to `cu`, so an application that
disagrees can recompute the figure without rebuilding the library.

`nVmStep` is kept per statement and is exact.

## Rollback

Work that is rolled back is charged for compute and refunded for storage.

A statement that fails, or whose transaction is later rolled back, really
executed its bytecode and really moved its pages, so it is billed for them.
The storage it claimed never became durable, so the rollback reports a delta
handing it back. Summing `suAllocDelta` or `suLiveDelta` over every metered
statement therefore always tracks the real size of the database.

| scenario | compute charged | storage |
|---|---|---|
| `BEGIN; INSERT 1.6MB; ROLLBACK` | 52017 vm steps on the insert | insert +1638400, rollback −1638400 |
| `INSERT` failing on a `UNIQUE` constraint | 11 vm steps | 0 |
| `BEGIN; DELETE; ROLLBACK` | 5711 vm steps on the delete | delete −1556480, rollback **+1556480** |
| `SAVEPOINT`; `DELETE`; `ROLLBACK TO` | 5861 vm steps on the delete | delete −1597440, rollback **+1597440** |

Undoing a transaction also costs I/O, and that was missed at first. When a
transaction has already pushed dirty pages out to the database file, rolling
it back has to read the originals out of the journal and write them into the
database again; on a large transaction against a small page cache that is
the dominant cost of the statement. The counters behind the meter come from
`PAGER_STAT_WRITE`, which the journal playback path never touches, so a
`ROLLBACK` that rewrote 798 pages reported the three instructions it
executed and nothing else. Those write-backs are now counted:

| statement (`cache_size=10`, `UPDATE` of 4000 rows) | nPageWrite | nPageUndo | cu |
|---|---|---|---|
| `UPDATE` inside a transaction | 794 | 0 | 271009 |
| `ROLLBACK` | 0 | **798** | **159603** |
| `COMMIT` instead, for comparison | 7 | 0 | 1403 |
| `ROLLBACK TO` a savepoint | 0 | **798** | **159604** |

A rollback that never had to touch the database file reports zero, which is
the honest answer: a transaction small enough to be undone entirely within
the page cache really does no write-back I/O.

## How it works

Metering is armed in `sqlite3Step()` when the statement starts running,
which records the counter baselines that isolate this run, and the report is
emitted when the statement finishes, alongside the existing profile
callback. Both are driven from the same `Vdbe` flag, so a statement that is
not metered pays for one branch.

Storage is read inside `sqlite3VdbeHalt()`, before it commits or rolls back.
That is not a matter of taste: `sqlite3BtreeGetMeta()` needs page 1 of the
database loaded to read the freelist, and `sqlite3PagerPagecount()` asserts
that a read transaction is open, so once the transaction closes the size is
no longer available. `sqlite3VdbeHalt()` is the last moment at which it is.

Sampling before the commit creates three problems, each handled where it
arises:

- **A commit can change the size.** `VACUUM` rebuilds the file and
  auto-vacuum truncates it, both inside the commit, after the sample. The
  committed size is therefore re-read from the file afterwards and the
  difference is attributed to the statement responsible. The cached page
  count is no use here — it is only refreshed when a transaction opens, so
  after a `VACUUM` it still describes the old file.
- **A rollback invalidates the sample.** Each database remembers where its
  open transaction started, so a rollback can emit the exact compensating
  delta. The hook sits in `sqlite3RollbackAll()`, the one choke point every
  full rollback passes through, including an explicit `ROLLBACK` by way of
  `OP_AutoCommit`. The compensation is parked on the connection rather than
  applied directly, because a rollback can happen deep inside
  `sqlite3VdbeHalt()`, below the point where the statement to charge is
  known; the report picks it up on its way out.
- **A statement-level rollback also invalidates it.** There the surrounding
  transaction is still open, so the reverted state is simply re-sampled and
  the earlier delta cancels out.

Rollback page write-backs are counted in a new `PAGER_STAT_UNDO` slot rather
than folded into `PAGER_STAT_WRITE`, because that counter is what the public
`SQLITE_DBSTATUS_CACHE_WRITE` reports and its meaning should not move
underneath existing callers.

### Files

| file | change |
|---|---|
| `src/sqlite.h.in` | `SQLITE_TRACE_METER`, `sqlite3_meter`, documentation |
| `src/sqliteInt.h` | per-database storage baselines, pending compensation, cost weights |
| `src/vdbeInt.h` | per-statement meter state |
| `src/vdbeapi.c` | arm metering, emit the report at end of statement |
| `src/vdbeaux.c` | sampling, rollback compensation, post-commit correction |
| `src/main.c` | rollback hook in `sqlite3RollbackAll()` |
| `src/pager.c`, `src/pager.h` | count and expose undo page write-backs |
| `src/shell.c.in` | `.trace --meter` |

## Limitations

**Page I/O attribution.** Page reads and writes are counted by the pager,
which does not know which statement asked for them. A connection that
interleaves `sqlite3_step()` calls on two statements will charge I/O to
whichever finishes first. `nVmStep` is unaffected.

**One delta can land on the next statement.** This happens when a statement
is abandoned part way through, by calling `sqlite3_reset()` after
`sqlite3_step()` has returned `SQLITE_ROW`, and when a `VACUUM` shrinks a
database that is in WAL mode or held in memory — there the new size cannot
be read until the next transaction opens, and reading the file size would be
misleading, so the correction is skipped rather than guessed. Running totals
stay correct; only the attribution of a single delta moves.

**The first metered statement establishes a baseline** for each database it
touches and reports zero delta for it.

**No savepoint-level storage baseline.** A rollback hands back what the
*transaction* claimed. `ROLLBACK TO` a savepoint is handled correctly
because the statement doing it samples the reverted state itself, not
because savepoint boundaries are tracked.

## Testing

`make xdevtest` — the release-level suite, which is the full suite minus the
checks that assume an unmodified source tree. 18 build configurations
including `sanitize` (ASAN/UBSAN), `secure_delete`, `no_lookaside`,
`update_delete_limit` and `extra_robustness`.

```
878125 / 878127 jobs, 15,069,486 tests
failures outside the valgrind configuration: 0
```

Also built clean, with no warnings, under `SQLITE_OMIT_TRACE`,
`SQLITE_OMIT_DEPRECATED`, both together, `SQLITE_THREADSAFE=0`, and
non-default cost weights.

Two gaps, both in the environment rather than the code:

- The suite's valgrind configuration did not run, because valgrind is not
  installed on the build machine. All 1294 reported failures are that
  configuration failing to launch.
- Two `misc7.test` jobs spin forever inside `Tcl_Close` in Tcl 8.6.14, with
  no SQLite frames on the stack. Reproduced identically on an unmodified
  checkout of `8364e34cb` — same output, same stopping point, same exit —
  so it is not caused by these changes. `misc7.test` is excluded from
  `allquicktests`, so the quick suites never exercise it.

## On upstreaming

Nothing here is proposed for upstream. `AGENTS.md` states that the SQLite
project does not accept agentic code, and the sampling points this feature
needs are in `sqlite3VdbeHalt()` and `sqlite3RollbackAll()`, which is a lot
of core to disturb for a feature most callers will never enable.
