---
title: supplement 9 类的基本结构与组成
date: 2025-11-20 22:11:14
tags:
categories: Effective C++
---
在C++中，一个完整的类通常包含以下部分。以下是类的通用形式和各部分的说明：

## 1. 基本结构

```cpp
class ClassName {
// 访问修饰符
public:     // 公有成员
private:    // 私有成员（默认）
protected:  // 保护成员

// 构造函数和析构函数
public:
    ClassName();                           // 默认构造函数
    ClassName(const ClassName& other);     // 拷贝构造函数
    ClassName(ClassName&& other) noexcept; // 移动构造函数
    ~ClassName();                          // 析构函数

// 赋值运算符
    ClassName& operator=(const ClassName& other);     // 拷贝赋值
    ClassName& operator=(ClassName&& other) noexcept; // 移动赋值

// 成员函数
    void memberFunction();
    int getValue() const;  // const成员函数
    
// 静态成员
    static int staticMember;
    static void staticFunction();

// 成员变量
private:
    int m_memberVar;
    std::string m_name;
};
```

## 2. 详细示例

```cpp
#include <string>
#include <iostream>

class Person {
private: // 数据成员通常设为private
    std::string m_name;
    int m_age;
    static int s_count;  // 静态成员变量

public: // 构造函数
    // 默认构造函数
    Person() : m_name("Unknown"), m_age(0) { s_count++; }
    
    // 带参数构造函数
    Person(const std::string& name, int age) : m_name(name), m_age(age) { s_count++; }
    
    // 拷贝构造函数
    Person(const Person& other) : m_name(other.m_name), m_age(other.m_age) { s_count++; }
    
    // 移动构造函数
    Person(Person&& other) noexcept 
        : m_name(std::move(other.m_name)), m_age(other.m_age) { s_count++; }
    
    // 析构函数
    ~Person() { s_count--; }

public: // 赋值运算符
    // 拷贝赋值
    Person& operator=(const Person& other) {
        if (this != &other) {
            m_name = other.m_name;
            m_age = other.m_age;
        }
        return *this;
    }
    
    // 移动赋值
    Person& operator=(Person&& other) noexcept {
        if (this != &other) {
            m_name = std::move(other.m_name);
            m_age = other.m_age;
        }
        return *this;
    }

public: // 成员函数
    // getter函数
    std::string getName() const { return m_name; }
    int getAge() const { return m_age; }
    
    // setter函数
    void setName(const std::string& name) { m_name = name; }
    void setAge(int age) { m_age = age; }
    
    // 普通成员函数
    void introduce() const {
        std::cout << "Name: " << m_name << ", Age: " << m_age << std::endl;
    }
    
    // 静态成员函数
    static int getCount() { return s_count; }

private: // 辅助函数
    void validateAge() {
        if (m_age < 0) m_age = 0;
    }
};

// 静态成员定义
int Person::s_count = 0;
```

## 3. 现代C++的改进写法

```cpp
class ModernClass {
private:
    std::string m_data;
    std::unique_ptr<int> m_ptr;

public:
    // 使用=default
    ModernClass() = default;
    
    // 委托构造函数
    ModernClass(const std::string& data) : m_data(data) {}
    
    // 使用auto返回类型
    auto getData() const -> const std::string& { return m_data; }
    
    // noexcept说明符
    void safeFunction() noexcept { }
    
    // constexpr构造函数
    constexpr ModernClass(int value) : m_data(std::to_string(value)) {}
    
    // 三路比较运算符 (C++20)
    auto operator<=>(const ModernClass&) const = default;
};
```

## 4. 必要编写的核心部分

**必须包含的部分：**
- 访问修饰符（public/private/protected）
- 数据成员
- 成员函数
- 至少一个构造函数

**推荐包含的部分：**
- 析构函数
- 拷贝构造和拷贝赋值（或禁用它们）
- 移动构造和移动赋值（C++11+）
- getter/setter方法封装数据

## 5. 特殊情况的处理

```cpp
class NonCopyable {
public:
    NonCopyable() = default;
    
    // 禁用拷贝
    NonCopyable(const NonCopyable&) = delete;
    NonCopyable& operator=(const NonCopyable&) = delete;
    
    // 允许移动
    NonCopyable(NonCopyable&&) = default;
    NonCopyable& operator=(NonCopyable&&) = default;
};
```

根据实际需求选择需要实现的部分，避免过度设计。