# C++ 知识点笔记

## 1. Final Specifier (final 说明符)

### 作用
`final` 说明符用于：
1. 阻止虚函数在派生类中被覆盖
2. 阻止类被继承

### 示例
```c++
struct A {
  virtual void foo();
};

struct B : A {
  virtual void foo() final;  // 阻止进一步覆盖
};

struct C : B {
  virtual void foo(); // Error -- 'foo' declaration overrides a 'final' function
};

struct A final {};      // 阻止类被继承
struct B : A {};        // Error -- base class 'A' is marked 'final'
```

### 要点
- 用于虚函数时，防止派生类覆盖该函数
- 用于类时，防止该类被继承
- 是编译时检查，违反会导致编译错误

---

## 2. Deleted Function (删除函数)

### 作用
使用 `= delete` 显式删除函数实现，禁止特定操作。

### 示例
```c++
class A {
  int x;

public:
  A(int x) : x{x} {};
  A(const A&) = delete;             // 删除复制构造函数
  A& operator=(const A&) = delete;  // 删除复制赋值运算符
};

A x {123};
A y = x; // Error -- calls deleted copy constructor
y = x;   // Error -- operator= is deleted
```

### 与 C++11 前的方法对比

| 特性 | = delete | private 不定义 |
|-----|----------|---------------|
| 语法清晰度 | 明确表达意图 | 需要理解惯例 |
| 错误时机 | 编译时 | 链接时 |
| 成员/友元访问 | 完全禁止 | 仍可访问（链接错误） |
| 可删除任意函数 | 是 | 否，仅限特殊成员函数 |

---

## 3. 其他特殊函数声明语法

### 3.1 `= default`
显式请求编译器生成默认实现。
```c++
class A {
public:
  A() = default;                    // 默认构造函数
  A(const A&) = default;            // 默认复制构造函数
  A& operator=(const A&) = default; // 默认复制赋值运算符
  ~A() = default;                   // 默认析构函数
};
```

### 3.2 `= 0`
声明纯虚函数，使类成为抽象类。
```c++
class Shape {
public:
  virtual double area() = 0;  // 纯虚函数
  virtual ~Shape() = default;
};
```

### 3.3 `override`
显式标记函数覆盖基类虚函数。
```c++
struct Derived : Base {
  void foo() override;  // 必须覆盖 Base::foo()
};
```

### 3.4 `constexpr`
声明常量表达式函数，可在编译时求值。
```c++
constexpr int square(int x) {
  return x * x;
}
constexpr int result = square(5);  // 编译时计算
```

### 3.5 `consteval` (C++20)
声明立即函数，必须在编译时求值。
```c++
consteval int square(int x) {
  return x * x;
}
int arr[square(3)];  // 必须在编译时计算
```

### 3.6 `noexcept`
指定函数不会抛出异常。
```c++
void foo() noexcept;           // 保证不抛异常
void bar() noexcept(true);     // 同上
void baz() noexcept(false);    // 可能抛异常
```

### 3.7 `explicit`
防止隐式类型转换。
```c++
class A {
public:
  explicit A(int x);           // 禁止 A a = 42;
  explicit operator bool();    // 禁止 bool b = a;
};
```

### 3.8 `inline`
建议内联展开，允许多次定义。
```c++
inline int add(int a, int b) {
  return a + b;
}
```

---

## 4. std::initializer_list (初始化列表)

### 作用
支持统一初始化语法，允许使用花括号 `{}` 初始化对象。

### 特性
- 轻量级代理对象，指向元素数组
- 只读访问，不可修改元素
- 支持任意长度的同类型元素

### 示例
```c++
class A {
public:
  A(std::initializer_list<int> list) {
    for (int x : list) {
      // 处理每个元素
    }
  }
};

A a{1, 2, 3, 4, 5};  // 使用初始化列表
A b = {1, 2, 3};     // 同上
```

### 初始化优先级
当构造函数重载时：
1. 优先匹配 `std::initializer_list` 构造函数
2. 如果不匹配，再考虑其他构造函数

```c++
class A {
public:
  A(int x, int y) { /* 两个参数 */ }
  A(std::initializer_list<int>) { /* 初始化列表 */ }
};

A a{1, 2};     // 调用 initializer_list 版本
A b(1, 2);     // 调用两个参数版本
```

---

## 5. Explicit Conversion Function (显式转换函数)

### 作用
使用 `explicit` 修饰转换函数，防止隐式转换。

### 示例
```c++
struct A {
  operator bool() const { return true; }  // 隐式转换
};

struct B {
  explicit operator bool() const { return true; }  // 显式转换
};

A a;
if (a);       // OK: 调用 A::operator bool()
bool ba = a;  // OK: 隐式转换

B b;
if (b);       // OK: 上下文转换
bool bb = b;  // Error: 复制初始化不考虑显式转换函数
```

### 关键区别

| 初始化方式 | 隐式转换函数 | 显式转换函数 |
|-----------|-------------|-------------|
| `if (obj)` | OK | OK（上下文转换） |
| `bool b = obj` | OK | Error（复制初始化） |
| `bool b(obj)` | OK | OK（直接初始化） |
| `bool b = bool(obj)` | OK | OK（显式转换） |

### 上下文转换
以下场景允许显式转换函数：
- `if`、`while`、`for` 条件
- `!`、`&&`、`||` 操作数
- `?:` 条件表达式
- `static_assert` 条件

---

## 6. Ref-Qualified Member Function (引用限定的成员函数)

### 作用
根据 `*this` 是左值还是右值，选择不同的成员函数重载。

### 语法
```c++
struct Foo {
  void func() &;       // *this 是左值时调用
  void func() const&;  // *this 是 const 左值时调用
  void func() &&;      // *this 是右值时调用
  void func() const&&; // *this 是 const 右值时调用
};
```

### 完整示例
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
Bar bar = foo.getBar();              // 调用 Bar& getBar() &

const Foo foo2{};
Bar bar2 = foo2.getBar();            // 调用 const Bar& getBar() const&

Foo{}.getBar();                      // 调用 Bar&& getBar() &&
std::move(foo).getBar();             // 调用 Bar&& getBar() &&
std::move(foo2).getBar();            // 调用 const Bar&& getBar() const&
```

### 设计意图
优化资源转移：
- 左值对象：返回引用，避免不必要的拷贝或移动
- 右值对象：返回右值引用，允许资源被移动

### 与 const 限定符的区别

| 限定符 | 作用对象 | 目的 |
|-------|---------|-----|
| `const` | 成员函数的行为 | 防止修改对象状态 |
| `&`/`&&` | 对象本身（`*this`）的值类别 | 根据对象是左值还是右值选择不同行为 |

---

## 7. std::forward (完美转发)

### 作用
返回传递给它的参数，同时保持其值类别（左值/右值）和 cv 限定符。

### 定义
```c++
template <typename T>
T&& forward(typename remove_reference<T>::type& arg) {
  return static_cast<T&&>(arg);
}
```

### 核心原理

#### 引用折叠规则
| 模板参数 T | `T&&` 的实际类型 |
|-----------|-----------------|
| `int` | `int&&`（右值引用） |
| `int&` | `int&`（左值引用） |
| `const int&` | `const int&`（const 左值引用） |

#### 工作流程
```c++
template <typename T>
A wrapper(T&& arg) {
  return A{std::forward<T>(arg)};
}

wrapper(A{});           // T = A，调用移动构造
A a;
wrapper(a);             // T = A&，调用复制构造
wrapper(std::move(a));  // T = A，调用移动构造
```

### 为什么需要 std::forward

**问题**：命名变量永远是左值，即使它是右值引用类型。
```c++
template <typename T>
A bad_wrapper(T&& arg) {
  return A{arg};  // ❌ arg 是左值，总是调用复制构造
}
```

**解决**：使用 `std::forward<T>(arg)` 保持原始值类别。

### std::forward vs std::move

| 特性 | std::forward | std::move |
|-----|-------------|-----------|
| 目的 | 保持值类别 | 强制转换为右值 |
| 条件性 | 有条件地转换（根据 T） | 无条件转换 |
| 使用场景 | 泛型转发函数 | 明确需要移动语义 |

### 典型应用

#### 工厂函数
```c++
template <typename T, typename... Args>
std::unique_ptr<T> make_unique(Args&&... args) {
  return std::unique_ptr<T>(new T(std::forward<Args>(args)...));
}
```

#### emplace 方法
```c++
template <typename... Args>
void emplace_back(Args&&... args) {
  new (end()) T(std::forward<Args>(args)...);
}
```

---

## 参考资源

- [CPP11_zh.md](file:///home/ubt/workspace/modern-cpp-features/CPP11_zh.md)
