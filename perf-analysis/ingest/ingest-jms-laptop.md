# dromon fhir-search-bench: ingest journal (James's laptop)

## Setup

As the repo's instructions (`fhir-search-bench/README.md`, `docs/getting-started.md`), except:

- **Generate the Malli schemas first** (`docs/getting-started.md`, "FHIR Schema Generation"): `fhir-search-bench` depends on `fhir/malli/{uscore8,r4b}`, which are gitignored. The bench README doesn't mention it.
- **JDK 24+, not 21.** The `:xtdb` alias's `--sun-misc-unsafe-memory-access=allow` is rejected on 21; we used JDK 25 via `JAVA_HOME`.
- **10x dataset:** `bb synthea :population 120` then `bench/run ... :max-resources 1000000` (the default 12 patients loads ~7k docs; the 10k cap otherwise truncates).
- **Heap:** the alias asks for `-Xmx28g`; on 15 GB we passed `-J-Xmx10g`.
- **Load only:** `:skip-search? true`.

Machine: i7-8565U laptop (4 cores / 8 threads, thermally limited), 15 GB RAM, JDK 25, XTDB 2.2.0-beta3, local log + local storage, pipelined replica appends ON (store sets it in code).

### This branch's code
- **Writes are always `:put-docs`, one doc per op** (`:delete-docs` for DELETE), whatever `:query-mode` - the SQL `INSERT` write path is gone. Reads are untouched.
- **`DROMON_XTDB_CONN=adbc`** runs each transaction on `xt/open-adbc-conn` over the tenant's node (in-process); unset or `jdbc` uses the store's Hikari pool (pgwire), as shipped. That's the only difference between the two.
- **The bench logs Observations loaded vs expected** after the timer, and takes `:skip-search?`.
- Logs, thread dumps and JFR recordings cited below under `scratchpad/` were not kept.
- The earlier experiments (`DROMON_XTDB_COALESCE_PUT_DOCS`, `:query-mode :xtql`, etc.) were env-gated patches this branch no longer carries; the scripts that drove them weren't kept.

## Headline results

10x dataset (120 Synthea patients, 73,566 docs, 126 txs, one tx per patient bundle split at 2,000 docs, p50 491 docs/tx), 4 loader threads, load only, docs/s.

| write path | JDBC pool, cold | JDBC pool, warm | ADBC conn, cold | ADBC conn, warm |
|---|---:|---:|---:|---:|
| **(a)** one SQL `INSERT` per doc | 3,132 (23.5 s) | 2,262 (32.5 s) ‡ | 5,745 (12.8 s) | 7,569 (9.7 s) |
| **(b)** one `:put-docs` per doc (**this branch**) | ~3,500 (21 s; single runs R3, S1000-b) | not run | 8,395 (8.8 s) | 8,405 (8.75 s) ‡ |
| **(c)** one `:put-docs` per table per tx | 9,655 (7.6 s) | ~14,600 (~5.0 s) | 13,747 (5.4 s) | 23,332 (3.15 s) |

‡ not trustworthy: the laptop throttled. Warm pool (a) loads 3-5 were 32.5 / 25.8 / 36.9 s, slower than cold; warm ADBC (b) loads 1-5 were 13.3 / 6.95 / 8.75 / 7.25 / 15.35 s (peaking ~10.6k/s on load 2).

- **write path:** what one tx's documents become. On the pool, `:put-docs` is a `COPY ... FROM STDIN` in transit-json; on ADBC it's a `TxOp$PutDocs` built from the rows (beta3 `xtdb/adbc.clj:105`).
- **connection:** the store's Hikari pool vs an in-process ADBC connection per tx.
- **cold:** a fresh JVM per load, medians of 3 with a 60 s cooldown. **warm:** 5 loads in one JVM, median of loads 3-5. Each load gets a fresh store, node and data dir either way.
- **dataset:** only R1 is 1x (12 patients, 6,996 docs: 5.54 s, 1,263 docs/s cold, JDBC (a), the bench's defaults). Everything else is 10x.
- Single runs elsewhere varied up to 1.7x with laptop heat and are indicative only. Everything below is the chronological record; C1-C6 there were env-gated experiments in a patch, not this branch's code.

## Write path (as of dromon 98bb43a, `:query-mode :sql`)

Per transaction (`fhir-store-xtdb2/src/fhir_store_xtdb2/core.clj:2720`):

1. One pooled JDBC connection (Hikari -> pgwire -> the in-process node).
2. Per entry: parse method/type/id, sort DELETE -> POST -> PUT, walk the body rewriting `urn:uuid` refs.
3. Per entry: `refuse-unencodable-input!` on type/id; `ensure-declared!` per distinct type (set lookup when warm).
4. Current-version read: only for PUT + ifMatch -> none in this bench.
5. Per POST/PUT: lifecycle pass-through, `encode-resource-doc` (surrogate walk + Malli storage encoder, adds `_tokens` columns), then `extract-and-build-sql` -> one `[:sql "INSERT INTO t (cols...) VALUES (?...)" args]` op. Column list is the doc's own keys, so SQL text varies per doc shape.
6. One `xt/execute-tx` over all ops: `BEGIN READ WRITE` / one statement per op / `COMMIT` / `SHOW LATEST_SUBMITTED_TX`. Synchronous.
7. Response built from in-memory metadata; no read-back.

Bench driver (`fhir-search-bench/src/fhir_search_bench/bench.clj:83`): bundles split at `max-tx` 2000, fixed pool of 4 threads, each running step 1-7. Timer covers load only (dataset parse excluded).

## Runs

| # | when | dataset | docs | txs | docs/tx (p50 / p90 / max) | write path | concurrency | load time | docs/s | notes |
|---|---|---|---:|---:|---|---|---:|---:|---:|---|
| R1 | 2026-10-17 19:43 | 12 patients, seed 42 | 6,996 | ~14-16 (not measured) | not measured | SQL INSERT per doc | 4 | 5.54 s | 1,263 | cold JVM, `-Xmx28g` |
| R2 | 2026-10-17 | 120 patients, seed 42 | 73,566 | 126 | 491 / 1,021 / 2,000 (mean 584) | SQL INSERT per doc | 4 | 28.10 s | 2,618 | `-J-Xmx10g` (verified via jcmd) |

| R3 | 2026-10-17 | 120 patients, seed 42 | 73,566 | 126 | as R2 | `:xtql`: one `:put-docs` (one COPY) per doc (C1) | 4 | 21.17 s | 3,476 | `-J-Xmx10g`; 1.33x R2. Searches: code/category hits match R2; date and value-quantity queries return 0 hits under xtql reads (unexplained - xtql search gap, or put-docs storing those fields differently?) |
| R4 | 2026-10-17 | 120 patients, seed 42 | 73,566 | 126 | as R2 | `:xtql` + C2: one `:put-docs` (one COPY) per table per tx | 4 | 7.22 s | **10,194** | `-J-Xmx10g`; **3.89x R2, 2.93x R3**. No chunk fell back to batch. code/category hit counts identical to R2/R3 (same data landed) |

Derived (R2): ~0.9 s per transaction per thread; ~1.5 ms per document per thread.

## Changes

### C1 (uncommitted, bench only): `:query-mode` option on `bench/run`
`fhir-search-bench/src/fhir_search_bench/bench.clj` - `run` takes `:query-mode` (default `:sql`) and passes it to `create-xtdb-store`. Enables R3. No effect on R1/R2 behaviour.

### C2 (uncommitted, store, env-gated): coalesce `:put-docs` per table
`fhir-store-xtdb2/src/fhir_store_xtdb2/core.clj`, before `xt/execute-tx` in `transact-transaction`. With `DROMON_XTDB_COALESCE_PUT_DOCS` set, and only when every op is a `:put-docs` and there are no lifecycle ops (so `own-op-count` indexing is undisturbed), groups ops by table into one multi-doc `:put-docs` each -> one COPY per table per tx. Order across tables is not preserved; safe here because every doc is a POST with a distinct id. Only active under `:query-mode :xtql`.

## Write path, `:query-mode :xtql` (R3)
Same as above to step 5, except each POST becomes `[:put-docs :<ResourceType> doc]` (`core.clj:2798`). In beta3, `xt/execute-tx` runs each `:put-docs` op as its own `COPY "<schema>"."<table>" FROM STDIN WITH (FORMAT 'transit-json')` (xtdb-api `xtdb/api.clj:182`): one COPY round trip, transit encode and server-side temp file per document. Note `:sql` ops with one arg row go through `jdbc/execute!`, not a batch (`api.clj:218`).

## Notes (out of scope for ingest)
- **Top-k is not in beta3.** `c3a8fab98` "a bounded :sort keeps only its top k rows" (and its prerequisite `5b0382606`, `:order-by` + `:top` -> one `:sort`) landed 28 Sep, after the `v2.2.0-beta3` tag (`591d5c593`, 26 Sep). The bench's sorted top-50 phase-1 query (`ORDER BY effectivedatetime DESC, _id LIMIT 50`) is that shape: at 10x those pages grew ~3x (28-60 ms -> 62-129 ms). Revisit with a beta3+cherry-pick build for an isolated A/B.

### C3 (uncommitted, bench only): `:skip-search?` option on `bench/run`
Skips the 27 search queries so a run times the load alone. Used by the S-series sweep.

## Parked
- **Q6 (parked by James: query side out of scope).** Under `:query-mode :xtql` (R3, R4) the date and value-quantity searches returned 0 hits while code/category matched R2. Either an XTQL read-path gap, or `put-docs` storing `effectiveDateTime`/`valueQuantity` differently from the INSERT path. Only the second would matter for ingest (it'd make R3/R4 write different data than R2).

## Sweep: docs/tx x write path (S-series, load only, `:skip-search? true`)
Logs in `scratchpad/sweep/`, results in `fhir-search-bench/target/runs/`. `max-tx` 2000 points are R2/R3/R4. `max-tx` 1 has no (c): one doc per tx leaves nothing to coalesce.

### S-series attempt 1 (2026-10-17 ~20:00): all 8 runs failed in ~15 s
My bug: the sweep script passed `:query-mode sql` (a symbol), and `create-xtdb-store` asserts the keyword. Nothing was loaded. Fixed to `:$mode`; rerun started ~20:05.

### S1-a observation (single-put txs, SQL INSERT), mid-run at ~7 min
- Still loading after 7 min (vs 28 s for R2): >15x slower, so single-put is the dominant docs/tx effect.
- JVM at ~20% CPU: latency-bound, not CPU-bound.
- 3 thread dumps (`scratchpad/s1a-threads.txt.{1,2,3}`): 3-4 of the 4 loader threads in every dump are blocked in `xt/execute-tx` -> `submit-tx*` at `COMMIT` (beta3 `xtdb/api.clj:245`), waiting on pgwire; 2 `DefaultDispatcher` threads are in `FileChannel.force` from `LocalLog.writeMessages` (`LocalLog.kt:169`) - one per log (source `LOG` and `REPLICA_LOG`).
- `LocalLog` group-commits up to 10 queued messages per `force(true)` (beta3 `LocalLog.kt:195-200`), but only 4 loaders are ever in flight, so each fsync carries <=4 txs, twice per tx (source log then replica log). Throughput here is ~4 / (2 x fsync latency + indexing), i.e. this sweep point measures the disk's fsync latency at concurrency 4 more than XTDB.
- Log growth while sampling: `LOG` ~464 KB/s, `REPLICA_LOG` ~1.06 MB/s; 654 MB of log at ~7 min.

### S1-a stopped (James: stop if we know why)
Killed at 12 min 30 s, still loading - so single-put SQL INSERT at concurrency 4 is >27x slower than R2 (28.1 s) on this machine. Cause per the thread dumps above: loaders wait on COMMIT, log writers in fsync; <=4 txs per group commit, two fsyncs per tx. S1-b (single-put COPY) dropped for the same reason. Replaced with S1-a-c16 / S1-b-c16 (concurrency 16, Tyler's thread count), to see how far LocalLog's group commit (up to 10 msgs per fsync) closes the gap.

Sweep rerun (attempt 2, ~20:20): S100-{a,b,c}, S1000-{a,b,c} at concurrency 4, then S1-{a,b}-c16.

### Single-put on LocalLog: OUT OF SCOPE (James)
No single-tx work has been done against the local log, and the log Tyler's real deployment uses is unknown (James checking with him). S1-*-c16 dropped; the sweep is stopped when it reaches them.

### Sweep results (attempt 2, concurrency 4, 73,566 docs, load only)

| docs/tx (max-tx) | txs | (a) SQL INSERT per doc | (b) COPY per doc | (c) COPY per table per tx |
|---|---:|---:|---:|---:|
| 100 | ~790 | S100-a: 25.85 s, 2,846/s | S100-b: 24.46 s, 3,007/s | S100-c: 12.77 s, 5,761/s |
| 1000 (bundle-limited, p50 491) | ~150 | S1000-a: 34.99 s, 2,102/s | S1000-b: 20.97 s, 3,508/s | S1000-c: 7.25 s, 10,149/s |
| 2000 (bundle-limited, p50 491) | 126 | R2: 28.10 s, 2,618/s | R3: 21.17 s, 3,476/s | R4: 7.22 s, 10,194/s |

- (a) and (b) are flat across docs/tx: their cost is per document.
- **Noise:** S1000-a and R2 split nearly identically (bundle-limited) yet took 35.0 s vs 28.1 s - ~25% run-to-run spread on (a), n=1 each. Treat single-run differences under ~25% as noise until repeated. (c)'s two bundle-limited runs agree to 0.4%.
- (c) is per-tx-bound: 100 -> ~584 docs/tx takes it 5.8k -> 10.2k/s. 1000 vs 2000 cap are identical because Synthea bundles (p50 491, p90 1,021) rarely reach either cap.
- To push (c) further along this axis needs transactions larger than a patient bundle, i.e. merging bundles - which changes what the bench models (one FHIR transaction per patient bundle).

Stopping the sweep: my watcher's `pkill -f` on the sweep script's name matched its own command line and killed itself (exit 144) after killing the sweep; one S1-*-c16 JVM was left and killed by hand at 10 s. No S1-c16 result recorded (out of scope anyway).

## Profile of (c) - R5 (2026-10-17 ~20:23)
Same config as R4 plus `-XX:StartFlightRecording=settings=profile`: 8.62 s, 8,537/s (JFR overhead vs R4's 7.22 s ~ +19%, or noise - see below). Recording `scratchpad/prof/R5-c.jfr`.

Execution samples during the load, by thread group (main thread excluded: it's startup compile + dataset parse, untimed):

| thread group | samples | what it is | top buckets |
|---|---:|---|---|
| `pool-2-thread-*` (4 loaders) | 1,257 | client: the store's transact-transaction | store misc 44% (mostly `transform.clj` 277-332 token columns + `core.clj` 2835/2867 entry/op building), transit write 22%, urn-uuid resolve 15%, malli encode 10%, `clojure.walk` 5%, surrogate check 1% |
| `pgwire-connection-*` | 744 | server: pgwire + tx execution | arrow 41%, transit READ of the COPY payload 24%, other xtdb 22%, pgwire 11% |
| `DefaultDispatcher-worker-*` | 263 | server: log + indexer coroutines | arrow 68%, log 18%, indexer 7% |

- Client:server samples ~ 1,257 : 1,007 - the client-side document processing is the larger half.
- Utilisation is low: 4 loaders x 8.6 s at 10 ms sampling would be ~3,440 samples; got 1,257 (~37% on-CPU). Whole load ~2,260 samples over 8 vCPUs x 860 periods ~ 33%. So at concurrency 4 neither side is CPU-saturated -> tried more concurrency.
- Machine: i7-8565U, 4 cores / 8 threads, laptop, `powersave` governor, on AC. Package temp 78 C straight after runs.

## Concurrency on (c), 1 run each, back to back (~20:30)

| run | concurrency | load | docs/s |
|---|---:|---:|---:|
| C8-c | 8 | 7.99 s | 9,204 |
| C16-c | 16 | 8.87 s | 8,291 |
| C4-c | 4 | 12.00 s | 6,128 |

**C4-c is the same config as R4 (7.22 s) and S1000-c (7.25 s), yet took 12.0 s.** Back-to-back runs on a hot laptop are not reproducible to better than ~1.7x. None of the concurrency differences above can be read from single runs. Next: repeat with cooldown, n=3, report medians.

## Repeats with cooldown (n=3, interleaved, 60 s idle before each; ~20:35-20:55)
Logs `scratchpad/repeat/`. 73,566 docs, bundle-limited txs (126, p50 491 docs/tx), load only.

| config | runs (s) | median | median docs/s | spread | vs (a) |
|---|---|---:|---:|---:|---:|
| (a) SQL INSERT per doc, conc 4 | 24.51, 23.15, 22.68 | 23.15 s | 3,178 | 8% | 1.00x |
| (c) COPY per table per tx, conc 4 | 7.43, 7.20, 7.13 | 7.20 s | 10,211 | 4% | **3.21x** |
| (c) COPY per table per tx, conc 8 | 7.19, 6.56, 6.57 | 6.57 s | 11,195 | 10% | **3.52x** |

- Cooled, spreads are 4-10% - the earlier 1.7x outlier (C4-c 12.0 s) and R2/S1000-a (28.1 / 35.0 s) were thermal. **Treat R1-R4, the S-series and C-series as indicative only; these medians are the reference numbers.**
- (c) at conc 8 vs 4: ~9% faster on the median, ranges overlap by one run (7.19 vs 7.13). Small, consistent with the profile's ~1/3 CPU utilisation: more client threads aren't the lever.
- Every start temp 49-54 C except a-c4-1 (69 C, first in the batch) - it's also the slowest (a), so the true (a) median may be nearer 23 s.

## Where things stand (end of autonomous stretch)
- Best ingest so far: **(c), one `:put-docs`/COPY per table per tx: ~10-11k docs/s**, 3.2-3.5x the as-shipped SQL INSERT path (~3.2k docs/s cooled). Requires `:query-mode :xtql` today, plus C2 (uncommitted).
- Remaining client-side cost per the R5 profile: token-column building (`transform.clj`), transit encoding, urn-uuid resolution, Malli encode. Server side: Arrow + transit decode of the COPY payload.
- Not yet tried: decoupling the write path from `:query-mode` (put-docs writes with SQL reads); Arrow-format COPY instead of transit; cheaper urn resolution / token building.
- Out of scope: single-put on LocalLog (pending Tyler's log), search incl. top-k (not in beta3), XTQL reads (Q6 parked).

## Finding the rest of the time (James: "10k/s isn't quick")

### W1: wall-clock blocking in the R5 recording (JFR SocketRead/ThreadPark > threshold)
Loader threads (4 x ~8.6 s = ~34 s of thread time): ~12.6 s on-CPU (ExecutionSample estimate), blocked in socket reads waiting on the server for: COMMIT 7.8 s (105 events, ~75 ms each), COPY end 6.5 s (975 events, ~7 ms), SHOW LATEST_SUBMITTED_TX 1.8 s (~17 ms each), COPY start 0.4 s, BEGIN 0.25 s. FileForce (fsync) only 0.93 s total over 74 calls - not the bottleneck here.

### W2: per-thread CPU from /proc over the load window ((c) conc 4, 8.60 s)
| thread group | CPU s | cores busy |
|---|---:|---:|
| **C2 JIT compiler (x3)** | **19.04** | **2.21** |
| ForkJoinPool-1 (x9: virtual-thread carriers - pgwire connections run here) | 9.43 | 1.10 |
| DefaultDispatcher (x9: log/indexer coroutines) | 3.25 | 0.38 |
| C1 JIT | 2.42 | 0.28 |
| GC threads + G1 concurrent | 3.40 | 0.39 |
Loader threads (`pool-2-thread-*`) are missing: the bench's pool is created after the start snapshot and shut down before the end one, so they exited before being read. Wall time is the bench's own timer. **JIT compilation is ~half the measured CPU during the load: the JVM never reaches steady state in a 7-8 s load.**

### W3: N loads in one JVM ((c) conc 4; 60 s cooldown first)
`bench/run`'s load called 5 times in one JVM (`-J-Xmx10g`, `:max-resources 1000000 :query-mode :xtql :concurrency 4 :skip-search? true`), with `DROMON_XTDB_COALESCE_PUT_DOCS=1` and the patch applied for (c).

| run in JVM | load | docs/s |
|---:|---:|---:|
| 1 (cold) | 8.17 s | 9,007 |
| 2 | 5.65 s | 13,019 |
| 3 | 4.85 s | 15,184 |
| 4 | 5.08 s | 14,481 |
| 5 | 5.04 s | 14,597 |
Warm steady state ~5.0 s / **~14.6k docs/s**, ~1.6x the cold number. JIT warm-up is ~40% of a cold load. Each run is a fresh store + node + data dir; only the JVM is shared.

### W4: warm JFR profile ((c) conc 4, 5th load in one JVM: 5.18 s, 14,194/s) - `prof/W4-c-warm.jfr`
Loader on-CPU 902 samples (~9.0 thread-s of 20.7). Inclusive on loader threads: `encode-resource-doc` 42.5% (Malli storage encoder ~34.5%, surrogate check 7.5%, token columns 6.9%, key stringify 4.7%, datetime coercion ~4%); `xtdb.serde/write-transit-seq` 21%; `resolve-urn-uuid-references` 17.6% (`clojure.walk`). Server: pgwire connections ~4.4 thread-s CPU (arrow 42%, transit READ 30%, other xtdb 16%, pgwire 9%); log/indexer coroutines ~1.6 (arrow 74%, log 19%).

### W5: zero-threshold I/O events - INCONCLUSIVE for socket waits
Same setup with SocketRead/FileForce thresholds at 0 (`prof/W5-c-warm-allwaits.jfr`, 5.25 s). Loader SocketRead totals only 1.96 s over 550 events - less than W4's thresholded 8.7 s - so JFR is dropping/throttling socket events; not trusted. FileForce is consistent: 261 forces, LOG mean 7.0 ms (0.88 s), REPLICA_LOG mean 7.6 ms (1.03 s).

### W6: stage stopwatch by var wrapping (stopwatches wrapped around the stage vars at the REPL; no repo edits)
5th load in one JVM: 4.80 s wall, 15,316/s; loader thread-time 4 x 4.80 = 19.2 s.

| stage (thread-time, summed over 4 loaders) | s | % of loader time |
|---|---:|---:|
| `load-chunk!` (whole transact-transaction) | 19.04 | 99% |
| `xt/execute-tx` (**halved**: the 2-arity calls the 3-arity through the var, so the wrapper counted each call twice; raw 22.71) | ~11.35 | ~59% |
| - of which `put-docs!` (COPY: transit write + round trip + server work) | 6.45 | 34% |
| -- of which `write-transit-seq` (XT client, transit encode) | 1.97 | 10% |
| -- so COPY waiting on server ~ 6.45 - 1.97 | ~4.48 | ~23% |
| - so BEGIN + COMMIT + SHOW LATEST_SUBMITTED_TX ~ 11.35 - 6.45 | ~4.90 | ~26% |
| outside execute-tx: dromon store prep ~ 19.04 - 11.35 | ~7.69 | ~40% |
| - `encode-resource-doc` (incl surrogate check) | 4.35 | 23% |
| -- `refuse-unencodable-text!` | 0.80 | 4% |
| - `resolve-urn-uuid-references` | 2.09 | 11% |
| - other prep (entry metas, ops, coalescing, response) | ~1.25 | ~7% |

**Loader time split (warm): dromon client code ~40% | XT client (transit encode) ~10% | waiting on server for COPY ~23% | waiting on server for BEGIN/COMMIT/SHOW ~26%.** Server-side waits include fsync: 2 x ~7 ms per tx on two logs.

## James: dromon client code is out of scope - XT PoV only.

### C4 (uncommitted, store, env-gated): `DROMON_XTDB_ASYNC_SUBMIT` -> `xt/submit-tx` instead of `xt/execute-tx`
`core.clj` transact-transaction; versionIds come back nil under it (connection submit-tx returns only :tx-id). **Result: no meaningful gain - LEFT OUT (James).** Warm async loads 4.6-5.4 s vs 4.8-5.2 s with execute-tx; the end-of-load sync took 6-40 ms, so indexing had nearly caught up anyway.

### C5 (uncommitted, bench): sync + row check
`load-dataset-pooled!`: inside the timer, `SELECT 1` on a connection taken from the store's pool (James: a connection gets read-your-writes, a node query does not - first version queried the node and was wrong). Outside the timer: observation/patient counts and `xt.txs` committed/failed.

## Grouping detail: docs per statement and per transaction (pool runs only)
73,566 docs; conc 4 and cold JVM unless marked. "Statement" = one INSERT or one COPY. Docs/COPY and statements/tx computed from the dataset (grouped COPY = one per resource type per tx). ‡ = hot-laptop run (up to 1.7x inflated). Single runs are indicative; medians (n=3, 60 s cooldown) are the reference numbers.

| run | write path | docs/statement (mean / p50 / max) | statements/tx | docs/tx (p50 / max) | txs | load | docs/s |
|---|---|---|---:|---|---:|---:|---:|
| S1-a | INSERT | 1 | 1 | 1 | 73,566 | **stopped at 12.5 min** (fsync-bound) | <100 |
| S100-a | INSERT | 1 | ~93 | 100 / 100 | 793 | 25.8 s | 2,846 |
| S1000-a | INSERT | 1 | ~525 | 491 / 1,000 | 140 | 35.0 s ‡ | 2,102 |
| R2 | INSERT | 1 | ~584 | 491 / 2,000 | 126 | 28.1 s ‡ | 2,618 |
| **a, median of 3** | INSERT | 1 | ~584 | 491 / 2,000 | 126 | **23.2 s** | **3,178** |
| S100-b | COPY per doc | 1 | ~93 | 100 / 100 | 793 | 24.5 s | 3,007 |
| S1000-b | COPY per doc | 1 | ~525 | 491 / 1,000 | 140 | 21.0 s | 3,508 |
| R3 | COPY per doc | 1 | ~584 | 491 / 2,000 | 126 | 21.2 s | 3,476 |
| S100-c | COPY per table per tx | 10.2 / 5 / 100 | 9.1 | 100 / 100 | 793 | 12.8 s | 5,761 |
| S1000-c | COPY per table per tx | 40.3 / 12 / 660 | 13.1 | 491 / 1,000 | 140 | 7.2 s | 10,149 |
| R4 | COPY per table per tx | 43.3 / 13 / 1,309 | 13.5 | 491 / 2,000 | 126 | 7.2 s | 10,194 |
| **c, median of 3** | COPY per table per tx | 43.3 / 13 / 1,309 | 13.5 | 491 / 2,000 | 126 | **7.2 s** | **10,211** |
| c, conc 8, median of 3 | COPY per table per tx | 43.3 / 13 / 1,309 | 13.5 | 491 / 2,000 | 126 | 6.6 s | 11,195 |
| c warm (5th load in one JVM) | COPY per table per tx | 43.3 / 13 / 1,309 | 13.5 | 491 / 2,000 | 126 | ~5.0 s | ~14,600 |

- Grouped COPYs are still small (p50 13 docs): a Synthea patient bundle interleaves ~19 resource types, so a 491-doc tx splits into ~13 COPYs; the mean (43) is carried by a few big types (Observation up to 1,309).
- Tx size is bundle-bounded: p50 491 docs, so the 1000 and 2000 caps barely differ. Larger txs would mean merging bundles, which changes what the bench models.
- Bigger txs help only when statements get bigger with them: at 1 doc per statement, 100 -> ~584 docs/tx changes nothing; grouped COPY goes 10 -> 43 docs/COPY and 5.8k -> 10.2k/s.

## Reconciling with Tyler's posted table (2026-10-17 evening)
His table: embedded / server / Datomic columns; load "100k resources" 11.6 s embedded; writes incl. "transaction, 1,000 entries: 286 ms, 3,379 entries/s", "single create: 3.25 ms, 288 ops/s", "batch, 100 entries: 328 ms, 307 entries/s". Source harness for the write rows not found in dromon (Q5). `docs/xtdb-perf-findings.md` (Tyler, 2026-06-02, beta1) is Inferno request latency, not this table.

### T1: (a) INSERT, single thread, max-tx 1000, warm runs in one JVM
| run in JVM | load | docs/s |
|---:|---:|---:|
| 1 | 62.56 s | 1,176 |
| 2 | 67.13 s | 1,096 |
| 3 | 65.85 s | 1,117 |
140 txs, mean ~525 docs/tx -> ~465 ms per tx -> ~0.89 s per 1,000 docs, vs Tyler's 286 ms: **~3.1x slower**. No warm-up gain on this path (unlike (c)). Conc 4 gets ~2.9x this (3.2k/s cold median).

### Side by side (ours scaled where noted)
| measure | Tyler (embedded) | ours | ratio ours/his |
|---|---:|---:|---:|
| load, 100k resources | 11.6 s (8.6k/s) | 23.2 s for 73.6k (3.2k/s) -> ~31.5 s per 100k | ~2.7x |
| transaction of 1,000 entries | 286 ms | ~890 ms (T1, single thread) | ~3.1x |
| body weight, all hits | 188 ms | 395 ms (1,301 hits, R2 single run) | ~2.1x (2.9x if scaled by 100/73.6) |
| category=laboratory, all hits | 2,724 ms | 3,077 ms (14,054 hits) | ~1.1x (1.5x scaled) |
| 10-code OR, all hits | 1,188 ms | 2,143 ms (9,916 hits) | ~1.8x (2.5x scaled) |
| unsorted top 50 | 9-20 ms | 18-32 ms | ~1.6-2x |
| body weight, top 50 by -date | 45 ms | 96 ms | ~2.1x |
| category=laboratory, top 50 by -date | 59 ms | 113 ms | ~1.9x |
| single create (1-doc tx) | 3.25 ms | floor ~14 ms: 2 fsyncs x ~7 ms (W5) on the local log | >=4x |

- Everything except single-doc writes sits in a 1.5-3x band -> consistent with a uniform machine factor (ours: i7-8565U 15 W laptop, 2018, thermally limited) plus unknowns (his dromon/XTDB version and date, dataset size/population, JIT warmth, concurrency).
- Single-doc writes don't fit the band: his 3.25 ms per 1-doc tx is below our two-fsync floor. assumption: his log's fsync is near-free (macOS `fsync` does not flush the drive cache unless F_FULLFSYNC; or an in-memory log; or a different log entirely). This is the same question as "which log is Tyler using".
- Search ours is single runs on a hot laptop (R2) - lower confidence than the load numbers.

## The store's Hikari pool re-introduces pgwire (XT PoV finding)
- James: `xt/q` against an in-process node no longer goes over pgwire (changed to in-process/ADBC in a recent beta). Confirmed in beta3 `xtdb/api.clj` (jar): the node's in-process `Xtdb.Connection` impl lives in `xtdb.node.impl`; `java.sql.Connection` / `DataSource` / `Object` impls go through JDBC.
- dromon's store hands `xt/q`/`xt/execute-tx` a `java.sql.Connection` from a per-tenant Hikari pool over the node (`docs/xtdb-perf-findings.md`, "the one big win" on beta1). Our loader-thread dumps show `org.postgresql` socket reads (S1-a dumps; W1 SocketRead stacks): **every store call is still pgwire**, and the pool is now what puts it there.
- Several store workarounds documented in CLAUDE.md exist only because of pgwire (`prepareThreshold=0`, the 0A000 / 08P01 eviction carve-outs, the stale-plan re-run, surrogate pairs mangled by pgjdbc). Candidate experiment: pass the node itself (in-process connectable) for writes and compare ingest.

## ADBC connection vs pooled JDBC (2026-10-17, late)

### C6 (uncommitted, store, env-gated): `DROMON_XTDB_ADBC_CONN`
`transact-transaction` takes `(.connect node)` - an in-process `Xtdb$Connection` - instead of `jdbc/get-connection pool`, and runs the whole tx (incl. `xt/execute-tx`) on it. Reads still go through the pool. (A first cut passed the node itself to `execute-tx`, which opens an ADBC connection per call; James: hold a connection, an ADBC one rather than JDBC.) Run in a separate worktree with the patch applied.

### Cold, n=3 interleaved, 60 s cooldown before each
| config | runs (s) | median | docs/s |
|---|---|---:|---:|
| pool (a) INSERT per doc | 23.49 / 23.37 / 24.28 | 23.49 s | 3,132 |
| ADBC (a) INSERT per doc | 12.80 / 14.28 / 12.50 | 12.80 s | 5,745 |
| ADBC (b) put-docs per doc | 8.02 / 8.76 ‡ / 8.85 | 8.76 s | 8,395 |
| pool (c) put-docs per table per tx | 7.17 / 7.62 / 7.86 | 7.62 s | 9,655 |
| ADBC (c) put-docs per table per tx | 5.35 / 5.07 / 6.12 | 5.35 s | 13,747 |

‡ started at 90 C; all others 53-65 C. All 29,208 Observations landed in every run; no batch fallbacks.

- ADBC beats the pool on every path: (a) ~1.8x, (c) ~1.4x. (a) gains most: no pgwire round trip per `INSERT`.
- Grouping still pays on ADBC but less: (b) -> (c) ~1.6x, vs ~2.8x on the pool where each COPY is its own round trip.

### Warm (5 loads in one JVM, median of loads 3-5)
| config | loads 1-5 (s) | median 3-5 | docs/s |
|---|---|---:|---:|
| ADBC (c) | 5.21 / 3.28 / 3.32 / 3.15 / 3.01 | 3.15 s | 23,332 |
| pool (c) (W3, earlier) | 8.17 / 5.65 / 4.85 / 5.08 / 5.04 | 5.04 s | ~14,600 |

- Warm ADBC (c) is ~1.6x warm pool (c), and reaches steady state by load 2 (the pool was still ~10% off at load 2).

## XT-side ideas noted along the way (later)
- Per-table recency in DDL: a table declaring its default valid-time view, so atemporal reads of event-like tables (valid-time history) don't need `FOR VALID_TIME ALL` per query.
- Clustering: page metadata (numeric min/max, string blooms; struct children and list elements included, `ColumnMetadata.kt`) only prunes when a page's rows cluster on the column. Pages are cut by IID - raw bytes for UUID `_id`s, SHA-256 for strings (`Iid.kt`) - and dromon's ids are strings. Per-table PKs (likely XT 2.3) would give this clustering; parked.
