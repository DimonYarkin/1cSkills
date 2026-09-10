---
name: 1c-forms
description: >
  1С:Предприятие 8.3/8.5 forms development guide for managed forms.
  Use when creating, editing, or debugging forms in 1C configurations and extensions.
  Covers Form.form XML structure, attributes, items, commands, handlers, report forms,
  and common pitfalls discovered during UNF development.
---

# 1C Managed Forms Development Guide

## 1. Form.form XML Structure (Critical Rules)

### Rule 1: Items MUST be at TOP LEVEL
```xml
<!-- CORRECT — items at top level -->
<form:Form xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:form="http://g5.1c.ru/v8/dt/form">
  <items xsi:type="form:FormField">
    <name>ПолеВыбора</name>
    <id>1</id>
    ...
  </items>
  <items xsi:type="form:Button">
    <name>Кнопка</name>
    <id>2</id>
    ...
  </items>
</form:Form>

<!-- WRONG — items wrapped in FormGroup (elements won't display!) -->
<form:Form>
  <items xsi:type="form:FormGroup">
    <name>Группа</name>
    <items xsi:type="form:FormField">...</items>  <!-- NOT RENDERED -->
  </items>
</form:Form>
```

**EXCEPTION:** `form:FormGroup` can contain items, but top-level elements should NOT be nested.

### Rule 2: SpreadsheetDocumentField uses SpreadSheetDocFieldExtInfo (capital S)
```xml
<items xsi:type="form:FormField">
  <name>ДокументРезультат</name>
  <id>1</id>
  <dataPath xsi:type="form:DataPath">
    <segments>РезультатОтчета</segments>
  </dataPath>
  <type>SpreadsheetDocumentField</type>
  <extInfo xsi:type="form:SpreadSheetDocFieldExtInfo">  <!-- Note: SpreadSheet, not Spreadsheet -->
    <width>120</width>
    <autoMaxWidth>true</autoMaxWidth>
    <height>40</height>
    <autoMaxHeight>true</autoMaxHeight>
    <horizontalStretch>true</horizontalStretch>
    <verticalStretch>true</verticalStretch>
    <verticalScrollBar>ScrollAlways</verticalScrollBar>
    <horizontalScrollBar>ScrollAlways</horizontalScrollBar>
    <selectionShowMode>Always</selectionShowMode>
  </extInfo>
</items>
```

### Rule 3: InputField uses InputFieldExtInfo
```xml
<items xsi:type="form:FormField">
  <name>ПериодПоле</name>
  <id>2</id>
  <dataPath xsi:type="form:DataPath">
    <segments>ПериодОтчета</segments>
  </dataPath>
  <type>InputField</type>
  <extInfo xsi:type="form:InputFieldExtInfo">
    <autoMaxWidth>true</autoMaxWidth>
    <autoMaxHeight>true</autoMaxHeight>
    <choiceButton>true</choiceButton>
  </extInfo>
</items>
```

### Rule 4: Button uses commandName
```xml
<items xsi:type="form:Button">
  <name>СформироватьКнопка</name>
  <id>3</id>
  <type>UsualButton</type>
  <commandName>Form.Command.Сформировать</commandName>  <!-- Must match formCommands name -->
  <buttonImportance>VeryImportant</buttonImportance>
  <representation>TextAndPicture</representation>
</items>
```

## 2. dataPath Format (Critical)

```xml
<!-- CORRECT — with xsi:type -->
<dataPath xsi:type="form:DataPath">
  <segments>АтрибутФормы</segments>
</dataPath>

<!-- WRONG — plain string -->
<dataPath>АтрибутФормы</dataPath>  <!-- WON'T WORK -->
```

## 3. Required Form Properties

```xml
<form:Form>
  <!-- These are required for the form to work properly -->
  <handlers>
    <event>OnCreateAtServer</event>
    <name>ПриСозданииНаСервере</name>
  </handlers>
  <windowOpeningMode>DontBlock</windowOpeningMode>
  <saveWindowSettings>true</saveWindowSettings>
  <autoTitle>true</autoTitle>
  <autoUrl>true</autoUrl>
  <group>Vertical</group>
  <autoFillCheck>true</autoFillCheck>
  <allowFormCustomize>true</allowFormCustomize>
  <enabled>true</enabled>
  <showTitle>auto</showTitle>
  <showCloseButton>true</showCloseButton>
</form:Form>
```

## 4. Report Form MDO — Required defaultForm

```xml
<mdclass:Report ...>
  <name>МойОтчет</name>
  <defaultForm>Report.МойОтчет.Form.ФормаОтчета</defaultForm>  <!-- REQUIRED! -->
  <forms uuid="...">
    <name>ФормаОтчета</name>
  </forms>
  <templates uuid="...">
    <name>Основной</name>
  </templates>
</mdclass:Report>
```

**Without `<defaultForm>` the report won't open its form!**

## 5. Complete Working Form Example

```xml
<?xml version="1.0" encoding="UTF-8"?>
<form:Form xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xmlns:form="http://g5.1c.ru/v8/dt/form">

  <!-- 1. Input field for period -->
  <items xsi:type="form:FormField">
    <name>ПериодОтчетаПоле</name>
    <id>1</id>
    <visible>true</visible>
    <enabled>true</enabled>
    <userVisible><common>true</common></userVisible>
    <dataPath xsi:type="form:DataPath">
      <segments>ПериодОтчета</segments>
    </dataPath>
    <title>Период</title>
    <titleLocation>Left</titleLocation>
    <type>InputField</type>
    <editMode>Auto</editMode>
    <showInHeader>true</showInHeader>
    <extInfo xsi:type="form:InputFieldExtInfo">
      <autoMaxWidth>true</autoMaxWidth>
      <autoMaxHeight>true</autoMaxHeight>
      <choiceButton>true</choiceButton>
    </extInfo>
  </items>

  <!-- 2. Buttons -->
  <items xsi:type="form:Button">
    <name>СформироватьКнопка</name>
    <id>2</id>
    <visible>true</visible>
    <enabled>true</enabled>
    <userVisible><common>true</common></userVisible>
    <type>UsualButton</type>
    <commandName>Form.Command.Сформировать</commandName>
    <buttonImportance>VeryImportant</buttonImportance>
    <representation>TextAndPicture</representation>
  </items>

  <!-- 3. Spreadsheet document -->
  <items xsi:type="form:FormField">
    <name>РезультатОтчетаПоле</name>
    <id>3</id>
    <visible>true</visible>
    <enabled>true</enabled>
    <userVisible><common>true</common></userVisible>
    <dataPath xsi:type="form:DataPath">
      <segments>РезультатОтчета</segments>
    </dataPath>
    <titleLocation>None</titleLocation>
    <type>SpreadsheetDocumentField</type>
    <editMode>Auto</editMode>
    <showInHeader>true</showInHeader>
    <extInfo xsi:type="form:SpreadSheetDocFieldExtInfo">
      <width>120</width>
      <autoMaxWidth>true</autoMaxWidth>
      <height>40</height>
      <autoMaxHeight>true</autoMaxHeight>
      <horizontalStretch>true</horizontalStretch>
      <verticalStretch>true</verticalStretch>
      <verticalScrollBar>ScrollAlways</verticalScrollBar>
      <horizontalScrollBar>ScrollAlways</horizontalScrollBar>
      <selectionShowMode>Always</selectionShowMode>
    </extInfo>
  </items>

  <!-- Command bar -->
  <autoCommandBar>
    <name>ФормаКоманднаяПанель</name>
    <id>-1</id>
    <horizontalAlign>Left</horizontalAlign>
  </autoCommandBar>

  <!-- Event handlers -->
  <handlers>
    <event>OnCreateAtServer</event>
    <name>ПриСозданииНаСервере</name>
  </handlers>

  <!-- Form settings -->
  <windowOpeningMode>DontBlock</windowOpeningMode>
  <saveWindowSettings>true</saveWindowSettings>
  <autoTitle>true</autoTitle>
  <autoUrl>true</autoUrl>
  <group>Vertical</group>
  <autoFillCheck>true</autoFillCheck>
  <allowFormCustomize>true</allowFormCustomize>
  <enabled>true</enabled>
  <showTitle>auto</showTitle>
  <showCloseButton>true</showCloseButton>

  <!-- Attributes -->
  <attributes>
    <name>ПериодОтчета</name>
    <id>1</id>
    <valueType><types>StandardPeriod</types></valueType>
    <view><common>true</common></view>
    <edit><common>true</common></edit>
  </attributes>
  <attributes>
    <name>РезультатОтчета</name>
    <id>2</id>
    <valueType><types>SpreadsheetDocument</types></valueType>
    <view><common>true</common></view>
    <edit><common>true</common></edit>
    <extInfo xsi:type="form:SpreadsheetDocumentExtInfo"/>
  </attributes>

  <!-- Commands -->
  <formCommands>
    <name>Сформировать</name>
    <id>1</id>
    <use><common>true</common></use>
    <currentRowUse>Auto</currentRowUse>
  </formCommands>
  <formCommands>
    <name>ЗаполнитьНастройки</name>
    <id>2</id>
    <use><common>true</common></use>
    <currentRowUse>Auto</currentRowUse>
  </formCommands>

  <commandInterface>
    <navigationPanel/>
    <commandBar/>
  </commandInterface>
</form:Form>
```

## 6. Form Module Pattern

```1c
#Область ОбработчикиСобытийФормы

&НаСервере
Процедура ПриСозданииНаСервере(Отказ, СтандартнаяОбработка)
    ПериодОтчета.Вариант = ВариантСтандартногоПериода.ЭтотМесяц;
КонецПроцедуры

#КонецОбласти

#Область ОбработчикиКомандФормы

&НаКлиенте
Процедура Сформировать(Команда)
    СформироватьНаСервере();
КонецПроцедуры

&НаКлиенте
Процедура ЗаполнитьНастройки(Команда)
    ЗаполнитьНастройкиНаСервере();
    ПоказатьПредупреждение(, "Настройки заполнены.");
КонецПроцедуры

#КонецОбласти

#Область СлужебныеПроцедурыИФункции

&НаСервере
Процедура СформироватьНаСервере()
    ОтчетОбъект = РеквизитФормыВЗначение("Отчет");
    ОтчетОбъект.СформироватьОтчёт(РезультатОтчета,
        ПериодОтчета.ДатаНачала, ПериодОтчета.ДатаОкончания);
    ЗначениеВРеквизитФормы(ОтчетОбъект, "Отчет");
КонецПроцедуры

&НаСервереБезКонтекста
Процедура ЗаполнитьНастройкиНаСервере()
    Отчеты.МойОтчет.ЗаполнитьФормулыПоУмолчанию();
КонецПроцедуры

#КонецОбласти
```

## 7. UUID Validation Rules

UUIDs in MDO files MUST contain only hex digits (0-9, a-f):

```xml
<!-- CORRECT -->
<dimensions uuid="4c26a438-afe9-44fa-a5be-8c8811048e57">

<!-- WRONG — contains 'g', 'h', 'i' -->
<dimensions uuid="e8f4d2b3-6c5a-4903-b7d2-9g3f4e5c6b7d">  <!-- CRASH! -->
```

## 8. Information Register MDO

### Periodicity values (valid only):
- `Second`, `Minute`, `Hour`, `Day`, `Month`, `Quarter`, `Year`
- For non-periodical: simply **omit** `<informationRegisterPeriodicity>` entirely

```xml
<!-- CORRECT for non-periodical register -->
<mdclass:InformationRegister ...>
  <!-- NO informationRegisterPeriodicity element -->
</mdclass:InformationRegister>

<!-- WRONG -->
<informationRegisterPeriodicity>Nonperiodical</informationRegisterPeriodicity>  <!-- DOESN'T EXIST -->
```

## 9. Common Pitfalls Summary

| Problem | Cause | Fix |
|---------|-------|-----|
| Form elements don't display | Items wrapped in FormGroup | Put items at top level |
| SpreadsheetDocumentField blank | Used `form:SpreadsheetDocumentFieldExtInfo` | Use `form:SpreadSheetDocFieldExtInfo` (capital S) |
| Report doesn't open form | Missing `defaultForm` in MDO | Add `<defaultForm>Report.X.Form.Y</defaultForm>` |
| UUID crash | Invalid hex chars in UUID | Use only 0-9, a-f |
| Register export error | `Nonperiodical` value | Remove `informationRegisterPeriodicity` element |
| Buttons not working | `commandName` mismatch | Name must be `Form.Command.ИмяКоманды` |
| dataPath not binding | Plain string `dataPath` | Use `xsi:type="form:DataPath"` with `<segments>` |
| Handlers not firing | Missing `<handlers>` on form level | Add `<handlers><event>OnCreateAtServer</event>` |

## Source

Discovered during UNF unfAiAgent and unftest extension development:
- ФормаОтчета ОПиУ — period + buttons + spreadsheet document
- АнализБазыКонтрагентов — working reference form
- Баланс/Расшифровка — minimal working form pattern
