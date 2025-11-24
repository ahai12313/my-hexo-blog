---
title: supplement 16 原始指针的六大问题
categories: Supplement C++
date: 2025-11-24 22:22:32
tags:
priority: 15
---

## 原始指针的六大问题

### 1. **无法区分单对象还是数组**
```cpp
Widget* ptr;  // 指向单个Widget还是Widget数组？无法从声明看出
```

### 2. **所有权不明确**
```cpp
void process(Widget* ptr) {
    // 调用后需要delete ptr吗？还是由调用者负责？
    // 完全依赖文档和约定，容易出错
}
```

### 3. **销毁方式未知**
```cpp
// 应该用哪种方式销毁？
delete ptr;          // 标准delete
customDeleter(ptr);  // 自定义删除器
free(ptr);          // C风格free
```

### 4. **delete形式不确定**
```cpp
// 是单个对象还是数组？
delete ptr;    // 单对象
delete[] ptr;  // 数组
// 用错会导致未定义行为
```

### 5. **异常安全难以保证**
```cpp
void riskyFunction() {
    Widget* ptr = new Widget;
    
    someOperation();  // 可能抛出异常
    anotherOp();      // 可能抛出异常
    
    delete ptr;  // 如果前面抛出异常，这里永远不会执行→内存泄漏
}
```

### 6. **悬空指针风险**
```cpp
Widget* ptr = new Widget;
delete ptr;      // 对象被销毁
// ptr现在成为悬空指针，但无法检测
ptr->doSomething();  // 未定义行为！
```

## 智能指针解决方案

### unique_ptr：独占所有权
```cpp
// 明确表达单对象所有权
std::unique_ptr<Widget> ptr1 = std::make_unique<Widget>();

// 明确表达数组所有权  
std::unique_ptr<Widget[]> ptr2 = std::make_unique<Widget[]>(10);

// 自定义删除器
auto deleter = File* f { fclose(f); };
std::unique_ptr<File, decltype(deleter)> filePtr(fopen("data.txt", "r"), deleter);
```

### shared_ptr：共享所有权
```cpp
auto ptr = std::make_shared<Widget>();  // 引用计数=1

{
    auto ptr2 = ptr;  // 引用计数=2
    // 使用对象...
}  // ptr2析构，引用计数=1

// ptr仍然有效，引用计数=1
```

### weak_ptr：避免循环引用
```cpp
class Observer {
    std::weak_ptr<Subject> subject;  // 不增加引用计数
    
    void observe(std::shared_ptr<Subject> s) {
        subject = s;  // 弱引用，避免循环引用
    }
};
```

## 现代C++最佳实践

### 优先使用make函数
```cpp
// 好：异常安全，更高效
auto ptr = std::make_unique<Widget>(arg1, arg2);
auto shared = std::make_shared<Widget>(arg1, arg2);

// 避免：可能泄漏
std::unique_ptr<Widget>(new Widget(arg1, arg2));
```

### 明确所有权语义
```cpp
// 工厂函数返回unique_ptr，明确转移所有权
std::unique_ptr<Widget> createWidget();

// 参数：只读访问传递const引用
void process(const Widget& widget);

// 参数：需要共享所有权传递shared_ptr
void share(std::shared_ptr<Widget> widget);

// 返回值：移动语义高效返回
std::unique_ptr<Widget> create() {
    auto widget = std::make_unique<Widget>();
    // ... 配置widget
    return widget;  // 移动语义，无开销
}
```

## 关键结论

1. **永远优先使用智能指针**而不是原始指针
2. **默认使用unique_ptr**，需要共享所有权时才用shared_ptr
3. **避免使用auto_ptr**，它已被unique_ptr取代
4. **使用make_unique/make_shared**来创建智能指针
5. **原始指针应仅用于观察**，不表示所有权

智能指针是现代C++资源管理的基石，它们几乎消除了手动内存管理的所有痛点，让代码更安全、更清晰、更易于维护。