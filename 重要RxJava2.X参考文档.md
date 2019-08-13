### 参考文章:

[ RxAndroid WIKI](https://github.com/ReactiveX/RxAndroid/wiki)   这个wiki所有android开发者必须去读,至少知道我们android上常用的Rx系列
[ RxJAVA WIKI](https://github.com/ReactiveX/RxJava/wiki) 这个wiki所有JAVA开发者必须去读
[Android RxJava：最基础的操作符详解 - 创建操作符](https://www.jianshu.com/p/e19f8ed863b1)
[RxJava Cold Observable 和 Hot Observable ](https://www.jianshu.com/p/12fb42bcf9fd)
[Android RxJava：图文详解 变换操作符](https://www.jianshu.com/p/904c14d253ba)
[Android RxJava：功能性操作符 全面讲解](https://www.jianshu.com/p/b0c3669affdb)
[Android RxJava：组合 / 合并操作符 详细教程](https://www.jianshu.com/p/c2a7c03da16d)
[<RxJava操作符大全>](https://www.jianshu.com/p/3fdd9ddb534b)
[<RxJava官网-Observable>](http://reactivex.io/documentation/observable.html)
[<RxJava官网-Operators>](http://reactivex.io/documentation/operators.html)
[<RxJava官网-Single>](http://reactivex.io/documentation/single.html)
[<RxJava官网-Subject>](http://reactivex.io/documentation/subject.html)
[<RxJava官网-Scheduler>](http://reactivex.io/documentation/scheduler.html)
[<RxJava2 实战系列文章>](https://www.jianshu.com/p/c935d0860186)
[RxJava2 基础认知](https://luhaoaimama1.github.io/2017/07/31/rxjava/)


### 参考DEMO

[ RxJava2Examples ](https://github.com/nanchen2251/RxJava2Examples)
[ RxSample ](https://github.com/imZeJun/RxSample)
[ RxJava2-Android-Samples ](https://github.com/amitshekhariitbhu/RxJava2-Android-Samples)
[Awesome-RxJava](https://github.com/zhaimi/Awesome-RxJava)  推荐看下


### 源码分析

[RxJava2.X 源码解析（一）： 探索RxJava2分发订阅流程](https://www.catbro.cn/detail/5a6037029a7b56c5c24fbc66.html)
[RxJava2.X 源码解析（二) ：探索RxJava2神秘的随意取消订阅流程的原理 ](https://www.catbro.cn/detail/5a6037029a7b56c5c24fbc68.html)
[RxJava2.X 源码分析（三）：探索RxJava2之订阅线程切换原理](https://www.catbro.cn/detail/5a6037029a7b56c5c24fbc69.html)
[RxJava2.X 源码分析（四）：探索RxJava2之观察者线程切换原理 ](https://www.catbro.cn/detail/5a6037029a7b56c5c24fbc6b.html)
[RxJava2.X 源码分析（五）：论RxJava2.X切换线程次数的有效性 ](https://www.catbro.cn/detail/5a6037029a7b56c5c24fbc6c.html)
[RxJava2.X 源码分析（六）：变换操作符的实现原理（上）](https://www.catbro.cn/detail/5a6037029a7b56c5c24fbc6d.html)
[RxJava2.X 源码分析（七）：变换操作符的实现原理强化篇（下）](https://www.catbro.cn/detail/5a6037029a7b56c5c24fbc6e.html)

## RxAndorid wiki 部分截取:

Q Since it cannot provide every type of binding to Android in one place, a  variety of third-party add-ons are available to fill in the gaps:

-  [RxLifecycle](https://github.com/trello/RxLifecycle) - Lifecycle handling APIs for Android apps using RxJava
-  [AutoDispose](https://github.com/uber/AutoDispose) - Automatic binding+disposal of RxJava 2 streams (from Uber)
-  [RxBinding](https://github.com/JakeWharton/RxBinding) - RxJava binding APIs for Android's UI widgets.
-  [SqlBrite](https://github.com/square/sqlbrite) - A lightweight wrapper around SQLiteOpenHelper and ContentResolver which introduces reactive stream semantics to queries.
-  [Android-ReactiveLocation](https://github.com/mcharmas/Android-ReactiveLocation) - Library that wraps location play services API boilerplate with a reactive friendly API. (RxJava 1)
-  [RxLocation](https://github.com/patloew/RxLocation) - Reactive Location APIs Library for Android. (RxJava 2)
-  [rx-preferences](https://github.com/f2prateek/rx-preferences) - Reactive SharedPreferences for Android
-  [RxFit](https://github.com/patloew/RxFit) - Reactive Fitness API Library for Android
-  [RxWear](https://github.com/patloew/RxWear) - Reactive Wearable API Library for Android
-  [RxPermissions](https://github.com/tbruyelle/RxPermissions) - Android runtime permissions powered by RxJava
-  [RxPermission](https://github.com/vanniktech/RxPermission) - Reactive permissions for Android
-  [RxNotification](https://github.com/pucamafra/RxNotification) - Easy way to register, remove and manage notifications using RxJava
-  [RxClipboard](https://github.com/zsavely/RxClipboard) - RxJava binding APIs for Android clipboard.
-  [RxBroadcast](https://github.com/cantrowitz/RxBroadcast) - RxJava bindings for `Broadcast` and `LocalBroadcast`.
-  [RxBroadcastReceiver](https://github.com/karczews/RxBroadcastReceiver) - Simple RxJava2 binding for Android BroadcastReceiver
-  [RxAndroidBle](https://github.com/Polidea/RxAndroidBle) - Reactive library for handling Bluetooth LE devices.
-  [RxImagePicker](https://github.com/MLSDev/RxImagePicker) - Reactive library for selecting images from gallery or camera.
-  [ReactiveNetwork](https://github.com/pwittchen/ReactiveNetwork) - Reactive library listening network connection state and Internet connectivity (compatible with RxJava1.x and RxJava2.x)
-  [ReactiveBeacons](https://github.com/pwittchen/ReactiveBeacons) - Reactive library scanning BLE (Bluetooth Low Energy) beacons nearby (compatible with RxJava1.x and RxJava2.x)
-  [ReactiveAirplaneMode](https://github.com/pwittchen/ReactiveAirplaneMode) - Reactive library listening airplane mode (compatible with RxJava2.x)
-  [ReactiveSensors](https://github.com/pwittchen/ReactiveSensors) - Reactive library monitoring device hardware sensors with RxJava (compatible with RxJava1.x and RxJava2.x)
-  [RxDataBinding](https://github.com/oldergod/RxDataBinding) - RxJava2 binding APIs for Android's Data Binding Library.
-  [RxLocationManager](https://github.com/Zellius/RxLocationManager) - RxJava/RxJava2 wrap around standard Android LocationManager without Google Play Services.
-  [RxDownloader](https://github.com/oussaki/RxDownloader) - Reactive library for downloading files (compatible with RxJava2.x).