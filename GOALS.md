# service-atoms — Goals

> Replaces DNS for services inside the ecosystem while remaining decentralized at the implementation layer — identity, protocol, schema, policy, and endpoint primitives composed into typed services.

*This document is derived from `aish/ARCHITECTURE.md` (now `xdao/xdao/ARCHITECTURE.md` §The *-Atoms Catalogs). Sections marked **Generated** are pattern-based and are intended as a starting point for revision, not as decided plan.*

---

## What this catalog makes civilization-grade

DNS resolves names to addresses. Inside an ecosystem, services need much more: protocol contracts, request/response schemas, auth policies, retry semantics, rate limits, circuit breakers. Today each service catalogs this in OpenAPI here, Protobuf there, Helm charts elsewhere — none of them composable or machine-portable across runtimes.

By cataloging the primitives, `service-atoms` turns this domain from opaque-and-ephemeral to typed, versioned, composable, machine-readable, and open — the civilization-grade properties the ecosystem requires.

## What it catalogs

### Atom types

- **`identity`** — Service identity primitives (DID-like, but ecosystem-native).
- **`protocol`** — Wire protocols (HTTP/REST, gRPC, queue, event-stream).
- **`schema`** — Request, response, and event shapes — typed and versioned.
- **`policy`** — Auth requirements, rate limits, retry policies, circuit-breaker thresholds.
- **`endpoint-pattern`** — URL/topic patterns including parameterization conventions.

### Compositions: `services`

A service composition assembles identity + protocol + schemas + endpoints + policies into a typed, version-tagged service definition. Cataloged once, consumable by any runtime that speaks the protocol.

### Rule types

- **`version-compatibility`** — Semver-style compatibility between service versions.
- **`auth-required`** — Which endpoints require which auth methods (gated by identity-atoms).
- **`schema-evolution`** — Permitted backwards-compatible changes; forced-major boundaries.

## Runtime consumers

- **universal-bus** — Future runtime — the service-layer execution engine. Universal Bus consumes service-atoms to route, authorize, and validate requests.
- **aish** — Via Universal Bus plugin once both exist. Until then, aish can `aish run` against catalog'd services for natural-language invocation.

## Status & priority

**Current status:** `proposed`

**Priority tier:** Tier 4 — Build when companion runtime exists

**Trigger / activation condition:** Universal Bus runtime exists. Until then, service-atoms accumulates spec without runtime pull.

## Roadmap *(Generated — milestone shapes mirror aish's roadmap pattern; revise as actual work begins)*

### v0.1 — Bootstrap & spec acceptance

**Goal:** Service definition schema accepted as XAIP. Three real services cataloged as proof.

**Success criterion:** Three services round-trip from catalog → generated client → working call.

**Kill criterion:** Universal Bus design diverges substantively from service-atoms schema — pivot or pause.

**Work:**

- [ ] XAIP: service-atoms composition schema
- [ ] Define 5 atom types' schemas
- [ ] Catalog three internal services as seed
- [ ] Generate OpenAPI export from service composition
- [ ] Verify export against actual service

### v0.2 — Adoption & expansion

**Goal:** Universal Bus consumes service-atoms for routing.

**Work:**

- [ ] Universal Bus discovery flow integrated
- [ ] Auth policy enforcement via identity-atoms
- [ ] Add 20 cataloged services

### v1.0 — Operational

**Goal:** Service-atoms is the canonical source-of-truth for every ecosystem-internal service.

## Concrete atom example *(Generated — illustrative, not seed content)*

```yaml
services/auth-issuer/definition.yml
---
id: auth-issuer
type: composition
version: 1.0.0
identity:
  did: did:cse:auth-issuer
protocol: https-grpc-dual
schemas:
  request: schemas/issue-token-v1.yml
  response: schemas/token-response-v1.yml
endpoints:
  - pattern: /v1/tokens
    methods: [POST]
    auth: bearer
policies:
  - rate-limit: 100/minute/identity
  - retry: exponential-backoff(3, 250ms)
```

## Adoption strategy *(Generated)*

Adoption follows Universal Bus. Pre-bus, service-atoms is a coordination tool — the spec everyone agrees to so the bus can be built.

## Civilization-grade property checklist

Every catalog must satisfy these before v1.0. Failing any blocks a release.

| Property | Mechanism in this catalog |
|---|---|
| Typed | JSON Schema in `schemas/` validates every atom, composition, rule |
| Versioned | Every atom has a semver `version` field; compositions reference atoms by version-pinned ID |
| Machine-readable | `exports/catalog.json` published on every release |
| Composable | Compositions reference atoms by ID; CI verifies references resolve and no circular dependencies |
| Open | Apache-2.0 licensed; LICENSE file present |
| Durable | No external dependencies for primary content (no remote image URLs, no vendor APIs in the hot path) |

## Related

- **Spec:** [atoms-spec](https://github.com/convergent-systems-co/atoms-spec) — the canonical structure every catalog conforms to
- **Tools:** [atoms-tools](https://github.com/convergent-systems-co/atoms-tools) — CLI for validate / export / bootstrap / resolve
- **Federation:** [xdao](https://github.com/convergent-systems-co/xdao) — ecosystem directory and discovery
- **Umbrella:** [atoms](https://github.com/convergent-systems-co/atoms) — every catalog as a git submodule
- **Manifest:** [`ATOMS.yml`](./ATOMS.yml) — this catalog's machine-readable manifest
- **Standard:** [`README.md`](./README.md) — catalog overview and contribution flow
