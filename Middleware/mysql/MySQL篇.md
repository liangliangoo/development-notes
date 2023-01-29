# MySQL 专题篇

## 索引

## sql
### group by 发生了什么

MySQL group by 实现方式有三种：1、松散索引；2、紧凑索引；3、临时文件（文件排序）

#### 1、松散索引
 MySQL group by 语句在利用索引时，group by （多个字段的时候，必须满足最左前缀原则）可根据索引，即可对数据分组，此时完全不用去访问表的数据值（索引健对应的数据）

#### 2、紧凑索引

 当group by引用的字段无法构成所建索引的最左前缀索引时，也就是说group by不能利用索引时。如何where语句（如果有的话）弥补了这种差距，比如：
group by 引用的字段为（c2，c3），而索引为（c1，c2，c3）。此时如果where语句限定了c1=a（某一个值），那么此时mysql的执行过程为先根据where语句进行一次选择，

对选出来的结果集，可以利用索引。这种方式，从整体上来说，group by并没有利用索引，但是从过程来说，在选出的结果中利用了索引，这种方式就是紧凑索引。

#### 3、临时文件

满足group by 子句 最一般的方法就是扫描整张表 然后创建新的临时表，临时表中 每个组的所有行是连续的， 然后使用该临时表找到对应的组 应用 累计函数（如果有）


### order by 发生了什么

#### 1、order by 字段 不存在索引
全表扫描，如果所查询字段所占内存数 小于 max_length_for_sort_data 先讲加载到的数据放入sort_buffer当中，当内存不足会写入临时表中，如果大于max_length_for_sort_data 则会直接写入文件后排序，内存中的数据一般采用的是快排、文件中的数据会采用归并排序

#### 2、order by 字段是索引
##### 1、全字段排序
##### 2、rowId 排序


## MySQL中的日志

### redo log & undo log & binlog

## MySQL 2PC

## 慢SQL 记录
### or 导致的慢sql
为什么 or 会导致索引失效？
索引 + 全表扫描 + merge

1. case1
```sql
# idx_oaid_imei(oaid,imei) 为联合索引
select * from user_backdata where oaid = 'a' or imei = 'a';
# 线上数据量为 500W sql 耗时 5.0 s 左右
# 优化 sql
select * from user_backdata where oaid = 'a' union
select * from user_backdata where imei = 'b';
```


### in 导致慢sql
为什么 in 会导致 慢sql?
MySQL优化器决定使用某个索引执行查询的仅仅是因为：使用该索引时的成本足够低。也就是说即使我们有下边的语句：
```sql
SELECT * FROM t WHERE key1 IN ('b', 'c');
```
</br>
MySQL优化器需要去分析一下如果使用二级索引idx_key1执行查询的话，键值在['b', 'b']和['c', 'c']这两个范围区间的记录共有多少条，然后通过一定方式计算出成本，与全表扫描的成本相对比，选取成本更低的那种方式执行查询。</br></br>
MySQL优化器针对IN子句对应的范围区间的多少而指定了不同的策略：</br>

   1. 如果IN子句对应的范围区间比较少，那么将率先去访问一下存储引擎，看一下每个范围区间中的记录有多少条（如果范围区间的记录比较少，那么统计结果就是精确的，反之会采用一定的手段计算一个模糊的值，当然算法也比较麻烦），这种在查询真正执行前优化器就率先访问索引来计算需要扫描的索引记录数量的方式称之为index dive。</br>
   2. 如果IN子句对应的范围区间比较多，这样就不能采用index dive的方式去真正的访问二级索引idx_key1（因为那将耗费大量的时间），而是需要采用之前在背地里产生的一些统计数据去估算匹配的二级索引记录有多少条（很显然根据统计数据去估算记录条数比index dive的方式精确性差了很多）。
那什么时候采用index dive的统计方式，什么时候采用index statistic的统计方式呢？

这就取决于我们要说的系统变量eq_range_index_dive_limit的值了。
这个值在5.6版本默认是10，5.7版本默认是200
ep_range_index_dive_limit参数提供一个阈值，优化器在预估扫描行数时，会根据这个参数，来进行预估策略。通常优化器有两种预估策略：索引统计和索引下潜。
   1. 当低于eq_range_index_dive_limit参数阀值时，采用index dive方式预估影响行数，该方式优点是相对准确，但不适合对大量值进行快速预估。
   2. 当大于或等于eq_range_index_dive_limit参数阀值时，采用index statistics方式预估影响行数，该方式优点是计算预估值的方式简单，可以快速获得预估数据，但相对偏差较大。</br>
</br>

case1
```sql
# 索引
idx_activityCode_prizeKey(activity_code,prize_key);

# sql
select * from activity_draw_log where activity_code = 'lucky-charm' and prize_key in ('prize_key1','prize_key2','prize_key') order by id;

```