# Validation plan for native-layer ideas in Lance

This plan turns the candidates in [ideas_analysis.md](ideas_analysis.md) into
measurable experiments. The comparison used native-layer commit `6d50a6537`
and Lance commit `5ba577117`. No performance gain has been measured yet. The
numbers below are proposed decision thresholds, not predictions.

## Goals and order

| Priority | Candidate | Question to answer | First stop condition |
| --- | --- | --- | --- |
| 1 | Staged reads for `AND` filters | Can reading a selective, cheap predicate first avoid enough later column I/O to offset extra read requests? | No material filter-column bytes can be skipped in representative queries. |
| 2 | Move substantial batch decode to `spawn_cpu` | Does synchronous decode on Tokio workers delay I/O and other scan tasks under load? | Decode tasks are short or scheduler delays do not correlate with decode. |
| 3 | Shared byte admission for writes | Do concurrent encoding tasks account for a meaningful part of peak write memory? | Peak memory comes mainly from input arrays, caches, or other state that the proposed admission cannot control. |
| Conditional | Tail-read and page/file-size tuning | Are metadata round trips or current sizing defaults material bottlenecks? | Baseline profiles do not show those costs. |

Run the three main experiments separately. Keep the Lance file format and
on-disk bytes compatible with released stable formats. Follow
[Lance's contribution rules](../../lance/AGENTS.md) and
[Rust concurrency rules](../../lance/rust/AGENTS.md) when building prototypes.

## Shared measurement protocol

1. Record the Lance commit, Rust version, CPU model/core count, RAM, storage
   type, object-store endpoint/region if used, runtime thread settings, Cargo
   profile, and every benchmark parameter. Keep a baseline worktree and one
   candidate worktree from the same starting commit. Rebuild both with
   `release-with-debug` and run them on the same machine.
2. Generate deterministic Arrow inputs with a recorded seed. For read tests,
   write each fixture once and read the **same files** in both builds. For
   write tests, use the same schema, row values, batch boundaries, and input
   stream in both builds, but write to separate fresh output paths. Do not
   include fixture creation, validation, or cleanup in read timings.
3. Warm up code and storage separately. Label runs as process-cache warm,
   storage-cache warm, or controlled cold. Reopen the dataset as required by
   the scenario. Do not call a run cold unless the relevant Lance/session,
   object-store, and OS caches were reset or bypassed. Use a new process or
   isolated paths when cache reset cannot be verified.
4. Use the same projection, output consumption, filter, statistics/index
   settings, materialization style, batch size, read-ahead, and scan order for
   each A/B pair. Drain batches and count rows or compute a small checksum;
   avoid collecting all batches. Benchmark whole operations, including their
   scheduling and sink work, and profile the important substeps separately.
5. Warm up first, then collect at least ten measured A/B pairs per case in
   alternating order (AB, BA, ...). Report median, p95 where relevant, the
   paired percentage change, and a 95% bootstrap interval over paired changes.
   Repeat borderline or noisy cases on another run/day. Treat a change whose
   interval crosses zero as inconclusive.
6. For reads, report elapsed time, rows/s, rows returned, bytes read, I/O
   requests, CPU time, peak RSS, and p95 query latency under concurrent load.
   For writes, report elapsed time, input MiB/s and rows/s, output bytes,
   peak RSS, encoding queue depth, admission wait time, and sink idle time.
   Capture error counts and output checksums in both directions.
7. Compare a representative workload set, not just the best synthetic case.
   Proposed acceptance gate for read changes: at least 10% lower median
   latency in two target cases with a 95% paired interval excluding zero, no
   more than 5% regression in common control cases, and identical results.
   For write admission: at least 25% lower peak RSS in the memory-stressed
   case, no more than 5% throughput loss at normal load, and identical
   readback. These gates should be agreed before examining results.

### Bench harness and observability

- Extend or add a parameterized scan benchmark near
  [Lance's scan bench](../../lance/rust/lance/benches/scan.rs). The current bench
  creates `./test.lance`, runs a full scan, and collects every batch. It does
  not exercise selective filters and must not be used as evidence for this
  candidate without changes. Use temporary/explicit fixture paths and a
  streaming checksum in the new harness.
- Use existing
  [filtered-read metrics](../../lance/rust/lance/src/io/exec/filtered_read.rs)
  (`fragments_scanned`, `ranges_scanned`, `rows_scanned`, task wait time) and
  [I/O metrics](../../lance/rust/lance/src/io/exec/utils.rs) (requests, bytes,
  IOPS) where they cover the operation. The
  [I/O scheduler](../../lance/rust/lance-io/src/scheduler.rs) exposes pending
  and active I/O plus reserved bytes in tracing. Add narrow timing/counters
  where existing metrics do not distinguish the proposed stages.
- Use the existing
  [decoder](../../lance/rust/lance-encoding/benches/decoder.rs),
  [encoder](../../lance/rust/lance-encoding/benches/encoder.rs), and
  [scheduler](../../lance/rust/lance-io/benches/scheduler.rs) microbenchmarks
  to understand local costs. They cannot replace end-to-end scan/write tests.
- Run benchmark binaries from `../lance/rust`, following the repo's
  [profile guidance](../../lance/AGENTS.md):

  ```bash
  cargo bench --profile release-with-debug -p lance --bench scan
  cargo bench --profile release-with-debug -p lance-encoding --bench decoder
  cargo bench --profile release-with-debug -p lance-encoding --bench encoder
  cargo bench --profile release-with-debug -p lance-io --bench scheduler
  ```

  The scan command becomes relevant to selective filters only after the new
  cases are added. Run one benchmark process at a time on an otherwise idle
  machine. Retain profiler output and raw per-run results.

## 1. Staged reads for selective conjunctions

**Hypothesis.** Lance's
[filtered read](../../lance/rust/lance/src/io/exec/filtered_read.rs) reads all
filter columns in its initial projection. For a filter such as `cheap = 1 AND
large_text = 'x'`, evaluating `cheap = 1` first could skip most reads and
decode of `large_text`. Extra row selections and small object-store requests
may erase this gain.

1. Build deterministic fixtures with at least two predicate columns and one
   projected payload column. Include narrow integer, medium string, and wide
   binary/string filter columns; compressible and less-compressible values;
   multiple fragments and page boundaries; and selective rows scattered
   across pages as well as clustered together. Record compressed bytes per
   column and how many pages contain surviving rows.
2. Start with simple deterministic `AND` predicates over different columns.
   Vary first-predicate pass rates: about 0.1%, 1%, 10%, 50%, and 100%.
   Include both orders of cheap/expensive predicates. Use a single predicate
   and an unselective conjunction as negative controls. Test a narrow and a
   wide projection. Keep scalar index use disabled to isolate staged reads;
   run statistics disabled first, then repeat key cases with statistics
   enabled. Keep the same `MaterializationStyle` in each A/B pair via
   [scanner settings](../../lance/rust/lance/src/dataset/scanner.rs).
3. Measure baseline elapsed time, per-column bytes requested, requests,
   decoded rows, filter evaluation time, CPU, and peak memory on local SSD.
   Repeat with realistic object-store latency, if an object store is
   available. If not, use a documented latency emulator and mark the cloud
   result as unverified. Calculate an upper bound from bytes in later filter
   columns that could be skipped at page/range granularity.
4. Prototype a **fixed two-stage order** first. Read the first predicate
   columns, evaluate them, convert surviving physical rows into the next
   stage's selection, read the second predicate columns for those rows, then
   materialize payload using the final selection. Preserve the original
   output order and row IDs. Use the current-format path only. Do not infer a
   benefit from merely reordering expression evaluation: it must reduce
   actual read/decode work.
5. If fixed staging wins, compare ordering heuristics. Start with metadata
   compressed bytes for the next column group; then test estimated bytes per
   rejected row when selectivity estimates are reliable. Charge each stage
   for the additional request/selection overhead. Fall back to the existing
   plan when estimates are missing or a predicate is unsupported. Verify that
   the ordering calculation itself does not dominate small scans.
6. Check exact result equivalence against the current scanner: `NULL` and
   three-valued Boolean behavior; all-pass and none-pass masks; filters with
   overlapping columns; deleted rows; multiple fragments; stable row IDs;
   page/batch edges; empty input; and row ordering. Cover `OR`, nested
   expressions, unsupported functions, and partial scalar-index matches by
   keeping the baseline path unless equivalence is proved. Compare returned
   rows and values, not only row counts.

**Decision.** Continue only if saved physical bytes and decode time exceed
added request and selection costs in representative selective cases. Reject
or narrow the optimization if local gains reverse under object-store latency
or unselective controls regress beyond the shared gate. Publish the
selectivity/column-width boundary where staging helps.

## 2. CPU pool placement for batch decode

**Hypothesis.** Some scan paths dispatch
[`NextDecodeTask::into_batch`](../../lance/rust/lance-encoding/src/decoder.rs)
through `tokio::spawn`. If a substantial synchronous decode occupies Tokio
workers, I/O completions and other scans may wait. Lance already has
[`spawn_cpu`](../../lance/rust/lance-core/src/utils/tokio.rs); the question is
which decode tasks, if any, should use it.

1. Instrument queue-to-start, `into_batch` CPU duration, completion-to-next
   I/O scheduling, and Tokio worker utilization. Relate these to scheduler
   pending I/O, read bytes, throughput, and p95 query latency. Test small and
   large batches, narrow and wide projections, and both `lz4` and `zstd`.
   First try the existing concurrency/read-ahead settings to see whether
   configuration alone removes the delay.
2. Run an end-to-end workload with one heavy compressed scan plus several
   concurrent short selective scans. Measure both aggregate rows/s and p95
   latency of the short scans. Run on small and large CPU allocations, with
   at least one single-compute-worker configuration. Keep the I/O fixture
   and cache state fixed across builds.
3. Prototype dispatching only large, pure CPU `into_batch` work to
   `spawn_cpu`; leave tiny tasks inline. Measure actual task duration to
   choose the cutoff. Lance's Rust guidance suggests about 100 microseconds
   as the minimum useful CPU task size. Do not put I/O, locks, channel waits,
   or `block_on` inside the CPU closure. Do not nest `spawn_cpu` calls.
4. Re-run the decoder microbenchmark to measure dispatch overhead, then the
   concurrent scan workload to see whether it changes real latency. Add
   correctness and lifecycle tests for decode errors, task cancellation,
   dropped consumers, small-batch fallback, and a one-worker CPU pool. Check
   that memory does not grow through queued decoded batches.

**Decision.** Keep the change only if queue traces show a real Tokio delay,
the targeted p95 scan latency improves under load, and small-scan throughput
does not cross the regression gate. If decode is short or I/O remains the
bottleneck, retain the current dispatch path.

## 3. Shared byte admission for concurrent writes

**Hypothesis.** Lance bounds some per-writer buffers through
[`FileWriterOptions`](../../lance/rust/lance-file/src/writer.rs) and caps page
size on a best-effort basis. It can also spill page metadata. In-flight
encoded pages across many writers may still raise peak RSS. A shared byte
budget might bound that memory, but waiting can leave the sink idle.

1. Build a write harness that streams bounded, repeatable Arrow batches. Use
   fresh output paths per run. Test 1, 4, and 16 concurrent writers; narrow
   and wide schemas (for example 4 and 64 columns); small and large batches;
   compressible and less-compressible values; and `lz4` and `zstd` where
   supported. Keep the format version and
   [write options](../../lance/rust/lance/src/dataset/write.rs) fixed. Measure
   local SSD first and object-store writes if available. Include a normal
   workload and a memory-stressed workload.
2. In a separate process per run, sample RSS and, where available, allocator
   live bytes. Add counters for input arrays held, data cache bytes,
   encoding tasks/pages in flight, encoded bytes waiting for ordered sink
   writes, page metadata spill, sink busy/idle time, and time spent waiting
   for admission. Separate these categories before deciding what to bound.
   Output size alone is not a proxy for resident memory.
3. Sweep existing `data_cache_bytes`, `max_page_bytes`,
   `max_rows_per_file`, and `max_bytes_per_file` settings first. A good
   configuration may meet the memory target without new concurrency code.
   Record page count, file count, compression ratio, and downstream read
   throughput so a smaller write buffer does not quietly damage reads.
4. If encoded work dominates RSS, prototype one shared byte budget at the
   async boundary before starting CPU encoding. Reserve a conservative size
   estimate; reconcile it with actual encoded bytes; release the reservation
   when the sink has written or dropped the page. Document how a page larger
   than the budget makes progress. Waiting for capacity must occur in async
   code, never inside `spawn_cpu`. Account for cancellation, errors, and
   writers closing while work is pending. Avoid a global lock held across
   `.await`.
5. Sweep budgets around the measured normal in-flight footprint rather than
   choosing a fixed number in advance. For each cap, report peak RSS,
   resident encoded bytes, input MiB/s, p95 batch-write latency, sink idle
   fraction, and queue wait time. Verify readback row-for-row, schema,
   deletions if used, output file count/size, and cleanup after cancellation
   or failed writes. Test a single CPU worker for deadlock risk.

**Decision.** Keep byte admission only if it materially reduces peak RSS in
the stressed case without starving the sink or exceeding the normal-load
throughput regression gate. State clearly whether the limit applies to
encoded work only or to all writer memory; RSS is not a hard cap.

## Conditional tuning experiments

- **Metadata tail read:** If traces show multiple range requests to open each
  file, sweep the tail-read size in
  [file reader](../../lance/rust/lance-file/src/reader.rs). Separate cold and
  warm opens, and compare small versus large metadata. Record bytes fetched
  beyond the footer and open latency. Stop if one-request opens already
  dominate or extra bytes outweigh saved requests.
- **Page/file sizing:** If write profiles show too many small pages or files,
  sweep existing `max_page_bytes`, `data_cache_bytes`, and
  `max_bytes_per_file`. Measure write throughput, peak RSS, file count,
  compression ratio, and subsequent scan latency together. Keep this as
  configuration tuning unless a repeatable limitation remains.

## Deliverables and implementation gates

For each candidate, save the fixture generator and seed, benchmark code,
exact commands/configuration, raw A/B runs, flamegraphs/traces, correctness
test results, and a one-page decision. Use this result table:

| Candidate and case | Baseline median / p95 | Variant median / p95 | Paired change and 95% interval | Bytes/requests or RSS | Correctness | Decision |
| --- | --- | --- | --- | --- | --- | --- |
| Example: staged filter, 1% pass, wide second column, local SSD | Fill after run | Fill after run | Fill after run | Fill after run | Pass/fail | Keep, narrow, reject, or inconclusive |

For any prototype, run focused tests in the touched crates, then the
repository's Rust checks from `../lance/rust` before proposing a change:

```bash
cargo test -p lance <focused_test_name>
cargo test -p lance-encoding <focused_test_name>
cargo check --workspace --tests --benches
cargo fmt --all -- --check
cargo clippy --all --tests --benches -- -D warnings
```

Run only the relevant focused test commands for the touched crates. Add
tests for every behavior change as required by Lance. Compare against the
current path and relevant stable-format fixtures when compatibility is in
scope. Report any checks not run and why.
