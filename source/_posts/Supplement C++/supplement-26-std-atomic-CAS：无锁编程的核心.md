---
title: supplement 26 std::atomic+ CAS：无锁编程的核心
categories: Supplement C++
date: 2025-12-15 22:35:42
tags:
priority: 26
---
# `std::atomic` + CAS：无锁编程的核心

`std::atomic` 的 CAS（Compare-And-Swap，比较并交换）是**无锁编程**的基础，也是实现高效并发数据结构的核心机制。

## 1. 什么是CAS？

### 1.1 CAS的核心思想

CAS操作包含三个参数：
1. **内存位置**：要修改的内存地址
2. **期望值**：我们认为这个内存位置**应该**有的值
3. **新值**：如果当前值等于期望值，我们想要**设置**的新值

CAS操作原子的执行以下逻辑：
```cpp
bool compare_and_swap(int* location, int expected, int new_value) {
    if (*location == expected) {  // 比较
        *location = new_value;    // 交换
        return true;
    }
    return false;
}
```

### 1.2 可视化理解

```
内存位置: [当前值 = 5]
线程A: 期望=5, 新值=10 → 成功 (5==5) → 内存变为[10]
线程B: 期望=5, 新值=8 → 失败 (10≠5) → 内存保持[10]
```

## 2. C++中的CAS接口

### 2.1 两个主要函数

```cpp
#include <atomic>
#include <iostream>

void demonstrate_cas_functions() {
    std::atomic<int> value(10);
    int expected = 10;
    
    // 1. compare_exchange_weak: 可能虚假失败
    bool success_weak = value.compare_exchange_weak(
        expected,  // 期望值 (会更新为实际值)
        20,        // 新值
        std::memory_order_acq_rel,
        std::memory_order_acquire
    );
    std::cout << "weak CAS: " << (success_weak ? "成功" : "失败") 
              << ", 新expected: " << expected << std::endl;
    
    // 重置
    value = 10;
    expected = 10;
    
    // 2. compare_exchange_strong: 不会虚假失败
    bool success_strong = value.compare_exchange_strong(
        expected,  // 期望值 (会更新为实际值)
        20,        // 新值
        std::memory_order_acq_rel,
        std::memory_order_acquire
    );
    std::cout << "strong CAS: " << (success_strong ? "成功" : "失败")
              << ", 新expected: " << expected << std::endl;
}
```

### 2.2 weak vs strong CAS

| 特性 | `compare_exchange_weak` | `compare_exchange_strong` |
|------|-------------------------|---------------------------|
| **虚假失败** | 允许 | 不允许 |
| **性能** | 通常更高 | 通常较低 |
| **使用场景** | 循环中 | 单次尝试 |
| **平台限制** | 某些平台不支持 | 所有平台支持 |

## 3. 基本CAS模式

### 3.1 原子更新计数器

```cpp
#include <atomic>

class AtomicCounter {
    std::atomic<int> count{0};
    
public:
    void increment() {
        int expected = count.load();  // 读取当前值
        int desired = expected + 1;   // 计算新值
        
        // 尝试CAS直到成功
        while (!count.compare_exchange_weak(expected, desired)) {
            // CAS失败，expected已经被更新为当前实际值
            desired = expected + 1;  // 重新计算新值
        }
    }
    
    int get() const {
        return count.load();
    }
};
```

### 3.2 无锁栈的简单实现

```cpp
#include <atomic>
#include <memory>

template<typename T>
class LockFreeStack {
private:
    struct Node {
        T data;
        Node* next;
        
        Node(const T& data) : data(data), next(nullptr) {}
    };
    
    std::atomic<Node*> head{nullptr};
    
public:
    void push(const T& data) {
        Node* new_node = new Node(data);
        new_node->next = head.load();  // 设置新节点的next为当前头节点
        
        // 尝试CAS更新头节点
        while (!head.compare_exchange_weak(new_node->next, new_node)) {
            // CAS失败，说明有其他线程修改了head
            // new_node->next已经被更新为当前的head
            // 继续尝试
        }
    }
    
    bool pop(T& result) {
        Node* old_head = head.load();
        
        // 空栈检查
        if (!old_head) {
            return false;
        }
        
        // 尝试弹出
        while (!head.compare_exchange_weak(old_head, old_head->next)) {
            // 如果CAS失败，old_head已经被更新为当前的head
            if (!old_head) {
                return false;  // 栈变为空
            }
        }
        
        result = old_head->data;
        delete old_head;
        return true;
    }
};
```

## 4. 高级CAS应用

### 4.1 无锁队列实现

```cpp
#include <atomic>
#include <memory>

template<typename T>
class LockFreeQueue {
private:
    struct Node {
        std::shared_ptr<T> data;
        std::atomic<Node*> next;
        
        Node() : data(nullptr), next(nullptr) {}
    };
    
    std::atomic<Node*> head;
    std::atomic<Node*> tail;
    
public:
    LockFreeQueue() {
        Node* dummy = new Node();
        head.store(dummy);
        tail.store(dummy);
    }
    
    ~LockFreeQueue() {
        while (Node* node = head.load()) {
            head.store(node->next);
            delete node;
        }
    }
    
    void enqueue(T new_value) {
        std::shared_ptr<T> new_data = std::make_shared<T>(std::move(new_value));
        Node* new_node = new Node();
        
        while (true) {
            Node* old_tail = tail.load();
            Node* next = old_tail->next.load();
            
            // 检查tail是否仍然是最新的
            if (old_tail == tail.load()) {
                if (next == nullptr) {
                    // 尝试连接新节点
                    if (old_tail->next.compare_exchange_weak(next, new_node)) {
                        // 成功连接，现在尝试移动tail
                        tail.compare_exchange_weak(old_tail, new_node);
                        return;
                    }
                } else {
                    // 帮助其他线程：移动tail到下一个节点
                    tail.compare_exchange_weak(old_tail, next);
                }
            }
        }
    }
    
    bool dequeue(T& result) {
        while (true) {
            Node* old_head = head.load();
            Node* old_tail = tail.load();
            Node* next = old_head->next.load();
            
            if (old_head == head.load()) {
                if (old_head == old_tail) {
                    if (next == nullptr) {
                        return false;  // 队列为空
                    }
                    // 帮助移动tail
                    tail.compare_exchange_weak(old_tail, next);
                } else {
                    // 读取数据
                    if (next->data) {
                        result = *next->data;
                    }
                    
                    // 尝试移动head
                    if (head.compare_exchange_weak(old_head, next)) {
                        delete old_head;
                        return true;
                    }
                }
            }
        }
    }
};
```

### 4.2 无锁哈希表

```cpp
#include <atomic>
#include <vector>
#include <functional>

template<typename K, typename V>
class LockFreeHashTable {
private:
    struct Node {
        K key;
        V value;
        std::atomic<Node*> next;
        
        Node(const K& k, const V& v) : key(k), value(v), next(nullptr) {}
    };
    
    struct Bucket {
        std::atomic<Node*> head{nullptr};
        
        ~Bucket() {
            Node* current = head.load();
            while (current) {
                Node* next = current->next.load();
                delete current;
                current = next;
            }
        }
    };
    
    std::vector<Bucket> buckets;
    size_t bucket_count;
    std::hash<K> hasher;
    
public:
    LockFreeHashTable(size_t size = 64) : bucket_count(size), buckets(size) {}
    
    bool insert(const K& key, const V& value) {
        size_t index = hasher(key) % bucket_count;
        Bucket& bucket = buckets[index];
        
        Node* new_node = new Node(key, value);
        new_node->next.store(bucket.head.load());
        
        while (!bucket.head.compare_exchange_weak(new_node->next, new_node)) {
            // 检查key是否已存在
            Node* current = new_node->next.load();
            while (current) {
                if (current->key == key) {
                    delete new_node;
                    return false;  // key已存在
                }
                current = current->next.load();
            }
        }
        
        return true;
    }
    
    bool find(const K& key, V& result) {
        size_t index = hasher(key) % bucket_count;
        Bucket& bucket = buckets[index];
        
        Node* current = bucket.head.load();
        while (current) {
            if (current->key == key) {
                result = current->value;
                return true;
            }
            current = current->next.load();
        }
        
        return false;
    }
};
```

## 5. CAS的常见模式

### 5.1 CAS循环模式

```cpp
template<typename T>
class AtomicValue {
    std::atomic<T> value;
    
public:
    // 原子地增加delta
    T fetch_add(T delta) {
        T expected = value.load();
        T desired = expected + delta;
        
        while (!value.compare_exchange_weak(expected, desired)) {
            desired = expected + delta;
        }
        
        return expected;  // 返回旧值
    }
    
    // 原子地乘以因子
    T fetch_multiply(T factor) {
        T expected = value.load();
        T desired = expected * factor;
        
        while (!value.compare_exchange_weak(expected, desired)) {
            desired = expected * factor;
        }
        
        return expected;
    }
    
    // 原子地更新，使用自定义函数
    template<typename Func>
    T transform(Func&& func) {
        T expected = value.load();
        T desired = func(expected);
        
        while (!value.compare_exchange_weak(expected, desired)) {
            desired = func(expected);
        }
        
        return expected;
    }
};
```

### 5.2 双重检查锁定模式

```cpp
#include <atomic>
#include <mutex>

template<typename T>
class DoubleCheckedSingleton {
    static std::atomic<T*> instance;
    static std::mutex creation_mutex;
    
    DoubleCheckedSingleton() = default;
    
public:
    static T* get_instance() {
        T* tmp = instance.load(std::memory_order_acquire);
        if (tmp == nullptr) {
            std::lock_guard<std::mutex> lock(creation_mutex);
            tmp = instance.load(std::memory_order_relaxed);
            if (tmp == nullptr) {
                tmp = new T();
                instance.store(tmp, std::memory_order_release);
            }
        }
        return tmp;
    }
    
    static void cleanup() {
        T* old_instance = instance.exchange(nullptr);
        if (old_instance) {
            delete old_instance;
        }
    }
};
```

## 6. CAS的内存顺序

### 6.1 不同内存顺序的影响

```cpp
#include <atomic>
#include <iostream>
#include <thread>

class CASWithMemoryOrder {
    std::atomic<int> data{0};
    std::atomic<bool> ready{false};
    
public:
    void producer() {
        data.store(42, std::memory_order_relaxed);
        bool expected = false;
        
        // 使用CAS发布数据
        while (!ready.compare_exchange_weak(
            expected, 
            true,
            std::memory_order_release,  // 成功时的内存顺序
            std::memory_order_relaxed   // 失败时的内存顺序
        )) {
            expected = false;  // CAS失败后重置expected
        }
    }
    
    void consumer() {
        // 等待生产者完成
        bool expected = true;
        while (!ready.compare_exchange_weak(
            expected,
            true,
            std::memory_order_acquire,  // 成功时的内存顺序
            std::memory_order_relaxed   // 失败时的内存顺序
        )) {
            expected = true;
            std::this_thread::yield();
        }
        
        // 现在可以安全地读取data
        std::cout << "Data: " << data.load(std::memory_order_acquire) << std::endl;
    }
};
```

### 6.2 内存顺序选择指南

| 内存顺序 | CAS成功时 | CAS失败时 | 适用场景 |
|----------|-----------|-----------|----------|
| `relaxed/relaxed` | 无同步 | 无同步 | 简单的计数器，不需要同步 |
| `acquire/relaxed` | acquire | 无同步 | 读取共享数据 |
| `release/relaxed` | release | 无同步 | 发布数据 |
| `acq_rel/acquire` | acq_rel | acquire | 读-修改-写操作 |
| `seq_cst/seq_cst` | seq_cst | seq_cst | 需要严格的全局顺序 |

## 7. CAS的挑战和解决方案

### 7.1 ABA问题

**问题描述**：
```
初始: 内存值 = A
线程1: 读取A
线程2: 修改内存值为B
线程2: 修改内存值为A (又变回A)
线程1: 执行CAS(期望=A, 新值=C) → 成功！但这是错误的
```

**解决方案**：使用带有版本号的指针

```cpp
#include <atomic>

template<typename T>
struct VersionedPointer {
    T* pointer;
    uint64_t version;
    
    bool operator==(const VersionedPointer& other) const {
        return pointer == other.pointer && version == other.version;
    }
};

template<typename T>
class AtomicVersionedPointer {
    std::atomic<VersionedPointer<T>> data;
    
public:
    bool compare_exchange(T*& expected_pointer, T* new_pointer, 
                          uint64_t& expected_version) {
        VersionedPointer<T> expected{expected_pointer, expected_version};
        VersionedPointer<T> desired{new_pointer, expected_version + 1};
        
        bool success = data.compare_exchange_strong(expected, desired);
        if (!success) {
            expected_pointer = expected.pointer;
            expected_version = expected.version;
        }
        
        return success;
    }
    
    T* load(uint64_t& version) const {
        VersionedPointer<T> current = data.load();
        version = current.version;
        return current.pointer;
    }
};
```

### 7.2 内存回收问题

```cpp
#include <atomic>
#include <memory>

template<typename T>
class HazardPointerStack {
private:
    struct Node {
        T data;
        std::atomic<Node*> next;
        
        Node(const T& data) : data(data), next(nullptr) {}
    };
    
    std::atomic<Node*> head{nullptr};
    
    // 危险指针（每个线程一个）
    static thread_local Node* hazard_pointer;
    
    // 延迟删除列表
    static thread_local std::vector<Node*> retired_list;
    
    void retire(Node* node) {
        retired_list.push_back(node);
        
        // 定期清理
        if (retired_list.size() >= 10) {
            reclaim_retired();
        }
    }
    
    void reclaim_retired() {
        // 检查所有危险指针
        std::vector<Node*> still_in_use = collect_hazard_pointers();
        
        // 删除不再使用的节点
        std::vector<Node*> new_retired;
        for (Node* node : retired_list) {
            if (std::find(still_in_use.begin(), still_in_use.end(), node) == still_in_use.end()) {
                delete node;
            } else {
                new_retired.push_back(node);
            }
        }
        
        retired_list.swap(new_retired);
    }
    
public:
    void push(const T& data) {
        Node* new_node = new Node(data);
        new_node->next = head.load();
        
        while (!head.compare_exchange_weak(new_node->next, new_node)) {
            // 循环直到成功
        }
    }
    
    bool pop(T& result) {
        Node* old_head = head.load();
        
        do {
            if (!old_head) {
                return false;  // 栈为空
            }
            
            // 设置危险指针
            hazard_pointer = old_head;
            
            // 验证head没有被修改
            if (head.load() != old_head) {
                hazard_pointer = nullptr;
                old_head = head.load();
                continue;
            }
            
        } while (!head.compare_exchange_weak(old_head, old_head->next));
        
        // 清除危险指针
        hazard_pointer = nullptr;
        
        result = old_head->data;
        retire(old_head);
        return true;
    }
};
```

## 8. 性能优化技巧

### 8.1 减少CAS争用

```cpp
class OptimizedCounter {
    struct PaddedCounter {
        std::atomic<int> value;
        char padding[64];  // 填充到缓存行大小
        
        PaddedCounter() : value(0) {}
    };
    
    // 每个线程有自己的计数器
    static thread_local PaddedCounter tls_counter;
    std::vector<PaddedCounter*> all_counters;
    std::atomic<int> global_total{0};
    
public:
    void increment() {
        tls_counter.value.fetch_add(1, std::memory_order_relaxed);
        
        // 定期合并到全局计数器
        static thread_local int local_accumulator = 0;
        if (++local_accumulator >= 1000) {
            int local_value = tls_counter.value.exchange(0, std::memory_order_relaxed);
            global_total.fetch_add(local_value, std::memory_order_relaxed);
            local_accumulator = 0;
        }
    }
    
    int get() const {
        int total = global_total.load(std::memory_order_acquire);
        
        // 加上所有线程的本地计数器
        for (const auto& counter : all_counters) {
            total += counter->value.load(std::memory_order_relaxed);
        }
        
        return total;
    }
};
```

### 8.2 批量CAS操作

```cpp
#include <atomic>
#include <vector>

class BatchCASQueue {
    struct Node {
        std::atomic<int> data[8];  // 批量数据
        std::atomic<Node*> next{nullptr};
    };
    
    std::atomic<Node*> head{nullptr};
    std::atomic<Node*> tail{nullptr};
    
public:
    bool enqueue_batch(const std::vector<int>& batch) {
        if (batch.empty()) return true;
        
        Node* new_node = new Node();
        
        // 批量填充数据
        for (size_t i = 0; i < std::min(batch.size(), size_t(8)); ++i) {
            new_node->data[i].store(batch[i], std::memory_order_relaxed);
        }
        
        new_node->next.store(nullptr, std::memory_order_relaxed);
        
        // CAS连接新节点
        Node* old_tail = tail.load();
        while (!tail.compare_exchange_weak(old_tail, new_node)) {
            // 循环直到成功
        }
        
        if (old_tail) {
            old_tail->next.store(new_node, std::memory_order_release);
        } else {
            head.store(new_node, std::memory_order_release);
        }
        
        return true;
    }
};
```

## 9. 调试和测试CAS代码

### 9.1 调试技巧

```cpp
#include <atomic>
#include <iostream>
#include <thread>
#include <vector>

class DebuggableCAS {
    std::atomic<int> value{0};
    
public:
    bool compare_exchange_with_debug(int& expected, int desired, 
                                     const char* operation) {
        int before = value.load();
        bool success = value.compare_exchange_weak(expected, desired);
        int after = value.load();
        
        std::cout << operation << ": "
                  << (success ? "成功" : "失败") << ", "
                  << "期望=" << expected << ", "
                  << "目标=" << desired << ", "
                  << "之前=" << before << ", "
                  << "之后=" << after << std::endl;
        
        return success;
    }
    
    void test_race_condition() {
        std::vector<std::thread> threads;
        
        for (int i = 0; i < 5; ++i) {
            threads.emplace_back([this, i] {
                int expected = 0;
                int desired = i + 1;
                
                while (!compare_exchange_with_debug(expected, desired, 
                                                   ("线程" + std::to_string(i)).c_str())) {
                    // 失败后继续尝试
                }
            });
        }
        
        for (auto& t : threads) {
            t.join();
        }
    }
};
```

### 9.2 压力测试

```cpp
#include <atomic>
#include <thread>
#include <iostream>
#include <chrono>

class StressTest {
    std::atomic<int> counter{0};
    
public:
    void run_test(int num_threads, int iterations) {
        std::vector<std::thread> threads;
        auto start = std::chrono::high_resolution_clock::now();
        
        for (int i = 0; i < num_threads; ++i) {
            threads.emplace_back([this, iterations] {
                for (int j = 0; j < iterations; ++j) {
                    int expected = counter.load();
                    int desired = expected + 1;
                    
                    // 使用CAS递增
                    while (!counter.compare_exchange_weak(expected, desired)) {
                        desired = expected + 1;
                    }
                }
            });
        }
        
        for (auto& t : threads) {
            t.join();
        }
        
        auto end = std::chrono::high_resolution_clock::now();
        auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
        
        std::cout << "线程数: " << num_threads
                  << ", 迭代次数: " << iterations
                  << ", 总操作: " << counter.load()
                  << ", 时间: " << duration.count() << "ms"
                  << ", 吞吐量: " << (num_threads * iterations * 1000.0 / duration.count()) 
                  << " 操作/秒" << std::endl;
    }
};
```

## 总结

`std::atomic` + CAS 是C++无锁编程的核心，它允许我们构建高效、可扩展的并发数据结构。关键要点：

1. **CAS基础**：比较并交换是实现无锁算法的基本构建块
2. **weak vs strong**：在循环中使用weak，单次尝试使用strong
3. **内存顺序**：根据需求选择合适的内存顺序
4. **常见问题**：注意ABA问题、内存回收等挑战
5. **性能优化**：减少争用，使用批量操作，注意缓存友好性
6. **正确性**：无锁编程极其复杂，需要仔细测试和验证

**使用建议**：
- 如果没有必要，避免使用无锁编程
- 如果必须使用，从简单的CAS模式开始
- 始终进行压力测试和竞态条件检测
- 考虑使用现有的无锁库而不是自己实现

CAS是强大的工具，但也需要谨慎使用。正确应用的CAS可以显著提升并发性能，但错误使用可能导致难以调试的问题。