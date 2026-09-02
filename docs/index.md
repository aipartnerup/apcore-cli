# apcore-cli

**The CLI Adapter for apcore — Expose modules as high-performance, AI-perceivable command-line tools.**

> Build once, invoke by Code, AI, or Terminal.
> Every call validated, authorized, and evidenced.

apcore-cli takes your **apcore modules** and automatically exposes them as **CLI subcommands** — with zero code changes. It is the terminal-native counterpart to `apcore-mcp` (Model Context Protocol) and `apcore-a2a` (Agent-to-Agent).

## Key Features

- **Zero-code CLI generation** — `create_cli()` scans your extensions directory and registers every module as a subcommand automatically.
- **AI-perceivable output** — `--format json` emits structured JSON consumed by AI agents; TTY mode renders rich tables for humans.
- **Full apcore pipeline integration** — `--dry-run`, `--trace`, `--strategy`, and `--stream` expose the apcore execution pipeline directly from the terminal.
- **Cross-language** — identical CLI behaviour across Python, TypeScript, and Rust SDKs, verified by the shared conformance suite.
- **Exposure filtering** — `expose:` in `apcore.yaml` controls which modules appear as commands (whitelist / blacklist / glob patterns).

## Quick Install

=== "Python"
    ```bash
    pip install apcore-cli
    ```

=== "TypeScript"
    ```bash
    npm install apcore-cli
    ```

=== "Rust"
    ```bash
    cargo add apcore-cli
    ```

## Documentation

- [Getting Started](getting-started.md) — quickstart guide
- [Project Manifest](project-apcore-cli.md) — scope, features, roadmap
- [Software Requirements](srs.md) — functional and non-functional requirements
- [Technical Design](tech-design.md) — architecture and implementation details
- [Feature Specs](features/overview.md) — per-feature detail pages

## Implementations

apcore-cli has three language implementations that share this specification:

- [apcore-cli-python](https://github.com/aiperceivable/apcore-cli-python)
- [apcore-cli-typescript](https://github.com/aiperceivable/apcore-cli-typescript)
- [apcore-cli-rust](https://github.com/aiperceivable/apcore-cli-rust)
