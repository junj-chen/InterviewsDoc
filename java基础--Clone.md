### 1. java基础--Clone

参考：http://www.cyc2018.xyz/Java/Java%20%E5%9F%BA%E7%A1%80.html#clone

​		https://www.cnblogs.com/Qian123/p/5710533.html  （未看）

1. clone() 是 Object 的 protected 方法，它不是 public，一个类不显式去重写 clone()，其它类就不能直接去调用该类实例的 clone() 方法。

2. 重写 Object 的 clone方法后，进行调用克隆也会报错

   ~~~java
   java.lang.CloneNotSupportedException: CloneExample
   ~~~

   以上抛出了 CloneNotSupportedException，这是因为 CloneExample 没有实现 Cloneable 接口。

   Cloneable 接口只是规定，如果一个类没有实现 Cloneable 接口又调用了 clone() 方法，就会抛出 CloneNotSupportedException。

3. 注意数组的拷贝

   ~~~java
       public static int get(){
   
           int[] a = new int[]{3,2,1};
   
           int[] clone = a.clone();
           clone[0] = 100;
           System.out.println(a[0]);  // 3，表示更改clone后的对象，对原数组没有改变
   
           int[] ints = new int[a.length];
   
           ints = a;  //浅拷贝，指向同一个内存地址
   
           Arrays.sort(ints);  // 对 引用的对象进行排序，会影响原数组
   
           System.out.println(a[0]);  // 1，原数组被更改
   
           a[0] = 333;  //修改原数组的值
   
           System.out.println(clone[0]);  // 100，拷贝后的对象不变
   
           System.out.println(ints[0]);  // 333 浅拷贝的对象值改变
           
           return 0;
   
       }
   
   ~~~

   

### 2. java基础--访问权限

参考：http://www.cyc2018.xyz/Java/Java%20%E5%9F%BA%E7%A1%80.html#六、继承

Java 中有三个访问权限修饰符：private、protected 以及 public，如果不加访问修饰符，表示包级可见。

protected 用于修饰成员，表示在继承体系中成员对于子类可见，但是这个访问修饰符对于类没有意义。















