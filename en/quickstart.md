===================================
Getting Started
===================================

:Version: 5.0.6
:Date: 2022-09

.. default-role:: code
.. include:: rstcommon.rst
.. contents::

Introduction
============

Asynchronous and concurrency are a major problem in programming. In order to solve this problem, asynchronous task control mechanisms are provided in various languages or libraries. Java provides support for Future through standard library. `Future` was introduced in 1.5, and [CompletableFuture](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html) was introduced in JDK 8, further improving the Future mechanism.

  ```java
  CompletableFuture<HistoryData> tag = previousTag.thenCompose(this::toNow);
  return tag.thenCompose(hd -> {
      if (Objects.isNull(hd) || CollectionUtils.isEmpty(hd.getInfos())) {
          return getHistoryFromStore(hd);
      }
      return CompletableFuture.completedFuture(hd);
  });
  ```

However, after adopting the Future programming mode, new complexities are introduced, and some of the original pain points are still unresolved, especially:

- Programming style converting from imperative programming to declarative programming. Declarative programming is usually not intuitive and the cost of understanding is high.
- The conversion between synchronous semantics and asynchronous semantics is not easy. Future's API has a high learning cost. For example, the various combination methods of CompletableFuture are very complex and hard to understand.
- Error handling is still complex. Future usually contains Running/Cancelled/Failed/Done states, and some states are handled through exceptions. It is still tedious to handle errors completely.
- Future is not convenient for testing intermediate states. For a chained Future, it is usually not convenient to specify the asynchronous execution order to detect possible concurrency errors. Of course, this is a difficulty common to all concurrent programming.

In order to further reduce the difficulty of asynchronous and concurrency control, we learn from the designs of other languages and implement the Promise mechanism in Java. That is, when executing an asynchronous task, the promise (Promise) will return the execution result at a certain point in time. This represents an unknown result. The entity is called a Promise. Using Promise abstraction can greatly reduce the mental burden on developers, greatly improve coding efficiency, and reduce the possibility of errors.

Let's first look at a code snippet using Promise to implement multi-threaded computing tasks for computationally intensive tasks.

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

We can see that compared to weakly typed languages such as Javascript, Java is strongly typed. (`Promise<String>` should be read as `Promise of String`). Using the generic mechanism can make code writing safer and more convenient. In addition, Javascript does not support multi-threading. The Promise component we designed naturally supports multi-threading.

In the above example, each accumulated operation is executed in a different thread. We have built in two simple and general thread pools. By default, Promise uses `ForkJoinPool` as the thread pool, which is suitable for computing-intensive tasks. For blocking IO types, you can use the built-in `Promise.ThreadPerTaskExecutor`, a simple thread pool model. If you need a more complex strategy, you can use a custom thread pool and pass it in the constructor.

For different scenarios, we abstract an asynchronous task into two types in design, one is suitable for most scenarios and requires regular tasks that need to be executed immediately, which we call `Promise`, and the other is asynchronous that needs to be executed with delay. Tasks are called [Deferred].

Deferred
========

`Deferred`:idx: is a Promise that is not executed immediately, realizing the separation of **data consumption** and **data acquisition**.

  ```java
  // Generate a promise object
  Deferred<String> deferred = new Deferred<>();
  //data consumption
  deferred.then(s -> {
      //The value of s is something, s is processed here
  });
  //data collection
  deferred.offer((resolve, reject) -> {
      resolve.apply("something");
  });
  //There is no order requirement for the above two steps, and the execution results are consistent.
  //Because the state of the promise object is cached, many problems caused by timing are reduced.
  ```

Normally, data acquisition and consumption are usually written together, but upon careful analysis we find that these are actually two completely different things.

For example, in the factory production line, if workers are not only responsible for processing goods (data consumption), but also responsible for picking up the goods to be processed (data acquisition), this is usually the practice of small workshops. In a professional assembly line, each worker only needs to simply complete the process he is responsible for while waiting for the product to be delivered to him. The small workshop approach increases the mental burden on workers, makes them more prone to errors, and reduces the efficiency of the assembly line.

`Deferred` helps us abstract this process, complete the separation of data acquisition and data processing, and better fulfill the idea of `Single Responsibility (SRP)`.

In practice, for example, if a function interface returns a String, it means that the function caller can use the String immediately. If you want to express the semantics of using String in the future, there is nothing you can do. **Deferred** provides this semantics. The semantics it expresses is that a String will be provided in the future. As for who will provide this String and how to obtain this String, that is completely another matter. No data processor is required. Care, the data processor only needs to know how to process this data and send it to the next process.

Since **Deferred** is also a **Promise**, it fully possesses the capabilities of Promise. Compared with Promise, Deferred only has one more **Initialized** state, as shown below

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

Core API
========

In just a few minutes, mastering the usage of the following core interfaces will help you get twice the result with half the effort in your daily work.

then
----

`then`:idx: The method is executed when the previous asynchronous operation is successful and returns a new Promise.

  ```java
  new Promise<String>((resolve, reject) -> {
        resolve.apply("something");
  }).then(s -> {
      // s is something
      // TODO processes something
  });
  ```

except
------

`except`:idx: Contrary to then, it is executed when the asynchronous operation fails and returns a new Promise.

  ```java
    new Promise<String>((resolve, reject) -> {
        // reject.apply(result);
    }).except((error, s) -> {
        // s is something
      	// Processing logic when TODO fails
    });
  ```

Using the above two interfaces, a simple asynchronous task processing process can be implemented. Under Android, it can replace [AsyncTask](https://developer.android.com/reference/android/os/AsyncTask).

  ```java
  new Promise<String>((resolve, reject) -> {
    	// Execute asynchronous tasks
    	// If successful, call resolve to change the state
    	// If it fails, call reject to change the state
  }).then(s -> {
      //Processing on success
  }).except((error, s) -> {
    	// Handling on failure
  });
  ```

Of course, since each interface of Promise returns a new Promise, you can also write it like this

  ```java
  new Promise<String>((resolve, reject) -> {
      //Execute asynchronous tasks
    	// If successful, call resolve to change the state
    	// If it fails, call reject to change the status
  }).except((error, o) -> {
    	// Handling on failure
  }).then(s -> {
    	// Processing on success
  });
  ```


It has no impact on the execution results. By now, you have mastered the basic usage of Promise.


done
----
`done`:idx: Executed when the status of the previous asynchronous operation is determined (success or failure), and returns a new Promise. In some scenarios, we don't care whether the asynchronous task succeeds or fails, such as data reporting, but we just need to perform some operations when the task is completed.

  ```java
  new Promise<String>((resolve, reject) -> {
      //Execute asynchronous tasks
  }).done(o -> {
      //The asynchronous task is completed, whether it is successful or failed
  });
  ```

> Note: Once the state of a promise is determined, it will not change its own state and its data. This is different from a state machine.

Since each interface of Promise returns a new Promise object, it allows us to perform chain operations and solves the "callback hell" problem caused by conventional callback-based programming. The above two code snippets will both return a new Promise object, whose type is `Promise<Integer>`.
The above interface can meet the processing flow of a single asynchronous task. In actual situations, asynchronous tasks often have serial and parallel requirements. Promise has powerful abstract description capabilities for asynchronous operations, allowing us to combine asynchronous task processing flows.

Let's look at a common situation **Serial Combination**. Here we need to introduce a new interface [next]. For more advanced combination methods, please refer to [Concurrency Control]

next
----

`next`:idx: The operation is executed when the previous asynchronous operation is successful, receives the result of the previous step, initiates the next new asynchronous task, returns the Promise of the task, and can perform data type conversion.

  ```java
  new Promise<String>((resolve, reject) -> {
      resolve.apply("something");
  }).next(o -> new Promise<Integer>((resolve, reject) -> {
      resolve.apply(o.length());
  })).then(integer -> {
      // integer is 9
  });
  ```

In actual situations, the asynchronous processing process may fail due to various reasons, so we'd better add an exception handling branch.

  ```java
  new Promise<String>((resolve, reject) -> {
      resolve.apply("something");
  }).next(o -> new Promise<Integer>((resolve, reject) -> {
      resolve.apply(o.length());
  })).then(integer -> {
      // integer is 9
  }).except((error, o) -> {
      //Processing flow in case of failure
  });
  ```

timeout
-------

The Promise A+ specification does not define a standard processing method for timeout (`timeout`:idx: ), but timeout processing is very common in actual business environments, so we have extended the specification and built-in support for timeout.
In the following code, if the asynchronous task is not completed within 1 second (resolve or reject is actively called), the Promise will automatically enter the error state (reject state).
> Delayed operations, such as timers, are generally the source of application memory leaks. If Promise reverses the state (resolve or reject) before the timer ends, the timer will be automatically canceled to avoid memory leaks.

  ```java
  new Promise<String>((resolve, reject) -> {
      // Must call resolve or reject within the specified time to deliver the result
      // otherwise it will automatically change to reject state
  }, Timeout.ofSeconds(1)).except((err, o) -> {
      // Error handling process
      // If no task is completed within 1s, err is PromiseTimeoutException
  }).then(s -> {
      // Processing on success
  });
  ```


The above is the core interface of Promise.

Utilities
=========

The following interfaces provide capabilities that are often used in practice. Such interfaces are wrappers of the core API.

delay
-----

We can very simply add `delay`:idx: delay execution effect to any Promise object, which can replace `Thread.sleep`. This interface returns a new Promise with delay effect added.

Specific examples of usage

  ```java
  new Promise<String>((resolve, reject) -> {
      resolve.apply("something")
  }).delay(Timeout.ofMillis(500)).then(o -> {
      //Delay 500ms to receive results
      // o is something
  })
  ```

Or in the form of static methods, the essence is the same.

```java
Promise.delay(aPromise, Timeout.ofMillis(delay))
```

You can also use deferred operations as the starting point for business logic.

  ```java
  Promise.delay(Timeout.ofMillis(500)).next(unused -> new Promise<String>((resolve, reject) -> {
      resolve.apply("something")
  })).then(o -> {
      //Delay 500ms to receive results
      // o is something
  });
  ```

recover
-------

It is very simple to add exception recovery logic `recover`:idx: to any Promise object. Executed when the previous asynchronous operation fails, receives the error message of the previous step, executes a new asynchronous task to obtain the same type of data, and returns the Promise of the task.

Common application scenarios:
- Failed to read cache, reading from network
- Network error, switch IP and try again

This method is similar to the `next` interface, the differences are as follows
| Interface | Execution timing | Data type |
| ------ | ------ | ------ |
| **next** | Executed when the previous step is successful | **can** return a Promise with a different data type** from the previous step |
| **recover** | Executed when the previous step fails| **Must** return a Promise of the same data type** as the previous step|

Typical usage:

  ```java
  new Promise<String>((resolve, reject) -> reject.apply("error"))
  .recover((error, o) -> new Promise<String>((resolve, reject) -> resolve.apply("something")))
  .then(s -> {
      // s is something
  })
  ```

Or in the form of a static method

  ```java
  Promise<String> aPromise = new Promise<>((resolve, reject) -> reject.apply("error"));
      //Error recovery
  Promise.recover(aPromise, (error, o) -> new Promise<>((resolve, reject) -> resolve.apply("something")))
  .then(s -> {
      //s is something
  })
  ```

Because recover also returns a new Promise object, it can implement very flexible recovery logic, such as multiple attempts, or adding delay and other strategies. Please refer to [retry].

retry
-----

`retry`:idx: it is to execute the recovery process with a certain strategy. We can add a retry strategy to an asynchronous task very simply.

  ```java
  int retryCountMax = 3;
  Promise.retry(() -> new Promise<>((Task<String, Object>) (resolve, reject) -> {
      // This task will be executed retryCountMax times at most. If successful, it will not be retried.
  }), retryCountMax).then(s -> {
      // Success, consume the data
  }).except((error, o) -> {
      // Entering this branch means that all three retries failed.
  })
  ```

We can specify a **retry strategy**, such as not retrying if there is no network, or retrying at different intervals each time to avoid causing server pressure.

  ```java
  int retryCountMax = 3;
  Promise.retry(() -> new Promise<>((Task<String, Object>) (resolve, reject) -> {
      // This task will be executed retryCountMax times at most. If successful, it will not be retried.
  }), retryCountMax, new RetryStrategy() {
      @Override
      public Timeout delay(int retrySeq) {
          // Specify the interval between each retry
          // Here to achieve the effect of retrying at intervals of 1s, 2s, 3s...
          return Timeout.ofSeconds((long) retrySeq);
      }
        
      @Override
      public boolean condition(int attemptsRemain, Throwable throwable, Object o) {
          // If this method returns true, continue to retry, return false to abort the retry.
          // The input parameters include the current number of retries and the error cause of the last retry.
          return true;
      }
  }).then(s -> {
      // Success, consume the data
  }).except((error, o) -> {
      // Entering this branch means that all three retries failed.
  })
  ```

There are two commonly used retry strategies built into the component. If no retry strategy parameters are specified, the default **Default** retry strategy is used, specifically **retry at a fixed interval of 1 second until the maximum number of retries**.

```java
Promise.retry(..., retryMaxCount);
// Equivalent to
Promise.retry(..., retryMaxCount,RetryStrategy.Default);
```

The two built-in retry strategies are implemented as follows. This interface can also be extended to implement a custom strategy more suitable for the business.

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

Likewise, this method returns a new Promise object.

> Note that when using this interface, the number of retries must be controlled within a reasonable range. Generally, 3 to 5 retries can meet most needs. This interface is not suitable for long-term retry or infinite retry scenarios. It is recommended to use `TimerTask` to further encapsulate Promise. Please refer to the following example.

  ```java
  /**
  * Retry a task at a fixed interval until it is actively canceled or times out.
  * If timeout is set to null, which means the task will never time out.
  * @param task The task to be executed
  * @param periodMills retry interval, in milliseconds
  * @param timeout timeout, which can be null, which means the task will never timeout.
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

`validation`:idx: further verify the data and return a new Promise object representing the verification result.

  ```java
  Promise.resolve("something")
         .validate(s -> s.length() == 9)
         .then(s -> {
             //The result is legal
         }).except((error, o) -> {
             //If the verification fails, the error type is ValidationFailedException.
         });
  ```

We can collect and store commonly used data verification rules for use at any time, and combine them with Promise to achieve more powerful effects.

  ```java
  //The following rules verify that the input string cannot be empty, and the length must be greater than 5
  Validation<String> ensureStrLengthIsFive = ((Validation<String>) o -> o != null).except(1001, "string is null")
      .andThen(((Validation<String>) s -> s.length() > 5).except(1002,"string length not corrent"));
    		
  Promise.resolve("hi").validate(ensureStrLengthIsFive).except((error, o) -> {
      if (error instanceof ValidationFailedException) {
          long errCode = ((ValidationFailedException) error).getErrorCode(); // Error code 1002
          String errMsg = error.getMessage(); // Corresponding error message
      }
  });
    
  Promise.resolve("something").validate(ensureStrLengthIsFive).then(s -> {
    // ok meets the requirements and passes the verification
  });
  ```

Control Flow
=============

join
----

`join`:idx: Convert asynchronous operations to synchronization. In some scenarios, we may need to obtain the execution results of a task synchronously.
In the event that asynchronous execution fails, this method will throw a `PromiseException`, with the following possible situations.

- `PromiseCompleteException`
The asynchronous task execution failed, that is, the asynchronous task called the **reject.apply** method.
- `PromiseTimeoutException`
The asynchronous task execution times out. **Each promise can specify a timeout period**. If the status reversal is not completed within the specified time, this exception will be thrown.
- `PromiseException`
Other exceptions that occur during asynchronous task execution, such as NP, etc. **Exceptions that occur during the execution of Promise will be caught** to ensure that it will not cause the program to crash.

  ```java
  String s = new Promise<String>((resolve, reject) -> {
      // Asynchronous tasks
  }).join();
  //If no exception occurs, process s here
  ```

abort
-----

`abort`:idx: Abort the task, its execution result depends on the status of the current Promise. For a Promise that is already in the **Done** state, it has no effect, otherwise it will be immediately reversed to the Reject state.

Traditional Promise does not have an interface that supports cancellation. TC39 officially [had a proposal](https://github.com/tc39/proposal-cancelable-promises) but it has been rejected.
The current official solution is to use [AbortController](https://developer.mozilla.org/en-US/docs/Web/API/AbortController/abort) in conjunction with Promise to implement the cancellation function. In actual use, a single Promised object is acceptable, but in chain calls, it is difficult for us to know at what stage the asynchronous task execution has reached, which Promise object should be canceled, or even the reference to the Promise object cannot be obtained.

In addition, there are also solutions to use a cancelable promise object wrapped by the `Promise.race` interface, such as [a discussion on SF](https://stackoverflow.com/questions/30233302/promise-is-it -possible-to-force-cancel-a-promise). The author believes that this solution is relatively cumbersome to use and the code is not very readable.

In our implementation, canceling a Promise is very simple. You only need to call the abort method of the Promise object. With one call, you can easily cancel the entire Promise chain, and by the way, [Structured Concurrency] is achieved (https://en.wikipedia .org/wiki/Structured_concurrency).

  ```java
  Promise<Void> promise = Promise.delay(Timeout.ofMillis(2000), null);
  promise.except((error, o) -> {
      //The error type is AbortException
  }).done(o -> {
      //The o type is PromiseException, and the object type returned by getCause is AbortException.
  });
  promise.abort(); //Task cancellation
  ```

After the abort interface is called, the abort event will propagate throughout the asynchronous task chain and cancel the entire task chain. For example, in the above code, after the Promise is canceled, if the 2000ms timer resource has not been applied for, it will no longer be applied for. If it has been applied for, the timer resource will be automatically released.

Developers can also implement the aftermath of resource release by overriding the `onAbort` method. The following code example demonstrates the combination with OKHttp.

  ```java
  AbortController controller = new AbortController();
  Promise<Integer> promise = new Promise<Integer>((resolve, reject) -> {
      // Register the operation of cleaning up resources
      controller.signal.onAbort(reason -> {
          // Multi-thread resource competition needs to be considered here
          // Because sending HTTP requests and canceling HTTP requests may not be in the same thread.
          OkHttpUtils.cancelCall(client);
      });
      // Send HTTP request (synchronous method, time-consuming operation)
      Response response = client.newCall(request).execute()
      // Use resources
      resolve.apply(response.code());
  }) {
      @Override
      public void onAbort(String reason) {
          controller.abort(reason);
      }
  }.except((error, o) -> {
      //The error type is AbortException
  }).done(o -> {
      //The o type is PromiseException, and the object type returned by getCause is AbortException.
  });
    
  promise.abort(); // Task cancellation
  ```

Example of use in Kotlin.

  ```java
  //Create an AbortController object
  val abortController = AbortController()
  
  //Create a Promise object
  val promise = object : Promise<String>(Task<String, Any> { resolve, _ ->
      val t1 = Thread.currentThread()
      abortController.signal.onAbort {
          t1.interrupt()
      }
  	//simulate a time consuming task 
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
  
  //Abort Promise execution when needed
  promise.abort()
  ```


Concurrency
===========

all
---
`all`:idx: Accept any number of Promise objects and execute asynchronous tasks concurrently. If all tasks are successful, if one fails, the entire task is considered a failure.

race
----
`race`:idx: Accept any number of Promise objects and execute asynchronous tasks concurrently. Time is the first priority. For multiple tasks, the result returned first shall prevail. The success of this result is the overall success, and the failure of this result is the overall failure.

any
---
`any`:idx: Accept any number of Promise objects and execute asynchronous tasks concurrently. Waiting for one of them to succeed is considered successful. If all tasks fail, it will enter the error state and output an error list.

allSettled
----------
`allSettled`:idx: Task priority, all tasks must be executed and will never enter a failed state.

For details, please refer to the [MDN Promise API Document](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise). The specific implementation is consistent.

Memory Model
=============

Asynchronous tasks and multi-thread concurrency are a major problem in business processing. If you are not careful, problems such as crashes, deadlocks, and variable synchronization may occur. Using Promise technology can solve the above two major problems safely and efficiently.

First of all, Promise solves the problem of asynchronous data flow management. In addition, each asynchronous task of Promise is naturally executed by multiple threads. When concurrency is possible, it will be executed concurrently to maximize the running efficiency of the program, such as `all`, `race`, `any` and other interfaces, which abstract common concurrency models.

If we expand here, the length will be relatively long. Here is just a brief explanation. Using Promise mode for asynchronous programming, developers generally do not need to worry about such issues.


Other Instructions
==================

Requirements
------------
Does not rely on any third-party libraries, specific requirements are as follows

- Java platform: Java version >=1.8
- Android platform: Android version >=24 (Nougat)

The Android platform can support older Android versions (not yet implemented) via [android-retrofuture](https://github.com/retrostreams/android-retrofuture).

resources
----------
- Future/Promise are slightly different, see (https://en.wikipedia.org/wiki/Futures_and_promises) for details
- Google implemented the Swift version of the Promise library (https://github.com/google/promises)
- [RxJava](https://github.com/ReactiveX/RxJava) A reactive programming, a programming model based on the concept of asynchronous data flow.
- Promise specification [Promise A+](https://promisesaplus.com/).
- [MDN Promise API Documentation](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise).
- [Promise A+ Specification](https://promisesaplus.com/)
