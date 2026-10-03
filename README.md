# Stomattice

A distributed rate limiter written in Rust.

Stomattice is a gossip-based, decentralized rate limiter. It ships as a library
crate and as a standalone process in a container, and it answers two questions
that are usually conflated:

- **Inbound protection** — limit what callers do to us. Our latency is already
  on the caller's path, so a round trip to a key's owner is affordable, and a
  limiter outage should **fail open**: the alternative is failing our own
  callers.
- **Outbound conditioning** — throttle what we do to a resource that carries a
  quota. No round trip is affordable, staleness only *shifts* the burst, and
  **fail-open is the wrong default**: when the limiter cannot decide, it hammers
  the resource it exists to protect.

Those two differ in decision mode and failure policy, which is why both are
per-descriptor fields rather than globals.

> **Status: pre-alpha — design phase.** There is no code yet. The repository is
> being scaffolded, the Cargo workspace lands with the first code PR, and
> nothing here is usable, stable, or installable. Watch the repository if you
> want to follow along.

## Design

The design of record is [`docs/design/architecture.md`](docs/design/architecture.md).
It covers scope and non-goals, the CRDT state model and its merge laws, the
decision hot path, key ownership and consistent hashing, failure modes with
their bounds, and the questions that are still open.

Where the design record and this README disagree, the design record wins.

The record is a design under construction, not a specification. Its §13 lists
what it has not answered — including whether outbound conditioning needs the
CRDT core at all, which is a scope question and still open.

## Contributing

Read [`CONTRIBUTING.md`](CONTRIBUTING.md) first. The short version: keep the
correctness invariants intact, one pull request per concern, and commits are
certified under the [DCO](CONTRIBUTING.md#sign-off-your-commits-dco). There is
no CLA.

## License

`MIT OR Apache-2.0`, at your option — see [`LICENSE-MIT`](LICENSE-MIT) and
[`LICENSE-APACHE`](LICENSE-APACHE). Contributions come in under the same terms
(`inbound = outbound`).

The name and any logo are marks of the copyright holder. Neither license grants
trademark rights; see [`TRADEMARK.md`](TRADEMARK.md).

Stomattice is a project of the [Infinite Primates](https://github.com/infiniteprimates)
org. The copyright holder is named in [`AUTHORS.md`](AUTHORS.md).
