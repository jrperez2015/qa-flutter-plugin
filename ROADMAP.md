# Roadmap

## Shipped

### v1.1 — Autonomous orchestrator (2026-04-23)

Closes 4 of 7 roadmap gaps and upgrades report quality. See [CHANGELOG.md](CHANGELOG.md) and [`docs/roadmap-integration.md`](docs/roadmap-integration.md).

| Gap | Status |
|---|---|
| G1 — Backend bootstrap (flutter_drive) | ✅ Implemented via `qa-flutter-bootstrap` |
| G2 — Device bootstrap config | ✅ Implemented via `autonomous.device.boot_avd` |
| G4 — JSON artifact alongside markdown | ✅ Implemented (schema v1.0) |
| G6 — Scheduled-run idempotency (marker lock) | ✅ Implemented |

### v1.0 — Initial release (2026-04-18)

Three skills (Android runner, Android tester, Web runner) + one orchestrator agent. Manual installation.

---

## Deferred (future releases)

| Gap | Why deferred | Trigger to prioritize |
|---|---|---|
| G3 — Notifications on NO-GO | Out of plugin scope — CI owns notification channels | User requests in-plugin webhook support |
| G5 — Waiver approval workflow | Compliance-driven; current inline `release_gate.waivers` is enough for small teams | Multi-reviewer audit trail requested |
| G7 — Cost tracking per run | Scale-driven | Token spend becomes a budgeting concern |

---

## v2.0 — Marketplace-ready

**Goal:** `/plugin install qa-flutter` works without manual marketplace registration.

**Required work:**

1. Host this repo as a public marketplace JSON source
2. GitHub Releases triggered by semver tags (`git tag v2.0.0 && git push --tags`)
3. Validate plugin loads correctly via marketplace install path across Windows / macOS / Linux
4. Add `version` bumping to CHANGELOG workflow

---

## v2.x — Extended platforms

- `qa-flutter-ios-runner` — XCUITest via `idb` or `xcrun simctl`
- `qa-flutter-desktop-runner` — validates desktop builds compile and launch

---

## v3.0 — CI/CD native

**Goal:** Run QA checks in CI without a human in the loop.

**Required work:**

1. Package as a GitHub Actions composite action (`action.yml`)
2. Web runner produces machine-readable JSON output (alongside Markdown checklist) — *partially done for release-gate in v1.1*
3. Android runner supports headless emulator via GitHub-hosted runners or self-hosted
4. PR comment bot posts checklist summary on each push

**Example usage in CI:**

```yaml
- uses: jrperez2015/qa-flutter-plugin@v3
  with:
    platform: web
    flutter-path: .
```

---

## v3.x — Trend analysis

Once multiple runs accumulate `release-gate-*.json` artifacts, add an aggregator skill that reads them across time and surfaces regressions (flaky tests, degrading coverage, recurring root-causes).
