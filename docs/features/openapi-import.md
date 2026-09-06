---
description: "OpenAPI Import (FE-15) adds the apcli openapi subcommand group. FE-15a delivers scan and generate — reading an OpenAPI 3.x document into ScannedModule form and materializing it as binding artifacts. FE-15b, deferred, adds binding-driven and startup HTTP proxying."
---

# Feature Spec: OpenAPI Import

**Feature ID**: FE-15
**Status**: FE-15a Draft (this release) · FE-15b Deferred
**Priority**: P1
**Parent**: [Tech Design v2.0](../tech-design.md) Section 8.16

**SRS Requirements**: FR-OAPI-001, FR-OAPI-002, FR-OAPI-003
**Related Features**: FE-04 (Discovery), FE-08 (Output Formatter), FE-09 (Grouped Commands), FE-10 (Init Command), FE-12 (Exposure Filtering)
**Requires**: apcore-toolkit >= 0.11.0

---

## 1. Description

OpenAPI spec-driven discovery has been a deferred item since v0.8.0. The CHANGELOG note recorded both the deferral and its reasoning:

> **Issues #15 (OpenAPI spec-driven discovery) and #16 (RFC 8628 device-auth flow) deferred** out of v0.8.0 scope. Both belong primarily in `apcore-toolkit` (with thin cli-side adapters), require their own RFCs, and span 2–3 release cycles each. Tracked for v0.9+.

apcore-toolkit 0.11.0 ships the toolkit half in all three languages: `OpenAPIScanner` turns an OpenAPI 3.0/3.1 document into a `ScannedModule` list, `derive_module_id` gives that list byte-identical naming across SDKs under a 24-case conformance corpus, and `HTTPProxyRegistryWriter` can make the results executable as HTTP proxies.

FE-15 is the CLI-side adapter, delivered in **two stages**.

### 1.1 Scope split

**FE-15a — this release.** The document-to-artifact half:

- **`apcli openapi scan <SOURCE>`** — read a document and show what it would produce.
- **`apcli openapi generate <SOURCE> -o DIR`** — materialize the scan as `.binding.yaml` files or host-language source.

Neither command registers a module, builds an executor, or issues a request to the described API. `scan` of a local file performs no network I/O at all; `scan` of an `http(s)://` source fetches exactly one document, the one named on the command line. This is a genuinely read-and-write-files capability, and it needs no registry — which is why it can ship in all three SDKs simultaneously.

**FE-15b — deferred.** The execution half:

- `--binding` loading of OpenAPI-derived artifacts, dispatched to `HTTPProxyRegistryWriter` so the operations become executable commands.
- `--openapi <SOURCE>` startup registration directly from a document, with no intermediate files.
- Full HTTP proxy semantics, including correct parameter-location encoding.

§8 records why each of these is deferred and what must land first. The short version: two independent prerequisites, one upstream in apcore-toolkit and one in the CLI's own TypeScript and Rust SDKs.

!!! warning "FE-15a does not make an API callable"
    A user who runs `openapi generate` gets binding files. Passing those files to `--binding` does **not** yet produce working commands — see §8.1. The commands' `--help` and the README must not imply otherwise. This limitation is the whole reason for the split, and stating it plainly is part of the deliverable.

### 1.2 Scope boundary against the toolkit

The CLI is an adapter, not a second implementation. Everything the toolkit already owns stays there:

- **Module ID derivation is the toolkit's.** `derive_module_id` is the primary subject of the cross-SDK conformance corpus and MUST match byte-for-byte in Python, TypeScript, and Rust. The CLI MUST call it and MUST NOT re-derive, normalize, kebab-case, or otherwise post-process the IDs it returns. A CLI that renames modules would silently break the guarantee three SDKs are tested against.
- **Schema extraction is the toolkit's.** `input_schema` merges path, query, and body parameters into one flat object schema; FE-02 then converts that to flags exactly as it does for any other module. No OpenAPI-specific flag handling is added.
- **The execution contract is the toolkit's.** Exactly two flat metadata keys carry it: `http_method` (uppercase, mandatory) and `url_path`. The CLI MUST NOT invent additional routing keys.

Semantic verb naming (`POST /tasks` → `tasks create` rather than `tasks.post`) is explicitly **out of scope**; it is the subject of the separate draft `ideas/cli-naming-convention.md` and would have to land in the toolkit's derivation algorithm, not here. §9 records this.

---

## 2. Requirements Traceability

| Req ID | SRS Ref | Stage | Description |
|--------|---------|-------|-------------|
| FR-15-01 | FR-OAPI-001 | 15a | `apcli openapi scan <SOURCE>` renders the modules an OpenAPI document would produce, in every FE-08 format. |
| FR-15-02 | FR-OAPI-001 | 15a | Scanner warnings (unresolvable `$ref`, external `$ref`, no 2xx response, duplicate ID) are surfaced, not swallowed. |
| FR-15-03 | FR-OAPI-002 | 15a | `apcli openapi generate <SOURCE> -o DIR` writes `.binding.yaml` files carrying an intact routing contract. |
| FR-15-04 | FR-OAPI-002 | 15a | `generate` emits binding YAML only; no host-language source writer is offered, because none can resolve an OpenAPI `target` (§4.4). |
| FR-15-05 | FR-OAPI-001 | 15a | `--include` / `--exclude` / `--prefix` / `--no-deprecated` are forwarded verbatim to `OpenAPIScanner.scan`. |
| FR-15-06 | FR-OAPI-001 | 15a | Operations that FE-15b will be unable to proxy correctly are reported at scan and generate time. |
| FR-15-07 | FR-OAPI-002 | 15a | Credentials supplied to fetch a document are never written into a generated artifact. |
| FR-15-08 | FR-OAPI-003 | 15b | `--binding` registers OpenAPI-derived artifacts as executable HTTP proxies. |
| FR-15-09 | FR-OAPI-003 | 15b | `--openapi <SOURCE>` registers operations at startup with no intermediate files. |
| FR-15-10 | FR-OAPI-003 | 15b | Per-operation registration failures are reported as warnings without aborting the batch. |

---

## 3. Module Paths

=== "Python"

    ```
    apcore_cli/openapi_cmd.py    register_openapi_command (new)
    apcore_cli/openapi_source.py load_openapi_source, detect_proxy_hazards (new)
    apcore_cli/factory.py        registrar table entry
    apcore_cli/builtin_group.py  APCLI_SUBCOMMAND_NAMES
    ```

=== "TypeScript"

    ```
    src/openapi-cmd.ts      registerOpenapiCommand (new)
    src/openapi-source.ts   loadOpenapiSource, detectProxyHazards (new)
    src/main.ts             registrar table entry
    src/builtin-group.ts    APCLI_SUBCOMMAND_NAMES
    ```

=== "Rust"

    ```
    src/openapi_cmd.rs      register_openapi_command, dispatch_openapi (new)
    src/openapi_source.rs   load_openapi_source, detect_proxy_hazards (new)
    src/main.rs             dispatch arm
    src/lib.rs              registrar table
    src/builtin_group.rs    APCLI_SUBCOMMAND_NAMES
    ```

FE-15a touches no config module and no executor construction path — it introduces no config keys and needs no registry.

---

## 4. Implementation Details

### 4.1 Source loading

`SOURCE` is a local filesystem path or an `http(s)://` URL, taken **verbatim**. The CLI MUST NOT probe candidate paths (`/openapi.json`, `/v3/api-docs`, …); a wrong URL must produce an honest 404 rather than a surprising success against a different document. Format detection is content sniffing, not file extension — a leading `{` or `[` is parsed as JSON, everything else as YAML.

All of this is `load_spec`'s behaviour and the CLI delegates to it.

## Contract: load_openapi_source

### Inputs
- source: str, required — Local path or `http(s)://` URL, verbatim.
- headers: list[str], optional — Repeated `--header "Key: Value"` values; ignored for local files.
- timeout: float, optional — Seconds; default `30.0`.

### Errors
- Source unreadable or unfetchable — exit `47`, message `Cannot read OpenAPI source '{source}': {detail}`.
- Document is not OpenAPI 3.0.x / 3.1.x — exit `47`, message reproduces the toolkit's, which names the offending `openapi` value and states that Swagger 2.0 is unsupported.
- Malformed JSON / YAML — exit `47`.
- HTTP support unavailable in this build — exit `47`, naming the missing extra or feature (§4.6).

### Returns
- On success: dict — the parsed document.

### Properties
- async: false (Python) / true (TypeScript, Rust)
- thread_safe: true
- pure: false (filesystem and, for a URL source, one network fetch)

---

!!! warning "Timeout units differ upstream"
    `load_spec`'s timeout is seconds in Python and Rust and **milliseconds** in TypeScript, and `HTTPProxyRegistryWriter` defaults to 60 s where `load_spec` defaults to 30 s. The CLI MUST expose a single `--openapi-timeout SECS` in **seconds** in all three SDKs and convert at the call boundary. A user must not have to know which SDK they are running.

### 4.2 `apcli openapi scan`

```
apcli openapi scan <SOURCE>
    [--include REGEX] [--exclude REGEX] [--prefix PREFIX] [--no-deprecated]
    [--header "K: V"]... [--openapi-timeout SECS]
    [--format {table,json,csv,yaml,jsonl,markdown,skill}]
```

Loads the document, runs `OpenAPIScanner().scan(...)`, and renders the result. Nothing is written and no module is registered.

The scan options map one-to-one onto `scan()` keyword arguments — `--include`/`--exclude` to `include`/`exclude`, `--prefix` to `base_path_prefix`, `--no-deprecated` to `include_deprecated=false` — and are forwarded verbatim. The hooks (`transform_operation`, `derive_module_id`, `transform_module`) are **not** exposed on the CLI: overriding derivation hands back the cross-SDK naming guarantee, which is not a thing a command-line flag should be able to do silently.

Rendering reuses FE-08 wholesale. `scan()` returns `ScannedModule` values, which is exactly the type `format_modules` / `format_module` already accept — the existing `markdown` and `skill` styles work with no adaptation, and the `_descriptor_to_scanned` bridge the formatter uses for registry descriptors is simply not needed on this path.

```
7 operations from ./openapi.yaml (OpenAPI 3.1.0, Petstore 1.0.0)

  Module ID            │ Route                  │ Description             │ Tags
───────────────────────┼────────────────────────┼─────────────────────────┼───────
  listPets             │ GET /pets              │ List all pets           │ pets
  createPets           │ POST /pets             │ Create a pet            │ pets
  showPetById          │ GET /pets/{petId}      │ Info for a specific pet │ pets
  pets.petid.delete    │ DELETE /pets/{petId}   │                         │ pets

2 warnings:
  showPetById          no 2xx response defined; output_schema is empty
  pets.petid.delete    external $ref not fetched: ./common.yaml#/Error

1 operation(s) cannot be proxied by FE-15b (deferred HTTP proxy dispatch):
  createPets           POST sends `limit`, `dry_run` in the body, not the query string
```

The hazard block is rendered inline and names each affected operation with its method and its offending parameter names. There is no separate flag to expand it — an earlier draft's sample referenced an `--explain-hazards` option that was never part of the command signature, and no such flag exists.

Warnings MUST be rendered, not dropped. The scanner is a degrade-with-warning design: an operation with an unresolvable `$ref` still yields a module, just a less useful one, and the warning is the only signal that the resulting flags are incomplete. In machine formats each module carries its own `warnings` array.

Exit `0` even when warnings or hazards are present — a partially-understood document is a successful scan. Only the error conditions in §6 exit non-zero.

### 4.3 Proxy-hazard detection

The toolkit's proxy writer decides body-versus-query by HTTP method alone: `POST`/`PUT`/`PATCH` send every non-path input as a JSON body, everything else as a query string. It has no parameter-location information to do otherwise, because `OpenAPIScanner` deliberately does not record one — the toolkit judged that emitting `path_params`/`query_params`/`body_params` would create a second source of truth that looks authoritative and is ignored.

The consequence is that a **query parameter declared on a `POST`, `PUT`, or `PATCH` operation would be sent in the request body**. The failure is silent: the server ignores the value or rejects the request, and nothing in the CLI reports a fault. This is the defect that defers FE-15b (§8.2).

FE-15a cannot fix it, but it can and MUST make it visible. The CLI holds the raw document, which still carries `parameters[].in`, so it can identify affected operations without duplicating any toolkit routing logic — this is a diagnostic, not a routing decision.

## Contract: detect_proxy_hazards

### Inputs
- spec: dict, required — The parsed OpenAPI document.
- modules: list[ScannedModule], required — The scan result, for module-ID correlation.

### Errors
- (none raised — a malformed `parameters` entry yields no hazard rather than an exception)

### Returns
- On success: list[Hazard] — one entry per affected operation, carrying `module_id`, `http_method`, `url_path`, and the offending parameter names.

### Properties
- async: false
- thread_safe: true
- pure: true

---

Hazards are reported by `scan` and by `generate`, are counted separately from scanner warnings, and never change the exit code in FE-15a. In machine formats they appear under a top-level `hazards` key rather than inside a module's `warnings` array, because they are a statement about a *future* execution path rather than about the scan that just ran.

### 4.4 `apcli openapi generate`

```
apcli openapi generate <SOURCE> -o DIR
    [--dry-run] [--force]
    [<all scan options>]
```

Writes the scanned modules to disk as `<id>.binding.yaml` through the toolkit's `YAMLWriter`, in every SDK. This is the cross-language artifact: the same document produces comparable output from Python, TypeScript, and Rust.

`--dry-run` lists the paths that would be written without creating them. `--force` overwrites existing files; without it, an existing file is skipped with a warning and the command still exits `0` — matching `apcli init`'s non-destructive default.

!!! note "There is deliberately no host-language source writer"
    An earlier draft of this spec offered `--writer native`, mapping to the toolkit's `PythonWriter` / `TypeScriptWriter` / `RustWriter` on the `apcli init` precedent that each SDK scaffolds in its own language. **It cannot work, for the same reason `RegistryWriter` cannot (§4.5).** Every toolkit source writer resolves `target` as a `module.path:callable` import path and rejects anything else — `PythonWriter._generate_code` raises `ValueError: Invalid target format: 'GET /pets'. Expected 'module.path:callable'.` — while an OpenAPI-derived `target` is always a route descriptor. The flag could therefore never succeed for any input this command can produce.

    Emitting genuine host-language source for an OpenAPI operation means emitting an **HTTP proxy implementation**, which is a different generator from the stub-importing one the toolkit ships. That belongs with FE-15b and its upstream work, not here. `generate` produces binding artifacts only.

**What the artifact must carry.** `YAMLWriter` emits `target` and `metadata` verbatim from the `ScannedModule`, and `BindingLoader` reads `metadata` back through a prototype-pollution filter that strips only `__proto__`, `constructor`, and `prototype`. The routing keys therefore survive a full round-trip. FE-15a asserts this explicitly (T-OAPI-20) because FE-15b depends on it entirely:

```yaml
bindings:
  - module_id: "createPets"
    target: "POST /pets"          # route descriptor, NOT an import path — see §4.5
    metadata:
      http_method: "POST"          # uppercase, mandatory
      url_path: "/pets"            # leading slash, braces retained
      openapi:
        spec_version: "3.1.0"
        operation_id: "createPets"
```

**No base URL is written.** A base URL in the artifact would be metadata that nothing in this release consumes — precisely the "looks authoritative and is ignored" antipattern the toolkit spec warns against, and the reason the CLI is not free to invent routing keys. The URL becomes part of the artifact in FE-15b, where a dispatcher exists to read it. §8.3 records the resolution order that will apply then, including the fact that baking a URL makes an artifact environment-specific.

**No credentials are written.** Headers supplied via `--header` to fetch a protected document exist only for that fetch. The CLI MUST NOT copy them into any generated file, and MUST NOT copy `securitySchemes` or credential-bearing examples from the document — the scanner already declines to lift those into metadata.

### 4.5 The `target` field is not an import path

`OpenAPIScanner` sets `target` to a route descriptor such as `"GET /pets"`. This is a documented, deliberate deviation from `target`'s usual `module.path:callable` meaning, safe upstream because the proxy writer reads `metadata` and never `target`.

It is **not** safe for the generic binding path: `RegistryWriter._to_function_module` resolves `target` as a dotted import path, so a `.binding.yaml` carrying `target: "GET /pets"` cannot be registered by it. Any implementation that routes OpenAPI-derived bindings through `RegistryWriter` will fail at registration.

FE-15a does not register anything, so this is inert here. It is recorded in this spec, rather than only in FE-15b's, because it is the property that makes the FE-15b dispatch design mandatory rather than optional, and because a reader of a generated artifact needs to know that its `target` is descriptive.

### 4.6 Dependency implications

Fetching a document over `http(s)://` is an optional dependency upstream, and each SDK gates it differently:

| SDK | Upstream gate | Required change |
|-----|--------------|-----------------|
| Python | `httpx` lives in the `http-proxy` extra; `load_spec` imports it lazily | `apcore-cli` depends on `apcore-toolkit[http-proxy]` |
| TypeScript | none — `loadSpec` is an unconditional export | none |
| Rust | `load_spec` sits behind the `http-proxy` **feature flag** | `apcore-toolkit = { version = ">=0.11.0", features = ["http-proxy"] }` |

Local-file scanning and both writers need none of this. An SDK that cannot reach the HTTP path MUST fail with an actionable message naming the missing extra or feature, never with a bare `ImportError` or a missing-symbol link error.

### 4.7 Registration

`openapi` is registered as a nested group under `apcli`, mirroring `apcli config` and `apcli init`. There is no root-level entry point: root-level shims were retired in v0.8 and `RESERVED_GROUP_NAMES` keeps `apcli` the only reserved root name, so `<cli> apcli openapi <sub>` is the sole path.

Four edit sites per SDK, per the FE-13 §4.9 pattern:

1. `register_openapi_command(apcli_group)` in the new module.
2. An `("openapi", requires_executor=False, …)` row in the registrar table — FE-15a needs neither registry nor executor.
3. `"openapi"` added to `APCLI_SUBCOMMAND_NAMES`.
4. Rust only: a dispatch arm in `main.rs`.

`openapi` is not in `_ALWAYS_REGISTERED` and is not a system command.

---

## 5. Configuration

FE-15a introduces **no configuration keys**. Every input is a command argument. The `openapi.*` keys sketched for startup registration belong to FE-15b and are specified in §8.3.

---

## 6. Error Handling

| Condition | Exit Code | Error Message | Reference |
|-----------|-----------|---------------|-----------|
| Source file missing / unreadable | 47 | `Cannot read OpenAPI source '{source}': {detail}` | FR-15-01 |
| URL fetch fails (network, 4xx, 5xx) | 47 | `Cannot read OpenAPI source '{source}': HTTP {status}` | FR-15-01 |
| Malformed JSON / YAML | 47 | `Cannot parse OpenAPI source '{source}': {detail}` | FR-15-01 |
| Swagger 2.0 or non-3.x document | 47 | Toolkit message, verbatim (names the `openapi` value found) | FR-15-01 |
| HTTP support missing (Python extra / Rust feature) | 47 | Names the missing extra or feature and how to enable it | FR-15-01 |
| Invalid `--include` / `--exclude` regex | 2 | `Invalid regex for --{flag}: {detail}` | FR-15-05 |
| Output directory not writable | 1 | OS-level error propagated | FR-15-03 |
| Existing output file, no `--force` | 0 | WARNING per skipped file | FR-15-03 |
| Scanner warnings present | 0 | Warnings rendered; not an error | FR-15-02 |
| Proxy hazards detected | 0 | Hazards rendered; not an error in 15a | FR-15-06 |

---

## 7. Security Considerations

1. **The source is trusted input, and the CLI is the trusting caller.** `load_spec` performs no allowlisting. In the CLI the source comes from argv, which is already trusted at the same level as `--extensions-dir` (which loads and executes code). No new trust boundary is crossed. The CLI MUST NOT accept a spec URL from any less-trusted channel — notably not from a scanned document's own contents.
2. **External `$ref` is never fetched.** The toolkit refuses external refs and records a warning instead. The CLI MUST NOT add a "resolve external refs" flag: that would turn a document into a fetch instruction list.
3. **Credentials never reach disk.** See §4.4.
4. **Generated files are derived from a remote document.** A `.binding.yaml` written by `generate` carries descriptions, schemas, and a route contract taken from a document the CLI did not author. It is scaffolding for a human to review, exactly like `apcli init` output, and FE-15a neither imports, loads, nor executes anything it writes. (FE-15a emits no host-language source at all — see §4.4.)
5. **FE-15a issues no request to the described API.** The only network activity is the single document fetch when `SOURCE` is a URL. Nothing generated by this release can, on its own, cause the CLI to call the API — that capability arrives with FE-15b and carries its own review.

---

## 8. Deferred: FE-15b

FE-15b makes OpenAPI-derived modules executable. It is deferred on two independent prerequisites, neither of which is about OpenAPI.

### 8.1 Prerequisite: `--binding` is a registration path in only one SDK

The intended FE-15b delivery is `generate` → `--binding` → executable commands. That chain exists today only in Python:

| SDK | What `--binding` does today | Evidence |
|-----|----------------------------|----------|
| Python | `BindingLoader.load()` → `DisplayResolver` → **`RegistryWriter.write(scanned, registry)`** — real registration | `factory.py` `_apply_toolkit_integration` |
| TypeScript | Populates a **display-overlay map** only; the code comments that this path "doesn't go through RegistryWriter" | `main.ts` `loadBindingDisplayOverlay` |
| Rust | Constructs a `DisplayResolver`, discards it, logs one line | `main.rs`, `let _resolver = …` |

TypeScript carries a second blocker behind that one: in standalone mode it has no registry at all. The resolved extensions directory is discarded (`void resolvedExtDir; // Will be used when apcore-js registry is wired`) and the fallback registry exits `47` with "no module registry wired" the moment `list` or `describe` touches it.

FE-15b therefore requires `--binding` to become a registration path in TypeScript and Rust, and requires TypeScript's standalone registry to be wired. Both are pre-existing implementation debt unrelated to OpenAPI, and both must land before FE-15b can claim three-language parity.

**Why FE-15b is not simply shipped Python-first:** the dispatch design in §8.3 is the fix for the `target`-is-not-an-import-path defect (§4.5), and the test that proves it works is an end-to-end execution test. Shipping that in one language would leave the central correctness claim of the feature verified in one third of the ecosystem.

### 8.2 Prerequisite: parameter-location metadata in apcore-toolkit

Per §4.3, `HTTPProxyRegistryWriter` cannot correctly encode a query parameter declared on a body method, and the information needed to fix it is not carried by `ScannedModule`. The fix belongs upstream: apcore-toolkit must retain parameter locations and the proxy writer must encode query and body separately. That is an apcore-toolkit 0.12.0 change spanning three SDKs and the conformance corpus.

Until it lands, FE-15b MUST NOT register an affected operation. The required behaviour, once FE-15b exists, is **per-operation refusal**, not batch abort:

- The affected operation is not registered and produces no command.
- A WARNING names the operation, the method, and the offending parameters.
- Every other operation in the document registers normally.
- The command's exit code is unaffected.

Batch abort would contradict FR-15-10 and the `WriteResult` design, which exists so one bad module does not cost the other forty. FE-15a's hazard detection (§4.3) is the same analysis, reported one stage earlier.

### 8.3 Deferred design: binding dispatch and base URL

Recorded here so FE-15a's artifact format is built against a known consumer.

**Dispatch.** After `BindingLoader.load()`, the CLI partitions the loaded modules: an entry whose `metadata` carries both a valid `http_method` (a non-empty uppercase string in the known method set) and a valid `url_path` (a non-empty string beginning with `/`) is an OpenAPI-derived proxy module and is routed to `HTTPProxyRegistryWriter`; every other entry continues to `RegistryWriter`. Validation is part of the partition, not an afterthought: the toolkit spec marks both keys `!!! danger` because a malformed value makes the writer fall back to `GET /`, pointing every module at the API root. An entry that carries one key but not the other, or a malformed value, is refused with a named error rather than silently routed either way.

**Base URL resolution.** Every tier must be a value some dispatcher actually reads:

| Tier | Source |
|------|--------|
| 1 | `--openapi-base-url` (explicit override) |
| 2 | `metadata.openapi.base_url` in the artifact, written by `generate --base-url` |
| 3 | `metadata.openapi.server_url` — the document's own `servers[0].url`, when absolute and usable |
| — | otherwise: error naming `--openapi-base-url` |

Tier 2 exists only because tier 1 does: an artifact with a baked-in URL is environment-specific, and a single generated bundle deployed to staging and production needs the override. Multi-environment setups should omit `--base-url` at generate time and supply it at load time.

`HTTPProxyRegistryWriter.write()` never raises; a per-module failure returns a `WriteResult` carrying `verified: false` and a `verification_error`. FE-15b MUST inspect every result and emit a WARNING per failure. Absence of an exception is not evidence of success, and a refused operation (§8.2) must be visible in exactly this way.

### 8.4 Deferred verification

Recorded so the intent is not lost; these become FE-15b's matrix.

| Test ID | Description |
|---------|-------------|
| T-OAPI-30 | `generate` → `--binding DIR` → **exec** a GET proxy against a stub server; response returned as module output. |
| T-OAPI-31 | `generate` → `--binding` → a POST operation carrying `in: query` parameters is **not** registered; WARNING names it; no silently-misrouting command exists. |
| T-OAPI-32 | Same document: every unaffected operation still registers and executes. |
| T-OAPI-33 | Binding entry with `http_method` but no `url_path` → refused with a named error, not routed. |
| T-OAPI-34 | Binding entry with lowercase `http_method` → refused, not silently defaulted to `GET /`. |
| T-OAPI-35 | Base URL precedence: flag > artifact > document `server_url`. |
| T-OAPI-36 | No base URL resolvable → error naming `--openapi-base-url`. |
| T-OAPI-37 | `--openapi ./spec.yaml` startup registration → `apcli list` shows the operations. |
| T-OAPI-38 | `exec` a registered DELETE proxy → `destructive` annotation triggers the FE-03 approval gate. |
| T-OAPI-39 | `--openapi` + `expose.mode: include` → FE-12 filtering applies. |
| T-OAPI-40 | `--openapi` in embedded mode → flag not registered. |

---

## 9. Verification (FE-15a)

| Test ID | Description | Expected Result |
|---------|-------------|-----------------|
| T-OAPI-01 | `openapi scan ./petstore.yaml` | One module per operation; count matches the document. |
| T-OAPI-02 | `openapi scan ./petstore.json` | JSON parsed by content sniffing; same result as YAML. |
| T-OAPI-03 | `openapi scan` on a doc with `operationId` | Module IDs equal the toolkit's `derive_module_id` output, case preserved. |
| T-OAPI-04 | `openapi scan` on a doc without `operationId` | IDs follow the path-and-method algorithm (`pets.petid.get`). |
| T-OAPI-05 | `openapi scan --prefix api` | Every ID prefixed `api.`; prefix applied before filtering and dedup. |
| T-OAPI-06 | `openapi scan --include '^pets'` | Only matching IDs returned. |
| T-OAPI-07 | `openapi scan --exclude` with an invalid regex | Exit 2. |
| T-OAPI-08 | `openapi scan --no-deprecated` | Operations with `deprecated: true` omitted. |
| T-OAPI-09 | `openapi scan` on an op with `deprecated: "false"` (string) | Not treated as deprecated; module present. |
| T-OAPI-10 | `openapi scan` on an op with no 2xx response | Warning rendered; module still present; exit 0. |
| T-OAPI-11 | `openapi scan` on an op with an external `$ref` | Warning names the ref; ref not fetched. |
| T-OAPI-12 | `openapi scan` on a Swagger 2.0 document | Exit 47; message names `swagger` and 3.0/3.1. |
| T-OAPI-13 | `openapi scan --format json` | Valid JSON; each module carries `warnings`; hazards under a top-level `hazards` key. |
| T-OAPI-14 | `openapi scan --format markdown` / `skill` | Rendered through toolkit `format_modules`, no adaptation layer. |
| T-OAPI-15 | `openapi scan https://…` with HTTP support absent | Exit 47; message names the missing extra (Python) or feature (Rust). |
| T-OAPI-16 | `openapi scan` on a POST with `in: query` parameters | Hazard reported, named with method and parameters; exit 0. |
| T-OAPI-17 | `openapi scan` on a GET with query parameters | **No** hazard reported (query is correct for GET). |
| T-OAPI-18 | `openapi generate -o DIR` | One `.binding.yaml` per module in DIR. |
| T-OAPI-19 | `openapi generate -o DIR --dry-run` | Paths listed; no files created. |
| T-OAPI-20 | `openapi generate` artifact structure | Each file carries `target`, `metadata.http_method` (uppercase, known method), and `metadata.url_path` (leading `/`, braces retained); reloading via `BindingLoader` preserves both routing keys. |
| T-OAPI-21 | `openapi generate` over an existing file without `--force` | File unchanged; WARNING; exit 0. |
| T-OAPI-22 | `openapi generate --force` over an existing file | File overwritten. |
| T-OAPI-23 | *(withdrawn — `--writer native` removed, see §4.4)* | — |
| T-OAPI-24 | `openapi generate` with `--header` | Header value absent from every generated file. |
| T-OAPI-25 | `openapi generate` on a doc with `securitySchemes` | No credential material in any generated file. |
| T-OAPI-26 | `openapi generate` reports hazards | Same hazard set as `scan` on the same document. |
| T-OAPI-27 | `openapi scan` / `generate` with no registry wired (TS standalone) | Succeeds — neither command touches the registry. |

---

## 10. Open Questions

| # | Question | Current disposition |
|---|----------|---------------------|
| 1 | Should the CLI apply semantic verb naming (`create`/`list`/`get`) instead of `.post`/`.get`? | Out of scope. It would have to change `derive_module_id`, which is conformance-pinned across three SDKs. Tracked in `ideas/cli-naming-convention.md`; if adopted it lands in apcore-toolkit first. |
| 2 | Should `openapi generate` support `--writer registry`? | No. `RegistryWriter` registers in-process rather than writing files, and cannot resolve an OpenAPI `target` anyway (§4.5). |
| 3 | Should `--openapi` accept multiple documents? | Deferred to FE-15b. Duplicate IDs across documents need a dedup policy the toolkit's per-document `deduplicate_ids` does not provide. |
| 4 | Should scan output adopt the toolkit `TuiViewModel`? | Deferred to the tui-view-model Phase 2 work, which converts the CLI's whole `--format table` path at once. Doing it for one command would fork the renderer. |
| 5 | Should device-auth (issue #16) land with this? | No. FE-15b's `auth_header_factory` is the seam; the flow is a separate feature with its own RFC. |

---

## 11. Impact on Existing Features

- **FE-13 Built-in Group** — with FE-14 also landing, `APCLI_SUBCOMMAND_NAMES` grows from 13 to 15. The drift guard in each SDK asserting the registrar table covers this set must be updated in all three.
- **FE-08 Output Formatter** — no new formats. `scan` is the first CLI surface to hand `ScannedModule` values to the formatter without going through `_descriptor_to_scanned`, exercising an existing path more directly rather than adding one.
- **FE-02 Schema Parser** — unaffected in FE-15a, which registers nothing. When FE-15b lands, note that `input_schema` flattens path, query, and body into one object; see §4.3.
- **Dependencies** — Python gains the `http-proxy` extra, Rust the `http-proxy` feature, for URL sources only. The README Version Compatibility table records it.
- **Conformance** — `apcli --help` gains `openapi`, so all five `apcli-visibility` `expected_help.txt` fixtures change. A new `openapi_scan` CLI fixture family is **not** proposed: the derivation corpus already lives in apcore-toolkit, and duplicating it here would create a second source of truth.
