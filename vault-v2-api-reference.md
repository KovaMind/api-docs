# Vault v2 API Reference

Secure credential storage with opaque handles. Vault v2 ensures that **no endpoint ever returns plaintext credential values** — credentials go in, opaque handles come out, and execution happens server-side.

Base URL: `https://api.kovamind.ai`

All endpoints require `Authorization: Bearer km_live_xxx`.

> **Security invariant:** Plaintext secrets are accepted only at ingestion (`POST /vault/v2/credentials`) and are never returned by any endpoint. All subsequent operations reference credentials by opaque handle.

---

## POST /vault/v2/setup

First-time vault setup. Derives an encryption key from the provided passphrase and returns a 12-word recovery phrase. Must be called once before any other vault endpoint.

**Request:**

```bash
curl -X POST https://api.kovamind.ai/vault/v2/setup \
  -H "Authorization: Bearer km_live_xxx" \
  -H "Content-Type: application/json" \
  -d '{
    "passphrase": "my-strong-vault-passphrase"
  }'
```

**Response:**

```json
{
  "recovery_words": [
    "alpha", "bravo", "charlie", "delta", "echo", "foxtrot",
    "golf", "hotel", "india", "juliet", "kilo", "lima"
  ],
  "vault_id": "vlt_a1b2c3d4e5f6",
  "status": "initialized"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `passphrase` | string | Yes | Passphrase used to derive the vault encryption key (min 12 chars) |

| Response Field | Type | Description |
|----------------|------|-------------|
| `recovery_words` | string[] | 12-word BIP-39 recovery phrase — **store offline, shown only once** |
| `vault_id` | string | Unique vault identifier |
| `status` | string | Always `initialized` on success |

---

## POST /vault/v2/unlock

Unlock the vault for the current session. The derived key is held in memory until lock or session expiry.

**Request:**

```bash
curl -X POST https://api.kovamind.ai/vault/v2/unlock \
  -H "Authorization: Bearer km_live_xxx" \
  -H "Content-Type: application/json" \
  -d '{
    "passphrase": "my-strong-vault-passphrase"
  }'
```

**Response:**

```json
{
  "vault_id": "vlt_a1b2c3d4e5f6",
  "status": "unlocked",
  "expires_at": "2026-03-20T18:30:00Z"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `passphrase` | string | Yes | Vault passphrase |

| Response Field | Type | Description |
|----------------|------|-------------|
| `vault_id` | string | Vault identifier |
| `status` | string | `unlocked` |
| `expires_at` | string | ISO 8601 timestamp when the session auto-locks |

---

## POST /vault/v2/lock

Lock the vault immediately. Zeros the encryption key from memory.

**Request:**

```bash
curl -X POST https://api.kovamind.ai/vault/v2/lock \
  -H "Authorization: Bearer km_live_xxx" \
  -H "Content-Type: application/json"
```

**Response:**

```json
{
  "vault_id": "vlt_a1b2c3d4e5f6",
  "status": "locked"
}
```

No request body required.

---

## POST /vault/v2/credentials

Store a credential in the vault. Returns an opaque handle — **the plaintext value is never returned by any endpoint after storage**.

**Request:**

```bash
curl -X POST https://api.kovamind.ai/vault/v2/credentials \
  -H "Authorization: Bearer km_live_xxx" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "AWS Production",
    "type": "api_key",
    "value": "AKIAIOSFODNN7EXAMPLE",
    "metadata": {
      "provider": "aws",
      "environment": "production",
      "region": "us-east-1"
    }
  }'
```

**Response:**

```json
{
  "id": "cred_x9k2m4p7q1w3",
  "handle": "hdl_r8t5v2n6j0y4",
  "name": "AWS Production",
  "type": "api_key",
  "metadata": {
    "provider": "aws",
    "environment": "production",
    "region": "us-east-1"
  },
  "created_at": "2026-03-20T14:22:00Z"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Human-readable credential name |
| `type` | string | Yes | Credential type: `api_key`, `oauth_token`, `password`, `certificate`, `ssh_key`, `other` |
| `value` | string | Yes | The secret value to store (encrypted at rest, **never returned**) |
| `metadata` | object | No | Arbitrary key-value metadata (not encrypted, used for search/filtering) |

| Response Field | Type | Description |
|----------------|------|-------------|
| `id` | string | Credential identifier (for deletion) |
| `handle` | string | Opaque handle (for execution and reference) |
| `name` | string | Credential name |
| `type` | string | Credential type |
| `metadata` | object | Metadata (if provided) |
| `created_at` | string | ISO 8601 creation timestamp |

---

## GET /vault/v2/credentials

List all stored credentials. Returns metadata only — **no plaintext values**.

**Request:**

```bash
curl https://api.kovamind.ai/vault/v2/credentials \
  -H "Authorization: Bearer km_live_xxx"
```

**Response:**

```json
{
  "credentials": [
    {
      "id": "cred_x9k2m4p7q1w3",
      "handle": "hdl_r8t5v2n6j0y4",
      "name": "AWS Production",
      "type": "api_key",
      "metadata": {
        "provider": "aws",
        "environment": "production",
        "region": "us-east-1"
      },
      "created_at": "2026-03-20T14:22:00Z",
      "last_used_at": "2026-03-20T16:45:00Z"
    }
  ]
}
```

---

## DELETE /vault/v2/credentials/{id}

Delete a credential permanently. This action is irreversible.

**Request:**

```bash
curl -X DELETE https://api.kovamind.ai/vault/v2/credentials/cred_x9k2m4p7q1w3 \
  -H "Authorization: Bearer km_live_xxx"
```

**Response:**

```json
{
  "id": "cred_x9k2m4p7q1w3",
  "deleted": true
}
```

| Path Parameter | Type | Description |
|----------------|------|-------------|
| `id` | string | Credential ID to delete |

---

## GET /vault/v2/handles

List all opaque handles. Handles are the tokens agents use to reference credentials without ever seeing the underlying secret.

**Request:**

```bash
curl https://api.kovamind.ai/vault/v2/handles \
  -H "Authorization: Bearer km_live_xxx"
```

**Response:**

```json
{
  "handles": [
    {
      "handle": "hdl_r8t5v2n6j0y4",
      "credential_id": "cred_x9k2m4p7q1w3",
      "name": "AWS Production",
      "type": "api_key",
      "created_at": "2026-03-20T14:22:00Z"
    }
  ]
}
```

---

## GET /vault/v2/find

Search credentials by name, type, or metadata fields. Returns metadata only — **no plaintext values**.

**Request:**

```bash
curl "https://api.kovamind.ai/vault/v2/find?q=aws&type=api_key" \
  -H "Authorization: Bearer km_live_xxx"
```

**Response:**

```json
{
  "results": [
    {
      "id": "cred_x9k2m4p7q1w3",
      "handle": "hdl_r8t5v2n6j0y4",
      "name": "AWS Production",
      "type": "api_key",
      "metadata": {
        "provider": "aws",
        "environment": "production",
        "region": "us-east-1"
      },
      "created_at": "2026-03-20T14:22:00Z"
    }
  ]
}
```

| Query Parameter | Type | Required | Description |
|-----------------|------|----------|-------------|
| `q` | string | No | Free-text search across name and metadata values |
| `type` | string | No | Filter by credential type |
| `metadata.KEY` | string | No | Filter by specific metadata key (e.g., `metadata.provider=aws`) |

---

## POST /vault/v2/execute

Execute an action using a stored credential. The server resolves the opaque handle to the real secret, performs the action, and returns only the result. **The credential value never leaves the server.**

**Request:**

```bash
curl -X POST https://api.kovamind.ai/vault/v2/execute \
  -H "Authorization: Bearer km_live_xxx" \
  -H "Content-Type: application/json" \
  -d '{
    "handle": "hdl_r8t5v2n6j0y4",
    "action": "http_request",
    "params": {
      "method": "GET",
      "url": "https://s3.amazonaws.com/?list-type=2",
      "headers": {
        "x-amz-date": "20260320T143000Z"
      }
    }
  }'
```

**Response:**

```json
{
  "execution_id": "exec_m3k7p2q9w1x5",
  "handle": "hdl_r8t5v2n6j0y4",
  "action": "http_request",
  "status": "success",
  "result": {
    "status_code": 200,
    "body": "<ListAllMyBucketsResult>...</ListAllMyBucketsResult>",
    "headers": {
      "content-type": "application/xml"
    }
  },
  "executed_at": "2026-03-20T14:30:05Z"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `handle` | string | Yes | Opaque handle referencing the credential to use |
| `action` | string | Yes | Action type: `http_request`, `smtp_send`, `dns_lookup`, `custom` |
| `params` | object | Yes | Action-specific parameters (see below) |

**Action: `http_request`**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `method` | string | Yes | HTTP method (`GET`, `POST`, `PUT`, `DELETE`, etc.) |
| `url` | string | Yes | Target URL |
| `headers` | object | No | Additional headers (auth header injected automatically) |
| `body` | string | No | Request body |

**Action: `smtp_send`**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `to` | string | Yes | Recipient email |
| `subject` | string | Yes | Email subject |
| `body` | string | Yes | Email body |

**Action: `custom`**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `plugin` | string | Yes | Registered plugin identifier |
| `input` | object | Yes | Plugin-specific input payload |

---

## POST /vault/v2/recover

Recover vault access using the 12-word recovery phrase. Sets a new passphrase and unlocks the vault.

**Request:**

```bash
curl -X POST https://api.kovamind.ai/vault/v2/recover \
  -H "Authorization: Bearer km_live_xxx" \
  -H "Content-Type: application/json" \
  -d '{
    "recovery_words": [
      "alpha", "bravo", "charlie", "delta", "echo", "foxtrot",
      "golf", "hotel", "india", "juliet", "kilo", "lima"
    ],
    "new_passphrase": "my-new-strong-passphrase"
  }'
```

**Response:**

```json
{
  "vault_id": "vlt_a1b2c3d4e5f6",
  "status": "unlocked",
  "expires_at": "2026-03-20T18:30:00Z"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `recovery_words` | string[] | Yes | The 12-word recovery phrase from initial setup |
| `new_passphrase` | string | Yes | New passphrase to set (min 12 chars) |

---

## Errors

All vault endpoints use the same error format as the core API:

```json
{
  "detail": "Human-readable error message"
}
```

| Status | Error | Description |
|--------|-------|-------------|
| 400 | `ValidationError` | Invalid request body or parameters |
| 401 | `AuthError` | Invalid or missing API key |
| 403 | `VaultLockedError` | Vault is locked — call `/vault/v2/unlock` first |
| 404 | `NotFoundError` | Credential or handle not found |
| 409 | `VaultExistsError` | Vault already initialized (on duplicate `/setup`) |
| 429 | `RateLimitError` | Rate limit exceeded |
| 500 | `ServerError` | Internal server error |

---

## Rate Limits

| Endpoint | Free | Paid |
|----------|------|------|
| `/vault/v2/setup` | 1/lifetime | 1/lifetime |
| `/vault/v2/unlock` | 10/hour | 100/hour |
| `/vault/v2/credentials` (POST) | 50/day | 1,000/day |
| `/vault/v2/credentials` (GET) | 200/day | 10,000/day |
| `/vault/v2/execute` | 100/day | 10,000/day |
| `/vault/v2/find` | 200/day | 10,000/day |
