---
title: "Terraform across multiple clouds: what's actually worth reusing"
date: 2026-07-15
tags: ["terraform", "devops"]
summary: "A full abstraction layer over AWS/Yandex Cloud/VK Cloud almost never pays off — but a few things are still worth sharing."
---

Working with cross-regional infrastructure on several providers at once
(AWS, Yandex Cloud, VK Cloud) quickly raises the question: write one shared
Terraform layer on top of all three, or keep separate modules per provider.

## Full abstraction usually doesn't pay off

The temptation to write one `network`, `compute`, `database` module with the
provider as an input parameter is understandable, but in practice providers
don't just differ in resource names — the whole model diverges. Somewhere
managed Kubernetes is one resource, somewhere it's a dozen linked ones;
somewhere there's a managed queue service, somewhere you deploy it yourself.
The abstraction either collapses to the lowest common denominator (losing
each cloud's convenient features) or grows enough conditional logic that
it's harder to read than three separate modules would have been.

## What's actually worth unifying

- **State and backend structure** — a consistent approach to splitting
  state by environment/region, even if the resources themselves are
  described separately per cloud.
- **Naming conventions and tags** — this isn't really about Terraform code,
  it's about discipline, but it's exactly what tends to drift first once
  you're working with several providers at once.
- **The CI pipeline** (`plan` on MR, `apply` only from main, mandatory plan
  review) — the process around Terraform matters as much as the code
  itself, and it's entirely reusable across clouds.

## What isn't

Resource modules are better kept provider-specific. Trying to build "one
database module for every cloud" usually ends with, six months later, one
provider branch that's actually used and the rest turning into unmaintained
dead code.

The takeaway is simple: unify the process and discipline around Terraform —
yes; unify the actual resource modules between fundamentally different
clouds — almost never worth the cost.
