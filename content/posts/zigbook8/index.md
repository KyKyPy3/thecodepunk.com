---
title: Срезы и строки
date: 2025-02-28 15:00:00
showTableOfContents: true
showComments: true
katex: true
tags:
  - zig
  - zigbook
---
Срезы в Zig представляют собой структуру данных, которая содержит указатель на начало последовательности элементов и длину этой последовательности. Их можно рассматривать как "вид" на подмножество массива или области памяти. Эта концепция будет знакома разработчикам, имеющим опыт работы с языками Rust или Go.

## Объявление и создание срезов

Тип среза в Zig обозначается как `[]T`, где `T` — это тип элементов. Например, `[]u8` представляет срез беззнаковых 8-битных целых чисел, а `[]const u8` — срез, элементы которого не могут быть изменены (что часто используется для строк).

Если Вы попробуете создать срез из массива или из части массива, но при этом границы вашего среза будут известны на этапе компиляции, то на самом деле вы создадите не срез, а просто указатель на массив. Давайте рассмотрим эти два сценария:

```zig
const std = @import("std");

pub fn main() void {
    // Исходный массив
    var array = [_]i32{ 1, 2, 3, 4, 5 };

    // Создание среза всего массива
    const slice1 = &array;

    std.debug.print("Slice1 type: {}\n", .{@TypeOf(slice1)});

    // Создание среза части массива
    const slice2 = array[1..3];

    std.debug.print("Slice2 type: {}\n", .{@TypeOf(slice2)});
}
```

Этот код выведет следующее:

```
Slice1 type: *[5]i32
Slice2 type: *[2]i32
```

Для того, чтобы явно определить срез, нам необходимо указать тип создаваемой переменной как тип среза, например:

```zig
const std = @import("std");

pub fn main() void {
    var array = [_]i32{ 1, 2, 3, 4, 5 };

    // Создание среза элементов со 2-го по 4-й (индексы 1-3)
    const slice: []i32 = array[1..4];

    std.debug.print("Slice type: {}\n", .{@TypeOf(slice)});
}
```

В результате мы получим вывод:

```
Slice type: []i32
```

Еще один из вариантов определения среза, это использование рассмотренных ранее аллокаторов:

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // Создание среза с выделением памяти
    var dynamic_slice = try allocator.alloc(i32, 5);
    defer allocator.free(dynamic_slice);

    // Заполнение среза
    for (dynamic_slice, 0..) |*item, i| {
        item.* = @intCast(i + 1);
    }

    // Вывод среза
    for (dynamic_slice) |item| {
        std.debug.print("{} ", .{item});
    }
    std.debug.print("\n", .{});
}
```

Для создания среза Вам не обязательно использовать маасив, Вы можете создавать срезу из других срезов:

```zig
const subslice = slice[1..3];
```

В этом случае оба среза указаывают на один и тот же массив, поэтому изменения в одном срезе будут отражаться в другом. Работая со срезами Вы всегда можете получить текущую длину среза через `slice.len`:

```zig
const length = slice.len;
```

### Доступ к элементам среза

Доступ к элементам среза осуществляется так же, как и к элементам массива — с помощью оператора индексации `[]`:

```zig
const std = @import("std");

pub fn main() void {
    var array = [_]i32{ 1, 2, 3, 4, 5 };
    var slice: []i32 = array[0..];

    // Доступ к элементу по индексу
    const third_element = slice[2];
    std.debug.print("Третий элемент: {}\n", .{third_element});

    // Изменение элемента по индексу
    slice[0] = 10;
    std.debug.print("Обновленный первый элемент: {}\n", .{slice[0]});
}
```

### Модификация срезов

Важно понимать, что срезы — это "проекция" на существующие данные. Когда вы модифицируете элемент среза, вы фактически изменяете соответствующий элемент в базовом массиве или области памяти:

```zig
const std = @import("std");

pub fn main() void {
    var array = [_]i32{ 1, 2, 3, 4, 5 };

    // Создаем два среза на один и тот же массив
    var slice1: []i32 = array[0..3];
    var slice2: []i32 = array[2..5];

    // Изменяем общий элемент через первый срез
    slice1[2] = 30;

    // Проверяем, что изменение отразилось и во втором срезе
    std.debug.print("slice2[0] = {}\n", .{slice2[0]});
    // Вывод: slice2[0] = 30

    // Проверяем, что изменение отразилось в исходном массиве
    std.debug.print("array[2] = {}\n", .{array[2]});
    // Вывод: array[2] = 30
}
```

Кроме изменения конкретного элемента среза, мы также може обьединять срезы с помощью аллокаторов:

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var slice1 = [_]i32{ 1, 2, 3 };
    var slice2 = [_]i32{ 4, 5, 6 };

    // Создаем новый срез, который вместит оба среза
    var result = try allocator.alloc(i32, slice1.len + slice2.len);
    defer allocator.free(result);

    // Копируем данные из первого среза
    std.mem.copy(i32, result[0..slice1.len], &slice1);

    // Копируем данные из второго среза
    std.mem.copy(i32, result[slice1.len..], &slice2);

    // Выводим результат
    for (result) |item| {
        std.debug.print("{} ", .{item});
    }
    std.debug.print("\n", .{});
    // Вывод: 1 2 3 4 5 6
}
```

Иногда чтобы изменить элемент Вам необходимо его найти в срезе. Для этого можно использовать функции стандартной библиотеки в Zig:

```zig
const std = @import("std");

pub fn main() void {
    var array = [_]i32{ 1, 2, 3, 4, 5 };
    var slice = &array;

    // Поиск первого вхождения элемента 3
    const index = std.mem.indexOf(i32, slice, &[_]i32{3});

    if (index) |i| {
        std.debug.print("Элемент 3 найден на позиции {}\n", .{i});
    } else {
        std.debug.print("Элемент 3 не найден\n", .{});
    }
}
```

### Sentinel срезы
Как и массивы в Zig, срезы могут быть sentinel-срезами. Sentinel-срез - это срез, который завершается нулевым значением. Это позволяет использовать срезы в функциях, которые ожидают, что срез будет завершен нулевым значением. Это особенно полезно для совместимости с C-строками, которые заканчиваются нулевым байтом:

```zig
const std = @import("std");

pub fn main() void {
    // Создаем сентинельный срез, заканчивающийся нулем
    var sentinel_array = [_]u8{ 'h', 'e', 'l', 'l', 'o', 0 };
    var sentinel_slice: [:0]u8 = sentinel_array[0..5 :0];

    // В sentinel_slice автоматически добавляется сентинель '0' после 'o'
    std.debug.print("Длина среза: {}\n", .{sentinel_slice.len});
    // Вывод: Длина среза: 5

    // Доступ к сентинелю
    const sentinel = sentinel_slice[sentinel_slice.len];
    std.debug.print("Sentinel: {}\n", .{sentinel});
    // Вывод: Sentinel: 0
}
```

Тип sentinel среза обозначается как `[:sentinel]T`, где `sentinel` — это значение-маркер.

## Строки
