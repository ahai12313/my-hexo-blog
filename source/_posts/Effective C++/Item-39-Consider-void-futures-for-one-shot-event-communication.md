---
title: 'Item 39: Consider void futures for one-shot event communication'
categories: Effective C++
date: 2025-12-15 22:30:11
tags:
priority: 39
---
# Item 39: 考虑使用 void future 进行一次性事件通信

## 概述

在并发编程中，一个常见需求是让一个任务（检测任务）通知另一个异步运行的任务（响应任务）某个特定事件已经发生，因为响应任务必须等待该事件发生后才能继续执行。本文探讨实现这种线程间通信的最佳方式，并重点介绍使用 `std::promise<void>` 和 `std::future<void>` 的简洁方案。

## 目录

1. 通信需求场景
2. 传统方法及其问题
3. 基于 void future 的方案
4. 实现细节与示例
5. 多线程扩展：shared_future
6. 异常安全考虑
7. 性能对比分析
8. 适用场景与限制
9. 最佳实践建议

## 1. 通信需求场景

### 典型应用场景

```cpp
// 场景1：等待资源初始化完成
class ResourceManager {
    std::future<void> init_future_;
    std::promise<void> init_promise_;
    
public:
    void initialize() {
        std::thread([this] {
            load_resources();          // 耗时初始化
            init_promise_.set_value(); // 通知初始化完成
        }).detach();
    }
    
    void use_resource() {
        init_future_.wait();           // 等待初始化完成
        // 安全使用资源
    }
};

// 场景2：等待前置计算完成
void pipeline_processing() {
    std::promise<void> stage1_done;
    auto stage1_future = stage1_done.get_future();
    
    // 阶段1：数据预处理
    std::thread stage1([&stage1_done] {
        preprocess_data();
        stage1_done.set_value();
    });
    
    // 阶段2：等待阶段1完成后开始
    std::thread stage2([&stage1_future] {
        stage1_future.wait();
        process_data();
    });
    
    stage1.join();
    stage2.join();
}
```

## 2. 传统方法及其问题

### 2.1 条件变量方法

```cpp
// 使用条件变量的传统方法
class ConditionVariableApproach {
    std::condition_variable cv_;
    std::mutex mutex_;
    bool event_occurred_{false};
    
public:
    void detecting_task() {
        // 执行检测工作...
        {
            std::lock_guard<std::mutex> lock(mutex_);
            event_occurred_ = true;
        }
        cv_.notify_one();  // 通知响应任务
    }
    
    void reacting_task() {
        // 准备响应...
        {
            std::unique_lock<std::mutex> lock(mutex_);
            cv_.wait(lock, [this] { return event_occurred_; });
        }
        // 执行响应工作...
    }
};
```

**问题分析：**

1. **不必要的互斥锁**：即使不需要保护共享数据，也必须使用互斥锁
2. **顺序依赖**：如果检测任务先通知，响应任务将永远等待
3. **虚假唤醒**：条件变量可能在没有通知的情况下返回

### 2.2 原子标志方法

```cpp
// 使用原子标志的轮询方法
class AtomicFlagApproach {
    std::atomic<bool> flag_{false};
    
public:
    void detecting_task() {
        // 执行检测工作...
        flag_ = true;  // 设置标志
    }
    
    void reacting_task() {
        // 准备响应...
        while (!flag_) {  // 忙等待
            std::this_thread::yield();
        }
        // 执行响应工作...
    }
};
```

**问题分析：**

1. **CPU浪费**：忙等待消耗CPU周期
2. **响应延迟**：轮询间隔导致响应延迟
3. **能耗问题**：阻止CPU进入低功耗状态

### 2.3 混合方法（条件变量+标志）

```cpp
// 结合条件变量和标志
class HybridApproach {
    std::condition_variable cv_;
    std::mutex mutex_;
    bool event_occurred_{false};
    
public:
    void detecting_task() {
        {
            std::lock_guard<std::mutex> lock(mutex_);
            event_occurred_ = true;
        }
        cv_.notify_one();
    }
    
    void reacting_task() {
        {
            std::unique_lock<std::mutex> lock(mutex_);
            cv_.wait(lock, [this] { return event_occurred_; });
        }
        // 响应...
    }
};
```

**问题分析：**

1. **设计冗余**：需要同时维护条件和标志
2. **复杂性**：需要正确管理互斥锁和条件变量
3. **仍有虚假唤醒**：虽然通过谓词解决，但仍需检查

## 3. 基于 void future 的方案

### 3.1 基本概念

`std::promise<void>` 和 `std::future<void>` 提供了一种简洁的一次性事件通信机制：

```cpp
class FutureApproach {
    std::promise<void> promise_;
    std::future<void> future_{promise_.get_future()};
    
public:
    void detecting_task() {
        // 执行检测工作...
        promise_.set_value();  // 发出事件信号
    }
    
    void reacting_task() {
        future_.wait();        // 等待事件发生
        // 执行响应工作...
    }
};
```

### 3.2 工作原理

```
检测任务                   共享状态                    响应任务
    |                         |                          |
    | --- set_value() ----->  |                          |
    |                         | --- 状态变为"就绪" --->  |
    |                         |                          | wait()返回
    |                         |                          | --- 继续执行 -->
```

### 3.3 完整示例

```cpp
#include <iostream>
#include <future>
#include <thread>
#include <chrono>

class EventNotifier {
    std::promise<void> data_ready_promise_;
    std::future<void> data_ready_future_{data_ready_promise_.get_future()};
    
public:
    void data_producer() {
        std::cout << "生产者: 开始准备数据..." << std::endl;
        std::this_thread::sleep_for(std::chrono::seconds(2));  // 模拟耗时操作
        
        std::cout << "生产者: 数据准备完成，通知消费者" << std::endl;
        data_ready_promise_.set_value();  // 设置promise，发出信号
    }
    
    void data_consumer() {
        std::cout << "消费者: 等待数据..." << std::endl;
        data_ready_future_.wait();  // 等待promise被设置
        
        std::cout << "消费者: 收到通知，开始处理数据" << std::endl;
        // 处理数据...
    }
};

int main() {
    EventNotifier notifier;
    
    std::thread consumer(&EventNotifier::data_consumer, &notifier);
    std::this_thread::sleep_for(std::chrono::milliseconds(500));  // 让消费者先启动
    std::thread producer(&EventNotifier::data_producer, &notifier);
    
    producer.join();
    consumer.join();
    
    return 0;
}
```

## 4. 实现细节与示例

### 4.1 基本用法模式

```cpp
// 模式1：简单通知
void simple_notification() {
    std::promise<void> ready_promise;
    auto ready_future = ready_promise.get_future();
    
    std::thread worker([&ready_promise] {
        std::this_thread::sleep_for(std::chrono::seconds(1));
        ready_promise.set_value();  // 通知主线程
    });
    
    std::cout << "主线程: 等待工作完成..." << std::endl;
    ready_future.wait();  // 阻塞直到收到通知
    std::cout << "主线程: 工作完成" << std::endl;
    
    worker.join();
}

// 模式2：带超时的等待
void notification_with_timeout() {
    std::promise<void> ready_promise;
    auto ready_future = ready_promise.get_future();
    
    std::thread worker([&ready_promise] {
        std::this_thread::sleep_for(std::chrono::seconds(3));  // 长时间工作
        ready_promise.set_value();
    });
    
    std::cout << "主线程: 等待工作完成（最多2秒）..." << std::endl;
    auto status = ready_future.wait_for(std::chrono::seconds(2));
    
    if (status == std::future_status::ready) {
        std::cout << "主线程: 工作及时完成" << std::endl;
    } else {
        std::cout << "主线程: 等待超时" << std::endl;
        // 可以取消工作或采取其他措施
    }
    
    worker.detach();  // 注意：工作线程仍在运行！
}
```

### 4.2 复杂示例：初始化协调

```cpp
class DatabaseConnection {
    std::promise<void> connection_ready_;
    std::future<void> connection_future_{connection_ready_.get_future()};
    std::thread init_thread_;
    
public:
    DatabaseConnection(const std::string& connection_string) {
        // 在后台线程中初始化连接
        init_thread_ = std::thread([this, connection_string] {
            initialize_connection(connection_string);
            connection_ready_.set_value();
        });
    }
    
    ~DatabaseConnection() {
        if (init_thread_.joinable()) {
            init_thread_.join();
        }
    }
    
    void execute_query(const std::string& query) {
        // 等待连接就绪
        connection_future_.wait();
        // 执行查询
        std::cout << "执行查询: " << query << std::endl;
    }
    
private:
    void initialize_connection(const std::string& conn_str) {
        std::cout << "初始化数据库连接: " << conn_str << std::endl;
        std::this_thread::sleep_for(std::chrono::seconds(2));  // 模拟耗时初始化
        std::cout << "数据库连接就绪" << std::endl;
    }
};

int main() {
    DatabaseConnection db("server=localhost;database=test");
    
    // 可以立即尝试使用，内部会等待连接就绪
    db.execute_query("SELECT * FROM users");
    db.execute_query("UPDATE stats SET value = 100");
    
    return 0;
}
```

## 5. 多线程扩展：shared_future

### 5.1 多个等待者的场景

```cpp
class MultiWaiterEvent {
    std::promise<void> event_promise_;
    std::shared_future<void> event_future_;
    
public:
    MultiWaiterEvent() : event_future_(event_promise_.get_future().share()) {}
    
    void signal_event() {
        std::cout << "发出事件信号" << std::endl;
        event_promise_.set_value();
    }
    
    void wait_for_event(int id) {
        std::cout << "等待者 " << id << ": 等待事件" << std::endl;
        event_future_.wait();  // 多个线程可以共享等待同一个future
        std::cout << "等待者 " << id << ": 收到事件" << std::endl;
    }
};

void test_multiple_waiters() {
    MultiWaiterEvent event;
    std::vector<std::thread> waiters;
    
    // 创建5个等待线程
    for (int i = 0; i < 5; ++i) {
        waiters.emplace_back(&MultiWaiterEvent::wait_for_event, &event, i);
    }
    
    std::this_thread::sleep_for(std::chrono::seconds(1));
    
    // 发出信号，所有等待者都会被唤醒
    event.signal_event();
    
    for (auto& waiter : waiters) {
        waiter.join();
    }
}
```

### 5.2 一次性广播事件

```cpp
class OneTimeBroadcaster {
    std::promise<void> broadcast_promise_;
    std::shared_future<void> broadcast_future_;
    std::vector<std::thread> listeners_;
    
public:
    OneTimeBroadcaster() : broadcast_future_(broadcast_promise_.get_future().share()) {}
    
    void add_listener(const std::string& name) {
        listeners_.emplace_back([this, name] {
            std::cout << name << ": 等待广播..." << std::endl;
            broadcast_future_.wait();
            std::cout << name << ": 收到广播消息!" << std::endl;
        });
    }
    
    void broadcast() {
        std::cout << "广播者: 开始广播" << std::endl;
        std::this_thread::sleep_for(std::chrono::seconds(1));
        broadcast_promise_.set_value();
    }
    
    ~OneTimeBroadcaster() {
        for (auto& listener : listeners_) {
            if (listener.joinable()) {
                listener.join();
            }
        }
    }
};

int main() {
    OneTimeBroadcaster broadcaster;
    
    // 添加多个监听者
    broadcaster.add_listener("监听者A");
    broadcaster.add_listener("监听者B");
    broadcaster.add_listener("监听者C");
    
    std::this_thread::sleep_for(std::chrono::milliseconds(500));
    broadcaster.broadcast();
    
    return 0;
}
```

## 6. 异常安全考虑

### 6.1 基本异常安全

```cpp
void safe_detection_with_exception() {
    std::promise<void> detection_promise;
    auto detection_future = detection_promise.get_future();
    
    std::thread detector([&detection_promise] {
        try {
            // 模拟可能失败的操作
            std::this_thread::sleep_for(std::chrono::seconds(1));
            
            // 模拟异常情况
            if (std::rand() % 2 == 0) {
                throw std::runtime_error("检测任务失败!");
            }
            
            detection_promise.set_value();  // 正常完成
        } catch (...) {
            // 捕获异常并传播
            detection_promise.set_exception(std::current_exception());
        }
    });
    
    try {
        detection_future.wait();  // 等待检测完成
        std::cout << "检测成功完成" << std::endl;
    } catch (const std::exception& e) {
        std::cerr << "检测失败: " << e.what() << std::endl;
    }
    
    detector.join();
}
```

### 6.2 RAII包装器

```cpp
class ScopedEventSignal {
    std::promise<void>& promise_;
    bool signaled_{false};
    
public:
    explicit ScopedEventSignal(std::promise<void>& promise) 
        : promise_(promise) {}
    
    ~ScopedEventSignal() {
        if (!signaled_) {
            try {
                promise_.set_value();  // 确保总是设置值
            } catch (...) {
                // 忽略析构函数中的异常
            }
        }
    }
    
    void signal() {
        if (!signaled_) {
            promise_.set_value();
            signaled_ = true;
        }
    }
    
    // 禁止拷贝
    ScopedEventSignal(const ScopedEventSignal&) = delete;
    ScopedEventSignal& operator=(const ScopedEventSignal&) = delete;
};

void safe_resource_initialization() {
    std::promise<void> init_promise;
    auto init_future = init_promise.get_future();
    
    std::thread initializer([&init_promise] {
        ScopedEventSignal signaler(init_promise);  // RAII包装
        
        initialize_resource();  // 可能抛出异常
        
        signaler.signal();  // 显式发出信号
        // 如果异常抛出，signaler的析构函数会设置promise
    });
    
    init_future.wait();
    std::cout << "资源初始化完成" << std::endl;
    
    initializer.join();
}
```

## 7. 性能对比分析

### 7.1 基准测试

```cpp
#include <chrono>
#include <iostream>
#include <future>
#include <condition_variable>
#include <atomic>

class PerformanceBenchmark {
public:
    // 测试条件变量方法
    static void test_condition_variable(int iterations) {
        auto start = std::chrono::high_resolution_clock::now();
        
        for (int i = 0; i < iterations; ++i) {
            std::condition_variable cv;
            std::mutex mtx;
            bool ready = false;
            
            std::thread responder([&] {
                std::unique_lock<std::mutex> lock(mtx);
                cv.wait(lock, [&] { return ready; });
            });
            
            std::thread detector([&] {
                std::lock_guard<std::mutex> lock(mtx);
                ready = true;
                cv.notify_one();
            });
            
            detector.join();
            responder.join();
        }
        
        auto end = std::chrono::high_resolution_clock::now();
        auto duration = std::chrono::duration_cast<std::chrono::microseconds>(end - start);
        std::cout << "条件变量: " << duration.count() << " 微秒" << std::endl;
    }
    
    // 测试future方法
    static void test_future(int iterations) {
        auto start = std::chrono::high_resolution_clock::now();
        
        for (int i = 0; i < iterations; ++i) {
            std::promise<void> p;
            auto f = p.get_future();
            
            std::thread responder([&] {
                f.wait();
            });
            
            std::thread detector([&] {
                p.set_value();
            });
            
            detector.join();
            responder.join();
        }
        
        auto end = std::chrono::high_resolution_clock::now();
        auto duration = std::chrono::duration_cast<std::chrono::microseconds>(end - start);
        std::cout << "Future方法: " << duration.count() << " 微秒" << std::endl;
    }
    
    // 测试原子标志方法（忙等待）
    static void test_atomic_flag(int iterations) {
        auto start = std::chrono::high_resolution_clock::now();
        
        for (int i = 0; i < iterations; ++i) {
            std::atomic<bool> flag{false};
            
            std::thread responder([&] {
                while (!flag.load()) {
                    std::this_thread::yield();
                }
            });
            
            std::thread detector([&] {
                flag.store(true);
            });
            
            detector.join();
            responder.join();
        }
        
        auto end = std::chrono::high_resolution_clock::now();
        auto duration = std::chrono::duration_cast<std::chrono::microseconds>(end - start);
        std::cout << "原子标志: " << duration.count() << " 微秒" << std::endl;
    }
};

int main() {
    const int iterations = 10000;
    
    std::cout << "性能对比（" << iterations << "次迭代）:" << std::endl;
    PerformanceBenchmark::test_condition_variable(iterations);
    PerformanceBenchmark::test_future(iterations);
    PerformanceBenchmark::test_atomic_flag(iterations);
    
    return 0;
}
```

### 7.2 内存使用分析

```cpp
class MemoryAnalysis {
public:
    struct MemoryUsage {
        size_t condition_variable_size;
        size_t future_promise_size;
        size_t atomic_flag_size;
    };
    
    static MemoryUsage analyze() {
        MemoryUsage usage;
        
        // 条件变量方法的内存占用
        {
            std::condition_variable cv;
            std::mutex mtx;
            bool flag = false;
            
            usage.condition_variable_size = sizeof(cv) + sizeof(mtx) + sizeof(flag);
        }
        
        // future/promise方法的内存占用
        {
            std::promise<void> p;
            std::future<void> f = p.get_future();
            
            usage.future_promise_size = sizeof(p) + sizeof(f);
            // 注意：实际还包括动态分配的共享状态
        }
        
        // 原子标志方法的内存占用
        {
            std::atomic<bool> flag{false};
            usage.atomic_flag_size = sizeof(flag);
        }
        
        return usage;
    }
};
```

## 8. 适用场景与限制

### 8.1 适用场景

```cpp
// 场景1：一次性初始化
class LazyInitializer {
    std::once_flag init_flag_;
    std::promise<void> init_promise_;
    std::future<void> init_future_{init_promise_.get_future()};
    
public:
    void initialize() {
        std::call_once(init_flag_, [this] {
            perform_initialization();
            init_promise_.set_value();
        });
    }
    
    void use_resource() {
        init_future_.wait();  // 等待初始化完成
        // 使用资源...
    }
};

// 场景2：阶段同步
class PipelineStage {
    std::promise<void> stage_complete_;
    std::future<void> next_stage_ready_;
    
public:
    void set_next_stage(PipelineStage& next) {
        next_stage_ready_ = next.get_ready_future();
    }
    
    void execute() {
        if (next_stage_ready_.valid()) {
            next_stage_ready_.wait();  // 等待下一阶段就绪
        }
        
        process_data();
        stage_complete_.set_value();  // 通知本阶段完成
    }
    
    std::future<void> get_ready_future() {
        return stage_complete_.get_future();
    }
};
```

### 8.2 限制与替代方案

```cpp
// 限制1：只能使用一次
class ReusableEventNotifier {
    // future/promise只能使用一次，不能重复使用
    // 需要每次创建新的promise/future对
    
public:
    std::future<void> wait_for_event() {
        std::promise<void> p;
        auto f = p.get_future();
        // 保存promise以便稍后设置...
        return f;
    }
};

// 限制2：需要堆分配（共享状态）
// 替代方案：无锁队列或环形缓冲区
template<typename T>
class LockFreeEventChannel {
    std::atomic<int> signal_count_{0};
    
public:
    void signal() {
        signal_count_.fetch_add(1, std::memory_order_release);
    }
    
    void wait() {
        while (signal_count_.load(std::memory_order_acquire) == 0) {
            std::this_thread::yield();
        }
        signal_count_.fetch_sub(1, std::memory_order_relaxed);
    }
};
```

## 9. 最佳实践建议

### 9.1 通用指导原则

1. **优先考虑语义清晰性**：在简单的一次性事件通信中，future/promise 方法通常比条件变量更清晰。
2. **注意生命周期管理**：确保 promise 在 future 等待期间保持有效。
3. **处理异常情况**：使用 try-catch 确保 promise 总是被设置（值或异常）。
4. **考虑性能需求**：在性能关键路径上，评估 future/promise 的开销是否可接受。

### 9.2 代码示例

```cpp
class EventCoordinator {
    std::promise<void> completion_promise_;
    std::shared_future<void> completion_future_;
    
public:
    EventCoordinator() 
        : completion_future_(completion_promise_.get_future().share()) {}
    
    // 启动多个工作线程
    void start_workers(int num_workers) {
        for (int i = 0; i < num_workers; ++i) {
            std::thread([this, i] {
                worker_task(i);
            }).detach();
        }
    }
    
    // 发出完成信号
    void signal_completion() {
        try {
            completion_promise_.set_value();
        } catch (const std::future_error& e) {
            // 防止重复设置
            if (e.code() != std::future_errc::promise_already_satisfied) {
                throw;
            }
        }
    }
    
    // 等待所有工作完成
    void wait_for_completion() {
        completion_future_.wait();
    }
    
private:
    void worker_task(int id) {
        std::cout << "Worker " << id << ": 等待开始信号" << std::endl;
        completion_future_.wait();  // 所有worker等待同一个信号
        
        std::cout << "Worker " << id << ": 开始工作" << std::endl;
        // 执行实际工作...
    }
};

// 使用示例
void example_usage() {
    EventCoordinator coordinator;
    
    // 启动5个工作线程
    coordinator.start_workers(5);
    
    std::this_thread::sleep_for(std::chrono::seconds(1));
    std::cout << "协调者: 发出开始信号" << std::endl;
    
    // 通知所有worker开始工作
    coordinator.signal_completion();
    
    // 等待所有worker完成（如果需要）
    // coordinator.wait_for_completion();
    
    std::this_thread::sleep_for(std::chrono::seconds(2));
}
```

### 9.3 设计模式总结

| 方法 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| **条件变量** | 可重复使用，灵活 | 需要互斥锁，可能虚假唤醒 | 复杂同步，可重复通知 |
| **原子标志** | 无锁，低延迟 | 忙等待，浪费CPU | 非常短的等待时间 |
| **Future/Promise** | 简洁，无虚假唤醒 | 一次性使用，有堆分配开销 | 一次性事件，简单同步 |

## 结论

`std::promise<void>` 和 `std::future<void>` 为一次性事件通信提供了优雅、安全的解决方案。虽然有其局限性（主要是只能使用一次），但在许多场景下，它们比传统的条件变量或原子标志方法更简洁、更安全。

**关键收获：**
1. 使用 void future 避免了许多与条件变量相关的常见错误
2. 对于一次性事件，这是最简洁的解决方案
3. 使用 `std::shared_future<void>` 支持多个等待者
4. 总是考虑异常安全性，确保 promise 被正确设置
5. 在性能关键代码中评估开销，但对于大多数应用，简洁性和安全性更重要

当你需要简单的线程间事件通知时，考虑使用 void future——它可能是解决你问题的最优雅方案。