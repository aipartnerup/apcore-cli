---
description: "TTY-aware Human-in-the-Loop approval gate (FE-03) that intercepts modules with requires_approval, supporting --yes/env bypass, non-TTY exit 46, and a 60s timeout."
---

# Feature Spec: Approval Gate

**Feature ID**: FE-03
**Status**: Ready for Implementation
**Priority**: P1
**Parent**: [Tech Design v2.0](../tech-design.md) Section 8.4
**SRS Requirements**: FR-APPR-001, FR-APPR-002, FR-APPR-003, FR-APPR-004, FR-APPR-005

---

## 1. Description

The Approval Gate is a TTY-aware Human-in-the-Loop (HITL) middleware that intercepts module execution when `annotations.requires_approval` is `true`. It prompts interactive users for confirmation, blocks non-interactive callers without a bypass, supports `--yes` and `APCORE_CLI_AUTO_APPROVE=1` bypass mechanisms, and enforces a 60-second timeout on TTY prompts.

---

## 2. Requirements Traceability

| Req ID | SRS Ref | Description |
|--------|---------|-------------|
| FR-03-01 | FR-APPR-001 | Check `annotations.requires_approval` field. |
| FR-03-02 | FR-APPR-002 | TTY interactive prompt with `[y/N]` default deny. |
| FR-03-03 | FR-APPR-003 | Non-TTY rejection with exit 46 and help message. |
| FR-03-04 | FR-APPR-004 | `--yes` flag and `APCORE_CLI_AUTO_APPROVE=1` bypass. |
| FR-03-05 | FR-APPR-005 | 60-second timeout on TTY prompts. |

---

## 3. Module Path

`apcore_cli/approval.py`

---

## 4. Implementation Details

## Contract: check_approval

### Inputs
- module_def: Any, required — Module definition object. Must carry `annotations.requires_approval` (bool) to trigger the gate. If absent or not a boolean `True`, approval is skipped.
- auto_approve: bool, required — When `True`, bypasses the prompt (corresponds to `--yes` flag).
  validates: must be bool; always bool from Click
- timeout: int, optional — Seconds before the TTY prompt times out. Default: `60`. Range: `1..3600` (clamped to bounds).
  validates: must be int in valid range

### Errors
- `ApprovalDeniedError` — when approval is denied (user types n/Enter) or stdin is non-TTY with no bypass available. Maps to **exit code 46** when bubbled to the CLI top-level.
- `ApprovalTimeoutError` — when the TTY prompt does not receive a response within `timeout` seconds. Maps to **exit code 46**.
- (Rust: the same conditions surface as `ApprovalError::Denied`, `ApprovalError::Timeout`, and `ApprovalError::NonInteractive` enum variants. Python/TS fold the non-TTY case into `ApprovalDeniedError`.)

> **Migration note (post D11-001):** Earlier revisions of this spec declared `SystemExit(46)` directly. Implementations now raise typed errors so callers (e.g., the CLI dispatcher) can flush the audit log before mapping to the process exit code. Callers MUST translate `ApprovalDeniedError` and `ApprovalTimeoutError` to exit 46 at the top-level entry point.

### Returns
- On success (approved or not required): `None`
- On denial / non-TTY: raises `ApprovalDeniedError` (caller maps to exit 46)
- On timeout: raises `ApprovalTimeoutError` (caller maps to exit 46)

### Properties
- async: false
- thread_safe: false (uses signal.alarm on Unix — not safe to call from multiple threads)
- pure: false (prompts terminal, reads env vars, may raise typed errors that callers translate to process exit)

---

## Contract: CliApprovalHandler.check_approval

### Inputs
- module_def: Any, required — Module definition object; same semantics as `check_approval`.
- auto_approve: bool, required — Bypass flag from `--yes` or programmatic override.
- timeout: int, optional — Prompt timeout in seconds. Default: `60`.

### Errors
- `ApprovalDeniedError` — approval denied (user input) or stdin non-TTY without bypass. Maps to exit code 46.
- `ApprovalTimeoutError` — prompt timed out. Maps to exit code 46.
- (Rust: surfaces as `ApprovalError::{Denied, Timeout, NonInteractive}`.)

### Returns
- On success: `None`
- On denial / non-TTY: raises `ApprovalDeniedError`
- On timeout: raises `ApprovalTimeoutError`

### Properties
- async: false
- thread_safe: false
- pure: false

---

## Contract: CliApprovalHandler.request_approval

### Inputs
- module_def: Any, required — Module definition to request approval for.
- context: Any, optional — Execution context (unused in v0.7, reserved for future async approval flows).

### Errors
- `ApprovalDeniedError` — approval denied or stdin non-TTY without bypass. Maps to exit code 46.
- `ApprovalTimeoutError` — prompt timed out. Maps to exit code 46.
- (Rust: surfaces as `ApprovalError::{Denied, Timeout, NonInteractive}`.)

### Returns
- On approval granted: `None`
- On denial / non-TTY: raises `ApprovalDeniedError`
- On timeout: raises `ApprovalTimeoutError`

### Properties
- async: false
- thread_safe: false
- pure: false (prompts terminal, may call sys.exit)

---

### 4.1 Function: `check_approval`

**Signature**: `check_approval(module_def: Any, auto_approve: bool, timeout: int = 60) -> None`

**Returns**: `None` if approved (or approval not required). Raises `SystemExit` if denied/timed out/pending.

Logic steps:
1. Read `annotations = getattr(module_def, "annotations", None)`.
2. If `annotations is None`, or is neither a `dict` nor an object with a `requires_approval` attribute: return (no approval needed).
3. Read `requires`: if `annotations` is a `dict`, use `annotations.get("requires_approval", False)`; otherwise use `getattr(annotations, "requires_approval", False)`.
4. If `requires` is not exactly `True` (boolean): return (skip). This handles `"true"` string, `1` int, `None`, etc.
5. Check bypass mechanisms in priority order:
   a. If `auto_approve is True` (from `--yes` flag):
      - Log INFO: "Approval bypassed via --yes flag for module '{module_id}'."
      - Return.
   b. Read `env_val = os.environ.get("APCORE_CLI_AUTO_APPROVE", "")`.
      - If `env_val == "1"`:
        - Log INFO: "Approval bypassed via APCORE_CLI_AUTO_APPROVE for module '{module_id}'."
        - Return.
      - If `env_val != ""` and `env_val != "1"`:
        - Log WARNING: "APCORE_CLI_AUTO_APPROVE is set to '{env_val}', expected '1'. Ignoring."
6. Check TTY status: `is_tty = sys.stdin.isatty()`.
7. If `not is_tty`:
   - Log ERROR: "Non-interactive environment, no bypass provided for module '{module_id}'."
   - Raise `ApprovalDeniedError("Module '{module_id}' requires approval but no interactive terminal is available. Use --yes or set APCORE_CLI_AUTO_APPROVE=1 to bypass.")`. The CLI top-level catches this, flushes any pending audit records, writes the message to stderr, and exits with code 46. (Rust surfaces this case as `ApprovalError::NonInteractive` instead of `ApprovalError::Denied`.)
8. If `is_tty`:
   - Call `_prompt_with_timeout(module_def)`.

### 4.2 Function: `_prompt_with_timeout`

**Signature**: `_prompt_with_timeout(module_def: ModuleDefinition, timeout: int = 60) -> None`

Logic steps:
1. Extract `message = module_def.annotations.get("approval_message", None)`.
2. If `message is None`: set `message = f"Module '{module_def.canonical_id}' requires approval to execute."`.
3. Display `message` to terminal via `click.echo(message, err=True)`.
4. Set up timeout:
   - **Unix (POSIX)**: Use `signal.alarm(timeout)` with `signal.signal(signal.SIGALRM, _timeout_handler)`.
   - **Windows**: Use `threading.Timer(timeout, _timeout_interrupt)`.
5. Try:
   a. Call `approved = click.confirm("Proceed?", default=False)`.
   b. Cancel the alarm/timer.
   c. If `approved`:
      - Log INFO: "User approved execution of module '{module_id}'."
      - Return.
   d. If not `approved`:
      - Log WARNING: "Approval rejected by user for module '{module_id}'."
      - Raise `ApprovalDeniedError("Approval denied.")`. The CLI top-level catches it, flushes pending audit records, writes the message to stderr, and exits with code 46.
6. On timeout (SIGALRM or timer fires):
   - Log WARNING: "Approval timed out after {timeout}s for module '{module_id}'."
   - Raise `ApprovalTimeoutError(f"Approval prompt timed out after {timeout} seconds.")`. The CLI top-level catches it, flushes pending audit records, writes the message to stderr, and exits with code 46.

### 4.3 Timeout Handler

**Unix:**
```python
def _timeout_handler(signum, frame):
    raise ApprovalTimeoutError()
```

**Windows:**
```python
def _timeout_interrupt():
    import ctypes
    ctypes.pythonapi.PyThreadState_SetAsyncExc(
        ctypes.c_ulong(main_thread_id),
        ctypes.py_object(ApprovalTimeoutError)
    )
```

### 4.4 Custom Exceptions

```python
class ApprovalDeniedError(Exception):
    """Raised when approval is denied — user input, or non-TTY with no bypass."""
    pass


class ApprovalTimeoutError(Exception):
    """Raised when the approval prompt times out."""
    pass
```

In Rust, the equivalent set is encoded as variants of a single `ApprovalError` enum (`Denied`, `Timeout`, `NonInteractive`). The Python/TS `ApprovalDeniedError` covers both Rust's `Denied` and `NonInteractive` cases.

---

## 5. Parameter Validation

| Parameter | Type | Valid Values | Invalid Handling |
|-----------|------|-------------|------------------|
| `module_def.annotations` | `dict \| None` | Dict with `requires_approval` key, or `None`. | If `None`: skip approval. If not dict: skip approval. |
| `annotations.requires_approval` | `bool` | `True` or `False` | Any non-boolean (string `"true"`, int `1`, `None`): treat as `False`, skip. |
| `auto_approve` | `bool` | `True` or `False` | Always boolean from Click. |
| `APCORE_CLI_AUTO_APPROVE` | `str` (env var) | `"1"` to activate bypass. | `"true"`, `"yes"`, `"0"`, empty string: not bypass. Log WARNING for non-empty non-"1" values. |
| `timeout` | `int` | `1..3600` | Default: 60. Out of range: clamp to bounds. |

---

## 6. Flow Diagram

```
check_approval(module_def, auto_approve)
  |
  +-- annotations.requires_approval != true? --> RETURN (proceed)
  |
  +-- --yes flag? --> Log bypass --> RETURN (proceed)
  |
  +-- APCORE_CLI_AUTO_APPROVE == "1"? --> Log bypass --> RETURN (proceed)
  |
  +-- APCORE_CLI_AUTO_APPROVE set but != "1"? --> Log WARNING (continue to TTY check)
  |
  +-- Non-TTY? --> stderr: "requires approval but no interactive terminal" --> EXIT 46
  |
  +-- TTY? --> Display message + "Proceed? [y/N]:"
       |
       +-- User types "y" within 60s --> Log approved --> RETURN (proceed)
       |
       +-- User types "n" or Enter within 60s --> Log denied --> EXIT 46
       |
       +-- 60s timeout --> Log timeout --> EXIT 46
```

---

## 7. Error Handling

| Condition | Exit Code | Error Message | SRS Reference |
|-----------|-----------|---------------|---------------|
| Approval denied (user typed n/N/Enter) | 46 | "Error: Approval denied." | FR-APPR-002 AF-1 |
| Approval timeout (60s) | 46 | "Error: Approval prompt timed out after 60 seconds." | FR-APPR-005 AF-1 |
| Non-TTY, no bypass | 46 | "Error: Module '{id}' requires approval but no interactive terminal is available. Use --yes or set APCORE_CLI_AUTO_APPROVE=1 to bypass." | FR-APPR-003 |

---

## 8. Verification

| Test ID | Description | Expected Result |
|---------|-------------|-----------------|
| T-APPR-01 | Module with `requires_approval: true` in TTY, user types `y` | Execution proceeds. Log contains "approved". |
| T-APPR-02 | Module with `requires_approval: true` in TTY, user types `n` | Exit 46. stderr: "Approval denied." |
| T-APPR-03 | Module with `requires_approval: true` in TTY, user presses Enter | Exit 46. Default is deny (N). |
| T-APPR-04 | Module with `requires_approval: true` in non-TTY, no bypass | Exit 46. stderr: "no interactive terminal". |
| T-APPR-05 | Module with `requires_approval: true`, `--yes` flag | Execution proceeds. Log: "bypassed via --yes flag". |
| T-APPR-06 | Module with `requires_approval: true`, `APCORE_CLI_AUTO_APPROVE=1` | Execution proceeds. Log: "bypassed via APCORE_CLI_AUTO_APPROVE". |
| T-APPR-07 | `APCORE_CLI_AUTO_APPROVE=true` (not "1") | WARNING logged. Bypass NOT active. Falls through to TTY/non-TTY check. |
| T-APPR-08 | Module with `requires_approval: false` | No prompt. Execution proceeds immediately. |
| T-APPR-09 | Module with no `annotations` field | No prompt. Execution proceeds immediately. |
| T-APPR-10 | TTY prompt, 60s timeout | Exit 46. stderr: "timed out after 60 seconds". |
| T-APPR-11 | Both `--yes` and `APCORE_CLI_AUTO_APPROVE=1` set | `--yes` takes priority. Log: "bypassed via --yes flag". |
| T-APPR-12 | Module with custom `approval_message` | Custom message displayed before prompt. |
| T-APPR-13 | Module with no `approval_message` | Default message: "Module '{id}' requires approval to execute." |
