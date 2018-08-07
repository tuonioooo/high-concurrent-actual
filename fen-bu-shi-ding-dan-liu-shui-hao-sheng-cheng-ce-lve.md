# 分布式订单流水号生成策略

* ## **基于数据库**

1\)UUID: UUID是通用唯一识别码（Universally Unique Identifier）的缩写，由时间戳、机器标识码、随机数组成。优点是简单粗暴（可使用JDK自带的UUID实现- java.util.UUID），缺点是无序，不适合建立索引。

2\)DB sequence:可以使用数据库提供的自增序列（如oracle和PostgreSQL的sequence，Redis的incr）作为订单号，缺点是位数不确定，有溢出风险，高并发情况下有性能瓶颈。

* ## **基于雪花算法**

Snowflake: 也叫雪花算法，是由Twitter开源的分布式id生成算法，生成id时的因子是：时间戳+递增序列+机器号+业务号，默认生成的id是18位的long型数字。我测了一下4核CPU每秒可生成100w+个id，可以说性能还是不错的。查看其源码会发现，该算法只有简单的逻辑判断与位运算，没有字符串拼接也没有时间格式化。这应该是该算法相较于普通算法性能较高的主要原因。下面截图是snowflake算法生成id的关键代码：

![](/assets/import-snowflake.png)

twitter-scala版本： https://github.com/twitter/snowflake

java版本：https://github.com/downgoon/snowflake

* ## 基于redis实现

https://github.com/tuonioooo/high-concurrent-cache  相关书籍讲解

https://github.com/tuonioooo/high-concurrent-cache/blob/master/redis/shi-zhan-chang-jing.md



