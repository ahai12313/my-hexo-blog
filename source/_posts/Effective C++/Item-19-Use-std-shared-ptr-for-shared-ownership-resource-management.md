---
title: 'Item 19: Use std::shared_ptr for shared-ownership resource management'
categories: Effective C++
date: 2025-11-25 21:57:26
tags:
priority: 19
---
# Item 19：理解 std::shared_ptr 的工作原理

## 1. 概述

`std::shared_ptr` 是 C++11 引入的智能指针，实现了**共享所有权**语义。与 `std::unique_ptr` 的独占所有权不同，`std::shared_ptr` 允许多个指针共享同一个资源的所有权，通过引用计数机制自动管理资源生命周期。

### 1.1 核心特性对比
| 特性 | `std::unique_ptr` | `std::shared_ptr` |
|------|-------------------|-------------------|
| 所有权语义 | 独占所有权 | 共享所有权 |
| 拷贝语义 | 禁止拷贝，仅可移动 | 允许拷贝和移动 |
| 性能开销 | 接近原始指针 | 有额外开销（引用计数原子操作） |
| 内存占用 | 通常与原始指针相同 | 两倍指针大小（控制块指针+对象指针） |
| 适用场景 | 单一所有者资源 | 多个所有者共享资源 |

## 2. 基本用法和语法

### 2.1 创建 shared_ptr
```cpp
#include <memory>
#include <iostream>

class Resource {
public:
    Resource() { std::cout << "Resource constructed\n"; }
    ~Resource() { std::cout << "Resource destroyed\n"; }
    void use() { std::cout << "Using resource\n"; }
};

void basicUsage() {
    // 方法1：使用 std::make_shared（推荐）
    auto sp1 = std::make_shared<Resource>();
    
    // 方法2：从原始指针构造（不推荐，有风险）
    Resource* raw = new Resource();
    std::shared_ptr<Resource> sp2(raw);
    
    // 方法3：从 unique_ptr 转移所有权
    std::unique_ptr<Resource> up = std::make_unique<Resource>();
    std::shared_ptr<Resource> sp3 = std::move(up);
    
    // 使用资源
    sp1->use();
    sp2->use();
    sp3->use();
    
} // 自动调用析构函数
```

### 2.2 共享所有权演示
```cpp
void ownershipDemo() {
    auto sharedResource = std::make_shared<Resource>();
    
    std::cout << "引用计数: " << sharedResource.use_count() << "\n"; // 1
    
    {
        auto copy1 = sharedResource; // 拷贝构造，引用计数+1
        std::cout << "引用计数: " << sharedResource.use_count() << "\n"; // 2
        
        auto copy2 = sharedResource; // 引用计数+1
        std::cout << "引用计数: " << sharedResource.use_count() << "\n"; // 3
    } // copy1, copy2 析构，引用计数-2
    
    std::cout << "引用计数: " << sharedResource.use_count() << "\n"; // 1
} // sharedResource 析构，引用计数为0，资源被销毁
```

## 3. 内部实现机制

### 3.1 控制块结构
`std::shared_ptr` 的核心是**控制块**（Control Block），包含以下信息：

```cpp
// 概念上的控制块结构
struct ControlBlock {
    std::atomic<long> use_count;    // 强引用计数
    std::atomic<long> weak_count;   // 弱引用计数
    void(*deleter)(void*);          // 自定义删除器
    void* allocated_memory;         // 分配的内存块（用于make_shared优化）
};
```

### 3.2 内存布局示意图
```
std::shared_ptr<Widget>
┌─────────────────┐    ┌─────────────┐
│  对象指针 (Widget*) │───→│  Widget对象  │
├─────────────────┤    └─────────────┘
│控制块指针 (void*)  │───→┌─────────────────┐
└─────────────────┘    │    控制块        │
                       ├─────────────────┤
                       │  use_count: 2   │
                       │  weak_count: 0  │
                       │  deleter: delete│
                       │  ...            │
                       └─────────────────┘
```

## 4. 控制块创建规则

### 4.1 控制块创建时机
```cpp
void controlBlockCreation() {
    // 情况1：std::make_shared 总是创建控制块
    auto sp1 = std::make_shared<int>(42);  // ✅ 创建控制块
    
    // 情况2：从原始指针构造时创建控制块
    int* raw = new int(100);
    std::shared_ptr<int> sp2(raw);         // ✅ 创建控制块
    
    // 情况3：从 unique_ptr 转移时创建控制块
    std::unique_ptr<int> up(new int(200));
    std::shared_ptr<int> sp3 = std::move(up); // ✅ 创建控制块
    
    // 情况4：从其他 shared_ptr 拷贝时不创建控制块
    std::shared_ptr<int> sp4 = sp1;        // ❌ 不创建，共享控制块
    
    // 情况5：从 weak_ptr 升级时不创建控制块
    std::weak_ptr<int> wp = sp1;
    if (auto sp5 = wp.lock()) {            // ❌ 不创建，共享控制块
        // 使用 sp5
    }
}
```

### 4.2 危险的多重控制块
```cpp
void dangerousMultipleControlBlocks() {
    int* raw = new int(42);  // 🚨 原始指针
    
    // 错误：创建两个独立的控制块！
    std::shared_ptr<int> sp1(raw, int* p { 
        std::cout << "删除器1调用\n"; delete p; 
    });
    
    std::shared_ptr<int> sp2(raw, int* p { 
        std::cout << "删除器2调用\n"; delete p; 
    });  // 🚨 未定义行为！
    
    // 运行结果：资源被删除两次！
    // 输出可能是：
    // 删除器1调用
    // 删除器2调用
    // 然后程序崩溃
}

void correctApproach() {
    // 正确做法1：使用 std::make_shared
    auto sp1 = std::make_shared<int>(42);
    auto sp2 = sp1;  // ✅ 共享控制块
    
    // 正确做法2：避免原始指针变量
    std::shared_ptr<int> sp3(new int(100), int* p {
        std::cout << "安全删除\n"; delete p;
    });
    auto sp4 = sp3;  // ✅ 共享控制块
}
```

## 5. 自定义删除器

### 5.1 灵活的自定义删除器支持
```cpp
#include <fstream>
#include <vector>

class FileResource {
public:
    void close() { std::cout << "文件关闭\n"; }
};

void customDeleterExamples() {
    // 1. Lambda 删除器
    auto fileDeleter = std::FILE* file {
        if (file) {
            std::cout << "关闭文件\n";
            std::fclose(file);
        }
    };
    std::shared_ptr<std::FILE> file(std::fopen("data.txt", "r"), fileDeleter);
    
    // 2. 函数对象删除器
    struct ArrayDeleter {
        void operator()(int* arr) const {
            std::cout << "删除数组\n";
            delete[] arr;
        }
    };
    std::shared_ptr<int> arr(new int[100], ArrayDeleter{});
    
    // 3. 标准函数删除器
    std::shared_ptr<std::vector<int>> vec(
        new std::vector<int>(100),
        auto* ptr { 
            std::cout << "自定义vector删除器\n"; 
            delete ptr; 
        }
    );
}
```

### 5.2 与 std::unique_ptr 的删除器差异
```cpp
void deleterComparison() {
    auto logger = int* p { 
        std::cout << "删除: " << *p << "\n"; 
        delete p; 
    };
    
    // unique_ptr: 删除器是类型的一部分
    std::unique_ptr<int, decltype(logger)> up1(new int(42), logger);
    // std::unique_ptr<int, decltype(logger)> up2 = up1;  // ❌ 不能拷贝
    
    // shared_ptr: 删除器不是类型的一部分
    std::shared_ptr<int> sp1(new int(42), logger);
    std::shared_ptr<int> sp2 = sp1;  // ✅ 可以拷贝
    std::shared_ptr<int> sp3(new int(100), logger);
    
    // 可以放入同一容器
    std::vector<std::shared_ptr<int>> pointers;
    pointers.push_back(sp1);
    pointers.push_back(sp2);
    pointers.push_back(sp3);  // ✅ 所有类型相同
}
```

## 6. std::enable_shared_from_this

### 6.1 问题场景：错误的 this 指针用法
```cpp
class ProblematicWidget {
    std::vector<std::shared_ptr<ProblematicWidget>> processedWidgets;
    
public:
    void process() {
        // 🚨 危险：从 this 创建新的 shared_ptr
        processedWidgets.emplace_back(this);  // 可能创建新控制块！
    }
};

void demonstrateProblem() {
    auto widget = std::make_shared<ProblematicWidget>();
    widget->process();  // 🚨 可能创建重复的控制块！
    
    // 如果外部已经有 shared_ptr 指向这个 widget，
    // 就会有两个控制块，导致双重删除
}
```

### 6.2 正确解决方案：enable_shared_from_this
```cpp
class SafeWidget : public std::enable_shared_from_this<SafeWidget> {
    std::vector<std::shared_ptr<SafeWidget>> processedWidgets;
    
public:
    // 工厂方法确保安全创建
    static std::shared_ptr<SafeWidget> create() {
        return std::make_shared<SafeWidget>();
    }
    
    void process() {
        // ✅ 安全：使用 shared_from_this()
        auto self = shared_from_this();  // 共享现有控制块
        processedWidgets.emplace_back(self);
        std::cout << "安全处理，引用计数: " << self.use_count() << "\n";
    }
    
private:
    SafeWidget() = default;  // 强制使用工厂方法
};

void safeUsage() {
    auto widget = SafeWidget::create();  // 必须通过 shared_ptr 创建
    widget->process();  // ✅ 安全：共享控制块
    
    // 错误用法：直接栈上创建
    // SafeWidget badWidget;  // ❌ 编译错误：构造函数私有
    // badWidget.process();   // ❌ 运行时错误：没有控制块
}
```

## 7. 性能分析和优化

### 7.1 性能开销分析
```cpp
#include <chrono>

void performanceBenchmark() {
    const int iterations = 1000000;
    
    // 原始指针性能基准
    auto start1 = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < iterations; ++i) {
        int* raw = new int(i);
        delete raw;
    }
    auto end1 = std::chrono::high_resolution_clock::now();
    
    // shared_ptr 性能测试
    auto start2 = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < iterations; ++i) {
        auto sp = std::make_shared<int>(i);
    }
    auto end2 = std::chrono::high_resolution_clock::now();
    
    auto duration1 = std::chrono::duration_cast<std::chrono::microseconds>(end1 - start1);
    auto duration2 = std::chrono::duration_cast<std::chrono::microseconds>(end2 - start2);
    
    std::cout << "原始指针: " << duration1.count() << "μs\n";
    std::cout << "shared_ptr: " << duration2.count() << "μs\n";
    std::cout << "开销: " << (duration2.count() - duration1.count()) * 100.0 / duration1.count() << "%\n";
}
```

### 7.2 优化技巧
```cpp
class PerformanceOptimizations {
public:
    // 技巧1：优先使用 std::make_shared
    void optimization1() {
        // 好：一次内存分配（对象+控制块）
        auto optimal = std::make_shared<std::vector<int>>(1000, 42);
        
        // 不好：两次内存分配
        std::shared_ptr<std::vector<int>> suboptimal(new std::vector<int>(1000, 42));
    }
    
    // 技巧2：使用移动语义避免原子操作
    void optimization2() {
        auto createHeavyResource =  {
            return std::make_shared<std::array<int, 1000>>();
        };
        
        // 移动构造：无原子操作开销
        auto resource1 = createHeavyResource();
        auto resource2 = std::move(resource1);  // ✅ 高效
        
        // 拷贝构造：有原子操作开销  
        auto resource3 = resource2;  // ❌ 有开销
    }
    
    // 技巧3：避免不必要的 shared_ptr 拷贝
    void optimization3() {
        auto heavyObject = std::make_shared<std::string>(1000000, 'x');
        
        // 不好：不必要的拷贝
        processResource(heavyObject);  // 如果processResource不需要所有权，传递引用
        
        // 好：传递引用
        processResourceRef(*heavyObject);
    }
    
private:
    void processResource(std::shared_ptr<std::string> resource) {
        // 需要所有权时使用
    }
    
    void processResourceRef(const std::string& resource) {
        // 只需要访问时使用引用
    }
};
```

## 8. 线程安全考虑

### 8.1 引用计数的线程安全
```cpp
#include <thread>
#include <vector>

class ThreadSafety {
public:
    void referenceCountSafety() {
        auto sharedData = std::make_shared<std::vector<int>>();
        
        std::vector<std::thread> threads;
        for (int i = 0; i < 10; ++i) {
            threads.emplace_back( {  // 拷贝到线程
                // ✅ 引用计数操作是原子的
                // 每个线程安全地持有 shared_ptr
                data->push_back(42);
            });
        }
        
        for (auto& t : threads) {
            t.join();
        }
    }
};
```

### 8.2 数据竞态警告
```cpp
class DataRaceExample {
    std::shared_ptr<std::vector<int>> data_ = std::make_shared<std::vector<int>>();
    
public:
    void demonstrateDataRace() {
        std::thread writer( {
            for (int i = 0; i < 1000; ++i) {
                data_->push_back(i);  // 🚨 可能数据竞态！
            }
        });
        
        std::thread reader( {
            for (int i = 0; i < 1000; ++i) {
                if (!data_->empty()) {
                    int value = data_->back();  // 🚨 可能数据竞态！
                }
            }
        });
        
        writer.join();
        reader.join();
        
        // shared_ptr 保证控制块线程安全，但不保证指向的数据线程安全！
    }
    
    void threadSafeVersion() {
        auto localData = std::make_shared<std::vector<int>>();
        
        // 每个线程使用自己的数据副本
        std::thread worker( mutable {
            // 线程内操作，无竞态
            for (int i = 0; i < 1000; ++i) {
                data->push_back(i);
            }
        });
        
        worker.join();
        
        // 合并结果到主数据（需要同步）
        std::lock_guard<std::mutex> lock(mutex_);
        data_->insert(data_->end(), localData->begin(), localData->end());
    }
    
private:
    std::mutex mutex_;
};
```

## 9. 使用场景和最佳实践

### 9.1 适用场景
```cpp
class AppropriateUseCases {
public:
    // 场景1：缓存系统
    class Cache {
        std::unordered_map<std::string, std::shared_ptr<const std::string>> cache_;
        
    public:
        std::shared_ptr<const std::string> get(const std::string& key) {
            auto it = cache_.find(key);
            if (it != cache_.end()) {
                return it->second;  // 多个客户端可以共享缓存项
            }
            return nullptr;
        }
    };
    
    // 场景2：观察者模式
    class Observer : public std::enable_shared_from_this<Observer> {
    public:
        void subscribeTo(std::shared_ptr<Observable> observable) {
            observable->addObserver(shared_from_this());
        }
    };
    
    // 场景3：共享配置数据
    class Application {
        std::shared_ptr<const Config> config_ = loadConfig();
        
    public:
        std::shared_ptr<const Config> getConfig() const {
            return config_;  // 所有组件共享同一配置
        }
    };
};
```

### 9.2 不适用场景
```cpp
class InappropriateUseCases {
public:
    void avoidWhenNotNeeded() {
        // 情况1：独占所有权时使用 unique_ptr
        std::unique_ptr<Resource> exclusive = std::make_unique<Resource>();
        
        // 情况2：局部对象使用栈分配
        Resource stackResource;  // 更高效
        
        // 情况3：数组使用 vector 或 unique_ptr
        std::vector<int> vec(100);  // 而不是 shared_ptr<int[]>
        std::unique_ptr<int[]> arr(new int[100]);
    }
    
    void circularReferenceProblem() {
        struct Node {
            std::shared_ptr<Node> next;
            // std::shared_ptr<Node> prev;  // 🚨 循环引用！
        };
        
        auto node1 = std::make_shared<Node>();
        auto node2 = std::make_shared<Node>();
        node1->next = node2;
        node2->next = node1;  // 循环引用，内存泄漏！
        
        // 解决方案：使用 weak_ptr 打破循环
        struct SafeNode {
            std::shared_ptr<SafeNode> next;
            std::weak_ptr<SafeNode> prev;  // ✅ 使用 weak_ptr
        };
    }
};
```

## 10. 高级主题和陷阱

### 10.1 控制块生命周期
```cpp
void controlBlockLifetime() {
    // make_shared 优化：对象和控制块一起分配
    auto sp1 = std::make_shared<int>(42);
    
    // 即使 shared_ptr 全部销毁，控制块可能仍然存在（如果有 weak_ptr）
    std::weak_ptr<int> wp = sp1;
    
    sp1.reset();  // 对象被销毁
    
    if (wp.expired()) {
        std::cout << "对象已销毁\n";
    }
    // 控制块仍然存在，直到最后一个 weak_ptr 销毁
}
```

### 10.2 自定义分配器
```cpp
#include <memory_resource>

void customAllocatorExample() {
    // 使用自定义内存池
    std::pmr::unsynchronized_pool_resource pool;
    
    auto custom_deleter = int* p { 
        std::cout << "自定义删除器\n"; 
        delete p; 
    };
    
    // 使用自定义分配器的 shared_ptr
    std::shared_ptr<int> sp(
        new int(42), 
        custom_deleter,
        std::pmr::polymorphic_allocator<int>(&pool)
    );
}
```

## 11. 最佳实践总结

### 11.1 决策流程图
```
需要智能指针吗？
├── 否 → 使用栈对象或容器
└── 是 → 
    ├── 单一所有者？ → std::unique_ptr
    ├── 多个所有者共享？ → std::shared_ptr
    │   ├── 需要观察而不拥有？ → std::weak_ptr
    │   └── 注意循环引用问题
    └── 仅观察不拥有？ → std::weak_ptr
```

### 11.2 代码审查清单
- [ ] 是否避免了从原始指针创建多个 `shared_ptr`？
- [ ] 是否优先使用 `std::make_shared`？
- [ ] 类需要从成员函数返回 `shared_ptr` 时，是否继承了 `enable_shared_from_this`？
- [ ] 是否考虑了循环引用的可能性？
- [ ] 是否在适当场景使用 `unique_ptr` 替代？
- [ ] 是否理解了数据访问的线程安全需求？

### 11.3 性能优化检查表
1. **内存分配**：优先使用 `std::make_shared` 合并分配
2. **拷贝开销**：在可能的情况下使用移动语义
3. **原子操作**：避免不必要的 `shared_ptr` 拷贝
4. **控制块**：避免意外的多重控制块创建
5. **生命周期**：及时释放不再需要的 `shared_ptr`

## 12. 总结

`std::shared_ptr` 是 C++ 中强大的共享所有权工具，但需要理解其成本和正确使用方法：

**核心要点**：
- 使用引用计数实现共享所有权语义
- 控制块管理确保正确的资源生命周期
- 原子操作保证线程安全的引用计数
- 灵活的自定义删除器支持

**性能考虑**：
- 比 `unique_ptr` 更大的内存开销（两倍指针大小）
- 原子操作带来的性能成本
- 控制块管理的复杂性

**正确使用准则**：
- 优先使用 `std::make_shared` 创建
- 避免从原始指针创建多个 `shared_ptr`
- 需要从成员函数创建 `shared_ptr` 时使用 `enable_shared_from_this`
- 在适当场景考虑更轻量的 `unique_ptr`
- 注意循环引用问题，必要时使用 `weak_ptr`

通过遵循这些准则，你可以安全高效地使用 `std::shared_ptr` 来管理需要共享所有权的资源。