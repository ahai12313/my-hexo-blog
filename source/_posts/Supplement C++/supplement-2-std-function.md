---
title: supplement 2 std::function
date: 2025-11-18 22:18:09
tags:
categories: Supplement C++
priority: 1
---
`std::function` 是 C++11 引入的一个**函数包装器模板**，它可以存储、复制和调用任何可调用对象（函数、Lambda 表达式、函数对象等）。

## 基本定义

```cpp
#include <functional>

// 存储返回 void，无参数的函数
std::function<void()> func;

// 存储返回 int，接受两个 int 参数的函数  
std::function<int(int, int)> func2;
```

## 可以存储的可调用对象

### 1. 普通函数
```cpp
int add(int a, int b) {
    return a + b;
}

std::function<int(int, int)> f = add;
std::cout << f(2, 3); // 输出 5
```

### 2. Lambda 表达式
```cpp
std::function<int(int, int)> f = int a, int b {
    return a + b;
};
```

### 3. 函数对象（重载了 operator() 的类）
```cpp
struct Adder {
    int operator()(int a, int b) const {
        return a + b;
    }
};

Adder adder;
std::function<int(int, int)> f = adder;
```

### 4. 成员函数（需要绑定对象）
```cpp
class Calculator {
public:
    int multiply(int a, int b) {
        return a * b;
    }
};

Calculator calc;
std::function<int(int, int)> f = std::bind(&Calculator::multiply, &calc, 
                                          std::placeholders::_1, 
                                          std::placeholders::_2);
```

## 主要特性和操作

### 1. 调用操作
```cpp
std::function<int(int, int)> f = int a, int b { return a + b; };
int result = f(10, 20); // 调用存储的函数
```

### 2. 检查是否为空
```cpp
std::function<void()> f;
if (f) {  // 或者 f != nullptr
    f();  // 只有不为空时才调用
}
```

### 3. 重置和交换
```cpp
std::function<void()> f = []{ std::cout << "Hello"; };
f(); // 输出 Hello

f = nullptr;  // 重置为空
f = []{ std::cout << "World"; }; // 重新赋值

std::function<void()> g;
std::swap(f, g); // 交换两个 function 对象
```

## 类型擦除机制

`std::function` 的核心是**类型擦除**技术：

```cpp
// 内部简化实现
template<typename T>
class function;  // 未定义

template<typename Ret, typename... Args>
class function<Ret(Args...)> {
private:
    // 基类接口
    struct callable_base {
        virtual Ret operator()(Args...) = 0;
        virtual ~callable_base() = default;
    };
    
    // 具体类型的包装器
    template<typename F>
    struct callable_impl : callable_base {
        F f;
        callable_impl(F&& func) : f(std::forward<F>(func)) {}
        Ret operator()(Args... args) override {
            return f(args...);
        }
    };
    
    std::unique_ptr<callable_base> ptr;

public:
    template<typename F>
    function(F&& f) : ptr(new callable_impl<std::decay_t<F>>(std::forward<F>(f))) {}
    
    Ret operator()(Args... args) {
        return (*ptr)(args...);
    }
};
```

## 与闭包类型的区别

| 特性 | 闭包类型（Lambda） | `std::function` |
|------|-------------------|-----------------|
| **类型** | 具体的匿名类型 | 类型擦除的包装器 |
| **性能** | 零开销，内联优化 | 有运行时开销（虚函数调用） |
| **大小** | 取决于捕获内容 | 固定大小（通常较大） |
| **灵活性** | 只能存储特定Lambda | 可存储任何可调用对象 |
| **类型安全** | 编译时类型检查 | 运行时类型检查 |

## 实际应用场景

### 1. 回调函数系统
```cpp
class Button {
    std::function<void()> onClick;
public:
    void setOnClick(std::function<void()> callback) {
        onClick = std::move(callback);
    }
    
    void click() {
        if (onClick) onClick();
    }
};

Button btn;
btn.setOnClick([]{ std::cout << "Button clicked!"; });
btn.click();
```

### 2. 事件处理器
```cpp
class EventSystem {
    std::unordered_map<std::string, std::function<void(int)>> handlers;
public:
    void on(const std::string& event, std::function<void(int)> handler) {
        handlers[event] = std::move(handler);
    }
    
    void emit(const std::string& event, int value) {
        if (handlers.count(event)) {
            handlersvalue;
        }
    }
};
```

### 3. 策略模式
```cpp
class Sorter {
    std::function<bool(int, int)> comparator;
public:
    void setComparator(std::function<bool(int, int)> comp) {
        comparator = std::move(comp);
    }
    
    void sort(std::vector<int>& data) {
        std::sort(data.begin(), data.end(), comparator);
    }
};

Sorter sorter;
sorter.setComparator(int a, int b { return a > b; }); // 降序
```

## 性能考虑

```cpp
// 高性能场景：直接使用闭包类型（零开销）
auto lambda = int x { return x * 2; };
// 编译器可以内联优化

// 灵活但较慢：使用 std::function
std::function<int(int)> f = lambda;
// 有虚函数调用开销，难以内联
```

## 总结

`std::function` 是 C++ 中强大的多态函数包装器：

- **统一接口**：可以用统一类型存储各种可调用对象
- **类型安全**：提供编译时类型检查
- **灵活性强**：支持函数组合和回调机制
- **性能代价**：相比直接调用有额外开销

适用于需要存储或传递不同类型可调用对象的场景，但在性能关键路径中应谨慎使用。