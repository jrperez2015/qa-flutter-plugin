---
name: qa-flutter-android-runner
description: Use when running QA tests against a Flutter Android app using Appium. Orchestrates bootstrap, feature test dispatch via qa-flutter-android-tester, and teardown.
argument-hint: "<test objective>"
---

# qa-flutter-android-runner

Orchestrates a full QA test run against a Flutter Android app.

Invoked as: `/qa-flutter-android-runner "<test objective>"`

Example: `/qa-flutter-android-runner "test el flujo de registro y login de usuario"`

## Configuration

Before running, set these two values to match your project:

- `CONFIG_PATH`: absolute path to your `qa-agent.yaml` file
- `QA_AGENT_DIR`: absolute path to your `qa-agent/` directory (the one containing `scripts/`)

The skill will prompt for these values at runtime if not configured.

## Execution — follow every step in order

### Step 1: Read configuration

If CONFIG_PATH has not been set, ask the user:
> "What is the absolute path to your qa-agent.yaml?"
Store as CONFIG_PATH. Also ask:
> "What is the absolute path to your qa-agent/ directory (containing scripts/)?"
Store as QA_AGENT_DIR.

Use the Read tool to read CONFIG_PATH. Extract:
- `agent.max_features_per_run`
- `agent.agent_max_execution_seconds`
- `agent.reports_output_dir`
- `project.backend.path`
- `project.flutter.path`

**Path check:** If `project.backend.path` is empty or not set, ask the user:
> "What is the absolute path to your backend project?"
Store as `BACKEND_PATH`.

If `project.flutter.path` is empty or not set, ask the user:
> "What is the absolute path to your Flutter project?"
Store as `FLUTTER_PATH`.

These values are used in memory for this run only — the yaml file is NOT modified.

### Step 2: Bootstrap — start services

Use TaskCreate to create a task "Bootstrap services" and mark it in_progress.

Run via Bash:
```
cd {QA_AGENT_DIR} && python scripts/bootstrap.py qa-agent.yaml
```

Read `{QA_AGENT_DIR}/{reports_output_dir}/bootstrap-status.json`.

If ANY of `api_ready`, `flutter_ready`, `appium_ready` is false:
- Print the errors list
- Print: "Bootstrap failed. Aborting test run. Check services and retry."
- Mark bootstrap task completed.
- STOP. Do not dispatch sub-agents.

Mark the bootstrap task completed.

### Step 3: Parse test objective into feature list

Given the argument passed to `/qa-flutter-android-runner`, identify the individual features to test.

Rules:
- Extract concrete feature names as lowercase slugs (e.g. "signup", "login", "password-recovery")
- If more features are identified than `max_features_per_run`, take the first N and note the rest as "not executed" in the summary
- If the objective is ambiguous, test the most specific features you can infer

Print the feature list to the user before dispatching sub-agents.

### Step 4: Dispatch sub-agents — one per feature, strictly sequential

For EACH feature in the list (do NOT dispatch more than one at a time):

1. Use TaskCreate to create a task "Test feature: {feature_name}" and mark it in_progress.

2. Record the start time.

3. Use the Agent tool to dispatch the qa-flutter-android-tester skill with this exact prompt:
```
CONFIG_PATH={CONFIG_PATH}
FEATURE_NAME={feature_name}
QA_AGENT_DIR={QA_AGENT_DIR}
BACKEND_PATH={BACKEND_PATH}
FLUTTER_PATH={FLUTTER_PATH}

Follow the qa-flutter-android-tester skill instructions exactly.
```

4. If the sub-agent does not complete within `agent_max_execution_seconds` seconds:
   - Use TaskUpdate to mark the task as completed with a TIMEOUT note.
   - Create a timeout placeholder report at `{QA_AGENT_DIR}/{reports_output_dir}/{timestamp}-{feature_name}.md`:
     ```
     # QA Report — Feature: {feature_name}
     **Resultado:** ❌ TIMEOUT — agent exceeded {agent_max_execution_seconds}s
     ```
   - Mark the task completed with label "TIMEOUT".

5. On success: the sub-agent returns the path to the report file. Read that file and store its content.

6. Mark the task completed.

7. WAIT for this sub-agent to fully complete before starting the next one.

### Step 5: Generate summary report

Current datetime formatted as YYYY-MM-DDTHH-MM = `{timestamp}`.

Write `{QA_AGENT_DIR}/{reports_output_dir}/{timestamp}-summary.md`:

```markdown
# QA Summary — {timestamp}
**Features solicitadas:** {total} | **Ejecutadas:** {executed} | **No ejecutadas:** {skipped}

| Feature | Resultado | Duración | Errores principales |
|---------|-----------|----------|---------------------|
{one row per executed feature, values from each feature report}

## Features no ejecutadas (límite max_features_per_run)
{list, or "Ninguna." if all features were tested}

## Errores para planificación
{numbered list: one actionable item per FAIL or TIMEOUT feature}
{if all passed: "Sin errores. Todos los tests pasaron."}
```

### Step 6: Teardown — stop services

Run via Bash:
```
cd {QA_AGENT_DIR} && python scripts/teardown.py {reports_output_dir}
```

### Step 7: Final output

Print:
```
Test run complete.
Reports directory: {QA_AGENT_DIR}/{reports_output_dir}/
Summary: {QA_AGENT_DIR}/{reports_output_dir}/{timestamp}-summary.md
```
