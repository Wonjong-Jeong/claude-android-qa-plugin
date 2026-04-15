# claude-android-qa-plugin

Quantitatively compares Figma design specs with Android Composable source code to verify implementation accuracy.

---

## Installation

### Method 1 — Claude Code CLI

```bash
claude plugin install https://github.com/Wonjong-Jeong/claude-android-qa-plugin
```

### Method 2 — Marketplace

**1. Register marketplace** (one-time):

```
/plugin marketplace add https://github.com/Wonjong-Jeong/claude-android-qa-plugin
```

**2. Install plugin**:

```
/plugin install claude-android-qa-plugin@claude-android-qa-plugin
```

### Method 3 — settings.json (auto-activate)

Add the following to `~/.claude/settings.json` to load automatically in all sessions:

```json
{
  "extraKnownMarketplaces": {
    "claude-android-qa-plugin": {
      "source": {
        "source": "github",
        "repo": "Wonjong-Jeong/claude-android-qa-plugin"
      }
    }
  },
  "enabledPlugins": {
    "claude-android-qa-plugin@claude-android-qa-plugin": true
  }
}
```

### Method 4 — Local clone

```bash
git clone https://github.com/Wonjong-Jeong/claude-android-qa-plugin.git
claude --plugin-dir ./claude-android-qa-plugin
```

---

## Tools

### Skill

| Skill | Description |
|-------|-------------|
| `/design-qa` | Entry point for design QA — orchestrates agents, merges results, and generates the final report |

### Agents

| Agent | Description |
|-------|-------------|
| `figma-spec-parser-agent` | Extracts quantitative design spec data from Figma via MCP (supports shallow and deep scan modes) |
| `source-analyzer-agent` | Analyzes Android Composable source code to extract structure, values, tokens, and conditional branches |
| `spec-comparator-agent` | Performs element mapping and quantitative numeric verification between Figma spec and source analysis results |

---

## Pipeline

```
/design-qa (Skill — orchestration + report)
    │
Phase 0: Input parsing + work directory
    │
Phase 1: ┌─ figma-spec-parser-agent ─┐── (parallel)
          └─ source-analyzer-agent    ─┘
    │
Phase 2: spec-comparator-agent ────────
    │
Phase 3: Report generation (Skill) ────
    │
Output: docs/design-qa/<screen_name>.md
```

---

## Usage

### Required Environment

- Claude Code (latest version)
- Figma Desktop with MCP server enabled

### Input Example

```
screen_name: LoginScreen
figma: https://www.figma.com/design/XXXX/File?node-id=700-11696
project_root: /path/to/project
module_path: feature/auth
```

- `project_root` is auto-detected via `git rev-parse --show-toplevel` if omitted.
- `module_path` defaults to `app` if omitted.
- `composable_fqn` can be specified directly when auto-discovery fails.

### Output

A markdown report saved to `docs/design-qa/<screen_name>.md` containing:
- Summary table (total verified, pass, minor, critical, mapping rate)
- Critical and minor issue lists with file locations
- Full comparison table for all verified properties
- Element mapping and unmapped element details

---

## Verification Targets

| # | Category | Comparison | Tolerance |
|---|----------|------------|-----------|
| 1 | Typography | fontSize, fontWeight, fontFamily, lineHeight, letterSpacing | fontSize ±0sp, fontWeight exact |
| 2 | Color | fill, foreground (RGBA to HEX) | HEX exact match |
| 3 | Icon | Icon name matching | Exact match |
| 4 | Size | width, height (fixed values only) | ±1dp |
| 5 | Padding | Internal padding | ±2dp |
| 6 | Spacing | itemSpacing vs spacedBy/Spacer | ±2dp |
| 7 | Corner Radius | All or individual corners | ±1dp |
| 8 | Opacity | Node opacity | ±0.05 |
| 9 | Stroke/Border | Width, color | Width ±0dp, color HEX exact |
| 10 | Text content | String content | Exact match |
| 11 | Design token binding | Figma variable vs code token | Exact match |

---

## Exclusions

Node hierarchy, Auto Layout direction/alignment, Constraint relationships, Component Variant info, Shadow/Elevation, Blur effects (LAYER_BLUR, BACKGROUND_BLUR)

---

## Figma MCP Setup

Enable the MCP server in Figma Desktop: **Settings > Enable Developer Tools > MCP Server**.

---

## Requirements

| Requirement | Description |
|-------------|-------------|
| Claude Code | Latest version |
| Figma Desktop | With MCP server enabled |
| Figma MCP | Required for Figma spec extraction via `get_design_context` and `get_variable_defs` |
