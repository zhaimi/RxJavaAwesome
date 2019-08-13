### 1. 作用

- 对事件序列中的事件 / 整个事件序列 进行**加工处理**（即变换），使得其转变成不同的事件 / 整个事件序列
- 具体原理如下

![img](https://upload-images.jianshu.io/upload_images/944365-45e3d263d098c4ee.png)


### 2. 类型

- `RxJava`中常见的变换操作符如下：

  ![img](https:////upload-images.jianshu.io/upload_images/944365-71eb569b296c1f18.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000)

- 下面，我将对每种操作符进行详细介绍

> 注：本文只讲解`RxJava2`在开发过程中常用的变换操作符

------

### 3. 应用场景 & 对应操作符 介绍

- 下面，我将对 `RxJava2` 中的变换操作符进行逐个讲解
- 注：在使用`RxJava 2`操作符前，记得在项目的`Gradle`中添加依赖：

```groovy
dependencies {
      compile 'io.reactivex.rxjava2:rxandroid:2.0.1'
      compile 'io.reactivex.rxjava2:rxjava:2.0.7'
      // 注：RxJava2 与 RxJava1 不能共存，即依赖不能同时存在
}
```

#### 3.1 Map（）

- 作用
   对 被观察者发送的每1个事件都通过 **指定的函数** 处理，从而变换成另外一种事件

> 即， **将被观察者发送的事件转换为任意的类型事件。**

- 原理

![img](https://upload-images.jianshu.io/upload_images/944365-a9c0b5eb2cc573d6.png)

示意图

- 应用场景
   数据类型转换

- 具体使用
   下面以将 使用`Map（）` 将事件的参数从 **整型** 变换成  **字符串类型** 为例子说明

![img](https://upload-images.jianshu.io/upload_images/944365-dc0e57fb348f6eab.png)


```java
 // 采用RxJava基于事件流的链式操作
        Observable.create(new ObservableOnSubscribe<Integer>() {

            // 1. 被观察者发送事件 = 参数为整型 = 1、2、3
            @Override
            public void subscribe(ObservableEmitter<Integer> emitter) throws Exception {
                emitter.onNext(1);
                emitter.onNext(2);
                emitter.onNext(3);

            }
            // 2. 使用Map变换操作符中的Function函数对被观察者发送的事件进行统一变换：整型变换成字符串类型
        }).map(new Function<Integer, String>() {
            @Override
            public String apply(Integer integer) throws Exception {
                return "使用 Map变换操作符 将事件" + integer +"的参数从 整型"+integer + " 变换成 字符串类型" + integer ;
            }
        }).subscribe(new Consumer<String>() {

            // 3. 观察者接收事件时，是接收到变换后的事件 = 字符串类型
            @Override
            public void accept(String s) throws Exception {
                Log.d(TAG, s);
            }
        });
```

- 测试结果
![img](https://upload-images.jianshu.io/upload_images/944365-d86cc16df735b4ff.png)

从上面可以看出，`map()` 将参数中的 `Integer` 类型对象转换成一个 `String`类型 对象后返回

> 同时，事件的参数类型也由 `Integer` 类型变成了 `String` 类型


#### 3.2 FlatMap（）

- 作用：将被观察者发送的事件序列进行 **拆分  & 单独转换**，再合并成一个新的事件序列，最后再进行发送
- 原理

1. 为事件序列中每个事件都创建一个 `Observable` 对象；
2. 将对每个 原始事件 转换后的 新事件 都放入到对应 `Observable`对象；
3. 将新建的每个`Observable` 都合并到一个 新建的、总的`Observable` 对象；
4. 新建的、总的`Observable` 对象 将 新合并的事件序列 发送给观察者（`Observer`）

![img](https://upload-images.jianshu.io/upload_images/944365-a6f852c071db2f15.png)

- 应用场景
   无序的将被观察者发送的整个事件序列进行变换
- 具体使用

```
// 采用RxJava基于事件流的链式操作
        Observable.create(new ObservableOnSubscribe<Integer>() {
            @Override
            public void subscribe(ObservableEmitter<Integer> emitter) throws Exception {
                emitter.onNext(1);
                emitter.onNext(2);
                emitter.onNext(3);
            }

            // 采用flatMap（）变换操作符
        }).flatMap(new Function<Integer, ObservableSource<String>>() {
            @Override
            public ObservableSource<String> apply(Integer integer) throws Exception {
                final List<String> list = new ArrayList<>();
                for (int i = 0; i < 3; i++) {
                    list.add("我是事件 " + integer + "拆分后的子事件" + i);
                    // 通过flatMap中将被观察者生产的事件序列先进行拆分，再将每个事件转换为一个新的发送三个String事件
                    // 最终合并，再发送给被观察者
                }
                return Observable.fromIterable(list);
            }
        }).subscribe(new Consumer<String>() {
            @Override
            public void accept(String s) throws Exception {
                Log.d(TAG, s);
            }
        });
```

  ![img](https:////upload-images.jianshu.io/upload_images/944365-b019ab94b9b10359.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000)


> 注：新合并生成的事件序列顺序是无序的，即 与旧序列发送事件的顺序无关

#### 3.3 ConcatMap（）

- 作用：类似`FlatMap（）`操作符
- 与`FlatMap`（）的 区别在于：**拆分 & 重新合并生成的事件序列 的顺序 = 被观察者旧序列生产的顺序**
- 原理

![img](https://upload-images.jianshu.io/upload_images/944365-f4340f283e5a954d.png)
- 应用场景
   有序的将被观察者发送的整个事件序列进行变换
- 具体使用

```
// 采用RxJava基于事件流的链式操作
        Observable.create(new ObservableOnSubscribe<Integer>() {
            @Override
            public void subscribe(ObservableEmitter<Integer> emitter) throws Exception {
                emitter.onNext(1);
                emitter.onNext(2);
                emitter.onNext(3);
            }

            // 采用concatMap（）变换操作符
        }).concatMap(new Function<Integer, ObservableSource<String>>() {
            @Override
            public ObservableSource<String> apply(Integer integer) throws Exception {
                final List<String> list = new ArrayList<>();
                for (int i = 0; i < 3; i++) {
                    list.add("我是事件 " + integer + "拆分后的子事件" + i);
                    // 通过concatMap中将被观察者生产的事件序列先进行拆分，再将每个事件转换为一个新的发送三个String事件
                    // 最终合并，再发送给被观察者
                }
                return Observable.fromIterable(list);
            }
        }).subscribe(new Consumer<String>() {
            @Override
            public void accept(String s) throws Exception {
                Log.d(TAG, s);
            }
        });
```

- 测试结果
  ![img](https://upload-images.jianshu.io/upload_images/944365-7ff08c4f84945fa8.png)

> 注：新合并生成的事件序列顺序是有序的，即 严格按照旧序列发送事件的顺序

#### 3.4 Buffer（）

- 作用
   定期从 被观察者（`Obervable`）需要发送的事件中 获取一定数量的事件 & 放到缓存区中，最终发送
- 原理
![img](https://upload-images.jianshu.io/upload_images/944365-5278a339e4337494.png)

- 应用场景
   缓存被观察者发送的事件

- 具体使用
   那么，`Buffer（）`每次是获取多少个事件放到缓存区中的呢？下面我将通过一个例子来说明

```
// 被观察者 需要发送5个数字
        Observable.just(1, 2, 3, 4, 5)
                .buffer(3, 1) // 设置缓存区大小 & 步长
                                    // 缓存区大小 = 每次从被观察者中获取的事件数量
                                    // 步长 = 每次获取新事件的数量
                .subscribe(new Observer<List<Integer>>() {
                    @Override
                    public void onSubscribe(Disposable d) {

                    }
                    @Override
                    public void onNext(List<Integer> stringList) {
                        //
                        Log.d(TAG, " 缓存区里的事件数量 = " +  stringList.size());
                        for (Integer value : stringList) {
                            Log.d(TAG, " 事件 = " + value);
                        }
                    }

                    @Override
                    public void onError(Throwable e) {
                        Log.d(TAG, "对Error事件作出响应" );
                    }

                    @Override
                    public void onComplete() {
                        Log.d(TAG, "对Complete事件作出响应");
                    }
                });
```

- 测试结果
![img](https:////upload-images.jianshu.io/upload_images/944365-f1d4e320b7c62dd9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000)

- 过程解释

下面，我将通过一个图来解释`Buffer（）`原理 & 整个例子的结果

![img](https://upload-images.jianshu.io/upload_images/944365-33a49ffd2ec60794.png)

至此，关于`RxJava2`中主要的变换操作符已经讲解完毕

### 4. 实际开发需求案例

- 变换操作符的主要开发需求场景 = 嵌套回调（`Callback hell`）
- 下面，我将采用一个实际应用场景实例来讲解嵌套回调（`Callback hell`）

> 具体请看文章[Android RxJava 实际应用案例讲解：网络请求嵌套回调](https://www.jianshu.com/p/5f5d61f04f96)

------

### 5. Demo地址

上述所有的`Demo`源代码都存放在：[Carson_Ho的Github地址：RxJava2_变换操作符](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2FCarson-Ho%2FRxJava_Operators)


### 6. 总结

- 下面，我将用一张图总结 `RxJava2` 中常用的变换操作符

![img](https:////upload-images.jianshu.io/upload_images/944365-dc0a7df673324e21.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000)

示意图

