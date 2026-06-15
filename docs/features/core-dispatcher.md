---
description: "Core Dispatcher (FE-01) providing the LazyModuleGroup/GroupedModuleGroup Click entry point that lazily discovers Registry modules, handles STDIN JSON, and delegates to the apcore Executor."
---

# Feature Spec: Core Dispatcher

**Feature ID**: FE-01
**Status**: Ready for Implementation
**Priority**: P0
**Parent**: [Tech Design v2.0](../tech-design.md) Section 8.2
**SRS Requirements**: FR-DISP-001, FR-DISP-002, FR-DISP-003, FR-DISP-004, FR-DISP-006
**Related Features**: [Grouped Commands (FE-09)](grouped-commands.md)

---

## 1. Description

The Core Dispatcher is the primary entry point for `apcore-cli`. It provides the `LazyModuleGroup` Click Group subclass (and its `GroupedModuleGroup` subclass) that lazily discovers modules from the apcore Registry, routes commands to built-in subcommands or dynamically generated module commands, handles STDIN JSON input, and delegates execution to the apcore Executor.

**v2.0 update:** In Tech Design v2.0, `create_cli()` uses `GroupedModuleGroup` (extends `LazyModuleGroup`) as the root Click group. This organizes modules into nested subcommand groups by namespace prefix. The `LazyModuleGroup` base class remains unchanged. See [Grouped Commands (FE-09)](grouped-commands.md) for the full grouped commands specification.

---

## 2. Requirements Traceability

| Req ID | SRS Ref | Description |
|--------|---------|-------------|
| FR-01-01 | FR-DISP-001 | Base command entry point with `--help` and `--version`. Program name is resolved dynamically (see FR-01-06). |
| FR-01-02 | FR-DISP-002 | `exec <module_id>` subcommand routing and execution via Executor. |
| FR-01-03 | FR-DISP-003 | Extensions directory loading from configurable path. |
| FR-01-04 | FR-DISP-001 AF-3 | Version flag: `{prog_name} --version` prints `{prog_name}, version X.Y.Z`. |
| FR-01-05 | FR-DISP-004 | STDIN JSON input when `--input -` is specified. |
| FR-01-06 | FR-DISP-006 | CLI program name resolved from `argv[0]` basename; explicit `prog_name` parameter overrides. |
| FR-01-07 | FR-DISP-007 | All-options help mode: built-in options hidden by default, shown with `--all-options`. |
| FR-01-08 | FR-DISP-008 | Pre-populated registry: `create_cli()` accepts optional `registry` and `executor` parameters, skipping filesystem discovery when provided. |

---

## 3. Module Path

`apcore_cli/cli.py`

---

## 4. Implementation Details

## Contract: LazyModuleGroup.__init__

### Inputs
- registry: Registry, required — apcore Registry instance for module discovery.
- executor: Executor, required — apcore Executor instance for module invocation.
- help_text_max_length: int, optional — Maximum characters before help text is truncated. Default: `1000`.
- extensions_root: str | None, optional — Path to extensions directory, forwarded to sandbox runner.
- **kwargs: Any — passed through to `click.Group.__init__`.

### Errors
- (none raised by constructor itself)

### Returns
- On success: LazyModuleGroup instance

### Properties
- async: false
- thread_safe: false (builds alias map lazily; not safe for concurrent use)
- pure: false (may read from registry on first list_commands call)

---

## Contract: LazyModuleGroup.list_commands

### Inputs
- ctx: click.Context, required — Click context (unused internally, required by Click interface).

### Errors
- (none raised — catches registry exceptions internally and logs WARNING)

### Returns
- On success: list[str] — sorted, deduplicated list of command names (registered commands + module aliases)

### Properties
- async: false
- thread_safe: false
- pure: false (may call registry.list() on first invocation)

---

## Contract: build_module_command

### Inputs
- module_def: ModuleDescriptor, required — apcore module descriptor with `input_schema`, `description`, and display overlay.
- executor: Executor, required — Executor used in the command callback.
- help_text_max_length: int, optional — Maximum help text length. Default: `1000`.
- cmd_name: str | None, optional — CLI command name override. Defaults to display alias or module_id.
- extensions_root: str | None, optional — Passed to Sandbox for subprocess isolation.
  validates: schema properties must not collide with reserved option names
  reject_with: SystemExit(2) — on reserved option name collision

### Errors
- SystemExit(2) — schema property name collides with reserved CLI option (`input`, `yes`, `large_input`, etc.)

### Returns
- On success: click.Command — fully configured Click command with schema-generated options and built-in options

### Properties
- async: false
- thread_safe: false (uses module-level globals `_all_options_help` and `_docs_url`)
- pure: false (reads module-level globals)

---

## Contract: collect_input

### Inputs
- stdin_flag: str | None, required — Input source: `None` or `""` = CLI only; `"-"` = stdin; any other string = file path.
  validates: STDIN must be ≤10 MB unless `large_input=True`; STDIN/file content must be valid JSON object
  reject_with: SystemExit(2) — on oversized input, invalid JSON, or non-object JSON
- cli_kwargs: dict[str, Any], required — Parsed CLI keyword arguments; None values are stripped before merge.
- large_input: bool, optional — When True, bypasses the 10 MB size guard. Default: `False`.

### Errors
- SystemExit(2) — STDIN/file exceeds 10 MB and `large_input` is False
- SystemExit(2) — STDIN/file contains invalid JSON
- SystemExit(2) — STDIN/file JSON is not an object (dict)
- SystemExit(2) — file path does not exist or cannot be read

### Returns
- On success: dict[str, Any] — merged input where CLI flag values override STDIN/file values for duplicate keys

### Properties
- async: false
- thread_safe: false (reads sys.stdin)
- pure: false (may read stdin or filesystem)

---

## Contract: validate_module_id

### Inputs
- module_id: str, required — Module ID string to validate.
  validates: length ≤192 chars; matches regex `^[a-z][a-z0-9_]*(\.[a-z][a-z0-9_]*)*$`
  reject_with: SystemExit(2)

### Errors
- SystemExit(2) — length exceeds 192 characters
- SystemExit(2) — does not match the canonical ID regex

### Returns
- On success: None (returns silently)
- On failure: raises SystemExit(2)

### Properties
- async: false
- thread_safe: true
- pure: false (calls sys.exit on failure)

---

## Contract: create_cli

### Inputs
- extensions_dir: str | None, optional — Override extensions directory path.
- prog_name: str | None, optional — Override program name shown in help. Defaults to `os.path.basename(sys.argv[0])`.
- commands_dir: str | None, optional — Convention-based commands directory (requires apcore-toolkit).
- binding_path: str | None, optional — Path to binding.yaml for display overlay.
- registry: Registry | None, optional — Pre-populated registry; skips filesystem discovery when provided.
- executor: Executor | None, optional — Pre-built executor; requires `registry` to also be set.
  validates: `executor` without `registry` is rejected
  reject_with: ValueError("executor requires registry — pass both or neither")
- extra_commands: list | None, optional — Additional Click commands to add to root.
  validates: names must not be in `RESERVED_GROUP_NAMES`
  reject_with: ValueError
- app: APCore | None, optional — Unified APCore client; mutually exclusive with `registry`/`executor`.
  reject_with: ValueError("app is mutually exclusive with registry/executor")
- expose: dict | ExposureFilter | None, optional — Exposure filter config (FE-12).
- apcli: bool | dict | ApcliGroup | None, optional — Built-in apcli group config (FE-13).
- allowed_prefixes: list[str] | None, optional — Allowlist for binding/convention target resolution. Cross-SDK status: Python (`allowed_prefixes`) and TypeScript (`allowedPrefixes`, threaded through `createCli` → `applyToolkitIntegration` → `loadBindingDisplayOverlay`) both expose this option. **Rust does not yet have this option**; the parity gap is tracked separately and is NOT a "Python-only" feature any more.

### Errors
- ValueError — `executor` provided without `registry`
- ValueError — `app` provided alongside `registry` or `executor`
- ValueError — extra command name collides with reserved name or existing non-shim command
- SystemExit(47) — extensions directory not found or unreadable
- SystemExit(2) — invalid `apcli` type

### Returns
- On success: click.Group — fully assembled CLI group ready for invocation

### Properties
- async: false
- thread_safe: false (initialises module-level globals)
- pure: false (reads filesystem, env vars, config files, initializes logging)

---

## Contract: main

### Inputs
- prog_name: str | None, optional — Override program name shown in help and error output. Default: `None` — the basename of `sys.argv[0]` is used.

### Errors
- SystemExit — `main` is the process entry-point shim; every error path translates to a `SystemExit` whose code matches the canonical exit-code table in §6 below. Notably:
  - SystemExit(0) — normal completion.
  - SystemExit(1) — module execution error.
  - SystemExit(2) — invalid CLI input, missing argument, or invalid module ID format (propagated from `validate_module_id` / `collect_input` / Click).
  - SystemExit(44) — module not found, disabled, or load error.
  - SystemExit(45) — schema validation failure.
  - SystemExit(46) — approval denied / timed out / no TTY.
  - SystemExit(47) — extensions directory missing or unreadable; configuration / decryption error.
  - SystemExit(48) — schema contains a circular `$ref`.
  - SystemExit(77) — ACL denied or authentication failure.
  - SystemExit(130) — SIGINT / Ctrl-C.

### Returns
- On success: None — process exits via Click's `standalone_mode=True` with exit code 0; the function does not return normally on error paths because Click translates exceptions into `SystemExit` before control returns.
- Stdin: not consumed by `main` itself; STDIN handling is delegated to `collect_input` per command (only commands that opt in via `--input -` or `--input <file>` read STDIN).

### Properties
- async: false
- thread_safe: false (entry-point shim — initialises module-level globals via `create_cli`)
- pure: false (reads `sys.argv`, env vars, filesystem; writes stdout/stderr; calls `sys.exit`)

> Note: `main` is intentionally a thin wrapper around `create_cli`. New behavioral knobs SHOULD be added to `create_cli` and surfaced here only when an entry-point semantic differs (e.g., `--extensions-dir` pre-extraction). See §4.6 for the procedural detail and `Contract: create_cli` above for the full parameter surface.

---

### 4.1 Class: `LazyModuleGroup`

**File**: `apcore_cli/cli.py`

```python
class LazyModuleGroup(click.Group):
    """Custom Click Group that lazily loads apcore modules as subcommands.

    Base class for GroupedModuleGroup. Handles display overlay alias resolution,
    descriptor caching, and lazy command building.
    """

    def __init__(self, registry: Registry, executor: Executor,
                 help_text_max_length: int = 1000, **kwargs):
        super().__init__(**kwargs)
        self._registry = registry
        self._executor = executor
        self._help_text_max_length = help_text_max_length
        self._module_cache: dict[str, click.Command] = {}
        self._alias_map: dict[str, str] = {}           # CLI alias -> module_id
        self._alias_map_built: bool = False
        self._descriptor_cache: dict[str, Any] = {}    # module_id -> descriptor
```

**Method: `list_commands(ctx) -> list[str]`**

Logic steps (post-FE-13, v0.7+):
1. The only reserved root entry is `apcli` (`RESERVED_GROUP_NAMES = frozenset({"apcli"})`). The 14-entry `BUILTIN_COMMANDS` constant was retired in v0.7.0; the canonical built-in subcommand set now lives in `apcore_cli.builtin_group.APCLI_SUBCOMMAND_NAMES` and is reachable as `<cli> apcli <sub>`. Do NOT splice it back at the root.
2. If `_alias_map_built` is False: call `_build_alias_map()`.
3. Compute `root_names = set(self._alias_map.keys())`. If the apcli group is registered AND visible per `ApcliGroup.is_group_visible()`, add `"apcli"`.
4. Return `sorted(root_names)`.

Edge cases:
- Registry returns empty list: return `["apcli"]` (when visible) or `[]`.
- Registry raises exception during `list()`: catch, log WARNING, return `["apcli"]` (when visible) or `[]`.

> Cross-ref: [Tech Design §8.2.1](../tech-design.md), [features/builtin-group.md §4.9](builtin-group.md).

**Method: `_build_alias_map()`**

Logic steps:
- Iterates `self._registry.list()` to get all module definitions.
- For each module definition: reads `mod.metadata["display"]["cli"]["alias"]` if present, else uses `mod.canonical_id` (or `mod.module_id`).
- Stores alias → module_id in `self._alias_map`.
- Stores module_id → module_def in `self._descriptor_cache`.
- Sets `self._alias_map_built = True` only on success (inside try block).
- On failure: logs WARNING, does not set the flag (allows retry on transient registry errors).

**Method: `get_command(ctx, cmd_name) -> click.Command | None`**

Logic steps:
1. Check `self.commands` dict for built-in commands. If found, return it.
2. Ensure alias map is built (call `_build_alias_map()` if not).
3. Look up `module_id = self._alias_map.get(cmd_name)`. If not found, return `None`.
4. Check `self._descriptor_cache.get(module_id)` — if found, use cached descriptor.
5. Else call `self._registry.get_definition(module_id)`.
6. Call `build_module_command(module_def, self._executor)`.
7. Store result in `self._module_cache[cmd_name]`.
8. Return the command.

### 4.2 Function: `build_module_command`

**Signature**: `build_module_command(module_def: ModuleDefinition, executor: Executor, help_text_max_length: int = 1000) -> click.Command`

Logic steps:
1. Get `input_schema` from `module_def.input_schema`.
2. Call `resolve_refs(input_schema, max_depth=32)` to inline all `$ref` references.
3. Call `schema_to_click_options(resolved_schema)` to generate Click options.
4. Create a Click command with:
   - `name`: `metadata["display"]["cli"]["alias"]` if present, else `module_def.canonical_id`
   - `help`: `metadata["display"]["cli"]["description"]` if present, else `module_def.description`
   - Built-in options: `--input`, `--yes`, `--large-input`, `--format` (hidden from `--help` by default; shown when `--all-options` is passed), `--sandbox` (always hidden — not yet implemented)
   - Global option: `--all-options` (controls built-in option visibility in help output)
5. The command callback:
   a. Call `collect_input(stdin_input, kwargs, large_input)` to merge STDIN + CLI flags.
   b. Call `jsonschema.validate(merged, resolved_schema)`. On failure: exit 45 with validation error detail.
   c. Call `check_approval(module_def, auto_approve)`. On denial/timeout: exit 46.
   d. Record `audit_start = time.monotonic()`.
   e. Call `executor.call(module_def.canonical_id, merged)`.
   f. Compute `duration_ms = int((time.monotonic() - audit_start) * 1000)`.
   g. Call `audit_logger.log_execution(module_id, merged, "success", 0, duration_ms)`.
   h. Call `format_output(result, ctx)` to write to stdout.
   i. Exit with code 0.
6. On `Executor` error: catch, write error to stderr, audit log with `"error"`, exit with mapped error code.
7. On `KeyboardInterrupt`: write "Execution cancelled." to stderr, exit 130.
8. Append all generated Click options to `command.params`.
9. Return the command.

### 4.3 Function: `collect_input`

**Signature**: `collect_input(stdin_flag: str | None, cli_kwargs: dict, large_input: bool) -> dict`

Logic steps:
1. If `stdin_flag` is None or empty string: return `cli_kwargs` (with None values removed).
2. If `stdin_flag == "-"`:
   a. Read `sys.stdin.read()` into `raw`.
   b. Check `len(raw.encode('utf-8'))`:
      - If `> 10_485_760` and `large_input is False`: exit code 2, message "STDIN input exceeds 10MB limit. Use --large-input to override."
   c. If `raw` is empty (0 bytes): set `stdin_data = {}`.
   d. Else: parse `json.loads(raw)` into `stdin_data`.
      - On `json.JSONDecodeError`: exit code 2, message "STDIN does not contain valid JSON: {e.msg}."
   e. If `stdin_data` is not a `dict`: exit code 2, message "STDIN JSON must be an object, got {type(stdin_data).__name__}."
   f. Merge: `result = {**stdin_data, **cli_kwargs_non_none}` (CLI flags override STDIN for duplicate keys).
3. Return `result`.

### 4.4 Function: `validate_module_id`

**Signature**: `validate_module_id(module_id: str) -> None`

Logic steps:
1. Check `len(module_id) > 192`: exit code 2, message "Invalid module ID format: '{module_id}'. Maximum length is 192 characters."
2. Check `re.fullmatch(r'^[a-z][a-z0-9_]*(\.[a-z][a-z0-9_]*)*$', module_id)`: if None, exit code 2, message "Invalid module ID format: '{module_id}'."

### 4.5 Function: `create_cli`

**Canonical signature, logic steps, and cross-language equivalents** are owned by [Tech Design v2.0 §8.2.7](../tech-design.md). That section is the **single source of truth** — refer there for the current 13-parameter form (including FE-13 `apcli=`, Issue #18 `version=`, Issue #19 `description=`, and the v0.7.0 additions `app=`, `allowed_prefixes=`).

This feature spec intentionally does NOT duplicate the signature, logic steps, or cross-language equivalents block — duplicating them previously caused this section to drift two minor versions behind reality (the historical 8-parameter form was retained here long after `apcli`, `version`, `description`, `app`, and `allowed_prefixes` had landed in code). To fix this once, the per-parameter contract lives in §3 above (Contract block) and the procedural steps live in tech-design.md.

**File:** `apcore_cli/factory.py` (split out of `apcore_cli/__main__.py` in v0.5.1+).

**Purpose:** Factory function that assembles and returns the fully configured Click group. Separating construction from invocation enables library users to embed the CLI in a larger application and override settings (including the program name) without touching entry-point code.

**Pre-populated registry support (v0.5.1):** When a `registry` parameter is provided, filesystem discovery is skipped entirely. This enables frameworks that register modules at runtime (e.g. apflow's bridge, which populates the registry programmatically via `create_apflow_registry()`) to generate CLI commands from their existing registry without requiring an extensions directory on disk.

### 4.6 Function: `main`

**Signature**: `main(prog_name: str | None = None) -> None`

**File**: `apcore_cli/__main__.py`

**Purpose**: Standard entry-point shim. Pre-extracts `--extensions-dir` from `argv` before Click parses (required because the registry must be instantiated before Click runs), then delegates to `create_cli()`.

Logic steps:
1. Call `_extract_extensions_dir(sys.argv[1:])` to extract `--extensions-dir` value from raw `argv` before Click processes it.
2. Call `create_cli(extensions_dir=ext_dir, prog_name=prog_name)` to obtain the configured CLI group.
3. Invoke `cli(standalone_mode=True)`.

**Library integration patterns (cross-language normative reference):**

| Integration Pattern | How to specify program name |
|--------------------|-----------------------------|
| Default entry point | No action needed. `argv[0]` basename is used automatically. |
| Custom entry point in `pyproject.toml` | Add `myproject = "apcore_cli.__main__:main"` under `[project.scripts]`. The name `myproject` is used automatically. |
| Programmatic override | Call `main(prog_name="myproject")` or `create_cli(prog_name="myproject")`. |
| Other languages (Go, TypeScript, Rust) | Pass the desired name to the equivalent `createCLI(progName: string)` factory; fall back to `os.Args[0]` basename when not provided. |

---

## 5. Boundary Values & Limits

| Parameter | Minimum | Maximum | Default | SRS Reference |
|-----------|---------|---------|---------|---------------|
| Module ID length | 1 char | 192 chars | — | FR-DISP-002 (PROTOCOL_SPEC §2.7) |
| STDIN buffer | 0 bytes | 10 MB (configurable) | 10 MB | FR-DISP-004 |
| Registry module count | 0 | 1,000 (design target) | — | NFR-PERF-003 |
| Startup time | — | 100 ms | — | NFR-PERF-001 |
| Adapter overhead | — | 50 ms | — | NFR-PERF-002 |

---

## 6. Error Handling

| Condition | Exit Code | Error Message | SRS Reference |
|-----------|-----------|---------------|---------------|
| Invalid module ID format | 2 | "Error: Invalid module ID format: '{id}'." | FR-DISP-002 AF-2 |
| Module not found | 44 | "Error: Module '{id}' not found in registry." | FR-DISP-002 AF-1 |
| Module disabled | 44 | "Error: Module '{id}' is disabled." | FR-DISP-002 AF-5 |
| Module load error | 44 | "Error: Module '{id}' failed to load: {detail}." | FR-DISP-002 AF-7 |
| Schema validation failure | 45 | "Error: Validation failed for '{prop}': {constraint}." | FR-DISP-002 AF-3 |
| Module execution error | 1 | "Error: Module '{id}' execution failed: {detail}." | FR-DISP-002 AF-4 |
| STDIN exceeds buffer | 2 | "Error: STDIN input exceeds 10MB limit. Use --large-input to override." | FR-DISP-004 AF-1 |
| STDIN invalid JSON | 2 | "Error: STDIN does not contain valid JSON: {detail}." | FR-DISP-004 AF-2 |
| STDIN not object | 2 | "Error: STDIN JSON must be an object, got {type}." | FR-DISP-004 AF-3 |
| Extensions dir missing | 47 | "Error: Extensions directory not found: '{path}'. Set APCORE_EXTENSIONS_ROOT or verify the path." | FR-DISP-003 AF-1 |
| Extensions dir unreadable | 47 | "Error: Cannot read extensions directory: '{path}'. Check file permissions." | FR-DISP-003 AF-2 |
| ACL denied | 77 | "Error: Permission denied for module '{id}'." | FR-DISP-002 AF-8 |
| User interruption | 130 | "Execution cancelled." | FR-DISP-002 AF-6 |

---

## 7. Verification

| Test ID | Description | Expected Result |
|---------|-------------|-----------------|
| T-DISP-01 | Run `apcore-cli --help` with valid extensions | Output contains "exec", "list", "describe" and module IDs. Exit 0. |
| T-DISP-02 | Run `apcore-cli --version` | Output: "apcore-cli, version X.Y.Z". Exit 0. |
| T-DISP-03 | Run `apcore-cli exec non.existent` | stderr: "not found". Exit 44. |
| T-DISP-04 | Run `apcore-cli exec "INVALID!ID"` | stderr: "Invalid module ID format". Exit 2. |
| T-DISP-05 | Run `apcore-cli exec math.add --a 5 --b 10` | stdout: module result. Exit 0. |
| T-DISP-06 | Pipe `echo '{"a":5,"b":10}' \| apcore-cli exec math.add --input -` | Module receives `{a:5, b:10}`. Exit 0. |
| T-DISP-07 | Pipe `echo '{"a":5}' \| apcore-cli exec math.add --input - --a 99` | Module receives `{a:99}` (CLI flag overrides STDIN). |
| T-DISP-08 | Pipe 15MB JSON without `--large-input` | stderr: "exceeds 10MB limit". Exit 2. |
| T-DISP-09 | Pipe invalid JSON | stderr: "does not contain valid JSON". Exit 2. |
| T-DISP-10 | Set `APCORE_EXTENSIONS_ROOT=/tmp/test` | Modules loaded from `/tmp/test`. |
| T-DISP-11 | Extensions dir does not exist | stderr: "not found". Exit 47. |
| T-DISP-12 | Extensions dir with one corrupt module | Corrupt module skipped with WARNING. Valid modules available. |
| T-DISP-13 | `--extensions-dir` overrides `APCORE_EXTENSIONS_ROOT` | CLI flag path is used. |
| T-DISP-14 | Default entry point `apcore-cli --version` | Output: `apcore-cli, version X.Y.Z`. Exit 0. |
| T-DISP-15 | Downstream entry point `myproject --version` (installed as `myproject = "apcore_cli.__main__:main"`) | Output: `myproject, version X.Y.Z`. Exit 0. |
| T-DISP-16 | `create_cli(prog_name="custom-name")` invoked with `--help` | Help output contains `custom-name`. Does not contain `apcore-cli`. |
| T-DISP-17 | `create_cli(prog_name=None)` invoked when `argv[0]` is `pytest` | Help output contains `pytest` (argv[0] basename). Falls back gracefully. |
| T-DISP-18 | `create_cli(registry=mock_registry, executor=mock_executor)` | CLI created without filesystem access. No exit 47. Modules from registry available. |
| T-DISP-19 | `create_cli(registry=mock_registry)` (executor omitted) | Executor auto-built from provided registry. CLI created successfully. |
| T-DISP-20 | `create_cli(executor=mock_executor)` (registry omitted) | `ValueError` raised: "executor requires registry". |

---

## 8. Cross-SDK API surface

This appendix enumerates the public symbols that embedders may rely on, with each SDK's per-language name and parity status. All entries are valid for v0.8+ unless otherwise noted.

### 8.1 `register_*_command` factory family

All three SDKs export the per-subcommand registrar family (e.g. `register_list_command`, `register_describe_command`, `register_exec_command`, `register_validate_command`, `register_health_command`, `register_usage_command`, `register_enable_command`, `register_disable_command`, `register_reload_command`, `register_config_command`, `register_completion_command`, `register_pipeline_command`, `register_init_command`) plus the helper `configure_man_help`, as a public composition API for embedders building their own root command tree. See `features/builtin-group.md` §4.9 for the full split. Python parity with TS / Rust landed in v0.8.0 via `apcore_cli/__init__.py` re-exports (D1-001).

<!-- Audit D4-W1 (2026-05-12): Six discovery-family registrars listed above ship as public
     composition API in v0.9 without per-symbol Contract blocks. Mirrors the deferral pattern
     used for register_system_commands at usability-enhancements.md (D4-013). Full per-subcommand
     Contract split is tracked for v0.10. -->

#### 8.1.1 Per-subcommand Contract blocks — deferral

The six per-subcommand registrars listed in §8.1 (`register_list_command`, `register_describe_command`, `register_exec_command`, `register_validate_command`, `register_completion_command`, `register_pipeline_command`) ship as public composition API in v0.9. Their formal `## Contract:` blocks are deferred to v0.10 alongside the per-subcommand contract split tracked by [D4-013] (see `features/usability-enhancements.md` — the same deferral pattern applies to the six `register_system_commands`-fanout subcommands `health`, `usage`, `enable`, `disable`, `reload`, `config`). The parallel `register_init_command` (see `features/init-command.md` §`## Contract: register_init_command`) and `register_system_commands` (see `features/usability-enhancements.md` §`## Contract: register_system_commands`) DO carry full Contract blocks today and provide the canonical template that the six v0.9 registrars will follow in v0.10.

**Behavioral envelope (v0.9) — applies uniformly to all six registrars:**

- **Inputs:** `(parent_cmd, executor, registry, *, prog_name?, **kwargs)` — the same shape across all six. `parent_cmd` is the Click `Group` (Python) / Commander `Command` (TS) / `clap::Command` (Rust) the registrar attaches its subcommand to. `executor` and `registry` mirror the contracts of `create_cli` (see §6.1). `prog_name` and additional language-idiomatic keyword arguments are passed through to the underlying handler.
- **Errors:** each registrar MAY raise `ApcliGroupError` (or its language-equivalent — `ApcliGroupError` in Python/TS, `ApcliGroupError` variant in Rust's error enum) if `parent_cmd` is malformed (e.g., not a `Group`/`Command`, or already has a conflicting subcommand name). No other declared errors — argument validation for the *invocation* of the registered subcommand is owned by the underlying handler, not the registrar.
- **Returns:** void / `None` / `()` — the registrar attaches a subcommand to `parent_cmd` as a side effect and returns nothing. Embedders MUST NOT depend on a return value.
- **Properties:** `async = false`, `thread_safe = false` (attaches to a `parent_cmd` which is per-process; calling registrars concurrently against the same `parent_cmd` is undefined behavior), `pure = false` (mutates `parent_cmd`).

The underlying handlers (`list_cmd`, `describe_cmd`, `exec_cmd`, `validate_cmd`, `completion_cmd`, `pipeline_cmd`) carry their own `## Contract:` blocks in `features/discovery.md` and `features/cli-dispatcher.md`; the registrar wrappers inherit their input validation, error semantics, and exit-code mapping from those handlers unchanged.

### 8.2 Error-class table — `ModuleNotFoundError`

| Language | Class name | Module | Notes |
|----------|-----------|--------|-------|
| Python | `ModuleNotFoundError` | `apcore_cli.errors` | Previously `CliModuleNotFoundError` in v0.7.x — kept as a deprecated alias `CliModuleNotFoundError = ModuleNotFoundError` until v0.10.0 (D1-002). |
| TypeScript | `ModuleNotFoundError` | `errors.ts` | — |
| Rust | `ModuleNotFoundError` | `src/lib.rs` | — |

Maps to exit code 44.

### 8.3 Programmatic exit-code mapping (embedder helpers)

| Language | Public surface |
|----------|---------------|
| Python | `apcore_cli.EXIT_CODES` dict + `apcore_cli.exit_code_for_error(err)` helper (parity with TS `EXIT_CODES` / `exitCodeForError` and Rust `EXIT_*` constants in `src/lib.rs`). 24 `EXIT_*` constants exported from `apcore_cli/exit_codes.py`. (D1-003) |
| TypeScript | `EXIT_CODES` dict + `exitCodeForError(err)` helper exported from `errors.ts`. |
| Rust | 24 `EXIT_*` constants in `src/lib.rs`; `exit_code_for_error(&Error)` helper. |

### 8.4 `create_cli` / `createCli` `allowed_prefixes` / `allowedPrefixes`

| Language | Parameter | Status |
|----------|-----------|--------|
| Python | `allowed_prefixes: list[str] \| None` | Available since v0.7.0 (FE-12). |
| TypeScript | `allowedPrefixes?: string[]` (on `CreateCliOptions`) | Available since v0.8.0 (D1-006). Threaded through `createCli` → `applyToolkitIntegration` → `loadBindingDisplayOverlay`. |
| Rust | (not yet implemented) | Cross-SDK parity gap — tracked separately. The Rust source comment in `lib.rs` claiming this is "Python-only" is stale as of v0.8.0 and should be updated when Rust adds the option. |

### 8.5 Output formatter helpers

All three SDKs export the four formatter helpers — `resolve_format`, `format_module_list`, `format_module_detail`, `format_exec_result` — as of v0.8.0 (D1-007). See `features/output-formatter.md` for the per-helper Contract. Earlier "Python/TS only export `format_exec_result`" caveats are no longer accurate.

### 8.6 Sandbox builder API

All three SDKs expose the fluent builder methods `with_extensions_root` / `withExtensionsRoot` and `with_max_output_bytes` / `withMaxOutputBytes` on the `Sandbox` constructor as of v0.8.0 (D1-004). See `features/security.md` `Contract: Sandbox.__init__` for the canonical example.
