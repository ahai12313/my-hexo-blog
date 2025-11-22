---
title: 'Item 13: Prefer const_iterators to iterators'
date: 2025-11-22 18:56:41
tags:
categories: Effective C++
priority: 12
---
# Item 13：优先选用const_iterators而非iterators

## 1. 基本概念

### 1.1 const_iterators的作用
`const_iterator`是STL中的"指向const的指针"等价物，指向的值不能被修改。遵循"尽可能使用const"的原则，当不需要修改迭代器指向的内容时，应该使用`const_iterator`。

```cpp
std::vector<int> vec = {1, 2, 3, 4, 5};

// 不好的做法：使用iterator但不需要修改
std::vector<int>::iterator it = vec.begin();
*it = 10;  // 可以修改，但不应该被允许

// 好的做法：使用const_iterator
std::vector<int>::const_iterator cit = vec.cbegin();
// *cit = 10;  // ❌ 编译错误：不能修改const内容
```

## 2. C++98中的const_iterator问题

### 2.1 获取困难
在C++98中，从非const容器获取`const_iterator`很麻烦：

```cpp
// C++98代码：繁琐且容易出错
std::vector<int> values;
typedef std::vector<int>::iterator IterT;
typedef std::vector<int>::const_iterator ConstIterT;

// 需要强制转换来获取const_iterator
ConstIterT ci = std::find(
    static_cast<ConstIterT>(values.begin()),
    static_cast<ConstIterT>(values.end()), 
    1983
);

// 但insert需要iterator，需要再转换回去（可能不工作）
values.insert(static_cast<IterT>(ci), 1998);  // ❌ 可能编译失败
```

### 2.2 使用限制
C++98中，插入和删除操作只接受`iterator`，不接受`const_iterator`：

```cpp
std::vector<int> vec = {1, 2, 3};
std::vector<int>::const_iterator cit = vec.begin();

// C++98：编译错误
// vec.insert(cit, 0);  // ❌ const_iterator不能用于insert

// 必须使用iterator
std::vector<int>::iterator it = vec.begin();
vec.insert(it, 0);  // ✅
```

## 3. C++11的改进

### 3.1 方便的获取方式
C++11引入了`cbegin()`和`cend()`成员函数：

```cpp
std::vector<int> values = {1979, 1983, 1985};

// C++11：简单直观
auto it = std::find(values.cbegin(), values.cend(), 1983);
values.insert(it, 1998);  // ✅ const_iterator现在可以被接受
```

### 3.2 完整的const_iterator支持
C++11中，STL成员函数现在接受`const_iterator`：

| 操作 | C++98 | C++11 |
|------|-------|-------|
| 获取const_iterator | 困难 | `cbegin()/cend()` |
| insert/erase参数 | 只接受iterator | 接受const_iterator |
| 通用性 | 有限 | 完整支持 |

## 4. 通用代码中的const_iterator

### 4.1 泛型编程的最佳实践
对于最大化通用的库代码，应该使用非成员函数版本：

```cpp
// 通用模板函数
template<typename C, typename V>
void findAndInsert(C& container, const V& targetVal, const V& insertVal) {
    using std::cbegin;
    using std::cend;
    
    auto it = std::find(cbegin(container), cend(container), targetVal);
    container.insert(it, insertVal);
}
```

### 4.2 C++11的限制和C++14的解决方案
C++11标准遗漏了非成员函数的`cbegin/cend`等，C++14中修复：

```cpp
// C++14：完全支持
std::vector<int> vec = {1, 2, 3};
auto cit1 = std::cbegin(vec);  // ✅ C++14
auto cit2 = vec.cbegin();      // ✅ C++11
```

## 5. 为C++11实现缺失的非成员函数

### 5.1 自定义cbegin实现
如果在C++11中需要通用代码，可以自己实现：

```cpp
template <class C>
auto cbegin(const C& container) -> decltype(std::begin(container)) {
    return std::begin(container);  // 对const容器调用begin返回const_iterator
}

// 使用示例
template<typename C, typename V>
void genericFind(const C& container, const V& value) {
    auto it = std::find(cbegin(container), cend(container), value);
    // ... 处理结果
}
```

### 5.2 实现原理说明
```cpp
// 为什么这样工作：
std::vector<int> vec;
const std::vector<int>& const_vec = vec;

// std::begin(const_vec) 返回 const_iterator
// 因为begin()有const重载版本：
// iterator begin();
// const_iterator begin() const;
```

## 6. 完整的非成员函数家族

### 6.1 C++14中的完整集合
```cpp
// 非const版本
template<class C> auto begin(C& c) -> decltype(c.begin());
template<class C> auto end(C& c) -> decltype(c.end());

// const版本  
template<class C> auto cbegin(const C& c) -> decltype(std::begin(c));
template<class C> auto cend(const C& c) -> decltype(std::end(c));

// 反向迭代器
template<class C> auto rbegin(C& c) -> decltype(c.rbegin());
template<class C> auto rend(C& c) -> decltype(c.rend());
template<class C> auto crbegin(const C& c) -> decltype(std::rbegin(c));
template<class C> auto crend(const C& c) -> decltype(std::rend(c));
```

### 6.2 内置数组的特化支持
非成员函数也支持内置数组：

```cpp
int arr[] = {1, 2, 3, 4, 5};

// 对数组使用非成员函数
auto begin_it = std::begin(arr);    // 返回int*
auto cbegin_it = std::cbegin(arr);  // 返回const int*

// 等价于
int* begin_ptr = arr;
const int* cbegin_ptr = arr;
```

## 7. 实际应用示例

### 7.1 搜索而不修改
```cpp
template<typename Container, typename Value>
bool contains(const Container& c, const Value& v) {
    // 使用const_iterator，明确表示不修改容器
    return std::find(std::cbegin(c), std::cend(c), v) != std::cend(c);
}

void example() {
    std::vector<std::string> names = {"Alice", "Bob", "Charlie"};
    
    // 明确的const语义：搜索但不修改
    if (contains(names, "Bob")) {
        std::cout << "Found Bob!\n";
    }
}
```

### 7.2 通用算法封装
```cpp
// 安全的查找函数，返回const_iterator
template<typename Container, typename Value>
auto safeFind(const Container& c, const Value& v) 
    -> decltype(std::cbegin(c)) 
{
    return std::find(std::cbegin(c), std::cend(c), v);
}

void safeExample() {
    std::set<int> numbers = {1, 2, 3, 4, 5};
    
    auto it = safeFind(numbers, 3);
    if (it != std::cend(numbers)) {
        // it是const_iterator，防止意外修改
        // *it = 10;  // ❌ 编译错误
        std::cout << "Found: " << *it << std::endl;
    }
}
```

## 8. 性能考虑

### 8.1 零开销抽象
`const_iterator`和`iterator`在性能上没有区别：

```cpp
std::vector<int> vec(1000);

// 性能相同
for (auto it = vec.begin(); it != vec.end(); ++it) { /* 使用iterator */ }
for (auto it = vec.cbegin(); it != vec.cend(); ++it) { /* 使用const_iterator */ }
```

### 8.2 编译时优化
由于是编译时特性，使用`const_iterator`不会带来运行时开销，但能提供更好的类型安全。

## 9. 现代C++最佳实践

### 9.1 通用代码模板
```cpp
// 最大化通用的容器处理函数
template<typename Container>
void processContainer(const Container& c) {
    // 总是使用const版本，除非需要修改
    using std::cbegin;
    using std::cend;
    
    for (auto it = cbegin(c); it != cend(c); ++it) {
        // 只读访问 *it
    }
    
    // 或者使用range-based for（更简单）
    for (const auto& element : c) {
        // 只读访问element
    }
}
```

### 9.2 结合auto和const_iterator
```cpp
std::vector<std::string> data = getData();

// 好：明确的const语义
for (auto cit = data.cbegin(); cit != data.cend(); ++cit) {
    std::cout << *cit << std::endl;
}

// 更好：range-based for + const引用
for (const auto& item : data) {
    std::cout << item << std::endl;
}

// 需要修改时才使用非const
for (auto& item : data) {
    item = "modified";  // 可以修改
}
```

## 10. 特殊情况处理

### 10.1 需要修改时的正确做法
```cpp
std::vector<int> values = {1, 2, 3, 4, 5};

// 需要修改时：先查找，再获取可修改的iterator
auto const_it = std::find(values.cbegin(), values.cend(), 3);
if (const_it != values.cend()) {
    // 计算位置，获取可修改的iterator
    auto distance = std::distance(values.cbegin(), const_it);
    auto it = values.begin();
    std::advance(it, distance);
    
    *it = 30;  // 现在可以修改
}
```

### 10.2 容器适配器的特殊情况
```cpp
std::stack<int> s;
s.push(1);
s.push(2);
s.push(3);

// 栈没有iterator，需要使用底层容器
// 不推荐的做法：访问底层容器
// auto& underlying = s.c;  // 错误：底层容器通常是protected

// 推荐：如果需要迭代，选择支持迭代的容器
std::vector<int> vec = {1, 2, 3};
std::stack<int, std::vector<int>> stack_with_vec(vec);
// 但stack仍然不提供迭代器接口
```

## 11. 总结

### 11.1 关键要点
1. **优先使用const_iterator**：遵循"尽可能使用const"的原则
2. **C++11大幅改进**：`cbegin()/cend()`使const_iterator易于使用
3. **通用代码使用非成员函数**：`std::cbegin()`比成员函数版本更通用
4. **零开销抽象**：const_iterator提供安全性而无性能损失

### 11.2 现代C++代码风格
```cpp
// 现代C++最佳实践示例
template<typename T>
void modernCode(const std::vector<T>& data) {
    // 方法1：range-based for（推荐）
    for (const auto& item : data) {
        process(item);
    }
    
    // 方法2：算法+const_iterator
    auto result = std::find_if(std::cbegin(data), std::cend(data),
                              const T& item { return isValid(item); });
    
    // 方法3：通用非成员函数
    if (auto it = std::find(std::cbegin(data), std::cend(data), target);
        it != std::cend(data)) {
        // 找到目标
    }
}
```

### 11.3 需要记住的要点
- 优先选用const_iterators而非iterators
- 在最大化通用代码中，优先选用非成员函数版本的begin、end、rbegin等
- const_iterator提供编译时安全性，帮助捕获潜在的错误
- 现代C++使const_iterator的使用变得简单实用

通过遵循这些准则，可以编写出更安全、更清晰、更易于维护的C++代码。