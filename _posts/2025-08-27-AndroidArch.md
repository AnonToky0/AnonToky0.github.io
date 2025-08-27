---
title: "Android 架构"
date: 2025-08-27 15:00:00 +0800
categories: [Android]
tags: [Android]
---

## 系统进程(System Process)
在操作系统中专门用于运行系统服务和管理操作的进程，负责管理硬件资源、调度应用程序、维护系统状态等关键任务。

### Android中的系统进程
- system_server 进程：Android 设备启动后，Zygote 进程会fork出 system_server 进程，这是 Android 系统中最重要的系统进程之一。
- system_server 进程中运行着大量核心系统服务，如 ActivityManagerService（AMS）、WindowManagerService、PackageManagerService、PowerManagerService 等。
- 这些服务负责管理应用生命周期、窗口管理、包管理、电源管理等，是 Android 系统的“大脑”。

### 系统进程与应用进程的区别

| 方面           | 系统进程                      | 应用进程                      |
|----------------|------------------------------|------------------------------|
| 启动方式       | 由系统启动，通常在设备启动时自动启动 | 由系统或用户启动，运行用户应用代码  |
| 运行权限       | 拥有较高权限，能访问系统资源和硬件接口 | 受限权限，运行在沙箱环境中         |
| 运行内容       | 运行系统服务和管理程序           | 运行用户应用程序和业务逻辑         |
| 进程数量       | 通常较少，关键服务集中运行         | 多个，按应用数量和需求动态创建      |

### 系统进程作用举例
- ActivityManagerService (AMS)：管理应用进程和 Activity 生命周期。
- WindowManagerService：管理窗口和界面显示。
- PackageManagerService：管理应用安装和权限。
- InputManagerService：管理输入事件。

### PackageManagerService (PMS)
- 当用户安装一个 APK 文件时，Android 系统的 PackageManagerService（PMS） 负责解析和管理安装包。
- PMS 会从 APK 中读取并解析 AndroidManifest.xml 文件。
- 在解析过程中，PMS 会提取所有的组件声明，包括 <activity>, <service>, <receiver>, <provider> 等。

## 客户端进程(Client Process)
客户端进程是指运行 Android 应用程序代码的 Linux 进程。每个应用通常运行在独立的客户端进程中，确保应用间相互隔离，提升系统安全性和稳定性。

### 组成
- 主线程（UI线程）：客户端进程中最重要的线程，负责界面绘制、事件分发和处理。Android 把客户端进程的主线程称为 ActivityThread 。
- 其他线程：用于后台任务、网络请求、数据库操作等。

### ApplicationThread
ApplicationThread 是一个 Binder 接口的实现，运行在客户端进程中，由 ActivityThread 创建并管理。
- 作为客户端进程对系统进程（主要是 AMS）的“代理”，接收来自系统进程的 IPC 调用。
- 系统进程通过 ApplicationThread 向客户端进程发送指令，比如启动 Activity、暂停 Activity、调度生命周期事件等。
- ApplicationThread 的实现把这些调用转发给 ActivityThread 。

### 运行机制
- AMS 持有客户端进程中 ApplicationThread 接口的代理对象（Binder Proxy）。
- AMS 通过 Binder 调用 ApplicationThread 中的方法，通知客户端进程执行相应的操作。
- ApplicationThread 接收到调用后，转发给 ActivityThread ，由它在主线程中处理。

## APK合法性验证与安装准备
### 验证APK合法性
- 包名、版本号校验
- 签名校验(V1+V2签名方案)
- 其他应用验证
    - 系统中的`package-verifier` 组件会对安装包进行安全扫描，检测恶意代码、权限滥用等。

### 权限申请与用户确认
- 应用中声明的权限会在安装时展示给用户

### 安装文件准备
- 拷贝安装包
    - 将安装包复制到应用私有目录 /data/data/{包名}/
    - 将APK 中的Native 库文件(.so文件)复制到对应lib目录下，方便系统加载

### 解析APK文件
- 添加APK路径到`AssetManager`,通过 AssetManager 管理 APK 中的资源文件，方便后续资源加载。
- 解析 AndroidManifest.xml：
    - 解析应用声明的组件、权限、版本信息等。
    - 生成系统使用的 Package 类对象，保存应用的相关元数据。
    - 区分单个 APK 安装和多 APK 组合安装（如拆分 APK、动态特性模块等）。

### 测试包验证
- 检查 APK 是否标记为测试包（android:testOnly="true"）。
- 测试包一般不能直接发布到正式环境，系统会限制其安装或升级。

### 填充签名和证书信息
- 将解析出的签名证书信息填充到 Package 类中。
- 这样系统可以在运行时验证应用签名，保证应用完整性和安全。

### 覆盖安装验证
如果是覆盖（升级）安装，需要进行以下验证：  
- 签名是否一致：新 APK 必须与已安装应用签名匹配，防止恶意替换。
- 包配置是否兼容：如版本号、权限等是否合理。

