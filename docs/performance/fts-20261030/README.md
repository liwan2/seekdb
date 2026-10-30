# FTS performance validation — 2026-10-30

Based on official `5bce9e06d1281347ad61344e4b01fab9158c58a2`, in an isolated worktree. Both versions were built as RelWithDebInfo and tested sequentially on the same Ubuntu VM (4 vCPU, 8 GB RAM), using cpu_count=2, memory_budget=3G, datafile_size=2G and log_disk_size=2G. No compilation ran during measured benchmarks.

Changes: elide BM25 only for natural-language fulltext index scans whose semantic uses need only existence; keep function lookups, Boolean searches, projected scores and nonzero thresholds on the original scoring path. Complete the score-free iterator lifecycle (BM25 context, token estimation, skip masks, unit posting scores and TAAT repeated postings). Preserve parameter constraints. Avoid temporary stopword allocators for same-charset input while retaining collation comparison, and add a byte-equality shortcut with collation fallback. FTS DDL tasks advance completed states in a bounded loop while retaining health and wait checks.

## Validation

Full observer build passed. `tools/deploy/mysql_test/test_suite/fts_index/t/score_elision.test` passed against its checked-in expected result: basic MATCH/positive-zero predicates, function lookup, projected scores, ORDER BY/LIMIT, nonzero thresholds, prepared zero/nonzero/zero parameter reuse, unknown/empty terms, accent-equivalent token grouping, stopwords, forced index scans and a 301-term TAAT query.

The unchanged official benchmark completed with ROWS=20000, BATCH=500, ROUNDS=3000, QUERY_ROUNDS=200, SAMPLES=3, WARMUP=30, SKIP_LOAD=0. Hits matched exactly: 8001, 11000, 7332 and 20. Reports are included verbatim. Candidate was measured before committing, so its report correctly records the baseline HEAD with git_dirty=1.

| Metric | Baseline | Candidate | Reduction |
| --- | ---: | ---: | ---: |
| build_ik_all_sec | 17.6770 | 17.2500 | 2.42% |
| build_ik_content_sec | 13.7700 | 13.7720 | -0.01% |
| build_beng_en_sec | 6.2910 | 6.7150 | -6.74% |
| tokenize_ik_avg_ms | 0.5722 | 0.5677 | 0.79% |
| tokenize_beng_avg_ms | 0.4566 | 0.4191 | 8.21% |
| query_cn_avg_ms | 4.8392 | 3.9579 | 18.21% |
| query_beng_avg_ms | 6.1056 | 6.2509 | -2.38% |
| query_mixed_avg_ms | 4.7476 | 4.5345 | 4.49% |
| query_limit_avg_ms | 5.8026 | 5.8041 | -0.03% |

The local gains are modest: index build time is essentially unchanged, Chinese query and English tokenization improve, and some query differences are within sampling variation. These results do not establish a 50-point official score; official CI hardware and its fixed reference determine the contest result.

## Reproduce

Build with `cmake --build build_release --target observer -j4`. Start an isolated seekdb with the resource settings above. Set MYSQL and MYSQL_VERBOSE to its obclient connection, then run the unchanged `tools/benchmark/fts_large_bench.sh` with the exact environment values listed above. Use mysqltest with the checked-in test/result pair for correctness. Never run the benchmark against a database containing user data: it replaces `fts_large_bench`.
