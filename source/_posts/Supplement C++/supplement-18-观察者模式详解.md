---
title: supplement 18 观察者模式详解
categories: Supplement C++
date: 2025-11-25 23:08:24
tags:
priority: 18
---
# 观察者模式详解

观察者模式是一种行为设计模式，它定义了对象之间的一对多依赖关系，当一个对象的状态发生改变时，所有依赖于它的对象都会自动得到通知并更新。

## 基本概念

### 核心角色
- **Subject（主题）**：被观察的对象，维护观察者列表并提供注册/注销接口
- **Observer（观察者）**：定义更新接口，接收主题状态变化的通知
- **ConcreteSubject（具体主题）**：具体被观察的对象，状态改变时通知观察者
- **ConcreteObserver（具体观察者）**：实现更新接口，保持与主题状态一致

## 传统实现 vs 现代C++实现

### 传统观察者模式（可能有问题）
```cpp
// 传统实现 - 可能有生命周期问题
class Observable;  // 前向声明

class Observer {
public:
    virtual void update(Observable* subject) = 0;
    virtual ~Observer() = default;
};

class Observable {
    std::vector<Observer*> observers_;  // 原始指针，危险！
    
public:
    void addObserver(Observer* observer) {
        observers_.push_back(observer);
    }
    
    void removeObserver(Observer* observer) {
        observers_.erase(std::remove(observers_.begin(), observers_.end(), observer), 
                        observers_.end());
    }
    
    void notifyObservers() {
        for (auto observer : observers_) {
            observer->update(this);  // 如果observer已被删除，程序崩溃！
        }
    }
};
```

### 现代C++实现（使用智能指针）
```cpp
#include <memory>
#include <vector>
#include <iostream>

// 前向声明
class Observable;

// 观察者基类
class Observer : public std::enable_shared_from_this<Observer> {
public:
    virtual void update(std::shared_ptr<Observable> subject) = 0;
    virtual ~Observer() = default;
    
    // 订阅方法
    void subscribeTo(std::shared_ptr<Observable> observable) {
        observable->addObserver(shared_from_this());
    }
    
    // 取消订阅
    void unsubscribeFrom(std::shared_ptr<Observable> observable) {
        observable->removeObserver(shared_from_this());
    }
};

// 被观察者
class Observable {
    std::vector<std::weak_ptr<Observer>> observers_;  // 使用weak_ptr避免循环引用
    
public:
    void addObserver(std::shared_ptr<Observer> observer) {
        observers_.push_back(observer);  // shared_ptr隐式转换为weak_ptr
    }
    
    void removeObserver(std::shared_ptr<Observer> observer) {
        observers_.erase(
            std::remove_if(observers_.begin(), observers_.end(),
                const std::weak_ptr<Observer>& wp {
                    if (auto sp = wp.lock()) {
                        return sp == observer;
                    }
                    return false;
                }),
            observers_.end());
    }
    
    void notifyObservers() {
        // 移除已销毁的观察者
        observers_.erase(
            std::remove_if(observers_.begin(), observers_.end(),
                const std::weak_ptr<Observer>& wp {
                    return wp.expired();
                }),
            observers_.end());
        
        // 通知存活的观察者
        for (auto& weak_observer : observers_) {
            if (auto observer = weak_observer.lock()) {
                observer->update(shared_from_this());
            }
        }
    }
    
    virtual ~Observable() = default;
};
```

## 具体示例：温度监控系统

```cpp
#include <memory>
#include <vector>
#include <iostream>
#include <random>

// 温度传感器（被观察者）
class TemperatureSensor : public Observable, public std::enable_shared_from_this<TemperatureSensor> {
private:
    double current_temperature_;
    
public:
    TemperatureSensor() : current_temperature_(20.0) {}
    
    void setTemperature(double temp) {
        if (current_temperature_ != temp) {
            current_temperature_ = temp;
            std::cout << "温度传感器: 温度变为 " << temp << "°C\n";
            notifyObservers();  // 通知所有观察者
        }
    }
    
    double getTemperature() const { return current_temperature_; }
};

// 显示屏观察者
class Display : public Observer {
private:
    std::string name_;
    
public:
    Display(const std::string& name) : name_(name) {}
    
    void update(std::shared_ptr<Observable> subject) override {
        if (auto sensor = std::dynamic_pointer_cast<TemperatureSensor>(subject)) {
            std::cout << "[" << name_ << "] 显示温度: " 
                      << sensor->getTemperature() << "°C\n";
        }
    }
};

// 报警器观察者
class Alarm : public Observer {
private:
    double threshold_;
    
public:
    Alarm(double threshold) : threshold_(threshold) {}
    
    void update(std::shared_ptr<Observable> subject) override {
        if (auto sensor = std::dynamic_pointer_cast<TemperatureSensor>(subject)) {
            double temp = sensor->getTemperature();
            if (temp > threshold_) {
                std::cout << "🚨 报警！温度过高: " << temp << "°C (阈值: " 
                          << threshold_ << "°C)\n";
            }
        }
    }
};

// 日志记录观察者
class Logger : public Observer {
public:
    void update(std::shared_ptr<Observable> subject) override {
        if (auto sensor = std::dynamic_pointer_cast<TemperatureSensor>(subject)) {
            std::cout << "📝 日志: 温度记录为 " 
                      << sensor->getTemperature() << "°C\n";
        }
    }
};
```

## 使用示例

```cpp
void temperatureMonitoringDemo() {
    // 创建温度传感器
    auto sensor = std::make_shared<TemperatureSensor>();
    
    // 创建观察者
    auto livingRoomDisplay = std::make_shared<Display>("客厅显示屏");
    auto bedroomDisplay = std::make_shared<Display>("卧室显示屏");
    auto highTempAlarm = std::make_shared<Alarm>(30.0);
    auto logger = std::make_shared<Logger>();
    
    // 观察者订阅传感器
    livingRoomDisplay->subscribeTo(sensor);
    bedroomDisplay->subscribeTo(sensor);
    highTempAlarm->subscribeTo(sensor);
    logger->subscribeTo(sensor);
    
    std::cout << "=== 开始温度监控 ===\n";
    
    // 模拟温度变化
    sensor->setTemperature(25.5);
    sensor->setTemperature(28.0);
    sensor->setTemperature(32.5);  // 触发报警
    
    std::cout << "\n=== 卧室显示屏取消订阅 ===\n";
    bedroomDisplay->unsubscribeFrom(sensor);
    
    sensor->setTemperature(29.0);  // 只有客厅显示屏、报警器、日志器接收通知
    
    // 演示自动清理：观察者超出作用域自动取消订阅
    {
        auto tempDisplay = std::make_shared<Display>("临时显示屏");
        tempDisplay->subscribeTo(sensor);
        
        std::cout << "\n=== 临时显示屏订阅后温度变化 ===\n";
        sensor->setTemperature(27.5);
        
    }  // tempDisplay超出作用域被销毁
    
    std::cout << "\n=== 临时显示屏销毁后温度变化 ===\n";
    sensor->setTemperature(26.0);  // 临时显示屏自动从观察者列表移除
}
```

## 观察者模式的优势

### 1. **松耦合**
```cpp
// 主题和观察者相互独立
// 可以轻松添加新的观察者类型
class MobileApp : public Observer {
    void update(std::shared_ptr<Observable> subject) override {
        // 手机App接收通知的实现
    }
};

// 无需修改TemperatureSensor即可支持新观察者
auto mobileApp = std::make_shared<MobileApp>();
mobileApp->subscribeTo(sensor);  // 直接订阅
```

### 2. **动态关系**
```cpp
// 运行时动态添加/移除观察者
sensor->setTemperature(25.0);

// 动态添加观察者
auto newDisplay = std::make_shared<Display>("新显示屏");
newDisplay->subscribeTo(sensor);

sensor->setTemperature(26.0);  // 新显示屏也会收到通知

// 动态移除观察者
newDisplay->unsubscribeFrom(sensor);
```

### 3. **自动内存管理**
```cpp
// 使用weak_ptr避免循环引用
// 观察者被销毁时自动从列表中移除

{
    auto temporaryObserver = std::make_shared<Display>("临时观察者");
    temporaryObserver->subscribeTo(sensor);
    
    // temporaryObserver超出作用域被销毁
    // Observable会自动清理过期的weak_ptr
}
```

## 实际应用场景

### GUI事件处理
```cpp
class Button : public Observable {
public:
    void click() {
        // ... 按钮点击逻辑
        notifyObservers();  // 通知所有监听点击事件的观察者
    }
};

class ClickHandler : public Observer {
    void update(std::shared_ptr<Observable> subject) override {
        if (auto button = std::dynamic_pointer_cast<Button>(subject)) {
            std::cout << "按钮被点击了！\n";
        }
    }
};
```

### 发布-订阅系统
```cpp
class MessageBus : public Observable {
    std::string current_message_;
    
public:
    void publish(const std::string& message) {
        current_message_ = message;
        notifyObservers();
    }
    
    std::string getMessage() const { return current_message_; }
};

class Subscriber : public Observer {
    std::string name_;
    
public:
    Subscriber(const std::string& name) : name_(name) {}
    
    void update(std::shared_ptr<Observable> subject) override {
        if (auto bus = std::dynamic_pointer_cast<MessageBus>(subject)) {
            std::cout << "[" << name_ << "] 收到消息: " 
                      << bus->getMessage() << "\n";
        }
    }
};
```

## 总结

观察者模式的核心价值：
- **解耦**：主题和观察者松散耦合
- **扩展性**：容易添加新的观察者
- **动态性**：运行时可以动态建立和解除关系
- **安全性**：使用智能指针自动管理生命周期

在现代C++中，结合`std::shared_ptr`、`std::weak_ptr`和`std::enable_shared_from_this`，可以构建出既安全又灵活的观察者模式实现。