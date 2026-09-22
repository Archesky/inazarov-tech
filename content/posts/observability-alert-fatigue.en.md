---
title: "Alert fatigue: cutting the noise without losing real incidents"
date: 2026-06-03
tags: ["monitoring", "sre", "devops"]
summary: "The problem is almost never too few alerts — it's too many. A look at what actually helps cut false positives."
---

A familiar situation: monitoring is set up, there are plenty of alerts, but
at some point on-call engineers stop reacting to them — because most turn
out to be false positives or not actually important. After that it doesn't
much matter how well Prometheus/VictoriaMetrics or Grafana are configured —
if an alert isn't trusted, it doesn't work.

## Separate symptom-based alerts from cause-based ones

An alert like "CPU above 80%" is a cause, not a symptom. High CPU usage on
its own can be completely normal for a service and say nothing about actual
user impact. It's more useful to alert on symptoms (latency, error rate,
availability) and dig into the cause inside the incident — not the other
way around.

## Every alert needs an unambiguous runbook

If it's unclear what to do when an alert fires, it either gets ignored or
gets handled by guesswork. A runbook doesn't need to be long, but it should
answer "what to check first" — not just restate the alert's name.

## Revisiting thresholds is a process, not a one-time setup

Thresholds picked "by eye" during initial monitoring setup are almost
always either too sensitive or not sensitive enough. A useful practice is
to periodically check how many times an alert fired over the last month
and how many of those actually required action. If that ratio is badly
skewed, the threshold or condition is worth revisiting rather than left
as-is "just in case."

## Deduplication and grouping at the Alertmanager level

The same problem across several nodes/pods shouldn't arrive as ten separate
notifications — grouping by shared labels (service, environment, cause)
sharply reduces the subjective sense of noise, even when the actual number
of problems hasn't changed.

Overall this isn't about "adding more rules" — it's the opposite process:
regularly auditing what's already configured and ruthlessly removing alerts
that don't lead to action.
