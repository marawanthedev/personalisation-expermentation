# Personalization Platform: Technical Specification

| Field | Value |
|---|---|
| Status | Draft |
| Owner | TBD |
| Last updated | 2026-10-10 |
| Scope | Experimentation, personalization, event tracking, and the publishing pipeline |
| Out of scope | Image and media storage (S3/CDN), adaptive or bandit allocation |

## TL;DR

- **Two flows, one rule set.** Experiments and personalization both read one rule set from Cloudflare KV, which the Node service publishes from Postgres.
- **Experiments use hashing, not storage.** Each user is bucketed with `hash(salt + userId) mod 10000`. Logged-in users use their account ID, and guests use a signed cookie ID. Assignments are sticky without any per-user data.
- **Personalization uses a per-user Durable Object.** It stores click and page-view counts. Rules are re-evaluated on every request by default, and can be latched per rule (section 10A).
- **Consent gates all guest state.** Without consent there is no cookie, no Durable Object write, and no event tracking. Guests get control.
- **Rule changes take about 60 seconds to reach every edge.** Rollbacks are a new version with the treatment set to control. An optional kill switch is available for rules that can't wait.
- **Publishing is reliable.** Postgres is the source of truth, with an outbox table and a reconciliation job so KV eventually matches it.
- **Security is layered.** The WAF handles rate limits and bots, `/api/track` checks the origin, and the guest cookie is HMAC-signed with rotation support.
- **Open items before launch:** retention periods, whether logged-in users without consent can be bucketed, and the guest-to-account merge procedure.

## 1. Summary

The platform delivers per-user content from the edge using two independent mechanisms:

- **Experimentation** assigns users to treatments by hashing a stable identifier with an experiment-specific salt. It stores nothing per user, so stickiness comes from the math. Logged-in users are bucketed by account ID, and guests by a signed cookie ID.
- **Personalization** changes content based on what a user has done, such as clicks and page views. It reads per-user state from a Durable Object and re-evaluates rules on every request, so it is not sticky by default.

Both read one rule set from Cloudflare KV. A Node.js service backed by PostgreSQL is the only writer to KV and the source of truth for every rule version. Publishing uses an outbox table and a reconciliation job, so KV eventually matches Postgres even if a push fails.

Guest state is created and used only with consent. When a guest logs in, their guest state is merged into the account once. Rule changes reach edges within about 60 seconds.

## 2. Goals and non-goals

**Goals**

- Serve experiment treatments and personalized content from the edge, without a round trip to origin.
- Keep experiment assignments stable for each user, without storing per-user data.
- Record behavioral events from the browser and use them in later requests.
- Make rule changes and rollbacks take effect within about 60 seconds.
- Require consent before writing any identifier or behavioral state.

**Non-goals**

- Instant (sub-second) rule propagation. The optional kill switch (section 13) is the only exception.
- Storing or serving images through this platform.
- Adaptive or bandit allocation.
- Cross-device tracking for guests.

## 3. Glossary

| Term | Meaning |
|---|---|
| Experiment | A split of users into treatments, assigned by hashing. |
| Treatment | A variant a user is shown, such as `X` or `control`. |
| Bucket | A number from 0 to 9999 derived from a salt and a user ID. |
| Salt | A per-experiment random string, fixed for the life of the experiment. |
| Range | A span of buckets mapped to one treatment. |
| Personalization rule | A condition over behavioral counts that selects content. |
| Re-evaluated | Checked against current state on every request. The default for personalization rules. |
| Latched | Decision stored once it first matches, and kept until a limit is reached (section 10A). |
| Guest ID | A signed random ID stored in the `uid` cookie for users who are not logged in. |
| Account ID | The ID of a logged-in user. |
| Rule set | All active experiments and personalization rules, with a version number. Stored as one KV key. |
| Consent | The user's choice, recorded by the consent management platform (CMP). |

## 4. Architecture

![Architecture diagram](architecture/architecture.png)

```text
 Control panel (Next.js)
        |  save rule
        v
 Node.js + PostgreSQL            <- source of truth, audit log, outbox
        |  publish rule set
        v
 Cloudflare KV: rules:current    <- experiments, personalization rules, version
        |
        v
 Cloudflare WAF                  <- rate limits, bot scoring, managed rules
        |
        v
 Edge Worker                     <- consent, identity, hashing, evaluation, cookies
        |              |
        |              v
        |       User Durable Object   <- per-user counters and events (personalization only)
        v
 Browser (Next.js + SDK)         <- renders content, sends events
```

Page shells are the same for every visitor and can be cached. Personalized responses are per user and are never cached.

## 5. Components

| Component | Responsibilities | Does not |
|---|---|---|
| Control panel (Next.js) | Create and edit experiments and personalization rules. | Talk to KV or any Durable Object directly. |
| Node.js + PostgreSQL service | Validate rules, store versions and audit logs, publish to KV, retry failed publishes, reconcile Postgres with KV. | Serve traffic. |
| KV (`rules:current`) | Hold the rule set and its version. | Hold any per-user data. |
| Edge Worker | Handle identity, consent, cookies, hashing, evaluation, event intake, exposure logging, and fallbacks. | Persist anything itself. |
| User Durable Object | Store counters, timestamped events, and latches for one user, with atomic writes. | Hold experiment assignments. |
| SDK (browser) | Send tracking events to `/api/track` and fetch content from `/api/personalize`. | Decide treatments. |
| WAF | Rate limiting, bot scoring, managed rules, request filtering. | Business logic. |

## 6. Data model

### 6.1 PostgreSQL

```sql
CREATE TABLE experiments (
  id          TEXT PRIMARY KEY,
  salt        TEXT NOT NULL,            -- generated once, never updated
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE rule_versions (
  version     INT PRIMARY KEY,
  payload     JSONB NOT NULL,           -- full rule set as published
  changed_by  TEXT NOT NULL,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE publish_outbox (
  id          BIGSERIAL PRIMARY KEY,
  version     INT NOT NULL REFERENCES rule_versions(version),
  status      TEXT NOT NULL DEFAULT 'pending',   -- pending | sent | failed
  attempts    INT NOT NULL DEFAULT 0,
  last_error  TEXT,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE guest_links (
  guest_id    TEXT PRIMARY KEY,
  account_id  TEXT NOT NULL,
  linked_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
  merged_at   TIMESTAMPTZ               -- set once guest state has been copied
);
```

Add a database trigger or constraint that rejects updates to `experiments.salt`.

### 6.2 KV rule set (`rules:current`)

```json
{
  "version": 12,
  "latchEpoch": 1,
  "trackedEvents": ["page_view:shop", "click:pants", "click:apply_auto_loan"],
  "experiments": [
    {
      "id": "checkout-banner",
      "salt": "a81f0c...",
      "ranges": [
        { "from": 0,    "to": 6999, "treatment": "X" },
        { "from": 7000, "to": 9999, "treatment": "control" }
      ]
    }
  ],
  "personalization": [
    {
      "id": "hero-auto",
      "stickiness": "reevaluate",
      "when": {
        "all": [
          { "event": "click:apply_auto_loan", "window_ms": 86400000, "op": "gte", "value": 1 }
        ]
      },
      "serve": "auto_financing_hero"
    }
  ],
  "content": {
    "auto_financing_hero": {
      "title": "Looking for premium auto financing?",
      "description": "Get a decision in minutes.",
      "classes": "bg-amber-600 text-slate-950 font-black p-8 rounded-xl"
    }
  }
}
```

Ranges for an experiment must cover buckets 0 to 9999 exactly once. Control is an explicit range.

### 6.3 Durable Object storage (per user)

A SQLite-backed Durable Object with two tables:

```sql
CREATE TABLE IF NOT EXISTS events (
  name TEXT NOT NULL,       -- for example click:pants
  at   INTEGER NOT NULL     -- epoch milliseconds
);
CREATE INDEX IF NOT EXISTS events_name_at ON events (name, at);

CREATE TABLE IF NOT EXISTS latches (
  rule_id    TEXT PRIMARY KEY,
  serve      TEXT NOT NULL,      -- content key, resolved against rules.content at read time
  epoch      INTEGER NOT NULL,
  latched_at INTEGER NOT NULL,
  ttl_ms     INTEGER             -- NULL means no expiry
);
```

Events older than the retention limit (section 14) are deleted on write. The `latches` table is only populated for rules that use latching (section 10A).

## 7. Rule definitions

### 7.1 Experiments

- `id` is unique and never reused.
- `salt` is generated by the Node service when the experiment is created and never changes.
- `ranges` must cover 0 to 9999 exactly, with no gaps or overlaps.
- Each treatment must be in the experiment's known treatment list.

### 7.2 Personalization rules

| Field | Allowed values |
|---|---|
| `stickiness` | `reevaluate` (default) or `latch`. See section 10A. |
| `latch_ttl_ms` | Optional. Only with `latch`. A latch expires after this many milliseconds. |
| `when.all` | List of clauses. All must match. |
| `event` | An entry in `trackedEvents`. |
| `window_ms` | Greater than 0 and at most the retention limit. |
| `op` | `gt`, `gte`, `lt`, `lte`, `eq`. |
| `value` | Number. |
| `serve` | A key in `content`. |

Conditions use this fixed grammar. They are never evaluated as code. A clause counts events of one name within the window. A rule matches when all its clauses match. Rules are evaluated in order, and the first rule that resolves wins.

## 8. Identity

| Visitor | Identity | Cookie | Notes |
|---|---|---|---|
| Logged in | `acct:<accountId>` | Login session only | No `uid` cookie. Used for experiments and personalization. |
| Guest with consent | `guest:<guestId>` | Signed `uid` cookie | Created on first consent. Reissued with the same ID on each visit. |
| Guest without consent | None | None | Gets control for experiments and personalization. |

Guest IDs survive login and logout. After login, the guest ID is linked to the account so personalization state can be merged (section 8.3).

### 8.1 Cookie format

Value: `<guestId>.<hmacHex>`. Attributes:

```text
uid=<value>; Path=/; Max-Age=2592000; HttpOnly; Secure; SameSite=Lax
```

`SameSite=Lax` is used instead of `Strict`. With `Strict`, the browser withholds the cookie on top-level navigation from another site, such as a link in an email, so a returning user would be treated as a new guest.

### 8.2 Cookie lifecycle

- Created when a guest with consent first reaches a route that needs identity.
- Reissued with the same ID and a fresh 30-day expiry on each visit that presents a valid cookie. This is a sliding window.
- Replaced with a new ID only if the cookie is missing or its signature does not verify.
- Deleted when consent is withdrawn or an erasure request is processed.
- Browsers, including Safari's tracking protection, may shorten the lifetime. Do not rely on the full 30 days.

### 8.3 Guest to account linking

When a guest with a valid cookie logs in, the guest state is merged into the account once.

1. **Record the link.** Insert a row into `guest_links` (section 6.1). The guest ID comes from the cookie, after signature verification. A guest ID can be linked to one account only. The first link wins.
2. **Copy events.** Read all events from `guest:<guestId>` and insert them into `acct:<accountId>`, keeping their timestamps. Copy latches too, but keep the account's latch if it already has one for the same rule.
3. **Mark merged.** Set `merged_at` in Postgres.
4. **Clear the guest object.** Delete the guest Durable Object's data.

The steps are not atomic across two Durable Objects, so the order matters. If the process fails after step 2, the retry must not copy events twice. The merge runs only when `merged_at` is null, and the copy step skips events already present in the account object. Step 3 is the point of no return: once it is set, the guest object is never merged again.

After `merged_at` is set, events for that guest ID are written to the account object instead of the guest object, so nothing is lost in the gap.

Merge rules:

- Merge only with consent. Without consent there is no link.
- Logout keeps the guest ID. A later login on the same device does not merge again.
- If a second account logs in on the same device, it does not receive the guest's history.
- Guests on other devices are never linked.

## 9. Consent

One consent covers experiments and personalization for guests.

- Without consent, the Worker sets no cookie, creates no ID, writes no Durable Object state, and records no events. Guests receive control.
- Logged-in users without consent: experiments are assigned by account ID, since no device storage is involved. Personalization is disabled. Confirm this with the privacy team.
- The CMP stores the consent decision. The Worker reads it from a first-party consent cookie set by the CMP.
- Withdrawing consent deletes the `uid` cookie and the guest's Durable Object.

## 10. Flows

### 10.1 Experiment assignment

1. Worker resolves identity (section 8). If none, return control.
2. Read `rules:current` from KV with `cacheTtl: 60`.
3. For each experiment, compute `bucket = hash(salt + ":" + identityId) mod 10000`.
4. Return the treatment whose range contains the bucket.
5. Record an exposure event through `ctx.waitUntil`.
6. No storage writes.

### 10.2 Personalization

1. Worker checks consent. Without it, return control.
2. Resolve identity.
3. Make one call to the user's Durable Object for the counts and latches the rules need (section 10A).
4. Evaluate personalization rules in order. The first rule that resolves selects content.
5. Return content and experiment treatments. Record an exposure event through `ctx.waitUntil`, with the source (`latch` or `evaluated`).

### 10.3 Event tracking

1. The SDK listens for interactions and calls `navigator.sendBeacon("/api/track", ...)`.
2. The WAF rate-limits the request and checks body size and content type.
3. Worker checks the `Origin` header against the site origin.
4. Worker checks that the event name is in `trackedEvents`.
5. Worker checks consent and identity. Without either, return `204` and drop the event.
6. Worker writes the event to the user's Durable Object through `ctx.waitUntil`, and returns `202`.

### 10.4 Publishing

1. The control panel sends the change to the Node service.
2. The service validates the rule set (section 7).
3. In one database transaction, insert a `rule_versions` row and a `publish_outbox` row.
4. Write the rule set to KV under `rules:current`.
5. Mark the outbox row `sent`. On failure, leave it `pending`, increment `attempts`, and record `last_error`.
6. A background job retries `pending` rows and compares the latest Postgres version with the version in KV.

The outbox row is what makes the two writes safe. If the process crashes between the database commit and the KV push, the pending row survives and the job retries the push.

### 10.5 Distribution change (30% to 40%)

The service changes the ranges, bumps the version, and publishes. Buckets in the newly added range move to the new treatment. Other buckets keep their treatment. No per-user work is needed.

### 10.6 Rollback

The service sets the treatment range to control and publishes. Edges pick it up within about 60 seconds.

### 10.7 Guest to login

The guest is linked and merged as described in section 8.3. Experiment buckets use the account ID after login, so a user can switch treatment at login. This is accepted and documented (D4).

## 10A. Stickiness model

Personalization rules depend on engagement data, and that data changes. A rule can therefore be sticky only if its decision is stored. The platform supports three models, and each rule chooses one.

| Model | Depends on engagement data | Sticky | Storage | Use for |
|---|---|---|---|---|
| Stateless hash (experiments) | No | Yes, by math | None | A/B splits, percentage rollouts |
| Re-evaluated (default for personalization) | Yes | No, follows current data | Per-user counts | Content that should reflect what the user is doing now |
| Latched (opt-in per rule) | Yes, at first match only | Yes, by stored decision | Counts plus a latch record | Content that should not flip once shown |

### 10A.1 Rule definition

A latched personalization rule sets `stickiness` to `latch`, with an optional TTL:

```json
{
  "id": "hero-auto",
  "stickiness": "latch",
  "latch_ttl_ms": 86400000,
  "when": {
    "all": [
      { "event": "click:apply_auto_loan", "window_ms": 86400000, "op": "gte", "value": 1 }
    ]
  },
  "serve": "auto_financing_hero"
}
```

- `stickiness`: `"reevaluate"` (default) or `"latch"`.
- `latch_ttl_ms`: optional. When set, a latch expires after this long and the rule is evaluated again. When omitted, the latch lasts until the epoch changes or the user's data is deleted.

The rule set has a `latchEpoch` number. Bumping it invalidates every latch for every rule. A latch written under an older epoch is ignored and replaced on the next match.

### 10A.2 Latch storage

The `latches` table in section 6.3 holds one row per latched rule per user. The latch stores the content key, not the content, so edits to content apply to latched users immediately.

### 10A.3 Decision flow

For each request on `/api/personalize`:

1. The Worker sends one call to the user's Durable Object with the counts it needs, the current epoch, and the list of latched rules with their TTLs. The Durable Object returns the counts and any valid latches. A latch is valid if its epoch matches and it has not expired.
2. The Worker walks the personalization rules in order:
   - If the rule is latched and a valid latch exists, serve the latched content key.
   - If the rule's conditions match the counts, serve its content key. For a latched rule, store the latch first.
3. The first rule that resolves wins. Stop there.

Storing a latch is a second Durable Object call. It happens only the first time a latched rule fires for a user.

### 10A.4 Worker logic

```js
async function decide(env, identityId, rules) {
  const latchRules = rules.personalization
    .filter(r => r.stickiness === "latch")
    .map(r => ({ id: r.id, ttlMs: r.latch_ttl_ms ?? null }));

  const res = await userState(env, identityId).fetch("https://user-state/decide", {
    method: "POST",
    body: JSON.stringify({
      queries: collectQueries(rules.personalization),
      epoch: rules.latchEpoch,
      latchRules,
    }),
  });
  return res.json(); // { counts, latches: { ruleId: serve } }
}

async function storeLatch(env, identityId, rule, epoch) {
  const res = await userState(env, identityId).fetch("https://user-state/latch", {
    method: "POST",
    body: JSON.stringify({
      ruleId: rule.id,
      serve: rule.serve,
      epoch,
      ttlMs: rule.latch_ttl_ms ?? null,
    }),
  });
  return (await res.json()).serve; // the stored latch, even if a concurrent request wrote first
}

async function choosePersonalized(env, identityId, rules) {
  const { counts, latches } = await decide(env, identityId, rules);

  for (const rule of rules.personalization) {
    const latched = rule.stickiness === "latch";
    if (latched && latches[rule.id]) return { key: latches[rule.id], source: "latch" };

    const matched = rule.when.all.every(clause => {
      const count = counts[`${clause.event}|${clause.window_ms}`] ?? 0;
      return OPS[clause.op](count, clause.value);
    });
    if (!matched) continue;

    if (!latched) return { key: rule.serve, source: "evaluated" };
    const key = await storeLatch(env, identityId, rule, rules.latchEpoch);
    return { key, source: "latch" };
  }
  return { key: null, source: null };
}
```

### 10A.5 Durable Object endpoints

```js
if (pathname === "/decide") {
  const now = Date.now();
  const latches = {};
  for (const { id, ttlMs } of body.latchRules) {
    const row = sql.exec(
      "SELECT serve, epoch, latched_at, ttl_ms FROM latches WHERE rule_id = ?", id
    ).toArray()[0];
    if (row && row.epoch === body.epoch && (row.ttl_ms === null || row.latched_at + row.ttl_ms > now)) {
      latches[id] = row.serve;
    }
  }
  const counts = readCounts(sql, body.queries, now);
  return Response.json({ counts, latches });
}

if (pathname === "/latch") {
  const now = Date.now();
  sql.exec(
    `DELETE FROM latches WHERE rule_id = ? AND (epoch != ? OR (ttl_ms IS NOT NULL AND latched_at + ttl_ms <= ?))`,
    body.ruleId, body.epoch, now
  );
  sql.exec(
    `INSERT OR IGNORE INTO latches (rule_id, serve, epoch, latched_at, ttl_ms) VALUES (?, ?, ?, ?, ?)`,
    body.ruleId, body.serve, body.epoch, now, body.ttlMs
  );
  const row = sql.exec("SELECT serve FROM latches WHERE rule_id = ?", body.ruleId).one();
  return Response.json({ serve: row.serve });
}
```

`readCounts` is the counting query used by `/read` in section 11.5, factored into a helper. The Durable Object processes one request at a time, so the delete, insert, and select in `/latch` cannot interleave with another request.

### 10A.6 Behavior summary

- **Stable for the latch's lifetime.** A latched user keeps the same content key while the latch is valid, even if their counts drop below the threshold.
- **Priority is by rule order.** A higher-priority rule that starts matching overrides a lower-priority latch. This is intentional. If a latch must win over higher-priority rules, move the rule up or use a separate experiment.
- **Latch lifetime is bounded by epoch, TTL, or deletion.** Any of the three ends it.

### 10A.7 Interaction with other parts of the design

| Area | Behavior |
|---|---|
| Re-evaluated rules | Unchanged. They never read or write latches. |
| Experiments | Unchanged. Latching applies only to personalization rules. |
| Kill switch (section 13) | Returns control without reading latches. Latches are left intact and resume when the switch is cleared. |
| Consent withdrawn | The Durable Object is deleted, so latches are deleted with it. |
| Guest to account merge (section 8.3) | Latches are copied with the events. If the account already has a latch for a rule, the account's latch is kept. |
| Rule removed | The rule is no longer in the rule set, so its latch is never read. The row is removed by retention. |
| Content edited | Latched users see the edited content, since the latch stores only the content key. |
| Content key renamed or removed | Latched users get no content for that key until the next evaluation. Do not rename a content key while latches exist. Publish validation should reject this. |
| Concurrent first match | The Durable Object serializes writes. The first stored latch wins, and the other request serves that stored value. |
| Exposure logs | Record `source` as `latch` or `evaluated`, so results can be analyzed by model. |
| Storage | One row per latched rule per user. Rows past their TTL are removed by the same cleanup job as other retention data. |

### 10A.8 Testing

- A latched rule keeps its content key after its counts drop below the threshold.
- Bumping `latchEpoch` causes the next match to store a new latch.
- A latch with a TTL is ignored after expiry and replaced on the next match.
- Two concurrent first matches store one latch, and both requests serve that value.
- A higher-priority re-evaluated rule overrides a lower-priority latch.
- Withdrawing consent removes all latches for that user.
- Merging a guest into an account carries the latches over, and an existing account latch is kept.

### 10A.9 Open decisions

1. **Default TTL for latched rules.** Forever until epoch change, or a fixed default such as 30 days.
2. **Per-rule epoch.** Currently one epoch invalidates every latch. Decide whether a per-rule reset is needed.
3. **Latched rules inside experiments.** Confirm that a latched personalization rule should never change an experiment's assignment.

## 11. Code

### 11.1 Worker entry point (`edge-decision-engine/src/index.js`)

```js
const BUCKETS = 10000;
const GUEST_COOKIE_MAX_AGE = 60 * 60 * 24 * 30;

export default {
  async fetch(request, env, ctx) {
    const { pathname } = new URL(request.url);
    if (pathname === "/api/personalize" && request.method === "GET") {
      return handlePersonalize(request, env, ctx);
    }
    if (pathname === "/api/track" && request.method === "POST") {
      return handleTrack(request, env, ctx);
    }
    return fetch(request); // page shells are served from origin
  },
};

async function loadRules(env) {
  try {
    return await env.RULES_KV.get("rules:current", { type: "json", cacheTtl: 60 });
  } catch {
    return null;
  }
}

function fnv1a(input) {
  let hash = 0x811c9dc5;
  for (let i = 0; i < input.length; i++) {
    hash ^= input.charCodeAt(i);
    hash = Math.imul(hash, 0x01000193) >>> 0;
  }
  return hash >>> 0;
}

function bucketFor(salt, identityId) {
  return fnv1a(`${salt}:${identityId}`) % BUCKETS;
}

function evaluateExperiment(experiment, identityId) {
  const bucket = bucketFor(experiment.salt, identityId);
  const range = experiment.ranges.find(r => bucket >= r.from && bucket <= r.to);
  return range ? range.treatment : "control";
}

function userState(env, identityId) {
  return env.USER_STATE.get(env.USER_STATE.idFromName(identityId));
}
```

The hash is FNV-1a, pinned in code. Test vectors are generated from this implementation and checked in CI, so an accidental change fails the build. The hash input is ASCII-only, since identity IDs are UUIDs with prefixes.

### 11.2 Identity and cookies

```js
async function hmacHex(value, secret) {
  const key = await crypto.subtle.importKey(
    "raw", new TextEncoder().encode(secret),
    { name: "HMAC", hash: "SHA-256" }, false, ["sign"]
  );
  const sig = await crypto.subtle.sign("HMAC", key, new TextEncoder().encode(value));
  return [...new Uint8Array(sig)].map(b => b.toString(16).padStart(2, "0")).join("");
}

function safeEqual(a, b) {
  if (a.length !== b.length) return false;
  let diff = 0;
  for (let i = 0; i < a.length; i++) diff |= a.charCodeAt(i) ^ b.charCodeAt(i);
  return diff === 0;
}

async function verifyGuestCookie(raw, env) {
  const dot = raw.lastIndexOf(".");
  if (dot < 0) return null;
  const id = raw.slice(0, dot);
  const sig = raw.slice(dot + 1);
  const secrets = [env.COOKIE_SECRET_CURRENT, env.COOKIE_SECRET_PREVIOUS].filter(Boolean);
  for (const secret of secrets) {
    if (safeEqual(sig, await hmacHex(id, secret))) return id;
  }
  return null;
}

async function guestCookieHeader(id, env) {
  const value = `${id}.${await hmacHex(id, env.COOKIE_SECRET_CURRENT)}`;
  return `uid=${value}; Path=/; Max-Age=${GUEST_COOKIE_MAX_AGE}; HttpOnly; Secure; SameSite=Lax`;
}

function parseCookies(header) {
  return Object.fromEntries(
    (header || "").split(";").map(c => c.trim()).filter(Boolean).map(c => {
      const i = c.indexOf("=");
      return [c.slice(0, i), c.slice(i + 1)];
    })
  );
}

async function hasConsent(request) {
  return parseCookies(request.headers.get("Cookie")).consent === "granted";
}

async function resolveIdentity(request, env) {
  const accountId = await getAccountId(request, env); // validates your session
  if (accountId) return { id: `acct:${accountId}`, setCookie: null };
  if (!(await hasConsent(request))) return null;

  const cookies = parseCookies(request.headers.get("Cookie"));
  const existing = cookies.uid ? await verifyGuestCookie(cookies.uid, env) : null;
  const guestId = existing ?? crypto.randomUUID();
  return {
    id: `guest:${guestId}`,
    setCookie: await guestCookieHeader(guestId, env),
  };
}
```

The consent cookie check is a placeholder for the CMP integration. `getAccountId` is the existing session validation.

### 11.3 Personalization

```js
const OPS = {
  gt: (a, b) => a > b,
  gte: (a, b) => a >= b,
  lt: (a, b) => a < b,
  lte: (a, b) => a <= b,
  eq: (a, b) => a === b,
};

function collectQueries(rules) {
  const seen = new Map();
  for (const rule of rules) {
    for (const clause of rule.when.all) {
      const key = `${clause.event}|${clause.window_ms}`;
      if (!seen.has(key)) seen.set(key, { event: clause.event, windowMs: clause.window_ms });
    }
  }
  return [...seen.values()];
}

async function handlePersonalize(request, env, ctx) {
  const noStore = { "Cache-Control": "private, no-store" };
  const rules = await loadRules(env);
  if (!rules) return Response.json({ experiments: {}, content: null }, { headers: noStore });

  const identity = await resolveIdentity(request, env);
  const experiments = Object.fromEntries(
    rules.experiments.map(exp => [exp.id, identity ? evaluateExperiment(exp, identity.id) : "control"])
  );

  let choice = { key: null, source: null };
  if (identity && (await hasConsent(request)) && rules.personalization.length) {
    try {
      choice = await choosePersonalized(env, identity.id, rules);
    } catch {
      choice = { key: null, source: null };
    }
  }
  const content = choice.key ? rules.content[choice.key] : null;

  ctx.waitUntil(env.EXPOSURES.send({
    at: Date.now(),
    ruleVersion: rules.version,
    user: identity?.id ?? null,
    experiments,
    content: choice.key,
    source: choice.source,
  }));

  const headers = { ...noStore };
  if (identity?.setCookie) headers["Set-Cookie"] = identity.setCookie;
  return Response.json({ experiments, content }, { headers });
}
```

`choosePersonalized` and the Durable Object calls it uses are defined in section 10A. The exposure record's `user` field should be pseudonymized before it reaches analytics (section 14).

### 11.4 Event intake

```js
const MAX_TRACK_BYTES = 1024;

async function handleTrack(request, env, ctx) {
  if (request.headers.get("Origin") !== env.SITE_ORIGIN) {
    return new Response(null, { status: 403 });
  }
  if (!request.headers.get("Content-Type")?.startsWith("application/json")) {
    return new Response(null, { status: 415 });
  }
  if (Number(request.headers.get("Content-Length") ?? 0) > MAX_TRACK_BYTES) {
    return new Response(null, { status: 413 });
  }

  const rules = await loadRules(env);
  const body = await request.json().catch(() => null);
  if (!rules || !body || !rules.trackedEvents.includes(body.name)) {
    return new Response(null, { status: 400 });
  }

  const identity = await resolveIdentity(request, env);
  if (!identity || !(await hasConsent(request))) {
    return new Response(null, { status: 204 });
  }

  ctx.waitUntil(userState(env, identity.id).fetch("https://user-state/track", {
    method: "POST",
    body: JSON.stringify({ name: body.name, at: Date.now() }),
  }));

  const headers = {};
  if (identity.setCookie) headers["Set-Cookie"] = identity.setCookie;
  return new Response(null, { status: 202, headers });
}
```

### 11.5 Durable Object (`edge-decision-engine/src/user-state.js`)

```js
const RETENTION_MS = 7 * 24 * 60 * 60 * 1000;

export class UserState {
  constructor(state) {
    this.state = state;
    state.blockConcurrencyWhile(async () => {
      state.storage.sql.exec(
        "CREATE TABLE IF NOT EXISTS events (name TEXT NOT NULL, at INTEGER NOT NULL)"
      );
      state.storage.sql.exec(
        "CREATE INDEX IF NOT EXISTS events_name_at ON events (name, at)"
      );
      state.storage.sql.exec(`CREATE TABLE IF NOT EXISTS latches (
        rule_id TEXT PRIMARY KEY,
        serve TEXT NOT NULL,
        epoch INTEGER NOT NULL,
        latched_at INTEGER NOT NULL,
        ttl_ms INTEGER
      )`);
    });
  }

  async fetch(request) {
    const { pathname } = new URL(request.url);
    const body = await request.json();
    const sql = this.state.storage.sql;

    if (pathname === "/track") {
      sql.exec("INSERT INTO events (name, at) VALUES (?, ?)", body.name, body.at);
      sql.exec("DELETE FROM events WHERE at < ?", Date.now() - RETENTION_MS);
      return new Response(null, { status: 204 });
    }

    if (pathname === "/read") {
      return Response.json(readCounts(sql, body.queries, Date.now()));
    }

    if (pathname === "/decide" || pathname === "/latch") {
      return this.decideOrLatch(pathname, body);
    }

    return new Response("not found", { status: 404 });
  }
}

function readCounts(sql, queries, now) {
  const result = {};
  for (const { event, windowMs } of queries) {
    const row = sql.exec(
      "SELECT COUNT(*) AS c FROM events WHERE name = ? AND at >= ?",
      event, now - windowMs
    ).one();
    result[`${event}|${windowMs}`] = row.c;
  }
  return result;
}
```

`decideOrLatch` holds the latch logic from section 10A.5. The Durable Object does not check the event allowlist. The Worker does that before calling it. `windowMs` must be validated against `RETENTION_MS` at publish time, so no query asks for data that has already been deleted.

### 11.6 Wrangler configuration (`edge-decision-engine/wrangler.json`)

```json
{
  "name": "edge-decision-engine",
  "main": "src/index.js",
  "compatibility_date": "2026-10-10",
  "kv_namespaces": [
    { "binding": "RULES_KV", "id": "<namespace-id>" }
  ],
  "durable_objects": {
    "bindings": [{ "name": "USER_STATE", "class_name": "UserState" }]
  },
  "migrations": [
    { "tag": "v1", "new_sqlite_classes": ["UserState"] }
  ],
  "queues": {
    "producers": [{ "binding": "EXPOSURES", "queue": "exposure-events" }]
  },
  "vars": {
    "SITE_ORIGIN": "https://example.com"
  }
}
```

Secrets are set with `wrangler secret put` and never committed:

- `COOKIE_SECRET_CURRENT`
- `COOKIE_SECRET_PREVIOUS` (optional, used during rotation)

### 11.7 Publishing from the Node service

```js
function validateExperiment(exp) {
  const ranges = [...exp.ranges].sort((a, b) => a.from - b.from);
  let next = 0;
  for (const r of ranges) {
    if (r.from !== next || r.to < r.from) {
      throw new Error(`experiment ${exp.id}: gap or overlap at bucket ${r.from}`);
    }
    next = r.to + 1;
  }
  if (next !== 10000) throw new Error(`experiment ${exp.id}: ranges must cover 0-9999`);
}

async function publishRules(db, rules, changedBy, cf) {
  rules.experiments.forEach(validateExperiment);

  const payload = await db.transaction(async tx => {
    const { rows: [{ next }] } = await tx.query(
      "SELECT COALESCE(MAX(version), 0) + 1 AS next FROM rule_versions"
    );
    const versioned = { ...rules, version: next };
    await tx.query(
      "INSERT INTO rule_versions (version, payload, changed_by) VALUES ($1, $2, $3)",
      [next, versioned, changedBy]
    );
    await tx.query("INSERT INTO publish_outbox (version) VALUES ($1)", [next]);
    return versioned;
  });

  await pushToKv(cf, payload);
  await db.query(
    "UPDATE publish_outbox SET status = 'sent' WHERE version = $1",
    [payload.version]
  );
}

async function pushToKv(cf, payload) {
  const url = `https://api.cloudflare.com/client/v4/accounts/${cf.accountId}/storage/kv/namespaces/${cf.namespaceId}/values/rules:current`;
  const res = await fetch(url, {
    method: "PUT",
    headers: { Authorization: `Bearer ${cf.apiToken}`, "Content-Type": "application/json" },
    body: JSON.stringify(payload),
  });
  if (!res.ok) throw new Error(`KV write failed: ${res.status}`);
}
```

The outbox job retries rows where `status = 'pending'`, and reconciles the latest Postgres version with the version the Worker reads from KV. Any mismatch raises an alert.

### 11.8 Browser SDK (`frontend-customer-app/lib/edge-sdk.js`)

```js
class EdgeSDK {
  track(name) {
    const blob = new Blob([JSON.stringify({ name })], { type: "application/json" });
    navigator.sendBeacon("/api/track", blob);
  }

  async personalize() {
    const res = await fetch("/api/personalize", { credentials: "same-origin" });
    return res.json();
  }
}

export const edgeSdk = new EdgeSDK();
```

Same-origin paths mean no CORS configuration is needed. `sendBeacon` does not block navigation. The SDK sends nothing until the CMP has recorded consent.

## 12. Security

- **WAF in front of every route.** Managed rules, bot scoring, and rate limits on `/api/track` and `/api/personalize`.
- **Origin check on `/api/track`.** Requests from other origins get `403`, so third-party pages cannot inject events.
- **Input limits on `/api/track`.** Content type, body size, and an allowlist of event names.
- **Signed guest cookie.** HMAC-SHA-256, verified with constant-time comparison, with rotation through `COOKIE_SECRET_PREVIOUS`.
- **Cookie attributes.** `HttpOnly`, `Secure`, `SameSite=Lax`.
- **No eval.** Conditions use the fixed grammar in section 7.2.
- **Secrets.** Stored as Worker secrets and in the Node service's secret store. Never in the frontend bundle or in KV.
- **Cloudflare API token.** Scoped to KV write access on one namespace, used only by the Node service.
- **Personal data in logs.** Exposure records use pseudonymous IDs. Raw account IDs are not sent to analytics.
- **Personalized responses.** Always `private, no-store`.

## 13. Optional: instant kill switch

Not part of the default design. Add it only for rules where a 60-second delay is unacceptable.

- A single global flag stored in a Durable Object.
- The Worker reads it first on every request. When set to killed, the Worker returns control without reading KV or any user Durable Object.
- Cost: one extra Durable Object read per request, and one hot object that every request touches. Cache the flag briefly in the Worker if traffic grows.
- Killing the flag deletes nothing. Clearing it restores previous behavior.

## 14. Retention and privacy

Open decision. The options below are not yet chosen.

| Data | Options to decide between |
|---|---|
| Guest Durable Object records | Expire after 30 days of inactivity (matches the cookie), or after 90 days. |
| Behavioral events | Keep only the longest window in use, with a hard cap set by `RETENTION_MS`. |
| Latches | Expire with the Durable Object, or sooner through `latch_ttl_ms`. |
| Exposure records | Keep for the analysis window plus a fixed period in analytics. |
| Erasure requests | Delete the Durable Object, clear the cookie, and remove exposure records, within a fixed deadline. |

Retention is enforced by a scheduled cleanup job or Durable Object alarms. Whatever is chosen must match the privacy notice.

## 15. Edge cases

| Case | Handling |
|---|---|
| Guest cookie missing or invalid | New ID created with consent. Old state is orphaned and removed by retention. |
| Guest declines consent | Control for experiments and personalization. Track the share of users who decline. |
| Salt changed | Reshuffles everyone. Blocked by the database trigger. |
| Hash function changed | Same effect as a salt change. Prevented by test vectors in CI. |
| Ranges with gaps or overlaps | Rejected at publish. |
| Bad rule JSON in KV | Worker returns control and logs the error. |
| KV unreachable | Fail open: control. |
| Durable Object unreachable | Personalization returns control. Experiments are unaffected. |
| Postgres written, KV push failed | Outbox retries. Reconciliation job alerts on mismatch. |
| Edges on different versions | Possible for up to about 60 seconds. Accepted. |
| Event written but Durable Object fails | Event is lost. Accepted for analytics signals. Do not use for billing or entitlements. |
| Same user in two experiments | Independent salts, so independent assignments. Overlap rules are a product decision. |
| Experiment percentages finer than 1% | 10,000 buckets, as specified. |
| Login switches treatment | Accepted and documented. |
| Guest merge interrupted | Merge runs only while `merged_at` is null. Copy skips events already present. |
| Bots inflating counts | Filtered at the WAF and by bot scoring before events are written. |
| Free plan limits | Development only. Production needs the Paid plan. |

## 16. Observability

- Log every fallback to control with its reason: consent, KV error, Durable Object error, bad rule, or missing experiment.
- Alert on fallback rate spikes. A silent fallback hides outages.
- Track publish latency and outbox backlog.
- Track Durable Object request volume and KV read volume against plan limits.
- Dashboards: treatment share per experiment compared with the configured split, consent acceptance rate, and latch versus evaluated exposure share.

## 17. Testing

- **Hash test vectors.** Fixed inputs and expected buckets, checked in CI. Any change to the hash fails the build.
- **Range validation.** Unit tests for gaps, overlaps, and full coverage.
- **Condition evaluator.** Tests for each operator and for unknown events.
- **Cookie signing.** Valid cookie, tampered cookie, previous-secret cookie, and missing cookie.
- **Durable Object.** Track and read within one window, pruning behavior, and concurrent increments.
- **Latching.** The cases listed in section 10A.8.
- **Guest merge.** Interrupted merge, retried merge, second account on the same device, and events arriving during the merge.
- **End-to-end.** Consent granted and denied, logged-in and guest, distribution change, rollback, and a publish with a KV failure followed by a successful retry.
- **Load.** `/api/personalize` and `/api/track` under expected peak traffic on the Paid plan.

## 18. Rollout

1. **Build.** Deploy the Worker with no rules published. All requests get control.
2. **Internal.** Publish a test experiment to internal accounts only. Verify assignments and exposure logs.
3. **Small.** Enable one experiment for a small share of traffic. Compare treatment shares with the configured split.
4. **Personalization.** Enable one re-evaluated rule for consented users. Watch Durable Object volume and latency.
5. **Latching.** Enable one latched rule once the re-evaluated rollout is stable.
6. **Full.** Expand traffic once dashboards and alerts are stable.

## 19. Decisions log

| # | Decision |
|---|---|
| D1 | Experiments use hashing only, with no per-user storage. |
| D2 | Personalization uses a per-user Durable Object for counters and events. |
| D3 | Personalization rules are re-evaluated by default. Latching is opt-in per rule (section 10A). |
| D4 | Login can switch an experiment treatment. Accepted and documented. |
| D5 | Guest IDs are kept across login and logout. |
| D6 | One consent covers experiments and personalization for guests. |
| D7 | Rule changes propagate through KV, with about 60 seconds of delay. |
| D8 | Cookies use `SameSite=Lax`, not `Strict`. |
| D9 | Image and media storage are out of scope. |
| D10 | The instant kill switch is optional and not part of the default design. |
| D11 | Guest state merges into the account once, at first login. A guest ID links to one account only. |

## 20. Open questions

1. **Retention periods** for guest Durable Objects, events, latches, exposure records, and erasure deadlines (section 14).
2. **Logged-in users without consent:** confirm that account-based experiment assignment is acceptable without consent.
3. **Overlap rules** between experiments.
4. **Fail-open behavior** when KV is unavailable is assumed. Confirm.
5. **Whether any rule needs the kill switch** (section 13).
6. **Latch defaults** (section 10A.9).

## 21. Later

- Frequency caps and exposure dedupe that must be atomic, using the same per-user Durable Object.
- Adaptive or bandit allocation, which needs stored state.
- Results analysis: metrics, significance, and sample ratio checks. Build or adopt a tool once assignment is live (see [findings.md](findings.md)).

## 22. References

- Cloudflare Durable Objects limits: https://developers.cloudflare.com/durable-objects/platform/limits/
- Cloudflare Durable Objects pricing: https://developers.cloudflare.com/durable-objects/platform/pricing/
