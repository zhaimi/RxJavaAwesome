这篇文章中，会展示一个Java9中FlowAPI的列子，通过Publisher和Subscriber接口来构建响应式程序。最后你将会理解这种全新的编程模式和她的优缺点。所有的代码可在Github上下载。

------

### Java9 Flow API介绍

#### JDK9响应式编程

Java是一个“古老”并且广泛应用的编程语言，但Java9中引入了一些新鲜有趣的特性。这篇文章主要介绍FlowAPI这个新特性，通过FlowAPI我们仅仅使用JDK就能够搭建响应式应用程序，而不需要其他额外的类库，如RxJava或Project Reactor。

尽管如此，当你看到过接口文档后你就会明白到正如字面所说，这只是一个API而已。她仅仅包含了一些Interface和一个实现类：

- Interface `Flow.Publisher<T>`定义了生产数据和控制事件的方法。
- Interface `Flow.Subscriber<T>`定义了消费数据和事件的方法。
- Interface `Flow.Subscription` 定义了链接Publisher和Subscriber的方法。
- Interface `Flow.Processor<T,R>`定义了转换Publisher到Subscriber的方法
- 最后，class `SubmissionPublisher<T>`是`Flow.Publisher<T>`的实现，她可以灵活的生产数据，同时与[Reactive Stream](https://link.jianshu.com?t=http%3A%2F%2Fwww.reactive-streams.org%2F)兼容。

虽然Java9中没有很多FlowAPI的实现类可供我们使用，但是依靠这些接口第三方可以提供的响应式编程得到了规范和统一，比如从JDBC driver到RabbitMQ的响应式实现。

#### Pull，Push，Pull-Push

我对响应式编程的理解是， 这是一种数据消费者控制数据流的编程方式。需要指出是，当消费速度低于生产速度时，消费者要求生产者降低速度以完全消费数据（这个现象称作back-pressure）。这种处理方式不是在制造混乱，你可能已经使用过这种模式，只是最近因为在主要框架和平台上使用才变得更流行，比如Java9，Spring5。另外在分布式系统中处理大规模数据传输时也使用到了这种模式。

回顾过去可以帮我们更好的理解这种模式。几年前，最常见的消费数据模式是pull-based。client端不断轮询服务端以获取数据。这种模式的优点是当client端资源有限时可以更好的控制数据流（停止轮询），而缺点是当服务端没有数据时轮询是对计算资源和网络资源的浪费。

随着时间推移，处理数据的模式转变为push-based，生产者不关心消费者的消费能力，直接推送数据。这种模式的缺点是当消费资源低于生产资源时会造成缓冲区溢出从而数据丢失，当丢失率维持在较小的数值时还可以接受，但是当这个比率变大时我们会希望生产者降速以避免大规模数据丢失。

响应式编程是一种pull-push混合模式以综合他们的优点，这种模式下消费者负责请求数据以控制生产者数据流，同时当处理资源不足时也可以选择阻断或者丢弃数据，接下来我们会看到一个典型案例。

### Flow与Stream

响应式编程并不是为了替换传统编程，其实两者相互兼容而且可以互相协作完成任务。Java8中引入的StreamAPI通过map，reduce以及其他操作可以完美的处理数据集，而FlowAPI则专注于处理数据的流通，比如对数据的请求，减速，丢弃，阻塞等。同时你可以使用Streams作为数据源（publisher），当必要时阻塞丢弃其中的数据。你也可以在Subscriber中使用Streams以进行数据的归并操作。更值得一提的时reactive streams不仅兼容传统编程方式，而且还支持函数式编程以极大的提高可读性和可维护性。

有一点可能会使我们感到困惑：如果你需要在两个系统间传输数据，同时进行转形操作，如何使用Flows和Streams来完成？这种情况下，我们使用Java8的Function来做数据转换，但是如何在Publisher和Subscriber之间使用StreamAPI呢？答案是我们可以在Publisher和Subscriber之间再加一个subscriber，她可以从最初的publisher获取数据，转换，然后再作为一个新的publisher，而使最初的subscriber订阅这个新的publisher，也是Java9中的接口`Flow.Processor<T,R>`，我们只需要实现这个接口并编写转换数据的functions。

从技术上讲，我们完全可以使用Flows来替换Streams，但任何时候都这么做就显得过于偏激。比如，我们创建一个Publisher来作为int数组的数据源，然后在Processor中转换Integer为String，最后创建一个Subscriber来归并到一个String中。这个时候就完全没有必要使用Flows，因为这不是在控制两个模块或两个线程间的数据通信，这个时候使用Streams更为合理。

### 一个杂志出版商的使用场景



![img](https:////upload-images.jianshu.io/upload_images/6361044-2fc31b82fd657416.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/780)

image



本文中给出的示例代码是以杂志出版商为模型。假设出版商有两个订阅客户。

出版商将为每个订阅客户出版20本杂志。出版商知道他们的客户有时在邮递杂志时会不在家，而当他们的邮箱（subscriber buffer）不巧被塞满时邮递员会退回或丢弃杂志，出版商不希望出现这种情况。

于是出版商发明了一个邮递系统：当客户在家时再给出版商致电，出版商会立即邮递一份杂志。出版商打算在办公室为每个客户保留一个小号的邮箱以防当杂志出版时客户没有第一时间致电获取。出版商认为为每个客户预留一个可以容纳8份杂志的邮件已经足够（publisher buffer）。

于是一名员工提出了以下不同的场景：

1. 如果客户请求杂志足够迅速，将不会存在邮箱容量的问题。
2. 如果客户没有以杂志出版的速度发出请求，那么邮箱将被塞满。这位员工提出以下几种处理方案：
    a. 增加邮箱容量，为每位客户提供可容纳20份杂志的邮箱。（publisher增加buffer）
    b. 直到客户请求下一份杂志之前停止印刷，并且根据客户请求的速度降低印刷速度以清空邮箱。
    c. 新的杂志直接丢掉。
    d. 一个折中的方案： 如果邮箱满了，在下次打印之前等待一段时间，如果还是没有足够的空间则丢弃新的杂志。

出版商无法承受花费过多的资源仅仅是因为一个速度慢的客户，那将是巨大的浪费，最终选择了方案d，最大程度上减少客户损失。

本文示例代码中选用了方案d是因为如果我们使用了一个虚拟的无穷buffer，这对理解Reactive模式的中概念是不利的，代码也将变得过于简易，无法与其他方案进行比较，接下来让我们来看代码吧。

### Java9 Flow 代码示例

#### 一个简单的Subscriber（full-control of Flow）

从订阅者开始，`MagazineSubscriber`实现了`Flow.Subscriber<Integer>`，订阅者将收到一个数字，但请假设这是一份杂志正如上面的使用场景提到的。

```java
package com.thepracticaldeveloper;

import java.util.concurrent.Flow;
import java.util.stream.IntStream;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class MagazineSubscriber implements Flow.Subscriber<Integer> {

  public static final String JACK = "Jack";
  public static final String PETE = "Pete";

  private static final Logger log = LoggerFactory.
    getLogger(MagazineSubscriber.class);

  private final long sleepTime;
  private final String subscriberName;
  private Flow.Subscription subscription;
  private int nextMagazineExpected;
  private int totalRead;

  MagazineSubscriber(final long sleepTime, final String subscriberName) {
    this.sleepTime = sleepTime;
    this.subscriberName = subscriberName;
    this.nextMagazineExpected = 1;
    this.totalRead = 0;
  }

  @Override
  public void onSubscribe(final Flow.Subscription subscription) {
    this.subscription = subscription;
    subscription.request(1);
  }

  @Override
  public void onNext(final Integer magazineNumber) {
    if (magazineNumber != nextMagazineExpected) {
      IntStream.range(nextMagazineExpected, magazineNumber).forEach(
        (msgNumber) ->
          log("Oh no! I missed the magazine " + msgNumber)
      );
      // Catch up with the number to keep tracking missing ones
      nextMagazineExpected = magazineNumber;
    }
    log("Great! I got a new magazine: " + magazineNumber);
    takeSomeRest();
    nextMagazineExpected++;
    totalRead++;

    log("I'll get another magazine now, next one should be: " +
      nextMagazineExpected);
    subscription.request(1);
  }

  @Override
  public void onError(final Throwable throwable) {
    log("Oops I got an error from the Publisher: " + throwable.getMessage());
  }

  @Override
  public void onComplete() {
    log("Finally! I completed the subscription, I got in total " +
      totalRead + " magazines.");
  }

  private void log(final String logMessage) {
    log.info("<=========== [" + subscriberName + "] : " + logMessage);
  }

  public String getSubscriberName() {
    return subscriberName;
  }

  private void takeSomeRest() {
    try {
      Thread.sleep(sleepTime);
    } catch (InterruptedException e) {
      throw new RuntimeException(e);
    }
  }
}
```

class中实现了必要的方法如下：

-  `onSubscriber(subscription)` Publisher在被指定一个新的Subscriber时调用此方法。 一般来说你需要在subscriber内部保存这个subscrition实例，因为后面会需要通过她向publisher发送信号来完成：请求更多数据，或者取消订阅。 一般在这里我们会直接请求第一个数据，正如代码中所示。
-  `onNext(magazineNumber)` 每当新的数据产生，这个方法会被调用。在我们的示例中，我们用到了最经典的使用方式：处理这个数据的同时再请求下一个数据。然而我们在这中间添加了一段可配置的sleep时间，这样我们可以尝试订阅者在不同场景下的表现。剩下的一段逻辑判断仅仅是记录下丢失的杂志（当publisher出现丢弃数据的时候）。
-  `onError(throwable)` 当publisher出现异常时会调用subscriber的这个方法。在我们的实现中publisher丢弃数据时会产生异常。
-  `onComplete()` 当publisher数据推送完毕时会调用此方法，于是整个订阅过程结束。

#### 通过Java9 SubmissionPublisher发送数据

我们将使用Java9 `SubmissionPublisher`类来创建publisher。正如javadoc所述， 当subscribers消费过慢，就像Reactive Streams中的Publisher一样她会阻塞或丢弃数据。在深入理解之前让我们先看代码。

```java
package com.thepracticaldeveloper;

import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.SubmissionPublisher;
import java.util.concurrent.TimeUnit;
import java.util.stream.IntStream;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class ReactiveFlowApp {

  private static final int NUMBER_OF_MAGAZINES = 20;
  private static final long MAX_SECONDS_TO_KEEP_IT_WHEN_NO_SPACE = 2;
  private static final Logger log =
    LoggerFactory.getLogger(ReactiveFlowApp.class);

  public static void main(String[] args) throws Exception {
    final ReactiveFlowApp app = new ReactiveFlowApp();

    log.info("\n\n### CASE 1: Subscribers are fast, buffer size is not so " +
      "important in this case.");
    app.magazineDeliveryExample(100L, 100L, 8);

    log.info("\n\n### CASE 2: A slow subscriber, but a good enough buffer " +
      "size on the publisher's side to keep all items until they're picked up");
    app.magazineDeliveryExample(1000L, 3000L, NUMBER_OF_MAGAZINES);

    log.info("\n\n### CASE 3: A slow subscriber, and a very limited buffer " +
      "size on the publisher's side so it's important to keep the slow " +
      "subscriber under control");
    app.magazineDeliveryExample(1000L, 3000L, 8);

  }

  void magazineDeliveryExample(final long sleepTimeJack,
                               final long sleepTimePete,
                               final int maxStorageInPO) throws Exception {
    final SubmissionPublisher<Integer> publisher =
      new SubmissionPublisher<>(ForkJoinPool.commonPool(), maxStorageInPO);

    final MagazineSubscriber jack = new MagazineSubscriber(
      sleepTimeJack,
      MagazineSubscriber.JACK
    );
    final MagazineSubscriber pete = new MagazineSubscriber(
      sleepTimePete,
      MagazineSubscriber.PETE
    );

    publisher.subscribe(jack);
    publisher.subscribe(pete);

    log.info("Printing 20 magazines per subscriber, with room in publisher for "
      + maxStorageInPO + ". They have " + MAX_SECONDS_TO_KEEP_IT_WHEN_NO_SPACE +
      " seconds to consume each magazine.");
    IntStream.rangeClosed(1, 20).forEach((number) -> {
      log.info("Offering magazine " + number + " to consumers");
      final int lag = publisher.offer(
        number,
        MAX_SECONDS_TO_KEEP_IT_WHEN_NO_SPACE,
        TimeUnit.SECONDS,
        (subscriber, msg) -> {
          subscriber.onError(
            new RuntimeException("Hey " + ((MagazineSubscriber) subscriber)
              .getSubscriberName() + "! You are too slow getting magazines" +
              " and we don't have more space for them! " +
              "I'll drop your magazine: " + msg));
          return false; // don't retry, we don't believe in second opportunities
        });
      if (lag < 0) {
        log("Dropping " + -lag + " magazines");
      } else {
        log("The slowest consumer has " + lag +
          " magazines in total to be picked up");
      }
    });

    // Blocks until all subscribers are done (this part could be improved
    // with latches, but this way we keep it simple)
    while (publisher.estimateMaximumLag() > 0) {
      Thread.sleep(500L);
    }

    // Closes the publisher, calling the onComplete() method on every subscriber
    publisher.close();
    // give some time to the slowest consumer to wake up and notice
    // that it's completed
    Thread.sleep(Math.max(sleepTimeJack, sleepTimePete));
  }

  private static void log(final String message) {
    log.info("===========> " + message);
  }

}
```

`magazineDeliveryExample`中我们为两个不同的subscribers设置了两个不同的等待时间， 并且设置了缓存容量`maxStorageInPO`
 步骤如下：

1. 创建`SubmissionPublisher`并设置一个标准的线程池（每个subscriber拥有一个线程）
2. 创建两个subscribers，通过传递变量设置不同的消费时间和不同的名字，以在log中方便区别
3. 用20个数字的的stream数据集作为数据源以扮演“杂志打印机”，我们调用`offer`，并传递以下变量：
    a. 提供给subscribers的数据。
    b. 第二和第三个变量是等待subscribers获取杂志的最大时间。
    c. 控制器以处理数据丢弃的情况。这里我们抛出了一个异常，返回false意味着告诉publisher不需要重试。
4. 当丢弃数据发生时，`offer`方法返回一个负数，否则将返回publisher的最大容量（以供最慢的subscriber消费），同时打印这个数字。
    5 . 最后我们添加了一个循环等待以防止主进程过早结束。这里一个是等待publisher清空缓存数据，另外等待最慢的subscriber收到`onComplete`回调信号（`close()`调用之后）

`main()`方法中使用不同参数调用以上逻辑三次，以模拟之前介绍的三种不同真是场景。

1. 消费者消费速度很快，publisher缓存区不会发生问题。
2. 其中一个消费者速度很慢，以至缓存被填满，然而缓存区足够大以容纳所有所有数据，不会发生丢弃。
3. 其中一个消费者速度很慢，同时缓存区不够大，这是控制器被出发了多次，subscriber没有收到所有数据。

你还可以尝试其他组合，比如设置`MAX_SECONDS_TO_WAIT_WHEN_NO_SPACE`为很大的数字，这时`offer`表象将类似于`submit`，或者可以尝试将两个消费者速度同时降低（会出现大量丢弃数据)。