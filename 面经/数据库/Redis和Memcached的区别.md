# Redis 与 Memcached 共同点：

* 都是基于内存的数据库，一般都用来当做缓存使用。
* 都有过期策略。
* 两者的性能都非常高。

# Redis 与 Memcached 区别：

* Redis 支持的数据类型更丰富（String、Hash、List、Set、ZSet），而 Memcached 只支持最简单的 key-value 数据类型；
* Redis 支持数据的持久化，可以将内存中的数据保持在磁盘中，重启的时候可以再次加载进行使用，而 Memcached 没有持久化功能，数据全部存在内存之中，Memcached 重启或者挂掉后，数据就没了；
* Redis 原生支持集群模式，Memcached 没有原生的集群模式，需要依靠客户端来实现往集群中分片写入数据；
* Redis 支持发布订阅模型、Lua 脚本、事务等功能，而 Memcached 不支持；