### 一:认识WMS

##### 1.1:初识WMS

WindowManagerService服务（WMS）的实现是相当复杂的，毕竟它要管理的整个系统所有窗口的UI，而在任何一个系统中，窗口管理子系统都是极其复杂的。而WMS就是管理整个系统的窗口的。

##### 1.2:WMS的功能

- 为所有窗口分配Surface。客户端向WMS添加一个窗口的过程，其实就是WMS为其分配一块Suiface的过程，一块块Surface在WMS的管理下有序的排布在屏幕上。Window的本质就是Surface。
- 管理Surface的显示顺序、尺寸、位置
- 管理窗口动画
- 输入系统相关：WMS是派发系统按键和触摸消息的最佳人选，当接收到一个触摸事件，它需要寻找一个最合适的窗口来处理消息，而WMS是窗口的管理者，系统中所有的窗口状态和信息都在其掌握之中。

##### 1.3:认识Window

Window表明它是和窗口相关的，窗口是一个抽象的概念，从用户的角度来讲，它是一个界面；从SurfaceFlinger的角度来看，它是一个Layer，承载着和界面有关的数据和属性；从WMS角度来看，它是一个WIndowState，用于管理和界面有关的状态。

有这样的一个比喻。整个界面就像由N个演员参与的话剧：SurfaceFling是摄像机，它只负责客观的捕捉当前的画面，然后真实的呈现给观众；WMS就是导演，它要负责话剧的舞台效果、演员站位；ViewRoot就是各个演员的长相和表情，取决于它们各自的条件与努力。可见，WMS与SurfaceFling的一个重要区别就是——后者只做与显示相关的事情，而WMS要处理对输入事件的派发。

Android支持的窗口类型很多，统一可以分为三大类，另外各个种类下还细分为若干子类型，且都在WindowManager.java中有定义。这三类分别是ApplicationWindow，SystemWindow，SubWindow。

##### 1.4:WMS工作流程

关于WMS的工作流程大致归纳如下几步：

- 窗口大小和位置（X轴和Y轴）的计算过程
- 窗口的组织方式
- 输入法窗口的调整过程
- 壁纸窗口的调整过程
- 窗口Z轴位置的计算和调整过程
- Activity窗口的启动窗口的显示过程
- Activity窗口的切换过程
- Activity窗口的动画显示过程

##### 1.5:WMS的设计思想

我们知道，学习一个框架其实就是学习一种编程思想，或者说理解他的设计模式，只有掌握了设计思想理解起来就事半功倍了，WMS的设计思想就是利用了桥接模式。

![1697699147119](C:\Users\xiaolei1.shen\AppData\Roaming\Typora\typora-user-images\1697699147119.png)

### 二:认识AMS

##### 2.1:初识AMS

AMS是系统的引导服务，应用进程的启动、切换和调度、四大组件的启动和管理都需要AMS的支持。

##### 2.2:AMS的功能

- 统一调度所有应用程序的Activity的生命周期
- 启动或杀死应用程序的进程
- 启动并调度Service的生命周期
- 注册BroadcastReceiver，并接收和分发Broadcast
- 启动并发布ContentProvider
- 调度task
- 处理应用程序的Crash
- 查询系统当前运行状态

##### 2.3:AMS的启动流程

关于AMS的启动流程大致我认为可以分为如下几步，这里我也是初步概括一下。

1. 启动ActivityManagerService。
2. 调用setSystemProcess。
3. 调用installSystemProviders方法。
4. 调用systemReady方法。

##### 2.4:AMS的工作流程

AMS的工作流程，其实就是Activity的启动和调度的过程，所有的启动方式，最终都是通过Binder机制的Client端，调用Server端的AMS的startActivityXXX()系列方法。所以可见，工作流程又包括Client端和Server端两个。下面简单介绍一下这俩端的职责。

###### 1:客户端工作流程

1. Launcher主线程捕获onClick()点击事件后，调用Launcher.startActivitySafely()方法。Launcher.startActivitySafely()内部调用了Launcher.startActivity()方法，Launcher.startActivity()内部调用了Launcher的父类Activity的startActivity()方法。

2. Activity.startActivity()调用Activity.startActivityForResult()方法，传入该方法的requestCode参数若为-1，则表示Activity启动成功后，不需要执行Launcher.onActivityResult()方法处理返回结果。

3. 启动Activity需要与系统ActivityManagerService交互，必须纳入Instrumentation的监控，因此需要将启动请求转交instrumentation，即调用Instrumentation.execStartActivity()方法。

4. Instrumentation.execStartActivity()首先通过ActivityMonitor检查启动请求，然后调用ActivityManagerNative.getDefault()得到ActivityManagerProxy代理对象，进而调用该代理对象的startActivity()方法。

5. ActivityManagerProxy是ActivityManagerService的代理对象，因此其内部存储的是BinderProxy，调用ActivityManagerProxy.startActivity()，实质是调用BinderProxy.transact()向Binder驱动发送START_ACTIVITY_TRANSACTION命令。Binder驱动将处理逻辑从Launcher所在进程切换到ActivityManagerService所在进程。

###### 2:服务端工作流程

启动Activity的请求从Client端传递给Server端后，便进入了启动应用的七个阶段，这里也是整理出具体流程。

2.1 预启动
ActivityManagerService.startActivity()
ActivityStack.startActivityMayWait()
ActivityStack.startActivityLocked()
ActivityStack.startActivityUncheckedLocked()
ActivityStack.startActivityLocked()（重载）
ActivityStack.resumeTopActivityLocked()
2.2 暂停
ActivityStack.startPausingLocked()
ApplicationThreadProxy.schedulePauseActivity()
ActivityThread.handlePauseActivity()
ActivityThread.performPauseActivity()
ActivityManagerProxy.activityPaused()
completePausedLocked()
2.3 启动应用程序进程
第二次进入ActivityStack.resumeTopActivityLocked()
ActivityStack.startSpecificActivityLocked()
startProcessLocked()
startProcessLocked()（重载）
Process.start()
2.4 加载应用程序Activity
ActivityThread.main()
ActivityThread.attach()
ActivityManagerService.attachApplication()
ApplicationThread.bindApplication()
ActivityThread.handleBindApplication()
2.5 显示Activity
ActivityStack.realStartActivityLocked()
ApplicationThread.scheduleLaunchActivity()
ActivityThead.handleLaunchActivity()
ActivityThread.performLaunchActivity()
ActivityThread.handleResumeActivity()
ActivityThread.performResumeActivity()
Activity.performResume()
ActivityStack.completeResumeLocked()
2.6 Activity Idle状态的处理
2.7 停止源Activity
ActivityStack.stopActivityLocked()
ApplicationThreadProxy.scheduleStopActivity()
ActivityThread.handleStopActivity()
ActivityThread.performStopActivityInner()
2.5:AMS的设计思想
AMS也用到了设计思想，主要是代理模式。关于代理模式这里就不再讲解，参考《Android源码设计模式》第18章。

![1697699127106](C:\Users\xiaolei1.shen\AppData\Roaming\Typora\typora-user-images\1697699127106.png)


根据上图我们可以看出，ActivityManagerProxy和ActivityManagerNative都实现了IActivityManager，ActivityManagerProxy就是代理部分，ActivityManagerNative就是实现部分，但ActivityManagerNative是个抽象类，并不处理过多的具体逻辑，大部分具体逻辑是由ActivityManagerService承担，这就是为什么我们说真实部分应该为ActivityManagerService。

### 三:认识PMS

##### 3.1:初识PMS

PackageManagerService（简称 PMS），是 Android 系统核心服务之一，处理包管理相关的工作，常见的比如安装、卸载应用等。PMS是系统服务，那么应用层肯定有个PackageManager作为binder call client端来供使用，但是这里要注意，PackageManager是个抽象类，一般使用的是它的实现类：ApplicationPackageManager。因此PackageManager功能的具体实现还是ApplicationPackageManager这个实现类。

##### 3.2:PMS的功能

- 提供一个应用程序的所有信息（ApplicationInfo）
- 提供四大组件的信息
- 查询permission相关信息
- 提供包的信息
- 安装、卸载APK

## 总结

##### 服务端

- AMS 主要用于管理所有应用程序的Activity
- WMS 管理各个窗口，隐藏，显示等
- PMS 用来管理跟踪所有应用APK，安装，解析，控制权限等.
- 还有用来处理触摸消息的两个类KeyInputQueue和InputDispatchThread，一个用来读消息，一个用来分发消息.。

##### 客户端

主要包括ActivityThread，Activity，DecodeView及父类View，PhoneWindow，ViewRootImpl及内部类等
ActivityThread主要用来和AMS通讯的客户端，Activity是我们编写应用比较熟悉的类。

**关于AMS、WMS、PMS的详细讲解见下面博客：**

AMS：

https://blog.csdn.net/freekiteyu/article/details/82774947

[一篇文章了解相见恨晚的 Android Binder 进程间通讯机制_binder通信-CSDN博客](https://blog.csdn.net/freekiteyu/article/details/70082302)

https://blog.csdn.net/freekiteyu/article/details/79318031

[一篇文章看明白 Activity 与 Window 与 View 之间的关系-CSDN博客](https://blog.csdn.net/freekiteyu/article/details/79408969)

https://blog.csdn.net/freekiteyu/article/details/79483406

https://blog.csdn.net/freekiteyu/article/details/79785720

[一篇文章看明白 Android PackageManagerService 工作流程_packagemanager更新时如何更新dexpathlist-CSDN博客](https://blog.csdn.net/freekiteyu/article/details/82774947)

https://blog.csdn.net/freekiteyu/article/details/84849651

WMS：

https://blog.csdn.net/luoshengyang/article/details/8479101

[Android窗口管理服务WindowManagerService对窗口的组织方式分析-CSDN博客](https://blog.csdn.net/luoshengyang/article/details/8498908)

https://blog.csdn.net/luoshengyang/article/details/8526644

https://blog.csdn.net/luoshengyang/article/details/8570428

https://blog.csdn.net/luoshengyang/article/details/8577789

https://blog.csdn.net/luoshengyang/article/details/8596449

[Android窗口管理服务WindowManagerService显示窗口动画的原理分析-CSDN博客](https://blog.csdn.net/luoshengyang/article/details/8611754)

PMS：

[Android解析ActivityManagerService（一）AMS启动流程和AMS家族_android 解析 activitymanagersservice 刘望舒-CSDN博客](https://liuwangshu.blog.csdn.net/article/details/76405596)

[Android解析ActivityManagerService（二）ActivityTask和Activity栈管理-CSDN博客](https://liuwangshu.blog.csdn.net/article/details/77542286)

