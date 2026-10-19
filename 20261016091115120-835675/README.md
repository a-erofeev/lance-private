# Object store experiment: 20261016091115120-835675

Endpoint: <endpoint>; data prefix: <data_path>. Main cases use 10 alternating AB/BA pairs after a warm-up per variant; the short-only decoder control uses 40 and isolated decoder scans use 20. Elapsed time excludes process startup; every run uses a fresh process and Lance session. Object-store caches are not controlled. The same fixture is used by each paired read or decode variant. A new prefix is used for every write. Output bytes include dataset objects found under each written path. The read fixture has 65,536 rows in four fragments; write inputs have eight 1,024-row batches shared across writers. The wide write and decode fixtures have 64 string columns. Intervals use 10,000 bootstrap resamples of the median paired percentage change. See [run-config.json](run-config.json) for source, machine, and run settings.

## Staged reads

| Case | baseline median / p95 | staged median / p95 | Median paired change, 95% interval |
| --- | ---: | ---: | ---: |
| cheap_0 | 3361.8 / 3425.1 | 1456.9 / 1559.5 | -56.6% [-58.3%, -55.3%] |
| cheap_1 | 3796.8 / 4003.8 | 3895.0 / 4213.2 | 0.3% [-10.7%, 6.2%] |
| cheap_10 | 3798.8 / 4226.9 | 4516.9 / 5300.2 | 19.4% [10.9%, 23.7%] |
| cheap_100 | 3861.5 / 4057.5 | 3665.7 / 3884.7 | -4.4% [-7.5%, 0.5%] |
| cheap_500 | 4093.0 / 4321.8 | 3600.4 / 3790.7 | -10.2% [-16.0%, -8.8%] |
| cheap_750 | 4289.8 / 4491.0 | 3665.3 / 3783.4 | -13.8% [-16.7%, -10.5%] |
| cheap_1000 | 4376.9 / 4588.9 | 3565.8 / 3918.3 | -14.9% [-21.0%, -10.9%] |
| wide_first_1pct | 3628.6 / 3930.0 | 3467.3 / 3908.3 | -0.2% [-9.0%, 3.1%] |
| wide_projection_1pct | 4144.0 / 4456.6 | 5592.7 / 5768.7 | 35.4% [24.2%, 39.1%] |
| stats_1pct | 3730.3 / 3915.4 | 4559.6 / 4974.8 | 22.7% [18.6%, 27.9%] |
| clustered_1pct | 3593.4 / 3732.4 | 2602.2 / 2731.0 | -27.7% [-30.3%, -25.5%] |
| medium_1pct | 3108.0 / 3375.1 | 2970.4 / 3044.3 | -6.1% [-10.3%, -1.8%] |
| compressible_1pct | 2452.6 / 2526.6 | 2279.8 / 2347.8 | -7.4% [-8.6%, -4.7%] |

Raw data: [read-runs.csv](read-runs.csv). Negative change favors staged.


## Read bytes

| Case | baseline median / p95 | staged median / p95 | Median paired change, 95% interval |
| --- | ---: | ---: | ---: |
| cheap_0 | 21323745.0 / 21323745.0 | 353408.0 / 353408.0 | -98.3% [-98.3%, -98.3%] |
| cheap_1 | 21353953.0 / 21353953.0 | 856108.0 / 856108.0 | -96.0% [-96.0%, -96.0%] |
| cheap_10 | 21445969.0 / 21445969.0 | 13264448.0 / 13264448.0 | -38.1% [-38.1%, -38.1%] |
| cheap_100 | 21447937.0 / 21447937.0 | 21413396.0 / 21413396.0 | -0.2% [-0.2%, -0.2%] |
| cheap_500 | 21448001.0 / 21448018.6 | 21446693.0 / 21446693.0 | -0.0% [-0.0%, -0.0%] |
| cheap_750 | 21448097.0 / 21448097.0 | 21447086.0 / 21447086.0 | -0.0% [-0.0%, -0.0%] |
| cheap_1000 | 21448097.0 / 21448097.0 | 21447904.0 / 21447904.0 | -0.0% [-0.0%, -0.0%] |
| wide_first_1pct | 21445969.0 / 21445969.0 | 21447905.0 / 21447905.0 | 0.0% [0.0%, 0.0%] |
| wide_projection_1pct | 22613425.0 / 22613425.0 | 14474672.0 / 14474672.0 | -36.0% [-36.0%, -36.0%] |
| stats_1pct | 21445969.0 / 21445969.0 | 13264448.0 / 13264448.0 | -38.1% [-38.1%, -38.1%] |
| clustered_1pct | 21259048.0 / 21259048.0 | 546832.0 / 546832.0 | -97.4% [-97.4%, -97.4%] |
| medium_1pct | 3483176.0 / 3483176.0 | 3454912.0 / 3454912.0 | -0.8% [-0.8%, -0.8%] |
| compressible_1pct | 529252.0 / 529252.0 | 531188.0 / 531188.0 | 0.4% [0.4%, 0.4%] |

Raw data: [read-runs.csv](read-runs.csv). Negative change favors staged.


## Read requests

| Case | baseline median / p95 | staged median / p95 | Median paired change, 95% interval |
| --- | ---: | ---: | ---: |
| cheap_0 | 20.0 / 20.0 | 12.0 / 12.0 | -40.0% [-40.0%, -40.0%] |
| cheap_1 | 28.0 / 28.0 | 28.0 / 28.0 | 0.0% [0.0%, 0.0%] |
| cheap_10 | 28.0 / 28.0 | 28.0 / 28.0 | 0.0% [0.0%, 0.0%] |
| cheap_100 | 30.0 / 30.0 | 28.0 / 28.0 | -6.7% [-6.7%, -6.7%] |
| cheap_500 | 35.0 / 35.5 | 28.0 / 28.0 | -20.0% [-20.0%, -17.6%] |
| cheap_750 | 42.0 / 42.0 | 28.0 / 28.0 | -33.3% [-33.3%, -33.3%] |
| cheap_1000 | 44.0 / 44.0 | 28.0 / 28.0 | -36.4% [-36.4%, -34.9%] |
| wide_first_1pct | 28.0 / 28.0 | 28.0 / 28.0 | 0.0% [0.0%, 0.0%] |
| wide_projection_1pct | 36.0 / 36.0 | 36.0 / 36.0 | 0.0% [0.0%, 0.0%] |
| stats_1pct | 28.0 / 28.0 | 28.0 / 28.0 | 0.0% [0.0%, 0.0%] |
| clustered_1pct | 22.0 / 22.0 | 16.0 / 16.0 | -27.3% [-27.3%, -27.3%] |
| medium_1pct | 28.0 / 28.0 | 28.0 / 28.0 | 0.0% [0.0%, 0.0%] |
| compressible_1pct | 28.0 / 28.0 | 28.0 / 28.0 | 0.0% [0.0%, 0.0%] |

Raw data: [read-runs.csv](read-runs.csv). Negative change favors staged.


Read order: 41 of 260 measured runs and 4 of 26 warm-ups had one or more adjacent ID decreases. The result checksum uses rows sorted by ID. Equal checksums do not establish ordered-scan equivalence. [Measured counts](read-runs.csv); [warm-up counts](read-warmups.csv).

Read row-set equivalence: 24 cases passed. [Details](read-correctness.csv). [Column I/O](column-reads.csv).

## Writer elapsed time

| Case | default median / p95 | tuned median / p95 | Median paired change, 95% interval |
| --- | ---: | ---: | ---: |
| normal_1writer_4col_lz4_cache8 | 2628.8 / 9097.7 | 2519.1 / 4146.0 | -6.3% [-10.6%, 0.9%] |
| normal_4writer_4col_zstd_cache8 | 3322.4 / 5606.3 | 3201.1 / 5668.2 | -2.4% [-21.3%, 38.4%] |
| stress_4writer_64col_lz4_cache8 | 11385.0 / 15195.9 | 12869.7 / 22190.5 | 35.8% [-8.9%, 74.1%] |
| stress_16writer_64col_lz4_cache8 | 20827.7 / 28741.5 | 30578.2 / 43136.0 | 46.7% [17.5%, 70.8%] |
| normal_4writer_4col_zstd_cache32 | 3145.2 / 5273.3 | 3081.8 / 3577.0 | -3.0% [-23.5%, 2.0%] |
| stress_4writer_64col_lz4_cache32 | 10734.6 / 25781.8 | 10797.1 / 19141.2 | -13.9% [-51.3%, 64.1%] |
| stress_16writer_64col_lz4_cache32 | 20672.1 / 26299.7 | 21249.9 / 27942.0 | 10.1% [-8.6%, 15.6%] |

Raw data: [write-runs.csv](write-runs.csv). Negative change favors tuned.


## Writer peak RSS (KiB)

| Case | default median / p95 | tuned median / p95 | Median paired change, 95% interval |
| --- | ---: | ---: | ---: |
| normal_1writer_4col_lz4_cache8 | 68202.0 / 68623.2 | 68274.0 / 68725.8 | 0.0% [-0.5%, 0.6%] |
| normal_4writer_4col_zstd_cache8 | 91828.0 / 97089.2 | 90308.0 / 97461.2 | -1.4% [-3.9%, 2.3%] |
| stress_4writer_64col_lz4_cache8 | 911178.0 / 930087.2 | 252684.0 / 267776.8 | -72.1% [-73.0%, -71.3%] |
| stress_16writer_64col_lz4_cache8 | 3574754.0 / 3747588.4 | 594876.0 / 667567.0 | -83.3% [-84.3%, -82.3%] |
| normal_4writer_4col_zstd_cache32 | 92838.0 / 105643.2 | 91618.0 / 94896.0 | -0.3% [-4.9%, 3.0%] |
| stress_4writer_64col_lz4_cache32 | 898976.0 / 946497.6 | 509280.0 / 521421.2 | -43.3% [-45.6%, -42.4%] |
| stress_16writer_64col_lz4_cache32 | 3523788.0 / 3653176.4 | 1868846.0 / 1940629.0 | -47.4% [-48.6%, -42.5%] |

Raw data: [write-runs.csv](write-runs.csv). Negative change favors tuned.


## Written object bytes

| Case | default median / p95 | tuned median / p95 | Median paired change, 95% interval |
| --- | ---: | ---: | ---: |
| normal_1writer_4col_lz4_cache8 | 3310226.0 / 3310226.0 | 3310226.0 / 3310226.0 | 0.0% [0.0%, 0.0%] |
| normal_4writer_4col_zstd_cache8 | 8467968.0 / 8467968.0 | 8467967.0 / 8467968.0 | -0.0% [-0.0%, 0.0%] |
| stress_4writer_64col_lz4_cache8 | 211055071.0 / 211055072.0 | 211203423.0 / 211203424.0 | 0.1% [0.1%, 0.1%] |
| stress_16writer_64col_lz4_cache8 | 844220283.5 / 844220285.5 | 844813691.0 / 844813693.1 | 0.1% [0.1%, 0.1%] |
| normal_4writer_4col_zstd_cache32 | 8467967.0 / 8467968.0 | 8467967.0 / 8467968.0 | 0.0% [-0.0%, 0.0%] |
| stress_4writer_64col_lz4_cache32 | 211055071.0 / 211055072.0 | 211076571.5 / 211076572.0 | 0.0% [0.0%, 0.0%] |
| stress_16writer_64col_lz4_cache32 | 844220284.5 / 844220286.5 | 844306284.5 / 844306287.0 | 0.0% [0.0%, 0.0%] |

Raw data: [write-runs.csv](write-runs.csv). Negative change favors tuned.


## Downstream full scans

| Case | default median / p95 | tuned median / p95 | Median paired change, 95% interval |
| --- | ---: | ---: | ---: |
| cache_8 | 3731.2 / 4833.9 | 5187.0 / 5366.5 | 39.6% [24.3%, 43.6%] |
| cache_32 | 3736.3 / 4173.1 | 3866.4 / 4655.9 | 4.8% [-4.8%, 16.7%] |
| cache_40 | 3728.7 / 4814.7 | 3730.3 / 4754.8 | 3.5% [-13.4%, 9.4%] |
| cache_64 | 3605.1 / 3921.6 | 3659.8 / 4377.5 | 2.2% [-0.9%, 4.0%] |

Raw data: [downstream-runs.csv](downstream-runs.csv). Negative change favors tuned.


## Downstream read bytes

| Case | default median / p95 | tuned median / p95 | Median paired change, 95% interval |
| --- | ---: | ---: | ---: |
| cache_8 | 52801808.0 / 52801808.0 | 52801808.0 / 52801808.0 | 0.0% [0.0%, 0.0%] |
| cache_32 | 52801808.0 / 52801808.0 | 52801808.0 / 52801808.0 | 0.0% [0.0%, 0.0%] |
| cache_40 | 52801808.0 / 52801808.0 | 52801808.0 / 52801808.0 | 0.0% [0.0%, 0.0%] |
| cache_64 | 52801808.0 / 52801808.0 | 52801808.0 / 52801808.0 | 0.0% [0.0%, 0.0%] |

Raw data: [downstream-runs.csv](downstream-runs.csv). Negative change favors tuned.


## Downstream read requests

| Case | default median / p95 | tuned median / p95 | Median paired change, 95% interval |
| --- | ---: | ---: | ---: |
| cache_8 | 131.0 / 131.0 | 1027.0 / 1027.0 | 684.0% [684.0%, 684.0%] |
| cache_32 | 131.0 / 131.0 | 259.0 / 259.0 | 97.7% [97.7%, 97.7%] |
| cache_40 | 131.0 / 131.0 | 259.0 / 259.0 | 97.7% [97.7%, 97.7%] |
| cache_64 | 131.0 / 131.0 | 131.0 / 131.0 | 0.0% [0.0%, 0.0%] |

Raw data: [downstream-runs.csv](downstream-runs.csv). Negative change favors tuned.


Cache and page sweeps are single runs per setting. [One writer](cache-sweep.csv); [16 writers](stress-cache-sweep.csv); [configuration isolation](write-config-sweep.csv). The paired runs provide stronger evidence.

## Concurrent short query p95

| Case | baseline median / p95 | candidate median / p95 | Median paired change, 95% interval |
| --- | ---: | ---: | ---: |
| wide_zstd_default_one_worker | 1050.5 / 1184.8 | 1038.1 / 1239.0 | -3.9% [-8.7%, 8.5%] |
| wide_zstd_spawn_one_worker | 1035.2 / 1411.6 | 1123.6 / 1389.4 | 4.4% [-11.0%, 12.9%] |
| wide_zstd_spawn_eight_workers | 1001.8 / 1464.2 | 1255.6 / 1423.8 | 11.8% [-3.0%, 40.7%] |
| narrow_lz4_spawn_one_worker | 1049.7 / 1233.9 | 1009.1 / 1087.6 | -5.0% [-10.0%, 2.4%] |
| short_only_spawn_one_worker | 992.8 / 1122.0 | 996.0 / 1114.9 | -1.9% [-4.7%, 0.9%] |

Raw data: [decode-runs.csv](decode-runs.csv). Negative change favors candidate.


## Concurrent total elapsed time

| Case | baseline median / p95 | candidate median / p95 | Median paired change, 95% interval |
| --- | ---: | ---: | ---: |
| wide_zstd_default_one_worker | 10632.4 / 10839.2 | 10388.8 / 14470.2 | -1.3% [-3.4%, 15.5%] |
| wide_zstd_spawn_one_worker | 10699.7 / 10931.1 | 10544.8 / 17011.9 | 1.0% [-4.2%, 28.0%] |
| wide_zstd_spawn_eight_workers | 10040.3 / 13453.8 | 10018.0 / 15466.5 | -0.2% [-8.9%, 35.8%] |
| narrow_lz4_spawn_one_worker | 7338.6 / 7845.9 | 6947.7 / 7820.1 | 0.6% [-8.8%, 8.5%] |
| short_only_spawn_one_worker | 5528.8 / 5890.9 | 5524.1 / 5787.9 | -0.9% [-2.8%, 1.2%] |

Raw data: [decode-runs.csv](decode-runs.csv). Negative change favors candidate.


## Isolated full scans

| Case | baseline median / p95 | candidate median / p95 | Median paired change, 95% interval |
| --- | ---: | ---: | ---: |
| wide | 3416.4 / 3623.7 | 3436.9 / 3606.6 | 1.1% [-3.5%, 2.2%] |
| narrow | 2241.5 / 2360.3 | 2233.1 / 2390.7 | 0.2% [-3.0%, 5.4%] |

Raw data: [decode-solo-runs.csv](decode-solo-runs.csv). Negative change favors candidate.


Eight versus 64 full scan passes: [peak RSS data](decode-memory-runs.csv). [Baseline trace](decode-baseline-trace.json); [candidate trace](decode-candidate-trace.json). The short-only control uses 40 pairs.
