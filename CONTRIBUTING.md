# Contributing to qa-flutter

## How to add a new platform runner

1. Create `skills/qa-flutter-<platform>-runner/SKILL.md` with this frontmatter:
   ```yaml
   ---
   name: qa-flutter-<platform>-runner
   description: Use when running QA tests against a Flutter <Platform> app. [describe trigger conditions]
   argument-hint: "<test objective>"
   ---
   ```
2. Follow the step structure from `qa-flutter-android-runner` as a reference.
3. Register the new runner in `agents/qa-stability-agent.md` — add a condition to the platform routing steps.
4. Update `README.md` skills table.

## Conventions

- **No emojis** in skill bodies or agent steps (project convention)
- **English** for technical identifiers and documentation
- **No hardcoded absolute paths** — skills must ask the user for paths at runtime
- Frontmatter `description` must start with "Use when..." and describe trigger conditions only (not the workflow)

## PR flow

1. Fork the repository
2. Create a branch: `feat/your-runner-name` or `fix/issue-description`
3. Make your changes
4. Verify no hardcoded paths: `grep -rEn "[A-Z]:[/\\\\]" skills/ agents/`
5. Open a PR describing which skill or agent you changed and why

## Reporting bugs

Use the [bug report template](.github/ISSUE_TEMPLATE/bug_report.md). Include:
- Platform (Android/Web)
- Your `qa-agent.yaml` with sensitive values redacted
- The exact error or unexpected behavior
