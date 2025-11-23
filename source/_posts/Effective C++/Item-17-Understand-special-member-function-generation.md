---
title: 'Item 17: Understand special member function generation'
categories: Effective C++
date: 2025-11-23 20:56:51
tags:
priority: 16
---
# Item 17：理解特殊成员函数的生成规则

## 1. 特殊成员函数概述

### 1.1 什么是特殊成员函数
特殊成员函数是C++编译器**自动生成**的函数，包括：

| C++98 特殊成员函数 | C++11 新增特殊成员函数 |
|-------------------|----------------------|
| 默认构造函数 | 移动构造函数 |
| 析构函数 | 移动赋值运算符 |
| 拷贝构造函数 |  |
| 拷贝赋值运算符 |  |

## 2. C++98 特殊成员函数生成规则

### 2.1 基本生成规则
```cpp
class Widget98 {
    // 编译器自动生成（如果需要）：
    // Widget98();                          // 默认构造函数
    // ~Widget98();                         // 析构函数  
    // Widget98(const Widget98&);           // 拷贝构造函数
    // Widget98& operator=(const Widget98&);// 拷贝赋值运算符
};
```

### 2.2 生成条件
- **按需生成**：只有在代码中使用时才生成
- **默认构造函数**：仅在类**没有声明任何构造函数**时生成
- **内联且public**：生成的函数都是内联的public函数
- **虚函数**：仅当基类有虚析构函数时，派生类的析构函数才为虚函数

## 3. C++11 新增的移动操作

### 3.1 移动操作的签名
```cpp
class Widget {
public:
    Widget(Widget&& rhs);              // 移动构造函数
    Widget& operator=(Widget&& rhs);   // 移动赋值运算符
};
```

### 3.2 成员移动语义
```cpp
class String {
    std::string data_;
    int size_;
    
public:
    // 编译器生成的移动构造函数大致如下：
    String(String&& rhs) 
        : data_(std::move(rhs.data_)),    // 移动string成员
          size_(std::move(rhs.size_))     // 移动int成员
    {}
};
```

## 4. 移动操作的生成规则

### 4.1 生成条件（重要！）
移动操作在**同时满足以下三个条件**时才会生成：

```cpp
class Movable {
    // 条件1：没有声明拷贝操作
    // 条件2：没有声明移动操作  
    // 条件3：没有声明析构函数
    // → 编译器会生成移动操作
};

class NotMovable {
public:
    NotMovable(const NotMovable&);    // 条件1不满足：声明了拷贝操作
    ~NotMovable();                    // 条件3不满足：声明了析构函数
    // ❌ 不会生成移动操作！
};
```

### 4.2 移动操作间的依赖关系
```cpp
class PartialMove {
public:
    PartialMove(PartialMove&&);       // 声明了移动构造函数
    // ❌ 不会生成移动赋值运算符（相互依赖）
    
    PartialMove& operator=(const PartialMove&) = default;  // 显式要求生成拷贝赋值
};
```

## 5. 拷贝操作的生成规则变化

### 5.1 拷贝与移动的交互
```cpp
class Interactions {
public:
    Interactions(const Interactions&);    // 声明拷贝构造函数
    // ❌ 不会生成移动操作（拷贝操作抑制移动操作生成）
    
    Interactions(Interactions&&) = default; // 显式声明移动构造函数  
    // ❌ 不会生成拷贝操作（移动操作抑制拷贝操作生成）
};
```

### 5.2 生成规则的总结表
| 声明的函数 | 默认构造函数 | 析构函数 | 拷贝构造函数 | 拷贝赋值运算符 | 移动构造函数 | 移动赋值运算符 |
|-----------|------------|---------|------------|-------------|------------|-------------|
| 无声明 | 生成 | 生成 | 生成 | 生成 | 生成 | 生成 |
| 默认构造函数 | - | 生成 | 生成 | 生成 | 生成 | 生成 |
| 析构函数 | 生成 | - | 生成 | 生成 | ❌ | ❌ |
| 拷贝构造函数 | ❌ | 生成 | - | 生成 | ❌ | ❌ |
| 拷贝赋值运算符 | 生成 | 生成 | 生成 | - | ❌ | ❌ |
| 移动构造函数 | ❌ | 生成 | ❌ | ❌ | - | ❌ |
| 移动赋值运算符 | 生成 | 生成 | ❌ | ❌ | ❌ | - |

## 6. "三法则" 与 "五法则"

### 6.1 三法则（Rule of Three）
```cpp
// 传统三法则：需要资源管理的类应该声明这三个函数
class RuleOfThree {
public:
    RuleOfThree(const char* data) : data_(new char[strlen(data) + 1]) {
        strcpy(data_, data);
    }
    
    // 1. 拷贝构造函数
    RuleOfThree(const RuleOfThree& other) 
        : data_(new char[strlen(other.data_) + 1]) {
        strcpy(data_, other.data_);
    }
    
    // 2. 拷贝赋值运算符
    RuleOfThree& operator=(const RuleOfThree& other) {
        if (this != &other) {
            delete[] data_;
            data_ = new char[strlen(other.data_) + 1];
            strcpy(data_, other.data_);
        }
        return *this;
    }
    
    // 3. 析构函数
    ~RuleOfThree() { delete[] data_; }
    
private:
    char* data_;
};
```

### 6.2 五法则（Rule of Five） - C++11扩展
```cpp
class RuleOfFive {
public:
    // 三法则的三个函数 +
    
    // 4. 移动构造函数
    RuleOfFive(RuleOfFive&& other) noexcept 
        : data_(other.data_) {          // 转移资源所有权
        other.data_ = nullptr;
    }
    
    // 5. 移动赋值运算符  
    RuleOfFive& operator=(RuleOfFive&& other) noexcept {
        if (this != &other) {
            delete[] data_;
            data_ = other.data_;        // 转移资源所有权
            other.data_ = nullptr;
        }
        return *this;
    }
    
private:
    char* data_;
};
```

## 7. 使用 `= default` 显式控制

### 7.1 显式要求默认行为
```cpp
class ExplicitDefault {
public:
    ExplicitDefault() = default;
    ~ExplicitDefault() = default;
    
    // 显式要求编译器生成拷贝操作
    ExplicitDefault(const ExplicitDefault&) = default;
    ExplicitDefault& operator=(const ExplicitDefault&) = default;
    
    // 显式要求编译器生成移动操作
    ExplicitDefault(ExplicitDefault&&) = default;
    ExplicitDefault& operator=(ExplicitDefault&&) = default;
};
```

### 7.2 多态基类的正确做法
```cpp
class PolymorphicBase {
public:
    virtual ~PolymorphicBase() = default;  // 虚析构函数
    
    // 由于声明了析构函数，需要显式要求移动操作
    PolymorphicBase(PolymorphicBase&&) = default;
    PolymorphicBase& operator=(PolymorphicBase&&) = default;
    
    // 移动操作抑制了拷贝操作，需要显式要求
    PolymorphicBase(const PolymorphicBase&) = default;
    PolymorphicBase& operator=(const PolymorphicBase&) = default;
};
```

## 8. 实际案例与陷阱

### 8.1 隐蔽的性能陷阱
```cpp
// 初始版本：性能良好
class StringTable {
public:
    // 编译器生成所有特殊成员函数，包括移动操作
private:
    std::map<int, std::string> values;
};

// 添加日志功能后：性能灾难！
class StringTableWithLog {
public:
    StringTableWithLog() { makeLogEntry("Creating"); }
    ~StringTableWithLog() { makeLogEntry("Destroying"); }  // 🚨 抑制移动操作生成！
    
    // 现在移动StringTableWithLog会执行拷贝！
private:
    std::map<int, std::string> values;  // 拷贝map非常昂贵！
};

void example() {
    StringTableWithLog t1;
    StringTableWithLog t2 = std::move(t1);  // 实际执行拷贝！
}
```

### 8.2 正确的解决方案
```cpp
class CorrectStringTable {
public:
    CorrectStringTable() { makeLogEntry("Creating"); }
    ~CorrectStringTable() { makeLogEntry("Destroying"); }
    
    // 显式要求移动操作
    CorrectStringTable(CorrectStringTable&&) = default;
    CorrectStringTable& operator=(CorrectStringTable&&) = default;
    
    // 移动操作抑制了拷贝操作，需要显式要求
    CorrectStringTable(const CorrectStringTable&) = default;
    CorrectStringTable& operator=(const CorrectStringTable&) = default;
    
private:
    std::map<int, std::string> values;
};
```

## 9. 成员函数模板的影响

### 9.1 模板不抑制特殊成员函数生成
```cpp
class Widget {
public:
    // 成员函数模板
    template<typename T>
    Widget(const T& rhs);  // 通用拷贝构造函数
    
    template<typename T>  
    Widget& operator=(const T& rhs);  // 通用赋值运算符
    
    // 编译器仍然会生成特殊的成员函数！
    // Widget(const Widget&);           // 仍然生成
    // Widget& operator=(const Widget&);// 仍然生成
};
```

## 10. C++11特殊成员函数完整规则

### 10.1 详细生成规则

#### 10.1.1 默认构造函数
- **生成条件**：类中没有用户声明的构造函数
- **行为**：与C++98相同

#### 10.1.2 析构函数  
- **生成条件**：总是生成（除非被显式删除）
- **新特性**：默认`noexcept`（见Item 14）
- **虚函数**：仅当基类有虚析构函数时

#### 10.1.3 拷贝构造函数
- **生成条件**：没有用户声明的拷贝构造函数
- **抑制条件**：用户声明了移动操作
- **不推荐**：在声明了拷贝赋值运算符或析构函数的类中生成（已弃用）

#### 10.1.4 拷贝赋值运算符
- **生成条件**：没有用户声明的拷贝赋值运算符  
- **抑制条件**：用户声明了移动操作
- **不推荐**：在声明了拷贝构造函数或析构函数的类中生成（已弃用）

#### 10.1.5 移动操作
- **生成条件**：同时满足：
  1. 没有用户声明的拷贝操作
  2. 没有用户声明的移动操作  
  3. 没有用户声明的析构函数

## 11. 现代C++最佳实践

### 11.1 五法则实践
```cpp
class ModernClass {
public:
    // 1. 默认构造函数（如果需要）
    ModernClass() = default;
    
    // 2. 析构函数
    ~ModernClass() = default;
    
    // 3. 拷贝构造函数
    ModernClass(const ModernClass&) = default;
    
    // 4. 拷贝赋值运算符
    ModernClass& operator=(const ModernClass&) = default;
    
    // 5. 移动构造函数
    ModernClass(ModernClass&&) = default;
    
    // 6. 移动赋值运算符
    ModernClass& operator=(ModernClass&&) = default;
    
    // 其他成员函数...
};
```

### 11.2 资源管理类的模板
```cpp
template<typename T>
class ResourceManager {
public:
    // 遵循五法则
    ResourceManager() = default;
    ~ResourceManager() { cleanup(); }
    
    ResourceManager(const ResourceManager&) = delete;  // 禁止拷贝
    ResourceManager& operator=(const ResourceManager&) = delete;
    
    ResourceManager(ResourceManager&& other) noexcept 
        : resource_(std::exchange(other.resource_, nullptr)) {}
        
    ResourceManager& operator=(ResourceManager&& other) noexcept {
        if (this != &other) {
            cleanup();
            resource_ = std::exchange(other.resource_, nullptr);
        }
        return *this;
    }
    
private:
    T* resource_ = nullptr;
    void cleanup() { /* 资源清理逻辑 */ }
};
```

## 12. 总结与检查清单

### 12.1 关键要点总结
1. **移动操作生成很保守**：只有"干净"的类才会自动生成移动操作
2. **拷贝移动互斥**：声明拷贝操作会抑制移动操作，反之亦然
3. **析构函数的影响**：用户声明的析构函数会抑制移动操作生成
4. **显式优于隐式**：使用`= default`明确表达意图
5. **模板无影响**：成员函数模板不抑制特殊成员函数生成

### 12.2 代码审查检查清单
- [ ] 需要资源管理的类是否遵循五法则？
- [ ] 多态基类是否显式声明了虚析构函数？
- [ ] 是否使用了`= default`来明确意图？
- [ ] 是否避免了意外的移动操作抑制？
- [ ] 成员函数模板是否影响了特殊成员函数生成？

### 12.3 决策流程图
```
需要特殊成员函数吗？
├── 是自动生成的吗？
│   ├── 是 → 验证生成条件是否满足
│   └── 否 → 显式声明（=default或自定义实现）
├── 需要移动语义吗？
│   ├── 是 → 确保没有抑制移动操作生成的声明
│   └── 否 → 考虑显式删除移动操作
└── 需要禁止拷贝吗？
    ├── 是 → 使用=delete
    └── 否 → 确保拷贝语义正确
```

理解这些规则对于编写高效、正确的C++代码至关重要，特别是在现代C++中移动语义普遍使用的情况下。