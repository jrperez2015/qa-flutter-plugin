---
name: qa-flutter-unit-generator
description: Use when generating or executing unit, widget, or state-management tests for a Flutter feature — covers the base of the testing pyramid. Detects BLoC/Cubit/Provider/Riverpod/GetX stacks automatically. Runs via flutter test (no device, no backend). NOT for integration/E2E tests — use qa-flutter-manual-runner or qa-flutter-android-runner. NOT for Flutter web stability checks — use qa-flutter-web-runner.
argument-hint: "<feature-name> [--layer=repository|state|widget|all] [--auto]"
---

# qa-flutter-unit-generator

Generates and executes unit, widget, and state-management tests for a Flutter feature via `flutter test`. Complements `qa-flutter-manual-runner` (integration/E2E) — together they cover the full pyramid.

**Invocation examples:**
- `/qa-unit login` — all layers for the `login` feature
- `/qa-unit login --layer=repository` — only data layer
- `/qa-unit login --layer=state` — only BLoC/Cubit/Provider/Riverpod/GetX
- `/qa-unit login --layer=widget` — only widget tests
- `/qa-unit login --auto` — unattended, auto-resolves candidates
- `/qa-unit login --golden` — adds a golden test to the widget suite

## Scope

| Layer | Here | Elsewhere |
|---|---|---|
| Repository / DataSource / Service | ✅ | — |
| BLoC / Cubit / Provider / Riverpod / GetX | ✅ | — |
| Widget (single screen, mocked state) | ✅ | — |
| Golden (visual regression) | ✅ opt-in `--golden` | — |
| Integration / E2E (flutter drive) | ❌ | `qa-flutter-manual-runner` |
| Platform channels / native | ❌ | (out of scope) |

## When NOT to use

- The feature has no code yet (use `my-custom-brainstorming` first, then implement).
- You need to validate UI against a running backend (use `qa-flutter-manual-runner`).
- You need stability check on Flutter web (use `qa-flutter-web-runner`).
- The app uses no state management and no repository layer — nothing testable here. Move to widget tests directly.

## Template references

Layer-specific templates live in `references/`. Load only the one matching the layer being generated:

| Layer | Reference |
|---|---|
| Repository / Service / DataSource | [references/repository-template.md](references/repository-template.md) |
| BLoC / Riverpod / Provider / GetX | [references/state-templates.md](references/state-templates.md) |
| Widget | [references/widget-template.md](references/widget-template.md) |

---

## Step 1 — Parse arguments

- `FEATURE` = first positional token, trimmed and lowercased.
- `LAYER` = value of `--layer=...`, default `all`. Valid: `repository`, `state`, `widget`, `all`. Invalid → abort with `⛔ --layer inválido`.
- `AUTO_MODE` = true if `--auto` present.
- `GOLDEN` = true if `--golden` present.

In `AUTO_MODE`, print: `Ejecutando en modo automático — sin interacción durante el run.`

## Step 2 — Read configuration

Read `qa-agent.yaml` from project root (cwd → walk up to `pubspec.yaml` sibling → ask user). Extract with defaults:

| Variable | yaml path | Default |
|---|---|---|
| `REPORTS_DIR` | `reports.output_dir` | `test/docs/QA_REPORTS` |
| `UNIT_TEST_ROOT` | `unit.test_root` | `test` |
| `COVERAGE_TARGET` | `unit.coverage_target` | `80` |
| `UNIT_TIMEOUT` | `timeouts.unit_test_seconds` | `60` |

If yaml absent, use defaults and warn — do NOT abort (unit tests don't need the runner config).

## Step 3 — Pre-flight: detect stack

### 3.1 — Read `pubspec.yaml`

Extract `APP_PACKAGE_NAME` (from `name:`), `APP_VERSION` (from `version:`), and the full `dev_dependencies` + `dependencies` blocks.

### 3.2 — Detect `STATE_STACK`

Scan dependencies for (in order of precedence):

| Package | STATE_STACK |
|---|---|
| `flutter_bloc` or `bloc` | `bloc` |
| `riverpod` / `flutter_riverpod` / `hooks_riverpod` | `riverpod` |
| `provider` | `provider` |
| `get` | `getx` |
| `mobx` | `mobx` |
| none of the above | `setstate` |

If multiple, pick the one with more import sites under `lib/` (Grep `^import .*<package>`). Ties → prompt user or alphabetical first in `--auto`.

### 3.3 — Detect `MOCK_LIB`

| dev_dependencies contain | MOCK_LIB |
|---|---|
| `mocktail` | `mocktail` |
| `mockito` | `mockito` |
| neither | `none` |

If `none`:
- Interactive: ask `¿Instalar mocktail (recomendado) o mockito?`. Edit `pubspec.yaml`, run `flutter pub get`.
- Auto: add `mocktail: ^1.0.0`, run `flutter pub get`, set `MOCK_LIB=mocktail`, flag `DEPS_ADDED=true`.

### 3.4 — Detect `bloc_test` when needed

If `STATE_STACK == bloc` and `bloc_test` missing → add `bloc_test: ^9.1.5`, `flutter pub get`, set `DEPS_ADDED=true`.

### 3.5 — Detect GetIt

Grep `lib/` for `GetIt.` or `get_it`. If found → `USES_GETIT=true`. Generated tests must include `GetIt.I.reset()` in `tearDown`.

### 3.6 — Run `flutter pub get` if needed

If `DEPS_ADDED=true`:
```bash
flutter pub get
```
Abort on non-zero exit.

## Step 4 — Route by feature

For each requested layer:
1. If `test/**/<feature>_*_test.dart` exists for that layer → **[Section F — Execute existing]**
2. Else → **[Section A — Semantic resolution]** → per-layer generation

---

## Section A — Semantic resolution

### A.1 — Repository candidates

Glob `lib/**/*_repository*.dart`, `lib/**/*_service*.dart`, `lib/**/*_datasource*.dart`, `lib/**/*_dao*.dart`, `lib/src/data/**/*.dart`, `lib/src/domain/**/*.dart`. Score 0–100:

| Criterion | Points |
|---|---|
| Class name contains FEATURE keywords | +40 |
| Imports `http`, `dio`, `sqflite`, `shared_preferences` | +20 |
| Under `data/` or `domain/` directory | +15 |
| Has async / Future methods | +10 |
| Referenced by provider/bloc matching FEATURE | +25 |

### A.2 — State-management candidates

By `STATE_STACK`:
- `bloc` → `*_bloc.dart`, `*_cubit.dart`, `*_event.dart`, `*_state.dart`
- `riverpod` → `*_provider.dart`, `*_notifier.dart`, files with `@riverpod`
- `provider` → `*_provider.dart`, `*_notifier.dart` (ChangeNotifier extenders)
- `getx` → `*_controller.dart` extending `GetxController`
- `setstate` → skip

Score: class matches FEATURE +50, depends on A.1 repo +30, under `presentation/` or `bloc/` +20.

### A.3 — Widget candidates

Glob `lib/**/*_page.dart`, `lib/**/*_screen.dart`, `lib/**/*_view.dart`, `lib/src/pages/**/*.dart`. Score: name matches FEATURE +50, registered as route +25, imports state from A.2 +25.

### A.4 — Select

Build `CANDIDATES = { repository: [...], state: [...], widget: [...] }`.

**Interactive** — for each layer with 2+ candidates, present a numbered list and ask which to cover (or `skip`).

**Auto** — top-scored per layer; if top < 30 → mark SKIPPED and log.

If no candidates across all layers → abort:
```
⛔ No se encontró implementación para '{FEATURE}' en las capas solicitadas.
```

---

## Section B — Generate Repository test

Load `references/repository-template.md`. Follow its analysis + template sections. Output to `{UNIT_TEST_ROOT}/<relative_path>_test.dart`.

## Section C — Generate State test

Load `references/state-templates.md`. Branch by `STATE_STACK`. Output to `test/<relative_path>/<name>_<stack>_test.dart`.

## Section D — Generate Widget test

Load `references/widget-template.md`. Follow analysis + template. Apply wrapper by `STATE_STACK`. Output to `test/<relative_path>/<widget_name>_test.dart`.

---

## Section E — Execute + coverage

For each generated `TEST_FILE`, sequentially (timeout `UNIT_TIMEOUT`):

```bash
flutter test --coverage --reporter=expanded {TEST_FILE}
```

Capture stdout+stderr as `TEST_OUTPUT`, record `EXIT_CODE`, `DURATION`.

**Determine RESULT:**

| Condition | RESULT |
|---|---|
| EXIT_CODE == 0, all passed | PASS |
| EXIT_CODE == 0 but `[E]` failures | PARCIAL |
| Compile error in output | COMPILE_ERROR |
| Killed by timeout | TIMEOUT |
| Other non-zero exit | ERROR |

**Coverage:** parse `coverage/lcov.info`. For this file's source, compute `LINE_COVERAGE = LH/LF * 100`. If block missing → `LINE_COVERAGE = 0` and flag `COVERAGE_PARSE_FAIL=true`.

---

## Section F — Persist or discard

| RESULT | Action | MODE_LABEL |
|---|---|---|
| PASS | keep | `unit → persistido` |
| PARCIAL | keep, flag review | `unit → persistido con fallos` |
| COMPILE_ERROR / TIMEOUT / ERROR | `rm TEST_FILE` + `.mocks.dart` | `unit → descartado` |

Existing tests routed from Step 4 → **never delete**, keep and report.

---

## Section G — Report

`TIMESTAMP` = current datetime as `YYYY-MM-DDTHH-MM`.

Write `{REPORTS_DIR}/{TIMESTAMP}-{FEATURE}-unit.md`:

```markdown
# Unit/Widget QA Report — Feature: {FEATURE}
**Fecha:** {TIMESTAMP} | **App:** {APP_PACKAGE_NAME} v{APP_VERSION}
**Stack detectado:** state={STATE_STACK}, mock={MOCK_LIB}{if USES_GETIT:}, DI=GetIt{/if}
**Capas ejecutadas:** {list}
{if DEPS_ADDED:} **Nota:** Se añadieron dependencias al pubspec.yaml — revisar antes de commitear.

## Resultados por capa
| Capa | Archivo | Resultado | Duración | Cobertura línea |
|------|---------|-----------|----------|-----------------|
| Repository | {path} | ✅ PASS | {D}s | {LC}% |
| State ({STATE_STACK}) | {path} | ⚠ PARCIAL | {D}s | {LC}% |
| Widget | {path} | ❌ COMPILE_ERROR | — | — |

## Cobertura agregada
- **Objetivo:** {COVERAGE_TARGET}%
- **Obtenida (promedio PASS):** {AGG}%
- **Estado:** {✅ cumple | ❌ bajo objetivo}

## Pendientes para corrección
{one actionable item per FAIL/PARCIAL/low-coverage, or "Sin pendientes."}

## Próximos pasos sugeridos
- Si falta integración: `/qa-run {FEATURE}`
- Si hay Keys faltantes en widgets, añadirlos antes de re-generar
{if STATE_STACK == setstate:} - Considerar migrar a un state manager testeable
```

Print: `Reporte: {REPORTS_DIR}/{TIMESTAMP}-{FEATURE}-unit.md`.

**Coverage loop opcional:** If `AGG < COVERAGE_TARGET` and not `AUTO_MODE`, ask if generate additional tests for uncovered branches (parse `lcov.info` for `DA:X,0` lines).

---

## Principles (enforced by templates)

All templates in `references/` bake these in. Do not deviate:

1. **Given-When-Then** naming in every test.
2. **Layer isolation** — mock dependencies, never the SUT. Never mock Riverpod providers — override their deps.
3. **`GetIt.I.reset()` in tearDown** when `USES_GETIT`.
4. **Dispose in tearDown** — `sut.close()` (bloc), `container.dispose()` (riverpod), `Get.reset()` (getx), `sut.dispose()` (notifier).
5. **Widget tests**: `physicalSize` + `devicePixelRatio` + `addTearDown(reset)`.
6. **Finders**: prefer `find.byKey` over `find.text` when keys exist.
7. **Never** `await Future.delayed(...)` — use `pump()`, `pumpAndSettle()`, or `Completer`.
8. **Reset** `debugDefaultTargetPlatformOverride = null` at test end if mutated.
9. **Match `any()` (mocktail) vs `any` (mockito)** to `MOCK_LIB`.

## Common mistakes

| Mistake | Fix |
|---|---|
| Mocking a Riverpod provider directly | Use `ProviderContainer(overrides: [...])` on its deps |
| No `physicalSize` in widget tests | Add it + `addTearDown(tester.view.resetPhysicalSize)` |
| `Future.delayed` as a wait | Use `Completer` or `pumpAndSettle` |
| Missing `GetIt.I.reset()` | Tests pollute each other — always reset |
| Finding widgets by text string | Add Keys to source + `find.byKey` |
| Mixing `any` / `any()` syntax | Match to `MOCK_LIB` — they're not interchangeable |
| Tests deleted on FAIL even when fix is trivial | Use `--no-auto-discard` (TODO: add flag) or keep manually |

## Differences vs `qa-flutter-manual-runner`

| Aspect | manual-runner | unit-generator |
|---|---|---|
| Test type | `flutter drive` / `integration_test` | `flutter test` |
| Device required | ✅ | ❌ |
| Backend required | ✅ | ❌ (mocked) |
| Coverage in report | ❌ | ✅ |
| Output dir | `integration_test/manual/` | `test/` (mirrors `lib/`) |
| Persistence | PASS only | PASS or PARCIAL |
| MUTANT tagging | ✅ | ❌ (tests never hit backend) |
| Runtime | minutes/feature | seconds/file |

Run both for full-pyramid coverage:
```
/qa-unit <feature>    # base + middle
/qa-run  <feature>    # top
```
