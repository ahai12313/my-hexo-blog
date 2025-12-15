---
title: supplement 28 CAS vs 锁：深度比较
categories: Supplement C++
date: 2025-12-15 22:37:30
tags:
priority: 28
---
# CAS vs 锁：深度比较

## 核心概念对比

| 特性 | CAS（比较并交换） | 锁（互斥锁） |
|------|----------------|------------|
| 并发控制模型 | 乐观并发控制 | 悲观并发控制 |
| 基本思想 | "我相信冲突不常发生，先尝试操作，失败就重试" | "我假设冲突经常发生，先获取独占权再操作" |
| 实现方式 | 硬件原子指令 | 操作系统原语（系统调用） |
| 阻塞行为 | 非阻塞（忙等待） | 阻塞（线程挂起） |
| 竞争处理 | 重试直到成功 | 等待锁释放 |

## 代码实现对比

### 1. 计数器示例

**CAS实现：**
```cpp
#include <atomic>

class CAS_Counter {
    std::atomic<int> value{0};
    
public:
    void increment() {
        int expected = value.load();
        int desired = expected + 1;
        
        // 乐观锁：先尝试，冲突则重试
        while (!value.compare_exchange_weak(expected, desired)) {
            desired = expected + 1;  // 基于新值重新计算
        }
    }
    
    int get() { return value.load(); }
};
```

**锁实现：**
```cpp
#include <mutex>

class Lock_Counter {
    int value{0};
    std::mutex mtx;
    
public:
    void increment() {
        std::lock_guard<std::mutex> lock(mtx);  // 悲观锁：先获取独占权
        ++value;  // 安全修改
    }
    
    int get() {
        std::lock_guard<std::mutex> lock(mtx);
        return value;
    }
};
```

## 性能特性对比

### 1. 低竞争场景
```cpp
// 测试代码框架
void test_performance() {
    auto start = std::chrono::high_resolution_clock::now();
    
    // 执行大量增量操作
    for (int i = 0; i < 1'000'000; ++i) {
        // CAS或锁的increment
    }
    
    auto end = std::chrono::high_resolution_clock::now();
    // 比较时间
}
```

| 场景 | CAS性能 | 锁性能 | 原因 |
|------|---------|--------|------|
| 单线程 | 优秀 | 一般 | CAS：几条指令；锁：仍需系统调用 |
| 低竞争 | 优秀 | 良好 | CAS：很少冲突；锁：少量竞争 |
| 高竞争 | 可能较差 | 可能较好 | CAS：大量重试；锁：线程阻塞，减少CPU浪费 |

## 内存和CPU使用

### 2. 资源消耗对比
```cpp
// 模拟高竞争场景
void high_contention_test(int num_threads) {
    // 使用CAS：CPU使用率100%，所有线程忙等
    // 使用锁：CPU使用率可能较低，线程在等待队列中休眠
}
```

| 资源 | CAS | 锁 |
|------|-----|----|
| CPU使用 | 高（忙等待） | 低（阻塞时释放CPU） |
| 内存访问 | 大量原子操作 | 较少原子操作 |
| 缓存一致性 | 频繁缓存行失效 | 较少的缓存行失效 |
| 系统调用 | 无 | 有（进入内核） |

## 复杂度和可维护性

### 3. 链表插入示例

**CAS实现（复杂）：**
```cpp
template<typename T>
class LockFreeStack {
    struct Node {
        T data;
        Node* next;
    };
    
    std::atomic<Node*> head{nullptr};
    
public:
    void push(const T& value) {
        Node* newNode = new Node{value, nullptr};
        
        do {
            newNode->next = head.load();  // 读取当前头节点
        } while (!head.compare_exchange_weak(
            newNode->next,  // 期望值
            newNode         // 新值
        ));
    }
    
    // 弹出操作更复杂，需要处理ABA问题
};
```

**锁实现（简单）：**
```cpp
template<typename T>
class ThreadSafeStack {
    struct Node {
        T data;
        Node* next;
    };
    
    Node* head{nullptr};
    std::mutex mtx;
    
public:
    void push(const T& value) {
        Node* newNode = new Node{value, nullptr};
        std::lock_guard<std::mutex> lock(mtx);
        newNode->next = head;
        head = newNode;
    }
    
    // 弹出操作简单明了
    bool pop(T& value) {
        std::lock_guard<std::mutex> lock(mtx);
        if (!head) return false;
        
        Node* oldHead = head;
        value = oldHead->data;
        head = head->next;
        delete oldHead;
        return true;
    }
};
```

## 高级特性对比

### 4. 可组合性
```cpp
// 锁可以组合
class BankAccount {
    double balance{0};
    std::mutex mtx;
    
public:
    bool transfer(BankAccount& to, double amount) {
        // 需要锁定两个账户
        std::lock(mtx, to.mtx);  // 同时锁定，避免死锁
        std::lock_guard<std::mutex> lock1(mtx, std::adopt_lock);
        std::lock_guard<std::mutex> lock2(to.mtx, std::adopt_lock);
        
        if (balance < amount) return false;
        
        balance -= amount;
        to.balance += amount;
        return true;
    }
};
// CAS难以实现这种组合操作
```

## 死锁和活性问题

### 5. 常见问题
```cpp
// CAS的问题
void cas_issues() {
    // 1. ABA问题
    // 值从A变成B又变回A，某些CAS操作可能错误成功
    // 需要版本号或标签来解决
}

// 锁的问题
void lock_issues() {
    // 1. 死锁
    std::mutex mtx1, mtx2;
    
    // 线程1
    mtx1.lock();
    mtx2.lock();  // 如果线程2先锁mtx2，就死锁
    
    // 2. 优先级反转
    // 3. 饥饿
}
```

## 适用场景指南

### 6. 何时使用CAS？

```cpp
// 场景1：简单的原子操作
std::atomic<int> counter;

// 场景2：高性能计数器
class HighPerfCounter {
    std::atomic<int64_t> value;
    // 使用CAS实现递增
};

// 场景3：无锁数据结构
template<typename T>
class LockFreeQueue {
    // 使用CAS实现入队出队
};
```

**适合CAS的场景：**
- 操作简单（递增、比较、设置标志）
- 竞争程度低到中等
- 不允许阻塞（实时系统）
- 需要最大并发度

### 7. 何时使用锁？

```cpp
// 场景1：复杂临界区
class ComplexOperation {
    std::vector<int> data;
    std::mutex mtx;
    
    void process() {
        std::lock_guard<std::mutex> lock(mtx);
        // 多个相关操作
        data.push_back(1);
        std::sort(data.begin(), data.end());
        data.erase(std::unique(data.begin(), data.end()), data.end());
    }
};

// 场景2：IO操作
class FileHandler {
    std::mutex mtx;
    std::ofstream file;
    
    void write(const std::string& msg) {
        std::lock_guard<std::mutex> lock(mtx);
        file << msg << std::endl;
    }
};
```

**适合锁的场景：**
- 复杂操作序列
- 高竞争环境
- 需要处理多种资源
- 代码可维护性更重要
- 涉及I/O操作

## 现代实践建议

### 8. 混合使用
```cpp
// 使用锁保护大部分代码，CAS优化热点路径
class OptimizedCounter {
    struct AlignedCounter {
        alignas(64) std::atomic<int> value{0};  // 缓存行对齐
    };
    
    std::vector<AlignedCounter> per_thread_counters;
    std::mutex merge_mtx;
    
public:
    // 每个线程更新自己的计数器（无竞争）
    void increment_local(int thread_id) {
        ++per_thread_counters[thread_id].value;
    }
    
    // 定期合并（需要锁）
    int get_total() {
        std::lock_guard<std::mutex> lock(merge_mtx);
        int total = 0;
        for (auto& counter : per_thread_counters) {
            total += counter.value.load();
        }
        return total;
    }
};
```

## 总结表格

| 方面 | CAS | 锁 |
|------|-----|----|
| **编程复杂度** | 高，容易出错 | 低，易于理解 |
| **调试难度** | 高（数据竞争难以复现） | 中（死锁可调试） |
| **可组合性** | 差 | 好 |
| **内存使用** | 低 | 中（需要锁对象） |
| **吞吐量** | 低竞争时高 | 高竞争时可能更好 |
| **延迟** | 可预测 | 不可预测（可能阻塞） |
| **扩展性** | 好 | 一般 |
| **适用数据结构** | 简单结构 | 任意复杂结构 |
| **系统调用** | 无 | 有（可能进入内核） |
| **优先级反转** | 无 | 可能发生 |
| **实时性** | 适合（无阻塞） | 不适合（可能阻塞） |

## 实际建议

1. **优先使用锁**：大部分场景下，锁更简单、更安全
2. **Profile后优化**：确认锁是性能瓶颈后再考虑CAS
3. **考虑读写锁**：读多写少场景用`std::shared_mutex`
4. **使用原子类型**：简单操作用`std::atomic`
5. **无锁数据结构**：使用经过验证的库（如Boost.Lockfree）
6. **避免自己实现**：除非是专家，否则不要实现复杂的无锁数据结构

```cpp
// 好的默认选择
std::mutex mtx;           // 大多数情况
std::shared_mutex rw_mtx; // 读多写少
std::atomic<int> counter; // 简单计数器
std::atomic_flag flag;     // 简单标志

// 特殊需求再考虑
bool need_lock_free = false;  // 需要仔细评估
```

记住：**正确性 > 性能**。CAS虽然性能可能更好，但复杂性和风险也更高。现代锁的实现已经非常优化，在大部分场景下足够高效。