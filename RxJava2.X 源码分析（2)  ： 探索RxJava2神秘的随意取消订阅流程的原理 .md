- ### 一、前言

- 基于RxJava2.1.1
- 我们在前一篇# [RxJava2.0源码解析（一）](http://cherylgood.cn/articles/2017/07/10/1499671003613.html)初步分析了RxJava从创建到执行的流程。
- 本篇我们将探索RxJava2.x提供给我们的Disposable能力的来源。
- 要相信，任何神奇的功能，当你探索了其本质之后，收获都是巨大的。

### 二、从Demo到原理

```
//1、观察者创建一个Observer
Observer observer = new Observer() {
    @Override
  public void onSubscribe(@NonNull Disposable d) {
        Log.d(TAG, "onSubscribe");
        disposable = d;
    }

    @Override
  public void onNext(@NonNull String s) {
        Log.d(TAG, "onNext data is :" + s);
        if (s.equals("hello")) {
            //执行了hello之后终止
  disposable.dispose();
        }
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
        Log.i(TAG, "发送 hello");
        e.onNext("world");
        Log.i(TAG, "发送 world");
        e.onComplete();
        Log.i(TAG, "调用 onComplete");

    }

});
observable.subscribe(observer);
```

- (￣∇￣)猜猜会输出什么呢？

  ```
    onSubscribe
    onNext data is :hello
    发送 hello
    发送 world
    调用 onComplete
  ```

- 在发送玩hello之后，成功终止了后面的Reactive流。从结果我们还发现，后面的Reactive流被终止了，也就是订阅者或者观察者收不到后面的信息了，但是生产者或者说被订阅者、被观察者的代码还是会继续执行的。

- Ok，我们从哪开始入手呢？我们发现，在我们执行了  disposable.dispose();后，触发了该事件，我们看下   disposable.dispose();到底做了什么呢，很开心的，我们点进   disposable.dispose();的源码，╮(╯_╰)╭，好吧，只是接口

  ```
    public interface Disposable {
        /**
     * Dispose the resource, the operation should be idempotent. */  void dispose();
  
        /**
     * Returns true if this resource has been disposed. * @return true if this resource has been disposed
     */  boolean isDisposed();
    }
  ```

- 此时我们要回忆一下上一篇的一段代码

  ```
    public ObservableCreate(ObservableOnSubscribe<T> source) {
        this.source = source;
    }
  
    @Override
    protected void subscribeActual(Observersuper T> observer) {
        CreateEmitter<T> parent = new CreateEmitter<T>(observer);
        observer.onSubscribe(parent);
  
        try {
            source.subscribe(parent);
        } catch (Throwable ex) {
            Exceptions.throwIfFatal(ex);
            parent.onError(ex);
        }
    }
  ```

- 我们之前分析到在执行`source.subscribe(parent);`触发数据分发事件之前先执行了`observer.onSubscribe(parent);`这句代码，所传入的`parent`也就对应了我们的`Disposable`

- `parent`是`CreateEmitter`类型的，但是`CreateEmitter`是实现了`Disposable`接口的一个类。而`parent`又是我们的`observer`的一个包装后的对象。

- OK，分析到这里我们来整理下前面的环节，根据Demo来解释下：首先在执行下面代码之前

  ```
        e.onNext("hello");
        Log.i(TAG, "发送 hello");
        e.onNext("world");
        Log.i(TAG, "发送 world");
        e.onComplete();
        Log.i(TAG, "调用 onComplete");
  ```

- 先执行了`observer.onSubscribe(parent);，`我们在demo中也是通过传入的`parent`调用其`dispose`方法来终止`Reactive`流，而执行分发`hello`等数据的`e`也是我们的`parent`，也就是他们都是同一个对象。而执行`e.onNext("hello");`的`e`对象也是`observer`的一个包装后的`ObservableEmitter`类型的对象。

- 总结：Observer自己来控制了Reactive流状态。

------

- Ok，此时如果我说关键点应该在ObservableEmitter这个类上面，你觉得可能性有多少呢？(￣∇￣)

- 关键点就是`CreateEmitter<T> parent = new CreateEmitter<T>(observer);`这个包装的过程，我们来看下其源码

  ```java
    static final class CreateEmitter<T>
    extends AtomicReference
    implements ObservableEmitter<T>, Disposable {
  
        private static final long serialVersionUID = -3434801548987643227L;
  
        final Observersuper T> observer;
  
        CreateEmitter(Observersuper T> observer) {
            this.observer = observer;
        }
  
        @Override
      public void onNext(T t) {
            if (t == null) {
                onError(new NullPointerException("onNext called with null. Null values are generally not allowed in 2.x operators and sources."));
                return;
            }
            if (!isDisposed()) {
                observer.onNext(t);
            }
        }
  
        @Override
      public void onError(Throwable t) {
            if (!tryOnError(t)) {
                RxJavaPlugins.onError(t);
            }
        }
  
        @Override
      public boolean tryOnError(Throwable t) {
            if (t == null) {
                t = new NullPointerException("onError called with null. Null values are generally not allowed in 2.x operators and sources.");
            }
            if (!isDisposed()) {
                try {
                    observer.onError(t);
                } finally {
                    dispose();
                }
                return true;
            }
            return false;
        }
  
        @Override
      public void onComplete() {
            if (!isDisposed()) {
                try {
                    observer.onComplete();
                } finally {
                    dispose();
                }
            }
        }
  
        @Override
      public void setDisposable(Disposable d) {
            DisposableHelper.set(this, d);
        }
  
        @Override
      public void setCancellable(Cancellable c) {
            setDisposable(new CancellableDisposable(c));
        }
  
        @Override
      public ObservableEmitter<T> serialize() {
            return new SerializedEmitter<T>(this);
        }
  
        @Override
      public void dispose() {
            DisposableHelper.dispose(this);
        }
  
        @Override
      public boolean isDisposed() {
            return DisposableHelper.isDisposed(get());
        }
    }
  ```

- 因为其实现了`ObservableEmitter<T>, Disposable`接口类，所以需实现其方法。这里其实是使用了装饰者模式，其魅力所在一会就会看到了。

- 我们可以看到在`ObservableEmitter`内部通过`final Observersuper T> observer;`存储了我们的`observer`，这样有什么用呢？看Demo，我们在调用`e.onNext("hello");`时，调用的时`ObservableEmitter`对象的`onNext`方法，然后`ObservableEmitter`对象的`onNext`方法内部再通过`observer`调用`onNext`方法，但是从源码我们可以发现，其并不是简单的调用哦。

  ```
    @Override
    public void onNext(T t) {
        if (t == null) {
            onError(new NullPointerException("onNext called with null. Null values are generally not allowed in 2.x operators and sources."));
            return;
        }
        if (!isDisposed()) {
            observer.onNext(t);
        }
    }
  ```

- 1、先判断传入的数据是否为null

- 2、判断`isDisposed()`，如果`isDisposed()`返回false则不执行`onNext`。

- 

------

- isDisposed()什么时候会返回false呢？按照demo，也就是我们调用了`disposable.dispose();`后，`disposable`前面分析了就是`CreateEmitter<T> parent`,我们看`CreateEmitter.dispose()`

  ```
    @Override
    public void dispose() {
        DisposableHelper.dispose(this);
    }
  ```

- 里面调用了`DisposableHelper.dispose(this);`，我们看`isDisposed()`

  ```
    @Override
    public boolean isDisposed() {
        return DisposableHelper.isDisposed(get());
    }
  ```

  #### `RxJava`的`onComplete();`与`onError(t);`只有一个会被执行的秘密原来是它？

- 再看另外两个方法的调用

  ```
    @Override
    public void onError(Throwable t) {
        if (!tryOnError(t)) {
            RxJavaPlugins.onError(t);
        }
    }
  
    @Override
    public boolean tryOnError(Throwable t) {
        if (t == null) {
            t = new NullPointerException("onError called with null. Null values are generally not allowed in 2.x operators and sources.");
        }
        if (!isDisposed()) {
            try {
                observer.onError(t);
            } finally {
                dispose();
            }
            return true;
        }
        return false;
    }
  
    @Override
    public void onComplete() {
        if (!isDisposed()) {
            try {
                observer.onComplete();
            } finally {
                dispose();
            }
        }
    }
  ```

- 其内部也基本做了同样的操作，先判断`!isDisposed()`后再决定是否执行。

- 但是再这里还有一点哦，我们应该知道

  ```
  onComplete();
  ```

  和

  ```
  onError(t)
  ```

  只有一个会发生，其实现原理也是通过isDisposed这个方法哦，我们可以看到，不关是先执行

  ```
  onComplete();
  ```

  还是先执行

  ```
  onError(t)
  ```

  ，最终都会调用

  ```
  dispose();
  ```

  ，而调用了

  ```
  dispose();
  ```

  后，

  ```
  isDisposed()
  ```

  为false，也就不会再执行另外一个了。而且如果人为先调用

  ```
  onComplete
  ```

  再调用

  ```
  onError
  ```

  ，

  ```
  onComplete
  ```

  不会被触发，而且会抛出

  ```
  NullPointerException
  ```

  异常。

  #### 小结：

- 此时我们的目的基本达到了，我们知道了`Reactive`流是如何被终止的以及`RxJava`的`onComplete();`与`onError(t);`只有一个会被执行的原因。

------

- 我们虽然知道了原因，但是秉着刨根问底的态度，抵挡不住内心的好奇，我还是决定挖一挖`DisposableHelper`这个类，当然如果不想了解`DisposableHelper`的话，看到这里也就可以了；

- Ok，前面分析到，代码里调用了`DisposableHelper`类的静态方法，我们看下其调用的两个静态方法分别做了什么？

  ```
    public enum DisposableHelper implements Disposable {
     DISPOSED;
  
     public static boolean isDisposed(Disposable d) {
       // 判断上次记录的终点标识的是否是 当前执行的Observer，如果是返回true
        return d == DISPOSED;
    }
    ....
    public static boolean dispose(AtomicReference field) {
        //1、current为我们当前的observer的Disposable的值，第一次调用时current是null
        Disposable current = field.get();
        //2、终止标识
        Disposable d = DISPOSED;
        //3、两次不相同，说明observer未调用过dispose，
        if (current != d) {
            //4、将终止标识的值设置給当前的observer的Disposable，并返回设置前的observer的Disposable的值，此时如果调用isDisposed(Disposable d)返回的就是ture了
            current = field.getAndSet(d);
            if (current != d) {
                //第一次调用时会走到这里，此时current==null，返回true，
                //current不为null时说明当前的observer调用了多次dispose(),而如果多次调用了Disposable的值还不是DISPOSED，说明之前设置失败，所以再次调用dispose();
                if (current != null) {
                    current.dispose();
                }
                return true;
            }
        }
        return false;
    }
    ....
    }    
  ```

- 1、`DISPOSED`：作为是否要终止的枚举类型的标识

- 2、`isDisposed`:判断上次记录的终点标识的是否是 当前执行的`Observer`，如果是返回`true`

- 3、`dispose`:采用了原子性引用类`AtomicReference`,目的是防止多线程操作出现的错误。

- 更详细的分析放入了代码中

### 总结：

- 通过本次，1、我们了解了RxJava的随意终止Reactive流的能力的来源；2、过程中也明白了`RxJava`的`onComplete();`与`onError(t);`只有一个会被执行的秘密。
- 实现该能力的主要方式还是利用了装饰者模式
- 从中体会了设计模式的魅力所在，当然我们还接触了AtomicReference这个类，在平时估计很少接触到。


