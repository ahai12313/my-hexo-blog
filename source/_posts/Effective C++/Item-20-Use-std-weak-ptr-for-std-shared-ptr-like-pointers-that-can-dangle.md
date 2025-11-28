---
title: 'Item 20: Use std::weak_ptr for std::shared_ptr like pointers that can dangle'
categories: Effective C++
date: 2025-11-29 00:18:04
tags:
priority: 20
---
# Item 20: 使用 std::weak_ptr 处理可能悬空的类似 std::shared_ptr 的指针

## 1. 核心概念

`std::weak_ptr` 是 C++ 标准库提供的一种弱引用智能指针，它与 `std::shared_ptr` 协同工作，但不参与对象的共享所有权管理。其核心特性是：

- **不增加引用计数**：不会阻止所指向对象的销毁
- **可检测悬空**：能够判断指向的对象是否已被释放
- **需与 `std::shared_ptr` 配合使用**：不能独立管理对象生命周期

## 2. 基本用法

### 2.1 创建与初始化

```cpp
// 从 shared_ptr 创建 weak_ptr
auto sp = std::make_shared<Widget>();
std::weak_ptr<Widget> wp(sp);  // 不增加引用计数

// 引用计数示例
std::cout << sp.use_count();  // 输出: 1
std::weak_ptr<Widget> wp2 = sp;
std::cout << sp.use_count();  // 输出: 1 (weak_ptr 不增加计数)
```

### 2.2 检测与访问

```cpp
// 方法1: 使用 lock() - 安全无异常
std::weak_ptr<Widget> wp;
if (auto sp = wp.lock()) {  // 如果未过期，sp 指向有效对象
    sp->doSomething();      // 安全使用
} else {
    // 处理对象已销毁的情况
}

// 方法2: 使用构造函数 - 可能抛出异常
try {
    std::shared_ptr<Widget> sp(wp);  // 如果 wp 过期，抛出 std::bad_weak_ptr
    sp->doSomething();
} catch (const std::bad_weak_ptr&) {
    // 处理异常
}

// 直接检测是否过期
if (wp.expired()) {
    // 对象已被销毁
}
```

## 3. 主要应用场景

### 3.1 缓存实现

```cpp
class WidgetCache {
private:
    mutable std::mutex mutex_;
    std::unordered_map<int, std::weak_ptr<const Widget>> cache_;

public:
    std::shared_ptr<const Widget> getWidget(int id) const {
        std::lock_guard<std::mutex> lock(mutex_);
        
        auto it = cache_.find(id);
        if (it != cache_.end()) {
            if (auto sp = it->second.lock()) {
                return sp;  // 缓存命中
            } else {
                cache_.erase(it);  // 清理过期缓存
            }
        }
        
        // 缓存未命中，创建新对象
        auto sp = createExpensiveWidget(id);
        cache_[id] = sp;
        return sp;
    }
};
```

**优势**：自动清理过期缓存项，避免内存泄漏，同时允许对象在不再使用时被及时释放。

### 3.2 观察者模式

```cpp
class Subject;  // 前向声明

class Observer : public std::enable_shared_from_this<Observer> {
public:
    virtual void update(const Subject& subject) = 0;
    virtual ~Observer() = default;
};

class Subject {
private:
    std::vector<std::weak_ptr<Observer>> observers_;
    mutable std::mutex mutex_;

public:
    void attach(std::weak_ptr<Observer> observer) {
        std::lock_guard<std::mutex> lock(mutex_);
        observers_.push_back(observer);
    }
    
    void notifyObservers() {
        std::lock_guard<std::mutex> lock(mutex_);
        auto it = observers_.begin();
        while (it != observers_.end()) {
            if (auto observer = it->lock()) {
                observer->update(*this);
                ++it;
            } else {
                it = observers_.erase(it);  // 移除已销毁的观察者
            }
        }
    }
};
```

**优势**：主题不控制观察者生命周期，避免意外延长对象寿命，自动清理无效观察者引用。

### 3.3 打破循环引用

```cpp
class Child;  // 前向声明

class Parent : public std::enable_shared_from_this<Parent> {
private:
    std::vector<std::shared_ptr<Child>> children_;
    // 不需要反向引用，避免循环
public:
    void addChild(std::shared_ptr<Child> child) {
        children_.push_back(child);
    }
};

class Child {
private:
    std::weak_ptr<Parent> parent_;  // 使用 weak_ptr 避免循环引用
    
public:
    void setParent(std::shared_ptr<Parent> parent) {
        parent_ = parent;
    }
    
    std::shared_ptr<Parent> getParent() const {
        return parent_.lock();  // 安全获取父节点
    }
};

// 使用示例
auto parent = std::make_shared<Parent>();
auto child = std::make_shared<Child>();
child->setParent(parent);
parent->addChild(child);  // 不会形成循环引用
```

**循环引用问题对比**：

| 指针类型 | 问题描述 | 解决方案 |
|---------|---------|---------|
| `std::shared_ptr` | 形成循环引用，内存泄漏 | ❌ 避免使用 |
| 原始指针 | 可能访问已销毁对象 | ❌ 不安全 |
| `std::weak_ptr` | 安全检测悬空，打破循环 | ✅ 推荐使用 |

## 4. 线程安全性考虑

### 4.1 竞态条件处理

```cpp
class ThreadSafeCache {
private:
    mutable std::shared_mutex mutex_;
    std::unordered_map<int, std::weak_ptr<Widget>> cache_;

public:
    std::shared_ptr<Widget> getOrCreate(int id) {
        // 读锁 - 快速路径
        {
            std::shared_lock read_lock(mutex_);
            if (auto it = cache_.find(id); it != cache_.end()) {
                if (auto sp = it->second.lock()) {
                    return sp;
                }
            }
        }
        
        // 写锁 - 慢速路径
        std::unique_lock write_lock(mutex_);
        
        // 双重检查，避免重复创建
        if (auto it = cache_.find(id); it != cache_.end()) {
            if (auto sp = it->second.lock()) {
                return sp;
            } else {
                cache_.erase(it);
            }
        }
        
        auto sp = std::make_shared<Widget>(id);
        cache_[id] = sp;
        return sp;
    }
};
```

**关键点**：`lock()` 操作是原子的，避免了检查与使用之间的竞态条件。

## 5. 性能特征

### 5.1 内存布局

```
std::weak_ptr 对象
┌─────────────┐    ┌─────────────────┐
│  控制块指针  │───▶│   控制块         │
└─────────────┘    ├─────────────────┤
                   │  强引用计数      │
                   │  弱引用计数      │
                   │  删除器          │
                   │  分配器          │
                   └─────────────────┘
```

### 5.2 性能特点

- **大小**：与 `std::shared_ptr` 相同（通常为 2 个指针）
- **操作成本**：涉及原子操作，但通常优化良好
- **控制块开销**：与 `std::shared_ptr` 共享同一控制块

## 6. 最佳实践

### 6.1 使用准则

```cpp
// ✅ 推荐做法
class RecommendedUsage {
public:
    void processWidget(std::weak_ptr<Widget> wp) {
        if (auto sp = wp.lock()) {  // 安全提升
            sp->process();          // 在 shared_ptr 生命周期内使用
        }
        // sp 离开作用域，不影响对象生命周期
    }
};

// ❌ 避免的做法
class BadUsage {
private:
    std::weak_ptr<Widget> wp_;
    
public:
    void unsafeProcess() {
        if (!wp_.expired()) {       // 竞态条件！
            // 此处对象可能已被销毁
            // auto sp = wp_.lock(); // 太晚了！
        }
    }
};
```

### 6.2 设计模式选择

| 场景 | 推荐指针类型 | 理由 |
|------|-------------|------|
| 独占所有权 | `std::unique_ptr` | 零开销，明确所有权 |
| 共享所有权 | `std::shared_ptr` | 自动生命周期管理 |
| 缓存引用 | `std::weak_ptr` | 避免阻止对象销毁 |
| 观察者引用 | `std::weak_ptr` | 避免意外延长生命周期 |
| 树形结构子到父 | 原始指针 | 父节点生命周期更长 |
| 非层次化循环引用 | `std::weak_ptr` | 打破循环，避免泄漏 |

## 7. 总结

`std::weak_ptr` 是 C++ 智能指针体系中的重要组成部分，专门用于处理需要观察对象但不控制其生命周期的场景。其主要价值体现在：

1. **资源管理**：在缓存等场景中避免不必要的对象保留
2. **架构安全**：在观察者模式中提供安全的对象观察机制  
3. **内存安全**：有效解决 `std::shared_ptr` 的循环引用问题
4. **线程安全**：提供原子性的对象访问检查机制

正确使用 `std::weak_ptr` 可以构建出既安全又高效的 C++ 应用程序，是现代 C++ 资源管理的核心工具之一。