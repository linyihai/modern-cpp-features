# C++11

## 概述

以下描述和示例来自各种资源（见[致谢](#致谢)部分），并以我自己的话进行了总结。

C++11 包含以下新语言特性：
- [Move Semantics(移动语义)](#移动语义)
- [Variadic Template(变参模板)](#变参模板)
- [Rvalue Reference(右值引用)](#右值引用)
- [Forwarding Reference(转发引用)](#转发引用)
- [Initializer List(初始化列表)](#初始化列表)
- [Static Assert(静态断言)](#静态断言)
- [auto](#auto)
- [Lambda Expression(Lambda 表达式)](#lambda-表达式)
- [decltype](#decltype)
- [Type Alias(类型别名)](#类型别名)
- [nullptr](#nullptr)
- [Strongly Typed Enum(强类型枚举)](#强类型枚举)
- [Attribute(属性)](#属性)
- [constexpr](#constexpr)
- [Delegating Constructor(委托构造函数)](#委托构造函数)
- [User-Defined Literal(用户自定义字面量)](#用户自定义字面量)
- [Explicit Virtual Override(显式虚函数覆盖)](#显式虚函数覆盖)
- [Final Specifier(final 说明符)](#final-说明符)
- [Default Function(默认函数)](#默认函数)
- [Deleted Function(删除函数)](#删除函数)
- [Range-Based For Loop(范围 for 循环)](#范围-for-循环)
- [Special Member Functions for Move Semantics(移动语义的特殊成员函数)](#移动语义的特殊成员函数)
- [Converting Constructor(转换构造函数)](#转换构造函数)
- [Explicit Conversion Function(显式转换函数)](#显式转换函数)
- [Inline Namespace(内联命名空间)](#内联命名空间)
- [Non-Static Data Member Initializer(非静态数据成员初始化器)](#非静态数据成员初始化器)
- [Right Angle Bracket(右尖括号)](#右尖括号)
- [Ref-Qualified Member Function(引用限定成员函数)](#引用限定成员函数)
- [Trailing Return Type(返回类型后置)](#返回类型后置)
- [Noexcept Specifier(noexcept 说明符)](#noexcept-说明符)
- [char32_t and char16_t(char32_t 和 char16_t)](#char32_t-和-char16_t)
- [Raw String Literal(原始字符串字面量)](#原始字符串字面量)

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

也称为（非官方）**万能引用**。当 `T` 是模板类型参数或使用 `auto&&` 时，使用语法 `T&&` 创建转发引用。这实现了***Perfect Forwarding(完美转发)***：在传递参数时保持其值类别（例如，左值保持为左值，临时对象作为右值转发）。

转发引用允许引用根据类型绑定到左值或右值。转发引用遵循***Reference Collapsing(引用折叠)***规则：
* `T& &` 变为 `T&`
* `T& &&` 变为 `T&`
* `T&& &` 变为 `T&`
* `T&& &&` 变为 `T&&`

`auto` 类型推导与左值和右值：
```c++
int x = 0; // `x` 是类型为 `int` 的左值
auto&& al = x; // `al` 是类型为 `int&` 的左值 -- 绑定到左值 `x`
auto&& ar = 0; // `ar` 是类型为 `int&&` 的左值 -- 绑定到右值临时对象 `0`
```

模板类型参数推导与左值和右值：
```c++
// 从 C++14 或更高版本开始：
void f(auto&& t) {
  // ...
}

// 从 C++11 或更高版本开始：
template <typename T>
void f(T&& t) {
  // ...
}

int x = 0;
f(0); // T 是 int，推导为 f(int &&) => f(int&&)
f(x); // T 是 int&，推导为 f(int& &&) => f(int&)

int& y = x;
f(y); // T 是 int&，推导为 f(int& &&) => f(int&)

int&& z = 0; // 注意：`z` 是类型为 `int&&` 的左值。
f(z); // T 是 int&，推导为 f(int& &&) => f(int&)
f(std::move(z)); // T 是 int，推导为 f(int &&) => f(int&&)
```

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
默认情况下，按值捕获的变量不能在 lambda 内部修改，因为编译器生成的方法被标记为 `const`。`mutable` 关键字允许修改捕获的变量。该关键字放在参数列表之后（即使参数列表为空，也必须存在）。
```c++
int x = 1;

auto f1 = [&x] { x = 2; }; // OK: x 是引用，修改原始对象

auto f2 = [x] { x = 2; }; // ERROR: lambda 只能对捕获的值执行 const 操作
// 对比：
auto f3 = [x]() mutable { x = 2; }; // OK: lambda 可以对捕获的值执行任何操作
```

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
```c++
struct A {
  virtual void foo();
};

struct B : A {
  virtual void foo() final;
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

A a1 = f(A{}); // 从右值临时对象移动构造
A a2 = std::move(a1); // 使用 std::move 移动构造
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

转换函数现在可以使用 `explicit` 说明符来使其显式。
```c++
struct A {
  operator bool() const { return true; }
};

struct B {
  explicit operator bool() const { return true; }
};

A a;
if (a); // OK：调用 A::operator bool()
bool ba = a; // OK：复制初始化选择 A::operator bool()

B b;
if (b); // OK：调用 B::operator bool()
bool bb = b; // 错误：复制初始化不考虑 B::operator bool()
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

int version {Program::getVersion()};              // 使用 Version2 中的 getVersion()
int oldVersion {Program::Version1::getVersion()}; // 使用 Version1 中的 getVersion()
bool firstVersion {Program::isFirstVersion()};    // 添加 Version2 后无法编译
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

成员函数现在可以根据 `*this` 是左值还是右值引用进行限定。

```c++
struct Bar {
  // ...
};

struct Foo {
  Bar& getBar() & { return bar; }
  const Bar& getBar() const& { return bar; }
  Bar&& getBar() && { return std::move(bar); }
  const Bar&& getBar() const&& { return std::move(bar); }
private:
  Bar bar;
};

Foo foo{};
Bar bar = foo.getBar(); // 调用 `Bar& getBar() &`

const Foo foo2{};
Bar bar2 = foo2.getBar(); // 调用 `Bar& Foo::getBar() const&`

Foo{}.getBar(); // 调用 `Bar&& Foo::getBar() &&`
std::move(foo).getBar(); // 调用 `Bar&& Foo::getBar() &&`
std::move(foo2).getBar(); // 调用 `const Bar&& Foo::getBar() const&`
```

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

`noexcept` 说明符指定函数是否可能抛出异常。它是 `throw()` 的改进版本。

```c++
void func1() noexcept;        // 不抛出
void func2() noexcept(true);  // 不抛出
void func3() throw();         // 不抛出

void func4() noexcept(false); // 可能抛出
```

非抛异常函数允许调用可能抛出异常的函数。每当抛出异常并且搜索处理程序时遇到非抛异常函数的最外层块时，将调用 `std::terminate` 函数。

```c++
extern void f();  // 可能抛出
void g() noexcept {
    f();          // 有效，即使 f 抛出
    throw 42;     // 有效，实际上调用 std::terminate
}
```

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
std::unique_ptr<int> p1 {new int{0}};  // 实际上应该使用 std::make_unique
std::unique_ptr<int> p2 = p1; // 错误 -- 不能复制 unique_ptr
std::unique_ptr<int> p3 = std::move(p1); // 将 `p1` 移动到 `p3` 中
                                         // 现在取消引用 `p1` 持有的对象是不安全的
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