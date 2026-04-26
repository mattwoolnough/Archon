# Orchestration Hardening Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make workflow-level orchestration settings truthful, apply Codex workflow-level config, and document/test deterministic review gate patterns.

**Architecture:** Keep the existing workflow engine shape. Fix YAML loading in `loader.ts`, compute effective provider config in `dag-executor.ts`, and add a dedicated docs guide for deterministic gates using existing `output_format`, `bash`, `when`, `trigger_rule`, and `approval` primitives. No new node type or provider abstraction is introduced.

**Tech Stack:** TypeScript, Bun test, Zod schemas from `@hono/zod-openapi`, Archon workflow DAG executor, Markdown docs under `packages/docs-web`.

---

## File Structure

- Modify `packages/workflows/src/loader.ts`
  - Responsibility: parse YAML into `WorkflowDefinition` and fail fast on invalid strict workflow-level orchestration options.
- Modify `packages/workflows/src/loader.test.ts`
  - Responsibility: prove workflow-level Claude SDK options survive YAML discovery and invalid strict options produce `validation_error`.
- Modify `packages/workflows/src/dag-executor.ts`
  - Responsibility: carry loaded workflow options into provider execution and build Codex effective `assistantConfig` for prompt, approval-reject, and loop nodes.
- Modify `packages/workflows/src/dag-executor.test.ts`
  - Responsibility: prove loaded workflow options reach provider options, Codex workflow config overrides assistant defaults without erasing undefined fields, and deterministic gate validator nodes run fail-closed.
- Create `packages/docs-web/src/content/docs/guides/deterministic-gates.md`
  - Responsibility: user-facing guide for hard review gates.
- Modify `packages/docs-web/src/content/docs/guides/index.md`
  - Responsibility: link the new deterministic gates guide from the guides index.

Implementation should run in a dedicated worktree. Do not edit the untracked draft workflow `.archon/workflows/claude-codex-gated-delivery.yaml` as part of this plan.

---

### Task 1: Add Loader Tests for Workflow-Level Strict Options

**Files:**
- Modify: `packages/workflows/src/loader.test.ts`

- [ ] **Step 1: Add a passing-shape test that currently fails because fields are dropped**

Insert this test inside the `parseWorkflow (via discoverWorkflows)` describe block, near the existing `should parse codex options fields` test:

```ts
    it('should parse workflow-level Claude SDK options', async () => {
      const workflowDir = join(testDir, '.archon', 'workflows');
      await mkdir(workflowDir, { recursive: true });

      const yaml = `name: claude-sdk-options
description: Claude SDK options are parsed at workflow level
provider: claude
model: sonnet
effort: high
thinking: enabled
fallbackModel: opus
betas:
  - fine-grained-tool-streaming-2025-05-14
sandbox:
  enabled: true
  network:
    allowManagedDomainsOnly: true
  filesystem:
    denyRead:
      - .env
nodes:
  - id: test
    command: test
`;
      await writeFile(join(workflowDir, 'claude-sdk-options.yaml'), yaml);

      const result = await discoverWorkflows(testDir, { loadDefaults: false });

      expect(result.errors).toHaveLength(0);
      expect(result.workflows).toHaveLength(1);

      const workflow = result.workflows[0].workflow;
      expect(workflow.effort).toBe('high');
      expect(workflow.thinking).toEqual({ type: 'enabled' });
      expect(workflow.fallbackModel).toBe('opus');
      expect(workflow.betas).toEqual(['fine-grained-tool-streaming-2025-05-14']);
      expect(workflow.sandbox).toEqual({
        enabled: true,
        network: { allowManagedDomainsOnly: true },
        filesystem: { denyRead: ['.env'] },
      });
    });
```

- [ ] **Step 2: Add a fail-fast validation test for strict workflow-level fields**

Insert this test after the test from Step 1:

```ts
    it('should reject invalid workflow-level Claude SDK options', async () => {
      const workflowDir = join(testDir, '.archon', 'workflows');
      await mkdir(workflowDir, { recursive: true });

      const invalidCases = [
        { filename: 'bad-effort.yaml', field: 'effort', fragment: 'effort: extreme' },
        { filename: 'bad-thinking.yaml', field: 'thinking', fragment: 'thinking: maybe' },
        { filename: 'bad-fallback.yaml', field: 'fallbackModel', fragment: 'fallbackModel: ""' },
        { filename: 'bad-betas.yaml', field: 'betas', fragment: 'betas: []' },
        { filename: 'bad-sandbox.yaml', field: 'sandbox', fragment: 'sandbox: true' },
      ];

      for (const testCase of invalidCases) {
        const workflowName = testCase.filename.replace('.yaml', '');
        const yaml = `name: ${workflowName}
description: Invalid strict workflow option
provider: claude
model: sonnet
${testCase.fragment}
nodes:
  - id: test
    command: test
`;
        await writeFile(join(workflowDir, testCase.filename), yaml);
      }

      const result = await discoverWorkflows(testDir, { loadDefaults: false });

      expect(result.workflows).toHaveLength(0);
      expect(result.errors).toHaveLength(invalidCases.length);

      for (const testCase of invalidCases) {
        expect(result.errors.some(error => error.error.includes(testCase.field))).toBe(true);
      }
      expect(result.errors.every(error => error.errorType === 'validation_error')).toBe(true);
    });
```

- [ ] **Step 3: Run loader tests and confirm the new tests fail**

Run:

```bash
bun test packages/workflows/src/loader.test.ts
```

Expected: the new valid-options test fails because `workflow.effort`, `workflow.thinking`, `workflow.fallbackModel`, `workflow.betas`, and `workflow.sandbox` are `undefined`. The invalid-options test may also fail because current loader behavior does not validate these fields.

- [ ] **Step 4: Commit the failing tests**

```bash
git add packages/workflows/src/loader.test.ts
git commit -m "test(workflows): cover workflow-level SDK option loading"
```

---

### Task 2: Preserve and Validate Workflow-Level Strict Options in Loader

**Files:**
- Modify: `packages/workflows/src/loader.ts`
- Test: `packages/workflows/src/loader.test.ts`

- [ ] **Step 1: Import `workflowBaseSchema`**

Change the workflow schema import in `packages/workflows/src/loader.ts` from:

```ts
import { modelReasoningEffortSchema, webSearchModeSchema } from './schemas/workflow';
```

to:

```ts
import {
  modelReasoningEffortSchema,
  webSearchModeSchema,
  workflowBaseSchema,
} from './schemas/workflow';
```

- [ ] **Step 2: Add a strict workflow option schema**

Add this near the top of `loader.ts`, after `getLog()`:

```ts
const strictWorkflowOptionsSchema = workflowBaseSchema.pick({
  effort: true,
  thinking: true,
  fallbackModel: true,
  betas: true,
  sandbox: true,
});

function formatWorkflowOptionIssue(issue: z.ZodIssue): string {
  const path = issue.path.length > 0 ? issue.path.join('.') : 'workflow';
  return `${path}: ${issue.message}`;
}
```

- [ ] **Step 3: Parse strict workflow options in `parseWorkflow()`**

Add this block after `additionalDirectories` is computed and before `interactive` is parsed:

```ts
    const strictWorkflowOptionsResult = strictWorkflowOptionsSchema.safeParse(raw);
    if (!strictWorkflowOptionsResult.success) {
      return {
        workflow: null,
        error: {
          filename,
          error: `Workflow-level option validation failed: ${strictWorkflowOptionsResult.error.issues
            .map(formatWorkflowOptionIssue)
            .join('; ')}`,
          errorType: 'validation_error',
        },
      };
    }
    const { effort, thinking, fallbackModel, betas, sandbox } = strictWorkflowOptionsResult.data;
```

- [ ] **Step 4: Include strict options in the returned workflow**

Change the returned `workflow` object so it includes the five parsed fields:

```ts
      workflow: {
        name: raw.name,
        description: raw.description,
        provider,
        model,
        modelReasoningEffort,
        webSearchMode,
        additionalDirectories,
        interactive,
        effort,
        thinking,
        fallbackModel,
        betas,
        sandbox,
        nodes: dagNodes,
        ...(worktreePolicy ? { worktree: worktreePolicy } : {}),
      },
```

- [ ] **Step 5: Run loader tests**

Run:

```bash
bun test packages/workflows/src/loader.test.ts
```

Expected: PASS.

- [ ] **Step 6: Commit loader implementation**

```bash
git add packages/workflows/src/loader.ts packages/workflows/src/loader.test.ts
git commit -m "fix(workflows): preserve workflow-level SDK options"
```

---

### Task 3: Add Executor Tests for Loaded Options and Codex Effective Config

**Files:**
- Modify: `packages/workflows/src/dag-executor.test.ts`

- [ ] **Step 1: Add YAML-loaded Claude option round-trip test**

Insert this test inside the `executeDagWorkflow -- Claude SDK advanced options` describe block, near the existing workflow-level effort tests:

```ts
  it('forwards YAML-loaded workflow-level effort to the provider node config', async () => {
    const parsed = parseWorkflow(
      `name: loaded-effort-test
description: Loaded workflow effort should reach execution
provider: claude
model: sonnet
effort: high
nodes:
  - id: step1
    command: my-cmd
`,
      'loaded-effort.yaml'
    );
    if (parsed.error) {
      throw new Error(parsed.error.error);
    }

    const mockDeps = createMockDeps();
    const platform = createMockPlatform();
    const workflowRun = makeWorkflowRun();

    await executeDagWorkflow(
      mockDeps,
      platform,
      'conv-dag',
      testDir,
      parsed.workflow,
      workflowRun,
      parsed.workflow.provider ?? 'claude',
      parsed.workflow.model,
      join(testDir, 'artifacts'),
      join(testDir, 'logs'),
      'main',
      'docs/',
      minimalConfig
    );

    expect(mockSendQueryDag.mock.calls.length).toBeGreaterThan(0);
    const optionsArg = mockSendQueryDag.mock.calls[0][3] as Record<string, unknown>;
    const nodeConfig = optionsArg.nodeConfig as Record<string, unknown>;
    expect(nodeConfig.effort).toBe('high');
  });
```

- [ ] **Step 2: Add Codex override test**

Insert this test in the same describe block, after the Claude-only Codex warning test or in a new `executeDagWorkflow -- Codex workflow config` describe block that uses the same setup:

```ts
  it('merges workflow-level Codex config over assistant defaults', async () => {
    mockGetAgentProviderDag.mockImplementation(() => ({
      sendQuery: mockSendQueryDag,
      getType: () => 'codex',
      getCapabilities: mockCodexCapabilities,
    }));

    const mockDeps = createMockDeps();
    const platform = createMockPlatform();
    const workflowRun = makeWorkflowRun();
    const config: WorkflowConfig = {
      ...minimalConfig,
      assistants: {
        claude: {},
        codex: {
          modelReasoningEffort: 'low',
          webSearchMode: 'cached',
          additionalDirectories: ['/assistant-default'],
        },
      },
    };

    await executeDagWorkflow(
      mockDeps,
      platform,
      'conv-dag',
      testDir,
      {
        name: 'codex-workflow-config',
        provider: 'claude',
        modelReasoningEffort: 'high',
        webSearchMode: 'live',
        additionalDirectories: ['/workflow-override'],
        nodes: [{ id: 'step1', command: 'my-cmd', provider: 'codex' }],
      },
      workflowRun,
      'claude',
      undefined,
      join(testDir, 'artifacts'),
      join(testDir, 'logs'),
      'main',
      'docs/',
      config
    );

    expect(mockSendQueryDag.mock.calls.length).toBeGreaterThan(0);
    const optionsArg = mockSendQueryDag.mock.calls[0][3] as Record<string, unknown>;
    const assistantConfig = optionsArg.assistantConfig as Record<string, unknown>;
    expect(assistantConfig.modelReasoningEffort).toBe('high');
    expect(assistantConfig.webSearchMode).toBe('live');
    expect(assistantConfig.additionalDirectories).toEqual(['/workflow-override']);
  });
```

- [ ] **Step 3: Add undefined-preservation test**

Insert this test after the override test:

```ts
  it('does not erase Codex assistant defaults when workflow values are undefined', async () => {
    mockGetAgentProviderDag.mockImplementation(() => ({
      sendQuery: mockSendQueryDag,
      getType: () => 'codex',
      getCapabilities: mockCodexCapabilities,
    }));

    const mockDeps = createMockDeps();
    const platform = createMockPlatform();
    const workflowRun = makeWorkflowRun();
    const config: WorkflowConfig = {
      ...minimalConfig,
      assistants: {
        claude: {},
        codex: {
          modelReasoningEffort: 'medium',
          webSearchMode: 'cached',
          additionalDirectories: ['/assistant-default'],
        },
      },
    };

    await executeDagWorkflow(
      mockDeps,
      platform,
      'conv-dag',
      testDir,
      {
        name: 'codex-partial-workflow-config',
        provider: 'claude',
        webSearchMode: 'live',
        nodes: [{ id: 'step1', command: 'my-cmd', provider: 'codex' }],
      },
      workflowRun,
      'claude',
      undefined,
      join(testDir, 'artifacts'),
      join(testDir, 'logs'),
      'main',
      'docs/',
      config
    );

    expect(mockSendQueryDag.mock.calls.length).toBeGreaterThan(0);
    const optionsArg = mockSendQueryDag.mock.calls[0][3] as Record<string, unknown>;
    const assistantConfig = optionsArg.assistantConfig as Record<string, unknown>;
    expect(assistantConfig.modelReasoningEffort).toBe('medium');
    expect(assistantConfig.webSearchMode).toBe('live');
    expect(assistantConfig.additionalDirectories).toEqual(['/assistant-default']);
  });
```

- [ ] **Step 4: Run executor tests and confirm new Codex tests fail**

Run:

```bash
bun test packages/workflows/src/dag-executor.test.ts
```

Expected: the YAML-loaded effort test should pass after Task 2. The Codex config tests should fail because `assistantConfig` currently uses only `config.assistants[provider]`.

- [ ] **Step 5: Commit failing executor tests**

```bash
git add packages/workflows/src/dag-executor.test.ts
git commit -m "test(workflows): cover Codex workflow config merging"
```

---

### Task 4: Apply Codex Workflow-Level Config in DAG Executor

**Files:**
- Modify: `packages/workflows/src/dag-executor.ts`
- Test: `packages/workflows/src/dag-executor.test.ts`

- [ ] **Step 1: Import Codex workflow config types**

Update the schema type import in `packages/workflows/src/dag-executor.ts` so it includes `ModelReasoningEffort` and `WebSearchMode`:

```ts
  WorkflowRun,
  EffortLevel,
  ThinkingConfig,
  SandboxSettings,
  ModelReasoningEffort,
  WebSearchMode,
```

- [ ] **Step 2: Extend `WorkflowLevelOptions`**

Replace the current `WorkflowLevelOptions` interface with:

```ts
/** Workflow-level provider options. Per-node overrides take precedence via ?? where applicable. */
interface WorkflowLevelOptions {
  effort?: EffortLevel;
  thinking?: ThinkingConfig;
  fallbackModel?: string;
  betas?: string[];
  sandbox?: SandboxSettings;
  modelReasoningEffort?: ModelReasoningEffort;
  webSearchMode?: WebSearchMode;
  additionalDirectories?: string[];
}
```

- [ ] **Step 3: Add a helper for effective assistant config**

Add this helper below `WorkflowLevelOptions`:

```ts
function buildEffectiveAssistantConfig(
  provider: string,
  providerAssistantConfig: Record<string, unknown> | undefined,
  workflowLevelOptions: WorkflowLevelOptions
): Record<string, unknown> {
  const assistantConfig: Record<string, unknown> = { ...(providerAssistantConfig ?? {}) };

  if (provider !== 'codex') {
    return assistantConfig;
  }

  if (workflowLevelOptions.modelReasoningEffort !== undefined) {
    assistantConfig.modelReasoningEffort = workflowLevelOptions.modelReasoningEffort;
  }
  if (workflowLevelOptions.webSearchMode !== undefined) {
    assistantConfig.webSearchMode = workflowLevelOptions.webSearchMode;
  }
  if (workflowLevelOptions.additionalDirectories !== undefined) {
    assistantConfig.additionalDirectories = workflowLevelOptions.additionalDirectories;
  }

  return assistantConfig;
}
```

- [ ] **Step 4: Use the helper in prompt-node provider resolution**

Replace this block in `resolveNodeProviderAndModel()`:

```ts
  // Pass assistantConfig from config - provider parses internally
  const assistantConfig = config.assistants[provider] ?? {};
```

with:

```ts
  // Pass effective assistantConfig - provider parses internally.
  const assistantConfig = buildEffectiveAssistantConfig(
    provider,
    providerAssistantConfig,
    workflowLevelOptions
  );
```

- [ ] **Step 5: Use the helper in loop-node options**

Replace this line in `buildLoopNodeOptions()`:

```ts
  options.assistantConfig = config.assistants[provider] ?? {};
```

with:

```ts
  options.assistantConfig = buildEffectiveAssistantConfig(
    provider,
    config.assistants[provider],
    workflowLevelOptions ?? {}
  );
```

- [ ] **Step 6: Include Codex fields when constructing `workflowLevelOptions`**

Change the `workflowLevelOptions` object inside `executeDagWorkflow()` to:

```ts
  const workflowLevelOptions = {
    effort: workflow.effort,
    thinking: workflow.thinking,
    fallbackModel: workflow.fallbackModel,
    betas: workflow.betas,
    sandbox: workflow.sandbox,
    modelReasoningEffort: workflow.modelReasoningEffort,
    webSearchMode: workflow.webSearchMode,
    additionalDirectories: workflow.additionalDirectories,
  };
```

- [ ] **Step 7: Run executor tests**

Run:

```bash
bun test packages/workflows/src/dag-executor.test.ts
```

Expected: PASS.

- [ ] **Step 8: Commit executor implementation**

```bash
git add packages/workflows/src/dag-executor.ts packages/workflows/src/dag-executor.test.ts
git commit -m "fix(workflows): apply Codex workflow-level config"
```

---

### Task 5: Document and Test Deterministic Gate Pattern

**Files:**
- Modify: `packages/workflows/src/dag-executor.test.ts`
- Create: `packages/docs-web/src/content/docs/guides/deterministic-gates.md`
- Modify: `packages/docs-web/src/content/docs/guides/index.md`

- [ ] **Step 1: Add a regression test for `trigger_rule: all_done` gate validation**

Add this test in `packages/workflows/src/dag-executor.test.ts` near the `executeDagWorkflow -- when condition parse errors` tests or in a new `executeDagWorkflow -- deterministic gates` describe block:

```ts
describe('executeDagWorkflow -- deterministic gates', () => {
  let testDir: string;

  beforeEach(async () => {
    testDir = join(
      tmpdir(),
      `dag-gate-test-${Date.now()}-${Math.random().toString(36).slice(2)}`
    );
    const commandsDir = join(testDir, '.archon', 'commands');
    await mkdir(commandsDir, { recursive: true });
    await writeFile(join(commandsDir, 'review.md'), 'Review the artifact.');

    mockSendQueryDag.mockClear();
    mockGetAgentProviderDag.mockClear();
    mockLogFn.mockClear();

    mockGetAgentProviderDag.mockImplementation(() => ({
      sendQuery: mockSendQueryDag,
      getType: () => 'claude',
      getCapabilities: mockClaudeCapabilities,
    }));
  });

  afterEach(async () => {
    await rm(testDir, { recursive: true, force: true });
  });

  it('runs validation with all_done and emits GATE_INVALID when reviewer fails', async () => {
    mockSendQueryDag.mockImplementation(function* () {
      yield {
        type: 'result',
        isError: true,
        errorSubtype: 'error_during_execution',
        errors: ['reviewer crashed'],
        sessionId: 'failed-review',
      };
    });

    const store = createMockStore();
    const mockDeps = createMockDeps(store);
    const platform = createMockPlatform();
    const workflowRun = makeWorkflowRun();

    await executeDagWorkflow(
      mockDeps,
      platform,
      'conv-dag',
      testDir,
      {
        name: 'deterministic-gate',
        nodes: [
          { id: 'spec-review', command: 'review' },
          {
            id: 'spec-validate',
            depends_on: ['spec-review'],
            trigger_rule: 'all_done',
            bash: `set -euo pipefail
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
issues = data.get("blocking_issues")
if data.get("verdict") == "APPROVED" and isinstance(issues, list) and len(issues) == 0:
    print("GATE_APPROVED")
else:
    print("GATE_REJECTED")
PY`,
          },
          {
            id: 'plan-author',
            depends_on: ['spec-validate'],
            when: "$spec-validate.output == 'GATE_APPROVED'",
            prompt: 'Author the plan.',
          },
        ],
      },
      workflowRun,
      'claude',
      undefined,
      join(testDir, 'artifacts'),
      join(testDir, 'logs'),
      'main',
      'docs/',
      minimalConfig
    );

    const eventCalls = (store.createWorkflowEvent as ReturnType<typeof mock>).mock.calls;
    const validateCompleted = eventCalls.find((call: unknown[]) => {
      const event = call[0] as Record<string, unknown>;
      return event.event_type === 'node_completed' && event.step_name === 'spec-validate';
    });
    expect(validateCompleted).toBeDefined();
    const validateData = (validateCompleted?.[0] as Record<string, unknown>).data as Record<
      string,
      unknown
    >;
    expect(validateData.node_output).toBe('GATE_INVALID');

    const planSkipped = eventCalls.find((call: unknown[]) => {
      const event = call[0] as Record<string, unknown>;
      return event.event_type === 'node_skipped' && event.step_name === 'plan-author';
    });
    expect(planSkipped).toBeDefined();
  });
});
```

- [ ] **Step 2: Run the deterministic gate test**

Run:

```bash
bun test packages/workflows/src/dag-executor.test.ts
```

Expected: PASS. If this fails because of test placement or mock state, keep the behavior unchanged and move this new describe block into its own `bun test` invocation in `packages/workflows/package.json`, following the mock isolation rules in `CLAUDE.md`.

- [ ] **Step 3: Create deterministic gates guide**

Create `packages/docs-web/src/content/docs/guides/deterministic-gates.md` with this content:

````md
---
title: Deterministic Gates
description: Build hard review gates that advance only on machine-checkable workflow output.
category: guides
area: workflows
audience: [user]
status: current
sidebar:
  order: 4
---

Deterministic gates let a workflow use AI judgment without letting raw model text decide whether the workflow advances. A reviewer node can produce the judgment, but a bash or script node should validate the result and emit one exact token for downstream `when:` conditions.

Use this pattern when a workflow must enforce boundaries such as spec approval before planning, plan approval before coding, or quality passing before review.

## Gate Shape

```text
author node
  -> reviewer node with output_format
  -> validation node with trigger_rule: all_done
  -> downstream nodes that branch on validation output
  -> optional approval halt
```

The validation node uses `trigger_rule: all_done` so it still runs when the reviewer crashes or returns an SDK error. In that case `$review.output` is empty, and the validator should emit `GATE_INVALID`.

## Reviewer Contract

Give reviewer nodes a strict `output_format` with fields that code can validate:

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
```

## Validation Node

The validator must fail closed. It should treat empty output, malformed JSON, missing fields, non-approved verdicts, and blocking issues as not approved.

```yaml
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
```

`$spec-review.output` is shell-quoted by Archon before the bash script runs, so the JSON arrives as one argument to Python.

## Branching

Branch on the validator output, not on reviewer prose:

```yaml
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

## Loop Policy

Do not use a model-emitted `until` signal as the only hard gate. If a loop controls progression, pair it with `until_bash` or a downstream validation node.

For quality loops, prefer `until_bash` that reruns the actual quality command:

```yaml
- id: quality-fix-loop
  provider: claude
  loop:
    prompt: |
      Fix the quality failures. Output QUALITY_FIXED after the checks pass.
    until: QUALITY_FIXED
    max_iterations: 3
    until_bash: |
      bun run validate
```

The model signal is useful for communication. The command exit code is the gate.
````

- [ ] **Step 4: Link the new guide from guides index**

Add this bullet under `## Workflow Authoring` in `packages/docs-web/src/content/docs/guides/index.md`:

```md
- [Deterministic Gates](/guides/deterministic-gates/) - Build hard review gates from structured reviewer output and validation nodes
```

- [ ] **Step 5: Run docs formatting check for changed docs**

Run:

```bash
bun x prettier --check packages/docs-web/src/content/docs/guides/deterministic-gates.md packages/docs-web/src/content/docs/guides/index.md
```

Expected: PASS.

- [ ] **Step 6: Commit gate docs and regression test**

```bash
git add packages/workflows/src/dag-executor.test.ts packages/docs-web/src/content/docs/guides/deterministic-gates.md packages/docs-web/src/content/docs/guides/index.md
git commit -m "docs(workflows): document deterministic gate pattern"
```

---

### Task 6: Final Verification

**Files:**
- Verify only unless failures require fixes.

- [ ] **Step 1: Run workflow package tests**

```bash
bun --filter @archon/workflows test
```

Expected: PASS.

- [ ] **Step 2: Run provider package tests**

```bash
bun --filter @archon/providers test
```

Expected: PASS.

- [ ] **Step 3: Run focused type checks**

```bash
bun --filter @archon/workflows type-check
bun --filter @archon/providers type-check
```

Expected: PASS. If TypeScript cannot resolve `bun-types`, run `bun install` once from the repo root, then rerun both commands.

- [ ] **Step 4: Run full repository validation**

```bash
bun run validate
```

Expected: PASS.

- [ ] **Step 5: Inspect final diff**

```bash
git status --short
git diff --stat
git diff -- packages/workflows/src/loader.ts packages/workflows/src/dag-executor.ts
```

Expected: only the planned files are modified. Existing untracked draft files may remain untracked and should not be included unless the user explicitly requests it.

- [ ] **Step 6: Commit verification-only fixes if needed**

Only run this if final verification required small fixes:

```bash
git add packages/workflows/src packages/docs-web/src/content/docs/guides
git commit -m "chore(workflows): finalize orchestration hardening"
```

---

## Self-Review

Spec coverage:

- Loader drops `effort`, `thinking`, `fallbackModel`, `betas`, `sandbox`: covered by Tasks 1 and 2.
- Codex workflow config not applied: covered by Tasks 3 and 4, including undefined-preserving merge behavior.
- Deterministic gate pattern: covered by Task 5 with both a regression test and a dedicated guide.
- Test isolation note: covered in Task 5 Step 2 and final verification commands.
- No new `gate:` primitive, no auth hardening, no Codex read-only sandbox: preserved by this plan.

Placeholder scan:

- The plan contains no deferred implementation markers.
- Every code-changing task includes concrete snippets, commands, and expected results.

Type consistency:

- `WorkflowLevelOptions` carries `ModelReasoningEffort`, `WebSearchMode`, and `additionalDirectories`.
- `buildEffectiveAssistantConfig()` is used by both prompt-node resolution and loop-node option construction.
- Tests inspect `optionsArg.nodeConfig` and `optionsArg.assistantConfig`, matching current `SendQueryOptions` usage.
