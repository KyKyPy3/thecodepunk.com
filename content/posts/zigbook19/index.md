---
title: Сборка проекта
date: 2025-04-22 15:00:00
showTableOfContents: true
showComments: true
katex: true
tags:
  - zig
  - zigbook
  - build
---

Если 20 лет назад при появлении нового языка программирования наличие встроенного инструментария для сборки не было обязательным требованием, то сейчас, если у языка нет качественного тулинга «из коробки», риск того, что он не станет популярным, весьма высок.

Хороший инструментарий для управления кодом проекта — это краеугольный камень, от качества реализации которого зависит, будут ли разработчики использовать новый язык или нет. Сегодня программисты ожидают, что язык предложит встроенные инструменты, которые умеют:

* **Управлять зависимостями проекта и версионированием самого проекта**
  <br>
  Простое подключение внешних библиотек, «lock‑файл» для повторяемости сборок, удобные команды обновления и возможности разрешения конфликтов библиотек.

* **Запускать тесты, линтеры, статический анализ и фаззинг**
  <br>
  Сейчас уже не достаточно иметь возможность только запустить unit тесты, современный мир требует от разработчика запуска интеграционных, фаззинг и тестов производительности. Проведение статического анализа кода, анализа на наличие CVE и других уязвимостей кода.

* **Форматировать исходники по официальному стилю**
  <br>
  Автоформаттер устраняет бессмысленные споры в code review и ускоряет командную работу. Наличие языков со встроенным тулингом автоформатирования кода сильно упрощает разработку в командах.

* **Собирать проект в разных режимах**
  <br>
  Поддержка различных режимов сборки (debug, release, профилировка) и возможность гибко настраивать параметры компиляции.

* **Генерировать дополнительные артефакты**
  <br>
  Документацию, gRPC‑стабы, OpenAPI‑спеки, миграции БД и т. д. Сегодня многие проекты при сборке порождают множество второстепенных артефактов, и удобно, когда их генерацию можно встроить в основную систему сборки.

* **Кросс‑компилировать под несколько платформ**
  <br>
  Сборка проекта для всех поддерживаемых платформ (Windows, Linux, macOS, ARM, WebAssembly) нередко требует «танцев с бубном» даже в тех языках, которые заявляют полную кросс‑компиляцию. Так, например, собрать приложение на Rust под все операционные системы не так просто, как это описано в документации.

* **Масштабироваться под любой размер проекта**
  <br>
  Сейчас нередко проекты представляют из себя огромные кодовые базы со сложной иерархией взаимоотношений и хорошо если тулинг одинаково быстро справляться как с «игрушечной» CLI‑утилитой, так и с монорепой на миллионы строк кода.

И это лишь базовый список: часто ждут ещё hot‑reload, шаблоны проектов («scaffolding»), интеграцию с IDE и облачными CI/CD.

Я видел немало перспективных языков, чья популярность угасала именно из‑за отсутствия такого фундамента. Автор фокусировался на синтаксисе или фичах рантайма или языка, но не уделял времени тому, как разработчику на практике собрать и запустить реальный продукт. В результате люди приходили в язык, возились со сборкой и уходили, так как не готовы бесконечно писать кастомные скрипты и заниматься их поддержкой.

Язык Zig разрабатывался как язык где вы фокусируетесь на отладке логики вашего кода, а не на отладке особенностей языка или особенностей сборки. Поэтому авторы языка сразу начали продумывать и развивать тулинг в языке, продумывая систему сборки, тестирования и анализа кода. Конечно тулинг в Zig на текущий момент еще молод и не все желаемые фичи реализованы, но уже есть много полезных инструментов и возможностей для разработки.

Но давайте уже начнем погружаться в то, что дает нам система сборки в Zig и начнем мы с различных режимов сборки.

## Режимы сборки
На текущий момент система сборки Zig поддерживает четыре режима сборки: `Debug`, `ReleaseFast`, `ReleaseSafe` и `ReleaseSmall`. Каждый из этих режимов сборки предлагает различные преимущества и характеристики. Компилятор zig по умолчанию использует режим сборки `Debug`, когда вы явно не выбираете режим сборки. Давайте рассмотрим чем отличаются эти 4 режима сборки проекта:

### Debug
В режиме Debug компилятор Zig собирает программу без оптимизаций, но с полной отладочной информацией. Это удобно для разработки и тестирования: вы можете пошагово просматривать выполнение кода. Кроме того, в Debug‑сборке автоматически включаются все защитные проверки — assert, контроль выхода за границы массивов и т. д.

### ReleaseFast
В режиме ReleaseFast компилятор Zig ориентируется на максимальную скорость выполнения программы. Ради этого отключается часть защитных проверок (что может привести к неопределённому поведению, если код небезопасен), а время компиляции обычно возрастает. Этот профиль стоит выбирать, когда в вашем приложении производительность критична — к примеру, для игр или high‑performance‑вычислений.

### ReleaseSafe
Если вам нужна высокая производительность, но при этом важно сохранить большую часть защитных проверок, выбирайте профиль ReleaseSafe. В этом режиме Zig отключает лишь те проверки, которые существенно замедляют работу (например, некоторые дорогостоящие runtime‑проверки), но оставляет контроль на undefined behavior и другие критически важные гарантии.

Таким образом, ReleaseSafe — это компромисс между максимальной скоростью и полной безопасностью: вы получаете почти «релизную» производительность, не отказываясь от ключевых механизмов защиты.

### ReleaseSmall
В режиме ReleaseSmall компилятор Zig оптимизирует программу так, чтобы минимизировать размер итогового исполняемого файла. Все защитные проверки отключены, а рантайм урезан до предела. Такой вариант сборки особенно полезен при подготовке прошивок и приложений для встраиваемых устройств, где каждый килобайт важен.

Давайте теперь возьмем простую программу на Zig и попробуем собрать ее в различных режимах и посмотрим на отличия в результате сборки. Вот код нашей простой программы:

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.DebugAllocator(.{ .verbose_log = true }).init;
    defer {
        const result = gpa.deinit();
        std.debug.print("Gpa deinit result: {}\n", .{result});
    }

    var list = std.ArrayList(u8).init(gpa.allocator());
    defer list.deinit();

    try list.append(13);
    try list.append(42);

    std.log.info("Log print", .{});
    std.debug.print("Debug print\n", .{});
}
```

Сначала давайте соберем и запустим нашу программу как делали это раньше, в режиме `Debug`:

```
$ zig build run -Doptimize=Debug
info(gpa): small alloc 128 bytes at 0x1010e0000
info: Log print
Debug print
info(gpa): small free 128 bytes at u8@1010e0000
Gpa deinit result: heap.Check.ok

$ l ./zig-out/bin/simple
-rwxr-xr-x@ 1 roman  staff   1.3M Apr 24 00:19 ./zig-out/bin/simple
```

В результате сборки в режиме `Debug` мы получили исполняемый файл размером около 1.3 МБ. Это связано с тем, что в режиме `Debug` компилятор Zig включает дополнительные проверки и отладочную информацию, которая увеличивает размер исполняемого файла. В выводе нашей программы мы видим все сообщения, которые мы хотели вывести. Теперь давайте соберем нашу программу в режиме `ReleaseFast`:

```
$ zig build run -Doptimize=ReleaseFast
Debug print
Gpa deinit result: heap.Check.ok

$ l ./zig-out/bin/simple
-rwxr-xr-x@ 1 roman  staff   211K Apr 24 00:23 ./zig-out/bin/simple
```

После сборки в режиме `ReleaseFast` размер исполняемого файла уменьшился до 211 КБ — значительно меньше, чем в режиме `Debug`. Это объясняется тем, что компилятор Zig в режиме `ReleaseFast` выполняет оптимизации и убирает лишние проверки вместе с отладочной информацией.

Кроме того, весь вывод, отправлявшийся через систему логирования, исчез: в консоль выводятся только сообщения, печатаемые функцией `std.debug.print`. Давайте теперь соберем в режиме `ReleaseSafe`:

```
$ zig build run -Doptimize=ReleaseSafe
info(gpa): small alloc 128 bytes at 0x100280000
info: Log print
Debug print
info(gpa): small free 128 bytes at u8@100280000
Gpa deinit result: heap.Check.ok

$ l ./zig-out/bin/simple
-rwxr-xr-x@ 1 roman  staff   304K Apr 24 00:22 ./zig-out/bin/simple
```

В режиме `ReleaseSafe` размер исполняемого файла увеличился до 304 КБ, так как в этом режиме компилятор Zig выполняет дополнительные проверки и оптимизации, которые могут увеличить размер исполняемого файла. А также к нам снова вернулся вывод сообщений через систему логирования. И наконец давайте соберем наше приложение в режиме `ReleaseSmall`:

```
$ zig build run -Doptimize=ReleaseSmall
Debug print
Gpa deinit result: heap.Check.ok

$ l ./zig-out/bin/simple
-rwxr-xr-x@ 1 roman  staff    57K Apr 24 00:22 ./zig-out/bin/simple
```

Вывод через логирование снова пропал, а размер нашего приложения снизился до 57 КБ. Это довольно существенное уменьшение, если сравнивать его например с размером исполняемого файла в режиме `Debug`.

Помимо готовых режимов сборки, в системе Zig есть настройки, которые позволяют тонко управлять размером и производительностью программы. Например, даже собираясь в режиме `Debug`, можно передать опцию `strip`, чтобы удалить из исполняемого файла отладочные символы и тем самым уменьшить его размер.

```zig
...
const target = b.standardTargetOptions(.{});
const optimize = b.standardOptimizeOption(.{});

const lib_mod = b.createModule(.{
    .root_source_file = b.path("src/root.zig"),
    .target = target,
    .optimize = optimize,
});

const exe_mod = b.createModule(.{
    .root_source_file = b.path("src/main.zig"),
    .target = target,
    .optimize = optimize,
});

exe_mod.strip = true; // Добавили вот эту строчку
...
```

В результате после сборки нашего приложения его размер должен существенно уменьшится:

```
$ zig build run -Doptimize=Debug
info(gpa): small alloc 128 bytes at 0x1001c0000
info: Log print
Debug print
info(gpa): small free 128 bytes at u8@1001c0000
Gpa deinit result: heap.Check.ok

$ l ./zig-out/bin/simple
-rwxr-xr-x@ 1 roman  staff   979K Apr 24 09:54 ./zig-out/bin/simple
```

Итак мы рассмотрели с вами режимы сборки приложения и давайте теперь перейдем к самой системе сборки Zig.

## Система сборки Zig
Система сборки Zig с точки зрения разработчика ­— это файл `build.zig`, в котором должна присутствовать публичная функция `build`, с единственным параметром типа `*std.Build`. Это эквивалентно требованию для исполняемого файла иметь функцию `main`, которая является точкой входа в программу.

Во всём остальном это обычный Zig‑код, такой же, как и любой другой код вашего приложения: в нём можно выполнять вычисления, читать и записывать файлы, отправлять запросы на сервер и т. д.

Когда нам надо собрать наше приложение в любом языке программирования, это обычно состоит из набора задач, которые нам надо решить:

* Обработка различных флагов сборки, которые мы предоставляем пользователю для управления процессом сборки
* Сборка внешних зависимостей нашего приложения, в том числе и написанных на другом языке программирования
* Сборка наших библиотек в статическом или в динамическом режиме
* Сборка основного кода нашего приложения

Для того, чтобы выполнить указанные задачи сборки приложения, разработчик должен задать шаги (steps) в сборочном скрипте `build.zig`. В целом этот подход не уникален и должен быть вам знаком если вы уже использовали такие системы сборки как CMake, Go Task, Octa и д.р. Как только вы описали все необходимые шаги сборки вашего приложения, система сборки Zig построит ациклический граф зависимостей и выполнит все необходимые шаги сборки в правильном порядке.

При описании шагов сборки есть два варианта: воспользоваться готовыми шагами, поставляемыми вместе с системой сборки Zig, или написать собственные шаги на языке Zig. Сначала рассмотрим, какие шаги доступны «из коробки», а затем разберёмся, как добавлять в сборку свои собственные.

В стандартной поставке Zig доступны следующие шаги сборки:
* **addExecutable** - добавляет сборку исполняемого файла
* **addLibrary** - добавляет сборку статической или динамической библиотеки
* **addTest** - добавляет сборку исполняемого файла с тестами приложения

У каждого из перечисленных выше шагов сборки есть набор параметров, позволяющих настроить его поведение. Рассмотрим следующую задачу: мы хотим собрать исполняемый модуль программы и статическую библиотеку. Это примерно то, что создаётся командой `zig init`, но мы напишем сборочный скрипт с нуля. Код нашей библиотеки и исполняемого файла довольно прост:

```zig
// Файл root.zig
const std = @import("std");

pub export fn add(a: i32, b: i32) i32 {
    return a + b;
}
```

```zig
// Файл main.zig
const std = @import("std");
const simple_lib = @import("simple_lib");

pub fn main() !void {
    const result = simple_lib.add(1, 2);

    std.debug.print("add result: {d}\n", .{result});
}
```


Итак давайте начнём со сборки статической библиотеки, потому что исполняемый файл зависит от неё и без неё не соберётся. Чтобы добавить сборку библиотеки, нужно вставить следующий код в файл `build.zig`:

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const lib_mod = b.createModule(.{
        .root_source_file = b.path("src/root.zig"),
        .target = b.resolveTargetQuery(.{
            .cpu_arch = .aarch64,
            .os_tag = .macos,
        }),
        .optimize = .Debug,
    });

    const lib = b.addLibrary(.{
        .linkage = .static,
        .name = "simple",
        .root_module = lib_mod,
    });

    b.installArtifact(lib);
}
```

Давайте разберёмся, что получилось. Чтобы собрать библиотеку, мы сперва создаём модуль, который будет участвовать в её сборке. В нём указываем путь к корневому файлу библиотеки, параметры оптимизации и целевую платформу. При указании пути к нашему файлу с исходным кодом библиотеки мы используем метод `path` у билд системы, что позволяет нам использовать относительный путь от корня нашего проекта.

Параметры оптимизации мы уже обсуждали, а вот выбор платформы ещё нет. Для любого артефакта нужно указать, под какую платформу и систему он собирается. Компилятору передаются три значения:
*	архитектура CPU (например, aarch64 или x86_64);
*	операционная система (windows, macos, linux и др.);
*	версия ABI, если ее уточнение необходимо;

Мы собираем приложение под macOS на базе M1, поэтому в качестве архитектуры указываем `aarch64`, а в имени ОС — `macos`. О кросс‑компиляции поговорим позже, а сейчас вернёмся к коду. Чтобы добавить шаг сборки библиотеки, вызываем метод `addLibrary` и передаём в него:

*	имя библиотеки;
*	тип сборки (статическая или динамическая);
*	созданный на предыдущем шаге модуль;

Затем вызываем `installArtifact`, который поместит собранную библиотеку в каталог `zig-out/lib`. Если этот шаг пропустить, то библиотека будет собрана и удалена по окончании процесса сборки. В целом, так как мы собираем статически слинкованную библиотеку, нас устраивает вариант, при котором её удаляют. Однако мы решили пока оставлять её после сборки, чтобы лучше понимать, что происходит.

В методе `addLibrary` мы указали имя нашей библиотеки как `simple` в результате чего сборочная система создаст файл с именем `libsimple.a` в каталоге `zig-out/lib`. Приставка `lib` добавляется автоматически и нужна чтобы показать что это библиотека.

Итак, это всё, что нужно было сделать для сборки библиотеки. Давайте запустим процесс и посмотрим, что получилось:

```
$ zig build --summary all
Build Summary: 3/3 steps succeeded
install success
└─ install simple success
   └─ zig build-lib simple Debug aarch64-macos success 692ms MaxRSS:204M

$ l ./zig-out/lib/libsimple.a
-rw-r--r--@ 1 roman  staff   1.6M Apr 24 14:06 ./zig-out/lib/libsimple.a
```

Статическая библиотека успешно собралась и теперь лежит в каталоге `zig-out/lib`. Посмотрим, как подключить её к программе, и соберём основной исполняемый модуль:

```zig
// тут все что было раньше для сборки библиотеки

const exe_mod = b.createModule(.{
    .root_source_file = b.path("src/main.zig"),
    .target = b.resolveTargetQuery(.{
        .cpu_arch = .aarch64,
        .os_tag = .macos,
    }),
    .optimize = .Debug,
});

exe_mod.addImport("simple_lib", lib_mod);

const exe = b.addExecutable(.{
    .name = "simple",
    .root_module = exe_mod,
});

b.installArtifact(exe);
```

Чтобы собрать исполняемый модуль, мы выполняем шаги, аналогичные сборке библиотеки, с несколькими отличиями. Во‑первых, добавляем библиотеку в список импортов модуля через метод `addImport`. если этого не сделать то мы получим ошибку что компилятор не нашел импортируемую нами библиотеку. Во‑вторых, вместо `addLibrary` используем метод `addExecutable`, который создаёт шаг компиляции исполняемого файла. Запустим сборку и убедимся, что в результате мы получим готовую программу в каталоге `zig-out/bin`:

```
$ zig build --summary all
Build Summary: 5/5 steps succeeded
install success
├─ install simple success
│  └─ zig build-lib simple Debug aarch64-macos success 720ms MaxRSS:210M
└─ install simple success
   └─ zig build-exe simple Debug aarch64-macos success 858ms MaxRSS:237M

$ l ./zig-out/bin/simple
-rwxr-xr-x@ 1 roman  staff   1.2M Apr 24 14:19 ./zig-out/bin/simple

$ ./zig-out/bin/simple
All your codebase are belong to us.
Run `zig build test` to run the tests.
```

Как видим, всё успешно собралось и работает. Однако код можно слегка улучшить. Сейчас параметры оптимизации и целевая платформа зашиты прямо в исходнике, что неудобно: лучше использовать значения по умолчанию, которые при необходимости переопределяются через командную строку. Такая возможность существует, поэтому внесём соответствующие правки:

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const lib_mod = b.createModule(.{
        .root_source_file = b.path("src/root.zig"),
        .target = target,
        .optimize = optimize,
    });

    const lib = b.addLibrary(.{
        .linkage = .static,
        .name = "simple",
        .root_module = lib_mod,
    });

    b.installArtifact(lib);

    const exe_mod = b.createModule(.{
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    exe_mod.addImport("simple_lib", lib_mod);

    const exe = b.addExecutable(.{
        .name = "simple",
        .root_module = exe_mod,
    });

    b.installArtifact(exe);
}
```

Мы добавили вызов двух методов сборочной системы — `standardTargetOptions` и `standardOptimizeOption`. В результате:
*	`standardOptimizeOption` вернул стандартный уровень оптимизации со значением `Debug`.
*	`standardTargetOptions` вернул параметры целевой платформы, совпадающей с системой, на которой выполняется сборка.

Давайте запустим сборку приложения и убедимся, что все работает как и раньше:

```
$ zig build --summary all
Build Summary: 5/5 steps succeeded
install success
├─ install simple success
│  └─ zig build-lib simple Debug native success 713ms MaxRSS:206M
└─ install simple success
   └─ zig build-exe simple Debug native success 855ms MaxRSS:236M
```

Мы научились собирать наше приложение, но что если мы хотим еще и запускать его по результату сборки. Это очень удобно когда вы можете скомпилировать и запустить приложение всего одной командой `zig build run`. Для того чтобы добавить в наш срипт сборки шаг запуска приложения нам необходимо сделать две вещи - добавить Run артефакт, через вызов метода `addRunArtifact` и добавить шаг запуска приложения, через вызов метода `step`. Давайте внесем изменения в наш скрипт сборки:

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const lib_mod = b.createModule(.{
        .root_source_file = b.path("src/root.zig"),
        .target = target,
        .optimize = optimize,
    });

    const lib = b.addLibrary(.{
        .linkage = .static,
        .name = "simple",
        .root_module = lib_mod,
    });

    b.installArtifact(lib);

    const exe_mod = b.createModule(.{
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    exe_mod.addImport("simple_lib", lib_mod);

    const exe = b.addExecutable(.{
        .name = "simple",
        .root_module = exe_mod,
        .version = .{ .major = 1, .minor = 0, .patch = 0 },
    });

    b.installArtifact(exe);

    // Добавляем исполняемый файл как артефакт для запуска
    const run_cmd = b.addRunArtifact(exe);

    // Пробрасываем аргументы командной строки в исполняемый файл
    if (b.args) |args| {
        run_cmd.addArgs(args);
    }

    // Добавляем шаг запуска приложения
    const run_step = b.step("run", "Run the app");
    run_step.dependOn(&run_cmd.step);
}
```

Я прокоментировал все изменения, которые мы внесли в наш скрипт сборки. В целом там нет ничего сложного. Теперь если мы выполним команду `zig build run`, то наше приложение скомпилируется и запустится:

```
$ zig build run --summary all
All your codebase are belong to us.
Run `zig build test` to run the tests.
Build Summary: 3/3 steps succeeded
run success
└─ run simple success 231ms MaxRSS:1M
   └─ zig build-exe simple Debug native success 868ms MaxRSS:228M
```

Если вы хотите увидеть доступные шаги сборки вашего приложения, то вы можете выполнить команду `zig build --help`:

```
$ zig build --help
Usage: /Users/roman/.zvm/0.14.0/zig build [steps] [options]

Steps:
  install (default)            Copy build artifacts to prefix path
  uninstall                    Remove build artifacts from prefix path
  run                          Run the app

General Options:
...
```

Как мы видим наш шаг запуска приложения перечислен среди списка доступных шагов сборки и содержит описание, которое мы указали при создании шага.

## Динамические библиотеки
Давайте предположим, что мы хотим чтобы наша библиотека подключалась к приложению не статически, а динамически. Чтобы мы могли загрузить ее с диска и вызвать из нее нужную нам функцию. Для этого нам нужно создать динамическую библиотеку с расширением `.so`, `.dll` или `.dylib` в зависимости от операционной системы. Давайте рассмотрим как изменится наш скрипт сборки:

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const lib_mod = b.createModule(.{
        .root_source_file = b.path("src/root.zig"),
        .target = target,
        .optimize = optimize,
    });

    const lib = b.addLibrary(.{
        .linkage = .dynamic, // Мы меняем тип библиотеки с статической на динамическую
        .name = "simple",
        .root_module = lib_mod,
    });

    b.installArtifact(lib);

    const exe_mod = b.createModule(.{
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    const exe = b.addExecutable(.{
        .name = "simple",
        .root_module = exe_mod,
    });

    b.installArtifact(exe);
}
```

Скрипт сборки претерпел незначительные изменения: мы изменили тип библиотеки с статической на динамическую и убрали явный импорт этой библиотеки в исполняемый файл.

Зато исходный код как самой библиотеки, так и исполняемого файла был существенно переработан — в него внесены важные изменения, необходимые для корректной работы с динамической библиотекой:

```zig
// Файл root.zig
const std = @import("std");

export fn add(a: i32, b: i32) callconv(.C) i32 {
    return a + b;
}
```

Как мы видим мы добавили задание конвенцию вызова функции как C, а в остальном код нашей библиотеки не изменился. Давайте теперь рассмотрим код нашего основного файла:

```zig
// Файл main.zig
const std = @import("std");
const builtin = @import("builtin");

var Add: *const fn (a: i32, b: i32) callconv(.C) i32 = undefined;

pub fn main() !void {
    const libname = switch (builtin.target.os.tag) {
        .linux => "./zig-out/lib/libsimple.so",
        .windows => "./zig-out/lib/libsimple.dll",
        .macos => "./zig-out/lib/libsimple.dylib",
        else => @compileError("Unsupported platform " ++ @tagName(builtin.target.os.tag)),
    };

    var lib = try std.DynLib.open(libname);
    defer lib.close();

    Add = lib.lookup(@TypeOf(Add), "add") orelse return error.NoSuchSymbol;
    const result = Add(1, 2);

    std.debug.print("add result: {d}\n", .{result});
}
```

Для того, чтобы использовать функцию из динамической библиотеки, мы должны сначала загрузить библиотеку используя функцию `std.DynLib.open`. Причем в зависимости от версии ОС нам необходимо загружать библиотеку с соответствующим расширением. Например, для Linux мы загружаем библиотеку с расширением `.so`, для Windows - с расширением `.dll`, а для macOS - с расширением `.dylib`. После загрузки библиотеки мы можем использовать функцию `lookup` для поиска функции в библиотеке. Нам нужно указать тип функции, которую мы ищем, и имя функции. Если функция не найдена, мы возвращаем ошибку `error.NoSuchSymbol`. Если функция найдена, мы присваиваем указатель на функцию переменной `Add`. Дальше мы вызываем функцию `Add` с аргументами `1` и `2` и выводим результат.

Давайте скомпилируем нашу программу и убедимся что она работает:

```zig
$ zig build --summary all
Build Summary: 5/5 steps succeeded
install success
├─ install simple success
│  └─ zig build-lib simple Debug native success 857ms MaxRSS:234M
└─ install simple success
   └─ zig build-exe simple Debug native success 874ms MaxRSS:240M

$ l ./zig-out/lib/libsimple.dylib
-rwxr-xr-x@ 1 roman  staff   1.2M Apr 25 00:35 ./zig-out/lib/libsimple.dylib

$ ./zig-out/bin/simple
add result: 3
```

Как мы видим наша программа успешно работает и мы используем динамическую загрузку нашей библиотеки `libsimple.dylib`.

## Добавление тестов
Теперь давайте рассмотрим, как добавить запуск тестов в наш скрипт сборки проекта. Как уже упоминалось ранее, система сборки Zig предоставляет специальный встроенный метод `addTest`, который позволяет интегрировать выполнение тестов непосредственно в процесс сборки:

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const lib_mod = b.createModule(.{
        .root_source_file = b.path("src/root.zig"),
        .target = target,
        .optimize = optimize,
    });

    const exe_mod = b.createModule(.{
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    exe_mod.addImport("simple_lib", lib_mod);

    const lib = b.addLibrary(.{
        .linkage = .static,
        .name = "simple",
        .root_module = lib_mod,
    });

    b.installArtifact(lib);

    const exe = b.addExecutable(.{
        .name = "simple",
        .root_module = exe_mod,
    });

    b.installArtifact(exe);

    const run_cmd = b.addRunArtifact(exe);

    run_cmd.step.dependOn(b.getInstallStep());

    if (b.args) |args| {
        run_cmd.addArgs(args);
    }

    const run_step = b.step("run", "Run the app");
    run_step.dependOn(&run_cmd.step);

    // Добавляем шаг для запуска тестов библиотеки
    const lib_unit_tests = b.addTest(.{
        .root_module = lib_mod,
    });

    const run_lib_unit_tests = b.addRunArtifact(lib_unit_tests);

    // Добавляем шаг для запуска тестов исполняемого файла
    const exe_unit_tests = b.addTest(.{
        .root_module = exe_mod,
    });

    const run_exe_unit_tests = b.addRunArtifact(exe_unit_tests);

    // Добавляем шаг для запуска тестов библиотеки и исполняемого файла
    const test_step = b.step("test", "Run unit tests");
    test_step.dependOn(&run_lib_unit_tests.step);
    test_step.dependOn(&run_exe_unit_tests.step);
}
```

Большая часть содержимого файла сборки нам уже знакома, однако появилась новая часть, связанная с добавлением и запуском тестов. Давайте разберёмся, как она работает.

Для того чтобы добавить тесты для нашей библиотеки или исполняемого файла, мы начинаем с создания шага тестирования с помощью метода `addTest`. В этот метод мы передаём путь к модулю, для которого хотим запустить тесты. Это может быть как основной исходный файл библиотеки, так и отдельный файл с тестами.

Затем мы создаём шаг выполнения скомпилированного тестового артефакта, используя метод `addRunArtifact`. Он позволяет запустить собранный тестовый бинарник в процессе сборки.

Далее мы объединяем этот шаг в отдельную цель сборки с помощью `b.step`, присваивая ей имя, в нашем случае "test". Это позволяет запускать тесты командой `zig build test`, что делает процесс тестирования частью стандартного рабочего процесса.

Давайте запустим наши тесты и убедимся что наш скрипт работает как ожидается:

```
$ zig build test --summary all
Build Summary: 5/5 steps succeeded; 4/4 tests passed
test success
├─ run test 1 passed 2ms MaxRSS:1M
│  └─ zig test Debug native cached 53ms MaxRSS:36M
└─ run test 3 passed 2ms MaxRSS:1M
   └─ zig test Debug native cached 54ms MaxRSS:36M
```

## Добавление пользовательских параметров
Иногда возникает необходимость параметризировать процесс сборки вашего приложения, чтобы пользователь мог указать определённые параметры, на основании которых будет выбрана нужная конфигурация сборки. Для этого можно использовать параметры командной строки, которые передаются в процесс сборки. Мы уже видели, как это делается для параметров сборки, предоставляемых самой системой сборки. Теперь давайте рассмотрим, как добавить пользовательские параметры сборки.

Для того чтобы добавить пользовательский параметр сборки необходимо использовать метод `option` у системы сборки и передать ему имя параметра сборки, тип параметра и описание, что делает данный параметр для вывода пользователю в команде помощи. Давайте реализуем следующую программу, которая в заивимости от параметра переданного во время сборки будет определать использовать ей собственную реализацию функции или использовать реализацию из статически слинкованной библиотеки. Это конечно искуственный прием, но он поможет нам разобрать ряд интересных моментов использования пользовательских параметров при сборке. давайте начнем с нашего сборочного скрипта:

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    // Добавляем пользовательский параметр сборки
    const use_internal = b.option(bool, "use_internal", "Use internal add implementation") orelse false;

    const exe_mod = b.createModule(.{
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    // Если пользователь указал не использовать реализацию из библиотеки, то не надо ее компилировать и линковать
    if (!use_internal) {
        const lib_mod = b.createModule(.{
            .root_source_file = b.path("src/root.zig"),
            .target = target,
            .optimize = optimize,
        });

        const lib = b.addLibrary(.{
            .linkage = .static,
            .name = "simple",
            .root_module = lib_mod,
        });

        exe_mod.addImport("simple_lib", lib_mod);

        b.installArtifact(lib);
    }

    const exe = b.addExecutable(.{
        .name = "simple",
        .root_module = exe_mod,
    });

    // делаем опции доступными для исполняемого файла
    const options = b.addOptions();
    options.addOption(bool, "use_internal", use_internal);

    exe.root_module.addOptions("config", options);

    b.installArtifact(exe);

    const run_cmd = b.addRunArtifact(exe);

    run_cmd.step.dependOn(b.getInstallStep());

    if (b.args) |args| {
        run_cmd.addArgs(args);
    }

    const run_step = b.step("run", "Run the app");
    run_step.dependOn(&run_cmd.step);
}
```

Давайте подробнее рассмотрим внесённые изменения.

Во-первых, мы добавили пользовательский флаг с помощью метода `b.option`. Этот флаг позволяет передавать в сборочную систему пользовательские параметры, которые можно использовать для управления конфигурацией сборки.

Во-вторых, на основе значения этого флага мы определяем, нужно ли компилировать нашу библиотеку и линковать её с исполняемым файлом. Это удобно, когда, например, вы хотите собирать проект с дополнительными возможностями или без них, в зависимости от потребностей.

Кроме того, поскольку нам необходимо, чтобы исполняемый файл “знал”, с какой конфигурацией он был собран, мы передаём значения пользовательских опций в исполняемый файл с помощью метода `addOptions`. Это позволяет получить доступ к этим параметрам прямо в коде и использовать их для условной логики выполнения.

На этом, собственно, заканчиваются все изменения, которые мы внесли в сборочный скрипт. Теперь давайте посмотрим, как это повлияло на код основного файла проекта.

```zig
const std = @import("std");
const config = @import("config");
const lib = if (config.use_internal == false) @import("simple_lib") else null;

fn add(a: i32, b: i32) i32 {
    return a + b;
}

pub fn main() !void {
    const result = if (config.use_internal == false)
        lib.add(1, 2)
    else
        add(1, 2);

    const used_implementation = if (config.use_internal == false) "external" else "internal";

    std.debug.print("Result for {s} implementation: {d}\n", .{ used_implementation, result });
}
```

В файле основного модуля мы импортируем модуль `config`, который, как мы помним, был сгенерирован сборочным скриптом. В этот модуль мы передали параметры, указанные пользователем при запуске сборки. Таким образом, `config` содержит информацию о том, какие опции были выбраны при конфигурации проекта.

Затем, в зависимости от значения параметра `use_internal`, мы либо импортируем нашу внешнюю библиотеку и используем функцию `add` из неё, либо используем встроенную реализацию этой функции, определённую внутри самого исполняемого файла. Такой подход позволяет гибко управлять логикой работы программы в зависимости от сборочной конфигурации.

В целом, код довольно простой и наглядный. Давайте теперь запустим его и убедимся, что всё работает так, как мы ожидаем.

```
$ zig build run -Duse_internal=true
Result for internal implementation: 3

$ zig build run -Duse_internal=false
Result for external implementation: 3
```

Как мы видим все прекрасно работает и мы либо используем встроенную реализацию либо внешнюю. В следующей главе мы рассмотрим как мы можем компилировать нашу программу для других платформ и архитектур.

## Кросс-компиляция
При публикации вашего продукта часто возникает необходимость собрать его для различных платформ и архитектур. Поскольку сборочный скрипт `build.zig` представляет собой обычный Zig-код, реализация кросс-платформенной сборки не вызывает особых трудностей. Более того, это не требует использования дополнительных инструментов, виртуализации или контейнеров — всё можно сделать средствами самой Zig-системы сборки.

Давайте рассмотрим, как собрать наше простое приложение для наиболее популярных платформ.

```zig
const std = @import("std");

pub fn build(b: *std.Build) !void {
    const optimize = b.standardOptimizeOption(.{});

    const use_internal = b.option(bool, "use_internal", "Use internal add implementation") orelse false;

    const options = b.addOptions();
    options.addOption(bool, "use_internal", use_internal);

    const targets: []const std.Target.Query = &.{
        .{ .cpu_arch = .aarch64, .os_tag = .macos },
        .{ .cpu_arch = .aarch64, .os_tag = .linux },
        .{ .cpu_arch = .x86_64, .os_tag = .linux, .abi = .gnu },
        .{ .cpu_arch = .x86_64, .os_tag = .linux, .abi = .musl },
        .{ .cpu_arch = .x86_64, .os_tag = .windows },
    };

    for (targets) |t| {
        try addBuildForPlatform(b, optimize, t, use_internal, options);
    }
}

fn addBuildForPlatform(
    b: *std.Build,
    optimize: std.builtin.OptimizeMode,
    platform: std.Target.Query,
    use_internal: bool,
    options: *std.Build.Step.Options,
) !void {
    const exe_mod = b.createModule(.{
        .root_source_file = b.path("src/main.zig"),
        .target = b.resolveTargetQuery(platform),
        .optimize = optimize,
    });

    if (!use_internal) {
        const lib_mod = b.createModule(.{
            .root_source_file = b.path("src/root.zig"),
            .target = b.resolveTargetQuery(platform),
            .optimize = optimize,
        });

        exe_mod.addImport("simple_lib", lib_mod);

        const exe = b.addExecutable(.{
            .name = "simple",
            .root_module = exe_mod,
        });

        exe.root_module.addOptions("config", options);

        const target_output = b.addInstallArtifact(exe, .{
            .dest_dir = .{
                .override = .{
                    .custom = try platform.zigTriple(b.allocator),
                },
            },
        });

        b.getInstallStep().dependOn(&target_output.step);
    }
}
```

В нашем примере сборку нашей библиотеки и исполняемого файла мы вынесли в отдельную функцию, в которой использовали функцию `zigTriple` для того чтобы свормировать директорию для артефактов на основе архитектуры и операционной системы. Давайте посмотрим как это работает:

```
$ zig build --summary all
Build Summary: 12/12 steps succeeded
install success
├─ install simple success
│  └─ zig build-exe simple Debug aarch64-macos success 946ms MaxRSS:236M
│     └─ options cached
├─ install simple success
│  └─ zig build-exe simple Debug aarch64-linux success 2s MaxRSS:289M
│     └─ options (reused)
├─ install simple success
│  └─ zig build-exe simple Debug x86_64-linux-gnu success 2s MaxRSS:273M
│     └─ options (reused)
├─ install simple success
│  └─ zig build-exe simple Debug x86_64-linux-musl success 2s MaxRSS:278M
│     └─ options (reused)
└─ install simple success
   └─ zig build-exe simple Debug x86_64-windows success 2s MaxRSS:266M
      └─ options (reused)

l  l ./zig-out
total 0
drwxr-xr-x@ 7 roman  staff   224B Apr 25 16:56 .
drwxr-xr-x  7 roman  staff   224B Apr 25 16:56 ..
drwxr-xr-x@ 3 roman  staff    96B Apr 25 16:56 aarch64-linux
drwxr-xr-x@ 3 roman  staff    96B Apr 25 16:56 aarch64-macos
drwxr-xr-x@ 3 roman  staff    96B Apr 25 16:56 x86_64-linux-gnu
drwxr-xr-x@ 3 roman  staff    96B Apr 25 16:56 x86_64-linux-musl
drwxr-xr-x@ 4 roman  staff   128B Apr 25 16:56 x86_64-windows
```

Как мы видим, система сборки создала отдельную папку для каждой целевой платформы, указанных в нашем скрипте, и сгенерировала соответствующие артефакты. Поскольку сборочный скрипт — это обычный код на языке Zig, реализовать такую многоцелевую сборку оказалось довольно просто.

## Кеширование результатов сборки
Довольно важный момент, который хочется отметить отдельно это кеширование результатов сборки. Когда мы запускаем сборку командой `zig build` в корневой папке проекта создается папка `.zig-cache` где сохраняются результаты каждого этапа сборки. Это позволяет существенно ускорить процесс сборки, особенно при многократных запусках, так как те артефакты сборки, которые не менялись пересобираться не будут и возьмутся из кеша. давайте посмотрим как это работает.

Первый запуск сборки:
```
$ zig build run --summary all
Result for external implementation: 3
Build Summary: 8/8 steps succeeded
run success
└─ run simple success 202ms MaxRSS:1M
   ├─ zig build-exe simple Debug native success 841ms MaxRSS:228M
   │  └─ options success
   └─ install success
      ├─ install simple success
      │  └─ zig build-lib simple Debug native success 710ms MaxRSS:210M
      └─ install simple success
         └─ zig build-exe simple Debug native (+1 more reused dependencies)
```

При первом запуске сблорки проекта наш кеш еще отсутствует и поэтому все артефакты сборки будут собраны заново. После выполнения нашей команды у нас появится папка `.zig-cache` в корневой папке проекта.

Второй запуск сборки:
```
$ zig build run --summary all
Result for external implementation: 3
Build Summary: 8/8 steps succeeded
run success
└─ run simple success 2ms MaxRSS:1M
   ├─ zig build-exe simple Debug native cached 52ms MaxRSS:36M
   │  └─ options cached
   └─ install cached
      ├─ install simple cached
      │  └─ zig build-lib simple Debug native cached 50ms MaxRSS:36M
      └─ install simple cached
         └─ zig build-exe simple Debug native (+1 more reused dependencies)
```

При повторном запуске сборки видно, что все артефакты — такие как библиотека и исполняемый файл — были взяты из кэша, поскольку исходный код не изменился. Теперь давайте изменим код нашего основного модуля `main.zig` добавив туда еще функцию `sub` и снова запустим сборку:

```
$ zig build run --summary all
Result for external implementation: 3
Build Summary: 8/8 steps succeeded
run success
└─ run simple success 199ms MaxRSS:1M
   ├─ zig build-exe simple Debug native success 833ms MaxRSS:231M
   │  └─ options cached
   └─ install success
      ├─ install simple cached
      │  └─ zig build-lib simple Debug native cached 54ms MaxRSS:36M
      └─ install simple success
         └─ zig build-exe simple Debug native (+1 more reused dependencies)
```

Я не привожу здесь код изменений, так как он достаточно тривиален. Как видно из вывода, библиотека была взята из кэша, поскольку её код не изменился. А вот исполняемый файл был пересобран, потому что изменился связанный с ним код или параметры сборки.

## Генерация артифактов при компиляции
Иногда возникает необходимость сгенерировать часть кода во время компиляции приложения. Некоторые языки программирования поддерживают это с помощью встроенных средств метапрограммирования. В Zig, как мы помним, с помощью comptime нельзя создавать файлы на диске.

Однако мы всё же можем генерировать исходные файлы с помощью скрипта сборки. Давайте рассмотрим, как это работает, на простом примере.

Предположим, нам нужно создать файл `version.zig`, в котором будет содержаться информация о версии приложения: номер версии, хеш коммита, дата сборки и т.п. Мы хотим, чтобы этот файл генерировался автоматически во время сборки. Ниже покажем, как это реализовать.

Для начала давайте рассмотрим исходник нашего приложения:

```zig
const std = @import("std");
const version = @import("generated/version.zig");

pub fn main() !void {
    std.debug.print(
        \\My Awesome App
        \\Version: {s}
        \\Commit: {s}
        \\Built: {s}
        \\Zig: {s}
        \\
    , .{ version.version, version.commit_hash, version.build_time, version.zig_version });
}
```

Тут в целом все просто - мы просто импортируем модуль `version.zig`, который будет сгенерирован во время сборки и выводим информацию из него в консоль. Давайте теперь перейдем к нашему скрипту сборки.

```zig
const std = @import("std");

pub fn build(b: *std.Build) !void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const gen_step = b.step("generate-version", "Generate version infos");

    const gen_mod = b.createModule(.{
        .root_source_file = b.path("src/build_helpers/generate_version.zig"),
        .target = target,
        .optimize = optimize,
    });

    const generator = b.addExecutable(.{
        .name = "version-generator",
        .root_module = gen_mod,
    });

    const run_gen = b.addRunArtifact(generator);
    const output_file = run_gen.addOutputFileArg("version.zig");
    const write_files = b.addUpdateSourceFiles();
    write_files.addCopyFileToSource(output_file, "src/generated/version.zig");

    gen_step.dependOn(&run_gen.step);

    write_files.step.dependOn(gen_step);

    const exe_mod = b.createModule(.{
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    const exe = b.addExecutable(.{
        .name = "simple",
        .root_module = exe_mod,
    });

    exe.step.dependOn(&write_files.step);
    b.installArtifact(exe);

    const run_cmd = b.addRunArtifact(exe);
    run_cmd.step.dependOn(b.getInstallStep());

    if (b.args) |args| {
        run_cmd.addArgs(args);
    }

    const run_step = b.step("run", "Run the app");
    run_step.dependOn(&run_cmd.step);
}
```

Итак, давайте разберёмся, как устроен скрипт сборки.

Первым делом мы добавили новый шаг генерации файла с информацией о версии, используя метод `step`. Затем с помощью метода `addExecutable` мы подключили наш скрипт-генератор, указав путь к файлу c кодом нашего генератора, который отвечает за создание `version.zig`.

После этого мы добавили генератор в сборочный процесс как шаг выполнения (run step) с помощью метода `addRunArtifact`. Пока всё стандартно и все это мы уже видели и раньше.

Далее мы использовали метод `addOutputFileArg`, чтобы передать имя файла, который должен быть сгенерирован. Эта функция создаёт путь к файлу во временной директории сборки и передаёт его нашему генератору как аргумент командной строки.

Наконец, с помощью методов `addUpdateSourceFiles` и `addCopyFileToSource` мы копируем сгенерированный файл в папку `src/generated`. И это все что нам нужно сделать чтобы добавить генерации в наш скрипт сборки. Теперь осталось рассмотреть наш скрипт генерации:

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    var args = std.process.args();
    _ = args.next();

    const filename = args.next().?;

    try generate(gpa.allocator(), filename);
}

pub fn generate(allocator: std.mem.Allocator, filename: []const u8) !void {
    const version_file = try std.fs.cwd().createFile(filename, .{ .read = true });
    defer version_file.close();

    const git_hash = getGitHash(allocator) catch "unknown";
    const build_time = getBuildTime();

    try version_file.writer().print(
        \\// AUTOGENERATED - DO NOT EDIT
        \\pub const version = "{s}";
        \\pub const commit_hash = "{s}";
        \\pub const build_time = "{s}";
        \\pub const zig_version = "{s}";
        \\
    , .{
        "0.1.0", // Можно брать из build.zig.zon
        git_hash,
        build_time,
        @import("builtin").zig_version_string,
    });
}

fn getGitHash(allocator: std.mem.Allocator) ![]const u8 {
    var child = std.process.Child.init(&.{ "git", "rev-parse", "--short", "HEAD" }, allocator);
    child.stdout_behavior = .Pipe;
    child.stderr_behavior = .Pipe;

    var stdout: std.ArrayListUnmanaged(u8) = .empty;
    defer stdout.deinit(allocator);
    var stderr: std.ArrayListUnmanaged(u8) = .empty;
    defer stderr.deinit(allocator);

    try child.spawn();
    try child.collectOutput(allocator, &stdout, &stderr, 1024);
    _ = try child.wait();

    return std.mem.trim(u8, stdout.items, " \n");
}

fn getBuildTime() []const u8 {
    const now = std.time.timestamp();
    return std.fmt.allocPrint(std.heap.page_allocator, "{d}", .{now}) catch "unknown";
}
```

Наш скрипт генерации — это обычный исполняемый файл с классической точкой входа в виде функции `main`. Внутри этой функции мы обрабатываем параметры командной строки, извлекаем из них путь к файлу, который должен быть сгенерирован скриптом сборки, и записываем в него информацию о версии приложения, хэше коммита, времени сборки и версии Zig.

Мы не будем подробно рассматривать сам код генерации этих данных — сейчас он не так важен.

Теперь давайте запустим скрипт сборки и проверим, всё ли работает как ожидается.

```
$ zig build --summary all
Build Summary: 7/7 steps succeeded
install success
└─ install simple success
   └─ zig build-exe simple Debug native success 839ms MaxRSS:235M
      └─ UpdateSourceFiles success
         ├─ run version-generator (version.zig) success 187ms MaxRSS:4M
         │  └─ zig build-exe version-generator Debug native success 1s MaxRSS:245M
         └─ generate-version success
            └─ run version-generator (version.zig) (+1 more reused dependencies)

$ l ./src/generated/version.zig
-rw-r--r--@ 1 roman  staff   159B Apr 26 01:16 ./src/generated/version.zig

$ ./zig-out/bin/simple
My Awesome App
Version: 0.1.0
Commit:
Built: 1745619364
Zig: 0.14.0
```

Как мы видим наш файл с версией успешно сгенерирован и помещен в папку `src/generated`. Также наше приложение работает как ожидается и выводит информацию о версии приложения.

## Управление зависимостями
Нам осталось рассмотреть ещё одну важную тему — управление зависимостями вашего приложения.

Когда вы создаёте новый проект с помощью команды `zig init`, помимо файла `build.zig` в папке проекта появляется также файл `build.zig.zon`. Этот файл содержит описание проекта и его зависимостей, и именно через него осуществляется управление зависимостями вашего проекта или публикация вашего проекта как отдельного пакета.

Давайте посмотрим, что он содержит:

```zig
.{
    .name = .simple,

    .version = "0.0.0",

    .fingerprint = 0xc17b3d0257d622fe,

    .minimum_zig_version = "0.14.0",

    .dependencies = .{
    },

    .paths = .{
        "build.zig",
        "build.zig.zon",
        "src",
    },
}
```

Давайте рассмотрим из чего состоит наш файл `build.zig.zon`:

* **name**
  <br>
  Краткое имя вашего проекта. Когда ваш проект добавляется в качестве зависимости к другому проекту это имя будет использовано в качестве ключа в списке зависимостей.
* **version**
  <br>
  Версия вашего проекта в формате семантического версионирования.
* **fingerprint**
  <br>
  Уникальный идентификатор вашего проекта, который генерируется в момент создания проекта и не изменяется.
* **minimum_zig_version**
  <br>
  Минимальная версия Zig, необходимая для сборки вашего проекта.
* **dependencies**
  <br>
  Список зависимостей вашего проекта
* **paths**
  <br>
  Список путей к файлам и папкам вашего проекта, которые должны входить в состав вашего пакета. Только файлы перечисленные в этом массиве будут скачаныв при загрузке вашей библиотеки как пакета.

Как мы видим файл довольно небольшой и содержит не много полей по сравнению например с тем что поддерживает npm конфиг для JS библиотек. Давайте рассмотрим теперь как установить зависимость и как подключить ее потом к проекту. Для того чтобы поставить зависимость необходимо воспользоваться командой `zig fetch <путь к пакету>`. Давайте создадим новый проект и добавим к нему как зависимость библиотеку для создания http сервера `httpz`:

```
$ mkdir http_server
$ cd http_server
$ zig init
$ zig fetch --save git+https://github.com/karlseguin/http.zig#master
```

Если после этих команд мы откроем файл `build/zig.zon` в нашей директории `http_server`, то мы увидим что наша зависимость была успешно добавлена в секцию `dependencies`:

```zig
...
.dependencies = .{
    .httpz = .{
        .url = "git+https://github.com/karlseguin/http.zig?ref=master#163fd691f46e222d3aa9d15831f7128ce55a58bc",
        .hash = "httpz-0.0.0-PNVzrA63BgDRcEWrLJ0p9VGOK5ib-neHf2RN0SoueEMR",
    },
},
...
```

Как мы видим наша зависимость располагается по ключу `httpz` и содержит ссылку на репозиторий GitHub и хеш скачанной версии пакуета, чтобы контролировать целостность и версионировать зависимость.

Давайте теперь рассмотрим как добавить нашу зависимость в проект. Для этого надо открыть файл `build.zig` и добавить следующий код:

```zig
...
const httpz = b.dependency("httpz", .{
    .target = target,
    .optimize = optimize,
});

exe_mod.addImport("httpz", httpz.module("httpz"));
...
```

С помощью метода `dependency` мы добавляем нашу зависимость в проект, главное чтобы имя нашей зависимости совпадало с тем именем, что указан в файле `build.zig.zon`. Дальше мы добавляем нашу зависимость в импорты к нашему модулю используя метод `addImport`. Все это все что нужно чтобы подключить зависимость к нашему проекту. Давайте теперь посомтрим на наш файл `main.zig` и соберем наш проект:

```zig
const std = @import("std");
const httpz = @import("httpz");

const PORT = 8801;

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var server = try httpz.Server(void).init(allocator, .{
        .port = PORT,
        .request = .{
            .max_form_count = 20,
        },
    }, {});
    defer server.deinit();

    defer server.stop();

    std.debug.print("listening http://localhost:{d}/\n", .{PORT});

    try server.listen();
}
```

Для того, чтобы использовать нашу зависимость мы просто импортируем ее в проект. Теперь если запустим наш код, то увидим что наш сервер стартанул и ждес соединения:

```
$ zig build run
listening http://localhost:8801/
```

Чтобы остановить наше приложение нажмите Ctrl + C.

## Заключение
В этой главе мы подробно рассмотрели систему сборки Zig — один из ключевых инструментов, делающих язык привлекательным не только с точки зрения синтаксиса и возможностей, но и с точки зрения удобства практического использования.

Мы научились работать с самой системой сборки Zig через скрипт `build.zig`, начиная с базовых примеров для компиляции исполняемых файлов и статических библиотек и заканчивая созданием динамических библиотек и конфигурированием их загрузки в рантайме.

Мы также рассмотрели важные аспекты девелоперского процесса, такие как тестирование, передача пользовательских параметров сборки и кросс-компиляция для различных платформ. Эти функции делают Zig пригодным для серьезной промышленной разработки, где часто требуется поддержка множества операционных систем и архитектур.

Наконец, мы рассмотрели, как использовать систему сборки для генерации исходных файлов во время компиляции — механизм, полезный для включения в проект информации о версии, конфигурации или других данных, которые должны определяться в момент сборки.

Система сборки Zig, хотя и является относительно новой, уже сейчас предоставляет достаточно инструментов для организации сложных проектов. Она позволяет разработчикам фокусироваться на решении бизнес-задач, а не на настройке и поддержке сценариев сборки.
