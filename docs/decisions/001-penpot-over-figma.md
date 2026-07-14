# ADR 001: Use Penpot as the design-tool target instead of Figma

**Date:** 2026-07-14  
**Status:** Accepted

## Context

ThroughLine's upstream implementation uses Figma as its design source-of-truth, accessed via `figma-console-mcp`. The goal of this fork is to remove the Figma license dependency while preserving the full token-sync and component-generation pipeline.

We evaluated three alternatives to Figma:

**Remotion** — ruled out immediately. Different problem domain (video rendering), not a design canvas. Not a substitute for a design source-of-truth.

**Raw JSON token files** — authoring DTCG tokens by hand, without a design canvas. Loses the visual canvas and the ability to inspect components alongside their token values. Eliminates the design-to-code link that ThroughLine is built around.

**Tokens Studio / Supernova / zeroheight** — either proprietary, Figma-dependent, or both. Doesn't solve the license problem.

**Penpot (self-hosted)** — open-source, self-hostable, actively maintained. Has:
- A plugin API and browser extension system
- Native DTCG-format token export
- An official MCP server (merged into the main `penpot/penpot` monorepo as of 2.16)
- A REST API with personal access tokens

## Decision

Use self-hosted Penpot as the design-tool target. Port only the design-tool integration layer (environment-setup, audit, token-builder, sync/crosswalk skills). Carry the tool-agnostic orchestration forward unchanged.

## Consequences

- Removes the Figma license requirement
- Keeps the entire pipeline on self-controlled infrastructure
- Requires validating two open technical questions before the token skills can be ported (see `docs/architecture/rewrite-plan.md`)
- The MCP connection model differs from Figma's: Penpot MCP rides a browser session (no persistent server process), which means CI-driven sync must use the REST API rather than MCP
