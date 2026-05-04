# Changelog

All notable changes to this project will be documented in this file.

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).
This project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.3.1] — 2026-05-04

Centralize all QA configuration files under `qa-plugin-config/` in the target Flutter project. **Breaking change** — existing projects must migrate files (see Migration guide in README).

### Changed

- **All skills and agents** — config resolution updated from project root (`qa-agent.yaml`, `qa-knowledge.yaml`, `.env`, `qa-plans/`, `qa-reports/`) to `qa-plugin-config/` subfolder (`qa-plugin-config/qa-agent.yaml`, etc.). Walk-up logic preserved: cwd → walk up to `pubspec.yaml` sibling → ask user.
- **`qa-flutter-bootstrap`** — introduces `PROJECT_ROOT` / `YAML_DIR` distinction. `start_cwd` resolves relative to `PROJECT_ROOT` (parent of `qa-plugin-config/`) for backward-compatible relative paths. Bootstrap marker path formula: `{PROJECT_ROOT}/{reports.output_dir}/.qa-bootstrap-marker`.
- **`qa-flutter-android-runner` / `qa-flutter-web-runner`** — `QA_AGENT_DIR` now points to `{PROJECT_ROOT}/qa-plugin-config`; Appium companion scripts expected at `qa-plugin-config/scripts/`.
- **`qa-flutter-manual-runner`** — `.env` path: `qa-plugin-config/.env`. `PLAN_DIR` default: `qa-plugin-config/qa-plans/`. Reports default: `qa-plugin-config/qa-reports/`.
- **`qa-flutter-test-planner`** — plan output default: `qa-plugin-config/qa-plans/<feature>.md`. `PLAN_DIR` default: `qa-plugin-config/qa-plans/`.
- **`qa-flutter-unit-generator`** — `REPORTS_DIR` default: `qa-plugin-config/qa-reports`.
- **`qa-flutter-release-gate`** — reads `qa-plugin-config/qa-agent.yaml`.
- **`qa-knowledge-manager`** — knowledge file path: `qa-plugin-config/qa-knowledge.yaml`.
- **`qa-stability-agent`** — all internal references updated to `qa-plugin-config/` paths.
- **`templates/qa-knowledge.yaml`** — install comment updated to `qa-plugin-config/`.
- **`commands/qa-plan.md`** — output path updated.
- **README** — Quick start, configuration, troubleshooting, and migration guide updated for new structure.
- **Plugin manifest** — version bumped to `1.3.1`.

### Migration

Projects upgrading from `≤ 1.30` must run:

```bash
mkdir -p qa-plugin-config
mv qa-agent.yaml    qa-plugin-config/
mv qa-knowledge.yaml qa-plugin-config/    # if present
mv .env             qa-plugin-config/     # if present
mv qa-plans/        qa-plugin-config/     # if present
# Update reports.output_dir in qa-agent.yaml to: qa-plugin-config/qa-reports
```

Then update `.gitignore`:
```
qa-plugin-config/.env
qa-plugin-config/qa-reports/
```

---

## [1.30] — 2026-05-04

QA planning layer. Closes roadmap gap **G8** (planning artifact). Backward-compatible — runners keep their existing on-the-fly behavior when no plan is provided.

### Added

- **`qa-flutter-test-planner` skill** — produces an auditable, human-editable QA test plan at `qa-plans/<feature>.md` covering: scope, pantallas a cubrir, precondiciones / datos requeridos, flujos end-to-end (happy + error), criterios de aceptación, riesgos / fuera de scope. Markdown body with YAML frontmatter that downstream runners parse. Invokable as `/qa-plan <feature>` (interactive) or `/qa-plan <feature> --auto` (CI / scheduled refresh).
- **`/qa-plan` slash command** routing to the planner skill.
- **`planning:` block in `qa-agent.yaml`** — `enabled`, `test_plan_dir` (default `qa-plans/`), `require_plan`, `score_threshold`, `flow_depth`, `aliases`, `precondition_severity_overrides`. All optional.
- **References under the planner skill** — `plan-template.md` (canonical artifact format), `screens-discovery.md` (multi-layer semantic resolution playbook generalized from manual-runner § C.1), `preconditions-checklist.md` (canonical list of preconditions with detection sources and severity rules).

### Changed

- **`qa-flutter-manual-runner`** — `--plan=<path>` flag added. New Section C.0 reads the plan when provided and skips the on-the-fly resolution in C.1–C.2; C.3.2 uses plan flow steps as test step seeds.
- **`qa-flutter-android-runner`** — Step 3 uses plan's Section 2 (pantallas) as feature list and Section 4 (flujos) as test scenarios passed to the `qa-flutter-android-tester` sub-agent. Falls back to free-form parsing without `--plan`.
- **`qa-flutter-web-runner`** — Step 6 uses plan's Section 2 instead of git-diff analysis when `--plan` is provided. Step 7 checklist seeded from plan flows + preconditions.
- **`qa-stability-agent`** — new Step 2.5 resolves plans for each feature in the implementation summary. Auto-injects `--plan=<path>` to the routed runner. Honors `planning.require_plan: true` by aborting with NO-GO when plans are missing.
- **`commands/qa-run.md`** — accepts `--plan=<path>` and documents it.
- **README** — new "Planificación de QA" section documenting when to plan, configuration, plan vs. report distinction. Quick start updated to suggest `/qa-plan` first.
- **Plugin manifest** — version bumped to `1.30`.

### Roadmap

- **G8 — Planning artifact** ✅ closed in this release.

## [1.2.0] — 2026-05-02

Self-learning cycle (autoaprendizaje) and skill performance improvements.

### Added

- **`qa-knowledge-manager` skill** — autoaprendizaje cycle that wraps every QA run with two transparent steps: `preflight` (applies known environment fixes before the run) and `learn` (captures new errors after the run and updates `qa-knowledge.yaml` with user confirmation). Also exposes `register` (manual fix entry) and `report` (knowledge base audit) modes.
- **`templates/qa-knowledge.yaml`** — starter template for the knowledge file. Copy to project root and commit with the project; if absent, the orchestrator behaves exactly as v1.1.x.

### Changed

- **`qa-stability-agent` → v2.1** — adds Steps 0a (knowledge preflight) and 0b (knowledge learn) as non-fatal wrappers around the existing run. New flags: `--auto` (CI non-interactive), `--dry-run` (simulate plan without running tests), `--skip-preflight`, `--skip-learn`. Verdict semantics updated: `GO` / `NO-GO` / `CONDITIONAL` with exit codes 0 / 1 / 2 respectively. Fully backwards-compatible: without `qa-knowledge.yaml` the agent is identical to v2.0.
- **`qa-flutter-manual-runner`** — Grep-before-Read optimization in the semantic resolution phase; `pubspec.yaml` is now read once in Step 2 and reused in Sections C and E, eliminating redundant reads per feature.
- **`qa-flutter-unit-generator`** — Grep-before-Read filter applied before full file reads in Section A (repository/bloc/widget candidates), reducing unnecessary context load on large codebases.

## [1.1.1] — 2026-04-24

Usability patch — slash commands and CLI permission docs.

### Added

- **`commands/` directory** with four slash commands wrapping the most-used skills: `/qa-run`, `/qa-unit`, `/qa-release-gate`, `/qa-bootstrap`. Previously the skills could only be invoked via natural language or `claude -p` prompts — now they appear in the slash menu.
- **README — "CLI permissions in unattended mode"** section explaining the `.claude/settings.local.json` allowlist needed when running `claude -p` (non-interactive) so the orchestrator does not hang on the first denied tool call.

### Notes

- No skill or agent behavior changed in this release. Existing skills and tags continue to work; the slash commands are additive convenience.

## [1.1.0] — 2026-04-23

Autonomous orchestrator cohort. Closes roadmap gaps G1, G2, G4, G6 and upgrades report quality.

### Added

- **`qa-flutter-bootstrap` skill** — unified `--up` / `--down` lifecycle for backend + Android device. Writes a marker file at `<reports-dir>/.qa-bootstrap-marker` that doubles as lock (G6) and teardown state. Idempotent; survives crashes via PID-ward health checks.
- **`qa-flutter-unit-generator` skill** — unit / widget / state test generator and runner (fast layer of the pyramid, no device required).
- **`qa-flutter-manual-runner` skill** — `/qa-run` command for `flutter_drive` + `integration_test` based E2E runs. New default Android stack.
- **`qa-flutter-release-gate` skill** — `/qa-release-gate` GO/NO-GO production gate wrapping `qa-stability-agent`; emits JSON artifact alongside Markdown report (G4, schema v1.0); applies 5-rule root-cause correlation over findings.
- **`autonomous.*` block in `qa-agent.yaml`** — declarative backend/device bootstrap consumed by `qa-flutter-bootstrap`.
- **Diagnóstico sections in per-feature reports** — error principal, stack trace top 8 frames, filtered log tail.
- **Appium bootstrap contract v1** — unified marker-based handoff for companion `scripts/bootstrap.py` / `teardown.py`. See [`docs/references/appium-bootstrap-contract.md`](docs/references/appium-bootstrap-contract.md). Legacy fallback retained one minor for migration.

### Changed

- **`qa-stability-agent`** — Step 0 (invoke `qa-flutter-bootstrap --up`) + Step 5 (mandatory teardown). Runner delegation unchanged.
- **`qa-flutter-android-runner`** — migrates bootstrap invocation to `qa-flutter-bootstrap --up --caller=android-runner`; deprecates direct `scripts/bootstrap.py` invocation.
- **Default Android stack is now `flutter_drive`** — Appium remains supported as opt-in via `project.android_stack: "appium"`. Decision rationale in [`docs/android-stacks.md`](docs/android-stacks.md).

### Gaps deferred

- G3 — Notifications on NO-GO (out of plugin scope; handle via CI notification step)
- G5 — Waiver approval workflow (compliance-driven, future)
- G7 — Cost tracking per run (future)

## [1.0.0] — 2026-04-18

### Added

- `qa-flutter:qa-stability-agent` — orchestrator agent that auto-detects Flutter platform and routes to the correct runner
- `qa-flutter-android-runner` — Appium-based QA runner for Flutter Android apps
- `qa-flutter-android-tester` — single-feature Appium tester (sub-agent of android-runner)
- `qa-flutter-web-runner` — stability checker for Flutter web apps; runs `flutter analyze`, `flutter test`, `flutter build web`, git diff analysis, and generates a functional test checklist
- Generic configuration: all hardcoded paths removed; skills prompt users for paths at runtime
