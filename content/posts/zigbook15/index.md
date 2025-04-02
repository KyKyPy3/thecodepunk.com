---
title: Хеш-карты (HashMap)
date: 2025-03-26 15:00:00
showTableOfContents: true
showComments: true
katex: true
tags:
  - zig
  - zigbook
  - hashmap
---

Хеш-карта (HashMap) - это одна из самых полезных структур данных в программировании, предоставляющая ассоциативный массив с возможностью быстрого доступа к элементам по ключу за константное время (O(1)). Во всех языках программирования есть такая структура данных, так как разнообразие задач, где она применяется очень велико. Приличная часть задач на LeetCode решается с использованием HashMap. В языке Python такая структура называется dict, в языке Rust - HashMap, в языке C++ - std::map или std::unordered_map. Прежде чем рассматривать реализацию HashMap в Zig, давайте познакомимся с основными понятиями и свойствами этой структуры данных.

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

Когда уровень заполненности массива бакетов достигает определенного порога (обычно 60–80%), HashMap увеличивает его размер (как правило, в 2 раза) и перераспределяет элементы. Этот процесс называется rehashing и может быть затратным по времени. Поэтому, если известно примерное количество элементов, рекомендуется заранее задавать начальную емкость (capacity), чтобы уменьшить количество перераспределений.

Приведенный здесь пример реализации HashMap условный, так как детали внутренней реализации хеш-таблицы в разных языках программирования могут отличатся. Так например в Go хеш-таблица реализована в виде массива, который хранит пары ключ-значения вместе, в языке Zig используются два массива - один для хранения ключей, другой для хранения значений. Но при этом особенности реализации не влияют на характеристики работы данной структуры данных.

При работе с HashMap используются три основные операции: вставка, поиск и удаление. Благодаря своей структуре в среднем случае каждая из них выполняется за O(1), что делает HashMap эффективным инструментом для быстрого доступа к данным.

## Реализация HashMap в Zig
Стандартная библиотека языка Zig предоставляет нам сразу несколько вариантов реализации HashMap. Каждая из реализаций имеет свои плюсы и минусы и может быть использована в зависимости от конкретных требований. Часть реализаций HashMap можно найти в модуле `std.hash_map`, а часть (ArrayHashMap, StringArrayHashMap) расположены просто в корне стандартной библиотеки. В целом здесь подход такой же как мы видели и при изучении аллокаторов - для разных задач мы можем использовать разные реализации. Но прежде чем мы перейдем к рассмотрению различных реализаций HashMap, давайте познакомимся с Unmanaged типами.

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

В нашем примере, мы передаем аллокатор при создании нашего списка, он сохраняется внутри структуры списка и затем используется для аллокации памяти для элементов списка, а также в методе `deinit`. Это в целом стандартный подход, который естественно используется при решении такой задачи. Но давайте теперь попробуем разбить нашу структуру на две `UserListUnmanaged` и `UserList` и посмотрим, что из этого получится:

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

В новой реализации наш тип `UserList` по сути стал оберткой вокруг типа `UserListUnmanaged`. Вся логика нашего списка пользователей сосредоточена в типе `UserListUnmanaged`, у которого есть особенность - он не хранит аллокатор внутри себя, а ожидает, что он будет передан в каждый метод. Это довольно типичный шаблон проектирования для стандартной библиотеки Zig, которые дает следующие преимущества:

* В методах типа `UserListUnmanaged` мы более явно по сигнатуре метода видим, есть ли внутри него работа с аллокациями или нет.
* Тип `UserListUnmanaged` не хранит аллокатор внутри себя и тем самым экономит немного памяти, если Вам нужно хранить большое (миллионы) количество таких типов в приложении.
* Вы можетет инициализировать тип `UserListUnmanaged` с помощью короткого варианта инициализации типа `.{}`.

Все типы с приставкой `Unmanaged` по сути являются базовыми типами для хеш-таблиц, где эта приставка отсутсвует, так например `HashMapUnmanaged` является базовым типом для `HashMap`, а `StringHashMapUnmanaged` является базовым типом для `StringHashMap`. Если заглянуть в основные типы хеш-таблиц, то можно увидеть, что большая часть их методов является "тонкой" оберткой вокруг методов из `Unmanaged` типа.

### AutoHashMap
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

В нашем примере мы создаем хеш-таблицу с ключом `i32` и значением `User`. Тип ключа `i32` это примитивный тип и наша хеш-таблица автоматически сгенерирует для этого типа стандартные функции `hash` и `eql`. Это значительно упрощает использование хеш-таблиц для повседневных задач, когда мы работаем с примитивными типами данных или простыми структурами.

Но это не означает, что мы можем использовать только примитивные типы в качестве ключей. Мы также можем использовать пользовательские типы. Давайте поменяем местами ключ и значение в нашей таблицы и посмотрим, что будет:

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

Как мы видим наша таблица продолжает работать корректно, даже если мы используем пользовательский тип в качестве ключа. Это происходит потому, что структура `User` содержит только поля примитивных типов, для которых есть стандартные реализации функций хеширования и сравнения. `AutoHashMap` может автоматически обрабатывать такие структуры, рекурсивно применяя функции хеширования к каждому полю.

Давайте продолжим развивать наш код и добавим имя пользователя к нашей структуре:

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
/Users/roman/.zvm/0.14.0/lib/std/hash/auto_hash.zig:192:9: error: std.hash.autoHash does not allow slices as well as unions and
structs containing slices here (main.User) because the intent is unclear. Consider using std.hash.autoHashStrat or providing
your own hash function instead.
```

Но почему вдруг наш код перестал компилироваться? Это связано с тем, что структура `User` содержит поле name, которое является срезом (slice) типа u8. Срезы не могут быть использованы в качестве ключей в `AutoHashMap`, так как их хэш-функция не может быть сгенерирована однозначно. Так как строки это по сути указатели на области памяти, то тут сразу возникает вопрос что означает "строки равны". Можем ли мы считать, что строки равные если равны их длины и содержимое, или нам нужно чтобы адреса, по которым располагаются строки были одинаковыми. Чтобы работа хеш-таблицы не приводила к неожиданному поведению, некоторые типы данных нельзя использовать с `AutoHashMap`. В этом случае, либо можно использовать специализированный тип хеш-таблицы (`StringHashMap`), либо самому определеить функции `hash` и `eql` для нашего типа `User`.

Это ограничение связано с тем, что в Zig срезы могут сравниваться по-разному в зависимости от контекста. Когда мы сравниваем два среза, мы можем сравнивать их адреса в памяти (если мы хотим знать, указывают ли они на одну и ту же область памяти) или их содержимое (если нас интересует, содержат ли они одинаковые данные). Эта неоднозначность делает невозможным автоматическое хеширование и сравнение срезов без дополнительных указаний.

### StringHashMap
Когда мы используем в качестве ключа нашей хеш-таблицы строку, то у нас есть два способа сравнения строк - `std.meta.eql` и `std.mem.eql`. Поведение этих функций отличается, так как `std.meta.eql` сравнивает строки по их адресам, а `std.mem.eql` сравнивает строки по их содержимому. Так как в реальной жизни большинству разработчиков не нужно сравнивать строки по их адресам, то чаще всего используется `std.mem.eql`. Для облегчения использования хеш-таблиц со строками, в которых для сравнения используется `std.mem.eql`, можно использовать `StringHashMap`. Давайте рассмотрим пример:

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

    var map = std.StringHashMap(User).init(allocator);
    defer map.deinit();

    const user: User = .{ .id = 1, .name = "John Doe", .balance = 100 };
    try map.put("1", user);

    std.debug.print("User {s} ({d})\n", .{ map.get("1").?.name, map.get("1").?.id });
}
```

В этом примере мы используем хеш-таблицу, где в качестве ключа мы используем строковые литералы и у нас нет ошибки компиляции, так как наш тип хеш-карты `StringHashMap` знает как хешировать и сравнивать строки.

## HashMap
Если ваши типы данных не попадают под возможности `AutoHashMap` и `StringHashMap`, то вам прийдется использовать `HashMap` и нужно будет определить функции `hash` и `eql` для вашего типа данных. В целом в этом нет ничего сложного, давайте рассмотрим пример:

```zig
const std = @import("std");

const User = struct {
    id: i32,
    name: []const u8,

    pub const HashContext = struct {
        pub fn hash(_: HashContext, u: User) u64 {
            var h = std.hash.Wyhash.init(0);
            h.update(u.name);
            h.update(std.mem.asBytes(&u.id));
            return h.final();
        }
        pub fn eql(_: HashContext, a: User, b: User) bool {
            if (a.id != b.id) return false;
            return std.mem.eql(u8, a.name, b.name);
        }
    };
};

pub fn main() !void {
    var gpa = std.heap.DebugAllocator(.{}).init;
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var map = std.HashMap(User, i32, User.HashContext, std.hash_map.default_max_load_percentage).init(allocator);
    defer map.deinit();

    try map.put(.{ .id = 1, .name = "Alice" }, 1);
    try map.put(.{ .id = 2, .name = "Bob" }, 2);

    std.debug.print("User {d}\n", .{map.get(.{ .id = 1, .name = "Alice" }).?});
    std.debug.print("User {d}\n", .{map.get(.{ .id = 2, .name = "Bob" }).?});
}
```

В данном примере мы определили собственный контекст хеширования `HashContext` внутри структуры `User`. Этот контекст содержит две функции:

* `hash`: вычисляет хеш-значение для экземпляра `User`. Мы используем алгоритм Wyhash (один из быстрых хеш-алгоритмов, доступных в стандартной библиотеке Zig), обновляем его значениями полей `name` и `id`, и возвращаем окончательное хеш-значение.

* `eql`: сравнивает два экземпляра `User`. Сначала проверяем, равны ли `id`, и если нет, сразу возвращаем `false`. Если `id` равны, сравниваем содержимое полей `name` с помощью `std.mem.eql`.

Затем мы создаем `HashMap` с типом ключа `User`, типом значения `i32`, контекстом хеширования `User.HashContext` и стандартным максимальным процентом загрузки. Теперь мы можем использовать нашу структуру `User` в качестве ключа в хеш-карте, даже несмотря на то, что она содержит срез.

Этот подход дает нам полный контроль над тем, как хешируются и сравниваются наши пользовательские типы, что особенно полезно для сложных структур данных или когда мы хотим оптимизировать производительность хеш-карты для конкретного случая использования.

Давайте теперь рассмотрим как мы можем работать с хеш-таблицей.

### Работа с элементами
Давайте рассмотрим наиболее частые методы работы с хеш-таблицей:

* `put`: добавляет элемент в хеш-таблицу.
* `get`: получает элемент из хеш-таблицы по ключу.
* `remove`: удаляет элемент из хеш-таблицы по ключу.
* `contains`: проверяет, содержит ли хеш-таблица элемент с заданным ключом.
* `count`: возвращает количество элементов в хеш-таблице.
* `fetchRemove`: возвращает элемент из хеш-таблицы по ключу и удаляет его.

Давайте теперь рассмотрим пример использования этих методов:

```zig
const std = @import("std");

const User = struct {
    id: i32,
    name: []const u8,
};

pub fn main() !void {
    var gpa = std.heap.DebugAllocator(.{}).init;
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var map = std.AutoHashMap(i32, User).init(allocator);
    defer map.deinit();

    try map.put(1, .{ .id = 1, .name = "Alice" });
    try map.put(2, .{ .id = 2, .name = "Bob" });

    std.debug.print("Hashmap contains: {d}\n", .{map.count()});

    const user = map.fetchRemove(1).?;

    std.debug.print("Removed user: {d}\n", .{user.value.id});

    std.debug.print("Hashmap contains user with id 1? {}\n", .{map.contains(1)});

    std.debug.print("User {1s} ({0})\n", map.get(2).?);
}
```

Данный код выведет:

```
Hashmap contains: 2
Removed user: 1
Hashmap contains user with id 1? false
User Bob (2)
```

Как мы можем заметить некоторые методы, такие как `get` и `remove`, могут вернуть `null`, поэтому мы используем оператор `?` для обработки возможных ошибок. Если вы передадите в эти методы ключ элемента, которого нет в хеш-таблице, то она не сможет найти его в массиве и должна вернуть вам `null`. Остальные методы показанные в примере довольно интуитивны и не требуют обьяснения. Давайте теперь попробуем изменить что-то в нашей хеш-таблице:

```zig
const std = @import("std");

const User = struct {
    id: i32,
    name: []const u8,
};

pub fn main() !void {
    var gpa = std.heap.DebugAllocator(.{}).init;
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var map = std.AutoHashMap(i32, User).init(allocator);
    defer map.deinit();

    try map.put(1, .{ .id = 1, .name = "Alice" });
    try map.put(2, .{ .id = 2, .name = "Bob" });

    var user = map.get(1).?;

    user.name = "Alice Smith";

    std.debug.print("User {1s} ({0})\n", map.get(1).?);
}
```

Как мы видим наш план не удался. Значение пользователя хранимое в хеш-карте не изменилось как мы планировали. Все дело в том, что метод `get` возвращает копию значения, а не ссылку на него. Поэтому, когда мы изменяем значение `user.name`, мы изменяем только копию, а не оригинальное значение в хеш-таблице. Для того чтобы изменить значение в хеш-таблице, нам нужно получить ссылку на значение и для этого у таблицы есть метод `getPtr` который возвращает указатель на значение:

```zig
const std = @import("std");

const User = struct {
    id: i32,
    name: []const u8,
};

pub fn main() !void {
    var gpa = std.heap.DebugAllocator(.{}).init;
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var map = std.AutoHashMap(i32, User).init(allocator);
    defer map.deinit();

    try map.put(1, .{ .id = 1, .name = "Alice" });
    try map.put(2, .{ .id = 2, .name = "Bob" });

    const user = map.getPtr(1).?;

    user.name = "Alice Smith";

    std.debug.print("User {1s} ({0})\n", map.get(1).?);
}
```

Теперь наш код выведет:

```
User Alice Smith (1)
```

Но при использовании метода `getPtr` есть одна опасность. Как мы уже упоминали при рассмотрении как устроена хеш-таблица, внутри нее все значения лежат в массиве, который имеет некоторый начальный размер. Если мы добавим в нашу хеш-таблицу много элементов, то в какой-то момент заполнение массива достигнет 80% и таблица начнет процесс реаллокации, выделил новый массив и перенеся туда данные. В этом случаи все ссылки на элементы таблицы, полученные через метод `getPtr`, станут недействительными и мы получим ошибку при попытке использовать их. Давайте рассмотрим пример:

```zig
const std = @import("std");

const User = struct {
    id: usize,
    name: []const u8,
};

pub fn main() !void {
    var gpa = std.heap.DebugAllocator(.{}).init;
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var map = std.AutoHashMap(usize, User).init(allocator);
    defer map.deinit();

    try map.put(1, .{ .id = 1, .name = "Alice" });
    try map.put(2, .{ .id = 2, .name = "Bob" });

    const user = map.getPtr(1).?;

    for (0..50) |i| {
        try map.put(i, .{ .id = i, .name = "User" });
    }

    user.name = "Alice Smith";

    std.debug.print("User {1s} ({0})\n", map.get(1).?);
}
```

Если мы запустим нашу программу, то получим ошибку выполнения:

```
Segmentation fault at address 0x104a20080
aborting due to recursive panic
run
└─ run simple failure
error: the following command terminated unexpectedly:
/Users/roman/Projects/zig/simple/zig-out/bin/simple
Build Summary: 5/7 steps succeeded; 1 failed
run transitive failure
└─ run simple failure
error: the following build command failed with exit code 1
```

Как мы и говорили, ссылка на элемент хеш-таблицы, полученная через метод `getPtr`, может стать недействительной после реаллокации массива, как и получилось в нашем примере. В этом случае мы должны обновить ссылку на элемент хеш-таблицы, чтобы она указывала на актуальный элемент. Но что если нам надо чтобы мы могли изменять элементы хеш-таблицы и нам бы не приходилось постоянно следить за валидностью указателей. Можем ли мы как-то исправить проблему? Да можем. Мы можем хранить в нашей хеш-таблице не только значения, но и указатели на значения. Это позволит нам изменять значения в хеш-таблице без необходимости следить за валидностью указателей. Давайте рассмотрим пример:

```zig
const std = @import("std");

const User = struct {
    id: usize,
    name: []const u8,
};

pub fn main() !void {
    var gpa = std.heap.DebugAllocator(.{}).init;
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var map = std.AutoHashMap(usize, *User).init(allocator);
    defer map.deinit();

    const user1 = try allocator.create(User);
    user1.* = .{ .id = 1, .name = "Alice" };
    try map.put(1, user1);

    const user2 = try allocator.create(User);
    user2.* = .{ .id = 2, .name = "Bob" };
    try map.put(2, user2);

    const first = map.get(1).?;

    for (0..50) |i| {
        const user = try allocator.create(User);
        user.* = .{ .id = i, .name = "User" };
        try map.put(i, user);
    }

    first.name = "Alice Smith";

    std.debug.print("User {s}\n", .{map.get(1).?.name});
}
```

Если мы запустим нашу программу, то получим кучу ошибок о том что у нас утекла память:

```
User User
error(gpa): memory address 0x101040000 leaked:
/Users/roman/Projects/zig/simple/src/main.zig:16:39: 0x100f227df in main (simple)
    const user1 = try allocator.create(User);
```

Когда мы добавляли в нашу хеш-таблицу структуры `User`, по сути наша таблица владела этими структурами и при вызове `deinit`, происходит освобождение массивов в котором таблица хранила структуры `User`. Но если мы передаем в нашу таблицу указатели на структуры `User`, то таблица не владеет этими структурами, а владеет только указателями на них. Следовательно освобождение памяти, занимаемой структуров лежит на нас и мы должны об этом позаботится. Давайте исправим наш код и снова запустим:

```zig
const std = @import("std");

const User = struct {
    id: usize,
    name: []const u8,
};

pub fn main() !void {
    var gpa = std.heap.DebugAllocator(.{}).init;
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var map = std.AutoHashMap(usize, *User).init(allocator);
    defer map.deinit();

    const user1 = try allocator.create(User);
    defer allocator.destroy(user1);
    user1.* = .{ .id = 1, .name = "Alice" };
    try map.put(1, user1);

    const user2 = try allocator.create(User);
    defer allocator.destroy(user2);
    user2.* = .{ .id = 2, .name = "Bob" };
    try map.put(2, user2);

    const first = map.get(1).?;

    for (0..50) |i| {
        const user = try allocator.create(User);
        defer allocator.destroy(user);
        user.* = .{ .id = i, .name = "User" };
        try map.put(i, user);
    }

    first.name = "Alice Smith";

    std.debug.print("User {s}\n", .{map.get(1).?.name});
}
```

Теперь наш код выполняется без ошибок. Проблема с расширением массива для хеш-таблицы больше у нас не воспроизводится, так как мы не храним `User` в таблице, а храним только указатели. Когда наш массив заполнится на 80%, наша таблица запустит процесс расширения массива и перенесет наши указатели в новый массив, но при этом указатели поо прежнему будут ссылаться на структуры `User` созданные нами в куче и проблем с доступом к ним не будет.

Когда мы используем для хранения в хеш-таблице указатели на структуры, вместо самих структур у нас разное время жизни у таблицы и данных, которые мы там храним и очень важно об этом помнить у правильно очищать память в вашей программе. Мы поговорим еще более подробно про владение данными в следующей главе.

### Итерирование по хеш-таблице
Довольно часто у вас возникает необходимость пройтись по данным, которые хранятся в хеш-таблице и тут таблица предоставляет нам сразу несколько методов для итерирования по данным:

* iterator - возвращает итератор по парам ключ/значение для хеш-таблицы
* keyIterator - возвращает итератор по ключам хеш-таблицы
* valueIterator - возвращает итератор по значениям хеш-таблицы

Давайте рассмотрим пример:

```zig
const std = @import("std");

const User = struct {
    id: usize,
    name: []const u8,
};

pub fn main() !void {
    var gpa = std.heap.DebugAllocator(.{}).init;
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var map = std.AutoHashMap(usize, User).init(allocator);
    defer map.deinit();

    try map.put(1, .{ .id = 1, .name = "Alice" });
    try map.put(2, .{ .id = 2, .name = "Bob" });

    var it = map.iterator();
    while (it.next()) |entry| {
        std.debug.print("User {s}\n", .{entry.value_ptr.name});
    }
}
```

Наш код выведет:
```
User Alice
User Bob
```

Важно отметить, что итератор возвращает нам структуру `Entry`, в которой лежат указатели на ключ и значение. Это позволяет нам не только итерироваться по данным в таблице для чтения, но и менять значения во время итерации, что может быть удобно для обновления данных в хеш-таблице. Однако надо понимать, что изменение значения ключа может возможно только если новое значение даст тот же самый хеш, что и старое значение.

### ArrayHashMap


### StringArrayHashMap
