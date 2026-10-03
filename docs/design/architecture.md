# Architectural Record — **Stomattice** (distributed gossip rate limiter)

- **Status:** Proposed — v1.1 (ready for build kickoff). Revised 2026-09-30 with a 2026-09-29 research memo and an independent envelope audit (2026-09-30). See §16 Change log.
- **Date:** 2026-09-22 · **Revised:** 2026-09-30
- **Consolidated by:** the principal architect — final sign-off
- **Inputs:** the initial design, specialist design reviews, and a 2026-09-29 research memo. This record carries the conclusions and their effect on the design, not the inputs themselves.
- **Decisions in the 2026-09-30 revision:** D1 decision mode is per descriptor, with the O(1)-vs-O(writers) state consequence (§8); D2 eviction is convergence-gated (§6.4, B4); D3 `try` / `wait` / `deny` are distinct primitives with a per-descriptor failure policy, and outbound fails closed (§5, §10); D4 the "safety net" framing is deleted (§2, §8); D5 the cost model is stated and corrected against the fanout / owner-shard / coalescing model (§11.1); D6 hot-key batching and the cached mitigation short-circuit are required components (§7.1, §11.2); D7 prior art, novelty flags and honest positioning are recorded (§15); D8 Merkle anti-entropy is flagged as unproven, and the outbound conditioning fork is recorded as an open architectural alternative (§15.2–§15.3).
- **Name:** **Stomattice** (official, 2026-10-03) — *stomata* (leaf pores whose guard cells regulate gas exchange) + *lattice* (max-only merge over descriptor keys **is** the join of a join-semilattice, the structure CRDTs are defined over). Verified clean on crates.io, npm, PyPI, NuGet, RubyGems and as a GitHub org name; no implementation collisions (extensive search, 2026-10-03). Crate prefix `stomattice-`, daemon binary `stomattice`. See Q1.
- **Companion docs:** the build plan (epics → milestones → pull requests) and the design-review record, both kept with the project plan rather than in this repository.

---

## 1. Scope & Non-Goals

### In scope
- A **decentralized, gossip-based rate limiter** in **Rust only**, shippable as (a) a **library crate** and (b) a **standalone executable in a Docker container**.
- Three deployment mechanisms served by one architecture: **in-process**, **sidecar**, **standalone cluster**.
- **Multiple descriptors** with **compound keys**; each unique descriptor entry is an **independent limiter bucket** (its own CRDT).
- High availability, sub-millisecond decision latency, tolerance of short over-limit bursts.
- **Two use cases on one engine — and they want opposite defaults:**
  1. **Inbound protection** (classic limiting). Our latency is already on the caller's path, so a round trip to a key's owner is affordable, and a limiter outage should **fail open** — the alternative is failing our own callers.
  2. **Outbound self-throttling (conditioning)** — gating our own calls so they do not stampede a quota'd external resource. No round trip is affordable, staleness only *shifts* the burst, and **fail-open is the wrong default**: when the limiter cannot decide, it hammers the resource it exists to protect. Bounded wait, then fail-closed.
  These differ in **decision mode** (§8) and **failure policy** (§10), which is why both are per-descriptor fields (§5) and not globals. Note the asymmetry with the field: Uber's GRL fails open (§15.2) because it is an *inbound* mesh limiter protecting Uber's own services — that reasoning inverts for us outbound.

### Non-goals (explicit)
- **Strict global accuracy is NOT a requirement.** This is the load-bearing trade: we buy availability and latency with bounded, documented overshoot.
- No central coordination service, no consensus/quorum on the hot path, no synchronous global barrier.
- Not a general-purpose data store; not a policy engine; not an auth system.
- No cross-region strong consistency; no exactly-once accounting.
- No decrementing counters. Anywhere. (See §6.)

---

## 2. Inherited Constraints (non-negotiable)

These came out of the initial design and are treated as requirements, not suggestions:

**Build criteria (Kenneth):**
1. **Rust only.**
2. Ship **both** a library crate **and** a Dockerized executable.
3. Serve **three** deployment mechanisms: in-process link, localhost-HTTP sidecar, and cluster mode behind a service/LB.
4. **Multiple descriptors, compound keys.** `descriptor kind × concrete key` = independent bucket. Config supports multiple concurrent descriptors; concrete values extracted from request/context (headers, auth, path, method, query, body).

**Correctness fixes (research review — fix at build time, baked into §6):**
- **B1 — Rollover is not a valid CRDT op as written.** Dropping "Previous" and starting "Current" on a local clock is non-monotonic; epoch disagreement → a node drops counts another holds → under-count → **over-allow**. *Fix: key counts by `window_id` (epoch); merge same-epoch via elementwise `max`; treat rollover as an epoch-versioned replacement; GC an epoch only on cluster agreement (version/tombstone watermark), never on local clock alone.*
- **B2 — `start_time` reset on re-route is the highest over-allow risk.** *Fix: `start_time` lives in the CRDT payload and merges `min()`; it is NEVER re-pinned on ownership change. Monotonic clock for elapsed/pacing; wall clock for epoch identity.*
- **B3 — Enforce max-only merges; counts ≥ 0.** No operation may lower a counter.
- **B4 — Per-key eviction is not convergence-aware (added 2026-09-30).** The cardinality guards (`max_keys`, LRU/TTL, §5) can drop a bucket's state before its deltas have propagated. Anti-entropy cannot recover state that exists nowhere — so the consumption is lost permanently, the key returns under-counted, and the limit under-enforces for the rest of the window. This is the same class as B1 (a local decision destroying state the cluster has not agreed to). *Fix: eviction is gated on convergence — one of (a) deltas acknowledged by R replicas, (b) TTL > max convergence time + safety margin, (c) evicting leaves a tombstone carrying the final count for at least one convergence window. "Evict now, reconcile later" is unsound and is not a permitted implementation.*

**Operational constraints (research review):**
- **Delta-state** gossip (send only changed records), **not** full-state broadcast. Interval anti-entropy (Merkle/digest) every **5–30 s**.
- Delta gossip every **~100–250 ms**; adapt to topology/churn; bound so `lag × rate` stays under the tolerated burst.
- **Stable node IDs** (never ephemeral hostname/IP) — otherwise phantom keys.
- **SWIM-style suspicion + phi-accrual** failure detection (avoid false-positive node death silently dropping counts). `memberlist` uses Lifeguard for exactly this.
- **Clock skew:** epoch windows assume synced clocks; require tight NTP/chrony + per-node offset monitoring. The `MaxBucket` cap erases drift only when idle, **not** under continuous load.
- **Partition/split-brain → bounded ~2× double-admit risk**; consider per-key ownership lease and/or document the bounded overshoot.
- **Compromised node (trust-everyone gossip):** max-merge lets one node inflate a victim's consumption (lock them out) or under-count itself. Consider authenticated/clamped updates — **best-effort tier**.

**Generation semantics (research review):**
- Tokens are computed **identically on every node** from elapsed time. Every node computes **100 %** of global capacity — never divided by node count. Capped via `min(MaxBucket, …)`.
- No token balancing / no quota distribution. Global aggregate = `global generation − sum(all nodes' consumption)`.
- `EstimatedConsumed = PrevWindowCount × (1 − %throughCurrent) + CurrWindowCount`; `TokensAvailable = MaxBucket − EstimatedConsumed`.
- Rollover (Cloudflare two-window): drop old Previous, shift Current→Previous, init empty Current.
- Self-cleaning: record evicted when older than `2 × WindowSize`; epoch rollover purges dead node IDs.
- Consistent hashing at the routing tier localizes a key's traffic to one node (its local counter is effectively real-time). **Gossip is not a fallback that engages on an event — it runs continuously** (delta gossip plus periodic anti-entropy) and is the correction path at all times. An ownership change alters *which* node is primary, never *whether* correction runs. See §8 for the explicit non-decision.

---

## 3. System Overview

```
                         ┌──────────────────────────────────────────────┐
   Rust app (linked)     │  stomattice-api  (in-process handle)              │
   ─────────────────────►│  check(ctx) -> Decision                       │
                         └───────────────┬──────────────────────────────┘
                                         │ (same core in all modes)
   Non-Rust app          ┌───────────────▼──────────────┐
   ───localhost HTTP/WS─►│  stomattice-http  (sidecar)       │
                         └───────────────┬──────────────┘
                                         │
   Service / LB          ┌───────────────▼──────────────┐
   ───────HTTP API──────►│  stomattice  (cluster node)      │
                         │  routing (HRW) → owner        │
                         └───────┬───────────────┬───────┘
                                 │               │
                    ┌────────────▼───┐   ┌───────▼─────────────┐
                    │ stomattice-core    │   │ stomattice-gossip       │
                    │ CRDT + windows │◄──┤ delta gossip + SWIM │
                    │ + decision     │   │ + anti-entropy      │
                    └────────────────┘   └──────────┬──────────┘
                                                    │ UDP/QUIC (peer net)
                                              other nodes …
```

**One core, three skins.** Everything above `stomattice-core` is I/O and wiring; the decision and CRDT logic is identical in-process, sidecar, and cluster. That is what makes "three deployment mechanisms, one architecture" true rather than aspirational.

---

## 4. Component / Module Breakdown (crate layout)

```
stomattice/                         # Cargo workspace
├── Cargo.toml                  # [workspace] members
├── crates/
│   ├── stomattice-core/            # CRDT + window math + decision hot path. No I/O. #![forbid(unsafe_code)]
│   │   ├── crdt/               # EpochCounter (delta G-Counter), merge, GC/tombstone, start_time
│   │   ├── window/             # two-window weighted estimator, generation, tokens_available
│   │   ├── decision/           # check/consume, multi-descriptor two-phase, retry_after
│   │   ├── clock.rs            # Clock trait (monotonic + wall), injectable for tests
│   │   └── digest.rs           # Merkle/digest for anti-entropy
│   ├── stomattice-key/             # descriptor grammar, compound keys, extractors, registry, config schema
│   ├── stomattice-gossip/          # GossipEngine trait, delta gossip, anti-entropy, membership (SWIM/phi)
│   ├── stomattice-route/           # consistent hashing (HRW), ownership, membership→ring
│   ├── stomattice-api/             # in-process embedding API (builder + handle + traits)
│   ├── stomattice-http/            # sidecar server: HTTP/1.1+2, WebSocket, HTTP/3 (flagged); request→ctx mapping
│   ├── stomattice-client/          # Rust client for sidecar/cluster (h1/h2/ws/h3)
│   └── stomattice/                # binary: config → mode wiring (in-process lib reuse / sidecar / cluster)
├── xtask/                      # cargo xtask: codegen, fixtures, release helpers
├── deploy/                     # Dockerfile, docker-compose, k8s manifests, helm chart
├── docs/                       # this record, ADRs, user guides, ops, security
├── benches/                    # criterion hot-path + gossip benches
├── fuzz/                       # cargo-fuzz: config grammar, extractors, delta codec
└── tests/                      # workspace-level integration + determinism harness
```

**Dependency direction (enforced in CI):** `stomattice-core` depends on nothing in the workspace. `key` and `gossip` depend on `core`. `route` depends on `gossip`. `api` depends on `core` + `key`. `http`/`client` depend on `api`. `stomattice` depends on all. No cycles; core stays I/O-free and testable.

Rationale for the split: **`core` is the correctness-critical, pure crate** — pure functions of (state, clock, time) — so it can be property-tested and simulated without sockets. `gossip` and `route` are the distributed concerns. `http`/`client` are transport. `stomattice` is wiring. This lets the crates be worked on in parallel without stepping on each other.

---

## 5. Key & Descriptor Model

**Model:** a descriptor is a limit rule; `descriptor id × concrete key` identifies an **independent bucket with its own CRDT**. A single request is evaluated against every configured descriptor.

```
DescriptorSpec {
  id:            String,           // "by_apikey_apiendpoint"
  kind:          DescriptorKind,   // Ip | User | ApiKey | ApiKeyApiEndpoint | Header | Claim | Path | Query | Body | Composite
  components:    Vec<Component>,   // ordered extractors
  separator:     char = '\x1f',    // unit separator; length-prefixed canonical bytes on the wire
  transforms:    Vec<Transform>,   // Lowercase | Trim | NormalizeIp(prefix) | NormalizePath | Hash(blake3|sha256, bits)
  missing:       MissingPolicy,    // Reject | Skip | Coarser(fallback_desc_id)
  limit:         u64,              // MaxBucket
  window:        Duration,         // WindowSize
  burst:         u64,              // tolerated burst (see §11)
  max_keys:      u64,              // cardinality cap
  decision_mode: DecisionMode,     // RouteToOwner | LocalReplica       (D1 — see §8)
  wait:          WaitPolicy,       // None | Bounded(max_ms)            (D3 — see §10)
  on_failure:    FailurePolicy,    // Open | Closed | BoundedWaitThenClosed (D3 — see §10)
}
```

**Component / extractor grammar:**

```
Component  := Source '(' Selector? ')'
Source     := header | claim | path | method | query | cookie | body | static
Selector   := name | jsonpath | index          # e.g. header("X-Api-Key"), claim("sub"), body("$.user.id")
Compound   := Component ('+' Component)*       # -> join(render(c), sep)
```

Example: `by_apikey_apiendpoint` = `claim("apikey") + path.route()` → bucket key `apikeyS4GSDFBNMS␟GetUsers`, displayed as `"apikeyS4GSDFBNMS:GetUsers"`.

**Designed-in guards:**
- **Canonicalize before join** — lowercase IPs, trim, normalize `/users/123` → route template, hash secrets so raw identifiers never appear in keys. Fragmentation is the enemy of accuracy and memory.
- **Never substitute empty string** for a missing component — that silently merges distinct principals. Missing policy is explicit per descriptor.
- **Cardinality is bounded, not assumed.** Per-descriptor `max_keys` + eviction + a cardinality budget + a metric. Unbounded `by_user`/`by_path` is the classic memory blowup; the grammar surfaces it at validation time. **Eviction is convergence-gated and is a state transition, not a cache drop (§6.4, B4)** — the budget is enforced by refusing *new* buckets, never by discarding live ones.
- **`stomattice validate --explain`** prints, for a config, the concrete keys a sample request would produce, estimated cardinality, and lint warnings. Validation is a first-class UX, not a schema error dump.

---

## 6. CRDT State Model (the correctness core)

**It is a delta-state G-Counter, keyed by epoch. It is NOT a PN-Counter and must never be labelled one.** Consumption is monotonic within a window; **merge = elementwise `max()` per node per epoch**; **nothing decrements**.

```
type NodeId  = u128;         // stable, opaque; never hostname/IP
type EpochId = u64;          // wall-aligned window index (see below)
type Version = (u64, NodeId); // Lamport-style total order, for tombstones/watermark

struct BucketState {
    kind:        DescriptorKind,       // or descriptor id
    bucket_key:  BucketKey,            // canonical compound key
    start_time:  u64,                  // CRDT payload; join = min(); NEVER re-pinned
    epochs:      BTreeMap<EpochId, EpochState>,
}

struct EpochState {
    counts:    BTreeMap<NodeId, u64>,  // per-node monotonic G-Counter (>= 0, max-only)
    tombstone: Option<Version>,        // cluster-agreed epoch deletion
}
```

### 6.1 Epoch identity — **the correction**
An early design proposed deriving the epoch from `elapsed = now_monotonic − start_time`. **An early review flagged this as unsafe:** monotonic clocks have arbitrary per-node offsets/resets, so `A.now − B.start` mixes clock domains and the same real window can split into *different* epoch IDs on different nodes — each resetting its limit independently → double-admit.

**Decision (incorporating that review):**
- **Epoch identity = wall-clock-aligned:** `epoch_id = floor(unix_wall_now / WindowSize)`. This is a *label*, not a measurement — all nodes name the same window the same way.
- **Monotonic clock** is used for **elapsed/pacing only** (sub-millisecond measurement, "how far through this window are we"), never for epoch identity.
- `start_time` remains in the CRDT and merges `min()` — it anchors bucket birth and the conservative earliest-wins semantics (**B2**), but it is **not** the epoch key. It is set once and only ever lowered by merge.
- **Guard:** require `WindowSize > max_expected_skew` (with overlap/grace), enforced by config validation; monitor per-node offset (§12).

This keeps the two-window math and the monotonic-elapsed requirement while removing the clock-domain mixing bug.

### 6.2 Merge
```
merge(a, b):
  for (epoch_id, eb) in b.epochs:
    match a.epochs[epoch_id]:
      vacant    -> insert eb
      occupied  -> counts[n] = max(counts[n], eb.counts[n])  for all n   # elementwise max
                   start_time = min(start_time, eb.start_time)
                   tombstone  = max(tombstone, eb.tombstone)
```
Properties to prove (property tests, §M1): **commutative, associative, idempotent** per epoch; counters never decrease; counts ≥ 0.

### 6.3 Rollover & GC — **the B1 fix**
Rollover is **not** "drop Previous". It is:
1. Compute the current `epoch_id` from wall clock.
2. Write new increments under that `epoch_id`. Old epochs **remain** — no local-clock deletion.
3. The two-window estimator reads `epoch_id` and `epoch_id − 1`; older epochs are invisible to the decision but still mergeable.
4. **GC an epoch only on cluster agreement:** a tombstone `Version` is gossiped; a node deletes an epoch only when it has **applied** the tombstone and its local version vector **dominates** the tombstone's. Otherwise it retains the epoch and triggers anti-entropy.
5. Global GC only when all nodes ack, or under a **lease** that forbids further increments to a closed epoch.

**Consequence:** a lagging or partitioned node can never lose counts it holds, and can never over-allow *because its clock moved*. It can still over-allow due to partial membership — that is the bounded, documented split-brain risk (§11), not a correctness bug.

### 6.4 Key eviction must follow the same protocol — **required protocol (research review, 2026-09-29; gate settled 2026-09-30)**

§6.3 makes *epoch* deletion convergence-aware. **Per-key state eviction is not, and that is a correctness gap, not merely a memory-control detail.**

If a bucket's state is dropped (LRU/TTL under `max_keys`, or memory pressure) *before* its deltas have propagated, and no replica holds the final count, that consumption is lost permanently — the counter cannot be retracted upward again, so the window under-counts → **over-allow**. Same direction of error as B1/B2, and silent.

**Adopted shape (to be specified in M1/M3):**
- Eviction is a **state transition**, and it takes the same three phases as epoch GC: `open → closed` (no further increments; deltas flushed and acked) `→ tombstoned` (cluster-agreed, applied only when the local version vector dominates the tombstone's).
- **A bucket that is still receiving traffic is not evictable.** The cardinality budget is enforced by **denying new buckets** (or failing open per §10), never by silently dropping live ones.
- §6.3's rule transfers verbatim: never delete on a local clock alone; on a failed dominance test, retain and force anti-entropy.
- **Q8** is the same question from the other end: a re-created bucket must not receive a fresh full budget mid-window, and it inherits `start_time` rather than re-pinning it (**B2**).

**Status:** the *gate* is settled and is a correctness requirement, not a tuning knob (**B4** in §2; **Q8**): eviction must be convergence-gated — R-replica ack, or `TTL > max convergence + margin`, or a tombstone carrying the final count for one convergence window. Still open: the re-creation semantics (Q8). Resolve alongside the epoch tombstone work — they are one protocol. Do not build a second, weaker tombstone mechanism for keys.

---

## 7. Handler Flow (decision path, gossip path, rollover)

### 7.1 Decision path (hot path — target < 1 ms p99)
```
check(ctx) -> Decision:
  keys = extract(descriptors, ctx)            # borrow-heavy, minimal alloc
  # two-phase: check all, then commit all
  plan = []
  for (desc, bucket_key) in keys:
      bucket = registry.get_or_create(desc.id, bucket_key)   # sharded map, read-optimized
      est    = estimator.estimated_consumed(bucket, now)     # two-window weighted
      avail  = max_bucket - est
      if avail < cost: return Deny { retry_after, descriptor: desc.id, remaining: avail }
      plan.push(bucket)
  for bucket in plan:                         # commit only if ALL pass
      bucket.local_increment(node_id, epoch(now), cost)      # monotonic; emits a delta
  return Allow { remaining, reset_at, epoch }
```
- **Decision mode (D1 — see §8):** `RouteToOwner` executes the decision on the key's HRW owner (one hop, authoritative counter); `LocalReplica` executes it against the local replica (no hop, bounded staleness). Set per descriptor.
- **`wait` (D3 — see §10):** a bounded-wait primitive sits alongside try/deny. Implemented **non-blocking** — no parked thread; the caller is re-scheduled against the next window boundary (LINE's `acquire(maxWaitForMillis)` precedent). It is the outbound default because a deny *is* a retry: refusing a message handler rebuilds the stampede the limiter exists to prevent.
- **Hot-key short-circuit (Cloudflare's cached mitigation flag):** once a bucket is known over-limit, the node caches that fact and short-circuits subsequent checks for that key without touching the registry — the mitigation path is a cached boolean, not a per-request state read. If a key's *network* path (not its state) is the bottleneck, Gubernator-style batching (500 µs window) is the fallback.
- **Multi-descriptor atomicity:** check-all-then-commit. A tiny TOCTOU window is *accepted* — bursts are tolerated by design, and the alternative (rollback) would introduce decrements. This is a deliberate trade, documented.
- **No I/O on the hot path.** No consensus, no network, no disk. Local state + local clock only.
- **Local ownership** (consistent hashing, §8) makes this node's counters effectively real-time for routed keys.

### 7.2 Gossip path (background, off the hot path)
- **Delta gossip (~100–250 ms):** ship records changed since the peer's last ack — `(bucket_key, epoch, node_id, count, start_time, tombstone)`. Adaptive interval; bounded so `lag × rate < tolerated burst`. **Coalesced per gossip interval — never per request.** Per-request propagation is the anti-pattern the record exists to avoid: it turns one per-key record into O(requests × peers) traffic, ~4,000× the coalesced figure (§11.1).
- **Anti-entropy (5–30 s):** digest/Merkle exchange per bucket-range; reconcile divergence. This is the durable correctness backstop for dropped deltas.
- **Membership:** SWIM + phi-accrual (Lifeguard-class) suspicion; stable node IDs; membership feeds the router.
- **Never blocks a decision.** Gossip failure degrades freshness, not availability.

### 7.3 Rollover path
Driven by §6.3: wall-clock epoch advances; two-window reads `{epoch, epoch−1}`; GC is tombstone-driven and cluster-agreed. No local-drop rollover anywhere in the code.

---

## 8. Key Ownership & Consistent Hashing

**Goal:** localize a key's traffic to one node so that node's local counter is near-real-time. Gossip is *not* the fallback for that path — it is always on (delta ~100–250 ms, anti-entropy 5–30 s), so crash / scale-in / re-route change *which* node is primary, not *whether* correction is running.

> **Explicit non-decision (research review, 2026-09-29).** The phrase "safety net" (here and in §2) reads as a discrete fallback that engages on an event. The design has no such trigger, and adding one is a *failure-model* change, not a detail: SWIM/phi suspicion under CPU starvation or network delay produces false-positive node death — the reason Lifeguard exists — so a failure-triggered path would fire spuriously and drop counts. If a discrete trigger is ever wanted, evaluate these explicitly: **(a)** membership change (a node is suspected/dead and its keys are re-homed), **(b)** ring movement (ownership re-assignment), **(c)** divergence (the periodic Merkle digest finds a mismatch). Today all three are handled *by the continuous path*, not by a separate one.

**Decision: rendezvous hashing (HRW) over the live membership set.** HRW *is* a consistent hash (highest-random-weight), and it beats a vnode ring here: no ring/vnode tuning, minimal disruption on membership change, trivially correct for small-to-medium clusters. Crate: `hashring` (ring) or hand-rolled HRW via `xxhash`. Final choice is a spike (§M5, Q3).

**Routing-tier options (design review):**

| Option | Who routes | Verdict |
|---|---|---|
| **SDK/client-side HRW** | the caller (Rust SDK / sidecar) | **Preferred.** The descriptor key is visible to the caller; no extra hop; works for non-Rust via the sidecar. |
| L7 LB consistent-hash | the LB | Second — only if it can parse the key from headers/auth (not always true; the LB may not see it). |
| Server-side forward-to-owner | receiving node | **Fallback only.** Adds a hop and tail latency; use when the caller can't route. |

**Ownership vs `start_time` (B2):** on re-route or rebalance, the new owner **inherits `start_time` from the CRDT payload** — it is never reset to "now". A re-pinned start time would change bucket identity and make existing counts unreachable → over-allow. This is enforced by an invariant test, not by convention.

**Decision D1 — the decision mode is per descriptor, not global (2026-09-30).** The record previously implied two incompatible things at once: the route-each-key-to-one-owner option, and §1's "local optimistic accounting + eventual correction". They are indistinguishable on a diagram and differ completely in cost, so they are now chosen explicitly, per descriptor:

| Mode | Where the decision runs | Per-key state | Cost | Default for |
|---|---|---|---|---|
| **RouteToOwner** | the key's HRW owner (one hop) | **O(1)** — single writer plus an ordering tag (`owner_id`, `seq`) | one network hop on the request path | **inbound protection** — the latency is on the request path anyway, and the owner's counter is authoritative |
| **LocalReplica** | any node, against its local replica | **O(writers)** — every node that touched the key holds a component | no hop; bounded staleness | **outbound conditioning** — a round trip is unaffordable and staleness only shifts the burst |

The consequence is structural, not cosmetic: **the O(N) per-key blowup is a consequence of the ownership choice, not of the CRDT family.** Single-writer-per-key keeps the two-integer shape Cloudflare uses; write-everywhere is what forces a per-replica component (§11.1, row 6). So §6's "drop the PN framing, keep a G-Counter" is right about the *name* and under-states the point — resolve the mode and the state cost resolves itself.

**Residual trade (unchanged):** a single request may touch several descriptors whose keys hash to different owners. Under RouteToOwner we either (a) pin the whole request by a configured **routing key** (default: coarsest descriptor component) and let gossip cover the rest, or (b) forward per-bucket to owners. See Q3/Q4. Under LocalReplica the question does not arise.

---

## 9. Deployment Matrix

| Mechanism | Audience | Entry surface | Gossip | State |
|---|---|---|---|---|
| **In-process** | Rust-native, lightweight | linked crate (`stomattice-api`) | none (local-only) or optional cluster join | local only / local+merged |
| **Sidecar** | non-Rust workloads | localhost HTTP (UDS preferred → TCP), WebSocket, HTTP/3 (flagged); Rust `stomattice-client` | optional — sidecar may join a cluster | local (+cluster if joined) |
| **Cluster** | large scale | service/LB → `stomattice` HTTP API; SDK-side HRW routing | required | replicated via gossip, reconciled by anti-entropy |

All three share `stomattice-core`. The binary `stomattice` is the sidecar **and** the cluster node — mode is configuration, not a separate program.

---

## 10. Transport

**Design-review findings, adopted:**
- **Sidecar localhost:** **Unix domain socket** first (no TCP, lowest latency, fd perms, backpressure); **HTTP/1.1 keep-alive over TCP** second (debuggability, universal clients); **HTTP/2** for multiplexed streams; **WebSocket** for push/config/streaming — **not** for request-response checks; **HTTP/3/QUIC loses on localhost** (UDP overhead, no loss/reordering benefit). *An earlier "async/websocket + HTTP/3 for the sidecar" proposal is deprioritized for the localhost hop; H3 is offered as a flagged endpoint for parity/edge cases, not the default.*
- **Cluster peer transport:** UDP or QUIC for gossip (spike-decided, Q2); Protobuf/gRPC for optional owner-forwarding.
- **Wire contract (v0):**
  - `POST /v1/check` `{key, cost, ttl_ms?, op_id?}` → `{allowed, remaining, reset_at_ms, retry_after_ms?, limit, node, epoch}`
  - `POST /v1/batch` — array in/out.
  - `GET /v1/healthz`, `GET /v1/metrics` (Prometheus).
  - Idempotency via `op_id`.
  - `POST /v1/wait` `{key, cost, max_wait_ms, op_id?}` → same response shape plus `waited_ms`. **Bounded and non-blocking** (re-scheduled at the next window boundary, never a parked thread).
  - **Three primitives, not one (D3 — 2026-09-30):** `try` (decide now), `wait` (bounded wait, then decide), `deny` (explicit refusal with `retry_after`). A limiter that can only deny pushes the retry decision onto the caller — which is how the stampede is rebuilt.
  - **Failure policy is per descriptor (D3), not global.** `on_failure ∈ {Open, Closed, BoundedWaitThenClosed}`. **Outbound defaults to `BoundedWaitThenClosed`**: fail-open for outbound means "when the limiter cannot decide, hammer the resource you were protecting" — there the failure mode is external and not self-inflicted. **Inbound defaults to `Open`**: there the failure mode is our own availability. (Uber GRL fails open too, and that is correct *for its direction* — the asymmetry is the point, not a contradiction.) *(Never fail-closed silently.)*
  - **Unavailability fallback:** if the sidecar is unreachable, the caller falls back to a per-instance emergency token bucket + circuit breaker. **Still open:** the reconcile rule for that bucket's consumption — merge it into the G-Counter (inflates) or not (never converges post-partition) — the first convergent finding, now Q11 below.
- **Crates:** `tokio`, `hyper`/`hyper-util`, `axum`; `h2`; `quinn` (H3); `axum::extract::ws` / `tokio-tungstenite`; `tonic`+`prost`; `xxhash-rust`/`hashring`; `tower` (timeout/buffer/circuit); `tracing`/`metrics`.

---

## 11. Failure Modes & Bounds

| Failure | Effect | Bound / mitigation |
|---|---|---|
| **Partition / split-brain** | each side sees partial membership; max-merge cannot retract admits → up to **~2× double-admit** | documented bound; optional **per-key ownership lease** (minority rejects) to convert to strict; static per-node budgets cap overshoot at sum-of-budgets |
| **Clock skew / jump** | epoch disagreement | wall-aligned epoch IDs (§6.1); require `WindowSize > max_skew`; per-node offset monitoring; `MaxBucket` cap only helps when idle — monitored, not assumed |
| **Gossip lag** | stale estimates → transient over/under | bounded by `interval × rate < tolerated burst`; anti-entropy reconciles |
| **Node crash / scale-in** | ownership moves | HRW minimal movement; gossip replays deltas; `start_time` inherited (never re-pinned) |
| **False-positive node death** | counts silently dropped → over-allow | SWIM + **phi-accrual** suspicion (no binary death); stable node IDs |
| **Compromised node** | inflate a victim (lock-out) or under-count itself | **best-effort tier** (Q7): authenticated deltas + clamped per-node update rates; documented as out of the strict-trust model |
| **High-cardinality keys** | memory blowup / fragmentation; **and evicting live state loses that consumption permanently → over-allow (§6.4)** | `max_keys` + **convergence-aware eviction (§6.4)** + cardinality budget + validation-time lint; the budget is enforced by denying *new* buckets, never by dropping live ones |
| **Sidecar down** | caller can't check | per-descriptor failure policy (D3, §10): `Open` inbound, `BoundedWaitThenClosed` outbound; emergency local bucket + circuit breaker; reconcile rule still open (Q11) |
| **Hot key** | one bucket concentrates traffic; a per-request state read becomes the bottleneck | locally cached over-limit flag (Cloudflare) short-circuits the check without a registry read; Gubernator-style batching (500 µs) if the *network* path is the bottleneck (§7.1) |

### 11.1 Cost model — corrected 2026-09-30

The earlier estimates priced the **anti-pattern**, not this design. The arithmetic was right in every case; the assumptions were wrong, and the dominant error was assuming full broadcast instead of O(1) fanout and owner-only replication.

| # | Assumption as written | Correction | Effect |
|---|---|---|---|
| 1 | membership gossip contacts all ~49 peers per interval → ~200 KB/s | SWIM/memberlist contacts a constant fanout (~3 peers, ~200 ms) — that O(1) fanout is *why* gossip is O(N) and not O(N²) | **~15 KB/s** per node (÷16) |
| 2 | every changed key replicated to all peers → 125 MB/s | owner-only generation (~400 of 20k keys per node under HRW) × **R = 3** replicas | **~154 KB/s** (÷816) |
| 3 | propagate per request → 627 MB/s at 400k rps | coalesce per gossip interval (400k/s → 100k/interval → ≤20k distinct deltas → ~400 owner keys) | **~154 KB/s** at fixed topology (÷250); see the decomposition below |
| 4 | 16 B/key resident | 16 B is the counter *payload*; resident is 40–80 B once map/entry overhead, epoch and `start_time` are counted | **40–80 MB** per 1M keys, not 16 MB |
| 5 | state replicated to all N → 800 B/key | R = 3 replicas | **48 B/key** *payload* (÷17); ~120–240 B/key resident |
| 6 | a PN-Counter per key is O(N) | conditional, not structural — O(1) under single-writer, **O(writers) under `LocalReplica`** (D1, §8) | architectural, not numeric |

**Net:** ~150 KB/s per node and ~48 MB per 1M keys *of payload* — and *only* with (a) delta gossip at O(1) fanout, (b) coalescing per interval, (c) owner-sharded writes with R replicas. Skip any one of the three and the original figures are correct and the system is unbuildable.

**Independent audit (2026-09-30) — six corrections to the memo's framing.** The arithmetic survives; several framings do not:

1. **Scope of the 400k rps is unstated.** 627 MB/s is per node *only if* 400k rps is per node. Cluster-wide it is ~12.5 MB/s per node — a 50× swing. State the scope; read this table as per-node.
2. **The ~4,000× headline conflates two independent decisions.** At fixed topology (owner-sharded, R = 3) coalescing is ~250× (38.4 MB/s → 153.6 KB/s). At fixed cadence, replacing broadcast with owner-shard + R is ~816×. Their product is ~4,083×. "Batching is the difference between feasible and absurd" overstates batching alone; the **decomposition** is the useful part, because the two factors are separate design levers.
3. **Payload floor ≠ resident cost.** Row 5's 48 B/key is a payload floor; resident at R = 3 on the 40–80 B/entry basis is ~120–240 B/key. Never mix the two units in one column.
4. **Dirty fraction `f` is unstated and load-bearing.** Rows 2 and 3 assume *every* tracked key is dirty every round. Real cost scales linearly with `f` — a workload where 1 % of keys move per interval is ~100× cheaper than the worst case in the table.
5. **Unit convention.** SI vs binary swings every ratio by ~2.4 %. Say which is meant.
6. **The sharpest corollary:** under `LocalReplica` — the outbound default — there is no single writer, so per-key state is **O(writers)**: up to N components per key, needing version vectors / LWW rather than two counters, plus merge cost. **The coordination-free mode is the cheap mode on latency and the expensive mode on state.** That trade is invisible in the memo and belongs in the design, not a footnote.

**No published comparables exist.** No limiter in the landscape publishes bytes/node or bytes/key under attack (Gubernator publishes rps/node and latency; the academic CRDT limiters publish 3 KB per gossip at trivial scale). These figures are reasoned, not measured — which is why the **simulation is the next step** (Q10) and why the simulation instruments the envelope instead of trusting it. Treat them as the design's opening bid, not as its evidence.

**Bounded overshoot is a feature, not a bug.** The design explicitly buys availability and sub-ms latency with a documented, monitored overshoot envelope. Any proposal that removes the envelope must state what latency/availability it spends.

---

## 12. Observability

- **Cluster:** gossip round-trip time, delta lag (per peer), anti-entropy divergence count, membership churn, tombstone/GC watermark, per-node clock offset (NTP/chrony + app-level).
- **Decision:** p50/p99/p999 latency, allow/deny ratio per descriptor, `estimated_consumed` vs `MaxBucket`, retry_after distribution, emergency-fallback activations.
- **State:** bucket count, cardinality per descriptor, eviction rate, **evictions refused because the bucket was still live**, **new-bucket admissions refused at the cardinality budget (§6.4)**, memory per bucket.
- **Alerts:** clock offset > `WindowSize/4`; delta lag > burst budget; divergence non-converging; cardinality cap hits.

---

## 13. Open Design Questions (resolve during build)

- **Q1 — Name/repo. ✅ RESOLVED 2026-10-03 (Kenneth).** The project is **Stomattice** (etymology and collision status in the header). crates.io was the binding constraint — crate names are globally unique — and `stomattice` is free on crates.io, npm, PyPI, NuGet, RubyGems and as a GitHub org name, with no exact-match web results. **Crate prefix `stomattice-`; daemon binary `stomattice`.** Reserve only the crates we will actually publish (`stomattice`, `stomattice-core`, `stomattice-client`, plus any other member we ship); speculative name-holding is squatting, and crate names are permanent once published — so decide before any placeholder publish.
- **Q2 — Peer transport + membership crate.** `memberlist` (SWIM+Lifeguard) + custom delta layer vs `chitchat` (delta-scuttlebutt) vs hand-rolled. **Spike in M4.**
- **Q3 — Ring vs HRW**, and the routing-key selection (per-request routing key vs per-bucket ownership + forwarding). **Spike in M5.**
- **Q4 — Multi-descriptor atomicity** under a partially-committing plan (check-all-then-commit is the default; confirm).
- **Q5 — GC protocol details:** watermark source (per-peer version vector vs global quorum), tombstone retention, lease vs full-ack.
- **Q6 — Bootstrap/discovery:** static seeds vs membership gossip vs k8s StatefulSet+headless (leaning StatefulSet for stable node IDs).
- **Q7 — Compromised-node tier:** authenticated deltas + per-node rate clamping — best-effort scope confirmation.
- **Q8 — Bucket eviction/re-creation (partly resolved 2026-09-30).** The *gate* is settled and is a correctness requirement, not a tuning knob: eviction must be convergence-gated (B4 in §2, §6.4). Still open: the re-creation semantics — how a returning key is treated so it does not receive a fresh full budget mid-window (tombstone-with-final-count is the leading option, since it satisfies both).
- **Q9 — Cardinality budget defaults** per descriptor kind.
- **Q10 — Simulation (the next step).** Validate the corrected cost model (§11.1) and the convergence/overshoot envelope against the *chosen* design: N nodes, M keys, HRW ownership, R = 3 replicas, delta gossip 100–250 ms, anti-entropy 5–30 s; measure bytes/node, resident bytes/key, convergence lag, and overshoot under partition, churn and skew. Nothing in the landscape publishes these numbers (§11.1), so this is the only route from reasoned to measured. Owner: the accuracy-harness lane, with the implementation owner on the harness.
- **Q11 — Emergency-bucket reconcile rule (open since the 2026-09-22 design reviews).** When the sidecar is unreachable the caller falls back to a local emergency bucket (§10). Its consumption must either merge into the G-Counter (inflating, but converging) or be explicitly marked as unattributed (never converging, but honest) — and either way emitted as a metric. This is the kind of gap that surfaces as a 3am incident; it is still unspecified.
- **Q12 — Does outbound conditioning need the CRDT core at all?** §15.2 shows the closest published analogues to our outbound goal — Uber GRL's drop-ratio directives, Netflix's adaptive per-node limits — reaching it *without* per-key state, without gossip and without a CRDT. If conditioning is a directive-plus-local-shedding problem, then D1's `LocalReplica` mode is over-built for it: we would be paying O(writers) per-key state (§11.1) for coordination the use case does not need. Decide **before** the cluster work lands: (a) one engine, two modes, or (b) a conditioning path deliberately *not* CRDT-based. **Kenneth's call — it is a scope question, not a tuning question.**

---

## 14. Consequences

**Becomes easier:** horizontal scale without a coordination service; sub-ms decisions; three deployment shapes from one core; descriptors composable from config; correctness isolated in a pure, property-testable crate.

**Becomes harder:** reasoning about global accuracy (bounded, not exact); clock hygiene is now operationally load-bearing; high-cardinality descriptors need active budget management; recovered-node rejoin and epoch GC need the tombstone protocol to be right.

**Irreversible-ish (one-way doors):** the CRDT payload shape and epoch model (wire format v0). Get §6 right before M2; a later change to epoch identity would require a flag day.

---

## 15. Prior art & positioning (added 2026-09-30)

### 15.1 Verified characterizations
Cloudflare's PoP-local sharded counter with async increments and a locally cached mitigation flag: accurate as cited (§3, §7.1). Gubernator: consistent-hash single owner, opt-in GLOBAL mode (local answer + async owner update + owner broadcast), peer discovery via etcd / Kubernetes EndpointSlice / round-robin DNS — **memberlist is historical, not current**; its hot-key answer is **BATCHING** (500 µs default, observed batches of 1,000), not gossip. golimit: Uber ringpop, shared-nothing, every node reads and writes counters (**ringpop is unmaintained**). Doorman: central coordinator with client leases and explicit optimistic/pessimistic/safe expiry modes (**archived Nov 2024**). LINE: per-instance quota split with wall-clock-aligned windows, NTP required, consumer-side **wait** semantics. The academic CRDT limiters (Chalmers thesis; IJSET; `souviks22`) are max-merge, small-scale artifacts — `souviks22` is **not** a PN-Counter, it is max-based reconciliation with libp2p gossip, measured at 3 nodes / 1,000 users / 3 KB per gossip.

### 15.2 The gaps that matter
- **Uber GRL (2026) — the largest one.** Control-plane-directed *probabilistic dropping*: mesh clients receive a drop-ratio directive and shed that fraction locally; zone aggregators recompute every second; no per-request counters on the hot path; fails **open**; ~80M rps across 1,100+ services with 2–3 s enforcement lag. It is a third architecture family (control-plane/aggregate), not a variant of ours, and it is the closest published analogue to **outbound conditioning** — drop-by-ratio is proportional shedding rather than deny.
- **Netflix `concurrency-limits`** — adaptive per-node limits (Vegas / Gradient2), no global counter. Coordination-free conditioning.
- **Envoy `ratelimit`** — Go/gRPC over Redis, local over-limit cache, `shadow_mode`, and a documented async-increment overshoot. The ancestor of the descriptor-grammar idea.
- **Consequence for the outbound use case:** Uber GRL and Netflix both argue that *conditioning* may not need shared state at all. That is a real architectural fork, not a detail. It does not change the v0 plan (shared state is what makes inbound protection work, and one core must serve both), but a **local adaptive mode** for outbound is now recorded as the alternative family to evaluate before that use case is built — it would delete the CRDT from that path, so it needs a deliberate decision rather than arriving by accretion.

### 15.3 Novelty flags — do not treat as proven
- **Merkle/summary anti-entropy has no precedent in the rate-limiter landscape.** It is borrowed from KV-store practice (Dynamo/Cassandra key-range Merkle trees, segment hashes, periodic repair); `chitchat`'s digest-based divergence detection is the closest thing. It is a component to be validated, not a design pattern to be relied on — and §6.3's GC protocol depends on it.
- **No published gossip-volume or memory-per-key numbers exist** for any limiter under attack (§11.1). Our cost model is reasoned. Q10 is the validation.

### 15.4 Where stomattice sits (honest positioning)
Three families: (a) centralized (Redis, AWS API Gateway), (b) owner-routed (Cloudflare, Gubernator default), (c) control-plane/aggregate (Uber GRL, Doorman, LINE). Ours — local optimistic accounting + consistent-hash ownership + gossip correction — is **closest to Gubernator's GLOBAL mode, but with owner broadcast replaced by a CRDT**. The claim is therefore narrow and testable: *make Gubernator-GLOBAL work without a single owner and without a central control plane.* We are not reinventing Cloudflare, and we should not describe it that way.

---

*Draft v1.1 — second pass, 2026-09-30.*

## 16. Change log

- **2026-09-22 — v1.0.** Consolidated from the initial design and three design reviews.
- **2026-09-30 — v1.1, first pass** (memo truncated in transit): §2/§8 "safety net" framing replaced with continuous gossip plus an explicit non-decision; §6.4 added (**B4** — per-key eviction is not convergence-aware); §11 risk row and §12 metrics updated to match. *The memo's §1.3–§1.5 and §§2–§4 were absent from that pass; superseded by the entry below.*
- **2026-09-30 — v1.1, second pass** (full memo + independent audit): **D1** decision mode per descriptor, with the O(1)/O(writers) consequence (§8); **D3** `try`/`wait`/`deny` as distinct primitives, per-descriptor failure policy, and the outbound fail-closed asymmetry (§5, §7.1, §10); **D5** corrected cost model **§11.1** (O(1) fanout, owner-shard with R = 3, coalescing — ÷16 / ÷816, and the 4,083× headline **decomposed** into ~250× coalescing at fixed topology × ~816× topology, plus six framing corrections from the independent audit); hot-key short-circuit and batching (§7.1, §11); **§15** prior art, novelty flags and honest positioning; Q1's naming research (superseded 2026-10-03 — see the entry below); Q8's eviction gate; new **Q10** (simulation), **Q11** (emergency-bucket reconcile), **Q12** (does conditioning need the CRDT core at all).
- **2026-10-03 — project named (Q1 closed).** The project is **Stomattice**; the earlier placeholder names are retired. Crate prefix `stomattice-`, daemon binary `stomattice`. Resolved by Kenneth after a registry/org collision search. No design content changed.
