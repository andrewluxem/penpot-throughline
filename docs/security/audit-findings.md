# Security Audit Findings

Audited 2026-07-14. Applies Track A protocol (executable software). Re-audit required on any version bump of either project.

---

## Penpot self-host + official MCP

**Verdict: CONDITIONAL (YELLOW) — deploy with hardened compose only**

### CVE status (all fixed at ≤ 2.15.0, clean in 2.16)

| CVE | Severity | Status in 2.16 |
|---|---|---|
| CVE-2026-45805 | HIGH (CVSS 8.8) | Patched in monorepo; standalone repo still vulnerable |
| CVE-2026-44986 | MEDIUM | Fixed |
| CVE-2026-45806 | MEDIUM | Fixed |
| CVE-2026-26202 | LOW | Fixed |

### Critical finding: two MCP codebases, different patch state

**CVE-2026-45805** — unauthenticated MCP REPL endpoint (`/execute`), binds `0.0.0.0`, CVSS 8.8.

- **`penpot/penpot-mcp` (standalone repo)** — still vulnerable as of audit date. The standalone repo markets itself as "official" but does not receive the same patch cadence as the monorepo.
- **`penpotapp/mcp:2.16` (monorepo-bundled)** — patched; REPL binds `localhost` only.

**Action: deploy the 2.16 Docker image only. Never build from the standalone repo.**

### Stock compose issues (all addressed in hardened compose)

| Finding | Severity | Fix |
|---|---|---|
| Default `PENPOT_SECRET_KEY: change-this-insecure-key` | HIGH | Required env var; no default |
| Default `penpot:penpot` DB credentials | HIGH | Required env var; no default |
| `disable-secure-session-cookies` in default flags | MEDIUM | Removed from flags |
| `disable-email-verification` in default flags | MEDIUM | Removed from flags |
| MailCatcher UI on `:1080` (renders auth/reset emails unauthenticated) | HIGH | Service removed |
| No image digest pinning | MEDIUM | All 7 images pinned |
| WS plugin bridge (4402) binds `0.0.0.0` (contained only by unpublished port) | MEDIUM | Port not published; documented constraint |

### WS bridge network constraint

Port 4402 (the plugin WS bridge) binds all interfaces inside the backend container. It is not published in the hardened compose, but this is the only thing containing it — the code is not localhost-bound. This port **must never** be:
- Added to the compose `ports:` section
- Exposed via a firewall rule
- Routed through Tailscale or any VPN ACL

### Community MCP alternative (`montevive/penpot-mcp`) — do not use

- Requires plaintext account password in `.env`
- Defaults to Penpot's cloud API (not your self-hosted instance)
- Prints the live session token to stdout with `DEBUG=true` as default

### Plugin manifest review

The official Penpot browser plugin manifest is least-privilege: it correctly withholds `user:read` and `allow:downloads`. One minor over-scope: `comment:read/write` is declared but has no corresponding code path in the plugin. Severity: LOW; no action required, but worth re-checking if the plugin is updated.

---

## ThroughLine (jrpease/throughline, v0.13.0 / @radicool/throughline)

**Verdict: CONDITIONAL (YELLOW) — leaning APPROVE; one residual finding**

### Installer review (`scripts/install.mjs`)

Clean:
- Zero dependencies, no install hooks
- No `child_process`, network calls, or environment variable reads
- Writes only inside the target project directory
- Guarded against indirect invocation

### Figma token handling

README's claim that the token "stays yours" holds up: the token flows only as environment interpolation into the MCP client config. No repo code reads, stores, or logs it.

### Prompt injection sweep

112 markdown/skill files scanned for embedded instructions and prompt injection. Zero-width character, bidi override, and BOM sweep included. All clean.

### Residual finding: unpinned MCP dependency

**Severity: MEDIUM**

`.mcp.json` originally launched `figma-console-mcp@latest`, which receives the Figma token. `@latest` is unpinned — a malicious publish to that package could silently exfiltrate the token on next install.

**Fixed during audit:** pinned to `figma-console-mcp@1.34.0` (the version `@latest` resolved to at ThroughLine's actual v0.13.0 release date, 2026-07-05). See `.mcp.json.pin-note.md` in the upstream fork for the tarball integrity hash.

**Still outstanding upstream:** adapter configs for Cursor (`cursor/.cursor/mcp.json`), Codex (`codex/codex-mcp.toml`), and generic agents (`generic/AGENTS.md`) still reference `@latest`. These should be pinned to the same `1.34.0` before using those adapters.

### Project maturity note

ThroughLine is a young project (31 stars, single maintainer) as of audit date. Re-audit on any version bump before updating.

---

## Audit methodology

- Static analysis only; no code executed during audit
- All repos cloned read-only to sandbox; no `npm install`, `docker compose up`, or network calls from cloned code
- Evidence chain: 283 source files hashed; chain digest `b1761c8e4750…` (local sandbox only, not committed)
- CVE data verified against live GitHub Security Advisories at audit date
