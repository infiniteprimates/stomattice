<p align="center">
  <img src="docs/assets/stomattice-logo.svg" width="320" alt="A split monstera leaf in shades of green, perforated with oval holes and dotted with small round pores.">
</p>

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

The design is not published yet. It is being settled alongside the
implementation, and its shape is still moving — a record published now would read
as a commitment, and every later change would become a correction rather than a
decision. Design decisions land in this repository with the code that depends on
them.

## Contributing

Read [`CONTRIBUTING.md`](CONTRIBUTING.md) first. The short version: keep the
correctness invariants intact, one pull request per concern, and commits are
certified under the [DCO](CONTRIBUTING.md#sign-off-your-commits-dco). There is
no CLA.

## License

`MIT OR Apache-2.0`, at your option — see [`LICENSE-MIT`](LICENSE-MIT) and
[`LICENSE-APACHE`](LICENSE-APACHE). Contributions come in under the same terms
(`inbound = outbound`).

The logo file (`docs/assets/stomattice-logo.svg`) carries that same license; the
stomattice name and logo **as brand identifiers** are governed separately — see
[`TRADEMARK.md`](TRADEMARK.md).

Stomattice is a project of the [Infinite Primates](https://github.com/infiniteprimates)
org. The copyright holder is named in [`AUTHORS.md`](AUTHORS.md).
