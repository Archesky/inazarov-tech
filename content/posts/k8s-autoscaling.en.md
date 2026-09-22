---
title: "HPA, VPA and KEDA: when to use which Kubernetes autoscaler"
date: 2026-09-10
tags: ["kubernetes", "devops"]
summary: "The three autoscaling mechanisms in Kubernetes solve different problems — mixing them up is usually what leads to over-provisioned resources or OOM kills."
---

"Autoscaling in Kubernetes" often gets treated as one thing, but in practice
it's three fairly different mechanisms that solve different problems.

## HPA — scaling the number of pods

Horizontal Pod Autoscaler reacts to metrics (usually CPU/memory, but custom
metrics via Prometheus Adapter work too) and changes the **number of
replicas**. It's the simplest and most predictable tool — a good fit for
stateless services that scale horizontally well.

A common mistake is setting up HPA without sane `requests` — then scaling
either kicks in too late or too early.

## VPA — scaling the pod's resources

Vertical Pod Autoscaler doesn't change replica count, it changes the pod's
own `requests`/`limits`. Useful where horizontal scaling doesn't make sense
(stateful workloads, for example), or simply to have requests/limits picked
automatically instead of guessing them by hand.

One important caveat: VPA in `Auto` mode recreates the pod when it changes
resources — this isn't seamless, and it needs to be accounted for.

## KEDA — scaling on external events

KEDA extends HPA with metrics from external systems — queues (RabbitMQ,
Kafka), databases, cloud services, and so on. It's especially useful for
workers that process a queue and can otherwise scale down to zero — something
HPA can't do out of the box.

## Bottom line

In practice, the combination that actually works is HPA for stateless
services under user traffic, KEDA for workers reacting to queues, and VPA
(more often in `Recommendation` mode than `Auto`) as a tool for picking
sane requests/limits rather than as a standalone autoscaler. Combined with
policies like Kyverno that standardize requests/limits at the namespace
level, this noticeably cuts down both over-provisioning and OOM kills in
production.
