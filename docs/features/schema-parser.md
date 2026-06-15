---
description: "Schema Parser (FE-02) converting a module's JSON Schema input_schema into Click options: type mapping, boolean flag pairs, enum choices, required enforcement, and $ref/$defs resolution."
---

# Feature Spec: Schema Parser

**Feature ID**: FE-02
**Status**: Ready for Implementation
**Priority**: P0
**Parent**: [Tech Design v2.0](../tech-design.md) Section 8.3
**SRS Requirements**: FR-SCHEMA-001, FR-SCHEMA-002, FR-SCHEMA-003, FR-SCHEMA-004, FR-SCHEMA-005, FR-SCHEMA-006

---

## 1. Description

The Schema Parser converts a module's JSON Schema `input_schema` into Click CLI options. It handles type mapping, boolean flag pairs, enum choices, required enforcement, help text generation, and `$ref`/`$defs` resolution. The parser produces a list of `click.Option` objects that are appended to the dynamically generated module command.

---

## 2. Requirements Traceability

| Req ID | SRS Ref | Description |
|--------|---------|-------------|
| FR-02-01 | FR-SCHEMA-001 | Property-to-flag mapping with type conversion. |
| FR-02-02 | FR-SCHEMA-002 | Boolean flag pairs (`--flag/--no-flag`). |
| FR-02-03 | FR-SCHEMA-003 | Enum-to-Choice mapping. |
| FR-02-04 | FR-SCHEMA-004 | Required property enforcement. |
| FR-02-05 | FR-SCHEMA-005 | Help text from `x-llm-description` or `description`. |
| FR-02-06 | FR-SCHEMA-006 | `$ref` and `$defs` resolution with circular detection. |

---

## 3. Module Paths

- `apcore_cli/schema_parser.py` — Type mapping, option generation.
- `apcore_cli/ref_resolver.py` — `$ref` resolution, `allOf`/`anyOf`/`oneOf` flattening.

---

## 4. Implementation Details

## Contract: schema_to_click_options

### Inputs
- schema: dict, required — JSON Schema dict with `properties` and optional `required` array.
  validates: property names must not appear in the reserved CLI option set (`input`, `yes`, `large_input`, `format`, `fields`, `sandbox`, `all_options`, `dry_run`, `trace`, `stream`, `strategy`, `approval_timeout`, `approval_token`); flag names must not collide after underscore-to-hyphen conversion
  reject_with: SystemExit(48) — on reserved property name OR flag name collision (both are schema-author errors and share exit code 48 across SDKs)
- max_help_length: int, optional — Maximum help text length before truncation. Default: `1000`.

### Errors
- SystemExit(48) — schema property name conflicts with a reserved CLI option (cross-SDK parity, audit D11-NEW-005, 2026-05-08)
- SystemExit(48) — two properties map to the same `--flag` name after underscore-to-hyphen conversion

### Returns
- On success: list[click.Option] — one Click option per schema property, in iteration order of `properties`

### Properties
- async: false
- thread_safe: true
- pure: false (may call sys.exit on reserved name or flag collision)

### Cross-language notes

- **Rust** (`schema_to_clap_args` in `apcore-cli-rust/src/schema_parser.rs`): returns `Err(SchemaParserError::ReservedPropertyName)` / `Err(SchemaParserError::FlagCollision)` rather than calling `process::exit` directly. The CLI dispatcher (`apcore-cli-rust/src/cli.rs`) maps both variants to `EXIT_SCHEMA_CIRCULAR_REF` (48) so the observable exit code matches Python and TypeScript.
- **TypeScript** (`schemaToCliOptions` in `apcore-cli-typescript/src/schema-parser.ts`): calls `process.exit(EXIT_CODES.SCHEMA_CIRCULAR_REF)` (= 48) for both reserved-name and flag-collision, matching Python `sys.exit(48)`.
- The `SCHEMA_CIRCULAR_REF` constant name is historical — exit code 48 is the protocol-spec slot for general schema-validity errors and is reused across circular-ref, missing-target, depth-exceeded, reserved-name, and flag-collision violations.

---

## Contract: resolve_refs

### Inputs
- schema: dict, required — JSON Schema dict potentially containing `$ref` and `$defs`/`definitions`.
- max_depth: int, optional — Maximum **`$ref` hop count** along any single resolution chain. Default: `32`. Counts only `$ref` traversals and composition-branch (`allOf`/`anyOf`/`oneOf`) descents; does NOT increment for plain nested-properties recursion (audit D11-NEW-003 alignment, 2026-05-08).
  validates: `$ref` chain depth must not exceed `max_depth`; `$ref` targets must exist in `$defs`
  reject_with: SystemExit(48) — on circular ref or depth exceeded; SystemExit(45) — on unresolvable ref
- module_id: str, optional — Used in error messages only.

### Errors
- SystemExit(48) — circular `$ref` detected
- SystemExit(48) — `$ref` resolution depth exceeds `max_depth`
- SystemExit(45) — `$ref` target not found in `$defs`/`definitions`

### Returns
- On success: dict — fully inlined schema with all `$ref` references replaced by their targets; `$defs` key removed

### Properties
- async: false
- thread_safe: true (operates on a deep copy)
- pure: false (may call sys.exit)

### Algorithm

```
resolve_refs(schema, max_depth=32, module_id="?"):
  defs = schema["$defs"] or schema["definitions"] or {}
  result = resolve_node(deep_copy(schema), defs, depth=0, max_depth, visited={}, module_id)
  delete result["$defs"]; delete result["definitions"]
  return result

resolve_node(node, defs, depth, max_depth, visited, module_id):
  # 1. Depth guard — counts only $ref hops + composition descents.
  if depth > max_depth: exit(48, "ref chain exceeds max_depth")

  # 2. $ref resolution — circular detection via visited-path set.
  if node has "$ref":
    if node["$ref"] in visited: exit(48, "circular $ref")
    target_key = node["$ref"].split("/")[-1]
    if target_key not in defs: exit(45, "unresolvable $ref")
    return resolve_node(defs[target_key], defs, depth+1, max_depth,
                        visited ∪ {node["$ref"]}, module_id)

  # 3. allOf composition — merge sibling-first, then each branch.
  #    Required: parent.required + Σ branches[].required, dedup first-seen.
  #    Properties: parent.properties (seed), branches[].properties (later wins).
  if node has "allOf":
    merged.properties = parent.properties (copy)
    merged.required   = parent.required (copy)
    for sub in node["allOf"]:
      r = resolve_node(sub, defs, depth+1, ...)
      merged.properties.update(r.properties)        # later branch wins
      merged.required.extend(r.required)
    merged.required = dedup_first_seen(merged.required)   # cross-SDK parity
    copy_remaining_keys(node, merged, except="allOf")
    return merged

  # 4. anyOf / oneOf — Required = (sibling required) ∪ (∩ across branches).
  for keyword in ("anyOf", "oneOf"):
    if node has keyword:
      merged.properties = parent.properties (copy)
      sibling_required  = parent.required (copy)
      branch_required_sets = []
      for sub in node[keyword]:
        r = resolve_node(sub, defs, depth+1, ...)
        merged.properties.update(r.properties)
        if r.required: branch_required_sets.append(set(r.required))
      branch_required = ∩(branch_required_sets)            # set-intersection
      merged.required = dedup_first_seen(sibling_required + branch_required)
      copy_remaining_keys(node, merged, except=keyword)
      return merged

  # 5. Plain object / nested properties — recurse on .properties values without
  #    incrementing depth (depth counter is for $ref + composition only).
  if node["type"] == "object" and node has "properties":
    for k, v in node["properties"]:
      node["properties"][k] = resolve_node(v, defs, depth, max_depth, visited, module_id)
  return node
```

**Cross-SDK-critical invariants** (every implementation must preserve):
1. Depth counter increments ONLY on `$ref` hops and composition-branch descents — NOT on nested-properties recursion. Audit D11-NEW-003 (2026-05-08).
2. Sibling `required` is captured BEFORE the branch loop and merged with branch-derived `required`. Audit D11-NEW-001 (2026-05-08).
3. `required` arrays are deduped first-seen-wins after merging in BOTH `allOf` and `anyOf`/`oneOf` paths. Audit D9-NEW-002 (2026-05-08).
4. Visited-set is path-keyed (`#/$defs/X`), not value-keyed — same target reachable via two siblings is allowed; cycles are blocked.

---

### 4.1 Function: `schema_to_click_options`

**File**: `apcore_cli/schema_parser.py`

**Signature**: `schema_to_click_options(schema: dict, max_help_length: int = 1000) -> list[click.Option]`

Logic steps:
1. Extract `properties = schema.get("properties", {})`.
2. Extract `required_list = schema.get("required", [])`.
3. Initialize `options: list[click.Option] = []`.
4. Initialize `flag_names: dict[str, str] = {}` for collision detection.
5. For each `(prop_name, prop_schema)` in `properties.items()`:
   a. Compute `flag_name = "--" + prop_name.replace("_", "-")`.
   b. Check collision: if `flag_name` already in `flag_names`, exit code 48, message "Flag name collision: properties '{prop_name}' and '{flag_names[flag_name]}' both map to '{flag_name}'."
   c. Store `flag_names[flag_name] = prop_name`.
   d. Determine Click type via `_map_type(prop_name, prop_schema)`.
   e. Determine `is_required = prop_name in required_list`.
   f. Determine `help_text = _extract_help(prop_schema)`.
   g. Determine `default = prop_schema.get("default", None)`.
   h. If type is boolean: create flag pair option (see section 4.2).
   i. Else if `enum` field present: create Choice option (see section 4.3).
   j. Else: create standard `click.Option([flag_name], type=click_type, required=is_required, default=default, help=help_text)`.
   k. Append option to `options`.
6. Return `options`.

### 4.2 Function: `_map_type`

**Signature**: `_map_type(prop_name: str, prop_schema: dict) -> click.ParamType | tuple[str, bool]`

**Type mapping table:**

| JSON Schema `type` | Click Type | Special Handling |
|--------------------|-----------|------------------|
| `"string"` | `click.STRING` | If `prop_name` ends with `_file` or `x-cli-file: true`: use `click.Path(exists=True)`. |
| `"integer"` | `click.INT` | — |
| `"number"` | `click.FLOAT` | — |
| `"boolean"` | `is_flag=True` | Returns marker for boolean flag pair creation. |
| `"object"` | `click.STRING` | JSON string expected. Parsed at validation time. |
| `"array"` | `click.STRING` | JSON string expected. Parsed at validation time. |
| Unknown type | `click.STRING` | Log WARNING: "Unknown schema type '{type}' for property '{name}', defaulting to string." |
| Missing `type` field | `click.STRING` | Log WARNING: "No type specified for property '{name}', defaulting to string." |

### 4.3 Boolean Flag Pair Creation

When `type` is `"boolean"`:

```python
default_val = prop_schema.get("default", False)
option = click.Option(
    [f"--{flag_name}/--no-{flag_name}"],
    default=default_val,
    help=help_text,
    show_default=True,
)
```

Edge cases:
- `default: true` in schema: flag default is `True`. User must specify `--no-flag` to disable.
- Boolean with `enum: [true]`: ignore enum constraint, treat as standard boolean flag.

### 4.4 Enum Choice Creation

When `enum` field is present (and type is not boolean):

```python
enum_values = prop_schema["enum"]
if not enum_values:  # Empty array
    logger.warning(f"Empty enum for property '{prop_name}', no values allowed.")
    # Fall through to standard string option
else:
    string_values = [str(v) for v in enum_values]
    original_types = {str(v): type(v) for v in enum_values}
    option = click.Option(
        [flag_name],
        type=click.Choice(string_values),
        required=is_required,
        default=str(default) if default is not None else None,
        help=help_text,
    )
    # Store original_types mapping for post-parse reconversion
```

Post-parse reconversion logic:
1. After Click parses the choice as a string, check `original_types[selected_value]`.
2. If original type was `int`, convert back: `int(selected_value)`.
3. If original type was `float`, convert back: `float(selected_value)`.
4. If original type was `bool`, convert back: `selected_value.lower() == "true"`.
5. Otherwise, keep as string.

### 4.5 Required Property Handling

Logic steps:
1. Read `required` array from schema.
2. For each `prop_name` in `required`:
   a. If `prop_name` not in `properties`: log WARNING "Required property '{prop_name}' not found in properties, skipping." Continue.
   b. Set `required=True` on the corresponding Click option.
3. **STDIN interaction**: When `--input -` is used, required enforcement must be deferred:
   a. Set all options to `required=False` at Click level.
   b. After STDIN merge, validate completeness via `jsonschema.validate()`.
   c. If validation fails for missing required field: exit code 45 with field name in message.

### 4.6 Help Text Extraction

**Function**: `_extract_help(prop_schema: dict, max_length: int = 1000) -> str | None`

Logic steps:
1. Check `prop_schema.get("x-llm-description")`. If non-empty string, use it.
2. Else check `prop_schema.get("description")`. If non-empty string, use it.
3. If `max_length > 0` and selected text length > `max_length` chars: truncate to `(max_length - 3)` chars + `"..."`.
4. If neither field present: return `None`.

The `max_length` parameter defaults to 1000 and is configurable via `cli.help_text_max_length`.

### 4.7 Reference Resolution

**File**: `apcore_cli/ref_resolver.py`

**Function**: `resolve_refs(schema: dict, max_depth: int = 32, module_id: str = "") -> dict`

Logic steps:
1. Deep-copy the input schema to avoid mutation.
2. Extract `defs = schema.get("$defs", schema.get("definitions", {}))`.
3. Call `_resolve_node(schema, defs, visited=set(), depth=0, max_depth=max_depth)`.
4. Remove `$defs` and `definitions` keys from the result.
5. Return the fully inlined schema.

**Function**: `_resolve_node(node: dict, defs: dict, visited: set, depth: int, max_depth: int) -> dict`

Logic steps:
1. If `"$ref"` in `node`:
   a. `ref_path = node["$ref"]` (e.g., `"#/$defs/Address"`).
   b. If `depth >= max_depth`: exit code 48, message "$ref resolution depth exceeded maximum of {max_depth} for module '{module_id}'."
   c. If `ref_path` in `visited`: exit code 48, message "Circular $ref detected in schema for module '{module_id}' at path '{ref_path}'."
   d. Parse ref target: extract key from path (e.g., `"Address"` from `"#/$defs/Address"`).
   e. If key not in `defs`: exit code 45, message "Unresolvable $ref '{ref_path}' in schema for module '{module_id}'."
   f. Add `ref_path` to `visited`.
   g. Return `_resolve_node(defs[key], defs, visited, depth + 1, max_depth)`.
2. If `"allOf"` in `node`:
   a. Initialize `merged = {"properties": {}, "required": []}`.
   b. **Sibling preservation:** seed `merged["properties"]` from the parent node's own `properties` (if any) and seed `merged["required"]` from the parent's own `required` list (if any) BEFORE iterating branches. Parent fields fill gaps; branches win on key conflict.
   c. For each sub-schema in `node["allOf"]`:
      - Recursively resolve the sub-schema (composition descent — counts toward `max_depth`).
      - Merge its `properties` into `merged["properties"]` (later entries override).
      - Extend `merged["required"]` with sub-schema's `required`.
   d. Copy any non-composition keys from `node` (e.g., `description`) into `merged` (skipping keys already merged).
   e. Return `merged`.
3. If `"anyOf"` or `"oneOf"` in `node`:
   a. Initialize `merged = {"properties": {}, "required": []}`.
   b. **Sibling preservation (audit D11-NEW-001, 2026-05-08):** seed `merged["properties"]` from the parent node's own `properties` (if any) and capture `sibling_required = list(node.get("required", []))` BEFORE iterating branches. Per JSON Schema semantics, a parent's `required` applies in addition to the branch intersection.
   c. Collect `all_required_sets = []`.
   d. For each sub-schema:
      - Recursively resolve the sub-schema (composition descent — counts toward `max_depth`).
      - Merge its `properties` into `merged["properties"]` (union; branches win on key conflict).
      - Collect its `required` list into `all_required_sets`.
   e. Compute branch intersection: `branch_required = list(set.intersection(*all_required_sets))` if all sets non-empty, else `[]`.
   f. Combine: `merged["required"] = sibling_required ∪ branch_required` (deduplicated, sibling-first order).
   g. Copy any non-composition keys from `node` into `merged` (skipping keys already merged).
   h. Return `merged`.
4. Recursively process nested `properties` values. **Plain nested-properties recursion does NOT increment `depth`** — `max_depth` counts only `$ref` hops + composition-branch descents.
5. Return node.

---

## 5. Boundary Values & Limits

| Parameter | Minimum | Maximum | Default | SRS Reference |
|-----------|---------|---------|---------|---------------|
| `$ref` resolution depth | 0 | 32 | 32 | FR-SCHEMA-006 |
| Schema composition nesting | 0 | 3 levels | — | FR-SCHEMA-006 |
| Property name length | 1 char | No limit (ID is limited) | — | FR-SCHEMA-001 |
| Enum array size | 0 (empty) | No limit | — | FR-SCHEMA-003 |
| Help text truncation | — | Configurable (`cli.help_text_max_length`) | 1000 chars | FR-SCHEMA-005 AF-1 |

---

## 6. Error Handling

| Condition | Exit Code | Error Message | SRS Reference |
|-----------|-----------|---------------|---------------|
| Flag name collision | 48 | "Error: Flag name collision: properties '{a}' and '{b}' both map to '{flag}'." | FR-SCHEMA-001 AF-3 |
| Circular $ref | 48 | "Error: Circular $ref detected in schema for module '{id}' at path '{path}'." | FR-SCHEMA-006 AF-1 |
| $ref depth exceeded | 48 | "Error: $ref resolution depth exceeded maximum of 32 for module '{id}'." | FR-SCHEMA-006 AF-2 |
| Unresolvable $ref | 45 | "Error: Unresolvable $ref '{ref}' in schema for module '{id}'." | FR-SCHEMA-006 AF-3 |
| Unknown schema type | — (WARNING) | Log: "Unknown schema type '{type}' for property '{name}', defaulting to string." | FR-SCHEMA-001 AF-1 |
| Empty enum | — (WARNING) | Log: "Empty enum for property '{name}', no values allowed." | FR-SCHEMA-003 AF-1 |

---

## 7. Verification

| Test ID | Description | Expected Result |
|---------|-------------|-----------------|
| T-SCHEMA-01 | Property `name` of type `string` | `--name` flag accepts string value. |
| T-SCHEMA-02 | Property `count` of type `integer` | `--count 5` passes integer `5` to module. |
| T-SCHEMA-03 | Property `rate` of type `number` | `--rate 3.14` passes float `3.14`. |
| T-SCHEMA-04 | Property `quiet` of type `boolean` | `--quiet` passes `True`, `--no-quiet` passes `False`. (Uses `quiet` to avoid visual collision with the retired global `--verbose` / current `--all-options` flag.) |
| T-SCHEMA-05 | Property `data` of type `object` | `--data '{"key":"val"}'` passes JSON string. |
| T-SCHEMA-06 | Property `items` of type `array` | `--items '[1,2,3]'` passes JSON string. |
| T-SCHEMA-07 | Property `input_file` of type `string` | Flag name is `--input-file` (underscore to hyphen). |
| T-SCHEMA-08 | Property `format` with `enum: ["json","csv"]` | `--format json` passes "json". `--format yaml` rejected by Click. |
| T-SCHEMA-09 | Required property `name` omitted | Click shows "Missing required option '--name'". Exit 2. |
| T-SCHEMA-10 | Required property via STDIN | `echo '{"name":"test"}' \| apcore-cli exec mod --input -` satisfies requirement. |
| T-SCHEMA-11 | `$ref: "#/$defs/Address"` with valid def | Address properties appear as top-level flags. |
| T-SCHEMA-12 | Circular `$ref` (A -> B -> A) | Exit 48, "Circular $ref detected". |
| T-SCHEMA-13 | `$ref` depth > 32 | Exit 48, "depth exceeded". |
| T-SCHEMA-14 | `allOf` with two sub-schemas | Merged properties from both sub-schemas. |
| T-SCHEMA-15 | Help from `x-llm-description` | Help text uses `x-llm-description`, not `description`. |
| T-SCHEMA-16 | Help text > 1000 chars (default limit) | Truncated to 997 + "...". |
| T-SCHEMA-17 | Boolean default `true` | Default is `True` without explicit flag. |
| T-SCHEMA-18 | Enum with integer values `[1,2,3]` | Choice accepts "1", reconverts to int `1`. |
| T-SCHEMA-19 | Two properties collide after hyphen conversion | Exit 48, "Flag name collision". |
| T-SCHEMA-20 | File property convention (`_file` suffix) | Uses `click.Path(exists=True)`. |
