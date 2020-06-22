Hive中常见的高级查询有：group by、Order by、join、distribute by、sort by、cluster by、Union all。今天我们就来谈谈group by操作，group by操作表示按照某些字段的值进行分组，有相同的值放到一起，语法样例如下：

```java
select col1,col2,count(1),sel_expr(聚合操作)
from tableName
where condition
group by col1,col2
having...
```
注意：
(1)：select后面的非聚合列必须出现在group by中(如上面的col1和col2)。

(2)：除了普通列就是一些聚合操作。



group的特性：

(1)：使用了reduce操作，受限于reduce数量，通过参数mapred.reduce.tasks设置reduce个数。

(2)：输出文件个数与reduce数量相同，文件大小与reduce处理的数量有关。

问题：

(1)：网络负载过重。

(2)：出现数据倾斜(我们可以通过hive.groupby.skewindata参数来优化数据倾斜的问题)。

下面我们通过一些语句来深入体会group by相关的操作：

```java
insert overwrite table pv_gender_sum
select pv_users.gender count(distinct pv_users.userid)
from pv_users
group by pv_users.gender;
```
附：上述语句是从pv_users表中查询出性别和去重后的总人数，并且根据性别分组，然后将数据覆盖插入到pv_gender_sum中。


在select语句中可以有多个聚合操作，但是如果多个聚合操作中同时使用了distinct去重，那么distinct去重的列必须相同，如下语句不合法：
```java
insert overwrite table pv_gender_agg
select pv_users.gender,count(distinct pv_users.userid),count(distinct pv_users.ip)
from pv_users
group by pv_users.gender;
```
注：上述语句之所以不合法，是因为distinct关键字去重的列不一样。一个是对userid去重，一个是对ip去重！
对上述非法语句做如下修改及将distinct的类改为一致就正确：

```java
insert overwrite table pv_gender_agg
select pv_users.gender,count(distinct pv_users.userid),count(distinct pv_users.userid)
from pv_users
group by pv_users.gender;
```
注：上述语句正确无误。


还有一个要注意的就是文章开头所说的知识点即select后面的非聚合列必须出现在group by中，否则非法，如下：
```java
select uid,name,count(sal)
from users
group by uid;
```
注：上述语句是非法的因为select中出现了两个两个非聚合列即uid和name，但是group by中只有uid,所以非法。


修改上述语句即将name也加到group by后面。

```java
select uid,name,count(sal)
from users
group by uid,name;
```
注：上述语句就正确。


下面我们来看看一些优化的属性：
(1)：Reduce的个数设置

设置reduce的数量：mapred.reduce.tasks，默认为1个，如下图：



当将reduce的个数设置为3个的时候，如下：





(2)：group by的Map端聚合

hive.map.aggr控制如何聚合，我使用的版本是0.90，默认是开启的即为true，这个时候Hive会在Map端做第一级的聚合。这通常提供更好的效果，但是要求更多的内存才能运行效果。
```java
hive> 
hive> set hive.map.aggr=true;
hive> select count(1) from employees;
```
(3)：数据倾斜
hive.groupby.skewdata属性设定是否在数据分布不均衡，即发生倾斜时进行负载均衡，当选项hive.groupby.skewdata=true时，生成的查询计划会有两个MapReduce即产生两个Job，在第一个MapReduce中，Map的输出结果会随机的分布到不同的Reduce中，对Reduce做部分聚合操作并输出结果，此时相同的group by key有可能分发到不同的reduce中，从而达到负载均衡的目的，第二个MapReduce任务根据预处理的数据按照group by key分布到Reduce中(此时Key相同就分布到同一个Reduce中)，最后完成聚合操作。
