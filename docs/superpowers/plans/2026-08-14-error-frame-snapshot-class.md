# Error Frame Snapshot Class Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or
> superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace frame structures in `ДиагностикаОшибок.ВСтруктуру()` with immutable public
`СнимокКадраОшибки` objects constructed directly from `ИнформацияОбОшибке`.

**Architecture:** `ПросмотретьЦепочку()` remains responsible for traversal, ordering, and truncation. The new class
reads and stores one physical frame, delegates envelope recognition to the internal module, and exposes only getters.
`ВСтруктуру()` maps viewed physical frames to class instances and returns them in `СнимкиКадров`.

**Tech Stack:** OneScript 2.x, OPM, OneUnit, asserts.

## Global Constraints

- Minimum supported runtime is OneScript `2.0.0`; target runtime family is OneScript `2.x`.
- Runtime dependencies are forbidden; `asserts` and `oneunit` remain development-only dependencies.
- Public modules remain `ФабрикаОшибок` and `ДиагностикаОшибок`; `СнимокКадраОшибки` is a public class.
- The class constructor accepts exactly one `ИнформацияОбОшибке` and rejects every other type.
- All class variables are private and start with `_`; the class exposes getters but no setters or exported variables.
- `Стек()` returns a new array on every call; `Данные()` returns the original arbitrary value without deep copying.
- `ВСтруктуру()` returns exactly `ТипОшибки`, `Усечена`, and `СнимкиКадров`; no version field is added.
- Diagnostics preserve physical outer-to-root order, tolerate unreadable optional properties, and inspect at most
  64 frames.
- Public constructors and methods use the repository documentation format; maximum line length is 120 characters.
- Tests follow `ТестДолжен_...`, OneUnit lifecycle, and explicit preparation/action/assertion comments.

---

### Task 1: Public `СнимокКадраОшибки` Value Object

**Files:**
- Create: `src/Классы/СнимокКадраОшибки.os`
- Create: `tests/ТестыСнимкаКадраОшибки.os`
- Modify: `src/internal/Модули/ВнутреннееУстройствоОшибок.os`

**Interfaces:**
- Consumes: `ВнутреннееУстройствоОшибок.ПроверитьОшибку(Ошибка)`.
- Produces: `ВнутреннееУстройствоОшибок.РаспознатьКадр(Кадр)` as a service export returning
  `Кадр, Вид, ТипОшибки, Данные`.
- Produces: public constructor `Новый СнимокКадраОшибки(Ошибка)` and getters `Вид()`, `ТипОшибки()`,
  `Сообщение()`, `Данные()`, `ИмяМодуля()`, `НомерСтроки()`, `ИсходнаяСтрока()`, and `Стек()`.

- [ ] **Step 1: Write failing constructor and getter tests**

Create `tests/ТестыСнимкаКадраОшибки.os` with focused tests that use real errors:

```bsl
#Использовать asserts
#Использовать "fixtures"
#Использовать "../src"

#Область Тесты

&Тест
Процедура ТестДолжен_СоздатьСнимокСобственногоКадра() Экспорт

	// Подготовка
	Данные = Новый Структура("Путь", "config.json");
	Ошибка = ФабрикаОшибок.Создать("fs.file_not_found", "Файл не найден", Данные);

	// Действие
	Снимок = Новый СнимокКадраОшибки(Ошибка);

	// Проверка
	Ожидаем.Что(ТипЗнч(Снимок)).Равно(Тип("СнимокКадраОшибки"));
	Ожидаем.Что(Снимок.Вид()).Равно("Ошибка");
	Ожидаем.Что(Снимок.ТипОшибки()).Равно("fs.file_not_found");
	Ожидаем.Что(Снимок.Сообщение()).Равно("Файл не найден");
	Ожидаем.Что(Снимок.Данные()).Равно(Данные);

КонецПроцедуры

&Тест
Процедура ТестДолжен_ВернутьНезависимыеМассивыСтека() Экспорт

	// Подготовка
	Ошибка = ФабрикаОшибок.Создать("core.failed", "Ошибка");
	Снимок = Новый СнимокКадраОшибки(Ошибка);
	ПервыйСтек = Снимок.Стек();
	ИсходныйРазмер = ПервыйСтек.Количество();

	// Действие
	ПервыйСтек.Добавить("Проверочный элемент");

	// Проверка
	Ожидаем.Что(Снимок.Стек().Количество()).Равно(ИсходныйРазмер);

КонецПроцедуры

&Тест
Процедура ТестДолжен_ОтклонитьНеверныйТипАргументаКонструктора() Экспорт

	// Подготовка
	Параметры = Новый Массив;
	Параметры.Добавить("не ошибка");

	// Действие и Проверка
	Ожидаем.Что(ЭтотОбъект)
		.Метод("СоздатьСнимок", Параметры)
		.ВыбрасываетИсключение("Ошибка должна иметь тип ИнформацияОбОшибке");

КонецПроцедуры

#КонецОбласти

#Область СлужебныеПроцедурыИФункции

Функция СоздатьСнимок(Ошибка)
	Возврат Новый СнимокКадраОшибки(Ошибка);
КонецФункции

#КонецОбласти
```

Add focused cases in the same file for a context frame, a foreign frame, damaged metadata, an unavailable
`ИмяМодуля`, and all location getters. Assert that attempting to read `Снимок.Вид` through a helper function throws,
while `Снимок.Вид()` succeeds; repeat the direct-field check for one private field rather than testing implementation
names such as `_Вид`.

- [ ] **Step 2: Run OneUnit and verify the RED state**

Run:

```powershell
oneunit execute
```

Expected: FAIL while loading or executing `ТестыСнимкаКадраОшибки` because type `СнимокКадраОшибки` does not exist.
Existing suites must remain discoverable.

- [ ] **Step 3: Export single-frame recognition without duplicating envelope logic**

Move `РаспознатьКадр` from `СлужебныеПроцедурыИФункции` to `СлужебныйПрограммныйИнтерфейс`, add `Экспорт`, and
document its structure result:

```bsl
// Распознаёт один физический кадр без обхода его причин.
//
// Параметры:
//  Кадр - ИнформацияОбОшибке - Распознаваемый физический кадр.
//
// Возвращаемое значение:
//  Структура:
//   * Кадр - ИнформацияОбОшибке - Исходный физический кадр;
//   * Вид - Строка - Распознанный вид кадра;
//   * ТипОшибки - Строка, Неопределено - Машинный тип собственного кадра;
//   * Данные - Произвольный, Неопределено - Данные собственного кадра.
Функция РаспознатьКадр(Кадр) Экспорт
```

Keep its existing tolerant marker/kind/type validation unchanged. Do not add a second envelope parser to the class.

- [ ] **Step 4: Implement the minimal immutable class**

Create `src/Классы/СнимокКадраОшибки.os` with this organization and behavior:

```bsl
#Использовать "../internal"

Перем _Вид; // Строка
Перем _ТипОшибки; // Строка, Неопределено
Перем _Сообщение; // Строка, Неопределено
Перем _Данные; // Произвольный, Неопределено
Перем _ИмяМодуля; // Строка, Неопределено
Перем _НомерСтроки; // Число, Неопределено
Перем _ИсходнаяСтрока; // Строка, Неопределено
Перем _Стек; // Массив

#Область ОбработчикиСобытий

// Создаёт безопасный снимок физического кадра ошибки.
//
// Параметры:
//  Ошибка - ИнформацияОбОшибке - Физический кадр для создания снимка.
Процедура ПриСозданииОбъекта(Ошибка)

	ВнутреннееУстройствоОшибок.ПроверитьОшибку(Ошибка);
	ОписаниеКадра = ВнутреннееУстройствоОшибок.РаспознатьКадр(Ошибка);

	_Вид = ОписаниеКадра.Вид;
	_ТипОшибки = ОписаниеКадра.ТипОшибки;
	_Данные = ОписаниеКадра.Данные;
	_Сообщение = Неопределено;
	_ИмяМодуля = Неопределено;
	_НомерСтроки = Неопределено;
	_ИсходнаяСтрока = Неопределено;
	_Стек = Новый Массив;

	БезопасноПрочитатьСвойства(Ошибка);

КонецПроцедуры

#КонецОбласти

#Область ПрограммныйИнтерфейс

Функция Вид() Экспорт
	Возврат _Вид;
КонецФункции

Функция ТипОшибки() Экспорт
	Возврат _ТипОшибки;
КонецФункции

Функция Сообщение() Экспорт
	Возврат _Сообщение;
КонецФункции

Функция Данные() Экспорт
	Возврат _Данные;
КонецФункции

Функция ИмяМодуля() Экспорт
	Возврат _ИмяМодуля;
КонецФункции

Функция НомерСтроки() Экспорт
	Возврат _НомерСтроки;
КонецФункции

Функция ИсходнаяСтрока() Экспорт
	Возврат _ИсходнаяСтрока;
КонецФункции

Функция Стек() Экспорт

	Результат = Новый Массив;
	Для Каждого ЭлементСтека Из _Стек Цикл
		Результат.Добавить(ЭлементСтека);
	КонецЦикла;

	Возврат Результат;

КонецФункции

#КонецОбласти
```

Document every getter with `Возвращаемое значение`. Implement `БезопасноПрочитатьСвойства(Ошибка)` in
`СлужебныеПроцедурыИФункции`: initialize first as shown, then read `Описание`, `ИмяМодуля`, `НомерСтроки`,
`ИсходнаяСтрока`, and `ПолучитьСтекВызовов()` in separate `Попытка/Исключение` blocks. Copy stack elements into `_Стек`
inside a guarded loop; never retain the platform array.

- [ ] **Step 5: Run the class and full suites in GREEN**

Run:

```powershell
oneunit execute
git diff --check
```

Expected: all factory, diagnostics, representation, and class tests PASS; whitespace check returns no output.

- [ ] **Step 6: Commit the public class checkpoint**

```powershell
git add -- `
  'src/Классы/СнимокКадраОшибки.os' `
  'src/internal/Модули/ВнутреннееУстройствоОшибок.os' `
  'tests/ТестыСнимкаКадраОшибки.os'
git commit -m "feat: добавить класс снимка кадра ошибки"
```

---

### Task 2: Integrate Snapshot Objects into `ВСтруктуру`

**Files:**
- Modify: `src/Модули/ДиагностикаОшибок.os`
- Modify: `src/internal/Модули/ВнутреннееУстройствоОшибок.os`
- Modify: `tests/ТестыПредставленияОшибок.os`
- Modify: `docs/superpowers/specs/2026-08-14-error-library-design.md`
- Modify: `docs/superpowers/plans/2026-08-14-error-library.md`

**Interfaces:**
- Consumes: `Новый СнимокКадраОшибки(Ошибка)` and all getters from Task 1.
- Consumes: `ВнутреннееУстройствоОшибок.ПросмотретьЦепочку(Ошибка)` returning `Кадры, Усечена`.
- Produces: `ДиагностикаОшибок.ВСтруктуру(Ошибка)` returning exactly `ТипОшибки, Усечена, СнимкиКадров`.
- Removes: `ВнутреннееУстройствоОшибок.СоздатьСнимок(Ошибка)` and private `СоздатьСнимокКадра(ОписаниеКадра)`.

- [ ] **Step 1: Rewrite representation tests for the new public contract**

In `tests/ТестыПредставленияОшибок.os`, replace all `.Кадры` access on the `ВСтруктуру()` result with
`.СнимкиКадров`, and replace direct structure fields with getters. The mixed-chain assertions become:

```bsl
ПроверитьПоляСнимка(Снимок);
Ожидаем.Что(Снимок.ТипОшибки).Равно("config.load_failed");
Ожидаем.Что(Снимок.Усечена).Равно(Ложь);
Ожидаем.Что(Снимок.СнимкиКадров.Количество()).Равно(4);
Ожидаем.Что(ТипЗнч(Снимок.СнимкиКадров[0])).Равно(Тип("СнимокКадраОшибки"));
Ожидаем.Что(Снимок.СнимкиКадров[0].Вид()).Равно("Контекст");
Ожидаем.Что(Снимок.СнимкиКадров[1].Вид()).Равно("Ошибка");
Ожидаем.Что(Снимок.СнимкиКадров[2].Вид()).Равно("ВнешняяОшибка");
Ожидаем.Что(Снимок.СнимкиКадров[3].Вид()).Равно("Ошибка");
Ожидаем.Что(Снимок.СнимкиКадров[1].ТипОшибки()).Равно("config.load_failed");
Ожидаем.Что(Снимок.СнимкиКадров[2].Данные()).Равно(Неопределено);
```

Replace `ПроверитьПоляСнимка` with exact top-level field checks:

```bsl
Процедура ПроверитьПоляСнимка(Снимок)

	Ожидаем.Что(Снимок.Количество()).Равно(3);
	Ожидаем.Что(Снимок.Свойство("ТипОшибки")).Равно(Истина);
	Ожидаем.Что(Снимок.Свойство("Усечена")).Равно(Истина);
	Ожидаем.Что(Снимок.Свойство("СнимкиКадров")).Равно(Истина);
	Ожидаем.Что(Снимок.Свойство("Кадры")).Равно(Ложь);
	Ожидаем.Что(Снимок.Свойство("Версия")).Равно(Ложь);

КонецПроцедуры
```

Delete the old `ПроверитьПоляКадра` structure-field helper. Update location, unavailable-property, foreign metadata,
fresh stack, and 64/65-frame tests to call getters. Preserve all current behavior assertions.

- [ ] **Step 2: Run OneUnit and verify the integration RED state**

Run:

```powershell
oneunit execute
```

Expected: the class suite from Task 1 passes, while representation tests FAIL because `ВСтруктуру()` still returns
`Кадры` containing structures and does not provide `СнимкиКадров`.

- [ ] **Step 3: Move snapshot assembly to the public diagnostics module**

Replace `ДиагностикаОшибок.ВСтруктуру` with:

```bsl
// Возвращает безопасное представление физической цепочки ошибки.
//
// Параметры:
//  Ошибка - ИнформацияОбОшибке - Исследуемая ошибка.
//
// Возвращаемое значение:
//  Структура:
//   * ТипОшибки - Строка, Неопределено - Эффективный тип ошибки;
//   * Усечена - Булево - Признак наличия непросмотренной причины;
//   * СнимкиКадров - Массив из СнимокКадраОшибки - Снимки физических кадров.
Функция ВСтруктуру(Ошибка) Экспорт

	Просмотр = ВнутреннееУстройствоОшибок.ПросмотретьЦепочку(Ошибка);
	Результат = Новый Структура(
		"ТипОшибки, Усечена, СнимкиКадров",
		Неопределено,
		Просмотр.Усечена,
		Новый Массив
	);

	Для Каждого ОписаниеКадра Из Просмотр.Кадры Цикл
		Результат.СнимкиКадров.Добавить(Новый СнимокКадраОшибки(ОписаниеКадра.Кадр));

		Если Результат.ТипОшибки = Неопределено И ОписаниеКадра.Вид = ВидыКадровОшибок.Ошибка Тогда
			Результат.ТипОшибки = ОписаниеКадра.ТипОшибки;
		КонецЕсли;
	КонецЦикла;

	Возврат Результат;

КонецФункции
```

Ensure the source loader can discover `src/Классы` through the existing `#Использовать "../src"` package root; add
a relative `#Использовать` only if the focused RED run proves discovery is not automatic.

- [ ] **Step 4: Delete obsolete internal snapshot assembly**

From `src/internal/Модули/ВнутреннееУстройствоОшибок.os`, delete the exported `СоздатьСнимок(Ошибка)` function and
the private `СоздатьСнимокКадра(ОписаниеКадра)` function. Keep `ПросмотретьЦепочку`, exported `РаспознатьКадр`, and
all shared validation helpers.

- [ ] **Step 5: Update the contract documentation**

In both existing design and implementation-plan documents:

- replace the top-level `Кадры` field of `ВСтруктуру` with `СнимкиКадров`;
- state that each element has type `СнимокКадраОшибки` and is read through getters;
- remove claims that frame snapshots are `Структура` values with public fields;
- document the direct constructor and eight getters;
- keep the explicit absence of structural-snapshot versioning;
- keep physical ordering, safe optional reads, and the 64-frame limit unchanged.

- [ ] **Step 6: Run full verification in GREEN**

Run:

```powershell
oneunit execute
git diff --check
rg -n "\.Кадры\[[0-9]+\]\.(Вид|ТипОшибки|Сообщение|Данные|ИмяМодуля|НомерСтроки|ИсходнаяСтрока|Стек)" tests
rg -n "Структура.*снимк|Кадры.*Массив из Структура" docs/superpowers
```

Expected: all tests PASS, `git diff --check` produces no output, and both searches produce no stale public-contract
matches. Internal `Просмотр.Кадры` and query methods over it remain valid and must not be renamed.

- [ ] **Step 7: Commit the integration checkpoint**

```powershell
git add -- `
  'src/Модули/ДиагностикаОшибок.os' `
  'src/internal/Модули/ВнутреннееУстройствоОшибок.os' `
  'tests/ТестыПредставленияОшибок.os' `
  'docs/superpowers/specs/2026-08-14-error-library-design.md' `
  'docs/superpowers/plans/2026-08-14-error-library.md'
git commit -m "refactor: вернуть снимки кадров объектами"
```
