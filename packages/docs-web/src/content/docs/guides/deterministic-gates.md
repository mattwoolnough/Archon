---
title: Deterministic Gates
description: Build fail-closed review gates using existing workflow primitives — structured reviewer output, bash validation nodes, and deterministic branching tokens.
category: guides
area: workflows
audience: [user]
status: current
sidebar:
  order: 5
---

A **deterministic gate** is a workflow pattern where the decision to advance — or
halt — is made by code, not by reading raw model prose. The model provides
judgment; a bash validation node converts that judgment into a single
machine-readable token; downstream `when:` conditions branch only on that token.

This guide shows how to build a hard review gate using `output_format`, `bash`,
`when`, and `approval` nodes that already exist in the workflow engine — no new
primitives are needed.

## Why deterministic?

A reviewer node that returns free text can:

- Emit an approval string that looks right but contains a hidden caveat.
- Omit required fields entirely, causing a downstream `when:` to branch on an
  empty string.
- Return valid JSON but with `blocking_issues` that a string-comparison miss.

A deterministic gate eliminates these failure modes by:

1. Requiring the reviewer to return strict JSON via `output_format`.
2. Running a short bash script that validates the JSON and emits one of four
   tokens: `GATE_APPROVED`, `GATE_REJECTED`, `GATE_NEEDS_HUMAN`, or
   `GATE_INVALID`.
3. Branching downstream nodes only on those tokens — never on raw model output.

## Gate shape

Every hard gate follows this shape:

```
author node
  → reviewer node  (output_format with verdict enum + blocking_issues)
  → validation node  (bash — emits a gate token)
  → continue node  (when: GATE_APPROVED)
  → halt node      (when: != GATE_APPROVED — approval or cancel)
```

## Reviewer node schema

The reviewer node must declare an `output_format` with:

- A required `verdict` enum (`APPROVED` | `REJECTED` | `NEEDS_HUMAN`).
- A required `blocking_issues` array. An empty array means no blockers.
- Any additional required fields your downstream validation needs.

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

## Validation node

The validation node is a `bash:` node. It reads the reviewer's captured output,
parses it, and emits exactly one gate token to stdout.

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
    issues  = data.get("blocking_issues")

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

### Validation rules

The validation node must treat every ambiguous case as failure:

| Condition | Token |
|-|-|
| Reviewer output is empty | `GATE_INVALID` |
| Reviewer output is not valid JSON | `GATE_INVALID` |
| `blocking_issues` is missing or not an array | `GATE_INVALID` |
| `verdict == APPROVED` and `blocking_issues` is empty | `GATE_APPROVED` |
| `verdict == APPROVED` and `blocking_issues` is non-empty | `GATE_REJECTED` |
| `verdict == NEEDS_HUMAN` | `GATE_NEEDS_HUMAN` |
| Any other verdict | `GATE_REJECTED` |

## Downstream branching

Branch exclusively on the validation token — never on the reviewer's raw output.

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
    message: "Spec gate did not approve. Review the validation result before continuing."
    capture_response: true
```

## Loop policy

A model-emitted `until` completion signal is not sufficient for a hard gate.
Hard loops must use `until_bash` or route through a validation node before
advancing the workflow.

If `until` and `until_bash` disagree, the downstream validation node is
authoritative.

## Approval policy

Human `approval` nodes are for explicit halt paths only:

- Reviewer returned `NEEDS_HUMAN`.
- Maximum retry attempts exhausted.
- Validation returned `GATE_INVALID`.

A human approving a halt does not retroactively make malformed reviewer output
valid — it records an operator override at the gate.

## Complete minimal example

```yaml
name: gated-spec
description: Author a spec, review it, and gate on deterministic validation.
interactive: true
provider: claude

nodes:
  - id: spec-author
    prompt: |
      Write a concise spec for the requested feature and save it to
      .specify/spec.md.

  - id: spec-review
    depends_on: [spec-author]
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
          print("GATE_INVALID"); sys.exit(0)
      try:
          data = json.loads(raw)
      except Exception:
          print("GATE_INVALID"); sys.exit(0)
      verdict = data.get("verdict")
      issues  = data.get("blocking_issues")
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
    prompt: |
      Author the implementation plan from the approved spec.

  - id: spec-halt
    depends_on: [spec-validate]
    when: "$spec-validate.output != 'GATE_APPROVED'"
    approval:
      message: "Spec gate did not approve. Check spec-validate output before continuing."
      capture_response: true
```

## Checklist

Before shipping a workflow with a hard review gate:

- [ ] Reviewer node declares `output_format` with `verdict` enum and `blocking_issues` array.
- [ ] Validation node handles empty reviewer output as `GATE_INVALID`.
- [ ] Validation node handles non-JSON reviewer output as `GATE_INVALID`.
- [ ] Downstream `when:` conditions reference `$<validate-node>.output`, not `$<reviewer-node>.output`.
- [ ] A halt path exists for every non-`GATE_APPROVED` token.
- [ ] Hard loops use `until_bash` rather than relying solely on a model-emitted completion signal.
