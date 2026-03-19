# Kova Mind API Documentation

API reference, OpenAPI spec, and integration guides for the [Kova Mind](https://kovamind.ai) memory API.

## Quick links

| Resource | Description |
|----------|-------------|
| [OpenAPI Spec](./openapi.yaml) | Full API specification (OpenAPI 3.1) |
| [REST API Reference](./api-reference.md) | Endpoint docs with curl examples |
| [ChatGPT Custom GPT Guide](./chatgpt-gpt-action.md) | Give any GPT persistent memory |

## SDKs

| Language | Package | Install |
|----------|---------|---------|
| Python | [kovamind](https://github.com/KovaMind/python-sdk) | `pip install kovamind` |
| Node.js/TypeScript | [kovamind](https://github.com/KovaMind/js-sdk) | `npm install kovamind` |
| MCP (Claude, Cursor, VS Code) | [@kovamind/mcp-server](https://github.com/KovaMind/mcp-server) | `npx @kovamind/mcp-server` |

## Base URL

```
https://api.kovamind.ai
```

## Authentication

All endpoints except `/health` require a Bearer token:

```
Authorization: Bearer km_live_xxxxxxxxxxxxxxxx
```

Get your API key at [kovamind.ai](https://kovamind.ai).
