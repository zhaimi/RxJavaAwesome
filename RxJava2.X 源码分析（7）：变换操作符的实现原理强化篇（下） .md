# RxJava2.X 源码分析（七）：变换操作符的实现原理强化篇（下） 

### 一、前言

- Ok，我们在上一篇[RxJava2.X 源码分析（六）：变换操作符的实现原理（上）](http://www.catbro.cn/articles/2017/07/17/1500260864699.html)中分析了RxJava2中转换操作符`map`的实现过程
- 本次我们将紧跟上篇的步伐，分析比`map`更为强大的`flatMap`操作符的实现流程

### 二、源码分析

- 首先，我们还是老样子，先看一个demo

  ```
    Observable observable = Observable.create(new ObservableOnSubscribe() {
        @Override
      public void subscribe(@NonNull ObservableEmitter emitter) throws Exception {
            emitter.onNext(1);
            emitter.onNext(2);
            emitter.onNext(3);
  
        }
  
    });
  
    observable.flatMap(new Function>() {
        @Override
      public ObservableSource apply(@NonNull Integer integer) throws Exception {
  
            return Observable.just(integer*integer);
        }
    }).subscribe(new Consumer() {
        @Override
      public void accept(@NonNull Integer integer) throws Exception {
            Log.i(TAG, ">>>data is : " + integer);
        }
    });
  ```

- 输出结果：

  ```
    07-18 09:42:25.550 2502-2515/? I/RxJavaDemo2: >>>data is : 1
    07-18 09:42:25.550 2502-2515/? I/RxJavaDemo2: >>>data is : 4
    07-18 09:42:25.550 2502-2515/? I/RxJavaDemo2: >>>data is : 9
  ```

- 这结果不是上篇一样的么？也就是说实现了相同的功能，但是，重点是在`flatMap`的回调`apply`中我们返回的并不是直接的数据，而是`Observable`,那么这个区别能提供怎样的能力呢？
  1、比如连续两个串行的网络请求
  2、如数据类型的中间转换等等，

- 也就是说，我们能以很自然流畅的方式在中间做一些转换，能做的事情很多哦。

------

- OK，瞎逼逼了这么久，我们开始进入正题，看下内部是如何实现的呢？从flatMap方法进去。

  ```
    public final <R> Observable<R> flatMap(Functionsuper T, ? extends ObservableSourceextends R>> mapper) {
        return flatMap(mapper, false);
    }
  ```

- Ok，简单的wrapper了一层，第二个参数为是否延迟错误处理，默认false，我们继续

  ```
    public final <R> Observable<R> flatMap(Functionsuper T, ? extends ObservableSourceextends R>> mapper, boolean delayErrors) {
        return flatMap(mapper, delayErrors, Integer.MAX_VALUE);
    }
  ```

- (￣∇￣)又wrapper了一层。OK，我忍你，第三个参数为最大的并发量，OK，再进去

  ```
    public final <R> Observable<R> flatMap(Functionsuper T, ? extends ObservableSourceextends R>> mapper, boolean delayErrors, int maxConcurrency) {
        return flatMap(mapper, delayErrors, maxConcurrency, bufferSize());
    }
  ```

- 又来了一层，╮(╯_╰)╭，OK，你牛逼，第四个参数为队列的大小，OK，我们还是继续前进吧

  ```
    public final <R> Observable<R> flatMap(Functionsuper T, ? extends ObservableSourceextends R>> mapper,
            boolean delayErrors, int maxConcurrency, int bufferSize) {
            //1、做一些值的判断，不符合规定我就报一场
        ObjectHelper.requireNonNull(mapper, "mapper is null");
        ObjectHelper.verifyPositive(maxConcurrency, "maxConcurrency");
        ObjectHelper.verifyPositive(bufferSize, "bufferSize");
        //2、判断我们的上游Obsevable是否是ScalarCallable类型，做特殊处理，我们的上游如果是Observable.just就会走这里
        if (this instanceof ScalarCallable) {
            @SuppressWarnings("unchecked")
            T v = ((ScalarCallable<T>)this).call();
            if (v == null) {
                return empty();
            }
            return ObservableScalarXMap.scalarXMap(v, mapper);
        }
        //3、OK，看到我们熟悉的方法了
        return RxJavaPlugins.onAssembly(new ObservableFlatMap<T, R>(this, mapper, delayErrors, maxConcurrency, bufferSize));
  ```

- OK，我们看下`ObservableScalarXMap.scalarXMap(v, mapper);`

  ```
    public static <T, U> Observable<U> scalarXMap(T value,
            Functionsuper T, ? extends ObservableSourceextends U>> mapper) {
        return RxJavaPlugins.onAssembly(new ScalarXMapObservable<T, U>(value, mapper));
    }
  ```

- 我们看到了非常熟悉的方法`RxJavaPlugins.onAssembly`,OK,我们看下ScalarXMapObservable

  ```
    static final class ScalarXMapObservable<T, R> extends Observable<R> {
  
        final T value;
  
        final Functionsuper T, ? extends ObservableSourceextends R>> mapper;
  
        ScalarXMapObservable(T value,
                Functionsuper T, ? extends ObservableSourceextends R>> mapper) {
            //1、传入的值
            this.value = value;
            //2、转换函数
            this.mapper = mapper;
        }
  
        @SuppressWarnings("unchecked")
        @Override
      public void subscribeActual(Observer<? super R> s) {
            ObservableSource<? extends R> other;
            try {
                //3、获取转换后的Observable
                other = ObjectHelper.requireNonNull(mapper.apply(value), "The mapper returned a null ObservableSource");
            } catch (Throwable e) {
                EmptyDisposable.error(e, s);
                return;
            }
            ／／ScalarCallable实现Callable接口，走这里
            if (other instanceof Callable) {
                R u;
  
                try {
                    u = ((Callable<R>)other).call();
                } catch (Throwable ex) {
                    Exceptions.throwIfFatal(ex);
                    EmptyDisposable.error(ex, s);
                    return;
                }
  
                if (u == null) {
                    EmptyDisposable.complete(s);
                    return;
                }
                //4、对下游Observer Wrapper
                ScalarDisposable<R> sd = new ScalarDisposable<R>(s, u);
                //5、调用onSubscribe
                s.onSubscribe(sd);
                //6、熟悉的run，我们看ScalarDisposable的run
                sd.run();
            } else {
                other.subscribe(s);
            }
        }
    }
  ```

- 看构造函数

  ```
    public ScalarDisposable(Observersuper T> observer, T value) {
        //下游Observer
        this.observer = observer;
        //传递的值
        this.value = value;
    }
  ```

- 看run

  ```
    public void run() {
        if (get() == START && compareAndSet(START, ON_NEXT)) {
            //OK,数据传递给下游onNext
            observer.onNext(value);
            if (get() == ON_NEXT) {
                lazySet(ON_COMPLETE);
                observer.onComplete();
            }
        }
    }
  ```

- Observable.just是相对简单点，但是本次我们demo的Observable并不是通过just构造，所以走的就是下面这个了`return RxJavaPlugins.onAssembly(new ObservableFlatMap(this, mapper, delayErrors, maxConcurrency, bufferSize));`

- OK,那么我们来分析下`Observable.fromArray`这种形式的

------

- OK,代码有点长，我们分段来

public final class ObservableFlatMap<T, U> extends AbstractObservableWithUpstream<T, U> {}

- 可以看到，ObservableFlatMap的包装以之前的线程实现类型，同样是继承了AbstractObservableWithUpstream类，也就是说，内部有`source`存储我们上游的`Observable`

- 构造方法比较简单，主要就是赋值

  ```
    public ObservableFlatMap(ObservableSource<T> source,
            Functionsuper T, ? extends ObservableSourceextends U>> mapper,
            boolean delayErrors, int maxConcurrency, int bufferSize) {
        super(source);
        this.mapper = mapper;
        this.delayErrors = delayErrors;
        this.maxConcurrency = maxConcurrency;
        this.bufferSize = bufferSize;
    }
  ```

------

- Ok，我们看重点subscribeActual方法，其触发是在下游Observer调用subscribe订阅时

  ```
    @Override
    public void subscribeActual(Observer<? super U> t) {
        //1、同样是对Scalar的处理
        if (ObservableScalarXMap.tryScalarXMapSubscribe(source, t, mapper)) {
            return;
        }
        //2、重点在这里，t为下游的Observer，mapper为我们的funcation，T为接收类型，R为转换后类型
  
        source.subscribe(new MergeObserver<T, U>(t, mapper, delayErrors, maxConcurrency, bufferSize));
    }
  ```

- 看2处代码：`source`为`上游Observable`，也就是我们这里new了一个`中间Obsever`来订阅`上游Observable`，然后`中间Observer`接收到上游下发的数据，猜测还是调用来`mapper.apply`获得转换后的Obsevable类型，然后还是看代码吧，猜太多都对了就不想往下看下l。

  ```
    static final class MergeObserver<T, U> extends AtomicInteger implements Disposable, Observer<T> {
  
       ...
        MergeObserver(Observersuper U> actual, Functionsuper T, ? extends ObservableSourceextends U>> mapper,
                boolean delayErrors, int maxConcurrency, int bufferSize) {
            //1、下游的Observer，用于后面数据的传递
            this.actual = actual;
            //2、funcation,用于转换函数的操作
            this.mapper = mapper;
            this.delayErrors = delayErrors;
            //3、并发数，因为中间我们转换后的Obsevable可能接收一个值然后产生多个值，如调用Observable.fromArray(),
            this.maxConcurrency = maxConcurrency;
            this.bufferSize = bufferSize;
            //4、存储apply返回的Observable
            if (maxConcurrency != Integer.MAX_VALUE) {
                sources = new ArrayDeque<? extends U>>(maxConcurrency);
            }
            this.observers = new AtomicReference[]>(EMPTY);
        }
    }
  ```

- OK，看完了构造方法，我们看下`MergeObserver`的`onSubscribe`以及`onXXX()`,执行完`source.subscribe(new MergeObserver<T, U>(t, mapper, delayErrors, maxConcurrency, bufferSize));`后，我们的`onSubscribe`会被调用，然后就是`onXXX()`；

  ```
    public void onSubscribe(Disposable s) {
        if (DisposableHelper.validate(this.s, s)) {
            this.s = s;
            actual.onSubscribe(this);
        }
    }
  ```

- Ok，在onSubscribe中只是简单的调用actual.onSubscribe(this)进行事件的传递，同时存储Disposable进行Disposable事件的管理。

------

- OK，那么重点应该在onNext(T t)

  ```
    @Override
    public void onNext(T t) {
        // safeguard against misbehaving sources
      if (done) {
            return;
        }
        ObservableSource<? extends U> p;
        try {
            //1、调用mapper.apply获得转换函数的返回值Observable。
            p = ObjectHelper.requireNonNull(mapper.apply(t), "The mapper returned a null ObservableSource");
        } catch (Throwable e) {
        //2、发生异常时终止
            Exceptions.throwIfFatal(e);
            s.dispose();
            onError(e);
            return;
        }
      //3、如果你明确设置了maxConcurrency，需要入队列进行排队处理，因为要控制并发量
        if (maxConcurrency != Integer.MAX_VALUE) {
            synchronized (this) {
                if (wip == maxConcurrency) {
                    sources.offer(p);
                    return;
                }
                wip++;
            }
        }
      //4、内部订阅处理，因为返回的p是observable，所以内部肯定还需要通过下游的Observer订阅它，这样数据流才能继续下发。
        subscribeInner(p);
    }
  ```

- OK，那么我们继续分析subscribeInner，看内部如何订阅

  ```
    void subscribeInner(ObservableSourceextends U> p) {
        for (;;) {
            if (p instanceof Callable) {
                tryEmitScalar(((Callableextends U>)p));
  
                if (maxConcurrency != Integer.MAX_VALUE) {
                    synchronized (this) {
                        p = sources.poll();
                        if (p == null) {
                            wip--;
                            break;
                        }
                    }
                } else {
                    break;
                }
            } else {
            //1、this及为我们的MergeObserver对象，再次wrapper，
                InnerObserver<T, U> inner = new InnerObserver<T, U>(this, uniqueId++);
                if (addInner(inner)) {
                      //触发订阅，如此，我们下游的Observer就开始等着接收数据流了。                    p.subscribe(inner);
                }
                break;
            }
        }
    }
  ```

- 

- OK，基本上整个flatMap的转换流程也就分析完了，其余太细的点，我们就不继续深入，因为本次我们的目的就是了解其实现流程即可。在宏观上进行把控。

### 三、总结

- Ok，根据上面的分析，其实对于`flatMap`的操作过程我们已经很清楚了，其跟`map`基本一样，通过在中间使用装饰者模式插入一个中间的Observable和Observe，你可以想象为代理。
- `代理Observable`做的事就是接收`下游Obsever`的订阅事件，然后通过`代理Obsever`订阅`上游Observer`，然后在`上游Observer`下发数据給`代理Observer`时，通过先调用`mapper.apply`转换回调函数获得转换后的`Observable`，内部通过`下游Observer`再次订阅，然后转换后返回的`Observable`下发数据给`下游Obsever`。

- OK，通过从第一篇到本篇，我们不难发现，基本都是一样的模式，通过装饰者模式中间产生一个Observable和Observer，完成订阅事件的传递以及下发数据流的传递，进过我们中间产生的Observable和Observer时，根据具体需求做一些操作。
- 理解了这些，相信你再去看其他操作符的代码时，按这个思路去看，基本都是Ok的。
- 希望通过本篇的学习，大家对于RxJava2的内部实现的一些流程更为熟悉

