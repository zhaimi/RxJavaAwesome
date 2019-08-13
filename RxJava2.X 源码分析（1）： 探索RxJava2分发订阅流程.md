- ### 一、前言

- RxJava是什么呢？根据`RxJava`在`GitHub`上给出的描述
  RxJava  – Reactive Extensions for the JVM – a library for composing  asynchronous and event-based programs using observable sequences for the  Java
  大致意思是：一个可以在JVM上使用的，是由异步的基于事件编写的通过使用可观察序列构成的一个库。
  关键词：`异步`，`基于事件`，`可观察序列`
- 之前只是了解了Rx1.x时候的源码和使用方式，由于当时成员技术栈不统一，就没有在产品中使用。现在随着Rx的持续发热，身为主程的我依然留着对rx的喜爱，故现决定引入rx。
- 虽然有过使用rx的经历，但是现在rx升级到了2.0的版本，变化幅度还是蛮大的，所以抱着从0开始的心态，从新学习Rx2.X的相关代码及使用注意事项。
- 本次学习历程所定目标如下：
  1.初步了解RxJava2.X的使用流程
  2.探索`Observable`发送数据的流程
  3.明白`Observer`是如何接收数据的
  4.解析`Observable`与`Observer`的勾搭（如何关联）过程
  5.探索RxJava线程切换的奥秘
  6.了解RxJava操作符的实现原理
- 本次学习基于RxJava2.1.1版本的源码

### 二、从Demo到原理

```java
  //1、观察者创建一个Observer
  Observer observer = new Observer() {
      @Override
    public void onSubscribe(@NonNull Disposable d) {
          Log.d(TAG, "onSubscribe");
      }

      @Override
    public void onNext(@NonNull String s) {
          Log.d(TAG, "onNext data is :" + s);

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
          e.onNext("hello");
          e.onNext("world");
          e.onComplete();

      }
  });
  observable.subscribe(observer);
```

- 结果输出：

  ```java
    onSubscribe
    onNext data is :hello
    onNext data is :world
    onComplete
  ```

- 可以看到，Observer的onSubscribe是最先被调用的，这个回调会有什么用呢？我们后面会讲到。

------

- OK，从哪开始入手呢？`Observable.create`，嗯，整个流程是从create开始的，那么我们就从源头开始吧。先看一下create，他返回的是一个observable对象，也就是被观察的对象。create方法需要传入一个ObservableOnSubscribe来创建，我们看下ObservableOnSubscribe是什么

  ```java
    public interface ObservableOnSubscribe<T> {
        /**
     * Called for each Observer that subscribes. * @param e the safe emitter instance, never null
     * @throws Exception on error
     */  void subscribe(@NonNull ObservableEmitter<T> e) throws Exception;
    }
  ```

- 该接口会接收一个ObservableEmitter的一个对象，然后通过该对象我们可以发送消息也可以安全地取消消息，我们继续看ObservableEmitter这个接口类

  ```java
    public interface ObservableEmitter<T> extends Emitter<T> {
  
          void setDisposable(@Nullable Disposable d);
  
          void setCancellable(@Nullable Cancellable c);
  
          boolean isDisposed();
  
          @NonNull
          ObservableEmitter<T> serialize();
  
           @Experimental
          boolean tryOnError(@NonNull Throwable t);
    }
  ```

- ObservableEmitter是对Emitter的扩展，而扩展的方法证实RxJava2.0之后引入的，提供了可中途取消等新能力，我们继续看Emitter

  ```java
    public interface Emitter<T> {
  
           void onNext(@NonNull T value);
  
          void onError(@NonNull Throwable error);
  
          void onComplete();
    }
  ```

- 里面的三个方法使用过rx的应该非常眼熟了。看到这里，我们只是了解了传递参数的数据结构，了解到的信息还是比较少的。我们继续看下create内部做了什么操作呢？

  ```java
    public static <T> Observable<T> create(ObservableOnSubscribe<T> source) {
        ObjectHelper.requireNonNull(source, "source is null");
        return RxJavaPlugins.onAssembly(new ObservableCreate<T>(source));
    }
  ```

- RxJavaPlugins或许你会很陌生，其实我也很陌生，不过没关系，我觉得后面会经常遇到RxJavaPlugins，熟悉它是必然的；

- 可以看到我们传入ObservableOnSubscribe被用来创建ObservableCreate，其实ObservableCreate就是Observable的一个实现类哦。

  #### 思路梳理

- OK，到这里我们先梳理一下思路：
  1、Observable通过调用create创建一个Observable
  2、调用create时需要传入一个ObservableOnSubscribe类型的实例参数
  3、最终传入的ObservableOnSubscribe类型的实例参数作为ObservableCreate构造函数的参数传入，一个Observable就此诞生了

------

-ObservableCreate又是个什么东东呢？我们分步来，先看ObservableCreate的两个方法

```java
  public final class ObservableCreate<T> extends Observable<T> {
      final ObservableOnSubscribe<T> source;

      public ObservableCreate(ObservableOnSubscribe<T> source) {
          this.source = source;
      }

      @Override
    protected void subscribeActual(Observer<?super T> observer) {
          CreateEmitter<T> parent = new CreateEmitter<T>(observer);
          observer.onSubscribe(parent);

          try {
              source.subscribe(parent);
          } catch (Throwable ex) {
              Exceptions.throwIfFatal(ex);
              parent.onError(ex);
          }
      }
  .....
  }
```

- source：Observable.createc传入的ObservableOnSubscribe实例

- subscribeActual回调方法，它在调用Observable.subscribe时被调用，即与观察者或则订阅者发生联系时触发。subscribeActual也是实现我们主要逻辑的地方，我们来仔细分析下subscribeActual方法：
  1、首先subscribeActual传入的参数为Observer类型，也就是我们subscribe时传入的观察者，到底是不是呢？后面会分析到。
  2、传入的Observer会被包装成一个CreateEmitter，`CreateEmitter`继承了AtomicReference提供了原子级的控制能力。RxJava2.0提供的新特性与之息息相关哦，这个我们先给它来个关键标签，后面再详细分析。
  3、 观察者(observer)调用自己的onSubscribe(parent);将包装后的observer传入。这个也是RxJava2.0的变化，真正的订阅在`source.subscribe(parent);`这句代码被执行后开始，而在此之前先调用了onSubscribe方法来提供RxJava2.0后引入的新能力（如中断能力）。从这里我们也就知道了为何观察者的onSubscribe最先被调用了。（被订阅者说：我也很无辜，他自己调用了自己，我也控制不了╮(╯_╰)╭）
  4、被订阅者或者说被观察者（source）调用subscribe订阅方法与观察者发生联系。这里进行了异常捕获，如果subscribe抛出了未被捕获的异常，则调用 parent.onError(ex);
  5、在执行subscribe时也就对应了我们demo中的

  ```java
    public void subscribe(@NonNull ObservableEmitter e) throws Exception {
        e.onNext("hello");
        e.onNext("world");
        e.onComplete();
  
    }
  ```

- Ok，看来subscribeActual这个回调确实很重要，前面我们也说了subscribeActual回调方法在Observable.subscribe被调用时执行的，真的像我说的一样么？万一我看走眼了

  ```java
    @SchedulerSupport(SchedulerSupport.NONE)
    @Override
    public final void subscribe(Observersuper T> observer) {
        ObjectHelper.requireNonNull(observer, "observer is null");
        try {
            observer = RxJavaPlugins.onSubscribe(this, observer);
  
            ObjectHelper.requireNonNull(observer, "Plugin returned null Observer");
  
            subscribeActual(observer);
        } catch (NullPointerException e) { // NOPMD
      throw e;
        } catch (Throwable e) {
            Exceptions.throwIfFatal(e);
            // can't call onError because no way to know if a Disposable has been set or not
     // can't call onSubscribe because the call might have set a Subscription already  RxJavaPlugins.onError(e);
  
            NullPointerException npe = new NullPointerException("Actually not, but can't throw other exceptions due to RS");
            npe.initCause(e);
            throw npe;
        }
    }
  ```

- OK，代码不多，可以看到RxJavaPlugins.onSubscribe(this, observer);，我们RxJava2.0中的Hook能力就是来自这里了。然后继续看下面subscribeActual(observer);被调用了。

  #### 思路梳理

  1、传入的ObservableOnSubscribe最终被用来创建成ObservableCreate

  2、ObservableCreate持有我们的被观察者对象以及订阅时所触发的回调subscribeActual

  3、在subscribeActual实现了我们的主要逻辑，包括

  ```
  observer.onSubscribe(parent);
  ```

  ,

  ```
  source.subscribe(parent);
  ```

  ,

  ```
  parent.onError(ex);
  ```

  的调用

  4、在Observable的subscribe被调用时开始执行事件分发流程

