# C++11

## 概述

以下描述和示例来自各种资源（见[致谢](#致谢)部分），并以我自己的话进行了总结。

C++11 包含以下新语言特性：
- [C++11](#c11)
  - [概述](#概述)
  - [C++11 语言特性](#c11-语言特性)
    - [Move Semantics(移动语义)](#move-semantics移动语义)
    - [Rvalue Reference(右值引用)](#rvalue-reference右值引用)
    - [Forwarding Reference(转发引用)](#forwarding-reference转发引用)
      - [引用折叠规则](#引用折叠规则)
      - [推导过程详解](#推导过程详解)
      - [典型使用场景](#典型使用场景)
      - [不使用 std::forward 时改变值类别的情况](#不使用-stdforward-时改变值类别的情况)
    - [Variadic Template(变参模板)](#variadic-template变参模板)
    - [Initializer List(初始化列表)](#initializer-list初始化列表)
    - [Static Assert(静态断言)](#static-assert静态断言)
    - [auto](#auto)
    - [Lambda Expression(Lambda 表达式)](#lambda-expressionlambda-表达式)
      - [按值捕获、按引用捕获与 mutable 的关系](#按值捕获按引用捕获与-mutable-的关系)
    - [decltype](#decltype)
    - [Type Alias(类型别名)](#type-alias类型别名)
    - [nullptr](#nullptr)
    - [Strongly Typed Enum(强类型枚举)](#strongly-typed-enum强类型枚举)
    - [Attribute(属性)](#attribute属性)
    - [constexpr](#constexpr)
    - [Delegating Constructor(委托构造函数)](#delegating-constructor委托构造函数)
    - [User-Defined Literal(用户自定义字面量)](#user-defined-literal用户自定义字面量)
    - [Explicit Virtual Override(显式虚函数覆盖)](#explicit-virtual-override显式虚函数覆盖)
    - [Final Specifier(final 说明符)](#final-specifierfinal-说明符)
    - [Default Function(默认函数)](#default-function默认函数)
    - [Deleted Function(删除函数)](#deleted-function删除函数)
    - [Range-Based For Loop(范围 for 循环)](#range-based-for-loop范围-for-循环)
    - [Move Semantics(移动语义)的特殊成员函数](#move-semantics移动语义的特殊成员函数)
    - [Converting Constructor(转换构造函数)](#converting-constructor转换构造函数)
    - [Explicit Conversion Function(显式转换函数)](#explicit-conversion-function显式转换函数)
    - [Inline Namespace(内联命名空间)](#inline-namespace内联命名空间)
    - [Non-Static Data Member Initializer(非静态数据成员初始化器)](#non-static-data-member-initializer非静态数据成员初始化器)
    - [Right Angle Bracket(右尖括号)](#right-angle-bracket右尖括号)
    - [Ref-Qualified Member Function(引用限定成员函数)](#ref-qualified-member-function引用限定成员函数)
    - [Trailing Return Type(返回类型后置)](#trailing-return-type返回类型后置)
    - [Noexcept Specifier(noexcept 说明符)](#noexcept-specifiernoexcept-说明符)
    - [char32\_t and char16\_t(char32\_t 和 char16\_t)](#char32_t-and-char16_tchar32_t-和-char16_t)
    - [Raw String Literal(原始字符串字面量)](#raw-string-literal原始字符串字面量)
  - [C++11 标准库特性](#c11-标准库特性)
    - [std::move](#stdmove)
    - [std::forward](#stdforward)
    - [std::thread](#stdthread)
    - [std::to\_string](#stdto_string)
    - [Type Trait(类型特征)](#type-trait类型特征)
    - [Smart Pointer(智能指针)](#smart-pointer智能指针)
    - [std::chrono](#stdchrono)
    - [Tuple(元组)](#tuple元组)
    - [std::tie](#stdtie)
    - [std::array](#stdarray)
    - [Unordered Container(无序容器)](#unordered-container无序容器)
    - [std::make\_shared](#stdmake_shared)
    - [std::ref](#stdref)
    - [Memory Model(内存模型)](#memory-model内存模型)
    - [std::async](#stdasync)
    - [std::begin/end](#stdbeginend)
  - [致谢](#致谢)
  - [作者](#作者)
  - [内容贡献者](#内容贡献者)
  - [许可证](#许可证)

C++11 包含以下新标准库特性：
- [std::move](#stdmove)
- [std::forward](#stdforward)
- [std::thread](#stdthread)
- [std::to_string](#stdto_string)
- [Type Trait(类型特征)](#类型特征)
- [Smart Pointer(智能指针)](#智能指针)
- [std::chrono](#stdchrono)
- [Tuple(元组)](#元组)
- [std::tie](#stdtie)
- [std::array](#stdarray)
- [Unordered Container(无序容器)](#无序容器)
- [std::make_shared](#stdmake_shared)
- [std::ref](#stdref)
- [Memory Model(内存模型)](#内存模型)
- [std::async](#stdasync)
- [std::begin/end](#stdbeginend)

## C++11 语言特性

### Move Semantics(移动语义)

移动一个对象意味着将其管理的某些资源的所有权转移给另一个对象。

移动语义的第一个好处是性能优化。当一个对象即将到达其生命周期的末尾时（无论是临时对象还是通过显式调用 `std::move`），移动通常是一种更便宜的资源转移方式。例如，移动一个 `std::vector` 只是将一些指针和内部状态复制到新向量中——而复制则需要复制向量中的每个元素，如果旧向量很快就会被销毁，这既昂贵又不必要。

移动还使得非可拷贝类型（如 `std::unique_ptr`，参见[智能指针](#智能指针)）能够在语言层面保证同一时间只有一个实例管理资源，同时能够在不同作用域之间转移实例。

参见以下章节：[右值引用](#右值引用)、[移动语义的特殊成员函数](#移动语义的特殊成员函数)、[`std::move`](#stdmove)、[`std::forward`](#stdforward)、[`转发引用`](#转发引用)。

### Rvalue Reference(右值引用)

C++11 引入了一种新的引用类型，称为***Rvalue Reference(右值引用)***。对于非模板类型参数 `T`（如 `int` 或用户定义类型），右值引用的语法是 `T&&`。右值引用只能绑定到右值。

左值和右值的类型推导：
```c++
int x = 0; // `x` 是类型为 `int` 的左值
int& xl = x; // `xl` 是类型为 `int&` 的左值
int&& xr = x; // 编译错误 -- `x` 是左值
int&& xr2 = 0; // `xr2` 是类型为 `int&&` 的左值 -- 绑定到右值临时对象 `0`

void f(int& x) {}
void f(int&& x) {}

f(x);  // 调用 f(int&)
f(xl); // 调用 f(int&)
f(3);  // 调用 f(int&&)
f(std::move(x)); // 调用 f(int&&)

f(xr2);           // 调用 f(int&)
f(std::move(xr2)); // 调用 f(int&& x)
```

参见：[`std::move`](#stdmove)、[`std::forward`](#stdforward)、[`转发引用`](#转发引用)。

### Forwarding Reference(转发引用)

也称为（非官方）**万能引用(Universal Reference)**。当 `T` 是模板类型参数或使用 `auto&&` 时，使用语法 `T&&` 创建转发引用。这实现了***Perfect Forwarding(完美转发)***：在传递参数时保持其值类别（例如，左值保持为左值，临时对象作为右值转发）。

转发引用允许一个引用根据初始化它的表达式是左值还是右值，分别绑定为左值引用或右值引用。这是通过两条规则的配合实现的：

1. **模板参数推导的特殊规则**：当 `T&&` 中的 `T` 是模板参数时，如果传入左值，`T` 被推导为 `T&`（而非 `T`）；如果传入右值，`T` 被推导为 `T`（保持不变）。
2. **引用折叠(Reference Collapsing)**：编译器不允许"引用的引用"直接存在，会自动折叠为单层引用。

#### 引用折叠规则

引用折叠的规则非常简单，核心只有一个原则：

> **只要有左值引用 `&` 参与，结果就是左值引用。只有两个都是右值引用 `&&` 时，结果才是右值引用。**

换句话说：**左值引用具有"传染性"**，只要折叠链中有一个 `&`，最终结果就是 `&`。

| 折叠表达式 | 结果 | 记忆口诀 |
|:---:|:---:|:---:|
| `T& &` | `T&` | 有 `&` → `&` |
| `T& &&` | `T&` | 有 `&` → `&` |
| `T&& &` | `T&` | 有 `&` → `&` |
| `T&& &&` | `T&&` | 无 `&` → `&&`（唯一的例外） |

更方便的记忆方式：**只有 `&& + && = &&`，其他任何组合都等于 `&`**。类似于逻辑与运算——把 `&` 看作 `false`，`&&` 看作 `true` 也能帮助记忆。

#### 推导过程详解

`auto` 类型推导与左值和右值：
```c++
int x = 0; // `x` 是类型为 `int` 的左值
auto&& al = x; // `al` 是类型为 `int&` 的左值 -- 绑定到左值 `x`
auto&& ar = 0; // `ar` 是类型为 `int&&` 的左值 -- 绑定到右值临时对象 `0`
```

模板类型参数推导与左值和右值——完整推导过程：
```c++
template <typename T>
void f(T&& t) {
  // ...
}

int x = 0;
f(0); // 0 是右值 → T 推导为 int     → T&& 即 int&&     → f(int&&)
f(x); // x 是左值 → T 推导为 int&    → T&& 即 int& &&   → 折叠为 f(int&)

int& y = x;
f(y); // y 是左值 → T 推导为 int&    → T&& 即 int& &&   → 折叠为 f(int&)

int&& z = 0; // 注意：`z` 是类型为 `int&&` 的左值（有名字的右值引用是左值）。
f(z); // z 是左值 → T 推导为 int&    → T&& 即 int& &&   → 折叠为 f(int&)
f(std::move(z)); // std::move(z) 是右值 → T 推导为 int → T&& 即 int&& → f(int&&)
```

**关键理解**：有名字的右值引用变量（如 `z`）本身是左值，因为可以取地址。只有 `std::move` 将其转换为右值后，才会触发右值版本的推导。

#### 典型使用场景

**1. 工厂函数 / 完美转发构造函数参数**

最经典的场景：将参数原封不动地转发给另一个函数，保持其值类别（左值/右值）不变。

```c++
template <typename T, typename Arg>
std::shared_ptr<T> factory(Arg&& arg) {
  // 使用 std::forward 保持 arg 的值类别进行转发
  // 如果 arg 是左值，则以左值引用方式传递给 T 的构造函数
  // 如果 arg 是右值，则以右值引用方式（移动）传递给 T 的构造函数
  return std::shared_ptr<T>(new T(std::forward<Arg>(arg)));
}
```

这正是 `std::make_shared`、`std::make_unique` 等标准库函数的实现原理。

**2. 容器原地构造（emplace 系列）**

```c++
std::vector<std::string> vec;
std::string s = "hello";

vec.push_back(s);            // 拷贝 s
vec.push_back(std::move(s)); // 移动 s
vec.emplace_back("world");   // 直接在容器内部原地构造，完美转发参数
```

`emplace_back` 内部使用转发引用接收任意参数，完美转发给元素的构造函数，避免不必要的拷贝和移动。

**3. 通用包装器 / 代理函数**

```c++
// 一个通用的日志包装器，不改变被调用函数的参数语义
template <typename F, typename... Args>
auto log_and_call(F&& f, Args&&... args) -> decltype(auto) {
  std::cout << "Calling function..." << std::endl;
  return std::forward<F>(f)(std::forward<Args>(args)...);
}
```

**4. 与 `std::forward` 配合实现完美转发**

`std::forward` 的作用是：根据模板参数 `T` 的类型，将参数还原为其原始值类别进行转发。

```c++
template <typename T>
void wrapper(T&& arg) {
  // arg 是左值（有名字）
  // std::forward<T> 根据 T 的推导结果决定转发的值类别：
  //   - T 被推导为 int&  → std::forward<T>(arg) 返回左值引用
  //   - T 被推导为 int   → std::forward<T>(arg) 返回右值引用（等同于 std::move）
  foo(std::forward<T>(arg));
}
```

**核心记忆**：
- 当你想**无条件转为右值**时，使用 `std::move`
- 当你想**保持原值类别转发**时，使用 `std::forward`（它利用了转发引用和引用折叠）

#### 不使用 std::forward 时改变值类别的情况

在 C++ 中，即使不显式使用 `std::forward`，值类别也可能在以下情况发生隐式改变。理解这些规则有助于写出正确的代码，并避免意外的拷贝或悬垂引用。

**1. 所有具名变量都是左值（最常见的陷阱）**

这是最基础的规则：任何有名字的变量都是左值，即使它的类型是右值引用。在转发引用函数中，如果不使用 `std::forward`，参数将始终以左值传递。

```c++
void sink(std::string& s)  { std::cout << "lvalue\n"; }
void sink(std::string&& s) { std::cout << "rvalue\n"; }

template <typename T>
void wrapper(T&& arg) {
  sink(arg);                    // arg 有名字 → 永远是左值 → 调用 sink(string&)
  sink(std::forward<T>(arg));   // forward 恢复原始值类别 → 正确分发
}

wrapper(std::string("hello"));  // 不使用 forward: 输出 "lvalue"
                                // 使用 forward:   输出 "rvalue"
```

**2. return 语句中的隐式移动**

当 `return` 一个局部变量或按值传递的函数参数时，编译器会自动将其视为右值（xvalue），优先匹配移动构造函数。这是 C++11 起标准规定的行为，无需手动 `std::move`——反而手动写 `std::move` 会抑制 NRVO 优化。

这里容易产生一个疑问：局部变量离开作用域不就销毁了吗，为什么还能移动？关键在于 **return 的执行顺序**：

1. 先对 `return` 表达式求值，构造返回值（此时局部变量还活着）
2. 移动构造将局部变量的资源（如堆内存指针）转移给返回值
3. 局部变量离开作用域，析构——此时它已是空壳，析构几乎无开销

所以移动发生在销毁**之前**，本质是"趁它还没死，把值偷走"。

```c++
std::string create() {
  std::string s = "hello";
  return s;             // 隐式移动（或 NRVO），不要写 return std::move(s);
}

std::string passthrough(std::string s) {
  return s;             // 按值参数也是隐式移动，同样不需要 std::move
}
```

**3. 三目运算符的混合值类别**

当 `?:` 的两个分支值类别不同时，结果会被统一为右值：

```c++
int x = 1;
int& lref = x;
int&& rref = 0;

auto&  a = true ? lref : lref;  // 两个都是左值 → 结果是左值引用，OK
// auto& b = true ? lref : 0;   // 错误！左值+右值混合 → 结果是右值，不能绑定到左值引用
auto&& c = true ? lref : 0;     // OK，转发引用可以绑定右值
```

**4. 右值对象的成员访问**

对右值对象访问非静态数据成员（或非引用类型的成员函数返回值），结果是一个右值（xvalue）：

```c++
struct A { int data; int& ref; };

A{}.data;   // xvalue（右值），可以移动
A{}.ref;    // lvalue（引用成员保持引用类型）
```

**5. 临时量实体化**

当一个纯右值（prvalue）需要绑定到引用时，会发生临时量实体化（C++17 起正式命名），产生一个 xvalue：

```c++
struct S { int m; };

int&& r = S{}.m;  // S{}.m 是 xvalue，实体化为临时对象，生命周期延长到 r 的作用域结束
```

**6. 显式转型为右值引用**

除了 `std::move`，直接使用 `static_cast` 也可以改变值类别（`std::move` 内部就是 `static_cast`）：

```c++
int x = 0;
int&& r = static_cast<int&&>(x);  // 将左值 x 显式转为右值引用，等价于 std::move(x)
```

**总结对比**

| 场景 | 值类别变化 | 说明 |
|:---|:---:|:---|
| 具名变量（含右值引用） | 右值 → 左值 | 有名字就是左值 |
| `return` 局部变量 | 左值 → 右值 | 隐式移动，别加 `std::move` |
| `return` 按值参数 | 左值 → 右值 | 同上 |
| `?:` 混合分支 | 左值 → 右值 | 左值+右值 → 结果统一为右值 |
| 右值.数据成员 | 右值保持 | xvalue 可以移动 |
| `static_cast<T&&>` | 左值 → 右值 | 显式转换，`std::move` 的本质 |

参见：[`std::move`](#stdmove)、[`std::forward`](#stdforward)、[`右值引用`](#右值引用)。

### Variadic Template(变参模板)

`...` 语法创建一个***Parameter Pack(参数包)***或展开它。模板***Parameter Pack(参数包)***是接受零个或多个模板参数（非类型、类型或模板）的模板参数。至少包含一个参数包的模板称为***Variadic Template(变参模板)***。
```c++
template <typename... T>
struct arity {
  constexpr static int value = sizeof...(T);
};
static_assert(arity<>::value == 0);
static_assert(arity<char, short, int>::value == 3);
```

一个有趣的用法是从***Parameter Pack(参数包)***创建一个***Initializer List(初始化列表)***，以便遍历变参函数参数。
```c++
template <typename First, typename... Args>
auto sum(const First first, const Args... args) -> decltype(first) {
  const auto values = {first, args...};
  return std::accumulate(values.begin(), values.end(), First{0});
}

sum(1, 2, 3, 4, 5); // 15
sum(1, 2, 3);       // 6
sum(1.5, 2.0, 3.7); // 7.2
```

### Initializer List(初始化列表)

使用"花括号列表"语法创建的轻量级数组状元素容器。例如，`{ 1, 2, 3 }` 创建一个整数序列，其类型为 `std::initializer_list<int>`。可用作向函数传递对象向量的替代方案。
```c++
int sum(const std::initializer_list<int>& list) {
  int total = 0;
  for (auto& e : list) {
    total += e;
  }

  return total;
}

auto list = {1, 2, 3};
sum(list); // == 6
sum({1, 2, 3}); // == 6
sum({}); // == 0
```

### Static Assert(静态断言)

在编译时求值的断言。
```c++
constexpr int x = 0;
constexpr int y = 1;
static_assert(x == y, "x != y");
```

### auto

`auto` 类型的变量由编译器根据其初始化器的类型推导。
```c++
auto a = 3.14; // double
auto b = 1; // int
auto& c = b; // int&
auto d = { 0 }; // std::initializer_list<int>
auto&& e = 1; // int&&
auto&& f = b; // int&
auto g = new auto(123); // int*
const auto h = 1; // const int
auto i = 1, j = 2, k = 3; // int, int, int
auto l = 1, m = true, n = 1.61; // 错误 -- `l` 推导为 int，`m` 是 bool
auto o; // 错误 -- `o` 需要初始化器
```

对于可读性非常有用，尤其是对于复杂类型：
```c++
std::vector<int> v = ...;
std::vector<int>::const_iterator cit = v.cbegin();
// 对比：
auto cit = v.cbegin();
```

函数也可以使用 `auto` 推导返回类型。在 C++11 中，必须显式指定返回类型，或使用 `decltype`：
```c++
template <typename X, typename Y>
auto add(X x, Y y) -> decltype(x + y) {
  return x + y;
}
add(1, 2); // == 3
add(1, 2.0); // == 3.0
add(1.5, 1.5); // == 3.0
```
上面示例中的返回类型后置是表达式 `x + y` 的***Declared Type(声明类型)***（参见 [`decltype`](#decltype) 部分）。例如，如果 `x` 是整数，`y` 是 double，则 `decltype(x + y)` 是 double。因此，上面的函数将根据表达式 `x + y` 产生的类型来推导类型。请注意，返回类型后置可以访问其参数，以及适当情况下的 `this`。

### Lambda Expression(Lambda 表达式)

Lambda 是一种能够捕获作用域内变量的无名函数对象。它包含：一个**捕获列表**；一组可选的参数（带可选的返回类型后置）；以及一个函数体。捕获列表示例：
* `[]` - 不捕获任何内容。
* `[=]` - 按值捕获作用域内的局部对象（局部变量、参数）。
* `[&]` - 按引用捕获作用域内的局部对象（局部变量、参数）。
* `[this]` - 按引用捕获 `this`。
* `[a, &b]` - 按值捕获对象 `a`，按引用捕获对象 `b`。

```c++
int x = 1;

auto getX = [=] { return x; };
getX(); // == 1

auto addX = [=](int y) { return x + y; };
addX(1); // == 2

auto getXRef = [&]() -> int& { return x; };
getXRef(); // `x` 的 int&
```

#### 按值捕获、按引用捕获与 mutable 的关系

理解这三者的关键是：**Lambda 本质上是一个匿名函数对象，编译器生成的 `operator()` 默认被标记为 `const`**，因此无法在函数体内修改任何成员（即捕获的变量）。`mutable` 的作用就是移除这个 `const` 限定。

**1. 按值捕获 `[x]`**

按值捕获时，lambda 内部持有一份**外部变量的拷贝**。由于 `operator()` 是 `const`，默认不能修改这份拷贝。`mutable` 允许修改拷贝，但**无论如何都不影响外部原始变量**。

```c++
int x = 1;

// 按值捕获，operator() 是 const → 不能修改
auto f1 = [x] { return x + 1; };    // OK: 只读
// auto f2 = [x] { x = 2; };        // ERROR: const 成员函数不能修改成员
// auto f3 = [x] { ++x; };          // ERROR: 同上

// 按值捕获 + mutable，operator() 不是 const → 可以修改内部拷贝
auto f4 = [x]() mutable { x = 2; return x; };
f4();       // 返回 2，lambda 内部拷贝被改为 2
// x 仍然是 1  ← 外部变量完全不受影响
```

**2. 按引用捕获 `[&x]`**

按引用捕获时，lambda 内部持有的是**外部变量的引用**。修改引用指向的对象不需要 `operator()` 本身是 non-const（就像 `const` 成员函数中可以通过 `int&` 成员修改外部变量一样）。因此**按引用捕获不需要 `mutable` 就能修改外部变量**。

```c++
int x = 1;

auto f1 = [&x] { x = 2; };          // OK: 通过引用修改外部变量，不需要 mutable
f1();
// x == 2  ← 外部变量被修改了

auto f2 = [&x] { ++x; };            // OK: 同样不需要 mutable
f2();
// x == 3
```

**3. 总结对比**

| 捕获方式 | 不加 mutable | 加 mutable | 影响外部变量？ |
|:---:|:---|:---|:---:|
| 按值 `[x]` | 只读，不能修改 | 可修改 lambda 内部的拷贝 | **永不**影响 |
| 按引用 `[&x]` | 可修改，影响外部 | 可修改（多余，但合法） | **始终**影响 |

**4. 混合捕获的完整示例**

```c++
int a = 1, b = 2, c = 3;

auto lambda = [a, &b, &c]() mutable {
  a = 10;   // OK: mutable 允许修改按值捕获的拷贝，外部 a 仍是 1
  b = 20;   // OK: 引用捕获，不需要 mutable，外部 b 被改为 20
  c = 30;   // OK: 同上
  // 无法在此访问未列出的变量
};

lambda();
// a == 1   (未变：按值捕获只是拷贝)
// b == 20  (已变：按引用捕获)
// c == 30  (已变：按引用捕获)
```

**记忆口诀**：
- 按值 = 复制一份，修改需要 `mutable`，永远不影响外部
- 按引用 = 别名，直接就能改，不需要 `mutable`，始终影响外部
- `mutable` 只对按值捕获有意义，对引用捕获来说是多余的

### decltype

`decltype` 是一个运算符，返回传递给它的表达式的***Declared Type(声明类型)***。如果 cv 限定符和引用是表达式的一部分，则会保留它们。`decltype` 的示例：
```c++
int a = 1; // `a` 声明为类型 `int`
decltype(a) b = a; // `decltype(a)` 是 `int`
const int& c = a; // `c` 声明为类型 `const int&`
decltype(c) d = a; // `decltype(c)` 是 `const int&`
decltype(123) e = 123; // `decltype(123)` 是 `int`
int&& f = 1; // `f` 声明为类型 `int&&`
decltype(f) g = 1; // `decltype(f)` 是 `int&&`
decltype((a)) h = g; // `decltype((a))` 是 int&
```
```c++
template <typename X, typename Y>
auto add(X x, Y y) -> decltype(x + y) {
  return x + y;
}
add(1, 2.0); // `decltype(x + y)` => `decltype(3.0)` => `double`
```

参见：[`decltype(auto) (C++14)`](README_zh.md#decltypeauto)。

### Type Alias(类型别名)

语义上类似于使用 `typedef`，但是使用 `using` 的类型别名更容易阅读并且与模板兼容。
```c++
template <typename T>
using Vec = std::vector<T>;
Vec<int> v; // std::vector<int>

using String = std::string;
String s {"foo"};
```

### nullptr

C++11 引入了一种新的空指针类型，旨在取代 C 的 `NULL` 宏。`nullptr` 本身的类型是 `std::nullptr_t`，可以隐式转换为指针类型，并且与 `NULL` 不同，除了 `bool` 之外不能转换为整数类型。
```c++
void foo(int);
void foo(char*);
foo(NULL); // 错误 -- 歧义
foo(nullptr); // 调用 foo(char*)
```

### Strongly Typed Enum(强类型枚举)

类型安全的枚举，解决了 C 风格枚举的各种问题，包括：隐式转换、无法指定底层类型、作用域污染。
```c++
// 指定底层类型为 `unsigned int`
enum class Color : unsigned int { Red = 0xff0000, Green = 0xff00, Blue = 0xff };
// `Alert` 中的 `Red`/`Green` 与 `Color` 不冲突
enum class Alert : bool { Red, Green };
Color c = Color::Red;
```

### Attribute(属性)

提供了一种统一的语法来替代 `__attribute__(...)`、`__declspec` 等。
```c++
// `noreturn` 属性表示 `f` 不返回。
[[ noreturn ]] void f() {
  throw "error";
}
```

### constexpr

常量表达式是编译器可能在编译时求值的表达式。常量表达式中只能执行非复杂计算（这些规则在后续版本中逐渐放宽）。使用 `constexpr` 说明符来表示变量、函数等是常量表达式。
```c++
constexpr int square(int x) {
  return x * x;
}

int square2(int x) {
  return x * x;
}

int a = square(2);  // mov DWORD PTR [rbp-4], 4

int b = square2(2); // mov edi, 2
                    // call square2(int)
                    // mov DWORD PTR [rbp-8], eax
```
在前面的代码片段中，请注意调用 `square` 时的计算是在编译时执行的，然后将结果嵌入到代码生成中，而 `square2` 是在运行时调用的。

`constexpr` 值是编译器可以求值的，但不保证在编译时求值：
```c++
const int x = 123;
constexpr const int& y = x; // 错误 -- constexpr 变量 `y` 必须由常量表达式初始化
```

类的常量表达式：
```c++
struct Complex {
  constexpr Complex(double r, double i) : re{r}, im{i} { }
  constexpr double real() { return re; }
  constexpr double imag() { return im; }

private:
  double re;
  double im;
};

constexpr Complex I(0, 1);
```

### Delegating Constructor(委托构造函数)

构造函数现在可以使用初始化列表调用同一类中的其他构造函数。
```c++
struct Foo {
  int foo;
  Foo(int foo) : foo{foo} {}
  Foo() : Foo(0) {}
};

Foo foo;
foo.foo; // == 0
```

### User-Defined Literal(用户自定义字面量)

用户自定义字面量允许您扩展语言并添加自己的语法。要创建字面量，定义一个 `T operator "" X(...) { ... }` 函数，返回类型 `T`，名称为 `X`。请注意，此函数的名称定义了字面量的名称。任何不以下划线开头的字面量名称都是保留的，不会被调用。根据字面量所调用的类型，用户自定义字面量函数应该接受什么参数有相应的规则。

摄氏度转换为华氏度：
```c++
// 整数字面量需要 `unsigned long long` 参数。
long long operator "" _celsius(unsigned long long tempCelsius) {
  return std::llround(tempCelsius * 1.8 + 32);
}
24_celsius; // == 75
```

字符串转整数：
```c++
// 需要 `const char*` 和 `std::size_t` 作为参数。
int operator "" _int(const char* str, std::size_t) {
  return std::stoi(str);
}

"123"_int; // == 123，类型为 `int`
```

### Explicit Virtual Override(显式虚函数覆盖)

指定虚函数覆盖另一个虚函数。如果虚函数没有覆盖父类的虚函数，会抛出编译错误。
```c++
struct A {
  virtual void foo();
  void bar();
};

struct B : A {
  void foo() override; // 正确 -- B::foo 覆盖 A::foo
  void bar() override; // 错误 -- A::bar 不是虚函数
  void baz() override; // 错误 -- B::baz 没有覆盖 A::baz
};
```

### Final Specifier(final 说明符)

指定虚函数不能在派生类中被覆盖，或者类不能被继承。

**注意**：`final` 只阻止子类覆盖，**不要求当前类必须提供实现**。如果当前类没有重写，则沿用基类的实现，同时禁止子类再做覆盖。

```c++
struct A {
  virtual void foo() { /* A 的实现 */ }
};

struct B : A {
  virtual void foo() final;  // 声明为 final，但未提供实现
                             // 效果：沿用 A::foo，且禁止子类覆盖
};

struct C : B {
  virtual void foo(); // 错误 -- 'foo' 的声明覆盖了一个 'final' 函数
};
```

类不能被继承。
```c++
struct A final {};
struct B : A {}; // 错误 -- 基类 'A' 被标记为 'final'
```

### Default Function(默认函数)

提供函数默认实现的更优雅、高效的方式，例如构造函数。
```c++
struct A {
  A() = default;
  A(int x) : x{x} {}
  int x {1};
};
A a; // a.x == 1
A a2 {123}; // a.x == 123
```

带继承：
```c++
struct B {
  B() : x{1} {}
  int x;
};

struct C : B {
  // 调用 B::B
  C() = default;
};

C c; // c.x == 1
```

### Deleted Function(删除函数)

提供函数删除实现的更优雅、高效的方式。用于防止对象复制。
```c++
class A {
  int x;

public:
  A(int x) : x{x} {};
  A(const A&) = delete;
  A& operator=(const A&) = delete;
};

A x {123};
A y = x; // 错误 -- 调用已删除的复制构造函数
y = x; // 错误 -- operator= 已删除
```

### Range-Based For Loop(范围 for 循环)

用于遍历容器元素的语法糖。
```c++
std::array<int, 5> a {1, 2, 3, 4, 5};
for (int& x : a) x *= 2;
// a == { 2, 4, 6, 8, 10 }
```

注意使用 `int` 和 `int&` 的区别：
```c++
std::array<int, 5> a {1, 2, 3, 4, 5};
for (int x : a) x *= 2;
// a == { 1, 2, 3, 4, 5 }
```

### Move Semantics(移动语义)的特殊成员函数

复制构造函数和复制赋值运算符在进行复制时被调用，随着 C++11 引入移动语义，现在有了移动构造函数和移动赋值运算符用于移动操作。
```c++
struct A {
  std::string s;
  A() : s{"test"} {}
  A(const A& o) : s{o.s} {}
  A(A&& o) : s{std::move(o.s)} {}
  A& operator=(A&& o) {
   s = std::move(o.s);
   return *this;
  }
};

A f(A a) {
  return a;
}

A a1 = f(A{}); // A{} 是右值(prvalue)临时对象，a1 是类型为 A 的具名变量(左值)
A a2 = std::move(a1); // 使用 std::move 将 a1 转为右值引用，移动构造 a2
A a3 = A{};
a2 = std::move(a3); // 使用 std::move 移动赋值
a1 = f(A{}); // 从右值临时对象移动赋值
```

### Converting Constructor(转换构造函数)

转换构造函数会将花括号列表语法的值转换为构造函数参数。
```c++
struct A {
  A(int) {}
  A(int, int) {}
  A(int, int, int) {}
};

A a {0, 0}; // 调用 A::A(int, int)
A b(0, 0); // 调用 A::A(int, int)
A c = {0, 0}; // 调用 A::A(int, int)
A d {0, 0, 0}; // 调用 A::A(int, int, int)
```

注意花括号列表语法不允许窄化转换：
```c++
struct A {
  A(int) {}
};

A a(1.1); // OK
A b {1.1}; // 错误：从 double 到 int 的窄化转换
```

注意如果构造函数接受 `std::initializer_list`，它将被优先调用：
```c++
struct A {
  A(int) {}
  A(int, int) {}
  A(int, int, int) {}
  A(std::initializer_list<int>) {}
};

A a {0, 0}; // 调用 A::A(std::initializer_list<int>)
A b(0, 0); // 调用 A::A(int, int)
A c = {0, 0}; // 调用 A::A(std::initializer_list<int>)
A d {0, 0, 0}; // 调用 A::A(std::initializer_list<int>)
```

### Explicit Conversion Function(显式转换函数)

C++11 允许转换函数（`operator T()`）使用 `explicit` 说明符，防止在某些上下文中发生隐式转换。`explicit` 转换函数在以下语境中**仍可被调用**：

* 显式类型转换：`static_cast<bool>(obj)`
* 直接初始化：`bool b(obj)` 或 `bool b{obj}`
* **"布尔语境"**：`if`、`while`、`for` 的条件，`!`、`&&`、`||` 的操作数，`?:` 的条件部分

`explicit` 转换函数**被禁止**的语境：

* 复制初始化：`bool b = obj;`
* 隐式函数参数转换：将对象传给期望 `bool` 的函数参数
* 隐式返回值转换：函数返回 `bool` 时返回该对象

```c++
struct A {
  operator bool() const { return true; }        // 隐式转换
};

struct B {
  explicit operator bool() const { return true; } // 显式转换
};

A a;
if (a);              // OK: 布尔语境，隐式转换允许
bool ba1 = a;        // OK: 复制初始化，隐式转换允许
bool ba2(a);         // OK: 直接初始化
bool ba3 = static_cast<bool>(a); // OK: 显式转换
void fa(bool v);
fa(a);               // OK: 隐式转换为函数参数

B b;
if (b);              // OK: 布尔语境始终允许 explicit 转换
bool bb1 = b;        // 错误: 复制初始化不考虑 explicit 转换
bool bb2(b);         // OK: 直接初始化考虑 explicit 转换
bool bb3 = static_cast<bool>(b); // OK: 显式转换
!b;                  // OK: ! 运算符属于布尔语境
b && true;           // OK: && 运算符属于布尔语境
void fb(bool v);
fb(b);               // 错误: 隐式转换为函数参数，explicit 不允许
// fb(static_cast<bool>(b)); // OK: 显式转换后可以
```

### Inline Namespace(内联命名空间)

内联命名空间的所有成员都被视为其父命名空间的一部分，允许函数特化并简化版本控制过程。这是一个传递属性，如果 A 包含 B，B 又包含 C，且 B 和 C 都是内联命名空间，则 C 的成员可以像在 A 上一样使用。

```c++
namespace Program {
  namespace Version1 {
    int getVersion() { return 1; }
    bool isFirstVersion() { return true; }
  }
  inline namespace Version2 {
    int getVersion() { return 2; }
  }
}

int version {Program::getVersion()};              // 调用 Version2::getVersion()，返回 2
int oldVersion {Program::Version1::getVersion()}; // 显式调用 Version1::getVersion()，返回 1
bool firstVersion {Program::isFirstVersion()};    // 错误: isFirstVersion 在 Version1 中，但 Version1 不是内联命名空间
                                                  // 必须写成 Program::Version1::isFirstVersion()
```

### Non-Static Data Member Initializer(非静态数据成员初始化器)

允许非静态数据成员在声明时进行初始化，可能简化构造函数中的默认初始化。

```c++
// C++11 之前的默认初始化
class Human {
    Human() : age{0} {}
  private:
    unsigned age;
};
// C++11 中的默认初始化
class Human {
  private:
    unsigned age {0};
};
```

### Right Angle Bracket(右尖括号)

C++11 现在能够推断一系列右尖括号是用作运算符还是作为 typedef 的结束语句，而无需添加空格。

```c++
typedef std::map<int, std::map <int, std::map <int, int> > > cpp98LongTypedef;
typedef std::map<int, std::map <int, std::map <int, int>>>   cpp11LongTypedef;
```

### Ref-Qualified Member Function(引用限定成员函数)

C++11 允许在成员函数声明的末尾加上 `&` 或 `&&`，用来限定**调用该函数的对象（`*this`）必须是什么值类别**。这是对 `const` 成员函数的扩展——`const` 限定的是 `*this` 的常量性，而 `&`/`&&` 限定的是 `*this` 的左值/右值性。

**动机**：当对象是右值（临时对象、`std::move` 后的对象）时，可以安全地"偷走"其内部资源；当对象是左值时，应该返回引用，避免不必要的拷贝。

**四种重载组合**：

| 声明 | 可调用条件 | 典型用途 |
|:---|:---|:---|
| `R f() &` | `*this` 是非 const 左值 | 返回内部成员的引用 |
| `R f() const&` | `*this` 是 const 左值 | 返回内部成员的 const 引用 |
| `R f() &&` | `*this` 是非 const 右值 | 移动内部资源，返回右值引用 |
| `R f() const&&` | `*this` 是 const 右值 | 极少使用，const 右值无法移动 |

```c++
struct Bar {
  // ...
};

struct Foo {
  Bar& getBar() &             { return bar; }           // ① 左值 → 返回引用
  const Bar& getBar() const&  { return bar; }           // ② const 左值 → 返回 const 引用
  Bar&& getBar() &&           { return std::move(bar); } // ③ 右值 → 移动内部资源
  const Bar&& getBar() const&&{ return std::move(bar); } // ④ const 右值 → const 右值引用
private:
  Bar bar;
};

Foo foo{};                         // foo 是非 const 左值
const Foo foo2{};                  // foo2 是 const 左值

// foo 是非 const 左值 → *this 是 Foo& → 匹配 ①
Bar bar1 = foo.getBar();           // 调用 Bar& getBar() &

// foo2 是 const 左值 → *this 是 const Foo& → 匹配 ②
const Bar& bar2ref = foo2.getBar();// 调用 const Bar& getBar() const&

// Foo{} 是右值(临时对象) → *this 是 Foo&& → 匹配 ③
Foo{}.getBar();                    // 调用 Bar&& getBar() &&

// std::move(foo) 将 foo 转为右值 → *this 是 Foo&& → 匹配 ③
std::move(foo).getBar();           // 调用 Bar&& getBar() &&

// std::move(foo2) 将 foo2 转为 const 右值 → *this 是 const Foo&& → 匹配 ④
std::move(foo2).getBar();          // 调用 const Bar&& getBar() const&&
```

**调度规则**：编译器根据 `*this` 的**常量性**（const 与否）和**值类别**（左值/右值）选择最匹配的重载。
- 非 const 左值 → 先匹配 `&`，没有则匹配 `const&`
- const 左值 → 只能匹配 `const&`
- 非 const 右值 → 先匹配 `&&`，没有则匹配 `const&&`
- const 右值 → 只能匹配 `const&&`

**关于返回值与限定符"一致"的规律**：

上面示例中返回值类型和函数限定符看起来一一对应（`Bar&` 配 `&`，`const Bar&` 配 `const&`，`Bar&&` 配 `&&`），这是**设计惯例，而非语言强制要求**。你可以写 `Bar& getBar() &&`，编译器不会报错，但不合理——因为 `*this` 是右值时对象即将销毁，返回左值引用会让调用方持有一个将死对象的引用，这是危险的。正确做法是：

- `*this` 是左值（`&`/`const&`）→ 返回引用（`Bar&`/`const Bar&`），因为对象还活着
- `*this` 是右值（`&&`/`const&&`）→ 返回右值引用（`Bar&&`/`const Bar&&`），允许移动资源
- const 性传递：`*this` 是 const → 返回的引用也应是 const

一言以蔽之：**限定符描述"我是什么"，返回值描述"我给你什么"，两者应语义一致**。

### Trailing Return Type(返回类型后置)

C++11 允许函数和 lambda 使用替代语法来指定其返回类型。
```c++
int f() {
  return 123;
}
// 对比：
auto f() -> int {
  return 123;
}
```
```c++
auto g = []() -> int {
  return 123;
};
```
此特性在某些返回类型无法解析时特别有用：
```c++
// 注意：这无法编译！
template <typename T, typename U>
decltype(a + b) add(T a, U b) {
    return a + b;
}

// 返回类型后置允许这样做：
template <typename T, typename U>
auto add(T a, U b) -> decltype(a + b) {
    return a + b;
}
```
在 C++14 中，可以改用 [`decltype(auto) (C++14)`](README_zh.md#decltypeauto)。

### Noexcept Specifier(noexcept 说明符)

`noexcept` 说明符指定函数是否可能抛出异常。它是 `throw()` 的改进版本（`throw()` 在 C++11 中已废弃，C++17 中移除）。

```c++
void func1() noexcept;        // 不抛出
void func2() noexcept(true);  // 不抛出
void func3() throw();         // 不抛出（废弃语法）

void func4() noexcept(false); // 可能抛出
```

**异常从 `noexcept` 函数逃逸会怎样？**

如果一个异常在 `noexcept` 函数内部没有被捕获，试图逃逸出该函数时，**`std::terminate()` 会被立即调用，进程终止**。这与普通函数不同——普通函数中未被捕获的异常会沿调用栈向上传播，逐层执行栈展开（调用析构函数），只有当异常最终到达 `main()` 仍未被捕获时，才调用 `std::terminate()`。

```c++
extern void f();  // 可能抛出

void g() noexcept {
    f();          // 如果 f() 抛出且未在 g() 内部捕获，std::terminate() 立即触发
    throw 42;     // 同样，std::terminate() 立即触发，进程终止
}
```

对比：同样的代码去掉 `noexcept`，异常会正常向上传播，调用方有机会 `catch`。

**`noexcept` 的作用是什么？**

1. **编译器优化**：编译器知道该函数不会抛出，可以省略栈展开相关的代码，生成更高效的机器码
2. **影响移动语义**：`std::vector` 在扩容时，只有移动构造函数标记为 `noexcept` 才会使用移动而非拷贝——这是强异常安全保证的要求（如果移动可能抛异常，`vector` 无法保证在扩容失败时回滚到一致状态）
3. **文档意图**：明确告诉调用方"这个函数不会抛异常"，便于调用方做决策

**`noexcept` 运算符**：

`noexcept(表达式)` 是一个编译期运算符，返回 `true` 如果表达式不会抛出异常：

```c++
void may_throw();
void no_throw() noexcept;

bool b1 = noexcept(may_throw());  // false
bool b2 = noexcept(no_throw());   // true
```

这常用于泛型编程中根据类型的异常特性做条件编译。

### char32_t and char16_t(char32_t 和 char16_t)

提供用于表示 UTF-8 字符串的标准类型。
```c++
char32_t utf8_str[] = U"\u0123";
char16_t utf8_str[] = u"\u0123";
```

### Raw String Literal(原始字符串字面量)

C++11 引入了一种新的方式来声明字符串字面量，称为"原始字符串字面量"。转义序列（制表符、换行符、单个反斜杠等）产生的字符可以原始输入，同时保留格式。这在编写文学文本时很有用，因为文学文本可能包含大量引号或特殊格式。这可以使您的字符串字面量更易于阅读和维护。

原始字符串字面量使用以下语法声明：
```
R"delimiter(raw_characters)delimiter"
```
其中：
* `delimiter` 是可选的字符序列，由除括号、反斜杠和空格之外的任何源字符组成。
* `raw_characters` 是任何原始字符序列；不得包含结束序列 `")delimiter"`。

示例：
```cpp
// msg1 和 msg2 是等价的。
const char* msg1 = "\nHello,\n\tworld!\n";
const char* msg2 = R"(
Hello,
	world!
)";
```

## C++11 标准库特性

### std::move

`std::move` 表示传递给它的对象可能会转移其资源。使用已被移动的对象时应小心，因为它们可能处于未指定状态（参见：[What can I do with a moved-from object?](http://stackoverflow.com/questions/7027523/what-can-i-do-with-a-moved-from-object)）。

`std::move` 的定义（执行移动只不过是转换为右值引用）：
```c++
template <typename T>
typename remove_reference<T>::type&& move(T&& arg) {
  return static_cast<typename remove_reference<T>::type&&>(arg);
}
```

转移 `std::unique_ptr`：
```c++
// 方式一：直接 new + unique_ptr（不推荐）
std::unique_ptr<int> p1 {new int{0}};

// 方式二：使用 std::make_unique（推荐，C++14 起）
// auto p1 = std::make_unique<int>(0);

// 为什么推荐 make_unique？
// 1. 异常安全：如果 new 和 unique_ptr 构造之间有其他操作抛异常，裸 new 可能泄漏
// 2. 代码简洁：无需重复写类型，无需显式 new/delete
// 3. 无 new/delete 配对错误风险
```

```c++
std::unique_ptr<int> p2 = p1;           // 错误: unique_ptr 独占所有权，禁止复制
std::unique_ptr<int> p3 = std::move(p1); // 正确: 将所有权从 p1 转移到 p3
                                         // 此后 p1 内部指针被置为 nullptr
                                         // 解引用 p1（如 *p1）是未定义行为
```

### std::forward

返回传递给它的参数，同时保持其值类别和 cv 限定符。对泛型代码和工厂函数很有用。与[转发引用](#转发引用)一起使用。

`std::forward` 的定义：
```c++
template <typename T>
T&& forward(typename remove_reference<T>::type& arg) {
  return static_cast<T&&>(arg);
}
```

函数 `wrapper` 的示例，它只是将其他 `A` 对象转发给新 `A` 对象的复制或移动构造函数：
```c++
struct A {
  A() = default;
  A(const A& o) { std::cout << "copied" << std::endl; }
  A(A&& o) { std::cout << "moved" << std::endl; }
};

template <typename T>
A wrapper(T&& arg) {
  return A{std::forward<T>(arg)};
}

wrapper(A{}); // moved
A a;
wrapper(a); // copied
wrapper(std::move(a)); // moved
```

参见：[转发引用](#转发引用)、[右值引用](#右值引用)。

### std::thread

`std::thread` 库提供了控制线程的标准方式，例如启动和终止线程。在下面的示例中，启动多个线程来执行不同的计算，然后程序等待所有线程完成。

```c++
void foo(bool clause) { /* do something... */ }

std::vector<std::thread> threadsVector;
threadsVector.emplace_back([]() {
  // 将被调用的 Lambda 函数
});
threadsVector.emplace_back(foo, true);  // 线程将运行 foo(true)
for (auto& thread : threadsVector) {
  thread.join(); // 等待线程完成
}
```

### std::to_string

将数值参数转换为 `std::string`。
```c++
std::to_string(1.2); // == "1.2"
std::to_string(123); // == "123"
```

### Type Trait(类型特征)

类型特征定义了一个基于模板的编译时接口，用于查询或修改类型的属性。
```c++
static_assert(std::is_integral<int>::value);
static_assert(std::is_same<int, int>::value);
static_assert(std::is_same<std::conditional<true, int, double>::type, int>::value);
```

### Smart Pointer(智能指针)

C++11 引入了新的智能指针：`std::unique_ptr`、`std::shared_ptr`、`std::weak_ptr`。`std::auto_ptr` 现在已弃用，并最终在 C++17 中移除。

`std::unique_ptr` 是一个不可复制、可移动的指针，它管理自己的堆分配内存。**注意：建议使用 `std::make_X` 辅助函数而不是构造函数。参见 [std::make_unique](https://github.com/AnthonyCalandra/modern-cpp-features/blob/master/CPP14.md#stdmake_unique) 和 [std::make_shared](#stdmake_shared) 章节。**
```c++
std::unique_ptr<Foo> p1 { new Foo{} };  // `p1` 拥有 `Foo`
if (p1) {
  p1->bar();
}

{
  std::unique_ptr<Foo> p2 {std::move(p1)};  // 现在 `p2` 拥有 `Foo`
  f(*p2);

  p1 = std::move(p2);  // 所有权返回给 `p1` -- `p2` 被销毁
}

if (p1) {
  p1->bar();
}
// `Foo` 实例在 `p1` 离开作用域时被销毁
```

`std::shared_ptr` 是一个智能指针，管理由多个所有者共享的资源。共享指针持有一个**控制块**，其中包含托管对象和引用计数器等几个组件。所有控制块访问都是线程安全的，但操作托管对象本身**不是**线程安全的。
```c++
void foo(std::shared_ptr<T> t) {
  // 对 `t` 执行某些操作...
}

void bar(std::shared_ptr<T> t) {
  // 对 `t` 执行某些操作...
}

void baz(std::shared_ptr<T> t) {
  // 对 `t` 执行某些操作...
}

std::shared_ptr<T> p1 {new T{}};
// 也许这些在其他线程中执行？
foo(p1);
bar(p1);
baz(p1);
```

### std::chrono

chrono 库包含一组处理**持续时间**、**时钟**和**时间点**的实用函数和类型。该库的一个用例是代码基准测试：
```c++
std::chrono::time_point<std::chrono::steady_clock> start, end;
start = std::chrono::steady_clock::now();
// 一些计算...
end = std::chrono::steady_clock::now();

std::chrono::duration<double> elapsed_seconds = end - start;
double t = elapsed_seconds.count(); // t 是秒数，表示为 `double`
```

### Tuple(元组)

元组是固定大小的异构值集合。通过使用 [`std::tie`](#stdtie) 解包或使用 `std::get` 来访问 `std::tuple` 的元素。
```c++
// `playerProfile` 的类型为 `std::tuple<int, const char*, const char*>`。
auto playerProfile = std::make_tuple(51, "Frans Nielsen", "NYI");
std::get<0>(playerProfile); // 51
std::get<1>(playerProfile); // "Frans Nielsen"
std::get<2>(playerProfile); // "NYI"
```

### std::tie

创建左值引用的元组。用于解包 `std::pair` 和 `std::tuple` 对象。使用 `std::ignore` 作为忽略值的占位符。在 C++17 中，应改用结构化绑定。
```c++
// 用于元组...
std::string playerName;
std::tie(std::ignore, playerName, std::ignore) = std::make_tuple(91, "John Tavares", "NYI");

// 用于 pair...
std::string yes, no;
std::tie(yes, no) = std::make_pair("yes", "no");
```

### std::array

`std::array` 是构建在 C 风格数组之上的容器。支持常见的容器操作，如排序。
```c++
std::array<int, 3> a = {2, 1, 3};
std::sort(a.begin(), a.end()); // a == { 1, 2, 3 }
for (int& x : a) x *= 2; // a == { 2, 4, 6 }
```

### Unordered Container(无序容器)

这些容器在搜索、插入和删除操作上保持平均常数时间复杂度。为了实现常数时间复杂度，通过将元素哈希到桶中来牺牲顺序换取速度。有四种无序容器：
* `unordered_set`
* `unordered_multiset`
* `unordered_map`
* `unordered_multimap`

### std::make_shared

`std::make_shared` 是创建 `std::shared_ptr` 实例的推荐方式，原因如下：
* 避免使用 `new` 运算符。
* 防止在指定指针应持有的底层类型时代码重复。
* 提供异常安全。假设我们像这样调用函数 `foo`：
```c++
foo(std::shared_ptr<T>{new T{}}, function_that_throws(), std::shared_ptr<T>{new T{}});
```
编译器可以自由调用 `new T{}`，然后 `function_that_throws()`，等等... 由于我们在第一次构造 `T` 时在堆上分配了数据，我们在这里引入了内存泄漏。使用 `std::make_shared`，我们获得了异常安全：
```c++
foo(std::make_shared<T>(), function_that_throws(), std::make_shared<T>());
```
* 防止必须进行两次分配。调用 `std::shared_ptr{ new T{} }` 时，我们必须为 `T` 分配内存，然后在共享指针中必须为指针内的控制块分配内存。

有关 `std::unique_ptr` 和 `std::shared_ptr` 的更多信息，请参见[智能指针](#智能指针)部分。

### std::ref

`std::ref(val)` 用于创建 `std::reference_wrapper` 类型的对象，该对象持有 val 的引用。用于普通引用传递使用 `&` 无法编译或 `&` 因类型推导而丢失的情况。`std::cref` 类似，但创建的引用包装器持有对 val 的 const 引用。

```c++
// 创建一个容器来存储对象的引用。
auto val = 99;
auto _ref = std::ref(val);
_ref++;
auto _cref = std::cref(val);
//_cref++; 无法编译
std::vector<std::reference_wrapper<int>>vec; // vector<int&>vec 无法编译
vec.push_back(_ref); // vec.push_back(&i) 无法编译
cout << val << endl; // 输出 100
cout << vec[0] << endl; // 输出 100
cout << _cref; // 输出 100
```

### Memory Model(内存模型)

C++11 为 C++ 引入了内存模型，这意味着对线程和原子操作的库支持。这些操作包括（但不限于）原子加载/存储、比较并交换、原子标志、承诺、未来、锁和条件变量。

参见：[std::thread](#stdthread)

### std::async

`std::async` 异步或延迟计算给定函数，然后返回一个 `std::future`，该 future 持有该函数调用的结果。

第一个参数是策略，可以是：
1. `std::launch::async | std::launch::deferred` 由实现决定是执行异步执行还是延迟计算。
1. `std::launch::async` 在新线程上运行可调用对象。
1. `std::launch::deferred` 在当前线程上执行延迟计算。

```c++
int foo() {
  /* 在这里执行某些操作，然后返回结果。 */
  return 1000;
}

auto handle = std::async(std::launch::async, foo);  // 创建异步任务
auto result = handle.get();  // 等待结果
```

### std::begin/end

添加了 `std::begin` 和 `std::end` 自由函数，用于泛型返回容器的开始和结束迭代器。这些函数也适用于没有 `begin` 和 `end` 成员函数的原始数组。

```c++
template <typename T>
int CountTwos(const T& container) {
  return std::count_if(std::begin(container), std::end(container), [](int item) {
    return item == 2;
  });
}

std::vector<int> vec = {2, 2, 43, 435, 4543, 534};
int arr[8] = {2, 43, 45, 435, 32, 32, 32, 32};
auto a = CountTwos(vec); // 2
auto b = CountTwos(arr);  // 1
```

## 致谢
* [cppreference](http://en.cppreference.com/w/cpp) - 特别有助于查找新库特性的示例和文档。
* [C++ Rvalue References Explained](http://web.archive.org/web/20240324121501/http://thbecker.net/articles/rvalue_references/section_01.html) - 我用来理解右值引用、完美转发和移动语义的很好介绍。
* [clang](http://clang.llvm.org/cxx_status.html) 和 [gcc](https://gcc.gnu.org/projects/cxx-status.html) 的标准支持页面。这里还包含了我用来帮助查找语言/库特性描述、它旨在解决的问题以及一些示例的提案。
* [Compiler explorer](https://godbolt.org/)
* [Scott Meyers' Effective Modern C++](https://www.amazon.com/Effective-Modern-Specific-Ways-Improve/dp/1491903996) - 强烈推荐的书！
* [Jason Turner's C++ Weekly](https://www.youtube.com/channel/UCxHAlbZQNFU2LgEtiqd2Maw) - 不错的 C++ 相关视频合集。
* [What can I do with a moved-from object?](http://stackoverflow.com/questions/7027523/what-can-i-do-with-a-moved-from-object)
* [What are some uses of decltype(auto)?](http://stackoverflow.com/questions/24109737/what-are-some-uses-of-decltypeauto)
* 以及更多我忘记的 SO 帖子...

## 作者
Anthony Calandra

## 内容贡献者
参见：https://github.com/AnthonyCalandra/modern-cpp-features/graphs/contributors

## 许可证
MIT