# SQLStorm StackOverflow — energy per query (laptop2 run)

Energy and time of **11431 SQL queries** from SQLStorm's StackOverflow set, measured on one laptop
(Dell Latitude 7490, Intel i5-8350U, clock fixed at 2.5 GHz) with PostgreSQL 18.6 on the
StackOverflow (dba.stackexchange) 1 GB database ("laptop2" is the measuring machine's name in
`run_info.csv`). The per-query CSVs are keyed by `query`.

**Query id.** `query` is the number in the SQLStorm file name: query 15407 is
`stackoverflow/queries/15407.sql` in SQLStorm v1.0 (commit `b3bb0b96`).
Each query was executed as `EXPLAIN (ANALYZE, TIMING OFF, COSTS ON, SUMMARY ON, BUFFERS) <query>`:
it runs to completion on the server, but its rows are not sent to the client.

## Files

| file | rows | content |
|---|---|---|
| `query_energy.csv` | 11431 | every fully successful query and its energy per execution — **the labels** |
| `excluded_queries.csv` | 363 | queries of the run that are not in `query_energy.csv`, and why |
| `query_execution_stats.csv` | 11386 | PostgreSQL execution statistics of each successful query |
| `duplicates_and_similar.csv` | 3921 | duplicates, near-duplicates and same-output queries |
| `duplicate_and_similar_groups.csv` | 452 | groups of queries with identical output, and the SQL variants in each |
| `empty_output_queries.csv` | 944 | queries whose result is empty |
| `run_info.csv` | 50 | run parameters, machine, OS, PostgreSQL, dataset provenance |
| `database_tables.csv` | 26 | tables and indexes of the database, row counts and sizes |

## How the energy was measured

Every query in the set (11794) was run once, in a fixed random order, over 81.81 h
(2026-09-30T09:43:47Z to 2026-10-03T19:32:30Z). For each query:

1. thermal gate: wait until the CPU package is between 40 and 60 °C;
2. two single-copy warm-ups (the first fills the cache);
3. a measured batch of **1** copy and a measured batch of **16** back-to-back copies, each recording
   wall time and RAPL energy (package, cores, DRAM).

The energy of one execution is the **slope** `energy_j = (E16 − E1) / 15`. It removes what each
batch pays once (starting the client, connecting, batch edges), leaving what one more execution of the
query costs. Time per execution `time_s` is the same slope on wall time. No idle power was subtracted:
`energy_j` is the whole package energy while the query runs. Statement timeout 15s
per copy; parallel query at the planner default.

**Fully successful** means none of the four batches failed. 363 queries are excluded:
361 warm-up timed out at 15 s (measured batches skipped); 2 measured N=16 batch failed (timeout).

**Precision.** 2727 queries are exact copies of other queries (452 sets) and were measured
independently, which gives the repeat noise of one slope on this machine:
`energy_j_se = sqrt((0.8 mJ)^2 + (0.89% x energy_j)^2)`, about
0.9% for most queries. It was fitted on slopes up to 12 J and is
extrapolated above. The noise has a heavy tail: 0.6% of the copies lie more than 3
standard errors from their set's mean (the worst set spreads by 10%).

## `query_energy.csv`

| column | unit | meaning |
|---|---|---|
| `query` | — | SQLStorm query number (file name) |
| `energy_j` | J | **package energy per execution**, `(e16_j − e1_j) / 15` |
| `energy_j_se` | J | estimated standard error of `energy_j` (repeat noise model above) |
| `time_s` | s | wall time per execution, `(t16_s − t1_s) / 15` |
| `energy_core_j` | J | CPU-core part of `energy_j` (RAPL core domain), same slope |
| `energy_dram_j` | J | DRAM energy per execution (separate RAPL domain, not included in `energy_j`) |
| `fixed_overhead_j` | J | per-batch fixed cost, `e1_j − energy_j` |
| `e1_j`, `e16_j` | J | measured package energy of the 1-copy and the 16-copy batch |
| `t1_s`, `t16_s` | s | measured wall time of the two batches |
| `power_w` | W | mean package power during the 16-copy batch |
| `temp_start_c` | °C | package temperature at the start of the 16-copy batch |
| `mhz_mean` | MHz | mean clock over all 8 logical CPUs during the 16-copy batch (idle CPUs included, so below the 2.5 GHz ceiling) |
| `duplicate_queries` | — | the other queries whose SQL is identical to this one (`;`-separated; empty if none) |
| `empty_output` | — | the query returns no rows |
| `wallclock_dependent` | — | filters on `NOW()` / `CURRENT_DATE`, so its result and cost depend on the run date |
| `sql_hash` | — | md5 of the normalised SQL text (see below), to check a query file is the same one |

## `query_execution_stats.csv`

From PostgreSQL's `EXPLAIN ANALYZE` output of the measured single copy: planning and execution time
(ms), number of plan and scan nodes, rows returned (`rows_out`), rows processed by all nodes,
the planner's row estimate, bytes processed, rows removed by filters, parallel workers launched,
relations touched (as PostgreSQL names them, so CTE and alias names appear too), shared-buffer hits and reads, temp blocks read and written (8 kB blocks);
`server_execution_ms_median16` is the median over the 16-copy batch. The harness logged no per-copy
samples for 45 of the successful queries, so they are absent from this file. These are measured during
execution — useful as targets or for analysis, but not available before a query runs.

## `duplicates_and_similar.csv`

How queries relate to each other in form (SQL text) and in output, for every query that has at least
one such relation. Nothing was removed or merged because of it: all queries keep their own
measurement in `query_energy.csv`. SQL is normalised before comparing: the SQLStorm header comment and
`EXPLAIN (…)` wrapper are removed, comments removed, whitespace collapsed, lower-cased.

| column | meaning |
|---|---|
| `in_query_energy` | the query is in `query_energy.csv` |
| `sql_hash` | md5 of the normalised SQL |
| **form: identical** | |
| `duplicate_queries` | the other queries whose SQL is identical to this one (`;`-separated) |
| `duplicate_count` | how many there are |
| **form: similar** | |
| `similar_queries` | all queries whose SQL differs from this one but has token 3-gram Jaccard similarity ≥ 0.8 (`;`-separated) |
| `similar_count` | how many there are |
| `most_similar_query`, `most_similar_jaccard` | the closest of them and its similarity |

All three use the same form: a list of the other queries in the relation and its length. The
relations are symmetric — if 15027 lists 15043, then 15043 lists 15027.
| **output** | |
| `same_output_queries` | the other queries whose result file is byte-identical to this one's (header, every row, row count; `;`-separated) |
| `same_output_count` | how many there are |

11794 queries contain 9519 distinct SQL texts; 3834 queries have
at least one similar query, and 3050 share their exact output with another query. Similar
queries often differ in a single clause (a LIMIT, a filter, a column), and their energies can differ
widely; copies of identical SQL were measured independently.

## `duplicate_and_similar_groups.csv`

The same information grouped: one row per group of queries whose result files are byte-identical
(452 groups, 3050 queries). Every set of identical-SQL queries falls
inside one of these groups, since copies of a query always returned the same output.

| column | meaning |
|---|---|
| `group` | group id, G1 = largest |
| `n_queries` | queries in the group |
| `n_sql` | distinct SQL texts among them (normalised as above) |
| `kind` | `same SQL (copies)`, `different SQL`, or `copies + different SQL` |
| `n_in_query_energy` | how many of the group's queries are in `query_energy.csv` |
| `rows` | rows in the shared result (empty when capped) |
| `capped` | the result had 1000 rows or more (the output run kept the first 1000) |
| `ncol` | columns in the result |
| `sim_min` | lowest token similarity between two of the group's distinct SQL texts (difflib ratio, 1 = same text) |
| `queries` | the group's queries: `=` joins identical SQL, `|` separates different SQL texts, e.g. `15027 = 15043 | 16207` |
| `header` | the result's column names |

## `empty_output_queries.csv`

The 944 queries whose result is empty, from a separate output run (`output_runner`, 60 s
timeout, on a second, identical Latitude 7490 with the same database — every table the same size — and
identical query files).
`likely_reason` is the first matching SQL feature (a heuristic). `wallclock_dependent` queries filter
on `NOW()`/`CURRENT_DATE`; the data ends in 2024, so their date window is empty when run now, they
skip most of their work, and their energy is lower than the SQL suggests. `date_window_days` is the
shortest `INTERVAL` in the query.

## Caveats

* One machine, one run: each query was measured once (one 1-copy and one 16-copy batch).
* Energy is measured while running `EXPLAIN ANALYZE`, without sending rows to the client.
* `wallclock_dependent` queries (230 in `query_energy.csv`) would cost
  differently on another date.
* Excluded queries are not missing at random: they are the slowest (they hit the 15 s timeout).
