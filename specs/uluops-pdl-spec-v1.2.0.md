# UluOps Pipeline Definition Language (PDL) Specification

**Version:** 1.2.0
**Status:** Draft
**Created:** 2026-01-14
**Updated:** 2026-03-07

---

## Table of Contents

- [Overview](#overview)
- [Design Principles](#design-principles)
- [Relationship to Other Definition Languages](#relationship-to-other-definition-languages)
- [Schema Structure](#schema-structure)
  - [CI/CD Pipeline](#cicd-pipeline)
  - [Multi-Workflow Orchestration](#multi-workflow-orchestration)
  - [Hybrid Pipeline (Workflows + Commands)](#hybrid-pipeline-workflows--commands)
- [Detailed Field Reference](#detailed-field-reference)
  - [Interface Block](#interface-block)
  - [Triggers Block](#triggers-block)
  - [Environment Block](#environment-block)
  - [Stages Block](#stages-block)
  - [Approvals Block](#approvals-block)
  - [Rollback Block](#rollback-block)
  - [Notifications Block](#notifications-block)
  - [Artifacts Block](#artifacts-block)
  - [State Block](#state-block)
  - [Postflight Block](#postflight-block)
- [Stage Dependency Graph](#stage-dependency-graph)
- [Trigger Event Types](#trigger-event-types)
- [Approval Workflows](#approval-workflows)
- [Rollback Strategies](#rollback-strategies)
- [Data Flow Between Stages](#data-flow-between-stages)
- [Composition Rules](#composition-rules)
- [Runtime Behavior](#runtime-behavior)
- [Examples](#examples)
- [JSON Schema Reference](#json-schema-reference)
- [Revision History](#revision-history)
- [Appendix: Type Definitions](#appendix-type-definitions)

---

## Overview

The **Pipeline Definition Language (PDL)** is a YAML-based specification for defining multi-workflow orchestrations in the UluOps validation framework. Pipelines compose workflows and commands into **stages** with explicit dependencies, triggers, approvals, rollback strategies, and external notifications.

### Key Characteristics

| Aspect | Description |
|--------|-------------|
| **Format** | YAML with JSON Schema validation |
| **File Extension** | `.pipeline.yaml` |
| **Schema URL** | `https://uluops.ai/schemas/pdl/v1.0.0/pipeline.json` |
| **Scope** | Multi-workflow/command orchestration with CI/CD integration |

### What Pipelines Do

Pipelines serve as the **top-level orchestration layer**:

1. **Organize workflows/commands into stages** with explicit dependencies
2. **Trigger execution** based on events (git push, PR, schedule, webhook, manual)
3. **Enforce approval gates** requiring human authorization before critical stages
4. **Manage environment** (variables, secrets) scoped to pipeline or stage
5. **Handle failures** with automatic rollback strategies
6. **Notify external systems** (Slack, email, GitHub status)
7. **Preserve artifacts** with configurable retention policies
8. **Maintain state** across pipeline runs

### What Pipelines Do NOT Do

Pipelines delegate to lower layers:

- **No validation/execution logic** — that belongs in agents (ADL)
- **No single-agent invocation context** — that belongs in commands (CDL)
- **No multi-command phase orchestration** — that belongs in workflows (WDL)
- **No scoring or criteria** — that belongs in agents (ADL)

### Pipeline vs Workflow

| Aspect | Workflow (WDL) | Pipeline (PDL) |
|--------|----------------|----------------|
| **Contains** | Commands in phases | Workflows and/or commands in stages |
| **Triggers** | None (invoked by caller) | Git events, webhooks, schedules, manual |
| **Approvals** | None | Human gates before stages |
| **Rollback** | None | Automatic failure recovery |
| **Notifications** | None | External system integration |
| **Artifacts** | Output files | Preserved files with retention |
| **State** | Per-run only | Persisted across runs |
| **Environment** | Inherited from caller | Own variables and secrets |

---

## Design Principles

### 1. Workflow/Command Orchestration, Not Agent Orchestration

Pipelines compose workflows and commands, not agents directly. Agents must be wrapped in commands, which can then be referenced in workflows or directly in pipeline stages.

```yaml
# ✓ Correct: Reference workflows and commands
stages:
  - id: validate
    workflows:
      - ship@1.0.0
      
  - id: quick-check
    commands:
      - validate@1.0.0

# ✗ Wrong: Reference agents directly
stages:
  - id: validate
    agents:
      - code-validator
```

### 2. Event-Driven Execution

Pipelines define when they should run via triggers. Unlike workflows which are invoked programmatically, pipelines respond to external events.

```yaml
triggers:
  - type: git_push
    branches: [main, master]
    
  - type: schedule
    cron: "0 2 * * *"
```

### 3. Human-in-the-Loop Approvals

Critical stages can require human approval before execution. This enables safe deployment workflows with mandatory review checkpoints.

```yaml
stages:
  - id: deploy-prod
    approval:
      required: true
      approvers: ["@release-team", "@security-team"]
      timeout_hours: 24
```

### 4. Fail-Safe with Rollback

Pipelines can define automatic rollback strategies to recover from failures. This ensures production stability even when deployments fail.

```yaml
rollback:
  enabled: true
  strategy: revert_to_last_success
  triggers:
    - condition: "stages.deploy.failed"
      action: rollback_deploy
```

### 5. Declarative Stage Dependencies

Stages declare explicit dependencies. The runtime builds a DAG (Directed Acyclic Graph) and determines execution order, parallelizing where possible.

```yaml
stages:
  - id: build
    # No depends_on = root stage
    
  - id: test
    depends_on: [build]
    
  - id: deploy
    depends_on: [test]
```

### 6. Consistent Interface Pattern

Pipelines use the same `interface` block structure as Agents (ADL), Commands (CDL), and Workflows (WDL) for consistency across the definition language family.

```yaml
pipeline:
  interface:
    name: release-pipeline
    version: 1.0.0
    displayName: Release Pipeline
    description: Complete CI/CD pipeline from validation to production deployment
    domain: software
```

---

## Relationship to Other Definition Languages

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
                    PDL (Pipeline Definition)  ◄── YOU ARE HERE
```

### Composition Hierarchy

| Level | Definition | Contains | Example |
|-------|------------|----------|---------|
| Agent (ADL) | Atomic validation/execution unit | Scoring, criteria, tasks | `code-validator.agent.yaml` |
| Command (CDL) | Single agent + execution context | Preflight, overrides, output | `validate.command.yaml` |
| Workflow (WDL) | Multiple commands + phases | Gates, aggregation, context | `ship.workflow.yaml` |
| **Pipeline (PDL)** | **Workflows/commands + stages** | **Triggers, approvals, rollback** | **`release.pipeline.yaml`** |

### Composition Rules

```
Pipeline (PDL)
    ├── Can reference: Workflows (WDL), Commands (CDL)
    ├── Can contain: Inline shell commands (steps)
    └── Cannot reference: Agents (ADL) directly

Workflow (WDL)
    ├── Can reference: Commands (CDL)
    └── Cannot reference: Agents, Pipelines

Command (CDL)
    ├── Can reference: Agents (ADL)
    └── Cannot reference: Commands, Workflows, Pipelines

Agent (ADL)
    └── Cannot reference: anything (atomic unit)
```

### Why Pipelines Exist

Pipelines solve the **CI/CD integration problem**:

- **Workflows** define *which commands* to run and *in what order*
- **Pipelines** define *when workflows run*, *who approves them*, *what happens on failure*

Without pipelines, CI/CD systems would need custom scripts to handle triggers, approvals, rollbacks, and notifications.

---

## Schema Structure

### CI/CD Pipeline

A complete CI/CD pipeline with validation, testing, and deployment:

```yaml
pipeline:
  interface:
    name: release-pipeline
    version: 1.0.0
    displayName: Release Pipeline
    description: Complete CI/CD pipeline from development validation to production deployment
    domain: software
    subdomain: ci-cd
    tags: [release, ci-cd, deployment]

  # ───────────────────────────────────────────────────────────────────
  # TRIGGERS
  # ───────────────────────────────────────────────────────────────────
  triggers:
    - type: git_push
      branches: [main, master]
      paths:
        include: ["src/**", "package.json"]
        exclude: ["**/*.md", "docs/**"]
        
    - type: pull_request
      actions: [opened, synchronize, reopened]
      branches: [main, master]
      
    - type: schedule
      cron: "0 2 * * 1"  # Weekly Monday 2 AM
      timezone: "UTC"
      
    - type: manual
      allowed: true
      parameters:
        - name: skip_tests
          type: boolean
          default: false
        - name: environment
          type: string
          options: [staging, production]
          default: staging

  # ───────────────────────────────────────────────────────────────────
  # ENVIRONMENT
  # ───────────────────────────────────────────────────────────────────
  environment:
    variables:
      - name: NODE_ENV
        value: "production"
        
      - name: CI
        value: "true"
        
      - name: VALIDATION_THRESHOLD
        value: "{{ params.strict ? 85 : 70 }}"
        
    secrets:
      - name: NPM_TOKEN
        required: true
        stages: [build, publish]
        
      - name: DEPLOY_KEY
        required: true
        stages: [deploy-staging, deploy-prod]
        
      - name: SLACK_WEBHOOK
        required: false
        stages: [notify]

  # ───────────────────────────────────────────────────────────────────
  # STAGES
  # ───────────────────────────────────────────────────────────────────
  stages:
    # Stage 1: Pre-flight checks
    - id: preflight
      name: Pre-Flight Checks
      parallel: true
      
      steps:
        - name: Install dependencies
          command: npm ci
          
        - name: Verify clean git state
          command: git status --porcelain
          expect_empty: true
          
        - name: Check Node version
          command: node --version
          expect_match: "^v2[02]\\."
          
      gate:
        on_failure: abort
        
    # Stage 2: Core validation (workflow)
    - id: validate
      name: Core Validation
      depends_on: [preflight]
      
      workflows:
        - ref: ship@>=1.0.0
          args:
            target: "."
            strict: "{{ params.strict || false }}"
            
      gate:
        threshold: 70
        on_failure: abort
        
    # Stage 3: Security audit (workflow + steps)
    - id: security
      name: Security Review
      depends_on: [validate]
      parallel: false
      
      workflows:
        - ref: security-audit@>=1.0.0
          args:
            target: "."
            
      steps:
        - name: Dependency audit
          command: npm audit --audit-level=high
          continue_on_error: false
          
        - name: License check
          command: npx license-checker --onlyAllow "MIT;Apache-2.0;BSD-3-Clause;ISC"
          
      gate:
        threshold: 85
        on_failure: abort
        
    # Stage 4: Integration tests (conditional)
    - id: integration
      name: Integration Testing
      depends_on: [security]
      condition: "!params.skip_tests && context.has_integration_tests"
      timeout: 900000  # 15 minutes
      
      steps:
        - name: Start test database
          command: docker-compose up -d test-db
          
        - name: Run integration tests
          command: npm run test:integration
          env:
            DATABASE_URL: "postgres://localhost:5432/test"
            
        - name: Stop test database
          command: docker-compose down
          always_run: true
          
      gate:
        on_failure: warn
        
    # Stage 5: Build
    - id: build
      name: Build
      depends_on: [security]  # Can run parallel to integration
      
      steps:
        - name: Build production bundle
          command: npm run build
          
        - name: Generate types
          command: npm run build:types
          
      artifacts:
        - name: dist
          path: "./dist/**/*"
          
    # Stage 6: Deploy to staging
    - id: deploy-staging
      name: Deploy to Staging
      depends_on: [build, integration]
      condition: "trigger.type == 'git_push' || params.environment == 'staging'"
      
      steps:
        - name: Deploy to staging
          command: npm run deploy:staging
          env:
            DEPLOY_KEY: "{{ secrets.DEPLOY_KEY }}"
            
        - name: Smoke test
          command: npm run test:smoke -- --env=staging
          
      gate:
        on_failure: abort
        
    # Stage 7: Deploy to production (requires approval)
    - id: deploy-prod
      name: Deploy to Production
      depends_on: [deploy-staging]
      condition: "trigger.branch == 'main' && params.environment == 'production'"
      
      approval:
        required: true
        approvers: ["@release-team"]
        min_approvals: 1
        timeout_hours: 24
        auto_reject_on_change: true
        
      steps:
        - name: Deploy to production
          command: npm run deploy:prod
          env:
            DEPLOY_KEY: "{{ secrets.DEPLOY_KEY }}"
            
        - name: Verify deployment
          command: npm run test:smoke -- --env=production
          retries: 3
          retry_delay: 30000
          
      gate:
        on_failure: abort

  # ───────────────────────────────────────────────────────────────────
  # ROLLBACK
  # ───────────────────────────────────────────────────────────────────
  rollback:
    enabled: true
    strategy: revert_to_last_success
    
    triggers:
      - condition: "stages.deploy-prod.failed"
        action: rollback_production
        
      - condition: "stages.deploy-staging.failed && stages.deploy-prod.started"
        action: rollback_all
        
    actions:
      rollback_production:
        steps:
          - name: Revert production deployment
            command: npm run deploy:prod:rollback
          - name: Verify rollback
            command: npm run test:smoke -- --env=production
            
      rollback_all:
        steps:
          - name: Revert staging
            command: npm run deploy:staging:rollback
          - name: Revert production
            command: npm run deploy:prod:rollback
            
    notify_on_rollback: true

  # ───────────────────────────────────────────────────────────────────
  # NOTIFICATIONS
  # ───────────────────────────────────────────────────────────────────
  notifications:
    channels:
      - type: slack
        webhook: "{{ secrets.SLACK_WEBHOOK }}"
        events: [started, stage_complete, success, failure, rollback]
        channel: "#deployments"
        
      - type: email
        recipients: ["team@example.com"]
        events: [failure, rollback]
        
      - type: github_status
        enabled: true
        context: "uluops/release-pipeline"
        
    templates:
      started: |
        🚀 Pipeline started: {{ pipeline.name }} v{{ pipeline.version }}
        Trigger: {{ trigger.type }} ({{ trigger.ref || trigger.branch }})
        
      success: |
        ✅ Pipeline completed successfully
        Duration: {{ pipeline.duration | duration }}
        Stages: {{ stages | length }} completed
        
      failure: |
        ❌ Pipeline failed at stage: {{ failed_stage.name }}
        Error: {{ failed_stage.error }}
        Duration: {{ pipeline.duration | duration }}
        
      rollback: |
        ⚠️ Rollback triggered
        Stage: {{ failed_stage.name }}
        Action: {{ rollback.action }}

  # ───────────────────────────────────────────────────────────────────
  # ARTIFACTS
  # ───────────────────────────────────────────────────────────────────
  artifacts:
    - name: build_output
      path: "./dist/**/*"
      retention_days: 30
      compression: gzip
      
    - name: test_results
      path: "./coverage/**/*"
      retention_days: 7
      
    - name: validation_reports
      path: "./**/*-features-list.md"
      retention_days: 90
      
    - name: logs
      path: "./.uluops/logs/**/*"
      retention_days: 14

  # ───────────────────────────────────────────────────────────────────
  # STATE
  # ───────────────────────────────────────────────────────────────────
  state:
    persistence: true
    ttl: "30d"
    
    tracked:
      - last_successful_deploy
      - last_rollback
      - deployment_count
      - failure_count
```

### Multi-Workflow Orchestration

A simpler pipeline that orchestrates multiple validation workflows:

```yaml
pipeline:
  interface:
    name: full-validation
    version: 1.0.0
    displayName: Full Validation Pipeline
    description: Comprehensive validation pipeline running multiple specialized workflows
    domain: software

  triggers:
    - type: pull_request
      actions: [opened, synchronize]
      
    - type: manual
      allowed: true

  stages:
    - id: pre-implementation
      name: Pre-Implementation Review
      workflows:
        - ref: pre-implementation@>=1.0.0
          args:
            target: "."
      gate:
        threshold: 70
        on_failure: warn
        
    - id: post-implementation
      name: Post-Implementation Validation
      depends_on: [pre-implementation]
      workflows:
        - ref: post-implementation@>=1.0.0
          args:
            target: "."
      gate:
        threshold: 75
        on_failure: abort
        
    - id: ship-gate
      name: Ship Readiness Gate
      depends_on: [post-implementation]
      workflows:
        - ref: ship@>=1.0.0
          args:
            target: "."
            strict: true
      gate:
        threshold: 80
        on_failure: abort

  notifications:
    channels:
      - type: github_status
        enabled: true
        context: "uluops/validation"
```

### Hybrid Pipeline (Workflows + Commands)

A pipeline that mixes workflows with direct command invocations:

```yaml
pipeline:
  interface:
    name: audit-and-fix
    version: 1.0.0
    displayName: Audit and Fix Pipeline
    description: Run validation audit then attempt automatic fixes
    domain: software

  triggers:
    - type: schedule
      cron: "0 3 * * *"  # Daily at 3 AM
      
    - type: manual
      allowed: true
      parameters:
        - name: auto_fix
          type: boolean
          default: true

  stages:
    # Stage 1: Run full audit workflow
    - id: audit
      name: Full Audit
      workflows:
        - ref: ship@>=1.0.0
          args:
            target: "."
      gate:
        threshold: 0  # Always continue (we want to find issues)
        on_failure: warn
        
    # Stage 2: Run individual fix commands (conditional)
    - id: auto-fix
      name: Automatic Fixes
      depends_on: [audit]
      condition: "params.auto_fix && stages.audit.score < 90"
      
      commands:
        - ref: lint-fix@>=1.0.0
          args:
            target: "."
          timeout: 300000
          
        - ref: type-fix@>=1.0.0
          args:
            target: "."
          condition: "context.typescript_detected"
          
      sequential: true  # Run commands in order
      
    # Stage 3: Re-validate after fixes
    - id: verify
      name: Verify Fixes
      depends_on: [auto-fix]
      condition: "stages.auto-fix.completed"
      
      workflows:
        - ref: ship@>=1.0.0
          args:
            target: "."
            
      gate:
        threshold: 80
        on_failure: warn

  artifacts:
    - name: audit_before
      path: "./.uluops/audit-before.json"
      source: "stages.audit.output"
      
    - name: audit_after
      path: "./.uluops/audit-after.json"
      source: "stages.verify.output"
      condition: "stages.verify.completed"
```

---

## Detailed Field Reference

### Interface Block

The interface block identifies the pipeline and is consistent with ADL, CDL, and WDL.

```yaml
interface:
  name: release-pipeline          # Required: kebab-case identifier
  version: 1.0.0                  # Required: semantic version
  displayName: Release Pipeline   # Required: human-readable (3-50 chars)
  description: Full CI/CD...      # Required: purpose description (20-500 chars)
  domain: software                # Required: primary domain
  subdomain: ci-cd                # Optional: domain refinement
  tags: [release, deploy]         # Optional: searchable tags
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Kebab-case identifier (`^[a-z][a-z0-9]*(-[a-z0-9]+)*$`) |
| `version` | string | Yes | Semantic version (`^\d+\.\d+\.\d+$`) |
| `displayName` | string | Yes | Human-readable name (3-50 characters) |
| `description` | string | Yes | Pipeline purpose and scope (20-500 characters) |
| `domain` | Domain | Yes | Primary domain classification |
| `subdomain` | string | No | Domain refinement (kebab-case) |
| `tags` | string[] | No | Searchable discovery tags |

### Triggers Block

Triggers define when the pipeline executes.

```yaml
triggers:
  - type: git_push
    branches: [main, master]
    paths:
      include: ["src/**"]
      exclude: ["**/*.md"]
      
  - type: pull_request
    actions: [opened, synchronize, reopened]
    branches: [main]
    
  - type: schedule
    cron: "0 2 * * *"
    timezone: "UTC"
    
  - type: webhook
    event: custom_event
    secret: "{{ secrets.WEBHOOK_SECRET }}"
    
  - type: manual
    allowed: true
    parameters:
      - name: environment
        type: string
        options: [staging, production]
        default: staging
```

#### Trigger Types

| Type | Description | Key Fields |
|------|-------------|------------|
| `git_push` | Triggered on push to branch | `branches`, `tags`, `paths` |
| `pull_request` | Triggered on PR events | `actions`, `branches`, `paths` |
| `schedule` | Cron-based scheduling | `cron`, `timezone` |
| `webhook` | External HTTP webhook | `event`, `secret`, `payload_filter` |
| `manual` | Manual invocation | `allowed`, `parameters` |
| `pipeline` | Triggered by another pipeline | `pipeline`, `condition` |

#### git_push Trigger

```yaml
- type: git_push
  branches: [main, master]        # Branch patterns (glob supported)
  tags: ["v*"]                    # Tag patterns
  paths:
    include: ["src/**", "*.json"] # Files that trigger (glob)
    exclude: ["**/*.md"]          # Files that don't trigger
```

#### pull_request Trigger

```yaml
- type: pull_request
  actions:                        # PR actions that trigger
    - opened                      # PR opened
    - synchronize                 # New commits pushed
    - reopened                    # PR reopened
    - closed                      # PR closed (merged or not)
    - labeled                     # Label added
  branches: [main]                # Target branch filter
  paths:
    include: ["src/**"]
    exclude: ["docs/**"]
```

#### schedule Trigger

```yaml
- type: schedule
  cron: "0 2 * * 1-5"            # Cron expression (required)
  timezone: "America/New_York"    # Timezone (default: UTC)
  skip_if_running: true           # Don't start if previous still running
```

#### webhook Trigger

```yaml
- type: webhook
  event: deployment_requested     # Event name to match
  secret: "{{ secrets.HOOK }}"    # HMAC verification secret
  payload_filter: "$.env == 'production'"  # JSONPath filter
```

#### manual Trigger

```yaml
- type: manual
  allowed: true                   # Enable manual triggering
  parameters:                     # Input parameters
    - name: environment
      type: string
      options: [dev, staging, prod]
      default: staging
      required: true
    - name: skip_tests
      type: boolean
      default: false
    - name: version
      type: string
      pattern: "^v\\d+\\.\\d+\\.\\d+$"
```

### Environment Block

Environment variables and secrets for the pipeline.

```yaml
environment:
  variables:
    - name: NODE_ENV
      value: "production"
      
    - name: LOG_LEVEL
      value: "{{ trigger.type == 'manual' ? 'debug' : 'info' }}"
      
    - name: DEPLOY_TARGET
      value: "{{ params.environment || 'staging' }}"
      
  secrets:
    - name: NPM_TOKEN
      required: true
      stages: [build, publish]    # Stages that can access
      
    - name: DEPLOY_KEY
      required: true
      stages: [deploy-staging, deploy-prod]
      provider: vault             # Optional: secret provider
      path: "secret/deploy/key"   # Provider-specific path
```

| Field | Type | Description |
|-------|------|-------------|
| `variables` | Variable[] | Environment variables |
| `secrets` | Secret[] | Secret references |

#### Variable Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Variable name (uppercase recommended) |
| `value` | string | Yes | Value or template expression |
| `stages` | string[] | No | Limit to specific stages |

#### Secret Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Secret name |
| `required` | boolean | No | Fail if not available (default: false) |
| `stages` | string[] | No | Stages that can access |
| `provider` | string | No | Secret provider (vault, aws, etc.) |
| `path` | string | No | Provider-specific path |

### Stages Block

Stages are the primary execution units in a pipeline.

```yaml
stages:
  - id: build
    name: Build
    depends_on: []                # Root stage
    condition: "true"             # Always run
    timeout: 600000               # 10 minutes
    parallel: true                # Run steps in parallel
    
    # Workflow references
    workflows:
      - ref: build-workflow@1.0.0
        args:
          target: "."
          
    # Command references
    commands:
      - ref: validate@1.0.0
        args:
          strict: true

    # Direct agent references
    agents:
      - ref: code-validator
        model: sonnet
      - ref: security-analyst
        model: opus
        condition: "stage.preflight.passed"

    # Inline steps
    steps:
      - name: Install dependencies
        command: npm ci
        
      - name: Build
        command: npm run build
        
    # Gate configuration
    gate:
      threshold: 70
      on_failure: abort
      
    # Stage-level artifacts
    artifacts:
      - name: dist
        path: "./dist/**/*"
        
    # Approval (if required)
    approval:
      required: false
```

#### Stage Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | Yes | Unique stage identifier (kebab-case) |
| `name` | string | Yes | Human-readable stage name |
| `depends_on` | string[] | No | Stage dependencies (empty = root) |
| `condition` | string | No | Enable condition expression |
| `timeout` | number | No | Timeout in milliseconds |
| `parallel` | boolean | No | Run items in parallel (default: false) |
| `workflows` | WorkflowRef[] | No | Workflow references |
| `commands` | CommandRef[] | No | Command references |
| `agents` | AgentRef[] | No | Direct agent references (subagent_type identifiers) |
| `steps` | Step[] | No | Inline shell steps |
| `sequential` | boolean | No | Force sequential execution |
| `gate` | Gate | No | Pass/fail gate |
| `artifacts` | Artifact[] | No | Stage-specific artifacts |
| `approval` | Approval | No | Human approval requirement |
| `env` | Record | No | Stage-specific environment |

#### Workflow Reference

```yaml
workflows:
  - ref: ship@>=1.0.0             # Version requirement
    args:                         # Arguments to pass
      target: "."
      strict: true
    condition: "context.has_src"  # Conditional execution
    timeout: 300000               # Override timeout
```

#### Command Reference

```yaml
commands:
  - ref: validate@latest
    args:
      target: "."
    condition: "true"
    timeout: 120000
```

#### Agent Reference

Direct agent references invoke agents by their `subagent_type` identifier without CDL indirection. Unlike commands and workflows which use `name@version` format, agent refs use bare kebab-case identifiers since agents are resolved by type, not version.

```yaml
agents:
  - ref: dx-validator               # subagent_type identifier (no version)
    model: sonnet                   # Optional: model alias override
    args:                           # Optional: arguments to pass
      target: "."
    condition: "true"               # Optional: conditional execution
    timeout: 300000                 # Optional: timeout in milliseconds
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `ref` | string | Yes | Agent subagent_type identifier (kebab-case, no version) |
| `model` | string | No | Model alias override: `sonnet`, `opus`, or `haiku` |
| `args` | object | No | Arguments to pass to the agent |
| `condition` | string | No | Conditional execution expression |
| `timeout` | number | No | Agent timeout in milliseconds (minimum 1000) |

**Resolution chain:** `agents.ref` maps directly to the Agent tool's `subagent_type` parameter. No CDL lookup is needed — the executor passes the ref value straight through.

**When to use `agents` vs `commands`:**
- Use `agents` when you want direct, lightweight agent invocation without CDL indirection
- Use `commands` when you need the CDL layer's pre-flight checks, argument validation, or output formatting
- Both produce the same result (an agent execution), but `agents` is more direct

#### Inline Step

```yaml
steps:
  - name: Install dependencies    # Required: step name
    command: npm ci               # Required: shell command
    working_dir: "./packages/core"  # Optional: working directory
    env:                          # Optional: additional env vars
      NODE_ENV: "development"
    timeout: 60000                # Optional: step timeout
    continue_on_error: false      # Optional: don't fail stage
    always_run: false             # Optional: run even if prior failed
    retries: 0                    # Optional: retry count
    retry_delay: 5000             # Optional: delay between retries
    expect_empty: false           # Optional: expect no output
    expect_match: ""              # Optional: regex output match
```

#### Gate Configuration

```yaml
gate:
  threshold: 70                   # Score threshold (workflows)
  aggregate: min                  # Aggregation: min, max, average
  on_failure: abort               # abort, warn, skip
  on_success: continue            # continue, skip_remaining
```

### Approvals Block

Human approval requirements for stages.

```yaml
approval:
  required: true
  approvers:                      # Who can approve
    - "@release-team"             # Team mention
    - "user@example.com"          # Individual email
    - "github:octocat"            # Platform-specific
  min_approvals: 2                # Minimum required
  timeout_hours: 24               # Auto-reject after
  auto_reject_on_change: true     # Reject if source changes
  allow_self_approval: false      # Triggerer can't approve
  
  notification:
    channels: [slack, email]
    message: "Deployment to {{ params.environment }} requires approval"
```

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `required` | boolean | false | Whether approval is required |
| `approvers` | string[] | [] | List of approvers |
| `min_approvals` | number | 1 | Minimum approvals needed |
| `timeout_hours` | number | 72 | Hours before auto-reject |
| `auto_reject_on_change` | boolean | true | Reject if code changes |
| `allow_self_approval` | boolean | false | Allow triggerer to approve |
| `notification` | Notification | - | Approval request notification |

### Rollback Block

Automatic failure recovery configuration.

```yaml
rollback:
  enabled: true
  strategy: revert_to_last_success  # Strategy type
  
  triggers:
    - condition: "stages.deploy-prod.failed"
      action: rollback_production
      
    - condition: "stages.verify.score < 50"
      action: rollback_and_notify
      
  actions:
    rollback_production:
      steps:
        - name: Revert deployment
          command: kubectl rollout undo deployment/app
        - name: Verify
          command: npm run test:smoke
      timeout: 300000
      
    rollback_and_notify:
      steps:
        - name: Revert
          command: npm run deploy:rollback
      notify: true
      
  preserve_logs: true
  notify_on_rollback: true
  max_rollback_attempts: 3
```

#### Rollback Strategies

| Strategy | Description |
|----------|-------------|
| `revert_to_last_success` | Restore last successful deployment |
| `revert_to_previous` | Restore immediate previous state |
| `custom` | Execute custom rollback actions |
| `none` | No automatic rollback |

### Notifications Block

External system notification configuration.

```yaml
notifications:
  channels:
    - type: slack
      webhook: "{{ secrets.SLACK_WEBHOOK }}"
      channel: "#deployments"
      events: [started, success, failure, rollback]
      
    - type: email
      recipients:
        - "team@example.com"
        - "{{ trigger.author_email }}"
      events: [failure]
      
    - type: github_status
      enabled: true
      context: "uluops/ci"
      
    - type: webhook
      url: "https://api.example.com/notify"
      method: POST
      headers:
        Authorization: "Bearer {{ secrets.API_TOKEN }}"
      events: [success, failure]
      
  templates:
    started: "🚀 {{ pipeline.name }} started by {{ trigger.author }}"
    success: "✅ {{ pipeline.name }} completed in {{ pipeline.duration | duration }}"
    failure: "❌ {{ pipeline.name }} failed at {{ failed_stage.name }}"
    rollback: "⚠️ Rollback triggered for {{ pipeline.name }}"
    
  rate_limit:
    max_per_hour: 10
    deduplicate: true
```

#### Notification Channel Types

| Type | Description | Key Fields |
|------|-------------|------------|
| `slack` | Slack webhook | `webhook`, `channel` |
| `email` | Email notification | `recipients`, `subject` |
| `github_status` | GitHub commit status | `context`, `target_url` |
| `webhook` | Generic HTTP webhook | `url`, `method`, `headers` |
| `pagerduty` | PagerDuty incident | `routing_key`, `severity` |
| `teams` | Microsoft Teams | `webhook` |

#### Notification Events

| Event | Description |
|-------|-------------|
| `started` | Pipeline started |
| `stage_started` | Stage started |
| `stage_complete` | Stage completed (success or failure) |
| `approval_required` | Approval requested |
| `approved` | Approval granted |
| `rejected` | Approval rejected |
| `success` | Pipeline completed successfully |
| `failure` | Pipeline failed |
| `rollback` | Rollback triggered |
| `timeout` | Pipeline or stage timed out |

### Artifacts Block

File preservation configuration.

```yaml
artifacts:
  - name: build_output
    path: "./dist/**/*"
    retention_days: 30
    compression: gzip
    
  - name: test_results
    path: "./coverage/**/*"
    retention_days: 7
    required: true                # Fail if not found
    
  - name: logs
    path: "./.uluops/logs/**/*"
    retention_days: 14
    condition: "pipeline.status == 'failed'"
    
  - name: validation_report
    source: "stages.validate.output"
    format: json
    retention_days: 90
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Artifact identifier |
| `path` | string | No | File glob pattern |
| `source` | string | No | Stage output reference |
| `format` | string | No | Output format (json, yaml, raw) |
| `retention_days` | number | No | Days to retain (default: 30) |
| `compression` | string | No | Compression type (gzip, none) |
| `required` | boolean | No | Fail if artifact missing |
| `condition` | string | No | Conditional preservation |

### State Block

Cross-run state persistence.

```yaml
state:
  persistence: true
  ttl: "30d"                      # Time to live
  
  tracked:
    - last_successful_deploy      # Auto-tracked: timestamp
    - last_rollback               # Auto-tracked: timestamp
    - deployment_count            # Auto-tracked: counter
    - failure_count               # Auto-tracked: counter
    - consecutive_failures        # Auto-tracked: counter
    
  custom:
    - name: last_version
      value: "{{ stages.build.version }}"
      on: success
      
    - name: environment_status
      value: "{{ params.environment }}: {{ pipeline.status }}"
      
  conditions:
    pause_on_consecutive_failures: 3  # Pause after N failures
    require_manual_after_rollback: true
```

| Field | Type | Description |
|-------|------|-------------|
| `persistence` | boolean | Enable state persistence |
| `ttl` | string | Time to live (e.g., "30d", "168h") |
| `tracked` | string[] | Automatically tracked values |
| `custom` | CustomState[] | User-defined state values |
| `conditions` | Conditions | State-based behavior conditions |

### Postflight Block

Actions executed after all stages complete. Handles tracker persistence, token metric collection, and conditional handlers for success/failure outcomes.

```yaml
postflight:
  tracker:
    enabled: true
    project: "$ARGUMENTS"                    # Tracker project name
    workflow_type: "{{ pipeline.interface.name }}"  # Run workflow type
    include_token_metrics: true              # Collect from agent-metrics buffer
    token_metrics_command: "agent-metrics buffer list --since 5m -f tracker"
    field_mappings:
      score_source: "stage.gate.score"       # Where to find validator scores
      status_source: "stage.gate.decision"   # Where to find PASS/FAIL
      model_source: "agent-metrics"          # Source for model identification
      recommendations_source: "stage.output.issues"  # Where to find issues
    verify_save: true                        # Verify saved count matches

  on_success:
    - description: "Report pipeline success"
      command: "echo 'All gates passed'"

  on_failure:
    - description: "Report pipeline failure"
      command: "echo 'Pipeline failed'"

  always:
    - description: "Cleanup temporary files"
      command: "rm -rf /tmp/pipeline-*"
```

#### Tracker Configuration

The `tracker` sub-block configures automatic persistence of pipeline results to the UluOps validation tracker via `save_run`. This replaces the need for manual tracker calls in each pipeline's rendered markdown.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `enabled` | boolean | `true` | Enable tracker persistence |
| `project` | string | — | Tracker project name. Supports `$ARGUMENTS` or template expressions |
| `workflow_type` | string | `pipeline.interface.name` | Workflow type identifier for the tracker run |
| `include_token_metrics` | boolean | `true` | Collect token usage from agent-metrics buffer |
| `token_metrics_command` | string | `agent-metrics buffer list --since 5m -f tracker` | Command to extract token metrics |
| `field_mappings` | FieldMappings | (see below) | Maps pipeline outputs to tracker fields |
| `verify_save` | boolean | `true` | After saving, verify that `total_issues` matches saved count |

#### Field Mappings

Field mappings tell the renderer how to map stage outputs to `save_run` fields.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `score_source` | string | `stage.gate.score` | Expression to extract validator scores |
| `status_source` | string | `stage.gate.decision` | Expression to extract PASS/FAIL status |
| `model_source` | string | `agent-metrics` | Source for model identification |
| `recommendations_source` | string | `stage.output.issues` | Expression to extract issue recommendations |

#### Conditional Handlers

The `on_success`, `on_failure`, and `always` arrays define actions that run after stages complete:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `description` | string | Yes | Human-readable description |
| `command` | string | No | Shell command to execute |
| `condition` | string | No | Conditional expression |

**Execution order:** `on_success` OR `on_failure` runs first (based on outcome), then `always` runs regardless.

---

## Stage Dependency Graph

Pipelines build a DAG (Directed Acyclic Graph) from stage dependencies.

### Execution Order

```yaml
stages:
  - id: A
    # No depends_on = root node (executed first)
    
  - id: B
    depends_on: [A]
    
  - id: C
    depends_on: [A]
    # B and C can run in parallel after A
    
  - id: D
    depends_on: [B, C]
    # D waits for both B and C
```

**Visualization:**

```
    A
   / \
  B   C
   \ /
    D
```

### Parallel Execution

By default, independent stages run in parallel:

```yaml
stages:
  - id: lint          # Root
  - id: test          # Root
  - id: typecheck     # Root
  # All three run in parallel
  
  - id: build
    depends_on: [lint, test, typecheck]
    # Waits for all three
```

**Visualization:**

```
  lint    test    typecheck
    \      |      /
     \     |     /
      \    |    /
       \   |   /
        build
```

### Conditional Branches

Conditions create runtime-determined paths:

```yaml
stages:
  - id: validate
    # Always runs
    
  - id: deploy-staging
    depends_on: [validate]
    condition: "trigger.branch != 'main'"
    
  - id: deploy-prod
    depends_on: [validate]
    condition: "trigger.branch == 'main'"
```

**Visualization:**

```
        validate
        /      \
       /        \
  [branch!=main] [branch==main]
      |              |
deploy-staging   deploy-prod
```

### Cycle Detection

The runtime detects cycles and fails validation:

```yaml
# ✗ Invalid: Circular dependency
stages:
  - id: A
    depends_on: [C]
  - id: B
    depends_on: [A]
  - id: C
    depends_on: [B]
    
# Error: Circular dependency detected: A -> C -> B -> A
```

---

## Trigger Event Types

### Git Push Events

```yaml
- type: git_push
  branches:
    - main
    - "release/*"
    - "!release/beta-*"  # Exclude pattern
  tags:
    - "v*"
    - "!v*-rc*"
  paths:
    include:
      - "src/**"
      - "package.json"
    exclude:
      - "**/*.md"
      - "**/*.test.ts"
```

### Pull Request Events

| Action | Description |
|--------|-------------|
| `opened` | PR opened |
| `closed` | PR closed (merged or not) |
| `synchronize` | New commits pushed |
| `reopened` | PR reopened |
| `labeled` | Label added |
| `unlabeled` | Label removed |
| `review_requested` | Review requested |
| `review_submitted` | Review submitted |
| `converted_to_draft` | Converted to draft |
| `ready_for_review` | Ready for review |

### Schedule Expressions

Cron format: `minute hour day-of-month month day-of-week`

| Expression | Description |
|------------|-------------|
| `0 * * * *` | Every hour |
| `0 2 * * *` | Daily at 2 AM |
| `0 2 * * 1-5` | Weekdays at 2 AM |
| `0 0 1 * *` | First of month |
| `*/15 * * * *` | Every 15 minutes |

### Webhook Payload Filtering

```yaml
- type: webhook
  event: deployment
  payload_filter: |
    $.environment == 'production' &&
    $.action == 'requested'
```

---

## Approval Workflows

### Simple Approval

```yaml
approval:
  required: true
  approvers: ["@ops-team"]
```

### Multi-Level Approval

```yaml
stages:
  - id: security-review
    approval:
      required: true
      approvers: ["@security-team"]
      min_approvals: 1
      
  - id: deploy-prod
    depends_on: [security-review]
    approval:
      required: true
      approvers: ["@release-team", "@ops-team"]
      min_approvals: 2
```

### Conditional Approval

```yaml
approval:
  required: "{{ params.environment == 'production' }}"
  approvers: ["@release-team"]
  skip_for:
    - branches: ["hotfix/*"]
    - authors: ["@emergency-deployers"]
```

### Approval Timeout Behavior

| Behavior | Description |
|----------|-------------|
| `reject` | Auto-reject after timeout (default) |
| `approve` | Auto-approve after timeout |
| `extend` | Notify and extend timeout |
| `pause` | Pause pipeline indefinitely |

---

## Rollback Strategies

### Revert to Last Success

```yaml
rollback:
  enabled: true
  strategy: revert_to_last_success
```

Restores the last known good state from pipeline history.

### Revert to Previous

```yaml
rollback:
  enabled: true
  strategy: revert_to_previous
```

Restores the immediate previous state, regardless of success.

### Custom Rollback

```yaml
rollback:
  enabled: true
  strategy: custom
  
  actions:
    default:
      steps:
        - name: Scale down
          command: kubectl scale deployment/app --replicas=0
        - name: Restore backup
          command: ./scripts/restore-backup.sh
        - name: Scale up
          command: kubectl scale deployment/app --replicas=3
```

### Rollback Triggers

```yaml
rollback:
  triggers:
    # Trigger on stage failure
    - condition: "stages.deploy.failed"
      action: rollback_deploy
      
    # Trigger on low score
    - condition: "stages.verify.score < 50"
      action: full_rollback
      
    # Trigger on timeout
    - condition: "stages.health-check.timeout"
      action: emergency_rollback
      
    # Compound conditions
    - condition: |
        stages.deploy.completed &&
        stages.verify.failed &&
        state.consecutive_failures > 2
      action: rollback_and_pause
```

---

## Data Flow Between Stages

### Output Variables

Stages can export values for downstream consumption:

```yaml
stages:
  - id: build
    steps:
      - name: Build
        command: npm run build
    outputs:
      - name: version
        value: "{{ steps.build.env.npm_package_version }}"
      - name: hash
        command: git rev-parse HEAD
        
  - id: deploy
    depends_on: [build]
    steps:
      - name: Deploy
        command: deploy --version={{ stages.build.outputs.version }}
```

### Input Parameters

Stages can receive inputs from dependencies:

```yaml
stages:
  - id: validate
    workflows:
      - ref: ship@1.0.0
    outputs:
      - name: score
        value: "{{ workflow.score }}"
      - name: issues
        value: "{{ workflow.issues | json }}"
        
  - id: fix
    depends_on: [validate]
    condition: "stages.validate.outputs.score < 80"
    inputs:
      issues: "{{ stages.validate.outputs.issues }}"
    commands:
      - ref: fix-issues@1.0.0
        args:
          issues: "{{ inputs.issues }}"
```

### Artifact Passing

Artifacts can be passed between stages:

```yaml
stages:
  - id: build
    steps:
      - command: npm run build
    artifacts:
      - name: dist
        path: "./dist/**/*"
        
  - id: deploy
    depends_on: [build]
    artifacts_from: [build]  # Import build artifacts
    steps:
      - name: Deploy
        command: aws s3 sync ./dist s3://bucket/
```

---

## Composition Rules

### What Pipelines Can Reference

| Reference Type | Allowed | Example |
|----------------|---------|---------|
| Workflows (WDL) | ✓ | `ship@1.0.0` |
| Commands (CDL) | ✓ | `validate@1.0.0` |
| Agents (ADL) | ✗ | Must wrap in command |
| Other Pipelines | ✓ (via trigger) | `type: pipeline` |

### Version Resolution

```yaml
workflows:
  - ref: ship@1.0.0         # Exact version
  - ref: ship@>=1.0.0       # Minimum version
  - ref: ship@^1.0.0        # Compatible version (1.x.x)
  - ref: ship@~1.0.0        # Patch version (1.0.x)
  - ref: ship@latest        # Latest published
```

### Reference Validation

The registry validates references at publish time:

1. All referenced workflows/commands must exist
2. Version constraints must be satisfiable
3. No circular pipeline triggers
4. Domain compatibility is checked

---

## Runtime Behavior

### Pipeline Lifecycle

```
┌─────────────────────────────────────────────────────────────────┐
│                      PIPELINE LIFECYCLE                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. TRIGGER                                                     │
│     └─ Event received (push, PR, schedule, webhook, manual)     │
│                                                                 │
│  2. VALIDATION                                                  │
│     ├─ Check trigger conditions                                 │
│     ├─ Resolve environment variables                            │
│     ├─ Validate secrets availability                            │
│     └─ Build stage dependency graph                             │
│                                                                 │
│  3. EXECUTION                                                   │
│     ├─ Execute stages in dependency order                       │
│     ├─ Run parallel stages concurrently                         │
│     ├─ Wait for approvals when required                         │
│     ├─ Check gates after each stage                             │
│     └─ Collect artifacts and outputs                            │
│                                                                 │
│  4. COMPLETION                                                  │
│     ├─ Aggregate final status                                   │
│     ├─ Trigger rollback if needed                               │
│     ├─ Send notifications                                       │
│     ├─ Preserve artifacts                                       │
│     └─ Update state                                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Stage Execution

```
┌─────────────────────────────────────────────────────────────────┐
│                      STAGE EXECUTION                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Check stage condition                                       │
│     └─ If false: skip stage, mark as skipped                    │
│                                                                 │
│  2. Wait for dependencies                                       │
│     └─ All depends_on stages must complete                      │
│                                                                 │
│  3. Check approval (if required)                                │
│     ├─ Send approval request                                    │
│     ├─ Wait for approvals or timeout                            │
│     └─ If rejected: fail stage                                  │
│                                                                 │
│  4. Execute items (workflows, commands, steps)                  │
│     ├─ If parallel: execute concurrently                        │
│     ├─ If sequential: execute in order                          │
│     └─ Collect outputs and artifacts                            │
│                                                                 │
│  5. Check gate (if defined)                                     │
│     ├─ Aggregate scores (if workflows)                          │
│     ├─ Compare to threshold                                     │
│     └─ Apply on_failure action                                  │
│                                                                 │
│  6. Update stage status                                         │
│     └─ success | failed | skipped | timeout                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Timeout Handling

| Level | Field | Default | Behavior |
|-------|-------|---------|----------|
| Pipeline | `timeout` | 3600000 (1hr) | Abort all stages |
| Stage | `timeout` | 600000 (10min) | Fail stage |
| Step | `timeout` | 60000 (1min) | Fail step |
| Approval | `timeout_hours` | 72 | Auto-reject |

### Failure Modes

| Mode | Trigger | Action |
|------|---------|--------|
| `abort` | Stage gate failed | Stop pipeline immediately |
| `warn` | Stage gate failed | Continue with warning |
| `skip` | Stage gate failed | Skip dependent stages |
| `rollback` | Stage failed | Execute rollback actions |

---

## Examples

### Minimal Pipeline

```yaml
pipeline:
  interface:
    name: simple
    version: 1.0.0
    displayName: Simple Pipeline
    description: Minimal pipeline example
    domain: software

  triggers:
    - type: manual
      allowed: true

  stages:
    - id: validate
      workflows:
        - ref: ship@1.0.0
```

### Monorepo Pipeline

```yaml
pipeline:
  interface:
    name: monorepo-ci
    version: 1.0.0
    displayName: Monorepo CI
    description: CI pipeline for monorepo with multiple packages
    domain: software

  triggers:
    - type: pull_request
      actions: [opened, synchronize]
      paths:
        include: ["packages/**"]

  environment:
    variables:
      - name: CHANGED_PACKAGES
        value: "{{ trigger.changed_files | packages }}"

  stages:
    - id: detect-changes
      steps:
        - name: Detect changed packages
          command: |
            npx lerna changed --json > changed.json
      outputs:
        - name: packages
          command: cat changed.json | jq -r '.[].name'
          
    - id: lint
      depends_on: [detect-changes]
      parallel: true
      steps:
        - name: Lint changed packages
          command: npx lerna run lint --scope='{{ stages.detect-changes.outputs.packages }}'
          
    - id: test
      depends_on: [detect-changes]
      parallel: true
      steps:
        - name: Test changed packages
          command: npx lerna run test --scope='{{ stages.detect-changes.outputs.packages }}'
          
    - id: build
      depends_on: [lint, test]
      steps:
        - name: Build changed packages
          command: npx lerna run build --scope='{{ stages.detect-changes.outputs.packages }}'
```

### Blue-Green Deployment

```yaml
pipeline:
  interface:
    name: blue-green-deploy
    version: 1.0.0
    displayName: Blue-Green Deployment
    description: Zero-downtime deployment with blue-green strategy
    domain: software

  triggers:
    - type: git_push
      branches: [main]

  environment:
    secrets:
      - name: KUBE_CONFIG
        required: true

  stages:
    - id: build
      steps:
        - name: Build image
          command: docker build -t app:{{ trigger.sha }} .
        - name: Push image
          command: docker push app:{{ trigger.sha }}
          
    - id: deploy-green
      depends_on: [build]
      steps:
        - name: Deploy to green
          command: kubectl set image deployment/app-green app=app:{{ trigger.sha }}
        - name: Wait for rollout
          command: kubectl rollout status deployment/app-green --timeout=300s
          
    - id: verify-green
      depends_on: [deploy-green]
      workflows:
        - ref: smoke-test@1.0.0
          args:
            target: "https://green.example.com"
      gate:
        threshold: 90
        on_failure: abort
        
    - id: switch-traffic
      depends_on: [verify-green]
      approval:
        required: true
        approvers: ["@ops-team"]
        timeout_hours: 4
      steps:
        - name: Switch service to green
          command: kubectl patch service/app -p '{"spec":{"selector":{"version":"green"}}}'
          
    - id: verify-production
      depends_on: [switch-traffic]
      workflows:
        - ref: smoke-test@1.0.0
          args:
            target: "https://app.example.com"
      gate:
        threshold: 95
        on_failure: abort
        
    - id: cleanup-blue
      depends_on: [verify-production]
      steps:
        - name: Scale down blue
          command: kubectl scale deployment/app-blue --replicas=0

  rollback:
    enabled: true
    triggers:
      - condition: "stages.verify-production.failed"
        action: rollback_traffic
    actions:
      rollback_traffic:
        steps:
          - name: Switch back to blue
            command: kubectl patch service/app -p '{"spec":{"selector":{"version":"blue"}}}'
          - name: Scale up blue
            command: kubectl scale deployment/app-blue --replicas=3
```

---

## JSON Schema Reference

The complete JSON Schema is available at:

- **URL:** `https://uluops.ai/schemas/pdl/v1.2.0/pipeline.json`
- **File:** `pdl-schema-v1.2.0.json`

### Schema Validation

```bash
# Validate a pipeline definition
ulu validate ./my-pipeline.pipeline.yaml

# Validate against specific schema version
ulu validate ./my-pipeline.pipeline.yaml --schema pdl@1.0.0
```

### Minimal Schema Structure

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "https://uluops.ai/schemas/pdl/v1.2.0/pipeline.json",
  "title": "Pipeline Definition Language (PDL) Schema",
  "description": "Schema for defining multi-workflow pipelines with triggers, approvals, rollback, and tracker integration",
  "version": "1.2.0",
  "type": "object",
  "required": ["pipeline"],
  "properties": {
    "pipeline": {
      "type": "object",
      "required": ["interface", "stages"],
      "properties": {
        "interface": { "$ref": "#/$defs/interface" },
        "triggers": { "$ref": "#/$defs/triggers" },
        "environment": { "$ref": "#/$defs/environment" },
        "stages": { "$ref": "#/$defs/stages" },
        "rollback": { "$ref": "#/$defs/rollback" },
        "notifications": { "$ref": "#/$defs/notifications" },
        "artifacts": { "$ref": "#/$defs/artifacts" },
        "state": { "$ref": "#/$defs/state" },
        "postflight": { "$ref": "#/$defs/postflight" }
      }
    }
  }
}
```

---

## Revision History

| Version | Date | Changes |
|---------|------|---------|
| 1.2.0 | 2026-03-07 | Added `agents` execution primitive for direct agent invocation by `subagent_type` identifier. New `agentRef` schema type with `ref` (kebab-case, no version), optional `model` (sonnet/opus/haiku), `args`, `condition`, and `timeout`. Eliminates CDL indirection for pipeline-internal agent orchestration. |
| 1.1.0 | 2026-03-07 | Added postflight block for tracker persistence integration. Tracker sub-block configures automatic `save_run` calls with field mappings, token metric collection, and save verification. Conditional handlers (`on_success`, `on_failure`, `always`) for post-completion actions. |
| 1.0.0 | 2026-01-14 | Initial PDL specification. Multi-workflow/command orchestration with stages, triggers (git_push, pull_request, schedule, webhook, manual), environment (variables, secrets), approvals (multi-level, conditional), rollback (strategies, triggers, actions), notifications (slack, email, github_status, webhook), artifacts (retention, compression), and state (persistence, tracking). Interface block consistent with ADL/CDL/WDL. |

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
  | 'career'
  | 'compliance'
  | 'business'
  | 'education'
  | 'documentation'
  | 'security';

type TriggerType = 
  | 'git_push' 
  | 'pull_request' 
  | 'schedule' 
  | 'webhook' 
  | 'manual' 
  | 'pipeline';

type PullRequestAction = 
  | 'opened' 
  | 'closed' 
  | 'synchronize' 
  | 'reopened' 
  | 'labeled' 
  | 'unlabeled'
  | 'review_requested'
  | 'review_submitted'
  | 'converted_to_draft'
  | 'ready_for_review';

type NotificationChannel = 
  | 'slack' 
  | 'email' 
  | 'github_status' 
  | 'webhook' 
  | 'pagerduty' 
  | 'teams';

type NotificationEvent = 
  | 'started' 
  | 'stage_started' 
  | 'stage_complete'
  | 'approval_required'
  | 'approved'
  | 'rejected'
  | 'success' 
  | 'failure' 
  | 'rollback'
  | 'timeout';

type RollbackStrategy = 
  | 'revert_to_last_success' 
  | 'revert_to_previous' 
  | 'custom' 
  | 'none';

type GateAction = 'abort' | 'warn' | 'skip';

type StageStatus = 'pending' | 'running' | 'success' | 'failed' | 'skipped' | 'timeout';

type ApprovalTimeoutBehavior = 'reject' | 'approve' | 'extend' | 'pause';

type VersionReq = 
  | string                    // Exact: "1.0.0"
  | `>=${string}`            // Minimum: ">=1.0.0"
  | `^${string}`             // Compatible: "^1.0.0"
  | `~${string}`             // Patch: "~1.0.0"
  | 'latest';                // Latest published
```

### Interface Types

```typescript
interface PipelineInterface {
  name: string;              // Kebab-case identifier
  version: string;           // SemVer
  displayName: string;       // Human-readable (3-50 chars)
  description: string;       // Usage description (20-500 chars)
  domain: Domain;
  subdomain?: string;
  tags?: string[];
}
```

### Trigger Types

```typescript
interface Trigger {
  type: TriggerType;
}

interface GitPushTrigger extends Trigger {
  type: 'git_push';
  branches?: string[];
  tags?: string[];
  paths?: {
    include?: string[];
    exclude?: string[];
  };
}

interface PullRequestTrigger extends Trigger {
  type: 'pull_request';
  actions?: PullRequestAction[];
  branches?: string[];
  paths?: {
    include?: string[];
    exclude?: string[];
  };
}

interface ScheduleTrigger extends Trigger {
  type: 'schedule';
  cron: string;
  timezone?: string;
  skip_if_running?: boolean;
}

interface WebhookTrigger extends Trigger {
  type: 'webhook';
  event: string;
  secret?: string;
  payload_filter?: string;
}

interface ManualTrigger extends Trigger {
  type: 'manual';
  allowed: boolean;
  parameters?: Parameter[];
}

interface Parameter {
  name: string;
  type: 'string' | 'boolean' | 'number';
  default?: any;
  required?: boolean;
  options?: any[];
  pattern?: string;
}
```

### Environment Types

```typescript
interface Environment {
  variables?: Variable[];
  secrets?: Secret[];
}

interface Variable {
  name: string;
  value: string;
  stages?: string[];
}

interface Secret {
  name: string;
  required?: boolean;
  stages?: string[];
  provider?: string;
  path?: string;
}
```

### Stage Types

```typescript
interface Stage {
  id: string;
  name: string;
  depends_on?: string[];
  condition?: string;
  timeout?: number;
  parallel?: boolean;
  sequential?: boolean;
  workflows?: WorkflowRef[];
  commands?: CommandRef[];
  steps?: Step[];
  gate?: Gate;
  artifacts?: Artifact[];
  approval?: Approval;
  env?: Record<string, string>;
  inputs?: Record<string, string>;
  outputs?: Output[];
}

interface WorkflowRef {
  ref: string;
  args?: Record<string, any>;
  condition?: string;
  timeout?: number;
}

interface CommandRef {
  ref: string;
  args?: Record<string, any>;
  condition?: string;
  timeout?: number;
}

interface Step {
  name: string;
  command: string;
  working_dir?: string;
  env?: Record<string, string>;
  timeout?: number;
  continue_on_error?: boolean;
  always_run?: boolean;
  retries?: number;
  retry_delay?: number;
  expect_empty?: boolean;
  expect_match?: string;
}

interface Gate {
  threshold?: number;
  aggregate?: 'min' | 'max' | 'average';
  on_failure?: GateAction;
  on_success?: 'continue' | 'skip_remaining';
}

interface Output {
  name: string;
  value?: string;
  command?: string;
}
```

### Approval Types

```typescript
interface Approval {
  required: boolean | string;
  approvers?: string[];
  min_approvals?: number;
  timeout_hours?: number;
  timeout_behavior?: ApprovalTimeoutBehavior;
  auto_reject_on_change?: boolean;
  allow_self_approval?: boolean;
  skip_for?: SkipCondition[];
  notification?: ApprovalNotification;
}

interface SkipCondition {
  branches?: string[];
  authors?: string[];
  labels?: string[];
}

interface ApprovalNotification {
  channels: string[];
  message?: string;
}
```

### Rollback Types

```typescript
interface Rollback {
  enabled: boolean;
  strategy: RollbackStrategy;
  triggers?: RollbackTrigger[];
  actions?: Record<string, RollbackAction>;
  preserve_logs?: boolean;
  notify_on_rollback?: boolean;
  max_rollback_attempts?: number;
}

interface RollbackTrigger {
  condition: string;
  action: string;
}

interface RollbackAction {
  steps: Step[];
  timeout?: number;
  notify?: boolean;
}
```

### Notification Types

```typescript
interface Notifications {
  channels?: NotificationChannelConfig[];
  templates?: Record<string, string>;
  rate_limit?: RateLimit;
}

interface NotificationChannelConfig {
  type: NotificationChannel;
  events: NotificationEvent[];
  // Type-specific fields
  webhook?: string;
  channel?: string;
  recipients?: string[];
  url?: string;
  method?: string;
  headers?: Record<string, string>;
  context?: string;
  enabled?: boolean;
}

interface RateLimit {
  max_per_hour?: number;
  deduplicate?: boolean;
}
```

### Artifact Types

```typescript
interface Artifact {
  name: string;
  path?: string;
  source?: string;
  format?: 'json' | 'yaml' | 'raw';
  retention_days?: number;
  compression?: 'gzip' | 'none';
  required?: boolean;
  condition?: string;
}
```

### State Types

```typescript
interface State {
  persistence: boolean;
  ttl?: string;
  tracked?: string[];
  custom?: CustomState[];
  conditions?: StateConditions;
}

interface CustomState {
  name: string;
  value: string;
  on?: 'success' | 'failure' | 'always';
}

interface StateConditions {
  pause_on_consecutive_failures?: number;
  require_manual_after_rollback?: boolean;
}
```

### Postflight Types

```typescript
interface Postflight {
  tracker?: PostflightTracker;
  on_success?: PostflightAction[];
  on_failure?: PostflightAction[];
  always?: PostflightAction[];
}

interface PostflightTracker {
  enabled?: boolean;               // Default: true
  project?: string;                // Tracker project name
  workflow_type?: string;          // Workflow type for tracker run
  include_token_metrics?: boolean; // Default: true
  token_metrics_command?: string;  // Command to extract metrics
  field_mappings?: PostflightFieldMappings;
  verify_save?: boolean;           // Default: true
}

interface PostflightFieldMappings {
  score_source?: string;           // Default: "stage.gate.score"
  status_source?: string;          // Default: "stage.gate.decision"
  model_source?: string;           // Default: "agent-metrics"
  recommendations_source?: string; // Default: "stage.output.issues"
}

interface PostflightAction {
  description: string;
  command?: string;
  condition?: string;
}
```

### Complete Pipeline Type

```typescript
interface Pipeline {
  interface: PipelineInterface;
  triggers?: Trigger[];
  environment?: Environment;
  stages: Stage[];
  rollback?: Rollback;
  notifications?: Notifications;
  artifacts?: Artifact[];
  state?: State;
  postflight?: Postflight;
}
```
