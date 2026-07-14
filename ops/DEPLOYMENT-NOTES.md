# Penpot Deployment Notes

## Which image to use

**Use `penpotapp/mcp:2.16` (the monorepo-bundled MCP).** Two Penpot MCP codebases exist:

| Codebase | Image/source | Status |
|---|---|---|
| `penpot/penpot-mcp` (standalone GitHub repo) | Build from source | **⚠ VULNERABLE** — CVE-2026-45805 unpatched (unauthenticated `/execute` REPL, binds `0.0.0.0`, CVSS 8.8) |
| `/mcp` inside `penpot/penpot` monorepo | `penpotapp/mcp:2.16` | **✓ Patched** — REPL binds localhost only |

All four known Penpot CVEs (CVE-2026-44986, -45805, -45806, -26202) are fixed at or below 2.15.0. The 2.16 image is clean against all of them.

**Do not build the standalone `penpot/penpot-mcp` repo. Deploy the 2.16 compose only.**

## MCP authentication model

The official Penpot MCP server stores no credentials. The plugin rides the browser's existing authenticated session — there is no token to manage or rotate for MCP access. This is the correct model.

The community `montevive/penpot-mcp` alternative requires a plaintext account password in `.env`, defaults to Penpot's cloud API (not your self-hosted instance), and prints the live session token to stdout with `DEBUG=true` as default. **Do not use it.**

## Hardened compose changes (vs. stock)

The compose in `docker-compose.hardened.yaml` makes these changes:

1. **Image digest pinning** — all 7 images pinned by `sha256:` digest. Digests must be re-verified and updated on each Penpot release.
2. **Secrets from env** — `PENPOT_SECRET_KEY` and `PENPOT_DATABASE_PASSWORD` are required environment variables with no default; compose will refuse to start if they are missing. Generate with `openssl rand -hex 32`.
3. **No insecure flags** — stock compose ships `disable-secure-session-cookies` and `disable-email-verification` in `PENPOT_FLAGS`. Both are removed. Re-enable email verification by configuring SMTP.
4. **No MailCatcher** — stock compose includes a MailCatcher service that captures outbound email (including auth/reset tokens) and exposes a UI on port 1080 with no authentication. Removed entirely.
5. **WS bridge port not published** — Penpot's plugin bridge (port 4402) binds `0.0.0.0` inside the backend container. It is **not published** in the compose `ports:` section. This is the only thing containing it — the code is not localhost-bound — so this port must never be added to a firewall rule or Tailscale ACL that exposes it to the LAN.

## Pre-flight checklist

- [ ] `cp ops/.env.example ops/.env` and fill in `PENPOT_SECRET_KEY` and `PENPOT_DATABASE_PASSWORD`
- [ ] Verify image digests still match `docker manifest inspect <image>:2.16 --verbose | grep Digest`
- [ ] Confirm port 4402 is not exposed in any firewall / VPN ACL
- [ ] Confirm `:1080` (MailCatcher) is not open anywhere
- [ ] If on LAN/Tailscale, confirm the Docker host does not have IP forwarding enabled in a way that routes internal container ports externally

## Updating Penpot

1. Pull the new release notes and check the changelog for schema migrations.
2. Re-verify and update all image digests in the compose file.
3. Update `.env.example` if new required vars appear.
4. `docker compose -f ops/docker-compose.hardened.yaml pull && docker compose up -d`
5. Re-run the security audit protocol for any changed images.

## Accessing the MCP from Claude Code

The Penpot MCP runs as a browser extension, not a standalone server. To use it:

1. Install the Penpot browser extension (bundled with Penpot; load unpacked from `/mcp/plugin` in the monorepo or via the marketplace).
2. Open your self-hosted Penpot instance in Chrome.
3. In Claude Code (or any MCP client), add the Penpot MCP plugin pointing at your instance URL.

The plugin communicates over the WS bridge (port 4402, localhost only). It has no persistent process to manage outside the browser session — it's alive when the browser is open to Penpot, dormant otherwise.
