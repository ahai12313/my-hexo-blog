---
title: supplement 15 三五法则
categories: Supplement C++
date: 2025-11-23 22:46:03
tags:
priority: 14
---
# C++ 三五法则（Rule of Five）

三五法则是现代C++中关于资源管理类的**核心设计原则**，它扩展了传统的三法则（Rule of Three）。

## 基本概念

### 三法则（传统）
如果一个类需要自定义以下任何一个，通常需要自定义所有三个：
1. **析构函数**
2. **拷贝构造函数**  
3. **拷贝赋值运算符**

### 三五法则（现代C++11+）
如果一个类需要自定义以下任何一个，通常需要自定义所有五个：
1. **析构函数**
2. **拷贝构造函数**
3. **拷贝赋值运算符**
4. **移动构造函数**
5. **移动赋值运算符**

## 详细解释

### 五个特殊成员函数

```cpp
class ResourceManager {
    int* data;
    size_t size;
    
public:
    // 1. 析构函数
    ~ResourceManager() { delete[] data; }
    
    // 2. 拷贝构造函数
    ResourceManager(const ResourceManager& other) 
        : size(other.size), data(new int[other.size]) {
        std::copy(other.data, other.data + size, data);
    }
    
    // 3. 拷贝赋值运算符
    ResourceManager& operator=(const ResourceManager& other) {
        if (this != &other) {
            delete[] data;
            size = other.size;
            data = new int[size];
            std::copy(other.data, other.data + size, data);
        }
        return *this;
    }
    
    // 4. 移动构造函数
    ResourceManager(ResourceManager&& other) noexcept 
        : data(other.data), size(other.size) {
        other.data = nullptr;  // 重要：置空源对象
        other.size = 0;
    }
    
    // 5. 移动赋值运算符
    ResourceManager& operator=(ResourceManager&& other) noexcept {
        if (this != &other) {
            delete[] data;      // 释放当前资源
            data = other.data;  // 接管资源
            size = other.size;
            other.data = nullptr;
            other.size = 0;
        }
        return *this;
    }
};
```

## 违反三五法则的危险

### 示例：不完整的移动语义
```cpp
class Dangerous {
    std::vector<int>* heavyData;
public:
    Dangerous() : heavyData(new std::vector<int>(1000)) {}
    
    ~Dangerous() { delete heavyData; }  // 有析构函数
    
    // 只声明了移动构造函数，违反三五法则！
    Dangerous(Dangerous&& other) : heavyData(other.heavyData) {
        other.heavyData = nullptr;
    }
    
    // 缺少移动赋值运算符！
    // 编译器会删除拷贝操作！
};

int main() {
    Dangerous d1;
    Dangerous d2 = std::move(d1);  // ✅ 移动构造OK
    
    Dangerous d3;
    // d3 = std::move(d2);  // ❌ 编译错误！没有移动赋值运算符
    // 拷贝操作也被隐式删除
}
```

## 三五法则的例外情况

### 1. 使用 `= default`
```cpp
class SafeByDefault {
    std::unique_ptr<int> ptr;  // 自动管理资源
public:
    // 使用默认实现，遵循零法则
    ~SafeByDefault() = default;
    SafeByDefault(const SafeByDefault&) = default;
    SafeByDefault& operator=(const SafeByDefault&) = default;
    SafeByDefault(SafeByDefault&&) = default;
    SafeByDefault& operator=(SafeByDefault&&) = default;
};
```

### 2. 删除不需要的操作
```cpp
class NonCopyable {
    std::unique_ptr<int> resource;
public:
    NonCopyable() = default;
    ~NonCopyable() = default;
    
    // 禁止拷贝，允许移动
    NonCopyable(const NonCopyable&) = delete;
    NonCopyable& operator=(const NonCopyable&) = delete;
    
    NonCopyable(NonCopyable&&) = default;
    NonCopyable& operator=(NonCopyable&&) = default;
};
```

## 现代最佳实践：零法则（Rule of Zero）

**优先使用**零法则：让类不需要自定义任何特殊成员函数。

```cpp
// 遵循零法则 - 最佳实践
class RuleOfZero {
    std::vector<int> data;           // 自动管理内存
    std::unique_ptr<Resource> res;    // 自动管理资源
    std::string name;                // 自动管理字符串
    
    // 不需要自定义五大函数！
    // 编译器生成的默认版本完全正确
public:
    RuleOfZero(std::string n) : name(std::move(n)) {}
    
    // 自动获得正确的拷贝/移动语义
};
```

## 总结表格

| 法则 | 适用场景 | 核心思想 |
|------|----------|----------|
| **三法则** | C++98/03资源管理 | 自定义析构→需自定义拷贝 |
| **三五法则** | 现代C++资源管理 | 自定义任一→需自定义所有五个 |
| **零法则** | 现代C++最佳实践 | 使用RAII对象，避免自定义 |

**关键建议**：优先遵循零法则，如果必须自定义资源管理，则严格遵守三五法则！