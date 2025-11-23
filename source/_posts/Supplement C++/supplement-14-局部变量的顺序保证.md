---
title: supplement 14 局部变量的顺序保证
categories: Supplement C++
date: 2025-11-23 20:27:30
tags:
priority: 13
---

## 什么是局部变量的顺序保证？

在C++中，对于**同一个线程内**的局部变量访问，编译器必须保持**明显的执行顺序**。

```cpp
void example() {
    // 局部变量 - 有严格的顺序保证
    auto a = foo();     // 语句1
    auto b = bar(a);    // 语句2 - 依赖a，必须在语句1之后执行
    use(b);             // 语句3 - 依赖b，必须在语句2之后执行
}
```

编译器**不能**重排序这些语句，因为存在**数据依赖**。

## 对比：成员变量没有这种保证

```cpp
class Example {
    int x, y;
public:
    void bad_reordering() {
        x = 1;          // 语句A - 可能被重排
        y = 2;          // 语句B - 可能被重排到语句A之前
    }
    
    void good_ordering() {
        auto local_x = 1;  // 语句1 - 局部变量
        auto local_y = 2;   // 语句2 - 可能被重排到语句1之前？不，有依赖时不能
        x = local_x;       // 语句3
        y = local_y;       // 语句4
    }
};
```

## 具体到我们的缓存例子

### 原始代码（安全）
```cpp
auto newObj = std::make_unique<ExpensiveObject>(compute());  // 1. 构造局部变量
cached_.store(newObj.get(), std::memory_order_release);       // 2. 使用局部变量
cacheHolder_ = std::move(newObj);                            // 3. 转移所有权
```

**顺序保证的来源**：
1. **语句2依赖语句1**：`newObj.get()` 必须在 `newObj` 构造完成后才能调用
2. **语句3依赖语句1**：`std::move(newObj)` 需要 `newObj` 存在
3. **编译器必须遵守这个顺序**：因为存在明确的数据依赖关系

### 你的写法（不安全）
```cpp
cacheHolder_ = std::make_unique<ExpensiveObject>(compute());  // 1. 成员赋值
cached_.store(cacheHolder_.get(), std::memory_order_release); // 2. 成员使用
```

**缺少顺序保证的原因**：
1. **两个语句都操作成员变量**，没有局部变量作为"粘合剂"
2. **没有明显的数据依赖**：语句2不"依赖"语句1的结果
3. **编译器可以重排序**：只要单线程行为不变

## 通过代码演示理解

让我写个简单的demo来展示这种区别：

```cpp
#include <iostream>

struct Observer {
    static void log(const char* msg) {
        std::cout << msg << std::endl;
    }
};

struct TestObject {
    TestObject() { Observer::log("TestObject构造完成"); }
    void use() { Observer::log("TestObject被使用"); }
};

class WithLocalVar {
public:
    void safe_publication() {
        Observer::log("=== 使用局部变量 ===");
        
        // 使用局部变量 - 有顺序保证
        auto localObj = TestObject();  // 构造必须完成
        member_ = &localObj;           // 然后才能取地址
        member_->use();                // 然后才能使用
    }
    
private:
    TestObject* member_ = nullptr;
};

class WithoutLocalVar {
public:
    void unsafe_publication() {
        Observer::log("=== 直接使用成员变量 ===");
        
        // 直接使用成员变量 - 可能被重排序
        // 理论上编译器可以：
        // 1. 先分配内存
        // 2. 将指针赋给member_  
        // 3. 然后才构造对象
        // （但实际编译器通常不会这么激进）
        member_.reset(new TestObject());
        member_->use();
    }
    
private:
    std::unique_ptr<TestObject> member_;
};

int main() {
    WithLocalVar demo1;
    demo1.safe_publication();
    
    WithoutLocalVar demo2;  
    demo2.unsafe_publication();
    
    return 0;
}
```

## 数据依赖的具体类型

局部变量提供的顺序保证主要来自几种数据依赖：

### 1. **写后读依赖**
```cpp
int a = 10;     // 写a
int b = a + 1;  // 读a - 依赖前面的写操作
```

### 2. **写后写依赖**  
```cpp
int a = 10;     // 写a
a = 20;         // 再次写a - 依赖前面的写操作（顺序重要）
```

### 3. **读后写依赖**
```cpp
int b = a + 1;  // 读a
a = 30;         // 写a - 与前面的读操作顺序重要
```

## 在我们的缓存代码中

原始代码利用了**写后读依赖**：

```cpp
// 明确的依赖链：
auto newObj = std::make_unique<ExpensiveObject>(compute());  // 写newObj
cached_.store(newObj.get(), std::memory_order_release);       // 读newObj（依赖！）
```

你的写法缺少这种依赖：

```cpp
cacheHolder_ = std::make_unique<ExpensiveObject>(compute());  // 写cacheHolder_
cached_.store(cacheHolder_.get(), std::memory_order_release); // 读cacheHolder_
```

虽然看起来也有依赖，但这是**成员变量**的依赖，不是**局部变量**的依赖，约束力较弱。

## 为什么局部变量的约束更强？

### 局部变量的约束：
- **编译器可见的依赖关系**
- **语言标准强制保证的顺序**  
- **优化受到严格限制**

### 成员变量的约束：
- **可能被其他线程访问**（编译器不知道我们的锁保护）
- **优化空间更大**
- **重排序限制较少**

## 实际验证

你可以通过查看汇编代码来验证：

```cpp
// 测试函数1：使用局部变量
void test_local() {
    auto local = create_object();
    publish(local.get());
}

// 测试函数2：直接使用成员  
void test_member() {
    member_ = create_object();
    publish(member_.get());
}
```

编译后查看汇编，会发现`test_local`中的顺序更加固定。

## 总结

"局部变量提供的顺序保证"指的是：

1. **数据依赖强制顺序**：对局部变量的使用必须在其初始化之后
2. **编译器必须遵守**：这种依赖关系限制了重排序的可能性  
3. **比成员变量更强的保证**：局部变量有更严格的顺序约束
4. **语言规则保障**：C++标准明确要求保持这种顺序

在并发编程中，我们利用局部变量的这种特性来建立"happens-before"关系，确保：
- **对象完全构造** → **然后发布指针**
- **其他线程看到指针** → **一定能看到完全构造的对象**

这就是为什么原始代码使用局部变量是安全的关键原因。