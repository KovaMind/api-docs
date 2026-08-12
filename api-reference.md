# Kova Mind REST API Reference

Base URL: `https://api.kovamind.io`

All endpoints require `Authorization: Bearer km_live_xxx` unless noted.

> **Scope note:** this reference covers the six core public endpoints. The
> API exposes 100+ further endpoints (encrypted-credential vault, emotions,
> archive search, sessions, usage/limits, and more) that are not yet covered
> here — they are real and in production use, but their request/response
> shapes are documented only in the interactive docs at
> `https://api.kovamind.io/api/docs` until this reference catches up.

---

## POST /api/memory/extract

Extract memory patterns from a conversation.

**Request:**

```bash
curl -X POST https://api.kovamind.io/api/memory/extract \
  -H "Authorization: Bearer km_live_xxx" \
  -H "Content-Type: application/json" \
  -d '{
    "conversation": [
      {"role": "user", "content": "I love dark mode and hate Comic Sans."},
      {"role": "assistant", "content": "Noted!"}
    ],
    "user_id": "alex",
    "session_id": "optional-session-id"
  }'
```

**Response:**

```json
{
  "patterns_extracted": 2,
  "patterns": [
    {
      "pattern_id": "42",
      "pattern_type": "preference",
      "user_id": "alex",
      "content": "prefers dark mode",
      "confidence": 0.95,
      "source": "conversation",
      "created_at": "2026-08-01T10:30:00Z",
      "last_reinforced": null,
      "reinforcement_count": 0
    },
    {
      "pattern_id": "43",
      "pattern_type": "preference",
      "user_id": "alex",
      "content": "dislikes Comic Sans",
      "confidence": 0.90,
      "source": "conversation",
      "created_at": "2026-08-01T10:30:00Z",
      "last_reinforced": null,
      "reinforcement_count": 0
    }
  ],
  "decisions_found": 0,
  "corrections_found": 0,
  "processing_time_ms": 125.5,
  "consolidation_pending": false
}
```

Request fields:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `conversation` | array | Yes | Messages with `role` and `content` (max 10,000 chars each) |
| `user_id` | string | Yes | Unique user identifier (max 128 chars) |
| `session_id` | string | No | Group extractions across calls |
| `app` | string | No | Originating app name (e.g. `vscode`) for usage attribution |
| `tags` | array | No | Free-form tags attached to every extracted pattern (max 32) |

Pattern objects carry `pattern_id`, `pattern_type`, `user_id`, `content`,
`confidence` (0.0–1.0), `source`, `created_at`, `last_reinforced`,
`reinforcement_count`, and — on retrieval — a `relevance` score. Additional
optional enrichment fields (emotion, domain, priority tier, memory state)
may appear; treat unknown fields as informational.

---

## POST /api/memory/retrieve

Retrieve relevant memory patterns for a context.

**Request:**

```bash
curl -X POST https://api.kovamind.io/api/memory/retrieve \
  -H "Authorization: Bearer km_live_xxx" \
  -H "Content-Type: application/json" \
  -d '{
    "context": "User is asking about editor preferences",
    "user_id": "alex",
    "max_patterns": 5,
    "min_confidence": 0.5
  }'
```

**Response:**

```json
{
  "patterns": [
    {
      "pattern_id": "42",
      "pattern_type": "preference",
      "user_id": "alex",
      "content": "prefers dark mode",
      "confidence": 0.95,
      "relevance": 0.87,
      "source": "conversation",
      "created_at": "2026-08-01T10:30:00Z",
      "last_reinforced": "2026-08-02T09:00:00Z",
      "reinforcement_count": 3
    }
  ],
  "decisions": [],
  "milestones": [],
  "episodic": [],
  "retrieval_time_ms": 45.2,
  "total_matches": 1,
  "degraded": false,
  "degraded_reasons": []
}
```

Request fields:

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `context` | string | Yes | — | Natural language query |
| `user_id` | string | Yes | — | User whose memory to search |
| `max_patterns` | integer | No | 10 | Max results (1–100) |
| `min_confidence` | float | No | 0.1 | Min confidence (0.0–1.0) |
| `pattern_type` | string | No | — | Filter by pattern type (e.g. `preference`) |
| `include_tiers` | boolean | No | false | Return tiered results in a `tiers` field |
| `token_budget` | integer | No | 2000 | Token budget for tiered retrieval (100–10,000) |

Response extras beyond the example above: `injection_text` (pre-formatted
prompt block), `token_budget_used`, `tiers` (when `include_tiers=true`),
plus optional diagnostic arrays (`stale_flags`, `conflicts`,
`media_results`, `confidence_warnings`, `proactive_suggestions`).
`degraded: true` with reason codes in `degraded_reasons` means a retrieval
stage failed and results are partial.

A `GET /api/memory/retrieve` form with the same fields as query parameters
also exists; POST is canonical (long context strings exceed URL limits).

---

## POST /api/memory/reinforce

Confirm, contradict, or mark a stored pattern as used.

**Request:**

```bash
curl -X POST https://api.kovamind.io/api/memory/reinforce \
  -H "Authorization: Bearer km_live_xxx" \
  -H "Content-Type: application/json" \
  -d '{
    "pattern_id": "42",
    "reinforcement_type": "confirmed",
    "context": "User explicitly confirmed this preference"
  }'
```

**Response:**

```json
{
  "pattern_id": "42",
  "previous_confidence": 0.75,
  "new_confidence": 0.85,
  "reinforcement_type": "confirmed",
  "timestamp": "2026-08-01T10:30:00Z"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `pattern_id` | string | Yes | Pattern to reinforce (numeric string; non-numeric returns `400`) |
| `reinforcement_type` | string | Yes | `confirmed`, `contradicted`, or `used` |
| `context` | string | No | Explanation for the reinforcement |

These three values are the only legal `reinforcement_type` values:

| Value | Effect | When to send it |
|-------|--------|-----------------|
| `confirmed` | confidence +0.10 (capped at 1.0) | User validated the memory is correct |
| `contradicted` | confidence −0.15 (floored at 0.0) | User pushed back / said it's wrong |
| `used` | confidence +0.05 (capped at 1.0) | Your agent surfaced the pattern and the user didn't object |

---

## POST /api/memory/surprise

Score how surprising new content is vs existing memories.

**Request:**

```bash
curl -X POST https://api.kovamind.io/api/memory/surprise \
  -H "Authorization: Bearer km_live_xxx" \
  -H "Content-Type: application/json" \
  -d '{
    "content": "Alex now prefers light mode",
    "user_id": "alex"
  }'
```

**Response:**

```json
{
  "user_id": "alex",
  "content": "Alex now prefers light mode",
  "surprise_score": 0.8215,
  "route": "contradict",
  "thresholds": {
    "reinforce_below": 0.3,
    "contradict_above": 0.7
  }
}
```

`content` is echoed back truncated to the first 100 characters. The
`thresholds` object reports the routing cut-offs the server used (defaults
shown; they are server-configurable):

| Score | Route | Meaning |
|-------|-------|---------|
| below `reinforce_below` (default 0.3) | `reinforce` | Already known — boosts the existing pattern |
| between the thresholds | `update` | New information — store normally |
| at or above `contradict_above` (default 0.7) | `contradict` | Conflicts with memory — flag the old pattern |

---

## GET /api/memory/context

Emotion context for a conversation. Returns detected emotions with
intensities, the dominant emotion, and an overall sentiment label.

**Request:**

```bash
curl "https://api.kovamind.io/api/memory/context?conversation_id=conv-123" \
  -H "Authorization: Bearer km_live_xxx"
```

**Response:**

```json
{
  "conversation_id": "conv-123",
  "emotions": {"joy": 0.8},
  "dominant_emotion": "joy",
  "sentiment": "positive"
}
```

`sentiment` is one of `positive` / `neutral` / `negative`. `emotions` may be
empty when the conversation has no recorded emotional signal yet.

---

## GET /health

Check API health. **No authentication required.** Also available at
`/api/health` with an identical payload.

**Request:**

```bash
curl https://api.kovamind.io/health
```

**Response:**

```json
{
  "status": "healthy",
  "version": "1.0.0",
  "database": "ok",
  "routers_mounted": {
    "memory": true,
    "vault": true,
    "auditor": true,
    "inside_out": true,
    "webhook": true,
    "emotions": true
  }
}
```

`status` is `healthy` or `degraded`; `database` is `ok` or `degraded`. A
`missing_tables` array appears only when expected database tables are
missing. When the database or the memory subsystem is down the same JSON
shape is returned with HTTP `503`.

---

## Errors

| Status | Description |
|--------|-------------|
| 400 | Malformed value the endpoint can't use (e.g. non-numeric `pattern_id` on reinforce) |
| 401 | Invalid or missing API key |
| 403 | API key is bound to a different agent identity (see [Bound API keys](#bound-api-keys)) |
| 404 | Pattern or resource not found |
| 422 | Request body failed validation (`detail` is an array of validation errors) |
| 429 | Rate limit exceeded (see [Rate limits](#rate-limits) for the headers) |
| 500 | Internal server error |

Errors return:

```json
{
  "detail": "Human-readable error message"
}
```

---

## Bound API keys

An API key can be **bound** server-side to a single agent identity (one `user_id`).

- **Bound key:** if a request's `user_id` differs from the identity the key is bound to, the API returns `403`:

  ```json
  {
    "detail": "API key is bound to a different agent identity"
  }
  ```

- **Unbound key:** the client-supplied `user_id` is passed through unchanged, so one key can serve many users.

This lets you lock a key to a specific agent (e.g. a single deployed bot) so a leaked or misused key cannot read or write another user's memory.

The `user_id` mismatch check only applies to endpoints that **accept a `user_id` in their body** — `/api/memory/extract`, `/api/memory/retrieve`, and `/api/memory/surprise`. On those, send the `user_id` that matches the key's bound identity (or use an unbound key), otherwise you get the `403` above.

Endpoints that **don't take a `user_id`** — such as `/api/memory/reinforce` — aren't subject to the `user_id` mismatch check; they operate on a target pattern that already belongs to the key's bound identity, so there is no `user_id` to match.

---

## Rate Limits

Rate limiting is **tier-based** and enforced per tenant in two layers:

**1. Per-minute burst window** (60 seconds, per endpoint category):

| Tier | extract | retrieve | all other endpoints |
|------|---------|----------|---------------------|
| Free | 200/min | 500/min | 1,000/min |
| Starter | 500/min | 1,500/min | 3,000/min |
| Pro | 1,000/min | 2,500/min | 5,000/min |
| Team | unlimited | unlimited | unlimited |

**2. Per-day quota** (UTC calendar day). Defaults below — Free and Starter
daily quotas can be operator-adjusted, so treat these as the shipped
defaults rather than a contract:

| Tier | extract | retrieve | surprise | all other endpoints |
|------|---------|----------|----------|---------------------|
| Free | 100/day | 500/day | 200/day | 1,000/day |
| Starter | 2,000/day | 10,000/day | 5,000/day | 20,000/day |
| Pro | 20,000/day | 100,000/day | 50,000/day | 200,000/day |
| Team | unlimited | unlimited | unlimited | unlimited |

**Headers.** Every authenticated response carries the burst-layer triplet:

- `X-RateLimit-Limit` — your per-minute cap (the literal string `unlimited` on uncapped tiers)
- `X-RateLimit-Remaining` — requests left in the current minute window
- `X-RateLimit-Reset` — unix-epoch seconds when the minute window flips

On `429`:

- burst-window hit → `Retry-After: 60` plus the `X-RateLimit-*` triplet
- daily-quota hit → `Retry-After: 86400` plus `X-Quota-Limit` and `X-Quota-Used`

Customers with a dashboard session token can call `GET /api/v1/my/limits`
for their live caps and current usage in JSON.

---

## Vault v2 (beta)

The API also exposes encrypted-credential routes under `/api/vault/v2/*`, and the
official SDKs export helpers for them. **Vault v2 is experimental** — the routes
and their request/response shapes may change without notice and are not yet
covered by this reference. Treat anything you build on `/api/vault/v2/*` as beta
and pin your SDK version. Stable, fully-specified docs will follow once the
surface settles.
