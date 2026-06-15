<div align="center">
  <img src="./apcore-cli-logo.svg" alt="apcore-cli logo" width="200"/>
</div>

# apcore-cli

[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![Python](https://img.shields.io/badge/python-3.11%2B-blue)](https://github.com/aiperceivable/apcore-cli-python)
[![TypeScript](https://img.shields.io/badge/typescript-node%2018%2B-blue)](https://github.com/aiperceivable/apcore-cli-typescript)
[![Rust](https://img.shields.io/badge/rust-2021%20edition-orange)](https://github.com/aiperceivable/apcore-cli-rust)

**The CLI Adapter for apcore — Expose modules as high-performance, AI-perceivable command-line tools.**

> **Build once, invoke by Code, AI, or Terminal.**

| | |
|---|---|
| **Spec repo** | [github.com/aiperceivable/apcore-cli](https://github.com/aiperceivable/apcore-cli) |
| **Python SDK** | [github.com/aiperceivable/apcore-cli-python](https://github.com/aiperceivable/apcore-cli-python) |
| **TypeScript SDK** | [github.com/aiperceivable/apcore-cli-typescript](https://github.com/aiperceivable/apcore-cli-typescript) |
| **Rust SDK** | [github.com/aiperceivable/apcore-cli-rust](https://github.com/aiperceivable/apcore-cli-rust) |
| **apcore core** | [github.com/aiperceivable/apcore](https://github.com/aiperceivable/apcore) |

---

## What is this?

`apcore-cli` is the "inside-out" adapter for the apcore ecosystem. It takes your **apcore modules** and automatically exposes them as **CLI subcommands** — with zero code changes.

It serves as the terminal-native counterpart to `apcore-mcp` (Model Context Protocol) and `apcore-a2a` (Agent-to-Agent).

```
    apcore Module Registry            apcore-cli Adapter (grouped)
  ━━━━━━━━━━━━━━━━━━━━━━━━━━        ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  math.add(a, b)             ──┐
  math.sub(a, b)             ──┤──→  apcore-cli math add --a 1 --b 2
  system.health.summary()    ──┤──→  apcore-cli apcli health
  git.commit(msg)            ──┤──→  apcore-cli apcli list --tag math
  db.query(sql)              ──┘──→  apcore-cli apcli describe math.add
```

---

## Getting Started

### Step 1: Install

=== "Python"

    ```bash
    pip install apcore-cli
    ```

    **Requirements:** Python 3.11+, pip or poetry

=== "TypeScript"

    ```bash
    npm install apcore-cli
    # or
    pnpm add apcore-cli
    ```

    **Requirements:** Node.js 18+, npm or pnpm

=== "Rust"

    ```toml
    # Cargo.toml
    [dependencies]
    apcore-cli = "0.7"
    ```

    ```bash
    cargo add apcore-cli
    ```

    **Requirements:** Rust 2021 edition, cargo

### Step 2: Try it with example modules

=== "Python"

    ```bash
    git clone https://github.com/aiperceivable/apcore-cli-python.git
    cd apcore-cli-python
    pip install -e ".[dev]"

    export APCORE_EXTENSIONS_ROOT=examples/extensions

    apcore-cli math.add --a 5 --b 10
    # {"sum": 15}
    ```

=== "TypeScript"

    ```bash
    git clone https://github.com/aiperceivable/apcore-cli-typescript.git
    cd apcore-cli-typescript
    pnpm install

    export APCORE_EXTENSIONS_ROOT=examples/extensions

    npx apcore-cli math.add --a 5 --b 10
    # {"sum": 15}
    ```

=== "Rust"

    ```bash
    git clone https://github.com/aiperceivable/apcore-cli-rust.git
    cd apcore-cli-rust
    cargo build --release

    export APCORE_EXTENSIONS_ROOT=examples/extensions

    ./target/release/apcore-cli math.add --a 5 --b 10
    # {"sum": 15}
    ```

### Step 3: Explore available modules

> **v0.8 reminder:** in standalone mode the `apcli` group is visible by default; in embedded mode integrators may hide or filter it (see [features/builtin-group.md](docs/features/builtin-group.md)). Built-ins are reachable via `<cli> apcli <sub>` only — root-level shims (e.g. `apcore-cli list`) were removed in v0.8.0. User business modules (e.g. `apcore-cli math.add`) remain unprefixed.

```bash
# List all discovered modules
apcore-cli apcli list

# Search by keyword
apcore-cli apcli list --search email

# Filter by tag
apcore-cli apcli list --tag math

# Filter by annotation
apcore-cli apcli list --annotation destructive --annotation requires-approval

# See full details for a module
apcore-cli apcli describe math.add

# Validate without executing
apcore-cli apcli validate math.add --input '{"a": 5, "b": "hello"}'
```

### Step 4: STDIN piping & output formats

```bash
# Pipe JSON input
echo '{"a": 100, "b": 200}' | apcore-cli math.add --input -
# {"sum": 300}

# CLI flags override STDIN values
echo '{"a": 1, "b": 2}' | apcore-cli math.add --input - --a 999
# {"sum": 1001}

# Output as CSV
apcore-cli math.add --a 5 --b 10 --format csv

# Select specific fields
apcore-cli sysutil.info --fields os,hostname

# Chain with other tools
apcore-cli sysutil.info --format json | jq '.os, .hostname'
```

### Step 5: Use with your own project

```bash
# One-time setup
export APCORE_EXTENSIONS_ROOT=./extensions

# Or pass it directly
apcore-cli --extensions-dir ./extensions math.add --a 42 --b 58
```

### Step 6: Write your first module

=== "Python"

    ```python
    # extensions/greet/hello.py
    from pydantic import BaseModel

    class Input(BaseModel):
        name: str
        greeting: str = "Hello"

    class Output(BaseModel):
        message: str

    class GreetHello:
        input_schema = Input
        output_schema = Output
        description = "Greet someone by name"

        def execute(self, inputs, context=None):
            return {"message": f"{inputs['greeting']}, {inputs['name']}!"}
    ```

=== "TypeScript"

    ```typescript
    // extensions/greet/hello.ts
    import { module } from "apcore-js";

    export const greetHello = module({
      id: "greet.hello",
      description: "Greet someone by name",
      inputSchema: {
        properties: {
          name: { type: "string" },
          greeting: { type: "string", default: "Hello" },
        },
        required: ["name"],
      },
      execute: async (inputs) => {
        return { message: `${inputs.greeting}, ${inputs.name}!` };
      },
    });
    ```

=== "Rust"

    ```rust
    // extensions/greet/hello.rs
    use apcore::module;
    use serde::{Deserialize, Serialize};
    use serde_json::Value;

    #[derive(Deserialize)]
    struct Input {
        name: String,
        #[serde(default = "default_greeting")]
        greeting: String,
    }

    fn default_greeting() -> String { "Hello".into() }

    #[derive(Serialize)]
    struct Output { message: String }

    pub fn execute(inputs: Value, _ctx: Option<Value>) -> Value {
        let input: Input = serde_json::from_value(inputs).unwrap();
        serde_json::to_value(Output {
            message: format!("{}, {}!", input.greeting, input.name),
        }).unwrap()
    }
    ```

Run it:

```bash
apcore-cli greet.hello --name World
# {"message": "Hello, World!"}
```

---

## Programmatic Usage (Library Embedding)

=== "Python"

    ```python
    from apcore_cli import create_cli

    # Standalone mode
    cli = create_cli(extensions_dir="./extensions", prog_name="myapp")
    cli(standalone_mode=True)

    # Embedded mode (pre-populated registry)
    from apcore import Registry, Executor
    registry = Registry("./extensions")
    registry.discover()
    executor = Executor(registry)
    cli = create_cli(registry=registry, executor=executor, prog_name="myapp")

    # With custom commands
    import click

    @click.command()
    @click.argument("env")
    def deploy(env):
        """Deploy to target environment."""
        click.echo(f"Deploying to {env}...")

    cli = create_cli(
        extensions_dir="./extensions",
        prog_name="myapp",
        extra_commands=[deploy],
    )
    ```

=== "TypeScript"

    ```typescript
    import { createCli } from "apcore-cli";

    // Standalone mode
    const cli = createCli({ extensionsDir: "./extensions", progName: "myapp" });

    // Embedded mode (pre-populated registry)
    const cli = createCli({
      registry: myRegistry,
      executor: myExecutor,
      progName: "myapp",
    });

    // With custom commands
    import { Command } from "commander";

    const deployCmd = new Command("deploy")
      .argument("<env>")
      .action((env) => console.log(`Deploying to ${env}...`));

    const cli = createCli({
      extensionsDir: "./extensions",
      progName: "myapp",
      extraCommands: [deployCmd],
    });
    ```

=== "Rust"

    ```rust
    use apcore_cli::CliConfig;
    use clap::Command;

    // Standalone mode
    let config = CliConfig {
        extensions_dir: Some("./extensions".into()),
        prog_name: Some("myapp".into()),
        ..Default::default()
    };

    // Embedded mode (pre-populated registry)
    let config = CliConfig {
        registry: Some(Arc::new(my_registry)),
        executor: Some(Arc::new(my_executor)),
        prog_name: Some("myapp".into()),
        ..Default::default()
    };

    // With custom commands
    let deploy = Command::new("deploy")
        .arg(clap::Arg::new("env").required(true));

    let config = CliConfig {
        extra_commands: vec![deploy],
        ..Default::default()
    };
    ```

---

## Examples

### Dry-run validation

```bash
# Check if a call would succeed without executing
apcore-cli math.add --a 5 --b hello --dry-run

#   ✓ module_id           OK
#   ✓ module_lookup        OK
#   ✓ call_chain           OK
#   ✓ acl                  OK
#   ✗ schema               {"field": "b", "expected": "integer"}
#
# Result: FAIL (1 error(s), 0 warning(s))
```

### Execution trace

```bash
# See which pipeline steps ran and their timing
apcore-cli math.add --a 5 --b 10 --trace

# {"sum": 15}
#
# Pipeline Trace (strategy: standard, 11 steps, 23.4ms)
#   ✓ context_creation           0.1ms
#   ✓ call_chain_guard           0.0ms
#   ✓ module_lookup              1.2ms
#   ...
```

### System management

```bash
# Health overview
apcore-cli apcli health

# Usage statistics
apcore-cli apcli usage --period 7d

# Disable a module
apcore-cli apcli disable db.dangerous --reason "release freeze" --yes

# View pipeline strategy
apcore-cli apcli describe-pipeline --strategy minimal
```

### Streaming output

```bash
# Stream large results as JSONL
apcore-cli data.export --query "SELECT *" --stream

# Each line is one JSON object, flushed immediately
# {"row": 1, "data": {...}}
# {"row": 2, "data": {...}}
```

---

## Key Features

- **Module Exposure Filtering** -- Declarative `expose` config in `apcore.yaml` controls which modules appear as CLI commands (`all`, `include`, `exclude` modes with glob patterns); hidden modules remain invocable via `exec`
- **Grouped Commands** -- Modules with dots in their names are auto-grouped into nested subcommands (`apcore-cli product list` instead of `apcore-cli product.list`); `display.cli.group` in binding.yaml overrides the auto-detected group
- **Display Overlay** -- `metadata["display"]["cli"]` controls CLI command names, descriptions, and guidance per module (§5.13); set via `binding_path` in `create_cli()` / `fastapi-apcore`
- **Zero-Config Routing** -- Automatically maps module IDs (e.g., `math.add`) to CLI commands
- **Schema-Driven Args** -- Uses `input_schema` to generate CLI arguments, types, and validation
- **Boolean Flag Pairs** -- `--quiet` / `--no-quiet` from `"type": "boolean"` schema properties (applies to any schema-declared boolean; `verbose: boolean` is now a valid schema property as of v0.9.0, when the global `--verbose` flag was retired in favour of `--all-options`)
- **Enum Choices** -- `"enum": ["json", "csv"]` becomes `--format json` with Click validation
- **STDIN Piping** -- `--input -` reads JSON from STDIN, CLI flags override for duplicate keys
- **TTY-Adaptive Output** -- Rich tables for terminals, JSON for pipes (configurable via `--format`)
- **Byte-Equivalent Tabular Output** _(v0.9.0)_ -- `--format csv` and `--format jsonl` delegate to `apcore-toolkit` so output is byte-identical across Python / TypeScript / Rust SDKs. Fixes a class of cross-SDK divergence (Python repr for nested values, TypeScript heterogeneous-keys data loss, Rust `\n` vs CRLF). See [tech-design ADR-09](docs/tech-design.md)
- **Approval Gate** -- TTY-aware HITL prompts for modules with `requires_approval: true`, with `--yes` bypass and 60s timeout
- **Schema Validation** -- Inputs validated against JSON Schema before execution, with `$ref`/`allOf`/`anyOf`/`oneOf` resolution
- **Security** -- API key auth (keyring + AES-256-GCM), append-only audit logging, subprocess sandboxing
- **Shell Completions** -- `apcore-cli apcli completion bash|zsh|fish` generates completion scripts with dynamic module ID completion
- **Man Pages** -- `apcore-cli apcli man <command>` generates roff-formatted man pages; `--help --man` generates a full program man page covering all commands
- **Verbose Help** -- Built-in apcore options are hidden by default; pass `--help --all-options` to show the full option list
- **Documentation Links** -- `set_docs_url()` / `setDocsUrl()` adds online documentation URLs to help footers and man pages
- **Audit Logging** -- All executions logged to `~/.apcore-cli/audit.jsonl` with SHA-256 input hashing

---

## CLI Reference

```
apcore-cli [OPTIONS] COMMAND [ARGS]
```

### Global Options

| Option | Default | Description |
|--------|---------|-------------|
| `--extensions-dir` | `./extensions` | Path to apcore extensions directory |
| `--log-level` | `WARNING` | Logging: `DEBUG`, `INFO`, `WARNING`, `ERROR` |
| `--version` | | Show version and exit |
| `--help` | | Show help and exit |

### Built-in Commands

| Command | Description |
|---------|-------------|
| `list` | List available modules with optional tag filtering, search, and annotation filters |
| `describe <module_id>` | Show full module metadata and schemas |
| `validate <module_id>` | Run preflight checks without executing (dry-run) |
| `exec <module_id>` | Internal routing alias for module execution (modules are also available as direct top-level subcommands) |
| `health [<module_id>]` | Show module health status (requires system modules) |
| `usage [<module_id>]` | Show module usage statistics (requires system modules) |
| `enable <module_id>` | Enable a disabled module at runtime |
| `disable <module_id>` | Disable a module at runtime |
| `reload <module_id>` | Hot-reload a module from disk |
| `config get <key>` | Read a runtime configuration value |
| `config set <key> <value>` | Update a runtime configuration value (requires approval) |
| `describe-pipeline` | Show execution pipeline steps for a strategy |
| `completion <shell>` | Generate shell completion script (bash/zsh/fish) |
| `man <command>` | Generate man page in roff format |

### Module Execution Options

When executing a module (e.g. `apcore-cli math.add`), these built-in options are always available:

| Option | Description |
|--------|-------------|
| `--input -` | Read JSON input from STDIN |
| `--yes` / `-y` | Bypass approval prompts |
| `--large-input` | Allow STDIN input larger than 10MB |
| `--format` | Output format: `json`, `table`, `csv`, `yaml`, `jsonl`, `markdown`, `skill` (list/describe only). **v0.9.0:** `csv` and `jsonl` are now byte-identical across SDKs (delegated to `apcore-toolkit`); see [tech-design ADR-09](docs/tech-design.md) and [features/output-formatter.md](docs/features/output-formatter.md). |
| `--fields` | Comma-separated dot-paths to select from the result |
| `--dry-run` | Run preflight checks without executing |
| `--trace` | Show execution pipeline trace with per-step timing |
| `--stream` | Stream output as JSONL (one JSON object per line) |
| `--strategy` | Execution pipeline: `standard`, `internal`, `testing`, `performance`, `minimal` |
| `--approval-timeout` | Override approval prompt timeout in seconds (default: 60) |
| `--approval-token` | Resume a pending approval with the given token |
| `--sandbox` | Run module in subprocess sandbox |

Schema-generated flags (e.g. `--a`, `--b`) are added automatically from the module's `input_schema`.

### Exit Codes

| Code | Meaning |
|------|---------|
| `0` | Success |
| `1` | Module execution error |
| `2` | Invalid CLI input |
| `44` | Module not found / disabled / load error |
| `45` | Schema validation error |
| `46` | Approval denied or timed out |
| `47` | Configuration error |
| `48` | Schema circular reference |
| `77` | ACL denied |
| `130` | Execution cancelled (Ctrl+C) |

---

## Configuration

apcore-cli uses a 4-tier configuration precedence:

1. **CLI flag** (highest): `--extensions-dir ./custom`
2. **Environment variable**: `APCORE_EXTENSIONS_ROOT=./custom`
3. **Config file**: `apcore.yaml`
4. **Default** (lowest): `./extensions`

### Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `APCORE_EXTENSIONS_ROOT` | Path to extensions directory | `./extensions` |
| `APCORE_CLI_AUTO_APPROVE` | Set to `1` to bypass all approval prompts | *(unset)* |
| `APCORE_CLI_LOGGING_LEVEL` | CLI-specific log level (takes priority over `APCORE_LOGGING_LEVEL`) | `WARNING` |
| `APCORE_LOGGING_LEVEL` | Global apcore log level (fallback when `APCORE_CLI_LOGGING_LEVEL` is unset) | `WARNING` |
| `APCORE_AUTH_API_KEY` | API key for remote registry authentication | *(unset)* |
| `APCORE_CLI_SANDBOX` | Set to `1` to enable subprocess sandboxing | *(unset)* |
| `APCORE_CLI_HELP_TEXT_MAX_LENGTH` | Maximum characters for CLI option help text before truncation | `1000` |
| `APCORE_CLI_APPROVAL_TIMEOUT` | Approval prompt timeout in seconds | `60` |
| `APCORE_CLI_STRATEGY` | Default execution pipeline strategy | `standard` |
| `APCORE_CLI_GROUP_DEPTH` | Multi-level grouping depth for nested subcommands | `1` |
| `APCORE_CLI_EXPOSE_MODE` | Module exposure mode: `all`, `include`, `exclude` | `all` |

### Config File (`apcore.yaml`)

```yaml
extensions:
  root: ./extensions
logging:
  level: DEBUG
sandbox:
  enabled: false
cli:
  help_text_max_length: 1000
  approval_timeout: 60
  strategy: standard
  group_depth: 1
expose:
  mode: all               # "all" | "include" | "exclude"
  include: []             # Glob patterns (used when mode is "include")
  exclude: []             # Glob patterns (used when mode is "exclude")
```

---

## How It Works

### Mapping: apcore to CLI

| apcore | CLI |
|--------|-----|
| `metadata["display"]["cli"]["alias"]` or `module_id` | Command name — auto-grouped by first `.` segment (`apcore-cli math add`) |
| `metadata["display"]["cli"]["description"]` or `description` | `--help` text |
| `input_schema.properties` | CLI flags (`--a`, `--b`) |
| `input_schema.required` | Required flag enforcement |
| `annotations.requires_approval` | HITL approval prompt |

### Architecture

```
User / AI Agent (terminal)
    |
    v
apcore-cli (the adapter)
    |
    +-- ConfigResolver       4-tier config precedence
    +-- GroupedModuleGroup   Auto-grouped nested Click commands (extends LazyModuleGroup)
    +-- SchemaParser         JSON Schema -> Click options
    +-- RefResolver          $ref / allOf / anyOf / oneOf
    +-- ApprovalGate         TTY-aware HITL approval
    +-- OutputFormatter      TTY-adaptive JSON/table output
    +-- AuditLogger          JSON Lines execution logging
    +-- Sandbox              Subprocess isolation
    |
    v
apcore Registry + Executor (your modules, unchanged)
```

---

## SDKs

| Language | Repository | Status |
|----------|-----------|--------|
| **Python** | [apcore-cli-python](https://github.com/aiperceivable/apcore-cli-python) | v0.10.1 |
| **TypeScript** | [apcore-cli-typescript](https://github.com/aiperceivable/apcore-cli-typescript) | v0.10.1 |
| **Rust** | [apcore-cli-rust](https://github.com/aiperceivable/apcore-cli-rust) | v0.10.1 |

## Version Compatibility

apcore-cli is part of the broader apcore ecosystem; packages that share a
minor version line are tested to work together. Snapshot below is the
**currently tested combination** (2026-06-15). Full cross-ecosystem matrix
lives in [`apcore` README](https://github.com/aiperceivable/apcore#version-compatibility).

| Component | Required version | Notes |
|---|---|---|
| `apcore` core SDK | `>= 0.24.0` (tested with 0.24.0) | apcore-cli-python / -typescript pin `>=0.24.0` (open); apcore-cli-rust pins `apcore = "0.24"` (caret 0.24.x) |
| `apcore-toolkit` | `>= 0.8.1` (tested with 0.8.1) | **required** runtime dep (no soft fallback as of 0.10.0); per-language pin varies — see 6.8 note below |

### Known dependency-pin divergence (tracked as issue 6.8)

| CLI package | apcore-toolkit pin | Effective range |
|---|---|---|
| apcore-cli-python | `apcore-toolkit>=0.8.1` | open upper bound — accepts future toolkit minors |
| apcore-cli-typescript | `"apcore-toolkit": ">=0.8.1"` | open upper bound |
| apcore-cli-rust | `apcore-toolkit = "=0.8.1"` | **exact pin** — blocks future toolkit minors until manually bumped |

Reconciliation (adopting consistent caret semantics across all three) is
planned for a follow-up coordinated release.

---

## Project Structure

This repository contains the **specification and design documents**:

```
apcore-cli/
├── docs/
│   ├── project-apcore-cli.md  Project manifest (12 features)
│   ├── tech-design.md         Tech Design v2.0
│   ├── srs.md                 Software Requirements Specification
│   └── features/
│       ├── overview.md                 Feature index
│       ├── core-dispatcher.md          FE-01: CLI entry point
│       ├── schema-parser.md            FE-02: JSON Schema → flags
│       ├── approval-gate.md            FE-03: HITL approval
│       ├── discovery.md                FE-04: list / describe
│       ├── config-resolver.md          FE-07: 4-tier config
│       ├── output-formatter.md         FE-08: TTY-adaptive output
│       ├── security.md                 FE-05: Auth, encryption
│       ├── shell-integration.md        FE-06: Completions, man pages
│       ├── grouped-commands.md         FE-09: Nested subcommands
│       ├── init-command.md             FE-10: Module scaffolding
│       ├── usability-enhancements.md   FE-11: v0.6.0 enhancements
│       └── exposure-filtering.md      FE-12: Module exposure filtering
├── conformance/               Cross-language test fixtures
├── CHANGELOG.md
└── README.md
```

For the **implementation**, see the SDKs:

=== "Python"

    ```
    apcore-cli-python/src/apcore_cli/
    ├── __main__.py          Entry point, create_cli()
    ├── cli.py               Dispatcher, GroupedModuleGroup
    ├── approval.py          HITL approval + CliApprovalHandler
    ├── output.py            TTY-adaptive formatting
    ├── discovery.py         list, describe, validate commands
    ├── system_cmd.py        health, usage, enable, disable, reload, config
    ├── strategy.py          describe-pipeline command
    ├── config.py            4-tier ConfigResolver
    ├── schema_parser.py     JSON Schema → Click options
    ├── shell.py             Completion + man pages
    └── security/            Audit, auth, encryption, sandbox
    ```

=== "TypeScript"

    ```
    apcore-cli-typescript/src/
    ├── main.ts              Entry point, createCli()
    ├── cli.ts               LazyModuleGroup, GroupedModuleGroup
    ├── approval.ts          HITL approval + CliApprovalHandler
    ├── output.ts            TTY-adaptive formatting
    ├── discovery.ts         list, describe, validate commands
    ├── system-cmd.ts        health, usage, enable, disable, reload, config
    ├── strategy.ts          describe-pipeline command
    ├── config.ts            4-tier ConfigResolver
    ├── schema-parser.ts     JSON Schema → Commander options
    ├── shell.ts             Completion + man pages
    └── security/            Audit, auth, encryption, sandbox
    ```

=== "Rust"

    ```
    apcore-cli-rust/src/
    ├── main.rs              Entry point, dispatch routing
    ├── cli.rs               Dispatcher, module execution
    ├── lib.rs               CliConfig, exit codes
    ├── approval.rs          HITL approval + CliApprovalHandler
    ├── output.rs            TTY-adaptive formatting
    ├── discovery.rs         list, describe commands
    ├── validate.rs          validate command
    ├── system_cmd.rs        health, usage, enable, disable, reload, config
    ├── strategy.rs          describe-pipeline command
    ├── config.rs            4-tier ConfigResolver
    ├── schema_parser.rs     JSON Schema → clap args
    ├── shell.rs             Completion + man pages
    └── security/            Audit, auth, encryption, sandbox
    ```

---

## Related Projects

| Project | Description |
|---------|-------------|
| [apcore](https://github.com/aiperceivable/apcore) | Core protocol and framework for AI-Perceivable modules |
| [apcore-mcp](https://github.com/aiperceivable/apcore-mcp) | Model Context Protocol adapter |
| [apcore-a2a](https://github.com/aiperceivable/apcore-a2a) | Agent-to-Agent adapter |
| [django-apcore](https://github.com/aiperceivable/django-apcore) | Django integration |
| [flask-apcore](https://github.com/aiperceivable/flask-apcore) | Flask integration |

## License

Apache-2.0
