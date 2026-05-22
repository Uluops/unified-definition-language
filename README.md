# Unified Definition Language

A family of four declarative definition languages for AI agent operations. Each level addresses a single operational concern, forming a strict composition hierarchy from atomic cognitive operations to governance orchestrations.

## The Problem

Using AI for expert judgment means writing prompts directly. A single prompt conflates what the agent should evaluate, how it should score, what constitutes success, and how it should present results. Changing any one concern requires rewriting the entire prompt. There is no versioning, no composition, no reproducibility, and no measurement.

## The Solution

Four structured definition languages, each at a distinct level of abstraction:

```
Level 4 — PDL (Pipeline)    Governance: triggers, approvals, rollback
Level 3 — WDL (Workflow)    Orchestration: phases, quality gates, aggregation
Level 2 — CDL (Command)     Invocation: model selection, preflight, iteration
Level 1 — ADL (Agent)       Cognition: domain knowledge, scoring, decisions
```

### ADL — Agent Definition Language

The content layer. Each agent definition specifies evaluation criteria, scoring methodology, decision logic, output schemas, edge cases, and domain context. ADL encodes what an agent knows about its domain concern.

### CDL — Command Definition Language

The execution layer. Each command wraps an agent reference with execution parameters — model selection, preflight checks, iteration controls, and overrides. CDL specifies how to invoke an agent without modifying the agent itself.

### WDL — Workflow Definition Language

The coordination layer. Each workflow composes agents and commands into phases with declared dependencies, quality gates, and aggregation logic. WDL orchestrates multi-agent operations with gates on continuous quality scores, not binary exit codes.

### PDL — Pipeline Definition Language

The governance layer. Each pipeline composes workflows into stages with triggers, approvals, rollback behaviors, and external notifications. PDL specifies when and under what conditions operations execute.

## Key Properties

**Deterministic rendering.** A pure-function rendering pipeline transforms structured YAML definitions into runtime prompts through versioned translator templates. The same definition always produces the same prompt, enabling meaningful regression detection and result attribution.

**Strict composition rules.** Each level composes only the level directly below it. Pipelines compose workflows. Workflows compose commands and agents. Commands wrap agents. Agents are atomic. No level reaches across the hierarchy.

**Three-axis independent versioning.** Definition content, language schemas, and rendering templates evolve independently. A translator improvement benefits all existing definitions without requiring definition changes.

**Domain portability.** A profile substitution system injects domain-specific vocabulary at rendering time rather than through design-time schema specialization. The same infrastructure serves agents across software engineering, security, legal compliance, documentation, and other domains.

**Structured agent type system.** A polymorphic type system supporting validators, executors, analysts, generators, explorers, and forecasters — each with a distinct output schema and decision vocabulary.

## Related

- [Agents & Pipelines](https://github.com/aself101/agents-and-pipelines) — Agent, command, and pipeline definitions built with UDL
- [Failure Taxonomy](https://github.com/Uluops/failure-taxonomy) — The four-domain classification system integrated into agent output schemas
- [Cognitive Lens Library](https://github.com/Uluops/cognitive-lens-library) — Thinker profiles that inform cognitive lens agent definitions
- [UluOps](https://uluops.ai) — The platform infrastructure for registry, tracking, and execution

## License

This work is licensed under [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/). Copyright 2025-2026 Ulu Labs Inc.
