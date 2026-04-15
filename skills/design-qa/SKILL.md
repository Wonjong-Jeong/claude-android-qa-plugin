---
description: >
  Compares Figma design specs with Android Composable source code quantitatively
  to verify implementation accuracy. Auto-triggered by requests like
  "run design QA", "compare Figma with code", etc.
allowed-tools:
  - Read
  - Write
  - Glob
  - Bash
  - Agent
---

# Design QA — Quantitative Spec Verification

Parse `$ARGUMENTS` to detect quantitative mismatches between Figma design specs and Android Composable source code.

**Core role**: Split-collect Figma specs by component, compare with source code quantitatively, and generate a report.

**Target**: Jetpack Compose projects only (XML layouts not supported)

---

## Verification Scope

### Verification Targets (11 items)

| # | Category | Comparison | Tolerance |
|---|----------|------------|-----------|
| 1 | Typography | fontSize, fontWeight, fontFamily, lineHeight, letterSpacing | fontSize ±0sp, fontWeight exact |
| 2 | Color | fill, foreground (RGBA → HEX) | HEX exact match |
| 3 | Icon | Icon name matching | Exact match |
| 4 | Size | width, height (fixed values only) | ±1dp |
| 5 | Padding | Internal padding | ±2dp |
| 6 | Spacing | itemSpacing ↔ spacedBy/Spacer | ±2dp |
| 7 | Corner Radius | All or individual corners | ±1dp |
| 8 | Opacity | Node opacity | ±0.05 |
| 9 | Stroke/Border | Width, color | Width ±0dp, color HEX exact |
| 10 | Text content | String content | Exact match |
| 11 | Design token binding | Figma variable ↔ code token | Exact match |

### Exclusions (6 items) — Not compared

| # | Item | Reason |
|---|------|--------|
| 1 | Node hierarchy | Same UI can be built with different view trees |
| 2 | Auto Layout direction/alignment | Figma Frame ≠ Compose layout structure |
| 3 | Constraint relationships | Same visual result achievable with different constraints |
| 4 | Component Variant info | Figma variants ≠ code state management |
| 5 | Shadow / Elevation | Figma shadow ≠ Android elevation rendering |
| 6 | Blur effects | LAYER_BLUR, BACKGROUND_BLUR have no direct Compose equivalent |

---

## File-Based Data Transfer Protocol

Data between sub-agents is passed via **file paths**. Never inline result text in prompts.

```
/tmp/design-qa/<screen_name>/
├── source-analysis.json                    ← source-analyzer-agent output
├── figma-shallow.json                      ← Phase 1A: shallow scan result
├── figma-spec/
│   ├── <component_name_1>.json             ← Phase 1B: per-component deep scan
│   ├── <component_name_2>.json
│   └── ...
├── figma-spec.json                         ← Phase 1C: merged final Figma spec
└── spec-comparison.json                    ← spec-comparator-agent output
```

---

## Parallel Agent Limit

> **Maximum concurrent agents: 3**
>
> Never dispatch more than 3 agents simultaneously in a single response.
> If there are 4+ components, batch them in groups of 3.

---

## Execution Flow

```
Phase 0: Input parsing + work directory creation
    │
Phase 1A: ┌─ Agent(source-analyzer)         ─┐── (parallel, max 2)
           └─ Agent(figma-spec-parser): shallow ─┘
    │         Skill reads results, identifies containers
    │
Phase 1B: Agent(figma-spec-parser) × N ─────── (batched parallel, max 3)
    │         Per-component deep scan
    │
Phase 1C: Skill merges partial results → figma-spec.json
    │
Phase 2: Agent(spec-comparator) ─────────
    │
Phase 3: Report generation (Skill performs directly) ──
    │
Phase 4: Result delivery + user interaction
```

---

## Phase 0 — Input Parsing + Work Directory

### Input Spec

```
screen_name: <screen name>
figma: <Figma frame URL or node ID>
project_root: <project root>          # auto-detected
module_path: <module path>            # default: app
composable_fqn: <FQN>                # optional — when auto-discovery fails
```

**project_root detection**:
```bash
git rev-parse --show-toplevel 2>/dev/null || pwd
```

**Figma URL parsing**: Convert hyphens to colons in `node-id` (`700-11696` → `700:11696`)

### Work Directory

```bash
mkdir -p /tmp/design-qa/<screen_name>/figma-spec
```

---

## Phase 1A — Shallow Scan (Parallel)

> Dispatch source-analyzer and figma-spec-parser **simultaneously**. (2 agents)

### 1A.1 source-analyzer-agent

```
Prompt:
  screen_name, project_root, module_path, composable_fqn
  import_depth: 1                    # direct imports only
  output_dir: /tmp/design-qa/<screen_name>
```

**Returns**: `source_analysis_path` + `error`

### 1A.2 figma-spec-parser-agent (concurrent with 1A.1)

```
Prompt:
  screen_name, figma_node_id: <root frame node ID>
  depth: 1                           # direct children only
  output_path: /tmp/design-qa/<screen_name>/figma-shallow.json
```

**Returns**: `figma_shallow_path` + `child_containers` + `error`

`child_containers` — list of child nodes that need deep scanning:
```json
[
  { "node_id": "700:11700", "name": "TopBar", "type": "FRAME", "child_count": 3 },
  { "node_id": "700:11720", "name": "ContentArea", "type": "FRAME", "child_count": 5 },
  { "node_id": "700:11780", "name": "BottomBar", "type": "FRAME", "child_count": 2 }
]
```

Selection criteria: Nodes with `type` FRAME or AUTO_LAYOUT and `child_count > 0`.
Text, Icon, Vector, and other leaf nodes are fully extracted in the shallow scan.

### 1A.3 Validation + User Interaction

- source-analyzer `error: "composable_not_found"` → **Ask user for FQN**
- figma-spec-parser `error` → Abort QA, inform user
- **Both failed** → Abort QA, report errors to user
- **Success**: Inform user "Found N components. Proceeding with deep analysis..."

---

## Phase 1B — Per-Component Deep Scan (Batched Parallel)

Dispatch figma-spec-parser **individually** for each node in `child_containers`.

> **Batch rule**: Max 3 agents per batch. 4+ components → batch in groups of 3.
> Example: 7 components → Batch 1 (3) → Batch 2 (3) → Batch 3 (1)

Each call:
```
Prompt:
  screen_name, figma_node_id: <component node ID>
  depth: 2                           # depth 2 from this component
  output_path: /tmp/design-qa/<screen_name>/figma-spec/<component_name>.json
```

**Returns**: `output_path` + `error`

Failed components are logged; processing continues with remaining components.

**Rate limit between batches**: Natural delay occurs as Skill reads batch results and prepares next dispatch. No explicit delay needed. If an agent in a batch fails with 429, retry that batch after 3 seconds.

---

## Phase 1C — Figma Spec Merge (Skill Performs Directly)

Skill reads all partial results and merges into a single `figma-spec.json`:

1. Load root frame + leaf node properties from `figma-shallow.json`
2. Load detailed properties from each `figma-spec/<component_name>.json`
3. Replace corresponding children in the root tree with detailed results
4. Save merged result to `/tmp/design-qa/<screen_name>/figma-spec.json`

---

## Phase 2 — Numeric Comparison

### 2.1 spec-comparator-agent

```
Prompt:
  screen_name, project_root,
  source_analysis_path: /tmp/design-qa/<screen_name>/source-analysis.json
  figma_spec_path: /tmp/design-qa/<screen_name>/figma-spec.json
  output_dir: /tmp/design-qa/<screen_name>
```

**Returns**: `spec_comparison_path` + `mapping_rate` + `summary`

### 2.2 Mapping Rate Check

- `mapping_rate < 0.5` → **Inform user of mapping status** + confirm whether to proceed

---

## Phase 3 — Report Generation (Skill Performs Directly)

Read spec-comparator results and generate the report directly.

### 3.1 File Load

- `spec_comparison_path` → element_map, unmatched_figma, unmatched_source, numeric_results, text_issues, icon_issues, branch_match, mapping_rate
- `source_analysis_path` → composable_fqn (metadata)

### 3.2 Issue Aggregation

```
all_issues = numeric_results(fail) + text_issues + icon_issues
```

- **Critical**: Issues with Critical severity
- **Minor**: Issues with Minor severity
- **Pass**: All verification items that passed

### 3.3 Report Writing

Save to `docs/design-qa/<screen_name>.md`.

Report template:

```markdown
# Design QA Report — <screen_name>

> Generated: <date> | Figma nodes: <node_ids> | Method: Quantitative spec comparison

## Summary

| Item | Value |
|------|-------|
| Total verified | N |
| Pass | N |
| Minor | N |
| Critical | N |
| Mapping rate | XX% |

## Critical Issues

| # | Element | Category | Property | Figma | Source | Location |
|---|---------|----------|----------|-------|--------|----------|
| 1 | ... | Size | ... | ... | ... | File.kt:L |

## Minor Issues

(Same format)

## Full Comparison Table

(All numeric_results from spec-comparator)

## Element Mapping

| Figma Node | Source Location | Match Method | Confidence |
|------------|----------------|--------------|------------|

## Unmapped Elements

### Figma only (suspected missing implementation)
### Source only (suspected missing spec)

## Branch State Match

| Matched Branch | Mapping Rate | Location |
|---------------|--------------|----------|

## Conditional Rendering Reference

| Condition | Affected Elements | Verification Status |
|-----------|-------------------|---------------------|
```

Section omission rule: Omit sections with zero issues.

---

## Phase 4 — Result Delivery + User Interaction

Present results to the user:

```
## Design QA Complete — <screen_name>

Method: Quantitative spec comparison
Report: docs/design-qa/<screen_name>.md

### Issue Summary
- Critical: N
- Minor: N
- Pass: N

### Critical Issues
1. [Size] SubmitButton height: Figma 52dp ≠ Source 48.dp (LoginScreen.kt:78)
2. [Color] HeaderText color: Figma #FF0000 ≠ Source #CC0000 (LoginScreen.kt:32)
```

If Critical issues exist → **Ask user whether to proceed with fixes** (possible since this is a Skill).

---

## Error Handling

| Situation | Action |
|-----------|--------|
| source-analyzer failed | Ask user for FQN |
| figma-spec-parser all failed | Report source-only analysis (confidence: LOW) |
| spec-comparator mapping_rate < 50% | Ask user to confirm mapping |
| Both failed | Abort QA, report errors |
| /tmp/design-qa inaccessible | Fallback to project_root/build/design-qa |

---

## Output Policy

- Report: `docs/design-qa/<screen_name>.md`
- All judgments include `confidence` tag
- Pass only **file paths** to sub-agents, never inline result text in prompts
