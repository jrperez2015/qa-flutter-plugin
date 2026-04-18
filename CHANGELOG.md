# Changelog

All notable changes to this project will be documented in this file.

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).
This project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0] — 2026-04-18

### Added
- `qa-flutter:qa-stability-agent` — orchestrator agent that auto-detects Flutter platform and routes to the correct runner
- `qa-flutter-android-runner` — Appium-based QA runner for Flutter Android apps
- `qa-flutter-android-tester` — single-feature Appium tester (sub-agent of android-runner)
- `qa-flutter-web-runner` — stability checker for Flutter web apps; runs flutter analyze, flutter test, flutter build web, git diff analysis, and generates a functional test checklist
- Generic configuration: all hardcoded paths removed; skills prompt users for paths at runtime
