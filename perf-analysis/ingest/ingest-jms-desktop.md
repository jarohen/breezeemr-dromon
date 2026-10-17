# dromon fhir-search-bench: ingest journal (James's desktop)

The laptop journal (`ingest-jms-laptop.md`) is the record of how we got here; this one re-runs its connection comparison at PR #1's scale on a faster machine.

## Setup

As `ingest-jms-laptop.md`'s setup, except:

- **`clojure`, not `clj`**: `clj` needs `rlwrap`, which isn't installed here.
- **Schemas before `bb synthea`**: the bench's classpath names `fhir/malli/uscore8`, so `bb synthea` won't start without them either.
- **JDK 26** (OpenJDK, HotSpot C2) via `JAVA_HOME`.
- **100x dataset:** `bb synthea :population 1200`, then `bench/run ... :max-resources 2000000`: 1,202 bundles, 943,116 docs (440,700 Observations), 3.6 GB of Synthea output. PR #1 ran 1,300 patients / 1.13M.
- **Heap:** the alias's own `-Xmx28g`.
- **Load only:** `:skip-search? true`.
- **No cooldown, one run each:** a 100x load is long enough to be effectively warm, and the desktop doesn't throttle.

Machine: AMD Ryzen 7 5800X (8 cores / 16 threads), 62 GB RAM, NVMe, JDK 26.0.2, XTDB 2.2.0-beta3, local log + local storage.

### This branch's code
As the laptop journal's, plus: `load-dataset-pooled!` logs loaded / total, elapsed and res/s each time the loaded count crosses a multiple of 50k.

## Headline results

100x dataset, 4 loader threads, `max-tx` 2000, one `:put-docs` per doc, fresh JVM per run, ADBC run first then JDBC, 2026-10-18 ~00:05.

| connection | load | docs/s |
|---|---:|---:|
| JDBC pool (pgwire) | 102.95 s | 9,161 |
| ADBC conn (in-process) | 51.29 s | **18,388** |

- **ADBC is 2.0x the pool.** Same write path, transaction shapes and concurrency (4 loader threads, each transaction opening its own `open-adbc-conn` or borrowing a pooled connection); the connection is the only difference.
- **Steady throughout:** after the first 100k, ADBC held 17.5-18.6k docs/s and JDBC 8.3-8.8k docs/s cumulative, with no slowdown as the tables grew.
- All 943,116 docs and 440,700 Observations landed in both runs; no chunk fell back to batch.

### Against the laptop (row (b), `:put-docs` per doc, 10x, cold)

| connection | laptop | desktop | desktop / laptop |
|---|---:|---:|---:|
| JDBC pool | ~3,500 | 9,161 | 2.6x |
| ADBC conn | 8,395 | 18,388 | 2.2x |
| ADBC / JDBC | 2.4x | 2.0x | |

The machine moves the absolute numbers by more than the connection does; the connection's ratio holds within ~20% across the two.

## Where the time goes: JFR on both connections (30x)

`:max-resources 300000` over the same 100x output (first 359 bundles, 301,390 docs), one run each with `-XX:StartFlightRecording=settings=profile,jdk.FileForce#threshold=0ms`. Loads under JFR: ADBC 16.10 s (18,715/s), JDBC 37.42 s (8,053/s), so profiling costs little.

Each on-CPU sample (Java threads, during the load) goes to the innermost frame owned by dromon (`fhir_store*`), the XT client (`xtdb.api`, `xtdb.adbc`, `TxOp` construction and close, pgjdbc, Hikari) or the XT server (the rest of `xtdb.*`, and pgwire/log/indexer threads). Library frames (Arrow, transit, Malli, Clojure) go to their nearest owning caller.

| on-CPU samples | ADBC | JDBC |
|---|---:|---:|
| dromon client | 1,321 (28%) | 1,312 (19%) |
| XT client | 1,179 (25%): build `TxOp$PutDocs` from rows 620, free it 559 | 748 (11%): transit-encode the COPY, pgjdbc |
| XT server | 2,074 (44%): encode tx for the log 669 (on the loader thread), indexer 686, compactor/storage 609, log 89 | 4,702 (69%): build tx-op from transit 1,320, encode for log 731, compactor/storage 707, indexer 682, pgwire 568, free tx-op 490, log 95 |
| unattributed | 93 (2%) | 76 (1%) |
| total | 4,670 | 6,838 |

- **The server's own work is the same on both connections:** indexer ~685, compactor/storage ~610-710, log encoding ~670-730 samples. JDBC's extra ~2.2k samples are all transport: transit encode on the client and decode on the server, plus pgwire.
- **One `TxOp` per doc is XT's biggest single cost:** building and freeing a one-row relation per document takes ~1.2k samples on ADBC and ~1.8k on JDBC. `RelationWriter.close` under `TxOp.close` (Arrow buffer release across the nested struct children) is 559 samples, ~12% of all ADBC CPU.
  - `TxOp.PutDocs.openFromRows` -> `Relation.openFromRows` (beta3 `Relation.kt:269`) builds an empty `Relation` and `writeRow`s each doc into it, so the relation's columns come from the doc. Every nested key of a FHIR resource, plus dromon's `_tokens` columns, becomes a vector with its own validity/offset/data buffers from the netty pool. `TxWriter` copies the relation into the log message, then `TxOp.close` frees every buffer.
  - So the cost scales with columns per op, not docs per op: at one doc per `:put-docs`, every doc pays for its whole schema twice, once to build and once to free. The laptop's grouped path, (c), had a table's docs in a transaction share one relation.
- **Loader threads:** ADBC loaders were on-CPU for ~32.6 of 64.4 thread-s, and JDBC loaders for ~21.4 of 149.7. The rest is waiting on the commit. JFR caps how many threads it samples per period, so these are rough.
- **Not in the samples:** JIT and GC threads.

### fsync (local log, btrfs on NVMe)

| | LOG | REPLICA_LOG | total over the load |
|---|---|---|---:|
| ADBC | 442 forces, mean 4.59 ms | 484, mean 4.51 ms | 4.2 s of 16.1 s |
| JDBC | 431 forces, mean 23.0 ms | 615, mean 21.8 ms | 23.3 s of 37.4 s |

Two forces per transaction, as on the laptop. On ADBC that's a floor of ~9 ms per transaction, about 1 s of each loader's 16 s. JDBC's ~5x slower fsync is unexplained: the force counts are similar, and pgwire's per-COPY temp file goes to `/tmp`, which is tmpfs here.

## Wall-clock caveat: fsync latency swings between runs

The local log's mean fsync was 4.5 ms in the ADBC JFR run, 22 ms in the JDBC one and ~34 ms in I1's (below), on the same btrfs volume (NVMe, 72% full, `discard=async`) with similar force counts. Per-fsync latency has therefore varied ~7x between runs, and it is in every wall-clock number in this journal, the ADBC/JDBC headline included. Sample counts are unaffected.

## I1: one op per table in each run of `:put-docs`

`coalesce-doc-ops` in `transact-transaction` (`fhir-store-xtdb2/src/fhir_store_xtdb2/core.clj`) partitions a transaction's own ops into consecutive runs of the same op type, then turns each run of `:put-docs` (or `:delete-docs`) into one op per table, each carrying that table's docs in their original order. Unconditional on this branch.

- **Within a run of `:put-docs`, ops on different tables commute.** Grouping by table keeps every same-table pair in order, so two writes to one resource keep their order too. Nothing outside the run moves.
- **An unguarded transaction is one run of deletes, then one run of puts** (`transact-transaction` sorts DELETE -> POST -> PUT), so it groups fully: 45 docs per op over 60 Synthea bundles (55,857 docs). Merging only consecutive same-table ops would give 2.1 (median 1): Synthea interleaves ~19 types per patient.
- **`ifMatch` guards split the put runs themselves.** A PUT carrying `request.ifMatch` (FHIR's "only if still at this version") is checked up front against a bulk read of current versions, then again at commit by an `ASSERT EXISTS (... version matches)` placed directly before its PUT, catching a write that lands between the read and the commit. That `ASSERT` ends the run before it, so a guarded PUT keeps its guard immediately ahead of it, at the cost of its run being short. The bench sends no `ifMatch`, so its transactions do no reads and group fully.
- **Lifecycle ops still follow the store's own**, and `own-op-count` counts after grouping, so a failing lifecycle op is still told apart.

### First measurement (ADBC; an earlier cut grouping puts by table over the whole transaction: identical to the above on this workload)

| ADBC | one op per doc | one op per table |
|---|---:|---:|
| 100x load | 51.29 s (18,388/s) | 64.16 s (14,699/s) |
| 30x load, under JFR | 16.10 s (18,715/s) | 18.07 s (16,676/s) |
| 30x on-CPU samples, total | 4,670 | 2,829 |
| - dromon | 1,321 | 1,280 |
| - build tx-op (`openFromRows`) | 620 | 252 |
| - encode tx for log | 669 | 137 |
| - free tx-op (`TxOp.close`) | 559 | 14 |
| - indexer | 686 | 325 |
| - compactor/storage | 609 | 611 |
| 30x fsync, mean | 4.5 ms | ~34 ms |

- **CPU fell by 40%**, nearly all of it XT's per-op work: freeing the per-op relations all but disappears, and the indexer halves.
- **Wall time rose, because the run was fsync-bound**: 27 s of fsync across the two logs over an 18 s load, with loaders on-CPU ~25% of their time. Grouping doesn't plausibly slow fsync (if anything, the log messages shrink), so the wall-clock comparison needs fsync taken out first. Pending: tmpfs data dir vs interleaved NVMe runs.

Raised against XT as xtdb/xtdb#6214: a transaction of single-document `:put-docs` should not build a relation per document.
