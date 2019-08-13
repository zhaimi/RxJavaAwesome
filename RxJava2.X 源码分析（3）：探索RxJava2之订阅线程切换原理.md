### 一、前言

- 基于RxJava2.1.1
- 我们在前面的 [RxJava2.0使用详解（一）](http://cherylgood.cn/articles/2017/07/10/1499671003613.html)初步分析了RxJava从创建到执行的流程。[RxJava2.0使用详解（二) ](http://cherylgood.cn/articles/2017/07/11/1499770780242.html)中分析了RxJava的随意终止Reactive流的能力的来源；也明白了`RxJava`的`onComplete();`与`onError(t);`只有一个会被执行的秘密。
- 本次我们将探索RxJava2.x线程切换的实现原理。做到知其然，知其所以然。
- Ok，开始我们的探索之旅吧！

### 二、从Demo到源码

- 本次我们将在上次的demo基础了做点改动。

  ```
    //1、观察者创建一个Observer
    Observer observer = new Observer() {
        @Override
      public void onSubscribe(@NonNull Disposable d) {
            Log.d(TAG, "onSubscribe");
            Log.d(TAG,"work thread is"+Thread.currentThread().getName());
            disposable = d;
        }
  
        @Override
      public void onNext(@NonNull String s) {
            Log.d(TAG, "onNext data is :" + s);
            Log.d(TAG,"work thread is"+Thread.currentThread().getName());
            if (s.equals("hello")) {
                //执行了hello之后终止
      disposable.dispose();
                disposable.dispose();
            }
            CompositeDisposable compositeDisposable = new CompositeDisposable();
            compositeDisposable.dispose();
  
        }
  
        @Override
      public void onError(@NonNull Throwable e) {
            Log.d(TAG, "onError data is :" + e.toString());
        }
  
        @Override
      public void onComplete() {
            Log.d(TAG, "onComplete");
        }
    };
  
    Observable observable = Observable.create(new ObservableOnSubscribe() {
        @Override
      public void subscribe(@NonNull ObservableEmitter e) throws Exception {
            Log.d(TAG,"work thread is"+Thread.currentThread().getName());
            e.onNext("hello");
            Log.i(TAG, "发送 hello");
            e.onNext("world");
            Log.i(TAG, "发送 world");
            e.onComplete();
            Log.i(TAG, "调用 onComplete");
  
        }
  
    });
  ```

- 版本1:不存在线程切换

  ```
    observable.subscribe(observer);
  ```

- 输出结果：

  ```
    07-13 14:58:13.173 818-865/? D/RxJavaDemo2: onSubscribe
    07-13 14:58:13.173 818-865/? D/RxJavaDemo2: work thread isInstr: android.support.test.runner.AndroidJUnitRunner
    07-13 14:58:13.173 818-865/? D/RxJavaDemo2: work thread isInstr: android.support.test.runner.AndroidJUnitRunner
    07-13 14:58:13.173 818-865/? D/RxJavaDemo2: onNext data is :hello
    07-13 14:58:13.173 818-865/? D/RxJavaDemo2: work thread isInstr: android.support.test.runner.AndroidJUnitRunner
    07-13 14:58:13.173 818-865/? I/RxJavaDemo2: 发送 hello
    07-13 14:58:13.173 818-865/? I/RxJavaDemo2: 发送 world
    07-13 14:58:13.173 818-865/? I/RxJavaDemo2: 调用 onComplete
    07-13 14:58:13.173 818-865/? I/TestRunner: finished: testRx(com.guanaj.rxdemo.RxJavaDemo2)
  ```

- 版本2:切换线程（切换操作是如此的潇洒）

  ```
    observable
            .subscribeOn(Schedulers.io())//切换到io线程
            .observeOn(AndroidSchedulers.mainThread())//切换到主线程
            .subscribe(observer);
  ```

- 输出结果：

  ```
    07-13 14:43:56.699 29727-29749/? D/RxJavaDemo2: onSubscribe
    07-13 14:43:56.699 29727-29749/? D/RxJavaDemo2: work thread isInstr: android.support.test.runner.AndroidJUnitRunner
    07-13 14:43:56.699 29727-29749/? I/TestRunner: finished: testRx(com.guanaj.rxdemo.RxJavaDemo2)
    07-13 14:43:56.699 29727-29753/? D/RxJavaDemo2: work thread isRxCachedThreadScheduler-1
    07-13 14:43:56.699 29727-29753/? I/RxJavaDemo2: 发送 hello
    07-13 14:43:56.699 29727-29753/? I/RxJavaDemo2: 发送 world
    07-13 14:43:56.699 29727-29753/? I/RxJavaDemo2: 调用 onComplete
    07-13 14:43:56.699 29727-29727/? D/RxJavaDemo2: onNext data is :hello
    07-13 14:43:56.699 29727-29727/? D/RxJavaDemo2: work thread ismain
  ```

- 结果分析（因为我用的是@RunWith(AndroidJUnit4.class)执行代码，所以在工作线程是AndroidJUnitRunner）：

- 现在我们现象，后面根据现象分析原因。

  ### 没线程切换的版本：

  1、在那里调用subscribe，则都在当前线程执行。

  ### 存在版本切换的版本：

  1、被观察者的onSubscribe在调用subscribe的线程中执行，

  2、被观察者的subscribe在RxJava2的RxCachedThreadScheduler-1中运行。

  3、onNext工作在主线程

------

- OK，现象看完了，我们开始看本质吧！但是，从哪入手呢？还是老办法，哪里触发的行为就哪里下手(￣∇￣)

- OK，我们先来探索切换Observable工作线程的`subscribeOn`方法入手

  ```
    @CheckReturnValue
    @SchedulerSupport(SchedulerSupport.CUSTOM)
    public final Observable<T> subscribeOn(Scheduler scheduler) {
        ObjectHelper.requireNonNull(scheduler, "scheduler is null");
        return RxJavaPlugins.onAssembly(new ObservableSubscribeOn<T>(this, scheduler));
    }
  ```

- 看到了RxJavaPlugins.onAssembly，前面分析过，为`hook`服务，现在看成是返回传入的`Obserable`即可。这里的`this`为我们的`observable`,`scheduler`就是我们传入的`Schedulers.io()`;我们继续看`ObservableSubscribeOn`；

  ```
    public final class ObservableSubscribeOn<T> extends AbstractObservableWithUpstream<T, T> {....}
  ```

- 其继承`AbstractObservableWithUpstream`

```
    abstract class AbstractObservableWithUpstream<T, U> extends Observable<U> implements HasUpstreamObservableSource<T> {

        /** The source consumable Observable. */
      protected final ObservableSource<T> source;

        /**
     * Constructs the ObservableSource with the given consumable. * @param source the consumable Observable
     */  AbstractObservableWithUpstream(ObservableSource<T> source) {
            this.source = source;
        }

        @Override
      public final ObservableSource<T> source() {
            return source;
        }

    }
```

- `AbstractObservableWithUpstream`继承自`Observable`，其作用是通过`source`字段保存上游的`Observable`,因为`Observable`是`ObservableSource`接口的实现类，所以我们可以**认为Observable和ObservableSource在本文中是相等的**：，
- 也就是说`ObservableSubscribeOn`是对`Observble`进行了一次wrapper操作

------

- OK，我们继续回来看`ObservableSubscribeOn`的源码

  ```
    public final class ObservableSubscribeOn<T> extends AbstractObservableWithUpstream<T, T> {
        //1、线程调度器
        final Scheduler scheduler;
        public ObservableSubscribeOn(ObservableSource<T> source, Scheduler scheduler) {
            //2、存储上游的obserble
            super(source);
            this.scheduler = scheduler;
        }
  
        @Override
      public void subscribeActual(final Observer<? super T> s) {
          //以下为关键部分
            //3、对我们下游的observer进行一次wrapper
            final SubscribeOnObserver<T> parent = new SubscribeOnObserver<T>(s);
            //4、同样，先自己调用自己的onSubscribe
            s.onSubscribe(parent);
            //5、（高能量聚集了）将调度的线程的Disposable赋值给当前的Disposable。scheduler可以看成是某个线程上的调度器。new SubscribeTask(parent)工作在其指定的线程里面。SubscribeTask是一个Runnable，也就是说调度器触发Runnable的run()运行，
            //***是不是恍然大悟，那么run()里面的代码就是运行在scheduler的线程上了。这样也就实现了线程的切换了。
            parent.setDisposable(scheduler.scheduleDirect(new SubscribeTask(parent)));
        }
        static final class SubscribeOnObserver<T> extends AtomicReference implements Observer<T>, Disposable {....}
        ...
        }
  ```

- Ok，我们开看下SubscribeTask

  ```
    final class SubscribeTask implements Runnable {
        private final SubscribeOnObserver<T> parent;
  
        SubscribeTask(SubscribeOnObserver<T> parent) {
            this.parent = parent;
        }
  
        @Override
      public void run() {
            source.subscribe(parent);
        }
    }
  ```

- 1、`parent`就是我们包装后的`observer`,其内部保存了下游的`observer`

- 2、`source`即通过`ObservableSubscribeOn`wrapper后存储我们上游的`observable`

- 所以

  ```
  run
  ```

  里面的

  ```
  source.subscribe(parent);
  ```

  即为wrapper的

  ```
  observer
  ```

  订阅了上游的

  ```
  observable
  ```

  ,触发了上游

  ```
  observable
  ```

  的

  ```
  subscribeActual
  ```

  ，开始执行数据的分发

  > 上层obserable -》wrapper产生的observer -》真实的observser

#### 思路梳理（重要）

- Ok，分析到这里思路基本清晰了
  1、在执行subscribeOn时，在Observable和observer中插入了一个(wrapper)obserabler和(wrapper)Observer

  > `原observable->【(Wrapper)observable||(Wrapper)Observer】->（原）Observer`

- 2、`observable.subscribe`触发->(Wrapper)`Observable.subscribeActual()`内部调用->`parent.setDisposable(scheduler.scheduleDirect(new SubscribeTask(parent)));`,->`scheduler`在指定线程调用(完成线程切换)->`SubscribeTask.run`,run内部调用->`原Observable.subscribe((Wrapper)Observer)`触发->(原)`Observable.subscribeActual()`开始数据分发

- Ok,此时分发给的是`(Wrapper)Observer`，那应该是(Wrapper)Observer分发给了`(原）Observer`,我们看下是不是

------

- OK，(Wrapper)Observer对`(原)Observer`进行了wrapper，我们看下源码：

  ```
    static final class SubscribeOnObserver<T> extends AtomicReference implements Observer<T>, Disposable {
        //6、存储下游的observer
        final Observer<? super T> actual;
        //7、保存下游observer的Disposable，下游的Disposable交由其管理
        final AtomicReference<Disposable> s;
  
        SubscribeOnObserver(Observersuper T> actual) {
            this.actual = actual;
            this.s = new AtomicReference<Disposable>();
        }
  
        @Override
      public void onSubscribe(Disposable s) {
      //8、onSubscribe()方法在observer调用subscribe时触发，Observer传入自己的Disposable，赋值给this.s，交由当前的包装的Observer管理。同样是装饰者模式的魅力所在。
            DisposableHelper.setOnce(this.s, s);
        }
        //当前Observer可以理解为下游observer和上游obserable的一个中间observer。
        //9、这里直接调用下游observer的对应方法。
        @Override
      public void onNext(T t) {
            actual.onNext(t);
        }
  
        @Override
      public void onError(Throwable t) {
            actual.onError(t);
        }
  
        @Override
      public void onComplete() {
            actual.onComplete();
        }
      //10、取消订阅时，要同时取消下游的observer和当前的observer，因为上游obserable分发订阅数据判断是否需要派发时判断的是与之最近的observer。
      //上层obserable-》wrapper产生的observer》真实的observser
        @Override
      public void dispose() {
            DisposableHelper.dispose(s);
            DisposableHelper.dispose(this);
        }
  
        @Override
      public boolean isDisposed() {
            return DisposableHelper.isDisposed(get());
        }
        //11、subscribeActual()中被调用，目的是将Schedulers返回的Worker加入管理
        void setDisposable(Disposable d) {
            DisposableHelper.setOnce(this, d);
        }
    }
  ```

- Ok，确实是(Wrapper)Observer分发给了`(原）Observer`。

- 到这里的时候，整个流程基本OK了，但是，我们在`5`和`11`处说了,调度Worker也会加入Disposable进行管理，我还是要一探究竟(￣∇￣)。

------

- Ok，有了对SubscribeOnObserver分析的铺垫，我们现在可以分析第5处`parent.setDisposable(scheduler.scheduleDirect(new SubscribeTask(parent)));`的代码了，我们先看`scheduler.scheduleDirect()`这句

  ```
    @NonNull
    public Disposable scheduleDirect(@NonNull Runnable run) {
        //12、以毫秒为单位，无延迟调度
        return scheduleDirect(run, 0L, TimeUnit.NANOSECONDS);
    }
  ```

- OK，其返回一个`Disposable`,我们看下这个`Disposable`是否真的是调度的线程的。

  ```
    @NonNull
    public Disposable scheduleDirect(@NonNull Runnable run, long delay, @NonNull TimeUnit unit) {
        //13、Worker实现了Disposable的一个调度工作者类
        final Worker w = createWorker();
        //14、hook，排除hook干扰，可以理解为decoratedRun==run
        final Runnable decoratedRun = RxJavaPlugins.onSchedule(run);
        //15、DisposeTask同样是实现了Disposable的Task
        DisposeTask task = new DisposeTask(decoratedRun, w);
        //16、开始执行
        w.schedule(task, delay, unit);
        //17、确实是返回了管理run的worker
        return task;
    }
  ```

- Ok，现在终点转移到`DisposeTask`,我们把run给了`DisposeTask`，然后`worker`调度task开始执行`run`

- OK,那么我们可以猜测`w.schedule(task, delay, unit)`执行后应该是调度了task的某个方法，然后`task`开始执行`Runnable`的`run`

- 是不是真的呢？我们来看下`new DisposeTask(decoratedRun, w)`做了什么

  ```
    static final class DisposeTask implements Runnable, Disposable {
        //18、我们外部传入的runnable
        final Runnable decoratedRun;
        //19、调度工作者
        final Worker w;
        //20、当前线程
        Thread runner;
        DisposeTask(Runnable decoratedRun, Worker w) {
            this.decoratedRun = decoratedRun;
            this.w = w;
        }
  
        @Override
      public void run() {
            //21、获取执decoratedRun的线程
            runner = Thread.currentThread();
            try {
              //22、OK，高能来了。decoratedRun的run被执行
                decoratedRun.run();
            } finally {
                dispose();
                runner = null;
            }
        }
  
        @Override
      public void dispose() {
  
            if (runner == Thread.currentThread() && w instanceof NewThreadWorker) {
                ((NewThreadWorker)w).shutdown();
            } else {
                //在DisposeTask被取消时，告诉Worker取消，因为DisposeTask是Worker进行管理的
                w.dispose();
            }
        }
  
        @Override
      public boolean isDisposed() {
            return w.isDisposed();
        }
    }
  ```

  #### 结论：

- scheduler.scheduleDirect无延迟调用->worker->worker调度->DisposeTask->DisposeTask.run执行->decoratedRun.run();

- decoratedRun即我们外部的SubscribeTask

### 总结

- 我们从subscribeOn入手分析了Observable线程切换的流程。其基本是通过中间插入包装类，也就是装饰者模式的体现，巧妙的实现了线程的切换。
- 其内部也对Disposed做了处理，保证Disposed的传递。
- 装饰者模式的使用贯穿了RxJava2的各处(个人理解），再次体会了设计模式的魅力。
- 由于本篇过长，observeOn订阅者线程的切换就再分一篇吧。

