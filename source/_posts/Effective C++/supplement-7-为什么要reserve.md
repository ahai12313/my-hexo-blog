---
title: supplement 7 为什么要reserve
date: 2025-11-20 22:08:31
tags:
categories: Effective C++
---
C++ 中使用 `reserve()` 的主要原因是**性能优化**。让我详细解释为什么要使用它：

## 核心原因：避免重复的内存分配

### 1. 向量（vector）的自然增长方式
当你不使用 `reserve()` 时，向量的工作方式是这样的：

```cpp
std::vector<int> vec;
for (int i = 0; i < 1000; ++i) {
    vec.push_back(i); // 可能触发多次重新分配
}
```

向量内部需要多次重新分配内存：
- 初始容量可能是 0
- 当加入第1个元素时，分配容量 1
- 当加入第2个元素时，重新分配容量 2
- 当加入第3个元素时，重新分配容量 4
- 当加入第5个元素时，重新分配容量 8
- ...以此类推（通常是翻倍增长）

### 2. 重新分配的代价很高
每次重新分配都需要：
- **分配新内存**：在堆上找一块更大的连续内存
- **复制元素**：将旧内存中的所有元素复制到新内存
- **销毁旧内存**：释放原来的内存块
- **更新迭代器/指针/引用**：所有指向原元素的指针、引用、迭代器都失效

## 性能对比示例

```cpp
#include <vector>
#include <chrono>
#include <iostream>

void without_reserve() {
    std::vector<int> vec;
    auto start = std::chrono::high_resolution_clock::now();
    
    for (int i = 0; i < 1000000; ++i) {
        vec.push_back(i); // 多次重新分配！
    }
    
    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    std::cout << "不使用 reserve: " << duration.count() << " ms" << std::endl;
}

void with_reserve() {
    std::vector<int> vec;
    vec.reserve(1000000); // 一次性分配足够内存
    auto start = std::chrono::high_resolution_clock::now();
    
    for (int i = 0; i < 1000000; ++i) {
        vec.push_back(i); // 无重新分配！
    }
    
    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    std::cout << "使用 reserve:    " << duration.count() << " ms" << std::endl;
}
```

**典型结果：**
```
不使用 reserve: 15-25 ms
使用 reserve:    5-8 ms
```

## 其他重要原因

### 3. 保持引用和迭代器的有效性
```cpp
std::vector<int> vec = {1, 2, 3};
int& ref = vec[0];  // 引用第一个元素

vec.push_back(4);   // 可能触发重新分配
// ref 现在可能失效！可能指向已释放的内存

// 但如果预先 reserve 足够空间：
vec.reserve(10);    // 预先分配足够空间
int& safe_ref = vec[0];
vec.push_back(4);   // 不会重新分配，ref 保持有效
```

### 4. 可预测的内存使用
```cpp
// 你知道需要处理 10000 个元素
std::vector<Data> dataset;
dataset.reserve(10000); // 一次性分配所需内存

// 避免内存碎片化，提高缓存局部性
```

## 实际应用场景

### 场景1：读取文件
```cpp
std::vector<std::string> lines;
lines.reserve(estimate_line_count(filename)); // 预估行数
// 然后读取文件，避免多次重新分配
```

### 场景2：处理已知数量的数据
```cpp
void process_users(const std::vector<User>& users) {
    std::vector<Result> results;
    results.reserve(users.size()); // 结果数量与用户数量相同
    
    for (const auto& user : users) {
        results.push_back(process_user(user));
    }
}
```

### 场景3：字符串拼接
```cpp
std::string build_sql_query(const QueryParams& params) {
    std::string query;
    query.reserve(256); // 预估SQL查询的大致长度
    
    query += "SELECT * FROM table WHERE ";
    query += "id = " + std::to_string(params.id);
    // ... 更多拼接操作
    
    return query;
}
```

## 什么时候不需要使用 reserve？

1. **元素数量未知**：如果你完全不知道会有多少元素，reserve 可能没有帮助
2. **小规模数据**：对于几十个元素，性能差异可以忽略
3. **一次性构造**：使用初始化列表或范围构造函数时
   ```cpp
   std::vector<int> vec = {1, 2, 3, 4, 5}; // 不需要 reserve
   ```

## 总结

使用 `reserve()` 的主要理由是：
- **性能优化**：避免昂贵的重复内存分配
- **稳定性**：保持指针/引用/迭代器的有效性  
- **内存效率**：减少内存碎片，提高缓存友好性

这是一个"用空间换时间"的经典优化策略，在C++高性能编程中非常重要。