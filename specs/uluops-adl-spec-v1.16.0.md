# UluOps Agent Definition Language (ADL) Specification

**Version:** 1.16.0
**Status:** Draft
**Created:** 2026-01-11
**Updated:** 2026-05-10
**Supersedes:** ADL v1.15.0, ADL v1.14.0, ADL v1.13.0, ADL v1.12.0, ADL v1.11.0, ADL v1.10.0, ADL v1.9.0, ADL v1.8.0, ADL v1.7.0, ADL v1.6.0, ADL v1.5.1, ADL v1.5.0, ADL v1.4.0, ADL v1.3.0, ADL v1.2.0, VDL v1.1.0

---

## Table of Contents

- [Overview](#overview)
- [Design Principles](#design-principles)
- [Agent Types](#agent-types)
- [Domain Profiles](#domain-profiles)
- [Schema Structure](#schema-structure)
  - [Common Schema](#common-schema)
  - [Validator Schema](#validator-schema)
  - [Executor Schema](#executor-schema)
  - [Analyst Schema](#analyst-schema)
  - [Generator Schema](#generator-schema)
  - [Explorer Schema](#explorer-schema)
  - [Forecaster Schema](#forecaster-schema)
- [Detailed Field Reference](#detailed-field-reference)
  - [Interface Block](#interface-block)
  - [Epistemic Nature Block](#epistemic-nature-block)
  - [Defaults Block](#defaults-block)
  - [Context Block](#context-block)
  - [Mission Block](#mission-block)
  - [Knowledge Base Block](#knowledge-base-block)
  - [Scoring Block (Validators)](#scoring-block-validators)
  - [Tasks Block (Executors)](#tasks-block-executors)
  - [Decisions Block](#decisions-block)
  - [Completion Block (Executors)](#completion-block-executors)
  - [Auto-Fail Block](#auto-fail-block)
  - [Deductions Block](#deductions-block)
  - [Rollback Block (Executors)](#rollback-block-executors)
  - [Process Block](#process-block)
  - [Output Block](#output-block)
  - [Edge Cases Block](#edge-cases-block)
  - [Tone Block](#tone-block)
  - [Forecast Block (Forecasters)](#forecast-block-forecasters)
- [Failure Taxonomy](#failure-taxonomy)
- [Retry Configuration](#retry-configuration)
- [Composition Rules](#composition-rules)
- [Runtime Rendering](#runtime-rendering)
- [Migration from ADL v1.8.0](#migration-from-adl-v180)
- [Migration from ADL v1.7.0](#migration-from-adl-v170)
- [Migration from ADL v1.6.0](#migration-from-adl-v160)
- [Migration from ADL v1.5.1](#migration-from-adl-v151)
- [Migration from VDL](#migration-from-vdl)
- [Migration from ADL v1.1.0](#migration-from-adl-v110)
- [Examples](#examples)
- [JSON Schema Reference](#json-schema-reference)
- [Revision History](#revision-history)
- [Appendix: Type Definitions](#appendix-type-definitions)

---

## Overview

The **Agent Definition Language (ADL)** is a YAML-based specification for defining atomic AI agents in the UluOps validation framework. ADL unifies the previous Validator Definition Language (VDL) with new executor capabilities, providing a single schema for both analysis (scoring) and action (task execution) agents.

### Key Characteristics

| Aspect | Description |
|--------|-------------|
| **Format** | YAML with JSON Schema validation |
| **File Extension** | `.agent.yaml` (recommended) or `.yaml` |
| **Schema URL** | `https://uluops.ai/schemas/adl/v1.13.0/agent.json` |
| **Discriminator** | `agent.interface.agentType`: `validator`, `executor`, `analyst`, `generator`, or `explorer` |

### Agent Types Summary

| Type | Purpose | Output | Decision Vocabulary |
|------|---------|--------|---------------------|
| **Validator** | Analyze and score | Score + Findings | PASS / WARN / FAIL |
| **Executor** | Perform tasks | Artifacts + Changes | COMPLETE / PARTIAL / FAILED |
| **Analyst** | Investigate and assess | Analysis Report + Recommendations | Custom (defined per agent) |
| **Generator** | Produce artifacts | Generated Artifacts + Status | Custom (defined per agent) |
| **Explorer** | Discover and map | Exploration Report | None (narrative output) |

### Relationship to Other Definition Languages

```
                    ADL (Agent Definition)
                           │
                           │ wrapped by
                           ▼
                    CDL (Command Definition)
                           │
                           │ orchestrated by
                           ▼
                    WDL (Workflow Definition)
                           │
                           │ staged by
                           ▼
                    PDL (Pipeline Definition)
```

Agents are **atomic units** that cannot reference other definitions. They are composed into higher-level structures via Commands, Workflows, and Pipelines.

---

## Design Principles

### 1. Declarative Over Imperative

ADL describes **what** an agent evaluates or accomplishes, not **how** it executes. The runtime (SDK) interprets the definition and orchestrates execution.

```yaml
# ✔ Declarative: What to check
criteria:
  - name: Type annotations present
    points: 10

# ✗ Imperative: How to check (belongs in process block)
# run: "tsc --noEmit && count annotations"
```

### 2. Objective Scoring (Validators)

Validator agents use a **100-point scoring framework** with explicit criteria, eliminating subjective judgment:

- Total points always sum to 100
- Categories have defined weights (10-30 points each)
- Criteria have measurable verification methods
- Thresholds map scores to decisions deterministically

### 3. Task-Based Execution (Executors)

Executor agents define **discrete tasks** with clear inputs, outputs, and completion criteria:

- Tasks have explicit dependencies
- Inputs are typed and validated
- Outputs are captured as artifacts
- Completion is binary or progress-based

### 4. Single Responsibility

Each agent focuses on one domain concern. Composition happens at the Command/Workflow level:

```yaml
# ✔ Single responsibility
name: code-validator        # Only code quality
name: type-safety-validator # Only type safety
name: code-fixer            # Only fixes issues

# ✗ Multiple responsibilities (use workflow instead)
# name: validate-and-fix-code
```

### 5. Backward Compatibility

ADL maintains full backward compatibility with VDL v1.1.0 and ADL v1.1.0:

- Existing `.yaml` validators work with minimal changes
- Root key changes from `validator:` to `agent:`
- `metadata:` renamed to `interface:`
- New `agentType: validator` field required
- All v1.1.0 definitions valid in v1.2.0
- New v1.2.0 blocks (mission, knowledge_base) are optional

---

## Agent Types

### Validators

**Purpose:** Analyze targets and produce scored assessments with actionable findings.

**Characteristics:**
- 100-point scoring framework
- Category/criteria breakdown
- Decision vocabulary: PASS, WARN (optional), FAIL
- Output: Score + Findings + Decision
- Side effects: None (read-only analysis)

**Use Cases:**
- Code quality validation
- Security auditing
- Documentation completeness
- API contract compliance
- Test coverage analysis

### Executors

**Purpose:** Perform tasks that modify state or produce artifacts.

**Characteristics:**
- Task-based execution model
- Typed inputs and outputs
- Decision vocabulary: COMPLETE, PARTIAL, FAILED
- Output: Artifacts + Changes + Status
- Side effects: Creates/modifies files, external calls

**Use Cases:**
- Issue auto-fixing
- Code generation
- Documentation generation
- Migration scripts
- Refactoring operations

### Analysts

**NEW in v1.6.0**

**Purpose:** Investigate topics, assess situations, and produce analytical reports with recommendations.

**Characteristics:**
- Process-driven execution model
- Optional scoring framework (analysts CAN score but don't have to)
- Custom decision vocabulary
- Output: Analysis Report + Findings + Recommendations
- Side effects: None (read-only analysis)

**Use Cases:**
- Ecosystem pattern analysis
- Strategic assessment
- Trend analysis
- Data science review
- Risk assessment

**Composition Rules:**
- Requires: `process` (must define analysis phases)
- Forbids: `tasks`, `completion`, `rollback`
- Optional: `scoring`, `decisions` (for assessment-style agents that produce scores)

### Generators

**NEW in v1.6.0**

**Purpose:** Produce artifacts (files, configurations, scaffolds) from structured inputs.

**Characteristics:**
- Task-based execution model (defines what to produce)
- No scoring or judgment capabilities
- Custom decision vocabulary
- Output: Generated Artifacts + Status
- Side effects: Creates files and configurations

**Use Cases:**
- Agent prompt scaffolding
- Configuration generation
- Code scaffolding
- Template expansion
- Schema generation

**Composition Rules:**
- Requires: `tasks` (defines what to produce)
- Forbids: `scoring`, `deductions`, `auto_fail` (generators produce artifacts, not judgments)
- Optional: `completion` (quality self-check)

### Explorers

**NEW in v1.8.0**

**Purpose:** Discover, map, and report on codebases, systems, or structures using tool-augmented exploration.

**Characteristics:**
- Mission-driven with minimal ceremony
- Lightweight — no scoring framework, no decisions, no structured output
- Tool-augmented (semantic search, call graph tracing, file traversal)
- Output: Narrative exploration report
- Side effects: None (read-only exploration)

**Use Cases:**
- Codebase mapping and architecture discovery
- Dependency tracing and call graph analysis
- System exploration before implementation planning
- Semantic search across large repositories
- Knowledge extraction from unfamiliar codebases

**Composition Rules:**
- Requires: `interface`, `mission`
- Forbids: `scoring`, `decisions`, `deductions`, `auto_fail`, `tasks`, `completion`, `rollback`
- Optional: `knowledge_base`, `process`, `edge_cases`, `output`, `tone`

**Design Rationale:** Explorers represent a fundamentally different agent archetype. Traditional agent types (validator, executor, analyst, generator) all carry structured output overhead — scoring frameworks, decision vocabularies, task models, or completion criteria. Explorers strip this away, recognizing that discovery-oriented agents need only a mission, tool guidance, and a process to follow. The lightweight definition reduces prompt token overhead (59% smaller than equivalent analyst definitions) while improving exploration quality by removing scoring-related cognitive load from the LLM.

### Forecasters

**NEW in v1.9.0**

**Purpose:** Model future states, emergent effects, and causal consequences of artifacts after they enter the world and are acted upon by time, people, and systems.

**Characteristics:**
- Process-driven with prediction-oriented output
- No scoring framework — predictions are probability-weighted, not binary pass/fail
- Custom decision vocabulary (e.g., SOUND/PERVERSE, DURABLE/FRAGILE, BOUNDED/GENERATIVE)
- Output: Structured predictions with causal chains, risk surfaces, and timelines
- Side effects: None (read-only prediction)
- Optional `forecast` section for specifying prediction lens

**Use Cases:**
- Perverse outcome detection (metric gaming, threshold satisficing)
- Unintended consequence analysis
- Adoption drift modeling
- Temporal decay prediction
- Cascade depth analysis
- Capability emergence forecasting

**Composition Rules:**
- Requires: `interface`, `mission`, `process`
- Forbids: `tasks`, `completion`, `rollback`
- Optional: `scoring`, `decisions`, `deductions`, `auto_fail`, `knowledge_base`, `edge_cases`, `output`, `tone`, `forecast`

**Design Rationale:** Forecasters operate on "what will be" rather than "what is." Every other agent type evaluates or acts on current state. Forecasters trace causal chains into states that don't yet exist but will emerge. This requires modeling actors, incentives, systems, and time — a fundamentally different cognitive operation from scoring or exploring. Like analysts, forecasters may optionally include scoring frameworks for assessment-style prediction (e.g., scoring the severity of predicted perverse outcomes). The optional `forecast` section captures the three axes that define a forecaster's analytical lens: actor type, time horizon, and propagation mechanism.

---

## Domain Profiles

**NEW in v1.6.0**

Domain profiles provide domain-specific vocabulary and content for agent generation, preventing software-domain assumptions from leaking into non-software agents.

### Overview

When the ADL factory renders an agent prompt, software-specific content is embedded by default (e.g., severity descriptions reference "crashes", edge cases include "Non-Git Repository"). Domain profiles replace these defaults with domain-appropriate content.

### Profile Reference

Profiles are referenced via the `domain_profile` field in the interface block:

```yaml
interface:
  name: contract-law-practitioner
  agentType: validator
  domain: legal
  domain_profile: legal    # Loads legal.profile.yaml
```

### Profile Location

Profiles are stored at `udl/definition-languages/profiles/{name}.profile.yaml` and validated against `domain-profile-schema-v1.0.0.json`.

### Profile Sections

| Section | Required | Description |
|---------|----------|-------------|
| `interface` | ✔ | Profile identity (domain, version, displayName, description) |
| `severity_descriptions` | ✔ | Domain-appropriate severity labels and descriptions |
| `issue_types` | ✔ | Domain vocabulary for issue classification with tracker mapping |
| `common_edge_cases` | ✔ | Domain-specific edge cases (replaces software defaults) |
| `terminology` | ✔ | Vocabulary substitutions (artifact, review_unit, location_unit, etc.) |
| `output_examples` | No | Domain reference examples for output formatting |
| `knowledge_defaults` | No | Content preferences (code_language, example_format, references_style) |
| `context_adjustments` | No | Severity context guidance for domain-specific situations |

### Tracker Integration

Domain profiles define a `tracker_mapping` that maps domain-specific issue types to the universal tracker enum:

```yaml
issue_types:
  types:
    - id: omission
      label: Omission
      description: A missing element that should be present
  tracker_mapping:
    omission: feature      # Maps to universal "feature" type
    drafting: refactor     # Maps to universal "refactor" type
    non-compliance: bug    # Maps to universal "bug" type
```

Note: Universal types (`feature`, `bug`, `refactor`, `config`, `docs`, `infra`, `security`, `test`, `observation`, `deficiency`, `ambiguity`) don't need tracker_mapping entries — they pass through as-is. Only domain-specific types (like `omission`, `drafting`) require mapping.

Agent output includes both `type` (universal, for tracker) and `domain_type` (domain-specific, for display).

### Available Profiles

| Profile | Domain | Description |
|---------|--------|-------------|
| `software` | software | Default profile — extracts current implicit software defaults |
| `legal` | legal | Contract law, compliance, regulatory analysis |

---

## Schema Structure

### Common Schema

All agents share this base structure:

```yaml
agent:
  interface:           # Required: Identity and classification
    name: string
    version: string
    displayName: string
    description: string
    agentType: validator | executor | analyst | generator | explorer
    domain: string
    subdomain: string?
    domain_profile: string?              # NEW in v1.6.0: Domain profile reference
    risk_level: low | standard | high | critical
    tools: Tool[]?
    tags: string[]?
    triggers: Triggers?
    dependencies: Dependencies?
    epistemic_nature: EpistemicNature?    # NEW in v1.13.0: Multi-axis epistemic classification

  defaults:            # Optional: Runtime defaults
    model: haiku | sonnet | opus
    timeout: number    # milliseconds
    max_tokens: number
    temperature: number

  context:             # Optional: Execution context
    working_directory: string?
    environment: Record<string, string>?
    timeout_behavior: fail | warn | continue
    shell: bash | sh | zsh | powershell

  mission:             # NEW in v1.2.0: Agent identity and purpose
    opener: string?                      # Present-tense immersive opening
    stakes: string?                      # Why this validation matters
    outcome_framing: string?             # What the agent produces
    role_boundaries: string[]?           # Scope clarifications
    taxonomy_mandate: boolean?           # Require failure codes (default: true)
    vocabulary_rationale: string?        # NEW in v1.5.0: Explains decision vocabulary choice
    explicit_prohibitions: string[]?     # NEW in v1.5.0: Actions agent must NOT do

  knowledge_base:      # NEW in v1.2.0: Embedded domain expertise
    sections: KnowledgeSection[]?
    global_references: string[]?

  process:             # Optional: Execution phases
    phases: Phase[]

  output:              # Optional: Output formatting
    format: markdown | json | html | structured
    schema: string?    # JSON schema reference
    sections: Section[]?

  edge_cases:          # Optional: Special handling
    EdgeCase[]

  tone:                # Optional: Communication style
    attributes: string[]
    guidelines: string[]
```

### Validator Schema

Validators add scoring-specific sections:

```yaml
agent:
  interface:
    agentType: validator
    # ... common fields

  defaults:
    # ... common fields

  context:
    # ... context fields

  mission:             # NEW in v1.2.0
    # ... mission fields

  knowledge_base:      # NEW in v1.2.0
    # ... knowledge base fields

  scoring:             # Required for validators
    maxScore: 100
    categories: Category[]
    constraints: Constraints?

  decisions:           # Required for validators
    vocabulary:
      positive: string   # e.g., "PASS"
      negative: string   # e.g., "FAIL"
      conditional: string?  # e.g., "WARN"
    thresholds: Threshold[]
    preset: string?      # Alternative to explicit thresholds
    tracking: DecisionTracking?

  auto_fail:           # Optional: Critical failure conditions
    enabled: boolean
    conditions: AutoFailCondition[]

  deductions:          # Optional: Severity-based point deductions
    severity_scale: SeverityLevel[]

  # ... common optional sections (process, output, edge_cases, tone)
```

### Executor Schema

Executors add task-specific sections:

```yaml
agent:
  interface:
    agentType: executor
    # ... common fields

  defaults:
    # ... common fields

  context:
    # ... context fields

  mission:             # NEW in v1.2.0
    # ... mission fields

  knowledge_base:      # NEW in v1.2.0
    # ... knowledge base fields

  tasks:               # Required for executors
    inputs: Input[]      # What the executor needs
    operations: Task[]   # What the executor does
    outputs: Output[]    # What the executor produces

  completion:          # Required for executors
    vocabulary:
      complete: string   # e.g., "COMPLETE"
      partial: string    # e.g., "PARTIAL"
      failed: string     # e.g., "FAILED"
    criteria:            # How to determine completion
      - condition: string
        decision: complete | partial | failed

  rollback:            # Optional: Recovery on failure
    enabled: boolean
    strategy: git_restore | git_stash | backup_restore | manual | none
    preserve_logs: boolean
    notify_on_rollback: boolean

  # ... common optional sections (process, output, edge_cases, tone)
```

### Analyst Schema

**NEW in v1.6.0**

Analysts add process-driven analysis with optional scoring:

```yaml
agent:
  interface:
    agentType: analyst
    # ... common fields

  defaults:
    # ... common fields

  context:
    # ... context fields

  mission:
    # ... mission fields

  knowledge_base:
    # ... knowledge base fields

  process:             # Required for analysts
    phases: Phase[]

  scoring:             # Optional for analysts (assessment-style)
    maxScore: 100
    categories: Category[]

  decisions:           # Optional for analysts
    vocabulary:
      positive: string
      negative: string
      conditional: string?
    thresholds: Threshold[]

  # Forbidden: tasks, completion, rollback
  # Common optional: output, edge_cases, tone, auto_fail, deductions
```

### Generator Schema

**NEW in v1.6.0**

Generators define task-based artifact production:

```yaml
agent:
  interface:
    agentType: generator
    # ... common fields

  defaults:
    # ... common fields

  context:
    # ... context fields

  mission:
    # ... mission fields

  knowledge_base:
    # ... knowledge base fields

  tasks:               # Required for generators
    inputs: Input[]
    operations: Task[]
    outputs: Output[]

  completion:          # Optional for generators (quality self-check)
    vocabulary:
      complete: string
      partial: string
      failed: string
    criteria: CompletionCriterion[]

  # Forbidden: scoring, deductions, auto_fail
  # Common optional: process, output, edge_cases, tone
```

### Explorer Schema

**NEW in v1.8.0**

Explorers define lightweight, mission-driven agents with tool guidance:

```yaml
agent:
  interface:
    agentType: explorer
    # ... common fields

  defaults:
    # ... common fields

  mission:             # Required for explorers
    opener: string
    stakes: string?
    outcome_framing: string?
    role_boundaries: string[]?
    explicit_prohibitions: string[]?

  knowledge_base:      # Optional: Tool guidance and domain expertise
    sections:
      - id: string
        title: string
        content: string

  process:             # Optional: Exploration phases
    phases: Phase[]

  edge_cases:          # Optional: Special handling
    EdgeCase[]

  # Forbidden: scoring, decisions, deductions, auto_fail, tasks, completion, rollback
  # Common optional: defaults, context, output, tone
```

### Forecaster Schema

**NEW in v1.9.0**

Forecasters define prediction-oriented agents that model future states and causal consequences:

```yaml
agent:
  interface:
    agentType: forecaster
    # ... common fields

  defaults:
    # ... common fields

  mission:             # Required: prediction framing
    opener: string
    stakes: string?
    outcome_framing: string?
    role_boundaries: string[]?
    explicit_prohibitions: string[]?

  process:             # Required: analysis phases
    phases: Phase[]

  forecast:            # Optional: prediction lens configuration (v1.9.0)
    actor_type: enum   # rational | naive | adversarial | systemic | cultural | creative
    time_horizon: enum # near-term | medium-term | long-term
    propagation_mechanism: string
    prediction_format: enum # probability-weighted | spectrum | timeline | depth-map | interaction-map

  scoring:             # Optional: assessment-style forecasters may score
    maxScore: 100
    categories: Category[]

  decisions:           # Optional: prediction vocabulary
    vocabulary:
      positive: string
      negative: string

  # Forbidden: tasks, completion, rollback
  # Common optional: knowledge_base, edge_cases, output, tone, deductions, auto_fail
```

---

## Detailed Field Reference

### Interface Block

The `interface` block defines agent identity and classification.

```yaml
interface:
  # Required fields
  name: code-validator
  version: "1.2.0"
  displayName: Code Validator
  description: >
    Validates code quality and best practices. Run after implementation
    to catch issues before code review. Blocks deployment if critical
    issues are found.
  agentType: validator
  domain: software

  # Optional fields
  subdomain: quality
  domain_profile: software               # NEW in v1.6.0
  risk_level: standard                    # NEW in v1.1.0
  tools:                                  # NEW in v1.1.0
    - Read
    - Grep
    - Glob
    - Bash
  tags:
    - typescript
    - javascript
    - quality
  triggers:
    file_patterns:
      - "src/**/*.ts"
    project_markers:
      - "package.json"
    explicit_only: false
  dependencies:
    requires: []
    recommends:
      - pre-implementation-architect
    conflicts: []                         # NEW in v1.1.0
  epistemic_nature:                        # NEW in v1.13.0
    verifiability: mechanically_checkable
    determinism: stochastic
    claim_type: factual
```

#### Field Definitions

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | ✔ | Kebab-case identifier. Pattern: `^[a-z][a-z0-9]*(-[a-z0-9]+)*$` |
| `version` | string | ✔ | Semantic version. Pattern: `^\d+\.\d+\.\d+$` |
| `displayName` | string | ✔ | Human-readable name for UI display (3-50 chars) |
| `description` | string | ✔ | 20-500 chars: When to use + what it does + what it blocks/enables |
| `agentType` | enum | ✔ | `validator`, `executor`, `analyst`, or `generator` |
| `domain` | enum | ✔ | Primary domain (see Domain Values) |
| `subdomain` | string | | Optional refinement of domain |
| `domain_profile` | string | | Reference to a domain profile for domain-specific rendering (v1.6.0). Pattern: `^[a-z][a-z0-9]*(-[a-z0-9]+)*$`. Profiles loaded from `udl/definition-languages/profiles/{name}.profile.yaml` |
| `risk_level` | enum | | Risk classification: `low`, `standard`, `high`, `critical` (default: `standard`) |
| `tools` | Tool[] | | Runtime capabilities required (see Tool Values) |
| `tags` | string[] | | Searchable tags for discovery |
| `triggers` | object | | Activation configuration |
| `dependencies` | object | | Agent dependency declarations |
| `epistemic_nature` | EpistemicNature | | Multi-axis epistemic classification (v1.13.0). See [Epistemic Nature Block](#epistemic-nature-block) |

#### Domain Values

| Domain | Description | Example Agents |
|--------|-------------|----------------|
| `software` | Code, APIs, infrastructure, DevOps | code-validator, code-optimizer |
| `security` | Vulnerability analysis, threat modeling, penetration testing | security-analyst, prompt-security-analyst |
| `compliance` | Regulatory, audit, standards (cross-cuts legal/financial/medical) | sox-validator, gdpr-checker |
| `legal` | Contracts, compliance, regulations | contract-reviewer, compliance-checker |
| `medical` | Clinical, diagnostic, pharmaceutical | diagnosis-validator, trial-auditor |
| `financial` | Portfolio, risk, trading, compliance | risk-assessor, trade-validator |
| `scientific` | Research, data analysis, methodology | methodology-reviewer, data-validator |
| `content` | Writing, media, marketing, creative | prose-formatter, seo-validator |
| `general` | Cross-domain, meta-level, utilities | vdl-meta-validator |
| `career` | Job applications, resumes, interviews | ats-optimization, role-fit-analyzer |
| `business` | Strategy, operations, planning | impact-quantification |
| `education` | Learning, curriculum, assessment | course-validator |
| `documentation` | Technical writing, API docs | docs-validator |

**Note:** Domains `career`, `business`, `education`, and `documentation` are included for VDL backward compatibility.

#### Tool Values

| Tool | Description |
|------|-------------|
| `Read` | Read file contents |
| `Grep` | Pattern search in files |
| `Glob` | File path matching |
| `Bash` | Shell command execution |
| `Web` | HTTP requests |
| `API` | External API calls |
| `MCP` | Model Context Protocol servers |

#### Dependencies Fields

| Field | Type | Description |
|-------|------|-------------|
| `requires` | string[] | Agents that must run before this one (hard dependency) |
| `recommends` | string[] | Agents that should run before this one (soft dependency) |
| `conflicts` | string[] | Agents that cannot run alongside this one |

---

### Epistemic Nature Block

The `epistemic_nature` block classifies the agent's epistemic characteristics across three independent axes. Each axis is optional — agents may declare one, two, or all three. This classification supports empirical analysis of agent effectiveness (e.g., false-positive rate correlation with verifiability) and helps consumers understand what kind of claims an agent makes.

```yaml
epistemic_nature:
  verifiability: mechanically_checkable
  determinism: stochastic
  claim_type: factual
```

#### Axes

| Axis | Values | Question Answered |
|------|--------|-------------------|
| `verifiability` | `mechanically_checkable`, `expert_judgment`, `not_checkable` | Can a third party confirm the finding? |
| `determinism` | `deterministic`, `stochastic`, `environment_dependent` | Same input → same output? |
| `claim_type` | `factual`, `normative`, `observational` | What kind of assertion does the agent make? |

#### Verifiability

| Value | Description | Examples |
|-------|-------------|----------|
| `mechanically_checkable` | Binary truth claims that can be verified by automated tooling. The finding is either correct or incorrect with no judgment required. | Type compiles, file exists, test passes, export present |
| `expert_judgment` | Findings that mix verifiable facts with judgment calls. A domain expert could evaluate the finding, but reasonable experts might disagree. | Security vulnerability severity, API design quality, test coverage adequacy |
| `not_checkable` | Recommendations, observations, or opinions where "correct" is not well-defined. Value is in provoking thought, not in being right. | Architecture suggestions, assumption surfacing, pattern observations |

#### Determinism

| Value | Description | Examples |
|-------|-------------|----------|
| `deterministic` | Given identical input, the agent produces identical findings every time. | Linter, type checker, schema validator |
| `stochastic` | LLM-based variance means findings may differ across runs on the same input. Most AI agents are stochastic. | Code review, security analysis, prompt quality |
| `environment_dependent` | Findings depend on external runtime state that changes independently of the input artifact. | Chaos testing (service availability), load testing (current throughput), runtime validation (API responses) |

#### Claim Type

| Value | Description | Examples |
|-------|-------------|----------|
| `factual` | Claims about what IS — the state of the artifact as observed. Can be true or false. | "This function has no error handling", "The export is missing" |
| `normative` | Claims about what SHOULD BE — prescriptive recommendations. Cannot be "false" in the same way as factual claims. | "This design should use composition over inheritance", "Consider adding retry logic" |
| `observational` | Claims about what was NOTICED — patterns, structures, or phenomena surfaced for the reader's consideration. | "These three modules share a similar structure", "This assumption is present but unstated" |

#### Classification Guidelines

When classifying an agent:
1. **Verifiability:** Look at the agent's scoring criteria and auto-fail conditions. If they reference binary checks (file exists, test passes), lean toward `mechanically_checkable`. If they reference qualitative judgment ("is this clear?", "is this severe?"), lean toward `expert_judgment`.
2. **Determinism:** Almost all LLM-based agents are `stochastic`. Use `deterministic` only for agents that wrap deterministic tooling (linters, compilers). Use `environment_dependent` only for agents that execute live tests against running systems.
3. **Claim Type:** Look at the agent's output format. Scores and pass/fail verdicts suggest `factual`. Recommendations and suggestions suggest `normative`. Pattern inventories and exploration reports suggest `observational`.

#### Example Classifications

| Agent | Verifiability | Determinism | Claim Type |
|-------|--------------|-------------|------------|
| code-validator | mechanically_checkable | stochastic | factual |
| security-analyst | expert_judgment | stochastic | factual |
| code-optimizer | not_checkable | stochastic | normative |
| chaos-validator | mechanically_checkable | environment_dependent | factual |
| assumption-excavator | not_checkable | stochastic | observational |
| pre-implementation-architect | expert_judgment | stochastic | normative |

---

### Defaults Block

The `defaults` block specifies runtime configuration when the agent is invoked directly (not wrapped in a Command).

```yaml
defaults:
  model: sonnet
  timeout: 300000        # 5 minutes in ms
  max_tokens: 16000      # NEW in v1.1.0
  temperature: 0         # NEW in v1.1.0
```

#### Field Definitions

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `model` | enum/string | `sonnet` | LLM model: `haiku`, `sonnet`, `opus`, or versioned (e.g., `sonnet-4.5`) |
| `timeout` | number | `300000` | Maximum execution time in milliseconds |
| `max_tokens` | number | — | Maximum tokens for model response (1000-200000) |
| `temperature` | number | `0` | Model temperature (0 = deterministic, 1 = creative) |

**Model Versioning (v1.1.0):** Models can be specified with version qualifiers:
- `sonnet` — Use current default Sonnet
- `sonnet-4.5` — Use specific Sonnet version

**Note:** When an agent is wrapped in a Command, the Command's execution settings override these defaults.

---

### Context Block

**NEW in v1.1.0**

The `context` block configures the execution environment.

```yaml
context:
  working_directory: "./src"
  environment:
    NODE_ENV: "test"
    DEBUG: "true"
  timeout_behavior: fail
  shell: bash
```

#### Field Definitions

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `working_directory` | string | `.` | Base directory for relative paths |
| `environment` | object | `{}` | Environment variables to set |
| `timeout_behavior` | enum | `fail` | Behavior when timeout reached: `fail`, `warn`, `continue` |
| `shell` | enum | `bash` | Shell for command execution: `bash`, `sh`, `zsh`, `powershell` |
| `data_sources` | DataSource[] | `[]` | External data sources the agent can access (v1.5.1) |

#### Data Sources (v1.5.1)

The `data_sources` field documents external APIs, MCP servers, or CLI tools the agent can query for information.

```yaml
context:
  data_sources:
    - name: "uluops-tracker MCP Server"
      description: "Primary source for validation history and issue tracking"
      tools:
        - name: query_issues
          purpose: "List open/completed issues by project, priority, validator"
          usage: "query_issues(project: string, status?: 'open'|'completed')"
        - name: get_run_details
          purpose: "View recommendations from a specific validation run"
          usage: "get_run_details(project: string, run_number?: number)"

    - name: "Git History"
      description: "Version control for agent definition changes"
      commands:
        - command: "git log --follow --oneline -- agents/*.md"
          purpose: "Track agent file modification history"
        - command: "git diff HEAD~N -- udl/adl/v*/*.yaml"
          purpose: "Compare YAML changes between versions"
```

**DataSource Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | ✔ | Display name of the data source |
| `description` | string | ✔ | What this data source provides |
| `tools` | Tool[] | No | MCP tools or API endpoints available |
| `commands` | Command[] | No | CLI commands for accessing data |

**Tool Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | ✔ | Tool or endpoint name |
| `purpose` | string | ✔ | What this tool does |
| `usage` | string | No | Example usage or function signature |

**Command Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `command` | string | ✔ | The shell command |
| `purpose` | string | ✔ | What this command does |

#### Timeout Behaviors

| Behavior | Description |
|----------|-------------|
| `fail` | Immediately fail the agent on timeout |
| `warn` | Log warning but attempt to continue |
| `continue` | Silently continue (use with caution) |

---

### Mission Block

**NEW in v1.2.0**

The `mission` block captures the agent's identity, purpose framing, and behavioral boundaries. While optional, including a mission block significantly improves agent effectiveness during YAML→Agent translation.

```yaml
mission:
  opener: "You are performing a COMPREHENSIVE security audit of a completed project before deployment."
  stakes: "This is a critical gate. A thorough audit prevents vulnerabilities from reaching production."
  outcome_framing: "Provide a SECURE/CONDITIONAL/BLOCKED decision with an objective numerical score."
  role_boundaries:
    - "You are an analyst - identify and classify issues objectively"
    - "Do not auto-fail for tooling issues outside the codebase"
  taxonomy_mandate: true
  # NEW in v1.5.0
  vocabulary_rationale: >
    Uses CONSISTENT/INCONSISTENT instead of PASS/FAIL because state validation
    is about data consistency across requests, not binary correctness.
  explicit_prohibitions:
    - "Do NOT proceed if prerequisite runtime-validator failed or was skipped"
    - "Do NOT test single-request endpoints—use runtime-validator instead"
    - "Do NOT downgrade session isolation failures—they are always critical"
  # NEW in v1.7.0
  epistemic_limitations:
    - "You infer assumptions from text, not from the author's mental state"
    - "Your own analysis carries assumptions about the taxonomy's sufficiency"
```

#### Field Definitions

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `opener` | string | No | Present-tense immersive opening statement (20-2000 chars) |
| `stakes` | string | No | Why this validation matters - consequences of pass/fail (max 1000 chars) |
| `outcome_framing` | string | No | What the agent produces - decision format and deliverables (max 1000 chars) |
| `role_boundaries` | string[] | No | Clarifications about agent scope and limitations (max 5 items, 200 chars each) |
| `taxonomy_mandate` | boolean | No | Whether every issue must include failure taxonomy classification (default: `true`) |
| `vocabulary_rationale` | string | No | Explains why the decision vocabulary was chosen for this domain (max 2000 chars) - v1.5.0 |
| `explicit_prohibitions` | string[] | No | Actions the agent must NOT do - hard boundaries to prevent overreach (max 10 items, 200 chars each) - v1.5.0 |
| `epistemic_limitations` | string[] | No | Explicit boundaries of what the agent cannot assess or guarantee (max 10 items, 1000 chars each) - v1.7.0 |

#### Vocabulary Rationale (v1.5.0)

The `vocabulary_rationale` field explains why a particular decision vocabulary (PASS/FAIL, CONSISTENT/INCONSISTENT, SECURE/BLOCKED, etc.) was chosen for this agent. This helps users understand the semantic meaning behind decisions.

```yaml
vocabulary_rationale: >
  Uses CONSISTENT/INCONSISTENT instead of PASS/FAIL because state validation
  is about data consistency across requests, not binary correctness. A workflow
  that completes but has stale reads is "inconsistent" not "broken."
```

#### Explicit Prohibitions (v1.5.0)

The `explicit_prohibitions` field defines hard boundaries for agent behavior - actions it must NOT do. This prevents scope creep and ensures agents stay within their designated responsibility.

```yaml
explicit_prohibitions:
  - "Do NOT proceed if prerequisite runtime-validator failed or was skipped"
  - "Do NOT test single-request endpoints—use runtime-validator instead"
  - "Do NOT test cross-service integrations—use integration-validator instead"
  - "Do NOT downgrade session isolation failures—they are always critical"
```

**Best Practices:**
- Start each prohibition with "Do NOT" for consistency
- Be specific about what alternative agent or action should be used instead
- Include prohibitions for common scope violations in your domain
- Limit to 10 items maximum to avoid overwhelming the agent

#### Epistemic Limitations (v1.7.0)

The `epistemic_limitations` field explicitly acknowledges what the agent cannot know, assess, or guarantee. This prevents overconfident outputs and helps users understand the boundaries of the agent's analysis.

```yaml
epistemic_limitations:
  - >
    You infer assumptions from text, not from the author's mental state. You cannot
    know what the author was aware of — only what the text takes for granted.
  - >
    Your own analysis carries assumptions: that the taxonomy is sufficient, that
    the methodology produces distinct findings, and that scores are calibrated.
```

**Best Practices:**
- Frame as genuine knowledge boundaries, not disclaimers
- Acknowledge both domain limitations ("cannot verify runtime behavior") and meta-limitations ("our taxonomy may be incomplete")
- Keep each limitation to 1-2 sentences for clarity
- Useful for analyst agents where epistemic humility improves output quality

#### Translation Behavior

When translating YAML to executable agent:

1. **With mission block**: Render opener as first paragraph, stakes as second, outcome_framing in "Your Mission" section
2. **Without mission block**: Generate generic opener from interface.description

---

### Knowledge Base Block

**NEW in v1.2.0**

The `knowledge_base` block embeds domain expertise that transforms structural validation into effective judgment. Each section corresponds to a scoring category and contains patterns, examples, and references.

```yaml
knowledge_base:
  global_references:
    - "OWASP Top 10 2021"
    - "CWE Top 25 2023"
  sections:
    - category_ref: secrets_credentials
      what_to_check:
        - "Hardcoded API keys, passwords, tokens"
        - "AWS credentials (AKIA pattern)"
        - "Private keys (.pem, .key files)"
      detection_patterns:
        - label: "CRITICAL - Hardcoded secrets"
          pattern: "(const|let|var)\\s+\\w*(key|secret|password)\\w*\\s*=\\s*['\"][^'\"]{10,}['\"]"
          severity: critical
          false_positive_hint: "Check if value is a placeholder like 'your-key-here'"
        - label: "CRITICAL - AWS Access Key"
          pattern: "AKIA[0-9A-Z]{16}"
          severity: critical
      red_flags:
        - description: "Hardcoded API key"
          code: |
            // CRITICAL: Hardcoded secrets
            const API_KEY = "sk-abc123..."
            const password = "admin123"
          language: javascript
          severity: critical
          why: "Secrets in source code can be extracted from repositories, logs, or compiled binaries"
      safe_patterns:
        - description: "Environment variables with validation"
          code: |
            // GOOD: Environment variables only
            const apiKey = process.env.API_KEY
            if (!apiKey) throw new Error('API_KEY required')
          language: javascript
          why: "Secrets stay outside codebase, missing secrets fail fast"
      references:
        - "CWE-798: Use of Hard-coded Credentials"
        - "OWASP A07:2021 - Identification and Authentication Failures"
      common_mistakes:
        - mistake: "Using default/fallback secrets"
          why_wrong: "Default secrets defeat the purpose of environment variables"
          correct_approach: "Fail fast if required secrets are missing"
```

#### Knowledge Base Fields

| Field | Type | Description |
|-------|------|-------------|
| `sections` | KnowledgeSection[] | Knowledge sections, typically one per scoring category |
| `failure_code_examples` | FailureCodeExample[] | Examples mapping issues to failure taxonomy codes (v1.3.0) |
| `global_references` | string[] | Standards that apply across all sections (e.g., OWASP Top 10) |
| `key_definitions` | Record&lt;string, string&gt; | Key term definitions used throughout the agent's analysis (v1.7.0) |
| `domain_taxonomy` | DomainTaxonomy | Domain-specific classification taxonomy with categories and rating scale (v1.7.0) |

#### Failure Code Examples (v1.3.0)

The `failure_code_examples` field provides worked examples of how to classify issues using the failure taxonomy. This helps LLMs consistently apply the correct codes.

```yaml
knowledge_base:
  failure_code_examples:
    - issue: "Function performs both validation AND database write"
      failure_code: "PRA-FRA/M"
      explanation: >
        Domain: Pragmatic (code works but is fragile)
        Mode: FRA (Fragility - poor separation makes testing/maintenance hard)
        Severity: M (Medium - not blocking, but should fix)

    - issue: "Missing null check before user.email access"
      failure_code: "SEM-COM/H"
      explanation: >
        Domain: Semantic (incomplete handling of case)
        Mode: COM (Incompleteness - null case not handled)
        Severity: H (High - will crash in production)
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `issue` | string | ✔ | Description of the issue |
| `failure_code` | string | ✔ | Full failure code with severity (pattern: `^(STR|SEM|PRA|EPI)-[A-Z]{3}/[CHMLI]$`) |
| `explanation` | string | No | Detailed explanation of why this code applies |

#### Key Definitions (v1.7.0)

The `key_definitions` field provides a glossary of terms used throughout the agent's analysis. This ensures the LLM and users share a consistent vocabulary.

```yaml
knowledge_base:
  key_definitions:
    artifact: >
      Any document, configuration, specification, code, plan, prompt, or structured
      output that encodes decisions and carries implicit assumptions.
    fragility: >
      The degree to which an assumption, if violated, would cause cascading or
      hard-to-diagnose failures rather than clean, containable ones.
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| Key (string) | — | — | The term being defined |
| Value (string) | — | — | The definition of the term |

#### Domain Taxonomy (v1.7.0)

The `domain_taxonomy` field defines a domain-specific classification system with categories and an optional rating scale. This is used by analyst agents to structure their findings.

```yaml
knowledge_base:
  domain_taxonomy:
    completeness_note: >
      These five categories are a starting framework, not an exhaustive partition.
      Create ad-hoc categories rather than force-fitting.
    categories:
      - id: ENV
        name: Environmental
        description: "Assumptions about the execution environment"
      - id: DEP
        name: Dependency
        description: "Assumptions about external dependencies"
    scale:
      name: Assumption Fragility Scale
      description: "How badly would things break if this assumption were violated?"
      anti_bias_guidance: >
        Resist anchoring on the first fragility estimate. Consider both the
        probability of violation AND the blast radius.
      levels:
        - score: "5"
          label: Catastrophic
          meaning: "System-level failure, unrecoverable without redesign"
          indicators:
            - "No fallback path exists"
            - "Failure cascades to multiple components"
        - score: "1"
          label: Trivial
          meaning: "Minor inconvenience, easy workaround exists"
```

**DomainTaxonomy Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `completeness_note` | string | No | Caveat about taxonomy coverage |
| `categories` | TaxonomyCategory[] | ✔ | Classification categories |
| `scale` | TaxonomyScale | No | Rating scale for findings (alias: `fragility_scale`) |

**TaxonomyCategory Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | No | Short identifier (e.g., "ENV", "DEP") |
| `name` | string | ✔ | Display name |
| `description` | string | No | What this category covers |
| `items` | string[] | No | Example items in this category |

**TaxonomyScale Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | No | Scale name (e.g., "Fragility Scale") |
| `description` | string | No | What the scale measures |
| `anti_bias_guidance` | string | No | Guidance to prevent scoring bias |
| `levels` | TaxonomyLevel[] | ✔ | Scale levels from highest to lowest |

**TaxonomyLevel Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `score` | string | No | Numeric score value (alternative: `level`) |
| `label` | string | ✔ | Level name (e.g., "Catastrophic", "Trivial") |
| `meaning` | string | No | What this level means (alternative: `description`) |
| `indicators` | string[] | No | Observable signs of this level |

#### Knowledge Section Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `category_ref` | string | ✔ | References `scoring.categories[].id` |
| `description` | string | No | Description of what this knowledge section covers (v1.7.0) |
| `what_to_check` | string[] | No | Checklist of items to evaluate |
| `detection_patterns` | DetectionPattern[] | No | Patterns for identifying issues vs. false positives |
| `red_flags` | CodeExample[] | No | Examples of problematic patterns |
| `safe_patterns` | CodeExample[] | No | Examples of correct implementations |
| `references` | string[] | No | External references (CWE, OWASP, RFCs) |
| `common_mistakes` | Mistake[] | No | Frequent errors with corrections |

#### Detection Pattern Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `label` | string | ✔ | Human-readable pattern name (e.g., "CRITICAL - Hardcoded secrets") |
| `description` | string | No | When this pattern applies |
| `pattern` | string | No | Regex or glob pattern for detection |
| `severity` | enum | ✔ | `critical`, `high`, `medium`, `low`, `info` |
| `false_positive_hint` | string | No | How to distinguish from false positives |

#### Code Example Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `description` | string | ✔ | What this example demonstrates |
| `code` | string | ✔ | Code snippet (may include comments) |
| `language` | string | No | Programming language (default: `javascript`) |
| `severity` | enum | No | For red_flags: severity of this pattern |
| `why` | string | No | Explanation of why this is good/bad |
| `anti_pattern` | boolean | No | Whether this example demonstrates an anti-pattern (v1.7.0) |

#### Common Mistake Fields

| Field | Type | Description |
|-------|------|-------------|
| `mistake` | string | Description of the common mistake |
| `why_wrong` | string | Explanation of why it's problematic |
| `correct_approach` | string | How to do it correctly |

---

### Scoring Block (Validators)

The `scoring` block defines the 100-point evaluation framework for validators.

```yaml
scoring:
  maxScore: 100

  # NEW in v1.3.0: Calibration examples for consistent LLM scoring
  calibration_examples:
    - score: 95
      scenario: "Clean phase with minor style issues"
      description: "All tests pass, no security issues, good error handling. Only issues: 2 functions slightly over 50 lines."
      deductions:
        - criterion: single_purpose_functions
          points_lost: 3
          reason: "2 functions at 55-60 lines"
        - criterion: documentation_present
          points_lost: 2
          reason: "1 exported function missing JSDoc"

    - score: 75
      scenario: "Acceptable phase with moderate issues"
      description: "Tests pass but coverage incomplete. Some error handling gaps in non-critical paths."

    - score: 55
      scenario: "Failing phase with critical issues"
      description: "Has security issue, missing tests for core functionality, multiple error handling gaps."

  categories:
    - id: code_quality
      name: Code Quality
      weight: 30
      description: Naming, structure, and maintainability
      criteria:
        - id: naming_conventions
          name: Consistent naming conventions
          points: 10
          failure_taxonomy:
            domain: semantic
            failure_mode: SEM-AMB
            default_severity: medium
          verification:
            method: hybrid
            checks:
              - Variables use camelCase
              - Classes use PascalCase
              - Constants use SCREAMING_SNAKE_CASE
            automation:
              tool: ast
              pattern: "naming-convention-check"
              
        - id: no_duplication
          name: No code duplication
          points: 10
          verification:
            method: automated
            automation:
              tool: jscpd
              threshold: 5  # Max 5% duplication
              
        - id: error_handling
          name: Proper error handling
          points: 10
          verification:
            method: manual
            checks:
              - All async operations have try-catch
              - Errors are logged with context
              - User-facing errors are sanitized
              
    - id: best_practices
      name: Best Practices
      weight: 25
      criteria:
        - id: solid_principles
          name: SOLID principles adherence
          points: 15
          
        - id: abstraction_levels
          name: Appropriate abstraction levels
          points: 10
          
  # NEW in v1.7.0: Score interpretation and weight rationale
  interpretation: >
    Scores above 70 indicate the artifact's assumption profile has been adequately
    surfaced. Scores below 50 suggest critical buried assumptions remain.
  weight_rationale: >
    Equal weighting (20/20/20/20/20) prevents systematic bias toward any single
    category across diverse artifact types.

  constraints:
    min_categories: 4
    max_categories: 6
    min_category_weight: 10
    max_category_weight: 30
    min_criterion_points: 2
    max_criterion_points: 10
    total_must_equal: 100
```

#### Category Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | ✔ | Snake_case identifier |
| `name` | string | ✔ | Display name |
| `weight` | number | ✔ | Points for this category (10-30) |
| `description` | string | | What this category evaluates |
| `criteria` | Criterion[] | ✔ | Individual evaluation criteria (1-10 items) |

#### Criterion Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | ✔ | Snake_case identifier |
| `name` | string | ✔ | What is being checked |
| `points` | number | ✔ | Points for this criterion (2-10) |
| `description` | string | | Detailed description |
| `failure_taxonomy` | object | | Failure classification (see Failure Taxonomy) |
| `verification` | object | | How to verify (method, checks, automation) |

#### Scoring Constraints

| Constraint | Value | Rationale |
|------------|-------|-----------|
| Total points | Must equal 100 | Standardized scoring |
| Category count | 4-6 | Balanced granularity |
| Category weight | 10-30 each | No single category dominates |
| Criterion points | 2-10 each | Meaningful differentiation |
| Criteria per category | 1-10 | Manageable verification |

#### Calibration Examples (v1.3.0)

Calibration examples provide score-to-scenario mappings that help LLMs score consistently. Each example shows what a specific score "looks like" in practice.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `score` | integer | ✔ | Expected score for this scenario (0-100) |
| `scenario` | string | ✔ | Short label describing the scenario |
| `description` | string | No | Detailed explanation of the scenario |
| `deductions` | Deduction[] | No | Specific point deductions that led to this score |

**Deduction Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `criterion` | string | ✔ | The criterion id that was deducted |
| `points_lost` | integer | ✔ | Points deducted for this criterion (1-10) |
| `reason` | string | No | Explanation of why points were deducted |

**Best Practices:**
- Include 3 examples spanning the score range (e.g., 95, 75, 55)
- High score example shows near-perfect with minor issues
- Middle score example shows passing threshold with moderate issues
- Low score example shows failing with critical issues

#### Score Interpretation (v1.7.0)

The `interpretation` field provides guidance on how to interpret the total score. This helps both LLMs and human readers understand what a specific score range means in context.

```yaml
scoring:
  interpretation: >
    Scores above 70 indicate the artifact's assumption profile has been adequately
    surfaced. Scores below 50 suggest critical buried assumptions remain that could
    cause failure before anyone notices them.
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `interpretation` | string | No | How to interpret the total score (max 3000 chars) |

#### Weight Rationale (v1.7.0)

The `weight_rationale` field explains why categories are weighted as they are. This is particularly important when weights are non-obvious or when equal weighting is a deliberate choice.

```yaml
scoring:
  weight_rationale: >
    Equal weighting (20/20/20/20/20) is a deliberate default, not a claim that
    all categories are equally important for every artifact. When a category is
    less relevant, note this rather than leaving it unscored.
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `weight_rationale` | string | No | Explanation of why categories are weighted as they are (max 3000 chars) |

#### Verification Object

```yaml
verification:
  method: manual | automated | hybrid
  checks:
    - "Human-readable verification step"
  automation:
    tool: grep | ast | linter | test | custom | tsc | npm-audit | coverage | jscpd | glob | json-schema | api-extractor | eslint | pylint | ruff | prettier | black
    pattern: "regex or tool-specific pattern"
    command: "shell command to execute"
    threshold: 0  # Max matches for pass
    timeout: 30000  # Tool-specific timeout
  # NEW in v1.2.0:
  verify_guidance: "Count files matching vs. not matching convention"
  partial_scoring:
    - points: 5
      condition: "≥95% adherence"
      label: "Full credit"
    - points: 3
      condition: "80-94% adherence"
      label: "Minor inconsistencies"
  definition_clarifications:
    - term: "adherence"
      definition: "Files matching convention / Total files in category"
```

#### Enhanced Verification Fields (v1.2.0)

| Field | Type | Description |
|-------|------|-------------|
| `verify_guidance` | string | Explicit HOW-TO instructions for verification (max 1000 chars). Renders as `*Verify:*` in agent. |
| `partial_scoring` | PartialScore[] | Graduated scoring rules. Renders as `*Partial:*` in agent. |
| `definition_clarifications` | Definition[] | Clarifies ambiguous terms. Renders as `*Definition:*` in agent. |
| `formula` | string | Mathematical formula or algorithm for computing scores (v1.5.1). Renders in code block. |
| `thresholds` | object | Named numeric thresholds for verification checks (v1.5.1). Renders as key=value pairs. |

#### Formula and Thresholds (v1.5.1)

For complex criteria requiring mathematical computation, use `formula` to document the calculation and `thresholds` to define named numeric values.

```yaml
criteria:
  - id: improvement_patterns
    name: "Score improvement patterns identified"
    points: 10
    verification:
      method: manual
      checks:
        - "≥3 distinct improvement patterns documented"
        - "Each pattern has ≥3 supporting examples"
        - "Confidence score calculated for each pattern"
      formula: |
        Confidence = (occurrences / total_possible_occurrences) × (1 - variance_coefficient)
        Where variance_coefficient = stddev(score_lifts) / mean(score_lifts)
        Thresholds: ≥80% = high confidence, 50-79% = medium, <50% = low
      thresholds:
        high_confidence: 80
        medium_confidence: 50
        minimum_occurrences: 3

  - id: temporal_coverage
    name: "Sufficient temporal coverage"
    points: 8
    verification:
      method: hybrid
      checks:
        - "Query returns ≥5 validation runs per agent"
        - "Time span covers ≥30 days"
      thresholds:
        minimum_runs_per_agent: 5
        minimum_days_coverage: 30
        maximum_gap_days: 7
```

The `formula` field renders as a code block in the generated agent markdown. The `thresholds` field renders as a comma-separated list of key=value pairs.

#### Partial Score Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `points` | integer | ✔ | Points awarded when condition is met |
| `condition` | string | ✔ | Human-readable condition (e.g., "≥95% adherence") |
| `condition_expression` | string | No | Machine-parseable condition (e.g., "adherence >= 0.95") |
| `label` | string | No | Optional label for this tier (e.g., "Full credit") |

#### Definition Clarification Fields

| Field | Type | Description |
|-------|------|-------------|
| `term` | string | The term being defined |
| `definition` | string | Clear definition of the term |

---

### Tasks Block (Executors)

The `tasks` block defines the execution model for executors.

```yaml
tasks:
  inputs:
    - id: issues
      name: Issues to Fix
      type: array
      itemType: Issue
      description: List of issues from validation run
      required: true
      source: file
      validation:                         # NEW in v1.1.0
        min: 1
        
    - id: scope
      name: Fix Scope
      type: enum
      values: [file, category, all]
      default: file
      description: How broadly to apply fixes
      
    - id: dry_run
      name: Dry Run Mode
      type: boolean
      default: false
      description: Preview changes without applying
      
  operations:
    - id: analyze_issues
      name: Analyze Issues
      description: Group and prioritize issues by fixability
      depends_on: []
      
    - id: generate_fixes
      name: Generate Fixes
      description: Create fix patches for each issue
      depends_on: [analyze_issues]
      retry:                              # NEW in v1.1.0
        max_attempts: 2
        backoff_ms: 1000
      
    - id: apply_fixes
      name: Apply Fixes
      description: Apply patches to source files
      depends_on: [generate_fixes]
      condition: "!inputs.dry_run"
      rollback_on_failure: true           # NEW in v1.1.0
      
    - id: verify_fixes
      name: Verify Fixes
      description: Run quick validation on fixed files
      depends_on: [apply_fixes]
      
  outputs:
    - id: fixes_applied
      name: Fixes Applied
      type: array
      itemType: FixResult
      description: List of successfully applied fixes
      required: true                      # NEW in v1.1.0
      
    - id: fixes_skipped
      name: Fixes Skipped
      type: array
      itemType: SkippedFix
      description: Issues that couldn't be auto-fixed
      required: false                     # NEW in v1.1.0
      
    - id: diff
      name: Change Diff
      type: string
      format: unified-diff
      description: Git-style diff of all changes
```

#### Input Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | ✔ | Snake_case identifier |
| `name` | string | ✔ | Display name |
| `type` | enum | ✔ | `string`, `number`, `boolean`, `array`, `object`, `enum`, `file`, `path` |
| `itemType` | string | | For arrays: type of items |
| `values` | string[] | | For enums: allowed values |
| `default` | any | | Default value if not provided |
| `required` | boolean | | Whether input is mandatory (default: `false`) |
| `source` | enum | | `file`, `argument`, `previous_phase`, `environment`, `stdin`, `prompt` |
| `description` | string | | What this input is for |
| `validation` | object | | Input validation rules (v1.1.0) |

#### Input Validation (v1.1.0)

```yaml
validation:
  pattern: "^[a-z]+$"    # Regex for strings
  min: 1                  # Min value/length
  max: 100                # Max value/length
  custom: "value.length > 0"  # Custom expression
```

#### Operation Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | ✔ | Snake_case identifier |
| `name` | string | ✔ | Display name |
| `description` | string | ✔ | What this operation does |
| `depends_on` | string[] | | Operation IDs that must complete first |
| `condition` | string | | Expression for conditional execution |
| `timeout` | number | | Operation-specific timeout (ms) |
| `retry` | RetryConfig | | Retry configuration (v1.1.0) |
| `rollback_on_failure` | boolean | | Trigger rollback if this operation fails (default: `false`) |

#### Output Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | ✔ | Snake_case identifier |
| `name` | string | ✔ | Display name |
| `type` | enum | ✔ | `string`, `number`, `boolean`, `array`, `object`, `file`, `path` |
| `itemType` | string | | For arrays: type of items |
| `format` | string | | Format hint (e.g., `unified-diff`, `json`, `path`, `markdown`) |
| `description` | string | | What this output contains |
| `required` | boolean | | Whether this output must be produced (default: `true`) |

---

### Decisions Block

The `decisions` block maps scores to decision outcomes for validators.

```yaml
decisions:
  vocabulary:
    positive: PASS
    negative: FAIL
    conditional: WARN  # Optional

  thresholds:
    - decision: positive
      min_score: 80
      label: "Ready for deployment"

    - decision: conditional
      min_score: 60
      max_score: 79
      label: "Needs attention"

    - decision: negative
      max_score: 59
      label: "Requires fixes"

  # Alternative: use a preset
  # preset: quality_gate

  tracking:
    category: gate
    notify_on:                            # NEW in v1.1.0
      - negative

  # NEW in v1.7.0: Guidance for edge cases near decision boundaries
  decision_guidance: >
    Scores near 70 require careful judgment. If the artifact's critical assumptions
    have been surfaced but categorization is incomplete, lean toward EXAMINED.
    If critical assumptions were missed entirely, lean toward UNEXAMINED.

  # NEW in v1.5.0: Explicit success criteria beyond score
  success_criteria:
    description: "A workflow functions correctly when ALL of the following are true"
    criteria:
      - "All workflow steps execute without 5xx errors"
      - "State persists correctly after each write operation"
      - "Final state matches expected outcome"
      - "No auto-fail conditions are triggered"
```

#### Decision Presets

| Preset | Pass | Warn | Fail | Use Case |
|--------|------|------|------|----------|
| `low_risk` | ≥60 | 40-59 | <40 | Documentation, style |
| `quality_gate` | ≥70 | 50-69 | <50 | Standard code quality |
| `high_stakes` | ≥80 | 60-79 | <60 | Production deployments |
| `security` | ≥85 | 70-84 | <70 | Security audits |
| `critical` | ≥90 | 80-89 | <80 | Safety-critical systems |

#### Decision Tracking

| Field | Type | Description |
|-------|------|-------------|
| `category` | enum | How decisions are tracked: `gate`, `safety`, `advisory` |
| `notify_on` | string[] | Decision outcomes that trigger notifications (v1.1.0) |

**Tracking Categories:**

| Category | Vocabulary | Description |
|----------|------------|-------------|
| `gate` | Binary (PASS/FAIL) | Hard gate that blocks progression |
| `safety` | Ternary (PASS/WARN/FAIL) | Safety checks with warning state |
| `advisory` | Any | Non-blocking recommendations |

#### Decision Guidance (v1.7.0)

The `decision_guidance` field provides nuanced guidance for edge cases near decision boundaries. While thresholds define the numeric cutoffs, decision guidance helps the LLM make judgment calls when scores fall near boundaries.

```yaml
decision_guidance: >
  Scores near 70 require careful judgment. If the artifact's critical assumptions
  have been surfaced but categorization is incomplete, lean toward EXAMINED.
  If critical assumptions were missed entirely, lean toward UNEXAMINED regardless
  of overall score.
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `decision_guidance` | string | No | Guidance for edge cases near decision boundaries (max 2000 chars) |

**Best Practices:**
- Focus on the gray areas — clear pass/fail cases don't need guidance
- Reference specific score ranges where judgment matters
- Explain which factors should tip the balance in either direction
- Complement, don't contradict, the numeric thresholds

#### Success Criteria (v1.5.0)

The `success_criteria` field defines explicit conditions for a positive decision that go beyond just the score threshold. This is particularly useful for validators where certain conditions must be true regardless of score.

```yaml
success_criteria:
  description: "A workflow functions correctly when ALL of the following are true"
  criteria:
    - "All workflow steps execute without 5xx errors"
    - "State persists correctly after each write operation (verified by read-back)"
    - "Final state matches expected outcome"
    - "No error logs indicating silent failures"
    - "State is isolated between users/sessions"
    - "No auto-fail conditions are triggered"
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `description` | string | No | Introductory text explaining what must be true for success (max 1000 chars) |
| `criteria` | string[] | ✔ | List of conditions that must ALL be true for a positive decision (1-10 items, 200 chars each) |

**Best Practices:**
- Use success criteria to capture semantic requirements that scores don't fully express
- Each criterion should be independently verifiable
- Include both functional ("steps execute") and quality ("state persists") criteria
- Always include "No auto-fail conditions are triggered" if using auto-fail
- Keep criteria measurable - avoid vague terms like "works correctly"

**Relationship to Thresholds:**

Success criteria work alongside score thresholds. For a positive decision:
1. Score must meet the positive threshold (e.g., ≥75)
2. AND all success criteria must be true
3. AND no auto-fail conditions triggered

If score meets threshold but criteria fail, the decision is downgraded to conditional or negative.

---

### Completion Block (Executors)

The `completion` block maps execution states to decision outcomes for executors.

```yaml
completion:
  vocabulary:
    complete: COMPLETE
    partial: PARTIAL
    failed: FAILED
    
  criteria:
    - condition: "outputs.fixes_applied.length == inputs.issues.length"
      decision: complete
      label: "All issues fixed"
      
    - condition: "outputs.fixes_applied.length > 0"
      decision: partial
      label: "Some issues fixed"
      
    - condition: "outputs.fixes_applied.length == 0"
      decision: failed
      label: "No fixes applied"
```

---

### Auto-Fail Block

The `auto_fail` block defines conditions that immediately fail validation regardless of score.

```yaml
auto_fail:
  enabled: true

  conditions:
    - id: secrets_detected
      display_id: "AF-001"   # NEW in v1.3.0: Human-readable ID
      name: Hardcoded secrets detected
      severity: critical
      detection:
        method: pattern
        patterns:
          - "(?i)(api[_-]?key|secret|password)\\s*[:=]\\s*['\"][^'\"]{8,}"
          - "(?i)bearer\\s+[a-zA-Z0-9]{20,}"
        exclude_patterns:
          - "\\.example$"
          - "_test\\."
      remediation: Remove secrets and use environment variables
      
    - id: build_broken
      name: Build fails
      severity: critical
      detection:
        method: tool
        command: "npm run build 2>&1"
        failure_condition: "exit_code != 0"
      remediation: Fix compilation errors before validation
```

#### Auto-Fail Block Fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `enabled` | boolean | `true` | Whether auto-fail is enabled |
| `conditions` | array | — | List of auto-fail conditions |

#### Auto-Fail Condition Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | ✔ | Snake_case identifier |
| `display_id` | string | No | Human-readable numbered ID (e.g., "AF-001") - v1.3.0 |
| `name` | string | ✔ | Display name |
| `severity` | enum | ✔ | Must be `critical` |
| `detection` | object | ✔ | How to detect this condition |
| `category_override` | string | | Category to zero when triggered |
| `evidence_required` | boolean | | Whether to collect evidence (default: `true`) |
| `remediation` | string | | How to fix the issue |

**Display ID Pattern (v1.3.0):** The `display_id` field uses the pattern `^[A-Z]{2,4}-\d{3}$` (e.g., "AF-001", "AF-002"). This provides:
- Consistent, scannable identifiers in reports
- Easy reference in documentation and issues
- Sequential numbering for tracking

#### Detection Methods

| Method | Description | Fields |
|--------|-------------|--------|
| `pattern` | Regex pattern matching | `patterns`, `exclude_patterns` |
| `semantic` | LLM-based detection | `description` |
| `tool` | External command | `command`, `failure_condition` |

---

### Deductions Block

The `deductions` block defines severity-based point deductions.

```yaml
deductions:
  severity_scale:
    - level: critical
      points: auto_fail
      emoji: "🔴"
      label: "Critical"
      action: "Must fix immediately"
      
    - level: high
      points: -10
      emoji: "🟠"
      label: "High"
      action: "Fix before merge"
      
    - level: medium
      points: -5
      emoji: "🟡"
      label: "Medium"
      action: "Should fix"
      
    - level: low
      points: -2
      emoji: "🔵"
      label: "Low"
      action: "Consider fixing"
      
    - level: info
      points: 0
      emoji: "⚪"
      label: "Info"
      action: "For awareness"
```

---

### Rollback Block (Executors)

The `rollback` block configures recovery on executor failure.

```yaml
rollback:
  enabled: true
  strategy: git_restore
  preserve_logs: true                     # NEW in v1.1.0
  notify_on_rollback: true                # NEW in v1.1.0
```

#### Field Definitions

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `enabled` | boolean | `false` | Whether rollback is enabled |
| `strategy` | enum | — | Rollback strategy (see below) |
| `preserve_logs` | boolean | `true` | Keep execution logs after rollback |
| `notify_on_rollback` | boolean | `true` | Send notification when rollback occurs |

#### Rollback Strategies

| Strategy | Description |
|----------|-------------|
| `git_restore` | Use `git restore` to revert changes |
| `git_stash` | Use `git stash` to save and revert changes |
| `backup_restore` | Restore from backup files |
| `manual` | Require manual intervention |
| `none` | No rollback (fail in place) |

---

### Process Block

The `process` block defines the execution phases for the agent.

```yaml
process:
  # NEW in v1.4.0: Reasoning scaffolding for consistent evaluation
  reasoning_scaffolding:
    description: "For each criterion, follow this reasoning process"
    steps:
      - action: gather_evidence
        description: "List specific code locations that pass or fail the criterion"
        example: "Found 3 functions >50 lines: auth.js:120 (85 lines), users.js:45 (67 lines)"
      - action: apply_threshold
        description: "Compare against quantitative criteria from verification checks"
        example: "Threshold is 50 lines; 3 functions exceed it"
      - action: document_reasoning
        description: "Explain point deductions with file:line references"
        example: "Award 2/5 pts - 3 functions violate single-purpose"

  # NEW in v1.4.0: Pre-decision checklist for completeness
  pre_decision_checklist:
    - "Scored all categories"
    - "Every deduction has file:line reference"
    - "Every issue includes failure code from taxonomy"
    - "Checked all auto-fail conditions"
    - "Decision aligns with score AND critical issue presence"

  phases:
    - id: discovery
      name: Discovery
      description: Identify files and scope
      steps:
        - action: Detect project type
          tool: Glob
          command: "find . -name 'package.json' -o -name 'tsconfig.json'"
          output_variable: project_files
          
        - action: List source files
          tool: Glob
          command: "find ./src -name '*.ts' -o -name '*.tsx'"
          output_variable: source_files
          
    - id: analysis
      name: Analysis
      description: Evaluate against criteria
      retry:                              # NEW in v1.1.0: Phase-level retry
        max_attempts: 2
      steps:
        - action: Check naming conventions
          tool: Read
          condition: "{{ source_files.length > 0 }}"
          
        - action: Run linter
          tool: Bash
          command: "npm run lint -- --format json"
          commands:                       # NEW in v1.1.0: Conditional commands
            - condition: "project_type == 'node'"
              result: "npm run lint"
            - condition: "project_type == 'python'"
              result: "pylint src/"
          output_variable: lint_results
          patterns:                       # NEW in v1.1.0: Category patterns
            - category: code_quality
              grep: "error|warning"
          timeout: 60000                  # NEW in v1.1.0: Step timeout
          
    - id: scoring
      name: Scoring
      description: Calculate final score
      steps:
        - action: Aggregate category scores
        - action: Apply deductions
        - action: Check auto-fail conditions
        - action: Determine decision
```

#### Reasoning Scaffolding (v1.4.0, extended v1.7.0)

The `reasoning_scaffolding` block defines a structured reasoning process for LLMs to follow when evaluating criteria. This ensures consistent, traceable reasoning. Supports two modes: **steps** (sequential actions per criterion) and **passes** (multi-pass methodologies across the entire analysis).

| Field | Type | Description |
|-------|------|-------------|
| `description` | string | Explanation of the reasoning process |
| `steps` | ReasoningStep[] | Ordered steps in the reasoning process (v1.4.0) |
| `passes` | ReasoningPass[] | Multi-pass methodology for analysis (v1.7.0) |
| `pass_attribution_requirement` | string | Requirement for attributing findings to specific passes (v1.7.0) |

**Reasoning Step Fields (v1.4.0):**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `action` | string | ✔ | Snake_case identifier for the action |
| `description` | string | No | What this reasoning step accomplishes |
| `example` | string | No | Concrete example of this step in action |

**Reasoning Pass Fields (v1.7.0):**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | No | Short identifier for the pass |
| `name` | string | ✔ | Display name (e.g., "Structural Pass") |
| `question` | string | No | Guiding question for this pass |
| `focus` | string \| string[] | No | What to focus on — single string or list of items |
| `method` | string | No | How to conduct this pass |
| `output` | string | No | What this pass produces |
| `key_questions` | string[] | No | Specific questions to answer during this pass |

**Multi-Pass Example (v1.7.0):**

```yaml
process:
  reasoning_scaffolding:
    description: >
      Work through three sequential passes. Each pass targets a different
      layer. Do not merge passes — they look for different things.
    passes:
      - id: structural
        name: Structural Pass
        question: "What does the text take for granted about its own structure?"
        focus:
          - "Completeness assumptions"
          - "Ordering assumptions"
          - "Format assumptions"
        method: "Read each section and ask: what is NOT here that the text assumes IS here?"
        output: "List of structural assumptions with locations"
        key_questions:
          - "What sections or components are assumed to exist but not referenced?"
          - "What ordering dependencies are implicit?"

      - id: semantic
        name: Semantic Pass
        question: "What does the text take for granted about meaning?"
        focus: "Vocabulary, domain knowledge, shared context"
        output: "List of semantic assumptions"

    pass_attribution_requirement: >
      Each finding MUST list which pass discovered it. After completing all
      passes, verify that findings are distributed across passes.
```

**Steps vs. Passes:**

| Aspect | Steps (v1.4.0) | Passes (v1.7.0) |
|--------|----------------|-----------------|
| Scope | Per-criterion evaluation | Entire analysis methodology |
| Use case | Validators scoring criteria | Analysts with multi-pass methods |
| Structure | Sequential actions | Named passes with focus areas |
| Attribution | Not required | Required per `pass_attribution_requirement` |

**Best Practices:**
- Use **steps** for validators where each criterion follows the same reasoning process
- Use **passes** for analysts with distinct methodological phases
- Include concrete examples with file:line references
- For passes, always include `pass_attribution_requirement` to ensure findings are traceable
- Common steps: `gather_evidence`, `apply_threshold`, `adjust_for_context`, `document_reasoning`

#### Pre-Decision Checklist (v1.4.0)

The `pre_decision_checklist` is an array of strings that the LLM should verify before finalizing its decision. This prevents common mistakes like forgetting to check auto-fail conditions.

```yaml
pre_decision_checklist:
  - "Scored all 4 categories (30+25+25+20 = 100 possible)"
  - "Every deduction has file:line reference"
  - "Checked all 5 auto-fail conditions"
  - "Decision aligns with score AND critical issue presence"
```

#### Phase Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | ✔ | Snake_case identifier |
| `name` | string | ✔ | Display name |
| `description` | string | | What this phase accomplishes |
| `steps` | Step[] | | Ordered steps in this phase |
| `instructions` | string | | Free-form instructions (alternative to steps) |
| `retry` | RetryConfig | | Phase-level retry configuration (v1.1.0) |

#### Step Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `action` | string | ✔ | What this step does |
| `description` | string | | Detailed description |
| `tool` | enum | | `Read`, `Grep`, `Glob`, `Bash`, `Web`, `API`, `MCP` |
| `command` | string | | Command to execute |
| `commands` | array | | Conditional command selection (v1.1.0) |
| `condition` | string | | Jinja2-style condition |
| `output_variable` | string | | Variable to store output |
| `fallback` | string | | Action if step fails |
| `patterns` | array | | Category-based grep patterns (v1.1.0) |
| `timeout` | number | | Step-specific timeout in ms (v1.1.0) |
| `retry` | RetryConfig | | Step-level retry configuration (v1.1.0) |

---

### Output Block

The `output` block defines how results are formatted and presented.

```yaml
output:
  format: structured
  schema: validator-output-v1.1

  # NEW in v1.3.0: Token budget guidance
  token_budget:
    target: 2000
    max: 4000
    guidance: "Aim for concise reports. Expand only for complex phases with 10+ files."

  # NEW in v1.3.0: Explicit section ordering
  section_order:
    - header
    - score_summary
    - issues
    - decision
    - json_output

  sections:
    - id: header
      template: |
        # {{ interface.displayName }} Report
        
        **Target:** {{ target }}
        **Score:** {{ score }}/100
        **Decision:** {{ decision }}
        
    - id: categories
      template: |
        ## Category Breakdown
        
        | Category | Score | Max |
        |----------|-------|-----|
        {% for cat in categories %}
        | {{ cat.name }} | {{ cat.score }} | {{ cat.maxPoints }} |
        {% endfor %}
        
    - id: findings
      condition: "{{ findings.length > 0 }}"
      template: |
        ## Findings
        
        {% for finding in findings %}
        ### {{ finding.severity | upper }}: {{ finding.title }}
        
        **Location:** {{ finding.filePath }}:{{ finding.lineNumber }}
        
        {{ finding.description }}
        {% endfor %}
        
  classification:
    enabled: true
    taxonomy_version: "0.2.2"
    include_codes: true
    allow_secondary: true                 # NEW in v1.1.0
    
  structured:                             # For structured format
    enabled: true
    schema_ref: "validator-output-v1.1"
    include_metadata: true
    
  symbols:
    separator: "┅"
    box_top: "┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓"
    box_bottom: "┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛"
    decision_positive: "✅"
    decision_negative: "❌"
    decision_conditional: "⚠️"
```

#### Output Block Fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `format` | enum | `markdown` | Output format (see Output Formats) |
| `schema` | string | — | JSON schema reference |
| `token_budget` | TokenBudget | — | Output length guidance (v1.3.0) |
| `section_order` | string[] | — | Explicit ordering of output sections (v1.3.0) |
| `sections` | array | — | Template sections for output |
| `symbols` | object | — | Output formatting symbols |
| `classification` | object | — | Failure classification config |
| `structured` | object | — | Structured output config |
| `examples` | OutputExample[] | — | Example outputs demonstrating format (v1.4.0) |
| `templates` | Record&lt;string, string&gt; | — | Named output templates for different sections (v1.7.0) |
| `metrics` | Record&lt;string, MetricDefinition&gt; | — | Metric vocabulary for self-documenting system_metrics and epistemic_assessment keys (v1.12.0) |
| `implications` | object | — | Output implications section configuration (v1.10.0) |

#### Token Budget (v1.3.0)

The `token_budget` field provides output length guidance for LLMs.

| Field | Type | Description |
|-------|------|-------------|
| `target` | integer | Target token count for typical output (minimum: 100) |
| `max` | integer | Maximum token count - hard limit (minimum: 100) |
| `guidance` | string | Free-form guidance about when to expand/compress |

**Best Practices:**
- Set `target` to typical output length for clean runs
- Set `max` to 2x target for complex cases
- Use `guidance` to explain expansion triggers (e.g., "Expand for 10+ issues")

#### Section Order (v1.3.0)

The `section_order` array specifies explicit ordering of output sections by their `id`. Sections not listed appear after listed sections in definition order.

```yaml
section_order:
  - header
  - taxonomy_reference
  - score_summary
  - reasoning_trace
  - issues
  - auto_fail_check
  - decision
  - json_output
```

#### Output Examples (v1.4.0)

The `examples` field provides complete output examples that demonstrate the expected format. This helps LLMs generate consistent, correctly-structured outputs.

```yaml
output:
  examples:
    - scenario: "Phase with critical issue causing FAIL"
      input_summary: "2 files modified: src/auth/login.ts, src/api/users.ts"
      output: |
        🔍 VALIDATOR REPORT - PHASE 3

        Files Reviewed:
        - src/auth/login.ts
        - src/api/users.ts

        📊 Score: 65/100
        ...
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `scenario` | string | ✔ | Description of the scenario being demonstrated |
| `input_summary` | string | No | Brief summary of the input that produced this output |
| `output` | string | ✔ | The complete example output text |

#### Output Templates (v1.7.0)

The `templates` field provides named templates for different sections of the output. Unlike `sections` (which define the complete output structure), `templates` are reusable fragments that the LLM can reference when constructing output.

```yaml
output:
  templates:
    header: |
      # ASSUMPTION EXCAVATOR
      **Target:** {target}
      **Score:** {score}/100
      **Decision:** {decision}
    assumption_entry: |
      ### A{n}: {title}
      **Category:** {category} | **Fragility:** {fragility}/5
      **Evidence:** {evidence}
      **Implication:** {implication}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| Key (string) | — | — | Template name (e.g., "header", "assumption_entry") |
| Value (string) | — | — | Template content with placeholder variables |

**Best Practices:**
- Use templates for repeating structures (e.g., individual findings)
- Use `{placeholder}` syntax for variable substitution
- Complement `examples` — templates show structure, examples show completed output
- Keep template names descriptive (e.g., "decision_examined" vs. "template_1")

#### Output Implications (v1.10.0)

The `implications` field configures the output section where agents surface what their findings suggest, scoped to the agent's epistemic function. This replaces the generic "RECOMMENDATIONS" section with type-specific labels per `agent-output-implications-spec-v0.1.0`.

Implications must be expressed from within the agent's epistemic function — a precision rule, not a silence rule. An analyst surfaces what the analysis reveals; a forecaster surfaces what the trajectory suggests.

```yaml
output:
  implications:
    section_label: AUDIT IMPLICATIONS
    framing_question: "What do the four-cause gaps suggest about the artifact's structural and teleological coherence?"
    scope_boundary: "Must not prescribe implementation changes — surface what the analysis reveals"
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `section_label` | string | ✔ | Section heading matching agent type (e.g., AUDIT IMPLICATIONS, TRAJECTORY IMPLICATIONS, VALIDATION IMPLICATIONS, DISCOVERY IMPLICATIONS) |
| `framing_question` | string | No | The question this implications section answers, scoped to the agent's epistemic function |
| `scope_boundary` | string | No | What the agent must not do in implications (e.g., must not prescribe implementation changes) |

**Default Labels by Agent Type:**

| Agent Type | Default Section Label |
|------------|----------------------|
| `analyst` | AUDIT IMPLICATIONS |
| `forecaster` | TRAJECTORY IMPLICATIONS |
| `validator` | VALIDATION IMPLICATIONS |
| `explorer` | DISCOVERY IMPLICATIONS |

#### Metrics Vocabulary (v1.12.0)

The `metrics` field defines a vocabulary for the metric keys that an agent produces in `system_metrics` and `epistemic_assessment` output. This enables dashboards and consumers to render self-documenting metrics with human-readable labels, descriptions, and directional sentiment — without hardcoded knowledge of each agent's specific vocabulary.

**Problem solved:** Agent analysis data uses framework-specific keys like `calcifiedCount` (Nietzsche), `tensionHealthRatio` (Heraclitus), or `fs1PowerProjection` (epistemic failure signatures). These are meaningless to dashboard users without context. The metrics vocabulary provides that context at the definition level.

```yaml
output:
  metrics:
    # System metrics
    calcifiedCount:
      label: "Calcified Conventions"
      description: "Conventions persisting through inertia rather than active purpose. High counts suggest the codebase has accumulated practices nobody questions."
      type: integer
      sentiment: lower_is_better
      section: system_metrics
    activeReactiveRatio:
      label: "Active / Reactive Ratio"
      description: "Ratio of conventions driven by creative choice vs reactive response to incidents. Higher active ratios indicate intentional design."
      type: ratio
      sentiment: higher_is_better
      section: system_metrics
    conventionsTraced:
      label: "Conventions Traced"
      description: "Total number of conventions identified and genealogically analyzed."
      type: integer
      sentiment: neutral
      section: system_metrics
    creativeRecency:
      label: "Creative Recency"
      description: "Date of the most recent convention that was actively created rather than reactively adopted."
      type: date
      sentiment: neutral
      section: system_metrics
    accumulationTrajectory:
      label: "Accumulation Trajectory"
      description: "Whether convention count is growing (accumulating), stable, or declining (shedding). Stable or shedding suggests healthy convention hygiene."
      type: enum
      sentiment: neutral
      section: system_metrics

    # Epistemic assessment metrics
    fsRiskOverall:
      label: "Failure Signature Risk (Overall)"
      description: "Aggregate risk that the analysis contains systematic blind spots or failure patterns. LOW means the agent's findings are well-grounded."
      type: enum
      sentiment: lower_is_better
      section: epistemic_assessment
    fs1PowerProjection:
      label: "Power Projection Risk"
      description: "Risk that the agent is projecting power dynamics onto structures that are better explained by simpler causes."
      type: enum
      sentiment: lower_is_better
      section: epistemic_assessment
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `label` | string | Yes | Human-readable label for dashboard display (max 100 chars) |
| `description` | string | No | Explanation of what this metric measures and why it matters (max 1000 chars) |
| `type` | enum | No | Data type: `integer`, `float`, `ratio`, `percentage`, `string`, `boolean`, `date`, `enum` |
| `sentiment` | enum | No | Directional interpretation: `higher_is_better`, `lower_is_better`, `neutral`, `target_range` |
| `range` | object | No | Expected value range with `min`, `max`, `target` (for `target_range` sentiment) |
| `section` | enum | No | Which output section: `system_metrics` (default) or `epistemic_assessment` |

**Dashboard consumption pattern:**

1. When rendering an agent's system metrics or epistemic assessment, the dashboard fetches the agent's ADL definition from the registry (cached)
2. For each metric key in the data, it looks up the key in `output.metrics`
3. If found: renders `label` as the display name, `description` as a tooltip, and uses `sentiment` for green/amber/red coloring
4. If not found (backwards compat): renders the key with snake_case → space formatting, no tooltip, neutral coloring

**Best practices:**
- Define metrics for ALL keys that appear in your agent's `system_metrics` and `epistemic_assessment` output
- Use `description` to explain what the metric means to a non-expert — the dashboard user may not know the agent's philosophical framework
- Set `sentiment` whenever the metric has a clear directional preference — this enables at-a-glance health assessment
- Use `section` to distinguish system metrics from epistemic assessment metrics — they render in different UI sections

#### Output Formats

| Format | Description | Use Case |
|--------|-------------|----------|
| `markdown` | Human-readable report | CLI output, documentation |
| `json` | Machine-parseable | API responses, CI/CD |
| `html` | Web-formatted output | Dashboards, reports |
| `structured` | SDK-compatible schema | Programmatic consumption |

#### Classification Fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `enabled` | boolean | `false` | Enable failure classification |
| `taxonomy_version` | string | — | Version of failure taxonomy |
| `include_codes` | boolean | `true` | Include failure codes in output |
| `allow_secondary` | boolean | `false` | Allow secondary failure classifications |

#### Structured Output Fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `enabled` | boolean | `false` | Enable structured output |
| `schema_ref` | string | — | JSON Schema reference for validation |
| `include_metadata` | boolean | `true` | Include execution metadata |

---

### Edge Cases Block

The `edge_cases` block handles special situations.

```yaml
edge_cases:
  - id: empty_project
    condition: "No source files found"
    behavior:
      - Report as info, not error
      - Return neutral score (N/A)
      - Suggest running after implementation

  - id: generated_code
    condition: "File path contains /generated/ or /dist/"
    condition_expression: "path.includes('/generated/') || path.includes('/dist/')"  # NEW in v1.2.0
    behavior:
      - Exclude from analysis
      - Note exclusion in report
    score_adjustment:
      exclude_categories: [code_quality]
      rescale: true
    report_wording: "Generated code excluded from analysis"  # NEW in v1.2.0
    judgment_rationale: "Generated code is not authored by developers and should not affect quality scores"  # NEW in v1.2.0

  - id: no_auth_code
    condition: "No authentication code patterns found in codebase"
    condition_expression: "grep_count('jwt|token|auth|session|password') == 0"  # NEW in v1.2.0
    behavior:
      - Check if project type requires authentication
      - Score Auth as 20/20 only if auth not required (CLI tool, static site)
      - Otherwise flag for verification
    report_wording: "No auth detected - verify if auth required"  # NEW in v1.2.0
    judgment_rationale: "Some projects legitimately don't need auth; flagging rather than penalizing avoids false negatives"  # NEW in v1.2.0
    decision_override:  # NEW in v1.2.0
      affects_decision: false
    compound_conditions:  # NEW in v1.2.0
      - also_true: minimal_codebase
        behavior_override:
          - "Small project without auth may be intentional - reduce severity"
        priority: merge
    severity: high  # NEW in v1.2.0

  - id: minimal_codebase
    condition: "Fewer than 5 source files in project"
    condition_expression: "source_file_count < 5"  # NEW in v1.2.0
    behavior:
      - Apply expedited review
    score_adjustment:
      fixed_score: 80
    report_wording: "Minimal codebase - limited audit scope"  # NEW in v1.2.0
    judgment_rationale: "Small codebases still need review but findings should be contextualized"  # NEW in v1.2.0
    severity: low  # NEW in v1.2.0
```

#### Edge Case Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | ✔ | Snake_case identifier |
| `condition` | string | ✔ | Human-readable condition description |
| `condition_expression` | string | No | Machine-parseable condition for automated detection (v1.2.0) |
| `behavior` | string[] | ✔ | Steps to take when condition is met |
| `score_adjustment` | object | No | How to adjust scoring |
| `report_wording` | string | No | Exact text to include in report (max 600 chars, v1.2.0) |
| `judgment_rationale` | string | No | WHY this handling is appropriate (max 1000 chars, v1.2.0) |
| `decision_override` | object | No | Whether/how to override score-based decision (v1.2.0) |
| `compound_conditions` | CompoundCondition[] | No | Handle "what if both X and Y?" scenarios (v1.2.0) |
| `severity` | enum | No | Importance of correct handling: `critical`, `high`, `medium`, `low`, `info` (v1.2.0) |

#### Score Adjustment Fields

| Field | Type | Description |
|-------|------|-------------|
| `exclude_categories` | string[] | Categories to exclude from scoring |
| `rescale` | boolean | Rescale remaining categories to 100 |
| `fixed_score` | number | Override with fixed score (0-100) |

#### Decision Override Fields (v1.2.0)

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `affects_decision` | boolean | `false` | Whether this edge case can change the final decision |
| `forced_decision` | enum | — | If affects_decision=true, which decision to force: `positive`, `conditional`, `negative` |
| `override_rationale` | string | — | Why this edge case justifies overriding the score-based decision |

#### Compound Condition Fields (v1.2.0)

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `also_true` | string | — | Another edge case ID that may co-occur |
| `behavior_override` | string[] | — | What to do when BOTH conditions are true |
| `priority` | enum | `merge` | Which edge case takes precedence: `this`, `other`, `merge` |

---

### Tone Block

The `tone` block defines communication style for reports.

```yaml
tone:
  attributes:
    - Direct and actionable
    - Evidence-based
    - Constructive, not punitive
    
  guidelines:
    - Lead with the most important finding
    - Include file:line references for all issues
    - Provide specific remediation steps
    - Acknowledge what's working well
    - Avoid jargon and buzzwords
```

### Forecast Block (Forecasters)

**NEW in v1.9.0**

The `forecast` block configures the prediction lens for forecaster agents. It captures the three axes that define how a forecaster examines artifacts: who interacts with it, over what time scale, and through what mechanism effects propagate.

```yaml
forecast:
  actor_type: rational        # Who interacts with the artifact
  time_horizon: near-term     # Temporal scope of predictions
  propagation_mechanism: >    # How effects propagate
    Metric gaming and threshold satisficing — rational actors
    optimize against measurable criteria to satisfy the letter
    while undermining the spirit.
  prediction_format: probability-weighted  # Output structure
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `actor_type` | enum | No | Primary actor: `rational`, `naive`, `adversarial`, `systemic`, `cultural`, `creative` |
| `time_horizon` | enum | No | Temporal scope: `near-term`, `medium-term`, `long-term` |
| `propagation_mechanism` | string | No | How effects propagate (free-form description) |
| `prediction_format` | enum | No | Output structure: `probability-weighted`, `spectrum`, `timeline`, `depth-map`, `interaction-map` |

---

## Failure Taxonomy

ADL v1.1.0 provides comprehensive failure taxonomy type definitions.

### Domain Codes

| Code | Label | Description |
|------|-------|-------------|
| `STR` | Structural | Form, format, and structure issues |
| `SEM` | Semantic | Meaning and correctness issues |
| `PRA` | Pragmatic | Purpose and effectiveness issues |
| `EPI` | Epistemic | Knowledge and certainty issues |

### Failure Mode Codes

| Domain | Code | Name | Description |
|--------|------|------|-------------|
| STR | `STR-OMI` | Omission | Missing required elements |
| STR | `STR-EXC` | Excess | Unnecessary or redundant elements |
| STR | `STR-MAL` | Malformation | Incorrectly structured elements |
| STR | `STR-INC` | Inconsistency | Elements contradict structurally |
| STR | `STR-SYN` | Syntax | Syntax or specification violation |
| STR | `STR-FMT` | Format | Formatting or layout issue |
| SEM | `SEM-INC` | Incorrectness | Factually or logically wrong |
| SEM | `SEM-COM` | Incompleteness | Partially correct, missing key aspects |
| SEM | `SEM-AMB` | Ambiguity | Multiple valid interpretations |
| SEM | `SEM-COH` | Incoherence | Internal logical contradiction |
| SEM | `SEM-TYP` | Type Error | Type system violation |
| SEM | `SEM-LOG` | Logic Error | Logical reasoning flaw |
| PRA | `PRA-ALI` | Misalignment | Doesn't serve stated purpose |
| PRA | `PRA-MAT` | Mismatch | Wrong for audience or context |
| PRA | `PRA-EFF` | Inefficiency | Achieves goal suboptimally |
| PRA | `PRA-FRA` | Fragility | Works now but breaks under change |
| PRA | `PRA-DOC` | Documentation | Missing or inadequate documentation |
| PRA | `PRA-TST` | Testing | Insufficient test coverage or quality |
| EPI | `EPI-OVR` | Overclaiming | Confidence exceeds evidence |
| EPI | `EPI-UND` | Underclaiming | Evidence exceeds expressed confidence |
| EPI | `EPI-GRN` | Ungrounded | Claims without traceable support |
| EPI | `EPI-FAL` | Unfalsifiable | No way to verify or refute |
| EPI | `EPI-VAL` | Validation | Verification method gap |
| EPI | `EPI-VER` | Unverifiable | Cannot be independently verified |

### Severity Codes

| Code | Points | Action |
|------|--------|--------|
| `critical` | auto_fail | Must fix immediately |
| `high` | -10 | Fix before merge |
| `medium` | -5 | Should fix |
| `low` | -2 | Consider fixing |
| `info` | 0 | For awareness |

### Usage in Criteria

```yaml
criteria:
  - id: naming_conventions
    name: Consistent naming conventions
    points: 10
    failure_taxonomy:
      domain: semantic           # Human-readable
      failure_mode: SEM-AMB      # Canonical code
      default_severity: medium
```

---

## Retry Configuration

**NEW in v1.1.0**

Retry configuration can be applied at operation, phase, or step level.

```yaml
retry:
  max_attempts: 3
  backoff_ms: 1000
  backoff_multiplier: 2
  retry_on:
    - "ECONNRESET"
    - "ETIMEDOUT"
    - "rate_limit"
```

### Fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `max_attempts` | number | `0` | Maximum retry attempts (0 = no retries) |
| `backoff_ms` | number | `1000` | Initial backoff delay in milliseconds |
| `backoff_multiplier` | number | `2` | Multiplier for exponential backoff |
| `retry_on` | string[] | `[]` | Error types/codes that trigger retry |

### Backoff Calculation

```
delay = backoff_ms * (backoff_multiplier ^ attempt)

Example (backoff_ms=1000, multiplier=2):
  Attempt 1: 1000ms
  Attempt 2: 2000ms
  Attempt 3: 4000ms
```

---

## Composition Rules

### What Agents Can Reference

```
Agent (ADL)
    ├── Cannot reference: Other agents, commands, workflows, pipelines
    └── Is: Atomic, self-contained unit
```

Agents are the **leaf nodes** of the composition hierarchy. All composition happens at higher levels:

| Level | Can Reference |
|-------|---------------|
| Agent | Nothing (atomic) |
| Command | Agents only |
| Workflow | Commands only |
| Pipeline | Workflows and Commands |

### Agent Type Composition Rules

Each agent type has required and forbidden blocks enforced by the JSON Schema:

| Agent Type | Required Blocks | Forbidden Blocks | Optional Blocks |
|------------|----------------|-------------------|-----------------|
| **Validator** | `scoring`, `decisions` | `tasks`, `completion`, `rollback` | All common blocks |
| **Executor** | `tasks`, `completion` | `scoring`, `decisions`, `deductions` | All common blocks |
| **Analyst** (v1.6.0) | `process` | `tasks`, `completion`, `rollback` | `scoring`, `decisions` + all common |
| **Generator** (v1.6.0) | `tasks` | `scoring`, `deductions`, `auto_fail` | `completion` + all common |
| **Explorer** (v1.8.0) | `interface`, `mission` | `scoring`, `decisions`, `deductions`, `auto_fail`, `tasks`, `completion`, `rollback` | `knowledge_base`, `process`, `edge_cases`, `output`, `tone` |
| **Forecaster** (v1.9.0) | `interface`, `mission`, `process` | `tasks`, `completion`, `rollback` | `scoring`, `decisions`, `deductions`, `auto_fail`, `forecast` + all common |

**Common blocks** available to all types: `interface`, `defaults`, `context`, `mission`, `knowledge_base`, `process`, `output`, `edge_cases`, `tone`, `handoff`, `cognitive_lens` (v1.10.0), `composition` (v1.10.0).

**Analyst rationale:** Analysts are process-driven. They investigate and assess, producing analytical reports. Some analysts produce scores (e.g., strategy assessment), so `scoring`/`decisions` are optional rather than forbidden. Task execution and rollback don't apply.

**Generator rationale:** Generators produce artifacts from inputs, so they use the task execution model. Scoring, deductions, and auto-fail are forbidden because generators create things — they don't judge them.

**Explorer rationale:** Explorers are discovery agents — they navigate, map, and report. They need only a mission and optional tool guidance. All structured output machinery (scoring, decisions, tasks, completion) is forbidden because it adds token overhead without improving exploration quality. This is the lightest agent type, optimized for tool-augmented codebase and system exploration.

**Forecaster rationale:** Forecasters predict future states and causal consequences. They require `process` (analysis phases for tracing causal chains) and `mission` (prediction framing). Like analysts, forecasters may optionally include scoring for assessment-style prediction (e.g., scoring perverse outcome severity). The optional `forecast` section captures the prediction lens (actor type, time horizon, propagation mechanism). `decisions` is optional for custom prediction vocabulary (e.g., SOUND/PERVERSE).

### Multi-Agent Composition

When Commands wrap multiple agents:

| Composition | Valid? | Aggregation |
|-------------|--------|-------------|
| Validators only | ✅ | Score aggregation (avg, min, max) |
| Executors only | ✅ | Sequential execution |
| Analysts only | ✅ | Report aggregation |
| Generators only | ✅ | Sequential generation |
| Explorers only | ✅ | Report aggregation |
| Forecasters only | ✅ | Prediction synthesis |
| Validators → Executors | ✅ | Pipeline (validate then fix) |
| Executors → Validators | ✅ | Pipeline (do then verify) |
| Analysts → Generators | ✅ | Pipeline (analyze then generate) |
| Explorers → Analysts | ✅ | Pipeline (discover then assess) |
| Explorers → Generators | ✅ | Pipeline (discover then produce) |
| Forecasters → Analysts | ✅ | Pipeline (predict then assess) |
| Forecasters → Forecasters | ✅ | Pipeline (cascade prediction) |
| Mixed parallel | ❌ | Must be sequential |

---

## Runtime Rendering

The Registry API renders ADL definitions to runtime prompts. This is a **pure function** with no side effects:

```
ADL YAML → Render Function → Runtime Prompt + Tools + Schema
```

### Render Output Structure

```typescript
interface AgentRuntime {
  // Rendered system prompt
  prompt: string;
  
  // Tools available to the agent
  tools: Tool[];
  
  // Expected output schema
  outputSchema: JSONSchema;
  
  // Metadata for tracking
  metadata: {
    name: string;
    version: string;
    agentType: 'validator' | 'executor' | 'analyst' | 'generator' | 'explorer';
    domain: string;
    hash: string;  // Content-addressed hash
  };
}
```

### Render Determinism

To ensure reproducibility:

1. **No timestamps** in rendered output
2. **No random values** (UUIDs, etc.)
3. **Sorted object keys** for consistent hashing
4. **Version-locked dependencies** resolved before render

Test: `hash(render(yaml)) === hash(render(yaml))` must always pass.

---

## Migration from VDL

### Automated Migration

Most VDL files can be migrated with simple transformations:

```bash
# Migration command
ulu migrate vdl-to-adl ./validators/*.yaml
```

### Manual Changes Required

| VDL (v1.1.0) | ADL (v1.1.0) | Notes |
|--------------|--------------|-------|
| Root: `validator:` | Root: `agent:` | Structural change |
| `metadata:` | `interface:` | Renamed block |
| — | `interface.agentType: validator` | New required field |
| — | `interface.displayName` | New required field |
| `metadata.tools` | `interface.tools` | Moved to interface |
| `metadata.model` | `defaults.model` | Moved to defaults |
| `metadata.risk_level` | `interface.risk_level` | Moved to interface |
| `scoring.total_points: 100` | `scoring.maxScore: 100` | Renamed field |
| `scoring.categories[].points` | `scoring.categories[].weight` | Renamed field |
| All other fields | Same | No change |

### Example Migration

**Before (VDL v1.1.0):**

```yaml
validator:
  metadata:
    name: code-validator
    version: "1.2.0"
    description: Validates code quality...
    domain: software
    risk_level: standard
    tools: [Read, Grep, Glob, Bash]
    model: sonnet
    
  scoring:
    total_points: 100
    categories:
      - id: quality
        name: Code Quality
        points: 30
        # ...
```

**After (ADL v1.1.0):**

```yaml
agent:
  interface:
    name: code-validator
    version: "1.2.0"
    displayName: Code Validator           # NEW
    description: Validates code quality...
    agentType: validator                   # NEW
    domain: software
    risk_level: standard                   # MOVED from metadata
    tools: [Read, Grep, Glob, Bash]       # MOVED from metadata
    
  defaults:                                # RENAMED from metadata
    model: sonnet
    timeout: 300000
    
  scoring:
    maxScore: 100                          # RENAMED from total_points
    categories:
      - id: quality
        name: Code Quality
        weight: 30                         # RENAMED from points
        # ...
```

---

## Migration from ADL v1.9.0

### What Changed

1. **New domain enum value: `cognitive-lens`** — for agents encoding epistemic frameworks (thinkers' analytical methods)
2. **New optional block: `cognitive_lens`** — encodes thinker identity, epistemic depth, core axioms, and failure signatures
3. **New optional block: `composition`** — describes agent pairing relationships, blind spot coverage, and composition patterns
4. **Formalized block: `handoff`** — upstream/downstream data flow contracts (previously used but undocumented)

### Breaking Changes

None. All new fields are optional. All existing v1.9.0 definitions are valid v1.10.0 definitions.

### Migration Steps

1. **No changes required** for existing v1.9.0 definitions
2. **To add cognitive lens metadata**: Add `cognitive_lens` block with `thinker` (required) plus optional `epistemic_depth`, `core_axioms`, and `failure_signatures`
3. **To add composition guidance**: Add `composition` block with optional `pairs_well_with`, `covers_blind_spots_of`, and `has_blind_spots_covered_by`
4. **To change domain**: Set `domain: cognitive-lens` for epistemic framework agents (optional — `general` with `subdomain` also works)

### New Optional Fields

| Section | Field | Purpose |
|---------|-------|---------|
| `interface.domain` | `"cognitive-lens"` | New domain value for epistemic framework agents |
| `cognitive_lens.thinker` | `string` | Kebab-case thinker identifier (e.g., `aristotle`, `popper`) |
| `cognitive_lens.epistemic_depth.primary` | `enum` | Default depth: `first-order`, `second-order`, or `third-order` |
| `cognitive_lens.epistemic_depth.capable` | `array` | All depths this agent can operate at |
| `cognitive_lens.epistemic_depth.target_description` | `string` | What the agent targets at its primary depth |
| `cognitive_lens.core_axioms[].axiom` | `string` | Statement of a foundational assumption |
| `cognitive_lens.core_axioms[].implications` | `array` | Practical consequences of the axiom |
| `cognitive_lens.failure_signatures[].signature` | `string` | Name of a lens-specific failure mode |
| `cognitive_lens.failure_signatures[].description` | `string` | Why this failure occurs with this lens |
| `cognitive_lens.failure_signatures[].mitigation` | `string` | How to mitigate, typically via pairing |
| `composition.pairs_well_with[]` | `object` | Agent, reason, pattern, depth |
| `composition.covers_blind_spots_of[]` | `object` | Agent, blind_spot, how |
| `composition.has_blind_spots_covered_by[]` | `object` | Agent, blind_spot, how |
| `handoff.upstream_context` | `object` | Input contract, accepts list, and description |
| `handoff.upstream_context.input_contract` | `string` | Explicit contract for input requirements — format, cardinality, error behavior |
| `handoff.downstream_artifacts` | `object` | Produces list and description |

### Example: Cognitive Lens Agent

```yaml
agent:
  interface:
    name: aristotle-analyst
    version: "1.0.0"
    displayName: Aristotle Analyst
    description: >-
      Aristotelian four-cause analytical decomposition agent.
      Applies teleological framework to analyze artifacts.
    agentType: analyst
    domain: cognitive-lens        # NEW in v1.10.0
    subdomain: aristotelian-analysis
    tools: [Read, Grep, Glob, Bash]
    tags: [aristotle, four-causes, teleological, cognitive-lens]

  cognitive_lens:                 # NEW in v1.10.0
    thinker: "aristotle"
    epistemic_depth:
      primary: "first-order"
      capable: ["first-order", "second-order"]
      target_description: "Domain entities, systems, and phenomena"
    core_axioms:
      - axiom: "Everything has a telos — a natural end or purpose"
        implications:
          - "Understanding something means understanding what it's for"
          - "Dysfunction is deviation from natural function"
      - axiom: "Things have essential and accidental properties"
        implications:
          - "Analysis must distinguish what something necessarily is from what it happens to be"
    failure_signatures:
      - signature: "Teleological projection onto purposeless systems"
        description: "Not everything has a telos. Projecting purpose onto random or mechanical processes produces pseudoexplanation."
        mitigation: "Pair with Humean or Darwinian lens"
      - signature: "Essentialism in fluid domains"
        description: "Some domains resist essential/accidental distinction"
        mitigation: "Pair with process philosophy or constructivist lens"

  composition:                    # NEW in v1.10.0
    pairs_well_with:
      - agent: "hume-validator"
        reason: "Challenges teleological assumptions with empirical skepticism"
        pattern: "adversarial_dialectic"
      - agent: "darwin-analyst"
        reason: "Provides non-teleological alternative for apparent purpose"
        pattern: "parallel_reading"
    covers_blind_spots_of:
      - agent: "hume-analyst"
        blind_spot: "structural_explanation"
        how: "Formal and final causes provide explanatory structure Humean regularity analysis lacks"
    has_blind_spots_covered_by:
      - agent: "darwin-analyst"
        blind_spot: "unwarranted_teleology"
        how: "Natural selection explains apparent purpose without actual purpose"

  # Standard ADL fields continue below (mission, process, scoring, etc.)
```

### Epistemic Depth Explained

The `epistemic_depth` field encodes the order of analysis:

| Depth | Target | Example |
|-------|--------|---------|
| `first-order` | Domain artifacts directly | Aristotle analyst decomposes a system architecture |
| `second-order` | Reasoning about artifacts | Aristotle analyst decomposes *the analysis itself* |
| `third-order` | Framework that produced reasoning | Aristotle analyst examines *the Aristotelian method* |

### Composition Patterns

| Pattern | Description |
|---------|-------------|
| `adversarial_dialectic` | Agents challenge each other's assumptions |
| `parallel_reading` | Agents analyze the same target independently for comparison |
| `sequential_pipeline` | Output of one feeds into the next |
| `complementary_coverage` | Each covers the other's blind spots |

### Validation

```bash
# Validate against v1.10.0 schema
udl validate my-cognitive-lens.agent.yaml
```

---

## Migration from ADL v1.8.0

### What Changed

1. **New agent type: `forecaster`** — prediction-oriented agents that model future states and causal consequences
2. **New optional block: `forecast`** — configures the prediction lens (actor type, time horizon, propagation mechanism)
3. **`agentType` enum** now includes `"forecaster"` as a sixth value

### Migration Steps

1. **No changes required** for existing v1.8.0 definitions — all v1.8.0 definitions are valid v1.9.0 definitions
2. **To reclassify an analyst as forecaster**: Change `agentType: analyst` to `agentType: forecaster`, add `process` block if not present, optionally add `forecast` block
3. **Bump version** to major (e.g., 1.x.0 → 2.0.0) when changing agentType since it's a breaking classification change

---

## Migration from ADL v1.7.0

v1.8.0 is fully backward-compatible with v1.7.0 — all existing agent types continue to work unchanged. The only addition is the new `explorer` agent type.

### New Agent Type: Explorer

To create an explorer agent, set `agentType: explorer` in the interface block. Explorers are the lightest agent type:

```yaml
agent:
  interface:
    name: my-explorer
    version: "1.0.0"
    displayName: My Explorer
    description: Explores and maps a target system
    agentType: explorer    # NEW in v1.8.0
    domain: software
    tools: [Read, Grep, Glob, Bash]

  defaults:
    model: sonnet

  mission:
    opener: "You are exploring the target to discover its structure."
    # ... mission fields

  # Optional: knowledge_base, process, edge_cases, output, tone
  # Forbidden: scoring, decisions, deductions, auto_fail, tasks, completion, rollback
```

### Key Differences from Analyst

Explorers and analysts may seem similar but have different constraints:

| Aspect | Analyst | Explorer |
|--------|---------|----------|
| `process` | Required | Optional |
| `scoring` | Optional | Forbidden |
| `decisions` | Optional | Forbidden |
| `auto_fail` | Optional | Forbidden |
| Output | Structured analysis | Narrative report |
| Prompt overhead | ~250+ lines | ~100 lines |

### Validation

```bash
# Validate against v1.8.0 schema
udl validate my-explorer.agent.yaml
```

---

## Migration from ADL v1.6.0

v1.7.0 is fully backward-compatible with v1.6.0 — all new fields are optional. To take advantage of new features:

### New Optional Fields

| Section | Field | Purpose |
|---------|-------|---------|
| `mission` | `epistemic_limitations` | Acknowledge what the agent cannot assess |
| `knowledge_base` | `key_definitions` | Glossary of terms used in analysis |
| `knowledge_base` | `domain_taxonomy` | Domain-specific classification with categories and scale |
| `knowledge_base.sections[]` | `description` | Description of what each section covers |
| `scoring` | `interpretation` | Guide for understanding score ranges |
| `scoring` | `weight_rationale` | Explain why categories are weighted as they are |
| `decisions` | `decision_guidance` | Guidance for edge cases near boundaries |
| `process.reasoning_scaffolding` | `passes` | Multi-pass methodology (alternative to `steps`) |
| `process.reasoning_scaffolding` | `pass_attribution_requirement` | Require findings to attribute their discovery pass |
| `output` | `templates` | Named reusable output templates |

### Validation

```bash
# Validate against v1.7.0 schema
adl-factory validate my-agent.agent.yaml

# Upgrade recommendations
adl-factory validate my-agent.agent.yaml --suggest-upgrades
```

### Most Impactful for Analyst Agents

The v1.7.0 features are particularly valuable for analyst agents:
- `domain_taxonomy` structures domain-specific classification
- `passes` in reasoning scaffolding supports multi-pass methodologies
- `epistemic_limitations` improves output quality through epistemic humility
- `key_definitions` ensures consistent vocabulary
- `templates` provides reusable output structure

---

## Migration from ADL v1.5.1

ADL v1.7.0 is fully backward compatible with v1.5.1. No changes are required. Existing `validator` and `executor` agents continue to work without modification.

**New capabilities since v1.6.0:**
- Add `domain_profile` in interface to reference a domain profile for domain-specific rendering (e.g., legal, medical)
- Use `agentType: analyst` for process-driven analysis agents that investigate and produce reports
- Use `agentType: generator` for task-driven artifact production agents

**Domain profiles:**
- Add `domain_profile: legal` (or other profile name) to interface when building non-software agents
- Profiles provide domain-specific severity descriptions, issue types, edge cases, and terminology
- Create new profiles at `udl/definition-languages/profiles/{name}.profile.yaml`
- Validated against `domain-profile-schema-v1.0.0.json`

**Validation:**

```bash
# Validate against v1.7.0 schema
ulu validate ./my-agent.yaml --schema adl@1.6.0

# Upgrade recommendations
ulu upgrade ./my-agent.yaml --to 1.6.0 --suggest
```

---

## Migration from ADL v1.4.0

ADL v1.7.0 is fully backward compatible with v1.4.0. No changes are required.

**Optional enhancements available in v1.5.0:**
- Add `vocabulary_rationale` in mission to explain decision vocabulary choice
- Add `explicit_prohibitions` in mission for hard agent boundaries
- Add `success_criteria` in decisions for explicit conditions beyond score threshold

**Optional enhancements available in v1.5.1:**
- Add `data_sources` in context to document external APIs/MCP tools
- Add `formula` and `thresholds` in verification for mathematical criteria

**Optional enhancements available since v1.5.0+:**
- Add `domain_profile` in interface for domain-specific rendering
- Use `agentType: analyst` for process-driven analysis agents
- Use `agentType: generator` for task-driven artifact production agents

**Validation:**

```bash
# Validate against v1.7.0 schema
ulu validate ./my-agent.yaml --schema adl@1.6.0

# Upgrade recommendations
ulu upgrade ./my-agent.yaml --to 1.6.0 --suggest
```

---

## Migration from ADL v1.2.0/v1.3.0

ADL v1.7.0 is fully backward compatible with v1.2.0 and v1.3.0. No changes are required.

**Optional enhancements available in v1.3.0:**
- Add `calibration_examples` in scoring for consistent LLM scoring
- Add `failure_code_examples` in knowledge_base for taxonomy classification
- Add `token_budget` in output for length guidance
- Add `section_order` in output for explicit ordering
- Add `display_id` on auto-fail conditions

**Optional enhancements available in v1.4.0:**
- Add `reasoning_scaffolding` in process for structured LLM reasoning
- Add `pre_decision_checklist` in process for completeness verification
- Add `examples` in output for format demonstration

**Optional enhancements available in v1.5.0:**
- Add `vocabulary_rationale` in mission to explain decision vocabulary choice
- Add `explicit_prohibitions` in mission for hard agent boundaries
- Add `success_criteria` in decisions for explicit conditions beyond score threshold

**Optional enhancements available in v1.5.1:**
- Add `data_sources` in context to document external APIs/MCP tools
- Add `formula` and `thresholds` in verification for mathematical criteria

**Optional enhancements available since v1.5.0+:**
- Add `domain_profile` in interface for domain-specific rendering
- Use `agentType: analyst` for process-driven analysis agents
- Use `agentType: generator` for task-driven artifact production agents

**Validation:**

```bash
# Validate against v1.7.0 schema
ulu validate ./my-agent.yaml --schema adl@1.6.0

# Upgrade recommendations
ulu upgrade ./my-agent.yaml --to 1.6.0 --suggest
```

---

## Migration from ADL v1.1.0

ADL v1.2.0+ is fully backward compatible with v1.1.0. No changes are required.

**Optional enhancements available in v1.2.0:**
- Add `mission` block for agent identity and purpose framing
- Add `knowledge_base` block for embedded domain expertise
- Enhance `verification` with `verify_guidance`, `partial_scoring`, and `definition_clarifications`
- Enhance `edge_cases` with `report_wording`, `judgment_rationale`, `decision_override`, `compound_conditions`, and `severity`

**Validation:**

```bash
# Validate against v1.7.0 schema
ulu validate ./my-agent.yaml --schema adl@1.6.0

# Upgrade recommendations
ulu upgrade ./my-agent.yaml --to 1.6.0 --suggest
```

---

## Examples

### Complete Validator Example

```yaml
# code-validator.agent.yaml
agent:
  interface:
    name: code-validator
    version: "1.2.0"
    displayName: Code Validator
    description: >
      Validates code quality, maintainability, and best practices.
      Run after implementation to catch issues before review.
      Blocks deployment if critical issues are found.
    agentType: validator
    domain: software
    subdomain: quality
    risk_level: standard
    tools: [Read, Grep, Glob, Bash]
    tags: [typescript, javascript, quality]
    triggers:
      file_patterns:
        - "src/**/*.ts"
        - "src/**/*.js"
      explicit_only: false
    dependencies:
      recommends:
        - pre-implementation-architect
    epistemic_nature:                      # NEW in v1.13.0
      verifiability: mechanically_checkable
      determinism: stochastic
      claim_type: factual

  defaults:
    model: sonnet
    timeout: 300000

  context:
    working_directory: "."
    timeout_behavior: fail

  scoring:
    maxScore: 100
    
    categories:
      - id: code_quality
        name: Code Quality
        weight: 30
        criteria:
          - id: naming
            name: Consistent naming conventions
            points: 10
            failure_taxonomy:
              domain: semantic
              failure_mode: SEM-AMB
              default_severity: medium
          - id: duplication
            name: No code duplication
            points: 10
          - id: complexity
            name: Manageable complexity
            points: 10
            
      - id: error_handling
        name: Error Handling
        weight: 25
        criteria:
          - id: try_catch
            name: Proper try-catch usage
            points: 10
          - id: error_types
            name: Typed error classes
            points: 8
          - id: logging
            name: Error logging with context
            points: 7
            
      - id: testing
        name: Testing
        weight: 25
        criteria:
          - id: coverage
            name: Adequate test coverage
            points: 10
          - id: assertions
            name: Meaningful assertions
            points: 8
          - id: edge_cases
            name: Edge case coverage
            points: 7
            
      - id: documentation
        name: Documentation
        weight: 20
        criteria:
          - id: jsdoc
            name: JSDoc on public APIs
            points: 10
          - id: readme
            name: Updated README
            points: 10
            
  decisions:
    vocabulary:
      positive: PASS
      negative: FAIL
    preset: quality_gate
    tracking:
      category: gate
      notify_on:
        - negative
    
  auto_fail:
    enabled: true
    conditions:
      - id: build_fails
        name: Build fails
        severity: critical
        detection:
          method: tool
          command: "npm run build"
          failure_condition: "exit_code != 0"
          
  output:
    format: structured
    schema: validator-output-v1.1
    classification:
      enabled: true
      taxonomy_version: "0.2.2"
      allow_secondary: true
```

### Complete Executor Example

```yaml
# code-fixer.agent.yaml
agent:
  interface:
    name: code-fixer
    version: "1.0.0"
    displayName: Code Fixer
    description: >
      Automatically fixes code issues identified by validators.
      Applies safe transformations for common problems.
      Requires clean git state before execution.
    agentType: executor
    domain: software
    subdomain: automation
    risk_level: standard
    tools: [Read, Grep, Bash]
    tags: [typescript, javascript, refactoring]
    epistemic_nature:                      # NEW in v1.13.0
      verifiability: mechanically_checkable
      determinism: stochastic
      claim_type: factual

  defaults:
    model: opus  # Higher capability for code generation
    timeout: 600000

  context:
    working_directory: "."
    timeout_behavior: fail
    shell: bash

  tasks:
    inputs:
      - id: issues
        name: Issues to Fix
        type: array
        itemType: ValidationIssue
        required: true
        source: file
        description: Issues from validation run
        validation:
          min: 1
        
      - id: scope
        name: Fix Scope
        type: enum
        values: [single, category, all]
        default: single
        
      - id: dry_run
        name: Dry Run
        type: boolean
        default: false
        
    operations:
      - id: analyze
        name: Analyze Fixability
        description: Determine which issues can be auto-fixed
        depends_on: []
        
      - id: generate
        name: Generate Patches
        description: Create fix patches for each fixable issue
        depends_on: [analyze]
        retry:
          max_attempts: 2
          backoff_ms: 1000
        
      - id: apply
        name: Apply Patches
        description: Apply fixes to source files
        depends_on: [generate]
        condition: "!inputs.dry_run"
        rollback_on_failure: true
        
      - id: format
        name: Format Code
        description: Run prettier on modified files
        depends_on: [apply]
        
      - id: verify
        name: Verify Fixes
        description: Quick validation of fixed code
        depends_on: [format]
        
    outputs:
      - id: applied
        name: Fixes Applied
        type: array
        itemType: AppliedFix
        required: true
        
      - id: skipped
        name: Fixes Skipped
        type: array
        itemType: SkippedFix
        required: false
        
      - id: diff
        name: Change Diff
        type: string
        format: unified-diff
        
  completion:
    vocabulary:
      complete: COMPLETE
      partial: PARTIAL
      failed: FAILED
    criteria:
      - condition: "outputs.applied.length == inputs.issues.length"
        decision: complete
      - condition: "outputs.applied.length > 0"
        decision: partial
      - condition: "outputs.applied.length == 0"
        decision: failed
        
  rollback:
    enabled: true
    strategy: git_restore
    preserve_logs: true
    notify_on_rollback: true
    
  output:
    format: structured
    schema: executor-output-v1.0
```

### Complete Analyst Example (v1.7.0)

```yaml
# assumption-excavator.agent.yaml (abridged)
agent:
  interface:
    name: assumption-excavator
    version: "1.1.0"
    displayName: Assumption Excavator
    description: >
      Surfaces implicit assumptions buried in any artifact. Produces a ranked
      assumption inventory with fragility scores.
    agentType: analyst
    domain: general
    tools: [Read, Grep, Glob]
    epistemic_nature:                      # NEW in v1.13.0
      verifiability: not_checkable
      claim_type: observational

  defaults:
    model: opus
    timeout: 300000

  mission:
    opener: "You are an epistemic analyst specializing in assumption archaeology."
    stakes: >
      Every artifact carries hidden assumptions into production. Surface them
      now, before they surface themselves.
    outcome_framing: "Produce an EXAMINED/UNEXAMINED decision with a ranked assumption inventory."
    vocabulary_rationale: >
      Uses EXAMINED/UNEXAMINED rather than PASS/FAIL because assumptions are
      not wrong by nature — the question is whether they have been surfaced.
    explicit_prohibitions:
      - "Do NOT evaluate whether the artifact achieves its stated goal"
      - "Do NOT rewrite or improve the artifact"
      - "Do NOT flag stated assumptions — only implicit, buried ones"
    # NEW in v1.7.0
    epistemic_limitations:
      - >
        You infer assumptions from text, not from the author's mental state.
        Frame findings as 'the text assumes X' rather than 'the author didn't realize X.'

  knowledge_base:
    # NEW in v1.7.0
    key_definitions:
      artifact: >
        Any document, configuration, specification, or structured output that
        encodes decisions and carries implicit assumptions.
    # NEW in v1.7.0
    domain_taxonomy:
      completeness_note: "These five categories are a starting framework."
      categories:
        - id: ENV
          name: Environmental
          description: "Assumptions about the execution environment"
        - id: DEP
          name: Dependency
          description: "Assumptions about external dependencies"
        - id: BEH
          name: Behavioral
          description: "Assumptions about how actors behave"
      scale:
        name: Assumption Fragility Scale
        description: "How badly would things break if violated?"
        levels:
          - score: "5"
            label: Catastrophic
            meaning: "System-level failure, unrecoverable"
          - score: "1"
            label: Trivial
            meaning: "Minor inconvenience, easy workaround"

    sections:
      - category_ref: environmental
        common_mistakes:
          - mistake: "Assuming the execution environment is stable"
            why_wrong: "APIs change, models update, infrastructure drifts"
            correct_approach: "Identify where the artifact would silently break"

  scoring:
    maxScore: 100
    # NEW in v1.7.0
    interpretation: >
      Scores above 70 indicate the assumption profile has been adequately surfaced.
    weight_rationale: >
      Equal weighting prevents systematic bias toward any single category.
    categories:
      - id: environmental
        name: Environmental Assumptions
        weight: 20
        criteria:
          - id: env_surface
            name: Environmental assumptions surfaced
            points: 10
          - id: env_fragility
            name: Environmental fragility assessed
            points: 10
      - id: dependency
        name: Dependency Assumptions
        weight: 20
        criteria:
          - id: dep_surface
            name: Dependency assumptions surfaced
            points: 10
          - id: dep_fragility
            name: Dependency fragility assessed
            points: 10
      - id: behavioral
        name: Behavioral Assumptions
        weight: 20
        criteria:
          - id: beh_surface
            name: Behavioral assumptions surfaced
            points: 10
          - id: beh_fragility
            name: Behavioral fragility assessed
            points: 10
      - id: temporal
        name: Temporal Assumptions
        weight: 20
        criteria:
          - id: tmp_surface
            name: Temporal assumptions surfaced
            points: 10
          - id: tmp_fragility
            name: Temporal fragility assessed
            points: 10
      - id: scale
        name: Scale & Scope Assumptions
        weight: 20
        criteria:
          - id: scl_surface
            name: Scale assumptions surfaced
            points: 10
          - id: scl_fragility
            name: Scale fragility assessed
            points: 10

  decisions:
    vocabulary:
      positive: EXAMINED
      negative: UNEXAMINED
    thresholds:
      - decision: positive
        min_score: 70
      - decision: negative
        max_score: 69
    # NEW in v1.7.0
    decision_guidance: >
      Scores near 70 require judgment. If critical assumptions were surfaced
      but categorization is incomplete, lean toward EXAMINED.

  process:
    # Multi-pass reasoning (v1.7.0)
    reasoning_scaffolding:
      description: "Work through three sequential passes."
      passes:
        - id: structural
          name: Structural Pass
          question: "What does the text take for granted about its own structure?"
          focus:
            - "Completeness assumptions"
            - "Ordering assumptions"
          method: "Read each section and ask: what is NOT here?"
          output: "List of structural assumptions"
        - id: semantic
          name: Semantic Pass
          question: "What does the text take for granted about meaning?"
          focus: "Vocabulary, domain knowledge, shared context"
          output: "List of semantic assumptions"
        - id: epistemic
          name: Epistemic Pass
          question: "What confidence claims are embedded without evidence?"
          focus: "Certainty levels, scope claims, causal claims"
          output: "List of epistemic assumptions"
      pass_attribution_requirement: >
        Each assumption MUST list which pass discovered it.
    pre_decision_checklist:
      - "All five categories scored"
      - "Every assumption has a fragility score"
      - "Pass attribution present for all findings"

  output:
    format: structured
    token_budget:
      target: 3000
      max: 6000
    # NEW in v1.7.0
    templates:
      header: |
        # ASSUMPTION EXCAVATOR
        **Target:** {target}
        **Score:** {score}/100
        **Decision:** {decision}
      assumption_entry: |
        ### A{n}: {title}
        **Category:** {category} | **Fragility:** {fragility}/5
        **Evidence:** {evidence}
    examples:
      - scenario: "Agent definition with 11 buried assumptions"
        input_summary: "prompt-engineer.agent.yaml (1100 lines)"
        output: |
          # ASSUMPTION EXCAVATOR
          Target: prompt-engineer.agent.yaml
          Score: 82/100
          Decision: EXAMINED
          ...
```

### Complete Generator Example (v1.6.0)

```yaml
# agent-scaffolder.agent.yaml
agent:
  interface:
    name: agent-scaffolder
    version: "1.0.0"
    displayName: Agent Scaffolder
    description: >
      Generates new agent prompt files from requirements. Produces
      agent markdown, ADL schema, and command file following project
      conventions.
    agentType: generator
    domain: software
    subdomain: automation
    tools: [Read, Grep, Glob, Bash]

  defaults:
    model: opus
    timeout: 600000

  mission:
    opener: "You are scaffolding a new AI agent following project conventions."
    stakes: "A well-scaffolded agent saves hours of manual setup and ensures consistency."
    outcome_framing: "Produce complete, validated agent files ready for deployment."

  # Generators require tasks
  tasks:
    inputs:
      - id: requirements
        name: Agent Requirements
        type: string
        required: true
        description: Natural language description of what the agent should do

      - id: agent_type
        name: Agent Type
        type: enum
        values: [validator, executor, analyst, generator, explorer]
        default: validator

      - id: domain
        name: Domain
        type: enum
        values: [software, legal, medical, financial, general]
        default: software

    operations:
      - id: analyze_requirements
        name: Analyze Requirements
        description: Parse requirements into structured agent specification
        depends_on: []

      - id: generate_adl
        name: Generate ADL Schema
        description: Create the ADL YAML definition
        depends_on: [analyze_requirements]

      - id: generate_prompt
        name: Generate Agent Prompt
        description: Create the agent markdown file
        depends_on: [generate_adl]

      - id: generate_command
        name: Generate Command File
        description: Create the invocation command
        depends_on: [generate_prompt]

    outputs:
      - id: adl_file
        name: ADL Schema
        type: file
        format: yaml
        required: true

      - id: prompt_file
        name: Agent Prompt
        type: file
        format: markdown
        required: true

      - id: command_file
        name: Command File
        type: file
        format: markdown
        required: true

  # Optional completion self-check
  completion:
    vocabulary:
      complete: GENERATED
      partial: PARTIAL
      failed: FAILED
    criteria:
      - condition: "All 3 output files created successfully"
        decision: complete
      - condition: "At least ADL file created"
        decision: partial
      - condition: "No files created"
        decision: failed

  # Generators cannot have: scoring, deductions, auto_fail

  output:
    format: structured
    schema: generator-output-v1.0
```

### Complete Explorer Example (v1.8.0)

```yaml
# deep-explore.agent.yaml
agent:
  interface:
    name: deep-explore
    version: "2.0.0"
    displayName: Deep Codebase Explorer
    description: >
      Deep codebase exploration using semantic search and call graph
      tracing. Maps architecture, traces dependencies, and reports
      findings as narrative exploration reports.
    agentType: explorer
    domain: software
    subdomain: exploration
    tools: [Read, Grep, Glob, Bash]

  defaults:
    model: sonnet
    timeout: 300000

  mission:
    opener: >
      You are a codebase explorer. Your job is to discover, trace,
      and map the target so thoroughly that someone reading your
      report can navigate confidently without opening a single file.
    stakes: >
      Incomplete exploration leads to wrong assumptions, missed
      dependencies, and architectural blind spots that compound
      throughout a project.
    outcome_framing: >
      Produce a comprehensive exploration report covering
      architecture, key patterns, dependencies, and notable findings.
    role_boundaries:
      - Explore and report only — do not modify any files
      - Focus on structure and relationships, not style opinions
    explicit_prohibitions:
      - Never modify, create, or delete any files
      - Never execute tests or builds

  knowledge_base:
    sections:
      - id: semantic_search
        title: Semantic Search
        content: |
          Use grepai for semantic code search:
          grepai search "<query>" --toon --compact

      - id: call_tracing
        title: Call Graph Tracing
        content: |
          Trace function relationships:
          grepai trace callers "<Function>" --toon
          grepai trace callees "<Function>" --toon
          grepai trace graph "<Function>" --toon

      - id: exploration_strategy
        title: Exploration Strategy
        content: |
          Start broad (directory structure, entry points), then
          narrow to specific subsystems. Trace call graphs to
          understand data flow. Cross-reference with tests to
          validate understanding.

  process:
    phases:
      - id: discover
        name: Discovery
        description: Map top-level structure and identify entry points
        commands:
          - "List directory tree"
          - "Identify package.json, config files, entry points"

      - id: trace
        name: Trace
        description: Follow key paths through the codebase
        commands:
          - "Trace call graphs from entry points"
          - "Map module dependencies"

      - id: examine
        name: Examine
        description: Read critical files and understand patterns
        commands:
          - "Read key implementation files"
          - "Identify design patterns and conventions"

      - id: synthesize
        name: Synthesize
        description: Compile findings into coherent report
        commands:
          - "Organize discoveries by subsystem"
          - "Note architectural decisions and trade-offs"

  edge_cases:
    - id: monorepo
      scenario: Target is a monorepo with multiple packages
      guidance: Explore each package's entry point separately

    - id: no_grepai
      scenario: grepai is not available or not indexed
      guidance: Fall back to Grep and Glob for code search

    - id: large_codebase
      scenario: Codebase is too large to explore exhaustively
      guidance: Prioritize entry points and public API surfaces

  # Explorers cannot have: scoring, decisions, deductions, auto_fail,
  #   tasks, completion, rollback
```

---

## JSON Schema Reference

The complete JSON Schema is available at:

- **URL:** `https://uluops.ai/schemas/adl/v1.8.0/agent.json`
- **File:** `adl-schema-v1.8.0.json`

### Schema Validation

```bash
# Validate an agent definition
ulu validate ./my-agent.agent.yaml

# Validate against specific schema version
ulu validate ./my-agent.agent.yaml --schema adl@1.7.0
```

---

## Revision History

| Version | Date | Changes |
|---------|------|---------|
| 1.16.0 | 2026-05-10 | Tightened: `forecast` section now forbidden on non-forecaster agent types (validator, executor, analyst, generator, explorer) via `"forecast": false` in allOf rules. Previously, `forecast` was schema-permitted for all types but only rendered by the forecaster template, causing silent absorption where the field passed validation but vanished from output. Discovered by Bateson pipeline (FD-1). Non-breaking: no existing definitions use `forecast` on non-forecaster types. |
| 1.15.0 | 2026-04-07 | Relaxed: maxLength constraints on 25 narrative fields to support richer cognitive lens agent definitions. Key bumps: `opener` 800→2000, `stakes` 400→1000, `outcome_framing` 300→1000, `vocabulary_rationale` 800→2000, `axiom` 300→800, list items (`role_boundaries`, `explicit_prohibitions`, `criteria`, `implications`) 200→800, `epistemic_limitations` items 1000→1500, `interface.description` 500→1000, `decision_guidance` 1500→2000, `interpretation`/`weight_rationale` 2000→3000, plus 11 additional narrative fields. Motivated by cognitive lens agents requiring richer philosophical framing that was being truncated by schema constraints. With 200K-1M token context windows, these fields are negligible in overall prompt size. No structural or behavioral changes — this release only relaxes string length limits. |
| 1.14.0 | 2026-04-03 | Added: `input_contract` optional string field in `handoff.upstream_context` for explicit input requirements — format, cardinality, and error behavior when input is missing or ambiguous. Distinct from `description` (which describes what works well) and `accepts` (which lists artifact types). `input_contract` specifies what the agent *requires* to function. |
| 1.13.0 | 2026-03-24 | Added: `epistemic_nature` object in interface block for multi-axis epistemic classification of agents. Three independent optional axes: `verifiability` (mechanically_checkable, expert_judgment, not_checkable), `determinism` (deterministic, stochastic, environment_dependent), `claim_type` (factual, normative, observational). Supports empirical analysis of agent effectiveness and helps consumers understand what kind of claims an agent makes. Added `epistemicNature` to schema $defs. |
| 1.12.0 | 2026-03-15 | Added: `metrics` vocabulary in output block for self-documenting system_metrics and epistemic_assessment keys. Each metric definition includes `label` (required), `description`, `type` (integer/float/ratio/percentage/string/boolean/date/enum), `sentiment` (higher_is_better/lower_is_better/neutral/target_range), `range` (min/max/target), and `section` (system_metrics/epistemic_assessment). Enables dashboards to render human-readable metric labels, descriptions as tooltips, and directional sentiment coloring without hardcoded agent-specific knowledge. Added `metricDefinition` to schema $defs. |
| 1.11.0 | 2026-03-14 | Re-versioned from v1.10.0 with minor description updates. |
| 1.10.0 | 2026-03-02 | Added: `cognitive-lens` domain enum value for epistemic framework agents. Added `cognitive_lens` block with `thinker` (required), `epistemic_depth` (primary/capable/target_description), `core_axioms` (axiom + implications), and `failure_signatures` (signature + description + mitigation). Added `composition` block with `pairs_well_with`, `covers_blind_spots_of`, and `has_blind_spots_covered_by` for agent pairing metadata. Formalized `handoff` block (upstream_context/downstream_artifacts) previously used but undocumented. Added composition patterns enum: adversarial_dialectic, parallel_reading, sequential_pipeline, complementary_coverage. Added `output.implications` field (`section_label`, `framing_question`, `scope_boundary`) per agent-output-implications-spec-v0.1.0 for type-specific epistemic scoping of output sections (replaces generic RECOMMENDATIONS). |
| 1.9.0 | 2026-02-27 | Added: `forecaster` agent type — prediction-oriented agents that model future states, emergent effects, and causal consequences of artifacts. Forecasters require `interface`, `mission`, and `process`; forbid `tasks`, `completion`, `rollback`. Like analysts, forecasters may optionally include `scoring`, `decisions`, `deductions`, and `auto_fail` for assessment-style prediction. Added optional `forecast` block with `actor_type`, `time_horizon`, `propagation_mechanism`, and `prediction_format` fields. Reclassified `perverse-outcome-detector` from analyst to forecaster as first instance of the new type. |
| 1.8.0 | 2026-02-26 | Added: `explorer` agent type — lightweight, mission-driven agents with no scoring framework for tool-augmented codebase exploration. Explorers require only `interface` and `mission`; forbid `scoring`, `decisions`, `deductions`, `auto_fail`, `tasks`, `completion`, `rollback`. Added explorer template, composition rules, multi-agent composition patterns, migration guide, and complete example (deep-explore). |
| 1.7.0 | 2026-02-20 | Added: `epistemic_limitations` in mission for acknowledging agent knowledge boundaries; `key_definitions` in knowledge_base for term glossary; `domain_taxonomy` in knowledge_base with categories, scale/fragility_scale, and anti-bias guidance; `description` in knowledge sections; `anti_pattern` in code examples; `interpretation` and `weight_rationale` in scoring; `decision_guidance` in decisions; `passes` and `pass_attribution_requirement` in reasoning_scaffolding for multi-pass methodologies; `templates` in output for named reusable templates. Relaxed maxLength on content-bearing strings (epistemic_limitations items: 300→1000, interpretation/weight_rationale: 500→2000). Updated analyst example with assumption-excavator showcasing all v1.7.0 features. |
| 1.6.0 | 2026-02-07 | Added: `domain_profile` optional field in interface for referencing domain profiles; `analyst` and `generator` agent types with composition rules; domain profile system (`domain-profile-schema-v1.0.0.json`, `software.profile.yaml`, `legal.profile.yaml`); analyst/generator factory templates; domain-conditional rendering in partials. |
| 1.5.1 | 2026-01-28 | Added: `data_sources` in context block for documenting external APIs/MCP tools; `formula` and `thresholds` in verification for mathematical criteria. |
| 1.5.0 | 2026-01-23 | Added: `vocabulary_rationale` in mission block to explain decision vocabulary choice; `explicit_prohibitions` in mission for hard agent boundaries; `success_criteria` in decisions block for explicit conditions beyond score threshold. |
| 1.4.0 | 2026-01-15 | Added: `reasoning_scaffolding` in process block for structured LLM reasoning; `pre_decision_checklist` in process for completeness verification; `examples` in output block for format demonstration. |
| 1.3.0 | 2026-01-14 | Added: `calibration_examples` in scoring for consistent LLM scoring; `failure_code_examples` in knowledge_base for taxonomy classification; `token_budget` in output for length guidance; `section_order` in output for explicit ordering; `display_id` on auto-fail conditions for human-readable IDs. |
| 1.2.0 | 2026-01-13 | Added: `mission` block for agent identity and purpose; `knowledge_base` block with sections, detection patterns, code examples, and references; extended `verification` with `verify_guidance`, `partial_scoring`, `definition_clarifications`; extended `edgeCase` with `condition_expression`, `report_wording`, `judgment_rationale`, `decision_override`, `compound_conditions`, `severity`. |
| 1.1.0 | 2026-01-12 | Added: `risk_level`, `tools` to interface; `context` block; retry configuration; input validation; extended domains for VDL compatibility; `commands`/`patterns` in process steps; `allow_secondary` in classification; `notify_on` in tracking; `fixed_score` in edge cases; comprehensive failure taxonomy definitions. |
| 1.0.0 | 2026-01-11 | Initial ADL specification. Unifies VDL (validators) with new executor support. Supersedes VDL v1.1.0. |

---

## Appendix: Type Definitions

### Common Types

```typescript
type Domain = 
  | 'software' 
  | 'legal' 
  | 'medical' 
  | 'financial' 
  | 'scientific' 
  | 'content' 
  | 'general'
  | 'career'        // VDL compatibility
  | 'compliance'    // VDL compatibility
  | 'business'      // VDL compatibility
  | 'education'     // VDL compatibility
  | 'documentation'    // VDL compatibility
  | 'cognitive-lens';  // v1.10.0 — epistemic framework agents

type AgentType = 'validator' | 'executor' | 'analyst' | 'generator' | 'explorer' | 'forecaster';

// Epistemic Nature Types (v1.13.0)
type Verifiability = 'mechanically_checkable' | 'expert_judgment' | 'not_checkable';
type Determinism = 'deterministic' | 'stochastic' | 'environment_dependent';
type ClaimType = 'factual' | 'normative' | 'observational';

interface EpistemicNature {
  verifiability?: Verifiability;
  determinism?: Determinism;
  claim_type?: ClaimType;
}

// Cognitive Lens Types (v1.10.0)
type EpistemicDepthLevel = 'first-order' | 'second-order' | 'third-order';
type CompositionPattern = 'adversarial_dialectic' | 'parallel_reading' | 'sequential_pipeline' | 'complementary_coverage';

type Model = 'haiku' | 'sonnet' | 'opus' | `${string}-${string}`;

type Tool = 'Read' | 'Grep' | 'Glob' | 'Bash' | 'Web' | 'API' | 'MCP';

type RiskLevel = 'low' | 'standard' | 'high' | 'critical';

type TimeoutBehavior = 'fail' | 'warn' | 'continue';

type Shell = 'bash' | 'sh' | 'zsh' | 'powershell';
```

### Failure Taxonomy Types

```typescript
type FailureDomainLabel = 'structural' | 'semantic' | 'pragmatic' | 'epistemic';

type FailureDomainCode = 'STR' | 'SEM' | 'PRA' | 'EPI';

type FailureModeCode = 
  | 'STR-OMI' | 'STR-EXC' | 'STR-MAL' | 'STR-INC' | 'STR-SYN' | 'STR-FMT'
  | 'SEM-INC' | 'SEM-COM' | 'SEM-AMB' | 'SEM-COH' | 'SEM-TYP' | 'SEM-LOG'
  | 'PRA-ALI' | 'PRA-MAT' | 'PRA-EFF' | 'PRA-FRA' | 'PRA-DOC' | 'PRA-TST'
  | 'EPI-OVR' | 'EPI-UND' | 'EPI-GRN' | 'EPI-FAL' | 'EPI-VAL' | 'EPI-VER';

type SeverityCode = 'critical' | 'high' | 'medium' | 'low' | 'info';

interface FailureTaxonomy {
  domain?: FailureDomainLabel;
  domain_code?: FailureDomainCode;
  failure_mode: FailureModeCode;
  default_severity?: SeverityCode;
}
```

### Retry Types

```typescript
interface RetryConfig {
  max_attempts?: number;      // 0-10, default: 0
  backoff_ms?: number;        // default: 1000
  backoff_multiplier?: number; // 1-5, default: 2
  retry_on?: string[];        // Error types to retry
}
```

### Validator Types (extended v1.5.0, v1.7.0)

```typescript
interface Scoring {
  maxScore: 100;
  categories: Category[];
  calibration_examples?: CalibrationExample[];
  constraints?: ScoringConstraints;
  interpretation?: string;      // v1.7.0
  weight_rationale?: string;    // v1.7.0
}

interface ValidatorDecision {
  vocabulary: {
    positive: string;  // e.g., "PASS"
    negative: string;  // e.g., "FAIL"
    conditional?: string;  // e.g., "WARN"
  };
  thresholds?: Threshold[];
  preset?: 'low_risk' | 'quality_gate' | 'high_stakes' | 'security' | 'critical';
  tracking?: {
    category: 'gate' | 'safety' | 'advisory';
    notify_on?: ('positive' | 'conditional' | 'negative')[];
  };
  success_criteria?: SuccessCriteria;  // v1.5.0
  decision_guidance?: string;          // v1.7.0
}

interface Threshold {
  decision: 'positive' | 'conditional' | 'negative';
  min_score?: number;
  max_score?: number;
  label?: string;
}

// v1.5.0
interface SuccessCriteria {
  description?: string;
  criteria: string[];  // 1-10 items, 200 chars each
}
```

### Executor Types

```typescript
interface ExecutorCompletion {
  vocabulary: {
    complete: string;  // e.g., "COMPLETE"
    partial: string;   // e.g., "PARTIAL"
    failed: string;    // e.g., "FAILED"
  };
  criteria: CompletionCriterion[];
}

interface CompletionCriterion {
  condition: string;  // Expression
  decision: 'complete' | 'partial' | 'failed';
  label?: string;
}

interface TaskInput {
  id: string;
  name: string;
  type: 'string' | 'number' | 'boolean' | 'array' | 'object' | 'enum' | 'file' | 'path';
  itemType?: string;
  values?: string[];
  default?: any;
  required?: boolean;
  source?: 'file' | 'argument' | 'previous_phase' | 'environment' | 'stdin' | 'prompt';
  description?: string;
  validation?: InputValidation;
}

interface InputValidation {
  pattern?: string;
  min?: number;
  max?: number;
  custom?: string;
}

interface TaskOutput {
  id: string;
  name: string;
  type: 'string' | 'number' | 'boolean' | 'array' | 'object' | 'file' | 'path';
  itemType?: string;
  format?: string;
  description?: string;
  required?: boolean;
}
```

### Context Types

```typescript
interface Context {
  working_directory?: string;
  environment?: Record<string, string>;
  timeout_behavior?: TimeoutBehavior;
  shell?: Shell;
}
```

### Rollback Types

```typescript
interface Rollback {
  enabled?: boolean;
  strategy?: 'git_restore' | 'git_stash' | 'backup_restore' | 'manual' | 'none';
  preserve_logs?: boolean;
  notify_on_rollback?: boolean;
}
```

### Mission Types (v1.2.0, extended v1.5.0, v1.7.0)

```typescript
interface Mission {
  opener?: string;
  stakes?: string;
  outcome_framing?: string;
  role_boundaries?: string[];
  taxonomy_mandate?: boolean;
  vocabulary_rationale?: string;    // v1.5.0
  explicit_prohibitions?: string[]; // v1.5.0
  epistemic_limitations?: string[]; // v1.7.0
}
```

### Knowledge Base Types (v1.2.0, extended v1.3.0, v1.7.0)

```typescript
interface KnowledgeBase {
  sections?: KnowledgeSection[];
  failure_code_examples?: FailureCodeExample[];  // v1.3.0
  global_references?: string[];
  key_definitions?: Record<string, string>;      // v1.7.0
  domain_taxonomy?: DomainTaxonomy;              // v1.7.0
}

// v1.3.0
interface FailureCodeExample {
  issue: string;
  failure_code: string;  // Pattern: ^(STR|SEM|PRA|EPI)-[A-Z]{3}/[CHMLI]$
  explanation?: string;
}

interface KnowledgeSection {
  category_ref: string;
  description?: string;                          // v1.7.0
  what_to_check?: string[];
  detection_patterns?: DetectionPattern[];
  red_flags?: CodeExample[];
  safe_patterns?: CodeExample[];
  references?: string[];
  common_mistakes?: CommonMistake[];
}

interface DetectionPattern {
  label: string;
  description?: string;
  pattern?: string;
  severity: SeverityCode;
  false_positive_hint?: string;
}

interface CodeExample {
  description: string;
  code: string;
  language?: string;
  severity?: SeverityCode;
  why?: string;
  anti_pattern?: boolean;                        // v1.7.0
}

interface CommonMistake {
  mistake: string;
  why_wrong: string;
  correct_approach: string;
}

// v1.7.0
interface DomainTaxonomy {
  completeness_note?: string;
  categories: TaxonomyCategory[];
  scale?: TaxonomyScale;
  fragility_scale?: TaxonomyScale;  // alias for scale
}

interface TaxonomyCategory {
  id?: string;
  name: string;
  description?: string;
  items?: string[];
}

interface TaxonomyScale {
  name?: string;
  description?: string;
  anti_bias_guidance?: string;
  levels: TaxonomyLevel[];
}

interface TaxonomyLevel {
  level?: string;
  score?: string;
  label: string;
  description?: string;
  meaning?: string;
  indicators?: string[];
}
```

### Enhanced Verification Types (v1.2.0)

```typescript
interface PartialScore {
  points: number;
  condition: string;
  condition_expression?: string;
  label?: string;
}

interface DefinitionClarification {
  term: string;
  definition: string;
}

// Extended Verification (v1.2.0 additions)
interface Verification {
  method: 'manual' | 'automated' | 'hybrid';
  checks?: string[];
  automation?: AutomationConfig;
  verify_guidance?: string;           // v1.2.0
  partial_scoring?: PartialScore[];   // v1.2.0
  definition_clarifications?: DefinitionClarification[];  // v1.2.0
}
```

### Enhanced Edge Case Types (v1.2.0)

```typescript
interface CompoundCondition {
  also_true: string;
  behavior_override?: string[];
  priority?: 'this' | 'other' | 'merge';
}

interface DecisionOverride {
  affects_decision?: boolean;
  forced_decision?: 'positive' | 'conditional' | 'negative';
  override_rationale?: string;
}

// Extended EdgeCase (v1.2.0)
interface EdgeCase {
  id: string;
  condition: string;
  condition_expression?: string;      // v1.2.0
  behavior: string[];
  score_adjustment?: ScoreAdjustment | null;
  report_wording?: string;            // v1.2.0
  judgment_rationale?: string;        // v1.2.0
  decision_override?: DecisionOverride;  // v1.2.0
  compound_conditions?: CompoundCondition[];  // v1.2.0
  severity?: SeverityCode;            // v1.2.0
}

interface ScoreAdjustment {
  exclude_categories?: string[];
  rescale?: boolean;
  fixed_score?: number;
}
```

### Calibration Types (v1.3.0)

```typescript
interface CalibrationExample {
  score: number;  // 0-100
  scenario: string;
  description?: string;
  deductions?: CalibrationDeduction[];
}

interface CalibrationDeduction {
  criterion: string;
  points_lost: number;  // 1-10
  reason?: string;
}
```

### Token Budget Types (v1.3.0)

```typescript
interface TokenBudget {
  target: number;   // Minimum: 100
  max: number;      // Minimum: 100
  guidance?: string;
}
```

### Auto-Fail Types (extended v1.3.0)

```typescript
interface AutoFailCondition {
  id: string;
  display_id?: string;  // v1.3.0: Pattern ^[A-Z]{2,4}-\d{3}$
  name: string;
  severity: 'critical';
  detection: AutoFailDetection;
  category_override?: string;
  evidence_required?: boolean;
  remediation?: string;
}
```

### Process Types (extended v1.4.0, v1.7.0)

```typescript
// v1.4.0, extended v1.7.0
interface ReasoningScaffolding {
  description?: string;
  steps?: ReasoningStep[];                       // v1.4.0
  passes?: ReasoningPass[];                      // v1.7.0
  pass_attribution_requirement?: string;         // v1.7.0
}

// v1.4.0
interface ReasoningStep {
  action: string;  // Pattern: ^[a-z][a-z0-9]*(_[a-z0-9]+)*$
  description?: string;
  example?: string;
}

// v1.7.0
interface ReasoningPass {
  id?: string;
  name: string;
  question?: string;
  focus?: string | string[];
  method?: string;
  output?: string;
  key_questions?: string[];
}

// Extended Process (v1.4.0)
interface Process {
  reasoning_scaffolding?: ReasoningScaffolding;  // v1.4.0
  pre_decision_checklist?: string[];             // v1.4.0
  phases?: ProcessPhase[];
}
```

### Output Types (extended v1.3.0, v1.4.0, v1.7.0)

```typescript
// Extended Output (v1.3.0, v1.4.0, v1.7.0)
interface Output {
  format?: 'markdown' | 'json' | 'html' | 'structured';
  schema?: string;
  token_budget?: TokenBudget;      // v1.3.0
  section_order?: string[];        // v1.3.0
  sections?: OutputSection[];
  symbols?: OutputSymbols;
  classification?: OutputClassification;
  structured?: StructuredOutputConfig;
  examples?: OutputExample[];      // v1.4.0
  templates?: Record<string, string>;  // v1.7.0
}

// v1.4.0
interface OutputExample {
  scenario: string;
  input_summary?: string;
  output: string;
}
```

### Domain Profile Types (v1.6.0)

```typescript
interface DomainProfile {
  interface: {
    domain: string;
    version: string;
    displayName: string;
    description: string;
  };
  severity_descriptions: Record<SeverityCode, {
    label: string;
    description: string;
    action: string;
  }>;
  issue_types: {
    types: DomainIssueType[];
    tracker_mapping: Record<string, string>;
  };
  common_edge_cases: DomainEdgeCase[];
  terminology: {
    artifact: string;
    review_unit: string;
    location_unit: string;
    positive_outcome: string;
    negative_outcome: string;
  };
  output_examples?: {
    issue_example?: {
      title: string;
      file_path: string;
      line_reference: string;
      failure_code: string;
      description: string;
    };
    location_format?: string;
  };
  knowledge_defaults?: {
    code_language?: string | null;
    example_format?: 'code_block' | 'prose' | 'table';
    references_style?: 'url' | 'legal_citation' | 'doi';
  };
  context_adjustments?: {
    context: string;
    adjustment: string;
  }[];
}

interface DomainIssueType {
  id: string;
  label: string;
  description: string;
}

interface DomainEdgeCase {
  id: string;
  condition: string;
  behavior: string[];
}
```
