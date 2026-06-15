---
description: "Module Exposure Filtering (FE-12) with declarative expose config (all/include/exclude) and glob patterns controlling which discovered modules surface as CLI commands; a UX filter, not security."
---

# Feature Spec: Module Exposure Filtering

**Feature ID**: FE-12
**Status**: Ready for Implementation
**Priority**: P1
**Parent**: [Tech Design v2.0](../tech-design.md) Section 8.2
**SRS Requirements**: FR-DISP-001, FR-DISP-002, FR-DISC-001, NFR-USB-001
**Related Features**: [Core Dispatcher (FE-01)](core-dispatcher.md), [Grouped Commands (FE-09)](grouped-commands.md), [Discovery (FE-04)](discovery.md), [Config Resolver (FE-07)](config-resolver.md)

---

## 1. Description

The Module Exposure Filtering feature provides declarative control over which discovered modules are exposed as CLI commands. When `apcore-cli` is used with web framework integrations (e.g., FastAPI, Django), the scanner may discover 50+ API endpoints, but only a subset — admin operations, batch jobs, health checks — are suitable for CLI use. Without filtering, `--help` output becomes noisy, tab completion is unusable, and users risk invoking endpoints not designed for terminal interaction (webhooks, SSE streams, file uploads).

This feature adds an `expose` section to `apcore.yaml` (or `CliConfig` for programmatic usage) that supports three modes: `all` (default, current behavior), `include` (whitelist — only listed modules become CLI commands), and `exclude` (blacklist — all modules exposed except listed ones). Module ID patterns support glob-style wildcards for namespace-level control (e.g., `admin.*`, `webhooks.*`).

Filtering is applied at CLI command registration time, **after** discovery but **before** Click `Command` building. The Registry retains all modules — `list --exposure all` still shows hidden modules, and `exec <module_id>` can still invoke them directly. This preserves debuggability while keeping the default CLI surface clean.

---

## 2. Requirements Traceability

| Req ID | SRS Ref | Description |
|--------|---------|-------------|
| FR-12-01 | FR-DISP-001 | Root `--help` only shows exposed modules/groups. Hidden modules do not appear. |
| FR-12-02 | FR-DISP-002 | Hidden modules remain invocable via `exec <module_id>` (exposure is a UX filter, not a security boundary). |
| FR-12-03 | NFR-USB-001 | `expose` configuration in `apcore.yaml` with `mode`, `include`, and `exclude` fields. |
| FR-12-04 | NFR-USB-001 | Glob-pattern matching on module IDs (e.g., `admin.*`, `*.get`). |
| FR-12-05 | FR-DISC-001 | `list` command gains `--exposure` filter: `exposed` (default), `hidden`, `all`. |
| FR-12-06 | FR-DISC-001 | `list` output includes exposure status column when `--exposure all` is used. |
| FR-12-07 | NFR-USB-001 | `CliConfig.expose` field for programmatic usage (framework integrations). |
| FR-12-08 | NFR-USB-001 | 4-tier config precedence: `CliConfig.expose` > `--expose-mode` CLI flag > env var > `apcore.yaml`. |
| FR-12-09 | FR-DISP-001 | When `mode: include` and the include list is empty, zero modules are exposed (explicit opt-in). |
| FR-12-10 | FR-DISP-001 | When `mode: exclude` and the exclude list is empty, all modules are exposed (equivalent to `all`). |

---

## 3. Module Path

`apcore_cli/exposure.py` (new module)

Integration points:
- `apcore_cli/cli.py` — `GroupedModuleGroup._build_group_map()` calls `ExposureFilter.is_exposed()`
- `apcore_cli/discovery.py` — `list` command applies exposure filter
- `apcore_cli/config.py` — `ConfigResolver` gains `expose.*` keys
- `apcore_cli/__main__.py` — `create_cli()` accepts `expose` config

---

## 4. Implementation Details

### 4.1 Configuration Schema

**File**: `apcore.yaml`

```yaml
expose:
  mode: include          # "all" | "include" | "exclude". Default: "all"
  include:               # Used when mode is "include". Glob patterns.
    - admin.*
    - jobs.*
    - system.health
    - db.migrate
  exclude:               # Used when mode is "exclude". Glob patterns.
    - webhooks.*
    - internal.*
    - "*.sse"
```

**Validation rules:**
- `mode` must be one of `"all"`, `"include"`, `"exclude"`. Invalid value: exit 2.
- When `mode: include`, the `include` list is read; `exclude` list is ignored.
- When `mode: exclude`, the `exclude` list is read; `include` list is ignored.
- When `mode: all`, both lists are ignored.
- Each pattern must be a non-empty string. Empty string in list: log WARNING, skip.
- Duplicate patterns: deduplicated silently.

## Contract: ExposureFilter.__init__

### Inputs
- mode: str, optional — Filtering mode. One of `"all"`, `"include"`, `"exclude"`. Default: `"all"`.
  validates: must be one of the three accepted values; validated lazily in `from_config`, not in `__init__`
- include: list[str] | None, optional — Glob patterns for include mode. Default: `None` (treated as `[]`).
  validates: empty strings are skipped with WARNING (only when constructed via `from_config`)
- exclude: list[str] | None, optional — Glob patterns for exclude mode. Default: `None` (treated as `[]`).
  validates: same as include

### Errors
- (none raised by `__init__` itself — validation errors are raised by `from_config`)

### Returns
- On success: ExposureFilter instance with pre-compiled regex patterns

### Properties
- async: false
- thread_safe: false (shared state in compiled pattern lists; do not mutate after construction)
- pure: false (pre-compiles regexes as side effect)

---

## Contract: ExposureFilter.is_exposed

### Inputs
- module_id: str, required — The module identifier to check against the current filter mode.

### Errors
- (none raised — always returns bool)

### Returns
- On success: `True` if the module should be visible as a CLI command, `False` if hidden.
  - Mode `"all"`: always `True`
  - Mode `"include"`: `True` iff at least one include pattern matches `module_id`
  - Mode `"exclude"`: `True` iff no exclude pattern matches `module_id`
  - Mode `"none"` (or any unknown mode string, which is clamped to `"none"` with a WARNING at construction time): `False` (fail-closed across all SDKs as of D11-008, 2026-04). Earlier Python versions returned `True` (fail-open) here; that divergence is closed.

### Properties
- async: false
- thread_safe: true (read-only after construction)
- pure: true (no I/O, no side effects)

---

## Contract: ExposureFilter.filter_modules

### Inputs
- module_ids: list[str], required — List of module identifiers to partition.

### Errors
- (none raised — always returns tuple)

### Returns
- On success: `tuple[list[str], list[str]]` — `(exposed, hidden)` where each original ID appears in exactly one list, preserving input order within each partition.

### Properties
- async: false
- thread_safe: true (read-only after construction)
- pure: true (no I/O, no side effects)

---

## Contract: ExposureFilter.from_config

### Inputs
- config: dict, required — Parsed config dict. Expected shape: `{"expose": {"mode": ..., "include": [...], "exclude": [...]}}`.
  validates: `expose` must be a dict; `mode` must be one of `"all"`, `"include"`, `"exclude"`; `include`/`exclude` must be lists
  reject_with: click.BadParameter — when `mode` is not one of the valid values

### Errors
- click.BadParameter(f"Invalid expose mode: '{mode}'. Must be one of: all, include, exclude.") — when `mode` value is invalid

### Returns
- On success: ExposureFilter instance
- On non-dict `expose` value: returns `ExposureFilter()` (mode=all) after logging WARNING
- On non-list include/exclude: returns with empty list after logging WARNING

### Properties
- async: false
- thread_safe: true (no shared state; creates a new instance each call)
- pure: false (logs warnings as side effect)

---

### 4.2 Class: `ExposureFilter`

**File**: `apcore_cli/exposure.py`

```python
class ExposureFilter:
    """Determines which modules are exposed as CLI commands.

    Filtering modes:
    - all: every discovered module becomes a CLI command (default)
    - include: only modules matching at least one include pattern are exposed
    - exclude: all modules are exposed except those matching any exclude pattern
    """

    def __init__(
        self,
        mode: str = "all",
        include: list[str] | None = None,
        exclude: list[str] | None = None,
    ) -> None:
        self._mode = mode
        self._include_patterns = list(set(include or []))
        self._exclude_patterns = list(set(exclude or []))

    @classmethod
    def from_config(cls, config: dict) -> "ExposureFilter":
        """Create from parsed apcore.yaml expose section."""

    def is_exposed(self, module_id: str) -> bool:
        """Return True if the module should be exposed as a CLI command."""

    def filter_modules(self, module_ids: list[str]) -> tuple[list[str], list[str]]:
        """Partition module_ids into (exposed, hidden) lists."""
```

### 4.3 Method: `is_exposed`

**Signature**: `is_exposed(module_id: str) -> bool`

Logic steps:
1. If `self._mode == "all"`: return `True`.
2. If `self._mode == "include"`:
   a. For each pattern in `self._include_patterns`:
      - If `_glob_match(module_id, pattern)` is True: return `True`.
   b. Return `False` (no pattern matched — module is hidden).
3. If `self._mode == "exclude"`:
   a. For each pattern in `self._exclude_patterns`:
      - If `_glob_match(module_id, pattern)` is True: return `False`.
   b. Return `True` (no exclusion pattern matched — module is exposed).

### 4.4 Function: `_glob_match`

**Signature**: `_glob_match(module_id: str, pattern: str) -> bool`

Uses `fnmatch.fnmatch` semantics applied to module IDs:

| Pattern | Matches | Does Not Match |
|---------|---------|----------------|
| `admin.*` | `admin.users`, `admin.config` | `admin`, `admin_tools.users` |
| `admin.**` | `admin.users`, `admin.users.list` | `admin` |
| `*.get` | `product.get`, `user.get` | `product.get.all` |
| `system.health` | `system.health` (exact) | `system.health.check` |
| `*` | Any single-segment ID | N/A |
| `**` | Any module ID | N/A |

Implementation:
1. Replace `**` with a temporary sentinel (e.g., `__GLOBSTAR__`).
2. Replace remaining `*` with `[^.]*` (matches within a single dotted segment).
3. Replace sentinel with `.*` (matches across segments).
4. Anchor: `^pattern$`.
5. Compile and match via `re.fullmatch`.
6. Cache compiled regexes in `self._compiled_include` / `self._compiled_exclude` (built once in `__init__`).

### 4.5 Method: `from_config`

**Signature**: `from_config(cls, config: dict) -> ExposureFilter`

Logic steps:
1. Read `expose = config.get("expose", {})`. If not a dict: log WARNING, return `ExposureFilter()` (mode=all).
2. Read `mode = expose.get("mode", "all")`.
3. Validate `mode in ("all", "include", "exclude")`. If invalid: raise `click.BadParameter(f"Invalid expose mode: '{mode}'. Must be one of: all, include, exclude.")`.
4. Read `include = expose.get("include", [])`. If not a list: log WARNING, set to `[]`.
5. Read `exclude = expose.get("exclude", [])`. If not a list: log WARNING, set to `[]`.
6. Filter out empty strings with WARNING.
7. Return `ExposureFilter(mode=mode, include=include, exclude=exclude)`.

### 4.6 Method: `filter_modules`

**Signature**: `filter_modules(module_ids: list[str]) -> tuple[list[str], list[str]]`

Logic steps:
1. `exposed = []`, `hidden = []`.
2. For each `module_id` in `module_ids`:
   a. If `self.is_exposed(module_id)`: append to `exposed`.
   b. Else: append to `hidden`.
3. Return `(exposed, hidden)`.

### 4.7 Integration: `GroupedModuleGroup._build_group_map()`

**File**: `apcore_cli/cli.py`

Modify `_build_group_map()` to apply exposure filtering before building groups:

```python
def _build_group_map(self) -> None:
    if self._group_map_built:
        return
    self._build_alias_map()
    self._group_map = {}
    self._top_level_modules = {}
    for module_id in self._registry.list():
        descriptor = self._descriptor_cache.get(module_id)
        if descriptor is None:
            continue
        # NEW: Skip hidden modules
        if not self._exposure_filter.is_exposed(module_id):
            continue
        group, cmd = self._resolve_group(module_id, descriptor)
        if group is None:
            self._top_level_modules[cmd] = (module_id, descriptor)
        else:
            self._group_map.setdefault(group, {})[cmd] = (module_id, descriptor)
    # ... collision check, set flag
```

The `_exposure_filter` is passed via `GroupedModuleGroup.__init__()` as a new parameter. Default: `ExposureFilter()` (mode=all, backward compatible).

### 4.8 Integration: `create_cli()` Update

**File**: `apcore_cli/__main__.py`

Add exposure filter construction to `create_cli()`:

```python
# Full signature: see tech-design §8.2.7 "Consolidated Signature"
# This feature adds the `expose` parameter:
create_cli(..., expose: dict | ExposureFilter | None = None)
```

```python
def create_cli(
    # ... canonical parameters, see tech-design §8.2.7 ...
    expose: dict | ExposureFilter | None = None,  # NEW
) -> click.Group:
    # ... existing logic ...

    # Build exposure filter (4-tier precedence)
    if isinstance(expose, ExposureFilter):
        exposure_filter = expose
    elif isinstance(expose, dict):
        exposure_filter = ExposureFilter.from_config({"expose": expose})
    else:
        # Fall through to config file
        config = ConfigResolver()
        expose_config = config.resolve("expose", env_var="APCORE_CLI_EXPOSE_MODE")
        exposure_filter = ExposureFilter.from_config(config._config_file or {})

    # Pass to GroupedModuleGroup
    cli = click.Group(
        cls=GroupedModuleGroup,
        exposure_filter=exposure_filter,  # NEW
        # ... existing params ...
    )
```

### 4.9 Integration: `CliConfig` Update

**Cross-language normative reference:**

| Language | API | Exposure config |
|----------|-----|-----------------|
| Python | `create_cli(expose={"mode": "include", "include": ["admin.*"]})` | `dict` or `ExposureFilter` instance |
| TypeScript | `createCli({ expose: { mode: "include", include: ["admin.*"] } })` | `ExposeConfig` interface |
| Rust | `CliConfig { expose: Some(ExposeConfig { mode: Include, include: vec!["admin.*"] }), .. }` | `ExposeConfig` struct |

### 4.10 Integration: `list` Command Enhancement

**File**: `apcore_cli/discovery.py`

Add `--exposure` option to the enhanced `list` command:

```python
@click.option(
    "--exposure",
    type=click.Choice(["exposed", "hidden", "all"]),
    default="exposed",
    help="Filter by exposure status. Default: exposed.",
)
```

Logic changes in `cmd_list_enhanced()`:
1. After existing status/tag/annotation filtering, apply exposure filter:
   a. If `exposure == "exposed"`: keep only `is_exposed(m.module_id) == True`.
   b. If `exposure == "hidden"`: keep only `is_exposed(m.module_id) == False`.
   c. If `exposure == "all"`: keep all, add "Exposure" column to table output.
2. The "Exposure" column shows `"✓"` for exposed, `"—"` for hidden.

**Example output with `--exposure all`:**

```
+----------------+--------------------------+----------+----------+
| ID             | Description              | Tags     | Exposure |
+----------------+--------------------------+----------+----------+
| admin.users    | Manage user accounts.    | admin    | ✓        |
| admin.config   | Manage app settings.     | admin    | ✓        |
| webhooks.stripe| Handle Stripe events.    | webhook  | —        |
| internal.debug | Internal debug endpoint. | internal | —        |
+----------------+--------------------------+----------+----------+
```

### 4.11 Integration: `exec` Command Behavior

Hidden modules remain invocable via `exec <module_id>`. The `exec` subcommand bypasses the exposure filter entirely — it resolves directly from the Registry without checking `is_exposed()`.

This is intentional: exposure filtering is a **UX optimization**, not a security boundary. Developers who know the module_id can always invoke it directly for debugging, scripting, or edge cases.

### 4.12 Integration: Shell Completion

Shell completion scripts (bash/zsh/fish) must respect exposure filtering:
- Tab completion for `apcore-cli <group>` only lists exposed groups.
- Tab completion for `apcore-cli <group> <command>` only lists exposed commands within the group.
- Tab completion for `apcore-cli exec <module_id>` lists ALL modules (exposed + hidden).

---

## 5. Configuration Precedence

The exposure mode follows the existing 4-tier precedence from FE-07:

| Tier | Source | Example |
|------|--------|---------|
| 1 (highest) | `CliConfig.expose` / `create_cli(expose=...)` | Framework integration passes filter programmatically |
| 2 | `APCORE_CLI_EXPOSE_MODE` env var | `APCORE_CLI_EXPOSE_MODE=all` disables filtering temporarily |
| 3 | `apcore.yaml` `expose` section | Declarative project-level config |
| 4 (lowest) | Default | `mode: all` (backward compatible) |

**Note:** The env var only controls the `mode` field. The `include`/`exclude` patterns always come from `apcore.yaml` or `CliConfig.expose`. This prevents unwieldy shell exports for pattern lists.

---

## 6. Boundary Values & Limits

| Parameter | Minimum | Maximum | Default | Reference |
|-----------|---------|---------|---------|-----------|
| Include patterns | 0 | 500 | — | Each pattern validated on startup |
| Exclude patterns | 0 | 500 | — | Same |
| Pattern length | 1 char | 128 chars | — | Aligned with module_id max length |
| Glob nesting depth | N/A | N/A | — | `*` matches one segment, `**` matches across |
| Modules evaluated | 0 | 1000+ | — | Filter is O(modules × patterns) but patterns are regex-compiled once |

---

## 7. Error Handling

| Condition | Exit Code | Error Message | Reference |
|-----------|-----------|---------------|-----------|
| Invalid `mode` value in `apcore.yaml` | 2 | "Invalid expose mode: '{mode}'. Must be one of: all, include, exclude." | FR-12-03 |
| `expose` is not a dict in `apcore.yaml` | N/A (WARNING) | WARNING: "Invalid 'expose' config (expected dict), using mode: all." | FR-12-03 |
| `include`/`exclude` is not a list | N/A (WARNING) | WARNING: "Invalid 'expose.include' (expected list), ignoring." | FR-12-03 |
| Empty string in pattern list | N/A (WARNING) | WARNING: "Empty pattern in expose.include, skipping." | FR-12-04 |
| Invalid glob pattern (unclosed bracket) | N/A (WARNING) | WARNING: "Invalid pattern '{pattern}' in expose.include, skipping." | FR-12-04 |
| `mode: include` with no include list | N/A | Zero modules exposed. `--help` shows only built-in commands. | FR-12-09 |
| `mode: exclude` with no exclude list | N/A | All modules exposed (equivalent to `mode: all`). | FR-12-10 |

---

## 8. Verification

| Test ID | Description | Expected Result |
|---------|-------------|-----------------|
| T-EXP-01 | `mode: all` (default) | All discovered modules appear in `--help` and `list`. |
| T-EXP-02 | `mode: include`, `include: [admin.*]` with 3 admin + 2 user modules | Only `admin.*` modules in `--help`. `list` shows 3 modules. |
| T-EXP-03 | `mode: exclude`, `exclude: [webhooks.*, internal.*]` | All modules except `webhooks.*` and `internal.*` in `--help`. |
| T-EXP-04 | `mode: include`, `include: [admin.*]`, run `exec user.list` | Module executes successfully (exec bypasses exposure filter). |
| T-EXP-05 | `mode: include`, `include: []` (empty list) | Zero modules exposed. `--help` shows only built-in commands. |
| T-EXP-06 | `mode: exclude`, `exclude: []` (empty list) | All modules exposed (same as `mode: all`). |
| T-EXP-07 | `list --exposure all` | Shows all modules with "Exposure" column (✓/—). |
| T-EXP-08 | `list --exposure hidden` | Shows only hidden modules. |
| T-EXP-09 | `list --exposure exposed` (default) | Shows only exposed modules (matches `--help`). |
| T-EXP-10 | Pattern `admin.*` matches `admin.users` but not `admin.users.list` | Single-segment glob correctly scoped. |
| T-EXP-11 | Pattern `admin.**` matches `admin.users` and `admin.users.list` | Multi-segment glob matches recursively. |
| T-EXP-12 | Pattern `*.get` matches `product.get` and `user.get` | Wildcard matches any single segment prefix. |
| T-EXP-13 | Pattern `system.health` matches exact `system.health` only | No wildcard — exact match. |
| T-EXP-14 | `create_cli(expose={"mode": "include", "include": ["jobs.*"]})` | Programmatic exposure config works, overrides `apcore.yaml`. |
| T-EXP-15 | `APCORE_CLI_EXPOSE_MODE=all` overrides `apcore.yaml` `mode: include` | Env var disables filtering. |
| T-EXP-16 | Invalid `mode: whitelist` in `apcore.yaml` | Exit 2 with error message. |
| T-EXP-17 | Grouped commands: `mode: include`, `include: [product.*]` with product and user groups | Only `product` group in `--help`. `user` group hidden. |
| T-EXP-18 | Shell completion for `apcore-cli <TAB>` with exposure filter | Only exposed groups/commands in completion. |
| T-EXP-19 | Shell completion for `apcore-cli exec <TAB>` | All modules (exposed + hidden) in completion. |
| T-EXP-20 | 500 modules, 50 include patterns, startup time | Filter evaluation < 10ms. No impact on 100ms startup budget. |
| T-EXP-21 | Duplicate patterns in include list | Deduplicated silently. No double-matching. |
| T-EXP-22 | `mode: include` with `include: [admin.*]`, `list --status disabled` | Disabled admin modules shown (status filter is independent of exposure). |
| T-EXP-23 | `ExposureFilter` with `mode: all`, `filter_modules()` | Returns `(all_modules, [])` — no hidden modules. |
| T-EXP-24 | `describe admin.users` when module is hidden | Full metadata displayed (describe is not filtered by exposure). |

---

## 9. Impact on Existing Features

| Feature | Impact | Changes Required |
|---------|--------|-----------------|
| **Core Dispatcher (FE-01)** | `create_cli()` gains `expose` parameter; `GroupedModuleGroup` gains `exposure_filter` | Add parameter, pass through |
| **Grouped Commands (FE-09)** | `_build_group_map()` skips hidden modules; empty groups after filtering are omitted | Add `is_exposed()` check in loop |
| **Discovery (FE-04)** | `list` gains `--exposure` option; `describe` not affected | Add option + filter logic |
| **Config Resolver (FE-07)** | New config keys under `expose.*` | Add to DEFAULTS, document |
| **Shell Integration (FE-06)** | Completion scripts respect exposure filter | Update generators |
| **Output Formatter (FE-08)** | `format_module_list` gains optional "Exposure" column | Add column conditionally |
| **Schema Parser (FE-02)** | No impact | None |
| **Approval Gate (FE-03)** | No impact | None |
| **Security (FE-05)** | No impact (exposure is not a security boundary) | None |
| **Init Command (FE-10)** | No impact | None |
| **Usability Enhancements (FE-11)** | `enable`/`disable` is orthogonal to exposure; both filters apply independently | None |

---

## 10. Usage Examples

### 10.1 FastAPI Project (50+ endpoints, expose admin + jobs only)

```yaml
# apcore.yaml
expose:
  mode: include
  include:
    - admin.*
    - jobs.*
    - system.health
    - db.migrate
```

```bash
$ myapi --help
Usage: myapi [OPTIONS] COMMAND [ARGS]...

Commands:
  exec       Execute a module by ID
  list       List available modules
  describe   Show module details

Groups:
  admin      (3 commands)
  jobs       (2 commands)

Modules:
  db.migrate     Run database migration
  system.health  Check system health

$ myapi list --exposure all
+-------------------+--------------------------+----------+
| ID                | Description              | Exposure |
+-------------------+--------------------------+----------+
| admin.users       | Manage users             | ✓        |
| admin.config      | Manage config            | ✓        |
| admin.audit       | View audit log           | ✓        |
| jobs.reindex      | Reindex search           | ✓        |
| jobs.cleanup      | Cleanup expired data     | ✓        |
| db.migrate        | Run database migration   | ✓        |
| system.health     | Check system health      | ✓        |
| webhooks.stripe   | Handle Stripe events     | —        |
| webhooks.github   | Handle GitHub events     | —        |
| internal.debug    | Debug endpoint           | —        |
| users.list.get    | List users (API)         | —        |
| users.create.post | Create user (API)        | —        |
+-------------------+--------------------------+----------+
```

### 10.2 Small CLI Project (exclude a few noisy endpoints)

```yaml
# apcore.yaml
expose:
  mode: exclude
  exclude:
    - webhooks.*
    - internal.*
```

### 10.3 Framework Integration (programmatic)

```python
# In django-apcore or flask-apcore
from apcore_cli import create_cli, ExposureFilter

cli = create_cli(
    registry=my_registry,
    executor=my_executor,
    expose=ExposureFilter(
        mode="include",
        include=["admin.*", "manage.*"],
    ),
)
```
