---
title: Взаимодействие с C
date: 2025-04-20 15:00:00
showTableOfContents: true
showComments: true
katex: true
tags:
  - zig
  - zigbook
  - c
  - build
---

Одним из ключевых преимуществ языка Zig является его простота и эффективность при взаимодействии с кодом, написанным на C. Это делает Zig отличным выбором как для постепенной миграции существующих проектов с C, так и для интеграции с уже проверенными и широко используемыми C-библиотеками. Благодаря этому вы можете использовать Zig в реальных проектах, не начиная всё с нуля.

Вот несколько вариантов использования кода на C в Zig:

* Компиляция C-кода вместе с Zig
* Использование C-библиотек
* Создание библиотек, совместимых с C

Хотя взаимодействие с C-кодом не является уникальной возможностью Zig — большинство языков реализуют такую поддержку через механизм FFI (Foreign Function Interface) — Zig предлагает особенно удобные и мощные средства для такой интеграции. Он позволяет подключать C-код напрямую, без необходимости писать промежуточный код на C или использовать отдельные инструменты для связывания.

Для успешной работы с C-кодом на уровне бинарного взаимодействия необходимо соблюдать требования стандартного ABI (Application Binary Interface) — набора правил, определяющих, как функции и данные взаимодействуют на уровне машинного кода. Чтобы Zig-код мог корректно работать с функциями и структурами из C, необходимо учитывать следующие аспекты:

*	Расположение типов данных в памяти — типы должны иметь те же размеры и выравнивание, что и в C. Это гарантирует корректную интерпретацию данных при передаче между языками.
*	Именование функций и переменных (name mangling) — должно быть совместимо с тем, как C-компиляторы формируют имена символов. Zig использует C-совместимое именование при использовании extern.
*	Соглашение о вызовах (calling convention) — способ передачи аргументов в функцию и возврата значений должен совпадать с тем, что ожидает C.

Если все эти условия выполнены, вы сможете без проблем вызывать C-функции из Zig, использовать C-структуры и даже экспортировать Zig-функции, чтобы они были доступны из C-кода.

Давайте теперь подробнее рассмотрим каждый аспект соответсвия с-ABI со стороны Zig.

## Соответствие типов данных
Чтобы наш код на Zig мог корректно взаимодействовать с кодом на C, в первую очередь необходимо обеспечить совместимость типов данных. Ведь именно через них осуществляется обмен информацией между языками. Если типы данных не совпадают по размеру, выравниванию или представлению в памяти, это может привести к неожиданным ошибкам на этапе выполнения или, что ещё хуже, к тихим багам.

Zig предоставляет специальный набор типов данных, которые по своей структуре и поведению полностью совместимы с C:

| C тип              | Zig тип     | Описание                           |
|--------------------|-------------|------------------------------------|
| int                | c_int       | 32-битное целое число              |
| char               | c_char      | 8-битный символ                    |
| unsigned int       | c_uint      | 32-битное беззнаковое целое число  |
| long               | c_long      | 64-битное целое число              |
| unsigned long      | c_ulong     | 64-битное беззнаковое целое число  |
| long long          | c_longlong  | 64-битное целое число              |
| unsigned long long | c_ulonglong | 64-битное беззнаковое целое число  |
| void               | anyopaque   | Пустой тип                         |
| float              | c_float     | 32-битное число с плавающей точкой |
| double             | c_double    | 64-битное число с плавающей точкой |

Несмотря на то что Zig предоставляет отдельный набор типов, совместимых с C, при использовании примитивных типов данных компилятор Zig автоматически приводит их к соответствующим C-типам при необходимости. Это означает, что вам не нужно вручную заботиться о приведении типов при передаче данных в C-функции — Zig сам позаботится о корректном соответствии по размеру, выравниванию и представлению данных.

Благодаря этому взаимодействие с C-кодом на уровне примитивных типов становится простым и прозрачным, что значительно упрощает интеграцию между языками.

Однако, как только мы выходим за рамки примитивных типов, ситуация становится немного сложнее. Особенно это касается таких типов, как срезы (slices), строки и структуры (structs).

Когда мы рассматривали строки и срезы в предыдущих главах, мы упоминали, что Zig поддерживает строки в стиле C — то есть null-терминированные массивы символов. Более того, Zig умеет автоматически приводить строковые литералы как к строкам в формате Zig ([]const u8), так и к C-строкам ([*:0]const u8), когда это необходимо.

Рассмотрим следующий пример, демонстрирующий, как происходит такое приведение строк при вызове C-функции:

```zig
const std = @import("std");

pub fn main() void {
    _ = std.c.printf("Привет, мир!");
}
```

В этом примере строковый литерал "Привет, мир!" используется как обычная строка Zig ([]const u8), но при передаче в функцию `printf` он автоматически преобразуется в C-строку ([*:0]const u8), которую `printf` и ожидает.

А теперь давайте немного изменим наш пример и посмотрим, что произойдёт, если мы попробуем передать в `printf` строку в стиле Zig напрямую:

```zig
const std = @import("std");

pub fn main() void {
    const zigString: []const u8 = "Привет, мир!";
    _ = std.c.printf(zigString);
}
```

Если мы запустим данный код мы получим следующий вывод:

```
$ zig build run
run
└─ run simple
   └─ zig build-exe simple Debug native 1 errors
src/main.zig:8:18: error: expected type '[*c]const u8', found '[]const u8'
    _ = c.printf(zigString);
                 ^~~~~~~~~
```

Как мы видим, компиляция завершилась с ошибкой: наши типы данных не совпадают — функция ожидает строку в стиле C ([*c]const u8), а мы пытаемся передать ей строку в стиле Zig ([]const u8).

Zig намеренно избегает неявных преобразований типов и вместо этого сообщает о несоответствии на этапе компиляции, ожидая, что разработчик сам явно устранит ошибку.

Чтобы исправить эту проблему, нам нужно вручную привести один тип к другому — в данном случае, например, с помощью функции `@ptrCast`:

```zig
const std = @import("std");

pub fn main() void {
    const zigString: []const u8 = "Привет, мир!";
    _ = std.c.printf(@ptrCast(zigString));
}
```

Теперь наш пример успешно компилируется и работает. При необходимости передачи массива из Zig в C вы можете действовать аналогичным образом.

Теперь давайте рассмотрим передачу более сложных типов данных, таких как структуры. На самом деле, здесь всё довольно просто. В главе про выравнивание мы уже упоминали использование внешних (extern) структур, которые можно применять для обмена данными между Zig и C.

Давайте вспомним как создаются такие структуры:

```zig
// Структура, совместимая с C
export const Point = extern struct {
    x: c_int,
    y: c_int,
};

export fn create_point(x: c_int, y: c_int) Point {
    return Point{ .x = x, .y = y };
}
```

Для того чтобы создать структуру, совместимую с C, достаточно просто пометить её как `extern` и использовать в ней типы данных, совместимые с C.

Префикс `extern`, указанный перед определением структуры, изменяет правила выравнивания и порядок размещения полей в соответствии с соглашениями C, что обеспечивает корректную совместимость на уровне ABI.

В нашем примере мы создаём структуру точки (Point) в двумерном пространстве и определяем функцию, совместимую с C, которая возвращает экземпляр этой структуры.

Но что если мы хотим использовать в коде на Zig структуру определенную в C коде. Давайте рассмотрим уже знакомую нам структуру пользователя (User) с двумя полями `id` и `name` и на этот раз будем использовать структуру `User` определенную в C коде. Для начала давайте рассмотрим как выглядит наша структура:

```c
typedef struct {
    int id;
    char name[100];
} User;
```

Как мы видим `id` имеет тип данных целого числа, а `name` - массив символов длиной 100, что по сути является строкой длиной в 100 символов. Теперь давайте используем нашу структуру в коде на Zig:

```zig
const std = @import("std");
const c = @cImport({
    @cInclude("user.h");
});

pub fn main() !void {
    var gpa = std.heap.DebugAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var user: c.User = undefined;
    user.id = 1;

    var name = try allocator.alloc(u8, 6);
    defer allocator.free(name);

    @memcpy(name[0..(name.len - 1)], "alice");
    name[name.len - 1] = 0;

    const copy_len = @min(name.len, 100);
    @memcpy(user.name[0..copy_len], name[0..copy_len]);

    _ = std.c.printf("User ID: %d\n", user.id);
    _ = std.c.printf("User Name: %s\n", &user.name);
}
```

Первое, что появилось нового в нашем коде — это использование директив `@cImport` и `@cInclude`. Блок `@cImport` предназначен для подключения C-кода в программу на Zig. По сути, `@cImport` открывает специальный блок, внутри которого можно использовать такие директивы, как `@cInclude` для подключения заголовочных файлов и `@cDefine` для задания C-препроцессорных макросов.

Однако просто импортировать заголовочный файл `user.h` недостаточно — компилятор Zig не сможет собрать проект, если не будет знать, где искать этот файл.

Чтобы компиляция прошла успешно, необходимо явно указать путь к папке, в которой находится `user.h`. Для этого в файл `build.zig` нужно добавить всего одну строку:

```zig
...
const exe_mod = b.createModule(.{
    .root_source_file = b.path("src/main.zig"),
    .target = target,
    .optimize = optimize,
});

exe_mod.addIncludePath(b.path("src")); // Добавляем вот эту строчку
...
```

Эта строка добавляет папку с исходным кодом в список путей для поиска заголовочных файлов. После этого компилятор сможет найти `user.h`, и сборка пройдет успешно.

Теперь давайте запустим наш код — и, если всё настроено правильно, мы увидим, что программа работает как ожидалось.

```
$ zig build run
User ID: 1
User Name: alice
```

Как видно из нашего примера, при работе со структурами из C существует ряд нюансов. Например, при объявлении структуры мы присваиваем ей значение `undefined`, чтобы указать, что память под эту структуру не инициализирована и может содержать произвольные данные.

Затем мы вручную заполняем поля структуры. В частности, для строкового поля (имени) нам необходимо самостоятельно выделить память и скопировать в неё данные из строки, поскольку C-структуры, как правило, не работают со срезами Zig — они ожидают указатель на заранее выделенный буфер. Кроме того, при создании C-строки важно не забыть вручную добавить нулевой байт (0) в конец строки, чтобы она соответствовала формату null-terminated, принятому в языке C.

В целом, при работе с C-структурами в Zig важно быть особенно внимательным: нужно учитывать особенности управления памятью, ручную инициализацию полей, а также соответствие типов и выравнивания. Однако вам довольно редко прийдется инициализировать вручную С-структуры, так как обычно в C коде есть специальные функции, которые сами инициализируют все объекты за вас. Давайте рассмотрим пример:

```zig
const c = @cImport({
    @cInclude("time.h");
    @cInclude("stdio.h");
});

pub fn main() void {
    var tm: c.struct_tm = undefined;
    var time_val: c.time_t = undefined;

    // Get the current time and store it in time_val
    _ = c.time(&time_val);

    // Pass the address of time_val to localtime_r
    _ = c.localtime_r(&time_val, &tm);

    // Print full time data
    _ = c.printf("Date: %d-%02d-%02d\n", tm.tm_year + 1900, // Year (tm_year is years since 1900)
        tm.tm_mon + 1, // Month (tm_mon is 0-11)
        tm.tm_mday // Day of the month (1-31)
    );

    _ = c.printf("Time: %02d:%02d:%02d\n", tm.tm_hour, // Hour (0-23)
        tm.tm_min, // Minute (0-59)
        tm.tm_sec // Second (0-59)
    );

    // Get weekday name
    const weekday_names = [_][]const u8{ "Sunday", "Monday", "Tuesday", "Wednesday", "Thursday", "Friday", "Saturday" };
    const weekday = weekday_names[@intCast(tm.tm_wday)];

    _ = c.printf("Weekday: %s\n", weekday.ptr);

    // Print day of year (0-365)
    _ = c.printf("Day of year: %d\n", tm.tm_yday + 1); // tm_yday is 0-365
}
```

Данный код выведет:

```
Date: 2025-04-22
Time: 09:51:42
Weekday: Tuesday
Day of year: 112
```

Как мы видим для инициализации структур с текущей датой и временем мы использовали C функции, такие как `time`, `localtime_r`.

## Соглашение о вызовах
Мы рассмотрели как передавать типы данных между Zig и C чтобы они были совместимы, теперь нам осталось рассмотреть как вызывать функции в совместимом с С соглашении. Нам может потребоваться передать в функцию на языке C функцию из языка Zig. Давайте напишем пример такой задачи и потом рассмотрим как же обеспечивается совместимость вызова функций:

```zig
const std = @import("std");

const c = @cImport({
    @cInclude("stdlib.h");
    @cInclude("string.h");
});

fn compare(a: ?*const anyopaque, b: ?*const anyopaque) callconv(.C) c_int {
    const str_a: [*:0]const u8 = @ptrCast(@alignCast(a.?));
    const str_b: [*:0]const u8 = @ptrCast(@alignCast(b.?));
    return c.strcmp(str_a, str_b);
}

pub fn main() void {
    const strings = [_][*:0]const u8{ "hello", "world", "zig", "abc" };

    c.qsort(
        @ptrCast(@constCast(&strings)),
        strings.len,
        @sizeOf([*:0]const u8),
        compare,
    );

    for (strings) |s| {
        _ = std.c.printf("%s\n", s);
    }
}
```

В нашем примере мы используем функцию быстрой сортировки (qsort) из стандартной библиотеки C для сортировки массива строк.

Функция `qsort` требует на вход указатель на функцию сравнения, которая должна принимать два указателя на элементы массива и возвращать значение типа `c_int`. В нашем случае функция `compare` принимает два указателя на неизвестный тип (`anyopaque`) и использует `strcmp` из стандартной библиотеки C для сравнения строк.

Чтобы функция `compare` была совместима с вызовом из C-кода, мы явно указываем соглашение о вызовах в стиле C с помощью `callconv(.C)`. Внутри нашей функции мы явно приводим указатели к типу `[*:0]const u8` и используем `strcmp` для сравнения строк.

Мы уже рассмотрели, как обеспечить соответствие соглашениям ABI при взаимодействии C-кода и кода на Zig. Теперь давайте подробнее разберёмся, как происходит импорт и компиляция C-кода в Zig.

## Сборка C кода
Давайте начнем с того, что разберемся как с помощью компилятора Zig собирать C код. Zig включает в себя встроенный компилятор C, что позволяет напрямую компилировать и запускать C-программы без необходимости устанавливать сторонние инструменты, такие как GCC или Clang. Это особенно удобно при переносе проектов или для быстрой компиляции отдельных C-файлов.

Рассмотрим следующий простой пример на C, и попробуем скомпилировать его с помощью компилятора Zig:

```c
// Файл math.h
#ifndef MATH_H
#define MATH_H

#ifndef STEP_BY
  #define STEP_BY 2
#endif

int* range(int start, int end, int* length);

#endif
```

```c
// Файл math.c
#include <stdlib.h>
#include "math.h"

int* range(int start, int end, int* length) {
  if (STEP_BY == 0) {
      *length = 0;
      return NULL;
  }

  int len = 0;
  if ((STEP_BY > 0 && start <= end) || (STEP_BY < 0 && start >= end)) {
      len = ((end - start) / STEP_BY) + 1;
  }

  int* arr = (int*)malloc(len * sizeof(int));
  if (arr == NULL) {
      *length = 0;
      return NULL;
  }

  for (int i = 0; i < len; i++) {
      arr[i] = start + (i * STEP_BY);
  }

  *length = len;
  return arr;
}
```

```c
// Файл main.c
#include <stdio.h>
#include <stdlib.h>
#include "math.h"

int main() {
  int start = 10;
  int end = 20;
  int length = 0;

  int* result = range(start, end, &length);

  if (result != NULL) {
      printf("Range from %d to %d with step %d:\n", start, end, STEP_BY);
      for (int i = 0; i < length; i++) {
          printf("%d ", result[i]);
      }
      printf("\n");

      free(result);
  } else {
      printf("Failed to create range or invalid parameters.\n");
  }

  return 0;
}
```

Наш код определяет функцию `range`, которая создает массив целых чисел в заданном диапазоне с заданным шагом. Функция возвращает указатель на массив и его длину. Далее мы выводим получившийся массив пользователю. Давайте теперь скомпилируем и запустим нашу программу:

```
$ zig cc ./src/math.c ./src/main.c -o math
$ ./math
```

В результате мы увидим следующий вывод:

```
Range from 10 to 20 with step 2:
10 12 14 16 18 20
```

Как мы видим компилятор Zig позволяет нам успешно компилировать C код. Если вам необходимо собрать код под конкретную платформу или архитектуру, вы можете использовать опции компилятора Zig для этого:

```
zig cc ./src/math.c ./src/main.c -o math -target x86_64-linux-gnu
```

Эта команда соберёт нашу программу для архитектуры x86_64 под Linux. Однако гораздо удобнее интегрировать сборку C-кода прямо в сборочную систему Zig, чтобы не приходилось каждый раз вручную указывать опции компилятора.

Такой подход особенно полезен в проектах, где используются как Zig-, так и C-файлы, а также при необходимости подключения внешних заголовочных файлов или C-библиотек. Использование файла `build.zig` позволяет централизованно управлять сборочным процессом, упростить настройку зависимостей и повысить переносимость проекта между различными платформами.

Пример конфигурации сборки с использованием `build.zig` может выглядеть следующим образом:

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe_mod = b.addModule("main", .{
        .target = target,
        .optimize = optimize,
        .link_libc = true, // добавляем поддержку libc
    });

    exe_mod.addIncludePath(b.path("src"));
    exe_mod.addCSourceFiles(.{ .files = &.{ "src/main.c", "src/math.c" }, .flags = &.{"-std=c11"} });

    const exe = b.addExecutable(.{
        .name = "math",
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
}
```

Теперь если мы выполним команду `zig build run`, то наша программа успешно соберется и запустится.

## Импортирование C-кода
Теперь давайте рассмотрим, каким образом можно импортировать C-код в программу, написанную на языке Zig. Для этого мы воспользуемся уже готовым кодом на языке C и перепишем файл `main.c`, реализовав его функциональность на Zig.

В результате у нас получится файл `main.zig`, в котором будет производиться вызов функций, определённых в C-коде. Ниже представлен финальный вид этого файла:

```zig
const std = @import("std");

const c = @cImport({
    @cInclude("stdlib.h");
    @cInclude("math.c");
});

pub fn main() !void {
    const start: i32 = 10;
    const end: i32 = 20;
    var length: i32 = 0;

    const result = c.range(start, end, &length);
    if (result == null) {
        std.debug.print("Failed to create range or invalid parameters.\n", .{});
    } else {
        std.debug.print("Range from {d} to {d} with step {d}:\n", .{ start, end, c.STEP_BY });

        const result_slice = result[0..@intCast(length)];

        for (result_slice) |value| {
            std.debug.print("{d} ", .{value});
        }
        std.debug.print("\n", .{});

        c.free(result);
    }
}
```

Первое, что необходимо сделать при использовании C-кода в программе на Zig — это импортировать его. Для этого в Zig предусмотрена специальная директива `@cImport`, которая позволяет подключать C-заголовочные файлы и использовать C-функции непосредственно в Zig-коде.

Как уже упоминалось внутри блока `@cImport` можно использовать директиву `@cInclude` для подключения конкретных заголовочных файлов. Это аналогично директиве `#include` в языке C.

В нашем случае необходимо подключить заголовочный файл `math.h`, поскольку мы планируем использовать определённую в нём функцию `range`. Также нам нужно подключить `stdlib.h`, так как в программе присутствует работа с динамически выделенной памятью, которую нужно освободить с помощью функции `free`.

Дальше у нас происходит вызов нашей функции `range` из заголовочного файла `math.h` и преобразование результата в массив нужной длины. После вывода результат пользователю, нам необходимо освободить выделенную память с помощью функции `free`.

Для того, чтобы скомпилировать и запустить нашу программу нам необходимо внести изменения в файл `build.zig`, добавив в него соответствующие команды для компиляции и запуска программы:

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe_mod = b.createModule(.{
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    exe_mod.addIncludePath(b.path("src"));

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

    const exe_unit_tests = b.addTest(.{
        .root_module = exe_mod,
    });

    const run_exe_unit_tests = b.addRunArtifact(exe_unit_tests);

    const test_step = b.step("test", "Run unit tests");
    test_step.dependOn(&run_exe_unit_tests.step);
}
```

Наш сборочный файл изменился незначительно: теперь вместо сборки файла `main.c` мы указываем путь к файлу `main.zig` в качестве корневого модуля. Однако для успешной сборки программы нам по-прежнему необходимо указать путь к заголовочным файлам на языке C.

Есть еще один момент, который стоит упомянуть, прежде чем мы двинемся дальше. В секции `@cImport` мы можем использовать директиву `@cDefine` для того, чтобы переопределять макросы из заголовочных файлов на языке C. Давайте изменим директиву `@cImport` в нашем файле на следующий и посмотрим как изменится вывод нашей программы:

```zig
const c = @cImport({
    @cInclude("stdlib.h");
    @cDefine("STEP_BY", "4");

    @cInclude("math.c");
});
...
```

Теперь если мы запустим нашу программу, то увидим, что шаг с которым мы итерируемся по числам изменился:

```
Range from 10 to 20 with step 4:
10 14 18
```

## Использование C библиотеки
Еще один из вариантов использования C библиотеки в Zig - это использование функций из статической библиотеки. Для того чтобы продемонстрировать как работать со статическими библиотеками нам необходимо изменить два файла - файл сборки и наш файл `main.zig`. Давайте начнем с файла сборки `build.zig`:

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const lib = b.addStaticLibrary(.{
        .name = "math",
        .target = target,
        .optimize = optimize,
        .link_libc = true,
    });

    lib.addCSourceFiles(.{ .files = &.{"src/math.c"}, .flags = &.{"-std=c11"} });
    lib.addIncludePath(b.path("src"));

    b.installArtifact(lib);

    const exe_mod = b.createModule(.{
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
        .link_libc = true,
    });

    const exe = b.addExecutable(.{
        .name = "simple",
        .root_module = exe_mod,
    });

    exe.linkLibrary(lib);

    b.installArtifact(exe);

    const run_cmd = b.addRunArtifact(exe);

    run_cmd.step.dependOn(b.getInstallStep());

    if (b.args) |args| {
        run_cmd.addArgs(args);
    }

    const run_step = b.step("run", "Run the app");
    run_step.dependOn(&run_cmd.step);

    const exe_unit_tests = b.addTest(.{
        .root_module = exe_mod,
    });

    const run_exe_unit_tests = b.addRunArtifact(exe_unit_tests);

    const test_step = b.step("test", "Run unit tests");
    test_step.dependOn(&run_exe_unit_tests.step);
}
```

В файл сборки мы внесли следующие изменения. Сначала добавляется сборка нашей статической библиотеки с использованием возможностей Zig по сборке C-кода. Затем, при сборке основного приложения, мы подключаем (линкуем) эту статическую библиотеку с помощью функции `linkLibrary`. И это все изменения, которые нужно сделать в файле `build.zig`. Теперь перейдем к тому как будет выглядеть наш файл `main.zig`:

```zig
const std = @import("std");

extern fn range(start: i32, end: i32, length: *i32) ?[*]i32;

pub fn main() !void {
    const start: i32 = 10;
    const end: i32 = 20;
    var length: i32 = 0;

    const result = range(start, end, &length);
    if (result == null) {
        std.debug.print("Failed to create range or invalid parameters.\n", .{});
    } else {
        std.debug.print("Range from {d} to {d} with step {d}:\n", .{ start, end, 2 });

        const result_slice = result.?[0..@intCast(length)];

        for (result_slice) |value| {
            std.debug.print("{d} ", .{value});
        }
        std.debug.print("\n", .{});

        std.c.free(result);
    }
}
```

Мы убрали из нашего файла импортирование C кода, и вместо этого описали нашу функцию `range` как внешнюю функцию, используя ключевое слово `extern`.  Остальной код практически не изменился, разве что теперь для освобождения памяти мы используем функцию `std.c.free` вместо `std.free`.

## Испрользование Zig кода в C
Итак нам осталось рассмотреть еще один вариант использования интеграции Zig и кода на языке C - использования Zig библиотеки в C-коде. Для демонстрации этой возможности нам прийдется инвертировать наш пример и перенести нашу функцию `range` в Zig код, а использование ее перенести в C код. Давайте посмотрим сначала как нам реализовать библиотеку на Zig:

```zig
const std = @import("std");

export fn zig_range(start: i32, end: i32, step: i32, len: *usize) [*]i32 {
    if (step == 0) {
        len.* = 0;
        return undefined;
    }

    len.* = if ((step > 0 and start <= end) or (step < 0 and start >= end))
        @intCast(@divTrunc(end - start, step) + 1)
    else
        0;

    const allocator = std.heap.c_allocator;
    const slice = allocator.alloc(i32, len.*) catch {
        len.* = 0;
        return undefined;
    };

    for (slice, 0..) |*val, i| {
        val.* = start + @as(i32, @intCast(i)) * step;
    }

    return slice.ptr;
}

export fn zig_free_range(ptr: [*]i32, len: usize) void {
    const allocator = std.heap.c_allocator;
    const slice = ptr[0..len];
    allocator.free(slice);
}
```

В целом в коде нашей библиотеки на Zig нет ничего такого, чтобы мы не рассматривали ранее. Основной момент это использование ключевого слова `extern` для объявления функций, которые будут использоваться в C-коде. Теперь давайте соберем нашу библиотеку на Zig и для этого нам надо выполнить команду:

```
zig build-lib ./src/ziglib.zig -static -OReleaseSafe
```

В результате у нас появится два файла `libziglib.a` и `libziglib.a.o`. Это наша статическая библиотека, которую мы можем использовать в C-коде. Теперь рассмотрим как будет выглядеть на код на C:

```c
// Файл ziglib.h
#ifndef ZIGLIB_H
#define ZIGLIB_H

#include <stdint.h>

#ifdef __cplusplus
extern "C" {
#endif

int32_t* zig_range(int32_t start, int32_t end, int32_t step, size_t* length);
void zig_free_range(int32_t* array, size_t length);

#ifdef __cplusplus
}
#endif

#endif // ZIGLIB_H
```

```c
// Файл main.c
#include <stdio.h>
#include "ziglib.h"

int main() {
    size_t length;
    int32_t* numbers = zig_range(10, 20, 2, &length);

    if (numbers && length > 0) {
        printf("Range from Zig:\n");
        for (size_t i = 0; i < length; i++) {
            printf("%d ", numbers[i]);
        }
        printf("\n");

        zig_free_range(numbers, length);
    } else {
        printf("Failed to generate range\n");
    }

    return 0;
}
```

Для того чтобы использовать нашу библиотеку мы формируем файл `ziglib.h` в котором описываем сигнатуры наших функций, а дальше просто используем наши функции из статической библиотеки в нашем коде на языке C. Для того чтобы скомпилировать наш проект с помощью gcc необходимо выполнить следующие команду:

```
gcc ./src/main.c -o main -L. -lziglib -I.
```

После этого у нас появится файл main, при запуске которого мы увидим уже знакомый вывод нашей программы.

## Заключение
В этой главе мы рассмотрели различные аспекты взаимодействия между Zig и C, который является одной из сильных сторон языка Zig. Такое взаимодействие позволяет разработчикам использовать огромное количество существующих C-библиотек и постепенно мигрировать с C на Zig, не переписывая весь код сразу.

Важно отметить, что, хотя работа с C-кодом в Zig достаточно проста, она все же требует понимания особенностей взаимодействия на уровне ABI и внимательного отношения к управлению памятью, особенно при работе с C-строками и структурами данных.

В целом, возможность бесшовной интеграции с C является одним из ключевых преимуществ Zig, делающим его практичным выбором для разработки системного программного обеспечения, особенно в проектах, где уже используются компоненты на C или требуется взаимодействие с существующими C-библиотеками.
