---
description: "Security Manager (FE-05) sub-package: API key auth for remote registries, keyring + AES-256-GCM encrypted config, JSON Lines audit logging, and subprocess execution sandboxing."
---

# Feature Spec: Security Manager

**Feature ID**: FE-05
**Status**: Ready for Implementation
**Priority**: P1 (Auth, Encryption, Audit) / P2 (Sandbox)
**Parent**: [Tech Design v2.0](../tech-design.md) Section 8.6
**SRS Requirements**: FR-SEC-001, FR-SEC-002, FR-SEC-003, FR-SEC-004

---

## 1. Description

The Security Manager is a sub-package providing four security components: API key authentication for remote registries, encrypted configuration storage using OS keyring with AES-256-GCM fallback, append-only JSON Lines audit logging of all module executions, and subprocess-based execution sandboxing for untrusted modules.

---

## 2. Requirements Traceability

| Req ID | SRS Ref | Description |
|--------|---------|-------------|
| FR-05-01 | FR-SEC-001 | API key authentication for remote registries. |
| FR-05-02 | FR-SEC-002 | Encrypted configuration storage (keyring + AES-256-GCM). |
| FR-05-03 | FR-SEC-003 | Audit logging in JSON Lines format. |
| FR-05-04 | FR-SEC-004 | Subprocess-based execution sandboxing. |

---

## 3. Module Paths

```
apcore_cli/security/
├── __init__.py            # Exports: AuthProvider, ConfigEncryptor, AuditLogger, Sandbox
├── auth.py                # API key authentication
├── config_encryptor.py    # Keyring + AES-256-GCM fallback
├── audit.py               # JSON Lines audit logging
└── sandbox.py             # Subprocess isolation
```

---

## 4. Implementation Details

## Contract: Sandbox._sandboxed_execute

### Inputs
- module_id: str, required — Canonical module identifier to execute in the sandbox.
- input_data: dict, required — JSON-serializable input dict passed to the module via stdin.

### Errors
- ModuleExecutionError(stderr_content) — when the subprocess exits non-zero
- ModuleExecutionError("sandbox output exceeds size limit") — when stdout exceeds 64 MiB
- ModuleExecutionError("Module '{id}' timed out in sandbox.") — when subprocess exceeds 300 s timeout
- Rust language note: surfaces these as `ModuleExecutionError::{NonZeroExit, OutputParseFailed, Timeout}`; additionally exposes `ModuleExecutionError::{SpawnFailed, ModuleError}` for language-specific failure modes (subprocess spawn, in-process executor passthrough error preservation). Mirrors the cross-language note pattern used for `AuthenticationError::MalformedApiKey` in `AuthProvider.authenticate_request`.

### Returns
- On success: Any — parsed JSON from subprocess stdout
- On failure: raises ModuleExecutionError (never returns None)

### Properties
- async: false
- thread_safe: false (spawns subprocess; each call creates a temp dir)
- pure: false (spawns subprocess, creates temporary directory, may read filesystem)

### Algorithm

```
_sandboxed_execute(module_id, input_data):
  # 1. Build the environment from two explicit sources. Never a bulk copy of
  #    os.environ — caller-defined secrets must not leak.
  env = {}
  for key in SANDBOX_ALLOW_KEYS:           # PATH, LANG, LC_ALL
    if key in os.environ: env[key] = os.environ[key]
  for key, val in os.environ.items():      # framework config the child needs
    if key.startswith(SANDBOX_ALLOW_PREFIX)         # "APCORE_"
       and not key.startswith(SANDBOX_DENY_PREFIX)  # "APCORE_AUTH_"
       and key not in SANDBOX_DENY_KEYS:            # {"APCORE_AUTH_API_KEY"}
      env[key] = val

  # 2. Create per-call tempdir (auto-removed on context exit / Drop).
  with TemporaryDirectory() as tmp:
    env["TMPDIR"] = tmp                    # subprocess sees its own scratch
    env["HOME"]  = tmp                     # confine accidental ~/ writes

    # 2b. Propagate and absolutize EVERY path-typed var before injection. The
    #     child's cwd is `tmp` (step 4), so a relative value silently re-roots
    #     there. Resolution is against the PARENT's cwd.
    root = self.extensions_root or env.get("APCORE_EXTENSIONS_ROOT")
    if root is None:
      raise ModuleExecutionError("sandbox: extensions root not propagated")
    env["APCORE_EXTENSIONS_ROOT"] = absolute(root)
    # Step 1's APCORE_ pass-through forwards these VERBATIM, so each must be
    # re-absolutized here or the pass-through silently defeats invariant 6.
    for key in SANDBOX_PATH_TYPED_VARS:      # {"APCORE_ACL_ROOT", ...}
      if key in env: env[key] = absolute(env[key])

    # 3. Resolve module entry-point. Outside-extensions-root resolution rejects
    #    via Sandbox.with_extensions_root prefix check (audit D1-004).
    cmd = locate_executable(module_id)
    if not within(cmd, self.extensions_root): raise ModuleExecutionError("escape")

    # 4. Spawn with bounded stdin / stdout / stderr.
    proc = Popen([cmd], env=env, cwd=tmp,
                 stdin=PIPE, stdout=PIPE, stderr=PIPE,
                 timeout=300)               # SIGTERM-then-SIGKILL ladder
    stdout, stderr = proc.communicate(json.dumps(input_data).encode())
                                            # buffered up to max_output_bytes
                                            # (default 64 MiB; configurable via
                                            # Sandbox.with_max_output_bytes — D1-004)

    # 5. Output-size guard — per-stream, raise BEFORE returning, even on
    #    exit_code==0. Each stream is bounded independently so a module
    #    cannot evade the cap by splitting payload across stdout/stderr
    #    (audit D10-001, 2026-05-08).
    if len(stdout) > self.max_output_bytes:
      raise ModuleExecutionError("sandbox stdout exceeds size limit")
    if len(stderr) > self.max_output_bytes:
      raise ModuleExecutionError("sandbox stderr exceeds size limit")

    # 6. Non-zero exit → raise with stderr body for the caller's error message.
    if proc.returncode != 0:
      raise ModuleExecutionError(stderr.decode(errors="replace"))

    # 7. Parse JSON or raise. (TempDir auto-removes on context unwind even on
    #    exception path — guarantee preserved across all 3 SDKs.)
    return json.loads(stdout)
```

**Cross-SDK-critical invariants** (every implementation must preserve):
1. Environment is built from **two explicit sources** and is never a bulk copy of the parent environment: a fixed key allowlist (`PATH`, `LANG`, `LC_ALL`) plus an `APCORE_`-prefix pass-through, minus the `APCORE_AUTH_` deny prefix and the explicit deny keys. The prefix pass-through is the one place inheritance is permitted — the child needs the framework's own configuration to function at all — and the deny prefix is what keeps credentials out of it. Adding a fixed key, or widening or narrowing either prefix, requires updating all 3 SDKs.

   *Corrected 2026-09-06.* This invariant previously read "allowlist-built, NOT inherited+filtered" over a fixed list of `PATH, HOME, LANG, LC_*, TZ, USER, TMPDIR`. Both halves were wrong. All three SDKs implement, and have long implemented, exactly the prefix-based inherit-and-filter the old wording forbade — necessarily, since `APCORE_EXTENSIONS_ROOT` must reach the child and was not even in the list the same section called "inherited" (§4.4 builder methods). `HOME` and `TMPDIR` are *assigned* to the tempdir in step 2, not inherited, and `TZ` / `USER` / `LC_*` are forwarded by no SDK. The spec was stale, not the implementations.
2. Tempdir cleanup MUST run on the exception path (Python `TemporaryDirectory` context, Rust `tempfile::TempDir` Drop, TS `fs.rm({recursive:true})` in finally).
3. Output-size check fires BEFORE return and is **per-stream** — `len(stdout) > cap` and `len(stderr) > cap` are checked independently. A 0-exit module producing 100 MiB on either pipe is rejected. Audit D1-004 `with_max_output_bytes` (2026-05); audit D10-001 per-stream alignment (2026-05-08).
4. Timeout uses SIGTERM-first / SIGKILL-fallback (300 s default); never raw SIGKILL on first signal.
5. `extensions_root` confinement check happens BEFORE spawn — never resolve symlinks via subprocess. Audit D1-004 `with_extensions_root`.
6. **Path-typed values crossing the sandbox boundary MUST be propagated, absolutized, and never guessed by the child.** Three obligations; any one alone leaves the default configuration broken, which is why this is a single invariant rather than three.
   - **Absolutize in the parent.** A path-typed env var MUST be resolved against the **parent's** cwd before injection. The child's cwd is the per-call tempdir (step 4), so a relative value silently re-roots there.
   - **Propagate at every call site.** The parent MUST hand its **resolved** extensions root to `Sandbox` on **every** sandbox path. Absolutizing a variable that was never set fixes nothing, and a call site that constructs `Sandbox` without supplying the root is the common form of this defect — it is invisible to any test that configures an absolute root.
   - **Fail loudly in the child.** The child MUST NOT fall back to a relative default when the variable is absent; it MUST exit non-zero with the same named error (`sandbox: extensions root not propagated`). A child-side default of `./extensions` resolves against the tempdir and surfaces as `CONFIG_NOT_FOUND` on a directory the operator never named. The parent's pre-spawn check is the normal path; the child's check covers a runner invoked directly and a future parent/child mismatch, so both are required and neither substitutes for the other.

   **The set is explicit, not inferred.** `SANDBOX_PATH_TYPED_VARS` names every `APCORE_*` variable whose value is a filesystem path — currently `APCORE_EXTENSIONS_ROOT` and `APCORE_ACL_ROOT`. It exists because step 1's `APCORE_`-prefix pass-through forwards matching variables **verbatim**: absolutizing only the one the sandbox happens to know about leaves every other path-typed variable to re-root in the tempdir, which is invariant 6 defeated by the mechanism that implements invariant 1. Adding a path-typed `APCORE_*` config key requires adding it to this set in all 3 SDKs.

   `acl.root` is in that set for a reason worth stating. [`acl-governance.md` §4.10](acl-governance.md) permits forwarding `APCORE_ACL_ROOT` into the child as defence in depth; without absolutization that forwarding is a no-op that *looks* like protection. A relative root re-roots to the tempdir, the path does not exist, and §4.2's "missing path attaches nothing" invariant then leaves the child with **no ACL at all** — fail-open, not fail-closed. The parent-side gate §4.10 mandates is what actually protects that path; this clause only stops the optional second layer from being illusory.

---

## Contract: Sandbox.execute

### Inputs
- module_id: str, required — Canonical module identifier.
- input_data: dict, required — Input dict for the module.
- executor: Executor, required — Executor instance used in non-sandbox path.

### Errors
- ModuleExecutionError — from `_sandboxed_execute` when sandbox is enabled and subprocess fails
- (any error from `executor.call` when sandbox is disabled)

### Returns
- On success: Any — module result (passed through from executor or sandboxed subprocess)
- On failure: raises ModuleExecutionError or executor-specific error

### Properties
- async: depends_on_language — Python sync; Rust async (tokio subprocess I/O); TS see implementation. Mirrors the language-idiom split documented for `AuthProvider.authenticate_request` below: the spec acknowledges that subprocess and in-process executor I/O are blocking on Python's stdlib but await-driven on tokio, and embedders MUST treat the return as awaitable in Rust and as a direct return in Python.
- thread_safe: false (sandbox path spawns subprocess; non-sandbox path inherits executor thread-safety)
- pure: false (I/O: calls executor or spawns subprocess)

---

## Contract: AuthProvider.authenticate_request

### Inputs
- headers: dict, required — HTTP headers dict to augment with the Authorization header.

### Errors
- `AuthenticationError("Remote registry requires authentication. Set --api-key, APCORE_AUTH_API_KEY, or auth.api_key in config.")` — when no API key is available (exit code 77).
- `AuthenticationError("Malformed API key: contains invalid characters (CR or LF). Re-export the variable or update the config without trailing newlines.")` — when the resolved API key contains a carriage return (`\r`) or line feed (`\n`) character. This is a defensive guard against header-injection / smuggling vectors that occur when shells append a stray newline (e.g., `export APCORE_AUTH_API_KEY="$(...)"` where the inner command emits a trailing newline). Maps to exit code 77. Rust surfaces this as `AuthenticationError::MalformedApiKey`.

### Returns
- On success: dict — the input headers dict with `"Authorization": "Bearer {key}"` added (in-place mutation; the same reference is returned).
- On no key or malformed key: raises AuthenticationError

### Properties
- async: depends_on_language — Python and Rust expose `authenticate_request` as synchronous (the underlying keyring APIs on those platforms are blocking). TypeScript exposes it as `async` because Node.js `keytar` is asynchronous. Callers MUST treat the return as awaitable in TypeScript and as a direct return in Python/Rust. A sync façade in TypeScript (e.g., a `warm()` method that pre-loads the cached API key into a private field, after which `authenticate_request` is synchronous) is acceptable as an alternative implementation strategy if cross-language sync parity is desired by an embedder.
- thread_safe: true (reads config and env; no mutable shared state)
- pure: false (reads env vars and config file)

---

## Contract: ConfigEncryptor.aes_encrypt (internal: `_aes_encrypt`)

### Inputs
- plaintext: str, required — UTF-8 string to encrypt.

### Errors
- (cryptography library errors propagate — unexpected; input is always valid UTF-8 str)

### Returns
- On success: bytes — `16-byte salt || 12-byte nonce || 16-byte GCM tag || ciphertext` (enc:v2 wire format)

### Properties
- async: false
- thread_safe: false (uses OS random bytes; safe to call from single thread)
- pure: false (reads OS entropy for salt and nonce)

---

## Contract: ConfigEncryptor.store

### Inputs
- key: str, required — Config key name used as the keyring label (e.g., `"auth.api_key"`).
- value: str, required — Plaintext secret value to store.

### Errors
- keyring errors propagate if keyring.set_password fails unexpectedly
  - **Cross-language error-wrap note**: the wrap shape diverges by SDK convention.
    - Python: re-raises the raw `keyring` exception (no wrap).
    - TypeScript: catches and rethrows as `ConfigDecryptionError` with prepended message `"Failed to store '{key}' in OS keyring: ..."`.
    - Rust: maps any keyring failure to `ConfigDecryptionError::KeyringError(String)` carrying the underlying error's display string.
    All three preserve the underlying cause and surface a non-zero exit; the divergence is in the exception class wrapping, not in the observable failure mode.

### Returns
- On success with keyring: str — `"keyring:{key}"` reference string
- On success without keyring: str — `"enc:v2:{base64}"` ciphertext reference string

### Properties
- async: false
- thread_safe: false (keyring calls are not guaranteed thread-safe)
- pure: false (writes to OS keyring or derives encryption key using hostname/username)

---

## Contract: ConfigEncryptor.retrieve

### Inputs
- config_value: str, required — The stored reference value (prefixed with `keyring:`, `enc:v2:`, or `enc:`).
- key: str, required — Config key name for error messages.
  validates: must start with `keyring:`, `enc:v2:`, or `enc:` to trigger decryption; otherwise returned as-is

### Errors
- ConfigDecryptionError(f"Keyring entry not found for '{ref_key}'.") — when keyring lookup returns None
- ConfigDecryptionError(f"Failed to decrypt configuration value '{key}'. Re-configure with 'apcore-cli config set {key}'.") — on decryption failure (wrong key, corrupted ciphertext, or `InvalidTag`)

### Returns
- On success: str — plaintext value
- On failure: raises ConfigDecryptionError (exit code 47)

### Properties
- async: false
- thread_safe: false (keyring and PBKDF2 calls are not thread-safe across instances)
- pure: false (reads OS keyring or derives key from hostname/username/PBKDF2)

### Algorithm

```
retrieve(key, ref):
  # `ref` is the stored reference string. Three on-wire shapes:
  #   "keyring:<name>"   → opaque keyring entry; plaintext after lookup.
  #   "enc:v2:<b64>"     → AES-GCM(salt, nonce, tag, ct); current format.
  #   "enc:<hex|b64>"    → legacy v1 envelope; PBKDF2-HMAC-SHA256(hostname).

  # 1. Plain keyring path (no encryption).
  if ref.startswith("keyring:"):
    name = ref[len("keyring:"):]
    val = keyring.get_password("apcore-cli", name)
    if val is None:
      raise ConfigDecryptionError(f"Keyring entry not found for '{name}'.")
    return val

  # 2. AES-GCM v2 (current format).
  if ref.startswith("enc:v2:"):
    blob = b64decode(ref[len("enc:v2:"):])
    salt, nonce, tag, ct = blob[:16], blob[16:28], blob[28:44], blob[44:]
    derived_key = PBKDF2_HMAC_SHA256(host_user_pepper(), salt, iter=600_000, length=32)
    try:
      return AESGCM(derived_key).decrypt(nonce, ct + tag, aad=None).decode("utf-8")
    except InvalidTag:
      raise ConfigDecryptionError(user_facing_message(key))   # audit D10-truncated #1
                                                               # — never expose
                                                               # InvalidTag verbatim

  # 3. Legacy v1 envelope (back-compat read-only; new writes go through v2).
  if ref.startswith("enc:"):
    blob = decode_legacy_envelope(ref[len("enc:"):])
    derived_key = legacy_v1_key_from(hostname(), username())
    try:
      return AES_decrypt(derived_key, blob)
    except Exception:
      raise ConfigDecryptionError(user_facing_message(key))

  # 4. None of the prefixes matched — programmer error (caller didn't gate on
  #    is_encrypted_ref). Distinct from a decryption failure.
  raise ConfigDecryptionError(f"Unknown reference format for '{key}'.")
```

**Cross-SDK-critical invariants** (every implementation must preserve):
1. **Order of try-paths is keyring → enc:v2: → enc:** (longest prefix first). `enc:v2:` must be checked BEFORE the bare `enc:` prefix or v2 references would mis-route into the legacy path. Audit D10-truncated #1.
2. The user-facing error string for a decryption failure MUST NOT leak the underlying cryptography exception name (`InvalidTag`, `InvalidSignature`, etc.). Always re-raise as the canonical `ConfigDecryptionError` with the spec-mandated message. Audit D10-truncated #1 (2026-05).
3. PBKDF2 iteration count: **v2 = 600 000** (OWASP 2024+, used on all new writes); **v1 legacy read = 100 000** (fallback for values encrypted by SDK ≤v0.6). Never write new v2 values with 100 000 iterations — that weakens cross-host-portable secrets and breaks round-trip.
4. `keyring:` is plaintext-after-lookup; the keyring backend is the security boundary. Do NOT attempt to AES-decrypt a `keyring:` value.
5. v1 (`enc:` no-version-tag) is read-only and write-deprecated. New `set` calls always emit `enc:v2:`. The v1 branch exists strictly for migration of pre-2025 configs.

---

## Contract: AuthProvider.__init__

### Inputs
- config: ConfigResolver, required — Config resolver used to look up `auth.api_key` and (transitively) the bound `ConfigEncryptor`.
- encryptor: ConfigEncryptor | None, optional — Explicit encryptor override. Default: `None` (the encryptor reachable through `config` is used when an encrypted reference is resolved).

### Errors
- (none raised by constructor itself)

### Returns
- On success: AuthProvider instance

### Properties
- async: false
- thread_safe: true (constructor only stores references; performs no I/O)
- pure: true (no I/O, no mutation of inputs)
- **Rust language note**: Rust splits this into two factories — `AuthProvider::new(config)` (1 param, encryptor=None) and `AuthProvider::with_encryptor(config, encryptor)` (2 params). This is the idiomatic Rust pattern for a constructor with one optional argument; both factories produce the same behavioral result as Python `AuthProvider(config, encryptor=None)` and TypeScript `new AuthProvider(config, encryptor?)`. Mirrors the cross-language note pattern used for `AuthenticationError::MalformedApiKey` in `AuthProvider.authenticate_request`.

---

## Contract: AuthProvider.get_api_key

### Inputs
- (no parameters — uses instance state)

### Errors
- ConfigDecryptionError — propagated from `ConfigEncryptor.retrieve` when an `enc:` / `enc:v2:` / `keyring:` reference fails to decrypt or look up. Maps to exit code 47.

### Returns
- On success: str | None — resolved (and decrypted, when stored as `keyring:` or `enc:`) API key, or `None` when no source provided one.
- Resolution chain (first non-empty wins): `--api-key` CLI flag → `APCORE_AUTH_API_KEY` env var → `auth.api_key` in config (decrypted via `ConfigEncryptor.retrieve` when prefixed with `keyring:` or `enc:`).

### Properties
- async: false
- thread_safe: true (reads CLI flag, env var, and config; relies on `ConfigEncryptor.retrieve` thread-safety, which is per-instance)
- pure: false (reads env vars, config file, OS keyring)

---

## Contract: AuthProvider.handle_response

### Inputs
- status_code: int, required — HTTP status code returned by the remote registry.

### Errors
- AuthenticationError("Authentication failed. Verify your API key.") — when `status_code` is `401` or `403`. Maps to exit code 77.

### Returns
- On success: None (any status other than 401/403 is treated as success for auth-handling purposes; downstream code is responsible for non-auth status handling).

### Properties
- async: false
- thread_safe: true (no shared state read or written)
- pure: true (no I/O; result depends only on the input status_code)

---

## Contract: AuditLogger.__init__

### Inputs
- path: Path | None, optional — Override path for the JSON Lines audit log. Default: `None` — resolves to `AuditLogger.DEFAULT_PATH` (`Path.home() / ".apcore-cli" / "audit.jsonl"`).

### Errors
- OSError — propagated only when the parent-directory `mkdir(parents=True, exist_ok=True)` cannot create the directory (e.g., read-only filesystem, permission denied). Implementations MAY swallow this and defer the failure to the first `log_execution` call to keep CLI startup robust; the canonical Python implementation chooses to propagate.

### Returns
- On success: AuditLogger instance with `_path` resolved and parent directory created.

### Properties
- async: false
- thread_safe: true (single-instance construction; the directory-creation call is idempotent under `exist_ok=True`)
- pure: false (creates parent directory on disk if missing — observable filesystem side effect)

---

## Contract: AuditLogger.log_execution

### Inputs
- module_id: str, required — Canonical module identifier (already validated upstream against `^[a-z][a-z0-9_]*(\.[a-z][a-z0-9_]*)*$`).
- input_data: dict, required — Module input payload. Hashed for the audit entry (never stored verbatim).
- status: Literal["success", "error"], required — Outcome classification.
  - validation: must be one of `"success"` or `"error"`
  - reject_with: ValueError (caller-side; not enforced by this method)
- exit_code: int, required — Process exit code, expected range `0..=255`.
- duration_ms: int, required — Wall-clock duration in milliseconds, expected `>= 0`.

### Errors
- (none raised) — `OSError` from the underlying append-write is intentionally swallowed: the method logs a WARNING to stderr (`"Warning: Could not write audit log: {e}."`) and returns. Audit-log unavailability MUST NOT abort module execution, because the audit log is an after-the-fact security record, not a precondition.

### Returns
- On success: None — appends one JSON Lines entry `{timestamp, user, module_id, input_hash, status, exit_code, duration_ms}` to `self._path`.
- On disk error: None — WARNING logged to stderr; no entry written; no raise.

### Properties
- async: false
- thread_safe: false — this is a security-gated surface; concurrent callers MUST serialize externally. The reference Python implementation does not hold a lock around the append-write, so two concurrent calls may interleave bytes in the file. Per-call salt (`secrets.token_bytes(16)`) for the input-hash means concurrent callers do not share hashing state.
- pure: false (reads OS user identity, generates per-call random salt, appends to the audit file).

---

## Contract: Sandbox.__init__

### Inputs
- enabled: bool, optional — Toggle for subprocess isolation. Default: `False` (sandbox disabled — `execute` delegates directly to the supplied `Executor`).
- timeout: int, optional — Per-call subprocess timeout in seconds. Default: `300` (5 minutes).

### Errors
- (none raised by constructor itself)

### Returns
- On success: Sandbox instance with `_enabled`, `_timeout`, `_extensions_root` (None), and `_max_output_bytes` (None) set.

### Properties
- async: false
- thread_safe: true (constructor only stores primitive flags / option values)
- pure: true (no I/O, no mutation of inputs)

### Builder methods (fluent style)
- `with_extensions_root(path)` — sets the canonical extensions root for the sandbox subprocess. Takes precedence over the inherited `APCORE_EXTENSIONS_ROOT` env var; injected as that env var into the subprocess, **absolutized against the parent's cwd** (invariant 6). A relative `path` is accepted and resolved here rather than rejected, so callers may pass through whatever the FE-07 chain produced.

  **Callers MUST supply it.** The CLI's resolved extensions root comes from the FE-07 chain (`--extensions-dir` > `APCORE_EXTENSIONS_ROOT` > `apcore.yaml` `extensions.root` > `./extensions`), and its default is *relative*. A `Sandbox` constructed without this builder, and run without `APCORE_EXTENSIONS_ROOT` in the environment, has nothing to inject; per invariant 6 it MUST then fail before spawn with `sandbox: extensions root not propagated`. It MUST NOT spawn and leave the child to guess — that is the defect this contract exists to prevent, and the child is required to refuse the same way if it is ever reached without the variable.

  Every sandbox call site — business-module dispatch **and** `apcli exec` — must call this builder. The two paths are wired independently and one being correct hides nothing about the other, so the divergence is invisible unless a test runs `apcli exec --sandbox` under a *relative* extensions root (T-SEC-19).
- `with_max_output_bytes(n)` — overrides the per-stream output cap; replaces the default 64 MiB limit. `None` (the default) preserves the 64 MiB cap.

Each builder returns the same `Sandbox` instance (Python / TypeScript) or a moved owned value (Rust) so calls can be chained. All three SDKs expose the same fluent surface.

Example (canonical fluent style):
- Python:  `Sandbox(enabled=True, timeout=30).with_extensions_root("/srv/exts").with_max_output_bytes(2 * 1024 * 1024)`
- Rust:    `Sandbox::new(true, 30).with_extensions_root(Some(PathBuf::from("/srv/exts"))).with_max_output_bytes(2 * 1024 * 1024)`
- TS:      `new Sandbox(true, 30).withExtensionsRoot("/srv/exts").withMaxOutputBytes(2 * 1024 * 1024)`

---

### 4.1 AuthProvider (`auth.py`)

**Class**: `AuthProvider`

**Constructor**: `__init__(self, config: ConfigResolver)`

**Method: `get_api_key() -> str | None`**

Logic steps:
1. Call `self._config.resolve("auth.api_key", cli_flag="--api-key", env_var="APCORE_AUTH_API_KEY")`.
2. If result is not None and starts with `keyring:` or `enc:`: call `self._config.encryptor.retrieve(result, "auth.api_key")`.
3. Return the resolved (and possibly decrypted) API key, or `None`.

**Method: `authenticate_request(headers: dict) -> dict`**

Logic steps:
1. Call `key = self.get_api_key()`.
2. If `key is None`: raise `AuthenticationError("Remote registry requires authentication. Set --api-key, APCORE_AUTH_API_KEY, or auth.api_key in config.")`.
3. If `"\r" in key` or `"\n" in key`: raise `AuthenticationError("Malformed API key: contains invalid characters (CR or LF). Re-export the variable or update the config without trailing newlines.")`. This is a defensive guard against header injection / shell-newline-bleed (a stray `\n` from `export VAR="$(...)"` would otherwise produce a multi-line `Authorization` header).
4. Mutate `headers` in place: `headers["Authorization"] = f"Bearer {key.strip()}"`.
5. Return `headers` — the same reference as the input. Callers that share the headers dict observe the new `Authorization` entry without re-reading the return value.

**Method: `handle_response(status_code: int) -> None`**

Logic steps:
1. If `status_code == 401` or `status_code == 403`: raise `AuthenticationError("Authentication failed. Verify your API key.")`.
2. Otherwise: return (no action).

| Scenario | Behavior | Exit Code |
|----------|----------|-----------|
| API key from `--api-key abc123` | `Authorization: Bearer abc123` | — |
| API key from `APCORE_AUTH_API_KEY=abc123` | `Authorization: Bearer abc123` | — |
| API key from config file (encrypted) | Decrypted, used as Bearer token | — |
| No API key, remote registry configured | `AuthenticationError` | 77 |
| HTTP 401 response | `AuthenticationError("Authentication failed")` | 77 |
| HTTP 403 response | `AuthenticationError("Authentication failed")` | 77 |
| Local-only registry | `get_api_key()` returns None, but not called | — |

### 4.2 ConfigEncryptor (`config_encryptor.py`)

**Class**: `ConfigEncryptor`

**Constant**: `SERVICE_NAME = "apcore-cli"`

> **Constructor fallibility (audit D10-W4, 2026-05-12).** The Rust constructor `ConfigEncryptor::new() -> Result<Self, ConfigDecryptionError>` is fallible — it validates that at least one of {keyring availability, AES-256-GCM support} is reachable before returning. Python `ConfigEncryptor()` and TypeScript `new ConfigEncryptor()` are infallible by convention; the same validation is deferred to the first `store()` / `retrieve()` call. The error class (`ConfigDecryptionError`) is the same across all three SDKs — only the point at which it surfaces differs. Mirrors the cross-language constructor-shape note pattern used for `AuthProvider.__init__` (Python/TS unary, Rust split into `new()` / `with_encryptor()`).

**Method: `store(key: str, value: str) -> str`**

Logic steps:
1. Call `self._keyring_available()`.
2. If `True`:
   a. Call `keyring.set_password(self.SERVICE_NAME, key, value)`.
   b. Return `f"keyring:{key}"`.
3. If `False`:
   a. Log WARNING: "OS keyring unavailable. Using file-based encryption."
   b. Call `ciphertext = self._aes_encrypt(value)`.
   c. Return `f"enc:v2:{base64.b64encode(ciphertext).decode()}"`.

**Method: `retrieve(config_value: str, key: str) -> str`**

Logic steps:
1. If `config_value.startswith("keyring:")`:
   a. Extract `ref_key = config_value[len("keyring:"):]`.
   b. Call `result = keyring.get_password(self.SERVICE_NAME, ref_key)`.
   c. If `result is None`: raise `ConfigDecryptionError(f"Keyring entry not found for '{ref_key}'.")`.
   d. Return `result`.
2. If `config_value.startswith("enc:v2:")`:
   a. `data = base64.b64decode(config_value[len("enc:v2:"):])`
   b. Try `return self._aes_decrypt(data)` (v2: salt embedded in data).
   c. On `InvalidTag` or `ValueError`: raise `ConfigDecryptionError(f"Failed to decrypt configuration value '{key}'. Re-configure with 'apcore-cli config set {key}'.")`.
3. If `config_value.startswith("enc:")`:
   a. `ciphertext = base64.b64decode(config_value[len("enc:"):])`
   b. Try decryption using static salt `b"apcore-cli-config-v1"` + 600k iterations.
   c. If decryption fails, retry with 100k iterations (values written by SDK v0.6 and earlier).
   d. On both failures: raise `ConfigDecryptionError(f"Failed to decrypt configuration value '{key}'. Re-configure with 'apcore-cli config set {key}'.")`.
4. Otherwise: return `config_value` (plaintext, legacy or non-sensitive).

**Method: `_keyring_available() -> bool`**

Logic steps:
1. Try: `kr = keyring.get_keyring()`.
2. If `isinstance(kr, keyring.backends.fail.Keyring)`: return `False`.
3. Return `True`.
4. On any exception: return `False`.

**Method: `_derive_key() -> bytes`**

Logic steps:
1. `hostname = socket.gethostname()` (or OS equivalent).
2. `username = USER ?? LOGNAME ?? USERNAME ?? "unknown"` — canonical four-tier env-var fallback (audit D11-W1, 2026-05-08). Order is fixed so v1/v2 ciphertexts encrypted under any platform decrypt cross-SDK: `USER` covers POSIX, `LOGNAME` covers some POSIX login shells, `USERNAME` covers Windows. Python: `os.getenv("USER") or os.getenv("LOGNAME") or os.getenv("USERNAME") or "unknown"`.
3. `salt = random 16-byte value` — generated per encryption and embedded in the ciphertext.
4. `material = f"{hostname}:{username}".encode()`.
5. Return PBKDF2-HMAC-SHA256(`material`, `salt`, iterations=**600_000**).

Output: 32-byte AES-256 key.

> **Security note**: 600,000 iterations follows OWASP 2024+ recommendations for PBKDF2-HMAC-SHA256. The salt is per-encryption and random (not static), removing the v1 machine-binding limitation.

**Method: `_aes_encrypt(plaintext: str) -> bytes`**

Logic steps:
1. `salt = random 16-byte value` — random per encryption.
2. `key = self._derive_key(salt)` — PBKDF2 with the per-encryption salt.
3. `nonce = random 12-byte value` — 96-bit nonce for GCM.
4. AES-256-GCM encrypt `plaintext.encode("utf-8")`.
5. Return `salt (16 bytes) + nonce (12 bytes) + GCM tag (16 bytes) + ciphertext`.

**Method: `_aes_decrypt(data: bytes) -> str`**

Logic steps:
1. `salt = data[:16]`, `nonce = data[16:28]`, `tag = data[28:44]`, `ct = data[44:]`.
2. `key = self._derive_key(salt)`.
3. AES-256-GCM decrypt with `(nonce, tag, ct)`.
4. Return decoded UTF-8 string.

**Wire formats:**

```
enc:v2:<base64 of (16-byte salt || 12-byte nonce || 16-byte GCM tag || ciphertext)>
```

The `enc:v2:` prefix identifies the per-encryption-salt format (written by all three SDKs).

Legacy format (read-only, written by SDK v0.6 and earlier):
```
enc:<base64 of (12-byte nonce || 16-byte GCM tag || ciphertext)>
```
When reading `enc:` (v1), implementations MUST try decryption with 600k iterations + static salt `b"apcore-cli-config-v1"` first; if decryption fails they MUST re-try with 100k iterations + same static salt (for values encrypted by very early SDK versions). On re-encrypt (e.g., `config set`), the implementation MUST write the v2 format.

**Dependencies**: `keyring >= 24.0`, `cryptography >= 41.0`.

### 4.3 AuditLogger (`audit.py`)

**Class**: `AuditLogger`

**Constant**: `DEFAULT_PATH = Path.home() / ".apcore-cli" / "audit.jsonl"`

**Constructor**: `__init__(self, path: Path | None = None)`

Logic steps:
1. `self._path = path or self.DEFAULT_PATH`.
2. Call `self._ensure_directory()`: `self._path.parent.mkdir(parents=True, exist_ok=True)`.

**Method: `log_execution(module_id, input_data, status, exit_code, duration_ms) -> None`**

> **Cross-SDK signature parity (audit D10-W2, 2026-05-12; return-type clarified 2026-05-13).** All three SDKs use the **same 5-parameter signature** `log_execution(module_id, input_data, status, exit_code, duration_ms)`. Return type is infallible in all three (Python `-> None`, TypeScript `: void`, Rust `-> ()`): I/O failures are swallowed and surfaced as a one-shot `tracing::warn!` / `logger.warning` per the D11-010 "write-failure dedup" contract, never propagated to the caller. The earlier spec note suggesting Rust returned `Result<(), AuditLogError>` was stale — `AuditLogError` is defined in `apcore-cli-rs/src/audit.rs` for future use but is not currently surfaced. The emitted JSONL entry is byte-identical across languages.

| Parameter | Type | Validation |
|-----------|------|------------|
| `module_id` | `str` | Canonical ID format (already validated upstream). |
| `input_data` | `dict` | Any dict. Serialized for hashing only (not stored). |
| `status` | `Literal["success", "error"]` | Must be one of two values. |
| `exit_code` | `int` | 0-255. |
| `duration_ms` | `int` | >= 0. |

Logic steps:
1. Build `entry` dict:
   - `"timestamp"`: `datetime.now(UTC).isoformat(timespec="milliseconds").replace("+00:00", "Z")`.
   - `"user"`: call `self._get_user()`.
   - `"module_id"`: `module_id`.
   - `"input_hash"`: `hashlib.sha256(salt + json.dumps(input_data, sort_keys=True).encode()).hexdigest()` where `salt = secrets.token_bytes(16)`. A fresh 16-byte random salt is generated per invocation to prevent cross-invocation input correlation.
   - `"status"`: `status`.
   - `"exit_code"`: `exit_code`.
   - `"duration_ms"`: `duration_ms`.
2. Try: open `self._path` in append mode, write `json.dumps(entry) + "\n"`.
3. On `OSError as e`: log WARNING to stderr "Warning: Could not write audit log: {e}." Do NOT raise or exit.

**Method: `_get_user() -> str`**

Logic steps:
1. Try: return `os.getlogin()`.
2. On `OSError`: try `pwd.getpwuid(os.getuid()).pw_name` (Unix only).
3. On `ImportError` / `KeyError` / `AttributeError`: return `USER ?? LOGNAME ?? USERNAME ?? "unknown"` — same canonical four-tier env-var fallback used by `ConfigEncryptor._derive_key` so audit-log `user` field and encryption material agree on the same identity across SDKs (audit D11-W1, 2026-05-08).

**Audit entry example:**
```json
{"timestamp":"2026-03-14T10:30:45.123Z","user":"tercelyi","module_id":"math.add","input_hash":"a1b2c3d4...","status":"success","exit_code":0,"duration_ms":42}
```

### 4.4 Sandbox (`sandbox.py`)

**Class**: `Sandbox`

**Constructor**: `__init__(self, enabled: bool = False)`

**Method: `execute(module_id: str, input_data: dict, executor: Executor) -> Any`**

> **Cross-SDK parity note (audit D10 follow-up, 2026-05-13):** All three SDKs use the same 3-parameter form for `execute(module_id, input_data, executor)`. Rust's signature is `async fn execute(&self, module_id: &str, input_data: Value, executor: &apcore::Executor) -> Result<Value, ModuleExecutionError>`. The Rust sandbox additionally binds `APCORE_EXTENSIONS_ROOT` at construction time via the `withExtensionsRoot` builder, but the per-call `executor` argument is still passed in (and used to drive the non-sandbox passthrough path). The emitted error types are identical across languages.

Logic steps:
1. If `not self._enabled`: return `executor.call(module_id, input_data)` (no sandbox).
2. If `self._enabled`: call `self._sandboxed_execute(module_id, input_data)`.

**Method: `_sandboxed_execute(module_id: str, input_data: dict) -> Any`**

Logic steps:
1. Build restricted environment dict using prefix-allow + deny strategy:
   - Allow: `PATH`, `LANG`, `LC_ALL` (locale / executable lookup).
   - Allow prefix: all `APCORE_*` variables from host.
   - Deny prefix: `APCORE_AUTH_*` (auth credentials must not cross trust boundary).
   - **`PYTHONPATH` (and any other `*PATH` variable that influences module / package resolution — e.g., `NODE_PATH`, `RUBYLIB`, `GOPATH`, `LD_LIBRARY_PATH`) MUST NOT cross the sandbox boundary, regardless of language.** Rationale: an attacker-controlled module-resolution path can inject a sibling import path the host did not authorize, defeating the trust boundary. The hardened reference impl (`apcore-cli-python/src/apcore_cli/security/sandbox.py`) defines `_SANDBOX_ALLOW_KEYS = ("PATH", "LANG", "LC_ALL")` and explicitly omits `PYTHONPATH`; future SDKs MUST follow this convention.
   - Omit all other environment variables.
2. Create temporary directory (platform temp location; prefix `apcore_sandbox_`).
3. Set `HOME` and `TMPDIR` to temporary directory.
3b. **Propagate and absolutize path-typed variables (invariant 6).** Resolve the extensions root as `self._extensions_root` else the inherited `APCORE_EXTENSIONS_ROOT`; if neither is present, raise `ModuleExecutionError("sandbox: extensions root not propagated")` **before** spawning. Absolutize it against the **parent's** cwd and set it in `env`. Then absolutize every other variable in `SANDBOX_PATH_TYPED_VARS` (currently `APCORE_ACL_ROOT`) that step 1's prefix pass-through carried in — the pass-through forwards them verbatim, so skipping this re-roots them to the tempdir.
4. Serialize `input_data` to JSON string.
5. Spawn the **language-specific sandbox runner entry point** as a child process:
   - Python: `[sys.executable, "-m", "apcore_cli._sandbox_runner", module_id]`
   - TypeScript/Node: `[process.execPath, "<pkg_runner_script>", "--internal-sandbox-runner", module_id]`
   - Rust: `[std::env::current_exe(), "--internal-sandbox-runner", module_id]`
   - Pass serialized JSON to stdin.
   - Capture stdout (result) and stderr (error messages).
   - Timeout: **300 seconds** (5 minutes).
   - **The runner MUST NOT default a missing `APCORE_EXTENSIONS_ROOT`.** It reads the variable and, when absent, exits non-zero with `sandbox: extensions root not propagated` on stderr. It MUST NOT fall back to `./extensions`, which its own cwd (the tempdir) would resolve to a directory the operator never named. Step 3b makes this unreachable on the normal path; the runner check covers direct invocation and any future parent/child mismatch, and is verified independently (T-SEC-21a).
6. If process exits non-zero: raise `ModuleExecutionError(stderr_content)`.
7. If `len(stdout) > 64 MiB` OR `len(stderr) > 64 MiB`: raise `ModuleExecutionError("sandbox <stream> exceeds size limit")` (per-stream — audit D10-001, 2026-05-08).
8. Parse stdout as JSON and return.
9. Cleanup: temporary directory is auto-deleted.

**Language-specific sandbox runner** (invoked as child process):

Each language port ships a runner entry point that reads `module_id` from argv, deserializes JSON input from stdin, instantiates a fresh `Registry` + `Executor` in the restricted environment, calls the module, and writes JSON result to stdout. The runner MUST NOT read from the host filesystem beyond the extensions directory.

**Restricted environment contents:**

| Variable | Source | Purpose |
|----------|--------|---------|
| `PATH` | Host | Locate runtime executable |
| `LANG` | Host | Locale |
| `LC_ALL` | Host | Locale |
| `APCORE_*` | Host (excluding `APCORE_AUTH_*`) | apcore configuration |
| `HOME` | Temp dir | Prevent access to real home |
| `TMPDIR` | Temp dir | Isolate temp files |

**Explicitly excluded** (security-gated; MUST NOT cross the sandbox boundary): `PYTHONPATH`, `NODE_PATH`, `RUBYLIB`, `GOPATH`, `LD_LIBRARY_PATH`, and any other `*PATH` variable that influences module / package / shared-library resolution. Allowing such variables would let a module inject a sibling import path the host did not authorize. The canonical allow-list lives in `apcore-cli-python/src/apcore_cli/security/sandbox.py` as `_SANDBOX_ALLOW_KEYS` and is the reference all language ports must match.

---

## 5. Error Handling

| Condition | Exit Code | Error Message | SRS Reference |
|-----------|-----------|---------------|---------------|
| No API key for remote registry | 77 | "Error: Remote registry requires authentication..." | FR-SEC-001 AF-1 |
| Authentication failed (401/403) | 77 | "Error: Authentication failed. Verify your API key." | FR-SEC-001 AF-2 |
| Config decryption failure | 47 | "Error: Failed to decrypt configuration value '{key}'..." | FR-SEC-002 AF-2 |
| Audit log write failure | — (WARNING) | "Warning: Could not write audit log: {reason}." | FR-SEC-003 AF-1 |
| Sandbox subprocess failure | 1 | "Error: Module '{id}' execution failed: {stderr}." | FR-SEC-004 |
| Sandbox timeout (5 min) | 1 | "Error: Module '{id}' timed out in sandbox." | FR-SEC-004 |

---

## 6. Verification

| Test ID | Description | Expected Result |
|---------|-------------|-----------------|
| T-SEC-01 | Set `APCORE_AUTH_API_KEY=abc123`, access remote registry | HTTP request includes `Authorization: Bearer abc123`. |
| T-SEC-02 | No API key, remote registry configured | Exit 77. stderr: "requires authentication". |
| T-SEC-03 | Remote registry returns 401 | Exit 77. stderr: "Authentication failed". |
| T-SEC-04 | Store API key with keyring available | `apcore.yaml` contains `keyring:auth.api_key`. |
| T-SEC-05 | Store API key without keyring | `apcore.yaml` contains `enc:v2:base64...`. |
| T-SEC-06 | Retrieve keyring-stored value | Correct value returned from OS keyring. |
| T-SEC-07 | Retrieve AES-encrypted value | Correct value decrypted and returned. |
| T-SEC-08 | Corrupted ciphertext | Exit 47. stderr: "Failed to decrypt". |
| T-SEC-09 | Machine change (different hostname) | Exit 47. AES key derivation produces different key. |
| T-SEC-10 | Successful module execution | Audit log contains entry with `status: "success"`, `exit_code: 0`. |
| T-SEC-11 | Failed module execution | Audit log contains entry with `status: "error"`, correct exit code. |
| T-SEC-12 | Audit log not writable | Module executes successfully. WARNING on stderr. |
| T-SEC-13 | `os.getlogin()` fails | Audit entry has `user` from `USER` env var or `"unknown"`. |
| T-SEC-14 | `--sandbox` flag provided | Module runs in restricted subprocess. `HOME` env var is temp dir. |
| T-SEC-15 | Sandbox: module writes file | File written to temp dir, not user's working directory. |
| T-SEC-16 | No sandbox flag | Module runs in current process. Normal environment. |
| T-SEC-17 | Local-only registry | No API key required. No authentication headers. |
| T-SEC-18 | Inspect `apcore.yaml` for stored secret | Value is NOT plaintext. Prefixed with `keyring:` or `enc:`. |
| T-SEC-19 | **`apcli exec --sandbox` under a *relative* `extensions.root` (the default `./extensions`)** | Module executes successfully. **Discriminating case for invariant 6** — a test written with an absolute root passes against the defect. Must run `apcli exec`, not business-module dispatch: the two call sites are wired independently and one may be correct while the other is not. |
| T-SEC-20 | Business-module dispatch `--sandbox` under a relative `extensions.root` | Module executes successfully. Paired with T-SEC-19 so a fix to one path cannot mask the other. |
| T-SEC-21 | **Parent side.** Sandbox with the extensions root neither set on `Sandbox` nor present in the environment | Raises `sandbox: extensions root not propagated` **before** spawn; assert no process was started, not merely that the call failed. |
| T-SEC-21a | **Runner side, tested independently.** Invoke the internal sandbox runner directly with `APCORE_EXTENSIONS_ROOT` unset | Exits non-zero with `sandbox: extensions root not propagated`; asserts it never resolves `./extensions`. **T-SEC-21 alone cannot cover this** — it passes with the runner's relative default fully intact, since the parent refuses first. The two obligations of invariant 6 need two tests. |
| T-SEC-22 | Inspect the injected child environment, with **both** `extensions.root` and `acl.root` configured relative | `APCORE_EXTENSIONS_ROOT` **and** `APCORE_ACL_ROOT` are present and **absolute**. Asserting only the extensions root passes against a step-3b that forgets `SANDBOX_PATH_TYPED_VARS`, which is how the `APCORE_` pass-through defeats invariant 6. |
