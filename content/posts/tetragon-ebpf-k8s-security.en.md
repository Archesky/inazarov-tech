---
title: "Tetragon and eBPF-based security: runtime visibility for Kubernetes that doesn't lie"
date: 2026-02-10
tags: ["kubernetes", "security", "ebpf"]
summary: "A deep look at what eBPF actually changes about runtime security in Kubernetes, how Tetragon fits into that picture, and where it still needs help from the rest of the stack."
---

Most Kubernetes security tooling operates on one of two layers: static
analysis (scan the image, scan the manifest, scan the Helm chart before
anything runs) or log-based detection (parse audit logs, parse application
logs, correlate after the fact). Both are useful and both share the same
blind spot — they don't see what a process inside a running container
actually *does* at the kernel level. That's the gap eBPF-based tools like
Tetragon (from the Cilium project, built on the same eBPF foundation as
Cilium's networking) are built to close.

## Why kernel-level visibility is different in kind, not just degree

A container is, from the kernel's point of view, just a set of processes
with namespace and cgroup isolation. Every meaningful security-relevant
action — opening a file, executing a binary, making a syscall, establishing
a network connection — passes through the kernel. Userspace agents that
watch logs or poll `/proc` are working from a summary someone else wrote;
an eBPF program attached to the right kernel hook sees the actual event as
it happens, with no ability for anything in userspace (including a
compromised application or even a compromised container runtime, up to a
point) to hide it after the fact.

This matters concretely for a specific class of attack: a process that
executes a malicious binary and exits within milliseconds, before any
polling-based agent gets a chance to observe it. Log-based tools miss this
by construction. A kernel-level tracepoint doesn't.

## What Tetragon actually gives you

Tetragon exposes a `TracingPolicy` CRD that lets you describe, declaratively,
which kernel events to watch and what to do about them. In practice this
covers three overlapping use cases:

- **Process execution visibility** — every `execve` in the cluster, enriched
  with Kubernetes metadata (pod, namespace, labels, container image) rather
  than a bare PID, which is what makes this actually usable instead of
  academically interesting.
- **File access monitoring** — watching specific paths (`/etc/shadow`,
  service account token mounts, secrets volumes) for reads or writes that
  shouldn't be happening from a given workload.
- **Network-level enforcement** — because Tetragon shares its eBPF
  foundation with Cilium, it can correlate process-level events with
  network connections, answering "which process made this connection" in
  a way that's much harder to spoof than userspace network monitoring.

The part that separates it from a pure observability tool: TracingPolicies
support **enforcement**, not just logging. You can kill a process or block
a syscall the moment a policy matches, in-kernel, before the action
completes — which is a meaningfully different guarantee than "we logged it
and paged someone."

## A concrete example: catching in-memory execution

A common post-exploitation technique is writing a payload to a
memory-backed filesystem (`/dev/shm`, `memfd_create`) and executing it
directly, specifically to avoid leaving a file on disk for a
signature-based scanner to find later. This is exactly the kind of thing
static and log-based tooling structurally can't catch — there's no file to
scan, because by design there's nothing left after execution.

A Tetragon policy watching `execve` with a filter on the binary's path
prefix (or lack of a backing file at all, for `memfd_create`-based
execution) catches this at the moment of execution, kernel-side, regardless
of whether anything was ever written to persistent storage.

## Where it still needs the rest of the stack

None of this replaces image scanning, admission control, or network
policy — it sits alongside them, closing a specific gap. A few things worth
being explicit about:

- **Policy authoring has a real learning curve.** Writing effective
  TracingPolicies requires understanding which kernel hooks matter for a
  given threat model — this isn't a checkbox feature, it needs someone who
  understands both the workload and the kernel-level primitives.
- **Overhead is real, though usually small.** eBPF programs run in the
  kernel's fast path; well-written ones add microseconds, not
  milliseconds — but "well-written" is doing some work in that sentence,
  and dense policies on high-throughput nodes deserve actual benchmarking,
  not just trust.
- **It doesn't replace admission control.** Tetragon tells you what
  happened at runtime; it doesn't stop a bad pod spec from being scheduled
  in the first place — that's still OPA/Gatekeeper/Kyverno's job.
- **Alert fatigue applies here too.** A `TracingPolicy` that's too broad
  generates the same kind of noise any other overly sensitive alerting
  does — the general discipline of tuning thresholds and pruning rules that
  don't lead to action applies just as much to kernel-level events as to
  application metrics.

## The honest takeaway

eBPF-based tools like Tetragon don't make Kubernetes security "solved" —
they close a specific, previously-unaddressable gap: knowing with certainty
what actually executed and what actually touched the filesystem or network,
at the one layer that can't lie about it. That's a meaningfully different
guarantee than log correlation, and worth the investment for workloads
where "we think we would have noticed" isn't good enough — but it's one
layer in a stack, not a replacement for the rest of it.
