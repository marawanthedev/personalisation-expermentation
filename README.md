# personalisation-expermentation

Design docs and specification for an edge-native personalization and experimentation platform built on Cloudflare Workers, KV, and Durable Objects, with a Node.js and PostgreSQL control plane.

## Documents

| Document | What it covers |
|---|---|
| [docs/tech-spec.md](docs/tech-spec.md) | Full technical specification: architecture, data model, flows, code, security, edge cases, testing, rollout |
| [docs/findings.md](docs/findings.md) | Platform findings and decisions made so far, with the reasoning behind each |
| [docs/architecture/](docs/architecture/) | Architecture diagram |

## TL;DR

- **Two flows, one rule set.** Experiments and personalization both read one rule set from Cloudflare KV, which the Node service publishes from Postgres.
- **Experiments use hashing, not storage.** Users are bucketed with `hash(salt + userId) mod 10000`. Assignments are sticky without per-user data.
- **Personalization uses a per-user Durable Object.** It stores click and page-view counts. Rules are re-evaluated on every request by default, and can be latched per rule.
- **Consent gates all guest state.** Without consent there is no cookie, no Durable Object write, and no event tracking.
- **Rule changes take about 60 seconds to reach every edge.** Rollbacks are a new version with the treatment set to control.
- **Publishing is reliable.** Postgres is the source of truth, with an outbox table and a reconciliation job so KV eventually matches it.
- **Security is layered.** WAF rate limits and bot scoring, origin checks on event intake, and HMAC-signed guest cookies with rotation.

## Status

Design phase. No code has been written yet. Open items are listed in the spec (section 20) and the findings document.
