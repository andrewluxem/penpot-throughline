# Skill: penpot-environment-setup

Sets up the ThroughLine ↔ Penpot integration in a new project. Run this once before any other ThroughLine skill.

---

## What this skill does

1. Verifies the Penpot instance is reachable
2. Validates authentication (session cookie or access token)
3. Confirms the MCP plugin is configured (for interactive sessions)
4. Smoke-tests the REST API with `get-file` to confirm token data is accessible
5. Writes a `.penpot.env` config file for downstream skills to consume

---

## Prerequisites

- A running Penpot instance (self-hosted or cloud). For self-hosted, use the hardened compose in `ops/docker-compose.hardened.yaml`.
- A Penpot account with access to the target project
- The file ID of the Penpot design file you want to sync from (copy from the URL: `https://<host>/design/<file-id>/...`)

**For interactive sessions (MCP path):**
- Claude Code with the Penpot MCP plugin configured (see MCP Configuration below)
- Chrome open on the Penpot editor with the file loaded

**For CI/headless sessions (REST API path):**
- Service account credentials (email + password)
- Or: access token (requires `enable-access-tokens` flag in Penpot backend; see Access Tokens below)

---

## MCP Configuration (interactive sessions only)

The Penpot MCP is **not** a standalone stdio server, and there is **no `.mcp.json` `command` entry for it.** It is a browser plugin served by the `penpotapp/mcp:2.16` container on **port 4400**. It bridges tool calls over the WS session while the editor is open. There is no separate process to launch from Claude Code's side and no token to configure.

**Do not** put `"command": "penpot-mcp"` in `.mcp.json`. No first-party binary of that name ships with the monorepo image; the only thing that name resolves to is the standalone `penpot/penpot-mcp` repo, which is CVE-vulnerable (CVE-2026-45805). A standalone-process config is both architecturally wrong and a security risk.

What this means in practice:
- **Interactive only.** MCP tools exist only while the `penpotapp/mcp:2.16` container is running and a browser is open on the Penpot editor. Headless/CI runs must use the REST API path (Steps 1–5 below).
- **Separate container.** The MCP plugin server is `penpotapp/mcp:2.16`, which runs its own Vite plugin server on port 4400. It is not part of the `penpot-frontend` or `penpot-backend` images. Add it to your compose when you need interactive MCP (see compose snippet below).
- **Security:** Use only `penpotapp/mcp:2.16` — never the standalone `penpot/penpot-mcp` repo (CVE-2026-45805) or the community `montevive/penpot-mcp` (stores a plaintext password, logs the session token).

### Adding the MCP container (optional, interactive sessions only)

Add to `ops/docker-compose.hardened.yaml` when you need the interactive MCP path:

```yaml
penpot-mcp:
  # Pin by digest before use — this is a placeholder tag
  image: penpotapp/mcp:2.16
  restart: unless-stopped
  ports:
    - "127.0.0.1:4400:4400"   # bind localhost only — never expose externally
  environment:
    PENPOT_BASE_URL: "http://penpot-frontend"
```

Binding to `127.0.0.1` ensures the plugin server is reachable from your browser but not from the LAN or Tailscale.

### Installing the plugin in the Penpot editor

Once the MCP container is running:

1. Open the Penpot editor (any file)
2. Click the plugin icon in the top toolbar (puzzle piece)
3. Paste the plugin manifest URL: `http://localhost:4400/manifest.json`

Verify it's serving correctly first:
```bash
curl -s http://localhost:4400/manifest.json | python3 -m json.tool | head -10
```
Expected: JSON with `name`, `host`, `code` fields. If you get HTML, the container isn't running.

---

## Step 1 — Verify instance reachability

```bash
curl -s -o /dev/null -w "%{http_code}" http://localhost:9001/api/rpc/command/get-profile
```

Expected: `401` (unauthenticated but reachable). Any `000` or connection refused means the instance is not up.

---

## Step 2 — Authenticate and get a session cookie

```bash
curl -s -c /tmp/penpot.cookies \
  -X POST "http://localhost:9001/api/rpc/command/login-with-password" \
  -H "Content-Type: application/json" \
  -d '{"email": "YOUR_EMAIL", "password": "YOUR_PASSWORD"}' \
  | python3 -m json.tool | grep -E '"(fullname|email)"'
```

Replace `YOUR_EMAIL` and `YOUR_PASSWORD`. A successful response includes `fullname` and `email`. The session cookie is written to `/tmp/penpot.cookies`.

**Never commit credentials.** Use environment variables:

```bash
curl -s -c /tmp/penpot.cookies \
  -X POST "http://localhost:9001/api/rpc/command/login-with-password" \
  -H "Content-Type: application/json" \
  -d "{\"email\": \"$PENPOT_EMAIL\", \"password\": \"$PENPOT_PASSWORD\"}"
```

### Access tokens (alternative to password auth)

If your Penpot instance has access tokens enabled (`enable-access-tokens` in `PENPOT_FLAGS`):

```bash
curl -s -H "Authorization: Token YOUR_ACCESS_TOKEN" \
  "http://localhost:9001/api/rpc/command/get-profile"
```

Swap `-c /tmp/penpot.cookies` for `-H "Authorization: Token $PENPOT_ACCESS_TOKEN"` in all subsequent calls.

---

## Step 3 — Discover team and project

```bash
# Get teams
curl -s -b /tmp/penpot.cookies \
  "http://localhost:9001/api/rpc/command/get-teams" \
  | python3 -m json.tool | grep -E '"(name|id)"' | head -20

# Get files in a project (replace PROJECT_ID)
curl -s -b /tmp/penpot.cookies \
  "http://localhost:9001/api/rpc/command/get-project-files?project-id=PROJECT_ID" \
  | python3 -m json.tool | grep -E '"(name|id)"'
```

Note the team ID — you'll need it to extract the feature flags for Step 4.

---

## Step 4 — Smoke-test get-file with token data

The `get-file` endpoint requires you to declare the features the file uses. Pass all features from the team's feature flags to avoid `feature-not-supported` errors.

```bash
# Standard feature set for Penpot 2.16 files with design tokens:
FEATURES="design-tokens/v1,fdata/path-data,variants/v1,layout/grid,components/v2,fdata/shape-data-type,styles/v2,fdata/objects-map,tokens/numeric-input,render-wasm/v1"

curl -s -b /tmp/penpot.cookies \
  "http://localhost:9001/api/rpc/command/get-file?id=FILE_ID&features=$FEATURES" \
  | python3 -m json.tool 2>/dev/null | grep -c '"~:tokens-lib"'
```

Expected output: `1` (the key is present). If `0`, the file has no design tokens yet — add some via Assets → Tokens in the editor.

To see the full token structure:

```bash
curl -s -b /tmp/penpot.cookies \
  "http://localhost:9001/api/rpc/command/get-file?id=FILE_ID&features=$FEATURES" \
  | python3 -m json.tool 2>/dev/null | grep -A 80 '"~:tokens-lib"'
```

---

## Step 5 — Write the project config

Create `.penpot.env` in the project root (add to `.gitignore`):

```bash
cat > .penpot.env << 'EOF'
# Penpot connection config — consumed by ThroughLine skills
# DO NOT COMMIT — add .penpot.env to .gitignore

PENPOT_BASE_URL=http://localhost:9001
PENPOT_FILE_ID=YOUR_FILE_ID
PENPOT_PROJECT_ID=YOUR_PROJECT_ID
PENPOT_TEAM_ID=YOUR_TEAM_ID

# Auth — use ONE of:
# Option A: session auth (local dev)
PENPOT_EMAIL=your@email.com
# PENPOT_PASSWORD is intentionally not stored here — pass at runtime

# Option B: access token (CI)
# PENPOT_ACCESS_TOKEN=your-token-here
EOF
```

Add to `.gitignore`:
```
.penpot.env
```

---

## Validation checklist

- [ ] `curl` to `/api/rpc/command/get-profile` returns `401` (instance reachable)
- [ ] Login returns a profile object with your email
- [ ] `get-teams` returns at least one team
- [ ] `get-project-files` returns the target file
- [ ] `get-file` with features returns `"~:tokens-lib"` in the response
- [ ] `.penpot.env` is written and in `.gitignore`
- [ ] MCP plugin installed in Penpot editor (if using interactive path)

---

## Troubleshooting

**`feature-not-supported` from get-file** — The file uses a feature not in your `features=` query param. Get the full list from the team response under `~:features`, then pass all of them.

**`not-found` from get-project-files** — You're likely passing a file ID instead of a project ID. The project ID is a different UUID. Find it from `get-teams` → navigate to the project in the UI → copy from the URL.

**Empty `~:tokens-lib`** — No design tokens have been created in the file yet. Open the file in the editor, go to Assets (sidebar) → Tokens tab, and create at least one token set.

**Port 4402 connection refused** — This is the WS plugin bridge; it's not published in the hardened compose. Only port 9001 (HTTP) is externally accessible. The MCP plugin connects via the browser session, not via port 4402 directly.

**`disable-email-verification` required for local dev** — Without SMTP configured, new accounts need email verification disabled in the backend flags. See the hardened compose comments.

---

## Next skill

After environment setup is validated, run `design-system-audit` to inventory the full file: components, token sets, themes, and styles.
