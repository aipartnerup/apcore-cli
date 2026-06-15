---
description: "Shell Integration (FE-06) providing the completion subcommand for bash/zsh/fish scripts and man-page generation in roff format, both driven by Registry module metadata."
---

# Feature Spec: Shell Integration

**Feature ID**: FE-06
**Status**: Ready for Implementation
**Priority**: P2
**Parent**: [Tech Design v2.0](../tech-design.md) Section 8.7
**SRS Requirements**: FR-SHELL-001, FR-SHELL-002

---

## 1. Description

Shell Integration provides two subcommands: `apcore-cli completion <shell>` for generating shell completion scripts (Bash, Zsh, Fish) and `apcore-cli man <command>` for generating man pages in roff format. Both commands leverage module metadata from the Registry for dynamic content generation.

---

## 2. Requirements Traceability

| Req ID | SRS Ref | Description |
|--------|---------|-------------|
| FR-06-01 | FR-SHELL-001 | Shell completion script generation for bash, zsh, fish. |
| FR-06-02 | FR-SHELL-002 | Man page generation in roff format (single-command via `man <command>`). |
| FR-06-03 | FR-SHELL-002 | Full-program man page via `--help --man` (covers all registered commands including downstream). |

---

## 3. Module Path

`apcore_cli/shell.py`

---

## 4. Implementation Details

## Contract: register_completion_command

> **Note:** The pre-v0.7 `register_shell_commands(cli, prog_name)` wrapper — which
> attached both `completion` and `man` as flat root subcommands — was **removed in
> v0.7.0 / FE-13**. The canonical wiring now registers `completion` under the `apcli`
> builtin group via `register_completion_command`, and exposes `man` as a root
> `--man` overlay via [`configure_man_help`](#contract-configure_man_help).
> `apcore-cli-rust` and `apcore-cli-typescript` dropped the equivalent wrapper in the
> same release.

### Inputs
- apcli_group: click.Group, required — Group to register the `completion` command on (the `apcli` builtin group in canonical wiring).
- prog_name: str, optional — Program name captured in the command closure for completion-script generation. Default: `"apcore-cli"`.

### Errors
- (none raised)

### Returns
- On success: None — `completion` command registered on `apcli_group` as side effect

### Properties
- async: false
- thread_safe: false
- pure: false (mutates `apcli_group` by adding a command)

---

## Contract: build_program_man_page

### Inputs
- cli: click.Group, required — Root CLI group to generate man page from.
- prog_name: str, required — Program name for `.TH` header and command references.
- version: str, required — Program version string.
- description: str | None, optional — Override program description. Defaults to program help text.
- docs_url: str | None, optional — Base URL for online docs link in SEE ALSO section.

### Errors
- (none raised)

### Returns
- On success: str — complete roff-formatted man page text

### Properties
- async: false
- thread_safe: true (read-only access to cli object)
- pure: false (reads cli command metadata)

---

## Contract: configure_man_help

### Inputs
- cli: click.Group, required — Root CLI group on which the hidden `--man` option is registered and the help interceptor is wired. (TS: `program: Command`; Rust: command tree built lazily after the pre-parse hook fires.)
- prog_name: str, required — Program name used for the `.TH` header, command labels, and SEE ALSO references in the generated man page.
- version: str, required — Program version string emitted in the `.TH` header.
- description: str | None, optional — Override program description used for `.SH NAME` / `.SH DESCRIPTION`. Default: `None` — derived from the `cli` group's help text, falling back to `"{prog_name} CLI"`.
- docs_url: str | None, optional — Base URL forwarded to `set_docs_url` for SEE ALSO and per-command help footers. Default: `None` (no online docs link).

### Errors
- (none raised) — pre-parse failure for `--man` / `--help` is non-fatal: when neither flag is present, the call is a no-op aside from registering the hidden option.

### Returns
- On success: None — registers the hidden `--man` option on `cli` and installs the help interceptor as a side effect. When `--man` and `--help` are both observed at runtime, the interceptor calls `build_program_man_page`, prints the result to stdout, and exits the process with code 0.

### Properties
- async: false
- thread_safe: false (mutates `cli` by adding a hidden option and installing a help-text hook; also calls `set_docs_url`, which mutates module-level global state)
- pure: false (mutates cli, mutates the module-level `_docs_url` global, reads the help text)

---

## Contract: set_docs_url

### Inputs
- url: str | None, required — Base URL for online documentation, or `None` to clear. (TS: `string | null`; Rust: `Option<String>`.)
  - validation: when not `None`, MUST be a non-empty string. Implementations MAY additionally require the value to start with `http://` or `https://` for safety; the canonical Python implementation does not enforce this and trusts the caller.
  - reject_with: ValueError (caller-side; not enforced by this method in the reference implementation)

### Errors
- (none raised) — invalid URLs are accepted as-is and surfaced verbatim in help output / man page; the caller is responsible for validation.

### Returns
- On success: None — updates the module-level `_docs_url` global. After the call:
  - Per-command `--help` footer appends: `Docs: {url}/commands/{command_name}` when `url` is set.
  - `build_program_man_page` SEE ALSO appends: `Full documentation at {url}` when `url` is set.
  - When `url` is `None`/`null`, no docs link is shown anywhere (the prior value is cleared).

### Properties
- async: false
- thread_safe: false — mutates a module-level global without locking. Concurrent calls from multiple threads may race; embedders MUST configure this once at startup before exposing the CLI to concurrent invocations.
- pure: false (mutates module-level global state)
- idempotent: true (calling twice with the same `url` produces the same observable state)

---

### 4.1 Command: `completion`

**Registration**: `@cli.command("completion")`

**Signature**: `completion(shell: str) -> None`

**Click decorators**:
```python
@click.argument("shell", type=click.Choice(["bash", "zsh", "fish"]))
```

Logic steps:
1. Resolve `resolved = ctx.find_root().info_name or prog_name`.
2. If `shell == "bash"`: call `_generate_bash_completion(resolved)` and `click.echo()` the result.
3. If `shell == "zsh"`: call `_generate_zsh_completion(resolved)` and `click.echo()` the result.
4. If `shell == "fish"`: call `_generate_fish_completion(resolved)` and `click.echo()` the result.
5. Exit code 0.


### 4.2 Helper: `_make_function_name`

**Signature**: `_make_function_name(prog_name: str) -> str`

Converts `prog_name` to a valid POSIX shell identifier by replacing any character that is not `[a-zA-Z0-9]` with `_` and prepending `_`. Example: `"my-tool"` → `"_my_tool"`.

### 4.3 Function: `_generate_bash_completion`

**Signature**: `_generate_bash_completion(prog_name: str) -> str`

Logic steps:
1. Compute shell function name via `_make_function_name(prog_name)`.
2. Build the inline module-list shell command as `"<prog_name>" list --format json 2>/dev/null | python3 -c "..."`. `prog_name` is double-quoted in the embedded command to handle names that contain spaces.
3. Generate the Bash completion script body (example for `prog_name="apcore-cli"`):
   ```bash
   _apcore_cli() {
       local cur prev opts
       COMPREPLY=()
       cur="${COMP_WORDS[COMP_CWORD]}"
       prev="${COMP_WORDS[COMP_CWORD-1]}"

       if [[ ${COMP_CWORD} -eq 1 ]]; then
           opts="exec list describe completion man"
           COMPREPLY=( $(compgen -W "${opts}" -- ${cur}) )
           return 0
       fi

       if [[ "${COMP_WORDS[1]}" == "exec" && ${COMP_CWORD} -eq 2 ]]; then
           local modules=$("apcore-cli" list --format json 2>/dev/null | python3 -c "import sys,json;[print(m['id']) for m in json.load(sys.stdin)]" 2>/dev/null)
           COMPREPLY=( $(compgen -W "${modules}" -- ${cur}) )
           return 0
       fi
   }
   complete -F _apcore_cli apcore-cli
   ```
4. Return the script as a string.

### 4.4 Function: `_generate_zsh_completion`

**Signature**: `_generate_zsh_completion(prog_name: str) -> str`

Logic steps:
1. Compute shell function name via `_make_function_name(prog_name)`.
2. Generate Zsh completion function using `compdef` and `compadd`. First-level completions: `exec`, `list`, `describe`, `completion`, `man`. Second-level (after `exec`): dynamic module IDs via inline command with `prog_name` double-quoted.
3. Emit `compdef {fn} {prog_name}` to register the function.
4. Return the script as a string.

### 4.5 Function: `_generate_fish_completion`

**Signature**: `_generate_fish_completion(prog_name: str) -> str`

Logic steps:
1. Generate Fish completions using `complete -c {prog_name}` commands. Subcommands: `exec`, `list`, `describe`, `completion`, `man`.
2. For `exec` second-level completion: inline command with `prog_name` double-quoted.
3. Return the script as a string.

### 4.6 Function: `register_completion_command`

**Signature**: `register_completion_command(apcli_group: click.Group, prog_name: str = "apcore-cli") -> None`

Registers the `completion` command on `apcli_group` (the `apcli` builtin group). The `prog_name` parameter is captured in the command closure so the completion generators use the correct program name at runtime (resolved from `ctx.find_root().info_name` when available, otherwise from the `prog_name` argument). `man` is no longer a flat subcommand — it is exposed as a root `--man` overlay via `configure_man_help` (§4.9).

> **Removed:** `register_shell_commands(cli, prog_name)` (the pre-v0.7 flat wrapper that attached `completion` and `man` as root subcommands) was removed in v0.7.0 / FE-13. Use `register_completion_command` for `completion` and `configure_man_help` for `man`.

### 4.7 Command: `man`

**Registration**: `@cli.command("man")`

**Signature**: `man_cmd(command: str) -> None`

**Click decorators**:
```python
@click.argument("command")
```

Logic steps:
1. Look up `command` in the CLI command registry (built-in subcommands and module IDs).
2. If not found: write to stderr "Error: Unknown command '{command}'." Exit 2.
3. Build roff document:
   a. `.TH "{PROG}-{COMMAND}" "1" "{date}" "{prog} {version}" "{prog} Manual"` — all names are dynamic from `prog_name`.
   b. `.SH NAME` section: `{prog}-{command} \\- {brief_description}` (first line of command help, trailing period stripped).
   c. `.SH SYNOPSIS` section: dynamically built from the command's actual Click parameters. For each `click.Option`: `[--flag \\fITYPE\\fR]` (required options omit brackets); for each `click.Argument`: `\\fIMETA\\fR` (optional arguments get brackets). Not a static `[OPTIONS] [ARGUMENTS]` placeholder.
   d. `.SH DESCRIPTION` section: full command description from Click help (if present).
   e. `.SH OPTIONS` section: for each Click option:
      - `.TP` with `\\fB{flag_name}\\fR \\fI{type}\\fR` (flags use `\\fB{flag_name}\\fR` only).
      - Help text on next line, followed by `Default: {value}.` if a non-flag default exists.
   f. `.SH ENVIRONMENT` section: four standard env vars:
      - `APCORE_EXTENSIONS_ROOT` — path to extensions directory, overrides default `./extensions`.
      - `APCORE_CLI_AUTO_APPROVE` — set to `1` to bypass approval prompts.
      - `APCORE_CLI_LOGGING_LEVEL` — CLI-specific log verbosity; takes priority over `APCORE_LOGGING_LEVEL`. One of: `DEBUG`, `INFO`, `WARNING`, `ERROR`. Default: `WARNING`.
      - `APCORE_LOGGING_LEVEL` — global apcore log verbosity; used as fallback when `APCORE_CLI_LOGGING_LEVEL` is not set.
   g. `.SH EXIT CODES` section: table of all exit codes:
      | Code | Meaning |
      |------|---------|
      | 0 | Success. |
      | 1 | Module execution error. |
      | 2 | Invalid CLI input or missing argument. |
      | 44 | Module not found, disabled, or failed to load. |
      | 45 | Input failed JSON Schema validation. |
      | 46 | Approval denied, timed out, or no interactive terminal available. |
      | 47 | Configuration error (extensions directory not found or unreadable). |
      | 48 | Schema contains a circular `$ref`. |
      | 77 | ACL denied — insufficient permissions for this module. |
      | 130 | Execution cancelled by user (SIGINT / Ctrl-C). |
   h. `.SH SEE ALSO` section: links to sibling man pages using `prog_name`.
4. Print the roff document to stdout.
5. Exit code 0.

**Usage for immediate display:**
```bash
apcore-cli man exec | man -l -
```

### 4.8 Function: `build_program_man_page`

**Signature (TypeScript)**: `buildProgramManPage(program: Command, progName: string, version: string, description?: string, docsUrl?: string): string`
**Signature (Python)**: `build_program_man_page(cli: click.Group, prog_name: str, version: str, description: str | None = None, docs_url: str | None = None) -> str`
**Signature (Rust)**: `build_program_man_page(cmd: &clap::Command, prog_name: &str, version: &str, description: Option<&str>, docs_url: Option<&str>) -> String`

Generates a complete roff man page covering the entire CLI program. Covers all registered commands, including downstream business commands injected via `GroupedModuleGroup`.

Logic steps:
1. Resolve description: explicit parameter, or program description, or `"{prog_name} CLI"`.
2. Generate `.TH` header with program name (uppercased), version, manual label.
3. Generate `.SH NAME`, `.SH SYNOPSIS`, `.SH DESCRIPTION`.
4. Generate `.SH GLOBAL OPTIONS`: iterate visible options on the root command, excluding `help`, `version`, `all`, `man`.
5. Generate `.SH COMMANDS`: iterate all visible subcommands:
   a. For each command: emit `.TP` with `{prog_name} {command_name}`, description, and visible options.
   b. For nested subcommands (groups): emit `.TP` with `{prog_name} {group} {subcommand}`, description, and visible options.
   c. Hidden options (from `--all-options` mode) are excluded via `visibleOptions()` / `is_hide_set()`.
6. Generate `.SH ENVIRONMENT`: standard apcore env vars.
7. Generate `.SH EXIT CODES`: standard exit code table.
8. Generate `.SH SEE ALSO`: pointer to `--help --all-options`.
9. Return the roff string.

### 4.9 Function: `configure_man_help`

**Signature (TypeScript)**: `configureManHelp(program: Command, progName: string, version: string, description?: string, docsUrl?: string): void`
**Signature (Python)**: `configure_man_help(cli: click.Group, prog_name: str, version: str, description: str | None = None, docs_url: str | None = None) -> None`
**Signature (Rust)**: Pre-parse `has_man_flag()` in main + call `build_program_man_page()` (clap has no hook system). Pass `docs_url` from caller.

Adds `--man` as a hidden global option and intercepts help output. When `--man` is present alongside `--help`, outputs the complete roff man page and exits.

Logic steps:
1. Add `--man` as a hidden option to the root command.
2. **TypeScript (Commander.js)**: Use `addHelpText("beforeAll", ...)` to intercept. If `program.opts().man` is true, call `buildProgramManPage()`, write to stdout, and `process.exit(0)`.
3. **Python (Click)**: Pre-parse `sys.argv` for `--man` and `--help`. If both present, call `build_program_man_page()`, print, and `sys.exit(0)`. (Click's eager `--help` handling requires pre-parsing.)
4. **Rust (clap)**: Pre-parse `--man` and `--help` from raw argv before clap processes arguments. If both present, build the CLI command tree (without directory validation), call `build_program_man_page()`, print, and `std::process::exit(0)`.

**Usage for downstream projects:**
```typescript
// TypeScript
import { configureManHelp } from 'apcore-cli';
configureManHelp(program, 'reach', '0.2.0', 'ReachForge: The Social Influence Engine', 'https://reachforge.dev/docs');
```
```python
# Python
from apcore_cli.shell import configure_man_help
configure_man_help(cli, "reach", "0.2.0", "ReachForge: The Social Influence Engine", "https://reachforge.dev/docs")
```

### 4.10 Function: `set_docs_url`

**Signature (TypeScript)**: `setDocsUrl(url: string | null): void`
**Signature (Python)**: `set_docs_url(url: str | None) -> None`
**Signature (Rust)**: `set_docs_url(url: Option<String>)`

Sets the base URL for online documentation. When set:
- Per-command `--help` footer appends: `Docs: {url}/commands/{command_name}`
- `build_program_man_page` SEE ALSO appends: `Full documentation at {url}`

When not set (default `null`/`None`): no docs link is shown anywhere.

**Usage for man page installation:**
```bash
# Generate man page file
reach --help --man > reach.1

# Install (symlink for development)
sudo mkdir -p /usr/local/share/man/man1
sudo ln -sf /path/to/reach.1 /usr/local/share/man/man1/reach.1

# Verify
man reach
```

---

## 5. Parameter Validation

| Parameter | Type | Valid Values | Invalid Handling | SRS Reference |
|-----------|------|-------------|------------------|---------------|
| `shell` (completion) | `click.Choice` | `"bash"`, `"zsh"`, `"fish"` | Click rejects with "Invalid value". Exit 2. | FR-SHELL-001 AF-1 |
| `command` (man) | `str` | Any valid subcommand or module ID | "Error: Unknown command '{command}'." Exit 2. | FR-SHELL-002 AF-1 |

---

## 6. Error Handling

| Condition | Exit Code | Error Message | SRS Reference |
|-----------|-----------|---------------|---------------|
| Unsupported shell | 2 | Click auto: "Invalid value for 'SHELL'..." | FR-SHELL-001 AF-1 |
| Unknown command for man | 2 | "Error: Unknown command '{command}'." | FR-SHELL-002 AF-1 |

---

## 7. Verification

| Test ID | Description | Expected Result |
|---------|-------------|-----------------|
| T-SHELL-01 | `apcore-cli completion bash` | stdout: valid Bash completion script containing "exec", "list", "describe". Exit 0. |
| T-SHELL-02 | `apcore-cli completion zsh` | stdout: valid Zsh completion script. Exit 0. |
| T-SHELL-03 | `apcore-cli completion fish` | stdout: valid Fish completion script. Exit 0. |
| T-SHELL-04 | `apcore-cli completion invalid` | Click rejects. Exit 2. |
| T-SHELL-05 | Bash completion sourced, type `apcore-cli exec math.<TAB>` | `math.add` appears as completion candidate. |
| T-SHELL-06 | `apcore-cli man exec` | stdout: roff format with `.TH`, command name. Exit 0. |
| T-SHELL-07 | `apcore-cli man exec \| man -l -` | Man page renders correctly. |
| T-SHELL-08 | `apcore-cli man nonexistent` | stderr: "Unknown command". Exit 2. |
| T-SHELL-09 | `apcore-cli --help --man` | stdout: complete roff man page with `.TH`, `.SH COMMANDS` covering all registered commands. Exit 0. |
| T-SHELL-10 | `apcore-cli --help --man \| mandoc -a` | Full man page renders correctly with all commands and options. |
| T-SHELL-11 | Downstream project calls `configure_man_help(program, "myapp", "1.0.0")`, then runs `myapp --help --man` | stdout: roff man page with program name "myapp" and all downstream commands. |
| T-SHELL-12 | `apcore-cli --help --man` without `--all-options` | Built-in options (`--input`, `--yes`, etc.) do NOT appear in the COMMANDS section. |
| T-SHELL-13 | `apcore-cli --help --man --all-options` | Built-in options appear in the COMMANDS section. |
