# Security Policy

## Supported versions

**None yet.** stomattice is pre-alpha: there is no released version, no
published artifact, and nothing to deploy. Only the tip of the default branch is
looked at.

This section will list supported versions once there is a release to support.

## Reporting a vulnerability

Please **do not** open a public issue for security-sensitive findings.

Report vulnerabilities privately to the maintainers via GitHub's
[private vulnerability reporting](https://docs.github.com/code-security/security-advisories/guidance-on-reporting-and-writing-information-about-vulnerabilities/privately-reporting-a-security-vulnerability)
feature on this repository, or to the maintainer contact published by the
[Infinite Primates](https://github.com/infiniteprimates) org.

Please include:

- A description of the issue and its impact
- Steps to reproduce, or a proof-of-concept if available
- Affected versions, or the commit you tested

## Scope

In scope: the code in this repository — the CRDT state model and its merge, the
wire format, gossip and membership, routing, the decision path, and the API
surface.

Two things worth stating plainly:

- **A rate limiter is a control, not an authorization boundary.** Defeating or
  bypassing a configured limit is a security-relevant finding. Depending on
  stomattice to enforce *who* may do something is not a property it offers.
- **The wire format decodes input from peers.** A panic, an unbounded
  allocation, or a length-prefix confusion in the codec is in scope. The codec
  is a fuzz target for exactly that reason.

There is no upstream project to redirect you to. stomattice is written from
scratch and vendors and redistributes no third-party code; if that changes, this
section changes with it.

## What to expect

- An acknowledgment within a reasonable window.
- A good-faith effort to triage and, for in-scope issues, publish a fix.
- This is a small project with no SLAs. If you need a guaranteed response
  window, maintain a hardened internal fork.

## Threat model

Not written yet. It lands alongside the design record's failure-mode work and
will be referenced here. Until then, read §11 of
[`docs/design/architecture.md`](docs/design/architecture.md) as design intent
rather than as assurance.
