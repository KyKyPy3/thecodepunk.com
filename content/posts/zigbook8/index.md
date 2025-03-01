---
title: Слайсы и строки
date: 2025-02-28 15:00:00
showTableOfContents: true
showComments: true
katex: true
tags:
  - zig
  - zigbook
---
Слайсы в Zig представляют собой структуру данных, которая содержит указатель на начало последовательности элементов и длину этой последовательности. Их можно рассматривать как "вид" на подмножество массива или области памяти. Эта концепция будет знакома разработчикам, имеющим опыт работы с языками Rust или Go.

### Объявление и создание слайсов

Тип слайса в Zig обозначается как `[]T`, где `T` — это тип элементов. Например, `[]u8` представляет слайс беззнаковых 8-битных целых чисел, а `[]const u8` — слайс, элементы которого не могут быть изменены (что часто используется для строк).

Если Вы попробуете создать слайс из массива или из части массива, но при этом границы вашего слайса будут известны на этапе компиляции, то на самом деле вы создадите не слайс, а просто указатель на массив. Давайте рассмотрим эти два сценария:

```zig
const std = @import("std");

pub fn main() void {
    // Исходный массив
    var array = [_]i32{ 1, 2, 3, 4, 5 };

    // Создание слайса всего массива
    const slice1 = &array;

    std.debug.print("Slice1 type: {}\n", .{@TypeOf(slice1)});

    const slice2 = array[1..3];

    std.debug.print("Slice2 type: {}\n", .{@TypeOf(slice2)});
}
```

Этот код выведет следующее:

```shell
Slice1 type: *[5]i32
Slice2 type: *[2]i32
```

Для того, чтобы явно определить слайс, нам необходимо указать тип создаваемой переменной как тип слайса, например:

```zig
const std = @import("std");

pub fn main() void {
    var array = [_]i32{ 1, 2, 3, 4, 5 };

    // Создание слайса элементов со 2-го по 4-й (индексы 1-3)
    const slice: []i32 = array[1..4];

    std.debug.print("Slice type: {}\n", .{@TypeOf(slice)});
}
```

В результате мы получим вывод:

```shell
Slice type: []i32
```

Еще один из вариантов определения слайса, это использование рассмотренных ранее аллокаторов:

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // Создание слайса с выделением памяти
    var dynamic_slice = try allocator.alloc(i32, 5);
    defer allocator.free(dynamic_slice);

    // Заполнение слайса
    for (dynamic_slice, 0..) |*item, i| {
        item.* = @intCast(i + 1);
    }

    // Вывод слайса
    for (dynamic_slice) |item| {
        std.debug.print("{} ", .{item});
    }
    std.debug.print("\n", .{});
}
```
