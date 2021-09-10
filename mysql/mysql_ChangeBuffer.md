##### 1. `ChangeBuffer`介绍

`ChangeBuffer`是一种特殊的数据结构，应用在数据页更新的时候。在更新数据页的时候，二级索引页没有在缓存池中，`ChangeBuffer`可以存储二级索引页的更新操作，当下次查询用到该数据页的时候，将数据页从磁盘读入内存，然后执行 `ChangeBuffer`中存储的更新操作（合并）。该方式可以有效减少磁盘读取数据的开销。

结构;

<img src="C:\Users\localuser\AppData\Roaming\Typora\typora-user-images\image-20210804110307654.png" alt="image-20210804110307654" style="zoom:50%;" />

从结构可知：`ChangeBuffer`是 `BufferPool`的一部分



##### 2. 合并时机：

	1. 虽然名字叫作change buffer，实际上它是可以持久化的数据。也就是说，change buffer在内存中有拷贝，也会被写入到磁盘上
 	2. 将`ChangeBuffer`中的操作应用到原数据页的过程叫做 merge， 除了访问原数据页会触发 merge外，系统后台进程也会定期 merge，当数据库正常关闭时，也会执行 merge 操作



##### 3. `ChangeBuffer`的好处：

 1. 更新操作先缓存到 `changeBuffer`，可以有效减少读磁盘的开销

 2. 数据页读入内存需要占据 `bufferPool `的空间，这种方式也可以有效提高 内存利用率

    

##### 4. 什么条件下使用 `ChangeBuffer`？

 	1. 需要操作的数据页没有在内存中，（如果已经在内存中，则直接更新即可，无需记录操作）
      	2. 数据页不在内存中，对于非聚集索引大致可以分为两类，唯一索引和普通索引 （在插入数据情景下）

小总结：`ChangeBuffer`是缓存更新操作，防止大量的读取磁盘和有效减少内存的占用，并且其适用于非聚集索引中的 普通索引，不适用于 唯一索引



##### 5. 普通索引下的使用场景？

**普通索引的所有场景，使用change buffer都可以起到加速作用吗？**

因为merge操作是在数据进行真正更新的时候，`ChangeBuffer`目的是将记录的更新操作进行缓存，所以在一个数据页做merge之前，`ChangeBuffer`记录的变更越多，获得的收益越大！

1. 因此对于写多读少的业务而言，页面写完之后马上被访问的概率小，此时的 `ChangeBuffer`的效果最好，比如 账单类，日志类
2. 但是一个业务如果写入之后马上做读取，那么即使满足了条件，将更新先记录在change buffer，但之后由于马上要访问这个数据页，会立即触发merge过程。这样随机访问IO的次数不会减少，反而增加了change buffer的维护代价。所以，对于这种业务模式来说，change buffer反而起到了副作用





















































































































