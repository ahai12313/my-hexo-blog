---
title: 'Item 11: Prefer deleted functions to private undefined ones'
date: 2025-11-22 18:54:50
tags:
categories: Effective C++
---
# Item 11：优先选用删除函数，而非私有未定义函数

## 1. 问题背景

在C++编程中，有时我们需要阻止特定函数被调用。对于普通函数，最简单的做法是不声明它们。然而，对于C++自动生成的**特殊成员函数**（如拷贝构造函数、拷贝赋值运算符等），我们需要更明确的方法来阻止其使用。

## 2. C++98的解决方案及其局限性

### 2.1 传统做法：私有化且不定义

在C++98中，常见的做法是将目标函数声明为`private`且不提供定义：

```cpp
// C++98 风格：私有未定义函数
class NonCopyable {
private:
    NonCopyable(const NonCopyable&);            // 只声明，不定义
    NonCopyable& operator=(const NonCopyable&); // 只声明，不定义
public:
    NonCopyable() = default;
};
```

### 2.2 工作原理
- **私有声明**：阻止外部代码直接调用
- **不定义**：防止成员函数或友元函数调用时链接成功

### 2.3 局限性
1. **错误诊断延迟**：误用只能在**链接时**被发现，而非编译时
2. **适用范围有限**：只能用于类成员函数，无法用于普通函数或模板
3. **错误信息不清晰**：编译器可能先报私有访问错误，而非函数不可用

## 3. C++11的现代解决方案：删除函数

### 3.1 基本语法

```cpp
// C++11 风格：删除函数
class NonCopyable {
public:
    NonCopyable(const NonCopyable&) = delete;
    NonCopyable& operator=(const NonCopyable&) = delete;
    NonCopyable() = default;
};
```

### 3.2 核心优势

| 特性 | C++98（私有未定义） | C++11（删除函数） |
|------|-------------------|------------------|
| 错误检测时机 | 链接时 | **编译时** |
| 错误信息质量 | 可能混淆访问权限 | 直接明确 |
| 适用范围 | 仅类成员函数 | **任何函数** |
| 模板支持 | 有限制 | 完整支持 |

### 3.3 错误检测对比

```cpp
class Widget98 {
private:
    Widget98(const Widget98&); // 私有未定义
};

class Widget11 {
public:
    Widget11(const Widget11&) = delete; // 删除函数
};

void test() {
    // C++98: 编译通过，链接错误
    // Widget98 w1;
    // Widget98 w2(w1); // 链接时错误
    
    // C++11: 编译错误
    // Widget11 w1;
    // Widget11 w2(w1); // 编译时错误：使用已删除的函数
}
```

## 4. 删除函数的进阶应用

### 4.1 阻止不期望的类型转换

```cpp
// 只允许int参数，阻止隐式转换
bool isLucky(int number);            // 基础函数

// 删除不期望的重载版本
bool isLucky(char) = delete;         // 阻止char参数
bool isLucky(bool) = delete;         // 阻止bool参数  
bool isLucky(double) = delete;       // 阻止double/float参数

// 使用示例
void usage() {
    isLucky(42);     // ✅ 允许
    // isLucky('a'); // ❌ 编译错误：调用已删除函数
    // isLucky(3.14); // ❌ 编译错误
}
```

### 4.2 控制模板实例化

```cpp
// 基础函数模板
template<typename T>
void processPointer(T* ptr) {
    // 处理指针的通用实现
}

// 禁止特定类型的实例化
template<>
void processPointer<void>(void*) = delete;

template<>
void processPointer<char>(char*) = delete;

// 也可以禁止const版本
template<>
void processPointer<const void>(const void*) = delete;
```

### 4.3 类内模板函数的删除

```cpp
class Widget {
public:
    // 通用模板
    template<typename T>
    void processPointer(T* ptr) { /* ... */ }
};

// 在类外删除特定实例化（命名空间作用域）
template<>
void Widget::processPointer<void>(void*) = delete;
```

## 5. 最佳实践指南

### 5.1 声明为public

删除函数应声明为`public`以获得更清晰的错误信息：

```cpp
class BestPractice {
public:  // 不是private!
    BestPractice(const BestPractice&) = delete;
    BestPractice& operator=(const BestPractice&) = delete;
};
```

**原因**：编译器先检查可访问性，再检查删除状态。如果是`private`，可能只报"private访问错误"而掩盖了"函数被删除"的事实。

### 5.2 迁移现有代码

将传统的C++98代码迁移到现代C++：

```cpp
// 迁移前（C++98）
class OldClass {
private:
    OldClass(const OldClass&);            // 要迁移
    OldClass& operator=(const OldClass&); // 要迁移
};

// 迁移后（C++11）
class NewClass {
public:  // 改为public
    NewClass(const NewClass&) = delete;            // 改为删除函数
    NewClass& operator=(const NewClass&) = delete; // 改为删除函数
};
```

### 5.3 完整示例：不可拷贝的资源管理类

```cpp
class UniqueResource {
private:
    ResourceHandle handle_;
    
public:
    // 删除拷贝操作，允许移动操作
    UniqueResource(const UniqueResource&) = delete;
    UniqueResource& operator=(const UniqueResource&) = delete;
    
    // 允许移动
    UniqueResource(UniqueResource&&) = default;
    UniqueResource& operator=(UniqueResource&&) = default;
    
    UniqueResource() : handle_(acquire_resource()) {}
    ~UniqueResource() { release_resource(handle_); }
    
    // 其他成员函数...
};
```

## 6. 总结

**删除函数相比私有未定义函数的主要优势：**

1. **更早的错误检测**：编译时而非链接时
2. **更清晰的错误信息**：直接指出函数被删除
3. **更广的适用范围**：可用于任何函数，包括非成员函数和模板
4. **更好的模板支持**：可精确控制模板实例化

**核心建议**：在新代码中始终使用`= delete`，在维护旧代码时优先将私有未定义函数迁移为public删除函数。

通过采用删除函数，我们能够编写出更安全、更清晰、更易于维护的现代C++代码。