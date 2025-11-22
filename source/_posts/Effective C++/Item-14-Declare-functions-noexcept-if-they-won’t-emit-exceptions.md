---
title: 'Item 14: Declare functions noexcept if they won’t emit exceptions'
date: 2025-11-22 21:13:47
tags:
categories: Effective C++
---
# Item 14：如果函数不会抛出异常，就将其声明为noexcept

## 1. 概述

`noexcept`是C++11引入的关键字，用于声明函数不会抛出任何异常。它替代了C++98中笨拙的异常规范，提供了更清晰、更高效的异常处理机制。

### 1.1 历史背景
- C++98使用`throw()`异常规范，但难以维护和使用
- C++11引入`noexcept`，提供简单的二分法：函数要么可能抛出异常，要么保证不抛出

### 1.2 基本语法
```cpp
// 无条件noexcept - 函数保证不抛出异常
void function() noexcept;

// 条件性noexcept - 根据表达式结果决定
void function() noexcept(noexcept(expression));

// 非noexcept函数 - 可能抛出异常
void function();
```

## 2. noexcept的优化价值

### 2.1 编译器优化机会
`noexcept`为编译器提供了重要的优化信息：

```cpp
class Optimizable {
public:
    // noexcept版本 - 允许更多优化
    void process() noexcept {
        // 编译器不需要维护完整的栈展开信息
        // 对象析构顺序可以重新排列
        heavy_computation();
    }
    
    // 非noexcept版本 - 优化受限
    void process() {
        // 编译器必须准备处理异常
        // 需要维护完整的栈展开信息
        heavy_computation();
    }
};
```

### 2.2 性能对比
| 异常规范类型 | 优化级别 | 栈展开要求 | 析构顺序约束 |
|------------|---------|-----------|------------|
| `noexcept` | 高      | 不需要    | 宽松       |
| `throw()`  | 中      | 需要      | 严格       |
| 无规范     | 低      | 需要      | 严格       |

## 3. 移动操作与noexcept

### 3.1 标准容器的行为
标准库容器（如`std::vector`）在重新分配内存时，会根据移动操作是否`noexcept`选择最优策略：

```cpp
class Widget {
private:
    std::vector<int> data_;
    
public:
    // 重要的移动操作应声明为noexcept
    Widget(Widget&& other) noexcept 
        : data_(std::move(other.data_)) {}  // 触发vector的移动
    
    Widget& operator=(Widget&& other) noexcept {
        data_ = std::move(other.data_);
        return *this;
    }
    
    // 如果移动操作不是noexcept，vector会使用复制而非移动
    // Widget(Widget&& other)  // 没有noexcept - 性能损失!
};

void demonstrate_vector_behavior() {
    std::vector<Widget> widgets;
    widgets.reserve(10);
    
    for (int i = 0; i < 20; ++i) {
        widgets.emplace_back();  // 可能触发重新分配
        // 如果Widget的移动操作是noexcept，使用移动语义
        // 否则，使用复制语义以保证异常安全
    }
}
```

### 3.2 异常安全保证
`push_back`的强异常安全保证要求：
- 如果重新分配过程中发生异常，原始容器状态不变
- 只有`noexcept`移动操作可以安全地用于重新分配

## 4. swap函数的noexcept

### 4.1 标准库中的条件性noexcept
```cpp
// 数组swap的条件性noexcept
template <class T, size_t N>
void swap(T (&a)[N], T (&b)[N]) 
    noexcept(noexcept(swap(*a, *b)));

// pair的swap也是条件性noexcept
template <class T1, class T2>
struct pair {
    void swap(pair& p) 
        noexcept(noexcept(swap(first, p.first)) && 
                 noexcept(swap(second, p.second)));
};
```

### 4.2 自定义类型的swap实现
```cpp
class Resource {
private:
    int* data_;
    size_t size_;
    
public:
    // 自定义swap应尽量声明为noexcept
    friend void swap(Resource& a, Resource& b) noexcept {
        using std::swap;
        swap(a.data_, b.data_);  // 指针交换是noexcept
        swap(a.size_, b.size_);  // 基本类型交换是noexcept
    }
    
    // 成员函数版本的swap
    void swap(Resource& other) noexcept {
        using std::swap;
        swap(*this, other);
    }
};
```

## 5. 正确使用noexcept的准则

### 5.1 应该使用noexcept的情况
```cpp
class ProperNoexceptUsage {
public:
    // 1. 移动操作 - 几乎总是应该noexcept
    ProperNoexceptUsage(ProperNoexceptUsage&&) noexcept = default;
    ProperNoexceptUsage& operator=(ProperNoexceptUsage&&) noexcept = default;
    
    // 2. 析构函数 - 隐式noexcept，但可以显式声明
    ~ProperNoExceptUsage() noexcept = default;
    
    // 3. 简单的getter函数
    int value() const noexcept { return value_; }
    
    // 4. 数学运算（如果确实不会抛出）
    double calculate() const noexcept { 
        return value_ * 1.5;  // 简单的数学运算
    }
    
    // 5. swap函数
    void swap(ProperNoexceptUsage& other) noexcept {
        std::swap(value_, other.value_);
    }

private:
    int value_;
};
```

### 5.2 不应该使用noexcept的情况
```cpp
class AvoidNoexcept {
public:
    // 1. 可能抛出异常的函数
    void loadFromFile(const std::string& filename) {
        // 文件操作可能抛出异常
        std::ifstream file(filename);
        if (!file) {
            throw std::runtime_error("Cannot open file");
        }
        // 不要声明为noexcept!
    }
    
    // 2. 异常中性函数（调用可能抛出异常的函数）
    void processData() {
        validateInput();    // 可能抛出
        transformData();     // 可能抛出  
        saveResults();       // 可能抛出
        // 不要声明为noexcept!
    }
    
    // 3. 虚函数（除非所有重写都能保证不抛出）
    virtual void operation() {
        // 派生类可能抛出异常
        // 通常不应该声明为noexcept
    }
};
```

### 5.3 窄合同函数的特殊考虑
```cpp
class NarrowContract {
public:
    // 窄合同函数：有先决条件
    // 先决条件：size <= MAX_SIZE
    void processBuffer(const char* buffer, size_t size) {
        // 可以选择检查先决条件
        if (size > MAX_SIZE) {
            // 选项1：抛出异常（但不能用noexcept）
            throw std::invalid_argument("Size too large");
            
            // 选项2：终止程序（可以用noexcept）
            // std::terminate();
        }
        
        // 实际处理...
    }
    
    // 如果选择选项2，可以声明为noexcept
    void processBufferNoExcept(const char* buffer, size_t size) noexcept {
        if (size > MAX_SIZE) {
            std::terminate();  // 违反先决条件，终止程序
        }
        // 处理逻辑...
    }
};
```

## 6. 条件性noexcept详解

### 6.1 基本用法
```cpp
template<typename T>
class Container {
private:
    T* data_;
    size_t size_;
    
public:
    // 移动构造函数：只有当T的移动构造是noexcept时，本函数才是noexcept
    Container(Container&& other) 
        noexcept(noexcept(T(std::declval<T>())))  // 使用noexcept运算符
        : data_(other.data_), size_(other.size_) 
    {
        other.data_ = nullptr;
        other.size_ = 0;
    }
    
    // swap：只有当T的swap是noexcept时，本函数才是noexcept
    void swap(Container& other) 
        noexcept(noexcept(swap(std::declval<T&>(), std::declval<T&>()))) 
    {
        using std::swap;
        swap(data_, other.data_);
        swap(size_, other.size_);
    }
};
```

### 6.2 实际应用示例
```cpp
#include <type_traits>

template<typename T>
void optimized_swap(T& a, T& b) 
    noexcept(noexcept(a.swap(b)) && std::is_nothrow_move_constructible_v<T>)
{
    // 如果T有noexcept的swap且可noexcept移动构造，使用优化版本
    T temp(std::move(a));
    a = std::move(b);
    b = std::move(temp);
}
```

## 7. 测试和调试noexcept

### 7.1 检测noexcept状态
```cpp
#include <type_traits>

class TestClass {
public:
    void noexcept_func() noexcept {}
    void may_throw_func() {}
};

void test_noexcept_detection() {
    // 使用type_traits检测noexcept
    static_assert(noexcept(TestClass().noexcept_func()), 
                  "Should be noexcept");
    
    static_assert(!noexcept(TestClass().may_throw_func()), 
                  "Should not be noexcept");
    
    // 运行时检测
    TestClass obj;
    bool is_noexcept = noexcept(obj.noexcept_func());
    std::cout << "Function is noexcept: " << is_noexcept << std::endl;
}
```

### 7.2 调试noexcept违规
```cpp
void debug_noexcept_violation() {
    auto problematic_func =  noexcept {
        try {
            // 调用可能抛出异常的函数
            potentially_throwing_operation();
        } catch (...) {
            // 在noexcept函数中捕获异常是允许的
            // 但不能让异常逃逸
            handle_error_gracefully();
            
            // 如果让异常逃逸，程序会终止
            // throw;  // 这会导致std::terminate()被调用
        }
    };
}
```

## 8. 现代C++最佳实践

### 8.1 结合现代特性
```cpp
class ModernNoexcept {
private:
    std::unique_ptr<Resource> resource_;
    std::vector<int> data_;
    
public:
    // 使用=default和noexcept结合
    ModernNoexcept(ModernNoexcept&&) noexcept = default;
    ModernNoexcept& operator=(ModernNoexcept&&) noexcept = default;
    
    // 使用nodiscard等属性
    [[nodiscard]] int getValue() const noexcept { 
        return value_; 
    }
    
    // 条件性noexcept与概念（C++20）
    template<typename T>
    requires std::is_nothrow_swappable_v<T>
    void safe_swap(T& a, T& b) noexcept {
        std::swap(a, b);
    }
    
private:
    int value_;
};
```

### 8.2 错误处理策略
```cpp
class ErrorHandling {
public:
    // 策略1：返回错误码（适合noexcept函数）
    std::error_code processData() noexcept {
        if (!validate()) {
            return std::make_error_code(std::errc::invalid_argument);
        }
        // 处理逻辑...
        return {};
    }
    
    // 策略2：使用std::optional（C++17）
    std::optional<Result> compute() noexcept {
        if (!preconditions_met()) {
            return std::nullopt;
        }
        return Result{/*...*/};
    }
    
    // 策略3：使用std::expected（C++23）
    // std::expected<Result, Error> calculate() noexcept;
};
```

## 9. 性能测试示例

### 9.1 基准测试对比
```cpp
#include <chrono>
#include <vector>

class BenchmarkDemo {
public:
    void noexcept_operation() noexcept {
        // 简单的数学运算
        result_ = value_ * 2 + 1;
    }
    
    void non_noexcept_operation() {
        // 相同的运算，但没有noexcept
        result_ = value_ * 2 + 1;
    }
    
    void run_benchmark() {
        constexpr size_t iterations = 1'000'000;
        
        auto start = std::chrono::high_resolution_clock::now();
        for (size_t i = 0; i < iterations; ++i) {
            noexcept_operation();
        }
        auto end = std::chrono::high_resolution_clock::now();
        
        auto noexcept_duration = end - start;
        
        start = std::chrono::high_resolution_clock::now();
        for (size_t i = 0; i < iterations; ++i) {
            non_noexcept_operation();
        }
        end = std::chrono::high_resolution_clock::now();
        
        auto non_noexcept_duration = end - start;
        
        std::cout << "Noexcept version: " 
                  << std::chrono::duration_cast<std::chrono::microseconds>(
                         noexcept_duration).count() << " μs\n";
        std::cout << "Non-noexcept version: " 
                  << std::chrono::duration_cast<std::chrono::microseconds>(
                         non_noexcept_duration).count() << " μs\n";
    }
    
private:
    int value_ = 42;
    int result_ = 0;
};
```

## 10. 总结与建议

### 10.1 关键要点总结

| 场景 | 建议 | 理由 |
|------|------|------|
| 移动操作 | **总是**使用`noexcept` | 标准库优化依赖于此 |
| 析构函数 | 隐式`noexcept`，无需显式声明 | 语言规则保证 |
| `swap`函数 | **尽量**使用`noexcept` | 影响整个调用链的性能 |
| 简单getter | 使用`noexcept` | 明显不会抛出异常 |
| 文件I/O/网络 | **不要**使用`noexcept` | 很可能抛出异常 |
| 虚函数 | 谨慎使用`noexcept` | 限制派生类的实现 |

### 10.2 最终建议

1. **正确性优先**：只在确定函数不会抛出异常时使用`noexcept`
2. **性能关键路径**：对移动操作、`swap`等关键函数积极使用`noexcept`
3. **接口设计**：将`noexcept`视为函数接口的重要部分
4. **测试验证**：确保`noexcept`承诺在函数的所有执行路径上都成立
5. **文档记录**：在复杂函数中记录使用`noexcept`的理由

`noexcept`是C++异常处理机制的重要进步，正确使用可以显著提升代码性能和可读性，但需要开发者谨慎负责地使用。