---
title: "Chaos engineering without a Netflix-sized budget"
date: 2026-04-02
tags: ["sre", "devops", "reliability"]
summary: "You don't need a dedicated chaos team to get value from deliberately breaking things — a few cheap, low-ceremony experiments go a long way."
---

Chaos engineering has a branding problem: it's associated with companies
running fleets of services at a scale where a dedicated team runs
automated fault-injection platforms in production, all the time. That
image quietly convinces a lot of smaller teams that the practice isn't for
them. In practice, the useful part of chaos engineering scales down fine —
it's the tooling and ceremony that don't.

## Start with questions, not tools

The actual starting point isn't "which chaos tool should we install" —
it's "what do we believe is true about our system's failure behavior, and
have we ever actually checked?" Things like: does the app really recover
gracefully when a database replica disappears? Does the retry logic in
service A actually back off, or does it hammer service B into a worse
outage during a partial failure? These questions can be answered with a
staging environment and thirty minutes, no platform required.

## The cheapest experiment: kill a pod on purpose

Before reaching for anything fancier, just deleting a pod by hand and
watching what happens is already a chaos experiment. Does the readiness
probe actually take it out of the load balancer before the connection
drops, or is there a window of failed requests? Does the replacement pod
come up fast enough, or does a dependent service time out waiting on it?
This costs nothing and often surfaces the same class of problem a much
more elaborate tool would.

## Game days beat always-on chaos for small teams

Continuous, automated fault injection in production is genuinely valuable —
at a scale and maturity level most teams aren't at yet. A scheduled "game
day" — a couple of hours, a specific scenario picked in advance (a node
drains, a dependency starts timing out, a region's connectivity degrades),
the whole on-call team watching and reacting — gets most of the learning
value with a fraction of the tooling investment and none of the risk of
running unattended fault injection against a system nobody's fully
confident in yet.

## Write down what you learn, not just that you ran the test

The actual output of a chaos experiment isn't "we ran it" — it's a short
list of things that didn't behave the way anyone assumed, plus an owner
and a rough timeline for each one. Without that, the exercise becomes
theater: everyone feels reassured that "resilience testing happens here"
without anything about the system actually changing.

## Bottom line

The value of chaos engineering isn't in the platform — it's in deliberately
checking assumptions about failure behavior before an actual incident does
it for you at 3am. That part is available to a team of any size, starting
today, with tools everyone already has.
