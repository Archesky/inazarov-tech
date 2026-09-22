---
title: "GitOps: what it actually solves (and what it doesn't)"
date: 2026-05-11
tags: ["gitops", "kubernetes", "devops"]
summary: "GitOps gets sold as a silver bullet for deployment problems. It solves a narrower, more specific problem than the pitch usually suggests."
---

GitOps tends to get pitched as a general solution to "deployment problems" —
in practice it solves one specific problem very well, and people quietly
expect it to solve several others it was never meant to touch.

## What it actually solves: drift

The core promise of GitOps — a reconciliation loop (Argo CD, Flux) that
continuously compares the cluster's real state to what's declared in Git and
corrects any difference — solves configuration drift specifically. Someone
`kubectl edit`'d a deployment directly in prod at 2am during an incident and
forgot to update Git? The next reconciliation pass quietly reverts it. That
alone eliminates an entire category of "why is prod different from what's
in the repo" incidents that used to require someone to remember and
manually reconcile.

## What it doesn't solve: deployment strategy

Continuous reconciliation is orthogonal to *how* a new version rolls out.
Canary, blue/green, progressive delivery with automated rollback based on
metrics — none of that comes from "state lives in Git." Argo Rollouts or
Flagger bolt progressive delivery on top of a GitOps controller, but the
GitOps part itself is silent on rollout strategy. Teams that adopt GitOps
and expect safer deployments as a side effect are often surprised that a
bad rollout still needs the same guardrails it always did.

## What it doesn't solve: secrets

Git-native storage is exactly the wrong place for secrets, and every GitOps
setup ends up bolting on something else to handle them — Sealed Secrets,
External Secrets Operator, SOPS, Vault integration. This isn't a flaw in
GitOps so much as a reminder that "everything as a Git-tracked manifest" was
never a complete story to begin with; it's a partial pattern that still
needs the rest of the stack around it.

## What it doesn't solve: multi-cluster coordination

A reconciliation loop per cluster doesn't automatically give you ordered,
coordinated rollouts across clusters, or a single source of truth for
"what version is actually running where" across a fleet — that still needs
tooling on top (or a deliberate branching/promotion strategy across
environments), not something GitOps hands you for free.

## The honest framing

GitOps is a very good answer to "how do we make sure the cluster matches
what's declared, continuously, without anyone remembering to run
`kubectl apply`." It's not an answer to deployment safety, secret
management, or fleet-wide coordination — those need their own tools sitting
next to it, not instead of it.
