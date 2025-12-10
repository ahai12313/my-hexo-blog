---
title: 'Item 35: Prefer task-based programming to thread based'
categories: Effective C++
date: 2025-12-10 21:55:00
tags:
priority: 35
---
# 条款35：优先使用基于任务的编程而非基于线程的编程

## 概述

在C++并发编程中，开发者在异步执行任务时有两个基本选择：基于线程的直接编程和基于任务的抽象编程。本条款详细分析两者的优劣，明确推荐在大多数情况下优先使用基于任务的方法。

## 核心概念对比

### 基于线程的编程
```cpp
int doAsyncWork();
std::thread t(doAsyncWork);  // 直接创建线程执行任务
```

### 基于任务的编程
```cpp
int doAsyncWork();
auto fut = std::async(doAsyncWork);  // 将任务提交给运行时系统
```

## 为什么基于任务的编程更优

### 1. 更好的结果处理机制
- **返回值获取**：`std::async`返回的`std::future`通过`get()`方法提供简单的返回值获取机制
- **异常传播**：任务抛出的异常可通过`get()`传播到调用方，避免了线程未处理异常导致程序终止的问题
- **代码简洁性**：无需手动实现线程间通信机制

### 2. 自动化的线程管理
基于线程的编程需要手动处理以下问题，而`std::async`将这些责任转移给标准库实现：

| 问题 | 基于线程编程 | 基于任务编程 |
|------|------------|------------|
| 线程资源耗尽 | 需手动处理异常，设计回退策略 | 运行时调度器自动处理 |
| 超额订阅 | 需根据硬件特性手动优化 | 调度器自动负载均衡 |
| 负载均衡 | 难以实现跨进程优化 | 调度器拥有系统级视角 |
| 上下文切换开销 | 需避免过多线程创建 | 调度器可复用线程池 |

### 3. 适应现代硬件架构
- **CPU核心多样性**：现代CPU架构复杂，不同核心可能有不同特性
- **缓存优化**：`std::async`实现可利用工作窃取算法优化缓存利用率
- **动态负载适应**：运行时调度器可根据系统负载动态调整线程策略

## 使用建议与最佳实践

### 默认使用场景
```cpp
// 推荐：让运行时系统决定最优执行策略
auto result1 = std::async(doAsyncWork);

// 当需要保证真正异步执行时（如GUI线程）
auto result2 = std::async(std::launch::async, doAsyncWork);

// 获取结果
try {
    int value = result1.get();  // 获取返回值，自动处理异常
} catch (const SomeException& e) {
    // 处理任务抛出的异常
}
```

### 异常安全的异步调用
```cpp
// 基于线程的方法需要显式异常处理
std::thread t([&] {
    try {
        doAsyncWork();
    } catch (...) {
        // 必须捕获所有异常
        exception_ptr p = current_exception();
        // 需要通过其他机制传递异常
    }
});

// 基于任务的方法异常处理更简单
auto fut = std::async(doAsyncWork);
// ... 稍后
try {
    fut.get();  // 异常在此处重新抛出
} catch (const MyException& e) {
    // 统一处理异常
}
```

## 需要直接使用std::thread的场景

虽然`std::async`适用于大多数情况，但以下场景仍需要直接使用`std::thread`：

### 1. 访问底层线程API
```cpp
std::thread t([]{ /* 任务代码 */ });

// 设置线程优先级（平台特定）
#ifdef _WIN32
    SetThreadPriority(t.native_handle(), THREAD_PRIORITY_HIGHEST);
#elif defined(__linux__)
    pthread_setschedparam(t.native_handle(), SCHED_FIFO, &param);
#endif
```

### 2. 实现特定线程架构
```cpp
// 实现自定义线程池
class ThreadPool {
    std::vector<std::thread> workers;
    // ... 任务队列和其他成员
    
public:
    void initialize(size_t numThreads) {
        workers.reserve(numThreads);
        for (size_t i = 0; i < numThreads; ++i) {
            workers.emplace_back(&ThreadPool::workerThread, this);
        }
    }
    // ... 其他实现
};
```

### 3. 需要精确控制线程生命周期的场景
```cpp
// 需要确保线程在对象析构前结束
class ResourceOwner {
    std::thread worker;
    std::atomic<bool> stop_flag{false};
    
public:
    ResourceOwner() : worker([this]{ backgroundTask(); }) {}
    
    ~ResourceOwner() {
        stop_flag = true;
        if (worker.joinable()) worker.join();
    }
    
    // ... 禁止复制，允许移动
};
```

## 技术细节与性能考量

### 1. 默认启动策略
```cpp
// 默认策略：std::launch::async | std::launch::deferred
auto fut = std::async(doAsyncWork);
// 运行时决定：立即异步执行 或 延迟到get()/wait()时同步执行
```

### 2. 明确启动策略
```cpp
// 强制异步执行
auto fut1 = std::async(std::launch::async, doAsyncWork);

// 延迟执行（惰性求值）
auto fut2 = std::async(std::launch::deferred, doAsyncWork);
```

### 3. 避免常见陷阱
```cpp
// 错误：临时future析构会阻塞等待完成
void processData() {
    std::async(std::launch::async, []{ 
        processChunk1(); 
    });
    std::async(std::launch::async, []{ 
        processChunk2(); 
    });  // 此处会阻塞等待第一个任务完成！
}  // 析构时会再次阻塞等待第二个任务

// 正确：保存future对象
void processDataCorrectly() {
    auto fut1 = std::async(std::launch::async, processChunk1);
    auto fut2 = std::async(std::launch::async, processChunk2);
    // ... 并行执行
}
```

## 示例：实际应用对比

### 场景：并行计算多个值
```cpp
// 基于线程的实现（复杂且容易出错）
std::vector<std::thread> threads;
std::vector<int> results(4);
std::exception_ptr eptr;

for (int i = 0; i < 4; ++i) {
    threads.emplace_back([i, &results, &eptr] {
        try {
            results[i] = compute(i);
        } catch (...) {
            eptr = std::current_exception();
        }
    });
}

for (auto& t : threads) t.join();
if (eptr) std::rethrow_exception(eptr);

// 基于任务的实现（简洁且安全）
std::vector<std::future<int>> futures;
for (int i = 0; i < 4; ++i) {
    futures.push_back(std::async(std::launch::async, compute, i));
}

std::vector<int> results;
for (auto& fut : futures) {
    results.push_back(fut.get());  // 自动处理异常
}
```

## 总结

### 核心建议
1. **默认使用`std::async`**：适用于绝大多数并发编程场景
2. **明确启动策略**：GUI线程等需要响应性的场景使用`std::launch::async`
3. **管理future生命周期**：避免临时future导致意外阻塞
4. **仅在必要时使用`std::thread`**：当需要底层控制或实现特定并发模式时

### 决策流程图
```
开始
  ↓
需要异步执行任务？
  ├─ 是 → 有特殊线程控制需求？
  │      ├─ 是 → 使用 std::thread
  │      └─ 否 → 使用 std::async
  │             ├─ 需要确保异步执行 → 指定 std::launch::async
  │             ├─ 需要惰性求值 → 指定 std::launch::deferred
  │             └─ 无特殊要求 → 使用默认策略
  │
  └─ 否 → 同步执行
```

### 最终建议
基于任务的编程提供了更高的抽象级别，将复杂的线程管理任务委托给标准库实现。这不仅减少了错误机会，还能随着运行时系统的发展（如更智能的线程池和工作窃取算法）自动获得性能提升。仅在需要底层控制或实现特定并发架构时，才应考虑直接使用`std::thread`。