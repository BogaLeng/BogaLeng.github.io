---
title: Java-获取线程执行结果
tags:
  - Java
createTime: 2025/04/25 19:55:45
permalink: /article/d8adnssp/
---
在上文[Java线程池](/article/exgeihpe/)中，我们使用可以使用线程池来提交任务。

不过，倘若需要任务的执行结果呢？该如何操作？

按照上文我们提交的任务，都是继承了Thread类，或者实现了Runnable接口的。如果需要获取执行结果，就必须通过共享变量或者线程通信的方式来达到目的，这样使用起来就比较麻烦。

## 一、Callable接口

Callable 位于 `java.util.concurrent` 包下，也是一个接口，它定义了一个 `call()` 方法：

```java
public interface Callable<V> {
    V call() throws Exception;
}
```

我们可以通过向线程池提交一个实现了Callable接口的任务，然后就可以获取其运行结果：

```java
// 创建一个包含5个线程的线程池
ExecutorService executorService = Executors.newFixedThreadPool(5);

// 创建一个Callable任务
Callable<String> task = new Callable<String>() {
    public String call() {
        return "Hello from " + Thread.currentThread().getName();
    }
};

// 提交任务到ExecutorService执行，并获取Future对象
Future[] futures = new Future[10];
for (int i = 0; i < 10; i++) {
    futures[i] = executorService.submit(task);
}

// 通过Future对象获取任务的结果
for (int i = 0; i < 10; i++) {
    System.out.println(futures[i].get());
}

// 关闭ExecutorService
executorService.shutdown();
```

如果觉得这样比较啰嗦，还可以采用lambda表达式：

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;

public class callableDemo {
    public static void main(String[] args) {
        // 创建一个包含5个线程的线程池
        ExecutorService executorService = Executors.newFixedThreadPool(5);


        // 提交任务到ExecutorService执行，并获取Future对象
        Future[] futures = new Future[10];
        for (int i = 0; i < 10; i++) {
        	//使用lambda表达式创建
            futures[i] = executorService.submit(() -> "Hello From " + Thread.currentThread().getName());
        }

        // 通过Future对象获取任务的结果
        for (int i = 0; i < 10; i++) {
            try {
                System.out.println(futures[i].get() + " ");
            } catch (Exception e) {
                e.printStackTrace();
            }
        }

        // 关闭ExecutorService
        executorService.shutdown();
    }
}
```

## 二、FutureTask

由于 Future 只是一个接口，如果直接 new 的话，编译器是会有一个 ⚠️ 警告的，它会提醒我们最好使用 `FutureTask`。

### 2.1 FutureTask的作用

`FutureTask` 既可以作为 Runnable 被线程执行，又可以作为 Future 得到 Callable 的返回值。

示例程序：

```java
    public static void main(String[] args) {
        int times = 100;
        FutureTask<String>[] futureTasks = new FutureTask[100];
        while (times-- > 0) {
            final int finalTimes = times;
            System.out.print("Times: " + times);
            Callable<String> callableTask = () -> {
                return "Hello From " + Thread.currentThread().getName();

            };
            futureTasks[100 - times - 1] = new FutureTask<>(callableTask);
            exec.submit(futureTasks[100 - times - 1]);
        }
        for (FutureTask<String> futureTask : futureTasks) {
            try {
                System.out.println(futureTask.get());
            } catch (InterruptedException | ExecutionException e) {
                e.printStackTrace();
            }
        }
    }
```

