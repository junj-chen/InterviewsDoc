java 8 中的异步实现

1. jdk1.8之前的Future

JDK并发包中的 Future，通过向线程池中提交任务 Callable 接口的实现，通过future 中的 get获取返回结果，jdk1.8之前并不是真的异步，调用 get方法的时候是通过 阻塞获取，也可以调用非阻塞的方法`isDone`来确定操作是否完成，`isDone`这种方式有点儿类似下面的过程：

![image-20210911143248014](C:\Users\localuser\AppData\Roaming\Typora\typora-user-images\image-20210911143248014.png)



Future 缺点：

（1）不支持手动完成

这个意思指的是，我提交了一个任务，但是执行太慢了，我通过其他路径已经获取到了任务结果，现在没法把这个任务结果，通知到正在执行的线程，所以必须主动取消或者一直等待它执行完成。

（2）不支持进一步的非阻塞调用

这个指的是我们通过Future的get方法会一直阻塞到任务完成，但是我还想在获取任务之后，执行额外的任务，因为Future不支持回调函数，所以无法实现这个功能。

（3）不支持链式调用

这个指的是对于Future的执行结果，我们想继续传到下一个Future处理使用，从而形成一个链式的pipline调用，这在Future中是没法实现的。

（4）不支持多个Future合并

比如我们有10个Future并行执行，我们想在所有的Future运行完毕之后，执行某些函数，是没法通过Future实现的。

（5）不支持异常处理

Future的API没有任何的异常处理的api，所以在异步运行时，如果出了问题是不好定位的。



2. 直到jdk1.8才算真正支持了异步操作，jdk1.8中提供了`lambda`表达式，使得java向函数式语言又靠近了一步。借助jdk原生的`CompletableFuture`可以实现异步的操作，同时结合`lambada`表达式大大简化了代码量。

1. 基本使用：

~~~java
// 具体的函数
// 以下的方法是使用异步编程最简单的步骤，CompletableFuture.get()的方法会阻塞直到任务完成，这其实还是同步的概念，这对于一个异步系统是不够的，因为真正的异步是需要支持回调函数，
static CompletableFuture<Void>  runAsync(Runnable runnable)
static CompletableFuture<Void>  runAsync(Runnable runnable, Executor executor)
static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier)
static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier, Executor executor)   // 提供使用 Executor 线程池的方法
~~~



示例：

```java
public static void demo2() throws ExecutionException, InterruptedException {

    CompletableFuture<String> future = CompletableFuture.supplyAsync(new Supplier<String>() {
        @Override
        public String get() {
            System.out.println(Thread.currentThread().getName()+"正在执行一个没有返回值的异步任务。");
            try {
                TimeUnit.SECONDS.sleep(2);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            return "0000000000000";
        }
    });

    System.out.println(future.get());  // get方法 实现一个同步的阻塞
    System.out.println(Thread.currentThread().getName()+" 结束。");
    
}
```



2. 真正的异步使用

真正的异步是需要支持回调函数，这样以来，我们就可以直接在某个任务干完之后，接着执行回调里面的函数，从而做到真正的异步概念。

在CompletableFuture里面，我们通过thenApply(), thenAccept()，thenRun()方法，来运行一个回调函数。

（1）thenApply()

这个方法，其实用过函数式编程的人非常容易理解，类似于scala和spark的map算子，通过这个方法可以进行多次链式转化并返回最终的加工结果

```java
public static void asyncCallback() throws ExecutionException, InterruptedException {

        CompletableFuture<String> task=CompletableFuture.supplyAsync(new Supplier<String>() {
            @Override
            public String get() {
                System.out.println(getThreadName()+"supplyAsync");
                return "123";
            }
        });

        CompletableFuture<Integer> result1 = task.thenApply(number->{
            System.out.println(getThreadName()+"thenApply1");
            return Integer.parseInt(number);
        });

        CompletableFuture<Integer> result2 = result1.thenApply(number->{
            System.out.println(getThreadName()+"thenApply2");
            return number*2;
        });

        System.out.println(getThreadName()+" => "+result2.get());

    }
```



（2）thenAccept()

这个方法，可以接受Futrue的一个返回值，但是本身不在返回任何值，适合用于多个callback函数的最后一步操作使用。



（3） thenRun()

这个方法与上一个方法类似，一般也用于回调函数最后的执行，但这个方法不接受回调函数的返回值，纯粹就代表执行任务的最后一个步骤：



（4）thenCompose合并两个有依赖关系的CompletableFutures的执行结果

CompletableFutures在执行两个依赖的任务合并时，会返回一个嵌套的结果列表，为了避免这种情况我们可以使用thenCompose来返回，直接获取最顶层的结果数据即可



具体参考：https://cloud.tencent.com/developer/article/1366581





































































