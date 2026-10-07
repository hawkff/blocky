# Wildcard list memory on ARM64

The compact wildcard cache partitions entries by TLD and stores the remaining names in
sorted, concatenated length buckets. A two-bit membership filter rejects most misses
before binary search. It uses the standard library and adds no dependencies.

Lookups search the broadest parent first and return the same `*.base` rule as the trie.
Entry normalization and duplicate removal remain in place. Exact and regex matching
are unchanged. Direct wildcard-cache calls retain the trie's case-sensitive query
contract; the resolver lowercases DNS questions before the lookup.

## Measurement setup

Measurements used native Linux/ARM64 in Namespace, four CPUs, 16 GiB RAM,
Go 1.26.8 and `GOMAXPROCS=4`. The baseline is upstream commit `36a950b9`.
Both binaries used `GOOS=linux GOARCH=arm64 CGO_ENABLED=0`, `-trimpath` and
`-ldflags='-s -w'`. These are container measurements, not device measurements.

The inputs were downloaded once on 2026-10-07 and reused from local files outside the
repository. This TIF snapshot has 1,837,872 wildcard entries. List sizes change over time.
Sources are HaGeZi's `wildcard/tif.txt`, `wildcard/tif.medium.txt` and
`wildcard/ultimate.txt` from
[the dns-blocklists repository](https://github.com/hagezi/dns-blocklists), plus
[OISD domainswild](https://big.oisd.nl/domainswild).

## Retained heap

Each cache was built from a streamed file. The table reports the increase in
`runtime.MemStats.HeapAlloc` after `runtime.GC()`, with only the finished cache kept
alive. The exact-name control strips `*.` from the same input. Its storage is unchanged.

| List | Wildcard entries | Exact control, MiB | Trie before, MiB | Compact after, MiB | Wildcard reduction |
| --- | ---: | ---: | ---: | ---: | ---: |
| TIF full | 1,837,872 | 28.53 | 149.86 | 24.52 | 83.6% |
| TIF medium | 535,300 | 8.77 | 47.22 | 7.94 | 83.2% |
| Ultimate | 234,839 | 4.16 | 21.79 | 4.01 | 81.6% |
| OISD | 240,564 | 4.53 | 21.11 | 4.30 | 79.6% |

## Lookup cost

Median of three one-second benchmark samples, in ns/op. Each workload cycles through
one entry per 997 input rules. Base hits query the stored base without `*.`; subdomain
hits add a label. Unknown-TLD misses append `.invalid`; known-TLD misses change the
leftmost label. The benchmark verifies the expected hit or miss before timing.

| List | Lookup | Trie before, ns/op | Compact after, ns/op | Before allocs/op | After allocs/op |
| --- | --- | ---: | ---: | ---: | ---: |
| TIF full | Base hit | 2862 | 2339 | 10 | 7 |
| TIF full | Subdomain hit | 2382 | 2035 | 10 | 7 |
| TIF full | Unknown-TLD miss | 27.79 | 22.51 | 0 | 0 |
| TIF full | Known-TLD miss | 236.4 | 71.33 | 0 | 0 |
| TIF medium | Base hit | 2056 | 1889 | 10 | 7 |
| TIF medium | Subdomain hit | 1948 | 1661 | 10 | 7 |
| TIF medium | Unknown-TLD miss | 25.52 | 22.38 | 0 | 0 |
| TIF medium | Known-TLD miss | 81.5 | 59.36 | 0 | 0 |
| Ultimate | Base hit | 1907 | 1529 | 10 | 7 |
| Ultimate | Subdomain hit | 1809 | 1436 | 10 | 7 |
| Ultimate | Unknown-TLD miss | 25.58 | 24.68 | 0 | 0 |
| Ultimate | Known-TLD miss | 83.75 | 63.64 | 0 | 0 |
| OISD | Base hit | 1760 | 1517 | 10 | 7 |
| OISD | Subdomain hit | 1672 | 1422 | 10 | 7 |
| OISD | Unknown-TLD miss | 25.7 | 22.61 | 0 | 0 |
| OISD | Known-TLD miss | 75.6 | 61.98 | 0 | 0 |

Hits still include the existing logging-prefix allocations. Misses allocate nothing.
The exact-name control retains six allocations per hit and zero per miss. Its timing
varied between runs despite unchanged code, so small timing differences should not be
read as precise speedup estimates.

| Exact control | Before hit, ns/op | After hit, ns/op | Before unknown-TLD miss, ns/op | After unknown-TLD miss, ns/op |
| --- | ---: | ---: | ---: | ---: |
| TIF full | 2300 | 2054 | 431.3 | 532.2 |
| TIF medium | 1744 | 1617 | 270 | 263.6 |
| Ultimate | 1281 | 1217 | 253.6 | 251 |
| OISD | 1308 | 1186 | 261.7 | 255.8 |

## Process memory through refresh

The end-to-end configuration loads all four files as separate denylist groups and
selects all groups. Startup uses `failOnError`, source concurrency four, and no periodic
refresh. Each sample starts a fresh process, waits for its HTTP listener, waits five
seconds, records RSS, then runs `blocky lists refresh`. It records `VmHWM` after the
command completes and RSS five seconds later. `VmHWM` includes startup; it was not reset.
There is no query traffic during these measurements. Values below are medians of
three runs, with default `GOGC=100` and no memory limit unless stated.
The parser rejects one malformed punycode entry in full TIF on both revisions, leaving
1,837,871 entries in that group. The cache-only benchmark includes all raw wildcard lines.

| Variant | RSS after load, MiB | VmHWM after refresh, MiB | RSS after refresh, MiB | Refresh, seconds |
| --- | ---: | ---: | ---: | ---: |
| Before, trie and parallel groups | 349.0 | 882.1 | 882.1 | 7.05 |
| After, compact and serial groups | 97.6 | 255.2 | 168.2 | 9.97 |
| After, experimental GC after each swap | 95.9 | 254.2 | 133.2 | 9.82 |
| After, experimental FreeOSMemory after each swap | 92.6 | 252.7 | 100.1 | 9.81 |
| After, GOGC=50 | 91.5 | 217.3 | 135.4 | 10.64 |
| After, GOMEMLIMIT=256MiB | 95.1 | 255.0 | 172.9 | 10.32 |
| After, GOMEMLIMIT=192MiB | 95.6 | 208.1 | 148.6 | 10.83 |

Serial group refresh limits construction to one replacement group at a time, at the cost
of refresh throughput. Readers keep the old group until the replacement swaps in.
Overlapping refresh requests for the same cache serialize too. Denylists and allowlists
have independent refresh locks.

`debug.FreeOSMemory()` after each swap reduced retained RSS but did not materially
reduce the peak. The implementation does not force process-wide collections during
refresh. `GOGC=50` reduced peak RSS by about 15% compared with the new default, with a
longer refresh. `GOMEMLIMIT=256MiB` barely changed this workload's peak. Reducing it to
`192MiB` brought peak RSS to 208.1 MiB and refresh time to 10.83 seconds. These limits
are workload measurements, not recommended budgets for every installation.

Set `GOMEMLIMIT` below the process's available memory budget, leaving room for the
executable and memory outside Go. It is a soft runtime limit, not an RSS cap. Lower
limits can spend more CPU on garbage collection; limits below the live working set
cannot make an oversized list fit. See [memory guidance](additional_information.md#memory-on-small-systems).

## Pruning decision

A separate per-length-bucket experiment compared keeping covered children with pruning
them after sorting. All four individual inputs already omit redundant children, so
pruning saved no storage and increased construction time. The combined input deduplicates
entries across the four files. The generated input has 100,000 parents and four children
per parent.

| Input | Kept entries | Pruned entries | Kept heap, MiB | Pruned heap, MiB | Keep build, ms | Prune build, ms |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| Four-list union | 2,052,964 | 2,039,330 | 32.78 | 32.40 | 1472 | 2033 |
| Generated overlapping names | 500,000 | 100,000 | 7.79 | 1.33 | 253 | 297 |

The implementation removes duplicate names but retains covered children, except when
an entire TLD is blocked. Searching parents first preserves the reported rule. This
avoids a pruning pass that saved only 0.38 MiB on the combined real inputs. Lists with
many redundant children can use more memory than the trie; pruning would help that
workload.

The unpartitioned reversed-domain prototype used 30.30 MiB on full TIF. Plain domain
buckets used 28.53 MiB but made unknown-TLD misses take about 600 ns. Partitioning by TLD
and filtering misses avoids that regression while using less heap than either layout.

## Reproduce

Use the same downloaded file for each revision. Copy
`cache/stringcache/wildcard_cache_benchmark_test.go` into the baseline checkout too.
Run on the target architecture:

```sh
BLOCKY_WILDCARD_LIST="$LIST_FILE" go test ./cache/stringcache \
  -run '^$' -bench '^BenchmarkLargeList$' -benchmem -benchtime=1s -count=3

BLOCKY_WILDCARD_LIST="$LIST_FILE" go test ./cache/stringcache \
  -run TestCache -ginkgo.focus='agrees with the trie on a supplied'

go test ./cache/stringcache ./lists -bench . -benchmem
GOOS=linux GOARCH=arm64 CGO_ENABLED=0 go build -trimpath -ldflags='-s -w' -o blocky .
```

The differential comparison covers returned rules as well as match results, with
uppercase input, root dots, empty labels, punycode and parent/child entries. It checks
both raw cache queries and resolver-normalized queries. The generated cases also reverse
insertion order. Adblock `||domain^` parsing remains exact-match behavior.

Input SHA-256 hashes:

```text
90b9c6b515c0b4953f02af17331e902f869470654e5d9e5033380f8c5fcaf841  oisd.txt
89750edfa219c266d697db09f34bcf997bc1f5fcb4ac46a85029ad9bd9e5e5af  tif.medium.txt
978419eb8bd3c2f6fa0fd14103649f1825461e237c0cf0a2a900947056ca4f49  tif.txt
4bb0e68d4d6e129bc98c984a01cc08674945e4e9081750d0c0d8440069f26f81  ultimate.txt
```
