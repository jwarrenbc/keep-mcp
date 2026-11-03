# Auth Proxy Deployment Example (Caddy + SSE Reverse Proxy)

This directory contains a lightweight, zero-buffering reverse proxy configuration using [Caddy](https://caddyserver.com/) to protect a hosted `keep-mcp` server.

## Overview

When hosting `keep-mcp` on cloud container platforms (Render, Google Cloud Run, Fly.io, Railway), you can run `keep-mcp` as a **Private Service** (inaccessible to the public internet) and place this auth proxy in front as the public ingress gateway.

```
MCP Clients (Gemini, Cursor, Claude, etc.)
                   │
                   ▼ HTTPS (Public Ingress)
    ┌──────────────────────────────┐
    │     Caddy Auth Proxy         │
    │  - Port 8080 (Public)        │
    │  - Validates API Key         │
    │  - Zero-buffering for SSE    │
    │  - Health probe on /healthz  │
    └──────────────┬───────────────┘
                   │ Internal Private Network
                   ▼
    ┌──────────────────────────────┐
    │     keep-mcp Backend         │
    │  - Port 8080 (Private)       │
    │  - Not exposed to internet   │
    └──────────────────────────────┘
```

## Features

- **SSE Streaming Support:** Configured with `flush_interval -1` and `buffer_requests false` to guarantee real-time, unbuffered Server-Sent Events (SSE) and Streamable HTTP delivery.
- **Multiple Auth Methods:** Accepts the API key via:
  - `Authorization: Bearer <MCP_API_KEY>`
  - `X-API-Key: <MCP_API_KEY>`
  - `?token=<MCP_API_KEY>` (URL query parameter for clients that only support a single URL field)
- **Health Probes:** Exposes an unauthenticated `/healthz` endpoint returning `200 OK` for platform health and readiness checks.
- **Minimal Footprint:** Uses `caddy:2-alpine` (~15 MB compressed image).

## Environment Variables

| Variable | Description | Default |
|---|---|---|
| `MCP_API_KEY` | The secret API key required to authenticate requests | *(Required)* |
| `BACKEND_URL` | The URL of the internal `keep-mcp` service | `http://keep-mcp:8080` |
| `PORT` | Port the proxy listens on (automatically set by Render / Cloud Run) | `8080` |

## Deployment on Render

1. **Deploy `keep-mcp` as a Private Service:**
   - In Render Dashboard, create or configure `keep-mcp` as a **Private Service**.
   - Note its internal service name (e.g. `keep-mcp` listening on port `8080`).

2. **Deploy the Auth Proxy as a Web Service:**
   - Click **New +** → **Web Service**.
   - Connect your repository.
   - Choose **Docker** environment and set the **Docker Context** and **Dockerfile Path**:
     - Docker Context: `docs/deployments/auth-proxy`
     - Dockerfile Path: `docs/deployments/auth-proxy/Dockerfile`
   - Set the environment variables:
     - `MCP_API_KEY`: `<your-secret-api-key>`
     - `BACKEND_URL`: `http://<keep-mcp-service-name>:8080`
   - Deploy!
