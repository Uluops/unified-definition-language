# UluOps Workflow Definition Language (WDL) Specification

**Version:** 3.0.0
**Status:** Draft
**Created:** 2026-03-26
**Updated:** 2026-03-26

---

## Overview

The **Workflow Definition Language (WDL)** is a YAML-based specification for defining multi-step orchestrations in the UluOps framework. Workflows compose commands and agents into **phases** with dependencies, conditional execution, gates, and aggregated results.

| Aspect | Description |
|--------|-------------|
| **Format** | YAML with JSON Schema validation |
| **File Extension** | `.workflow.yaml` |
| **Schema URL** | `https://uluops.ai/schemas/wdl/v3.0.0/workflow.json` |
| **Scope** | Multi-step orchestration with phases and gates |

### Design Goals (v3.0.0)

1. **Runtime-only content** — Every field in the YAML produces operational behavior in the runtime markdown. Documentation-only content belongs in external docs.
2. **250-line runtime target** — Generated markdown commands should not exceed 250 lines.
3. **Lean phases** — Phase definitions carry only what the runtime needs: identity, steps, gates, conditions, and focus areas.

---

## Composition Hierarchy

| Level | Definition | Contains | Example |
|-------|------------|----------|---------|
| Agent (ADL) | Atomic validation/execution unit | Scoring, criteria, tasks | `code-validator.agent.yaml` |
| Command (CDL) | Single agent + execution context | Preflight, overrides, output | `validate.command.yaml` |
| **Workflow (WDL)** | **Commands/agents + phases** | **Gates, aggregation, context** | **`ship.workflow.yaml`** |
| Pipeline (PDL) | Multiple workflows + stages | Triggers, approvals, rollback | `release.pipeline.yaml` |

---

## Schema Structure

```yaml
workflow:
  interface:        # Required: Identity and classification
  arguments:        # Optional: CLI arguments
  context:          # Optional: Context detection for conditional phases
  orchestration:    # Required: Phases, dependencies, execution order
  aggregation:      # Optional: Score aggregation and decisions
  outputs:          # Optional: Artifacts, tracking, verification
  preflight:        # Optional: Pre-execution checks
  postflight:       # Optional: Post-execution actions
```

---

## Field Reference

### Interface Block (Required)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Kebab-case identifier |
| `version` | string | Yes | SemVer (e.g., `1.0.0`) |
| `displayName` | string | Yes | Human-readable name (3-50 chars) |
| `description` | string | Yes | Usage + purpose (20-500 chars) |
| `domain` | Domain | Yes | Primary domain |
| `duration` | string | No | Expected duration |
| `model` | Model | No | Default LLM tier (default: `sonnet`) |

### Arguments Block

| Field | Type | Description |
|-------|------|-------------|
| `positional` | PositionalArg[] | Positional arguments in order |
| `flags` | FlagDef[] | Named flags (--flag) |
| `derived` | DerivedVar[] | Computed variables |
| `examples` | UsageExample[] | Usage examples (max 3) |

### Context Block

| Field | Type | Description |
|-------|------|-------------|
| `detectors` | Detector[] | Context detectors run at workflow start |
| `variables` | ContextVariable[] | Derived from detector results |

**Detector check types:** `file_exists`, `dir_exists`, `grep`, `find`, `command`, `json_field`, `yaml_field`, `env_var`

### Orchestration Block (Required)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `phases` | Phase[] | Yes | Phases to execute (min 1) |
| `execution_mode` | ExecutionMode | No | User-selectable strategy |
| `on_failure` | `stop\|continue\|abort` | No | Global failure behavior (default: `stop`) |
| `max_parallel` | integer | No | Max parallel phases (1-10) |
| `timeout` | integer | No | Workflow timeout in ms |

### Phase

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | Yes | Kebab-case identifier |
| `name` | string | Yes | Human-readable name (3-50 chars) |
| `steps` | StepRef[] | Yes* | Steps to execute (command or agent refs) |
| `commands` | CommandRef[] | Yes* | Deprecated: use `steps` |
| `group` | integer | No | Parallel execution group |
| `barrier` | boolean | No | Must pass before next group |
| `depends_on` | string[] | No | Phase IDs that must complete first |
| `condition` | string | No | Expression for conditional execution |
| `skip_if` | string | No | Expression to skip phase |
| `gate` | Gate | No | Threshold and failure behavior |
| `inputs` | object | No | Input mappings for execute phases |
| `timeout` | integer | No | Phase timeout in ms |
| `focus` | string[] | No | Key areas (max 5 items) |
| `type` | PhaseType | No | `validate\|execute\|mixed` (default: `validate`) |

*One of `steps` or `commands` is required.

### Gate

| Field | Type | Description |
|-------|------|-------------|
| `threshold` | integer | Score threshold (0-100) |
| `warn_threshold` | integer | Warning threshold (> threshold) |
| `aggregate` | AggregateMethod | How to combine multi-step scores |
| `on_fail` | GateAction | `stop\|warn\|skip\|abort` |
| `on_warn` | GateAction | Behavior on warning |

### Aggregation Block

| Field | Type | Description |
|-------|------|-------------|
| `score.method` | AggregateMethod | `min\|max\|average\|weighted_average\|all\|any` |
| `score.weights` | object | Phase→weight mapping (sum to 1.0) |
| `score.skip_missing` | boolean | Skip skipped phases in calculation |
| `decision` | object | Label→expression mapping |

### Outputs Block

| Field | Type | Description |
|-------|------|-------------|
| `artifacts` | Artifact[] | Files to generate |
| `variables` | OutputVariable[] | Variables to expose |
| `verification.post_save` | VerifyCheck[] | Post-save integrity checks |
| `tracking` | TrackingConfig | Tracker integration |

### Preflight Block

| Field | Type | Description |
|-------|------|-------------|
| `checks` | PreflightCheck[] | Validation checks before execution |
| `extraction` | PreflightExtraction[] | Metadata gathering before phases |

### Postflight Block

| Field | Type | Description |
|-------|------|-------------|
| `on_pass` | PostflightAction | Action on pass (message + exit_code) |
| `on_warn` | PostflightAction | Action on warning |
| `on_fail` | PostflightAction | Action on failure |

Postflight also accepts custom decision keys (e.g., `on_ship`, `on_hold`).

---

## Migration from v2.0.0

### Removed Blocks

| Block | Migration |
|-------|-----------|
| `handoffs` | Remove entirely. Agent data passing is implicit via phase ordering. |
| `iteration` | Remove entirely. Move to external docs if needed. |
| `troubleshooting` | Remove entirely. Move to external docs if needed. |

### Removed Fields

| Location | Field | Migration |
|----------|-------|-----------|
| `interface` | `philosophy` | Remove |
| `interface` | `legend` | Remove |
| `interface` | `tags` | Remove |
| `interface` | `subdomain` | Remove |
| `interface` | `comparison` | Remove |
| `interface` | `operational` | Remove |
| `orchestration` | `diagram` | Remove |
| `orchestration.execution_mode` | `options` (verbose) | Remove; `prompt` + `default` sufficient |
| `orchestration.execution_mode` | `notes` | Remove |
| `phase` | `capture_note` | Remove; agents capture findings by default |
| `phase` | `if_failing` | Remove; agents have their own failure guidance |
| `phase` | `threshold_rationale` | Remove; docs concern |
| `phase` | `decision_criteria` | Remove; agent-level concern |
| `phase` | `auto_fail_conditions` | Remove; agent-level concern |
| `phase` | `skip_conditions` | Remove; use `condition` field instead |
| `phase` | `alternatives` | Remove; unused in practice |
| `phase` | `key_outputs` | Remove; never materialized |
| `phase` | `inline_checks` | Remove; rarely used |
| `aggregation` | `report_template` | Remove |
| `aggregation` | `decision_table` | Remove; `decision` map sufficient |
| `outputs.verification` | `pre_save` | Remove; post_save only |
| `preflight` | `banner` | Remove; runtime generates status |
| `postflight` action | multiline `message` | Keep, but prefer single line (max 200 chars) |

### Kept with Constraints

| Field | Change |
|-------|--------|
| `arguments.examples` | Max 3 items (was unlimited) |
| `phase.focus` | Max 5 items (was unlimited) |
| `postflight.*.message` | Max 200 chars (was unlimited) |

---

## Revision History

| Version | Date | Changes |
|---------|------|---------|
| 3.0.0 | 2026-03-26 | **Breaking:** Removed handoffs, iteration, troubleshooting blocks. Removed 15 phase fields, 6 interface fields, diagram, banner, decision_table. Added maxItems constraints. Target: 250-line runtime. |
| 2.0.0 | 2026-03-01 | Step references (command/agent), inline_checks, preflightExtraction |
| 1.5.0 | — | Multi-command phases, gates, context detection |
| 1.2.0 | 2026-02-20 | Philosophy, legend, duration, handoffs, iteration, troubleshooting, decision_criteria |
| 1.1.0 | 2026-02-08 | Phase barriers, context variables, output verification |
| 1.0.0 | — | Initial release |
