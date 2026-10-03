# Contributing

Thanks for contributing to **stomattice**. This is a small, focused project, so
the bar is simple: keep the correctness invariants intact, and certify that you
have the right to submit what you send.

## The invariants

These are the properties the design exists to hold. A change that breaks one
needs a strong argument, and the argument belongs in the pull request.

1. **Merge laws.** The delta G-Counter merges by elementwise max and nothing
   else. Merge stays commutative, associative and idempotent, counters never
   decrease, and there is no decrement operation anywhere in the state model. A
   counter that can go backwards can retract a consumption it already admitted.
2. **No state is destroyed before the cluster has agreed.** Eviction is
   convergence-gated: a bucket may be evicted only once its deltas are
   acknowledged by R replicas, *or* its TTL exceeds the maximum convergence time
   plus a margin, *or* the eviction leaves a tombstone carrying the final count
   for at least one convergence window. **"Evict now, reconcile later" is
   unsound and is not a permitted implementation** — anti-entropy cannot recover
   state that exists nowhere.
3. **No `unsafe`.** Every crate carries `#![forbid(unsafe_code)]`.
4. **Dependency direction is enforced, not hoped for.** The core does not depend
   on transport; transport does not depend on the API layer. The check is part
   of `cargo xtask check`, so a violation fails the build rather than a review.

The reasoning behind these — including the four correctness bugs they come from,
and the failure bound each one protects — is in
[`docs/design/architecture.md`](docs/design/architecture.md).

## Getting started

The Cargo workspace lands with the first code PR. From then on the local gate is
one command:

```bash
cargo xtask check
```

That is local CI parity: `fmt`, `clippy -D warnings`, tests, `cargo deny`, the
MSRV check, and the dependency-direction check.

## Sign off your commits (DCO)

There is **no CLA** — see [`AUTHORS.md`](AUTHORS.md#contributors). Contributions
are accepted under the [Developer Certificate of Origin
1.1](https://developercertificate.org/), the same `inbound = outbound`
certification the Linux kernel uses: you certify that you had the right to
submit what you send.

Certify by adding a `Signed-off-by` line to every commit. Git does it for you:

```bash
git commit -s -m "core: elementwise max merge for the delta G-Counter"
```

which appends, from your configured `user.name` and `user.email`:

```text
Signed-off-by: Your Name <you@example.com>
```

Use your real name and a reachable address. The sign-off is a first-person
statement about a contribution you made — do not sign off on someone else's
behalf. (The one narrow exception is the maintainer certifying an agent-authored
branch; see [Remediation](#remediation).)

A pull request fails the DCO check if any commit in it is missing a valid
sign-off. [Remediation](#remediation) is how that gets fixed without rewriting
history.

The full text you are certifying:

```text
Developer Certificate of Origin
Version 1.1

Copyright (C) 2004, 2006 The Linux Foundation and its contributors.

Everyone is permitted to copy and distribute verbatim copies of this
license document, but changing it is not allowed.


Developer's Certificate of Origin 1.1

By making a contribution to this project, I certify that:

(a) The contribution was created in whole or in part by me and I
    have the right to submit it under the open source license
    indicated in the file; or

(b) The contribution is based upon previous work that, to the best
    of my knowledge, is covered under an appropriate open source
    license and I have the right under that license to submit that
    work with modifications, whether created in whole or in part
    by me, under the same open source license (unless I am
    permitted to submit under a different license), as indicated
    in the file; or

(c) The contribution was provided directly to me by some other
    person who certified (a), (b) or (c) and I have not modified
    it.

(d) I understand and agree that this project and the contribution
    are public and that a record of the contribution (including all
    personal information I submit with it, including my sign-off) is
    maintained indefinitely and may be redistributed consistent with
    this project or the open source license(s) involved.
```

## Automation

Part of this project is written by AI agents (the infiniteprimates agent team)
working under the maintainer's authority. They are tooling, not authors — see
[`AUTHORS.md`](AUTHORS.md#automation). That has two consequences for how commits
are made:

- **Agents never add `Signed-off-by`.** The sign-off is a first-person legal
  certification and a non-person cannot make one. Every agent-authored change is
  certified by a human — the maintainer — through the remediation path below.
- **Agent commits carry an `Assisted-by:` trailer** naming the agent, instead:

  ```text
  Assisted-by: <login>[bot] <<id>+<login>[bot]@users.noreply.github.com>
  ```

  For example:

  ```text
  Assisted-by: zeroclaw-tony[bot] <329726292+zeroclaw-tony[bot]@users.noreply.github.com>
  ```

  `<id>` is the bot account's numeric id, which GitHub embeds in the commit
  address. It changes if the account is renamed or replaced, so read it from the
  App rather than from this file.

  A trailer discloses; it does not certify. Keep it to one line: GitHub's
  squash-message builder generates a `Co-authored-by:` line for every distinct
  author on a branch, so a branch that mixes identities republishes all of them
  into `main`'s history.

## Remediation

A **remediation commit** retroactively adds a missing sign-off. It is a new
commit, so history is not rewritten and no one's work is disturbed. Both forms
are enabled in [`.github/dco.yml`](.github/dco.yml).

### Individual

Authored by the same person as the commits it covers:

```text
DCO remediation commit for Your Name <you@example.com>

I, Your Name <you@example.com>, hereby add my Signed-off-by to this commit: <SHA>
I, Your Name <you@example.com>, hereby add my Signed-off-by to this commit: <SHA>

Signed-off-by: Your Name <you@example.com>
```

### Third-party

Authored by the maintainer on behalf of the failing commit's author. This is the
normal path for agent-authored pull requests:

```text
Third-party DCO remediation commit for <author>

On behalf of <author>, I, <maintainer>, hereby add my Signed-off-by to this commit: <SHA>

Signed-off-by: <maintainer>
```

Only sign off for someone whose authority you actually hold — you cannot certify
a contributor you have no relationship with.

A remediation commit moves the pull request's head, and the ruleset dismisses
stale reviews on push. Approve **after** it lands, not before.

> A failed DCO check normally also offers a **Set DCO to pass** override button.
> It is disabled in this repo — see [`allowOverrideAction`](.github/dco.yml). A
> remediation commit is the only path, because it records the certification in
> git history while the override would record it only in GitHub's audit log. A
> maintainer can re-enable the button for a one-off by setting that key to
> `true`.

## Submitting changes

- Open an issue first for anything bigger than a small fix. The maintainers can
  tell you whether it conflicts with the invariants above — that conversation is
  cheaper before the code than after it.
- **One pull request, one concern.** Aim for roughly 400 lines of
  non-generated diff, tests included. A PR that does two things gets reviewed as
  neither.
- Anything touching the CRDT core or the wire format is reviewed by the
  architect as well as a second reviewer; see [Review](#review).
- **Wire-format changes are breaking by default.** The format is versioned and
  frozen per version, so a change means a new version, not an edit.

## Review

Every pull request is reviewed by someone other than its author, and the
reviewer follows what the change touches:

| The change touches | Reviewed by |
| --- | --- |
| Infrastructure, CI, packaging | the infra owner |
| Tests, benchmarks, chaos assertions | the test owner |
| CRDT core correctness, the wire format | the architect, always |

A review of a correctness change is expected to ask two things: which invariant
it holds, and which test would fail if it broke.

## Style

- Rust only, formatted with `rustfmt`, clean under `clippy -D warnings`.
- `#![forbid(unsafe_code)]`. If you think you need `unsafe`, open an issue
  first — the answer is usually a different data structure.
- Comments explain *why*, not *what*. Where a decision was contested, cite the
  design-record section it came from.
- New behavior needs a test that fails without it. Prefer a property test where
  the property is the point — the merge laws are.

## Maintainers

Maintained by the [Infinite Primates](https://github.com/infiniteprimates) org,
on behalf of the copyright holder named in [`AUTHORS.md`](AUTHORS.md).
