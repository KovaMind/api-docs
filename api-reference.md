# Kova Mind REST API Reference

Base URL: `https://api.kovamind.io`

All endpoints require `Authorization: Bearer km_live_xxx` unless noted.

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
  "patterns": [
    {
      "id": "42",
      "pattern": "prefers dark mode",
      "category": "preference",
      "confidence": 0.95,
      "user_id": "alex"
    },
    {
      "id": "43",
      "pattern": "dislikes Comic Sans",
      "category": "preference",
      "confidence": 0.90,
      "user_id": "alex"
    }
  ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `conversation` | array | Yes | Messages with `role` and `content` (max 10,000 chars each) |
| `user_id` | string | Yes | Unique user identifier |
| `session_id` | string | No | Group extractions across calls |

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
      "id": "42",
      "pattern": "prefers dark mode",
      "category": "preference",
      "confidence": 0.95,
      "user_id": "alex"
    }
  ]
}
```

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `context` | string | Yes | — | Natural language query |
| `user_id` | string | Yes | — | User whose memory to search |
| `max_patterns` | integer | No | 10 | Max results (1–100) |
| `min_confidence` | float | No | 0.3 | Min confidence (0.0–1.0) |

---

## POST /api/memory/reinforce

Confirm, deny, strengthen, or weaken a stored pattern.

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
  "type": "confirmed",
  "success": true
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `pattern_id` | string | Yes | Pattern to reinforce |
| `reinforcement_type` | string | Yes | `confirmed`, `denied`, `strengthened`, `weakened` |
| `context` | string | No | Explanation for the reinforcement |

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
  "surprise_score": 0.82,
  "route": "contradict"
}
```

| Score Range | Route | Meaning |
|-------------|-------|---------|
| 0.0 – 0.29 | `reinforce` | Already known — boosts existing pattern |
| 0.3 – 0.69 | `update` | New information — stored normally |
| 0.7 – 1.0 | `contradict` | Conflicts with memory — old pattern flagged |

---

## GET /health

Check API health. **No authentication required.**

**Request:**

```bash
curl https://api.kovamind.io/health
```

**Response:**

```json
{
  "status": "ok",
  "version": "1.0.0"
}
```

---

## Errors

| Status | Error | Description |
|--------|-------|-------------|
| 401 | `AuthError` | Invalid or missing API key |
| 403 | `IdentityError` | API key is bound to a different agent identity (see [Bound API keys](#bound-api-keys)) |
| 404 | `NotFoundError` | Pattern or resource not found |
| 429 | `RateLimitError` | Rate limit exceeded (check `Retry-After` header) |
| 500 | `ServerError` | Internal server error |

All errors return:

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

This lets you lock a key to a specific agent (e.g. a single deployed bot) so a leaked or misused key cannot read or write another user's memory. The memory endpoints (`/api/memory/extract`, `/api/memory/retrieve`, `/api/memory/reinforce`, `/api/memory/surprise`) all enforce this binding. Send the `user_id` that matches the key's bound identity, or use an unbound key.

---

## Rate Limits

| Endpoint | Free | Paid |
|----------|------|------|
| `/api/memory/extract` | 100/day | 10,000/day |
| `/api/memory/retrieve` | 200/day | 20,000/day |
| `/api/memory/surprise` | 500/day | Unlimited |

When rate limited, the response includes a `Retry-After` header with seconds to wait.

---

## Vault v2 (beta)

The API also exposes encrypted-credential routes under `/api/vault/v2/*`, and the
official SDKs export helpers for them. **Vault v2 is experimental** — the routes
and their request/response shapes may change without notice and are not yet
covered by this reference. Treat anything you build on `/api/vault/v2/*` as beta
and pin your SDK version. Stable, fully-specified docs will follow once the
surface settles.
