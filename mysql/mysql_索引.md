##### 1. mysql索引的结构：

mysql数据库索引使用的是B+树。

B+树是B树的一种变形树，其只有在底层的叶子节点上保存数据，**非叶子节点**上只保存索引，不保存数据，同时叶子节点上维护了顺序访问的指针。

B树，所有节点上都存储数据和索引，且在叶子节点没有维护顺序访问的指针，可能会导致的问题如下：

 	1. 如果数据量过大，导致每一页存储的索引比较少，这样会导致树的深度加大，查询时IO操作次数多，影响查询效率，B+树中数据都存储在叶子节点，非叶子节点只存储索引，这样会降低树的高度
 	2. B树中没有维护顺序访问的指针，当出现区间访问时，需要多次查询数据，B+树中只指向了相邻的节点，只需要进行遍历即可



##### 2. 为啥不使用 Hash表？

	1. Hash不支持范围查询
 	2. 如果是等值查询，哈希索引有明显的优势，只需要经过一次算法查找就可以得到所需的键值，但是存在hash碰撞的问题
 	3. 哈希索引没办法利用索引完成排序
 	4. 哈希索引不支持多列联合索引的最左匹配规则



##### 3. MyISAM引擎的底层结构 （底层都是B+树结构）

​	MyISAM索引文件和数据文件是分离的，叶子节点存放的是数据文件的相对地址，不是数据文件

​	MyISAM 主键的索引文件结构与 非主键的索引文件结构相同，叶子节点都是存储的 数据文件的相对地址，不是数据文件，因为都是采用 B+树，非叶子节点都是存储的索引地址，且叶子节点维护了顺序访问的指针

<img src="C:\Users\localuser\AppData\Roaming\Typora\typora-user-images\image-20210805150843134.png" alt="image-20210805150843134" style="zoom:45%;" />      <img src="C:\Users\localuser\AppData\Roaming\Typora\typora-user-images\image-20210805151002801.png" alt="image-20210805151002801" style="zoom:44%;align: right" />



##### 4. InnoDB引擎的底层结构（底层使用的是B+树）

​	InnoDB 的索引可以分为聚集索引和非聚集索引

聚集索引：

 1. 聚集索引的数据行的物理顺序与列值的逻辑顺序相同，一个表只能拥有一个聚集索引

 2. 聚集索引一般就是主键索引

 3. 聚集索引是一种存储方式，**索引的叶子节点就是对应的数据节点**，叶子节点上都包含了完整的数据记录

    <img src="C:\Users\localuser\AppData\Roaming\Typora\typora-user-images\image-20210805152707057.png" alt="image-20210805152707057" style="zoom:70%;" />		

​	

非聚集索引：

	1. 非聚集索引中，叶子节点存储的是 主键的值，而不是所有的数据，所以会有回表的过程
 	2. 非聚集索引中，索引的逻辑顺序与磁盘上的物理存储顺序不同，一个表可以拥有多个非聚集索引



##### 5. 覆盖索引

​	覆盖索引的意思是指一个查询语句的执行只用从索引中就可以取得，不必从数据表中读取，这样的索引称为覆盖索引，这样可以减少回表的过程



##### 6. 索引下推 

​	索引下推 （index condition pushdown）简称 ICP， 是mysql5.6的版本上推出，用于查询优化

- 在不使用ICP的情况下，在使用**非主键索引（又叫普通索引或者二级索引）**进行查询时，存储引擎通过索引检索到数据，然后返回给MySQL服务器，服务器然后判断数据是否符合条件 。

- 在使用ICP的情况下，如果存在某些被索引的列判断条件时，一部分判断条件传递给存储引擎，然后由存储引擎通过判断索引是否符合MySQL服务器传递的条件，只有当索引符合条件时才会将数据检索出来返回给MySQL服务器 。

- **索引条件下推优化可以减少存储引擎查询基础表的次数，也可以减少MySQL服务器从存储引擎接收数据的次数**。

  

适用条件：

	1. 对于InnDB引擎只适用于二级索引，因为InnDB的聚簇索引会将整行数据读到InnDB的缓冲区，这样一来索引条件下推的主要目的减少IO次数就失去了意义。因为数据已经在内存中了，不再需要去读取了
 	2. 引用子查询的条件不能下推
 	3. 调用存储过程的条件不能下推，存储引擎无法调用位于MySQL服务器中的存储过程
 	4. 触发条件不能下推。



##### 7. 为什么使用联合索引

	1. 减小开销，建一个联合索引(col1,col2,col3)，实际相当于建了(col1),(col1,col2),(col1,col2,col3)三个索引。每多一个索引，都会增加写操作的开销和磁盘空间的开销。对于大量数据的表，使用联合索引会大大的减少开销！  
~~~
2. 覆盖索引。对联合索引(col1,col2,col3)，如果有如下的sql: select col1,col2,col3 from test where col1=1 and col2=2。那么MySQL可以直接通过遍历索引取得数据，而无需回表，这减少了很多的随机io操作。减少io操作，特别的随机io其实是dba主要的优化策略。所以，在真正的实际应用中，覆盖索引是主要的提升性能的优化手段之一。
~~~

~~~
3. 索引列越多，通过索引筛选出的数据越少。  
~~~



#####  8.建立索引指南 

参考：https://www.cnblogs.com/Qian123/p/5666569.html 

 	1. 对于需要在指定范围内的快速或频繁查询的数据列;
 	2. 经常用在WHERE子句中的数据列。
 	3. 经常出现在关键字order by、group by、distinct后面的字段，建立索引。如果建立的是复合索引，索引的字段顺序要和这些关键字后面的字段顺序一致，否则索引不会被使用。
 	4. 对于定义为text、image和bit的数据类型的列不要建立索引。（占用的空间太大，需要综合考虑）
 	5. 限制表上的索引数目。对一个存在大量更新操作的表，所建索引的数目一般不要超过3个，最多不要超过5个。索引虽说提高了访问速度，但太多索引会影响数据的更新操作。
 	6.  对复合索引，按照字段在查询条件中出现的频度建立索引。



 1. 对查询进行优化，要尽量**避免全表扫描，首先应考虑在 where 及 order by 涉及的列上建立索引**。（排序是非常消耗时间性能）

 2. 应尽量**避免在 where 子句中对字段进行 null 值判断**，否则将导致引擎放弃使用索引而进行全表扫描

 3. 应尽量**避免在 where 子句中使用 != 或 <> 操作符**，否则将引擎放弃使用索引而进行全表扫描。

 4. 应尽量**避免在 where 子句中使用 or 来连接条件**，如果一个字段有索引，一个字段没有索引，将导致引擎放弃使用索引而进行全表扫描

    ~~~sql
    select id from t where num=10 or Name = 'admin'  -- 该SQL 导致放弃建立的索引
    
    -- 可以修改为 union 连接两部分的值
    select id from t where num = 10
    union all
    select id from t where Name = 'admin'
    
    ~~~

5. **in 和 not in 也要慎用**，否则会导致全表扫描，对于**连续的数值，能用 between** 就不要用 in 了，很多时候**用 exists 代替 in** 是一个好的选择：

   ~~~sql
   -- 语句1
   select num from a where num in(select num from b)
   
   -- 替换为
   select num from a where exists(select 1 from b where num=a.num)
   ~~~

6. 应尽量避免在 where 子句中对字段进行表达式操作，这将导致引擎放弃使用索引而进行全表扫描。

   ~~~sql
   --- 原始语句1
   select id from t where num/2 = 100
   
   --- 修改为
   select id from t where num = 100*2
   ~~~

7. 应尽量避免在where子句中对字段进行函数操作，这将导致引擎放弃使用索引而进行全表扫描。

   ~~~sql
   select id from t where substring(name,1,3) = ’abc’       -– name以abc开头的id
   select id from t where datediff(day,createdate,’2005-11-30′) = 0    -–‘2005-11-30’    --生成的id
   
   --- 修改
   select id from t where name like 'abc%'
   select id from t where createdate >= '2005-11-30' and createdate < '2005-12-1'
   ~~~

 8. 在索引列上进行数据类型的隐形转换也会导致索引失效，比如字符串类型一定要加上 引号



##### 9. 前缀索引

前缀索引是对文本或者字符串的前几个字符串建立索引，这样索引的长度更短，查询快

建立语法：

~~~sql
Alter Table table_name Add KEY(Column_name(Prefix_length));
~~~

其中 prefix_length 长度需要计算：

1. 首先计算索引列对于全列的区分度

   ~~~SQL 
   select count(Distinct column_name) / count(*) from table_name;
   ~~~

2. 再计算前缀长度取多少与 全列计算的区分度相等即可

   ~~~SQL
   select count(Distinct left(column_name, prefix_length)) / count(*) from table_name
   ~~~

   





























































































































































































































