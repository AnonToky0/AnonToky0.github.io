---
title: "Linux常用命令"
date: 2025-09-05 10:00:00 +0800
categories: [Linux]
tags: [Linux]
---

## 文件与目录
- ls 列出目录内容
- cd 切换当前工作目录
- pwd 显示当前工作目录的绝对路径
- mkdir 创建新目录
- rm 删除文件或目录
- cp 复制文件或目录
- mv 移动或重命名文件/目录
- find 在指定目录下查找文件

## 文件内容
- cat 查看文件内容
- more/less 分页查看文件内容
- head/tail 查看文件开头/结尾内容

### grep 文本搜索
``` SHELL
grep [选项] PATTERN [FILE...]
```
- PATTERN：要搜索的字符串或正则表达式。
- [FILE...]：一个或多个文件，默认从标准输入读取
示例：  
``` SHELL
grep "hello" file.txt
```
用于查找file.txt中包含字符串hello的所有行，输出匹配行的内容

## 进程管理
- ps 显示当前进程的快照
- top/htop 动态实时显示进程状态和系统资源使用情况
- kill 向进程发送信号以终止或控制进程

## 权限管理
- chomod 修改文件或目录的访问权限
- chown 更改文件或目录的所有者或所属组

## 系统与网络
- df 显示磁盘空间使用情况
- du 估算文件或目录的磁盘使用量
- ifconfig/ip 配置或显示网络接口信息
- ping 测试与目标主机的网络连通性

## 压缩归档
- tar 打包和解压文件，支持gzip/bzip2

## 工具
- awk 文本处理工具
- sed 流编辑器，用于文本替换、删除、插入等
- wget/curl 从网络下载文件或与网络服务交互
- man 查看命令的手册页，获取详细帮助信息