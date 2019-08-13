# RxJava2.X 源码分析（四）：探索RxJava2之观察者线程切换原理

### 一、前言

- 基于RxJava2.1.1

- 我们在前面的 [RxJava2.0使用详解（一）](http://cherylgood.cn/articles/2017/07/10/1499671003613.html)初步分析了RxJava从创建到执行的流程。[RxJava2.0使用详解（二) ](http://cherylgood.cn/articles/2017/07/11/1499770780242.html)中分析了RxJava的随意终止Reactive流的能力的来源；也明白了`RxJava`的`onComplete();`与`onError(t);`只有一个会被执行的秘密。[RxJava2.X 源码分析（三）](http://cherylgood.cn/articles/2017/07/13/1499925392920.html)中探索了RxJava2调用subscribeOn切换被观察者线程的原理。

- 本次我们将继续探索

  ```
  RxJava2.x
  ```

  切换观察者的原理，分析

  ```
  observeOn
  ```

  与

  ```
  subscribeOn
  ```

  的不同之处。继续实现我们在第一篇中定下的小目标

  ### 二、从Demo到原理

- OK，我们的Demo还是上次的demo，忘记了的小伙伴可以点击[RxJava2.X 源码分析（三）](http://cherylgood.cn/articles/2017/07/13/1499925392920.html)，这里就不再重复了哦，我们直接进入正题。

- Ok，按照套路，我们从`observeOn`方法入手。

- Ok，我点～^_^

  ```
  @CheckReturnValue
  @SchedulerSupport(SchedulerSupport.CUSTOM)
  public final Observable<T> observeOn(Scheduler scheduler) {
      //false为默认无延迟发送错误，bufferSize为缓冲区大小
      return observeOn(scheduler, false, bufferSize());
  }
  ```

- 我们继续往下看，我猜套路跟`subscribeOn`的逃不多，也是采用装饰者模式，wrapper我们的`Observable`和`Observer`产生一个中间被观察者和观察中，通过中间被观察者订阅上游被观察者，通过中间观察者接收上游被观察者下发的数据，然后通过线程切换将数据传递给下游观察者。

- Ok，我们来验证下才想。我觉得就是没完全猜对，也能猜对其中的大部分。

  ```
  @CheckReturnValue
  @SchedulerSupport(SchedulerSupport.CUSTOM)
  public final Observable<T> observeOn(Scheduler scheduler, boolean delayError, int bufferSize) {
      ObjectHelper.requireNonNull(scheduler, "scheduler is null");
      ObjectHelper.verifyPositive(bufferSize, "bufferSize");
      return RxJavaPlugins.onAssembly(new ObservableObserveOn<T>(this, scheduler, delayError, bufferSize));
  }
  ```

- Ok，熟悉的`RxJavaPlugins.onAssembly`hook处理，略过，直接看`new ObservableObserveOn(this, scheduler, delayError, bufferSize)`这句

  ```
    public final class ObservableObserveOn<T> extends AbstractObservableWithUpstream<T, T> {
        final Scheduler scheduler;
        final boolean delayError;
        final int bufferSize;
        public ObservableObserveOn(ObservableSource<T> source, Scheduler scheduler, boolean delayError, int bufferSize) {
            super(source);
            this.scheduler = scheduler;
            this.delayError = delayError;
            this.bufferSize = bufferSize;
        }
  
        @Override
      protected void subscribeActual(Observersuper T> observer) {
           //1、在当前线程调度，但不是立即执行，放入队列中
            if (scheduler instanceof TrampolineScheduler) {
                source.subscribe(observer);
            } else {
             //2、本次走的是这里
                Scheduler.Worker w = scheduler.createWorker();
              //3
                source.subscribe(new ObserveOnObserver<T>(observer, w, delayError, bufferSize));
            }
        }
  ```

- Ok,果然，熟悉的模式，对我们上游的`Observable`,下游的`Observer`wrapper一次。
  1、`ObservableObserveOn`继承了`AbstractObservableWithUpstream`
  2、`source`保存上游的`Observable`
  3、`scheduler`为本次的调度器
  4、在下游调用`subscribe`订阅时触发->`subscribeActual`->Wrapper了下游的`Observer`观察者

- 3处：source为游Observable，下游Observer被wrapper到ObserveOnObserver，发生订阅数件，上游Observable开始执行subscribeActual，调用ObserveOnObserver的onSubscribe以及onNext、onError、onComplete等

------

- OK，我们接着看Observer被包装进 ObserveOnObserver的样子，代码有点多，我们分段讲解

```
static final class ObserveOnObserver<T> extends BasicIntQueueDisposable<T>
    implements Observer<T>, Runnable {

        private static final long serialVersionUID = 6576896619930983584L;
        //下游的Observer
        final Observersuper T> actual;
        //调度工作者
        final Scheduler.Worker worker;
        //是否延迟错误，默认false
        final boolean delayError;
        //队列大小
        final int bufferSize;
        //存储上游Observable下发的数据队列
        SimpleQueue<T> queue;
        //存储下游Observer的Disposable
        Disposable s;
        //存储错误信息
        Throwable error;
        //校验是否完毕
        volatile boolean done;
        //是否被取消
        volatile boolean cancelled;
        //存储执行模式，同步或者异步 同步
        int sourceMode;

        boolean outputFused;

        ObserveOnObserver(Observersuper T> actual, Scheduler.Worker worker, boolean delayError, int bufferSize) {
            this.actual = actual;
            this.worker = worker;
            this.delayError = delayError;
            this.bufferSize = bufferSize;
        }

        @Override
      public void onSubscribe(Disposable s) {

            if (DisposableHelper.validate(this.s, s)) {
                this.s = s;
                if (s instanceof QueueDisposable) {
                    @SuppressWarnings("unchecked")
                    QueueDisposable<T> qd = (QueueDisposable<T>) s;

                    int m = qd.requestFusion(QueueDisposable.ANY | QueueDisposable.BOUNDARY);

                  //1、判断执行模式并调用onSubscribe传递给下游Observer
                    if (m == QueueDisposable.SYNC) {
                        sourceMode = m;
                        queue = qd;
                        //true 后面的onXX方法都不会被调用
                        done = true;
                        actual.onSubscribe(this);
                        //2、同步模式下，直接调用schedule
                        schedule();
                        return;
                    }
                    if (m == QueueDisposable.ASYNC) {
                        sourceMode = m;
                        queue = qd;
                        actual.onSubscribe(this);
                        //2、异步模式下，等待schedule
                        return;
                    }
                }

                queue = new SpscLinkedArrayQueue<T>(bufferSize);
                //判断执行模式并调用onSubscribe传递给下游Observer
                actual.onSubscribe(this);
            }
        }
```

- OK，执行玩这里之后，就到我们的onXX方法了

- 首先可无限调用的

  ```
  onNext
  ```

  ```
  @Override
  public void onNext(T t) {
     //3、数据源是同步模式或者执行过error / complete 会是true
    if (done) {
        return;
    }
    //如果数据源不是异步类型，
    if (sourceMode != QueueDisposable.ASYNC) {
        //4、上游Observable下发的数据压入queue
        queue.offer(t);
    }
    //5、开始调度
    schedule();
  }
  ```

- 其次只能触发一次的onError，基本差不多

  ```
  @Override
    public void onError(Throwable t) {
        if (done) {
            //6、已完成再执行会抛一场
            RxJavaPlugins.onError(t);
            return;
        }
        //7、记录错误信息
        error = t;
        //8、标识已完成
        done = true;
        //9、开始调度
        schedule();
    }
  ```

- 同样是只能触发一次的onComplete，同样的套路，就不说了

  ```
  @Override
    public void onComplete() {
        if (done) {
            return;
        }
        done = true;
        schedule();
    }
  ```

- 然后就是我们的关键点`schedule();`

```
 //关键点就是直接、简单、里面线程调度工作者调用schedule(this)，传入了this
    void schedule() {
           //getAndIncrement很关键，他原子性的保证了worker.schedule(this);在调度完之前不会被再次调度
        if (getAndIncrement() == 0) {
            worker.schedule(this);
        }
    }
```

- 什么？传入了this？那么说明什么呢？(￣∇￣)

- 嗯？this是个`runnable`，没错，我们的`ObserveOnObserver`实现了`Runnable`接口

- 那么，接下来自然是调用

  ```
  run
  ```

  方法

  ```
  @Override
  public void run() {
        //outputFused一般是false
      if (outputFused) {
          drainFused();
      } else {
          drainNormal();
      }
  ```

- 好吧，在看drainNormal前，我们先看一个函数

  ```
  //从名字看是检测是否已终止
    boolean checkTerminated(boolean d, boolean empty, Observersuper T> a) {
        //1、订阅已取消
        if (cancelled) {
            //清空队列
            queue.clear();
            return true;
        }
        //2、d其实是done，
        if (d) {
            //done==ture可能的情况onNext刚被调度完，onError或者onCompele被调用，
            Throwable e = error;
            if (delayError) {
                //delayError==true时等到队列为空才调用
                if (empty) {
                    if (e != null) {
                        a.onError(e);
                    } else {
                        a.onComplete();
                    }
                    worker.dispose();
                    return true;
                }
            } else {
                //否则直接调用
                if (e != null) {
                    queue.clear();
                    a.onError(e);
                    worker.dispose();
                    return true;
                } else
     if (empty) {
                    a.onComplete();
                    worker.dispose();
                    return true;
                }
            }
        }
        //否则未终结
        return false;
    }
  ```

- true：1、订阅被取消cancelled==true，2、done==true onNext刚被调度完，onError或者onCompele被调用
- 继续看drainNormal

```
void drainNormal() {
      int missed = 1;
      final SimpleQueue<T> q = queue;
      final Observersuper T> a = actual;
      //Ok,死循环，我们来看下有哪些出口
      for (;;) {
      //Ok，出口，该方法前面分析的
      if (checkTerminated(done, q.isEmpty(), a)) {
              return;
          }

          //在此死循环
          for (;;) {
              boolean d = done;
              T v;
              try {
                  //分发数据出队列
                  v = q.poll();
              } catch (Throwable ex) {
                  //有异常时终止退出
                  Exceptions.throwIfFatal(ex);
                  s.dispose();
                  q.clear();
                  a.onError(ex);
                  //停止worker（线程）
                  worker.dispose();
                  return;
              }
              boolean empty = v == null;
              //判断队列是否为空
              if (checkTerminated(d, empty, a)) {
                  return;
              }
               //没数据退出
              if (empty) {
                  break;
              }
              //数据下发给下游Obsever，这里支付者onNext，onComplete和onError主要放在了checkTerminated里面回调
              a.onNext(v);
          }
       //保证此时确实有一个 worker.schedule(this);正在被执行，
          missed = addAndGet(-missed);
       //为何要这样做呢？我的理解是保证drainNormal方法被原子性调用，如果执行了addAndGet之后getAndIncrement() == 0就成立了，此时又一个worker.schedule(this);被调用了，那么就不能执行break了
          if (missed == 0) {
              break;
          }
      }
  }
```

### 总结

- Ok，看到这里我们基本了解了observeOn的实现流程，同样是老套路，使用装饰者模式，中间Wrapper了我们的Observable和Observer，通过中间增加一个Observable和Observer来实现线程的切换。

