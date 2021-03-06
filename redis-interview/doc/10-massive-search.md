# 如何在海量数据中查询一个值是否存在？

- [典型回答](#典型回答)
- [考点分析](#考点分析)
- [知识扩展](#知识扩展)
  - [1.开启布隆过滤器](#1开启布隆过滤器)
  - [2.布隆过滤器原理分析](#2布隆过滤器原理分析)
  - [3.在代码中实现布隆过滤器](#3在代码中实现布隆过滤器)
- [总结](#总结)

一般面试中考察的题目通常是由三类组成的：基础面试题、进阶面试题、开放性面试题。而本文的题目则属于一个开放性的面试题，但对于 Redis 这种以数据为核心的缓存中间件来说，实现在海量数据中查询一个值是否存在还是相对比较容易的。

因为是海量数据，所以就无法将每个键值都存起来，然后再从结果中检索数据了。比如数据库中的 `select count(1) from tablename where id='XXX'`，或者是使用 Redis 普通的查询方法 `get XXX` 等方式，只能依靠专门处理此问题的“特殊功能”和相关方法来实现数据的查找。

本文的面试题是：如何在海量数据中查询一个值是否存在？

### 典型回答

可以使用**布隆过滤器**统计一个值是否存在于海量数据中。布隆过滤器（Bloom Filter）是 1970 年由布隆提出的。它实际上是一个很长的二进制向量和一系列随机映射函数。**布隆过滤器可以用于检索一个元素是否在一个集合中**。优点：**计算和查询速度很快**，空间效率和查询时间都比一般的算法要好的多；缺点：**存在一定的误差**，有一定的误识别率和删除困难。

在 Redis 中布隆过滤器的用法如下：

1. bf.add 添加元素
2. bf.exists 判断某个元素是否存在
3. bf.madd 添加多个元素
4. bf.mexists 判断多个元素是否存在
5. bf.reserve **设置布隆过滤器的准确率**

使用示例如下：

```shell
> bf.add user xiaoming
(integer) 1
> bf.add user xiaohong
(integer) 1
> bf.add user laowang
(integer) 1
> bf.exists user laowang
(integer) 1
> bf.exists user lao
(integer) 0
> bf.madd user huahua feifei
1) (integer) 1
2) (integer) 1
> bf.mexists user feifei laomiao
1) (integer) 1
2) (integer) 0
```

可以看出以上结果没有任何误差。再来看一下准确率 `bf.reserve` 的使用：

```shell
> bf.reserve user 0.01 200
(error) ERR item exists # 已经存的 key 设置会报错
> bf.reserve userlist 0.9 10
OK
```

可以看出此命令必须在元素刚开始执行，否则会报错。它有三个参数：key、error_rate 和 initial_size。 其中：

- error_rate：允许布隆过滤器的错误率，这个值越低过滤器占用空间也就越大，因为此值决定了位数组的大小，位数组是用来存储结果的，它的空间占用的越大 (存储的信息越多)，错误率就越低，它的默认值是 0.01；
- initial_size：布隆过滤器存储的元素大小，实际存储的值大于此值，准确率就会降低，它的默认值是 100。

布隆过滤器常见使用场景有：

- 垃圾邮件过滤
- 爬虫里的 URL 去重
- 判断一个值在亿级数据中是否存在

布隆过滤器在数据库领域的使用也比较广泛。例如：HBase、Cassandra、LevelDB、RocksDB 内部都有使用布隆过滤器。

### 考点分析

这道题考察的是对布隆过滤器是否了解，如果没有相关知识筹备，通常的回答是先存储数据再进行检索，这显然就中了面试官的“圈套”，把一道“送分题”活活搞成了“送命题”。

和此知识点相关的面试题还有以下这些：

- 在 Redis 中可以直接使用布隆过滤器吗？
- 布隆过滤器的原理是什么？
- 在代码中如何实现布隆过滤器？

### 知识扩展

#### 1.开启布隆过滤器

在 Redis 中使用布隆过滤器需要先安装布隆过滤器的相关模块，在 Redis 4.0 版本之后就可以使用 modules (扩展模块) 的方式引入相应的功能，本文提供两种方式的开启方式：编译安装和 Docker 安装。

① 编译安装

```shell
> git clone https://github.com/RedisBloom/RedisBloom.git
> cd RedisBloom
> make # 编译 redisbloom
> cp redisbloom.so /path/to/modules/
# 在redis配置文件加载模块
loadmodule /path/to/modules/redisbloom.so
> redis-server # 启动
> redis-cli # 进入客户端命令行
```

② Docker 开启布隆过滤器

使用 Docker 的方式开启布隆过滤器比较简单，只需要拉取并启动相应的含有布隆过滤器的 Redis 镜像即可。实现命令如下：

```shell
> docker pull redislabs/rebloom # 拉取镜像
> docker run -p 6379:6379 --name redisbloom redislabs/rebloom # 运行容器
```

验证布隆过滤器是否正常开启：服务启动之后，只需使用 redis-cli 连接到服务端，输入 `bf.add` 看有没有命令提示，就可以判断是否正常启动了。如果有命令提示则表明 Redis 服务器已经开启了布隆过滤器。

```shell
bf.add key ...options...
```

#### 2.布隆过滤器原理分析

Redis 布隆过滤器的实现，依靠的是它数据结构中的位数组，每次存储键值的时候，不是直接把数据存储在数据结构中，因为这样太占空间了，它是利用几个不同的无偏哈希函数，把此元素的 hash 值均匀的存储在位数组中，也就是说，每次添加时会通过几个无偏哈希函数算出它的位置，把这些位置设置成 1 就完成了添加操作。

当进行元素判断时，查询此元素的几个哈希位置上的值是否为 1，如果全部为 1，则表示此值存在，如果有一个值为 0，则表示不存在。因为此位置是通过 hash 计算得来的，所以即使这个位置是 1，并不能确定是那个元素把它标识为 1 的，因此**布隆过滤器查询此值存在时，此值不一定存在，但查询此值不存在时，此值一定不存在**。

并且当位数组存储值比较稀疏的时候，查询的准确率越高，而当位数组存储的值越来越多时，误差也会增大。

位数组和 key 之间的关系。如下图所示： 

<img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1gi490igi48j30oi0b3jrm.jpg" alt="位数组和 key 的关系.jpg" width="550" />

#### 3.在代码中实现布隆过滤器

下面用 Java 语言来实现布隆过滤器，因为 Jedis 没有直接操作布隆过滤器的方法，所以使用 Jedis 操作 Lua 脚本的方式来实现布隆过滤器。代码如下：

```java
public class BloomExample {
  // 添加元素
  public static boolean bfAdd(Jedis jedis, String key, String value) {
    String luaStr = "return redis.call('bf.add', KEYS[1], KEYS[2])";
    Object result = jedis.eval(luaStr, Arrays.asList(key, value), Collections.emptyList());
    return result.equals(1L);
  }

  // 查询元素是否存在
  public static boolean bfExists(Jedis jedis, String key, String value) {
    String luaStr = "return redis.call('bf.exists', KEYS[1], KEYS[2])";
    Object result = jedis.eval(luaStr, Arrays.asList(key, value), Collections.emptyList());
    return result.equals(1L);
  }
}
```

```java
Jedis jedis = JedisUtil.getJedis();
for (int i = 1; i < 10_001; i++) {
  bfAdd(jedis, USER_MAP, "user_" + i);
  boolean exists = bfExists(jedis, USER_MAP, "user_" + i);
  if (!exists) {
    System.out.println("未找到数据 i=" + i);
    break;
  }
}
System.out.println("执行完成");
```

但发现执行了半天，执行的结果竟然是：

```
执行完成
```

没有任何误差，奇怪了，于是在循环次数后面又加了一个 0，执行了大半天之后，发现依旧是相同的结果，依旧没有任何误差。 这是因为**对于布隆过滤器来说，它说没有的值一定没有，它说有的值有可能没有**。 于是我们把程序改一下，重新换一个 key 值，把条件改为查询存在的数据。代码如下：

```java
Jedis jedis = JedisUtil.getJedis();
for (int i = 1; i < 10_001; i++) {
  bfAdd(jedis, USERLIST_MAP, "user_" + i);
  boolean exists = bfExists(jedis, USERLIST_MAP, "user_" + (i + 1));
  if (exists) {
    System.out.println("找到了" + i);
    break;
  }
}
System.out.println("执行完成");
```

这次发现执行不一会就出现了如下信息：

```
找到了334
执行完成
```

说明循环执行了一会之后就出现误差了，代码执行也符合布隆过滤器的预期。

### 总结

本文介绍了布隆过滤器的概念，以及布隆过滤器在 Redis 中的开启与使用方法，并讲了布隆过滤器的实现原理以及它的优缺点分析，布隆过滤器有两个重要的参数：error_rate 允许布隆过滤器的错误率、initial_size 布隆过滤器存储的元素大小。当布隆过滤器的位数组存储值比较稀疏的时候，查询的准确率较高，而当位数组存储的值越来越多时，误差也会增大。
