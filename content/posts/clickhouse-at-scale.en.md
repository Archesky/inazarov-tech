---
title: "ClickHouse at scale: what actually determines query performance"
date: 2025-11-05
tags: ["clickhouse", "databases", "devops"]
summary: "ClickHouse rewards a small number of upfront modeling decisions disproportionately — and punishes getting them wrong just as disproportionately, once data volume grows."
---

ClickHouse's performance reputation is well-earned, and it also creates a
specific trap: a schema and query pattern that performs brilliantly at a
few million rows can degrade badly at a few billion, for reasons that have
nothing to do with hardware and everything to do with a handful of
modeling decisions made early and rarely revisited.

## The primary key is a sort order, not an index in the traditional sense

This is the single most common source of confusion coming from a
traditional RDBMS background. ClickHouse's primary key defines the
physical sort order of data on disk — it's used to build a sparse index
(one entry per N rows, not one per row) that lets the engine skip whole
blocks of data that can't contain matching rows, not to do a row-level
lookup the way a B-tree index does. A query that filters on a column not
present as a reasonably-early component of the primary key gets essentially
no benefit from indexing at all and falls back to scanning proportionally
more of the table.

The practical consequence: primary key column order needs to match actual
query filter patterns, from lowest to highest cardinality of the columns
that actually appear in `WHERE` clauses — not entity structure, not
"logical" ordering, and specifically not high-cardinality columns first,
which defeats the sparse index's ability to skip blocks almost entirely.

## Partitioning is for data lifecycle, not query performance

`PARTITION BY` is frequently mistaken for a performance lever the way it
sometimes is in other systems — in ClickHouse its real job is enabling
efficient bulk operations on whole partitions: dropping old data (`DROP
PARTITION` instead of a slow row-by-row `DELETE`), and enabling `TTL`-based
expiry. Over-partitioning (a common mistake: partitioning by day when
monthly would do, or partitioning on a high-cardinality column) creates a
large number of small parts, and having too many parts is a real
operational problem in ClickHouse — merges struggle to keep up, and
`Too many parts` becomes an error you see in production, not a theoretical
one.

## Materialized views: real-time cost paid on every insert

A materialized view in ClickHouse isn't a periodically refreshed snapshot
the way it often is elsewhere — it's a trigger that runs on every insert
into the source table and writes results into the target table
immediately. This is exactly what makes them useful for real-time
aggregation, and exactly why an expensive materialized view definition
becomes a tax paid on every single insert into the source table, not an
occasional background cost. A materialized view that seemed free at low
insert volume can become the actual bottleneck on the write path once
ingest rate grows, and it's easy to miss because the cost is distributed
across every insert rather than showing up as one visible slow operation.

## Codec selection matters more than most default configurations assume

Column-level compression codecs are configurable per-column, and the
default `LZ4` is a reasonable general-purpose choice, but it leaves real
performance on the table for specific, common data shapes: `Delta` or
`DoubleDelta` codecs on monotonically increasing columns (timestamps,
auto-incrementing IDs) and `Gorilla` on slowly-changing float metrics can
meaningfully reduce both storage footprint and the I/O cost of scanning
those columns, because less data physically needs to be read off disk for
the same logical query. This is a case where the default isn't wrong, just
generic — and generic leaves real performance on the table specifically
for the time-series and metrics-style data ClickHouse is most often chosen
for in the first place.

## Distributed queries hide a coordination cost that's easy to underestimate

A query against a `Distributed` table fans out to every shard and merges
results on the initiating node — which means the initiating node's
resources (particularly memory, for large intermediate result sets) become
a real, easy-to-overlook bottleneck distinct from per-shard performance.
A query that runs fine when tested against a single shard directly can
behave very differently once it's actually issued as a distributed query
across the full cluster, purely because of coordination and merge overhead
that doesn't exist in the single-shard case.

## What this adds up to

None of these are ClickHouse being fragile — they're the direct, logical
consequence of a column-oriented, merge-tree-based engine that trades
flexibility for raw scan performance. The trap is that all of it performs
fine at small scale regardless of whether these decisions were made
correctly, which means the cost of getting them wrong is specifically
deferred to the point where data volume is large enough that a genuine
performance problem is also the most expensive and disruptive point to
fix it.
