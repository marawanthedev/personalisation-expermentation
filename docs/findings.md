# Findings and Decisions

This document records what we learned while designing the platform, and why each decision was made. The [tech spec](tech-spec.md) is the authoritative design. This file explains the reasoning behind it.

## 1. Platform facts we verified

| Topic | Finding | Source |
|---|---|---|
| KV propagation | Edges cache reads. Minimum `cacheTtl` is 60 seconds. There is no purge or invalidate API, so changes can take up to about a minute to reach every edge. | Cloudflare KV docs |
| KV writes from outside Workers | Possible through the Cloudflare REST API, so the Node service can publish without a Worker in between. | Cloudflare API |
| Durable Object storage | SQLite-backed objects can store up to 10 GB each on the Paid plan. Per-user records are a few hundred bytes, so this is not a practical limit. | [Limits](https://developers.cloudflare.com/durable-objects/platform/limits/) |
| Free plan | Durable Objects are available on the free plan with SQLite storage. Account storage is capped at 5 GB, and there are daily request and row limits. | [Pricing](https://developers.cloudflare.com/durable-objects/platform/pricing/) |
| Durable Object concurrency | Each object processes one request at a time. A single hot object, such as a global flag read on every request, limits throughput. | Cloudflare Durable Objects model |

The free-plan daily limits are recalled rather than checked against the pricing page this session. Confirm them before budgeting.

## 2. Why KV alone is not enough for event-driven consistency

KV cannot give event-based consistency. Its minimum cache is 60 seconds and it has no invalidation, so no amount of publishing can make edges see a change instantly. Two options were considered:

- **Durable Objects as the source of truth for rules.** Gives immediate consistency but adds a hop to every request and makes one object the bottleneck.
- **KV for rules, accepting about 60 seconds of delay.** Chosen as the default. An optional kill switch covers the rules that cannot wait.

Decision: KV holds the rule set. Durable Objects hold per-user state only.

## 3. Experiments: hashing instead of storage

Storing each user's assignment would need a write per user and a lookup on every request. Deterministic hashing gives the same answer with no storage:

- `bucket = hash(salt + ":" + id) mod 10000`
- Changing the percentages moves only the users whose buckets fall in the changed ranges.
- A fixed salt per experiment keeps assignments independent across experiments.

The hash must be pinned in code, and test vectors guard against accidental changes, because changing the hash reshuffles every user.

Decision: experiments are stateless. Logged-in users are bucketed by account ID. Guests are bucketed by their cookie ID.

## 4. Personalization: state, and when stickiness is possible

Behavioral rules ("viewed the shop page more than five times", "clicked item X in the last 24 hours") depend on counts that change. We concluded that a rule can be sticky only if its decision is stored:

| Option | Sticky | Storage |
|---|---|---|
| Stateless hash | Yes | None |
| Re-evaluated against current counts | No | Counts |
| Latched (decision stored on first match) | Yes | Counts and a latch |

Decision: personalization rules re-evaluate by default. Latching is opt-in per rule, and the design for it is in spec section 10A.

Rolling windows are computed from timestamped events, not by clearing data on a schedule. Clearing everyone's data every 24 hours resets all users at the same moment and breaks any rule that needs history.

## 5. Cookies and identity

- Guests get a random ID in a signed cookie. Logged-in users are identified by account ID and do not need the guest cookie.
- The cookie is reissued with the same ID on each visit, so the expiry slides. A guest who loses the cookie gets a new identity.
- `SameSite=Lax`, not `Strict`. `Strict` drops the cookie on navigation from external links, so returning users would look like new guests.
- The cookie is HMAC-signed so users cannot forge IDs and read another user's state. Signing does not stop someone who copies a valid cookie.
- Browser protections, such as Safari's tracking prevention, may shorten cookie lifetimes. Plan for guest state to be lost sooner than 30 days.

## 6. Consent

- Setting a random ID cookie and storing behavioral state on a user's device needs consent under EU ePrivacy rules, unless it is strictly necessary. Behavioral personalization is generally not strictly necessary.
- The random ID is still personal data under GDPR when it is tied to behavior, so it needs a lawful basis, disclosure, and erasure support.
- Malaysia's PDPA has similar consent principles.
- Decision: one consent covers experiments and personalization for guests. Without it, guests get control and nothing is stored.

This is engineering analysis, not legal advice. Privacy review is required before launch.

## 7. Guest to account linking

- Guest state and account state live in different Durable Objects, so merging them is not atomic.
- Decision: a guest merges into an account once, at first login. A guest ID links to one account only, so a shared computer cannot leak one person's history into another account.
- The merge is idempotent. Copying happens only while `merged_at` is null, and the copy skips events that already exist.
- Login can switch an experiment treatment, because the bucket changes from guest ID to account ID. This is accepted.

## 8. Publishing reliability

Writing to Postgres and pushing to KV are two separate writes that cannot be made atomic. Without protection, a crash between them leaves Postgres and KV disagreeing.

Decision: an outbox table records each publish in the same transaction as the version. A background job retries pending pushes, and a reconciliation job compares the latest Postgres version with KV to catch anything the outbox missed, such as a manual edit in the dashboard.

## 9. Security layers

- **WAF.** Rate limits and bot scoring on the endpoints an attacker could flood.
- **Origin check on event intake.** Stops third-party pages from sending events for your visitors.
- **Same-origin APIs.** Avoids cross-origin credentials and the CORS configuration they require.
- **Fixed condition grammar.** Rules are data, never code, so the Worker never evaluates arbitrary expressions.
- **Secrets stay server-side.** Cookie keys and API tokens are never in the frontend or in KV.

## 10. Earlier design corrections

An earlier masterplan proposed several things we changed:

- Storing behavioral profiles in KV. KV is eventually consistent with no atomic increment, so concurrent clicks would overwrite each other. Counters moved to Durable Objects.
- `SameSite=Strict` cookies. Changed to `Lax` (see section 5).
- A cache purge webhook on content updates. Not needed, because personalized responses are never cached and rule changes reach edges through the KV TTL.
- A claim of sub-millisecond KV lookups. Removed. Cold reads and Durable Object hops add latency, and no figure should be promised until it is measured.
- Image and media storage through S3. Out of scope for this platform.

## 11. Reference points

Contentful's personalization engine, as described in a Cloudflare Connect 2025 session abstract, uses Workers, Durable Objects, and KV together, with KV holding personalization rules and Durable Objects holding user state. This matches the split in this design. The abstract is a summary, not a technical write-up, so the exact data flow is not confirmed.

## 12. Build versus buy

A split harness has two parts of very different difficulty:

- **Assignment** is easy. It is roughly the code in section 11.1 of the spec, plus publishing.
- **Analysis** is hard. Exposure logs must be trustworthy, metrics must be defined and joined correctly, statistical methods must be right, and sample ratio mismatches must be caught.

Rough estimate for an internal tool, assuming existing auth and a data warehouse:

| Tier | Scope | Effort (2 to 3 engineers) |
|---|---|---|
| 1 | Assignment, edge serving, publishing, basic control panel, exposure logging | About 2 to 3 months |
| 2 | Metrics, significance, sample ratio checks, results dashboard | About 2 to 3 more months, plus a data person |
| 3 | SDKs for web, mobile, and server, QA overrides, approvals, audit, alerting | About 3 to 6 more months, 3 to 4 engineers |

Recommendation: build tier 1 as part of this platform. For analysis, self-host an existing tool or use the warehouse, rather than building statistics from scratch.

## 13. Open items

1. Retention periods for guest state, events, latches, exposure records, and erasure deadlines.
2. Whether logged-in users without consent can be assigned to experiments by account ID.
3. Overlap rules between experiments.
4. Default latch TTL, and whether a latch reset should be per rule or global.
5. Whether any rule needs the instant kill switch.
6. Confirmation of the free-plan limits, and the Paid plan budget.
7. Privacy review of the consent and profiling approach before launch.
