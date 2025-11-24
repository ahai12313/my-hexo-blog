---
title: 'Item 18: Use std::unique_ptr for exclusive-ownership resource management'
categories: Effective C++
date: 2025-11-24 22:23:21
tags:
priority: 18
---
# Item 18：优先使用 std::unique_ptr

## 1. 概述

`std::unique_ptr` 是 C++11 引入的智能指针，代表了**独占所有权**语义。它是现代 C++ 资源管理的基石，提供了自动内存管理的同时，保持了与原始指针相近的性能。

### 1.1 核心特性
- **独占所有权**：同一时间只有一个 `unique_ptr` 拥有资源
- **零开销抽象**：运行时性能与原始指针相当
- **自动生命周期管理**：离开作用域时自动释放资源
- **仅可移动**：支持移动语义，禁止拷贝操作

## 2. 基本用法

### 2.1 创建和基本操作
```cpp
#include <memory>

// 创建 unique_ptr
std::unique_ptr<int> ptr1 = std::make_unique<int>(42);  // C++14 推荐
std::unique_ptr<int> ptr2(new int(100));               // 传统方式

// 解引用操作
*ptr1 = 50;                    // 修改指向的值
int value = *ptr2;             // 获取值

// 成员访问
class Widget {
public:
    void process() { /* ... */ }
};
std::unique_ptr<Widget> widget = std::make_unique<Widget>();
widget->process();             // 通过指针调用成员函数

// 检查是否为空
if (ptr1) {                    // 布尔上下文检查
    std::cout << "ptr1 指向: " << *ptr1 << std::endl;
}

// 释放所有权
int* raw_ptr = ptr1.release();  // ptr1 变为空，返回原始指针
```

### 2.2 所有权转移
```cpp
std::unique_ptr<int> source = std::make_unique<int>(42);

// 移动语义转移所有权
std::unique_ptr<int> target = std::move(source);

// 此时 source 为空，target 拥有资源
assert(source == nullptr);
assert(*target == 42);

// 禁止拷贝操作
// std::unique_ptr<int> copy = source;  // 编译错误！
```

## 3. 性能优势

### 3.1 零开销设计
```cpp
// 性能对比测试
void rawPointerPerformance() {
    int* ptr = new int(42);
    *ptr = 100;           // 解引用
    int value = *ptr;     // 读取
    delete ptr;           // 手动释放
}

void uniquePtrPerformance() {
    auto ptr = std::make_unique<int>(42);
    *ptr = 100;           // 相同的机器指令
    int value = *ptr;     // 相同的机器指令
}                         // 自动释放，无额外开销
```

**性能特征**：
- **大小相同**：`sizeof(unique_ptr<T>) == sizeof(T*)`
- **操作等价**：解引用生成相同的汇编指令
- **构造/析构微小开销**：只在生命周期端点有少量额外操作

### 3.2 内存布局对比
```cpp
struct Data {
    int x, y, z;
};

// 内存布局完全相同
Data* raw_ptr;                    // 8字节（64位系统）
std::unique_ptr<Data> smart_ptr; // 8字节（64位系统）
```

## 4. 工厂模式应用

### 4.1 多态工厂示例
```cpp
#include <memory>
#include <vector>

// 投资类层次结构
class Investment {
public:
    virtual ~Investment() = default;  // 关键：虚析构函数
    virtual double calculateReturn() const = 0;
    virtual const char* name() const = 0;
};

class Stock : public Investment {
public:
    double calculateReturn() const override { return 0.15; }
    const char* name() const override { return "Stock"; }
};

class Bond : public Investment {
public:
    double calculateReturn() const override { return 0.05; }
    const char* name() const override { return "Bond"; }
};

class RealEstate : public Investment {
public:
    double calculateReturn() const override { return 0.08; }
    const char* name() const override { return "Real Estate"; }
};

// 工厂函数返回 unique_ptr
std::unique_ptr<Investment> createInvestment(const std::string& type) {
    if (type == "stock") return std::make_unique<Stock>();
    if (type == "bond") return std::make_unique<Bond>();
    if (type == "real_estate") return std::make_unique<RealEstate>();
    throw std::invalid_argument("Unknown investment type: " + type);
}
```

### 4.2 异常安全保证
```cpp
void processInvestment() {
    try {
        // 即使抛出异常，资源也会正确释放
        auto investment = createInvestment("stock");
        
        // 可能抛出异常的操作
        processComplexCalculation(investment.get());
        
        // 如果这里抛出异常，investment 仍会正确析构
        throw std::runtime_error("Something went wrong");
        
    } catch (const std::exception& e) {
        std::cout << "异常处理中，资源已自动清理: " << e.what() << std::endl;
    }
    // investment 自动析构，无需手动 delete
}
```

## 5. 自定义删除器

### 5.1 基本自定义删除器
```cpp
// 文件资源管理
auto fileDeleter = FILE* file {
    if (file) {
        std::cout << "关闭文件: " << file << std::endl;
        fclose(file);
    }
};

void fileExample() {
    // 使用自定义删除器管理文件
    std::unique_ptr<FILE, decltype(fileDeleter)> 
        file(fopen("data.txt", "r"), fileDeleter);
    
    if (file) {
        char buffer[100];
        fgets(buffer, 100, file.get());
        std::cout << "读取: " << buffer << std::endl;
    }
    // 文件自动关闭
}

// 网络连接管理
struct SocketDeleter {
    void operator()(int* socket) const {
        std::cout << "关闭 socket: " << *socket << std::endl;
        close(*socket);  // 假设 close() 函数存在
        delete socket;
    }
};

void networkExample() {
    std::unique_ptr<int, SocketDeleter> socket(new int(createSocket()));
    // 使用 socket...
    // 自动调用 SocketDeleter 清理
}
```

### 5.2 带日志的删除器（生产级）
```cpp
class DatabaseConnection {
public:
    DatabaseConnection(const std::string& connectionString) 
        : connectionString_(connectionString) {
        std::cout << "连接数据库: " << connectionString_ << std::endl;
    }
    
    void execute(const std::string& query) {
        std::cout << "执行查询: " << query << std::endl;
    }
    
private:
    std::string connectionString_;
};

auto createLoggingDeleter(const std::string& component) {
    return DatabaseConnection* conn {
        std::cout << "[" << component << "] 清理数据库连接" << std::endl;
        delete conn;
    };
}

void productionExample() {
    auto deleter = createLoggingDeleter("DataProcessor");
    std::unique_ptr<DatabaseConnection, decltype(deleter)> 
        db(new DatabaseConnection("server=localhost;db=test"), deleter);
    
    db->execute("SELECT * FROM users");
    // 自动记录日志并清理
}
```

## 6. 性能优化技巧

### 6.1 删除器类型优化
```cpp
// 方案1：无状态 lambda（最优）
auto statelessDeleter = int* p { 
    std::cout << "删除指针" << std::endl;
    delete p; 
};
std::unique_ptr<int, decltype(statelessDeleter)> p1(new int, statelessDeleter);
static_assert(sizeof(p1) == sizeof(int*), "大小与原始指针相同");

// 方案2：函数指针（大小增加）
void functionDeleter(int* p) { delete p; }
std::unique_ptr<int, void(*)(int*)> p2(new int, functionDeleter);
static_assert(sizeof(p2) > sizeof(int*), "包含函数指针");

// 方案3：有状态函数对象（大小可能更大）
struct StatefulDeleter {
    std::string log_message;  // 状态数据
    void operator()(int* p) { 
        std::cout << log_message << std::endl;
        delete p; 
    }
};
std::unique_ptr<int, StatefulDeleter> p3(new int, StatefulDeleter{"删除记录"});
static_assert(sizeof(p3) > sizeof(int*), "包含状态数据");
```

### 6.2 使用 std::make_unique（C++14）
```cpp
// 更安全、更高效的创建方式
class Widget {
public:
    Widget(int a, double b, const std::string& c) 
        : a_(a), b_(b), c_(c) {}
private:
    int a_;
    double b_;
    std::string c_;
};

void safeCreation() {
    // 推荐：异常安全，更高效
    auto widget1 = std::make_unique<Widget>(1, 2.0, "hello");
    
    // 不推荐：可能内存泄漏
    auto widget2 = std::unique_ptr<Widget>(
        new Widget(1, 2.0, "hello"));  // 可能泄漏
}

// make_unique 的优势：
// 1. 异常安全：不会在 new 和 unique_ptr 构造之间泄漏
// 2. 代码简洁：不需要重复类型
// 3. 更高效：一次内存分配（对象+控制块）
```

## 7. 容器中的使用

### 7.1 在标准容器中存储
```cpp
void containerExample() {
    std::vector<std::unique_ptr<Investment>> portfolio;
    
    // 使用移动语义添加到容器
    portfolio.push_back(std::make_unique<Stock>());
    portfolio.push_back(std::make_unique<Bond>());
    portfolio.push_back(std::make_unique<RealEstate>());
    
    // 使用 emplace_back 更高效
    portfolio.emplace_back(std::make_unique<Stock>());
    
    // 遍历和操作
    for (const auto& investment : portfolio) {
        std::cout << investment->name() 
                  << " 收益率: " << investment->calculateReturn() 
                  << std::endl;
    }
    
    // 自动清理所有资源
}

// 映射表示例
void mapExample() {
    std::map<std::string, std::unique_ptr<Investment>> investmentMap;
    
    // 插入到映射表
    investmentMap["tech"] = std::make_unique<Stock>();
    investmentMap["safe"] = std::make_unique<Bond>();
    
    // 查找和访问
    if (auto it = investmentMap.find("tech"); it != investmentMap.end()) {
        it->second->calculateReturn();
    }
}
```

### 7.2 所有权转移模式
```cpp
class InvestmentPortfolio {
private:
    std::vector<std::unique_ptr<Investment>> investments_;
    
public:
    // 接管所有权
    void addInvestment(std::unique_ptr<Investment> investment) {
        investments_.push_back(std::move(investment));
    }
    
    // 转移所有权出去
    std::unique_ptr<Investment> removeInvestment(size_t index) {
        if (index >= investments_.size()) {
            return nullptr;
        }
        
        auto it = investments_.begin() + index;
        std::unique_ptr<Investment> result = std::move(*it);
        investments_.erase(it);
        return result;
    }
    
    // 计算总收益
    double totalReturn() const {
        double total = 0.0;
        for (const auto& inv : investments_) {
            total += inv->calculateReturn();
        }
        return total;
    }
};

void portfolioDemo() {
    InvestmentPortfolio portfolio;
    
    // 添加投资
    portfolio.addInvestment(std::make_unique<Stock>());
    portfolio.addInvestment(std::make_unique<Bond>());
    
    // 转移所有权
    auto movedInvestment = portfolio.removeInvestment(0);
    if (movedInvestment) {
        std::cout << "移出的投资: " << movedInvestment->name() << std::endl;
    }
}
```

## 8. 高级特性

### 8.1 数组特化版本
```cpp
void arrayExample() {
    // 管理动态数组
    std::unique_ptr<int[]> arr = std::make_unique<int[]>(100);
    
    // 使用数组操作符
    for (int i = 0; i < 100; ++i) {
        arr[i] = i * i;
    }
    
    // 自动调用 delete[]
    // 但通常更推荐使用 std::vector
}

// 与 std::vector 对比
void vectorComparison() {
    // unique_ptr 数组版本
    std::unique_ptr<int[]> arr_ptr = std::make_unique<int[]>(100);
    
    // std::vector（通常更优）
    std::vector<int> vec(100);
    
    // vector 优势：
    // - 更丰富的API（size(), push_back(), 等）
    // - 自动调整大小
    // - 更好的异常安全
    // - 标准算法支持
}
```

### 8.2 与 std::shared_ptr 的转换
```cpp
void conversionExample() {
    // 工厂返回 unique_ptr
    auto unique_inv = createInvestment("stock");
    
    // 需要共享所有权时轻松转换
    std::shared_ptr<Investment> shared_inv = std::move(unique_inv);
    
    // 现在可以多个地方共享
    useSharedInvestment(shared_inv);
    anotherUse(shared_inv);
    
    // unique_inv 现在为空
    assert(!unique_inv);
}

// 工厂函数的最佳实践
template<typename... Args>
auto createInvestmentSmart(Args&&... args) {
    // 返回 unique_ptr，让调用者决定是否需要共享
    return std::make_unique<Stock>(std::forward<Args>(args)...);
}

void clientFlexibility() {
    // 调用者可以根据需要选择所有权语义
    auto unique = createInvestmentSmart();  // 独占所有权
    auto shared = std::shared_ptr<Investment>(createInvestmentSmart());  // 共享所有权
    
    // 或者直接创建 shared_ptr
    auto direct_shared = std::make_shared<Stock>();  // 如果需要共享所有权
}
```

## 9. 实际应用模式

### 9.1 Pimpl 惯用法
```cpp
// widget.h
class Widget {
public:
    Widget();
    ~Widget();  // 必须在实现文件中定义，因为 Impl 是不完整类型
    
    void process();
    Widget(Widget&& other) noexcept;
    Widget& operator=(Widget&& other) noexcept;
    
    // 禁止拷贝
    Widget(const Widget&) = delete;
    Widget& operator=(const Widget&) = delete;
    
private:
    struct Impl;
    std::unique_ptr<Impl> pImpl_;
};

// widget.cpp
#include "widget.h"

struct Widget::Impl {
    int data;
    std::string name;
    std::vector<double> values;
    
    void complexOperation() { /* 复杂实现 */ }
};

Widget::Widget() : pImpl_(std::make_unique<Impl>()) {}

// 必须在 Impl 定义后定义析构函数
Widget::~Widget() = default;

// 移动操作
Widget::Widget(Widget&& other) noexcept = default;
Widget& Widget::operator=(Widget&& other) noexcept = default;

void Widget::process() {
    pImpl_->complexOperation();
}
```

### 9.2 资源管理类模板
```cpp
template<typename T>
class ResourceHandle {
public:
    // 构造函数获取资源
    explicit ResourceHandle(T* resource) : resource_(resource) {}
    
    // 禁止拷贝
    ResourceHandle(const ResourceHandle&) = delete;
    ResourceHandle& operator=(const ResourceHandle&) = delete;
    
    // 允许移动
    ResourceHandle(ResourceHandle&&) = default;
    ResourceHandle& operator=(ResourceHandle&&) = default;
    
    // 访问资源
    T* get() const { return resource_.get(); }
    T* operator->() const { return resource_.get(); }
    T& operator*() const { return *resource_; }
    
    // 释放所有权
    T* release() { return resource_.release(); }
    
    // 重置资源
    void reset(T* new_resource = nullptr) { resource_.reset(new_resource); }
    
private:
    std::unique_ptr<T> resource_;
};

// 使用示例
void resourceExample() {
    ResourceHandle<FILE> file(fopen("data.txt", "r"));
    if (file) {
        // 使用文件...
    }
    // 自动关闭文件
}
```

## 10. 错误处理和调试

### 10.1 常见错误模式
```cpp
void commonMistakes() {
    // 错误1：尝试拷贝
    auto ptr1 = std::make_unique<int>(42);
    // auto ptr2 = ptr1;  // 编译错误！
    
    // 正确：移动所有权
    auto ptr3 = std::move(ptr1);
    
    // 错误2：悬空指针
    int* raw_ptr = nullptr;
    {
        auto temp = std::make_unique<int>(100);
        raw_ptr = temp.get();  // 危险：temp 析构后 raw_ptr 悬空
    }
    // *raw_ptr = 50;  // 未定义行为！
    
    // 错误3：错误的所有权管理
    auto ptr4 = std::make_unique<int>(200);
    // someFunctionThatTakesOwnership(ptr4.get());  // 错误！不转移所有权
    someFunctionThatTakesOwnership(ptr4.release());  // 正确
}
```

### 10.2 调试技巧
```cpp
// 带调试信息的删除器
auto createDebugDeleter(const char* resource_name) {
    return auto* ptr {
        std::cout << "调试: 释放 " << resource_name 
                  << " 地址: " << ptr << std::endl;
        delete ptr;
    };
}

void debugExample() {
    auto debug_deleter = createDebugDeleter("调试资源");
    std::unique_ptr<int, decltype(debug_deleter)> 
        debug_ptr(new int(42), debug_deleter);
    
    // 使用资源...
    *debug_ptr = 100;
    
}  // 输出: "调试: 释放 调试资源 地址: 0x..."
```

## 11. 性能基准测试

### 11.1 性能对比测试
```cpp
#include <chrono>
#include <vector>

void benchmark() {
    const size_t iterations = 1000000;
    
    // 原始指针测试
    auto start1 = std::chrono::high_resolution_clock::now();
    for (size_t i = 0; i < iterations; ++i) {
        int* ptr = new int(i);
        *ptr += 1;
        delete ptr;
    }
    auto end1 = std::chrono::high_resolution_clock::now();
    
    // unique_ptr 测试
    auto start2 = std::chrono::high_resolution_clock::now();
    for (size_t i = 0; i < iterations; ++i) {
        auto ptr = std::make_unique<int>(i);
        *ptr += 1;
    }
    auto end2 = std::chrono::high_resolution_clock::now();
    
    auto duration1 = std::chrono::duration_cast<std::chrono::microseconds>(end1 - start1);
    auto duration2 = std::chrono::duration_cast<std::chrono::microseconds>(end2 - start2);
    
    std::cout << "原始指针: " << duration1.count() << "μs\n";
    std::cout << "unique_ptr: " << duration2.count() << "μs\n";
    std::cout << "开销: " << (duration2.count() - duration1.count()) * 100.0 / duration1.count() << "%\n";
}
```

## 12. 最佳实践总结

### 12.1 使用场景决策表
| 场景 | 推荐方案 | 理由 |
|------|----------|------|
| 独占资源管理 | `std::unique_ptr` | 语义明确，性能最优 |
| 共享资源管理 | `std::shared_ptr` | 需要共享所有权时 |
| 观察资源（不拥有） | 原始指针或引用 | 避免不必要的所有权 |
| 数组管理 | `std::vector`（通常） | 更丰富的功能 |
| C API 集成 | `std::unique_ptr` 带自定义删除器 | 自动资源管理 |

### 12.2 代码审查清单
- [ ] 是否使用了 `std::make_unique` 而不是 `new`？
- [ ] 移动操作是否正确使用了 `std::move`？
- [ ] 是否避免了不必要的所有权共享？
- [ ] 自定义删除器是否无状态（如可能）？
- [ ] 多态基类是否有虚析构函数？
- [ ] 是否正确处理了异常安全？

### 12.3 性能优化检查点
1. **优先使用 `std::make_unique`**：异常安全，更高效
2. **选择无状态删除器**：保持小尺寸
3. **避免过早优化**：`unique_ptr` 开销通常可忽略
4. **使用移动语义**：避免不必要的资源管理开销

## 13. 结论

`std::unique_ptr` 是现代 C++ 资源管理的核心工具，它：
- 提供了**零开销**的自动内存管理
- 明确了**独占所有权**语义
- 支持**灵活的定制**（自定义删除器）
- 与 C++ 生态系统**完美集成**

**遵循的原则**：
- 默认使用 `std::unique_ptr` 管理独占资源
- 使用 `std::make_unique` 创建对象
- 通过移动语义转移所有权
- 为多态基类声明虚析构函数
- 根据需要选择合适的删除器策略

掌握 `std::unique_ptr` 是编写现代、安全、高效 C++ 代码的关键技能。