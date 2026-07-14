# Skill: design-system-audit

Reads a Penpot design file and produces a full structured inventory of its design system: token sets, themes, components, pages, and shared styles. The audit report is consumed by downstream skills (`token-builder`, `component-builder`).

Run this skill after `penpot-environment-setup` has written `.penpot.env`.

---

## What this skill does

1. Loads connection config from `.penpot.env`
2. Authenticates against the Penpot REST API
3. Fetches the full file data via `get-file` with `Accept: application/json` (returns clean DTCG JSON — no Transit decoder needed)
4. Parses and inventories:
   - **Token sets** — every named set and its tokens, grouped by type
   - **Themes** — theme groups and their active-set configurations
   - **Components** — all components in the file's shared library
   - **Pages** — page list with object counts per page
   - **Shared styles** — colors and typography defined at the file level
5. Writes a structured audit report to `design-system-audit.json`

---

## Prerequisites

- `penpot-environment-setup` has been run and `.penpot.env` exists in the project root
- `.penpot.env` contains:
  ```
  PENPOT_BASE_URL=http://localhost:9001
  PENPOT_FILE_ID=<uuid>
  PENPOT_PROJECT_ID=<uuid>
  PENPOT_TEAM_ID=<uuid>
  PENPOT_FEATURES=design-tokens/v1,fdata/path-data,variants/v1,layout/grid,components/v2,fdata/shape-data-type,styles/v2,fdata/objects-map,tokens/numeric-input,render-wasm/v1
  ```
- `python3` is available (no extra packages required — uses stdlib only)
- Cookie session at `/tmp/penpot.cookies` OR `$PENPOT_ACCESS_TOKEN` is set

---

## Step 1 — Load the project config

```bash
set -a && source .penpot.env && set +a
```

Verify the key vars are set:

```bash
echo "Base URL : $PENPOT_BASE_URL"
echo "File ID  : $PENPOT_FILE_ID"
echo "Features : $PENPOT_FEATURES"
```

---

## Step 2 — Authenticate (if session cookie is stale)

If `/tmp/penpot.cookies` exists and is less than 24 hours old, skip this step. Otherwise re-authenticate:

```bash
curl -s -c /tmp/penpot.cookies \
  -X POST "$PENPOT_BASE_URL/api/rpc/command/login-with-password" \
  -H "Content-Type: application/json" \
  -d "{\"email\": \"$PENPOT_EMAIL\", \"password\": \"$PENPOT_PASSWORD\"}" \
  | python3 -c "import sys, json; d=json.load(sys.stdin); print('Authenticated as:', d.get('fullname', d.get('email', 'unknown')))"
```

**Access token alternative (CI):**

```bash
# Replace -b /tmp/penpot.cookies with this header in all curl commands below:
# -H "Authorization: Token $PENPOT_ACCESS_TOKEN"
```

---

## Step 3 — Fetch the full file

This single call returns all data needed for the audit. The `Accept: application/json` header bypasses Transit encoding — Penpot 2.16 returns clean JSON.

```bash
curl -s \
  -b /tmp/penpot.cookies \
  -H "Accept: application/json" \
  "$PENPOT_BASE_URL/api/rpc/command/get-file?id=$PENPOT_FILE_ID&features=$PENPOT_FEATURES" \
  -o /tmp/penpot-file.json
```

Confirm the fetch succeeded and the response is valid JSON:

```bash
python3 -c "
import json, sys
with open('/tmp/penpot-file.json') as f:
    d = json.load(f)
name = d.get('name', '<unnamed>')
pages = len(d.get('data', {}).get('pages', []))
print(f'File: {name}')
print(f'Pages: {pages}')
print('File data fetched OK' if pages > 0 else 'WARNING: no pages found — check FILE_ID and FEATURES')
"
```

---

## Step 4 — Run the audit script

Save the script below as `/tmp/penpot_audit.py`, then run it. It reads `/tmp/penpot-file.json` and writes the structured audit to `design-system-audit.json` in your current directory.

```python
#!/usr/bin/env python3
"""
ThroughLine — Penpot design-system-audit
Parses a get-file JSON response and writes design-system-audit.json.

Input:  /tmp/penpot-file.json   (from Step 3)
Output: ./design-system-audit.json
"""

import json
import sys
from collections import defaultdict
from pathlib import Path

SOURCE = Path("/tmp/penpot-file.json")
OUTPUT = Path("design-system-audit.json")

# ── Load raw file ─────────────────────────────────────────────────────────────

with SOURCE.open() as f:
    raw = json.load(f)

data = raw.get("data", {})
file_meta = {
    "id":   raw.get("id"),
    "name": raw.get("name"),
}

# ── Token sets ────────────────────────────────────────────────────────────────

HIDDEN_THEME = "__PENPOT__HIDDEN__TOKEN__THEME__"

tokens_lib = data.get("tokensLib", {})
raw_sets   = tokens_lib.get("sets", {})
raw_themes = tokens_lib.get("themes", {})

token_sets = {}
for set_name, set_body in raw_sets.items():
    tokens_in_set = set_body.get("tokens", {})
    by_type = defaultdict(list)
    for token_name, token_body in tokens_in_set.items():
        t = token_body.get("$type") or token_body.get("type", "unknown")
        by_type[t].append({
            "name":        token_name,
            "value":       token_body.get("$value") or token_body.get("value"),
            "description": token_body.get("$description") or token_body.get("description"),
        })
    token_sets[set_name] = {
        "token_count": len(tokens_in_set),
        "by_type":     dict(by_type),
    }

# ── Themes ────────────────────────────────────────────────────────────────────

themes = {}
for group_key, group_body in raw_themes.items():
    for theme_name, theme_body in group_body.items():
        if theme_name == HIDDEN_THEME:
            label = "(default / hidden)"
        else:
            label = theme_name
        themes[label] = {
            "group":       group_key or "(default)",
            "active_sets": theme_body.get("activeSets", []),
        }

# ── Pages ─────────────────────────────────────────────────────────────────────

pages_list  = data.get("pages", [])
pages_index = data.get("pagesIndex", {})

pages = []
for page_id in pages_list:
    page_data = pages_index.get(page_id, {})
    objects   = page_data.get("objects", {})
    pages.append({
        "id":           page_id,
        "name":         page_data.get("name", page_id),
        "object_count": len(objects),
    })

# ── Components ────────────────────────────────────────────────────────────────

components_raw = data.get("components", {})
components = []
for comp_id, comp in components_raw.items():
    components.append({
        "id":       comp_id,
        "name":     comp.get("name"),
        "path":     comp.get("path", ""),
        "modified": comp.get("modifiedAt"),
    })
components.sort(key=lambda c: (c["path"] or "", c["name"] or ""))

# ── Shared styles — colors ────────────────────────────────────────────────────

colors_raw = data.get("colors", {})
colors = []
for color_id, color in colors_raw.items():
    colors.append({
        "id":    color_id,
        "name":  color.get("name"),
        "value": color.get("color") or color.get("value"),
        "path":  color.get("path", ""),
    })
colors.sort(key=lambda c: (c["path"] or "", c["name"] or ""))

# ── Shared styles — typography ─────────────────────────────────────────────────

typography_raw = data.get("typographies", {})
typography = []
for typo_id, typo in typography_raw.items():
    fonts = typo.get("fonts", [])
    primary_font = fonts[0] if fonts else {}
    typography.append({
        "id":          typo_id,
        "name":        typo.get("name"),
        "font_family": primary_font.get("fontFamily"),
        "font_weight": primary_font.get("fontWeight"),
        "font_size":   primary_font.get("fontSize"),
        "path":        typo.get("path", ""),
    })
typography.sort(key=lambda t: (t["path"] or "", t["name"] or ""))

# ── Assemble audit report ─────────────────────────────────────────────────────

audit = {
    "meta": {
        "generated_by":  "ThroughLine / design-system-audit",
        "penpot_file_id": file_meta["id"],
        "penpot_file_name": file_meta["name"],
    },
    "summary": {
        "token_set_count":   len(token_sets),
        "theme_count":       len(themes),
        "page_count":        len(pages),
        "component_count":   len(components),
        "color_count":       len(colors),
        "typography_count":  len(typography),
        "total_token_count": sum(s["token_count"] for s in token_sets.values()),
    },
    "token_sets": token_sets,
    "themes":     themes,
    "pages":      pages,
    "components": components,
    "colors":     colors,
    "typography": typography,
}

with OUTPUT.open("w") as f:
    json.dump(audit, f, indent=2, default=str)

# ── Print summary ─────────────────────────────────────────────────────────────

s = audit["summary"]
print(f"\n{'='*56}")
print(f"  ThroughLine — Design System Audit")
print(f"  File: {file_meta['name']}")
print(f"{'='*56}")
print(f"  Token sets    : {s['token_set_count']:>4}  ({s['total_token_count']} tokens total)")
print(f"  Themes        : {s['theme_count']:>4}")
print(f"  Pages         : {s['page_count']:>4}")
print(f"  Components    : {s['component_count']:>4}")
print(f"  Colors        : {s['color_count']:>4}")
print(f"  Typography    : {s['typography_count']:>4}")
print(f"{'='*56}\n")

print("Token sets:")
for set_name, set_info in token_sets.items():
    types = ", ".join(
        f"{t}: {len(tokens)}"
        for t, tokens in set_info["by_type"].items()
    )
    print(f"  [{set_name}]  {set_info['token_count']} tokens  ({types})")

print("\nThemes:")
for theme_label, theme_info in themes.items():
    sets = ", ".join(theme_info["active_sets"]) or "(none)"
    print(f"  [{theme_label}]  active sets: {sets}")

print("\nPages:")
for page in pages:
    print(f"  {page['name']}  ({page['object_count']} objects)")

if components:
    print(f"\nComponents ({len(components)}):")
    for comp in components[:20]:
        prefix = f"{comp['path']}/" if comp["path"] else ""
        print(f"  {prefix}{comp['name']}")
    if len(components) > 20:
        print(f"  ... and {len(components) - 20} more (see design-system-audit.json)")

if colors:
    print(f"\nColors ({len(colors)}):")
    for color in colors[:10]:
        prefix = f"{color['path']}/" if color["path"] else ""
        print(f"  {prefix}{color['name']}  {color['value']}")
    if len(colors) > 10:
        print(f"  ... and {len(colors) - 10} more (see design-system-audit.json)")

if typography:
    print(f"\nTypography ({len(typography)}):")
    for typo in typography:
        print(f"  {typo['name']}  {typo['font_family']} {typo['font_weight']} / {typo['font_size']}")

print(f"\nAudit written to: {OUTPUT.resolve()}\n")
```

Run it:

```bash
python3 /tmp/penpot_audit.py
```

---

## Step 5 — Inspect the audit report

The script prints a summary to stdout and writes the full report to `design-system-audit.json`.

### Reading specific sections

```bash
# Token set names and counts
python3 -c "
import json
with open('design-system-audit.json') as f:
    d = json.load(f)
for name, info in d['token_sets'].items():
    print(f\"{name}: {info['token_count']} tokens\")
"

# Active-set configuration for each theme
python3 -c "
import json
with open('design-system-audit.json') as f:
    d = json.load(f)
for label, theme in d['themes'].items():
    print(f\"{label}: {theme['active_sets']}\")
"

# Component paths (for component-builder)
python3 -c "
import json
with open('design-system-audit.json') as f:
    d = json.load(f)
for comp in d['components']:
    prefix = comp['path'] + '/' if comp['path'] else ''
    print(f\"{prefix}{comp['name']}\")
"
```

---

## Output format

`design-system-audit.json` has this top-level shape:

```json
{
  "meta": {
    "generated_by": "ThroughLine / design-system-audit",
    "penpot_file_id": "<uuid>",
    "penpot_file_name": "My Design System"
  },
  "summary": {
    "token_set_count": 2,
    "theme_count": 2,
    "page_count": 3,
    "component_count": 47,
    "color_count": 12,
    "typography_count": 5,
    "total_token_count": 84
  },
  "token_sets": {
    "Global": {
      "token_count": 42,
      "by_type": {
        "color":   [{ "name": "color.primary", "value": "#0066FF", "description": null }],
        "spacing": [{ "name": "spacing.sm",    "value": "8",        "description": null }]
      }
    }
  },
  "themes": {
    "(default / hidden)": {
      "group": "(default)",
      "active_sets": ["Global"]
    },
    "dark": {
      "group": "",
      "active_sets": ["Global", "Global-dark"]
    }
  },
  "pages": [
    { "id": "<uuid>", "name": "Design", "object_count": 312 }
  ],
  "components": [
    { "id": "<uuid>", "name": "Button/Primary", "path": "Button", "modified": "..." }
  ],
  "colors": [
    { "id": "<uuid>", "name": "Brand/Primary", "value": "#0066FF", "path": "Brand" }
  ],
  "typography": [
    { "id": "<uuid>", "name": "Heading 1", "font_family": "Inter", "font_weight": "700", "font_size": "32", "path": "" }
  ]
}
```

### Key fields consumed by downstream skills

| Field | Consumed by |
|---|---|
| `token_sets` | `token-builder` — reads token names, values, types per set |
| `themes[*].active_sets` | `token-builder` — determines which sets to merge per theme |
| `components` | `component-builder` — uses name and path to scaffold React components |
| `colors` + `typography` | `token-crosswalk-builder` — maps Penpot styles to code references |

---

## Troubleshooting

**`json.JSONDecodeError` on `/tmp/penpot-file.json`**

The file contains a Transit-encoded response instead of JSON. This means the `Accept: application/json` header was not sent or was ignored. Confirm your curl command includes `-H "Accept: application/json"` and that you are running Penpot 2.16 or later. Earlier versions do not support the plain-JSON accept type.

**`token_sets` is empty (`{}`) but tokens exist in the editor**

The `design-tokens/v1` feature is not in the `features=` query parameter. Ensure `PENPOT_FEATURES` includes `design-tokens/v1` and re-run Step 3. You can verify by checking the raw file:

```bash
python3 -c "
import json
with open('/tmp/penpot-file.json') as f:
    d = json.load(f)
print(list(d.get('data', {}).get('tokensLib', {}).get('sets', {}).keys()))
"
```

**`components` is empty but the file has components**

The component library may be in a separate shared library file. Penpot stores local components under `data.components` but externally linked libraries under `data.libraries`. Run the same audit against the library file's ID to capture those components.

**`feature-not-supported` error from `get-file`**

The file uses a feature not declared in `PENPOT_FEATURES`. Fetch the team's feature list and diff it against your env:

```bash
curl -s -b /tmp/penpot.cookies \
  -H "Accept: application/json" \
  "$PENPOT_BASE_URL/api/rpc/command/get-teams" \
  | python3 -c "
import json, sys
teams = json.load(sys.stdin)
for t in teams:
    print(t.get('name'), ':', t.get('features', []))
"
```

Add any missing features to `PENPOT_FEATURES` in `.penpot.env`, then re-run `source .penpot.env` and Step 3.

**`pages` is populated but `object_count` is 0 for all pages**

The `get-file` response omits page object data unless `fdata/objects-map` is in the features list. Ensure that feature is present in `PENPOT_FEATURES`.

**`themes` shows only `(default / hidden)` and no named themes**

The file is using a single default token set with no explicit themes configured. This is normal for files early in token adoption. The hidden theme activates all sets by default. Proceed to `token-builder` — it will merge all sets as a single output.

**Stale cookie (`401` on `get-file`)**

Re-run Step 2 to refresh the session. If using access token auth, confirm `PENPOT_ACCESS_TOKEN` is set and swap `-b /tmp/penpot.cookies` for `-H "Authorization: Token $PENPOT_ACCESS_TOKEN"` in the curl command.

---

## Validation checklist

- [ ] `.penpot.env` sourced; `$PENPOT_FILE_ID` is non-empty
- [ ] `/tmp/penpot-file.json` is valid JSON (not Transit)
- [ ] Audit script runs without errors
- [ ] `token_set_count` matches the number of sets visible in the Penpot editor (Assets → Tokens)
- [ ] `component_count` matches the count visible in Assets → Components
- [ ] `design-system-audit.json` written to project root
- [ ] Downstream skill (`token-builder`) can read the report: `python3 -c "import json; d=json.load(open('design-system-audit.json')); print(list(d['token_sets'].keys()))"`

---

## What's next

Pass `design-system-audit.json` to the `token-builder` skill. That skill reads `token_sets` and `themes` to generate one DTCG token file per theme (one file per active-set combination), then runs Style Dictionary to produce typed exports.

Key inputs `token-builder` expects from this report:
- `token_sets` — the flat token values per set
- `themes[*].active_sets` — the merge order for each theme (later sets override earlier ones)
- `summary.total_token_count` — used as a sanity check after generation
