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

## 数据存储
### SharedPreferences
存储少量的键值对数据，适合保存简单的配置信息、用户设置等。
数据以XML文件形式保存在应用的私有目录中。

SharedPreferences 底层使用的是**XML**文件  
XML 文件中以键值对的形式存储数据，例如：
``` xml
<map>
    <string name="username">Alice</string>
    <int name="age" value="25" />
    <boolean name="isPremium" value="true" />
</map>
```

- 读写流程
将对应的 XML 文件解析为内存中的一个`Map<String, Object>`
- 读取方法主要是通过它的接口提供的各种 get 方法来实现
- 写入时，调用 SharedPreferences.Editor 的 commit() 或 apply() 方法。
    - commit() 是同步写入，直接写入文件，返回写入结果。
    - apply() 是异步写入，先更新内存缓存，然后在后台线程写入文件，不阻塞调用线程。
SharedPreferences 读取是**线程安全**的，多个线程可以同时读取。

2. 文件存储
- 内部存储(Internal Storage)
    - 存储在设备内部存储空间，应用私有。
    - 通过 `openFileOutput()`和`openFileInput()`读写文件

- 外部存储(External Storage)
    - 存储在SD卡或设备的公共存储空间
    - 适合存储大文件、共享文件、如图片、音视频等
    - 需要申请权限
    - 访问方式通过 Environment.getExternalStorageDirectory()等API

3. SQLite数据库
用于存储结构化数据，支持复杂查询和事务。
- 内嵌关系型数据库，支持SQL语句
- 适合存储大量数据或复杂数据关系
- 通过`SQLiteOpenHelper`管理数据库创建和升级

``` java
SQLiteOpenHelper helper = new MyDatabaseHelper(context);
SQLiteDatabase db = helper.getWritableDatabase();
db.execSQL("INSERT INTO user (name, age) VALUES (?, ?)", new Object[]{"Bob", 30});
```

4. ContentProvider
用于实现应用间数据共享
- 通过URL 访问数据，封装数据访问逻辑
- 支持跨进程访问
- 常用于联系人、媒体库等系统数据访问，也可以自定义内容提供者

5. 网络存储
将数据存储在远程服务器或云端。
- 通过HTTP/HTTPS、WebSocket等协议访问
- 适合跨设备同步、备份和共享
常用服务：Firebase、RESful API等

## 热发布(Hot Deployment), 热更新(Hot update), 热修复(Hotfix)
强调不中断服务或使用，侧重点有区别。

| 概念       | 定义                  | 侧重点            | 
|------------|---------------------------------------|---------------------|
| **热发布** | 在系统运行中，动态发布新版本代码或资源，无需停止服务。         | 整体应用或模块的替换或升级    |
| **热更新** | 在运行时更新应用程序的部分代码或资源，通常指客户端或游戏端。   | 小范围代码或资源的在线更新   | 
| **热修复** | 针对线上出现的bug，快速修复并更新代码，无需重新发布完整版本。   | 快速修复线上缺陷    | 


### 热发布(Hot Deployment)
在系统或应用程序正在运行的情况下，直接更新、替换代码、配置或资源，而无需停止系统或服务的发布方式。  
换句话说，就是实现“不停机更新”，用户几乎感受不到服务中断。

#### 热发布原理
热发布的核心目标是在系统运行时动态替换或更新代码及资源，保证服务不中断。实现这一目标，通常依赖以下几个关键原理：
1. 动态模块化设计
- 模块化拆分
系统被拆分成多个独立模块或组件，每个模块相对独立，职责单一。
- 动态加载和卸载
系统支持在运行时加载新的模块版本，同时卸载旧版本模块。如Java中的类加载器(ClassLoader)机制可以实现类的动态加载和卸载。

2. 版本管理和兼容性
- 多版本共存
允许新旧版本模块在一定时间内共存，避免直接替换导致的冲突。
- 接口兼容
新版本需要保持与旧版本接口兼容，确保调用方不受影响。
- 数据结构兼容
数据格式或存储结构需要兼容，防止因更新导致数据错误。

3. 状态迁移和持久化
- 状态保存
更新时保存当前模块的运行状态（如内存数据、会话信息）。
- 状态恢复
新版本模块启动后，恢复之前保存的状态，保证业务连续性。
- 无状态设计
越来越多系统采用无状态设计，将状态保存在外部存储（数据库、缓存），简化热发布。

4. 负载均衡与流量切换（分布式系统）
- 渐进式更新
通过负载均衡器，将一部分流量切换到新版本实例，观察运行情况。
- 蓝绿部署/滚动更新
先部署新版本环境（蓝），切换流量后关闭旧版本环境（绿），减少风险。
- 回滚机制
出现问题时，快速将流量切回旧版本。

### 热更新
通常是客户端或部分代码、资源的更新。通过在线更新资源（如图片、配置文件）或脚本代码，减少用户下载完整安装包。  
多用于移动端、游戏等需要频繁更新资源的场景。

常用热更新框架  

| 框架名称 | 说明                              | 支持内容           | 状态                 |
|----------|---------------------------------|--------------------|----------------------|
| Tinker   | 腾讯开源，支持代码、资源、so库热更新 | 代码、资源、Native库 | 活跃，广泛使用       |
| AndFix   | 阿里开源，支持代码热修复           | 代码                | 已停止维护           |
| Robust   | 美团开源，支持代码热修复           | 代码                | 维护中               |
| Sophix   | 阿里巴巴，基于Tinker的热更新方案   | 代码、资源          | 商业支持，部分闭源   |

#### 流程简述
以Tinker为例，
1. 集成Tinker SDK
在项目中添加Tinker依赖和Gradle插件。
2. 开发和测试
正常开发应用，发布基础包。
3. 生成补丁包
修复bug后，使用Tinker工具生成补丁包（patch），仅包含修改的dex、资源或so。
4. 发布补丁
将补丁包上传到服务器。
5. 加载补丁
应用启动时或运行时检查补丁，下载并动态加载。
6. 补丁生效
补丁代码和资源替换原有内容，修复问题。

### 热修复 
范围最小，专注于快速修复代码缺陷。通过注入补丁代码、替换部分函数或模块。

## 动态加载
程序在运行时（而不是编译或启动时）按需加载代码或资源的技术。

### 典型应用
1. 动态加载Dex文件
Android应用的代码编译成dex格式。  
通过`DexClassLoader`或`PathClassLoader`在运行时加载外部dex文件。  
例如热修复框架（如Tinker）就是通过动态加载补丁dex文件来替换或修复代码。

2. 动态加载so库（Native库）
通过`System.loadLibrary`或`System.load`在运行时加载本地so库。
支持根据设备或需求加载不同的本地实现。  

3. 动态加载资源
通过Resources和AssetManager加载外部资源包。
用于主题切换、资源更新等场景

### so库
指**共享对象库**（Shared Object Library）,是一种动态链接库(Dynamic Link Library，DLL)的形式。它包含了编译后的本地（Native）代码，可以被应用程序在运行时加载和调用。

- Android应用主要用Java/Kotlin开发，但有时需要调用本地代码提升性能或使用系统底层功能
- 本地代码打包成.so文件，放在APK的lib/目录下，按CPU架构分类（如armeabi-v7a、arm64-v8a等）
- 通过JNI（Java Native Interface）机制，Java代码可以调用.so库里的本地函数
- 运行时可以通过System.loadLibrary("库名")加载对应的.so库

- **本地代码（Native Code）**
是指直接运行在操作系统底层、使用平台特定的机器指令集编写的程序代码，通常用**C、C++**等编程语言开发，而不是运行在虚拟机或解释器上的代码。

## Gradle

### Gradle 构建时，Task的执行顺序
1. 任务依赖关系
任务可以通过`dependOn`,`mustRunAfter`,`shouldRunAfter`等方法声明依赖关系

2. 拓扑排序
Gradle 根据所有任务的依赖关系构建一个有向无环图  
保证依赖的任务先执行，依赖它的任务后执行

3. 配置与执行
- 配置阶段，解析所有任务及其依赖关系，构建任务依赖图
- 执行阶段，Gradle会按照依赖图的拓扑顺序执行任务

4. 并行执行
如果任务之间没有依赖关系，Gradle 可以并行执行这些任务（如果启用了并行构建）。如果没有并行，Gradle 会按它们在命令行或脚本中声明的顺序依次执行

## 编译 打包流程
1. 源码准备阶段
- Java/Kotlin源码：应用的业务逻辑代码
- 资源文件：布局XML、图片、字符串资源等
- AndroidManifest.xml:应用配置文件，声明组件和权限等
- 第三方库依赖：通过gradle引入的依赖

2. 资源处理阶段
aapt2(Android Asset Packaging Tool 2)
- 编译所有资源文件(XML、图片等)
- 生成二进制资源文件(.flat文件)
- 生成`R.java`文件(资源索引的Java类)，供代码引用资源ID
- 生成资源包(resources.ap_)

3. 代码编译阶段
- Java/Kotlin 编译
    - Java 代码通过 javac 编译成 .class 字节码文件
    - Kotlin 代码通过 Kotlin 编译器编译成 .class 文件
- 将所有编译生成的 .class 文件（包括应用代码和第三方库的 .class 文件）汇集到一个统一的输入目录或列表中

4. 字节码转换阶段
- Dex 转换
    - 使用`D8`或`dx`工具将`.class`文件转换成`Dalvik`可执行格式`.dex`文件
- 多Dex 支持
    对于方法数超过限制的应用，启用`MultiDex`支持

5. 代码优化阶段
- ProGuard /R8 混淆和优化
    - 混淆类名、方法名、减小代码体积
    - 删除未使用的代码和资源

6. 打包阶段
- aapt2打包
    将处理好的资源包(resources.ap_)、dex文件、Native库(.so)、资产文件等打包成APK文件
- APK 结构
    classes.dex：Dalvik字节码
    res/:编译后的资源
    AndroidManifest.xml：应用清单(已编译成二进制格式)
    lib/：Native库
    assets/：原始资产文件
    META-INF/:签名相关文件

7. 签名阶段
APK签名
    使用开发者的签名证书对APK进行签名
    确保APK的完整性和来源可信

8. 对齐阶段
zipalign  
    对APK文件进行字节对齐优化
    提高运行时读取效率，减少内存消耗
