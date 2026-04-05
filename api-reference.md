# Kova Mind REST API Reference

Base URL: `https://api.kovamind.io`

All endpoints require `Authorization: Bearer km_live_xxx` unless noted.

---

## POST /memory/extract

Extract memory patterns from a conversation.

**Request:**

```bash
curl -X POST https://api.kovamind.io/memory/extract \
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

## POST /memory/retrieve

Retrieve relevant memory patterns for a context.

**Request:**

```bash
curl -X POST https://api.kovamind.io/memory/retrieve \
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

## POST /memory/reinforce

Confirm, deny, strengthen, or weaken a stored pattern.

**Request:**

```bash
curl -X POST https://api.kovamind.io/memory/reinforce \
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

## POST /memory/surprise

Score how surprising new content is vs existing memories.

**Request:**

```bash
curl -X POST https://api.kovamind.io/memory/surprise \
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

## Rate Limits

| Endpoint | Free | Paid |
|----------|------|------|
| `/memory/extract` | 100/day | 10,000/day |
| `/memory/retrieve` | 200/day | 20,000/day |
| `/memory/surprise` | 500/day | Unlimited |

When rate limited, the response includes a `Retry-After` header with seconds to wait.
