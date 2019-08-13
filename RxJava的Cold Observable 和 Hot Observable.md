Hot Observable 无论有没有 Subscriber 订阅，事件始终都会发生。当 Hot Observable 有多个订阅者时，Hot Observable 与订阅者们的关系是一对多的关系，可以与多个订阅者共享信息。

然而，Cold Observable 只有 Subscriber 订阅时，才开始执行发射数据流的代码。并且 Cold Observable 和 Subscriber 只能是一对一的关系，当有多个不同的订阅者时，消息是重新完整发送的。也就是说对 Cold Observable 而言，有多个Subscriber的时候，他们各自的事件是独立的。

如果上面的解释有点枯燥的话，那么下面会更加形象地说明 Cold 和 Hot 的区别：

> Think of a hot Observable as a radio station. All of the listeners that are listening to it at this moment listen to the same song.
>  A cold Observable is a music CD. Many people can buy it and listen to it independently.
>  by Nickolay Tsvetinov

#### Cold Observable

Observable 的 just、creat、range、fromXXX 等操作符都能生成Cold Observable。

```java
        Consumer<Long> subscriber1 = new Consumer<Long>() {
            @Override
            public void accept(@NonNull Long aLong) throws Exception {
                System.out.println("subscriber1: "+aLong);
            }
        };

        Consumer<Long> subscriber2 = new Consumer<Long>() {
            @Override
            public void accept(@NonNull Long aLong) throws Exception {
                System.out.println("   subscriber2: "+aLong);
            }
        };

        Observable<Long> observable = Observable.create(new ObservableOnSubscribe<Long>() {
            @Override
            public void subscribe(@NonNull ObservableEmitter<Long> e) throws Exception {
                Observable.interval(10, TimeUnit.MILLISECONDS,Schedulers.computation())
                        .take(Integer.MAX_VALUE)
                        .subscribe(e::onNext);
            }
        }).observeOn(Schedulers.newThread());

        observable.subscribe(subscriber1);
        observable.subscribe(subscriber2);

        try {
            Thread.sleep(100L);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
```

执行结果：

```
subscriber1: 0
   subscriber2: 0
subscriber1: 1
   subscriber2: 1
subscriber1: 2
   subscriber2: 2
   subscriber2: 3
subscriber1: 3
subscriber1: 4
   subscriber2: 4
   subscriber2: 5
subscriber1: 5
subscriber1: 6
   subscriber2: 6
subscriber1: 7
   subscriber2: 7
subscriber1: 8
   subscriber2: 8
subscriber1: 9
   subscriber2: 9
```

可以看出，subscriber1 和 subscriber2 的结果并不一定是相同的，二者是完全独立的。

尽管 Cold Observable 很好，但是对于某些事件不确定何时发生以及不确定 Observable 发射的元素数量，那还得使用 Hot Observable。比如：UI交互的事件、网络环境的变化、地理位置的变化、服务器推送消息的到达等等。

#### Cold Observable 如何转换成 Hot Observable？

###### 1. 使用publish，生成 ConnectableObservable

使用 publish 操作符，可以让 Cold Observable 转换成 Hot Observable。它将原先的 Observable 转换成 ConnectableObservable。

来看看刚才的例子：

```
        Consumer<Long> subscriber1 = new Consumer<Long>() {
            @Override
            public void accept(@NonNull Long aLong) throws Exception {
                System.out.println("subscriber1: "+aLong);
            }
        };

        Consumer<Long> subscriber2 = new Consumer<Long>() {
            @Override
            public void accept(@NonNull Long aLong) throws Exception {
                System.out.println("   subscriber2: "+aLong);
            }
        };

        Consumer<Long> subscriber3 = new Consumer<Long>() {
            @Override
            public void accept(@NonNull Long aLong) throws Exception {
                System.out.println("      subscriber3: "+aLong);
            }
        };

        ConnectableObservable<Long> observable = Observable.create(new ObservableOnSubscribe<Long>() {
            @Override
            public void subscribe(@NonNull ObservableEmitter<Long> e) throws Exception {
                Observable.interval(10, TimeUnit.MILLISECONDS,Schedulers.computation())
                        .take(Integer.MAX_VALUE)
                        .subscribe(e::onNext);
            }
        }).observeOn(Schedulers.newThread()).publish();
        observable.connect();

        observable.subscribe(subscriber1);
        observable.subscribe(subscriber2);

        try {
            Thread.sleep(20L);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        observable.subscribe(subscriber3);

        try {
            Thread.sleep(100L);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
```

注意，生成的 ConnectableObservable 需要调用connect()才能真正执行。

执行结果：

```
subscriber1: 0
   subscriber2: 0
subscriber1: 1
   subscriber2: 1
subscriber1: 2
   subscriber2: 2
      subscriber3: 2
subscriber1: 3
   subscriber2: 3
      subscriber3: 3
subscriber1: 4
   subscriber2: 4
      subscriber3: 4
subscriber1: 5
   subscriber2: 5
      subscriber3: 5
subscriber1: 6
   subscriber2: 6
      subscriber3: 6
subscriber1: 7
   subscriber2: 7
      subscriber3: 7
subscriber1: 8
   subscriber2: 8
      subscriber3: 8
subscriber1: 9
   subscriber2: 9
      subscriber3: 9
subscriber1: 10
   subscriber2: 10
      subscriber3: 10
subscriber1: 11
   subscriber2: 11
      subscriber3: 11
```

可以看到，多个订阅的 Subscriber 共享同一事件。
 在这里，ConnectableObservable 是线程安全的。

##### 2. 使用Subject/Processor

Subject 和 Processor 的作用是相同的。Processor 是 RxJava2.x 新增的类，继承自 Flowable 支持背压控制。而 Subject 则不支持背压控制。

```java
        Consumer<Long> subscriber1 = new Consumer<Long>() {
            @Override
            public void accept(@NonNull Long aLong) throws Exception {
                System.out.println("subscriber1: "+aLong);
            }
        };

        Consumer<Long> subscriber2 = new Consumer<Long>() {
            @Override
            public void accept(@NonNull Long aLong) throws Exception {
                System.out.println("   subscriber2: "+aLong);
            }
        };

        Consumer<Long> subscriber3 = new Consumer<Long>() {
            @Override
            public void accept(@NonNull Long aLong) throws Exception {
                System.out.println("      subscriber3: "+aLong);
            }
        };

        Observable<Long> observable = Observable.create(new ObservableOnSubscribe<Long>() {
            @Override
            public void subscribe(@NonNull ObservableEmitter<Long> e) throws Exception {
                Observable.interval(10, TimeUnit.MILLISECONDS,Schedulers.computation())
                        .take(Integer.MAX_VALUE)
                        .subscribe(e::onNext);
            }
        }).observeOn(Schedulers.newThread());

        PublishSubject<Long> subject = PublishSubject.create();
        observable.subscribe(subject);

        subject.subscribe(subscriber1);
        subject.subscribe(subscriber2);

        try {
            Thread.sleep(20L);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        subject.subscribe(subscriber3);

        try {
            Thread.sleep(100L);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
```

执行结果跟上面使用 publish 操作符是一样的。

Subject 既是 Observable 又是 Observer(Subscriber)。这一点可以从 Subject 的源码上看到。

```java
import io.reactivex.*;
import io.reactivex.annotations.*;

/**
 * Represents an Observer and an Observable at the same time, allowing
 * multicasting events from a single source to multiple child Subscribers.
 * <p>All methods except the onSubscribe, onNext, onError and onComplete are thread-safe.
 * Use {@link #toSerialized()} to make these methods thread-safe as well.
 *
 * @param <T> the item value type
 */
public abstract class Subject<T> extends Observable<T> implements Observer<T> {
    /**
     * Returns true if the subject has any Observers.
     * <p>The method is thread-safe.
     * @return true if the subject has any Observers
     */
    public abstract boolean hasObservers();

    /**
     * Returns true if the subject has reached a terminal state through an error event.
     * <p>The method is thread-safe.
     * @return true if the subject has reached a terminal state through an error event
     * @see #getThrowable()
     * &see {@link #hasComplete()}
     */
    public abstract boolean hasThrowable();

    /**
     * Returns true if the subject has reached a terminal state through a complete event.
     * <p>The method is thread-safe.
     * @return true if the subject has reached a terminal state through a complete event
     * @see #hasThrowable()
     */
    public abstract boolean hasComplete();

    /**
     * Returns the error that caused the Subject to terminate or null if the Subject
     * hasn't terminated yet.
     * <p>The method is thread-safe.
     * @return the error that caused the Subject to terminate or null if the Subject
     * hasn't terminated yet
     */
    @Nullable
    public abstract Throwable getThrowable();

    /**
     * Wraps this Subject and serializes the calls to the onSubscribe, onNext, onError and
     * onComplete methods, making them thread-safe.
     * <p>The method is thread-safe.
     * @return the wrapped and serialized subject
     */
    @NonNull
    public final Subject<T> toSerialized() {
        if (this instanceof SerializedSubject) {
            return this;
        }
        return new SerializedSubject<T>(this);
    }
}
```

当 Subject 作为 Subscriber 时，它可以订阅目标 Cold Observable 使对方开始发送事件。同时它又作为Observable 转发或者发送新的事件，让 Cold Observable 借助 Subject 转换为 Hot Observable。

注意，Subject 并不是线程安全的，如果想要其线程安全需要调用`toSerialized()`方法。(在RxJava1.x的时代还可以用 SerializedSubject 代替 Subject，但是在RxJava2.x以后SerializedSubject不再是一个public class）
 然而，很多基于 EventBus 改造的 RxBus 并没有这么做，包括我以前也写过这样的 RxBus :( 。这样的做法是非常危险的，因为会遇到并发的情况。

#### Hot Observable 如何转换成 Cold Observable？

##### 1. ConnectableObservable的refCount操作符

reactivex官网的解释是

> make a Connectable Observable behave like an ordinary Observable



![img](https:////upload-images.jianshu.io/upload_images/2613397-7a9fbb425a012864.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000)

RefCount.png

RefCount操作符把从一个可连接的 Observable 连接和断开的过程自动化了。它操作一个可连接的Observable，返回一个普通的Observable。当第一个订阅者订阅这个Observable时，RefCount连接到下层的可连接Observable。RefCount跟踪有多少个观察者订阅它，直到最后一个观察者完成才断开与下层可连接Observable的连接。

如果所有的订阅者都取消订阅了，则数据流停止。如果重新订阅则重新开始数据流。

```java
        Consumer<Long> subscriber1 = new Consumer<Long>() {
            @Override
            public void accept(@NonNull Long aLong) throws Exception {
                System.out.println("subscriber1: "+aLong);
            }
        };

        Consumer<Long> subscriber2 = new Consumer<Long>() {
            @Override
            public void accept(@NonNull Long aLong) throws Exception {
                System.out.println("   subscriber2: "+aLong);
            }
        };

        ConnectableObservable<Long> connectableObservable = Observable.create(new ObservableOnSubscribe<Long>() {
            @Override
            public void subscribe(@NonNull ObservableEmitter<Long> e) throws Exception {
                Observable.interval(10, TimeUnit.MILLISECONDS,Schedulers.computation())
                        .take(Integer.MAX_VALUE)
                        .subscribe(e::onNext);
            }
        }).observeOn(Schedulers.newThread()).publish();
        connectableObservable.connect();

        Observable<Long> observable = connectableObservable.refCount();

        Disposable disposable1 = observable.subscribe(subscriber1);
        Disposable disposable2 = observable.subscribe(subscriber2);

        try {
            Thread.sleep(20L);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        disposable1.dispose();
        disposable2.dispose();

        System.out.println("重新开始数据流");

        disposable1 = observable.subscribe(subscriber1);
        disposable2 = observable.subscribe(subscriber2);

        try {
            Thread.sleep(20L);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
```

执行结果：

```
subscriber1: 0
   subscriber2: 0
subscriber1: 1
   subscriber2: 1
重新开始数据流
subscriber1: 0
   subscriber2: 0
subscriber1: 1
   subscriber2: 1
```

如果不是所有的订阅者都取消了订阅，只取消了部分。部分的订阅者重新开始订阅，则不会从头开始数据流。

```java
        Consumer<Long> subscriber1 = new Consumer<Long>() {
            @Override
            public void accept(@NonNull Long aLong) throws Exception {
                System.out.println("subscriber1: "+aLong);
            }
        };

        Consumer<Long> subscriber2 = new Consumer<Long>() {
            @Override
            public void accept(@NonNull Long aLong) throws Exception {
                System.out.println("   subscriber2: "+aLong);
            }
        };

        Consumer<Long> subscriber3 = new Consumer<Long>() {
            @Override
            public void accept(@NonNull Long aLong) throws Exception {
                System.out.println("      subscriber3: "+aLong);
            }
        };

        ConnectableObservable<Long> connectableObservable = Observable.create(new ObservableOnSubscribe<Long>() {
            @Override
            public void subscribe(@NonNull ObservableEmitter<Long> e) throws Exception {
                Observable.interval(10, TimeUnit.MILLISECONDS,Schedulers.computation())
                        .take(Integer.MAX_VALUE)
                        .subscribe(e::onNext);
            }
        }).observeOn(Schedulers.newThread()).publish();
        connectableObservable.connect();

        Observable<Long> observable = connectableObservable.refCount();

        Disposable disposable1 = observable.subscribe(subscriber1);
        Disposable disposable2 = observable.subscribe(subscriber2);
        observable.subscribe(subscriber3);

        try {
            Thread.sleep(20L);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        disposable1.dispose();
        disposable2.dispose();

        System.out.println("subscriber1、subscriber2 重新订阅");

        disposable1 = observable.subscribe(subscriber1);
        disposable2 = observable.subscribe(subscriber2);

        try {
            Thread.sleep(20L);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
```

执行结果：

```
subscriber1: 0
   subscriber2: 0
      subscriber3: 0
subscriber1: 1
   subscriber2: 1
      subscriber3: 1
subscriber1、subscriber2 重新订阅
      subscriber3: 2
subscriber1: 2
   subscriber2: 2
      subscriber3: 3
subscriber1: 3
   subscriber2: 3
      subscriber3: 4
subscriber1: 4
   subscriber2: 4
```

在这里，subscriber1和subscriber2先取消了订阅，subscriber3并没有取消订阅。之后，subscriber1和subscriber2又重新订阅。最终subscriber1、subscriber2、subscriber3的值保持一致。

## 2. Observable的share操作符

share操作符封装了publish().refCount()调用，可以看其源码。

```java
    /**
     * Returns a new {@link ObservableSource} that multicasts (shares) the original {@link ObservableSource}. As long as
     * there is at least one {@link Observer} this {@link ObservableSource} will be subscribed and emitting data.
     * When all subscribers have disposed it will dispose the source {@link ObservableSource}.
     * <p>
     * This is an alias for {@link #publish()}.{@link ConnectableObservable#refCount()}.
     * <p>
     * ![](http://upload-images.jianshu.io/upload_images/2613397-81dcef165b69aca2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
     * <dl>
     *  <dt><b>Scheduler:</b></dt>
     *  <dd>{@code share} does not operate by default on a particular {@link Scheduler}.</dd>
     * </dl>
     *
     * @return an {@code ObservableSource} that upon connection causes the source {@code ObservableSource} to emit items
     *         to its {@link Observer}s
     * @see <a href="http://reactivex.io/documentation/operators/refcount.html">ReactiveX operators documentation: RefCount</a>
     */
    @CheckReturnValue
    @SchedulerSupport(SchedulerSupport.NONE)
    public final Observable<T> share() {
        return publish().refCount();
    }
```

#### 总结

理解了 Hot Observable 和 Cold Observable 的区别才能够写出更好Rx代码。同理，也能理解Hot & Cold Flowable。再者，在其他语言的Rx版本中包括 RxSwift、RxJS 等也存在 Hot Observable 和 Cold Observable 这样的概念。