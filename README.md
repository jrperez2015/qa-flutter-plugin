# qa-flutter

[![Plugin Validation](https://github.com/jrperez2015/qa-flutter-plugin/actions/workflows/validate-plugin.yml/badge.svg)](https://github.com/jrperez2015/qa-flutter-plugin/actions/workflows/validate-plugin.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Version](https://img.shields.io/badge/version-1.1.1-blue.svg)](CHANGELOG.md)

QA plugin for Flutter projects — covers the full testing pyramid (unit/widget/state → integration/E2E → stability) across Android and Web, with an **autonomous orchestrator** that brings up the backend, boots the device, runs tests, emits an actionable report + JSON artifact, and tears everything down. Designed as a pre-release stability gate.

## Installation

The plugin is distributed as a Claude Code plugin via a local marketplace. Until public marketplaces land, the install path is:

**1. Clone the repo into your Claude Code plugins directory:**

```bash
# Windows
git clone https://github.com/jrperez2015/qa-flutter-plugin C:/Users/<user>/.claude/plugins/local/qa-flutter

# macOS / Linux
git clone https://github.com/jrperez2015/qa-flutter-plugin ~/.claude/plugins/local/qa-flutter
```

**2. Create the local marketplace manifest** at `~/.claude/plugins/local/.claude-plugin/marketplace.json`:

```json
{
  "$schema": "https://anthropic.com/claude-code/marketplace.schema.json",
  "name": "local",
  "description": "Local plugins",
  "owner": { "name": "<Your name>", "email": "<your-email>" },
  "plugins": [
    {
      "name": "qa-flutter",
      "description": "QA orchestration for Flutter with autonomous lifecycle",
      "author": { "name": "<Your name>", "email": "<your-email>" },
      "source": "./qa-flutter",
      "category": "development"
    }
  ]
}
```

**3. Register and install:**

```bash
claude plugin marketplace add "<path-to-local-marketplace-dir>"
claude plugin install qa-flutter@local
claude plugin list   # expect: qa-flutter@local  enabled
```

Restart Claude Code. Skills (`qa-flutter:qa-flutter-bootstrap`, `qa-flutter:qa-flutter-manual-runner`, `qa-flutter:qa-flutter-release-gate`, etc.) appear in the system prompt.

**Updating:** `git pull` in the clone directory, then restart Claude Code. No re-install needed.

**Prerequisites:** see the [Prerequisites](#prerequisites) section below.

---

## Quick start

1. Install the plugin (copy `qa-flutter/` into `~/.claude/plugins/local/` or load via marketplace).
2. Install prerequisites for your stack (see [Prerequisites](#prerequisites)).
3. Create `qa-agent.yaml` at your project root (see [Configuration](#configuration--qa-agentyaml)).
4. Run:
   ```
   /qa-unit <feature>    # unit + widget + state tests (fast, no device)
   /qa-run  <feature>    # integration tests via flutter_drive
   ```
5. At end of branch, let the orchestrator auto-invoke `qa-stability-agent` for the final QA pass.

## Prerequisites

| Requirement | Needed by |
|---|---|
| Flutter SDK `^3.10.0` | All skills |
| `adb` in PATH | Android skills |
| Python 3 | Appium stack only |
| Appium server + UiAutomator2 driver | Appium stack only |
| Android emulator or physical device | Android E2E skills |
| Chrome (for manual web QA) | Web runner |

## Components

### Agent

| Component | Kind | How it runs |
|---|---|---|
| `qa-flutter:qa-stability-agent` | Sub-agent | Auto-delegated by Claude after implementation completes, or invoke explicitly with `@qa-stability-agent` |

### Skills (slash commands)

| Skill | Slash command | Layer | Mechanism | Platform |
|---|---|---|---|---|
| `qa-flutter-unit-generator` | `/qa-unit <feature>` | Unit / Widget / State | `flutter test` | Any |
| `qa-flutter-manual-runner` | `/qa-run <feature>` | Integration / E2E | `flutter drive` + `integration_test` | Android |
| `qa-flutter-android-runner` | `/qa-flutter-android-runner "<objective>"` | Integration / E2E | Appium + Python orchestrator | Android |
| `qa-flutter-android-tester` | (sub-agent, not user-invoked) | Integration / E2E | Appium — single feature | Android |
| `qa-flutter-web-runner` | `/qa-flutter-web-runner "<objective>"` | Stability + checklist | `flutter analyze/test/build web` + git diff | Web |
| `qa-flutter-release-gate` | `/qa-release-gate [--threshold] [--version]` | Go/No-Go release gate | Wraps `qa-stability-agent` + severity classification | Any |

---

## Using `qa-stability-agent`

### What it does

Entry point for **post-implementation QA**. Reads `qa-agent.yaml`, detects the platform and Android stack, then delegates to the correct runner skill. Optionally runs unit coverage after the main E2E pass.

### When to use

- After all tasks in an implementation plan are marked complete.
- At a significant mid-plan checkpoint before continuing with remaining tasks.
- Before merging a development branch (stability gate).

### When NOT to use

- **Mid-implementation** — partial code makes QA noisy.
- **For code review** — use `superpowers:requesting-code-review` instead.
- **Without `qa-agent.yaml`** — the agent needs it to route; it will ask the user for a path, but skipping the config is not recommended.

### How to invoke

**Auto-delegation** (default): Claude routes to this agent when the conversation signals implementation completion. Phrases that trigger it:

- "Listo, todas las tareas completadas"
- "Implementation done, ready for QA"
- "All tasks done"
- After `my-subagent-driven-development` finishes its plan

**Explicit invocation** from the main session:

```
@qa-stability-agent
```

Or ask Claude directly: `"Invoca qa-stability-agent para validar la rama"`.

### Generic example — end-to-end

**Scenario:** You just finished implementing a feature `password-recovery` using `my-subagent-driven-development`. All tasks are marked complete. You want to validate stability before opening a PR.

**1. Ensure `qa-agent.yaml` exists:**
```yaml
project:
  platform: "android"
  android_stack: "flutter_drive"
device:
  id: emulator-5554
  app_package: com.example.app
backend:
  test_url: "http://localhost:8080/api"
reports:
  output_dir: test/docs/QA_REPORTS
post_run:
  include_unit: true        # also run unit coverage
```

**2. Trigger the agent:**
```
user: "All tasks done, validar estabilidad"
```

**3. What happens internally:**
1. Agent reads `qa-agent.yaml`.
2. `platform=android`, `android_stack=flutter_drive` → routes to `qa-flutter-manual-runner` with objective `regresion`.
3. Manual runner executes all tests in `integration_test/manual/*.dart` sequentially.
4. `post_run.include_unit == true` → for each feature mentioned in the implementation summary (here, `password-recovery`), invokes `qa-flutter-unit-generator --auto`.
5. Aggregates reports.

**4. What the agent returns:**
```
Main QA report: test/docs/QA_REPORTS/2026-04-23T14-30-summary.md
Unit reports:
  - test/docs/QA_REPORTS/2026-04-23T14-42-password-recovery-unit.md
Overall verdict: PASS
```

### Verdict semantics

| Verdict | Meaning |
|---|---|
| `PASS` | All executed features passed; unit coverage meets target if run |
| `PARCIAL` | Some tests failed or coverage below target — review the linked reports |
| `FAIL` | Main runner aborted (bootstrap failure, device lost, build broken) |

### Failure modes

| Failure | Agent behavior |
|---|---|
| `qa-agent.yaml` not found | Asks for path; aborts if user doesn't provide |
| Delegated skill aborts in pre-flight (backend down, device missing) | Returns the pre-flight error message verbatim; no retry |
| Main runner passes, `post_run.include_unit` step fails | Returns main PASS + unit FAIL — does not override main verdict |
| Ambiguous `project.platform` | Defaults to `android`; if `android_stack` also absent, defaults to `flutter_drive` |

### Limitations

- **Does not run in parallel** — features are tested sequentially.
- **Does not retry failed runs** — a flaky test that fails once is recorded as fail.
- **Does not verify backend data cleanup** — tests that create data (MUTANT-tagged) assume a test-profile backend handles isolation.
- **Does not open PRs or merge** — pure QA gate; integration with git is manual.
- **Does not cover golden tests** — pass `--golden` when invoking `qa-flutter-unit-generator` directly if you need visual regression.

---

## Stack default and trade-offs

**Default:** `flutter_drive` via `qa-flutter-manual-runner`.

Why the default: fewer moving parts (no Python, no Appium server), faster feedback (30–90s per run vs 2–5min), Dart test code you can commit and refactor, access to the widget tree for assertions. Good enough for 80% of functional QA on Flutter apps.

**Use Appium when you need any of:**
- Native gestures beyond what the widget layer exposes (multi-touch with specific velocity, long-press at coordinates, system UI interactions).
- Multi-app flows (permission dialogs, intents to other apps, deep links opened from Chrome).
- Testing a pre-compiled APK (staging or release build) as a black box.
- Portability to iOS at a later stage (Appium's protocol is cross-platform; `flutter_drive` tests require adaptation).

Configure in `qa-agent.yaml`:

```yaml
project:
  platform: "android"
  android_stack: "flutter_drive"   # or "appium". Default: flutter_drive.
```

Full decision matrix: [docs/android-stacks.md](docs/android-stacks.md).

---

## Typical flows

### After a feature implementation (manual, during dev)
```
/qa-unit <feature>                → base + middle of pyramid
/qa-run  <feature>                → top of pyramid (integration via flutter_drive)
```

### Post-branch stability checkpoint (automated)
Either let Claude auto-delegate to `qa-stability-agent`, or invoke explicitly:
```
@qa-stability-agent
```

### Release gate — GO/NO-GO to production
```
/qa-release-gate --threshold=normal --version=v1.2.3
```
Emits a report classifying findings by severity (Critical/High/Medium/Low) and a binary verdict. See [Release gate](#release-gate) below for details and scheduling.

### CI / unattended
```bash
claude -p "/qa-run regresion --auto" --output-format json
claude -p "/qa-unit <feature> --auto"
claude -p "/qa-release-gate --auto --threshold=strict" --output-format json
```

---

## Configuration — `qa-agent.yaml`

All components resolve this file at runtime by: (1) cwd, (2) walking up to find `pubspec.yaml`, (3) asking the user. **No hardcoded paths.**

### Flutter web
```yaml
project:
  platform: "web"
  flutter:
    path: <absolute path to Flutter project>
  web:
    base_url: "http://localhost:8080"
  auth:                           # optional
    email: qa@example.com
    password: secret
agent:
  reports_output_dir: qa-reports
```

### Android — flutter_drive stack (recommended default)
```yaml
project:
  platform: "android"
  android_stack: "flutter_drive"
device:
  id: emulator-5554
  app_package: com.example.app
timeouts:
  test_seconds: 120
  adb_retry_count: 2
  adb_retry_wait_seconds: 3
execution:
  reset_app_before_each: true
backend:
  test_url: "http://localhost:8080/api"
reports:
  output_dir: test/docs/QA_REPORTS
unit:                             # optional — for unit-generator
  test_root: test
  coverage_target: 80
```

Plus `.env` at the Flutter project root:
```
TEST_EMAIL=qa-user@test.com
TEST_PASSWORD=your-test-password
API_BASE_URL=http://localhost:8080/api
```
`API_BASE_URL` must contain `backend.test_url` (pre-flight rejects mismatches to protect prod data).

### Android — Appium stack
```yaml
project:
  platform: "android"
  android_stack: "appium"
  backend:
    path: <absolute path to backend repo>
  flutter:
    path: <absolute path to Flutter project>
appium:
  device_name: emulator-5554
  app_package: com.example.app
agent:
  max_features_per_run: 5
  agent_max_execution_seconds: 600
  feature_timeout_seconds: 180
  reports_output_dir: qa-reports
test_data:
  email_domain: test.com
  use_uuid: true
```

Requires a companion `qa-agent/` directory next to the yaml with `scripts/bootstrap.py`, `scripts/teardown.py`, `scripts/appium_runner.py`.

### Optional — post-run unit coverage after stability
```yaml
post_run:
  include_unit: true
```
When `true`, `qa-stability-agent` runs `qa-flutter-unit-generator` for each feature after the main E2E pass.

---

## Release gate

`qa-flutter-release-gate` is a **go/no-go production gate**. It wraps `qa-stability-agent`, classifies every finding by production impact, and emits a binary verdict.

### Severity buckets

| Bucket | Examples | Blocks GO |
|---|---|---|
| 🔴 Critical | Login/payment E2E fail, compile error, crash on boot, analyzer `error •` | Always |
| 🟠 High | Secondary E2E fail, timeout, coverage below floor for critical feature | In `strict` and `normal` profiles |
| 🟡 Medium | Unit PARCIAL on non-critical classes, coverage well below target | Only in `strict` if >2 |
| 🔵 Low | Analyzer warnings, minor flakiness, single token retry | Never |

Full rules in [qa-flutter-release-gate/references/severity-rules.md](skills/qa-flutter-release-gate/references/severity-rules.md).

### Threshold profiles

| Profile | GO if |
|---|---|
| `strict` | 0 Critical, 0 High, ≤2 Medium |
| `normal` (default) | 0 Critical, 0 High |
| `lenient` | 0 Critical |

### Gate config in `qa-agent.yaml`

```yaml
release_gate:
  threshold: "normal"                   # strict | normal | lenient
  critical_features:                    # E2E fail on these → Critical
    - "login"
    - "payment"
    - "checkout"
  coverage_floor: 70                    # min coverage for critical features (%)
  waivers:                              # temporary High-downgrades
    - finding_id: "login-e2e-step-3"
      reason: "Flaky network in CI — TICKET-123"
      expires: "2026-05-31"
```

### Scheduling (autonomous routine)

Three supported patterns:

**A) Scheduled via `anthropic-skills:schedule`** — cron within Claude:
```
user: schedule '/qa-release-gate --auto --threshold=normal' every day at 03:00
```

**B) External CI** — GitHub Actions, Task Scheduler, cron:
```bash
claude -p "/qa-release-gate --auto --threshold=strict --version=${VERSION}" \
  --output-format json
```
Read `RELEASE_GATE_EXIT` from output; fail the job if non-zero.

**C) Pre-release manual** — invoked by an engineer before tagging:
```
/qa-release-gate --version=v1.2.3
```

### Gate report output

Written to `{reports_output_dir}/release-gate-{version}-{timestamp}.md`. Includes: verdict, counts per severity, full list of Critical+High findings with links to underlying reports, waivers applied, prioritized action list, exit code.

Example verdict line in the report header:
```
**Fecha:** 2026-04-23T03-00 | **Threshold:** normal | **Veredicto:** 🔴 NO-GO
```

---

## Autonomous execution

Since the release that introduces `qa-flutter-bootstrap`, the release gate runs end-to-end without manual infrastructure setup — bringing up the backend, verifying the device, running tests, emitting the report, and tearing everything down. This is the intended path for nightly CI and pre-release gates.

### Enable it

Add an `autonomous` block to your `qa-agent.yaml`:

```yaml
autonomous:
  backend:
    start_cmd: "./gradlew bootRun --args='--spring.profiles.active=test'"
    start_cwd: "../transfer_rest_api"         # relative to this yaml's directory
    health_url: "http://localhost:8080/api/health"
    ready_timeout_seconds: 120                # optional, default 120
    stop_cmd: "pkill -f 'gradlew bootRun'"    # optional; kill by pid is fallback
    stop_timeout_seconds: 30                  # optional, default 30
  device:
    boot_avd: "Pixel_5_API_33"                # exact AVD name (must exist; skill does not create)
    boot_timeout_seconds: 90                  # optional, default 90
```

Leave out any sub-block that doesn't apply (e.g. web projects omit `device`).

### Invoke

On-demand (manual or CI):
```
claude -p "/qa-release-gate --auto --threshold=normal" --output-format json
```

Scheduled (via `anthropic-skills:schedule`):
```
user: schedule '/qa-release-gate --auto --threshold=strict' every day at 03:00
```

Either way, the gate handles lifecycle. See [docs/references/appium-bootstrap-contract.md](docs/references/appium-bootstrap-contract.md) if you use the Appium stack (you must update your `scripts/bootstrap.py` to the v1 contract).

### CLI permissions in unattended mode

When you run `claude -p "..."` (non-interactive) or use `--permission-mode=bypassPermissions`, Claude Code **denies any tool not in an allowlist** instead of prompting. Without configuration, the orchestrator hangs on the first call to `flutter`, `adb`, `emulator`, or PowerShell.

**Fix:** create `.claude/settings.local.json` at your Flutter project root with the tools the orchestrator needs:

```json
{
  "permissions": {
    "allow": [
      "Bash(git:*)",
      "Bash(curl:*)",
      "Bash(adb:*)",
      "Bash(flutter:*)",
      "Bash(emulator:*)",
      "Bash(rm:*)",
      "Bash(mkdir:*)",
      "Bash(cp:*)",
      "Bash(node:*)",
      "PowerShell(*)",
      "Read(*)",
      "Write(*)",
      "Edit(*)",
      "MultiEdit(*)",
      "Grep(*)",
      "Glob(*)",
      "Monitor(*)",
      "BashOutput(*)",
      "TodoWrite(*)",
      "Task(*)"
    ]
  }
}
```

**Why each entry matters:**

| Pattern | Used by |
|---|---|
| `Bash(git:*)`, `Bash(curl:*)` | git operations, backend health checks |
| `Bash(adb:*)`, `Bash(flutter:*)`, `Bash(emulator:*)` | Android device interaction, test execution |
| `Bash(rm:*)`, `Bash(mkdir:*)`, `Bash(cp:*)`, `Bash(node:*)` | Filesystem operations, Node-based plugin validation |
| `PowerShell(*)` | Windows-side backend lifecycle scripts (start/stop the Spring Boot app) |
| `Read(*)`, `Write(*)`, `Edit(*)`, `MultiEdit(*)`, `Grep(*)`, `Glob(*)` | Skill-driven file inspection and report generation |
| `Monitor(*)`, `BashOutput(*)` | Reading output from long-running background processes (test runs, emulator boot) |
| `TodoWrite(*)` | Skill progress tracking |
| `Task(*)` | Sub-agent dispatch (release-gate spawns the stability agent, etc.) |

Trim entries you do not use (e.g. omit `PowerShell(*)` on macOS/Linux, omit `Bash(emulator:*)` for web-only projects).

This file is per-project and not committed (include `.claude/settings.local.json` in your `.gitignore` if you don't want it shared).

**Quick alternatives:**

| Use case | Command |
|---|---|
| One-off debug, accept all risk | `claude -p "/qa-release-gate --auto" --dangerously-skip-permissions` |
| Production / scheduled | `claude -p "/qa-release-gate --auto"` with `settings.local.json` (recommended) |
| Manual tuning per session | `claude` (interactive) — approve each tool the first time, Claude remembers |

If you have the `update-config` skill installed, you can ask Claude to set the allowlist for you: *"configurame settings.local.json con allowlist para git, flutter, adb, emulator, powershell, curl"*.

### Recovery

If a run crashes hard (SIGKILL or machine power loss), a stale marker may remain at `<reports-dir>/.qa-bootstrap-marker`. Recover with:

```bash
claude -p "invoke the qa-flutter-bootstrap skill with --down"
```

The skill is idempotent — it tears down only what the marker says was started, and handles dead PIDs gracefully.

### Gap status

| Gap | Status |
|---|---|
| G1 — Backend bootstrap (flutter_drive) | ✅ Implemented via `qa-flutter-bootstrap` |
| G2 — Device bootstrap config | ✅ Implemented via `autonomous.device.boot_avd` |
| G4 — JSON artifact alongside markdown | ✅ Implemented (schema v1.0) |
| G6 — Scheduled-run idempotency (lock) | ✅ Implemented via marker file |
| G3 — Notifications on NO-GO | ⚠ Out of plugin scope — handle via CI notification step |
| G5 — Waiver approval workflow | ⏳ Future |
| G7 — Cost tracking per run | ⏳ Future |

---

## Reports

### Location

| Skill | Report path |
|---|---|
| `qa-flutter-manual-runner` | `test/docs/QA_REPORTS/{timestamp}-{feature}.md` |
| `qa-flutter-unit-generator` | `{reports.output_dir}/{timestamp}-{feature}-unit.md` |
| `qa-flutter-android-runner` | `{qa-agent-dir}/{reports_output_dir}/{timestamp}-{feature}.md` |
| `qa-flutter-web-runner` | `{qa-agent-dir}/{reports_output_dir}/{timestamp}-web-checklist.md` |

All suite runs also produce a `{timestamp}-summary.md` aggregating per-feature reports.

### Report skeleton (integration)

```markdown
# QA Report — Feature: login
**Fecha:** 2026-04-23T14-30 | **Duración:** 47s | **Resultado:** ✅ PASS
**App:** my_flutter_app v1.2.3 | **Dispositivo:** emulator-5554
**Modo:** regresion individual

## Objetivo
Verifies the login happy path and one validation-error case.

## Pasos ejecutados
| # | Descripción | Resultado | Duración |
|---|-------------|-----------|----------|
| 1 | Navigate to login | ✅ PASS | 2s |
| 2 | Enter valid credentials | ✅ PASS | 1s |
| 3 | Tap submit | ✅ PASS | 3s |
| 4 | Assert home screen | ✅ PASS | 2s |

## Pendientes para corrección
Sin pendientes.
```

---

## File layout

```
qa-flutter/
  .claude-plugin/plugin.json
  agents/
    qa-stability-agent.md
  skills/
    qa-flutter-unit-generator/
      SKILL.md
      references/                   ← layer templates
    qa-flutter-manual-runner/
    qa-flutter-android-runner/
    qa-flutter-android-tester/
    qa-flutter-web-runner/
  docs/
    android-stacks.md               ← decision matrix
  README.md                         ← this file
```

## Migration guide — Appium users

If you already use the Appium stack (`project.android_stack: "appium"`), the release that introduces `qa-flutter-bootstrap` changes how your companion scripts are invoked.

**What changed:**
- `qa-flutter-android-runner` no longer invokes `python scripts/bootstrap.py` directly. It invokes `qa-flutter-bootstrap --up --caller=android-runner`, which in turn delegates to your `scripts/bootstrap.py` passing a new `--marker=<path>` flag.
- Your `bootstrap.py` and `teardown.py` must accept `--marker` and write/respect a marker file per the [v1 contract](docs/references/appium-bootstrap-contract.md).

**Migration steps:**
1. Read the [contract doc](docs/references/appium-bootstrap-contract.md).
2. Update `scripts/bootstrap.py` to accept `--marker` and write the marker YAML atomically.
3. Update `scripts/teardown.py` to accept `--marker`, read it, clean up, and delete it on success.
4. Update your `qa-agent.yaml` to use the unified `autonomous.*` block (previously you may have had `backend_endpoint`, `appium.boot`, etc. in custom fields).
5. Test end-to-end with: `claude -p "/qa-release-gate --auto"`.

**During the deprecation window (current minor release):** if your scripts ignore `--marker`, the plugin falls back to the legacy `bootstrap-status.json` method and synthesizes a minimal marker. A deprecation warning is printed. At the next minor release, this fallback is removed and Appium runs will fail unless your scripts are updated.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Agent replies "¿Cuál es la ruta absoluta al archivo qa-agent.yaml?" | yaml not found in cwd or parent | Create the yaml at project root, or answer with absolute path |
| "Backend no disponible" pre-flight error | Test backend not running, or `API_BASE_URL` mismatch | Start test backend; verify `.env` matches `backend.test_url` |
| "No hay dispositivos ADB ni AVDs disponibles" | No emulator running and no AVD to launch | Start an emulator manually or create an AVD in Android Studio |
| Tests pass locally but fail in CI | Flaky timing, screen size, or device state | Check report — look for `TOKEN_RETRY` notes or missing `pumpAndSettle` |
| `BUILD_FAILED` in web runner | Compile errors in Flutter web target | Run `flutter build web` manually and fix before re-running QA |
