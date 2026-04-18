---
name: qa-flutter-web-runner
description: Use when validating stability of a Flutter web app after implementation changes. Runs flutter analyze, flutter test, flutter build web, analyzes git diff, and generates a functional test checklist.
argument-hint: "<objective or 'stability-check'>"
---

# qa-flutter-web-runner

Orchestrates a stability QA check for a Flutter **web** app after implementation changes.
Generates a functional test checklist (manual) as input for a downstream autonomous QA agent.

Invoked as: `/qa-flutter-web-runner "<objective or 'stability-check'>"` 
OR delegated automatically by `qa-stability-agent` when `project.platform` is `"web"`.

## Configuration

Set these values to match your project:

- `CONFIG_PATH`: absolute path to your `qa-agent.yaml`
- `QA_AGENT_DIR`: absolute path to your `qa-agent/` directory

Expected `qa-agent.yaml` fields for web mode:
```yaml
project:
  platform: "web"
  flutter:
    path: "<absolute path to Flutter project>"
  auth:                         # optional — omit if app is unauthenticated
    email: "qa@example.com"
    password: "secret"
  web:
    base_url: "http://localhost:8080"   # where the app runs locally
agent:
  reports_output_dir: "qa-reports"
```

## Execution — follow every step in order

### Step 1: Read configuration

If CONFIG_PATH has not been set, ask the user:
> "What is the absolute path to your qa-agent.yaml?"
Store as CONFIG_PATH. Also ask:
> "What is the absolute path to your qa-agent/ directory?"
Store as QA_AGENT_DIR.

Use the Read tool to read CONFIG_PATH. Extract:
- `project.flutter.path`
- `project.auth` (entire block — may be absent)
- `project.web.base_url`
- `agent.reports_output_dir`

**Path check:** If `project.flutter.path` is empty or not set, ask:
> "What is the absolute path to your Flutter web project?"
Store as `FLUTTER_PATH`. Do NOT modify the yaml.

If `project.web.base_url` is absent, default to `http://localhost:8080`.

### Step 2: Validate project structure

Run these checks in parallel via Bash:

```bash
# Check pubspec.yaml exists
test -f "{FLUTTER_PATH}/pubspec.yaml" && echo "OK" || echo "MISSING"

# Check flutter is available
flutter --version 2>&1 | head -1

# Check web is an enabled platform
cat "{FLUTTER_PATH}/pubspec.yaml" | grep -A5 "flutter:"
```

If pubspec.yaml is missing: print error and STOP.

### Step 3: Static analysis

Use TaskCreate to create a task "flutter analyze" and mark it in_progress.

Run via Bash (cwd = FLUTTER_PATH):
```bash
cd "{FLUTTER_PATH}" && flutter analyze 2>&1
```

- If exit code is non-zero or output contains "error •": record as **ANALYSIS_FAILED**
- Warnings are allowed but must be listed in the report
- Mark task completed.

### Step 4: Unit and widget tests

Use TaskCreate to create a task "flutter test" and mark it in_progress.

Run via Bash (cwd = FLUTTER_PATH):
```bash
cd "{FLUTTER_PATH}" && flutter test 2>&1
```

Extract:
- Total tests, passed, failed
- Names of any failing tests

Record as **TESTS_PASSED** or **TESTS_FAILED (N failures)**. Mark task completed.

### Step 5: Web build validation

Use TaskCreate to create a task "flutter build web" and mark it in_progress.

Run via Bash (cwd = FLUTTER_PATH):
```bash
cd "{FLUTTER_PATH}" && flutter build web 2>&1 | tail -20
```

- If exit code is non-zero: record as **BUILD_FAILED** — include last 20 lines of output
- If successful: record as **BUILD_OK**
- Mark task completed.

If **BUILD_FAILED**: print error and STOP. The app cannot be tested further.

### Step 6: Identify impacted functionalities

Use TaskCreate to create a task "Analyze git diff" and mark it in_progress.

Run via Bash (cwd = FLUTTER_PATH):
```bash
cd "{FLUTTER_PATH}" && git diff HEAD~1 --name-only 2>/dev/null || git diff --name-only 2>/dev/null
```

Map changed files to functional areas using these rules:
- `lib/pages/**` → **Pages / UI flows** (note the specific page names)
- `lib/state/**` or `lib/notifiers/**` → **State / business logic**
- `lib/services/**` → **Network / API integration**
- `lib/domain/**` or `lib/models/**` → **Data models / DTOs**
- `lib/config/**` → **App configuration / routing**
- `lib/widgets/**` or `lib/components/**` → **Shared UI components**
- `test/**` → **Test coverage** (note added/removed tests)
- `pubspec.yaml` → **Dependencies** (check for version changes)
- `web/**` → **Web-specific assets / index.html**

Produce a deduplicated list of impacted areas. Mark task completed.

### Step 7: Generate functional test checklist

Current datetime formatted as `YYYY-MM-DDTHH-MM` = `{timestamp}`.

Write `{QA_AGENT_DIR}/{reports_output_dir}/{timestamp}-web-checklist.md`:

```markdown
# QA Checklist — Flutter Web — {timestamp}

## Build Status
| Check | Result |
|-------|--------|
| flutter analyze | {ANALYSIS_RESULT} |
| flutter test | {TESTS_RESULT} |
| flutter build web | {BUILD_RESULT} |

## Functionalities Impacted by Recent Changes

{For each impacted area, one bullet: "**Area:** files changed"}

## Functional Test Cases

### App startup
- [ ] La app carga en Chrome sin errores de consola (F12 → Console)
- [ ] La página inicial se muestra correctamente
- [ ] No hay errores de red en la pestaña Network
{IF auth block present:}
- [ ] Con credenciales válidas ({auth.email}) se accede a la página principal
- [ ] Con credenciales inválidas aparece mensaje de error apropiado
{END IF}

{For each impacted area, generate a section:}
### {Area name}
{3–5 test cases relevant to that area, as checkboxes:}
- [ ] {Happy path: what should work correctly}
- [ ] {Edge case or validation: what should be rejected/handled}
- [ ] {UI feedback: loading states, error messages, success confirmation}

### Regresión general
- [ ] Navegar por todas las secciones sin que la app crashee
- [ ] Recargar la página (F5) mantiene el estado esperado
- [ ] Funciona en modo responsive (mobile viewport)

## Análisis (salida bruta)
<details>
<summary>Archivos modificados</summary>

{raw list of changed files from git diff}

</details>

## Notas para el agente QA autónomo
Este checklist fue generado automáticamente post-implementación.
Impacto detectado en: {comma-separated list of impacted areas}.
Prioridad de validación: build status → startup → áreas impactadas → regresión.
```

Mark task completed.

### Step 8: Final output

Print:
```
QA Web checklist generated.
Reports directory: {QA_AGENT_DIR}/{reports_output_dir}/
Checklist: {QA_AGENT_DIR}/{reports_output_dir}/{timestamp}-web-checklist.md

Build: {BUILD_RESULT} | Tests: {TESTS_RESULT} | Analysis: {ANALYSIS_RESULT}
Impacted areas: {count} — {comma-separated list}
```
