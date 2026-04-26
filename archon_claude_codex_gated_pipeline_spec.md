# SPEC: Archon-Based Claude Author + Codex Reviewer Gated Delivery Pipeline

**Status:** Draft v2 (rewritten against Archon's actual workflow schema)
**Target repo type:** Existing brownfield AI-driven cybersecurity product repo
**Primary goal:** Use Archon to orchestrate a deterministic spec → plan → code workflow where Claude authors artefacts and Codex independently reviews each artefact at hard gates, with revision loops and human halt points.

> v2 supersedes v1 because v1 used YAML keys (`type:`, `agent:`, `output:`, `readonly:`, `{{var}}`) that do not exist in Archon and modelled loops with a shape Archon does not support. The pipeline goal is unchanged; the implementation has been re-grounded in `packages/workflows/src/schemas/*.ts` (DAG node, loop, workflow) and the `archon-adversarial-dev.yaml` reference workflow.

---

## 1. Background and rationale

The desired workflow is a controlled AI software-delivery factory, not free-form multi-agent chat. The system must enforce progression rules externally:

```text
spec approved before plan
plan approved before code
quality + code review approved before next phase/spec
```

Archon's DAG workflow engine fits this shape: it supports `nodes` with `depends_on` edges, per-node `provider` (Claude vs Codex), structured-JSON output via `output_format`, conditional progression via `when:` and `trigger_rule:`, human-halt approval nodes (`approval:` with `on_reject`), and bash nodes for deterministic checks. What it does **not** support is also important and shapes this spec:

- No `{{var}}` substitution — only `$VAR` (`$1..$9`, `$ARGUMENTS`, `$ARTIFACTS_DIR`, `$WORKFLOW_ID`, `$BASE_BRANCH`, `$REJECTION_REASON`, and `$nodeId.output[.field]`).
- No node-level `output:` file path — node output is the AI/stdout text, addressable as `$nodeId.output`. Files must be written by the node's prompt or bash directly.
- No `readonly:` flag for Codex — `allowed_tools`/`denied_tools` apply only to Claude (`packages/providers/src/codex/capabilities.ts:9` declares `toolRestrictions: false`). Read-only is enforced **externally**, by snapshotting git state before every Codex node and aborting if the working tree or HEAD changed afterwards.
- A `loop:` node executes one prompt under one provider repeatedly until a completion signal — it cannot interleave Claude (author) and Codex (reviewer) within a single loop. Cross-provider revision must be expressed as **explicit pre-declared attempts** in the DAG, gated with `when:` against the previous attempt's verdict.
- No automatic global attempt counter (`{{attempt}}`). Attempts are encoded as suffixes on the node IDs (`-author-1`, `-author-2`, …) and the directory layout (`attempt-1/`, `attempt-2/`, …). State across attempts lives in a state JSON file the workflow itself maintains.
- No upstream-gate-reopen / cascade-staleness mechanism. v2 drops `reopen_gate` semantics from the engine layer; if Codex thinks the spec is wrong while reviewing the plan, the workflow halts at the plan gate's approval node and the operator restarts the workflow with revised inputs.

**Source references:**

- Archon repo: `/Users/moare/source/Archon`
- DAG schema: `packages/workflows/src/schemas/dag-node.ts`
- Loop schema: `packages/workflows/src/schemas/loop.ts`
- Workflow base schema: `packages/workflows/src/schemas/workflow.ts`
- Reference workflow with state-machine pattern: `.archon/workflows/defaults/archon-adversarial-dev.yaml`
- Variable substitution: `packages/workflows/src/utils/variable-substitution.ts`, `packages/workflows/src/executor-shared.ts:282–334`

---

## 2. Scope

### 2.1 In scope

1. Brownfield context generation (Gate 0).
2. Spec drafting by Claude (Gate 1).
3. Spec review by Codex (Gate 1).
4. Spec revision loops, up to 3 attempts, then human halt.
5. Plan drafting by Claude (Gate 2).
6. Plan review by Codex (Gate 2).
7. Plan revision loops, up to 3 attempts, then human halt.
8. Code implementation by Claude (Gate 3).
9. Mechanical quality checks via bash node (Gate 3).
10. Code review by Codex (Gate 4).
11. Code fix/review loops, up to 3 attempts, then human halt.
12. Human halt-and-resume via Archon's `approval` nodes and `/workflow approve|reject <run-id>`.
13. Versioned artefacts under `.factory/runs/<feature-id>/`.
14. Git audit trail via per-attempt commits authored by Claude.
15. Snapshot/diff guard around every Codex node to enforce read-only.

### 2.2 Out of scope for v2

1. Any feature requiring backward DAG edges (e.g. CODE_REVIEW reopening SPEC). v1's `reopen_gate` mechanism is removed — operators restart the workflow when an upstream artefact is wrong.
2. `make quality` as a hard requirement. v2 prefers `make quality` if available and falls back to the project's `bun run validate`. Either path emits `QUALITY_PASS` or `QUALITY_FAIL` to a downstream node via stdout.
3. Telegram, Slack, or external notification integration.
4. Multi-developer coordination.
5. Full CI/CD replacement.
6. Fully autonomous merge to main.
7. Long-running production daemon mode.
8. Durable workflow engine replacement such as Temporal or Prefect.
9. Allowing Codex to author or mutate code (enforced by the snapshot guard).
10. Allowing Claude to approve its own outputs (enforced by per-node `provider` and human halt).

---

## 3. Actors and responsibilities

Unchanged from v1 (sections 3.1–3.4). Provider mapping in YAML: `provider: claude` for author nodes, `provider: codex` for reviewer nodes.

---

## 4. High-level workflow

```text
START
  ↓
init (bash) — write .factory/runs/<feature-id>/state/pipeline.json
  ↓
brownfield-author-1 (claude) → snapshot → review-1 (codex) → mutation-check → decide-1
  ↓ if rejected
brownfield-author-2 (claude) → snapshot → review-2 (codex) → mutation-check → decide-2
  ↓ if rejected
brownfield-author-3 (claude) → snapshot → review-3 (codex) → mutation-check → decide-3
  ↓
brownfield-final → if not APPROVED: brownfield-halt (approval node)
  ↓ APPROVED
spec-author-1 → … (same 5-node-per-attempt × 3 attempts pattern)
  ↓ APPROVED
plan-author-1 → … (same pattern)
  ↓ APPROVED
code-implement (claude) → quality (bash) → code-quality-fix-loop (claude loop, 3 iters)
  ↓ QUALITY_PASS
code-review-snapshot → code-review-1 (codex) → code-review-mutation-check
  ↓
code-review-fix-loop (claude loop, 3 iters)
  ↓
code-final → if not APPROVED: code-halt (approval node)
  ↓
finalise (bash) → done
```

**Why pre-declared attempts instead of one revision loop:** Archon's `loop:` node has a single `provider:`. To alternate Claude and Codex per iteration, attempts must be declared as discrete DAG nodes. The cost is verbose YAML; the gain is true model independence per attempt.

**Why a Claude-only loop is acceptable for code-fix and quality-fix:** at those points the reviewer's verdict already exists as a frozen artefact (`review.json` / `quality.log`). The fix loop just needs Claude to address it deterministically; Codex re-review is a separate node downstream.

---

## 5. Required gates

The acceptance criteria for each gate (sections 5.1–5.5 in v1) are unchanged. The implementation now uses these mechanisms:

| Concern                   | v1 (broken)                                           | v2 (works)                                                                                                                                                                                                      |
| ------------------------- | ----------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Per-attempt artefact path | `output: …/attempt-{{attempt}}/artifact.md` on a node | Author prompt instructs Claude to `Write` to `.factory/runs/$1/<gate>/attempt-N/artifact.md`; N is a literal in the node ID (`-author-1`, `-author-2`, …)                                                       |
| Reviewer JSON             | Implicit                                              | `provider: codex` + `output_format:` JSON schema; the schema is duplicated on each Codex node (Archon has no `$ref` support)                                                                                    |
| Reviewer is read-only     | `readonly: true` on the node                          | Bash `snapshot` node before the Codex node (records `git rev-parse HEAD` and `git status --porcelain`) + bash `mutation-check` node after it that aborts the run if either changed                              |
| Quality gate              | `bash: make quality` only                             | Bash node detects `make quality` then falls back to `bun run validate`, tees output to `.factory/runs/$1/code/attempt-N/quality.log`, prints `QUALITY_PASS` / `QUALITY_FAIL` / `QUALITY_INFRASTRUCTURE_MISSING` |
| Quality fix loop          | Implicit                                              | `loop:` node with `until: QUALITY_FIXED` and `until_bash:` re-running the quality command — single provider (Claude) is fine here                                                                               |
| Human halt                | Sentinel file `.factory/HUMAN_INPUT_REQUIRED.md`      | First-class `approval:` node with `capture_response: true`; the operator runs `/workflow approve <run-id> [feedback]` or `/workflow reject <run-id> "<reason>"`                                                 |

---

## 6. Reviewer output contract

Unchanged from v1 (verdicts, rejection types, JSON shape). Implementation note: the `output_format` JSON schema is **inline on every Codex node** because Archon does not resolve `$ref`. To keep the YAML manageable, define the review schema once as YAML anchors (`&review-schema` / `*review-schema`) — the YAML loader resolves anchors before Archon parses.

Example (anchor at the top of `nodes:`):

```yaml
nodes:
  - &codex-review-schema
    output_format: &review_schema
      type: object
      required: [verdict, blocking_issues, required_changes, confidence]
      properties:
        verdict: { type: string, enum: [APPROVED, REJECTED, NEEDS_HUMAN] }
        # … rest of the schema
```

Then later: `output_format: *review_schema`.

---

## 7. Revision loop policy

### 7.1 Max attempts

```yaml
gates:
  brownfield_context:
    max_attempts: 3
  spec:
    max_attempts: 3 # was 5 in v1; lowered because each attempt is 5 declared YAML nodes
  plan:
    max_attempts: 3
  code_quality:
    max_attempts: 3 # implemented as loop max_iterations
  code_review:
    max_attempts: 3 # implemented as loop max_iterations
```

### 7.2 Convergence rules

The workflow must halt for human input if:

- The third auto-attempt at any gate is not `APPROVED`. (The `<gate>-final` node returns `NEEDS_HUMAN` and the `<gate>-halt` approval node pauses.)
- Any Codex node returns verdict `NEEDS_HUMAN` regardless of attempt count. (Same path: decide node propagates, `<gate>-final` returns `NEEDS_HUMAN`.)
- Any `mutation-check` node detects that Codex modified the working tree. (FATAL — workflow exits non-zero, not a halt.)
- The quality command is missing entirely. (`quality` node prints `QUALITY_INFRASTRUCTURE_MISSING` and the rest of the gate is skipped via `when:`.)

Repeated-blocker detection (same blocker ID in 3 consecutive `review.json` files) is implemented inside the bash decide nodes by reading prior `review.json` files and short-circuiting to `NEEDS_HUMAN` if the same `blocking_issues[*].id` reappears.

### 7.3 Narrow revision rule

Each author-N prompt for N>1 must contain literal text equivalent to:

> Read `.factory/runs/$1/<gate>/attempt-(N-1)/review.json`. Address every blocking issue. Do NOT rewrite unrelated sections. Do NOT expand scope. Preserve approved decisions. Output a one-paragraph revision summary at `.factory/runs/$1/<gate>/attempt-N/revision-summary.md`. If feedback requires human judgement, output `NEEDS_HUMAN` and stop.

---

## 8. Upstream reopen rules — REMOVED

v1 §8 described an upstream reopen mechanism with cascade staleness. Archon does not support backward DAG edges or cascade-staleness markers. v2 removes this concept. If Codex returns `reopen_gate: SPEC` while reviewing the plan, the decide bash node treats it as `NEEDS_HUMAN` and the operator restarts the whole workflow from `init` with revised inputs. The `reopen_gate` field remains in the reviewer JSON contract for documentation purposes only.

---

## 9. Artefact storage

### 9.1 Canonical artefacts (in repo, committed)

```text
.specify/spec.md
.specify/plan.md
.specify/tasks.md
```

### 9.2 Attempt history (in repo, committed)

```text
.factory/runs/<feature-id>/
  brownfield/attempt-001/{artifact.md, review.json, revision-summary.md}
  spec/attempt-001/{artifact.md, review.json, revision-summary.md}
  plan/attempt-001/{artifact.md, review.json, revision-summary.md}
  code/attempt-001/{diff.patch, quality.log, review.json, revision-summary.md}
  state/pipeline.json
```

### 9.3 State file

`.factory/runs/<feature-id>/state/pipeline.json` is written by the `init` bash node and updated by every `*-decide-N` bash node. Atomic write pattern:

```bash
python3 - "$STATE" "$VERDICT" "$ATTEMPT" <<'PY'
import json, sys, os
path, verdict, attempt = sys.argv[1], sys.argv[2], int(sys.argv[3])
with open(path) as f: s = json.load(f)
s["attempt"] = attempt
s["gate_state"] = "APPROVED" if verdict == "APPROVED" else "REVISION_REQUIRED" if verdict == "REJECTED" else "NEEDS_HUMAN"
tmp = path + ".tmp"
with open(tmp, "w") as f: json.dump(s, f, indent=2)
os.replace(tmp, path)
PY
```

`os.replace` is atomic on POSIX.

### 9.4 Workflow run state

Independent of the file above, Archon writes its own per-run state to the database (`workflow_runs` table). The file is for human auditability and reviewer-prompt context; the database is for engine semantics. Don't try to merge the two.

---

## 10. Git policy

### 10.1 Branching

Use one feature branch per feature/change: `archon/<feature-id>`. The workflow pins `worktree.enabled: true` so each run gets its own isolated checkout (required for the snapshot guard to compare against a stable HEAD).

### 10.2 Commits

Claude commits at every successful author attempt. Each commit message follows:

```text
<gate> attempt <N>: <verdict>
```

Example: `spec attempt 002: approved`. Codex review summaries are stored as files (`.factory/runs/.../review.json`), not in commit bodies.

### 10.3 Snapshot guard commits

The snapshot/mutation-check pair around every Codex node intentionally records `git status --porcelain` rather than a full diff. If Codex creates a commit _or_ modifies the working tree, the post-check fails the run.

---

## 11. Archon workflow design

### 11.1 Workflow location

```text
.archon/workflows/claude-codex-gated-delivery.yaml
```

### 11.2 Concrete YAML (starter — see file)

The complete starter file lives at `.archon/workflows/claude-codex-gated-delivery.yaml`. It demonstrates:

- The `init` bash node (argument parsing, directory tree, state.json write).
- The full 5-node-per-attempt pattern for the **brownfield** gate (attempts 1 and 2 spelled out, attempt 3 templated as a comment).
- The `<gate>-final` + `<gate>-halt` approval pattern.
- The complete code phase: `code-implement` → `quality` → `code-quality-fix-loop` (loop) → `code-review-snapshot` → `code-review-1` (codex with full `output_format`) → `code-review-mutation-check` → `code-review-fix-loop` (loop) → `code-final` → `code-halt`.

The **spec** and **plan** gates replicate the brownfield pattern node-for-node — only the directory name (`spec/` vs `plan/`), the author prompt, and the reviewer's approval criteria change. Cloning the brownfield block once gates 0 + 3 + 4 work end-to-end is the recommended path: each clone is mechanical.

### 11.3 Per-attempt structure (canonical)

```yaml
- id: <gate>-author-N
  depends_on: [<gate>-decide-(N-1)]   # for N=1, depends_on the prior gate's -final
  when: "$<gate>-decide-(N-1).output != 'APPROVED'"   # omitted for N=1
  provider: claude
  allowed_tools: [Read, Write, Glob, Grep, Bash]
  prompt: |
    # author or revise; write to .factory/runs/$1/<gate>/attempt-N/artifact.md

- id: <gate>-snapshot-N
  depends_on: [<gate>-author-N]
  when: …
  bash: |
    git rev-parse HEAD > "$ARTIFACTS_DIR/codex-pre-sha-<gate>N.txt"
    git status --porcelain > "$ARTIFACTS_DIR/codex-pre-status-<gate>N.txt"

- id: <gate>-review-N
  depends_on: [<gate>-snapshot-N]
  when: …
  provider: codex
  output_format: *review_schema   # YAML anchor — same schema everywhere
  prompt: |
    # read attempt-N/artifact.md, return strict JSON

- id: <gate>-mutation-check-N
  depends_on: [<gate>-review-N]
  when: …
  bash: |
    [ "$(cat $ARTIFACTS_DIR/codex-pre-sha-<gate>N.txt)" = "$(git rev-parse HEAD)" ] \
      || { echo "FATAL: Codex committed during review"; exit 1; }
    diff -q "$ARTIFACTS_DIR/codex-pre-status-<gate>N.txt" <(git status --porcelain) \
      || { echo "FATAL: Codex modified working tree"; exit 1; }

- id: <gate>-decide-N
  depends_on: [<gate>-mutation-check-N]
  when: …
  bash: |
    # write review.json to attempt-N/, update state.json, echo verdict
```

### 11.4 Per-gate structure (canonical)

After the N attempt blocks:

```yaml
- id: <gate>-final
  depends_on: [<gate>-decide-1, <gate>-decide-2, <gate>-decide-3]
  trigger_rule: all_done
  bash: |
    for V in '$<gate>-decide-1.output' '$<gate>-decide-2.output' '$<gate>-decide-3.output'; do
      [ "$V" = "APPROVED" ] && { echo APPROVED; exit 0; }
    done
    echo NEEDS_HUMAN

- id: <gate>-halt
  depends_on: [<gate>-final]
  when: "$<gate>-final.output != 'APPROVED'"
  approval:
    message: '<gate> failed for $1. Approve to override or reject to abort.'
    capture_response: true
    on_reject:
      prompt: 'Halt. Reviewer rejected with: $REJECTION_REASON'
      max_attempts: 1
```

The next gate's first author node depends on `<gate>-final` with `when: "$<gate>-final.output == 'APPROVED'"`.

---

## 12. Prompt templates

Inline in the YAML for v2. Externalising prompts as `.archon/prompts/*.md` is supported (use `command:` nodes that point to `.archon/commands/<name>.md`), but inline keeps the workflow self-contained and easier to validate. Externalise once the inline prompts stabilise.

### 12.1 Shared reviewer instruction (prepend to every Codex prompt)

```text
You are the independent reviewer only. You are read-only.
Do not Write, Edit, or modify any file. Do not run destructive commands.
Do not commit. Do not push.
Return strict JSON only, conforming to the response schema.
Approve only when all blocking criteria are satisfied.
Return NEEDS_HUMAN when a decision requires human judgement or the reviewer is uncertain.
```

### 12.2 Shared author revision instruction (prepend to every claude `-author-N` prompt for N>1)

```text
You are revising a previously reviewed artefact.
Read the prior review at .factory/runs/$1/<gate>/attempt-(N-1)/review.json.
Address every blocking issue. Do not expand scope. Do not rewrite unrelated sections.
Preserve approved decisions.
If the feedback requires human judgement, stop and output NEEDS_HUMAN.
Produce a revision summary at .factory/runs/$1/<gate>/attempt-N/revision-summary.md.
```

---

## 13. Human halt/resume behaviour

v2 replaces v1's sentinel-file mechanism with Archon's first-class `approval` nodes. Mechanism:

1. When `<gate>-final` returns anything other than `APPROVED`, the `<gate>-halt` node fires.
2. Archon transitions the run to `paused` state and surfaces a message to the active platform adapter (CLI, web, Slack, Telegram).
3. Operator resumes via:
   - `/workflow approve <run-id> [optional override note]` — proceeds to the next gate as if the artefact were approved.
   - `/workflow reject <run-id> "<reason>"` — runs `on_reject.prompt` (a one-shot Claude node that just halts cleanly), then ends the run.
4. The run-id is the workflow run UUID (visible via `archon workflow status` or the web UI).

The capture_response on the approval node stores the operator's note as `$<gate>-halt.output`, available to downstream nodes if needed.

---

## 14. Security and safety requirements

1. Codex review nodes are made read-only externally via the snapshot/mutation-check guard. There is no engine-level read-only flag; the guard is the contract.
2. The mutation-check compares both `git rev-parse HEAD` and `git status --porcelain` before/after. Either changing fails the run with a FATAL message.
3. Destructive commands (`git push --force`, `git clean -fd`, `rm -rf` outside `$ARTIFACTS_DIR`) are disallowed by convention; `denied_tools` on Claude nodes can enforce a subset (`Bash(git push*)`, `Bash(git clean*)`, `Bash(rm*)`).
4. Authentication, authorisation, audit logging, and data-handling changes require explicit reviewer attention — encoded as approval criteria in the spec/code reviewer prompts.
5. The workflow must not exfiltrate secrets into artefacts, prompts, or review files.
6. `.env`, credentials, API tokens, and secrets must not appear in `.factory/runs/`. Quality logs must be scrubbed before commit (a `grep -E '(api[_-]?key|password|token|secret)'` check in the quality bash node, failing the run if any match is found).
7. Worktree isolation (`worktree.enabled: true`) protects the main checkout from any accidental Codex mutation.

---

## 15. Acceptance criteria

### 15.1 Workflow acceptance

- [ ] `.archon/workflows/claude-codex-gated-delivery.yaml` loads under `bun run cli validate workflows claude-codex-gated-delivery`.
- [ ] Running on an existing repo with a trivial feature produces all four gates' artefacts under `.factory/runs/<feature-id>/`.
- [ ] Brownfield pack is produced before the spec gate runs.
- [ ] Spec gate runs at most 3 attempts, then halts at `spec-halt` if not APPROVED.
- [ ] Plan gate runs at most 3 attempts, then halts at `plan-halt` if not APPROVED.
- [ ] Code gate runs `make quality` (or `bun run validate` fallback) before code review.
- [ ] Code gate's quality-fix loop runs at most 3 iterations.
- [ ] Code gate's review-fix loop runs at most 3 iterations.
- [ ] Reviewer JSON validates against the inline `output_format` schema (the SDK enforces this; malformed JSON fails the node and propagates).
- [ ] Approval gates pause the run cleanly; `/workflow approve|reject <run-id>` resumes correctly.
- [ ] Every Codex node is bracketed by `snapshot` + `mutation-check`; if Codex mutates state, the run aborts FATAL.
- [ ] Canonical `.specify/spec.md` and `.specify/plan.md` are updated only after a gate's `*-final` node returns `APPROVED`.

### 15.2 Review contract acceptance

- [ ] Malformed reviewer JSON fails the gate (SDK-level enforcement).
- [ ] `verdict: APPROVED` advances; the next gate's first node fires.
- [ ] `verdict: REJECTED` triggers the next attempt via `when:` gating.
- [ ] `verdict: NEEDS_HUMAN` short-circuits to the gate's `-halt` approval node.
- [ ] A repeated blocker ID across 3 attempts is detected by the `decide-3` bash node and forces `NEEDS_HUMAN` regardless of the third reviewer verdict.

### 15.3 Code quality acceptance

- [ ] `make quality` exists OR the bash node falls back to `bun run validate`; if neither is found, the run halts with `QUALITY_INFRASTRUCTURE_MISSING`.
- [ ] Code review does not run if the quality gate failed and the fix loop exhausted its iterations.
- [ ] Quality logs are stored at `.factory/runs/$1/code/attempt-N/quality.log`.
- [ ] Codex receives the diff and quality log paths in its review prompt.

---

## 16. Implementation plan

### Phase 1: Repo preparation

1. Confirm `bun run validate` passes on the current `dev` branch.
2. Confirm `.factory/` is git-tracked (or add to git).
3. Confirm `worktree.baseBranch` and the Codex CLI binary are configured in `.archon/config.yaml`.

### Phase 2: Workflow skeleton

1. Land `.archon/workflows/claude-codex-gated-delivery.yaml` (starter file already in place).
2. Run `bun run cli validate workflows claude-codex-gated-delivery` and fix any schema errors.
3. Replace the inline `output_format` blocks with a single YAML anchor (`*review_schema`).

### Phase 3: Fill in the spec + plan gates

1. Clone the brownfield gate's 17 nodes for `spec`, adjusting prompts and paths.
2. Clone again for `plan`.
3. Wire `spec-author-1` to `brownfield-final` with `when: APPROVED`, etc.

### Phase 4: Trial run

1. Pick a low-risk feature (e.g. "rename CLI flag X to Y").
2. Invoke `archon run claude-codex-gated-delivery rename-x-to-y "Rename --x flag to --y across CLI and docs"`.
3. Verify each gate's artefacts, the snapshot guard, and at least one approval-halt path.

### Phase 5: Harden

1. Tighten Claude `denied_tools` (block `git push`, `git clean`, `rm` outside `$ARTIFACTS_DIR`).
2. Add the secret-scrubbing grep to the quality node.
3. Add CI validation that runs `bun run cli validate workflows` on every PR.

---

## 17. Risks and mitigations

| Risk                                              | Impact | Mitigation                                                                               |
| ------------------------------------------------- | -----: | ---------------------------------------------------------------------------------------- |
| Archon YAML schema drifts                         | Medium | CI runs `bun run cli validate workflows` on every PR.                                    |
| Codex mutates the repo despite prompt instruction |   High | Snapshot/mutation-check guard around every Codex node.                                   |
| Claude expands scope during revision              | Medium | Narrow revision instruction (§12.2); reviewer flags scope creep as blocker.              |
| Review loops never converge                       | Medium | Hard cap at 3 attempts per gate, then human halt.                                        |
| Quality target missing                            |   High | Bash node falls back to `bun run validate`; halts with explicit error if neither exists. |
| Brownfield context incomplete                     |   High | Codex must approve before spec drafting; same gate pattern as the rest.                  |
| Secrets in logs                                   |   High | Secret-scrub grep on `quality.log` before commit (added in Phase 5).                     |
| Cross-provider revision inside a single loop      | Medium | Engine cannot do this; v2 uses pre-declared attempts as the supported alternative.       |

---

## 18. Open implementation questions

1. Should `.factory/runs/` be committed entirely, or should large quality logs be truncated/summarised?
2. Should the workflow auto-create the feature branch (`archon/<feature-id>`) or require it to exist?
3. Should the `finalise` node open a PR via `gh pr create`, or stop before PR creation?
4. Should the quality bash node fail the run on stderr noise even if exit code is 0?

---

## 19. Recommended v2 decision defaults

```yaml
feature_branch: archon/<feature-id>
worktree_enabled: true
commit_every_attempt: true
store_reviews_as_files: true
external_notifications: false
quality_command_preference: [make quality, bun run validate]
codex_mode: snapshot-guarded reviewer
claude_mode: author + fix loops
max_attempts:
  brownfield_context: 3
  spec: 3
  plan: 3
  code_quality: 3
  code_review: 3
```

---

## 20. Definition of done

This work is complete when an operator can run a single Archon workflow for a brownfield repo feature and observe the following enforced sequence:

```text
brownfield context drafted → Codex approved (or human halt at brownfield-halt)
spec drafted/revised → Codex approved (or human halt at spec-halt)
plan drafted/revised → Codex approved (or human halt at plan-halt)
code implemented/fixed → quality passed → Codex approved (or human halt at code-halt)
finalise prints commit summary
```

At no point may Claude advance to the next stage without the required Codex approval or quality result. Any Codex node that mutates the working tree fails the run FATAL. If a design decision is required, the workflow halts at an approval node, captures the operator's response via `/workflow approve|reject`, and either overrides or aborts.
