---
description: "Built-in Command Group (FE-13) relocating all apcore-cli commands under a reserved apcli namespace, with visibility modes (all/none/include/exclude) and disable_env opt-out."
---

# Feature Spec: Built-in Command Group (`apcli`)

**Feature ID**: FE-13
**Status**: Implemented (v0.7.0; refined v0.8.0)
**Priority**: P0
**Parent**: [Tech Design v2.0](../tech-design.md) Section 8.2
**SRS Requirements**: FR-DISP-001, FR-DISP-002, FR-DISP-009, FR-DISC-001, NFR-USB-001
**Related Features**: [Core Dispatcher (FE-01)](core-dispatcher.md), [Grouped Commands (FE-09)](grouped-commands.md), [Discovery (FE-04)](discovery.md), [Init Command (FE-10)](init-command.md), [Exposure Filtering (FE-12)](exposure-filtering.md)
**Breaking Change**: Yes (see §11 Migration)

---

## 1. Description

The Built-in Command Group feature restructures the apcore-cli command surface by moving **all apcore-cli-provided commands** (`list`, `describe`, `exec`, `init`, `validate`, `health`, `usage`, `enable`, `disable`, `reload`, `config`, `completion`, `describe-pipeline`, etc.) under a single reserved group named **`apcli`**. The root level retains only universally recognized meta-commands and flags (`help`, `--help`, `--version`, `--all-options`, `--man`, `--log-level`), plus user business modules/groups discovered from the registry.

This addresses two problems with the pre-v0.7 design:

1. **Namespace pollution in branded CLIs.** When apcore-cli is embedded into a framework integration (e.g., `aisee`, a vision-domain CLI built on `createCli({ registry, executor })`), the built-in commands `list`, `init`, `describe` are common English verbs that collide with real business commands. A terminal user running `aisee list` may reasonably expect to list vision tasks, not apcore modules.
2. **Help-output noise.** Without filtering, `aisee --help` exposes developer-tooling commands (`init module`, `describe-pipeline`, `config set`) to end users who should never see them.

The feature introduces an `apcli` configuration key with three visibility modes — fully visible, fully hidden, or partially visible (subcommand whitelist/blacklist) — plus a default policy that auto-detects embedded vs. standalone usage. Crucially, **hidden ≠ unreachable**: when the `apcli` group is hidden, `aisee apcli list` still executes successfully; it simply does not appear in `--help` output or shell completion. This preserves debuggability, CI scripting, and the "UX filter, not security boundary" philosophy established by FE-12.

A secondary concern is **respecting integrator intent against user-environment overrides**. The feature supports the industry-standard precedence (parameters > env var > config file > default, per Terraform/AWS CLI/dbt/Claude Code conventions) but adds a `disableEnv` opt-out in the `apcli` configuration. Integrators with strong product opinions — who do not want end users to circumvent a hidden `apcli` via `APCORE_CLI_APCLI=show` — can set `disableEnv: true` to sever that mechanism at the library boundary.

As a net simplification, the `BUILTIN_COMMANDS` reserved-names list and the per-command `builtins: {...}` override surface are **retired**. Because every apcore-cli-provided command now lives under the `apcli.*` namespace, business modules at the root level can never collide with built-ins — the collision problem is eliminated at the structural level rather than at config-validation time.

### 1.1 Visibility at a Glance

| Integrator intent | Config |
|-------------------|--------|
| Expose the entire built-in surface (typical for developer CLIs) | `apcli: true` |
| **Hide the entire `apcli` group from root `--help`** (typical for branded CLIs) | `apcli: false` *or* `{mode: none}` |
| Expose only select built-in subcommands | `{mode: include, include: [...]}` |
| Hide only select built-in subcommands | `{mode: exclude, exclude: [...]}` |
| Seal against end-user override via `APCORE_CLI_APCLI` env var | add `disable_env: true` |
| **Total lockdown** (root hidden, env sealed, only `exec` reachable) | `{mode: include, include: [], disable_env: true}` |

Hiding is an output filter; registered subcommands remain reachable via `<cli> apcli <subcommand>` under `mode: none` by design (FE-12 alignment). See §4.11 for the full reachability matrix.

---

## 2. Requirements Traceability

| Req ID | SRS Ref | Description |
|--------|---------|-------------|
| FR-13-01 | FR-DISP-009 | All apcore-cli-provided commands (except `help` and the five meta-flags) register under the `apcli` group. |
| FR-13-02 | FR-DISP-009 | Root `--help` shows only `help`, business modules/groups, and (conditionally) `apcli`. |
| FR-13-03 | FR-DISP-009 (AC-2) | Hidden `apcli` group remains invocable: `<cli> apcli list` executes successfully even when `apcli` is not listed in `--help`. |
| FR-13-04 | FR-DISP-009 | `apcli` config accepts: boolean shorthand (`true`/`false`), or object form with `mode`/`include`/`exclude`/`disableEnv` fields. |
| FR-13-05 | FR-DISP-009 (AC-1) | Default: embedded mode (registry injected via `create_cli()`) → `apcli` hidden; standalone mode (registry discovered from filesystem) → `apcli` visible. |
| FR-13-06 | FR-DISC-001 | `apcli list` / `apcli describe` preserve all behavior from pre-v0.7 `list` / `describe`. |
| FR-13-07 | FR-DISP-009 | Env var `APCORE_CLI_APCLI=show\|hide\|1\|0\|true\|false` forces visibility when env-var reading is not disabled. |
| FR-13-08 | FR-DISP-009 | Shell completion (bash/zsh/fish) respects `apcli` visibility: hidden group and its subcommands do not appear in completion when `apcli` is not visible. |
| FR-13-09 | FR-DISP-009 (AC-7) | Business module whose CLI alias, top-level command name, or group name is `apcli` is rejected with exit code 2 and clear error (the name is reserved). |
| FR-13-10 | FR-DISP-009 (AC-6) | 4-tier precedence for `apcli` visibility: `CliConfig.apcli` > `APCORE_CLI_APCLI` env var (when not disabled) > `apcore.yaml` > auto-detected default. |
| FR-13-11 | FR-DISP-009 (AC-5) | `apcli.disableEnv: true` severs the env-var override, making `CliConfig.apcli` / `apcore.yaml` / auto-detect the only resolution sources. `disableEnv` is configured via CliConfig or apcore.yaml only — it is not exposed as its own env var. |
| FR-13-12 | FR-DISP-009 (AC-4) | `exec` subcommand under `apcli` is always registered when the `apcli` group exists, regardless of `include`/`exclude` subcommand filtering — preserves FE-12's hidden-module-invocation guarantee. |
| FR-13-13 | FR-DISP-009 (AC-8) | Discovery-related root flags (`--extensions-dir`, `--commands-dir`, `--binding`) are registered only in standalone mode; in embedded mode (registry injected) they are not registered at all. |
| FR-13-14 | NFR-USB-001 | `--all-options` behavior is orthogonal to `apcli` visibility: `--all-options` continues to control per-command option density in help output and does not alter `apcli` group visibility. |

---

## 3. Module Path

- `apcore_cli/builtin_group.py` (new module — `ApcliGroup` class, visibility resolution)
- `apcore_cli/__main__.py` — `create_cli()` refactored to register under `apcli` group; discovery flags gated on standalone mode
- `apcore_cli/cli.py` — `GroupedModuleGroup` reserves `apcli` as a protected name; retires `BUILTIN_COMMANDS` constant; introduces `RESERVED_GROUP_NAMES`
- `apcore_cli/discovery.py` — `register_discovery_commands()` split into `register_list_command()`, `register_describe_command()`, `register_exec_command()`, `register_validate_command()` for per-subcommand registration control
- `apcore_cli/init_cmd.py` — `register_init_command()` attaches to `apcli` group
- `apcore_cli/system_cmd.py` — `register_system_commands()` split into `register_health_command()`, `register_usage_command()`, `register_enable_command()`, `register_disable_command()`, `register_reload_command()`, `register_config_command()` for per-subcommand registration control
- `apcore_cli/shell.py` — `register_shell_commands()` split into `register_completion_command()`; attaches to `apcli` group
- `apcore_cli/strategy.py` — `register_pipeline_command()` attaches to `apcli` group
- `apcore_cli/config.py` — `ConfigResolver` gains `apcli.*` keys

---

## 4. Implementation Details

### 4.1 Command Surface After Restructure

**Root level — always present:**

| Entry | Type | Description |
|-------|------|-------------|
| `help` | Command | Standard help dispatcher (Click/Commander built-in). |
| `--help` / `-h` | Flag | Per-command help. |
| `--version` | Flag | Print program version. |
| `--all-options` | Flag | Show built-in per-command options in `--help`. |
| `--man` | Flag | Render formatted man-page style help (used with `--help --man`). |
| `--log-level` | Flag | Set log level (DEBUG/INFO/WARNING/ERROR). |
| `<business-group>` | Group | User modules auto-grouped by namespace (see FE-09). |
| `<business-module>` | Command | Top-level user modules (no namespace prefix). |
| `apcli` | Group | Built-in group — visibility controlled by config. |

**Root level — standalone mode only (NOT registered when registry is injected):**

| Entry | Type | Description |
|-------|------|-------------|
| `--extensions-dir` | Flag | Path to extensions directory (filesystem discovery). |
| `--commands-dir` | Flag | Path to convention-based commands directory (apcore-toolkit). |
| `--binding` | Flag | Path to `binding.yaml` for display overlay (apcore-toolkit). |

Rationale: these three flags only have meaning when apcore-cli performs its own filesystem discovery. In embedded mode — where the integrator has pre-built the `registry` and passes it programmatically — the flags are inert (do nothing) and pollute `--help --all-options` output. Omitting them entirely in embedded mode results in cleaner help and an explicit "unknown option" error if a user attempts them.

**Under `apcli` group:**

| Subcommand | Source | Notes |
|------------|--------|-------|
| `apcli list` | `discovery.py` | Renamed from root `list`. |
| `apcli describe <module-id>` | `discovery.py` | Renamed from root `describe`. |
| `apcli exec <module-id>` | `discovery.py` | Renamed from root `exec`. **Always registered** (see §4.9). |
| `apcli validate <module-id>` | `discovery.py` | Renamed from root `validate`. |
| `apcli init module <id>` | `init_cmd.py` | Renamed from root `init`. |
| `apcli health` | `system_cmd.py` | Renamed from root `health`. |
| `apcli usage` | `system_cmd.py` | Renamed from root `usage`. |
| `apcli enable <module-id>` | `system_cmd.py` | Renamed from root `enable`. |
| `apcli disable <module-id>` | `system_cmd.py` | Renamed from root `disable`. |
| `apcli reload` | `system_cmd.py` | Renamed from root `reload`. |
| `apcli config get / set` | `system_cmd.py` | Renamed from root `config`. |
| `apcli completion <shell>` | `shell.py` | Renamed from root `completion`. |
| `apcli describe-pipeline` | `strategy.py` | Renamed from root `describe-pipeline`. |

### 4.2 Configuration Schema

**File**: `apcore.yaml`

```yaml
# Shorthand: hide the entire apcli group.
apcli: false

# Shorthand: show the entire apcli group.
apcli: true

# Object form: partial visibility via subcommand whitelist.
apcli:
  mode: include
  include:
    - list
    - describe

# Object form: partial visibility via subcommand blacklist.
apcli:
  mode: exclude
  exclude:
    - init
    - config
    - describe-pipeline

# Object form: lock down (hide + disable env-var override).
apcli:
  mode: none
  disable_env: true      # End users cannot use APCORE_CLI_APCLI to override.

# Object form: fully expanded.
apcli:
  mode: all              # "all" | "none" | "include" | "exclude". Default: auto-detect.
  include: []            # Used when mode is "include". Subcommand names.
  exclude: []            # Used when mode is "exclude". Subcommand names.
  disable_env: false     # Default: false (honor APCORE_CLI_APCLI env var).
```

**Semantic mapping (boolean shorthand → object form):**

| Shorthand | Equivalent Object Form |
|-----------|------------------------|
| `apcli: true` | `apcli: { mode: "all", disable_env: false }` |
| `apcli: false` | `apcli: { mode: "none", disable_env: false }` |
| `apcli: null` / omitted | Auto-detect visibility (see §4.5), `disable_env: false` |

**Validation rules:**
- `mode` must be one of `"all"`, `"none"`, `"include"`, `"exclude"`. Invalid value (including `"auto"` — which is an internal sentinel, not accepted from user config): exit 2.
- `mode: include` with empty `include` list: zero subcommands exposed (effectively equivalent to `mode: none`, but `exec` still special-cased per §4.9). No error.
- `mode: exclude` with empty `exclude` list: all subcommands exposed (equivalent to `mode: all`). No error.
- Unknown subcommand names in `include` / `exclude`: log WARNING, ignore. (Forward-compatible — allows config to reference subcommands added in future apcore-cli versions.)
- `disable_env` must be a boolean; default `false`. Non-boolean value: log WARNING, treat as `false`.
- `disable_env` may be set independently of `mode`. A config like `apcli: {disable_env: true}` with no `mode` key is valid and means "auto-detect visibility, but do not honor `APCORE_CLI_APCLI` env var" (internally treated as `mode: auto, disable_env: true`).
- `include` and `exclude` lists match against the **first-level apcli subcommand name only** (e.g., `list`, `config`). Nested paths (`config.set`) are not supported in v0.7 — if `config` is in the list, its entire subtree is controlled.

## Built-in group rename (cross-SDK feature, 2026-05-08)

Downstream branded CLIs that embed apcore-cli often prefer their built-ins under a custom namespace rather than the literal `apcli` (e.g., `mycorp-cli admin health` is friendlier than `mycorp-cli apcli health` to users who don't know what `apcli` is). The SDK supports this via an optional rename kwarg:

| SDK | API |
|-----|-----|
| Python | `create_cli(builtin_group_name="apcli", ...)` (default `"apcli"`) |
| TypeScript | `createCli({ builtinGroupName: "apcli", ... })` (default `"apcli"`). Note: `createCli` also accepts 3 positional arguments `(extensionsDirOrOpts?, progName?, verbose?)` where the first can be a `CreateCliOptions` object containing `builtinGroupName`; callers should prefer the options-object form for clarity. |
| Rust | Documented parity gap until embedding API is rebuilt; see `apcore-cli-rust/src/lib.rs:148-167` |

**Validation**: the resolved name MUST match `/^[a-z][a-z0-9_-]*$/` (non-empty, lowercase, leading letter, alphanumeric + `_` / `-`). Invalid values cause exit code 2 with a user-facing error message before any commands register.

**What renames**: the click/commander/clap sub-group node (so `apcli health` becomes `<resolved_name> health`); the `RESERVED_GROUP_NAMES` collision check (so a business module cannot claim the resolved name); listing/help filtering (so the renamed group is the one excluded from registry-driven `listCommands` output).

**What does NOT rename** (intentional cross-SDK invariants):
- Env var `APCORE_CLI_APCLI` — this is an apcore-cli-internal toggle, exposed to devops/CI rather than to end users. Stable across SDK versions and rename overrides.
- Config keys `apcli.mode` / `apcli.include` / `apcli.exclude` / `apcli.disable_env` — same rationale; the keyspace identity is the apcore-cli project, not the runtime group name.
- Spec / documentation / test names — examples in this document continue to use `apcli` as the canonical default. Implementations MUST preserve all behavior described here when the resolved name is `apcli`; the rename feature is purely a presentation-layer relabel.

**Pre-built ApcliGroup conflict**: when a caller passes both an `ApcliGroup` instance to `create_cli(apcli=...)` AND a `builtin_group_name` kwarg with a different value, both SDKs raise an error (TypeError in Python, Error in TS) and exit 2 — never silently pick a winner.

---

## Contract: ApcliGroup.__init__

### Inputs
- mode: str, optional — Internal visibility mode. One of `"auto"`, `"all"`, `"none"`, `"include"`, `"exclude"`. Default: `"auto"` (internal sentinel; never accepted from user config).
  validates: `"auto"` is only valid when constructed internally; user-facing constructors reject it
- include: list[str] | None, optional — Subcommand names to include. Used when `mode="include"`. Default: `None` (treated as `[]`).
- exclude: list[str] | None, optional — Subcommand names to exclude. Used when `mode="exclude"`. Default: `None` (treated as `[]`).
- disable_env: bool, optional — When True, severs `APCORE_CLI_APCLI` env-var override (Tier 2 skipped in `resolve_visibility`). Default: `False`.
- registry_injected: bool, optional — Whether the registry was provided programmatically (embedded mode). Used by Tier 4 auto-detect. Default: `False`.
- from_cli_config: bool, optional — True when constructed via `from_cli_config()` (Tier 1). A non-auto mode from Tier 1 wins outright over env var and yaml. Default: `False`.
- name: str, optional — Resolved built-in-group name (default `"apcli"`). Forwarded from `create_cli(builtin_group_name=...)` / `createCli({ builtinGroupName })`. Must match `/^[a-z][a-z0-9_-]*$/`.
  reject_with: ValueError (Python) / Error (TypeScript) — when the name fails the regex; create_cli/createCli catches and exits 2.

### Errors
- (none raised by constructor itself — validation errors are raised by `_build` / `from_cli_config` callers)

### Returns
- On success: ApcliGroup instance

### Properties
- async: false
- thread_safe: false (immutable after construction; `resolve_visibility` reads env at call time)
- pure: false (`resolve_visibility` reads `os.environ`)

---

## Contract: ApcliGroup.from_cli_config

### Inputs
- config: bool | dict | None, required — Tier 1 config value passed to `create_cli(apcli=...)`. `True` → `mode=all`; `False` → `mode=none`; `None` → `mode=auto`; `dict` → parsed per §4.2.
  reject_with: TypeError — when `config` is not bool, dict, or None; exit code 2 in create_cli()
- registry_injected: bool, required (keyword-only) — Whether registry was injected programmatically.

### Errors
- TypeError — when `config` is not bool, dict, or None (caught and converted to exit 2 by `create_cli()`)
- SystemExit(2) — when `config` is a dict with an invalid `mode` value (including `"auto"`)

### Returns
- On success: ApcliGroup with `_from_cli_config=True` (Tier 1 precedence)

### Properties
- async: false
- thread_safe: true (creates a new instance; no shared state)
- pure: false (may call sys.exit on invalid mode)

---

## Contract: ApcliGroup.from_yaml

### Inputs
- config: Any, required — Raw value from `apcore.yaml` at the `apcli:` key. Expected: bool, dict, or None. Unexpected types are coerced to `None` with a WARNING log (lenient shim).
- registry_injected: bool, required (keyword-only) — Whether registry was injected programmatically.

### Errors
- SystemExit(2) — when `config` is a dict with an invalid `mode` value

### Returns
- On success: ApcliGroup with `_from_cli_config=False` (Tier 3 precedence; env var may override)
- On unexpected `config` type: returns `ApcliGroup(mode="auto")` after logging WARNING

### Properties
- async: false
- thread_safe: true (creates a new instance)
- pure: false (logs WARNING on invalid input; may call sys.exit)

---

## Contract: ApcliGroup.resolve_visibility

### Inputs
- (no parameters — uses instance state and reads env at call time)

### Errors
- (none raised — always returns one of the four resolved modes)

### Returns
- On success: str — one of `"all"`, `"none"`, `"include"`, `"exclude"` (never `"auto"`)
  - Tier 1 (CliConfig, `_from_cli_config=True`, non-auto mode): returns `_mode` directly
  - Tier 2 (env var `APCORE_CLI_APCLI`, when `_disable_env=False`): `show/1/true` → `"all"`, `hide/0/false` → `"none"`
  - Tier 3 (yaml, non-auto mode): returns `_mode`
  - Tier 4 (auto-detect): `"none"` if `_registry_injected`, else `"all"`

### Properties
- async: false
- thread_safe: true (reads env; no mutation)
- pure: false (reads `os.environ`)

### Algorithm

```
resolve_visibility(self):
  # Tier 1: programmatic CliConfig wins outright. Embedders that explicitly
  # construct an ApcliGroup with a non-"auto" mode have made a deliberate
  # choice and the env var must NOT override it.
  if self._from_cli_config and self._mode != "auto":
    return self._mode

  # Tier 2: env var APCORE_CLI_APCLI (when not disabled). Two-value parser:
  #   show/1/true  → "all"
  #   hide/0/false → "none"
  # Other values are ignored (fall through). Env-disable flag is for tests
  # that need a hermetic environment; production embedders leave it unset.
  if not self._disable_env:
    raw = os.environ.get("APCORE_CLI_APCLI", "").strip().lower()
    if raw in ("show", "1", "true"):  return "all"
    if raw in ("hide", "0", "false"): return "none"
    # Any other value (including unset) → fall through to Tier 3.

  # Tier 3: yaml-loaded mode (apcli-config.yaml). Same shape as Tier 1 but
  # represents a config-file decision rather than a programmatic one. Env
  # already had its turn — yaml is honored only when env didn't decide.
  if self._mode != "auto":
    return self._mode

  # Tier 4: auto-detect by injection presence.
  #   _registry_injected = True  → embedded inside a host CLI; hide built-ins
  #                                 to avoid command-name pollution → "none"
  #   _registry_injected = False → standalone apcore-cli; expose all → "all"
  if self._registry_injected:
    return "none"
  return "all"
```

**Cross-SDK-critical invariants** (every implementation must preserve):
1. **Tier order is strict** — CliConfig > env > yaml > auto. The earlier-tier non-auto value wins; later tiers are NOT consulted.
2. The env-var parser is case-insensitive and trim-on-read. A leading/trailing space must NOT cause a Tier-3/Tier-4 fall-through silently.
3. Tier 4 default differs by injection state — embedded SDKs return `"none"`, standalone CLIs return `"all"`. Inverting this would surface or hide built-ins by accident in roughly half the deployments.
4. The function is idempotent w.r.t. instance state — repeated calls return the same value within one process unless `os.environ` mutates between calls.

---

## Contract: ApcliGroup.is_subcommand_included

### Inputs
- subcommand: str, required — First-level apcli subcommand name (e.g., `"list"`, `"init"`).

### Errors
- AssertionError — when called under `mode="all"` or `mode="none"` (callers must bypass this method for those modes)

### Returns
- On success (`mode="include"`): `True` iff `subcommand in self._include`
- On success (`mode="exclude"`): `True` iff `subcommand not in self._exclude`

### Properties
- async: false
- thread_safe: true (read-only after construction)
- pure: false (calls `resolve_visibility` which reads env)

---

## Contract: ApcliGroup.is_group_visible

### Inputs
- (no parameters — uses instance state)

### Errors
- (none raised)

### Returns
- On success: bool — `True` if `resolve_visibility() != "none"`; `False` when mode resolves to `"none"`

### Properties
- async: false
- thread_safe: true
- pure: false (calls `resolve_visibility` which reads env)

---

### 4.3 Class: `ApcliGroup`

**File**: `apcore_cli/builtin_group.py` (new)

```python
class ApcliGroup:
    """Built-in command group configuration for apcore-cli-provided commands.

    Encapsulates visibility resolution and subcommand filtering for the
    `apcli` group. Instantiated once by create_cli() and attached to the
    root Click group.
    """

    def __init__(
        self,
        mode: str = "auto",                  # internal sentinel; not accepted from user config
        include: list[str] | None = None,
        exclude: list[str] | None = None,
        disable_env: bool = False,
        registry_injected: bool = False,
        from_cli_config: bool = False,       # True when constructed from Tier 1 (CliConfig)
    ) -> None:
        ...

    @classmethod
    def from_cli_config(
        cls,
        config: bool | dict | None,
        *,
        registry_injected: bool,
    ) -> "ApcliGroup":
        """Tier 1 constructor — value came from create_cli(apcli=...).

        Sets from_cli_config=True internally. A Tier 1 value wins over env var
        and yaml: resolve_visibility() will NOT consult APCORE_CLI_APCLI.
        """
        ...

    @classmethod
    def from_yaml(
        cls,
        config: bool | dict | None,
        *,
        registry_injected: bool,
    ) -> "ApcliGroup":
        """Tier 3 constructor — value came from apcore.yaml.

        Sets from_cli_config=False. Env var (Tier 2) may override.
        """
        ...

    def resolve_visibility(self) -> str:
        """Return effective mode after applying tier precedence.

        Returns one of: "all", "none", "include", "exclude".
        """
        ...

    def is_subcommand_included(self, subcommand: str) -> bool:
        """True if the subcommand passes the include/exclude filter.

        Only meaningful when mode is "include" or "exclude". For "all" and
        "none", the dispatcher bypasses this check — under "all" every
        subcommand registers; under "none" every subcommand registers but the
        group itself is marked hidden in root --help.
        """
        ...

    def is_group_visible(self) -> bool:
        """True if the `apcli` group itself should appear in root `--help`."""
        ...
```

### 4.4 Method: `resolve_visibility`

**Signature**: `resolve_visibility(self) -> str`

Logic steps:
1. **Tier 1 — CliConfig.** If `self._from_cli_config` is True and `self._mode != "auto"`: return `self._mode` directly. Env var and yaml are bypassed.
2. **Tier 2 — env var.** Unless `self._disable_env` is True, check `APCORE_CLI_APCLI`:
   - Value `"show"` / `"1"` / `"true"` (case-insensitive) → return `"all"`.
   - Value `"hide"` / `"0"` / `"false"` (case-insensitive) → return `"none"`.
   - Other non-empty values → log WARNING, ignore and continue.
3. **Tier 3 — yaml.** If `self._mode != "auto"`: return `self._mode`.
4. **Tier 4 — auto-detect.** If `self._registry_injected` is True, return `"none"`; else return `"all"`.

**Precedence encoding:** the Tier 1 / Tier 3 distinction lives in the `_from_cli_config` flag set at construction time by the class methods `from_cli_config()` (Tier 1) vs. `from_yaml()` (Tier 3). This is the only way the algorithm can correctly implement "env var overrides yaml but not programmatic config."

**Interaction with `disable_env`:** `disable_env` is orthogonal to tier — it can be set alongside Tier 1 or Tier 3. When True, Tier 2 is always skipped. A Tier 1 value already wins over env var regardless of `disable_env`, so setting `disable_env: true` in CliConfig is defense-in-depth (harmless but unnecessary). The primary use case is setting it in Tier 3 (yaml) so that a committed yaml decision cannot be circumvented by end-user env vars.

### 4.5 Auto-Detect Default Logic

The `mode: auto` internal sentinel applies when no explicit `apcli` config is provided anywhere (not in CliConfig, not in yaml):

| Condition | Default | Rationale |
|-----------|---------|-----------|
| `create_cli(registry=reg, ...)` — registry injected | `mode: none` | Embedded CLI. End users consume business modules, not developer tools. |
| `create_cli()` — no registry, filesystem discovery | `mode: all` | Standalone `apcore-cli` binary. Developer/module-author audience. |

The `registry_injected` flag is captured by `create_cli()` at construction time. It is **not** re-evaluated at command-dispatch time.

**Note:** `"auto"` is an internal sentinel only. Users writing `mode: auto` in `apcore.yaml` receive an exit-2 validation error per §4.2. The way to request auto-detect behavior is to omit the `apcli` key entirely or set `apcli: null`.

### 4.6 Method: `is_subcommand_included`

**Signature**: `is_subcommand_included(self, subcommand: str) -> bool`

**Purpose:** answers "does this subcommand pass the `include`/`exclude` filter?" Only meaningful in filtered modes. Callers MUST check `resolve_visibility()` first and only consult this method when the mode is `"include"` or `"exclude"`.

Logic steps:
1. Let `mode = self.resolve_visibility()`.
2. If `mode == "include"`: return `subcommand in self._include`.
3. If `mode == "exclude"`: return `subcommand not in self._exclude`.
4. For `mode in ("all", "none")`: this method should not be called; raise `AssertionError("is_subcommand_included called under mode='{mode}'; caller should bypass")`.

**Why separate from "registered":** `mode == "none"` registers every subcommand (group-level output filter, not surface reduction). Conflating "visibility" and "registered" caused a bug in draft v1 where shell completion hid reachable commands. The dispatcher and completion generator both bypass this method for `"all"` and `"none"` and enumerate the actually-registered Click commands instead.

### 4.7 Method: `is_group_visible`

**Signature**: `is_group_visible(self) -> bool`

Logic steps:
1. Let `mode = self.resolve_visibility()`.
2. Return `mode != "none"`.

Note: when `mode == "include"` with an empty list, `is_group_visible()` returns True (the group itself appears), but only `exec` is registered under it (via the always-registered rule, §4.9). The group's `--help` will show just `exec`. This is the "total lockdown" shape — see §4.11 and §10.3. Logged at DEBUG level on startup.

### 4.8 Integration: `create_cli()` Update

**File**: `apcore_cli/__main__.py`

The `create_cli()` signature gains an `apcli` parameter. See tech-design §8.2.7 for the full canonical signature; this feature adds:

```python
def create_cli(
    # ... canonical parameters, see tech-design §8.2.7 ...
    apcli: bool | dict | ApcliGroup | None = None,  # NEW
) -> click.Group:
```

Construction logic:

```python
registry_injected = registry is not None

# Build ApcliGroup — dispatch by source to preserve Tier 1 vs Tier 3 precedence
if isinstance(apcli, ApcliGroup):
    apcli_cfg = apcli                                            # Pre-built; assumed to carry its own tier flag
elif apcli is not None and isinstance(apcli, (bool, dict)):
    apcli_cfg = ApcliGroup.from_cli_config(                      # Tier 1 path
        apcli, registry_injected=registry_injected,
    )
elif apcli is None:
    # Tier 3 (yaml) path. ConfigResolver must return the raw object at the
    # "apcli" key (non-leaf lookup). See M1 note below.
    yaml_value = ConfigResolver().resolve_object("apcli")        # bool | dict | None
    apcli_cfg = ApcliGroup.from_yaml(
        yaml_value, registry_injected=registry_injected,
    )
else:
    raise TypeError(f"apcli: expected bool, dict, ApcliGroup, or None; got {type(apcli).__name__}")

# Build root Click group (see tech-design §8.2.7 for full setup)
root = click.Group(cls=GroupedModuleGroup, ...)

# Conditionally register discovery-related root flags (standalone mode only)
if not registry_injected:
    root.params.append(click.Option(["--extensions-dir"], ...))
    root.params.append(click.Option(["--commands-dir"], ...))
    root.params.append(click.Option(["--binding"], ...))

# Build apcli Click group — always created, hidden from help conditionally
apcli_click_group = click.Group(
    name="apcli",
    help="Built-in apcore-cli commands.",
    hidden=not apcli_cfg.is_group_visible(),
)

# Register subcommands into apcli, filtered by visibility (§4.9)
_register_apcli_subcommands(apcli_click_group, apcli_cfg, registry, executor)

root.add_command(apcli_click_group)
return root
```

**M1 note — ConfigResolver non-leaf lookup:** FE-07's current `ConfigResolver.resolve()` is designed for scalar paths (e.g., `apcli.mode`). Returning the whole object at a non-leaf key (`apcli` → `dict | bool`) requires a new `resolve_object()` method on ConfigResolver (or equivalent). This is an FE-07 impact not called out in §9 of the draft — implementer should extend ConfigResolver first.

### 4.9 Integration: Per-Subcommand Registration

**File**: Multiple (`discovery.py`, `system_cmd.py`, `shell.py`, `strategy.py`, `init_cmd.py`)

Pre-v0.8, registration was batched (`register_discovery_commands` registered four subcommands in one call). This is incompatible with per-subcommand `include`/`exclude` filtering. The registrar functions are split 1-to-1 with subcommands:

```python
# Before (pre-v0.7)
def register_discovery_commands(cli: click.Group, registry: Registry) -> None:
    cli.add_command(list_cmd)
    cli.add_command(describe_cmd)
    cli.add_command(exec_cmd)
    cli.add_command(validate_cmd)

# After (v0.7) — one function per subcommand
def register_list_command(apcli_group: click.Group, registry: Registry) -> None: ...
def register_describe_command(apcli_group: click.Group, registry: Registry) -> None: ...
def register_exec_command(apcli_group: click.Group, registry: Registry, executor: Executor) -> None: ...
def register_validate_command(apcli_group: click.Group, registry: Registry, executor: Executor) -> None: ...
```

Same pattern applies to `system_cmd.py` (split into 6 registrars: health, usage, enable, disable, reload, config) and `shell.py` (split into 1 registrar: completion).

**Registration rules:**

| Mode | Subcommand registration |
|------|-------------------------|
| `"all"` | Every subcommand registers. Group visible in root `--help`. |
| `"none"` | Every subcommand registers. Group marked `hidden=True` so it does not appear in root `--help`, but `<cli> apcli <sub>` and `<cli> apcli --help` still work. |
| `"include"` | Only subcommands listed in `include` register, **plus `exec`** (always — FE-12 guarantee). |
| `"exclude"` | All subcommands except those listed in `exclude` register, **plus `exec`** (always). `exec` cannot be excluded. |

Design rationale: group-level hide (`mode: none`) is an **output filter** — the integrator wants a clean `--help` but retains full debugging capability. Subcommand-level hide (`include` / `exclude`) is a **surface reduction** — the integrator is declaring which tools they want to expose at all, and unlisted tools become unreachable (except `exec`).

Central dispatcher in `create_cli()`:

```python
# Subcommands that are always registered when the apcli group exists (FE-12 guarantee)
_ALWAYS_REGISTERED: frozenset[str] = frozenset({"exec"})

def _register_apcli_subcommands(
    apcli_group: click.Group,
    apcli_cfg: ApcliGroup,
    registry: Registry | None,
    executor: Executor | None,
) -> None:
    # (name, registrar_callable, requires_executor)
    registrars: list[tuple[str, Callable, bool]] = [
        ("list",              lambda: register_list_command(apcli_group, registry),               False),
        ("describe",          lambda: register_describe_command(apcli_group, registry),           False),
        ("exec",              lambda: register_exec_command(apcli_group, registry, executor),     True),
        ("validate",          lambda: register_validate_command(apcli_group, registry, executor), True),
        ("init",              lambda: register_init_command(apcli_group),                         False),
        ("health",            lambda: register_health_command(apcli_group, executor),             True),
        ("usage",             lambda: register_usage_command(apcli_group, executor),              True),
        ("enable",            lambda: register_enable_command(apcli_group, registry),             False),
        ("disable",           lambda: register_disable_command(apcli_group, registry),            False),
        ("reload",            lambda: register_reload_command(apcli_group, registry),             False),
        ("config",            lambda: register_config_command(apcli_group, executor),             True),
        ("completion",        lambda: register_completion_command(apcli_group),                   False),
        ("describe-pipeline", lambda: register_pipeline_command(apcli_group, executor),           True),  # Rust: register_pipeline_command(cli) -> Command — no executor param; executor wired via closure
    ]

    mode = apcli_cfg.resolve_visibility()
    for name, registrar, requires_exec in registrars:
        if requires_exec and executor is None:
            continue                                  # Skip commands that need an executor when none is available
        if mode in ("all", "none"):
            registrar()                                # All subcommands register; group visibility handled separately
            continue
        # mode in ("include", "exclude") — consult the filter, but always keep exec
        if name in _ALWAYS_REGISTERED or apcli_cfg.is_subcommand_included(name):
            registrar()
```

**Exception for `exec`:** `exec` is always registered whenever the `apcli` group exists, regardless of `include` / `exclude`. This preserves FE-12's hidden-module-invocation guarantee: hidden business modules remain reachable via `<cli> apcli exec <module_id>`. A user writing `include: []` to achieve total lockdown still leaves `exec` accessible — to truly block all module invocation, additional FE-12 / FE-05 mechanisms must be used, since FE-13 is a UX-layer feature.

### 4.10 Reserved Name Enforcement

**File**: `apcore_cli/cli.py` — `GroupedModuleGroup._build_group_map()`

Extends the existing validation to reject `apcli` as any user-controlled command identifier:

```python
RESERVED_GROUP_NAMES = frozenset({"apcli"})

# In _build_group_map():
for module_id, descriptor in self._modules.items():
    group, cmd = self._resolve_group(module_id, descriptor)
    if group in RESERVED_GROUP_NAMES:
        raise click.UsageError(
            f"Module '{module_id}': group name '{group}' is reserved. "
            f"Use a different CLI alias or set display.cli.group to another value."
        )
    if group is None and cmd in RESERVED_GROUP_NAMES:
        raise click.UsageError(
            f"Module '{module_id}': top-level CLI name '{cmd}' is reserved. "
            f"Use a different CLI alias."
        )
```

Checks cover three cases:
1. Module with `display.cli.group: apcli` → rejected (explicit group name).
2. Module auto-grouped as `apcli` via dotted ID (e.g., `apcli.foo`) → rejected (auto-extracted group).
3. Top-level module (no dot) whose alias or module_id is literally `apcli` → rejected (would collide with the apcli group at root).

This replaces the retired `BUILTIN_COMMANDS` collision check. Since all apcore-cli commands now live under a single reserved namespace, the only collision surface is the `apcli` name itself.

### 4.11 Hidden-but-Reachable Semantics

| Mode | `apcli` in root `--help` | `<cli> apcli --help` subcommands | `<cli> apcli list` executes | `<cli> apcli init` executes | `<cli> apcli exec <id>` executes |
|------|--------------------------|----------------------------------|------------------------------|------------------------------|-----------------------------------|
| `all` | ✅ visible | All subcommands | ✅ yes | ✅ yes | ✅ yes |
| `none` | ❌ hidden | All subcommands (group `--help` still works) | ✅ yes | ✅ yes | ✅ yes |
| `include: [list]` | ✅ visible | `list` + `exec` (always-registered) | ✅ yes | ❌ `No such command` | ✅ yes |
| `exclude: [init]` | ✅ visible | All except `init` | ✅ yes | ❌ `No such command` | ✅ yes |
| `include: []` (**total lockdown**) | ✅ visible (but shows only `exec`) | Only `exec` | ❌ `No such command` | ❌ `No such command` | ✅ yes (FE-12 guarantee) |

The asymmetry between **group-level hide** (`mode: none`, reachable) and **subcommand-level hide** (`include`/`exclude`, unreachable for filtered subcommands) is deliberate:

- Group-level hide is an **output filter**: integrators with `apcli: false` want a clean `--help` but full debugging capability. This matches FE-12's "UX filter, not security boundary" rule.
- Subcommand-level hide is a **surface reduction**: an integrator listing which subcommands they want implies the others should be gone. Failing loudly is better than silent success — except for `exec`, which is universally guaranteed.

**"Total lockdown" is NOT achieved by `mode: none` alone.** Intuition might suggest `apcli: false` means "everything under apcli disappears", but per the reachability matrix, `<cli> apcli list` still works — `mode: none` is an output filter. To make all subcommands *except* `exec` truly unreachable, use `{mode: include, include: [], disable_env: true}`. `exec` remains reachable in every mode per the FE-12 contract; blocking arbitrary module invocation requires FE-05 (security) or FE-12 mechanisms, not FE-13.

### 4.12 `disableEnv` Semantics

`disable_env` (YAML) / `disableEnv` (JS/TS/Go) / `disable_env` (Python/Rust) severs the env-var override channel.

| `disable_env` | `APCORE_CLI_APCLI` env var | Effect |
|---------------|----------------------------|--------|
| `false` (default) | unset | No effect; visibility from yaml/default. |
| `false` (default) | `show` / `hide` | Overrides yaml/default per Tier 2 in precedence. |
| `true` | unset | No effect. |
| `true` | `show` / `hide` | **Ignored.** Visibility from yaml/default only. |

**Key design decisions:**

1. `disableEnv` is **config-only** — not exposed as its own env var. Exposing an env var that disables env var reading produces circular semantics and undermines the lockdown purpose (end users could re-enable the override via env var).
2. Scope is limited to `APCORE_CLI_APCLI`. Other apcore-cli env vars (`APCORE_CLI_LOGGING_LEVEL`, `APCORE_EXTENSIONS_ROOT`, etc.) are unaffected — they serve distinct purposes (logging, discovery) and remain available.
3. The intended use case is **branded-CLI lockdown**: an integrator who set `apcli: false` does not want a user casually running `APCORE_CLI_APCLI=show aisee --help` to circumvent the product decision. `disable_env: true` seals that path.
4. Debugging an apcore-cli-powered branded CLI with `disable_env: true` requires either editing yaml locally or using standard Unix env manipulation (`env -u`, `unset`) — there is no runtime escape.

### 4.13 Integration: Shell Completion

Shell completion scripts (bash/zsh/fish) must reflect **what is actually reachable**, not a separate visibility notion:

- `<cli> <TAB>` (root): suggest `apcli` iff `is_group_visible()` returns True.
- `<cli> apcli <TAB>` (group): suggest **all subcommands actually registered** on the `apcli` Click group. This correctly covers all four modes:
  - `mode: all` / `mode: none` → all subcommands registered → all shown in completion
  - `mode: include` / `mode: exclude` → only passing subcommands registered → only those shown (plus `exec`, always-registered)
- `<cli> apcli list <TAB>` (subcommand): identical to pre-v0.7 `<cli> list <TAB>`.

Completion generators MUST enumerate the `apcli` group's registered subcommands at generation time (via Click/Commander introspection) — **not** consult `is_subcommand_included()`, which has narrower semantics. This avoids the draft-v1 bug where `mode: none` completions hid subcommands that were actually reachable.

Generated scripts are static. When apcli config changes, regenerate via `<cli> apcli completion <shell>`.

### 4.14 Cross-Language Normative Reference

| Language | API | Config type | Auto-detect trigger |
|----------|-----|-------------|---------------------|
| Python | `create_cli(apcli=False)` or `create_cli(apcli={"mode": "include", "include": ["list"], "disable_env": True})` | `bool \| dict \| ApcliGroup \| None` | `registry` is present |
| TypeScript | `createCli({ apcli: false })` or `createCli({ apcli: { mode: "include", include: ["list"], disableEnv: true } })` | `boolean \| ApcliConfig \| ApcliGroup \| undefined` | `registry` is present |
| Rust | `CliConfig { apcli: Some(ApcliConfig { mode: ApcliMode::None, disable_env: true, ..Default::default() }), .. }` | `Option<ApcliConfig>` struct (orthogonal `mode` + `disable_env` fields) | `cli_config.registry.is_some()` |
| Go | `CliConfig{Apcli: &ApcliConfig{Mode: ApcliModeNone, DisableEnv: true}}` or `CliConfig{Apcli: &ApcliConfig{Mode: ApcliModeInclude, Include: []string{"list"}}}` | `*ApcliConfig` struct, pointer for nullability | `cfg.Registry != nil` |

**Rust struct + enum (normative):**

Design: `mode` (visibility) and `disable_env` are orthogonal concerns. Separating them into a struct with a mode enum keeps every combination representable and aligns with the Go layout.

```rust
#[derive(Clone, Debug, Default)]
pub struct ApcliConfig {
    pub mode: ApcliMode,
    pub disable_env: bool,
}

#[derive(Clone, Debug, Default)]
pub enum ApcliMode {
    #[default]
    Auto,                          // internal sentinel (not user-facing)
    All,
    None,
    Include(Vec<String>),
    Exclude(Vec<String>),
}

pub struct CliConfig {
    pub registry: Option<Registry>,
    pub executor: Option<Executor>,
    pub apcli: Option<ApcliConfig>,
    // ... other fields
}
```

**Go struct (normative):**

```go
type ApcliMode string

const (
    ApcliModeAll     ApcliMode = "all"
    ApcliModeNone    ApcliMode = "none"
    ApcliModeInclude ApcliMode = "include"
    ApcliModeExclude ApcliMode = "exclude"
)

type ApcliConfig struct {
    Mode       ApcliMode
    Include    []string
    Exclude    []string
    DisableEnv bool
}

type CliConfig struct {
    Registry *Registry
    Executor *Executor
    Apcli    *ApcliConfig // nil = auto-detect
    // ... other fields
}
```

**TypeScript interface (normative):**

```typescript
export type ApcliConfig =
  | boolean
  | {
      mode: "all" | "none" | "include" | "exclude";
      include?: string[];
      exclude?: string[];
      disableEnv?: boolean;
    };

export interface CreateCliOptions {
  registry?: Registry;
  executor?: Executor;
  apcli?: ApcliConfig;
  // ... other fields
}
```

All four implementations MUST produce identical output for equivalent inputs. The conformance suite (`conformance/fixtures/apcli-visibility/`) provides cross-language fixtures.

---

## 5. Configuration Precedence

The `apcli` visibility follows the industry-standard 4-tier precedence:

| Tier | Source | Notes |
|------|--------|-------|
| 1 (highest) | `CliConfig.apcli` / `create_cli(apcli=...)` | Framework integration passes visibility programmatically. Non-null value wins over env var and yaml. |
| 2 | `APCORE_CLI_APCLI` env var | **Skipped when `apcli.disable_env: true` is set by Tier 1 or Tier 3.** |
| 3 | `apcore.yaml` `apcli:` key | Declarative project-level config. |
| 4 (lowest) | Auto-detected default | `none` when registry injected, `all` otherwise. |

**Env var value semantics:**

| Value | Effect |
|-------|--------|
| `show`, `1`, `true` | Force `mode: all` |
| `hide`, `0`, `false` | Force `mode: none` |
| unset | No override |
| any other non-empty | WARNING, ignored |

The env var controls **group-level visibility only** (all-or-nothing). It cannot convey subcommand whitelists — complex filtering belongs in config files.

**`disable_env` resolution:** when `disable_env` is set in Tier 1 (CliConfig), it wins. When Tier 1 is null, Tier 3 (yaml) `disable_env` applies. Default is `false`. `disable_env` cannot be set via env var.

---

## 6. Boundary Values & Limits

| Parameter | Minimum | Maximum | Default | Reference |
|-----------|---------|---------|---------|-----------|
| Include list length | 0 | (same as total subcommand count) | — | §4.2 |
| Exclude list length | 0 | (same as total subcommand count) | — | §4.2 |
| Subcommand name length | 1 | 32 chars | — | Aligned with Click command-name conventions |
| Visibility resolution cost | O(1) env var read + O(len(list)) membership check | — | — | Negligible (<1µs) |
| `apcli` group registration cost | Single Click group creation + conditional subcommand adds | — | — | No measurable startup impact |

---

## 7. Error Handling

| Condition | Exit Code | Error Message | Reference |
|-----------|-----------|---------------|-----------|
| Invalid `mode` value in `apcore.yaml` (including `"auto"`) | 2 | `"Invalid apcli mode: '{mode}'. Must be one of: all, none, include, exclude."` | FR-13-04 |
| `apcli` is neither bool, dict, nor null | N/A (TypeError) | `"apcli: expected bool, dict, ApcliGroup, or None; got {type}"` | FR-13-04 |
| Non-boolean `disable_env` value | N/A (WARNING) | `"WARNING: apcli.disable_env must be boolean; got {type}. Treating as false."` | FR-13-11 |
| Unknown subcommand in `include`/`exclude` | N/A (WARNING) | `"WARNING: Unknown apcli subcommand '{name}' in include list — ignoring."` | FR-13-04 |
| Business module's CLI group name is `apcli` | 2 | `"Module '{module_id}': group name 'apcli' is reserved. Use a different CLI alias or set display.cli.group to another value."` | FR-13-09 |
| Business module's top-level CLI name is `apcli` | 2 | `"Module '{module_id}': top-level CLI name 'apcli' is reserved. Use a different CLI alias."` | FR-13-09 |
| Hidden group, user runs `<cli> apcli list` | 0 | (normal execution — intentional escape hatch) | FR-13-03 |
| Subcommand-level hidden, user runs `<cli> apcli init` when `init` not in include | 2 | Click's standard `"No such command 'init'"` | §4.11 |
| `APCORE_CLI_APCLI` env var with unknown value | N/A (WARNING) | `"WARNING: Unknown APCORE_CLI_APCLI value '{val}', ignoring. Expected: show, hide, 1, 0, true, false."` | FR-13-07 |
| Embedded mode, user passes `--extensions-dir` | 2 | Click's standard `"Error: No such option: --extensions-dir"` | FR-13-13 |

---

## 8. Verification

| Test ID | Description | Expected Result |
|---------|-------------|-----------------|
| T-APCLI-01 | Standalone `apcore-cli --help` (no registry injected) | `apcli` group visible in root help. |
| T-APCLI-02 | Embedded `createCli({ registry })` with no `apcli` config | `apcli` group hidden from root help. |
| T-APCLI-03 | `apcli: true` with embedded registry | `apcli` group visible, overriding auto-default. |
| T-APCLI-04 | `apcli: false` in standalone mode | `apcli` group hidden, overriding auto-default. |
| T-APCLI-05 | Hidden `apcli`, run `<cli> apcli list` | Executes successfully. |
| T-APCLI-06 | Hidden `apcli`, run `<cli> apcli --help` | Shows subcommand list (group itself is reachable). |
| T-APCLI-07 | `apcli: { mode: include, include: [list, describe] }` | `list`, `describe`, and `exec` visible in `<cli> apcli --help` (exec always registered). |
| T-APCLI-08 | Same config, run `<cli> apcli init` | Error: "No such command 'init'". |
| T-APCLI-09 | `apcli: { mode: exclude, exclude: [init] }` | All subcommands except `init` visible and reachable. |
| T-APCLI-10 | `APCORE_CLI_APCLI=show` with yaml `apcli: false` | Group visible (env overrides yaml, Tier 2 > Tier 3). |
| T-APCLI-11 | `APCORE_CLI_APCLI=hide` with yaml `apcli: true` | Group hidden (env overrides yaml). |
| T-APCLI-12 | `APCORE_CLI_APCLI=bogus` | WARNING logged; yaml/default value used. |
| T-APCLI-13 | Programmatic `createCli({ apcli: false })` + `APCORE_CLI_APCLI=show` | Group hidden (Tier 1 > Tier 2). |
| T-APCLI-14 | `apcli: { mode: none, disable_env: true }` + `APCORE_CLI_APCLI=show` | Group hidden; env var ignored. |
| T-APCLI-15 | `apcli: { mode: none, disable_env: true }` + `APCORE_CLI_APCLI` unset | Group hidden (no change vs. without disable_env). |
| T-APCLI-16 | Business module registered with CLI group `apcli` | Exit 2 with reserved-name error. |
| T-APCLI-17 | Business module registered with top-level CLI name `apcli` | Exit 2 with reserved-name error. |
| T-APCLI-18 | Root `--help` output | Shows `help`, `--help`, `--version`, `--all-options`, `--man`, `--log-level`, business modules/groups, `apcli` (if visible). No stray built-in commands. |
| T-APCLI-19 | `<cli> apcli list --help` | Identical output to pre-v0.7 `<cli> list --help` (behavioral parity). |
| T-APCLI-20 | `apcli: { mode: include, include: [] }` | Group visible; `<cli> apcli --help` shows only `exec` (always-registered). |
| T-APCLI-21 | `apcli: { mode: exclude, exclude: [] }` | Equivalent to `mode: all`. |
| T-APCLI-22 | Invalid `mode: whitelist` in apcore.yaml | Exit 2 with error. |
| T-APCLI-23 | yaml `mode: auto` | Exit 2 — `auto` not accepted in user config. |
| T-APCLI-24 | `mode: include, include: [list]`, run `<cli> apcli exec some.module` | Executes successfully (exec always registered, FE-12 guarantee). |
| T-APCLI-25 | Unknown subcommand `foo` in `include` list | WARNING logged; entry ignored. |
| T-APCLI-26 | Non-boolean `disable_env: "yes"` in yaml | WARNING logged; treated as `false`. |
| T-APCLI-27 | Embedded mode, `<cli> --extensions-dir foo` | Exit 2 — unknown option (flag not registered in embedded mode). |
| T-APCLI-28 | Standalone mode, `<cli> --extensions-dir foo` | Flag accepted; discovery path honored. |
| T-APCLI-29 | Shell completion `<cli> <TAB>` with `apcli: false` | `apcli` absent from completion candidates. |
| T-APCLI-30 | Shell completion `<cli> apcli <TAB>` with `include: [list]` | `list` and `exec` in completion candidates. |
| T-APCLI-31 | Cross-language parity: Python/TS/Rust/Go, same `apcli: false` | Identical `--help` outputs. |
| T-APCLI-32 | Startup time with `apcli` group | Within 5% of pre-v0.7 baseline (no regression). |
| T-APCLI-33 | `createCli({ apcli: new ApcliGroup({...}) })` | Programmatic `ApcliGroup` instance accepted. |
| T-APCLI-34 | `<cli> --all-options --help` with `apcli: false` | Per-command options shown for business commands; `apcli` group remains hidden (orthogonal behaviors, FR-13-14). |
| T-APCLI-35 | `env -u APCORE_CLI_APCLI <cli> --help` with `disable_env: true` set | Standard unset behavior; works regardless of disable_env. |
| T-APCLI-36 | CliConfig form equivalence: `createCli({apcli: false})`, `createCli({apcli: {mode: "none"}})`, and `createCli({apcli: new ApcliGroup({mode: "none"})})` | All three produce byte-identical `--help` output. |
| T-APCLI-37 | yaml `apcli: {disable_env: true}` with no `mode` key | Treated as `mode: auto, disable_env: true`. Visibility follows auto-detect; env var ignored. |
| T-APCLI-38 | yaml sets `apcli: false`, CliConfig passes `apcli: true` | CliConfig wins (Tier 1 > Tier 3); group visible. |
| T-APCLI-39 | CliConfig passes `apcli: {mode: "include", include: ["list"]}`, `APCORE_CLI_APCLI=show` in env | CliConfig wins regardless of disable_env (Tier 1 > Tier 2); only `list` + `exec` visible. |
| T-APCLI-40 | `mode: none` + `<cli> apcli <TAB>` shell completion | All subcommands suggested (not empty — reachable ≠ visible). Regression guard for draft-v1 bug. |
| T-APCLI-41 | Total lockdown `{mode: include, include: [], disable_env: true}` + `APCORE_CLI_APCLI=show` | Only `exec` under apcli; env var sealed. `<cli> apcli list` → No such command; `<cli> apcli exec X` → executes. |

---

## 9. Impact on Existing Features

| Feature | Impact | Changes Required |
|---------|--------|-----------------|
| **Core Dispatcher (FE-01)** | `create_cli()` gains `apcli` parameter; all built-in command registrations move under a sub-group; discovery flags gated on standalone mode | Add parameter; refactor registration dispatch; conditional flag registration |
| **Schema Parser (FE-02)** | No impact | None |
| **Approval Gate (FE-03)** | No impact | None |
| **Discovery (FE-04)** | `list`/`describe`/`exec`/`validate` move under `apcli` group; registrars split per-subcommand | Split `register_discovery_commands` into four functions |
| **Security (FE-05)** | No impact | None |
| **Shell Integration (FE-06)** | `completion` subcommand moves under `apcli`; completion generators enumerate actually-registered subcommands on the apcli group (not `is_subcommand_included`) | Update generators per §4.13 |
| **Config Resolver (FE-07)** | New config key `apcli:` (non-leaf — returns bool or dict). Requires adding `resolve_object()` method alongside the existing scalar `resolve()`. | Add `resolve_object()` API; add `apcli` to DEFAULTS schema; document |
| **Output Formatter (FE-08)** | No impact (apcli group renders through existing man/help pipeline) | None |
| **Grouped Commands (FE-09)** | Reserves `apcli` as a protected name; retires `BUILTIN_COMMANDS` collision check | Replace constant with `RESERVED_GROUP_NAMES = {"apcli"}`; check top-level command names too |
| **Init Command (FE-10)** | `init` moves under `apcli` group | Update `register_init_command` signature |
| **Usability Enhancements (FE-11)** | `health`/`usage`/`enable`/`disable`/`reload`/`config` move under `apcli`; `--all-options` behavior preserved and orthogonal to apcli visibility | Split `register_system_commands` into per-subcommand registrars |
| **Exposure Filtering (FE-12)** | Orthogonal — `expose` controls business-module visibility; `apcli` controls built-in group visibility. FE-12's `exec` guarantee preserved via `_ALWAYS_REGISTERED` | None. Both filters compose independently. |

---

## 10. Usage Examples

### 10.1 Branded CLI (embedded, clean surface)

```typescript
// aisee/src/cli.ts
import { createCli } from "apcore-cli";
import { app } from "./app.js";

const cli = createCli({
  registry: app.registry,
  executor: app.executor,
  // apcli defaults to hidden because registry is injected — no config needed
});
cli.parse(process.argv);
```

```bash
$ aisee --help
Usage: aisee [options] [command]

aisee — AI-powered vision CLI

Options:
  -h, --help        Display help for command
  --version         Show aisee version
  --all-options     Show all options in help output
  --log-level <lvl> Logging level (DEBUG|INFO|WARNING|ERROR)

Commands:
  vision <cmd>      Vision analysis commands
  search            Search indexed assets
  help [command]    Display help for command

$ aisee apcli list
# Still works — escape hatch for debugging and CI
ID              Description
vision.ocr      Extract text from images
vision.describe Describe image contents
search          Search indexed assets
```

### 10.2 Branded CLI With Partial apcli Exposure

A team wants `aisee apcli list` surfaced for operator convenience, but hides developer tools.

```yaml
# apcore.yaml in aisee repo
apcli:
  mode: include
  include:
    - list
    - describe
```

```bash
$ aisee --help
# ...
Commands:
  vision <cmd>    Vision analysis commands
  search          Search indexed assets
  apcli <cmd>     Built-in apcore-cli commands
  help [command]  Display help for command

$ aisee apcli --help
Commands:
  list            List available modules
  describe <id>   Show module details
  exec <id>       Execute module by ID  # always-registered
```

### 10.3 Branded CLI With Hard Lockdown

Product team decides end users should never see or trigger any apcli behavior, even via env var.

```yaml
# apcore.yaml
apcli:
  mode: none
  disable_env: true
```

```bash
$ APCORE_CLI_APCLI=show aisee --help
# apcli still hidden — disable_env severs the env-var override path
Commands:
  vision <cmd>    Vision analysis commands
  ...

$ aisee apcli list
# Still reachable — mode: none is group-level hide, not surface reduction.
# To truly block, combine with mode: include + empty list.
```

For **total** lockdown (nothing reachable under apcli except `exec` per FE-12):

```yaml
apcli:
  mode: include
  include: []          # only exec remains
  disable_env: true
```

### 10.4 Standalone `apcore-cli` Binary (unchanged developer UX)

```bash
$ apcore-cli --help
# Full surface available — apcli auto-defaults to visible in standalone mode.
# Discovery flags (--extensions-dir, --commands-dir, --binding) available with --all-options.
Commands:
  apcli <cmd>     Built-in apcore-cli commands
  help [command]  Display help for command
  <user groups from ./extensions>
```

### 10.5 Runtime Override for Unlocked Branded CLI

The env-var override works only when **all three conditions** hold:

1. The branded CLI did not pass `apcli` programmatically to `create_cli()` (Tier 1 absent).
2. `apcore.yaml` does not set `disable_env: true`.
3. `apcore.yaml` does not pin a Tier-3 value that the user's intent (`show` / `hide`) conflicts with in a way the user cannot tolerate (env beats yaml, so this is usually fine).

```bash
# Works for the common case (integrator only relied on auto-detect default):
$ APCORE_CLI_APCLI=show aisee --help
# apcli group now visible; can run `aisee apcli list` etc.
```

If the integrator locked down with `disable_env: true` or passed programmatic config, env vars are ignored. Alternative: read the integrator's `apcore.yaml` locally, or use `env -u APCORE_CLI_APCLI aisee ...` for clean invocation.

### 10.6 Rust

```rust
use apcore_cli::{create_cli, CliConfig, ApcliConfig, ApcliMode};

let cli = create_cli(CliConfig {
    registry: Some(app.registry),
    executor: Some(app.executor),
    apcli: Some(ApcliConfig {
        mode: ApcliMode::Include(vec!["list".into(), "describe".into()]),
        disable_env: false,
    }),
    ..Default::default()
});
cli.run();
```

### 10.7 Go

```go
import "github.com/aipartnerup/apcore-cli-go"

cli := apcorecli.CreateCli(apcorecli.CliConfig{
    Registry: app.Registry,
    Executor: app.Executor,
    Apcli: &apcorecli.ApcliConfig{
        Mode:       apcorecli.ApcliModeExclude,
        Exclude:    []string{"init", "describe-pipeline"},
        DisableEnv: true,
    },
})
cli.Run()
```

---

## 11. Migration (Breaking Change)

This feature is a **breaking change** for standalone `apcore-cli` users who have scripts or documentation referencing root-level built-in commands.

### 11.1 What Breaks

| Pre-v0.7 invocation | v0.7+ invocation |
|---------------------|------------------|
| `apcore-cli list` | `apcore-cli apcli list` |
| `apcore-cli describe <id>` | `apcore-cli apcli describe <id>` |
| `apcore-cli exec <id>` | `apcore-cli apcli exec <id>` |
| `apcore-cli init module <id>` | `apcore-cli apcli init module <id>` |
| `apcore-cli health` | `apcore-cli apcli health` |
| (all other root built-ins) | (prefix with `apcli`) |

**What does NOT break:**
- Business module invocations (unaffected — modules remain at root level or in their own groups per FE-09).
- Embedded integrations (`create_cli(registry=...)`) — no script ever relied on built-in commands being exposed in that mode; the new default (hidden) is strictly better.
- `--help` / `--version` / `--all-options` / `--log-level` / `--man` flags.

### 11.2 Deprecation Window

For **v0.7.x**, apcore-cli emits a deprecation warning when a root-level built-in command is invoked (standalone mode only):

```
$ apcore-cli list
WARNING: 'list' as a root-level command is deprecated. Use 'apcore-cli apcli list' instead.
         Will be removed in v0.8. See: https://aiperceivable.github.io/apcore-cli/features/builtin-group/#11-migration
```

The command still executes. In **v0.8.0**, root-level aliases are removed entirely.

The deprecation wrapper is implemented by registering thin shim commands at the root for each former built-in, each of which prints the warning and invokes the corresponding `apcli` subcommand. The shim is active **only in standalone mode** (never in embedded mode — embedded integrators' end users should not see deprecation warnings for commands they were never meant to know about).

### 11.3 Version Rollout

| Version | Behavior |
|---------|----------|
| pre-v0.7 | Flat surface (all built-ins at root) |
| v0.7.0 | `apcli` group introduced; standalone mode emits deprecation warnings on old flat invocations; embedded mode defaults to hidden (no deprecation shown); discovery flags gated on standalone mode |
| v0.8.0 | Flat-surface deprecation shims removed; `apcli` group is the only path |

### 11.4 Retiring `BUILTIN_COMMANDS`

The `BUILTIN_COMMANDS` constant in `apcore_cli/cli.py` (and equivalents in the other-language implementations) is **retired** in v0.7.0. It is replaced by:

- `RESERVED_GROUP_NAMES = frozenset({"apcli"})` — one-element set covering group names, auto-grouped prefixes, and top-level command names
- Removal of the per-command collision check in `GroupedModuleGroup._build_group_map()` (was: warn + drop module on collision)

External code that imported `BUILTIN_COMMANDS` will break at import time. Impact assessed as low (internal constant, not part of public API per semver).

---

## 12. Open Questions

1. **Should `help` also move under `apcli`?** Current design keeps `help` at root because it is the most universally recognized meta-command. Resolution: **root `help` preferred** — the English verb `help` has decades of meta-command consensus across `man`, `git`, `docker`, etc.
2. **Should `config get/set` stay nested under `apcli config <subcmd>` or flatten to `apcli config-get` / `apcli config-set`?** Current design preserves the pre-v0.7 nested structure (`apcli config get <key>`). Rationale: the config surface has distinct read vs. write semantics that subcommand grouping expresses well.
3. **Go SDK timing.** `apcore-cli-go` does not exist as of v0.7.0. The normative Go signature in §4.14 is a forward-looking contract; it becomes binding when the Go SDK is bootstrapped (planned for a later minor release per roadmap). Conformance tests for Go are deferred until then.
