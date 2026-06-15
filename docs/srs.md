---
description: "Software Requirements Specification (SRS-APCORE-CLI-001) defining functional and non-functional requirements for apcore-cli: dispatch, schema, approval, discovery, security, shell, and output."
---

# Software Requirements Specification: apcore-cli

---

## 1. Document Information

| Field | Value |
|-------|-------|
| **Document ID** | SRS-APCORE-CLI-001 |
| **Version** | 0.8 |
| **Author** | Spec Forge |
| **Date** | 2026-05-07 |
| **Status** | Active |

---

## 2. Revision History

| Version | Date | Author | Description |
|---------|------|--------|-------------|
| 0.8 | 2026-05-07 | Spec Forge | Rewrite FR-DISC-004 around the seven-format canonical choice set (`table, json, csv, yaml, jsonl, markdown, skill`); drop stale AC-4 yaml-rejection assertion; add FR-DISC-003 AF-4 documenting toolkit delegation. Issue #20. |
| 0.7 | 2026-04-15 | Spec Forge | Add requirements for FE-10 Init Command (§5.7), FE-11 Usability Enhancements (§5.8), FE-12 Exposure Filtering (§5.9). Update status to Active. |
| 0.1 | 2026-03-14 | Spec Forge | Initial draft |

---

## 3. Introduction

### 3.1 Purpose

This Software Requirements Specification defines the functional and non-functional requirements for `apcore-cli`, the CLI adapter for the apcore ecosystem. It provides a verifiable, traceable baseline for design, implementation, testing, and acceptance of the system.

The intended audience includes developers implementing apcore-cli, QA engineers writing test plans, and stakeholders reviewing feature completeness.

### 3.2 Scope

`apcore-cli` is a Python-based command-line interface that exposes apcore modules as terminal subcommands. The system shall:

- Provide an `exec` command that routes to and invokes apcore modules via the Executor.
- Auto-generate CLI flags from module JSON Schema definitions with zero manual configuration.
- Enforce Human-in-the-Loop (HITL) approval gates for modules annotated as requiring approval.
- Offer `list` and `describe` commands for terminal-native module discovery.
- Integrate with shell ecosystems (completion scripts, man pages).
- Authenticate, encrypt sensitive configuration, log all executions, and sandbox module execution.

The system shall NOT provide remote execution via apcore-a2a (deferred to Phase 4) or replace apcore-mcp (they are complementary adapters).

### 3.3 Definitions, Acronyms, and Abbreviations

| Term | Definition |
|------|-----------|
| **apcore** | The core protocol and framework for building AI-Perceivable modules. Provides Registry, Executor, and the 3-layer metadata model (Discovery, Capabilities, Execution). |
| **HITL** | Human-in-the-Loop. A safety pattern requiring explicit human confirmation before executing sensitive or destructive operations. |
| **Registry** | An apcore component that discovers and indexes module definitions from an extensions directory. Provides `list()`, `get_definition()`, and `get_schema()` APIs. |
| **Module** | An apcore module. A self-describing unit of business logic with metadata (description, input_schema, output_schema, annotations). |
| **Canonical ID** | A dot-separated, lowercase identifier for a module (e.g., `math.add`). Format: `^[a-z][a-z0-9_]*(\.[a-z][a-z0-9_]*)*$`. Maximum 192 characters (see PROTOCOL_SPEC §2.7). |
| **TTY** | Teletypewriter. In this context, refers to an interactive terminal session where `sys.stdin.isatty()` returns `True`. |
| **STDIN** | Standard Input. The default input stream for a process, used for piped JSON input. |
| **JSON Schema** | A vocabulary for annotating and validating JSON documents (RFC draft-bhutton-json-schema-01). Used by apcore modules to define `input_schema` and `output_schema`. |
| **PROTOCOL_SPEC** | The apcore Protocol Specification document defining module structure, error codes, environment variable conventions, and metadata standards. |
| **Executor** | The apcore execution engine that runs modules through a middleware chain (ACL, observability, approval handling). |
| **Click** | A Python package for creating CLI interfaces with composable commands and automatic help generation. |

### 3.4 References

| Ref ID | Document | Location |
|--------|----------|----------|
| REF-01 | Tech Design v2.0: apcore-cli | `docs/tech-design.md` |
| REF-02 | Feature Spec: Core Dispatcher | `docs/features/core-dispatcher.md` |
| REF-03 | Feature Spec: Schema Parser | `docs/features/schema-parser.md` |
| REF-04 | Feature Spec: Approval Gate | `docs/features/approval-gate.md` |
| REF-05 | Feature Spec: Discovery | `docs/features/discovery.md` |
| REF-06 | Feature Spec: Security Manager | `docs/features/security.md` |
| REF-07 | Feature Spec: Shell Integration | `docs/features/shell-integration.md` |
| REF-08 | Feature Spec: Config Resolver | `docs/features/config-resolver.md` |
| REF-09 | Feature Spec: Output Formatter | `docs/features/output-formatter.md` |
| REF-10 | apcore PROTOCOL_SPEC | `apcore/PROTOCOL_SPEC.md` |
| REF-11 | Idea Draft: apcore-cli | `ideas/draft.md` |
| REF-12 | Project Manifest: apcore-cli | `docs/project-apcore-cli.md` |

### 3.5 Overview

Section 4 describes the system context, user characteristics, and constraints. Section 5 enumerates all functional requirements organized by module. Section 6 specifies non-functional requirements with measurable targets. Section 7 defines data requirements. Section 8 covers external interface requirements. Section 9 documents requirement sources and traceability. Section 10 provides the appendix.

---

## 4. Overall Description

### 4.1 Product Perspective

`apcore-cli` is one adapter in the apcore ecosystem, alongside `apcore-mcp` (Model Context Protocol adapter) and `apcore-a2a` (Agent-to-Agent adapter). It occupies the CLI channel, translating terminal interactions into apcore module invocations.

```
                    +------------------+
                    |   Developer      |
                    |   (Terminal)     |
                    +--------+---------+
                             |
                             v
+------------------+   +----+-------------+   +------------------+
|   AI Agent       +-->|   apcore-cli     +-->|   apcore Module  |
|   (Shell Exec)   |   |   (The Adapter)  |   |   (Business      |
+------------------+   +----+-------------+   |    Logic)        |
                             |                 +------------------+
                             v
                    +--------+---------+
                    |   apcore Registry |
                    |   (Local/Remote)  |
                    +------------------+
```

apcore-cli is a standalone CLI process. It reads module metadata from the Registry, maps JSON Schema to CLI flags, enforces approval gates, and delegates execution to the apcore Executor.

### 4.2 Product Functions

The system provides eight feature groups:

1. **Core Dispatcher (FE-01, P0)**: CLI entry point (`apcore-cli`), extensions directory loading, module routing and execution via Executor, STDIN JSON piping.
2. **Schema Parser (FE-02, P0)**: Zero-config generation of CLI flags from module JSON Schema `input_schema`, including type mapping, boolean flag pairs, enum choices, required enforcement, help text extraction, and `$ref` resolution.
3. **Approval Gate (FE-03, P1)**: TTY-aware HITL approval prompting for modules with `annotations.requires_approval`, with bypass mechanisms (`--yes`, `APCORE_CLI_AUTO_APPROVE=1`) and timeout enforcement.
4. **Discovery (FE-04, P1)**: `list` and `describe` subcommands for browsing available modules, with tag filtering, format selection (table/json), and TTY-adaptive defaults.
5. **Security Manager (FE-05, P1/P2)**: API key authentication, encrypted config storage (keyring + AES-256-GCM), append-only audit logging, and subprocess sandboxing.
6. **Shell Integration (FE-06, P2)**: Shell completion scripts (bash/zsh/fish) and man page generation.
7. **Config Resolver (FE-07, P0)**: 4-tier configuration precedence (CLI flag > Environment variable > Config file > Default).
8. **Output Formatter (FE-08, P1)**: TTY-adaptive output formatting (JSON for pipes, rich tables for terminals).

### 4.3 User Characteristics

| User Type | Proficiency | Interaction Mode | Key Needs |
|-----------|-------------|------------------|-----------|
| **Developer** | High technical proficiency. Familiar with CLI tools, Unix pipes, JSON. | Interactive terminal (TTY). Types commands, reads formatted output. | Descriptive help text, readable error messages, tab completion, approval prompts. |
| **AI Agent** | Programmatic. Invokes commands via shell exec (`subprocess`, `os.system`). | Non-interactive (non-TTY). Parses stdout, interprets exit codes. | Deterministic exit codes, structured JSON output, no interactive prompts (requires `--yes` or auto-approve). |

### 4.4 Constraints

1. **Language Constraint**: The system shall be implemented in Python >= 3.11, aligned with apcore >= 0.17.1 requirements.
2. **Framework Constraint**: The system shall use the `click` library for CLI command construction (Tech Design ADR-01).
3. **Naming Constraint**: The default CLI program name (as shown in `--help` and `--version` output) shall be derived from the invoking entry-point script name (`argv[0]` basename). It shall NOT be hardcoded to `apcore-cli`. Downstream projects that install their own entry-point script shall receive their project name in all output automatically. An explicit `prog_name` parameter shall override this. Fallback when `argv[0]` is unavailable: `apcore-cli`. See FR-DISP-006 and Tech Design ADR-02.
4. **Execution Constraint**: The system shall use apcore's `Executor` for module invocation, preserving the full execution pipeline (Tech Design ADR-03). The system uses the standard 11-step pipeline by default; a pre-configured Executor with a custom `ExecutionStrategy` may be provided via `create_cli(executor=...)`.
5. **Error Code Constraint**: All exit codes shall align with PROTOCOL_SPEC section 8.
6. **Environment Variable Constraint**: All environment variables shall follow the `APCORE_{SECTION}_{KEY}` naming convention (PROTOCOL_SPEC section 9.2).

### 4.5 Assumptions and Dependencies

**Assumptions:**

1. The apcore Registry is available on the local filesystem at a known path.
2. Modules conform to the apcore 3-layer metadata model and provide valid JSON Schema `input_schema`.
3. The host system has Python >= 3.11 installed with pip or poetry available.
4. Terminal emulators used by developers correctly report TTY status via `sys.stdin.isatty()`.

**Dependencies:**

| Dependency | Version | Purpose |
|-----------|---------|---------|
| `apcore` | >= 0.17.1 | Core protocol, Registry, Executor, error hierarchy, Config Bus, Execution Pipeline Strategy |
| `click` | >= 8.1 | CLI framework for command construction |
| `jsonschema` | >= 4.20 | JSON Schema validation and parsing |
| `rich` | >= 13.0 | Terminal output formatting (tables, syntax highlighting) |

---

## 5. Functional Requirements

### 5.1 Core Dispatcher (DISP)

#### FR-DISP-001: Base Command Entry Point

| Field | Value |
|-------|-------|
| **ID** | FR-DISP-001 |
| **Title** | Base Command Entry Point |
| **Priority** | P0 |
| **Priority Rationale** | Foundation requirement. All other features depend on the base command existing. Without this, no CLI interaction is possible. |
| **Source** | Tech Design v2.0 ADR-02; Feature Spec FE-01 FR-01-01 |

**Description:** The system shall provide a base command `apcore-cli` that serves as the root entry point for all CLI interactions. The base command shall display a help message listing all available subcommands when invoked with `--help` or when invoked with no arguments.

**Actors:** Developer, AI Agent

**Preconditions:**
- `apcore-cli` is installed and available on the system PATH.

**Main Flow:**
1. The user invokes `apcore-cli` with no arguments or with `--help`.
2. The system shall load the apcore Registry from the configured extensions directory.
3. The system shall enumerate all registered modules via `Registry.list()`.
4. The system shall display a help message containing:
   - The program name (`apcore-cli`).
   - A brief description of the tool.
   - A single built-in group entry `apcli` (when visible per FE-13 visibility resolution — see [features/builtin-group.md §4.1](features/builtin-group.md)). The 13 built-in subcommands (`list`, `describe`, `exec`, `validate`, `init`, `health`, `usage`, `enable`, `disable`, `reload`, `config`, `completion`, `describe-pipeline`) are reachable as `<cli> apcli <sub>`, not at root.
   - A list of available module Canonical IDs (as dynamically registered top-level subcommands).
5. The system shall exit with code 0.

**Alternative Flows:**

- **AF-1: Registry Load Failure.** If the extensions directory does not exist or is inaccessible, the system shall write an error message to stderr suggesting the user set `APCORE_EXTENSIONS_ROOT` or verify the default path (`./extensions`), and exit with code 47 (`CONFIG_NOT_FOUND`).
- **AF-2: Empty Registry.** If the Registry contains zero modules, the system shall display the help message with an informational note "No modules found in registry" and exit with code 0.
- **AF-3: Version Flag.** If the user invokes `apcore-cli --version`, the system shall print the installed version string in the format `apcore-cli, version X.Y.Z` and exit with code 0.

**Postconditions:**
- Help text is written to stdout.
- Exit code is 0 (success) or 47 (configuration error).

**Acceptance Criteria:**

- **AC-1:** Given `apcore-cli` is installed and a valid extensions directory exists, when the user runs `apcore-cli --help`, then the help output shall contain `apcli` (when visible) plus any registered module IDs/groups, and the process shall exit with code 0.
- **AC-2:** Given the configured extensions directory does not exist, when the user runs `apcore-cli --help`, then stderr shall contain a message including the text "extensions" and a suggestion to set `APCORE_EXTENSIONS_ROOT`, and the process shall exit with code 47.
- **AC-3:** Given the Registry contains modules `math.add` and `text.summarize`, when the user runs `apcore-cli --help`, then the output shall list both `math.add` and `text.summarize`.
- **AC-4:** Given an embedded host invokes `create_cli(..., apcli=False)`, when the user runs `<cli> --help`, then the help output shall NOT list `apcli` (group hidden per FE-13). Cross-reference [features/builtin-group.md §4.1](features/builtin-group.md).

---

#### FR-DISP-002: Module Execution

| Field | Value |
|-------|-------|
| **ID** | FR-DISP-002 |
| **Title** | Module Execution via Direct Subcommand |
| **Priority** | P0 |
| **Priority Rationale** | Core value proposition. Without execution, the CLI has no utility. |
| **Source** | Tech Design v2.0 ADR-03; Feature Spec FE-01 FR-01-02; ideas/draft.md FR-001 |

**Description:** The system shall expose each registered module as a direct top-level subcommand (e.g., `apcore-cli math.add`). When invoked, the system accepts a Canonical Module ID as the command name, resolves the module from the Registry, validates input arguments against the module's `input_schema`, and invokes the module via the apcore `Executor.call()` method. The system shall write the module's result to stdout and exit with code 0 on success.

**Actors:** Developer, AI Agent

**Preconditions:**
- The Registry is loaded and contains at least one module.
- The module's `input_schema` is a valid JSON Schema document.

**Main Flow:**
1. The user invokes `apcore-cli <module_id> [--flags...]`.
2. The system shall validate that `<module_id>` conforms to the Canonical ID format: `^[a-z][a-z0-9_]*(\.[a-z][a-z0-9_]*)*$`, with a maximum length of 192 characters (PROTOCOL_SPEC §2.7).
3. The system shall look up `<module_id>` in the Registry via `Registry.get_definition()`.
4. The system shall parse the module's `input_schema` and generate CLI flags (delegated to Schema Parser, see section 5.2).
5. The system shall collect input values from CLI flags and/or STDIN (see FR-DISP-004).
6. The system shall validate the collected input against `input_schema` using `jsonschema.validate()`.
7. The system shall check `annotations.requires_approval` and invoke the Approval Gate if required (see section 5.3).
8. The system shall invoke the module via `Executor.call(module_id, validated_input)`.
9. The system shall write the module result to stdout.
10. The system shall exit with code 0.

**Alternative Flows:**

- **AF-1: Module Not Found.** If the Registry does not contain `<module_id>`, the system shall write to stderr: `Error: Module '<module_id>' not found in registry.` and exit with code 44 (`MODULE_NOT_FOUND`).
- **AF-2: Invalid Module ID Format.** If `<module_id>` does not match the Canonical ID regex or exceeds 192 characters, the system shall write to stderr: `Error: Invalid module ID format: '<module_id>'.` and exit with code 2 (`GENERAL_INVALID_INPUT`).
- **AF-3: Schema Validation Failure.** If the provided input fails JSON Schema validation, the system shall write to stderr a validation error message identifying the failing property and constraint, and exit with code 45 (`SCHEMA_VALIDATION_ERROR`).
- **AF-4: Module Execution Error.** If the Executor returns an error from the module, the system shall write the error message to stderr and exit with code 1 (`MODULE_EXECUTE_ERROR`).
- **AF-5: Module Disabled.** If the module exists but is marked as disabled, the system shall write to stderr: `Error: Module '<module_id>' is disabled.` and exit with code 44 (`MODULE_DISABLED`).
- **AF-6: User Interruption.** If the user sends SIGINT (Ctrl+C) during execution, the system shall write to stderr: `Execution cancelled.` and exit with code 130 (`EXECUTION_CANCELLED`).
- **AF-7: Module Load Error.** If the module is registered but fails to load (import error, missing entry_point), the system shall write to stderr a diagnostic message and exit with code 44 (`MODULE_LOAD_ERROR`).
- **AF-8: ACL Denied.** If the Executor's ACL middleware rejects the invocation, the system shall write to stderr: `Error: Permission denied for module '<module_id>'.` and exit with code 77 (`ACL_DENIED`).

**Postconditions:**
- On success: module result is written to stdout; exit code is 0.
- On failure: error message is written to stderr; exit code corresponds to the error category per PROTOCOL_SPEC section 8.

**Acceptance Criteria:**

- **AC-1:** Given a registered module `math.add` with `input_schema` requiring `a` (integer) and `b` (integer), when the user runs `apcore-cli math.add --a 5 --b 10`, then stdout shall contain the module result and the process shall exit with code 0.
- **AC-2:** Given no module `non.existent` in the Registry, when the user runs `apcore-cli non.existent`, then stderr shall contain "not found" and the process shall exit with code 44.
- **AC-3:** Given module `math.add` requires integer `a`, when the user runs `apcore-cli math.add --a "hello"`, then stderr shall contain a schema validation error and the process shall exit with code 45.
- **AC-4:** Given a module ID `INVALID!ID`, when the user runs `apcore-cli "INVALID!ID"`, then stderr shall contain "Invalid module ID format" and the process shall exit with code 2.

---

#### FR-DISP-003: Extensions Directory Loading

| Field | Value |
|-------|-------|
| **ID** | FR-DISP-003 |
| **Title** | Extensions Directory Loading |
| **Priority** | P0 |
| **Priority Rationale** | Required for module discovery. Without a loaded Registry, no commands can be generated or executed. |
| **Source** | Feature Spec FE-01 FR-01-03; ideas/draft.md NFR-002 |

**Description:** The system shall load the apcore extensions directory from a configurable path, instantiate a Registry via `Registry(path)`, and invoke `Registry.discover()` to index all available modules. The path shall be resolved using the configuration precedence defined in FR-DISP-005.

**Actors:** System (internal)

**Preconditions:**
- The system is starting up or a command requiring the Registry has been invoked.

**Main Flow:**
1. The system shall resolve the extensions directory path using the following precedence (highest to lowest): CLI flag `--extensions-dir`, environment variable `APCORE_EXTENSIONS_ROOT`, configuration file `apcore.yaml` key `extensions.root`, default value `./extensions`.
2. The system shall verify that the resolved path exists and is a readable directory.
3. The system shall instantiate `Registry(resolved_path)`.
4. The system shall invoke `Registry.discover()` to scan and index modules.
5. The system shall log at DEBUG level: `"Loading extensions from {resolved_path}"`.
6. The system shall log at INFO level: `"Initialized apcore-cli with {N} modules."` where N is the count of discovered modules.

**Alternative Flows:**

- **AF-1: Path Does Not Exist.** If the resolved path does not exist as a directory, the system shall write to stderr: `Error: Extensions directory not found: '{resolved_path}'. Set APCORE_EXTENSIONS_ROOT or verify the path.` and exit with code 47 (`CONFIG_NOT_FOUND`).
- **AF-2: Path Not Readable.** If the resolved path exists but is not readable (permission denied), the system shall write to stderr: `Error: Cannot read extensions directory: '{resolved_path}'. Check file permissions.` and exit with code 47 (`CONFIG_NOT_FOUND`).
- **AF-3: Corrupt Module in Directory.** If one or more modules in the directory fail to load (malformed metadata, import errors), the system shall log a WARNING for each failed module, skip it, and continue loading remaining modules. The system shall not exit with an error unless zero modules load successfully.
- **AF-4: Configuration File Malformed.** If `apcore.yaml` exists but is not valid YAML or lacks the `extensions.root` key, the system shall log a WARNING and fall through to the default path.

**Postconditions:**
- The Registry is instantiated and populated with discovered modules.
- The count of loaded modules is logged.

**Acceptance Criteria:**

- **AC-1:** Given `APCORE_EXTENSIONS_ROOT` is set to `/tmp/my-extensions` containing 3 modules, when the user runs any `apcore-cli` command, then the system shall load modules from `/tmp/my-extensions`.
- **AC-2:** Given `--extensions-dir /custom/path` is provided and `APCORE_EXTENSIONS_ROOT` is also set, when the user runs `apcore-cli --extensions-dir /custom/path list`, then the system shall load from `/custom/path` (CLI flag takes precedence).
- **AC-3:** Given no CLI flag, no env var, no config file, when the user runs `apcore-cli list`, then the system shall attempt to load from `./extensions`.
- **AC-4:** Given the extensions directory contains one valid module and one corrupt module, when the system loads, then the valid module shall be available and a warning shall be logged for the corrupt module.

---

#### FR-DISP-004: STDIN JSON Input

| Field | Value |
|-------|-------|
| **ID** | FR-DISP-004 |
| **Title** | STDIN JSON Input |
| **Priority** | P0 |
| **Priority Rationale** | Enables Unix pipe composability. Critical for AI Agent usage where structured input is piped from upstream processes. |
| **Source** | Feature Spec FE-01 FR-01-05; ideas/draft.md FR-003 |

**Description:** The system shall accept JSON input from STDIN when the user specifies `--input -` on the `exec` subcommand. The system shall read the entire STDIN stream, parse it as JSON, and merge the resulting key-value pairs with any CLI flags provided. CLI flags shall take precedence over STDIN values for the same key.

**Actors:** Developer, AI Agent

**Preconditions:**
- The `exec` subcommand is being invoked.
- STDIN contains a valid JSON object (not an array or primitive).

**Main Flow:**
1. The user invokes `apcore-cli exec <module_id> --input -` with JSON piped to STDIN.
2. The system shall read the STDIN stream up to the buffer limit (10 MB by default).
3. The system shall parse the stream content as a JSON object.
4. The system shall merge the parsed JSON keys with any CLI flag values, where CLI flags take precedence for duplicate keys.
5. The system shall pass the merged input to schema validation and subsequent execution.

**Alternative Flows:**

- **AF-1: STDIN Exceeds Buffer Limit.** If STDIN data exceeds 10 MB and `--large-input` is not specified, the system shall write to stderr: `Error: STDIN input exceeds 10MB limit. Use --large-input to override.` and exit with code 2 (`GENERAL_INVALID_INPUT`).
- **AF-2: Invalid JSON.** If the STDIN content is not valid JSON, the system shall write to stderr: `Error: STDIN does not contain valid JSON: {parse_error_detail}.` and exit with code 2 (`GENERAL_INVALID_INPUT`).
- **AF-3: JSON Not an Object.** If the STDIN content parses as a JSON array, string, number, boolean, or null instead of an object, the system shall write to stderr: `Error: STDIN JSON must be an object, got {actual_type}.` and exit with code 2 (`GENERAL_INVALID_INPUT`).
- **AF-4: Empty STDIN.** If STDIN is empty (0 bytes) when `--input -` is specified, the system shall treat the input as an empty object `{}` and proceed with CLI flag values only.
- **AF-5: STDIN Without --input Flag.** If STDIN has piped data but `--input -` is not specified, the system shall ignore STDIN and use only CLI flag values.
- **AF-6: Large Input Override.** If `--large-input` is specified, the system shall read STDIN without the 10 MB limit.

**Postconditions:**
- STDIN JSON is parsed, merged with CLI flags, and passed to schema validation.

**Acceptance Criteria:**

- **AC-1:** Given module `math.add` expects `a` and `b`, when the user runs `echo '{"a":5,"b":10}' | apcore-cli math.add --input -`, then the module shall receive `{a:5, b:10}` and execute successfully.
- **AC-2:** Given `echo '{"a":5}' | apcore-cli math.add --input - --b 20`, when executed, then the module shall receive `{a:5, b:20}` (CLI flag `--b` overrides nothing; STDIN provides `a`).
- **AC-3:** Given `echo '{"a":5,"b":10}' | apcore-cli math.add --input - --a 99`, when executed, then the module shall receive `{a:99, b:10}` (CLI flag `--a 99` takes precedence over STDIN `a:5`).
- **AC-4:** Given STDIN contains 15 MB of JSON and `--large-input` is not specified, when executed, then stderr shall contain "exceeds 10MB limit" and the process shall exit with code 2.

---

#### FR-DISP-005: Configuration Precedence

| Field | Value |
|-------|-------|
| **ID** | FR-DISP-005 |
| **Title** | Configuration Precedence |
| **Priority** | P0 |
| **Priority Rationale** | Foundational behavior required by PROTOCOL_SPEC section 9.2. Governs how all configurable values are resolved. |
| **Source** | Tech Design v2.0 section 8.3; PROTOCOL_SPEC section 9.2 |

**Description:** The system shall resolve all configurable values using a four-tier precedence hierarchy. For any given configuration key, the system shall use the value from the highest-priority source that provides a value.

**Actors:** System (internal)

**Preconditions:**
- The system is resolving a configuration value.

**Main Flow:**
1. The system shall check for a CLI flag value (e.g., `--extensions-dir`, `--yes`, `--log-level`). If present, use it.
2. If no CLI flag, the system shall check for an environment variable (e.g., `APCORE_EXTENSIONS_ROOT`, `APCORE_CLI_AUTO_APPROVE`, `APCORE_CLI_LOGGING_LEVEL`, `APCORE_LOGGING_LEVEL`). If set and non-empty, use it. For logging level, `APCORE_CLI_LOGGING_LEVEL` takes priority over `APCORE_LOGGING_LEVEL`.
3. If no environment variable, the system shall check for a value in the configuration file `apcore.yaml`. If the file exists, is valid YAML, and contains the relevant key, use it.
4. If no configuration file value, the system shall use the built-in default value.

**Alternative Flows:**

- **AF-1: Config File Not Found.** If `apcore.yaml` is not present at the expected location, the system shall skip tier 3 silently and fall through to the default.
- **AF-2: Config File Malformed.** If `apcore.yaml` exists but contains invalid YAML, the system shall log a WARNING: `"Configuration file 'apcore.yaml' is malformed, using defaults."` and fall through to the default.

**Postconditions:**
- The configuration value is resolved from the highest-priority available source.

**Acceptance Criteria:**

- **AC-1:** Given `--extensions-dir /cli-path` is provided, `APCORE_EXTENSIONS_ROOT=/env-path` is set, and `apcore.yaml` contains `extensions.root: /config-path`, when the system resolves the extensions directory, then it shall use `/cli-path`.
- **AC-2:** Given no CLI flag, `APCORE_EXTENSIONS_ROOT=/env-path` is set, and `apcore.yaml` contains `extensions.root: /config-path`, when the system resolves the extensions directory, then it shall use `/env-path`.
- **AC-3:** Given no CLI flag, no env var, and `apcore.yaml` contains `extensions.root: /config-path`, when the system resolves the extensions directory, then it shall use `/config-path`.
- **AC-4:** Given no CLI flag, no env var, and no config file, when the system resolves the extensions directory, then it shall use `./extensions`.

---

#### FR-DISP-006: CLI Program Name Customization

| Field | Value |
|-------|-------|
| **ID** | FR-DISP-006 |
| **Title** | CLI Program Name Customization |
| **Priority** | P1 |
| **Priority Rationale** | Required for library-mode deployments. Downstream projects that redistribute `apcore-cli` under their own brand receive a broken user experience if the program name is hardcoded. |
| **Source** | Tech Design ADR-02 (revised); Feature Spec FE-01 FR-01-06 |

**Description:** The system shall resolve the CLI program name (used in `--help` output, `--version` output, and error messages) dynamically rather than using a hardcoded value. The resolution shall follow a 2-tier precedence: an explicit parameter overrides automatic detection; otherwise the basename of `argv[0]` is used. This enables downstream projects to publish the CLI under their own project name with zero source-code changes.

**Actors:** Developer, AI Agent, Downstream Library User

**Preconditions:**
- The CLI adapter is installed either as a standalone package or as a dependency of a downstream project.

**Main Flow:**
1. At startup, the system shall resolve `prog_name` using the following precedence (highest to lowest):
   - **Tier 1 — Explicit parameter.** If `prog_name` is passed as an argument to `create_cli(prog_name=...)` or `main(prog_name=...)`, use it verbatim.
   - **Tier 2 — `argv[0]` basename.** Compute `os.path.basename(sys.argv[0])` and use it as `prog_name`.
2. The resolved `prog_name` shall be used as:
   - The Click group `name` parameter (controls the command name in `--help` output).
   - The `prog_name` parameter of `click.version_option()` (controls the version string prefix).
3. The version string format shall be: `{prog_name}, version {X.Y.Z}`.

**Alternative Flows:**

- **AF-1: `argv[0]` empty or unavailable.** If `os.path.basename(sys.argv[0])` returns an empty string or raises an exception, the system shall fall back to the string `apcore-cli`.
- **AF-2: Explicit parameter is `None`.** If `prog_name=None` is passed explicitly, the system shall treat it as "not provided" and proceed to Tier 2 (`argv[0]` basename).

**Postconditions:**
- All user-visible output (help, version, error messages referencing the command name) uses the resolved `prog_name`.

**Acceptance Criteria:**

- **AC-1:** Given a downstream project installs entry-point `myproject = "apcore_cli.__main__:main"`, when the user runs `myproject --help`, then the output shall contain `myproject` as the command name, not `apcore-cli`.
- **AC-2:** Given a downstream project installs entry-point `myproject = "apcore_cli.__main__:main"`, when the user runs `myproject --version`, then the output shall be `myproject, version X.Y.Z`.
- **AC-3:** Given `create_cli(prog_name="custom-name")` is called programmatically, when `--help` is invoked, then the output shall contain `custom-name` regardless of `argv[0]`.
- **AC-4:** Given the default `apcore-cli` entry point is used, when `apcore-cli --version` is run, then the output shall be `apcore-cli, version X.Y.Z`.
- **AC-5:** Given `prog_name=None` is passed to `create_cli()`, when `--help` is invoked, then the output shall contain the `argv[0]` basename (Tier 2 resolution applies).

---

#### FR-DISP-007: Verbose Help Mode

| Field | Value |
|-------|-------|
| **ID** | FR-DISP-007 |
| **Title** | All-Options Help Mode |
| **Priority** | P2 |
| **Priority Rationale** | UX improvement. Built-in apcore options are noise for most users; AI agents and power users can opt in via `--all-options`. |
| **Source** | Feature Spec FE-01; v0.4.0 |

**Description:** The system shall hide four built-in options (`--input`, `--yes`, `--large-input`, `--format`) from `--help` output by default. When the global `--all-options` flag is passed alongside `--help`, the system shall display the full option list including these built-in options. The `--sandbox` option is always hidden (not yet implemented) and is not affected by `--all-options`. The `--all-options` flag shall be pre-parsed from `argv` before the CLI framework processes arguments, since help rendering occurs during parsing. The flag name `all_options` shall be added to the reserved property names set to prevent schema collisions.

**Actors:** Developer, AI Agent

**Preconditions:**
- The CLI adapter is installed and a module command exists.

**Main Flow:**
1. At startup, the system shall scan raw `argv` for `--all-options` before the CLI framework parses arguments.
2. If `--all-options` is present, the system shall set a global flag indicating all-options help mode.
3. When building module commands via `build_module_command()`, four built-in options (`--input`, `--yes`, `--large-input`, `--format`) shall be marked as hidden when all-options mode is off, and visible when all-options mode is on. The `--sandbox` option shall remain always hidden until implemented.
4. The `--all-options` flag shall be registered as a global CLI option with help text: "Show all options in help output (including built-in options)."

**Alternative Flows:**
- **AF-1: `--all-options` without `--help`.** The flag is accepted but has no visible effect on command execution.
- **AF-2: Schema property named `all_options`.** The system shall reject it with exit code 48 (`EXIT_SCHEMA_CIRCULAR_REF` — the shared reserved-name / flag-collision slot), as `all_options` is a reserved flag name.

**Postconditions:**
- Default `--help` output shows only schema-derived options.
- `--help --all-options` output shows all options (schema-derived + built-in).

**Acceptance Criteria:**
- **AC-1:** Given a module with property `name`, when the user runs `apcore-cli module --help`, then `--input`, `--yes`, `--large-input`, `--format`, and `--sandbox` shall NOT appear in the output.
- **AC-2:** Given the same module, when the user runs `apcore-cli module --help --all-options`, then `--input`, `--yes`, `--large-input`, and `--format` SHALL appear in the output. `--sandbox` shall remain hidden (not yet implemented).
- **AC-3:** Given a module with a schema property named `all_options`, when the system builds the command, then it shall exit with code 48 and a collision error message.

---

#### FR-DISP-008: Pre-Populated Registry Support

| Field | Value |
|-------|-------|
| **ID** | FR-DISP-008 |
| **Title** | Pre-Populated Registry Support |
| **Priority** | P1 |
| **Priority Rationale** | Required for framework embedding. Without this, frameworks that register modules at runtime cannot use apcore-cli — they must hand-write CLI commands or maintain a parallel extensions directory. |
| **Source** | Feature Spec FE-01 FR-01-08; v0.5.1 |

**Description:** The `create_cli()` factory function shall accept optional `registry` and `executor` parameters. When a pre-populated `Registry` instance is provided, the system shall skip filesystem discovery entirely (no extensions directory resolution, no directory validation, no `Registry.discover()` call). If `executor` is omitted but `registry` is provided, the system shall auto-build an `Executor` from the given registry. Passing `executor` without `registry` is a validation error.

**Actors:** Framework Developer

**Preconditions:**
- The caller has a populated `Registry` instance with modules already registered.

**Main Flow:**
1. The caller invokes `create_cli(registry=my_registry, prog_name="myapp")`.
2. The system validates that `executor` is not provided without `registry`.
3. The system skips extensions directory resolution and filesystem discovery.
4. If `executor` is None, the system instantiates `Executor(registry)`.
5. The system proceeds to build the CLI group using the provided registry and executor.

**Alternative Flows:**
- **AF-1: `executor` without `registry`.** The system raises `ValueError("executor requires registry — pass both or neither")`.
- **AF-2: Both `registry` and `extensions_dir` provided.** `registry` takes precedence; `extensions_dir` is ignored.

**Postconditions:**
- CLI group is created with module commands derived from the pre-populated registry.
- No filesystem access occurred for module discovery.

**Acceptance Criteria:**
- **AC-1:** Given a pre-populated Registry with N modules, when `create_cli(registry=reg)` is called, then the CLI group shall contain commands for all N modules without requiring an extensions directory.
- **AC-2:** Given `create_cli(registry=reg)` is called without an extensions directory on disk, then the system shall NOT exit with code 47.
- **AC-3:** Given `create_cli(executor=exec)` is called without `registry`, then the system shall raise `ValueError`.

**Cross-language equivalents:**

| Language | API |
|----------|-----|
| Python | `create_cli(registry=reg, executor=exec)` |
| TypeScript | `createCli({ registry, executor, progName })` via `CreateCliOptions` |
| Rust | `CliConfig { registry: Some(provider), executor: Some(exec), .. }` |

---

#### FR-DISP-009: Built-in Command Group (`apcli`)

| Field | Value |
|-------|-------|
| **ID** | FR-DISP-009 |
| **Title** | Built-in Command Group (`apcli`) |
| **Priority** | P0 |
| **Priority Rationale** | Required for every branded/embedded CLI. Without this, built-in commands pollute the root namespace, colliding with business-command verbs (`list`, `init`, `describe`) and exposing developer-only tooling to end users. Also required for the retirement of the brittle `BUILTIN_COMMANDS` collision-check mechanism. |
| **Source** | Feature Spec FE-13; v0.7.0 |

**Description:** The system shall relocate all apcore-cli-provided commands (`list`, `describe`, `exec`, `init`, `validate`, `health`, `usage`, `enable`, `disable`, `reload`, `config`, `completion`, `describe-pipeline`) under a single reserved group named `apcli`. The root level shall retain only universally recognized meta-commands and flags (`help`, `--help`, `--version`, `--all-options`, `--man`, `--log-level`), plus user business modules/groups. The system shall accept an `apcli` configuration (via `CliConfig` parameter or `apcore.yaml` key) that controls group-level visibility (`all`/`none`/`include`/`exclude`) and supports a `disable_env` opt-out to sever the `APCORE_CLI_APCLI` environment-variable override.

**Actors:** Framework Developer, End User

**Preconditions:**
- The system is being invoked via the apcore-cli library (embedded or standalone).

**Main Flow:**
1. During `create_cli()`, the system constructs an `ApcliGroup` configuration, resolving tier precedence: `CliConfig.apcli` > `APCORE_CLI_APCLI` env var (when not disabled) > `apcore.yaml` `apcli:` key > auto-detected default.
2. The system creates a Click/Commander group named `apcli` under the root.
3. The system registers subcommands under `apcli` per the visibility mode: under `all`/`none` every subcommand registers; under `include`/`exclude` only the filtered subset registers (plus `exec`, always).
4. The system marks the `apcli` group `hidden=True` when `is_group_visible()` returns False.
5. The system registers `--extensions-dir`, `--commands-dir`, `--binding` root flags only in standalone mode (when `registry` is not injected).
6. The system rejects business modules whose CLI group name, auto-grouped prefix, or top-level command name equals `apcli` with exit code 2 (reserved name).

**Alternative Flows:**
- **AF-1: `apcli: false` + `APCORE_CLI_APCLI=show`.** Env var override succeeds (Tier 2 > Tier 3) — group becomes visible.
- **AF-2: `apcli: {disable_env: true}` + env var set.** Env var ignored; visibility from yaml/default only.
- **AF-3: `create_cli(apcli=False)` + env var set.** CliConfig wins (Tier 1 > Tier 2) — env var ignored regardless of `disable_env`.
- **AF-4: `mode: none`, user runs `<cli> apcli list`.** Succeeds — group-level hide is an output filter, not a surface reduction.
- **AF-5: `mode: include, include: []`, user runs `<cli> apcli exec X`.** Succeeds — `exec` is always registered per FE-12 guarantee.
- **AF-6: Embedded mode, user passes `--extensions-dir`.** Click-standard "No such option" error (flag not registered in embedded mode).

**Postconditions:**
- Root `--help` output is clean of developer-tooling commands unless the integrator explicitly opts in via `apcli: true`.
- All apcore-cli built-in commands remain reachable via the `apcli` group path for debugging, CI, and scripting.
- Business modules cannot collide with built-in command names at the root level.

**Acceptance Criteria:**
- **AC-1:** Given `create_cli(registry=reg)` is called with no `apcli` config, when the user runs `<cli> --help`, then the `apcli` group shall NOT appear.
- **AC-2:** Given `<cli> apcli list` is invoked with `apcli: false`, then the command shall execute successfully.
- **AC-3:** Given `apcli: {mode: include, include: [list]}`, when the user runs `<cli> apcli init`, then the system shall respond with "No such command 'init'".
- **AC-4:** Given `apcli: {mode: include, include: [list]}`, when the user runs `<cli> apcli exec some.module`, then the command shall execute successfully (FE-12 preservation).
- **AC-5:** Given `apcli: {mode: none, disable_env: true}` and `APCORE_CLI_APCLI=show`, when the user runs `<cli> --help`, then the `apcli` group shall remain hidden.
- **AC-6:** Given `create_cli(apcli=False)` and `APCORE_CLI_APCLI=show`, when the user runs `<cli> --help`, then the `apcli` group shall remain hidden (Tier 1 precedence).
- **AC-7:** Given a business module whose CLI top-level name or group name is `apcli`, when the CLI is built, then the system shall exit with code 2 and a reserved-name error.
- **AC-8:** Given `registry` is injected (embedded mode), when `<cli> --help --all-options` is run, then `--extensions-dir`, `--commands-dir`, `--binding` shall NOT appear.

**Cross-language equivalents:**

| Language | API |
|----------|-----|
| Python | `create_cli(apcli=False)` or `create_cli(apcli={"mode": "include", "include": ["list"], "disable_env": True})` |
| TypeScript | `createCli({ apcli: false })` or `createCli({ apcli: { mode: "include", include: ["list"], disableEnv: true } })` |
| Rust | `CliConfig { apcli: Some(ApcliConfig { mode: ApcliMode::None, disable_env: true }), .. }` |
| Go | `CliConfig{Apcli: &ApcliConfig{Mode: ApcliModeNone, DisableEnv: true}}` (Go SDK planned for a later minor release) |

---

### 5.2 Schema Parser (SCHEMA)

#### FR-SCHEMA-001: Property-to-Flag Mapping

| Field | Value |
|-------|-------|
| **ID** | FR-SCHEMA-001 |
| **Title** | Property-to-Flag Mapping |
| **Priority** | P0 |
| **Priority Rationale** | Core schema-to-CLI conversion. Without this, modules cannot receive input from the command line. |
| **Source** | Feature Spec FE-02 FR-02-01; ideas/draft.md FR-002 |

**Description:** The system shall map each property in a module's JSON Schema `input_schema.properties` to a CLI flag in the format `--property-name value`. Property names containing underscores shall be converted to kebab-case for the flag name (e.g., `input_file` becomes `--input-file`). The Click option type shall be determined by the JSON Schema `type` field.

**Actors:** System (internal)

**Preconditions:**
- A module has been resolved from the Registry.
- The module's `input_schema` is a valid JSON Schema object with a `properties` field.

**Main Flow:**
1. The system shall iterate over each key-value pair in `input_schema.properties`.
2. For each property, the system shall create a `click.Option` with:
   - Flag name: `--{property_name}` with underscores replaced by hyphens.
   - Type mapping:
     - `"string"` maps to `click.STRING`.
     - `"integer"` maps to `click.INT`.
     - `"number"` maps to `click.FLOAT`.
     - `"boolean"` maps to a flag pair (see FR-SCHEMA-002).
     - `"object"` maps to `click.STRING` (expects JSON string input).
     - `"array"` maps to `click.STRING` (expects JSON string input).
   - Default value: the JSON Schema `default` value if specified, otherwise `None`.
3. The system shall register all generated options on the `exec` Click command for the target module.

**Alternative Flows:**

- **AF-1: Unknown Type.** If a property has a `type` value not in the set `{string, integer, number, boolean, object, array}`, the system shall default to `click.STRING` and log a WARNING: `"Unknown schema type '{type}' for property '{name}', defaulting to string."`.
- **AF-2: No Type Specified.** If a property lacks a `type` field entirely, the system shall default to `click.STRING` and log a WARNING.
- **AF-3: Property Name Collision.** If converting underscores to hyphens causes two properties to map to the same flag name, the system shall write to stderr an error identifying the collision and exit with code 48 (`SCHEMA_CIRCULAR_REF` category, schema error).
- **AF-4: File Property Convention.** If a property name ends with `_file` or the property has an `x-cli-file: true` annotation, the system shall use `click.Path(exists=True)` as the type.

**Postconditions:**
- Each JSON Schema property is represented as a CLI flag on the exec command.

**Acceptance Criteria:**

- **AC-1:** Given a module with `input_schema.properties.name` of type `string`, when the system generates CLI flags, then `--name` shall accept a string value.
- **AC-2:** Given a module with `input_schema.properties.count` of type `integer`, when the user provides `--count 5`, then the value `5` shall be passed as an integer to the module.
- **AC-3:** Given a property named `input_file` of type `string`, when the system generates CLI flags, then the flag name shall be `--input-file`.
- **AC-4:** Given a property of type `object`, when the user provides `--data '{"key":"val"}'`, then the string shall be passed through for JSON parsing.

---

#### FR-SCHEMA-002: Boolean Flag Handling

| Field | Value |
|-------|-------|
| **ID** | FR-SCHEMA-002 |
| **Title** | Boolean Flag Handling |
| **Priority** | P0 |
| **Priority Rationale** | Boolean properties are common in module schemas. Incorrect mapping breaks module invocation for any boolean parameter. |
| **Source** | Feature Spec FE-02 FR-02-02 |

**Description:** The system shall map JSON Schema properties with `type: boolean` to Click boolean flag pairs. Each boolean property shall generate both a positive flag (`--property-name`) and a negative flag (`--no-property-name`). The default value shall be `False` unless the schema specifies `default: true`.

**Actors:** System (internal)

**Preconditions:**
- The Schema Parser is processing a property with `type: boolean`.

**Main Flow:**
1. The system shall create a `click.Option` with `is_flag=True` and flag declarations `--property-name/--no-property-name`.
2. If the schema specifies `default: true`, the default shall be `True`; otherwise `False`.
3. The flag value (`True` or `False`) shall be passed to the module input as the corresponding boolean.

**Alternative Flows:**

- **AF-1: Boolean with Enum.** If a boolean property also specifies an `enum` (e.g., `[true]`), the system shall treat it as a standard boolean flag and ignore the enum constraint (the flag inherently constrains to boolean values).

**Postconditions:**
- Boolean properties are represented as `--flag/--no-flag` pairs.

**Acceptance Criteria:**

- **AC-1:** Given a property `quiet` of type `boolean` with no default, when the user provides `--quiet`, then the module shall receive `quiet: true`. (Uses `quiet` to avoid visual collision with the retired global `--verbose` / current `--all-options` flag; the mechanism applies to any boolean property name.)
- **AC-2:** Given a property `quiet` of type `boolean` with no default, when the user provides `--no-quiet`, then the module shall receive `quiet: false`.
- **AC-3:** Given a property `quiet` of type `boolean` with `default: true`, when the user provides no flag, then the module shall receive `quiet: true`.

---

#### FR-SCHEMA-003: Enum Choice Mapping

| Field | Value |
|-------|-------|
| **ID** | FR-SCHEMA-003 |
| **Title** | Enum Choice Mapping |
| **Priority** | P0 |
| **Priority Rationale** | Enum properties are frequent in module schemas. Click.Choice provides immediate validation and help text. |
| **Source** | Feature Spec FE-02 FR-02-03 |

**Description:** The system shall map JSON Schema properties with an `enum` array to `click.Choice` options. The allowed values shall be the string representations of each enum member. Click shall enforce validation at the CLI layer, rejecting values not in the enum.

**Actors:** System (internal)

**Preconditions:**
- The Schema Parser is processing a property with an `enum` field.

**Main Flow:**
1. The system shall extract the `enum` array from the property.
2. The system shall convert each enum value to its string representation.
3. The system shall create a `click.Option` with `type=click.Choice([str(v) for v in enum_values])`.
4. The help text shall list the allowed values.

**Alternative Flows:**

- **AF-1: Empty Enum Array.** If the `enum` array is empty (`[]`), the system shall log a WARNING: `"Empty enum for property '{name}', no values allowed."` and treat the property as a regular string.
- **AF-2: Enum with Mixed Types.** If enum values include non-string types (integers, booleans), the system shall convert all to strings for the Click.Choice and reconvert the selected value back to the original type before passing to the module.

**Postconditions:**
- The CLI rejects values not in the enum before module invocation.

**Acceptance Criteria:**

- **AC-1:** Given a property `format` with `enum: ["json", "csv", "xml"]`, when the user provides `--format json`, then the value `"json"` shall be passed to the module.
- **AC-2:** Given a property `format` with `enum: ["json", "csv", "xml"]`, when the user provides `--format yaml`, then Click shall reject the input with an error listing the allowed values, and the process shall exit with code 2.

---

#### FR-SCHEMA-004: Required Property Enforcement

| Field | Value |
|-------|-------|
| **ID** | FR-SCHEMA-004 |
| **Title** | Required Property Enforcement |
| **Priority** | P0 |
| **Priority Rationale** | Without required enforcement, modules receive incomplete input leading to runtime errors that are harder to diagnose than CLI-level validation. |
| **Source** | Feature Spec FE-02 FR-02-04 |

**Description:** The system shall mark CLI flags corresponding to properties listed in the JSON Schema `required` array as `required=True` in Click. Click shall enforce presence of these flags and display an error if they are omitted.

**Actors:** System (internal)

**Preconditions:**
- The Schema Parser is processing a module's `input_schema` that contains a `required` array.

**Main Flow:**
1. The system shall read the `required` array from the `input_schema`.
2. For each property name in the `required` array, the system shall set `required=True` on the corresponding `click.Option`.
3. If a required flag is not provided on the command line and not supplied via STDIN (`--input -`), Click shall display an error: `Error: Missing required option '--{flag-name}'.`

**Alternative Flows:**

- **AF-1: Required Property Not in Properties.** If a property name in `required` does not appear in `properties`, the system shall log a WARNING and ignore the entry.
- **AF-2: Required Satisfied by STDIN.** If a required property is not provided as a CLI flag but is present in the STDIN JSON input (when `--input -` is used), the requirement shall be considered satisfied after merge.

**Postconditions:**
- All required properties are either provided via CLI flags or STDIN, or the system exits with an error.

**Acceptance Criteria:**

- **AC-1:** Given a module with required property `name`, when the user runs `apcore-cli exec module.id` without `--name`, then Click shall display "Missing required option '--name'" and exit with code 2.
- **AC-2:** Given a module with required property `name`, when the user runs `echo '{"name":"test"}' | apcore-cli exec module.id --input -`, then the requirement shall be satisfied.

---

#### FR-SCHEMA-005: Help Text Generation

| Field | Value |
|-------|-------|
| **ID** | FR-SCHEMA-005 |
| **Title** | Help Text Generation |
| **Priority** | P0 |
| **Priority Rationale** | Help text is the primary documentation mechanism for both developers reading `--help` and AI agents parsing command metadata. |
| **Source** | Feature Spec FE-02 FR-02-05 |

**Description:** The system shall generate CLI help text for each flag from the JSON Schema property metadata. If the property contains an `x-llm-description` field, the system shall use its value as the help text (optimized for AI agent consumption). Otherwise, the system shall use the `description` field. If neither field is present, the system shall display no help text for that flag.

**Actors:** System (internal)

**Preconditions:**
- The Schema Parser is generating Click options for a module.

**Main Flow:**
1. For each property, the system shall check for `x-llm-description` first.
2. If `x-llm-description` is present and is a non-empty string, use it as the Click option `help` parameter.
3. If `x-llm-description` is absent or empty, check for `description`.
4. If `description` is present and is a non-empty string, use it as the Click option `help` parameter.
5. If neither field is present, set `help=None` (Click displays no help text for the option).

**Alternative Flows:**

- **AF-1: Description Exceeds Configured Limit.** If the description exceeds the configured `cli.help_text_max_length` (default: 1000 characters), the system shall truncate it to `(limit - 3)` characters followed by `...` for the CLI help display. The limit is configurable via config file, environment variable (`APCORE_CLI_HELP_TEXT_MAX_LENGTH`), or built-in default. The full description shall remain available in `describe` output.

**Postconditions:**
- Each CLI flag has descriptive help text derived from the schema.

**Acceptance Criteria:**

- **AC-1:** Given a property with `description: "The user's full name"`, when the user runs `apcore-cli exec module.id --help`, then the `--name` flag help text shall contain "The user's full name".
- **AC-2:** Given a property with both `description: "Name"` and `x-llm-description: "Full legal name of the requesting user"`, when the system generates the flag, then the help text shall be "Full legal name of the requesting user".
- **AC-3:** Given a property with neither `description` nor `x-llm-description`, when the user runs `--help`, then the flag shall appear without help text.

---

#### FR-SCHEMA-006: Reference Resolution

| Field | Value |
|-------|-------|
| **ID** | FR-SCHEMA-006 |
| **Title** | JSON Schema Reference Resolution |
| **Priority** | P0 |
| **Priority Rationale** | Modules with composed schemas using `$ref` and `$defs` cannot be invoked without resolution. Blocking for any non-trivial module. |
| **Source** | Feature Spec FE-02 FR-02-06; Tech Design v2.0 section 8.2 |

**Description:** The system shall inline all `$ref` and `$defs` references in a module's `input_schema` before parsing properties into CLI flags. Resolution shall track visited references to detect circular references. The maximum `$ref` resolution depth shall be 32. Schema composition keywords (`allOf`, `anyOf`, `oneOf`) shall be flattened to top-level properties up to 3 levels of nesting.

**Actors:** System (internal)

**Preconditions:**
- The module's `input_schema` contains `$ref` or `$defs` references.

**Main Flow:**
1. The system shall walk the `input_schema` tree.
2. For each `$ref` encountered, the system shall:
   a. Increment the depth counter.
   b. Check if the depth exceeds 32. If so, abort with error.
   c. Check if the `$ref` target is in the visited set. If so, abort with circular reference error.
   d. Add the `$ref` target to the visited set.
   e. Resolve the reference and inline the target schema.
3. For `allOf` arrays, the system shall merge all sub-schemas into a single schema with combined `properties` and `required` arrays.
4. For `anyOf`/`oneOf` arrays, the system shall merge all sub-schema properties (union), marking only shared required properties as required.
5. After resolution, the system shall pass the fully inlined schema to the property-to-flag mapping (FR-SCHEMA-001).

**Alternative Flows:**

- **AF-1: Circular Reference Detected.** If a `$ref` target is already in the visited set, the system shall write to stderr: `Error: Circular $ref detected in schema for module '{module_id}' at path '{ref_path}'.` and exit with code 48 (`SCHEMA_CIRCULAR_REF`).
- **AF-2: Max Depth Exceeded.** If the resolution depth exceeds 32, the system shall write to stderr: `Error: $ref resolution depth exceeded maximum of 32 for module '{module_id}'.` and exit with code 48 (`SCHEMA_CIRCULAR_REF`).
- **AF-3: Unresolvable Reference.** If a `$ref` target path does not exist in the schema's `$defs`, the system shall write to stderr: `Error: Unresolvable $ref '{ref_value}' in schema for module '{module_id}'.` and exit with code 45 (`SCHEMA_VALIDATION_ERROR`).

**Postconditions:**
- The `input_schema` is fully inlined with no `$ref` or `$defs` references remaining.

**Acceptance Criteria:**

- **AC-1:** Given a schema with `$ref: "#/$defs/Address"` and a valid `$defs.Address` definition, when the system resolves references, then the Address properties shall appear as top-level CLI flags.
- **AC-2:** Given a schema with a circular reference (`A -> B -> A`), when the system resolves references, then it shall exit with code 48 and stderr shall contain "Circular $ref detected".
- **AC-3:** Given a schema with `$ref` nesting depth of 33, when the system resolves references, then it shall exit with code 48 and stderr shall contain "depth exceeded".

---

### 5.3 Approval Gate (APPR)

#### FR-APPR-001: Approval Check

| Field | Value |
|-------|-------|
| **ID** | FR-APPR-001 |
| **Title** | Approval Annotation Check |
| **Priority** | P1 |
| **Priority Rationale** | Safety mechanism. Required before executing any destructive or sensitive module. Lower than P0 because non-approval modules can function without this. |
| **Source** | Feature Spec FE-03 FR-03-01 |

**Description:** The system shall inspect the module's `annotations.requires_approval` field after schema validation and before execution. If the field is `true`, the system shall invoke the Approval Gate workflow. If the field is `false`, absent, or not a boolean, the system shall skip the Approval Gate and proceed directly to execution.

**Actors:** System (internal)

**Preconditions:**
- The module has been resolved from the Registry.
- Input arguments have been validated against the schema.

**Main Flow:**
1. The system shall read `annotations.requires_approval` from the module definition.
2. If the value is boolean `true`, the system shall proceed to the Approval Gate (FR-APPR-002 or FR-APPR-003 depending on TTY status).
3. If the value is `false`, absent, `null`, or any non-boolean type, the system shall skip the Approval Gate and proceed to execution.

**Alternative Flows:**

- **AF-1: Annotations Field Missing.** If the module definition has no `annotations` field, the system shall treat `requires_approval` as `false` and proceed to execution.

**Postconditions:**
- The system either enters the Approval Gate workflow or proceeds to execution.

**Acceptance Criteria:**

- **AC-1:** Given a module with `annotations.requires_approval: true`, when the user invokes `exec`, then the Approval Gate shall activate before execution.
- **AC-2:** Given a module with `annotations.requires_approval: false`, when the user invokes `exec`, then execution shall proceed without any approval prompt.
- **AC-3:** Given a module with no `annotations` field, when the user invokes `exec`, then execution shall proceed without any approval prompt.

---

#### FR-APPR-002: TTY Interactive Prompt

| Field | Value |
|-------|-------|
| **ID** | FR-APPR-002 |
| **Title** | TTY Interactive Approval Prompt |
| **Priority** | P1 |
| **Priority Rationale** | Primary interactive safety mechanism for human developers in terminal sessions. |
| **Source** | Feature Spec FE-03 FR-03-03 |

**Description:** When the Approval Gate is activated and the process is running in a TTY (i.e., `sys.stdin.isatty()` returns `True`), the system shall display the module's `ApprovalRequest.message` to the terminal and prompt the user for confirmation with `[y/N]`. The default response shall be `N` (deny). The system shall use `click.confirm()` for the prompt.

**Actors:** Developer

**Preconditions:**
- `annotations.requires_approval` is `true`.
- `sys.stdin.isatty()` returns `True`.
- No bypass mechanism is active (see FR-APPR-004).

**Main Flow:**
1. The system shall display the approval message from the module's metadata.
2. The system shall display the prompt: `"Proceed? [y/N]: "`.
3. The user types `y` or `Y` and presses Enter.
4. The system shall log at INFO level: `"User approved execution of module '{module_id}'."`.
5. The system shall proceed to module execution.

**Alternative Flows:**

- **AF-1: User Denies.** If the user types `n`, `N`, or presses Enter without input (default N), the system shall log at WARNING level: `"Approval rejected by user for module '{module_id}'."`, write to stderr: `Error: Approval denied.`, and exit with code 46 (`APPROVAL_DENIED`).
- **AF-2: Invalid Input.** If the user types any value other than `y`, `Y`, `n`, `N`, or empty, `click.confirm()` shall re-prompt. This is handled by Click's built-in behavior.
- **AF-3: Approval Message Missing.** If the module does not provide an `ApprovalRequest.message`, the system shall display a default message: `"Module '{module_id}' requires approval to execute."`.

**Postconditions:**
- User has approved (proceed to execution) or denied (exit with code 46).

**Acceptance Criteria:**

- **AC-1:** Given a TTY session and a module with `requires_approval: true` and message "This will delete data", when the Approval Gate activates, then the terminal shall display "This will delete data" followed by "Proceed? [y/N]: ".
- **AC-2:** Given the prompt is displayed and the user types `y`, then execution shall proceed.
- **AC-3:** Given the prompt is displayed and the user presses Enter (empty input), then the process shall exit with code 46.

---

#### FR-APPR-003: Non-TTY Behavior

| Field | Value |
|-------|-------|
| **ID** | FR-APPR-003 |
| **Title** | Non-TTY Approval Behavior |
| **Priority** | P1 |
| **Priority Rationale** | Critical for AI Agent usage. Agents invoke CLI via shell exec in non-TTY contexts and need deterministic failure behavior. |
| **Source** | Feature Spec FE-03 FR-03-04 |

**Description:** When the Approval Gate is activated and the process is NOT running in a TTY (i.e., `sys.stdin.isatty()` returns `False`), and no bypass mechanism is active, the system shall reject execution with exit code 46 (`APPROVAL_PENDING`). The system shall write to stderr a message instructing the caller to use `--yes` or `APCORE_CLI_AUTO_APPROVE=1`.

**Actors:** AI Agent

**Preconditions:**
- `annotations.requires_approval` is `true`.
- `sys.stdin.isatty()` returns `False`.
- Neither `--yes` flag nor `APCORE_CLI_AUTO_APPROVE=1` is active.

**Main Flow:**
1. The system shall detect the non-TTY environment via `sys.stdin.isatty() == False`.
2. The system shall write to stderr: `Error: Module '{module_id}' requires approval but no interactive terminal is available. Use --yes or set APCORE_CLI_AUTO_APPROVE=1 to bypass.`
3. The system shall log at ERROR level: `"Non-interactive environment, no bypass provided for module '{module_id}'."`.
4. The system shall exit with code 46 (`APPROVAL_PENDING`).

**Alternative Flows:**

- **AF-1: Bypass Active.** If `--yes` or `APCORE_CLI_AUTO_APPROVE=1` is active, this flow does not apply; see FR-APPR-004.

**Postconditions:**
- Execution is rejected. Exit code is 46.

**Acceptance Criteria:**

- **AC-1:** Given a non-TTY environment and a module with `requires_approval: true` and no bypass, when the agent invokes `apcore-cli exec module.id`, then stderr shall contain "requires approval but no interactive terminal" and the process shall exit with code 46.
- **AC-2:** Given the same conditions but `--yes` is provided, then execution shall proceed (FR-APPR-004 governs).

---

#### FR-APPR-004: Bypass Mechanisms

| Field | Value |
|-------|-------|
| **ID** | FR-APPR-004 |
| **Title** | Approval Bypass Mechanisms |
| **Priority** | P1 |
| **Priority Rationale** | Enables CI/CD pipelines, scripted workflows, and agent automation to run approval-required modules without interactive prompts. |
| **Source** | Feature Spec FE-03 FR-03-05; Tech Design v2.0 section 8.3 |

**Description:** The system shall provide two mechanisms to bypass the Approval Gate. Bypass evaluation shall follow a strict priority order: (1) `--yes` CLI flag (highest), (2) `APCORE_CLI_AUTO_APPROVE=1` environment variable. If either is active, the system shall skip the TTY prompt and non-TTY rejection, log the bypass, and proceed directly to execution.

**Actors:** Developer, AI Agent, CI/CD System

**Preconditions:**
- `annotations.requires_approval` is `true`.

**Main Flow:**
1. The system shall check for `--yes` CLI flag. If present, bypass approval.
2. If `--yes` is not present, the system shall check for `APCORE_CLI_AUTO_APPROVE` environment variable. If its value is exactly `"1"`, bypass approval.
3. When bypassing, the system shall log at INFO level: `"Approval bypassed via {mechanism} for module '{module_id}'."` where `{mechanism}` is `--yes flag` or `APCORE_CLI_AUTO_APPROVE`.
4. The system shall proceed to module execution.

**Alternative Flows:**

- **AF-1: APCORE_CLI_AUTO_APPROVE Invalid Value.** If `APCORE_CLI_AUTO_APPROVE` is set to any value other than `"1"` (e.g., `"true"`, `"yes"`, `"0"`), the system shall treat it as unset and not bypass. The system shall log a WARNING: `"APCORE_CLI_AUTO_APPROVE is set to '{value}', expected '1'. Ignoring."`.
- **AF-2: Both Mechanisms Active.** If both `--yes` and `APCORE_CLI_AUTO_APPROVE=1` are active, the system shall use `--yes` (higher priority) and log accordingly. No error.

**Postconditions:**
- Approval is bypassed. Module execution proceeds.

**Acceptance Criteria:**

- **AC-1:** Given `--yes` is provided, when invoking a module with `requires_approval: true`, then execution shall proceed without prompting and the log shall contain "bypassed via --yes flag".
- **AC-2:** Given `APCORE_CLI_AUTO_APPROVE=1` is set and no `--yes` flag, when invoking a module with `requires_approval: true` in a non-TTY, then execution shall proceed.
- **AC-3:** Given `APCORE_CLI_AUTO_APPROVE=true` (not `"1"`), when invoking a module with `requires_approval: true` in a non-TTY, then the process shall exit with code 46 and a warning shall be logged about the invalid value.

---

#### FR-APPR-005: Approval Timeout

| Field | Value |
|-------|-------|
| **ID** | FR-APPR-005 |
| **Title** | Approval Timeout |
| **Priority** | P1 |
| **Priority Rationale** | Prevents indefinite blocking in TTY sessions where the user steps away or forgets to respond. |
| **Source** | Feature Spec FE-03 section 3.2 |

**Description:** When the Approval Gate displays a TTY prompt (FR-APPR-002), the system shall enforce a timeout of 60 seconds. If the user does not respond within the timeout period, the system shall treat the prompt as denied due to timeout.

**Actors:** Developer

**Preconditions:**
- The TTY approval prompt (FR-APPR-002) is displayed and waiting for input.

**Main Flow:**
1. The system shall start a 60-second timer when the prompt is displayed.
2. If the user responds within 60 seconds, proceed per FR-APPR-002 (approved or denied).
3. If 60 seconds elapse without user input, proceed to AF-1.

**Alternative Flows:**

- **AF-1: Timeout Reached.** The system shall log at WARNING level: `"Approval timed out after 60s for module '{module_id}'."`. The system shall write to stderr: `Error: Approval prompt timed out after 60 seconds.` and exit with code 46 (`APPROVAL_TIMEOUT`).

**Postconditions:**
- User responded within timeout (proceed per FR-APPR-002) or timeout triggered (exit with code 46).

**Acceptance Criteria:**

- **AC-1:** Given a TTY approval prompt is displayed and 60 seconds elapse without input, then stderr shall contain "timed out after 60 seconds" and the process shall exit with code 46.
- **AC-2:** Given a TTY approval prompt is displayed and the user responds with `y` at 59 seconds, then execution shall proceed normally.

---

### 5.4 Discovery (DISC)

#### FR-DISC-001: Module Listing

| Field | Value |
|-------|-------|
| **ID** | FR-DISC-001 |
| **Title** | Module Listing |
| **Priority** | P1 |
| **Priority Rationale** | Enables developers and agents to discover available modules. Without listing, users must know module IDs a priori. |
| **Source** | Feature Spec FE-04 FR-04-01 |

**Description:** The system shall provide an `apcore-cli list` subcommand that displays all modules in the Registry as a formatted table. The table shall contain columns for Module ID, Description (first 80 characters), and Tags. The system shall invoke `Registry.list()` to retrieve the module set.

**Actors:** Developer, AI Agent

**Preconditions:**
- The Registry is loaded with at least zero modules.

**Main Flow:**
1. The user invokes `apcore-cli list`.
2. The system shall invoke `Registry.list()` to retrieve all module definitions.
3. The system shall format the results as a table with columns: `ID`, `Description`, `Tags`.
4. The Description column shall display the first 80 characters of the module's `description` field, appending `...` if truncated.
5. The Tags column shall display a comma-separated list of the module's tags.
6. The system shall render the table using `rich.table.Table` when output format is `table`.
7. The system shall write the table to stdout and exit with code 0.

**Alternative Flows:**

- **AF-1: Empty Registry.** If the Registry contains zero modules, the system shall display an empty table with headers and a note: `"No modules found."` and exit with code 0.
- **AF-2: JSON Output Format.** If `--format json` is specified (or the default for non-TTY per FR-DISC-004), the system shall output a JSON array of objects with keys `id`, `description`, `tags`.

**Postconditions:**
- Module list is written to stdout. Exit code is 0.

**Acceptance Criteria:**

- **AC-1:** Given a Registry with modules `math.add` (tags: `math, core`) and `text.summarize` (tags: `text`), when the user runs `apcore-cli list`, then stdout shall contain a table with both module IDs, their descriptions, and tags.
- **AC-2:** Given an empty Registry, when the user runs `apcore-cli list`, then stdout shall contain "No modules found" and the process shall exit with code 0.
- **AC-3:** Given a module with a 120-character description, when listed, then the Description column shall show 80 characters followed by `...`.

---

#### FR-DISC-002: Tag Filtering

| Field | Value |
|-------|-------|
| **ID** | FR-DISC-002 |
| **Title** | Tag Filtering |
| **Priority** | P1 |
| **Priority Rationale** | Enables focused discovery in large registries. AND logic prevents flooding results and matches developer expectations. |
| **Source** | Feature Spec FE-04 FR-04-03 |

**Description:** The system shall accept one or more `--tag` flags on the `list` subcommand. When tags are provided, the system shall filter the module list to include only modules whose tag set contains ALL specified tags (AND logic). The `--tag` flag shall be repeatable.

**Actors:** Developer, AI Agent

**Preconditions:**
- The `list` subcommand is invoked with at least one `--tag` flag.

**Main Flow:**
1. The user invokes `apcore-cli list --tag math --tag core`.
2. The system shall collect all `--tag` values into a set: `{math, core}`.
3. The system shall invoke `Registry.list(tags=["math", "core"])` or filter the full list by tag intersection.
4. The system shall include only modules where `module.tags` is a superset of the specified tags.
5. The system shall display the filtered results per FR-DISC-001.

**Alternative Flows:**

- **AF-1: No Matching Modules.** If no modules match all specified tags, the system shall display an empty table with the note: `"No modules found matching tags: {tag_list}."` and exit with code 0.
- **AF-2: Tag Does Not Exist.** If a specified tag does not exist on any module, this is not an error; the result is simply an empty set (per AF-1).

**Postconditions:**
- Only modules matching ALL specified tags are displayed.

**Acceptance Criteria:**

- **AC-1:** Given modules `math.add` (tags: `math, core`) and `text.summarize` (tags: `text`), when the user runs `apcore-cli list --tag math`, then only `math.add` shall appear.
- **AC-2:** Given the same modules, when the user runs `apcore-cli list --tag math --tag core`, then only `math.add` shall appear (it has both tags).
- **AC-3:** Given the same modules, when the user runs `apcore-cli list --tag math --tag text`, then no modules shall appear (no module has both tags).

---

#### FR-DISC-003: Module Description

| Field | Value |
|-------|-------|
| **ID** | FR-DISC-003 |
| **Title** | Module Description Display |
| **Priority** | P1 |
| **Priority Rationale** | Enables detailed module inspection before execution. Critical for both developer understanding and agent capability assessment. |
| **Source** | Feature Spec FE-04 FR-04-02, FR-04-04 |

**Description:** The system shall provide an `apcore-cli describe <module_id>` subcommand that displays the full metadata for a single module. The output shall include the module's core fields (description, input_schema, output_schema), annotations (readonly, destructive, idempotent, requires_approval), and extension metadata (x-when-to-use, x-preconditions). JSON schemas shall be rendered with syntax highlighting using `rich.syntax` when the output format is `table`.

**Actors:** Developer, AI Agent

**Preconditions:**
- The Registry is loaded.

**Main Flow:**
1. The user invokes `apcore-cli describe <module_id>`.
2. The system shall validate `<module_id>` conforms to the Canonical ID format.
3. The system shall look up `<module_id>` in the Registry via `Registry.get_definition()` and `Registry.get_schema()`.
4. The system shall format and display:
   - **Core**: Module ID, description (full text), input_schema (syntax-highlighted JSON), output_schema (syntax-highlighted JSON).
   - **Annotations**: readonly, destructive, idempotent, requires_approval, and any other annotation fields.
   - **Extension**: All `x-` prefixed metadata fields.
5. The system shall write the output to stdout and exit with code 0.

**Alternative Flows:**

- **AF-1: Module Not Found.** If `<module_id>` is not in the Registry, the system shall write to stderr: `Error: Module '{module_id}' not found.` and exit with code 44 (`MODULE_NOT_FOUND`).
- **AF-2: JSON Output Format.** If `--format json` is specified, the system shall output the module metadata as a single JSON object without syntax highlighting.
- **AF-3: Module Without Optional Fields.** If the module lacks `output_schema`, annotations, or extension metadata, the system shall omit those sections from the display rather than showing empty sections.
- **AF-4: Markdown / Skill Output Format.** If `--format markdown` or `--format skill` is specified, the system shall delegate rendering to `apcore_toolkit.format_module(..., style="markdown"|"skill", display=True)` so the body is byte-identical to the toolkit's contract (see FR-DISC-004 main flow steps 5–6 and apcore-toolkit `formatting.md` § "Style Matrix").

**Postconditions:**
- Full module metadata is displayed. Exit code is 0 or 44.

**Acceptance Criteria:**

- **AC-1:** Given module `math.add` with `input_schema: {properties: {a: {type: integer}, b: {type: integer}}}`, when the user runs `apcore-cli describe math.add`, then stdout shall contain the module ID, description, and the input schema.
- **AC-2:** Given module `math.add` with `annotations.requires_approval: true`, when described, then stdout shall display `requires_approval: true`.
- **AC-3:** Given module ID `non.existent`, when the user runs `apcore-cli describe non.existent`, then stderr shall contain "not found" and the process shall exit with code 44.

---

#### FR-DISC-004: Output Format Selection

| Field | Value |
|-------|-------|
| **ID** | FR-DISC-004 |
| **Title** | Output Format Selection |
| **Priority** | P1 |
| **Priority Rationale** | AI agents require structured output (JSON for programmatic consumption; Markdown / Skill for prompt injection and Agent-Skill export to Claude Code / Gemini CLI). Developers prefer readable tables. TTY-adaptive defaults eliminate the need for explicit flags in common cases. |
| **Source** | Feature Spec FE-08 (formerly FE-04 FR-04-05); apcore-toolkit `formatting.md` Style Matrix |

**Description:** The system shall accept a `--format` flag whose canonical choice set depends on the subcommand's data shape:

- **`list` and `describe`** (render `ScannedModule` metadata) accept the full set `[table, json, csv, yaml, jsonl, markdown, skill]`. The `markdown` and `skill` styles delegate to `apcore_toolkit.format_module(s)` so output is byte-identical across the Python / TypeScript / Rust SDKs (see apcore-toolkit `formatting.md` § "Annotations Rendering").
- **`apcli system *`, `apcli strategy *`, and `apcli exec`** (render arbitrary business / health / strategy results, not `ScannedModule`) accept `[table, json, csv, yaml, jsonl]`. `markdown` and `skill` are deliberately excluded — the surface-aware toolkit formatters target `ScannedModule` and are not meaningful for these payloads.

The default shall be TTY-adaptive: `table` when `sys.stdout.isatty()` returns `True`, `json` when `sys.stdout.isatty()` returns `False`. The `--format` flag shall override the TTY-adaptive default.

**Actors:** Developer, AI Agent

**Preconditions:**
- One of the supported subcommands is invoked.

**Main Flow:**
1. The system shall check for a `--format` flag.
2. If `--format table` is specified, render output as a formatted table using `rich` (or the SDK's idiomatic equivalent).
3. If `--format json` is specified, render output as a JSON document (array for `list`, object for `describe`) with 2-space indentation.
4. If `--format csv` or `--format jsonl` is specified, the SDK MUST delegate to `apcore_toolkit.format_csv(rows)` or `apcore_toolkit.format_jsonl(rows)` respectively. Per-SDK reimplementation is prohibited; output MUST be byte-identical across Python / TypeScript / Rust (see tech-design ADR-09 and `apcore-toolkit/conformance/fixtures/format_csv.json` / `format_jsonl.json`). If `--format yaml` is specified, render via the SDK's idiomatic YAML library (currently Tier 2 SDK-native — byte-equivalent YAML pending). (added v0.6.0; csv/jsonl promoted to toolkit-delegated tier v0.7.0)
5. If `--format markdown` is specified, delegate to `apcore_toolkit.format_module(s)(..., style="markdown")`. For `list`, the result is the per-module Markdown bodies joined by a blank line (optionally grouped by tag/prefix).
6. If `--format skill` is specified, delegate to `apcore_toolkit.format_module(s)(..., style="skill")` — Markdown body prefixed with the vendor-neutral YAML frontmatter (`name`, `description`). The output is shaped to be written as `.claude/skills/<id>/SKILL.md` or `.gemini/skills/<id>/SKILL.md`.
7. If no `--format` is specified:
   a. If `sys.stdout.isatty()` is `True`, default to `table`.
   b. If `sys.stdout.isatty()` is `False`, default to `json`.

**Alternative Flows:**

- **AF-1: Invalid Format Value.** If the user provides a value outside the canonical choice set, Click (or the SDK's CLI parser) shall reject the input with an error listing the allowed choices and exit with code 2.
- **AF-2: Non-module subcommand scope.** The `apcli exec`, `apcli system *`, and `apcli strategy *` subcommands render arbitrary business / health / strategy results, not `ScannedModule` metadata. Their `--format` choice set is `[table, json, csv, yaml, jsonl]`; `markdown` and `skill` are not accepted.

**Postconditions:**
- Output is rendered in the selected or defaulted format.

**Acceptance Criteria:**

- **AC-1:** Given a TTY session, when the user runs `apcore-cli list` without `--format`, then the output shall be a formatted table.
- **AC-2:** Given a non-TTY context (e.g., piped output), when the agent runs `apcore-cli list` without `--format`, then the output shall be valid JSON.
- **AC-3:** Given a TTY session, when the user runs `apcore-cli list --format json`, then the output shall be valid JSON (overriding the TTY default).
- **AC-4:** Given `--format markdown` on `list` or `describe`, then the output shall match the Markdown rendering produced by `apcore_toolkit.format_module(s)(..., style="markdown")` byte-for-byte.
- **AC-5:** Given `--format skill` on `describe <id>`, then the first line of stdout shall be `---`, followed by `name:` and `description:` lines, followed by `---`, followed by the same Markdown body as `--format markdown`. The result shall be loadable by Claude Code and Gemini CLI as a SKILL definition.
- **AC-6:** Given `--format <unknown>` on any covered subcommand, then the SDK's CLI parser shall reject the value and exit with code 2.
- **AC-7 (added v0.7.0):** Given `--format csv` or `--format jsonl` on any covered subcommand, the SDK's stdout shall be byte-identical to `apcore_toolkit.format_csv(rows)` / `format_jsonl(rows)` respectively. The 3 SDKs (Python / TypeScript / Rust) shall pass the shared conformance corpus at `apcore-toolkit/conformance/fixtures/`. Nested values shall be serialized as canonical compact JSON (no Python repr, no `[object Object]`). CSV header columns shall be the union of keys across all rows (heterogeneous rows shall not drop fields). CSV line terminator shall be CRLF per RFC 4180.

---

### 5.5 Security (SEC)

#### FR-SEC-001: API Key Authentication

| Field | Value |
|-------|-------|
| **ID** | FR-SEC-001 |
| **Title** | API Key Authentication |
| **Priority** | P1 |
| **Priority Rationale** | Required for remote registry access. Without authentication, remote registries cannot verify caller identity. |
| **Source** | User requirements (security stack) |

**Description:** The system shall authenticate with remote registries using API keys. The API key shall be resolved using the configuration precedence (FR-DISP-005): CLI flag `--api-key`, environment variable `APCORE_AUTH_API_KEY`, or configuration file `apcore.yaml` key `auth.api_key`. The system shall transmit the API key in the `Authorization` header as `Bearer {key}` when making remote registry requests.

**Actors:** Developer, AI Agent

**Preconditions:**
- A remote registry endpoint is configured.
- An API key is available through one of the configuration sources.

**Main Flow:**
1. The system shall resolve the API key using the configuration precedence order.
2. The system shall include the API key as `Authorization: Bearer {key}` in all HTTP requests to the remote registry.
3. The system shall proceed with the Registry operation (list, get_definition, etc.).

**Alternative Flows:**

- **AF-1: No API Key Configured.** If a remote registry is configured but no API key is available from any source, the system shall write to stderr: `Error: Remote registry requires authentication. Set --api-key, APCORE_AUTH_API_KEY, or auth.api_key in config.` and exit with code 77 (`ACL_DENIED`).
- **AF-2: Authentication Rejected.** If the remote registry responds with HTTP 401 or 403, the system shall write to stderr: `Error: Authentication failed. Verify your API key.` and exit with code 77 (`ACL_DENIED`).
- **AF-3: Local Registry Only.** If no remote registry is configured (local-only mode), the system shall not require or use an API key.

**Postconditions:**
- Remote registry requests include valid authentication credentials.

**Acceptance Criteria:**

- **AC-1:** Given a remote registry and `APCORE_AUTH_API_KEY=abc123`, when the system accesses the registry, then HTTP requests shall include `Authorization: Bearer abc123`.
- **AC-2:** Given a remote registry and no API key configured, when the system attempts access, then stderr shall contain "requires authentication" and the process shall exit with code 77.
- **AC-3:** Given a local-only registry, when the system operates, then no API key shall be required.

---

#### FR-SEC-002: Encrypted Configuration

| Field | Value |
|-------|-------|
| **ID** | FR-SEC-002 |
| **Title** | Encrypted Configuration Storage |
| **Priority** | P1 |
| **Priority Rationale** | Prevents credential exposure in plaintext config files on shared or compromised filesystems. |
| **Source** | User requirements (security stack) |

**Description:** The system shall encrypt sensitive values (API keys, tokens, secrets) when persisting them to the configuration file `apcore.yaml`. The system shall use the operating system's native keyring (via `keyring` library) as the primary storage mechanism. If the keyring is unavailable, the system shall encrypt values using AES-256-GCM with a machine-derived key and store the ciphertext in the config file with an `enc:` prefix.

**Actors:** System (internal)

**Preconditions:**
- A sensitive value is being written to the configuration file.

**Main Flow:**
1. The system shall detect if the OS keyring is available.
2. If available, the system shall store the sensitive value in the keyring under the service name `apcore-cli` and key name matching the config key (e.g., `auth.api_key`).
3. The config file shall contain a reference: `auth.api_key: "keyring:auth.api_key"`.
4. When reading the value, the system shall detect the `keyring:` prefix and retrieve from the OS keyring.

**Alternative Flows:**

- **AF-1: Keyring Unavailable.** If the OS keyring is not available (headless server, container), the system shall encrypt the value using AES-256-GCM with a machine-derived key (hostname + user + salt), store as `auth.api_key: "enc:base64ciphertext"`, and log a WARNING: `"OS keyring unavailable. Using file-based encryption."`.
- **AF-2: Decryption Failure.** If the system cannot decrypt a stored value (key mismatch, corrupted data), the system shall write to stderr: `Error: Failed to decrypt configuration value '{key}'. Re-configure with 'apcore-cli config set {key}'.` and exit with code 47 (`CONFIG_INVALID`).

**Postconditions:**
- Sensitive values are not stored in plaintext in the configuration file.

**Acceptance Criteria:**

- **AC-1:** Given a sensitive value `auth.api_key` is configured, when inspecting `apcore.yaml`, then the value shall NOT appear in plaintext; it shall be prefixed with `keyring:` or `enc:`.
- **AC-2:** Given the OS keyring is available, when the system reads `auth.api_key`, then it shall retrieve the value from the keyring and use it for authentication.
- **AC-3:** Given the system cannot decrypt a stored value, when the system reads the config, then stderr shall contain "Failed to decrypt" and the process shall exit with code 47.

---

#### FR-SEC-003: Audit Logging

| Field | Value |
|-------|-------|
| **ID** | FR-SEC-003 |
| **Title** | Audit Logging |
| **Priority** | P1 |
| **Priority Rationale** | Provides accountability and forensic capability for all module executions, essential for compliance and debugging. |
| **Source** | User requirements (security stack) |

**Description:** The system shall log every module execution to an append-only audit log file. Each audit entry shall contain: ISO 8601 timestamp, invoking user (from `os.getlogin()` or `USER` env var), module Canonical ID, input hash (SHA-256 of the serialized input, not the raw input), execution result status (`success` or `error`), exit code, and execution duration in milliseconds.

**Actors:** System (internal)

**Preconditions:**
- A module execution has been initiated (after approval, if applicable).

**Main Flow:**
1. The system shall record the start timestamp.
2. After module execution completes (success or failure), the system shall compute the SHA-256 hash of the serialized input JSON.
3. The system shall write an audit log entry in JSON Lines format to the audit log file (default: `~/.apcore-cli/audit.jsonl`).
4. Each entry shall contain: `{"timestamp": "...", "user": "...", "module_id": "...", "input_hash": "...", "status": "success|error", "exit_code": 0, "duration_ms": 42}`.

**Alternative Flows:**

- **AF-1: Audit Log Write Failure.** If the audit log file cannot be written (permissions, disk full), the system shall log a WARNING to stderr: `"Warning: Could not write audit log: {reason}."` and continue execution. Audit log failure shall NOT prevent module execution.
- **AF-2: User Cannot Be Determined.** If `os.getlogin()` raises an exception and `USER` env var is unset, the system shall record `"user": "unknown"`.

**Postconditions:**
- An audit entry is appended to the audit log file.

**Acceptance Criteria:**

- **AC-1:** Given a successful execution of `math.add`, when the audit log is inspected, then it shall contain a JSON line with `module_id: "math.add"`, `status: "success"`, `exit_code: 0`, and a valid ISO 8601 timestamp.
- **AC-2:** Given a failed execution, when the audit log is inspected, then it shall contain `status: "error"` and the corresponding exit code.
- **AC-3:** Given the audit log file is not writable, when a module executes, then the module shall still execute successfully and a warning shall appear on stderr.

---

#### FR-SEC-004: Execution Sandboxing

| Field | Value |
|-------|-------|
| **ID** | FR-SEC-004 |
| **Title** | Execution Sandboxing |
| **Priority** | P2 |
| **Priority Rationale** | Defense-in-depth measure. Important for running untrusted modules but not blocking for initial release with trusted local modules. |
| **Source** | User requirements (security stack) |

**Description:** The system shall isolate module execution from the host system by restricting the module's access to the filesystem, network, and environment variables. The system shall use process-level isolation (subprocess with restricted environment) for module execution when the `--sandbox` flag is provided or `APCORE_CLI_SANDBOX=1` is set.

**Actors:** System (internal)

**Preconditions:**
- Sandbox mode is activated via `--sandbox` flag or `APCORE_CLI_SANDBOX=1`.

**Main Flow:**
1. The system shall create a restricted subprocess environment containing only explicitly allowed environment variables.
2. The system shall set the working directory to a temporary directory.
3. The system shall invoke the module within this restricted environment.
4. After execution, the system shall clean up the temporary directory.

**Alternative Flows:**

- **AF-1: Sandbox Not Requested.** If neither `--sandbox` nor `APCORE_CLI_SANDBOX=1` is active, the system shall execute the module in the current process environment (no isolation).
- **AF-2: Platform Not Supported.** If the current platform does not support the sandboxing mechanism, the system shall log a WARNING: `"Sandboxing not available on this platform. Executing without isolation."` and proceed without sandbox.

**Postconditions:**
- Module execution is isolated (when sandbox is active) or runs in the normal environment.

**Acceptance Criteria:**

- **AC-1:** Given `--sandbox` is provided, when a module attempts to read `HOME` environment variable, then it shall not find it in the sandbox environment.
- **AC-2:** Given `--sandbox` is provided, when a module writes a file, then it shall be written to the temporary directory, not the user's working directory.
- **AC-3:** Given no sandbox flag, when a module executes, then it shall run with access to the normal environment.

---

### 5.6 Shell Integration (SHELL)

#### FR-SHELL-001: Shell Completion

| Field | Value |
|-------|-------|
| **ID** | FR-SHELL-001 |
| **Title** | Shell Completion Script Generation |
| **Priority** | P2 |
| **Priority Rationale** | Quality-of-life improvement for developers. Not blocking for MVP but significantly improves usability. |
| **Source** | User requirements (shell ecosystem integration) |

**Description:** The system shall generate shell completion scripts for Bash, Zsh, and Fish shells. The generated scripts shall provide completion for subcommands (`exec`, `list`, `describe`), module Canonical IDs (dynamic, from Registry), and module-specific flags (dynamic, from `input_schema`). The system shall provide an `apcore-cli completion <shell>` subcommand where `<shell>` is one of `bash`, `zsh`, `fish`.

**Actors:** Developer

**Preconditions:**
- The Registry is loaded.

**Main Flow:**
1. The user invokes `apcore-cli completion bash` (or `zsh` or `fish`).
2. The system shall generate a shell completion script appropriate for the specified shell.
3. The script shall include completions for:
   - Top-level subcommands: `exec`, `list`, `describe`, `completion`.
   - Module IDs: dynamically enumerated from the Registry.
   - Module flags: dynamically generated from each module's `input_schema`.
4. The system shall write the script to stdout and exit with code 0.
5. The user shall source the script in their shell profile.

**Alternative Flows:**

- **AF-1: Unsupported Shell.** If the user specifies a shell other than `bash`, `zsh`, or `fish`, the system shall write to stderr: `Error: Unsupported shell '{shell}'. Supported: bash, zsh, fish.` and exit with code 2.

**Postconditions:**
- A shell completion script is written to stdout.

**Acceptance Criteria:**

- **AC-1:** Given the user runs `apcore-cli completion bash`, then stdout shall contain a valid Bash completion script that includes `exec`, `list`, and `describe` as completable subcommands.
- **AC-2:** Given the user runs `apcore-cli completion invalid`, then stderr shall contain "Unsupported shell" and the process shall exit with code 2.
- **AC-3:** Given the Registry contains `math.add`, when the generated Bash completion is sourced and the user types `apcore-cli exec math.<TAB>`, then `math.add` shall appear as a completion candidate.

---

#### FR-SHELL-002: Man Page Generation

| Field | Value |
|-------|-------|
| **ID** | FR-SHELL-002 |
| **Title** | Man Page Generation |
| **Priority** | P2 |
| **Priority Rationale** | Standard Unix convention for CLI documentation. Not blocking but provides integration with `man` command. |
| **Source** | User requirements (shell ecosystem integration) |

**Description:** The system shall generate man pages in roff format from module metadata and CLI command definitions. Two generation modes are supported:

1. **Single-command mode** — `apcore-cli man <command>` generates a man page for one command.
2. **Full-program mode** — `--help --man` generates a complete man page covering ALL registered commands (including downstream business commands injected via GroupedModuleGroup). This mode does not occupy a subcommand name and is the recommended approach for downstream projects.

The system shall provide `build_program_man_page()` and `configure_man_help()` as public API functions so downstream projects can integrate man page generation with a single function call.

**Actors:** Developer, AI Agent

**Preconditions:**
- The Registry is loaded (for module-specific man pages).

**Main Flow (single-command mode):**
1. The user invokes `apcore-cli man exec` or `apcore-cli man list`.
2. The system shall generate a man page in roff format for the specified command.
3. The man page shall include: NAME, SYNOPSIS, DESCRIPTION, OPTIONS, EXIT CODES, SEE ALSO.
4. The system shall write the roff output to stdout.
5. The user may pipe to `man -l -` for immediate display.

**Main Flow (full-program mode):**
1. The downstream project calls `configure_man_help(program, prog_name, version, description)` during CLI setup.
2. This adds a hidden `--man` global option to the CLI.
3. When the user invokes `{prog_name} --help --man`, the system pre-parses `--man` from argv before the CLI framework processes help.
4. The system calls `build_program_man_page()` which iterates all visible commands (including downstream business commands), generating a complete roff man page with: TH, NAME, SYNOPSIS, DESCRIPTION, GLOBAL OPTIONS, COMMANDS (with nested subcommands and their options), ENVIRONMENT, EXIT CODES, SEE ALSO.
5. The system writes the roff output to stdout and exits.
6. Hidden options (from `--all-options` mode) are excluded from the man page by default.

**Alternative Flows:**

- **AF-1: Unknown Command (single-command mode).** If the specified command does not exist, the system shall write to stderr: `Error: Unknown command '{command}'.` and exit with code 2.
- **AF-2: `--man` without `--help`.** The `--man` flag is accepted but has no effect on command execution.
- **AF-3: `--help --man --all-options`.** The man page includes all options (including built-in options that are normally hidden).

**Postconditions:**
- A man page in roff format is written to stdout.

**Acceptance Criteria:**

- **AC-1:** Given the user runs `apcore-cli man exec`, then stdout shall contain roff-formatted text including `.TH` (title heading) and the command name.
- **AC-2:** Given the user runs `apcore-cli man exec | man -l -`, then the man page shall render correctly in the terminal.
- **AC-3:** Given a downstream project calls `configure_man_help(program, "myapp", "1.0.0")`, when the user runs `myapp --help --man`, then stdout shall contain a complete roff man page covering all registered commands.
- **AC-4:** Given a downstream project has registered module commands via `GroupedModuleGroup`, when `--help --man` is invoked, then the man page shall include all module commands and their schema-derived options.
- **AC-5:** Given `--help --man` is invoked, then the man page shall NOT include the `--man` option itself in the GLOBAL OPTIONS section (it is hidden).

---

### 5.7 Init Command (INIT)

#### FR-INIT-001: Module Scaffolding via `init module` Subcommand

| Field | Value |
|-------|-------|
| **ID** | FR-INIT-001 |
| **Title** | Module Scaffolding via `init module` Subcommand |
| **Priority** | P1 |
| **Priority Rationale** | Accelerates developer onboarding and eliminates boilerplate. Not blocking for core execution features but significantly reduces time-to-first-module. |
| **Source** | Feature Spec FE-10 FR-10-01; Tech Design v2.0 §8.12 |

**Description:** The system shall provide an `apcore-cli init module <module_id>` subcommand that generates a boilerplate module file from a template. The `init` group is registered as a `click.Group` on the root CLI, with `module` as a subcommand. Generated files shall embed the provided `module_id` and `--description` value. The command shall exit with code 0 on success and code 1 if the output directory is not writable.

**Actors:** Developer

**Preconditions:**
- `apcore-cli` is installed and available on the system PATH.
- The output directory is writable (or defaults to the style-dependent directory).

**Main Flow:**
1. The user invokes `apcore-cli init module <module_id> [--style STYLE] [--dir DIR] [--description TEXT]`.
2. The system shall split `module_id` on the last `.` to derive `(prefix, func_name)`. If no `.` is present, `func_name` equals `module_id` and no group constant is emitted.
3. The system shall select the output directory based on `--style` if `--dir` is not specified: `extensions/` for `decorator`, `commands/` for `convention`, `bindings/` for `binding`.
4. The system shall render the appropriate template for the chosen style and write the generated file(s) to the output directory.
5. The system shall exit with code 0.

**Alternative Flows:**

- **AF-1: Output Directory Not Writable.** If the target directory cannot be written to, the system shall propagate the OS-level `PermissionError` via Click and exit with code 1.
- **AF-2: Binding Style with Existing Companion File.** If the companion source file already exists under `commands/`, the system shall skip overwriting it and only create the `.binding.yaml` file.

**Postconditions:**
- One or more boilerplate files are written to the output directory.
- Exit code is 0 on success.

**Acceptance Criteria:**

- **AC-1:** Given `apcore-cli init module ops.deploy --style decorator`, then `extensions/ops_deploy.py` shall be created containing a `@module`-decorated function with `id="ops.deploy"`.
- **AC-2:** Given `apcore-cli init module ops.deploy --style convention`, then `commands/ops.py` shall be created containing `CLI_GROUP = "ops"` and a `deploy()` function.
- **AC-3:** Given `apcore-cli init module ops.deploy --style binding`, then `bindings/ops_deploy.binding.yaml` and `commands/ops.py` shall both be created.
- **AC-4:** Given `apcore-cli init module standalone` (no dot in ID), then the generated file shall define `func_name = "standalone"` with no `CLI_GROUP` constant.
- **AC-5:** Given `--dir /tmp/custom`, then the generated file shall be written under `/tmp/custom/` rather than the style default directory.

---

#### FR-INIT-002: Style Selection (`--style {decorator,convention,binding}`)

| Field | Value |
|-------|-------|
| **ID** | FR-INIT-002 |
| **Title** | Style Selection Flag |
| **Priority** | P1 |
| **Priority Rationale** | The three styles map to the three apcore module authoring patterns. All are required for full feature parity. |
| **Source** | Feature Spec FE-10 FR-10-02, FR-10-03, FR-10-04 |

**Description:** The `init module` subcommand shall accept a `--style` flag with values `decorator`, `convention`, and `binding`. The default value is `convention`. Each style produces a distinct file structure and template: `decorator` generates a `@module`-decorated function in `extensions/`; `convention` generates a plain function with metadata constants in `commands/`; `binding` generates a `.binding.yaml` file in `bindings/` and a companion source file in `commands/`.

**Actors:** Developer

**Preconditions:**
- `apcore-cli init module <module_id>` is invoked.

**Main Flow:**
1. The user provides `--style decorator`, `--style convention`, or `--style binding`.
2. The system selects the appropriate template and default output directory for the chosen style.
3. The system renders the template with the resolved `module_id`, `func_name`, `prefix`, and `--description` values.
4. The system writes the output file(s) and exits with code 0.

**Alternative Flows:**

- **AF-1: Invalid Style Value.** If the user provides a `--style` value other than `decorator`, `convention`, or `binding`, Click shall reject the input and display a usage error. Exit code 2.

**Postconditions:**
- Generated file(s) use the correct template for the chosen style.

**Acceptance Criteria:**

- **AC-1:** Given `--style decorator`, then the generated Python file shall contain `from apcore import module` and the `@module(id=..., description=...)` decorator.
- **AC-2:** Given `--style convention`, then the generated Python file shall contain the `CLI_GROUP` constant (when `module_id` contains a `.`) and a function matching `func_name`.
- **AC-3:** Given `--style binding`, then a `.binding.yaml` file shall be created containing `module_id`, `target`, `description`, and `auto_schema: true` fields.
- **AC-4:** Given `--style invalid_value`, then Click shall output a usage error referencing the valid choices and exit with code 2.

---

#### FR-INIT-003: `--dir` Output Path Override

| Field | Value |
|-------|-------|
| **ID** | FR-INIT-003 |
| **Title** | Output Directory Override |
| **Priority** | P1 |
| **Priority Rationale** | Non-standard project layouts require flexible output path control. |
| **Source** | Feature Spec FE-10 FR-10-05 |

**Description:** The `init module` subcommand shall accept a `--dir` flag that overrides the style-dependent default output directory. When `--dir` is provided, the generated file(s) shall be written to the specified directory regardless of the `--style` value. The system shall not perform path-traversal validation beyond standard OS permissions — directory writes to absolute paths outside the project are permitted.

**Actors:** Developer

**Preconditions:**
- `apcore-cli init module <module_id> --dir <path>` is invoked.

**Main Flow:**
1. The user provides `--dir <path>`.
2. The system uses `<path>` as the output directory instead of the style default.
3. The system writes the generated file to `<path>/<derived_filename>` and exits with code 0.

**Postconditions:**
- File is written to the user-specified directory.

**Acceptance Criteria:**

- **AC-1:** Given `--dir /tmp/custom` with any style, then the generated file shall be created under `/tmp/custom/` instead of the style-default directory.
- **AC-2:** Given `--dir` pointing to a non-writable path, then the command shall exit with code 1 and propagate the OS permission error.

---

#### FR-INIT-004: `--description` Template Injection

| Field | Value |
|-------|-------|
| **ID** | FR-INIT-004 |
| **Title** | Description Text Injection |
| **Priority** | P1 |
| **Priority Rationale** | Eliminates manual edit of the TODO placeholder, reducing time-to-working-module. |
| **Source** | Feature Spec FE-10 FR-10-06 |

**Description:** The `init module` subcommand shall accept a `--description` flag (short: `-d`) whose value is embedded in the generated file in place of the default `"TODO: add description"` placeholder. The description shall appear in the module decorator argument, the binding YAML `description` field, or the convention file's metadata constant, as appropriate for the chosen style.

**Actors:** Developer

**Preconditions:**
- `apcore-cli init module <module_id> --description "My module"` is invoked.

**Main Flow:**
1. The user provides `--description "My description text"`.
2. The system renders the template with the provided description in the appropriate location.
3. The generated file is written and the command exits with code 0.

**Postconditions:**
- The generated file contains the user-supplied description string.

**Acceptance Criteria:**

- **AC-1:** Given `--description "Processes payment transactions"`, then the generated file shall contain `"Processes payment transactions"` in the module decorator or YAML `description` field.
- **AC-2:** Given no `--description` flag, then the generated file shall contain the placeholder `"TODO: add description"`.

---

#### FR-INIT-005: `--commands-dir` on `create_cli()` with ConventionScanner

| Field | Value |
|-------|-------|
| **ID** | FR-INIT-005 |
| **Title** | Convention Module Auto-Discovery via `--commands-dir` |
| **Priority** | P1 |
| **Priority Rationale** | Enables zero-configuration workflow: scaffold a module with `init`, then run the CLI immediately. |
| **Source** | Feature Spec FE-10 FR-10-07, FR-10-08 |

**Description:** The `create_cli()` function shall accept an optional `commands_dir` parameter. When provided, the system shall attempt to import `ConventionScanner` from `apcore_toolkit.convention_scanner` and scan the given directory for convention-style modules. Discovered modules shall be registered in the Registry via `RegistryWriter`. The `--commands-dir` value shall also be exposed as a CLI option so users can pass it at invocation time. If `apcore-toolkit` is not installed, the system shall log a WARNING and continue without convention module scanning.

**Actors:** Developer

**Preconditions:**
- `create_cli(commands_dir=<path>)` is called, or `apcore-cli --commands-dir <path>` is invoked.

**Main Flow:**
1. `create_cli()` receives a non-None `commands_dir` value.
2. The system imports `ConventionScanner` and `RegistryWriter` from `apcore_toolkit`.
3. The system calls `ConventionScanner().scan(commands_dir)` to discover modules.
4. If modules are found, the system writes them to the Registry via `RegistryWriter` and logs the count at INFO level.
5. The CLI continues initialization with the additional modules available.

**Alternative Flows:**

- **AF-1: `apcore-toolkit` Not Installed.** The `ImportError` is caught, a WARNING is logged (`"apcore-toolkit not installed — convention module scanning unavailable"`), and the CLI continues without convention modules. No traceback is shown.
- **AF-2: `commands_dir` Does Not Exist.** `ConventionScanner` returns an empty list; a WARNING is logged. The CLI starts normally with zero convention modules.

**Postconditions:**
- Convention modules from `commands_dir` are registered and available as CLI commands (when `apcore-toolkit` is installed).
- Exit code is 0 on success.

**Acceptance Criteria:**

- **AC-1:** Given `create_cli(commands_dir="commands/")` with `apcore-toolkit` installed and modules present, then convention modules from `commands/` shall appear in the `list` output.
- **AC-2:** Given `create_cli(commands_dir="commands/")` without `apcore-toolkit` installed, then a WARNING shall be logged and the CLI shall start normally with zero convention modules.
- **AC-3:** Given `apcore-cli --commands-dir ./cmds list`, then convention modules from `./cmds` shall appear in the module list.

### 5.8 Usability Enhancements (USB)

#### FR-USB-001: Dry-Run / Validate Mode

| Field | Value |
|-------|-------|
| **ID** | FR-USB-001 |
| **Title** | Dry-Run Preflight Mode |
| **Priority** | P0 |
| **Priority Rationale** | Enables safe validation of module inputs without executing destructive operations. Critical for AI agents and CI pipelines. |
| **Source** | Feature Spec FE-11 FR-11-01; Tech Design v2.0 |

**Description:** The system shall provide a `--dry-run` flag on module execution commands and a standalone `validate <module_id>` subcommand. Both shall invoke `Executor.validate()` and display the resulting `PreflightResult` without executing the module. TTY output uses human-readable check symbols (`✓`, `✗`, `○`, `⚠`); non-TTY output uses JSON. Exit code reflects the first failed check (0 = all passed, 44 = module not found, 45 = schema invalid, 77 = ACL denied, 1 = other failure).

**Acceptance Criteria:**

- **AC-1:** Given `--dry-run` with valid inputs, then exit code shall be 0 and all checks shall show `✓` in TTY output; no module execution side effects shall occur.
- **AC-2:** Given `--dry-run` with an invalid schema field, then exit code shall be 45 and the schema check shall show `✗` with the field error detail.

---

#### FR-USB-002: System Management Commands

| Field | Value |
|-------|-------|
| **ID** | FR-USB-002 |
| **Title** | System Management Commands (`health`, `usage`, `enable`, `disable`, `reload`, `config`) |
| **Priority** | P0 |
| **Priority Rationale** | Exposes apcore's built-in system modules via CLI. Required for runtime governance and operational visibility. |
| **Source** | Feature Spec FE-11 FR-11-02 |

**Description:** The system shall register six built-in system management commands (`health`, `usage`, `enable`, `disable`, `reload`, `config`) that delegate to corresponding `system.*` apcore modules via `Executor.call()`. Commands are registered only when system modules are available (detected via `executor.validate("system.health.summary", {})`). The `enable`, `disable`, `reload`, and `config set` commands require a `--reason` flag and trigger the standard approval gate.

**Acceptance Criteria:**

- **AC-1:** Given system modules are registered, when the user runs `apcore-cli health`, then a health summary table shall be displayed showing module statuses; when system modules are absent, the command shall not appear in `--help`.
- **AC-2:** Given `apcore-cli disable <module_id>` without `--reason`, then Click shall reject the invocation with a missing required option error and exit with code 2.

---

#### FR-USB-003: Enhanced Error Output

| Field | Value |
|-------|-------|
| **ID** | FR-USB-003 |
| **Title** | Enhanced Error Output with `ai_guidance`, `suggestion`, `retryable` |
| **Priority** | P0 |
| **Priority Rationale** | Actionable error messages reduce developer friction and improve AI agent self-correction. |
| **Source** | Feature Spec FE-11 FR-11-03 |

**Description:** The system shall extract and display `ai_guidance`, `suggestion`, and `retryable` fields from apcore error responses when present. TTY output shall render these fields in a structured format below the error message. Non-TTY output shall include these fields in the JSON error object. The `retryable` field shall be indicated in TTY output with a retry hint.

**Acceptance Criteria:**

- **AC-1:** Given an apcore error response containing `ai_guidance` and `suggestion` fields, then TTY output shall display both fields below the primary error message.
- **AC-2:** Given a non-TTY execution context, then the JSON error object shall include `ai_guidance`, `suggestion`, and `retryable` fields when present in the apcore error.

---

#### FR-USB-004: Pipeline Trace (`--trace`)

| Field | Value |
|-------|-------|
| **ID** | FR-USB-004 |
| **Title** | Execution Pipeline Trace |
| **Priority** | P1 |
| **Priority Rationale** | Provides observability into the execution pipeline for debugging and performance analysis. |
| **Source** | Feature Spec FE-11 FR-11-04 |

**Description:** The system shall provide a `--trace` flag on module execution commands. When active, the system shall invoke `Executor.call_with_trace()` and display the execution pipeline steps with timing, middleware names, and status for each step. TTY output renders a timeline view; non-TTY output emits a JSON trace object alongside the module result.

**Acceptance Criteria:**

- **AC-1:** Given `--trace` is provided, then the output shall include each pipeline step name, its execution duration in milliseconds, and pass/fail status.
- **AC-2:** Given `--trace --format json`, then the output shall include a `"trace"` key in the JSON result containing an array of pipeline step objects.

---

#### FR-USB-005: Approval Handler Protocol

| Field | Value |
|-------|-------|
| **ID** | FR-USB-005 |
| **Title** | Standard `ApprovalHandler` Protocol Integration |
| **Priority** | P1 |
| **Priority Rationale** | Aligns apcore-cli's approval mechanism with the standard apcore `ApprovalHandler` protocol, enabling cross-language consistency. |
| **Source** | Feature Spec FE-11 FR-11-05 |

**Description:** The system shall implement a `CliApprovalHandler` class conforming to the apcore `ApprovalHandler` async protocol. The handler shall implement `request_approval(module_id, inputs, context)` and `on_approved(module_id, inputs, context)` / `on_denied(module_id, inputs, context)` methods. The `CliApprovalHandler` shall be passed to the `Executor` during `create_cli()` initialization, replacing any ad-hoc approval prompting logic.

**Acceptance Criteria:**

- **AC-1:** Given a module with `requires_approval=True` and an interactive TTY, when the user runs the module without `--yes`, then `CliApprovalHandler.request_approval()` shall be called and a confirmation prompt shall be displayed.
- **AC-2:** Given `--yes` flag or `APCORE_CLI_AUTO_APPROVE=1`, then `CliApprovalHandler` shall skip the interactive prompt and proceed with execution.

---

#### FR-USB-006: Stream Output (`--stream`)

| Field | Value |
|-------|-------|
| **ID** | FR-USB-006 |
| **Title** | Streaming Output via `Executor.stream()` |
| **Priority** | P1 |
| **Priority Rationale** | Required for long-running modules that emit incremental results (e.g., LLM inference, batch processors). |
| **Source** | Feature Spec FE-11 FR-11-06 |

**Description:** The system shall provide a `--stream` flag on module execution commands. When active, the system shall invoke `Executor.stream(module_id, inputs)` instead of `Executor.call()` and write each yielded chunk to stdout as a newline-delimited JSON record. Streaming is only available for modules that declare `annotations.supports_streaming = true`; for non-streaming modules, the system shall emit a warning and fall back to standard `call()`.

> **Implementation note:** `Executor.stream()` is not yet present in the verified public API of any SDK (apcore-cli-python v0.9.0, apcore-cli-typescript v0.9.0, apcore-cli-rust v0.9.0). This is a NOT-YET-IMPLEMENTED requirement — the `--stream` flag is accepted by the CLI but behaves as a no-op fallback until the apcore Executor exposes `stream()`.

**Acceptance Criteria:**

- **AC-1:** Given `--stream` on a streaming-capable module, then each result chunk shall be written to stdout as a newline-terminated JSON object as it is yielded.
- **AC-2:** Given `--stream` on a non-streaming module, then a warning shall be emitted to stderr and the command shall fall back to standard `Executor.call()` behavior.

---

#### FR-USB-007: Enhanced List Filters

| Field | Value |
|-------|-------|
| **ID** | FR-USB-007 |
| **Title** | Enhanced `list` Command Filters |
| **Priority** | P1 |
| **Priority Rationale** | Large module registries (50+ modules) are not navigable without search and filter capabilities. |
| **Source** | Feature Spec FE-11 FR-11-07 |

**Description:** The `list` command shall be enhanced with additional filter and sort flags: `--search <text>` (substring match on ID and description), `--status {enabled,disabled,all}`, `--annotation <key>[=<value>]`, `--sort {id,name,status}`, `--reverse`, `--deprecated` (show only deprecated modules), and `--deps <module_id>` (show modules that depend on the given ID). All filters are composable.

**Acceptance Criteria:**

- **AC-1:** Given `apcore-cli list --search "email"`, then only modules whose ID or description contains "email" (case-insensitive) shall be shown.
- **AC-2:** Given `apcore-cli list --status disabled`, then only modules with `status: disabled` shall appear in the output.

---

#### FR-USB-008: Execution Strategy Selection (`--strategy`)

| Field | Value |
|-------|-------|
| **ID** | FR-USB-008 |
| **Title** | Execution Strategy Override |
| **Priority** | P2 |
| **Priority Rationale** | Advanced feature for power users who need to control the execution pipeline. Not required for standard use cases. |
| **Source** | Feature Spec FE-11 FR-11-08 |

**Description:** The system shall provide a `--strategy <name>` flag on module execution commands. When provided, the system shall resolve the named `ExecutionStrategy` from the Executor's registered strategies and use it for the invocation instead of the default strategy. If the named strategy does not exist, the system shall exit with code 2 and display available strategy names.

**Acceptance Criteria:**

- **AC-1:** Given `--strategy minimal` when a `minimal` strategy is registered, then the module shall be executed using the `minimal` strategy rather than the default pipeline.
- **AC-2:** Given `--strategy nonexistent`, then the system shall exit with code 2 and list available strategy names in the error message.

---

#### FR-USB-009: Extended Output Format (`csv`/`yaml`/`jsonl`/`--fields`)

| Field | Value |
|-------|-------|
| **ID** | FR-USB-009 |
| **Title** | Extended Output Format Options |
| **Priority** | P2 |
| **Priority Rationale** | Enables integration with downstream tools (spreadsheets, YAML configs, log aggregators). Low adoption risk — purely additive. |
| **Source** | Feature Spec FE-11 FR-11-09 |

**Description:** The `--format` flag shall be extended to support `csv`, `yaml`, and `jsonl` in addition to the existing `table` and `json` values. A `--fields <dotpath>[,<dotpath>...]` flag shall be added to select specific output fields from the module result or list output. `--fields` is composable with all `--format` values.

**Acceptance Criteria:**

- **AC-1:** Given `--format jsonl`, then each output record shall be emitted as a single-line JSON object terminated by a newline, with one record per line.
- **AC-2:** Given `--fields id,description --format csv`, then `list` output shall contain only the `id` and `description` columns in CSV format.

---

#### FR-USB-010: Multi-Level Grouping

| Field | Value |
|-------|-------|
| **ID** | FR-USB-010 |
| **Title** | Multi-Level Nested Command Grouping |
| **Priority** | P2 |
| **Priority Rationale** | Required for projects with deeply namespaced module IDs (e.g., `admin.users.create`). Improves `--help` legibility at scale. |
| **Source** | Feature Spec FE-11 FR-11-10 |

**Description:** The `GroupedModuleGroup` shall support multi-level nesting of Click groups for module IDs with two or more dot-separated namespace segments (e.g., `admin.users.create` maps to `admin > users > create`). Each intermediate namespace segment that contains sub-commands shall be rendered as a nested `click.Group`. Single-segment module IDs and the existing single-level grouping behavior shall be preserved unchanged.

**Acceptance Criteria:**

- **AC-1:** Given modules `admin.users.create` and `admin.users.delete`, then `apcore-cli admin --help` shall show a `users` subgroup, and `apcore-cli admin users --help` shall show `create` and `delete` commands.
- **AC-2:** Given a mix of single-level (`math.add`) and multi-level (`admin.users.create`) module IDs, then both grouping styles shall coexist without conflict in the same CLI.

---

#### FR-USB-011: Extra Commands Extension Point

| Field | Value |
|-------|-------|
| **ID** | FR-USB-011 |
| **Title** | `extra_commands` Injection on `create_cli()` |
| **Priority** | P2 |
| **Priority Rationale** | Enables framework integrations (Django, FastAPI) to inject custom commands without forking the CLI entrypoint. |
| **Source** | Feature Spec FE-11 FR-11-11 |

**Description:** The `create_cli()` function shall accept an `extra_commands` parameter (type: `list[click.BaseCommand] | None`). When provided, each command in the list shall be added to the root CLI group after the `apcli` group and module commands are registered. Commands in `extra_commands` shall not conflict with `RESERVED_GROUP_NAMES = frozenset({"apcli"})`; if a name collision is detected, the system shall raise a `ValueError` at CLI construction time with the conflicting command name. (Pre-FE-13 wording referred to `BUILTIN_COMMANDS`; that constant was retired in v0.7.0 — only `apcli` is now root-reserved.)

**Acceptance Criteria:**

- **AC-1:** Given `create_cli(extra_commands=[my_custom_cmd])`, then `my_custom_cmd` shall appear in `apcore-cli --help` output alongside the `apcli` group entry.
- **AC-2:** Given `extra_commands` containing a command whose name is `"apcli"` (the only entry in `RESERVED_GROUP_NAMES`), then `create_cli()` shall raise `ValueError` with the conflicting name before the CLI is returned.
- **AC-3:** Given `extra_commands` containing a command whose name collides with a non-shim live root command (e.g., a registered user module ID), then `create_cli()` shall raise `ValueError` with the conflicting name.

---

### 5.9 Module Exposure Filtering (EXP)

#### FR-EXP-001: `ExposureFilter` Class

| Field | Value |
|-------|-------|
| **ID** | FR-EXP-001 |
| **Title** | `ExposureFilter` Class |
| **Priority** | P1 |
| **Priority Rationale** | Foundation of the feature. All other EXP requirements depend on this class existing. Required for framework integrations where registry size makes full CLI exposure impractical. |
| **Source** | Feature Spec FE-12 FR-12-01, FR-12-09, FR-12-10; Tech Design v2.0 §8.2 |

**Description:** The system shall provide an `ExposureFilter` class in `apcore_cli/exposure.py` with three attributes: `mode` (one of `"all"`, `"include"`, `"exclude"`), `include` (list of glob-pattern strings), and `exclude` (list of glob-pattern strings). The class shall expose `is_exposed(module_id: str) -> bool` and `filter_modules(module_ids: list[str]) -> tuple[list[str], list[str]]` methods. When `mode="all"`, `is_exposed()` shall return `True` for all module IDs. When `mode="include"` and the include list is empty, `is_exposed()` shall return `False` for all modules. When `mode="exclude"` and the exclude list is empty, `is_exposed()` shall return `True` for all modules.

**Acceptance Criteria:**

- **AC-1:** Given `ExposureFilter(mode="all")`, then `is_exposed("any.module")` shall return `True` for any module ID regardless of include/exclude lists.
- **AC-2:** Given `ExposureFilter(mode="include", include=[])`, then `is_exposed("any.module")` shall return `False` (empty include list means zero modules exposed).
- **AC-3:** Given `ExposureFilter(mode="exclude", exclude=[])`, then `is_exposed("any.module")` shall return `True` (empty exclude list means all modules exposed).
- **AC-4:** Given `filter_modules(["a.x", "b.y"])` with `mode="all"`, then the method shall return `(["a.x", "b.y"], [])` — no hidden modules.

---

#### FR-EXP-002: Exposure Config Loading

| Field | Value |
|-------|-------|
| **ID** | FR-EXP-002 |
| **Title** | Exposure Config Loading from `apcore.yaml` |
| **Priority** | P1 |
| **Priority Rationale** | Declarative project-level config is the primary integration path for most users. |
| **Source** | Feature Spec FE-12 FR-12-03, FR-12-08 |

**Description:** The `ExposureFilter` class shall provide a `from_config(cls, config: dict) -> ExposureFilter` classmethod that reads the `expose` section from a parsed `apcore.yaml` config dict. It shall validate `mode` against the allowed values (`"all"`, `"include"`, `"exclude"`), raising `click.BadParameter` for invalid values (exit code 2). Invalid `include`/`exclude` types (non-list) shall trigger a WARNING log and default to an empty list. The config loading follows 4-tier precedence: `create_cli(expose=...)` > `APCORE_CLI_EXPOSE_MODE` env var > `apcore.yaml` > default `mode: all`.

**Acceptance Criteria:**

- **AC-1:** Given a valid `apcore.yaml` with `expose: {mode: include, include: [admin.*]}`, then `ExposureFilter.from_config(config)` shall return an `ExposureFilter` with `mode="include"` and `include_patterns=["admin.*"]`.
- **AC-2:** Given `expose: {mode: whitelist}` (invalid mode), then `from_config()` shall raise `click.BadParameter` with a message identifying the invalid value and the allowed values, causing exit code 2.
- **AC-3:** Given `expose: {mode: include, include: "not-a-list"}`, then `from_config()` shall log a WARNING and treat the include list as empty.

---

#### FR-EXP-003: Glob Pattern Matching

| Field | Value |
|-------|-------|
| **ID** | FR-EXP-003 |
| **Title** | Glob Pattern Matching Semantics |
| **Priority** | P1 |
| **Priority Rationale** | Namespace-level control (e.g., `admin.*`) is the primary use case. Without correct glob semantics, filtering is unusable for real-world registries. |
| **Source** | Feature Spec FE-12 FR-12-04 |

**Description:** Pattern matching on module IDs shall follow the semantics: `*` matches any characters within a single dot-separated segment (does not cross `.`); `**` matches any characters across any number of dot-separated segments. Patterns are anchored to the full module ID. Regex compilation shall occur once at `ExposureFilter.__init__` time and be cached. Empty patterns shall be skipped with a WARNING log. Duplicate patterns shall be deduplicated silently.

**Acceptance Criteria:**

- **AC-1:** Given pattern `admin.*`, then `is_exposed("admin.users")` shall return `True` and `is_exposed("admin.users.list")` shall return `False` (single-segment wildcard does not cross dots).
- **AC-2:** Given pattern `admin.**`, then both `is_exposed("admin.users")` and `is_exposed("admin.users.list")` shall return `True` (multi-segment wildcard crosses dots).
- **AC-3:** Given pattern `*.get`, then `is_exposed("product.get")` shall return `True` and `is_exposed("product.get.all")` shall return `False`.
- **AC-4:** Given pattern `system.health` (exact), then `is_exposed("system.health")` shall return `True` and `is_exposed("system.health.check")` shall return `False`.

---

#### FR-EXP-004: `create_cli` `expose=` Parameter

| Field | Value |
|-------|-------|
| **ID** | FR-EXP-004 |
| **Title** | `expose=` Parameter on `create_cli()` |
| **Priority** | P1 |
| **Priority Rationale** | Programmatic integration path for framework adapters (Django, FastAPI). Highest-precedence tier in the 4-tier config chain. |
| **Source** | Feature Spec FE-12 FR-12-07, FR-12-08 |

**Description:** The `create_cli()` function shall accept an `expose` parameter of type `dict | ExposureFilter | None`. When an `ExposureFilter` instance is passed, it shall be used directly. When a `dict` is passed, it shall be interpreted as the `expose` section of an `apcore.yaml` config and passed to `ExposureFilter.from_config()`. When `None` is passed, the system shall fall through to lower-precedence tiers (`APCORE_CLI_EXPOSE_MODE` env var, then `apcore.yaml`, then default `mode: all`). The resulting `ExposureFilter` shall be passed to `GroupedModuleGroup` at CLI construction time and applied during `_build_group_map()`.

**Acceptance Criteria:**

- **AC-1:** Given `create_cli(expose=ExposureFilter(mode="include", include=["admin.*"]))`, then only `admin.*` modules shall appear in `--help` and `apcore-cli admin --help`; all other module groups shall be hidden.
- **AC-2:** Given `create_cli(expose={"mode": "exclude", "exclude": ["webhooks.*"]})`, then all modules except `webhooks.*` shall appear in `--help`.
- **AC-3:** Given `create_cli(expose=None)` with no `apcore.yaml` and no env var, then all modules shall be exposed (default `mode: all` behavior is preserved).

---

#### FR-EXP-005: `list --exposure` Filter Flag

| Field | Value |
|-------|-------|
| **ID** | FR-EXP-005 |
| **Title** | `list --exposure` Filter Flag |
| **Priority** | P1 |
| **Priority Rationale** | Operators need visibility into which modules are hidden for debugging and auditing exposure configurations. |
| **Source** | Feature Spec FE-12 FR-12-05, FR-12-06 |

**Description:** The `list` command shall gain an `--exposure` option accepting `exposed` (default), `hidden`, or `all`. When `--exposure exposed` (default), only modules for which `ExposureFilter.is_exposed()` returns `True` shall be shown. When `--exposure hidden`, only hidden modules shall be shown. When `--exposure all`, all modules shall be shown with an additional "Exposure" column in table output indicating `✓` (exposed) or `—` (hidden). This filter is independent of and composable with existing `--status`, `--tag`, and `--search` filters.

**Acceptance Criteria:**

- **AC-1:** Given `mode: include, include: [admin.*]` and modules `admin.users`, `admin.config`, `webhooks.stripe`, then `apcore-cli list` (default `--exposure exposed`) shall show only `admin.users` and `admin.config`.
- **AC-2:** Given the same config, `apcore-cli list --exposure hidden` shall show only `webhooks.stripe`.
- **AC-3:** Given `apcore-cli list --exposure all`, then all modules shall appear and the output table shall include an "Exposure" column with `✓` for exposed modules and `—` for hidden modules.

---

### 5.10 CRUD Matrix

| Entity | Create | Read | Update | Delete |
|--------|--------|------|--------|--------|
| **Module Registry** | FR-DISP-003 (discover/load from directory) | FR-DISP-002 (lookup module), FR-DISC-001 (list), FR-DISC-003 (describe) | N/A (read-only in CLI; managed externally) | N/A (managed externally) |
| **Module Definition** | N/A (defined by module authors) | FR-DISP-002 (resolve for exec), FR-DISC-003 (describe), FR-SCHEMA-001 (parse schema) | N/A (immutable at runtime) | N/A |
| **CLI Configuration** | FR-SEC-002 (initial config write) | FR-DISP-005 (precedence resolution), FR-SEC-001 (API key retrieval) | FR-SEC-002 (re-encryption) | N/A (manual file management) |
| **Audit Log** | FR-SEC-003 (append entry on each execution) | External tooling (not in CLI scope) | N/A (append-only) | N/A (retention managed externally) |
| **Approval Record** | FR-APPR-002 (user approves/denies), FR-APPR-004 (bypass logged) | FR-SEC-003 (recorded in audit log) | N/A (immutable) | N/A |

---

## 6. Non-Functional Requirements

### 6.1 Performance

#### NFR-PERF-001: CLI Startup Time

| Field | Value |
|-------|-------|
| **ID** | NFR-PERF-001 |
| **Title** | CLI Startup Time |
| **Metric** | Wall-clock time from process start to help text output |
| **Target** | < 100 milliseconds |
| **Measurement Method** | `time apcore-cli --help` averaged over 10 runs with a warm filesystem cache, using a Registry containing 100 modules |
| **Threshold Rationale** | Tech Design v2.0 section 3.2 Goal: "Minimal overhead (<100ms) for one-shot executions." ideas/draft.md NFR-001: "Execution overhead < 100ms." Sub-100ms ensures CLI feels instantaneous to developers and does not bottleneck agent workflows. |

#### NFR-PERF-002: Module Execution Overhead

| Field | Value |
|-------|-------|
| **ID** | NFR-PERF-002 |
| **Title** | Module Execution Overhead |
| **Metric** | Wall-clock time spent in apcore-cli adapter code (excluding module execution time) |
| **Target** | < 50 milliseconds |
| **Measurement Method** | Execute a no-op module (returns immediately) via `apcore-cli exec noop` and measure total time minus module execution time. Average over 10 runs. |
| **Threshold Rationale** | The adapter layer must not introduce perceptible latency. At 50ms overhead + <100ms startup, total CLI overhead stays under 150ms, ensuring the CLI remains faster than JSON-RPC/MCP alternatives for simple invocations (as stated in ideas/draft.md section 2: "Token Bloat" problem). |

#### NFR-PERF-003: Registry Capacity

| Field | Value |
|-------|-------|
| **ID** | NFR-PERF-003 |
| **Title** | Registry Capacity |
| **Metric** | Number of modules the Registry can index without degradation of startup time target (NFR-PERF-001) |
| **Target** | 1,000 modules |
| **Measurement Method** | Generate a synthetic Registry with 1,000 module definitions and measure startup time. Verify it remains < 100ms. |
| **Threshold Rationale** | Feature Spec FE-01 section 3.1: "Should handle up to 1,000 modules." This provides 10x headroom over the 100-module baseline, accommodating growth of enterprise module libraries. |

---

### 6.2 Security

#### NFR-SEC-001: Credential Storage

| Field | Value |
|-------|-------|
| **ID** | NFR-SEC-001 |
| **Title** | No Plaintext Credential Storage |
| **Metric** | Number of plaintext secrets found in configuration files |
| **Target** | 0 |
| **Measurement Method** | Automated scan of `apcore.yaml` and `~/.apcore-cli/` for known secret patterns (API keys, tokens) using a secret detection tool. Run as part of CI. |
| **Threshold Rationale** | User requirements specify "encrypted config." OWASP guidelines prohibit plaintext credential storage. Any plaintext secret is a critical vulnerability. |

#### NFR-SEC-002: Input Validation

| Field | Value |
|-------|-------|
| **ID** | NFR-SEC-002 |
| **Title** | Input Validation Against JSON Schema |
| **Metric** | Percentage of input fields validated against JSON Schema before module execution |
| **Target** | 100% |
| **Measurement Method** | Unit tests verifying that for every module execution path, `jsonschema.validate()` is invoked before `Executor.call()`. Code review of the execution pipeline. |
| **Threshold Rationale** | Unvalidated input bypasses the module's declared contract, potentially causing injection vulnerabilities or unexpected behavior. PROTOCOL_SPEC requires schema validation at each adapter boundary. |

#### NFR-SEC-003: STDIN Buffer Limit

| Field | Value |
|-------|-------|
| **ID** | NFR-SEC-003 |
| **Title** | STDIN Buffer Limit |
| **Metric** | Maximum STDIN size accepted without `--large-input` flag |
| **Target** | 10 MB |
| **Measurement Method** | Pipe input exceeding 10 MB to `apcore-cli exec module --input -` without `--large-input` and verify rejection with exit code 2. |
| **Threshold Rationale** | Tech Design v2.0 section 8.2: "STDIN > 10MB: Reject with GENERAL_INVALID_INPUT." The 10MB limit prevents out-of-memory conditions in memory-constrained environments while accommodating typical JSON payloads (99th percentile < 1MB). |

---

### 6.3 Reliability

#### NFR-REL-001: Deterministic Exit Codes

| Field | Value |
|-------|-------|
| **ID** | NFR-REL-001 |
| **Title** | Deterministic Exit Codes |
| **Metric** | Percentage of error conditions that produce the documented exit code |
| **Target** | 100% |
| **Measurement Method** | Integration test suite covering all error taxonomy entries (Tech Design section 8.1). Each test triggers a specific error condition and asserts the exit code. |
| **Threshold Rationale** | PROTOCOL_SPEC section 8 mandates specific error codes. AI agents rely on exit codes for programmatic error handling. Any non-deterministic exit code breaks agent workflows. |

#### NFR-REL-002: Graceful Degradation

| Field | Value |
|-------|-------|
| **ID** | NFR-REL-002 |
| **Title** | Graceful Degradation on Missing Extensions |
| **Metric** | System crash count when extensions directory is missing or corrupt |
| **Target** | 0 crashes (all failures produce a user-facing error message and a non-zero exit code) |
| **Measurement Method** | Test with: (1) missing directory, (2) empty directory, (3) directory with corrupt module files, (4) directory with permission denied. Verify no Python tracebacks reach stdout/stderr. |
| **Threshold Rationale** | Feature Spec FE-01 requires graceful handling of missing/corrupt extensions. Python tracebacks exposed to users degrade trust and are unusable by AI agents. |

---

### 6.4 Maintainability

#### NFR-MNT-001: Test Coverage

| Field | Value |
|-------|-------|
| **ID** | NFR-MNT-001 |
| **Title** | Unit Test Coverage |
| **Metric** | Line coverage percentage measured by `coverage.py` |
| **Target** | >= 80% |
| **Measurement Method** | Run `pytest --cov=apcore_cli --cov-report=term-missing` and verify the total coverage percentage. |
| **Threshold Rationale** | Industry standard for production CLI tools. 80% provides confidence in regression detection while allowing pragmatic exclusion of trivial code paths (main entry point, platform-specific branches). |

#### NFR-MNT-002: Structured Logging

| Field | Value |
|-------|-------|
| **ID** | NFR-MNT-002 |
| **Title** | Structured Logging |
| **Metric** | Percentage of components with dedicated logger namespace and configurable verbosity |
| **Target** | 100% of components (dispatcher, schema, approval, discovery) |
| **Measurement Method** | Code review verifying each component uses `logging.getLogger("apcore_cli.{component}")` and respects `APCORE_CLI_LOGGING_LEVEL` (CLI-specific, highest priority) / `APCORE_LOGGING_LEVEL` (global fallback) / `--log-level`. |
| **Threshold Rationale** | Tech Design v2.0 section 8.4 specifies structured logging with namespace `apcore_cli`. Consistent logging enables debugging in production and integration with centralized log aggregation. |

---

### 6.5 Portability

#### NFR-PRT-001: OS Compatibility

| Field | Value |
|-------|-------|
| **ID** | NFR-PRT-001 |
| **Title** | Cross-OS Compatibility |
| **Metric** | Number of supported operating systems where the full test suite passes |
| **Target** | 3 (Linux, macOS, Windows) with Python >= 3.11 |
| **Measurement Method** | CI matrix running the full test suite on Ubuntu latest, macOS latest, and Windows latest with Python 3.11 and 3.12. |
| **Threshold Rationale** | User requirements specify "Linux, macOS, Windows (Python 3.11+)." Python's cross-platform nature enables this with minimal platform-specific code. Requires 3.11+ to align with apcore >= 0.17.1. |

#### NFR-PRT-002: Terminal Compatibility

| Field | Value |
|-------|-------|
| **ID** | NFR-PRT-002 |
| **Title** | Terminal Emulator Compatibility |
| **Metric** | Number of terminal emulators where output renders correctly (no broken characters, correct colors) |
| **Target** | Correct rendering in: Terminal.app, iTerm2, GNOME Terminal, Windows Terminal, VS Code integrated terminal |
| **Measurement Method** | Manual verification of `list` (table output) and `describe` (syntax-highlighted JSON) in each terminal emulator. |
| **Threshold Rationale** | The `rich` library provides cross-terminal compatibility, but edge cases exist with color support and Unicode. The five listed terminals cover the majority of developer usage. |

---

### 6.6 Usability

#### NFR-USB-001: Help Text Quality

| Field | Value |
|-------|-------|
| **ID** | NFR-USB-001 |
| **Title** | Help Text Coverage |
| **Metric** | Percentage of commands and flags with non-empty help text |
| **Target** | 100% |
| **Measurement Method** | Parse the output of `apcore-cli --help`, `apcore-cli exec --help`, `apcore-cli list --help`, `apcore-cli describe --help` and verify every listed command/flag has associated descriptive text (non-empty string after the flag name). |
| **Threshold Rationale** | User requirements specify "every command and flag has descriptive help text." Help text is the primary discoverability mechanism for both human developers and AI agents introspecting CLI capabilities. |

#### NFR-USB-002: Error Message Quality

| Field | Value |
|-------|-------|
| **ID** | NFR-USB-002 |
| **Title** | Actionable Error Messages |
| **Metric** | Percentage of error messages that include (1) what went wrong, (2) a suggested fix or next step |
| **Target** | 100% of error exits (codes 1, 2, 44, 45, 46, 47, 48, 77, 130) |
| **Measurement Method** | Review all error message strings in the codebase. Each must contain a description of the problem AND at least one of: a CLI flag to try, an env var to set, a config to check, or an action to take. |
| **Threshold Rationale** | User requirements specify "user-friendly error messages with suggested fixes." Error messages without actionable guidance increase support burden and degrade developer experience. |

---

## 7. Data Requirements

### 7.1 Data Model

```
+-------------------+       +---------------------+       +------------------+
| Module Registry   |       | Module Definition   |       | Input Schema     |
|-------------------|       |---------------------|       |------------------|
| extensions_path   |1    *| canonical_id (PK)   |1    1| properties       |
| module_count      +-------+ description          +-------+ required[]       |
| discovered_at     |       | tags[]               |       | $defs            |
+-------------------+       | annotations          |       | type             |
                            |   .requires_approval |       +------------------+
                            |   .readonly          |
                            |   .destructive       |       +------------------+
                            |   .idempotent        |       | Output Schema    |
                            | extension_metadata   |1    1|------------------|
                            | input_schema (FK)    +-------+ properties       |
                            | output_schema (FK)   |       | type             |
                            | enabled              |       +------------------+
                            +---------------------+

+-------------------+       +---------------------+
| CLI Configuration |       | Audit Log Entry     |
|-------------------|       |---------------------|
| extensions_root   |       | timestamp (ISO 8601)|
| auth.api_key      |       | user                |
| logging_level     |       | module_id           |
| sandbox_enabled   |       | input_hash (SHA-256)|
+-------------------+       | status              |
                            | exit_code           |
                            | duration_ms         |
                            +---------------------+
```

### 7.2 Data Dictionary

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `canonical_id` | String | Pattern: `^[a-z][a-z0-9_]*(\.[a-z][a-z0-9_]*)*$`. Max 192 chars (PROTOCOL_SPEC §2.7). | Unique dot-separated identifier for a module. |
| `description` | String | Max 4,096 characters. | Human-readable description of the module's purpose. |
| `tags` | Array of String | Each tag: `^[a-z][a-z0-9_-]*$`. Max 32 tags per module. | Categorization labels for discovery filtering. |
| `requires_approval` | Boolean | Default: `false`. | Whether the module requires HITL approval before execution. |
| `readonly` | Boolean | Default: `false`. | Whether the module only reads data (no side effects). |
| `destructive` | Boolean | Default: `false`. | Whether the module performs destructive operations. |
| `idempotent` | Boolean | Default: `false`. | Whether repeated execution produces the same result. |
| `extensions_root` | String (path) | Valid filesystem path. Max 4,096 characters. | Path to the apcore extensions directory. |
| `auth.api_key` | String (secret) | Stored encrypted (see FR-SEC-002). Max 512 characters. | API key for remote registry authentication. |
| `logging_level` | Enum | `DEBUG`, `INFO`, `WARNING`, `ERROR`. Default: `WARNING`. | Verbosity level for structured logging. |
| `timestamp` | String (ISO 8601) | Format: `YYYY-MM-DDTHH:MM:SS.fffZ`. | When the audit event occurred. |
| `input_hash` | String (hex) | 64 characters (SHA-256). | Hash of the serialized input for auditability without storing raw input. |
| `status` | Enum | `success`, `error`. | Outcome of the module execution. |
| `exit_code` | Integer | 0-255. | Unix exit code of the CLI process. |
| `duration_ms` | Integer | >= 0. | Module execution duration in milliseconds. |

---

## 8. External Interface Requirements

### 8.1 User Interfaces

The system's user interface is the command-line terminal. All interaction occurs through:

- **Command invocation**: `apcore-cli <subcommand> [options] [arguments]`.
- **Help display**: Formatted text output via Click's built-in help rendering and `rich` formatting.
- **Table output**: Module listing rendered as formatted tables via `rich.table.Table`.
- **JSON output**: Machine-readable JSON written to stdout for non-TTY consumers.
- **Approval prompts**: Interactive `[y/N]` prompts rendered via `click.confirm()` in TTY mode.
- **Error output**: Error messages written to stderr, never to stdout (to preserve pipe integrity).

### 8.2 Hardware Interfaces

Not applicable. The system runs as a software process and does not interface directly with hardware.

### 8.3 Software Interfaces

| Interface | Version | Protocol | Direction | Description |
|-----------|---------|----------|-----------|-------------|
| `apcore` | >= 0.17.1 | Python API | Outbound | Registry discovery, module metadata retrieval, Executor invocation, Config Bus namespace registration, Execution Pipeline Strategy. The system imports `apcore.Registry`, `apcore.Executor`, `apcore.Config`, and the error hierarchy. |
| `click` | >= 8.1 | Python API | Internal | CLI command tree construction, argument parsing, help generation, interactive prompts. |
| `jsonschema` | >= 4.20 | Python API | Internal | Input validation against module JSON Schema definitions. |
| `rich` | >= 13.0 | Python API | Internal | Terminal output formatting including tables, syntax highlighting, and styled text. |
| `keyring` | >= 24.0 | Python API | Outbound | OS-native credential storage for encrypted configuration (FR-SEC-002). |

### 8.4 Communication Interfaces

| Channel | Direction | Format | Description |
|---------|-----------|--------|-------------|
| **stdin** | Input | UTF-8 JSON | Accepts piped JSON input when `--input -` is specified. Buffer limit: 10 MB (configurable). |
| **stdout** | Output | UTF-8 text or JSON | Module execution results, table displays, JSON output, man pages, completion scripts. |
| **stderr** | Output | UTF-8 text | Error messages, warnings, diagnostic output. Never mixed with stdout to preserve pipe integrity. |
| **Exit codes** | Output | Integer (0-255) | Deterministic exit codes per PROTOCOL_SPEC section 8. See Tech Design section 8.1 for full taxonomy. |
| **Environment variables** | Input | String key-value | Configuration via `APCORE_*` prefixed variables (see Tech Design section 8.3). |
| **Filesystem** | Input/Output | Files | Extensions directory (input), configuration file (input/output), audit log (output). |

---

## 9. Requirements Source

This SRS was developed in **standalone mode** without an upstream Product Requirements Document (PRD). Requirements were derived from the following sources:

| Source | Location | Contribution |
|--------|----------|-------------|
| **Tech Design v2.0** | `docs/tech-design.md` | Architecture decisions (ADR-01 through ADR-03), error taxonomy, environment variable conventions, component design, performance targets. |
| **Feature Specs** | `docs/features/*.md` | Functional requirements for Core Dispatcher (FE-01), Schema Parser (FE-02), Approval Gate (FE-03), and Discovery (FE-04). Boundary values and verification criteria. |
| **User Clarification Interview** | N/A (inline) | Security requirements (auth, encryption, audit, sandbox), shell integration (completion, man pages), user characteristics (developer + AI agent), performance constraints. |
| **ideas/draft.md** | `ideas/draft.md` | Original requirement IDs (FR-001 through FR-004, NFR-001 through NFR-002), problem statement, and validation rationale. |

**Traceability Note:** *To establish full traceability from business objectives through product requirements to software requirements, consider running `/spec-forge:prd` first, then re-running `/spec-forge:srs`.*

---

## 10. Appendix

### 10.1 Open Questions

| ID | Question | Status | Owner |
|----|----------|--------|-------|
| OQ-01 | What is the maximum module execution timeout? Tech Design mentions `MODULE_TIMEOUT` but no default value is specified. | Open | Engineering |
| OQ-02 | Should shell completion (FR-SHELL-001) use Click's built-in completion or a custom implementation for dynamic module ID completion? | Open | Engineering |
| OQ-03 | What is the retention policy for audit log entries (FR-SEC-003)? Should the CLI provide a `log rotate` command? | Open | Product |
| OQ-04 | Should the STDIN buffer limit (10 MB) be configurable via `apcore.yaml` in addition to the `--large-input` flag? | Open | Engineering |
| OQ-05 | What sandbox mechanism shall be used for FR-SEC-004 on Windows, where Unix-style process isolation is not directly available? | Open | Engineering |
| OQ-06 | Should `apcore-cli exec` output default to `table` (TTY) / `json` (non-TTY) like Discovery, or always output raw module result? | Open | Product |
| OQ-07 | How should `anyOf`/`oneOf` schema composition be presented to users when multiple mutually exclusive property sets exist? | Open | Engineering |

### 10.2 Glossary

| Term | Definition |
|------|-----------|
| **Adapter** | A component that translates between the apcore module interface and a specific communication channel (CLI, MCP, A2A). |
| **Cold Start** | The first invocation of the CLI process, with no cached state from previous runs. |
| **Exit Code** | A numeric value (0-255) returned by a Unix process to indicate success (0) or a specific error category. |
| **JSON Lines** | A format where each line is a valid JSON object, commonly used for append-only log files. |
| **Kebab-case** | A naming convention where words are separated by hyphens (e.g., `input-file`). |
| **Middleware Chain** | A sequence of processing steps (ACL, observability, approval) that wrap module execution in the apcore Executor. |
| **Roff** | The formatting language used by Unix `man` pages. |
| **Snake_case** | A naming convention where words are separated by underscores (e.g., `input_file`). |

### 10.3 Change Request Log

| CR ID | Date | Requestor | Description | Status |
|-------|------|-----------|-------------|--------|
| _(none)_ | — | — | — | — |
