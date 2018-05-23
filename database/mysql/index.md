## MySQL 索引

- 如果索引包含多个列，那么列的顺序也十分重要 因为MySQL只能高效地使用索引的最左前缀列

## 索引的类型

在MySQL中，索引时在存储引擎层而不是在服务器层实现的，所以没有统一的索引标准

- B-Tree 索引

    - InnoDB使用的是B+Tree ，
    - MyISAM使用前缀压缩技术使得索引更小，但InnoDB 则按照原数据格式进行存储
    - B-Tree 通常意味着所有的值都是按顺序存储的，并且每一个叶子页到根的距离相同
    - B-Tree 索引能够加快访问数据的速度，因为存储引擎不再需要进行全表扫描来获取需要的数据，取而代之的是从索引的根节点开始进行搜索
    - B-Tree 对索引列是顺序组织存储的，索引很适合查找范围数据
    - B-Tree 索引适用于全键值、键值范围、键前缀查找，其中键前缀查找只适用于根据最左前缀的查找，索引只对如下类型的查询有效：
      
      - 全值匹配
      - 匹配最左前缀
      - 匹配列前缀
      - 匹配范围值
      - 精确匹配某一列并范围匹配另外一列
      - 只访问索引的查询
      
    - B-Tree 索引的限制
        
        - 如果不是按照索引的最左列开始查找，则无法使用索引。例如无法查找某个特定生日的人，也无法查找姓氏以某个字母结尾的人
        - 不能跳过索引中列
        - 如果查询中有某个列的范围查询，则其右边所有列都无法使用索引优化查找
        
- 哈希索引
  
  哈希索引 hash index 基于哈希表实现，只有精确匹配索引所有列的查询才有效。对于每一行数据，存储引擎都会对所有的索引列计算一个哈希码 hash code,哈希吗是一个较小的值，并且不同键值的行计算出来的哈希码也不一样。哈希索引将所有的哈希码存储在索引中，同时，在哈希表中保存指向每个数据行的指针。

- 空间数据索引 R-Tree

  和B-Tree索引不同，该索引无须前缀查询。空间索引会从所有维度来索引数据，查询时，可以有效地的使用任意维度来组合查询。mysql的GIS 支持并不完善。开源关系数据库中对GIS的解决方案做得比较好的是PostgreSQL的PostGIS.

- 全文索引

  - 全文索引查找的是文本中的关键字，而不是直接比较索引中的值。全文索引更类似于搜索引擎 做的事，而不是简单的where条件匹配
  - 在相同的列上同时创建全文索引和基于值的B-Tree索引不会有冲突，全文索引适用于match against 操作，而不是普通的where条件操作。

- 其他索引类别
  - TokuDB使用分形树索引 fractal tree index
  - 聚簇索引
  - 覆盖索引

## 索引的优点

 索引可以让服务器快速地定位到表的指定位置

 最常见的B-Tree索引，按照顺序存储数据，所以MySQL可以用来做order by 和 group by 操作。 因为数据是有序的，所以B-Tree 也就会将相关的列值都存储在一起。 最后， 因为索引中存储了实际的列值，所以某些索引就能够完成全部查询。据此特性，总结下来索引有如下三个有点：
  
  - 索引大大减少了服务器需要扫描的数据量
  - 索引可以帮助服务器避免排序和临时表
  - 索引可以将随机I/O 变为顺序I/O

 如果想深入理解索引这部分内容，强烈建议阅读由Tapio Lahdenmaki 和Mike Leach 编写的 Relational database index design and the Optimizers (wiley press)一书，该书详细介绍了如何计算索引的成本和作用，如何评估查询速度、如何分析索引维度的代价和其带来的好处

 ## 索引是最好的解决方案吗?

 索引并不总是 最好的工具。 总的来说，只有当索引帮助存储引擎快速查找到记录带来的好处大于其带来的额外工作时，索引才是有效的。

 - 对于 非常小的表，大部分情况下 简单的全表扫描更高效。
 - 对于中到大型的表，索引就非常有效
 - 对于特大型的表， 建立和使用索引的代价将随之增长。这种情况下，则需要一种技术可以直接区分出查询需要的一组数据，而不是一条记录一条记录地匹配。例如可以使用分区技术
 - 如果表的数量特别多，可以建立一个元数据信息表，用来查询需要用到的某些特性。例如，执行那些需要聚合多个应用分布在多个表的数据查询，则需要记录“那个用户的信息存储在那个表中”的元数据，这样在查询时，就可以直接忽略那些不包含指定用户信息的表
 - 对于TB级别的数据，定位单条记录的意义不大，所以经常会使用块级别的元数据技术来替代所以。






## 高性能的索引策略

 正确的创建和使用索引是实现性能查询的基础

 使用那个索引，以及如何评估选择不同索引的性能影响的技巧，需要持续不断的学习

### 1. 独立的列

查询中的列不是独立的，则mysql就不会使用索引

独立的列是指 索引列不能是表达式的一部分，也不能是函数的参数

下面的查询无法使用 actor_id的索引
~~~sql
SELECT actor_id FROM actor Where actor_id + 1 = 5;
~~~

下面是另一个常见的错误
~~~sql
select id from a where TO_DAYS(CURRENT_DATE) - TODAYS(date_col) <= 10;
~~~
### 2. 前缀索引和索引选择性

有时候需要索引很长的字符列，这会让索引变得大且慢。 通常可以索引开始的部分字符，这样可以大大节约索引空间，从而提高索引的效率。但这样也会降低索引的选择性。

>索引的选择性是指， 不重复的索引值(cardinality)和数据表的记录总数(#T)的比值，范围从 1/#T 到1 之间 。 索引的选择性越高则查询效率越高，因为选择性高的索引可以让MySQL在查找时过滤掉更多的行，唯一索引的选择性是1， 这是最好的索引选择性，性能也是最好的。

一般情况下，某个列前缀的选择性也是足够高的，足以满足查询性能。对于BLOB、text、或者很长的varchar类型的列，必须使用前缀索引，因为mysql不允许索引这些列的完整长度

诀窍在于要选择足够长的前缀以保证较高的选择性，同时又不能太长（以便节约空间）。前缀应该足够长，以使得前缀索引的选择性接近于索引整个列，换句话说，前缀的基数应该接近于完整列的基数

mysql 无法使用前缀索引做order by 和group by ,也无法使用前缀索引做覆盖扫描

### 3. 多列索引

- "把where条件里面的列都建上索引"，这个建议是非常错误的，这样一来最好的情况下也只能是 一星索引，
- 有时如果无法设计一个三星索引，那么不如忽略掉where 子句，集中精力优化索引列的顺序，或者创建一个全覆盖索引

在多个列上简历独立的单列索引，大部分情况下并不能提高MySQL的查询性能。

- 当出现服务器对多个索引做相交操作时（通常有多个AND 条件），通常意味着需要一个包含所有相关列的多列索引，而不是多个独立的单列索引
- 当服务器 需要对多个索引做联合操作时（通常有多个OR条件），通常需要耗费大量的cpu和内存资源在算法的缓存、排序和合并操作上。
- 更重要的是，优化器不会把这些计算到 查询成本 cost中，优化器只关心随机页面读取

### 4. 选择合适的索引列顺序


### 5. 聚簇索引

聚簇索引并不是一种单独的索引类型，而是一种数据存储的方式。当表有聚簇索引时，他的数据行实际上存放在索引的叶子页leaf page 中

术语聚簇 表示数据行和相邻的键值紧凑地存储在一起。 因为无法同时把数据行存放在两个不同的地方，所以一个表只能有一个聚簇索引

如果没有定义主键，InnoDB会选择一个唯一的非空索引代替。如果没有这样的索引，InnoDB会隐式定义一个主键来作为聚簇索引。

InnoDB只聚集在同一个页面中的记录。包含相邻键值的页面可能会相距甚远。
