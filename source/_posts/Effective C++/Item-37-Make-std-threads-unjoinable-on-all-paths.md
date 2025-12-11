---
title: 'Item 37: Make std::threads unjoinable on all paths'
categories: Effective C++
date: 2025-12-11 20:20:17
tags:
priority: 37
---
# Item 37: 确保 std::thread 在所有路径上都是不可连接的

## 摘要
在多线程C++编程中，正确管理 `std::thread` 对象的生命周期至关重要。本条目深入探讨了 `std::thread` 的可连接性概念，解释了为什么在销毁时保持可连接状态会导致程序终止，并介绍了如何使用RAII（Resource Acquisition Is Initialization，资源获取即初始化）模式来确保线程在所有执行路径上都能被正确处理。

## 目录
1. 引言：std::thread 的可连接性
2. 问题：可连接线程销毁的风险
3. 分析：为什么设计如此
4. 解决方案：ThreadRAII 包装类
5. ThreadRAII 的实现细节
6. 使用示例
7. 移动语义支持
8. 线程管理的最佳实践
9. 常见陷阱与调试技巧
10. 总结

---

## 1. 引言：std::thread 的可连接性

在C++中，每个 `std::thread` 对象都处于两种状态之一：可连接（joinable）或不可连接（unjoinable）。

### 1.1 可连接状态
一个 `std::thread` 在以下情况下是可连接的：
- 关联的线程正在运行
- 关联的线程被阻塞或等待调度
- 关联的线程已运行完成但尚未被连接或分离

### 1.2 不可连接状态
一个 `std::thread` 在以下情况下是不可连接的：
- 默认构造（没有关联的执行线程）
- 已被移动到另一个 `std::thread` 对象
- 已被连接（`join()` 已被调用）
- 已被分离（`detach()` 已被调用）

```cpp
#include <iostream>
#include <thread>

void threadFunction() {
    std::this_thread::sleep_for(std::chrono::milliseconds(100));
}

int main() {
    std::thread t1;  // 默认构造，不可连接
    std::thread t2(threadFunction);  // 可连接
    
    std::thread t3 = std::move(t2);  // t2 现在不可连接（已被移动）
    
    if (t1.joinable()) std::cout << "t1 is joinable\n";
    if (t2.joinable()) std::cout << "t2 is joinable\n";  // 不会输出
    if (t3.joinable()) std::cout << "t3 is joinable\n";
    
    t3.join();  // t3 现在不可连接
    if (t3.joinable()) std::cout << "t3 is joinable\n";  // 不会输出
    
    return 0;
}
```

## 2. 问题：可连接线程销毁的风险

### 2.1 关键规则
**如果一个可连接的 `std::thread` 对象被销毁，程序会终止。**

```cpp
#include <iostream>
#include <thread>

void riskyCode() {
    std::thread t( {
        std::this_thread::sleep_for(std::chrono::seconds(1));
        std::cout << "Thread completed\n";
    });
    
    // 注意：这里没有调用 t.join() 或 t.detach()
    // 当 t 被销毁时（离开作用域），如果它仍然可连接，程序终止
}  // 程序在此终止！

int main() {
    try {
        riskyCode();
    } catch (...) {
        std::cout << "Exception caught\n";
    }
    std::cout << "This will not be printed\n";
    return 0;
}
```

### 2.2 实际场景中的问题
考虑一个更实际的例子，其中线程管理可能失败：

```cpp
#include <iostream>
#include <thread>
#include <vector>
#include <stdexcept>

bool doWork(const std::vector<int>& data) {
    std::vector<int> results;
    
    // 启动一个线程来处理数据
    std::thread worker( {
        for (int value : data) {
            if (value < 0) {
                // 模拟一个错误条件
                throw std::runtime_error("Negative value found!");
            }
            results.push_back(value * 2);
        }
    });
    
    // 做一些其他工作
    bool success = processData();
    
    if (success) {
        worker.join();  // 等待线程完成
        useResults(results);
        return true;
    } else {
        // 注意：如果 success 为 false，worker 没有被连接
        // 当 worker 离开作用域时，程序会终止
        return false;
    }
    
    // 问题：如果 processData() 抛出异常，worker 也不会被连接
}
```

## 3. 分析：为什么设计如此

标准委员会选择在可连接线程被销毁时终止程序，是因为其他替代方案更糟糕：

### 3.1 隐式连接
```cpp
// 伪代码：如果 std::thread 析构函数隐式连接
~thread() {
    if (joinable()) join();  // 不要这样做！
}
```
**问题**：
- 可能导致意外的长时间阻塞
- 在错误处理路径中，可能不期望等待线程完成
- 难以调试的性能问题

### 3.2 隐式分离
```cpp
// 伪代码：如果 std::thread 析构函数隐式分离
~thread() {
    if (joinable()) detach();  // 不要这样做！
}
```
**问题**：
- 线程可能访问已销毁的局部变量
- 难以调试的悬挂引用和内存损坏
- 资源泄漏风险

### 3.3 示例：隐式分离的危险
```cpp
#include <iostream>
#include <thread>
#include <vector>

void dangerousExample() {
    std::vector<int> data = {1, 2, 3, 4, 5};
    std::thread t( {
        // 这个 lambda 捕获了 data 的引用
        std::this_thread::sleep_for(std::chrono::seconds(1));
        // 危险：当 dangerousExample 返回后，data 被销毁
        // 但线程可能还在运行并试图访问 data
        for (int value : data) {
            std::cout << value << " ";
        }
    });
    
    // 如果这里隐式分离，线程在后台运行
    // 当 dangerousExample 返回，data 被销毁
    // 但线程试图访问已销毁的 data -> 未定义行为
}
```

## 4. 解决方案：ThreadRAII 包装类

为了解决这个问题，我们使用RAII模式创建一个线程包装类：

### 4.1 RAII 原则
RAII（Resource Acquisition Is Initialization）是C++的核心惯用法：
- 在构造函数中获取资源
- 在析构函数中释放资源
- 确保资源在任何退出路径上都被正确释放

### 4.2 ThreadRAII 设计目标
1. 封装 `std::thread` 对象
2. 在析构函数中自动处理线程（连接或分离）
3. 提供对底层线程的安全访问
4. 支持移动语义

## 5. ThreadRAII 的实现细节

### 5.1 基本实现
```cpp
#include <thread>
#include <utility>

class ThreadRAII {
public:
    enum class DtorAction { join, detach };
    
    // 构造函数：接管线程所有权
    ThreadRAII(std::thread&& t, DtorAction action)
        : action(action), thread(std::move(t)) {
        // 注意：成员初始化顺序与声明顺序相同
        // 将 thread 放在最后声明是重要的
    }
    
    // 析构函数：根据策略处理线程
    ~ThreadRAII() {
        if (thread.joinable()) {
            if (action == DtorAction::join) {
                thread.join();
            } else {
                thread.detach();
            }
        }
    }
    
    // 禁止拷贝
    ThreadRAII(const ThreadRAII&) = delete;
    ThreadRAII& operator=(const ThreadRAII&) = delete;
    
    // 支持移动
    ThreadRAII(ThreadRAII&&) = default;
    ThreadRAII& operator=(ThreadRAII&&) = default;
    
    // 访问底层线程
    std::thread& get() { return thread; }
    const std::thread& get() const { return thread; }
    
private:
    DtorAction action;
    std::thread thread;  // 重要：最后声明
};
```

### 5.2 为什么将 thread 成员放在最后声明？
```cpp
class ThreadRAII {
private:
    DtorAction action;
    std::thread thread;  // 最后声明
    
public:
    ThreadRAII(std::thread&& t, DtorAction action)
        : action(action), thread(std::move(t)) {
        // thread 是最后一个初始化的成员
        // 这意味着当 thread 开始运行时，
        // 所有其他成员（如 action）都已经被初始化
    }
};
```

**原因**：`std::thread` 对象在构造后可能立即开始执行。如果线程函数访问类的其他成员，我们需要确保那些成员已经初始化。通过将 `std::thread` 成员最后声明，可以确保在它被初始化时，所有其他成员都已经初始化。

### 5.3 线程安全性考虑
```cpp
~ThreadRAII() {
    if (thread.joinable()) {  // 检查是否可连接
        if (action == DtorAction::join) {
            thread.join();
        } else {
            thread.detach();
        }
    }
}
```

**注意**：在单线程上下文中，`joinable()` 检查是安全的。在多线程场景中，如果多个线程同时操作同一个 `ThreadRAII` 对象，可能会有竞争条件。通常，应该只有一个线程管理 `ThreadRAII` 对象的生命周期。

## 6. 使用示例

### 6.1 修复 doWork 函数
```cpp
#include <iostream>
#include <thread>
#include <vector>
#include <functional>
#include <chrono>

// 模拟一些工作函数
bool conditionsAreSatisfied() {
    return true;  // 实际情况中这里可能有更复杂的逻辑
}

void performComputation(const std::vector<int>&) {
    // 计算结果
}

// 使用 ThreadRAII 的安全版本
bool doWorkSafe(std::function<bool(int)> filter, int maxVal = 1000000) {
    std::vector<int> goodVals;
    
    // 使用 RAII 包装线程
    ThreadRAII t(
        std::thread( {
            for (int i = 0; i <= maxVal; ++i) {
                if (filter(i)) {
                    goodVals.push_back(i);
                }
            }
        }),
        ThreadRAII::DtorAction::join  // 析构时连接
    );
    
    // 可以访问底层线程（例如设置优先级）
    auto nativeHandle = t.get().native_handle();
    // ... 设置线程优先级等操作
    
    if (conditionsAreSatisfied()) {
        t.get().join();  // 显式连接（可选，因为析构函数也会连接）
        performComputation(goodVals);
        return true;
    }
    
    // 如果 conditionsAreSatisfied() 返回 false
    // 或者抛出异常，ThreadRAII 析构函数会自动处理线程
    return false;
}

// 测试函数
int main() {
    auto filter = int x { return x % 2 == 0; };  // 过滤偶数
    
    bool result = doWorkSafe(filter, 100);
    
    if (result) {
        std::cout << "Computation performed successfully\n";
    } else {
        std::cout << "Computation not performed\n";
    }
    
    return 0;
}
```

### 6.2 异常安全示例
```cpp
#include <iostream>
#include <thread>
#include <stdexcept>

void mightThrow() {
    throw std::runtime_error("Something went wrong!");
}

void threadFunction() {
    std::this_thread::sleep_for(std::chrono::seconds(2));
    std::cout << "Thread completed\n";
}

void exceptionSafeExample() {
    // 使用 ThreadRAII 确保异常安全
    ThreadRAII t(
        std::thread(threadFunction),
        ThreadRAII::DtorAction::join
    );
    
    // 如果这里抛出异常，t 的析构函数会被调用
    // 线程会被正确连接，程序不会终止
    mightThrow();
    
    t.get().join();  // 正常情况下的连接
    std::cout << "Function completed normally\n";
}

int main() {
    try {
        exceptionSafeExample();
    } catch (const std::exception& e) {
        std::cout << "Caught exception: " << e.what() << "\n";
    }
    return 0;
}
```

## 7. 移动语义支持

### 7.1 默认移动操作
由于 `ThreadRAII` 声明了析构函数，编译器不会自动生成移动操作。我们需要显式请求它们：

```cpp
class ThreadRAII {
public:
    // ... 其他成员 ...
    
    // 支持移动构造
    ThreadRAII(ThreadRAII&& other) noexcept
        : action(other.action), thread(std::move(other.thread)) {
    }
    
    // 支持移动赋值
    ThreadRAII& operator=(ThreadRAII&& other) noexcept {
        if (this != &other) {
            // 首先处理当前线程
            if (thread.joinable()) {
                if (action == DtorAction::join) {
                    thread.join();
                } else {
                    thread.detach();
                }
            }
            
            // 然后接管新线程
            action = other.action;
            thread = std::move(other.thread);
        }
        return *this;
    }
    
    // ... 其他成员 ...
};
```

### 7.2 使用 = default
在C++11中，我们可以使用 `= default` 来请求编译器生成移动操作：

```cpp
class ThreadRAII {
public:
    // ... 构造函数、析构函数等 ...
    
    // 使用 = default 请求移动操作
    ThreadRAII(ThreadRAII&&) = default;
    ThreadRAII& operator=(ThreadRAII&&) = default;
    
    // ... 其他成员 ...
};
```

**注意**：使用 `= default` 时，要确保所有成员都支持移动操作。在我们的例子中，`DtorAction` 是枚举，`std::thread` 支持移动，所以是安全的。

## 8. 线程管理的最佳实践

### 8.1 连接 vs 分离
何时选择连接，何时选择分离？

**选择连接（join）** 当：
- 需要线程的执行结果
- 线程访问调用者的局部变量
- 需要确保线程在继续之前已完成
- 想要处理线程中可能抛出的异常

**选择分离（detach）** 当：
- 线程是真正的后台任务
- 线程不访问任何局部变量
- 线程的生命周期独立于创建它的作用域
- 不关心线程何时完成

### 8.2 改进的 ThreadRAII
```cpp
class ThreadRAII {
public:
    // 使用枚举类避免命名冲突
    enum class DtorAction { join, detach };
    
    // 构造函数：可以指定线程函数和参数
    template<typename Function, typename... Args>
    explicit ThreadRAII(DtorAction action, Function&& func, Args&&... args)
        : action(action),
          thread(std::forward<Function>(func), std::forward<Args>(args)...) {
    }
    
    ~ThreadRAII() {
        if (thread.joinable()) {
            if (action == DtorAction::join) {
                thread.join();
            } else {
                thread.detach();
            }
        }
    }
    
    // 删除拷贝操作
    ThreadRAII(const ThreadRAII&) = delete;
    ThreadRAII& operator=(const ThreadRAII&) = delete;
    
    // 允许移动
    ThreadRAII(ThreadRAII&&) = default;
    ThreadRAII& operator=(ThreadRAII&&) = default;
    
    // 获取底层线程
    std::thread& get() { return thread; }
    const std::thread& get() const { return thread; }
    
    // 检查线程是否可连接
    bool joinable() const { return thread.joinable(); }
    
    // 显式连接/分离
    void join() { thread.join(); }
    void detach() { thread.detach(); }
    
private:
    DtorAction action;
    std::thread thread;
};
```

### 8.3 使用示例
```cpp
// 创建线程并自动管理
ThreadRAII worker(ThreadRAII::DtorAction::join,  {
    std::cout << "Worker thread running\n";
    std::this_thread::sleep_for(std::chrono::seconds(1));
    std::cout << "Worker thread completed\n";
});

// 线程在 worker 析构时自动连接
```

## 9. 常见陷阱与调试技巧

### 9.1 悬挂引用
```cpp
void danglingReferenceExample() {
    int localValue = 42;
    
    ThreadRAII t(
        ThreadRAII::DtorAction::detach,
         {  // 危险：捕获局部变量引用
            std::this_thread::sleep_for(std::chrono::seconds(1));
            std::cout << localValue << "\n";  // 未定义行为！
        }
    );
    
    // 函数返回，localValue 被销毁
    // 但分离的线程可能还在运行并试图访问它
}
```

**修复**：通过值捕获局部变量：
```cpp
ThreadRAII t(
    ThreadRAII::DtorAction::detach,
     {  // 通过值捕获
        std::this_thread::sleep_for(std::chrono::seconds(1));
        std::cout << localValue << "\n";  // 安全
    }
);
```

### 9.2 死锁风险
```cpp
void potentialDeadlock() {
    std::mutex mtx1, mtx2;
    
    ThreadRAII t1(
        ThreadRAII::DtorAction::join,
         {
            std::lock_guard<std::mutex> lock1(mtx1);
            std::this_thread::sleep_for(std::chrono::milliseconds(100));
            std::lock_guard<std::mutex> lock2(mtx2);  // 可能死锁
        }
    );
    
    std::lock_guard<std::mutex> lock2(mtx2);
    std::this_thread::sleep_for(std::chrono::milliseconds(100));
    std::lock_guard<std::mutex> lock1(mtx1);  // 可能死锁
    
    t1.join();  // 如果死锁，这里会永远阻塞
}
```

### 9.3 调试技巧
1. **使用线程命名**（平台相关）帮助调试
2. **记录线程ID** 以便跟踪
3. **使用作用域守卫** 记录线程生命周期

```cpp
class ScopedThreadLogger {
public:
    explicit ScopedThreadLogger(const std::string& name)
        : name(name), start(std::chrono::steady_clock::now()) {
        std::cout << "Thread " << name << " started\n";
    }
    
    ~ScopedThreadLogger() {
        auto end = std::chrono::steady_clock::now();
        auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
        std::cout << "Thread " << name << " ended after " << duration.count() << "ms\n";
    }
    
private:
    std::string name;
    std::chrono::steady_clock::time_point start;
};

void monitoredThreadFunction() {
    ScopedThreadLogger logger("WorkerThread");
    // ... 线程工作 ...
}
```

## 10. 总结

### 10.1 关键要点
1. **永远不要让可连接的 `std::thread` 被销毁**：这会导致程序终止。
2. **使用RAII管理线程**：`ThreadRAII` 类确保线程在所有退出路径上都被正确处理。
3. **在类中最后声明 `std::thread` 成员**：确保当线程开始运行时，所有其他成员都已初始化。
4. **明确连接与分离的选择**：连接更安全，分离适合独立的后台任务。
5. **注意线程生命周期**：避免悬挂引用和资源泄漏。

### 10.2 决策指南
| 场景 | 建议 | 原因 |
|------|------|------|
| 需要线程结果 | 使用连接 | 确保结果可用 |
| 后台任务 | 使用分离 | 不阻塞主线程 |
| 访问局部变量 | 使用连接 | 避免悬挂引用 |
| 可能抛出异常 | 使用连接 | 异常安全 |
| 独立任务 | 使用分离 | 资源自动清理 |

### 10.3 最终建议
1. **优先使用任务（task）而非原始线程**：`std::async` 通常更安全。
2. **当需要线程控制时使用 `ThreadRAII`**：如设置优先级或亲和性。
3. **编写异常安全的代码**：确保异常不会导致线程泄漏。
4. **测试并发场景**：特别是在重负载和错误条件下。

通过遵循这些准则，你可以编写出安全、健壮的多线程C++代码，避免常见的线程管理陷阱，并确保程序在所有执行路径上都能正确运行。