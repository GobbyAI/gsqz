# Contributing to gsqz

Thanks for your interest in contributing to gsqz! This project is part of the [Gobby](https://github.com/GobbyAI/gobby) suite.

## Getting Started

```bash
git clone https://github.com/GobbyAI/gsqz.git
cd gsqz
cargo build
cargo test
```

The `rust-toolchain.toml` file ensures you have the right toolchain and components (clippy, rustfmt) installed automatically.

## Development

### Building

```bash
cargo build                          # Dev build (includes gobby feature)
cargo build --no-default-features    # Build without gobby daemon integration
cargo build --release                # Optimized release build
```

### Testing

```bash
cargo test                   # Run all tests
cargo test <test_name>       # Run a single test
cargo clippy -- -D warnings  # Lint
cargo fmt --check            # Check formatting
```

All PRs must pass CI (fmt, clippy, tests) before merging.

### Project Structure

```
src/
  main.rs          — CLI entry point, command execution, ANSI stripping
  config.rs        — Layered config loading, step deserialization
  compressor.rs    — Pipeline matching, step orchestration, thresholds
  daemon.rs        — Optional gobby daemon HTTP integration
  primitives/
    filter.rs      — Remove lines by regex pattern
    group.rs       — Aggregate lines by mode (8 modes)
    truncate.rs    — Head + tail with omission markers
    dedup.rs       — Collapse consecutive similar lines
```

### Adding a Pipeline

Add a new entry to the project's `config.yaml`:

```yaml
pipelines:
  my-tool:
    match: '\bmy-tool\b'    # Regex matched against the full command
    steps:
      - filter_lines:
          patterns:
            - '^\s*$'       # Remove blank lines
      - group_lines:
          mode: errors_warnings
      - truncate:
          head: 20
          tail: 10
      - dedup: {}
```

### Adding a Group Mode

1. Add the function in `src/primitives/group.rs`
2. Add the mode name to the `group_lines()` dispatcher match
3. Add tests
4. Document in the README

## Pull Requests

- Keep PRs focused — one feature or fix per PR
- Add tests for new functionality
- Run `cargo fmt` before committing
- Write clear commit messages

## Reporting Issues

[Open an issue](https://github.com/GobbyAI/gsqz/issues/new) with:
- What you expected to happen
- What actually happened
- The command and output (if applicable)
- Your platform and gsqz version (`gsqz --version`)

## License

By contributing, you agree that your contributions will be licensed under the [Apache 2.0 License](LICENSE).
