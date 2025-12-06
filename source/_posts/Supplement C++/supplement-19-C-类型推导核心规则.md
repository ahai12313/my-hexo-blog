---
title: supplement 19 C++类型推导核心规则
categories: Supplement C++
date: 2025-12-06 22:43:37
tags:
priority: 19
---
# C++类型推导核心规则

## 一、模板类型推导（template type deduction）

### 1. ParamType是指针或引用（非万能引用）
- **规则**：忽略引用部分，匹配参数类型
- **例子**：
```cpp
template<typename T> void f(T& param);
int x = 27;
const int cx = x;
const int& rx = x;

f(x);   // T是int, param是int&
f(cx);  // T是const int, param是const int&
f(rx);  // T是const int, param是const int&
```

### 2. ParamType是万能引用（T&&）
- **规则**：左值推导为引用，右值推导为普通类型
- **例子**：
```cpp
template<typename T> void f(T&& param);
int x = 27;
const int cx = x;
const int& rx = x;

f(x);   // x是左值，T是int&, param是int&
f(cx);  // cx是左值，T是const int&, param是const int&
f(27);  // 27是右值，T是int, param是int&&
```

### 3. ParamType既非指针也非引用（按值传递）
- **规则**：忽略引用、const、volatile
- **例子**：
```cpp
template<typename T> void f(T param);
int x = 27;
const int cx = x;
const int& rx = x;

f(x);   // T是int, param是int
f(cx);  // T是int, param是int
f(rx);  // T是int, param是int
```

## 二、auto类型推导规则

### 基本规则
```cpp
auto x = 27;        // int
auto& rx = x;       // int&
const auto& crx = x;// const int&
auto&& uref = x;    // int& (万能引用，左值)
auto&& rref = 27;   // int&& (万能引用，右值)
```

### 特殊规则
- **数组和函数不会退化**（与模板推导不同）：
```cpp
auto arr = {1, 2, 3};  // std::initializer_list<int>
```

## 三、decltype类型推导规则

### 1. 对变量名
```cpp
int x = 0;
const int& rx = x;
decltype(x) a;     // int
decltype(rx) b = x;// const int&，必须初始化
```

### 2. 对表达式
```cpp
int x = 0;
decltype((x)) y = x;  // int&，因为(x)是表达式
decltype(x++) z;      // int，x++返回右值
```

## 四、引用折叠规则（reference collapsing）

### 规则
- `T& &` → `T&`
- `T& &&` → `T&`
- `T&& &` → `T&`
- `T&& &&` → `T&&`

### 应用
```cpp
template<typename T>
void f(T&& param) {  // 万能引用
    // 根据实参类型，T会被推导为引用或非引用
}
```

## 五、类型推导优先级（从高到低）

1. **万能引用推导**（最精确，保持值类别）
2. **引用/指针推导**（保持const属性）
3. **值传递推导**（最宽松，忽略const/引用）

## 六、核心记忆要点

| 推导场景 | 忽略引用 | 忽略const | 数组退化 | 函数退化 |
|---------|---------|----------|---------|---------|
| 模板（值传递） | ✓ | ✓ | ✓ | ✓ |
| 模板（引用传递）| ✗ | ✗ | ✗ | ✗ |
| auto | 同模板 | 同模板 | ✗ | ✗ |
| decltype | ✗ | ✗ | ✗ | ✗ |

## 七、实际编程中的规则总结

1. **auto推导 ≈ 模板推导**，除了对`{}`的处理
2. **decltype(变量名)** 给出变量的声明类型
3. **decltype(表达式)** 给出表达式的值类别和类型
4. **万能引用** 根据实参的左/右值性推导引用类型
5. **值传递** 会忽略顶层const和引用

## 八、一句话总结

**C++类型推导的核心是：匹配模式，然后调整。先匹配ParamType的模式，再根据值类别、const属性等进行调整，auto遵循模板规则，decltype看表达式，万能引用特殊处理。**