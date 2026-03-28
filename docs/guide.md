# gsqz User Guide

gsqz wraps shell commands and compresses their output for LLM consumption. Instead of feeding 500 lines of `git status` or test output into a context window, gsqz reduces it to the essential information.

## Installation

```bash
cargo install --path .
```

## Quick Start

```bash
# Wrap any command
gsqz -- git status
gsqz -- cargo test
gsqz -- uv run pytest tests/

# See compression stats
gsqz --stats -- git diff

# Inspect the resolved config (all layers merged)
gsqz --dump-config
```

When compression is applied, output is prefixed with a header:
```
[Output compressed by gsqz — git-status, 72% reduction]
Modified (3):
  src/main.rs
  src/lib.rs
  src/config.rs
Untracked (1):
  new_file.txt
```

## How It Works

1. gsqz runs your command via `sh -c` and captures stdout + stderr
2. ANSI escape codes are stripped
3. The command string is matched against pipeline regexes (first match wins)
4. The matched pipeline's steps run sequentially, each transforming the line list
5. If no pipeline matches, the fallback steps apply (default: truncate head:20, tail:20)
6. If compression doesn't achieve at least 5% reduction, the original output is returned unchanged

Output under 1000 characters is never compressed (configurable via `min_output_length`).

gsqz always exits with code 0 — the LLM reads pass/fail from the content itself.

## Pipeline Steps

Each pipeline defines an ordered list of steps. Each step takes the current lines and produces new lines.

### `filter_lines`

Removes lines matching any of the given regex patterns. Use this to strip noise — progress bars, blank lines, boilerplate headers.

```yaml
- filter_lines:
    patterns:
      - '^\s*$'            # blank lines
      - '^On branch '      # git status header
      - '^\s*Compiling '   # cargo build progress
```

### `group_lines`

Aggregates lines into a structured summary. Available modes:

| Mode | Use Case | What It Does |
|------|----------|-------------|
| `git_status` | `git status` | Groups files by status (M/A/D/R/C/U/??) with counts. Shows up to 20 files per group. |
| `pytest_failures` | pytest output | Extracts FAILURES/ERRORS sections and the summary line. Drops passing tests entirely. |
| `test_failures` | Any test runner | Detects FAIL/FAILED/ERROR lines and includes everything from the first failure onward. If no failures, outputs "All tests passed." |
| `lint_by_rule` | Linter output | Groups diagnostics by rule code (e.g. E001, W123, `[rule-name]`). Shows up to 5 examples per rule. |
| `by_extension` | `ls`, `tree` | Groups files by extension, sorted by count descending. Shows up to 10 per group. |
| `by_directory` | `find` | Groups paths by parent directory, sorted by count descending. Shows up to 10 per group. |
| `by_file` | `grep`, `rg` | Groups `file:line:match` output by file path. Shows up to 5 matches per file. |
| `errors_warnings` | Build output | Separates errors (up to 20 shown) from warnings (up to 10 shown), plus the last 3 lines of other output. |

### `truncate`

Keeps the first N and last N lines, replacing the middle with `[... X lines omitted ...]`.

```yaml
- truncate:
    head: 20       # keep first 20 lines (default: 20)
    tail: 10       # keep last 10 lines (default: 10)
```

**Per-section mode** truncates each section independently, useful for diffs:

```yaml
- truncate:
    per_file_lines: 50       # max lines per section
    file_marker: '^@@\s'     # regex that marks section boundaries
```

Each section is split at the marker and truncated individually, with `[... X lines omitted in section ...]` markers.

### `dedup`

Collapses consecutive identical or near-identical lines. "Near-identical" means lines that differ only in numbers (e.g. `error at pos 42` and `error at pos 99` are considered duplicates).

```yaml
- dedup: {}
```

Output:
```
error at pos 42
  [repeated 3 times]
```

## Configuration

gsqz uses layered config. Each layer can add new pipelines or override existing ones by name.

| Layer | Path | Purpose |
|-------|------|---------|
| Built-in | Compiled into binary | 28 default pipelines |
| Global | `~/.gobby/gsqz.yaml` | User-wide overrides |
| Project | `.gobby/gsqz.yaml` | Project-specific pipelines |
| CLI | `--config path/to/file.yaml` | One-off override |

Later layers win. Pipelines are merged by name (overlay replaces). Settings only override if they differ from defaults. Excluded commands are additive across layers.

### Settings

```yaml
settings:
  min_output_length: 1000      # skip compression below this char count
  max_compressed_lines: 100    # hard cap on compressed output lines
  daemon_url: "http://..."     # optional gobby daemon URL
```

### Excluding Commands

Commands matching these regexes are never compressed:

```yaml
excluded_commands:
  - '\bcat\b'
  - '\becho\b'
```

### Full Pipeline Example

```yaml
pipelines:
  terraform-plan:
    match: '\bterraform\s+plan\b'
    steps:
      - filter_lines:
          patterns:
            - '^\s*$'
            - '^\s*#'
            - 'Refreshing state'
      - group_lines:
          mode: errors_warnings
      - truncate:
          head: 40
          tail: 10
```

## Debugging

```bash
# See which pipelines are loaded and their match patterns
gsqz --dump-config

# See which strategy matched and compression ratio
gsqz --stats -- your-command-here
```

The `--stats` flag prints to stderr:
```
[gsqz] strategy=pytest original=12847 compressed=1923 savings=85.0%
```

Strategy names to look for:
- A pipeline name (e.g. `git-status`, `pytest`, `cargo-test`) — matched and compressed
- `fallback` — no pipeline matched, generic truncation applied
- `passthrough` — output was too short, compression didn't help enough, or output was empty
- `excluded` — command matched an exclusion pattern
