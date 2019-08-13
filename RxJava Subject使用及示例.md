本文介绍RxJava中Subject的使用。小白同学们看清楚并记好了，这里讲的是Subject，不是Subscribe，不是Subscription，不是subscribeOn，不是OnSubscribe，不是Schedulers，更不是Single，而是**Subject**！

这么多S开头的的单词有没有把你弄糊涂啊，英语好的同学可以略过这段。这里把RxJava中容易和Subject弄混的概念都拿出来介绍一下，还是那句话，看不明白没关系，以后还会讲的！

| 概念        | 解释                                                         |
| :---------- | :----------------------------------------------------------- |
| Subject     | 翻译为主题、科目。可以想象成杂志中的板块吧....               |
| Subscribe   | 订阅。可以想象成订阅杂志啊...                                |
| OnSubscribe | 当订阅的时候要做的事情。可以想象为当有用户订阅的时候开始写文章.... |
| subscribeOn | 做事情所在的线程                                             |
| Schedulers  | 调度器。上节有讲，用于指定在哪个线程工作，与subscribeOn配套使用 |
| Single      | 单一的输入输出异步任务                                       |

------

RxJava中常见的Subject有4种，分别是** AsyncSubject**、** BehaviorSubject**、** PublishSubject**、** ReplaySubject**。

> 值得注意的是一定要用Subcect.create()的方式创建并使用，不要用just(T)、from(T)、create(T)创建，否则会导致失效...

### AsyncSubject

简单的说使用AsyncSubject无论输入多少参数，永远只输出最后一个参数。



![AS img](https://mcxiaoke.gitbooks.io/rxdocs/content/images/S.AsyncSubject.png)

AS img

```
// 无论订阅的时候AsyncSubject是否Completed，均可以收到最后一个值的回调
AsyncSubject as = AsyncSubject.create();
as.onNext(1);
as.onNext(2);
as.onNext(3);
// 这里订阅收到3
as.onCompleted();
// 结束后，这里订阅也能收到3
as.subscribe(
        new Action1<Integer>() {
            @Override
            public void call(Integer o) {
                LogHelper.e("S:" + o);// 这里只会输出3
            }
        });

// 不要这样使用Subject
AsyncSubject.just(1, 2, 3).subscribe(
        new Action1<Integer>() {
            @Override
            public void call(Integer o) {
                // 这里会输出1， 2， 3
                LogHelper.e("S:" + o);
            }
        });
// 因为just(T)、from(T)、create(T)会把Subject转换为Obserable
```

但是如果因为发生了错误而终止，AsyncSubject将不会发射任何数据，只是简单的向前传递这个错误通知。



![AS error img](https://mcxiaoke.gitbooks.io/rxdocs/content/images/S.AsyncSubject.e.png)

AS error img

### BehaviorSubject

BehaviorSubject会发送离订阅最近的上一个值，没有上一个值的时候会发送默认值。看图



![BS img](https://mcxiaoke.gitbooks.io/rxdocs/content/images/S.BehaviorSubject.png)

BS img



如果遇到错误会直接中断



![BS error img](https://mcxiaoke.gitbooks.io/rxdocs/content/images/S.BehaviorSubject.e.png)

BS error img

```
// 注意订阅时机，以下这个例子收不到回调
BehaviorSubject bs = BehaviorSubject.create(-1);
// 这里订阅回调-1, 1, 2, 3
bs.onNext(1);
// 这里订阅回调1, 2, 3
bs.onNext(2);
// 这里订阅回调2, 3
bs.onNext(3);
// 这里订阅回调3
bs.onCompleted();
// 这里订阅没回调
bs.subscribe(
        new Action1<Integer>() {
            @Override
            public void call(Integer o) {
                LogHelper.e("S:" + o);
            }
        });
```

### PublishSubject

可以说是最*正常*的Subject，从那里订阅就从那里开始发送数据。
 

![PS img](https://mcxiaoke.gitbooks.io/rxdocs/content/images/S.PublishSubject.png)

PS img



```
PublishSubject bs = PublishSubject.create();
// 这里订阅接收1， 2， 3
bs.onNext(1);
// 这里订阅接收2， 3
bs.onNext(2);
// 这里订阅接收3
bs.onNext(3);
bs.onCompleted();
// 这里订阅无接收
bs.subscribe(
        new Action1<Integer>() {
            @Override
            public void call(Integer o) {
                LogHelper.e("S:" + o);
            }
        });
```

### ReplaySubject

无论何时订阅，都会将所有历史订阅内容全部发出。

```
ReplaySubject bs = ReplaySubject.create();
// 无论何时订阅都会收到1，2，3
bs.onNext(1);
bs.onNext(2);
bs.onNext(3);
bs.onCompleted();
bs.subscribe(
        new Action1<Integer>() {
            @Override
            public void call(Integer o) {
                LogHelper.e("S:" + o);
            }
        });
```

### 总结

总的来说Subject没法指定异步线程，更像是EventBus通过订阅来实现事件通知。