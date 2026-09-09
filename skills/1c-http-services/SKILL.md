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

| Symptom | Cause |
|---------|-------|
| 404 without auth | Endpoint not registered |
| 401 without auth, 500 with auth | Module crashes during execution |
| 500 on ALL endpoints | Extension has compilation error |
| 200 but wrong data | Check JSON parsing (ЧтениеJSON vs ПрочитатьJSON) |

**Check logs:**
- Apache: `C:\Apache24\logs\error.log`
- 1C infobase: `1Cv8Log/` directory
- EDT: `C:\Users\admin\AppData\Local\Temp\1cedt\`

## 9. Testing with curl/PowerShell

```powershell
# Ping test
$cred = [System.Convert]::ToBase64String([System.Text.Encoding]::UTF8.GetBytes("user:pass"))
Invoke-WebRequest -Uri "http://host/hs/test/ping/" -Headers @{Authorization="Basic $cred"}

# POST with JSON body
$body = '{"Команда":"EXECUTEQUERY","ТекстЗапроса":"ВЫБРАТЬ 1 КАК Число"}'
Invoke-WebRequest -Uri "http://host/hs/ai/api/" -Method POST `
    -Body ([System.Text.Encoding]::UTF8.GetBytes($body)) `
    -ContentType "application/json;charset=utf-8" `
    -Headers @{Authorization="Basic $cred"}
```
