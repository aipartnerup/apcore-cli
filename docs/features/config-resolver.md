---
description: "Config Resolver (FE-07) implementing the 4-tier precedence hierarchy CLI flag > env var > apcore.yaml > default, with graceful malformed-file handling and dot-notation key access."
---

# Feature Spec: Config Resolver

**Feature ID**: FE-07
**Status**: Ready for Implementation
**Priority**: P0
**Parent**: [Tech Design v2.0](../tech-design.md) Section 8.8
**SRS Requirements**: FR-DISP-005

---

## 1. Description

The Config Resolver implements the 4-tier configuration precedence hierarchy used throughout `apcore-cli`. For any configurable value, the resolver checks sources in order: CLI flag (highest priority), environment variable, config file (`apcore.yaml`), and built-in default (lowest priority). It handles malformed config files gracefully and supports flattened dot-notation key access.

---

## 2. Requirements Traceability

| Req ID | SRS Ref | Description |
|--------|---------|-------------|
| FR-07-01 | FR-DISP-005 | 4-tier configuration precedence: CLI > Env > File > Default. |
| FR-07-02 | FR-DISP-005 AF-1 | Config file not found: skip silently. |
| FR-07-03 | FR-DISP-005 AF-2 | Config file malformed: log WARNING, use defaults. |

---

## 3. Module Path

`apcore_cli/config.py`

---

## 4. Implementation Details

## Contract: ConfigResolver.__init__

### Inputs
- cli_flags: dict[str, Any], optional — Parsed CLI flag values keyed by flag name. Default: `{}`.
- config_path: str, optional — Path to `apcore.yaml`. Default: `"apcore.yaml"`.

### Errors
- (none raised — missing or malformed config file is handled silently/with WARNING)

### Returns
- On success: ConfigResolver instance with loaded (or empty) config

### Properties
- async: false
- thread_safe: false
- pure: false (reads filesystem)

---

## Contract: ConfigResolver.resolve

### Inputs
- key: str, required — Dot-notation config key (e.g., `"extensions.root"`).
- cli_flag: str, optional — Corresponding CLI flag name for Tier 1 lookup.
- env_var: str, optional — Environment variable name for Tier 2 lookup.

### Errors
- (none raised — returns None for unknown keys)

### Returns
- On success: Any — resolved value from the highest-priority tier that provides a non-None/non-empty value; or `None` if no tier has a value.

### Properties
- async: false
- thread_safe: true (read-only after construction)
- pure: false (reads os.environ)

---

### 4.1 Class: `ConfigResolver`

**Constructor**: `__init__(self, cli_flags: dict[str, Any] = None, config_path: str = "apcore.yaml")`

Logic steps:
1. Store `self._cli_flags = cli_flags or {}`.
2. Store `self._config_path = config_path`.
3. Call `self._config_file = self._load_config_file()`.

### 4.2 Method: `resolve`

**Signature**: `resolve(key: str, cli_flag: str = None, env_var: str = None) -> Any`

Logic steps:
1. **Tier 1 — CLI flag**: If `cli_flag` is not None and `cli_flag` exists in `self._cli_flags`:
   a. Get `value = self._cli_flags[cli_flag]`.
   b. If `value is not None`: return `value`.
2. **Tier 2 — Environment variable**: If `env_var` is not None:
   a. Get `env_value = os.environ.get(env_var)`.
   b. If `env_value is not None` and `env_value != ""`: return `env_value`.
3. **Tier 3 — Config file**: If `self._config_file is not None` and `key in self._config_file`:
   a. Return `self._config_file[key]`.
4. **Tier 4 — Default**: Return `self.DEFAULTS.get(key)`.

### 4.3 Default Values

```python
DEFAULTS = {
    "extensions.root": "./extensions",
    "logging.level": "WARNING",
    "sandbox.enabled": False,
    "cli.stdin_buffer_limit": 10_485_760,  # 10 MB
    "cli.auto_approve": False,
    "cli.help_text_max_length": 1000,
    "cli.approval_timeout": 60,        # added v0.6.0 (FE-11) — interactive approval prompt timeout (seconds)
    "cli.strategy": "standard",        # added v0.6.0 (FE-11) — pipeline strategy name
    "cli.group_depth": 1,              # added v0.6.0 (FE-09 group_depth) — multi-level grouping depth
}
```

**Environment-variable overrides** (Tier 2 — see [Tech Design §8.8](../tech-design.md)):
- `APCORE_EXTENSIONS_ROOT` → `extensions.root`
- `APCORE_LOGGING_LEVEL` / `APCORE_CLI_LOGGING_LEVEL` → `logging.level` (CLI-specific takes priority over global)
- `APCORE_CLI_SANDBOX` → `sandbox.enabled`
- `APCORE_CLI_AUTO_APPROVE` → `cli.auto_approve`
- `APCORE_CLI_HELP_TEXT_MAX_LENGTH` → `cli.help_text_max_length`
- `APCORE_CLI_APPROVAL_TIMEOUT` → `cli.approval_timeout` (added v0.6.0)
- `APCORE_CLI_STRATEGY` → `cli.strategy` (added v0.6.0)
- `APCORE_CLI_GROUP_DEPTH` → `cli.group_depth` (added v0.6.0)
- `APCORE_CLI_APCLI` → apcli visibility override (added v0.7.0, FE-13 — see [features/builtin-group.md](builtin-group.md))

### 4.4 Method: `_load_config_file`

**Signature**: `_load_config_file() -> dict | None`

Logic steps:
1. Try to open `self._config_path`.
2. On `FileNotFoundError`: return `None` (silent, per FR-DISP-005 AF-1).
3. Parse with `yaml.safe_load()`.
4. On `yaml.YAMLError`: log WARNING "Configuration file '{path}' is malformed, using defaults." Return `None`.
5. If parsed result is not a `dict`: log WARNING, return `None`.
6. Call `self._flatten_dict(config)` to convert nested YAML to dot-notation.
7. Return flattened dict.

### 4.5 Method: `_flatten_dict`

**Signature**: `_flatten_dict(d: dict, prefix: str = "") -> dict`

Logic steps:
1. Initialize `result = {}`.
2. For each `(key, value)` in `d.items()`:
   a. `full_key = f"{prefix}.{key}" if prefix else key`.
   b. If `isinstance(value, dict)`: recurse `result.update(self._flatten_dict(value, full_key))`.
   c. Else: `result[full_key] = value`.
3. Return `result`.

**Example:**
```yaml
# apcore.yaml
extensions:
  root: /custom/path
auth:
  api_key: "keyring:auth.api_key"
logging:
  level: DEBUG
```

Flattened to:
```python
{
    "extensions.root": "/custom/path",
    "auth.api_key": "keyring:auth.api_key",
    "logging.level": "DEBUG",
}
```

---

## 5. Configuration Keys

| Key | CLI Flag | Environment Variable | Default | Type | SRS Reference |
|-----|----------|---------------------|---------|------|---------------|
| `extensions.root` | `--extensions-dir` | `APCORE_EXTENSIONS_ROOT` | `./extensions` | `str` (path) | FR-DISP-003, FR-DISP-005 |
| `auth.api_key` | `--api-key` | `APCORE_AUTH_API_KEY` | `None` | `str` (secret) | FR-SEC-001 |
| `logging.level` | `--log-level` | `APCORE_LOGGING_LEVEL` (note: `APCORE_CLI_LOGGING_LEVEL` takes priority if both are set) | `WARNING` | `str` (enum) | NFR-MNT-002 |
| `sandbox.enabled` | `--sandbox` | `APCORE_CLI_SANDBOX` | `False` | `bool` | FR-SEC-004 |
| `cli.auto_approve` | `--yes` | `APCORE_CLI_AUTO_APPROVE` | `False` | `bool` | FR-APPR-004 |
| `cli.help_text_max_length` | — | `APCORE_CLI_HELP_TEXT_MAX_LENGTH` | `1000` | `int` | FR-SCHEMA-005 |

---

## 6. Parameter Validation

| Parameter | Type | Valid Values | Invalid Handling |
|-----------|------|-------------|------------------|
| `config_path` | `str` | Valid file path or non-existent path. | Non-existent: skip silently. Invalid YAML: WARNING + use defaults. |
| `cli_flags` | `dict` | Keys are flag names, values are parsed types. | `None`: treated as empty dict. |
| `key` | `str` | Dot-notation config key. | Unknown key: returns `None` from DEFAULTS. |
| `env_var` | `str` | Valid env var name. | Unset or empty string: skip to next tier. |

---

## 7. Precedence Test Matrix

| Test | CLI Flag | Env Var | Config File | Default | Expected Result |
|------|----------|---------|-------------|---------|-----------------|
| All sources set | `/cli-path` | `/env-path` | `/config-path` | `./extensions` | `/cli-path` |
| No CLI flag | — | `/env-path` | `/config-path` | `./extensions` | `/env-path` |
| No CLI flag, no env | — | — | `/config-path` | `./extensions` | `/config-path` |
| No CLI flag, no env, no config | — | — | — | `./extensions` | `./extensions` |
| CLI flag is `None` | `None` | `/env-path` | — | `./extensions` | `/env-path` |
| Env var is empty string | — | `""` | `/config-path` | `./extensions` | `/config-path` |
| Config file not found | — | — | (missing) | `./extensions` | `./extensions` |
| Config file malformed | — | — | (invalid YAML) | `./extensions` | `./extensions` + WARNING |

---

## 8. Error Handling

| Condition | Behavior | SRS Reference |
|-----------|----------|---------------|
| Config file not found | Skip silently, fall through to default. | FR-DISP-005 AF-1 |
| Config file malformed YAML | Log WARNING, fall through to default. | FR-DISP-005 AF-2 |
| Config file not a dict at root | Log WARNING, fall through to default. | FR-DISP-005 AF-2 |
| Unknown config key | Return `None` (from DEFAULTS.get). | — |

---

## 9. Verification

| Test ID | Description | Expected Result |
|---------|-------------|-----------------|
| T-CFG-01 | CLI flag `--extensions-dir /cli` with env var set | Uses `/cli` (Tier 1 wins). |
| T-CFG-02 | No CLI flag, `APCORE_EXTENSIONS_ROOT=/env` set | Uses `/env` (Tier 2 wins). |
| T-CFG-03 | No CLI flag, no env, `apcore.yaml` has `extensions.root: /config` | Uses `/config` (Tier 3 wins). |
| T-CFG-04 | No CLI flag, no env, no config file | Uses `./extensions` (Tier 4 default). |
| T-CFG-05 | Config file does not exist | No error. Default used. |
| T-CFG-06 | Config file is invalid YAML | WARNING logged. Default used. |
| T-CFG-07 | Config file has nested keys | Flattened correctly: `extensions.root` accessible. |
| T-CFG-08 | Env var set to empty string | Skipped. Falls through to config file or default. |
| T-CFG-09 | CLI flag explicitly set to `None` | Skipped. Falls through to env var or lower. |
