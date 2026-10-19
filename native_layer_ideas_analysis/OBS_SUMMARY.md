# Native-layer ideas: object-store results and conclusions

Analysis date: 2026-10-19. Run: `20261016091115120-835675`, started on
2026-10-16. Measurements come from the [provided results](../20261016091115120-835675/README.md).
The scope follows the three prioritized experiments in the original
[idea review](../../q2/ideas_for_lance/ideas_analysis.md) and
[validation plan](../../q2/ideas_for_lance/validation_plan.md).

The object-store run supports selective further work, but it does not justify
enabling any of the three ideas as a general optimization:

- **Staged reads:** retain the idea for clustered survivors and empty first-stage
  results. The main scattered 1% case is **19.4% slower**, despite reading 38.1%
  fewer bytes. The prototype also returns rows out of order in 41 of 130 measured
  staged runs. It needs a correctness fix and a storage-aware cost model.
- **Writer memory:** smaller caches consistently reduce memory, but 8 MiB costs
  **46.7% more write time** with 16 wide writers and **39.6% more time in later
  full scans**. A 32 MiB cache is a useful workload-specific candidate, with
  roughly 47% lower peak RSS, but its performance cost is not established closely
  enough for a default change. Shared byte admission remains unproven.
- **CPU-pool decode:** the large local short-query latency improvement does not
  appear here. Every object-store latency interval includes zero. Keep the local
  evidence as a reason for a targeted scheduling change, but do not claim an
  object-store speedup from this run.

**The missing traces do not block these conclusions.** They limit the explanation
of decoder task placement and waiting time. A separate source-provenance gap
limits claims about the exact Lance version tested.

## Evidence and measurement limits

I checked all 12 CSV files and independently reproduced all 84 table rows in the
generated README, including medians, p95 values, paired changes, and intervals.
The paired files have complete pairs, alternating AB/BA order, and matching
recorded row counts and checksums where those fields exist. The duplicate results
directory under `native-layer-v10-check/` is identical to the supplied root
directory. Raw files were left unchanged.

The client was an Intel Core i7-13700 with 24 logical CPUs and about 31 GiB RAM.
Main cases use ten pairs after warm-up; decoder isolated scans use 20, the
short-only control uses 40, and memory probes use three. Every measured process
has a fresh Lance session. Object-store caches and external load were not
controlled. The endpoint and region are redacted, so these are results for this
S3-compatible storage setup, with no verified provider, network distance, or
server configuration.

Tables below use **median paired percentage change**:
`100 × (variant − baseline) / baseline`, followed by a 95% bootstrap interval.
Negative changes favor the variant. This statistic can differ from the change
between the two displayed medians. Intervals use the runner's 10,000 resamples;
they are per-case intervals, without adjustment for multiple comparisons. An
interval containing zero establishes neither a speedup nor performance
equivalence. Ten samples per variant also give a limited estimate of tail latency.

Read fixtures contain 65,536 rows in four fragments. Write fixtures contain eight
1,024-row batches per writer; the 64-column cases have 64 string columns plus
`id`. Input arrays are shared across concurrent writers. The supplied harness
times scans after opening the dataset and times writes after constructing input.
Write listing and readback happen after the timer and RSS sample. Peak RSS is
the process high-water mark, including input allocations, rather than a live
writer-buffer measurement. Read bytes and requests come from scanner summaries,
not transport-level HTTP logs; they do not establish request retry counts or
billable operations.

This run does not separately test metadata-tail sizing, file-size targets, or
new pruning/caching mechanisms. The original review identified several native-layer
features as already present in Lance; these measurements provide no reason to
duplicate them.

### Source provenance needs reconciliation

The [run configuration](../20261016091115120-835675/run-config.json) records
`94ae984d05f9f8a524ccfd0cae2a1855c221b0f7`. That commit's workspace version is
`14.0.0-beta.5`. The supplied [build helper](object-store/build-binaries.sh)
requires `6e59a80ed6cbe95e173ada5adf8ffa2f201efb22`, whose workspace version is
`10.0.1-beta.0`. These are different source histories, not just different commit
labels for the same version.

Two recorded artifact hashes also differ from the files now supplied:

| Artifact | Recorded SHA-256 prefix | Supplied SHA-256 prefix | Match |
| --- | --- | --- | --- |
| Harness | `63d46a32cec0` | `b54be03f200c` | No |
| Read allowlist patch | `27eb68bc6291` | `efc4a930a3f8` | No |
| Writer options patch | `72d4f7493f88` | `72d4f7493f88` | Yes |
| Decoder patch | `ac6787717ed5` | `ac6787717ed5` | Yes |

The packaged source archive contains the same harness and read patch as the
current supplied files, so it does not resolve the mismatches. The runner records
the launching checkout's HEAD, not the binary's embedded build revision. A checkout
change after building is one possible explanation, but it is unverified. It also
does not record the read/write binary selected by `LANCE_VALIDATION_BIN`, binary
hashes, or build logs.

Consequently, the CSV comparisons remain evidence about the executed variants,
but **this run cannot be identified confidently as the exact packaged v10 build**.
Descriptions of implementation details below refer to the supplied code; they
cannot prove the exact source of the executed binaries. Before using these
numbers in a release-specific proposal, recover the actual build revisions,
modified harness/read patch, build commands, and binary identities.

## 1. Staged reads

The baseline evaluates both predicates in one scan. The staged prototype scans
the first predicate, collects surviving row IDs, then uses an allowlisted second
scan for the remaining predicate and output. This changes the access plan as
well as predicate evaluation order. Most cases disable statistics and indices
and use late materialization; `stats_1pct` enables statistics in both variants.

Case names `cheap_1`, `cheap_10`, and `cheap_100` mean approximately **0.1%, 1%,
and 10% first-predicate pass rates**, respectively. `cheap_1000` passes all
non-null cheap values; the second predicate still rejects about 80% of rows.
The wide-projection case adds the string `payload` to the returned `id`.

| Case | Baseline / staged median, seconds | Paired latency change, 95% interval | Baseline / staged bytes, MB | Baseline / staged requests |
| --- | ---: | ---: | ---: | ---: |
| No first-stage matches (`cheap_0`) | 3.362 / 1.457 | -56.6% [-58.3%, -55.3%] | 21.324 / 0.353 | 20 / 12 |
| Scattered 0.1% (`cheap_1`) | 3.797 / 3.895 | +0.3% [-10.7%, +6.2%] | 21.354 / 0.856 | 28 / 28 |
| Scattered 1% (`cheap_10`) | 3.799 / 4.517 | +19.4% [+10.9%, +23.7%] | 21.446 / 13.264 | 28 / 28 |
| Scattered 10% (`cheap_100`) | 3.862 / 3.666 | -4.4% [-7.5%, +0.5%] | 21.448 / 21.413 | 30 / 28 |
| Scattered 50% (`cheap_500`) | 4.093 / 3.600 | -10.2% [-16.0%, -8.8%] | 21.448 / 21.447 | 35 / 28 |
| Scattered 75% (`cheap_750`) | 4.290 / 3.665 | -13.8% [-16.7%, -10.5%] | 21.448 / 21.447 | 42 / 28 |
| All non-null cheap values (`cheap_1000`) | 4.377 / 3.566 | -14.9% [-21.0%, -10.9%] | 21.448 / 21.448 | 44 / 28 |
| Wide predicate first, 1% cheap pass | 3.629 / 3.467 | -0.2% [-9.0%, +3.1%] | 21.446 / 21.448 | 28 / 28 |
| Scattered 1%, payload projection | 4.144 / 5.593 | +35.4% [+24.2%, +39.1%] | 22.613 / 14.475 | 36 / 36 |
| Scattered 1%, statistics enabled | 3.730 / 4.560 | +22.7% [+18.6%, +27.9%] | 21.446 / 13.264 | 28 / 28 |
| Clustered 1% | 3.593 / 2.602 | -27.7% [-30.3%, -25.5%] | 21.259 / 0.547 | 22 / 16 |
| Scattered 1%, 64-character strings | 3.108 / 2.970 | -6.1% [-10.3%, -1.8%] | 3.483 / 3.455 | 28 / 28 |
| Scattered 1%, compressible strings | 2.453 / 2.280 | -7.4% [-8.6%, -4.7%] | 0.529 / 0.531 | 28 / 28 |

MB means decimal megabytes. Source: [read-runs.csv](../20261016091115120-835675/read-runs.csv).

### Interpretation

**Fewer bytes alone do not predict lower latency.** The 0.1% case saves 96.0%
of bytes without a clear timing gain. The scattered 1% case saves 38.1% but
regresses in all ten pairs. Adding payload projection or enabling statistics
also regresses in all ten pairs. Total request counts are unchanged in these
comparisons. Two sequential scan stages can increase the critical path even
when their combined request count stays the same. That is a plausible explanation,
but request timing and stage spans are needed to establish it.

**Survivor locality is a strong positive signal.** Clustered 1% survivors save
97.4% of bytes and reduce requests from 22 to 16. The staged scan wins in every
pair. The empty-result case also wins every pair. These are the clearest targets
for continued staged-read work in this storage setup.

**High-pass-rate wins test a different effect.** At 50–100% first-stage pass,
staging saves almost no bytes, while requests fall to 28 from 35–44. All those
cases win every pair. This suggests an access-plan or request-coalescing opportunity
in the baseline path. It does not validate the original hypothesis of avoiding
wide-column I/O. Compare plans and request ranges before building a predicate
ordering policy around these wins. The staged plan also raises median RSS in
the all-pass case from 109.5 to 156.9 MiB.

The [column-only probes](../20261016091115120-835675/column-reads.csv) reinforce
the role of fixed latency: the cheap column reads 0.353 MB in 12 requests but
still takes 1.420 seconds; the default wide column reads 21.232 MB in 12 requests
and takes 2.767 seconds. These are individual diagnostic runs, not latency
estimates for each request.

### Correctness blocks production use

All measured pairs match row count and the checksum of results sorted by `id`.
All 24 extra equivalence cases also pass that comparison, including nulls,
stable-ID-enabled data, deleted rows, and an all-deleted fixture. **Sorted
checksums do not verify output order.**

| Check group | Baseline runs with order violations | Staged runs with order violations |
| --- | ---: | ---: |
| Measured runs | 0 / 130 | 41 / 130 |
| Warm-ups | 0 / 13 | 4 / 13 |
| Additional equivalence cases | 0 / 24 | 9 / 24 |

Measured staged violations occur in `cheap_100` (8/10), `cheap_500` (7/10),
`cheap_750` (9/10), `cheap_1000` (8/10), and `wide_first_1pct` (9/10).
There are 49 recorded violations across the 41 affected measured runs. The
supplied harness counts adjacent IDs that fail strict increase and separately
rejects duplicates. The resulting evidence is an ordering failure for this
monotonic-ID fixture, even though the returned row sets agree.

The `pass` status in [read-correctness.csv](../20261016091115120-835675/read-correctness.csv)
therefore means row-set/payload equivalence only. It is not a complete correctness
pass. An order-preserving implementation must be retested and rebenchmarked;
these timings do not establish its performance. Remapped stable IDs after
compaction, a physically empty dataset, general expression shapes, and scalar
index combinations also remain outside the demonstrated coverage. The
`empty` fixture was created with rows and then fully deleted.

**Decision:** reject unconditional staging. Keep a narrow experiment around
clustered survivors and empty first-stage results. Fix order preservation first,
then estimate touched ranges, request latency, serial stages, and projection
cost, rather than using row selectivity alone. Investigate the baseline's extra
requests in the high-pass cases separately.

## 2. Writer cache and output layout

This experiment tunes existing buffering. It does not implement shared byte
admission across writers. The supplied v10 writer defaults to 8 MiB **per
column**; the tuned value is a total data-cache setting **per writer**. It is
neither a process-wide budget nor a hard RSS limit. A zero cache value in the
CSVs selects the default; it does not disable caching.

The paired 8 MiB treatment also sets `max_page_bytes = 256 KiB`. The 32 MiB
treatment changes the cache alone. All persistent downstream fixtures use the
cache setting alone, including the 8 MiB fixture.

| Workload and cache treatment | Default / tuned median, seconds | Paired elapsed change, 95% interval | Default / tuned peak RSS, MiB | Paired RSS change, 95% interval |
| --- | ---: | ---: | ---: | ---: |
| 1 writer, 4 strings, LZ4; 8 MiB | 2.629 / 2.519 | -6.3% [-10.6%, +0.9%] | 66.6 / 66.7 | 0.0% [-0.5%, +0.6%] |
| 4 writers, 4 strings, ZSTD; 8 MiB | 3.322 / 3.201 | -2.4% [-21.3%, +38.4%] | 89.7 / 88.2 | -1.4% [-3.9%, +2.3%] |
| 4 writers, 64 strings, LZ4; 8 MiB | 11.385 / 12.870 | +35.8% [-8.9%, +74.1%] | 889.8 / 246.8 | -72.1% [-73.0%, -71.3%] |
| 16 writers, 64 strings, LZ4; 8 MiB | 20.828 / 30.578 | +46.7% [+17.5%, +70.8%] | 3491.0 / 580.9 | -83.3% [-84.3%, -82.3%] |
| 4 writers, 4 strings, ZSTD; 32 MiB | 3.145 / 3.082 | -3.0% [-23.5%, +2.0%] | 90.7 / 89.5 | -0.3% [-4.9%, +3.0%] |
| 4 writers, 64 strings, LZ4; 32 MiB | 10.735 / 10.797 | -13.9% [-51.3%, +64.1%] | 877.9 / 497.3 | -43.3% [-45.6%, -42.4%] |
| 16 writers, 64 strings, LZ4; 32 MiB | 20.672 / 21.250 | +10.1% [-8.6%, +15.6%] | 3441.2 / 1825.0 | -47.4% [-48.6%, -42.5%] |

MiB means 1,048,576 bytes. Source: [write-runs.csv](../20261016091115120-835675/write-runs.csv).

The memory reduction is large and consistent in the wide cases. Narrow cases
show little change. In the 16-writer 8 MiB case, elapsed time worsens in every
pair. The 32 MiB memory result is also clear, but its write-time interval permits
a regression above 5%. The wide four-writer timing intervals are especially
broad; neither a speedup nor a small regression has been established.

### Later reads expose a substantial layout cost

Each comparison repeatedly scans one persistent default output and one output
written with the stated total cache setting. All scans return 8,192 rows and
the same ID checksum. Every layout reports **52,801,808 bytes read**.

| Writer cache used for output | Default / tuned scan median, seconds | Paired elapsed change, 95% interval | Default / tuned scan requests |
| --- | ---: | ---: | ---: |
| 8 MiB | 3.731 / 5.187 | +39.6% [+24.3%, +43.6%] | 131 / 1027 |
| 32 MiB | 3.736 / 3.866 | +4.8% [-4.8%, +16.7%] | 131 / 259 |
| 40 MiB | 3.729 / 3.730 | +3.5% [-13.4%, +9.4%] | 131 / 259 |
| 64 MiB | 3.605 / 3.660 | +2.2% [-0.9%, +4.0%] | 131 / 131 |

Source: [downstream-runs.csv](../20261016091115120-835675/downstream-runs.csv).

The 8 MiB output causes **7.84 times as many scanner requests** and higher
median scan CPU time, 670 to 1,144 ms, while bytes read stay equal. This is
consistent with smaller read ranges and additional per-request/per-page work.
Exact page counts and request timing were not collected, so the relative
contributions are unknown. Written object size changes by less than 0.1% in the
wide paired cases; output size alone misses this read cost.

The 32 and 40 MiB outputs nearly double request counts. Their latency intervals
include zero but also permit regressions above 5%; they do not establish harmless
downstream performance. The 64 MiB latency interval is within the proposed 5%
regression bound in this fixture, and its request count matches default.

### What the single-run sweeps add

The [configuration isolation sweep](../20261016091115120-835675/write-config-sweep.csv)
shows that cache-only and cache-plus-page treatments both sharply reduce wide
writer memory. Page-only does not reproduce that reduction. This supports cache
sizing as the main memory control. Timing in this one-run sweep is too variable
to isolate the page setting's performance effect.

The [16-writer cache sweep](../20261016091115120-835675/stress-cache-sweep.csv)
shows an intermediate trade-off: 40 MiB uses 2,182,644 KiB versus 3,409,408 KiB
for default, about 36% less memory in that single comparison. At 64 MiB, RSS is
3,392,516 KiB, only about 0.5% below default. Thus the setting that restores the
default request count provides almost no write-memory benefit in this fixture.
The [one-writer sweep](../20261016091115120-835675/cache-sweep.csv) shows the same
broad trend. These are diagnostic samples, not paired estimates or an established
optimal cache size.

Every paired write records matching readback rows and checksum. In the supplied
harness that check validates `id` and the first and last string columns against
the generator. The runner additionally verifies all columns for each persistent
one-writer cache-sweep output. That all-column verification is checked during
execution but has no separate CSV record. The current data do not prove all-column
equivalence for every concurrent write or across both decoder binaries.

**Decision:** keep cache size as an explicit workload trade-off. The 8 MiB
treatment can be useful under memory pressure when slower writes and later scans
are acceptable. Prioritize repeated 32–40 MiB measurements if a balanced setting
is needed. Do not change the global default or add shared admission based on this
run: encoded bytes in flight, queue depth, admission waiting, and sink idle time
were not measured, so the proposed admission mechanism has not been validated.

## 3. CPU-pool structural decode

The supplied candidate patch moves sufficiently large spawned structural batch
decodes to Lance's CPU pool. Its estimate-based cutoff is 1 MiB. Smaller spawned
tasks retain the Tokio path, and inline decode remains inline. The experiment
runs eight heavy full scans alongside four clients, each making 20 short queries.
The short-only control omits heavy scans. One-worker cases use one Tokio worker
and one CPU worker; the eight-worker case uses eight Tokio and six CPU workers.

The following medians are medians of the **within-run short-query p95** values
from 80 queries. They are not a pooled p95 over all runs.

| Case | Baseline / candidate short p95, ms | Paired short-p95 change, 95% interval | Paired total-elapsed change, 95% interval |
| --- | ---: | ---: | ---: |
| Wide ZSTD, default dispatch, one worker | 1050.5 / 1038.1 | -3.9% [-8.7%, +8.5%] | -1.3% [-3.4%, +15.5%] |
| Wide ZSTD, forced spawn, one worker | 1035.2 / 1123.6 | +4.4% [-11.0%, +12.9%] | +1.0% [-4.2%, +28.0%] |
| Wide ZSTD, forced spawn, eight workers | 1001.8 / 1255.6 | +11.8% [-3.0%, +40.7%] | -0.2% [-8.9%, +35.8%] |
| Narrow LZ4, forced spawn, one worker | 1049.7 / 1009.1 | -5.0% [-10.0%, +2.4%] | +0.6% [-8.8%, +8.5%] |
| Short queries only, 40 pairs | 992.8 / 996.0 | -1.9% [-4.7%, +0.9%] | -0.9% [-2.8%, +1.2%] |

Source: [decode-runs.csv](../20261016091115120-835675/decode-runs.csv).

No case establishes a latency improvement. The eight-worker p95 interval is
particularly broad and allows a substantial regression; it also does not prove
a regression. The narrow and short-only p95 controls stay within the 5% upper
regression bound, while the wide cases do not establish that bound.

The short-only baseline already has about a 993 ms p95 and a 206 ms median
short-query latency. Its total CPU time is about 305 ms over 5.529 seconds of
elapsed time. Large tails therefore exist even without heavy decode. This is
consistent with storage/network waiting or another shared latency source
masking the CPU scheduling effect. It does not identify that source or exclude
short Tokio stalls.

The [isolated scans](../20261016091115120-835675/decode-solo-runs.csv) also show
no clear timing benefit:

| Fixture | Baseline / candidate median, seconds | Paired elapsed change, 95% interval | Bytes / requests, both variants |
| --- | ---: | ---: | ---: |
| Wide ZSTD | 3.416 / 3.437 | +1.1% [-3.5%, +2.2%] | 33,707,808 / 131 |
| Narrow LZ4 | 2.242 / 2.233 | +0.2% [-3.0%, +5.4%] | 3,373,328 / 11 |

These scans match row counts and the ordered ID checksum. The scan consumes all
columns, but its checksum covers only IDs. The concurrent workload checks heavy
row counts and every short query's expected count. Those checks do not establish
all-column decoder correctness. The earlier local decoder tests remain separate
evidence; this analysis did not rebuild or run tests for the remote binaries.

### Memory probe

| Heavy full-scan passes | Baseline median peak RSS, KiB | Candidate median peak RSS, KiB |
| --- | ---: | ---: |
| 8 | 165,484 | 147,244 |
| 64 | 167,480 | 151,004 |

Source: [decode-memory-runs.csv](../20261016091115120-835675/decode-memory-runs.csv).

Candidate RSS is lower in every memory-probe pair. From eight to 64 passes,
median RSS grows by about 1.2% in baseline and 2.6% in candidate. This small
three-pair probe shows no large candidate-specific accumulation under the tested
read-ahead settings. It does not prove a hard memory bound or exclude a slow leak.

**Decision:** object-store performance is inconclusive. The local CPU-contention
result still supports investigating the targeted patch, but this remote run
does not independently support merging it for latency gains. Reconcile build
identity and examine compact decoder trace statistics before making a causal
claim about its remote behavior.

## Comparison with the earlier local findings

The [local summary](../native-layer-ideas-check/SUMMARY.md) used warm NVMe,
Lance commit `cfec2dc5...`, and an i7-9700K. Its staged prototype used a physical
row mask; the supplied object-store prototype uses a row-ID allowlist. Source,
hardware, storage, and some controls differ. The table compares the direction
of findings, not a controlled estimate of storage's effect.

| Question | Local paired result | Object-store paired result | Implication |
| --- | ---: | ---: | --- |
| Scattered 1% staged read | -78.8% elapsed | +19.4% elapsed | The main local target does not transfer to this run. |
| Clustered 1% staged read | -84.7% elapsed | -27.7% elapsed | Locality remains a promising target. |
| All-pass first predicate | +5.2% elapsed | -14.9% elapsed | Plan/request differences matter; remote ordering is also incorrect. |
| 16 wide writers, 8 MiB treatment | -64.3% elapsed; -84.0% RSS | +46.7% elapsed; -83.3% RSS | Memory benefit transfers; write-time benefit reverses. |
| Full scan of 8 MiB output | +19.4% elapsed | +39.6% elapsed | The downstream read penalty is material in both runs. |
| Wide ZSTD decode, one worker, default dispatch | -93.7% short p95 | -3.9%, interval includes zero | No demonstrated remote latency benefit. |

The different prototypes also produce different I/O savings: local scattered
1% staging reads about 0.876 MB, while this run reads 13.264 MB. The reversal
cannot be attributed solely to object-store latency. Read-path selection,
coalescing, branch differences, and the unidentified harness revision may all
matter. Repeat with one verified build and matching access plans to separate
these effects.

## Recommended next steps and trace requirement

1. **Recover build provenance.** Record the actual baseline/candidate source
   revisions, dirty diff, exact harness and patches, binary hashes, and read/write
   binary path. This is required for a version-specific engineering decision.
2. **Repair staged order preservation.** Assert ordered output directly across
   fragments, predicate orders, projections, and stable-ID cases. Then repeat
   clustered 1%, scattered 0.1% and 1%, payload projection, and all-pass controls.
   Record per-stage elapsed time and request-range/timing summaries.
3. **Repeat cache settings around 32–40 MiB.** Pair write RSS/time measurements
   with downstream scans, page counts, and request counts. The 32 MiB memory gain
   is established; a downstream cost below 5% is not. There is no need to repeat
   the full sweep before resolving this narrower trade-off.
4. **Use decoder traces to explain, not replace, paired timings.** Extract task
   counts by thread pool, `task_to_batch` duration distributions, scan overlap,
   and queue-wait fields if the export retains them. To validate a CPU-bound
   benefit, use a controlled workload where short-query latency is not already
   dominated by the remote setup.

The absent `decode-baseline-trace.json` and `decode-candidate-trace.json` are
**not required to finish this report**. Without them, this analysis cannot confirm
that the executed candidate actually dispatched large decodes to `lance-cpu`,
measure decode-span durations, or relate task placement to the observed tails.
A compact trace summary would usually be sufficient; the full large files are
only needed if that summary leaves a specific question unresolved. Traces cannot
resolve the harness/patch hash mismatch and cannot turn inconclusive paired
timings into a demonstrated speedup.

## Reproduction of the analysis

Run from the repository root; Node built-ins are the only dependencies:

```bash
node native-layer-v10-check/analyze-results.mjs
```

The [analysis script](analyze-results.mjs) validates pair completeness, pair
order, recorded equivalence, expected counts, and the generated README figures.
It writes [machine-readable results](object-store-analysis-20261016091115120-835675.json)
with all recomputed statistics, source CSV hashes, artifact-hash comparisons,
ordering counts, memory medians, and diagnostic sweeps. Optional first and second
arguments select another input directory and output JSON path. No benchmarks,
remote access, or Rust/Python changes are part of this analysis.

**Conclusion:** the strongest reusable finding is that write buffering must be
evaluated together with its later read-request cost. Staged filtering remains
worth pursuing where it avoids whole ranges, with output order preserved. CPU
decode separation has local evidence, while its object-store benefit remains
unverified. None of these results supports a universal default change.
