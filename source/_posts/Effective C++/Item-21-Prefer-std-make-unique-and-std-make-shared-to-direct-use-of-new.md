---
title: 'Item 21: Prefer std::make_unique and std::make_shared to direct use of new'
categories: Effective C++
date: 2025-11-29 01:09:06
tags:
priority: 21
---
# Item 21: 优先使用 make 函数而非直接使用 new

## 1. 概述

### 1.1 make 函数家族
C++标准库提供了三个主要的make函数，用于创建智能指针：

| 函数 | 引入标准 | 描述 |
|------|----------|------|
| `std::make_unique` | C++14 | 创建`std::unique_ptr` |
| `std::make_shared` | C++11 | 创建`std::shared_ptr` |
| `std::allocate_shared` | C++11 | 使用自定义分配器创建`std::shared_ptr` |

### 1.2 基本语法
```cpp
// 创建唯一所有权指针
auto up = std::make_unique<Widget>(arg1, arg2);

// 创建共享所有权指针  
auto sp = std::make_shared<Widget>(arg1, arg2);

// 使用自定义分配器
auto alloc = MyAllocator<Widget>();
auto asp = std::allocate_shared<Widget>(alloc, arg1, arg2);
```

## 2. 使用 make 函数的优势

### 2.1 避免代码重复

**问题示例**：
```cpp
// ❌ 类型名称重复
std::unique_ptr<Widget> up1(new Widget(args));
std::shared_ptr<Widget> sp1(new Widget(args));

// ✅ 使用 make 函数 - 无重复
auto up2 = std::make_unique<Widget>(args);
auto sp2 = std::make_shared<Widget>(args);
```

**优势分析**：
- **编译时优化**：减少代码体积，加快编译速度
- **维护性**：修改类型时只需改动一处
- **一致性**：避免因复制粘贴导致的错误

### 2.2 异常安全保证

#### 2.2.1 问题场景
```cpp
void processWidget(std::shared_ptr<Widget> spw, int priority);

// ❌ 潜在的资源泄漏
processWidget(std::shared_ptr<Widget>(new Widget), computePriority());
```

**可能的执行顺序**：
1. `new Widget` - 分配内存
2. `computePriority()` - **可能抛出异常**
3. 构造`std::shared_ptr` - 如果步骤2异常，Widget内存泄漏

#### 2.2.2 安全解决方案
```cpp
// ✅ 使用 make_shared - 原子性操作
processWidget(std::make_shared<Widget>(), computePriority());

// ✅ 手动异常安全版本
std::shared_ptr<Widget> spw(new Widget);  // 单独语句
processWidget(std::move(spw), computePriority());  // 移动语义优化
```

### 2.3 性能优化（make_shared特有）

#### 2.3.1 内存分配对比

**直接使用new**：
```cpp
std::shared_ptr<Widget> spw(new Widget);
// 内存分配次数：2次
// 1. 分配Widget对象内存
// 2. 分配控制块内存
```

**使用make_shared**：
```cpp
auto spw = std::make_shared<Widget>();
// 内存分配次数：1次  
// 单次分配：对象内存 + 控制块内存（连续）
```

#### 2.3.2 性能优势分析

| 指标 | 直接使用new | make_shared | 优势 |
|------|------------|-------------|------|
| 分配次数 | 2次 | 1次 | 减少50% |
| 内存局部性 | 差（对象和控制块分离） | 好（连续内存） | 缓存友好 |
| 总内存占用 | 控制块+对象+额外开销 | 控制块+对象 | 减少碎片 |

## 3. make函数的局限性

### 3.1 自定义删除器不支持

```cpp
// 自定义删除器示例
auto fileDeleter = FILE* fp { 
    if (fp) fclose(fp); 
};

// ✅ 使用new支持自定义删除器
std::unique_ptr<FILE, decltype(fileDeleter)> 
    filePtr(fopen("data.txt", "r"), fileDeleter);

std::shared_ptr<FILE> sharedFile(fopen("data.txt", "r"), fileDeleter);

// ❌ make函数不支持自定义删除器
// auto sp = std::make_shared<FILE>(fopen("data.txt", "r"), fileDeleter); // 错误
```

### 3.2 初始化列表问题

#### 3.2.1 问题描述
```cpp
// ❌ 歧义：10个值为20的元素，还是包含10和20两个元素？
auto vec1 = std::make_shared<std::vector<int>>(10, 20);
// 实际结果：10个元素，每个值为20（使用圆括号转发）

// ✅ 期望的初始化列表方式
std::vector<int> desired{10, 20};  // 包含10,20两个元素
```

#### 3.2.2 解决方案
```cpp
// ✅ 方法1：使用auto推导initializer_list
auto initList = {10, 20};  // std::initializer_list<int>
auto vec2 = std::make_shared<std::vector<int>>(initList);

// ✅ 方法2：直接使用new
auto vec3 = std::shared_ptr<std::vector<int>>(
    new std::vector<int>{1, 2, 3, 4, 5});
```

### 3.3 类特定内存管理

**适用场景**：类重载了`operator new`/`operator delete`

```cpp
class PoolAllocated {
public:
    static void* operator new(size_t size) {
        return memoryPool.allocate(size);
    }
    static void operator delete(void* ptr) noexcept {
        memoryPool.deallocate(ptr);
    }
private:
    static MemoryPool memoryPool;
};

// ❌ 不适合make_shared（需要连续分配对象+控制块）
// auto obj = std::make_shared<PoolAllocated>();

// ✅ 直接使用new
auto obj = std::shared_ptr<PoolAllocated>(new PoolAllocated);
```

### 3.4 大对象与弱指针生命周期问题

#### 3.4.1 内存释放时机差异

**make_shared场景**：
```cpp
class LargeObject { /* 占用大量内存 */ };

{
    auto bigObj = std::make_shared<LargeObject>();
    std::weak_ptr<LargeObject> weakRef = bigObj;
    
    bigObj.reset();  // 对象析构，但内存不能立即释放
    // 内存要等到weakRef销毁才能释放（对象+控制块连续）
}
```

**直接使用new场景**：
```cpp
{
    std::shared_ptr<LargeObject> bigObj(new LargeObject);
    std::weak_ptr<LargeObject> weakRef = bigObj;
    
    bigObj.reset();  // 对象立即析构，对象内存立即释放
    // 只有控制块内存等待weakRef销毁
}
```

#### 3.4.2 适用情况对比

| 场景 | 推荐方法 | 理由 |
|------|----------|------|
| 小对象，短期weak_ptr | make_shared | 性能优势明显 |
| 大对象，长期weak_ptr | 直接使用new | 及时释放对象内存 |
| 内存敏感系统 | 直接使用new | 更细粒度的内存控制 |

## 4. 最佳实践指南

### 4.1 默认选择策略

```cpp
// ✅ 默认情况：优先使用make函数
auto defaultCase1 = std::make_unique<MyClass>(arg1, arg2);
auto defaultCase2 = std::make_shared<MyClass>(arg1, arg2);

// ✅ 性能关键：使用make_shared
auto perfCritical = std::make_shared<PerformanceSensitiveClass>();
```

### 4.2 特殊情况处理

```cpp
// 1. 自定义删除器
auto filePtr = std::shared_ptr<FILE>(
    fopen("data.bin", "rb"), 
    FILE* fp { if (fp) fclose(fp); }
);

// 2. 大括号初始化
auto config = std::make_shared<std::unordered_map<std::string, int>>(
    std::initializer_list<std::pair<const std::string, int>>{
        {"timeout", 30}, {"retries", 3}
    }
);

// 3. 类特定内存管理
auto poolObj = std::shared_ptr<PoolAllocated>(new PoolAllocated);

// 4. 大对象+长期weak_ptr
auto largeObj = std::shared_ptr<LargeData>(new LargeData);
std::weak_ptr<LargeData> longTermRef = largeObj;
```

### 4.3 异常安全模式

```cpp
class ResourceManager {
public:
    void safeOperation() {
        // 模式1：分离创建和使用（基础版）
        auto resource = std::shared_ptr<Resource>(new Resource, customDeleter);
        useResource(resource);
        
        // 模式2：分离创建和使用（优化版）
        auto tempResource = std::shared_ptr<Resource>(new Resource, customDeleter);
        useResourceOptimized(std::move(tempResource));  // 移动避免拷贝
    }
    
private:
    void useResource(std::shared_ptr<Resource> res) {
        // 接收左值，可能涉及引用计数操作
    }
    
    void useResourceOptimized(std::shared_ptr<Resource>&& res) {
        // 接收右值，无引用计数操作
    }
};
```

## 5. 实现细节与技巧

### 5.1 C++11中的make_unique实现

```cpp
// 基础版本（C++11兼容）
template<typename T, typename... Args>
std::unique_ptr<T> make_unique(Args&&... args) {
    return std::unique_ptr<T>(new T(std::forward<Args>(args)...));
}

// 数组特化版本
template<typename T>
std::unique_ptr<T> make_unique(size_t size) {
    return std::unique_ptr<T>(new typename std::remove_extent<T>::type);
}
```

### 5.2 性能测试示例

```cpp
#include <chrono>
#include <memory>
#include <vector>

void performanceComparison() {
    constexpr size_t iterations = 1000000;
    
    // 测试make_shared性能
    auto start1 = std::chrono::high_resolution_clock::now();
    for (size_t i = 0; i < iterations; ++i) {
        auto sp = std::make_shared<std::vector<int>>(100, 42);
    }
    auto end1 = std::chrono::high_resolution_clock::now();
    
    // 测试直接使用new性能  
    auto start2 = std::chrono::high_resolution_clock::now();
    for (size_t i = 0; i < iterations; ++i) {
        auto sp = std::shared_ptr<std::vector<int>>(
            new std::vector<int>(100, 42));
    }
    auto end2 = std::chrono::high_resolution_clock::now();
    
    auto duration1 = std::chrono::duration_cast<std::chrono::milliseconds>(
        end1 - start1);
    auto duration2 = std::chrono::duration_cast<std::chrono::milliseconds>(
        end2 - start2);
    
    std::cout << "make_shared: " << duration1.count() << "ms\n";
    std::cout << "direct new: " << duration2.count() << "ms\n";
}
```

## 6. 总结

### 6.1 决策流程图

```
是否需要智能指针？
    ↓
是 → 是否需要自定义删除器？
        ↓
    是 → 使用 new + 智能指针构造函数
        ↓  
    否 → 是否需要大括号初始化？
        ↓
        是 → 使用 new 或 auto + initializer_list
        ↓
        否 → 对象是否有自定义operator new/delete？
            ↓
            是 → 使用 new
            ↓  
            否 → 是否大对象 + 长期weak_ptr？
                ↓
                是 → 使用 new
                ↓
                否 → 使用对应的make函数 ✅
```

### 6.2 核心原则

1. **默认优先**：在大多数情况下优先使用make函数
2. **了解局限**：清楚知道make函数不适用的场景  
3. **性能权衡**：根据具体场景在便利性和性能之间权衡
4. **异常安全**：确保资源管理代码的异常安全性

### 63. 关键要点记忆

- ✅ **使用make函数**：代码简洁、异常安全、性能优化
- ⚠️ **避免make函数**：自定义删除器、大括号初始化、特殊内存管理
- 🔧 **手动优化**：大对象+长期weak_ptr场景考虑直接使用new

通过理解这些原则和场景，可以在实际开发中做出最合适的选择，写出既安全又高效的C++代码。