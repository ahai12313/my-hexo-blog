---
title: 'Item 31: Avoid default capture modes'
categories: Effective C++
date: 2025-12-06 22:41:52
tags:
priority: 31
---
# 条款31：避免使用默认捕获模式

## 摘要
C++ lambda表达式提供了两种默认捕获模式：`[&]`（按引用捕获）和`[=]`（按值捕获）。虽然这些语法便捷，但它们隐藏着严重的安全风险。**默认按引用捕获可能导致悬空引用，默认按值捕获则可能产生悬空指针（特别是`this`指针），并误导开发者认为闭包是自包含的**。本条款建议始终使用显式捕获，明确列出需要捕获的变量。

## 详细分析

### 1. 默认按引用捕获的问题：悬空引用

#### 问题场景
```cpp
using FilterContainer = std::vector<std::function<bool(int)>>;
FilterContainer filters;

void addDivisorFilter() {
    auto calc1 = computeSomeValue1();
    auto calc2 = computeSomeValue2();
    auto divisor = computeDivisor(calc1, calc2);  // 局部变量
    
    filters.emplace_back(
        int value { return value % divisor == 0; }  // 危险！捕获了divisor的引用
    );
    // divisor在此处被销毁，但lambda被存储在filters中
    // 后续使用此lambda时将访问已销毁的内存
}
```

#### 问题分析
- `divisor`是局部变量，在`addDivisorFilter()`返回时被销毁
- lambda通过默认引用捕获`[&]`获取了`divisor`的引用
- lambda被存储在容器`filters`中，生命周期超过`divisor`
- 后续调用lambda时，访问的是已销毁的`divisor`，导致**未定义行为**

#### 显式捕获的改进
```cpp
void addDivisorFilter() {
    auto calc1 = computeSomeValue1();
    auto calc2 = computeSomeValue2();
    auto divisor = computeDivisor(calc1, calc2);
    
    // 显式捕获，更清晰地展示依赖关系
    filters.emplace_back(
        int value { return value % divisor == 0; }
    );
    // 仍然有问题，但至少更容易看出问题所在
}
```
显式捕获使代码审查时更容易识别生命周期问题，但它本身并不解决问题。

### 2. 默认按值捕获的问题：悬空指针与误解

#### 问题1：对指针的按值捕获
```cpp
void foo() {
    auto ptr = std::make_unique<int>(42);
    
    auto lambda =  { 
        std::cout << *ptr << std::endl;  // 捕获的是指针本身，不是指针指向的数据
    };
    
    ptr.reset();  // 释放内存
    lambda();     // 未定义行为：访问已释放的内存
}
```
- `[=]`只复制指针本身，不复制指针指向的数据
- 如果外部代码释放了指针指向的内存，lambda内部仍持有悬空指针

#### 问题2：成员函数中的陷阱
```cpp
class Widget {
public:
    void addFilter() const {
        // 错误！实际捕获的是this指针，不是divisor
        filters.emplace_back(
            int value { return value % divisor == 0; }
        );
    }
    
private:
    int divisor = 5;
    static inline std::vector<std::function<bool(int)>> filters;
};
```

##### 编译器的视角
编译器实际上将代码视为：
```cpp
void Widget::addFilter() const {
    auto currentObjectPtr = this;  // 隐式捕获this指针
    
    filters.emplace_back(
        int value {
            return value % currentObjectPtr->divisor == 0;  // 通过指针访问成员
        }
    );
}
```

##### 实际问题
```cpp
void doSomeWork() {
    auto pw = std::make_unique<Widget>();
    pw->addFilter();  // 添加lambda，内部捕获了this指针
    // Widget在此处被销毁
    // filters中的lambda现在包含悬空的this指针
}
```

#### 问题3：静态变量的误导
```cpp
void addDivisorFilter() {
    static auto divisor = computeDivisor();  // 静态变量
    
    filters.emplace_back(
        int value {  // 实际上没有捕获任何东西！
            return value % divisor == 0;  // 直接引用静态变量
        }
    );
    
    ++divisor;  // 修改会影响所有已创建的lambda
}
```
- 静态变量不能被捕获
- lambda直接引用原始静态变量
- `[=]`给人"自包含副本"的错误印象
- 修改静态变量会影响所有lambda的行为

### 3. 安全替代方案

#### 方案1：显式捕获需要的变量
```cpp
void addDivisorFilter() {
    auto calc1 = computeSomeValue1();
    auto calc2 = computeSomeValue2();
    auto divisor = computeDivisor(calc1, calc2);
    
    // 按值捕获局部变量
    filters.emplace_back(
        int value {  // 显式按值捕获
            return value % divisor == 0;
        }
    );
}
```

#### 方案2：处理成员变量
```cpp
class Widget {
public:
    void addFilter() const {
        // 方法1：创建局部副本
        auto divisorCopy = divisor;
        filters.emplace_back(
            int value {  // 捕获副本
                return value % divisorCopy == 0;
            }
        );
        
        // 方法2：C++14广义lambda捕获
        filters.emplace_back(
            int value {  // 直接初始化副本
                return value % divisor == 0;
            }
        );
    }
    
private:
    int divisor = 5;
};
```

#### 方案3：立即使用的lambda可使用默认捕获
```cpp
template<typename C>
void processContainer(const C& container) {
    auto divisor = computeDivisor();
    
    // lambda被立即使用，不会存储，相对安全
    if (std::all_of(container.begin(), container.end(),
                   const auto& value {  // 可接受使用引用捕获
                       return value % divisor == 0;
                   })) {
        // ...
    }
}
```
注意：即使如此，显式捕获仍然更清晰、更安全。

### 4. 最佳实践总结

1. **优先使用显式捕获列表**
   ```cpp
   // 不推荐
   [&] { /* ... */ }
   [=] { /* ... */ }
   
   // 推荐
   [&x, &y] { /* ... */ }  // 明确按引用捕获哪些变量
   [x, y] { /* ... */ }    // 明确按值捕获哪些变量
   ```

2. **注意成员变量的捕获**
   - 在成员函数中，lambda不能直接捕获成员变量
   - 通过`[=]`捕获的是`this`指针，而非成员变量本身
   - 创建局部副本或使用广义lambda捕获

3. **理解静态变量的行为**
   - 静态变量不会被捕获
   - lambda总是引用原始静态变量
   - 静态变量的修改会影响所有lambda

4. **考虑使用C++14广义lambda捕获**
   ```cpp
   // C++14及以后可用
   auto lambda = [value = computeValue()] { 
       return value * 2; 
   };
   ```

5. **代码审查时特别关注**
   - lambda的生命周期是否超过捕获的变量
   - 按引用捕获的变量是否可能被修改
   - 成员函数中的lambda是否正确处理成员变量

## 结论

默认捕获模式`[&]`和`[=]`虽然提供编码便利，但引入了潜在的安全风险和维护难题。它们掩盖了lambda对外部变量的实际依赖关系，可能导致悬空引用、悬空指针和难以调试的行为。

**建议始终使用显式捕获**，明确列出lambda所需的所有外部变量。这种做法虽然需要更多输入，但显著提高了代码的：
- **安全性**：减少悬空引用和指针的风险
- **可读性**：清晰展示lambda的依赖关系
- **可维护性**：便于理解、修改和调试
- **可审查性**：更容易进行代码审查和静态分析

在C++中，明确表达意图通常是更好的选择，而lambda的捕获模式也不例外。通过显式列出捕获的变量，你可以编写出更安全、更清晰、更易于维护的代码。