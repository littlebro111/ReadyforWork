> 建议浏览博客：[https://blog.csdn.net/qq_40991313/article/details/130355955](https://blog.csdn.net/qq_40991313/article/details/130355955)
# 1. 使用 `EXPLAIN` 分析执行计划

`EXPLAIN` 是分析SQL性能的核心工具，可展示数据库优化器如何执行查询，包括索引使用、表连接方式等关键信息。

## 关键字段解析 ：

- **`type`（访问类型）** ：  
    从优到劣依次为：`system` > `const`（结果只有一条的主键或唯一索引扫描） > `eq_ref`（唯一索引扫描） > `ref`（非唯一索引扫描） > `range`（索引范围扫描） > `index`（全索引扫描） > `ALL`（全表扫描）。
- **`key`（实际使用的索引）** ：  
    若为 `NULL`，表示未使用索引，需检查索引设计或查询条件。
- **`rows`（扫描行数）** ：  
    扫描行数越多，性能越差。优化目标是减少扫描行数。
- **`Extra`（附加信息）** ：
    - `Using where`：需通过过滤条件筛选数据。
    - `Using filesort`：当查询语句中包含groupby操作，而且无法利用索引l完成排序操作的时候，这时不得不选择相应的排序算法进行，甚至可能会通过文件排序，效率是很低的，所以要避免这种问题的出现。
    - `Using temporary`：使了用临时表保存中间结果，MySQL在对查询结果排序时使用临时表，常见于排序orderby和分组查询groupby。效率低，要避免这种问题的出现。
    - `Using index`：所需数据只需在索引即可全部获得，不须要再到表中取数据，也就是使用了覆盖索引，避免了回表操作，效率不错。

## 检查索引使用情况

#### (1) 是否命中索引？
- 通过执行计划的 `key` 字段确认索引是否被使用。
- 未命中索引的常见原因：
    - 条件列未建立索引。
    - 索引列被函数或表达式修改（如 `WHERE UPPER(name) = 'JOHN'`）。
    - 不符合最左前缀原则（复合索引）。 
#### (2) 索引选择性
- **选择性公式**：`选择性 = 不同值数量 / 总行数`。
- 高选择性列（如唯一键）更适合建立索引，低选择性列（如性别）索引效果差。

# 2. 使用 `PROFILE` 分析执行阶段耗时

`PROFILE` 可细化SQL执行的每个阶段（如解析、优化、执行）的耗时，定位具体瓶颈。

```mysql
SHOW PROFILES;          -- 查看历史查询耗时
SET profiling = 1;      -- 开启性能分析
SELECT * FROM table;
SHOW PROFILE FOR QUERY 1;
```

重点关注 `Sending data`、`Sorting result` 等阶段的时间占比。

详情可以查看：[MySQL 有效利用 profile 分析 SQL 语句的执行过程-腾讯云开发者社区-腾讯云](https://cloud.tencent.com/developer/article/1449108)

# 3. 检查慢查询日志（Slow Query Log）

慢查询日志记录执行时间超过阈值的SQL，帮助定位长期性能问题。

```mysql
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;  -- 设置阈值为1秒
```

使用 `mysqldumpslow` 或 `pt-query-digest` 分析日志，找出高频慢SQL。

详情可以查看：[https://blog.csdn.net/huangjhai/article/details/118714169](https://blog.csdn.net/huangjhai/article/details/118714169)

# 优化 SQL 语句结构

- 建表时，选取最适用的字段属性，尽可能减少定义字段宽度，尽量把字段设置NOTNULL；
- 在经常性的检索列上，建立必要索引，以加快搜索速率，避免全表扫描；
- 尽量使用覆盖索引，当select的数据列被所建索引覆盖时不需要回表，可以直接取得数据。
- 多次查询同样的数据，可以考虑缓存该组数据；
- SQL查询优化；
- 使用连接（JOIN）来代替子查询；
- 使用联合（UNION）来代替手动创建的临时表；