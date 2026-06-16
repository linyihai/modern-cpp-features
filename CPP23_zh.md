# C++23

## 概述

这些描述和示例取自各种资源（参见[致谢](#致谢)部分），并以我自己的话进行了总结。

C++23 包含以下新语言特性：
- [Consteval If(consteval if)](#consteval-if)
- [Deducing this(推导 `this`)](#deducing-this)
- [Multidimensional Subscript Operator(多维下标运算符)](#multidimensional-subscript-operator)
- [Increasing Range-Based For Safety(增强基于范围的 `for` 循环安全性)](#increasing-range-based-for-safety)

C++23 包含以下新标准库特性：
- [C++23](#c23)
  - [概述](#概述)
  - [C++23 语言特性](#c23-语言特性)
    - [Consteval If(consteval if)](#consteval-if)
    - [Deducing this(推导 `this`)](#推导-this)
    - [Multidimensional Subscript Operator(多维下标运算符)](#多维下标运算符)
    - [Increasing Range-Based For Safety(增强基于范围的 `for` 循环安全性)](#增强基于范围的-for-循环安全性)
  - [C++23 标准库特性](#c23-标准库特性)
    - [Stacktrace Library(Stacktrace 库)](#stacktrace-库)
    - [字符串和字符串视图的 `contains`](#字符串和字符串视图的-contains)
    - [`std::to_underlying`](#stdto_underlying)
    - [`spanstream`](#spanstream)
    - [Input/Output Pointers(输入/输出指针)](#输入输出指针)
    - [Monadic Operations for std::optional(`std::optional` 的单子操作)](#stdoptional-的单子操作)
    - [`std::expected`](#stdexpected)
    - [`std::unreachable`](#stdunreachable)
  - [致谢](#致谢)
  - [作者](#作者)
  - [内容贡献者](#内容贡献者)
  - [许可证](#许可证)

## C++23 语言特性

### Consteval If(consteval if)

编写在常量求值期间实例化的代码。
```c++
consteval int f(int i) { return i; }

constexpr int g(int i) {
  if consteval {
      return f(i);
  } else {
      return 42;
  }
}
```

### Deducing this(推导 `this`)

使用 C++23 引入的显式对象成员函数，通过指定成员函数的第一个参数并添加 `this` 关键字前缀，可以推导对象的类型和值类别：
```c++
// 使用推导 this 的新方式：
struct T {
  decltype(auto) operator[](this auto& self, std::size_t idx) { 
    return self.mVector[idx]; 
  }
};

// 旧方式：
struct T {
  value_t& operator[](std::size_t idx) {
    return mVector[idx];
  }
  const value_t& operator[](std::size_t idx) const {
    return mVector[idx];
  }
};
```

### Multidimensional Subscript Operator(多维下标运算符)

为 `operator[]` 指定零个或多个参数：
```c++
template <typename T, std::size_t Z, std::size_t Y, std::size_t X>
struct Array3d {
  std::array<T, X * Y * Z> m{};

  T& operator[](std::size_t z, std::size_t y, std::size_t x) {
      return m[z * Y * X + y * X + x];
  }
};

Array3d<int, 4, 3, 2> v;
v[3, 2, 1] = 42;
```

### Increasing Range-Based For Safety(增强基于范围的 `for` 循环安全性)

修复了 C++ 中最重要的控制结构之一的一些臭名昭著的生命周期问题。

以下是 C++23 之前存在问题但现已修复的代码片段示例：

* `for (auto e : getTmp().getRef())`
* `for (auto e : getVector()[0])`
* `for (auto valueElem : getMap()["key"])`
* `for (auto e : get<0>(getTuple()))`
* `for (auto e : getOptionalCollection().value())`
* `for (char c : get<std::string>(getVariant()))`

## C++23 标准库特性

### Stacktrace Library(Stacktrace 库)

堆栈跟踪是调用序列的近似表示，由堆栈跟踪条目组成。堆栈跟踪条目（由 `std::stacktrace_entry` 表示）包含源文件和行号等信息，以及一个描述字段。

Linux 系统上的示例输出：
```c++
#include <print>
#include <stacktrace>

int main() {
    std::println("{}", std::stacktrace::current());
}
```
```
  0#  main at /app/example.cpp:5 [0x5ee42e3db747]
  1#  <unknown> [0x76e76dc29d8f]
  2#  __libc_start_main [0x76e76dc29e3f]
  3#  _start [0x5ee42e3db644]
```

### 字符串和字符串视图的 `contains`

一个更简单的函数，用于查询字符串或字符串视图中是否包含子字符串：
```c++
std::string{"foobarbaz"}.contains("bar"); // == true
std::string{"foobarbaz"}.contains("bat"); // == false
```

### `std::to_underlying`

支持将枚举转换为其底层类型的常用工具：
```c++
enum class MyEnum : int { A = 1, B, C };
std::to_underlying(MyEnum::A); // == 1
std::to_underlying(MyEnum::C); // == 3
```

### `spanstream`

`strstream` 的替代品，使用字符 span 作为外部提供的缓冲区。不拥有缓冲区也不进行重新分配。
```c++
char input[] = "10 20 30";
std::ispanstream is{std::span<char>{input}};
int i;
is >> i; // i == 10
is >> i; // i == 20
is >> i; // i == 30
```
```c++
char output[30]{}; // 零初始化数组
std::ospanstream os{std::span<char>{output}};
os << 10 << 20 << 30;
std::span<char> sp = os.span();
```

### Input/Output Pointers(输入/输出指针)

`std::out_ptr` 和 `std::inout_ptr` 是支持 C API 和智能指针的抽象，通过创建临时的指向指针的指针，在析构时更新智能指针。简而言之：它是一个可转换为 `T**` 的对象，在超出作用域时更新（通过 `reset` 调用或语义等效的行为）创建它时使用的智能指针。

这个抽象还可以在抛出异常时安全地管理相关内存的生命周期。
```c++
// p_handle 被写入（输出）。
int c_api_create_handle(MyHandle** p_handle);
// p_handle 既被读取（输入）又被写入（输出）。
int c_api_recreate_handle(MyHandle** p_handle);
void c_api_delete_handle(MyHandle* handle);

struct resource_deleter {
	void operator()(MyHandle* handle) {
		c_api_delete_handle(handle);
	}
};
```
```c++
std::unique_ptr<MyHandle, resource_deleter> resource(nullptr);
int err = c_api_create_handle(std::out_ptr(resource));
// `resource` 现在拥有 `c_api_create_handle` 内部分配的内存。
```
```c++
std::shared_ptr<MyHandle> resource(nullptr);
int err = c_api_recreate_handle(std::inout_ptr(resource), resource_deleter{});
// `resource` 现在共享 `c_api_recreate_handle` 内部分配的内存。
```

输入/输出指针都支持隐式转换为 `void**`，以及显式转换为用户指定的类型。

### Monadic Operations for std::optional(`std::optional` 的单子操作)

支持 `std::optional` 的各种 `and_then`、`transform` 和 `or_else` 操作。
```c++
std::optional<int> parse_int(const std::string&);
std::optional<int> ensure_non_negative(int);
std::optional<double> default_value_or_empty(double);

std::optional<double> stringToSqrtDouble(const std::string& input) {
  return parse_int(input)
    .and_then(ensure_non_negative)
    .transform([](int x) {
      return std::sqrt(x);
    })
    .or_else(default_value_or_empty);
}
```

### `std::expected`

`std::expected` 提供了一种表示值和潜在错误值的方式，两者都包含在一个类型中。还支持对预期值和非预期（即错误）值的各种单子操作。

使用 `std::unexpected` 存储非预期（即错误）值。
```c++
enum class StringToSqrtDoubleError {
    ParseError, NegativeNumber
};

std::expected<int, StringToSqrtDoubleError> parse_int(const std::string&);

std::expected<double, StringToSqrtDoubleError> stringToSqrtDouble(const std::string& input) {
    auto parsed = parse_int(input);
    if (!parsed) return parsed;

    auto parsedInt = *parsed;
    if (parsedInt < 0) return std::unexpected(StringToSqrtDoubleError::NegativeNumber);

    return std::sqrt(parsedInt);
}
```

### `std::unreachable`

提供一种显式标记代码路径为不可达的方法。如果代码路径被到达，可能表现出未定义行为。
```c++
enum class MyEnum { A, B, C };

int convertMyEnumToInt(MyEnum e) {
    switch (e) {
        case MyEnum::A: return 0;
        case MyEnum::B: return 1;
        case MyEnum::C: return 2;
        default: std::unreachable(); 
    }
}
```

## 致谢
* [cppreference](http://en.cppreference.com/w/cpp) - 特别有助于查找新标准库特性的示例和文档。
* [C++ Rvalue References Explained](http://web.archive.org/web/20240324121501/http://thbecker.net/articles/rvalue_references/section_01.html) - 一篇很棒的介绍文章，我用它来理解右值引用、完美转发和移动语义。
* [clang](http://clang.llvm.org/cxx_status.html) 和 [gcc](https://gcc.gnu.org/projects/cxx-status.html) 的标准支持页面。这里还包含了语言/库特性的提案，我用它们来帮助查找描述、要解决的问题以及一些示例。
* [Compiler explorer](https://godbolt.org/)
* [Scott Meyers' Effective Modern C++](https://www.amazon.com/Effective-Modern-Specific-Ways-Improve/dp/1491903996) - 强烈推荐的书！
* [Jason Turner's C++ Weekly](https://www.youtube.com/channel/UCxHAlbZQNFU2LgEtiqd2Maw) - 很好的 C++ 相关视频合集。
* [What can I do with a moved-from object?](http://stackoverflow.com/questions/7027523/what-can-i-do-with-a-moved-from-object)
* [What are some uses of decltype(auto)?](http://stackoverflow.com/questions/24109737/what-are-some-uses-of-decltypeauto)
* 还有许多我忘记的 SO 帖子...

## 作者
Anthony Calandra

## 内容贡献者
参见：https://github.com/AnthonyCalandra/modern-cpp-features/graphs/contributors

## 许可证
MIT