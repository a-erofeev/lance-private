# Native-layer ideas on current Lance: local validation

The source analysis and plan in `../../q2/ideas_for_lance/` were accessible.
Tests used Lance commit `cfec2dc5b77f0fb46e5d5b5d3b491c8c28c85ab1`,
local NVMe storage, the `release-with-debug` profile, warm storage cache, and
fresh Lance processes. No object store was available. Each main case has ten
alternating AB/BA pairs after warm-up. Negative changes mean the variant is
faster; intervals are 95% bootstrap intervals for the median paired change.

| Candidate and case | Baseline median / p95 | Variant median / p95 | Paired change, 95% interval | Physical I/O or peak RSS | Correctness | Decision |
| --- | ---: | ---: | ---: | --- | --- | --- |
| Staged read, cheap first, 1% pass, wide string | 29.9 / 31.2 ms | 6.3 / 6.9 ms | -78.8% [-79.8%, -78.0%] | 21.2 / 0.88 MB; 24 / 32 requests | pass | Narrow: selective wide predicates only |
| Staged read, 100% pass control | 31.3 / 32.4 ms | 32.8 / 33.3 ms | +5.2% [+2.4%, +7.6%] | 21.2 / 21.2 MB; 34 / 42 requests | pass | Reject unconditional staging |
| CPU decode, wide ZSTD, one Tokio worker; short queries | 6.8 / 95.6 ms; 789 ms total | 3.0 / 6.1 ms; 647 ms total | -93.7% [-93.8%, -93.6%] short p95 | 156 / 151 MiB peak RSS | pass | Targeted decoder change for review |
| Write cache 8 MiB, 16 wide writers | 745 / 749 ms | 266 / 276 ms | -64.3% [-64.9%, -63.8%] | 1,176 / 188 MiB peak RSS | pass | Configuration helps writes |
| Full scan of 8 MiB cache output | 32.6 / 33.9 ms | 39.1 / 40.5 ms | +19.4% [+16.2%, +22.9%] | 52.82 / 52.86 MB read | pass | Do not use as universal default |
| Write cache 32 MiB, 16 wide writers | 718 / 722 ms | 486 / 492 ms | -32.6% [-33.5%, -31.7%] | 1,150 / 672 MiB peak RSS | pass | Workload-specific trade-off |

The staged-read target passes at selective rates, but controls exceed the
proposed 5% regression gate. The two-scan prototype needs a reliable fallback
and an object-store test. The CPU-pool change has a strong one-worker p95
gain; the 40-pair short-only repeat and narrow control stay within the 5% p95
regression gate. Isolated wide full scans also improved by 1.9%
[-4.2%, -1.4%] across 20 pairs; the narrow full-scan change is inconclusive.
The eight-worker p95 change remains inconclusive. The writer's existing cache
setting explains most of the memory change. Smaller caches also slow later
full scans; shared byte admission is not justified by these data.

See [read-decision.md](read-decision.md), [decode-decision.md](decode-decision.md),
and [write-decision.md](write-decision.md) for details, raw-data links, and
remaining limits. The structural decoder change is present in the working
tree and saved in [decode-cpu-final.patch](decode-cpu-final.patch).

## Verification and limits

- All 24 staged-read equivalence cases passed. Baseline and candidate CPU
  builds returned the same 64-column, 8,192-row checksum. Paired writes passed
  readback, and persistent wide outputs passed all-column readback.
- All 21 decoder module tests passed with one CPU worker, including decode
  error, dropped-consumer, fallback, and large-batch tests. `cargo fmt` check,
  package Clippy with `-D warnings`, and repository-wide
  `cargo check --workspace --tests --benches --offline` passed. JavaScript
  syntax checks passed for the changed decode scripts.
- Object-store latency, controlled cold cache, remapped stable row IDs, and
  writer queue-depth/sink-idle metrics remain unmeasured. The Chrome traces
  show task placement but did not retain numeric queue-wait fields. The
  existing decoder Criterion bench does not exercise the changed spawn path.
