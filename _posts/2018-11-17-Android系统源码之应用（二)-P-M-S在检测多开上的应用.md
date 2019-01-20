---
layout:     post
title:      Android系统源码之应用 （二）PMS
date:       2018-11-17
author:     sg
catalog: true
tags:
    - Android
    - 源码
---

### 1.背景
对于移动黑产来说，使用多开来刷应用是基本操作。所以我们必须要对多开进行检测。发现用户设备不合法时强行退出应用或者限制用户的操作逻辑。

### 2.应用是如何多开的
经过调查，虽然多开的工具多种多样，但是归根结底有如下三种多开方式：
- 通过各种插件化框架直接加载原APK；
- 动态生成一个安装包，在这个安装包中插件框架加载原APK；
- 使用另一个用户重新安装原始apk。
前两种方式咱们按下不表，这篇文章主要分析第三种方式。

### 3.如何鉴别App是否多开了
两个内容是一样的App要想共存，有两个条件，要么是包名不同，要么是用户不同
![多开特征，只需关注前两行](https://ws4.sinaimg.cn/large/006tNbRwly1fxz5151cqzj31r20gekjl.jpg)
Android中的用户可以理解为应用的ID,每个应用对应一个用户。Android使用用户，共享用户和用户组来管理权限。
而在第三中情况中用户的包名没有篡改，但却对应着两个用户，这一般发生于系统级或root用户的多开。多开应用使用

```java
    void installPackageAsUser(in String originPath,
            in IPackageInstallObserver2 observer,
            int flags,
            in String installerPackageName,
            in VerificationParams verificationParams,
            in String packageAbiOverride,
            int userId);
    ```
    
来重新安装一遍应用。因此我们需要分析下Apk安装的过程。
### 4.Apk安装流程
有了前面文章的经验，我们知道借助Android命名规范来查看源码效率是十分高的，因此我们直接定位到IPackageManger这个文件。
发现安装应用除了上面提到的installPackageAsUser还有installPackage方法。
然后我们再次根据命名规范来打开PackageManagerService，在其中搜索这个方法。
跟进代码调用流程，跳过一些包的拷贝，验证，存储空间验证等无关的逻辑。最终到了关键方法

```java
private void installPackageLI(InstallArgs args, PackageInstalledInfo res) {}
```
然后是方法
```java
public Package parsePackage(File packageFile, int flags) throws PackageParserException {}
```
接下来我们用第三方工具分析一下代码跳转，因为这里逻辑实在太繁琐了，用肉眼看不太现实。源码阅读工具Windows下可以选用SourceInsight，Mac平台下使用Understand，看下代码调用图，发现这样一条代码流程。
![解析流程](https://ws1.sinaimg.cn/large/006tNbRwly1fxz75ceeokj30wc0403ye.jpg)
在最终节点
```java
private Activity parseActivityAlias(Package owner, Resources res,
        XmlPullParser parser, AttributeSet attrs, int flags, String[] outError)
        throws XmlPullParserException, IOException {
```
看到XmlPullParser根据标签名将Intent保存在owner变量中，然后回传给PackageManager的mActivity中。
### 5.多开的检测
经过上面的分析我们想构造一个极其特殊的Activity，正常情况下只有我们的应用有，如果我们发现整个系统中有多个这样的Activity，那么就说明被多开了。
于是发现了这样一个Api,我们的系统级多开检测工具瞬间就可以写好了
```java
public List<ResolveInfo> queryIntentActivities(Intent intent,
        String resolvedType, int flags, int userId) {
```
首先构造这样一个Activity，作为我们应用的标识

```xml
		<activity  
		    android:exported="true"  
		    android:name=".EXP" >  
		    <intent-filter>  
		        <action android:name="unique" />  
		        <category android:name="android.intent.category.DEFAULT" />  
		    </intent-filter>  
		</activity> 
```

检测时

```java
public boolean isMultiOpen(){  
    Intent i = new Intent("xxx123321");  
    List<ResolveInfo> list = getPackageManager().queryIntentActivities(i, MATCH_DEFAULT_ONLY);  
    //这里可以视实际情况加一些限制  
    return list.size() > 1;  
} 
```java
	
这样多开就无法逃脱我们的法眼了。


