# Changelog

All notable changes to gsqz will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.0] - 2026-03-27

### Added

- YAML-configurable output compression with layered config system
- Four composable compression primitives: filter, group, dedup, truncate
- 20+ built-in pipelines for common CLI tools (git, cargo, pytest, eslint, etc.)
- 8 grouping modes: git_status, pytest_failures, test_failures, lint_by_rule, by_extension, by_directory, by_file, errors_warnings
- Per-section truncation for git diffs
- Optional gobby daemon integration for runtime config overrides and savings reporting
- CI workflow with clippy, rustfmt, and test checks
- Cross-platform release pipeline (macOS, Linux, Windows — x86_64 + ARM64)
- 78 tests across all modules

[0.1.0]: https://github.com/GobbyAI/gsqz/releases/tag/v0.1.0
