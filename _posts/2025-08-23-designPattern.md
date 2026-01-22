---
title: "设计模式"
date: 2025-08-26 16:30:00 +0800
categories: [算法]
tags: [C++，设计模式]
---

## 设计模式
设计模式是软件工程中为解决特定问题而总结的可复用的解决方案。
    设计模式通常分三大类：
        1. 创建型模式：关注对象的创建过程，帮助在不同情况下创建对象
        2. 结构型模式：关注类和对象的组合，帮助构建更大的结构
        3. 行为型模式：关注对象之间的通信和职责分配，帮助管理对象间的交互
        
-创建型模式：主要解决对象的创建问题，帮助系统在不同情况下灵活创建对象
    1. 单例模式
        确保一个类只有一个实例，并提供一个全局访问点
        应用场景：
            需要一个全局唯一的对象，如配置管理器、日志记录器等
        懒汉模式：
            单例模式的一种实现方式，特点是在需要实例时才会创建实例(即懒加载)
    2. 工厂方法模式
        定义一个用于创建对象的接口，让子类决定实例化哪一个类，工厂方法使得类的实例化延迟到子类
        应用场景：
            当一个类不知道它所需的对象的具体类型时。
            当一个类希望由子类来指定它所创建的对象时。
    3. 


## 责任链(Chain of Responsibility)
责任链模式是一种行为设计模式，它让多个对象都有机会处理请求，将这些对象连成一条链，并沿着链传递请求，直到有对象处理它为止。  
- 核心思想：将请求的发送者和接收者解耦，请求沿着链传递，直到被某个处理者处理。
- 结构：
    - 抽象处理者（Handler）：定义处理请求的接口，并保持对下一个处理者的引用。
    - 具体处理者（ConcreteHandler）：实现请求处理，决定是否处理请求，或传递给下一个处理者。

### 优点
- 降低耦合度：发送者只需知道链的第一个处理者，具体处理者之间相互独立。
- 增强灵活性：可以动态改变链的结构，灵活组合处理者。
- 增强扩展性：新增处理者简单，不影响其他代码。

### 使用场景
1. 多个对象可以处理一个请求，但具体哪个对象处理不确定。
例如：日志系统中不同级别的日志处理器。

2. 希望动态指定请求的处理者，避免请求发送者与接收者耦合。
例如：事件处理系统，事件沿着组件树传递。

3. 处理请求的对象集合应被动态指定或者动态改变。
例如：权限校验链，多个校验器依次验证。

4. 请求处理者之间存在先后顺序，且可灵活调整。
例如：审批流程，一级审批不通过则传递给二级审批。

### 应用举例
- 日志处理系统：不同日志级别由不同处理器处理。
- 事件处理系统：GUI事件从子组件传递到父组件。
- 请求审批流程：请假申请依次通过部门经理、HR、总经理审批。
- 权限校验链：依次进行身份验证、权限验证、访问控制。

### 代码示例

``` cpp
#include <iostream>
#include <string>
#include <memory>

using namespace std;

enum LogLevel
{
    INFO,
    DEBUG,
    ERROR
};

class Handler
{
protected:
    shared_ptr<Handler> nextHandler;

public:
    virtual ~Handler() = default;

    void setNext(shared_ptr<Handler> next)
    {
        nextHandler = next;
    }

    void handleRequest(LogLevel level, const string &message)
    {
        if (canHandle(level))
        {
            process(message);
        }
        else if (nextHandler)
        {
            nextHandler->handleRequest(level, message);
        }
        else
        {
            cout << "No Handler for this log level.\n";
        }
    }

    virtual bool canHandle(LogLevel level) = 0;
    virtual void process(const string &message) = 0;
};

class InfoHandler : public Handler
{
public:
    bool canHandle(LogLevel level) override
    {
        return level == INFO;
    }
    void process(const string &message) override
    {
        cout << "[INFO]:" << message << endl;
    }
};

class ErrorHandler : public Handler
{
public:
    bool canHandle(LogLevel level) override
    {
        return level == ERROR;
    }
    void process(const string &message) override
    {
        cout << "[ERROR]:" << message << endl;
    }
};

class DebugHandler : public Handler
{
public:
    bool canHandle(LogLevel level) override
    {
        return level == DEBUG;
    }
    void process(const string &message) override
    {
        cout << "[DEBUG]:" << message << endl;
    }
};

int main()
{
    auto infoHandler = make_shared<InfoHandler>();
    auto debugHandler = make_shared<DebugHandler>();
    auto errorHandler = make_shared<ErrorHandler>();

    infoHandler->setNext(debugHandler);
    debugHandler->setNext(errorHandler);

    // 测试日志处理
    infoHandler->handleRequest(INFO, "This is an info message.");
    infoHandler->handleRequest(DEBUG, "This is a debug message.");
    infoHandler->handleRequest(ERROR, "This is an error message.");
    infoHandler->handleRequest((LogLevel)100, "This level does not exist.");

    return 0;
}
```

## 单例模式

``` cpp

class Singleton(){
public:
    static Singleton& getInstance(){
        static Singleton instance;
        return instance;
    }

    Single(const Single&) = delete;
    Single& operator = (const Singleton&) = delete;

    void dosometing(){

    }

private:
    Singleton() = default;
    ~Singleton() = default;

};

```