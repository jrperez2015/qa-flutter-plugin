# Roadmap

## v1.x — Stable (current)

Manual installation. Three skills (Android runner, Android tester, Web runner) + one orchestrator agent.

**v1.1 planned:**
- iOS runner (XCUITest-based)
- Webhook notifications on test failure

---

## v2.0 — Marketplace-ready

**Goal:** `/plugin install qa-flutter` works without manual `installed_plugins.json` editing.

**Required work:**
1. Register on `claude-plugins-official` marketplace or host a custom marketplace JSON
2. Set up GitHub Releases triggered by semver tags (`git tag v2.0.0 && git push --tags`)
3. Add `version` bumping to CHANGELOG workflow
4. Validate plugin loads correctly via marketplace install path

**Migration from v1:** Users who installed manually can switch by running `/plugin install qa-flutter` and removing the manual entry from `installed_plugins.json`.

---

## v2.x — Extended platforms

- `qa-flutter-ios-runner` — XCUITest via `idb` or `xcrun simctl`
- `qa-flutter-desktop-runner` — validates desktop builds compile and launch

---

## v3.0 — CI/CD native

**Goal:** Run QA checks in CI without a human in the loop.

**Required work:**
1. Package as a GitHub Actions composite action (`action.yml`)
2. Web runner produces machine-readable JSON output (alongside Markdown checklist)
3. Android runner supports headless emulator via GitHub-hosted runners or self-hosted
4. PR comment bot posts checklist summary on each push

**Example usage in CI:**
```yaml
- uses: YOUR_USERNAME/qa-flutter-plugin@v3
  with:
    platform: web
    flutter-path: .
```
