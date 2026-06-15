---
description: "Grouped CLI Commands (FE-09) organizing dotted module IDs into nested Click subcommand groups by namespace prefix, with display.cli.group override and collapsed root help for 50+ modules."
---

# Feature Spec: Grouped CLI Commands

**Feature ID**: FE-09
**Status**: Ready for Implementation
**Priority**: P0
**Parent**: [Tech Design v2.0](../tech-design.md) Section 8.2.2
**SRS Requirements**: FR-DISP-001, FR-DISP-002, NFR-PERF-001, NFR-PERF-003, NFR-USB-001

---

## 1. Description

The Grouped CLI Commands feature extends the Core Dispatcher to organize modules into nested `click.Group` subcommands based on namespace prefixes. Instead of presenting all modules as flat top-level commands (unusable with 50+ routes), the CLI creates a two-level command hierarchy: `apcore-cli <group> <command>`. Groups are auto-detected from the first `.` segment of the CLI alias or module_id, with an explicit `display.cli.group` override available in binding.yaml.

---

## 2. Requirements Traceability

| Req ID | SRS Ref | Description |
|--------|---------|-------------|
| FR-09-01 | FR-DISP-001 | Root `--help` shows groups with command counts instead of flat module list. |
| FR-09-02 | FR-DISP-002 | Module execution via `apcore-cli <group> <command> [--flags]` invocation syntax. |
| FR-09-03 | FR-DISP-001 | Group `--help` shows individual commands within the group. |
| FR-09-04 | NFR-PERF-001 | Group map builds lazily, startup time remains < 100ms with 1000 modules. |
| FR-09-05 | NFR-USB-001 | `display.cli.group` override enables explicit group assignment. |
| FR-09-06 | NFR-USB-001 | `display.cli.group: ""` opts a module out of grouping (remains top-level). |
| FR-09-07 | FR-DISP-002 | Single-command groups remain as groups (not promoted to top-level). |
| FR-09-08 | FR-DISC-001 | `list` command shows grouped display by default, `--flat` flag for legacy flat view. |
| FR-09-09 | FR-DISC-003 | `describe` command resolves `group.command` notation to underlying module_id. |
| FR-09-10 | FR-SHELL-001 | Shell completion scripts handle two-level group/command structure. |

---

## 3. Module Path

`apcore_cli/cli.py` (classes: `GroupedModuleGroup`, `_LazyGroup`)

---

## 4. Implementation Details

## Contract: GroupedModuleGroup.__init__

### Inputs
- **kwargs: Any — forwarded to `LazyModuleGroup.__init__`; additionally consumes `exposure_filter` keyword.
  - exposure_filter: ExposureFilter | None, optional — Module visibility filter (FE-12). Default: `ExposureFilter()` (mode=all).

### Errors
- (none raised by constructor)

### Returns
- On success: GroupedModuleGroup instance

### Properties
- async: false
- thread_safe: false
- pure: false (reads module-level exposure filter)

---

## Contract: GroupedModuleGroup._resolve_group

### Inputs
- module_id: str, required — Canonical module identifier.
- descriptor: Any, required — Module descriptor with optional `display.cli.group` and `display.cli.alias` overlay.
- group_depth: int = 1, optional — Number of leading dot-segments treated as the group prefix. Default `1` matches the canonical single-level grouping (`a.b.c` → group `a`, command `b.c`). TypeScript accepts this parameter directly on `resolveGroup(moduleId, descriptor, groupDepth)`; Python folds depth handling into `_build_group_map` so `_resolve_group` itself is depth-1-only — both routes converge to the same group map.

### Errors
- (none raised — logs WARNING for empty module_id)

### Returns
- On success: tuple[str | None, str] — `(group_name, command_name)` where `group_name` is `None` for top-level modules

### Properties
- async: false
- thread_safe: true (read-only)
- pure: true (no I/O, no side effects, no logging except on empty module_id edge case)

### Language note
- **Rust**: no `GroupedModuleGroup` class — `clap` differs structurally from `click` (Python) and `commander` (TypeScript). The equivalent group-resolution logic is inlined in the per-command builder path (`build_module_command` and friends in `cli.rs`); the resulting routing is byte-identical, only the class shape differs.

---

## Contract: GroupedModuleGroup._build_group_map

### Inputs
- (no parameters — uses instance state)

### Errors
- click.UsageError — when a module's resolved group name or top-level CLI name is in `RESERVED_GROUP_NAMES` (e.g., `"apcli"`); exit code 2

### Returns
- On success: None — populates `_group_map` and `_top_level_modules` as side effects; sets `_group_map_built = True`
- On transient registry error: logs WARNING, does NOT set `_group_map_built` (allows retry)

### Properties
- async: false
- thread_safe: false (mutates instance state)
- pure: false (reads registry, mutates instance)

---

## Contract: GroupedModuleGroup.list_commands

### Inputs
- ctx: click.Context, required — Click context. Accepted to satisfy the Click `Group.list_commands` signature; not read by this override.

### Errors
- click.UsageError — propagated from `_build_group_map` when a module's resolved group / top-level CLI name collides with `RESERVED_GROUP_NAMES` (e.g., `"apcli"`). Maps to exit code 2.

### Returns
- On success: list[str] — sorted, deduplicated root entries: union of `_group_map` keys, `_top_level_modules` keys, and `"apcli"` when the apcli group is registered AND visible per `ApcliGroup.is_group_visible()`.
- Sort order: ascending lexicographic (`sorted(...)`); the `apcli` entry is included in the same sort, not pinned to the front.
- Empty registry: returns `["apcli"]` when the apcli group is visible, otherwise `[]`.

### Properties
- async: false
- thread_safe: false (calls `_build_group_map`, which mutates instance state on first call)
- pure: false (reads registry on first call via `_build_group_map`)

---

## Contract: GroupedModuleGroup.get_command

### Inputs
- ctx: click.Context, required — Click context. Forwarded to nested groups when applicable; not used directly for resolution.
- cmd_name: str, required — Root entry name to resolve.

### Errors
- (none raised — unknown names return `None`; `_build_group_map` errors propagate as `click.UsageError`, mapping to exit code 2)

### Returns
- On success (built-in / shim): click.Command — when `cmd_name` is in `self.commands` (e.g., the `apcli` group shim, registered before module discovery).
- On success (group): click.Group — `_LazyGroup` instance built from `_group_map[cmd_name]` and cached in `self._group_cache` on first access.
- On success (top-level module): click.Command — built lazily via `build_module_command(...)` from `_top_level_modules[cmd_name]` and cached in `self._module_cache`.
- On miss: None — `cmd_name` is neither built-in, nor a group, nor a top-level module.

Lookup order (first match wins): (1) `self.commands` (built-ins and the `apcli` shim), (2) `self._group_cache`, (3) `self._group_map` (build + cache), (4) `self._module_cache`, (5) `self._top_level_modules` (build + cache), (6) return `None`.

### Properties
- async: false
- thread_safe: false (mutates `_group_cache` and `_module_cache` on first access)
- pure: false (reads registry transitively, builds Click commands)

---

### 4.1 Class: `GroupedModuleGroup`

**File**: `apcore_cli/cli.py`

```python
class GroupedModuleGroup(LazyModuleGroup):
    """Click Group that organizes modules into nested subcommand groups.

    Group resolution priority:
    1. display.cli.group (explicit override from binding.yaml)
    2. First '.' segment of CLI alias (auto-detected)
    3. Top-level (no group - module has no '.' in name)
    """

    def __init__(self, **kwargs: Any) -> None:
        super().__init__(**kwargs)
        self._group_map: dict[str, dict[str, tuple[str, Any]]] = {}
        self._top_level_modules: dict[str, tuple[str, Any]] = {}
        self._group_cache: dict[str, click.Group] = {}
        self._group_map_built: bool = False
```

### 4.2 Method: `_resolve_group`

**Signature**: `_resolve_group(module_id: str, descriptor: Any) -> tuple[str | None, str]`

Returns `(group_name, command_name)`. `group_name` is `None` for top-level modules.

Logic steps:
1. Read `display = _get_display(descriptor)`.
2. Read `cli_display = display.get("cli") or {}`.
3. Read `explicit_group = cli_display.get("group")`.
4. If `explicit_group` is a non-empty string: return `(explicit_group, cli_display.get("alias") or module_id)`.
5. If `explicit_group` is an empty string `""`: return `(None, cli_display.get("alias") or module_id)`. This is the opt-out mechanism.
6. Determine `cli_name = cli_display.get("alias") or module_id`.
7. If `"."` in `cli_name`: split on first `.` and return `(prefix, suffix)`.
8. Else: return `(None, cli_name)`.

Edge cases:
- `module_id` is empty string: log WARNING, return `(None, module_id)`.
- `descriptor` is None: caller must ensure this never happens (skip in `_build_group_map`).
- `cli_name` contains multiple dots (e.g., `a.b.c`): split on FIRST dot only. `a` is group, `b.c` is command name within the group.

### 4.3 Method: `_build_group_map`

**Signature**: `_build_group_map() -> None`

Logic steps:
1. If `_group_map_built` is True, return (idempotent).
2. Call `self._build_alias_map()` to populate alias map and descriptor cache.
3. Initialize `_group_map = {}` and `_top_level_modules = {}`.
4. For each `module_id` in `self._registry.list()`:
   a. Get `descriptor = self._descriptor_cache.get(module_id)`. If None, skip.
   b. Call `(group, cmd) = self._resolve_group(module_id, descriptor)`.
   c. If `group is None`: `self._top_level_modules[cmd] = (module_id, descriptor)`.
   d. Else: `self._group_map.setdefault(group, {})[cmd] = (module_id, descriptor)`.
5. Check for collisions between group names and `RESERVED_GROUP_NAMES = frozenset({"apcli"})` (the only reserved root name post-FE-13). For each collision: raise `click.UsageError` (exit 2) with message `"Group name '{name}' conflicts with the reserved 'apcli' group. Choose a different module-ID prefix or set display.cli.group explicitly."`. Pre-FE-13 this was a soft WARNING against `BUILTIN_COMMANDS`; the v0.7+ contract is stricter because there is only one reserved name to defend.
6. Set `_group_map_built = True`.
7. Wrap entire method in try/except. On failure: log WARNING `"Failed to build group map"`, do NOT set the flag (allows retry on transient errors).

### 4.4 Method: `list_commands`

**Signature**: `list_commands(ctx: click.Context) -> list[str]`

Logic steps (post-FE-13, v0.7+):
1. The only reserved root entry is `apcli` (`RESERVED_GROUP_NAMES = frozenset({"apcli"})`). The 14-entry `BUILTIN_COMMANDS` constant was retired in v0.7.0; built-in subcommands now live under `<cli> apcli <sub>` and are NOT spliced at the root.
2. Call `_build_group_map()`.
3. `group_names = list(self._group_map.keys())` — collisions with `RESERVED_GROUP_NAMES` already rejected in `_build_group_map` step 5.
4. `top_names = list(self._top_level_modules.keys())`.
5. Compute `root = set(group_names) | set(top_names)`. If the apcli group is registered AND visible per `ApcliGroup.is_group_visible()`, add `"apcli"`.
6. Return `sorted(root)`.

> Cross-ref: [Tech Design §8.2.2](../tech-design.md), [features/builtin-group.md §4.9](builtin-group.md).

### 4.5 Method: `get_command`

**Signature**: `get_command(ctx: click.Context, cmd_name: str) -> click.Command | None`

Logic steps:
1. Check `self.commands` for built-in commands. If found, return it.
2. Call `_build_group_map()`.
3. Check `self._group_cache.get(cmd_name)`. If found, return cached group.
4. If `cmd_name in self._group_map`:
   a. Create `_LazyGroup` with members, executor, help_text_max_length.
   b. Cache in `self._group_cache[cmd_name]`.
   c. Return it.
5. If `cmd_name in self._top_level_modules`:
   a. Check `self._module_cache.get(cmd_name)`. If found, return it.
   b. Get `(module_id, descriptor)`.
   c. Build command via `build_module_command()`.
   d. Cache and return.
6. Return None.

### 4.6 Class: `_LazyGroup`

**File**: `apcore_cli/cli.py` (nested or private class)

```python
class _LazyGroup(click.Group):
    """Nested group that lazily builds commands from module descriptors."""

    def __init__(
        self,
        members: dict[str, tuple[str, Any]],
        executor: Executor,
        help_text_max_length: int = 1000,
        **kwargs: Any,
    ) -> None:
        super().__init__(**kwargs)
        self._members = members
        self._executor = executor
        self._help_text_max_length = help_text_max_length
        self._cmd_cache: dict[str, click.Command] = {}

    def list_commands(self, ctx: click.Context) -> list[str]:
        return sorted(self._members.keys())

    def get_command(self, ctx: click.Context, cmd_name: str) -> click.Command | None:
        if cmd_name in self._cmd_cache:
            return self._cmd_cache[cmd_name]
        entry = self._members.get(cmd_name)
        if entry is None:
            return None
        module_id, descriptor = entry
        cmd = build_module_command(
            descriptor,
            self._executor,
            help_text_max_length=self._help_text_max_length,
            cmd_name=cmd_name,
        )
        self._cmd_cache[cmd_name] = cmd
        return cmd
```

### 4.7 Method: `format_help`

Override `click.Group.format_help()` on `GroupedModuleGroup` to produce collapsed group display.

Logic steps:
1. Call `_build_group_map()`.
2. Write usage line: `formatter.write_usage(...)`.
3. Write root help text (from `self.help`).
4. Write "Options:" section (standard Click behavior for global options).
5. Write "Commands:" section listing the `apcli` group entry (when visible per `ApcliGroup.is_group_visible()`) with its help text. The 13 apcli subcommands are reachable via `<cli> apcli --help` and are NOT enumerated at the root.
6. If `self._top_level_modules` is non-empty: write "Modules:" section with top-level module names and descriptions.
7. If `self._group_map` is non-empty: write "Groups:" section. For each group in sorted order:
   a. `count = len(self._group_map[group_name])`.
   b. `suffix = "s" if count != 1 else ""`.
   c. `desc = f"({count} command{suffix})"`.
   d. Write `  {group_name:20s}  {desc}`.

### 4.8 Integration: `create_cli` Update

**File**: `apcore_cli/__main__.py`

Change the `cls` parameter from `LazyModuleGroup` to `GroupedModuleGroup`:

```python
@click.group(
    cls=GroupedModuleGroup,      # Changed from LazyModuleGroup
    registry=registry,
    executor=executor,
    help_text_max_length=help_text_max_length,
    name=prog_name,
    help="CLI adapter for the apcore module ecosystem.",
)
```

No other changes to `create_cli()` are needed. The `GroupedModuleGroup` inherits all `LazyModuleGroup` behavior and adds grouping on top.

### 4.9 Binding.yaml Group Override

The `display.cli.group` field is a **CLI-only convention**. It is NOT part of the PROTOCOL_SPEC §5.13 display overlay schema. It is documented here and in this feature spec only.

**Example binding.yaml:**

```yaml
bindings:
  - module_id: "product.get_product_product__product_id_.get"
    display:
      cli:
        group: "product"           # Explicit group name
        alias: "get"               # Command name within the group
        description: "Get product by ID"

  - module_id: "internal.health_check"
    display:
      cli:
        group: ""                  # Opt out of grouping (top-level)
        alias: "healthcheck"
        description: "Run health check"

  - module_id: "user.list_users.get"
    display:
      cli:
        alias: "user.list"         # Auto-groups to "user" group, command "list"
        description: "List all users"
```

### 4.10 Impact on Existing Features

| Feature | Impact | Changes Required |
|---------|--------|-----------------|
| **Core Dispatcher (FE-01)** | `create_cli()` uses `GroupedModuleGroup` instead of `LazyModuleGroup` | One line change in `__main__.py` |
| **Discovery (FE-04)** | `list` command shows grouped display by default; `describe` resolves `group.command` notation | Add `--flat` flag, add group resolution to `describe` |
| **Shell Integration (FE-06)** | Completion scripts must handle two-level groups | Update bash/zsh/fish completion generators |
| **Output Formatter (FE-08)** | `format_module_list` supports grouped display | Add `format_grouped_module_list()` function |
| **Schema Parser (FE-02)** | No impact | None |
| **Approval Gate (FE-03)** | No impact | None |
| **Security (FE-05)** | No impact | None |
| **Config Resolver (FE-07)** | No impact | None |

---

## 5. Boundary Values & Limits

| Parameter | Minimum | Maximum | Default | Reference |
|-----------|---------|---------|---------|-----------|
| Group name length | 1 char | 96 chars | — | Derived from module_id max 192 chars split on `.` |
| Command name length | 1 char | 191 chars | — | Module_id max 192 minus group prefix and `.` |
| Groups per CLI instance | 0 | 500 (design target) | — | 1000 modules / ~2 commands per group |
| Commands per group | 1 | 1000 | — | No artificial limit |
| Nesting depth | 1 level | 1 level | 1 | Only single-level grouping (split on first `.`) |

---

## 6. Error Handling

| Condition | Exit Code | Error Message | Reference |
|-----------|-----------|---------------|-----------|
| Group name collides with built-in command | N/A (WARNING) | WARNING log: "Group name '{name}' conflicts with built-in command." | FR-09-01 |
| `_build_group_map()` fails | N/A (WARNING) | WARNING log: "Failed to build group map". Falls back to flat mode. | NFR-REL-002 |
| Unknown group name at invocation | 2 | Click's built-in "No such command" error | FR-DISP-002 |
| Unknown command within group | 2 | Click's built-in "No such command" error within the group context | FR-DISP-002 |
| `describe` with unresolvable group.command | 44 | "Error: Module '{id}' not found." | FR-DISC-003 |

---

## 7. Verification

| Test ID | Description | Expected Result |
|---------|-------------|-----------------|
| T-GRP-01 | `_resolve_group()` with explicit `display.cli.group = "admin"` | Returns `("admin", alias)` |
| T-GRP-02 | `_resolve_group()` with `display.cli.group = ""` | Returns `(None, alias)` (top-level) |
| T-GRP-03 | `_resolve_group()` with module_id `user.create`, no display overlay | Returns `("user", "create")` |
| T-GRP-04 | `_resolve_group()` with module_id `standalone`, no display overlay | Returns `(None, "standalone")` |
| T-GRP-05 | `_build_group_map()` with 3 product modules + 2 user modules + 1 standalone | `_group_map` = `{product: 3, user: 2}`, `_top_level_modules` = `{standalone: 1}` |
| T-GRP-06 | `list_commands()` with above data | Returns sorted: `[completion, describe, exec, list, man, product, standalone, user]` |
| T-GRP-07 | `get_command(ctx, "product")` | Returns `_LazyGroup` with 3 commands |
| T-GRP-08 | `get_command(ctx, "standalone")` | Returns `click.Command` (top-level module) |
| T-GRP-09 | `apcore-cli product --help` via `CliRunner` | Output contains "list", "get", "create" as subcommands |
| T-GRP-10 | `apcore-cli product list --category food` via `CliRunner` | Module `product.list` executed with `{category: "food"}`, exit 0 |
| T-GRP-11 | `apcore-cli --help` via `CliRunner` | Output contains "Groups:" section with "product" and command count |
| T-GRP-12 | `_resolve_group()` with alias `a.b.c` | Returns `("a", "b.c")` — split on first `.` only |
| T-GRP-13 | `get_command(ctx, "list")` | Returns built-in `list` command (not a group named "list") |
| T-GRP-14 | `_build_group_map()` when module_id `list.something` exists | WARNING logged for group name collision with built-in command |
| T-GRP-15 | `apcore-cli health check` with single-command group | Group `health` has one command `check`, executed successfully |
| T-GRP-16 | `apcore-cli list --flat` via `CliRunner` | Flat table output matching v1.0 behavior |
| T-GRP-17 | `apcore-cli describe product.list` | Resolves to correct module_id, shows full metadata |
| T-GRP-18 | Registry with 500 modules across 50 groups, `--help` | Root help shows 50 group names with counts, startup < 100ms |
