===================================
   Java Promise快速入门
===================================

:Version: 5.0.6
:Date: 2022-9

.. default-role:: code
.. include:: rstcommon.rst
.. contents::

简介
======

异步与并发是业务编码过程中的一大难题，为了解决这个问题，在各种语言或库中都提供了异步任务控制机制。Java通过系统库的方式提供了对 Future 的支持。在 1.5 引入了 Future，在 JDK 8 引入了[CompletableFuture](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html)，进一步完善了 Future 机制。

  ```java
  CompletableFuture<HistoryData> tag = previousTag.thenCompose(this::toNow);
  return tag.thenCompose(hd -> {
      if (Objects.isNull(hd) || CollectionUtils.isEmpty(hd.getInfos())) {
          return getHistoryFromStore(hd);
      }
      return CompletableFuture.completedFuture(hd);
  });
  ```

不过，采用 Future 模式后，引入了一些新的复杂度，并且原有的部分痛点仍然没有解决，尤其是：

- 可能会从命令式编程(Imperative Programming)转换为声明式编程(Declarative Programming)，声明式编程通常并不直观，理解成本较高。
- 同步语义与异步语义的转换并不简单。Future 的API 有较高的学习成本。比如 CompletableFuture 各种组合的方法非常复杂。
- 错误处理仍然较复杂。 Future 通常包含 Running/Cancelled/Failed/Done 状态，部分状态又是通过异常来处理的，完整地处理错误仍然很冗长。
- Future 并不方便对中间状态进行测试。对于一个链状的 Future，通常不方便指定异步执行顺序，以此发现可能的并发错误。当然，这是所有并发编程的共同困难。

为进一步降低异步与并发控制的难度，我们借鉴其他语言的设计，在Java中实现Promise机制，即当执行一个异步任务时，许诺（Promise）在某个时间点会返回执行结果，这个表示未知结果的实体被称为Promise。使用Promise抽象可以大大降低开发者的心智负担，极大提高编码效率，降低出错可能性。

我们先来看一段使用Promise后的代码片段，针对计算密集型任务，实现多线程下计算任务。

  ```java
  Promise<Integer> promise = Promise.resolve(0);
  int i = 0;
  do {
      int c = i;
      promise = promise.next(n -> Promise.resolve(n + c));
  } while (++i <= 100);
  int val = promise.join();
  assertThat(val).isEqualTo(5050);
  ```

我们可以看到，与Javascript等弱类型语言相比，Java是强类型的。（`Promise<String>`应读作`Promise of String`）。使用泛型机制可以使代码编写起来更加安全，方便。另外Javascript不支持多线程，我们设计的Promise组件是天然支持多线程的。

上述例子，每一个累加的操作均在不同的线程执行，我们内置了两个简单通用的线程池，在默认情况下，Promise使用`ForkJoinPool`做为线程池，适合计算密集型任务。针对阻塞IO类型，可以使用内置的`Promise.ThreadPerTaskExecutor`这种简单的线程池模型。如果需要更复杂的策略，则可以使用自定义线程池，在构造函数中传入。

针对不同的场景，设计上我们把一个异步任务抽象成两种类型，一种是适合大多数场景，常规的需要立即执行的任务，我们称作`Promise`，另外一种是需要延迟执行的异步任务，我们称作[Deferred]。

Deferred
========

`Deferred`:idx: 是一个不立即执行的Promise，实现**数据消费**与**数据获取**的分离。

  ```java
  // 生成一个promise对象
  Deferred<String> deferred = new Deferred<>();
  //数据消费
  deferred.then(s -> {
      // s的值为something，此处对s进行处理
  });
  //数据获取
  deferred.offer((resolve, reject) -> {
      resolve.apply("something");
  });
  //以上两个步骤并没有先后顺序的要求，执行的结果是一致的
  //因为promise对象的状态是缓存的，减少很多因时序产生的问题。
  ```

通常情况下，数据的获取和消费通常是一起编写的，但仔细分析我们发现这其实两个完全不同的事情。

在工厂产线中，如果工人不但要负责加工商品（数据消费），还要负责去取待加工商品(数据获取)，这通常是小作坊的做法。而专业的流水线，每个工人，坐等流水线把商品送到面前，只需简单完成自己负责的工序即可。小作坊的方式加重了工人的心智负担，更容易出错，降低了流水线的效率。

`Deferred`帮助我们抽象了这个过程，完成了数据获取和数据处理这两个事情的分离，更好的履行了`单一职责(SRP)`的思想。

在实践中，比如一个函数接口返回一个String，代表函数调用者可以立即使用这个String，如果要表达**将来**要使用String的语义，就无能为力了。**Deferred**提供了这个语义，它表达的语义是**将来**会提供一个String，至于谁来提供这个String，怎么获取这个String，那完全是另外一回事，不需要数据处理者关心，数据处理者只需要知道如何处理这个数据并送入下一道工序即可。

由于**Deferred**也是一个**Promise**，因此完全具备Promise所具有的能力，Deferred相比Promise只是多了一个**Initialized**状态，如下图

  ```
                     +-------------------+                  
                     |                   |                  
     Deferred------->|    Initialized    |                  
                     |                   |                  
                     +---------+---------+                  
                               |                            
  +----------------------------+---------------------------+
  |                            |                           |
  |                  +---------v---------+                 |
  |                  |                   |                 |
  |   Promise------->|      Pending      |                 |
  |                  |                   |                 |
  |                  +---------+---------+                 |
  |                            |                           |
  |   +----------------+       |        +---------------+  |
  |   |                |       |        |               |  |
  |   |    Resovled    <-------+-------->    Rejected   |  |
  |   |                |                |               |  |
  |   +-------+--------+                +--------+------+  |
  |           |                                  |         |
  |           |        +---------------+         |         |
  |           |        |               |         |         |
  |           +-------->     Done      <---------+         |
  |                    |               |                   |
  |                    +---------------+                   |
  |                                                        |
  +--------------------------------------------------------+
  ```

核心接口
============

仅需几分钟，掌握以下核心接口的使用方法，就能令你在日常工作中事半功倍。

then
----

`then`:idx: 方法上一步异步操作成功时执行，返回一个新的Promise。

  ```java
  new Promise<String>((resolve, reject) -> {
       resolve.apply("something");
  }).then(s -> {
      // s is something
  	  // TODO 对something进行处理
  });
  ```

except
------

`except`:idx: 与then相反，在异步操作失败时执行，返回一个新的Promise。

  ```java
  new Promise<String>((resolve, reject) -> {
       reject.apply(result);
  }).except((error, s) -> {
      // s is something
  	// TODO 失败时的处理逻辑
  });
  ```

使用以上两个接口就可以实现一个简单的异步任务处理流程。在Android下，可以取代 [AsyncTask](https://developer.android.com/reference/android/os/AsyncTask)。

  ```java
  new Promise<String>((resolve, reject) -> {
  	//执行异步任务
  	//如成功，调用resolve改变状态
  	//如失败，调用reject改变状态
  }).then(s -> {
      //成功时的处理
  }).except((error, s) -> {
  	//失败时的处理
  });
  ```

当然由于`Promise的每个接口返回的都是一个新的Promise`你也可以这么写

  ```java
  new Promise<String>((resolve, reject) -> {
      //执行异步任务
  	//如成功，调用resolve改变状态
  	//如失败，调用reject改变状态
  }).except((error, o) -> {
  	//失败时的处理
  }).then(s -> {
  	//成功时的处理
  });
  ```


对于执行结果没有任何影响，到这里为止，你已经掌握了Promise的基本用法了。


done
----
`done`:idx: 在上一步异步操作状态确定时执行(成功或失败)，返回一个新的Promise。在一些场景下，我们并不关心异步任务是成功，还是失败，比如数据上报之类的，只是需要在任务执行完毕时进行一些操作。

  ```java
  new Promise<String>((resolve, reject) -> {
      //执行异步任务
  }).done(o -> {
      //异步任务完成，不管是成功还是失败
  });
  ```

> 注意:promise的状态一旦确定，就不会再改变自身的状态和其数据，这是跟状态机的区别。

由于Promise的每一个接口返回的都是一个新的Promise对象，因此允许我们进行链式操作，解决了常规基于回调编程带来的`回调地狱`问题。以上两个代码片段，均会返回一个新的Promise对象，其类型是 `Promise<Integer>`。
以上接口能够满足单个异步任务的处理流程，实际情况下异步任务往往有串行和并行的需求，Promise拥有强大的异步操作抽象描述能力，使我们可以对异步任务处理流程进行组合。

下面我们看一种常见的情形 **串行组合**，这里我们需要引入一个新的接口[next]，更高级的组合方式，请参考 [并发控制]

next
----

`next`:idx: 操作在上一步异步操作成功时执行，接收上一步的结果，发起下一个新的异步任务，返回该任务的Promise，可以做数据类型的转换。

  ```java
  new Promise<String>((resolve, reject) -> {
      resolve.apply("something");
  }).next(o -> new Promise<Integer>((resolve, reject) -> {
      resolve.apply(o.length());
  })).then(integer -> {
      // integer is 9
  });
  ```

在实际情况下，由于各种原因异步处理流程可能失败，因此我们最好加入异常处理分支。

  ```java
  new Promise<String>((resolve, reject) -> {
      resolve.apply("something");
  }).next(o -> new Promise<Integer>((resolve, reject) -> {
      resolve.apply(o.length());
  })).then(integer -> {
      // integer is 9
  }).except((error, o) -> {
      //失败时的处理流程
  });
  ```

timeout
-------

Promise A+规范里没有定义超时(`timeout`:idx: )的标准处理方式，但超时处理在实际业务环境下是非常普遍的，因此我们对规范进行扩展，内置了对超时的支持。
以下代码，如异步任务在1秒内没有完成(主动调用resolve或reject)，该Promise自动进入错误状态(即reject状态)。
> 延时操作，如定时器，一般是导致应用内存泄露问题的根源，如Promise在定时器结束之前扭转状态(resolve或reject)，则定时器会被自动取消，以免产生内存泄露。

  ```java
  new Promise<String>((resolve, reject) -> {
    // 须在指定时间内，调用resolve或者reject传递结果，否则自动扭转为reject状态
  }, Timeout.ofSeconds(1)).except((err, o) -> {
    // 错误处理流程
    // 如果1s内没有任务没有完成，则err为 PromiseTimeoutException
  }).then(s -> {
  	//成功时的处理
  });
  ```


以上就是Promise的核心接口。

实用接口
============

以下接口提供实践中经常需要用到的能力，此类接口均是对Promise核心API的包装。

delay
-----

我们可以非常简单的给任意一个Promise对象加入`delay`:idx: 延时执行的效果，可以替代`Thread.sleep`，此接口返回加入延迟效果的新Promise。

具体使用方法举例

  ```java
  new Promise<String>((resolve, reject) -> {
      resolve.apply("something")
  }).delay(Timeout.ofMillis(500)).then(o -> {
     // 延迟500ms收到结果
     // o is something
  })
  ```

或者静态方法的形式，本质一样的。

  ```java
  Promise.delay(aPromise, Timeout.ofMillis(delay))
  ```

也可以以延迟操作作为业务逻辑的起点。

  ```java
  Promise.delay(Timeout.ofMillis(500)).next(unused -> new Promise<String>((resolve, reject) -> {
     resolve.apply("something")
  })).then(o -> {
     // 延迟500ms收到结果
     // o is something
  });
  ```

recover
-------

可以非常简单的给任意一个Promise对象加入异常恢复逻辑`recover`:idx: 。在上一步异步操作失败时执行，接收上一步错误信息，执行新的获取`同类型数据`的异步任务，返回该任务的Promise。

常见的应用场景：
- 读取缓存失败，从网络中读取
- 网络错误，切换IP重试

此方法跟`next`接口有相似之处，区别如下
| 接口 | 执行时机 | 数据类型 |
| ------ | ------ | ------ |
| **next** | 上一步成功时执行 | **可以**返回与上一步**不同数据类型**的Promise |
| **recover** | 上一步失败时执行 | **必须**返回与上一步**相同数据类型**的Promise|

典型使用方法

  ```java
  new Promise<String>((resolve, reject) -> reject.apply("error"))
  .recover((error, o) -> new Promise<String>((resolve, reject) -> resolve.apply("something")))
  .then(s -> {
      // s is something
  })
  ```

或者静态方法的形式

  ```java
  Promise<String> aPromise = new Promise<>((resolve, reject) -> reject.apply("error"));
  //错误恢复
  Promise.recover(aPromise, (error, o) -> new Promise<>((resolve, reject) -> resolve.apply("something")))
          .then(s -> {
                // s is something  
          })
  ```

因为recover返回的也是一个新的Promise对象，可以实现非常灵活的恢复逻辑，比如多次尝试，或者加入延时等策略，请参考[retry]。

retry
-----

`retry`:idx: 简单来说就是以一定的策略，来执行recover的过程。我们可以非常简单的给一个异步任务加上重试策略。

  ```java
  int retryCountMax = 3;
  Promise.retry(() -> new Promise<>((Task<String, Object>) (resolve, reject) -> {
    	// 此任务最多会执行 retryCountMax 次，如成功则不再重试
  }), retryCountMax).then(s -> {
  	// 成功，对数据进行消费
  }).except((error, o) -> {
     // 进入此分支，说明3次重试均失败
  })
  ```

我们可以指定**重试策略**，比如没有网络就不要重试了，或以每次以不同的时间间隔重试，避免造成服务器压力。

  ```java
  int retryCountMax = 3;
  Promise.retry(() -> new Promise<>((Task<String, Object>) (resolve, reject) -> {
    	// 此任务最多会执行 retryCountMax 次，如成功则不再重试
  }), retryCountMax, new RetryStrategy() {
      @Override
      public Timeout delay(int retrySeq) {
  		// 指定每次重试的间隔
  		// 此处实现间隔1s,2s,3s...重试的效果
          return Timeout.ofSeconds((long) retrySeq);
      }
  
      @Override
      public boolean condition(int attemptsRemain, Throwable throwable, Object o) {
  	     // 该方法返回true，则继续重试，返回false中止重试。
  	     // 入参包含当前重试的次数，以及上一次重试的错误原因。
          return true;
      }
  }).then(s -> {
  	// 成功，对数据进行消费
  }).except((error, o) -> {
     // 进入此分支，说明3次重试均失败
  })
  ```

组件内部内置两种常用的重试策略，如果不指定重试策略参数，则使用默认**Default**重试策略，具体为`固定间隔1秒重试直到最大重试次数`。

  ```java
  Promise.retry(..., retryMaxCount);
  // 相当于
  Promise.retry(..., retryMaxCount,RetryStrategy.Default);
  ```

内置的两种重试策略具体实现如下，也可以扩展此接口以实现更适合业务的自定义策略。

  ```java
  public interface RetryStrategy {
      default Timeout delay(int retrySeq) {
          return Timeout.ofSeconds(1);
      }
      default boolean condition(int attemptsRemain, Throwable throwable, Object o) {
          return true;
      }
  
      RetryStrategy Default = new RetryStrategy(){};
      RetryStrategy Common = new RetryStrategy() {
          @Override
          public Timeout delay(int retrySeq) {
              return Timeout.ofSeconds(2L* retrySeq);//2,4,6,8 etc...
          }
      };
  }
  ```

同样的，该方法返回一个新的Promise对象。

> 注意，使用此接口须将重试次数控制在合理范围内，一般3至5次重试就能满足大多数需求。此接口不适用于长时间重试或无限重试的场景，建议使用`TimerTask`对Promise进行进一步封装，请参考如下例子。

  ```java
  /**
   * 固定的间隔重试某个任务，直到主动取消或超时为止，timeout设为null，代表该任务永不超时
   * @param task 需要执行的任务
   * @param periodMills 重试间隔时间，单位为毫秒
   * @param timeout 超时时间，可以为null，代表该任务永不超时
   **/
  public static <T> Promise<T> fixRateRepeat(Supplier<Promise<T>> task, long periodMills,  Timeout timeout) {
      Timer t = new Timer();
      TimerTask tt = new TimerTask() {
          @Override
          public void run() {
              task.get();
          }
      };
      Deferred<T> deferred = new Deferred<T>(timeout);
      if (timeout != null) {// timeout从offer方法调度后开始计算
          deferred.offer((resolve, reject) -> {});
      }
      deferred.done(o -> tt.cancel());
      t.scheduleAtFixedRate(tt, 0, periodMills);
      return deferred;
  }
  ```

validation
----------

`validation`:idx: 对数据进行进一步校验，返回代表校验结果的新Promise对象。

  ```java
  Promise.resolve("something")
          .validate(s -> s.length() == 9)
          .then(s -> {
               // 结果是合法的
           }).except((error, o) -> {
               // 如校验失败 error 类型为 ValidationFailedException
           });
  ```

我们可以把常用的数据校验规则收归并存储起来，随时使用，结合Promise实现更为强大效果

  ```java
  // 以下规则校验输入的字符串不能为空，且长度必须大于5
  Validation<String> ensureStrLengthIsFive = ((Validation<String>) o -> o != null).except(1001, "string is null")
          .andThen(((Validation<String>) s -> s.length() > 5).except(1002,"string length not corrent"));
  		
  Promise.resolve("hi").validate(ensureStrLengthIsFive).except((error, o) -> {
      if (error instanceof ValidationFailedException) {
          long errCode = ((ValidationFailedException) error).getErrorCode(); // 错误码 1002
          String errMsg = error.getMessage(); // 对应的错误提示
      }
  });
  
  Promise.resolve("something").validate(ensureStrLengthIsFive).then(s -> {
      // ok 满足要求，校验通过
  });
  ```

流程控制
============

join
----

`join`:idx: 把异步操作转换为同步。在某些场景，我们可能需要同步获取一个任务的执行执行结果。
在异步执行失败的情况下，该方法会抛出一个`PromiseException`，有以下可能的情况。

- `PromiseCompleteException`
异步任务执行失败，即异步任务调用了**reject.apply**方法。
- `PromiseTimeoutException`
异步任务执行超时，**每个promise都可以指定一个超时时间**，没有在规定的时间内完成状态扭转，则抛出此异常。
- `PromiseException`
异步任务执行过程中发生的其它异常，如NP等。**Promise在执行过程中发生的异常都会被捕捉**，确保不会引起程序崩溃。

  ```java
  String s = new Promise<String>((resolve, reject) -> {
      // 异步任务
  }).join();
  //如果没有发生异常，在这里对s进行处理
  ```

abort
-----

`abort`:idx: 放弃任务，其执行结果取决于当前Promise的状态。对于已经处于 **Done** 状态的Promise，无任何效果，否则立即扭转为Reject态。

传统的Promise并没有支持取消的接口，TC39官方[曾经有个提案](https://github.com/tc39/proposal-cancelable-promises) 但已经被否决 。
目前官方提供的解决方案是使用[AbortController]( https://developer.mozilla.org/en-US/docs/Web/API/AbortController/abort) 与Promise配合实现取消功能。实际使用过程中对单个Promised对象尚可接受，但在链式调用中，我们很难知道异步任务执行到了什么阶段，应该取消那个Promise对象，甚至取不到Promise对象的引用。

除此之外，也有使用`Promise.race`接口包装的一个可取消的promise对象的方案，比如 [SF上的一个讨论](https://stackoverflow.com/questions/30233302/promise-is-it-possible-to-force-cancel-a-promise)。笔者认为此方案使用起来相对繁琐，代码可读性也不高。 

在我们的实现中，取消一个Promise非常简单，只需要调用Promise对象的abort 方法即可，一次调用，可以轻松取消掉整个Promise链，顺便实现了 [结构化并发](https://en.wikipedia.org/wiki/Structured_concurrency)的效果。

  ```java
  Promise<Void> promise = Promise.delay(Timeout.ofMillis(2000), null);
  promise.except((error, o) -> {
      // error类型为 AbortException
  }).done(o -> {
      // o类型为PromiseException，getCause返回对象类型为 AbortException
  });
  promise.abort(); // 任务取消
  ```

在abort接口被调用后，abort事件会在整个异步任务链上传播，取消掉整个任务链。比如上述代码中，Promise被取消后，2000ms的定时器资源如果没有申请，那么不再申请，如果已经申请了，该定时器资源会被自动释放。

开发者也可以通过覆写`onAbort`方法，实现资源释放的善后工作，如下代码示例演示与OKHttp的结合。

  ```java
  AbortController controller = new AbortController();
  Promise<Integer> promise = new Promise<Integer>((resolve, reject) -> {
  	// 注册清理资源的操作
  	controller.signal.onAbort(reason -> {
  		// 这里需要考虑多线程资源竞争，因为发送HTTP请求与取消HTTP请求可能不在同一个线程
  		OkHttpUtils.cancelCall(client);
      });
  	// 发送HTTP请求（同步方法，耗时操作）
  	Response response = client.newCall(request).execute()
  	// 使用资源
  	resolve.apply(response.code());
  }) {
      @Override
      public void onAbort(String reason) {
          controller.abort(reason);
      }
  }.except((error, o) -> {
      // error类型为 AbortException
  }).done(o -> {
      // o类型为PromiseException，getCause返回对象类型为 AbortException
  });
  
  promise.abort();// 任务取消
  ```

在Kotlin里使用的例子。

  ```
  // 创建一个 AbortController 对象
  val abortController = AbortController()
  
  // 创建一个Promise对象，并执行
  val promise = object : Promise<String>(Task<String, Any> { resolve, _ ->
      val t1 = Thread.currentThread()
      abortController.signal.onAbort {
          t1.interrupt()
      }
  	// 模拟耗时操作
      try {
          Thread.sleep(2000)
      } catch (e: InterruptedException) {
          Log.d(TAG, "InterruptedException: $e")
      }
      resolve.apply("SUCCESS")
  }, Timeout.ofMillis(3000)) {
      override fun onAbort(reason: String?) {
          abortController.abort(reason)
      }
  }.then {
      Log.d(TAG, "then")
  }.except { throwable: Throwable?, any: Any? ->
      Log.d(TAG, "except $throwable $any")
  }.done { 
      Log.d(TAG, "done")
  }
  
  // 在需要时中止 Promise 的执行
  promise.abort()
  ```


并发控制
============

all
---
`all`:idx: 接受任意个Promise对象，并发执行异步任务。全部任务成功，有一个失败则视为整体失败。

race
----
`race`:idx: 接受任意个Promise对象，并发执行异步任务。时间是第一优先级，多个任务以最先返回的那个结果为准，此结果成功即为整体成功，失败则为整体失败。

any
---
`any`:idx: 接受任意个Promise对象，并发执行异步任务。等待其中一个成功即为成功，全部任务失败则进入错误状态，输出错误列表。

allSettled
----------
`allSettled`:idx: 任务优先，所有任务必须执行完毕，永远不会进入失败状态。

详情可参阅 [MDN Promise API文档](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise) ，具体实现是一致的。

内存安全
============

异步任务和多线程并发是业务处理过程中的一大难题，稍有不慎便会出现崩溃，死锁，变量同步等问题。使用Promise技术可以安全高效的解决上述两大难题。

首先，Promise解决了异步数据流管理问题，另外Promise每个异步任务是天然多线程执行的。在可以并发的情况下，会并发执行以最大化提升程序的运行效率，例如`all`, `race`, `any`等接口，这些接口抽象了常用的并发模型。

这里如果展开来讲的话，篇幅会比较长，这里只是做个简略的说明，使用Promise模式进行异步编程，开发者一般不需要关心此类问题。


其他说明
============

使用要求
------------
不依赖任何第三方库，具体要求如下

- Java平台: Java版本>=1.8
- 安卓平台: Android版本>=24 (Nougat)

安卓平台可以通过[android-retrofuture](https://github.com/retrostreams/android-retrofuture) 支持更旧的安卓版本（尚未实现）。

资源
------------
- Future/Promise 有细微的不同，具体见(https://en.wikipedia.org/wiki/Futures_and_promises)
- Google实现了Swift版本的Promise库(https://github.com/google/promises)
- [RxJava](https://github.com/ReactiveX/RxJava) 一种响应式编程，基于异步数据流概念的编程模式。
- Promise规范 [Promise A+](https://promisesaplus.com/)。
- [MDN Promise API文档](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise)。
- [Promise A+规范](https://promisesaplus.com/)
