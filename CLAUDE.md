# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build & Test Commands

```bash
cargo build                              # Dev build (includes gobby feature by default)
cargo build --release                    # Release build (optimized for size: opt-level=z, LTO, stripped)
cargo build --no-default-features        # Build without gobby daemon integration
cargo test                               # Run all tests
cargo test <test_name>                   # Run a single test (e.g. cargo test test_dedup_identical)
cargo clippy                             # Lint
```

## Architecture

gsqz is a YAML-configurable output compressor for LLM token optimization. It wraps shell commands, captures their output, and applies pattern-based compression pipelines to reduce token usage while preserving critical information.

**Data flow:** CLI parses args → loads layered config → executes shell command → strips ANSI codes → optionally fetches daemon config overrides → matches command against pipeline regexes (first match wins) → applies step sequence → optionally reports savings to daemon → prints compressed output.

**Always exits with code 0** — this is intentional to prevent Claude Code from framing compressed output as an error.

### Key modules

- **`config.rs`** — Layered config system: built-in `config.yaml` → global (`~/.gobby/gsqz.yaml`) → project (`.gobby/gsqz.yaml`) → CLI override. Custom `Visitor` deserializer for the polymorphic `Step` enum.
- **`compressor.rs`** — Orchestrator that compiles pipeline regexes, matches commands, applies steps, and enforces thresholds (min output length, max compressed lines, 95% savings threshold).
- **`daemon.rs`** — Feature-gated (`#[cfg(feature = "gobby")]`) HTTP integration with the gobby daemon for runtime config overrides and savings reporting. All HTTP calls are fire-and-forget with 1s timeouts.
- **`primitives/`** — Four composable operations on line collections:
  - `filter` — Remove lines matching regex patterns
  - `group` — Aggregate lines using one of 8 modes (git_status, pytest_failures, test_failures, lint_by_rule, by_extension, by_directory, by_file, errors_warnings)
  - `dedup` — Collapse consecutive identical/near-identical lines (normalizes numbers for similarity)
  - `truncate` — Keep head+tail with omission marker; supports per-section truncation via file markers

### Config pipeline structure

Pipelines in `config.yaml` match commands via regex and apply ordered steps:
```yaml
pipelines:
  name:
    match: '<regex>'
    steps:
      - filter_lines: { patterns: [...] }
      - group_lines: { mode: <mode> }
      - truncate: { head: 20, tail: 10 }
      - dedup: {}
```

The `fallback` section applies when no pipeline matches. `excluded_commands` skip compression entirely.
