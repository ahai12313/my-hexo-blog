---
title: supplement 20 全局变量、静态局部变量、静态成员变量详细对比
categories: Supplement C++
date: 2025-12-06 22:44:16
tags:
priority: 20
---
# 全局变量、静态局部变量、静态成员变量详细对比

## 1. 基本定义对比

| 类别 | 定义位置 | 声明示例 | 关键特征 |
|------|---------|---------|---------|
| 全局变量 | 所有函数和类之外 | `int global = 10;` | 文件作用域，外部链接(默认) |
| 静态局部变量 | 函数内部 | `void f() { static int x = 0; }` | 块作用域，程序生命周期 |
| 静态成员变量 | 类内部 | `class A { static int s; };` | 类作用域，所有实例共享 |

## 2. 存储生命周期对比

### 内存分布
```cpp
#include <iostream>

// 全局变量 - 存储在数据段(.data 或 .bss)
int g_uninitialized;     // .bss段（零初始化）
int g_initialized = 10;  // .data段
const int g_const = 20;  // .rodata段（只读数据）

class MyClass {
    static int s_class;  // 静态成员 - 在类外定义
    int m_instance;      // 实例成员 - 每个对象独立
public:
    void func() {
        static int s_local = 0;  // 静态局部 - 存储在数据段
        int local = 0;           // 局部变量 - 存储在栈
    }
};

int MyClass::s_class = 30;  // 静态成员定义
```

## 3. 详细特性对比表格

| 特性 | 全局变量 | 静态局部变量 | 静态成员变量 |
|------|---------|-------------|-------------|
| **存储位置** | 数据段(全局区) | 数据段(全局区) | 数据段(全局区) |
| **生命周期** | 程序启动 → 结束 | 第一次使用 → 程序结束 | 程序启动 → 结束 |
| **初始化时机** | 程序启动前(main之前) | 第一次执行到声明处 | 程序启动前(main之前) |
| **默认初始化** | 零初始化(未显式初始化) | 零初始化(未显式初始化) | 必须显式初始化 |
| **作用域** | 从声明点到文件尾 | 声明所在的块内 | 类作用域 |
| **链接性** | 外部链接(默认) | 内部链接(无链接) | 外部链接 |
| **可见性** | 文件内可见，外部可用extern | 仅函数内可见 | 类内可见，类外需作用域限定 |
| **线程安全初始化** | 是(C++11) | 是(C++11) | 是(C++11) |

## 4. 作用域和可见性详细对比

### 全局变量
```cpp
// file1.cpp
int global_var = 100;           // 外部链接
static int file_static = 200;   // 内部链接(文件作用域全局)

void foo() {
    std::cout << global_var << std::endl;  // 可访问
    std::cout << file_static << std::endl; // 可访问
}

// file2.cpp
extern int global_var;  // 正确：链接到file1.cpp的global_var
// extern int file_static;  // 错误：链接错误，file_static内部链接

void bar() {
    std::cout << global_var << std::endl;  // 输出100
}
```

### 静态局部变量
```cpp
void counter1() {
    static int count = 0;  // 只有counter1可访问
    ++count;
    std::cout << "counter1: " << count << std::endl;
}

void counter2() {
    static int count = 0;  // 不同的静态变量
    ++count;
    std::cout << "counter2: " << count << std::endl;
}

void cannot_access() {
    // std::cout << count << std::endl;  // 错误：count不可见
}
```

### 静态成员变量
```cpp
class BankAccount {
    static double interest_rate;  // 类作用域
    double balance;              // 实例作用域
    
public:
    static void set_rate(double new_rate) {
        interest_rate = new_rate;  // 所有实例共享
    }
    
    double calculate_interest() {
        return balance * interest_rate;
    }
};

double BankAccount::interest_rate = 0.05;  // 必须在类外定义

int main() {
    // 通过类名访问
    BankAccount::set_rate(0.03);
    
    BankAccount a1, a2;
    a1.set_rate(0.04);  // 通过对象访问（但不推荐）
    // 修改的是同一个interest_rate
}
```

## 5. 初始化时机和顺序

### 初始化顺序
```cpp
#include <iostream>

// 全局变量 - 在main之前初始化
int global1 = init_global1();  // 1. 动态初始化
int global2 = 20;              // 2. 常量初始化

int init_global1() {
    std::cout << "Initializing global1" << std::endl;
    return 10;
}

class MyClass {
    static int s_member;  // 声明
public:
    MyClass() { 
        std::cout << "MyClass constructor" << std::endl; 
    }
};

// 静态成员 - 在main之前初始化
int MyClass::s_member =  {
    std::cout << "Initializing static member" << std::endl;
    return 30;
}();

void func() {
    // 静态局部变量 - 第一次调用时初始化
    static MyClass local_static;  // 第一次调用func时初始化
    std::cout << "Inside func" << std::endl;
}

int main() {
    std::cout << "Entering main" << std::endl;
    func();  // 第一次调用，初始化local_static
    func();  // 不再初始化
    return 0;
}
```

输出顺序：
```
Initializing global1
Initializing static member
Entering main
MyClass constructor
Inside func
Inside func
```

## 6. 线程安全对比

### 初始化线程安全
```cpp
#include <iostream>
#include <thread>
#include <mutex>

// 全局变量 - C++11保证初始化线程安全
std::string& get_global_string() {
    static std::string s = "Global";  // 线程安全初始化
    return s;
}

// 静态局部变量 - 同样线程安全(C++11)
std::shared_ptr<int> get_singleton() {
    static std::shared_ptr<int> instance = 
        std::make_shared<int>(42);  // 线程安全初始化
    return instance;
}

// 静态成员变量 - 在main之前初始化，但多线程修改需要同步
class Counter {
    static int count;
    static std::mutex mtx;
    
public:
    void increment() {
        std::lock_guard<std::mutex> lock(mtx);
        ++count;
    }
    
    static int get_count() {
        std::lock_guard<std::mutex> lock(mtx);
        return count;
    }
};

int Counter::count = 0;
std::mutex Counter::mtx;
```

## 7. 在lambda中的行为对比

```cpp
#include <iostream>
#include <functional>
#include <vector>

int global_var = 100;

class Widget {
    static int class_static;
    int instance_var = 300;
public:
    std::function<void()> create_lambda() {
        static int func_static = 400;
        int local_var = 500;
        
        // 所有静态变量都不需要捕获
        return  {  // 只捕获local_var和instance_var
            std::cout << "Global: " << global_var << std::endl;
            std::cout << "Class static: " << class_static << std::endl;
            std::cout << "Func static: " << func_static << std::endl;
            std::cout << "Instance var: " << instance_var << std::endl;
            std::cout << "Local var: " << local_var << std::endl;
        };
    }
};

int Widget::class_static = 200;
```

## 8. 单例模式中的不同应用

### 使用静态局部变量（Meyers' Singleton）
```cpp
class SingletonLocalStatic {
public:
    static SingletonLocalStatic& instance() {
        static SingletonLocalStatic instance;  // 线程安全(C++11)
        return instance;
    }
private:
    SingletonLocalStatic() = default;
};
```

### 使用静态成员变量
```cpp
class SingletonStaticMember {
    static SingletonStaticMember* instance;
    static std::once_flag init_flag;
    
public:
    static SingletonStaticMember& get_instance() {
        std::call_once(init_flag,  {
            instance = new SingletonStaticMember();
        });
        return *instance;
    }
private:
    SingletonStaticMember() = default;
};

SingletonStaticMember* SingletonStaticMember::instance = nullptr;
std::once_flag SingletonStaticMember::init_flag;
```

### 使用全局变量
```cpp
// 不推荐，难以控制初始化顺序
SingletonGlobal& get_global_singleton() {
    static SingletonGlobal* instance = new SingletonGlobal();
    return *instance;
}
```

## 9. 编译单元和链接

```cpp
// 编译单元1: a.cpp
int external_var = 1;        // 外部链接，可被其他文件访问
static int internal_var = 2; // 内部链接，仅本文件
const int const_var = 3;     // 内部链接(C++中const默认内部链接)

// 编译单元2: b.cpp
extern int external_var;     // 正确：引用a.cpp的变量
// extern int internal_var;  // 错误：internal_var内部链接
// extern int const_var;     // 错误：const_var内部链接

void func() {
    static int no_linkage = 4;  // 无链接，仅函数内可见
}
```

## 10. 性能和使用场景

| 变量类型 | 性能特点 | 使用场景 | 注意事项 |
|---------|---------|---------|---------|
| 全局变量 | 访问快，但破坏封装 | 1. 程序配置<br>2. 日志系统<br>3. 全局状态 | 1. 避免过度使用<br>2. 注意初始化顺序<br>3. 多线程同步 |
| 静态局部变量 | 延迟初始化，节省内存 | 1. 单例模式<br>2. 函数调用计数<br>3. 缓存 | 1. 线程安全初始化<br>2. 避免递归调用<br>3. 注意重入问题 |
| 静态成员变量 | 类级别共享 | 1. 类相关配置<br>2. 对象计数<br>3. 共享资源池 | 1. 必须在类外定义<br>2. 注意多线程访问<br>3. 继承时的行为 |

## 11. 实际代码示例

```cpp
#include <iostream>
#include <vector>
#include <mutex>

// 全局变量：应用程序配置
struct AppConfig {
    int max_connections = 100;
    std::string log_level = "INFO";
} g_config;

// 静态成员变量：对象计数器
class DatabaseConnection {
    static int total_connections;  // 所有实例共享
    static std::mutex counter_mutex;
    int connection_id;
    
public:
    DatabaseConnection() {
        std::lock_guard<std::mutex> lock(counter_mutex);
        connection_id = ++total_connections;
        std::cout << "Created connection " << connection_id 
                  << " (total: " << total_connections << ")" << std::endl;
    }
    
    ~DatabaseConnection() {
        std::lock_guard<std::mutex> lock(counter_mutex);
        --total_connections;
        std::cout << "Closed connection " << connection_id 
                  << " (total: " << total_connections << ")" << std::endl;
    }
    
    static int get_total() {
        std::lock_guard<std::mutex> lock(counter_mutex);
        return total_connections;
    }
};

int DatabaseConnection::total_connections = 0;
std::mutex DatabaseConnection::counter_mutex;

// 静态局部变量：缓存
std::vector<int> compute_expensive_data() {
    static std::vector<int> cache;  // 第一次计算后缓存
    static std::once_flag init_flag;
    
    std::call_once(init_flag,  {
        std::cout << "Computing expensive data..." << std::endl;
        for (int i = 0; i < 1000000; ++i) {
            cache.push_back(i * i % 100);
        }
    });
    
    return cache;
}

int main() {
    // 使用静态局部变量缓存
    auto data1 = compute_expensive_data();
    auto data2 = compute_expensive_data();  // 使用缓存
    
    // 使用静态成员变量计数
    {
        DatabaseConnection conn1;
        {
            DatabaseConnection conn2, conn3;
            std::cout << "Current connections: " 
                      << DatabaseConnection::get_total() << std::endl;
        }
    }
    
    return 0;
}
```

## 12. 总结对比

| 维度 | 全局变量 | 静态局部变量 | 静态成员变量 |
|------|---------|-------------|-------------|
| **核心区别** | 文件级共享 | 函数级共享 | 类级共享 |
| **封装性** | 差(全局可见) | 好(函数内可见) | 中等(类内可见) |
| **内存效率** | 程序启动时分配 | 第一次使用时分配 | 程序启动时分配 |
| **多线程** | 需显式同步 | 初始化安全，修改需同步 | 需显式同步 |
| **适用场景** | 真正的全局状态 | 函数私有持久状态 | 类相关共享状态 |
| **测试难度** | 高(依赖全局状态) | 中(依赖函数状态) | 中(依赖类状态) |

**关键记忆点**：
- 所有三种都存储在**数据段**，生命周期是**程序运行期**
- 区别主要在**作用域**和**链接性**
- 在lambda中，**都不需要捕获**，可直接访问
- C++11后，**初始化都是线程安全的**，但**修改需要同步**