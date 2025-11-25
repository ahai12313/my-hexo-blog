---
title: supplement 17 循环引用问题详解
categories: Supplement C++
date: 2025-11-25 23:07:31
tags:
priority: 17
---
# 循环引用问题详解

这是一个关于智能指针内存泄漏的经典问题。让我详细解释循环引用的场景、原理和解决方案。

## 循环引用场景分析

### 问题代码：双向链表节点
```cpp
struct Node {
    std::shared_ptr<Node> next;
    std::shared_ptr<Node> prev;  // 🚨 循环引用！
    
    ~Node() {
        std::cout << "Node destroyed\n";  // 永远不会执行！
    }
};

void circularReferenceProblem() {
    auto node1 = std::make_shared<Node>();
    auto node2 = std::make_shared<Node>();
    
    node1->next = node2;  // node2引用计数+1 = 2
    node2->prev = node1;  // node1引用计数+1 = 2
    
    // 函数结束时：
    // node2局部变量销毁，引用计数-1 = 1（因为node1->next还引用着）
    // node1局部变量销毁，引用计数-1 = 1（因为node2->prev还引用着）
    // 结果：两个节点都无法被销毁！
}
```

## 引用计数变化演示

```cpp
void demonstrateReferenceCounting() {
    std::cout << "=== 创建节点 ===\n";
    auto node1 = std::make_shared<Node>();  // node1引用计数 = 1
    auto node2 = std::make_shared<Node>();  // node2引用计数 = 1
    
    std::cout << "=== 建立双向链接 ===\n";
    node1->next = node2;  // node2引用计数 = 2
    node2->prev = node1;  // node1引用计数 = 2
    
    std::cout << "=== 局部变量离开作用域 ===\n";
    // node2销毁：引用计数-1 = 1（node1->next仍然引用）
    // node1销毁：引用计数-1 = 1（node2->prev仍然引用）
    
    std::cout << "=== 内存泄漏发生！ ===\n";
    // 两个节点的引用计数都保持为1，永远不会变为0
    // 析构函数永远不会被调用
}
```

## 内存泄漏验证

```cpp
#include <iostream>
#include <memory>
#include <vector>

class MemoryTracker {
    static int aliveCount;
    int id;
    
public:
    MemoryTracker() : id(++aliveCount) {
        std::cout << "MemoryTracker " << id << " 构造\n";
    }
    
    ~MemoryTracker() {
        std::cout << "MemoryTracker " << id << " 析构\n";
        --aliveCount;
    }
    
    static int getAliveCount() { return aliveCount; }
};

int MemoryTracker::aliveCount = 0;

void testMemoryLeak() {
    std::cout << "初始存活对象: " << MemoryTracker::getAliveCount() << "\n";
    
    {
        struct LeakyNode {
            std::shared_ptr<LeakyNode> next;
            std::shared_ptr<LeakyNode> prev;
            MemoryTracker tracker;
        };
        
        auto node1 = std::make_shared<LeakyNode>();
        auto node2 = std::make_shared<LeakyNode>();
        
        node1->next = node2;
        node2->prev = node1;  // 循环引用！
        
        std::cout << "循环引用建立后存活对象: " << MemoryTracker::getAliveCount() << "\n";
    }  // node1和node2离开作用域，但内存泄漏！
    
    std::cout << "离开作用域后存活对象: " << MemoryTracker::getAliveCount() << "\n";
    // 输出不会是0，证明内存泄漏！
}
```

## 解决方案：使用 `weak_ptr`

### 方案1：双向链表使用 weak_ptr
```cpp
struct SafeNode {
    std::shared_ptr<SafeNode> next;
    std::weak_ptr<SafeNode> prev;  // ✅ 使用 weak_ptr 打破循环
    
    std::string data;
    
    ~SafeNode() {
        std::cout << "SafeNode 正常析构\n";
    }
    
    // 获取前一个节点（如果还存在）
    std::shared_ptr<SafeNode> getPrevious() {
        return prev.lock();  // 尝试提升为 shared_ptr
    }
};

void safeLinkedList() {
    auto node1 = std::make_shared<SafeNode>();
    auto node2 = std::make_shared<SafeNode>();
    
    node1->next = node2;     // node2引用计数+1 = 2
    node2->prev = node1;     // node1引用计数不变（weak_ptr不增加计数）= 1
    
    // 函数结束时：
    // node2销毁：引用计数-1 = 1（node1->next仍然引用）
    // node1销毁：引用计数-1 = 0 → 调用析构函数
    // node1析构时，next成员销毁 → node2引用计数-1 = 0 → 调用析构函数
}
```

### 方案2：树形结构（父节点使用 weak_ptr）
```cpp
class TreeNode : public std::enable_shared_from_this<TreeNode> {
    std::weak_ptr<TreeNode> parent_;  // 父节点使用weak_ptr
    std::vector<std::shared_ptr<TreeNode>> children_;  // 子节点使用shared_ptr
    
public:
    std::string name;
    
    TreeNode(const std::string& nodeName) : name(nodeName) {}
    
    ~TreeNode() {
        std::cout << "TreeNode " << name << " 析构\n";
    }
    
    void addChild(std::shared_ptr<TreeNode> child) {
        child->parent_ = shared_from_this();  // 设置父节点（weak_ptr）
        children_.push_back(child);
    }
    
    std::shared_ptr<TreeNode> getParent() {
        return parent_.lock();
    }
    
    void printTree(int depth = 0) {
        std::string indent(depth * 2, ' ');
        std::cout << indent << name << "\n";
        for (auto& child : children_) {
            child->printTree(depth + 1);
        }
    }
};

void testTreeStructure() {
    auto root = std::make_shared<TreeNode>("Root");
    auto child1 = std::make_shared<TreeNode>("Child1");
    auto child2 = std::make_shared<TreeNode>("Child2");
    
    root->addChild(child1);
    root->addChild(child2);
    
    root->printTree();
    
    // 函数结束时所有节点正常销毁，无内存泄漏
}
```

## 更多循环引用场景

### 场景1：观察者模式中的循环引用
```cpp
// 错误的实现：相互持有shared_ptr
class ChatRoom;  // 前向声明

class User {
    std::shared_ptr<ChatRoom> room_;  // 🚨 危险！
public:
    void joinRoom(std::shared_ptr<ChatRoom> room) {
        room_ = room;
    }
};

class ChatRoom {
    std::vector<std::shared_ptr<User>> users_;  // 🚨 危险！
public:
    void addUser(std::shared_ptr<User> user) {
        users_.push_back(user);
    }
};

// 正确的实现：使用weak_ptr
class SafeUser : public std::enable_shared_from_this<SafeUser> {
    std::weak_ptr<ChatRoom> room_;  // ✅ 安全
public:
    void joinRoom(std::shared_ptr<ChatRoom> room) {
        room_ = room;
    }
};

class SafeChatRoom {
    std::vector<std::weak_ptr<SafeUser>> users_;  // ✅ 安全
public:
    void addUser(std::shared_ptr<SafeUser> user) {
        users_.push_back(user);
    }
};
```

### 场景2：缓存系统中的循环引用
```cpp
class ResourceCache {
    std::unordered_map<std::string, std::shared_ptr<Resource>> cache_;
    
public:
    std::shared_ptr<Resource> getResource(const std::string& key) {
        auto it = cache_.find(key);
        if (it != cache_.end()) {
            return it->second;
        }
        
        // 创建新资源，资源内部可能引用缓存（导致循环引用）
        auto resource = std::make_shared<Resource>(key);
        
        // 解决方案：资源内部使用weak_ptr引用缓存
        resource->setCache(weak_from_this());  // 假设Resource有这个方法
        
        cache_[key] = resource;
        return resource;
    }
};
```

## 检测和调试循环引用

### 方法1：使用自定义删除器调试
```cpp
template<typename T>
void debugDeleter(T* ptr) {
    std::cout << "删除对象，地址: " << ptr << "\n";
    delete ptr;
}

void debugCircularReference() {
    struct DebugNode {
        std::shared_ptr<DebugNode> next;
        std::shared_ptr<DebugNode> prev;
        int data;
    };
    
    // 使用调试删除器
    auto node1 = std::shared_ptr<DebugNode>(new DebugNode, debugDeleter<DebugNode>);
    auto node2 = std::shared_ptr<DebugNode>(new DebugNode, debugDeleter<DebugNode>);
    
    node1->next = node2;
    node2->prev = node1;
    
    std::cout << "建立循环引用，观察析构函数是否被调用...\n";
}
```

### 方法2：监控引用计数
```cpp
class RefCountMonitor {
public:
    static void monitor(const std::shared_ptr<void>& sp, const std::string& name) {
        std::cout << name << " 引用计数: " << sp.use_count() << "\n";
    }
};

void monitorReferenceCounts() {
    struct Node {
        std::shared_ptr<Node> next;
        std::shared_ptr<Node> prev;
    };
    
    auto node1 = std::make_shared<Node>();
    auto node2 = std::make_shared<Node>();
    
    RefCountMonitor::monitor(node1, "node1初始");
    RefCountMonitor::monitor(node2, "node2初始");
    
    node1->next = node2;
    RefCountMonitor::monitor(node2, "node1->next = node2 后");
    
    node2->prev = node1;  
    RefCountMonitor::monitor(node1, "node2->prev = node1 后");
}
```

## 最佳实践总结

### 1. **设计时避免循环引用**
```cpp
// 提前规划所有权关系
// 子对象不应该拥有父对象的强引用
```

### 2. **使用weak_ptr打破循环**
```cpp
// 在可能形成循环的地方使用weak_ptr
class Child {
    std::weak_ptr<Parent> parent_;  // 而不是shared_ptr
};
```

### 3. **使用RAII和作用域**
```cpp
// 让对象的生命周期自然结束
{
    auto resource = acquireResource();
    useResource(resource);
} // resource自动释放
```

### 4. **定期代码审查**
```cpp
// 检查所有shared_ptr的使用
// 确保没有形成"引用环"
```

## 工具支持

### Valgrind 检测
```bash
valgrind --leak-check=full ./your_program
```

### AddressSanitizer
```bash
g++ -fsanitize=address -g your_program.cpp
```

循环引用是C++智能指针使用中最常见的问题之一，但通过合理的设计和`weak_ptr`的正确使用，完全可以避免。关键是要在编码时始终保持对对象所有权关系的清晰认识。