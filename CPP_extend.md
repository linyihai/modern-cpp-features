# C++ 扩展内容

## C++ 标准库特性

### std::thread

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

### 智能指针

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

### std::chrono

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

### Tuple（元组）与 std::tie

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

### std::ref / std::cref

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

### 内存模型与原子操作

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

### std::async

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

### std::begin / std::end

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

### std::variant

`std::variant` 是 C++17 引入的**类型安全的联合体**，存储若干个指定类型中的某一个值，且不会像 C 的 `union` 那样有未定义行为。

**为什么需要：** C 的 `union` 无法跟踪当前存储的是哪个类型，需要手动用额外标志位判断，容易出错。

```c++
// C 的 union —— 不安全
union Value { int i; double d; };
Value v;
v.i = 42;
// v.d 此时是 undefined behavior
```

#### 基本用法

```c++
#include <variant>
#include <iostream>
#include <string>

std::variant<int, double, std::string> v{42};
std::cout << std::get<int>(v);  // 42

v = 3.14;
std::cout << std::get<double>(v);  // 3.14

v = "hello";
std::cout << std::get<std::string>(v);  // hello
```

**获取错误的类型会抛异常：**

```c++
v = 3.14;
// std::cout << std::get<int>(v);  // ❌ 抛出 std::bad_variant_access！
// 现在 v 存储的是 double，"hello" 已经被覆盖了
// 无法再取出 int 或 string

try {
    auto val = std::get<int>(v);  // 抛出异常
} catch (const std::bad_variant_access& e) {
    std::cout << "当前 variant 存储的是 double，不是 int";
}
```

这与 C 的 `union` 不同——C 的 `union` 不会报错，只会默默把数据解释为错误类型，产生未定义行为。`variant` 在运行时检查当前存储的类型，不匹配就直接抛异常。可以用 `std::holds_alternative<T>(v)` 先检查当前类型再获取：

```c++
if (std::holds_alternative<int>(v)) {
    std::cout << std::get<int>(v);
} else if (std::holds_alternative<double>(v)) {
    std::cout << std::get<double>(v);
}
```

#### 使用场景

**场景 1：替代枚举+union 的简单多态**

```c++
// C++17 之前 —— 需要 enum + union + switch
enum class ShapeType { Circle, Square, Triangle };
struct Shape {
    ShapeType type;
    union {
        double radius;
        double side;
        struct { double base, height; } triangle;
    };
};

// C++17 —— 一行搞定
using Shape = std::variant<Circle, Square, Triangle>;
Shape s = Circle{5.0};
```

**场景 2：错误处理（替代异常或输出参数）**

```c++
#include <variant>
#include <string>
#include <iostream>

// 返回值要么是 int，要么是错误信息
std::variant<int, std::string> parse_int(const std::string& s) {
    char* end;
    long val = std::strtol(s.data(), &end, 10);
    if (*end != '\0') return std::string("无效数字: " + s);
    return static_cast<int>(val);
}

// 使用
auto result = parse_int("42");
if (std::holds_alternative<int>(result)) {
    std::cout << "数字: " << std::get<int>(result);
} else {
    std::cout << "错误: " << std::get<std::string>(result);
}
```

**场景 3：访问者模式——替代 switch**

```c++
#include <variant>
#include <iostream>

struct Circle   { double radius; };
struct Square   { double side; };
struct Triangle { double base; double height; };

using Shape = std::variant<Circle, Square, Triangle>;

double area(const Shape& s) {
    return std::visit([](const auto& shape) {
        if constexpr (std::is_same_v<std::decay_t<decltype(shape)>, Circle>) {
            return 3.14159 * shape.radius * shape.radius;
        } else if constexpr (std::is_same_v<std::decay_t<decltype(shape)>, Square>) {
            return shape.side * shape.side;
        } else {
            return 0.5 * shape.base * shape.height;
        }
    }, s);
}

int main() {
    Shape s = Circle{5.0};
    std::cout << area(s);  // 78.5398
}
```

---

### std::optional

`std::optional` 表示一个**可能存在也可能不存在的值**，替代返回"哨兵值"（-1、nullptr、空字符串）的惯用法。

```c++
#include <optional>
#include <iostream>

// C++17 之前 —— 用特殊值表示"不存在"
int find_in_vec(const std::vector<int>& v, int target) {
    auto it = std::find(v.begin(), v.end(), target);
    return it == v.end() ? -1 : *it;  // -1 是哨兵值
}
// 如果容器中真有 -1，就无法区分

// C++17 —— 语义清晰
std::optional<int> find_in_vec(const std::vector<int>& v, int target) {
    auto it = std::find(v.begin(), v.end(), target);
    if (it == v.end()) return std::nullopt;  // 明确表示"不存在"
    return *it;
}
```

#### 使用场景

**场景 1：可能有返回值的函数**

```c++
#include <optional>
#include <iostream>

std::optional<double> safe_divide(double a, double b) {
    if (b == 0) return std::nullopt;  // 除零，无意义
    return a / b;
}

// 使用
auto result = safe_divide(10, 2);
if (result) {
    std::cout << *result;              // 5
    std::cout << result.value();       // 5
}
std::cout << result.value_or(0);       // 5（有值时）或 0（无值时）

auto bad = safe_divide(10, 0);
std::cout << bad.value_or(-1);         // -1
```

**场景 2：避免输出参数**

```c++
// C++17 之前 —— 用输出参数
bool get_config(const std::string& key, int& out_value);

// C++17 —— 返回 optional
std::optional<int> get_config(const std::string& key);

if (auto timeout = get_config("timeout")) {
    setup_connection(*timeout);
}
```

**场景 3：monadic 操作（C++23）**

```c++
// C++23 起支持 transform / and_then / or_else
auto result = get_config("timeout")
    .transform([](int v) { return v * 1000; })
    .or_else([] { return std::optional<int>{3000}; });
```

---

### std::any

`std::any` 是一个**可以存储任意类型单值**的类型安全容器，像类型擦除的盒子。

```c++
#include <any>
#include <iostream>
#include <string>

std::any x = 42;                 // 存 int
x = std::string("hello");        // 存 string
x = 3.14;                        // 存 double

// 读取时必须知道类型
std::cout << std::any_cast<double>(x);  // 3.14

// 类型不匹配抛异常
try {
    std::any_cast<int>(x);  // 抛出 std::bad_any_cast
} catch (const std::bad_any_cast& e) {
    std::cout << "类型错误";
}
```

#### 使用场景

**场景 1：异构容器**

```c++
#include <any>
#include <vector>
#include <iostream>
#include <string>

// 存储不同类型的值到同一个容器
std::vector<std::any> mixed;
mixed.push_back(42);
mixed.push_back(3.14);
mixed.push_back(std::string("hello"));

// 使用时需要知道类型
for (const auto& item : mixed) {
    if (item.type() == typeid(int)) {
        std::cout << std::any_cast<int>(item) << " (int)\n";
    } else if (item.type() == typeid(double)) {
        std::cout << std::any_cast<double>(item) << " (double)\n";
    } else if (item.type() == typeid(std::string)) {
        std::cout << std::any_cast<std::string>(item) << " (string)\n";
    }
}
```

**场景 2：属性系统**

```c++
struct PropertyBag {
    void set(const std::string& name, std::any value) {
        props_[name] = value;
    }
    template<typename T>
    T get(const std::string& name) const {
        auto it = props_.find(name);
        if (it == props_.end()) throw std::runtime_error("不存在");
        return std::any_cast<T>(it->second);
    }
private:
    std::map<std::string, std::any> props_;
};

PropertyBag bag;
bag.set("name", std::string("Alice"));
bag.set("age", 30);
std::cout << bag.get<std::string>("name");  // Alice
```

> **注意**：`std::any` 有动态内存分配开销，能用 `variant` 时优先用 `variant`。

---

### std::string_view

`std::string_view` 是**字符串的非拥有引用**——它不持有字符串数据，只持有指针和长度。零拷贝、只读访问。

```c++
#include <string_view>
#include <iostream>

// C++17 之前 —— 每次传字符串都会拷贝
void process(const std::string& s);  // 从 const char* 构造时会拷贝

// C++17 —— 任何字符串类型都能传，零拷贝
void process(std::string_view s);    // 只读引用，不拷贝
```

#### 使用场景

**场景 1：函数参数统一接受各种字符串类型**

```c++
#include <string_view>
#include <iostream>
#include <string>

// 一个函数同时接受 string、const char*、string_view
void print_first_char(std::string_view sv) {
    if (!sv.empty()) std::cout << sv.front();
}

int main() {
    std::string s = "hello";
    const char* c = "world";
    std::string_view sv = "!";

    print_first_char(s);   // ✅ 无拷贝
    print_first_char(c);   // ✅ 无拷贝
    print_first_char(sv);  // ✅ 无拷贝
}
```

**场景 2：高效分割字符串**

```c++
#include <string_view>
#include <vector>
#include <iostream>

// 分割字符串，返回 string_view 切片——零拷贝！
std::vector<std::string_view> split(std::string_view sv, char delim) {
    std::vector<std::string_view> result;
    while (!sv.empty()) {
        auto pos = sv.find(delim);
        if (pos == std::string_view::npos) {
            result.push_back(sv);
            break;
        }
        result.push_back(sv.substr(0, pos));  // 切出子串——零拷贝！
        sv.remove_prefix(pos + 1);
    }
    return result;
}

int main() {
    std::string data = "a,b,c,d";
    auto parts = split(data, ',');
    for (auto p : parts) std::cout << p << " ";  // a b c d
    // data 仍然完整，"a"、"b" 等指向 data 内部的子区间
}
```

**场景 3：解析日志行**

```c++
void parse_line(std::string_view line) {
    auto colon = line.find(':');
    auto key = line.substr(0, colon);       // 零拷贝
    auto value = line.substr(colon + 1);    // 零拷贝
    // 处理 key/value...
}
```

#### 注意事项

```c++
// ❌ 危险：string_view 不拥有数据
std::string_view bad() {
    std::string s = "hello";
    return s;  // s 析构后，返回的 string_view 是悬空指针！
}

// ✅ 安全的做法：确保原始字符串的生命周期长于 string_view
std::string data = "hello";
std::string_view sv = data;  // data 还在，sv 有效
```

---

### std::invoke

`std::invoke` 以统一的方式调用**任何可调用对象**——普通函数、函数指针、lambda、成员函数指针、成员对象指针。

```c++
#include <functional>
#include <iostream>

struct Foo {
    int value = 42;
    void bar(int n) { std::cout << n; }
};

int add(int a, int b) { return a + b; }

int main() {
    // 普通函数
    std::cout << std::invoke(add, 1, 2);                // 3

    // lambda
    std::cout << std::invoke([](int x) { return x * 2; }, 5); // 10

    // 成员函数指针
    Foo f;
    std::invoke(&Foo::bar, f, 99);                      // 99

    // 成员对象指针
    std::cout << std::invoke(&Foo::value, f);           // 42
}
```

#### 主要价值

`std::invoke` 是一致性接口，在模板元编程中尤其有用——你不知道调用者传的是普通函数还是成员函数，`invoke` 统一处理：

```c++
template<typename F, typename... Args>
auto call(F&& f, Args&&... args) {
    return std::invoke(std::forward<F>(f), std::forward<Args>(args)...);
    // 不管 f 是函数、lambda、还是成员函数指针，都能用
}
```

---

### std::apply

`std::apply` 用**元组中的元素作为参数**调用可调用对象。

```c++
#include <tuple>
#include <functional>
#include <iostream>

int add(int a, int b, int c) { return a + b + c; }

int main() {
    auto args = std::make_tuple(1, 2, 3);
    std::cout << std::apply(add, args);  // 6
}
```

#### 使用场景

**场景 1：将 tuple 展开为函数参数**

```c++
#include <tuple>
#include <iostream>

void print(const std::string& s, int n, double d) {
    std::cout << s << ", " << n << ", " << d;
}

int main() {
    auto t = std::make_tuple("hello", 42, 3.14);
    std::apply(print, t);  // hello, 42, 3.14
}
```

**场景 2：遍历 tuple 元素**

```c++
#include <tuple>
#include <iostream>

struct Printer {
    template<typename T>
    void operator()(const T& v) const {
        std::cout << v << " ";
    }
};

int main() {
    auto t = std::make_tuple(1, 2.5, "hello");
    std::apply([](auto&&... args) {
        (Printer{}(args), ...);  // 折叠表达式展开
    }, t);  // 1 2.5 hello
}
```

**场景 3：延迟执行（保存参数，稍后调用）**

```c++
template<typename F, typename... Args>
auto delay_call(F&& f, Args&&... args) {
    return [f = std::forward<F>(f),
            args = std::make_tuple(std::forward<Args>(args)...)]() {
        return std::apply(f, args);
    };
}

auto delayed = delay_call(add, 3, 4);
// ... 稍后执行
std::cout << delayed();  // 7
```

---

### std::filesystem

`std::filesystem` 提供了跨平台的文件系统操作——遍历目录、查询属性、创建/复制/删除文件和目录。

```c++
#include <filesystem>
#include <iostream>
namespace fs = std::filesystem;
```

#### 使用场景

**场景 1：批量重命名文件**

```c++
#include <filesystem>
#include <iostream>
namespace fs = std::filesystem;

void add_prefix(const std::string& dir, const std::string& prefix) {
    for (auto& entry : fs::directory_iterator(dir)) {
        if (entry.is_regular_file()) {
            auto new_name = entry.path().parent_path() /
                            (prefix + entry.path().filename().string());
            fs::rename(entry.path(), new_name);
            std::cout << "重命名: " << entry.path() << " → " << new_name << std::endl;
        }
    }
}
```

**场景 2：磁盘空间检查**

```c++
#include <filesystem>
namespace fs = std::filesystem;

bool has_enough_space(const std::string& path, uintmax_t needed) {
    auto space = fs::space(path);
    return space.available > needed;
}

// 使用
if (has_enough_space("/tmp", 1024 * 1024 * 100)) {
    // 有足够的空间...
}
```

**场景 3：递归查找文件**

```c++
#include <filesystem>
#include <vector>
namespace fs = std::filesystem;

std::vector<fs::path> find_all(const std::string& root,
                                const std::string& ext) {
    std::vector<fs::path> result;
    for (auto& entry : fs::recursive_directory_iterator(root)) {
        if (entry.is_regular_file() && entry.path().extension() == ext) {
            result.push_back(entry.path());
        }
    }
    return result;
}

// 一行代码就能递归遍历整个目录树！
```

---

### std::byte

`std::byte` 是**表示字节数据的标准类型**，替代 `char`/`unsigned char` 来做内存操作，避免与字符语义混淆。

```c++
#include <cstddef>
#include <iostream>

std::byte b{0xFF};
int i = std::to_integer<int>(b);  // 255

std::byte a{0x0F};
std::byte c = a & b;              // 位运算
std::cout << std::to_integer<int>(c);  // 15
```

```c++
#include <cstddef>
#include <vector>
#include <fstream>

// 读取文件到字节数组
std::vector<std::byte> read_binary(const std::string& path) {
    std::ifstream file(path, std::ios::binary | std::ios::ate);
    auto size = file.tellg();
    file.seekg(0);
    std::vector<std::byte> buffer(size);
    file.read(reinterpret_cast<char*>(buffer.data()), size);
    return buffer;
}
```

---

### Splicing for Maps and Sets（map 和 set 的拼接）

C++17 允许在关联容器间**移动节点**而不复制或分配内存——`extract` 取出节点，`insert` 将节点插入另一个容器。

```c++
#include <map>
#include <iostream>

int main() {
    std::map<int, std::string> src{{1, "one"}, {2, "two"}, {3, "three"}};
    std::map<int, std::string> dst{{4, "four"}};

    // 从 src 取出节点，插入 dst
    auto node = src.extract(2);
    dst.insert(std::move(node));

    // src: {1, "one"}, {3, "three"}
    // dst: {2, "two"}, {4, "four"}

    // 批量合并——无需复制键值对
    dst.merge(src);
    // src 为空，dst 包含所有 4 个元素
}
```

**核心价值**：零拷贝。C++17 之前，要在两个 map 之间移动元素，必须复制（或移动构造）键值对，这涉及堆分配。`extract` + `insert` 直接在红黑树中重链接节点。

---

### Parallel Algorithm（并行算法）

C++17 为大多数 STL 算法增加了**并行执行策略**，让排序、查找等操作自动利用多核。

```c++
#include <vector>
#include <algorithm>
#include <execution>
#include <iostream>

std::vector<int> data(1000000);
// ... 填充数据

// 顺序执行（默认）
std::sort(data.begin(), data.end());

// 并行执行
std::sort(std::execution::par, data.begin(), data.end());

// 向量化 + 并行
std::sort(std::execution::par_unseq, data.begin(), data.end());
```

#### 三种执行策略

| 策略 | 含义 |
|:---|:---|
| `std::execution::seq` | 顺序执行，同普通版本 |
| `std::execution::par` | 并行执行（多线程） |
| `std::execution::par_unseq` | 并行 + 向量化（允许交错） |

---

### std::sample

从序列中随机采样 n 个元素，每个元素等概率被选中。

```c++
#include <algorithm>
#include <vector>
#include <random>
#include <iostream>

std::vector<int> pop = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
std::vector<int> sample(3);

std::sample(pop.begin(), pop.end(),
            sample.begin(), 3,
            std::mt19937{std::random_device{}()});
// 随机选出 3 个元素
```

**真实场景：生成随机密码/ID**

```c++
const std::string chars =
    "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789";
std::string password(8, ' ');
std::sample(chars.begin(), chars.end(),
            password.begin(), 8,
            std::mt19937{std::random_device{}()});
// password = 随机 8 位字符串
```

---

### std::clamp

将值限制在上下界之间。比手写 `std::min(std::max(v, lo), hi)` 清晰得多。

```c++
#include <algorithm>
#include <iostream>

std::cout << std::clamp(42, 0, 10);   // 10（超出上限）
std::cout << std::clamp(-5, 0, 10);   // 0（低于下限）
std::cout << std::clamp(7, 0, 10);    // 7（在范围内）

// 实用场景：限制颜色值
int r = std::clamp(256, 0, 255);      // 255
int g = std::clamp(-10, 0, 255);      // 0
int b = std::clamp(128, 0, 255);      // 128
```

---

### std::reduce

类似 `std::accumulate` 但**并行执行**（且元素顺序不确定）。

```c++
#include <numeric>
#include <vector>
#include <iostream>

std::vector<int> v = {1, 2, 3, 4, 5};

// 普通求和
int sum = std::accumulate(v.begin(), v.end(), 0);  // 15

// 并行求和
int psum = std::reduce(std::execution::par,
                       v.begin(), v.end());         // 15

// 带初始值和操作
int product = std::reduce(v.begin(), v.end(),
                          1, std::multiplies<>{});  // 120
```

> **注意**：`reduce` 对操作符的结合性和交换性有要求（如 `+`、`*`、`min`、`max`），而 `accumulate` 严格按顺序执行。

---

### Prefix Sum Algorithm（前缀和算法）

```c++
#include <numeric>
#include <vector>
#include <iostream>

std::vector<int> v = {1, 2, 3, 4, 5};
std::vector<int> out(5);

// 包含性扫描（inclusive scan）：每个位置包含自身
std::inclusive_scan(v.begin(), v.end(), out.begin());
// out = {1, 3, 6, 10, 15}

// 排除性扫描（exclusive scan）：每个位置不包含自身
std::exclusive_scan(v.begin(), v.end(), out.begin(), 0);
// out = {0, 1, 3, 6, 10}
```

---

### GCD and LCM（最大公约数和最小公倍数）

```c++
#include <numeric>
#include <iostream>

std::cout << std::gcd(12, 8);   // 4
std::cout << std::lcm(12, 8);   // 24
std::cout << std::gcd(17, 5);   // 1（互质）
```

---

### std::not_fn

返回函数取反后的新可调用对象。类似 `std::not1`/`std::not2` 的现代替代，且不限于一元/二元谓词。

```c++
#include <functional>
#include <vector>
#include <algorithm>
#include <iostream>

auto is_even = [](int n) { return n % 2 == 0; };
auto is_odd = std::not_fn(is_even);

std::vector<int> v = {1, 2, 3, 4, 5};
std::copy_if(v.begin(), v.end(),
             std::ostream_iterator<int>(std::cout, " "),
             is_odd);  // 1 3 5
```

#### 简化代码的对比

```c++
// C++17 之前 —— 需要手写 lambda 取反
std::copy_if(v.begin(), v.end(), out,
             [](int n) { return n % 2 != 0; });

// C++17 —— 用现有函数取反
std::copy_if(v.begin(), v.end(), out,
             std::not_fn(is_even));
```

---

### String Conversion to/from Numbers（高性能数字-字符串转换）

C++17 提供了 `std::to_chars` 和 `std::from_chars`——**无异常、无动态分配、最快**的数字字符串转换。

```c++
#include <charconv>
#include <iostream>
#include <string>

// 整数 → 字符串
std::string str(10, ' ');
auto [ptr, ec] = std::to_chars(str.data(),
                                str.data() + str.size(),
                                12345);
*ptr = '\0';  // 手动终止
std::cout << str;  // "12345"

// 字符串 → 整数
int value;
auto [ptr2, ec2] = std::from_chars(str.data(),
                                    str.data() + str.size(),
                                    value);
if (ec2 == std::errc{}) {
    std::cout << value;  // 12345
}
```

#### vs 传统方式

```c++
// C 风格：sprintf —— 不安全，慢
char buf[20];
sprintf(buf, "%d", 42);

// C++ 风格：std::to_string —— 有动态分配
std::string s = std::to_string(42);

// C++17：to_chars —— 无分配、无异常、最快
char buf[20];
auto [p, e] = std::to_chars(buf, buf + 20, 42);
```

---

### Rounding Functions for Chrono（时间舍入函数）

C++17 为 `std::chrono::duration` 和 `time_point` 提供了 `abs`、`round`、`ceil`、`floor` 函数。

```c++
#include <chrono>
#include <iostream>
using namespace std::chrono;

milliseconds a{-5500};

// 绝对值
auto d = abs(a);  // 5500ms

// 向下取整到秒
auto s = floor<seconds>(d);  // 5s

// 向上取整到秒
auto c = ceil<seconds>(d);   // 6s

// 四舍五入到秒
auto r = round<seconds>(d);  // 6s
```

**实用场景：超时时间格式化**

```c++
#include <chrono>
#include <iostream>
using namespace std::chrono;

std::string format_time(milliseconds ms) {
    auto sec = duration_cast<seconds>(ms);
    auto remain = ms - sec;
    return std::to_string(sec.count()) + "s " +
           std::to_string(remain.count()) + "ms";
}

// 或更精确：秒四舍五入
auto rounded = round<seconds>(milliseconds{5500});  // 6s
```

---

### C++20 标准库常用新特性（精选）

> 从 C++20 标准库新增中挑选**常用、能精简代码**的实用特性，全部标注 **C++20**。每个特性都给出"之前怎么写 vs 现在怎么写"的精简对比。

---

### Text Formatting（文本格式化，std::format，C++20）

### 特性说明

`std::format` 提供编译时检查的字符串格式化，替代 C 的 `sprintf` 和 C++ 的 `ostringstream` 拼接。它是 C++20 最受欢迎的库特性之一，也是后来 C++23 `std::print` 的基础。

```cpp
#include <format>
std::format("{}", 123);                 // "123"
std::format("{} {}", "数字:", 123);      // "数字: 123"
std::format("{:.2f}", 3.14159);          // "3.14"（保留两位小数）
std::format("{:>5}", 42);                // "   42"（右对齐宽度5）
```

### 为什么需要？之前怎么做

| 旧方式 | 问题 | 新方式 |
|:---|:---|:---|
| `sprintf(buf, "%d", x)` | 缓冲区溢出风险、类型不安全 | `std::format("{}", x)` 编译期检查 |
| `std::to_string(x) + " " + std::to_string(y)` | 冗长、易错 | `std::format("{} {}", x, y)` |
| `ostringstream` | 样板多（`ostringstream oss; oss << ...; oss.str()`） | 一行搞定 |
| `snprintf` | 手动算长度、易截断 | 自动分配 |

```cpp
// C++17 及之前 —— 拼接多个值
std::ostringstream oss;
oss << "x=" << x << ", y=" << y;
auto s = oss.str();

// C++20 —— 一行
auto s = std::format("x={}, y={}", x, y);
```

### 最佳实践

1. **编译期检查格式串**：`{}` 与参数数量/类型不匹配在编译期报错
2. **格式化说明符**：`{:d}` 整数、`{:.2f}` 浮点、`{:x}` 十六进制、`{:>10}` 右对齐
3. **比 `sprintf` 安全**：无缓冲区溢出，自动管理内存
4. **C++23 有 `std::print`**：直接 `std::print("{}", x)` 输出到 `stdout`，无需再 `std::format` + `cout`

---

### Uniform Container Erasure（统一容器擦除，std::erase / erase_if，C++20）

### 特性说明

为所有 STL 容器提供 `std::erase` 和 `std::erase_if`，**一行删除**符合条件的元素。这是本批最"精简代码"的特性之一。

```c++
#include <vector>
std::vector v{0, 1, 0, 2, 0, 3};

std::erase(v, 0);   // 删除所有等于 0 的元素 → v == {1, 2, 3}
std::erase_if(v, [](int n) { return n == 0; });  // 按条件删除
```

### 为什么需要？之前怎么做

C++17 及之前要"删除所有满足条件的元素"，必须用著名的 **erase-remove 惯用法**：

```cpp
// C++17 及之前 —— erase-remove 惯用法，反直觉、易错
v.erase(std::remove(v.begin(), v.end(), 0), v.end());

// C++20 —— 一行，可读性翻倍
std::erase(v, 0);
```

| 容器类型 | 之前（erase-remove） | 现在 |
|:---|:---|:---|
| `vector`/`deque`/`string` | `v.erase(std::remove(...), v.end())` | `std::erase(v, x)` |
| `list`/`forward_list` | `v.remove(x)`（成员函数） | `std::erase(v, x)` |
| `map`/`set` 等关联容器 | 手写循环遍历 + `erase(it++)` | `std::erase_if(m, pred)` |

**为什么值得用**：erase-remove 惯用法的三个参数（迭代器对 + 值）反直觉且容易写错；`std::erase` 只需容器 + 值，`std::erase_if` 只需容器 + 谓词，语义直白。

### 使用场景

```cpp
// 删除所有偶数
std::erase_if(v, [](int n) { return n % 2 == 0; });

// 删除 map 中所有值小于 10 的条目
std::erase_if(m, [](const auto& kv) { return kv.second < 10; });
```

---

### Starts_with and Ends_with（字符串前后缀判断，C++20）

### 特性说明

字符串（`std::string`）和字符串视图（`std::string_view`）新增 `starts_with` / `ends_with` 成员函数，**一行判断**是否以某字符串开头/结尾。

```c++
std::string str = "foobar";
str.starts_with("foo");   // true
str.ends_with("baz");     // false
str.ends_with("bar");     // true
```

### 为什么需要？之前怎么做

```cpp
// C++17 及之前 —— 冗长且易错
str.rfind("bar", 0) == 0;                    // starts_with（rfind 技巧）
str.size() >= 3 && str.compare(str.size()-3, 3, "bar") == 0;  // ends_with

// C++20 —— 一行、语义直白
str.starts_with("foo");
str.ends_with("bar");
```

**应用场景**：判断文件扩展名、URL 协议前缀、删除行 `#` 注释、路径匹配等高频操作。

```cpp
bool is_markdown(const std::string& f) {
    return f.ends_with(".md") || f.ends_with(".markdown");
}
if (line.starts_with("# ")) { /* 是标题 */ }
```

---

### Check if Associative Container Has Element（关联容器查找，contains，C++20）

### 特性说明

`map`、`set` 等关联容器新增 `contains` 成员函数，**一行判断**是否包含某键，替代啰嗦的 `find() != end()`。

```c++
#include <map>
std::map<int, char> m {{1, 'a'}, {2, 'b'}};
m.contains(2);    // true
m.contains(123);  // false
```

### 为什么需要？之前怎么做

```cpp
// C++17 及之前 —— find() != end() 样板多且易读性差
if (m.find(key) != m.end()) { /* 存在 */ }

// C++20 —— 一行
if (m.contains(key)) { /* 存在 */ }
```

**关键优点**：`contains` 明确表达"只关心是否存在"的意图，且**不会**像 `find` 那样返回迭代器、暗示你后续要用它——语义更干净。

```cpp
// 应用：查缓存、查黑白名单
if (blacklist.contains(ip)) { reject(ip); }
if (cache.contains(key)) { return cache[key]; }  // 可再配合 operator[] 或 at()
```

---

### std::span（连续内存视图，C++20）

### 特性说明

`std::span` 是**非拥有**的连续内存视图——只持有"指针 + 长度"，零拷贝访问。它统一了数组、`vector`、`array` 等连续容器的接口，替代 C 风格的"指针 + 长度"参数。

```c++
#include <span>
void print_ints(std::span<const int> ints) {
    for (const auto n : ints) { std::cout << n << " "; }
}

print_ints(std::vector{1, 2, 3});           // ✅ 传 vector
print_ints(std::array<int, 5>{1,2,3,4,5});  // ✅ 传 array
int arr[] = {1, 2, 3};
print_ints(arr);                            // ✅ 传 C 数组
```

### 为什么需要？之前怎么做

```cpp
// C++17 及之前 —— 三种容器要写三个重载，或用"指针+长度"
void print(const std::vector<int>& v);
void print(const int* data, size_t size);   // 指针+长度：容易丢失大小信息

// C++20 —— 一个 span 通吃所有连续容器
void print(std::span<const int> ints);
```

| 特点 | 说明 |
|:---|:---|
| 非拥有 | 不管理生命周期，只是"看"别人内存 |
| 边界安全 | `span` 携带大小，可 `.size()`、`.empty()`、迭代器 |
| 零拷贝 | 传递时只复制指针+长度 |
| 适用 | 函数参数（统一接口）、子区间 `span.first(n)` / `.subspan()` |

**注意**：`span` 不拥有数据，被引用容器的生命周期必须长于 `span`（同 `string_view` 的悬垂风险）。

---

### Safe Integral Comparison（安全整数比较，C++20）

### 特性说明

`<utility>` 提供 `std::cmp_*` 系列函数，安全比较整数（包括不同类型的整数），**杜绝符号/隐式转换坑**。

```cpp
#include <utility>
-1 > 0U;                    // == true（⚠️ C++ 经典坑：-1 被转成无符号）
std::cmp_greater(-1, 0U);   // == false（✅ 正确）

std::cmp_equal(0U, 0);            // true
std::cmp_less_equal(-1, 1U);      // true
std::cmp_less(-1U, 0);            // true（正确按数学含义比较）
```

### 为什么需要？之前怎么做

```cpp
// C++17 及之前 —— 隐式转换坑
int n = -1;
unsigned u = 10;
bool bad = n < u;   // ❌ n 被转成 unsigned 变成巨大数，结果错误！

// C++20 —— 类型安全
bool ok = std::cmp_less(n, u);   // ✅ 正确比较，-1 < 10
```

**提供的函数**：`cmp_equal`、`cmp_not_equal`、`cmp_less`、`cmp_greater`、`cmp_less_equal`、`cmp_greater_equal`。只要两个操作数都是整数类型即可安全比较（可混合符号、混合宽度）。

**应用场景**：遍历索引与 `size()`（返回 `size_t` 无符号）比较、用户输入与边界比较等极易踩坑的场景。

---

### std::bit_cast（安全位重解释，C++20）

### 特性说明

`std::bit_cast` 提供**类型安全**的位级重解释——把对象从一种类型按位重新解释为另一种类型，替代不安全的 `reinterpret_cast` 和 C 风格转换。C++20 同时保证了它可在编译期求值（constexpr）。

```c++
#include <bit>
float f = 123.0f;
int i = std::bit_cast<int>(f);   // 把 float 的位按 int 解释
```

### 为什么需要？之前怎么做

```cpp
// C++17 及之前 —— reinterpret_cast：别名规则违规 → 未定义行为！
float f = 123.0f;
int i = *reinterpret_cast<int*>(&f);   // ❌ 违反 strict aliasing，UB

// 或 memcpy（安全但啰嗦）
int j;
static_assert(sizeof(f) == sizeof(j));
std::memcpy(&j, &f, sizeof(f));        // ✅ 安全但样板多

// C++20 —— 一行、安全、constexpr
int k = std::bit_cast<int>(f);
```

**关键要求**：源类型和目标类型大小相同、都是可平凡复制（trivially copyable）类型，否则编译错误。`std::bit_cast` 的语义等价于 `memcpy`（安全、无别名违规），但更简洁且能在编译期用。

**应用**：浮点取位（快速比较/解析 IEEE754）、类型擦除、网络字节序处理、实现 `std::chrono` 底层转换等。

---

### std::midpoint（安全中点，C++20）

### 特性说明

`std::midpoint(a, b)` 安全计算两个整数的中点，**无溢出风险**；也支持浮点数和指针。

```c++
#include <numeric>
std::midpoint(1, 3);   // 2
std::midpoint(2, 4);   // 3
std::midpoint(10, 20); // 15
```

### 为什么需要？之前怎么做

```cpp
// 朴素写法 —— 可能溢出！
int mid = (a + b) / 2;   // ❌ 若 a、b 都很大，(a+b) 溢出 int

// 改进写法 —— 但要手动处理符号，容易出错
int mid = a + (b - a) / 2;   // b-a 仍可能溢出（一正一负）

// C++20 —— 一行，标准保证无溢出
int mid = std::midpoint(a, b);
```

**应用场景**：二分查找的中间索引计算（经典溢出点！）、几何中点、区间插值。`std::midpoint(a, b)` 内部用无溢出的位运算/代数恒等式实现。

---

### std::to_array（数组转换，C++20）

### 特性说明

`std::to_array` 把 C 风格数组/"类数组"对象转换为 `std::array`，让"列表初始化即得 `std::array`"。

```c++
#include <array>
auto a = std::to_array("foo");          // std::array<char, 4>（含 '\0'）
auto b = std::to_array<int>({1, 2, 3}); // std::array<int, 3>
```

### 为什么需要？之前怎么做

```cpp
// C++17 及之前 —— 字面量列表无法直接初始化 std::array
// std::array<int, 3> a = {1, 2, 3};   // 要手写类型和大小
// auto a = std::array{1, 2, 3};       // CTAD 可以，但字符串不行

// C++20 —— 从字符串/数组/字面量直接转
auto s = std::to_array("hi");          // array<char, 3>
auto v = std::to_array({1, 2, 3});     // array<int, 3>
```

**应用场景**：需要把字符串字面量或 C 数组转为 `std::array`（以便用 `.size()`、迭代器、作为模板参数）时。注意 `to_array` 会**拷贝**数据（非视图）。

---

### std::bind_front（绑定前 N 个参数，C++20）

### 特性说明

`std::bind_front` 把可调用对象的前 N 个参数绑定为固定值，生成新可调用对象——比 `std::bind` 更简单直观。

```c++
#include <functional>
const auto f = [](int a, int b, int c) { return a + b + c; };

const auto g = std::bind_front(f, 1, 1);  // 绑定前两个参数
g(1);   // == 3（等价 f(1, 1, 1)）
```

### 为什么需要？vs std::bind

```cpp
// C++11 —— std::bind：占位符 _1、_2 反直觉且冗长
using namespace std::placeholders;
auto g = std::bind(f, 1, 1, _1);
g(1);   // == 3

// C++20 —— std::bind_front：无需占位符，剩余参数自动接到末尾
auto g = std::bind_front(f, 1, 1);
g(1);   // == 3
```

| | `std::bind` | `std::bind_front` |
|:---|:---|:---|
| 占位符 | 需要 `_1`、`_2` | ❌ 不需要 |
| 绑定位置 | 任意位置 | 只绑定**前 N 个** |
| 语义 | 复杂、易错 | 简单直白 |

**应用场景**：给成员函数/回调预置配置参数、部分应用函数、给 STL 算法绑定固定参数。

---

### std::jthread（可中断线程，C++20）

### 特性说明

`std::jthread` 是 `std::thread` 的增强版：**析构时自动 `join`**（不会 `terminate`），并内置**协作式取消**（stop token）机制。

```cpp
#include <thread>
std::jthread t{
    [](std::stop_token stoken) {
        while (!stoken.stop_requested()) {
            std::this_thread::sleep_for(1s);
            // 正常工作...
        }
    }
};
t.request_stop();   // 请求线程停止（协作式）
// t 析构时自动 join，不会崩溃
```

### 为什么需要？vs std::thread

```cpp
// std::thread —— 忘记 join/detach 析构时直接 terminate 崩溃！
std::thread t([]{ /* ... */ });
// 忘记 t.join() → t 析构 → std::terminate！

// std::jthread —— 析构自动 join，永远不会因忘记 join 而崩溃
std::jthread t([]{ /* ... */ });
// 无需显式 join
```

| | `std::thread` | `std::jthread` |
|:---|:---|:---|
| 析构行为 | 未 join/detach → `terminate` | **自动 join**，安全 |
| 取消机制 | ❌ 无 | ✅ `std::stop_token` |
| 请求停止 | 无 | `request_stop()` |
| 适合 | 需要完全控制 | 大多数常规场景（更安全） |

**应用场景**：后台任务、可取消的工作循环、资源清理（RAII 自动 join）。配合 `std::stop_token`/`std::stop_callback` 实现协作式取消，无需手动管理线程生命周期。

---

### Math Constants（数学常数，std::numbers，C++20）

### 特性说明

`<numbers>` 头文件提供标准数学常数，类型可指定（默认 `double`）：

```c++
#include <numbers>
std::numbers::pi;          // 3.14159...
std::numbers::e;           // 2.71828...
std::numbers::pi_v<float>; // float 精度的 π
```

### 为什么需要？之前怎么做

```cpp
// 之前 —— 手写宏/常量，精度不统一、易错
#define PI 3.14159                  // ❌ 精度低
const double PI = 3.141592653589793; // ✅ 但每处都要自己定义

// C++20 —— 标准提供，类型可指定
std::numbers::pi;                    // double
std::numbers::pi_v<long double>;     // 高精度
```

**提供的常数**：`e`、`log2e`、`log10e`、`pi`、`inv_pi`、`inv_sqrtpi`、`ln2`、`ln10`、`sqrt2`、`sqrt3`、`inv_sqrt3`、`egamma`、`phi` 等。**类型安全**：用 `_v` 后缀指定 `float`/`double`/`long double`，避免隐式转换精度损失。

---

### Bit Operation（位操作，std::popcount 等，C++20）

### 特性说明

`<bit>` 头文件提供无分支的位操作函数，包括 `popcount`（统计 1 的个数）、`rotl`、`rotr`、`countl_zero`、`countr_zero` 等。

```c++
#include <bit>
std::popcount(0u);            // 0
std::popcount(1u);            // 1
std::popcount(0b1111'0000u);  // 4

std::rotl(0b0000'0001u, 1);   // 0b0000'0010（循环左移）
std::countl_zero(0b0001'0000u); // 3（前导 0 个数）
```

### 为什么需要？之前怎么做

```cpp
// 之前 —— 手写循环统计 1 的个数
int popcount(unsigned x) {
    int cnt = 0;
    while (x) { x &= x - 1; ++cnt; }   // Brian Kernighan 算法
    return cnt;
}

// C++20 —— 一行，编译期可用（constexpr）
std::popcount(x);
```

**关键优点**：标准保证这些函数是 `constexpr`（编译期可用）、且实现映射到 CPU 的 `popcnt` 等指令（硬件加速）。应用场景：位图/布隆过滤器、奇偶校验、哈希优化、网络协议位解析。

---

### C++23 标准库常用新特性（精选）

> 从 C++23 标准库新增中挑选**常用、能精简代码**的实用特性，全部标注 **C++23**。每个特性都给出"之前怎么写 vs 现在怎么写"的精简对比。

---

### 字符串和字符串视图的 `contains`（C++23）

### 特性说明

`std::string` 和 `std::string_view` 新增 `contains` 成员函数，**一行判断**是否包含子字符串——比 C++20 的 `find() != npos` 语义直白得多。

```c++
std::string{"foobarbaz"}.contains("bar");  // == true
std::string{"foobarbaz"}.contains("bat");  // == false
std::string_view{"abc"}.contains('b');     // == true（也可查单个字符）
```

### 为什么需要？之前怎么做

```cpp
// C++17 及之前 —— find() != npos，啰嗦
if (str.find("bar") != std::string::npos) { /* 包含 */ }

// C++23 —— 一行
if (str.contains("bar")) { /* 包含 */ }
```

**精简对比**：`contains` 把"查找 + 判空"压缩成一个语义明确的调用，杜绝 `npos` 拼写错误。

---

### std::to_underlying（枚举转底层类型，C++23）

### 特性说明

`std::to_underlying` 把枚举值安全地转换为其底层整数类型，替代手写 `static_cast`。

```c++
#include <utility>
enum class MyEnum : int { A = 1, B, C };

std::to_underlying(MyEnum::A);  // == 1（int）
std::to_underlying(MyEnum::C);  // == 3
```

### 为什么需要？之前怎么做

```cpp
// 之前 —— 必须手写 static_cast，且要重复写底层类型
static_cast<int>(MyEnum::A);
// 或 (int)MyEnum::A   —— C 风格，类型不安全

// C++23 —— 一行，自动推导底层类型
std::to_underlying(MyEnum::A);
```

**关键优点**：不用管底层类型是 `int`/`short`/`char`，`std::to_underlying` 自动推导。应用场景：枚举序列化、存储到 `int` 容器、switch 与数值互转。

---

### Monadic Operations for std::optional（std::optional 单子操作，C++23）

### 特性说明

`std::optional` 新增 `and_then`、`transform`、`or_else` 三个**单子操作**，把"判空 + 分支处理"链式化，避免层层 `if` 嵌套。

```c++
std::optional<int> parse_int(const std::string&);
std::optional<int> ensure_non_negative(int);
std::optional<double> default_value_or_empty(double);

std::optional<double> stringToSqrtDouble(const std::string& input) {
  return parse_int(input)                // optional<int>
    .and_then(ensure_non_negative)       // 有值才继续（返回 optional）
    .transform([](int x) { return std::sqrt(x); })  // 有值就转换
    .or_else(default_value_or_empty);    // 无值给默认
}
```

### 为什么需要？之前怎么做

```cpp
// C++23 之前 —— 层层 if 判空，缩进地狱
std::optional<double> f(const std::string& input) {
    auto a = parse_int(input);
    if (!a) return std::nullopt;
    auto b = ensure_non_negative(*a);
    if (!b) return std::nullopt;
    double v = std::sqrt(*b);
    if (!std::isfinite(v)) return std::nullopt;
    return v;
}

// C++23 —— 链式，无嵌套
std::optional<double> f(const std::string& input) {
    return parse_int(input)
        .and_then(ensure_non_negative)
        .transform([](int x) { return std::sqrt(x); });
}
```

| 方法 | 作用 | 返回 |
|:---|:---|:---|
| `transform(f)` | 有值 → 用 `f` 转换值 | `optional<U>` |
| `and_then(f)` | 有值 → 调 `f`（`f` 返回 optional） | `optional<U>` |
| `or_else(f)` | 无值 → 调 `f` 给默认值 | `optional<T>` |

> 顺带说明：C++23 还给 `std::expected` 提供同样的 `and_then`/`transform`/`or_else` 单子操作。

---

### std::expected（错误处理，C++23）

### 特性说明

`std::expected<T, E>` 同时包含**值**和**错误**（二选一），是**不使用异常**的错误处理首选——比 `optional` 多了错误类型信息。

```c++
#include <expected>

enum class StringToSqrtDoubleError { ParseError, NegativeNumber };

std::expected<int, StringToSqrtDoubleError> parse_int(const std::string&);

std::expected<double, StringToSqrtDoubleError> stringToSqrtDouble(const std::string& input) {
    auto parsed = parse_int(input);
    if (!parsed) return parsed;   // 透传错误

    auto parsedInt = *parsed;
    if (parsedInt < 0) return std::unexpected(StringToSqrtDoubleError::NegativeNumber);

    return std::sqrt(parsedInt);  // 正常返回值
}
```

### 为什么需要？vs optional / 异常

| 方式 | 能否携带错误信息 | 开销 | 适用 |
|:---|:---|:---|:---|
| `std::optional` | ❌ 只能"有没有" | 低 | 简单缺失场景 |
| 异常 | ✅ 丰富 | 高（栈展开） | 错误很"例外"时 |
| **`std::expected`** | ✅ 值+错误 | 低 | 错误是"正常流程"时 |

```cpp
// C++23 之前 —— optional 丢失错误原因，或异常性能不友好
// C++23 —— expected 明确表达"成功返回 T / 失败返回 E"
```

**关键用法**：
- 正常值：直接 `return value;`
- 错误值：`return std::unexpected(error);`
- 检查：`if (res)` 判断成功；`*res` / `res.value()` 取值；`res.error()` 取错误
- 单子操作：`and_then`/`transform`/`or_else` 链式处理（同 optional）

**应用场景**：解析器、配置文件加载、数值转换、数据库/网络调用等"错误是常态"的场景。

---

### std::unreachable（不可达标记，C++23）

### 特性说明

`std::unreachable()` 显式告诉编译器"这段代码不可达"。如果代码**真的被执行到**，是未定义行为——但能帮助编译器优化。

```c++
#include <utility>
enum class MyEnum { A, B, C };

int convertMyEnumToInt(MyEnum e) {
    switch (e) {
        case MyEnum::A: return 0;
        case MyEnum::B: return 1;
        case MyEnum::C: return 2;
        default: std::unreachable();   // 枚举值已穷尽，这里理论上到不了
    }
}
```

### 为什么需要？之前怎么做

```cpp
// 之前 —— 三种绕法都有缺陷
// 方式1：return 任意值 —— 误导、掩盖 bug
default: return -1;
// 方式2：assert(false) —— 调试有用，但 release 下保留代码
default: assert(false);
// 方式3：空 default —— 编译器可能警告"控制流到达函数末尾"

// C++23 —— 明确声明"不可达"，编译器可优化掉此分支
default: std::unreachable();
```

**关键优点**：
1. **编译器优化**：知道分支不可达后可移除相关代码
2. **意图明确**：`std::unreachable()` 自文档化"这里不该被执行"
3. **替代 `__builtin_unreachable`**：标准、可移植

> **警告**：如果 `std::unreachable()` 被执行到 → 未定义行为。只在**你能证明不可达**时使用（如穷尽枚举、前置条件保证）。

---

### Stacktrace Library（堆栈跟踪，C++23）

### 特性说明

`<stacktrace>` 提供标准化的堆栈跟踪——获取当前调用栈的条目（源文件、行号、函数名），用于调试和错误报告。

```c++
#include <print>
#include <stacktrace>

int main() {
    std::println("{}", std::stacktrace::current());
}
```

输出示例（Linux）：
```
  0#  main at /app/example.cpp:5 [0x5ee42e3db747]
  1#  <unknown> [0x76e76dc29d8f]
  2#  __libc_start_main [0x76e76dc29e3f]
  3#  _start [0x5ee42e3db644]
```

### 为什么需要？之前怎么做

```cpp
// 之前 —— 平台相关，各不相同
// Linux: backtrace() + backtrace_symbols()（<execinfo.h>）
// Windows: CaptureStackBackTrace()（<windows.h>）
// 每种平台写一套，不可移植

// C++23 —— 标准跨平台
#include <stacktrace>
auto st = std::stacktrace::current();
```

**关键类型**：
- `std::stacktrace_entry`：单个栈帧（源文件、行号、描述）
- `std::stacktrace`：整个调用栈（`current()` 获取，可遍历）

**应用场景**：异常日志附加调用栈、崩溃报告、调试定位、诊断工具。注意：生产环境可能因符号表/优化影响行号准确性。

---

## C++ 语言特性

### explicit 构造函数

#### 特性说明

C++11 允许将单参数构造函数（以及更多参数的构造函数）标记为 `explicit`，禁止编译器使用该构造函数进行**隐式类型转换**。

```c++
struct Widget {
    explicit Widget(int) {}  // explicit 构造函数
};

Widget w1{42};     // ✅ 直接初始化——显式调用，可以
// Widget w2 = 42; // ❌ 拷贝初始化——试图隐式转换 int→Widget，不行
Widget w3 = Widget{42};  // ✅ 显式转换，可以
Widget w4(42);     // ✅ 直接初始化，可以
Widget w5 = (Widget)42; // ✅ C 风格显式转换，可以
Widget w6 = static_cast<Widget>(42); // ✅ C++ 风格显式转换，可以
```

#### 为什么两种初始化方式有区别？

关键在于**拷贝初始化**（`=` 语法）会走"隐式转换序列"——编译器先尝试把右值**隐式转换**为目标类型，再用这个临时对象拷贝构造。`explicit` 构造函数不能参与隐式转换，所以被禁止。

```
Widget w{42};    ← 直接列表初始化：直接调用构造函数，explicit 与否都可以
Widget w = 42;   ← 拷贝初始化：先尝试隐式转换 int → Widget，
                    explicit 构造函数不能参与隐式转换，所以失败
```

#### 不同初始化方式对比

| 初始化形式 | 示例 | `explicit` 构造函数 | 非 `explicit` 构造函数 |
|:---|:---|:---:|:---:|
| 直接列表初始化 | `T obj{args}` | ✅ | ✅ |
| 直接初始化 | `T obj(args)` | ✅ | ✅ |
| 拷贝初始化 | `T obj = arg` | ❌ | ✅ |
| 复制列表初始化 | `T obj = {arg}` | ❌ | ✅ |
| static_cast | `static_cast<T>(arg)` | ✅ | ✅ |

#### 使用场景

**场景 1：防止意外的隐式转换**

```c++
struct URL {
    explicit URL(const std::string& path) { /* ... */ }
};

void fetch(const URL& url);

fetch("https://example.com");  // ❌ 编译错误！不会把 const char* 隐式转成 URL
fetch(URL{"https://example.com"}); // ✅ 显式构造，意图明确
```

没有 `explicit` 的话，`fetch("https://example.com")` 会默默创建临时 `URL` 对象，可能隐藏 bug。

**场景 2：布尔语境的安全用法**

`explicit operator bool()` 是最常见的显式转换函数：

```c++
struct FilePtr {
    FILE* fp;
    
    explicit operator bool() const {
        return fp != nullptr;
    }
};

FilePtr f{fopen("test.txt", "r")};

if (f) {           // ✅ 布尔语境允许 explicit 转换
    // 使用文件...
}

// bool b = f;     // ❌ 拷贝初始化，不允许
bool b(f);         // ✅ 直接初始化，允许
```

#### explicit 转换函数（C++11）

C++11 也允许转换函数（`operator T()`）标记为 `explicit`，与 `explicit` 构造函数的规则对称：

| 语境 | 示例 | 允许 `explicit` 转换？ |
|:---|:---|:---:|
| 布尔语境 | `if (obj)`、`while (obj)`、`!obj`、`obj && val` | ✅ 允许 |
| 直接初始化 | `bool b(obj)`、`bool b{obj}` | ✅ 允许 |
| 显式转换 | `static_cast<bool>(obj)` | ✅ 允许 |
| 拷贝初始化 | `bool b = obj` | ❌ 不允许 |
| 函数参数传递 | `void f(bool); f(obj);` | ❌ 不允许 |

```c++
struct Ptr {
    explicit operator bool() const { return ptr != nullptr; }
    void* ptr;
};

Ptr p{get_data()};

if (p) {           // ✅ 布尔语境
    // use p
}
bool valid(p);      // ✅ 直接初始化
bool v = p;         // ❌ 拷贝初始化
```

---

### Lambda Capture Initializer 扩展

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

---

### C++20：弃用隐式捕获 this（版本演进）

#### 演进背景

lambda 捕获 `this` 的方式在三个版本中逐步收紧：

| 版本 | 捕获 `this` 的方式 | 状态 |
|:---|:---|:---|
| C++11 | `[=]` 会**隐式**捕获 `this` | 默认行为（有隐患） |
| C++14 | `[this]`、`[*this]` 显式捕获；`[=]` 仍隐式捕获 | 补充显式写法 |
| **C++20** | `[=]` 隐式捕获 `this` **弃用**；推荐 `[=, this]` 或 `[=, *this]` | ⚠️ 弃用（deprecated） |

> **先澄清：废弃的是「`[=]` 隐式捕获 `this`」这一副作用，不是 `[=]` 语法本身。**
>
> `[=]` 按值捕获普通局部变量的行为**从未废弃、一直有效**。C++20 只是收紧了它的一个隐性副作用——以前 `[=]` 会"顺手"把 `this` 指针也隐式捕获进来，现在这个行为被标记为弃用（C++23 起直接报错）。所以：
>
> - `[=]` 捕获局部变量 → ✅ 正常使用，不受影响
> - `[=]` 顺带隐式捕获 `this` → ⚠️ C++20 弃用，须显式写 `[=, this]` 或 `[=, *this]`

**为什么弃用**：`[=]` 的字面意思是"按值捕获"，但它对 `this` 却做的是**按引用**（捕获指针）。这与直觉相悖，极易误导——你以为 lambda 持有了所有需要的数据的副本，实际它只持有一个指向外部对象的指针，**异步场景下对象销毁就会悬垂**：

```c++
struct Widget {
    int value;
    auto getCallback() {
        // [=] 看起来是"按值捕获"，但成员 value 实际上是通过 this 指针访问的！
        return [=] { return value; };  // 等价于 return [this]{ return value; };
    }
};

// 问题场景：回调被异步调用，而 Widget 已经销毁
Widget* w = new Widget{42};
auto cb = w->getCallback();   // lambda 捕获了 this（指向 w）
delete w;                     // w 销毁
cb();                         // ❌ 悬垂 this，未定义行为！
```

`[=]` 捕获的是 `this` **指针**（按值复制了指针本身），但指针指向的对象不会因为 lambda 存在而存活。这是 C++11 lambda 的一个经典陷阱。

#### C++20 的解决方案

**1. 显式按引用捕获：`[=, this]`**

```c++
struct int_value {
  int n = 0;
  auto getter_fn() {
    return [=, this]() { return n; };  // 明确：n 通过 this 指针访问
  }
};
```

**2. 显式按值捕获对象副本：`[=, *this]`**（推荐用于异步场景）

```c++
struct int_value {
  int n = 0;
  auto getter_fn() {
    // 捕获 this 指向对象的【副本】，lambda 独立拥有这份数据
    return [=, *this]() { return n; };
    // 即使原对象销毁，lambda 里的副本依然有效！
  }
};
```

这才是真正"按值捕获"了对象，与 `[=]` 的直觉一致。

#### 三种写法的对比

| 写法 | 捕获了什么 | 对象销毁后 | 适用场景 |
|:---|:---|:---|:---|
| `[=]`（C++20 弃用） | `this` 指针 | ❌ 悬垂，UB | 不推荐 |
| `[=, this]` | `this` 指针（显式） | ❌ 悬垂，UB | 同步调用、生命周期有保证 |
| `[=, *this]` | 对象**副本** | ✅ 安全 | 异步回调、需要独立数据 |

```c++
struct int_value {
  int n = 0;
  auto getter_fn() {
    return [=, *this]() { return n; };  // 捕获 n 的副本
  }
};
int_value v{42};
auto fn = v.getter_fn();
v.n = 100;      // 修改原对象
fn();           // 返回 42（用的是副本，不受影响）
```

#### 最佳实践

1. **异步/延迟回调**一律用 `[=, *this]`——避免悬垂指针
2. **同步调用且生命周期明确**用 `[=, this]`
3. **避免** `[=]` 单独使用访问成员变量（会触发弃用警告）
4. 编译器会给出 `-Wdeprecated` 警告提示改为显式捕获

---

### 模板相关特性总览（版本演进）

模板是 C++ 元编程的基石，相关特性从 C++14 到 C++20 逐步增强。本家族把散落在文档中的模板特性**按版本串成一条演进时间线**，后续每一节都对应其中一行：

| 版本 | 特性 | 一句话总结 | 解决的问题 |
|:---|:---|:---|:---|
| **C++14** | `Variable Template`（变量模板） | 变量也可以模板化 | 类型相关的常量、单位换算、`_v` 类型萃取 |
| **C++14** | `Compile-Time Integer Sequence`（编译时整数序列） | 编译期整数列表 | 展开参数包，array↔tuple 转换的桥梁 |
| **C++17** | `Class Template Argument Deduction`（CTAD） | 类模板参数自动推导 | 告别 `std::pair<int,double>` 式类型冗余 |
| **C++17** | `Declaring Non-Type Template Parameters with Auto` | 非类型模板参数用 `auto` | 免写 `int`/`long` 类型，值即类型 |
| **C++17** | `Folding Expression`（折叠表达式） | 参数包批量运算 | 替代递归展开，一行 `(x + ...)` 求和 |
| **C++20** | `Class Types in Non-Type Template Parameters`（非类型模板参数中的类类型） | 类对象也可作模板参数 | 编译期配置对象、`FixedString` 字符串模板参数 |
| **C++20** | `Template Syntax for Lambdas`（模板 lambda） | lambda 显式模板参数 | 泛型 lambda 的类型可命名、可约束 |

**四条演进主线**：

```
① 参数包操作：  C++14 整数序列（展开桥梁） ──▶ C++17 折叠表达式（直接运算）
② 参数推导：    C++14 变量模板（值随类型变） ─▶ C++17 CTAD（类模板推导）
③ 非类型参数：  C++17 auto 标量 ─────────────▶ C++20 类类型（结构化对象 / 字符串）
④ lambda 模板化：C++14 泛型 lambda（auto 参数）─▶ C++20 模板 lambda（显式模板参数）
```

**一句话记忆**：模板参数能装的东西越来越多——**类型**（一直可以）→ **标量值**（C++17 `auto`）→ **结构化对象**（C++20 类类型）；对参数包的操作也越来越直接——**展开**（C++14 整数序列）→ **直接折叠运算**（C++17）。

#### 基础：类型模板参数 vs 非类型模板参数

**"模板参数"不只是类型**——它分为两类，这是理解整个模板家族的前提：

```c++
// ① 类型模板参数（Type Template Parameter）——最常见
template <typename T>   // T 是一个"类型"
void f(T x);

// ② 非类型模板参数（Non-Type Template Parameter）——是个"值"
template <int N>        // N 是一个"整数常量"，不是类型
void g() { /* ... */ }
```

| | 类型模板参数 | 非类型模板参数 |
|:---|:---|:---|
| 本质 | 一个**类型** | 一个**值**（编译期常量） |
| 语法 | `typename T` / `class T` | `int N`、`auto V`、`Foo f`（C++20） |
| 传参示例 | `f<int>(x)` | `g<5>()` |
| 值必须在编译期知道 | 不必 | **必须**（字面量/constexpr） |
| 在模板体内表现为 | 一种类型（可声明变量） | 一个常量（可当 `constexpr` 用） |

```c++
// 类型模板参数：同一个模板，T 变 → 实例化成不同函数
template <typename T>
T add_one(T x) { return x + 1; }
add_one<int>(10);      // T = int
add_one<double>(10.5); // T = double

// 非类型模板参数：同一个模板，N 变 → 实例化成不同函数
template <int N>
int repeat(int x) { return x * N; }
repeat<3>(10);   // N = 3  → 30
repeat<10>(10);  // N = 10 → 100
```

**关键区别**：前者"类型可变"，后者"值可变"。共同点是——**都会在编译期实例化成不同的实体**。

**为什么需要"值"当模板参数？** 因为有些东西必须在编译期确定，函数参数（运行期）做不到：

```c++
// ❌ 函数参数是运行期的，不能决定编译期的东西
void make_array(int N) { int arr[N]; }   // 可变长数组，C++ 不支持

// ✅ 非类型模板参数是编译期的，可以决定编译期的东西
template <int N>
void make_array() {
    int arr[N];                 // 编译期就确定了大小
    std::array<int, N> a;       // 标准库容器也要编译期大小
}
```

`std::bitset<N>`、`std::array<T, N>` 的 `N`、`std::get<I>(tuple)` 的 `I`、模板元编程中的 `if constexpr (N > 0)`——**全都需要编译期常量**，只能靠非类型模板参数提供。

**一句话总结**：类型模板参数是"装类型的盒子"（`T` 可以是 `int`、`double`、任何类）；非类型模板参数是"装值的盒子"（`N` 必须是编译期已知的常量）。模板参数能装的东西越来越多：**类型**（一直可以）→ **标量值**（C++17 `auto`）→ **结构化对象**（C++20 类类型）。

---

### Variable Template（变量模板）

### 特性说明

C++14 允许变量被模板化，本质上就是**带模板参数的变量**。普通的全局变量是固定的值，而变量模板可以针对不同的模板参数生成不同的值。

```c++
template<class T>
constexpr T pi = T(3.1415926535897932385);
```

这里 `pi` 是一个变量模板，当用 `pi<float>` 时得到 `float` 类型的 π，用 `pi<double>` 时得到 `double` 类型的 π。

### 使用场景

#### 场景 1：数学常量——避免类型转换错误

```c++
#include <iostream>

template<typename T>
constexpr T pi = T(3.1415926535897932384626433832795);

int main() {
    std::cout << pi<float> << std::endl;      // 3.14159（float 精度）
    std::cout << pi<double> << std::endl;     // 3.14159（double 精度）
    std::cout << pi<long double> << std::endl; // 3.14159（更高精度）

    // 以前的做法会有隐式转换问题：
    auto r1 = 2.0 * 3.14159;       // double
    auto r2 = 2.0f * 3.14159;      // float * double → double（可能不是你想要的结果）
    // 现在类型完全可控：
    auto r3 = 2.0f * pi<float>;    // 明确用 float
}
```

#### 场景 2：单位换算系数

```c++
#include <iostream>

template<typename T>
constexpr T meters_per_mile = T(1609.344);

template<typename T>
constexpr T pounds_per_kilogram = T(2.20462);

template<typename T>
constexpr T fahrenheit_to_celsius_offset = T(32);

int main() {
    double miles = 10.0;
    double km = miles * meters_per_mile<double> / 1000.0;
    std::cout << miles << " 英里 = " << km << " 公里" << std::endl;

    float weight_lb = 150.0f;
    float weight_kg = weight_lb / pounds_per_kilogram<float>;
    std::cout << weight_lb << " 磅 = " << weight_kg << " 公斤" << std::endl;
}
```

#### 场景 3：类型相关的默认值

```c++
#include <iostream>
#include <limits>
#include <string>

// 不同类型有不同的"无效值"
template<typename T>
constexpr T invalid_value = T(-1);  // 对整数类型有效

template<>
constexpr double invalid_value<double> = std::numeric_limits<double>::quiet_NaN();

template<>
constexpr const char* invalid_value<const char*> = "N/A";

int main() {
    int id = invalid_value<int>;              // -1
    double price = invalid_value<double>;     // NaN
    const char* name = invalid_value<const char*>; // "N/A"

    std::cout << "id: " << id << std::endl;
    std::cout << "price: " << price << std::endl;
    std::cout << "name: " << name << std::endl;
}
```

#### 场景 4：编译时计算不同精度的物理量

```c++
#include <iostream>
#include <cmath>

template<typename T>
constexpr T gravitational_acceleration = T(9.80665);  // 重力加速度 m/s²

// 根据距离计算自由落体时间
template<typename T>
constexpr T free_fall_time(T height) {
    return std::sqrt(2 * height / gravitational_acceleration<T>);
}

int main() {
    constexpr float t_float = free_fall_time(100.0f);
    constexpr double t_double = free_fall_time(100.0);
    std::cout << "float 精度: " << t_float << " 秒" << std::endl;
    std::cout << "double 精度: " << t_double << " 秒" << std::endl;
}
```

#### 场景 5：类型萃取辅助（Traits Helper）

```c++
#include <iostream>
#include <type_traits>

// 判断类型是否为"整数族"（包括 char、bool 等）
template<typename T>
constexpr bool is_integer_family_v =
    std::is_integral_v<T> || std::is_same_v<T, char> || std::is_same_v<T, bool>;

// 结合 SFINAE 使用
template<typename T>
std::enable_if_t<is_integer_family_v<T>, void> print_value(T v) {
    std::cout << "整数: " << v << std::endl;
}

template<typename T>
std::enable_if_t<!is_integer_family_v<T>, void> print_value(T v) {
    std::cout << "非整数: " << v << std::endl;
}

int main() {
    print_value(42);       // 整数: 42
    print_value(3.14);     // 非整数: 3.14
    print_value('A');      // 整数: A（char 也被视为整数族）
    print_value(true);     // 整数: 1
}
```

> **小结**：变量模板让**类型相关的常量**有了统一的定义方式。C++17 中引入的大量 `_v` 后缀类型萃取（如 `is_same_v<T, U>`）正是基于此特性。

---

### Compile-Time Integer Sequence（编译时整数序列）

### 特性说明

`std::integer_sequence` 是 C++14 引入的一个工具，用于在**编译期**表示一个整数序列，如 `0, 1, 2, 3, 4`。它的核心价值在于**展开参数包**——当你有一个数组、tuple 或其他"打包"的数据，需要把它展开传递给一个可变参数函数时，整数序列就是桥梁。

核心类型层次：

| 类型 | 含义 |
|:---|:---|
| `std::integer_sequence<T, I...>` | 元素类型为 `T` 的序列 `I...`（如 `int, 0, 1, 2, 3`） |
| `std::index_sequence<I...>` | `std::integer_sequence<std::size_t, I...>` 的别名 |
| `std::make_index_sequence<N>` | 生成 `0, 1, ..., N-1` 的 `index_sequence` |
| `std::index_sequence_for<Ts...>` | 生成与参数包 `Ts...` 长度相同的 `index_sequence` |

### 快速理解：为什么需要整数序列？

考虑一个简单问题：把 `std::array<int, 4>` 转换为 `std::tuple<int, int, int, int>`：

```c++
std::array<int, 4> arr = {10, 20, 30, 40};
// 想要得到：std::make_tuple(arr[0], arr[1], arr[2], arr[3])
```

你不能这样写：
```c++
return std::make_tuple(arr[0], arr[1], arr[2], arr[3]);  // 硬编码，不可扩展
```

你需要一种方式在编译期生成索引 `0, 1, 2, 3` 并展开它们。`std::index_sequence` 就是做这个的：

```c++
template<typename T, std::size_t N, std::size_t... I>
auto array_to_tuple_impl(const std::array<T, N>& arr, std::index_sequence<I...>) {
    return std::make_tuple(arr[I]...);  // 展开为 arr[0], arr[1], arr[2], arr[3]
}

template<typename T, std::size_t N>
auto array_to_tuple(const std::array<T, N>& arr) {
    return array_to_tuple_impl(arr, std::make_index_sequence<N>{});
    //                         ↑                        ↑
    //                     array 对象          编译器自动生成 {0, 1, 2, 3}
}
```

`arr[I]...` 展开为 `arr[0], arr[1], arr[2], arr[3]`——这就是编译时整数序列的威力。

### 使用场景

#### 场景 1：array → tuple 转换

```c++
#include <array>
#include <tuple>
#include <iostream>

template<typename T, std::size_t N, std::size_t... I>
auto a2t_impl(const std::array<T, N>& a, std::index_sequence<I...>) {
    return std::make_tuple(a[I]...);
}

template<typename T, std::size_t N>
auto a2t(const std::array<T, N>& a) {
    return a2t_impl(a, std::make_index_sequence<N>{});
}

int main() {
    std::array<int, 4> arr = {1, 2, 3, 4};
    auto tup = a2t(arr);
    std::cout << std::get<0>(tup) << ", "
              << std::get<3>(tup) << std::endl;  // 1, 4
}
```

#### 场景 2：tuple 遍历打印（优雅地展开参数包）

```c++
#include <tuple>
#include <iostream>
#include <utility>

template<typename Tuple, std::size_t... I>
void print_tuple_impl(const Tuple& t, std::index_sequence<I...>) {
    // 折叠表达式 (C++17)：依次打印每个元素
    ((std::cout << (I == 0 ? "" : ", ") << std::get<I>(t)), ...);
}

template<typename... Args>
void print_tuple(const std::tuple<Args...>& t) {
    std::cout << "(";
    print_tuple_impl(t, std::index_sequence_for<Args...>{});
    std::cout << ")" << std::endl;
}

int main() {
    auto t = std::make_tuple(42, 3.14, "hello", 'x');
    print_tuple(t);  // (42, 3.14, hello, x)
}
```

#### 场景 3：自定义序列——跳过某些元素

```c++
#include <array>
#include <iostream>
#include <utility>

// 生成偶数索引序列 0, 2, 4, ...
template<typename T, T... N>
constexpr auto even_indices_impl(std::integer_sequence<T, N...>) {
    return std::integer_sequence<T, (N * 2)...>{};
}

template<std::size_t N>
using make_even_index_sequence = decltype(
    even_indices_impl(std::make_index_sequence<(N + 1) / 2>{})
);

// 只取偶数索引的元素
template<typename T, std::size_t N, std::size_t... I>
auto pick_even_impl(const std::array<T, N>& arr, std::index_sequence<I...>) {
    return std::array<T, sizeof...(I)>{arr[I]...};
}

template<typename T, std::size_t N>
auto pick_even(const std::array<T, N>& arr) {
    return pick_even_impl(arr, make_even_index_sequence<N>{});
}

int main() {
    std::array<int, 6> arr = {10, 20, 30, 40, 50, 60};
    auto evens = pick_even(arr);  // {10, 30, 50}
    for (auto v : evens) std::cout << v << " ";  // 10 30 50
    std::cout << std::endl;
}
```

#### 场景 4：将 `std::array` 作为参数包传递给函数

```c++
#include <array>
#include <iostream>
#include <utility>

// 接受可变参数的函数
template<typename... Args>
void process(Args... args) {
    ((std::cout << args * 2 << " "), ...);
}

template<typename T, std::size_t N, std::size_t... I>
void process_array_impl(const std::array<T, N>& arr, std::index_sequence<I...>) {
    process(arr[I]...);  // 将数组元素展开为参数包
}

template<typename T, std::size_t N>
void process_array(const std::array<T, N>& arr) {
    process_array_impl(arr, std::make_index_sequence<N>{});
}

int main() {
    std::array<int, 4> arr = {1, 2, 3, 4};
    process_array(arr);  // 2 4 6 8
}
```

#### 场景 5：从 tuple 构造自定义结构体

```c++
#include <tuple>
#include <string>
#include <iostream>
#include <utility>

struct Person {
    std::string name;
    int age;
    double height;

    void print() const {
        std::cout << name << ", " << age << "岁, " << height << "m" << std::endl;
    }
};

template<typename T, typename Tuple, std::size_t... I>
T make_from_tuple_impl(Tuple&& t, std::index_sequence<I...>) {
    return T{ std::get<I>(std::forward<Tuple>(t))... };
}

template<typename T, typename Tuple>
T make_from_tuple(Tuple&& t) {
    constexpr auto size = std::tuple_size_v<std::decay_t<Tuple>>;
    return make_from_tuple_impl<T>(
        std::forward<Tuple>(t),
        std::make_index_sequence<size>{}
    );
}

int main() {
    auto t = std::make_tuple("Alice", 30, 1.75);
    auto p = make_from_tuple<Person>(t);
    p.print();  // Alice, 30岁, 1.75m
}
```

### 关键要点总结

| 要点 | 说明 |
|:---|:---|
| **本质** | 编译期的整数列表，用于展开参数包 |
| **核心用法** | `std::make_index_sequence<N>{}` 生成 `{0, 1, ..., N-1}` |
| **展开语法** | `arr[I]...` 或 `std::get<I>(tuple)...` 将序列展开为逗号分隔的参数 |
| **适用对象** | `std::array`、`std::tuple`、`std::pair` 等所有"索引可访问"的容器 |
| **C++17 增强** | 结合折叠表达式 `(..., expr)` 使代码更简洁 |
| **典型模式** | 两层函数：实现层接受 `index_sequence`，接口层调用 `make_index_sequence` 自动生成序列 |

---

### Class Template Argument Deduction（类模板参数推导，CTAD）

### 为什么需要 CTAD？

在 C++17 之前，类模板**无法**像函数模板那样自动推导模板参数。每次使用类模板时，都必须显式写出所有模板参数，即使这些参数可以从构造函数参数中直接推断出来。

**C++17 之前的问题：**

```c++
// 函数模板可以自动推导 —— 没问题
template<typename T>
void foo(T x) {}
foo(42);       // T 自动推导为 int

// 类模板不能自动推导 —— 必须手写
std::pair<int, double> p(1, 2.5);  // 必须显式写 <int, double>
std::vector<std::string> v{"a", "b"};  // 必须显式写 <std::string>
```

这导致了大量**类型重复**：构造函数参数已经明确表达了类型，但程序员还要再写一遍模板参数。不仅冗余，而且容易出错（写错类型）。

### CTAD 解决了什么？

C++17 引入了 **类模板参数推导**（Class Template Argument Deduction, CTAD），允许编译器根据构造函数的参数自动推导类模板的模板参数。

```c++
// C++17 起 —— 编译器自动推导
std::pair p(1, 2.5);           // 推导为 std::pair<int, double>
std::vector v{1, 2, 3};        // 推导为 std::vector<int>
std::tuple t(1, "hello", 3.14); // 推导为 std::tuple<int, const char*, double>
```

### 工作原理

CTAD 的核心机制是**推导指引**（deduction guide）。编译器会：

1. 检查所有构造函数的参数类型
2. 尝试从实参推导模板参数
3. 如果推导成功，就实例化对应的类模板特化

对于标准库类型，编译器内置了推导指引。你也可以为自定义类模板编写推导指引：

```c++
template<typename T>
struct Container {
    T value;
    Container(T v) : value(v) {}
};

// 编译器可以自动推导：
Container c(42);  // 推导为 Container<int>

// 如果需要更复杂的推导，可以手写推导指引：
template<typename Iter>
Container(Iter begin, Iter end) -> Container<typename std::iterator_traits<Iter>::value_type>;
```

### 推导指引（Deduction Guide）详解

#### 推导指引语法

```
template<模板参数列表>
类名(构造函数参数类型列表) -> 类名<推导出的模板参数>;
```

以 `SmartPtr` 为例：

```c++
template<typename T>
struct SmartPtr {
    T* ptr;
    SmartPtr(T* p) : ptr(p) {}
};

// 推导指引
template<typename T>
SmartPtr(T*) -> SmartPtr<T>;

// 使用
int* raw = new int(42);
SmartPtr sp(raw);  // 推导为 SmartPtr<int>
```

**拆解推导指引的每一部分：**

| 部分 | 含义 |
|------|------|
| `template<typename T>` | 推导指引本身的模板参数 |
| `SmartPtr(T*)` | 当构造函数接受 `T*` 类型的参数时 |
| `->` | 推导出 |
| `SmartPtr<T>` | 类模板应实例化为 `SmartPtr<T>` |

**通俗理解**：这个推导指引告诉编译器——"当你看到有人用 `SmartPtr(某个指针)` 的方式构造对象时，请把指针的元素类型作为 `T`，实例化为 `SmartPtr<T>`"。

#### 完整推导过程

```c++
int* raw = new int(42);
SmartPtr sp(raw);  // 推导为 SmartPtr<int>
```

编译器的工作流程：

1. **看到构造调用**：`SmartPtr sp(raw)`
2. **查找推导指引**：找到 `SmartPtr(T*) -> SmartPtr<T>`
3. **模式匹配**：将 `raw` 的类型 `int*` 与推导指引的参数 `T*` 匹配
4. **解出 T**：`T*` = `int*` → `T = int`
5. **实例化**：生成 `SmartPtr<int>` 类型
6. **调用构造函数**：`SmartPtr<int>::SmartPtr(int* p)`

#### 更复杂的推导指引：迭代器范围构造

```c++
template<typename T>
class MyVector {
public:
    MyVector() = default;
    
    // 接受迭代器范围的构造函数
    template<typename Iter>
    MyVector(Iter begin, Iter end) {
        // 从迭代器范围复制元素
    }
};
```

这里构造函数本身是模板（`template<typename Iter>`），模板参数 `Iter` 和类的模板参数 `T` 没有直接关系。编译器**无法自动推导** `T` 是什么。

我们需要手写推导指引：

```c++
template<typename Iter>
MyVector(Iter, Iter) -> MyVector<typename std::iterator_traits<Iter>::value_type>;
```

这个推导指引说：
- 当用两个迭代器构造时
- 从迭代器的 `value_type`（通过 `std::iterator_traits` 获取）推导出 `T`

使用：

```c++
std::vector<int> v = {1, 2, 3};
MyVector mv(v.begin(), v.end());  // 推导为 MyVector<int>
```

#### 隐式推导指引 vs 显式推导指引

**隐式推导指引**：编译器会根据构造函数自动生成。

```c++
template<typename T>
struct Box {
    T value;
    Box(T v) : value(v) {}
};

Box b(42);  // ✅ 编译器自动生成推导指引：Box(T) -> Box<T>
```

**显式推导指引**：当隐式推导不够用或不符合预期时，手动编写。

```c++
template<typename T>
struct SmartPtr {
    T* ptr;
    SmartPtr(T* p) : ptr(p) {}
};

// 显式推导指引
template<typename T>
SmartPtr(T*) -> SmartPtr<T>;
```

#### 什么时候必须写推导指引？

| 场景 | 是否需要显式推导指引 |
|------|---------------------|
| 构造函数参数直接是 `T` | ❌ 不需要，编译器自动推导 |
| 构造函数参数是 `T*` | ⚠️ 通常需要（取决于编译器） |
| 构造函数参数是 `std::vector<T>` | ❌ 通常不需要 |
| 构造函数是模板（参数类型与 `T` 无关） | ✅ 必须写 |
| 需要从参数中提取类型（如迭代器的 `value_type`） | ✅ 必须写 |
| 希望改变默认推导行为 | ✅ 必须写 |

#### 推导指引不是函数

重要提醒：推导指引**只是编译期的类型推导规则**，不会生成任何运行时代码。

```c++
template<typename T>
SmartPtr(T*) -> SmartPtr<T>;  // 这只是告诉编译器如何推导类型
                               // 不会生成函数，不会有符号，不会有开销
```

它的作用**仅**在于帮助编译器确定 `SmartPtr sp(raw)` 中的 `sp` 应该是什么类型。一旦类型确定，就按照正常的构造函数调用流程执行。

### 使用场景

#### 场景 1：简化标准库容器创建

```c++
// 之前
std::vector<int> v = {1, 2, 3};
std::pair<std::string, int> p("age", 30);

// C++17
std::vector v = {1, 2, 3};
std::pair p("age", 30);
```

#### 场景 2：工厂函数替代方案

```c++
// 之前需要 make_pair / make_tuple
auto p = std::make_pair(1, 2.5);
auto t = std::make_tuple(1, "hello", 3.14);

// C++17 可以直接构造
std::pair p(1, 2.5);
std::tuple t(1, "hello", 3.14);
```

#### 场景 3：自定义类型推导

```c++
template<typename T>
struct SmartPtr {
    T* ptr;
    SmartPtr(T* p) : ptr(p) {}
};

// 推导指引
template<typename T>
SmartPtr(T*) -> SmartPtr<T>;

// 使用
int* raw = new int(42);
SmartPtr sp(raw);  // 推导为 SmartPtr<int>
```

#### 场景 4：与 `auto` 配合实现零类型重复

```c++
// 完全不需要写任何类型
auto v = std::vector{1, 2, 3};
auto p = std::pair{1, 2.5};
auto m = std::map{std::pair{1, "one"}, std::pair{2, "two"}};
```

#### 场景 5：带默认模板参数的类模板

```c++
template <typename T = float>
struct MyContainer {
  T val;
  MyContainer() : val{} {}
  MyContainer(T val) : val{val} {}
};

MyContainer c1{1};    // 推导为 MyContainer<int>
MyContainer c2;       // 推导为 MyContainer<float>（使用默认参数）
MyContainer c3{3.14}; // 推导为 MyContainer<double>
```

### 注意事项

| 注意事项 | 说明 |
|:---|:---|
| **聚合类型不支持 CTAD（C++17）** | C++17 中聚合类型（无构造函数）不能自动推导，C++20 才支持 |
| **推导可能不符合预期** | `std::vector v{"a", "b"}` 推导为 `vector<const char*>` 而非 `vector<std::string>` |
| **不能部分推导** | 要么全部推导，要么全部显式指定，不能只写一部分模板参数 |
| **自定义推导指引需谨慎** | 错误的推导指引会导致编译错误或意外的类型推导 |
| **复制/移动构造不参与推导** | 复制/移动构造函数不会触发 CTAD，它们只是创建同类型的副本 |

---

### Declaring Non-Type Template Parameters with Auto（使用 auto 声明非类型模板参数）

### 为什么需要这个特性？

在 C++17 之前，非类型模板参数（non-type template parameter）必须**显式指定类型**。这意味着你必须在编译期就知道参数的确切类型，即使这个类型可以从传入的值中直接推断出来。

**C++17 之前的问题：**

```c++
// 必须显式写类型
template <typename T, T... Values>
struct MySequence {
    static constexpr T size = sizeof...(Values);
};

// 使用
MySequence<int, 0, 1, 2> seq1;       // 必须写 <int, ...>
MySequence<long, 100L, 200L> seq2;   // 必须写 <long, ...>
```

这里 `int` 和 `long` 是**冗余的**——编译器完全可以从 `0, 1, 2` 推断出 `int`，从 `100L, 200L` 推断出 `long`。但 C++17 之前你必须手写。

**更严重的问题：类型不匹配**

```c++
// 容易出错：值与类型不匹配
MySequence<int, 10000000000> seq;  // 10000000000 超出 int 范围！
// 编译器可能静默截断或报错，取决于上下文
```

### C++17 的解决方案

C++17 允许用 `auto` 声明非类型模板参数，编译器会根据传入的值自动推导类型：

```c++
// C++17
template <auto... Values>
struct MySequence {
    static constexpr auto size = sizeof...(Values);
};

// 使用 —— 不需要写类型
MySequence<0, 1, 2> seq1;         // 推导为 int
MySequence<100L, 200L> seq2;      // 推导为 long
MySequence<'a', 'b', 'c'> seq3;   // 推导为 char
```

### 工作原理

`auto` 在这里遵循与普通变量声明相同的推导规则：

```c++
auto x = 0;      // x 是 int
auto y = 100L;   // y 是 long

// 等价于：
template <auto V> struct S1 {};  // S1<0>    → V 是 int
template <auto V> struct S2 {};  // S2<100L> → V 是 long
```

### 非类型模板参数的允许类型

`auto` 推导仍然受限于非类型模板参数的合法类型集合：

| 允许的类型 | 示例 |
|-----------|------|
| 整数类型 | `int`, `long`, `char` |
| 指针类型 | `int*`, `void*` |
| 左值引用 | `int&`, `MyClass&` |
| 枚举类型 | `enum Color { Red, Green }` |
| `std::nullptr_t` | `nullptr` |
| 浮点类型（C++20 起） | `double`, `float` |
| 字面量类类型（C++20 起） | 有 `constexpr` 构造函数的类 |

**不允许的类型：**
- `double`（C++17 不允许，C++20 允许）
- `std::string`（不是字面量类型）
- 动态分配的对象

### 使用场景

#### 场景 1：简化整数序列定义

```c++
// C++17 之前
template <typename T, T... I>
struct integer_sequence {};

auto s = integer_sequence<int, 0, 1, 2, 3>{};

// C++17
template <auto... I>
struct integer_sequence {};

auto s = integer_sequence<0, 1, 2, 3>{};  // 更简洁
```

#### 场景 2：编译期字符串/字符处理

```c++
template <char... Chars>
struct StringLiteral {
    static constexpr char value[] = {Chars..., '\0'};
};

// 编译期存储字符序列
auto hello = StringLiteral<'H', 'e', 'l', 'l', 'o'>{};
```

#### 场景 3：模板元编程中的类型无关序列

```c++
// 一个通用的编译期值列表，不关心具体类型
template <auto... Values>
struct ValueList {
    static constexpr std::size_t size = sizeof...(Values);
};

// 可以分别实例化不同类型的序列
auto ints = ValueList<1, 2, 3>{};
auto longs = ValueList<1L, 2L, 3L>{};
auto chars = ValueList<'a', 'b', 'c'>{};
```

#### 场景 4：枚举值的模板参数

```c++
enum class Color { Red, Green, Blue };

template <Color C>
struct ColorProcessor {
    static void process() {
        if constexpr (C == Color::Red) {
            // 处理红色
        }
    }
};

// C++17 可以用 auto
template <auto C>
struct GenericEnumProcessor {
    static void process() {
        // 适用于任何枚举类型
    }
};

GenericEnumProcessor<Color::Red> p1;
```

#### 场景 5：指针作为模板参数

```c++
int global_var = 42;

template <auto* Ptr>
struct PointerWrapper {
    static auto get() { return *Ptr; }
};

auto wrapper = PointerWrapper<&global_var>{};
auto val = wrapper.get();  // 42
```

#### 场景 6：与折叠表达式配合

```c++
// 编译期求和
template <auto... Values>
constexpr auto sum = (Values + ...);

static_assert(sum<1, 2, 3, 4> == 10);

// 编译期逻辑与
template <auto... Values>
constexpr bool all_true = (true && ... && Values);

static_assert(all_true<true, true, true> == true);
static_assert(all_true<true, false, true> == false);
```

### 注意事项

| 注意事项 | 说明 |
|---------|------|
| **同一参数包必须类型一致** | `ValueList<1, 2L>` 会编译错误，因为 `auto...` 推导出的类型必须相同 |
| **C++17 不支持浮点数** | `template <auto V>` 传 `3.14` 在 C++17 中非法，C++20 才支持 |
| **不能推导数组类型** | 非类型模板参数不能是数组类型 |

---

### Class Types in Non-Type Template Parameters（非类型模板参数中的类类型，C++20）

> **版本演进**：这是上一节 `Declaring Non-Type Template Parameters with Auto`（C++17）的延续——C++17 用 `auto` 让**整数/指针/枚举**等标量可作为非类型模板参数；C++20 进一步允许**类类型**参与。

### 特性说明

C++20 允许**类类型**（class type）作为非类型模板参数，只要它是**字面量类型**（LiteralType，即有 `constexpr` 构造函数、析构函数、所有成员可用 constexpr 构造）：

```c++
struct foo {
  foo() = default;
  constexpr foo(int) {}
};

template <foo f = {}>
auto get_foo() {
  return f;
}

get_foo();           // 使用默认参数 foo{}
get_foo<foo{123}>(); // 显式传 foo{123}
```

**作为模板参数传递的对象具有类型 `const T`，并具有静态存储持续时间**，含义是：

| 规定 | 含义 | 设计目的 |
|:---|:---|:---|
| **`const T`** | 模板参数对象是 `const` 的，**不可修改** | 保证所有实例化看到同一份不可变对象，可当编译期常量使用 |
| **静态存储持续时间** | 程序启动时创建、结束时销毁，全局只存在一份，地址稳定 | 生命周期覆盖整个程序，任何时刻访问都可靠（与 constexpr 全局量语义一致） |

> 通俗理解：非类型模板参数对象就像一份"**写在代码里的 constexpr 全局常量**"——它不可变、整个程序存活、所有用它的模板实例共享同一份。

### 为什么需要？（C++20 之前怎么做）

**痛点**：非类型模板参数长期只能接受标量（整数、指针、引用、枚举、`nullptr_t`）。想传一个"结构化的编译期配置"非常别扭：

**1. 只能拆成多个标量参数** —— 配置成员一多就爆炸

```c++
// C++17 及之前 —— 每个字段都是一个模板参数，又长又难维护
template <int Port, const char* Host, bool Verbose, int Timeout>
void serve() { /* ... */ }

serve<9090, "localhost", false, 5000>();  // 顺序一错全错
```

**2. 用类型包装（`std::integral_constant` 技巧）** —— 把一个值"伪装"成类型

```c++
// 传统元编程技巧：值 → 类型 → 再当模板参数
template <int N>
using int_ = std::integral_constant<int, N>;

template <typename... Ts>  // 参数包装的是"类型"，里面藏着值
void f() { /* 用 Ts::value 取回值 */ }
```

**3. 无法把 constexpr 对象直接当参数** —— 想传"一个 `Config` 对象"，做不到

C++20 直接解决：**把一个 constexpr 构造的类对象整体作为模板参数**，字段、语义、可读性全都有了。

### 使用场景

#### 场景 1：编译期配置对象（最实用）

```c++
struct Config {
    int port = 8080;
    const char* host = "localhost";
    bool verbose = false;

    constexpr Config(int p, const char* h, bool v)
        : port(p), host(h), verbose(v) {}
};

template <Config C>
void serve() {
    // C.port / C.host 都是编译期常量，可直接用于静态数组、switch 等
    char log_path[C.port > 10000 ? 128 : 64];  // 编译期分支
}

serve<Config{9090, "0.0.0.0", true}>();
```

#### 场景 2：编译期字符串（本特性最著名的应用）

C++ 一直缺少"字符串作为模板参数"（`template <const char* S>` 只能传外部链接的全局字符串，且很受限）。C++20 配合 `FixedString` 彻底解决：

```c++
template <std::size_t N>
struct FixedString {
    char data[N]{};
    constexpr FixedString(const char (&s)[N]) {
        for (std::size_t i = 0; i < N; ++i) data[i] = s[i];
    }
};

template <FixedString S>
struct Message {
    static constexpr const char* text = S.data;
};

Message<"hello"> m1;
Message<"world"> m2;
// 字符串字面量直接作为模板参数 —— 类型系统能区分 "hello" 和 "world"
static_assert(!std::is_same_v<decltype(m1), decltype(m2)>);
```

#### 场景 3：类型安全的单位/维度系统

```c++
template <typename Unit, int Value>
struct Quantity {
    static constexpr int value = Value;
    // 相同 Unit 的 Quantity 才能比较/运算
};

// C++20：结合类类型 + auto，可以写出更丰富的编译期值类型
struct Point {
    int x, y;
    constexpr Point(int a, int b) : x(a), y(b) {}
    friend constexpr bool operator==(const Point&, const Point&) = default;
};

template <Point P>
void draw_at() {
    static_assert(P == Point{0, 0}, "原点不能画");
    // P.x, P.y 编译期已知
}
```

### 与 C++17 `template<auto>` 的版本演进对比

| | C++17 `template <auto V>` | C++20 类类型非类型模板参数 |
|:---|:---|:---|
| 允许的类型 | 标量：整数、指针、引用、枚举、`nullptr_t`（浮点 C++20 才有） | **字面量类类型**（含 `constexpr` 构造） |
| 结构化信息 | ❌ 一个值，无内部结构 | ✅ 多个成员、方法、嵌套类型 |
| 字符串作为参数 | ❌ 很受限 | ✅ 配合 `FixedString` 可以 |
| 对象语义 | 无 | ✅ 有拷贝语义、可 `==` 比较 |
| 典型用途 | 编译期数值/常量 | 编译期配置、字符串、值对象 |

**演进链条**（模板参数越来越"强大"）：

```
C++17 之前：template<typename T>     只能传【类型】
C++17     ：template<auto V>         能传【标量值】（int / 指针 / 枚举…）
C++20     ：template<Foo f>          能传【结构化对象】（class 字面量）
```

### 最佳实践

1. **必须是字面量类型**：要有 `constexpr` 构造函数、`constexpr`（或默认）析构、所有成员 `constexpr` 可构造；否则不能作为模板参数
2. **一般配合 `constexpr` 使用**：否则对象无法在编译期构造，失去意义
3. **字段不可变**：参数对象是 `const T`，不能在编译期修改；需要变化的值请用普通运行时参数
4. **优先 `template <auto>` 泛化**：只需要单个标量时用 `auto`；需要结构化信息（多个字段、字符串）时才用类类型
5. **用 `static_assert` 在编译期校验**：如 `static_assert(C.port > 0)`，把非法配置挡在编译期
6. **警惕模板爆炸**：每个不同的模板参数值都会实例化一份，别把运行时常量误用成模板参数

### 对比 Rust

Rust 没有直接的等价物，最接近的是 **const generics（常量泛型）**：

```rust
fn foo<const N: usize>() { ... }
foo::<42>();  // 只能传整数等简单标量
```

| | C++20 | Rust |
|:---|:---|:---|
| 结构化 const 参数 | ✅ 类类型对象（含字符串） | ❌ 仅支持整数/bool/char 等标量 |
| 字符串 const 参数 | ✅ `FixedString` | ❌ 不可行 |
| 实现成熟度 | 已进标准 | 仍在演进（计划支持 struct const 参数） |

> **结论**：在"把结构化对象当编译期模板参数"这一点上，C++20 领先于 Rust——Rust 的 const generics 目前还只能处理简单标量。

---

### Folding Expression（折叠表达式）

### 为什么需要折叠表达式？

C++11 引入了可变参数模板（variadic templates），但缺少一种直接的方式对参数包中的所有元素执行操作。在 C++17 之前，要实现参数包求和，不得不借助递归模板展开：

```c++
// C++11/14：递归展开——繁琐、低效、难以阅读
template <typename T>
T sum(T v) { return v; }  // 递归终止条件

template <typename T, typename... Args>
T sum(T first, Args... rest) {
    return first + sum(rest...);  // 每多一个参数就多一层递归实例化
}
```

或者用 `std::initializer_list` 和 `std::accumulate` 的"技巧"：

```c++
template <typename... Args>
auto sum(Args... args) {
    std::common_type_t<Args...> arr[] = {args...};  // 转为数组
    return std::accumulate(std::begin(arr), std::end(arr), 0);
}
```

**折叠表达式的出现彻底解决了这个问题**——它让对参数包的批量操作变得像普通表达式一样直观。

### 核心语法

折叠表达式有四种形式，理解它们的区别是正确使用的关键：

| 形式 | 名称 | 含义 | 展开结果（以 `+` 为例） |
|:---|:---|:---|:---|
| `(... op e)` | 一元左折叠 | `...` 在参数包**左侧**，从左到右结合 | `((e1 op e2) op e3) op ...` |
| `(e op ...)` | 一元右折叠 | `...` 在参数包**右侧**，从右到左结合 | `e1 op (e2 op (... op eN))` |
| `(init op ... op e)` | 二元左折叠 | 参数包 `e` 在 `...` 右侧，`init` 在左侧，从左到右结合 | `(((init op e1) op e2) op e3) ...` |
| `(e op ... op init)` | 二元右折叠 | 参数包 `e` 在 `...` 左侧，`init` 在右侧，从右到左结合 | `e1 op (e2 op (e3 op (... op init)))` |

> **记忆口诀**：看 `...` 相对于参数包 `e` 的位置——`...` 在左就是左折叠（左结合），`...` 在右就是右折叠（右结合）。

**关键差异——用非结合运算符 `-` 看区别：**

```c++
// 一元左折叠：(... - args) → ((1 - 2) - 3) - 4 = -8
// 一元右折叠：(args - ...) → 1 - (2 - (3 - 4)) = -2
template <typename... Args>
auto unary_left(Args... args)  { return (... - args); }

template <typename... Args>
auto unary_right(Args... args) { return (args - ...); }

unary_left(1, 2, 3, 4);   // ((1-2)-3)-4 = -8
unary_right(1, 2, 3, 4);  // 1-(2-(3-4)) = -2

// 二元左折叠：(init - ... - args) → (((100 - 5) - 3) - 2 = 90
// 二元右折叠：(args - ... - init) → 5 - (3 - (2 - 100)) = -96
template <typename... Args>
auto binary_left(Args... args)  { return (100 - ... - args); }  // init=100, pack 在右

template <typename... Args>
auto binary_right(Args... args) { return (args - ... - 100); }  // init=100, pack 在左

binary_left(5, 3, 2);   // ((100-5)-3)-2 = 90
binary_right(5, 3, 2);  // 5-(3-(2-100)) = -96
```

**二元折叠 vs 一元折叠：**

| 对比 | 一元折叠 | 二元折叠 |
|:---|:---|:---|
| 初始值 | ❌ 无初始值，用参数包的首元素 | ✅ 提供显式初始值 |
| 空参数包 | ❌ 大多数运算符编译错误 | ✅ 直接返回初始值，安全 |
| 语法 | `(... + args)` 或 `(args + ...)` | `(init + ... + args)` 或 `(args + ... + init)` |

对于可结合的运算符（`+`、`*`、`&&`、`||`），左右折叠结果相同，但对**非结合性运算符**（`-`、`/`）则完全不同。

### 空参数包行为

| 运算符 | 空包返回值 | 说明 |
|:---|:---|:---|
| `&&` | `true` | 逻辑与，空集为真 |
| `||` | `false` | 逻辑或，空集为假 |
| `,` | `void()` | 逗号表达式，空包返回 void |
| 其他（`+`、`*` 等） | ❌ 编译错误 | 一元折叠不允许空包 |

**解决方案：使用二元折叠提供初始值。**

### 使用场景

#### 场景 1：求和与求积

```c++
#include <iostream>

template <typename... Args>
auto sum(Args... args) {
    return (... + args);  // 一元左折叠
}

template <typename... Args>
auto product(Args... args) {
    return (... * args);  // 一元左折叠
}

int main() {
    std::cout << sum(1, 2, 3, 4, 5) << std::endl;     // 15
    std::cout << product(2, 3, 4) << std::endl;       // 24

    // 空包安全——使用二元折叠
    auto safe_sum = [](auto... args) { return (0 + ... + args); };
    std::cout << safe_sum() << std::endl;              // 0
}
```

#### 场景 2：逻辑检查——所有/任意条件

```c++
#include <iostream>
#include <type_traits>

// 检查所有值是否为 true
template <typename... Args>
bool all_of(Args... args) {
    return (... && args);
}

// 检查是否有任意值为 true
template <typename... Args>
bool any_of(Args... args) {
    return (... || args);
}

// 编译时类型检查
template <typename... Args>
constexpr bool all_arithmetic_v = (std::is_arithmetic_v<Args> && ...);

int main() {
    std::cout << std::boolalpha;
    std::cout << all_of(true, true, true) << std::endl;   // true
    std::cout << any_of(false, false, true) << std::endl; // true
    std::cout << all_of() << std::endl;                   // true（空包）

    static_assert(all_arithmetic_v<int, double, long>);
    static_assert(!all_arithmetic_v<int, std::string, double>);
}
```

#### 场景 3：打印与遍历

```c++
#include <iostream>
#include <vector>

// 逐元素打印，空格分隔
template <typename... Args>
void print(Args&&... args) {
    ((std::cout << args << " "), ...);  // 逗号表达式折叠
    std::cout << std::endl;
}

// 带分隔符的打印
template <typename... Args>
void print_delimited(const std::string& delim, Args&&... args) {
    bool first = true;
    ((std::cout << (first ? "" : delim) << args, first = false), ...);
    std::cout << std::endl;
}

// 推入多个元素到容器
template <typename Container, typename... Args>
void push_all(Container& c, Args&&... args) {
    (c.push_back(std::forward<Args>(args)), ...);
}

int main() {
    print(1, 2, 3, "hello", 4.5);       // 1 2 3 hello 4.5
    print_delimited(", ", 10, 20, 30);   // 10, 20, 30

    std::vector<int> v;
    push_all(v, 1, 2, 3, 4, 5);
    for (auto x : v) std::cout << x << " ";  // 1 2 3 4 5
}
```

#### 场景 4：编译期类型匹配检查

```c++
#include <type_traits>
#include <iostream>
#include <string>

// 检查所有类型是否都可转换为字符串
template <typename... Args>
constexpr bool all_to_string_v =
    (std::is_convertible_v<Args, std::string> && ...);

// 检查参数包中是否包含某种类型
template <typename T, typename... Args>
constexpr bool contains_type_v =
    (std::is_same_v<T, Args> || ...);

int main() {
    static_assert(all_to_string_v<const char*, std::string, const char (&)[5]>);
    static_assert(!all_to_string_v<int, std::string>);

    static_assert(contains_type_v<int, double, int, float>);    // true
    static_assert(!contains_type_v<int, double, float, char>);  // false
}
```

#### 场景 5：调用每个参数的成员函数（访问者模式）

```c++
#include <iostream>
#include <memory>
#include <vector>

template <typename... Bases>
struct Visitor : Bases... {
    using Bases::operator()...;  // C++17：将所有基类的 operator() 引入作用域
};

struct Circle { void draw() const { std::cout << "○ "; } };
struct Square { void draw() const { std::cout << "□ "; } };
struct Triangle { void draw() const { std::cout << "△ "; } };

// 对每个对象调用 draw()
template <typename... Shapes>
void draw_all(const Shapes&... shapes) {
    (shapes.draw(), ...);
}

int main() {
    draw_all(Circle{}, Square{}, Triangle{}, Circle{});
    // ○ □ △ ○
}
```

#### 场景 6：与 CTAD 和 auto 模板参数配合

```c++
#include <iostream>

// 编译期求和——所有值在编译期确定
template <auto... Values>
constexpr auto compile_time_sum = (Values + ...);

// 编译期打印枚举值（非常实用）
template <auto... Values>
constexpr auto logical_and = (true && ... && Values);

int main() {
    constexpr auto s = compile_time_sum<1, 2, 3, 4, 5>;
    static_assert(s == 15);

    static_assert(logical_and<true, true, true>);    // true
    static_assert(!logical_and<true, false, true>);  // false
}
```

#### 场景 7：与 std::apply 配合处理元组

```c++
#include <tuple>
#include <iostream>
#include <functional>

// 对 tuple 的每个元素执行操作
template <typename Tuple, typename Func>
void for_each_tuple(Tuple&& t, Func&& f) {
    std::apply(
        [&f](auto&&... args) {
            (f(std::forward<decltype(args)>(args)), ...);
        },
        std::forward<Tuple>(t)
    );
}

int main() {
    auto t = std::make_tuple(1, 2.5, "hello");
    for_each_tuple(t, [](const auto& v) {
        std::cout << v << " ";
    });
    // 1 2.5 hello
}
```

#### 场景 8：类型安全的最小值/最大值

```c++
#include <iostream>
#include <algorithm>

template <typename T, typename... Args>
T min_of(T first, Args... rest) {
    // 二元右折叠：(first < ... < rest) 不适用
    // 改用逗号折叠 + std::min
    T result = first;
    ((result = std::min(result, static_cast<T>(rest))), ...);
    return result;
}

template <typename T, typename... Args>
T max_of(T first, Args... rest) {
    T result = first;
    ((result = std::max(result, static_cast<T>(rest))), ...);
    return result;
}

int main() {
    std::cout << min_of(5, 3, 8, 1, 9, 2) << std::endl;  // 1
    std::cout << max_of(5, 3, 8, 1, 9, 2) << std::endl;  // 9
}
```

### 常见陷阱

| 陷阱 | 说明 | 正确做法 |
|:---|:---|:---|
| **一元折叠空包** | `(... + args)` 在空包时编译错误 | 用二元折叠 `(0 + ... + args)` |
| **左/右折叠混淆** | `(... - args)` vs `(args - ...)` 对非结合运算符结果不同 | 明确使用 `(... op args)`（左折叠）或 `(args op ...)`（右折叠） |
| **逗号折叠与副作用顺序** | C++17 保证逗号表达式从左到右求值 | `(f(args), ...)` 按顺序执行 |
| **类型不匹配** | 参数包中元素类型不同时需注意类型转换 | 用 `static_cast<common_type_t<Args...>>` |
| **忘记括号** | 折叠表达式**必须**用括号包裹 | `(... + args)` ✅，`... + args` ❌ |

### 最佳实践

1. **优先使用一元左折叠** `(... op args)`：更符合直觉（从左到右计算），大多数情况下结果正确
2. **提供初始值的二元折叠**：当参数包可能为空时，用二元折叠提供安全默认值
3. **逗号折叠用于副作用操作**：`(f(args), ...)` 是遍历参数包执行操作的标准模式
4. **利用空包默认值**：`&&` 空包为 `true`，`||` 空包为 `false`，合理利用可以简化边界条件
5. **结合 `std::apply`**：折叠表达式 + `std::apply` = 对元组执行任意批量操作

---

### Auto 从花括号初始化列表推导的新规则

### 特性说明

C++17 修改了 `auto` 从花括号初始化列表（braced-init-list）推导的规则，修复了 C++11 中一个广受批评的设计缺陷。
### 规则速查

| 写法 | 初始化方式 | C++11/14 | C++17 起 |
|:---|:---|:---|:---|
| `auto x{3};` | 直接列表初始化（无 `=`） | `std::initializer_list<int>` | ✅ `int` |
| `auto x{1, 2, 3};` | 直接列表初始化（无 `=`） | `std::initializer_list<int>` | ❌ **编译错误** |
| `auto x = {3};` | 复制列表初始化（有 `=`） | `std::initializer_list<int>` | ✅ `std::initializer_list<int>` |
| `auto x = {1, 2, 3};` | 复制列表初始化（有 `=`） | `std::initializer_list<int>` | ✅ `std::initializer_list<int>` |

| `auto x = {1, 2, 3.0};` | 复制列表初始化（有 `=`） | 编译错误（类型不一致） | ❌ 编译错误（类型不一致） |

### 为什么需要这个变化？

C++11 中 `auto x{3};` 推导为 `std::initializer_list<int>` 被广泛认为是一个**设计缺陷**——绝大多数开发者的直觉是 `x` 应该是一个 `int`。这个行为与常见预期严重不符：

```c++
// C++11 —— 出乎意料的行为
auto x{3};       // 你以为 x 是 int？不，它是 initializer_list<int>
auto y{3, 4};    // initializer_list<int>

// 这导致了很多隐蔽的 bug：
auto z{42};                      // 你以为 z 是 int，但它是 initializer_list
auto sum = z + 1;                // 编译错误！initializer_list 不能 + 1
```

Scott Meyers 在《Effective Modern C++》中重点批评了这一点，称其为"一个令人惊讶的推导规则"。

### C++17 的修复策略

C++17 通过提案 [N3922](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2014/n3922.html) 修改了规则，采用了**一个折衷方案**：

1. **直接列表初始化**（无 `=`）`auto x{n}`：
   - 单元素：推导为**元素本身的类型**（`auto x{3}` → `int`）
   - 多元素：**编译错误**（`auto x{1, 2, 3}` → 错误）

2. **复制列表初始化**（有 `=`）`auto x = {n}`：
   - 行为不变，始终推导为 `std::initializer_list`

### 为什么多元素的直接列表初始化被禁止？

这是为了**保持语义一致性**：

```c++
auto x{3};       // 3 是一个 int → x 是 int            ✅ 合理
auto y{3, 4};    // 3, 4 也是 int → y 是什么？           🤔
```

如果 `auto y{3, 4}` 也推导为 `int`，那 `y` 应该等于 `3` 还是 `4`？语义矛盾。
如果推导为 `std::initializer_list<int>`，又和 `auto x{3}` 的行为不一致。

最终的决策是：**与其默默做错事，不如直接报错**。想要 `initializer_list` 用复制列表初始化即可。

### 实用建议

```c++
// ✅ C++17 推荐用法
auto a{42};                     // int——符合直觉
auto b = {1, 2, 3};            // initializer_list<int>——明确意图
auto c = std::initializer_list<int>{1, 2, 3};  // 显式，更清晰

// ❌ 避免
// auto d{1, 2, 3};             // C++17 编译错误

// 如果你需要推导出容器类型：
auto v1 = std::vector{1, 2, 3};  // C++17 CTAD 自动推导为 vector<int>
auto m = std::map{{1, "one"}, {2, "two"}};  // CTAD 推导
```

### 总结

| 版本 | 行为评价 |
|:---|:---|
| C++11 | 设计缺陷——`auto x{3}` 意外推导为 `initializer_list` |
| C++14 | ❌ 未修复，继承 C++11 行为 |
| C++17 | ✅ 修复：单元素推导为值本身，多元素报错 |
| 教训 | 直接列表初始化侧重"精确的单个值"，复制列表初始化侧重"列表" |

---

### Constexpr Lambda（编译期 lambda）

### 为什么需要 constexpr lambda？

C++11 引入了 `constexpr` 函数，C++14 放宽了其限制。但 lambda 表达式在 C++17 之前**不能显式标记为 `constexpr`**，也无法用于需要编译期求值的上下文（如 `static_assert`）。

这导致了一个尴尬的局面：同样的功能，用普通函数可以实现编译期计算，用 lambda 就不行——即使它们做的事情完全一样。

```c++
// C++14 —— 普通函数可以 constexpr，lambda 不行
constexpr int add(int a, int b) { return a + b; }
static_assert(add(1, 2) == 3);  // ✅ 可以

auto lambda_add = [](int a, int b) { return a + b; };
// static_assert(lambda_add(1, 2) == 3);  // ❌ C++14 编译错误
```

### 特性说明

C++17 允许 lambda 被标记为 `constexpr`，并且**即使不显式标记**，只要满足 `constexpr` 函数的要求，编译器也会自动使其可在编译期调用。

```c++
// C++17 —— lambda 可以用于编译期上下文

// 方式 1：显式标记 constexpr
auto identity = [](int n) constexpr { return n; };
static_assert(identity(123) == 123);

// 方式 2：隐式 constexpr（编译器自动判断）
auto add = [](int a, int b) { return a + b; };
static_assert(add(1, 2) == 3);  // ✅ C++17 可以
```

### 对比：C++14 vs C++17

| 方面 | C++14 | C++17 |
|:---|:---|:---|
| 显式 `constexpr` 标记 | ❌ 语法错误 | ✅ `[]() constexpr {}` |
| 隐式编译期求值 | ❌ 不能用于 `static_assert` | ✅ 满足条件即可 |
| `mutable` lambda 是否可为 constexpr | ❌ 不支持 | ✅ 支持 |
| 捕获子句的支持 | ❌ | ✅ 按值捕获的 constexpr 变量可用 |

### 使用场景

#### 场景 1：编译期计算工厂

```c++
constexpr auto make_multiplier = [](int factor) {
    return [factor](int x) constexpr { return x * factor; };
};

constexpr auto double_it = make_multiplier(2);
constexpr auto triple_it = make_multiplier(3);

static_assert(double_it(5) == 10);   // 10
static_assert(triple_it(5) == 15);   // 15
```

#### 场景 2：与折叠表达式结合——编译期序列操作

```c++
template <typename... Args>
constexpr auto sum_all = [](Args... args) {
    return (... + args);
};

static_assert(sum_all<int>(1, 2, 3, 4, 5) == 15);

// 编译期转换
constexpr auto transform = [](auto&&... args) {
    return ((args * 2) + ...);
};
static_assert(transform(1, 2, 3) == 12);  // (1*2)+(2*2)+(3*2) = 12
```

#### 场景 3：编译期配置与策略

```c++
constexpr auto get_algorithm = [](bool use_fast) {
    if constexpr (use_fast) {
        return [](int x) constexpr { return x * 2; };
    } else {
        return [](int x) constexpr { return x * x; };
    }
};

constexpr auto fast = get_algorithm(true);
constexpr auto safe = get_algorithm(false);

static_assert(fast(5) == 10);
static_assert(safe(5) == 25);
```

#### 场景 4：编译期类型萃取辅助

```c++
#include <type_traits>

constexpr auto is_integral = [](auto x) {
    return std::is_integral_v<decltype(x)>;
};

static_assert(is_integral(42));
static_assert(!is_integral(3.14));
```

#### 场景 5：编译期捕获上下文

```c++
constexpr int factor = 10;

// 捕获 constexpr 变量后用于编译期计算
constexpr auto compute = [factor](int x) constexpr {
    return x * factor;
};

static_assert(compute(7) == 70);
```

### 嵌套 lambda 的编译期执行

C++17 允许 lambda 嵌套返回 lambda，并且**全部在编译期执行**：

```c++
constexpr auto add = [](int x, int y) {
  auto L = [=] { return x; };
  auto R = [=] { return y; };
  return [=] { return L() + R(); };
};
static_assert(add(1, 2)() == 3);
```

展开执行过程：

1. `add(1, 2)` → 创建捕获 `x=1` 的 `L` 和捕获 `y=2` 的 `R`
2. 返回一个捕获 `L` 和 `R` 的新 lambda
3. 调用该 lambda → `L() + R()` → `1 + 2 = 3`
4. **全部在编译期完成**

### 注意事项

| 注意事项 | 说明 |
|:---|:---|
| **不能捕获运行时变量** | 如果捕获了运行时变量，lambda 不再是 constexpr |
| **按引用捕获通常不是 constexpr** | 按引用捕获的变量在编译期上下文中受限 |
| **`constexpr` 标记可选但推荐** | 显式标记让意图更清晰，且编译器能给出更好的错误信息 |

---

### Structured Binding（结构化绑定）

### 为什么需要结构化绑定？

C++17 之前，从复合类型（pair、tuple、struct）中提取多个值非常冗长：

```c++
// C++11/14 —— 繁琐且容易出错
std::map<std::string, int> scores;
auto ret = scores.insert({"Alice", 95});
if (ret.second) {
    // 使用 ret.first（迭代器）...
}

// 或者用 std::tie（需要先声明变量）
std::string key;
int value;
std::tie(key, value) = *scores.begin();
```

结构化绑定让这一切变得极其简洁直观：

```c++
// C++17 —— 一行搞定
auto [pos, inserted] = scores.insert({"Alice", 95});
auto [key, value] = *scores.begin();
```

### 基本语法

```c++
auto [binding1, binding2, ...] = expression;
//    ↑ 引入新变量名        ↑ 任何可解构的表达式

// 三种形式：
auto  [a, b] = expr;      // 按值复制（创建副本）
auto& [a, b] = expr;      // 左值引用（绑定到原对象）
const auto& [a, b] = expr; // const 引用（只读访问）
```

### 工作原理

结构化绑定**不是**在解构一个数组或 tuple，而是编译器在背后做这些事情：

```c++
auto [x, y] = some_pair;

// 编译器将其展开为（伪代码）：
auto __e = some_pair;           // 1. 创建隐藏变量
using __t = decltype(__e);      // 2. 获取类型
std::tuple_element<0, __t>::type& x = std::get<0>(__e);  // 3. x 绑定到第 0 元素
std::tuple_element<1, __t>::type& y = std::get<1>(__e);  // 4. y 绑定到第 1 元素
```

关键点：**`x` 和 `y` 不是新声明的独立变量，而是指向隐藏变量 `__e` 的别名**。

### 三类可解构对象

结构化绑定可作用于三类目标，每类的展开机制不同：

| 类型 | 展开方式 | 示例 |
|:---|:---|:---|
| **数组** | 直接绑定到数组元素 | `int arr[3]; auto [a,b,c] = arr;` |
| **元组式** | 通过 `std::get<I>` / `tuple_size` / `tuple_element` | `std::pair`、`std::tuple`、`std::array` |
| **聚合结构体** | 按成员声明顺序绑定 | `struct Point { int x, y; }; auto [x,y] = Point{};` |

#### 1. 数组

```c++
int arr[3] = {10, 20, 30};
auto [a, b, c] = arr;    // a=10, b=20, c=30（复制了数组副本）
auto& [ra, rb, rc] = arr; // ra, rb, rc 是 arr 元素的引用
ra = 100;                 // arr[0] 变为 100
```

#### 2. 元组式（pair、tuple、array 等）

```c++
std::pair<int, std::string> p{1, "hello"};
auto [id, name] = p;       // id=1, name="hello"

std::tuple<int, double, char> t{42, 3.14, 'A'};
auto [i, d, c] = t;        // i=42, d=3.14, c='A'

std::array<int, 4> arr{1, 2, 3, 4};
auto [x1, x2, x3, x4] = arr; // x1=1, x2=2, x3=3, x4=4
```

#### 3. 聚合结构体

```c++
struct Point { int x; int y; };
struct Line { Point start; Point end; };

Point p{10, 20};
auto [px, py] = p;        // px=10, py=20

Line line{{0, 0}, {100, 200}};
auto [s, e] = line;        // s=Point{0,0}, e=Point{100,200}
auto [sx, sy] = s;         // sx=0, sy=0

// 嵌套解包（C++17 不支持直接嵌套语法，需要分两步）
```

### 使用场景

#### 场景 1：简化 map 遍历

这是结构化绑定最常见的用途——再也不用写 `it->first` 和 `it->second`：

```c++
#include <map>
#include <string>
#include <iostream>

std::map<std::string, int> scores = {
    {"Alice", 95}, {"Bob", 87}, {"Charlie", 92}
};

// C++17 之前
for (const auto& pair : scores) {
    std::cout << pair.first << ": " << pair.second << std::endl;
}

// C++17 —— 一目了然
for (const auto& [name, score] : scores) {
    std::cout << name << ": " << score << std::endl;
}
```

#### 场景 2：函数返回多值

```c++
#include <tuple>
#include <string>
#include <iostream>

// 返回多值的函数
std::tuple<int, std::string, bool> parse_result(const std::string& input) {
    if (input.empty()) return {0, "", false};
    return {(int)input.size(), "ok", true};
}

int main() {
    auto [code, msg, success] = parse_result("hello");
    // code = 5, msg = "ok", success = true
    std::cout << code << ", " << msg << ", " << std::boolalpha << success;
}
```

#### 场景 3：insert/emplace 的返回值解包

```c++
#include <set>
#include <map>
#include <iostream>

// set::insert 返回 pair<iterator, bool>
std::set<int> s;
auto [it, inserted] = s.insert(42);
if (inserted) {
    std::cout << "插入成功: " << *it << std::endl;
}

// map::insert 类似
std::map<int, std::string> m;
auto [pos, ok] = m.insert({1, "one"});
```

#### 场景 4：与 range-based for 遍历自定义结构体

```c++
#include <vector>
#include <iostream>

struct Employee {
    int id;
    std::string name;
    double salary;
};

int main() {
    std::vector<Employee> employees = {
        {1, "Alice", 50000},
        {2, "Bob",   60000},
        {3, "Charlie", 55000}
    };

    double total = 0;
    for (const auto& [id, name, salary] : employees) {
        total += salary;
        std::cout << name << " 的薪资: " << salary << std::endl;
    }
    std::cout << "总薪资: " << total << std::endl;
}
```

#### 场景 5：配合 `const` 和引用控制语义

```c++
// 按值拷贝（修改不影响原数据）
auto [a, b] = some_pair;

// 按引用（可修改原数据）
auto& [ref_a, ref_b] = some_pair;

// const 引用（只读，避免拷贝）
const auto& [cr_a, cr_b] = some_pair;
```

#### 场景 6：条件语句中直接解包（C++17 `if` 初始化器 + 结构化绑定）

```c++
#include <map>
#include <iostream>

std::map<int, std::string> get_data() {
    return {{1, "one"}, {2, "two"}};
}

int main() {
    auto data = get_data();

    // 在 if 条件中直接解包查找结果
    if (auto [pos, found] = data.insert({3, "three"}); found) {
        std::cout << "插入成功: " << pos->second << std::endl;
    }

    // 在 if 条件中解包并检查
    if (auto [it, ok] = data.insert({1, "ONE"}); !ok) {
        std::cout << "已存在: " << it->second << std::endl;  // "one"（因为已有 key=1）
    }
}
```

### 注意事项与限制

| 注意事项/限制 | 说明 |
|:---|:---|
| **必须用 `auto`** | 不能指定绑定变量的类型（`int [a,b] = expr` 非法），由编译器自动推导 |
| **变量数量必须匹配** | 绑定数量必须与结构体成员/数组大小/tuple 元素数完全一致 |
| **不能嵌套解包** | `auto [[a,b], c] = expr;` C++17 非法，需分两步 |
| **不能忽略部分元素** | 没有类似 `std::ignore` 的机制（C++26 引入 `_` 前只能解包全部） |
| **不能对函数返回值取 `&`** | `auto& [a,b] = func();` 右值不可绑定到左值引用，需 `auto&&` |
| **不能用于 `union`** | 联合体不能用结构化绑定 |
| **不引入新的作用域** | 绑定变量与隐藏变量生命周期绑定 |
| **修饰符作用于隐藏变量而非绑定** | `const auto& [a,b] = expr` 中的 `const` 作用于隐藏变量 `__e`，而非 `a`、`b` 本身 |
| **性能与手写等价** | 编译器将绑定优化为直接访问，零开销抽象 |

### 结构化绑定 vs std::tie 对比

```c++
// 结构化绑定：声明新变量
auto [a, b] = func();     // ✅ 最简洁，声明即解包

// std::tie：给已有变量赋值
int x, y;
std::tie(x, y) = func();  // ✅ 需要已有变量时

// 忽略部分值
auto [a, b, c] = func();    // ❌ 不能忽略，必须声明 3 个变量
std::tie(a, std::ignore, c) = func(); // ✅ 用 std::ignore 忽略

// 引用语义
auto& [ra, rb] = pair_obj;  // ✅ 结构化绑定支持引用
int& ra2 = std::get<0>(pair_obj); // ⚠️ tie 不直接支持引用绑定

// 实现比较运算符
// std::tie 胜出
bool operator<(const T& o) const {
    return std::tie(a, b, c) < std::tie(o.a, o.b, o.c);
}
```

### 最佳实践总结

1. **优先用 `const auto&`** 遍历容器/结构体，避免不必要的拷贝
2. **map 遍历时务必用引用**：`for (const auto& [k, v] : map)` 避免拷贝键值对
3. **需要修改时用 `auto&`**：`for (auto& [k, v] : map) v = process(v);`
4. **小量值可用 `auto`**：如 `auto [x, y] = func()` 对基础类型按值拷贝开销可忽略
5. **`if` 初始化器 + 结构化绑定**是处理插入/查找结果的推荐模式
6. **与 `std::tie` 配合**：声明新变量用结构化绑定，已有变量赋值用 `tie`

---

### Selection Statements with Initializer（带初始化器的选择语句）

### 为什么需要这个特性？

C++17 之前，很多常见模式需要额外的作用域大括号或提前声明变量，导致代码多一层缩进、变量生命周期被不必要地延长。

```c++
// C++11/14 —— 锁 + 条件检查需要额外的大括号
{
    std::lock_guard<std::mutex> lk(mx_);
    if (v_.empty()) {
        v_.push_back(val);
    }
}   // 大括号缩进额外一层，且容易忘记

// 或者提前声明变量 —— 生命周期被不必要地延长
auto result = find_value(key);  // result 在 if 之后仍然存活
if (result != nullptr) {
    use(result);
}
// result 在这里还能被访问，容易误用
```

### 新语法格式

C++17 为 `if` 和 `switch` 增加了**初始化语句**部分，位于条件之前：

```
if (init; condition)         →  if (初始化; 条件) { ... }
if (init; condition) else    →  if (初始化; 条件) { ... } else { ... }
switch (init; condition)     →  switch (初始化; 条件) { ... }
```

语法拆解：

```c++
// ┌───── 初始化语句（可以是声明或表达式，末尾有分号）
// │      ┌──── 条件（bool 表达式或变量声明）
// │      │
if (auto x = get_value(); x > 0) {
    // x 在此作用域内可用
}
// x 在此处已销毁
```

初始化部分可以是：

| 类型 | 示例 |
|:---|:---|
| **变量声明** | `if (int x = foo(); x > 0)` |
| **对象声明** | `if (std::lock_guard<std::mutex> lk(mx); v.empty())` |
| **结构化绑定声明** | `if (auto [it, ok] = m.insert(kv); ok)` |
| **纯表达式** | 虽然语法允许，但不常见 |

关键行为：**初始化语句中声明的变量，其生命周期与整个 `if`/`switch` 语句绑定**，包括 `else` 分支。

```c++
if (auto x = get_value(); x > 0) {
    // x 可用
} else {
    // x 也在这里可用（比如打印错误信息）
}
// x 在此处销毁
```

### 使用场景

#### 场景 1：带锁的条件检查

```c++
#include <mutex>
#include <vector>

std::mutex mtx_;
std::vector<int> v_;

void append_if_empty(int val) {
    // C++17 —— 锁的生命周期绑定到 if 语句
    if (std::lock_guard<std::mutex> lk(mtx_); v_.empty()) {
        v_.push_back(val);
    }
    // 此处锁已释放
}
```

#### 场景 2：map/set 插入结果检查

```c++
#include <map>
#include <string>

std::map<int, std::string> cache;

void update_cache(int key, const std::string& val) {
    // 结构化绑定 + 初始化器的组合
    if (auto [pos, inserted] = cache.insert({key, val}); !inserted) {
        pos->second = val;  // 已存在，更新值
    }
    // pos 和 inserted 在此处已销毁
}
```

#### 场景 3：带条件的资源获取

```c++
#include <fstream>
#include <string>

std::string read_file(const std::string& path) {
    // 文件流 + 条件检查
    if (std::ifstream file(path); file.is_open()) {
        std::string content, line;
        while (std::getline(file, line)) {
            content += line + "\n";
        }
        return content;
    }
    return {};
    // file 在此处已自动关闭
}
```

#### 场景 4：switch 中的临时对象

```c++
#include <iostream>

enum class Status { OK, Bad, Unknown };

Status process(int code) {
    return code > 0 ? Status::OK : Status::Bad;
}

void handle(int input) {
    switch (Status s = process(input); s) {
        case Status::OK:
            std::cout << "成功" << std::endl;
            break;
        case Status::Bad:
            std::cout << "失败" << std::endl;
            break;
        default:
            std::cout << "未知" << std::endl;
            break;
    }
    // s 在此处已销毁
}
```

#### 场景 5：复杂条件多次计算

```c++
#include <vector>
#include <algorithm>

std::vector<int> data = {3, 1, 4, 1, 5, 9, 2, 6};

void process() {
    // 排序 + 检查 —— 排序结果仅在 if 内有效
    if (std::sort(data.begin(), data.end()); !data.empty()) {
        auto first = data.front();
        auto last = data.back();
        // 处理已排序的数据...
    }
    // data 仍然是排序后的状态，但初始化器的目的已达成
}
```

### 旧版本（C++11/14）如何实现

#### 方式 1：额外的大括号作用域

```c++
// C++17
if (std::lock_guard<std::mutex> lk(mx); v.empty()) {
    v.push_back(val);
}

// C++11/14 —— 手动加作用域
{
    std::lock_guard<std::mutex> lk(mx);
    if (v.empty()) {
        v.push_back(val);
    }
}
```

#### 方式 2：提前声明变量，扩大生命周期

```c++
// C++17
if (auto result = find_value(key); result != nullptr) {
    use(result);
}

// C++11/14 —— 变量提前声明，作用域被不必要地扩大
auto result = find_value(key);
if (result != nullptr) {
    use(result);
}
// result 在这里仍然存活，容易误用
```

#### 方式 3：IIFE（立即执行 lambda）

```c++
// C++11/14 —— 用 IIFE 模拟精准作用域
[&] {
    auto result = find_value(key);
    if (result != nullptr) {
        use(result);
    }
}();  // 立即执行，result 在此处销毁
```

### 注意事项

| 注意事项 | 说明 |
|:---|:---|
| **初始化部分必须有分号** | `if (cond)` 仍然是传统用法，`if (;cond)` 不符合直觉，但语法允许 |
| **与结构化绑定配合最佳** | `if (auto [it, ok] = m.insert(kv); ok)` 是 C++17 的经典组合 |
| **声明变量只能在当前 if/switch 使用** | 不能在 else if 的初始化部分再次声明同名变量 |
| **性能无额外开销** | 编译器展开后的代码与手写作用域等价 |

---

### Constexpr If（编译期条件分支）

### 特性说明

`if constexpr` 是 C++17 引入的**编译期条件分支**——条件在编译时求值，编译器只保留满足条件的分支，丢弃另一个分支的代码。

### 区分两个 constexpr

```c++
template <typename T>
constexpr bool isIntegral() {         // ← 这是 constexpr 函数（C++11）
  if constexpr (std::is_integral<T>::value) {  // ← 这是 constexpr if（C++17）
    return true;
  } else {
    return false;
  }
}
```

| | `constexpr` 函数（C++11） | `if constexpr`（C++17） |
|:---|:---|:---|
| 作用 | 函数可在编译期求值 | 分支在编译期选择 |
| 语法位置 | 函数返回类型前 | `if` 关键字后 |
| 对未走分支的处理 | 完全实例化 | **直接丢弃**，不生成代码 |

### 和普通 `if` 的本质区别

```c++
template <typename T>
void example(T v) {
    if constexpr (std::is_integral_v<T>) {
        // 仅当 T 是整数时编译
        v += 1;
        std::cout << "整数: " << v << std::endl;
    } else {
        // 仅当 T 不是整数时编译
        for (auto& x : v) x += 1;  // 需要 T 有迭代器
        std::cout << "容器: " << v.size() << std::endl;
    }
}

example(42);                    // ✅ 只编译整数分支
example(std::vector<int>{1,2}); // ✅ 只编译容器分支
```

如果用普通 `if`：

```c++
template <typename T>
void example(T v) {
    if (std::is_integral_v<T>) {         // ❌ 两个分支都会被编译
        v += 1;                          // vector 没有 += 1 → 编译错误
    } else {
        for (auto& x : v) x += 1;        // int 没有 begin/end → 编译错误
    }
}
```

普通 `if` 的两个分支都会被编译器实例化，导致无论如何都会编译失败。**`if constexpr` 的核心价值就是让语法上不合法的分支不会产生编译错误。**

### 语法要点

```c++
// 基本形式
if constexpr (condition) { ... }
if constexpr (condition) { ... } else { ... }
if constexpr (condition) { ... } else if constexpr (condition) { ... }

// condition 必须是编译期可求值的常量表达式
// 可以是：bool 常量、type_trait、sizeof、constexpr 函数返回值等

// 可混合普通 if 使用
if constexpr (cond1) {
    // ...
} else if (runtime_cond) {  // ② 普通 if
    // ...
}
```

### 使用场景

#### 场景 1：按类型选择不同实现（最核心用途）

```c++
#include <type_traits>
#include <vector>
#include <list>
#include <iostream>

template <typename Container>
void print_size(const Container& c) {
    if constexpr (std::is_same_v<Container, std::vector<int>>) {
        std::cout << "vector, size = " << c.size() << std::endl;
    } else if constexpr (std::is_same_v<Container, std::list<int>>) {
        std::cout << "list, size = " << c.size() << std::endl;
    } else {
        std::cout << "other, size = " << c.size() << std::endl;
    }
}
```

#### 场景 2：SFINAE 的简洁替代

C++11/14 中按类型选择实现需要 `enable_if` 或标签分发：

```c++
// C++11/14 —— 需要两个重载 + enable_if
template <typename T>
std::enable_if_t<std::is_integral_v<T>, void> process(T v) {
    // 整数版本
}

template <typename T>
std::enable_if_t<!std::is_integral_v<T>, void> process(T v) {
    // 非整数版本
}

// C++17 —— 一个函数搞定
template <typename T>
void process(T v) {
    if constexpr (std::is_integral_v<T>) {
        // 整数版本
    } else {
        // 非整数版本
    }
}
```

#### 场景 3：编译期递归终止

代替 C++11 的模板特化递归终止：

```c++
// C++11 —— 需要两个模板
template <typename T>
T sum(T v) { return v; }  // 递归终止（单独的函数重载）

template <typename T, typename... Args>
T sum(T first, Args... rest) {
    return first + sum(rest...);
}

// C++17 —— 一个函数就够了
template <typename T, typename... Args>
auto sum(T first, Args... rest) {
    if constexpr (sizeof...(rest) == 0) {
        return first;  // 无参数时直接返回
    } else {
        return first + sum(rest...);
    }
}
```

#### 场景 4：编译期返回不同类型

```c++
#include <type_traits>
#include <string>

template <typename T>
auto convert(T value) {
    if constexpr (std::is_arithmetic_v<T>) {
        return std::to_string(value);  // 返回 std::string
    } else {
        return value;  // 返回 T 本身
    }
}

// 注意：C++17 要求 if constexpr 的所有 return 类型必须一致（或可推导为同一类型）
// C++17 之前做不到这个——因为不同分支返回不同类型需要 `auto` 返回类型推导
```

#### 场景 5：编译期调试/断言

```c++
template <typename T>
void debug_print(const T& v) {
    if constexpr (std::is_same_v<T, std::string>) {
        std::cout << "字符串: " << v << std::endl;
    } else if constexpr (std::is_arithmetic_v<T>) {
        std::cout << "数值: " << v << std::endl;
    } else {
        static_assert(sizeof(T) == 0, "不支持的类型");  // ❌ 注意！这不会触发！
        // 因为 static_assert(false) 无论哪个分支都会触发编译错误
        // 正确做法：
    }
}

// 正确的编译期类型检查
template <typename T>
void checked_print(const T& v) {
    if constexpr (std::is_same_v<T, std::string>) {
        std::cout << v << std::endl;
    } else {
        static_assert(always_false_v<T>, "不支持的类型");
    }
}

template <typename T>
inline constexpr bool always_false_v = false;  // 依赖模板参数，不会被立即求值
```

### 常见陷阱

| 陷阱 | 说明 |
|:---|:---|
| **`static_assert(false)` 始终会触发** | 即使在不走的分支中也会编译失败，应使用 `static_assert(always_false_v<T>)` |
| **`return` 类型必须一致** | 所有分支的返回类型必须相同或能推导为同一类型 |
| **不能替代运行时 `if`** | 条件是运行时变量时不能用 `if constexpr` |
| **不是预处理器** | `if constexpr` 的未选中分支仍然会做**语法检查**（只是不生成代码） |

---

### Coroutine（协程）

### 特性说明

**协程（coroutine）**是可以暂停和恢复执行的特殊函数。要定义协程，函数体中必须存在 `co_return`、`co_await` 或 `co_yield` 关键字。C++20 的协程是**无栈**的；除非编译器优化掉，否则它们的状态分配在堆上。

```c++
// 生成器：每次调用产生一个值
generator<int> range(int start, int end) {
  while (start < end) {
    co_yield start;
    start++;
  }
}
for (int n : range(0, 10)) { /* ... */ }

// 异步任务
task<int> calculate() {
  co_return 42;
}
auto result = calculate();
co_await result;  // == 42
```

### C++20 协程的完善度

**结论：语言机制是完整的，但标准库配套"半成品"。** 这也是社区对 C++20 协程最集中的批评——委员会交付了"发动机"，却没交付"整车"。

| 层面 | 状态 | 说明 |
|:---|:---|:---|
| **语言核心** | ✅ 完整 | `co_await` / `co_yield` / `co_return`、`coroutine_handle`、promise/awaiter 机制 |
| **同步生成器** | ✅ C++23 才有 | `std::generator<T>` 直到 C++23 才进标准库 |
| **异步任务 `task`** | ❌ 至今没有 | C++20/23 都没有标准 `task<T>`，必须第三方或自己写 |
| **事件循环/执行器** | ❌ 没有 | 语言不提供调度器，全靠用户接入（io_uring、libuv、线程池…） |
| **调试支持** | ⚠️ 有改善 | GDB/LLDB 对协程帧的调试支持较新，仍不如普通函数 |
| **社区库** | ✅ 活跃 | cppcoro（Lewis Baker）、libunifex（Microsoft）等 |

**说白了**：C++20 协程是**"自定义点全集"**——promise、awaiter、frame 分配、恢复逻辑全部开放给你，但开箱即用的东西几乎没有。写一个 `task` 你需要自己实现 `promise_type`、`initial_suspend`、`final_suspend`、`await_suspend` 等一堆样板代码。

### 与 Rust 对比：内部也是状态机吗？

**是。两者编译后本质上都是状态机**，但展开的"所有权"和模型不同。

#### C++ 的机制

编译器把协程函数体改造成状态机，并把**所有局部变量 + 参数 + 当前挂起点**打包进一个**协程帧（coroutine frame）**：

```
协程函数
  ↓ 编译器改写
┌─────────────────────────────┐
│  协程帧（默认堆分配）          │
│  ├── promise 对象（用户定义） │
│  ├── 参数、局部变量           │
│  └── 当前挂起点的序号          │
└─────────────────────────────┘
```

关键自定义点：
- **`promise_type`**：定义启动/返回值/异常行为
- **`await_suspend(handle)`**：挂起后由谁、何时恢复——**这就是你接入"执行器/事件循环"的钩子**。语言不替你调度，恢复完全由你写的代码决定
- **frame 分配**：默认堆分配，可通过 promise 自定义 `operator new` 用内存池

所以 C++ 的恢复模型是**"谁想恢复谁就恢复"**——完全开放，但也意味着你必须自己保证线程安全、生命周期。

#### Rust 的机制

Rust 的 `async fn` 同样编译成**匿名枚举状态机**，但模型是**"轮询式（poll-based）"**：

```
async fn → 编译器生成一个 Future（枚举状态机）
Future::poll(ctx)
  ├── 可推进 → 返回 Poll::Ready(value)
  └── 阻塞   → 返回 Poll::Pending + 注册 Waker
                执行器（tokio）随后驱动其他任务
```

#### 对比表

| | C++20 | Rust |
|:---|:---|:---|
| 状态存储 | **协程帧，默认堆分配** | Future 是一个值，默认**栈上**（除非 boxed） |
| 驱动方式 | `await_suspend` 自定义恢复逻辑 | `poll` 由执行器反复调用 |
| 调度器 | 语言不管，用户自接 | 语言不管，第三方执行器（tokio/async-std） |
| 标准库配套 | 无 `task`，无执行器 | `Future`/`Pin`/`Waker` 在 std，执行器在第三方 |
| 惰性 | 可配置（`initial_suspend`） | **天然惰性**——没人 poll 就不执行 |
| 线程安全 | 无内建机制，手动管理 | `Send`/`Sync` auto-trait 编译期检查 |
| 典型开销 | 每次协程可能一次堆分配 | 无分配（栈上状态机） |

**一句话总结**：C++ 是"挂起后，你自己决定谁来 resume"；Rust 是"挂起后，执行器不断 poll 你，直到你 Ready"。Rust 把模型收敛成 `poll`，配套和安全性更完整；C++ 把模型完全开放，灵活但费手。

### 完整示例：异步流水线（"查用户资料 → 查其文章数 → 汇总加分"）

#### C++（C++20，需要自己写 task 骨架）

```cpp
#include <coroutine>
#include <iostream>
#include <thread>
#include <chrono>
#include <optional>
#include <functional>
#include <mutex>
#include <queue>
#include <condition_variable>

// ========== 极简事件循环（模拟 I/O 完成 → 恢复协程） ==========
struct Executor {
    std::mutex m;
    std::condition_variable cv;
    std::queue<std::function<void()>> jobs;
    void post(std::function<void()> f) {
        { std::lock_guard lk(m); jobs.push(std::move(f)); }
        cv.notify_one();
    }
    void run() {
        for (;;) {
            std::function<void()> f;
            { std::unique_lock lk(m);
              cv.wait(lk, [&] { return !jobs.empty(); });
              f = std::move(jobs.front()); jobs.pop(); }
            f();
        }
    }
};
Executor g_exec;

// ========== 模拟异步 I/O 的 awaiter ==========
struct async_io {
    int result; int delay_ms; std::coroutine_handle<> caller;
    bool await_ready() noexcept { return false; }
    void await_suspend(std::coroutine_handle<> h) {
        caller = h;
        std::thread([this] {
            std::this_thread::sleep_for(std::chrono::milliseconds(delay_ms));
            g_exec.post([this] { caller.resume(); });   // I/O 完成后恢复协程
        }).detach();
    }
    int await_resume() { return result; }
};

// ========== 极简 task<T>（C++20 没有标准 task，必须自己写） ==========
template <typename T>
struct Task {
    struct promise_type {
        std::optional<T> result;
        std::coroutine_handle<> continuation;
        Task get_return_object() {
            return Task{std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        std::suspend_always initial_suspend() noexcept { return {}; }  // 惰性
        std::suspend_always final_suspend() noexcept {
            if (continuation) continuation.resume();   // 完成后唤醒等待者
            return {};
        }
        void return_value(T v) { result = std::move(v); }
        void unhandled_exception() { std::terminate(); }
    };
    std::coroutine_handle<promise_type> h;
    explicit Task(std::coroutine_handle<promise_type> h) : h(h) {}
    Task(Task&& o) noexcept : h(o.h) { o.h = {}; }
    ~Task() { if (h) h.destroy(); }
    Task(const Task&) = delete;
    Task& operator=(const Task&) = delete;

    bool await_ready() noexcept { return false; }
    void await_suspend(std::coroutine_handle<> caller) {
        h.promise().continuation = caller;
        h.resume();              // 启动被等待的子协程
    }
    T await_resume() { return *h.promise().result; }
};

// ========== 业务协程 ==========
Task<int> fetch_user_posts_score(int uid) {
    int profile = co_await async_io{uid * 10, 100};   // "查用户资料"
    int posts   = co_await async_io{profile * 2, 150};// "查文章数"
    co_return profile + posts;
}

Task<int> compute(int uid) {
    int a = co_await fetch_user_posts_score(uid);     // 等待子协程
    int b = co_await async_io{a + 1, 50};             // 再一次异步
    co_return a + b;
}

// ========== 驱动 ==========
int main() {
    g_exec.post([&] {
        auto t = compute(42);   // 创建惰性协程
        t.h.resume();           // 启动
    });
    g_exec.run();               // 事件循环（简化：永久运行）
}
```

要点：**恢复逻辑、事件循环、task 全部是你自己拼的**——这就是"C++ 给机制、不给成品"的体现。

#### Rust（tokio 实现同样逻辑）

```rust
use tokio::time::{sleep, Duration};

// 模拟异步 I/O
async fn async_io(v: i32, d: Duration) -> i32 {
    sleep(d).await;   // tokio 定时器，返回 Pending → 事件循环驱动其他任务
    v
}

async fn fetch_user_posts_score(uid: u32) -> i32 {
    let profile = async_io(uid as i32 * 10, Duration::from_millis(100)).await;
    let posts   = async_io(profile * 2, Duration::from_millis(150)).await;
    profile + posts
}

async fn compute(uid: u32) -> i32 {
    let a = fetch_user_posts_score(uid).await;   // 等待子 future
    let b = async_io(a + 1, Duration::from_millis(50)).await;
    a + b
}

#[tokio::main]
async fn main() {
    let result = compute(42).await;
    println!("{result}");
}
```

#### 对比总结

| 对比项 | C++20 | Rust (tokio) |
|:---|:---|:---|
| **要写多少骨架** | task/promise/awaiter/执行器**全手写**（≈80 行） | 只需业务函数 + `#[tokio::main]` |
| **await 子任务** | `co_await fetch_user_posts_score(uid)` | `.await` |
| **标准库/生态** | 无 task 无执行器，靠 cppcoro/libunifex | tokio 提供运行时，`.await` 即用 |
| **状态机** | 编译器生成 + 堆帧 + 自定义恢复 | 编译器生成栈上 enum + poll |
| **安全** | 手动管线程/生命周期/异常 | 借用检查 + `Send` 自动检查 |
| **性能** | 帧可能堆分配，无内建优化 | 通常零分配 |

**结论**：C++20 协程把"积木"都给你了，但要搭出能用的异步框架你得自己动手（或引入 cppcoro/libunifex）；Rust 把积木和常见的框架都备好了，写起来更像"业务代码"，代价是模型更收敛、更不易自由发挥。两者底层都是状态机，区别在于**谁负责调度和恢复的约定**。

---

### Concept（概念）

### 特性说明

**Concept（概念）**是命名的编译时谓词，用于约束类型。它们采用以下形式：

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

### 提出的背景

**痛点：C++ 模板的约束没有"语法上的位置"。** C++20 之前，模板参数列表里只能写"参数"（`typename T`），不能写"我对 T 有什么要求"。于是：

- 约束写在**函数体内部**（`static_assert`），报错晚、报错烂
- 或者用 **SFINAE 技巧**硬凑，代码难以阅读
- **错误信息灾难**——模板实例化层层嵌套，编译器报出一屏"未满足 xxx 的约束"，跟实际原因毫无关系

C++ 委员会从 2003 年就开始提案（最早叫 *Concepts Lite*），经历了漫长反复，直到 C++20 才落地。核心目标：

1. **在模板签名处直接声明约束**，可读、可复用
2. **编译错误信息友好**——"类型不满足 concept"，而不是一屏模板栈
3. **参与重载决议**——基于约束选择正确的重载

### 与 Rust 的类比：完全可以

Rust 的 **trait bound** 和 C++20 的 **concept** 高度对应：

```rust
// Rust：T 必须实现 Display 和 Clone
fn print<T: Display + Clone>(t: T) { ... }
```

```c++
// C++20：T 必须满足 integral（内置概念）
template<std::integral T>
void print(T t) { ... }

// 或 requires 子句组合多个概念
template<typename T>
requires std::integral<T> && std::copyable<T>
void print(T t) { ... }
```

| | Rust | C++20 |
|:---|:---|:---|
| 约束声明 | `T: Trait` | `std::concept T` 或 `requires` |
| 组合 | `T: A + B` | `concept C = A<T> && B<T>` |
| 定义 | `trait Display { ... }` | `concept integral = std::is_integral_v<T>` |
| 作用时机 | 编译期 | 编译期 |
| 参与重载 | 通过 trait 实现选择 | 通过约束选择 |

**最大区别**：Rust 的 trait 同时携带**实现**（方法），concept 只表达**要求**（谓词），不携带实现——C++ 的实现仍靠模板/继承。

### 没有 Concept 之前，C++ 怎么约束？

#### 方式一：函数体内 `static_assert`（最直白，但最"晚"）

```c++
template<typename T>
void print(T t) {
    static_assert(std::is_integral_v<T>,
                  "T 必须是整数类型");
    // 编译到这里才检查，且不参与重载决议
}
```

缺点：约束在**实例化时**才生效，无法参与函数重载选择；只是"事后检查"。

#### 方式二：SFINAE + `std::enable_if`（C++11 经典手法）

```c++
// 只有 T 是整数时，这个重载才存在
template<typename T>
std::enable_if_t<std::is_integral_v<T>, void>
print(T t) { /* 整数版本 */ }

// 只有 T 不是整数时，这个重载才存在
template<typename T>
std::enable_if_t<!std::is_integral_v<T>, void>
print(T t) { /* 非整数版本 */ }
```

**SFINAE 原理**：模板替换失败不是错误（Substitution Failure Is Not An Error），`enable_if` 在条件为假时让替换失败，于是该重载被悄悄"剔除"。

缺点：
- **可读性灾难**——真正的签名被 `enable_if_t<..., void>` 淹没
- 条件越长越难写，逻辑反直觉（要写"非"）
- 错误信息依然糟糕

#### 方式三：标签分发（Tag Dispatch）

利用重载决议选择不同的"标签类型"：

```c++
struct integral_tag {};
struct other_tag {};

// 内部两个重载
template<typename T>
void print_impl(T t, integral_tag) { /* 整数 */ }
template<typename T>
void print_impl(T t, other_tag)    { /* 非整数 */ }

// 外部根据类型特征挑标签
template<typename T>
void print(T t) {
    print_impl(t,
        std::conditional_t<std::is_integral_v<T>,
                           integral_tag, other_tag>{});
}
```

缺点：一个功能拆成三份代码，绕。

#### 方式四：C++17 的 `if constexpr`（缓解但仍是"事后"）

```c++
template<typename T>
void print(T t) {
    if constexpr (std::is_integral_v<T>) { /* 整数 */ }
    else                                 { /* 非整数 */ }
}
```

这比 `enable_if` 清晰多了，但它仍然**不能约束签名**、**不参与重载决议**——本质是函数体内的分支，不是"接口级别的约束"。

### 对比总结

| 约束方式 | 版本 | 参与重载决议 | 约束在签名处 | 错误信息 |
|:---|:---|:---:|:---:|:---|
| `static_assert` | C++11 | ❌ | ❌ | 尚可（可自定义消息） |
| `enable_if` SFINAE | C++11 | ✅ | ✅（但丑） | ❌ 灾难 |
| 标签分发 | 老技巧 | ✅ | ❌ | 一般 |
| `if constexpr` | C++17 | ❌ | ❌ | 尚可 |
| **Concept** | **C++20** | ✅ | ✅ | ✅ 友好 |

```c++
// C++11 enable_if 的痛苦写法
template<typename T, typename = std::enable_if_t<
    std::is_integral_v<T> && !std::is_same_v<T, bool>>>
void f(T) {}

// C++20 concept 的清爽写法
template<std::integral T> requires (!std::same_as<T, bool>)
void f(T) {}
```

**一句话**：Concept 是"把约束从函数体/返回类型搬到签名上"的语法革命——它取代了 `enable_if` 那套绕来绕去的技巧，让 C++ 模板约束的表达方式和 Rust 的 trait bound 一样直接。

---

### Three-Way Comparison（三路比较，`<=>`）

### 特性说明

C++20 引入了太空船运算符（`<=>`），一次返回 `-1/0/1` 式的**三路结果**，定义它后编译器**自动生成** `==`、`!=`、`<`、`<=`、`>`、`>=` 全部六个比较运算符。这是 C++20 最受欢迎的特性之一。

```c++
struct foo {
  int a;
  bool b;
  char c;
  friend auto operator<=>(const foo&) const = default;
};

// 有了 <=> 之后自动获得：
// foo == foo、foo != foo、foo < foo、foo <= foo、foo > foo、foo >= foo
// 一行声明，六个运算符
```

**三种排序类型：**

| 类型 | 语义 | 示例 |
|:---|:---|:---|
| `std::strong_ordering` | 强排序：相等即完全相同（可互换） | 整数、`<=>` 全默认的简单类型 |
| `std::weak_ordering` | 弱排序：等价但不完全相同（比较上可互换） | 忽略大小写的字符串比较 |
| `std::partial_ordering` | 偏序：存在**不可比较**的情况 | 浮点数（NaN）、部分有序的数据 |

### 出现原因：C++17 及之前的比较运算符噩梦

C++17 之前，要实现一个支持排序的类型，必须手写**六个**运算符，而且它们之间没有任何语法上的关联——改一个忘了其他，就容易出现不一致（比如 `a < b` 和 `b > a` 结果矛盾）：

```c++
// C++17 及之前 —— 手写 6 个运算符，又臭又长又容易不一致
struct Person {
    std::string name;
    int age;

    friend bool operator==(const Person& a, const Person& b) {
        return a.name == b.name && a.age == b.age;
    }
    friend bool operator!=(const Person& a, const Person& b) {
        return !(a == b);
    }
    friend bool operator<(const Person& a, const Person& b) {
        return std::tie(a.name, a.age) < std::tie(b.name, b.age);
    }
    friend bool operator<=(const Person& a, const Person& b) {
        return !(b < a);
    }
    friend bool operator>(const Person& a, const Person& b) {
        return b < a;
    }
    friend bool operator>=(const Person& a, const Person& b) {
        return !(a < b);
    }
};
```

痛点：
1. **模板代码爆炸**——每个要排序的类型都要写 6 个运算符
2. **容易不一致**——`==` 和 `<` 分开实现，可能违背"相等则既不小于也不大于"的数学公理
3. **性能浪费**——比较两个对象可能多次遍历成员
4. **无标准方法**——`std::tie` 只是"技巧"，不是语言特性

### C++17 的"替换实现"：std::tie 字典序

C++17 里最常用的替代方案是 `std::tie`——把多个成员打包成 tuple，利用 tuple 已有的 `operator<` 做字典序比较：

```c++
// C++17 的常见写法 —— 用 std::tie 减少一半代码
struct Person {
    std::string name;
    int age;

    // 仍然要手写 6 个，但至少 < 可以用 tie 一行实现
    friend bool operator==(const Person& a, const Person& b) {
        return a.name == b.name && a.age == b.age;
    }
    friend bool operator<(const Person& a, const Person& b) {
        return std::tie(a.name, a.age) < std::tie(b.name, b.age);
    }
    friend bool operator!=(const Person& a, const Person& b) { return !(a == b); }
    friend bool operator<=(const Person& a, const Person& b) { return !(b < a); }
    friend bool operator>(const Person& a, const Person& b)  { return b < a; }
    friend bool operator>=(const Person& a, const Person& b) { return !(a < b); }
};
```

对比 C++20：

```c++
// C++20 —— 一行搞定所有
struct Person {
    std::string name;
    int age;
    auto operator<=>(const Person&) const = default;
};
```

### 最佳实践

**1. 需要全序比较 → 用 `<=> = default` 自动生成**

```c++
struct Point {
    int x, y;
    auto operator<=>(const Point&) const = default;
    // 自动生成全部 6 个运算符，按成员声明顺序做字典序比较
};
```

**2. 只需相等比较 → 单独用 `== = default`（C++20 允许）**

如果只需要 `==`/`!=`（不需要排序），可以只声明 `==`，`!=` 自动生成：

```c++
struct ID {
    std::string value;
    bool operator==(const ID&) const = default;
    // == 和 != 自动生成，不用写 < > 等
};
```

> 注：C++20 里 `!=` 会自动从 `==` 生成，反之亦然。

**3. 自定义语义（弱排序）→ 手动实现 `<=>`**

忽略大小写的字符串比较：

```c++
struct CaseInsensitiveString {
    std::string s;
    std::weak_ordering operator<=>(const CaseInsensitiveString& other) const {
        return std::lexicographical_compare_three_way(
            s.begin(), s.end(), other.s.begin(), other.s.end(),
            [](char a, char b) {
                return std::toupper(a) <=> std::toupper(b);
            });
    }
    // 只写 <=> 一个，其余运算符自动生成
};
```

**4. 需要处理不可比（NaN）→ 用 `partial_ordering`**

```c++
struct Value {
    double v;
    std::partial_ordering operator<=>(const Value& other) const {
        return v <=> other.v;  // 浮点：NaN 时返回 unordered
    }
};
```

**5. `<=>` 配合 Concept 在泛型中做类型约束**

```c++
// 约束 T 必须是"可比较"的
template<typename T>
concept comparable = requires(const T& a, const T& b) {
    { a <=> b } -> std::convertible_to<std::strong_ordering>;
};

template<comparable T>
T my_max(const T& a, const T& b) {
    return (a <=> b) > 0 ? a : b;  // 用 <=> 直接比较
}
```

### 对比总结

| | C++17 之前 | C++20 |
|:---|:---|:---|
| 代码量 | 6 个运算符全手写 | `<=> = default` 一行 |
| 一致性 | 手动保证，易出错 | 编译器保证所有运算符一致 |
| 性能 | 多次遍历成员 | 一次 `<=>` 完成 |
| 自定义语义 | 每个运算符单独改 | 只改 `<=>` 一个点 |
| 不可比情况 | 无标准表示 | `partial_ordering::unordered` |
| 泛型约束 | 无法表达"可比较" | Concept + `requires` |

---

### Designated Initializer（指定初始化器）

### 特性说明

C++20 引入了 C 风格的**指定初始化器**语法——用 `.成员名 = 值` 的方式初始化聚合类型的成员。在指定初始化器列表中未显式列出的任何成员字段都将被默认初始化。

```c++
struct A {
  int x;
  int y;
  int z = 123;
};
A a {.x = 1, .z = 2}; // a.x == 1, a.y == 0, a.z == 2
```

这是 **C++20 新增**的语法，借鉴自 C99（但 C++ 更严格）。

### C++20 之前怎么做？

**1. C++11/14/17：按位置初始化（Aggregate Initialization）**

```c++
struct A {
  int x;
  int y;
  int z = 123;  // 有默认值
};

// C++11~17 —— 只能按成员声明顺序填
A a1{1, 0, 2};   // x=1, y=0, z=2
A a2{1, 2};      // x=1, y=2, z=123（z 用默认值）
A a3{1};         // x=1, y=0, z=123
```

问题：
- **必须记得成员顺序**，一改成员顺序，所有初始化点都错位
- **可读性差**——`{1, 0, 2}` 看不出哪个是 x、哪个是 z
- **无法跳过中间的成员**——想给 x 和 z 赋值，却必须把 y 也填上（即使它应该用默认值）

**2. C++17 的补救：构造函数 + 占位**

```c++
struct A {
  int x;
  int y;
  int z;
  A(int x_ = 0, int y_ = 0, int z_ = 123)
    : x(x_), y(y_), z(z_) {}
};

A a{1, {}, 2};  // 用 {} 跳过 y，但仍需占位
```

### C++20 解决了什么

```c++
A a {.x = 1, .z = 2};  // x=1, y=0(默认), z=2
```

- **按名字赋值**，顺序无关、可读性强
- **可以跳过中间字段**（y 自动用默认初始化/值初始化）
- 成员顺序调整不再破坏调用点

### 限制与注意

```c++
struct A { int x; int y; int z = 123; };

A a1{.x = 1, .z = 2};          // ✅ 按声明顺序（x 在 z 前）正确
// A a2{.z = 2, .x = 1};       // ❌ 编译错误！必须按声明顺序
// A a3{.x = 1, .x = 2};       // ❌ 不能重复指定同一字段
// A a4{.x = 1, .w = 2};       // ❌ 没有这个成员
A a5{1, .z = 2};               // ❌ 不能混合位置初始化和指定初始化
```

**关键规则**：
1. 必须**按成员声明顺序**书写
2. 不能混合 `{1, .z = 2}` 这种位置+指定混用
3. 只能用于**聚合类型**（`struct`/`class` 无用户构造函数）
4. 跳过字段的成员走**默认成员初始化器**（如 `z = 123`）或**值初始化**

### 与 C 的对比

C99 也有同样语法，但 C++ 更严格——C 允许乱序，C++ 要求按声明顺序（这是为了避免 C++ 虚基类/继承布局带来的歧义）。

### 使用场景

**场景 1：配置结构体（最常用）**

```c++
struct Config {
    int port = 8080;
    std::string host = "localhost";
    bool verbose = false;
    int timeout_ms = 3000;
};

// C++17 —— 必须按顺序填，读起来全靠猜
Config c1{8080, "localhost", false, 5000};

// C++20 —— 按名字填，只改想改的
Config c2{.port = 9090, .timeout_ms = 5000};
// 其余字段自动用默认值
```

**场景 2：只初始化部分字段（跳过中间）**

```c++
struct RGB { int r; int g; int b; };

// C++20 —— 只设红色和蓝色，绿色留 0
RGB color{.r = 255, .b = 128};
```

**场景 3：可读性强的参数聚合**

```c++
struct Point { int x; int y; };

// C++20 —— 一眼看出哪个是 x 哪个是 y
Point p1{.x = 10, .y = 20};
Point p2{.y = 20, .x = 10};  // ❌ 顺序错了
```

---

### Constexpr Virtual Function（constexpr 虚函数，C++20）

### 特性说明

C++20 允许虚函数标记为 `constexpr`，使**同一套多态代码既能编译期求值，也能运行期分派**：

```c++
struct X1 {
  virtual int f() const = 0;   // 纯虚（基类不需要 constexpr）
};

struct X2: public X1 {
  constexpr virtual int f() const { return 2; }  // override 是 constexpr
};

constexpr X2 x2;   // 编译期对象
x2.f();            // == 2，编译期求值
```

### 回顾：虚函数是什么？

**虚函数（virtual function）** 是实现**运行时多态**（动态分派）的机制：

```c++
struct Shape {
    virtual double area() const { return 0; }  // 虚函数
    virtual ~Shape() = default;               // 虚析构（基类必须有）
};

struct Circle : Shape {
    double r;
    Circle(double r_) : r(r_) {}
    double area() const override { return 3.14159 * r * r; }
};

// 关键：用基类指针/引用调虚函数，按【运行时实际类型】分派
Shape* s = new Circle(2.0);
s->area();   // 调用的是 Circle::area，尽管 s 的静态类型是 Shape*
```

| 要点 | 说明 |
|:---|:---|
| **动态分派** | 通过基类指针/引用调用，运行时查 **vtable（虚函数表）** 找到实际重写函数 |
| **`override`**（C++11） | 显式声明"这是重写"，写错签名编译器会报错 |
| **纯虚函数** `= 0` | 使类成为抽象类，不能实例化，强制派生类实现 |
| **虚析构** | 基类析构必须是虚的，否则 `delete` 基类指针时派生类析构不被调用 → UB |
| **运行期开销** | 每次虚调用多一次间接寻址（vtable 查表），一般可忽略 |

**核心价值**：一套代码通过基类接口操作，运行时自动分派到不同实现——这是"开闭原则"的基础。

### 为什么需要？之前版本怎么做？

**痛点：`constexpr` 和 `virtual` 原本互斥。** C++20 之前，虚函数不能是 `constexpr`，意味着**编译期计算只能用非虚的静态/自由函数**，无法在编译期使用多态。想要"编译期多态"，只能绕路：

**方式一：模板（编译期静态分派）** —— 但失去虚函数的扩展性

```c++
// 模板：编译期静态分派，但新增类型要改模板/重载
template <typename Shape>
double area(const Shape& s) { return s.area(); }
// 运行时想要多态容器（存不同形状）就很麻烦
```

**方式二：手写 if-else 按类型分支** —— 每加一种类型都要改分支

```c++
enum class Kind { Circle, Square };
struct Shape { Kind kind; };
double area(const Shape& s) {
    if (s.kind == Kind::Circle)      return /* 圆面积 */;
    else if (s.kind == Kind::Square) return /* 方面积 */;
    // 新增类型必须回来改这个函数！
}
```

### C++20 解决了什么

允许虚函数标记 `constexpr`，使同一套多态代码**既能编译期求值，也能运行期分派**——不需要写两套。

### 关键机制：分派方式取决于"求值上下文"

| 调用方式 | 分派方式 | 说明 |
|:---|:---|:---|
| 编译期求值（`constexpr` 对象 + 已知类型） | **静态分派** | 编译期类型已知，直接确定调用 `X2::f()`，无需 vtable |
| 运行期调用（基类指针/引用） | **动态分派** | 照常走 vtable，与普通虚函数无差别 |

所以 `constexpr` 虚函数是"**同一函数、两种模式**"——编译期当普通 constexpr 函数用（类型已知，静态分派），运行期当普通虚函数用（类型未知，动态分派）。

### 目的总结

1. **编译期也能用多态**——把"按类型选择行为"扩展到编译期（`static_assert`、常量计算、模板元编程）
2. **一套代码两用**——constexpr 虚函数在编译期与运行期表现一致，无需写两套
3. **平滑迁移**——已有虚函数体系加 `constexpr` 即可让其中一部分在编译期可用，无需重构

### 使用场景

**场景 1：编译期按类型计算**

```c++
struct Shape {
    virtual double area() const = 0;
    virtual ~Shape() = default;
};

struct Circle : Shape {
    double r;
    constexpr Circle(double r_) : r(r_) {}
    constexpr double area() const override { return 3.14159 * r * r; }
};

struct Square : Shape {
    double side;
    constexpr Square(double s_) : side(s_) {}
    constexpr double area() const override { return side * side; }
};

// 编译期计算（类型已知，静态分派）
constexpr Circle c{2.0};
static_assert(c.area() > 12.5);   // 编译期断言

// 运行期计算（类型未知，动态分派）
Shape* s = new Circle(2.0);
double a = s->area();             // 运行期走 vtable
```

**场景 2：编译期策略/配置对象**

```c++
struct Policy {
    virtual int weight(int x) const = 0;
};
struct FastPolicy : Policy {
    constexpr int weight(int x) const override { return x * 2; }
};
struct SafePolicy : Policy {
    constexpr int weight(int x) const override { return x + 100; }
};

// 编译期选择策略
constexpr FastPolicy fast;
static_assert(fast.weight(5) == 10);
```

### 注意事项

| 注意事项 | 说明 |
|:---|:---|
| **函数体必须满足 constexpr 约束** | 无动态分配、无运行时 I/O 等，否则不能编译期求值 |
| **编译期求值时 override 也须 constexpr** | 实际调用到的 override 必须是 constexpr，否则报错 |
| **运行时仍保留虚函数开销** | `constexpr` 只是"允许编译期求值"，不会让运行时变快 |
| **编译期可做动态分派** | C++20 中，若对象动态类型在编译期已知，虚调用也能编译期求值 |
| **与 `consteval` 的区别** | `constexpr` 虚函数可编译期也可运行期；`consteval` 必须编译期 |

### 对比 Rust

Rust 最接近虚函数的是 **trait 对象（`dyn Trait`）**——运行时动态分派；**泛型 + trait bound** 是编译期静态分派。C++20 的 constexpr 虚函数相当于把"动态多态"也带进了编译期，而 Rust 的 `const fn` 目前**不支持 trait 方法动态分派**（const fn 调 trait 方法受限）——这一点上 C++20 走得更远。

| | C++20 constexpr 虚函数 | Rust |
|:---|:---|:---|
| 运行时动态分派 | ✅（照常 vtable） | ✅ `dyn Trait` |
| 编译期静态分派 | ✅ 模板 / constexpr 虚函数 | ✅ 泛型 + trait bound |
| 编译期 + 多态结合 | ✅ constexpr 虚函数 | ⚠️ const fn 调 trait 方法受限 |
| 一套代码两用 | ✅ 编译期/运行期同函数 | 需分别用泛型和 dyn |

---

### explicit(bool)（条件式 explicit，C++20）

### 特性说明

C++20 允许 `explicit` 后面跟一个**常量表达式**，把"是否显式"变成一个**编译期可计算的开关**：

```c++
explicit(true)   // 等价于 explicit（禁止隐式转换）
explicit(false)  // 等价于没有 explicit（允许隐式转换）
explicit(某常量表达式)  // 由条件决定！
```

```c++
struct foo {
  template <typename T>
  explicit(!std::is_integral_v<T>) foo(T) {}
};

foo a = 123;   // OK    —— T=int，integral → explicit(false) → 允许隐式转换
foo b = "123"; // ERROR —— T=const char*，非 integral → explicit(true) → 禁止隐式
foo c {"123"}; // OK    —— 直接初始化，explicit 与否都行
```

### 作用：把 explicit 从"固定关键字"升级为"编译期可计算开关"

C++11 的 `explicit` 是固定的——要么显式，要么不显式。C++20 的 `explicit(常量表达式)` 让你**根据编译期条件**决定构造函数/转换函数是否 `explicit`。上面例子做的事：对**整数类型**放行隐式转换，对**非整数类型**禁止。这是"基于模板参数类型，动态决定转换策略"——把"显式性"也纳入了**元编程**的控制范围。

### 只能用于模板吗？——不是！条件只需是常量表达式

`explicit(...)` 的条件只要是一个**常量表达式**（编译期可知）即可，**不要求必须在模板里**，只是模板场景最常用：

**场景 1：由编译期常量控制（非模板）**

```c++
constexpr bool kStrict = true;  // 或来自配置宏、编译选项

struct Config {
    explicit(kStrict) Config(int x) {}   // 编译期常量决定
};

Config a{42};   // ✅ 直接初始化
// Config b = 42;  // kStrict=true 时 ❌；kStrict=false 时 ✅
```

**场景 2：基于类型特征（type_trait）**

```c++
#include <type_traits>

struct FromString {
    // 基于"另一个类型"的特征决定自身构造的显式性
    explicit(std::is_same_v<decltype(""s), std::string>) FromString(std::string) {}
};
```

**场景 3：基于类自身的编译期标志**

```c++
template <typename T>
struct SmartPtr {
    static constexpr bool kChecked = true;  // 决定显式性
    explicit(SmartPtr::kChecked) SmartPtr(T* p) : ptr(p) {}
    T* ptr;
};
```

### 使用场景与最佳实践

| 场景 | 是否模板 | 典型用途 |
|:---|:---|:---|
| 构造函数模板，按参数类型决定显式性 | ✅ 模板 | 官方示例，最常见 |
| 一个类型，不同构造策略受编译期配置影响 | ❌ 可非模板 | 少见，用宏/常量控制 |
| 转换函数按条件显式 | ✅ 通常是模板 | 按类型决定是否允许隐式转换 |

**最佳实践**：
1. **配合 `std::is_*_v` / concept 使用**最自然——条件通常是类型特征
2. **优先用 concept**（C++20）：`requires` 表达式更可读，`explicit(bool)` 适合简短条件
3. **条件必须是常量表达式**：不能依赖运行时值
4. **只在"同一个模板需要差异化转换策略"时使用**，普通固定 explicit 更清晰

### 回顾：explicit 到底拦什么？

`explicit` 构造函数拦截的是**拷贝初始化**（`=` 写法的初始化）触发的隐式转换，但**不拦截**直接初始化和显式转换：

```c++
struct Foo {
    explicit Foo(int) {}
};

Foo f1{42};       // ✅ 直接列表初始化
Foo f2(42);       // ✅ 直接初始化
Foo f3 = Foo{42}; // ✅ 显式构造后再拷贝
// Foo f4 = 42;   // ❌ 拷贝初始化，隐式转换被拦

// ⚠️ 澄清："赋值"≠"初始化"——explicit 管不了运行时赋值
Foo f4 = 42;  // 若编译过（非 explicit 时），之后 f4 = 50 是赋值，explicit 管不到
```

| 初始化/转换方式 | 写法 | explicit 构造函数 |
|:---|:---|:---:|
| 直接初始化 | `T obj(arg)` / `T obj{arg}` | ✅ 允许 |
| 拷贝初始化 | `T obj = arg` | ❌ 禁止 |
| 函数参数传参 | `void f(T); f(arg);` | ❌ 禁止（拷贝初始化语境） |
| return 返回 | `T g() { return arg; }` | ❌ 禁止 |
| 显式转换 | `static_cast<T>(arg)` | ✅ 允许 |

> **一句话**：`explicit` 拦截的是「值 → 对象」的隐式初始化转换；一旦对象已经存在，之后的**赋值**是另一回事（由 `operator=` 决定），explicit 管不了。C++20 的 `explicit(bool)` 只是把这个开关变成了"编译期可计算"。

### 对比 Rust

Rust 没有直接的 `explicit` 概念，但 `From<T>` trait 承担了"隐式转换"角色——**Rust 里根本没有隐式类型转换**（除非手动实现 `From`/`Into`）。这与 C++ 的隐式转换哲学相反：

| | C++ | Rust |
|:---|:---|:---|
| 隐式转换 | 默认可能发生，用 `explicit` 拦 | **默认不存在**，用 `From`/`Into` 显式提供 |
| 控制粒度 | `explicit(bool)` 编译期条件 | trait 实现即开关（`From` 决定能否 `.into()`） |
| 思路 | "默认允许，选择性禁止" | "默认禁止，选择性提供" |

> Rust 从设计上避免了 C++ 隐式转换的坑——你无法"意外"得到一个隐式转换，必须显式 `impl From<T>`。C++20 的 `explicit(bool)` 则是给 C++ 的隐式转换加了更精细的闸门。

---

### Immediate Function（立即函数，consteval，C++20）

### 特性说明

带 `consteval` 说明符的函数**必须产生常量**，这些称为**立即函数（Immediate Function）**——它只能在编译期求值，一旦试图在运行期调用（传入运行时变量）就直接编译错误：

```c++
consteval int sqr(int n) {
  return n * n;
}

constexpr int r = sqr(100); // OK    —— 100 是常量，编译期求值
int x = 100;
int r2 = sqr(x); // ERROR    —— x 是运行时变量，consteval 不允许运行期求值
```

**为什么 `sqr(x)` 会报错？**

关键在于 `x` 是一个**运行时变量**（非 `constexpr`），它的值对编译器来说在编译期是"未知"的：

1. `consteval` 函数**只能在编译期求值**，编译器要求传入参数必须是常量表达式
2. `x` 不是常量表达式（它是可变的局部变量），无法在编译期算出 `sqr(x)` 的值
3. 于是编译直接报错——**这是强制性的，不是警告**

对比 `constexpr` 函数：`constexpr` 函数如果参数是常量就在编译期算，参数是变量就在运行期算（可降级）；而 `consteval` **不允许降级**，运行期调用直接编译失败。

### 为什么需要？constexpr 有什么不够？

**痛点：`constexpr` 函数"不保证在编译期求值"。** 一个 `constexpr` 函数，你无法确定它是否真的在编译期执行——它取决于**调用点**的参数：

```c++
constexpr int sqr(int n) { return n * n; }

constexpr int a = sqr(100); // ✅ 编译期求值（参数是常量）
int b = sqr(x);             // ✅ 运行期求值（参数是变量）——被"降级"了
```

`constexpr` 的语义是"**可能**在编译期求值"，编译器有权选择。但有些场景你**必须**保证在编译期算出来：

| 场景 | 为什么必须编译期 |
|:---|:---|
| 模板参数 | `std::array<T, N>` 的 `N` 必须是编译期常量 |
| 静态数组大小 | `int arr[CONSTEVAL_FN()]` 需要编译期大小 |
| `static_assert` | 断言的参数必须是常量表达式 |
| 编译期查表/生成数据 | 想要零运行时开销的预计算表 |

`consteval` 把这些"必须编译期"变成**强制约束**——如果做不到就直接编译错误，从语言层面杜绝"悄悄降级到运行期"。

### constexpr / consteval / constinit 三者对比

这三个概念常被混淆，一次讲清：

| | `constexpr` 函数 | `consteval` 函数（立即函数） | `constinit` 变量 |
|:---|:---|:---|:---|
| 本质 | 可用于编译期**也可**运行期的函数 | **只能**编译期求值的函数 | 必须编译期初始化的变量 |
| 能否运行期求值 | ✅ 可以（参数为变量时） | ❌ 强制编译期 | —（只约束初始化时刻） |
| 调用运行时变量 | ✅ 允许（降级为运行期） | ❌ 编译错误 | — |
| 典型用途 | 通用常量计算 | 强制编译期计算/校验 | 全局变量要求常量初始化 |

```c++
consteval int sqr(int n) { return n * n; }        // 立即函数：只能编译期
constexpr int cube(int n) { return n * n * n; }    // constexpr：编译期/运行期都行

int x = 5;
cube(x);       // ✅ OK，运行期计算
// sqr(x);     // ❌ ERROR，consteval 拒绝运行期
constinit int g = sqr(10);  // ✅ OK，编译期初始化全局变量
```

### 使用场景

**场景 1：强制编译期计算（禁止降级）**

```c++
// 计算 2 的 N 次幂，必须编译期算
consteval unsigned long long pow2(int n) {
    return 1ULL << n;
}

constexpr auto val = pow2(30);          // 编译期算好
static_assert(val == 1073741824ULL);
// int x = 30; pow2(x);                 // ❌ 编译错误，x 不是常量
```

**场景 2：编译期生成查表数据（零运行时开销）**

```c++
consteval std::array<int, 10> make_table() {
    std::array<int, 10> table{};
    for (int i = 0; i < 10; ++i) table[i] = i * i;
    return table;
}

constexpr auto squares = make_table();  // 编译期就生成好
// squares[3] == 9，运行时无任何计算
```

**场景 3：编译期校验配置/参数**

```c++
consteval int validate_port(int port) {
    if (port < 0 || port > 65535) throw "invalid port";  // 编译期报错
    return port;
}

constexpr int server_port = validate_port(8080);   // ✅
// constexpr int bad = validate_port(-1);           // ❌ 编译期抛出
```

### 最佳实践

1. **需要"强制编译期"时用 `consteval`，否则用 `constexpr`**——`consteval` 更严格，误用会阻碍合法的运行期调用
2. **`consteval` 可以调用 `constexpr` 函数**（编译期上下文中）；但 `constexpr` 函数**默认不能调用 `consteval` 函数**（除非在编译期求值的路径上）
3. **只在确实需要零运行时开销/编译期常量的地方用**——给普通函数乱加 `consteval` 会限制调用方式
4. **与 `constexpr` 配合做"编译期强制 + 运行期兜底"**：如果希望"能编译期就编译期、不能就运行期"，用 `constexpr`；如果"必须编译期"，用 `consteval`
5. **`consteval` 是函数级约束**，不会让函数体里的代码"更优化"——它只是**限制调用时机**
6. **调试体验**：`consteval` 函数体内可抛出异常（编译期表现为编译错误），可用于编译期校验

### 对比 Rust

Rust 的 `const fn` 与 `consteval` 概念最接近——都是**只能编译期求值**。但 Rust 的 `const fn` 限制更多（不能用循环内某些特性、不能堆分配、不能打印等），C++20 的 `consteval` 允许在编译期求值中使用更丰富的 C++ 特性：

| | C++20 `consteval` | Rust `const fn` |
|:---|:---|:---|
| 只能编译期求值 | ✅ | ✅ |
| 运行期调用 | ❌ 编译错误 | ❌ 编译错误 |
| 编译期循环 | ✅ 允许 | ✅（`const fn` 支持循环） |
| 编译期堆分配 | ✅ 允许（有限制） | ❌ 不支持 |
| 运行时降级 | ❌ 强制编译期 | ❌ 强制编译期 |
| 对应概念 | 立即函数 | `const fn` + `const` 上下文 |

> **对比要点**：C++ 有 `constexpr`（可编译期也可运行期）和 `consteval`（只能编译期）两个等级；Rust 只有一个 `const fn`（只能编译期），相当于 C++ 的 `consteval`。C++ 的 `constexpr` 在 Rust 中对应的更接近普通函数 + 调用点常量上下文。

---

### Consteval If（consteval if，C++23）

> **版本标注：C++23 语言特性**（`consteval` 家族 / 编译期求值的增强）

### 特性说明

`if consteval`（C++23）是 `consteval` 的增强——它让你在同一个函数里，**根据当前是否处于常量求值环境**选择走哪条分支。写**在常量求值期间实例化的代码**：

```c++
consteval int f(int i) { return i; }   // 立即函数：只能编译期调用

constexpr int g(int i) {
  if consteval {                        // 编译期求值时走这条分支
      return f(i);                      // ✅ f 是 consteval，只能在编译期调
  } else {                              // 运行期求值时走这条分支
      return 42;                        // 运行期不能调 f，就返回别的
  }
}
```

### 为什么需要？解决了什么问题

**痛点：`constexpr` 函数想调 `consteval` 函数，但不知道当前是否在编译期。**

- `consteval` 函数只能编译期调用
- `constexpr` 函数既可能编译期求值、也可能运行期求值
- 所以 `constexpr` 函数体内**直接调用 `consteval` 函数会编译失败**——编译器不知道这个调用点是不是在编译期上下文！

C++20 之前没有干净的解法，只能绕：

```c++
// C++20 之前的绕路方案
constexpr int g(int i) {
    // ❌ 直接调 consteval 函数 → 编译错误（不确定是否编译期）
    // return f(i);

    // 绕路 1：用 std::is_constant_evaluated() 判断
    if (std::is_constant_evaluated()) {
        // 但这里调 f(i) 仍可能报错，因为条件不是编译期分支
        // ...
    }
}
```

**C++23 的 `if consteval` 直接解决**：它像 `if constexpr` 一样是**编译期分支**——编译器知道当前是不是常量求值环境，从而：
- 编译期分支里可以安全调用 `consteval` 函数
- 运行期分支里调用会报错的代码

### 与 `if constexpr` 的区别

两者都是编译期分支，但判断的"条件"不同：

| | `if constexpr (cond)` | `if consteval` |
|:---|:---|:---|
| 判断什么 | 一个**常量表达式**为真/假 | **当前是否在常量求值**（隐式条件） |
| 条件来源 | 你写的表达式 | 编译器自动判断 |
| 典型用途 | 按类型/值选择实现 | 按"编译期还是运行期"选择实现 |
| 未选中分支 | 不生成代码 | 不生成代码 |

```c++
// if constexpr —— 判断"条件"是否成立
if constexpr (std::is_integral_v<T>) { /* 整数版 */ }

// if consteval —— 判断"是否编译期"
if consteval { /* 编译期分支 */ } else { /* 运行期分支 */ }
```

### 对比 `std::is_constant_evaluated()`（C++20）

C++20 已有 `std::is_constant_evaluated()` 函数检测是否在常量求值，但**两个分支都会被编译**，且不能直接调 `consteval` 函数：

```c++
// C++20 —— std::is_constant_evaluated()
constexpr int g(int i) {
    if (std::is_constant_evaluated()) {
        // 两个分支都会被实例化！这里仍不能直接调 consteval 函数
    }
}

// C++23 —— if consteval
constexpr int g(int i) {
    if consteval {
        return f(i);   // ✅ 编译期分支，可安全调 consteval 函数
    } else {
        return 42;
    }
}
```

| | `std::is_constant_evaluated()`（C++20） | `if consteval`（C++23） |
|:---|:---|:---|
| 检测编译期 | ✅ | ✅ |
| 两个分支都实例化 | ✅ 是（普通 if） | ❌ 否（编译期分支） |
| 分支内可调 `consteval` 函数 | ❌ 不行 | ✅ 可以 |
| 语法简洁度 | 需配合普通 if | 更直接 |

### 最佳实践

1. **想在 `constexpr` 函数里安全调用 `consteval` 函数** → 用 `if consteval` 包一层
2. **编译期要不同实现、运行期要不同实现** → `if consteval` 是首选
3. **配合 `consteval` 函数**：`if consteval` 分支是调用立即函数的安全场所
4. **`if consteval` 可以单独出现**（无 `else`）：运行期分支留空即可
5. **未选中分支不生成代码**：可用于放"只在编译期/只在运行期合法"的代码
6. **与 `if constexpr` 区分**：`if constexpr` 判断条件，`if consteval` 判断环境

### 对比 Rust

Rust 没有 `if consteval` 的直接对应。Rust 的 `const fn` 是"只能编译期"，也没有"同时编译期/运行期"的 `constexpr` 等价物，因此不需要在函数体内区分"当前是否编译期"。Rust 用 `#[cfg]` 区分编译期配置，但那作用于编译目标而非求值环境——概念不同。

| | C++23 `if consteval` | Rust |
|:---|:---|:---|
| 函数内判断是否编译期 | ✅ | ❌ 无对应（`const fn` 只能编译期） |
| 编译期/运行期双实现 | ✅ 同一函数内分支 | 需分别写 `const fn` 和普通 fn |
| 编译期分支调 consteval | ✅ | — |

> **结论**：`if consteval` 是 C++ 特有（因为 C++ 有 `constexpr` 这种"可编译期可运行期"的双模式函数，需要在体内区分环境）；Rust 的 `const fn` 天生只能编译期，不存在这个需求。

### consteval 家族版本演进

```
C++20 : consteval 函数          —— 只能编译期求值的"立即函数"
C++20 : std::is_constant_evaluated() —— 检测当前是否编译期（普通 if）
C++23 : if consteval            —— 编译期分支，可安全调 consteval 函数
```

---

### Deducing this（推导 this，C++23）

> **版本标注：C++23 语言特性**（显式对象成员函数 / 多态 lambda 语法扩展）

### 特性说明

C++23 引入**显式对象成员函数（explicit object member function）**：成员函数的第一个参数可以是**显式声明的 `this` 参数**，从而推导出对象的类型和值类别（左值/右值、const/非 const）。

```c++
// C++23 新方式：显式 this 参数
struct T {
  decltype(auto) operator[](this auto& self, std::size_t idx) {
    return self.mVector[idx];
  }
};

// C++23 之前的旧方式：需要写多个重载
struct T {
  value_t& operator[](std::size_t idx) {           // 非 const 左值
    return mVector[idx];
  }
  const value_t& operator[](std::size_t idx) const {  // const 左值
    return mVector[idx];
  }
  // 还要写 && 右值版本...
};
```

### 语法拆解

`this auto& self` 是核心，拆开看三部分：

```
this  auto&  self
 │      │      │
 │      │      └── 显式参数名（self 是惯例名，可任意起）
 │      └───────── auto&：自动推导对象类型 + 值类别（引用）
 └──────────────── this 关键字：标记这是"显式对象参数"
```

**关键点**：

| 部分 | 含义 |
|:---|:---|
| `this` | 告诉编译器这个参数代表"对象本身"（替代隐式 `this`） |
| `auto` | 让编译器自动推导对象类型 `T` |
| `&` / `&&` / 空 | 指定值类别：左值引用 / 右值引用 / 按值 |
| `self` | 参数名，函数体内用它访问成员（替代原来的 `this->`） |

**值类别由 `auto` 后的修饰符决定**：

```c++
this auto& self        // 左值（非 const）对象可用
this const auto& self  // 左值 + const 对象可用
this auto&& self       // 左值 + 右值都可用（转发引用，自动匹配）
this auto self         // 按值接收对象（拷贝，很少用）
```

### 为什么需要？解决了什么问题

**痛点 1：要处理多种"对象形态"，必须写多个重载**

C++23 之前，一个成员函数要同时支持 `T&`、`const T&`、`T&&`（右值）三种调用形态，就得写三个几乎相同的重载：

```c++
// C++23 之前 —— 三个重载，样板爆炸
struct T {
  value_t& at(size_t i) & { return m[i]; }              // 左值
  const value_t& at(size_t i) const& { return m[i]; }   // const 左值
  value_t&& at(size_t i) && { return std::move(m[i]); } // 右值
};

// C++23 —— 一个函数搞定，auto&& 自动匹配三种形态
struct T {
  decltype(auto) at(this auto&& self, size_t i) {
    return self.m[i];   // 左值时返回引用，右值时返回右值引用
  }
};
```

**痛点 2：递归 lambda 难写**

C++23 之前，lambda 想递归必须用 `std::function`（有开销）或 Y 组合子（晦涩）。显式 `this` 参数让 lambda 直接拿自己：

```c++
// C++23 之前 —— 用 std::function 递归，有堆分配开销
std::function<int(int)> fib = [&](int n) {
    return n < 2 ? n : fib(n-1) + fib(n-2);
};

// C++23 —— 显式 this，零开销、类型推导
auto fib = [](this auto self, int n) -> int {
    return n < 2 ? n : self(n-1) + self(n-2);
};
// self 就是 lambda 自身，直接递归
```

### 使用场景

**场景 1：一个成员函数通吃 const / 非 const / 左值 / 右值**

```c++
struct Vec {
    std::vector<int> data;

    // 一个函数替代 3 个重载
    decltype(auto) operator[](this auto&& self, size_t i) {
        return std::forward_like<decltype(self)>(self.data[i]);
        // 简单场景可直接 self.data[i]
    }
};

Vec v;
v[0] = 10;              // 左值 → 返回 int&
const Vec& cv = v;
cv[0];                  // const → 返回 const int&
std::move(v)[0];        // 右值 → 返回 int&&（允许移动）
```

**场景 2：lambda 递归（最常用的实际收益）**

```c++
auto factorial = [](this auto self, int n) -> int {
    return n <= 1 ? 1 : n * self(n - 1);
};
factorial(5);  // 120
```

**场景 3：泛型成员函数，类型随对象推导**

```c++
struct Wrapper {
    int value = 42;
    // 类型名可用：decltype(self) 就是当前对象类型
    void print_type(this auto& self) {
        std::cout << typeid(decltype(self)).name();
    }
};
```

### 注意事项与最佳实践

| 要点 | 说明 |
|:---|:---|
| **函数体内用 `self` 访问成员** | 不再是隐式 `this->`，而是显式参数 `self.member` |
| **`this auto&&` 通用写法** | 一个函数覆盖左值/右值/const 三种形态（最常用） |
| **`decltype(self)` 可得对象类型** | 泛型代码里可做类型推导/约束 |
| **返回值用 `decltype(auto)`** | 才能正确保留引用/值类别 |
| **lambda 递归零开销** | 替代 `std::function` 递归（避免堆分配 + 类型擦除） |
| **不能用在静态成员函数** | 静态成员没有对象，无 `this` |
| **命名惯例** | 参数常叫 `self`（Python 风格），但可任意命名 |

**最佳实践**：
1. **需要 const/非 const/左值/右值统一处理** → `this auto&& self` 一函数搞定
2. **递归 lambda** → 用 `this auto self`，比 `std::function` 高效
3. **返回值保留引用语义** → 用 `decltype(auto)`
4. **通用写法**：`(this auto&& self, ...)` 兼容所有对象形态
5. **结合模板约束**：`(this std::integral auto self, ...)` 用 concept 约束对象类型

### 对比 Rust

Rust 用 `&self` / `&mut self` / `self` / `self: Box<Self>` 区分方法接收者——概念上很像 C++23 的显式 this，但 Rust 是**声明式**（写死接收者形态），C++23 是**推导式**（auto 自动匹配）：

| | C++23 `this auto` | Rust `&self` / `&mut self` |
|:---|:---|:---|
| 接收者形态 | 一个函数 `auto&&` 自动匹配 | 每种形态写一个 impl |
| 左值/右值 | ✅ 自动推导 | ✅ 借用规则天然区分 |
| const 区分 | ✅ 一个函数内推导 | ✅ `&self` vs `&mut self` |
| 递归闭包 | ✅ `this auto self` | ✅ 递归闭包不可直接（需 Box/indirection） |
| 统一处理多形态 | ✅ 一个函数 | 需分别写（或宏） |

> **对比要点**：Rust 的方法接收者是"你声明哪种就是哪种"（`&self`、`&mut self` 分开写），C++23 的显式 this 是"一个函数，auto 自动匹配所有形态"。两者都能表达接收者的类型与值类别，C++23 用模板推导实现"一个顶多个"。

---

### Multidimensional Subscript Operator（多维下标运算符，C++23）

> **版本标注：C++23 语言特性**

### 特性说明

C++23 允许 `operator[]` 接受**零个或多个参数**，实现真正的多维下标访问——`v[x][y][z]` 或 `v[x, y, z]` 不再需要手写嵌套 `operator[]`。

```c++
template <typename T, std::size_t Z, std::size_t Y, std::size_t X>
struct Array3d {
  std::array<T, X * Y * Z> m{};

  T& operator[](std::size_t z, std::size_t y, std::size_t x) {
      return m[z * Y * X + y * X + x];
  }
};

Array3d<int, 4, 3, 2> v;
v[3, 2, 1] = 42;   // 一个 [] 传三个下标
```

### 为什么需要？之前怎么做

C++20 及之前，`operator[]` 只能接受**一个参数**。要模拟多维访问，只能用丑陋的嵌套：

```c++
// C++20 及之前 —— 嵌套 operator[]，每层一个 []
struct Array3d {
    std::vector<T> m;
    auto operator[](size_t z) {   // 第一层返回一个"中间对象"
        return Proxy{z, *this};    // 需要代理类！
    }
};
// 用起来：v[z][y][x] —— 中间对象 + 多层代理，样板爆炸

// C++23 —— 一个 [] 直接传所有下标
v[z, y, x] = 42;
```

| | C++20 及之前 | C++23 |
|:---|:---|:---|
| `operator[]` 参数 | **只能 1 个** | 0 个或多个 |
| 多维访问 | 嵌套 `[][]` + 代理类 | 直接 `[z, y, x]` |
| 代理样板 | ✅ 需要 | ❌ 不需要 |
| 简洁度 | 差 | 好 |

### 语法要点

```c++
struct Mat2d {
    // 多参数 operator[]：用逗号分隔
    T& operator[](std::size_t r, std::size_t c) { ... }
};
Mat2d m;
m[2, 3] = 5;   // 一个 []，两个下标

// 零参数（罕见）：operator[]() 也是合法的
// 注意：C++23 前 operator[] 至少 1 个参数
```

**注意**：`v[x, y]` 里的逗号**不再是逗号运算符**——在多参数 `operator[]` 的语境中，`[x, y]` 被解释为两个参数，语义与 C++17 之前（逗号运算符求值 x 再 y）完全不同。

### 使用场景与最佳实践

1. **多维容器/矩阵**：`v[z, y, x]` 比嵌套 `[][]` 清晰，且可直接做边界检查
2. **配合 `std::span`**：返回 `std::span<T>` 子区间而非代理
3. **可用于空参数**：`operator[]()` 可作为特殊索引语法
4. **与多维下标配套的 `mdspan`**（C++23 标准库）常用于科学计算
5. **避免破坏性修改**：新语法 `[a, b]` 语义变了，升级到 C++23 时注意旧代码里"利用逗号运算符当下标"的写法

### 对比 Rust

Rust 的 `Index`/`IndexMut` trait 也**只能接受一个索引参数**（如 `v[i]`）。Rust 的多维通常用 `v[[y, x]]`（传一个数组切片）或 `ndarray` 等库：

| | C++23 | Rust |
|:---|:---|:---|
| `operator[]`/`Index` 参数 | 0 个或多个 | **只能 1 个** |
| 多维下标 | `v[z, y, x]` 原生支持 | `v[[y, x]]` 传切片数组 |
| 多维库 | `std::mdspan`（C++23） | `ndarray` |

> Rust 用"数组字面量做下标"（`v[[i, j]]`）绕过单参数限制；C++23 直接原生支持多参数下标。

---

### Increasing Range-Based For Safety（增强范围 for 安全性，C++23）

> **版本标注：C++23 语言特性**（range-based for 生命周期修复）

### 特性说明

C++23 修复了 range-based `for` 循环最臭名昭著的一组**生命周期问题**——当范围表达式是**临时对象**时，之前会悬垂，现在编译器自动延长临时对象的生命周期到整个循环。

**C++23 之前会出问题（现已修复）的写法：**

```c++
for (auto e : getTmp().getRef())          // 临时对象的方法返回引用
for (auto e : getVector()[0])             // 临时 vector 的某元素
for (auto valueElem : getMap()["key"])    // 临时 map 的值
for (auto e : get<0>(getTuple()))         // 临时 tuple 的某元素
for (auto e : getOptionalCollection().value())  // 临时 optional 的值
for (char c : get<std::string>(getVariant()))   // 临时 variant 的字符串
```

### 为什么需要？之前怎么出问题的

**问题根源**：C++17 的 range-based `for` 只延长**范围对象本身**的临时生命周期，但不延长**范围表达式里其它临时对象**的生命周期：

```c++
// C++17 及之前 —— 悬垂 bug
for (auto e : getVector()[0]) {
    // getVector() 返回的临时 vector 在循环开始前就销毁了！
    // [0] 返回的引用指向已销毁的 vector → 未定义行为
}

// C++23 —— 编译器自动延长临时 vector 的生命周期到循环结束
for (auto e : getVector()[0]) {
    // ✅ 安全：临时对象在整个循环期间存活
}
```

**C++17 的老规则**：`for (auto x : expr)` 展开时，只有 `expr` 最外层的临时对象被延长生命周期；如果 `expr` 内部还有临时对象（如 `getVector()[0]` 里的 `getVector()`），那个内部临时对象**不会**被延长 → 悬垂。

**C++23 的新规则**：整个范围表达式里**所有**临时对象都被延长到循环结束，彻底修复这类 bug。

### 修复前 vs 修复后对比

```c++
// C++17 及之前：有问题的写法
for (auto e : getTmp().getRef()) { ... }   // getTmp() 临时对象悬垂

// C++23：同一写法现在安全了
for (auto e : getTmp().getRef()) { ... }   // ✅ 临时对象生命周期延长

// 若你之前在 C++17 下"必须"这样绕，现在可以简化：
// C++17 绕路：先存临时对象
auto tmp = getTmp();
for (auto e : tmp.getRef()) { ... }        // 手动延长

// C++23 直接写，编译器自动处理
for (auto e : getTmp().getRef()) { ... }
```

### 注意事项

| 要点 | 说明 |
|:---|:---|
| **只影响 range-based for** | 普通函数调用/表达式不受影响 |
| **C++23 才修复** | C++17/20 下这些写法仍悬垂，升级后变安全 |
| **是"修复"非"新特性"** | 语法不变，行为更安全 |
| **无法检测旧 bug** | 编译器无法知道你旧代码是不是故意这么写 |

### 对比 Rust

Rust 的 `for` 循环基于迭代器，借用检查器在**编译期**就阻止悬垂引用——这类生命周期 bug 在 Rust 里根本编译不过：

```rust
// Rust：借用检查器保证安全
for e in get_vec()[0].iter() { ... }
// 临时 vector 生命周期由借用规则保证，编译器强制
```

| | C++23 | Rust |
|:---|:---|:---|
| 临时对象生命周期 | 编译器自动延长（新规则） | 借用检查器强制 |
| 悬垂检测时机 | 运行期修复（语义安全） | 编译期阻止 |
| 语法 | 不变 | 借用标记 |

> **对比要点**：C++23 是"修运行时语义"（延长生命周期），Rust 是"编译期禁止"（借用检查）——两者结果都是更安全，但 Rust 在编译期就拦住。

---

### Template Syntax for Lambdas（Lambda 的模板语法）

### 特性说明

C++20 允许在 lambda 表达式中使用**显式模板参数列表**，直接在 `[]` 后面写 `template<...>`：

```c++
auto f = []<typename T>(std::vector<T> v) {
    // 现在可以直接用 T 了！
    T first = v.front();
};
```

### 之前版本没出现过：C++14 泛型 lambda 的局限

C++14 引入了泛型 lambda，用 `auto` 参数实现多态：

```c++
// C++14 —— auto 参数，但无法给类型命名
auto f = [](auto x) { return x; };
```

问题是**无法命名类型参数**，只能靠 `decltype` 变通：

```c++
// C++14 —— 只能用 decltype 间接获取类型
auto print = [](auto v) {
    using T = decltype(v);           // 绕弯获取类型
    std::cout << "size=" << sizeof(T) << std::endl;
    // 想声明同类型变量？decltype 再来一次
    decltype(v) copy = v;
};

// 多个参数想保证同一类型？做不到！
// auto f = [](auto a, auto b) { };   // a 和 b 可能类型不同
```

**C++14 泛型 lambda 的具体痛点：**

| 痛点 | 说明 |
|:---|:---|
| 类型无法命名 | 只能 `decltype(v)`，冗长且难以阅读 |
| 无法在参数前用类型 | 返回类型、声明变量时都要 `decltype` |
| 多参数类型不统一 | `auto a, auto b` 各自独立推导，无法要求同型 |
| 无法用非类型模板参数 | 如 `[]<size_t N>` 这种编译期常量 |
| 无法配合 Concept | 不能直接约束 `auto` 参数满足某个概念 |
| 类型特征用起来麻烦 | `std::is_same_v<decltype(v), int>` 又长又绕 |

### 现在支持这个特性有什么作用？

**1. 给类型命名（最核心）**

```c++
auto f = []<typename T>(T x) {
    T doubled = x * 2;              // ✅ 直接声明同类型变量
    std::cout << "类型大小: " << sizeof(T) << std::endl;
};
```

**2. 配合 Concept 约束参数**

```c++
// 约束 T 必须是整数
auto sum = []<std::integral T>(T a, T b) {
    return a + b;
};

// 使用 —— 传非整数会编译报错
sum(1, 2);          // ✅ 3
// sum(1.5, 2.5);   // ❌ double 不满足 integral
```

**3. 非类型模板参数 + 编译期常量**

```c++
// 接受编译期常量 N，用于固定大小数组
auto fill = []<std::size_t N>(const std::array<int, N>& arr) {
    static_assert(N > 0, "数组不能为空");
    // 可以在编译期知道数组大小！
    int sum = 0;
    for (auto v : arr) sum += v;
    return sum;
};

std::array<int, 3> a{1, 2, 3};
fill(a);  // 6
```

**4. 返回类型依赖参数类型时更清晰**

```c++
auto make_pair = []<typename T, typename U>(T a, U b) {
    // 需要 T 和 U 两个类型名
    using R = std::pair<T, U>;
    return R{a, b};
};
auto p = make_pair(1, "hello");  // pair<int, const char*>
```

**5. 模板模板参数 / 参数包（配合折叠表达式）**

```c++
// 可变参数包 + 折叠
auto sum_all = []<typename... Ts>(Ts... args) {
    return (0 + ... + args);   // 折叠表达式
};
sum_all(1, 2, 3, 4);  // 10
```

### 最佳实践

1. **需要类型名就用模板 lambda**——如果函数体里多次用到参数的类型，别再用 `decltype` 绕
2. **配合 Concept 做编译期约束**——`[]<std::integral T>` 让签名自带文档
3. **有编译期常量需求**（数组大小、模板参数）用非类型模板参数
4. **多参数同类型约束**——`[]<typename T>(T a, T b)` 保证 a、b 类型一致
5. **与 `if constexpr` / 折叠表达式组合**做编译期逻辑
6. **工厂函数 / 类型擦除包装器**中常用

### 没这个特性的替代方案（C++14/17）

如果坚持用 C++14/17，只能回到函数模板 + lambda 或手写仿函数：

```c++
// 替代方案 1：普通函数模板包装
template<typename T>
void print_impl(const T& v) {
    std::cout << sizeof(T) << std::endl;
}
auto print = [](auto v) { print_impl(v); };  // 还是绕

// 替代方案 2：手写仿函数（最彻底但最啰嗦）
struct Print {
    template<typename T>
    void operator()(const T& v) const {
        std::cout << sizeof(T) << std::endl;
    }
};
Print print;  // 用 print(42)
```

**结论**：C++20 的模板 lambda 把"泛型 lambda 的最后一个短板"补上了——让类型可命名、可约束、可参与模板元编程，同时保留了 lambda 的简洁性。

---

### Lambda Capture of Parameter Pack（Lambda 捕获参数包，C++20）

> **版本标注：C++20 语言特性**（lambda 家族 + 模板参数包的结合）

### 特性说明

C++20 允许 lambda 直接捕获**参数包**（parameter pack），按值或按引用捕获整个可变参数序列：

```c++
// 按值捕获参数包
template <typename... Args>
auto f(Args&&... args){
    return [...args = std::forward<Args>(args)] {
    };
}

// 按引用捕获参数包
template <typename... Args>
auto f(Args&&... args){
    return [&...args = std::forward<Args>(args)] {
    };
}
```

### 为什么需要？C++20 之前怎么做

**痛点：lambda 捕获的是"固定个数的变量"，而参数包是"不定个数的变量"。** C++14 的 init-capture 只能逐个捕获，无法把参数包整体展开进捕获列表：

> **澄清：这个特性为什么只在模板里出现？** 参数包是"模板世界的产物"，所以捕获它必然发生在模板里；但 lambda 本身是个通用工具，这个特性只是它多了一个在模板中才能用上的本领。

```c++
// C++11/14 —— 想捕获整个参数包？做不到！
template <typename... Args>
auto bad(Args... args) {
    // [args...] 这种语法 C++20 之前不存在！
    // 只能绕路：把参数包塞进 tuple 再捕获
    return [t = std::make_tuple(std::move(args)...)] { /* 用 tuple 变通 */ };
}
```

**C++14 的绕路方案**：先把参数包打包成 `std::tuple`，再捕获整个 tuple，用的时候用 `std::apply` 解包：

```c++
// C++14 —— 绕路：参数包 → tuple → 捕获 → apply 解包
template <typename... Args>
auto delayed_print(Args... args) {
    return [t = std::make_tuple(args...)] {
        std::apply([](auto... xs) { (std::cout << xs << " ", ...); }, t);
    };
}
```

问题：多一层 `tuple`/`apply` 样板，且类型信息被"打包"再"解包"，不够直接。

**C++20 直接解决**：用 `...` 展开语法让参数包**整体进入捕获列表**，语法上就是捕获列表里加一个包展开：

```c++
template <typename... Args>
auto f(Args&&... args){
    return [args...] { /* 按值捕获每个参数 */ };
    // 或 [...args = std::forward<Args>(args)] 完美转发捕获
}
```

### 两种捕获方式

**1. 按值捕获（`[...args = ...]`）**

```c++
template <typename... Args>
auto by_value(Args... args){
    return [args...] {           // 等价写法
        return (args + ...);     // 折叠表达式求和
    };
}
```

**2. 按引用捕获（`[&...args = ...]`）**

```c++
template <typename... Args>
auto by_ref(Args&... args){
    return [&args...] {          // 持有所有参数的引用
        (args *= 2, ...);        // 修改外部变量
    };
}
```

**3. 完美转发捕获（`[...args = std::forward<Args>(args)]`）**

官方示例用的这种，保留左值/右值属性，避免不必要的拷贝：

```c++
template <typename... Args>
auto forward_capture(Args&&... args){
    return [...args = std::forward<Args>(args)] {
        // args 各自保持原有的值类别语义
    };
}
```

### 使用场景

**场景 1：把参数包"存储"进闭包，延迟到调用时展开**

```c++
template <typename F, typename... Args>
auto bind_all(F f, Args... args){
    // 把参数包按值捕获进闭包，延迟调用
    return [f, args...]() { return f(args...); };
}

auto g = bind_all([](int a, int b) { return a + b; }, 3, 4);
g();  // 7
```

**场景 2：异步/线程中捕获参数包**

```c++
template <typename... Args>
void run_async(Args&&... args){
    std::thread([...args = std::forward<Args>(args)] {
        // 新线程中使用捕获的参数包
        process(args...);
    }).detach();
}
```

**场景 3：回调工厂——把参数打包存起来**

```c++
template <typename... Args>
auto make_logger(Args... args){
    return [args...]() {
        std::cout << "日志参数: ";
        ((std::cout << args << " "), ...);
        std::cout << std::endl;
    };
}

auto log = make_logger(1, "hello", 3.14);
log();  // 日志参数: 1 hello 3.14
```

### 注意事项与最佳实践

| 要点 | 说明 |
|:---|:---|
| **语法位置** | 捕获列表里用 `...` 展开，如 `[args...]` 或 `[...args = expr]` |
| **与 init-capture 配合** | `[...args = std::forward<Args>(args)]` 是"捕获参数包 + 每个元素初始化" |
| **按值捕获仅可移动类型** | 用 `[...args = std::move(args)]` 把不可复制类型移动进闭包 |
| **生命周期** | 按引用捕获 `[&args...]` 时，调用方参数必须先于 lambda 存活（同普通引用捕获） |
| **空参数包** | 参数包为空时捕获列表为空，lambda 仍合法 |
| **配合折叠表达式** | 捕获后在 lambda 体内用 `(args + ...)` 批量处理，是黄金组合 |
| **C++14 替代** | 只能 `make_tuple` 打包再 `apply` 解包，样板多 |

**最佳实践**：
1. **延迟调用/回调**用按值捕获（或完美转发），避免悬垂引用
2. **新线程/异步**务必按值（`std::move`/`std::forward`）捕获，捕获引用有竞态风险
3. **捕获后配合折叠表达式**是最常用的组合模式
4. **需要保留左右值语义**用 `[...args = std::forward<Args>(args)]`
5. **只想移动**用 `[...args = std::move(args)]`

### 与模板 lambda 的组合（两个 C++20 lambda 特性一起用）

```c++
// 模板 lambda（类型可命名）+ 参数包捕获（打包存储）＝ 完整方案
auto make_sum = []<typename... Args>(Args... args) {
    return [args...] { return (args + ...); };
};

auto sum = make_sum(1, 2, 3, 4);
sum();  // 10
```

### 版本演进链条（lambda 捕获越来越强）

```
C++11  : [=] [&] [x]          只能捕获【固定个数】的变量
C++14  : [x = expr]           可捕获【单个】表达式/移动捕获（init-capture）
C++20  : [args...]            可捕获【参数包】（不定个数，整体展开）
       + [=, *this] 等        隐式捕获 this 收紧，显式捕获
```

### 对比 Rust

Rust 闭包用 `move` 捕获外部变量，但**参数包（variadic）本身在 Rust 中不存在**——Rust 没有 C++ 这种可变参数模板。要处理"任意个参数"只能用宏（macro_rules）或固定元组。所以"捕获参数包"这个需求在 Rust 里没有直接对应：

| | C++20 | Rust |
|:---|:---|:---|
| 可变参数模板 | ✅ 参数包 | ❌ 无，用宏模拟 |
| 捕获不定个数参数 | ✅ `[args...]` | ❌ 需宏生成闭包 |
| 存储参数延迟调用 | ✅ 直接捕获包 | 用宏生成固定参数的闭包 |

> **结论**：这是 C++ 特有优势——参数包 + 闭包捕获的组合在 Rust 中没有直接对应物。

---

### constinit（编译期初始化要求，C++20）

### 特性说明

`constinit`（C++20）是**变量的说明符**，它只做一件事：**要求这个变量必须在编译期（静态初始化阶段）完成初始化**。它**不要求变量是 `const`**——初始化后仍然可以修改：

```c++
constinit int x = 42;   // ✅ 编译期初始化
x = 100;                // ✅ 可以改！constinit 不禁止修改
```

```c++
const char* g() { return "dynamic initialization"; }   // 运行期动态初始化
constexpr const char* f() { return "constant initializer"; }  // 编译期常量

constinit const char* c = f();  // ✅ f() 是常量表达式 → 编译期初始化
constinit const char* d = g();  // ❌ g() 是动态初始化 → 编译期无法确定
```

- `c` 用 `constexpr` 函数 `f()` 初始化 → 编译期可算 → ✅
- `d` 用普通函数 `g()` 初始化 → 运行期才执行 → ❌

### 四关键字对比：const / constinit / constexpr / consteval

这四个概念最容易混淆，核心区别是**"管函数还是管变量"** + **"要求编译期吗"** + **"可修改吗"**：

| 关键字 | 作用于 | 要求编译期求值？ | 对象可修改？ | 一句话 |
|:---|:---|:---:|:---:|:---|
| `const` | 变量 | ❌ 不要求 | ❌ 不可修改 | 只是"不可变"，初始化可运行期 |
| `constinit` | **只能变量** | ✅ 必须静态初始化 | ✅ 可修改 | "只要编译期初始化"，之后随便改 |
| `constexpr` | 变量 + 函数 | ✅ 变量必须编译期；函数"可在编译期求值" | ❌ 变量不可改 | 编译期求值 + 不可变（也可运行期调用） |
| `consteval` | **只能函数** | ✅ 函数必须编译期求值 | —（管函数） | "立即函数"，拒绝运行期调用 |

**逐项解读：**

| 对比项 | `const` | `constinit` | `constexpr` 变量 | `consteval` 函数 |
|:---|:---|:---|:---|:---|
| 适用对象 | 变量 | 变量 | 变量 | 函数 |
| 要求编译期初始化/求值 | ❌ | ✅ | ✅ | ✅ |
| 初始化后可修改 | ❌ | ✅ | ❌ | — |
| 存储期要求 | 无 | ✅ 静态存储期 | 无 | — |
| 能用于局部变量 | ✅ | ❌ | ✅ | — |

**关键记忆**：

```
const      = "值不可变"（不关心何时初始化）
constinit  = "必须编译期初始化"（值可变）
constexpr  = "编译期求值 + 不可变"（变量层面）
consteval  = "函数只能编译期求值"（函数层面）
```

```c++
const int a = compute();     // ✅ 可以！const 不要求编译期
constinit int b = compute(); // ❌ 必须编译期，compute() 是动态初始化
constexpr int c = compute(); // ❌ 若 compute() 非 constexpr
constinit int d = 42;        // ✅ 编译期初始化
d = 99;                      // ✅ 可改（constinit 允许）
constexpr int e = 42;        // ✅ 编译期初始化
// e = 99;                   // ❌ 不可改（constexpr 隐含 const）
consteval int f(int n){ return n; }  // 只能编译期调用
```

### 为什么需要？解决什么问题

**问题：静态初始化顺序（Static Initialization Order Fiasco）**

C++ 全局变量初始化分两种：

- **静态初始化**（编译期常量，顺序无关）
- **动态初始化**（运行期执行代码，**跨翻译单元顺序不定**！）

```c++
// file1.cpp
int a = compute();      // 动态初始化（运行期才执行）

// file2.cpp
int b = a + 1;          // 依赖 a —— 但 a 何时初始化？顺序不定！
                        // 如果 b 先初始化，读到未初始化的 a → 未定义行为
```

**constinit 怎么帮**：把"必须是静态初始化"变成**编译期检查**——若试图用它做动态初始化，直接编译报错，把潜在的运行期 bug 提前到编译期暴露：

```c++
constinit int x = 42;        // ✅ 常量，静态初始化，无顺序问题
constinit int y = compute(); // ❌ 编译错误！compute() 是动态初始化
                             //    → 从语言层面杜绝了顺序 fiasco
```

### 使用场景

**场景 1：全局/静态常量，确保无初始化顺序问题**

```c++
constinit const int kMaxSize = 4096;  // 静态初始化，全局安全
```

**场景 2：需要初始化后修改的全局状态**

```c++
constinit int log_level = 3;   // 编译期初始化为 3
void set_log_level(int l) { log_level = l; }  // 之后可改 ✅
```

**场景 3：constinit + consteval 配合**

```c++
consteval int sqr(int n) { return n * n; }

constinit int r = sqr(10);   // ✅ sqr 是编译期求值，r 静态初始化
```

### 注意事项与最佳实践

| 要点 | 说明 |
|:---|:---|
| **只能用于静态存储期变量** | 全局、命名空间作用域、静态局部、线程局部；**不能用于普通局部变量或非静态成员** |
| **不隐含 const** | 想不可变需自己加 `const`（`constinit const`） |
| **不能与 `constexpr` 同时用** | `constexpr` 已隐含"编译期初始化 + const"，再加 `constinit` 冗余报错 |
| **可配合 `const`** | `constinit const` 是常见组合：编译期初始化 + 不可变 |
| **跨翻译单元共享** | 用 `constinit const` 杜绝静态初始化顺序 fiasco |

**最佳实践**：
1. **全局/静态需要编译期初始化的变量** → 用 `constinit`（想更严格加 `const`）
2. **只想"编译期初始化但之后要改"** → 用 `constinit`（不用 `constexpr`）
3. **想要"编译期初始化 + 不可变"** → 用 `constexpr`
4. **跨翻译单元共享的初始化常量** → 用 `constinit const`，杜绝顺序 fiasco
5. **普通局部变量**不要用（`constinit` 要求静态存储期，会编译错误）

### 对比 Rust

Rust 的 `const` 与 `static` 承担了类似职责：

| | C++20 `constinit` | Rust `const` / `static` |
|:---|:---|:---|
| 编译期初始化 | ✅ 强制 | ✅ 强制 |
| 初始化后可修改 | ✅ 可以 | `static mut` 可以（但 `unsafe`）；`const` 不可 |
| 全局共享 | ✅ | ✅ `static` |
| 顺序 fiasco 防护 | ✅ 编译期检查 | ✅ 天生无此问题（`const` 内联） |
| 等价物 | `constinit`（可改）≈ `static mut` | — |

> Rust 的 `const` 必须在编译期求值，`static` 是全局共享但有初始化顺序保证（Rust 无顺序 fiasco）；C++20 的 `constinit` 相当于给 C++ 的全局变量补上了"必须静态初始化"的编译期保证。

---

### Range-Based For Loop with Initializer（带初始化器的范围 for 循环）

### 特性说明

C++20 允许在**范围 for 循环**的头部初始化一个变量，范围对象在循环头部声明，其生命周期被**限定在循环内部**：

```c++
for (auto v = std::vector{1, 2, 3}; auto& e : v) {
  std::cout << e;
}
// v 在这里已销毁
```

### 之前版本没有吗？

需要区分两种 for 循环：

```c++
// C++98 就有 —— 普通 for 循环可以在头部初始化
for (int i = 0; i < 10; ++i) { ... }

// C++11 就有 —— 范围 for，但头部只有"循环变量"，没有"初始化语句"的位置
for (auto& e : v) { ... }
```

**范围 for 的语法是** `for (声明 : 范围)`——头部只能写循环变量声明。想初始化"范围对象"，**C++17 及之前必须提前写在循环外面**：

```c++
// C++17 及之前 —— 必须把范围对象提前声明
auto v = std::vector{1, 2, 3};   // 作用域被不必要地扩大
for (auto& e : v) {
    std::cout << e;
}
// v 在循环结束后仍然存活
```

所以**带初始化器的范围 for 是 C++20 新增**的。

### 与 C++17 的 `if` 初始化器是同一族

这跟 **Selection Statements with Initializer**（`if (init; cond)`）是同一设计理念——都是把临时对象的作用域**限定在语句内部**：

| 特性 | 版本 | 语法 |
|:---|:---|:---|
| `if` / `switch` 初始化器 | C++17 | `if (init; cond)` |
| 范围 `for` 初始化器 | C++20 | `for (init; decl : range)` |

### 真正解决什么问题？生命周期！

最经典的场景：范围对象是**临时值**（如函数返回值），C++17 前会出现**悬垂引用**：

```c++
// ❌ C++17 之前的坑：范围是临时对象
for (auto& e : get_data()) {   // get_data() 返回的临时 vector
    // e 引用临时对象内部数据 → 但临时对象在循环开始前就销毁了！
    // 未定义行为！
}

// ✅ C++20 修复：临时对象生命周期延长到整个循环
for (auto v = get_data(); auto& e : v) {
    // v 在整个循环期间存活，安全！
}
```

### 使用场景

**场景 1：遍历函数返回的临时容器**

```c++
// 读取文件的每一行，直接遍历
for (auto lines = read_lines("config.txt"); auto& line : lines) {
    if (line.starts_with("#")) continue;  // 跳过注释
    process(line);
}
```

**场景 2：局部排序后遍历（不改原数据）**

```c++
std::vector<int> scores = {5, 3, 8, 1};

// C++20 —— 排序副本，只在这个循环内有效
for (auto sorted = scores; auto& s : sorted) {
    std::sort(sorted.begin(), sorted.end());
    std::cout << s << " ";   // 1 3 5 8
}
// sorted 已销毁，scores 保持原样
```

**场景 3：持有锁遍历（生命周期限定）**

```c++
std::mutex mtx;
std::vector<int> data;

// 锁和范围对象绑定在循环内，循环结束自动释放
for (auto lk = std::lock_guard(mtx); auto& e : data) {
    use(e);
}
// 锁在此处已释放
```

### 最佳实践与注意事项

| 要点 | 说明 |
|:---|:---|
| **范围对象生命周期限定在循环内** | 循环结束后自动销毁，避免污染外层作用域 |
| **优先用引用遍历** | `auto& e` 避免拷贝，`const auto& e` 只读遍历 |
| **解决临时范围悬垂** | 这是本特性最大价值——比 C++17 的手动声明更安全 |
| **可配合结构化绑定** | `for (auto m = map; auto& [k, v] : m)` 合法 |
| **与 C++17 `if` 初始化器同族** | 统一了"语句内初始化"的设计理念 |