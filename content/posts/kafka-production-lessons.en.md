---
title: "Kafka in production: the mistakes that don't show up until later"
date: 2025-12-08
tags: ["kafka", "devops", "databases"]
summary: "Partitioning, replication and consumer group design decisions in Kafka that look fine at low volume and become expensive exactly when traffic grows."
---

Kafka clusters tend to fail quietly at first — a partitioning choice or a
replication setting that's technically fine at moderate volume slowly turns
into a real problem as throughput grows, usually well after the decision
that caused it is buried in history. A few of these come up often enough to
be worth writing down.

## Partition count is a decision you'll regret changing later

Partition count sets the ceiling on consumer parallelism for a topic — you
can't have more active consumers in a group than partitions, full stop.
Under-provisioning shows up as consumers sitting idle while others fall
behind. The less obvious trap is over-provisioning: more partitions means
more open file handles and more replication traffic per broker, and
increasing partition count on an existing topic breaks the ordering
guarantee for any key whose messages were previously landing on one
partition, because the partition assignment for existing keys changes.
That makes "just add more partitions later" a much bigger decision than it
sounds, for any topic where per-key ordering actually matters to consumers.

## Replication factor and `min.insync.replicas` are a pair, not independent knobs

`replication.factor=3` is the common default, and on its own it says
nothing about durability without `min.insync.replicas` set alongside it.
With `acks=all` and `min.insync.replicas=1`, a producer gets an
acknowledgment once a single replica (which may just be the leader) has the
data — losing that one broker before replication catches up loses the
message, despite `acks=all` giving the impression of a strong guarantee.
`min.insync.replicas=2` with `replication.factor=3` is what actually buys
tolerance for one broker failure without data loss — the combination is
what matters, not either setting read in isolation.

## Consumer lag is the metric that actually tells you something

CPU and memory on brokers can look completely healthy while consumers
silently fall further and further behind — a slow downstream dependency, an
expensive per-message processing step, or a rebalance storm can all cause
growing lag without moving broker-side resource metrics at all. Monitoring
consumer group lag directly, per partition, is the metric that catches
problems the infrastructure-level dashboards miss entirely — by the time
broker CPU looks unhealthy, lag-driven problems are usually already well
past the point where they were easy to catch.

## Rebalance storms: the failure mode nobody sizes for upfront

The eager rebalancing protocol (still the default in a lot of deployments)
stops all consumers in a group during any rebalance — a single consumer
restart, deploy, or transient failure pauses the entire group's processing,
not just the affected consumer's partitions. At meaningful consumer group
size, deploys or rolling restarts done without care about rebalance
behavior can cause processing pauses that are longer and more disruptive
than the actual failure that triggered them. Cooperative sticky rebalancing
(available since Kafka 2.4) narrows the disruption to only the partitions
actually being reassigned instead of stopping the whole group, and is worth
the migration for any consumer group large enough that a full-group pause
is actually felt.

## Consumer group ID reuse is an easy way to lose data silently

Reusing a consumer group ID for a functionally different consumer (a new
version of a service with different processing logic, say) means it
inherits the previous consumer's committed offsets and picks up wherever
the old one left off — which is either exactly what you want or a silent
skip of everything that arrived between the old consumer stopping and the
new one starting, depending entirely on whether that was intentional. This
is rarely a documented decision; it's usually an accident of reusing a
convenient, already-configured group ID.

## Under-provisioned retention hides in tests and bites in an incident

Retention configured for steady-state volume looks fine until a downstream
consumer is down for a few hours during an incident and comes back to find
the messages it needed to replay have already been deleted. Sizing
retention for "how long could a consumer plausibly be down and still need
to catch up," not for steady-state disk usage, is the difference between
an incident that's inconvenient and one that loses data permanently.

## The pattern across all of these

None of these mistakes cause an immediate outage — that's exactly what
makes them dangerous. They're all decisions that look completely
reasonable in a load test at moderate volume and become expensive
specifically once real production traffic, real incidents, and real
multi-broker failures start happening — which is, unhelpfully, also the
point at which they're hardest and most disruptive to fix.
