# Mysql中MyISAM和InnoDB两种存储引擎的区别

InnoDB是具有事务、回滚和崩溃修复能力的事务安全型引擎，它可以实现行级锁来保证高性能的大量数据中的并发操作；

MyISAM是具有默认支持全文索引、压缩功能及较高查询性能的非事务性引擎。

- InnoDB支持事务；MyISAM不支持。
- InnoDB支持行级锁、表锁；MyISAM只支持表锁。
- InnoDB支持MVCC；MyISAM不支持。
- InnoDB增删改性能更优；MyISAM查询性能更优。
- InnoDB不支持全文索引（已支持）；MyISAM支持。
- InnoDB支持外键；MyISAM不支持外键。
- InnoDB和MyISAM都支持B+树索引；InnoDB还支持自适应哈希索引。
- InnoDB有崩溃恢复机制；MyISAM没有。
- InnoDB索引和数据都存储在一个文件中（frm表结构文件、ibd索引和数据文件）；MyISAM的索引和数据存储是分开的（frm表结构文件、MYD数据文件、MYI索引文件）。
- MyISAM实现了前缀压缩技术，占用存储空间更小（但会影响查找）；InnoDB是原始数据存储，占用存储更大。