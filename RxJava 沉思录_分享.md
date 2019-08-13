# RxJava 沉思录

### RxJava 真的好用吗?
`Rxjava`问世已经很多年了,我大概14年开始接触,后面通过 **扔物线** 的这篇文章 [ <给 Android 开发者的 RxJava 详解> ](https://gank.io/post/560e15be2dca930e00da1083) 开始入门学习的,当时看了很多遍,甚至研究过其源码,直到现在断断续续用了几年的`RxJava`,对这个框架设计理念以及优势仍然是懵懵懂懂,如果有人问:**RxJava到底好在哪？好用吗？** 我估计还是会一脸懵逼.

18年底,我看了一篇文章[《RxJava沉思录》](https://juejin.im/post/5b8f536c5188255c352d3528) ,这篇文章让我略微颠覆了对`Rxjava`的认知

### RxJava -->观察者模式?
几乎所有`RxJava`入门介绍，都会用一定的篇幅去介绍`观察者模式`，告诉你观察者模式是`RxJava`的核心，是基石:

```kotlin
Observable.just("123").subscribe(object : Observer<String> {
    override fun onComplete() {}

    override fun onSubscribe(d: Disposable) {}

    override fun onNext(s: String) {}

    override fun onError(e: Throwable) {}
})
```

这个例子所有关于`Rxjava`的文章一定会举例,但是仔细想想,这个`观察者模式`真的很高大上吗?
如果真的很厉害,那下面这段代码呢?
```java
button.setOnClickListener{v -> doSomeThing()}
```
这个应该是所有开发者接触的第一个`观察者模式`了=_=,所以`观察者模式`肯定不是我们使用`RxJava`的根本理由

### RxJava -->链式编程?

首先区分`链式编程`和`响应式编程`,这两种编程思想的代码格式很相似,
`链式编程`使用的就是Builder模式,例如非常经典的Android AlertDialog的创建
```java
AlertDialog.Builder builder = new AlertDialog.Builder(this)
              .setTitle("Hello Dialog")
              .setIcon(R.drawable.logo)
              .setMessage("This is a Dialog")
              .show();
```
`链式编程`一般没有严格的前后顺序关系,比如上面的setTitle和setMessage,谁前谁后无所谓

而`响应式编程`是基于事件驱动的,类似流水线,比如:
```java
Observable.just(context)
          .map{拿出钥匙}
          .map{开门}
          .map{拔掉钥匙}
          .map{关门}
          .subscribe()
```
`响应式编程`有比较鲜明的前后顺序,后面的步骤要依赖于之前的步骤

回到正题,每次谈到`RxJAVA`,必然少不了`链式编程`这个词汇,我们大部分人也形容链式编程是`RxJava`解决异步任务的“杀手锏”

```java
Observable.from(folders)
    .flatMap((Func1) (folder) -> { Observable.from(file.listFiles()) })
    .filter((Func1) (file) -> { file.getName().endsWith(".png") })
    .map((Func1) (file) -> { getBitmapFromFile(file) })
    .subscribeOn(Schedulers.io())
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe((Action1) (bitmap) -> { imageCollectorView.addImage(bitmap) });
```
这段代码出现的频率非常的高，好像是`RxJava`的链式编程给我们带来的好处的最佳佐证。然而回过头仔细想想，似乎并没有感受到：“它很长，但是很清晰”的感觉。

首先，**flatMap**， **filter**， **map** 这几个操作符，对于初学者来讲，并不好理解。
其次，虽然这段代码的逻辑本身并不复杂。

上面这段代码通常会带一个反例，使用*new Thread()*的方式实现的版本：

```java
new Thread() {
    @Override
    public void run() {
        super.run();
        for (File folder : folders) {
            File[] files = folder.listFiles();
            for (File file : files) {
                if (file.getName().endsWith(".png")) {
                    final Bitmap bitmap = getBitmapFromFile(file);
                    getActivity().runOnUiThread(new Runnable() {
                        @Override
                        public void run() {
                            imageCollectorView.addImage(bitmap);
                        }
                    });
                }
            }
        }
    }
}.start();
```

看起来，上面这段代码似乎真的没那么简洁，真是这样吗？既然都用了`lambda`，那么都得用才公平吧：

```java
new Thread(() -> {
    File[] pngFiles = new File[]{};
    for (File folder : folders) {
        pngFiles = ArrayUtils.addAll(pngFiles, folder.listFiles());
    }
    for (File file : pngFiles) {
        if (file.getName().endsWith(".png")) {
            final Bitmap bitmap = getBitmapFromFile(file);
            getActivity().runOnUiThread(() -> imageCollectorView.addImage(bitmap));
        }
    }
}).start();
```
坦率地讲，这段代码除了**new Thread().start()**有槽点以外，没什么大毛病。`RxJava`版本确实代码更少，同时省去了一个中间变量pngFiles。但其实这两种写法无论从性能还是项目可维护性上来看，并没有太大的差距，甚至，后一种写法反而更容易被大家接受。

准确的来说，我的关注点并不在大多数文章鼓吹的“链式编程”这一点上，把多个依次执行的异步操作的调用转化为类似同步代码调用那样的自上而下执行，并不是什么新鲜事，而且就这个具体的例子，`RxJava`相比原生的写法无法体现它的优势。

除此之外，对于处理异步任务，还有**Promise**这个流派，使用类似这样的API:

```java
promise
    .then(r1 -> task1(r1))
    .then(r2 -> task2(r2))
    .then(r3 -> task3(r3))
    ...
```

似乎比`RxJava`更加简洁直观，而且还不需要引入函数式编程的内容。这种写法，跟所谓的“逻辑简洁”也根本没什么关系。

这就是我要说的第二个论点：**链式编程对于RxJava来讲更像是一种语法糖**。

### RxJava等于异步加简洁吗？

**扔物线**用两个词形容了`RxJava`：**异步**、**简洁**。

虽然我们使用`RxJava`的场景大多数与异步有关，但是这个框架并不是与异步等价的。举个简单的例子：

```kotlin
Observable.just(1, 2, 3).subscribe { Log.i("TAG", "$it") }
```
上面的代码就是同步执行的，和异步没有关系。事实上，**RxJava默认就是同步执行的**，除非你手动切换到其他的`Scheduler`。

再看官方的介绍：

> a library for composing asynchronous and event-based programs by using observable sequences.

> 通过使用可观察序列来编写异步和基于事件的程序的库。

简言之，`RxJava`是一种**事件驱动型**编程范式，它以异步为切入点，试图一统**同步**和**异步**的世界。至此，我认为：**RxJava不等价于异步**。

至于`RxJava`是否简洁，这个见仁见智。一部分人认为`RxJava`减少了代码缩进，逻辑扁平化，看起来的确是简洁了不少；另一部分人可能认为`RxJava`引入了函数式编程、增加不少抽象的操作符，从而维护性降低，理解困难，对人员的要求更高了。

### RxJava是用来解决Callback Hell的吗？

几乎所有介绍`RxJava`的文章都会提到：**RxJava是用来解决Callback Hell问题的**。`Callback Hell`指的是过多的异步回调嵌套导致的代码呈现出的难以阅读的状态。

`Callback Hell`这个问题，最严重的重灾区是在Web领域，是使用`JavaScript`最常见的问题之一。由于客户端编程和前端编程具有一定的相似性，所以`Android`也存在这个问题。

因此类似`Promise`这样专门为异步编程打造的框架应用而生，`Android`平台上也有类似的实现[jdeferred](https://github.com/jdeferred/jdeferred)。

在我看来，`Promise`那样的框架，更像是那种纯粹的用来解决`Callback Hell`的框架。至于`RxJava`，我觉得它是一个更有野心的框架，正确使用了`RxJava`的话，的确能够解决`Callback Hell`的问题。但如果说`RxJava`就是用来解决 `Callback Hell`的难免有些以偏概全了。

另外**协程**现在也非常火，这个从语言层面实现了异步任务，几乎彻底解决了**Callback Hell**的问题，且更加接近于人类自然语言。`Kotlin`已经支持协程，相信标准`JDK`不久后也会加入。

```kotlin
private fun loadData() = launch(uiContext) {
    // ui thread
    view.showLoading() 
 
    val task1 = async(bgContext) { dataProvider.loadData("Task 1") }
    val task2 = async(bgContext) { dataProvider.loadData("Task 2") }
 
    // non ui thread, suspend until finished
    val result = "${task1.await()} ${task2.await()}" 
 
    // ui thread
    view.showData(result) 
}
```

### 如何理解RxJava

经过上面的分析，或许大家已经产生了对`RxJava`的消极想法，并且会产生一个疑问：那么`RxJava`存在的意义究竟是什么呢？

先举几个常见的例子：

1. 为View设置点击回调方法:

```java
btn.setOnClickListener(new OnClickListener() {
    @Override
    public void onClick(View v) {
        // callback body
    }
});
```

2. Service组件绑定操作:

```java
private ServiceConnection mConnection = new ServiceConnection() {
    @Override
    public void onServiceConnected(ComponentName className, IBinder service) {
        // callback body
    }
    @Override
    public void onServiceDisconnected(ComponentName arg0) {
        // callback body
    }
};
...
bindService(intent, mConnection, Context.BIND_AUTO_CREATE);
```

3. 使用Retrofit发起网络请求:

```java
Call<List<Photo>> call = service.getAllPhotos();
call.enqueue(new Callback<List<Photo>>() {
    @Override
    public void onResponse(Call<List<Photo>> call, Response<List<Photo>> response) {
        // callback body
    }
    @Override
    public void onFailure(Call<List<Photo>> call, Throwable t) {
        // callback body
    }
});
```

在日常开发中我们时时刻刻在面对着类似的回调函数，而且容易看出来，回调函数最本质的功能就是**返回异步调用的结果**给我们，剩下的都是大同小异。所以我们能不能不要去记忆各种各样的回调函数，只使用一种回调呢？如果我们定义统一的回调如下：

```java
public interface Callback<T> {
    public void onResult(T result);
}
```

那么以上 3 种情况，对应的回调变成了：

1. 为View设置点击事件对应的回调为`Callback<View>`
2. Service 组件绑定操作对应的回调为 `Callback<Pair<CompnentName, IBinder>> (onServiceConnected)`、 `Callback<CompnentName> (onServiceDisconnected)`
3. 使用Retrofit发起网络请求对应的回调为`Callback<List<Photo>> (onResponse)`、`Callback<Throwable> (onFailure)`

只要按照这种思路，我们可以把所有的异步回调封装成`Callback<T>`的形式，我们不再需要去记忆不同的回调，只需要和一种回调交互就可以了。

基于此，我认为`RxJava`存在首先最基本的意义就是：**统一了所有异步任务的回调接口**，而这个接口就是`Observable<T>`（即上面提到的`Callback<T>`），而有了 `Observable`以后的`RxJava`才刚刚插上了想象力的翅膀。

### 时间维度

前面已经提到了`Observable`，那么我们来看看它在时间维度上的重新组织事件的能力。
##### 点击事件防抖动

咱们测试同学经常干的事情，对一个可点View疯狂的快速点击导致一些Bug，每每提到类似Bug，真的不胜其烦。

以前我们最常用的方法就是:
```kotlin
var lastClickTimeStamp = 0L

btn.setOnClickListener(v -> {
    if (System.currentTimeMillis() - lastClickTimeStamp < 500) {
        // handle double click
    }
})
```
在`RxJava`中,只需要写成一下代码:

```kotlin
fun View.click(delayTime: Long = 500, onClick: (v: View) -> Unit) {
    RxClick(this).throttleFirst(delayTime, TimeUnit.MILLISECONDS).subscribe { onClick(this) }
}
```
此处夸一下Kotlin的优点-->**扩展** 

> **throttleFirst** 操作符仅发射固定时间长度内的第一个事件

![throttleFirst](http://reactivex.io/documentation/operators/images/throttleFirst.png)

虽然这个例子比较简单，但是它很好的表达了`Observable`可以在**时间维度上对其发射的事件进行重新组织**，从而做到之前 Callback形式不容易做到的事情

##### 检测双击事件

常规方式：

```kotlin
var lastClickTimeStamp = 0L

btn.setOnClickListener(v -> {
    if (System.currentTimeMillis() - lastClickTimeStamp < 500) {
        // handle double click
    }
})
```

RxJava方式：

```java
Observable<Long> clicks = RxView.clicks(btn)
    .map(o -> System.currentTimeMillis())
    .share();
    
clicks.zipWith(clicks.skip(1), (t1, t2) -> t2 - t1)
    .filter(interval -> interval < 500)
    .subscribe(o -> {
        // handle double click
    });
```

> **zipWith** 操作符对事件流自身相邻的两个元素做比较

![zipWith](http://reactivex.io/documentation/operators/images/zip.i.png)

> **share** 操作符在多个订阅者间共享源(其实就把它当做.publish().refcount()的调用链)



**==   怎么更复杂了。。。**



别急，马上加需求了：

> 如果用户在“短时间”内连续多次点击，只能算一次双击操作，这个需求显然是合理的。如果按照上面Callback的写法，虽然可以检测出双击操作，但是如果用户快速点击n次（间隔均小于500毫秒，n >= 2）, 就会触发n-1次双击事件

要实现这个需求也很简单：

```java
Observable<Object> clicks = RxView.clicks(btn).share()
clicks.buffer(clicks.debounce(500, TimeUnit.MILLISECONDS))
    .filter(events -> events.size >= 2)
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(o -> {
        // handle double click
    });
```

> **buffer** 操作符接受一个`Observable`，并在指定时间内把这个`Observable`所发射的元素分组，并将分组后的结果以新的`Observable`发射出去。
![Buffer](http://reactivex.io/documentation/operators/images/Buffer.png)


> **debounce** 操作符仅在过了指定的时间还没发射数据时才发射一个数据

![Debounce](http://reactivex.io/documentation/operators/images/debounce.png)

##### 实时搜索

首先我们需要考虑以下问题：

> 1. 防止用户输入过快，触发过多网络请求，需要对输入事件做一下防抖动。

> 2. 用户在输入关键词过程中可能触发多次请求，那么，如果后一次请求的结果先返回，前一次请求的结果后返回，这种情况应该保证界面展示的是后一次请求的结果。

> 3. 用户在输入关键词过程中可能触发多次请求，那么，如果后一次请求的结果返回时，前一次请求的结果尚未返回的情况下，就应该取消前一次请求。

综合考虑上面的因素以后，我们使用`RxJava`实现的代码如下：

```java
RxTextView.textChanges(input)
    .debounce(500, TimeUnit.MILLISECONDS)
    .switchMap(text -> api.queryKeyword(text.toString()))
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(results -> {
        // handle results
    });
```

> **swithMap** 当原始`Observable`发射一个新的数据时，它将取消订阅并停止监视之前那个数据的`Observable`，只监视当前这一个。

![switchMap](http://reactivex.io/documentation/operators/images/switchMap.png)

##### 综述

**RxJava的Observable可以通过一系列操作符从时间的维度上重新组织事件，从而简化订阅者的逻辑**


### 空间维度


#### 网络请求合并
情景：实现一个具有多种类型的`RecyclerView`列表，如图所示：

![multi_item_recyclervie](https://user-gold-cdn.xitu.io/2018/9/5/165a7e27adc8b361?imageView2/0/w/1280/h/960/ignore-error/1)

假设列表中有3种类型的数据，这3种类型数据来源于三个不同接口，这3种类型在列表中出现的顺序是可配置的，而且3种类型数据不一定全部需要展示，因此我们定义`Retrofit`接口如下：

```java
public interface Api {
    @GET("/path/to/api")
    Observable<List<ItemA>> getListOfA();
    @GET("/path/to/api")
    Observable<List<ItemB>> getListOfB();
    @GET("/path/to/api")
    Observable<List<ItemC>> getListOfC();
    // 需要展示的数据顺序
    @GET("/path/to/api")
    Observable<List<String>> getColumns();
}
```

假设*getColumns*接口返回的数据形如：

* ["a", "b", "c"]
* ["b", "a"]
* ["b", "c"]

使用`RxJava`来实现这个需求：

```java
Api api = ...
api.getColumns()
    .map(types -> {
        List<Observable<? extends List<? extends Item>>> list = new ArrayList<>();
        for (String type : types) {
            switch (type) {
                case "a":
                    list.add(api.getListOfA().startWith(new ArrayList<ItemA>()));
                    break;
                case "b":
                    list.add(api.getListOfB().startWith(new ArrayList<ItemB>()));
                    break;
                case "c":
                    list.add(api.getListOfAC().startWith(new ArrayList<ItemC>()));
                    break;
            }
        }
        return list;
    })
    .flatMap(requestObservables -> Observable.combineLatest(requestObservables, objects -> {
        List<Item> items = new ArrayList<>();
        for (Object response : objects) {
            items.addAll((List<? extends Item>) response);
        }
        return items;
    }))
    .subscribeOn(Schedulers.io())
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(data -> {
        adapter.setData(data);
        adapter.notifyDataSetChanged();
    });
```

> **map** 对发射的每一项数据应用一个函数，执行变换操作
![map](http://reactivex.io/documentation/operators/images/map.png)

> **startWith** 在数据序列的开头插入一条指定的项
![startWith](http://reactivex.io/documentation/operators/images/startWith.png)
> **FlatMap** 将一个发射数据的`Observable`变换为多个`Observables`，然后将它们发射的数据合并后放进一个单独的`Observable`
![flatMap](http://reactivex.io/documentation/operators/images/mergeMap.png)
> **combineLatest** 当`Observables`中的任何一个发射了数据时，使用一个函数结合每个`Observable`发射的最近数据项，并且基于这个函数的结果发射数据

![combineLatest](http://reactivex.io/documentation/operators/images/combineLatest.png)

通过`RxJava`的操作符，我们把4个接口返回的4个`Observable` **在空间维度进行了重新组织**，最终把它们转成了一个`Observable`，发射的元素类型是`List<Item>`，而这正是我们的订阅者(Adapter)所关心的数据类型，订阅者只需要监听这个`Observable`，并更新数据即可，并不需要关心数据到底来自哪里。


#### 缓存和网络请求

当我们进入一个新的页面，为了提升用户体验，不让页面空白太久，我们一般会先读取缓存中的数据，再去请求网络。目前产品需求是: 同时发起读取缓存、访问网络的请求，如果缓存的数据先回来，那么就先展示缓存的数据，而如果网络的数据先回来，那么就不再展示缓存的数据。

我们先分别定义	`Observable` 分别表示`缓存数据源` 和 `网络数据源`
```java
 private Observable<List<Result>> getCacheArticle(final long simulateTime) {
        return Observable.create(new ObservableOnSubscribe<List<Result>>() {
            @Override
            public void subscribe(ObservableEmitter<List<Result>> observableEmitter) throws Exception {
                try {
                    Log.d(TAG, "开始加载缓存数据");
                    Thread.sleep(simulateTime);
                    List<Result> results = new ArrayList<>();
                    for (int i = 0; i < 10; i++) {
                        Result entity = new Result();
                        entity.setType("缓存");
                        entity.setDesc("序号=" + i);
                        results.add(entity);
                    }
                    observableEmitter.onNext(results);
                    observableEmitter.onComplete();
                    Log.d(TAG, "结束加载缓存数据");
                } catch (InterruptedException e) {
                    if (!observableEmitter.isDisposed()) {
                        observableEmitter.onError(e);
                    }
                }
            }
        });
    }
```
```java
    private Observable<List<Result>> getNetworkArticle(final long simulateTime) {
        return Observable.create(new ObservableOnSubscribe<List<Result>>() {
            @Override
            public void subscribe(ObservableEmitter<List<Result>> observableEmitter) throws Exception {
                try {
                    Log.d(TAG, "开始加载网络数据");
                    Thread.sleep(simulateTime);
                    List<Result> results = new ArrayList<>();
                    for (int i = 0; i < 10; i++) {
                        Result entity = new Result();
                        entity.setType("网络");
                        entity.setDesc("序号=" + i);
                        results.add(entity);
                    }
                    //a.正常情况。
                    observableEmitter.onNext(results);
                    observableEmitter.onComplete();
                    //b.发生异常。
                    //observableEmitter.onError(new Throwable("netWork Error"));
                    Log.d(TAG, "结束加载网络数据");
                } catch (InterruptedException e) {
                    if (!observableEmitter.isDisposed()) {
                        observableEmitter.onError(e);
                    }
                }
            }
        }).onErrorResumeNext(new Function<Throwable, ObservableSource<? extends List<NewsResultEntity>>>() { 

            @Override
            public ObservableSource<? extends List<Result>> apply(Throwable throwable) throws Exception {
              //onErrorResumeNext 为了解决网络请求出错后,无法获取缓存的问题
              //如果网络请求先返回时发生了错误（例如没有网络等）导致发送了onError事件，从而使得缓存的Observable也无法发送事件，最后界面显示空白。
                Log.d(TAG, "网络请求发生错误throwable=" + throwable);
                return Observable.never();
            }
        });
    }
```

我们通过**publish**,**merge**和**takeUnti** 三个操作符实现刚才的产品需求(该需求不可以将merge替换为contact)

```java
 Observable<List<Result>> publishObservable =
                getNetworkArticle(500).subscribeOn(Schedulers.io())
                        .publish(new Function<Observable<List<Result>>, ObservableSource<List<Result>>>() {

            @Override
            public ObservableSource<List<Result>> apply(Observable<List<Result>> network) throws Exception {
                return Observable.merge(network, getCacheArticle(2000).subscribeOn(Schedulers.io()).takeUntil(network));
            }

        });
DisposableObserver<List<Result>> disposableObserver = getArticleObserver();
        publishObservable.observeOn(AndroidSchedulers.mainThread()).subscribe(disposableObserver);
 
```

> **merge**通过合并多个Observable发出的结果将多个Observable合并成一个Observable的操作。
![merge](http://reactivex.io/documentation/operators/images/merge.png)

> **publish** 将一个Observable转换为一个可连接的Observable
![publish](http://reactivex.io/documentation/operators/images/publishConnect.c.png)

> **takeUnti**  当发射的数据满足某个条件后（包含该数据），或者第二个Observable发送完毕，终止第一个Observable发送数据
![takeUtil](http://reactivex.io/documentation/operators/images/takeUntil.png)

##### 综述

从这几个例子来看，`RxJava`学习的曲线无疑是陡峭的，不过我认为以上的例子很好的表达我这一节要阐述的观点：**`Observable`在空间维度上对事件的重新组织，让我们的事件驱动型编程更具想象力**。

诚然，上述代码的确很难理解，但细细想来似乎还是有值得我们思考的地方。在我们固有的编程思维中，面对多少个异步任务，就会写多少个回调，如果任务之间有依赖关系，通常的做法就是修改订阅者（回调函数）逻辑以及新增数据结构保证依赖关系。`RxJava`给我们带来的新思路是，`Observable`的事件在到达订阅者之前，可以先通过操作符进行一系列变换，对订阅者屏蔽数据产生的复杂性，只提供给订阅者简单的数据接口。

### 总结

`RxJava`是一个思想优秀的框架，而且是那种在工程领域少见的带有学院派气息和理想主义色彩的框架，它是一种新型的事件驱动型编程范式。`RxJava`最重要的贡献，就是提升了我们原先对于事件驱动型编程的思考的维度，允许我们可以从时间和空间两个维度去重新组织事件。它以异步为切入点，试图统一**同步**和**异步**的世界。

或许工作中我们用得到的不多，但希望能从中领悟到不同的**编程思想**。

### 

### 参考文章:

[<RxJava操作符大全>](https://www.jianshu.com/p/3fdd9ddb534b)
[<RxJava官网-Observable>](http://reactivex.io/documentation/observable.html)
[<RxJava官网-Operators>](http://reactivex.io/documentation/operators.html)
[<RxJava官网-Single>](http://reactivex.io/documentation/single.html)
[<RxJava官网-Subject>](http://reactivex.io/documentation/subject.html)
[<RxJava官网-Scheduler>](http://reactivex.io/documentation/scheduler.html)
[<RxJava2 实战系列文章>](https://www.jianshu.com/p/c935d0860186)

### 参考DEMO
[ RxJava2Examples ](https://github.com/nanchen2251/RxJava2Examples)
[ RxSample ](https://github.com/imZeJun/RxSample)
[ RxJava2-Android-Samples ](https://github.com/amitshekhariitbhu/RxJava2-Android-Samples)

### 源码分析

[RxJava2.X 源码解析（一）： 探索RxJava2分发订阅流程](https://www.catbro.cn/detail/5a6037029a7b56c5c24fbc66.html)
[RxJava2.X 源码解析（二) ：探索RxJava2神秘的随意取消订阅流程的原理 ](https://www.catbro.cn/detail/5a6037029a7b56c5c24fbc68.html)
[RxJava2.X 源码分析（三）：探索RxJava2之订阅线程切换原理](https://www.catbro.cn/detail/5a6037029a7b56c5c24fbc69.html)
[RxJava2.X 源码分析（四）：探索RxJava2之观察者线程切换原理 ](https://www.catbro.cn/detail/5a6037029a7b56c5c24fbc6b.html)
[RxJava2.X 源码分析（五）：论RxJava2.X切换线程次数的有效性 ](https://www.catbro.cn/detail/5a6037029a7b56c5c24fbc6c.html)
[RxJava2.X 源码分析（六）：变换操作符的实现原理（上）](https://www.catbro.cn/detail/5a6037029a7b56c5c24fbc6d.html)
[RxJava2.X 源码分析（七）：变换操作符的实现原理强化篇（下）](https://www.catbro.cn/detail/5a6037029a7b56c5c24fbc6e.html)