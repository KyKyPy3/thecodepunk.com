---
title: Структуры
date: 2025-03-08 15:00:00
showTableOfContents: true
showComments: true
katex: true
tags:
  - zig
  - zigbook
  - struct
---

Для завершения базового блока ознакомления с языком Zig нам осталось рассмотреть еще одну очень важную тему - структуры. Так как язык Zig ближе к языку C, чем к таким языкам как C++ и Java, в нем нет привычного для этих языков подхода к ООП в виде классов, наследования и интерфейсов. В языке Zig для реализации некоторых принципов ООП используются структуры.

Структуры (struct) — это одна из ключевых конструкций в Zig, позволяющая группировать данные и создавать пользовательские типы. Они используются для организации сложных данных, представления объектов и моделирования реальных сущностей. В этой главе мы подробно рассмотрим, как работать со структурами в Zig, их синтаксис, особенности и примеры использования.

## Определение структур
Для того чтобы определить структуру в языке Zig используется ключевое слово `struct`, после которого идет имя вашей структуры и фигурные скобки, в которых определяются поля структуры. Рассмотрим простой пример структуры, представляющей точку в двумерном пространстве:

```zig
const Point = struct {
    x: i32,
    y: i32,
};
```

Обычно название структуры принято называть с большой буквы, тем самым мы указываем, что это имя типа данных. Имя структуры должно по возможности выразительно описывать ее назначение и содержимое. Например, если структура представляет собой данные о пользователе, ее имя может быть `User`, а если она содержит информацию о товаре, то `Product`. Это поможет другим разработчикам понять, что именно хранит данная структура и как ее использовать.

В блоке структуры, внутри фигурных скобок, мы определяем имена и типы элементов данных, которые называются «поля». Определение структуры всегда должно заканчиваться точкой с запятой, так же как это происходит в языке C.

## Создание экземпляров структур

Для того чтобы создать экземпляр структуры, мы используем ее имя и передаем значения для каждого поля в фигурных скобках. Например, чтобы создать экземпляр структуры `Point` с координатами (10, 20), мы пишем:

```zig
const std = @import("std");
const Point = struct {
    x: i32,
    y: i32,
};

pub fn main() void {
    const p = Point{ .x = 10, .y = 20 };
    std.debug.print("Point: ({d}, {d})\n", .{ p.x, p.y });
}
```

Обратите внимание на синтаксис инициализации полей: перед именем поля ставится точка (`.x = 10`). Это специальный синтаксис Zig, который делает код более читаемым и явно указывает, какие поля мы инициализируем.

Этот код создает экземпляр структуры `Point` с координатами (10, 20) и выводит его на экран:

```
Point: (10, 20)
```

## Конструкторы и деструкторы

Часто для создания экземпляров структур используется конструктор, который позволяет инициализировать поля структуры в более удобном и читаемом виде. В Zig нет встроенного механизма конструкторов, как в объектно-ориентированных языках, но мы можем определить функцию внутри структуры, которая будет выполнять роль конструктора.

Например, для структуры `Point` можно создать конструктор:

```zig
const std = @import("std");
const Point = struct {
    x: i32,
    y: i32,

    pub fn init(x: i32, y: i32) Point {
        return Point{ .x = x, .y = y };
    }
};

pub fn main() void {
    var p = Point.init(10, 20);
    std.debug.print("Point: ({d}, {d})\n", .{ p.x, p.y });
}
```

В данной реализации мы создаем экземпляр структуры `Point` с помощью функции `init`, которая принимает координаты `x` и `y` и возвращает экземпляр структуры `Point` с этими координатами.

Обычно для простых структур конструкторы не нужны, так как можно использовать инициализацию в фигурных скобках. Но они становятся полезными в следующих случаях:

1. Когда у структуры много полей, и мы хотим предоставить значения по умолчанию
2. Когда требуется валидация входных данных
3. Когда необходимо выполнить сложную логику при инициализации
4. Когда внутри структуры используется динамическое выделение памяти

Если ваша структура содержит поля, требующие освобождения ресурсов (например, динамически выделенную память), вам также понадобится функция деструктора. В Zig, по соглашению, для этой цели используется функция с именем `deinit`:

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

const DynamicArray = struct {
    data: []i32,
    allocator: *Allocator,

    pub fn init(allocator: *Allocator, size: usize) !DynamicArray {
        const data = try allocator.alloc(i32, size);
        return DynamicArray{
            .data = data,
            .allocator = allocator,
        };
    }

    pub fn deinit(self: *DynamicArray) void {
        self.allocator.free(self.data);
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = &gpa.allocator;

    var array = try DynamicArray.init(allocator, 10);
    defer array.deinit();

    // Работа с массивом...
    for (0..array.data.len) |i| {
        array.data[i] = @intCast(i32, i) * 2;
    }

    std.debug.print("Array: ", .{});
    for (array.data) |value| {
        std.debug.print("{d} ", .{value});
    }
    std.debug.print("\n", .{});
}
```

В этом примере мы создаем структуру `DynamicArray`, которая содержит динамически выделенный массив и ссылку на аллокатор. Конструктор `init` выделяет память для массива, а деструктор `deinit` освобождает эту память. Обратите внимание на использование ключевого слова `defer` для гарантированного вызова деструктора, даже в случае возникновения ошибок.

По соглашению в языке Zig для имен конструкторов и деструкторов структур используются имена `init` и `deinit` соответственно. Вы, конечно, можете использовать удобное вам имя для конструктора, но для деструктора настоятельно рекомендуется использовать имя `deinit`, чтобы соответствовать стандартной библиотеке и ожиданиям других разработчиков.

Вы также вправе создавать несколько конструкторов для одной структуры, если это необходимо. Например, вы можете создать конструктор, который принимает начальный размер массива и конструктор, который принимает начальный размер и начальное значение элементов массива.

## Методы структур

Структуры в Zig могут содержать не только поля данных, но и функции, которые работают с этими данными. Такие функции часто называют методами структуры. Рассмотрим пример структуры `Rectangle` с методами для вычисления площади и периметра:

```zig
const std = @import("std");

const Rectangle = struct {
    width: f32,
    height: f32,

    pub fn init(width: f32, height: f32) Rectangle {
        return Rectangle{ .width = width, .height = height };
    }

    pub fn area(self: Rectangle) f32 {
        return self.width * self.height;
    }

    pub fn perimeter(self: Rectangle) f32 {
        return 2 * (self.width + self.height);
    }

    pub fn scale(self: *Rectangle, factor: f32) void {
        self.width *= factor;
        self.height *= factor;
    }
};

pub fn main() void {
    var rect = Rectangle.init(5.0, 3.0);

    std.debug.print("Rectangle: width={d}, height={d}\n", .{ rect.width, rect.height });
    std.debug.print("Area: {d}\n", .{rect.area()});
    std.debug.print("Perimeter: {d}\n", .{rect.perimeter()});

    rect.scale(2.0);
    std.debug.print("After scaling: width={d}, height={d}\n", .{ rect.width, rect.height });
    std.debug.print("New area: {d}\n", .{rect.area()});
}
```

В этом примере мы определили три метода для структуры `Rectangle`:
1. `area()` - вычисляет площадь прямоугольника
2. `perimeter()` - вычисляет периметр прямоугольника
3. `scale()` - изменяет размеры прямоугольника, умножая их на указанный коэффициент

Обратите внимание, что методы `area()` и `perimeter()` принимают параметр `self` по значению, так как они только читают данные структуры. А метод `scale()` принимает `self` по указателю (`*Rectangle`), так как он изменяет поля структуры. Как Вы могли заметить параметр структуры, который мы передаем первым аргументом имеет наименование `self`. Это не обязательное требование, а просто соглашение и вы можете использовать любое другое имя, если хотите, например в нашем примере Вы можете использовать `rect` вместо `self`.

## Вложенные структуры
В Zig структуры могут быть вложенными, то есть одна структура может содержать другую структуру в качестве поля. Рассмотрим пример вложенной структуры:

```zig
const std = @import("std");

const Address = struct {
    street: []const u8,
    city: []const u8,
    zip_code: []const u8,
};

const Person = struct {
    name: []const u8,
    age: u32,
    address: Address,

    pub fn init(name: []const u8, age: u32, street: []const u8, city: []const u8, zip_code: []const u8) Person {
        return Person{
            .name = name,
            .age = age,
            .address = Address{
                .street = street,
                .city = city,
                .zip_code = zip_code,
            },
        };
    }

    pub fn printInfo(self: Person) void {
        std.debug.print("Name: {s}, Age: {d}\n", .{ self.name, self.age });
        std.debug.print("Address: {s}, {s}, {s}\n", .{
            self.address.street,
            self.address.city,
            self.address.zip_code
        });
    }
};

pub fn main() void {
    const person = Person.init(
        "John Doe",
        30,
        "123 Main St",
        "New York",
        "10001"
    );

    person.printInfo();
}
```

В этом примере структура `Person` содержит вложенную структуру `Address`. Это позволяет организовать данные в более структурированном и логичном виде.

## Модификатор доступа для полей и методов структуры
В Zig, ключевое слово `pub` (от англ. "public" - публичный) служит модификатором доступа, который определяет, видимы ли поля и методы структуры за её пределами. Это важный механизм для контроля доступа к данным и реализации инкапсуляции.

### Публичные и приватные поля
По умолчанию, все поля и методы структуры в Zig являются приватными - они доступны только внутри модуля, где определена структура. Если вы хотите сделать поле или метод доступным извне, вы должны пометить его ключевым словом `pub`:

```zig
const std = @import("std");

const Counter = struct {
    // Публичное поле, доступно извне
    pub value: u32,

    // Приватное поле, доступно только внутри структуры
    max_value: u32,

    pub fn init(max: u32) Counter {
        return Counter{
            .value = 0,
            .max_value = max,
        };
    }

    pub fn increment(self: *Counter) void {
        if (self.value < self.max_value) {
            self.value += 1;
        }
    }

    pub fn reset(self: *Counter) void {
        self.value = 0;
    }

    // Приватный метод, доступен только внутри модуля
    fn validate(self: Counter) bool {
        return self.value <= self.max_value;
    }
};

pub fn main() void {
    var counter = Counter.init(100);

    // Доступ к публичному полю
    std.debug.print("Initial value: {d}\n", .{counter.value});

    // Использование публичных методов
    counter.increment();
    counter.increment();
    std.debug.print("After increment: {d}\n", .{counter.value});

    counter.reset();
    std.debug.print("After reset: {d}\n", .{counter.value});

    // Следующая строка вызовет ошибку компиляции, так как max_value - приватное поле
    // std.debug.print("Max value: {d}\n", .{counter.max_value});

    // Следующая строка вызовет ошибку компиляции, так как validate - приватный метод
    // counter.validate();
}
```

В этом примере:
- `value` - публичное поле, к которому можно обращаться извне структуры
- `max_value` - приватное поле, доступное только внутри методов структуры
- `init`, `increment` и `reset` - публичные методы, которые можно вызывать извне
- `validate` - приватный метод, который можно вызывать только из других методов структуры

### Инкапсуляция в Zig

Использование ключевого слова `pub` позволяет реализовать принцип инкапсуляции из объектно-ориентированного программирования. Вы можете скрыть внутреннее состояние структуры, предоставляя доступ только через публичные методы:

```zig
const std = @import("std");

const BankAccount = struct {
    // Все поля приватные
    account_number: []const u8,
    balance: f64,
    owner: []const u8,

    pub fn init(account_number: []const u8, owner: []const u8) BankAccount {
        return BankAccount{
            .account_number = account_number,
            .balance = 0.0,
            .owner = owner,
        };
    }

    pub fn deposit(self: *BankAccount, amount: f64) !void {
        if (amount <= 0) {
            return error.InvalidAmount;
        }
        self.balance += amount;
    }

    pub fn withdraw(self: *BankAccount, amount: f64) !void {
        if (amount <= 0) {
            return error.InvalidAmount;
        }
        if (amount > self.balance) {
            return error.InsufficientFunds;
        }
        self.balance -= amount;
    }

    pub fn getBalance(self: BankAccount) f64 {
        return self.balance;
    }

    pub fn getOwner(self: BankAccount) []const u8 {
        return self.owner;
    }

    pub fn getAccountNumber(self: BankAccount) []const u8 {
        return self.account_number;
    }
};

pub fn main() !void {
    var account = BankAccount.init("123456789", "John Doe");

    try account.deposit(1000.0);
    std.debug.print("Balance after deposit: {d:.2}\n", .{account.getBalance()});

    try account.withdraw(500.0);
    std.debug.print("Balance after withdrawal: {d:.2}\n", .{account.getBalance()});

    std.debug.print("Account owner: {s}\n", .{account.getOwner()});
    std.debug.print("Account number: {s}\n", .{account.getAccountNumber()});

    // Ошибка компиляции - нет прямого доступа к приватным полям
    // std.debug.print("Direct balance access: {d}\n", .{account.balance});
}
```

В этом примере мы создали структуру `BankAccount` с полностью приватными полями. Доступ к этим полям предоставляется только через публичные методы, что обеспечивает контроль над тем, как данные могут быть изменены или прочитаны.

При проектировании структур в Zig рекомендуется следовать принципу "минимальной необходимой доступности":

1. Делайте публичными (`pub`) только те поля и методы, которые действительно должны быть доступны извне.
2. Для полей, которые не должны напрямую изменяться извне, предоставляйте специальные методы для чтения и изменения.
3. Внутренние вспомогательные методы оставляйте приватными.

Такой подход помогает создавать более надежный и поддерживаемый код, уменьшая риск неправильного использования структуры и облегчая изменение внутренней реализации без влияния на внешний интерфейс.
