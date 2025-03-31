---
title: HashMap
date: 2025-03-26 15:00:00
showTableOfContents: true
showComments: true
katex: true
tags:
  - zig
  - zigbook
  - hashmap
---

Хеш-карта (HashMap) - это одна из самых полезных структур данных в программировании, предоставляющая ассоциативный массив с возможностью быстрого доступа к элементам по ключу за константное время (O(1)). Во всех языках программирования есть такая структура данных, так как разнообразие задач, где она применяется очень велико. В языке Python такая структура называется dict, в языке Rust - HashMap, в языке C++ - std::map или std::unordered_map. Прежде чем рассматривать реализацию HashMap в Zig, давайте познакомимся с основными понятиями и свойствами этой структуры данных.

## Основные понятия и свойства HashMap
HashMap — это структура данных, состоящая из пар «ключ-значение», где ключ уникален и используется для быстрого доступа к соответствующему значению. При добавлении элемента в HashMap передается пара: ключ и значение. В результате значение сохраняется в структуре и становится доступным по указанному ключу.

HashMap основан на хеш-функции, которая преобразует ключ в целое число. Это число определяет индекс, по которому элемент хранится в массиве. Благодаря такому подходу доступ к элементам осуществляется за константное время O(1), даже если массив, лежащий в основе структуры, неупорядочен.

Внутри HashMap используется массив, элементы которого называют корзинами (buckets). В каждой корзине может находиться:

* null, если элемента нет
* элемент, если элемент есть
* список элементов (LinkedList), если в корзине есть коллизия

При вставке элемента в HashMap хеш-функция вычисляет индекс корзины. Однако разные ключи могут давать одинаковый индекс (это называется коллизией). В таком случае в ячейке корзины хранится не один элемент, а связанный список всех элементов с этим индексом. Это позволяет корректно обрабатывать коллизии и обеспечивать быстрый доступ к данным. Схематично это можно представить так:

{{< figure src="hashmap_dark.svg" class="post-image-dark small">}}
{{< figure src="hashmap_light.svg" class="post-image-light small">}}

Когда уровень заполненности массива бакетов достигает определенного порога (обычно 60–75%), HashMap увеличивает его размер (как правило, в 2 раза) и перераспределяет элементы. Этот процесс называется rehashing и может быть затратным по времени. Поэтому, если известно примерное количество элементов, рекомендуется заранее задавать начальную емкость (capacity), чтобы уменьшить количество перераспределений.

При работе с HashMap используются три основные операции: вставка, поиск и удаление. Благодаря своей структуре в среднем случае каждая из них выполняется за O(1), что делает HashMap эффективным инструментом для быстрого доступа к данным.

## Реализация HashMap в Zig
Стандартная библиотека языка Zig предоставляет нам сразу несколько вариантов реализации HashMap. Каждая из реализаций имеет свои плюсы и минусы и может быть использована в зависимости от конкретных требований. Часть реализаций HashMap можно найти в модуле `std.hash_map`, а часть (ArrayHashMap, StringArrayHashMap) расположены просто в корне стандартной библиотеки. В целом здесь подход такой же как мы видели и при изучении аллокаторов - для разных задач мы можем использовать разные реализации.

### Unmanaged типы
Если открыть модуль `std.hash_map`, то можно увидеть, что там также есть типы данных с приставкой `Unmanaged`. Это не отдельный тип хеш-таблиц, а стандартное правило именования типов данных, которые не управляют памятью самостоятельно.

Чтобы понять, о чем идет речь, давайте рассмотрим пример списка пользователей:

```zig
const std = @import("std");

const UserItem = struct {
    name: []const u8,
    id: u64,
};

const UserList = struct {
    head: ?*Node = null,
    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator) !UserList {
        return UserList{
            .allocator = allocator,
        };
    }

    pub fn deinit(self: *UserList) void {
        var node = self.head;
        while (node) |n| {
            node = n.next;
            self.allocator.free(n.value.name);

            self.allocator.destroy(n);
        }
    }

    pub fn addUser(self: *UserList, id: u64, name: []const u8) !void {
        const new_node = try self.allocator.create(Node);
        new_node.* = Node{
            .value = UserItem{
                .name = try self.allocator.dupe(u8, name),
                .id = id,
            },
            .next = self.head,
        };
        self.head = new_node;
    }

    pub fn removeUser(self: *UserList, id: u64) !void {
        var prev: ?*Node = null;
        var node = self.head;
        while (node) |n| {
            if (n.value.id == id) {
                if (prev) |p| {
                    p.next = n.next;
                } else {
                    self.head = n.next;
                }
                self.allocator.free(n.value.name);
                self.allocator.destroy(n);
                return;
            }
            prev = node;
            node = n.next;
        }
        return error.UserNotFound;
    }

    pub const Node = struct {
        value: UserItem,
        next: ?*Node,
    };
};

pub fn main() !void {
    var dalloc = std.heap.DebugAllocator(.{}).init;
    defer _ = dalloc.deinit();
    const allocator = dalloc.allocator();
    var user_list = try UserList.init(allocator);

    try user_list.addUser(1, "Alice");
    try user_list.addUser(2, "Bob");

    try user_list.removeUser(1);

    user_list.deinit();
}
```

В нашем примере, мы передаем аллокатор при создании нашего списка, он сохраняется внутри структуры списка и затем используется для аллокации памяти для элементов списка, а также в методе `deinit`. Это в целом стандартный подход, который естественно используется при решении такой задачи. Но давайте теперь попробуем разбить нашу структуру на две `UserListUnmanaged` и `UserList` и посмотрим что из этого получится:

```zig
const std = @import("std");

const UserItem = struct {
    name: []const u8,
    id: u64,
};

const UserList = struct {
    allocator: std.mem.Allocator,
    unmanaged: UserListUnmanaged,

    pub fn init(allocator: std.mem.Allocator) !UserList {
        return UserList{
            .allocator = allocator,
            .unmanaged = .{},
        };
    }

    pub fn deinit(self: *UserList) void {
        self.unmanaged.deinit(self.allocator);
    }

    pub fn addUser(self: *UserList, id: u64, name: []const u8) !void {
        try self.unmanaged.addUser(self.allocator, id, name);
    }

    pub fn removeUser(self: *UserList, id: u64) !void {
        try self.unmanaged.removeUser(self.allocator, id);
    }
};

const UserListUnmanaged = struct {
    head: ?*Node = null,

    pub fn deinit(self: *UserListUnmanaged, allocator: std.mem.Allocator) void {
        var node = self.head;
        while (node) |n| {
            node = n.next;
            allocator.free(n.value.name);

            allocator.destroy(n);
        }
    }

    pub fn addUser(self: *UserListUnmanaged, allocator: std.mem.Allocator, id: u64, name: []const u8) !void {
        const new_node = try allocator.create(Node);
        new_node.* = Node{
            .value = UserItem{
                .name = try allocator.dupe(u8, name),
                .id = id,
            },
            .next = self.head,
        };
        self.head = new_node;
    }

    pub fn removeUser(self: *UserListUnmanaged, allocator: std.mem.Allocator, id: u64) !void {
        var prev: ?*Node = null;
        var node = self.head;
        while (node) |n| {
            if (n.value.id == id) {
                if (prev) |p| {
                    p.next = n.next;
                } else {
                    self.head = n.next;
                }
                allocator.free(n.value.name);
                allocator.destroy(n);
                return;
            }
            prev = node;
            node = n.next;
        }
        return error.UserNotFound;
    }

    pub const Node = struct {
        value: UserItem,
        next: ?*Node,
    };
};

pub fn main() !void {
    var dalloc = std.heap.DebugAllocator(.{}).init;
    defer _ = dalloc.deinit();
    const allocator = dalloc.allocator();
    var user_list = try UserList.init(allocator);

    try user_list.addUser(1, "Alice");
    try user_list.addUser(2, "Bob");

    try user_list.removeUser(1);

    user_list.deinit();
}
```

В новой реализации наш тип `UserList` по сути стал оберткой вокруг типа `UserListUnmanaged`. Вся логика нашего списка пользователей сосредоточена в типе `UserListUnmanaged`, у которого есть особенность - он не хранит аллокатор внутри себя, а ожидают что он будет передан в каждый метод. Это довольно типичный шаблон проектирования для стандартной библиотеки Zig, которые дает следующие преимущества:

* В методах типа `UserListUnmanaged` мы более явно по сигнатуре метода видим, есть ли внутри него работа с аллокациями или нет.
* Тип `UserListUnmanaged` не хранит аллокатор внутри себя и тем самым экономит немного памяти, если Вам нужно хранить большое (миллионы) количество таких типов в приложении.
* Вы можетет инициализировать тип `UserListUnmanaged` с помощью короткого варианта инициализации типа `.{}`.

Все типы с приставкой `Unmanaged` по сути являются базовыми типами для хеш-таблиц, где эта приставка отсутсвует, так например `HashMapUnmanaged` является базовым типом для `HashMap`, а `StringHashMapUnmanaged` является базовым типом для `StringHashMap`. Если заглянуть в основные типы хеш-таблиц, то можно увидеть, что большая часть их методов является "тонкой" оберткой вокруг методов из `Unmanaged` типа.

### HashMap и AutoHashMap
Вся внутренняя работа структуры HashMap построена на двух функциях - `hash(key: K) u64` и `eql(key_a: K, key_b: K) bool`. Первая функция вычисляет хеш ключа в виде 64-битного целого числа, а вторая функция проверяет равенство двух ключей и нужна для того, чтобы эффективно обрабатывать коллизии. При инициализации какой-либо HashMap мы можем передать эти функции в качестве аргументов, изменив логику вычисления хеша и сравнения ключей.

Для того, чтобы создать стандартную хеш-таблицу, нужно передать типу `HashMap` несколько параметров - типы ключа и значения, аллокатор и контекст. Контекст, передаваемый в наш тип как раз и нужен для того, чтобы мы могли передать функции `hash` и `eql` в качестве аргументов при инициализации хеш-таблицы. Если вы используете простые типы данных в вашей хеш-карте, то нет смысла каждый раз самим писать функции `hash` и `eql`, а можно инициализировать хеш-карту с помощью `AutoHashMap`. Тогда функции `hash` и `eql` будут автоматически сгенерированы на основе типа ключа:

```zig
const std = @import("std");

const User = struct {
    id: i32,
    balance: usize,
};

pub fn main() !void {
    var gpa = std.heap.DebugAllocator(.{}).init;
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var map = std.AutoHashMap(i32, User).init(allocator);
    defer map.deinit();

    const user: User = .{ .id = 1, .balance = 100 };
    try map.put(1, user);

    std.debug.print("User {d} ({d})\n", map.get(1).?);
}
```

В нашем примере мы создаем хеш-таблицу с ключом `i32` и значением `User`. Тип ключа `i32` это примитивный тип и наша хеш-таблица автоматически сгенерирует для этого типа стандартные функции `hash` и `eql`. Но это не означает что мы можем использовать только примитивные типы в качестве ключей. Мы также можем использовать пользовательские типы. Давайте поменяем местами ключ и значение в нашей таблицы и посмотрим, что будет:

```zig
const std = @import("std");

const User = struct {
    id: i32,
    balance: usize,
};

pub fn main() !void {
    var gpa = std.heap.DebugAllocator(.{}).init;
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var map = std.AutoHashMap(User, i32).init(allocator);
    defer map.deinit();

    const user: User = .{ .id = 1, .balance = 100 };
    try map.put(user, 1);

    std.debug.print("User id {d}\n", .{map.get(user).?});
}
```

Как мы видим наша таблица продолжает работать корректно, даже если мы используем пользовательский тип в качестве ключа. Давайте продолжим развивать наш код и добавим имя пользователя к нашей структуре:

```zig
const std = @import("std");

const User = struct {
    id: i32,
    name: []const u8,
    balance: usize,
};

pub fn main() !void {
    var gpa = std.heap.DebugAllocator(.{}).init;
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var map = std.AutoHashMap(User, i32).init(allocator);
    defer map.deinit();

    const user: User = .{ .id = 1, .name = "John Doe", .balance = 100 };
    try map.put(user, 1);

    std.debug.print("User id {d}\n", .{map.get(user).?});
}
```

Если мы попытаемся скомпилировать наш код, то мы получим ошибку:

```
run
└─ run simple
   └─ install
      └─ install simple
         └─ zig build-exe simple Debug native 1 errors
/Users/roman/.zvm/0.14.0/lib/std/hash/auto_hash.zig:192:9: error: std.hash.autoHash does not allow slices as well as unions and structs containing slices here (main.User) because the intent is unclear. Consider using std.hash.autoHashStrat or providing your own hash function instead.
```

### ArrayHashMap

### StringHashMap

### StringArrayHashMap
