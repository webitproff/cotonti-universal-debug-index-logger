# Universal Debugging & Errors Logger for Cotonti

## Table of Contents

1. [Introduction](#introduction)
2. [The Problem](#problem)
3. [Why Standard Tools Don't Help](#why-standard-tools)
4. [What the Universal Handler Provides](#what-handler-provides)
5. [Requirements](#requirements)
6. [Installation: Step by Step](#installation)
7. [Configuration: Flags and Modes](#configuration)
8. [Where the Log File Lives](#log-location)
9. [Report Structure: Field by Field](#report-structure)
10. [In Practice: Reading the Report](#reading-the-report)
11. [Typical Scenarios](#scenarios)
12. [Disabling and Removal](#disabling)
13. [FAQ](#faq)

---

<a id="introduction"></a>

## 1. Introduction

Cotonti is a modular framework. Besides the core, it runs dozens of extensions: modules, plugins, themes, templates. When something breaks, the source of the problem is often not where PHP reports it. This makes diagnosis non-obvious and slow.

This guide describes a universal debugging approach: a single handler that is registered once at the entry point and collects the full picture of a failure. It does not touch the core, does not interfere with page output, and is toggled on and off with a single switch.

The guide is intended for a site administrator or a developer familiar with basic PHP concepts and the Cotonti structure. No special skills are required.

<a id="problem"></a>

## 2. The Problem

The classic case that usually triggers this whole diagnosis:

```
Deprecated: str_replace(): Passing null to parameter #3 ($subject)
of type array|string is deprecated
in /path/to/site/system/cotemplate.php on line 1923
```

The essence of the problem. A template contains a tag — for example `{PHP.c}`, `{MARKET_SOMETHING}`, or any other. No value was assigned to the tag. The template engine returns `null` when processing it. That `null` is then passed into the `str_replace()` function inside the core. Starting with PHP 8.1, passing `null` into this function is considered a deprecated construct. Hence the warning.

Similar situations:

- `Warning: Undefined array key` from a deep call — impossible to tell where the call came from.
- `Deprecated` in a module, but the message points to a core function.
- An error in a plugin, but the text only mentions a common function name.

What all of these have in common: the message shows the place of failure but does not show the cause or the culprit. Between those two points in Cotonti there can be 5–10 levels of calls.

<a id="why-standard-tools"></a>

## 3. Why Standard Tools Don't Help

Common approaches and their limitations:

**Editing the template.** The problem is not in the template but in the fact that the module's code did not assign a value to the tag. Editing the template accomplishes nothing here.

**Adding `var_dump`.** Requires editing a specific file. If the file is unknown, you have to edit the core, the template engine, plugins, and templates one by one. Every edit is temporary and must later be reverted. With multiple errors, the process drags on for hours.

**Editing the core.** More universal, but all edits are lost when Cotonti is updated. Plus, a modified core is a source of future problems that are hard to trace.

**Built-in PHP logs.** They record the fact of an error but provide no context: no URL, no stack, no tag name.

**A debugger like Xdebug.** A powerful tool, but it requires server configuration, an IDE, and reproducing the failure inside a debug session. On a production site, this is often unacceptable — the failure occurs on the client side, not on the developer's machine.

**Developer plugins.** Some ready-made ones exist, but they are usually either overloaded or tied to a specific Cotonti version. A custom handler is simpler and more reliable.

<a id="what-handler-provides"></a>

## 4. What the Universal Handler Provides

Instead of all the approaches listed above — a single PHP error interceptor that:

**Is registered once.** In `index.php`, at the entry point. No other edits are required anywhere.

**Does not touch the core.** Files in `system/` and all modules remain untouched. A Cotonti update will not break the debugging.

**Works on every page.** Any request, any module, any plugin.

**Does not interfere with output.** The report is written to a separate file. Page markup stays untouched, AJAX keeps working, visitors notice nothing.

**Is toggled on and off with a single variable.** Nothing needs to be cut out and pasted back in.

**Is configurable per task.** Catch errors from a single file, from the entire core, or from the entire project.

**Collects context.** Tag name, template file, culprit file, module name, call stack, URL, GET/POST, environment — everything lands in the report.

**Filters out noise.** By default it reacts only to the relevant class of errors, without cluttering the log with random warnings.

<a id="requirements"></a>

## 5. Requirements

- A Cotonti version that supports PHP 8.x.
- Edit access to the `index.php` file in the site root.
- Write permission on the directory containing `index.php` for the web server user.
- Basic understanding of the Cotonti structure: where `index.php` is, where `system/` is, where `plugins/` is, where `modules/` is.
- No third-party libraries or extensions need to be installed.

<a id="installation"></a>

## 6. Installation: Step by Step

**Step 1. Open `index.php`.**

The file is in the site root. Open it in any editor with PHP syntax highlighting.

**Step 2. Find the insertion point.**

The file contains two lines between which the debug block is inserted:

- the line `const COT_CODE = true;` — the upper boundary;
- the line `require_once './datas/config.php';` — the lower boundary.

These lines are not modified. They stay in place. Between them there is an empty slot where the new code goes.

**Step 3. Insert the debug block.**

Insert the handler code between the two lines mentioned above (it is provided as a separate file). When pasting, keep the indentation consistent with the surrounding code for readability.

**Step 4. Check permissions.**

Make sure the web server user has write permission on the directory containing `index.php`. This is required for creating the log file. If, for some reason, writing to the root is forbidden, the log file can be created in advance by hand and assigned the correct owner.

**Step 5. Save the file.**

Save `index.php`. No other files need to be modified.

**Step 6. Open any page of the site.**

Navigate to the page where the error occurs, or simply open the home page — the handler starts working from the very first request. The first records appear as soon as the condition is met.

**Step 7. Verify the log has been created.**

A file named `cot_index_debug.log` should appear in the directory containing `index.php`. If it did not appear, check permissions (see Step 4). If it appeared and is not empty, the handler is working.

<a id="configuration"></a>

## 7. Configuration: Flags and Modes

At the top of the debug block there are three variables. They control all of the behavior. Nothing else needs to be touched.

**Toggle variable.** The master flag. A value of `true` — debugging is active. A value of `false` — the handler is not registered, the block is effectively disabled. This lets you keep the code in `index.php` permanently and enable it only when needed.

**Filter-mode variable.** Determines which files errors are caught from:

- `'list'` — catch only from the files listed in the list (see below);
- `'all'` — catch from all project files: core, modules, plugins, themes.

**File list.** Used only in `'list'` mode. It is an array of file names to monitor. Comparison is done by partial path match, so it is enough to specify a bare file name without a directory — for example, `cotemplate.php` will catch a file with that name in any folder.

How to choose the mode:

- A single error in the template engine → mode `'list'`, with only `cotemplate.php` in the list.
- Core audit → mode `'list'`, with all files from `system/` in the list.
- Full diagnosis across all extensions → mode `'all'`.

Switching is done by changing a single character in the mode variable. Nothing to cut, nothing to paste — just change the value.

<a id="log-location"></a>

## 8. Where the Log File Lives

File: `cot_index_debug.log`. Location: in the same directory as `index.php`. That is, in the site root.

Write mode — append. This means that on every trigger, a new block is added to the end of the file. Older records are not erased. If you need to read the latest data, look at the end of the file. If you need to determine whether the same error keeps recurring, search the whole file.

Permissions: the file must be writable by the web server user. If the site runs as `www-data`, the file and directory must belong to it or be writable by it.

Size: under active monitoring the log grows quickly, especially in `'all'` mode. It is recommended to clear it periodically. The contents can be deleted without consequences — the file will be recreated on the next trigger. If the file needs to be temporarily "frozen" without deleting, simply disable debugging via the toggle flag.

<a id="report-structure"></a>

## 9. Report Structure: Field by Field

Each trigger is a separate block, delimited by separators. Inside there is a fixed set of fields.

**Date and time.** The moment of the trigger. Useful for correlating with other logs: the web server log, PHP logs, third-party service logs.

**Error code.** The numeric code PHP assigned to the error. For `Deprecated` this is 8192. For `Warning` it is 2. It helps distinguish message types from each other if the filtering was loosened.

**Message text.** The full wording from PHP. The very message visible on screen, but here — preserved as the entire string.

**Core file and line.** The place where PHP recorded the error. Usually this is either a line in `cotemplate.php` or in some other core function. This is **the place of failure**, not **the place of the cause**. There is no need to look at this file — in 99% of cases it is correct.

**URL.** The address of the page on which the trigger occurred, together with the query string. It clarifies under which requests the problem reproduces.

**HTTP method.** GET, POST, etc. Sometimes it helps understand the nature of the problem — for example, if the error only occurs on POST requests.

**Referer.** Where the user came from. Useful if the error is tied to a specific transition within the site.

**IP address.** The client whose request triggered the error. It lets you filter traffic by a specific user or session.

**User-Agent.** The browser or bot identification string. If the error only reproduces for certain clients, it will be visible here.

**Tag name.** The key field for template-related problems. This is the name of the tag that returned `null`. Displayed in curly braces so it visually stands out from the surrounding text.

**Template file.** The path to the `.tpl` file in which the problematic tag was encountered. It tells you which template to open.

**Culprit file.** The key field for diagnosis in general. This is the first PHP file outside the core from which the parsing call came. This is usually where the cause lies — a forgotten tag assignment, an incorrect template engine call, and so on. Provided in `file:line` format.

**Module or plugin name.** Determined from the path segment `plugins/<name>/` or `modules/<name>/`. If the error came from outside a plugin or module, `core` is written. This gives a quick idea of which part of the project the problem belongs to.

**Call stack.** The full chain of calls — from the error point to the entry point. Each line is a separate frame showing file, line, and function. For methods, the class is included. This block is the most informative, but also the bulkiest. It is used when the other fields gave no answer. The first file after the core in the stack is almost always the culprit.

**Environment snapshot.** Extension type, location, user ID. Useful for understanding under which conditions the failure reproduces.

**GET and POST.** Request parameters in JSON format. Sometimes the source of the error lies precisely in the request data — for example, if the client sent a value the code did not expect.

For different error types, some fields may be empty. That is normal. The handler tries to gather as much as possible, but some data is not always available. For template-related problems there will be no technical module information. For core-related errors there will be no tag name. You should look at whatever is filled in.

<a id="reading-the-report"></a>

## 10. In Practice: Reading the Report

The sequence for working with a single block.

**Step 1.** Open `cot_index_debug.log`. Go to the end of the file — the latest records are there.

**Step 2.** Read the **Message text** field. Understand the class of the problem: `str_replace`, `Undefined`, something else.

**Step 3.** Read the **Culprit file** field. This is the starting point. Open the indicated file, go to the indicated line.

**Step 4.** Look at what happens on that line. Usually it is a `$t->parse(...)`, `$t->assign(...)`, or similar template operation. Next to it is the code that builds the data for that call.

**Step 5.** If everything in the culprit file looks normal, look at the **Module or plugin name**. It is possible the problem is in a chain: the culprit file is called from another module, and the assignment was forgotten there.

**Step 6.** If that does not help, open the **Call stack**. Find the first file after the core. Often it is the real source.

**Step 7.** Read the **Tag name** and **Template file**. Confirm that the indicated template really contains such a tag, and that the corresponding PHP file does not assign it.

**Step 8.** Apply the fix. Save. Refresh the page. Verify that no new block has appeared in the log.

<a id="scenarios"></a>

## 11. Typical Scenarios

**Template complains, `Deprecated` from `str_replace`.** Mode `'list'`, with only `cotemplate.php` in the list. The report will have the full set: tag name, template file, culprit file. Open the culprit at the indicated line, find the spot where the tag is not assigned, add the assignment. Refresh the page — the block should disappear from the log.

**Plugin fails on a specific request.** Mode `'all'`. In the report — URL, HTTP method, stack. Find the plugin file in the stack, open it, look at the indicated line. Most likely, there is no input validation or the argument type is wrong.

**Full core audit.** Mode `'list'` with all files from `system/` in the list. All `Deprecated` and `Warning` from the core are visible. Process them one by one: look at the "Culprit file" field, fix, and verify that the record no longer appears in the log.

**Error only reproduces for a specific user.** Mode `'all'`. In the report — IP address and User-Agent. Correlate with known users or sessions. Use the URL to figure out which page failed.

**Error occurs on every page.** Mode `'all'`. In the report — call stack. Look for a common template: header, footer, sidebar. The problem is likely in one of the global variables or in a plugin included on every page.

**Intermittent failures with no obvious cause.** Mode `'all'`. Leave debugging running for several days. Correlate failures by date and time with other events on the server: load, actions of other admins, cache updates. Sometimes the cause turns out to be external — not in the code but in the data.

<a id="disabling"></a>

## 12. Disabling and Removal

**Temporary disable.** Set the toggle flag to `false`. The handler stops being registered. The code in `index.php` stays in place, the log stops growing.

**Re-enabling.** Change `false` back to `true`. Everything works again.

**Full removal.** Open `index.php`. Delete the entire inserted block — from the opening comment to the closing one. The file returns to its original state. The core and the other files are untouched.

**Deleting the log file.** The `cot_index_debug.log` file can be deleted at any time, regardless of whether debugging is running or not. It will be recreated on the next trigger.

<a id="faq"></a>

## 13. FAQ

**The log is not created.** A permissions issue. Determine which user the web server runs as and give that user write permission on the directory. Alternatively, create an empty `cot_index_debug.log` by hand and assign the correct owner.

**The log grows too fast.** Switch the mode from `'all'` to `'list'` and keep only the necessary files in the list. This sharply reduces the volume of records.

**My error is not in the log.** Check the toggle flag. Check that the error matches the message-text filter (by default — only `str_replace`). If you need to catch a broader class of errors, extend or remove the text filter.

**The error is there but the key fields are empty.** It is possible the error occurred outside the template parsing chain. In that case, look at the URL, stack, and environment — there will be enough data for diagnosis.

**The handler conflicts with other plugins.** It is possible the project already has its own `set_error_handler`. In that case our handler will replace it. The order is: our callback returns `false`, meaning "do not suppress the error" — it is passed further to PHP's standard handler. The functionality of other handlers is not affected.

**How to catch a broader class of errors.** Remove the message-text filter. The handler will then react to all errors that fall within `error_reporting()`. This is useful for a full audit, but the log grows very fast.

**How to exclude a specific file from monitoring.** In `'list'` mode, a file simply is not in the list — so it is not monitored. In `'all'` mode, excluding an individual file is possible by adding an extra check at the beginning of the callback.

**Can the handler be used on a development copy of the site.** Yes, it is universal. It also works on a production site, but you need to keep in mind the log file growth and watch the permissions.

**How to quickly find the most recent record.** Open the file and go to the end. A separator of `=` characters separates one block from another. The last separator before the end of the file is the start of the freshest block.

---

This is enough for working with the debugging in typical situations. If the guide needs to be extended — for example, adding a section on specific error types or on integration with external monitoring systems — it can be done as a separate addition.


___

# Универсальная отладка Cotonti: подробное руководство

## Содержание

1. [Введение](#intro-ru)
2. [Проблемная ситуация](#problem-ru)
3. [Почему стандартные средства не помогают](#why-ru)
4. [Что даёт универсальный обработчик](#what-ru)
5. [Требования](#req-ru)
6. [Установка: пошагово](#install-ru)
7. [Настройка: флаги и режимы](#config-ru)
8. [Где лежит лог-файл](#log-ru)
9. [Структура отчёта: поле за полем](#fields-ru)
10. [Практика: чтение отчёта](#reading-ru)
11. [Типовые сценарии](#scenarios-ru)
12. [Отключение и удаление](#off-ru)
13. [Частые вопросы](#faq-ru)

---

<a id="intro-ru"></a>

## 1. Введение

Cotonti — модульный фреймворк. Помимо ядра в нём работают десятки расширений: модули, плагины, темы, шаблоны. Когда что-то ломается, источник проблемы часто находится не там, где о ней сообщает PHP. Это делает диагностику неочевидной и долгой.

Данное руководство описывает универсальный способ отладки: единый обработчик, который регистрируется один раз в точке входа и собирает полную картину сбоя. Он не трогает ядро, не мешает выводу страницы и включается одним переключателем.

Руководство рассчитано на администратора сайта или разработчика, знакомого с базовыми понятиями PHP и структурой Cotonti. Специальных навыков не требуется.

<a id="problem-ru"></a>

## 2. Проблемная ситуация

Классический случай, ради которого обычно и начинается диагностика:

```
Deprecated: str_replace(): Passing null to parameter #3 ($subject)
of type array|string is deprecated
in /path/to/site/system/cotemplate.php on line 1923
```

Суть проблемы. Шаблон содержит тег — например `{PHP.c}`, `{MARKET_SOMETHING}` или любой другой. Тегу не было присвоено значение. Шаблонизатор при обработке возвращает `null`. Дальше `null` передаётся в функцию `str_replace()` внутри ядра. Начиная с PHP 8.1 передача `null` в эту функцию считается устаревшей конструкцией. Отсюда предупреждение.

Похожие ситуации:

- `Warning: Undefined array key` из глубокого вызова — невозможно понять, откуда пришёл вызов.
- `Deprecated` в модуле, но сообщение указывает на функцию ядра.
- Ошибка в плагине, но в тексте — только имя общей функции.

Общее у всех: сообщение показывает место сбоя, но не показывает причину и виновника. Между этими двумя понятиями в Cotonti может быть 5–10 уровней вызовов.

<a id="why-ru"></a>

## 3. Почему стандартные средства не помогают

Обычные подходы и их ограничения:

**Правка шаблона.** Проблема не в шаблоне, а в том, что код модуля не присвоил тегу значение. Правка шаблона тут ничего не даёт.

**Постановка `var_dump`.** Требует правки конкретного файла. Если файл неизвестен — надо править по очереди ядро, шаблонизатор, плагины, шаблоны. Каждая правка — временная, потом откатывать. При нескольких ошибках процесс затягивается на часы.

**Правка ядра.** Универсальнее, но при обновлении Cotonti все правки теряются. Плюс изменённое ядро — источник будущих проблем, которые сложно отследить.

**Встроенные логи PHP.** Логируют факт ошибки, но не дают контекст: ни URL, ни стека, ни имени тега.

**Отладчик типа Xdebug.** Мощный инструмент, но требует настройки сервера, IDE и воспроизведения сбоя в отладочной сессии. Для боевого сайта это часто неприемлемо — сбой возникает у клиента, а не у разработчика.

**Плагины для разработчика.** Есть готовые, но обычно они либо перегружены, либо завязаны на конкретную версию Cotonti. Собственный обработчик проще и надёжнее.

<a id="what-ru"></a>

## 4. Что даёт универсальный обработчик

Вместо всех перечисленных способов — один перехватчик ошибок PHP, который:

**Регистрируется один раз.** В `index.php`, в точке входа. Больше нигде правок не требуется.

**Не трогает ядро.** Файлы `system/` и все модули остаются в исходном виде. Обновление Cotonti не сломает отладку.

**Работает на всех страницах.** Любой запрос, любой модуль, любой плагин.

**Не вмешивается в вывод.** Отчёт пишется в отдельный файл. Вёрстка страницы остаётся нетронутой, AJAX работает, посетители ничего не замечают.

**Включается и выключается одной переменной.** Ничего резать и вставлять не нужно.

**Настраивается по задаче.** Ловить ошибки из одного файла, из всего ядра или из всего проекта.

**Собирает контекст.** Имя тега, файл шаблона, файл-виновник, имя модуля, стек вызовов, URL, GET/POST, окружение — всё попадает в отчёт.

**Отсеивает шум.** По умолчанию реагирует только на нужный класс ошибок, не засоряя лог случайными предупреждениями.

<a id="req-ru"></a>

## 5. Требования

- Cotonti версии, поддерживающей PHP 8.x.
- Доступ на редактирование файла `index.php` в корне сайта.
- Права на запись в директорию, где лежит `index.php`, для пользователя веб-сервера.
- Понимание базовой структуры Cotonti: где `index.php`, где `system/`, где `plugins/`, где `modules/`.
- Никаких сторонних библиотек и расширений устанавливать не требуется.

<a id="install-ru"></a>

## 6. Установка: пошагово

**Шаг 1. Открыть `index.php`.**

Файл находится в корне сайта. Открывается любым редактором с подсветкой PHP.

**Шаг 2. Найти точку вставки.**

В файле есть две строки, между которыми вставляется блок отладки:

- строка `const COT_CODE = true;` — верхняя граница;
- строка `require_once './datas/config.php';` — нижняя граница.

Эти строки не меняются. Они остаются на месте. Между ними — пустое место, куда вставляется новый код.

**Шаг 3. Вставить блок отладки.**

Между двумя указанными строками вставить код обработчика (предоставляется отдельным файлом). При вставке следить за отступами — они должны совпадать с окружающим кодом, для читаемости.

**Шаг 4. Проверить права.**

Убедиться, что пользователь веб-сервера имеет право на запись в директорию, где лежит `index.php`. Это нужно для создания лог-файла. Если по каким-то причинам запись в корень запрещена, файл лога можно создать заранее вручную и назначить ему владельца.

**Шаг 5. Сохранить файл.**

Сохранить `index.php`. Никаких других файлов менять не нужно.

**Шаг 6. Открыть любую страницу сайта.**

Перейти на страницу, где возникает ошибка, или просто открыть главную — обработчик начнёт работать с первого же запроса. Первые записи появятся сразу, как только сработает условие.

**Шаг 7. Проверить, что лог создан.**

В директории с `index.php` должен появиться файл `cot_index_debug.log`. Если он не появился — проверить права (см. Шаг 4). Если появился и не пуст — обработчик работает.

<a id="config-ru"></a>

## 7. Настройка: флаги и режимы

В начале блока отладки — три переменные. Ими управляется всё поведение. Ничего больше трогать не нужно.

**Переменная включения.** Мастер-флаг. Значение `true` — отладка работает. Значение `false` — обработчик не регистрируется, блок фактически отключён. Это позволяет оставить код в `index.php` насовсем и включать его только при необходимости.

**Переменная режима отбора.** Определяет, из каких файлов ловить ошибки:

- `'list'` — ловить только из файлов, перечисленных в списке (см. ниже);
- `'all'` — ловить из всех файлов проекта: ядро, модули, плагины, темы.

**Список файлов.** Используется только в режиме `'list'`. Это массив имён файлов, которые нужно мониторить. Сравнение идёт по частичному совпадению пути, поэтому достаточно указать имя без директории — например `cotemplate.php` поймает файл с таким именем в любой папке.

Логика применения:

- Одна ошибка в шаблонизаторе → режим `'list'`, в списке только `cotemplate.php`.
- Аудит ядра → режим `'list'`, в списке все файлы из `system/`.
- Полная диагностика по всем расширениям → режим `'all'`.

Переключение делается заменой одного символа в переменной режима. Ничего резать, ничего вставлять — только менять значение.

<a id="log-ru"></a>

## 8. Где лежит лог-файл

Файл: `cot_index_debug.log`. Расположение: в той же директории, что `index.php`. То есть в корне сайта.

Режим записи — дописывание. Это означает, что при каждом срабатывании новый блок добавляется в конец файла. Старые записи не стираются. Если нужно прочитать свежие данные — смотреть в конец файла. Если нужно понять, повторяется ли одна и та же ошибка — искать по всему файлу.

Права: файл должен быть доступен на запись пользователю веб-сервера. Если сайт работает под `www-data`, файл и директория должны принадлежать ему или быть доступными на запись.

Размер: при активном мониторинге лог растёт быстро, особенно в режиме `'all'`. Рекомендуется периодически его очищать. Содержимое можно удалять без последствий — при следующем срабатывании он создастся заново. Если файл нужно временно «заморозить», не удаляя, — достаточно выключить отладку флагом включения.

<a id="fields-ru"></a>

## 9. Структура отчёта: поле за полем

Каждое срабатывание — это отдельный блок, обрамлённый разделителями. Внутри — фиксированный набор полей.

**Дата и время.** Момент срабатывания. Полезно для сопоставления с другими логами: журналом веб-сервера, логами PHP, логами сторонних сервисов.

**Код ошибки.** Числовой код, который PHP присвоил ошибке. Для `Deprecated` это 8192. Для `Warning` — 2. Позволяет отличить типы сообщений друг от друга, если фильтрация была ослаблена.

**Текст сообщения.** Полная формулировка от PHP. То самое сообщение, которое видно на экране, но здесь — с сохранением всей строки целиком.

**Файл ядра и строка.** Место, где PHP зафиксировал ошибку. Обычно это либо строка в `cotemplate.php`, либо в другой функции ядра. Это **место сбоя**, но **не место причины**. На этот файл смотреть не надо — он в 99% случаев правильный.

**URL.** Адрес страницы, на которой произошло срабатывание, вместе с query-строкой. Даёт понимание, при каких запросах воспроизводится проблема.

**HTTP-метод.** GET, POST и т.д. Иногда помогает понять природу проблемы — например, если ошибка возникает только при POST-запросах.

**Referer.** Откуда пользователь пришёл. Полезно, если ошибка связана с конкретным переходом внутри сайта.

**IP-адрес.** Клиент, при запросе которого сработала ошибка. Даёт возможность отфильтровать трафик по конкретному пользователю или сессии.

**User-Agent.** Строка идентификации браузера или бота. Если ошибка воспроизводится только для определённых клиентов, будет видно здесь.

**Имя тега.** Ключевое поле при проблемах с шаблонами. Это имя тега, который вернул `null`. Оформлено в фигурных скобках — чтобы визуально отличалось от прочего текста.

**Файл шаблона.** Путь к `.tpl`, в котором встретился проблемный тег. По нему понятно, какой именно шаблон надо открыть.

**Файл-виновник.** Ключевое поле при диагностике вообще. Это первый PHP-файл вне ядра, откуда пришёл вызов парсинга. Чаще всего именно здесь и находится причина проблемы — забытое присвоение тега, неправильный вызов шаблонизатора и так далее. Указано в формате `файл:строка`.

**Имя модуля или плагина.** Определяется по сегменту пути `plugins/<name>/` или `modules/<name>/`. Если ошибка пришла вне плагина или модуля — пишется `core`. Это даёт быстрое понимание, к какой части проекта относится проблема.

**Стек вызовов.** Полная цепочка вызовов — от точки ошибки до точки входа. Каждая строка — отдельный фрейм с указанием файла, строки, функции. Для методов указывается класс. Этот блок — самый информативный, но и самый объёмный. Используется, когда остальные поля не дали ответа. Первый файл после ядра в стеке — почти всегда и есть виновник.

**Снимок окружения.** Тип расширения, локация, ID пользователя. Полезно для понимания, при каких условиях воспроизводится сбой.

**GET и POST.** Параметры запроса в формате JSON. Иногда источник ошибки лежит именно в данных запроса — например, если клиент передал значение, которого код не ожидал.

Для разных типов ошибок часть полей может быть пустой. Это нормально. Обработчик пытается собрать максимум, но некоторые данные доступны не всегда. При проблемах с шаблоном не будет никакой технической информации о модуле. При ошибках в ядре — не будет имени тега. Смотреть надо на то, что заполнено.

<a id="reading-ru"></a>

## 10. Практика: чтение отчёта

Порядок работы с одним блоком.

**Шаг 1.** Открыть `cot_index_debug.log`. Перейти в конец файла — там свежие записи.

**Шаг 2.** Прочитать поле **Текст сообщения**. Понять класс проблемы: `str_replace`, `Undefined`, что-то ещё.

**Шаг 3.** Прочитать поле **Файл-виновник**. Это отправная точка. Открыть указанный файл, перейти на указанную строку.

**Шаг 4.** Посмотреть, что в этой строке происходит. Обычно это вызов `$t->parse(...)`, `$t->assign(...)` или аналогичная операция с шаблоном. Рядом — код, который формирует данные для этого вызова.

**Шаг 5.** Если в файле-виновнике всё выглядит нормально — смотреть **Имя модуля или плагина**. Возможно, проблема в цепочке: файл-виновник вызывается из другого модуля, и там пропущено присвоение.

**Шаг 6.** Если не помогло — открыть **Стек вызовов**. Найти первый файл после ядра. Часто он и есть настоящий источник.

**Шаг 7.** Прочитать **Имя тега** и **Файл шаблона**. Убедиться, что в указанном шаблоне действительно есть такой тег, и что в соответствующем PHP-файле он не присвоен.

**Шаг 8.** Внести исправление. Сохранить. Обновить страницу. Проверить, что новый блок в логе не появился.

<a id="scenarios-ru"></a>

## 11. Типовые сценарии

**Ругается шаблон, `Deprecated` от `str_replace`.** Режим `'list'`, в списке только `cotemplate.php`. В отчёте будет полный набор: имя тега, файл шаблона, файл-виновник. Открыть виновника по указанной строке, найти место, где не присвоен тег, добавить присвоение. Обновить страницу — блок из лога должен пропасть.

**Плагин падает при определённом запросе.** Режим `'all'`. В отчёте — URL, HTTP-метод, стек. Найти в стеке файл плагина, открыть, посмотреть указанную строку. Скорее всего, отсутствует проверка входных данных или неверный тип аргумента.

**Аудит всего ядра.** Режим `'list'` со списком всех файлов из `system/`. Видно все `Deprecated` и `Warning` из ядра. Обрабатывать по одному: смотреть поле «Файл-виновник», исправлять, проверять, что запись в логе больше не появляется.

**Ошибка воспроизводится только у конкретного пользователя.** Режим `'all'`. В отчёте — IP-адрес и User-Agent. Сопоставить с известными пользователями или сессиями. По URL понять, на какой странице сбой.

**Ошибка возникает на каждой странице.** Режим `'all'`. В отчёте — стек вызовов. Искать общий шаблон: header, footer, sidebar. Проблема, вероятно, в одной из глобальных переменных или в плагине, подключаемом на всех страницах.

**Периодические сбои без явной причины.** Режим `'all'`. Оставить отладку работать на несколько дней. По дате и времени сопоставить сбои с другими событиями на сервере: нагрузкой, действиями других админов, обновлениями кеша. Иногда причина окажется внешней — не в коде, а в данных.

<a id="off-ru"></a>

## 12. Отключение и удаление

**Временное отключение.** Установить флаг включения в значение `false`. Обработчик перестаёт регистрироваться. Код в `index.php` остаётся на месте, лог не растёт.

**Повторное включение.** Заменить `false` на `true`. Всё снова работает.

**Полное удаление.** Открыть `index.php`. Удалить весь вставленный блок целиком — от начального комментария до закрывающего. Файл вернётся в исходный вид. Движок и остальные файлы не затронуты.

**Удаление лог-файла.** Файл `cot_index_debug.log` можно удалить в любой момент, независимо от того, работает отладка или нет. При следующем срабатывании он создастся заново.

<a id="faq-ru"></a>

## 13. Частые вопросы

**Лог не создаётся.** Проблема с правами. Проверить, под каким пользователем работает веб-сервер, и дать этому пользователю право на запись в директорию. Альтернатива — создать пустой `cot_index_debug.log` вручную и назначить владельца.

**Лог растёт слишком быстро.** Переключить режим с `'all'` на `'list'` и оставить в списке только нужные файлы. Это резко сократит объём записей.

**В логе нет моей ошибки.** Проверить флаг включения. Проверить, что ошибка соответствует фильтру по тексту сообщения (по умолчанию — только `str_replace`). Если нужно ловить более широкий класс ошибок — расширить или снять фильтр по тексту.

**Ошибка есть, но в отчёте пусто в ключевых полях.** Возможно, ошибка возникла вне цепочки парсинга шаблона. В этом случае смотреть URL, стек и окружение — там будет достаточно данных для диагностики.

**Обработчик конфликтует с другими плагинами.** Возможна ситуация, когда в проекте уже стоит свой `set_error_handler`. В этом случае наш обработчик заменит его. Порядок: наш колбэк возвращает `false`, что означает «не подавлять ошибку» — она уйдёт дальше в стандартный обработчик PHP. Функциональность других обработчиков при этом не страдает.

**Как получить более широкий класс ошибок.** Снять фильтр по тексту сообщения. Тогда обработчик будет реагировать на все ошибки, попадающие в `error_reporting()`. Это полезно для полного аудита, но лог растёт очень быстро.

**Как исключить конкретный файл из мониторинга.** В режиме `'list'` файла просто нет в списке — он и не мониторится. В режиме `'all'` исключить отдельный файл можно, добавив дополнительную проверку в начале колбэка.

**Можно ли использовать обработчик на development-копии сайта.** Да, он универсален. На боевом сайте тоже работает, но нужно помнить о росте лог-файла и следить за правами.

**Как быстро найти самую свежую запись.** Открыть файл и перейти в конец. Разделитель из символов `=` отделяет один блок от другого. Последний разделитель перед концом файла — начало свежего блока.

---

Этого достаточно для работы с отладкой в типовых ситуациях. Если понадобится расширить руководство — например, добавить раздел про конкретные виды ошибок или про интеграцию с внешними системами мониторинга — это можно сделать отдельным дополнением.
