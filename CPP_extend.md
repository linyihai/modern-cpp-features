# C++ 扩展内容

## std::thread

### 基础示例——真正能看到并发

```c++
#include <iostream>
#include <thread>
#include <chrono>

void worker(int id, int delay_ms) {
    for (int i = 0; i < 3; ++i) {
        std::cout << "线程 " << id << " 正在工作... 第 " << i + 1 << " 次" << std::endl;
        std::this_thread::sleep_for(std::chrono::milliseconds(delay_ms));
    }
    std::cout << "线程 " << id << " 完成！" << std::endl;
}

int main() {
    std::thread t1(worker, 1, 300);  // 线程1，间隔300ms
    std::thread t2(worker, 2, 500);  // 线程2，间隔500ms

    t1.join();  // 等待线程1结束
    t2.join();  // 等待线程2结束

    std::cout << "所有线程已完成" << std::endl;
}
```

输出会交错出现，清楚地看到两个线程在**同时运行**：

```text
线程 1 正在工作... 第 1 次
线程 2 正在工作... 第 1 次
线程 1 正在工作... 第 2 次
线程 2 正在工作... 第 2 次
线程 1 正在工作... 第 3 次
线程 2 正在工作... 第 3 次
线程 1 完成！
线程 2 完成！
所有线程已完成
```

### 四种传递可调用对象的方式

```c++
#include <thread>
#include <iostream>

// 方式1：普通函数
void func(int x) { std::cout << x << std::endl; }
std::thread t1(func, 42);

// 方式2：Lambda 表达式
std::thread t2([](int x, int y) {
    std::cout << x + y << std::endl;
}, 3, 4);

// 方式3：函数对象 (functor)
struct Task {
    void operator()(const std::string& msg) {
        std::cout << msg << std::endl;
    }
};
std::thread t3(Task(), "hello");

// 方式4：成员函数指针
struct Foo {
    void bar(int n) { std::cout << n << std::endl; }
};
Foo obj;
std::thread t4(&Foo::bar, &obj, 100);  // 注意：需要传对象指针

t1.join(); t2.join(); t3.join(); t4.join();
```

### 线程同步——互斥锁

多线程同时访问共享数据时，必须加锁保护：

```c++
#include <thread>
#include <mutex>
#include <vector>
#include <iostream>

std::mutex mtx;
int counter = 0;

void increment() {
    for (int i = 0; i < 100000; ++i) {
        std::lock_guard<std::mutex> lock(mtx);  // RAII 自动加锁/解锁
        ++counter;
    }
}

int main() {
    std::vector<std::thread> threads;
    for (int i = 0; i < 4; ++i)
        threads.emplace_back(increment);

    for (auto& t : threads)
        t.join();

    std::cout << "counter = " << counter << std::endl;  // 400000
}
```

如果不加锁，`++counter` 不是原子操作，多个线程同时读写会导致数据竞争，结果不确定。

### emplace_back 与 std::thread 的配合

```c++
std::vector<std::thread> threads;

// 这两行等价：
threads.emplace_back(increment);             // 在 vector 内部原地构造 std::thread(increment)
threads.push_back(std::thread(increment));   // 先构造临时对象，再移动进 vector
```

`emplace_back` 的参数是**元素的构造函数参数**，不是元素本身。`std::thread` 的构造函数接受一个可调用对象，所以 `increment` 这个函数名直接传进去，`thread` 自己就知道怎么把它变成一条线程。

### 线程管理——`detach` 与 `join`

| 操作 | 含义 |
|:---|:---|
| `join()` | 阻塞等待线程结束，线程资源被回收 |
| `detach()` | 分离线程，线程在后台独立运行，不再可控 |
| `joinable()` | 检查线程是否可 join/detach |

```c++
std::thread t([]{
    std::this_thread::sleep_for(std::chrono::seconds(2));
    std::cout << "后台任务完成" << std::endl;
});
t.detach();  // 分离后，主线程可以立即继续，不等它

// 注意：如果 t 没有被 join 或 detach，析构时会调用 std::terminate 崩溃！
```

### 关键要点总结

| 要点 | 说明 |
|:---|:---|
| 构造函数立即启动线程 | 一旦构造完成，线程就开始执行，不需要显式 `start()` |
| 必须 join 或 detach | 线程对象析构前必须调用其中一个，否则程序 `std::terminate` |
| 参数按值拷贝传递 | 线程参数默认按值拷贝（即使函数签名是引用），需用 `std::ref()` 传递引用 |
| `std::this_thread` | 命名空间包含 `sleep_for`、`sleep_until`、`yield`、`get_id` 等 |
| 不可复制，只能移动 | `std::thread` 删除了拷贝构造，只能用 `std::move` 转移所有权 |

```c++
// 传递引用需要用 std::ref
void modify(int& n) { n += 1; }

int x = 0;
std::thread t(modify, std::ref(x));  // 不加 std::ref 会编译错误
t.join();
std::cout << x;  // 1
```

---

## 智能指针

C++11 引入了三种智能指针：`std::unique_ptr`、`std::shared_ptr`、`std::weak_ptr`，替代了已弃用的 `std::auto_ptr`（C++17 中彻底移除）。智能指针的核心思想是 **RAII（资源获取即初始化）**：将堆内存的生命周期绑定到栈对象的生命周期，离开作用域时自动释放，从根本上杜绝内存泄漏。

### 三种智能指针对比

| 特性 | `unique_ptr` | `shared_ptr` | `weak_ptr` |
|:---|:---|:---|:---|
| 所有权 | 独占 | 共享 | 不拥有（观察） |
| 可复制 | 否 | 是 | 是 |
| 可移动 | 是 | 是 | 是 |
| 引用计数 | 无 | 有（强引用计数 + 弱引用计数） | 无（不增加强引用计数） |
| 内存开销 | 与裸指针相同（默认删除器） | 多一个控制块（通常 2 个指针大小） | 同 `shared_ptr` |
| 典型场景 | 工厂函数、PIMPL、容器元素 | 共享资源、异步回调、图/树结构 | 打破循环引用、观察者模式、缓存 |

---

### std::unique_ptr —— 独占所有权

**使用场景：**

- **工厂函数返回动态分配的对象**：调用者获得唯一所有权
- **PIMPL（Pointer to Implementation）**：隐藏实现细节
- **容器中存储多态对象**：`std::vector<std::unique_ptr<Base>>`
- **资源句柄包装**：文件、套接字等需要精确控制生命周期的资源

**基本用法：**

```c++
#include <memory>
#include <iostream>

struct Foo {
    Foo() { std::cout << "Foo 构造" << std::endl; }
    ~Foo() { std::cout << "Foo 析构" << std::endl; }
    void bar() { std::cout << "bar()" << std::endl; }
};

// 推荐方式：使用 std::make_unique（C++14）
auto p1 = std::make_unique<Foo>();
p1->bar();

// 移动所有权
auto p2 = std::move(p1);    // p1 变为 nullptr，p2 拥有 Foo
assert(p1 == nullptr);

// 不能复制
// auto p3 = p2;            // 编译错误！unique_ptr 不可复制

// 自定义删除器（用于非 delete 资源）
auto deleter = [](FILE* f) { if (f) fclose(f); };
std::unique_ptr<FILE, decltype(deleter)> file(fopen("test.txt", "r"), deleter);

// 离开作用域时自动释放 Foo 和关闭文件
```

**注意事项：**

1. **不可复制，只能移动**：这保证了同一时刻只有一个所有者，语义清晰
2. **默认删除器零开销**：`sizeof(unique_ptr<T>)` == `sizeof(T*)`（当使用默认删除器时）
3. **自定义删除器会增加大小**：使用函数指针或 lambda 时，`unique_ptr` 大小会增加
4. **可以转换为 `shared_ptr`**：当需要共享所有权时，可以从 `unique_ptr` 移动构造 `shared_ptr`

---

### std::shared_ptr —— 共享所有权

**使用场景：**

- **多个对象共享同一资源**：多个窗口共享同一纹理、多个请求共享同一连接
- **异步回调中延长对象生命周期**：确保回调执行时对象仍然存在
- **图形/树结构中的节点**：多个父节点引用同一子节点
- **观察者模式中共享状态**

**基本用法：**

```c++
#include <memory>
#include <iostream>
#include <thread>

struct Resource {
    int data;
    Resource(int d) : data(d) {
        std::cout << "Resource(" << data << ") 构造" << std::endl;
    }
    ~Resource() {
        std::cout << "Resource(" << data << ") 析构" << std::endl;
    }
};

// 推荐方式：使用 std::make_shared
auto sp1 = std::make_shared<Resource>(42);

{
    auto sp2 = sp1;              // 引用计数 +1（现在是 2）
    auto sp3 = sp1;              // 引用计数 +1（现在是 3）
    std::cout << "引用计数: " << sp1.use_count() << std::endl; // 3
} // sp2、sp3 离开作用域，引用计数 -2（现在是 1），但 Resource 不会被释放

// 异步回调中延长生命周期
std::thread t([sp1]() {
    std::this_thread::sleep_for(std::chrono::seconds(1));
    std::cout << sp1->data << std::endl;  // sp1 确保 Resource 仍然存活
});
t.detach();

// sp1 离开作用域时引用计数归零，Resource 被释放
```

**控制块详解：**

`shared_ptr` 内部有一个**控制块**，包含：
- 强引用计数（`use_count`）：控制对象生命周期，归零时释放对象
- 弱引用计数（`weak_count`）：控制块自身的生命周期，归零时释放控制块

```
堆内存布局：
┌──────────────┐    ┌───────────────────┐
│ shared_ptr 1 │───▶│     控制块        │
│  ptr ────────┼──┐ │  use_count: 2    │
│  ctrl ───────┼┐ │ │  weak_count: 0   │
└──────────────┘│ │ └───────────────────┘
                │ │                       
┌──────────────┐│ │ ┌───────────────────┐
│ shared_ptr 2 ││ │ │     Resource      │
│  ptr ────────┼┘ │ │                   │
│  ctrl ───────┼──┘ │                   │
└──────────────┘    └───────────────────┘
```

**注意事项：**

1. **不要用同一个裸指针创建多个 `shared_ptr`**：会导致多个独立的控制块，一个释放后另一个变成悬空指针
   ```c++
   auto* raw = new Resource(1);
   std::shared_ptr<Resource> sp1(raw);  // 控制块 1
   std::shared_ptr<Resource> sp2(raw);  // 控制块 2 —— 危险！
   // 离开作用域时 double free！
   ```
2. **循环引用会导致内存泄漏**：当两个 `shared_ptr` 互相引用时，引用计数永远不会归零
   ```c++
   struct Node {
       std::shared_ptr<Node> next;  // 如果用 weak_ptr 就不会有循环引用问题
   };
   auto a = std::make_shared<Node>();
   auto b = std::make_shared<Node>();
   a->next = b;
   b->next = a;  // 循环引用！a 和 b 永远不会被释放
   ```
3. **多线程下 `shared_ptr` 的线程安全性**：

   `shared_ptr` 的线程安全需要分三个层面理解：

   | 操作 | 线程安全？ | 说明 |
   |:---|:---|:---|
   | 多个线程**各自复制/销毁**同一个 `shared_ptr` 的副本 | ✅ 安全 | 引用计数是原子操作，每个线程持有自己的副本 |
   | 多个线程**同时读写同一个 `shared_ptr` 变量**（非副本） | ❌ 不安全 | 需要用 `std::atomic_load`/`std::atomic_store` 或互斥锁 |
   | 多个线程通过 `shared_ptr` **操作托管对象** | ❌ 不安全 | 跟普通指针一样，需要加锁保护对象 |

   **场景一：安全的模式——每个线程持有自己的副本（推荐）**

   ```c++
   #include <memory>
   #include <thread>
   #include <vector>
   #include <iostream>

   struct Data {
       int value = 0;
   };

   auto sp = std::make_shared<Data>();

   std::vector<std::thread> threads;
   for (int i = 0; i < 4; ++i) {
       threads.emplace_back([sp]() {  // 按值捕获 sp，每个线程获得一个副本
           // 引用计数的增减是原子操作，安全
           // 但 sp->value 的读写不是线程安全的！
       });
   }
   for (auto& t : threads) t.join();
   // 所有线程结束，sp 的引用计数恢复到 1，安全
   ```

   **场景二：危险的模式——多线程修改同一个 `shared_ptr` 变量**

   ```c++
   #include <memory>
   #include <thread>

   auto sp = std::make_shared<int>(42);

   // 线程 1：重置 sp
   std::thread t1([&sp]() {
       sp.reset();  // 危险！和 t2 同时操作同一个 sp 变量
   });

   // 线程 2：复制 sp
   std::thread t2([&sp]() {
       auto copy = sp;  // 危险！引用计数读写竞争
   });
   // 这会导致数据竞争（data race），属于未定义行为！
   ```

   **正确的做法**：多线程共享同一个 `shared_ptr` 变量时，要么用互斥锁保护，要么用 `std::atomic_load`/`std::atomic_store`（C++20 前是自由函数，C++20 起有 `std::atomic<std::shared_ptr>`）：

   ```c++
   #include <memory>
   #include <thread>
   #include <mutex>

   std::shared_ptr<int> sp = std::make_shared<int>(42);
   std::mutex mtx;

   // 写线程
   std::thread writer([&]() {
       auto new_sp = std::make_shared<int>(100);
       std::lock_guard<std::mutex> lock(mtx);
       sp = new_sp;  // 加锁保护，安全
   });

   // 读线程
   std::thread reader([&]() {
       std::shared_ptr<int> local;
       {
           std::lock_guard<std::mutex> lock(mtx);
           local = sp;  // 加锁保护，安全
       }
       if (local) std::cout << *local << std::endl;
   });

   writer.join();
   reader.join();
   ```

   **关键结论**：`shared_ptr` 的引用计数是原子的，所以**复制/销毁 `shared_ptr` 对象本身**是线程安全的——但前提是每个线程操作的是自己的副本，而不是同一个 `shared_ptr` 变量。如果多个线程要修改**同一个 `shared_ptr` 变量**（比如重新赋值），必须加锁。

4. **不要在函数参数中直接创建 `shared_ptr`**：应使用 `make_shared` 避免异常安全问题

---

### std::weak_ptr —— 弱引用（观察者）

**使用场景：**

- **打破 `shared_ptr` 的循环引用**：这是最主要的使用场景
- **观察者模式**：观察者持有被观察者的 `weak_ptr`，不阻止其释放
- **缓存实现**：缓存持有 `weak_ptr`，当外部没有引用时自动清理
- **检查对象是否仍然存活**：通过 `expired()` 或 `lock()` 判断

**基本用法：**

```c++
#include <memory>
#include <iostream>

// 解决循环引用：用 weak_ptr 替代 shared_ptr
struct Node {
    std::weak_ptr<Node> parent;  // 弱引用父节点
    std::shared_ptr<Node> child; // 强引用子节点
    int value;
    
    Node(int v) : value(v) {}
    ~Node() { std::cout << "Node(" << value << ") 析构" << std::endl; }
};

auto root = std::make_shared<Node>(1);
auto leaf = std::make_shared<Node>(2);
root->child = leaf;
leaf->parent = root;  // 使用 weak_ptr，不会形成循环引用
// root 和 leaf 都能正常释放

// 使用 weak_ptr 前必须先 lock() 获取 shared_ptr
if (auto sp = leaf->parent.lock()) {
    std::cout << "父节点值: " << sp->value << std::endl;
} else {
    std::cout << "父节点已被释放" << std::endl;
}
```

**注意事项：**

1. **`weak_ptr` 不增加引用计数**：它不影响对象的生命周期
2. **使用前必须 `lock()`**：`lock()` 返回一个 `shared_ptr`，如果对象已释放则返回空指针。不能直接解引用 `weak_ptr`
3. **`expired()` 与 `lock()` 之间存在竞态条件**：在多线程环境中，`expired()` 返回 false 后对象可能被其他线程释放，应直接使用 `lock()` 并检查返回值
4. **`weak_ptr` 依赖控制块**：只有当 `shared_ptr` 存在时才能创建 `weak_ptr`，因为控制块由 `shared_ptr` 创建

---

### std::make_shared 与 std::make_unique

**为什么推荐使用 `make_*` 而不是 `new`：**

1. **异常安全**：函数参数求值顺序不确定，`new` 可能导致内存泄漏
   ```c++
   // 危险：如果 function_that_throws() 抛出异常，new T{} 的内存泄漏
   foo(std::shared_ptr<T>(new T{}), function_that_throws());

   // 安全：make_shared 是单个函数调用，不会泄漏
   foo(std::make_shared<T>(), function_that_throws());
   ```

2. **减少内存分配次数**：`make_shared` 一次性分配对象和控制块，`new` + `shared_ptr` 需要两次分配
   ```
   make_shared: 一次分配 → [控制块 | 对象]
   new + shared_ptr: 两次分配 → [控制块] + [对象]（分别分配）
   ```

3. **代码简洁**：避免重复写类型名
   ```c++
   auto sp = std::make_shared<Foo>(arg1, arg2);           // 简洁
   std::shared_ptr<Foo> sp(new Foo(arg1, arg2));           // 冗长
   ```

**`make_shared` 的局限性：**

1. 不能使用自定义删除器（需要用构造函数）
2. 对象和控制块在同一块内存中，如果 `weak_ptr` 存在，对象内存会延迟到最后一个 `weak_ptr` 释放时才回收（弱引用会阻止控制块释放，而控制块和对象在一起）
3. `make_unique` 是 C++14 引入的，C++11 中需要自己实现

**C++11 中实现 `make_unique`：**

```c++
template<typename T, typename... Args>
std::unique_ptr<T> make_unique(Args&&... args) {
    return std::unique_ptr<T>(new T(std::forward<Args>(args)...));
}
```

---

### 所有权转移速查

```c++
// unique_ptr → unique_ptr（移动）
auto u1 = std::make_unique<int>(42);
auto u2 = std::move(u1);           // u1 变为 nullptr

// unique_ptr → shared_ptr（移动）
auto u3 = std::make_unique<int>(42);
std::shared_ptr<int> s1 = std::move(u3);  // 所有权从 unique_ptr 转移到 shared_ptr

// shared_ptr → shared_ptr（复制/移动）
auto s2 = std::make_shared<int>(42);
auto s3 = s2;                      // 复制，引用计数 +1
auto s4 = std::move(s2);           // 移动，s2 变为 nullptr，引用计数不变

// shared_ptr → weak_ptr
std::weak_ptr<int> w1 = s3;        // 不增加引用计数

// weak_ptr → shared_ptr（需要 lock）
if (auto s5 = w1.lock()) {
    // s5 是有效的 shared_ptr
}

// 不能从 shared_ptr 转回 unique_ptr（所有权已共享，无法独占）
```

---

### 常见陷阱

| 陷阱 | 说明 |
|:---|:---|
| 用裸指针创建多个 `shared_ptr` | 导致 double free，必须从第一个 `shared_ptr` 复制 |
| `shared_ptr` 循环引用 | 内存泄漏，用 `weak_ptr` 打破循环 |
| 在函数参数中直接 `new` | 异常不安全，用 `make_shared`/`make_unique` |
| 将 `this` 指针直接给 `shared_ptr` | 导致多个控制块，应继承 `std::enable_shared_from_this` |
| 忘记 `weak_ptr::lock()` 检查 | 直接使用可能访问已释放的对象 |

---

## std::chrono

`std::chrono` 是 C++11 引入的时间库，提供了**时钟（Clock）**、**时间点（Time Point）**、**持续时间（Duration）** 三大核心抽象，替代 C 风格的 `time()`/`clock()`。

### 三大核心概念

| 概念 | 说明 | 典型类型 |
|:---|:---|:---|
| **Duration（时长）** | 时间间隔，模板参数是数值类型和 tick 周期 | `std::chrono::seconds`、`milliseconds`、`microseconds`、`nanoseconds`、`minutes`、`hours` |
| **Time Point（时间点）** | 时钟上的某个时刻，由时钟 + 时长组成 | `std::chrono::time_point<Clock, Duration>` |
| **Clock（时钟）** | 提供当前时间、tick 周期的时钟源 | `system_clock`、`steady_clock`、`high_resolution_clock` |

---

### 三种内置时钟

| 时钟 | 特点 | 适用场景 |
|:---|:---|:---|
| `std::chrono::system_clock` | 系统时钟，可转为日历时间（`to_time_t`），可被用户/NTP 调整 | 显示时间、日志时间戳 |
| `std::chrono::steady_clock` | 单调时钟，保证只会前进，不受系统时间调整影响 | 性能测量、超时计算 |
| `std::chrono::high_resolution_clock` | 最高精度时钟，通常是 `system_clock` 或 `steady_clock` 的别名 | 高精度计时 |

---

### 常用操作

**1. 代码计时（基准测试）**

```c++
#include <chrono>
#include <iostream>

auto start = std::chrono::steady_clock::now();
// 被测代码...
auto end = std::chrono::steady_clock::now();

auto elapsed = std::chrono::duration<double>(end - start);  // 秒
std::cout << "耗时: " << elapsed.count() << " 秒" << std::endl;

// 或者用不同单位
auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
std::cout << "耗时: " << ms.count() << " 毫秒" << std::endl;
```

**2. 时长字面量（C++14 起）**

```c++
using namespace std::chrono_literals;

auto half_day    = 12h;       // std::chrono::hours
auto timeout     = 500ms;     // std::chrono::milliseconds
auto interval    = 1s;        // std::chrono::seconds
auto precise     = 100us;     // std::chrono::microseconds
auto nano_delay  = 50ns;      // std::chrono::nanoseconds
```

**3. 时长运算和转换**

```c++
#include <chrono>
#include <iostream>

// 时长算术
auto t1 = std::chrono::seconds(10);
auto t2 = std::chrono::milliseconds(500);
auto total = t1 + t2;  // 自动转换为更精确的类型（milliseconds）
// total 类型为 std::chrono::milliseconds(10500)

// duration_cast：有损转换（向下取整）
auto sec = std::chrono::duration_cast<std::chrono::seconds>(
    std::chrono::milliseconds(2499)
);
std::cout << sec.count();  // 2 秒（不是 2.5！向下取整丢失了 499ms）

// 浮点数时长避免精度损失
auto precise_duration = std::chrono::duration<double>(std::chrono::milliseconds(2499));
std::cout << precise_duration.count();  // 2.499 秒
```

**4. 时间点运算**

```c++
auto now = std::chrono::system_clock::now();
auto tomorrow = now + std::chrono::hours(24);  // 时间点 + 时长 = 时间点

auto diff = tomorrow - now;  // 时间点 - 时间点 = 时长
std::cout << std::chrono::duration_cast<std::chrono::hours>(diff).count();  // 24
```

**5. 与 C 时间互转（仅 `system_clock`）**

```c++
auto now = std::chrono::system_clock::now();
std::time_t t = std::chrono::system_clock::to_time_t(now);
std::cout << std::ctime(&t);  // 打印可读时间

// 反向转换：time_t → time_point
std::time_t raw = std::time(nullptr);
auto tp = std::chrono::system_clock::from_time_t(raw);
```

**6. 超时/睡眠**

```c++
// 线程睡眠（配合 std::thread）
std::this_thread::sleep_for(std::chrono::milliseconds(100));

// 条件变量的超时等待
std::mutex mtx;
std::unique_lock<std::mutex> lock(mtx);
std::condition_variable cv;
cv.wait_for(lock, std::chrono::seconds(5));  // 最多等 5 秒
```

---

### 注意事项

| 注意事项 | 说明 |
|:---|:---|
| **`duration_cast` 是截断而非四舍五入** | `duration_cast<seconds>(milliseconds(1499))` 得到 1 秒，丢失 499ms |
| **`steady_clock` 不能转日历时间** | 没有 `to_time_t`，它只保证单调递增，起点不一定是 epoch |
| **`system_clock` 会受系统时间调整影响** | 用户改时间、NTP 同步会导致时钟跳变，不适合计时测量 |
| **`high_resolution_clock` 可能只是别名** | 在 MSVC 上是 `steady_clock`，在 libstdc++/libc++ 上是 `system_clock`，不保证单调 |
| **时长字面量在 C++14 中** | C++11 需要 `<chrono>` 即可，但 `using namespace std::chrono_literals` 和 `12h`、`500ms` 等字面量是 C++14 |
| **`now()` 不是零开销** | 涉及系统调用，不要在极热的循环中频繁调用，应批量采样 |
| **跨时钟的 `time_point` 不能直接运算** | `system_clock::now() - steady_clock::now()` 编译错误 |
| **浮点 `duration` 可以避免截断** | `std::chrono::duration<double>` 保留小数部分，适合需要精确值的场景 |

---

## Tuple（元组）与 std::tie

`std::tuple` 是一个**固定大小、异构类型**的容器——可以装任意数量、任意类型的值，编译期决定。`std::tie` 创建一个**左值引用的 tuple**，用于将已有变量绑定到 tuple 的元素上解包。

### std::tuple

#### 使用场景

**1. 函数返回多个值**

传统方式需要用指针/引用作为输出参数，tuple 让多返回值变得自然：

```c++
std::tuple<int, std::string, bool> parse_result(const std::string& input) {
    if (input.empty()) return {0, "", false};
    return {1, "ok", true};
}

auto [code, msg, success] = parse_result("test");  // C++17 结构化绑定
```

**2. 临时的异构数据聚合**

不想为一次性使用的数据定义 struct 时：

```c++
std::vector<std::tuple<int, std::string, double>> records;
records.emplace_back(1, "Alice", 95.5);
records.emplace_back(2, "Bob",   87.0);
std::sort(records.begin(), records.end());  // 按字典序排序
```

**3. 编译期元编程**

`std::tuple` 是可变参数模板的经典应用，配合 `std::tuple_size`、`std::tuple_element` 等在编译期操作类型序列。

**4. 函数参数打包/解包**

```c++
template<typename... Args>
void call_delayed(Args&&... args) {
    auto params = std::make_tuple(std::forward<Args>(args)...);
    // 稍后用 std::apply (C++17) 调用
}
```

#### 注意事项

| 注意事项 | 说明 |
|:---|:---|
| **`std::get` 索引必须是编译期常量** | `std::get<0>(t)` 中的 `0` 必须是字面量或 constexpr，不能是运行时变量 |
| **类型重名会冲突** | 如果 tuple 中有两个 `int`，`std::get<int>(t)` 会编译失败，只能用索引 |
| **性能开销** | 创建和复制 tuple 有开销，不适合高频调用场景，除非编译器优化掉 |
| **可读性差** | `std::get<0>(t)` 不如 `t.name` 有意义，复杂场景建议用 struct |
| **C++17 后优先用结构化绑定** | `auto [x, y] = t` 比 `std::get<0>(t)` 清晰得多 |

---

### std::tie

#### 使用场景

**1. 解包 tuple / pair 到已有变量**

和结构化绑定不同，`std::tie` 可以给**已经存在的变量**赋值：

```c++
int x, y, z;
std::tie(x, y, z) = some_function_returns_tuple();  // 复用已有变量
```

**2. 实现自定义比较运算符（经典用法）**

当一个类有多个成员时，可以用 `std::tie` 一行实现字典序比较：

```c++
struct Person {
    std::string name;
    int age;
    double score;

    bool operator<(const Person& other) const {
        return std::tie(name, age, score) <
               std::tie(other.name, other.age, other.score);
    }
    // 同样可以实现 ==、> 等
};
```

**3. 忽略部分返回值**

配合 `std::ignore` 只提取关心的值：

```c++
std::set<int> s;
std::set<int>::iterator it;
bool inserted;

std::tie(it, inserted) = s.insert(42);  // 需要两个值
// 或：只关心是否插入成功
std::tie(std::ignore, inserted) = s.insert(42);
```

#### 注意事项

| 注意事项 | 说明 |
|:---|:---|
| **只能绑定到左值** | `std::tie` 创建的是左值引用的 tuple，不能用 `std::tie(a, 42)` 这种常量 |
| **C++17 结构化绑定更简洁** | `auto [a, b] = func()` 声明的变量是新的，`tie` 的优势是给已有变量赋值 |
| **`std::ignore` 不能用于结构化绑定** | 结构化绑定中如果要忽略，用 `std::ignore` 不行，只能不声明（C++26 引入了 `_`） |
| **`tie` 返回的 tuple 是引用类型** | 对 `tie` 的赋值实际修改的是原始变量，这是它解包的基础 |
| **类型必须匹配** | `std::tie` 绑定的变量类型必须与 tuple 对应位置类型兼容 |

---

### tie vs 结构化绑定，什么时候用哪个？

| 场景 | 推荐 |
|:---|:---|
| 声明新变量并解包 | `auto [a, b] = func();`（C++17 结构化绑定） |
| 给已有变量赋值 | `std::tie(a, b) = func();` |
| 需要忽略部分值 | `std::tie(a, std::ignore) = func();` |
| 实现比较运算符 | `std::tie` 是最简洁的方式 |
| 需要引用语义 | `auto& [a, b] = func();` 结构化绑定也支持 |

---

## std::ref / std::cref

`std::ref` 和 `std::cref` 返回一个 `std::reference_wrapper<T>` 对象，核心作用是**把引用包装成一个可拷贝、可赋值的对象**，从而绕过 C++ 中"引用不能重新绑定"和"模板推导会丢失引用"的限制。

### 必须使用的三大场景

**场景一：`std::thread` 传递引用参数**

`std::thread` 的构造函数**按值拷贝**所有参数，即使函数参数签名是 `int&`，不加 `std::ref` 会编译失败：

```c++
#include <thread>
#include <iostream>

void modify(int& n) {
    n += 100;
}

int x = 0;
// std::thread t(modify, x);     // 编译错误！x 按值拷贝后是右值，不能绑定到 int&
std::thread t(modify, std::ref(x));  // 正确：std::ref 包装后传递引用
t.join();
std::cout << x;  // 100
```

**场景二：`std::bind` / `std::function` 绑定引用**

```c++
#include <functional>
#include <iostream>

void print(int& n) {
    std::cout << n << std::endl;
}

int x = 42;
// auto f = std::bind(print, x);        // x 被拷贝，修改外部 x 不会影响 f
auto f = std::bind(print, std::ref(x));  // 正确：持有 x 的引用

x = 100;
f();  // 输出 100
```

**场景三：容器中存储引用**

```c++
// std::vector<int&> vec;                   // 编译错误！不能声明引用的容器
std::vector<std::reference_wrapper<int>> vec;  // 正确

int a = 1, b = 2, c = 3;
vec.push_back(std::ref(a));
vec.push_back(std::ref(b));
vec.push_back(std::ref(c));

vec[0].get() = 100;  // 修改 a
std::cout << a;       // 100
```

---

### 其他实用场景

**场景四：`std::async` 传递引用**

```c++
int x = 0;
auto fut = std::async(std::launch::async, modify, std::ref(x));
fut.get();
std::cout << x;  // 100
```

**场景五：`std::make_pair` / `std::make_tuple` 保持引用**

```c++
int x = 42;
auto p = std::make_pair(std::ref(x), std::ref(x));  // pair<int&, int&>
p.first = 100;
std::cout << x;  // 100
```

**场景六：算法中传递可变的函数对象**

```c++
struct Counter {
    int count = 0;
    void operator()(int) { ++count; }
};

Counter c;
std::vector<int> v = {1, 2, 3, 4, 5};
std::for_each(v.begin(), v.end(), std::ref(c));
std::cout << c.count;  // 5
```

---

### `std::ref` vs `T&` 对比

| | `T&` | `std::reference_wrapper<T>` |
|:---|:---|:---|
| 可拷贝 | 否（"=" 指赋值给原对象） | 是（"=" 指重新绑定引用） |
| 可作为容器元素 | 否 | 是 |
| 可被模板推导保留引用 | 否（推导为 `T`） | 是（推导为 `T&`） |
| 访问方式 | 直接使用 | `.get()` 或隐式转换 |
| 语法 | 简洁 | 略显冗长 |

---

### 注意事项

| 注意事项 | 说明 |
|:---|:---|
| **不管理生命周期** | `std::ref` 只管引用，原对象销毁后变成悬空引用，与智能指针完全不同 |
| **`std::cref` 提供 const 保护** | `std::cref` 返回的 `reference_wrapper<const T>` 禁止修改，编译期捕获 |
| **隐式转换有限** | 不能隐式转换为 `T&` 用于函数参数，需显式调用 `.get()` 或使用 `static_cast` |
| **增加间接层** | 内部通过指针实现，有轻微性能开销 |
| **`std::thread`/`std::async` 中按值捕获引用** | 按照值捕获 `reference_wrapper`，但它内部是指针，实际传递的是引用 |

---

## 内存模型与原子操作

C++11 引入了内存模型，为多线程和原子操作提供了标准库支持。在此之前，多线程编程依赖编译器扩展和平台 API，C++11 统一了这一切。

### std::atomic —— 无锁的原子操作

`std::atomic<T>` 保证对变量的读写是**原子性**的，不会被线程切换打断，且不需要互斥锁：

```c++
#include <atomic>
#include <thread>
#include <iostream>

std::atomic<int> counter(0);

void increment() {
    for (int i = 0; i < 100000; ++i) {
        ++counter;  // 原子操作，无需加锁
    }
}

int main() {
    std::thread t1(increment);
    std::thread t2(increment);
    t1.join();
    t2.join();
    std::cout << counter;  // 200000，结果确定
}
```

**常用原子操作：**

| 操作 | 说明 |
|:---|:---|
| `load()` | 原子读取当前值 |
| `store(val)` | 原子写入新值 |
| `exchange(val)` | 原子写入新值并返回旧值 |
| `compare_exchange_weak(expected, desired)` | CAS，如果当前值等于 expected 则替换为 desired |
| `compare_exchange_strong(expected, desired)` | 同上，但不允许伪失败 |
| `fetch_add(n)` / `fetch_sub(n)` | 原子加减，返回旧值 |
| `operator++` / `operator--` | 原子自增/自减 |
| `is_lock_free()` | 检查是否无锁实现 |

### 内存顺序（Memory Order）

原子操作可以指定内存顺序，控制编译器和 CPU 的指令重排程度：

```c++
#include <atomic>

std::atomic<int> flag(0);
int data = 0;

// 写线程
void producer() {
    data = 42;                              // 普通写
    flag.store(1, std::memory_order_release); // 释放：保证 data=42 在 flag=1 之前完成
}

// 读线程
void consumer() {
    while (flag.load(std::memory_order_acquire) == 0);  // 获取：保证读到 flag=1 后 data=42 可见
    std::cout << data;  // 保证输出 42
}
```

| 内存顺序 | 说明 | 场景 |
|:---|:---|:---|
| `memory_order_relaxed` | 只保证原子性，不保证顺序 | 简单计数器，不依赖其他变量 |
| `memory_order_acquire` | 读操作，后续读写不能被重排到该操作之前 | 获取锁、读取 flag 后访问共享数据 |
| `memory_order_release` | 写操作，之前的读写不能被重排到该操作之后 | 释放锁、写入共享数据后设置 flag |
| `memory_order_acq_rel` | 同时具有 acquire 和 release 语义 | RMW 操作（如 `exchange`） |
| `memory_order_seq_cst` | 顺序一致性（默认），全局统一顺序 | 需要严格顺序保证时 |

**默认用 `seq_cst`** 最安全，追求性能时再根据场景降低到 `acquire`/`release`。

### std::atomic_flag —— 自旋锁

最简单的原子类型，只有 `test_and_set` 和 `clear` 两种操作，适合实现自旋锁：

```c++
#include <atomic>

class SpinLock {
    std::atomic_flag flag = ATOMIC_FLAG_INIT;
public:
    void lock() {
        while (flag.test_and_set(std::memory_order_acquire)) {
            // 自旋等待
        }
    }
    void unlock() {
        flag.clear(std::memory_order_release);
    }
};
```

### std::promise / std::future —— 线程间传递值

**一次性的线程间通信**：一个线程通过 `promise` 设置值，另一个线程通过 `future` 获取：

```c++
#include <future>
#include <thread>
#include <iostream>

void worker(std::promise<int> p) {
    std::this_thread::sleep_for(std::chrono::seconds(1));
    p.set_value(42);  // 设置结果
}

int main() {
    std::promise<int> p;
    std::future<int> f = p.get_future();

    std::thread t(worker, std::move(p));  // promise 只能移动
    std::cout << "等待结果..." << std::endl;
    std::cout << "结果: " << f.get() << std::endl;  // 阻塞直到 set_value

    t.join();
}
```

### std::condition_variable —— 等待/通知

让线程等待某个条件成立，避免忙等：

```c++
#include <mutex>
#include <condition_variable>
#include <queue>
#include <thread>

std::mutex mtx;
std::condition_variable cv;
std::queue<int> q;
bool done = false;

void producer() {
    for (int i = 0; i < 5; ++i) {
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
        {
            std::lock_guard<std::mutex> lock(mtx);
            q.push(i);
        }
        cv.notify_one();  // 通知一个等待的消费者
    }
    {
        std::lock_guard<std::mutex> lock(mtx);
        done = true;
    }
    cv.notify_all();  // 通知所有等待的消费者
}

void consumer(int id) {
    while (true) {
        std::unique_lock<std::mutex> lock(mtx);
        cv.wait(lock, [] { return !q.empty() || done; });  // 等待条件成立
        if (!q.empty()) {
            int val = q.front();
            q.pop();
            lock.unlock();
            std::cout << "消费者 " << id << " 取出: " << val << std::endl;
        } else if (done) {
            break;
        }
    }
}
```

### 注意事项

| 注意事项 | 说明 |
|:---|:---|
| **`is_lock_free()` 不是总是 true** | 大对象可能无法无锁实现，检查后决定是否用于性能敏感场景 |
| **`compare_exchange_weak` 可能伪失败** | 在循环中用 `weak` 版本更高效（某些平台），用 `strong` 版本保证不伪失败 |
| **`future::get()` 只能调用一次** | 调用后 `future` 变成无效状态，需要多次获取用 `shared_future` |
| **`promise` 只能移动不能复制** | 一个 promise 只能设置一次值，`set_value` 前不能销毁 |
| **`condition_variable::wait` 必须用 `unique_lock`** | 不能用 `lock_guard`，因为 `wait` 需要解锁/重新加锁的能力 |
| **虚假唤醒** | `wait` 可能在没有 `notify` 时返回，必须用带条件的 `wait(lock, pred)` 形式 |
| **默认内存顺序是 `seq_cst`** | 最安全但非最高性能，根据场景可降低到 `acquire`/`release` |

---

## std::async

`std::async` 异步或延迟执行一个函数，返回 `std::future` 持有结果。对比 `std::thread`，它更轻量级、更易用。

### 三种启动策略

```c++
#include <future>
#include <iostream>

int compute() {
    std::this_thread::sleep_for(std::chrono::seconds(2));
    return 42;
}

// 策略 1：强制异步（新线程）
auto f1 = std::async(std::launch::async, compute);

// 策略 2：延迟执行（调用 get()/wait() 时才在当前线程执行）
auto f2 = std::async(std::launch::deferred, compute);

// 策略 3：默认（async | deferred），由实现决定
auto f3 = std::async(compute);

std::cout << f1.get() << std::endl;  // 阻塞等待
std::cout << f2.get() << std::endl;  // 此时才执行 compute
```

| 策略 | 行为 | 何时用 |
|:---|:---|:---|
| `std::launch::async` | 必须新线程执行 | 明确需要并发，不怕线程开销 |
| `std::launch::deferred` | 延迟执行，调用 `get()` 时在当前线程执行 | 可能不需要计算，按需执行 |
| 默认（`async \| deferred`） | 由实现决定 | 不关心细节，但行为不确定 |

### 使用场景

**1. 并行计算——将任务分派到后台**

```c++
#include <future>
#include <vector>
#include <numeric>

// 并行求和
int parallel_sum(const std::vector<int>& v) {
    auto mid = v.begin() + v.size() / 2;
    auto fut = std::async(std::launch::async, [&] {
        return std::accumulate(v.begin(), mid, 0);
    });
    int right = std::accumulate(mid, v.end(), 0);
    return fut.get() + right;  // 合并结果
}
```

**2. 异步 I/O 或网络请求**

```c++
auto fut = std::async(std::launch::async, [] {
    return fetch_data_from_server();  // 耗时操作
});
// 主线程做其他事...
do_other_work();
auto data = fut.get();  // 需要结果时再等待
```

**3. 超时控制**

```c++
auto fut = std::async(std::launch::async, long_running_task);

if (fut.wait_for(std::chrono::seconds(5)) == std::future_status::timeout) {
    std::cout << "任务超时，但线程仍在后台运行！" << std::endl;
    // 注意：future 析构时会阻塞等待线程完成，无法真正取消
}
```

### std::async vs std::thread

| | `std::async` | `std::thread` |
|:---|:---|:---|
| 返回值 | 通过 `future` 获取 | 无法直接获取，需用 `promise` 或共享变量 |
| 线程管理 | 自动管理（可延迟执行） | 手动管理（必须 join/detach） |
| 异常传播 | 异常在 `future::get()` 时重新抛出 | 异常在子线程中直接 `std::terminate` |
| 开销 | 可能使用线程池（实现决定） | 总是创建新线程 |
| 可控性 | 低（不能取消、不能控制线程细节） | 高（完全控制） |

### 注意事项

| 注意事项 | 说明 |
|:---|:---|
| **`future` 析构会阻塞** | 如果 `async` 策略是 `launch::async`，`future` 析构时会等待任务完成，无法真正"取消" |
| **默认策略行为不确定** | 不带策略的 `std::async` 可能不创建新线程，性能敏感场景应显式指定 `launch::async` |
| **`get()` 只能调用一次** | 调用后 `future` 无效，再次调用抛异常。需要多次读取用 `shared_future` |
| **异常在 `get()` 时才抛出** | 如果异步函数抛异常，它会存储在 `future` 中，调用 `get()` 时重新抛出 |
| **传引用需要用 `std::ref`** | 和 `std::thread` 一样，`std::async` 按值拷贝参数 |
| **不能取消已启动的任务** | C++ 标准没有提供取消机制，`future` 析构只是等待，不是取消 |
| **`wait_for` 返回 `deferred` 时** | 如果是 `launch::deferred` 策略，`wait_for` 返回 `future_status::deferred`，任务尚未执行 |

---

## std::begin / std::end

`std::begin` 和 `std::end` 是自由函数，用于泛型获取容器的开始和结束迭代器。它们的核心价值在于**统一处理标准容器和 C 风格数组**——数组没有 `.begin()` 成员函数，但 `std::begin(arr)` 可以正常工作。

### 经典使用场景

**场景一：模板函数统一处理容器和 C 数组**

```c++
template <typename T>
int CountTwos(const T& container) {
    return std::count_if(std::begin(container), std::end(container),
                         [](int item) { return item == 2; });
}

std::vector<int> vec = {2, 2, 43, 435, 4543, 534};
int arr[8] = {2, 43, 45, 435, 32, 32, 32, 32};
auto a = CountTwos(vec);  // 2
auto b = CountTwos(arr);  // 1
```

**场景二：对 C 数组的子范围操作**

C 数组的 `std::begin` 返回指针，可以做算术运算，让子范围操作更安全：

```c++
int arr[] = {0, 1, 2, 3, 4, 5, 6, 7, 8, 9};

// 只处理前 5 个元素
auto it = std::find(std::begin(arr), std::begin(arr) + 5, 3);  // 找到 arr[3]

// 处理后 3 个元素
std::sort(std::end(arr) - 3, std::end(arr));
// arr = {0, 1, 2, 3, 4, 5, 7, 8, 9}

// 跳过头尾（第二个到倒数第二个）
for (auto it = std::begin(arr) + 1; it != std::end(arr) - 1; ++it) {
    // 处理 arr[1] 到 arr[8]
}
```

没有 `std::begin/end` 时只能手写 `arr` 和 `arr + N`，一旦数组大小变了就要改代码，容易出错。

**场景三：`std::size` / `std::empty` 配合使用（C++17）**

与 `std::begin/end` 组成完整泛型套件：

```c++
#include <iterator>

template <typename Container>
void process(Container& c) {
    if (std::empty(c)) return;  // 泛型判空

    auto n = std::size(c);      // 泛型获取大小
    auto first = std::begin(c);
    auto last = std::end(c);

    std::cout << "大小: " << n << "，首元素: " << *first << std::endl;
    std::sort(first, last);
}
```

**场景四：`std::cbegin` / `std::cend` 强制获取 const 迭代器（C++14）**

即使容器本身是非 const 的，用 `cbegin/cend` 也能保证只读遍历：

```c++
std::vector<int> v = {1, 2, 3, 4, 5};

// 即使 v 不是 const，cbegin 也返回 const_iterator
auto it = std::cbegin(v);  // std::vector<int>::const_iterator
// *it = 10;               // 编译错误！

// 泛型只读遍历
template <typename Container>
bool contains(const Container& c, int target) {
    return std::find(std::cbegin(c), std::cend(c), target) != std::cend(c);
}
```

**场景五：与 `std::distance` / `std::advance` 配合做泛型索引访问**

即使容器不支持随机访问（如 `std::list`），也能用这套组合做泛型"索引"：

```c++
template <typename Container>
auto get_nth(Container& c, size_t n) -> decltype(*std::begin(c)) {
    auto it = std::begin(c);
    std::advance(it, n);
    return *it;
}

std::vector<int> v = {1, 2, 3};
std::list<int>   l = {1, 2, 3};
int arr[] = {1, 2, 3};

get_nth(v, 2);   // 3（O(1) 随机访问）
get_nth(l, 2);   // 3（O(n) 线性遍历）
get_nth(arr, 2); // 3（O(1) 指针偏移）
```

**场景六：自定义容器自动获得泛型支持**

如果自定义容器实现了 `.begin()` / `.end()` 成员，`std::begin/end` 会自动调用它们：

```c++
struct MyContainer {
    int data[100];
    int* begin() { return data; }
    int* end()   { return data + 100; }
};

MyContainer c;
std::sort(std::begin(c), std::end(c));  // 自动调用 c.begin() / c.end()
```

---

### 注意事项

| 注意事项 | 说明 |
|:---|:---|
| **动态分配的数组不能用 `std::end`** | `int* p = new int[10]` 的指针丢失了大小信息，`std::end(p)` 编译失败 |
| **`std::cbegin/cend` 是 C++14** | C++11 只有 `std::begin/end`，const 版本需 C++14 |
| **`std::rbegin/rend` 是 C++14** | 反向迭代器自由函数也是 C++14 |
| **与 ADL 配合更安全** | 泛型代码中 `std::begin(c)` 比 `c.begin()` 更安全，后者要求类型必须有 `begin` 成员 |
| **`std::size` 是 C++17** | 泛型获取大小需 C++17，C++11 可用 `std::distance(std::begin(c), std::end(c))` 替代 |

---

## Lambda Capture Initializer 扩展

**C++11 的限制**：在 C++11 中，lambda 捕获列表只能通过 `[=]`（按值复制捕获）或 `[&]`（按引用捕获）或显式列出变量名来捕获。捕获的变量必须与封闭作用域中的变量同名，且只能捕获已有变量，无法通过表达式初始化捕获值。

C++14 引入了 **Lambda 捕获初始化器**（也叫 init-capture），允许使用任意表达式初始化 lambda 捕获。初始化表达式在 lambda **创建时**求值（而不是在调用时），捕获值的名称不需要与封闭作用域中的任何变量相关。

### 1. 基础用法：用表达式初始化捕获

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

### 2. 移动捕获：捕获仅可移动类型

因为现在可以将值移动（或转发）到 lambda 中，而以前只能通过复制或引用捕获，所以我们现在可以按值捕获仅可移动类型的 lambda。

```c++
auto p = std::make_unique<int>(1);
auto task1 = [=] { *p = 5; }; // ERROR: std::unique_ptr 不能被复制
auto task2 = [p = std::move(p)] { *p = 5; }; // OK: p 被移动构造到闭包对象中
```

### 3. 引用捕获重命名

使用这个特性，引用捕获可以使用与被引用变量不同的名称。

```c++
auto x = 1;
auto f = [&r = x, x = x * 10] {
  ++r;
  return r + x;
};
f(); // 将 x 设置为 2 并返回 12
```

### 4. 捕获成员变量（避免悬空 this 指针）

C++11 中捕获成员变量实际是通过捕获 `this` 指针隐式完成的，在异步场景中容易导致悬空指针。C++14 可以显式地将成员变量**按值复制**到闭包中，避免 `this` 生命周期问题。

```c++
struct Widget {
  int value;
  std::function<void()> getCallback() {
    // C++11: 捕获 this，异步调用时 this 可能已销毁
    // auto bad = [=] { return value; };

    // C++14: 将成员变量复制到闭包中，安全
    auto good = [v = value] { return v; };
    return good;
  }
};
```

### 5. 捕获时进行类型转换

```c++
std::vector<int> data = {1, 2, 3, 4, 5};
// 捕获 vector 的大小（而非引用整个 vector）
auto f = [size = data.size()] { return size; };
// 或者捕获后转换为其他类型
auto g = [value = static_cast<double>(data[0])] { return value * 2.5; };
```

### 6. 完美转发到闭包

结合 `std::forward` 实现完美转发捕获：

```c++
template <typename T>
auto makeLambda(T&& arg) {
  return [captured = std::forward<T>(arg)] {
    // 使用 captured...
  };
}
```

### 7. 异步/多线程场景：移动所有权到线程

```c++
auto data = std::make_unique<std::vector<int>>(100);
// 将 data 的所有权移动到新线程中
std::thread t([data = std::move(data)] {
  // 在新线程中独占访问 data
  (*data)[0] = 42;
});
```

### 8. 捕获 mutex 用于线程安全

```c++
std::mutex mtx;
int shared_resource = 0;
// 将 mutex 的引用捕获到闭包中，并给一个更清晰的名称
auto thread_safe_op = [&lock = mtx, &res = shared_resource] {
  std::lock_guard<std::mutex> guard(lock);
  ++res;
};
```

### 9. 捕获表达式结果

```c++
int a = 10, b = 20;
// 捕获表达式 a + b 的结果，而非 a 和 b 本身
auto f = [sum = a + b] { return sum; };
f(); // == 30
```

### C++11 vs C++14 捕获对比总结

| 场景 | C++11 | C++14 |
|------|-------|-------|
| 按值复制已有变量 | `[=]` 或 `[x]` | `[x]` 或 `[x = x]` |
| 按引用捕获 | `[&]` 或 `[&x]` | `[&x]` 或 `[&r = x]` |
| 移动捕获 | ❌ 不支持 | `[x = std::move(x)]` |
| 表达式初始化 | ❌ 不支持 | `[x = factory(2)]` |
| 捕获成员变量（按值） | ❌ 只能通过 `this` | `[v = this->value]` |
| 捕获时类型转换 | ❌ 不支持 | `[x = static_cast<double>(y)]` |
| 捕获时重命名 | ❌ 不支持 | `[&r = x]` 或 `[v = x]` |
| 完美转发捕获 | ❌ 不支持 | `[x = std::forward<T>(arg)]` |