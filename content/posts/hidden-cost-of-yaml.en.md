---
title: "The hidden cost of YAML"
date: 2026-03-05
tags: ["devops", "kubernetes", "infrastructure"]
summary: "YAML didn't create the complexity in modern infrastructure tooling, but it hides just enough of it to make things worse."
---

Every Kubernetes-adjacent tool ends up configured in YAML, and at some
point every team that's worked with it long enough arrives at the same
complaint. The interesting part isn't that YAML is annoying to write — it's
specifically how it fails.

## Whitespace-as-syntax turns typos into silent behavior changes

A misplaced indent in most languages is a parse error — loud, immediate,
caught before anything runs. In YAML, an extra or missing space often
produces a *different valid document* instead of an invalid one. A list
item that silently becomes a sibling instead of a child doesn't throw an
error; it just changes what gets applied, and the failure shows up
downstream as "why didn't this config take effect" rather than as a syntax
error at the point of the actual mistake.

## The type coercion trap

Unquoted `no`, `yes`, `on`, `off`, and certain numeric-looking strings get
silently parsed as booleans or numbers by YAML 1.1 parsers (the version
most tools still use) — a country code, a version string, a Slack channel
name starting with a digit, all quietly become something other than the
string that was typed. This class of bug is specific to YAML's type
inference and doesn't exist in formats that require explicit typing.

## Deep nesting hides the actual diff

A one-line change six levels deep in a values file for a Helm chart is
easy to make and easy to miss in review — the actual line that changed is
surrounded by so much structural noise that reviewers reasonably skim past
it. This isn't a YAML-specific problem exactly, but YAML's preferred style
(deep nesting, one key per line, minimal inline structure) makes it worse
than formats that encourage flatter, more explicit structure.

## What actually helps

None of this means abandoning YAML — the ecosystem is what it is. What
helps in practice: schema validation in CI (`kubeval`, `kubeconform`, or
tool-specific schemas) that catches structural mistakes before they reach
a cluster; linting that flags unquoted ambiguous scalars; and, where the
tool allows it, generating YAML from a real language (Jsonnet, CUE,
Dhall, or even plain templating with strict mode) rather than hand-editing
deeply nested files directly. None of these make YAML itself less
YAML-shaped — they just move the point where a mistake gets caught from
"in production" to "in CI," which is really the only property that
matters.
