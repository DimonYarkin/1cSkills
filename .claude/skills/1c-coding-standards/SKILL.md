---
name: 1c-coding-standards
description: >
  1С:Предприятие 8.3/8.5 coding standards extracted from BSP (БСП) library in UNF.
  Use when writing, reviewing, or refactoring 1C:Enterprise code in any configuration.
  Covers module structure, naming, documentation, error handling, regions, queries,
  and object module patterns. Source: СтандартныеПодсистемы library (1C-Soft, CC BY 4.0).
---

# 1С:Предприятие Coding Standards (BSP-derived)

## 1. Module Structure

Every module MUST follow this region hierarchy:

```
#Область ПрограммныйИнтерфейс          ← exported functions/procedures
  #Область <SubsectionName>             ← logical grouping
  #КонецОбласти
#КонецОбласти

#Область СлужебныйПрограммныйИнтерфейс  ← exported for other modules, not for external callers

#Область ОбработчикиСобытий             ← event handlers (ПриЗаписи, ПередЗаписью, etc.)

#Область СлужебныеПроцедурыИФункции    ← private implementation

#Область Инициализация                  ← module-level variables init (if needed)
```

**Object modules** additionally have:
```
#Область ОписаниеПеременных
  Перем ЭтоНовый;
  Перем ЕдиницаИзмеренияПередЗаписью;
#КонецОбласти

#Область ОбработчикиСобытий
  Процедура ОбработкаПроверкиЗаполнения(Отказ, ПроверяемыеРеквизиты)
  ...
#КонецОбласти
```

## 2. Naming Conventions

### Functions & Procedures
- **Russian names**, descriptive, verb-first for procedures: `СформироватьПлатежнуюСсылку()`, `РассчитатьАвтоматическиеСкидки()`
- **Predicate functions** start with `Это...`: `ЭтоЗаказНаряд()`, `ЭтоСсылкаGUID()`
- **Getter functions**: `ПолучитьФункциональнуюОпцию()`, `ПолучитьШапкуHTML()`
- **Prefixes by purpose**:
  - `Добавить...` — add to collection: `ДобавитьКомандыЗаполнения()`
  - `Установить...` — set state: `УстановитьПривилегированныйРежим()`
  - `Получить...` — get/retrieve: `ПолучитьЗначениеРеквизита()`
  - `Сформировать...` — generate/create: `СформироватьОтчёт()`
  - `Заполнить...` — fill/populate: `ЗаполнитьСтруктуруДанных()`
  - `Найти...` — search: `НайтиОрганизацию()`, `НайтиКонтрагента()`
  - `Проверить...` — validate: `ПроверитьЗаполнение()`

### Parameters
- Named in Russian, descriptive: `Знач ТекстСообщенияПользователю`, `Знач КлючДанных = Неопределено`
- Use `Знач` for input-only parameters
- Default values use `Неопределено` (not empty) when optional

### Variables
- Russian names: `СтруктураПараметры`, `ТаблицаЗапасы`, `ДанныеОтвета`
- Temporary/loop: short names OK: `НВарианта`, `Индекс`, `Строка`

## 3. Documentation Comments

Every exported function/procedure MUST have:

```1c
// Краткое описание назначения.
//
// Параметры:
//  ИмяПараметра - Тип - описание параметра.
//
// Возвращаемое значение:
//  Тип - описание результата.
//
// Пример:
//  1. Первый пример использования.
//  2. Второй пример использования.
```

**Key rules:**
- `//` comment style (NOT `///`)
- Section separators: `// Параметры:`, `// Возвращаемое значение:`, `// Пример:`
- Document **both** parameters and return values
- Include at least 1 usage example for complex functions
- For simple self-explanatory functions, a single-line comment suffices:

```1c
// Определяет по виду операции, что заказ является Заказ-нарядом.
Функция ЭтоЗаказНаряд() Экспорт
```

## 4. Error Handling Pattern

Always use structured error handling:

```1c
Процедура ВыполнитьОперацию() Экспорт

    Попытка
        // ... основная логика ...
    Исключение
        ИнформацияОбОшибке = ИнформацияОбОшибке();
        ТекстОшибки = ОбработкаОшибок.ПодробноеПредставлениеОшибки(ИнформацияОбОшибке);
        ЗаписьЖурналаРегистрации(
            НСтр("ru='Операция'", ОбщегоНазначения.КодОсновногоЯзыка()),
            УровеньЖурналаРегистрации.Ошибка,
            Метаданные.Документы.МойДокумент, ,
            ТекстОшибки);
    КонецПопытки;

КонецПроцедуры
```

**Key rules:**
- Store `ИнформацияОбОшибке()` in a variable first (platform requirement)
- Use `ОбработкаОшибок.ПодробноеПредставлениеОшибки()` for detailed error text
- Always log to `ЗаписьЖурналаРегистрации` with:
  - `НСтр("ru='...'")` for event name
  - `ОбщегоНазначения.КодОсновногоЯзыка()` for language code
  - `Метаданные.<MetadataObject>` for metadata reference
- For user-facing errors: `ОбщегоНазначения.СообщитьПользователю()`

## 5. Query Writing Standards

```1c
Запрос = Новый Запрос;
Запрос.УстановитьПараметр("МассивДокументов", МассивДокументов);
Запрос.Текст =
"ВЫБРАТЬ РАЗЛИЧНЫЕ
|    Документ.Ссылка КАК Ссылка,
|    ИСТИНА КАК ЕстьНДС
|ПОМЕСТИТЬ ВТ_ДокументыСНДС
|ИЗ
|    Документ.МойДокумент.ТабличнаяЧасть КАК Документ
|ГДЕ
|    Документ.Ссылка В (&МассивДокументов)
|    И Документ.Сумма > 0
|
|ИНДЕКСИРОВАТЬ ПО
|    Ссылка
|;
|
|//////////////////////////////////////////////////////////////////////////////////
|ВЫБРАТЬ
|    Документ.Ссылка КАК Ссылка
|ИЗ
|    ВТ_ДокументыСНДС КАК Документ";

Результат = Запрос.Выполнить();
```

**Key rules:**
- Always use `КАК` aliases for all columns
- Use `УНИКАЛЬНЫЙИДЕНТИФИКАТОР(Ссылка) КАК GUID` for GUIDs
- Use `ПОМЕСТИТЬ ВТ_` for temporary tables
- Separate query sections with `//////////////////////////////////////////////////////////////////////////////////`
- Use `Запрос.УстановитьПараметр()` — NEVER concatenate strings
- Column order: `Ссылка` first, then business fields, then computed fields

## 6. Module Preprocessor

```1c
#Если Сервер Или ТолстыйКлиентОбычноеПриложение Или ВнешнееСоединение Тогда
// server-side code only
#КонецЕсли
```

Always start server modules with this guard.

## 7. Procedure/Function Signatures

```1c
// Simple function
Функция ЭтоЗаказНаряд() Экспорт
    Возврат ВидОперации = Перечисления.ВидыОперацийЗаказПокупателя.ЗаказНаряд;
КонецФункции

// Function with parameters and defaults
Функция ПолучитьЗначениеРеквизита(Ссылка, ИмяРеквизита, Знач ЗначениеПоУмолчанию = Неопределено) Экспорт

// Procedure with output parameter
Процедура СообщитьПользователю(Знач ТекстСообщенияПользователю,
    Знач КлючДанных = Неопределено, Знач Поле = "",
    Знач ПутьКДанным = "", Отказ = Ложь) Экспорт
```

**Rules:**
- One `Экспорт` per function/procedure (NOT on separate line)
- `Знач` for input-only parameters
- Align continuation parameters with opening parenthesis
- Return type documentation: `// Возвращаемое значение: Тип - описание`

## 8. Event Handler Patterns

```1c
// Document: Before write
Процедура ПередЗаписью(Отказ, РежимЗаписи, РежимПроведения)
    Если ОбменДанными.Загрузка Тогда
        Возврат;
    КонецЕсли;
    // ... logic ...
КонецПроцедуры

// Document: Fill check
Процедура ОбработкаПроверкиЗаполнения(Отказ, ПроверяемыеРеквизиты)
    Если ЭтоГруппа Тогда
        Возврат;
    КонецЕсли;
    // ... validation logic using ПроверяемыеРеквизиты.Добавить("ИмяРеквизита")
КонецПроцедуры

// ManagerModule: Access restriction
Процедура ПриЗаполненииОграниченияДоступа(Ограничение) Экспорт
    Ограничение.Текст =
    "РазрешитьЧтениеИзменение
    |ГДЕ
    |    ЗначениеРазрешено(Организация)";
КонецПроцедуры
```

## 9. JSON Handling (1C 8.5+)

```1c
// Reading JSON — ALWAYS use ЧтениеJSON, NOT ПрочитатьJSON(строка)
Чтение = Новый ЧтениеJSON;
Чтение.УстановитьСтроку(ТелоЗапроса);
Данные = ПрочитатьJSON(Чтение);
Чтение.Закрыть();

// Writing JSON — use ЗаписьJSON
ЗаписьJSON = Новый ЗаписьJSON;
ЗаписьJSON.УстановитьСтроку();
ЗаписатьJSON(ЗаписьJSON, СтруктураДанных);
Результат = ЗаписьJSON.ЗакрытьИПолучитьСтроку();
```

**CRITICAL:** `ПрочитатьJSON(строка)` causes runtime errors in 1C 8.5. Always use `ЧтениеJSON` object.

## 10. HTTP Service Patterns

```1c
// Handler function — named by MDO <handler> value
Функция <handlerName>(Запрос)

    УстановитьПривилегированныйРежим(Истина);

    Попытка
        ТелоЗапроса = Запрос.ПолучитьТелоКакСтроку();

        // Parse JSON
        Чтение = Новый ЧтениеJSON;
        Чтение.УстановитьСтроку(ТелоЗапроса);
        Данные = ПрочитатьJSON(Чтение);
        Чтение.Закрыть();

        // Process and return
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

## 11. Extension Coding Rules

- ALL custom functions MUST have project prefix (e.g., `ai_`, `my_`)
- NEVER use names that conflict with platform or base configuration
- `containedObjects` in Configuration.mdo are REQUIRED for Adopted extensions
- Extension HTTP services use `<template>/path</template>` + `<handler>` MDO pattern

## 12. Localization

```1c
// Always use НСтр for user-facing strings
НСтр("ru = 'Сообщение об ошибке.'")

// For event log entries — include language code
НСтр("ru='Формирование платежной ссылки'", ОбщегоНазначения.КодОсновногоЯзыка())
```

## 13. Common Anti-Patterns to Avoid

- DO NOT use `ПрочитатьJSON(строка)` — use `ЧтениеJSON`
- DO NOT use `ЗаписьJSON.ОткрытьПоток()` — use `УстановитьСтроку()`
- DO NOT hardcode strings — use `НСтр("ru='...'")` 
- DO NOT skip error handling in exported procedures
- DO NOT use `Сообщить()` directly — use `ОбщегоНазначения.СообщитьПользователю()`
- DO NOT forget `Знач` on input-only parameters
- DO NOT create functions without region structure

## 14. Copyright Header (MANDATORY)

Every module file starts with copyright block (use your own for custom extensions):

```1c
///////////////////////////////////////////////////////////////////////////////////////////////////////
// Copyright (c) 2025, ООО 1С-Софт
// Все права защищены.
///////////////////////////////////////////////////////////////////////////////////////////////////////
```

## 15. Module Naming Convention (Role Suffixes)

| Suffix | Purpose | Example |
|--------|---------|---------|
| (none) | Core server module | `ОбщегоНазначения` |
| `Клиент` | Client-side only | `ОбщегоНазначенияКлиент` |
| `Сервер` | Server-side only | `СтандартныеПодсистемыСервер` |
| `КлиентСервер` | Both sides | `ОбщегоНазначенияКлиентСервер` |
| `ВызовСервера` | Server call from client | `ОбщегоНазначенияВызовСервера` |
| `ПовтИсп` | Reusable (cached) | `СтандартныеПодсистемыПовтИсп` |
| `Служебный` | Internal implementation | `ОбщегоНазначенияСлужебныйКлиентСервер` |
| `Переопределяемый` | Overridable hooks | `ОбщегоНазначенияПереопределяемый` |
| `Глобальный` | Global module | `ОбщегоНазначенияГлобальный` |
| `Локализация` | Localization strings | `СтроковыеФункцииКлиентСерверЛокализация` |

## 16. Delegation Pattern

Public API functions are thin facades delegating to Служебный modules:

```1c
Функция АвторизованныйПользователь() Экспорт
    Возврат ПользователиСлужебный.АвторизованныйПользователь();
КонецФункции
```

## 17. Conditional Subsystem Calls

For optional dependencies, check subsystem existence dynamically:

```1c
Если ОбщегоНазначения.ПодсистемаСуществует("СтандартныеПодсистемы.Мультиязычность") Тогда
    Модуль = ОбщегоНазначения.ОбщийМодуль("МультиязычностьСервер");
    Модуль.УстановкаПараметровСеанса(ИменаПараметровСеанса, УстановленныеПараметры);
КонецЕсли;
```

## 18. АПК Directives (Code Analysis Suppressions)

```1c
// АПК:142-выкл ... АПК:142-вкл  — suppress "too many parameters"
// АПК:488                       — dynamic code execution is verified safe
// //@skip-check doc-comment-field-type-strict
// //@skip-check doc-comment-collection-item-type
```

## 19. Deprecated Functions Region

```1c
#Область УстаревшиеПроцедурыИФункции
// Устарела. Следует использовать НоваяФункция.
Функция СтараяФункция(Параметр) Экспорт
    Возврат НоваяФункция(Параметр);
КонецФункции
#КонецОбласти
```

## 20. Object Module Patterns (Document/Catalog)

### ПередЗаписью Guard (ALWAYS first):
```1c
Процедура ПередЗаписью(Отказ, РежимЗаписи, РежимПроведения)
    Если ОбменДанными.Загрузка Тогда
        Возврат;
    КонецЕсли;
    // ... actual logic ...
КонецПроцедуры
```

### ОбработкаЗаполнения Dispatch:
```1c
Процедура ОбработкаЗаполнения(ДанныеЗаполнения, СтандартнаяОбработка)
    Если ТипЗнч(ДанныеЗаполнения) = Тип("СправочникСсылка.ДоговорыКонтрагентов") Тогда
        // ... specific fill logic
    Иначе
        ЗаполнениеОбъектовУНФ.ЗаполнитьДокумент(ЭтотОбъект, ДанныеЗаполнения);
    КонецЕсли;
КонецПроцедуры
```

### ОбработкаПроведения Standard Pattern:
```1c
Процедура ОбработкаПроведения(Отказ, РежимПроведения)
    ПроведениеДокументовУНФ.ИнициализироватьДополнительныеСвойстваДляПроведения(Ссылка, ДополнительныеСвойства);
    Документы.МойДокумент.ИнициализироватьДанныеДокумента(Ссылка, ДополнительныеСвойства);
    ПроведениеДокументовУНФ.ПодготовитьНаборыЗаписейКРегистрацииДвижений(ЭтотОбъект);
    // ... register movements ...
    ПроведениеДокументовУНФ.ЗакрытьМенеджерВременныхТаблиц(ЭтотОбъект);
КонецПроцедуры
```

### Transactional Error Handling:
```1c
НачатьТранзакцию();
Попытка
    БлокировкаДанных = Новый БлокировкаДанных;
    ЭлементБлокировки = БлокировкаДанных.Добавить("РегистрСведений.ЦеныНоменклатуры");
    ЭлементБлокировки.Режим = РежимБлокировкиДанных.Исключительный;
    БлокировкаДанных.Заблокировать();
    // ... business logic ...
    ЗафиксироватьТранзакцию();
Исключение
    ОтменитьТранзакцию();
    ЗаписьЖурналаРегистрации(
        НСтр("ru = 'Ошибка...'", ОбщегоНазначения.КодОсновногоЯзыка()),
        УровеньЖурналаРегистрации.Ошибка, , ,
        ОбработкаОшибок.ПодробноеПредставлениеОшибки(ИнформацияОбОшибке()));
    Отказ = Истина;
КонецПопытки;
```

### Data Locking Pattern:
```1c
БлокировкаДанных = Новый БлокировкаДанных;
ЭлементБлокировки = БлокировкаДанных.Добавить("РегистрСведений.ЦеныНоменклатуры");
ЭлементБлокировки.Режим = РежимБлокировкиДанных.Исключительный;
ЭлементБлокировки.ИсточникДанных = ТаблицаИсточник;
ЭлементБлокировки.ИспользоватьИзИсточникаДанных("Номенклатура", "Номенклатура");
ЭлементБлокировки.УстановитьЗначение("ВидЦен", ВидЦен);
БлокировкаДанных.Заблокировать();
```

### User Message with Context Field:
```1c
КонтекстноеПоле = ОбщегоНазначенияКлиентСервер.ПутьКТабличнойЧасти(
    "Запасы", СтрокаЗапасы.НомерСтроки, "Резерв");
ОбщегоНазначения.СообщитьПользователю(ТекстСообщения, ЭтотОбъект, КонтекстноеПоле, , Отказ);
```

## 21. Manager Module Standard Exports

Every ManagerModule exports these subsystem hooks:

| Procedure | Purpose |
|---|---|
| `ПриЗаполненииОграниченияДоступа(Ограничение)` | RBAC access rules |
| `ДобавитьКомандыЗаполнения(КомандыЗаполнения, Параметры)` | Fill commands |
| `ДобавитьКомандыСозданияНаОсновании(...)` | "Create on basis" commands |
| `ПриОпределенииНастроекВерсионированияОбъектов(Настройки)` | Versioning |
| `ПриОпределенииМетодовРазрешенныхДляВызоваКакПроизвольныйКод(Методы)` | Code permission |
| `ПоляЗагрузкиДанныхИзВнешнегоИсточника(...)` | External data import |

## Source

Extracted from UNF (1С:Управление Нашей Фирмой) BSP library modules:
- `ОбщегоНазначения` — core utility functions
- `СтроковыеФункции` — string operations
- `ЗаказПокупателя/ObjectModule` — document object patterns
- `ЗаказПокупателя/ManagerModule` — manager module patterns
- `Номенклатура/ObjectModule` — catalog object patterns

License: CC BY 4.0 (1C-Soft, 2025)
