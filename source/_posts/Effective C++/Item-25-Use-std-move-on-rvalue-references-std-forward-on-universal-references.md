---
title: >-
  Item 25: Use std::move on rvalue references, std::forward on universal
  references
categories: Effective C++
date: 2025-12-04 21:05:40
tags:
priority: 25
---
# Effective Modern C++ Item 25 文档

## 对右值引用使用std::move，对通用引用使用std::forward

### 核心原则

**右值引用**应该无条件地转换为右值（使用`std::move`），因为**它们总是绑定到右值**。

**通用引用**应该有条件地转换为右值（使用`std::forward`），因为**它们只是有时绑定到右值**。

### 1. 右值引用的正确用法

右值引用参数明确表示该对象可以被移动，因此在将其传递给其他函数时应该使用`std::move`：

```cpp
class Widget {
public:
    Widget(Widget&& rhs)               // rhs是右值引用
    : name(std::move(rhs.name)),       // 正确：使用std::move
      p(std::move(rhs.p))
    { ... }
private:
    std::string name;
    std::shared_ptr<SomeDataStructure> p;
};
```

### 2. 通用引用的正确用法

通用引用（`T&&`，其中`T`需要推导）可能绑定到左值或右值，因此应该使用`std::forward`来保持值类别的完整性：

```cpp
class Widget {
public:
    template<typename T>
    void setName(T&& newName)          // newName是通用引用
    { name = std::forward<T>(newName); } // 正确：使用std::forward
    ...
};
```

### 3. 常见错误及后果

#### 错误1：在通用引用上使用std::move

```cpp
// 错误的实现
template<typename T>
void setName(T&& newName)
{ name = std::move(newName); } // 危险！可能意外移动左值

// 使用示例
std::string n = "test";
w.setName(n); // n的值被意外移动，n现在状态未知！
```

**后果**：左值参数会被意外移动，导致调用方数据被修改，这是极其危险的行为。

#### 错误2：在右值引用上使用std::forward

虽然技术上可行，但代码冗长、容易出错且不符合习惯用法，应该避免。

### 4. 为什么不使用重载替代通用引用？

有人可能建议使用重载函数替代通用引用：

```cpp
class Widget {
public:
    void setName(const std::string& newName) { name = newName; } // 左值版本
    void setName(std::string&& newName) { name = std::move(newName); } // 右值版本
};
```

但这种方案存在三个问题：

1. **代码冗余**：需要维护多个重载版本
2. **性能损失**：可能引入不必要的临时对象创建
3. **可扩展性差**：n个参数需要2ⁿ个重载，对于变长参数模板完全不可行

### 5. 多次使用引用时的注意事项

当引用在函数内被多次使用时，只在**最后一次使用**时应用`std::move`或`std::forward`：

```cpp
template<typename T>
void setSignText(T&& text)
{
    sign.setText(text);                    // 第一次使用：不转换
    auto now = std::chrono::system_clock::now();
    signHistory.add(now, std::forward<T>(text)); // 最后一次使用：条件转换
}
```

### 6. 返回值的处理

#### 对于函数参数

当返回右值引用或通用引用参数时，应该使用`std::move`或`std::forward`：

```cpp
// 正确：返回右值引用参数
Matrix operator+(Matrix&& lhs, const Matrix& rhs)
{
    lhs += rhs;
    return std::move(lhs); // 启用移动语义
}

// 正确：返回通用引用参数  
template<typename T>
Fraction reduceAndCopy(T&& frac)
{
    frac.reduce();
    return std::forward<T>(frac); // 保持值类别
}
```

#### 对于局部变量（重要！）

**不要**对返回的局部变量使用`std::move`：

```cpp
// 正确：依赖RVO（返回值优化）
Widget makeWidget()
{
    Widget w;
    // ... 配置w
    return w; // 编译器会优化，可能完全避免拷贝/移动
}

// 错误：阻止RVO！
Widget makeWidget()
{
    Widget w;
    // ... 配置w
    return std::move(w); // 阻止优化，强制移动！
}
```

### 7. 返回值优化（RVO）详解

RVO允许编译器在满足以下条件时省略拷贝/移动：
1. 局部对象的类型与函数返回类型相同
2. 返回的是局部对象本身（不是其引用或转换结果）

即使编译器不执行RVO，标准也要求将返回的局部对象视为右值，所以手动添加`std::move`是多余且有害的。

### 8. 特殊情况下使用std::move_if_noexcept

在极少数需要异常安全的场景中，考虑使用`std::move_if_noexcept`替代`std::move`（详见Item 14）。

## 总结

| 情况 | 应该使用 | 说明 |
|------|----------|------|
| 右值引用参数转发 | `std::move` | 无条件转换为右值 |
| 通用引用参数转发 | `std::forward` | 有条件地保持值类别 |
| 返回右值引用参数 | `std::move` | 启用移动语义 |
| 返回通用引用参数 | `std::forward` | 保持值类别 |
| 返回局部变量 | **都不使用** | 依赖RVO |
|参数多次使用时| 最后一次使用才转换 | 确保之前的使用不受影响 |

**黄金法则**：相信编译器的优化能力，特别是在RVO方面；只在明确需要干预值类别时才使用`std::move`和`std::forward`。