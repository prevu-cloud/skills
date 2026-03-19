---
name: prevu-deploy
description: Deploy project previews to Prevu Cloud. Use when asked to preview, deploy, demo, or share a project via a temporary URL. Supports static sites (HTML/React/Vue/Next.js), backend services (Python/Node/Go), Docker images, and fullstack apps with infrastructure dependencies (PostgreSQL, Redis, MinIO). Triggers on "deploy preview", "prevu", "preview this", "share a demo", "deploy to preview", "give me a URL".
---

# Prevu Deploy

Deploy projects to [Prevu Cloud](https://prevu.cloud) for instant preview URLs.

**Key rule: Build locally, upload artifacts.** Prevu runs your compiled output — it does not build your project.

## Setup

The `@prevu/mcp-server` package must be configured. If not available, use the HTTP API directly.

```bash
npx @prevu/mcp-server
```

Or configure in your MCP client:
```json
{ "mcpServers": { "prevu": { "command": "npx", "args": ["-y", "@prevu/mcp-server"] } } }
```

## Deployment Modes

### Static Sites (React, Vue, Next.js, plain HTML)

1. **Build locally first**: `npm run build` (or equivalent)
2. Deploy the **output directory** (e.g., `dist/`, `build/`, `out/`)

```
create_preview(mode="static", path="./dist")
```

Next.js requires `output: 'export'` in `next.config.mjs` for static export.

### Runtime Backends (Python, Node.js, Go)

**Go**: Cross-compile to Linux binary locally, then deploy:
```bash
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o server .
```
```
create_preview(mode="runtime", runtime="go", path="./output", start_command="./server", runtime_port=8080, healthcheck_path="/health")
```

**Python**: Include `requirements.txt`. Dependencies are auto-installed via `pip install --target` in an init container:
```
create_preview(mode="runtime", runtime="python:3.12", path="./src", start_command="python app.py", runtime_port=8080, healthcheck_path="/health")
```

**Node.js**: Include `package.json`. Dependencies are auto-installed via `npm install --production`:
```
create_preview(mode="runtime", runtime="node:22", path="./src", start_command="node server.js", runtime_port=3000, healthcheck_path="/health")
```

### Docker Images

```
create_preview(mode="image", image_ref="ghcr.io/user/app:latest", runtime_port=3000, healthcheck_path="/health")
```

## Infrastructure Dependencies

Prevu auto-provisions databases and services. Add `dependencies` to `create_preview`:

```json
{
  "dependencies": [
    {
      "type": "postgres",
      "mode": "dsn",
      "env_mapping": { "DSN": "DATABASE_URL" }
    }
  ]
}
```

**Available dependency types:**

| Type | Modes | Canonical Keys |
|------|-------|---------------|
| `postgres` | `dsn` → `[DSN]`; `individual` → `[HOST, PORT, USER, PASSWORD, DATABASE, SSLMODE]` | Map to your env vars |
| `redis` | `url` → `[URL, PREFIX]`; `individual` → `[HOST, PORT, PASSWORD, PREFIX]` | |
| `minio` | `individual` → `[ENDPOINT, ACCESS_KEY, SECRET_KEY, BUCKET, USE_SSL, REGION]`; `url` → `[URL]` | |

PostgreSQL options: `{"extensions": "uuid-ossp,pgcrypto"}`.

Connection info is injected as environment variables into the preview container.

## Fullstack Apps (Frontend + Backend)

Deploy as two separate previews:

1. **Deploy backend first** → get its URL
2. **Build frontend** with backend URL as env var (e.g., `NEXT_PUBLIC_API_URL=https://xxx.prevu.page npm run build`)
3. **Deploy frontend** as static

The backend must handle CORS if frontend and backend are on different subdomains.

## Lifecycle

```
create_preview(...)        → returns preview_id, url, token
get_preview_status(id)     → check deployment progress
destroy_preview(id, token) → cleanup (also destroys provisioned dependencies)
```

- Previews auto-expire (default 1 hour, max 24 hours, set via `ttl_seconds`)
- Each preview gets a unique `*.prevu.page` subdomain with TLS
- A 6-character `claim_code` is returned for sharing

## Common Patterns

### Quick HTML demo
```
create_preview(mode="static", path="./public")
```

### Flask + PostgreSQL API
```
create_preview(
  mode="runtime", runtime="python:3.12", path="./backend",
  start_command="python app.py", runtime_port=8080,
  healthcheck_path="/health",
  dependencies=[{type: "postgres", mode: "dsn", env_mapping: {DSN: "DATABASE_URL"}}]
)
```

### Go microservice
```bash
# Build first!
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o server .
mkdir -p /tmp/deploy && cp server /tmp/deploy/
```
```
create_preview(
  mode="runtime", runtime="go", path="/tmp/deploy",
  start_command="./server", runtime_port=8080,
  healthcheck_path="/api/health"
)
```

## Troubleshooting

- **404 on root path**: Your app doesn't define a `/` route — this is app-level, not Prevu
- **CORS errors**: Backend needs `Access-Control-Allow-Origin` headers for cross-subdomain requests
- **ModuleNotFoundError (Python)**: Ensure `requirements.txt` is in the deploy directory
- **Binary won't execute (Go)**: Must cross-compile with `CGO_ENABLED=0 GOOS=linux GOARCH=amd64`
- **Deploy timeout**: Check `get_preview_status` — init containers pulling images or installing deps can take 30-60s

## HTTP API Fallback

If MCP is not available, use `curl`:

```bash
# Create
curl -X POST https://api.prevu.cloud/api/v1/previews \
  -H "Content-Type: application/json" \
  -d '{"name":"my-app","mode":"static","ttl_seconds":3600,"source_type":"mcp"}'

# Upload (static/runtime)
curl -X POST https://api.prevu.cloud/api/v1/previews/{id}/upload -F "file=@site.zip"

# Status
curl https://api.prevu.cloud/api/v1/previews/{id}

# Destroy
curl -X POST https://api.prevu.cloud/api/v1/previews/{id}/destroy -H "Authorization: {token}"
```
