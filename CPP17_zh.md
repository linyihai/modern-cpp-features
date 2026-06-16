# C++17

## 概述

以下描述和示例来自各种资源（见[致谢](#致谢)部分），并以我自己的话进行了总结。

C++17 包含以下新语言特性：
- [C++17](#c17)
  - [概述](#概述)
  - [C++17 语言特性](#c17-语言特性)
    - [Class Template Argument Deduction(类模板参数推导)](#类模板参数推导)
    - [Declaring Non-Type Template Parameters with Auto(使用 auto 声明非类型模板参数)](#使用-auto-声明非类型模板参数)
    - [Folding Expression(折叠表达式)](#折叠表达式)
    - [New Rules for Auto Deduction from Braced Init List(auto 从花括号初始化列表推导的新规则)](#auto-从花括号初始化列表推导的新规则)
    - [Constexpr Lambda(constexpr lambda)](#constexpr-lambda)
    - [Lambda Capture this by Value(Lambda 按值捕获 this)](#lambda-按值捕获-this)
    - [Inline Variable(内联变量)](#内联变量)
    - [Nested Namespace(嵌套命名空间)](#嵌套命名空间)
    - [Structured Binding(结构化绑定)](#结构化绑定)
    - [Selection Statements with Initializer(带初始化器的选择语句)](#带初始化器的选择语句)
    - [Constexpr If(constexpr if)](#constexpr-if)
    - [UTF-8 Character Literal(UTF-8 字符字面量)](#utf-8-字符字面量)
    - [Direct List Initialization of Enums(枚举的直接列表初始化)](#枚举的直接列表初始化)
    - [\[\[fallthrough\]\]、\[\[nodiscard\]\]、\[\[maybe\_unused\]\] 属性](#fallthroughnodiscardmaybe_unused-属性)
    - [\_\_has\_include](#__has_include)
  - [C++17 标准库特性](#c17-标准库特性)
    - [std::variant](#stdvariant)
    - [std::optional](#stdoptional)
    - [std::any](#stdany)
    - [std::string\_view](#stdstring_view)
    - [std::invoke](#stdinvoke)
    - [std::apply](#stdapply)
    - [std::filesystem](#stdfilesystem)
    - [std::byte](#stdbyte)
    - [Splicing for Maps and Sets(map 和 set 的拼接)](#map-和-set-的拼接)
    - [Parallel Algorithm(并行算法)](#并行算法)
    - [std::sample](#stdsample)
    - [std::clamp](#stdclamp)
    - [std::reduce](#stdreduce)
    - [Prefix Sum Algorithm(前缀和算法)](#前缀和算法)
    - [GCD and LCM(gcd 和 lcm)](#gcd-和-lcm)
    - [std::not\_fn](#stdnot_fn)
    - [String Conversion to/from Numbers(字符串与数字的转换)](#字符串与数字的转换)
    - [Rounding Functions for Chrono Durations and Timepoints(chrono 持续时间和时间点的舍入函数)](#chrono-持续时间和时间点的舍入函数)
  - [致谢](#致谢)
  - [作者](#作者)
  - [内容贡献者](#内容贡献者)
  - [许可证](#许可证)

C++17 包含以下新标准库特性：
- [C++17](#c17)
  - [概述](#概述)
  - [C++17 语言特性](#c17-语言特性)
    - [Class Template Argument Deduction(类模板参数推导)](#类模板参数推导)
    - [Declaring Non-Type Template Parameters with Auto(使用 auto 声明非类型模板参数)](#使用-auto-声明非类型模板参数)
    - [Folding Expression(折叠表达式)](#折叠表达式)
    - [New Rules for Auto Deduction from Braced Init List(auto 从花括号初始化列表推导的新规则)](#auto-从花括号初始化列表推导的新规则)
    - [Constexpr Lambda(constexpr lambda)](#constexpr-lambda)
    - [Lambda Capture this by Value(Lambda 按值捕获 this)](#lambda-按值捕获-this)
    - [Inline Variable(内联变量)](#内联变量)
    - [Nested Namespace(嵌套命名空间)](#嵌套命名空间)
    - [Structured Binding(结构化绑定)](#结构化绑定)
    - [Selection Statements with Initializer(带初始化器的选择语句)](#带初始化器的选择语句)
    - [Constexpr If(constexpr if)](#constexpr-if)
    - [UTF-8 Character Literal(UTF-8 字符字面量)](#utf-8-字符字面量)
    - [Direct List Initialization of Enums(枚举的直接列表初始化)](#枚举的直接列表初始化)
    - [\[\[fallthrough\]\]、\[\[nodiscard\]\]、\[\[maybe\_unused\]\] 属性](#fallthroughnodiscardmaybe_unused-属性)
    - [\_\_has\_include](#__has_include)
  - [C++17 标准库特性](#c17-标准库特性)
    - [std::variant](#stdvariant)
    - [std::optional](#stdoptional)
    - [std::any](#stdany)
    - [std::string\_view](#stdstring_view)
    - [std::invoke](#stdinvoke)
    - [std::apply](#stdapply)
    - [std::filesystem](#stdfilesystem)
    - [std::byte](#stdbyte)
    - [Splicing for Maps and Sets(map 和 set 的拼接)](#map-和-set-的拼接)
    - [Parallel Algorithm(并行算法)](#并行算法)
    - [std::sample](#stdsample)
    - [std::clamp](#stdclamp)
    - [std::reduce](#stdreduce)
    - [Prefix Sum Algorithm(前缀和算法)](#前缀和算法)
    - [GCD and LCM(gcd 和 lcm)](#gcd-和-lcm)
    - [std::not\_fn](#stdnot_fn)
    - [String Conversion to/from Numbers(字符串与数字的转换)](#字符串与数字的转换)
    - [Rounding Functions for Chrono Durations and Timepoints(chrono 持续时间和时间点的舍入函数)](#chrono-持续时间和时间点的舍入函数)
  - [致谢](#致谢)
  - [作者](#作者)
  - [内容贡献者](#内容贡献者)
  - [许可证](#许可证)

## C++17 语言特性

### Class Template Argument Deduction(类模板参数推导)

类似于函数的自动模板参数推导，但现在包括类构造函数。
```c++
template <typename T = float>
struct MyContainer {
  T val;
  MyContainer() : val{} {}
  MyContainer(T val) : val{val} {}
};
MyContainer c1 {1}; // OK MyContainer<int>
MyContainer c2; // OK MyContainer<float>
```

### Declaring Non-Type Template Parameters with Auto(使用 auto 声明非类型模板参数)

遵循 `auto` 的推导规则，同时尊重非类型模板参数列表的允许类型[\*]，可以从参数的类型推导模板参数：
```c++
template <auto... seq>
struct my_integer_sequence {
};

auto seq = std::integer_sequence<int, 0, 1, 2>();
auto seq2 = my_integer_sequence<0, 1, 2>();
```

### Folding Expression(折叠表达式)

折叠表达式对二元运算符上的模板参数包执行折叠。
* 形式为 `(... op e)` 或 `(e op ...)` 的表达式称为**一元折叠**。
* 形式为 `(e1 op ... op e2)` 的表达式称为**二元折叠**。`e1` 或 `e2` 是未展开的参数包，但不能两者都是。

```c++
template <typename... Args>
bool logicalAnd(Args... args) {
    return (true && ... && args);
}
logicalAnd(true, true, true); // == true
```

```c++
template <typename... Args>
auto sum(Args... args) {
    return (... + args);
}
sum(1.0, 2.0f, 3); // == 6.0
```

### New Rules for Auto Deduction from Braced Init List(auto 从花括号初始化列表推导的新规则)

`auto` 推导在使用统一初始化语法时的变化。以前，`auto x {3};` 推导为 `std::initializer_list<int>`，现在推导为 `int`。
```c++
auto x1 {1, 2, 3}; // 错误：不是单个元素
auto x2 = {1, 2, 3}; // x2 是 std::initializer_list<int>
auto x3 {3}; // x3 是 int
auto x4 {3.0}; // x4 是 double
```

### Constexpr Lambda(constexpr lambda)

使用 `constexpr` 的编译时 lambda。
```c++
auto identity = [](int n) constexpr { return n; };
static_assert(identity(123) == 123);
```

```c++
constexpr auto add = [](int x, int y) {
  auto L = [=] { return x; };
  auto R = [=] { return y; };
  return [=] { return L() + R(); };
};
static_assert(add(1, 2)() == 3);
```

### Lambda Capture this by Value(Lambda 按值捕获 this)

在 lambda 的环境中捕获 `this` 以前只能通过引用。一个问题示例是使用回调的异步代码，这些回调需要对象可用，但对象可能已经超出其生命周期。`*this`（C++17）现在会创建当前对象的副本，而 `this`（C++11）继续按引用捕获。
```c++
struct MyObj {
  int value {123};
  auto getValueCopy() {
    return [*this] { return value; };
  }
  auto getValueRef() {
    return [this] { return value; };
  }
};
MyObj mo;
auto valueCopy = mo.getValueCopy();
auto valueRef = mo.getValueRef();
mo.value = 321;
valueCopy(); // 123
valueRef(); // 321
```

### Inline Variable(内联变量)

inline 说明符也可以应用于变量以及函数。声明为 inline 的变量具有与声明为 inline 的函数相同的语义。
```c++
struct S { int x; };
inline S x1 = S{321};
S x2 = S{123};
```

它也可以用于声明和定义静态成员变量，这样就不需要在源文件中初始化。
```c++
struct S {
  S() : id{count++} {}
  ~S() { count--; }
  int id;
  static inline int count{0};
};
```

### Nested Namespace(嵌套命名空间)

使用命名空间解析运算符创建嵌套命名空间定义。
```c++
namespace A::B::C {
  int i;
}
```

### Structured Binding(结构化绑定)

一种解构初始化的提议，允许编写 `auto [ x, y, z ] = expr;`，其中 `expr` 的类型是元组式对象，其元素将绑定到变量 `x`、`y` 和 `z`。

```c++
using Coordinate = std::pair<int, int>;
Coordinate origin() {
  return Coordinate{0, 0};
}
const auto [ x, y ] = origin();
x; // == 0
y; // == 0
```

```c++
std::unordered_map<std::string, int> mapping {
  {"a", 1}, {"b", 2}, {"c", 3}
};
for (const auto& [key, value] : mapping) {
}
```

### Selection Statements with Initializer(带初始化器的选择语句)

`if` 和 `switch` 语句的新版本，简化常见代码模式并帮助用户保持作用域紧凑。
```c++
if (std::lock_guard<std::mutex> lk(mx); v.empty()) {
  v.push_back(val);
}
```

```c++
switch (Foo gadget(args); auto s = gadget.status()) {
  case OK: gadget.zip(); break;
  case Bad: throw BadFoo(s.message());
}
```

### Constexpr If(constexpr if)

编写根据编译时条件实例化的代码。
```c++
template <typename T>
constexpr bool isIntegral() {
  if constexpr (std::is_integral<T>::value) {
    return true;
  } else {
    return false;
  }
}
static_assert(isIntegral<int>() == true);
static_assert(isIntegral<double>() == false);
```

### UTF-8 Character Literal(UTF-8 字符字面量)

以 `u8` 开头的字符字面量是类型为 `char` 的字符字面量。UTF-8 字符字面量的值等于其 ISO 10646 码点值。
```c++
char x = u8'x';
```

### Direct List Initialization of Enums(枚举的直接列表初始化)

枚举现在可以使用花括号语法初始化。
```c++
enum byte : unsigned char {};
byte b {0}; // OK
byte c {-1}; // ERROR
byte d = byte{1}; // OK
byte e = byte{256}; // ERROR
```

### [[fallthrough]], [[nodiscard]], [[maybe_unused]] Attributes([[fallthrough]]、[[nodiscard]]、[[maybe_unused]] 属性)

C++17 引入了三个新属性：`[[fallthrough]]`、`[[nodiscard]]` 和 `[[maybe_unused]]`。

* `[[fallthrough]]` 向编译器指示在 switch 语句中落 fallthrough 是预期行为。
```c++
switch (n) {
  case 1: 
    [[fallthrough]];
  case 2:
    break;
}
```

* `[[nodiscard]]` 在函数或类具有此属性且其返回值被丢弃时发出警告。
```c++
[[nodiscard]] bool do_something() {
  return is_success;
}
do_something(); // 警告
```

* `[[maybe_unused]]` 向编译器指示变量或参数可能未使用且是有意的。
```c++
void my_callback(std::string msg, [[maybe_unused]] bool error) {
  log(msg);
}
```

### \_\_has\_include

`__has_include (operand)` 运算符可用于 `#if` 和 `#elif` 表达式中，以检查头文件或源文件是否可用。
```c++
#ifdef __has_include
#  if __has_include(<optional>)
#    include <optional>
#  elif __has_include(<experimental/optional>)
#    include <experimental/optional>
#  else
#    define have_optional 0
#  endif
#endif
```

## C++17 标准库特性

### std::variant

类模板 `std::variant` 表示类型安全的 `union`。
```c++
std::variant<int, double> v{ 12 };
std::get<int>(v); // == 12
v = 12.0;
std::get<double>(v); // == 12.0
```

### std::optional

类模板 `std::optional` 管理一个可选的包含值。
```c++
std::optional<std::string> create(bool b) {
  if (b) {
    return "Godzilla";
  } else {
    return {};
  }
}
create(false).value_or("empty"); // == "empty"
create(true).value(); // == "Godzilla"
```

### std::any

用于存储任意类型单值的类型安全容器。
```c++
std::any x {5};
x.has_value() // == true
std::any_cast<int>(x) // == 5
std::any_cast<int&>(x) = 10;
std::any_cast<int>(x) // == 10
```

### std::string_view

对字符串的非拥有引用。
```c++
std::string_view cppstr {"foo"};
std::string str {"   trim me"};
std::string_view v {str};
v.remove_prefix(std::min(v.find_first_not_of(" "), v.size()));
v; // == "trim me"
```

### std::invoke

调用 `Callable` 对象并传递参数。
```c++
const auto add = [](int x, int y) { return x + y; };
Proxy p{ add };
p(1, 2); // == 3
```

### std::apply

使用元组参数调用 `Callable` 对象。
```c++
auto add = [](int x, int y) {
  return x + y;
};
std::apply(add, std::make_tuple(1, 2)); // == 3
```

### std::filesystem

新的 `std::filesystem` 库提供了一种标准方式来操作文件、目录和文件系统中的路径。
```c++
const auto bigFilePath {"bigFileToCopy"};
if (std::filesystem::exists(bigFilePath)) {
  const auto bigFileSize {std::filesystem::file_size(bigFilePath)};
  std::filesystem::path tmpPath {"/tmp"};
  if (std::filesystem::space(tmpPath).available > bigFileSize) {
    std::filesystem::create_directory(tmpPath.append("example"));
    std::filesystem::copy_file(bigFilePath, tmpPath.append("newFile"));
  }
}
```

### std::byte

新的 `std::byte` 类型提供了一种标准方式来表示字节数据。
```c++
std::byte a {0};
std::byte b {0xFF};
int i = std::to_integer<int>(b); // 0xFF
std::byte c = a & b;
int j = std::to_integer<int>(c); // 0
```

### Splicing for Maps and Sets(map 和 set 的拼接)

移动节点并合并容器，无需昂贵的复制、移动或堆分配/释放。
```c++
std::map<int, string> src {{1, "one"}, {2, "two"}};
std::map<int, string> dst {{3, "three"}};
dst.insert(src.extract(src.find(1)));
```

### Parallel Algorithm(并行算法)

许多 STL 算法开始支持**并行执行策略**：`seq`、`par` 和 `par_unseq`。
```c++
std::vector<int> longVector;
auto result1 = std::find(std::execution::par, longVector.begin(), longVector.end(), 2);
auto result2 = std::sort(std::execution::seq, longVector.begin(), longVector.end());
```

### std::sample

从给定序列中采样 n 个元素（不重复），每个元素被选中的概率相等。
```c++
const std::string ALLOWED_CHARS = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789";
std::string guid;
std::sample(ALLOWED_CHARS.begin(), ALLOWED_CHARS.end(), std::back_inserter(guid),
  5, std::mt19937{ std::random_device{}() });
```

### std::clamp

将给定值限制在下限和上限之间。
```c++
std::clamp(42, -1, 1); // == 1
std::clamp(-42, -1, 1); // == -1
std::clamp(0, -1, 1); // == 0
```

### std::reduce

对给定范围的元素进行折叠。概念上类似于 `std::accumulate`，但 `std::reduce` 将并行执行折叠。
```c++
const std::array<int, 3> a{ 1, 2, 3 };
std::reduce(a.begin(), a.end()); // == 6
std::reduce(a.begin(), a.end(), 1, std::multiplies<>{}); // == 6
```

### Prefix Sum Algorithm(前缀和算法)

支持前缀和（包括包含和排除扫描）以及变换。
```c++
const std::array<int, 3> a{ 1, 2, 3 };
std::inclusive_scan(a.begin(), a.end(), std::ostream_iterator<int>{ std::cout, " " }, std::plus<>{}); // 1 3 6
std::exclusive_scan(a.begin(), a.end(), std::ostream_iterator<int>{ std::cout, " " }, 0, std::plus<>{}); // 0 1 3
```

### GCD and LCM(gcd 和 lcm)

最大公约数（GCD）和最小公倍数（LCM）。
```c++
const int p = 9;
const int q = 3;
std::gcd(p, q); // == 3
std::lcm(p, q); // == 9
```

### std::not_fn

返回给定函数结果的否定的实用函数。
```c++
const auto is_even = [](const auto n) { return n % 2 == 0; };
std::vector<int> v{ 0, 1, 2, 3, 4 };
std::copy_if(v.begin(), v.end(), ostream_it, is_even); // 0 2 4
std::copy_if(v.begin(), v.end(), ostream_it, std::not_fn(is_even)); // 1 3
```

### String Conversion to/from Numbers(字符串与数字的转换)

将整数和浮点数转换为字符串或反之。
```c++
const int n = 123;
std::string str;
str.resize(3);
const auto [ ptr, ec ] = std::to_chars(str.data(), str.data() + str.size(), n);
```

### Rounding Functions for Chrono Durations and Timepoints(chrono 持续时间和时间点的舍入函数)

为 `std::chrono::duration` 和 `std::chrono::time_point` 提供 abs、round、ceil 和 floor 辅助函数。
```c++
std::chrono::milliseconds a{ -5500 };
std::chrono::milliseconds d = std::chrono::abs(a); // == 5500ms
std::chrono::round<seconds>(d); // == 6s
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