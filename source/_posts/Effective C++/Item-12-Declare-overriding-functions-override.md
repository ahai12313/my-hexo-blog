---
title: 'Item 12: Declare overriding functions override'
date: 2025-11-22 18:55:49
tags:
categories: Effective C++
priority: 11
---
# Item 12：声明重写函数为override

## 1. 问题背景

C++中的面向对象编程围绕类、继承和虚函数展开。虚函数重写（overriding）是核心概念之一，但很容易出错。重写与重载（overloading）虽然听起来相似，但完全不同：

- **重写**：派生类函数覆盖基类虚函数，实现运行时多态
- **重载**：同一作用域内函数名相同但参数不同，编译时决定

## 2. 虚函数重写的要求

要使重写成功发生，必须满足以下所有条件：

### 2.1 C++98的基本要求
1. **基类函数必须是virtual的**
2. **函数名必须完全相同**（析构函数除外）
3. **参数类型必须完全相同**
4. **const限定必须相同**
5. **返回类型和异常规范必须兼容**

### 2.2 C++11新增要求
6. **引用限定符必须相同**

## 3. 引用限定符（Reference Qualifiers）

### 3.1 基本概念
引用限定符是C++11的相对不为人知的特性，允许限制成员函数只能被左值或右值调用：

```cpp
class Widget {
public:
    void doWork() &;    // 只能被左值对象调用
    void doWork() &&;   // 只能被右值对象调用
};

Widget w;               // 左值
w.doWork();             // 调用 Widget::doWork() &

Widget makeWidget();    // 返回右值的工厂函数
makeWidget().doWork();  // 调用 Widget::doWork() &&
```

### 3.2 与虚函数重写的关系
如果基类虚函数有引用限定符，派生类的重写函数必须有相同的引用限定符，否则不会发生重写。

## 4. 常见的重写错误示例

```cpp
class Base {
public:
    virtual void mf1() const;
    virtual void mf2(int x);
    virtual void mf3() &;
    void mf4() const;  // 非virtual！
};

class Derived: public Base {
public:
    virtual void mf1();        // ❌ 错误：缺少const
    virtual void mf2(unsigned int x);  // ❌ 错误：参数类型不匹配
    virtual void mf3() &&;     // ❌ 错误：引用限定符不匹配
    void mf4() const;          // ❌ 错误：基类函数非virtual
};
```

**问题分析：**
- `mf1`：基类有const，派生类没有
- `mf2`：参数类型不同（int vs unsigned int）
- `mf3`：引用限定符不同（& vs &&）
- `mf4`：基类函数不是virtual，无法重写

## 5. C++11解决方案：override关键字

### 5.1 基本用法
使用`override`明确表示函数意图重写基类虚函数：

```cpp
class Derived: public Base {
public:
    virtual void mf1() override;        // ❌ 编译错误：不满足重写条件
    virtual void mf2(unsigned int x) override;  // ❌ 编译错误
    virtual void mf3() && override;     // ❌ 编译错误
    void mf4() const override;          // ❌ 编译错误
};
```

### 5.2 正确的重写示例

```cpp
class Base {
public:
    virtual void mf1() const;
    virtual void mf2(int x);
    virtual void mf3() &;
    virtual void mf4() const;  // 必须声明为virtual
};

class Derived: public Base {
public:
    virtual void mf1() const override;    // ✅ 正确
    virtual void mf2(int x) override;      // ✅ 正确  
    virtual void mf3() & override;        // ✅ 正确
    void mf4() const override;            // ✅ 正确（virtual可选）
};
```

## 6. override的优势

### 6.1 编译时错误检测
- **无override**：错误可能被忽略，代码合法但行为错误
- **有override**：编译器立即报错，强制修正

### 6.2 代码维护性
当修改基类虚函数签名时：
```cpp
// 修改前
virtual void process(int x);

// 修改后  
virtual void process(long x);
```

使用override的派生类会立即编译失败，明确显示影响范围，便于评估修改代价。

### 6.3 上下文关键字
`override`是上下文关键字，只在成员函数声明末尾有特殊含义，兼容现有代码：

```cpp
class LegacyCode {
public:
    void override();  // ✅ 合法：override不是关键字
};
```

## 7. 引用限定符的实用案例

### 7.1 优化资源返回

```cpp
class Widget {
public:
    using DataType = std::vector<double>;
    
private:
    DataType values;

public:
    // 左值版本：返回引用，避免拷贝
    DataType& data() & {
        return values;
    }
    
    // 右值版本：移动返回，优化性能
    DataType data() && {
        return std::move(values);
    }
};
```

### 7.2 使用示例

```cpp
Widget w;
auto vals1 = w.data();              // 拷贝构造（左值版本）

auto vals2 = makeWidget().data();    // 移动构造（右值版本）
```

**性能优势**：对临时对象使用移动语义，避免不必要的拷贝。

## 8. final关键字（相关特性）

虽然不是本条款重点，但`final`与`override`相关：
- `final`用于虚函数：阻止进一步重写
- `final`用于类：阻止继承

```cpp
class Base {
public:
    virtual void process() final;  // 不能重写
};

class FinalClass final {  // 不能继承
    // ...
};
```

## 9. 最佳实践指南

### 9.1 强制使用override
**对所有意图重写的函数使用override**，包括：
- 明显的虚函数重写
- 可能被误认为重载的情况
- 复杂的模板相关重写

### 9.2 代码审查检查点
在代码审查中检查：
1. 所有虚函数重写是否都有override
2. 基类函数是否确实声明为virtual
3. 函数签名是否完全匹配
4. 引用限定符是否一致

### 9.3 现代C++类设计示例

```cpp
class Shape {
public:
    virtual ~Shape() = default;
    virtual double area() const = 0;
    virtual void draw() & = 0;
    virtual std::string getName() const && = 0;
};

class Circle : public Shape {
public:
    double area() const override;
    void draw() & override;           // 左值限定
    std::string getName() const && override;  // 右值限定
};
```

## 10. 总结

**override关键字的核心价值：**

1. **编译时安全**：及早发现重写错误
2. **代码清晰**：明确表达设计意图
3. **维护友好**：便于重构和修改
4. **团队协作**：统一代码标准

**需要记住的要点：**
- 声明所有重写函数为override
- 成员函数引用限定符支持对左值/右值对象的区别处理
- override是上下文关键字，不影响现有代码
- 结合final可提供更精细的继承控制

通过坚持使用override，可以显著提高虚函数重写的安全性和代码质量，这是现代C++编程的基本最佳实践。