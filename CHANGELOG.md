# Changelog

All notable changes to this project will be documented in this file.

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).
This project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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
