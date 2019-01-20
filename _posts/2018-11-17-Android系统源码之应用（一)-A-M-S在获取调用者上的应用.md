---
layout:     post
title:      Android系统源码之应用 （一）AMS
date:       2018-11-17
author:     sg
catalog: true
tags:
    - Android
    - 源码
---

>面试造航母，工作拧螺丝。

很多人这样评价面试。在Android面试中Android系统源码相关的知识逐渐从加分项变成了必考项，很多新手Android码农认为Android系统源码其实没什么卵用，学习它只是为了面试，这个系列我将结合实际应用来对Android系统源码进行分析，学习了分析方法之后，以后一旦遇到一些网上没有现成的解决方案时，可以自己通过阅读源码的方式来解决问题。

### 1.背景：如何获取一个Activity的调用者？

最近在工作中遇到了这样的需求。我们有一个Exported Activity是供第三方的App来唤醒，调用我们自己的应用起来，那相应的我们就需要给第三方的App开发者付钱作为推广费用。其中有一些邪恶的应用会考这个来赚钱，因此，我们就要把该Exported Activity的调用者上报，发生异常时进行分析。

### 2.Activity的跳转流程。

目前（18-10-27）为止，据我所知网上并没有一个通用的解决办法，那我就打算从源码中寻找解决方案，关注Activity跳转部分的代码，这里可以参考源码分析书籍和博客中的流程图来帮助理解。我把这部分的流程总结如下：
![Activity跳转流程](https://ws3.sinaimg.cn/large/006tNbRwly1fxz3oxd3qmj30rs0w1dht.jpg)
我对这个流程进行了一些精简，突出跳转的流程。现在看起来就比较简单，就不做过多的分析了。通过阅读这个流程图，并结合已有的知识我们可以发现有一些规律：

- 一次跨应用Activity跳转涉及到三个进程，分别是应用A、 AMS、 B。A与B和AMS交互是C/S型交互，AMS扮演服务端。

- A和B又不直接的和AMS交互，他们通过AMP（ActivityManagerProxy）和AMS交互。

- AMS通过访问ActivityStack这个数据结构来获得应用A和应用B的信息

### 3.如何获取调用者？解决方案一

分析完了这个流程，获取Activity调用者的方法无非就是两种：
1. 应用B在启动时接收了哪些参数
2. 应用B作为C/S的客户端，可以向AMS请求哪些信息
首先我们看第一个方案。
应用B是我们自己的应用，如何获取他的参数呢？这里有一个最简单的方法推荐给大家，就不需要跟源码了。就是在应用B的Activity上打一个断点，然后我们使用测试的应用A发送跳转Intent。之后会发现程序走到了断点，这里的断点设在了onResume上。默认在Android Studio的左下角有一个调用栈窗口。此时，他看起来是这样的：
![Activity调用栈](https://ws4.sinaimg.cn/large/006tNbRwly1fxz3pmnkddj30ga08gq6j.jpg)
这时可以点击任意一行记录，然后再右侧的窗口上输入变量名来获取他们当前的值。这里的每一行都由三部分组成，即方法行，类名，包名。然后我们从下向上的看，寻找有用的信息，这时我们看到了handleLaunchActivity这个函数，他的第一个参数我们看它的各种成员变量，发现有一个成员变量叫做referrer,正是应用A的包名，然后我们回到源码，跟踪这个字段的去向，最终发现他最终赋值给了Activity的mReferrer成员变量。那么解决方案就很简单了，我们只需要在B应用的Activity上使用反射获取这个字段就行了。

```java
efererField = Activity.class.getDeclaredField("mReferrer");
refererField.setAccessible(true);
referer = ((String) refererField.get(openingActivity));
```

### 4.如何获取调用者？解决方案二

我们先来看下可行性。
根据上面的流程图，我们知道应用A和应用B并不是直接和AMS通信的，他们借助ActivityManagerProxy这个代理类来和ActivityManagerService通信。如果熟悉AIDL编程的话，可以直接根据Android系统的命名规范来找到这个进程通信的接口就是IActivityManagaer,在这个接口中发现了如下两个方法

```java
    // These are not public because you need to be very careful in how you
    // manage your activity to make sure it is always the uid you expect.
    public int getLaunchedFromUid(IBinder activityToken) throws RemoteException;
    public String getLaunchedFromPackage(IBinder activityToken) throws RemoteException;
```

看来方法二是有希望实现的。找到这两个方法之后还要确认两点：
- 1.应用是否有权限调用这个接口
- 2.是否有已知的方法调用这两个接口
对于第一点，我们要跟进到ActivityManagerService中去，找到对于的标签（case）发现并没有权限校验的代码；对于第二点可以使用各种合适的源码分析工具查看调用图，貌似也并没有可以用的。
接下来就要想办法获得IPC的客户端了
在源码工程中找到ActivityManagerProxy,发现他是ActivityManagerNative的内部类，我们在ActivityManagerNative中看到下面这段熟悉的代码，这段代码在抉择使用代理还是本地对象：
```java
    static public IActivityManager asInterface(IBinder obj) {
        if (obj == null) {
            return null;
        }
        IActivityManager in =
            (IActivityManager)obj.queryLocalInterface(descriptor);
        if (in != null) {
            return in;
        }

        return new ActivityManagerProxy(obj);
    }
```
ActivityManagerNative也是一个实现了IActivityManager的抽象类，它包裹了ActivityManagerProxy,真正的远程客户端还是ActivityManagerProxy,然而ActivityManagerNative类是一个单例类，这意味着我们只需要这个类的类名就可以方便的使用反射来拿到他的实例。下面是代码
```java
Class<?> activityManagerNativeClass = Class.forName("android.app.ActivityManagerNative");
Field fieldMToken = Activity.class.getDeclaredField("mToken");
fieldMToken.setAccessible(true);
IBinder mActivityToken = (IBinder) fieldMToken.get(openingActivity);
Method methodGetLaunchedFromPackage = activityManagerNativeClass.getMethod("getLaunchedFromUid", IBinder.class);
Method methodGetDefault = activityManagerNativeClass.getMethod("getDefault", new Class[0]);
methodGetDefault.setAccessible(true);
Object refererObj = methodGetLaunchedFromPackage.invoke(methodGetDefault.invoke(null, null) , mActivityToken);
PackageManager pm = openingActivity.getPackageManager();
referer =  pm.getNameForUid((Integer) refererObj);
```
在测试中发现低版本的源码中没有getLaunchedFromPackage这个方法，所以上面使用了getLaunchedFromUid和PackageManager来曲线救国，以覆盖更多的版本。在实际的应用中，部分国产系统可能是对rom做了修改，在反射时候会报错，注意兼容性。

### 5.如何绕过调用者的检测

额，这一点不知道当讲不当讲，感觉有种为虎作伥的感觉。
上面的分析中，我们都是以本应用的视角来分析系统的源码，现在我们要以调用者的视角来进行分析。首先当然要分析的是startActivity,我们这里省去一些没什么卵用的跳转，直接从Instrumentation这个类进行分析，这是一个相当重要的类。

```java
    public ActivityResult execStartActivity(
        Context who, IBinder contextThread, IBinder token, String target,
        Intent intent, int requestCode, Bundle options) {
        IApplicationThread whoThread = (IApplicationThread) contextThread;
        if (mActivityMonitors != null) {
            synchronized (mSync) {
                final int N = mActivityMonitors.size();
                for (int i=0; i<N; i++) {
                    final ActivityMonitor am = mActivityMonitors.get(i);
                    if (am.match(who, null, intent)) {
                        am.mHits++;
                        if (am.isBlocking()) {
                            return requestCode >= 0 ? am.getResult() : null;
                        }
                        break;
                    }
                }
            }
        }
        try {
            intent.migrateExtraStreamToClipData();
            intent.prepareToLeaveProcess();
            int result = ActivityManagerNative.getDefault()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target, requestCode, 0, null, options);
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
            throw new RuntimeException("Failure from system", e);
        }
        return null;
    }
```
发现了熟悉的ActivityManagerNative,调用了startActivity方法，看一下方法的原型

```java
public int startActivity(IApplicationThread caller, String callingPackage, Intent intent,
        String resolvedType, IBinder resultTo, String resultWho, int requestCode, int flags,
        ProfilerInfo profilerInfo, Bundle options) throws RemoteException;
```

显然这里我们应该关注的是第二个参数，跟踪一下他是怎么来的。发现他来自ContextWrapper
类中的Context mBase成员变量中的String mBasePackageName成员变量。因此我们可以这样修改它的值

```java
Field field = ContextWrapper.class.getDeclaredField("mBase");
field.setAccessible(true);
Object object = field.get(this);
Field f = object.getClass().getDeclaredField("mBasePackageName");
f.setAccessible(true);
f.set(object, "fakeRefer");
```

测试通过。


