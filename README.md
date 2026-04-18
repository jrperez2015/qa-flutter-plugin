# qa-flutter — Claude Code Plugin

QA stability plugin for Flutter projects (Android + Web).

One agent auto-detects your platform from `qa-agent.yaml` and routes to the correct runner skill — Appium for Android or a static checklist generator for Flutter web.

[![Plugin Validation](https://github.com/YOUR_USERNAME/qa-flutter-plugin/actions/workflows/validate-plugin.yml/badge.svg)](https://github.com/YOUR_USERNAME/qa-flutter-plugin/actions/workflows/validate-plugin.yml)

---

## Requirements

- [Claude Code CLI](https://claude.ai/code)
- A `qa-agent.yaml` file configured for your project (see [Configuration](#configuration))
- **Android only:** Appium server running + Android emulator connected via ADB
- **Web only:** Flutter SDK with web support enabled

---

## Installation

### Manual (v1)

```bash
# 1. Clone to your Claude Code plugins directory
git clone https://github.com/YOUR_USERNAME/qa-flutter-plugin \
  "$HOME/.claude/plugins/local/qa-flutter"
```

Then add this entry to `~/.claude/plugins/installed_plugins.json` inside the `"plugins"` object:

```json
"qa-flutter@local": [
  {
    "scope": "user",
    "installPath": "/absolute/path/to/.claude/plugins/local/qa-flutter",
    "version": "1.0.0",
    "installedAt": "2026-01-01T00:00:00.000Z",
    "lastUpdated": "2026-01-01T00:00:00.000Z"
  }
]
```

Replace `installPath` with the actual path on your machine.

### Via Plugin Manager (v2 — coming soon)

```
/plugin install qa-flutter
```

---

## Configuration

Create a `qa-agent.yaml` file in your QA workspace directory:

```yaml
project:
  platform: "web"          # "web" or "android"
  flutter:
    path: "/path/to/your/flutter/project"
  backend:                 # Android only
    path: "/path/to/your/backend"
    start_command: "mvn spring-boot:run -Pdev"
    health_check_url: "http://localhost:PORT/health"
    ready_timeout_seconds: 60
  auth:                    # Web only, optional (omit if unauthenticated)
    email: "qa@example.com"
    password: "secret"
  web:
    base_url: "http://localhost:8080"

appium:                    # Android only
  server_url: "http://localhost:4723"
  device_name: "emulator-5554"
  app_package: "com.example.yourapp"
  app_activity: ".MainActivity"
  step_retry_attempts: 3
  step_retry_wait_seconds: 2

agent:
  max_features_per_run: 5
  agent_max_execution_seconds: 120
  feature_timeout_seconds: 90
  reports_output_dir: "qa-reports"

test_data:
  email_domain: "qa.local"
  use_uuid: true
```

---

## Usage

### Manual invocation

```
# Android — test specific features
/qa-flutter-android-runner "test login and signup flow"

# Web — stability check after implementation changes
/qa-flutter-web-runner stability-check
```

### Auto-invocation via orchestrator

If you use [superpowers](https://github.com/obra/superpowers) and `subagent-driven-development`, the `qa-flutter:qa-stability-agent` agent triggers automatically when an implementation phase completes. No manual invocation needed.

---

## What each skill does

| Skill | Platform | Output |
|-------|----------|--------|
| `qa-flutter-android-runner` | Android | Runs Appium tests, generates per-feature Markdown reports + summary |
| `qa-flutter-android-tester` | Android | Tests a single feature (sub-agent, do not invoke directly) |
| `qa-flutter-web-runner` | Web | Runs `flutter analyze`, `flutter test`, `flutter build web`, git diff analysis — generates a manual test checklist |

---

## Extensions

### Short-term (current architecture supports these)

- **New platform runners** — create `skills/qa-flutter-<platform>-runner/SKILL.md` following the existing runner pattern, then register it in `agents/qa-stability-agent.md`
- **Custom report formats** — modify the report template in `qa-flutter-android-tester` or `qa-flutter-web-runner`
- **Webhook notifications** — add a Step 8 to any runner that POSTs the summary to a Slack webhook or email service
- **Per-project config override** — extend the Configuration section to support a `.qa-flutter.yaml` project-local override file

### Long-term (requires architectural changes)

- **iOS runner** — XCUITest-based runner analogous to the Android runner; requires a new `qa-flutter-ios-tester` skill and iOS-capable bootstrap script
- **CI/CD integration** — GitHub Actions step that invokes the web runner headlessly and attaches the checklist as a PR artifact
- **Report history dashboard** — a web UI that reads the `qa-reports/` directory and visualizes trends across runs
- **Multi-project orchestration** — a coordinator agent that runs stability checks across multiple projects in sequence

---

## Roadmap

See [ROADMAP.md](ROADMAP.md) for the versioned plan toward marketplace distribution and extended platform support.

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

---

## License

MIT — see [LICENSE](LICENSE).
