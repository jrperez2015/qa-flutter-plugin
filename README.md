# qa-flutter — Claude Code Plugin

QA stability plugin for Flutter projects (Android + Web).

For **Android** projects, it runs Appium UI tests against your app and generates per-feature Markdown reports. For **Flutter web** projects, it runs `flutter analyze`, `flutter test`, and `flutter build web`, analyzes the git diff, and produces a manual test checklist for the changed areas.

One orchestrator agent auto-detects your platform from `qa-agent.yaml` and routes to the correct runner skill automatically.

[![Plugin Validation](https://github.com/jrperez2015/qa-flutter-plugin/actions/workflows/validate-plugin.yml/badge.svg)](https://github.com/jrperez2015/qa-flutter-plugin/actions/workflows/validate-plugin.yml)

---

## Requirements

- [Claude Code CLI](https://claude.ai/code)
- A `qa-agent.yaml` file configured for your project (see [Configuration](#configuration))
- **Android only:** Appium server running + Android emulator connected via ADB
- **Web only:** Flutter SDK with web support enabled

---

## Installation

### Manual (v1)

Estos pasos instalan el plugin `qa-flutter` (que contiene los skills `qa-flutter-android-runner`, `qa-flutter-android-tester`, `qa-flutter-web-runner`) en Claude Code como plugin de usuario.

> **Por qué estos pasos:** Claude Code no permite instalar plugins directamente desde una ruta de archivo. Requiere un "marketplace" registrado. Los pasos 2-4 crean y registran un marketplace local que apunta a la carpeta donde vive el plugin.

**Paso 1 — Copiar los archivos del plugin**

Copiar el contenido de `qa-flutter-plugin/` al directorio de plugins locales de Claude Code:

```
C:\Users\<usuario>\.claude\plugins\local\qa-flutter\
├── .claude-plugin\
│   └── plugin.json
├── agents\
│   └── qa-stability-agent.md
├── skills\
│   ├── qa-flutter-android-runner\
│   │   └── SKILL.md
│   ├── qa-flutter-android-tester\
│   │   └── SKILL.md
│   └── qa-flutter-web-runner\
│       └── SKILL.md
└── README.md
```

Si la carpeta `~/.claude/plugins/local/` no existe, crearla manualmente.

**Paso 2 — Crear el manifiesto del marketplace local**

Crear el archivo `C:\Users\<usuario>\.claude\plugins\local\.claude-plugin\marketplace.json` con este contenido:

```json
{
  "$schema": "https://anthropic.com/claude-code/marketplace.schema.json",
  "name": "local",
  "description": "Local plugins for this user's Claude Code installation",
  "owner": {
    "name": "<Tu nombre>",
    "email": "<tu-email>"
  },
  "plugins": [
    {
      "name": "qa-flutter",
      "description": "QA stability agents and runners for Flutter Android and Flutter web projects",
      "author": {
        "name": "<Tu nombre>",
        "email": "<tu-email>"
      },
      "source": "./qa-flutter",
      "category": "development"
    }
  ]
}
```

> **Nota:** El campo `"source": "./qa-flutter"` es una ruta relativa al directorio del marketplace (es decir, `~/.claude/plugins/local/qa-flutter/`).

**Paso 3 — Registrar el marketplace en Claude Code**

```bash
claude plugin marketplace add "C:/Users/<usuario>/.claude/plugins/local"
```

Resultado esperado:
```
✔ Successfully added marketplace: local (declared in user settings)
```

**Paso 4 — Instalar el plugin desde el marketplace local**

```bash
claude plugin install qa-flutter@local
```

Resultado esperado:
```
✔ Successfully installed plugin: qa-flutter@local (scope: user)
```

**Paso 5 — Verificar la instalación**

```bash
claude plugin list
```

Resultado esperado:
```
❯ qa-flutter@local
    Version: 1.0.0
    Scope: user
    Status: ✔ enabled
```

Si el status muestra `✘ failed to load`, revisar que el `marketplace.json` del paso 2 existe y que la carpeta `qa-flutter/` tiene la estructura correcta.

**Paso 6 — Reiniciar Claude Code**

Cerrar y volver a abrir Claude Code. Los skills aparecerán en el system prompt como:

```
- qa-flutter:qa-flutter-android-runner
- qa-flutter:qa-flutter-android-tester
- qa-flutter:qa-flutter-web-runner
```

---

### Errores frecuentes de instalación

**`Plugin qa-flutter not found in marketplace local`**
→ El plugin fue añadido directamente a `installed_plugins.json` sin registrar el marketplace. Ejecutar `claude plugin uninstall qa-flutter` y repetir desde el paso 3.

**`Marketplace file not found at .../.claude-plugin/marketplace.json`**
→ El archivo del paso 2 no existe o está en la ruta incorrecta. Verificar que está en `~/.claude/plugins/local/.claude-plugin/marketplace.json`.

**`failed to load` después de instalar correctamente**
→ Claude Code no ha recargado la lista de plugins. Reiniciar Claude Code (paso 6).

---



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
  email_domain: "qa.local"  # domain used for generated test emails
  use_uuid: true             # prepend random UUID to email addresses (avoids conflicts on re-run)
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

If you use [superpowers](https://github.com/obra/superpowers) — a Claude Code plugin that orchestrates multi-task implementation plans — the `qa-flutter:qa-stability-agent` agent triggers automatically when an implementation phase completes. It reads your `qa-agent.yaml`, detects the platform, and runs the correct QA runner. No manual invocation needed.

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
