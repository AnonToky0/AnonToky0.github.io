---
title: "C++ 数据类型"
date: 2025-09-07 11:00:00 +0800
categories: [C++]
tags: [C++]
---

C++数据类型主要包括以下几类：  

| 类型类别     | 说明               | 示例               |
|-----------|-------------------|--------------------------|
| 内置类型      | 编译器内置的基本数据类型  | `int`, `char`, `float`, `bool`  |
| 用户自定义类型  | 通过class/struct/enum定义的类型 | `class Person`, `enum Color`    |
| 指针类型    | 存储地址的类型       | `int*`, `char*`                 |
| 引用类型  | 变量的别名      | `int&`                         |
| 标准库类型   | C++标准库提供的各种模板类 | `std::string`, `std::vector`    |
| 函数类型 | 函数或函数指针类型   | `int(int, double)`    |
| 数组类型   | 固定大小的元素集合    | `int arr[10]`                   |
| 空类型  | 无类型（void）     | `void`          |
| 类型别名     | 给类型起别名    | `typedef unsigned int uint;`   |


## 内置类型(Built-in types)
也称 基本类型, 是C++语言中预定义的、直接由编译器支持的基本数据类型。

| 类型类别       | 具体类型         | 说明           |
| --------- | ------------------------ | ----------------- |
| 整型 | `int`, `short`, `long`, `long long` | 用于表示整数  |
| 字符型 | `char`, `wchar_t`, `char16_t`, `char32_t`|用于表示字符 |
| 浮点型  | `float`, `double`, `long double` | 用于表示带小数的实数    |
| 布尔型  | `bool`   | 表示真（true）或假（false）  |
| 空类型  | `void` | 表示无类型，常用于函数无返回值 |

## 用户自定义类型(User-defined types)

### 类(class)和结构体(struct)

``` cpp
class Person {
public:
    std::string name;
    int age;
};
```

### 枚举(enum)

``` cpp
enum Color { Red, Green, Blue };
```

### 联合体(union)

``` cpp
union Data {
    int i;
    float f;
};
```

## 指针类型(Pointer types)
- 指向其他类型的地址，也是内置类型，但特殊用途。

``` cpp
int* p;        // 指向int的指针
const char* s; // 指向const char的指针，这里是常量指针
```

## 引用类型(Reference types)
- 引用是某个变量的别名。

``` cpp
int a = 10;
int& ref = a;  // ref是a的引用
```

## 标准库类型(Standard Library Types)
- C++标准库提供的各种类型，通常是模板类。

  - **字符串**

    ```cpp
    std::string str = "Hello";
    ```

  - **容器**

    ```cpp
    std::vector<int> vec;
    std::map<int, std::string> m;
    ```

  - **智能指针**

    ```cpp
    std::unique_ptr<int> ptr(new int(5));
    ```

## 函数类型(Function types)
表示函数的类型，可以作为指针或对象。

``` cpp
int func(int, double);
int (*funcPtr)(int, double) = func;
```

## 数组类型(Array types)
固定大小的元素集合。

``` cpp
int arr[10];
```

## 其他类型()
- 空类型（void）：表示无类型，常用于函数无返回值。
- 类型别名（typedef / using）：给已有类型起别名。
``` cpp
typedef unsigned int uint;
using uint = unsigned int;
```

