# UluOps Command Definition Language (CDL) Specification

**Version:** 2.0.0
**Status:** Draft
**Created:** 2026-01-13
**Updated:** 2026-03-01

---

## Table of Contents

- [Overview](#overview)
- [Design Principles](#design-principles)
- [Relationship to Other Definition Languages](#relationship-to-other-definition-languages)
- [Command Types](#command-types)
- [Schema Structure](#schema-structure)
  - [Agent Command](#agent-command)
  - [Utility Command](#utility-command)
  - [Custom Command](#custom-command)
- [Detailed Field Reference](#detailed-field-reference)
  - [Interface Block](#interface-block)
  - [Invokes Block](#invokes-block)
  - [Execution Block](#execution-block)
  - [Preflight Block](#preflight-block)
  - [Overrides Block](#overrides-block)
  - [Arguments Block](#arguments-block)
  - [Iteration Block](#iteration-block)
  - [Inputs Block](#inputs-block)
  - [Postflight Block](#postflight-block)
  - [Tracking Block](#tracking-block)
  - [Output Block](#output-block)
- [Composition Rules](#composition-rules)
- [Runtime Behavior](#runtime-behavior)
- [Examples](#examples)
- [JSON Schema Reference](#json-schema-reference)
- [Migration from v1.3.0](#migration-from-v130)
- [Revision History](#revision-history)
- [Appendix: Type Definitions](#appendix-type-definitions)

---

## Overview

The **Command Definition Language (CDL)** is a YAML-based specification for defining execution commands in the UluOps validation framework. Commands provide execution context for agent invocations, procedural workflows, and iterative remediation loops.

### Key Characteristics

| Aspect | Description |
|--------|-------------|
| **Format** | YAML with JSON Schema validation |
| **File Extension** | `.command.yaml` |
| **Schema URL** | `https://uluops.ai/schemas/cdl/v2.0.0/command.json` |
| **Scope** | Single command with type-specific behavior |

### What Commands Do

Commands serve as the **invocation layer** between users and agents (or procedural steps):

1. **Wrap a single agent** with execution context (`agent` type)
2. **Define procedural workflows** without agent invocation (`utility` type)
3. **Perform preflight checks** before execution
4. **Override agent defaults** (model, thresholds, timeouts)
5. **Manage iterative loops** until quality thresholds are met
6. **Format results** for output
7. **Execute postflight actions** based on results

### What Commands Do NOT Do

Commands are deliberately focused:

- **No business logic** -- that belongs in agents
- **No multi-agent orchestration** -- that belongs in workflows
- **No phase sequencing** -- that belongs in workflows
- **No conditional agent selection** -- that belongs in workflows

---

## Design Principles

### 1. One Command, One Purpose

Each command has a single purpose determined by its `commandType`. Agent-backed commands wrap exactly one agent. Utility commands execute procedural steps directly.

```yaml
# Agent: One agent, any agent type
invokes:
  agent: code-validator@>=1.0.0

# Utility: No agent, procedural steps
# (invokes block omitted entirely)
```

### 2. Thin Wrapper Philosophy

Commands add context but not logic. The agent contains all validation/execution logic; the command provides the invocation envelope.

```yaml
# Command: Execution context
execution:
  model: opus
  timeout: 600000

# Not in Command: Validation logic (belongs in agent)
scoring:
  categories:
    - name: Code Quality
      weight: 30
```

### 3. Override, Don't Replace

Commands can override agent defaults but cannot fundamentally alter agent behavior.

```yaml
# Override: Threshold adjustment
overrides:
  threshold: 85  # Stricter than agent's default 70
```

### 4. Consistent Interface Pattern

All commands use the same `interface` block structure as Agents (ADL) for consistency across the definition language family.

```yaml
command:
  interface:
    name: validate
    version: "1.0.0"
    displayName: Code Validator
    commandType: agent
    description: Run code validation on target directory
    domain: software
```

### 5. Agent-Type Agnostic

The `agent` command type wraps any ADL agent regardless of the agent's own `agentType` (validator, executor, analyst, generator, explorer, forecaster). The agent's type determines output semantics — the command doesn't need to know or duplicate that classification.

---

## Relationship to Other Definition Languages

```
                    ADL (Agent Definition)
                           |
                           | wrapped by
                           v
                    CDL (Command Definition)  <-- YOU ARE HERE
                         /   \
                        v     v
              (commands)    (agents directly)
                         \   /
                    WDL (Workflow Definition)
                           |
                           | staged by
                           v
                    PDL (Pipeline Definition)
```

### Composition Hierarchy

| Level | Definition | Contains | Example |
|-------|------------|----------|---------|
| Agent (ADL) | Atomic validation/execution unit | Scoring, criteria, tasks | `code-validator.agent.yaml` |
| **Command (CDL)** | **Single command + execution context** | **Preflight, overrides, iteration** | **`validate.command.yaml`** |
| Workflow (WDL) | Commands and/or agents + phases | Gates, aggregation | `ship.workflow.yaml` |
| Pipeline (PDL) | Multiple workflows + stages | Triggers, approvals | `release.pipeline.yaml` |

### Why Commands Exist

Commands solve the **invocation context problem**:

- **Agents** define *what* to check/do
- **Commands** define *how* to invoke (which model, what preflight checks, how to present results)
- **Workflows** define *when* and *in what order* to run commands or agents

Without commands, every workflow would need to repeat execution context for each agent invocation. However, when no execution context is needed (no preflight, no overrides, no tracking), workflows can reference agents directly.

---

## Command Types

CDL v2.0.0 defines three command types, replacing the four types from v1.3.0.

| Type | Purpose | Agent? | Key Feature |
|------|---------|--------|-------------|
| `agent` | Wraps any ADL agent with execution context | Yes | Agent-type-aware decisions |
| `utility` | Procedural steps, no agent | No | Agent-less execution |
| `custom` | Escape hatch for advanced use cases | Optional | All fields optional |

### Type-Specific Requirements

| Field | `agent` | `utility` | `custom` |
|-------|:-------:|:---------:|:--------:|
| `interface.commandType` | Required | Required | Required |
| `invokes` | Required | Forbidden | Optional |
| `iteration` | Optional | Forbidden | Optional |
| `overrides.threshold` | Meaningful | Forbidden | Optional |

### Decision Vocabulary

The decision vocabulary depends on the wrapped agent's `agentType`, not the command type:

| Agent Type | Success | Warning | Failure |
|-----------|---------|---------|---------|
| Validator | `on_success` (PASS) | -- | `on_failure` (FAIL) |
| Executor | `on_success` (COMPLETE) | `on_partial` | `on_failure` (FAILED) |
| Analyst | `on_success` | -- | `on_failure` |
| Generator | `on_success` | `on_partial` | `on_failure` |
| Explorer | `on_success` | -- | `on_failure` |
| Forecaster | `on_success` | -- | `on_failure` |
| Utility (no agent) | `on_success` | -- | `on_failure` |

### When to Use Each Type

- **`agent`**: Wraps any ADL agent (validator, executor, analyst, generator, explorer, forecaster) with execution context. The agent's own `agentType` determines output semantics. Examples: `/agents:validate` (validator), `/agents:prompt-creator` (generator), `/fix:test-gaps` (validator + iteration).

- **`utility`**: No agent involved. The command defines procedural steps executed directly. Examples: `/commit-push`, `/issues:check-resolved`.

- **`custom`**: All fields optional. Escape hatch for advanced use cases that don't fit the agent/utility model. Use sparingly.

### Why v1.3.0's Four Types Became Three

CDL v1.3.0 had `validator`, `executor`, `utility`, and `fixer` types. This created coupling — the command type mirrored the agent's classification:

| Problem | Impact |
|---------|--------|
| Adding a new agent type (e.g., `forecaster`) would require a new CDL command type | Maintenance burden |
| `fixer` was a `validator` + `iteration`, not a distinct invocation pattern | Unnecessary complexity |
| `validator` vs `executor` distinction duplicated information already in the agent's ADL | Redundancy |

v2.0.0 collapses `validator`/`executor`/`fixer` into `agent`. The agent's own `agentType` (from ADL) determines scoring semantics and decision vocabulary. Iteration is available to any `agent` command, not just fixers.

---

## Schema Structure

### Agent Command

Commands that wrap any ADL agent:

```yaml
command:
  interface:
    name: validate
    version: "1.0.0"
    displayName: Code Validator
    commandType: agent
    description: Run code validation on target directory. Blocks progression if critical issues found.
    domain: software
    subdomain: quality

  invokes:
    agent: code-validator@>=1.0.0

  execution:
    model:
      default: sonnet
      allowed: [haiku, sonnet, opus]
    timeout: 300000

  preflight:
    checks:
      - check: path_exists
        path: $ARGUMENTS
        message: "Target directory not found"
    banner:
      template: "Running code validation on $ARGUMENTS..."

  overrides:
    threshold: 70

  postflight:
    steps:
      - action: persist_to_tracker
        required: true
        on_failure: block
      - action: verify_saved
        required: true
        on_failure: warn
      - action: present_summary
        required: true
    on_success:
      message: "Validation passed"
      exit_code: 0
    on_failure:
      message: "Validation failed - fix critical issues"
      exit_code: 1

  output:
    schema: validator-output-v1.1
    format: structured
```

### Utility Command

Commands that execute procedural steps without agent invocation:

```yaml
command:
  interface:
    name: commit-push
    version: "1.0.0"
    displayName: Commit and Push
    commandType: utility
    description: Analyzes changes, checks for resolved validation issues, recommends conventional commit message, and executes git commit/push with rebase.
    domain: software
    subdomain: scm
    tags:
      - utility
      - git
      - commit

  # No invokes block -- utility commands have no agent

  execution:
    model:
      default: sonnet
    timeout: 120000

  preflight:
    checks:
      - check: command
        command: git rev-parse --is-inside-work-tree
        message: Not inside a git repository
        on_fail: abort

  tracking:
    enabled: true
    required: false
    variant: custom
    custom:
      persistence_tool: mcp__uluops-tracker__update_status
      field_mappings:
        project: project_name
        status: completed

  postflight:
    on_success:
      message: "Commit and push completed successfully"
      exit_code: 0
    on_failure:
      message: "Commit/push failed"
      exit_code: 1

  output:
    schema: executor-output-v1.0
    format: summary
```

### Custom Command

Commands with no structural requirements beyond the interface block:

```yaml
command:
  interface:
    name: experimental-pipeline
    version: "0.1.0"
    displayName: Experimental Pipeline
    commandType: custom
    description: Experimental command with custom execution semantics
    domain: software

  # All other blocks are optional for custom type
  invokes:
    agent: some-agent@latest

  execution:
    model:
      default: sonnet
```

---

## Detailed Field Reference

### Interface Block

Identity and classification for the command. Mirrors ADL's interface block for consistency.

```yaml
interface:
  name: string              # Required: Kebab-case identifier
  version: string           # Required: SemVer (e.g., "1.0.0")
  displayName: string       # Required: Human-readable name
  commandType: CommandType  # Required: Command type (v2.0.0)
  description: string       # Required: When to use + what it does (20-500 chars)
  domain: Domain            # Required: Primary domain
  subdomain: string         # Optional: Domain refinement
  tags: string[]            # Optional: Searchable tags
```

**Field Details:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Kebab-case identifier (e.g., `validate`, `commit-push`) |
| `version` | string | Yes | Semantic version (e.g., `1.0.0`) |
| `displayName` | string | Yes | Human-readable name (3-50 chars) |
| `commandType` | CommandType | Yes | Command type: `agent`, `utility`, `custom` (v2.0.0) |
| `description` | string | Yes | Usage description (20-500 chars) |
| `domain` | Domain | Yes | Primary domain classification |
| `subdomain` | string | No | Optional refinement of domain |
| `tags` | string[] | No | Searchable tags for discovery |

**Example:**

```yaml
interface:
  name: audit
  version: "1.0.0"
  displayName: Code Auditor
  commandType: agent
  description: Deep runtime correctness audit. Catches async bugs, null dereferences, silent failures. Use as FINAL gate before ship. Opus model for deeper reasoning.
  domain: software
  subdomain: safety
  tags: [audit, runtime, async, null-safety]
```

---

### Invokes Block

Specifies which agent this command wraps.

```yaml
invokes:
  agent: AgentRef           # Required: Agent reference (name@version)
```

**Conditional Requirement:**

| Command Type | `invokes` Required? |
|-------------|---------------------|
| `agent` | Yes |
| `utility` | No (must be omitted) |
| `custom` | No (optional) |

**Agent Reference Format:**

| Format | Meaning | Example |
|--------|---------|---------|
| `name@1.0.0` | Exact version | `code-validator@1.0.0` |
| `name@>=1.0.0` | Minimum version | `code-validator@>=1.0.0` |
| `name@^1.0.0` | Compatible with 1.x.x | `code-validator@^1.0.0` |
| `name@~1.0.0` | Compatible with 1.0.x | `code-validator@~1.0.0` |
| `name@latest` | Latest published version | `code-validator@latest` |

**Example:**

```yaml
invokes:
  agent: code-validator@>=1.2.0
```

---

### Execution Block

Runtime configuration for agent invocation.

```yaml
execution:
  model:
    default: Model          # Required: Default model
    allowed: Model[]        # Optional: Allowed model overrides
  timeout: number           # Optional: Timeout in milliseconds
  max_retries: number       # Optional: Retry count on failure
  retry_delay: number       # Optional: Delay between retries (ms)
```

**Field Details:**

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `model.default` | Model | Yes | -- | Default model for invocation |
| `model.allowed` | Model[] | No | `[default]` | Models users can override to |
| `timeout` | number | No | `300000` | Timeout in milliseconds |
| `max_retries` | number | No | `0` | Number of retry attempts |
| `retry_delay` | number | No | `1000` | Delay between retries (ms) |

**Model Values:**

| Alias | Canonical | Use Case |
|-------|-----------|----------|
| `haiku` | `anthropic/claude-haiku-4-5` | Fast, simple validations |
| `sonnet` | `anthropic/claude-sonnet-4-6` | Standard validations |
| `opus` | `anthropic/claude-opus-4-6` | Deep reasoning, security |

---

### Preflight Block

Checks to run before execution.

```yaml
preflight:
  checks: PreflightCheck[]  # Optional: List of checks
  banner:
    template: string        # Optional: Display banner template
```

**PreflightCheck Types:**

| Check Type | Fields | Description |
|------------|--------|-------------|
| `path_exists` | `path` | Verify directory exists |
| `file_exists` | `path`, `required` | Verify file exists |
| `command` | `command`, `expect_output`, `expect_empty` | Run shell command |
| `git_clean` | -- | Verify clean git working directory |
| `env_var` | `name`, `required` | Verify environment variable |

**Check Failure Actions:**

| Action | Behavior |
|--------|----------|
| `abort` | Stop command execution, return error (default) |
| `warn` | Log warning, continue execution |
| `skip` | Skip remaining checks, continue execution |

---

### Overrides Block

Override agent default values for this command invocation.

```yaml
overrides:
  threshold: number         # Optional: Override decision threshold
  timeout: number           # Optional: Override agent timeout
  temperature: number       # Optional: Override model temperature
```

**Applicability by Command Type:**

| Command Type | `overrides` |
|-------------|-------------|
| `agent` | Meaningful (threshold semantics depend on agent type) |
| `utility` | Forbidden (no agent to override) |
| `custom` | Optional |

---

### Arguments Block

Configuration for command arguments, including usage examples for documentation.

```yaml
arguments:
  examples: string[]          # Optional: Example argument values (1-5 items)
```

**Purpose:**

The `arguments.examples` field provides custom usage examples that appear in generated command documentation. When not specified, generic examples (`./src`, `./lib`, `.`) are used.

---

### Iteration Block

Configuration for commands that run iterative review-fix-review loops. Available for `agent` and `custom` command types.

```yaml
iteration:
  max_attempts: number      # Required: Maximum iterations before escalation
  exit_threshold: number    # Required: Score that ends the loop
  target_agent: AgentRef    # Optional: Agent to re-run each cycle (defaults to invokes.agent)
  fix_patterns: boolean     # Optional: Include fix pattern guidance (default: true)
```

**Field Details:**

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `max_attempts` | integer | Yes | -- | Maximum fix-review iterations (1-20) |
| `exit_threshold` | number | Yes | -- | Score >= this value ends the loop (0-100) |
| `target_agent` | AgentRef | No | `invokes.agent` | Agent to re-run each cycle |
| `fix_patterns` | boolean | No | `true` | Whether to include fix pattern guidance |

**Applicability:**

| Command Type | `iteration` Available? |
|-------------|----------------------|
| `agent` | Yes (optional) |
| `utility` | No |
| `custom` | Yes (optional) |

**Iteration Flow:**

```
  +----------------+
  |   Run Review   |<-----------+
  +-------+--------+            |
          |                     |
          v                     |
    +-----------+     +--------+-------+
    | APPROVED  |     |    IMPROVE     |
    +-----+-----+     +--------+-------+
          |                     |
          v                     v
       [DONE]            [APPLY FIX]----+
```

**Example:**

```yaml
iteration:
  max_attempts: 5
  exit_threshold: 70
  target_agent: test-architect@1.0.0
  fix_patterns: true
```

This runs up to 5 iterations. Each iteration: (1) run test-architect, (2) if score >= 70, stop; (3) otherwise, apply fixes from the fix pattern guidance and re-review.

---

### Inputs Block

Define input sources for commands.

```yaml
inputs:
  - name: string            # Required: Input name
    source: InputSource     # Required: Where to get the value
    path: string            # For 'file' source
    value: any              # For 'default' source
    env: string             # For 'env' source
    arg: string             # For 'arg' source (CLI argument name)
```

**Input Sources:**

| Source | Fields | Description |
|--------|--------|-------------|
| `file` | `path` | Read from file |
| `default` | `value` | Use static default value |
| `env` | `env` | Read from environment variable |
| `arg` | `arg`, `default` | Read from CLI argument |
| `previous` | `from` | Output from previous command (workflow context) |

---

### Postflight Block

Actions to take after execution completes. The `steps` array defines an **ordered sequence of actions** that execute before presenting results to the user.

**Postflight Actions (v1.2.0+):**

| Action | Description | When to Use |
|--------|-------------|-------------|
| `persist_to_tracker` | Save results to validation tracker | Always first for tracked commands |
| `verify_saved` | Confirm save succeeded by querying tracker | After persist_to_tracker |
| `present_summary` | Display results to user | Always last |
| `custom_command` | Run custom shell command | Special integrations |

**Unified Decision Handlers (v2.0.0):**

| Handler | Meaning | When Triggered |
|---------|---------|----------------|
| `on_success` | Agent succeeded / task completed | Score >= threshold, task complete |
| `on_partial` | Partial completion | Executor reports partial results |
| `on_failure` | Agent failed / task failed | Score < threshold, task failed |

All command types use the same three handlers. The agent's own `agentType` determines the semantic meaning of "success" and "failure."

---

### Tracking Block

Metrics collection and persistence configuration.

```yaml
tracking:
  enabled: boolean          # Optional: Include tracking (default: true)
  required: boolean         # Optional: Persistence is mandatory (default: true) (v1.2.0)
  variant: TrackingVariant  # Optional: Tracking implementation (default: 'claude-code')
  custom:                   # Required when variant: 'custom'
    metrics_command: string # Optional: Shell command to get metrics
    persistence_tool: string # Optional: Tool/function for saving results
    field_mappings: object  # Optional: Custom field mappings
```

**Tracking Variants:**

| Variant | Metrics Source | Persistence | Use Case |
|---------|---------------|-------------|----------|
| `claude-code` | `agent-metrics buffer list` | MCP uluops-tracker | Claude Code CLI (default) |
| `sdk` | SDK `response.usage` object | Direct API call | Anthropic SDK direct usage |
| `custom` | User-defined command | User-defined tool | Custom integrations |
| `none` | (omitted) | (omitted) | No tracking needed |

> **Behavioral Note (v1.2.0):** When `required: true` (the default), the executor MUST save results to the tracker BEFORE presenting the summary to the user.

---

### Output Block

Output formatting configuration.

```yaml
output:
  schema: string            # Required: Output schema reference
  format: OutputFormat      # Optional: Output format
```

**Output Formats:**

| Format | Description |
|--------|-------------|
| `structured` | Full JSON output with all fields |
| `summary` | Condensed summary only |
| `minimal` | Score/decision only |

---

## Composition Rules

### What Commands Can Reference

| Target | Allowed | Notes |
|--------|---------|-------|
| Any ADL Agent | Yes | Via `invokes.agent` — any agentType |
| No Agent | Yes | Utility commands |
| Other Commands | No | Use Workflow for command composition |
| Workflows | No | Commands are lower-level than workflows |

### When to Use a Command vs Direct Agent Reference

| Use Case | Approach |
|----------|----------|
| Agent needs preflight checks | Create a command |
| Agent needs custom tracking | Create a command |
| Agent needs overrides (threshold, model, timeout) | Create a command |
| Agent needs iterative fix loops | Create a command |
| Agent can run with default config | Reference agent directly in workflow |

### Command Type Determination

In v2.0.0, command type is **explicitly declared** via `interface.commandType`:

| Command Type | Template |
|-------------|----------|
| `agent` | `agent-command.md.njk` |
| `utility` | `utility-command.md.njk` |
| `custom` | `agent-command.md.njk` |

---

## Runtime Behavior

### Agent Command Execution Flow

```
  1. PREFLIGHT     -- Display banner, run checks
  2. RESOLVE       -- Fetch agent, apply overrides
  3. EXECUTE       -- Invoke agent, capture result
  4. ITERATE?      -- If iteration block: review-fix loop
  5. POSTFLIGHT    -- Persist to tracker, present result
```

The postflight decision is based on the agent's own `agentType`:
- **Validator agents**: Score-based PASS/FAIL against threshold
- **Executor agents**: Task completion COMPLETE/PARTIAL/FAILED
- **Analyst agents**: Analysis result SUCCESS/FAILURE
- **Generator agents**: Generation result SUCCESS/PARTIAL/FAILURE
- **Explorer/Forecaster agents**: Result SUCCESS/FAILURE

### Utility Execution Flow

```
  1. PREFLIGHT     -- Display banner, run checks
  2. EXECUTE       -- Follow procedural steps (no agent)
  3. POSTFLIGHT    -- Execute steps, present result
```

### Custom Execution Flow

```
  1. PREFLIGHT     -- Display banner, run checks (if defined)
  2. RESOLVE       -- Fetch agent (if invokes defined)
  3. EXECUTE       -- Invoke agent or custom logic
  4. POSTFLIGHT    -- Execute steps (if defined)
```

---

## Examples

### Complete Agent Command (Validator Agent)

```yaml
command:
  interface:
    name: validate
    version: "1.0.0"
    displayName: Code Validator
    commandType: agent
    description: Run code validation on target directory. Validates code quality, standards, tests, and documentation. Blocks progression if critical issues found.
    domain: software
    subdomain: quality
    tags: [validation, code-quality, gate]

  invokes:
    agent: code-validator@>=1.2.0

  execution:
    model:
      default: sonnet
      allowed: [haiku, sonnet, opus]
    timeout: 300000

  preflight:
    checks:
      - check: path_exists
        path: $ARGUMENTS
        message: "Directory not found"
    banner:
      template: "Running code validation..."

  overrides:
    threshold: 70

  postflight:
    steps:
      - action: persist_to_tracker
        required: true
        on_failure: block
      - action: verify_saved
        required: true
        on_failure: warn
      - action: present_summary
        required: true
    on_success:
      message: "PASS - Ready for next phase"
      exit_code: 0
    on_failure:
      message: "FAIL - Critical issues must be fixed"
      exit_code: 1

  output:
    schema: validator-output-v1.1
    format: structured
```

### Agent Command with Iteration (Fix Loop)

```yaml
command:
  interface:
    name: fix-test-gaps
    version: "1.0.0"
    displayName: Fix Test Gaps
    commandType: agent
    description: Iteratively fixes test gaps until Test Architect approves. Runs review-fix-review loop until score meets threshold.
    domain: software
    subdomain: remediation
    tags: [fixer, testing, iteration]

  invokes:
    agent: test-architect@1.0.0

  execution:
    model:
      default: sonnet
    timeout: 600000

  preflight:
    checks:
      - check: path_exists
        path: $ARGUMENTS
        message: Target directory does not exist
        on_fail: abort
    banner:
      template: "Starting test gap remediation on $ARGUMENTS..."

  overrides:
    threshold: 70

  iteration:
    max_attempts: 5
    exit_threshold: 70
    fix_patterns: true

  arguments:
    examples:
      - ./src
      - ./packages/api

  tracking:
    enabled: true
    variant: claude-code

  postflight:
    steps:
      - action: persist_to_tracker
        required: true
        on_failure: block
      - action: verify_saved
        required: true
        on_failure: warn
      - action: present_summary
        required: true
    on_success:
      message: "Test quality threshold met"
      exit_code: 0
    on_failure:
      message: "Max iterations reached without meeting threshold"
      exit_code: 1

  output:
    schema: validator-output-v1.1
    format: structured
```

### Agent Command (Executor Agent)

```yaml
command:
  interface:
    name: prompt-creator
    version: "1.0.0"
    displayName: Prompt Creator
    commandType: agent
    description: Scaffolds new AI agent prompts following project conventions
    domain: software
    subdomain: generation

  invokes:
    agent: prompt-creator@>=1.0.0

  execution:
    model:
      default: sonnet
    timeout: 300000

  inputs:
    - name: requirements
      source: arg
      arg: requirements

  postflight:
    on_success:
      message: "Prompt created successfully"
      exit_code: 0
    on_partial:
      message: "Prompt partially created"
      exit_code: 2
    on_failure:
      message: "Prompt creation failed"
      exit_code: 1

  output:
    schema: executor-output-v1.0
    format: structured
```

### Complete Utility Command

```yaml
command:
  interface:
    name: commit-push
    version: "1.0.0"
    displayName: Commit and Push
    commandType: utility
    description: Analyzes changes, checks for resolved validation issues, recommends conventional commit message, and executes git commit/push with rebase.
    domain: software
    subdomain: scm
    tags: [utility, git, commit, issue-resolution]

  execution:
    model:
      default: sonnet
    timeout: 120000

  preflight:
    checks:
      - check: command
        command: git rev-parse --is-inside-work-tree
        message: Not inside a git repository
        on_fail: abort
      - check: command
        command: git status --porcelain
        expect_output: true
        message: No changes to commit
        on_fail: abort

  tracking:
    enabled: true
    required: false
    variant: custom
    custom:
      persistence_tool: mcp__uluops-tracker__update_status
      field_mappings:
        project: project_name
        status: completed

  postflight:
    on_success:
      message: "Commit and push completed successfully"
      exit_code: 0
    on_failure:
      message: "Commit/push failed"
      exit_code: 1

  output:
    schema: executor-output-v1.0
    format: summary
```

### Specialized Model Command (Opus for Deep Analysis)

```yaml
command:
  interface:
    name: audit
    version: "1.0.0"
    displayName: Code Auditor
    commandType: agent
    description: Deep runtime correctness audit. Catches async bugs, null dereferences, silent failures. Use as FINAL gate before ship. Opus model for deeper reasoning.
    domain: software
    subdomain: safety
    tags: [audit, runtime, async, null-safety, opus]

  invokes:
    agent: code-auditor@>=1.0.0

  execution:
    model:
      default: opus
      allowed: [opus]
    timeout: 600000

  preflight:
    checks:
      - check: path_exists
        path: $ARGUMENTS
        message: "Target directory not found"

  overrides:
    threshold: 80

  postflight:
    on_success:
      message: "SOUND - Runtime safety is production-ready"
      exit_code: 0
    on_failure:
      message: "UNSOUND - Critical runtime issues must be fixed"
      exit_code: 1

  output:
    schema: validator-output-v1.1
    format: structured
```

---

## JSON Schema Reference

The complete JSON Schema is available at:
`https://uluops.ai/schemas/cdl/v2.0.0/command.json`

### Schema Overview

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "https://uluops.ai/schemas/cdl/v2.0.0/command.json",
  "title": "Command Definition Language (CDL) Schema",
  "version": "2.0.0",
  "type": "object",
  "required": ["command"],
  "properties": {
    "command": {
      "type": "object",
      "required": ["interface"],
      "properties": {
        "interface": { "$ref": "#/$defs/interface" },
        "invokes": { "$ref": "#/$defs/invokes" },
        "execution": { "$ref": "#/$defs/execution" },
        "preflight": { "$ref": "#/$defs/preflight" },
        "overrides": { "$ref": "#/$defs/overrides" },
        "inputs": { "$ref": "#/$defs/inputs" },
        "arguments": { "$ref": "#/$defs/arguments" },
        "iteration": { "$ref": "#/$defs/iteration" },
        "postflight": { "$ref": "#/$defs/postflight" },
        "tracking": { "$ref": "#/$defs/tracking" },
        "output": { "$ref": "#/$defs/output" }
      },
      "allOf": [
        {
          "if": { "properties": { "interface": { "properties": { "commandType": { "const": "agent" } } } } },
          "then": { "required": ["interface", "invokes"] }
        }
      ]
    }
  }
}
```

### Conditional Validation Rules

The schema uses JSON Schema `allOf`/`if`/`then` for type-dependent requirements:

1. **agent**: `invokes` is required
2. **utility**: No conditional requirements beyond `interface`
3. **custom**: No conditional requirements beyond `interface`

---

## Migration from v1.3.0

### Command Type Changes

| v1.3.0 Type | v2.0.0 Type | Notes |
|-------------|-------------|-------|
| `validator` | `agent` | Wrapped agent's `agentType` determines scoring semantics |
| `executor` | `agent` | Wrapped agent's `agentType` determines completion semantics |
| `fixer` | `agent` + `iteration` | Iteration block remains, just remove the `fixer` type |
| `utility` | `utility` | No change |

### Postflight Handler Renames

| v1.3.0 Handler | v2.0.0 Handler | Context |
|----------------|----------------|---------|
| `on_pass` | `on_success` | Validator/Fixer success |
| `on_fail` | `on_failure` | Validator/Fixer failure |
| `on_warn` | *(removed)* | Use `on_partial` if needed |
| `on_complete` | `on_success` | Executor success |
| `on_partial` | `on_partial` | No change |
| `on_failed` | `on_failure` | Executor failure |

### Migration Examples

**Validator Command:**

```yaml
# v1.3.0
interface:
  commandType: validator
postflight:
  on_pass:
    message: "PASS"
  on_fail:
    message: "FAIL"

# v2.0.0
interface:
  commandType: agent
postflight:
  on_success:
    message: "PASS"
  on_failure:
    message: "FAIL"
```

**Executor Command:**

```yaml
# v1.3.0
interface:
  commandType: executor
postflight:
  on_complete:
    message: "Done"
  on_failed:
    message: "Failed"

# v2.0.0
interface:
  commandType: agent
postflight:
  on_success:
    message: "Done"
  on_failure:
    message: "Failed"
```

**Fixer Command:**

```yaml
# v1.3.0
interface:
  commandType: fixer
iteration:
  max_attempts: 5
  exit_threshold: 70

# v2.0.0
interface:
  commandType: agent
iteration:               # Iteration block stays the same
  max_attempts: 5
  exit_threshold: 70
```

### Automatic Migration

The CDL factory (`@uluops/definition-factory`) automatically migrates v1.3.0 documents:

1. `commandType: validator` / `executor` / `fixer` → `agent` (with deprecation warning)
2. Postflight handlers renamed to v2.0.0 names
3. No manual changes required for existing YAML files, but updating is recommended

---

## Revision History

| Version | Date | Changes |
|---------|------|---------|
| 2.0.0 | 2026-03-01 | **Breaking:** Collapsed `validator`/`executor`/`fixer` command types into `agent`. Added `custom` type. Unified postflight handlers: `on_success`/`on_partial`/`on_failure` (replaces type-specific `on_pass`/`on_fail`/`on_complete`/`on_failed`/`on_warn`). `iteration` block now optional for any `agent` command (not just fixer). Factory auto-migrates v1.3.0 documents. New unified template: `agent-command.md.njk`. |
| 1.3.0 | 2026-02-08 | Added `interface.commandType` (required enum: `validator`, `executor`, `utility`, `fixer`). Made `invokes` conditional (required for validator/executor/fixer, omitted for utility). Added `iteration` block for fixer commands (max_attempts, exit_threshold, target_agent, fix_patterns). New templates: `utility-command.md.njk`, `fixer-command.md.njk`. Updated factory with type-aware context builder and renderer. |
| 1.2.0 | 2026-01-30 | Added `tracking.required` field to make persistence mandatory before presenting results. Added `postflight.steps` array for ordered post-agent actions (persist_to_tracker, verify_saved, present_summary). Enables declarative control over save-then-present behavior. |
| 1.1.1 | 2026-01-18 | Added `arguments.examples` field for custom usage examples in generated documentation. Replaces generic auto-generated examples with user-specified paths. |
| 1.1.0 | 2026-01-18 | Added `tracking` block for configurable metrics collection and persistence. Supports `claude-code`, `sdk`, `custom`, and `none` variants. Enables SDK-based tracking for direct API usage without Claude Code CLI dependencies. |
| 1.0.1 | 2026-01-14 | Updated agent reference format to `name@version` syntax for consistency with Registry API. Added `arg` input source type for CLI argument passthrough. |
| 1.0.0 | 2026-01-13 | Initial CDL specification. Single-agent invocation model. Interface block consistent with ADL. Preflight checks, execution config, overrides, postflight actions. |

---

## Appendix: Type Definitions

### Common Types

```typescript
type CommandType = 'agent' | 'utility' | 'custom';

// Legacy types (auto-migrated to 'agent' by factory)
type LegacyCommandType = 'validator' | 'executor' | 'fixer';

type Domain =
  | 'software'
  | 'legal'
  | 'medical'
  | 'financial'
  | 'scientific'
  | 'content'
  | 'general'
  | 'career'
  | 'compliance'
  | 'business'
  | 'education'
  | 'documentation'
  | 'security';

type Model = 'haiku' | 'sonnet' | 'opus' | `${string}/${string}`;

type VersionReq =
  | string                    // Exact: "1.0.0"
  | `>=${string}`            // Minimum: ">=1.0.0"
  | `^${string}`             // Compatible: "^1.0.0"
  | `~${string}`             // Patch: "~1.0.0"
  | 'latest';                // Latest published

type OutputFormat = 'structured' | 'summary' | 'minimal';

type InputSource = 'file' | 'default' | 'env' | 'arg' | 'previous';

type TrackingVariant = 'claude-code' | 'sdk' | 'custom' | 'none';

type CheckType = 'path_exists' | 'file_exists' | 'command' | 'git_clean' | 'env_var';

type OnFailAction = 'abort' | 'warn' | 'skip';
```

### Interface Types

```typescript
interface CommandInterface {
  name: string;              // Kebab-case identifier
  version: string;           // SemVer
  displayName: string;       // Human-readable (3-50 chars)
  commandType: CommandType;  // Command type (v2.0.0)
  description: string;       // Usage description (20-500 chars)
  domain: Domain;
  subdomain?: string;
  tags?: string[];
}
```

### Invokes Types

```typescript
interface Invokes {
  agent: AgentRef;           // Agent reference (name@version format)
}

/** Agent reference format: name@version */
type AgentRef = `${string}@${VersionReq}`;
```

### Execution Types

```typescript
interface Execution {
  model: {
    default: Model;          // Default model
    allowed?: Model[];       // Allowed overrides
  };
  timeout?: number;          // Milliseconds (default: 300000)
  max_retries?: number;      // Retry count (default: 0)
  retry_delay?: number;      // Retry delay ms (default: 1000)
}
```

### Iteration Types

```typescript
interface Iteration {
  max_attempts: number;      // Maximum review-fix iterations (1-20)
  exit_threshold: number;    // Score that ends the loop (0-100)
  target_agent?: AgentRef;   // Agent to re-run (defaults to invokes.agent)
  fix_patterns?: boolean;    // Include fix guidance (default: true)
}
```

### Preflight Types

```typescript
interface Preflight {
  checks?: PreflightCheck[];
  banner?: {
    template: string;
  };
}

interface PreflightCheck {
  check: CheckType;
  path?: string;             // For path_exists, file_exists
  command?: string;          // For command type
  name?: string;             // For env_var type
  required?: boolean;        // For file_exists (default: true)
  expect_output?: boolean;   // For command type
  expect_empty?: boolean;    // For command type
  message?: string;          // Error/warning message
  on_fail?: OnFailAction;    // abort | warn | skip (default: abort)
}
```

### Overrides Types

```typescript
interface Overrides {
  threshold?: number;        // Override decision threshold
  timeout?: number;          // Override agent timeout (ms)
  temperature?: number;      // Override model temperature (0.0-1.0)
}
```

### Arguments Types

```typescript
interface Arguments {
  examples?: string[];         // Custom usage examples (1-5 items)
}
```

### Inputs Types

```typescript
interface Input {
  name: string;              // Input parameter name
  source: InputSource;       // file | default | env | arg | previous
  path?: string;             // For file source
  value?: any;               // For default source
  env?: string;              // For env source
  arg?: string;              // For arg source (CLI argument name)
  default?: any;             // Default value (for arg source)
  from?: string;             // For previous source
}
```

### Postflight Types

```typescript
interface Postflight {
  steps?: PostflightStep[];       // Ordered post-execution actions (v1.2.0)
  on_success?: PostflightMessage; // Success handler (v2.0.0)
  on_partial?: PostflightMessage; // Partial completion handler
  on_failure?: PostflightMessage; // Failure handler (v2.0.0)
}

interface PostflightStep {
  action: PostflightStepAction;
  required?: boolean;        // If true, failure blocks remaining steps (default: true)
  on_failure?: FailureAction;
  command?: string;          // For custom_command action
  message?: string;          // Display message
}

type PostflightStepAction =
  | 'persist_to_tracker'
  | 'verify_saved'
  | 'present_summary'
  | 'custom_command';

type FailureAction = 'block' | 'warn' | 'ignore';

interface PostflightMessage {
  message?: string;
  exit_code?: number;
}
```

### Tracking Types

```typescript
interface Tracking {
  enabled?: boolean;         // Include tracking (default: true)
  required?: boolean;        // Persistence mandatory before summary (default: true) (v1.2.0)
  variant?: TrackingVariant; // Tracking implementation (default: 'claude-code')
  custom?: CustomTracking;   // Required when variant='custom'
}

interface CustomTracking {
  metrics_command?: string;  // Shell command to retrieve metrics
  persistence_tool?: string; // Tool/function for saving results
  field_mappings?: Record<string, string>; // Custom field mappings
}
```

### Output Types

```typescript
interface Output {
  schema: string;            // Schema reference (e.g., "validator-output-v1.1")
  format?: OutputFormat;     // structured | summary | minimal
}
```

### Complete Command Type

```typescript
interface Command {
  interface: CommandInterface;
  invokes?: Invokes;          // Required for agent, omitted for utility, optional for custom
  execution?: Execution;
  preflight?: Preflight;
  overrides?: Overrides;
  inputs?: Input[];
  arguments?: Arguments;
  iteration?: Iteration;      // Optional for agent/custom (v2.0.0)
  postflight?: Postflight;
  tracking?: Tracking;
  output?: Output;
}
```
