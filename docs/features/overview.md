---
description: "Features overview for apcore-cli, mapping the 13 features (FE-01 through FE-13), the core execution path, and the dependency graph linking each feature spec and the project manifest."
---

# Features Overview: apcore-cli

`apcore-cli` exposes apcore modules as CLI subcommands through a layered architecture of 13 features. The core execution path runs: **Core Dispatcher** (FE-01) → **Schema Parser** (FE-02) → **Approval Gate** (FE-03) → **Security Manager** (FE-05). Module discovery is handled by **Discovery** (FE-04) with **Grouped Commands** (FE-09) for namespace organisation, **Exposure Filtering** (FE-12) for business-module access control, and **Built-in Command Group** (FE-13) to relocate apcore-cli-provided commands under a reserved `apcli` namespace. Output is managed by **Output Formatter** (FE-08) and extended by **Usability Enhancements** (FE-11). Shell integration (FE-06), configuration (FE-07), and scaffolding (FE-10, Init Command) round out the feature set.

For the full feature list, implementation order, and dependency graph, see the [Project Manifest](../project-apcore-cli.md).

---

## Feature Specifications

| ID | Feature | Spec |
|:---|:---|:---|
| FE-01 | Core Dispatcher | [core-dispatcher.md](core-dispatcher.md) |
| FE-02 | Schema Parser | [schema-parser.md](schema-parser.md) |
| FE-03 | Approval Gate | [approval-gate.md](approval-gate.md) |
| FE-04 | Discovery | [discovery.md](discovery.md) |
| FE-05 | Security Manager | [security.md](security.md) |
| FE-06 | Shell Integration | [shell-integration.md](shell-integration.md) |
| FE-07 | Config Resolver | [config-resolver.md](config-resolver.md) |
| FE-08 | Output Formatter | [output-formatter.md](output-formatter.md) |
| FE-09 | Grouped Commands | [grouped-commands.md](grouped-commands.md) |
| FE-10 | Init Command | [init-command.md](init-command.md) |
| FE-11 | Usability Enhancements (v0.6.0) | [usability-enhancements.md](usability-enhancements.md) |
| FE-12 | Module Exposure Filtering (v0.7.0) | [exposure-filtering.md](exposure-filtering.md) |
| FE-13 | Built-in Command Group (`apcli`) (v0.7.0) | [builtin-group.md](builtin-group.md) |

---

## Upstream References

| Document | Location |
|----------|----------|
| Tech Design v2.0 | [`docs/tech-design.md`](../tech-design.md) |
| SRS v0.1 | [`docs/srs.md`](../srs.md) |
| Project Manifest | [`docs/project-apcore-cli.md`](../project-apcore-cli.md) |
