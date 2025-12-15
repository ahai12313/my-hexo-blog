---
title: supplement 27 C++ CAS 中的内存序参数详解
categories: Supplement C++
date: 2025-12-15 22:36:41
tags:
priority: 27
---
# C++ CAS 中的内存序参数详解

## 两个内存序参数的作用

在 `compare_exchange_weak` 函数中，两个内存序参数分别控制：

```cpp
bool success = atomic_var.compare_exchange_weak(
    expected,         // 期望值引用
    desired,          // 要设置的新值
    success_order,    // 成功时的内存顺序
    failure_order     // 失败时的内存顺序
);
```

## 参数说明

### 1. 成功时的内存序
- 当 CAS 操作成功时（当前值等于期望值，成功设置为新值）
- 控制成功后，这个操作相对于其他内存操作的可见性和顺序

### 2. 失败时的内存序
- 当 CAS 操作失败时（当前值不等于期望值）
- 控制失败后，这个操作相对于其他内存操作的可见性和顺序
- 必须**不高于**成功时的内存顺序

## 为什么需要两个不同的内存序？

因为 CAS 操作成功和失败时的语义需求通常不同：

### 成功时
- 通常需要更严格的内存顺序
- 需要确保修改对其他线程立即可见
- 通常包含"写"语义，需要`release`或更强

### 失败时
- 通常只需要读取语义
- 不需要保证写的可见性
- 可以使用更宽松的内存顺序以提高性能

## 各种内存序详解

### 内存序级别（从弱到强）

```cpp
enum memory_order {
    memory_order_relaxed,     // 最宽松
    memory_order_consume,     // 不常用
    memory_order_acquire,     // 读操作屏障
    memory_order_release,     // 写操作屏障
    memory_order_acq_rel,     // 读写屏障
    memory_order_seq_cst      // 最严格（默认）
};
```

## 代码示例说明

### 示例1：不同的内存序组合

```cpp
#include <atomic>
#include <iostream>
#include <thread>

std::atomic<int> data(0);
std::atomic<bool> ready(false);
int shared_value = 0;

void writer_thread() {
    // 1. 先设置普通变量
    shared_value = 42;
    
    // 2. 设置标志
    data.store(100, std::memory_order_release);
    // release: 保证之前的写操作(shared_value=42)不会重排到store之后
    
    ready.store(true, std::memory_order_release);
}

void reader_thread() {
    int expected = 0;
    
    // 尝试从0改为200
    bool success = data.compare_exchange_weak(
        expected,  // 期望0
        200,       // 新值
        std::memory_order_acq_rel,  // 成功时：读写屏障
        std::memory_order_acquire   // 失败时：只读屏障
    );
    
    if (success) {
        std::cout << "CAS成功" << std::endl;
    } else {
        std::cout << "CAS失败，当前值: " << expected << std::endl;
    }
    
    // 等待标志
    while (!ready.load(std::memory_order_acquire)) {
        // 忙等待
    }
    
    // acquire保证：在读取ready为true之后，能看到writer_thread在store之前的所有写操作
    std::cout << "看到shared_value: " << shared_value << std::endl;
}
```

## 常用内存序组合

### 1. 默认组合（最强一致性）
```cpp
// 成功和失败都使用顺序一致性
atomic_var.compare_exchange_weak(
    expected, 
    desired,
    std::memory_order_seq_cst,  // 成功时：最强顺序
    std::memory_order_seq_cst   // 失败时：最强顺序
);
```
- 行为最直观
- 性能最差
- 适合调试或对内存顺序不了解时使用

### 2. 推荐组合（性能与正确性的平衡）
```cpp
// 成功时需要读写屏障，失败时只需要读屏障
atomic_var.compare_exchange_weak(
    expected,
    desired,
    std::memory_order_acq_rel,  // 成功：需要保证写的发布和后续读的获取
    std::memory_order_acquire   // 失败：只需要保证能读到最新值
);
```
- 这是**最常用**的组合
- 成功时：保证修改对其他线程可见（release），同时保证能看到其他线程的修改（acquire）
- 失败时：只需要保证能读到最新的值（acquire）

### 3. 宽松组合（最高性能）
```cpp
// 成功和失败都使用宽松顺序
atomic_var.compare_exchange_weak(
    expected,
    desired,
    std::memory_order_relaxed,  // 成功：不保证顺序
    std::memory_order_relaxed   // 失败：不保证顺序
);
```
- 性能最好
- 不保证操作的全局顺序
- 只适用于简单的标志位或计数器

## 实际应用场景

### 场景1：自旋锁实现
```cpp
class SpinLock {
    std::atomic<bool> locked{false};
    
public:
    void lock() {
        bool expected = false;
        // 尝试获取锁
        while (!locked.compare_exchange_weak(
            expected,           // 期望是false
            true,               // 设置为true
            std::memory_order_acquire,  // 成功时需要获取语义
            std::memory_order_relaxed   // 失败时不需要同步
        )) {
            expected = false;  // 重置期望值
        }
        // 成功获取锁，acquire保证能看到之前锁保护的临界区修改
    }
    
    void unlock() {
        locked.store(false, std::memory_order_release);
        // release保证临界区修改在锁释放前完成
    }
};
```

### 场景2：无锁栈的push操作
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
        Node* new_node = new Node{value, nullptr};
        
        do {
            new_node->next = head.load(std::memory_order_relaxed);
        } while (!head.compare_exchange_weak(
            new_node->next,     // 期望值
            new_node,           // 新值
            std::memory_order_release,  // 成功：需要发布新节点
            std::memory_order_relaxed   // 失败：不需要同步
        ));
        // 成功时，release保证新节点在head更新前完全构造
    }
    
    std::optional<T> pop() {
        Node* old_head = head.load(std::memory_order_relaxed);
        
        do {
            if (!old_head) return std::nullopt;
            
            Node* new_head = old_head->next;
        } while (!head.compare_exchange_weak(
            old_head,           // 期望值
            new_head,           // 新值
            std::memory_order_acquire,  // 成功：需要获取新head
            std::memory_order_relaxed   // 失败：不需要同步
        ));
        
        T value = old_head->data;
        delete old_head;
        return value;
    }
};
```

## 内存顺序规则总结

### 成功时的内存序
| 内存序 | 作用 | 适用场景 |
|--------|------|----------|
| `memory_order_seq_cst` | 全序，全局一致 | 需要最强保证的场景 |
| `memory_order_acq_rel` | 获取-释放 | 读写都需要同步（最常用） |
| `memory_order_release` | 释放语义 | 只需要保证写对其他线程可见 |
| `memory_order_acquire` | 获取语义 | 只需要保证能读到最新值 |
| `memory_order_relaxed` | 无同步 | 简单的计数器/标志位 |

### 失败时的内存序
| 内存序 | 作用 | 通常搭配 |
|--------|------|----------|
| `memory_order_seq_cst` | 与成功时相同 | 需要最强保证 |
| `memory_order_acquire` | 读取最新值 | 成功时用`acq_rel`或`release` |
| `memory_order_consume` | 数据依赖顺序 | 特殊场景，较少用 |
| `memory_order_relaxed` | 无同步 | 性能要求高的场景 |

## 组合有效性规则

```cpp
// 规则：失败内存序 不能强于 成功内存序
// 以下组合是有效的：

// 1. 相同级别
atomic_var.compare_exchange_weak(
    expected, desired,
    std::memory_order_seq_cst, std::memory_order_seq_cst
);
atomic_var.compare_exchange_weak(
    expected, desired,
    std::memory_order_acq_rel, std::memory_order_acquire
);
atomic_var.compare_exchange_weak(
    expected, desired,
    std::memory_order_release, std::memory_order_relaxed
);
atomic_var.compare_exchange_weak(
    expected, desired,
    std::memory_order_relaxed, std::memory_order_relaxed
);

// 2. 失败弱于成功
atomic_var.compare_exchange_weak(
    expected, desired,
    std::memory_order_seq_cst, std::memory_order_acquire
);
atomic_var.compare_exchange_weak(
    expected, desired,
    std::memory_order_acq_rel, std::memory_order_relaxed
);

// 以下组合是无效的（编译错误或未定义行为）：
atomic_var.compare_exchange_weak(
    expected, desired,
    std::memory_order_acquire,   // 成功：acquire
    std::memory_order_release    // 失败：release（不允许）
    // 错误：失败内存序不能包含release语义
);
```

## 实际使用建议

### 1. 默认选择
```cpp
// 如果不确定，使用这个组合
bool success = atomic_var.compare_exchange_weak(
    expected,
    desired,
    std::memory_order_acq_rel,  // 成功：获取-释放
    std::memory_order_acquire   // 失败：获取
);
```

### 2. 计数器场景
```cpp
// 简单的计数器，不需要严格同步
std::atomic<int> counter{0};
int expected = counter.load(std::memory_order_relaxed);
int desired = expected + 1;

// 使用宽松顺序
while (!counter.compare_exchange_weak(
    expected, desired,
    std::memory_order_relaxed,
    std::memory_order_relaxed
)) {
    desired = expected + 1;
}
```

### 3. 标志位场景
```cpp
// 设置标志，成功时需要发布，失败时只需要读取
std::atomic<bool> flag{false};
bool expected = false;

bool success = flag.compare_exchange_weak(
    expected,
    true,
    std::memory_order_release,  // 成功：发布修改
    std::memory_order_acquire   // 失败：获取最新值
);
```

## 注意事项

1. **错误使用示例**
```cpp
// 错误：失败时用release
// release只能用在包含store的操作上
bool success = atomic_var.compare_exchange_weak(
    expected, desired,
    std::memory_order_acquire,
    std::memory_order_release  // 错误！失败时不包含store
);

// 正确：失败时只能用relaxed、consume、acquire或seq_cst
bool success = atomic_var.compare_exchange_weak(
    expected, desired,
    std::memory_order_acq_rel,
    std::memory_order_acquire  // 正确
);
```

2. **x86架构的特殊性**
```cpp
// 在x86/x64架构上，由于TSO内存模型：
// - 所有store都有release语义
// - 所有load都有acquire语义
// 所以很多内存序是免费的

// 但在ARM/PowerPC等弱内存模型架构上：
// 内存序的指定至关重要
```

## 总结

- **第一个参数**：成功时的内存序，通常需要`acq_rel`或`release`
- **第二个参数**：失败时的内存序，通常只需要`acquire`或`relaxed`
- **常用组合**：`acq_rel` + `acquire` 适用于大多数场景
- **简单场景**：`relaxed` + `relaxed` 可用于性能敏感的简单操作
- **默认场景**：如果不确定，用默认的`seq_cst` + `seq_cst`

记住：内存序的目的是在正确性和性能之间找到平衡。过强的内存序会降低性能，过弱的内存序会导致数据竞争。