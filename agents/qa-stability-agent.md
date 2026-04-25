---
name: qa-stability-agent
description: Use proactively after a Flutter implementation phase completes or at significant mid-plan checkpoints to validate stability before continuing. Invoke when subagent-driven-development finishes all tasks, when the orchestrator decides to pause, or when the user signals implementation is done and needs QA validation. NOT for pre-merge code review — use superpowers:requesting-code-review instead.
model: inherit
---

# QA Stability Agent

Entry point for post-implementation QA on Flutter projects. Detects platform and test stack, then invokes the correct runner.

## Steps

### 0. Autonomous bootstrap (optional, delegates to qa-flutter-bootstrap)

Before any routing, delegate infrastructure lifecycle to the bootstrap skill.

1. Invoke the `qa-flutter-bootstrap` skill via the Skill tool with args: `--up --caller=stability-agent`.
2. Parse the stdout:
   - `QA_BOOTSTRAP_STATUS=UP` or `QA_BOOTSTRAP_STATUS=ALREADY_UP_BY={caller}` → continue to Step 1.
   - Any other status or non-zero exit → abort the agent. Return verdict `FAIL` with reason `"bootstrap failed: <error from stdout>"`. Skip Steps 1–4; still execute Step 5 (teardown is idempotent).

Composition rule: if the bootstrap skill reports `ALREADY_UP_BY=<other>`, this agent does NOT own the lifecycle — Step 5 will detect this and skip teardown accordingly.

### 1. Locate `qa-agent.yaml`

Resolve in this order:
1. `./qa-agent.yaml` in the current working directory.
2. `<project-root>/qa-agent.yaml` (walk up looking for a `pubspec.yaml` sibling).
3. If neither exists, ask the user: `"¿Cuál es la ruta absoluta al archivo qa-agent.yaml?"` and use the provided path.

**Never use a hardcoded absolute path.** The companion qa-agent directory (if needed by the Appium stack) is resolved relative to the yaml location.

### 2. Extract routing fields

From the yaml:
- `project.platform` — `"web"` or `"android"` (default: `"android"` if absent)
- `project.android_stack` — `"appium"` or `"flutter_drive"` (default: `"flutter_drive"` if absent)
- `post_run.include_unit` — optional boolean; if `true`, run unit coverage after the main runner

### 3. Route to the correct skill

| platform | android_stack | Skill to invoke |
|---|---|---|
| `web` | — | `qa-flutter-web-runner` with objective `"stability-check"` |
| `android` | `appium` | `qa-flutter-android-runner` with the implementation summary as objective |
| `android` | `flutter_drive` | `qa-flutter-manual-runner` with feature slug `regresion` |

Invoke via the Skill tool. Capture the returned report path.

### 4. Optional — unit/widget coverage pass

If `post_run.include_unit == true`, after the main runner completes:
- For each feature mentioned in the implementation summary, invoke `qa-flutter-unit-generator` with that feature name and `--auto`.
- Collect all unit reports.

### 5. Autonomous teardown (mandatory, always runs)

**This step runs even if Step 3 or Step 4 aborted.** The teardown is idempotent; do not skip on partial failure.

1. Invoke the `qa-flutter-bootstrap` skill via the Skill tool with args: `--down --caller=stability-agent`.
2. Parse the stdout:
   - `QA_BOOTSTRAP_STATUS=DOWN` → record as "Infrastructure teardown: OK".
   - `QA_BOOTSTRAP_STATUS=SKIPPED_NOT_OWNER=<owner>` → record as "Infrastructure teardown: deferred to {owner}".
   - `QA_BOOTSTRAP_STATUS=NO_MARKER` → record as "Infrastructure teardown: no marker (nothing to clean)".
   - `QA_BOOTSTRAP_STATUS=TEARDOWN_FAILED` → log warning, record as "Infrastructure teardown: FAILED — see marker file". Do NOT change the run's verdict.

This step MUST NOT change the overall verdict (PASS/PARCIAL/FAIL) determined by Steps 3–4.

### 6. Return

Return to the caller:
```
Main QA report: <path>
Unit reports (if any): <list of paths>
Infrastructure teardown: <OK | deferred to {owner} | no marker | FAILED>
Overall verdict: PASS | PARCIAL | FAIL
```

## When NOT to use

- When implementation is partial and stability validation would block ongoing work.
- When the project has no `qa-agent.yaml` and the user has not consented to creating one.
- For pre-merge code review — use `superpowers:requesting-code-review` instead.
