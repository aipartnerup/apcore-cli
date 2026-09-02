---
description: "Usability Enhancements (FE-11, v0.6.0) adding --dry-run, system commands, error ai_guidance, --trace, --stream, enhanced list, --strategy, and extra output formats over the apcore API."
---

# Feature Spec: Usability Enhancements (v0.6.0)

**Feature ID**: FE-11
**Status**: Implemented (v0.6.0)
**Priority**: P0–P2 (see per-section priority)
**Parent**: [Tech Design v2.0](../tech-design.md)
**apcore Dependency**: `>= 0.17.1`

---

## 1. Motivation

apcore-cli currently uses approximately 15–20% of the apcore public API surface. It treats apcore as a "registry + executor" only, missing the governance, observability, pipeline introspection, and system management capabilities that apcore already provides. This spec defines concrete enhancements that close these gaps.

**Design principles:**

- **Leverage, don't reimplement.** Every new capability maps directly to an existing apcore API.
- **Non-breaking.** All enhancements are additive flags, new subcommands, or optional behaviors.
- **Cross-language.** Designs are language-agnostic — Python, TypeScript, and Rust SDKs implement the same interface.

---

## 2. Requirements Traceability

| Req ID | Priority | Section | Description |
|--------|----------|---------|-------------|
| FR-11-01 | P0 | §3.1 | `--dry-run` preflight mode via `Executor.validate()` |
| FR-11-02 | P0 | §3.2 | System management commands (`health`, `usage`, `enable`, `disable`, `reload`, `config`) |
| FR-11-03 | P0 | §3.3 | Enhanced error output with `ai_guidance`, `suggestion`, `retryable` |
| FR-11-04 | P1 | §3.4 | `--trace` execution pipeline visualization via `call_with_trace()` |
| FR-11-05 | P1 | §3.5 | Standard `ApprovalHandler` protocol integration |
| FR-11-06 | P1 | §3.6 | `--stream` output via `Executor.stream()` |
| FR-11-07 | P1 | §3.7 | Enhanced `list` command (search, filter, sort) |
| FR-11-08 | P2 | §3.8 | `--strategy` selection for execution pipelines |
| FR-11-09 | P2 | §3.9 | Output format extensions (`csv`, `yaml`, `jsonl`, `--fields`) |
| FR-11-10 | P2 | §3.10 | Multi-level nested grouping |
| FR-11-11 | P2 | §3.11 | Custom command extension point |

---

## 3. Implementation Details

### 3.1 Dry-Run Preflight Mode (P0) {#dry-run}

**Problem:** Users and AI agents cannot verify that a module call will succeed without actually executing it. Destructive modules are especially risky.

**apcore APIs used:**
- `Executor.validate(module_id, inputs, context) → PreflightResult`
- `PipelineContext.dry_run = True` (skips impure steps: approval, middleware, execute)

#### 3.1.1 CLI Interface

**Flag on module execution commands:**

```
apcore-cli <module_id> [--flags...] --dry-run
```

**Standalone validate command:**

```
apcore-cli validate <module_id> [--flags...] [--input -]
```

Both invoke `Executor.validate()` and display the `PreflightResult`.

#### 3.1.2 Output Format

**TTY mode (human-readable):**

```
Preflight check: math.add --a 5 --b hello

  ✓ module_id         Valid format
  ✓ module_lookup     Module found (v1.2.0)
  ✓ call_chain        Depth 1/10, no cycles
  ✓ acl               Allowed (rule #2: "allow developers")
  ✗ schema            Validation failed
                      Field 'b': expected integer, got string
  ○ approval          Skipped (not required)
  ○ module_preflight  Skipped (module has no preflight method)

Result: FAIL (1 error, 0 warnings)
```

Symbols: `✓` passed, `✗` failed, `○` skipped, `⚠` passed with warnings.

**Non-TTY / JSON mode:**

```json
{
  "valid": false,
  "requires_approval": false,
  "checks": [
    {"check": "module_id", "passed": true},
    {"check": "module_lookup", "passed": true},
    {"check": "call_chain", "passed": true},
    {"check": "acl", "passed": true, "warnings": []},
    {"check": "schema", "passed": false, "error": {"field": "b", "expected": "integer", "got": "string"}},
    {"check": "approval", "passed": true},
    {"check": "module_preflight", "passed": true, "warnings": []}
  ]
}
```

#### 3.1.3 Exit Codes

| Condition | Exit Code |
|-----------|-----------|
| All checks passed | 0 |
| Schema validation failed | 45 |
| Module not found | 44 |
| ACL denied | 77 |
| Other check failures | 1 |

When multiple checks fail, the **first failed check** determines the exit code (following existing pipeline step order).

#### 3.1.4 Cross-Language API

| Language | Flag API | Standalone API |
|----------|----------|---------------|
| Python | `build_module_command(..., dry_run=True)` | `register_validate_command(cli, registry, executor)` |
| TypeScript | `buildModuleCommand({..., dryRun: true})` | `registerValidateCommand(cli, registry, executor)` |
| Rust | `ModuleCommandConfig { dry_run: true, .. }` | `register_validate_command(&mut cli, &registry, &executor)` |

#### 3.1.5 Implementation Pseudocode

```python
# In build_module_command callback, after input merging:
if dry_run:
    result = executor.validate(module_id, merged, context=None)
    format_preflight_result(result, fmt=output_format)
    sys.exit(0 if result.valid else _first_failed_exit_code(result))
    return
```

#### 3.1.6 Verification Tests

| ID | Test |
|----|------|
| T-DRY-01 | `--dry-run` with valid inputs → exit 0, all checks ✓ |
| T-DRY-02 | `--dry-run` with invalid schema → exit 45, schema check ✗ |
| T-DRY-03 | `--dry-run` with non-existent module → exit 44 |
| T-DRY-04 | `--dry-run` does NOT trigger actual module execution (verify with side-effect module) |
| T-DRY-05 | `--dry-run` does NOT trigger approval prompt |
| T-DRY-06 | `validate` command produces same result as `--dry-run` |
| T-DRY-07 | `--dry-run --format json` outputs JSON PreflightResult |
| T-DRY-08 | `--dry-run` with ACL denied module → exit 77 |

---

### 3.2 System Management Commands (P0) {#system-commands}

**Problem:** apcore provides `system.*` modules for health, usage, and runtime control, but CLI users have no way to access them.

**apcore APIs used:**
- `system.health.summary` / `system.health.module`
- `system.usage.summary` / `system.usage.module`
- `system.control.toggle_feature` / `system.control.reload_module` / `system.control.update_config`

#### 3.2.1 Command Design

All system commands are registered as **first-class built-in commands** alongside `list` and `describe`. They delegate to the corresponding `system.*` module via `Executor.call()`.

```
apcore-cli health [--threshold 0.01] [--all]
apcore-cli health <module_id> [--errors 10]
apcore-cli usage [--period 24h]
apcore-cli usage <module_id> [--period 24h]
apcore-cli enable <module_id> --reason "..."
apcore-cli disable <module_id> --reason "..."
apcore-cli reload <module_id> --reason "..."
apcore-cli config get <key>
apcore-cli config set <key> <value> --reason "..."
```

#### 3.2.2 `health` Command

**Invocation: `apcore-cli health`**

Calls `system.health.summary` → TTY-adaptive output.

**TTY output:**

```
Health Overview (4 modules)

  Module              Status     Error Rate   Top Error
  ──────────────────  ─────────  ──────────   ─────────────────
  math.add            healthy    0.00%        —
  db.query            degraded   3.20%        MODULE_TIMEOUT (12)
  email.send          error      15.70%       ACL_DENIED (89)
  report.build        unknown    0.00%        —

Summary: 1 healthy, 1 degraded, 1 error, 1 unknown
```

**Status tiers.** `system.health.summary` classifies every module into one of
**four** tiers — `healthy`, `degraded`, `error`, `unknown` — and the `summary`
object carries a count for each alongside `total_modules`. The tier set is owned
by apcore (`schemas/sys-health-summary.schema.json`, PROTOCOL_SPEC §6.6); the CLI
renders it and MUST NOT drop a tier.

`unknown` means **no calls have been recorded yet**, which is the state every
module in a freshly initialised project is in. It is therefore the tier the
summary line is most likely to be reporting on a first run, and omitting it
produces the specific defect of a populated table above a total that reads
`no data`. The summary line lists only non-zero tiers, and falls back to
`no data` only when every count is zero.

**Invocation: `apcore-cli health <module_id>`**

Calls `system.health.module` → detailed single-module output.

**TTY output:**

```
Module: db.query
Status: degraded
Calls: 1,240 total | 40 errors | 3.2% error rate
Latency: 45ms avg | 320ms p99

Recent Errors (top 3):
  MODULE_TIMEOUT     ×12  (last: 2026-04-05T10:23:00Z)
  SCHEMA_VALIDATION  ×8   (last: 2026-04-04T18:45:00Z)
  MODULE_EXECUTE     ×3   (last: 2026-04-03T09:12:00Z)
```

**Flags:**

| Flag | Default | Description |
|------|---------|-------------|
| `--threshold` | `0.01` | Error rate threshold for "healthy" status |
| `--all` | `false` | Include `healthy` modules. Only that tier is filtered — the default view still lists `degraded`, `error` **and** `unknown` modules |
| `--errors` | `10` | Max recent errors to display (module-level only) |

#### 3.2.3 `usage` Command

**Invocation: `apcore-cli usage`**

Calls `system.usage.summary`.

**TTY output:**

```
Usage Summary (last 24h)

  Module           Calls    Errors   Avg Latency   Trend
  ───────────────  ───────  ───────  ───────────   ──────
  math.add         8,420    0        2ms           stable
  db.query         1,240    40       45ms          rising ↑
  email.send       320      50       120ms         declining ↓

Total: 9,980 calls | 90 errors
```

**Invocation: `apcore-cli usage <module_id>`**

Calls `system.usage.module` → per-module breakdown with caller distribution.

**Flags:**

| Flag | Default | Description |
|------|---------|-------------|
| `--period` | `24h` | Time window: `1h`, `24h`, `7d`, `30d` |

#### 3.2.4 `enable` / `disable` Command

**Invocation:**

```
apcore-cli disable db.dangerous_migration --reason "blocking during release freeze"
apcore-cli enable  db.dangerous_migration --reason "release complete"
```

Calls `system.control.toggle_feature`. The `--reason` flag is **required** (enforced by Click).

Since `toggle_feature` has `requires_approval=True`, the standard approval gate fires — the user sees a confirmation prompt (or must pass `--yes`).

**Output:**

```
✓ Module 'db.dangerous_migration' disabled.
  Reason: blocking during release freeze
```

#### 3.2.5 `reload` Command

**Invocation:**

```
apcore-cli reload email.send --reason "updated template"
```

Calls `system.control.reload_module`. Requires approval.

**Output:**

```
✓ Module 'email.send' reloaded.
  Previous version: 1.0.0 → New version: 1.1.0
  Duration: 42ms
```

#### 3.2.6 `config` Command

**Invocation:**

```
apcore-cli config get executor.default_timeout
apcore-cli config set executor.default_timeout 60000 --reason "increase for batch jobs"
```

`config get`: Calls `Config.get(key)` directly (read-only, no approval).
`config set`: Calls `system.control.update_config` (requires approval).

**Output (get):**

```
executor.default_timeout = 30000
```

**Output (set):**

```
✓ Config updated: executor.default_timeout
  30000 → 60000
  Reason: increase for batch jobs
```

#### 3.2.7 Registration

## Contract: register_system_commands

### Inputs
- cli: click.Group, required — Root CLI group (or `apcli` sub-group) to register system commands on.
- executor: Executor, required — Executor used to probe for system module availability and to invoke system modules at runtime.

### Errors
- (none raised — silently no-ops when system modules are unavailable)

### Returns
- On success (system modules available): None — `health`, `usage`, `enable`, `disable`, `reload`, `config` commands registered on `cli`
- On system modules unavailable: None — returns without registering any commands

### Properties
- async: false
- thread_safe: false (mutates `cli` by adding commands)
- pure: false (calls `executor.validate` to probe availability; mutates `cli`)

### Per-subcommand authoritative behavior

The `register_system_commands` Contract above is the **authoritative cross-SDK reference for behavior shared by all six registered subcommands** (`health`, `usage`, `enable`, `disable`, `reload`, `config`). Audit D4-013 (2026-05-08) noted that splitting into one Contract block per subcommand was deferred to v0.10 to avoid premature spec churn while implementations stabilize. Until then, the following shared behavioral envelope applies to ALL six subcommands and any cross-SDK divergence here is a parity bug:

- **Approval gating**: `enable`, `disable`, `reload`, `config set` MUST go through `check_approval(synthetic_module_def, --yes-flag)` before invoking the executor. `health`, `usage`, `config get` are read-only and MUST NOT prompt for approval. (Audit D11-B-001, 2026-05-08, found this absent in TS — track separately.)
- **Exit-code mapping**: every subcommand maps executor errors through the canonical SDK error→exit-code table (Python `_ERROR_CODE_MAP`, Rust `map_module_error_to_exit_code`, TS `exitCodeForError`). Hard-coded `sys.exit(1)` collapse on a generic `except Exception` is a parity bug. (Audit D11-B-002, 2026-05-08, found this in Python.)
- **Output formatting**: every subcommand respects `--format` from the canonical exec set (`table | json | csv | yaml | jsonl`). `markdown` and `skill` are NOT accepted (those formats target `ScannedModule` metadata, not free-form business payloads — see `output-formatter.md`).
- **Module-availability gating**: the entire group registers atomically — either all six subcommands appear (executor returns success on `system.health.summary` probe) or none do. Partial registration is a parity bug.

When v0.10 splits these into six per-subcommand Contracts, those Contracts MUST inherit the four invariants above; any deviation requires a spec-amendment issue, not just an implementation change.

---

System commands are registered under the `apcli` built-in group (FE-13, v0.7.0+) via per-command registrars (`register_health_command`, `register_usage_command`, etc.), gated on system module availability:

```python
def register_system_commands(cli, executor):
    """Register system management commands. No-op if system modules are not enabled."""
    try:
        executor.validate("system.health.summary", {})
    except ModuleNotFoundError:
        return  # System modules not registered; skip silently
    
    cli.add_command(health_cmd)
    cli.add_command(usage_cmd)
    cli.add_command(enable_cmd)
    cli.add_command(disable_cmd)
    cli.add_command(reload_cmd)
    cli.add_command(config_cmd)
```

#### 3.2.8 Verification Tests

| ID | Test |
|----|------|
| T-SYS-01 | `health` with no modules → "No modules found" |
| T-SYS-02 | `health` shows degraded/error/unknown modules by default, `--all` adds healthy |
| T-SYS-02a | `health` summary line counts all four tiers; an all-`unknown` project reports `N unknown`, never `no data` |
| T-SYS-03 | `health <id>` displays single-module detail with recent errors |
| T-SYS-04 | `usage` displays summary table with call counts and trends |
| T-SYS-05 | `usage <id> --period 7d` passes period to system module |
| T-SYS-06 | `disable <id>` without `--reason` → Click error (missing required option) |
| T-SYS-07 | `disable <id> --reason "..."` triggers approval prompt when TTY |
| T-SYS-08 | `disable <id> --reason "..." --yes` bypasses approval |
| T-SYS-09 | `enable <id>` re-enables previously disabled module |
| T-SYS-10 | `reload <id>` shows version transition |
| T-SYS-11 | `config get <key>` returns current value |
| T-SYS-12 | `config set <key> <value> --reason "..."` triggers approval, shows old→new |
| T-SYS-13 | System commands not registered when system modules unavailable |
| T-SYS-14 | All system commands respect `--format json` |

---

### 3.3 Enhanced Error Output with AI Guidance (P0) {#error-guidance}

**Problem:** When module execution fails, the CLI displays a one-line error message. apcore errors carry `ai_guidance`, `suggestion`, `retryable`, and `user_fixable` fields that are discarded.

**apcore APIs used:**
- `ModuleError.ai_guidance: str | None`
- `ModuleError.suggestion: str | None`
- `ModuleError.retryable: bool | None`
- `ModuleError.user_fixable: bool | None`
- `ModuleError.details: dict | None`

#### 3.3.1 Error Display Format

**TTY mode:**

```
Error [SCHEMA_VALIDATION_ERROR]: Invalid table name format

  Details:
    field: table
    value: User-Info
    expected: lowercase letters and underscores only

  Suggestion: Rename 'User-Info' to 'user_info'
  Retryable: No (same input will fail again)

  Exit code: 45
```

For AI agents (non-TTY / `--format json`):

```json
{
  "error": true,
  "code": "SCHEMA_VALIDATION_ERROR",
  "message": "Invalid table name format",
  "details": {"field": "table", "value": "User-Info"},
  "suggestion": "Rename 'User-Info' to 'user_info'",
  "ai_guidance": "validate input schema before retry; this error will recur with same input",
  "retryable": false,
  "user_fixable": true,
  "exit_code": 45
}
```

#### 3.3.2 Display Rules

| Field | TTY Display | JSON Display | Condition |
|-------|-------------|--------------|-----------|
| `code` | Shown in header `[CODE]` | Always | Always |
| `message` | Shown in header | Always | Always |
| `details` | Indented key-value block | Full dict | When not None and not empty |
| `suggestion` | Labeled line | Always | When not None |
| `ai_guidance` | **Hidden** (only in JSON) | Always | When not None |
| `retryable` | "Retryable: Yes/No" with explanation | `true`/`false` | When not None |
| `user_fixable` | **Hidden** (only in JSON) | `true`/`false` | When not None |

**Rationale:** `ai_guidance` and `user_fixable` are machine-oriented; showing them to humans adds noise. `suggestion` is human-readable and always displayed.

#### 3.3.3 Implementation Change

**Current code** (`cli.py` error handler):

```python
except Exception as e:
    error_code = getattr(e, "code", None)
    exit_code = _ERROR_CODE_MAP.get(error_code, 1) if isinstance(error_code, str) else 1
    click.echo(f"Error: {e}", err=True)
    sys.exit(exit_code)
```

**New code:**

```python
except Exception as e:
    error_code = getattr(e, "code", None)
    exit_code = _ERROR_CODE_MAP.get(error_code, 1) if isinstance(error_code, str) else 1
    
    if output_format == "json" or not sys.stderr.isatty():
        _emit_error_json(e, exit_code)
    else:
        _emit_error_tty(e, exit_code)
    
    sys.exit(exit_code)


def _emit_error_json(e: Exception, exit_code: int) -> None:
    """Emit structured JSON error to stderr for AI agents."""
    payload = {"error": True, "code": getattr(e, "code", "UNKNOWN"), "message": str(e), "exit_code": exit_code}
    for field in ("details", "suggestion", "ai_guidance", "retryable", "user_fixable"):
        val = getattr(e, field, None)
        if val is not None:
            payload[field] = val
    click.echo(json.dumps(payload), err=True)


def _emit_error_tty(e: Exception, exit_code: int) -> None:
    """Emit human-readable error to stderr."""
    code = getattr(e, "code", None)
    header = f"Error [{code}]: {e}" if code else f"Error: {e}"
    click.echo(header, err=True)
    
    details = getattr(e, "details", None)
    if details:
        click.echo("", err=True)
        click.echo("  Details:", err=True)
        for k, v in details.items():
            click.echo(f"    {k}: {v}", err=True)
    
    suggestion = getattr(e, "suggestion", None)
    if suggestion:
        click.echo(f"\n  Suggestion: {suggestion}", err=True)
    
    retryable = getattr(e, "retryable", None)
    if retryable is not None:
        label = "Yes" if retryable else "No (same input will fail again)"
        click.echo(f"  Retryable: {label}", err=True)
    
    click.echo(f"\n  Exit code: {exit_code}", err=True)
```

#### 3.3.4 Verification Tests

| ID | Test |
|----|------|
| T-ERR-01 | Error with all guidance fields → TTY shows suggestion + retryable, hides ai_guidance |
| T-ERR-02 | Error with all guidance fields → JSON includes all fields |
| T-ERR-03 | Error with no guidance fields → falls back to existing one-line format |
| T-ERR-04 | Error with details dict → indented key-value display |
| T-ERR-05 | Non-ModuleError exceptions → graceful fallback, no crash |

---

### 3.4 Execution Pipeline Trace (P1) {#trace}

**Problem:** Module execution is opaque. Users cannot see which pipeline steps ran, what decisions were made, or where time was spent.

**apcore APIs used:**
- `Executor.call_with_trace(module_id, inputs, context) → (result, PipelineTrace)`
- `PipelineTrace.steps: list[StepTrace]`
- `StepTrace.name`, `.duration_ms`, `.skipped`, `.skip_reason`

#### 3.4.1 CLI Interface

```
apcore-cli <module_id> [--flags...] --trace
```

When `--trace` is specified, the CLI uses `call_with_trace()` instead of `call()`. After printing the result, it appends a pipeline trace summary to stderr.

#### 3.4.2 Trace Output

**TTY output (to stderr):**

```
{"sum": 15}

Pipeline Trace (strategy: standard, 11 steps, 23.4ms)
  ✓ context_creation       0.1ms
  ✓ call_chain_guard       0.0ms
  ✓ module_lookup          1.2ms
  ✓ acl_check              0.3ms
  ○ approval_gate          —      skipped (not required)
  ✓ middleware_before       2.1ms
  ✓ input_validation       0.4ms
  ✓ execute               18.2ms
  ✓ output_validation      0.3ms
  ✓ middleware_after        0.7ms
  ✓ return_result          0.1ms
```

**Non-TTY / JSON mode (merged into result):**

```json
{
  "result": {"sum": 15},
  "_trace": {
    "strategy": "standard",
    "total_duration_ms": 23.4,
    "success": true,
    "steps": [
      {"name": "context_creation", "duration_ms": 0.1, "skipped": false},
      {"name": "call_chain_guard", "duration_ms": 0.0, "skipped": false},
      {"name": "approval_gate", "duration_ms": 0, "skipped": true, "skip_reason": "no_match"}
    ]
  }
}
```

#### 3.4.3 Combining with `--dry-run`

`--trace --dry-run` is supported: runs `validate()` then prints which pipeline steps _would_ run (pure steps executed, impure steps shown as `○ skipped (dry_run)`).

#### 3.4.4 Verification Tests

| ID | Test |
|----|------|
| T-TRC-01 | `--trace` prints result + trace summary to stderr |
| T-TRC-02 | `--trace --format json` includes `_trace` key in output |
| T-TRC-03 | Skipped steps show skip_reason |
| T-TRC-04 | Total duration matches wall clock within 10% |
| T-TRC-05 | `--trace --dry-run` shows pure steps as executed, impure as skipped |

---

### 3.5 Standard ApprovalHandler Integration (P1) {#approval}

**Problem:** CLI hardcodes a simplified approval flow (TTY prompt + 60s timeout). apcore defines a full `ApprovalHandler` protocol with callbacks, async polling, and token-based resume — none of which the CLI uses.

**apcore APIs used:**
- `ApprovalHandler` protocol (`request_approval`, `check_approval`)
- `ApprovalRequest` / `ApprovalResult` dataclasses
- `_approval_token` input mechanism
- `CallbackApprovalHandler`
- `AutoApproveHandler`

#### 3.5.1 Design: CLI as ApprovalHandler

The CLI implements `ApprovalHandler` by wrapping its existing TTY prompt logic into the protocol interface:

```python
class CliApprovalHandler:
    """ApprovalHandler that prompts in TTY, auto-denies in non-TTY (unless bypassed)."""
    
    def __init__(self, auto_approve: bool = False, timeout: int = 60):
        self.auto_approve = auto_approve
        self.timeout = timeout
    
    async def request_approval(self, request: ApprovalRequest) -> ApprovalResult:
        if self.auto_approve:
            return ApprovalResult(status="approved", approved_by="auto_approve")
        
        if not sys.stdin.isatty():
            return ApprovalResult(status="rejected", reason="Non-interactive session without --yes")
        
        # TTY prompt with configurable timeout
        message = request.annotations.extra.get("approval_message") or _default_message(request)
        approved = _tty_prompt_with_timeout(message, self.timeout)
        
        if approved:
            return ApprovalResult(status="approved", approved_by="tty_user")
        return ApprovalResult(status="rejected", reason="User rejected")
    
    async def check_approval(self, approval_id: str) -> ApprovalResult:
        return ApprovalResult(status="rejected", reason="CLI does not support async approval polling")
```

The handler is passed to Executor at construction:

```python
handler = CliApprovalHandler(auto_approve=yes_flag, timeout=approval_timeout)
executor = Executor(registry, approval_handler=handler)
```

#### 3.5.2 New: Configurable Timeout

```
apcore-cli <module_id> --approval-timeout 120    # 120 seconds
```

Config key: `cli.approval_timeout` (default: 60).
Env var: `APCORE_CLI_APPROVAL_TIMEOUT`.

#### 3.5.3 New: Approval Token Resume

When an external approval system returns `status="pending"` with an `approval_id`, the CLI prints:

```
Approval pending. To resume after external approval:
  apcore-cli <module_id> [--flags...] --approval-token <token>
```

The `--approval-token` flag maps to `inputs["_approval_token"]`, which apcore recognizes and routes to `check_approval()` instead of `request_approval()`.

#### 3.5.4 New: Custom Handler via Config

For advanced users who want webhook-based or external approval:

```yaml
# apcore.yaml
apcore-cli:
  approval_handler: webhook
  approval_webhook_url: https://approvals.internal/api/review
```

Built-in handler types:
- `tty` (default) — interactive terminal prompt
- `auto` — always approve (equivalent to `--yes`)
- `webhook` — POST ApprovalRequest to URL, poll for result

#### 3.5.5 Migration Path

The existing `check_approval()` function in `approval.py` becomes a thin wrapper:

```python
# Before: check_approval(module_def, auto_approve)  # 2-arg form, pre-v0.6.0
# After:  handler.request_approval(ApprovalRequest(...))

# For backward compatibility, check_approval() delegates to the handler.
# The canonical v0.6.0 signature adds an explicit `timeout` parameter.
# The shim constructs a fresh CliApprovalHandler per call; the executor API
# does not expose `_approval_handler`, so handler state is not retrieved
# from the executor. In the normal create_cli() path, the module-level
# handler is wired onto Executor(registry, approval_handler=handler)
# and the Executor drives the approval flow directly.
def check_approval(module_def, auto_approve, timeout=60):
    handler = CliApprovalHandler(auto_approve=auto_approve, timeout=timeout)
    request = ApprovalRequest(
        module_id=module_def.module_id,
        arguments={},
        context=None,
        annotations=module_def.annotations,
        description=module_def.description,
    )
    result = asyncio.run(handler.request_approval(request))
    if result.status != "approved":
        raise ApprovalDeniedError(...)
```

#### 3.5.6 Verification Tests

| ID | Test |
|----|------|
| T-APR-01 | CliApprovalHandler auto-approves with `--yes` |
| T-APR-02 | CliApprovalHandler rejects in non-TTY without `--yes` |
| T-APR-03 | CliApprovalHandler prompts in TTY with custom timeout |
| T-APR-04 | `--approval-timeout 5` sets 5-second timeout |
| T-APR-05 | `--approval-token <token>` passes `_approval_token` to inputs |
| T-APR-06 | Pending approval prints resume instructions with token |
| T-APR-07 | Webhook handler POSTs to configured URL and polls |
| T-APR-08 | `APCORE_CLI_APPROVAL_TIMEOUT` env var overrides default |

---

### 3.6 Streaming Output (P1) {#streaming}

**Problem:** Large results are buffered entirely before output. Modules that produce incremental results (log analysis, data transformation) cannot stream to the terminal.

**apcore APIs used:**
- `Executor.stream(module_id, inputs, context) → AsyncIterator[dict]`
- `ModuleAnnotations.streaming: bool`

#### 3.6.1 CLI Interface

```
apcore-cli <module_id> [--flags...] --stream
```

#### 3.6.2 Behavior

1. When `--stream` is specified, CLI checks `annotations.streaming`:
   - `True` → use `Executor.stream()`, yield each chunk to stdout immediately
   - `False` → log WARNING: "Module does not declare streaming support. Falling back to standard execution.", use `Executor.call()`

2. Each chunk is written as a JSON line (JSONL) to stdout, flushed immediately:

```
{"partial": true, "chunk": 1, "data": {"row": "..."}}
{"partial": true, "chunk": 2, "data": {"row": "..."}}
{"partial": false, "chunk": 3, "data": {"summary": "3 rows processed"}}
```

3. In TTY mode, a spinner or progress line updates on stderr:

```
Streaming math.transform... (3 chunks received)
```

4. `--stream` always outputs JSONL, regardless of `--format`. Streaming and table formatting are mutually exclusive. If `--stream` is set and `--format table` is specified, the CLI logs WARNING: "Streaming mode always outputs JSONL; --format table is ignored." and proceeds with JSONL output.

#### 3.6.3 Implementation Note

Since `Executor.stream()` is async, the CLI uses `asyncio.run()` to drive the iterator:

```python
async def _stream_and_print(executor, module_id, inputs, format):
    chunks = 0
    async for chunk in executor.stream(module_id, inputs):
        chunks += 1
        if format == "json" or not sys.stdout.isatty():
            click.echo(json.dumps(chunk, default=str))
        else:
            # Update progress on stderr
            click.echo(f"\rStreaming {module_id}... ({chunks} chunks)", err=True, nl=False)
    if sys.stdout.isatty():
        click.echo("", err=True)  # newline after progress
```

#### 3.6.4 Verification Tests

| ID | Test |
|----|------|
| T-STR-01 | `--stream` on streaming module → JSONL output, one line per chunk |
| T-STR-02 | `--stream` on non-streaming module → WARNING + fallback to call() |
| T-STR-03 | Each chunk is flushed immediately (no buffering) |
| T-STR-04 | Stream error mid-flight → partial output preserved, error on stderr |
| T-STR-05 | `--stream --format json` emits JSONL |

---

### 3.7 Enhanced Discovery (P1) {#discovery}

**Problem:** `apcore-cli list` only supports exact tag filtering. No search, no status/annotation filtering, no sorting.

The enhanced list flags (`--search`, `--status`, `--annotation`, `--sort`, `--reverse`, `--deprecated`, `--deps`) are documented in the canonical location: [Discovery feature spec §Enhanced List Flags](discovery.md#enhanced-list-flags-v060).

**apcore APIs used:**
- `ModuleDescriptor`: all fields (annotations, tags, enabled, deprecated, dependencies)
- `ModuleAnnotations`: destructive, requires_approval, readonly, streaming, cacheable

**Verification tests:** T-LST-01 through T-LST-08 — defined in [discovery.md §8.1](discovery.md#81-enhanced-list-filter-tests-fe-11-37--referenced-from-usability-enhancementsmd).

---

### 3.8 Execution Strategy Selection (P2) {#strategy}

**Problem:** CLI always uses the standard 11-step pipeline. Developers testing locally may want to skip ACL/approval. Performance-sensitive batch scripts may want to skip middleware.

**apcore APIs used:**
- `Executor(registry)` constructed with a strategy via the Executor's own factory (see tech-design ADR-03); the strategy is **not** passed as a parameter to `Executor.call()` — `call(module_id, validated_input)` is always 2-arg
- Preset strategies: `standard`, `internal`, `testing`, `performance`

#### 3.8.1 CLI Interface

```
apcore-cli <module_id> [--flags...] --strategy testing
```

Config key: `cli.strategy` (default: `standard`).
Env var: `APCORE_CLI_STRATEGY`.

| Strategy | Steps Skipped | Use Case |
|----------|--------------|----------|
| `standard` | None | Production (default) |
| `internal` | ACL + Approval | Trusted internal scripts |
| `testing` | ACL + Approval + Call Chain Guard | Local development / test |
| `performance` | All middleware (before + after) | Latency-sensitive batch |
| `minimal` | All except context_creation, module_lookup, execute, return_result | Pre-validated internal hot paths (apcore 0.17.1+) |

#### 3.8.2 Safety Guard

When `--strategy` is not `standard` and stdout is TTY, print a warning to stderr:

```
⚠ Using 'testing' strategy — ACL and approval checks are skipped.
```

#### 3.8.3 `describe-pipeline` Subcommand

```
apcore-cli describe-pipeline [--strategy standard]
```

Shows the step list for a strategy:

```
Pipeline: standard (11 steps)

  #   Step                  Pure   Removable  Timeout
  ──   ─────────────────────  ────   ─────────  ───────
   1   context_creation       yes    no         —
   2   call_chain_guard       yes    yes        —
   3   module_lookup          yes    no         —
   4   acl_check              yes    yes        —
   5   approval_gate          no     yes        —
   6   middleware_before       no     yes        —
   7   input_validation       yes    no         —
   8   execute                no     no         30000ms
   9   output_validation      yes    yes        —
  10   middleware_after        no     yes        —
  11   return_result          no     no         —
```

#### 3.8.4 Verification Tests

| ID | Test |
|----|------|
| T-STG-01 | `--strategy testing` skips ACL and approval |
| T-STG-02 | `--strategy performance` skips middleware |
| T-STG-03 | Non-standard strategy prints warning to stderr on TTY |
| T-STG-04 | `APCORE_CLI_STRATEGY=internal` sets default strategy |
| T-STG-05 | `describe-pipeline` lists all steps for a strategy |

---

### 3.9 Output Format Extensions (P2) {#output-formats}

**Problem:** Only `json` and `table` formats are available. AI agents need JSONL for streaming integration. DevOps scripts need CSV. Humans may prefer YAML.

#### 3.9.1 New Format Options

Extend `--format` to accept:

| Format | Description | When |
|--------|-------------|------|
| `json` | Single JSON object (existing) | Default for non-TTY |
| `table` | Rich table (existing) | Default for TTY |
| `jsonl` | JSON Lines (one object per line) | Streaming, log pipelines |
| `csv` | Comma-separated values | Spreadsheet import, `list` output |
| `yaml` | YAML format | Human-readable config-like output |

#### 3.9.2 Field Selection

```
apcore-cli list --format csv --fields id,description,tags
apcore-cli <module_id> --fields result.status,result.count
```

`--fields` accepts a comma-separated list of dot-paths. For `list`, valid fields are: `id`, `description`, `tags`, `group`, `status`, `annotations`. For exec results, fields are dot-paths into the result JSON.

#### 3.9.3 Verification Tests

| ID | Test |
|----|------|
| T-FMT-01 | `--format csv` produces valid CSV with header row |
| T-FMT-02 | `--format yaml` produces valid YAML |
| T-FMT-03 | `--format jsonl` produces one JSON object per line |
| T-FMT-04 | `--fields id,tags` limits output columns |
| T-FMT-05 | Unknown format name → exit 2 with valid choices listed |

---

### 3.10 Multi-Level Nested Grouping (P2) {#multi-level}

**Problem:** Current grouping splits on the first `.` only, creating a flat two-level hierarchy. Projects with 3+ namespace levels (e.g., `admin.user.create`, `admin.user.delete`, `admin.role.list`) produce cluttered second-level menus.

#### 3.10.1 Design

**New config key:** `cli.group_depth` (default: `1`, max: `3`).

| `group_depth` | Module `admin.user.create` | CLI command |
|---------------|---------------------------|-------------|
| 1 (current) | group=`admin`, cmd=`user.create` | `apcore-cli admin user.create` |
| 2 | group=`admin` → subgroup=`user`, cmd=`create` | `apcore-cli admin user create` |
| 3 | same as 2 for 3-segment IDs | `apcore-cli admin user create` |

**Resolution algorithm:**

1. Split module_id (or CLI alias) by `.`.
2. Take up to `min(group_depth, len(segments) - 1)` segments as the group path.
3. Remaining segments joined by `.` become the command name.
4. `display.cli.group` override still takes precedence.

**Fallback:** If `group_depth > 1` but a module_id has only 2 segments, it still uses single-level grouping.

#### 3.10.2 Verification Tests

| ID | Test |
|----|------|
| T-DEPTH-01 | `group_depth=2` with `admin.user.create` → `admin user create` |
| T-DEPTH-02 | `group_depth=2` with `math.add` (2 segments) → `math add` (single level) |
| T-DEPTH-03 | `group_depth=1` (default) behaves identically to current implementation |
| T-DEPTH-04 | `display.cli.group` override still takes precedence over auto-grouping |

> **Note:** the `T-GRP-*` test ID prefix is reserved by [grouped-commands.md §7](grouped-commands.md) for FE-09 tests. The four `group_depth` tests above were renumbered from `T-GRP-*` to `T-DEPTH-*` to avoid collision with the FE-09 suite (T-GRP-01..T-GRP-18).

---

### 3.11 Custom Command Extension Point (P2) {#extensions}

**Problem:** Downstream projects that embed `apcore-cli` via `create_cli()` cannot add arbitrary custom commands alongside module commands.

#### 3.11.1 Design

Add `extra_commands` parameter to `create_cli()`:

```python
# Full signature: see tech-design §8.2.7 "Consolidated Signature"
# This feature adds the `extra_commands` parameter:
create_cli(..., extra_commands: list[click.Command | click.Group] | None = None)
```

**Cross-language equivalents:**

| Language | API |
|----------|-----|
| Python | `create_cli(extra_commands=[deploy_cmd, migrate_cmd])` |
| TypeScript | `createCli({ extraCommands: [deployCmd, migrateCmd] })` |
| Rust | `CliConfig { extra_commands: vec![deploy_cmd, migrate_cmd], .. }` |

**Registration order:** Extra commands are registered **after** built-in commands and **before** module commands. Name collisions with built-in commands (`list`, `describe`, `health`, etc.) raise `ValueError` at startup.

#### 3.11.2 Example

```python
import click
from apcore_cli import create_cli

@click.command()
@click.argument("environment")
def deploy(environment):
    """Deploy to target environment."""
    click.echo(f"Deploying to {environment}...")

cli = create_cli(
    extensions_dir="./extensions",
    prog_name="myapp",
    extra_commands=[deploy],
)
```

```bash
myapp deploy production
myapp math.add --a 1 --b 2    # module command still works
myapp list                     # built-in still works
```

#### 3.11.3 Verification Tests

| ID | Test |
|----|------|
| T-EXT-01 | Extra command appears in `--help` output |
| T-EXT-02 | Extra command is callable and receives arguments |
| T-EXT-03 | Extra command name collision with built-in → ValueError |
| T-EXT-04 | Extra command group works for nested subcommands |

---

## 4. Summary: New CLI Commands and Flags

### 4.1 New Built-in Commands

| Command | Section | Priority |
|---------|---------|----------|
| `validate <module_id>` | §3.1 | P0 |
| `health [<module_id>]` | §3.2 | P0 |
| `usage [<module_id>]` | §3.2 | P0 |
| `enable <module_id>` | §3.2 | P0 |
| `disable <module_id>` | §3.2 | P0 |
| `reload <module_id>` | §3.2 | P0 |
| `config get/set <key>` | §3.2 | P0 |
| `describe-pipeline` | §3.8 | P2 |

### 4.2 New Execution Flags

| Flag | Section | Priority |
|------|---------|----------|
| `--dry-run` | §3.1 | P0 |
| `--trace` | §3.4 | P1 |
| `--stream` | §3.6 | P1 |
| `--strategy <name>` | §3.8 | P2 |
| `--approval-timeout <seconds>` | §3.5 | P1 |
| `--approval-token <token>` | §3.5 | P1 |

### 4.3 New Discovery Flags

| Flag | Section | Priority |
|------|---------|----------|
| `--search` / `-s` | §3.7 | P1 |
| `--status` | §3.7 | P1 |
| `--annotation` / `-a` | §3.7 | P1 |
| `--sort` | §3.7 | P1 |
| `--reverse` | §3.7 | P1 |
| `--deprecated` | §3.7 | P1 |
| `--deps` | §3.7 | P1 |

### 4.4 New Output Flags

| Flag | Section | Priority |
|------|---------|----------|
| `--format csv\|yaml\|jsonl` | §3.9 | P2 |
| `--fields` | §3.9 | P2 |

### 4.5 New Config Keys

| Key | Env Var | Default | Section |
|-----|---------|---------|---------|
| `cli.approval_timeout` | `APCORE_CLI_APPROVAL_TIMEOUT` | `60` | §3.5 |
| `cli.strategy` | `APCORE_CLI_STRATEGY` | `standard` | §3.8 |
| `cli.group_depth` | `APCORE_CLI_GROUP_DEPTH` | `1` | §3.10 |

---

## 5. apcore API Utilization Impact

| apcore API | Before | After |
|------------|--------|-------|
| `Executor.validate()` | Not used | §3.1 dry-run |
| `Executor.call_with_trace()` | Not used | §3.4 trace |
| `Executor.stream()` | Not used | §3.6 streaming |
| `Executor(strategy=...)` | Not used | §3.8 strategy |
| `ApprovalHandler` protocol | Not used | §3.5 approval |
| `ModuleError.ai_guidance` | Discarded | §3.3 error output |
| `ModuleError.suggestion` | Discarded | §3.3 error output |
| `ModuleError.retryable` | Discarded | §3.3 error output |
| `ModuleAnnotations.*` | Only `requires_approval` | §3.7 filtering |
| `ModuleDescriptor.deprecated` | Not used | §3.7 filtering |
| `ModuleDescriptor.dependencies` | Not used | §3.7 deps column |
| `system.health.*` | Not used | §3.2 health command |
| `system.usage.*` | Not used | §3.2 usage command |
| `system.control.*` | Not used | §3.2 enable/disable/reload/config |

**Estimated utilization after implementation: ~55-60%** (up from ~15-20%).
