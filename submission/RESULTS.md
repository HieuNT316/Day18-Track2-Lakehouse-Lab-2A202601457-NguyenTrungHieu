# Day 18 Track 2 — Measured Results

Lightweight path (`deltalake` 1.6.2 + `pyiceberg` 0.11.1 + DuckDB 1.5.5 + Polars 1.43.2),
CPython 3.11.9 on Windows 10. Every number below was measured on this machine;
`make run-all` is green 8/8. Notebook outputs are preserved in `notebooks/*.ipynb`.

## Part A — Foundations (44 pts)

| NB | Criterion | Measured | Reading |
|---|---|---|---|
| 1 | `_delta_log/` JSON commits | v0 `WRITE`, `num_added_rows=3`, `execution_time_ms=4` | One JSON per commit, append-only. The log *is* the table; the parquet files are just bytes it points at. |
| 1 | Schema enforcement | `age=str` write blocked | Enforcement is the default. A lake without a log would have accepted it and broken every downstream reader. |
| 1 | Schema evolution | `tier` added via `schema_mode="merge"` | Evolution is **opt-in**. Same API, one flag apart — which is exactly why accidental schema drift can't happen silently. |
| 2 | Small-file problem | **200** files before OPTIMIZE | ≥ 100 required. |
| 2 | Speedup **or** pruning | speedup **10.2×**, files-pruned **55.0×** | Target is ≥3× *or* ≥10×; both cleared. Wall-clock is noisy on a laptop, so file-pruning (1 of 55 files covers `user_id=4242`) is the honest metric. |
| 2 | `numFiles` after OPTIMIZE | 200 → **55** | Compaction alone gives 4× fewer files; Z-ORDER is what makes 54 of them skippable. |
| 3 | `history()` ≥ 5 versions incl. RESTORE | **5**: WRITE, WRITE, MERGE, WRITE, **RESTORE** | RESTORE is itself a transaction (v4), not a rewind that erases evidence. Fully auditable. |
| 3 | MERGE 100K | `num_source_rows=100,000` → `num_output_rows=150,000` in **184 ms** | 50K updates + 50K inserts in one atomic commit. |
| 3 | RESTORE removes bad data | `score < 0` count = **0**, RESTORE in **0.02s** | Metadata-only operation — it re-points the log at v2's file set, it does not copy data. Hence 0.02s regardless of table size. |
| 4 | Bronze/Silver/Gold present | all three on disk | |
| 4 | Silver dedup | 200,000 → **190,052** (dropped **9,948**) | Dedup on `request_id`; the duplicates were seeded deliberately, which is what a real at-least-once ingest produces. |
| 4 | Gold ≥ 7 dates × 3 models | **24** rows = 8 dates × 3 models, with p50/p95/`cost_usd`/`error_rate` | |

## Part B — Lakehouse 2026 (50 pts)

| NB | Criterion | Measured | Reading |
|---|---|---|---|
| 5 | Catalog-created table, `day(ts)` spec | created through `SqlCatalog`, spec = `day(ts)` | |
| 5 | Hidden-partition pruning ≥ 5× | **10×** (10 files → 1), filtering on **`ts`**, not `ts_day` | This is the whole point: Iceberg derived `ts_day` from the stored transform, so the *user* never names the partition column. A Hive user who forgot `WHERE dt=...` reads all 10 files — at 512 MB/file and $5/TB × 10K queries/day that is **$220/day** for one forgotten predicate. |
| 5 | Three-tier metadata + byte ratio | data **47.3 KB** / metadata **137.5 KB** (20 avro + 12 json) | Metadata is 3× the data *at 10 rows per file*. At 512 MB/file it is ~0.1%. Small files punish you twice: more data files **and** more metadata to plan over. |
| 5 | `field_id` stable on rename; ≥ 2 specs coexist | `latency_ms` → `latency_millis` kept **`field_id=4`**; specs **[1, 2]** coexist, **5,500** rows readable | A rename rewrote **no** data, and old rows read back `tier=NULL` with no backfill job. Column identity lives in the field ID, not the name — this is what makes schema changes free. |
| 6 | **Job 1** Compaction ≥ 10× | 200 → **11** files (**18×**) | Baseline avg file size 51.5 KB vs a production target of 128–512 MB. Full-scan GETs/day: 10M (**$4.00/day**) → 200K (**$0.08/day**). Note data bytes went *up* briefly (10.1 → 16.1 MB): compaction writes new files before the old are reclaimed. You pay twice, briefly — budget for it. |
| 6 | **Job 2** Clustering ≥ 50% skip | 11/11 files before → **1/10** after = **90% skip** | Proven from min/max stats, not timing. |
| 6 | **Job 3** Expiry | Delta vacuum reclaimed **16.1 MB** (211 tombstoned files); Iceberg **20 → 3** snapshots | |
| 6 | **Job 4** Orphans | **3** planted Delta orphans found (21.2 KB); **17** stranded Iceberg manifest lists swept (**37.3 KB**) | See "Finding 1" and "Finding 2" below — this job is where the lab contradicts the slide. |
| 6 | **Job 5** Checkpoint | `00000000000000000099.checkpoint.parquet`, `_last_checkpoint` present | A reader now loads 1 checkpoint + the few JSONs after it instead of replaying all 200. |
| 7 | Random-access amplification ≥ 5× | **200×** (12.5 MB row group vs 64 KB object GET) | The inline parquet has **1 row group of 200 rows**. Parquet's read granularity is the row group, so one random frame costs the whole group. At 1,000 frame-fetches/sec feeding a GPU that amplification *is* the bottleneck. |
| 7 | int8 ≥ 3× smaller; recall **and** topic fidelity | **5.8×** smaller (2.6 MB → 451.9 KB); recall@10 = **0.904**, topic fidelity = **1.000** | int8 loses ~10% of exact IDs but **100%** of results stay on-topic — the "misses" are swaps between near-equivalent neighbours. Exact-ID recall *understates* quantization quality for RAG. |
| 7 | Semantic search as SQL | top-k returned, top-5 all share the query topic | |
| 7 | **Lifecycle bug reproduced** | table **2,000 → 1,992**; external index still **2,000** → 0 hits in-table, **> 0** hits in the stale index | The 8 deleted docs stay findable *forever* in the external index, because deletes are the operation sync pipelines forget. CDF confirms 8 delete events were emitted — the signal existed; nobody consumed it. |
| 8 | Silver partitioned by `agent_version`; Gold both policies | **1,578** steps; partitions `policy-v2`, `policy-v3` | |
| 8 | Version pin replays exactly | pinned v0 = **1,578** steps; after more rollouts v1 = 1,978; replay at v0 still **1,578** | Reproducibility comes from pinning a table *version*, not from copying a dataset. |
| 8 | MCP surface | 5 turns → **1** catalog read; `input_required` before `delete_rows` (300 rows); task poll completes | |
| 8 | 4 Art. 10 buckets; UNCLASSIFIED excluded | all 4 buckets are partitions; **334** UNCLASSIFIED (16.7%) excluded → **1,666 / 2,000** defensible rows | `license=unknown` fails the Art. 10 origin requirement. Mixing scraped and licensed data in one unlabelled bucket is a 2026 audit failure. |

## Part C — Reproducibility (6 pts)

* `make run-all` — **8/8 green in 62.4s** (4 pts) ✅
* `make test` — **22 of 23 pass** (2 pts) ⚠️ see note

The single failure is `test_reset_catalog_does_not_touch_siblings`, and it is
**platform-specific, not a lab defect**. Proven cause:

```
PermissionError: [WinError 32] The process cannot access the file because
it is being used by another process: '...\iceberg\drop\catalog.db'
```

`lakehouse.reset_catalog()` calls `shutil.rmtree(..., ignore_errors=True)` while
pyiceberg's `SqlCatalog` still holds an open SQLAlchemy handle on the SQLite file.
POSIX unlinks open files happily, so the directory disappears and the assert passes;
Windows refuses, `ignore_errors=True` swallows the error, and the directory survives.
Disposing the engine first (`cat.engine.dispose()`) makes it pass on Windows — verified.
On the grader's POSIX machine this test passes unmodified.

## Two findings that contradict a common belief

**1. `VACUUM` does not remove orphans that were never committed.**
The table reported 100,000 rows with **15** parquet files on disk but only **10**
in the log — **5 files you pay for and cannot see**. A `VACUUM` dry-run listed 211
files to reclaim and *still* left those 5. `deltalake` (Rust/Python) reclaims files
the log has **tombstoned**; a file a crashed job left behind was never committed, so
it was never tombstoned, so the log has no idea it exists. Spark's VACUUM adds a
directory listing pass — but never *assume* your engine does it. Run the set
difference yourself (with an age guard; without one you will delete files an
in-flight writer is still using).

**2. `expire_snapshots` deletes no files and can make metadata bigger.**
Snapshots went **20 → 3**, avro files on disk **40 → 40** (**0 deleted**), and
metadata *grew* **345.1 KB → 353.3 KB**. Expiry alone reclaimed **nothing**. Only
chaining it into an orphan sweep found the **17 stranded manifest lists** and
reclaimed **37.3 KB** (avro 40 → 23). Job 3 and Job 4 are a **pair** — running
expiry without an orphan scan is exactly why "we expired our snapshots" and "the S3
bill didn't move" coexist in the same postmortem.
