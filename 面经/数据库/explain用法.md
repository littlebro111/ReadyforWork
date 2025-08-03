Explain详解：在 select 语句之前增加 explain 关键字，执行后 MySQL 就会返回执行计划的信息，而不是执行 sql。但如果 from 中包含子查询，MySQL 仍会执行该子查询，并把子查询的结果放入临时表中。

## id

id 列的编号是 select 的序列号，有几个 select 就有几个 id，并且 id 是按照 select 出现的顺序增长的，id 列的值越大优先级越高，id 相同则是按照执行计划列从上往下执行，id 为空则是最后执行。

## select_type

表示对应行是简单查询还是复杂查询。

- simple：不包含子查询和 union 的简单查询
- primary：复杂查询中最外层的 select
- subquery：包含在 select 中的子查询（不在 from 的子句中）
- derived：包含在 from 子句中的子查询。mysql 会将查询结果放入一个临时表中，此临时表也叫衍生表。
- union：在 union 中的第二个和随后的 select，UNION RESULT 为合并的结果

## table

表示当前行访问的是哪张表。当 from 中有子查询时，table 列的格式为 `<derivedN>`，表示当前查询依赖 id=N 行的查询，所以先执行 id=N 行的查询。

## partitions

查询将匹配记录的分区。 对于非分区表，该值为 NULL。

## type

此列表示关联类型或访问类型。也就是 MySQL 决定如何查找表中的行。依次从最优到最差分别为：system > const > eq_ref > ref > range > index > all。一个好的SQL语句至少要达到range级别，杜绝出现all级别。

- NULL：MySQL 能在优化阶段分解查询语句，在执行阶段不用再去访问表或者索引。
- system、const：MySQL 对查询的某部分进行优化并把其转化成一个常量（可以通过 show warnings 命令查看结果）。system 是 const 的一个特例，表示表里只有一条元组匹配时为 system。
- eq_ref：主键或唯一键索引被连接使用，最多只会返回一条符合条件的记录。简单的 select 查询不会出现这种 type。
- ref：相比 eq_ref，不使用唯一索引，而是使用普通索引或者唯一索引的部分前缀，索引和某个值比较，会找到多个符合条件的行。
- range：通常出现在范围查询中，比如 in、between、大于、小于等。使用索引来检索给定范围的行。
-  index：扫描全索引拿到结果，一般是扫描某个二级索引，二级索引一般比较少，所以通常比 ALL 快一点。
- ALL：全表扫描，扫描聚簇索引的所有叶子节点。

## possible_keys

此列显示在查询中可能用到的索引。如果该列为 NULL，则表示没有相关索引，可以通过检查 where 子句看是否可以添加一个适当的索引来提高性能。

## key

此列显示 MySQL 在查询时实际用到的索引。在执行计划中可能出现 possible_keys 列有值，而 key 列为 null，这种情况可能是表中数据不多，MySQL 认为索引对当前查询帮助不大而选择了全表查询。如果想强制 MySQL 使用或忽视 possible_keys 列中的索引，在查询时可使用 force index、ignore index。

## key_len

此列显示 MySQL 在索引里使用的字节数，通过此列可以算出具体使用了索引中的那些列。索引最大长度为 768 字节，当长度过大时，MySQL 会做一个类似最左前缀处理，将前半部分字符提取出做索引。当字段可以为 null时，还需要 1 个字节去记录。

## ref

此列显示 key 列记录的索引中，表查找值时使用到的列或常量。常见的有 const、字段名。

## rows

此列是 MySQL 在查询中估计要读取的行数。注意这里不是结果集的行数。

## Extra列

详细说明。注意，常见的不太友好的值如下：Using filesort，Using temporary。

- Using index：使用覆盖索引（如果select后面查询的字段都可以从这个索引的树中获取，不需要通过辅助索引树找到主键，再通过主键去主键索引树里获取其它字段值，这种情况一般可以说是用到了覆盖索引）。
- Using where：使用 where 语句来处理结果，并且查询的列未被索引覆盖。
- Using index condition：查询的列不完全被索引覆盖，where条件中是一个查询的范围。
- Using temporary：MySQL需要创建一张临时表来处理查询。出现这种情况一般是要进行优化的。
- Using filesort：将使用外部排序而不是索引排序，数据较小时从内存排序，否则需要在磁盘完成排序。
- Select tables optimized away：使用某些聚合函数（比如 max、min）来访问存在索引的某个字段时。