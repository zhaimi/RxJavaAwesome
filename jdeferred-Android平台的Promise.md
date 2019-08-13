### 参考项目
https://github.com/jdeferred/jdeferred

### 笔记
如果when多个里面有一个出错都会直接调用`fail`不会触发一次`done`
依赖方法：

```
compile 'org.jdeferred:jdeferred-android-aar:${version}'
// or
compile 'org.jdeferred:jdeferred-android-aar:${version}@aar'
```

这个库是流式思想的实践，类同于rxjava，但更加简单（当然功能也没有rxjava强大）
核心：Promise<D,F,P>
 常用类：DeferredObject, DeferredManager
1,核心promise的方法：

```java
then()  //万能方法：可以传入DoneCallback, FailCallback, ProgressCallback, DoneFilter, FailFilter, ProgressFilter, DonePipe, FailPipe, ProgressPipe, 返回promise
done() //完成回调
fail() //失败回调
progress（） //进度回调
always() //无论什么结果都会回调
```

2，DeferredObject

```java
Deferred deferred = new DeferredObject();
deferred.resolve("成功了") //需要传给done的结果
deferred.reject("失败了")//需要传给fail的结果
deferred.notify(100)//需要传给progress的结果
实现了promise接口，可以调用promise内的方法
```

3，DeferredManager

```java
DeferredManager dm = new DefaultDeferredManager();
DeferredManager dm = new DefaultDeferredManager(executorService);//默认的ExecutorService是Executors.newCachedThreadPool();
AndroidDeferredManager dm = new AndroidDeferredManager();//针对android，AndroidDeferredManager 继承了DefaultDeferredManager

核心方法：dm.when(params)
params有以下几种方式：
Callable, Runnable, Future, 
DeferredRunnable, DeferredCallable, DeferredFutureTask，
DeferredAsyncTask
(中三种是上三种的子类，内部持有一个DeferredObject，上三种传入后也会被封装为DeferredFutureTask,内部持有一个DeferredObject)
(DeferredFutureTask继承于FutureTask(实现Runnable\Future,并持有Callable))
(DeferredAsyncTask继承与AsyncTask,是android特有的，内部持有一个DeferredObject)
```

4,支持同步和异步
 a,同步的使用：

```java
Deferred deferred = new DeferredObject();
Promise promise = deferred.promise();
promise.done(new DoneCallback() {
  public void onDone(Object result) {
    ...
  }
}).fail(new FailCallback() {
  public void onFail(Object rejection) {
    ...
  }
}).progress(new ProgressCallback() {
  public void onProgress(Object progress) {
    ...
  }
}).always(new AlwaysCallback() {
  public void onAlways(State state, Object result, Object rejection) {
    ...
  }
});

deferred.resolve("done");
deferred.reject("oops");
deferred.notify("100%");
原理：DeferredObject内部持有一个promise，当调用resolve()时，回调promise.done();调用reject()时，回调promise.fail()..以此类推
```

或者

```java
DeferredManager dm = new DefaultDeferredManager();
Promise p1, p2, p3;
// initialize p1, p2, p3
dm.when(p1, p2, p3)
  .done(…)
  .fail(…)
```

原理：DeferredManager内部对promise的返回做了原子性处理，保证了线程安全

b，异步的使用：

三种用法：

```java
//第一种：线程中使用deferred

final Deferred deferred = ...
Promise promise = deferred.promise();
promise.then(…);
Runnable r = new Runnable() {
  public void run() {
    while (…) {
      deferred.notify(myProgress);
    }
    deferred.resolve("done");
  }
}
//第二种：使用DeferredManager 和DeferredRunnable

DeferredManager dm = …;
dm.when(new DeferredRunnable<Double>(){
  public void run() {
    while (…) {
      notify(myProgress);
    }
  }
}).then(…);
//第三种：使用AndroidDeferredManager(内部封装了AsyncTask和handler)
AndroidDeferredManager dm = new AndroidDeferredManager();
dm.when(new DeferredAsyncTask<Void, Progress, Result>(){
      @Override
      protected Object doInBackgroundSafe(Void... voids) throws Exception {
          return null;
      }
}).then(...);
```



