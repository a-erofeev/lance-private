# Native-layer ideas: local Lance validation

This directory contains the local-storage experiments for the plan in
`../../q2/ideas_for_lance/validation_plan.md`. The source directory was readable
on 2026-10-08. The tested Lance checkout is
`cfec2dc5b77f0fb46e5d5b5d3b491c8c28c85ab1`. The original analysis used
an older Lance commit, so its performance claims are hypotheses for this run.

## Machine and run conditions

| Item | Value |
| --- | --- |
| CPU | Intel Core i7-9700K, 8 physical cores, 8 logical CPUs |
| RAM | 31 GiB installed; about 5-7 GiB available during setup |
| Storage | Local ext4 on `/dev/nvme0n1p2`, NVMe, no object store |
| OS | Linux 7.0.0-34-generic |
| Rust | `rustc 1.98.1`, `cargo 1.98.1` |
| Node | v24.21.0, used only to orchestrate runs and calculate intervals |
| Default runtime | Tokio's default worker count; Lance CPU pool default is CPU count minus two reserved I/O cores |
| Cache label | OS storage cache warm; each measured query/write uses a new process and new Lance session |
| Raw result profile | `dev` for files named `*-debug-*`; final runs use `release-with-debug` |
| File format | Default stable V2.2 (`WriteParams.data_storage_version = None`) |

The benchmark fixture source is [harness.rs](harness.rs). Its standalone
[Cargo.toml](Cargo.toml) avoids linking Lance's unrelated example development
dependencies.
The read fixture uses seed `0x15A7_2026`, 65,536 rows, 1,024-row batches, and
four 16,384-row fragments. Columns are `id`, a nullable integer `cheap`, a
nullable string `wide`, and a payload string. The default wide string has 512
pseudo-random lowercase characters for nonmatching rows. One in five non-null
rows has `wide = 'MATCH'`. The cheap predicate is `cheap < N`, with N from 0
through 1,000. A clustered fixture places the approximately 1% cheap matches
in the first 655 rows. Other fixtures use 64-character or highly compressible
wide strings. Read timings exclude fixture creation and output validation.

The write input has eight distinct 1,024-row batches, with 4 or 64 string
columns of 96 pseudo-random lowercase characters. Each process builds the
input before timing, then shares the Arrow arrays among 1, 4, or 16 concurrent
writers. Writers use separate fresh dataset paths. `lz4` or `zstd` is set in
field metadata. The default writer uses its normal cache and page options;
the tuned writer sets `data_cache_bytes = 8 MiB` per writer and
`max_page_bytes = 256 KiB`. Write timing includes the dataset writes but excludes
input construction, readback verification, and cleanup.

## Results

- [Cross-candidate summary and result table](SUMMARY.md)
- [Staged read decision](read-decision.md), [summary](read-results.md),
  [raw paired runs](read-runs.csv), and [per-column bytes](column-reads.csv)
- [Read correctness checks](read-correctness.json), including nullable values,
  empty output, deletions, stable row IDs, both predicate orders, and both
  projections
- [Write decision](write-decision.md), [8 MiB paired results](write-results.md),
  [32 MiB paired results](write-cache32-results.md),
  [configuration sweep](write-config-sweep.csv),
  [cache/scan trade-off](cache-tradeoff-results.md), and
  [downstream scan results](downstream-results.md)
- [CPU decode decision](decode-decision.md), [final paired results](decode-results.md),
  [raw runs](decode-runs.csv), [40-pair short-only repeat](decode-short-control-results.md),
  [isolated full-scan comparison](decode-solo-results.md),
  [longer-run memory data](decode-memory-runs.csv),
  [final source patch](decode-cpu-final.patch), and
  [baseline](decode-baseline-trace.json) and
  [candidate](decode-candidate-trace.json) Chrome traces. The original
  [prototype patch](decode-cpu-prototype.patch) and
  [prototype results](decode-prototype-results.md) are retained separately.
- [Environment and exact run conditions](environment.md)

For each case, the scripts warm up once per variant, then run ten measured
pairs in AB, BA order. Intervals are percentile bootstrap intervals from
10,000 resamples of the paired percentage changes. A new process is used for
each measurement. Timings and CPU measurements exclude process startup;
peak RSS is the process high-water mark and therefore includes input creation.
Reported read bytes and request counts come from Lance scan callbacks.

The staged-read variant first scans the first predicate column and `_rowid`,
then scans the other predicate and output columns using a row-ID mask. It is a
feasibility prototype built with Lance's public scanner API. It does not use
an integrated two-stage executor. It reads the same Lance files as the
baseline, and the benchmark drains each output stream while checking row
order and hashing output values. The variant is only used for two simple `AND`
predicates on different columns. Other expression shapes continue to use the
baseline scanner.

## Reproduction

For S3-compatible object store runs, use the separate
[object store instructions](object-store/README.md). They include a source-only
transfer archive, config file and environment-variable setup, binary builds,
and one-command report generation for each investigation.

Run from the repository root:

```bash
CARGO_TARGET_DIR="$PWD/target" cargo build --manifest-path native-layer-ideas-check/Cargo.toml --profile release-with-debug --offline
target/release-with-debug/native_layer_ideas_check fixture native-layer-ideas-check/read-fixture.lance
LANCE_VALIDATION_BIN="$PWD/target/release-with-debug/native_layer_ideas_check" node native-layer-ideas-check/run-read.mjs
node native-layer-ideas-check/analyze-read.mjs
LANCE_VALIDATION_BIN="$PWD/target/release-with-debug/native_layer_ideas_check" node native-layer-ideas-check/run-write.mjs
node native-layer-ideas-check/analyze-write.mjs
```

The 32 MiB write check is `run-write-cache32.mjs` followed by
`analyze-write.mjs -cache32`. The cache sweep and downstream scripts accept
additional arguments documented in their source.

To reproduce the CPU comparison, build an unmodified baseline from the recorded
commit in a separate worktree using this harness and lockfile. Build the current
checkout as the candidate with the command above. Set `LANCE_BASELINE_BIN` and
`LANCE_CANDIDATE_BIN` to the two binary paths, then run `run-decode.mjs` and
`analyze-decode.mjs`. Run `run-decode-memory.mjs` with the same binaries for
the 8-versus-64-pass memory check. The current structural decoder change is
also saved as `decode-cpu-final.patch`.

The repeated short-only control uses `LANCE_DECODE_CASE=short_only_spawn_one_worker`,
`LANCE_DECODE_PAIRS=40`, and `LANCE_DECODE_OUTPUT=decode-short-control-runs.csv`
with `run-decode.mjs`. Analyze it with
`node native-layer-ideas-check/analyze-decode.mjs decode-short-control-runs.csv decode-short-control-results.md 40`.
Run `run-decode-solo.mjs` and `analyze-decode-solo.mjs` with the same binary
paths for the isolated full-scan comparison.

The fixture command rejects an existing path. To reproduce all read cases,
create the clustered, medium, and compressible fixtures using the optional
fixture arguments documented by the binary. The write script deletes only its
own generated output paths after each run. Do not run benchmark processes in
parallel. The scripts set a 120-second read timeout and a 300-second write
timeout per process.

## Scope and limits

These measurements apply to a local NVMe filesystem with a warm OS cache.
They are not cloud or controlled-cold results. The two-scan staged prototype
has extra planning and row-mask costs that an integrated executor may change.
The paired write readback validates row count, `id`, and the first and last
string columns for every output. Persistent 64-column outputs also passed
row-by-row, all-column checks. No shared writer admission or production
staged-read change has been made. Configuration sweeps have one write run per
setting; paired results are the evidence for performance decisions. The CPU
decode change now has focused error, cancellation, fallback, and one-worker
tests. The local results do not establish object-store performance.
