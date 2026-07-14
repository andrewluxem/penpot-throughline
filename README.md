# penpot-throughline

A port of [ThroughLine](https://github.com/jrpease/throughline) — the Claude Code plugin that syncs design tokens and components from a design tool into a monorepo — to run against self-hosted [Penpot](https://penpot.app) instead of Figma.

**Status: Active development — pre-alpha. Nothing is runnable yet.**

---

## Why

ThroughLine is excellent orchestration for design-to-code token sync. The only Figma-specific layer is the design-tool integration (an MCP bridge + a handful of skills). Penpot is a self-hostable, open-source design canvas that exports DTCG-format tokens natively and ships an official MCP server. Swapping the integration layer removes the Figma license requirement and keeps the entire pipeline on infrastructure you control.

## What changes vs. upstream ThroughLine

Most of ThroughLine's orchestration — the manifest, Style Dictionary adapter, monorepo scaffolding, model routing — is tool-agnostic and carries over unchanged. Only the design-tool integration layer needs porting:

| Upstream skill | Status | Notes |
|---|---|---|
| `environment-setup` | Needs port | Figma token → Penpot access token + MCP config |
| `design-system-audit` | Needs port | Figma MCP calls → Penpot MCP calls |
| `token-builder` / `token-sheet-builder` | Needs port | Pending REST API depth validation (see open questions) |
| `token-sync-layer` / `token-crosswalk-builder` | Needs port | Pending REST API depth + token-set/modes validation |
| `component-builder` / `icon-system-builder` / `component-pipeline` | Needs port | After token layer is stable |
| `retrofit-planner` | Carry over | Tool-agnostic |

See [`docs/architecture/rewrite-plan.md`](docs/architecture/rewrite-plan.md) for the full plan.

## Infrastructure

`ops/docker-compose.hardened.yaml` is a hardened Penpot self-host compose. It:

- Pins all 7 images by digest
- Reads `PENPOT_SECRET_KEY` and `PENPOT_DATABASE_PASSWORD` from the environment (never hardcoded)
- Removes insecure session-cookie and email-verification flags
- Removes the MailCatcher UI port
- Does **not** publish the WS plugin bridge port (port 4402) externally

Copy `ops/.env.example` to `ops/.env`, fill in the required vars, then `docker compose -f ops/docker-compose.hardened.yaml up -d`.

## Security notes

A full security audit of Penpot (self-host + MCP) and the upstream ThroughLine plugin was conducted before this port began. Summary in [`docs/security/audit-findings.md`](docs/security/audit-findings.md).

**Critical:** there are two Penpot MCP codebases. Only the `penpotapp/mcp:2.16` image (which ships inside the main monorepo) is patched for CVE-2026-45805. The standalone `penpot/penpot-mcp` repo on GitHub still carries the vulnerability. **Deploy from this compose; do not build the standalone repo.**

## Open questions (gates for porting token skills)

1. Does Penpot's REST API expose full per-node/variable content reads in headless (non-MCP) mode? Needed for CI-driven token sync.
2. Does Penpot's token-sets model compose light/dark × multi-brand the same way Figma's mode-splitting does?

These will be validated against a live instance before the token skills are ported.

## Contributing

This is an early-stage personal project. Issues and PRs welcome once the port reaches a runnable state.

## License

MIT. Upstream ThroughLine is also MIT.
