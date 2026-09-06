# Changelog

All notable changes to the apcore-cli specification will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [0.12.0] - 2026-09-06

Two new features — **FE-14 ACL Governance** and **FE-15a OpenAPI Import** — plus the aligned **apcore 0.30.0** / **apcore-toolkit 0.11.1** runtime upgrade. All three SDKs ship as 0.12.0: Python **1004 passed / 5 xfailed** (was 815), TypeScript **807 / 807** (was 661), Rust **996 passed / 0 failed / 3 ignored** (was 511).

**Why a minor.** Both features add normative specification content and new command surface (`apcli acl`, `apcli openapi`), and FE-14 makes a previously-inert enforcement path live: a project that already ships `acl/global_acl.yaml` sees its rules take effect for the first time. The security fixes below also change what a working consumer observes.

### Added

- **FE-14 ACL Governance** (`docs/features/acl-governance.md`, SRS §5.10 `FR-ACL-001`…`004`, Tech Design §8.15). The CLI resolves an ACL root through the FE-07 4-tier chain and attaches it via `Executor.set_acl()`; a missing root attaches nothing, preserving apcore's invariant that enforcement stays off unless configured. Adds the `apcli acl` group — `list`, `check`, `validate`, `status` — plus `--identity-id` / `--identity-type` / `--role`, `ACL_RULE_ERROR -> 47`, audit wiring for `acl.audit.*`, and a strategy-bypass warning. Before this, the CLI carried exit code 77 and an `acl` preflight row that no configuration could ever reach.
- **FE-15a OpenAPI Import** (`docs/features/openapi-import.md`, SRS §5.11 `FR-OAPI-001`/`002`, Tech Design §8.16). `apcli openapi scan` and `generate` turn an OpenAPI 3.x document into `ScannedModule` form and `.binding.yaml` artifacts through the apcore-toolkit `OpenAPIScanner`. Closes the half of deferred issue #15 that does not require a registry.

### Changed

- **Version Compatibility refreshed to apcore 0.30.0 + apcore-toolkit 0.11.1.** Python `apcore>=0.30.0` / `apcore-toolkit[http-proxy]>=0.11.1`; TypeScript peer `apcore-js>=0.30.0` / `apcore-toolkit>=0.11.1`; Rust `apcore = ">=0.30"` / `apcore-toolkit = { version = ">=0.11.1", features = ["http-proxy"] }`. FE-15a's `http(s)://` spec sources need the `http-proxy` extra (Python) or feature (Rust); local files and `generate` do not. Each SDK verified the bump source-neutral by running its suite against the new floors before implementing anything else.
- **README "Built-in Commands" corrected to the `apcli` group.** It had listed built-ins as bare root commands since before the v0.8.0 removal of the root-level deprecation shims, so it documented an invocation form that has not worked for four minor versions.
- **`security.md` §4.4 sandbox invariant 1 rewritten.** It required an environment "allowlist-built, NOT inherited+filtered" over a fixed key list; all three SDKs implement, and had long implemented, exactly the `APCORE_`-prefix inherit-and-filter it forbade — necessarily, since `APCORE_EXTENSIONS_ROOT` must reach the child. The spec was stale, not the implementations.

### Fixed

- **`--sandbox` bypassed the ACL entirely, in all three SDKs** (`acl-governance.md` §4.10). Each sandbox runner builds its own `Registry` + `Executor` and never receives the attached ACL, so switching on a **security** flag switched off access control. The gate now runs in the parent before the spawn, refusing with exit 77. Rust additionally had, and fixed, the same bypass on its `FsDiscoverer` script-module path.
- **The gate itself could be written to fail silently.** A gate passing no `Context` leaves *conditional* deny rules inert (§6.5 makes every conditional rule a non-match without one) while they fire in-process; a gate omitting the argument projection makes `arguments` conditions UNEVALUABLE, which §6.1.1 resolves toward **denial** — so it wrongly denies calls such a rule was written to permit, as well as failing to grant. Both are now normative, with discriminating tests.
- **Test suites in all three SDKs wrote to the developer's real `~/.apcore-cli/audit.jsonl`** — measured at 1272 bytes per TypeScript run, 639 per Rust run, and 1006 accumulated rows in Python. Pre-existing; surfaced by FE-14's audit records. Fixed test-side only, verified by byte count and md5 across full runs, and positively confirmed by checking the redirected files captured the same volume rather than the writes having stopped for an unrelated reason.
- **`ACL_RULE_ERROR` reached no exit-code map**, so a malformed ACL file exited the generic `1`, indistinguishable from "the module ran and failed". Now `47`; `77` stays reserved for an actual access decision.
- **SRS `FR-SEC-004` AC-1 contradicted the feature spec** — it required `HOME` to be *absent* in the sandbox while `security.md` §4.4 assigns it to the tempdir. Corrected to the implemented behaviour.

### Notes

- **A retracted claim, recorded rather than deleted.** An earlier draft of §4.8 asserted the audit wiring was blocked on a public `ACL.set_audit_logger` that Python and TypeScript would have to gain, and set FE-14's floor to apcore 0.30.0 for that reason. It was wrong — all three SDKs accept the callback as a constructor argument — and the claim reached implementers before it was checked against the SDK sources. The wiring needs no upstream change; the only cost is `reload()`, which no apcore-cli SDK uses.
- **Six of the nine specification defects this release fixed were found by implementing it**, and most were of one kind: a decision the spec did not make, which three implementations then made differently — the identity sentinel (`@cli` vs `cli`), four root flag help strings (which the conformance fixtures byte-match), `acl check`'s subcommand-level wording, the audit record's key order, and the environment boolean spelling table. Two were worse than divergence: the identity-merge precedence rule had a natural implementation that silently dropped unrestated fields, and `--writer native` could not have worked for any input the command can produce.
- **FE-15b remains deferred** on two prerequisites, neither about OpenAPI: `--binding` is a registration path in Python only (TypeScript populates a display overlay and has no standalone registry; Rust logs a line), and apcore-toolkit cannot yet encode a query parameter declared on a body method. FE-15a reports the affected operations as hazards so they are visible before that lands.

---

## [0.11.0] - 2026-09-02

Tracks the aligned **apcore 0.28.0** and **apcore-toolkit 0.10.2** runtime upgrade across all three SDKs, which ship as **0.11.0**.

Unlike 0.10.1 through 0.10.3 — each of which was a tracking release carrying no specification change and no SDK source change — this one is neither. Reading the full 0.28.0 delta and exercising it against the running runtime surfaced **three** defects: two in `apcore-cli-python` alone, and one shared by all three SDKs. All are pre-existing rather than upgrade regressions — 0.28.0 is what made them reachable or visible.

**Why a minor.** Two reasons, either sufficient. The specification gained normative content: §3.2.2 now states the four-tier health status set with a **MUST NOT** on dropping a tier, corrects what `--all` actually filters, and adds conformance case **T-SYS-02a**. And two SDK fixes change what a working consumer observes — a script branching on exit code `1` from an `apcli` system command now sees `45`, and a caller doing `handler.request_approval(...)["status"]` now gets a `TypeError`. Neither was correct behaviour, but apcore's own 0.28.0 release note sets the house rule for exactly this case: a change a consumer can notice "must ship as a **minor** (or major) version bump, never a patch". This also re-aligns the specification and SDK version lines, which had drifted to 0.10.3 against 0.10.5.

### Changed

- **`README.md` Version Compatibility snapshot refreshed to apcore 0.28.0 + apcore-toolkit 0.10.2** (2026-09-02). All three SDKs raise their dependency floors and ship as 0.11.0:
  - `apcore-cli-python`: `apcore>=0.28.0`, `apcore-toolkit>=0.10.2`
  - `apcore-cli-typescript`: peer `apcore-js>=0.28.0`, peer `apcore-toolkit>=0.10.2`
  - `apcore-cli-rust`: `apcore = ">=0.28"`, `apcore-toolkit = ">=0.10.2"`

### Notes

- **The one 0.28.0 change that reaches the CLI layer is the approval gate's new composition (§6.9).** Since spec v1.28.0 the gate fires on the **union** of the module annotation, an ACL rule carrying `approval: required` (§6.1.6-§6.1.8), and `gate_destructive` — so a call to a module annotated `requires_approval: false` can now be routed to the CLI's `CliApprovalHandler`, which previously only saw annotation-gated modules. `Executor.validate()` reports the same union (§7.9.5), which the CLI forwards verbatim as `requires_approval` from `apcli validate` and `--dry-run`; that is a correctness gain the CLI gets for free.

  Verified end-to-end in all three languages against the 0.28.0 runtime, and each SDK now carries the probe as a permanent regression test (`tests/test_approval.py::TestApprovalGateEndToEnd`, `tests/acl-argument-scoped-approval.test.ts`, `tests/acl_argument_scoped_approval.rs`). The shape: a `git.push` module annotated `requires_approval: false` behind an `arguments: { has_key: ["force"] }` approval rule must run ungated for `git push` and reach the handler for `git push --force`, with preflight reporting `false` and `true` respectively.

  Each suite includes a **discriminating** case, because the happy path alone cannot detect the failure it is written for: with the handler set to refuse and no TTY, the ungated call must still succeed while the gated one must be denied. A gate that never fired would pass every other assertion. The Python cases were additionally confirmed to fail against the pre-fix handler.

- **`apcore-cli-python` failed that verification on two counts (both fixed, both with regression tests).**
  1. `CliApprovalHandler` returned a **mapping** where the protocol requires an `ApprovalResult`. apcore's gate reads the answer by attribute, so every gate-routed approval — annotation-sourced included, and therefore **pre-existing** — raised `AttributeError` inside the gate and surfaced as `MODULE_EXECUTE_ERROR`. Nothing in that SDK exercised `request_approval`, which is why it survived. apcore-cli-rust converts through `cli_to_apcore_result` and apcore-cli-typescript's `ApprovalResult` is a structurally-typed interface, so this was a Python-only divergence. 0.28.0 widened its reach from annotation-gated modules to any module an ACL rule gates.
  2. `exit_code_for_error` matched only on the CLI's own exception classes, so an apcore-raised error from an `apcli` system command exited **1** instead of its canonical code, contradicting the taxonomy `system_cmd._exit_on_system_error` documents. TS `exitCodeForError` and Rust `map_module_error_to_exit_code` both read the apcore wire code; Python had the map and never consulted it from that path. 0.28.0 makes it reachable on a routine command: `system.usage.*` now constrains `period` to `^[1-9][0-9]*[hd]$`, and 0.28.0 also stops Python's dict-declared schemas being validation pass-throughs, so `apcli usage --period 0h` now raises `SCHEMA_VALIDATION_ERROR` where it previously returned an empty window with exit 0.

- **A second defect, in all three SDKs at once: `apcli health` reported "no data" for a project whose modules it had just listed.** apcore classifies module health in **four** tiers — `healthy` / `degraded` / `error` / `unknown` — and every CLI's summary tally iterated only the first three. `unknown` means "no calls recorded yet", the state every module in a fresh project is in, so the most common case printed a populated table above a total that denied it.

  Pre-existing in all three, and surfaced by this release rather than caused by it: `sys-health-summary.schema.json` had declared the enum as `["healthy", "degraded", "unhealthy"]` — a value **no SDK emits** — and apcore 0.28.0 corrects it to the four tiers actually produced, splitting the summary's `unhealthy` count into `error` and `unknown`. Once the canonical contract names four tiers, rendering three is a plain omission. Fixed and regression-tested in all three SDKs, each confirmed to go red against the pre-fix formatter.

- **Specification corrected alongside the SDK fix (`docs/features/usability-enhancements.md` §3.2.2).** The spec was the origin of the three-tier reading: its worked example showed `Summary: 1 healthy, 1 degraded, 1 error` with no fourth tier, and its flag table described `--all` as "Include healthy modules (default: only degraded/error)". Both are now corrected — the example carries an `unknown` row and a four-tier summary, a new **Status tiers** paragraph states the four-tier set and names apcore as its owner (`schemas/sys-health-summary.schema.json`, §6.6), and the `--all` row records what the filter actually does.

  The flag description mattered more than it looks: `include_healthy: false` filters **only** the `healthy` tier, so the **default** `apcli health` — no `--all` — already lists `unknown` modules. The defect was therefore not confined to a flag most users never pass; it was the first thing a new project saw. Verified against apcore-python's `HealthSummaryModule`, whose counts dict has carried all four keys the whole time. `T-SYS-02` is corrected and `T-SYS-02a` added to pin the all-`unknown` case.

- **A fourth defect, found by cross-SDK diff rather than by the 0.28.0 delta: the exit-code maps disagreed.** `DEPENDENCY_NOT_FOUND` and `DEPENDENCY_VERSION_MISMATCH` — both real apcore `ErrorCode` variants — map to 44 in apcore-cli-python and apcore-cli-typescript, and fell through to 1 in apcore-cli-rust. The same dependency failure therefore ended a script with a different code depending on which CLI ran it, and 1 reads as "the module ran and failed" rather than "it could not be resolved".

  Method worth recording: the three maps were extracted programmatically and compared key-for-key rather than read. That reported **2 divergent of 22 codes**; after the fix it reports 0. Spot-reading had already missed this once. All three SDKs now carry an assertion on both codes.

- **`descriptor.dependencies` starts returning real data, with no CLI change.** apcore 0.28.0 makes `dependencies` a **parsed** field on the module descriptor in all three SDKs (spec §12.2, required since v1.10.0 but only apcore-rust implemented it). The CLIs already read `descriptor.dependencies`, which was absent in apcore-python and apcore-typescript — so `apcli list --show-deps` and the JSON `dependency_count` reported `0` for every module that declared dependencies, and now report the true count. An output change, not a code change; verified against a two-module registry.

- **`apcore-cli-typescript` and `apcore-cli-rust` needed no *upgrade-driven* source changes**, verified by their full suites (653 and 791 tests) plus the end-to-end approval probe above. Three of 0.28.0's BREAKING Rust changes name types the CLI crate references — `ACLRule` gaining a field, `AuditEntry` becoming `#[non_exhaustive]`, `CallbackApprovalHandler::new` becoming async and fallible — and all three are source-compatible because the crate constructs none of them and implements `apcore::ApprovalHandler` directly.

- **What the delta does not touch.** None of the three CLIs constructs or loads an `ACL`, calls `check()` / `check_access()`, reads an `AuditEntry`, or builds an `ACLRule` — so §6.8.1's fail-closed legacy boolean, §6.1.5's `effect` value closure, the new `approval` field and `validate_rules()` are all inert at this layer. `ExecutionPolicy.resolve()`'s new call-site parameters are additive and no CLI configures a policy. `p99_latency_ms` changing value and `hourly_distribution[].hour` changing format are display-only or unread: `hourly_distribution` appears in none of the three SDKs.

- **apcore-toolkit 0.10.2 is a dependency-tracking release with no source change**, so the surfaces the CLI consumes (`format_*`, `DisplayResolver`, `BindingLoader`, `RegistryWriter`) are unchanged.

## [0.10.3] - 2026-08-17

Compatibility maintenance release. Tracks the aligned **apcore 0.27.0** and **apcore-toolkit 0.10.x** runtime upgrade across all three SDKs. No specification changes — none of the apcore 0.26.0 → 0.27.0 delta touches any surface the CLI consumes, verified by running each SDK's full test suite against the 0.27.0 runtime (Python 798 passed / 5 xfailed, TypeScript 653 passed, Rust full `make check` including the 511-case conformance suite — all unmodified).

### Changed

- **`README.md` Version Compatibility snapshot refreshed to apcore 0.27.0 + apcore-toolkit 0.10.x** (2026-08-17). All three SDKs raise their `apcore` dependency floors and ship as 0.10.5:
  - `apcore-cli-python`: `apcore>=0.27.0`, `apcore-toolkit>=0.10.0`
  - `apcore-cli-typescript`: peer `apcore-js>=0.27.0`, peer `apcore-toolkit>=0.10.0`
  - `apcore-cli-rust`: `apcore = ">=0.27"`, `apcore-toolkit = ">=0.10.0"`
- **Dependency-pin divergence (issue 6.8) — Rust's exact pin retired.** Rust's `apcore-toolkit` pin is now `>=0.10.0` (open upper bound, changed at 0.10.3), so all three SDKs share open-upper-bound apcore-toolkit pins. The `=`-pin blockage previously documented in this section no longer exists; the table and the reconciliation note are updated to reflect the resolved state.

### Notes

- **No SDK source changes required.** The apcore 0.26.0 → 0.27.0 delta is BREAKING at the spec level (middleware `before_step`/`after_step` semantics, ACL-failed `validate()` introspection withholding, `Registry.register` metadata `dependencies` persistence, A23 schema-conversion rules, `pipeline.configure` 4-field set, `requires`/`provides` non-configurability, no module-boundary type coercion, plus per-SDK removals — Python `namespace_keys`, TypeScript tracing root exports, Rust `ErrorCode::ConfigurationError` rename / OtelTracing removal) — but every one of these lands on a surface the CLI does not consume: the CLIs never construct or configure middleware/pipelines, never call `Registry.register` (registration is via `discover()` / toolkit `RegistryWriter`), run their own schema→CLI-flag converters on the descriptor's `input_schema`, perform their own flag-level coercion before `executor.call`/`execute`, and read only `valid`/`checks`/`requiresApproval` from `PreflightResult` (never `predicted_changes`). Full per-change evidence is recorded in each SDK's 0.10.5 CHANGELOG entry.

## [0.10.2] - 2026-06-24

Compatibility maintenance release. Tracks the aligned **apcore 0.25.0** and **apcore-toolkit 0.9.1** runtime upgrade across all three SDKs. No specification changes — neither the apcore 0.24.0 → 0.25.0 delta (config-driven ACL discovery via `acl.root`) nor the apcore-toolkit 0.8.1 → 0.9.1 bug-fix delta touches any surface the CLI consumes.

### Changed

- **`README.md` Version Compatibility snapshot refreshed to apcore 0.25.0 + apcore-toolkit 0.9.1** (2026-06-24). All three SDKs raise their dependency floors and ship as 0.10.2:
  - `apcore-cli-python`: `apcore>=0.25.0`, `apcore-toolkit>=0.9.1`
  - `apcore-cli-typescript`: peer `apcore-js>=0.25.0`, `apcore-toolkit>=0.9.1`
  - `apcore-cli-rust`: `apcore = "0.25"`, `apcore-toolkit = "=0.9.1"`
- **Dependency-pin divergence (issue 6.8) — pins refreshed, structural divergence unchanged.** Python/TypeScript keep open-upper-bound pins (now `>=0.9.1`); Rust keeps an exact pin (now `=0.9.1`). Both are current as of this release; reconciliation to consistent caret semantics is still planned for a follow-up coordinated release.

### Notes

- **No SDK source changes required.** apcore 0.25.0 adds config-driven ACL discovery (`acl.root` activation + `ACL.discover`), auto-wired only by the `APCore` bootstrap and skipped when the caller supplies its own `Executor`. All three CLIs construct an `Executor` directly and never construct `APCore`, so ACL discovery does not engage; the change is backward-compatible regardless (a missing `acl.root` attaches no ACL, preserving the no-enforcement default), and Rust's `acl.root` now defaults to `./acl` instead of being hard-required — relaxing validation only. apcore-toolkit 0.9.1 is a cross-language bug-fix release (Python OpenAPI integer/`null` key handling, TypeScript `RegistryVerifier`/`RegistryWriter` fixes, Rust `RegistryWriter::write` relaxed to `&Registry`) whose stable surface consumed by the CLI (`format_*`, `DisplayResolver`, `BindingLoader`, `RegistryWriter`) is unchanged.

## [0.10.1] - 2026-06-15

Compatibility maintenance release. Tracks the aligned **apcore 0.24.0** and **apcore-toolkit 0.8.1** runtime upgrade across all three SDKs. No specification changes — the apcore 0.22.0 → 0.24.0 delta touches no surface the CLI consumes, verified against the full Python / TypeScript / Rust test suites (789 / 653 / 791 passing).

### Changed

- **`README.md` Version Compatibility snapshot refreshed to apcore 0.24.0 + apcore-toolkit 0.8.1** (2026-06-15). All three SDKs raise their dependency floors and ship as 0.10.1:
  - `apcore-cli-python`: `apcore>=0.24.0`, `apcore-toolkit>=0.8.1`
  - `apcore-cli-typescript`: peer `apcore-js>=0.24.0`, `apcore-toolkit>=0.8.1`
  - `apcore-cli-rust`: `apcore = "0.24"`, `apcore-toolkit = "=0.8.1"`
- **Dependency-pin divergence (issue 6.8) — pins refreshed, structural divergence unchanged.** Python/TypeScript keep open-upper-bound pins (now `>=0.8.1`); Rust keeps an exact pin (now `=0.8.1`). Both are current as of this release; reconciliation to consistent caret semantics is still planned for a follow-up coordinated release.

### Notes

- **No SDK source changes required.** The apcore 0.22→0.24 delta is internal to apcore or unused by the CLI layer: per-instance `ToggleState` isolation (#71), default AI error-recovery metadata (#70), and the **A-D-019** error-`details` `snake_case` alignment — which renames only the *inner* keys of the three call-chain-guard errors (`maxDepth`→`max_depth`, …) that the CLI forwards verbatim and never reads. Stable surfaces the CLI depends on (`Registry.list` / `get_definition`, `Executor.call`, the approval handler, toolkit `format_*`) are unchanged across the delta. Mirrors apcore-toolkit 0.8.1's own "runtime bump, zero API change" outcome.

## [0.10.0] - 2026-05-29

Maintenance release. Two spec-accuracy fixes (D9-004 shell-integration retirement, output-formatter architectural boundary clarification) and an ecosystem version compatibility section in the README.

### Added

- **`docs/features/output-formatter.md` §4.1 — Format Responsibility Boundaries** — new architectural guidelines section that formalises which layer owns each output format and provides a decision rule for adding new formats. Adds a reference table mapping each format (`json`, `table`, `yaml`, `csv`, `jsonl`, `markdown`, `skill`) to its implementation layer (apcore-cli self-impl vs. apcore-toolkit), the owning package, and the rationale. Decision rule: if two consumers written in different languages must produce identical bytes, the format belongs in `apcore-toolkit` with a conformance fixture; otherwise it is CLI-local. Resolves spec ambiguity raised in §6.6.
- **`README.md` — Version Compatibility section** — documents the currently tested ecosystem combination (2026-05-18 snapshot): `apcore` core SDK `>=0.21.0` (tested 0.22.0) and `apcore-toolkit` 0.7.0 (required, no soft fallback). Includes a "Known dependency-pin divergence" table (tracked as issue 6.8): Python and TypeScript use open-upper-bound pins (`>=0.7.0`) while Rust uses an exact pin (`=0.7.0`), blocking future toolkit minor bumps until manually updated. Reconciliation to consistent caret semantics is planned for a follow-up coordinated release.

### Fixed

- **`docs/features/shell-integration.md` — retire stale `register_shell_commands` spec** (D9-004) — the spec had been presenting `register_shell_commands(cli, prog_name)` as a live current API in both the Contract block (§4) and the function spec (§4.6), but the flat wrapper was **removed in v0.7.0 / FE-13** from Rust and TypeScript and is now test-only in Python. Replaced the Contract block with `register_completion_command` (the actual per-command registrar that attaches `completion` under the `apcli` builtin group) and converted §4.6 into a removal/migration note directing SDK implementors to `register_completion_command` for the `completion` subcommand and `configure_man_help` for the `man` overlay. Aligns with the existing retirement notes in `tech-design.md:1119` and `builtin-group.md:72`.

## [0.9.0] - 2026-05-13

Aligned spec release. Promotes csv/jsonl from SDK-native to **toolkit-delegated byte-equivalent** tier alongside markdown/skill. Requires apcore-toolkit `>=0.7.0` (was optional peer). Three SDKs (Python / TypeScript / Rust) refactored to delegate; all spec-driven conformance tests pass byte-identical across languages.

### Added

- **New conformance fixture: Algorithm C-SNAKE** (`conformance/fixtures/snake-case-kwargs/`). Verifies that schema property names containing underscores (`has_solution`, `sort_by`, `sort_order`) survive the round trip from CLI parsing to the input dict that `Executor.execute(module_id, input)` / `Executor.call(module_id, input)` receives. Includes a synthetic `test.snake_case_kwargs` module schema and five test cases (positive flag, negation, default fallback, snake_case string flag, multi-flag combination). Cross-SDK runners landed in apcore-cli-typescript / apcore-cli-python / apcore-cli-rust (Unreleased entries in each). Surfaces a class of bugs that single-word-only test fixtures (`math.add` with `a` / `b`) cannot detect: TypeScript commander auto-camelCases long flags and the TS SDK previously did not reverse-map them; Python click and Rust clap natively keep the snake_case attribute name.
- **ADR-09: Output Format Tiers — Toolkit-Delegated vs SDK-Native Presentation** (`docs/tech-design.md`) — formalises the three tiers. Tier 1 (byte-equivalent toolkit-delegated): csv, jsonl, markdown, skill. Tier 2 (SDK-native presentation): table, future tui. Tier 3 (trivial stdlib): json. Includes the bug table that drove the decision (apcore-cli-python Python repr; apcore-cli-typescript heterogeneous-keys data loss; apcore-cli-rust `\n` vs CRLF) and the downstream-consumer argument (aisee-cli reimplemented its own broken CSV).
- **FR-DISC-004 AC-7** (`docs/srs.md`) — `--format csv` / `--format jsonl` MUST produce byte-identical output across SDKs; nested values MUST be canonical compact JSON; CSV header MUST be union of keys across all rows; CSV line terminator MUST be CRLF.
- **T-OUT-12a / T-OUT-12b** (`docs/features/output-formatter.md`) — explicit conformance test cases for heterogeneous-keys regression and nested-object regression.

### Fixed (2026-05-13 — cross-SDK audit)

- **`security.md` `AuditLogger.log_execution` stale Rust language note** (D10-W2) — earlier entry claimed Rust collapsed `status + exit_code` into a single `Result<Value, ModuleExecutionError>` parameter (4-param signature). Actual source takes the same 5-param form as Python/TS. Replaced with a positive parity statement.
- **`security.md` `AuditLogger.log_execution` stale return-type claim** (D10 re-audit) — entry claimed Rust returns `Result<(), AuditLogError>`; actual source returns `()`. I/O failures are swallowed with `tracing::warn!` matching Python/TS. Corrected to `-> ()` infallible across all three SDKs.
- **`security.md` `Sandbox.execute` stale 2-param Rust language note** (D10 re-audit) — note claimed Rust binds the executor at construction time and takes only `(module_id, input)`. Actual Rust source takes the same 3-param form as Python/TS `(module_id, input_data, executor)`. Stale divergence note removed; replaced with parity confirmation plus note on construction-time `withExtensionsRoot` binding.
- **`security.md` `ConfigEncryptor` constructor fallibility note added** (D10-W4) — §4.2 now documents that Rust `ConfigEncryptor::new() -> Result<Self, ConfigDecryptionError>` validates keyring/AES reachability at construction while Python/TS defer validation to first `store()`/`retrieve()` call.
- **`output-formatter.md` Rust-idiom note for `format_module_*`** (D10-W3) — Rust `format_module_list` / `format_module_detail` / `format_exec_result` return `String` (callers `println!` it) while Python/TS write to stdout and return `None`/`void`. Spec note added to both functions confirming byte-on-stdout semantics are preserved.
- **`core-dispatcher.md` §8.1 per-subcommand registrar Contract blocks deferral** (D4-W1) — six public registrars (`register_list_command`, `register_describe_command`, `register_exec_command`, `register_validate_command`, `register_completion_command`, `register_pipeline_command`) now have an explicit §8.1.1 deferral note with a shared behavioral envelope (Inputs/Errors/Returns/Properties), mirroring the existing D4-013 deferral for `register_system_commands`.

### Changed

- **`docs/features/output-formatter.md` §4 dispatcher narrative** — csv/jsonl reclassified from "render rows via stdlib CSV writer / one JSON dump per line" to "MUST delegate to `apcore_toolkit.format_csv(rows)` / `format_jsonl(rows)`". yaml remains Tier 2 (SDK-native, may differ); markdown/skill remain toolkit-delegated as before.
- **`docs/srs.md` FR-DISC-004 main flow step 4** — wording promoted from "render via the standard library's CSV writer / `yaml.safe_dump` / one `json.dumps(row)` per line" to "the SDK MUST delegate to `apcore_toolkit.format_csv(rows)` / `format_jsonl(rows)`; per-SDK reimplementation is prohibited".

### Breaking changes

- **Global `--verbose` flag renamed to `--all-options`** — The help-display flag is now `--all-options`; use `<cli> --help --all-options` to reveal hidden built-in options (`--input`, `--yes`, `--large-input`, `--format`). `verbose` is removed from the reserved schema property names set — module schemas may now freely define `verbose: boolean` for runtime output control without collision. Tracked in [#21](https://github.com/aiperceivable/apcore-cli/issues/21).
- **apcore-toolkit promoted from optional to REQUIRED runtime dependency** for all 3 SDKs:
  - `apcore-cli-python`: was extras `[toolkit]`, now in `[project.dependencies]`
  - `apcore-cli-typescript`: peer-dep `optional: true` removed; minimum bumped to `>=0.7.0`
  - `apcore-cli-rust`: `optional = true` removed; `toolkit` Cargo feature retained as no-op in `default` for backward compat
- Downstream consumers using only `json` / `table` formats are unaffected at runtime but must install apcore-toolkit alongside apcore-cli.

### Why

Per-SDK reimplementations of csv/jsonl accumulated divergence — see ADR-09 for the full case study. The aisee-cli "CSV completely breaks in Excel" report surfaced the cumulative impact and triggered the architectural change.


## [0.8.0] - 2026-05-08

### Added

- **Built-in command group rename** (`builtin_group_name` kwarg / `builtinGroupName` option) — downstream branded CLIs that embed apcore-cli can now expose the built-ins under a custom namespace (e.g. `mycorp-cli admin health` instead of `mycorp-cli apcli health`). Default remains `"apcli"`. Validated against `/^[a-z][a-z0-9_-]*$/`. Env var `APCORE_CLI_APCLI` and config keys `apcli.*` deliberately do NOT rename — they are apcore-cli-internal toggles, not user-facing. New "Built-in group rename" section in `docs/features/builtin-group.md`; new `name` parameter on `Contract: ApcliGroup.__init__`. Cross-SDK parity: Python `create_cli(builtin_group_name=...)` and TypeScript `createCli({ builtinGroupName })`; Rust documented as parity gap pending embedding-API rebuild.
- **`## Algorithm:` blocks added to four high-complexity Contracts** (audit D4-012):
  - `resolve_refs` (`docs/features/schema-parser.md`) — captures the cross-SDK-critical merge invariants for `allOf` / `anyOf` / `oneOf` / `$ref`, depth-counter semantics, and visited-path tracking.
  - `Sandbox._sandboxed_execute` (`docs/features/security.md`) — env allowlist, tempdir cleanup-on-exception, output-size guard, SIGTERM/SIGKILL ladder.
  - `ConfigEncryptor.retrieve` (`docs/features/security.md`) — three-tier `keyring:` → `enc:v2:` → `enc:` fallback with order-of-prefix-check invariants and PBKDF2 iteration count.
  - `ApcliGroup.resolve_visibility` (`docs/features/builtin-group.md`) — strict 4-tier order (CliConfig > env > yaml > auto-detect) with case-insensitive trim-on-read env parsing.
- **`register_system_commands` per-subcommand authoritative-behavior section** added to `docs/features/usability-enhancements.md` (audit D4-013). Documents the four cross-SDK invariants the six system subcommands (`health`, `usage`, `enable`, `disable`, `reload`, `config`) must satisfy: client-side approval gating on mutating ops, canonical error→exit-code mapping, format-set restriction, atomic group registration. Full per-subcommand Contract split deferred to v0.10.
- **`docs/audit-report-2026-05-08.md`** — fresh release-gate snapshot reflecting all 33+ findings closed plus the D11 Tier B deep scan results (system_cmd / output / discovery). Supersedes the in-conversation `audit-report-2026-05-07.md` snapshot which was never persisted.

- **FR-DISC-004 — `markdown` and `skill` output formats** for `list` and
  `describe` (issue
  [#20](https://github.com/aiperceivable/apcore-cli/issues/20)). Both delegate
  to `apcore_toolkit.format_module(s)` (toolkit ≥0.6) so output is byte-identical
  across the Python / TypeScript / Rust SDKs. `--format skill` produces
  vendor-neutral SKILL.md content (`name` + `description` frontmatter) loadable
  as-is by Claude Code (`.claude/skills/<id>/SKILL.md`) and Gemini CLI
  (`.gemini/skills/<id>/SKILL.md`). `exec`, `apcli system *`, and
  `apcli strategy *` deliberately keep the five-format choice set
  `[table, json, csv, yaml, jsonl]` — markdown/skill target `ScannedModule`
  and do not apply to arbitrary business / health / strategy payloads.
  - SRS §5.4 FR-DISC-004 main flow, AF-2, and acceptance criteria rewritten
    around the seven-format canonical choice set.
  - SRS §5.4 FR-DISC-003 AF-4 added documenting the toolkit delegation.
  - Feature spec FE-08 §1, §3 (Contracts), §4.2, §4.3, §5, §7 updated;
    new test rows T-OUT-16 / T-OUT-17 / T-OUT-18.
  - Tech Design §3.2 Goal 4 expanded with the Claude Code / Gemini CLI
    skill-export workflow.

### Changed

- **`docs/features/schema-parser.md` Contract: `schema_to_click_options`** — explicitly documents that reserved-property-name violations exit with code 48 (was previously implicit). Both reserved-name and flag-name-collision share `SystemExit(48)` for cross-SDK parity (audit D11-NEW-005). New "Cross-language notes" section enumerates the per-SDK error-routing patterns (Python raises via `sys.exit`, Rust returns `Err(SchemaParserError::*)`, TypeScript exits via `process.exit(EXIT_CODES.SCHEMA_CIRCULAR_REF)`) — observable exit code is 48 across all three.
- **`docs/spec/...` resolve_refs `allOf` semantics** — the Algorithm block (see Added) codifies that `required` arrays are deduped first-seen-wins after merging in BOTH the `allOf` and `anyOf`/`oneOf` paths. This was previously implementation-defined; audit D9-NEW-002 found Python missing the dedup, Rust deduping parent-vs-branches but not branches-vs-branches, and TypeScript fully deduping. Spec now mandates full dedup everywhere.

- **`docs/tech-design.md` §8.2.7 — `create_cli()` canonical signature gains
  `version: str | None = None` and `description: str | None = None`** parameters
  (issues #18 and #19). Logic step 8 reworded so `click.version_option` is
  documented as opt-in (registered only when `version` is supplied), and the
  description defaults to `f"{prog_name} CLI"` rather than the legacy
  `apcore CLI — execute apcore modules from the command line` string.
- **`docs/features/discovery.md` §4.1 — `list --sort calls|errors|latency`**
  description updated to reflect the implemented behaviour: aggregates are
  read from the local audit log (`~/.apcore-cli/audit.jsonl`) over a default
  24h window; when no entries match, the SDK falls back to `id` sort and
  emits a user-visible note to stderr (issue #17). Replaces the prior
  "system modules unavailable; WARNING" placeholder text.
- **Conformance fixtures (`conformance/fixtures/apcli-visibility/*`)**
  refreshed to match the debranded help output and the new `version` /
  `description` opt-ins. All five fixtures (`standalone-default`,
  `embedded-default`, `cli-override`, `env-override`, `yaml-include`) now
  carry `"version": "0.8.0"` in `create_cli.json` so the cross-SDK
  `expected_help.txt` golden continues to verify `-V, --version` rendering
  in addition to apcli visibility.

### Fixed

- **SRS FR-DISC-004 AC-4 staleness** (issue
  [#20](https://github.com/aiperceivable/apcore-cli/issues/20)) — the previous
  AC asserting that `--format yaml` should be rejected was removed; csv/yaml/jsonl
  have been shipped since v0.6.0 and the new AC-6 covers invalid-format
  rejection generically.

### Note

- **Issues #15 (OpenAPI spec-driven discovery) and #16 (RFC 8628 device-auth
  flow) deferred** out of v0.8.0 scope. Both belong primarily in
  `apcore-toolkit` (with thin cli-side adapters), require their own RFCs,
  and span 2–3 release cycles each. Tracked for v0.9+.

---

## [0.7.0] - 2026-04-25

### Added

- **FE-12: Module Exposure Filtering** — Declarative control over which discovered modules are exposed as CLI commands. Feature spec: `docs/features/exposure-filtering.md`.
  - `expose` section in `apcore.yaml` with three modes: `all` (default), `include` (whitelist), `exclude` (blacklist).
  - Glob-pattern matching on module IDs (e.g., `admin.*`, `webhooks.*`, `*.sse`).
  - `ExposureFilter` class with `is_exposed()` and `filter_modules()` methods.
  - `create_cli(expose=...)` parameter accepting `dict` or `ExposureFilter` instance.
  - `list --exposure {exposed,hidden,all}` filter flag.
  - 4-tier config precedence: `CliConfig.expose` > `--expose-mode` CLI flag > env var > `apcore.yaml`.
  - Hidden modules remain invocable via `exec <module_id>` (UX filter, not a security boundary).
- **FE-13: Built-in Command Group (`apcli`)** — All apcore-cli-provided built-in commands (`list`, `describe`, `exec`, `init`, `validate`, `health`, `usage`, `enable`, `disable`, `reload`, `config`, `completion`, `describe-pipeline`) relocate under a single reserved group named `apcli`. Feature spec: `docs/features/builtin-group.md`. New SRS requirement: FR-DISP-009.
  - Root level retains only `help` + `--help`/`--version`/`--verbose`/`--man`/`--log-level` + user business modules/groups.
  - `apcli` config accepts boolean shorthand (`true`/`false`) or object form (`mode`/`include`/`exclude`/`disable_env`). Mode ∈ {`all`, `none`, `include`, `exclude`}.
  - Auto-detected default: embedded mode (registry injected) → `none` (hidden); standalone mode → `all` (visible).
  - 4-tier precedence: `CliConfig.apcli` > `APCORE_CLI_APCLI` env var (when not disabled) > `apcore.yaml` > auto-detect.
  - `disable_env: true` opt-out severs the env-var override (config-only field; not exposed as a separate env var).
  - "Hidden ≠ unreachable": `mode: none` keeps `<cli> apcli <sub>` reachable for debugging / CI, only hides the group from root `--help`. For true lockdown use `{mode: include, include: [], disable_env: true}`.
  - `exec` subcommand always registered when the `apcli` group exists, preserving FE-12's hidden-module-invocation guarantee.
  - Discovery flags (`--extensions-dir`, `--commands-dir`, `--binding`) register only in standalone mode; embedded-mode CLIs produce "unknown option" if they are passed.
  - `RESERVED_GROUP_NAMES = {"apcli"}` rejects business modules whose group name, auto-grouped prefix, or top-level command name is `apcli`.
  - Normative cross-language contracts for Python / TypeScript / Rust / Go (Go SDK deferred, contract forward-looking).
- **Apache 2.0 license** added (`LICENSE` file).
- SRS: added requirement sections for FE-10 (Init Command), FE-11 (Usability Enhancements), FE-12 (Exposure Filtering) as TODO backfill items (B-004); added FR-DISP-009 (Built-in Command Group) for FE-13.
- **FE-13 conformance fixtures** (`conformance/fixtures/apcli-visibility/`) — cross-language fixture set covering the 4-tier apcli visibility decision chain (`CliConfig` > `APCORE_CLI_APCLI` > `apcore.yaml` > auto-detect). Five scenarios: `standalone-default`, `embedded-default`, `cli-override`, `env-override`, `yaml-include`. Each ships `create_cli.json` (snake_case caller options), `env.json` (env overlay), optional `input.yaml` (materialized as `apcore.yaml`), and a byte-match `expected_help.txt` golden.
- **Canonical help format** (normative, see `conformance/fixtures/apcli-visibility/README.md`) — SDKs must configure their underlying help renderer (Commander.js, Click, clap) to produce clap v4 / GNU-style output: description first, `Usage: <prog> [OPTIONS] [COMMAND]`, `Commands:` before `Options:`, uppercase `<PLACEHOLDER>`, `[default: VALUE]`, `-h, --help` → "Print help", `-V, --version` → "Print version" (short flag required), `-h`/`-V` rendered last, long-only options aligned under short+long rows (4-space slot), and no line wrapping. Reference implementation: `apcore-cli-typescript/src/canonical-help.ts` (Commander `configureHelp({ formatHelp })` override).

### Changed

- **Tech Design v2.0** — Major revision:
  - `BUILTIN_COMMANDS` canonicalized as a 14-entry alphabetically sorted constant (later retired by FE-13, see Removed).
  - `create_cli()` consolidated signature (§8.2.7) is now the authoritative reference; feature specs reference it incrementally.
  - `create_cli()` gains `expose` parameter for programmatic exposure filtering.
  - `create_cli()` gains `apcli` parameter (FE-13) for built-in command group visibility control.
- Root-level built-in commands are deprecated in standalone mode: `apcore-cli list`, `apcore-cli describe`, `apcore-cli exec`, `apcore-cli init`, `apcore-cli health`, `apcore-cli usage`, etc. emit a deprecation WARNING and delegate to the new `apcli` subcommand. Shims removed in v0.8.0.
- Embedded integrations (`create_cli(registry=...)`) now default to `apcli: none` — built-in commands hidden from end-user `--help`. Integrators who want the previous behavior pass `apcli: true` explicitly.
- `register_discovery_commands()` / `register_system_commands()` / `register_shell_commands()` registrars split into per-subcommand functions (`register_list_command`, `register_health_command`, etc.) to enable per-subcommand include/exclude filtering.
- SRS references updated from "Tech Design v1.0" to "Tech Design v2.0" throughout.
- SRS AC-1 for FR-DISP-001 updated: help output must list all 14 built-in commands from `BUILTIN_COMMANDS`.
- Conformance fixture `cli_parity.json` updated: `run` → `exec`, `--json` → `--format json`, `--inputs` → `--input`, exit code `127` → `44`, removed `is_perceivable` key.
- FE-11 version label corrected from "v0.7.0" to "v0.6.0" in feature overview.

### Removed

- **`BUILTIN_COMMANDS` constant retired** (FE-13). The 14-entry list in `apcore_cli/cli.py` and equivalents in other-language SDKs is replaced by `RESERVED_GROUP_NAMES = {"apcli"}`. External code that imported the constant will break at import time; impact assessed as low (internal constant, not public API per semver).
- Per-command collision check in `GroupedModuleGroup._build_group_map()` (was: warn + drop module on collision). Collision surface reduced to the single reserved group name `apcli` — checks are centralized in `RESERVED_GROUP_NAMES` enforcement.

### Fixed

- Align `security.md` with hardened sandbox env whitelist (PYTHONPATH explicitly excluded) (D10-001). Spec previously stated "Python SDK additionally allows `PYTHONPATH`"; the hardened reference impl in `apcore-cli-python/src/apcore_cli/security/sandbox.py` defines `_SANDBOX_ALLOW_KEYS = ("PATH", "LANG", "LC_ALL")` and explicitly omits `PYTHONPATH` to prevent module imports from crossing the sandbox boundary. Spec now matches impl and adds a normative note that `PYTHONPATH` (and any `*PATH` variable that influences module / package / shared-library resolution — `NODE_PATH`, `RUBYLIB`, `GOPATH`, `LD_LIBRARY_PATH`, etc.) MUST NOT cross the sandbox boundary regardless of language.

### Migration

- Pre-v0.7 users invoking root-level built-ins see a deprecation warning pointing to the new `apcli`-prefixed path. Full migration table in `docs/features/builtin-group.md` §11. Shims removed in v0.8.0.

---

## [0.6.0] - 2026-04-06

### Changed

- **Dependency bump**: requires `apcore >= 0.17.1` (was `>= 0.15.1`). Incorporates three apcore releases:
  - **apcore 0.16.0**: Execution Pipeline Strategy (`ExecutionStrategy`, `PipelineEngine`, `PipelineTrace`, 11 built-in steps, preset strategies), `Executor.strategy` parameter, `call_with_trace()` / `call_async_with_trace()`, Config Bus enhancements (`env_style`, `max_depth`, `env_prefix` auto-derivation, `env_map`), `ContextKey<T>` typed accessors, `ModuleAnnotations.extra` field, ACL condition handlers (`register_condition()`, `$or`/`$not`, `async_check()`).
  - **apcore 0.17.0**: Pipeline v2 declarative step metadata (`match_modules`, `ignore_errors`, `pure`, `timeout_ms`), `PipelineContext.dry_run` / `version_hint` / `executed_middlewares`, `StepTrace.skip_reason`, `safety_check` → `call_chain_guard` rename, pipeline step reorder (middleware_before now executes before input_validation), YAML pipeline configuration.
  - **apcore 0.17.1**: `minimal` execution strategy preset (4-step pipeline), `requires`/`provides` step dependency metadata.
- Updated SRS, Tech Design, Project Manifest, and feature spec to reflect `apcore >= 0.17.1` dependency.
- Updated ADR-03 (Executor Integration) to document optional `strategy` parameter and `call_with_trace()` availability.
- Updated SRS Execution Constraint to reference the standard 11-step pipeline and custom `ExecutionStrategy` support via pre-configured Executor.

### Added

- **FE-11: Usability Enhancements** — 11 new capabilities implemented across Python, TypeScript, and Rust SDKs:
  - **`--dry-run` preflight mode** (§3.1) — Validates module call without executing via `Executor.validate()`. Standalone `validate` command also added.
  - **System management commands** (§3.2) — `health`, `usage`, `enable`, `disable`, `reload`, `config get`/`config set`. Delegates to `system.*` modules; graceful no-op when unavailable.
  - **Enhanced error output** (§3.3) — Structured JSON errors with `ai_guidance`, `suggestion`, `retryable`, `user_fixable`, `details`. TTY mode shows suggestion and retryable; JSON mode includes all fields.
  - **`--trace` pipeline visualization** (§3.4) — Displays per-step execution trace via `call_with_trace()`.
  - **ApprovalHandler integration** (§3.5) — `--approval-timeout` (configurable), `--approval-token` (async resume). `CliApprovalHandler` class wraps TTY prompt as standard `ApprovalHandler` protocol.
  - **`--stream` output** (§3.6) — JSONL streaming via `Executor.stream()` for modules with `annotations.streaming=true`.
  - **Enhanced `list` command** (§3.7) — `--search`, `--status`, `--annotation`, `--sort`, `--reverse`, `--deprecated`, `--deps` filters.
  - **`--strategy` selection** (§3.8) — Choose execution pipeline: `standard`, `internal`, `testing`, `performance`, `minimal`. New `describe-pipeline` command.
  - **Output format extensions** (§3.9) — `--format csv|yaml|jsonl` and `--fields` for dot-path field selection.
  - **Multi-level grouping** (§3.10) — `cli.group_depth` config (default: 1, max: 3).
  - **Custom command extension point** (§3.11) — `create_cli(extra_commands=[...])` for downstream projects.
- **New error code**: Handle `CONFIG_ENV_MAP_CONFLICT` from apcore 0.16.0.
- New config keys: `cli.approval_timeout` (60), `cli.strategy` ("standard"), `cli.group_depth` (1).
- New environment variables: `APCORE_CLI_APPROVAL_TIMEOUT`, `APCORE_CLI_STRATEGY`, `APCORE_CLI_GROUP_DEPTH`.

### Fixed

- **Schema parser**: Required schema properties now correctly enforced at CLI level (was silently optional).
- **Approval gate**: Fixed inverted logic in annotation type guard that could crash on malformed annotations.

---

## [0.5.1] - 2026-04-03

### Added

- **FR-DISP-008: Pre-populated registry support** — `create_cli()` accepts optional `registry` and `executor` parameters. When a pre-populated Registry is provided, filesystem discovery is skipped entirely. This enables frameworks that register modules at runtime (e.g. apflow's bridge) to generate CLI commands from their existing registry without requiring an extensions directory on disk.
- Cross-language API table in tech-design §8.2.7 and core-dispatcher §4.5: Python `create_cli(registry=)`, TypeScript `createCli({ registry })` via `CreateCliOptions`, Rust `CliConfig { registry }`.
- Verification tests T-DISP-18 through T-DISP-20 for pre-populated registry path.
- Passing `executor` without `registry` is a validation error (ValueError/throw).

---

## [0.5.0] - 2026-03-31

### Changed

- **Dependency bump**: requires `apcore >= 0.15.1` (was `>= 0.13.0`). Adds Config Bus namespace registration, canonical event type naming, Error Formatter Registry support, and simplified env var prefix convention.
- Updated SRS, Tech Design, and Project Manifest to reflect `apcore >= 0.15.1` dependency.

### Added

- **Config Bus integration** — `apcore-cli` registers namespace `apcore-cli` with env prefix `APCORE_CLI` at import time via `Config.register_namespace()`. Supports unified `apcore.yaml` configuration in namespace mode alongside legacy flat-key mode.
- **Canonical event types** — Spec updated to reference dot-namespaced canonical event names (`apcore.module.toggled`, `apcore.config.updated`, `apcore.module.reloaded`, `apcore.health.recovered`) from apcore 0.15.0. SDKs should adopt these names when implementing event subscriptions.
- **New error codes** — Handle `CONFIG_NAMESPACE_RESERVED`, `CONFIG_NAMESPACE_DUPLICATE`, `CONFIG_ENV_PREFIX_CONFLICT`, `CONFIG_MOUNT_ERROR`, `CONFIG_BIND_ERROR`, `ERROR_FORMATTER_DUPLICATE` from apcore 0.15.0.

## [0.4.0] - 2026-03-29

### Added

- **FR-DISP-007: Verbose help mode** — Built-in apcore options (`--input`, `--yes`, `--large-input`, `--format`, `--sandbox`) are now hidden from `--help` output by default. Pass `--help --verbose` to display the full option list. Added `verbose` to reserved flag names to prevent schema property collisions.
- **FR-SHELL-002: Universal man page generation** — `build_program_man_page()` generates a complete roff man page covering all registered commands (including downstream business commands). `configure_man_help()` adds `--help --man` support to any CLI program. Downstream projects get man pages with a single function call.
- **Documentation URL support** — `set_docs_url()` / `setDocsUrl()` sets a base URL for online documentation links in help footers and man pages. Documented in tech-design §8.7.6 and shell-integration §4.10.

### Changed

- `--sandbox` is now always hidden from help (not yet implemented). FR-DISP-007 updated from "five" to "four" toggled options.
- Improved built-in option descriptions across all three SDKs for clarity.
- Updated `core-dispatcher.md` feature spec: added FR-01-07 traceability entry and built-in option visibility note.
- Updated `tech-design.md`: documented verbose help behavior, `build_program_man_page` (§8.7.4), and `configure_man_help` (§8.7.5).
- Updated `srs.md`: added FR-DISP-007 requirement; updated FR-SHELL-002 with full-program mode and acceptance criteria.
- Updated `shell-integration.md` feature spec: added FR-06-03, §4.8 (`build_program_man_page`), §4.9 (`configure_man_help`), and verification tests T-SHELL-09 through T-SHELL-13.

## [0.3.0] - 2026-03-23

### Added

- **Display overlay routing (§5.13)** — `LazyModuleGroup` now reads `metadata["display"]["cli"]` for alias and description when building the command list and routing `get_command()`.
  - `_alias_map`: built from `metadata["display"]["cli"]["alias"]` (with module_id fallback), enabling invocation by alias.
  - `_descriptor_cache`: populated during alias map build to avoid double `registry.get_definition()` calls.
  - `_alias_map_built` flag only set on successful build, allowing retry after transient registry errors.
- **Display overlay in JSON output** — `format_module_list(..., "json")` reads `metadata["display"]["cli"]` for `id`, `description`, and `metadata["display"]["tags"]`.

### Changed

- `_ERROR_CODE_MAP.get(error_code, 1)`: guarded with `isinstance(error_code, str)` to prevent `None`-key lookup.
- Dependency bump: requires `apcore-toolkit >= 0.4.0` for `DisplayResolver`.
- Updated feature specs: `core-dispatcher.md` (alias map, descriptor cache), `output-formatter.md` (JSON branch display overlay).

### Tests

- `TestDisplayOverlayAliasRouting` (6 tests): `list_commands` uses CLI alias, `get_command` by alias, cache hit path, module_id fallback, `build_module_command` alias and description.
- `test_format_list_json_uses_display_overlay`: JSON output uses display overlay alias/description/tags.
- `test_format_list_json_falls_back_to_scanner_when_no_overlay`: JSON output falls back to scanner values.

### Added (Grouped Commands — FE-09)

- **Feature spec `grouped-commands.md`** (FE-09) — nested subcommand groups for CLI. Auto-groups by first `.` segment, with `display.cli.group` override. Includes 10 requirements (FR-09-01 through FR-09-10), boundary values, error handling table, and 18 verification test cases.
- **Tech Design v2.0** — full rewrite incorporating both §5.13 Display Overlay and Grouped CLI Commands. 3 alternative solutions with weighted comparison matrix, 8 ADRs, 5 sequence diagrams.
- **Updated `core-dispatcher.md`** — references FE-09, documents `GroupedModuleGroup` as v2.0 root group.
- **Updated `overview.md`** — added FE-09 row to feature table.

### Added (Convention Module Discovery — §5.14)

- **`apcore-cli init module <id>`** — scaffolding command with `--style` (decorator, convention, binding) and `--description` options. Generates module templates in the appropriate directory.
- **`--commands-dir` CLI option** — path to a convention commands directory. When set, `ConventionScanner` from `apcore-toolkit` scans for plain functions and registers them as modules.

### Tests (Convention Module Discovery)

- 6 new tests in `tests/test_init_cmd.py` covering all three styles and options.

---

## [0.2.2] - 2026-03-22

### Changed
- Rebrand: aipartnerup → aiperceivable

## [0.2.1] - 2026-03-19

### Changed
- **Schema Parser (FE-02)**: Help text truncation limit increased from 200 to 1000 characters (configurable via `cli.help_text_max_length`)
- **Schema Parser (FE-02)**: `_extract_help` signature updated — added `max_length: int = 1000` parameter
- **Schema Parser (FE-02)**: `schema_to_click_options` signature updated — added `max_help_length: int = 1000` parameter
- **Core Dispatcher (FE-01)**: `build_module_command` signature updated — added `help_text_max_length: int = 1000` parameter
- **Config Resolver (FE-07)**: DEFAULTS dict expanded to 6 keys — added `cli.stdin_buffer_limit`, `cli.auto_approve`, `cli.help_text_max_length`
- **Config Resolver (FE-07)**: `logging.level` default corrected from `"INFO"` to `"WARNING"` across all spec documents (tech-design, SRS, config-resolver, core-dispatcher)

### Added
- `cli.help_text_max_length` config key (default: 1000) with `APCORE_CLI_HELP_TEXT_MAX_LENGTH` env var support
- `APCORE_CLI_LOGGING_LEVEL` added to tech-design environment variables table (was documented in feature spec but missing from tech-design)

### Fixed
- **Spec chain contradiction**: `logging.level` default was `"INFO"` in 5 locations but `"WARNING"` in core-dispatcher — now consistently `"WARNING"` everywhere
- **Spec chain contradiction**: `check_approval` pseudocode passed 3 arguments but signature defined 2 — removed stale `ctx` argument from pseudocode
- `cli.auto_approve` was listed in config keys table but missing from DEFAULTS dict — added to both tech-design and config-resolver
- core-dispatcher SRS header now includes FR-DISP-005 and FR-DISP-006

## [0.2.0] - 2026-03-16

### Changed
- **Core Dispatcher (FE-01)**: Added 3-tier log level precedence spec — `--log-level` flag > `APCORE_CLI_LOGGING_LEVEL` > `APCORE_LOGGING_LEVEL` > `WARNING`; renumbered `create_cli` logic steps accordingly
- **Core Dispatcher (FE-01)**: `register_shell_commands()` call now passes `prog_name=prog_name` (FR-DISP-006 alignment)
- **Core Dispatcher (FE-01)**: `--log-level` accepted choices updated: `WARN` → `WARNING`
- **Core Dispatcher (FE-01)**: Added FR-DISP-006 requirement — CLI program name resolved from `argv[0]` basename with explicit `prog_name` parameter override
- **Shell Integration (FE-06)**: Added `4.2 _make_function_name` helper spec with POSIX identifier conversion example
- **Shell Integration (FE-06)**: Updated all generator function signatures to include `prog_name: str` parameter; documented `shlex.quote()` usage in all shell directive positions (not just embedded subshell commands)
- **Shell Integration (FE-06)**: Added `4.6 register_shell_commands` spec with `prog_name` parameter and closure capture semantics
- **Shell Integration (FE-06)**: Man page `.SH ENVIRONMENT` section now specifies 4 env vars including `APCORE_CLI_LOGGING_LEVEL`; updated `WARN` → `WARNING`
- **Approval Gate (FE-03)**: `check_approval` signature corrected — removed `ctx: click.Context` parameter (was not used in implementation)
- **Approval Gate (FE-03)**: `annotations` guard updated to support both dict access and attribute access
- **Security Manager (FE-05)**: `_hash_input` formula updated to include `secrets.token_bytes(16)` per-invocation salt (prevents cross-invocation correlation)
- **Security Manager (FE-05)**: `_get_user` extended with `pwd.getpwuid()` as second fallback step before env var lookup

### Added
- `APCORE_CLI_LOGGING_LEVEL` environment variable to CLI Reference and Environment Variables table in README (CLI-specific log level; takes priority over `APCORE_LOGGING_LEVEL`)

### Fixed
- README: `--log-level` default corrected to `WARNING` (was `INFO`); accepted values updated from `WARN` → `WARNING`

## [0.1.0] - 2026-03-15

### Added
- Initial specification and design documents
- Tech Design v0.4 and v1.0
- Software Requirements Specification (SRS) v0.1
- 8 feature specifications: Core Dispatcher, Schema Parser, Approval Gate, Discovery, Security Manager, Shell Integration, Config Resolver, Output Formatter
- Project manifest with feature table and implementation order
- Idea draft with problem validation and requirement IDs
- Beginner guide (Getting Started) section in README
- GitHub repository links and SDK reference table
- Related Projects section linking to ecosystem repos
- Full CLI Reference section (global options, commands, execution options, exit codes)
- Configuration section with 4-tier precedence and environment variables
- Architecture diagram and apcore-to-CLI mapping table
- CHANGELOG.md

[Unreleased]: https://github.com/aiperceivable/apcore-cli/compare/v0.7.0...HEAD
[0.7.0]: https://github.com/aiperceivable/apcore-cli/compare/v0.6.0...v0.7.0
[0.6.0]: https://github.com/aiperceivable/apcore-cli/compare/v0.5.1...v0.6.0
[0.5.1]: https://github.com/aiperceivable/apcore-cli/compare/v0.5.0...v0.5.1
[0.5.0]: https://github.com/aiperceivable/apcore-cli/compare/v0.4.0...v0.5.0
[0.4.0]: https://github.com/aiperceivable/apcore-cli/compare/v0.3.0...v0.4.0
[0.3.0]: https://github.com/aiperceivable/apcore-cli/compare/v0.2.2...v0.3.0
[0.2.2]: https://github.com/aiperceivable/apcore-cli/compare/v0.2.1...v0.2.2
[0.2.1]: https://github.com/aiperceivable/apcore-cli/compare/v0.2.0...v0.2.1
[0.2.0]: https://github.com/aiperceivable/apcore-cli/compare/v0.1.0...v0.2.0
[0.1.0]: https://github.com/aiperceivable/apcore-cli/releases/tag/v0.1.0
