## Rxjava2基础认知

- 形式正确的有限Observable
  调用观察者的onCompleted正好一次或者它的onError正好一次，而且此后不能再调用观察者的任何其它方法。如果onComplete 或者 onError 走任何一个 都会 主动解除订阅关系；
  - 如果解除订阅关系以后在发射 onError 则会 报错;而发射onComplete则不会。
  - 注意解除订阅关系 还是可以发射 onNext
- Disposable类:
  - dispose():主动解除订阅
  - isDisposed():查询是否解除订阅 true 代表 已经解除订阅
- CompositeDisposable类:可以快速解除所有添加的Disposable类
  每当我们得到一个Disposable时就调用CompositeDisposable.add()将它添加到容器中, 在退出的时候, 调用CompositeDisposable.clear() 即可快速解除.

```
CompositeDisposable compositeDisposable=new CompositeDisposable();
Observable.create(new ObservableOnSubscribe<Integer>() {
            @Override
            public void subscribe(ObservableEmitter<Integer> emitter) throws Exception {
                emitter.onNext(1);
                emitter.onComplete();或者 emitter.onError(new Throwable("O__O "));
            }
        }).subscribe(new Observer<Integer>() {

            private Disposable mDisposable;
            @Override
            public void onSubscribe(Disposable d) {
                <!-- 订阅   -->
                mDisposable = d;
                <!-- 添加到容器中 -->
                compositeDisposable.add(d);
            }

            @Override
            public void onNext(Integer value) {
                <!-- 判断mDisposable.isDisposed() 如果解除了则不需要处理 -->
            }

            @Override
            public void onError(Throwable e) {
            }

            @Override
            public void onComplete() {
            }
        });
        <!-- 解除所有订阅者 -->
        compositeDisposable.clear();
```



## 基础概念

- Scheduler scheduler

> timer() alt+点击timer可查看 关于timer的方法 可以看到时候有这个参数的变体!

[![img](https://ww1.sinaimg.cn/large/006tNc79gy1fhyjg4qv2fj30v204ot9a.jpg)](https://ww1.sinaimg.cn/large/006tNc79gy1fhyjg4qv2fj30v204ot9a.jpg)

- Callable bufferSupplier:自定义装载的容器

```
Observable.range(1, 10)
           //() -> new ArrayList<>() 则是bufferSupplier
           .buffer(2, 1,() -> new ArrayList<>())
           .subscribe(integers -> System.out.println(integers));
```

## 创建操作

- create : 创建一个具有发射能力的Observable

```
Observable.create(e -> {
    e.onNext("Love");
    e.onNext("For");
    e.onNext("You!");
    e.onComplete();
}).subscribe(s -> System.out.println(s));
```

- just:只是简单的原样发射,可将数组或Iterable当做单个数据。它接受一至九个参数

```
Observable.just("Love", "For", "You!")
                .subscribe(s -> System.out.println(s));
```

- empty:创建一个不发射任何数据但是正常终止的Observable
- never:创建一个不发射数据也不终止的Observable
- error:创建一个不发射数据以一个错误终止的Observable

```
Observable.empty();
Observable.never();
Observable.error(new Throwable("O__O"))
```

- timer 在延迟一段给定的时间后发射一个简单的数字0

```
Observable.timer(1000, TimeUnit.MILLISECONDS)
              .subscribe(s -> System.out.println(s));
```

- range:
  - start:起始值
  - count:一个是范 围的数据的数目。0不发送 ，负数 异常

```
Observable.range(5, 3)
        //输出 5,6,7
        .subscribe(s -> System.out.println(s));
```

- intervalRange
  - start,count:同range
  - initialDelay 发送第一个值的延迟时间
  - period  每两个发射物的间隔时间
  - unit,scheduler 额你懂的

```
Observable.intervalRange(5, 100, 3000, 100,
                TimeUnit.MILLISECONDS, Schedulers.io())
                .subscribe(s -> System.out.println(s));
```

- interval:相当于intervalRange的start=0；

  > period 这个值一旦设定后是不可变化的

```
//period 以后的美每次间隔 这个值一旦设定后是不可变化的  所以 count方法无效的！
int[] s = new int[]{0};
 Observable.interval(3000, 100 + count(s), TimeUnit.MILLISECONDS, Schedulers.io())
         .subscribe(s2 -> System.out.println(s2));

 private int count(int[] s) {
         int result = s[0] * 1000;
         s[0] = s[0] + 1;
         return result;
     }
```

- defer 直到有观察者订阅时才创建Observable，并且为每个观察者创建一个新的Observable

```
Observable.defer(() -> Observable.just("Love", "For", "You!"))
              .subscribe(s -> System.out.println(s));
```

- from系列

  - fromArray

    ```
    Integer[] items = {0, 1, 2, 3, 4, 5};
           Observable.fromArray(items).subscribe(
                   integer -> System.out.println(integer));
    ```

  - fromCallable

    ```
    Observable.fromCallable(() -> Arrays.asList("hello", "gaga"))
                    .subscribe(strings -> System.out.println(strings))
    ```

  - fromIterable

    ```
    Observable.fromIterable(Arrays.<String>asList("one", "two", "three"))
                 .subscribe(integer -> System.out.println(integer));
    ```

  - fromFuture

    ```
    Observable.fromFuture(Observable.just(1).toFuture())
                   .doOnComplete(() -> System.out.println("complete"))
                   .subscribe();
    ```

## 过滤操作

- elementAt:只发射第N项数据

```
<!-- 无默认值版本 -->
 Observable.just(1,2)
            .elementAt(0)
            .subscribe(o -> System.out.print(o ));//结果:1

<!-- 带默认值的变体版本 -->
Observable.range(0, 10)
//        如果索引值大于数据 项数，它会发射一个默认值(通过额外的参数指定)，而不是抛出异常。
//    但是如果你传递一 个负数索引值，它仍然会抛出一个 IndexOutOfBoundsException 异常。
                .elementAt(100, -100)
                .subscribe(o -> System.out.print(o + "\t"));
```

- IgnoreElements:如果你不关心一个Observable发射的数据，但是希望在它完成时或遇到错误终止时收到通知

  ```
  Observable.range(0, 10)
                 .ignoreElements()
                 .subscribe(() -> System.out.println("complete")
                         , throwable -> System.out.println("throwable"));
  ```

- take系列

  - 变体 count系列:只发射前面的N项数据

    ```
    Observable.range(0,10)
                   .take(3)
                   .subscribe(o -> System.out.print(o + "\t"))
    ```

  - 变体 time系列: 发射Observable开始的那段时间发射 的数据，

    ```
    Observable.range(0,10)
              .take(100, TimeUnit.MILLISECONDS)
              .subscribe(o -> System.out.print(o + "\t"));
    ```

- takeLast

  - 变体 count系列:只发射后面的N项数据

    ```
    Observable.range(0,10)
                        .takeLast(3)
                        .subscribe(o -> System.out.print(o + "\t"));
    ```

  - 变体 time系列: 发射在原始Observable的生命周 期内最后一段时间内发射的数据

    ```
    Observable.range(0,10)
                   .takeLast(100, TimeUnit.MILLISECONDS)
                   .subscribe(o -> System.out.print(o + "\t"));
    ```

- takeUntil:发送complete的结束条件 当然发送结束之前也会包括这个值

```
Observable.just(2,3,4,5)
        //发送complete的结束条件 当然发送结束之前也会包括这个值
        .takeUntil(integer ->  integer>3)
        .subscribe(o -> System.out.print(o + "\t"));//2
```

[![img](https://images2015.cnblogs.com/blog/693176/201611/693176-20161120161003623-609469114.png)](https://images2015.cnblogs.com/blog/693176/201611/693176-20161120161003623-609469114.png)

- takeWhile:当不满足这个条件 会发送结束 不会包括这个值

```
Observable.just(2,3,4,5)
        //当不满足这个条件 会发送结束 不会包括这个值
        .takeWhile(integer ->integer<=4 )
        .subscribe(o -> System.out.print(o + "\t"));//2,3,4
```

[![img](https://images2015.cnblogs.com/blog/693176/201611/693176-20161120161141123-130674910.png)](https://images2015.cnblogs.com/blog/693176/201611/693176-20161120161141123-130674910.png)

- skip系列

  - 变体 count系列:丢弃Observable发射的前N项数据

    ```
    Observable.range(0,5)
                   .skip(3)
                   .subscribe(o -> System.out.print(o + "\t"));
    ```

  - 变体 time系列:丢弃原始Observable开始的那段时间发 射的数据

    ```
    Observable.range(0,5)
                   .skip(3)
                   .subscribe(o -> System.out.print(o + "\t"));
    ```

- skipLast

  - 变体 count系列:丢弃Observable发射的前N项数据

    ```
    Observable.range(0,5)
                   .skipLast(3)
                   .subscribe(o -> System.out.print(o + "\t"));
    ```

  - 变体 time系列:丢弃在原始Observable的生命周 期内最后一段时间内发射的数据

    ```
    Observable.range(0,10)
                 .skipLast(100, TimeUnit.MILLISECONDS)
                 .subscribe(o -> System.out.print(o + "\t"));
    ```

- distinct:去重

  - keySelector:这个函数根据原始Observable发射的数据项产生一个 Key，然后，比较这些Key而不是数据本身，来判定两个数据是否是不同的

    ```
      Observable.just(1, 2, 1, 2, 3)
                //这个函数根据原始Observable发射的数据项产生一个 Key，
                // 然后，比较这些Key而不是数据本身，来判定两个数据是否是不同的
                .distinct(integer -> Math.random())
                .subscribe(o -> System.out.print(o + "\t"));
    日志:
    原因 key不同 所以当做数据不同处理
    1	2	1	2	3
    ```

  - 无参版本 就是内部实现了的keySelector通过生成的key就是value本身

    ```
    Observable.just(1, 2, 1, 2, 3)
                 .distinct()
                 .subscribe(o -> System.out.print(o + "\t"));
     日志:
     1	2	3
    ```

- distinctUntilChanged(相邻去重):它只判定一个数据和它的直接前驱是 否是不同的。

  > 其他概念与distinct一样

- throttleWithTimeout/debounce:

> 操作符会过滤掉发射速率过快的数据项
> throttleWithTimeout/debounce： 含义相同
> 如果发送数据后 指定时间段内没有新数据的话 。则发送这条
> 如果有新数据 则以这个新数据作为将要发送的数据项，并且重置这个时间段，重新计时。

```
    Observable.create(e -> {
            e.onNext("onNext 0");
            Thread.sleep(100);
            e.onNext("onNext 1");
            Thread.sleep(230);
            e.onNext("onNext 2");
            Thread.sleep(300);
            e.onNext("onNext 3");
            Thread.sleep(400);
            e.onNext("onNext 4");
            Thread.sleep(500);
            e.onNext("onNext 5");
            e.onNext("onNext 6");
        })
                .debounce(330, TimeUnit.MILLISECONDS)
//                .throttleWithTimeout(330, TimeUnit.MILLISECONDS)
                .subscribeOn(Schedulers.newThread())
                .observeOn(Schedulers.newThread())
                .subscribe(o -> System.out.println(o));//结果 3 4 6
```

- filter:只发射通过了谓词测试的数据项

  ```
  Observable.range(0, 10)
          //过滤掉false的元素
          .filter(integer -> integer % 2 == 0)
          .subscribe(o -> System.out.print(o + "\t"));
  ```

- ofType:ofType 是 filter 操作符的一个特殊形式。它过滤一个Observable只返回指定类型的数据

```
Observable.just(0, "what?", 1, "String", 3)
             //ofType 是 filter 操作符的一个特殊形式。它过滤一个Observable只返回指定类型的数据。
             .ofType(String.class)
             .subscribe(o -> System.out.print(o + "\t"));
```

- first:只发射第一项(或者满足某个条件的第一项)数 感觉和take(1)  elementAt(0)差不多

```
Observable.range(0, 10)
          //如果元数据没有发送  则有发送默认值
          .first(-1)
          .subscribe(o -> System.out.print(o + "\t"));
```

- last:只发射最后一项(或者满足某个条件的最后一项)数据 感觉和takeLast(1)差不多

```
Observable.empty()
              //如果元数据没有发送  则有发送默认值
              .last(-1)
              .subscribe(o -> System.out.print(o + "\t"));
```

- sample/throttleLast: 周期采样后 发送最后的数据

- throttleFirst:周期采样 的第一条数据 发送

  > 注意: 如果是已经被发送过的 则不会继续发送

```
 Observable.create(e -> {
            e.onNext("onNext 0");
            Thread.sleep(100);
            e.onNext("onNext 1");
            Thread.sleep(50);
            e.onNext("onNext 2");
            Thread.sleep(70);
            e.onNext("onNext 3");
            Thread.sleep(200);
            e.onNext("onNext 4");
            e.onNext("onNext 5");
        })
                .subscribeOn(Schedulers.newThread())
                .observeOn(Schedulers.newThread())
                <!--  结果 : onNext 2	onNext 3	onNext 5	-->
                .sample(200, TimeUnit.MILLISECONDS,Schedulers.newThread())
                <!--  结果 : onNext 2	onNext 3	onNext 5	-->
//                .throttleLast(200, TimeUnit.MILLISECONDS,Schedulers.newThread())
                <!--  结果 : onNext 0	onNext 3	onNext 4	-->
//                .throttleFirst(200, TimeUnit.MILLISECONDS,Schedulers.newThread())
                .subscribe(o -> System.out.print(o + "\t"));
```

## 辅助操作

- repeat:不是创建一个Observable,而是重复发射原始,Observable的数据序列，这个序列或者是无限的，或者通过 repeat(n) 指定重复次数

```
Observable.just("Love", "For", "You!")
          .repeat(3)//重复三次
          .subscribe(s -> System.out.println(s));
```

- repeatUntil:getAsBoolean 如果返回 true则不repeat false则repeat.主要用于动态控制

```
Observable.just("Love", "For", "You!")
                .repeatUntil(new BooleanSupplier() {
                    @Override
                    public boolean getAsBoolean() throws Exception {
                        System.out.println("getAsBoolean");
                        count++;
                        if (count == 3)
                            return true;
                        else
                            return false;
                    }
                }).subscribe(s -> System.out.println(s));
```

- delay:延迟一段指定的时间再发射来自Observable的发射物

  > 注意：
  > delay 不会平移 onError 通知，它会立即将这个通知传递给订阅者，同时丢弃任何待 发射的 onNext 通知。
  > 然而它会平移一个 onCompleted 通知

```
Observable.range(0, 3)
          .delay(1400, TimeUnit.MILLISECONDS)
          .subscribe(o -> System.out.println("===>" + o + "\t"));
```

- delaySubscription:让你你可以延迟订阅原始Observable

```
Observable.just(1)
                .delaySubscription(2000, TimeUnit.MILLISECONDS)
                .subscribe(o -> System.out.println("===>" + o + "\t")
                        , throwable -> System.out.println("===>throwable")
                        , () -> System.out.println("===>complete")
                        , disposable -> System.out.println("===>订阅"));
```

- do系列

  - doOnEach:注册一个回调，它产生的Observable每发射一项数据就会调用它一次

    ```
    Observable.range(0, 3)
                  .doOnEach(integerNotification -> System.out.println(integerNotification.getValue()))
                  .subscribe(o -> System.out.print("===>" + o + "\t"));
      日志:
      doOnEach:
      doOnEach:0===>0
      doOnEach:1===>1
      doOnEach:2===>2
      doOnEach:null
    ```

  - doOnNext:注类似doOnEach 不是接受一个 Notification 参数，而是接受发射的数据项。

    ```
     Observable.range(0, 3)
             .doOnNext(integer -> {
                 if (integer == 2)
                     throw new Error("O__O");
                 System.out.print(integer);
             })
             .subscribe(o -> System.out.print("===>" + o + "\t")
                     , throwable -> System.out.print("===>throwable")
                     , () -> System.out.print("===>complete"));
    日志:
    0===>0	1===>1	===>throwable
    ```

  - doOnSubscribe:注册一个动作，在观察者订阅时使用

    ```
    Observable.range(0, 3)
                  .doOnSubscribe(disposable -> System.out.print("开始订阅"))
                  .subscribe(o -> System.out.print("===>" + o + "\t"));
        日志:
         开始订阅===>0	===>1	===>2
    ```

  - doOnComplete:注册一个动作，在观察者OnComplete时使用

    ```
    Observable.range(0, 3)
                   .doOnComplete(() -> System.out.print("doOnComplete"))
                   .subscribe(o -> System.out.print("===>" + o + "\t"));
       日志:
        ===>0	===>1	===>2	doOnComplete
    ```

  - doOnError:注册一个动作，在观察者doOnError时使用

    ```
    Observable.error(new Throwable("?"))
                   .doOnError(throwable -> System.out.print("throwable"))
                   .subscribe(o -> System.out.print("===>" + o + "\t"));
      日志:
      异常信息....
      throwable
    ```

  - doOnTerminate:注册一个动作，Observable终止之前会被调用，无论是正 常还是异常终止。

    ```
     Observable.range(0, 3)
             .doOnTerminate(() -> System.out.print("\t doOnTerminate"))
             .subscribe(o -> System.out.print("===>" + o + "\t"));
    日志:
    ===>0	===>1	===>2		 doOnTerminate
    ```

  - doFinally:注册一个动作,当它产生的Observable终止之后会被调用，无论是正常还 是异常终止。在doOnTerminate之后执行

    ```
    Observable.range(0, 3)
                    .doFinally(() -> System.out.print("\t doFinally"))
                    .doOnTerminate(() -> System.out.print("\t  doOnTerminate"))
                    .subscribe(o -> System.out.print("===>" + o + "\t"));
     日志:
     ===>0	===>1	===>2		  doOnTerminate	 doFinally
    ```

  - doOnDispose:注册一个动作,当【观察者取消】订阅它生成的Observable它就会被调

    > 注意:貌似需要在 为出现complete和error的时候 dispose才会触发 ~

    ```
    Disposable ab = Observable.interval(1, TimeUnit.SECONDS)
                  .take(3)
                  .doOnDispose(() -> System.out.println("解除订阅"))
                  .subscribe(o -> System.out.print("===>" + o + "\t"));
          ab.dispose();
    日志:
    解除订阅
    ```

- materialize:将数据项和事件通知都当做数据项发射

- dematerialize:materialize相反

```
Observable.range(0, 3)
                //将Observable转换成一个通知列表。
                .materialize()
                //与上面的作用相反，将通知逆转回一个Observable
                .dematerialize()
                .subscribe(o -> System.out.print("===>" + o + "\t"));
```

- observeOn:指定一个观察者在哪个调度器上观察这个Observable
- subscribeOn:指定Observable自身在哪个调度器上执行

> 注意 遇到错误 会立即处理而不是等待下游还没观察的数据
> 既onError 通知会跳到(并吞掉)原始Observable发射的数据项前面

```
Observable.range(0, 3)
              .subscribeOn(Schedulers.newThread())
              .observeOn(Schedulers.newThread())
              .subscribe(o -> System.out.print("===>" + o + "\t"));
```

- subscribe:操作来自Observable的发射物和通知

  ```
  Javadoc: subscribe()
  Javadoc: subscribe(onNext)
  Javadoc: subscribe(onNext,onError)
  Javadoc: subscribe(onNext,onError,onComplete)
  Javadoc: subscribe(onNext,onError,onComplete,onSubscribe)
  Javadoc: subscribe(Observer)
  Javadoc: subscribe(Subscriber)
  ```

- foreach:forEach 方法是简化版的 subscribe ，你同样可以传递一到三个函数给它，解释和传递给 subscribe 时一样

> 不同的是，你无法使用 forEach 返回的对象取消订阅。也没办法传递一个可以用于取消订阅 的参数

```
Observable.range(0, 3)
        //subscribe的简化版本  没啥用
        .forEach(o -> System.out.println("===>" + o + "\t"));
```

- serialize:保证上游下游同一线程 ，防止不同线程下 onError 通知会跳到(并吞掉)原始Observable发射的数据项前面的错误行为

  ```
  Observable.range(0, 3)
             .serialize()
             .subscribe(o -> System.out.print("===>" + o + "\t"));
  ```

- Timestamp:它将一个发射T类型数据的Observable转换为一个发射类型 为Timestamped 的数据的Observable，每一项都包含数据的原始发射时间

  ```
  Observable.interval(100, TimeUnit.MILLISECONDS)
                 .take(3)
                  .timestamp()
                  .subscribe(o -> System.out.println("===>" + o + "\t")
                          , throwable -> System.out.println("===> throwable")
                          , () -> System.out.println("===> complete")
                          , disposable -> System.out.println("===> 订阅"));
      日志:
      ===> 订阅
      ===>Timed[time=1501224256554, unit=MILLISECONDS, value=0]
      ===>Timed[time=1501224256651, unit=MILLISECONDS, value=1]
      ===>Timed[time=1501224256751, unit=MILLISECONDS, value=2]
      ===> complete
  ```

- timeInterval:一个发射数据的Observable转换为发射那些数据发射时间间隔的Observable

```
        Observable.interval(100, TimeUnit.MILLISECONDS)
                .take(3)
//         把发送的数据 转化为  相邻发送数据的时间间隔实体
                .timeInterval()
//                .timeInterval(Schedulers.newThread())
                .subscribe(o -> System.out.println("===>" + o + "\t")
                        , throwable -> System.out.println("===>throwable")
                        , () -> System.out.println("===>complete")
                        , disposable -> System.out.println("===>订阅"));
    日志:
    ===>订阅
    ===>Timed[time=113, unit=MILLISECONDS, value=0]
    ===>Timed[time=102, unit=MILLISECONDS, value=1]
    ===>Timed[time=97, unit=MILLISECONDS, value=2]
    ===>complete
```

- timeout

  - 变体:过了一个指定的时长仍没有发射数据(不是仅仅考虑第一个)，它会发一个错误

    ```
     Observable.interval(100, TimeUnit.MILLISECONDS)
    //        过了一个指定的时长仍没有发射数据(不是仅仅考虑第一个)，它会发一个错误
                    .timeout(50, TimeUnit.MILLISECONDS)
                    .subscribe(o -> System.out.println("===>" + o + "\t")
                            , throwable -> System.out.println("===>timeout throwable")
                            , () -> System.out.println("===>timeout complete")
                            , disposable -> System.out.println("===>timeout 订阅"));
    timeout:
    ===>timeout 订阅
    ===>timeout throwable
    ```

  - 变体 备用Observable:过了一个指定的时长仍没有发射数据(不是仅仅考虑第一个)，它会发一个错误

    ```
     Observable<Integer> other;
     Observable.empty()
             // 过了一个指定的时长仍没有发射数据(不是仅仅考虑第一个)，他会用备用Observable 发送数据，本身的会发送一个compelte
             .timeout(50, TimeUnit.MILLISECONDS, other = Observable.just(2, 3, 4))
             .subscribe(o -> System.out.println("===>" + o + "\t")
                     , throwable -> System.out.println("===>timeout2 throwable")
                     , () -> System.out.println("===>timeout2 complete")
                     , disposable -> System.out.println("===>timeout2 订阅"));
     other.subscribe(o -> System.out.println("k ===>" + o + "\t"));
    timeout2:
    ===>timeout2 订阅
    ===>timeout2 complete
    k ===>2
    k ===>3
    k ===>4
    ```

## 变换操作

- map:对Observable发射的每一项数据应用一个函数，执行变换操作,就是方形过渡到圆形

```
Observable.just(1,2)
              .map(integer -> "This is result " + integer)
              .subscribe(s -> System.out.println(s));
```

- flatMap: 将一个发射数据的Observable变换为多个Observables，然后将它们发射的数据合并后放进一个单独的Observable
  - mapper:根据发射数据映射成Observable
  - combiner: 用来合并 的

> 注意:FlatMap 对这些Observables发射的数据做的是合并( merge )操作，因此它们可能是交 错的。

```
     Observable.just(1, 2, 3)
               .flatMap(integer -> Observable.range(integer * 10, 2)
                       , (a, b) -> {
                           //a ： 原始数据的 just(1,2,3) 中的值
                           //b ： 代表 flatMap后合并发送的数据的值
                           System.out.print("\n a:" + a + "\t b:" + b);
                           //return flatMap发送的值 ，经过处理后 而发送的值
                           return a + b;
                       })
               .subscribe(s -> System.out.print("\t"+s));
日志：
 <!-- 这里有顺序是因为没有在其他线程执行 -->
a:1	 b:10	11
a:1	 b:11	12
a:2	 b:20	22
a:2	 b:21	23
a:3	 b:30	33
a:3	 b:31	34
```

- concatMap:类似FlatMap但是保证顺序 因为没有合并操作！

```
Observable.just(1, 2, 3)
            .concatMap(integer -> Observable.range(integer * 10, 2))
            .subscribe(s -> System.out.print("\t"+s));
```

- cast:在发射之前强制将Observable发射的所有数据转换为指定类型

```
Observable.just(1, 2, "string")
               .cast(Integer.class)//订阅之后才能发横强转
               .subscribe(integer -> System.out.println(integer)
                       , throwable -> System.out.println(throwable.getMessage()));
```

- groupBy:通过keySelector的apply的值当做key 进行分组,发射GroupedObservable(有getKey()方法)的group 通过group继续订阅取得其组内的值;
  - keySelector:通过这个的返回值 当做key进行分组
  - valueSelector:value转换

```
  Observable.range(0, 10)
                .groupBy(integer -> integer % 2, integer -> "(" + integer + ")")
                .subscribe(group -> {
                    group.subscribe(integer -> System.out.println(
                            "key:" + group.getKey() + "==>value:" + integer));
                });
日志：
key:0==>value:(0)
key:1==>value:(1)
key:0==>value:(2)
key:1==>value:(3)
key:0==>value:(4)
key:1==>value:(5)
key:0==>value:(6)
key:1==>value:(7)
key:0==>value:(8)
key:1==>value:(9)
```

- window: 依照此范例 每三秒收集,Observable在此时间内发送的值。组装成Observable发送出去。

```
Observable.interval(1, TimeUnit.SECONDS).take(7)
                //返回值  Observable<Observable<T>> 即代表 发送Observable<T>
                .window(3, TimeUnit.SECONDS)
                .subscribeOn(Schedulers.io())
                .observeOn(Schedulers.io())
                .subscribe(integerObservable -> {
                    System.out.println(integerObservable);
                    integerObservable.subscribe(integer -> System.out.println(integerObservable+"===>"+integer));
                });
日志:
为什么不是 345一起？ 因为会有太细微的时间差。例如5如果在多线程切换的时候是超过3秒的1毫秒则就尴尬了~
io.reactivex.subjects.UnicastSubject@531c3d1c
io.reactivex.subjects.UnicastSubject@531c3d1c===>0
io.reactivex.subjects.UnicastSubject@531c3d1c===>1
io.reactivex.subjects.UnicastSubject@531c3d1c===>2
io.reactivex.subjects.UnicastSubject@2ea0f969
io.reactivex.subjects.UnicastSubject@2ea0f969===>3
io.reactivex.subjects.UnicastSubject@2ea0f969===>4
io.reactivex.subjects.UnicastSubject@2d30de03
io.reactivex.subjects.UnicastSubject@2d30de03===>5
io.reactivex.subjects.UnicastSubject@2d30de03===>6
```

- scan:连续地对数据序列的每一项应用一个函数，然后连续发射结果

> 感觉就是发送一个有 累加(函数) 过程序列
>
> ```
> * initialValue（可选） 其实就是放到 原始数据之前发射。
> * a 原始数据的中的值
> * b 则是最后应用scan函数后发送的值
> ```

```
 Observable.just(1, 4, 2)
                //7是用来 对于第一次的 a的值
                .scan(7, (a, b) -> {
                    //b 原始数据的 just(1,4,2) 中的值
                    //a 则是最后应用scan 发送的值
                    System.out.format("a:%d * b:%d\n", a, b);
                    return a * b;
                })
                .subscribe(integer -> System.out.println("===>："+integer));
日志：
===>：7
a:7 * b:1
===>：7
a:7 * b:4
===>：28
a:28 * b:2
===>：56
```

- buffer系列

  - 变体 count系列

    ```
    * 范例:发射[1-10]
    * buffer count 2 skip 1,结果 [1,2]  [2,3] [3,4] 3=2*1+1
    * buffer count 2 skip 2,结果 [1,2]  [3,4] [5,6] 5=2*2+1
    * buffer count 2 skip 3,结果 [1,2]  [4,5] [7,8] 7=2*3+1;
    ```

    - count:缓存的数量

    - skip:每个缓存创建的间隔数量

      > 则代表 每次初始偏移量 每次真正的起始值=fistValue+skip*skipCount;
      > 注意skip不能小于0
      > 可以小于count这样就会导致每个发送的list之间的值会有重复
      > 可以大于count这样就会导致每个发送的list之间的值和原有的值之间会有遗漏
      > 可以等于count就你懂的了

    - bufferSupplier:自定义缓存装载的容器；

    ```
    Observable.range(1, 10)
            .buffer(2, 1,() -> new ArrayList<>())//有默认的装载器
            <!-- 其他方法 -->
            <!-- .buffer(2)//skip 默认和count一样 -->
            <!--  .buffer(2, () -> new ArrayList<>())-->
            .subscribe(integers -> System.out.println(integers));
    
     解析:每发射1个。创建一个发射物list buffer,每个buffer缓存2个,收集的存入list后发送。
    ```

  - 变体 time系列

    - timespan:缓存的时间
    - timeskip:每个缓存创建的间隔时间  同skip 可以小于大于等于timespan

    ```
      Observable.interval(500, TimeUnit.MILLISECONDS).take(7)
                    .buffer(3, 2, TimeUnit.SECONDS, Schedulers.single(),
                            Functions.createArrayList(16))
                    .subscribe(integers -> System.out.println(integers));
    
    解析:每两秒创建一个发射物list buffer,每个buffer缓存三秒 收集的存入list后发送。
    日志:
      [0, 1, 2, 3, 4]
      [4, 5, 6]
    ```

    - 变体 自定义buffer创建和收集时间

    - bufferOpenings:每当 bufferOpenings 发射了一个数据时，它就 创建一个新的 List,开始装入之后的发射数据

    - closingSelector:每当 closingSelector 发射了一个数据时,就结束装填数据 发射List。

      ```
          <!-- 范例和time系列的就一样了 -->
       Consumer<Long> longConsumer = aLong -> System.out.println("开始创建 bufferSupplier");
              Consumer<Long> longConsumer2 = aLong -> System.out.println("结束收集");
              Observable.interval(500, TimeUnit.MILLISECONDS).take(7)
      //                .doOnNext(aLong -> System.out.println("原始发射物：" + aLong))
                      .buffer(Observable.interval(2, TimeUnit.SECONDS)
                                      .startWith(-1L)//为了刚开始就发射一次
                                      .take(2)//多余的我就不创建了
                                      .doOnNext(longConsumer)
                              , aLong -> Observable.timer(3, TimeUnit.SECONDS)
                                      .doOnNext(longConsumer2)
                              , () -> new ArrayList<>())
                      .subscribe(integers -> System.out.println("buffer发射物" + integers));
      
      日志:
      openings:
      开始创建 bufferSupplier
      开始创建 bufferSupplier
      结束收集
      buffer发射物[0, 1, 2, 3, 4]
      buffer发射物[4, 5, 6]
      ```

  - 变体 仅仅bufer创建时间

    - boundarySupplier 因为发送一个值代表上个缓存的发送 和这个缓存的创建

    > 这个缓存是连续的, 因为发送一个值代表上个缓存的发送 和这个缓存的创建
    > 有发射物的时候 没缓存就创建了 就是 默认第一个发射物的时候由内部创建
    > 注意 如果不发送事件缓存 存满了 会自动发送出去的

    ```
    Observable.interval(500, TimeUnit.MILLISECONDS).take(7)
                    .buffer(() -> Observable.timer(2, TimeUnit.SECONDS)
                                    .doOnNext(aLong -> System.out.println("开始创建 bufferSupplier"))
                            , () -> new ArrayList<Object>())
                    .subscribe(integers -> System.out.println(integers));
    日志:
    开始创建 bufferSupplier
    [0, 1, 2]
    [3, 4, 5, 6]
    ```

## 合并操作符

- zip(静态方法):只有当原始的Observable中的每一个都发射了 一条数据时 zip 才发射数据。接受一到九个参数

```
Observable<Long> observable1 = Observable.interval(100, TimeUnit.MILLISECONDS)
                .take(3)
                .subscribeOn(Schedulers.newThread());
        Observable<Long> observable2 = Observable.interval(200, TimeUnit.MILLISECONDS)
                .take(4)
                .subscribeOn(Schedulers.newThread());
        Observable.zip(observable1, observable2, (aLong, aLong2) -> {
            System.out.print("aLong:" + aLong + "\t aLong2:" + aLong2+"\t");
            return aLong + aLong2;
        }).subscribe(o -> System.out.println("===>" + o + "\t"));

日志:
aLong:0	 aLong2:0===>0
aLong:1	 aLong2:1===>2
aLong:2	 aLong2:2===>4
```

- zipWith:zip的非静态写法,总是接受两个参数，第一个参数是一个Observable或者一个Iterable。

```
observable1.zipWith( observable2, (aLong, aLong2) -> {
    System.out.print("aLong:" + aLong + "\t aLong2:" + aLong2+"\t");
    return aLong + aLong2;
}).subscribe(o -> System.out.println("===>" + o + "\t"));
```

- merge(静态方法):根据时间线 合并多个observer

```
  Observable<Long> ob1 = Observable.interval(100, TimeUnit.MILLISECONDS)
                .take(3)
                .subscribeOn(Schedulers.newThread());

        Observable<Long> ob2 = Observable.interval(50, TimeUnit.MILLISECONDS)
                .take(3)
                .map(aLong -> aLong + 10)
                .subscribeOn(Schedulers.newThread());
        Observable.merge(ob1, ob2)
                .subscribe(o -> System.out.print(o + "\t"));
日志结果:可以见出是根据时间线合并
10	10	0	0	11	11	12	12	1	1	2	2
```

- mergeWith:merge非静态写法

```
ob1.mergeWith(ob2)
               .subscribe(o -> System.out.print( o + "\t"));
```

- combineLatest(静态方法):使用一个函数结合它们最近发射的数据，然后发射这个函数的返回值,它接受二到九个Observable作为参数 或者单 个Observables列表作为参数

```
Observable<Long> observable1 = Observable.interval(100, TimeUnit.MILLISECONDS)
              .take(4)
              .subscribeOn(Schedulers.newThread());
      Observable<Long> observable2 = Observable.interval(200, TimeUnit.MILLISECONDS)
              .take(5)
              .subscribeOn(Schedulers.newThread());
      Observable.combineLatest(observable1, observable2, (aLong, aLong2) -> {
          System.out.print("aLong:" + aLong + "\t aLong2:" + aLong2+"\t");
          return aLong + aLong2;
      }).subscribe(o -> System.out.println("===>" + o + "\t"));
  日志:
  aLong:1	 aLong2:0	===>1
  aLong:2	 aLong2:0	===>2
  aLong:3	 aLong2:0	===>3
  aLong:3	 aLong2:1	===>4
  aLong:3	 aLong2:2	===>5
  aLong:3	 aLong2:3	===>6
  aLong:3	 aLong2:4	===>7
```

- withLatestFrom:类似zip ,但是只在单个原始Observable发射了一条数据时才发射数据,而不是两个都发

> 但是注意 如果没有合并元素 既辅助Observable一次都没发射的时候 是不发射数据的

```
       Observable<Long> observable2 = Observable.interval(150, TimeUnit.MILLISECONDS)
                .take(4)
                .subscribeOn(Schedulers.newThread());
        Observable.interval(100, TimeUnit.MILLISECONDS)
                .take(3)
                .subscribeOn(Schedulers.newThread())
                .withLatestFrom(observable2, (aLong, aLong2) -> {
                    System.out.print("aLong:" + aLong + "\t aLong2:" + aLong2 + "\t");
                    return aLong + aLong2;
                })
                .subscribe(o -> System.out.println("===>" + o + "\t"));

日志:
明明原始take是3为啥不是三条log呢 因为原始的发送0的时候 ，辅助Observable还没发送过数据
aLong:1	 aLong2:0	===>1
aLong:2	 aLong2:1	===>3
```

- switchMap:和flatMap类似,不同的是当原始Observable发射一个新的数据（Observable）时，它将取消订阅前一个Observable

```
        Observable.interval(500, TimeUnit.MILLISECONDS)
                .take(3)
                .doOnNext(aLong -> System.out.println())
                .switchMap(aLong -> Observable.intervalRange(aLong * 10, 3,
                        0, 300, TimeUnit.MILLISECONDS)
                        .subscribeOn(Schedulers.newThread()))
                .subscribe(aLong -> System.out.print(aLong+"\t"));
解析：因为发送2的时候 intervalRange发送第三条数据的时候已经是600ms既 500ms的时候原始数据发送了。导致取消订阅前一个Observable
所以 2 ,12没有发送 但是最后的22发送了 因为原始数据没有新发送的了

//        日志结果
//        0	1
//        10	11
//        20	21	22
//        而不是
//        0     1   2
//        10	11  12
//        20	21  22
```

- startWith:是concat()的对应部分,在Observable开始发射他们的数据之前,startWith()通过传递一个参数来先发射一个数据序列

```
   Observable.just("old")
                  <!-- 简化版本 T item  -->
                  .startWith("Start")
                  <!--  多次应用探究 -->
                  .startWith("Start2")
                  <!--  observer -->
                  .startWith(Observable.just("Other Observable"))
                   <!--  Iterable -->
                  .startWith(Arrays.asList("from Iterable"))
                   <!--  T... -->
                  .startWithArray("from Array", "from Array2")
                  .subscribe(s -> System.out.println(s));
日志:
from Array
from Array2
from Iterable
Other Observable
Start2
Start
old
```

- join:任何时候，只要在另一个Observable发射的数据定义的时间窗口内，这个Observable发射了。一条数据，就结合两个Observable发射的数据

[![img](https://ww3.sinaimg.cn/large/006tKfTcgy1fhzely2zzjj31am0r2go8.jpg)](https://ww3.sinaimg.cn/large/006tKfTcgy1fhzely2zzjj31am0r2go8.jpg)

```
<!-- 此demo 好使但是未让能理解透彻  仅仅想测试能结果的任用  想明白的话 此demo无效 -->
Observable.intervalRange(10, 4, 0, 300, TimeUnit.MILLISECONDS)
           .join(Observable.interval(100, TimeUnit.MILLISECONDS)
                           .take(7)
                   , aLong -> {
                       System.out.println("开始收集："+aLong);
                       return Observable.just(aLong);
                   }
                   , aLong -> Observable.timer(200, TimeUnit.MILLISECONDS)
                   , (aLong, aLong2) -> {
                       System.out.print("aLong:" + aLong + "\t aLong2:" + aLong2 + "\t");
                       return aLong + aLong2;
                   }
           )
           .subscribe(aLong -> System.out.println(aLong));
```

## 条件操作

- all:判定是否Observable发射的所有数据都满足某个条件

  ```
   Observable.just(2, 3, 4)
                  .all(integer -> integer > 3)
                  .subscribe((aBoolean, throwable) -> System.out.println(aBoolean));
  日志：false
  ```

- amb:给定多个Observable，只让第一个发射数据的Observable发射全部数据

  - ambArray(静态方法):根据测试结果这个静态方法发射的最后一个

    ```
      Observable.ambArray(
                Observable.intervalRange(0, 3, 200, 100, TimeUnit.MILLISECONDS)
                , Observable.intervalRange(10, 3, 300, 100, TimeUnit.MILLISECONDS)
                , Observable.intervalRange(20, 3, 100, 100, TimeUnit.MILLISECONDS)
        )
                .doOnComplete(() -> System.out.println("Complete"))
                .subscribe(aLong -> System.out.println(aLong));
    日志：
    20  21  22  Complete
    ```

  - ambWith:这个发射原始的

    ```
     Observable.intervalRange(0, 3, 200, 100, TimeUnit.MILLISECONDS)
                    .ambWith(Observable.intervalRange(10, 3, 300, 100, TimeUnit.MILLISECONDS))
                    .doOnComplete(() -> System.out.println("Complete"))
                    .subscribe(aLong -> System.out.println(aLong));
    日志：
    0   1   2   Complete
    ```

- contains:判定一个Observable是否发射一个特定的值

  ```
  Observable.just(2, 3, 4)
                  .contains(2)
                  .subscribe((aBoolean, throwable) -> System.out.println(aBoolean));
  ```

- switchIfEmpty:如果原始Observable正常终止后仍然没有发射任何数据，就使用备用的Observable

  ```
  Observable.empty()
          .switchIfEmpty(Observable.just(2, 3, 4))
          .subscribe(o -> System.out.println("===>" + o + "\t")); //2,3,4
  ```

- defaultIfEmpty:发射来自原始Observable的值，如果原始Observable没有发射任何值，就发射一个默认值,内部调用的switchIfEmpty。

  ```
  Observable.empty()
             .defaultIfEmpty(1)
             .subscribe(o -> System.out.println("===>" + o + "\t")); //1
  ```

- sequenceEqual:判定两个Observables是否发射相同的数据序列。（数据，发射顺序，终止状态）

  ```
  Observable.sequenceEqual(
                  Observable.just(2, 3, 4)
                  , Observable.just(2, 3, 4))
                  .subscribe((aBoolean, throwable) -> System.out.println(aBoolean));
  
  
  <!-- 它还有一个版本接受第三个参数，可以传递一个函数用于比较两个数据项是否相同。 -->
  Observable.sequenceEqual(
          Observable.just(2, 3, 4)
          , Observable.just(2, 3, 4)
          , (integer, integer2) -> integer + 1 == integer2)
          .subscribe((aBoolean, throwable) -> System.out.println(aBoolean));
  ```

- skipUntil:丢弃原始Observable发射的数据，直到第二个Observable发射了一项数据

```
Observable.intervalRange(30, 20, 500, 100, TimeUnit.MILLISECONDS)
        .skipUntil(Observable.timer(1000, TimeUnit.MILLISECONDS))
        .doOnNext(integer -> System.out.println(integer))
        //此时用这个主要是 测试环境 有执行时间 所以用阻塞比较好
        .blockingSubscribe();
```

- skipWhile:丢弃Observable发射的数据，直到一个指定的条件不成立

```
Observable.just(1,2,3,4)
               //从2开始 因为2条件不成立
               .skipWhile(aLong -> aLong==1)
               .doOnNext(integer -> System.out.println(integer))
               //此时用这个主要是 测试环境 有执行时间 所以用阻塞比较好
               .blockingSubscribe();
```

- takeUntil:当第二个Observable发射了一项数据或者终止时，丢弃原始Observable发射的任何数据

```
<!-- 条件变体 -->
Observable.just(2,3,4,5)
             .takeUntil(integer ->  integer<=4)
             .subscribe(o -> System.out.print(o + "\t"));//2,3,4
<!-- Observable变体 -->
Observable.intervalRange(30, 20, 500, 100, TimeUnit.MILLISECONDS)
             .takeUntil(Observable.timer(1000, TimeUnit.MILLISECONDS))
             .doOnNext(integer -> System.out.println(integer))
             .doOnComplete(() -> System.out.println("Complete"))
             //此时用这个主要是 测试环境 有执行时间 所以用阻塞比较好
             .blockingSubscribe();
```

- takeWhile:发射Observable发射的数据，直到一个指定的条件不成立

```
Observable.just(2,3,4,5)
        .takeWhile(integer ->integer<=4 )
        .subscribe(o -> System.out.print(o + "\t"));//2,3
```

## 错误处理

- onErrorReturn:让Observable遇到错误时发射一个特殊的项并且正常终止

  ```
  <!-- 遇到错误处理范例 -->
  Observable.error(new Throwable("我擦 空啊"))
              .onErrorReturnItem("hei")
              .subscribe(o -> System.out.println("===>" + o + "\t")
                      , throwable -> System.out.println("===>throwable")
                      , () -> System.out.println("===>complete"));
  日志:
  ===>hei
  ===>complete
  
  <!--  遇到错误不处理范例 -->
    Observable.error(new Throwable("我擦 空啊"))
                  .onErrorReturn(throwable -> {
                      System.out.println("错误信息：" + throwable.getMessage());
                      return throwable;
                  })
                  .subscribe(o -> System.out.println("===>" + o + "\t")
                          , throwable -> System.out.println("===>throwable")
                          , () -> System.out.println("===>complete"));
  日志:
  错误信息：我擦 空啊
  ===>java.lang.Throwable: 我擦 空啊
  ===>complete
  ```

- resumeNext:让Observable在遇到错误时开始发射第二个Observable的数据序列

  - onErrorResumeNext:可以处理所有的错误

    ```
    Observable.error(new Throwable("我擦 空啊"))
                  .onErrorResumeNext(throwable -> {
                      System.out.println("错误信息：" + throwable.getMessage());
                      return Observable.range(0, 3);
                  })
             .subscribe(o -> System.out.print("===>" + o + "\t")
                               , throwable -> System.out.print("===>throwable"+ "\t")
                               , () -> System.out.print("===>complete"+ "\t"));
      日志：
      错误信息：我擦 空啊
      ===>0	===>1	===>2	===>complete
    ```

  - onExceptionResumeNext:只能处理异常。

    > Throwable 不是一个 Exception ,它会将错误传递给观察者的 onError 方法，不会使用备用 的Observable。

    ```
    <!-- Throwable不能处理范例 -->
    Observable.error(new Throwable("我擦 空啊"))
                  .onExceptionResumeNext(observer -> Observable.range(0, 3))
                  .subscribe(o -> System.out.println("===>" + o + "\t")
                          , throwable -> System.out.println("===>throwable")
                          , () -> System.out.println("===>complete"));
      日志:
      ===>throwable
      <!-- 正确演示范例 无效ing 求解答~ todo -->
    ```

- retry:如果原始Observable遇到错误，重新订阅它期望它能正常终止

  - 变体count 重复次数

    ```
    Observable.create(e -> {
         e.onNext(1);
         e.onNext(2);
         e.onError(new Throwable("hehe"));
     })
             .retry(2)
             .subscribe(o -> System.out.print("===>" + o + "\t")
                     , throwable -> System.out.print("===>throwable\t")
                     , () -> System.out.print("===>complete\t"));
     日志:
     ===>1	===>2	===>1	===>2	===>1	===>2	===>throwable
    ```

  - 变体Predicate 条件判定 如果返回 true retry,false 放弃 retry

    ```
    Observable.create(e -> {
         e.onNext(1);
         e.onNext(2);
         e.onError(new Throwable("hehe"));
     })
             .retry(throwable -> throwable.getMessage().equals("hehe1"))
             .subscribe(o -> System.out.print("===>" + o + "\t")
                     , throwable -> System.out.print("===>throwable\t")
                     , () -> System.out.print("===>complete\t"));
     日志:
    ===>1	===>2	===>throwable
    ```

- retryWhen: 需要一个Observable 通过判断 throwableObservable,Observable发射一个数据 就重新订阅，发射的是 onError 通知，它就将这个通知传递给观察者然后终止。

  ```
    <!-- 正常范例 -->
    Observable.just(1, "2", 3)
                  .cast(Integer.class)
                  <!-- 结果：1,1,complete 原因这个Observable发了一次数据 -->
                  .retryWhen(throwableObservable -> Observable.timer(1, TimeUnit.SECONDS))
                  <!-- 结果：1,1,1,1,complete 原因这个Observable发了三次数据 -->
                  .retryWhen(throwableObservable -> Observable.interval(1, TimeUnit.SECONDS)
                      .take(3))
                  .subscribe(o -> System.out.println("retryWhen 1===>" + o + "\t")
                          , throwable -> System.out.println("retryWhen 1===>throwable")
                          , () -> System.out.println("retryWhen 1===>complete"));
  
  
      <!-- 通过判断throwable 进行处理范例 -->
      Observable.just(1, "2", 3)
                  .cast(Integer.class)
                  .retryWhen(throwableObservable -> {
                      return throwableObservable.switchMap(throwable -> {
                          if (throwable instanceof IllegalArgumentException)
                              return Observable.just(throwable);
                              <!-- 这种方式OK -->
  //                        else{
  //                            PublishSubject<Object> pb = PublishSubject.create();
  //                            pb .onError(throwable);
  //                            return pb;
  //                        }
                          else
                              //方法泛型
                              return Observable.<Object>error(throwable);
                            <!-- 这种方式也OK -->
  //                        return Observable.just(1).cast(String.class);
                      });
                  })
                  .subscribe(o -> System.out.println("retryWhen 2===>" + o + "\t")
                          , throwable -> System.out.println("retryWhen 2===>throwable")
                          , () -> System.out.println("retryWhen 2===>complete"));
  日志:
  retryWhen 2===>1
  retryWhen 2===>throwable
  ```

## 阻塞操作

- toList

  ```
  Observable.just(1, 2, 3)
                .toList().blockingGet()
                .forEach(aLong -> System.out.println(aLong));
  ```

- toSortList

  ```
  Observable.just(5, 2, 3)
             .toSortedList()
             .blockingGet()
             .forEach(integer -> System.out.println(integer))
  ```

- toMap

  ```
       Map<String, Integer> map = Observable.just(5, 2, 3)
  //                .toMap(integer -> integer + "_")
                  //key 就是5_,value就是5+10   mapSupplier map提供者
                  .toMap(integer -> integer + "_"
                          , integer -> integer + 10
                          , () -> new HashMap<>())
                  .blockingGet();
  ```

- toFuture

  > 这个操作符将Observable转换为一个返 回单个数据项的 Future   带有返回值的任务
  > 如果原始Observable发射多个数据项， Future 会收到1个 IllegalArgumentException
  > 如果原始Observable没有发射任何数据， Future 会收到一 个 NoSuchElementException
  > 如果你想将发射多个数据项的Observable转换为 Future ,可以这样 用: myObservable.toList().toFuture()

```
Observable.just(1, 2, 3)
                   .toList()//转换成Single<List<T>> 这样就变成一个数据了
                   .toFuture()
                   .get()
                   .forEach(integer -> System.out.println(integer));
```

- blockingSubscribe

  ```
  Observable.just(1, 2, 3)
                .blockingSubscribe(integer -> System.out.println(integer));
  ```

- blockingForEach:对BlockingObservable发射的每一项数据调用一个方法，会阻塞直到Observable完成。

  ```
  Observable.interval(100, TimeUnit.MILLISECONDS)
          .doOnNext(aLong -> {
              if (aLong == 10)
                  throw new RuntimeException();
          }).onErrorReturnItem(-1L)
          .blockingForEach(aLong -> System.out.println(aLong));
  ```

- blockingIterable

  ```
     Observable.just(1, 2, 3)
                  .blockingIterable()
  //                .blockingIterable(5);
                  .forEach(aLong -> System.out.println("aLong:" + aLong));
  ```

- blockingFirst

  ```
  Observable.empty()
     // .blockingFirst();
     //带默认值版本
      .blockingFirst(-1));
  ```

- blockingLast：

  ```
  Observable.just(1,2,3)
        // .blockingLast();
        //带默认值版本
         .blockingLast(-1));
  ```

- blockingMostRecent:返回一个总是返回Observable最近发射的数据的Iterable,类似于while的感觉

  ```
   Iterable<Long> c = Observable.interval(100, TimeUnit.MILLISECONDS)
              .doOnNext(aLong -> {
                  if (aLong == 10)
                      throw new RuntimeException();
              }).onErrorReturnItem(-1L)
              .blockingMostRecent(-3L);
  for (Long aLong : c) {
              System.out.println("aLong:" + aLong);
          }
  日志很长 可以自己一试变知
  ```

- blockingSingle:

> 终止时只发射了一个值，返回那个值
>  empty   无默认值 报错， 默认值的话显示默认值
> 多个值的话  有无默认值都报错

```
System.out.println("emit 1 value:" + Observable.just(1).blockingSingle());
System.out.println("default empty single:" + Observable.empty().blockingSingle(-1));
System.out.println("default emit 1 value:" + Observable.just(1).blockingSingle(-1));
try {
    System.out.println("empty single:" + Observable.empty().blockingSingle());
    System.out.println("emit many value:" + Observable.just(1, 2).blockingSingle());
    System.out.println("default emit many value:" + Observable.just(1, 2)
            .blockingSingle(-1));
} catch (Exception e) {
    e.printStackTrace();
}
日志:
emit 1 value:1
default empty single:-1
default emit 1 value:1
java.util.NoSuchElementException
```

## 组合操作

- compose:有多个 Observable ，并且他们都需要应用`一组相同的 变换`

```
<!--  用一个工具类去写 这样符合单一职责 -->
//composes 工具类
public class RxComposes {

    public static <T> ObservableTransformer<T, T> applyObservableAsync() {
        return upstream -> upstream.subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread());
    }

}


  Observable.empty()
                .compose(RxComposes.applyObservableAsync())
                .subscribe(integer -> System.out.println("ob3:" + integer));
```

- ConnectableObservable：可连接的Observable在 被订阅时并不开始发射数据，只有在它的 connect() 被调用时才开始用这种方法，
  你可以 等所有的潜在订阅者都订阅了这个Observable之后才开始发射数据。即使没有任何订阅者订阅它，你也可以使用 connect 让他发射

  - replay(Observable的方法): 每次订阅 都对单个订阅的重复播放一边

    - bufferSize:对源发射队列的缓存数量, 从而对新订阅的进行发射；

    > Observable的方法 返回是ConnectableObservable
    > 切记要让ConnectableObservable具有重播的能力,必须Obserable的时候调用replay,而不是ConnectableObservable 的时候调用replay

    ```
    //this  is  OK,too!
         ConnectableObservable<Integer> co = Observable.just(1, 2, 3)
                 //类似 publish直接转成 ConnectableObservable  切记要重复播放的话必须Obserable的时候调用replay
                 //而不是ConnectableObservable 的时候调用replay 所以 .publish().replay()则无效
                 .replay(3);//重复播放的 是1  2  3
    //           .replay(2);//重复播放的 是 2  3
    
         co.doOnSubscribe(disposable -> System.out.print("订阅1："))
                 .doFinally(() -> System.out.println())
                 .subscribe(integer -> System.out.print(integer + "\t"));
         co.connect();//此时开始发射数据 不同与 refCount 只发送一次
    
         co.doOnSubscribe(disposable -> System.out.print("订阅2："))
                 .doFinally(() -> System.out.println())
                 .subscribe(integer -> System.out.print(integer + "\t"));
    
         co.doOnSubscribe(disposable -> System.out.print("订阅3："))
                 .doFinally(() -> System.out.println())
                 .subscribe(integer -> System.out.print(integer + "\t"));
    
    replay(3)日志：只能缓存原始队列的两个【1,2,3】
    订阅1：1	2	3
    订阅2：1	2	3
    订阅3：1	2	3
    
    replay(2)日志：只能缓存原始队列的两个【2,3】
    订阅1：1	2	3
    订阅2：	2	3
    订阅3：	2	3
    ```

  - publish(Observable的方法):将普通的Observable转换为可连接的Observable

    ```
    ConnectableObservable<Integer> co = Observable.just(1, 2, 3)
                   .publish();
    
           co.subscribe(integer -> System.out.println("订阅1：" + integer));
           co.subscribe(integer -> System.out.println("订阅2：" + integer));
           co.subscribe(integer -> System.out.println("订阅3：" + integer));
           co.connect();//此时开始发射数据
    ```

    - refCount(ConnectableObservable的方法): 操作符把从一个可连接的Observable连接和断开的过程自动化了, 就像reply的感觉式样 每次订阅 都对单个订阅的重复播放一边

      ```
       Observable<Integer> co = Observable.just(1, 2, 3)
                      .publish()
                      //类似于reply  跟时间线有关  订阅开始就开始发送
                      .refCount();
      
              co.doOnSubscribe(disposable -> System.out.print("订阅1："))
                      .doFinally(() -> System.out.println())
                      .subscribe(integer -> System.out.print(integer + "\t"));
              co.doOnSubscribe(disposable -> System.out.print("订阅2："))
                      .doFinally(() -> System.out.println())
                      .subscribe(integer -> System.out.print(integer + "\t"));
      
              Observable.timer(300, TimeUnit.MILLISECONDS)
                      .doOnComplete(() -> {
                          co.doOnSubscribe(disposable -> System.out.print("订阅3："))
                                  .doFinally(() -> System.out.println())
                                  .subscribe(integer -> System.out.print(integer + "\t"));
                      }).blockingSubscribe();
      日志:
      订阅1：1	2	3
      订阅2：1	2	3
      订阅3：1	2	3
      ```

## Subjects

Subject可以看成是一个桥梁或者代理，在某些ReactiveX实现中(如RxJava)，它同时充当  了Observer和Observable的角色。因为它是一个Observer，它可以订阅一个或多个  Observable;又因为它是一个Observable，它可以转发它收到(Observe)的数据，也可以发射 新的数据。

> 对我来说为什么用subjects呢？所有Subject都可以直接发射,不需要 发射器的引用 和 Observable.create()不同

- AsyncSubject:简单的说使用AsyncSubject无论输入多少参数，永远只输出最后一个参数。

  > 但是如果因为发生了错误而终止，AsyncSubject将不会发射任何数据，只是简单的向前传递这个错误通知。

```
AsyncSubject<Integer> source = AsyncSubject.create();

    source.subscribe(o -> System.out.println("1:"+o)); // it will emit only 4 and onComplete

    source.onNext(1);
    source.onNext(2);
    source.onNext(3);

     <!-- it will emit 4 and onComplete for second observer also. -->
    source.subscribe(o -> System.out.println("2:"+o));

    source.onNext(4);
    source.onComplete();
    日志:
    1:4
    2:4
```

- BehaviorSubject:会发送离订阅最近的上一个值，没有上一个值的时候会发送默认值。

> 如果原始的Observable因为发生了一个错误而终止，BehaviorSubject将不会发射任何 数据，只是简单的向前传递这个错误通知。

```
 BehaviorSubject<Integer> source = BehaviorSubject.create();
        //默认值版本
//        BehaviorSubject<Integer> source = BehaviorSubject.createDefault(-1);

        source.subscribe(o -> System.out.println("1:"+o)); // it will get 1, 2, 3, 4 and onComplete

        source.onNext(1);
        source.onNext(2);
        source.onNext(3);

        <!-- it will emit 3(last emitted), 4 and onComplete for second observer also. -->
        source.subscribe(o -> System.out.println("2:"+o));

        source.onNext(4);
        source.onComplete();
        日志:
        1:1
        1:2
        1:3
        2:3
        1:4
        2:4
```

- ```
  publishSubject
  ```

  (subject里最常用的):可以说是最正常的Subject，从那里订阅就从那里开始发送数据。

  > 如果原始的Observable因为发生了一个错误而终止，PublishSubject将不会发射任何数据，只 是简单的向前传递这个错误通知。

```
PublishSubject bs = PublishSubject.create();

        bs.subscribe(o -> System.out.println("1:"+o));
        bs.onNext(1);
        bs.onNext(2);
        bs.subscribe(o -> System.out.println("2:"+o));
        bs.onNext(3);
        bs.onComplete();
        bs.subscribe(o -> System.out.println("3:"+o));
        日志：
        1:1
        1:2
        1:3
        2:3
```

- replaySubject: 无论何时订阅，都会将所有历史订阅内容全部发出。

```
        ReplaySubject bs = ReplaySubject.create();

        bs.subscribe(o -> System.out.println("1:"+o));
// 无论何时订阅都会收到1，2，3
        bs.onNext(1);
        bs.onNext(2);
        bs.onNext(3);
        bs.onComplete();

        bs.subscribe(o -> System.out.println("2:"+o));
        日志:
        1:1
        1:2
        1:3
        2:1
        2:2
        2:3
```

## Single与Completable

参考:http://developer.51cto.com/art/201703/535298.htm

使用场景:其实这个网络请求并不是一个连续事件流，你只会发起一次 Get 请求返回数据并且只收到一个事件。我们都知道这种情况下 onComplete 会紧跟着 onNext 被调用，那为什么不把它们合二为一呢？

- Single:它总是只发射一个值，或者一个错误通知，而不是发射 一系列的值。因此，不同于Observable需要三个方法onNext, onError, onCompleted，订阅Single只需要两 个方法:

> Single只会调用这两个方法中的一个，而且只会调用一次，调用了任何一个方法之后，订阅关 系终止。

```
* onSuccess - Single发射单个的值到这个方法
* onError - 如果无法发射需要的值，Single发射一个Throwable对象到这个方法
<!--  retrofit 范例-->
 public interface APIClient {

     @GET("my/api/path")
     Single<MyData> getMyData();
 }


 apiClient.getMyData()
     .subscribe(new Consumer<MyData>() {
         @Override
         public void accept(MyData myData) throws Exception {
             // handle data fetched successfully and API call completed
         }
     }, new Consumer<Throwable>() {
         @Override
         public void accept(Throwable throwable) throws Exception{
             // handle error event
         }
     });


     <!-- 单独使用范例： -->
        Single.just("Amit")
            .subscribe(s -> System.out.println(s)
                    , throwable -> System.out.println("异常"));
```

使用场景:通过 PUT 请求更新数据 我只关心 onComplete 事件。使用 Completable 时我们忽略 onNext 事件，只处理 onComplete 和 onError 事件

- Completable:本质上来说和 Observable 与 Single 不一样，因为它不发射数据。

```
<!--  retrofit 范例-->
public interface APIClient {

    @PUT("my/api/updatepath")
    Completable updateMyData(@Body MyData data);
}

apiClient.updateMyData(myUpdatedData)
    .subscribe(new Action() {
        @Override
        public void run() throws Exception {
            // handle completion
        }
    }, new Consumer<Throwable>() {
        @Override
        public void accept(Throwable throwable) throws Exception{
            // handle error
        }
    });


    <!-- 单独使用范例： -->
     Completable.timer(1000, TimeUnit.MILLISECONDS)
                    .subscribe(() -> System.out.println("成功")
                            , throwable -> System.out.println("异常"));
```

- andThen( Completable中的方法最常用):在这个操作符中你可以传任何Observable、Single、Flowable、Maybe或者其他Completable，它们会在原来的 Completable 结束后执行

  ```
  apiClient.updateMyData(myUpdatedData)
      .andThen(performOtherOperation()) // a Single<OtherResult>
      .subscribe(new Consumer<OtherResult>() {
          @Override
          public void accept(OtherResult result) throws Exception {
              // handle otherResult
          }
      }, new Consumer<Throwable>() {
          @Override
          public void accept(Throwable throwable) throws Exception{
              // handle error
          }
      });
  ```

## 自定义操作符

- lift 原理图

[![img](https://ww4.sinaimg.cn/large/006tKfTcgy1fi33z4i9jlj30mi0gp74i.jpg)](https://ww4.sinaimg.cn/large/006tKfTcgy1fi33z4i9jlj30mi0gp74i.jpg)

```
@Test
    public void lift(){
            Observable.just(1,2)
                    //也是代理模式  observer是真正订阅
                    .lift(observer -> new Observer<Integer>() {
                        @Override
                        public void onSubscribe(Disposable d) {

                        }

                        @Override
                        public void onNext(Integer integer) {
                            observer.onNext(integer+"?");
                        }

                        @Override
                        public void onError(Throwable e) {

                        }

                        @Override
                        public void onComplete() {

                        }
                    })
                    .subscribe(o -> System.out.println(o));
    }

    日志:
    1?
    2?
```

## 实用技巧

flatMap 与 zip 配合的实用范例:

```
Observable.fromArray(new File("/Users/fuzhipeng/Documents"))
               .flatMap(file -> Observable.fromArray(file.listFiles()))
               //比较经典的 就是Observable.just(file) 把 file一个元素转成 observer从而进行zip合并的难题解决了
               .flatMap(file ->
                       Observable.zip(Observable.just(file)
                               , Observable.timer(1, TimeUnit.SECONDS)
                               , (file1, aLong) -> file1))
               .filter(file -> file.getName().endsWith(".png"))
               .take(5)
               .map(file -> file.getName())
               .subscribeOn(Schedulers.io())
               .observeOn(Schedulers.newThread())
               .subscribe(s -> System.out.println(s));
       while (true) {
       }
```

map的实用范例:

```
//有些服务几口设计，返回数据外层会包裹一些额外信息,可以使用map()吧外层格式剥掉
       Observable.just(1)
               .map(integer -> new Integer[]{1, 2, 3})
               .subscribe(integers -> System.out.println(integers));
```

方法泛型的实用范例:

```
Observable.just(1, "2", 3)
                .cast(Integer.class)
                .retryWhen(throwableObservable -> {
                    return throwableObservable.switchMap(throwable -> {
                        if (throwable instanceof IllegalArgumentException)
                            return Observable.just(throwable);
                        //todo  方法泛型 如果我不写<Object> 则会报错
                        return Observable.<Object>error(throwable);
                        //这个报错！！！
//                        return Observable.error(throwable);
                    });
                })
                .subscribe(o -> System.out.println("===>" + o + "\t")
                        , throwable -> System.out.println("===>throwable")
                        , () -> System.out.println("===>complete"));
```

BehaviorSubject的使用技巧:

> cache BehaviorSubject 是桥梁 并且有 发送最近的缓存特性！

```
BehaviorSubject<Object> cache = BehaviorSubject.create();
        Observable.timer(1,TimeUnit.SECONDS)
                .subscribeOn(Schedulers.newThread())
                .observeOn(Schedulers.newThread())
                .subscribe(cache);

        //可以想象成上面是方法  这里是方法被调用
        cache.subscribe(o -> System.out.println(o));//结果0
```

Observable 发射元素的封装范例:

```
//创建一个Observable 可以直接发送的 原因 获取rx内部方法需要final很恶心 所以...
        RxEmitter<Integer> emitter = new RxEmitter();
        Observable.create(emitter)
                .subscribe(integer -> System.out.println(integer));
        emitter.onNext(1);
        emitter.onNext(2);


public class RxEmitter<T> implements ObservableOnSubscribe<T>, ObservableEmitter<T> {

    ObservableEmitter<T> e;

    @Override
    public void subscribe(ObservableEmitter<T> e) throws Exception {
        this.e = e;
    }

    @Override
    public void onNext(T value) {
        e.onNext(value);
    }

    @Override
    public void onError(Throwable error) {
        e.onError(error);
    }

    @Override
    public void onComplete() {
        e.onComplete();
    }

    @Override
    public void setDisposable(Disposable d) {
        e.setDisposable(d);
    }

    @Override
    public void setCancellable(Cancellable c) {
        e.setCancellable(c);
    }

    @Override
    public boolean isDisposed() {
        return e.isDisposed();
    }

    @Override
    public ObservableEmitter<T> serialize() {
        return e.serialize();
    }

    @Override
    public boolean tryOnError(Throwable t) {
        return e.tryOnError(t);
    }
}
```

## Reference&Thanks：

https://www.gitbook.com/book/mcxiaoke/rxdocs/details

> 基本上所有的都参考此文档 很神！

http://blog.csdn.net/maplejaw_/article/details/52396175

http://developer.51cto.com/art/201703/535298.htm

http://gank.io/post/560e15be2dca930e00da1083

https://github.com/amitshekhariitbhu/RxJava2-Android-Samples

http://www.jianshu.com/u/c50b715ccaeb