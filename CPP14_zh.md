# C++14

## 概述

以下描述和示例来自各种资源（见[致谢](#致谢)部分），并以我自己的话进行了总结。

C++14 包含以下新语言特性：
- [C++14](#c14)
  - [概述](#概述)
  - [C++14 语言特性](#c14-语言特性)
    - [Binary Literal(二进制字面量)](#二进制字面量)
    - [Generic Lambda Expression(泛型 Lambda 表达式)](#泛型-lambda-表达式)
    - [Lambda Capture Initializer(Lambda 捕获初始化器)](#lambda-捕获初始化器)
    - [Return Type Deduction(返回类型推导)](#返回类型推导)
    - [decltype(auto)](#decltypeauto)
    - [Relaxing Constraints on Constexpr Functions(放松 constexpr 函数的约束)](#放松-constexpr-函数的约束)
    - [Variable Template(变量模板)](#变量模板)
    - [[[deprecated]] Attribute([[deprecated]] 属性)](#deprecated-属性)
  - [C++14 标准库特性](#c14-标准库特性)
    - [标准库类型的用户自定义字面量](#标准库类型的用户自定义字面量)
    - [Compile-Time Integer Sequence(编译时整数序列)](#编译时整数序列)
    - [std::make\_unique](#stdmake_unique)
  - [致谢](#致谢)
  - [作者](#作者)
  - [内容贡献者](#内容贡献者)
  - [许可证](#许可证)

C++14 包含以下新标准库特性：
- [标准库类型的用户自定义字面量](#标准库类型的用户自定义字面量)
- [Compile-Time Integer Sequence(编译时整数序列)](#编译时整数序列)
- [std::make_unique](#stdmake_unique)

## C++14 语言特性

### Binary Literal(二进制字面量)

二进制字面量提供了一种方便的方式来表示二进制数。可以使用 `'` 分隔数字。
```c++
0b110 // == 6
0b1111'1111 // == 255
```

### Generic Lambda Expression(泛型 Lambda 表达式)

C++14 现在允许在参数列表中使用 `auto` 类型说明符，启用多态 lambda。
```c++
auto identity = [](auto x) { return x; };
int three = identity(3); // == 3
std::string foo = identity("foo"); // == "foo"
```

### Lambda Capture Initializer(Lambda 捕获初始化器)

这允许创建使用任意表达式初始化的 lambda 捕获。给捕获值的名称不需要与封闭作用域中的任何变量相关，并在 lambda 体中引入一个新名称。初始化表达式在 lambda 创建时求值（而不是在调用时）。
```c++
int factory(int i) { return i * 10; }
auto f = [x = factory(2)] { return x; }; // 返回 20

auto generator = [x = 0] () mutable {
  return x++;
};
auto a = generator(); // == 0
auto b = generator(); // == 1
auto c = generator(); // == 2
```

因为现在可以将值移动（或转发）到 lambda 中，而以前只能通过复制或引用捕获，所以我们现在可以按值捕获仅可移动类型的 lambda。
```c++
auto p = std::make_unique<int>(1);
auto task1 = [=] { *p = 5; }; // ERROR: std::unique_ptr 不能被复制
auto task2 = [p = std::move(p)] { *p = 5; }; // OK: p 被移动构造到闭包对象中
```

使用这个特性，引用捕获可以使用与被引用变量不同的名称。
```c++
auto x = 1;
auto f = [&r = x, x = x * 10] {
  ++r;
  return r + x;
};
f(); // 将 x 设置为 2 并返回 12
```

### Return Type Deduction(返回类型推导)

在 C++14 中使用 `auto` 返回类型，编译器会尝试为您推导类型。
```c++
auto f(int i) {
 return i;
}
```

对于 lambda，现在可以使用 `auto` 推导其返回类型：
```c++
template <typename T>
auto& f(T& t) {
  return t;
}
auto g = [](auto& x) -> auto& { return f(x); };
int y = 123;
int& z = g(y); // 对 `y` 的引用
```

### decltype(auto)

`decltype(auto)` 类型说明符也像 `auto` 一样推导类型。然而，它在推导返回类型时保留其引用和 cv 限定符，而 `auto` 则不会。
```c++
const int x = 0;
auto x1 = x; // int
decltype(auto) x2 = x; // const int
int y = 0;
int& y1 = y;
auto y2 = y1; // int
decltype(auto) y3 = y1; // int&
int&& z = 0;
auto z1 = std::move(z); // int
decltype(auto) z2 = std::move(z); // int&&
```

对泛型代码特别有用：
```c++
auto f(const int& i) {
 return i;
}

decltype(auto) g(const int& i) {
 return i;
}

int x = 123;
static_assert(std::is_same<int, decltype(f(x))>::value == 1);
static_assert(std::is_same<const int&, decltype(g(x))>::value == 1);
```

参见：[`decltype (C++11)`](README_zh.md#decltype)。

### Relaxing Constraints on Constexpr Functions(放松 constexpr 函数的约束)

在 C++11 中，`constexpr` 函数体只能包含非常有限的语法集。在 C++14 中，允许的语法集大大扩展，包括最常见的语法，如 `if` 语句、多个 `return`、循环等。
```c++
constexpr int factorial(int n) {
  if (n <= 1) {
    return 1;
  } else {
    return n * factorial(n - 1);
  }
}
factorial(5); // == 120
```

### Variable Template(变量模板)

C++14 允许变量被模板化：
```c++
template<class T>
constexpr T pi = T(3.1415926535897932385);
template<class T>
constexpr T e  = T(2.7182818284590452353);
```

### [[deprecated]] Attribute([[deprecated]] 属性)

C++14 引入了 `[[deprecated]]` 属性来表示某个单元不推荐使用，可能会产生编译警告。
```c++
[[deprecated]]
void old_method();
[[deprecated("Use new_method instead")]]
void legacy_method();
```

## C++14 标准库特性

### 标准库类型的用户自定义字面量

标准库类型的新用户自定义字面量，包括 `chrono` 和 `basic_string` 的新内置字面量。
```c++
using namespace std::chrono_literals;
auto day = 24h;
day.count(); // == 24
std::chrono::duration_cast<std::chrono::minutes>(day).count(); // == 1440
```

### Compile-Time Integer Sequence(编译时整数序列)

类模板 `std::integer_sequence` 表示编译时的整数序列。
```c++
template<typename Array, std::size_t... I>
decltype(auto) a2t_impl(const Array& a, std::integer_sequence<std::size_t, I...>) {
  return std::make_tuple(a[I]...);
}

template<typename T, std::size_t N, typename Indices = std::make_index_sequence<N>>
decltype(auto) a2t(const std::array<T, N>& a) {
  return a2t_impl(a, Indices());
}
```

### std::make_unique

`std::make_unique` 是创建 `std::unique_ptr` 实例的推荐方式，原因如下：
* 避免使用 `new` 运算符。
* 防止代码重复。
* 最重要的是，它提供异常安全。

```c++
foo(std::make_unique<T>(), function_that_throws(), std::make_unique<T>());
```

有关 `std::unique_ptr` 和 `std::shared_ptr` 的更多信息，请参见[智能指针 (C++11)](README_zh.md#智能指针)部分。

## 致谢
* [cppreference](http://en.cppreference.com/w/cpp)
* [C++ Rvalue References Explained](http://web.archive.org/web/20240324121501/http://thbecker.net/articles/rvalue_references/section_01.html)
* [clang](http://clang.llvm.org/cxx_status.html) 和 [gcc](https://gcc.gnu.org/projects/cxx-status.html) 的标准支持页面
* [Compiler explorer](https://godbolt.org/)
* [Scott Meyers' Effective Modern C++](https://www.amazon.com/Effective-Modern-Specific-Ways-Improve/dp/1491903996)
* [Jason Turner's C++ Weekly](https://www.youtube.com/channel/UCxHAlbZQNFU2LgEtiqd2Maw)

## 作者
Anthony Calandra

## 内容贡献者
参见：https://github.com/AnthonyCalandra/modern-cpp-features/graphs/contributors

## 许可证
MIT