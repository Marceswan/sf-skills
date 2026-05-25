---
name: generating-flow
description: "Creates and validates Salesforce Flows with 110-point scoring. TRIGGER when: user builds or edits record-triggered, screen, autolaunched, or scheduled flows, touches .flow-meta.xml files, or asks to generate flow metadata. DO NOT TRIGGER when: Apex automation (use generating-apex), process builder migration questions only, or non-Flow declarative config (use generating-custom-object)."
license: MIT
metadata:
  version: "2.1"
  author: "Jag Valaiyapathy"
  scoring: "110 points across 6 categories"
---

# generating-flow: Salesforce Flow Creation and Validation

Use this skill when the user needs **Flow design or Flow XML work**: record-triggered, screen, autolaunched, scheduled, or platform-event Flows, including generation, validation, architecture choices, and safe deployment sequencing.

## When This Skill Owns the Task

Use `generating-flow` when the work involves:
- `.flow-meta.xml` files
- Flow Builder architecture and XML generation
- record-triggered, screen, scheduled, autolaunched, or platform-event flows
- Flow-specific bulk safety, fault paths, and subflow orchestration

Delegate elsewhere when the user is:
- writing Apex-first automation -> [generating-apex](../generating-apex/SKILL.md)
- creating objects / fields first -> [generating-custom-object](../generating-custom-object/SKILL.md), [generating-custom-field](../generating-custom-field/SKILL.md)
- deploying metadata -> [deploying-metadata](../deploying-metadata/SKILL.md)
- seeding post-deploy test data -> [handling-sf-data](../handling-sf-data/SKILL.md)

---

## Required Context to Gather First

Ask for or infer:
- flow type
- trigger object / entry conditions
- core business goal
- whether this is new, refactor, or repair
- target org alias if deployment or validation is needed
- whether related objects / fields already exist

---

## Recommended Workflow

### 1. Choose the right automation tool
Before building, confirm Flow is the right answer rather than:
- formula field
- validation rule
- roll-up summary
- Apex

### 2. Choose the right Flow type
| Need | Default flow type |
|---|---|
| same-record update before save | before-save record-triggered |
| related-record work / emails / callouts | after-save record-triggered |
| guided UI | screen flow |
| reusable background logic | autolaunched / subflow |
| scheduled processing | scheduled flow |
| event-driven declarative response | platform-event flow |
| AI-evaluated routing (sentiment, intent, tone) | autolaunched with AI Decision element |

### 3. Start from a template
Prefer the provided assets:
- `assets/record-triggered-before-save.xml`
- `assets/record-triggered-after-save.xml`
- `assets/record-triggered-before-delete.xml`
- `assets/screen-flow-template.xml`
- `assets/screen-flow-with-lwc.xml`
- `assets/autolaunched-flow-template.xml`
- `assets/scheduled-flow-template.xml`
- `assets/platform-event-flow-template.xml`
- `assets/ai-decision-template.xml`
- `assets/apex-action-template.xml`
- `assets/wait-template.xml`
- `assets/bypass-check-decision.xml`
- `assets/elements/` and `assets/subflows/`

### 4. Validate against Flow guardrails
Focus on:
- no DML in loops
- no Get Records inside loops
- proper fault paths
- correct trigger conditions
- safe subflow composition
- AI Decision elements not placed inside loops (credit cost per iteration)
- AI Decision prompts include merge field references for data context

Use `hooks/scripts/validate_flow.py` and `hooks/scripts/simulate_flow.py` to catch structural problems before deploy.

### 5. Hand off deployment and testing
Use:
- [deploying-metadata](../deploying-metadata/SKILL.md) for deploy / dry-run
- [handling-sf-data](../handling-sf-data/SKILL.md) for high-volume test data

---

## MCP Pipeline Generation (execute_metadata_action)

When generating brand-new flow metadata XML from a natural-language request and the `execute_metadata_action` MCP tool is available, use the **mandatory 3-step pipeline** instead of hand-writing XML. This produces valid metadata that hand-authoring frequently breaks.

**MANDATORY: Follow this exact 3-step pipeline. Do NOT manually create flow metadata XML or use any other tool/API/method to generate flow metadata when this pipeline is available. Any deviation produces invalid or broken metadata.**

All 3 steps are called via the MCP tool `execute_metadata_action`; the `action` parameter selects the step.

### Step 1 (REQUIRED): `fetchGroundedObjectMetadata`
Fetches org schema metadata relevant to the request. Always called first.
- **userPrompt** (STRING, REQUIRED): the user's natural-language request
- **inflightMetadata** (ARRAY, REQUIRED): custom objects/fields from the local sfdx project, or `[]` if none
- Output **groundingMetadata** (STRING): pass directly to Step 2 (already a string, do NOT serialize again)

### Step 2 (REQUIRED): `flowElementSelection`
Selects flow elements and connections. Called after Step 1.
- **userPrompt** (STRING, REQUIRED): same value as Step 1
- **groundingMetadata** (STRING, REQUIRED): exact string from Step 1 output
- **operationId** (STRING, REQUIRED): empty string `""` on first call
- Output **operationId** (STRING): pass to Step 3; **userOutput** (STRING): reasoning you may show the user

### Step 3 (REQUIRED): `flowElementGeneration`
Generates flow metadata one element at a time. Called repeatedly until done.
- **operationId** (STRING, REQUIRED): from Step 2 output
- **requestSource** (STRING, REQUIRED): use `"A4V"` to get flow metadata in XML
- Outputs **isComplete** (BOOLEAN) and **result** (STRING): the final flow metadata appears in `result` only when `isComplete` is `true`

**Loop rules:** Keep calling Step 3 with the same `operationId` until `isComplete` is `true` or errors are returned. A flow may have any number of elements, so expect many iterations. Do NOT pause, summarize mid-loop, or ask the user to continue. If errors are returned, stop and surface them.

**inflightMetadata format:** ARRAY (not string). Use strict naming: object/field API name = `apiName`; field type = `type`; lookup target = `referenceTo`. Picklists include a `values` array. Use `[]` (array, not the string `"[]"`) when no custom objects are relevant. Never put flow requirements or text descriptions in `inflightMetadata` -- requirements go only in `userPrompt`.

**Multiple flows = multiple separate pipelines.** Split a multi-flow request into one focused `userPrompt` per flow, run each full 3-step pipeline SEQUENTIALLY (never in parallel), and complete each pipeline before starting the next. Scope each pipeline's `inflightMetadata` to only that flow's objects/fields.

**Do NOT modify pipeline XML output:** do not add, remove, or change nodes, tags, attributes, labels, or X/Y coordinates. The final XML must be identical to what the pipeline returned. **Exception:** when the user explicitly asks to fix validation/deployment errors in an already-generated flow, targeted manual edits to the XML are permitted.

---

## High-Signal Rules

### Flow architecture
- before-save for same-record field updates
- after-save for related records, emails, and callouts
- do not loop over `$Record`
- use subflows when logic becomes wide or repetitive

### Bulk safety
- no DML in loops
- no Get Records in loops
- test with **251+ records** when bulk behavior matters
- prefer Transform when the job is shaping data, not per-record branching

### Error handling
- every data-changing path should have fault handling
- avoid self-referencing fault connectors
- deploy Flows as Draft first when activation risk is non-trivial

---

## Output Format

When finishing, report in this order:
1. **Flow type and goal**
2. **Files created or updated**
3. **Architecture choices**
4. **Bulk/error-handling notes**
5. **Deploy/testing next steps**

Suggested shape:

```text
Flow: <name>
Type: <flow type>
Files: <paths>
Design: <trigger choice, subflows, key decisions>
Risks: <bulk safety, fault paths, dependencies>
Next step: <dry-run deploy, activate, or test>
```

---

## Flow Testing (CLI)

Run Flow tests from the command line without VS Code:

```bash
# Run all flow tests
sf flow run test --target-org <alias> --json

# Run tests for a specific flow
sf flow run test --class-names MyFlow --target-org <alias> --json

# Get results for an asynchronous run
sf flow get test --test-run-id <id> --target-org <alias> --json
```

Flow tests execute in the org and can take 1-5 minutes. `sf flow run test` returns a test run ID for asynchronous runs; use `sf flow get test` to retrieve results later. Always run with `--json` and use background execution for longer runs.

---

## Cross-Skill Integration

| Need | Delegate to | Reason |
|---|---|---|
| create objects / fields first | [generating-custom-object](../generating-custom-object/SKILL.md) | schema readiness |
| deploy / activate flow | [deploying-metadata](../deploying-metadata/SKILL.md) | safe deployment sequence |
| create realistic bulk test data | [handling-sf-data](../handling-sf-data/SKILL.md) | post-deploy verification |
| create Apex actions / invocables | [generating-apex](../generating-apex/SKILL.md) | imperative logic |
| embed LWC in a screen flow | [generating-lwc-components](../generating-lwc-components/SKILL.md) | custom UI components |
| expose Flow to Agentforce | [building-agentscript](../building-agentscript/SKILL.md) | agent action orchestration |

---

## Reference File Index

| File | When to read |
|------|-------------|
| `references/flow-best-practices.md` | Start here -- core Flow design and guardrail guidance |
| `references/flow-quick-reference.md` | Fast lookup of element types and conventions |
| `references/orchestration.md` | Subflow and multi-flow orchestration overview |
| `references/orchestration-guide.md` | Deep orchestration patterns |
| `references/orchestration-parent-child.md` | Parent/child subflow composition |
| `references/orchestration-sequential.md` | Sequential subflow chaining |
| `references/orchestration-conditional.md` | Conditional orchestration routing |
| `references/subflow-library.md` | Catalog of reusable subflow patterns |
| `references/governance-checklist.md` | Pre-deploy governance and review checklist |
| `references/transform-vs-loop-guide.md` | When to use Transform vs Loop elements |
| `references/triangle-pattern.md` | Triangle (trigger/subflow/Apex) design pattern |
| `references/ai-decision-guide.md` | AI Decision element design and cost guidance |
| `references/form-building-guide.md` | Screen flow form construction |
| `references/screen-flow-example.md` | Worked screen flow example |
| `references/record-trigger-example.md` | Worked record-triggered flow example |
| `references/integration-patterns.md` | External callout and integration patterns |
| `references/lwc-integration-guide.md` | Embedding LWC in screen flows |
| `references/agentforce-flow-integration.md` | Exposing Flows to Agentforce |
| `references/wait-patterns.md` | Wait / pause / scheduled-path patterns |
| `references/error-logging-example.md` | Fault-path error logging example |
| `references/multi-step-dml-rollback-example.md` | Multi-step DML rollback pattern |
| `references/testing-guide.md` | Flow testing strategy |
| `references/testing-checklist.md` | Pre-release Flow test checklist |
| `references/xml-gotchas.md` | Common Flow XML pitfalls |
| `assets/` | Flow XML templates (record-triggered, screen, scheduled, autolaunched, platform-event, AI decision, apex action, wait, bypass-check) plus `elements/` and `subflows/` building blocks |
| `assets/elements/` | Reusable element fragments (get-records, loop, record-delete, transform) |
| `assets/subflows/` | Reusable subflow XML (bulk-updater, dml-rollback, email-alert, error-logger, query-with-retry, record-validator) |
| `scripts/doc_generator.py` | Generate Flow documentation from metadata |
| `hooks/scripts/validate_flow.py` | Validate flow structure before deploy |
| `hooks/scripts/simulate_flow.py` | Simulate flow execution paths |
| `hooks/scripts/post-tool-validate.py` | Post-tool validation hook |

---

## Score Guide

| Score | Meaning |
|---|---|
| 88+ | production-ready Flow |
| 75-87 | good Flow with some review items |
| 60-74 | functional but needs stronger guardrails |
| < 60 | unsafe / incomplete for deployment |
