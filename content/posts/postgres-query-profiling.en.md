---
title: "Where to start when a PostgreSQL query goes slow"
date: 2026-08-22
tags: ["postgresql", "devops"]
summary: "EXPLAIN ANALYZE is only the start. A order of operations that saves time when debugging a specific slow query in production."
---

When a "this query got slow" complaint lands, here's the usual order of
operations.

## 1. EXPLAIN (ANALYZE, BUFFERS)

Without `BUFFERS` the plan is nearly useless for real analysis — buffers are
what show whether the query is reading from cache (`shared hit`) or from
disk (`read`). A sudden jump in `read` is almost always the first suspect.

## 2. Look at the estimate-vs-actual gap, not total time

A plan row where the planner expected, say, 100 rows but actually saw
100,000 pass through it is usually stale or insufficient statistics.
Running `ANALYZE` on the table (or revisiting `default_statistics_target`
for specific columns) often fixes it faster than rewriting the query itself.

## 3. pg_stat_statements — for a system-wide view, not a single query

If the problem isn't one query but "the database in general feels slow", a
targeted `EXPLAIN` won't help — you need `pg_stat_statements`, sorted by
`total_exec_time` or `mean_exec_time` depending on whether you're hunting
a rare-but-heavy query or a frequent-but-individually-unremarkable one.

## 4. Indexes — check real selectivity, not just presence

An index existing doesn't mean it's used, and being used doesn't mean it's
efficient. A common situation: a low-selectivity index (a boolean column,
say) that the planner correctly ignores — a Seq Scan can genuinely be
faster in that case.

## 5. Locks are their own category of problem

"Slow query" sometimes means not slow execution but long lock waits.
`pg_locks` combined with `pg_stat_activity` shows who's blocking whom —
and in setups with replication or backups running, this is the first thing
worth checking when a sudden degradation shows up with no code changes
behind it.

None of these steps replace the others — the actual cause is usually at the
intersection of two or three of them, not isolated to just one.
