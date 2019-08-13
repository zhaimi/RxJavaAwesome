 # RxJava2.X 源码分析（五）：论RxJava2.X切换线程次数的有效性 

- ### 一、前言

- 之前写了四篇从Demo到源码、从表现到内部实现原理，通过源码的分析初步学习了RxJava2.X的一些基本操作及原理，有如下几点
  1、`Observable`与`Observer`是如何发生订阅关系的
  2、`onNext`、`onComplete`、`onError`被调用的次数限制及实现流程
  3、`onSubscribe`方法为何会第一个被调用？及如何控制`Disposable`来取消订阅事件
  4、分两篇分析了RxJava2.X切换订阅线程和观察者线程的源码

- 接下来我们将根据之前的分析成果从设计上分析RxJava2.X多次切换线程的有效性

### 二、具体分析

- #### 1、切换订阅事件线程的有效性

- 在[RxJava2.X 源码分析（三）：探索RxJava2之订阅线程切换原理 ](http://www.catbro.cn/articles/2017/07/13/1499925392920.html)中我们分析了订阅线程切换的源码。

- 订阅事件的传递是从下往上传递，最终传递到上游被订阅者执行订阅流程

- 假设有三级，每级均发生线程切换：   

  > 下游Observer（订阅）->2级Observable（调用）  2级Observer（切换线程1订阅）->1级Observable （调用）1级Obsever  （切换线程2订阅）->上游Observable 触发真正的订阅事件  下发数据->1级Obsever（接收后下发）->2级Obsevser （接收后下发）->下游Obsever

- Ok，很显然，即使呢N此调用切换订阅线程的api接口，真正作用于订阅事件的线程是最接近上游Obsevable的一次。根据RxJava的调用习惯也就是第一次，所以`subscribeOn`的调用只有第一次生效

- #### 2、切换观察者线程的有效性

- 我们在[RxJava2.X 源码分析（四）](http://www.catbro.cn/articles/2017/07/14/1500020801533.html)中分析了观察者事件线程切换的源码

- 订阅数据的数据流是从上而下下发的，最终传递到下游的观察者的onXXX回调方法内

- 同样，假设有三级，美级均发生线程切换

  > 下游Observer（订阅）->2级Observable（调用） 2级Observer（订阅）->1级Observable  （调用）1级Obsever （订阅）->上游Observable 触发真正的订阅事件  下发数据->1级Obsever（接后切换线程1回调onXXX方法下发数据）->2级Obsevser  （接收后切换线程1回调onXXX方法下发数据））->下游Obsever 的onXXX回调方法收到数据

- Ok，很显然，每级的Observer的onXXX方法都在不同的线程中被调用。所以`observeOn`的调用会多次生效

### 三、总结

- Ok，本篇篇幅相对前面几篇，是不是长度很满意。
- 写这篇的目的有二
  1、梳理前两篇的调用次序
  2、分析`observeOn`与`subscribeOn`调用顺序的影响及有效性