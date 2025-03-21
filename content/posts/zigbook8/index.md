---
title: Функции, перечисления и работа с ошибками
date: 2025-03-07 15:00:00
showTableOfContents: true
showComments: true
katex: true
tags:
  - zig
  - zigbook
  - function
  - errors
  - enum
  - union
---

В этой главе мы рассмотрим, как объявлять и использовать функции в Zig, а также изучим их особенности. Во второй части главы мы затронем тему работы с перечислениями и в частности их использование для обратотки ошибок. В Zig функции могут возвращать не только результат вычислений, но и ошибки, если что-то идет не так. Механизм обработки ошибок в Zig имеет сходства с подходом, используемым в языке Rust, но при этом обладает своими уникальными особенностями.

## Функции
Функции — это один из основных строительных блоков в программировании. Они позволяют организовывать код в повторно используемые и модульные блоки, что делает программы более структурированными и удобными для поддержки. Zig не является исключением: в этом языке функции играют ключевую роль, предоставляя разработчикам мощные и гибкие инструменты для работы с кодом. Вы уже видели одну из самых важных функций языка – функцию main, которая является точкой входа многих программ.

### Объявление функций
В Zig функция объявляется с использованием ключевого слова `fn`. Базовая структура функции выглядит следующим образом:

```zig
fn имя_функции(параметры) возвращаемый_тип {
    // Тело функции
}
```

Например, простая функция выглядит так:

```zig
fn some_function(a: i32) {
  std.debug.print("Function param: {d}!\n", .{a});
}
```

Здесь:
* some_function — имя функции.
* a: i32 — параметр функции (32-битное целое число со знаком).

Как мы видим, функции также могут определяться вместе с параметрами, то есть специальными переменными, являющимися частью сигнатуры функции.
Когда у функции есть параметры, вы можете предоставить ей конкретные значения для этих параметров. Технически конкретные значения называются аргументами, но обычно можно встретить употребление термина «параметр», как для переменных в определении функции, так и для конкретных значений, передаваемых при вызове функции.

В сигнатурах функций необходимо объявлять тип каждого параметра. Это решение в дизайне языка является намеренным: обязательные аннотации типов
в определениях функций означают, что компилятор почти никогда не нуждается в том, чтобы выяснить, что вы имеете в виду.

Если вы хотите, чтобы функция имела более одного параметра, то отделите объявления параметров запятыми:

```zig
fn another_function(a: i32, b: i32) {
    std.debug.print("Function param: {d}!\n", .{a});
    std.debug.print("Function param: {d}!\n", .{b});
}
```

Функции могут возвращать значения в вызывающий их код. Мы не даем возвращаемым значениям имена, но объявляем их тип после закрывающей скобки `)`. Так если мы хотим вернуть сумму двух чисел, мы можем написать следующую функцию:

```zig
fn add(a: i32, b: i32) i32 {
    return a + b;
}
```

Для того чтобы вызвать функцию, мы используем ее имя и передаем ей аргументы в круглых скобках. Например, чтобы сложить два числа 3 и 5, мы вызываем функцию `add` следующим образом:

```zig
const result = add(3, 5);
```

При вызове функции, как мы уже знаем из главы про стек, компилятор создаст для нас стековый фрейм, где разместит все локальные переменные функции и переданные ей аргументы. Аргументы функции будут размещены на стеке в порядке справа налево. Для небольших функций компилятор может применить оптимизацию и передать параметры функции через регистры процессора.

### Передача параметров функции
Функции в Zig, как и в других языках программирования, могут принимать параметры, которые передаются в них при вызове. Однако в Zig есть несколько особенностей, связанных с передачей параметров, которые делают этот язык гибким и мощным.

Все параметры переданные в функцию в Zig являются константными и попытка изменить переданную переменную в теле функции приведет к ошибке компиляции. Это обеспечивает безопасность и предотвращает неожиданные изменения данных. Следующий код выведет ошибку компиляции:

```zig
const std = @import("std");

fn modifyValue(x: i32) void {
    x = 10;
    std.debug.print("Внутри функции: {}\n", .{x});
}

pub fn main() void {
    var num: i32 = 5;
    modifyValue(num);

    std.debug.print("Снаружи функции: {}\n", .{num});

    num = 15;
    std.debug.print("Снаружи функции: {}\n", .{num});
}
```

Этот код выведет:

```
run
└─ run simple
   └─ install
      └─ install simple
         └─ zig build-exe simple Debug native 1 errors
src/main.zig:4:5: error: cannot assign to constant
    x = 10;
    ^
```

Если мы хотим изменять переменную внутри функции, то у нас есть два варианта - вернуть новое значение из функции или передать в функцию указатель. Как мы помним из главы про указатели, чтобы передать в функцию указатель надо воспользоваться оператором `&`:

```zig
const std = @import("std");

fn modifyValue(x: *i32) void {
    x.* = 10; // Изменение оригинальной переменной
    std.debug.print("Внутри функции: {}\n", .{x.*});
}

pub fn main() void {
    var num: i32 = 5;
    modifyValue(&num); // Передаём указатель на num
    std.debug.print("Снаружи функции: {}\n", .{num});
}
```

В этом случае мы получим ожидаемый результат:

```
Внутри функции: 10
Снаружи функции: 10
```

В Zig нет поддержки дефолтных значений для параметров функций, но вы можете использовать опциональные параметры в функциях. Например, если мы не всегда хотим передавать какой-то аргумент, мы можем использовать опциональный тип:

```zig
const std = @import("std");

fn greet(name: ?[]const u8) void {
    const n = name orelse "Гость";
    std.debug.print("Привет, {s}!\n", .{n});
}

pub fn main() !void {
    greet(null);
    greet("Алиса");
}
```

Этот код выведет следущее:

```
Привет, Гость!
Привет, Алиса!
```

К сожалению, в Zig нет поддержки передачи в функцию переменного числа параметров, но мы можем использовать массивы или срезы для передачи переменного числа аргументов. Так как массивы и срезы в Zig содержат информацию о размере, Вы можете безопасно обработать такие переменные в теле функции:

```zig
const std = @import("std");

fn sum(numbers: []const i32) i32 {
    var total: i32 = 0;
    for (numbers) |num| {
        total += num;
    }
    return total;
}

pub fn main() !void {
    const result = sum(&[_]i32{ 1, 2, 3, 4 });

    std.debug.print("Сумма чисел: {d}\n", .{result});
}
```

Этот код выведет:

```
Сумма чисел: 10
```

Функции могут возвращать значения любых типов. Как мы уже рассматривали в главе 2 в Zig есть тип `void`, который используется для функций, не возвращающих значения. Например, функция, которая просто печатает сообщение на экран, может быть объявлена следующим образом:

```zig
fn print_message(message: []const u8) void {
    std.debug.print("{}", .{message});
}
```

Здесь:
* print_message — имя функции.
* message: []const u8 — параметр функции, который представляет собой строку, передаваемую в функцию.
* void после списка параметров — это возвращаемый тип функции, который указывает, что функция не возвращает никакого значения.

Второй тип данных, который мы рассматривали в главе 2 был тип `noreturn`. Этот тип используется тогда, когда надо указать, что функция никогда не вернет управление, например, когда функция вызывает `std.process.exit` или `@panic`.

```zig
fn exit_with_message(message: []const u8) noreturn {
    print_message(message);
    std.process.exit(1);
}
```

В данном примере, после вызова функции `print_message`, функция `exit_with_message` завершает выполнение программы с кодом ошибки 1, используя `std.process.exit(1)`. Это гарантирует, что программа завершится после вывода сообщения об ошибке.

```zig
fn print_message(message: []const u8) void {
    std.debug.print("Error: {s}\n", .{message});
}

fn exit_with_message(message: []const u8) void {
    print_message(message);
    std.process.exit(1);
}
```

### Передача функций

В Zig функции являются объектами первого класса, что означает, что их можно передавать в другие функции, возвращать из функций и хранить в переменных. Это мощная возможность, которая позволяет создавать гибкие и модульные программы. Для того, чтобы передать одну функцию в другую, нам необходимо обьявить тип передаваемой переменной используя ключевое слово `fn` и указав типы передаваемых и возвращаемых значений. Давайте рассмотрим пример:

```zig
const std = @import("std");

fn applyFunction(x: i32, func: fn(i32) i32) i32 {
    return func(x);
}

fn square(x: i32) i32 {
    return x * x;
}

fn double(x: i32) i32 {
    return x * 2;
}

pub fn main() void {
    const result1 = applyFunction(5, square); // Применяем square
    const result2 = applyFunction(5, double); // Применяем double
    std.debug.print("Результаты: {}, {}\n", .{result1, result2});
}
```

Данный пример кода выведет:

```
Результаты: 25, 10
```

В нашем примере функция `applyFunction` принимает два аргумента: число `x` и функцию `func`. Функция `applyFunction` вызывает функцию `func` с аргументом `x` и возвращает результат. В зависимости от переданной функции (square или double), applyFunction применяет её к числу.

Функции в Zig могут не только принимать другие функции в качестве параметров, но и возвращать их. Однако, поскольку Zig не поддерживает замыкания, сделать это сложнее, чем в некоторых других языках.

Чтобы вернуть функцию из другой функции, в Zig используются анонимные структуры. Мы еще не рассматривали структуры, поэтому код может показаться вам непонятным, но сейчас важно просто показать, что возвращение функций из функций в Zig возможно.

Этот подход полезен для реализации различных стратегий и создания функций высшего порядка. Одним из примеров является каррирование — функциональная техника, при которой функция, принимающая несколько аргументов, преобразуется в последовательность функций, каждая из которых принимает только один аргумент.

Каррирование позволяет разложить функцию с несколькими аргументами на цепочку вложенных функций. Каждая из них принимает один аргумент и возвращает новую функцию, ожидающую следующий аргумент.

Например, функция `add(a: i32, b: i32) i32` может быть преобразована в `add(a: i32) fn(i32) i32`, где add(5) возвращает функцию, которая принимает b и возвращает 5 + b. Давайте взглянем на код реализующий данную технику:

```zig
const std = @import("std");

fn add(comptime a: i32) fn (i32) i32 {
    return struct {
        fn carr(b: i32) i32 {
            return a + b;
        }
    }.carr;
}

pub fn main() void {
    const addFive = add(5); // Создаём функцию, которая прибавляет 5
    const result1 = addFive(3); // Вызываем её с аргументом 3
    std.debug.print("Результат: {}\n", .{result1});
}
```

В данном коде функция `add` реализует механизм каррирования и при вызове вернет функцию, которая будет к переданному аргументу прибавлять значение `a`. Для возврата функции мы используем анонимную структуру, в которой определяем нашу функцию, которая замкнет на себя значение `a` и возвращем ее из функции `add`. Если мы запустим наш код, то увидим результат:

```
Результат: 8
```

К сожалению из-за отсутсвия поддержки замыканий в Zig этот код выглядит не очень понятно и красиво, но тем не менее в Zig мы можем реализовывать даже такие паттерны как каррирование.

Еще один важный момент в том, что мы пометили нашу переменную `a` как `comptime` (значение известное на этапе компиляции), так как в Zig все функции являются `comptime` по умолчанию, а значит и все значения которые они оборачивают должны быть `comptime`. Это конечно сильно ограничивает возможности возвращения функций из функций и для вариантов с захватом изменяемых значений нужно уже использовать структуры.

### Область видимости функции
По умолчанию все функции, которые вы определяете в своем модуле не доступны из других модулей. Для того чтобы функцию сделать доступной из других модулей, вы можете использовать ключевое слово `pub` перед объявлением функции. Предположим, что у нас есть файл `math.zig`, где мы собрали функции для работы с математическими операциями. Если наша функция в данном модуле будет обьявлена без ключевого слова `pub`, то она будет недоступна из других модулей, а ее можно будет использовать только в файле `math.zig`. Если попытатся вызвать ее из другого модуля компилятор выдаст ошибку. Вот пример:

```zig
// Файл: math.zig

fn add(a: i32, b: i32) i32 {
    return a + b;
}
```

Чтобы исправить наш код, давайте добавим ключевое слово `pub` перед объявлением функции `add`:

```zig
// Файл: math.zig

pub fn add(a: i32, b: i32) i32 {
    return a + b;
}

// Файл: main.zig

const math = @import("math.zig");

pub fn main() void {
    const result = math.add(5, 3);
    std.debug.print("Результат сложения: {}\n", .{result});
}
```

Ключевое слово `pub` в Zig — это простой, но мощный инструмент для управления видимостью функций и других элементов. Оно помогает создавать модульные и безопасные программы, четко разделяя публичный API и внутреннюю реализацию. Используйте `pub` для экспорта функций, которые должны быть доступны извне, и оставляйте остальные функции приватными для улучшения инкапсуляции и читаемости кода.

### Экспорт и импорт функций для использования в других языках программирования
В Zig ключевые слова `extern` и `export` используются для взаимодействия с внешним миром — другими языками программирования или системами. Они позволяют указывать, как функции должны быть видимы за пределами текущего модуля или даже за пределами программы (например, при взаимодействии с C или другими языками). Давайте разберем их подробнее.

Иногда бывает полезно сделать доступной вашу функцию не только для других модулей кода на Zig, но и для других языков программирования, используя FFI (Foreign Function Interface). Для этого вы можете использовать ключевое слово `export`. Это полезно, если вы создаете библиотеку на Zig, которую хотите использовать в других проектах. Например, чтобы сделать доступной функцию `add`, вы можете использовать следующий код:

```zig
// Экспортируем функцию для использования в C
export fn add(a: c_int, b: c_int) c_int {
    return a + b;
}
```
Важный момент: типы переменных используемых в функции должны быть совместимы с тем языком, в который вы хотите экспортировать вашу функцию. В данном примере мы используем типы совместимые с языком C, но об этом мы поговорим позднее когда будем обсуждать интеграцию Zig и C.

Ключевое слово extern используется для объявления функций, которые определены вне Zig (например, в C или других языках). Это позволяет Zig вызывать функции из внешних библиотек или системных API.

Предположим, у нас есть C-библиотека с функцией add, которая складывает два числа:

```c
// Файл: math.c
int add(int a, int b) {
    return a + b;
}
```

Чтобы вызвать эту функцию из Zig, нужно объявить её с ключевым словом extern:

```zig
// Файл: main.zig

// Объявляем внешнюю функцию
extern fn add(a: c_int, b: c_int) c_int;

pub fn main() void {
    const result = add(5, 3);
    std.debug.print("Результат сложения: {}\n", .{result});
}
```

Здесь мы используем ключевое слово extern для объявления функции add, которая определена в C-библиотеке. Это позволяет Zig вызывать эту функцию из своего кода. Важно чтобы типы данных в `extern` функциях были совместимы с внешним языком (например, c_int, c_void и т.д.). Вторым важным моментом является то, что имя функции и её сигнатура (типы аргументов и возвращаемого значения) должны точно совпадать с тем, как она объявлена во внешней библиотеке

### Встраиваемые функции
В Zig ключевое слово `inline` используется для подсказки компилятору, что функцию следует "встроить" (inline) в месте её вызова. Это означает, что вместо вызова функции как отдельной сущности, её код будет вставлен непосредственно в то место, где она вызывается. Это может привести к оптимизации производительности, особенно для небольших функций, которые вызываются часто.

Для того чтобы обьявить вашу функцию как встраеиваемую, используйте ключевое слово `inline` перед объявлением функции. Это подсказка компилятору, что функцию следует встроить. Однако стоит помнить, что `inline` — это именно подсказка, а не строгое указание. Компилятор может проигнорировать её, если сочтёт, что встраивание неэффективно. Давайте посмотрим на примере:

```zig
const std = @import("std");

inline fn add(a: i32, b: i32) i32 {
    return a + b;
}

pub fn main() void {
    const result = add(5, 3);
    std.debug.print("Результат сложения: {}\n", .{result});
}
```

Для чего может быть полезно использовать встраивание функций:
* **Уменьшение накладных расходов**: Встраивание устраняет необходимость в вызове функции, что может быть полезно для небольших функций, вызываемых часто.
* **Оптимизация производительности**: Встроенные функции могут быть оптимизированы компилятором более эффективно, особенно если они работают с константами или небольшими данными.
* **Упрощение отладки**: Встроенные функции могут сделать код более линейным, что упрощает его анализ и отладку.

Однако надо быть осторожным со встраиванием функций, так как это может привести к увеличению размера исполняемого файла.

### Рекурсивные функции
Рекурсия — это мощный инструмент в программировании, который позволяет функции вызывать саму себя для решения задачи. В Zig рекурсивные функции поддерживаются на уровне языка и могут быть использованы для реализации сложных алгоритмов, таких как обход деревьев, вычисление факториалов или решение задач с разделяй и властвуй.

Один из классических примеров рекурсии — вычисление факториала числа. Факториал числа n (обозначается как n!) — это произведение всех положительных целых чисел от 1 до n.

```zig
const std = @import("std");

fn factorial(n: u32) u32 {
    if (n == 0) {
        return 1; // Базовый случай
    }
    return n * factorial(n - 1); // Рекурсивный случай
}

pub fn main() void {
    const result = factorial(5);
    std.debug.print("Факториал 5: {}\n", .{result});
}
```

Этот код выведет:

```
Факториал 5: 120
```

Рекурсивная функция всегда должна иметь базовый случай — условие, при котором рекурсия прекращается. Без базового случая функция будет вызывать себя бесконечно, что приведёт к переполнению стека. Как мы уже знаем каждый вызов рекурсивной функции добавляет новый кадр стека, который содержит локальные переменные и состояние функции. Если рекурсия слишком глубокая, это может привести к переполнению стека. В Zig, как и в других языках, важно следить за глубиной рекурсии.

Хвостовая рекурсия — это рекурсия, в которой рекурсивный вызов является последней операцией перед возвратом результата. В таких случаях компилятор может оптимизировать выполнение и не сохранять промежуточные вызовы в стеке, превращая рекурсию в обычный цикл.

```zig
const std = @import("std");

// Хвостовая рекурсивная функция вычисления факториала
fn factorial(n: u32, acc: u32) u32 {
    if (n == 0) return acc;
    return @call(.always_tail, factorial, .{ n - 1, acc * n });
}

pub fn main() void {
    const result = factorial(5, 1);
    std.debug.print("5! = {}\n", .{result}); // Вывод: 5! = 120
}
```

В данном коде мы используем дополнительный параметр `acc`, который хранит промежуточный результат. Рекурсивный вызов является последней операцией в функции, что позволяет компилятору оптимизировать его превратив в цикл `for`. По умолчанию Zig может не всегда оптимизировать хвостовую рекурсию. Использование функции @call с параметром `.always_tail` гарантирует, что рекурсия будет оптимизирована, а если это невозможно, компилятор выдаст ошибку.

### Использование @call
В Zig есть встроенная функция @call, которая позволяет вызывать функции с явным указанием способа вызова (calling convention). Это полезно в случаях, когда нужно контролировать, как именно вызывается функция, например, для взаимодействия с низкоуровневыми API или для оптимизации.

@call — это встроенная функция (builtin) в Zig, которая позволяет вызывать другую функцию, явно указывая:
* Способ вызова (calling convention).
* Аргументы функции.
* Возвращаемое значение.

Синтаксис:

```zig
@call(calling_convention, function, arguments)
```

Где:
* calling_convention — способ вызова функции (например, .auto, .c, .inline).
* function — функция, которую нужно вызвать.
* arguments — кортеж (tuple) аргументов, передаваемых в функцию.

Мы не будем рассматривать все возможные параметры, которые можно передать функции @call, а лишь рассмотрим некоторые из них. Полное описание параметров Вы можете найти в документации к языку Zig.

Список параметров:

* .always_tail, .never_tail — позволяет управлять, будет ли компилятор пытаться оптимизировать рекурсивный вызов в цикл `for`.
* .always_inline, never_inline — позволяет управлять, будет ли функция встроена (inlined) в вызывающую функцию.
* .auto — Zig автоматически выбирает оптимальный способ вызова функции. Это поведение по умолчанию, которое подходит для большинства случаев

## Перечисления и объединения
Перечисления (enums) и объединения (unions) — это мощные инструменты в Zig, которые позволяют создавать типы данных с ограниченным набором значений или альтернативных представлений. Они особенно полезны для моделирования состояний, обработки ошибок и работы с данными, которые могут принимать разные формы.

### Перечисления (enums)
Перечисления (enums) — это типы данных, которые могут принимать одно из нескольких предопределённых значений. Они полезны для создания ограниченного набора вариантов, например, для представления состояний или категорий. Давайте рассмотрим пример использования перечисления:

```zig
const std = @import("std");

const Color = enum {
    Red,
    Green,
    Blue,
};

pub fn main() void {
    const color = Color.Red;

    switch (color) {
        .Red => std.debug.print("Цвет: Красный\n", .{}),
        .Green => std.debug.print("Цвет: Зелёный\n", .{}),
        .Blue => std.debug.print("Цвет: Синий\n", .{}),
    }
}
```

Вывод:
```
Цвет: Красный
```

В этом примере мы определяем перечисление Color, в котором задаём три возможных значения: Red, Green и Blue. Затем создаём переменную color типа Color и присваиваем ей значение Color.Red. Далее мы используем оператор switch, чтобы проверить, какое значение содержит переменная color, и выводим соответствующее сообщение. Такой способ обработки перечислений называется “сопоставление с образцом” (pattern matching). По умолчанию каждому значению в перечислении присваивается целочисленное значение, начиная с 0. В нашем примере Red имеет значение 0, Green — 1, а Blue — 2.

Если нам не подходят целочисленные значения, которые присваиваются по умолчанию, мы можем либо задать начальное значение для первого элемента перечисления, либо использовать атрибуты для задания значений для каждого элемента. Например, чтобы сделать перечисление с кодами Http статусов для ответа на запрос мы можем написать следующее перечисление:

```zig
const std = @import("std");

const HttpStatus = enum(u32) {
    Ok = 200,
    Created = 201,
    BadRequest = 400,
    Unauthorized = 401,
    Forbidden = 403,
    NotFound = 404,
};

pub fn main() void {
    const status = HttpStatus.Ok;
    std.debug.print("Код статуса: {}\n", .{@intFromEnum(status)});
}
```

По умолчанию Zig будет использовать минимально возможный тип данных, в который вмещаются все варианты перечисления, но вы можете указать при определении перечесления какой тип данных нужно использовать для хранения значений. Для этого после ключевого слова `enum` надо в квадратных скобках указать тип данных, который будет использоваться для хранения значений перечисления:

```zig
const std = @import("std");

const Color = enum(u32) {
    Red,
    Green,
    Blue,
};

pub fn main() void {
    const color = Color.Red;
    std.debug.print("Цвет: {}\n", .{@intFromEnum(color)});
}
```

В Zig встроенная функция `@tagName` позволяет получить строковое представление значения перечисления (enum). Это особенно полезно для отладки, логирования или вывода значений перечисления в удобочитаемом виде. Рассмотрим простой пример перечисления и использование `@tagName` для получения строкового представления его значений:

```zig
const std = @import("std");

const Direction = enum {
    North,
    South,
    East,
    West,
};

pub fn main() void {
    const direction = Direction.North;
    std.debug.print("Направление: {s}\n", .{@tagName(direction)});
}
```

Этот код выведет:

```
Направление: North
```

Для того, чтобы наоборот получить числовое значение из перечисления нам необходимо использовать встроенную функцию `@intFromEnum`. Данный функция на вход принимает значение перечисления и возвращает его числовое представление:

```zig
const std = @import("std");

const Direction = enum {
    North,
    South,
    East,
    West,
};

pub fn main() void {
    const direction = Direction.North;
    std.debug.print("Направление: {}\n", .{@intFromEnum(direction)});
}
```

Этот код выведет:

```
Направление: 0
```

В Zig перечисления могут содержать функции привязанные к перечислению. Это может быть полезно, если Вы хотите сделать Ваш код более выразительным и понятным. Например, можно добавить функцию `toString` к перечислению `Direction`, которая будет возвращать строковое представление направления:

```zig
const std = @import("std");

const Direction = enum {
    North,
    South,
    East,
    West,

    fn toString(self: Direction) []const u8 {
        return switch (self) {
            Direction.North => "North",
            Direction.South => "South",
            Direction.East => "East",
            Direction.West => "West",
        };
    }
};

pub fn main() void {
    const direction = Direction.North;
    std.debug.print("Направление: {s}\n", .{direction.toString()});
}
```

Этот код выведет:

```
Направление: North
```

### Объединение (Union)
Объединения (unions) — это типы данных, которые могут хранить одно из нескольких возможных значений. В отличие от перечислений, объединения позволяют хранить данные разных типов в одной переменной. При этом компилятор будет использовать наиболее объемный тип для хранения всех значений объединения. Например давайте рассмотрим пример объединения `Number`:

```zig
const std = @import("std");

const Number = union {
    int: u8,
    float: f64,
    none: void,
};

pub fn main() void {
    std.debug.print("Size of: {}\n", .{@sizeOf(Number)});

    const n = Number{ .int = 32 };
    std.debug.print("Value: {d}\n", .{n.int});
}
```

Этот код выведет:

```
Size of: 16
Value: 32
```

В данном примере мы создали объединение `Number`, которое может хранить целое число (`u8`), дробное число (`f64`) или ничего (`void`). Так как в нашем обьединении тип `f64` имеет наибольший размер, то компилятор будет использовать его для хранения всех значений объединения. Вам может показаться странным, что размер нашего обьединения получился 16 байт, а не 8. Все дело в том, что по умолчанию Zig собирает debug сборку, где используется выравнивание по 8 байт. Если мы запустим нашу программу в режиме release, то увидим верный размер занимаемой памяти:

```
$ zig build run -Doptimize=ReleaseFast
Size of: 8
32
```

По умолчанию в перечислении может быть задано только одно значение и если Вы обратитесь к значению, которого нет в перечислении, то компилятор выдаст ошибку:

```zig
const std = @import("std");

const Number = union {
    int: u8,
    float: f64,
    none: void,
};

pub fn main() void {
    const n = Number{ .int = 32 };
    std.debug.print("{d}\n", .{n.float});
}
```

Выведет:

```
run
└─ run hello
   └─ zig build-exe hello ReleaseFast native 1 errors
src/main.zig:11:33: error: access of union field 'float' while field 'int' is active
    std.debug.print("{d}\n", .{n.float});
                               ~^~~~~~
src/main.zig:3:16: note: union declared here
```

### Маркированные объединения
Маркированные объединения — это мощный инструмент в языке Zig, который позволяет создавать типы данных, способные хранить значения разных типов, но с явной меткой (тегом), указывающей, какой именно тип данных используется в данный момент. Это делает их безопасной и удобной альтернативой традиционным объединениям (`union`) из языка C, где отсутствует встроенная проверка корректности типа. В Zig маркированные объединения реализуются с помощью ключевого слова `union`, которое используется вместе с перечислением (`enum`) для определения тегов.

```zig
const std = @import("std");

const Value = union(enum) {
    int: i32,
    float: f64,
    bool: bool,
};

pub fn main() !void {
    const val = Value{ .float = 5.0 };

    switch (val) {
        .float => |f| std.debug.print("Value is float: {}\n", .{f}),
        .int => |i| std.debug.print("Value is int: {}\n", .{i}),
        .bool => |b| std.debug.print("Value is bool: {}\n", .{b}),
    }
}
```

Этот код выведет:

```
Value is float: 5.0
```

В этом примере Value — это маркированное объединение, которое может хранить либо целое число (int), либо число с плавающей точкой (float), либо логическое значение (bool). Тег (enum) автоматически добавляется компилятором и позволяет безопасно работать с данными.

Предположим мы хотим написать функцию, которая получает наше значение `Value` и тег нашего типа, какой тип будет у параметра функции если наш тег компилятор выводит автоматически. Для того чтобы указать, что тип переменной это перечисление (`enum`), которое было сформировано компилятором мы можем использовать функцию из стандартной библиотеки `std.meta.Tag`, которая возвращает тип тега для заданного типа объединения:

```zig
const std = @import("std");

const Value = union(enum) {
    int: i32,
    float: f64,
    bool: bool,
};

fn is(value: Value, tag: std.meta.Tag(Value)) bool {
    return value == tag;
}

pub fn main() !void {
    const val = Value{ .float = 5.0 };

    switch (val) {
        .float => |f| std.debug.print("Value is float: {}\n", .{f}),
        .int => |i| std.debug.print("Value is int: {}\n", .{i}),
        .bool => |b| std.debug.print("Value is bool: {}\n", .{b}),
    }

    std.debug.print("Is integer value: {}\n", .{is(val, .int)});
}
```

Этот код выведет:

```
Value is float: 5.0
Is integer value: false
```

## Ошибки
Обработка ошибок — это важная часть разработки надёжного программного обеспечения. В Zig подход к обработке ошибок отличается простотой, прозрачностью и эффективностью. В отличие от исключений в других языках, Zig использует явную обработку ошибок, что делает код более предсказуемым и безопасным. Обработка ошибок в Zig очень похожа на подход, который использует Rust, но в отличие от Rust в Zig работать с ошибками гораздо проще.

В Zig ошибки представлены с помощью типа `error`, который очень похож на рассмотренные нами перечисления. Это набор уникальных идентификаторов, которые определяют возможные ошибки в программе. Например:

```zig
const FileError = error{
    NotFound,
    PermissionDenied,
    DiskFull,
};
```

Здесь `FileError` — это тип ошибки, который может принимать три значения: `NotFound`, `PermissionDenied` и `DiskFull`.

Функции в Zig могут возвращать ошибки с помощью специального синтаксиса `!`. Тип возвращаемого значения функции может быть объединением (union) результата и ошибки. Тип возвращаемого значения функции, которая может вернуть ошибку, записывается в формате `ErrorType!ReturnType`. Здесь `ErrorType` — это тип ошибки (например, error{NotFound}), а `ReturnType` — тип успешного результата.

Например:

```zig
const FileError = error{
    NotFound,
    PermissionDenied,
    DiskFull,
};

fn openFile(path: []const u8) FileError!void {
    if (path.len == 0) return error.NotFound;
    // Логика открытия файла...
}
```

В этом примере функция `openFile` возвращает либо `void` (если файл успешно открыт), либо ошибку типа `FileError`. В отличие от исключений используемых в других языках, ошибки в Zig это значения, которые должны быть сформированы в каком-то месте вашей программы и должны быть обработаны в другом. Если функция возвращает ошибку как один из вариантов своего значения, то Вы не можете отбросить ее обработку. Например если Вы попытаетесь присвоить результат функции `openFile` переменной `_`, тем самым пытаясь отбросить ошибку, то компилятор выдаст ошибку.

Иногда Вы могли видеть примеры кода, где мы не указывали конкретный тип ошибок, а просто указывали что наша функция может вернуть ошибку `!void`. В этом случае компилятор Zig сам автоматически выводит тип ошибки из всех возможных типов, которые могут быть возвращены функцией. Однако хорошей практикой является явное указание типа ошибок, во первый потому что это позволяет документировать ваш код, а во вторых в случае явного указания типа ошибок компилятор сможет проверять не возвращается ли из вашей функции ошибка не учтенная в типе ошибок.

В Zig вы можете обьединять наборы ошибок в новый набор используя оператор `||`. Это позволяет создавать более сложные наборы ошибок, которые могут быть использованы в других функциях:

```zig
const FileSystemError = FileError || error{
    InvalidPath,
    NotADirectory,
};
```

В данном примере мы объединяем `FileError` с новыми ошибками `InvalidPath` и `NotADirectory`, чтобы создать новый набор ошибок `FileSystemError`.

### Использование try и catch
В отличие от других языков, где конструкция `try/catch` используется для обработки исключений, в Zig эти операторы используются немного по своему и необходимы для обработки ошибок. Ключевое слово `catch` используется для обработки ошибки вызываемой вами функции. Давайте напишем функцию деления двух целых чисел и добавим в нее обработку ошибок:

```zig
const std = @import("std");

const MathError = error{
    DivisionByZero,
};

fn divide(a: u32, b: u32) MathError!u32 {
    if (b == 0) return error.DivisionByZero;
    return a / b;
}

pub fn main() void {
    const result = divide(10, 0) catch |err| {
        std.debug.print("Ошибка: {s}\n", .{@errorName(err)});
        return;
    };
    std.debug.print("Результат: {}\n", .{result});
}
```

Этот код выведет:

```
Ошибка: DivisionByZero
```

В этом примере в функции `divide` мы обрабатываем ошибку деления на ноль и возвращаем ошибку `DivisionByZero`. Если ошибки не возникает, функция возвращает результат деления. Для того чтобы обработать ошибку, мы используем оператор `catch`, который также позволяет нам захватить значение ошибки и вывести его например на экран пользователю. Для того чтобы вывести читаемый тип ошибки пользователю, мы используем функцию `@errorName` из стандартной библиотеки Zig. Но для отладки ваших ошибок мы также могли использовать спецификатор `!`, правда его результат выведет полное имя типа ошибки:

```zig
const std = @import("std");

const MathError = error{
    DivisionByZero,
};

fn divide(a: u32, b: u32) MathError!u32 {
    if (b == 0) return error.DivisionByZero;
    return a / b;
}

pub fn main() void {
    const result = divide(10, 0) catch |err| {
        std.debug.print("Ошибка: {!}\n", .{err});
        return;
    };
    std.debug.print("Результат: {}\n", .{result});
}
```

Будет выведено:

```
Ошибка: error.DivisionByZero
```

Теперь давайте рассмотрим ключевое слово `try`, которое позволяет упростить код обработки ошибок. Если в нашем коде нам часто нужно пробрасывать ошибку в вызываемую функцию то нам прийдется писать какой-то такой код обработки:

```zig
const std = @import("std");

const MathError = error{
    DivisionByZero,
};

fn divide(a: u32, b: u32) MathError!u32 {
    if (b == 0) return error.DivisionByZero;
    return a / b;
}

pub fn some_func() !void {
    const result = divide(10, 0) catch |err| {
        return err;
    };
    ...
}
```

Здесь мы просто пробрасываем ошибку из нашей функции выше и писать такой шаблонный код утомительно, поэтому Zig предлагает Вам оператор `try`. Мы можем переписать наш код следующим образом:

```zig
const std = @import("std");

const MathError = error{
    DivisionByZero,
};

fn divide(a: u32, b: u32) MathError!u32 {
    if (b == 0) return error.DivisionByZero;
    return a / b;
}

pub fn some_func() !void {
    const result = try divide(10, 0);
    ...
}
```

Согласитесь так гораздо короче. По сути оператор `try` делает следующее - если при вызове функции возникла ошибка, она будет возвращена вызывающей функции, если ошибки нет, то `try` просто вернет значение, которое вернула функция.

### Обработка ошибок
Обычно для обработки ошибок в Zig удобно использовать связку оператора `catch` и оператора `switch`. Это позволяет нам обрабатывать ошибки в удобном для нас виде. Предположим, у нас есть функция readFile, которая пытается прочитать файл и может вернуть различные ошибки, такие как NotFound, PermissionDenied или DiskFull. Мы хотим обработать каждую ошибку по-своему.

```zig
const std = @import("std");

const FileError = error{
    NotFound,
    PermissionDenied,
    DiskFull,
};

fn readFile(path: []const u8) FileError![]const u8 {
    if (std.mem.eql(u8, path, "")) return error.NotFound;
    if (std.mem.eql(u8, path, "/restricted")) return error.PermissionDenied;
    if (std.mem.eql(u8, path, "/full")) return error.DiskFull;
    return "Содержимое файла";
}

pub fn main() void {
    const path = "/restricted"; // Попробуйте изменить путь на "/full" или ""
    const result = readFile(path) catch |err| switch (err) {
        error.NotFound => {
            std.debug.print("Файл не найден: {s}\n", .{path});
            return;
        },
        error.PermissionDenied => {
            std.debug.print("Доступ запрещён: {s}\n", .{path});
            return;
        },
        error.DiskFull => {
            std.debug.print("Диск переполнен: {s}\n", .{path});
            return;
        },
    };
    std.debug.print("Успешно прочитано: {s}\n", .{result});
}
```

В данном примере мы используем оператор `switch` чтобы по разному обработать разные типы ошибок, которые могут возникнуть при чтении файла. В зависимости от типа ошибки, мы выводим соответствующее сообщение и завершаем программу.

Два других популярных способа обработки ошибок в Zig — это использование операторов `if` и `while`.

Оператор `if` позволяет проверить, произошла ли ошибка при выполнении операции. Если ошибки нет, значение присваивается переменной, которую можно использовать внутри блока `if`. В случае ошибки можно обработать её в блоке `else`.

Вот как это работает:

```zig
if (someFunctionThatMayFail()) |result| {
    std.debug.print("Success: {}\n", .{result});
} else |err| {
    std.debug.print("Error occurred: {}\n", .{err});
}
```

Также можно использовать оператор `while` для обработки ошибок, особенно когда требуется многократное выполнение операции до успешного результата. Принцип обработки ошибки в нем схож с тем как мы делаем это в операторе `if`:

```zig
const std = @import("std");

fn mightFail(counter: *u8) !u8 {
    if (counter.* < 3) {
        counter.* += 1;
        return error.TryAgain;
    }
    return 42; // Успешный результат после нескольких попыток
}

pub fn main() !void {
    var attempt: u8 = 0;

    while (mightFail(&attempt)) |result| {
        std.debug.print("Success: {}\n", .{result});
    } else |err| {
        std.debug.print("Error occurred: {}\n", .{err});
    }
}
```
