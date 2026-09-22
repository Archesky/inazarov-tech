---
title: "Running Cassandra in production: what actually matters once the honeymoon ends"
date: 2026-01-14
tags: ["cassandra", "databases", "devops"]
summary: "Cassandra is forgiving during a proof of concept and unforgiving under real production load. A practical rundown of the decisions that determine which one you get."
---

Cassandra's pitch — linear scalability, no single point of failure, tunable
consistency — holds up well in a proof of concept. What determines whether
it holds up in production is a set of decisions that are cheap to get right
early and expensive to fix later, because several of them touch data
layout and are effectively irreversible without a full rebuild.

## Data modeling is the decision that can't be undone cheaply

Cassandra's golden rule — model around your queries, not your entities —
gets repeated often enough that it's easy to nod along without internalizing
what it actually costs to violate. A table designed around "what does the
data look like" instead of "what will I query" tends to work fine at demo
scale and then requires either a full-table migration or an application-
level workaround once real query patterns show up. Partition key choice
specifically is close to permanent: changing it means rewriting the data
into a new table with a new key, online, under load, which is a project in
its own right, not a config change.

The failure mode that shows up most often in practice is a partition key
that seemed reasonably distributed in testing but concentrates badly on
real traffic — a "most active customer" or "most popular item" pattern
that turns one partition into a hot spot no amount of cluster-wide capacity
helps with, because the problem is on one set of replicas, not the cluster
as a whole.

## Compaction strategy is a workload decision, not a default

The default `SizeTieredCompactionStrategy` (STCS) is a reasonable choice
for write-heavy, append-mostly workloads, but it's a poor fit for tables
with frequent updates or deletes to the same rows — `LeveledCompactionStrategy`
(LCS) handles that pattern with far less read amplification, at the cost of
more compaction I/O overall. `TimeWindowCompactionStrategy` (TWCS) exists
specifically for time-series data with a natural TTL or expiry pattern, and
using STCS or LCS for that kind of workload instead usually shows up later
as compaction that never quite keeps up.

Getting this wrong doesn't show up immediately — it shows up as read
latency that slowly degrades over weeks as SSTables accumulate in a shape
the chosen strategy isn't good at merging, which makes it a genuinely easy
thing to miss during initial rollout and only notice once it's already
expensive to unwind.

## Consistency level is a per-query decision, not a cluster setting

`QUORUM` reads and writes are the common default, but treating consistency
level as a single cluster-wide choice instead of a per-query one leaves
real trade-off room on the table. Read-heavy, latency-sensitive lookups
where slightly stale data is fine are often better served by `ONE` on the
read side, provided writes are still done at a level that keeps the risk of
staleness acceptable for that specific access pattern. The mistake worth
avoiding in the other direction is using `ONE` broadly to shave off latency
and then being surprised by inconsistent reads during a node failure or
network partition — the trade-off `ONE` makes is not free, it's just
deferred to the moment something goes wrong.

## Repairs are not optional maintenance

Anti-entropy repair (`nodetool repair`) is how Cassandra reconciles data
that diverged across replicas — due to a node being briefly unavailable, a
dropped write, or any of the ordinary things that happen in a distributed
system. Skipping regular repairs doesn't cause an immediate incident; it
causes an accumulating gap between what different replicas believe is true,
which eventually surfaces as inconsistent reads that look like application
bugs until someone thinks to check repair history. `gc_grace_seconds`
matters directly here too — it needs to be longer than the interval between
repairs, or tombstones can be garbage collected before repair has a chance
to propagate the deletion everywhere, which resurrects deleted data. This
is a genuinely unpleasant class of bug to debug from the symptom backward.

## Node density and vnodes

Fewer, denser nodes reduce operational overhead but increase the blast
radius and the time a single node failure takes to recover from — more
data per node means longer streaming during bootstrap or repair. The
default vnode count (256 in older versions, reduced defaults in newer
ones) trades even token distribution for larger, slower repair and
bootstrap operations; a lower vnode count with careful initial token
allocation is a legitimate choice for larger clusters where repair/
bootstrap time is a real operational cost, not just a theoretical one.

## What this adds up to

None of these are exotic concerns — they're the standard list any
experienced Cassandra operator would give. What makes them worth writing
down explicitly is that every one of them is cheap to get right at design
time and expensive (full rebuild, painful migration, or a slow-burning
consistency bug) to fix once real data and real traffic are already on the
cluster. Cassandra's operational model rewards getting the boring decisions
right early far more than it rewards reacting well to problems later.
