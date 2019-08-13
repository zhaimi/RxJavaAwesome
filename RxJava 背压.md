#### 背压（backpressure）

当上下游在不同的线程中，通过Observable发射，处理，响应数据流时，如果上游发射数据的速度快于下游接收处理数据的速度，这样对于那些没来得及处理的数据就会造成积压，这些数据既不会丢失，也不会被垃圾回收机制回收，而是存放在一个异步缓存池中，如果缓存池中的数据一直得不到处理，越积越多，最后就会造成内存溢出，这便是响应式编程中的背压（backpressure）问题。
 例如，运行以下代码：

```
   public void demo1() {
        Observable
                .create(new ObservableOnSubscribe<Integer>() {
                    @Override
                    public void subscribe(ObservableEmitter<Integer> e) throws Exception {
                        int i = 0;
                        while (true) {
                            i++;
                            e.onNext(i);
                        }
                    }
                })
                .subscribeOn(Schedulers.newThread())
                .observeOn(Schedulers.newThread())
                .subscribe(new Consumer<Integer>() {
                    @Override
                    public void accept(Integer integer) throws Exception {
                        Thread.sleep(5000);
                        System.out.println(integer);
                    }
                });
    }
```

创建一个可观察对象Observable在Schedulers.newThread()的线程中不断发送数据，而观察者Observer在Schedulers.newThread()的另一个线程中每隔5秒接收打印一条数据。
 运行后，查看内存使用如下：



![img](https:////upload-images.jianshu.io/upload_images/6773051-d68c51355a9d8e2d.gif?imageMogr2/auto-orient/strip%7CimageView2/2/w/552)

backpressure.gif



由于上游通过Observable发射数据的速度大于下游通过Consumer接收处理数据的速度，而且上下游分别运行在不同的线程中，下游对数据的接收处理不会堵塞上游对数据的发射，造成上游数据积压，内存不断增加，最后便会导致内存溢出。

#### Flowable

既然在函数响应式编程中会产生背压（backpressure）问题，那么在函数响应式编程中就应该有解决方案。
 Rxjava2相对于Rxjava1最大的更新就是把对背压问题的处理逻辑从Observable中抽取出来产生了新的可观察对象Flowable。

在Rxjava2中，Flowable可以看做是为了解决背压问题，在Observable的基础上优化后的产物，与Observable不处在同一组观察者模式下，Observable是ObservableSource/Observer这一组观察者模式中ObservableSource的典型实现，而Flowable是Publisher与Subscriber这一组观察者模式中Publisher的典型实现。

所以在使用Flowable的时候，可观察对象不再是Observable,而是Flowable;观察者不再是Observer，而是Subscriber。Flowable与Subscriber之间依然通过subscribe()进行关联。

虽然在Rxjava2中，Flowable是在Observable的基础上优化后的产物，Observable能解决的问题Flowable也都能解决，但是并不代表Flowable可以完全取代Observable,在使用的过程中，并不能抛弃Observable而只用Flowable。

由于基于Flowable发射的数据流，以及对数据加工处理的各操作符都添加了背压支持，附加了额外的逻辑，其运行效率要比Observable慢得多。

**只有在需要处理背压问题时，才需要使用Flowable。**

由于只有在上下游运行在不同的线程中，且上游发射数据的速度大于下游接收处理数据的速度时，才会产生背压问题；
 所以，如果能够确定：
 1、上下游运行在同一个线程中，
 2、上下游工作在不同的线程中，但是下游处理数据的速度不慢于上游发射数据的速度，
 3、上下游工作在不同的线程中，但是数据流中只有一条数据
 则不会产生背压问题，就没有必要使用Flowable，以免影响性能。

类似于Observable,在使用Flowable时，也可以通过create操作符创建发射数据流，代码如下：

```
public void demo2() {
        Flowable
                .create(new FlowableOnSubscribe<Integer>() {
                    @Override
                    public void subscribe(FlowableEmitter<Integer> e) throws Exception {
                        System.out.println("发射----> 1");
                        e.onNext(1);
                        System.out.println("发射----> 2");
                        e.onNext(2);
                        System.out.println("发射----> 3");
                        e.onNext(3);
                        System.out.println("发射----> 完成");
                        e.onComplete();
                    }
                }, BackpressureStrategy.BUFFER) //create方法中多了一个BackpressureStrategy类型的参数
                .subscribeOn(Schedulers.newThread())//为上下游分别指定各自的线程
                .observeOn(Schedulers.newThread())
                .subscribe(new Subscriber<Integer>() {
                    @Override
                    public void onSubscribe(Subscription s) {   //onSubscribe回调的参数不是Disposable而是Subscription
                        s.request(Long.MAX_VALUE);            //注意此处，暂时先这么设置
                    }

                    @Override
                    public void onNext(Integer integer) {
                        System.out.println("接收----> " + integer);
                    }

                    @Override
                    public void onError(Throwable t) {
                    }

                    @Override
                    public void onComplete() {
                        System.out.println("接收----> 完成");
                    }
                });
    }
```

运行结果如下：

```
System.out: 发射----> 1
System.out: 发射----> 2
System.out: 发射----> 3
System.out: 发射----> 完成
System.out: 接收----> 1
System.out: 接收----> 2
System.out: 接收----> 3
System.out: 接收----> 完成
```

发射与处理数据流在形式上与Observable大同小异，发射器中均有onNext，onError，onComplete方法，订阅器中也均有onSubscribe，onNext，onError，onComplete方法。

但是在细节方面还是有三点不同
 一、create方法中多了一个**BackpressureStrategy**类型的参数。
 二、订阅器Subscriber中，方法onSubscribe回调的参数不是Disposable而是**Subscription**，多了行代码：

```
s.request(Long.MAX_VALUE);
```

三、Flowable发射数据时，使用特有的发射器**FlowableEmitter**，不同于Observable的ObservableEmitter

正是这三点不同赋予了Flowable不同于Observable的特性。

#### BackpressureStrategy背压策略

在通过create操作符创建Flowable时，多了一个BackpressureStrategy类型的参数，BackpressureStrategy是个枚举，源码如下：

```
package io.reactivex;

/**
 * Represents the options for applying backpressure to a source sequence.
 */
public enum BackpressureStrategy {
    /**
     * OnNext events are written without any buffering or dropping.
     * Downstream has to deal with any overflow.
     * <p>Useful when one applies one of the custom-parameter onBackpressureXXX operators.
     */
    MISSING,
    /**
     * Signals a MissingBackpressureException in case the downstream can't keep up.
     */
    ERROR,
    /**
     * Buffers <em>all</em> onNext values until the downstream consumes it.
     */
    BUFFER,
    /**
     * Drops the most recent onNext value if the downstream can't keep up.
     */
    DROP,
    /**
     * Keeps only the latest onNext value, overwriting any previous value if the
     * downstream can't keep up.
     */
    LATEST
}
```

当上游发送数据的速度快于下游接收数据的速度，且运行在不同的线程中时，Flowable通过自身特有的异步缓存池，来缓存没来得及处理的数据，缓存池的容量上限为128，在Flowable源码的开头即可看到

```
  /** The default buffer size. */
    static final int BUFFER_SIZE;
    static {
        BUFFER_SIZE = Math.max(1, Integer.getInteger("rx2.buffer-size", 128));
    }
```

不同于Observable，其异步缓存没有容量限制，对于没来得及处理的数据可以一直向里面添加，数据过多就会产生内存溢出（OOM）。

BackpressureStrategy的作用便是用来设置Flowable通过异步缓存池缓存数据的策略。在源码FlowableCreate类中，可以看到五个泛型分别对应五个java类

```
MISSING   ----> MissingEmitter
ERROR     ----> ErrorAsyncEmitter
DROP      ----> DropAsyncEmitter
LATEST    ----> LatestAsyncEmitter
BUFFER    ----> BufferAsyncEmitter
```

通过代理模式对原始的发射器进行了包装。

```
@Override
    public void subscribeActual(Subscriber<? super T> t) {
        BaseEmitter<T> emitter;

        switch (backpressure) {
            case MISSING: {
                emitter = new MissingEmitter<T>(t);
                break;
            }
            case ERROR: {
                emitter = new ErrorAsyncEmitter<T>(t);
                break;
            }
            case DROP: {
                emitter = new DropAsyncEmitter<T>(t);
                break;
            }
            case LATEST: {
                emitter = new LatestAsyncEmitter<T>(t);
                break;
            }
            default: {
                emitter = new BufferAsyncEmitter<T>(t, bufferSize());
                break;
            }
        }

        t.onSubscribe(emitter);
        try {
            source.subscribe(emitter);
        } catch (Throwable ex) {
            Exceptions.throwIfFatal(ex);
            emitter.onError(ex);
        }
    }
```

##### ERROR

对应于ErrorAsyncEmitter<T>类，在其源码

```
static final class ErrorAsyncEmitter<T> extends NoOverflowBaseAsyncEmitter<T> {
        private static final long serialVersionUID = 338953216916120960L;

        ErrorAsyncEmitter(Subscriber<? super T> actual) {
            super(actual);
        }

        @Override
        void onOverflow() {
            onError(new MissingBackpressureException("create: could not emit value due to lack of requests"));
        }

    }
```

onOverflow方法中可以看到
 在此策略下，如果放入Flowable的异步缓存池中的数据超限了，则会抛出MissingBackpressureException异常。

运行如下代码：

```
public void demo3() {
        Flowable
                .create(new FlowableOnSubscribe<Integer>() {
                    @Override
                    public void subscribe(FlowableEmitter<Integer> e) throws Exception {
                        for (int i = 1; i <= 129; i++) {
                            e.onNext(i);
                        }
                        e.onComplete();
                    }
                }, BackpressureStrategy.ERROR)
                .subscribeOn(Schedulers.newThread())
                .observeOn(Schedulers.newThread())
                .subscribe(new Subscriber<Integer>() {
                    @Override
                    public void onSubscribe(Subscription s) {
                        s.request(Long.MAX_VALUE);            //注意此处，暂时先这么设置
                    }

                    @Override
                    public void onNext(Integer integer) {
                        try {
                            Thread.sleep(10000);
                        } catch (InterruptedException ignore) {
                        }
                        System.out.println(integer);
                    }

                    @Override
                    public void onError(Throwable t) {
                        t.printStackTrace();
                    }

                    @Override
                    public void onComplete() {
                        System.out.println("接收----> 完成");
                    }
                });
    }
```

创建并通过Flowable发射129条数据，Subscriber的onNext方法睡10秒之后再开始接收，运行后会发现控制台打印如下异常：

```
W/System.err: io.reactivex.exceptions.MissingBackpressureException: create: could not emit value due to lack of requests
W/System.err:     at io.reactivex.internal.operators.flowable.FlowableCreate$ErrorAsyncEmitter.onOverflow(FlowableCreate.java:411)
W/System.err:     at io.reactivex.internal.operators.flowable.FlowableCreate$NoOverflowBaseAsyncEmitter.onNext(FlowableCreate.java:377)
W/System.err:     at net.fbi.rxjava2.RxJava2Demo$6.subscribe(RxJava2Demo.java:103)
W/System.err:     at io.reactivex.internal.operators.flowable.FlowableCreate.subscribeActual(FlowableCreate.java:72)
W/System.err:     at io.reactivex.Flowable.subscribe(Flowable.java:12218)
W/System.err:     at io.reactivex.internal.operators.flowable.FlowableSubscribeOn$SubscribeOnSubscriber.run(FlowableSubscribeOn.java:82)
W/System.err:     at io.reactivex.internal.schedulers.ScheduledRunnable.run(ScheduledRunnable.java:59)
W/System.err:     at io.reactivex.internal.schedulers.ScheduledRunnable.call(ScheduledRunnable.java:51)
W/System.err:     at java.util.concurrent.FutureTask.run(FutureTask.java:237)
W/System.err:     at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.access$201(ScheduledThreadPoolExecutor.java:154)
W/System.err:     at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:269)
W/System.err:     at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1113)
W/System.err:     at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:588)
W/System.err:     at java.lang.Thread.run(Thread.java:818)
```

如果将Flowable发射数据的条数改为128，则不会出现此异常。

##### DROP

对应于DropAsyncEmitter<T>类，通过DropAsyncEmitter类和它父类NoOverflowBaseAsyncEmitter的源码

```
    static final class DropAsyncEmitter<T> extends NoOverflowBaseAsyncEmitter<T> {
        private static final long serialVersionUID = 8360058422307496563L;

        DropAsyncEmitter(Subscriber<? super T> actual) {
            super(actual);
        }

        @Override
        void onOverflow() {
            // nothing to do
        }
    }
abstract static class NoOverflowBaseAsyncEmitter<T> extends BaseEmitter<T> {

        private static final long serialVersionUID = 4127754106204442833L;

        NoOverflowBaseAsyncEmitter(Subscriber<? super T> actual) {
            super(actual);
        }

        @Override
        public final void onNext(T t) {
            if (isCancelled()) {
                return;
            }

            if (t == null) {
                onError(new NullPointerException("onNext called with null. Null values are generally not allowed in 2.x operators and sources."));
                return;
            }

            if (get() != 0) {
                actual.onNext(t);
                BackpressureHelper.produced(this, 1);
            } else {
                onOverflow();
            }
        }

        abstract void onOverflow();
    }
```

可以看到，DropAsyncEmitter的onOverflow是个空方法，没有执行任何操作，父类的onNext中，在判断get() != 0，即缓存池未满的情况下，才会让被代理类调用onNext方法。
 所以在此策略下，如果Flowable的异步缓存池满了，会丢掉上游发送的数据。
 运行如下代码：

```
public void demo4() {
        Flowable
                .create(new FlowableOnSubscribe<Integer>() {
                    @Override
                    public void subscribe(FlowableEmitter<Integer> e) throws Exception {
                        String threadName = Thread.currentThread().getName();
                        System.out.println(threadName + "开始发射数据" + System.currentTimeMillis());
                        for (int i = 1; i <= 500; i++) {
                            System.out.println(threadName + "发射---->" + i);
                            e.onNext(i);
                            try {
                                Thread.sleep(100);//每隔100毫秒发射一次数据
                            } catch (Exception ex) {
                                e.onError(ex);
                            }
                        }
                        System.out.println(threadName + "发射数据结束" + System.currentTimeMillis());
                        e.onComplete();
                    }
                }, BackpressureStrategy.DROP)
                .subscribeOn(Schedulers.newThread())
                .observeOn(Schedulers.newThread())
                .subscribe(new Subscriber<Integer>() {
                    @Override
                    public void onSubscribe(Subscription s) {
                        s.request(Long.MAX_VALUE);            //注意此处，暂时先这么设置
                    }

                    @Override
                    public void onNext(Integer integer) {
                        try {
                            Thread.sleep(300);//每隔300毫秒接收一次数据
                        } catch (InterruptedException ignore) {
                        }
                        System.out.println(Thread.currentThread().getName() + "接收---------->" + integer);
                    }

                    @Override
                    public void onError(Throwable t) {
                        t.printStackTrace();
                    }

                    @Override
                    public void onComplete() {
                        System.out.println(Thread.currentThread().getName() + "接收----> 完成");
                    }
                });
    }
```

通过创建Flowable发射500条数据，每隔100毫秒发射一次，并记录开始发射和结束发射的时间，下游每隔300毫秒接收一次数据，运行后，控制台打印日志如下：



![img](https:////upload-images.jianshu.io/upload_images/6773051-9c3f26ad32d558a3.gif?imageMogr2/auto-orient/strip%7CimageView2/2/w/424)

GIF111.gif

通过日志



![img](https:////upload-images.jianshu.io/upload_images/6773051-eaf4ad8457018c72.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/359)

1.jpg



我们可以发现Subscriber在接收完第128条数据后，再次接收的时候已经到了288，而这之间的160条数据正是因为缓存池满了而被丢弃掉了。
 那么问题来了，当Flowable在发射第129条数据的时候，Subscriber已经接收了42条数据了，第129条数据为什么没有放入缓存池中呢？日志如下：





![img](https:////upload-images.jianshu.io/upload_images/6773051-c23597bb66f951df.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/344)

2.jpg

那是因为缓存池中数据的清理，并不是Subscriber接收一条，便清理一条，而是存在一个延迟，等累积一段时间后统一清理一次。也就是Subscriber接收到第96条数据时，缓存池才开始清理数据，之后Flowable发射的数据才得以放入。



![img](https:////upload-images.jianshu.io/upload_images/6773051-f3b96438710af53c.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/367)

3.jpg

查看日志可以发现，Subscriber接收到第96条数据后，Flowable发射第288条数据。而第128到288之间的数据，正好处于缓存池存满的状态，而被丢弃，所以Subscriber在接收完第128条数据之后，接收到的是第288条数据，而不是第129条。

##### LATEST

对应于LatestAsyncEmitter<T>类
 与Drop策略一样，如果缓存池满了，会丢掉将要放入缓存池中的数据，不同的是，不管缓存池的状态如何，LATEST都会将最后一条数据强行放入缓存池中，来保证观察者在接收到完成通知之前，能够接收到Flowable最新发射的一条数据。
 将上述代码中的DROP策略改为LATEST：

```
public void demo5() {
        Flowable
                .create(new FlowableOnSubscribe<Integer>() {
                    @Override
                    public void subscribe(FlowableEmitter<Integer> e) throws Exception {
                        String threadName = Thread.currentThread().getName();
                        System.out.println(threadName + "开始发射数据" + System.currentTimeMillis());
                        for (int i = 1; i <= 500; i++) {
                            System.out.println(threadName + "发射---->" + i);
                            e.onNext(i);
                            try {
                                Thread.sleep(100);
                            } catch (Exception ignore) {
                            }
                        }
                        System.out.println(threadName + "发射数据结束" + System.currentTimeMillis());
                        e.onComplete();

                    }
                }, BackpressureStrategy.LATEST)
                .subscribeOn(Schedulers.newThread())
                .observeOn(Schedulers.newThread())
                .subscribe(new Subscriber<Integer>() {
                    @Override
                    public void onSubscribe(Subscription s) {
                        s.request(Long.MAX_VALUE);            //注意此处，暂时先这么设置
                    }

                    @Override
                    public void onNext(Integer integer) {
                        try {
                            Thread.sleep(300);
                        } catch (InterruptedException ignore) {
                        }
                        System.out.println(Thread.currentThread().getName() + "接收---------->" + integer);
                    }

                    @Override
                    public void onError(Throwable t) {
                        t.printStackTrace();
                    }

                    @Override
                    public void onComplete() {
                        System.out.println(Thread.currentThread().getName() + "接收----> 完成");
                    }
                });
    }
```

运行后日志对比如下：
 DROP:





![img](https:////upload-images.jianshu.io/upload_images/6773051-731d7d2456cbc431.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/360)

DROP.jpg



LATEST：



![img](https:////upload-images.jianshu.io/upload_images/6773051-4f559f1f819ad03b.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/355)

LATEST.jpg

latest策略下Subscriber在接收完成之前，接收的数据是Flowable发射的最后一条数据，而Drop策略下不是。

##### BUFFER

Flowable处理背压的默认策略，对应于BufferAsyncEmitter<T>类
 其部分源码为：

```
static final class BufferAsyncEmitter<T> extends BaseEmitter<T> {
        private static final long serialVersionUID = 2427151001689639875L;
        final SpscLinkedArrayQueue<T> queue;
        . . . . . .
        final AtomicInteger wip;
        BufferAsyncEmitter(Subscriber<? super T> actual, int capacityHint) {
            super(actual);
            this.queue = new SpscLinkedArrayQueue<T>(capacityHint);
            this.wip = new AtomicInteger();
        }
        . . . . . .
}
```

在其构造方法中可以发现，其内部维护了一个缓存池SpscLinkedArrayQueue，其大小不限，此策略下，如果Flowable默认的异步缓存池满了，会通过此缓存池暂存数据，它与Observable的异步缓存池一样，可以无限制向里添加数据，不会抛出MissingBackpressureException异常，但会导致OOM。
 运行如下代码：

```
public void demo6() {
        Flowable
                .create(new FlowableOnSubscribe<Integer>() {
                    @Override
                    public void subscribe(FlowableEmitter<Integer> e) throws Exception {
                        int i = 0;
                        while (true) {
                            i++;
                            e.onNext(i);
                        }
                    }
                }, BackpressureStrategy.BUFFER)
                .subscribeOn(Schedulers.newThread())
                .observeOn(Schedulers.newThread())
                .subscribe(new Consumer<Integer>() {
                    @Override
                    public void accept(@NonNull Integer integer) throws Exception {
                        Thread.sleep(5000);
                        System.out.println(integer);
                    }
                });
    }
```

查看内存使用:



![img](https:////upload-images.jianshu.io/upload_images/6773051-07576ae68d0728ea.gif?imageMogr2/auto-orient/strip%7CimageView2/2/w/582)

GIF222.gif



会发现和使用Observable时一样，都会导致内存剧增，最后导致OOM,不同的是使用Flowable内存增长的速度要慢得多，那是因为基于Flowable发射的数据流，以及对数据加工处理的各操作符都添加了背压支持，附加了额外的逻辑，其运行效率要比Observable低得多。

##### MISSING

对应于MissingEmitter<T>类，
 通过其源码：

```
static final class MissingEmitter<T> extends BaseEmitter<T> {


        private static final long serialVersionUID = 3776720187248809713L;

        MissingEmitter(Subscriber<? super T> actual) {
            super(actual);
        }

        @Override
        public void onNext(T t) {
            if (isCancelled()) {
                return;
            }

            if (t != null) {
                actual.onNext(t);
            } else {
                onError(new NullPointerException("onNext called with null. Null values are generally not allowed in 2.x operators and sources."));
                return;
            }

            for (;;) {
                long r = get();
                if (r == 0L || compareAndSet(r, r - 1)) {
                    return;
                }
            }
        }

    }
```

可以发现，在传递数据时

```
actual.onNext(t);
```

并没有对缓存池的状态进行判断，所以在此策略下，通过Create方法创建的Flowable相当于没有指定背压策略，不会对通过onNext发射的数据做缓存或丢弃处理，需要下游通过背压操作符（onBackpressureBuffer()/onBackpressureDrop()/onBackpressureLatest()）指定背压策略。

##### onBackpressureXXX背压操作符

Flowable除了通过create创建的时候指定背压策略，也可以在通过其它创建操作符just，fromArray等创建后通过背压操作符指定背压策略。
 onBackpressureBuffer()对应BackpressureStrategy.BUFFER
 onBackpressureDrop()对应BackpressureStrategy.DROP
 onBackpressureLatest()对应BackpressureStrategy.LATEST
 例如代码

```
    public void demo7() {
        Flowable
                .create(new FlowableOnSubscribe<Integer>() {
                    @Override
                    public void subscribe(FlowableEmitter<Integer> e) throws Exception {
                        for (int i = 0; i < 500; i++) {
                            e.onNext(i);
                        }
                        e.onComplete();
                    }
                }, BackpressureStrategy.DROP)
                .observeOn(Schedulers.newThread())
                .subscribeOn(Schedulers.newThread())
                .subscribe(new Consumer<Integer>() {
                    @Override
                    public void accept(@NonNull Integer integer) throws Exception {
                        System.out.println(integer);
                    }
                });
    }
```

等同于，代码：

```
    public void demo8() {
        Flowable.range(0, 500)
                .onBackpressureDrop()
                .subscribeOn(Schedulers.newThread())
                .observeOn(Schedulers.newThread())
                .subscribe(new Consumer<Integer>() {
                    @Override
                    public void accept(@NonNull Integer integer) throws Exception {
                        System.out.println(integer);
                    }
                });
    }
```

#### Subscription

Subscription与Disposable均是观察者与可观察对象建立订阅状态后回调回来的参数，如同通过Disposable的dispose()方法可以取消Observer与Oberverable的订阅关系一样，通过Subscription的cancel()方法也可以取消Subscriber与Flowable的订阅关系。
 不同的是接口Subscription中多了一个方法request(long n)，如上面代码中的：

```
 s.request(Long.MAX_VALUE);   
```

此方法的作用是什么呢，去掉这个方法会有什么影响呢？
 运行如下代码：

```
public void demo9() {
        Flowable
                .create(new FlowableOnSubscribe<Integer>() {
                    @Override
                    public void subscribe(FlowableEmitter<Integer> e) throws Exception {
                        System.out.println("发射----> 1");
                        e.onNext(1);
                        System.out.println("发射----> 2");
                        e.onNext(2);
                        System.out.println("发射----> 3");
                        e.onNext(3);
                        System.out.println("发射----> 完成");
                        e.onComplete();
                    }
                }, BackpressureStrategy.BUFFER) //create方法中多了一个BackpressureStrategy类型的参数
                .subscribeOn(Schedulers.newThread())//为上下游分别指定各自的线程
                .observeOn(Schedulers.newThread())
                .subscribe(new Subscriber<Integer>() {
                    @Override
                    public void onSubscribe(Subscription s) {
                        //去掉代码s.request(Long.MAX_VALUE);
                    }

                    @Override
                    public void onNext(Integer integer) {
                        System.out.println("接收----> " + integer);
                    }

                    @Override
                    public void onError(Throwable t) {
                    }

                    @Override
                    public void onComplete() {
                        System.out.println("接收----> 完成");
                    }
                });
    }
```

运行结果如下：

```
System.out: 发射----> 1
System.out: 发射----> 2
System.out: 发射----> 3
System.out: 发射----> 完成
```

我们发现Flowable照常发送数据，而Subsriber不再接收数据。
 这是因为Flowable在设计的时候，采用了一种新的思路——**响应式拉取**方式，来设置下游对数据的请求数量，上游可以根据下游的需求量，按需发送数据。
 如果不显示调用request则默认下游的需求量为零，所以运行上面的代码后，上游Flowable发射的数据不会交给下游Subscriber处理。
 运行如下代码：

```
public void demo10() {
        Flowable
                .create(new FlowableOnSubscribe<Integer>() {
                    @Override
                    public void subscribe(FlowableEmitter<Integer> e) throws Exception {
                        System.out.println("发射----> 1");
                        e.onNext(1);
                        System.out.println("发射----> 2");
                        e.onNext(2);
                        System.out.println("发射----> 3");
                        e.onNext(3);
                        System.out.println("发射----> 完成");
                        e.onComplete();
                    }
                }, BackpressureStrategy.BUFFER) //create方法中多了一个BackpressureStrategy类型的参数
                .subscribeOn(Schedulers.newThread())//为上下游分别指定各自的线程
                .observeOn(Schedulers.newThread())
                .subscribe(new Subscriber<Integer>() {
                    @Override
                    public void onSubscribe(Subscription s) {
                        s.request(2);//设置Subscriber的消费能力为2
                    }

                    @Override
                    public void onNext(Integer integer) {
                        System.out.println("接收----> " + integer);
                    }

                    @Override
                    public void onError(Throwable t) {
                    }

                    @Override
                    public void onComplete() {
                        System.out.println("接收----> 完成");
                    }
                });
    }
```

运行结果如下：

```
System.out: 发射----> 1
System.out: 发射----> 2
System.out: 发射----> 3
System.out: 发射----> 完成
System.out: 接收----> 1
System.out: 接收----> 2
```

我们发现通过s.request(2);设置Subscriber的数据请求量为2条，超出其请求范围之外的数据则没有接收。
 多次调用request会产生怎样的结果呢？
 运行如下代码：

```
public void demo11() {
        Flowable
                .create(new FlowableOnSubscribe<Integer>() {
                    @Override
                    public void subscribe(FlowableEmitter<Integer> e) throws Exception {
                        String threadName = Thread.currentThread().getName();
                        for (int i = 1; i <= 10; i++) {
                            System.out.println(threadName + "发射---->" + i);
                            e.onNext(i);
                        }
                        System.out.println(threadName + "发射数据结束" + System.currentTimeMillis());
                        e.onComplete();
                    }
                }, BackpressureStrategy.BUFFER)
                .subscribeOn(Schedulers.newThread())
                .observeOn(Schedulers.newThread())
                .subscribe(new Subscriber<Integer>() {
                    @Override
                    public void onSubscribe(Subscription s) {
                        s.request(3);//调用两次request
                        s.request(4);
                    }

                    @Override
                    public void onNext(Integer integer) {
                        System.out.println(Thread.currentThread().getName() + "接收---------->" + integer);
                    }

                    @Override
                    public void onError(Throwable t) {
                        t.printStackTrace();
                    }

                    @Override
                    public void onComplete() {
                        System.out.println(Thread.currentThread().getName() + "接收----> 完成");
                    }
                });
    }
```

通过Flowable发射10条数据，在onSubscribe(Subscription s) 方法中调用两次request，运行结果如下：



![img](https:////upload-images.jianshu.io/upload_images/6773051-9fd6ff08fa332115.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/405)

AB417C9CAC5A4BD98375240B5A5C1D6A.jpg

我们发现Subscriber总共接收了7条数据，是两次需求累加后的数量。

通过日志我们发现，上游并没有根据下游的实际需求，发送数据，而是能发送多少，就发送多少，不管下游是否需要。
 而且超出下游需求之外的数据，仍然放到了异步缓存池中。这点我们可以通过以下代码来验证：

```
public void demo12() {
        Flowable
                .create(new FlowableOnSubscribe<Integer>() {
                    @Override
                    public void subscribe(FlowableEmitter<Integer> e) throws Exception {
                        for (int i = 1; i < 130; i++) {
                            System.out.println("发射---->" + i);
                            e.onNext(i);
                        }
                        e.onComplete();
                    }
                }, BackpressureStrategy.ERROR)
                .subscribeOn(Schedulers.newThread())
                .observeOn(Schedulers.newThread())
                .subscribe(new Subscriber<Integer>() {
                    @Override
                    public void onSubscribe(Subscription s) {
                        s.request(1);
                    }

                    @Override
                    public void onNext(Integer integer) {
                        System.out.println("接收------>" + integer);
                    }

                    @Override
                    public void onError(Throwable t) {
                        t.printStackTrace();
                    }

                    @Override
                    public void onComplete() {
                        System.out.println("接收------>完成");
                    }
                });
    }
```

通过Flowable发射130条数据，通过s.request(1)设置下游的数据请求量为1条，设置缓存策略为BackpressureStrategy.ERROR，如果异步缓存池超限，会导致MissingBackpressureException异常。
 运行之后，日志如下：



![img](https:////upload-images.jianshu.io/upload_images/6773051-f4c46e603e4dd906.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/845)

MissingBackpressureException.jpg

久违的异常出现了，所以超出下游需求之外的数据，仍然放到了异步缓存池中，并导致缓存池溢出。

那么上游如何才能按照下游的请求数量发送数据呢，
 虽然通过request可以设置下游的请求数量，但是上游并没有获取到这个数量，如何获取呢？
 这便需要用到Flowable与Observable的第三点区别，Flowable特有的发射器FlowableEmitter

#### FlowableEmitter

flowable的发射器FlowableEmitter与observable的发射器ObservableEmitter均继承自Emitter（Emitter在教程二中已经说过了）
 比较两者源码可以发现；

```
public interface ObservableEmitter<T> extends Emitter<T> {

    void setDisposable(Disposable d);

    void setCancellable(Cancellable c);

    boolean isDisposed();
  
    ObservableEmitter<T> serialize();
}
```

与

```
public interface FlowableEmitter<T> extends Emitter<T> {

    void setDisposable(Disposable s);

    void setCancellable(Cancellable c);

    long requested();

    boolean isCancelled();

    FlowableEmitter<T> serialize();
}
```

接口FlowableEmitter中多了一个方法

```
long requested();
```

我们可以通过这个方法来获取当前未完成的请求数量，
 运行下面的代码，这次我们要先丧失一下原则，虽然我们之前说过同步状态下不使用Flowable，但是这次我们需要先看一下同步状态下情况。

```
public void demo13() {
        Flowable
                .create(new FlowableOnSubscribe<Integer>() {
                    @Override
                    public void subscribe(FlowableEmitter<Integer> e) throws Exception {
                        String threadName = Thread.currentThread().getName();
                        for (int i = 1; i <= 5; i++) {
                            System.out.println("当前未完成的请求数量-->" + e.requested());
                            System.out.println(threadName + "发射---->" + i);
                            e.onNext(i);
                        }
                        System.out.println(threadName + "发射数据结束" + System.currentTimeMillis());
                        e.onComplete();
                    }
                }, BackpressureStrategy.BUFFER)//上下游运行在同一线程中
                .subscribe(new Subscriber<Integer>() {
                    @Override
                    public void onSubscribe(Subscription s) {
                        s.request(3);
                    }

                    @Override
                    public void onNext(Integer integer) {
                        System.out.println(Thread.currentThread().getName() + "接收---------->" + integer);
                    }

                    @Override
                    public void onError(Throwable t) {
                        t.printStackTrace();
                    }

                    @Override
                    public void onComplete() {
                        System.out.println(Thread.currentThread().getName() + "接收----> 完成");
                    }
                });
    }
```

打印日志如下：



![img](https:////upload-images.jianshu.io/upload_images/6773051-587f288da9916ac5.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/308)

4.jpg

通过日志我们发现， 通过*e.requested()*获取到的是一个动态的值，会随着下游已经接收的数据的数量而递减。
 在上面的代码中，我们没有指定上下游的线程，上下游运行在同一线程中。
 这与我们之前提到的，同步状态下不使用Flowable相违背。那是因为异步情况下*e.requested()*的值太复杂，必须通过同步情况过渡一下才能说得明白。
 我们在上面代码的基础上，给上下游指定独立的线程，代码如下

```
public void demo14() {
        Flowable
                .create(new FlowableOnSubscribe<Integer>() {
                    @Override
                    public void subscribe(FlowableEmitter<Integer> e) throws Exception {
                        String threadName = Thread.currentThread().getName();
                        for (int i = 1; i <= 5; i++) {
                            System.out.println("当前未完成的请求数量-->" + e.requested());
                            System.out.println(threadName + "发射---->" + i);
                            e.onNext(i);
                        }
                        System.out.println(threadName + "发射数据结束" + System.currentTimeMillis());
                        e.onComplete();
                    }
                }, BackpressureStrategy.BUFFER)
                .subscribeOn(Schedulers.newThread())//添加两行代码，为上下游分配独立的线程
                .observeOn(Schedulers.newThread())
                .subscribe(new Subscriber<Integer>() {
                    @Override
                    public void onSubscribe(Subscription s) {
                        s.request(3);
                    }

                    @Override
                    public void onNext(Integer integer) {
                        System.out.println(Thread.currentThread().getName() + "接收---------->" + integer);
                    }

                    @Override
                    public void onError(Throwable t) {
                        t.printStackTrace();
                    }

                    @Override
                    public void onComplete() {
                        System.out.println(Thread.currentThread().getName() + "接收----> 完成");
                    }
                });
    }
```

运行后日志如下：



![img](https:////upload-images.jianshu.io/upload_images/6773051-bd03a5018b599393.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/386)

log5.jpg



虽然我们指定了下游的数据请求量为3，但是我们在上游获取未完成请求数量的时候，并不是3，而是128。难道上游有个最小未完成请求数量？只要下游设置的数据请求量小于128，上游获取到的都是128？
 带着这个疑问，我们试一下当下游的数据请求量为500，大于128时的情况。

```
public void demo15() {
        Flowable
                .create(new FlowableOnSubscribe<Integer>() {
                    @Override
                    public void subscribe(FlowableEmitter<Integer> e) throws Exception {
                        String threadName = Thread.currentThread().getName();
                        for (int i = 1; i <= 5; i++) {
                            System.out.println("当前未完成的请求数量-->" + e.requested());
                            System.out.println(threadName + "发射---->" + i);
                            e.onNext(i);
                        }
                        System.out.println(threadName + "发射数据结束" + System.currentTimeMillis());
                        e.onComplete();
                    }
                }, BackpressureStrategy.BUFFER)
                .subscribeOn(Schedulers.newThread())//添加两行代码，为上下游分配独立的线程
                .observeOn(Schedulers.newThread())
                .subscribe(new Subscriber<Integer>() {
                    @Override
                    public void onSubscribe(Subscription s) {
                        s.request(500);
                    }

                    @Override
                    public void onNext(Integer integer) {
                        System.out.println(Thread.currentThread().getName() + "接收---------->" + integer);
                    }

                    @Override
                    public void onError(Throwable t) {
                        t.printStackTrace();
                    }

                    @Override
                    public void onComplete() {
                        System.out.println(Thread.currentThread().getName() + "接收----> 完成");
                    }
                });
    }
```

运行日志如下;





![img](https:////upload-images.jianshu.io/upload_images/6773051-b7bff24f0a6e7b4e.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/414)

log6.jpg


 结果还是128. 其实不论下游通过设置多少请求量，我们在上游获取到的初始未完成请求数量都是128。 这是为啥呢？ 还记得之前我们说过，Flowable有一个异步缓存池，上游发射的数据，先放到异步缓存池中，再由异步缓存池交给下游。所以上游在发射数据时，首先需要考虑的不是下游的数据请求量，而是缓存池中能不能放得下，否则在缓存池满的情况下依然会导致数据遗失或者背压异常。如果缓存池可以放得下，那就发送，至于是否超出了下游的数据需求量，可以在缓存池向下游传递数据时，再作判断，如果未超出，则将缓存池中的数据传递给下游，如果超出了，则不传递。 如果下游对数据的需求量超过缓存池的大小，而上游能获取到的最大需求量是128，上游对超出128的需求量是怎么获取到的呢？ 带着这个疑问，我们运行一下，下面的代码，上游发送150个数据，下游也需要150个数据。

```
public void demo16() {
        Flowable
                .create(new FlowableOnSubscribe<Integer>() {
                    @Override
                    public void subscribe(FlowableEmitter<Integer> e) throws Exception {
                        String threadName = Thread.currentThread().getName();
                        for (int i = 1; i <= 150; i++) {
                            System.out.println("当前未完成的请求数量-->" + e.requested());
                            System.out.println(threadName + "发射---->" + i);
                            e.onNext(i);
                        }
                        System.out.println(threadName + "发射数据结束" + System.currentTimeMillis());
                        e.onComplete();
                    }
                }, BackpressureStrategy.BUFFER)
                .subscribeOn(Schedulers.newThread())
                .observeOn(Schedulers.newThread())
                .subscribe(new Subscriber<Integer>() {
                    @Override
                    public void onSubscribe(Subscription s) {
                        s.request(150);
                    }

                    @Override
                    public void onNext(Integer integer) {
                        System.out.println(Thread.currentThread().getName() + "接收---------->" + integer);
                    }

                    @Override
                    public void onError(Throwable t) {
                        t.printStackTrace();
                    }

                    @Override
                    public void onComplete() {
                        System.out.println(Thread.currentThread().getName() + "接收----> 完成");
                    }
                });
    }
```

截取部分日志如下：
 



![img](https:////upload-images.jianshu.io/upload_images/6773051-648e811834111d88.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/377)

log7.jpg


 我们发现通过获取到的上游当前未完成请求数量并不是一直递减的，在递减到33时，又回升到了128.而回升的时机正好是在下游接收了96条数据之后。我们之前说过，异步缓存池中的数据并不是向下游发射一条便清理一条，而是每等累积到95条时，清理一次。通过获取到的值，正是在异步缓存池清理数据时，回升的。也就是，异步缓存池每次清理后，有剩余的空间时，都会导致上游未完成请求数量的回升，这样既不会引发背压异常，也不会导致数据遗失。 上游在发送数据的时候并不需要考虑下游需不需要，而只需要考虑异步缓存池中是否放得下，放得下便发，放不下便暂停。所以，通过获取到的值，并不是下游真正的数据请求数量，而是异步缓存池中可放入数据的数量。数据放入缓存池中后，再由缓存池按照下游的数据请求量向下传递，待到传递完的数据累积到95条之后，将其清除，腾出空间存放新的数据。如果下游处理数据缓慢，则缓存池向下游传递数据的速度也相应变慢，进而没有传递完的数据可清除，也就没有足够的空间存放新的数据，上游通过获取的值也就变成了0，如果此时，再发送数据的话，则会根据BackpressureStrategy背压策略的不同，抛出MissingBackpressureException异常，或者丢掉这条数据。 所以上游只需要在e.requested()等于0时，暂停发射数据，便可解决背压问题。

#### 最终方案

下面我们回到最初的问题
 运行下面代码：

```
public void demo17() {
        Observable
                .create(new ObservableOnSubscribe<Integer>() {
                    @Override
                    public void subscribe(ObservableEmitter<Integer> e) throws Exception {
                        int i = 0;
                        while (true) {
                            i++;
                            System.out.println("发射---->" + i);
                            e.onNext(i);
                        }
                    }
                })
                .subscribeOn(Schedulers.newThread())
                .observeOn(Schedulers.newThread())
                .subscribe(new Observer<Integer>() {
                    @Override
                    public void onSubscribe(Disposable d) {
                    }

                    @Override
                    public void onNext(Integer integer) {
                        try {
                            Thread.sleep(50);
                            System.out.println("接收------>" + integer);
                        } catch (InterruptedException ignore) {
                        }
                    }

                    @Override
                    public void onError(Throwable e) {
                        e.printStackTrace();
                    }

                    @Override
                    public void onComplete() {
                        System.out.println("接收------>完成");
                    }
                });
    }
```

由于下游处理数据的速度（Thread.sleep(50)）赶不上上游发射数据的速度，则会导致背压问题。
 运行后查看内存使用如下：



![img](https:////upload-images.jianshu.io/upload_images/6773051-bf918584b8c86a86.gif?imageMogr2/auto-orient/strip%7CimageView2/2/w/562)

GIF333.gif

内存暴增，很快就会OOM
 下面，对其通过Flowable做些改进，让其既不会产生背压问题，也不会引起异常或者数据丢失。
 代码如下：

```
public void demo18() {
        Flowable
                .create(new FlowableOnSubscribe<Integer>() {
                    @Override
                    public void subscribe(FlowableEmitter<Integer> e) throws Exception {
                        int i = 0;
                        while (true) {
                            if (e.requested() == 0) continue;//此处添加代码，让flowable按需发送数据
                            System.out.println("发射---->" + i);
                            i++;
                            e.onNext(i);
                        }
                    }
                }, BackpressureStrategy.MISSING)
                .subscribeOn(Schedulers.newThread())
                .observeOn(Schedulers.newThread())
                .subscribe(new Subscriber<Integer>() {
                    private Subscription mSubscription;

                    @Override
                    public void onSubscribe(Subscription s) {
                        s.request(1);            //设置初始请求数据量为1
                        mSubscription = s;
                    }

                    @Override
                    public void onNext(Integer integer) {
                        try {
                            Thread.sleep(50);
                            System.out.println("接收------>" + integer);
                            mSubscription.request(1);//每接收到一条数据增加一条请求量
                        } catch (InterruptedException ignore) {
                        }
                    }

                    @Override
                    public void onError(Throwable t) {
                    }

                    @Override
                    public void onComplete() {
                    }
                });
    }
```

下游处理数据的速度*Thread.sleep(50)*赶不上上游发射数据的速度，
 不同的是，我们在下游*onNext(Integer integer)* 方法中，每接收一条数据增加一条请求量，

```
mSubscription.request(1)
```

在上游添加代码

```
if(e.requested()==0)continue;
```

让上游按需发送数据。
 运行后查看内存：



![img](https:////upload-images.jianshu.io/upload_images/6773051-afa30fdace5c0d2e.gif?imageMogr2/auto-orient/strip%7CimageView2/2/w/562)

GIF999.gif

内存一直相当的平静，而且上游严格按照下游的需求量发送数据，不会产生MissingBackpressureException异常，或者丢失数据