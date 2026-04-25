---
description: Run integration tests for a Flutter Android feature via flutter_drive — generates the test if missing, regresses if it exists.
argument-hint: "<feature-name> [--auto]"
---

Invoke the `qa-flutter:qa-flutter-manual-runner` skill with arguments: $ARGUMENTS

Follow the skill's SKILL.md exactly — start with pre-flight checks (Step 3), no shortcuts.

If `$ARGUMENTS` is empty or `regresion`, run the full regression suite. Otherwise treat the argument as a feature name (look it up in `integration_test/manual/`; generate if missing).
