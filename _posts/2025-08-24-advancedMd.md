---
title: "Markdown进阶语法"
date: 2025-08-24 18:00:00 +0800
categories: [分类1, 分类2]
tags: [标签1, 标签2]
---
摘录自：[Markdown教程](https://markdown.com.cn/basic-syntax)

**注意**：并非所有应用程序都支持扩展语法元素。
## 表格
使用三个或多个连字符（---）创建每列的标题，并使用管道（|）分隔每列。可以选择在表的任一端添加管道。
```
| Syntax      | Description |
| ----------- | ----------- |
| Header      | Title       |
| Paragraph   | Text        |
```
效果如下：  

| Syntax | Description |
| --- | ----------- |
| Header | Title |
| Paragraph | Text |

- 单元格宽度可以变化，如下所示。呈现的输出将看起来相同。
```
| Syntax | Description |
| --- | ----------- |
| Header | Title |
| Paragraph | Text |
```
- **Tips**: 使用连字符和管道创建表可能很麻烦。可以使用[Markdown Tables Generator](https://www.tablesgenerator.com/markdown_tables)。使用图形界面构建表，然后将生成的Markdown格式的文本复制到文件中。

### 对齐
可以通过在标题行中的连字符的左侧，右侧或两侧添加冒号（:），将列中的文本对齐到左侧，右侧或中心。
```
| Syntax      | Description | Test Text     |
| :---        |    :----:   |          ---: |
| Header      | Title       | Here's this   |
| Paragraph   | Text        | And more      |
```
效果如下：  

| Syntax      | Description | Test Text     |
| :---        |    :----:   |          ---: |
| Header      | Title       | Here's this   |
| Paragraph   | Text        | And more      |

### 对齐
可以在表格中设置文本格式。例如，可以添加链接，代码（仅反引号（`）中的，而不是代码块）和强调。  
不能添加标题，块引用，列表，水平规则，图像或HTML标签。

### 表格中转义管道字符
可以使用表格的HTML字符代码`&#124;`在表中显示竖线`|`字符。