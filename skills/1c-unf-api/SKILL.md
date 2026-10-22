---
name: 1c-unf-api
description: >
  Live HTTP API of unfAiAgent extension for 1C:UNF (Управление Нашей Фирмой).
  Use when an agent or script must query data, create/read/update/delete documents,
  fetch balances/turnover/reports, or search in a UNF infobase over HTTP.
  Covers wire format (RU/EN), full command set, document draft-only rule,
  auto-fill, query gotchas, report templates, and Python/PowerShell clients.
---

# unfAiAgent HTTP API — operational contract for agents

Extension `unfAiAgent` exposes a single dispatcher over HTTP. This skill is the
**agent-facing contract**: what to send, what to expect, what never to do.

## 0. Runtime facts (live environment)

| Item | Value |
|------|-------|
| Endpoint | `POST http://localhost/unf/hs/ai/api` |
| Alt (lang prefix) | `http://localhost/unf/ru/hs/ai/api` |
| Auth | HTTP Basic `obmen` / `203501` |
| Content-Type | `application/json; charset=utf-8` |
| Method | **POST only** (GET on `/unf/hs/test/ping` for smoke) |
| Infobase | `E:\Байер\Новая база` (Apache alias `/unf` → `C:/www/unf/default.vrd`) |
| Extension project | `E:\1C\EDT\Test\unfAiAgent` |
| Deploy app id | `ce055247-ca1d-4cea-bdcc-ad07a4a86b5b` |
| Handler | `AI_AgentHTTPHandler.ОбработатьЗапрос` (thin wrapper `apiHandler`) |

Without auth → `401` + `WWW-Authenticate: Basic realm="1C:Enterprise 8.5"`.
Missing `/unf` prefix → `404`.

## 1. Wire format

Every request is a single JSON object. **RU and EN keys are both accepted.**

| RU | EN | Meaning |
|----|-----|---------|
| `Команда` | `command` | command name (required) |
| `ТекстЗапроса` | `query` | 1C query text (EXECUTEQUERY) |
| `ТипДокумента` | `type` | document type, e.g. `ЗаказПокупателя` |
| `Данные` | `data` | payload object for create/update |
| `Ссылка` | `ref` | document GUID string |
| `params` | `params` | extra params (reports) |
| `Справочник` | `catalog` | catalog name (SEARCH) |
| `Строка` | `term` | search term (SEARCH) |
| `Регистр` | `register` | register name |
| `Шаблон` | `template` | report template key |
| `НачалоПериода` | `period_start` | period start |
| `КонецПериода` | `period_end` | period end |
| `Организация` | `organization` | organization filter |

**Response envelope (always):**

```json
{"success": true,  "data": { ... }}
{"success": false, "error": "message"}
```

Empty query result still returns `columns` (field metadata).

## 2. Command set (live)

| Command | Purpose | Required fields |
|---------|---------|-----------------|
| `PING` | liveness | — |
| `EXECUTEQUERY` | SELECT-only 1C query | `query` / `ТекстЗапроса` |
| `CREATEDOCUMENT` | create **draft** document | `type`, `data` |
| `GETDOCUMENT` | read document by ref | `type`, `ref` |
| `UPDATEDOCUMENT` | update document (replaces ТЧ if given) | `type`, `ref`, `data` |
| `POSTDOCUMENT` | post (conduct) document | `type`, `ref` |
| `UNPOSTDOCUMENT` | unpost | `type`, `ref` |
| `DELETEDOCUMENT` | delete (mark or hard) | `type`, `ref`, optional `hard:true` |
| `GET_SCHEMA` | metadata/schema help | optional type |
| `SEARCH` | search catalog by term | `catalog`, `term` |
| `GET_BALANCES` | register balances | optional filters |
| `GET_TURNOVER` | register turnover | optional filters |
| `GET_REPORT_DATA` | template reports | optional `template`, period, org |

Planned / not live: `EXPORT_DATA`.

## 3. Documents — create-only rule (CRITICAL)

> **Documents are created as drafts and NOT posted.**
> Use `CREATEDOCUMENT` only. `POSTDOCUMENT` is optional and out of the main scenario.

- `CREATEDOCUMENT` writes with `РежимЗаписиДокумента.Запись` → `posted: false`.
- Result is a draft document (e.g. `НФНФ-000004`).
- Drafts have **no register movements** — they do not appear in `ЗаказыПокупателей.Остатки`
  or report `template=orders` until posted.

### CREATE payload (ЗаказПокупателя)

```json
{
  "command": "CREATEDOCUMENT",
  "type": "ЗаказПокупателя",
  "data": {
    "Организация": "БАЕР ПАК ООО",
    "Контрагент": "Розничный покупатель",
    "СтавкаНДС": "22%",
    "Товары": [
      {"Номенклатура": "Тестовая номенклатура", "Количество": 1, "Цена": 5000}
    ]
  }
}
```

ТЧ name: `Запасы` or `Товары` (both accepted).

### Server-side auto-fill (if fields omitted)

| Field | Auto-fill source |
|-------|------------------|
| Договор | `ДоговорыКонтрагентов` by `Владелец`=counterparty + `Организация` (+ `ВидДоговора`=`С покупателем`) |
| ЕдиницаИзмерения | `Номенклатура.ЕдиницаИзмерения` |
| СтавкаНДС | catalog `СтавкиНДС` exact name `"22%"` (catalog, **not** enum; also exists `22% / 122%`) |
| СуммаНДС | `Сумма * 22 / 122` (VAT-inclusive) |
| Курс / Кратность | 1 |
| СуммаВключаетНДС | true |
| СостояниеЗаказа | **«Не обработан»** (default; was «Завершен» — user requirement) |
| ДатаОтгрузки | header and line |
| КоэффициентЕдиницыДляДокументов | 0 |

Explicit values in `data` win over auto-fill.

### Other CRUD shapes

```json
{"command":"GETDOCUMENT","type":"ЗаказПокупателя","ref":"<guid>"}
{"command":"UPDATEDOCUMENT","type":"ЗаказПокупателя","ref":"<guid>","data":{...}}
{"command":"DELETEDOCUMENT","type":"ЗаказПокупателя","ref":"<guid>"}              // mark
{"command":"DELETEDOCUMENT","type":"ЗаказПокупателя","ref":"<guid>","hard":true} // hard
{"command":"POSTDOCUMENT","type":"ЗаказПокупателя","ref":"<guid>"}               // optional
{"command":"UNPOSTDOCUMENT","type":"ЗаказПокупателя","ref":"<guid>"}
```

`ref` is a **GUID string**, not `e1cib/data/...` URL.
`UPDATE` replaces tabular sections `Запасы`/`Товары` if they are present in `data`.
Non-existent ref → error `«Документ не найден»`.

## 4. Query language (EXECUTEQUERY)

- **SELECT only** (no DML).
- Start simple: `ВЫБРАТЬ ПЕРВЫЕ N` / `КОЛИЧЕСТВО(*)`, then add joins.
- Avoid heavy multi-join on large tables (timeout).
- Boolean filters: `ГДЕ Проведен` / `ГДЕ НЕ Проведен` (do not use `= ИСТИНА`).

### Order-state filter (string param)

```sql
-- WORKS:
ГДЕ Док.СостояниеЗаказа.Наименование = &Состояние   -- param "Не обработан"

-- FAILS (0 rows) — comparing enum ref to string:
ГДЕ Док.СостояниеЗаказа = &Состояние
```

`СостоянияЗаказовПокупателей`: В работе / Выставлен счет / Завершен / На выполнении / Не обработан.
**There is no «Отменен».** Filter by non-existent state → «Неверные параметры».

### Register field inventory (verified)

| Register | Available fields |
|----------|------------------|
| `Продажи.Обороты` | КоличествоОборот, СуммаОборот, СуммаНДСОборот, СебестоимостьОборот |
| `Запасы.Остатки` | СтруктурнаяЕдиница, КоличествоОстаток, СуммаОстаток |
| `ЗаказыПокупателей.Остатки` | **only** КоличествоОстаток (**no** Сумма/СуммаОстаток) |
| `ДенежныеСредства` | БанковскийСчетКасса, Валюта, СуммаОстаток (**no** СуммаУпрОстаток) |
| `РасчетыСПокупателями` | Договор, СуммаОстаток |

`РегистрНакопления.ЗаказыПокупателей`: dimensions Организация, ЗаказПокупателя, Номенклатура,
Характеристика, Склад; resource **Количество** (Число 15.3); attribute ДатаОтгрузки.

### Organization filter in virtual tables

```sql
-- WORKS:
ГДЕ (ВЫБОР КОГДА &ЕстьОтборПоОрганизации = ЛОЖЬ ТОГДА ИСТИНА ИНАЧЕ X.Организация = &Организация КОНЕЦ)
-- or
ГДЕ (&ЕстьОтборПоОрганизации = ЛОЖЬ ИЛИ X.Организация = &Организация)

-- FAILS in virtual table filters:
ГДЕ ?(...)   -- ternary not supported
```

### Query syntax gotchas

| Wrong | Right |
|-------|-------|
| `Док.Ссылка.УникальныйИдентификатор()` in query | get GUID on client / via API helper |
| `ВЫБРАТЬ *` on huge tables | explicit fields + `ПЕРВЫЕ N` |
| Date params without quotes/format | ISO → strip `T`, spaces, `-`, `:` → `Дата("yyyymmddhhmmss")` |
| `ПЕРВЫЕ N` + parameters in some contexts | hardcode N for probes |

## 5. Enumerations

Install values in BSL via safe helper `ЗначениеПеречисленияБезопасно("Имя", "ИмяЗначения1,ИмяЗначения2")`
(names without spaces in code: `ОблагаетсяНДС`, `ЗаказНаПродажу`).

Query enums: `ВЫБРАТЬ Ссылка ИЗ Перечисление.X`.

| Enum | Values |
|------|--------|
| `ТипыНалогообложенияНДС` | Облагается (с НДС) / Не облагается (без НДС) / Экспорт (0% НДС) |
| `ВидыОперацийЗаказПокупателя` | Заказ на продажу, Заказ на переработку, Заказ-наряд, Заявка на продажу, Заявка (заказ-наряд) |

**Enum iteration in BSL:** `Менеджер[Индекс]` / `Перечисления.Имя.Значение`.
NEVER `Для Каждого Из Перечисления[Имя]` (does not work as expected).

## 6. Document reference defaults (posted gold sample)

Ethalon posted `ЗаказПокупателя` (НФНФ-000001) header extras:
`НалогообложениеНДС=Облагается (с НДС)`, `ВидОперации=Заказ на продажу`,
`ХозяйственнаяОперация=Заказ на продажу`, `ВидЦен=Розничная цена`,
`СтруктурнаяЕдиницаПродажи=Основное подразделение`, `Валюта=руб.`, Курс/Кратность=1,
`СуммаВключаетНДС=true`, `ПоложениеДатыОтгрузки=В шапке`, `ПоложениеСклада=В шапке`,
`ВидЗаказа=Основной`, `ВариантЗавершения=Успешно`, `УчитыватьВНУ=True`,
`СпециальныйНалоговыйРежим=Не применяется`, `УчетПотребностиПоЗаказам=True`,
`УчетПотребностиПоСкладам=False`, `НДСВключатьВСтоимость=True`.

Line: `ТипНоменклатурыЗапас=true`, `Всего=Сумма`, `КлючСвязи=0`,
`ПроцентСкидкиНаценки=0`, `КоэффициентЕдиницыДляДокументов=0`, `КоличествоДляДокументов=0`.

**ВариантЗавершения** is cleared by `ПередЗаписью` unless state = «Завершен».
Set state «Завершен» → variant «Успешно».

## 7. GET_REPORT_DATA templates

Without `template` → list of templates. With `template` + `period_start` / `period_end` + `organization`:

| Key | Aliases |
|-----|---------|
| `sales` | продажи |
| `stock` | запасы, остатки, balances |
| `cash` | дс, деньги |
| `orders` | заказы |
| `receivables` | взаиморасчеты, дебиторка |
| `preorders` | предзаказы, ожидание |

Unclosed orders = `ЗаказыПокупателей.Остатки` with `КоличествоОстаток > 0`
(+ typically `НЕ ПометкаУдаления`, not «Завершен» depending on report).
**Drafts are intentionally invisible here** (no movements).

## 8. BSL / JSON runtime gotchas (8.5.1.x)

| Topic | Rule |
|-------|------|
| Read JSON | `ЧтениеJSON.УстановитьСтроку` + `ПрочитатьJSON(Чтение)` — NEVER `ПрочитатьJSON(строка)` |
| Write JSON | `ЗаписьJSON.УстановитьСтроку()` + `ЗаписатьJSON` + **`ЗаписьJSON.Закрыть()`** (get string from the closed writer). `ЗакрытьИПолучитьСтроку` may be unavailable / fail in 8.5.1.1529 |
| Boolean filled | **`ЗначениеЗаполнено(Ложь) = Истина`**. Never `НЕ ЗначениеЗаполнено(Булево)` for «flag unset». Check `Если НЕ Объект.Флаг` |
| GUID to JSON | only `Попытка Значение.УникальныйИдентификатор()` — direct call can throw |
| Query result column | use `Колонка.Имя` — **`КолонкаРезультатаЗапроса.Заголовок` does not exist** |
| Date from ISO | drop `T`, spaces, `-`, `:` → `Дата("yyyymmddhhmmss")` |
| Error detail | `ОбработкаОшибок.ПодробноеПредставлениеОшибки(ИнформацияОбОшибке())` **inline** in `Исключение` |

### Posting error classification

| Symptom | Meaning | Fix |
|---------|---------|-----|
| «Деление на 0» / «Период не может быть пустым» | missing business defaults | auto-fill (contract, VAT, dates) |
| `[ОшибкаХранимыхДанных]` | platform constraint on register write | data/contract mismatch; often not fixable by API alone |
| «Документ не найден» | bad ref | verify GUID |

Note: posting a **new** `ЗаказПокупателя` can hit `ОшибкаХранимыхДанных` even with fill —
another reason the project rule is **create draft only**.

## 9. Clients

### Python (preferred for agents / harness)

```python
# File-based script only (no python -c with Cyrillic/quotes in PowerShell)
import json, urllib.request, base64

URL = "http://localhost/unf/hs/ai/api"
AUTH = base64.b64encode(b"obmen:203501").decode()

def call(payload: dict) -> dict:
    body = json.dumps(payload, ensure_ascii=False).encode("utf-8")
    req = urllib.request.Request(URL, data=body, method="POST")
    req.add_header("Content-Type", "application/json; charset=utf-8")
    req.add_header("Authorization", f"Basic {AUTH}")
    with urllib.request.urlopen(req, timeout=60) as resp:
        return json.loads(resp.read().decode("utf-8"))

# Examples
call({"command": "PING"})
call({"command": "EXECUTEQUERY",
      "query": "ВЫБРАТЬ ПЕРВЫЕ 5 Номер, Дата ИЗ Документ.ЗаказПокупателя"})
call({"command": "CREATEDOCUMENT", "type": "ЗаказПокупателя", "data": {...}})
```

**Script convention:** helper probes live in `E:\1C\EDT\Test\_tmp_*.py`.
Never inline `python -c` with Cyrillic or nested quotes (PowerShell parse errors).

### PowerShell

Always encode UTF-8 and prefer `System.Net.HttpWebRequest` in non-interactive hosts.
See skill `1c-http-services` §9–13 for full templates.

## 10. Smoke tests

| File | Purpose |
|------|---------|
| `E:\1C\EDT\Test\test_http_api.py` | core commands |
| `E:\1C\EDT\Test\test_http_api_edge.py` | edge cases |
| `E:\1C\EDT\Test\test_http_document.py` | document CRUD |
| `E:\1C\EDT\Test\test_http_report.py` | GET_REPORT_DATA |

## 11. Agent harness checklist

1. Health: `PING` → `success:true`; else check Apache + 1C + extension.
2. Introspect: `GET_SCHEMA` / `SEARCH` before guessing fields.
3. Read before write: `EXECUTEQUERY` with `ПЕРВЫЕ` to verify names (org, CP, item).
4. Create drafts only; never call `POSTDOCUMENT` unless user explicitly asks to post.
5. Parse envelope: honor `success`, surface `error` text as-is.
6. Multi-step work: one file-based Python script per operation, log request/response JSON.
7. Timeouts: heavy queries fail — split, add `ПЕРВЫЕ`, filter by date.
8. Never invent register fields (e.g. `ЗаказыПокупателей.СуммаОстаток` does not exist).

## 12. Related skills

- `1c-http-services` — writing/maintaining the service itself (MDO, handlers)
- `1c-unf-structure` — UNF metadata map, catalogs/registers
- `1c-edt-mcp` — EDT MCP for code/query validation and deploy
- `1c-coding-standards` — BSL style and runtime pitfalls

## Source

Live surface of extension `unfAiAgent` (commands verified 2026-10-14/15).
Project rule: documents create-only drafts. Auth and paths are environment-specific.
