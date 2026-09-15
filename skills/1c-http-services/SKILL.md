---
name: 1c-http-services
description: >
  1С:Предприятие 8.5 HTTP services development guide.
  Use when creating, debugging, or testing HTTP services in 1C extensions.
  Covers MDO structure, handler patterns, JSON I/O, debugging, and deployment.
---

# 1C HTTP Services Development (1C 8.5)

## 1. MDO Structure (Extension HTTP Service)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<mdclass:HTTPService xmlns:mdclass="http://g5.1c.ru/v8/dt/metadata/mdclass" uuid="...">
  <name>ServiceName</name>
  <synonym><key>ru</key><value>Description</value></synonym>
  <rootURL>serviceroot</rootURL>
  <reuseSessions>AutoUse</reuseSessions>
  <sessionMaxAge>20</sessionMaxAge>
  <urlTemplates uuid="...">
    <name>endpoint</name>
    <template>/endpoint</template>
    <methods uuid="...">
      <name>POST</name>
      <httpMethod>POST</httpMethod>
      <handler>endpointPOST</handler>
    </methods>
  </urlTemplates>
</mdclass:HTTPService>
```

**URL pattern:** `/hs/<rootURL>/<template>/`
**Handler function:** MUST match `<handler>` value exactly.

## 2. Handler Function Pattern

```1c
// Handler function — name matches MDO <handler> value
Функция <handlerName>(Запрос)

    УстановитьПривилегированныйРежим(Истина);

    Попытка
        ТелоЗапроса = Запрос.ПолучитьТелоКакСтроку();

        // Parse JSON using ЧтениеJSON (NOT ПрочитатьJSON!)
        Чтение = Новый ЧтениеJSON;
        Чтение.УстановитьСтроку(ТелоЗапроса);
        Данные = ПрочитатьJSON(Чтение);
        Чтение.Закрыть();

        // Process...

        // Return JSON response
        Ответ = Новый HTTPСервисОтвет(200);
        Ответ.Заголовки.Вставить("Content-Type", "application/json;charset=utf-8");
        Ответ.УстановитьТелоИзСтроки(ТелоОтвета, КодировкаТекста.UTF8);
        Возврат Ответ;

    Исключение
        Ответ = Новый HTTPСервисОтвет(500);
        Ответ.Заголовки.Вставить("Content-Type", "application/json;charset=utf-8");
        Ответ.УстановитьТелоИзСтроки("error", КодировкаТекста.UTF8);
        Возврат Ответ;
    КонецПопытки;

КонецФункции
```

## 3. JSON I/O (CRITICAL for 1C 8.5)

### Reading JSON — ALWAYS use ЧтениеJSON:
```1c
// CORRECT:
Чтение = Новый ЧтениеJSON;
Чтение.УстановитьСтроку(ТелоЗапроса);
Данные = ПрочитатьJSON(Чтение);
Чтение.Закрыть();

// WRONG — causes 500 error:
Данные = ПрочитатьJSON(ТелоЗапроса);  // NEVER!
```

### Writing JSON — use ЗаписьJSON:
```1c
ЗаписьJSON = Новый ЗаписьJSON;
ЗаписьJSON.УстановитьСтроку();
ЗаписатьJSON(ЗаписьJSON, СтруктураДанных);
Результат = ЗаписьJSON.ЗакрытьИПолучитьСтроку();

// WRONG — fails in HTTP service context:
ЗаписьJSON.ОткрытьПоток();  // NEVER!
```

## 4. Naming Conventions

- ALL custom functions MUST have project prefix (e.g., `ai_`)
- Handler functions are named by MDO `<handler>` value — no prefix needed
- Convention: `<template><HTTPMethod>` — e.g., `apiPOST`, `pingGET`

## 5. Response Pattern

```1c
// Success
Функция ai_ОтветУспех(Данные)
    Ответ = Новый HTTPСервисОтвет(200);
    Ответ.Заголовки.Вставить("Content-Type", "application/json;charset=utf-8");
    Ответ.УстановитьТелоИзСтроки(ai_ЗаписатьJSONВСтроку(Данные), КодировкаТекста.UTF8);
    Возврат Ответ;
КонецФункции

// Error
Функция ai_ОтветОшибка(ТекстОшибки)
    Ответ = Новый HTTPСервисОтвет(400);
    Ответ.Заголовки.Вставить("Content-Type", "application/json;charset=utf-8");
    Ответ.УстановитьТелоИзСтроки(ai_ЗаписатьJSONВСтроку(
        Новый Структура("success, error", Ложь, ТекстОшибки)),
        КодировкаТекста.UTF8);
    Возврат Ответ;
КонецФункции
```

## 6. Common Query Pattern via API

```json
{
    "Команда": "EXECUTEQUERY",
    "ТекстЗапроса": "ВЫБРАТЬ ПЕРВЫЕ 10 Таблица.Поле КАК Алиас ИЗ Справочник.Имя КАК Таблица"
}
```

## 7. Extension Configuration Rules

- `containedObjects` in Configuration.mdo are REQUIRED — removing causes compilation error
- Subsystem `<content>` must not reference deleted objects
- Use `<externalConnection>true</externalConnection>` in common module MDO
- VRD `publishExtensionsByDefault="true"` auto-publishes extension HTTP services

## 8. Debugging HTTP Services

| Symptom | Cause | Fix |
|---------|-------|-----|
| 404 without auth | Endpoint not registered | Check VRD + extension applied |
| 401 without auth, 500 with auth | Module crashes | Check `1Cv8Log/` |
| 500 on ALL endpoints | Extension compilation error | Check EDT `get_project_errors` |
| 405 Method Not Allowed | Wrong HTTP method | Use POST for EXECUTEQUERY |
| Timeout on query | Too complex, heavy joins | Split into simpler queries (Rule 1-2) |
| 400 Bad Request | JSON encoding issue | Use `[System.Text.Encoding]::UTF8.GetBytes()` |

**Check logs:**
- Apache: `C:\Apache24\logs\error.log` and `access.log`
- 1C infobase: `E:\Байер\Новая база\1Cv8Log\`
- EDT: `C:\Users\admin\AppData\Local\Temp\1cedt\`

## 9. Testing with PowerShell

**IMPORTANT:** PowerShell in non-interactive mode throws `Invoke-WebRequest` errors about Prompt. Use `System.Net.HttpWebRequest` instead:

```powershell
[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.SecurityProtocolType]::Tls12
$cred = [Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes("obmen:203501"))
$json = [System.Text.Encoding]::UTF8.GetBytes('{"Команда":"EXECUTEQUERY","ТекстЗапроса":"ВЫБРАТЬ 1 КАК Тест"}')

$req = [System.Net.HttpWebRequest]::Create("http://localhost/unf/ru/hs/ai/api")
$req.Method = "POST"
$req.ContentType = "application/json; charset=utf-8"
$req.Timeout = 30000
$req.ContentLength = $json.Length
$req.Headers.Add("Authorization", "Basic $cred")
$s = $req.GetRequestStream()
$s.Write($json, 0, $json.Length)
$s.Close()
$resp = $req.GetResponse()
$r = New-Object System.IO.StreamReader($resp.GetResponseStream())
$result = $r.ReadToEnd()
$r.Close()
$resp.Close()
$result
```

**Error handling pattern:**
```powershell
try { ... } catch {
    $code = if ($_.Exception.Response) {
        $e = $_.Exception.Response
        $sr = New-Object System.IO.StreamReader($e.GetResponseStream())
        $errBody = $sr.ReadToEnd(); $sr.Close()
        "$([int]$e.StatusCode): $errBody"
    } else { "timeout" }
    "ERR ($code)"
}
```

## 10. AI_Agent Service — Known Configuration

**UNF deployment:**
- URL: `http://localhost/unf/ru/hs/ai/api`
- Method: POST
- Content-Type: `application/json; charset=utf-8`
- Auth: Basic `obmen:203501`
- VRD: `C:\www\unf\default.vrd` with `publishExtensionsByDefault="true"`

**Request format:**
```json
{"Команда":"EXECUTEQUERY","ТекстЗапроса":"ВЫБРАТЬ ..."}
```

**Response:** JSON array of objects, one per result row.

## 11. Query Optimization Rules

### Rule 1: Start Simple, Add Complexity
```sql
-- STEP 1: count first
ВЫБРАТЬ КОЛИЧЕСТВО(*) ИЗ Документ.ЗаказПокупателя

-- STEP 2: basic fields (no joins)
ВЫБРАТЬ ПЕРВЫЕ 5 Ссылка, Дата, Номер ИЗ Документ.ЗаказПокупателя

-- STEP 3: add joins one by one
ВЫБРАТЬ Дата, Номер, Контрагент.Наименование КАК Контрагент ИЗ Документ.ЗаказПокупателя
```

### Rule 2: Avoid Heavy Joins in One Query
```sql
-- BAD — times out on large tables:
ВЫБРАТЬ Дата, Номер, Контрагент.Наименование, СостояниеЗаказа.Наименование,
        СуммаДокумента ИЗ Документ.ЗаказПокупателя

-- GOOD — split into separate queries:
-- Query 1: basic data
ВЫБРАТЬ Дата, Номер, СуммаДокумента, СостояниеЗаказа.Наименование ИЗ Документ.ЗаказПокупателя
-- Query 2: customer names
ВЫБРАТЬ Номер, Контрагент.Наименование ИЗ Документ.ЗаказПокупателя
```

### Rule 3: Use ПЕРВЫЕ for Exploration
```sql
ВЫБРАТЬ ПЕРВЫЕ 10 Номенклатура.Наименование КАК Товар, Количество, Цена
ИЗ Документ.РасходнаяНакладная.Запасы
```

### Rule 4: Boolean Filters
```sql
-- CORRECT:
ГДЕ Проведен            -- Boolean field, no = needed
ГДЕ НЕ Проведен

-- WRONG:
ГДЕ Проведен = ИСТИНА   -- Works but verbose
```

### Rule 5: Count Before Fetching
```sql
-- Always check count first to avoid timeouts:
ВЫБРАТЬ КОЛИЧЕСТВО(*) КАК Колво ИЗ Документ.ЗаказПокупателя
```

### Rule 6: Common UNF Queries

**Sales (Расходные накладные):**
```sql
ВЫБРАТЬ Дата, Номер, Контрагент.Наименование КАК Контрагент,
        СуммаДокумента КАК Сумма
ИЗ Документ.РасходнаяНакладная ГДЕ Проведен
```

**Sales line items:**
```sql
ВЫБРАТЬ Номенклатура.Наименование КАК Товар, Количество, Цена,
        Сумма, СтавкаНДС, СуммаНДС, Всего
ИЗ Документ.РасходнаяНакладная.Запасы ГДЕ Ссылка.Проведен
```

**Customer orders (Заказы покупателей):**
```sql
ВЫБРАТЬ Дата, Номер, СуммаДокумента КАК Сумма,
        СостояниеЗаказа.Наименование КАК Состояние
ИЗ Документ.ЗаказПокупателя
```

**Organizations:**
```sql
ВЫБРАТЬ Наименование ИЗ Справочник.Организации
```

**Counters:**
```sql
ВЫБРАТЬ КОЛИЧЕСТВО(*) КАК Колво ИЗ Документ.РасходнаяНакладная ГДЕ Проведен
```
