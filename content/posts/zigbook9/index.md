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

Внутри фигурных скобок, образующих блок структуры, мы определяем поля — именованные элементы данных с указанием их типов. Каждое поле записывается в формате `имя: тип` и должно заканчиваться запятой (в отличие от языка C, где используется точка с запятой). Важно помнить, что всё определение структуры, как и в C, должно завершаться точкой с запятой после закрывающей фигурной скобки.

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
    std.debug.print("Type of: {}\n", .{@TypeOf(p)});
}
```

Обратите внимание на синтаксис инициализации полей: перед именем поля ставится точка (`.x = 10`). Это специальный синтаксис Zig, который делает код более читаемым и явно указывает, какие поля мы инициализируем.

Этот код создает экземпляр структуры `Point` с координатами (10, 20) и выводит его на экран:

```
Point: (10, 20)
Type of: main.Point
```

Как мы видим тип нашей переменной `p` вывелся как `main.Point`. Указание имени файла, где была объявлена структура позволяет при отладке определить, где располагается наша структура. Это бывает очень полезно при отладке кода.

При определении структуры для некоторых полей мы можем задать начальные значения по умолчанию, используя синтаксис инициализации полей. Тогда при создании экземпляра структуры мы можем не указывать значения для этих полей, а они будут инициализированы по умолчанию:

```zig
const Point = struct {
    x: i32,
    y: i32 = 0,
};

pub fn main() void {
    var p = Point{ .x = 10 };
    std.debug.print("Point: ({d}, {d})\n", .{ p.x, p.y });
}
```

В данном примере мы создаем экземпляр структуры `Point` с координатами (10, 0), так как поле `y` имеет значение по умолчанию 0, то при инициализации мы указываем только значение для поля `x`.

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

* Когда у структуры много полей, и мы хотим предоставить значения по умолчанию
* Когда требуется валидация входных данных
* Когда необходимо выполнить сложную логику при инициализации
* Когда внутри структуры используется динамическое выделение памяти

Если ваша структура содержит поля, требующие освобождения ресурсов (например, динамически выделенную память), вам также понадобится функция деструктора. В Zig, по соглашению, для этой цели используется функция с именем `deinit`:

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

const DynamicArray = struct {
    data: []i32,
    allocator: Allocator,

    pub fn init(allocator: Allocator, size: usize) !DynamicArray {
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
    var gpa = std.heap.GeneralPurposeAllocator(.{}).init;
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var array = try DynamicArray.init(allocator, 10);
    defer array.deinit();

    // Работа с массивом...
    for (0..array.data.len) |i| {
        array.data[i] = @as(i32, @intCast(i)) * 2;
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

Также как Вы могли заметить при вызове методов структуры мы используем точечную нотацию (`.`), чтобы вызвать методы структуры. Например, чтобы вызвать метод `area()` у структуры `Rectangle`, мы используем `rect.area()`. Это по сути "синтаксический сахар" для вызова метода структуры, но мы могли бы использовать и кклассический способ вызова функции `Rectangle.area(rect)`, но он менее удобен и не используется.

В Zig указатели на структуры имеют особую особенность - вы можете использовать оператор `.` для доступа к полям структуры через указатель напрямую, без необходимости сначала разыменовывать указатель. В большинстве языков программирования для доступа к полям структуры через указатель используется специальный синтаксис (например, `->` в C/C++). В Zig используется тот же оператор `.` как для обычных экземпляров структур, так и для указателей на структуры:

```zig
const std = @import("std");

const Person = struct {
    name: []const u8,
    age: u32,

    pub fn init(name: []const u8, age: u32) Person {
        return Person{
            .name = name,
            .age = age,
        };
    }
};

pub fn main() void {
    // Создаем экземпляр структуры
    var person = Person.init("John", 30);

    // Создаем указатель на структуру
    var person_ptr: *Person = &person;

    // Доступ к полям через указатель с использованием оператора '.'
    std.debug.print("Name: {s}, Age: {d}\n", .{person_ptr.name, person_ptr.age});

    // Изменение поля через указатель
    person_ptr.age = 31;
    std.debug.print("Updated age: {d}\n", .{person.age});
}
```

В этом примере мы создаем структуру `Person`, создаем экземпляр этой структуры, а затем получаем указатель на этот экземпляр. Затем мы используем оператор `.` для доступа к полям структуры через указатель и для изменения значения поля `age`.

### Цепочка вызовов методов через указатель

Поскольку в Zig можно использовать оператор `.` с указателями, это позволяет создавать цепочки вызовов методов (метод-чейнинг), что особенно полезно для создания реализации паттерна "Строитель":

```zig
const std = @import("std");

const StringBuilder = struct {
    buffer: [1024]u8 = undefined,
    length: usize = 0,

    pub fn init() StringBuilder {
        return StringBuilder{};
    }

    pub fn append(self: *StringBuilder, text: []const u8) *StringBuilder {
        std.mem.copyForwards(u8, self.buffer[self.length..], text);
        self.length += text.len;
        return self; // Возвращаем указатель на self
    }

    pub fn appendInt(self: *StringBuilder, value: i32) *StringBuilder {
        const written = std.fmt.formatIntBuf(self.buffer[self.length..], value, 10, .lower, .{});
        self.length += written;
        return self; // Возвращаем указатель на self
    }

    pub fn toString(self: *const StringBuilder) []const u8 {
        return self.buffer[0..self.length];
    }

    pub fn clear(self: *StringBuilder) *StringBuilder {
        self.length = 0;
        return self; // Возвращаем указатель на self
    }
};

pub fn main() void {
    var builder = StringBuilder.init();

    // Используем цепочку вызовов методов
    const result = builder
        .append("Hello, ")
        .append("the answer is ")
        .appendInt(42)
        .append("!")
        .toString();

    std.debug.print("{s}\n", .{result});

    // Очищаем и создаем новую строку
    const another_result = builder
        .clear()
        .append("Another ")
        .appendInt(123)
        .append(" message")
        .toString();

    std.debug.print("{s}\n", .{another_result});
}
```

В этом примере методы `append`, `appendInt` и `clear` возвращают указатель на `self`, что позволяет цепочкой вызывать методы. Обратите внимание, что мы используем оператор `.` для доступа к методам через указатель, что делает код более читаемым.

### Работа с опциональными указателями

Оператор `.` также можно использовать с опциональными указателями на структуры, но в этом случае вам нужно сначала разыменовать опциональный указатель:

```zig
const std = @import("std");

const Node = struct {
    value: i32,
    next: ?*Node,

    pub fn init(value: i32) Node {
        return Node{
            .value = value,
            .next = null,
        };
    }
};

pub fn main() void {
    var node1 = Node.init(1);
    var node2 = Node.init(2);

    // Связываем узлы
    node1.next = &node2;

    // Доступ к полям через опциональный указатель
    if (node1.next) |next_node| {
        std.debug.print("Next node value: {d}\n", .{next_node.value});
    }

    // Альтернативный способ доступа
    if (node1.next != null) {
        std.debug.print("Next node value: {d}\n", .{node1.next.?.value});
    }
}
```

В этом примере мы создаем два узла связного списка и связываем их. Затем мы демонстрируем два способа доступа к полям через опциональный указатель: с помощью оператора `|next_node|` и с помощью оператора `?`.

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

## Функция @fieldParentPtr и взаимодействие с вложенными структурами

Одной из мощных встроенных функций в Zig является `@fieldParentPtr`, которая позволяет получить указатель на родительскую структуру по указателю на одно из ее полей. Эта функция особенно полезна при реализации сложных структур данных, обработчиков событий и компонентно-ориентированных архитектур.

Функция `@fieldParentPtr` принимает два аргумента:
1. Имя поля как строковый литерал
2. Указатель на поле структуры

Она возвращает указатель на родительскую структуру. Синтаксис:

```zig
@fieldParentPtr("field_name", field_pointer)
```

Одним из наиболее распространенных применений `@fieldParentPtr` является реализация обработчиков событий или callback-функций. Рассмотрим пример с простой системой обработки событий:

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

// Определение типа обработчика событий
const EventHandler = struct {
    // Указатель на функцию обработки событий
    handleFn: *const fn (handler: *EventHandler, event: []const u8) void,

    // Обработка события
    pub fn handle(self: *EventHandler, event: []const u8) void {
        self.handleFn(self, event);
    }
};

// Структура кнопки, включающая обработчик событий
const Button = struct {
    label: []const u8,
    // Встроенный обработчик событий
    event_handler: EventHandler,

    pub fn init(label: []const u8) Button {
        return Button{
            .label = label,
            // Инициализируем обработчик событий с нашей функцией
            .event_handler = EventHandler{
                .handleFn = buttonHandleEvent,
            },
        };
    }

    pub fn click(self: *Button) void {
        std.debug.print("Button '{s}' clicked!\n", .{self.label});
        // Вызываем обработчик события
        self.event_handler.handle("click");
    }

    // Функция обработки событий для кнопки
    fn buttonHandleEvent(handler: *EventHandler, event: []const u8) void {
        // Получаем указатель на родительскую структуру Button
        // из указателя на вложенное поле event_handler
        const button: *Button = @fieldParentPtr("event_handler", handler);
        std.debug.print("Event '{s}' handled by button '{s}'\n", .{ event, button.label });
    }
};

pub fn main() void {
    var submit_button = Button.init("Submit");
    var cancel_button = Button.init("Cancel");

    submit_button.click();
    cancel_button.click();
}
```

В этом примере мы создали структуру `Button`, которая содержит поле `event_handler` типа `EventHandler`. Когда происходит событие (нажатие кнопки), вызывается метод `handle` обработчика событий, который, в свою очередь, вызывает функцию `buttonHandleEvent`.

Внутри функции `buttonHandleEvent` мы используем `@fieldParentPtr` для получения указателя на родительскую структуру `Button` из указателя на поле `event_handler`. Это позволяет нам получить доступ к полям родительской структуры (в данном случае, к полю `label`).

### Использование @fieldParentPtr для реализации примесей (mixins)

`@fieldParentPtr` также может использоваться для реализации примесей (mixins) — механизма, который позволяет добавить функциональность структуре через композицию:

```zig
const std = @import("std");

// Базовая примесь для хранения имени
const NamedMixin = struct {
    name: []const u8,

    pub fn getName(self: *NamedMixin) []const u8 {
        return self.name;
    }
};

// Базовая примесь для хранения возраста
const AgedMixin = struct {
    age: u32,

    pub fn getAge(self: *AgedMixin) u32 {
        return self.age;
    }

    pub fn isAdult(self: *AgedMixin) bool {
        return self.age >= 18;
    }
};

// Структура, использующая примеси
const Person = struct {
    named_part: NamedMixin,
    aged_part: AgedMixin,

    pub fn init(name: []const u8, age: u32) Person {
        return Person{
            .named_part = NamedMixin{ .name = name },
            .aged_part = AgedMixin{ .age = age },
        };
    }

    // Функция-помощник для получения указателя на Person из NamedMixin
    pub fn fromNamed(named: *NamedMixin) *Person {
        return @alignCast(@fieldParentPtr("named_part", named));
    }

    // Функция-помощник для получения указателя на Person из AgedMixin
    pub fn fromAged(aged: *AgedMixin) *Person {
        return @alignCast(@fieldParentPtr("aged_part", aged));
    }

    // Метод, использующий обе примеси
    pub fn introduce(self: *Person) void {
        std.debug.print("Hello, my name is {s} and I am {d} years old.\n", .{ self.named_part.getName(), self.aged_part.getAge() });

        if (self.aged_part.isAdult()) {
            std.debug.print("I am an adult.\n", .{});
        } else {
            std.debug.print("I am not an adult yet.\n", .{});
        }
    }
};

pub fn main() void {
    var person = Person.init("Alice", 25);
    person.introduce();

    // Можно также использовать функции примесей напрямую
    std.debug.print("Name: {s}\n", .{person.named_part.getName()});
    std.debug.print("Age: {d}\n", .{person.aged_part.getAge()});

    // Демонстрация использования функций-помощников
    const named_ref = &person.named_part;
    var person_from_named = Person.fromNamed(named_ref);
    std.debug.print("Retrieved person's age: {d}\n", .{person_from_named.aged_part.getAge()});

    const aged_ref = &person.aged_part;
    var person_from_aged = Person.fromAged(aged_ref);
    std.debug.print("Retrieved person's name: {s}\n", .{person_from_aged.named_part.getName()});
}
```

В этом примере мы создали две примеси (`NamedMixin` и `AgedMixin`), которые реализуют отдельные аспекты функциональности. Структура `Person` включает обе примеси как поля и предоставляет функции-помощники `fromNamed` и `fromAged`, которые используют `@fieldParentPtr` для получения указателя на `Person` из указателя на примесь.

### Реализация интрузивных структур данных

`@fieldParentPtr` также часто используется при реализации интрузивных структур данных, где элементы списка или дерева содержат узлы внутри себя, а не узлы содержат данные. Вот пример интрузивного связного списка:

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

// Интрузивный узел списка
const ListNode = struct {
    next: ?*ListNode = null,

    // Функция для получения следующего узла
    pub fn getNext(self: *ListNode) ?*ListNode {
        return self.next;
    }

    // Функция для установки следующего узла
    pub fn setNext(self: *ListNode, node: ?*ListNode) void {
        self.next = node;
    }
};

// Структура интрузивного списка
const IntrusiveList = struct {
    head: ?*ListNode = null,

    // Добавляет узел в начало списка
    pub fn prepend(self: *IntrusiveList, node: *ListNode) void {
        node.next = self.head;
        self.head = node;
    }

    // Выводит все элементы списка
    pub fn printAll(self: *IntrusiveList, comptime T: type, comptime field_name: []const u8) void {
        var current = self.head;
        while (current) |node| {
            // Получаем родительскую структуру из узла
            const item: *T = @fieldParentPtr(field_name, node);
            std.debug.print("{any}\n", .{item.*});
            current = node.next;
        }
    }
};

// Пример структуры, которая будет использоваться в списке
const Task = struct {
    id: u32,
    name: []const u8,
    // Интрузивный узел как часть структуры
    list_node: ListNode = ListNode{},

    pub fn init(id: u32, name: []const u8) Task {
        return Task{
            .id = id,
            .name = name,
        };
    }
};

pub fn main() void {
    var list = IntrusiveList{};

    var task1 = Task.init(1, "Wash dishes");
    var task2 = Task.init(2, "Clean room");
    var task3 = Task.init(3, "Do homework");

    // Добавляем задачи в список
    list.prepend(&task3.list_node);
    list.prepend(&task2.list_node);
    list.prepend(&task1.list_node);

    // Выводим все задачи
    std.debug.print("Tasks in the list:\n", .{});
    list.printAll(Task, "list_node");
}
```

В этом примере мы создали структуру `Task`, которая содержит поле `list_node` типа `ListNode`. При добавлении задачи в список мы передаем указатель на поле `list_node`, а не на всю структуру `Task`.

Когда нам нужно получить информацию о задаче из узла списка, мы используем `@fieldParentPtr` для получения указателя на структуру `Task` из указателя на поле `list_node`. Это позволяет реализовать интрузивный список, который не требует дополнительного выделения памяти для узлов списка.

Функция `@fieldParentPtr` предоставляет мощный механизм для работы со вложенными структурами в Zig. Она позволяет реализовать многие паттерны проектирования, которые обычно требуют более сложных языковых конструкций в других языках программирования. Однако при использовании `@fieldParentPtr` следует соблюдать осторожность, поскольку неправильное использование может привести к неопределенному поведению:

* **Проверяйте тип родительской структуры** - убедитесь, что тип, указанный в первом аргументе, соответствует фактическому типу родительской структуры.
* **Проверяйте имя поля** - убедитесь, что имя поля, указанное во втором аргументе, соответствует фактическому имени поля в родительской структуре.
* **Проверяйте указатель на поле** - убедитесь, что указатель, переданный в третьем аргументе, действительно указывает на поле родительской структуры.

Для более безопасного использования `@fieldParentPtr` рекомендуется создавать вспомогательные функции или методы, которые абстрагируют вызов `@fieldParentPtr` и делают код более читаемым и менее подверженным ошибкам.

## Модификатор доступа методов структуры
В Zig, ключевое слово `pub` (от англ. "public" - публичный) служит модификатором доступа, который определяет, видимы ли поля и методы структуры за её пределами. Это важный механизм для контроля доступа к данным и реализации инкапсуляции.

### Публичные и приватные методы
По умолчанию, все методы структуры в Zig являются приватными - они доступны только внутри структуры. Если вы хотите сделать метод доступным извне, вы должны пометить его ключевым словом `pub`:

```zig
const std = @import("std");

const Counter = struct {
    value: u32,
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

    // Следующая строка вызовет ошибку компиляции, так как validate - приватный метод
    // counter.validate();
}
```

В этом примере:
- `init`, `increment` и `reset` - публичные методы, которые можно вызывать извне
- `validate` - приватный метод, который можно вызывать только из других методов структуры

При проектировании структур в Zig рекомендуется следовать принципу "минимальной необходимой доступности":

1. Делайте публичными (`pub`) только те методы, которые действительно должны быть доступны извне.
3. Внутренние вспомогательные методы оставляйте приватными.

Такой подход помогает создавать более надежный и поддерживаемый код, уменьшая риск неправильного использования структуры и облегчая изменение внутренней реализации без влияния на внешний интерфейс.

## Анонимные структуры
В Zig также существуют анонимные структуры - структуры без имени. Они могут быть полезны, когда вам нужно создать временную структуру для однократного использования:

```zig
const std = @import("std");

pub fn main() void {
    const point = .{
        .x = 10,
        .y = 20,
    };

    std.debug.print("Point: ({d}, {d})\n", .{ point.x, point.y });

    // Тип анонимной структуры выводится автоматически
    std.debug.print("Type of point: {}\n", .{@TypeOf(point)});
}
```

Анонимные структуры особенно полезны при работе с функциями, которые принимают или возвращают несколько значений. Например:

```zig
const std = @import("std");

fn divideWithRemainder(a: i32, b: i32) struct { quotient: i32, remainder: i32 } {
    return .{
        .quotient = @divFloor(a, b),
        .remainder = @rem(a, b),
    };
}

pub fn main() void {
    const result = divideWithRemainder(17, 5);
    std.debug.print("17 ÷ 5 = {d} remainder {d}\n", .{ result.quotient, result.remainder });
}
```

## Структуры как пространства имен (namespace)
В Zig структуры могут использоваться не только для группировки данных, но и как пространства имен (namespace) для функций, которые логически связаны между собой. Это особенно полезно, когда вы хотите организовать набор функций, относящихся к определенной области или функциональности, без необходимости создавать отдельные экземпляры структуры.

### Создание пространства имен с помощью структуры

Структура без полей может служить контейнером для связанных функций:

```zig
const std = @import("std");

const Math = struct {
    pub fn abs(x: i32) i32 {
        return if (x >= 0) x else -x;
    }

    pub fn max(a: i32, b: i32) i32 {
        return if (a > b) a else b;
    }

    pub fn min(a: i32, b: i32) i32 {
        return if (a < b) a else b;
    }

    pub fn isPrime(n: u32) bool {
        if (n <= 1) return false;
        if (n <= 3) return true;
        if (n % 2 == 0 or n % 3 == 0) return false;

        var i: u32 = 5;
        while (i * i <= n) : (i += 6) {
            if (n % i == 0 or n % (i + 2) == 0) return false;
        }
        return true;
    }
};

pub fn main() void {
    const a = -5;
    const b = 10;

    std.debug.print("Absolute value of {d} is {d}\n", .{ a, Math.abs(a) });
    std.debug.print("Maximum of {d} and {d} is {d}\n", .{ a, b, Math.max(a, b) });
    std.debug.print("Minimum of {d} and {d} is {d}\n", .{ a, b, Math.min(a, b) });

    const num = 17;
    std.debug.print("Is {d} prime? {}\n", .{ num, Math.isPrime(num) });

    std.debug.print("Size of Math {}\n", .{ @sizeOf(Math) });
}
```

Этот код выведет:
```
Absolute value of -5 is 5
Maximum of -5 and 10 is 10
Minimum of -5 and 10 is -5
Is 17 prime? true
Size of Math 0
```

В этом примере структура `Math` используется как пространство имен для группировки математических функций. Обратите внимание, что мы не создаем экземпляр структуры, а вызываем функции напрямую через имя структуры: `Math.abs()`, `Math.max()` и т.д. Также мы можем увидеть что использование структур как пространств имен не приводит к дополнительному использованию памяти - размер нашей структуры Math равен 0, так как она не содержит данных.

### Вложенные пространства имен

Вы также можете создавать вложенные пространства имен, определяя структуры внутри других структур:

```zig
const std = @import("std");

const Geometry = struct {
    pub const PI = 3.14159265359;

    pub const Circle = struct {
        pub fn area(radius: f32) f32 {
            return PI * radius * radius;
        }

        pub fn circumference(radius: f32) f32 {
            return 2 * PI * radius;
        }
    };

    pub const Rectangle = struct {
        pub fn area(width: f32, height: f32) f32 {
            return width * height;
        }

        pub fn perimeter(width: f32, height: f32) f32 {
            return 2 * (width + height);
        }
    };

    pub const Triangle = struct {
        pub fn area(base: f32, height: f32) f32 {
            return 0.5 * base * height;
        }

        pub fn areaFromSides(a: f32, b: f32, c: f32) f32 {
            // Формула Герона
            const s = (a + b + c) / 2;
            return @sqrt(s * (s - a) * (s - b) * (s - c));
        }
    };
};

pub fn main() void {
    const radius = 5.0;
    std.debug.print("Circle area (r={d}): {d:.2}\n", .{
        radius,
        Geometry.Circle.area(radius)
    });
    std.debug.print("Circle circumference (r={d}): {d:.2}\n", .{
        radius,
        Geometry.Circle.circumference(radius)
    });

    const width = 10.0;
    const height = 5.0;
    std.debug.print("Rectangle area ({d}x{d}): {d:.2}\n", .{
        width,
        height,
        Geometry.Rectangle.area(width, height)
    });

    const a = 3.0;
    const b = 4.0;
    const c = 5.0;
    std.debug.print("Triangle area from sides ({d}, {d}, {d}): {d:.2}\n", .{
        a,
        b,
        c,
        Geometry.Triangle.areaFromSides(a, b, c)
    });
}
```

В этом примере структура `Geometry` содержит константу `PI` и три вложенных пространства имен: `Circle`, `Rectangle` и `Triangle`. Каждое из этих вложенных пространств имен содержит функции, связанные с соответствующей геометрической фигурой.

Использование структур как пространств имен - мощный инструмент для организации кода в Zig. Это позволяет создавать четкие, модульные и хорошо организованные API, даже без использования полноценной объектно-ориентированной парадигмы.

В отличие от некоторых других языков, где пространства имен являются отдельной языковой конструкцией, Zig повторно использует механизм структур, что делает язык более компактным и последовательным. Это хороший пример философии Zig: делать больше с меньшим количеством языковых конструкций.

Использование структур как пространства для функций полезно в следующих случаях:

* **Организация кода**: Структуры помогают группировать связанные функции, делая код более организованным и читаемым.
* **Предотвращение конфликтов имен**: Объединение функций в пространства имен помогает избежать конфликтов имен в больших проектах.
* **Ясное намерение**: Использование структур как пространств имен явно показывает, что функции концептуально связаны, даже если они не разделяют общие данные.
* **Гибкость API**: Вы можете постепенно добавлять новые функции в пространство имен без необходимости менять существующий код.

## Заключение

Структуры в Zig являются мощным механизмом для организации и группировки данных. Они позволяют создавать собственные типы данных, добавлять к ним методы и моделировать реальные объекты в вашей программе. Хотя Zig не является объектно-ориентированным языком, структуры и их методы предоставляют многие из преимуществ ООП, сохраняя при этом простоту и эффективность языка.
