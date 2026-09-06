---
description: "Project manifest for apcore-cli: the FE-01 through FE-15 feature table with descriptions and priorities, requirement traceability matrix, and project dependencies."
---

# Project Manifest: apcore-cli

**Project Name**: apcore-cli
**Strategy**: Multi-Split
**Status**: Released v0.6.0
**Date**: 2026-04-06

---

## FEATURE_MANIFEST

| ID | Feature Name | Description | Priority | Specification |
|:---|:---|:---|:---|:---|
| **FE-01** | **Core Dispatcher** | CLI entry point, Registry loading, STDIN piping, module execution via Executor. | P0 | [core-dispatcher.md](features/core-dispatcher.md) |
| **FE-02** | **Schema Parser** | Automatic generation of flags/options from JSON Schema `input_schema`, including `$ref` resolution. | P0 | [schema-parser.md](features/schema-parser.md) |
| **FE-03** | **Approval Gate** | TTY-aware HITL logic for modules requiring approval, with bypass and timeout. | P1 | [approval-gate.md](features/approval-gate.md) |
| **FE-04** | **Discovery** | Terminal-optimized `list` and `describe` commands with tag filtering and format selection. | P1 | [discovery.md](features/discovery.md) |
| **FE-05** | **Security Manager** | API key auth, encrypted config (keyring + AES-256-GCM), audit logging, subprocess sandboxing. | P1/P2 | [security.md](features/security.md) |
| **FE-06** | **Shell Integration** | Shell completion scripts (bash/zsh/fish) and man page generation. | P2 | [shell-integration.md](features/shell-integration.md) |
| **FE-07** | **Config Resolver** | 4-tier configuration precedence (CLI > Env > File > Default). | P0 | [config-resolver.md](features/config-resolver.md) |
| **FE-08** | **Output Formatter** | TTY-adaptive output formatting (JSON for pipes, tables for terminals). | P1 | [output-formatter.md](features/output-formatter.md) |
| **FE-09** | **Grouped Commands** | Auto-grouping of dotted module IDs into nested Click groups with `display.cli.group` override and collapsed root help. | P1 | [grouped-commands.md](features/grouped-commands.md) |
| **FE-10** | **Init Command** | Scaffolding subcommand (`apcore-cli init module <id>`) for decorator/convention/binding styles. | P1 | [init-command.md](features/init-command.md) |
| **FE-11** | **Usability Enhancements** | Dry-run, system commands, error guidance, trace, streaming, enhanced discovery, strategy selection, output formats, multi-level grouping, custom commands. | P0–P2 | [usability-enhancements.md](features/usability-enhancements.md) |
| **FE-12** | **Module Exposure Filtering** | Declarative `expose` config (all/include/exclude) with glob patterns to filter which modules surface as CLI commands. | P1 | [exposure-filtering.md](features/exposure-filtering.md) |
| **FE-13** | **Built-in Command Group (`apcli`)** | Relocates all apcore-cli-provided built-in commands under a reserved `apcli` group. Root keeps only meta-commands + business modules. Supports visibility modes (`all`/`none`/`include`/`exclude`) and `disable_env` opt-out for env-var override. Embedded-mode auto-hide. | P0 | [builtin-group.md](features/builtin-group.md) |
| **FE-14** | **ACL Governance** | Attaches an apcore `ACL` to the CLI's Executor from `acl.root`, and adds the `apcli acl` group (`list`/`check`/`validate`/`status`) for authoring, evaluating, and linting access-control rules from the terminal. | P1 | [acl-governance.md](features/acl-governance.md) |
| **FE-15** | **OpenAPI Import** | **FE-15a**: `apcli openapi scan` / `generate` — turns an OpenAPI 3.x document into `ScannedModule` form and binding artifacts via the apcore-toolkit `OpenAPIScanner`. No registry, no execution. **FE-15b (deferred)**: binding-driven and startup HTTP proxying, blocked on apcore-toolkit parameter-location metadata and on `--binding` registration parity in the TypeScript and Rust SDKs. | P1 | [openapi-import.md](features/openapi-import.md) |

---

## Requirement Traceability

Maps high-level requirements (from [ideas/draft.md](../ideas/draft.md)) to feature-level requirements.

| Idea Req | Description | Feature Specs |
|----------|-------------|---------------|
| FR-001 | Mapping of Canonical ID to subcommands | FR-01-01, FR-01-02 |
| FR-002 | Auto-generation of `--key value` flags from JSON Schema | FR-02-01 through FR-02-06 |
| FR-003 | Support for `stdin` piping (`-`) | FR-01-05 |
| FR-004 | TTY-aware approval for sensitive operations | FR-03-01 through FR-03-05 |
| NFR-001 | Execution overhead < 100ms | FR-01-03 (boundary: <100ms startup) |
| NFR-002 | Zero-config: point to apcore Registry | FR-01-03 (default: `./extensions`) |
| — | Discovery (`apcore-cli list`, `apcore-cli describe`) | FR-04-01 through FR-04-05 (maps to SPEC §3.2) |
| — | 4-tier configuration precedence | FR-07-01 through FR-07-03 |
| — | TTY-adaptive output formatting | FR-08-01 through FR-08-04 |
| — | Security (auth, encryption, audit, sandbox) | FR-05-01 through FR-05-04 |
| — | Shell completions and man pages | FR-06-01, FR-06-02 |

---

## Project Dependencies
- `apcore >= 0.29.0` (Core protocol, Registry, Executor, error hierarchy, Config Bus, Execution Pipeline Strategy, ACL — FE-14)
- `apcore-toolkit >= 0.11.0` (Formatting, BindingLoader, DisplayResolver, RegistryWriter, OpenAPIScanner — FE-15)
- `click >= 8.1` (CLI framework — confirmed in [Tech Design v2.0](tech-design.md), ADR-01)
- `jsonschema >= 4.20` (JSON Schema validation and parsing)
- `rich >= 13.0` (Terminal output formatting — tables, syntax highlighting)
- `pyyaml >= 6.0` (Configuration file parsing)
- `keyring >= 24.0` (OS keyring for encrypted config storage)
- `cryptography >= 41.0` (AES-256-GCM encryption)

---

## Implementation Order

1. **Config Resolver** (FE-07): Foundation for all configurable values.
2. **Core Dispatcher** (FE-01): Establishes the `apcore-cli` command structure and registry loading.
3. **Schema Parser** (FE-02): Connects module metadata to the command-line arguments.
4. **Output Formatter** (FE-08): TTY-adaptive rendering used by Discovery and exec.
5. **Discovery** (FE-04): Provides tools for browsing available modules.
6. **Approval Gate** (FE-03): Adds governance and safety for sensitive module execution.
7. **Security Manager** (FE-05): Authentication, encryption, auditing, sandboxing.
8. **Shell Integration** (FE-06): Completion scripts and man pages.
9. **Grouped Commands** (FE-09): Auto-grouping layer over Core Dispatcher (v0.3.0).
10. **Init Command** (FE-10): Scaffolding subcommand (v0.4.0).
11. **Usability Enhancements** (FE-11): Dry-run, system commands, trace, strategy, multi-level grouping, custom commands (v0.6.0).
12. **Module Exposure Filtering** (FE-12): Declarative exposure filter over discovered modules (v0.7.0).
13. **Built-in Command Group** (FE-13): Relocate apcore-cli built-ins under `apcli` group with visibility controls; retire `BUILTIN_COMMANDS` (v0.7.0).

---

## Dependency Graph

```
FE-07 Config Resolver (foundation)
  └── FE-01 Core Dispatcher (depends on FE-07)
        ├── FE-02 Schema Parser (depends on FE-01)
        │     └── FE-03 Approval Gate (depends on FE-02)
        ├── FE-08 Output Formatter (used by FE-01, FE-04)
        │     └── FE-04 Discovery (depends on FE-01, FE-08)
        ├── FE-05 Security Manager (depends on FE-07)
        ├── FE-06 Shell Integration (depends on FE-01, FE-02)
        ├── FE-09 Grouped Commands (depends on FE-01)
        │     ├── FE-12 Exposure Filtering (depends on FE-09, FE-07)
        │     └── FE-13 Built-in Command Group (depends on FE-09, FE-07, retires BUILTIN_COMMANDS)
        ├── FE-10 Init Command (depends on FE-01)
        └── FE-11 Usability Enhancements (depends on FE-01, FE-03, FE-07)
```

---

## MVP Definition
A functional `apcore-cli` command that can load a local `apcore` extensions directory, parse arguments for a simple module, and execute it while handling basic STDIN piping.
