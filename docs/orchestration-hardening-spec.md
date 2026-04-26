# Archon Orchestration Hardening Spec

Status: Proposed for implementation

Target: `packages/workflows` and `packages/providers`

Scope: Small hardening pass for workflow option propagation, Codex workflow-level config, and deterministic review gate validation.

## 1. Purpose

Archon is being evaluated as an orchestration layer for multi-model and agent development workflows. The current workflow engine has useful DAG, provider, approval, loop, and bash/script primitives, but three issues make workflow behavior harder to trust:

1. Several workflow-level options are accepted by schemas and read by the executor, but are dropped by the YAML loader.
2. Codex workflow-level reasoning/search settings are parsed but are not applied when Codex nodes execute.
3. Review gates can be modeled as sequencing gates, but hard approval should be decided by deterministic checks, not by unvalidated model text.

This spec defines a focused fix for those three items. It does not introduce a new workflow engine, a new provider abstraction, or a first-class `gate:` primitive.

## 2. Goals

1. Make workflow YAML truthful: fields accepted by the workflow schema must survive loading and reach the executor.
2. Make Codex per-workflow controls effective for Codex nodes.
3. Define a fail-closed deterministic gate pattern using existing workflow primitives.
4. Add focused tests that prevent regressions in loader, executor, and gate-validation behavior.

## 3. Non-Goals

1. No authentication, CORS, or deployment hardening.
2. No Codex read-only sandbox implementation.
3. No new `gate:` node type in this pass.
4. No UI changes beyond whatever existing workflow status surfaces already show.
5. No rewrite of the draft Claude/Codex delivery workflow.
6. No changes to provider SDK behavior except passing the already-supported effective config.

## 4. Current Problems

### 4.1 Workflow Options Dropped by Loader

`workflowBaseSchema` includes:

- `effort`
- `thinking`
- `fallbackModel`
- `betas`
- `sandbox`

`executeDagWorkflow()` reads these fields through `workflowLevelOptions`, and `resolveNodeProviderAndModel()` merges node-level values with workflow-level values.

However, `parseWorkflow()` currently returns only a subset of workflow-level fields. As a result, YAML authors can set options that appear valid but have no effect after loading.

### 4.2 Codex Workflow-Level Config Not Applied

The loader preserves:

- `modelReasoningEffort`
- `webSearchMode`
- `additionalDirectories`

The Codex provider can consume these values through `assistantConfig`, but DAG execution passes only `config.assistants[provider]` as `assistantConfig`. The workflow-level values are not merged into the Codex effective config.

### 4.3 Gates Depend Too Much on Model Text

Archon supports `output_format`, `when`, `approval`, `loop`, `until_bash`, `bash`, and `script` nodes. These are enough to create hard gates, but the current pattern is not explicit:

- A reviewer can output prose instead of valid structured data unless schema compliance is required and validated.
- A loop can complete from a model-emitted completion signal.
- A downstream `when` can compare a string that may be missing, malformed, or semantically inconsistent.

For hard delivery workflows, the model may provide judgment, but code must decide whether the workflow advances.

## 5. Design

### 5.1 Loader Hardening

`parseWorkflow()` must preserve every workflow-level field that the schema and executor intentionally support.

Required preserved fields:

- Existing fields:
  - `name`
  - `description`
  - `provider`
  - `model`
  - `modelReasoningEffort`
  - `webSearchMode`
  - `additionalDirectories`
  - `interactive`
  - `nodes`
  - `worktree`
- Newly fixed fields:
  - `effort`
  - `thinking`
  - `fallbackModel`
  - `betas`
  - `sandbox`

Validation policy:

- Use the existing Zod schemas for these fields.
- For strict orchestration controls, invalid values should fail workflow loading instead of being silently dropped.
- Preserve existing warn-and-ignore behavior only where it is already an explicit compatibility decision, such as `modelReasoningEffort`, `webSearchMode`, `additionalDirectories`, `interactive`, and `worktree.enabled`.

This is an intentional behavior change for `effort`, `thinking`, `fallbackModel`, `betas`, and `sandbox`: invalid workflow-level values should fail fast instead of disappearing. No bundled workflow under `.archon/workflows/defaults/` or `packages/workflows/src/defaults/` currently sets these fields at workflow level, so migration impact is expected to be zero. Failure messages should follow the existing provider/model validation style: include the workflow filename, the invalid field/value, and `errorType: 'validation_error'`.

Acceptance criteria:

- A workflow YAML containing `effort`, `thinking`, `fallbackModel`, `betas`, and `sandbox` loads into a `WorkflowDefinition` with those fields intact.
- Invalid strict fields fail load with `errorType: 'validation_error'`.
- A YAML-loaded workflow with `effort: high` reaches execution with the effective Claude node config containing `effort: 'high'`. This verifies the loader-to-executor-to-provider round trip, not only loader parsing.
- Existing loader tests for older fields continue to pass.

### 5.2 Codex Effective Config

When a node resolves to `provider: codex`, the executor must pass an effective assistant config that includes workflow-level Codex settings.

Precedence:

1. Workflow-level Codex config:
   - `modelReasoningEffort`
   - `webSearchMode`
   - `additionalDirectories`
2. Assistant config from `.archon/config.yaml` or equivalent loaded config.
3. Provider defaults.

The practical merge order is:

```ts
const workflowCodexConfig: Record<string, unknown> = {};
if (workflow.modelReasoningEffort !== undefined) {
  workflowCodexConfig.modelReasoningEffort = workflow.modelReasoningEffort;
}
if (workflow.webSearchMode !== undefined) {
  workflowCodexConfig.webSearchMode = workflow.webSearchMode;
}
if (workflow.additionalDirectories !== undefined) {
  workflowCodexConfig.additionalDirectories = workflow.additionalDirectories;
}

const effectiveAssistantConfig = {
  ...config.assistants.codex,
  ...workflowCodexConfig,
};
```

Only defined workflow-level values should override assistant config values. Undefined workflow fields must not erase assistant defaults.

This merge must apply only to Codex. Claude-specific workflow controls such as `effort`, `thinking`, `fallbackModel`, `betas`, and `sandbox` continue to flow through `nodeConfig` and existing Claude provider translation.

Acceptance criteria:

- A workflow with `provider: codex` and `modelReasoningEffort: high` causes Codex thread options to receive `modelReasoningEffort: 'high'`.
- A Codex node inside a Claude-default workflow still receives workflow-level Codex settings when the node sets `provider: codex`.
- Assistant config values remain in effect when workflow-level Codex values are absent.
- Workflow-level Codex values override assistant config values when both are present.

### 5.3 Deterministic Gate Pattern

This pass standardizes deterministic gates as a workflow authoring pattern using existing nodes.

Every hard review gate must have this shape:

```text
author node
  -> reviewer snapshot node, when mutation/read-only checks are needed
  -> reviewer node with output_format
  -> deterministic validation node
  -> decide/final node
  -> optional approval halt
```

The reviewer node must:

- Use `output_format` with a strict JSON object schema.
- Include a required `verdict` enum.
- Include a required `blocking_issues` array.
- Include enough required fields for downstream validation to make a decision without reading prose.

Recommended reviewer verdicts:

- `APPROVED`
- `REJECTED`
- `NEEDS_HUMAN`

The validation node must:

- Treat missing reviewer output as failure.
- Treat malformed reviewer JSON as failure.
- Treat schema-invalid reviewer data as failure.
- Treat `verdict != APPROVED` as not approved.
- Treat `blocking_issues.length > 0` as not approved.
- Check required artifacts exist.
- Check quality logs or quality sentinels exist when the gate depends on quality.
- Check any read-only mutation guard outputs when the gate depends on reviewer non-mutation.
- Emit one exact machine-readable stdout token for downstream `when` checks.

Recommended validation stdout tokens:

- `GATE_APPROVED`
- `GATE_REJECTED`
- `GATE_NEEDS_HUMAN`
- `GATE_INVALID`

Downstream nodes must branch only on these deterministic tokens, not on raw model prose.

Loop policy:

- A model-emitted `until` completion signal is not sufficient for a hard gate.
- Hard loops must use `until_bash` or a downstream validation node when their result controls progression.
- If `until` and `until_bash` disagree, downstream validation is authoritative.

Approval policy:

- Human approval nodes are used only for explicit halt paths:
  - reviewer verdict is `NEEDS_HUMAN`
  - max attempts are exhausted
  - validation returns `GATE_INVALID`
- Human approval does not retroactively make malformed reviewer output valid. It only records an operator override at the gate.

Acceptance criteria:

- A sample workflow gate can be represented using existing YAML primitives.
- The sample gate fails closed on missing reviewer output.
- The sample gate fails closed on malformed reviewer output.
- The sample gate does not advance when `verdict` is `APPROVED` but `blocking_issues` is non-empty.
- The sample gate advances only when validation emits `GATE_APPROVED`.

## 6. Implementation Plan

### 6.1 Loader Changes

Files:

- `packages/workflows/src/loader.ts`
- `packages/workflows/src/loader.test.ts`
- Possibly `packages/workflows/src/schemas.test.ts`

Tasks:

1. Parse and validate `effort`, `thinking`, `fallbackModel`, `betas`, and `sandbox`.
2. Include validated values in the returned `WorkflowDefinition`.
3. Add tests for successful preservation.
4. Add tests for invalid strict field rejection.

### 6.2 Codex Config Changes

Files:

- `packages/workflows/src/dag-executor.ts`
- `packages/workflows/src/dag-executor.test.ts`
- Possibly `packages/providers/src/codex/provider.test.ts`

Tasks:

1. Extend workflow-level option handling to include Codex workflow config.
2. Build an effective assistant config when resolved provider is `codex`.
3. Merge defined workflow-level Codex settings over assistant config. Build the workflow override object by conditional assignment so absent workflow values are not spread as present `undefined` keys.
4. Add tests for override and fallback behavior.

### 6.3 Gate Pattern Documentation and Tests

Files:

- `packages/docs-web/src/content/docs/guides/deterministic-gates.md`
- `packages/workflows/src/dag-executor.test.ts` or a new workflow validation fixture test

Tasks:

1. Document the deterministic gate pattern.
2. Provide a minimal YAML example.
3. Add tests around validation-node branching if existing test helpers can execute the pattern without excessive fixture setup.
4. Avoid adding a new `gate:` primitive in this pass.

## 7. Example Gate Pattern

```yaml
- id: spec-review
  provider: codex
  output_format:
    type: object
    required: [verdict, blocking_issues, confidence]
    properties:
      verdict:
        type: string
        enum: [APPROVED, REJECTED, NEEDS_HUMAN]
      blocking_issues:
        type: array
        items:
          type: object
          required: [id, severity, issue, required_change]
          properties:
            id: { type: string }
            severity: { type: string, enum: [blocker, major, minor] }
            issue: { type: string }
            required_change: { type: string }
      confidence:
        type: string
        enum: [high, medium, low]
  prompt: |
    Review .specify/spec.md against the requested objective.
    Return strict JSON only.

- id: spec-validate
  depends_on: [spec-review]
  trigger_rule: all_done
  bash: |
    set -euo pipefail

    RAW=$spec-review.output

    python3 - "$RAW" <<'PY'
    import json, sys

    raw = sys.argv[1]
    if not raw.strip():
        print("GATE_INVALID")
        sys.exit(0)

    try:
        data = json.loads(raw)
    except Exception:
        print("GATE_INVALID")
        sys.exit(0)

    verdict = data.get("verdict")
    issues = data.get("blocking_issues")
    if not isinstance(issues, list):
        print("GATE_INVALID")
    elif verdict == "APPROVED" and len(issues) == 0:
        print("GATE_APPROVED")
    elif verdict == "NEEDS_HUMAN":
        print("GATE_NEEDS_HUMAN")
    else:
        print("GATE_REJECTED")
    PY

- id: plan-author
  depends_on: [spec-validate]
  when: "$spec-validate.output == 'GATE_APPROVED'"
  provider: claude
  prompt: |
    Author the implementation plan from the approved spec.

- id: spec-halt
  depends_on: [spec-validate]
  when: "$spec-validate.output != 'GATE_APPROVED'"
  approval:
    message: Spec gate did not approve. Review the validation result before continuing.
    capture_response: true
```

## 8. Testing Requirements

Run at minimum:

```bash
bun --filter @archon/workflows test
bun --filter @archon/providers test
bun --filter @archon/workflows type-check
bun --filter @archon/providers type-check
```

If dependencies are missing in the checkout, run `bun install` first.

Follow the repo's test-isolation rules in `CLAUDE.md`. Bun's `mock.module()` is process-global and not undone by `mock.restore()`. If new tests mock a module path already mocked elsewhere with a different implementation, add a separate `bun test ...` invocation in the relevant package's `package.json` test script instead of running conflicting mocks in the same process.

Before considering the hardening complete, run the repository validation command:

```bash
bun run validate
```

## 9. Rollout

1. Implement loader hardening first.
2. Implement Codex effective config second.
3. Add deterministic gate documentation and sample tests third.
4. Update any draft Claude/Codex pipeline YAML only after the engine behavior is verified.

## 10. Success Criteria

The hardening is complete when:

1. Workflow-level options accepted by schema are no longer lost during loading.
2. Codex workflow-level reasoning/search settings affect Codex node execution.
3. Documentation gives workflow authors a concrete fail-closed gate pattern.
4. Tests cover the loader and Codex config regressions.
5. A hard review gate can be expressed without relying on raw model prose as the approval signal.
