---
description: "Output Formatter (FE-08) for TTY-adaptive rendering: rich tables for terminals, JSON for pipes, plus csv/yaml/jsonl and toolkit-delegated markdown/skill SKILL.md export styles."
---

# Feature Spec: Output Formatter

**Feature ID**: FE-08
**Status**: Ready for Implementation
**Priority**: P1
**Parent**: [Tech Design v2.0](../tech-design.md) Section 8.5
**SRS Requirements**: FR-DISC-004

---

## 1. Description

The Output Formatter provides TTY-adaptive output rendering for `apcore-cli`. It detects whether stdout is connected to a terminal or a pipe, and defaults to rich table formatting for TTY sessions or JSON for non-TTY contexts. It is used by Discovery (`list`, `describe`), the `apcli system *` and `apcli strategy *` subcommands, and `exec` output formatting.

The canonical choice set differs by data shape: `list` and `describe` (which render `ScannedModule` metadata) accept `[table, json, csv, yaml, jsonl, markdown, skill]`; `exec`, `apcli system *`, and `apcli strategy *` (which render arbitrary business / health / strategy payloads) accept `[table, json, csv, yaml, jsonl]`. `markdown` and `skill` are deliberately excluded from the latter set — the surface-aware toolkit formatters target `ScannedModule` and are not meaningful for free-form payloads.

For `markdown` and `skill` styles (added v0.9.0, this feature spec), the formatter delegates to `apcore_toolkit.format_module(s)` so the rendering is byte-identical across the Python / TypeScript / Rust SDKs (see apcore-toolkit `formatting.md` § "Annotations Rendering"). `markdown` produces an LLM-ready prose block; `skill` produces the same body wrapped in vendor-neutral YAML frontmatter (`name`, `description`) directly loadable by Claude Code (`.claude/skills/<id>/SKILL.md`) and Gemini CLI (`.gemini/skills/<id>/SKILL.md`).

---

## 2. Requirements Traceability

| Req ID | SRS Ref | Description |
|--------|---------|-------------|
| FR-08-01 | FR-DISC-004 | TTY-adaptive output format selection with `--format` override. |
| FR-08-02 | FR-DISC-001 | Table rendering for module lists using `rich.table.Table`. |
| FR-08-03 | FR-DISC-003 | Syntax-highlighted JSON rendering using `rich.syntax.Syntax`. |
| FR-08-04 | FR-DISC-004 AC-4 | `--format markdown` delegates to `apcore_toolkit.format_module(s)(..., style="markdown")` byte-identically across SDKs. (added v0.9.0) |
| FR-08-05 | FR-DISC-004 AC-5 | `--format skill` delegates to `apcore_toolkit.format_module(s)(..., style="skill")` and produces vendor-neutral Claude / Gemini SKILL.md content. (added v0.9.0) |

---

## 3. Module Path

`apcore_cli/output.py`

---

## 4. Implementation Details

### 4.1 Format Responsibility Boundaries (resolves 6.6)

The formatter surface area is split across two layers — the CLI owns
**presentation-only** formats (terminal-rendering nuance, optional deps),
and `apcore-toolkit` owns formats that **MUST be byte-identical across
the Python / TypeScript / Rust SDKs**. Adding a new format starts with
deciding which layer it belongs to.

| Format | Implementation layer | Owner | Reason |
|---|---|---|---|
| `json` | apcore-cli | CLI self-impl | Single-language stdlib (`JSON.stringify` / `json.dumps`); no cross-SDK byte-equivalence requirement |
| `table` | apcore-cli | CLI self-impl | Terminal-rendering dependent (`rich`, `cli-table3`, etc.); presentation is the whole point |
| `yaml` | apcore-cli + `pyyaml` / `js-yaml` | CLI self-impl | Optional dep, soft-fallback to `json` allowed; not part of cross-SDK conformance corpus |
| `csv` | apcore-toolkit `format_csv()` | toolkit (v0.7.0) | RFC 4180 nesting + CRLF + canonical compact JSON for nested cells — divergence between SDKs was a real bug (v0.7.0 release notes) |
| `jsonl` | apcore-toolkit `format_jsonl()` | toolkit (v0.7.0) | Canonical compact JSON per row + LF terminator + NaN/Inf → null — same byte-equivalence requirement as `csv` |
| `markdown` | apcore-toolkit `format_module(s)()` | toolkit | Surface-aware LLM-ready prose; the rendering rules (annotation summary, example dropping, prompt-injection guards) are part of the cross-SDK contract |
| `skill` | apcore-toolkit `format_module(s)()` | toolkit | Same body as `markdown` wrapped in vendor-neutral YAML frontmatter; loadable by Claude Code / Gemini CLI without per-vendor branching |

Decision rule: when adding a new format, ask *"do two consumers — one
written in PY and one in Rust — need to produce identical bytes?"*. If
yes → contribute the implementation to `apcore-toolkit` and add a
conformance fixture. If no → the format is CLI-local (probably
TTY-oriented) and can live in `apcore-cli-{py,ts,rust}` without parity.

## Contract: resolve_format

### Inputs
- explicit_format: str | None, required — Explicit format override. `None` triggers TTY detection.
  validates: expected values are members of the canonical choice set (see `format_module_list` / `format_module_detail`); invalid values should be rejected upstream by Click / the SDK's CLI parser.

### Errors
- (none raised)

### Returns
- On success: str — `"table"` when TTY and no explicit format; `"json"` when non-TTY and no explicit format; explicit_format when provided

### Properties
- async: false
- thread_safe: true
- pure: false (reads sys.stdout.isatty())

---

## Contract: format_module_list

### Inputs
- modules: list[ModuleDefinition], required — List of module definitions to render. Empty list shows "No modules found."
- format: str, required — one of `"table"`, `"json"`, `"csv"`, `"yaml"`, `"jsonl"`, `"markdown"`, `"skill"`. csv/yaml/jsonl added v0.6.0 per FE-11 §3.9; markdown/skill added v0.9.0 (this spec) and delegate to `apcore_toolkit.format_modules(..., style=...)`.
- filter_tags: tuple[str, ...], optional — Tags used to customize the empty-list message. Default: `()`.

### Errors
- (none raised)

### Returns
- On success: None — output written to stdout
- **Rust language note (audit D10-W3, 2026-05-12).** Rust's `format_module_list` returns `String` rather than writing to stdout directly; callers are expected to `println!` the returned string. This preserves the spec semantics (bytes-on-stdout) while improving testability — the Rust ecosystem prefers `String`-return for formatters and `println!`-at-call-site over implicit I/O. Python and TypeScript implementations write to stdout directly per the declared contract. Mirrors the same convention documented for `format_exec_result` below.

### Properties
- async: false
- thread_safe: false (writes to stdout)
- pure: false (I/O side effect)

---

## Contract: format_module_detail

### Inputs
- module_def: ModuleDefinition, required — Module definition to render in full detail.
- format: str, required — one of `"table"`, `"json"`, `"csv"`, `"yaml"`, `"jsonl"`, `"markdown"`, `"skill"`. csv/yaml/jsonl added v0.6.0 per FE-11 §3.9; markdown/skill added v0.9.0 (this spec) and delegate to `apcore_toolkit.format_module(..., style=...)`.

### Errors
- (none raised)

### Returns
- On success: None — output written to stdout
- **Rust language note (audit D10-W3, 2026-05-12).** Rust's `format_module_detail` returns `String` rather than writing to stdout directly; callers are expected to `println!` the returned string. Same rationale as `format_module_list` above — bytes-on-stdout semantics are preserved while keeping the formatter pure-by-language-idiom. Python and TypeScript implementations write to stdout directly per the declared contract.

### Properties
- async: false
- thread_safe: false (writes to stdout)
- pure: false

---

## Contract: format_exec_result

### Inputs
- result: Any, required — Module execution result. Supports dict, list, str, None, or any str-convertible type.
- format: str | None, optional — Output format hint. Allowed values: `"json"` (default for non-TTY), `"table"` (default for TTY), `"csv"`, `"yaml"`, `"jsonl"`. Default: `None` (auto via `resolve_format`). csv/yaml/jsonl added v0.6.0 per FE-11 §3.9. **Note:** `"markdown"` and `"skill"` are NOT accepted here — `exec` returns arbitrary business results (not `ScannedModule` metadata) so the surface-aware toolkit formatters do not apply.
- fields: tuple[str, ...] | str | None, optional — Dot-path field selector for `--fields`. When set, the formatter projects only the named fields from each result row before rendering. Added v0.6.0 per FE-11 §3.9.

### Errors
- (none raised — non-serializable types use `default=str` fallback)

### Returns
- On success: None — output written to stdout; empty stdout when result is None
- **Rust language note**: returns `String` instead of writing to stdout directly; the dispatcher caller (`cli.rs`) is responsible for `println!`. Net observable behavior matches Python (`click.echo`) and TypeScript (`process.stdout.write`); the return-string form is for testability and composability per Rust convention.

### Properties
- async: false
- thread_safe: false (writes to stdout)
- pure: false

---

### 4.1 Function: `resolve_format`

**Signature**: `resolve_format(explicit_format: str | None) -> str`

Logic steps:
1. If `explicit_format is not None`: return `explicit_format`.
2. If `sys.stdout.isatty()`: return `"table"`.
3. Else: return `"json"`.

### 4.2 Function: `format_module_list`

**Signature**: `format_module_list(modules: list[ModuleDefinition], format: str, filter_tags: tuple[str, ...] = ()) -> None`

Logic steps:
1. If `format == "table"`:
   a. Create `table = rich.table.Table(title="Modules")`.
   b. Add columns: `"ID"`, `"Description"`, `"Tags"`.
   c. For each module:
      - `desc = _truncate(module.description, 80)`.
      - `tags = ", ".join(module.tags)`.
      - `table.add_row(module.canonical_id, desc, tags)`.
   d. If no modules and `filter_tags`:
      - Print `f"No modules found matching tags: {', '.join(filter_tags)}."`.
   e. Elif no modules:
      - Print `"No modules found."`.
   f. Print table via `Console().print(table)`.
2. Elif `format == "json"`:
   a. For each module:
      - `_meta = module.metadata or {}`
      - `_disp = _meta.get("display") or {}`
      - `_cli = _disp.get("cli") or {}`
      - `mid = _cli.get("alias") or _disp.get("alias") or module.canonical_id`
      - `desc = _cli.get("description") or module.description`
      - `tags = _disp.get("tags") or module.tags`
      - Append `{"id": mid, "description": desc, "tags": tags}` to result list.
   b. Print `json.dumps(result, indent=2)` via `click.echo`.
3. Elif `format in ("markdown", "skill")` (added v0.9.0):
   a. Adapt each `ModuleDefinition` / `ModuleDescriptor` to the toolkit's `ScannedModule` shape (or pass through if compatible). The adapter MUST preserve `module_id`, `description`, `input_schema`, `output_schema`, `annotations`, `tags`, `examples`, and `display`.
   b. Call `apcore_toolkit.format_modules(scanned, style=format, display=True)`.
   c. Print the returned string via `click.echo` (or the SDK's stdout primitive).
   d. Empty input: still print empty string (no "No modules found." chrome — markdown/skill outputs are file-shaped and chrome would corrupt downstream pipelines).

### 4.3 Function: `format_module_detail`

**Signature**: `format_module_detail(module_def: ModuleDefinition, format: str) -> None`

Logic steps:
1. If `format == "table"`:
   a. Print `rich.panel.Panel(f"Module: {module_def.canonical_id}")`.
   b. Print `"\nDescription:\n  {module_def.description}\n"`.
   c. If `module_def.input_schema`:
      - Print `"\nInput Schema:"`.
      - Print `rich.syntax.Syntax(json.dumps(module_def.input_schema, indent=2), "json", theme="monokai")`.
   d. If `module_def.output_schema`:
      - Print `"\nOutput Schema:"`.
      - Print `rich.syntax.Syntax(json.dumps(module_def.output_schema, indent=2), "json", theme="monokai")`.
   e. If `module_def.annotations` and non-empty:
      - Print `"\nAnnotations:"`.
      - For each `(k, v)`: print `f"  {k}: {v}"`.
   f. Collect `x_fields = {k: v for k, v in vars(module_def).items() if k.startswith("x_") or k.startswith("x-")}`.
   g. If `x_fields`:
      - Print `"\nExtension Metadata:"`.
      - For each `(k, v)`: print `f"  {k}: {v}"`.
   h. If `module_def.tags`:
      - Print `f"\nTags: {', '.join(module_def.tags)}"`.
2. If `format == "json"`:
   a. Build dict with all non-None fields.
   b. Print `json.dumps(result, indent=2)`.
3. If `format in ("markdown", "skill")` (added v0.9.0):
   a. Adapt `module_def` to `ScannedModule` shape.
   b. Print `apcore_toolkit.format_module(scanned, style=format, display=True)` via `click.echo`.

### 4.4 Function: `format_exec_result`

**Signature**: `format_exec_result(result: Any, format: str | None = None, fields: tuple[str, ...] | str | None = None) -> None`

Logic steps:
1. Resolve format via `resolve_format(format)` (TTY-adaptive default).
2. If `fields` is set, project the named dot-paths from `result` before rendering (see §4.6).
3. Dispatch on the resolved format:
   - `"json"`: print `json.dumps(result, indent=2, default=str)`. (Tier 3 — stdlib, see tech-design ADR-09.)
   - `"table"`: render via Rich (or `comfy-table` / hand-rolled table in TS); for dict-of-scalars use a 2-column key/value table; for list-of-dicts use one row per item with columns from union-of-keys. (Tier 2 — SDK-native presentation.)
   - `"csv"` (added v0.6.0; **promoted to toolkit-delegated tier v0.7.0**): MUST delegate to `apcore_toolkit.format_csv(rows, bom=False)`. The toolkit owns the canonical contract: header = union of keys across all rows (fixes prior single-row-keys data-loss bug); cell values serialized via toolkit's canonical encoder (Python repr / `str()` MUST NOT be used for non-scalars); RFC 4180 CRLF terminator; embedded `,` / `"` / `\n` / `\r` quote-wrapped. (Tier 1 — byte-equivalent across SDKs.)
   - `"yaml"` (added v0.6.0): print via idiomatic per-SDK YAML emitter (PyYAML / js-yaml / serde_yaml_ng). Currently Tier 2 (SDK-native, may differ across SDKs); toolkit-delegated byte-equivalent YAML is pending — see tech-design ADR-09 § YAML status.
   - `"jsonl"` (added v0.6.0; **promoted to toolkit-delegated tier v0.7.0**): MUST delegate to `apcore_toolkit.format_jsonl(rows)`. Toolkit's canonical compact JSON form (no inter-token whitespace, insertion-order preserved, whole-number floats drop trailing `.0`, NaN/Inf → null, LF terminator). (Tier 1 — byte-equivalent across SDKs.)
4. If `result is None`: print nothing (empty stdout).
5. For non-mapping/non-list scalar `result` and a tabular format, render as a single-row table; for `"json"`/`"jsonl"` wrap the scalar.

### 4.5 Function: `format_module_detail` extensions

`format_module_detail` and `format_module_list` accept the same extended format set when called from the `apcli` group's `list`/`describe` paths. Backward-compatible mappings: legacy callers that pass only `"json"` or `"table"` continue to work.

### 4.6 Field Selection (`--fields`, added v0.6.0)

The `--fields` flag accepts a comma-separated list of dot-paths (e.g. `--fields id,description,metadata.author`). The formatter:

1. Parses the comma list into `paths: list[str]`.
2. For each row in `result` (or the single dict if not a list), traverses the dot-path with `dict.get` at each segment; missing keys → empty string in csv/table, `null` in json/yaml/jsonl.
3. Renders only the projected columns.

`--fields` applies to all five formats (`json`, `table`, `csv`, `yaml`, `jsonl`); the projection happens before format dispatch.

> Cross-ref: [usability-enhancements.md §3.9](usability-enhancements.md) (FE-11), [Tech Design §8.13.x](../tech-design.md).

### 4.5 Helper: `_truncate`

```python
def _truncate(text: str, max_length: int = 80) -> str:
    if len(text) <= max_length:
        return text
    return text[:max_length - 3] + "..."
```

---

## 5. Parameter Validation

| Parameter | Type | Valid Values | Invalid Handling | SRS Reference |
|-----------|------|-------------|------------------|---------------|
| `explicit_format` | `str \| None` | `"table"`, `"json"`, `"csv"`, `"yaml"`, `"jsonl"`, `"markdown"`, `"skill"`, `None` | `None` triggers TTY detection. Invalid values should be caught by Click upstream. `"markdown"` / `"skill"` are accepted only for `list` / `describe` / `system *` / `strategy *` (not `exec`). | FR-DISC-004 |
| `modules` | `list` | List of `ModuleDefinition` objects. | Empty list: display "No modules found." message. | FR-DISC-001 |
| `result` (exec) | `Any` | Any JSON-serializable type, string, or None. | Non-serializable: use `default=str` fallback. | — |

---

## 6. Error Handling

| Condition | Behavior | SRS Reference |
|-----------|----------|---------------|
| Empty module list | Display "No modules found." (table) or `[]` (JSON). Exit 0. | FR-DISC-001 AF-1 |
| Result not JSON-serializable | Use `default=str` in `json.dumps`. | — |
| Terminal without color support | `rich` auto-detects. Can be forced via `NO_COLOR=1` env var. | NFR-PRT-002 |

---

## 7. Verification

| Test ID | Description | Expected Result |
|---------|-------------|-----------------|
| T-OUT-01 | `resolve_format(None)` in TTY | Returns `"table"`. |
| T-OUT-02 | `resolve_format(None)` in non-TTY | Returns `"json"`. |
| T-OUT-03 | `resolve_format("json")` in TTY | Returns `"json"` (explicit overrides TTY). |
| T-OUT-04 | `format_module_list` with 2 modules, format="table" | Rich table with 2 rows. |
| T-OUT-05 | `format_module_list` with 0 modules, format="table" | "No modules found." message. |
| T-OUT-06 | `format_module_list` with modules, format="json" | Valid JSON array. |
| T-OUT-07 | `format_module_detail` with full metadata, format="table" | Syntax-highlighted schemas, annotations, tags. |
| T-OUT-08 | `format_module_detail` with minimal metadata (no output_schema) | Output Schema section omitted. |
| T-OUT-09 | `format_exec_result` with dict | JSON-formatted dict. |
| T-OUT-10 | `format_exec_result` with None | Empty stdout. |
| T-OUT-11 | Description truncation at 80 chars | 77 chars + "..." for text > 80 chars. |
| T-OUT-12 | `format_exec_result(rows, format="csv")` | Output MUST be byte-identical to `apcore_toolkit.format_csv(rows)` (toolkit conformance corpus at `apcore-toolkit/conformance/fixtures/format_csv.json`). Header = union of all row keys; nested values = canonical compact JSON; RFC 4180 CRLF terminator. (added v0.6.0; toolkit-delegated v0.7.0) |
| T-OUT-12a | `format_exec_result([{a:1},{a:2,b:3}], format="csv")` | Heterogeneous-keys regression — output header MUST be `a,b\r\n`; later-row extras MUST appear; first row emits empty cell for missing key. Output: `a,b\r\n1,\r\n2,3\r\n`. (added v0.7.0) |
| T-OUT-12b | `format_exec_result([{schema:{type:"object"}}], format="csv")` | Nested-object regression — cell MUST contain canonical JSON `{"type":"object"}` (double-quoted, doubled per RFC 4180), NOT Python repr `{'type': 'object'}`. (added v0.7.0) |
| T-OUT-13 | `format_exec_result(rows, format="yaml")` | YAML stream emitted via per-SDK idiomatic YAML library (Tier 2 — SDK-native; byte-equivalence pending). (added v0.6.0) |
| T-OUT-14 | `format_exec_result(rows, format="jsonl")` | Output MUST be byte-identical to `apcore_toolkit.format_jsonl(rows)` (toolkit conformance corpus at `apcore-toolkit/conformance/fixtures/format_jsonl.json`). Canonical compact JSON per row, LF terminator, no trailing blank. (added v0.6.0; toolkit-delegated v0.7.0) |
| T-OUT-15 | `format_exec_result(rows, format="json", fields="id,metadata.author")` | Output rows projected to only the two named dot-paths; missing keys produce `null`. (added v0.6.0) |
| T-OUT-16 | `format_module_list(modules, format="markdown")` | stdout matches `apcore_toolkit.format_modules(modules, style="markdown")` byte-for-byte. (added v0.9.0) |
| T-OUT-17 | `format_module_detail(module, format="skill")` | stdout starts with `---\nname: …\ndescription: …\n---\n` followed by the same Markdown body as `format="markdown"`. (added v0.9.0) |
| T-OUT-18 | `apcli exec mod --format skill` / `apcli system health --format markdown` | Click rejects with exit code 2 (markdown/skill not in `exec`/`system *`/`strategy *` choice sets — those payloads are not `ScannedModule` data). (added v0.9.0) |
