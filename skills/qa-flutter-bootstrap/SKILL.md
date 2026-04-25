---
name: qa-flutter-bootstrap
description: Use to bring up (start backend + verify device) and tear down (stop what we started) the infrastructure needed by an autonomous QA run on a Flutter project. Invoked by qa-stability-agent and qa-flutter-release-gate around the main run. Also invokable standalone for recovery after a failed run. NOT for running tests — use qa-flutter-manual-runner or qa-flutter-android-runner. NOT for configuring AVDs or installing SDKs — the skill assumes infra is provisioned.
argument-hint: "[--up | --down] [--caller=<name>]"
---

# qa-flutter-bootstrap

Owner of infrastructure lifecycle for autonomous QA runs. Brings up backend + device before tests; tears down only what it started after them. Enforces single-run invariant via a marker file that also serves as lock (G6).

**Invocation examples:**
- `qa-flutter-bootstrap --up --caller=release-gate` — before the gate's main work
- `qa-flutter-bootstrap --down --caller=release-gate` — after, always
- `qa-flutter-bootstrap --down` — standalone recovery after a crashed run

## Scope

This skill:
- ✅ Reads `autonomous.*` block from `qa-agent.yaml`.
- ✅ Starts backend via `start_cmd` and waits on `health_url`.
- ✅ Verifies / boots device via `autonomous.device.boot_avd`.
- ✅ Writes a marker file that functions as both teardown state and run lock.
- ✅ Supports composition: "outermost caller owns lifecycle" — nested invocations no-op.
- ✅ Delegates to user-owned `scripts/bootstrap.py` for the Appium stack.

This skill does NOT:
- Run tests — use `qa-flutter-manual-runner` or `qa-flutter-android-runner`.
- Configure the Android SDK, create AVDs, or install system images — out of scope.
- Manage multiple backends / service arrays / docker-compose natively — single `start_cmd` only.
- Send notifications — a CI step concern.

## When to use

- At the start of `qa-stability-agent` (Step 0) and `qa-flutter-release-gate` (Step -1) in autonomous mode.
- At the end of those same skills (Step 5 / Step 9 respectively), always.
- Standalone for recovery: `qa-flutter-bootstrap --down` will use an existing marker to tear down orphaned infra.

## When NOT to use

- For running or generating tests — wrong skill.
- When `qa-agent.yaml` has no `autonomous.*` block — the skill is a no-op but still valid; there is simply nothing to bring up.
- For configuring the SDK or creating AVDs — the skill assumes infra is provisioned.

## Configuration contract

From `qa-agent.yaml`:

```yaml
autonomous:
  backend:                                     # optional block
    start_cmd: "./gradlew bootRun --args='--spring.profiles.active=test'"
    start_cwd: "../transfer_rest_api"          # relative to the yaml directory
    health_url: "http://localhost:8080/api/health"
    ready_timeout_seconds: 120                 # default if absent
    stop_cmd: "pkill -f 'gradlew bootRun'"     # optional
    stop_timeout_seconds: 30
  device:                                      # optional
    boot_avd: "Pixel_5_API_33"                 # exact AVD name
    boot_timeout_seconds: 90
```

**Resolve `autonomous.backend.start_cwd` relative to the directory containing `qa-agent.yaml`**, not the shell's cwd.

## Marker file

Path: `{reports.output_dir}/.qa-bootstrap-marker` (default `qa-reports/.qa-bootstrap-marker`).

Schema:

```yaml
started_at: 2026-04-23T03:00:00Z
caller: release-gate
pid_ward: 34821
backend:
  we_started_it: true
  pid: 34821
  stop_cmd: "pkill -f 'gradlew bootRun'"
  stop_timeout_seconds: 30
  health_url: "http://localhost:8080/api/health"
device:
  id: emulator-5554
  we_started_it: false
  emulator_pid: null
teardown_failed: false
```

Presence of the marker = a run is in progress (G6 lock). Absence = no lifecycle ownership by this plugin.

---

## Step 1 — Parse arguments

- `MODE` = `up` if `--up` present, `down` if `--down` present. Error if both or neither.
- `CALLER` = value of `--caller=...`; default = `"standalone"`.

Emit to stdout:
```
QA_BOOTSTRAP_MODE={MODE}
QA_BOOTSTRAP_CALLER={CALLER}
```

## Step 2 — Resolve `qa-agent.yaml`

Follow the same resolution as the rest of the plugin:
1. `./qa-agent.yaml` in cwd.
2. Walk up from cwd until finding a `pubspec.yaml` sibling, then check for `qa-agent.yaml` there.
3. If interactive, ask the user. If `--auto` context implied (CALLER != standalone) and not found → abort.

Let `YAML_DIR` = directory containing the resolved yaml.

## Step 3 — Branch by MODE

- If `MODE == up` → follow Section UP below.
- If `MODE == down` → follow Section DOWN below.

---

## Section UP — bring up infrastructure

### U.1 — Lock / composition check

Let `MARKER_PATH` = `{YAML_DIR}/{reports.output_dir}/.qa-bootstrap-marker` (default `qa-reports/.qa-bootstrap-marker`).

If marker exists:
1. Read `caller` and `pid_ward` from it.
2. Verify `pid_ward` alive:
   - Linux/macOS: `kill -0 {pid_ward}` (exit 0 = alive).
   - Windows (if detected by shell/platform): `tasklist /FI "PID eq {pid_ward}"` parsed for the PID.
3. If pid alive AND `caller != CALLER` → composition rule: emit `QA_BOOTSTRAP_STATUS=ALREADY_UP_BY={caller}` + exit 0. Do nothing else.
4. If pid alive AND `caller == CALLER` → abort:
   ```
   ⛔ Otro run del mismo caller ya en curso desde {started_at} (pid {pid_ward}).
      Si es stale, borrá {MARKER_PATH} y reintentá.
   ```
5. If pid dead → log warning "marker stale from {caller} at {started_at}, cleaning up", delete marker, continue.
6. If marker has `teardown_failed: true` → abort:
   ```
   ⛔ Teardown anterior falló. Revisá procesos residuales y borrá {MARKER_PATH} manualmente.
   ```

If marker does not exist → continue.

### U.2 — Stack detection

Read `project.android_stack` from yaml (default `"flutter_drive"` if absent or if `project.platform` != `"android"`).

- If `android_stack == "appium"` → delegate to Section UP-APPIUM below.
- Else → continue inline.

### U.3 — Backend phase (if `autonomous.backend` present)

If `autonomous.backend` absent → skip to U.4.

1. **Pre-health check.** Run via Bash with 5s timeout:
   ```bash
   curl -s -o /dev/null -w "%{http_code}" --max-time 5 {health_url}
   ```
   If output is `200` → set `backend.we_started_it = false`, skip to U.4.

2. **Start backend in background.** Resolve `start_cwd` relative to `YAML_DIR`:
   ```bash
   cd {YAML_DIR}/{start_cwd} && {start_cmd} > /tmp/qa-bootstrap-backend.log 2>&1 &
   echo $!  # capture pid
   ```
   Record the pid as `BACKEND_PID`.

3. **Poll `health_url`.** Every 2s, up to `ready_timeout_seconds` (default 120):
   ```bash
   curl -s -o /dev/null -w "%{http_code}" --max-time 5 {health_url}
   ```
   If `200` → set `backend.we_started_it = true, backend.pid = BACKEND_PID`, continue.

4. **On timeout.** Best-effort cleanup:
   - Run `stop_cmd` if present.
   - `kill {BACKEND_PID}` if process still alive.
   Then abort:
   ```
   ⛔ Backend no arrancó en {ready_timeout_seconds}s. health_url: {health_url}
      Última respuesta: {last_http_code}. Ver /tmp/qa-bootstrap-backend.log
   ```

### U.4 — Device phase (Android only, if `autonomous.device.boot_avd` present)

If `project.platform != "android"` → skip (web doesn't need device).
If `autonomous.device.boot_avd` absent → note `device.id = null, device.we_started_it = false` in the marker, skip to U.5. The runner skill (manual-runner Section 3.3) will apply its own device logic with its own back-compat fallback.

Otherwise:

1. **Check ADB for existing device:**
   ```bash
   adb devices | grep -E "^(emulator-[0-9]+)\s+device$"
   ```
   Pick the first matching row. If ANY device already running → set `device.id = {found}, device.we_started_it = false`, skip to U.5.

2. **Verify AVD exists:**
   ```bash
   emulator -list-avds | grep -x "{boot_avd}"
   ```
   If empty → abort:
   ```
   ⛔ AVD '{boot_avd}' no existe. AVDs disponibles:
   $(emulator -list-avds)
      Creá el AVD con Android Studio o remové autonomous.device.boot_avd del yaml.
   ```

3. **Launch AVD in background:**
   ```bash
   emulator -avd {boot_avd} -no-snapshot-load > /tmp/qa-bootstrap-emulator.log 2>&1 &
   echo $!
   ```
   Record the pid as `EMULATOR_PID`.

4. **Poll boot completion.** Every 5s up to `boot_timeout_seconds` (default 90):
   ```bash
   adb wait-for-device
   adb shell getprop sys.boot_completed | tr -d '\r'
   ```
   When output is `1` → device booted. Run `adb devices` to capture the device id (e.g. `emulator-5554`). Set `device.id = {id}, device.we_started_it = true, device.emulator_pid = EMULATOR_PID`.

5. **On boot timeout.** Best-effort:
   ```bash
   adb -s {candidate_id} emu kill 2>/dev/null || true
   kill {EMULATOR_PID} 2>/dev/null || true
   ```
   Abort:
   ```
   ⛔ AVD '{boot_avd}' no booteó en {boot_timeout_seconds}s.
      Ver /tmp/qa-bootstrap-emulator.log
   ```

### U.5 — Write marker atomically

Compute `pid_ward`:
- If `backend.we_started_it == true` → `pid_ward = backend.pid`.
- Else if `device.we_started_it == true` → `pid_ward = device.emulator_pid`.
- Else → `pid_ward = $$` (the shell pid of this bootstrap invocation — lives until the caller returns).

Write:
```bash
cat > {MARKER_PATH}.tmp <<EOF
started_at: $(date -u +%Y-%m-%dT%H:%M:%SZ)
caller: {CALLER}
pid_ward: {pid_ward}
backend:
  we_started_it: {backend.we_started_it}
  pid: {backend.pid or 'null'}
  stop_cmd: '{backend.stop_cmd or ''}'
  stop_timeout_seconds: {backend.stop_timeout_seconds or 30}
  health_url: '{backend.health_url or ''}'
device:
  id: {device.id or 'null'}
  we_started_it: {device.we_started_it}
  emulator_pid: {device.emulator_pid or 'null'}
teardown_failed: false
EOF
mv {MARKER_PATH}.tmp {MARKER_PATH}
```

### U.6 — Emit concise stdout

```
QA_BOOTSTRAP_STATUS=UP
QA_BOOTSTRAP_MARKER={MARKER_PATH}
```

Exit 0.

### Section UP-APPIUM — delegate to companion `scripts/bootstrap.py`

The Appium stack owns its own bootstrap implementation in the user's companion `scripts/bootstrap.py`. This skill only:

1. Performs U.1 (lock check) — shared for both stacks.
2. Invokes `scripts/bootstrap.py` with `--marker={MARKER_PATH}` arg. Example:
   ```bash
   cd {QA_AGENT_DIR} && python scripts/bootstrap.py qa-agent.yaml --marker={MARKER_PATH}
   ```
   Where `QA_AGENT_DIR` is the directory that contains `qa-agent.yaml` (which must also contain `scripts/bootstrap.py` per Appium stack requirements — see `docs/references/appium-bootstrap-contract.md`).
3. If `bootstrap.py` exits non-zero → abort with its stderr in the error message. Do NOT write a marker (bootstrap.py should have written one before exiting with success).
4. On success, verify `{MARKER_PATH}` exists with valid schema. If not → abort with contract violation notice (the user's `bootstrap.py` is outdated; point them to `docs/references/appium-bootstrap-contract.md`).
5. Emit the same stdout as U.6.

**Fallback for pre-contract `bootstrap.py`:** if invoking with `--marker` returns the specific exit code `2` (reserved for "flag not recognized") OR if after success no marker was written, synthesize a minimal marker from the legacy `bootstrap-status.json` if present, with `we_started_it: true` for both backend and device (best-effort, conservative). Log deprecation warning:
```
⚠ scripts/bootstrap.py no soporta el contrato v1 (no escribió marker).
  Se sintetizó un marker best-effort. Actualizá según docs/references/appium-bootstrap-contract.md.
  Este fallback se removerá en el próximo release minor.
```

---

## Section DOWN — tear down infrastructure

### D.1 — Read marker

If `{MARKER_PATH}` does not exist:
```
QA_BOOTSTRAP_STATUS=NO_MARKER
```
Exit 0 (idempotent).

Parse marker. Let `MARKER_CALLER` = marker's `caller` field.

### D.2 — Composition check

If `MARKER_CALLER != CALLER`:
```
QA_BOOTSTRAP_STATUS=SKIPPED_NOT_OWNER
QA_BOOTSTRAP_OWNER={MARKER_CALLER}
```
Exit 0 (outermost caller owns lifecycle; we don't touch what we didn't bring up).

### D.3 — Stack detection

Read `project.android_stack` from yaml. If `appium` → delegate to Section DOWN-APPIUM below. Else continue inline.

### D.4 — Device teardown (inline)

If `device.we_started_it == true` AND `device.id != null`:
```bash
adb -s {device.id} emu kill
```
Wait up to 10s. If still running → `kill {device.emulator_pid} 2>/dev/null || true`. Record `device_stopped = true` if successful, else `device_stopped = false`.

### D.5 — Backend teardown (inline)

If `backend.we_started_it == true`:

1. If `backend.stop_cmd` present → run it with `backend.stop_timeout_seconds` timeout:
   ```bash
   timeout {stop_timeout_seconds} sh -c "{stop_cmd}"
   ```

2. If `backend.pid` alive after stop_cmd OR no stop_cmd:
   ```bash
   kill {backend.pid} 2>/dev/null
   ```
   Wait 5s. If still alive → `kill -9 {backend.pid} 2>/dev/null`.

3. Best-effort health re-check:
   ```bash
   curl -s -o /dev/null -w "%{http_code}" --max-time 2 {backend.health_url}
   ```
   If still `200` → log warning "backend still responding after stop". Record `backend_stopped = false`. Else `backend_stopped = true`.

### D.6 — Marker cleanup

- If `device_stopped == true` AND `backend_stopped == true` (or they were no-ops because `we_started_it == false`) → `rm {MARKER_PATH}`.
- Else → rewrite marker with `teardown_failed: true`. Next `--up` will refuse to continue until marker is manually cleared.

### D.7 — Emit concise stdout

```
QA_BOOTSTRAP_STATUS=DOWN              # or TEARDOWN_FAILED
```

Exit 0 (even on `TEARDOWN_FAILED` — the caller uses the status line, not exit code).

### Section DOWN-APPIUM — delegate to `scripts/teardown.py`

1. Invoke:
   ```bash
   cd {QA_AGENT_DIR} && python scripts/teardown.py qa-agent.yaml --marker={MARKER_PATH}
   ```
2. On exit 0 → emit the stdout from D.7 based on what teardown.py reports (it should delete the marker or mark `teardown_failed`).
3. On non-zero exit → keep marker with `teardown_failed: true`; emit `QA_BOOTSTRAP_STATUS=TEARDOWN_FAILED`.

Same fallback for pre-contract `teardown.py` as UP-APPIUM: if `--marker` flag is rejected (exit 2) or marker not updated after, best-effort clean the marker file ourselves and log deprecation warning.

---

## Common mistakes

| Mistake | Fix |
|---|---|
| `autonomous.backend.start_cwd` relative to shell cwd instead of yaml dir | Skill resolves relative to yaml directory. Document in README. |
| Using `pkill -f java` as `stop_cmd` | Too broad — kills unrelated java. Use specific pattern (artifact name, port, etc.) or let skill fallback to `kill {backend.pid}`. |
| `health_url` that returns 200 before DB migrations are done | Use a discriminating endpoint (e.g. one that verifies DB connectivity). |
| Running `--up` twice without `--down` | Marker lock aborts the second. |
| Removing the marker manually mid-run | `--down` won't know what to stop → leaks. Only remove the marker after a hard crash, and only if you have killed all backend/emulator processes yourself. |
| Expecting `--up` to create an AVD | Out of scope — the skill assumes the AVD in `boot_avd` already exists. |
| `scripts/bootstrap.py` not writing the marker | Old contract — update per `docs/references/appium-bootstrap-contract.md`. Fallback synthesis is temporary. |

## Security note

`autonomous.backend.start_cmd` executes shell arbitrary from the yaml. Trust model = same as any `./scripts/*.sh` in the repo (dev-authored, versioned). Don't embed secrets in `start_cmd` — pass them via `.env` or CI secret store, which the shell inherits via environment.

## Limitations

- Lock is **local to the repo** (marker file). Not global. If the same backend is shared across machines, you are responsible for cross-machine coordination — out of scope.
- Only a single backend is supported per run (see spec decision 5 — single `start_cmd`). Multi-service lifecycles must be encapsulated by the user in `start_cmd` (e.g. a shell script that launches all).
- `create_if_missing` for AVDs is out of scope. Provisioning the AVD is the user's responsibility.
- SIGKILL of the caller leaves a stale marker. Next `--up` detects staleness via `pid_ward` liveness and cleans automatically.
