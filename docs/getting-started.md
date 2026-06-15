---
description: "Getting-started guide for apcore-cli: pip install, running the Python SDK example modules, discovering modules, and building your own, in under five minutes."
---

# Getting Started

This guide walks you through installing `apcore-cli`, running your first module, and building your own — all in under 5 minutes.

---

## Prerequisites

- **Python 3.11+**
- An [apcore](https://github.com/aiperceivable/apcore) project with an extensions directory, or the example modules from the Python SDK

---

## Install

```bash
pip install apcore-cli
```

Verify the installation:

```bash
apcore-cli --version
```

---

## Quick start with example modules

The Python SDK ships with 8 example modules you can use immediately:

```bash
git clone https://github.com/aiperceivable/apcore-cli-python.git
cd apcore-cli-python
pip install -e ".[dev]"

export APCORE_EXTENSIONS_ROOT=examples/extensions
```

Run a module:

```bash
apcore-cli math.add --a 5 --b 10
# {"sum": 15}
```

---

## Discover modules

> **v0.8 reminder:** built-in commands live under the `apcli` group (`<cli> apcli <sub>`); root-level shims (e.g. `apcore-cli list`) were removed in v0.8.0. In standalone mode `apcli` is visible by default; embedded hosts can hide or filter it (see [features/builtin-group.md](features/builtin-group.md)). User business modules (e.g. `apcore-cli math.add`) remain unprefixed.

### List all modules

```bash
apcore-cli apcli list
```

In an interactive terminal you get a formatted table; when piped, output switches to JSON automatically.

### Filter by tag

```bash
apcore-cli apcli list --tag math
```

### Inspect a module

```bash
apcore-cli apcli describe math.add
```

This shows the module's description, input/output schemas, tags, and annotations (including `requires_approval`).

---

## Execute modules

### Basic execution

Flags are generated automatically from the module's `input_schema`:

```bash
apcore-cli math.add --a 42 --b 58
# {"sum": 100}
```

### STDIN piping

Use `--input -` to read JSON from STDIN. CLI flags override duplicate keys:

```bash
echo '{"a": 100, "b": 200}' | apcore-cli math.add --input -
# {"sum": 300}

# --a overrides the STDIN value
echo '{"a": 1, "b": 2}' | apcore-cli math.add --input - --a 999
# {"sum": 1001}
```

### Chain with other tools

```bash
apcore-cli sysutil.info | jq '.os, .hostname'
```

### Approval gate

Modules annotated with `requires_approval: true` will prompt for confirmation before execution. To bypass in scripts:

```bash
apcore-cli dangerous.operation --yes
# or
export APCORE_CLI_AUTO_APPROVE=1
```

### Output format

Force a specific format regardless of TTY detection:

```bash
apcore-cli apcli list --format json     # always JSON
apcore-cli apcli list --format table    # always table
```

---

## Write your first module

Create a Python file in your extensions directory:

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

Run it:

```bash
apcore-cli greet.hello --name World
# {"message": "Hello, World!"}
```

The module ID (`greet.hello`) is derived from the directory structure — no registration needed.

!!! tip "Schema-driven flags"
    `--name` is required (no default in `Input`), `--greeting` is optional with default `"Hello"`. Boolean fields become `--flag / --no-flag` pairs. Enum fields become choice options with validation.

---

## Configuration

`apcore-cli` resolves configuration in 4 tiers (highest wins):

| Priority | Source | Example |
|----------|--------|---------|
| 1 (highest) | CLI flag | `--extensions-dir ./custom` |
| 2 | Environment variable | `APCORE_EXTENSIONS_ROOT=./custom` |
| 3 | Config file | `apcore.yaml` |
| 4 (lowest) | Default | `./extensions` |

### Minimal config file

```yaml
# apcore.yaml
extensions:
  root: ./extensions
logging:
  level: DEBUG
sandbox:
  enabled: false
```

### Key environment variables

| Variable | Description | Default |
|----------|-------------|---------|
| `APCORE_EXTENSIONS_ROOT` | Path to extensions directory | `./extensions` |
| `APCORE_CLI_AUTO_APPROVE` | Set `1` to bypass approval prompts | *(unset)* |
| `APCORE_LOGGING_LEVEL` | Log level (`DEBUG`, `INFO`, `WARNING`, `ERROR`) | `WARNING` |

For the full configuration reference, see the [Config Resolver spec](features/config-resolver.md).

---

## Shell completions

Generate and install completion scripts for your shell:

=== "Bash"

    ```bash
    apcore-cli apcli completion bash > ~/.apcore-cli-complete.bash
    echo 'source ~/.apcore-cli-complete.bash' >> ~/.bashrc
    ```

=== "Zsh"

    ```bash
    apcore-cli apcli completion zsh > ~/.apcore-cli-complete.zsh
    echo 'source ~/.apcore-cli-complete.zsh' >> ~/.zshrc
    ```

=== "Fish"

    ```bash
    apcore-cli apcli completion fish > ~/.config/fish/completions/apcore-cli.fish
    ```

---

## Common exit codes

| Code | Meaning |
|------|---------|
| `0` | Success |
| `1` | Module execution error |
| `2` | Invalid CLI input |
| `44` | Module not found / disabled / load error |
| `45` | Schema validation error |
| `46` | Approval denied or timed out |
| `47` | Configuration error |

For the full exit code reference, see the [Tech Design](tech-design.md) Section 8.9.

---

## Next steps

- [Config Resolver](features/config-resolver.md) — Full configuration reference
- [Tech Design](tech-design.md) -- Architecture and design decisions
- [Feature Specs](features/overview.md) -- Detailed specs for each component
