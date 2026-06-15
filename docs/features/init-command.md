---
description: "Init Command (FE-10) scaffolding subcommand (apcli init module <id>) generating module boilerplate in decorator, convention, or binding styles for a zero-config scaffold-then-run workflow."
---

# Feature Spec: Init Command

**Feature ID**: FE-10
**Status**: Ready for Implementation
**Priority**: P1
**Parent**: [Tech Design v2.0](../tech-design.md) Section 8.12

**SRS Requirements**: FR-USB-001, FR-DISC-001, NFR-USB-001

---

## 1. Description

The Init Command feature provides a scaffolding subcommand (`apcore-cli init module <module_id>`) that generates boilerplate files for new apcore modules. It supports three styles — decorator, convention, and binding — each producing the appropriate file structure with sensible defaults. Combined with the `--commands-dir` CLI option on `create_cli()` and `ConventionScanner` integration, this enables a zero-configuration workflow: scaffold a module, then run the CLI immediately.

---

## 2. Requirements Traceability

| Req ID | SRS Ref | Description |
|--------|---------|-------------|
| FR-10-01 | FR-USB-001 | `apcore-cli init module <module_id>` creates a valid module file from a template. |
| FR-10-02 | FR-USB-001 | `--style decorator` generates a `@module`-decorated function in `extensions/`. |
| FR-10-03 | FR-USB-001 | `--style convention` generates a plain function in `commands/` with metadata constants. |
| FR-10-04 | FR-USB-001 | `--style binding` generates a `.binding.yaml` file in `bindings/` and a companion source file in `commands/`. |
| FR-10-05 | NFR-USB-001 | `--dir` overrides the default output directory for any style. |
| FR-10-06 | NFR-USB-001 | `--description` sets the module description in generated files. |
| FR-10-07 | FR-DISC-001 | `--commands-dir` CLI option on `create_cli()` triggers ConventionScanner at startup. |
| FR-10-08 | FR-DISC-001 | ConventionScanner import failure is handled gracefully with a WARNING log. |

---

## 3. Module Path

`apcore_cli/init_cmd.py` (function: `register_init_command`)
`apcore_cli/__main__.py` (function: `create_cli`, `--commands-dir` integration)

---

## 4. Implementation Details

## Contract: register_init_command

### Inputs
- cli: click.Group, required — Group to register the `init` command on.

### Errors
- (none raised)

### Returns
- On success: None — `init` group with `module` subcommand registered on `cli`

### Properties
- async: false
- thread_safe: false (mutates cli)
- pure: false (mutates cli group)

---

### 4.1 Command Structure

```
apcore-cli init module <MODULE_ID> [--style STYLE] [--dir DIR] [--description TEXT]
```

`init` is registered as a `click.Group` on the root CLI. `module` is a subcommand of `init`.

**Registration**: `register_init_command(cli)` is called in `create_cli()` after all other command registrations.

### 4.2 `MODULE_ID` Parsing

The `module_id` argument is split on the **last** `.` to derive `(prefix, func_name)`:

- `ops.deploy` -> prefix=`ops`, func_name=`deploy`
- `user.create` -> prefix=`user`, func_name=`create`
- `standalone` -> prefix=`standalone`, func_name=`standalone` (no dot)

### 4.3 Style: `decorator` (default dir: `extensions/`)

Generates a single source file at `{dir}/{module_id_with_underscores}.{ext}` using the host language's module definition syntax. Each CLI implementation generates code in its own language:

**Python** (`.py`):
```python
from apcore import module

@module(id="{module_id}", description="{description}")
def {func_name}() -> dict:
    # TODO: implement
    return {"status": "ok"}
```

**TypeScript** (`.ts`):
```typescript
import { module } from "apcore-js";
import { Type } from "@sinclair/typebox";

export const {funcName}Module = module({
  id: "{module_id}",
  description: "{description}",
  inputSchema: Type.Object({}),
  outputSchema: Type.Object({ status: Type.String() }),
  execute: (_inputs) => {
    // TODO: implement
    return { status: "ok" };
  },
});
```

**Rust** (`.rs`):
```rust
use apcore::module::Module;
use apcore::context::Context;
use apcore::errors::ModuleError;
use async_trait::async_trait;
use serde_json::{json, Value};

pub struct {FuncName}Module;

#[async_trait]
impl Module for {FuncName}Module {
    fn description(&self) -> &str { "{description}" }
    // ... input_schema, output_schema, execute
}
```

### 4.4 Style: `convention` (default dir: `commands/`)

Generates a source file in a directory structure derived from the prefix. Multi-segment prefixes create nested directories. Each implementation uses its own language syntax.

The `CLI_GROUP` constant is only emitted when the `module_id` contains a `.` separator. Language-specific syntax:
- Python: `CLI_GROUP = "{group}"`
- TypeScript: `export const CLI_GROUP = "{group}";`
- Rust: `pub const CLI_GROUP: &str = "{group}";`

### 4.5 Style: `binding` (default dir: `bindings/`)

Generates two files:

1. **Binding YAML** at `{dir}/{module_id_with_underscores}.binding.yaml`:

```yaml
bindings:
  - module_id: "{module_id}"
    target: "commands.{prefix}:{func_name}"
    description: "{description}"
    auto_schema: true
```

2. **Companion source** at `commands/{prefix_with_underscores}.{ext}` (only if it does not already exist), using the host language's function syntax.

### 4.6 Options

| Option | Short | Default | Description |
|--------|-------|---------|-------------|
| `--style` | — | `convention` | Module style: `decorator`, `convention`, or `binding`. |
| `--dir` | — | Style-dependent | Output directory. Defaults to `extensions/`, `commands/`, or `bindings/` depending on style. |
| `--description` | `-d` | `"TODO: add description"` | Module description embedded in generated files. |

### 4.7 `--commands-dir` on `create_cli()`

**File**: `apcore_cli/__main__.py`

`create_cli()` accepts an optional `commands_dir: str | None` parameter. When set:

1. Import `ConventionScanner` from `apcore_toolkit.convention_scanner`.
2. Import `RegistryWriter` from `apcore_toolkit`.
3. Call `scanner.scan(commands_dir)` to discover convention modules.
4. If modules are found, write them to the registry via `RegistryWriter`.
5. Log the count at INFO level.

```python
if commands_dir is not None:
    try:
        from apcore_toolkit.convention_scanner import ConventionScanner
        from apcore_toolkit import RegistryWriter

        conv_scanner = ConventionScanner()
        conv_modules = conv_scanner.scan(commands_dir)
        if conv_modules:
            writer = RegistryWriter(registry=registry)
            writer.write(conv_modules)
            logger.info("Convention scanner: registered %d modules from %s",
                        len(conv_modules), commands_dir)
    except ImportError:
        logger.warning("apcore-toolkit not installed — convention module scanning unavailable")
    except Exception as e:
        logger.warning("Convention module scanning failed: %s", e)
```

The `--commands-dir` value is also exposed as a CLI option so users can pass it at invocation time:

```
apcore-cli --commands-dir ./my-commands list
```

At the `main()` entry point, the value is extracted from `sys.argv` before Click parses it (via `_extract_commands_dir()`), ensuring the scanner runs during `create_cli()` initialization.

### 4.8 Graceful `ImportError` Handling

`apcore-toolkit` is an optional dependency of `apcore-cli`. If it is not installed:

- The `ImportError` is caught and a WARNING is logged: `"apcore-toolkit not installed — convention module scanning unavailable"`.
- The CLI continues to function normally with only decorator-based and binding-based modules.
- No traceback is shown to the user.

---

## 5. Boundary Values & Limits

| Parameter | Minimum | Maximum | Default | Reference |
|-----------|---------|---------|---------|-----------|
| `module_id` length | 1 char | 128 chars | — | PROTOCOL_SPEC module_id constraint |
| `description` length | 0 chars | Unbounded | `"TODO: add description"` | — |
| Nested directory depth (convention) | 0 levels | Unbounded | — | Derived from prefix dot segments |

---

## 6. Error Handling

| Condition | Exit Code | Error Message | Reference |
|-----------|-----------|---------------|-----------|
| `commands_dir` does not exist | N/A (WARNING) | WARNING log from ConventionScanner, empty module list returned. | FR-10-07 |
| `apcore-toolkit` not installed | N/A (WARNING) | WARNING log: "apcore-toolkit not installed — convention module scanning unavailable". | FR-10-08 |
| Output directory not writable | 1 | OS-level `PermissionError` propagated by Click. | FR-10-01 |
| Binding companion file already exists | N/A | File is not overwritten; only the `.binding.yaml` is created. | FR-10-04 |

---

## 7. Verification

| Test ID | Description | Expected Result |
|---------|-------------|-----------------|
| T-INIT-01 | `apcore-cli init module ops.deploy --style decorator` | Creates `extensions/ops_deploy.py` with `@module` decorator. |
| T-INIT-02 | `apcore-cli init module ops.deploy --style convention` | Creates `commands/ops.py` with `CLI_GROUP = "ops"` and `deploy()` function. |
| T-INIT-03 | `apcore-cli init module ops.deploy --style binding` | Creates `bindings/ops_deploy.binding.yaml` and `commands/ops.py`. |
| T-INIT-04 | `apcore-cli init module standalone` (no dot in ID) | Creates file with `func_name = "standalone"`, no `CLI_GROUP` line. |
| T-INIT-05 | `--dir /tmp/custom` overrides output directory | File created under `/tmp/custom/` instead of default. |
| T-INIT-06 | `--description "My module"` sets description in generated file | Generated file contains `"My module"` in docstring and decorator/binding. |
| T-INIT-07 | `create_cli(commands_dir="commands/")` with apcore-toolkit installed | ConventionScanner runs, modules registered in registry. |
| T-INIT-08 | `create_cli(commands_dir="commands/")` without apcore-toolkit | WARNING logged, CLI starts normally with zero convention modules. |
| T-INIT-09 | Binding style with existing companion `.py` file | Only `.binding.yaml` created; existing source file preserved. |
| T-INIT-10 | `apcore-cli --commands-dir ./cmds list` | Convention modules from `./cmds` appear in the module list. |
