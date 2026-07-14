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

## Penpot REST API (VALIDATED 2026-07-14)

**Open question resolved: YES, the REST API exposes full file content reads.**

Validated against a live self-hosted Penpot 2.16 instance via cookie-auth session (personal access tokens require an additional feature flag; session auth works for CI via a service account). Key findings:

- `POST /api/rpc/command/login-with-password` → returns session cookie
- `GET /api/rpc/command/get-teams` → returns team list with feature flags per team
- `GET /api/rpc/command/get-project-files?project-id=<id>` → returns file list
- `GET /api/rpc/command/get-file?id=<id>&features=<comma-separated>` → returns **full file data** including `data.pages`, component tree, and token data

The client must declare feature support matching what the file uses (pass all features from `get-teams` response). The API returns Transit+JSON encoding (`~:` keywords, `~u` UUIDs, `~m` timestamps, `~#set` sets) — the ThroughLine skill will need a Transit decoder or must negotiate `application/json` content type.

**Conclusion:** The `token-sync-layer` and `token-crosswalk-builder` skills can be implemented headlessly via REST API without requiring the MCP. The MCP remains useful for interactive/live sessions but is not a blocker for CI.

## Penpot token-sets model (VALIDATED 2026-07-14)

**Open question resolved: Penpot uses token sets + themes, NOT variable modes. The `token-builder` skill must be redesigned.**

Figma uses Variable Modes (a single collection has multiple modes, e.g., `light`/`dark`). ThroughLine's original `token-builder` pulls mode values directly.

Penpot's model is different:

### Token structure (confirmed via REST API on live 2.16 instance)

```
tokens-lib
  sets (ordered-map, Transit ~#ordered-map)
    "S-Global" → token-set named "Global"
      tokens (ordered-map)
        "color.primary" → { type: :color,   value: "#0066FF" }
        "spacing"       → { type: :spacing, value: "16"      }
        "dark"          → { type: :color,   value: "#338AFF", description: "dark" }
  themes (ordered-map)
    "" → {
      "__PENPOT__HIDDEN__TOKEN__THEME__" → token-theme {
        active-sets: ["Global"],
        name: "__PENPOT__HIDDEN__TOKEN__THEME__"
      }
    }
  active-themes: ["/__PENPOT__HIDDEN__TOKEN__THEME__"]
```

### Key differences from Figma modes

| Concept | Figma | Penpot |
|---|---|---|
| Multi-value grouping | Variable Mode (light/dark within one collection) | Token Set (separate named sets) |
| Activation | Switch mode on a collection | Switch active Theme (which activates a set list) |
| Default state | One mode is always active | Hidden default theme activates one set |
| Light/dark pattern | One collection, two modes | Two sets ("Global-light", "Global-dark"), two themes |
| Brand variants | Mode per brand | Set per brand, theme groups them |

### Implications for `token-builder` skill

The skill cannot be ported directly. It must be redesigned to:

1. Read all token sets from `tokens-lib.sets` (ordered-map)
2. Read themes from `tokens-lib.themes` to understand set groupings
3. For each theme, walk the active set list to produce a merged token output
4. Generate one token file per theme (equivalent to one file per mode in the Figma design)
5. Handle Transit+JSON decoding (see below) before any processing

The style-dictionary adapter and typed exports remain unchanged — only the extraction layer differs.

### Transit+JSON decoding requirement

The REST API returns Transit+JSON. All skills that call the REST API must decode the wire format before processing:

| Transit literal | Decoded value |
|---|---|
| `~:keyword` | ClojureScript keyword → strip `~:` prefix |
| `~u<uuid>` | UUID string |
| `~m<ms>` | Timestamp (ms since epoch) |
| `~#set [...]` | Set (deduplicated array) |
| `~#ordered-map [k1,v1,k2,v2,...]` | Order-preserving map (array of alternating k/v) |

A lightweight Transit decoder must be implemented in the `token-sync-layer` skill (or as a shared utility in the monorepo) before token data can be consumed. Alternatively, negotiate `application/json` content type if Penpot supports it — TBD during skill implementation.

## Port sequence

Both open questions are now resolved. Port begins:

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
