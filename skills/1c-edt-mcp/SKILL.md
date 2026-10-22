---
name: 1c-edt-mcp
description: >
  1C:Enterprise EDT MCP Server workflow for agents. Use when developing or
  deploying 1C extensions, validating queries against UNF metadata, reading
  module sources, checking project errors, or updating a live infobase.
  Covers JSON-RPC endpoint, tool discovery, parameter naming gotchas,
  project ownership for validate_query, deploy checklist, and logs.
---

# 1C EDT MCP Server — agent development bridge

EDT MCP bridges agents and the 1C:Enterprise EDT workspace (metadata, BSL,
queries, infobase deploy). Prefer it over COM (COMConnector is not registered
on this machine).

## 1. Endpoint and health

| Item | Value |
|------|-------|
| Base URL | `http://127.0.0.1:8765` |
| Health | `GET /health` → `{"status":"ok","live":true,"ready":true,"running":true,...}` |
| JSON-RPC | `/mcp`, `/mcp/rpc`, `/mcp/message` |
| Protocol | `2024-11-05` |
| Server | `edt-mcp-server` v2.15.x, EDT `2026.1.3.x` |
| Tools | ~87 (`evaluate_expression`, `debug_launch`, `get_metadata_objects`, `validate_query`, `create_infobase`, `update_database`, …) |

Workspace root (typical): `E:\1C\EDT\Test`

Projects in workspace: `UNF` (base УНФ), `unfAiAgent` (AI HTTP extension),
`UNF.Расширение1`, `UNF.ДоработанныйФункционал`, `ДоработанныйФункционал`,
`Расширение2`, `unftest`.

## 2. Discover tool parameters FIRST

**Never guess parameter names.** Call:

```text
get_tool_guide('tool_name')
```

Returns markdown with full parameter table, examples, gotchas.

**Parameter name is `toolName` (camelCase), NOT `tool_name`.**

Confirmed examples:

| Tool | Parameters |
|------|------------|
| `validate_query` | `projectName` (required), `queryText` (required), `dcsMode` (optional bool) |
| `revalidate_objects` | `projectName` + optional `objects: ['HTTPService.AI_Agent','CommonModule.X']` (English FQNs) |
| `get_tool_guide` | `toolName` |

## 3. validate_query — project ownership (CRITICAL)

`projectName` must be the project that **OWNS** the metadata, not an extension
that merely inherits it.

```text
# WRONG — fields "not found":
validate_query(projectName="Расширение2", queryText="... РегистрНакопления.ЗаказыПокупателей ...")

# RIGHT:
validate_query(projectName="UNF", queryText="... РегистрНакопления.ЗаказыПокупателей ...")
```

All «Поле не найдено» errors for base objects disappear when the owning project is used.

## 4. Error workflow after file edits

1. Change sources under `E:\1C\EDT\Test\<Project>\src\...`
2. `revalidate_objects` (project + objects)
3. `get_project_errors`
4. Iterate; some inherited-DCS warnings can persist (calculated fields with `""` format strings)

**Order matters:** always `revalidate_objects` before `get_project_errors` or results are stale.

## 5. Live infobase access without COM

`V83.COMConnector` → `80040154 Class not registered` on this machine.
Use `debug_launch` + `evaluate_expression` against a file infobase instead.

Event log (file IB): `<infobase>\1Cv8Log`, format `text-2.0`.
EDT `get_event_log` filters: `severity:[E,W]`, `eventContains`, `from`/`to`;
`offset` reads old→new (lazy). There is no `РегистрСведений.ЖурналРегистрации`.

## 6. Deploy extension to live UNF (verified)

Target: file IB `E:\Байер\Новая база`, project `unfAiAgent`.

| Step | Action |
|------|--------|
| 1 | `create_infobase` mode=`register` infobaseFile=`E:\Байер\Новая база` projectName=`unfAiAgent` setDefault=`true` → app id `ce055247-ca1d-4cea-bdcc-ad07a4a86b5b` |
| 2 | `set_infobase_credentials` user=`Администратор` password=`` access=`INFOBASE` (access=`OS` does not work; web-login `obmen`/`203501` is NOT Designer login) |
| 3 | On «Ошибка открытия соединения»: stop Designer AgentMode — `1cv8 DESIGNER /F "E:\Байер\Новая база" /AgentMode` |
| 4 | `update_database` applicationId=`ce055247-…` confirm=`true` externalInfobaseChanges=`override` |

Old app ids (do not use for live update): `unfAiAgent_ib` `470cfd7d-…`, UNF «Управление нашей фирмой Байер Новая» `8bc52b5c-…`.

## 7. Writing 1C sources with agent tools

| Rule | Detail |
|------|--------|
| BSL with Cyrillic | `write`/`edit` tools work (verified on `AI_AgentHTTPHandler` ~747 lines) |
| Inline bash | Avoid `python -c` / complex here-strings with Cyrillic — use file-based scripts (`_tmp_*.py`, `Set-Content`) |
| XML / DCS | Prefer `Copy-Item` for large templates; PowerShell `Set-Content` herestrings for surgical XML |
| `edit` on PS-written XML | May fail (BOM/whitespace) — full rewrite via PowerShell instead |

## 8. Extension compile checklist (EDT)

1. `Configuration.mdo`: `<extension xsi:type="mdclassExtension:ConfigurationExtension">`
2. `<interfaceCompatibilityMode>TaxiEnableVersion8_5</interfaceCompatibilityMode>`
3. `DT-INF/PROJECT.PMF`: `Base-Project: UNF`
4. Report MDO: `<templates>` present; forms inherited (no Forms/ / no `<defaultForm>` for reports)
5. `Template.dcs` — valid platform schema (see `1c-reports`)
6. Object names include `namePrefix` (e.g. `Расш2_`)

## 9. EDT style nags (safe to ignore when matching UNF)

- `module-structure-event-regions` vs `module-structure-method-in-regions` conflict.
  UNF style: top region `#Область ОбработчикиСобытий`.
- Prefer no extra `#Если` preprocessor in HTTP service modules (`bsl-legacy-check-if-preprocessor-part-environments`).
- CommonModule.mdo server pattern: `<server>true</server>` + `<externalConnection>true</externalConnection>` +
  `<clientOrdinaryApplication>true</clientOrdinaryApplication>` (no `serverCall`).
  Explicit false-flags trigger `common-module-type`.

## 10. Platform versions on machine

`8.3.23.1912`, `8.5.1.1343`, `8.5.1.1529`. Apache `:80` publishes 1C at `/unf/`.
EDT MCP listens on `:8765`.

## 11. Agent workflow recipe

```text
1. GET /health — MCP alive?
2. get_tool_guide('validate_query') if unsure of params
3. validate_query(projectName=<owner>, queryText=<select>)
4. Edit sources via write/edit
5. revalidate_objects → get_project_errors
6. update_database (deploy) when ready
7. Smoke via unfAiAgent HTTP (skill 1c-unf-api)
```

## Related skills

- `1c-unf-api` — runtime HTTP contract after deploy
- `1c-reports` — DCS / report modules
- `1c-coding-standards` — BSL conventions
- `1c-http-services` — HTTP service MDO/handlers

## Source

EDT MCP Server 2.15.x + workspace `E:\1C\EDT\Test` (deploy and validate paths verified 2026-10).
