### MySQL EXPLAIN 整理

​	MySQL的EXPLAIN命令用于SQL语句的查询执行计划（QEP）。这条命令的输出结果能够让我们了解MySQL优化器是如何执行SQL语句的。

​	这条命令并没有提供任何调整建议，但它能够提供重要的信息帮助你做出调优决策。

[TOC]

#### 1. 语法

```sql
EXPLAIN <select statement>; 
```

MySQL的EXPLAIN语法可以运行在SELECT语句或者特定的表，如果作用在表上，那么此命令等同于DESC表命令。

**对表执行**（部分结果）

```sql
mysql> explain mysql.user \G
*************************** 1. row ***************************
  Field: Host
   Type: char(60)
   Null: NO
    Key: PRI
Default: 
  Extra: 
*************************** 2. row ***************************
  Field: User
   Type: char(32)
   Null: NO
    Key: PRI
Default: 
  Extra: 
*************************** 3. row ***************************
  Field: Select_priv
   Type: enum('N','Y')
   Null: NO
    Key: 
Default: N
  Extra: 
```

**对SQL执行**

```sql
mysql> explain select * from mysql.user where user='root'\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: user
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 3
     filtered: 33.33
        Extra: Using where
1 row in set, 1 warning (0.00 sec)

```

#### 2. explain列详解

| Column          | JSONName        | Meaning                   |
| --------------- | --------------- | ------------------------- |
| `id`            | `select_id`     | 查询标识。id越大优先执行；id相同自上而下执行； |
| `select_type`   | None            | 查询的类型                     |
| `table`         | `table_name`    | 查询的表                      |
| `partitions`    | `partitions`    | Thematching partitions    |
| `type`          | `access_type`   | 连接类型                      |
| `possible_keys` | `possible_keys` | 可能选择的索引                   |
| `key`           | `key`           | 实际使用的索引                   |
| `key_len`       | `key_length`    | 使用的索引长度                   |
| `ref`           | `ref`           | 哪一列或常数在查询中与索引键列一起使用       |
| `rows`          | `rows`          | 估计查询的行数                   |
| `filtered`      | `filtered`      | 被条件过滤掉的行数百分比              |
| `Extra`         | None            | 解决查询的一些额外信息               |

##### 字段 select_type

| alue                  | JSONName                     | Meaning                     |
| --------------------- | ---------------------------- | --------------------------- |
| `SIMPLE`              | None                         | 简单查询 (不使用UNION或子查询)         |
| `PRIMARY`             | None                         | 外层查询，主查询                    |
| `UNION`               | None                         | UNION中第二个语句或后面的语句           |
| `DEPENDENTUNION`      | `dependent` (true)           | UNION中第二个语句或后面的语句，独立于外部查询   |
| `UNIONRESULT`         | union_result                 | UNION的结果                    |
| `SUBQUERY`            | None                         | 子查询中第一个SELECT               |
| `DEPENDENTSUBQUERY`   | `dependent` (true)           | 子查询中第一个SELECT，独立于外部查询       |
| `DERIVED`             | None                         | 子查询在 FROM子句中                |
| `MATERIALIZED`        | `materialized_from_subquery` | 物化子查询（不清楚是什么样的查询语句？）        |
| `UNCACHEABLESUBQUERY` | `cacheable` (false)          | 结果集不能被缓存的子查询，必须重新评估外层查询的每一行 |
| `UNCACHEABLEUNION`    | `cacheable` (false)          | UNION中第二个语句或后面的语句属于不可缓存的子查询 |

##### 字段 type

| type                    | Meaning                                  |
| ----------------------- | ---------------------------------------- |
| `system`                | 表仅一行数据 (=system table).这是const连接类型的特例。   |
| `const`                 | 表最多只有一个匹配行，在查询开始时被读取。因为只有一个值，优化器将该列值视为常量。当在primarykey或者unique索引作为常量比较时被使用。 |
| `eq_ref(engine=myisam)` | 来自前面表的结果集中读取一行，这是除system和const外最好的连接类型。当在使用PRIMARYKEY或者UNIQUE NOT NULL的索引时会被使用。 |
| `ref`                   | 对于前面表的结果集匹配查询的所有行，当连接使用索引key时，或者索引不是PRIMARYKEY和UNIQUE，则使用该类型。如果使用索引匹配少量行时，是不错的连接类型。 |
| `ref_or_null`           | 连接类型类似ref，只是搜索的行中包含NULL值MySQL做了额外的查找。    |
| `fulltext`              | 使用全文索引时出现。                               |
| `index_merge`           | 使用了索引合并优化。(未成功)                          |
| `unique_subquery`       | 该类型将ref替换成以下子查询的格式：value IN (SELECT **primary_key**  FROM single_table WHERE some_expr) |
| `index_subquery`        | 与 unique_subquery类似，但是将主键改为非唯一索引：value IN  (SELECT **key_column** FROM single_table WHERE some_expr) |
| `range`                 | 使用索引检索给定范围内的行。                           |
| `index`                 | 该连接类型与ALL相同，除了扫描索引树。如果查询的字段都在索引列中，则使用index类型，否则为ALL类型。 |
| `ALL`                   | 对于前面表的结果集中，进行了全表扫描。最差的一种类型，应考虑查询优化了！     |

**查询类型性能由优到差：**

> system > const > eq_ref > ref > fulltext > ref_or_null > index_merge > unique_subquery > index_subquery > range > index > ALL

##### 字段 Extra

该列输出关MySQL如何解决查询的额外信息。（下面列出部分常见的）

| **Extra**                     | Meaning                                  |
| ----------------------------- | ---------------------------------------- |
| `usingwhere`                  | 使用过滤条件                                   |
| `usingindex`                  | 从索引树中查找所有列                               |
| `usingtemporary`              | 使用临时表存储结果集，在使用``groupby``和``orderby``发生  |
| `selecttables optimized away` | 没有groupby情况下使用min(),max(),或者count(*)     |
| `usingfilesort`               | 有排序                                      |
| `notexists`                   | 在leftjoin中匹配一行之后将不再继续查询查询                |
| `distinct`                    | 查找到第一个匹配的行之后，MySQL则会停止对当前行的搜索            |
| `impossiblewhere`             | where子句总数失败的查询                           |
| `impossiblehaving`            | having子句总数失败的查询                          |
| `usingjoin buffer`            | 使用连接缓存                                   |
| `Usingindex for group-by`     | 与``Usingindex``类似，在使用``group-by``时可从索引中找到字段 |



#### 3. EXTENDED 

```sql
EXPLAIN EXTENDED <select statement>;  
```



### 参考文档

[MySQL EXPLAIN官方文档](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html)

[MySQL EXPLAIN EXTENDED官方文档](https://dev.mysql.com/doc/refman/5.7/en/explain-extended.html)



