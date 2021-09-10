##### 1. MySQL查询语句的执行流程

mysql的基本架构图：

<img src="C:\Users\localuser\AppData\Roaming\Typora\typora-user-images\image-20210804143751263.png" alt="image-20210804143751263" style="zoom:50%;" />

大体上，MySQL可以分为 Server层和存储引擎层

Server层包含了 连接器、查询缓存、分析器、优化器、执行器，包含了Mysql的核心服务，还有 内置的函数（日期、时间），还有跨存储引擎的功能，包括存储过程、触发器、视图等

存储引擎负责数据的存储和提取，架构模式是插件式的，支持InnoDB、MyISAM、Memory等多个存储引擎

**连接器：** 用于管理连接，检查密码权限、

**分析器：**用于对语句进行语法和词法分析

**优化器：**优化器是在表里面有多个索引的时候，决定使用哪个索引；或者在一个语句有多表关联（join）的时候，决定各个表的连接顺序。

**执行器：**MySQL通过分析器知道了你要做什么，通过优化器知道了该怎么做，于是就进入了执行器阶段，开始执行语句。

	1. 开始执行的时候，判断对表T是否有执行权限，没有就返回，有就继续执行
 	2. 调用InnoDB的引擎接口获取表的数据
 	3. 查询得到的数据作为结果集返回给客户端





##### 2. 更新语句的流程

更新语句除了上述的 查询流程之外，还涉及两个重要的 日志模块

**1. redo log  --- 重做日志**

​	**redo log是 InnoDB特有的日志**

​	mysql更新操作时，如果每一次的更新操作都需要写进磁盘，然后磁盘也要找到对应的那条记录，然后再更新，整个过程IO成本、查找成本都很高。

​	WAL技术：Write-Ahead-Loggin，关键点是先写日志，再写磁盘

​	具体是：在语句更新的时候，InnoDB会将记录写到 redo log里面，并且更新内存，这个时候内存已经更新了，同时，InnoDB会在适当的时间，将这个操作更新到磁盘里面，这个更新磁盘是在系统比较空闲的时间完成的。redo log的大小是固定的，写满了就需要擦除前面的部分，继续写

![image-20210804150856793](C:\Users\localuser\AppData\Roaming\Typora\typora-user-images\image-20210804150856793.png)

​	![image-20210804150904828](C:\Users\localuser\AppData\Roaming\Typora\typora-user-images\image-20210804150904828.png)

![image-20210804150919186](C:\Users\localuser\AppData\Roaming\Typora\typora-user-images\image-20210804150919186.png)



**2. bin log  --- 归档日志**

​	**bin log是server层的日志， 称为归档日志**

**3. 两个日志的区别**

	1. redo log是InnoDB引擎特有的，bin log是mysql 的Server层实现的，所有的引擎都可以使用
 	2. redo log是物理日志，记录的是 “某一个数据页上面的修改”；binlog是逻辑日志，记录的是语句的原始逻辑，比如“给ID=2这一行的c字段加1”
 	3. redo log 是循环写的，空间固定会用完，binlog是可以追加写入的，写到一定的大小切换下一个文件继续写，不会覆盖之前的内容
 	4. 两种日志记录写入磁盘的时间点不同，**binlog只在事务提交完成后一次性**写入，而redo log在上面也说了是在事务进行中不断被写入，这表现为日志并不是随事务提交的顺序进行写入

**4. 语句更新的操作**

​	这里我给出这个update语句的执行流程图，图中浅色框表示是在InnoDB内部执行的，深色框表示是在执行器中执行的。

<img src="C:\Users\localuser\AppData\Roaming\Typora\typora-user-images\image-20210804152038667.png" alt="image-20210804152038667" style="zoom:40%;" />



redo-log在事务提交的时候，由prepare状态改变为 commit，记录在内存中，可以保证 crash-safe情况



参考：https://cloud.tencent.com/developer/article/1417482



































































































































