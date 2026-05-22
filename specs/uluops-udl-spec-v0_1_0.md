# UluOps Unified Definition Language (UDL) Specification

**Version:** 0.1.0
**Status:** Draft
**Created:** 2026-04-13
**Author:** Alex Self, Ulu Labs Inc.

---

## Table of Contents

- [Abstract](#abstract)
- [1. Introduction](#1-introduction)
  - [1.1 Purpose](#11-purpose)
  - [1.2 Scope](#12-scope)
  - [1.3 Thesis](#13-thesis)
  - [1.4 Terminology](#14-terminology)
- [2. The Problem UDL Solves](#2-the-problem-udl-solves)
  - [2.1 The Judgment Infrastructure Gap](#21-the-judgment-infrastructure-gap)
  - [2.2 Why Existing Approaches Fail](#22-why-existing-approaches-fail)
  - [2.3 What Definition Languages Enable](#23-what-definition-languages-enable)
- [3. Architecture](#3-architecture)
  - [3.1 The Composition Hierarchy](#31-the-composition-hierarchy)
  - [3.2 Layer Responsibilities](#32-layer-responsibilities)
  - [3.3 Composition Rules](#33-composition-rules)
  - [3.4 Architectural Diagram](#34-architectural-diagram)
- [4. Language Family Members](#4-language-family-members)
  - [4.1 ADL — Agent Definition Language](#41-adl--agent-definition-language)
  - [4.2 CDL — Command Definition Language](#42-cdl--command-definition-language)
  - [4.3 WDL — Workflow Definition Language](#43-wdl--workflow-definition-language)
  - [4.4 PDL — Pipeline Definition Language](#44-pdl--pipeline-definition-language)
  - [4.5 Summary Matrix](#45-summary-matrix)
- [5. Shared Design Principles](#5-shared-design-principles)
  - [5.1 Declarative Over Imperative](#51-declarative-over-imperative)
  - [5.2 YAML with JSON Schema Validation](#52-yaml-with-json-schema-validation)
  - [5.3 Consistent Interface Pattern](#53-consistent-interface-pattern)
  - [5.4 Semantic Versioning](#54-semantic-versioning)
  - [5.5 Single Responsibility Per Definition](#55-single-responsibility-per-definition)
  - [5.6 Upward Composition, No Downward References](#56-upward-composition-no-downward-references)
  - [5.7 Domain Portability](#57-domain-portability)
  - [5.8 Forward Compatibility via Pattern Validation](#58-forward-compatibility-via-pattern-validation)
- [6. Shared Structural Patterns](#6-shared-structural-patterns)
  - [6.1 The Interface Block](#61-the-interface-block)
  - [6.2 The Root Object Convention](#62-the-root-object-convention)
  - [6.3 Naming Conventions](#63-naming-conventions)
  - [6.4 Domain Classification](#64-domain-classification)
  - [6.5 File Extension Convention](#65-file-extension-convention)
  - [6.6 Schema URL Convention](#66-schema-url-convention)
- [7. The Rendering Pipeline](#7-the-rendering-pipeline)
  - [7.1 Overview](#71-overview)
  - [7.2 Pure Function Property](#72-pure-function-property)
  - [7.3 Translation](#73-translation)
  - [7.4 Integrity Verification](#74-integrity-verification)
  - [7.5 Rendering Scope by Language](#75-rendering-scope-by-language)
- [8. Integration with Platform Services](#8-integration-with-platform-services)
  - [8.1 Registry Integration](#81-registry-integration)
  - [8.2 Tracker Integration](#82-tracker-integration)
  - [8.3 SDK Integration](#83-sdk-integration)
  - [8.4 MCP Integration](#84-mcp-integration)
- [9. The Failure Taxonomy](#9-the-failure-taxonomy)
  - [9.1 Relationship to UDL](#91-relationship-to-udl)
  - [9.2 Linguistic Grounding](#92-linguistic-grounding)
  - [9.3 Classification at Multiple Levels](#93-classification-at-multiple-levels)
- [10. Domain Profiles](#10-domain-profiles)
  - [10.1 Purpose](#101-purpose)
  - [10.2 How Profiles Work](#102-how-profiles-work)
  - [10.3 Portability Implications](#103-portability-implications)
- [11. Composition Patterns](#11-composition-patterns)
  - [11.1 Direct Composition](#111-direct-composition)
  - [11.2 Sequential Pipelines](#112-sequential-pipelines)
  - [11.3 Parallel Execution](#113-parallel-execution)
  - [11.4 Interleaved Compositions](#114-interleaved-compositions)
  - [11.5 Cognitive Lens Compositions](#115-cognitive-lens-compositions)
- [12. Versioning and Evolution](#12-versioning-and-evolution)
  - [12.1 Language Versioning](#121-language-versioning)
  - [12.2 Definition Versioning](#122-definition-versioning)
  - [12.3 Translator Versioning](#123-translator-versioning)
  - [12.4 Schema Evolution Strategy](#124-schema-evolution-strategy)
  - [12.5 UDL Family Versioning](#125-udl-family-versioning)
- [13. Licensing and Open Standard Strategy](#13-licensing-and-open-standard-strategy)
  - [13.1 Open vs. Patented Boundary](#131-open-vs-patented-boundary)
  - [13.2 Boundary Principle](#132-boundary-principle)
- [14. Novel Contributions](#14-novel-contributions)
- [15. Related Specifications](#15-related-specifications)
- [16. Design Decisions and Rationale](#16-design-decisions-and-rationale)
- [Appendix A: Glossary](#appendix-a-glossary)
- [Appendix B: Version History of Each Language](#appendix-b-version-history-of-each-language)
- [Changelog](#changelog)

---

## Abstract

The UluOps Unified Definition Language (UDL) is a family of four structured, YAML-based definition languages for specifying, composing, and orchestrating AI agent operations. The four languages — ADL (Agent Definition Language), CDL (Command Definition Language), WDL (Workflow Definition Language), and PDL (Pipeline Definition Language) — form a strict composition hierarchy from atomic agents to multi-workflow pipelines. UDL provides the declarative infrastructure for making AI reasoning observable, persistent, regressable, and improvable.

This specification defines the shared architecture, design principles, structural patterns, rendering pipeline, and composition rules that govern the entire UDL family. Individual language specifications define language-specific schemas and semantics; this document defines what is common across all four.

---

## 1. Introduction

### 1.1 Purpose

This specification serves three functions:

**Unification.** The four UDL languages evolved incrementally — ADL first (January 2026), followed by CDL, WDL, and PDL — each referencing shared patterns without a single document that formalized those patterns. This spec extracts and codifies what is shared so that the individual specs can reference a common foundation rather than duplicating it.

**Patent foundation.** The UDL family, as a coordinated system for defining AI agent operations at multiple levels of abstraction, constitutes a novel contribution. This spec provides the architectural documentation required for intellectual property protection — specifically, the system-level description that individual language specs do not provide because they each describe only their own layer.

**Onboarding.** A developer encountering the UluOps ecosystem for the first time needs to understand the overall architecture before diving into any individual language. This spec is that entry point.

### 1.2 Scope

This specification covers:

- The architectural relationship between the four languages
- Shared design principles and structural patterns
- The rendering pipeline that transforms definitions into runtime prompts
- Integration points with platform services (Registry, Tracker, SDK, MCP)
- Composition rules and patterns
- The failure taxonomy as it relates to the definition languages
- Versioning and evolution strategy

This specification does **not** cover:

- Language-specific schemas (see individual specs: ADL, CDL, WDL, PDL)
- Platform service APIs (see Registry API and Tracker API specs)
- Thinker profiles or cognitive lens architecture (see Thinker Profile Spec, Cognitive Lens Library Spec)
- The fingerprint-based issue correlation system (see Fingerprint Spec)
- Pricing, authentication, or commercial platform concerns

### 1.3 Thesis

UDL rests on a foundational claim: **judgment is now computational.** Large language models did not automate expert judgment — they made it executable. And anything executable can be instrumented, measured, versioned, and improved.

But executable judgment requires infrastructure. Without structured definitions, AI agents are ad-hoc prompts — untraceable, unreproducible, unimprovable. UDL provides the declarative layer that transforms ad-hoc prompting into engineered operations:

- **Definitions make judgment observable.** A YAML definition declares exactly what an agent evaluates, how it scores, and what constitutes success. The definition is readable by humans and parseable by machines.
- **Definitions make judgment persistent.** Stored in a versioned registry with immutable version records, definitions create an auditable history of how judgment criteria evolved.
- **Definitions make judgment regressable.** Because definitions are versioned and deterministically rendered, the same definition produces the same prompt on every execution. When findings change between runs, the change is attributable to the artifact under review, not to definition drift.
- **Definitions make judgment improvable.** The rendering pipeline separates definition authoring from prompt engineering. A definition can be improved (new criteria, adjusted thresholds, refined edge cases) and the improvement is traceable, testable, and reversible.

The four UDL languages provide this infrastructure at four levels of abstraction — from atomic cognitive operations (ADL) to enterprise-scale orchestration (PDL) — because judgment at scale requires composition, not just specification.

### 1.4 Terminology

Throughout this specification:

- **Definition** — A YAML document conforming to one of the four UDL schemas. A definition declares an agent, command, workflow, or pipeline.
- **Agent** — An atomic AI operation defined in ADL. The smallest unit of executable judgment.
- **Command** — An invocation context for a single agent (or a procedural step), defined in CDL.
- **Workflow** — A multi-step orchestration of commands and/or agents, defined in WDL.
- **Pipeline** — A multi-workflow orchestration with triggers, approvals, and rollback, defined in PDL.
- **Rendering** — The deterministic transformation of a YAML definition into a runtime prompt (markdown) suitable for LLM consumption.
- **Translation** — The template-based process that performs rendering. Translators are versioned independently of definitions.
- **Registry** — The platform service that stores, versions, and serves definitions.
- **Tracker** — The platform service that records execution results, correlates issues, and measures trends.

---

## 2. The Problem UDL Solves

### 2.1 The Judgment Infrastructure Gap

Before UDL, using AI for expert judgment meant writing prompts directly. This approach has structural limitations:

**No separation of concerns.** A single prompt conflates what the agent should evaluate (domain knowledge), how it should score (evaluation methodology), what constitutes success (thresholds and decisions), and how it should present results (output format). Changing any one concern requires rewriting the entire prompt.

**No versioning.** When a prompt is improved, the previous version is lost. There is no way to compare the performance of version N against version N+1 on the same artifact, and no way to roll back if a change degrades quality.

**No composition.** Complex judgment tasks require multiple agents operating in sequence, parallel, or both — with gates, aggregation, and conditional execution. Ad-hoc prompts cannot be composed into larger structures because they have no standardized interface.

**No reproducibility.** The same conceptual prompt, written by different people, produces different results. There is no canonical representation that guarantees deterministic prompt generation from a stable definition.

**No measurement.** Without structured output schemas, agent results cannot be programmatically compared across runs, correlated across agents, or trended over time.

### 2.2 Why Existing Approaches Fail

Several prior and contemporary systems address fragments of this problem. None address the full stack:

**Prompt libraries** (e.g., LangChain prompt templates) provide reusability but not structured evaluation criteria, scoring methodology, or composition hierarchy. They are string interpolation, not definition languages.

**AI framework configurations** (e.g., DSPy, CrewAI) define agent behavior programmatically. This couples definition to implementation, preventing separation of concerns between what an agent does and how it is executed. Definitions cannot be stored, versioned, or served independently of the runtime.

**CI/CD pipeline languages** (e.g., GitHub Actions YAML, GitLab CI) define execution workflows but have no concept of scoring, cognitive evaluation, or judgment-specific semantics. They orchestrate processes, not reasoning.

**Testing frameworks** (e.g., pytest, Jest) define assertions and test suites but operate on binary pass/fail. They cannot express partial scoring, multi-category evaluation, severity-aware thresholds, or the epistemological nuance required for judgment tasks where "correct" is a spectrum.

UDL addresses the full stack: structured definition of judgment operations, deterministic rendering to runtime prompts, composition at multiple levels of abstraction, and integration with persistent tracking for measurement and improvement.

### 2.3 What Definition Languages Enable

With UDL in place, the following capabilities become possible:

- **Agent-as-code.** Agent definitions are stored in version control, reviewed in pull requests, and deployed through CI/CD — the same workflow used for application code.
- **Registry-based discovery.** Definitions are published to a central registry, searchable by domain, agent type, tags, and effectiveness metrics. Teams discover and reuse existing agents rather than writing new prompts from scratch.
- **Fork-based innovation.** Any published definition can be forked, modified, and republished. Fork lineages are tracked, enabling evidence-based comparison of variants.
- **Deterministic rendering.** The same definition always produces the same prompt. This is the foundation for reproducible evaluation and meaningful regression detection.
- **Structured output.** Every agent type in ADL defines a structured output schema. Results are programmatically parseable, enabling automated scoring, trend analysis, and cross-agent comparison.
- **Composition at scale.** Workflows compose agents into multi-step evaluations with gates and aggregation. Pipelines compose workflows into CI/CD-integrated operations with triggers, approvals, and rollback.

---

## 3. Architecture

### 3.1 The Composition Hierarchy

UDL defines a strict four-level composition hierarchy. Each level wraps and orchestrates the level below it:

```
Level 4:  Pipeline (PDL)    — Multi-workflow orchestration
              │
              │  stages
              ▼
Level 3:  Workflow (WDL)    — Multi-step orchestration
              │
              │  phases
              ▼
Level 2:  Command (CDL)     — Single-agent invocation context
              │
              │  wraps
              ▼
Level 1:  Agent (ADL)       — Atomic cognitive operation
```

This hierarchy is not incidental — it reflects a fundamental decomposition of the concerns involved in running judgment at scale:

- **What to evaluate** → Agent (ADL)
- **How to invoke** → Command (CDL)
- **In what order** → Workflow (WDL)
- **When and under what governance** → Pipeline (PDL)

### 3.2 Layer Responsibilities

Each layer has precisely defined responsibilities and explicitly delegates everything else:

**ADL (Agent Definition Language)** — Defines the cognitive operation itself: what to evaluate, how to score, what constitutes a finding, how to make decisions. ADL is the content layer. An agent knows everything about its domain concern and nothing about how it will be invoked.

**CDL (Command Definition Language)** — Defines the invocation context: which model to use, what preflight checks to run, how to override defaults, whether to iterate until a threshold is met. CDL is the execution layer. A command knows how to invoke one agent but nothing about its relationship to other commands.

**WDL (Workflow Definition Language)** — Defines the orchestration: which commands (or agents) to run in what order, how to gate between phases, how to aggregate scores, and how to make workflow-level decisions. WDL is the coordination layer. A workflow knows the execution graph but nothing about external triggers or governance.

**PDL (Pipeline Definition Language)** — Defines the governance: what events trigger execution, who must approve critical stages, what happens on failure, how to notify external systems, and how to persist state across runs. PDL is the governance layer. A pipeline knows when and whether to run but delegates all cognitive work downward.

### 3.3 Composition Rules

The composition hierarchy enforces strict reference rules:

```
PDL  →  can reference:  WDL, CDL
WDL  →  can reference:  CDL, ADL (direct)
CDL  →  can reference:  ADL
ADL  →  cannot reference any other definition (atomic unit)
```

Key constraints:

- **No downward skipping.** PDL cannot reference ADL directly. Agents must be wrapped in commands or referenced through workflows.
- **No upward references.** ADL cannot reference CDL, WDL, or PDL. Lower layers are unaware of higher layers.
- **No lateral references.** An agent cannot reference another agent. A command cannot reference another command. Composition happens only through higher-level structures.
- **WDL shortcut.** Workflows can reference agents directly (without a CDL wrapper) when no execution context is needed. This is a convenience — structurally, the workflow treats the agent reference as an implicit command with default settings.

These rules ensure that each definition is independently testable, independently versionable, and independently understandable. An agent definition is complete in itself — it does not require knowledge of any command, workflow, or pipeline that might invoke it.

### 3.4 Architectural Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                     PDL — Pipeline Definition                       │
│  Triggers │ Stages │ Approvals │ Rollback │ Notifications │ State   │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                  WDL — Workflow Definition                    │   │
│  │  Phases │ Gates │ Dependencies │ Aggregation │ Context        │   │
│  │                                                               │   │
│  │  ┌────────────────────────────────────────────────────────┐  │   │
│  │  │              CDL — Command Definition                   │  │   │
│  │  │  Preflight │ Overrides │ Iteration │ Postflight         │  │   │
│  │  │                                                         │  │   │
│  │  │  ┌──────────────────────────────────────────────────┐  │  │   │
│  │  │  │            ADL — Agent Definition                 │  │  │   │
│  │  │  │  Mission │ Knowledge │ Scoring │ Process │ Output │  │  │   │
│  │  │  └──────────────────────────────────────────────────┘  │  │   │
│  │  └────────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 4. Language Family Members

### 4.1 ADL — Agent Definition Language

**Current Version:** 1.15.0
**File Extension:** `.agent.yaml`
**Root Object:** `agent`

ADL defines atomic AI agents — the smallest unit of executable judgment in the UluOps system. An agent encapsulates domain knowledge, evaluation methodology, scoring criteria, decision logic, and output structure in a single YAML document.

ADL supports six agent types, each with distinct output semantics:

| Type | Purpose | Output | Decision Vocabulary |
|------|---------|--------|---------------------|
| Validator | Analyze and score | Score + Findings | PASS / WARN / FAIL |
| Executor | Perform tasks | Artifacts + Changes | COMPLETE / PARTIAL / FAILED |
| Analyst | Investigate and assess | Analysis Report + Recommendations | Custom per agent |
| Generator | Produce artifacts | Generated Artifacts + Status | Custom per agent |
| Explorer | Discover and map | Exploration Report | Narrative (no decision) |
| Forecaster | Project and predict | Forecast Report + Scenarios | Custom per agent |

ADL is the most mature language in the family, having evolved through 15 major versions from its origin as the Validator Definition Language (VDL v1.0.0, January 2026). Its schema includes blocks for interface identity, epistemic nature, mission, knowledge base, scoring (validators), tasks (executors), decisions, process architecture, output format, edge cases, tone, and failure taxonomy integration.

**Key ADL innovation:** The 100-point scoring framework for validators, which replaces subjective pass/fail judgment with explicit, weighted, verifiable criteria. Every validator agent declares how 100 points are distributed across categories and criteria, making scoring deterministic and auditable.

### 4.2 CDL — Command Definition Language

**Current Version:** 2.0.0
**File Extension:** `.command.yaml`
**Root Object:** `command`

CDL defines the invocation context for a single agent. Commands solve the problem of separating what an agent evaluates (ADL) from how it is invoked — which model to use, what preflight checks to run, whether to iterate, and what to do after completion.

CDL supports three command types:

| Type | Purpose | Agent Required? |
|------|---------|-----------------|
| `agent` | Wraps any ADL agent with execution context | Yes |
| `utility` | Procedural steps without agent invocation | No |
| `custom` | Escape hatch for advanced use cases | Optional |

The `agent` command type is agent-type-agnostic — it wraps any ADL agent regardless of whether it is a validator, executor, analyst, generator, explorer, or forecaster. The agent's own `agentType` determines scoring semantics and decision vocabulary; the command does not duplicate this classification.

**Key CDL innovation:** The iteration block, which enables review-fix loops. An `agent` command can specify that execution should repeat until a quality threshold is met, with configurable maximum iterations. This transforms one-shot evaluation into convergent quality improvement — a command iterates an agent against an artifact until the artifact passes.

### 4.3 WDL — Workflow Definition Language

**Current Version:** 3.0.0
**File Extension:** `.workflow.yaml`
**Root Object:** `workflow`

WDL defines multi-step orchestrations that compose commands and agents into phases with dependencies, conditional execution, gates, and aggregated results. Workflows are the coordination layer — they determine the order, parallelism, and decision logic for multi-agent operations.

WDL's core structure is the phase — a named step that references one or more commands or agents and optionally declares gates (minimum score thresholds), conditions (skip logic), and dependencies (execution ordering). Phases can execute in parallel (via group assignment) or sequentially (via explicit dependencies).

**Key WDL innovation:** The DAG-based orchestration model with quality gates. Phases declare dependencies as a directed acyclic graph, and the runtime parallelizes independent phases automatically. Gates between phases enforce quality thresholds — a phase can block downstream execution if its aggregated score falls below a minimum. This enables progressive quality assurance: early phases catch coarse-grained issues, later phases perform fine-grained analysis, and no phase runs until prerequisites pass.

### 4.4 PDL — Pipeline Definition Language

**Current Version:** 1.2.0
**File Extension:** `.pipeline.yaml`
**Root Object:** `pipeline`

PDL defines the top-level orchestration layer — multi-workflow operations with event-driven triggers, human-in-the-loop approvals, failure rollback, external notifications, artifact preservation, and cross-run state persistence. Pipelines integrate UDL operations into CI/CD and production governance workflows.

PDL's core structure is the stage — a named unit that references one or more workflows or commands, declares dependencies on other stages, and optionally requires human approval before execution. Stages form a DAG, and the runtime parallelizes independent stages.

**Key PDL innovation:** The approval gate. Critical stages can require human authorization from specific teams before execution begins, with configurable timeout. This solves the governance problem for AI-in-the-loop workflows: automated quality assessment runs freely, but deployment decisions require human sign-off. The approval gate transforms AI agent output from autonomous action into human-supervised recommendation.

### 4.5 Summary Matrix

| Aspect | ADL | CDL | WDL | PDL |
|--------|-----|-----|-----|-----|
| **Current Version** | 1.15.0 | 2.0.0 | 3.0.0 | 1.2.0 |
| **Root Object** | `agent` | `command` | `workflow` | `pipeline` |
| **File Extension** | `.agent.yaml` | `.command.yaml` | `.workflow.yaml` | `.pipeline.yaml` |
| **Concern** | What to evaluate | How to invoke | In what order | When and under what governance |
| **Contains** | Criteria, knowledge, process | Agent reference, overrides, iteration | Phases, gates, aggregation | Stages, triggers, approvals, rollback |
| **References** | Nothing (atomic) | One ADL agent | CDL commands, ADL agents | WDL workflows, CDL commands |
| **Output** | Score + findings (varies by type) | Agent result + postflight | Aggregated score + phase results | Stage results + artifacts |
| **Rendered?** | Yes (translated to LLM prompt) | Yes (runtime markdown) | Yes (runtime markdown) | Structural (orchestration instructions) |

---

## 5. Shared Design Principles

The following principles apply to all four UDL languages. They are not guidelines — they are architectural invariants that the language schemas enforce.

### 5.1 Declarative Over Imperative

Every UDL language describes **what** should happen, not **how** it executes. Agents declare what to evaluate and how to score. Commands declare what overrides to apply and what preflight checks to run. Workflows declare what phases to execute and what gates to enforce. Pipelines declare what stages to run and what triggers to respond to.

The runtime (Core SDK) interprets definitions and orchestrates execution. This separation means that definitions are portable across runtime implementations — the same definition can be executed by different SDKs, in different programming languages, against different LLM providers.

### 5.2 YAML with JSON Schema Validation

All four languages use YAML as the authoring format and JSON Schema (Draft-07) for validation. This choice is deliberate:

- **YAML** is human-readable, supports comments, and is the de facto standard for declarative configuration in the developer ecosystem.
- **JSON Schema** provides machine-enforceable validation, enabling tooling (IDE autocomplete, CI validation, registry ingest checks) without custom parsers.

Every definition YAML document is validatable against its language's JSON Schema before storage or execution. The registry rejects definitions that fail schema validation.

### 5.3 Consistent Interface Pattern

All four languages use a structurally identical `interface` block as their primary identity mechanism:

```yaml
interface:
  name: string        # Kebab-case identifier (required)
  version: string     # Semantic version (required)
  displayName: string # Human-readable name, 3-50 chars (required)
  description: string # Purpose and scope, 20-500 chars (required)
  domain: string      # Primary domain classification (required)
  subdomain: string   # Optional domain refinement
  tags: string[]      # Optional searchable tags
```

This consistency means that every definition in the system — regardless of language — can be identified, searched, classified, and displayed using the same structure. Registry operations (list, search, filter) work identically across all four definition types.

### 5.4 Semantic Versioning

All definitions use semantic versioning (`MAJOR.MINOR.PATCH`). The version string follows the pattern `^\d+\.\d+\.\d+$` and is enforced by schema validation.

Semantic versioning enables:

- **Dependency resolution.** When a workflow references a command at a specific version, the registry can resolve exact matches or compatible ranges.
- **Breaking change signaling.** Major version bumps signal incompatible changes to consumers.
- **Immutable version records.** Once published, a specific version of a definition cannot be modified — only new versions can be created.

### 5.5 Single Responsibility Per Definition

Each definition serves one purpose:

- An agent evaluates one domain concern.
- A command invokes one agent (or performs one procedural step).
- A workflow orchestrates one logical operation (though it may contain multiple phases).
- A pipeline governs one deployment or CI/CD process.

This principle prevents definitions from accumulating unrelated responsibilities over time and ensures that each definition is independently testable, reviewable, and replaceable.

### 5.6 Upward Composition, No Downward References

Definitions compose strictly upward: agents are composed into commands, commands into workflows, workflows into pipelines. No definition references a definition at its own level or at a higher level.

This rule ensures that every definition is a self-contained unit. An agent can be understood, tested, and improved without knowing which commands invoke it. A command can be understood without knowing which workflows include it. This independence is what makes the fork-based ecosystem possible — a forked agent can replace its parent in any command that references it, without requiring changes to that command's definition.

### 5.7 Domain Portability

All four languages are domain-agnostic by design. The `domain` field in the interface block classifies a definition but does not constrain its schema. The same ADL schema supports software validation, legal document review, financial analysis, and any other domain where structured judgment is applicable.

Domain-specific vocabulary is injected through domain profiles (§10), not through language-level schema differences. This means the UDL architecture itself — the composition hierarchy, rendering pipeline, failure taxonomy, and tracking integration — works across all domains without modification.

### 5.8 Forward Compatibility via Pattern Validation

Where possible, UDL schemas use regex patterns rather than enumerations for extensible fields. The failure taxonomy's mode codes, for example, are validated against the pattern `^(STR|SEM|PRA|EPI)-[A-Z]{3}$` rather than an enumeration of known codes. This allows the taxonomy to be extended (new failure modes added) without schema changes and without breaking existing definitions.

The same principle applies to domain classifications, agent types, and other fields where the set of valid values is expected to grow. Pattern validation preserves forward compatibility while maintaining structural constraints.

---

## 6. Shared Structural Patterns

### 6.1 The Interface Block

Every UDL definition begins with an interface block inside a root object. The interface block is the definition's identity — it tells the registry, SDK, and human readers what this definition is, what version it is, and what domain it belongs to.

The interface block schema is shared across all four languages. Language-specific fields may be added (e.g., `agentType` in ADL, `commandType` in CDL), but the core fields — `name`, `version`, `displayName`, `description`, `domain` — are universal.

### 6.2 The Root Object Convention

Each language uses a single root object named after the definition type:

| Language | Root Object |
|----------|-------------|
| ADL | `agent:` |
| CDL | `command:` |
| WDL | `workflow:` |
| PDL | `pipeline:` |

This convention serves as a type discriminator — a YAML file's root key immediately identifies which language it belongs to, enabling automatic schema selection during validation and rendering.

### 6.3 Naming Conventions

All definitions use consistent naming:

- **Definition names:** Kebab-case, matching the pattern `^[a-z][a-z0-9]*(-[a-z0-9]+)*$`. Examples: `code-validator`, `ship`, `release-pipeline`.
- **Display names:** Human-readable, 3-50 characters. Examples: `Code Validator`, `Ship Workflow`, `Release Pipeline`.
- **Descriptions:** 20-500 characters, describing purpose and scope.

These constraints are schema-enforced across all four languages, ensuring consistent naming in registry listings, CLI output, and documentation.

### 6.4 Domain Classification

All definitions declare a `domain` field from a shared vocabulary:

- `software` — Software engineering artifacts
- `legal` — Legal documents, contracts, compliance
- `financial` — Financial analysis and reporting
- `research` — Academic and scientific research
- `general` — Domain-agnostic operations
- (extensible via pattern validation)

The optional `subdomain` field provides refinement (e.g., `domain: software`, `subdomain: security`).

### 6.5 File Extension Convention

Each language has a recommended file extension that encodes the definition type:

| Language | Extension | Example |
|----------|-----------|---------|
| ADL | `.agent.yaml` | `code-validator.agent.yaml` |
| CDL | `.command.yaml` | `validate.command.yaml` |
| WDL | `.workflow.yaml` | `ship.workflow.yaml` |
| PDL | `.pipeline.yaml` | `release.pipeline.yaml` |

The double extension (`.type.yaml`) enables file system tooling (IDE plugins, linters, watchers) to identify definition type without parsing the file.

### 6.6 Schema URL Convention

Each language version has a canonical schema URL following the pattern:

```
https://uluops.ai/schemas/{language}/{version}/{type}.json
```

Examples:

- `https://uluops.ai/schemas/adl/v1.15.0/agent.json`
- `https://uluops.ai/schemas/cdl/v2.0.0/command.json`
- `https://uluops.ai/schemas/wdl/v3.0.0/workflow.json`
- `https://uluops.ai/schemas/pdl/v1.2.0/pipeline.json`

Schema URLs are immutable — a published schema URL always resolves to the same schema.

---

## 7. The Rendering Pipeline

### 7.1 Overview

The rendering pipeline transforms a YAML definition into a runtime prompt — the markdown text that an LLM actually processes. This transformation is the critical bridge between declarative specification and executable judgment.

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ Definition  │────▶│ Translator  │────▶│  Rendered   │────▶│  SHA-256   │
│   (YAML)    │     │  Selection  │     │   Prompt    │     │   Hash     │
│             │     │  (Nunjucks) │     │ (Markdown)  │     │            │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
                                                                   │
                                                                   ▼
                                                            ┌─────────────┐
                                                            │  Integrity  │
                                                            │ Verification│
                                                            └─────────────┘
```

### 7.2 Pure Function Property

The rendering pipeline is a pure function: the same definition input always produces the same rendered prompt output. This property is architecturally critical because it enables:

- **Deterministic execution.** Running the same definition against the same artifact produces the same prompt, eliminating a variable in result comparison.
- **Reliable caching.** Rendered prompts can be cached by (definition hash, translator version) tuple.
- **Integrity verification.** The SHA-256 hash of a rendered prompt can be stored alongside execution results, enabling post-hoc verification that the correct prompt was used.
- **Testing in isolation.** Rendered prompts can be inspected and tested without executing the full pipeline.

### 7.3 Translation

Translation is the template-based process that converts structured YAML into prose markdown. The UluOps platform uses Nunjucks templates (translators) to perform this conversion.

Key properties of translators:

- **Versioned independently.** Translators have their own version numbers, decoupled from definition versions. A translator upgrade can improve prompt quality for all existing definitions without requiring definition changes.
- **Language-specific.** Each UDL language has its own translator templates, reflecting the different output structures of agents, commands, workflows, and pipelines.
- **Agent-type-aware.** Within ADL, translators produce different prompt structures for validators, executors, analysts, generators, explorers, and forecasters — selecting the appropriate template based on the `agentType` discriminator.

### 7.4 Integrity Verification

After rendering, the pipeline computes a SHA-256 hash of the rendered prompt. This hash is stored with the execution results in the Tracker, enabling:

- **Audit trail.** For any execution result, the exact prompt that produced it can be reconstructed and verified.
- **Tampering detection.** If a rendered prompt is modified between rendering and execution, the hash mismatch reveals the modification.
- **Regression attribution.** When an agent produces different results on subsequent runs, hash comparison determines whether the difference is due to a definition change, a translator change, or a genuine change in the artifact under review.

### 7.5 Rendering Scope by Language

Not all languages render to the same kind of output:

| Language | Renders To | Consumed By |
|----------|------------|-------------|
| ADL | LLM prompt (markdown) | AI model via SDK |
| CDL | Runtime instructions (markdown) | SDK execution engine |
| WDL | Orchestration instructions (markdown) | SDK workflow engine |
| PDL | Structural configuration | SDK pipeline engine / CI/CD system |

ADL rendering is the most complex — it produces the detailed, multi-section prompt that an LLM processes to perform evaluation. CDL and WDL rendering produce runtime instructions that the SDK interprets. PDL definitions are primarily structural and may not require template-based rendering at all.

---

## 8. Integration with Platform Services

UDL definitions do not exist in isolation — they are stored, served, executed, and tracked by the UluOps platform services.

### 8.1 Registry Integration

The Registry API (`uluops-registry-api`) is the single source of truth for all definitions:

- **Storage.** Definitions are stored as versioned YAML in the registry database, with each version record immutable after publication.
- **Validation.** The registry validates every definition against its language's JSON Schema on ingest, rejecting invalid definitions before they enter the system.
- **Discovery.** The registry provides search and filtering across all definition types — by name, domain, agent type, tags, visibility, and effectiveness metrics.
- **Resolution.** The SDK resolves definition references (e.g., `code-validator@1.2.0`) by querying the registry. Resolution returns the complete YAML and the translator version for rendering.
- **Forking.** Any published definition can be forked. The registry maintains the complete fork graph — the lineage of all definitions and their derivatives.
- **Dependency tracking.** The registry tracks which definitions reference which other definitions, enabling impact analysis when a definition changes.

### 8.2 Tracker Integration

The Tracker API (`uluops-validation-tracker-api`) records what happens when definitions are executed:

- **Run records.** Each execution produces a run record containing scores, findings, definition hashes, model information, and timestamps.
- **Issue correlation.** Findings are correlated across runs using fingerprint-based identity, enabling persistence tracking (how long has this issue existed?) and regression detection (did this issue reappear?).
- **Effectiveness metrics.** The tracker computes per-definition effectiveness metrics — pass rates, average scores, failure mode distributions, false positive rates — enabling evidence-based comparison of definitions and their forks.
- **Trend analysis.** Metrics are trended over time, revealing whether a definition's target artifacts are improving, whether a definition is becoming more or less noisy, and whether model upgrades affect results.

### 8.3 SDK Integration

The Core SDK (`@uluops/core`) is the execution engine:

- **Definition resolution.** The SDK fetches definitions from the registry at runtime.
- **Rendering.** The SDK invokes the rendering pipeline to transform YAML into prompts.
- **Execution.** The SDK sends rendered prompts to LLM providers and parses responses against expected output schemas.
- **Result submission.** The SDK submits execution results to the tracker for persistence and analysis.
- **Model agnosticism.** The SDK supports multiple LLM providers (Anthropic, OpenAI, Google) through the Vercel AI SDK, with provider-specific extraction logic to normalize diverse output formats.

### 8.4 MCP Integration

MCP (Model Context Protocol) servers expose UDL operations as tools that AI agents can discover and invoke natively:

- **Registry MCP.** Enables agent discovery, definition retrieval, and execution through MCP tool calls.
- **Tracker MCP.** Enables run submission, issue tracking, and analytics queries through MCP tool calls.

MCP integration means that AI agents can compose with UDL-defined agents — an AI assistant can invoke a UluOps validation agent as a tool, receive structured results, and incorporate them into its own reasoning.

---

## 9. The Failure Taxonomy

### 9.1 Relationship to UDL

The UluOps Failure Taxonomy (v1.0.0) is a companion specification that classifies the findings produced by UDL-defined agents. While not a definition language itself, the taxonomy is deeply integrated with UDL at multiple levels:

- **ADL integration.** Agent definitions can declare expected failure modes at the criterion level, enabling static analysis of coverage. Agents produce findings classified using taxonomy codes.
- **CDL integration.** Command postflight actions can filter or route based on failure classification.
- **WDL integration.** Workflow aggregation can weight or gate based on failure domain distribution.
- **Tracker integration.** All findings are stored with taxonomy classifications, enabling cross-run, cross-agent, and cross-domain analysis.

### 9.2 Linguistic Grounding

The taxonomy's four domains derive from the classical hierarchy of language analysis:

| Domain | Code | Linguistic Basis | Question |
|--------|------|------------------|----------|
| Structural | STR | Syntax | Is it well-formed? |
| Semantic | SEM | Semantics | Is it true and coherent? |
| Pragmatic | PRA | Pragmatics | Does it serve its purpose? |
| Epistemic | EPI | Epistemics | Are claims warranted by evidence? |

This linguistic grounding is not decorative — it provides an exhaustive decomposition. Any quality judgment about any artifact can be located within one of these four layers, because the layers correspond to the four aspects of any communicative act: form, meaning, contextual fit, and evidential status.

The taxonomy defines 24 failure modes (6 per domain) with collision-free compound codes (e.g., `STR-OMI` for structural omission, `SEM-INC` for semantic incorrectness). The compound code format `{DOMAIN}-{MODE}/{SEVERITY}` encodes domain, mode, and severity in a single parseable token.

### 9.3 Classification at Multiple Levels

The taxonomy operates at two levels within the UDL system:

**Definition-time classification (static).** Agent definitions can declare expected failure modes for each criterion. This enables coverage analysis — identifying which failure modes a given agent portfolio can detect.

**Runtime classification (dynamic).** Agent findings are classified when detected, either by the agent itself (using taxonomy codes embedded in the prompt) or by a post-hoc classifier. Runtime classification enables the full suite of tracker analytics.

---

## 10. Domain Profiles

### 10.1 Purpose

Domain profiles solve the vocabulary portability problem. UDL schemas are domain-agnostic, but the prompts rendered from those schemas inevitably contain domain-specific language — severity descriptions, issue type labels, edge case examples, and terminology. Without profiles, software-domain assumptions leak into non-software agents.

### 10.2 How Profiles Work

A domain profile is a YAML file stored at `udl/definition-languages/profiles/{name}.profile.yaml` and validated against its own JSON Schema (`domain-profile-schema-v1.0.0.json`). Profiles provide domain-appropriate content for sections that would otherwise default to software terminology:

- **Severity descriptions.** What "critical" means in software (crashes) vs. legal (regulatory violation) vs. financial (material misstatement).
- **Issue types.** Domain vocabulary for classification (e.g., legal profiles use "omission" and "non-compliance" instead of "bug" and "feature").
- **Edge cases.** Domain-specific scenarios (e.g., a legal profile includes "statute of limitations approaching" rather than "non-Git repository").
- **Terminology.** Vocabulary substitutions for domain-neutral terms (e.g., "artifact" → "contract", "review unit" → "clause").

### 10.3 Portability Implications

Domain profiles are what make UDL's domain portability claim concrete rather than theoretical. The claim is not merely that the schemas are generic — it is that a working rendering pipeline exists that substitutes domain-appropriate content at every point where domain assumptions would otherwise be embedded. The profile system has been validated with software and legal profiles; the architecture supports arbitrary domain extension.

---

## 11. Composition Patterns

UDL's composition hierarchy enables several patterns for combining agents into multi-agent operations.

### 11.1 Direct Composition

The simplest pattern: a workflow references agents directly, running them sequentially or in parallel against the same artifact.

```yaml
workflow:
  orchestration:
    phases:
      - id: validate
        steps:
          - agent: code-validator@1.0.0
          - agent: security-auditor@1.0.0
```

### 11.2 Sequential Pipelines

Agents run in sequence, with later agents receiving the output of earlier agents. Quality gates between phases ensure that only passing artifacts proceed to deeper analysis.

```yaml
orchestration:
  phases:
    - id: structural
      steps:
        - agent: structure-validator@1.0.0
      gate:
        threshold: 70
    - id: semantic
      depends_on: [structural]
      steps:
        - agent: logic-analyzer@1.0.0
```

### 11.3 Parallel Execution

Independent agents run simultaneously, with results aggregated after all complete. This pattern is used when agents examine different aspects of the same artifact and their analyses are independent.

```yaml
orchestration:
  phases:
    - id: style
      group: 1
      steps:
        - agent: style-validator@1.0.0
    - id: security
      group: 1
      steps:
        - agent: security-auditor@1.0.0
    - id: synthesis
      group: 2
      barrier: true
      steps:
        - agent: workflow-synthesis@1.0.0
```

### 11.4 Interleaved Compositions

The most powerful pattern: compositions that combine domain validators, cognitive lens agents, and meta-layer agents in a three-group DAG:

- **Group 1** (parallel): Domain-specific agents and cognitive lens agents read the source artifact independently.
- **Group 2** (meta-analysis): Meta-layer agents (assumption excavators, contradiction detectors, incentive mappers) read Group 1's output and find compound findings invisible to any single agent.
- **Group 3** (synthesis): A synthesis agent reads all prior output and produces a unified assessment.

This three-dimensional interleaving — domain expertise × cognitive framework × meta-analysis — is the architectural basis for cognitive parallax: the empirically validated finding that different cognitive lenses produce near-zero overlapping findings on the same artifact.

### 11.5 Cognitive Lens Compositions

A specific application of interleaved compositions: workflows that pair cognitive lens agents (Aristotle, Hume, Popper, Confucius, Nietzsche, etc.) to exploit their structural complementarity. Examples include:

- **Adversarial dialectic.** Confucius (rectification) + Nietzsche (genealogy) examining the same naming conventions.
- **School-as-phase.** Epictetus (control boundaries) → Marcus Aurelius (system meditations) → Seneca (forecasting) as a Stoic analysis pipeline.
- **Paradigm probe.** Kuhn (paradigm detection) + Popper (falsification) + Nietzsche (genealogy) examining the same architectural assumptions.

These compositions are specified in WDL and depend on the ADL thinker profile system for their agent definitions.

---

## 12. Versioning and Evolution

### 12.1 Language Versioning

Each UDL language has its own independent version number, following semantic versioning. Language versions advance based on schema changes — new fields, new types, structural modifications.

Current language versions:

| Language | Version | Schema Ref |
|----------|---------|------------|
| ADL | 1.15.0 | `adl-schema-v1_15_0.json` |
| CDL | 2.0.0 | `cdl-schema-v2_0_0.json` |
| WDL | 3.0.0 | `wdl-schema-v3_0_0.json` |
| PDL | 1.2.0 | `pdl-schema-v1_2_0.json` |

### 12.2 Definition Versioning

Individual definitions are versioned independently of the language they are written in. A `code-validator@2.3.1` agent definition is at definition version 2.3.1, regardless of whether it was authored against ADL v1.10.0 or v1.15.0.

Definition versions are immutable after publication. To modify a published definition, a new version must be created.

### 12.3 Translator Versioning

Translators (the Nunjucks templates that render definitions to prompts) are versioned independently of both language versions and definition versions. A translator upgrade improves rendering for all existing definitions — new prompt structures, better formatting, additional context — without requiring definition changes.

The translator version is recorded with every execution result, enabling attribution when rendering changes affect agent output.

### 12.4 Schema Evolution Strategy

UDL schemas evolve through additive changes wherever possible:

- **New optional fields** are added in minor versions. Existing definitions remain valid.
- **New enumeration values** are added in minor versions (where enumerations are used). Pattern-validated fields require no schema change for new values.
- **Breaking changes** (removing fields, changing types, restructuring) are reserved for major versions and require migration documentation.

The ADL spec includes explicit migration guides between major versions (VDL → ADL 1.0, ADL 1.5 → 1.6, 1.6 → 1.7, etc.), establishing a precedent for managed schema evolution across the UDL family.

### 12.5 UDL Family Versioning

In addition to individual language versions, UDL maintains a **family version** that tracks the coordinated state of the four-language system. The family version increments when a change to any language affects the shared architectural contract — the patterns and invariants that other languages depend on.

**Current UDL Family Version: 0.1.0**

#### What Triggers a Family Version Bump

| Change Type | Example | Family Version Impact |
|-------------|---------|----------------------|
| **Shared interface block change** | Adding a required field to the interface block that all four languages must adopt | MAJOR bump |
| **Composition rule change** | Allowing PDL to reference ADL directly (currently prohibited) | MAJOR bump |
| **New language addition** | Adding a fifth definition language to the family | MAJOR bump |
| **Shared naming convention change** | Changing the kebab-case pattern for definition names | MAJOR bump |
| **Rendering pipeline contract change** | Changing the hash algorithm from SHA-256 to SHA-3 | MAJOR bump |
| **New shared structural pattern** | Adding a universal `metadata` block across all four languages | MINOR bump |
| **New domain in shared vocabulary** | Adding `healthcare` to the domain classification list | MINOR bump |
| **Shared schema URL convention change** | Changing the URL pattern for schema hosting | MINOR bump |
| **Failure taxonomy version update** | New failure modes added to the taxonomy | MINOR bump |
| **Documentation-only changes** | Clarifying shared principles, fixing examples | PATCH bump |
| **Individual language changes** | ADL adding a new agent type, CDL adding a new block | No family version impact (unless it changes the shared contract) |

#### Family Version vs. Language Version

The family version and individual language versions are independent but related:

- A **family MAJOR bump** signals to all language spec maintainers that a shared invariant has changed. Each language spec must update to conform, though the timeline for individual language updates may vary.
- A **language MAJOR bump** does *not* automatically trigger a family version bump. ADL can ship a breaking change (e.g., restructuring the scoring block) that affects only ADL consumers, without impacting CDL, WDL, or PDL.
- The **family version is recorded** in each individual language spec's metadata, creating a traceable link between the language version and the family contract it conforms to.

#### Family Version Record

| UDL Family Version | Date | Languages | Key Change |
|--------------------|------|-----------|------------|
| 0.1.0 | 2026-04-13 | ADL 1.15.0, CDL 2.0.0, WDL 3.0.0, PDL 1.2.0 | Initial family specification |

#### Patent Relevance

The family version is the canonical reference for patent claims that describe the *system* of four coordinated languages. A patent claim referencing "the UDL system" should cite a specific family version, not individual language versions, because the claimed invention is the coordinated architecture — not any single language's schema. The family version ensures that patent claims have a stable, citable identifier for the system-level innovation.

---

## 13. Licensing and Open Standard Strategy

The UDL specification family and the Failure Taxonomy specification are intended to be published as open standards under the following terms:

- **Specifications.** CC BY 4.0 (Creative Commons Attribution) — anyone can use, adapt, and build upon the specifications.
- **Patents.** Royalty-free patent grant for implementations conforming to the specifications.
- **Tooling.** The UluOps platform (Registry, Tracker, SDK, MCP servers) is proprietary. The specifications are open; the implementation is commercial.

This strategy follows the model established by web standards (HTML, CSS, HTTP) and modern infrastructure standards (OpenAPI, GraphQL): the specification is public and freely implementable, creating ecosystem adoption; the reference implementation and hosted platform are commercial, creating revenue.

The open standard strategy is also a defensive IP posture: by publishing the specifications under CC BY 4.0 with a royalty-free patent grant, UluOps establishes prior art that prevents competitors from patenting the specification itself, while retaining patent protection for the novel system-level innovations that the specifications enable.

### 13.1 Open vs. Patented Boundary

The following table delineates the boundary between open specification (freely implementable) and patented system-level innovations (protected IP). This boundary is critical for patent counsel, ecosystem partners, and prospective implementors.

| Component | Status | License / Protection | Rationale |
|-----------|--------|---------------------|-----------|
| **OPEN SPECIFICATION** | | | |
| ADL YAML schema and field definitions | Open | CC BY 4.0 + royalty-free patent grant | Ecosystem adoption; anyone can write ADL agents |
| CDL YAML schema and field definitions | Open | CC BY 4.0 + royalty-free patent grant | Ecosystem adoption; anyone can write CDL commands |
| WDL YAML schema and field definitions | Open | CC BY 4.0 + royalty-free patent grant | Ecosystem adoption; anyone can write WDL workflows |
| PDL YAML schema and field definitions | Open | CC BY 4.0 + royalty-free patent grant | Ecosystem adoption; anyone can write PDL pipelines |
| UDL composition hierarchy (the four-level structure) | Open | CC BY 4.0 + royalty-free patent grant | Architectural standard; prior art established by publication |
| Failure taxonomy domain/mode/severity structure | Open | CC BY 4.0 + royalty-free patent grant | Classification standard; ecosystem interoperability |
| Interface block pattern and naming conventions | Open | CC BY 4.0 + royalty-free patent grant | Interoperability requirement |
| Domain profile specification | Open | CC BY 4.0 + royalty-free patent grant | Enables domain portability for any implementor |
| JSON Schema files for all four languages | Open | CC BY 4.0 + royalty-free patent grant | Validation tooling interoperability |
| **PATENTED INNOVATIONS** | | | |
| Fingerprint-based issue correlation across non-deterministic AI output | Protected | Patent pending (provisional filed) | Novel algorithm for identity in non-deterministic systems |
| Recursive validation architecture (agents evaluating their own definitions) | Protected | Patent pending (provisional filed) | Novel self-improvement methodology |
| Deterministic rendering pipeline (YAML → translator → prompt → SHA-256 hash) | Protected | Patent claims | Novel system for reproducible AI prompt generation |
| Integrated three-service platform (Registry + Tracker + SDK) | Protected | Patent claims | Novel platform architecture for AI judgment infrastructure |
| Wontfix-as-ground-truth calibration method | Protected | Patent claims | Novel use of human override data for AI reliability measurement |
| Cognitive parallax measurement via taxonomy-based overlap analysis | Protected | Patent claims | Novel method for quantifying complementarity across AI cognitive frameworks |
| Convergence characterization (Gödelian boundary detection in self-evaluating systems) | Protected | Patent claims | Novel empirical finding with measurement methodology |
| Quality-gated DAG orchestration with AI judgment score thresholds | Protected | Patent claims | Novel workflow primitive for AI judgment operations |
| Three-dimensional interleaved agent composition (domain × cognitive lens × meta-layer) | Protected | Patent claims | Novel composition architecture with empirically validated complementarity |
| **PROPRIETARY IMPLEMENTATION** | | | |
| UluOps Registry API service | Proprietary | Commercial license | Reference implementation |
| UluOps Tracker API service | Proprietary | Commercial license | Reference implementation |
| UluOps Core SDK (`@uluops/core`) | Proprietary | Commercial license | Reference implementation |
| UluOps MCP servers | Proprietary | Commercial license | Reference implementation |
| Nunjucks translator templates | Proprietary | Commercial license | Implementation of rendering pipeline |
| Thinker profiles and cognitive lens library | Proprietary | Commercial license | Curated content library |
| UluOps web platform (uluops.ai, uluops.dev) | Proprietary | Commercial license | Hosted platform |

### 13.2 Boundary Principle

The guiding principle for this boundary: **the specification is the grammar; the innovations are the semantics and the system.**

Anyone can implement the grammar — write ADL YAML, validate it against the schema, build a registry that stores it. The patents protect what happens when the grammar is combined with the rendering pipeline, the tracking system, the fingerprint algorithm, and the recursive improvement methodology. This mirrors how HTTP (open specification) coexists with patented innovations in web servers, CDNs, and search engines that use HTTP as their protocol.

Patent counsel should note: the open specification establishes prior art for the schemas and composition hierarchy, preventing competitors from patenting the definition formats themselves. The patented innovations sit *above* the specification — they describe what the system does with definitions, not what definitions look like.

---

## 14. Novel Contributions

The UDL system, taken as a whole, introduces several innovations not found in prior art:

**1. A four-level composition hierarchy for AI agent operations.** No prior system defines AI agent behavior at four distinct levels of abstraction (atomic operation, invocation context, multi-step orchestration, governance) with strict composition rules between levels. Existing frameworks either conflate these levels or address only one.

**2. Declarative definition of AI judgment with deterministic rendering.** The combination of structured YAML definitions and a pure-function rendering pipeline produces deterministic prompts from stable specifications. This property — same definition → same prompt → comparable results — is the foundation for meaningful regression detection in AI systems, and is not provided by any prior prompt management system.

**3. A 100-point scoring framework for AI evaluation agents.** ADL's validator scoring system replaces binary pass/fail with weighted, multi-category, criteria-level scoring. The framework requires explicit point allocation summing to 100, with defined thresholds mapping scores to decisions. This eliminates the subjectivity inherent in unstructured evaluation prompts.

**4. Agent-type-polymorphic command wrapping.** CDL's `agent` command type wraps any ADL agent regardless of its agent type. The agent's own type determines scoring semantics and decision vocabulary; the command provides execution context without duplicating type classification. This decoupling means that adding a new agent type to ADL requires no changes to CDL.

**5. Quality-gated DAG orchestration for AI judgment workflows.** WDL's phase-based orchestration combines directed acyclic graph dependency resolution with quality gate enforcement. Phases execute in parallel where independent and block on score thresholds where sequential.

*Prior art distinction.* Standard DAG executors (Airflow, Prefect, Dagster) orchestrate computational tasks with binary success/failure gates — a task either completes or it doesn't. WDL gates operate on *score thresholds from AI judgment output* — a phase can produce a score of 62 and the gate decides whether 62 is sufficient to proceed, based on a declared threshold. This is a fundamentally different primitive: the gate input is a continuous quality score produced by an AI agent's structured evaluation, not a process exit code. Furthermore, WDL gates are *schema-enforced* — the threshold, failure behavior (`stop`, `continue`, `abort`), and aggregation method are declared in the definition YAML and validated against JSON Schema before execution. They are not ad-hoc conditionals written in pipeline code. The combination of AI-judgment-scored gates + schema-enforced thresholds + DAG parallelization is not found in any prior workflow orchestration system.

**6. Domain-portable AI agent infrastructure via profile substitution.** The domain profile system enables the same schema, rendering pipeline, and tracking infrastructure to serve agents across arbitrary domains. Domain-specific vocabulary is injected at rendering time, not at schema design time. No prior system provides this level of domain portability for AI agent definitions.

**7. Linguistically-grounded failure taxonomy integrated with definition schemas.** The four-domain failure taxonomy (Structural, Semantic, Pragmatic, Epistemic) derives from the hierarchy of language analysis and is embedded in UDL schemas at both definition-time and runtime. No prior AI quality system uses this specific linguistic grounding as the basis for failure classification.

**8. Three-dimensional interleaved compositions with empirically validated cognitive parallax.** The ability to compose domain validators, cognitive lens agents, and meta-layer agents in a single workflow — producing compound findings invisible to any single agent — is an architectural capability enabled by UDL's composition hierarchy.

*Prior art distinction.* Multi-agent frameworks (CrewAI, AutoGen, LangGraph) enable multiple agents to collaborate, but the agents operate within the same cognitive framework — they are differentiated by role (researcher, writer, reviewer), not by epistemological orientation. UDL's interleaved compositions combine three structurally distinct agent families:

1. **Domain agents** — evaluate against domain-specific criteria (software quality, legal compliance, security posture).
2. **Cognitive lens agents** — evaluate through distinct epistemological frameworks (Aristotelian categorization, Popperian falsification, Nietzschean genealogy). These are not personality styles or tonal variations; they encode different axioms, characteristic reasoning moves, and quality criteria that produce structurally different findings from the same artifact.
3. **Meta-layer agents** — operate on the *combined output* of domain and cognitive agents to find compound findings (contradictions, hidden assumptions, unstated incentives) that no single agent can detect.

The three families are not interchangeable — they see different things by design. This is empirically validated: production runs demonstrate near-zero finding overlap between cognitive lenses examining the same codebase (cognitive parallax). The architectural requirement for this capability is a composition system (WDL) that supports three-group DAGs where Group 2 reads Group 1's output and Group 3 reads all prior output, combined with a classification system (the failure taxonomy) that can measure overlap quantitatively. No prior multi-agent framework provides this combination of structurally distinct agent families + empirically measured complementarity + schema-enforced composition rules.

---

## 15. Related Specifications

| Specification | Version | Relationship |
|---------------|---------|--------------|
| UluOps ADL Spec | v1.15.0 | ADL language specification |
| UluOps CDL Spec | v2.0.0 | CDL language specification |
| UluOps WDL Spec | v3.0.0 | WDL language specification |
| UluOps PDL Spec | v1.2.0 | PDL language specification |
| ADL JSON Schema | v1.15.0 | ADL validation schema |
| CDL JSON Schema | v2.0.0 | CDL validation schema |
| WDL JSON Schema | v3.0.0 | WDL validation schema |
| PDL JSON Schema | v1.2.0 | PDL validation schema |
| Failure Taxonomy Spec | v1.0.0 | Failure classification system |
| Failure Taxonomy Schema Defs | v1.0.0 | Taxonomy JSON Schema definitions |
| Registry API Spec | v0.5.1 | Definition storage and serving |
| Tracker API Spec | v1.1.0 | Execution tracking and analytics |
| Fingerprint Spec | v2.0.0 | Issue identity and correlation |
| Thinker Profile Spec | v0.1.0 | Cognitive lens agent specification |
| Cognitive Lens Library Spec | v0.3.0 | Thinker catalog and composition |
| Cognitive Operations Engineering Spec | v0.1.0 | Engineering methodology |
| Interleaved Workflow Compositions Spec | v0.3.0 | Multi-agent composition patterns |
| Domain Profile Schema | v1.0.0 | Domain profile validation |
| Recursive Appreciation Hypothesis | v1.4.0 | Self-evaluation convergence theory |
| UluOps Whitepaper | v2.1.0-draft | Theoretical foundation |
| UluOps Vision Synthesis | Feb 2026 | Unified platform vision |
| Patent Claims | v0.1.0 | IP claims and prior art analysis |

---

## 16. Design Decisions and Rationale

### DD-001: Four languages, not one — and the system is the invention

**Decision:** UDL is a family of four separate languages rather than a single monolithic schema with type discriminators.

**Rationale:** A single schema would be technically possible (a large YAML with a `type` discriminator selecting between agent/command/workflow/pipeline). But this would conflate separation of concerns. An agent author should not need to understand pipeline governance. A pipeline author should not need to understand scoring criteria. Four separate schemas enforce cognitive separation — you only learn the schema for the layer you're working at. This also enables independent versioning: ADL can evolve rapidly (15 major versions) while PDL evolves slowly (1 major version), because they are not coupled.

**Patent framing.** The novel contribution is the *coordinated system* of four languages, not any individual language in isolation. Each language is useful on its own — ADL can define agents without CDL, WDL, or PDL. But the system-level properties that constitute the core invention emerge only from their coordination:

- **Deterministic rendering + versioned registry + fingerprint-based tracking** enables regression detection. No single language provides this; it requires ADL (structured definition) + the rendering pipeline (deterministic prompt) + the tracker (cross-run correlation). Remove any component and the property disappears.
- **Quality-gated orchestration + score-based iteration** enables convergent quality improvement. This requires ADL (scoring criteria) + CDL (iteration loops) + WDL (quality gates). An ADL agent alone can score; it cannot iterate or gate.
- **Domain portability + cognitive parallax measurement** enables cross-framework analysis. This requires ADL (agent definitions with domain profiles) + WDL (interleaved compositions) + the failure taxonomy (classification) + the tracker (overlap measurement). No single language layer provides this.

The patent claims should reference the four-language system as the claimed invention, with individual language features as dependent claims. This is analogous to how TCP/IP is patented as a protocol suite — the novel contribution is the coordinated stack, not any single layer. An examiner evaluating ADL alone would find prior art in prompt templates; evaluating CDL alone would find prior art in CI/CD configurations; evaluating WDL alone would find prior art in workflow engines. The system of four languages with strict composition rules, shared structural patterns, a deterministic rendering pipeline, and integrated tracking has no prior art.

### DD-002: YAML over JSON, GraphQL, or custom DSL

**Decision:** All definitions are authored in YAML.

**Rationale:** JSON lacks comments and is noisy for human authoring. GraphQL is a query language, not a definition language. A custom DSL would require custom parsers, custom editors, and custom tooling — a prohibitive adoption barrier. YAML is the established standard for declarative configuration (Kubernetes, GitHub Actions, Docker Compose, Ansible), has universal tooling support, and supports comments for documentation. JSON Schema provides machine validation without sacrificing YAML's readability.

### DD-003: Strict composition hierarchy, not free-form graphs

**Decision:** Definitions compose strictly upward through a four-level hierarchy with no lateral or downward references.

**Rationale:** Free-form composition (any definition can reference any other) would be more flexible but would create dependency cycles, make definitions impossible to understand in isolation, and prevent independent versioning. The strict hierarchy trades flexibility for predictability, testability, and comprehensibility. The hierarchy also maps cleanly to organizational concerns: individual contributors author agents, team leads configure commands and workflows, platform engineers configure pipelines.

### DD-004: Rendering as a separate, versioned stage

**Decision:** Definition-to-prompt rendering is performed by versioned translators that are independent of both definition versions and language versions.

**Rationale:** Coupling rendering to definitions would mean that improving prompt quality requires modifying every definition. Coupling rendering to language versions would mean that translator improvements require a language version bump. Independent translator versioning means that a rendering improvement (better prompt structure, additional context injection, optimized token usage) benefits all existing definitions immediately, and the improvement is traceable through the translator version recorded in execution results.

### DD-005: The interface block is universal

**Decision:** All four languages share an identical core interface block structure.

**Rationale:** The interface block is the "face" of a definition — what the registry stores, what search returns, what the CLI displays. If each language had a different identity structure, every consumer (registry, CLI, dashboard, search) would need language-specific logic for basic operations. A universal interface block means that listing, searching, displaying, and comparing definitions works identically regardless of type.

### DD-006: Domain portability through profiles, not schema variants

**Decision:** Domain-specific vocabulary is injected through profiles at rendering time, not through language-level schema differences.

**Rationale:** The alternative — separate schemas for software agents, legal agents, financial agents — would fracture the ecosystem. A software ADL and a legal ADL would be incompatible, requiring separate tooling, separate registries, and separate training. By keeping schemas domain-neutral and injecting domain vocabulary at rendering time, the entire infrastructure (registry, tracker, SDK, compositions) works across all domains without modification.

---

## Appendix A: Glossary

| Term | Definition |
|------|------------|
| **ADL** | Agent Definition Language — schema for atomic AI agents |
| **CDL** | Command Definition Language — schema for single-agent invocation context |
| **WDL** | Workflow Definition Language — schema for multi-phase orchestrations |
| **PDL** | Pipeline Definition Language — schema for multi-workflow orchestrations with CI/CD integration |
| **UDL** | Unified Definition Language — the family name for ADL, CDL, WDL, and PDL collectively |
| **UDL Family Version** | The coordinated version of the four-language system; increments when shared architectural invariants change, independent of individual language versions |
| **Definition** | A YAML document conforming to one of the four UDL schemas |
| **Rendering** | Deterministic transformation of a definition into a runtime prompt |
| **Translation** | Template-based rendering using Nunjucks translators |
| **Translator** | A versioned Nunjucks template set that performs rendering |
| **Registry** | Platform service that stores, versions, and serves definitions |
| **Tracker** | Platform service that records execution results and provides analytics |
| **Fingerprint** | SHA-256 based identifier for correlating findings across runs |
| **Fork** | A derivative copy of a definition, maintaining lineage |
| **Lineage** | The graph of version and fork relationships for a definition family |
| **Domain Profile** | A YAML file providing domain-specific vocabulary for rendering |
| **Failure Taxonomy** | The four-domain (STR/SEM/PRA/EPI) classification system for findings |
| **Cognitive Parallax** | The empirically validated phenomenon that different cognitive lenses produce near-zero overlapping findings |
| **Recursive Appreciation** | The methodology by which AI systems evaluate and improve their own definitions |
| **Pure Function Property** | The guarantee that the same rendering input always produces the same output |

---

## Appendix B: Version History of Each Language

### ADL (Agent Definition Language)

| Version | Date | Milestone |
|---------|------|-----------|
| VDL 1.0.0 | 2026-01-11 | Initial validator-only schema |
| ADL 1.0.0 | 2026-01-11 | Unified agent schema (validators + executors) |
| ADL 1.2.0 | 2026-01-13 | Mission block, knowledge base, extended verification |
| ADL 1.5.0 | 2026-01-23 | Vocabulary rationale, explicit prohibitions, success criteria |
| ADL 1.6.0 | 2026-02-07 | Domain profiles, analyst/generator agent types |
| ADL 1.7.0 | — | Key definitions, domain taxonomy, epistemic limitations |
| ADL 1.8.0 | — | Explorer agent type |
| ADL 1.9.0 | — | Forecaster agent type |
| ADL 1.10.0 | 2026-03-02 | Schema consolidation |
| ADL 1.13.0 | — | Epistemic nature block, thinker profile integration |
| ADL 1.15.0 | 2026-04-07 | Current version |

### CDL (Command Definition Language)

| Version | Date | Milestone |
|---------|------|-----------|
| CDL 1.0.0 | 2026-01-13 | Initial specification (4 command types) |
| CDL 1.3.0 | — | Fixer type, iteration refinement |
| CDL 2.0.0 | 2026-03-01 | Collapsed to 3 types (agent/utility/custom), agent-type-agnostic |

### WDL (Workflow Definition Language)

| Version | Date | Milestone |
|---------|------|-----------|
| WDL 1.0.0 | 2026-01-14 | Initial specification |
| WDL 2.0.0 | — | Phase-based restructure |
| WDL 3.0.0 | 2026-03-26 | Runtime-only content, lean phases, 250-line target |

### PDL (Pipeline Definition Language)

| Version | Date | Milestone |
|---------|------|-----------|
| PDL 1.0.0 | 2026-01-14 | Initial specification |
| PDL 1.1.0 | 2026-03-07 | Postflight block, tracker persistence integration |
| PDL 1.2.0 | 2026-03-07 | Direct agent invocation via `agents` execution primitive |

---

## Changelog

### v0.1.0 — April 13, 2026

- Initial UDL specification
- Documented the four-language composition hierarchy
- Formalized shared design principles (8 principles)
- Codified shared structural patterns (interface block, root object, naming, domains, file extensions, schema URLs)
- Documented the rendering pipeline with pure function property
- Specified platform service integration (Registry, Tracker, SDK, MCP)
- Documented failure taxonomy relationship
- Documented domain profile system
- Described composition patterns (direct, sequential, parallel, interleaved, cognitive lens)
- Documented versioning and evolution strategy
- **UDL Family Versioning** (§12.5): Formal family version (0.1.0) with bump rules, family-vs-language version independence, patent relevance for system-level claims
- Stated licensing and open standard strategy
- **Open vs. Patented Boundary Table** (§13.1): Explicit delineation of open specification (CC BY 4.0), patented innovations, and proprietary implementation with rationale per component
- **Boundary Principle** (§13.2): "The specification is the grammar; the innovations are the semantics and the system"
- Enumerated 8 novel contributions
- **Sharpened prior art distinctions** for Novel Contributions #5 (quality-gated DAG — distinguished from Airflow/Prefect/Dagster via score-threshold gates on AI judgment output, schema-enforced thresholds) and #8 (three-dimensional compositions — distinguished from CrewAI/AutoGen/LangGraph via structurally distinct agent families with empirically validated near-zero finding overlap)
- Recorded 6 design decisions with rationale
- **DD-001 patent framing**: Formalized the "coordinated system" argument — the four-language system is the invention, individual languages are dependent claims; TCP/IP protocol suite analogy; examiner-facing prior art analysis per individual layer
- Complete glossary and version history appendices

---

*Prepared by Alex Self, Ulu Labs Inc. For internal use, patent preparation, and eventual public specification release.*
