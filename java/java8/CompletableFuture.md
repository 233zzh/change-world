## Java8——CompletableFuture的使用
### CompletableFuture概念

CompletableFuture在Java8中被用作异步编程而引入。异步编程将在主线程之外创建一个独立的子线程，并在上面运行一个非阻塞的任务，然后通知主线程，通过CompletableFuture我们可以并行的运行多个任务。

### CompletableFuture vs Future

CompletableFuture是Future的拓展，其实现了Future以及CompletionStage接口，除了Future的功能外提供了很多创建、组合、串联等任务编排功能。

使用Future获得异步执行结果需要调用阻塞方法get()或者轮询isDone()是否为true，这两种方法会导致主线程阻塞并等待，CompletableFuture针对Future做了改进，可以传入回调对象，当异步任务完成或者发生异常时，自动调用回调对象的回调方法。

### CompletableFuture常用API

（1）创建异步任务

| 接口方法 | 说明 |
| --- | --- |
|  CompletableFuture<Void> runAsync(Runnable runnable) | Runnable函数式接口类型为参数，无返回值 |
| CompletableFuture<Void> runAsync(Runnable runnable, Executor executor) | 同上，但区别在于指定线程池，而非使用默认的ForkJoinPool |
| <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier) | 以Supplier函数式接口类型为入参有返回值，获取返回值get方法会阻塞 |
| <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier, Executor executor) | 同上，但区别在于指定线程池，而非使用默认的ForkJoinPool |

（2）获取异步执行结果

| 接口方法 | 说明 |
| --- | --- |
| U get() | 获取CompletableFuture异步之后的返回值，抛出的是经过检查的异常 |
| U join() | 获取CompletableFuture异步之后的返回值，抛出的是uncheck异常 |

```java
// 线程池
private static ExecutorService threadPool = Executors.newFixedThreadPool(10);

// 异步任务创建
public void createAsyncTask() {
    // 有返回值supplyAsync
    CompletableFuture<String> supplyFuture = CompletableFuture.supplyAsync(()->{
        log.info("exe supplyAsync");
        return "我有返回值";
    });
    // 有返回值supplyAsync，线程池
    CompletableFuture<String> supplyFutureWithPool = CompletableFuture.supplyAsync(()->{
        log.info("exe supplyAsync with customize thread pool");
        return "我有返回值(我也用了线程池)";
    }, threadPool);
    // 获取异步结果 join
    String resByJoin = supplyFuture.join();
    log.info("resByJoin:{}", resByJoin);
    try {
				// get需要捕获异常
        String resByGet = supplyFutureWithPool.get();
        log.info("resByGet:{}", resByGet);
    } catch (InterruptedException | ExecutionException e) {
        log.error("resByGet, e", e);
        Thread.currentThread().interrupt();
    
		// 无返回值
    CompletableFuture<Void> run = CompletableFuture.runAsync(()-> log.info("runAsync"));
    log.info("runAsync-res:{}", run.join());//返回为空
}
```

```
[2023-07-25 23:12:07.285][INFO ][ForkJoinPool.commonPool-worker-9] com.tqy.toolsct.thread.CFLearn01 lambda$createAsyncTask$2:98 - exe supplyAsync
[2023-07-25 23:12:07.285][INFO ][pool-2-thread-1] com.tqy.toolsct.thread.CFLearn01 lambda$createAsyncTask$3:103 - exe supplyAsync with customize thread pool
[2023-07-25 23:12:07.296][INFO ][Test worker] com.tqy.toolsct.thread.CFLearn01 createAsyncTask:108 - resByJoin:我有返回值
[2023-07-25 23:12:07.296][INFO ][Test worker] com.tqy.toolsct.thread.CFLearn01 createAsyncTask:111 - resByGet:我有返回值(我也用了线程池)
[2023-07-25 23:12:07.297][INFO ][ForkJoinPool.commonPool-worker-9] com.tqy.toolsct.thread.CFLearn01 lambda$createAsyncTask$4:117 - runAsync
[2023-07-25 23:12:07.298][INFO ][Test worker] com.tqy.toolsct.thread.CFLearn01 createAsyncTask:118 - runAsync-res:null
```

（3）执行完异步任务后，执行回调方法

| 接口方法 | 说明 |
| --- | --- |
| CompletableFuture<Void> thenRun(Runnable action) | 第一个任务执行完成后，执行第二个回调方法任务,不依赖上一个任务结果，直接回调 |
| CompletableFuture<Void> thenRunAsync(Runnable action) | 第一个任务执行完成后，执行第二个回调方法任务,不依赖上一个任务结果，直接回调, 使用不一样线程池 |
| CompletableFuture<Void> thenRunAsync(Runnable action, Executor executor) | 同上依赖线程池 |
| CompletableFuture<Void> thenAccept(Consumer<? super T> action) (同样有Async等方法) | 第一个任务执行完成后，执行第二个回调方法任务，会将该任务的执行结果作为入参传递到回调方法中，但是回调方法是没有返回值的 |
| <U> CompletableFuture<U> thenApply(Function<? super T,? extends U> fn)  (同样有Async等方法) | 第一个任务执行完成后，执行第二个回调方法任务，会将该任务的执行结果作为入参传递到回调方法中，回调方法有返回值 |
| CompletableFuture<T> whenComplete(BiConsumer<? super T,? super Throwable> action) (同样有Async等方法) | 可以处理正常或者异常的结果，无返回值，使用的线程为前面执行任务的同一个线程 |
| <U> CompletableFuture<U> handle(BiFunction<? super T, Throwable, ? extends U> fn) (同样有Async等方法) | 可以处理正常或异常的结果，使用的线程为前面执行任务的同一个线程 |

```java
// thenRun
public void taskThenRun() {
    CompletableFuture<String> supplyFuture = CompletableFuture.supplyAsync(()->{
        log.info("执行任务一");
        return "test";}, threadPool);
    //CompletableFuture<Void> thenRun = supplyFuture.thenRun(()-> log.info("执行任务二")); //使用相同线程
    CompletableFuture<Void> thenRun = supplyFuture.thenRunAsync(()-> log.info("执行任务二")); //使用不同线程
    log.info("返回任务一：{}，任务二：{}", supplyFuture.join(), thenRun.join());
}

[2023-07-25 23:33:04.125][INFO ][pool-2-thread-1] com.tqy.toolsct.thread.CFLearn01 lambda$taskThenRun$9:158 - 执行任务一
[2023-07-25 23:33:04.130][INFO ][pool-2-thread-1] com.tqy.toolsct.thread.CFLearn01 lambda$taskThenRun$10:160 - 执行任务二
[2023-07-25 23:33:04.133][INFO ][Test worker] com.tqy.toolsct.thread.CFLearn01 taskThenRun:161 - 返回任务一：test，任务二：null

#使用不同线程
[2023-07-25 23:34:52.487][INFO ][pool-2-thread-1] com.tqy.toolsct.thread.CFLearn01 lambda$taskThenRun$9:158 - 执行任务一
[2023-07-25 23:34:52.493][INFO ][ForkJoinPool.commonPool-worker-9] com.tqy.toolsct.thread.CFLearn01 lambda$taskThenRun$10:161 - 执行任务二
[2023-07-25 23:34:52.497][INFO ][Test worker] com.tqy.toolsct.thread.CFLearn01 taskThenRun:162 - 返回任务一：test，任务二：null
```

```java
//thenAccept
public void taskThenAccept() {
    CompletableFuture<String> supplyFuture = CompletableFuture.supplyAsync(()->{
        log.info("执行任务一");
        return "test";}, threadPool);
		// 传递参数
    CompletableFuture<Void> thenRun = supplyFuture.thenAccept(a-> {
        log.info("执行任务二");
        log.info("获取任务一结果：{}", a);// 无返回
    });
//        CompletableFuture<Void> thenRun = supplyFuture.thenAcceptAsync(a-> {
//            log.info("执行任务二");
//            log.info("获取任务一结果：{}", a);
//        });
    log.info("返回任务一：{}，任务二：{}", supplyFuture.join(), thenRun.join());
}

[2023-07-26 08:21:06.314][INFO ][pool-2-thread-1] com.tqy.toolsct.thread.CFLearn01 lambda$taskThenAccept$11:167 - 执行任务一
[2023-07-26 08:21:06.320][INFO ][pool-2-thread-1] com.tqy.toolsct.thread.CFLearn01 lambda$taskThenAccept$12:170 - 执行任务二
[2023-07-26 08:21:06.323][INFO ][pool-2-thread-1] com.tqy.toolsct.thread.CFLearn01 lambda$taskThenAccept$12:171 - 获取任务一结果：test
[2023-07-26 08:21:06.324][INFO ][Test worker] com.tqy.toolsct.thread.CFLearn01 taskThenAccept:177 - 返回任务一：test，任务二：null
```

```java
//thenApply
public void taskThenApply() {
    CompletableFuture<String> supplyFuture = CompletableFuture.supplyAsync(()->{
        log.info("执行任务一");
        return "test";}, threadPool);
		// 传递参数
    CompletableFuture<String> thenRun = supplyFuture.thenApply(a-> {
        log.info("执行任务二");
        log.info("获取任务一结果：{}", a);
        return a+"hahah"; // 有返回
    });
//        CompletableFuture<Void> thenRun = supplyFuture.thenApplyAsync(a-> {
//            log.info("执行任务二");
//            log.info("获取任务一结果：{}", a);
//        return a+"hahah";
//        });
    log.info("返回任务一：{}，任务二：{}", supplyFuture.join(), thenRun.join());
}

[2023-07-26 20:34:32.090][INFO ][pool-2-thread-1] com.tqy.toolsct.thread.CFLearn01 lambda$taskThenApply$13:182 - 执行任务一
[2023-07-26 20:34:32.095][INFO ][pool-2-thread-1] com.tqy.toolsct.thread.CFLearn01 lambda$taskThenApply$14:185 - 执行任务二
[2023-07-26 20:34:32.097][INFO ][pool-2-thread-1] com.tqy.toolsct.thread.CFLearn01 lambda$taskThenApply$14:186 - 获取任务一结果：test
[2023-07-26 20:34:32.098][INFO ][Test worker] com.tqy.toolsct.thread.CFLearn01 taskThenApply:194 - 返回任务一：test，任务二：testhahah
```

```java

// whencomplete
public void whenTaskComplete() {
    CompletableFuture<String> supplyFuture = CompletableFuture.supplyAsync(()->{
        log.info("exe supplyAsync");
        //int i = 10 /0;// 异常
        return "我有返回值";
    }).whenComplete((res, e)->{
        if(e != null) {
            log.error("异步任务执行失败，进行回调", e);
        } else {
            log.info("whenComplete回调:前面任务执行结果为:{}", res);
        }
    });
    log.info("res:{}", supplyFuture.join());
}

[2023-07-25 23:05:47.639][INFO ][ForkJoinPool.commonPool-worker-9] com.tqy.toolsct.thread.CFLearn01 lambda$whenTaskComplete$4:120 - exe supplyAsync
[2023-07-25 23:05:47.647][INFO ][ForkJoinPool.commonPool-worker-9] com.tqy.toolsct.thread.CFLearn01 lambda$whenTaskComplete$5:127 - whenComplete回调:前面任务执行结果为:我有返回值
[2023-07-25 23:05:47.648][INFO ][Test worker] com.tqy.toolsct.thread.CFLearn01 whenTaskComplete:130 - res:我有返回值
```

```java
// 发生异常
public void whenTaskComplete() {
    CompletableFuture<String> supplyFuture = CompletableFuture.supplyAsync(()->{
        log.info("exe supplyAsync");
        //int i = 10 /0;// 异常
        return "我有返回值";
    }).whenComplete((res, e)->{
        if(e != null) {
            log.error("异步任务执行失败，进行回调", e);
        } else {
            log.info("whenComplete回调:前面任务执行结果为:{}", res);
        }
    });
    log.info("res:{}", supplyFuture.join());
}

[2023-07-25 23:07:01.447][INFO ][ForkJoinPool.commonPool-worker-9] com.tqy.toolsct.thread.CFLearn01 lambda$whenTaskComplete$4:120 - exe supplyAsync
[2023-07-25 23:07:01.453][ERROR][ForkJoinPool.commonPool-worker-9] com.tqy.toolsct.thread.CFLearn01 lambda$whenTaskComplete$5:125 - 异步任务执行失败，进行回调
java.util.concurrent.CompletionException: java.lang.ArithmeticException: / by zero
	at java.util.concurrent.CompletableFuture.encodeThrowable(CompletableFuture.java:273) ~[?:1.8.0_301]
	at java.util.concurrent.CompletableFuture.completeThrowable(CompletableFuture.java:280) [?:1.8.0_301]
	at java.util.concurrent.CompletableFuture$AsyncSupply.run(CompletableFuture.java:1606) [?:1.8.0_301]
	at java.util.concurrent.CompletableFuture$AsyncSupply.exec(CompletableFuture.java:1596) [?:1.8.0_301]
	at java.util.concurrent.ForkJoinTask.doExec(ForkJoinTask.java:289) [?:1.8.0_301]
	at java.util.concurrent.ForkJoinPool$WorkQueue.runTask(ForkJoinPool.java:1067) [?:1.8.0_301]
	at java.util.concurrent.ForkJoinPool.runWorker(ForkJoinPool.java:1703) [?:1.8.0_301]
	at java.util.concurrent.ForkJoinWorkerThread.run(ForkJoinWorkerThread.java:172) [?:1.8.0_301]
Caused by: java.lang.ArithmeticException: / by zero
	at com.tqy.toolsct.thread.CFLearn01.lambda$whenTaskComplete$4(CFLearn01.java:121) ~[main/:?]
	at java.util.concurrent.CompletableFuture$AsyncSupply.run(CompletableFuture.java:1604) ~[?:1.8.0_301]
	... 5 more
```

```java
// 自定义线程池
public void whenTaskComplete() {
    CompletableFuture<String> supplyFuture = CompletableFuture.supplyAsync(()->{
        log.info("exe supplyAsync");
        //int i = 10 /0;// 异常
        return "我有返回值";
    }).whenCompleteAsync((res, e)->{
        if(e != null) {
            log.error("异步任务执行失败，进行回调", e);
        } else {
            log.info("whenComplete回调:前面任务执行结果为:{}", res);
        }
    }, threadPool);
    log.info("res:{}", supplyFuture.join());
}

[2023-07-25 23:08:51.315][INFO ][ForkJoinPool.commonPool-worker-9] com.tqy.toolsct.thread.CFLearn01 lambda$whenTaskComplete$4:120 - exe supplyAsync
[2023-07-25 23:08:51.324][INFO ][pool-2-thread-1] com.tqy.toolsct.thread.CFLearn01 lambda$whenTaskComplete$5:127 - whenComplete回调:前面任务执行结果为:我有返回值
[2023-07-25 23:08:51.325][INFO ][Test worker] com.tqy.toolsct.thread.CFLearn01 whenTaskComplete:130 - res:我有返回值
```

```java
// handle
public void taskHandle() {
    CompletableFuture<String> supplyFuture = CompletableFuture.supplyAsync(()->{
        log.info("exe supplyAsync with thread");
        //int i = 10 /0;// 异常
        return "我有返回值";
    }, threadPool).handle((res, e)->{
        if (Objects.equals("我有返回值", res)) {
            log.info("上个任务结果:{}, 传入回调", res);
        }
        if(e != null) {
            log.error("异步任务执行失败，进行回调", e);
        } else {
            log.info("handle回调:前面任务执行结果为:{}", res);
        }
        return "我是handle处理过的："+ res;
    });
    log.info("res:{}", supplyFuture.join());
}

[2023-07-25 23:20:52.127][INFO ][pool-2-thread-1] com.tqy.toolsct.thread.CFLearn01 lambda$taskHandle$7:139 - exe supplyAsync with thread
[2023-07-25 23:20:52.134][INFO ][pool-2-thread-1] com.tqy.toolsct.thread.CFLearn01 lambda$taskHandle$8:144 - 上个任务结果:我有返回值, 传入回调
[2023-07-25 23:20:52.136][INFO ][pool-2-thread-1] com.tqy.toolsct.thread.CFLearn01 lambda$taskHandle$8:149 - handle回调:前面任务执行结果为:我有返回值
[2023-07-25 23:20:52.137][INFO ][Test worker] com.tqy.toolsct.thread.CFLearn01 taskHandle:153 - res:我是handle处理过的：我有返回值
```

（4）异步任务组合AND

| 接口方法 | 说明 |
| --- | --- |
| <U,V> CompletableFuture<V> thenCombine(CompletionStage<? extends U> other,BiFunction<? super T,? super U,? extends V> fn) | 将两个任务的执行结果作为方法入参传递到指定方法中，有返回值 |
| <U> CompletableFuture<Void> thenAcceptBoth(CompletionStage<? extends U> other,BiConsumer<? super T, ? super U> action) | 将两个任务的执行结果作为方法入参传递到指定方法中，无返回值 |
| CompletableFuture<Void> runAfterBoth(CompletionStage<?> other, Runnable action) | 不关心两个任务的执行结果，且没有返回值 |

```java
//thenCombine
public void taskCombine() {
    CompletableFuture<String> supplyFuture01 = CompletableFuture.supplyAsync(()->{
        log.info("执行任务一");
        return "test1";}, threadPool);
    CompletableFuture<String> supplyFuture02 = CompletableFuture.supplyAsync(()->{
        log.info("执行任务二");
        return "test2";});

    CompletableFuture<String> combine = supplyFuture01.thenCombineAsync(supplyFuture02, (f, s)->{
        log.info("合并任务");
        return "合并" + f + s; // 两个任务的结果处理，有返回值
    }, threadPool);
    log.info("combine:{}", combine.join());
}

[2023-07-26 20:46:37.675][INFO ][ForkJoinPool.commonPool-worker-9] com.tqy.toolsct.thread.CFLearn01 lambda$taskCombine$16:202 - 执行任务二
[2023-07-26 20:46:37.675][INFO ][pool-2-thread-1] com.tqy.toolsct.thread.CFLearn01 lambda$taskCombine$15:199 - 执行任务一
[2023-07-26 20:46:37.680][INFO ][pool-2-thread-2] com.tqy.toolsct.thread.CFLearn01 lambda$taskCombine$17:206 - 合并任务
[2023-07-26 20:46:37.683][INFO ][Test worker] com.tqy.toolsct.thread.CFLearn01 taskCombine:209 - combine:合并test1test2
```

```java
// thenAcceptBoth
public void taskAcceptBoth() {
    CompletableFuture<String> supplyFuture01 = CompletableFuture.supplyAsync(()->{
        log.info("执行任务一");
        return "test1";}, threadPool);
    CompletableFuture<String> supplyFuture02 = CompletableFuture.supplyAsync(()->{
        log.info("执行任务二");
        return "test2";}, threadPool);

    CompletableFuture<Void> acceptBoth = supplyFuture01.thenAcceptBothAsync(supplyFuture02, (f, s)->{
        log.info("合并任务" + f + s);//无返回值
    }, threadPool);
    log.info("acceptBoth:{}", acceptBoth.join());
}

[2023-07-26 20:54:38.088][INFO ][pool-2-thread-2] com.tqy.toolsct.thread.CFLearn01 lambda$taskAcceptBoth$19:217 - 执行任务二
[2023-07-26 20:54:38.088][INFO ][pool-2-thread-1] com.tqy.toolsct.thread.CFLearn01 lambda$taskAcceptBoth$18:214 - 执行任务一
[2023-07-26 20:54:38.095][INFO ][pool-2-thread-3] com.tqy.toolsct.thread.CFLearn01 lambda$taskAcceptBoth$20:221 - 合并任务test1test2
[2023-07-26 20:54:38.099][INFO ][Test worker] com.tqy.toolsct.thread.CFLearn01 taskAcceptBoth:223 - acceptBoth:null
```

```java
// runAfterBoth
public void taskRunBoth() {
    CompletableFuture<String> supplyFuture01 = CompletableFuture.supplyAsync(()->{
        log.info("执行任务一");
        return "test1";});
    CompletableFuture<String> supplyFuture02 = CompletableFuture.supplyAsync(()->{
        log.info("执行任务二");
        return "test2";}, threadPool);

    CompletableFuture<Void> taskRunBoth = supplyFuture01.runAfterBothAsync(supplyFuture02, ()->{
        log.info("两个任务处理完毕------"); // 无法接受任务一二的结果，无返回
    }, threadPool);
    log.info("taskRunBoth:{}", taskRunBoth.join());
}

[2023-07-26 20:57:28.000][INFO ][pool-2-thread-1] com.tqy.toolsct.thread.CFLearn01 lambda$taskRunBoth$22:231 - 执行任务二
[2023-07-26 20:57:28.000][INFO ][ForkJoinPool.commonPool-worker-9] com.tqy.toolsct.thread.CFLearn01 lambda$taskRunBoth$21:228 - 执行任务一
[2023-07-26 20:57:28.006][INFO ][pool-2-thread-2] com.tqy.toolsct.thread.CFLearn01 lambda$taskRunBoth$23:235 - 两个任务处理完毕------
[2023-07-26 20:57:28.009][INFO ][Test worker] com.tqy.toolsct.thread.CFLearn01 taskRunBoth:237 - taskRunBoth:null
```

(5) 任一异步任务OR

| 接口方法 |  说明 |
| --- | --- |
| <U> CompletableFuture<U> applyToEither(CompletionStage<? extends T> other, Function<? super T, U> fn) | 其中任一任务完成后，结果作为入参传递到指定方法，有返回结果，要求两个任务结果为同一类型 |
| CompletableFuture<Void> acceptEither(CompletionStage<? extends T> other, Consumer<? super T> action) | 其中任一任务完成后，结果作为入参传递到指定方法，无返回结果，要求两个任务结果为同一类型 |
| CompletableFuture<Void> runAfterEither(CompletionStage<?> other,Runnable action) | 其中任一任务完成后回调，无入参不返回结果，不要求两个任务结果为同一类型 |

```java
//applyToEither
public void taskApplyEither() {
    CompletableFuture<String> supplyFuture01 = CompletableFuture.supplyAsync(()->{
        log.info("执行任务一");
        try {
            Thread.sleep(500);
        } catch (Exception e) {

        }
        return "test1";});
    CompletableFuture<String> supplyFuture02 = CompletableFuture.supplyAsync(()->{
        log.info("执行任务二");
        try {
            Thread.sleep(200);
        } catch (Exception e) {

        }
        return "test2";}, threadPool);

    CompletableFuture<String> applyToEither = supplyFuture01.applyToEitherAsync(supplyFuture02, (f)->{
        log.info("任一个任务处理完毕------");
        return "执行完的任务"+f;
    }, threadPool);
    log.info("applyToEither:{}", applyToEither.join());
}

[2023-07-26 21:16:40.781][INFO ][pool-2-thread-1] com.tqy.toolsct.thread.CFLearn01 lambda$taskApplyEither$25:250 - 执行任务二
[2023-07-26 21:16:40.781][INFO ][ForkJoinPool.commonPool-worker-9] com.tqy.toolsct.thread.CFLearn01 lambda$taskApplyEither$24:242 - 执行任务一
[2023-07-26 21:16:40.989][INFO ][pool-2-thread-2] com.tqy.toolsct.thread.CFLearn01 lambda$taskApplyEither$26:259 - 任一个任务处理完毕------
[2023-07-26 21:16:40.994][INFO ][Test worker] com.tqy.toolsct.thread.CFLearn01 taskApplyEither:262 - applyToEither:执行完的任务test2
```

```java
// acceptEither
public void taskAcceptEither() {
    CompletableFuture<String> supplyFuture01 = CompletableFuture.supplyAsync(()->{
        log.info("执行任务一");
        try {
            Thread.sleep(500);
        } catch (Exception e) {

        }
        return "test1";});
    CompletableFuture<String> supplyFuture02 = CompletableFuture.supplyAsync(()->{
        log.info("执行任务二");
        try {
            Thread.sleep(200);
        } catch (Exception e) {

        }
        return "test2";}, threadPool);

    CompletableFuture<Void> taskAcceptEither = supplyFuture01.acceptEitherAsync(supplyFuture02, (f)->{
        log.info("任务处理完毕------执行结果{}", f);// 能接收无返回
    }, threadPool);
    log.info("taskAcceptEither:{}", taskAcceptEither.join());
}

[2023-07-26 21:17:50.124][INFO ][pool-2-thread-1] com.tqy.toolsct.thread.CFLearn01 lambda$taskAcceptEither$28:275 - 执行任务二
[2023-07-26 21:17:50.124][INFO ][ForkJoinPool.commonPool-worker-9] com.tqy.toolsct.thread.CFLearn01 lambda$taskAcceptEither$27:267 - 执行任务一
[2023-07-26 21:17:50.338][INFO ][pool-2-thread-2] com.tqy.toolsct.thread.CFLearn01 lambda$taskAcceptEither$29:284 - 任务处理完毕------执行结果test2
[2023-07-26 21:17:50.340][INFO ][Test worker] com.tqy.toolsct.thread.CFLearn01 taskAcceptEither:286 - taskAcceptEither:null

```

```java
public void taskRunEither() {
    CompletableFuture<String> supplyFuture01 = CompletableFuture.supplyAsync(()->{
        log.info("执行任务一");
        try {
            Thread.sleep(500);
        } catch (Exception e) {

        }
        return "test1";});
    CompletableFuture<String> supplyFuture02 = CompletableFuture.supplyAsync(()->{
        log.info("执行任务二");
        try {
            Thread.sleep(200);
        } catch (Exception e) {

        }
        return "test2";}, threadPool);

    CompletableFuture<Void> taskRunEither = supplyFuture01.runAfterEither(supplyFuture02, ()->{
        log.info("任一个任务处理完毕------");
    });
    log.info("taskRunEither:{}", taskRunEither.join());
}

[2023-07-26 21:19:08.341][INFO ][ForkJoinPool.commonPool-worker-9] com.tqy.toolsct.thread.CFLearn01 lambda$taskRunEither$30:291 - 执行任务一
[2023-07-26 21:19:08.341][INFO ][pool-2-thread-1] com.tqy.toolsct.thread.CFLearn01 lambda$taskRunEither$31:299 - 执行任务二
[2023-07-26 21:19:08.551][INFO ][pool-2-thread-1] com.tqy.toolsct.thread.CFLearn01 lambda$taskRunEither$32:308 - 任一个任务处理完毕------
[2023-07-26 21:19:08.556][INFO ][Test worker] com.tqy.toolsct.thread.CFLearn01 taskRunEither:310 - taskRunEither:null
```

（6）thenCompose

| 接口方法 | 接口说明 |
| --- | --- |
| <U> CompletableFuture<U> thenCompose(Function<? super T, ? extends CompletionStage<U>> fn) | 等待第一个阶段的完成， 它的结果传给一个指定方法，它的结果就是返回的CompletableFuture的结果 |

```java
public void taskThenCompose() {
    CompletableFuture<String> task1 = CompletableFuture.supplyAsync(()->{
        log.info("task1执行");
        return "task1";
    });

    CompletableFuture<String> task2 = CompletableFuture.supplyAsync(()->{
        log.info("task2执行");
        return "task2";
    }).thenCompose(f->{
        log.info("接收到任务:{}", f);
        return task1;
    });

    log.info("taskCompose:{}", task2.join());// 仍然返回task1
}

[2023-07-26 21:36:00.754][INFO ][ForkJoinPool.commonPool-worker-9] com.tqy.toolsct.thread.CFLearn01 lambda$taskThenCompose$33:315 - task1执行
[2023-07-26 21:36:00.754][INFO ][ForkJoinPool.commonPool-worker-2] com.tqy.toolsct.thread.CFLearn01 lambda$taskThenCompose$34:320 - task2执行
[2023-07-26 21:36:00.763][INFO ][ForkJoinPool.commonPool-worker-2] com.tqy.toolsct.thread.CFLearn01 lambda$taskThenCompose$35:323 - 接收到任务:task2
[2023-07-26 21:36:00.764][INFO ][Test worker] com.tqy.toolsct.thread.CFLearn01 taskThenCompose:327 - taskCompose:task1
```

（7）allOf、anyOf

| 接口方法 | 说明 |
| --- | --- |
| CompletableFuture<Void> allOf(CompletableFuture<?>... cfs) | 等待所有异步任务执行完，返回CompletableFutre，不带具体异步任务结果（Void） |
| CompletableFuture<Object> anyOf(CompletableFuture<?>... cfs) | 任意一个异步任务执行完毕，返回带有完成异步任务结果的CompletableFutre |

```java
public void taskAllOf() {
    long start = System.currentTimeMillis();
    CompletableFuture<String> task1 = CompletableFuture.supplyAsync(()->{
        log.info("task1执行");
        try {
            Thread.sleep(200);
        } catch (Exception e) {

        }
        return "task1";
    }, threadPool);
    CompletableFuture<String> task2 = CompletableFuture.supplyAsync(()->{
        log.info("task2执行");
        try {
            Thread.sleep(100);
        } catch (Exception e) {

        }
        return "task2";
    }, threadPool);
    CompletableFuture<String> task3 = CompletableFuture.supplyAsync(()->{
        log.info("task3执行");
        try {
            Thread.sleep(300);
        } catch (Exception e) {

        }
        return "task3";
    }, threadPool);
    List<CompletableFuture<String>> futures = new ArrayList<>();
    futures.add(task1);
    futures.add(task2);
    futures.add(task3);

		//等待所有任务执行完
    CompletableFuture<Void> all = CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]));
    // 汇集数据，前面提到的thenApply， allOf执行完毕进行回调将数据汇总
		CompletableFuture<List<String>> allRes = all.thenApply(v-> futures.stream().map(f->{
        try {
            return f.get();
        } catch (InterruptedException | ExecutionException e) {
            e.printStackTrace();
            Thread.currentThread().interrupt();
        }
        return "错误";
    }).collect(Collectors.toList()));
    List<String> res = allRes.join();
    long end = System.currentTimeMillis();

    log.info("耗时:{} res:{}", end-start, res);
}

[2023-07-26 22:00:31.991][INFO ][pool-2-thread-1] com.tqy.toolsct.thread.CFLearn01 lambda$taskAllOf$36:337 - task1执行
[2023-07-26 22:00:31.991][INFO ][pool-2-thread-2] com.tqy.toolsct.thread.CFLearn01 lambda$taskAllOf$37:346 - task2执行
[2023-07-26 22:00:31.991][INFO ][pool-2-thread-3] com.tqy.toolsct.thread.CFLearn01 lambda$taskAllOf$38:355 - task3执行
[2023-07-26 22:00:32.310][INFO ][Test worker] com.tqy.toolsct.thread.CFLearn01 taskAllOf:381 - 耗时:329 res:[task1, task2, task3]
```

```java
public void taskAnyOf() {
    long start = System.currentTimeMillis();
    CompletableFuture<String> task1 = CompletableFuture.supplyAsync(()->{
        log.info("task1执行");
        try {
            Thread.sleep(200);
        } catch (Exception e) {

        }
        return "task1";
    }, threadPool);
    CompletableFuture<String> task2 = CompletableFuture.supplyAsync(()->{
        log.info("task2执行");
        try {
            Thread.sleep(100);
        } catch (Exception e) {

        }
        return "task2";
    }, threadPool);
    CompletableFuture<String> task3 = CompletableFuture.supplyAsync(()->{
        log.info("task3执行");
        try {
            Thread.sleep(300);
        } catch (Exception e) {

        }
        return "task3";
    }, threadPool);
    List<CompletableFuture<String>> futures = new ArrayList<>();
    futures.add(task1);
    futures.add(task2);
    futures.add(task3);

    CompletableFuture<Object> any = CompletableFuture.anyOf(futures.toArray(new CompletableFuture[0]));
    long end = System.currentTimeMillis();
    log.info("耗时:{} res:{}", end-start, any.join());
}
[2023-07-26 22:05:19.524][INFO ][pool-2-thread-2] com.tqy.toolsct.thread.CFLearn01 lambda$taskAnyOf$42:396 - task2执行
[2023-07-26 22:05:19.524][INFO ][pool-2-thread-3] com.tqy.toolsct.thread.CFLearn01 lambda$taskAnyOf$43:405 - task3执行
[2023-07-26 22:05:19.524][INFO ][pool-2-thread-1] com.tqy.toolsct.thread.CFLearn01 lambda$taskAnyOf$41:387 - task1执行
[2023-07-26 22:05:19.638][INFO ][Test worker] com.tqy.toolsct.thread.CFLearn01 taskAnyOf:420 - 耗时:9 res:task2
```