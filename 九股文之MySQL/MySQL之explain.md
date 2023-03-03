# Explain

## 简介

MySQL 提供了一个 **EXPALIN** 命令，可以用于对 **SELECT 语句** 的执行计划进行分析，并详细的输出分析结果，供开发人员进行针对性的优化。

## 字段介绍

1. <u>id</u>: SELECT查询序列号，即为sql语句执行的顺序
2. <u>select_type</u>:  SELECT的类型，他有以下几种类型：
   - *simple*: 最简单的select，表示没有union与子查询
   - *primary*: 在有子查询的语句中，外面的那一层select即为primary
   - *union*: union查询语句的第二个语句，即为union
   - *union result*: union语句的结果
3. <u>table</u>: 所用到的表
4. <u>type</u>: 连接类型，后面再详细介绍
5. <u>possible_keys</u>: 可能使用到的索引
6. <u>key</u>: 实际选择的索引
7. <u>key_len</u>: 使用的索引长度
8. <u>ref</u>: 索引的哪一列被使用了
9. <u>rows</u>: 估计要扫描的行，数字越大select的效率越低
10. <u>extra</u>: 额外信息，后面再详细介绍

## type详解

type表示SQL语句对表的访问方式（MySQL在表中找到所需行的方式），又称访问类型，通常用type的值来判断SQL语句是否使用到索引。type的值有以下几种，由好到差依次是：

1. <u>system</u>: 该表只有一行（相当于系统表），system是const类型的特例

2. <u>const</u>: 针对主键或唯一索引的等值查询扫描, 最多只返回一行数据. const 查询速度非常快, 因为它仅仅读取一次即可

3. <u>eq_ref</u>: 通常出现在连表join查询中，连接的字段为主键或者唯一索引，表示对前表的每一个结果，都对应后表的唯一一条记录。并且查询的比较是=操作，查询效率比较高，如：SELECT * FROM ref_table,other_table WHERE  ref_table.key_column=other_table.column

4. <u>ref</u>: ref有三种情况：
   - 单表根据索引（非主键、非唯一索引），匹配到多行，如：`SELECT * FROM ref_table WHERE key_column=expr;`
   - 多表关联查询，单个索引，匹配多行，即前表的每一条记录对应后表的多条记录（关联到后表非唯一索引、非主键索引）,如：`SELECT * FROM ref_table,other_table   WHERE ref_table.key_column=other_table.column;`
   - SQL语句匹配最左索引原则

5. <u>full_text</u>: 匹配全文检索索引

6. <u>ref_or_null</u>: 该类型类似于ref，但是MySQL会额外搜索哪些行包含了NULL。这种类型常见于解析子查询，如：`SELECT * FROM ref_table WHERE key_column=expr OR key_column IS NULL;`

7. <u>index_merge</u>: 此类型表示使用了索引合并优化，表示一个查询里面用到了多个索引

8. <u>unique_subquert</u>: 该类型与eq_ref类似，但是使用了IN查询，且子查询是主键或者唯一索引。如：`value IN (SELECT primary_key FROM single_table WHERE some_expr)`

9. <u>index_subquery</u>: 该类型unique_subquery类似，只是子查询使用的是非唯一索引，如：`value IN (SELECT key_column FROM single_table WHERE some_expr)`

10. <u>range</u>: 范围扫描，表示检索了指定范围的行，主要用于限制索引的扫描。比较常见的范围扫描是带有BETWEEN子句或WHERE子句里有>、>=、<、<=、IS NULL、<=>、BETWEEN、LIKE、IN()等操作符。如：`SELECT * FROM tbl_name   WHERE key_column BETWEEN 10 and 20;`

11. <u>index</u>: 全索引扫描，和all类似，只不过index是全盘扫描了索引的数据。index通常比all快，因为索引数据通常小于表的数据。如果所有数据都在二级索引中拿到，不需要回表，那么extra的结果是using index。

12. <u>all</u>: 全表扫描，性能最差。

## extra详解

extra表示此次查询的附加信息，下面列举几个比较常见的值：

1. <u>Distinct</u>: 查找distinct值，当找到第一个匹配的行后，将停止为当前行组合搜索更多行
2. <u>Using Filesort</u>: 当查询中包含 ORDER BY 操作 ，而且**无法利用索引完成排序操作**的时候，MySQL Query Optimizer 不得不选择相应的排序算法来实现。数据较少时从内存排序，否则从磁盘排序。
3. <u>Using Index</u>: 仅使用索引树中的信息从表中检索列信息，而不必进行其他查找以读取实际行，即**索引覆盖**。
4. <u>Using Index Condition</u>: 表示先按条件过滤索引，过滤完索引后找到所有符合索引条件的数据行，随后用 WHERE 子句中的其他条件去过滤这些数据行，即**索引下推**。
5. <u>Using MRR</u>: 使用了Multi-Range Read优化策略，将**随机读改为顺序读**。
6. <u>Using Temporary</u>: MySQL需要创建一个临时表来保存结果,如果查询包含不同列的GROUP BY和 ORDER BY子句，通常会发生这种情况。
7. <u>Using Where</u>: 如果我们不是读取表的所有数据，或者不是仅仅通过索引就可以获取所有需要的数据（意味着要**回表**）则会出现Using Where信息。

## 索引建立原则

1. 对查询频次较高, 且数据量比较大的表, 建立索引。
2. 索引字段的选择, 最佳候选列应当从where子句的条件中提取, 如果where子句中的组合比较多, 那么应当挑选最常用, 过滤效果最好的列的组合。
3. 如果where后有多个条件经常被用到, 建议建立复合索引, 复合索引需要遵循最左前缀法则, N个列组合而成的复合索引, 相当于创建了N个索引。
4. 使用唯一索引, 区分度越高, 使用索引的效率越高。
5. 索引并非越多越好, 如果该表增、删、改操作较多, 慎重选择建立索引, 过多索引会降低表维护效率。
6. 使用短索引（索引列的值要尽量短）, 提高索引访问时的I/O效率, 因此也相应提升了查询效率。
7. 多表连接的字段上需要建立索引，这样可以极大提高表连接的效率。
8. ORDER BY、GROUP BY、DISTINCT 和 UNION 等操作的字段，排序操作非常消耗时间。添加索引能减少甚至避免排序，提高查询效率。

