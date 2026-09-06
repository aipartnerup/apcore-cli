---
description: "ACL Governance (FE-14) attaches an apcore ACL to the CLI's Executor from acl.root and adds the apcli acl subcommand group (list, check, validate, status) so operators can author, inspect, and lint access-control rules from the terminal."
---

# Feature Spec: ACL Governance

**Feature ID**: FE-14
**Status**: Draft
**Priority**: P1
**Parent**: [Tech Design v2.0](../tech-design.md) Section 8.15

**SRS Requirements**: FR-ACL-001, FR-ACL-002, FR-ACL-003, FR-ACL-004, FR-SEC-001
**Related Features**: FE-01 (Core Dispatcher), FE-03 (Approval Gate), FE-07 (Config Resolver), FE-11 (Usability Enhancements)
**Requires**: apcore >= 0.30.0, apcore-toolkit >= 0.11.1

---

## 1. Description

apcore has enforced access control since PROTOCOL_SPEC §6, and apcore-cli has always carried the *downstream* half of it: exit code `77` for `ACL_DENIED`, an `acl` row in `--dry-run` preflight output, and `acl_check` in `apcli describe-pipeline`. None of it has ever been reachable, because **no apcore-cli SDK has ever constructed or attached an `ACL`**. All three CLIs build an `Executor` directly rather than going through `APCore`, which is the bootstrap that performs `ACL.discover()`. The result is an executor whose `acl_check` step consults nothing, and a `governance_state()` that reports `unprotected_control_surface: true` for every project — including projects that ship an `acl/global_acl.yaml` and reasonably assume it is in force.

FE-14 closes that loop. It attaches an ACL to the CLI's executor from the same `acl.root` key apcore already owns, and adds an `apcli acl` subcommand group so the rules are authorable and inspectable from the terminal rather than only from a host application.

The feature is deliberately **enforcement-only-when-configured**: a missing `acl.root` attaches nothing and changes no behaviour, preserving apcore's `ACL.discover` invariant that a missing path MUST NOT synthesize an empty default-deny ACL. Every existing project therefore behaves exactly as it does today.

### 1.1 Why this belongs in the CLI layer

The `apcli` built-in group registers `system.control.*` modules (`enable`, `disable`, `reload`, `config set`) — the highest-privilege surface the CLI exposes. PROTOCOL_SPEC §6.6.5.1 defines `unprotected_control_surface` precisely for this shape: control modules registered, with neither an ACL gate nor a fully-annotated approval gate in front of them. Because the CLI currently cannot attach an ACL, the only protection available is the approval annotation. FE-14 makes the ACL half reachable, and `apcli acl status` makes the resulting posture legible.

---

## 2. Requirements Traceability

| Req ID | SRS Ref | Description |
|--------|---------|-------------|
| FR-14-01 | FR-ACL-001 | The CLI resolves an ACL root through the FE-07 4-tier precedence chain and attaches the loaded ACL via `Executor.set_acl()`. |
| FR-14-02 | FR-ACL-001 | A missing or empty ACL root attaches nothing; enforcement stays off and no default-deny ACL is synthesized. |
| FR-14-03 | FR-ACL-002 | `apcli acl list` renders the attached rule set and its `default_effect`. |
| FR-14-04 | FR-ACL-003 | `apcli acl check <TARGET>` evaluates a simulated call through `ACL.check_access()` and reports both axes (access, approval). |
| FR-14-05 | FR-ACL-004 | `apcli acl validate` reports every `RuleValidationFinding` from `ACL.validate_rules()`. |
| FR-14-06 | FR-ACL-004 | `apcli acl status` renders `Executor.governance_state()`, including `unprotected_control_surface`. |
| FR-14-07 | FR-SEC-001 | An ACL decision is written to the FE-05 audit log when `acl.audit.enabled` is true. |
| FR-14-08 | FR-ACL-001 | `--identity-id`, `--identity-type`, and `--role` build a `Context` identity so conditional rules are evaluable from the CLI. |
| FR-14-09 | FR-ACL-001 | A structurally invalid ACL file exits `47` rather than the generic `1`. |

---

## 3. Module Paths

=== "Python"

    ```
    apcore_cli/acl_cmd.py        register_acl_command (new)
    apcore_cli/acl_loader.py     resolve_acl_root, load_cli_acl (new)
    apcore_cli/factory.py        create_cli() wiring, --acl flag
    apcore_cli/config.py         acl.* DEFAULTS
    apcore_cli/exit_codes.py     ACL_RULE_ERROR mapping
    apcore_cli/builtin_group.py  APCLI_SUBCOMMAND_NAMES
    ```

=== "TypeScript"

    ```
    src/acl-cmd.ts         registerAclCommand (new)
    src/acl-loader.ts      resolveAclRoot, loadCliAcl (new)
    src/main.ts            createCli() wiring, --acl flag
    src/config.ts          acl.* DEFAULTS
    src/errors.ts          ACL_RULE_ERROR mapping
    src/builtin-group.ts   APCLI_SUBCOMMAND_NAMES
    ```

=== "Rust"

    ```
    src/acl_cmd.rs         register_acl_command, dispatch_acl (new)
    src/acl_loader.rs      resolve_acl_root, load_cli_acl (new)
    src/main.rs            executor wiring, --acl flag, dispatch arm
    src/lib.rs             registrar table, EXIT_* consts
    src/config.rs          acl.* DEFAULTS
    src/builtin_group.rs   APCLI_SUBCOMMAND_NAMES
    ```

---

## 4. Implementation Details

### 4.1 ACL root resolution

The ACL root is resolved through the standard FE-07 4-tier chain. Note that `acl.root` is an **apcore-owned** config key (it appears in apcore's own `Config` defaults), so its environment variable follows the apcore convention `APCORE_ACL_ROOT` — exactly as `extensions.root` is overridden by `APCORE_EXTENSIONS_ROOT` and not by an `APCORE_CLI_*` name.

| Tier | Source | Notes |
|------|--------|-------|
| 1 | `create_cli(acl=…)` / `--acl PATH` | `--acl` registers in **standalone mode only**, alongside `--extensions-dir`, `--commands-dir`, `--binding` (FE-13 §4.1). |
| 2 | `APCORE_ACL_ROOT` | apcore-owned key, apcore-prefixed variable. |
| 3 | `acl.root` in `apcore.yaml` | The same key apcore's own `ACL.discover` reads, so one file serves both. |
| 4 | `./acl` | Matches `Config.get_default("acl.root")`. |

## Contract: resolve_acl_root

### Inputs
- config: ConfigResolver, required — The FE-07 resolver.
- cli_flag: str | None, optional — Value of `--acl`, or the `acl=` argument to `create_cli()` when it is a path.

### Errors
- (none raised — an unresolvable root is reported as `None`)

### Returns
- On success: str | None — The resolved ACL root path, or `None` when tier 4 was reached and the default path does not exist.

### Properties
- async: false
- thread_safe: true (read-only after construction)
- pure: false (reads os.environ and the filesystem)

---

### 4.2 Loading and attaching

Given a resolved root, the CLI applies **exactly** the directory convention apcore's `ACL.discover` documents, then delegates the parse to `ACL.load`. The CLI MUST NOT reimplement YAML rule parsing — rule-key closure, `effect`/`approval` enum closure, and pattern-array arity are apcore's contract and are conformance-tested there.

Algorithm:

1. If the resolved path does not exist → attach nothing. Return `None`. **This is a hard invariant** (PROTOCOL_SPEC §6.1 missing-path rule): the CLI MUST NOT synthesize an empty ACL, because an empty ACL with `default_effect: deny` denies every call in every project that lacks an `acl/` directory.
2. If the path is a directory → load `<root>/global_acl.yaml`. If that file is absent → attach nothing, return `None`.
3. If the path is a file → load it directly.
4. On success, call `executor.set_acl(acl)`.

Attachment is skipped when the caller supplied its own executor **and** did not pass an explicit `acl=`. An embedded host that constructed its own `Executor` owns its own governance; silently attaching a CLI-discovered ACL over the host's configuration would change enforcement behind the host's back. An explicit `acl=` is an instruction and is always honoured.

## Contract: load_cli_acl

### Inputs
- root: str, required — Resolved ACL root (file or directory).

### Errors
- ConfigNotFoundError / equivalent — the path vanished between resolution and load. Exit `47`.
- ACLRuleError / equivalent — the file is structurally invalid (bad `default_effect`, unknown rule key, malformed pattern array, non-mapping `conditions`). Exit `47`.

### Returns
- On success: ACL | None — the loaded ACL, or `None` when the conventional file is absent.

### Properties
- async: false
- thread_safe: true
- pure: false (reads the filesystem)

---

### 4.3 Identity flags

apcore deliberately makes `Context.caller_id` unsettable by callers — it is managed exclusively by `Context.child()`, so a top-level CLI invocation is always the effective caller `@external`. The CLI MUST NOT fabricate a `caller_id`; doing so would let any user assume any module's identity by passing a flag.

What *is* settable is the identity, via `Context.create(identity=…)`, and that is what the `roles` and `identity_types` conditions read. FE-14 therefore adds three global flags, applied when the CLI builds the `Context` for `apcli exec`, `apcli validate`, and business-module dispatch:

| Flag | Repeatable | Maps to |
|------|-----------|---------|
| `--identity-id ID` | no | `Identity.id` |
| `--identity-type TYPE` | no | `Identity.type` (default `user`) |
| `--role ROLE` | yes | `Identity.roles` |

**Help strings are normative.** These flags are registered at the **root**, and the `apcli-visibility` conformance fixtures byte-match root `--help` across all three SDKs — so the help text is not a per-SDK stylistic choice. Left to each implementation it diverged immediately. Use exactly:

| Flag | Help text |
|---|---|
| `--acl <PATH>` | `Path to the ACL file or directory (default: ./acl)` |
| `--identity-id <ID>` | `Assert Identity.id for ACL conditions. Unauthenticated assertion, not authentication.` |
| `--identity-type <TYPE>` | `Assert Identity.type for ACL conditions (default: user). Unauthenticated assertion, not authentication.` |
| `--role <ROLE>` | `Assert an Identity role for ACL conditions. Repeatable. Unauthenticated assertion, not authentication.` |

The "unauthenticated assertion" clause on all three identity flags is required by §7 rule 1, which obliges the CLI to state it in `--help` for each flag. Value placeholders are uppercase per the conformance README's canonical help format.

`Identity.type`'s `"user"` default is fixed by this section; whether an SDK holds it in a named constant is a local choice and is **not** part of the cross-SDK surface — unlike `DEFAULT_IDENTITY_ID`, whose value is observable in `Identity.id` and is therefore normative in both value and export name.

When none of the three is given, no `Identity` is constructed and `Context.create()` is called as it is today — conditional rules keyed on `roles` or `identity_types` then simply do not match, with apcore's once-per-rule "no context" warning.

**Default identity id.** `--role` and `--identity-type` are usable without `--identity-id`, but apcore's `Identity` requires an id. The CLI supplies the literal `@cli`, exported as `DEFAULT_IDENTITY_ID`. Both the value and the constant name are normative and MUST be identical in all three SDKs — left to each implementation, this produced `@cli`/`DEFAULT_IDENTITY_ID` in one and `cli`/`DEFAULT_CLI_IDENTITY_ID` in another. The `@` prefix follows apcore's convention for synthetic principals (`@external`, `@system`) so the value cannot be mistaken for a real user whose id happens to be `cli`. Note this is `Identity.id`, not `caller_id`: it feeds no built-in condition and receives no special pattern-matching treatment.

**Context construction.** A `Context` is built when any identity flag, `--depth`, or (for `acl check`) `--input` is supplied; otherwise `None` is passed. This distinction is load-bearing: §6.5 makes a rule with conditions a plain non-match when no context is supplied, so passing a synthesized empty context would silently change which rules match.

!!! warning "Identity flags are not authentication"
    These flags are unauthenticated caller assertions, exactly like `--caller` on `apcli acl check`. They are useful for *evaluating* a rule set locally. A deployment that needs the identity to be trustworthy must supply it through FE-05 auth, not through argv. §7 states this normatively.

### 4.4 `apcli acl list`

```
apcli acl list [--format {table,json,csv,yaml,jsonl}]
```

Renders `acl.default_effect` and each rule in definition order, which is also evaluation order (first-match-wins, no priority sorting).

```
Default effect: deny   (source: ./acl/global_acl.yaml, 3 rules)

  # │ Effect │ Approval     │ Callers        │ Targets          │ Conditions        │ Description
────┼────────┼──────────────┼────────────────┼──────────────────┼───────────────────┼──────────────────────
  0 │ deny   │ not_required │ @external      │ system.control.* │ —                 │ no external control
  1 │ allow  │ required     │ *              │ db.migrate       │ roles             │ migrations need a human
  2 │ allow  │ not_required │ $or, admin.*, … │ *               │ max_call_depth    │ admins
```

The `Conditions` column lists condition **keys** only, comma-joined in lexicographic order; full condition bodies are available in `--format json`. This keeps the table readable while the machine format stays lossless.

JSON shape:

```json
{
  "source": "./acl/global_acl.yaml",
  "default_effect": "deny",
  "rules": [
    {
      "index": 0,
      "effect": "deny",
      "approval": "not_required",
      "callers": ["@external"],
      "targets": ["system.control.*"],
      "conditions": null,
      "description": "no external control"
    }
  ]
}
```

With no ACL attached: `table` prints `No ACL configured.` and exits `0`; `json` emits `{"source": null, "default_effect": null, "rules": []}` and exits `0`. Listing nothing is not an error.

### 4.5 `apcli acl check`

```
apcli acl check <TARGET> [--caller ID] [--identity-id ID] [--identity-type TYPE]
                         [--role ROLE]... [--depth N] [--input JSON]
                         [--format {table,json}]
```

Evaluates a **simulated** call through `ACL.check_access()` — a pure query that executes nothing. `--caller` accepts an arbitrary string here (unlike real execution, where the caller is always `@external`) precisely because nothing runs: the point is to answer "what would this rule set do to caller X". `--caller` defaults to `@external`, i.e. what an actual `apcli exec` would present.

`--input JSON` supplies the argument map used by the `arguments` condition (§6.1.7), which reads the governance projection and matches on **key presence only**. `--depth N` populates a synthetic call chain for `max_call_depth`.

**The identity flags appear at both levels, and their help text is pinned at both.** `acl check` carries its own `--identity-id` / `--identity-type` / `--role` in addition to the root ones from §4.3, because scoping an identity to a single query reads far better than `apcore-cli --role admin apcli acl check db.read`. Two spellings of one flag inside one CLI is a defect regardless of which level is byte-matched, so the subcommand copies MUST reuse the §4.3 wording verbatim. Only `--caller` is unique to this command:

| Flag | Help text |
|---|---|
| `--caller <ID>` | `Simulated caller ID (default: @external). Nothing is executed, so any value is accepted.` |
| `--depth <N>` | `Simulated call-chain depth for the max_call_depth condition.` |
| `--input <JSON>` | `Argument map for the arguments condition. Key presence only; values are not compared.` |

**Precedence:** a subcommand-level identity flag overrides its root-level counterpart for that invocation; a root flag not restated at the subcommand level still applies. Only the identity triple is layered this way — `--caller`, `--depth`, and `--input` exist solely on `acl check`.

The command MUST call `check_access()` and MUST NOT call `check()`. The boolean `check()` fails closed on approval — it returns `false` for a call that is allowed but needs a human — which would report "denied" for a rule set that in fact permits the call. Both axes must be shown separately.

```
Target:   db.migrate
Caller:   @external
Decision: ALLOW  (rule #1: "migrations need a human")
Approval: REQUIRED
Reason:   rule_match
```

## Contract: acl check

### Inputs
- target: str, required — Target module ID.
- caller: str, optional — Simulated caller ID; default `@external`.
- identity_id / identity_type / roles / depth / input: optional — Context construction inputs (§4.3).

### Errors
- No ACL attached — message `No ACL configured; nothing to check.`; exit `47`.

### Returns
- On success: exit `0` when `access == "allow"`, exit `77` when `access == "deny"`.

### Properties
- async: false
- thread_safe: true
- pure: true (no execution; one audit entry is emitted, see §4.8)

---

An allow-with-approval outcome exits `0`. Authorization and approval are independent axes (§6.1.6); the call *is* permitted, and conflating "needs a human" with "denied" would make the exit code unusable for the scripted policy checks this command exists for. The `approval_required` field carries that second axis.

### 4.6 `apcli acl validate`

```
apcli acl validate [--format {table,json}]
```

Runs `ACL.validate_rules()` and reports every `RuleValidationFinding`: unregistered condition keys, malformed `$or`/`$not` bodies, non-mapping `conditions`, malformed `callers`/`targets`, and tier-2 "never matches" rules that load cleanly and protect nothing.

```
3 findings:

  Rule │ Path              │ Key        │ Effect │ Sync │ Async
───────┼───────────────────┼────────────┼────────┼──────┼───────
     1 │ roles             │ roles      │ deny   │  no  │  no
     2 │ $or[1].mispelled  │ mispelled  │ allow  │  no  │  no
     4 │ targets           │ —          │ deny   │  no  │  no

A finding on a `deny` rule is the consequential one: that rule now denies
every call it matches.
```

The `Sync` and `Async` columns MUST be rendered separately and MUST NOT be collapsed into one boolean (§6.1.3 rule 3). A finding with `sync=no, async=yes` is an async-only handler: working under `async_check()`, unevaluable under `check()`.

Exit `0` when there are no findings, `47` when there is at least one. Exiting non-zero on any finding is the strict, CI-friendly default; the JSON output carries each finding's `effect` so a caller that wants to gate only on `deny` rules can do so. A `--fail-on {any,deny}` flag is deferred — see §9.

`validate_rules()` is a *runtime* check by design: condition handlers are registered process-wide and legitimately after load, so `ACL.load` only warns about unregistered keys. This command is the deterministic check to run once registration is complete.

### 4.7 `apcli acl status`

```
apcli acl status [--strict] [--format {table,json}]
```

Renders `Executor.governance_state()` — nine observations about what is actually gating the registry. `acl_configured` alone is not the answer: the ACL and approval gates are pipeline *steps*, and the `internal`, `testing`, and `minimal` strategies remove them, so an executor can hold an ACL that no step ever consults.

```
Control modules registered:   yes
Read modules registered:      yes
ACL configured:               yes  (./acl/global_acl.yaml)
Built-in ACL gate wired:      yes
Approval handler configured:  yes
Built-in approval gate wired: yes
Policy strict:                no
All control modules gated:    no
─────────────────────────────────
Unprotected control surface:  NO
```

Exit `0` always, unless `--strict` is passed and `unprotected_control_surface` is true, in which case exit `47`. The flag exists so a deployment can fail its own startup check without parsing output.

### 4.8 Audit wiring

apcore emits exactly one `AuditEntry` per `check_access()` call, but only through an `audit_logger` callback — and nothing in apcore wires the `acl.audit.*` config keys to one. The CLI is a legitimate consumer: when `acl.audit.enabled` is true (default `true` whenever an ACL is attached), the CLI installs its FE-05 `AuditLogger` as the callback, so ACL decisions land in `~/.apcore-cli/audit.jsonl` beside execution records.

**No upstream change is needed.** All three SDKs accept the callback as a constructor argument — Python `ACL(rules, default_effect, audit_logger=…)`, TypeScript `new ACL(rules, defaultEffect, auditLogger)`, Rust `ACL::new(rules, default_effect, audit_logger)`. What none of them offers is a *lossless* attach after `ACL.load()`: `load` takes no callback, so the CLI reads the file and then constructs the ACL it actually attaches.

```
src = ACL.load(resolved_path)
acl = ACL(src.rules, src.default_effect, audit_logger=cli_audit_logger)
```

Two requirements on that construction, both load-bearing:

1. **`default_effect` MUST be carried from the source ACL, never hardcoded.** A file may legitimately declare `default_effect: allow`; a rebuild that passes a literal `"deny"` silently inverts the governing default of every call no rule matched. `ACL.load` reads it as `data.get("default_effect", "deny")`, so the file's value is authoritative and only `src.default_effect` reproduces it.
2. **The rebuilt ACL loses `reload()`.** `reload()` requires the `_yaml_path` that only `ACL.load` sets, and raises `ACLRuleError("Cannot reload: ACL was not loaded from a YAML file")` otherwise. This costs the CLI nothing — no apcore-cli SDK calls `reload()` on any path — but it MUST NOT be hidden: an embedder that needs reloading supplies its own ACL through `create_cli(acl=…)`, which §4.2 attaches unchanged and never rebuilds.

Rust additionally exposes `ACL::set_audit_logger`, which preserves the loaded-from-YAML provenance. An SDK MAY use whichever mechanism its runtime offers; the observable contract is the same in all three — the attached ACL emits one `AuditEntry` per decision, and carries the file's rules and `default_effect` unchanged.

The callback is installed only when `acl.audit.enabled` is true. With auditing disabled the CLI attaches the ACL from `ACL.load` directly, with no rebuild and no callback, and apcore emits nothing beyond a debug line — so the `reload()` caveat above applies only to the auditing path.

`acl.audit.include_denied` governs **denied** decisions, matching the normative definition in apcore's `schemas/acl-config.schema.json` ("Whether to log denied access attempts", default `true`). The CLI adopts that meaning rather than forking a similarly-named key with inverted semantics:

| `enabled` | `include_denied` | Written to the audit log |
|---|---|---|
| `false` | (any) | nothing |
| `true` | `true` (default) | allow and deny decisions |
| `true` | `false` | allow decisions only |

The 13 `AuditEntry` fields are written verbatim in their `snake_case` wire form; the CLI MUST NOT rename or drop any of them, `handler_error` and `approval_required` included.

**Key order is normative.** The audit log is JSONL, so an unspecified order makes the same decision serialize to different bytes in different SDKs — the divergence class this spec has already hit on flag help text and identity sentinels. Fields MUST be emitted in apcore's own `AuditEntry` declaration order:

```
timestamp, caller_id, target_id, decision, reason, matched_rule,
matched_rule_index, identity_type, roles, call_depth, trace_id,
handler_error, approval_required
```

An SDK whose runtime surfaces these camelCased (TypeScript) MUST convert on write; the record on disk is `snake_case` in every SDK. A test asserting the serialized key list **equals** this sequence — not merely contains these names — pins field set, order, and casing in one assertion.

**When no FE-05 audit logger is installed**, the callback writes nothing and fails silently, matching `AuditLogger`'s own write-failure posture. A logging fault MUST NOT change an access decision: the callback swallows its own errors and the decision stands.

**Exactly these 13 fields, and no others.** The CLI MUST NOT add fields of its own — notably not the `user` field FE-05 puts on *execution* records. A consumer must be able to read an ACL record against apcore's `AuditEntry` rather than against a CLI dialect of it. The T-ACL-26 assertion is therefore an exact key-list equality, which is also the single place to change if cross-SDK parity later decides to carry `user`.

**Environment booleans need an explicit spelling table.** The `acl.audit.*` keys are booleans, and the environment tier can only deliver strings — where every language's naive coercion is wrong in a different way (`bool("false")` is `True` in Python; `"0".parse::<bool>()` is an error in Rust; every non-empty string is truthy in JavaScript). All three SDKs MUST accept, case-insensitively after trimming:

| Value | Spellings |
|---|---|
| true | `true`, `1`, `yes`, `on` |
| false | `false`, `0`, `no`, `off` |

**An unrecognized spelling falls back to the key's default, not to `false`.** `acl.audit.enabled` defaults to `true`, so a typo in `APCORE_ACL_AUDIT_ENABLED` leaves auditing **on**. Reading an unparseable value as "off" would let a misspelling silently stop the audit trail — the failure an operator is least likely to notice. An SDK MAY log a warning naming the key.

**Wire-shape details, all cross-SDK contracts rather than local choices:**

- **The FE-05 sink method is `log_acl_decision`** (`logAclDecision` in TypeScript), a sibling of `log_execution` rather than a reuse of it. The two record shapes differ — execution records carry `user`, ACL records carry the 13 `AuditEntry` fields — so one method writing both would have to branch on shape.
- **An absent optional is emitted as `null`, never dropped.** apcore's own `AuditEntry` serialization may skip empty fields; the CLI's record MUST NOT, because a consumer reading JSONL positionally or asserting a fixed key set would see the shape change with the data.
- **Order comes from the record's own declaration, not from a map.** In Rust specifically, building the record with `serde_json::json!` is wrong even where it currently produces the right order: `serde_json::Map` is a `BTreeMap` unless `preserve_order` is enabled, and that feature can arrive or leave transitively, so a map literal looks correct today and silently re-sorts alphabetically when an unrelated dependency changes. Use a struct (or the language's equivalent ordered form) whose field order *is* the wire order.
- **The 13 fields are a contract, not a mirror of the runtime.** apcore's `AuditEntry` is `#[non_exhaustive]`; a 14th field added upstream MUST NOT reach the wire record until this section and the record are updated together. Silently widening the record would break byte parity with SDKs still on the older runtime.
- **A process-wide audit kill switch, where an SDK has one, suppresses ACL records too.** apcore-cli-rust honours `APCORE_CLI_AUDIT_DISABLE=1`; Python and TypeScript have no equivalent, which is a pre-existing FE-05 divergence outside this feature's scope. Where the switch exists, writing ACL records into a log the operator switched off is the wrong reading of it.

### 4.9 Interaction with existing surfaces

Attaching an ACL makes three previously-inert code paths live. None of them requires new code — this section records what starts happening.

- **`apcli validate` / `--dry-run`.** The `acl` preflight row (`docs/features/usability-enhancements.md` §3.1) begins reporting real verdicts, and a denial exits `77` via the existing cascade. An ACL denial also suppresses module-level `preflight()`/`preview()` introspection, so `predicted_changes` is empty for a denied caller — correct, and already handled since the CLI reads only `valid`/`checks`/`requires_approval`.
- **Approval gate union.** Since apcore 0.28.0 the gate fires on the union of the module annotation, an ACL rule carrying `approval: required`, and `gate_destructive`. With an ACL attached, a module annotated `requires_approval: false` can now legitimately route to `CliApprovalHandler`. This path is already covered by each SDK's `acl-argument-scoped-approval` regression test.
- **Strategy skipping.** `--strategy internal|testing|minimal` removes the `acl_check` step. The existing `testing` banner already warns that ACL and approval checks are skipped; §6 extends that warning to `internal` and `minimal` when an ACL is attached, because silently bypassing a *configured* ACL is a materially different event from bypassing an absent one.

### 4.10 Every execution path MUST be gated

Attaching an ACL to the executor gates the calls that go **through that executor**. It gates nothing else — and the CLI has execution paths that build their own. Each is a silent, complete ACL bypass, and each was present in all three SDKs when FE-14 was first implemented, because §4.2 said "attach the ACL" and stopped there.

| Path | What it does | Present in |
|---|---|---|
| Sandbox subprocess (`--sandbox`) | The runner constructs a fresh `Registry` + `Executor` from `APCORE_EXTENSIONS_ROOT` and calls it directly. No ACL is attached. | Python, TypeScript, Rust |
| Filesystem script modules | Discovered executables are spawned as subprocesses and never reach `Executor.call`. | Rust (`FsDiscoverer`) |

The sandbox case inverts the user's intent: `--sandbox` is a **security** flag, so switching on stronger isolation switches off access control. A rule set that denies `system.control.*` is enforced for a plain call and ignored for a sandboxed one.

**Requirement.** Before delegating to any execution path that does not carry the attached ACL, the CLI MUST reach an access decision itself and MUST refuse a denied call with exit `77` — before the subprocess is spawned, not after. The decision belongs in the parent process, which already holds the ACL; that keeps one enforcement point rather than one per execution mechanism.

The CLI MUST NOT rely on the child re-loading the ACL as its only control. The sandbox forwards a narrow environment allowlist by design, so a child's view of `acl.root` is neither guaranteed nor trustworthy as a gate. Forwarding it for defence in depth is permitted; treating it as the control is not.

An ACL-sourced `approval: required` on one of these paths composes with the annotation before the CLI's own approval gate runs, exactly as apcore's gate does for a normal call — otherwise the same rule would demand a human on one path and not the other.

**The gate MUST always supply a `Context`, and MUST supply the argument projection.** This is where §4.3's rule does *not* generalize. §4.3 says a `Context` is built only when an identity flag or `--depth` is given; that is correct for `apcli acl check`, which *simulates* a call and is honestly context-free when nothing was asserted. It is wrong here. §6.5 makes **every** conditional rule a non-match when a call supplies no context, while apcore's pipeline creates a context at Step 1 for **every** real call — so a gate passing `None` would leave conditional `deny` rules inert on the delegated path while they fire in-process. That is the same silent bypass this section exists to close, one level down. The gate MUST therefore pass a real context (the identity-bearing one when flags were given, otherwise one reproducing what the executor would build for a `None` context), and MUST pass the call's arguments as the governance projection.

**Omitting the projection fails in both directions, not just open.** An `arguments` condition with no projection to read is UNEVALUABLE, and §6.1.1 resolves UNEVALUABLE toward refusing access: a `deny` rule *takes effect* and denies, while an `allow` rule does not grant. So a gate that forgets the projection silently **denies** calls an `arguments`-scoped `deny` rule was written to permit, as well as failing to grant ones an `allow` rule covered. A test that only checks "the denied call is denied" will pass against this defect; the discriminating case is an `arguments` condition whose key is **absent** from the call, which must be permitted and is not.

### 4.11 Registration

`acl` is registered as a nested group under `apcli`, mirroring the existing `apcli config get|set` and `apcli init module` precedent. Four edit sites per SDK, matching the FE-13 §4.9 dispatcher pattern:

1. `register_acl_command(apcli_group, executor, acl)` in the new module.
2. A `("acl", requires_executor=True, …)` row in the registrar table.
3. `"acl"` added to `APCLI_SUBCOMMAND_NAMES`.
4. Rust only: a dispatch arm in `main.rs`.

`acl` is **not** in `_ALWAYS_REGISTERED`; under `mode: include` it registers only when explicitly listed. It is not a system command and does not gate on `system.health.summary` availability.

!!! note "Known per-SDK gap: `apcli acl` is unreachable in TypeScript standalone mode"
    `acl` is a `requires_executor` entry, and the TypeScript SDK has no standalone registry — it discards the resolved extensions directory (`void resolvedExtDir; // Will be used when apcore-js registry is wired`) and installs a fallback registry that exits `47`. The group therefore never registers there outside an embedded host that injects a registry and executor.

    Root resolution and loading still run: a malformed ACL file exits `47` rather than letting the CLI proceed unprotected, which is the correct fail-closed posture. The group lights up on its own once that registry is wired — no FE-14 change will be needed. This is pre-existing debt unrelated to ACL, tracked alongside the FE-15b prerequisite in `openapi-import.md` §8.1.

---

## 5. Configuration Precedence

| Key | Env | Default | Description |
|-----|-----|---------|-------------|
| `acl.root` | `APCORE_ACL_ROOT` | `./acl` | ACL file or directory. Missing → no enforcement. |
| `acl.audit.enabled` | `APCORE_ACL_AUDIT_ENABLED` | `true` | Write ACL decisions to the FE-05 audit log. When `false`, no audit callback is installed at all. |
| `acl.audit.include_denied` | `APCORE_ACL_AUDIT_INCLUDE_DENIED` | `true` | Whether **denied** access attempts are logged. When `false`, only allow decisions are written. Semantics are apcore's (`schemas/acl-config.schema.json`), not the CLI's. |

There is deliberately **no `acl.enabled: false` switch.** A key whose only effect is to silently disable access control is a foot-gun that reads as configuration; apcore does not offer one either. To disable enforcement, point `acl.root` at a path that does not exist, or remove the file. The absence of a kill switch is intentional and is restated in §7.

---

## 6. Error Handling

| Condition | Exit Code | Error Message | Reference |
|-----------|-----------|---------------|-----------|
| ACL root missing / conventional file absent | 0 | (silent — no enforcement, no message above `--log-level INFO`) | FR-14-02 |
| ACL file structurally invalid | 47 | `Invalid ACL configuration in {path}: {detail}` | FR-14-09 |
| ACL file vanished between resolve and load | 47 | `ACL file not found: {path}` | FR-14-09 |
| `apcli acl check` → deny | 77 | `Access denied: {caller} -> {target}` | FR-14-04 |
| `apcli acl check` / `validate` / `list` with no ACL attached | 47 (`check`, `validate`) / 0 (`list`) | `No ACL configured; nothing to check.` | FR-14-03 |
| `apcli acl validate` reports >= 1 finding | 47 | (findings table on stdout) | FR-14-05 |
| `apcli acl status --strict` with unprotected control surface | 47 | `Unprotected control surface.` | FR-14-06 |
| Module execution denied by ACL | 77 | `Error: Permission denied for module '{id}'.` | Existing, FR-DISP-002 AF-8 |

### 6.1 New exit-code mapping

`ACL_RULE_ERROR` is a real apcore `ErrorCode` that no SDK's exit map carries, so a malformed ACL file currently falls through to the generic `1` — indistinguishable from "the module ran and failed". All three SDKs MUST add:

```
ACL_RULE_ERROR -> 47   (CONFIG_INVALID)
```

`47` is correct rather than `77`: the ACL could not be *read*, which is a configuration fault, not a denial. `77` must stay reserved for an actual access decision, or scripts branching on it will misreport a broken config as a permissions problem.

This is a cross-SDK map change of the kind the v0.11.0 audit caught for `DEPENDENCY_NOT_FOUND`; the maps MUST be compared key-for-key programmatically after the change, not spot-read.

### 6.2 Strategy bypass warning

When an ACL is attached and the selected strategy omits `acl_check`, the CLI emits to stderr:

```
⚠ Using '{strategy}' strategy — the configured ACL is not enforced.
```

The existing `testing` banner is extended to `internal` and `minimal` and reworded to say *configured*, because bypassing a real rule set is not the same event as running with no rules at all.

---

## 7. Security Considerations

1. **The identity flags are assertions, not authentication.** `--identity-id`, `--identity-type`, `--role`, and `acl check --caller` are unauthenticated argv values. A rule set that grants on `roles: [admin]` is trivially satisfiable by anyone who can run the binary. This is acceptable and useful for local evaluation, and unacceptable as a deployment's only control. Deployments needing trustworthy identity must source it from FE-05 auth. The CLI MUST document this in `--help` for each flag.
2. **`caller_id` is never forged.** Real execution always presents `@external`. The CLI MUST NOT expose a flag that sets `Context.caller_id`.
3. **No silent disable.** See §5.
4. **A missing ACL is not a denial.** Preserving apcore's missing-path invariant is a security-relevant *availability* property: synthesizing a default-deny ACL would break every existing project on upgrade.
5. **Audit entries may carry identity data.** `AuditEntry.roles` and `identity_type` reach `~/.apcore-cli/audit.jsonl`, which FE-05 already treats as sensitive. No new handling is required, but the widened field set is noted here.

---

## 8. Verification

| Test ID | Description | Expected Result |
|---------|-------------|-----------------|
| T-ACL-01 | No `acl/` directory, run any command | No ACL attached; behaviour identical to pre-FE-14; exit 0. |
| T-ACL-02 | `acl/global_acl.yaml` present in cwd | ACL attached; `apcli acl status` reports `ACL configured: yes`. |
| T-ACL-03 | `--acl ./custom.yaml` | Tier 1 wins over `acl.root` in `apcore.yaml`. |
| T-ACL-04 | `APCORE_ACL_ROOT=./other` with `acl.root` in yaml | Tier 2 wins over tier 3. |
| T-ACL-05 | `acl.root` points at a directory with no `global_acl.yaml` | No ACL attached; exit 0; no error. |
| T-ACL-06 | ACL file with an unknown rule key | Exit 47, message names the rule index. |
| T-ACL-07 | ACL file with `effect: permit` | Exit 47 (`effect` enum closure). |
| T-ACL-08 | ACL file with `callers: []` | Exit 47 (pattern-array arity). |
| T-ACL-09 | `apcli acl list` with 3 rules, `--format json` | `rules` array length 3, definition order preserved, `index` 0..2. |
| T-ACL-10 | `apcli acl list` with no ACL, `--format json` | `{"source": null, "default_effect": null, "rules": []}`, exit 0. |
| T-ACL-11 | `apcli acl check db.read` against an allow rule | Exit 0, `access: allow`, matched rule index reported. |
| T-ACL-12 | `apcli acl check system.control.disable` against a deny rule | Exit 77. |
| T-ACL-13 | `apcli acl check` on an allow rule carrying `approval: required` | **Exit 0** with `approval_required: true` — must not exit 77. |
| T-ACL-14 | `apcli acl check --role admin` against `conditions: {roles: [admin]}` | Condition satisfied; allow. |
| T-ACL-15 | `apcli acl check` with no `--role` against the same rule | Rule does not match (no context), falls through to `default_effect`. |
| T-ACL-16 | `apcli acl check --input '{"force": true}'` against `arguments: {has_key: [force]}` | Condition satisfied. |
| T-ACL-17 | `apcli acl validate` with an unregistered condition key | Exit 47; finding names rule index, path, key, effect. |
| T-ACL-18 | `apcli acl validate` with an async-only handler | Finding rendered with `sync: no, async: yes`; columns not collapsed. |
| T-ACL-19 | `apcli acl validate` on a clean rule set | Exit 0, `0 findings`. |
| T-ACL-20 | `apcli acl status` with control modules and no ACL | `unprotected_control_surface: true`. |
| T-ACL-21 | `apcli acl status --strict` with unprotected surface | Exit 47. |
| T-ACL-22 | `apcli exec` a module denied by ACL | Exit 77, error names the module. |
| T-ACL-23 | `apcli validate` (dry-run) a module denied by ACL | `acl` check row `passed: false`; exit 77. |
| T-ACL-24 | ACL rule `approval: required` on a module annotated `requires_approval: false` | Routed to `CliApprovalHandler`; discriminating case: handler refuses, non-TTY, call denied. |
| T-ACL-25 | `--strategy testing` with an ACL attached | Stderr warns the *configured* ACL is not enforced. |
| T-ACL-26 | `acl.audit.enabled: true`, run a denied call | One `AuditEntry` in the audit log with `decision: deny`, all 13 fields. |
| T-ACL-27 | `acl.audit.include_denied: false`, run a denied call and an allowed call | The **deny** entry is absent; the allow entry is written. |
| T-ACL-27a | `acl.audit.enabled: false` | No audit callback installed and **no rebuild** — the ACL from `ACL.load` is attached directly; no ACL entries written for either outcome. |
| T-ACL-27b | **`default_effect: allow` in the ACL file, with `acl.audit.enabled: true`** | The attached ACL's `default_effect` is still `allow`, and a call matching no rule is **permitted**. **Discriminating for §4.8 requirement 1** — a rebuild passing a literal `"deny"` inverts this silently, and every test using a `deny`-defaulted file passes against that defect. |
| T-ACL-27c | Embedder supplies its own ACL via `create_cli(acl=…)` with auditing enabled | The supplied ACL is attached **unchanged** — not rebuilt, retains `reload()`, and carries no CLI audit callback. Assert reference identity where the language permits it (Python `is`, TypeScript `toBe`); in Rust `Executor::set_acl` takes ownership and re-wraps in a fresh `Arc`, so the pointer changes in *every* implementation and identity is not expressible — assert the three named observables on the instance recovered from the executor instead, paired with a discriminating case showing the CLI's own load path *does* install a callback. |
| T-ACL-28 | Embedded host supplies its own `executor` and no `acl=` | CLI does not attach an ACL over the host's configuration. |
| T-ACL-29 | Embedded host supplies its own `executor` **and** `acl=` | CLI attaches the explicitly-passed ACL. |
| T-ACL-30 | Exit-code maps extracted programmatically from all three SDKs | `ACL_RULE_ERROR -> 47` present in all three; 0 divergent keys. |
| T-ACL-31 | `--sandbox` a module denied by ACL (§4.10) | Exit 77; the subprocess is **never spawned**. Discriminating: the same call without `--sandbox` must also exit 77, and with the rule removed both must succeed. |
| T-ACL-32 | `--sandbox` a module the ACL allows | Runs normally; sandbox isolation unaffected. |
| T-ACL-33 | Filesystem script module denied by ACL (Rust `FsDiscoverer` path) | Exit 77; process not spawned. |
| T-ACL-34 | ACL rule with `approval: required` on a sandboxed or script-module call | Routed to `CliApprovalHandler`, same as an in-process call. |

---

## 9. Open Questions

| # | Question | Current disposition |
|---|----------|---------------------|
| 1 | Should `apcli acl validate` gain `--fail-on {any,deny}`? | Deferred. Default `any` is strict and CI-friendly; JSON output already carries per-finding `effect`. Revisit if the strict default proves noisy on real rule sets. |
| 2 | Should `apcli acl` offer rule authoring (`add`/`remove`)? | Out of scope for v1. `ACL.add_rule` inserts at index 0 (highest priority) and mutates only the in-memory ACL, not the file — a CLI `add` that does not persist would mislead, and one that rewrites YAML would need to preserve comments. |
| 3 | Should `acl.root` support multiple scoped files (`acl/{scope}_acl.yaml`)? | Deferred to apcore. PROTOCOL_SPEC §3.1 names the convention but `ACL.discover` currently loads only `global_acl.yaml`; the CLI should not diverge from the runtime. |
| 4 | Should `apcli acl check` accept `--strategy` to reflect gate bypassing? | No. `check` queries the ACL directly and never runs a pipeline; `apcli acl status` is the surface that reports whether a strategy wires the gate. |

---

## 10. Impact on Existing Features

- **FE-07 Config Resolver** — gains three `acl.*` keys. The `APCORE_ACL_ROOT` variable is the first apcore-prefixed (rather than `APCORE_CLI_`-prefixed) variable the CLI resolves besides `APCORE_EXTENSIONS_ROOT`; the existing precedent is followed, not extended.
- **No upstream prerequisite.** An earlier draft of this spec claimed FE-14's audit wiring was blocked on a public `ACL.set_audit_logger` that Python and TypeScript would have to gain. That was wrong: all three SDKs already accept the callback as a constructor argument, and §4.8's load-then-construct sequence needs nothing new. The only cost is `reload()`, which no apcore-cli SDK uses. Recorded here because the claim reached implementers before it was checked against the SDK sources.
- **FE-13 Built-in Group** — `APCLI_SUBCOMMAND_NAMES` grows from 13 to 15 with FE-15 landing alongside (14 if FE-14 lands alone). Every SDK has a drift guard asserting the registrar table covers this set; all three must be updated together.
- **FE-05 Security** — the audit log gains ACL decision records alongside execution records. The JSONL reader in `system_usage` filters on its existing keys and is unaffected.
- **FE-11 Usability** — `--dry-run`'s `acl` row and the strategy warning banner change behaviour as described in §4.9. `apcli describe-pipeline` output is unchanged.
- **Conformance** — the `apcli-visibility` golden help files change, because `apcli --help` now lists `acl`. All five `expected_help.txt` fixtures must be regenerated.
