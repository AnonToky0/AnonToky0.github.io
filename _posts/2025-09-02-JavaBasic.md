---
title: "Java基础"
date: 2025-09-02 21:00:00 +0800
categories: [Java]
tags: [Java]
---
本文档暂时只提供对Lancet代码的补充说明，后续将扩充Java基础语法，工程结构等。

## 元注解
Java中的元注解是用来修饰注解的注解，常见的有以下几种：

| 元注解       | 作用               | 说明         |
|--------------|-------------------------------|-------------------------------------------|
| `@Retention` | 指定注解的生命周期| 如 `SOURCE`、`CLASS`、`RUNTIME`                              |
| `@Target`    | 指定注解可以应用的Java元素类型 | 如 `TYPE`（类、接口）、`METHOD`（方法）、`FIELD`（字段）等   |
| `@Documented`| 指示注解是否包含在Javadoc中    | 如果使用该注解，注解信息会被Javadoc工具提取出来，默认不包含   |
| `@Inherited` | 指示注解是否可以被子类继承  | 只对类注解有效，子类会自动继承父类的注解   |

## InsertClassVisitor 工作机制
负责将Hook方法织入到目标类的指定方法。
### 主要流程
1. 类级别处理
- 当 ASM 访问到一个类时，visit 方法会被调用。
- 通过 executeInfos.get(getContext().name) 获取当前类所有需要插入的 InsertInfo（即所有与该类相关的 hook 规则）。
- 这些 InsertInfo 会在后续方法遍历时用到。

2. 方法级别处理
- 遍历类的每个方法，查找与 InsertInfo 匹配的方法（方法名和方法签名一致）。
- 校验 static 修饰符是否一致（hook 方法和目标方法必须同为 static 或非 static）。

- 如果找到匹配项且方法不是 native/abstract：
    - 生成一个“孪生”方法（newName = name + "$twin"，私有），保存原始方法逻辑。
        - 构建 MethodChain，将所有 hook 方法链式织入到目标方法。
            - chain.headFromInsert：记录孪生方法的头部信息。
            - chain.next：将每个 hook 方法按顺序加入链条。
            - chain.fakePreMethod：生成一个“假方法”，用于后续织入。
    - 最终用 super.visitMethod 新增孪生方法，原方法将被 hook 逻辑替换。

3. 生成 super 调用（visitEnd 方法）
- 有些 hook 规则要求在插入逻辑中能访问父类的原始方法（createSuper 标记）。
- visitEnd 会自动为这些方法生成一个方法体，调用 super.method 并返回结果，- 保证 hook 后仍可访问原始父类逻辑。