# ThroughLine → Penpot: Rewrite Plan

## Goal

Port ThroughLine's design-tool integration layer from Figma to Penpot. The objective is a drop-in replacement for ThroughLine that works against a self-hosted Penpot instance, removing the Figma license dependency while preserving the full token-sync and component-generation pipeline.

## What is ThroughLine?

[ThroughLine](https://github.com/jrpease/throughline) (`@radicool/throughline` v0.13.0) is a Claude Code plugin that:
- Reads design tokens and component specs from a design tool (currently Figma via `figma-console-mcp`)
- Runs them through a Style Dictionary adapter
- Scaffolds a token-driven monorepo with typed exports

It is organized as a set of skills (markdown instruction files for Claude Code) and an MCP configuration that connects Claude to the design tool.

## Architecture split

ThroughLine's skills fall into two categories:

### Tool-agnostic (carry over unchanged)
- `retrofit-planner` — analyzes an existing codebase for token adoption gaps
- The Style Dictionary adapter and manifest structure
- Monorepo scaffolding (token file structure, typed exports, CI hooks)
- Model routing and orchestration

### Design-tool-specific (must be ported)
These are the skills that call Figma MCP APIs directly:

| Skill | What it does | Port complexity |
|---|---|---|
| `penpot-environment-setup` | Installs deps, configures MCP connection | Low — swap Figma token/MCP for Penpot access token + MCP plugin config |
| `design-system-audit` | Reads the design file to inventory tokens, components, styles | Medium — Penpot MCP has analogous read methods; API surface differs |
| `token-builder` | Extracts token values from the design file | High — blocked on REST API depth validation (see open questions) |
| `token-sheet-builder` | Builds a token reference sheet | High — same blocker |
| `token-sync-layer` | CI-driven token sync (headless) | High — MCP is live-session only; CI path requires REST API |
| `token-crosswalk-builder` | Maps Figma tokens to code references | High — same blocker + modes/token-sets question |
| `component-builder` | Generates React components from Figma specs | Medium — after token layer is stable |
| `icon-system-builder` | Builds the icon export pipeline | Medium |
| `component-pipeline` | Orchestrates component generation at scale | Medium |

## Penpot MCP surface (known)

Penpot's official MCP server exposes tools over the WS bridge (requires an open browser session):

- List files and pages in a project
- Read page contents (shapes, layers, components)
- Read design tokens (DTCG format, native)
- Read component library entries

This is sufficient for interactive audit and token reads. It is **not** available headless (no browser session = no MCP). CI-driven sync therefore must go through the REST API.

## Penpot REST API (open question — gates CI path)

Penpot has a public REST API accessible via personal access tokens (Settings → Access Tokens). The open question is whether it exposes **full per-node/variable content reads** — specifically:

- Can it return the complete token value tree for a file (not just metadata)?
- Can it return resolved token aliases across token sets?
- Can it return per-component spec data at the same depth as the MCP?

This must be validated against a live Penpot 2.16 instance before the `token-sync-layer` and `token-crosswalk-builder` skills are ported. If the REST API is shallow, an alternative is to write a Penpot plugin that can be triggered headlessly to dump tokens to a JSON artifact — more complex but feasible.

## Penpot token-sets model (open question — gates token skill design)

Figma's Variable Modes allow a single variable collection to have multiple modes (e.g., `light` and `dark`), and ThroughLine's `token-builder` uses mode references to generate themed token files.

Penpot uses token sets (groups of tokens with optional `$type` tagging). The open question is whether Penpot's token-set composition matches Figma's multi-mode pattern closely enough for ThroughLine's current token-builder design to port directly, or whether the skill needs to be redesigned around set composition.

This must be validated by building a real multi-brand, light/dark test file in Penpot before porting begins.

## Port sequence

Once the two open questions are resolved:

1. `penpot-environment-setup` — low-risk starting point; validates the MCP connection end-to-end
2. `design-system-audit` — validates MCP read depth for the interactive path
3. `token-builder` / `token-sheet-builder` — depends on token-sets validation
4. `token-sync-layer` / `token-crosswalk-builder` — depends on REST API depth + token-sets
5. `component-builder` / `icon-system-builder` / `component-pipeline` — after token layer stable
6. `retrofit-planner` — carry over; minimal changes expected

## MCP configuration

The upstream ThroughLine `.mcp.json` launches `figma-console-mcp`. The Penpot equivalent will configure the `penpotapp/mcp:2.16` plugin instead. The Penpot plugin has no token to configure (it rides the browser session), so the MCP config is simpler:

```json
{
  "mcpServers": {
    "penpot": {
      "command": "penpot-mcp",
      "env": {
        "PENPOT_BASE_URL": "http://localhost:9001"
      }
    }
  }
}
```

Exact invocation TBD after the `penpot-environment-setup` skill is drafted and validated.

## Security constraints carried forward

- All Penpot image versions must be pinned by digest in CI and compose
- `penpot/penpot-mcp` standalone repo must not be used (CVE-2026-45805)
- Port 4402 (WS bridge) must never be published externally
- `montevive/penpot-mcp` (community alternative) must not be used — stores plaintext password, logs session token
- Any skill that handles access tokens must treat them as secrets and never log or embed them
