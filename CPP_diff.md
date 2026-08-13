# C++ 版本差异记录

记录 C++11 到 C++23 各版本之间的关键行为变化和破坏性更改。

---

## 目录

- [C++98/03 → C++11](#c9803--c11)
- [C++11 → C++14](#c11--c14)
- [C++14 → C++17](#c14--c17)
- [C++17 → C++20](#c17--c20)
- [C++20 → C++23](#c20--c23)

---

## C++98/03 → C++11

### 语言特性变化

C++11 是 C++ 历史上**最大的一次更新**（俗称"Modern C++"的开端），引入了大量核心语言特性：

| 特性 | C++98/03 | C++11 |
|:---|:---|:---|
| `auto` 自动类型推导 | ❌ 不支持 | ✅ `auto x = 42;` |
| 范围 `for` 循环 | ❌ 不支持 | ✅ `for (auto& x : v)` |
| Lambda 表达式 | ❌ 不支持 | ✅ `[&](int x) { return x; }` |
| `nullptr` | ❌ 用 `NULL`/`0` | ✅ 类型安全的空指针 |
| 右值引用 / 移动语义 | ❌ 不支持 | ✅ `T&&`、`std::move`、移动构造/赋值 |
| 变长模板（variadic templates） | ❌ 不支持 | ✅ `template<typename... Args>` |
| `constexpr` | ❌ 不支持 | ✅ 编译期求值函数 |
| `decltype` | ❌ 不支持 | ✅ 获取表达式类型 |
| 统一初始化 / 初始化列表 | ❌ 有限 | ✅ `T obj{...}`、`std::initializer_list` |
| `enum class`（强类型枚举） | ❌ 不支持 | ✅ 强类型、需显式转换 |
| `static_assert` | ❌ 不支持 | ✅ 编译期断言 |
| `override` / `final` | ❌ 不支持 | ✅ 显式重写/终结标记 |
| `= default` / `= delete` | ❌ 不支持 | ✅ 显式默认/删除特殊成员函数 |
| 委托构造函数 | ❌ 不支持 | ✅ 构造函数相互调用 |
| 继承构造函数 | ❌ 不支持 | ✅ `using Base::Base;` |
| `noexcept` | ❌ 无（动态异常说明 `throw()`） | ✅ 静态异常说明 |
| `thread_local` | ❌ 不支持 | ✅ 线程局部存储 |
| 原始字符串字面量 | ❌ 不支持 | ✅ `R"(...)"` |
| `alignof` / `alignas` | ❌ 不支持 | ✅ 对齐控制 |

### 标准库变化

| 特性 | C++98/03 | C++11 |
|:---|:---|:---|
| 智能指针 | `std::auto_ptr`（有缺陷） | ✅ `unique_ptr` / `shared_ptr` / `weak_ptr` |
| 线程库 | ❌ 无标准线程 | ✅ `std::thread` / `std::mutex` / `std::atomic` |
| 哈希容器 | ❌ 无 | ✅ `unordered_map` / `unordered_set` |
| `std::tuple` | ❌ 无 | ✅ 异构元组 |
| `std::array` | ❌ 无 | ✅ 定长数组包装 |
| `std::chrono` | ❌ 无 | ✅ 时间库（duration/time_point/clock） |
| `std::regex` | ❌ 无 | ✅ 正则表达式 |
| `std::function` / `std::bind` | ❌ 无 | ✅ 通用可调用对象包装/绑定 |
| `std::begin` / `std::end` | ❌ 无 | ✅ 泛型迭代器自由函数 |
| `std::move` / `std::forward` | ❌ 无 | ✅ 移动/完美转发工具 |

### 破坏性变化

1. **字符串字面量不能隐式转 `char*`**——C++11 起必须用 `const char*`，旧代码 `char* s = "x";` 编译错误。
2. **`auto_ptr` 被标记为 deprecated**——建议用 `unique_ptr`，C++17 彻底移除。
3. **异常说明从动态改为静态**——`throw()` 语法被 `noexcept` 取代（`throw()` 仍可用但 deprecated）。
4. **`export` 模板被移除**——`export template` 从未被主流实现支持，C++11 删除。
5. **新增关键字**——`auto`、`nullptr`、`constexpr`、`decltype` 等成为保留字，用作标识符的旧代码需改名。

---

## C++11 → C++14

### 语言特性变化

| 特性 | C++11 | C++14 |
|:---|:---|:---|
| 返回类型推导 | ❌ 不支持（lambda 可推导） | ✅ 普通函数 `auto f()` 支持 |
| `constexpr` 函数限制 | 函数体只能包含单一 `return` | 放宽限制，支持 `if`、循环、多个 `return` |
| Lambda 捕获 | 只能按值/引用捕获已有变量 | 支持捕获初始化器 `[x = expr]`、移动捕获 |
| 变量模板 | ❌ 不支持 | ✅ `template<T> T pi = T(3.14);` |
| `[[deprecated]]` 属性 | ❌ 不存在 | ✅ 标记弃用实体 |
| 泛型 lambda | ❌ 不支持 | ✅ `[](auto x) { return x; }` |
| `decltype(auto)` | ❌ 不存在 | ✅ 保留引用和 cv 限定符的推导 |

### 标准库变化

| 特性 | C++11 | C++14 |
|:---|:---|:---|
| `std::make_unique` | ❌ 不存在（需自己实现） | ✅ 推荐使用 |
| `std::cbegin` / `std::cend` | ❌ 不存在 | ✅ const 迭代器自由函数 |
| `std::rbegin` / `std::rend` | ❌ 不存在 | ✅ 反向迭代器自由函数 |
| `std::integer_sequence` | ❌ 不存在 | ✅ 编译时整数序列 |
| 标准库类型字面量 | ❌ 不存在 | ✅ `24h`、`500ms`、`"s"s` 等 |

### 破坏性变化

- **无重大破坏性变化**。C++14 被称为"最小的 C++ 版本"，主要是对 C++11 的修补和扩充。

---

## C++14 → C++17

### 语言特性变化

| 特性 | C++14 | C++17 |
|:---|:---|:---|
| 类模板参数推导 (CTAD) | ❌ 必须显式写模板参数 | ✅ `std::pair p(1, 2.5)` 自动推导 |
| 折叠表达式 | ❌ 需递归展开参数包 | ✅ `(... + args)` 一行搞定 |
| `auto` 非类型模板参数 | ❌ 必须写类型 | ✅ `template<auto V>` |
| 结构化绑定 | ❌ 不支持 | ✅ `auto [x, y] = pair;` |
| `constexpr if` | ❌ 不支持 | ✅ 编译期分支 |
| `if`/`switch` 带初始化器 | ❌ 不支持 | ✅ `if (init; cond)` |
| 内联变量 | ❌ 需在源文件定义静态成员 | ✅ `inline static int count;` |
| 嵌套命名空间 | ❌ `namespace A::B::C {}` 非法 | ✅ 支持 |
| `constexpr` lambda | ❌ 不支持 | ✅ Lambda 可标记 `constexpr` |
| Lambda 按值捕获 `this` | ❌ 只能按引用捕获 `this` | ✅ `[*this] { ... }` |
| `[[fallthrough]]` 等属性 | ❌ 不存在 | ✅ `[[fallthrough]]`、`[[nodiscard]]`、`[[maybe_unused]]` |
| `__has_include` | ❌ 不存在 | ✅ 编译期检查头文件是否存在 |
| 枚举直接列表初始化 | ❌ `enum E e{0}` 非法 | ✅ 支持 |

### `auto` 从花括号初始化列表推导

| 写法 | C++11/14 | C++17 |
|:---|:---|:---|
| `auto x{3};` | `std::initializer_list<int>` | `int` |
| `auto x{1, 2, 3};` | `std::initializer_list<int>` | ❌ 编译错误 |
| `auto x = {1, 2, 3};` | `std::initializer_list<int>` | `std::initializer_list<int>`（不变） |

> **说明**：C++11 中 `auto x{3}` 推导为 `initializer_list` 被广泛认为是设计缺陷，C++17 修复了此行为。但多元素的直接列表初始化被禁止，因为无法合理推导为单一类型。

### 标准库变化

| 特性 | C++14 | C++17 |
|:---|:---|:---|
| `std::optional` | ❌ 不存在 | ✅ 可选值容器 |
| `std::variant` | ❌ 不存在 | ✅ 类型安全的 union |
| `std::any` | ❌ 不存在 | ✅ 任意类型容器 |
| `std::string_view` | ❌ 不存在 | ✅ 字符串非拥有引用 |
| `std::filesystem` | ❌ 不存在 | ✅ 文件系统操作 |
| `std::invoke` / `std::apply` | ❌ 不存在 | ✅ 通用可调用对象调用 |
| `std::byte` | ❌ 不存在 | ✅ 字节数据类型 |
| `std::not_fn` | ❌ 不存在 | ✅ 函数结果取反 |
| `std::clamp` | ❌ 不存在 | ✅ 值限制 |
| `std::reduce` | ❌ 不存在 | ✅ 并行折叠 |
| `std::gcd` / `std::lcm` | ❌ 不存在 | ✅ 最大公约数/最小公倍数 |
| `std::to_chars` / `std::from_chars` | ❌ 不存在 | ✅ 高性能数字字符串转换 |
| map/set 拼接 | ❌ 不支持 | ✅ `extract` / `insert(node)` |
| 并行算法 | ❌ 不支持 | ✅ `std::execution::par` |
| `std::sample` | ❌ 不存在 | ✅ 随机采样 |
| `std::size` / `std::empty` | ❌ 不存在 | ✅ 泛型大小/判空 |

### 破坏性变化

1. **`auto x{1, 2, 3}` 从合法变为编译错误**——C++11/14 中推导为 `initializer_list`，C++17 中直接报错。
2. **`std::auto_ptr` 被移除**——C++17 中彻底删除，需用 `std::unique_ptr` 替代。
3. **`throw()` 动态异常说明被移除**——C++17 中 `throw()` 是 `deprecated`，C++20 移除。
4. **`std::unexpected` / `std::set_unexpected` 移除**——旧的异常处理机制。
5. **三字符组（trigraphs）被移除**——`??=` 等序列不再被识别。

---

## C++17 → C++20

### 语言特性变化

| 特性 | C++17 | C++20 |
|:---|:---|:---|
| Concept（概念） | ❌ 不存在 | ✅ `template<typename T> concept ...` |
| 范围 for 循环支持初始化器 | ❌ 不支持 | ✅ `for (int i = 0; auto& x : v)` |
| 协程 (Coroutines) | ❌ 不存在 | ✅ `co_await`、`co_yield`、`co_return` |
| 模块 (Modules) | ❌ 不存在 | ✅ `import`、`export` |
| 三路比较运算符 `<=>` | ❌ 不存在 | ✅ 自动生成所有比较运算符 |
| 设计化初始化（Designated Initializers） | ❌ 不支持 | ✅ `.field = value` |
| `constexpr` 虚函数 | ❌ 不支持 | ✅ 支持 |
| `constexpr` `try`/`catch` | ❌ 不支持 | ✅ 部分支持 |
| `constexpr` `new`/`delete` | ❌ 不支持 | ✅ 支持（在 `constexpr` 上下文中） |
| `constexpr` `dynamic_cast` | ❌ 不支持 | ✅ 部分支持 |
| `constexpr` `std::vector` | ❌ 不支持 | ✅ 编译期可创建 vector |
| `constexpr` `std::string` | ❌ 不支持 | ✅ 编译期可创建 string |
| Lambda 模板语法 | ❌ 不支持 | ✅ `[]<typename T>(T x){}` |
| constexpr lambda 增强 | 不能默认构造/赋值 | ✅ 支持默认构造、赋值、`consteval` lambda |
| `consteval` | ❌ 不存在 | ✅ 立即函数（必须在编译期求值） |
| `constinit` | ❌ 不存在 | ✅ 保证静态初始化顺序 |
| `[[likely]]` / `[[unlikely]]` | ❌ 不存在 | ✅ 分支预测提示 |
| `[[no_unique_address]]` | ❌ 不存在 | ✅ 空基类优化标记 |
| 位域成员变量的默认成员初始化器 | ❌ 不支持 | ✅ `int b : 1 = 0;` |
| `using enum` | ❌ 不支持 | ✅ 将枚举成员引入作用域 |
| 非类型模板参数支持浮点 | ❌ 不支持 | ✅ `template<double V>` 合法 |
| CTAD 扩展到聚合类型 | ❌ 不支持 | ✅ 聚合类型也可自动推导 |

### 标准库变化

| 特性 | C++17 | C++20 |
|:---|:---|:---|
| Ranges 库 | ❌ 不存在 | ✅ `std::ranges`、`std::views` |
| `std::span` | ❌ 不存在 | ✅ 连续内存的非拥有视图 |
| `std::format` | ❌ 不存在 | ✅ 类型安全的格式化（类似 Python f-string） |
| `std::source_location` | ❌ 不存在 | ✅ 源码位置信息（替代 `__FILE__`、`__LINE__`） |
| `std::atomic<T>` 的 `wait`/`notify` | ❌ 不存在 | ✅ 原子等待/通知 |
| `std::jthread` | ❌ 不存在 | ✅ 自动 join 的线程 |
| `std::stop_token` | ❌ 不存在 | ✅ 协作式线程停止 |
| `std::bind_front` | ❌ 不存在 | ✅ 绑定前 N 个参数 |
| `std::bit_cast` | ❌ 不存在 | ✅ 类型双关的安全方式 |
| `std::midpoint` / `std::lerp` | ❌ 不存在 | ✅ 中点/线性插值 |
| `std::ssize` | ❌ 不存在 | ✅ 有符号大小 |
| `std::to_array` | ❌ 不存在 | ✅ 从 C 数组创建 `std::array` |
| 日历和时区支持 | ❌ 不存在 | ✅ `std::chrono` 增强（年/月/日、时区） |
| `std::basic_syncbuf` / `std::basic_osyncstream` | ❌ 不存在 | ✅ 同步输出流 |

### 破坏性变化

1. **聚合类型初始化变化**——有用户声明构造函数的类不再是聚合类型（C++17 及之前是），影响 `{}` 初始化行为。
2. **`std::memory_order` 枚举变为 `enum class`**——`memory_order_acquire` → `std::memory_order::acquire`（旧风格仍可用但 deprecated）。
3. **`std::is_pod` 被标记为 deprecated**——C++20 起不推荐使用 POD 概念。
4. **隐式生成比较运算符的变化**——如果声明了 `<=>` 或 `==`，编译器行为会变化。
5. **`std::result_of` 被移除**——用 `std::invoke_result` 替代。
6. **`std::allocator` 的 `construct`/`destroy` 被移除**——用 `std::allocator_traits`。

---

## C++20 → C++23

### 语言特性变化

| 特性 | C++20 | C++23 |
|:---|:---|:---|
| 显式对象成员函数（Deducing `this`） | ❌ 不存在 | ✅ `void f(this auto& self)` |
| `if`/`switch` 的 `auto` 初始化器扩展 | ❌ 有限 | ✅ 更灵活 |
| `static operator()` / `static operator[]` | ❌ 不支持 | ✅ 静态调用运算符 |
| `[[assume(expr)]]` | ❌ 不存在 | ✅ 编译器优化提示 |
| 多维下标运算符 `operator[]` | ❌ 只能单参数 | ✅ 支持多参数 `a[i, j]` |
| `constexpr` 函数支持 `std::unique_ptr` | ❌ 不支持 | ✅ 支持 |
| 字面量后缀 `_` 被保留 | ❌ 未定义 | ✅ 保留给未来使用 |
| `#elifdef` / `#elifndef` | ❌ 不存在 | ✅ 预处理指令 |
| `if consteval` | ❌ 不存在 | ✅ 编译期分支（可安全调 `consteval` 函数） |
| Lambda 捕获参数包 | ❌ 不支持 | ✅ `[...args = std::forward<Args>(args)]` |
| 自动 `operator==` 从 `<=>` 生成 | 仅部分 | ✅ 更完整 |
| Lambda 捕获 `[...]` 中支持 `this` 和 `*this` 一起 | ❌ 不能同时 | ✅ 可以同时 |
| 结构化绑定自定义点 | ❌ 不存在 | ✅ 可通过 `tuple_size`/`get` 自定义 |
| `[=]` 隐式捕获 `this` | 弃用（deprecated） | ❌ 直接报错 |

### 标准库变化

| 特性 | C++20 | C++23 |
|:---|:---|:---|
| `std::expected<T, E>` | ❌ 不存在 | ✅ 带错误码的返回值 |
| `std::flat_map` / `std::flat_set` | ❌ 不存在 | ✅ 基于连续存储的排序容器 |
| `std::mdspan` | ❌ 不存在 | ✅ 多维数组视图 |
| `std::generator` | ❌ 不存在 | ✅ 协程生成器 |
| `std::print` / `std::println` | ❌ 不存在 | ✅ 直接打印到 stdout |
| `std::out_ptr` / `std::inout_ptr` | ❌ 不存在 | ✅ 与 C 指针互操作 |
| `std::stacktrace` | ❌ 不存在 | ✅ 栈回溯 |
| `std::byteswap` | ❌ 不存在 | ✅ 字节序交换 |
| `std::to_underlying` | ❌ 不存在 | ✅ 枚举转底层整数类型 |
| `std::shift_left` / `std::shift_right` | ❌ 不存在 | ✅ 范围移位 |
| Ranges 的 `std::ranges::to` | ❌ 不存在 | ✅ 范围转容器 |
| `std::string::contains` | ❌ 不存在 | ✅ 子串判断（终于有了！） |
| `std::optional` 支持 Monadic 操作 | ❌ 不存在 | ✅ `and_then`、`or_else`、`transform` |
| `std::views::zip` / `std::views::enumerate` | ❌ 不存在 | ✅ 范围视图增强 |
| `std::unreachable` | ❌ 不存在 | ✅ 标记不可达分支 |
| `std::move_only_function` | ❌ 不存在 | ✅ 可移动的泛型函数包装 |
| 字符串 `starts_with`/`ends_with` 扩展 | 仅字符串 | ✅ `string_view` 也支持 |

### 破坏性变化

1. **`std::unary_function` / `std::binary_function` 被移除**——C++11 已 deprecated，C++23 彻底删除。
2. **`std::iterator` 被移除**——C++17 已 deprecated，不再需要继承它。
3. **`std::is_literal_type` 被移除**——C++17 已 deprecated。
4. **`std::allocator` 的 `is_always_equal` 默认变化**——部分实现调整。
5. **窄字符编码假设变化**——`char` 不再默认假设为 ASCII/UTF-8 之外的编码。

---

## 快速参考：常用写法在各版本的变化

| 写法 | C++11 | C++14 | C++17 | C++20 | C++23 |
|:---|:---|:---|:---|:---|:---|
| `auto x{3};` | `initializer_list` | `initializer_list` | ✅ `int` | ✅ `int` | ✅ `int` |
| `auto x{1, 2};` | `initializer_list` | `initializer_list` | ❌ 错误 | ❌ 错误 | ❌ 错误 |
| `auto f() { return x; }` | ❌ 错误 | ✅ 推导 | ✅ 推导 | ✅ 推导 | ✅ 推导 |
| `constexpr` 函数含循环 | ❌ 错误 | ✅ 支持 | ✅ 支持 | ✅ 支持 | ✅ 支持 |
| `std::make_unique<T>()` | ❌ 需自己实现 | ✅ 标准提供 | ✅ | ✅ | ✅ |
| `std::optional` | ❌ 不存在 | ❌ 不存在 | ✅ | ✅ | ✅ |
| `std::variant` | ❌ 不存在 | ❌ 不存在 | ✅ | ✅ | ✅ |
| `std::string_view` | ❌ 不存在 | ❌ 不存在 | ✅ | ✅ | ✅ |
| 三路比较 `<=>` | ❌ 不存在 | ❌ 不存在 | ❌ 不存在 | ✅ | ✅ |
| Concept | ❌ 不存在 | ❌ 不存在 | ❌ 不存在 | ✅ | ✅ |
| 协程 | ❌ 不存在 | ❌ 不存在 | ❌ 不存在 | ✅ | ✅ |
| `std::expected` | ❌ 不存在 | ❌ 不存在 | ❌ 不存在 | ❌ 不存在 | ✅ |
| `std::print` | ❌ 不存在 | ❌ 不存在 | ❌ 不存在 | ❌ 不存在 | ✅ |
| Lambda `[x = std::move(x)]` | ❌ 错误 | ✅ 支持 | ✅ | ✅ | ✅ |
| `if constexpr { }` | ❌ 不存在 | ❌ 不存在 | ✅ | ✅ | ✅ |
| `auto [x, y] = pair` | ❌ 不存在 | ❌ 不存在 | ✅ | ✅ | ✅ |
| `std::format("{}", x)` | ❌ 不存在 | ❌ 不存在 | ❌ 不存在 | ✅ | ✅ |
| 并行算法 `std::execution::par` | ❌ 不存在 | ❌ 不存在 | ✅ | ✅ | ✅ |
| `std::filesystem` | ❌ 不存在 | ❌ 不存在 | ✅ | ✅ | ✅ |
| `std::string::contains()` | ❌ 不存在 | ❌ 不存在 | ❌ 不存在 | ❌ 不存在 | ✅ |
| `if consteval` | ❌ 不存在 | ❌ 不存在 | ❌ 不存在 | ❌ 不存在 | ✅ |
| Lambda 捕获参数包 `[...args]` | ❌ 不存在 | ❌ 不存在 | ❌ 不存在 | ✅ | ✅ |
| `std::stacktrace` | ❌ 不存在 | ❌ 不存在 | ❌ 不存在 | ❌ 不存在 | ✅ |
