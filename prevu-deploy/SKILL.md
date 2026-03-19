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

### Static Sites (React, Vue, Next.js, Hugo, plain HTML)

**⚠️ Two-Phase Deploy (IMPORTANT):** Some static site generators (Hugo, Jekyll, etc.) bake the site URL into the HTML at build time (e.g., Hugo's `baseURL`). Since the preview subdomain is only known after creation, you **must** use a two-phase flow:

1. **Phase 1 — Create preview first** (before building):
   ```
   create_preview(mode="static", name="my-site", ttl_seconds=3600)
   # → returns subdomain, e.g. "abc123"
   # → preview URL: https://abc123.prevu.page
   ```

2. **Phase 2 — Build with the actual URL, then upload:**
   ```bash
   # Hugo example:
   hugo --minify -b "https://abc123.prevu.page/"
   
   # Then zip and upload the output directory
   ```

For frameworks that **don't** bake URLs into HTML (plain HTML, most React SPAs with relative paths), you can skip phase 1 and use the simple flow:

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

### Hugo / Jekyll site (two-phase)
```bash
# 1. Create preview first to get the subdomain
create_preview(mode="static", name="my-blog")
# → returns subdomain "abc123", url "https://abc123.prevu.page"

# 2. Build with actual URL
hugo --minify -b "https://abc123.prevu.page/"

# 3. Zip and upload
cd public && zip -r /tmp/site.zip .
# upload /tmp/site.zip to the preview
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

## Gotchas & Lessons Learned

### Hugo/Jekyll: baseURL must match preview URL (two-phase deploy)
Hugo, Jekyll, and similar SSGs bake the site URL into every HTML file at build time. If you build with a placeholder URL, all internal links and redirects will be broken. **You must use the two-phase flow:**
1. `create_preview()` first → get the subdomain
2. Build with the actual URL (e.g., `hugo -b "https://xxx.prevu.page/"`)
3. Upload the built output

This applies to any SSG that uses absolute URLs in output. React SPAs with relative paths (e.g., Vite with `base: './'`) don't have this issue.

### Next.js: Middleware blocks static export
If the project uses `next-intl`, `next-auth`, or any other **middleware-based** feature, `output: 'export'` will fail. You **must** use `output: 'standalone'` and deploy as a **runtime** preview (not static).

**Standalone deploy recipe:**
```bash
# 1. Add to next.config.ts/mjs:
#    output: 'standalone'

# 2. Build
npm run build

# 3. Assemble deploy directory (all 3 parts required!)
mkdir -p /tmp/deploy
cp -r .next/standalone/* /tmp/deploy/
cp -r .next/static /tmp/deploy/.next/static
cp -r public /tmp/deploy/public  # if exists

# 4. Deploy as runtime
create_preview(mode="runtime", runtime="node:22", path="/tmp/deploy",
  start_command="node server.js", runtime_port=3000)
```

**Why standalone?** Regular `.next` is 300MB+; standalone is ~20-30MB (includes only needed node_modules).

### Python: slim images have no wget/unzip
`python:*-slim` images lack `wget`, `curl`, `unzip`. Prevu's init containers use Python's built-in `urllib` + `zipfile` instead. **You don't need to handle this** — just know that if you see init container errors about missing commands, it's expected behavior that's already handled.

### Python: pip deps need PYTHONPATH
Dependencies are installed to `/app/.pylibs` (not system site-packages). Prevu auto-injects `PYTHONPATH=/app/.pylibs` so your app finds them. **No action needed** — but if you manually set `PYTHONPATH` in env vars, make sure to include `/app/.pylibs`.

### Go: must cross-compile
The preview runs on Linux amd64. If you build on macOS or Windows:
```bash
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o server .
```
Forgetting `CGO_ENABLED=0` can cause dynamic linking issues even on Linux.

### Artifact size matters
Large uploads (>100MB) are slow and may timeout. Always optimize:
- Next.js: use `standalone` output (~20MB vs 300MB+)
- Python: don't include `venv/` or `__pycache__/` in the zip
- Node.js: don't include `node_modules/` — they're installed by the init container
- Go: ship only the binary (single file, typically 10-20MB)

### Fullstack: deploy order is critical
1. Deploy **backend first** → get its `*.prevu.page` URL
2. Build frontend with backend URL baked in (e.g., `NEXT_PUBLIC_API_URL=https://xxx.prevu.page`)
3. Deploy frontend
4. Backend must allow CORS from the frontend's `*.prevu.page` origin

There is no service discovery between previews — you must pass URLs explicitly.

### Healthcheck path must exist
If you specify `healthcheck_path`, that endpoint must return HTTP 200. If your app has no health route, either:
- Add one (`GET /health` → 200)
- Or set `healthcheck_path="/"` if your root returns 200

The pod won't become Ready until the healthcheck passes (Kubernetes readiness probe).

## Troubleshooting

- **404 on root path**: Your app doesn't define a `/` route — this is app-level, not Prevu
- **CORS errors**: Backend needs `Access-Control-Allow-Origin` headers for cross-subdomain requests
- **ModuleNotFoundError (Python)**: Ensure `requirements.txt` is in the deploy directory
- **Binary won't execute (Go)**: Must cross-compile with `CGO_ENABLED=0 GOOS=linux GOARCH=amd64`
- **Deploy timeout**: Check `get_preview_status` — init containers pulling images or installing deps can take 30-60s
- **Pod stuck in Init**: Usually downloading artifacts or installing deps — wait 30-60s, then check status
- **Next.js 307 redirect on `/`**: Normal if using `next-intl` — it redirects to `/{locale}`. Your app works fine.

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
