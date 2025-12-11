---
title: 'Item 36: Specify std::launch::async if asynchronicity is essential'
categories: Effective C++
date: 2025-12-11 20:19:27
tags:
priority: 36
---
# Item 36: 如果异步是必要的，请指定 std::launch::async

## 摘要
`std::async` 是C++11引入的并发编程工具，它提供了一种高级抽象来异步执行任务。然而，它的默认启动策略包含令人惊讶的不确定性，可能导致微妙且难以调试的问题。本条目详细解释了 `std::async` 的两种启动策略，揭示了默认策略的潜在风险，并提供了安全使用 `std::async` 的最佳实践。

## 目录
1. 引言
2. std::async 的启动策略
3. 默认策略的问题
4. 线程局部存储（TLS）问题
5. 超时等待的陷阱
6. 解决方案：明确指定启动策略
7. 安全使用模式
8. 封装安全的异步函数
9. 性能考虑
10. 实际应用示例
11. 最佳实践总结
12. 附录：完整代码示例

---

## 1. 引言

`std::async` 是C++标准库提供的异步执行工具，它让并发编程变得更加简单。通过 `std::async`，我们可以轻松地将任务提交到后台执行，然后通过 `std::future` 获取结果。然而，这种便利性是有代价的：`std::async` 的默认行为并不总是如开发者预期的那样。

默认情况下，`std::async` 并不保证任务会在另一个线程上执行。相反，它允许运行时系统在两种策略之间选择：立即异步执行或延迟执行。这种灵活性虽然在某些情况下有益，但也引入了不确定性，可能导致难以调试的问题。

## 2. std::async 的启动策略

`std::async` 支持两种启动策略，通过 `std::launch` 枚举指定：

### 2.1 `std::launch::async`
- 任务**必须**在独立的线程上异步执行
- 立即启动一个新线程（或从线程池获取线程）来执行任务
- 保证真正的并发执行

```cpp
auto fut = std::async(std::launch::async,  {
    return 42;  // 在另一个线程上执行
});
```

### 2.2 `std::launch::deferred`
- 任务延迟执行，直到调用 `get()` 或 `wait()` 时才执行
- 执行是同步的，在调用 `get()` 或 `wait()` 的线程上运行
- 如果没有调用 `get()` 或 `wait()`，任务永远不会执行

```cpp
auto fut = std::async(std::launch::deferred,  {
    return 42;  // 不会立即执行
});
// ...
int result = fut.get();  // 此时才执行，同步执行
```

### 2.3 默认策略
如果不指定启动策略，`std::async` 使用默认策略，这是两种策略的按位或：

```cpp
// 以下两行代码等价
auto fut1 = std::async(func);
auto fut2 = std::async(std::launch::async | std::launch::deferred, func);
```

默认策略允许运行时系统选择最合适的执行方式，但这带来了不确定性。

## 3. 默认策略的问题

### 3.1 不确定性行为
使用默认策略，你无法预测：
1. 任务是否会与当前线程并发执行
2. 任务会在哪个线程上执行
3. 任务是否会执行（如果没有调用 `get()` 或 `wait()`）

```cpp
#include <iostream>
#include <future>
#include <thread>

void task() {
    std::cout << "Running in thread: " << std::this_thread::get_id() << std::endl;
}

int main() {
    std::cout << "Main thread: " << std::this_thread::get_id() << std::endl;
    
    auto fut = std::async(task);  // 使用默认策略
    
    // 无法预测：
    // 1. task是否会在另一个线程执行
    // 2. 如果是，是哪个线程
    // 3. 如果没有调用get()，task是否执行
    
    fut.get();  // 确保任务执行
    return 0;
}
```

### 3.2 代码路径依赖
任务是否执行可能依赖于代码路径：

```cpp
#include <iostream>
#include <future>

bool compute_result() {
    auto fut = std::async( { return 42; });
    
    if (some_condition()) {
        return fut.get() > 0;  // 任务可能在这里执行
    } else {
        return false;  // 任务可能永远不会执行
    }
}
```

## 4. 线程局部存储（TLS）问题

### 4.1 问题描述
当任务访问线程局部存储（`thread_local` 变量）时，默认策略会导致不确定性：无法预测访问的是哪个线程的TLS。

```cpp
#include <iostream>
#include <future>
#include <thread>

thread_local int tls_var = 0;

void worker() {
    tls_var = 100;  // 设置线程局部变量
    std::cout << "Worker TLS: " << tls_var << std::endl;
}

int main() {
    tls_var = 42;  // 设置主线程的TLS
    
    auto fut = std::async(worker);  // 默认策略
    
    // 问题：worker中访问的是哪个线程的tls_var？
    // 如果是新线程：输出 100
    // 如果在主线程（延迟执行）：输出 100（覆盖了主线程的42）
    
    fut.get();
    std::cout << "Main TLS: " << tls_var << std::endl;  // 可能是42或100
    return 0;
}
```

### 4.2 实际影响
这个问题在以下场景中特别危险：
- 使用线程局部日志记录器
- 线程特定的缓存
- 依赖线程局部状态的第三方库

## 5. 超时等待的陷阱

### 5.1 无限循环问题
最常见的陷阱是在使用 `wait_for()` 或 `wait_until()` 时，如果任务被延迟，会导致无限循环。

```cpp
#include <iostream>
#include <future>
#include <chrono>
#include <thread>

using namespace std::chrono_literals;

int compute() {
    std::this_thread::sleep_for(2s);
    return 42;
}

int main() {
    auto fut = std::async(compute);  // 默认策略
    
    // 危险：如果任务被延迟，这将是无限循环！
    while (fut.wait_for(100ms) != std::future_status::ready) {
        std::cout << "Waiting..." << std::endl;
    }
    
    std::cout << "Result: " << fut.get() << std::endl;
    return 0;
}
```

### 5.2 问题分析
- 如果任务异步执行，`wait_for()` 正常工作
- 如果任务被延迟，`wait_for()` 立即返回 `std::future_status::deferred`
- `deferred` 永远不会等于 `ready`，导致无限循环

## 6. 解决方案：明确指定启动策略

最简单直接的解决方案是：如果需要真正的异步执行，明确指定 `std::launch::async`。

```cpp
// 明确指定异步执行
auto fut = std::async(std::launch::async,  {
    // 这个任务保证在另一个线程上执行
    return compute_expensive_result();
});

// 明确指定延迟执行
auto fut2 = std::async(std::launch::deferred,  {
    // 这个任务在get()时同步执行
    return quick_computation();
});
```

## 7. 安全使用模式

### 7.1 检查任务是否被延迟
如果必须使用默认策略，需要检查任务是否被延迟：

```cpp
#include <iostream>
#include <future>
#include <chrono>

using namespace std::chrono_literals;

int compute() {
    // 耗时计算
    std::this_thread::sleep_for(2s);
    return 42;
}

void safe_wait_for_result() {
    auto fut = std::async(compute);  // 默认策略
    
    // 首先检查任务是否被延迟
    if (fut.wait_for(0s) == std::future_status::deferred) {
        // 任务被延迟，同步获取结果
        std::cout << "Task is deferred, executing synchronously..." << std::endl;
        int result = fut.get();
        std::cout << "Result: " << result << std::endl;
    } else {
        // 任务正在异步执行
        std::cout << "Task is running asynchronously..." << std::endl;
        
        // 安全的等待循环
        while (fut.wait_for(100ms) != std::future_status::ready) {
            std::cout << "Still waiting..." << std::endl;
            // 可以在这里做其他工作
        }
        
        int result = fut.get();
        std::cout << "Result: " << result << std::endl;
    }
}
```

### 7.2 使用包装函数
为了代码的清晰和一致性，可以创建包装函数：

```cpp
#include <future>
#include <type_traits>

// C++11版本
template<typename F, typename... Args>
auto reallyAsync(F&& f, Args&&... args)
    -> std::future<typename std::result_of<F(Args...)>::type>
{
    return std::async(std::launch::async,
                      std::forward<F>(f),
                      std::forward<Args>(args)...);
}

// C++14版本（更简洁）
template<typename F, typename... Args>
auto reallyAsync14(F&& f, Args&&... args)
{
    return std::async(std::launch::async,
                      std::forward<F>(f),
                      std::forward<Args>(args)...);
}
```

## 8. 封装安全的异步函数

### 8.1 完整的包装器实现
```cpp
#ifndef REALLY_ASYNC_H
#define REALLY_ASYNC_H

#include <future>
#include <type_traits>

namespace concurrency {
    
    // C++11版本
    template<typename F, typename... Args>
    auto reallyAsync(F&& f, Args&&... args)
        -> std::future<typename std::result_of<F(Args...)>::type>
    {
        return std::async(std::launch::async,
                          std::forward<F>(f),
                          std::forward<Args>(args)...);
    }
    
    // C++14版本（使用decltype(auto)）
    template<typename F, typename... Args>
    decltype(auto) reallyAsync14(F&& f, Args&&... args)
    {
        return std::async(std::launch::async,
                          std::forward<F>(f),
                          std::forward<Args>(args)...);
    }
    
    // 带异常处理的版本
    template<typename F, typename... Args>
    auto reallyAsyncWithException(F&& f, Args&&... args)
        -> std::future<typename std::result_of<F(Args...)>::type>
    {
        try {
            return std::async(std::launch::async,
                              std::forward<F>(f),
                              std::forward<Args>(args)...);
        } catch (const std::system_error& e) {
            // 处理线程创建失败等异常
            std::cerr << "Failed to launch async task: " << e.what() << std::endl;
            throw;
        }
    }
    
} // namespace concurrency

#endif // REALLY_ASYNC_H
```

### 8.2 使用示例
```cpp
#include "really_async.h"
#include <iostream>
#include <vector>
#include <numeric>

int compute_sum(const std::vector<int>& data) {
    return std::accumulate(data.begin(), data.end(), 0);
}

int main() {
    std::vector<int> data = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    
    // 使用包装函数，确保异步执行
    auto fut = concurrency::reallyAsync(compute_sum, data);
    
    // 主线程可以继续做其他工作
    std::cout << "Main thread is working..." << std::endl;
    
    // 获取结果
    int result = fut.get();
    std::cout << "Sum: " << result << std::endl;
    
    return 0;
}
```

## 9. 性能考虑

### 9.1 任务粒度
选择合适的任务粒度很重要：

```cpp
// 不推荐：任务太小，线程创建开销可能超过计算开销
auto fut1 = std::async(std::launch::async,  { return 1 + 1; });

// 推荐：较大的计算任务适合异步执行
auto fut2 = std::async(std::launch::async,  {
    std::vector<int> data(1000000);
    // 复杂的计算
    return process_large_data(data);
});

// 小任务使用延迟执行
auto fut3 = std::async(std::launch::deferred,  { return 1 + 1; });
```

### 9.2 线程池考虑
`std::async` 不保证使用线程池，但某些实现可能会使用。如果需要线程池，考虑使用其他库：

```cpp
// 使用Intel TBB
#include <tbb/tbb.h>

tbb::task_group tg;
tg.run( { /* 任务 */ });
tg.wait();

// 或使用线程池库
ThreadPool pool(4);
auto fut = pool.enqueue( { return 42; });
```

## 10. 实际应用示例

### 10.1 并行数据处理
```cpp
#include <iostream>
#include <future>
#include <vector>
#include <algorithm>
#include <chrono>

// 处理数据块
int process_chunk(const std::vector<int>& chunk) {
    int sum = 0;
    for (int val : chunk) {
        sum += val;
        // 模拟计算开销
        std::this_thread::sleep_for(std::chrono::microseconds(10));
    }
    return sum;
}

int main() {
    std::vector<int> data(10000);
    std::iota(data.begin(), data.end(), 0);
    
    // 分割数据
    size_t chunk_size = data.size() / 4;
    std::vector<std::future<int>> futures;
    
    auto start = std::chrono::high_resolution_clock::now();
    
    // 并行处理4个数据块
    for (size_t i = 0; i < 4; ++i) {
        auto begin = data.begin() + i * chunk_size;
        auto end = (i == 3) ? data.end() : begin + chunk_size;
        std::vector<int> chunk(begin, end);
        
        // 明确指定异步执行
        futures.push_back(std::async(std::launch::async, process_chunk, chunk));
    }
    
    // 合并结果
    int total = 0;
    for (auto& fut : futures) {
        total += fut.get();
    }
    
    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    
    std::cout << "Total: " << total << std::endl;
    std::cout << "Time: " << duration.count() << "ms" << std::endl;
    
    return 0;
}
```

### 10.2 异步I/O模拟
```cpp
#include <iostream>
#include <future>
#include <string>
#include <thread>

std::string fetch_data_from_server(const std::string& url) {
    // 模拟网络延迟
    std::this_thread::sleep_for(std::chrono::seconds(2));
    return "Data from " + url;
}

void process_data_while_waiting() {
    // 启动异步数据获取
    auto data_future = std::async(std::launch::async, 
                                  fetch_data_from_server, 
                                  "http://example.com");
    
    // 在等待数据时做其他工作
    std::cout << "Fetching data in background..." << std::endl;
    
    for (int i = 0; i < 5; ++i) {
        std::cout << "Processing other tasks... " << i << std::endl;
        std::this_thread::sleep_for(std::chrono::milliseconds(500));
    }
    
    // 获取数据（如果需要）
    try {
        std::string data = data_future.get();
        std::cout << "Received: " << data << std::endl;
    } catch (const std::exception& e) {
        std::cerr << "Error fetching data: " << e.what() << std::endl;
    }
}
```

## 11. 最佳实践总结

### 11.1 何时使用 std::launch::async
在以下情况下，总是使用 `std::launch::async`：
1. 需要真正的并发执行
2. 任务访问线程局部存储
3. 使用基于超时的等待
4. 任务必须执行（即使不调用 `get()`）
5. 需要控制执行线程的情况

### 11.2 何时使用 std::launch::deferred
在以下情况下，考虑使用 `std::launch::deferred`：
1. 任务很小，线程创建开销过大
2. 希望延迟计算，只在需要时执行
3. 需要同步执行的语义
4. 调试时希望控制执行时机

### 11.3 何时可以使用默认策略
只有在以下所有条件都满足时，才可以使用默认策略：
1. 任务不需要与调用线程并发执行
2. 不关心哪个线程的TLS被访问
3. 任务不执行也可以接受，或者保证会调用 `get()`/`wait()`
4. 没有使用基于超时的等待

### 11.4 一般建议
1. **明确意图**：总是考虑你的并发需求
2. **使用包装器**：创建 `reallyAsync` 函数确保一致性
3. **处理异常**：异步任务中的异常会传播到 `get()` 调用处
4. **资源管理**：注意异步任务的资源生命周期
5. **测试**：在不同负载下测试，因为延迟执行可能在重负载下更常见

## 12. 附录：完整代码示例

### 12.1 安全使用 std::async 的完整示例
```cpp
#include <iostream>
#include <future>
#include <chrono>
#include <thread>
#include <vector>
#include <numeric>
#include <stdexcept>

using namespace std::chrono_literals;

// 1. 基本的安全异步函数
template<typename F, typename... Args>
auto safe_async(F&& f, Args&&... args) {
    return std::async(std::launch::async,
                      std::forward<F>(f),
                      std::forward<Args>(args)...);
}

// 2. 演示TLS问题
void demonstrate_tls_issue() {
    std::cout << "\n=== Demonstrating TLS Issue ===\n";
    
    thread_local int counter = 0;
    
    auto task =  {
        counter++;
        std::cout << "Task counter: " << counter 
                  << " in thread: " << std::this_thread::get_id() << std::endl;
    };
    
    counter = 100;
    
    // 默认策略：不确定哪个线程的counter
    auto fut1 = std::async(task);
    
    // 明确异步：保证在新线程
    auto fut2 = std::async(std::launch::async, task);
    
    fut1.get();
    fut2.get();
    
    std::cout << "Main counter: " << counter << std::endl;
}

// 3. 演示超时陷阱
void demonstrate_timeout_trap() {
    std::cout << "\n=== Demonstrating Timeout Trap ===\n";
    
    auto long_task =  {
        std::this_thread::sleep_for(2s);
        return 42;
    };
    
    // 危险的方式（可能无限循环）
    auto dangerous_wait =  {
        auto fut = std::async(long_task);
        int wait_count = 0;
        
        while (fut.wait_for(100ms) != std::future_status::ready) {
            wait_count++;
            if (wait_count > 50) {  // 安全阀
                std::cout << "Breaking infinite loop!" << std::endl;
                break;
            }
        }
        
        if (wait_count <= 50) {
            std::cout << "Result: " << fut.get() << std::endl;
        }
    };
    
    // 安全的方式
    auto safe_wait =  {
        auto fut = std::async(long_task);
        
        // 检查是否延迟
        if (fut.wait_for(0s) == std::future_status::deferred) {
            std::cout << "Task is deferred, executing now..." << std::endl;
            std::cout << "Result: " << fut.get() << std::endl;
        } else {
            std::cout << "Task is running asynchronously..." << std::endl;
            int wait_count = 0;
            
            while (fut.wait_for(100ms) != std::future_status::ready) {
                wait_count++;
                if (wait_count % 10 == 0) {
                    std::cout << "Still waiting..." << std::endl;
                }
            }
            
            std::cout << "Result: " << fut.get() << " after " 
                      << wait_count * 100 << "ms" << std::endl;
        }
    };
    
    std::cout << "Dangerous wait (might loop forever if deferred):" << std::endl;
    dangerous_wait();
    
    std::cout << "\nSafe wait:" << std::endl;
    safe_wait();
}

// 4. 并行计算示例
void parallel_computation_example() {
    std::cout << "\n=== Parallel Computation Example ===\n";
    
    const size_t num_elements = 1000000;
    const size_t num_tasks = 4;
    
    std::vector<int> data(num_elements);
    std::iota(data.begin(), data.end(), 0);
    
    // 计算任务
    auto compute_chunk = const std::vector<int>& chunk {
        return std::accumulate(chunk.begin(), chunk.end(), 0);
    };
    
    // 并行计算
    std::vector<std::future<long long>> futures;
    size_t chunk_size = num_elements / num_tasks;
    
    auto start = std::chrono::high_resolution_clock::now();
    
    for (size_t i = 0; i < num_tasks; ++i) {
        auto chunk_begin = data.begin() + i * chunk_size;
        auto chunk_end = (i == num_tasks - 1) ? 
                         data.end() : chunk_begin + chunk_size;
        
        std::vector<int> chunk(chunk_begin, chunk_end);
        
        // 明确使用异步执行
        futures.push_back(safe_async(compute_chunk, chunk));
    }
    
    // 收集结果
    long long total = 0;
    for (auto& fut : futures) {
        total += fut.get();
    }
    
    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    
    std::cout << "Parallel sum: " << total << std::endl;
    std::cout << "Time: " << duration.count() << "ms" << std::endl;
    
    // 串行版本对比
    start = std::chrono::high_resolution_clock::now();
    long long serial_sum = std::accumulate(data.begin(), data.end(), 0LL);
    end = std::chrono::high_resolution_clock::now();
    duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    
    std::cout << "Serial sum: " << serial_sum << std::endl;
    std::cout << "Time: " << duration.count() << "ms" << std::endl;
}

int main() {
    std::cout << "=== Item 36: Specify std::launch::async if asynchronicity is essential ===\n";
    
    demonstrate_tls_issue();
    demonstrate_timeout_trap();
    parallel_computation_example();
    
    return 0;
}
```

### 12.2 编译和运行
```bash
# 使用C++14编译
g++ -std=c++14 -pthread -O2 item36_example.cpp -o item36_example

# 运行
./item36_example
```

## 结论

`std::async` 是一个强大的工具，但它的默认行为可能引入微妙的问题。通过理解启动策略的差异，并明确指定 `std::launch::async` 当异步执行是必要的，可以避免这些陷阱。创建和使用像 `reallyAsync` 这样的包装函数不仅可以提高代码的安全性，还能增强代码的可读性和可维护性。

记住黄金法则：**如果异步是必要的，请明确指定 `std::launch::async`**。