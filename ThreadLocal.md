`ThreadLocal `使用及原理

1. 思考？对使用10个线程进行时间的格式化！格式化时间对象为 线程不安全的类  `SimpleDateFormat `

（示例网址：https://mp.weixin.qq.com/s?__biz=MzU1NTkwODE4Mw==&mid=2247495485&idx=1&sn=d09a6c32f61699c54c860fd5879ae14b&chksm=fbcf8205ccb80b137afe1b71787cd9b5484c9a0afcd5f22a9c47f1fe72cba879dd5bb23d7384&scene=178&cur_album_id=1856147161684475909#rd）

~~~java
// 定义一个资源类，进行线程的操作
class TimePrint{
//    SimpleDateFormat simpleDateFormat = new SimpleDateFormat("mm:ss");
    // 每一个线程会获取到一个独立的 threadLocal副本进行存储
    ThreadLocal<SimpleDateFormat> local =ThreadLocal.withInitial(()->new SimpleDateFormat("mm:ss"));

    public  void print(Date date){

        SimpleDateFormat simpleDateFormat = local.get();

        String format = simpleDateFormat.format(date);
        System.out.println("时间： " + format);
    }
}

// 线程启动类进行
public class ThreadLocalDateFormat {
    public static void main(String[] args) {
        
        TimePrint timePrint = new TimePrint();  // 只定义了一个对象
// 启动10线程个线程进行操作
//        for (int i = 0; i < 10; i++){
//            int finalI = i;
//            new Thread(()->{
//                timePrint.print(new Date(finalI *1000));
//            }).start();
//        }

        // 使用线程池方式调用任务
        ThreadPoolExecutor executor = new ThreadPoolExecutor(3,3, 60, TimeUnit.SECONDS, new ArrayBlockingQueue<>(10));
        // 执行任务
        for(int i = 0; i < 10; i++){
            int finalI = i;
            executor.execute(()->{
                timePrint.print(new Date(finalI *1000));
            });
        }
    }
}

~~~



##### 1. 线程不安全实现方式

~~~java
/***
上述代码中，只定义了一个 TimePrint timePrint = new TimePrint(); TimePrint对象，
如果资源类中，只定义了一个成员变量simpleDateFormat， 当多个线程去操作时，会并发的进行时间的格式化，导致线程安全问题
*/

class TimePrint{
    
    SimpleDateFormat simpleDateFormat = new SimpleDateFormat("mm:ss");

    public  void print(Date date){
        String format = simpleDateFormat.format(date);
        System.out.println("时间： " + format);
    }
}

~~~



#####  2. 线程安全的实现方式

~~~java
/***
	将资源类中的成员变量去除，改变为使用 ThreadLocal 进行存储 时间格式化类，
	每一个线程调用该函数式，都会拷贝一个 ThreadLocal 的值到线程内部进行存储，避免了线程安全问题
*/


// 定义一个资源类，进行线程的操作
class TimePrint{
    
    // 每一个线程会获取到一个独立的 threadLocal副本进行存储
    ThreadLocal<SimpleDateFormat> local =ThreadLocal.withInitial(()->new SimpleDateFormat("mm:ss"));

    public  void print(Date date){

        SimpleDateFormat simpleDateFormat = local.get();  // 每一个独立的 threadLocal 获取 存储的值
        
        String format = simpleDateFormat.format(date);
        System.out.println("时间： " + format);
    }
}
~~~



##### 3. `ThreadLocal `的使用

`ThreadLocal` 常用的核心方法有三个：

1. **set 方法：用于设置线程独立变量副本。**没有 set 操作的 `ThreadLocal `容易引起脏数据。
2. **get 方法：用于获取线程独立变量副本。**没有 get 操作的 `ThreadLocal `对象没有意义。
3. **remove 方法：用于移除线程独立变量副本。**没有 remove 操作容易引起内存泄漏。



##### 4. `ThreadLocal `的高级用法

​	① 初始化：`initialValue` ，设置初始值，该方法是在 get函数之后执行

​	当使用了 `#threadLocal.set` 方法之后，`initialValue` 方法就不会被执行了

##### 为什么 set 之后，初始化代码就不执行了？

要理解这个问题，需要从 `ThreadLocal.get()` 方法的源码中得到答案，因为初始化方法 `initialValue` 在 `ThreadLocal` 创建时并不会立即执行，而是在调用了 `get` 方法只会才会执行

~~~java
// get 方法

public T get() {
       // 得到当前线程对象
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
    // 判断 ThreadLocal 中是否有数据
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
    // 当ThreadLocal中没有数据时，才会执行初始化方法
        return setInitialValue();
    }

~~~



##### ② 初始化2：`withInitial`方法

`withInitial` 方法的优势在于可以更简单的实现变量初始化，如下代码所示，

~~~java
// 每一个线程会获取到一个独立的 threadLocal副本进行存储
ThreadLocal<SimpleDateFormat> local =ThreadLocal.withInitial(()->new SimpleDateFormat("mm:ss"));
~~~



##### 5. `ThreadLocal`导致的内存溢出问题？

实验步骤;

1. 调节 IDEA 中 运行内存的大小 为50M， `-Xmx50m`
2. 实现具体的问题代码（线程池执行完后，线程具有长生命周期，导致内存不释放，触发内存溢出）

~~~java
/**
 * 演示ThreadLocal的内存溢出问题
 */
public class ThreadLocalOutMemry {

    // 定义一个 ThreadLocal对象
    private static ThreadLocal<MyTask> local = new ThreadLocal<>();

    public static void main(String[] args) throws InterruptedException {

        // 定义线程池
        ThreadPoolExecutor executor = new ThreadPoolExecutor(5, 5, 60, TimeUnit.SECONDS, new ArrayBlockingQueue<>(100));

        // 线程池执行任务
        for (int i = 0; i < 10; i++) {
            executeWork(executor);  // 执行任务
            TimeUnit.SECONDS.sleep(1);
        }
    }

    private static void executeWork(ThreadPoolExecutor executor){
        executor.execute(()->{

            System.out.println("创建对象");

            MyTask myTask = new MyTask();

            // 设置ThreadLocal的值
            local.set(myTask);  // 相当于每一个线程进来都分配 10M 的内存大小

            // 手动去掉对象，等待 GC 进行回收
            myTask = null;

        });
    }

    // 定义一个资源类 分配10m空间
    public static class MyTask{
        private byte[] bytes = new byte[10 * 1024 *1024];
    }
}
~~~

运行结果：

<img src="C:\Users\localuser\AppData\Roaming\Typora\typora-user-images\image-20210803172655403.png" alt="image-20210803172655403" style="zoom:80%;" />



~~~java
/***
	在线程池执行任务时，手动将 MyTask对象设置为 null，但是由于线程池生命走题长，且没有将 ThreadLocalMap中的 value值（10m对象）手动回收，导致内存溢出

*/
// 线程池执行任务
private static void executeWork(ThreadPoolExecutor executor){
    executor.execute(()->{

        System.out.println("创建对象");

        MyTask myTask = new MyTask();

        // 设置ThreadLocal的值
        local.set(myTask);  // 相当于每一个线程进来都分配 10M 的内存大小

        // 手动去掉对象，等待 GC 进行回收
        myTask = null;

    });
}

~~~



##### 6. 分析`ThreadLocal`的内部结构

~~~java
// ThreadLocal 的set函数

public void set(T value) {
    // 1. 获取当前线程
    Thread t = Thread.currentThread();
    // 2. 根据当前线程获取 ThreadLocalMap 对象
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}


/**
	我们可以看到 每一个Thread线程内部有一个数据存储的容器 ThreadLocalMap，
	当执行 ThreadLocal.set的时候，会将值放到ThreadLocalMap容器中

**/



// ThreadLocalMap 是 ThreadLocal中的一个 静态内部类，结构如下：
// ThreadLocal 源码
static class ThreadLocalMap {
    
		// 内部一个 Entry 类， 用于存储 值
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }

        /**
         * The initial capacity -- MUST be a power of two.
         */
        private static final int INITIAL_CAPACITY = 16;

        // ThreadLocalMap中存储了一个 Entry 的数组
        private Entry[] table;
    
    
    // ThreadLocalMap的 set函数
    private void set(ThreadLocal<?> key, Object value) {
		  // 实际存储数据的数组
            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);

            for (Entry e = tab[i]; e != null; e = tab[i = nextIndex(i, len)]) {
                
                ThreadLocal<?> k = e.get();
				// 如果有对应的 key 就直接更新 value 值 
                if (k == key) {
                    e.value = value;
                    return;
                }
                // 发现空位插入 Value
                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }
			// 新建一个 Entry 插入到数组中
            tab[i] = new Entry(key, value);
            int sz = ++size;
        // 判断是否需要进行扩容
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
}

~~~



上述源码可以看出：

`ThreadLocalMap`中有一个Entry数组用于存储所有的数据，而Entry是一个包含key和value的键值对， 其中 key 是 `ThreadLocal` 本身， value 为需要存储的值

其结构如下所示：

<img src="C:\Users\localuser\AppData\Roaming\Typora\typora-user-images\image-20210803193859499.png" alt="image-20210803193859499" style="zoom:80%;" />

其中引用关系如下:

~~~java
// Thread -> ThreadLocalMap -> Entry -> key,value(key 为 ThreadLocal)
// 因此当我们使用线程池来存储对象时，因为线程池有很长的生命周期，所以线程池会一直持有 value 值，
// 那么垃圾回收器就无法回收 value，所以就会导致内存一直被占用，从而导致内存溢出问题的发生。
~~~

##### 7. 解决内存溢出

~~~java
// finnaly中 调用 ThreadLocal的 remove 方法
private static void executeWork(ThreadPoolExecutor executor){
    executor.execute(()->{

        System.out.println("创建对象");

        try{

            MyTask myTask = new MyTask();

            // 设置ThreadLocal的值
            local.set(myTask);  // 相当于每一个线程进来都分配 10M 的内存大小

            // 手动去掉对象，等待 GC 进行回收
            // myTask = null;
            // 业务代码--
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }finally {
            // 最后处理完需要清空 ThreadLocalMap 中的 key, 这样会回收内存
            local.remove();
        }
    });
}

// remove方法
public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null)
        m.remove(this);  // 删除 ThreadLocalMap中的 ThreadLocal引用
}
~~~



##### 8. ThreadLocal 的应用场景之 数据库连接池

###### 什么是数据库连接池？

数据库连接池就是用来分配、管理、释放数据库连接的，当然使用普通的JDBC也可以实现 数据库的连接，但是，在使用JDBC的过程中，每次请求都需要进行 新建连接，关闭连接，这样的操作是非常的消耗性能，尤其是在高并发的场景下。

数据库连接池如何实现数据库连接的重复使用，降低性能消耗？

数据库连接池就是新建并且保存了一系列的连接，防止每次请求的时候进行 创建连接，关闭连接的资源消耗

![image-20210905111328310](C:\Users\localuser\AppData\Roaming\Typora\typora-user-images\image-20210905111328310.png)



ThreadLocal在数据库连接池中的作用？

ThreadLocal主要是要保证一个线程的多个dao操作，使用的是同一个Connection，实现事务（如果要保证使用同一个 Connection，也可以使用参数传递，但是不推荐）

数据库连接池，是将connection放进threadlocal里的，以保证每个线程从连接池中获得的都是线程自己的connection。



















































