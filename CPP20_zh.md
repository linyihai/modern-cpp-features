# C++20

## 概述

以下描述和示例来自各种资源（见[致谢](#致谢)部分），并以我自己的话进行了总结。

C++20 包含以下新语言特性：
- [C++20](#c20)
  - [概述](#概述)
  - [C++20 语言特性](#c20-语言特性)
    - [Coroutine(协程)](#coroutine协程)
    - [Concept(概念)](#concept概念)
    - [Three-Way Comparison(三路比较)](#three-way-comparison三路比较)
    - [Designated Initializer(指定初始化器)](#designated-initializer指定初始化器)
    - [Template Syntax for Lambdas(Lambda 的模板语法)](#template-syntax-for-lambdaslambda-的模板语法)
    - [Range-Based For Loop with Initializer(带初始化器的范围 for 循环)](#range-based-for-loop-with-initializer带初始化器的范围-for-循环)
    - [\[\[likely\]\] and \[\[unlikely\]\] Attributes(\[\[likely\]\] 和 \[\[unlikely\]\] 属性)](#likely-and-unlikely-attributeslikely-和-unlikely-属性)
    - [Deprecate Implicit Capture of this(弃用隐式捕获 this)](#deprecate-implicit-capture-of-this弃用隐式捕获-this)
    - [Class Types in Non-Type Template Parameters(非类型模板参数中的类类型)](#class-types-in-non-type-template-parameters非类型模板参数中的类类型)
    - [Constexpr Virtual Function(constexpr 虚函数)](#constexpr-virtual-functionconstexpr-虚函数)
    - [explicit(bool)](#explicitbool)
    - [Immediate Function(立即函数)](#immediate-function立即函数)
    - [using enum](#using-enum)
    - [Lambda Capture of Parameter Pack(Lambda 捕获参数包)](#lambda-capture-of-parameter-packlambda-捕获参数包)
    - [char8\_t](#char8_t)
    - [constinit](#constinit)
    - [\_\_VA\_OPT\_\_](#__va_opt__)
  - [C++20 标准库特性](#c20-标准库特性)
    - [Text Formatting(文本格式化)](#text-formatting文本格式化)
    - [Concept(概念)库](#concept概念库)
    - [Synchronized Buffered Output Stream(同步缓冲输出流)](#synchronized-buffered-output-stream同步缓冲输出流)
    - [std::span](#stdspan)
    - [Bit Operation(位操作)](#bit-operation位操作)
    - [Math Constants(数学常数)](#math-constants数学常数)
    - [std::is\_constant\_evaluated](#stdis_constant_evaluated)
    - [std::make\_shared Supports Arrays(std::make\_shared 支持数组)](#stdmake_shared-supports-arraysstdmake_shared-支持数组)
    - [Starts\_with and Ends\_with on Strings(字符串的 starts\_with 和 ends\_with)](#starts_with-and-ends_with-on-strings字符串的-starts_with-和-ends_with)
    - [Check if Associative Container Has Element(检查关联容器是否包含元素)](#check-if-associative-container-has-element检查关联容器是否包含元素)
    - [std::bit\_cast](#stdbit_cast)
    - [std::midpoint](#stdmidpoint)
    - [std::to\_array](#stdto_array)
    - [std::bind\_front](#stdbind_front)
    - [Uniform Container Erasure(统一容器擦除)](#uniform-container-erasure统一容器擦除)
    - [Three-Way Comparison(三路比较)辅助函数](#three-way-comparison三路比较辅助函数)
    - [std::lexicographical\_compare\_three\_way](#stdlexicographical_compare_three_way)
    - [std::jthread](#stdjthread)
    - [Safe Integral Comparison(安全整数比较)](#safe-integral-comparison安全整数比较)
  - [致谢](#致谢)
  - [作者](#作者)
  - [内容贡献者](#内容贡献者)
  - [许可证](#许可证)

C++20 包含以下新标准库特性：
- [C++20](#c20)
  - [概述](#概述)
  - [C++20 语言特性](#c20-语言特性)
    - [Coroutine(协程)](#coroutine协程)
    - [Concept(概念)](#concept概念)
    - [Three-Way Comparison(三路比较)](#three-way-comparison三路比较)
    - [Designated Initializer(指定初始化器)](#designated-initializer指定初始化器)
    - [Template Syntax for Lambdas(Lambda 的模板语法)](#template-syntax-for-lambdaslambda-的模板语法)
    - [Range-Based For Loop with Initializer(带初始化器的范围 for 循环)](#range-based-for-loop-with-initializer带初始化器的范围-for-循环)
    - [\[\[likely\]\] and \[\[unlikely\]\] Attributes(\[\[likely\]\] 和 \[\[unlikely\]\] 属性)](#likely-and-unlikely-attributeslikely-和-unlikely-属性)
    - [Deprecate Implicit Capture of this(弃用隐式捕获 this)](#deprecate-implicit-capture-of-this弃用隐式捕获-this)
    - [Class Types in Non-Type Template Parameters(非类型模板参数中的类类型)](#class-types-in-non-type-template-parameters非类型模板参数中的类类型)
    - [Constexpr Virtual Function(constexpr 虚函数)](#constexpr-virtual-functionconstexpr-虚函数)
    - [explicit(bool)](#explicitbool)
    - [Immediate Function(立即函数)](#immediate-function立即函数)
    - [using enum](#using-enum)
    - [Lambda Capture of Parameter Pack(Lambda 捕获参数包)](#lambda-capture-of-parameter-packlambda-捕获参数包)
    - [char8\_t](#char8_t)
    - [constinit](#constinit)
    - [\_\_VA\_OPT\_\_](#__va_opt__)
  - [C++20 标准库特性](#c20-标准库特性)
    - [Text Formatting(文本格式化)](#text-formatting文本格式化)
    - [Concept(概念)库](#concept概念库)
    - [Synchronized Buffered Output Stream(同步缓冲输出流)](#synchronized-buffered-output-stream同步缓冲输出流)
    - [std::span](#stdspan)
    - [Bit Operation(位操作)](#bit-operation位操作)
    - [Math Constants(数学常数)](#math-constants数学常数)
    - [std::is\_constant\_evaluated](#stdis_constant_evaluated)
    - [std::make\_shared Supports Arrays(std::make\_shared 支持数组)](#stdmake_shared-supports-arraysstdmake_shared-支持数组)
    - [Starts\_with and Ends\_with on Strings(字符串的 starts\_with 和 ends\_with)](#starts_with-and-ends_with-on-strings字符串的-starts_with-和-ends_with)
    - [Check if Associative Container Has Element(检查关联容器是否包含元素)](#check-if-associative-container-has-element检查关联容器是否包含元素)
    - [std::bit\_cast](#stdbit_cast)
    - [std::midpoint](#stdmidpoint)
    - [std::to\_array](#stdto_array)
    - [std::bind\_front](#stdbind_front)
    - [Uniform Container Erasure(统一容器擦除)](#uniform-container-erasure统一容器擦除)
    - [Three-Way Comparison(三路比较)辅助函数](#three-way-comparison三路比较辅助函数)
    - [std::lexicographical\_compare\_three\_way](#stdlexicographical_compare_three_way)
    - [std::jthread](#stdjthread)
    - [Safe Integral Comparison(安全整数比较)](#safe-integral-comparison安全整数比较)
  - [致谢](#致谢)
  - [作者](#作者)
  - [内容贡献者](#内容贡献者)
  - [许可证](#许可证)


## C++20 语言特性

### Coroutine(协程)

**注意：** 虽然这些示例说明了如何在基本层面上使用协程，但编译代码时还有很多其他事情发生。这些示例并不意味着完整覆盖 C++20 的协程。由于标准库尚未提供 `generator` 和 `task` 类，我使用了 cppcoro 库来编译这些示例。

***Coroutine(协程)***是可以暂停和恢复执行的特殊函数。要定义协程，函数体中必须存在 `co_return`、`co_await` 或 `co_yield` 关键字。C++20 的协程是无栈的；除非编译器优化掉，否则它们的状态分配在堆上。

协程的一个示例是**生成器**函数，它在每次调用时产生（即生成）一个值：
```c++
generator<int> range(int start, int end) {
  while (start < end) {
    co_yield start;
    start++;
  }
}

for (int n : range(0, 10)) {
  std::cout << n << std::endl;
}
```

协程的另一个示例是**任务**，它是在等待任务时执行的异步计算：
```c++
task<void> echo(socket s) {
  for (;;) {
    auto data = co_await s.async_read();
    co_await async_write(s, data);
  }
}
```

使用任务延迟计算值：
```c++
task<int> calculate_meaning_of_life() {
  co_return 42;
}

auto meaning_of_life = calculate_meaning_of_life();
co_await meaning_of_life; // == 42
```

### Concept(概念)

***Concept(概念)***是命名的编译时谓词，用于约束类型。它们采用以下形式：
```
template < template-parameter-list >
concept concept-name = constraint-expression;
```

其中 `constraint-expression` 求值为 constexpr 布尔值。**约束**应该建模语义要求，例如类型是否为数值或可哈希的。如果给定类型不满足它所绑定的概念（即 `constraint-expression` 返回 `false`），则会产生编译错误。
```c++
template <typename T>
concept always_satisfied = true;
template <typename T>
concept integral = std::is_integral_v<T>;
template <typename T>
concept signed_integral = integral<T> && std::is_signed_v<T>;
template <typename T>
concept unsigned_integral = integral<T> && !signed_integral<T>;
```

### Three-Way Comparison(三路比较)

C++20 引入了太空船运算符（`<=>`）作为编写比较函数的新方式，减少样板代码并帮助开发者定义更清晰的比较语义。定义三路比较运算符将自动生成其他比较运算符函数（即 `==`、`!=`、`<` 等）。

引入了三种排序：
* `std::strong_ordering`：强排序区分相等的项目（相同且可互换）。
* `std::weak_ordering`：弱排序区分等价的项目（不相同，但在比较目的上可以互换）。
* `std::partial_ordering`：偏序遵循弱排序的相同原则，但包括无法排序的情况。

默认的三路比较运算符执行成员比较：
```c++
struct foo {
  int a;
  bool b;
  char c;
  friend auto operator<=>(const foo&) const = default;
};
```

### Designated Initializer(指定初始化器)

C 风格的指定初始化器语法。在指定初始化器列表中未显式列出的任何成员字段都将被默认初始化。
```c++
struct A {
  int x;
  int y;
  int z = 123;
};
A a {.x = 1, .z = 2}; // a.x == 1, a.y == 0, a.z == 2
```

### Template Syntax for Lambdas(Lambda 的模板语法)

在 lambda 表达式中使用熟悉的模板语法。
```c++
auto f = []<typename T>(std::vector<T> v) {
};
```

### Range-Based For Loop with Initializer(带初始化器的范围 for 循环)

此特性简化常见代码模式，帮助保持作用域紧凑，并提供解决常见生命周期问题的优雅方案。
```c++
for (auto v = std::vector{1, 2, 3}; auto& e : v) {
  std::cout << e;
}
```

### [[likely]] and [[unlikely]] Attributes([[likely]] 和 [[unlikely]] 属性)

向优化器提供提示，指示标记的语句被执行的概率很高。
```c++
switch (n) {
case 1:
  break;
[[likely]] case 2:
  break;
}
```

### Deprecate Implicit Capture of this(弃用隐式捕获 this)

在 lambda 捕获中使用 `[=]` 隐式捕获 `this` 现已弃用；建议使用 `[=, this]` 或 `[=, *this]` 显式捕获。
```c++
struct int_value {
  int n = 0;
  auto getter_fn() {
    return [=, *this]() { return n; };
  }
};
```

### Class Types in Non-Type Template Parameters(非类型模板参数中的类类型)

类现在可以用于非类型模板参数。作为模板参数传递的对象具有类型 `const T`，并具有静态存储持续时间。
```c++
struct foo {
  foo() = default;
  constexpr foo(int) {}
};

template <foo f = {}>
auto get_foo() {
  return f;
}

get_foo();
get_foo<foo{123}>();
```

### Constexpr Virtual Function(constexpr 虚函数)

虚函数现在可以是 `constexpr` 并在编译时求值。
```c++
struct X1 {
  virtual int f() const = 0;
};

struct X2: public X1 {
  constexpr virtual int f() const { return 2; }
};

constexpr X2 x2;
x2.f(); // == 2
```

### explicit(bool)

在编译时有条件地选择构造函数是否显式。`explicit(true)` 等同于指定 `explicit`。
```c++
struct foo {
  template <typename T>
  explicit(!std::is_integral_v<T>) foo(T) {}
};

foo a = 123; // OK
foo b = "123"; // ERROR
foo c {"123"}; // OK
```

### Immediate Function(立即函数)

类似于 `constexpr` 函数，但带有 `consteval` 说明符的函数必须产生常量。这些称为***Immediate Function(立即函数)***。
```c++
consteval int sqr(int n) {
  return n * n;
}

constexpr int r = sqr(100); // OK
int x = 100;
int r2 = sqr(x); // ERROR
```

### using enum

将枚举的成员引入作用域以提高可读性。
```c++
enum class rgba_color_channel { red, green, blue, alpha };

std::string_view to_string(rgba_color_channel my_channel) {
  switch (my_channel) {
    using enum rgba_color_channel;
    case red:   return "red";
    case green: return "green";
    case blue:  return "blue";
    case alpha: return "alpha";
  }
}
```

### Lambda Capture of Parameter Pack(Lambda 捕获参数包)

按值捕获参数包：
```c++
template <typename... Args>
auto f(Args&&... args){
    return [...args = std::forward<Args>(args)] {
    };
}
```

按引用捕获参数包：
```c++
template <typename... Args>
auto f(Args&&... args){
    return [&...args = std::forward<Args>(args)] {
    };
}
```

### char8_t

提供用于表示 UTF-8 字符串的标准类型。
```c++
char8_t utf8_str[] = u8"\u0123";
```

### constinit

`constinit` 说明符要求变量必须在编译时初始化。
```c++
const char* g() { return "dynamic initialization"; }
constexpr const char* f() { return "constant initializer"; }

constinit const char* c = f();  // OK
constinit const char* d = g();  // ERROR
```

### \_\_VA\_OPT\_\_

帮助支持变参宏，通过在变参宏非空时求值给定参数。
```c++
#define F(...) f(0 __VA_OPT__(,) __VA_ARGS__)
F(a, b, c) // 替换为 f(0, a, b, c)
F()        // 替换为 f(0)
```

## C++20 标准库特性

### Text Formatting(文本格式化)

使用 `std::format` 为标准库提供编译时检查的字符串格式化库。
```cpp
std::format("{}", 123); // OK -- 返回 "123"
std::format("{} {}", "Here's a number:", 123); // OK
```

### Concept(概念)库

标准库还提供概念用于构建更复杂的概念。

**核心语言概念：**
- `same_as` - 指定两种类型相同。
- `derived_from` - 指定一种类型派生自另一种类型。
- `integral` - 指定类型是整数类型。

**对象概念：**
- `movable` - 指定类型的对象可以移动和交换。
- `copyable` - 指定类型的对象可以复制、移动和交换。
- `regular` - 指定类型是**常规**的，即它既是 `semiregular` 又是 `equality_comparable`。

### Synchronized Buffered Output Stream(同步缓冲输出流)

缓冲包装输出流的输出操作，确保同步（即输出不交错）。
```c++
std::osyncstream{std::cout} << "The value of x is:" << x << std::endl;
```

### std::span

span 是容器的视图（即非拥有），提供对连续元素组的边界检查访问。
```c++
void print_ints(std::span<const int> ints) {
    for (const auto n : ints) {
        std::cout << n << std::endl;
    }
}

print_ints(std::vector{ 1, 2, 3 });
print_ints(std::array<int, 5>{ 1, 2, 3, 4, 5 });
```

### Bit Operation(位操作)

C++20 提供了一个新的 `<bit>` 头文件，提供一些位操作，包括 popcount。
```c++
std::popcount(0u); // 0
std::popcount(1u); // 1
std::popcount(0b1111'0000u); // 4
```

### Math Constants(数学常数)

数学常数，包括 PI、欧拉数等，定义在 `<numbers>` 头文件中。
```c++
std::numbers::pi; // 3.14159...
std::numbers::e; // 2.71828...
```

### std::is_constant_evaluated

在编译时上下文中调用时为真的谓词函数。
```c++
constexpr bool is_compile_time() {
    return std::is_constant_evaluated();
}

constexpr bool a = is_compile_time(); // true
bool b = is_compile_time(); // false
```

### std::make_shared Supports Arrays(std::make_shared 支持数组)

```c++
auto p = std::make_shared<int[]>(5);
auto p = std::make_shared<int[5]>();
```

### Starts_with and Ends_with on Strings(字符串的 starts_with 和 ends_with)

字符串（和字符串视图）现在有 `starts_with` 和 `ends_with` 成员函数来检查字符串是否以给定字符串开头或结尾。
```c++
std::string str = "foobar";
str.starts_with("foo"); // true
str.ends_with("baz"); // false
```

### Check if Associative Container Has Element(检查关联容器是否包含元素)

关联容器（如 set 和 map）具有 `contains` 成员函数。
```c++
std::map<int, char> map {{1, 'a'}, {2, 'b'}};
map.contains(2); // true
map.contains(123); // false
```

### std::bit_cast

一种更安全的方式将对象从一种类型重新解释为另一种类型。
```c++
float f = 123.0;
int i = std::bit_cast<int>(f);
```

### std::midpoint

安全地计算两个整数的中点（无溢出）。
```c++
std::midpoint(1, 3); // == 2
```

### std::to_array

将给定的数组/"类数组"对象转换为 `std::array`。
```c++
std::to_array("foo"); // 返回 `std::array<char, 4>`
std::to_array<int>({1, 2, 3}); // 返回 `std::array<int, 3>`
```

### std::bind_front

将前 N 个参数绑定到给定的自由函数、lambda 或成员函数。
```c++
const auto f = [](int a, int b, int c) { return a + b + c; };
const auto g = std::bind_front(f, 1, 1);
g(1); // == 3
```

### Uniform Container Erasure(统一容器擦除)

为各种 STL 容器提供 `std::erase` 和/或 `std::erase_if`。
```c++
std::vector v{0, 1, 0, 2, 0, 3};
std::erase(v, 0); // v == {1, 2, 3}
std::erase_if(v, [](int n) { return n == 0; }); // v == {1, 2, 3}
```

### Three-Way Comparison(三路比较)辅助函数

为比较结果命名的辅助函数：
```c++
std::is_eq(0 <=> 0); // == true
std::is_lteq(0 <=> 1); // == true
std::is_gt(0 <=> 1); // == false
```

### std::lexicographical_compare_three_way

使用三路比较字典比较两个范围。
```c++
std::vector a{0, 0, 0}, b{0, 0, 0}, c{1, 1, 1};
auto cmp_ab = std::lexicographical_compare_three_way(a.begin(), a.end(), b.begin(), b.end());
std::is_eq(cmp_ab); // == true
```

### std::jthread

执行线程（如 `std::thread`），在销毁时自动 join，并可以被信号停止。
```cpp
std::jthread t{
    [](std::stop_token stoken) {
        while (!stoken.stop_requested()) {
            std::this_thread::sleep_for(1s);
        }
    }
};
t.request_stop();
```

### Safe Integral Comparison(安全整数比较)

比较整数，包括不同类型的整数，没有整数转换的危险。
```cpp
-1 > 0U; // == true
std::cmp_greater(-1, 0U); // == false

std::cmp_equal(0U, 0); // == true
std::cmp_less_equal(-1, 1U); // == true
```

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