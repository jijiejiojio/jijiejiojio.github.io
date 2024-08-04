---
title: MySQL索引
date: 2021-09-06 16:47:29
tags: MySQL
toc: true
---

# Mysql 逻辑架构简介 

## 整体架构图 



![image-20210903095621010](https://gitee.com/jijiejiojio/pic/raw/master/img/image-20210903095621010.png)

### 连接层

> 最上层是一些客户端和连接服务，包含本地 sock 通信和大多数基于客户端/服务端工具实现的类似于 tcp/ip 的通信。主要完成一些类似于连接处理、授权认证、及相关的安全方案。在该层上引入了线程池的概念，为通过认证安全接入的客户端提供线程。同样在该层上可以实现基于 SSL 的安全链接。服务器也会为安全接入的每个客户端验证它所具有的操作权限。

### 服务层

> Management Serveices & Utilities ：系统管理和控制工具 
>
> SQL Interface：SQL 接口。接受用户的 SQL 命令，并且返回用户需要查询的结果。比如 select from就是调用 SQL Interface 
>
> Parser ：解析器。 SQL 命令传递到解析器的时候会被解析器验证和解析 
>
> Optimizer ：查询优化器。 SQL 语句在查询之前会使用查询优化器对查询进行优化，比如有where 条件时，优化器来决定先投影还是先过滤。 
>
> Cache 和 Buffer ：查询缓存。如果查询缓存有命中的查询结果，查询语句就可以直接去查询缓存中取数据。这个缓存机制是由一系列小缓存组成的。比如表缓存，记录缓存，key 缓存，权限缓存等 

### 引擎层

> 存储引擎层，存储引擎真正的负责了 MySQL 中数据的存储和提取，服务器通过 API 与存储引擎进行通信。不同的存储引擎具有的功能不同，这样我们可以根据自己的实际需要进行选取。

### 存储层

> 数据存储层，主要是将数据存储在运行于裸设备的文件系统之上，并完成与存储引擎的交互。

## show profile 

利用 show profile 可以查看 sql 的执行周期！

## mysql 的查询流程大致是： 

mysql 客户端通过协议与 mysql 服务器建连接，发送查询语句，先检查查询缓存，如果命中，直接返回结果，否则进行语句解析,也就是说，在解析查询之前，服务器会先访问查询缓存(query cache)——它存储 SELECT 语句以及相应的查询结果集。如果某个查询结果已经位于缓存中，服务器就不会再对查询进行解析、优化、以及执行。它仅仅将缓存中的结果返回给用户即可，这将大大提高系统的性能。 

语法解析器和预处理：首先 mysql 通过关键字将 SQL 语句进行解析，并生成一颗对应的“解析树”。mysql 解析器将使用 mysql 语法规则验证和解析查询；预处理器则根据一些 mysql 规则进一步检查解析数是否合法。 

查询优化器当解析树被认为是合法的了，并且由优化器将其转化成执行计划。一条查询可以有很多种执行方式，最后都返回相同的结果。优化器的作用就是找到这其中最好的执行计划。 

然后，mysql 默认使用的 BTREE 索引，并且一个大致方向是:无论怎么折腾 sql，至少在目前来说，mysql 最多只用到表中的一个索引。

## SQL 的执行顺序 

手写的顺序

![image-20210903100540095](https://gitee.com/jijiejiojio/pic/raw/master/img/image-20210903100540095.png)

真正执行的顺序： 

随着 Mysql 版本的更新换代，其优化器也在不断的升级，优化器会分析不同执行顺序产生的性能消耗不同而动态调整执行顺序。下面是经常出现的查询顺序： 

![image-20210903100612461](https://gitee.com/jijiejiojio/pic/raw/master/img/image-20210903100612461.png)

![image-20210903100624697](https://gitee.com/jijiejiojio/pic/raw/master/img/image-20210903100624697.png)

# 存储引擎

MyISAM 和 InnoDB

|                | InnoDB                                                       | MyISAM                                                 |
| -------------- | ------------------------------------------------------------ | ------------------------------------------------------ |
| 外键           | 支持                                                         | 不支持                                                 |
| 事务           | 支持                                                         | 不支持                                                 |
| 行表锁         | 行锁，操作时只锁某一行，不对其他行有影响，适合高并发操作     | 表锁，即使操作一条记录也会锁住整张表，不适合高并发操作 |
| 缓存           | 不仅缓存索引还要缓存真实数据，对内存要求较高，而且内存大小对性能有决定性的影响 | 只缓存索引，不缓存真实数据                             |
| 关注点         | 并发写，事务，资源                                           | 读性能                                                 |
| 自带系统表使用 | 否                                                           | 是                                                     |
| 默认使用       | 是                                                           | 否                                                     |

# 常见的join查询图

![image-20210903103811921](https://gitee.com/jijiejiojio/pic/raw/master/img/image-20210903103811921.png)

# 索引优化分析

## 索引的概念

MySQL 官方对索引的定义为：索引（Index）是帮助 MySQL 高效获取数据的数据结构。可以得到索引的本质： **索引是数据结构**。可以简单理解为**排好序的快速查找数据结构。**

在数据之外，数据库系统还维护着满足特定查找算法的数据结构，这些数据结构以某种方式引用（指向）数据，这样就可以在这些数据结构上实现高级查找算法。这种数据结构，就是索引。下图就是一种可能的索引方式示例

![image-20210903110048494](https://gitee.com/jijiejiojio/pic/raw/master/img/image-20210903110048494.png)

左边是数据表，一共有两列七条记录，最左边的是数据记录的物理地址。为了加快 Col2 的查找，可以维护一个右边所示的二叉查找树，每个节点分别包含索引键值和一个指向对应数据记录物理地址的指针，这样就可以运用二叉查找在一定的复杂度内获取到相应数据，从而快速的检索出符合条件的记录。

一般来说索引本身也很大，不可能全部存储在内存中，因此索引往往以索引文件的形式存储的磁盘上。

## 优缺点

优点

提高数据检索的效率，降低数据库的IO成本。 

通过索引列对数据进行排序，降低数据排序的成本，降低了CPU的消耗。

缺点

虽然索引大大提高了查询速度，同时却会降低更新表的速度，如对表进行INSERT、UPDATE和DELETE。因为更新表时，MySQL不仅要保存数据，还要保存一下索引文件每次更新添加了索引列的字段，都会调整因为更新所带来的键值变化后的索引信息。 

实际上索引也是一张表，该表保存了主键与索引字段，并指向实体表的记录，所以索引列也是要占用空间的。

## MySQL索引

### btree

### b+tree

### 聚簇索引和非聚簇索引

聚簇索引并不是一种单独的索引类型，而是一种数据储存方式。术语‘聚簇’表示数据行和相邻的键值聚簇的存储在一起。如下图，左侧的索引就是聚簇索引，因为数据行在磁盘的排列和索引排序保持一致。

![image-20210906170235746](https://gitee.com/jijiejiojio/pic/raw/master/img/image-20210906170235746.png)

聚簇索引的好处：

按照聚簇索引排列顺序，查询显示一定范围数据的时候，由于数据都是紧密相连的，数据库不用从多个数据块中提取数据，所以节省了大量IO操作。

聚簇索引的限制：

> 对于mysql数据库目前只有innodb数据库引擎支持聚簇索引，二Myisam并不支持聚簇索引。
>
> 由于数据物理存储排序方式只能有一种，所以每个mysql的表只能有一个聚簇索引。一般情况下就是该表的主键。
>
> 为了充分利用聚簇索引的特性，所以innodb表的主键列尽量选用有序的顺序id，而不建议用无序的id，比如UUID。

## MySQL索引分类

### 单值索引

概念：即一个索引只包含单个列，一个表可以有多个单列索引

语法：create index idx_name on table(name)

### 唯一索引

概念：索引的值必须唯一，但允许有空值

语法：create unique index idx_name on table(name)

### 主键索引

概念：设定为主键后数据库会自动建立索引，innodb为聚簇索引

语法：alter table tablename add primary key tablename(id)

​			alter table tablename drop primary key

### 复合索引

概念：即一个索引包含多个列

语法：create index idx_name_age on table(name,age)

### 基本语法：

创建：create [unique] index indexname on tableName(column)

删除：drop index indexname on tablename

查看：show index from tableName\G

使用alter命令：

alter table tblName add primary key (column):添加一个主键，这意味着索引值必须是唯一的，且不能为null 

alter table tblName add index indexName(column)：添加普通索引，索引值可以出现多次

alter table tblName add fulltext indexName(column)：指定索引为full text，用于全文索引

## 索引创建的时机

适合创建索引的情况

> 主键主动建立唯一索引
>
> 频繁作为查询条件的字段应该创建索引
>
> 查询中与其他表关联的字段，外键关系建立索引
>
> 单键/组合索引的选择问题，组合索引性价比更高
>
> 查询中排序的字段，排序字段若通过索引去访问将大大提高排序速度
>
> 查询中统计或分组字段

不适合创建索引的情况

> 表记录太少
>
> 经常增删改表或字段
>
> where条件用不到的字段不创建索引
>
> 过滤性不好的不适合创建索引

## Explain 性能分析

使用 EXPLAIN 关键字可以模拟优化器执行 SQL 查询语句，从而知道 MySQL 是如何处理你的 SQL 语句的。分析你的查询语句或是表结构的性能瓶颈。 

用法： Explain+SQL 语句。

Explain 执行后返回的信息：

![image-20210903115907286](https://gitee.com/jijiejiojio/pic/raw/master/img/image-20210903115907286.png)

### id

select 查询的序列号,包含一组数字，表示查询中执行 select 子句或操作表的顺序。

id相同，执行顺序由上至下

explain select * from t1,t2,t3 where t1.id=t2.id and t2.id=t3.id 

![image-20210903142444411](https://gitee.com/jijiejiojio/pic/raw/master/img/image-20210903142444411.png)

![image-20210903141129696](https://gitee.com/jijiejiojio/pic/raw/master/img/image-20210903141129696.png)

id不同，如果是子查询，id的序号会递增，id值越大优先级越高，越被执行

explain select t1.id from t1 where t1.id in
	(select t2.id from t2 where t2.id in 
	(select t3.id from t3 where t3.content='')
	);

![image-20210903142523740](https://gitee.com/jijiejiojio/pic/raw/master/img/image-20210903142523740.png)

![image-20210903141754175](https://gitee.com/jijiejiojio/pic/raw/master/img/image-20210903141754175.png)

id有相同也有不同，如果id相同，认为是同一组，从上往下执行，再所有组中，id越大，优先级越高，越先执行衍生=derived

explain select t2.* from t2,(select * from t3 where t3.content='t3_79') s3 where s3.id=t2.id

![image-20210903142546759](https://gitee.com/jijiejiojio/pic/raw/master/img/image-20210903142546759.png)

![image-20210903142154853](https://gitee.com/jijiejiojio/pic/raw/master/img/image-20210903142154853.png)

关注点：id 号每个号码，表示一趟独立的查询。**一个 sql 的查询趟数越少越好。**

### select_type

select_type 代表查询的类型，主要是用于区别普通查询、联合查询、子查询等的复杂查询。

| select_type 属性     | 含义                                                         |
| -------------------- | ------------------------------------------------------------ |
| SIMPLE               | 简单的 select 查询,查询中不包含子查询或者 UNION              |
| PRIMARY              | 查询中若包含任何复杂的子部分，最外层查询则被标记为 Primary   |
| DERIVED              | 在 FROM 列表中包含的子查询被标记为 DERIVED(衍生) MySQL 会递归执行这些子查询, 把结果放在临时表里。 |
| SUBQUERY             | 在SELECT或WHERE列表中包含了子查询                            |
| DEPEDENT SUBQUERY    | 在SELECT或WHERE列表中包含了子查询,子查询基于外层             |
| UNCACHEABLE SUBQUERY | 无法使用缓存的子查询                                         |
| UNION                | 若第二个SELECT出现在UNION之后，则被标记为UNION； 若UNION包含在FROM子句的子查询中,外层SELECT将被标记为：DERIVED |
| UNION RESULT         | 从UNION表获取结果的SELECT                                    |

### table

这个数据是基于哪张表的

### type

type 是查询的访问类型。是较为重要的一个指标，结果值从最好到最坏依次是：

system>const>**eq_ref**>**ref**>fulltext>ref_or_nukk>index_merge>unique_subquery>index_subquery>**range**>index>all

一般来说，得保证查询知道打到range级别，最好能达到ref。

system

表只有一行记录（等于系统表），这是 const 类型的特列，平时不会出现，这个也可以忽略不计 

const

表示通过索引一次就找到了,const 用于比较 primary key 或者 unique 索引。因为只匹配一行数据，所以很快 

如将主键置于 where 列表中，MySQL 就能将该查询转换为一个常量。

![image-20210903144352260](https://gitee.com/jijiejiojio/pic/raw/master/img/image-20210903144352260.png)

eq_ref

唯一性索引扫描，对于每个索引键，表中只有一条记录与之匹配，常见于主键或唯一索引扫描。

![image-20210903144722891](https://gitee.com/jijiejiojio/pic/raw/master/img/image-20210903144722891.png)

ref

非唯一性索引扫描，返回匹配某个单独值得所有行。本质上也是一种索引访问，它返回所有匹配某个单独值的行，然而，它可能会找到多个符合条件的行，所以他应该属于查找和扫描的混合体。

没用索引前：

![image-20210903145030484](https://gitee.com/jijiejiojio/pic/raw/master/img/image-20210903145030484.png)

添加索引后

![image-20210903145150502](https://gitee.com/jijiejiojio/pic/raw/master/img/image-20210903145150502.png)

range

只检索给定范围的行,使用一个索引来选择行。key 列显示使用了哪个索引一般就是在你的 where 语句中出现了 between、<、>、in 等的查询这种范围扫描索引扫描比全表扫描要好，因为它只需要开始于索引的某一点，而结束语另一点，不用扫描全部索引。

![image-20210903145358786](https://gitee.com/jijiejiojio/pic/raw/master/img/image-20210903145358786.png)

index

出现index是sql使用了索引但是没用通过索引进行过滤，一般是使用了覆盖索引或者是利用索引进行了排序分组。

![image-20210903145605352](https://gitee.com/jijiejiojio/pic/raw/master/img/image-20210903145605352.png)

all 

Full Table Scan，将遍历全表以找到匹配的行。

index_merge 

在查询过程中需要多个索引组合使用，通常出现在有 or 的关键字的 sql 中。

![image-20210903150646306](https://gitee.com/jijiejiojio/pic/raw/master/img/image-20210903150646306.png)

ref_null

对于某个字段既需要关联条件，也需要 null 值得情况下。查询优化器会选择用 ref_or_null 连接查询。

![image-20210903150839576](https://gitee.com/jijiejiojio/pic/raw/master/img/image-20210903150839576.png)

index_subquery

利用索引来关联子查询，不再全表扫描

unique_subquery

该联接类型类似于 index_subquery。 子查询中的唯一索引。

### possible_keys

显示可能应用在这张表中的索引，一个或多个。查询涉及到的字段上若存在索引，则该索引将被列出，**但不一定被查询实际使用**。

### key

实际使用的索引。如果为NULL，则没有使用索引。

### key_len

表示索引中使用的字节数，可通过该列计算查询中使用的索引的长度。 key_len 字段能够帮你检查是否充分的利用上了索引。ken_len 越长，说明索引使用的越充分。

**如何计算：** 

①先看索引上字段的类型+长度比如 int=4 ; varchar(20) =20 ; char(20) =20 

②如果是 varchar 或者 char 这种字符串字段，视字符集要乘不同的值，比如 utf-8 要乘 3,GBK 要乘 2， 

③varchar 这种动态字符串要加 2 个字节 

④允许为空的字段要加 1 个字节 

第一组：key_len=age 的字节长度+name 的字节长度=4+1 + ( 20*3+2)=5+62=67 

第二组：key_len=age 的字节长度=4+1=5

![image-20210903151158008](https://gitee.com/jijiejiojio/pic/raw/master/img/image-20210903151158008.png)

![image-20210903151125065](https://gitee.com/jijiejiojio/pic/raw/master/img/image-20210903151125065.png)

### ref

显示索引的哪一列被使用了，如果可能的话，是一个常数。哪些列或常量被用于查找索引列上的值。

### rows

rows 列显示 MySQL 认为它执行查询时必须检查的行数。越少越好！

### extra

其他的额外重要的信息!

#### using filesort【不好的，需要优化】

说明 mysql 会对数据使用一个外部的索引排序，而不是按照表内的索引顺序进行读取。MySQL 中无法利用索引 

完成的排序操作称为“文件排序”。

![image-20210903183959071](https://gitee.com/jijiejiojio/pic/raw/master/img/image-20210903183959071.png)

#### using temporary【不好的，需要优化】

使了用临时表保存中间结果,MySQL 在对查询结果排序时使用临时表。常见于排序 order by 和分组查询 groupby。

![image-20210903184324977](https://gitee.com/jijiejiojio/pic/raw/master/img/image-20210903184324977.png)

优化：加上索引

![image-20210903184729301](https://gitee.com/jijiejiojio/pic/raw/master/img/image-20210903184729301.png)

#### using index

Using index 代表表示相应的 select 操作中使用了覆盖索引(Covering Index)，避免访问了表的数据行，效率不错！ 

如果同时出现 using where，表明索引被用来执行索引键值的查找;如果没有同时出现 using where，表明索引只是用来读取数据而非利用索引执行查找。 

利用索引进行了排序或分组。

#### using where

表明使用了 where 过滤。

#### using join buffer

使用了连接缓存。

#### impossible where

where 子句的值总是 false，不能用来获取任何元组。

![image-20210903184908526](https://gitee.com/jijiejiojio/pic/raw/master/img/image-20210903184908526.png)

#### select tables optimized away

在没有 GROUPBY 子句的情况下，基于索引优化 MIN/MAX 操作或者对于 MyISAM 存储引擎优化 COUNT(*)操作，不必等到执行阶段再进行计算，查询执行计划生成的阶段即完成优化

在 innodb 中：

![image-20210903185028353](https://gitee.com/jijiejiojio/pic/raw/master/img/image-20210903185028353.png)

在 Myisam 中：

![image-20210903185040754](https://gitee.com/jijiejiojio/pic/raw/master/img/image-20210903185040754.png)

## 单表使用索引常见的索引

全职匹配我最爱：查询的字段按照顺序在索引中都可以匹配到！

> SQL 中查询字段的顺序，跟使用索引中字段的顺序，没有关系。优化器会在不影响 SQL 执行结果的前提下，给你自动地优化

最佳左前缀法则：查询字段与索引字段顺序的不同会导致，索引无法充分使用，甚至索引失效！

> 原因：使用复合索引，需要遵循最佳左前缀法则，即如果索引了多列，要遵守最左前缀法则。指的是查询从索引的最左前列开始并且不跳过索引中的列。 
>
> 结论：过滤条件要使用索引必须按照索引建立时的顺序，依次满足，一旦跳过某个字段，索引后面的字段都无法被使用。

不要在索引列上做任何计算：不在索引列上做任何操作（计算、函数、(自动 or 手动)类型转换），会导致索引失效而转向全表扫描。

> 结论：等号左边无计算！
>
> 结论：等号右边无转换！ 

索引列上不能有范围查询 

> 建议：将可能做范围查询的字段的索引顺序放在最后
>
> 尽量使用覆盖索引：即查询列和索引列一直，不要写 select *!

使用不等于(!= 或者<>)的时候：mysql 在使用不等于(!= 或者<>)时，有时会无法使用索引会导致全表扫描。

> 字段的 is not null 和 is null：is not null 用不到索引，is null 可以用到索引

like 的前后模糊匹配：前缀不能出现模糊匹配！

> 减少使用 or：使用 union all 或者 union 来替代： 

## 口诀

全职匹配我最爱，最左前缀要遵守； 

带头大哥不能死，中间兄弟不能断； 

索引列上少计算，范围之后全失效； 

LIKE 百分写最右，覆盖索引不写*； 

不等空值还有 OR，索引影响要注意； 

VAR 引号不可丢，SQL 优化有诀窍。

# 关联查询优化

## left join

explain select * from class left join book on class.card=book.card

![image-20210906102030857](https://gitee.com/jijiejiojio/pic/raw/master/img/image-20210906102030857.png)

加了索引

create index idx_card on book(card)

![image-20210906102101045](https://gitee.com/jijiejiojio/pic/raw/master/img/image-20210906102101045.png)

结论：

在优化关联查询时，只有在被驱动表上建立索引才有效！ 

left join 时，左侧的为驱动表，右侧为被驱动表！ 

## inner join

结论：

inner join 时，mysql 会自己帮你把小结果集的表选为驱动表。 

# 慢日志查询

什么是慢日志查询：

> MySQL的慢查询日志是MySQL提供的一种日志记录，它用来记录在MySQL中响应时间超过阀值的语句，具体指运行时间超过long_query_time值的SQL，则会被记录到慢查询日志中。
>
> 具体指运行时间超过long_query_time值的SQL，则会被记录到慢查询日志中。long_query_time的默认值为10，意思是运行10秒以上的语句。
>
> 由他来查看哪些SQL超出了我们的最大忍耐时间值，比如一条sql执行超过5秒钟，我们就算慢SQL，希望能收集超过5秒的sql，结合之前explain进行全面分析。 



默认情况下，MySQL 数据库没有开启慢查询日志，需要我们手动来设置这个参数。 

当然，如果不是调优需要的话，一般不建议启动该参数，因为开启慢查询日志会或多或少带来一定的性能影响。慢查询日志支持将日志记录写入文件。