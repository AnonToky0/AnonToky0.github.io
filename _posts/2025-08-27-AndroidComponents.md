---
title: "Android 四大组件"
date: 2025-08-27 15:00:00 +0800
categories: [Android]
tags: [Android]
---

#### 引用
1. [Android 四大组件详解](https://blog.csdn.net/joye123/article/details/116197862)

## 四大组件简介
Android系统应用层框架为开发者提供了四大组件便于开发，分别是：`Activity`, `Service`, `BroadcastReceiver`, `ContentProvider`。用于在应用开发过程中，不同场景的功能实现。
- Activity
和用户交互，创建显示窗口，通过`setContentView`显示特定UI
- Service
处于后台长时间运行，用来后台执行耗时任务或为其他APP提供功能调用的服务。
- BroadcastReceiver
类似于观察者模式，开发人员可以添加自己关心的广播事件，从而为用户提供更好的使用体验。如网络连接变化、电池电量变化、开机启动等。
- ContentProvider
用来在不同应用程序之间共享数据。如果仅仅是在一个程序中存取数据，可以用`SQLiteDatabase`


### 系统如何管理四大组件
在使用前在`AndroidManifest.xml`中进行申明。`BroadcastReceiver`可以动态注册。

在应用程序安装时，应用安装程序通过`PackageInstaller`服务解析应用安装包，并将`AndroidManifest.xml`中声明的四大组件信息保存到`PackageManagerService`中

### 什么是Context
Context是关于APP环境的全局信息接口，是一个抽象类，实现都是由系统类实现的。Context允许访问APP特定的资源和类，如：
- 资源管理器AssetsManager、
- 包管理器PackageManager、
- 文本图片主题资源Resource、
- 主线程消息循环Looper
- 四大组件操作，如启动Activity、BroadcastReceiver，接收Intent
- SharedPreferences操作
- 私有目录文件操作
- 数据库创建、删除操作
- 获取系统服务
- 权限操作

Context的实现类为ContextImpl，ContextImpl为Activity或其他应用程序组件提供了基础的Context实现。

除了ContextImpl类外，ContextWrapper类也继承了Context，但是ContextWrapper类并不自己实现Context的方法，而是通过构造方法，代理给另外一个Context的实现。**代理模式**。这样ContextWrapper的子类就可以在不修改ContextWrapper类的情况下，修改其调用方法的实现。
> 这里说的另一个Context指的是ContextWrapper通过构造函数传入一个具体的 Context 实例（例如 ContextImpl 或其他 Context 的子类实例）

在开发中用到的Activity、Application、Service类都是集成自ContextWrapper类。  
一个App主进程中`context`对象的实例个数，除了`Application`,`Activity`,`Service`等，可能还有创建基于 `ContextWrapper` 的 Context 实例，比如 `ContextThemeWrapper`、`ContextWrapper` 本身的实例。  
> 请注意：四大组件不是都继承自context， BroadcastReceiver 直接继承自 `android.content.BroadcastReceiver`，ContentProvider 继承自 `android.content.ContentProvider`

在构建出实例后，通过ContextWrapper的attachBaseContext(Context)方法，将真正的Context实现类添加进去。这样就可以动态控制Activity、Application、Service类调用Context的方法实现。例如：

``` java
public class MyActivity extends Activity {
    @Override
    protected void attachBaseContext(Context newBase) {
        super.attachBaseContext(newBase);
        // 这里可以做一些额外操作，比如替换 Context，或者做日志记录
        Log.d("MyActivity", "attachBaseContext called");
    }

    @Override
    public Resources getResources() {
        // 在调用真实 Context 的 getResources() 之前，做点自己的处理
        Log.d("MyActivity", "getResources() called");
        return super.getResources();  // 实际调用的是 ContextWrapper 持有的真实 Context 的 getResources()
    }
}
```
运行流程简述：
- 系统创建`ContentImpl`
- 系统创建 MyActivity 实例(继承自 ContextWrapper)。
- 系统调用 MyActivity.attachBaseContext(ContextImpl)，将真实 Context 传入 ContextWrapper
- 当调用 MyActivity.getResources() 时
    - 先执行 MyActivity 重写的 getResources()，打印日志。
    - 再调用 super.getResources()，即调用 ContextWrapper 中的 getResources()。
    - ContextWrapper 通过持有的真实 Context（ContextImpl）的引用，调用它的 getResources() 方法，返回真正的资源

这是一种典型的**代理模式**应用，既保证了系统功能的统一实现，也方便开发者扩展和定制。

### 四大组件的实例化过程
通过ActivityThread，通过反射调用无参构造函数创建，如
``` java
Class<?> clazz = Class.forName(componentClassName);
Object instance = clazz.newInstance();  // 调用无参构造函数
```
#### AppComponentFactory
ActivityThread 实例化四大组件存在以下问题：
- 无法直接传递参数给组件的构造函数。
- 只能依赖无参构造函数和生命周期方法（如 onCreate()）来完成初始化。
- 如果你想“Hook”或“拦截”这个创建过程，传统方式很难实现。

`AppComponentFactory`在Android 9（API 28）引入，是一个工厂类，负责创建四大组件的实例。
- 系统在创建组件实例时，不再直接调用 Class.newInstance()，而是调用 AppComponentFactory 中对应的方法（比如 instantiateActivity()、instantiateApplication() 等）。
- 可以继承 AppComponentFactory，重写这些方法，控制组件的实例化过程，比如：
    - 返回自定义的组件实例（可以带参数的构造函数）。
    - 在创建实例前后做额外逻辑（日志、替换类、注入依赖等）。

### Activity、ContentProvider、Service三者的启动顺序
- 一个新的应用启动时，会优先初始化其Application类，创建了Application实例后，会立即调用其attach方法。
- 然后就会初始化应用中声明的ContentProvider。
- ContentProvider初始化完成后，会再调用Application类的onCreate方法。
- AMS在初始化完客户端的Application类后，会检查是否有需要运行的Service和BroadcastReceiver。

所以初始化顺序是Application.attach() -> ContentProvider -> Application.onCreate() -> Activity/Service/BroadcastReceiver。

## Application
Application 是 Android 应用程序的入口类，它代表整个应用程序的全局状态和环境。
- 系统启动应用时，会先创建 Application 类的实例。一个Android应用的主进程中，Application对象只有**一个实例**。
- Application 生命周期贯穿整个应用进程的生命周期，通常用于：
    - 初始化全局资源（比如单例、数据库、网络库）
    - 保存全局状态
    - 监听应用级别的生命周期事件（比如前后台切换）
- 多进程情况下，每个进程有**各自独立**的Application实例。
#### Application.attach()
主要用于:
- 绑定 Application 对象和应用的主线程（ActivityThread）；
- 传入系统的 Context 和其他必要的环境信息；
- 准备 Application 对象，使其能正常访问系统服务、资源等

- **调用时机**
    - 由系统启动应用时的主线程（ActivityThread）调用，通常在 ActivityThread.handleBindApplication() 中。
    - 这个调用发生在 Application 实例创建之后、onCreate() 方法之前。

## Activity
界面组件，生命周期由系统管理，负责和用户交互，创建显示窗口，显示UI等。

### attach()
用于完成 Activity 实例与系统框架的“绑定”工作。并不是 Activity 生命周期中的公开回调（比如 onCreate()），而是系统内部调用的方法。  
attach() 负责把 Activity 这个 Java 对象与系统底层的各种服务和资源（如窗口管理、上下文、线程、调度器等）连接起来，准备好后续的生命周期调用和 UI 渲染。
#### 调用时机
- 当客户端进程接收到 AMS 通过 Binder 通知启动 Activity 的请求后，由 ActivityThread（客户端进程中的主线程调度器）创建 Activity 实例。
- 创建好 Activity 对象后，第一步就是调用 attach()，完成 Activity 与系统环境的绑定。
- 之后才依次调用 onCreate()、onStart()、onResume() 等生命周期方法。

#### 主要工作内容
- 关联 Activity 的 Application 对象。
- 设置 Activity 的 Window（窗口）和 WindowManager。
- 关联 Instrumentation（用于监控和控制 Activity 生命周期的工具）。
- 绑定 ActivityThread（主线程）和 Handler，方便消息调度。
- 绑定 ActivityToken（AMS 传递的标识符，用于跨进程识别该 Activity）。
- 设置上下文环境和资源管理。
- 准备好 Activity 的视图层次结构和事件分发机制

### Activity生命周期
主要包括以下几个阶段，每个阶段对应一个或多个回调方法

| 生命周期状态    | 典型回调方法         | 说明                  |
|---------------|----------------------|------------------------------|
| **创建**       | `onCreate()`       | 初始化界面和数据                   |
| **启动**       | `onStart()`        | Activity 即将对用户可见            |
| **恢复（运行）**     | `onResume()`     | Activity 进入前台，用户可以交互    |
| **暂停**      | `onPause()`       | Activity 失去焦点，但仍部分可见    |
| **停止**      | `onStop()`         | Activity 完全不可见                |
| **重启**      | `onRestart()`        | Activity 从停止状态重新启动        |
| **销毁**       | `onDestroy()`      | Activity 被销毁，释放资源          |

#### onCreate(Bundle savedInstanceState)
- 调用时机：Activity 第一次创建时调用。
- 作用：
    - 初始化界面(`setContentView`)，
    - 初始化数据和成员变量
    - 绑定事件监听器
    - 读取 savedInstanceState 恢复状态
- **只调用一次**

#### onStart()
调用时机：Activity 即将对用户可见时调用（紧接 onCreate 或 onRestart 后）。
- 作用：
    - 准备让界面显示给用户
    - 可能开始动画或其他视觉效果

#### onResume()
- 调用时机：Activity 进入前台，用户开始交互时调用。
- 作用：
    - 恢复界面交互
    - 启动需要持续运行的资源（如摄像头、传感器监听）
    - 注意：Activity 处于前台，用户可以操作。

#### onPause()
- 调用时机：系统准备启动或恢复另一个 Activity，当前 Activity 失去焦点时调用。
- 作用：
    - 暂停动画、音频等占用 CPU 的资源
    - 保存临时数据（如编辑内容）
    - 释放摄像头等独占资源
- 注意：
    - 该方法执行时间必须很短，避免阻塞新 Activity 启动。
    - Activity 可能仍部分可见（如透明 Activity 覆盖）。

#### onStop()
- 调用时机：Activity 完全不可见时调用。
- 作用：
    - 释放较重的资源（如网络连接）
    - 保存持久数据
- 注意：Activity 仍然保留在后台，可以通过 onRestart() 重新启动。

#### onRestart()
- 调用时机：Activity 从停止状态重新启动时调用。
- 作用：
    - 准备重新显示界面
- 注意：紧接着调用 onStart()。

#### onDestroy()
- 调用时机：Activity 被系统销毁时调用（如用户退出或系统回收资源）。
- 作用：
    - 释放所有资源
    - 进行最终清理
- 注意：不保证一定调用（如系统强制杀进程）。

### Activity 示例场景

| 场景描述                               | 调用顺序                             |
|--------------------------------------|------------------------------------|
| 应用启动，Activity 创建显示            | `onCreate()` → `onStart()` → `onResume()` |
| 用户按 Home 键，Activity 进入后台      | `onPause()` → `onStop()`            |
| 用户返回应用，Activity 重新显示        | `onRestart()` → `onStart()` → `onResume()` |
| 用户按返回键，Activity 结束            | `onPause()` → `onStop()` → `onDestroy()` |

### Activity 创建
1. 通过 Activity 或 Context（具体实现是 ContextImpl）调用 startActivity(Intent) 来启动另一个 Activity。
- 关键区别：
    - 如果调用者是一个 Activity 上下文，startActivity() 会直接启动目标 Activity。
    - 如果调用者是非 Activity 上下文（例如 Application Context），则必须在 Intent 中指定 FLAG_ACTIVITY_NEW_TASK 标志，否则会抛出异常。
2. 经过 Instrumentation 监控类转发
- Instrumentation 是 Android 框架中的一个监控和控制入口，负责监控 Activity 的启动、生命周期等。
- startActivity() 调用最终会通过 Instrumentation.execStartActivity() 方法转发，Instrumentation 会做一些额外的处理（如测试、性能监控等），然后将启动请求传递给系统。
3. 通过IPC调用到ActivityManagerService(AMS)中
    - Android 采用客户端-服务端架构，所有应用进程通过 Binder 机制与系统服务通信。
    - Activity 启动请求通过 Binder IPC 调用发送到系统进程中的 ActivityManagerService（AMS），这是管理所有 Activity 生命周期和任务栈的核心系统服务。
4. 根据 Intent 从 PackageManagerService(PMS) 中找出对应的 Activity 信息
- AMS 根据传入的 Intent，需要找到对应的目标 Activity。
- 为此，AMS 会调用 PackageManagerService（PMS），这是系统中负责管理已安装应用及其组件信息的服务。
- PMS 在应用安装时会解析应用的 AndroidManifest.xml，提取四大组件（Activity、Service、BroadcastReceiver、ContentProvider）信息并缓存。
- AMS 通过 PMS 查询目标 Activity 的详细信息（如类名、权限、启动模式等）。
- 找到目标 Activity 对应的进程 ProcessRecord，通过 ApplicationThread 这个 Binder 接口，告知客户端进程创建指定的 Activity。

### 客户端进程启动Activity 流程
- AMS中`startActivityLocked`通过Binder调用应用进程的ActivityThread 中的启动入口（scheduleLaunchActivity）通知应用进程启动 Activity
- 收到 AMS 通过 Binder 发送的启动 Activity 消息。
- 创建对应的 Activity 实例。
- 按顺序调用生命周期方法：
    - attach()
    - onCreate()
    - onStart()
    - onRestoreInstanceState()
    - onPostCreate()
    - （如有新 Intent，调用 onNewIntent()）
    - onResume()
- UI 显示相关：
- 创建 Window、DecorView、ViewRootImpl。
- 通过 WindowManagerService 添加 DecorView。

#### 创建和启动关键类
1. ProcessRecord
- 代表一个当前运行的进程的全部信息。
- 在创建新进程之前，由 ActivityManagerService（AMS）创建 ProcessRecord 实例。
- 存储在AMS中的`ProcessMap`
- IApplicationThread thread：Binder 接口，用于 AMS 向客户端进程发送控制消息（如启动 Activity、杀死进程等）

2. ActivityRecord
代表任务栈中的一个 Activity 实例。
- 包含信息：
    - 从 AndroidManifest.xml 解析出的 Activity 信息。
    - Activity 所属的 Application 信息。
    - 真实的 Activity 组件信息（类名等）。
    - Activity 所运行的进程名称、包名。
    - 与 WindowManager 交互用的 token（窗口标识）。
    - 所属的 AMS 实例。
    - 启动该 Activity 的调用者。
    - 当前运行状态，如 stopped、finishing 等。

3. TaskRecord
- 代表一个任务栈（Task），包含多个 ActivityRecord。
- 用于管理一个任务栈中所有的 Activity 记录。

4. ActivityStarter
AMS 的辅助类，负责处理启动 Activity 的具体逻辑。
- 根据 Intent 和启动 Flags 决定如何启动 Activity。
- 处理权限检查、运行时权限申请。
- 处理特殊场景，如语音交互、延时启动（例如从悬浮窗返回时延迟启动）。
- 创建新的 ActivityRecord，承载启动请求的后续操作。

5. ActivityStack
管理一个应用中的所有 TaskRecord，以及所有 ActivityRecord 的状态变化。
- 找到合适的 TaskRecord 来放置新启动的 Activity。
- 管理 Activity 生命周期状态的切换。

6. ActivityStackSupervisor
管理系统中所有的 ActivityStack。
- 维护多个 ActivityStack，如桌面应用的 mHomeStack，当前活跃的 mFocusedStack。
- 处理启动新 Activity 时的调度逻辑。
- 通过 `ApplicationThread` Binder 接口通知客户端进程创建 Activity 实例。

7. Token (IApplicationToken.Stub 的子类)
AMS 与客户端进程之间用来标识和操作特定 ActivityRecord 的标记。
- 双方通过传递 Token 来确认操作的是哪个 Activity。

### LaunchMode
`launchMode` 是 `AndroidManifest.xml` 中 `<activity>` 标签的一个属性，用来控制启动 Activity 时系统如何创建和管理该 Activity 实例及其所在的任务栈（Task）。  
Android 一共有四种 LaunchMode:
- standard（默认）
- singleTop
- singleTask
- singleInstance

1. standard
- 每次启动该 Activity，系统都会创建一个新的实例，放入当前任务栈的栈顶。
- 不管当前任务栈中是否已有该 Activity 的实例，都会新建一个。
2. singleTop
- 如果当前任务栈的栈顶已经是该 Activity 的实例，那么不会重新创建新的实例，而是复用栈顶的这个实例，调用它的 onNewIntent() 方法。
- 如果栈顶不是该 Activity，则会创建新的实例放入栈顶。  
- 适用场景：通知栏点击打开的 Activity，避免重复创建。
3. singleTask
- 系统会检查整个任务栈中是否已经存在该 Activity 的实例。
- 如果存在，则将该 Activity 之上的所有 Activity 弹出（销毁），并复用该 Activity 的实例，调用 onNewIntent()。
- 如果不存在，则新建一个实例，并放入**新的任务栈**的栈顶。
该 Activity 总是作为任务栈的根（栈底）Activity。
- 全局只有一个该 Activity 实例。
- 适用场景：比如启动主界面 Activity，或者某些需要单例的界面。
4. singleInstance
- 该 Activity 独占一个任务栈，启动它会创建一个新的任务栈。当启动Acitivity时如果实例存在，会复用该实例，并通过 onNewIntent() 传递新的 Intent。
- 其他 Activity 不会加入这个任务栈。
- 全局只有一个该 Activity 实例。


### Activity启动模式标志(Flags)
这些 Flags 在代码中启动 Activity 时动态传入的标志，用来临时改变启动行为。启动时 Intent 中的 flags 优先级**高于** launchMode。
- FLAG_ACTIVITY_SINGLE_TOP  
    如果要启动的 Activity 已经位于当前任务栈的栈顶（top），则不会重新创建新的实例，而是复用栈顶的 Activity，调用它的 onNewIntent() 方法。
    - 示例：  
    当前任务栈顶是 Activity A，再次启动 Activity A 并加上此标志，则不会创建新实例，而是复用栈顶的 Activity A。
- FLAG_ACTIVITY_SINGLE_TASK  
    启动的 Activity 在任务栈中只允许有一个实例。
    - 行为：如果任务栈中已有该 Activity 的实例，会将该实例之上的所有 Activity 出栈（finish），并调用该实例的 onNewIntent()。
    - 作用场景：用于启动需要单例的 Activity，比如首页、主界面。
- FLAG_ACTIVITY_SINGLE_INSTANCE  
    该 Activity 独占一个任务栈（Task），系统保证该 Activity 只有一个实例，并且该任务栈中只有它自己。
- FLAG_ACTIVITY_CLEAR_TOP  
    启动 Activity 时，如果目标 Activity 已经存在于任务栈中，则将其之上的所有 Activity 出栈，并调用目标 Activity 的 onNewIntent()。
- FLAG_ACTIVITY_REORDER_TO_FRONT  
    当启动的 Activity 已经存在于当前任务栈中（但不一定在栈顶），系统会将该 Activity 移动到栈顶，而不是创建新的实例。

### 复用情况下的声明周期

| 情况    | 调用的方法   | 说明            |
|--------------|------------------------|----------------------|
| **新建 Activity**   | `onCreate()` → `onStart()` → `onResume()` | Activity 实例第一次创建时调用    |
| **复用已有实例（如 singleTop）** | `onNewIntent(Intent intent)` → `onRestart()`（如果之前停止） → `onStart()` → `onResume()` | 复用实例且收到新 Intent 时调用 |

#### AMS启动流程
综上，启动流程大致为：
- 根据启动参数（Intent、Flags 等）确定目标 Activity 应该在哪个任务栈（TaskRecord）中。
- 找到或创建合适的 TaskRecord。
- 将目标 Activity 添加到这个 TaskRecord 的 Activity 列表中（即任务栈中）。
- 更新 ActivityStack 的状态，切换栈顶 Activity 等。

##### 其他注意事项
- **判断条件**
启动一个 Activity 并不一定意味着这个 Activity 会立刻进入 Resumed（前台可交互）状态。系统需要根据当前状态和启动参数，决定是否让它立刻获得焦点（用户输入焦点），或者先让它处于可见但不活跃的状态。
1. 目标 Activity 是否允许获取焦点（是否设置了 FLAG_ALWAYS_FOCUSABLE）
- 如果目标 Activity 没有设置这个标志，意味着它可能是一个“非焦点”窗口（比如透明 Activity、对话框样式的 Activity，或者某些只显示但不接收输入的界面）。
- 这种情况下，系统不会马上让它进入前台交互状态（Resume），而是让它处于可见状态，但不抢占当前 Activity 的焦点。

2. 当前活动的 Activity 是否一直处于顶层且未完成（未调用 finish()）
- 如果当前栈顶的 Activity 还在运行（没有调用 finish()），它就是用户当前正在交互的界面。
- 在这种情况下，如果新启动的目标 Activity 不满足“总是允许获得焦点”，系统会避免立刻切换焦点，防止用户体验被打断。

### Activity的实例化过程
1. 当应用或系统通过 Intent 启动一个 Activity 时，ActivityManagerService（AMS，系统服务）会接收到启动请求。
2. AMS 会检查该应用进程是否已经存在：
    - 不存在，启动应用进程(Zygote fork出新进程)
    - 存在， 直接使用该进程
3. Binder通信
AMS 通过Binder IPC调用对应进程内的ActivityThread 的 HandleLaunchActivity() 方法
4. ActivityThread 创建Activity 实例  
`ActivityThread`是应用主线程的调度器，负责管理Activity生命周期。
- 在HandleLaunchActivity()中，通过反射调用 Activity的构造函数，创建Activity实例。
- 调用 attach() 方法，将 Activity和ActivityThread、Instrumentation、Window 等系统对象关联起来。
- 通过 attachBaseContext() 绑定底层的 ContextImpl。

## Service
- 使用场景：
    - 后台执行长时间的任务，而不需要与用户交互。例如后台播放音乐
    - 将本应用的功能通过接口的形式暴露给其他应用程序调用

- 两种启动Service的方式：
1. startService，Service会执行onCreate和onStartCommand。多次启动同一个Service不会多次执行onCreate，会多次执行onStartCommand
2. bindService，Service会执行onBind方法，返回给调用者一个Binder对象，用于功能接口调用

- Service优先级：
Service所在进程的优先级仅次于前台进程的优先级，系统会尽量避免杀死该进程，除非内存压力非常大。如果被系统杀死了，系统会稍后尝试重启该Service，并重新将Intent数据发送Service来恢复杀死之前的状态。

onStartCommand方法的返回值，决定Service被系统杀死后的操作：  
- START_NOT_STICKY 被系统杀死后不会被重启
- START_STICKY 被系统杀死后会重建，但是会发送一个null给onStartCommand
- START_REDELIVER_INTENT 被系统杀死后会重建，并且会逐一发送等待处理的Intent给onStartCommand

### startService
startService(Intent service) 是 Android 中启动 服务(Service) 的一种方式。
- 启动一个服务组件（Service），使其进入 启动状态（Started）。
- 服务会在后台持续运行，即使启动它的组件（如 Activity）销毁，服务仍然保持运行，除非调用 stopService() 或服务自己调用 stopSelf()。

#### startService 调用流程
1. 应用调用 Context 的 startService(Intent) 方法  
传入一个 Intent，指定要启动的服务组件（通过 ComponentName 或 Action 等）。
2. 调用到 ActivityManagerService（AMS）
    - 进程通过 Binder 调用系统进程中的 AMS 的 startService 方法。
    - AMS 会检查调用者是否有权限启动该 Service。
        - 如果没有权限，AMS 会返回一个特殊的 ComponentName（包名为 !/!!），表示启动失败。
3. AMS 处理启动请求，转到 ActiveServices 类
`ActiveServices` 是 AMS 中管理所有运行中 Service 的核心类。
- 如果目标 Service 所在进程存在且 Service 已启动：
    - AMS 不需要启动新进程或重新创建 Service。
    - AMS 会通过 Handler 发送一个超时消息，如果客户端处理超时，AMS 可以通过这个超时消息执行相应处理（比如杀死无响应进程，释放资源）。
    - 超时时间通常为 20 秒，如果 Service 进程处于后台，超时时间会延长到 200 秒（20*10秒），这主要是为了适应后台进程调度策略。

4. 如果目标 Service 所在进程不存在
- AMS 需要先启动目标进程。
- 启动进程后，客户端进程 会将 ApplicationThread的bind接口发送给 AMS，AMS 通过这个接口告知客户端初始化Application 类。
- Application 初始化完成后，AMS 会检查是否有等待启动的 Service 或 BroadcastReceiver。
- **等待启动的Service**
在ActiveServices类中完成,存在两个请求列表
- 启动请求列表：存放在进程未启动时，等待启动的 Service 请求。
- 重新启动请求列表：存放系统因 Service 意外被杀死后，根据 onStartCommand 返回值（START_STICKY 或 START_REDELIVER_INTENT）需要重启的 Service。

5. Service 第一次启动
- 当进程启动后，客户端进程会执行第一次启动 Service 的逻辑：
- 创建 Service 实例：
    - 客户端进程（应用进程）通过 Binder 通信，AMS 调用客户端进程的 ActivityThread 的 handleCreateService() 方法
    - handleCreateService() 是 ActivityThread 里启动 Service 的入口。
- 调用 Service 的 attach() 方法，将 Service 绑定到系统环境（Context、Handler、ActivityThread - 等）。
- 调用 Service 的 onCreate() 生命周期方法。
- ActivityThread.handleStartService() 调用 ->Service.onStartCommand() 处理启动请求
- 通知 AMS Service 创建完成。

4. 服务的生命周期回调
    - 系统调用服务的 onCreate()（如果是第一次启动）。
    - 调用 onStartCommand(Intent intent, int flags, int startId)，传入启动的 Intent。
5. 服务进入启动状态，开始执行后台任务

> scheduleServiceArgs：把客户端进程的Service启动请求的参数打包，方便 AMS 统一处理
##### 客户端进程如何处理重复调用 scheduleServiceArgs
- AMS 通过调用 scheduleServiceArgs 向客户端进程传递启动请求。
- 客户端根据传入的 token 获取对应的 Service 实例。
- 判断是否是任务删除调用（即是否是停止服务相关的请求）。
- 调用 Service 的 onStartCommand 方法，传入启动参数。
- 客户端调用完成后，通知 AMS 操作完成，AMS 收到后会移除超时消息。

## startService生命周期
1. `onCreate()`
- 调用时机：服务第一次被创建时调用
- 作用：进行一次初始化操作

2. `onStartCommand(Intent intent, int flags, int startId)`
- 调用时机：每次通过`startService`启动服务时都会调用
- 作用：处理启动请求，执行具体任务
- 返回值：控制服务被杀死后系统的重启行为，常见返回值包括：
    - `START_NOT_STICKY`：服务被杀死后，不重启
    - `START_STICKY`：服务被杀死后，系统尝试重启服务，但不传递原始Intent
    - `START_REDELIVER_INTENT`：服务被杀死后，系统重启服务并重新传递最后一个Intent(多次调用 startService(intent) 启动同一个服务时最新的Intent)

3. 服务运行中
- 服务在后台执行任务，可能会启动线程或定时器等。

4. `onDestroy()`
- 调用时机：服务被停止时调用
- 作用：释放资源、停止线程等清理工作
- 调用方式：
    - 通过 stopSelf() (服务自身调用) 或 stopService() (启动服务的外部组件调用)停止服务时调用。
    - 系统在资源紧张时也可能杀死服务，但不会调用 onDestroy()(而是强制杀死进程)。

### bindService
#### bindService 和 startService区别
- startService：
    - 仅启动 Service（如果没启动就创建并启动）。
    - 不提供通信接口，客户端与 Service 是松耦合的。
    - 主要调用生命周期方法：onCreate()（首次启动时），onStartCommand()（每次启动请求时）。
- bindService：
    - 绑定到 Service，建立客户端和 Service 的通信通道。
    - 需要提供一个 ServiceConnection 回调接口，通知客户端绑定成功或断开。
    - AMS 处理流程和 startService 类似，也会调用 bringUpServiceLocked，启动 Service（如果没启动）。

#### 启动流程相同点
无论客户端调用startService还是bindService，AMS 收到请求后，都会调用 bringUpServiceLocked 来“启动”或“唤起”目标服务。
> bringUpServiceLocked 是AMS类中的一个私有方法。
> bringUpServiceLocked 负责启动服务的整个流程，包括：
    > - 启动服务进程（如果服务还没运行）。
    > - 创建并初始化 Service 对象（调用 attach() 和 onCreate()）。
    > - 调用 onStartCommand() 或 onBind()，根据启动方式（startService 或 bindService）执行对应的回调。
    > - 维护服务的生命周期状态。
因此，这两种启动方法都会经历：  
- `attach()`：将Service绑到系统环境
- `onCreate()`：Service初始化

#### 绑定流程和回调
##### 第一次绑定
客户端首次调用 bindService，AMS 会：  
- 启动 Service（如果未启动）。
- 调用 Service 的 onBind(Intent) 方法，返回一个 Binder 对象（通信接口）。
- AMS 将这个 Binder 返回给客户端，客户端通过这个 Binder 与 Service 通信。

##### 重复绑定(多个客户端绑定同一个 Service)
- 因为已经有了 Binder 接口，直接把之前的 Binder 返回给新的绑定客户端即可。
- 这样避免重复创建 Binder，节省资源。

##### 重新绑定（客户端解绑后又绑定）
- 这时 AMS 不需要重新获取 Binder(因为之前已经有了)。
- AMS 会调用 Service 的 onRebind(Intent) 方法，通知 Service 有客户端重新绑定。
- 只有当 Service 在 onUnbind(Intent) 中返回 true，才会收到 onRebind() 回调。
- 如果 onUnbind() 返回 false，则不会调用 onRebind()

### Service启动时的生命周期
- 多次调用startService：
    - 执行一次 onCreate()
    - 多次执行 onStartCommand()

- 多次调用startService:
    - 执行一次 onCreate()
    - 执行一次 onBind()
    - 调用方的 ServiceConnection()也只会回调一次

### Service 和 AMS 的调用时序
<!-- <img src="{{ '/assets/img/posts/2025-08-27-AndroidComponents/service_process.png' | relative_url }}" alt="service_process"/> -->

| 客户端进程           | AMS进程                                  | Service进程
|----------------------|-------------------------------------|-------------- |
| 第一次startService   | 记录请求事项，创建目标进程 | 进程创建成功 |
|                      | 创建目标Service | 回调onCreate方法 |
|                      | 发送调用参数 | 回调onStartCommand方法 |
| 第二次startService   | 发送调用参数 | 回调onStartCommand方法 |

| 客户端进程           | AMS进程                                  | Service进程
|----------------------|-------------------------------------|-------------- |
| 第一次bindService   | 记录请求事项(Connection)，创建目标进程 | 进程创建成功 |
|                      | 创建目标Service | 回调onCreate方法 |
|                      | 绑定目标Service | 回调onBind方法 |
|                      | 获取并存储目标Service的Binder接口 |              |
| 回调onServiceConnected |                                 |              |
| 第二次bindService(不同的ServiceConnection)   | 查询存储的Service Binder接口记录 |     |
| 回调onServiceConnected |                                   |              |
| 第二次bindService(相同的ServiceConnection)   | 查询存储的Service Binder接口记录 |     |

> 相同的 ServiceConnection 对象，如果已经绑定成功，再次调用 bindService 并不会重复触发 onServiceConnected。

### bindService生命周期
bindService允许客户端(通常是Activity)与服务通信。

#### 基本流程
- 客户调用`bindService`绑定服务
- 系统调用服务的`onBind`方法，返回一个`IBinder`对象
- 客户端通过`ServiceConnection`的回调获取`IBinder`，与服务通信
- 当客户端不再需要服务时，调用`unbindService`解除绑定

#### 服务端流程
onBind(Intent intent)
当客户端调用bindService()时触发


## ContentProvider
用来在不同**应用程序**之间共享数据。如果仅仅是在一个程序中存取数据，可以用`SQLiteDatabase`。
- 它在内部实现了跨进程的调用，不需要数据提供者或调用者关心。
- 调用者通过URI向ContentProvider查询数据。

## BroadcastReceiver
类似于观察者模式，开发人员可以添加自己关心的广播事件，从而为用户提供更好的使用体验。如网络连接变化、电池电量变化、开机启动等。

### 注册方式
#### 静态注册
- 在`AndroidManifest.xml`中声明`<receiver>`节点。
- 配置`<intent-filter>`指定接收的广播类型(Action)。
``` xml
<receiver android:name=".MyBroadcastReceiver">
    <intent-filter>
        <action android:name="android.intent.action.BOOT_COMPLETED" />
        <action android:name="android.intent.action.AIRPLANE_MODE" />
    </intent-filter>
</receiver>
```
- 生命周期：由系统管理，应用未启动时也能接收广播。
- 适用场景：接收系统广播（如开机完成、网络变化等）。

注册后，当意图过滤器中的动作发生时，会回调BroadcastReceiver中的onRecevie方法.
``` java
public class MyTestReceiver extends BroadcastReceiver {
    private static final String TAG = "MyTestReceiver";

    @Override
    public void onReceive(Context context, Intent intent) {
        if (intent != null) {
            String action = intent.getAction();
            Log.d(TAG, "onReceive: receive action is " + action);
        }
    }
}
```

#### 动态注册
- 在代码中通过 Context.registerReceiver() 方法注册广播接收器。
- 需要传入 BroadcastReceiver 实例和 IntentFilter
``` java
IntentFilter intentFilter = new IntentFilter();
intentFilter.addAction(Intent.ACTION_SCREEN_OFF);

registerReceiver(new BroadcastReceiver() {
  @Override
  public void onReceive(Context context, Intent intent) {
      String action = intent == null ? "" : intent.getAction();
      if (Intent.ACTION_SCREEN_OFF.equalsIgnoreCase(action)) {
          Log.i(TAG, "onReceive: 屏幕关闭");
      }
      context.unregisterReceiver(this);
  }
}, intentFilter);
```
注册的BroadcastReceiver最好是静态类，防止内存泄漏。否则要及时取消注册。

- 生命周期：绑定在注册它的组件生命周期内（如 Activity、Service），组件销毁时需手动注销。
- 适用场景：接收应用内部广播或需要在特定时机监听的广播。

### 注册原理
#### 静态注册原理
当PMS解析AndroidManifest.xml时，对于`<receiver>`标签，PMS 会将广播接收器的信息（类名、intent-filter 等）解析出来，并存入系统的包管理数据库中。
- 这些信息被系统记录后，系统就知道该应用中有哪些静态注册的广播接收器，以及它们能接收哪些广播。

- PMS 会将解析出的 Receiver 信息存储到内存中的数据结构（如 PackageParser.Package 类的 receivers 集合）和持久化数据库中。
- 这样，当系统广播发送时，AMS(ActivityManagerService)和系统广播管理器（BroadcastQueue）可以快速匹配到对应的静态注册广播接收器，进行广播分发。

- PackageParser 类负责解析 APK 的 AndroidManifest.xml
- BroadcastReceiver 在PMS 解析时会被当做Activity处理

#### parseActivity 方法解析流程
- parseActivity() 会解析 `<activity>` 标签，也会解析 `<receiver>` 标签（源码中可能会有类似parseReceiver()，但结构类似）。
- 解析过程中，会重点处理 `<intent-filter>` 标签，获取该组件能接收的 Intent 类型。
- `<intent-filter>` 中的子标签包括：
- `<action>`：定义该组件能响应的动作（Action）。
- `<category>`：定义类别，进一步限定 Intent 匹配。
- `<data>`：定义数据类型、Scheme、Host、Path 等，支持更细粒度匹配。

#### 数据存储到 Package 类
- 解析出的组件信息会封装成对应的对象（如 Activity, Receiver）。
- 这些对象被统一存储到 Package 类中，Package 类是代表整个 APK 的数据结构。
- Package 类中有多个集合字段分别存储：
    - activities：存储所有 Activity 信息。
    - receivers：存储所有 BroadcastReceiver 信息。
    - services：存储所有 Service 信息。
    - providers：存储所有 ContentProvider 信息。

#### 动态注册原理
当应用通过Context.registerReceiver(BroadcastReceiver receiver, IntentFilter filter)动态注册时：
- 会先发送 IPC（跨进程通信）请求给 ActivityManagerService（AMS）。
- AMS 会将该广播接收器和它对应的 IntentFilter 信息保存起来，通常保存于 AMS 管理的广播接收器列表中。
- 这个列表是运行时动态维护的，包含所有动态注册的广播接收器。

发送广播时：  
- 当系统或应用发送广播时，广播请求会先到 AMS。
- AMS 会从两个地方查找匹配的广播接收者：
    - 动态注册的广播接收器列表（AMS 维护的动态注册表）
    - 静态注册的广播接收器列表（由 PackageManagerService 管理，来源于 Manifest 文件）
- AMS 根据广播的 Intent，匹配所有符合 IntentFilter 的广播接收器，并将广播投递给它们。

##### 取消动态注册
- 动态注册的广播接收器是运行时注册的，AMS 维护它们的列表。
- 当调用 Context.unregisterReceiver(BroadcastReceiver receiver) 时：
- 会通过 IPC 通知 AMS，从动态广播接收器列表中移除该接收器。
    移除后，该广播接收器将不再接收广播

##### 如何间接取消静态注册广播
- 静态注册的广播接收器写在 AndroidManifest.xml 中，安装时由 PackageManagerService（PMS）解析并记录。
- 但系统设计了**组件状态管理（Component State）**机制，可以启用或禁用某个组件。

- PMS 维护一个组件状态记录（通常在 PackageSetting 中），其中保存了组件（Activity、Receiver、Service 等）当前的启用/禁用状态。
- 这些状态会覆盖 Manifest 中的默认声明，实现对组件的动态控制。
- 当 AMS 或 PMS 查询某个 Intent 对应的广播接收器列表时：
IntentResolver 会检查每个广播接收器对应组件的状态。
如果组件状态是“禁用”，则该广播接收器会被过滤掉，不会返回给 AMS。

###### 修改流程
调用 PackageManager 的接口（如 setComponentEnabledSetting()）来修改组件状态。PMS 会：
- 修改 Setting 中对应组件的状态为“禁用”。
- 发送一个系统广播 Intent.ACTION_PACKAGE_CHANGED，通知 AMS 组件状态发生变化。
- AMS 收到该广播后。
    - 会检查是否有正在排队等待发送的广播目标是被禁用的组件。
    - 会移除这些广播，防止它们被投递。

### LocalBroadcastReceiver
LocalBroadcastManager 是 Android Support Library（以及 AndroidX）提供的一个类，用于在同一个应用内部的组件之间发送和接收广播。  
- 目的是提供一种更轻量、高效、安全的广播通信方式，避免系统全局广播带来的开销和安全隐患。
- LocalBroadcastReceiver是线程安全的，注册或发送广播时都是用Synchronized关键字包裹起来。
- 在构造LocalBroadcastReceiver实例时，会创建一个Handler，用于在主线程发送通知。

## Intent 
Intent 是 Android 组件间通信的核心机制，是一种消息对象，用于描述要执行的操作和携带的数据。通过 Intent，应用可以启动 Activity、启动或绑定 Service、发送广播等。

### Intent 作用
- 启动新的 Activity。
- 启动或绑定 Service。
- 发送广播消息。
- 传递数据给目标组件。
- 实现组件之间的解耦和灵活交互。

### Intent 组成部分

| 组成部分     | 说明                                  |
|--------------|-------------------------------------|
| **Action**   | 描述要执行的动作（如 VIEW、EDIT 等） |
| **Data**     | URI，指定操作的数据（如联系人、网页等） |
| **Category** | 给 Intent 添加额外的分类信息           |
| **Extras**   | 键值对数据，携带附加信息               |
| **Component**| 显式指定目标组件（包名+类名）          |
| **Flags**    | 控制 Intent 行为的标志位               |

### 类型
#### 显式 Intent
- 明确指定目标组件的类名。
- 常用于启动应用内部的 Activity 或 Service。

``` java
Intent intent = new Intent(context, TargetActivity.class);
startActivity(intent);
```

#### 隐式 Intent
- 不指定具体组件，通过 Action、Category、Data 匹配系统或其他应用中能处理该 Intent 的组件。
- 适合组件解耦和跨应用通信。

``` java
Intent intent = new Intent(Intent.ACTION_VIEW);
intent.setData(Uri.parse("https://www.example.com"));
startActivity(intent);
```

### 启动示例
``` java
Intent intent = new Intent(context, DetailActivity.class);
intent.putExtra("item_id", 1001);
startActivity(intent);
```

### 接收Intent
``` java
Intent intent = getIntent();
String username = intent.getStringExtra("username");
```

### Intent过滤器
- 用于隐式 Intent 的匹配。
- 在 AndroidManifest.xml 或代码中注册。
- 定义组件能响应哪些 Action、Category、Data。

### Intent是否可以传递对象
Android 的 `Intent` 通过 `Bundle` 传递数据，而 `Bundle`支持的数据类型有限。要传递自定义对象，必须让对象实现以下接口之一：
1. Serializable (Java 标准接口)
2. Parcelable (Android 推荐接口)

#### 1. 使用Serializable
``` java
public class User implements Serializable {
    private String name;
    private int age;
    // 构造方法、getter/setter等
}
```

``` java
Intent intent = new Intent(context, TargetActivity.class);
User user = new User("Tom", 20);
intent.putExtra("user_key", user);
startActivity(intent);
```

``` java
User user = (User) getIntent().getSerializableExtra("user_key");
```

- 优点：简单易用，Java 原生支持
- 缺点：性能较差，序列化开销较大

#### 2. 使用Parcelable

``` java
public class User implements Parcelable {
    private String name;
    private int age;

    protected User(Parcel in) {
        name = in.readString();
        age = in.readInt();
    }

    public static final Creator<User> CREATOR = new Creator<User>() {
        @Override
        public User createFromParcel(Parcel in) {
            return new User(in);
        }
        @Override
        public User[] newArray(int size) {
            return new User[size];
        }
    };

    @Override
    public int describeContents() {
        return 0;
    }
    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeString(name);
        dest.writeInt(age);
    }
}
```

``` java
Intent intent = new Intent(context, TargetActivity.class);
User user = new User("Tom", 20);
intent.putExtra("user_key", user);
startActivity(intent);
```

``` java
User user = getIntent().getParcelableExtra("user_key");
```

- 优点：性能优于 Serializable，适合 Android 平台
- 缺点：需要实现较多模板代码（可借助插件或工具自动生成）

## Activity 和 Service 如何通信
1. 使用Intent 启动Service 传递数据
- Activity启动Service时通过Intent携带数据
- Service 在 onStartCommand()中接收Intent。
- 适合单向传递，数据量小

2. 绑定Service
- Activity 调用`bindService()`绑定Service
- Service 提供binder对象，Activity通过Binder调用Service中的方法，实现双向通信
- 适合需要频繁交互、实时通信。

3. Messenger
- Messenger 是基于 Handler 的消息机制，适合跨进程通信。
- Activity 和 Service 各自持有 Messenger，通过 Message 发送数据。
- 异步通信，比较安全

4. BroadcastReceiver
- Service 发送广播，Activity 注册广播接收器监听。
- 适合广播通知，松耦合通信。
- 注意 Android 8.0+ 对后台广播的限制。

5. EventBus/ Rxjava
第三方库简化组件间通信。

当Service 后台完成任务，想要通知Activity，依据不同的通信方式，可以：  
- Activity注册对应广播，在onReceive中做对应处理
- bindService()
    - Activity 通过 bindService() 绑定 Service，获得 Service 的 Binder 对象。
    - Service 定义一个回调接口（Listener），Activity 实现该接口并传给 Service。
    - Service 完成任务后，通过回调接口通知 Activity。
    - Activity 在回调中更新 UI。

## CI/CD
CI (持续集成，Continuous Integration)
CD (持续交付/持续部署，Continuous Delivery/Continuous Deployment)

安卓开发CI/CD的目标
- 自动化构建APK或AAB包
- 自动运行单元测试和UI测试，保证代码质量
- 自动生成测试报告和代码覆盖率报告
- 自动进行代码扫描（安全、质量检查）
- 自动发布到测试环境（如内测渠道）或应用商店
- 提升开发效率，减少人为失误，快速反馈

### 工作流主要步骤
1. 代码提交（Commit）
开发者将代码推送到代码仓库（如GitHub、GitLab、Bitbucket）。
2. 触发构建
代码仓库触发CI系统（Jenkins、GitHub Actions、GitLab CI等）自动启动构建流程。
3. 环境准备
CI环境安装Android SDK、构建工具、依赖库。
- CI环境，指的是GitLab Runner运行任务的执行环境，也就是流水线中实际执行构建、测试脚本的机器或容器。  
是专门用来自动执行CI/CD任务的环境，通常是服务器、虚拟机或容器(比如Docker)。
4. 代码编译
使用Gradle构建工具编译代码，生成APK或AAB包。
5. 自动化测试
单元测试（JUnit、Mockito等）
UI测试（Espresso、UI Automator）
静态代码分析（Lint、SonarQube）
安全扫描（如依赖漏洞扫描）
6. 测试结果反馈
将测试结果、代码覆盖率报告发送给开发团队，若失败则阻止后续操作。
7. 打包与签名
自动完成应用签名，生成正式或测试版本。
8. 发布
自动上传到测试平台（Firebase App Distribution、TestFlight、蒲公英等）
9. 通知团队
通过邮件、Slack、钉钉等渠道通知构建和发布状态。

### Gitlabd 配置步骤
1. 创建 .gitlab-ci.yml文件
位于项目根目录下，定义流水线的阶段和任务

2. 定义`Stages`(阶段)
``` yml
stages:
  - build
  - test
  - deploy
```

3. 定义Jobs(任务)
每个Job属于某个`Stage`，描述任务执行的脚本和环境。

### 灰度测试
将新版本软件逐步、分批次地推送给部分用户，而不是一次性全部用户都升级。
- 只对部分用户（如VIP用户、内部员工、特定地域用户）开放新功能。
- 逐步增加访问新版本的用户比例，比如先1%，再5%，再20%，直到100%。
- 在特定时间段内只开放给部分用户。
- 针对特定设备型号或渠道用户推送新版本。

## 编译构建流程

## 视图渲染(View Rendering) 
视图渲染指的是将应用界面上的视图（View）及其子视图绘制到屏幕上的过程。它是 UI 显示的核心环节，涉及视图的测量、布局和绘制。
- 测量（Measure）：确定View及其子View的尺寸（宽高）
- 布局（Layout）：确定View及其子View的位置（坐标）
- 绘制（Draw）：将View内容绘制到屏幕上

对应的方法分别是：
- measure(int widthMeasureSpec, int heightMeasureSpec)
- layout(int left, int top, int right, int bottom)
- draw(Canvas canvas)

### 测量（Measure）
当 View 需要确定自身大小时（比如首次显示，或者父 View 尺寸变化时），系统会调用 measure() 方法。  
- 计算当前View的宽度和高度
- 递归测量所有子View，确定整个视图树的尺寸需求

- 流程：
1. 调用 measure(widthMeasureSpec, heightMeasureSpec)
widthMeasureSpec 和 heightMeasureSpec 是由父View传入的测量规格，包含尺寸和模式（MeasureSpec）。
- 尺寸：具体的尺寸值
- 模式：
    - EXACTLY：精确大小，View 必须是这个尺寸。
    - AT_MOST：最大尺寸，View 不能超过这个尺寸。
    - UNSPECIFIED：不限制大小，View 可以是任意大小。
    尺寸（Size）：具体的尺寸值。

2. View根据MeasureSpec计算自身大小
- 调用 onMeasure(int widthMeasureSpec, int heightMeasureSpec)，子类重写该方法来自定义测量逻辑。
- 自己决定宽高（调用 setMeasuredDimension(width, height)）
- 对于ViewGroup，会遍历所有子View，调用它们的measure方法，收集子View的尺寸信息

## 布局（Layout）
- 确定View在父容器中的位置（坐标范围）
- 递归对子View进行布局
核心方法：
``` java
View.layout(int l, int t, int r, int b)
```
分别是 View 在父容器中的左、上、右、下坐标。
- layout() 会调用 onLayout(boolean changed, int left, int top, int right, int bottom)。
- 对于普通 View，onLayout() 通常不需要重写。
- 对于 ViewGroup，onLayout() 需要重写，负责对子 View 进行定位

### 绘制（Draw）
将View内容绘制到屏幕的Canvas上

- 绘制流程
1. 绘制背景
调用 `drawBackground()`，绘制 `View` 的背景 `Drawable`。
2. 绘制内容
调用 `onDraw(Canvas canvas)`，由子类重写实现具体绘制内容。
3. 绘制子 View
对于 `ViewGroup`，调用 `dispatchDraw(Canvas canvas)` 绘制所有子 `View`。
4. 绘制前景
绘制滚动条或其他装饰

## dp
dp(density-independent pixels，独立像素)是一种根据屏幕密度进行缩放的单位，保证不同屏幕密度设备上显示大小一致。
dp 和像素(px)的关系
```
px = dp*(dpi/160)
```
- 其中，`dpi`是屏幕密度(dots per inch)
- 160 dpi 是基准密度(mdpi)


### 像素
像素是数字图像的最小单位，是构成屏幕显示图像的基本点。

**屏幕分辨率**  
- 屏幕分辨率通常用像素宽 × 像素高表示，例如 1920×1080 表示宽有1920个像素，高有1080个像素。

- 屏幕密度（dpi或ppi）表示每英寸有多少像素，密度越高，图像越清晰。

#### 屏幕密度
- ppi(Pixels Per Inch)指屏幕或数字图像的像素密度。
- dpi(Dots Per Inch)表示打印机每英寸能打印多少墨点。
- 在数字显示领域，**可以混用**

##### 屏幕密度分类
在Android开发中，常见的屏幕密度分类有：

| 密度分类 | dpi范围（约） | 说明                |
|----------|---------------|---------------------|
| ldpi     | ~120 dpi      | 低密度              |
| mdpi     | ~160 dpi      | 基准密度（1x）      |
| hdpi     | ~240 dpi      | 高密度              |
| xhdpi    | ~320 dpi      | 超高密度            |
| xxhdpi   | ~480 dpi      | 超超高密度          |
| xxxhdpi  | ~640 dpi      | 极高密度            |

例子：
- 320×480 分辨率的设备通常是早期的 mdpi 设备，dpi 约为160。
- 1280×720 分辨率的设备一般是 xhdpi 设备，dpi 约为320。

##### 准确计算dpi的方法
dpi（dots per inch）是指每英寸的像素数，计算公式是：
```
dpi = sqrt((宽像素)^2 + (高像素)^2) / 屏幕对角线尺寸（英寸）
```
- **宽像素**和**高像素**是屏幕的分辨率
- **屏幕对角线尺寸**是设备屏幕的物理尺寸，单位为英寸(inch)

### 位图(Bitmap) 和 矢量图(Vector Graphics)
Bitmap 是一种用像素点阵列来表示图像的格式，每个像素用一定数目的位（bit）来表示颜色信息。
矢量图基于数学描述的图像表示方式。

## Handler
[参考](https://blog.csdn.net/JMW1407/article/details/121966563)
Handler 是 Android 中用于处理线程间通信和消息调度的一个工具类, 与 Looper 和 MessageQueue 一起工作，主要作用是：
- 在特定线程中执行代码（通常是主线程，也称 UI 线程）
- 处理消息和 Runnable 任务的排队和执行

### 核心组成
- Looper：线程的消息循环器，负责循环读取消息队列中的消息并分发。
- MessageQueue：消息队列，存储等待处理的消息和任务。
- Handler：发送消息和任务到消息队列，并负责接收和处理消息。

### 工作原理
- 每个线程可以有一个 Looper（主线程默认有）。
- Handler 绑定到某个线程的 Looper，通过它将消息和任务放入该线程的消息队列。
- 线程的 Looper 循环读取消息队列，取出消息，调用对应的 Handler 的 handleMessage() 方法进行处理。
- 通过这种机制，实现了线程间通信和异步任务调度。

### 常见用法
1. 创建

``` java
// 默认绑定当前线程的 Looper（主线程中创建即绑定主线程）
Handler handler = new Handler(Looper.getMainLooper()) {
    @Override
    public void handleMessage(Message msg) {
        // 处理消息
    }
};
```

2. 发送消息

``` java
Message msg = Message.obtain();
msg.what = 1;
handler.sendMessage(msg);
```

3. 发送延时任务

``` java
handler.postDelayed(new Runnable() {
    @Override
    public void run() {
        // 延时执行的代码
    }
}, 1000); // 1秒后执行
```

### 典型应用场景
- 在子线程中向主线程发送消息，更新 UI
- 定时执行任务（如轮询、动画）
- 线程间通信，避免直接操作 UI 导致异常
- 实现消息机制，解耦代码

## Button

### OnTouchListener & OnClickListener

| 特性               | OnTouchListener               | OnClickListener       |
|--------------------|--------------------------------|---------------------------------|
| 监听的事件类型      | 监听触摸事件（Touch Event），包括按下、移动、抬起等 | 监听点击事件（Click Event），即“按下并抬起”动作 |
| 方法签名           | `boolean onTouch(View v, MotionEvent event)`      | `void onClick(View v)`                           |
| 事件粒度           | 低层次，捕获所有触摸动作（ACTION_DOWN、ACTION_MOVE、ACTION_UP 等） | 高层次，专门处理点击（按下并快速抬起） |
| 返回值             | `boolean`，决定事件是否被消费（`true`表示消费，事件不再传递） | 无返回值，点击事件默认被消费   |
| 触发时机           | 触摸屏幕时即时触发                                | 当用户完成一次点击（按下并抬起）时触发   |
| 使用场景           | 需要自定义复杂手势、拖拽、滑动等交互时使用        | 处理简单点击（按钮点击、列表项点击等） |

可以同时设置 OnTouchListener 和 OnClickListener，但需要注意事件传递和消费的关系。  
- 当设置了 OnTouchListener，并且其 onTouch 方法返回 true，表示事件被消费了，这时 OnClickListener 不会被调用，因为点击事件依赖于触摸事件的传递。
- 如果 onTouch 返回 false，表示事件没有被完全消费，事件会继续传递，系统会检测是否触发点击事件，从而调用 OnClickListener。

## 语句覆盖法(Statement Coverage)
语句覆盖法是一种软件测试的**覆盖率度量标准和测试设计准则**。

- 它是一种测试覆盖标准，用来衡量测试用例对程序代码执行的充分程度。
- 通过语句覆盖率，可以判断测试用例是否执行了程序中的所有语句。
- 它指导测试人员设计测试用例，确保代码的每条语句至少被执行一次，以发现程序中的潜在错误或死代码。

## resouces
### string.xml
#### 字符串资源冲突问题和排查解决方案
- 在多模块项目中，多个模块的 res/values/strings.xml 文件定义了相同的字符串资源 key。
- 编译打包时，这些资源会被合并。
- 运行时，系统会使用合并后资源中某个模块的字符串（通常是最后合并的那个），导致显示的字符串与预期不符。
- 资源合并顺序通常依赖于模块依赖关系和 Gradle 配置，比较难直接控制。

##### 排查步骤
1. 确认资源冲突
- 使用Android Studio的`Find in Path`搜索资源key

2. 查看合并后的资源
- 在Android Studio的Build视图，找到生成的APK或AAR的资源合并报告。
- 具体路径一般在 `app/build/intermediates/merged_res`，查看 merged 的 strings.xml，确认最终使用的是哪个模块的字符串。

3. 检查依赖关系
检查各模块的 build.gradle 中依赖关系，确认模块之间的依赖顺序。
依赖顺序影响资源合并的覆盖顺序。

## Fragment

### 生命周期

| 生命周期方法           | 说明                                                         |
|-----------------------|--------------------------------------------------------------|
| `onAttach(Context)`   | Fragment 与 Activity 关联，获取 Context。只调用一次。          |
| `onCreate(Bundle)`    | Fragment 创建，初始化非视图相关的数据。                        |
| `onCreateView(...)`   | 创建并返回 Fragment 的视图层次结构（UI）。                     |
| `onViewCreated(...)`  | 视图创建完成，进行视图相关初始化，如绑定控件、设置监听。         |
| `onActivityCreated()` | Activity 的 `onCreate()` 执行完毕，Fragment 可以安全访问 Activity。|
| `onStart()`           | Fragment 可见但不一定在前台。
| `onResume()`          | Fragment 处于前台，用户可交互。                               |
| `onPause()`           | Fragment 不再处于前台，通常用于保存数据或停止动画等。           |
| `onStop()`            | Fragment 不再可见。                                           |
| `onDestroyView()`     | 销毁 Fragment 的视图层次结构，释放与视图相关资源。             |
| `onDestroy()`         | Fragment 完全销毁，释放所有资源。                             |
| `onDetach()`          | Fragment 与 Activity 解除关联。                               |

结束生命周期的调用顺序： 先销毁视图，再销毁 Fragment 实例，最后解除与 Activity 的绑定。

``` java
onDestroyView() → onDestroy() → onDetach()
```

- `onDestroyView()`
    - 视图被销毁时调用。
    - 这里释放与视图相关的资源（比如绑定的控件、动画、监听器等）。
    - Fragment 实例仍然存在，但它的视图层次结构被移除。
    - 例如，Fragment 进入“后台”或被替换时会调用。

- `onDestroy()`
    - Fragment 完全销毁前调用。
    - 释放 Fragment 持有的所有非视图资源（如线程、数据库连接等）。
    - Fragment 实例即将被销毁。

- `onDetach()`
    - Fragment 与宿主 Activity 解除关联时调用。
    - Fragment 不再持有 Activity 的引用。
    - 这是 Fragment 生命周期的最后一个回调。

### Activity 和 Fragment 通信方式
`Fragment`通常是嵌入在`Activity`中的UI组件，传递数据或通知事件常见的有以下几种方式。

#### 接口回调
- Fragment定义一个接口，Activity实现该接口
- Fragment通过接口调用通知Activity
- Activity通过调用Fragment的公共方法传递数据

##### 代码示例
###### Fragment 定义接口
``` java
public class MyFragment extends Fragment {
    // 定义接口
    public interface OnDataPassListener{
        void onDataPass(string data);
    }

    private OnDataPassListener mListener;

    @override
    public void onAttach(Context context){
        super.onAttach(context);
        if (context instanceof OnDataPassListener){
            mListener = (OnDataPassListener) context;
        }else{
            throw new RuntimeException(context.toString() + "must implement OnDataPassListener");
        }
    }

    //需要传递数据时调用
    public void passDataToActivity(){
        if(mListener != null){
            mListener.onDataPass("Hello Activity");
        }
    }

    // 接收数据
    public void receiveData(String data){
        Log.d("MyFragment","Received data:" + data);
    }
}
```

###### Activity 实现接口
``` java
public class MyActivity extends AppCompatActivity implements MyFragment.OnDataPassListener {
    @override
    public void onDataPass(string data){
        // 接收到Fragment传来的数据
        Log.d("MyActivity", "Data from Fragment:"+ data);
    }

    // 通过Fragment实例调用公共方法传数据给Fragment
    public void sendDataToFragment(){
        MyFragment fragment = (MyFragment) getSupportFragmentManager().findFragmentById(R.id.my_fragment);
        if(fragment != null) {
            fragment.receiveData("Hello Fragment");
        }
    }
}
```

#### 通过Bundle传递参数(Activity -> Fragment)
- Activity 创建Fragment时，通过`setArguments(Bundle)`传递初始化参数
- Fragment 在 `onCreate()`或`onCreateView`中通过`getArguments()`获取。

##### 代码示例
###### Activity里创建Fragment并传递参数
``` java
public class MyActivity extends AppCompatAcitivity{
    @override
    protected void onCreate(Bundle savedInstanceState){
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        //创建Bundle, 放入要传递的参数
        Bundle bundle = new Bundle();
        bundle.putString("key1","value1");

        //创建 Fragment实例
        MyFragment fragment = new MyFragment();

        //将Bundle设置给Fragment
        fragment.setArguments(bundle);

        // 将Fragment添加到Activity布局中
        getSupportFragmentManager()
                .beginTransaction()
                .replace(R.id.fragment_container,fragment)
                .commit();
    }
}
```

###### Fragment里接收参数
``` java
public class MyFragment extents Fragment {
    private string receivedData;

    @override
    public void onCreate(Bundle savedInstanceState){
        super.onCreate(savedInstanceState);

        // 获取传递的参数
        Bundle argument = getArgument();
        if(argument != null){
            receivedData = arguments.getString("key1");
        }
    }

    @Nullable
    @override
    public View onCreateView(LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState){
        View view = inflater.inflate(R.layout.my_fragment, container, false);

        //假设用一个TextView显示传来的数据
        TextView textview = view.findViewById(R.id.text_view);
        textview.setText(receivedData);

        return view;
    }
}
```

#### 通过ViewModel (推荐用于Jetpack架构)
- 使用`ViewModel`和`LiveData`, Activity和Fragment共享同一个ViewModel
- 双方通过观察`LiveData`实现数据通信，解耦且生命周期安全。

#### 通过EventBus
- 使用第三方事件总线库(如greenrobot的EventBus) 进行发布订阅通信
- 适合多个组件间解耦通信。

## 内存抖动
内存抖动是指程序运行时频繁创建和销毁对象，导致垃圾回收频繁触发，从而影响性能和用户体验的现象。

### 举例说明
``` java
@override
public void onDraw(Canvas canvas){
    Paint paint = new Paint();
    canvas.drawCircle(50, 50, 20, paint);
}
```

`onDraw`方法非常频繁调用，每次都创建新的`Paint`对象，导致大量临时对象产生，很快就被回收，频繁触发GC，导致内存抖动。

### 带来的问题
- 性能下降：GC是重量级操作，会暂停应用线程执行，影响流畅度
- 卡顿和掉帧：用户界面响应变慢，动画不连贯。

### 如何减少内存抖动
1. 复用对象
避免在频繁调用的方法中创建临时对象，比如把`Paint`对象声明成成员变量，重复使用

2. 使用对象池
对于需要频繁创建的对象，可以用对象池技术复用对象。

3. 避免不必要的
尽量使用基本类型，避免自动装箱带来的频繁对象创建

4. 优化数据结构
使用合适的数据结构，减少临时对象的创建

5. 使用性能分析工具
利用Android Studio 的 Profiler、Heap Dump等工具定位内存抖动的热点代码

## 内存优化
内存优化是提升应用性能、避免内存泄漏和卡顿的重要环节。常用以下优化方式：

### 1. 避免内存泄漏
内存泄漏指应用不再使用的对象仍被引用，导致垃圾回收器无法回收，长时间累积会导致内存溢出。

#### 常见场景
- Activity 或 Fragment 被静态变量持有
- Handler、Runnable、TimerTask 持有外部类引用
- 资源未及时关闭（Cursor、Stream、Bitmap等）
- 监听器、BroadcastReceiver未注销

#### 优化措施
- 避免使用静态变量持有 Context，尤其是 Activity Context，尽量用 Application Context。
- Handler 使用静态内部类 + 弱引用持有外部类，防止隐式引用导致泄漏。
- 及时注销监听器、广播接收器。
- 使用 LeakCanary 等工具检测内存泄漏。

### 2. 减少内存分配与对象创建
频繁创建大量对象会导致频繁GC，影响性能。

#### 优化措施
- 重用对象，避免在循环或频繁调用中创建临时对象。
- 使用对象池（如 RecyclerView 的 ViewHolder 机制）。
- 使用基本类型数组代替包装类型数组（避免自动装箱）。
- 避免在绘制或动画中频繁创建对象。

### 3. 优化Bitmap使用
Bitmap(位图)是安卓中内存占用较大的资源。

#### 优化措施
- 使用`BitmapFactory.Options.inSampleSize`按需缩放图片，避免加载过大图片
- 使用`Bitmap.recycle()`及时释放不再使用的Bitmap
- 使用`LruCache`缓存Bitmap，避免重复加载
- 使用矢量图代替部分位图
- 使用Glide、Picasso等图片加载库，自动管理内存和缓存

### 4. 优化布局和视图层级
复杂布局和深层次的View层级会增加内存和绘制负担。

#### 优化措施
- 减少布局嵌套，使用`ConstraintLayout、RelativeLayout`替代多层`LinearLayout`
- 使用`ViewStub`延迟加载不立即需要的布局。
- 使用合适的布局宽高参数，避免过度测量
- 使用`RecycleView`替代`ListView`,提高视图复用效率

### 5. 使用合适的数据结构

#### 优化措施
- 使用`SparseArray`、`SparseBooleanArray`替代`HashMap<Integer, Object>`，节省内存。
- 选择合适的数据类型，避免装箱拆箱带来的额外开销。

### 6. 管理线程和异步任务
线程过多或未正确关闭会导致内存泄漏和资源浪费

#### 优化措施
- 使用线程池管理线程，避免频繁创建销毁
- 及时取消和释放异步任务
- 使用`AsyncTask`时，注意生命周期绑定，避免持有Activity导致泄漏
> AsyncTask 是 Android 平台提供的一个用于简化异步操作的类。它帮助开发者在后台线程执行耗时任务，同时方便地在主线程（UI线程）更新界面，避免了直接操作线程和 Handler 的复杂性。
> 如果在 Activity 中直接创建 AsyncTask（非静态内部类）, 当 AsyncTask 执行时间较长，Activity 已经被用户关闭（比如按返回键退出），但 AsyncTask 仍在后台执行。由于 AsyncTask 持有 Activity 引用，导致 Activity 无法被垃圾回收，造成内存泄漏。

### 7. ProGuard / R8 混淆和优化
### 8. 使用内存分析工具
- Android Profiler：Android Studio 自带的内存分析工具，实时监控内存分配和GC。
- LeakCanary：开源的内存泄漏检测库，自动检测并定位泄漏。

## 性能优化
1. 避免主线程阻塞
- 主线程负责界面绘制和用户交互，避免执行耗时操作（如网络请求、数据库查询、复杂计算）。
- 使用异步任务（`AsyncTask`、`Thread`、`HandlerThread`、`RxJava`、`Coroutine`等）处理耗时操作。

2. 优化布局和绘制
- 减少布局层级，避免过深的视图树（使用ConstraintLayout、合并布局）
- 使用ViewStub延迟加载不必要的视图。

3. 合理使用缓存
- 图片缓存（内存缓存和磁盘缓存），避免重复加载和解码图片。
- 数据缓存，减少网络请求次数。

4. 减少内存分配和垃圾回收
- 避免频繁创建临时对象，尤其是在绘制和滚动过程中。
- 使用对象池（如RecyclerView的ViewHolder机制）重用对象。
- 避免内存泄漏，及时释放资源。

5. 优化线程和并发
- 合理使用线程池，避免过多线程导致上下文切换开销。
- 使用高效的并发框架

6. 使用硬件加速和合适的动画
- 开启硬件加速，提升渲染效率。
- 优化动画，避免复杂动画造成卡顿。

7. 网络请求优化
减少请求次数，使用合适的请求方式（如HTTP/2、压缩）

8. 数据库优化
- 使用索引，加快查询速度。
- 避免主线程访问数据库。

9. 使用性能分析工具
`Android Profiler`、`Systrace`、`LeakCanary`、`StrictMode`等工具定位性能瓶颈和内存泄漏。