在Rxjava2中，Observale和Flowable都是用来发射数据流的，但是，我们在实际应用中，很多时候，需要发射的数据并不是数据流的形式，而只是一条单一的数据，或者一条完成通知，或者一条错误通知。在这种情况下，我们再使用Observable或者Flowable就显得有点大材小用，于是，为了满足这种单一数据或通知的使用场景，便出现了Observable的简化版——Single、Completable、Maybe。

##### Single

只发射一条单一的数据，或者一条异常通知，不能发射完成通知，其中数据与通知只能发射一个。

##### Completable

只发射一条完成通知，或者一条异常通知，不能发射数据，其中完成通知与异常通知只能发射一个

##### Maybe

可发射一条单一的数据，以及发射一条完成通知，或者一条异常通知，其中完成通知和异常通知只能发射一个，发射数据只能在发射完成通知或者异常通知之前，否则发射数据无效。

#### 示例一：Single发射单一数据



![img](https:////upload-images.jianshu.io/upload_images/6773051-b89a6ae482b9ccc0.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/689)

singleDemo1.jpg

#### 示例二：Single发射异常通知



![img](https:////upload-images.jianshu.io/upload_images/6773051-8f66edf2acb2c74d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/679)

singleDemo2.jpg

#### SingleEmitter：Single的发射器

可观察对象Single的发射器接口SingleEmitter中，

1、方法void onSuccess(T t)用来发射一条单一的数据，且一次订阅只能调用一次，不同于Observale的发射器ObservableEmitter中的void onNext(@NonNull T value)方法，在一次订阅中，可以多次调用多次发射。
 2、方法void onError(Throwable t)等同于ObservableEmitter中的void onError(@NonNull Throwable error)用来发射一条错误通知
 3、SingleEmitter中没有用来发射完成通知的void onComplete()方法。
 方法onSuccess与onError只可调用一个，若先调用onError则会导致onSuccess无效，若先调用onSuccess,则会抛出io.reactivex.exceptions.UndeliverableException异常。

#### SingleObserver：Single观察者

可观察对象Single对应的观察者为SingleObserver



![img](https:////upload-images.jianshu.io/upload_images/6773051-7c4ad0b3294a40f8.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/319)

SingleObserver.jpg



方法void onSubscribe(Disposable d)等同于Observer中的void onSubscribe(Disposable d)。
 方法void onSuccess(T t)类似于Observer中的onNext(T t)用来接收Single发的数据。
 方法void onError(Throwable e)等同于Observer中的void onError(Throwable e)用来处理异常通知。
 没有用来处理完成通知的方法void onComplete()

#### 示例三：Completable发射完成通知



![img](https:////upload-images.jianshu.io/upload_images/6773051-f5530a7511486a63.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/656)

completableDemo1.jpg

#### 示例四：Completable发射异常通知



![img](https:////upload-images.jianshu.io/upload_images/6773051-d6674f0bf34ad6ed.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/666)

completableDemo2.jpg

#### CompletableEmitter：Completable的发射器

可观察对象Completable的发射器接口CompletableEmitter中
 1、方法onComplete()等同于Observale的发射器ObservableEmitter中的onComplete()，用来发射完成通知。
 2、方法onError(Throwable e)等同于ObservableEmitter中的onError(Throwable e)，用来发射异常通知。
 方法onComplete与onError只可调用一个，若先调用onError则会导致onComplete无效，若先调用onComplete,则会抛出io.reactivex.exceptions.UndeliverableException异常。

#### CompletableObserver：Completable的观察者

可观察对象Completable对应的观察者为CompletableObserver



![img](https:////upload-images.jianshu.io/upload_images/6773051-4ad7d4ba49e7a36f.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/348)

completableObserver.jpg



其中的三个方法均等同于Observer中的对应的方法，CompletableObserver中没有用来发射数据的方法。

#### 示例五：Maybe发射单一数据和完成通知



![img](https:////upload-images.jianshu.io/upload_images/6773051-6aa0917523aba677.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/669)

maybeDemo1.jpg

#### 示例六：Maybe发射单一数据和异常通知



![img](https:////upload-images.jianshu.io/upload_images/6773051-809233732d0e471c.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/681)

maybeDemo2.jpg

#### MaybeEmitter：Maybe的发射器

可观察对象Maybe的发射器接口MaybeEmitter中
 1、方法void onSuccess(T t)用来发射一条单一的数据，且一次订阅只能调用一次，不同于Observale的发射器ObservableEmitter中的void onNext(@NonNull T value)方法，在一次订阅中，可以多次调用多次发射。
 2、方法void onError(Throwable t)等同于ObservableEmitter中的void onError(@NonNull Throwable error);用来发射一条错误通知
 3、方法onComplete()等同于Observale的发射器ObservableEmitter中的onComplete()，用来发射完成通知。
 方法onComplete与onError只可调用一个，若先调用onError则会导致onComplete无效，若先调用onComplete,则会抛出io.reactivex.exceptions.UndeliverableException异常。
 方法onSuccess必须在方法onComplete或onError之前调用，否则会导致方法onSuccess调用无效。

#### MaybeObserver：Maybe的观察者

可观察对象Maybe对应的观察者为MaybeObserver





![img](https:////upload-images.jianshu.io/upload_images/6773051-dbb29d060a16e6ce.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/350)

MaybeObserver.jpg


 方法void onSubscribe(Disposable d)等同于Observer中的void onSubscribe(Disposable d)。 方法void onSuccess(T t)类似于Observer中的onNext(T t)用来接收Single发的数据。 方法void onError(Throwable e)等同于Observer中的void onError(Throwable e)用来处理异常通知。 方法void onComplete()等同于Observer中的void onComplete()用来接收完成通知。