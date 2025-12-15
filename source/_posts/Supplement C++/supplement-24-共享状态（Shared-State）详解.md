---
title: supplement 24 共享状态（Shared State）详解
categories: Supplement C++
date: 2025-12-15 22:33:23
tags:
priority: 24
---
# 共享状态（Shared State）详解

`std::promise` 和 `std::future` 之间的**共享状态**是一个**抽象概念**，表示两者共享的内部数据结构。它是异步操作结果存储和同步的**底层机制**。

## 1. 共享状态是什么？

### 核心概念
```
┌─────────────────────────────────────────┐
│            共享状态（Shared State）       │
│  ┌──────────────────────────────────┐  │
│  │ 1. 结果值（或异常）                │  │
│  │ 2. 就绪标志（ready flag）         │  │
│  │ 3. 条件变量/通知机制               │  │
│  │ 4. 引用计数                       │  │
│  └──────────────────────────────────┘  │
└─────────────────────────────────────────┘
         ▲                    ▲
         │                    │
   可以写（set）          可以读（get）
┌──────────────┐     ┌──────────────┐
│  std::promise │     │  std::future  │
│  （生产者端）  │     │  （消费者端）  │
└──────────────┘     └──────────────┘
```

## 2. 共享状态的内部结构

```cpp
// 伪代码：共享状态的内部实现（概念上的）
struct SharedState {
    // 1. 存储的结果
    std::variant<
        std::monostate,    // 空状态
        ValueType,         // 结果值
        std::exception_ptr // 异常
    > result;
    
    // 2. 同步机制
    std::atomic<bool> ready{false};  // 结果是否就绪
    std::mutex mutex;                // 互斥锁
    std::condition_variable cond_var; // 条件变量
    
    // 3. 引用计数
    std::atomic<int> ref_count{0};   // 引用计数
    
    // 4. 其他元数据
    bool has_exception{false};       // 是否存储了异常
    bool is_shared{false};           // 是否为 shared_future
};
```

## 3. 共享状态的生命周期

### 创建共享状态
```cpp
#include <iostream>
#include <future>
#include <memory>

void demonstrate_shared_state_creation() {
    // 创建一个 promise
    std::promise<int> prom;
    
    // 内部发生的事：
    // 1. 分配一个共享状态对象
    // 2. promise 持有共享状态的写权限
    // 3. 引用计数 = 1（promise 持有）
    
    std::cout << "Promise created, shared state allocated" << std::endl;
    
    // 获取 future
    std::future<int> fut = prom.get_future();
    
    // 内部发生的事：
    // 1. 创建一个 future 对象
    // 2. future 获取共享状态的读权限
    // 3. 引用计数增加（现在为2）
    
    std::cout << "Future obtained, ref count increased" << std::endl;
}
```

### 共享状态的完整生命周期
```cpp
#include <iostream>
#include <future>
#include <thread>
#include <memory>

class TraceSharedState {
public:
    static void trace(const std::string& msg) {
        static std::mutex mtx;
        std::lock_guard<std::mutex> lock(mtx);
        std::cout << "[SharedState] " << msg << std::endl;
    }
};

void shared_state_lifecycle() {
    TraceSharedState::trace("=== 开始共享状态生命周期 ===");
    
    {
        // 阶段1: 创建共享状态
        TraceSharedState::trace("1. 创建 promise，分配共享状态");
        std::promise<std::string> prom;
        
        // 阶段2: 创建 future
        TraceSharedState::trace("2. 获取 future，增加引用计数");
        auto fut = prom.get_future();
        
        // 阶段3: 在另一个线程中设置值
        TraceSharedState::trace("3. 启动生产者线程");
        std::thread producer([&prom] {
            TraceSharedState::trace("生产者: 开始计算");
            std::this_thread::sleep_for(std::chrono::milliseconds(100));
            TraceSharedState::trace("生产者: 设置结果到共享状态");
            prom.set_value("Hello from shared state!");
            TraceSharedState::trace("生产者: 完成，promise 即将销毁");
        });
        
        // 阶段4: 获取结果
        TraceSharedState::trace("4. 消费者等待结果");
        std::string result = fut.get();
        TraceSharedState::trace("消费者: 获取结果: " + result);
        
        producer.join();
        
        // 阶段5: 离开作用域，析构顺序
        TraceSharedState::trace("5. 离开作用域，开始析构");
    }
    
    TraceSharedState::trace("=== 共享状态生命周期结束 ===");
}
```

## 4. 共享状态的操作流程

```cpp
#include <future>
#include <thread>
#include <iostream>
#include <chrono>

void shared_state_operations() {
    // 步骤1: 创建共享状态
    std::promise<int> prom;
    auto fut = prom.get_future();
    
    // 启动生产者和消费者
    std::thread producer([&prom] {
        std::cout << "Producer: Starting computation..." << std::endl;
        
        // 模拟计算
        std::this_thread::sleep_for(std::chrono::seconds(1));
        
        // 步骤2: 生产者写入共享状态
        std::cout << "Producer: Writing result to shared state" << std::endl;
        prom.set_value(42);
        
        // 内部发生了什么:
        // 1. 将值 42 存储到共享状态
        // 2. 设置 ready = true
        // 3. 通知所有等待的线程
        
        std::cout << "Producer: Result written, notifying" << std::endl;
    });
    
    std::thread consumer([&fut] {
        std::cout << "Consumer: Waiting for result..." << std::endl;
        
        // 步骤3: 消费者读取共享状态
        int result = fut.get();
        
        // 内部发生了什么:
        // 1. 检查 ready 标志
        // 2. 如果未就绪，阻塞等待通知
        // 3. 读取值
        // 4. 返回结果
        
        std::cout << "Consumer: Got result: " << result << std::endl;
    });
    
    producer.join();
    consumer.join();
}
```

## 5. 共享状态的内部实现细节

### 5.1 存储机制
```cpp
// 概念实现：共享状态如何存储不同类型
template<typename T>
class ConceptSharedState {
private:
    // 存储位置
    union Storage {
        T value;                 // 值
        std::exception_ptr exc;  // 异常
        Storage() {}             // 默认构造
        ~Storage() {}            // 析构
    } storage;
    
    // 存储什么
    enum class StoredType {
        Empty,
        Value,
        Exception
    } stored_type{StoredType::Empty};
    
    // 同步
    std::atomic<bool> ready{false};
    
public:
    // 存储值
    void set_value(const T& val) {
        new(&storage.value) T(val);  // 原位构造
        stored_type = StoredType::Value;
        ready = true;
    }
    
    // 存储异常
    void set_exception(std::exception_ptr exc) {
        new(&storage.exc) std::exception_ptr(std::move(exc));
        stored_type = StoredType::Exception;
        ready = true;
    }
    
    // 获取值
    T get() {
        if (!ready) {
            // 等待...
        }
        
        if (stored_type == StoredType::Value) {
            return storage.value;
        } else if (stored_type == StoredType::Exception) {
            std::rethrow_exception(storage.exc);
        } else {
            throw std::future_error(std::future_errc::no_state);
        }
    }
};
```

### 5.2 同步机制
```cpp
// 共享状态中的等待实现
template<typename T>
T SharedState<T>::wait_and_get() {
    std::unique_lock<std::mutex> lock(mutex);
    
    // 条件变量等待
    cond_var.wait(lock, [this] {
        return ready.load() || has_exception;
    });
    
    if (has_exception) {
        std::rethrow_exception(stored_exception);
    }
    
    return stored_value;
}
```

## 6. 共享状态与智能指针

```cpp
#include <memory>
#include <future>
#include <iostream>

// 共享状态的生命周期类似于 shared_ptr
void shared_state_analogy() {
    std::cout << "=== 共享状态与智能指针的类比 ===\n";
    
    // 类比：shared_ptr 的共享所有权
    {
        auto shared_data = std::make_shared<int>(100);
        
        // 多个 shared_ptr 共享同一数据
        auto ptr1 = shared_data;  // 引用计数: 2
        auto ptr2 = shared_data;  // 引用计数: 3
        
        std::cout << "shared_ptr 引用计数: " << shared_data.use_count() << std::endl;
    }
    
    std::cout << "\n";
    
    // promise/future 的共享状态
    {
        std::promise<int> prom;           // 创建共享状态，引用计数: 1
        auto fut = prom.get_future();     // 获取 future，引用计数: 2
        auto fut2 = fut;                  // 错误！future 不可复制
        
        // 但可以创建 shared_future
        std::promise<int> prom2;
        auto sfut = prom2.get_future().share();  // 转换为 shared_future
        
        auto sfut2 = sfut;  // 可以复制，引用计数增加
        auto sfut3 = sfut;  // 可以复制，引用计数再增加
        
        std::cout << "shared_future 允许多个消费者" << std::endl;
    }
}
```

## 7. 共享状态的内存管理

```cpp
#include <future>
#include <iostream>
#include <memory>

// 演示共享状态的内存管理
void shared_state_memory_management() {
    std::cout << "=== 共享状态内存管理 ===\n";
    
    // 情况1: 正常使用
    {
        std::promise<int> prom;
        auto fut = prom.get_future();
        
        std::thread([&prom] { prom.set_value(42); }).detach();
        
        // 获取结果
        fut.wait();
        std::cout << "Case 1: Normal usage, memory properly managed\n";
    }
    
    // 情况2: promise 被销毁，没有设置值
    {
        std::future<int> fut;
        
        {
            std::promise<int> prom;
            fut = prom.get_future();
            // promise 离开作用域，被销毁
            // 没有调用 set_value 或 set_exception
        }
        
        try {
            // 尝试获取值
            int val = fut.get();  // 抛出 std::future_error
        } catch (const std::future_error& e) {
            std::cout << "Case 2: Promise destroyed without setting value\n";
            std::cout << "  Exception: " << e.what() << std::endl;
            std::cout << "  Error code: " << e.code() << std::endl;
            
            if (e.code() == std::make_error_code(std::future_errc::broken_promise)) {
                std::cout << "  This is a broken promise!\n";
            }
        }
    }
    
    // 情况3: 多个 future 共享状态
    {
        std::promise<void> prom;
        
        // 转换为 shared_future 以实现多消费者
        std::shared_future<void> shared_fut = prom.get_future().share();
        
        // 多个线程可以等待同一个共享状态
        std::thread t1([shared_fut] {
            shared_fut.wait();
            std::cout << "Thread 1: Got notification\n";
        });
        
        std::thread t2([shared_fut] {
            shared_fut.wait();
            std::cout << "Thread 2: Got notification\n";
        });
        
        // 触发所有等待的线程
        prom.set_value();
        
        t1.join();
        t2.join();
        
        std::cout << "Case 3: Multiple consumers with shared_future\n";
    }
}
```

## 8. 共享状态的异常处理

```cpp
#include <future>
#include <iostream>
#include <stdexcept>

void shared_state_exception_handling() {
    std::cout << "=== 共享状态异常处理 ===\n";
    
    // 创建 promise 和 future
    std::promise<int> prom;
    auto fut = prom.get_future();
    
    // 在线程中抛出异常
    std::thread worker([&prom] {
        try {
            // 模拟可能抛出异常的操作
            throw std::runtime_error("Something went wrong in worker!");
        } catch (...) {
            // 捕获异常并存储到共享状态
            prom.set_exception(std::current_exception());
        }
    });
    
    // 在主线程中尝试获取结果
    try {
        int result = fut.get();  // 这里会抛出存储的异常
        std::cout << "Result: " << result << std::endl;
    } catch (const std::exception& e) {
        std::cout << "Caught exception from shared state: " << e.what() << std::endl;
        
        // 检查异常类型
        if (dynamic_cast<const std::runtime_error*>(&e)) {
            std::cout << "  It's a runtime_error\n";
        }
    }
    
    worker.join();
}
```

## 9. 共享状态的性能考虑

```cpp
#include <future>
#include <chrono>
#include <iostream>
#include <vector>

// 测试共享状态的开销
void benchmark_shared_state() {
    const int iterations = 10000;
    
    std::cout << "=== 共享状态性能测试 (" << iterations << " 次迭代) ===\n";
    
    auto start = std::chrono::high_resolution_clock::now();
    
    for (int i = 0; i < iterations; ++i) {
        // 创建和销毁共享状态
        std::promise<int> prom;
        auto fut = prom.get_future();
        
        // 立即设置值
        prom.set_value(i);
        
        // 立即获取值
        int val = fut.get();
        
        (void)val;  // 避免警告
    }
    
    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::microseconds>(end - start);
    
    std::cout << "总时间: " << duration.count() << " μs\n";
    std::cout << "平均每次操作: " << duration.count() / static_cast<double>(iterations) << " μs\n";
}

// 比较不同大小的共享状态
template<typename T>
void test_shared_state_size() {
    std::promise<T> prom;
    auto fut = prom.get_future();
    
    std::thread([&prom] {
        T value{};
        prom.set_value(std::move(value));
    }).detach();
    
    T result = fut.get();
    std::cout << "Type size: " << sizeof(T) << " bytes" << std::endl;
}
```

## 10. 共享状态的实现差异

不同标准库的实现可能有差异：

### GCC libstdc++ 实现
```cpp
// 简化的 libstdc++ 实现思路
namespace std {
    // 共享状态基类
    class _State_base {
    public:
        virtual ~_State_base();
        
        // 虚函数，用于设置值和异常
        virtual void _M_set_result(_Result_base*);
        virtual void _M_set_exception(exception_ptr);
        
    private:
        _Mutex _M_mutex;
        _Cond _M_cond;
        _Result_base* _M_result = nullptr;
        bool _M_ready = false;
    };
    
    // 模板化的共享状态
    template<typename _Res>
    class _State : public _State_base {
        _Res _M_value;  // 存储的值
    };
}
```

### MSVC STL 实现
```cpp
// 简化的 MSVC 实现思路
namespace std {
    // 使用 shared_ptr 管理共享状态
    template<class _Ty>
    class promise {
        shared_ptr<_Associated_state<_Ty>> _MyPromise;
    };
    
    template<class _Ty>
    class future {
        shared_ptr<_Associated_state<_Ty>> _MyFuture;
    };
}
```

## 11. 共享状态的使用陷阱

```cpp
#include <future>
#include <iostream>
#include <thread>

void shared_state_pitfalls() {
    std::cout << "=== 共享状态使用陷阱 ===\n";
    
    // 陷阱1: 多个 promise 设置同一个共享状态
    {
        std::promise<int> prom1;
        auto fut = prom1.get_future();
        
        std::promise<int> prom2 = std::move(prom1);  // 移动后 prom1 为空
        
        // 错误：不能有两个 promise 指向同一个共享状态
        // prom1.set_value(1);  // 崩溃！prom1 已为空
        
        prom2.set_value(2);  // 正确
        std::cout << "Trap 1: Only one promise can own a shared state\n";
    }
    
    std::cout << "\n";
    
    // 陷阱2: 多次调用 get()
    {
        std::promise<int> prom;
        auto fut = prom.get_future();
        
        std::thread([&prom] { prom.set_value(42); }).detach();
        
        int val1 = fut.get();  // 正确
        std::cout << "First get(): " << val1 << std::endl;
        
        try {
            int val2 = fut.get();  // 错误！future 已无效
        } catch (const std::future_error& e) {
            std::cout << "Trap 2: Cannot call get() twice\n";
            std::cout << "  Error: " << e.what() << std::endl;
        }
    }
    
    std::cout << "\n";
    
    // 陷阱3: 未捕获的异常
    {
        std::promise<int> prom;
        auto fut = prom.get_future();
        
        std::thread([&prom] {
            // 异常未捕获，程序终止
            // throw std::runtime_error("Uncaught!");
            
            // 正确：捕获并设置异常
            try {
                throw std::runtime_error("Caught!");
            } catch (...) {
                prom.set_exception(std::current_exception());
            }
        }).detach();
        
        try {
            fut.get();
        } catch (const std::exception& e) {
            std::cout << "Trap 3: Always catch exceptions in async tasks\n";
            std::cout << "  Exception: " << e.what() << std::endl;
        }
    }
}
```

## 总结

**共享状态**是 `std::promise` 和 `std::future` 之间的**核心通信机制**：

1. **本质**：一个内部数据结构，存储异步操作的结果和同步信息
2. **包含**：结果值/异常、就绪标志、同步原语、引用计数
3. **生命周期**：由 `promise` 和 `future` 共同管理，最后一个所有者销毁时释放
4. **线程安全**：内部实现了同步机制，确保线程安全的数据传递
5. **性能**：有一定开销，但对于异步通信是必要的
6. **异常安全**：支持异常传播，确保异常不会丢失

理解共享状态有助于：
- 正确使用 `promise`/`future` 模式
- 避免常见的并发错误
- 编写高效的异步代码
- 调试复杂的异步程序