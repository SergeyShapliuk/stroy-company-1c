# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

A 1C:Enterprise 8.3 configuration **«Управление строительной компанией»** (`УправлениеСтроительнойКомпанией`), stored as a hierarchical XML dump of the configuration (`ConfigDumpInfo.xml` `format="Hierarchical"`, dump format `version="2.20"`, compatibility mode `Version8_3_27`, managed application / Taxi interface, managed locks, Russian script variant). It is a training project: comments reference lesson numbers («занятие 17», «занятие 20»), and `DataProcessors/УчебнаяКонсоль` is a sandbox of language exercises (much of it gated behind `Если Ложь Тогда` or commented out).

There is no build system, linter, or test suite in the repo. Code is compiled and run only inside the 1C platform: changes made here have to be loaded into an infobase through the Designer (Конфигурация → Загрузить конфигурацию из файлов, or `1cv8 DESIGNER ... /LoadConfigFromFiles <dir>`), and changes made in the Designer come back via Выгрузить конфигурацию в файлы (`/DumpConfigToFiles`). `.idea/` is just a JetBrains IDE project wrapper; it is not part of the configuration.

## Layout conventions of the XML dump

- Each metadata object is `<Kind>/<Name>.xml` (properties, attributes, tabular sections, UUIDs) plus an optional `<Kind>/<Name>/` folder holding modules and forms:
  - Object module: `<Kind>/<Name>/Ext/ObjectModule.bsl`
  - Common module: `CommonModules/<Name>/Ext/Module.bsl`
  - Form: `<Kind>/<Name>/Forms/<Form>.xml` (form metadata), `.../Forms/<Form>/Ext/Form.xml` (layout, attributes, commands, event bindings), `.../Forms/<Form>/Ext/Form/Module.bsl` (form module).
- **Adding/removing/renaming a metadata object requires editing several files consistently**: the object's own XML, the `<ChildObjects>` list in `Configuration.xml`, `ConfigDumpInfo.xml` (object IDs/versions), and optionally a subsystem's `<Content>` in `Subsystems/*.xml`. New objects need fresh UUIDs. It is usually safer to create objects in the Designer and dump, and to edit only `.bsl` modules and existing attribute properties by hand.
- Form event handlers and commands are wired in `Form.xml` by procedure name (e.g. `МатериалыКоличествоПриИзменении`, command `Согласовать`); renaming a handler in `Module.bsl` without updating `Form.xml` breaks the binding.
- Files are UTF-8 with BOM; keep the BOM and tab indentation when editing `.bsl`.

## Domain model

- **Catalogs**: `Товары` (Артикул, Цена, ЕдиницаИзмерения, Описание — also used as "materials"), `Клиенты` (with `Enum.ВидыКлиентов`: Заказчик/Поставщик/Подрядчик), `Сотрудники`, `Склады`, `ОбъектыСтроительства`.
- **Documents**:
  - `ЗаявкаНаМатериалы` — material request (ОбъектСтроительства, Ответственный, Приоритет, Состояние, Комментарий, tabular section `Материалы`: Материал/Количество/Цена/Сумма). Its lifecycle is the `Enum.СостоянияЗаявок` state (Новая → Согласована / Отклонена), changed only by form commands `Согласовать` / `Отклонить` (the state field is read-only; `ПриКопировании` resets it to Новая; an empty state from old data is shown as Новая). Both commands first save unsaved edits with the form's own `Записать()` (so errors are highlighted), then change the state of the written document on the server: approval re-runs `ПроверитьЗаполнение()`, rejection deliberately does not. `ОбработкаПроверкиЗаполнения` rejects an empty `Материалы` and validates each row, reporting errors bound to `Материалы[i].<field>`; the platform-level fill check on the `Материал` column is turned off to avoid duplicate messages.
  - `ПоступлениеМатериалов` — receipt, entered from a `ЗаявкаНаМатериалы` via the request form's own «Создать на основании → Поступление материалов» command (`СоздатьПоступлениеМатериалов`, opens the receipt form with parameter `Основание` and a fresh uniqueness key). The metadata `BasedOn` list is intentionally empty: the platform's generated command re-activated an already open unsaved receipt form even for a different request. (`ОбработкаЗаполнения` copies Ответственный and rows; check-fill refuses a basis request that is not Согласована, an empty basis is allowed). Form has a read-only total and a `ПересчитатьСуммы` command that refreshes prices from `Товары` in a single server call.
- Posting is **denied on both documents** on purpose: there are no registers yet (stage 4 of the course, see `TECHDEBT.md`). A side effect relied upon: for non-posted documents fill checking (`ОбработкаПроверкиЗаполнения` + attribute «Проверка заполнения») runs on interactive write. Programmatic `Записать()` skips it, so server code that writes a document (e.g. `ЗаписатьСНовымСостоянием` in the request form) calls `ПроверитьЗаполнение()` explicitly.
- **Constants**: `НаименованиеКомпании`, `ДатаНачалаУчёта`.
- **Subsystems** (top-level sections): Продажи, Компания, Логистика, Разработка (the training data processors).

## Shared code

- `УчебныеФункции` — **client + server** module without `ServerCall`: `РассчитатьСумму`, `РассчитатьСуммуСНДС`. Compiled on both sides, so row recalculation in document forms runs locally with no server round-trip. Keep its code free of server-only objects.
- `УчебныеФункцииКлиент` — client-only module (`ОформитьСообщение`).
- `НормализацияСправочниковСервер` — server + ServerCall: `СобратьСтатистику`, `НормализоватьТовары` (uppercases/trims Артикул, defaults ЕдиницаИзмерения to `"шт."`, writes only changed items, counts errors), `СоздатьТоварЕслиНет`. Used by the `НормализацияСправочников` data processor form (commands Проверить / Исправить). The commented-out block at the top of the module is an earlier version kept for reference.

## Code conventions observed

- All identifiers and messages are in Russian; follow the same naming style (`ПересчитатьИтогПоЗаявке`, `...НаСервере` suffix for the server half of a client command).
- Form modules split work explicitly with `&НаКлиенте` / `&НаСервере` / `&НаСервереБезКонтекста`; client handlers call a server function that returns a result structure or an error string, and the client shows it (`Сообщить`, `ПоказатьПредупреждение`).
- User-facing validation errors use `СообщениеПользователю` with `Поле` and `УстановитьДанные(ЭтотОбъект)` so they attach to the field.
- Predefined enum values on the client are referenced via `ПредопределенноеЗначение("Перечисление....")`; on the server via `Перечисления.<Enum>.<Value>`.
