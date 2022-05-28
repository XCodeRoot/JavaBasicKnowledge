# Redis 基础和应用
## 基础数据结构
### Redis 安装
```
docker pull redis
docker run --name myredis -d -p 6379:6379 redis
docker exec -it myredis redis-cli
```

### 五种基础数据结构
#### string

Redis 所有的数据结构都以唯一的 key 字符串作为名称，然后通过这个唯一 key 值来获取相应的 value 数据。

字符串结构使用非常广泛，一个常见的用途就是缓存用户信息。

Redis 的字符串是动态字符串，是可以修改的字符串，内部结构的实现类似于 Java 的 ArrayList，采用预分配冗余空间的方式来减少内存的频繁分配。

当字符串长度小于 1MB 时，扩容都是加倍现有的空间。如果字符串长度超过 1MB，扩容时一次只会多扩 1MB 的空间，需要注意的是字符串最大长度为 512MB。

- 键值对
```shell
127.0.0.1:6379> set name codehole 
OK
127.0.0.1:6379> get name
"codehole"
127.0.0.1:6379> exists name     //判断name是否存在 存在为1 不存在为0
(integer) 1
127.0.0.1:6379> del name
(integer) 1
127.0.0.1:6379> exists name
(integer) 0
```

- 批量键值对
```shell
127.0.0.1:6379> set name1 codehole
OK
127.0.0.1:6379> set name2 holycoder
OK
127.0.0.1:6379> mget name1 name2 name3
1) "codehole"
2) "holycoder"
3) (nil)
127.0.0.1:6379> mset name1 boy name2 girl name3 unknown
OK
127.0.0.1:6379> mget name1 name2 name3
1) "boy"
2) "girl"
3) "unknown"
```

- 过期和 set 命令扩展
```shell
127.0.0.1:6379> set name codehole 
OK
127.0.0.1:6379> get name
"codehole"
127.0.0.1:6379> expire name 5   //设置过期时间
(integer) 1
127.0.0.1:6379> get name
"codehole"
127.0.0.1:6379> get name
(nil)
127.0.0.1:6379> 
127.0.0.1:6379> 
127.0.0.1:6379> setex name 5 codehole   //setex 等价于 set + expire
OK
127.0.0.1:6379> get name
"codehole"
127.0.0.1:6379> setnx name codehole     //如果 name 不存在则执行 set 命令
(integer) 1
```

- 计数
```shell
127.0.0.1:6379> set age 30
OK
127.0.0.1:6379> incr age
(integer) 31
127.0.0.1:6379> incrby age 5
(integer) 36
127.0.0.1:6379> incrby age -5
(integer) 31
127.0.0.1:6379> set codehole 9223372036854775807
OK
127.0.0.1:6379> incr codehole
(error) ERR increment or decrement would overflow
```
> 字符串由多个字节组成，每个字节又由 8 个 bit 组成，如此便可以将一个字符串看成很多 bit 的组合，这便是 bitmap（位图）数据结构。

#### list

Redis 的列表相当于 Java 语言里面的 LinkedList，注意它是链表而不是数组。列表中的每个元素都使用双向指针顺序，串起来可以同时支持前向后向遍历。

Reids 的列表结构常用来做异步队列使用。

- 队列
```shell
127.0.0.1:6379> rpush books python java golang
(integer) 3
127.0.0.1:6379> llen books
(integer) 3
127.0.0.1:6379> lpop books
"python"
127.0.0.1:6379> lpop books
"java"
127.0.0.1:6379> lpop books
"golang"
127.0.0.1:6379> lpop books
(nil)
```

- 栈
```shell
127.0.0.1:6379> rpush books python java golang
(integer) 3
127.0.0.1:6379> rpop books
"golang"
127.0.0.1:6379> rpop books
"java"
127.0.0.1:6379> rpop books
"python"
127.0.0.1:6379> rpop books
(nil)
```

- 慢操作

lindex 相当于 Java 链表的 get 方法，它需要对链表进行遍历，性能随着参数 index 增大而变差。

ltrim 的两个参数 start_index 和 end_index 定义了一个区间，保留区间内的值。

index 可以为负数，index=-1 表示倒数第一个元素。

```shell
127.0.0.1:6379> lindex books 1
"java"
127.0.0.1:6379> lrange books 0 -1
1) "python"
2) "java"
3) "golang"
127.0.0.1:6379> ltrim books 1 -1
OK
127.0.0.1:6379> lrange books 0 -1
1) "java"
2) "golang"
```

- 快速链表

Redis 底层存储的不是一个简单的 LinkedList，而是称之为"快速链表"（quicklist）的数据结构。

在列表元素较少的情况下，会使用一块连续的内存存储，这个结构是 ziplist，即压缩列表。它将所有的元素彼此紧挨着一起存储，分配的是一块连续的内存。当数据量比较多的时候才会改成 quicklist。

所以 Redis 将链表和 ziplist 结合起来组成了 quicklist，也就是将多个 ziplist 使用双向指针串起来使用。

#### hash

Redis 的字典相当于 Java 语言里面的 HashMap，实现结构上与 Java 的 HashMap 也是一样的，都是"数组+链表"二维结构。不同的是，Redis 的字典的值只能是字符串，另外 rehash 的方式不一样，Redis 为了追求高性能，不能堵塞服务，所以采用了渐进式 rehash 策略。

渐进式 rehash 会在 rehash 同时，保留新旧两个 hash 结构，查询时会同时查询两个 hash 结构，然后在后续的定时任务以及 hash 操作指令中，循序渐进地将旧 hash 的内容迁移到新的 hash 结构中。
```shell
127.0.0.1:6379> hset books java "think in java"
(integer) 1
127.0.0.1:6379> hset books python "python cookbook"
(integer) 1
127.0.0.1:6379> hset books golang "concurrency in go"
(integer) 1
127.0.0.1:6379> hgetall books
1) "java"
2) "think in java"
3) "python"
4) "python cookbook"
5) "golang"
6) "concurrency in go"
127.0.0.1:6379> hlen books
(integer) 3
127.0.0.1:6379> hget books java
"think in java"
127.0.0.1:6379> hmset books java "effective java" python "learning python" golang "modern golang programming"
OK
127.0.0.1:6379> hset user age 29
(integer) 1
127.0.0.1:6379> hincrby user age 5
(integer) 34
```

#### set

Redis 的集合相当于 Java 的 HashSet，它内部的键值对是无序的、唯一的。
```shell
127.0.0.1:6379> sadd books python
(integer) 1
127.0.0.1:6379> sadd books python
(integer) 0
127.0.0.1:6379> sadd books java golang 
(integer) 2
127.0.0.1:6379> smembers books  //查询所有元素
1) "java"
2) "golang"
3) "python"
127.0.0.1:6379> sismember books java    //查询某个元素是否存在 存在为1 不存在为0
(integer) 1
127.0.0.1:6379> sismember books rush
(integer) 0
127.0.0.1:6379> scard books //获取长度
(integer) 3
127.0.0.1:6379> spop books
"java"
```

#### zset
类似于 Java 的 SortedSet 和 HashMap 的结合体，一方面它是一个 set，保证了内部的 value 的唯一性，另一方面它可以给每个 value 赋予一个 score，代表这个 value 的排序权重。它的内部实现用的是一种叫做"跳跃列表"的数据结构。

zset 可以用来存储粉丝列表，value 值是粉丝的用户 ID，score 是关注时间。可以对粉丝列表按关注时间进行排序。

zset 可以用来存储学生的成绩，value 值是学生的 ID，score 是学生的考试成绩。对成绩按分数进行排序就可以得到他的名次。
```shell
127.0.0.1:6379> zadd books 9.0 "think in java"
(integer) 1
127.0.0.1:6379> zadd books 8.9 "java concurrency"
(integer) 1
127.0.0.1:6379> zadd books 8.6 "java cookbook"
(integer) 1
127.0.0.1:6379> zrange books 0 -1
1) "java cookbook"
2) "java concurrency"
3) "think in java"
127.0.0.1:6379> zrevrange books 0 -1
1) "think in java"
2) "java concurrency"
3) "java cookbook"
127.0.0.1:6379> zcard books
(integer) 3
127.0.0.1:6379> zscore books "java concurrency"
"8.9000000000000004"
127.0.0.1:6379> zrank books "java concurrency"
(integer) 1
127.0.0.1:6379> zrangebyscore books 0 8.91
1) "java cookbook"
2) "java concurrency"
127.0.0.1:6379> zrangebyscore books -inf 8.91 withscores
1) "java cookbook"
2) "8.5999999999999996"
3) "java concurrency"
4) "8.9000000000000004"
127.0.0.1:6379> zrem books "java concurrency"
(integer) 1
127.0.0.1:6379> zrange books 0 -1
1) "java cookbook"
2) "think in java"
```

### 容器型数据结构的通用规则

list、set、hash、zset 这四种共享下面两条通用规则：
1. create if not exists：如果容器不存在，那就创建一个再进行操作；
2. drop if no elements：如果容器里的元素没有了，那么立即删除容器，释放内存。


### 过期时间

过期是以对象为单位的，比如一个 hash 结构的过期是整个 hash 对象，而不是其中的某个子 key 的过期，还有一个需要特别注意的地方，如果一个字符串已经设置了过期时间，然后调用 set 方法修改了它，它的过期时间会消失。


## 分布式锁
### 分布式锁的奥义

分布式锁的本质就是要在 Redis 中占一个"坑"，占坑一般使用 set（set if not exists）命令，只允许一个客户端占坑。

但是有个问题，如果逻辑执行到中间出现了异常，可能会导致 del 指令没有被调用，这样就会陷入死锁，锁永远得不到释放。于是需要再加一个过期时间。

在 Redis 2.8 版本中，加入了 set 指令的扩展参数，使得 setnx 和 expire 指令可以在一起执行，切底解决了分布式锁的乱象。
```shell
127.0.0.1:6379> set lock:codehole true ex 5 nx
OK
127.0.0.1:6379> del lock:codehole
(integer) 0
```

### 超时问题
Redis 的分布式锁不能解决超时问题，如果在加锁和释放锁之间的逻辑执行的太长，以至于超出了锁的超时限制，就会出现问题。

有一个稍微安全一点的方案是将 set 指令的 value 参数设置为一个随机数，释放锁时先匹配随机数值是否一致，然后再删除 key，这是为了确保当前线程占有的锁不会被其他线程释放，除非这个锁是因为过期了而被服务器自动释放的。

但是匹配 value 和删除 key 不是一个原子操作，只能通过 Lua 脚本来处理。

### 可重入性

可重入性是指线程在持有锁的情况下再次请求加锁，如果一个锁支持同一个线程的多次加锁，那么这个锁就是可重入的。

## 延时队列

Redis 的 list 数据结构常用来作为异步消息队列使用，用 rpush 和 lpush 操作入队列，用 lpop 和 rpop 操作出队列。它可以支持多个生产者和多个消费者并发进出消息，每个消费者拿到的消息都是不同的列表元素。

### 阻塞读

客户端通过队列的 pop 操作获取消息，处理完了再接着获取消息再处理，但是如果队列空了，pop 就会陷入死循环，通常可以通过 sleep 解决问题，让线程睡一会儿。

但是睡眠会导致消息的延迟增大，使用 blpop 和 brpop 可以解决这种问题，b 代表 blocking。阻塞读在队列没有数据的时候，会立即进入休眠状态，一旦数据到来，则立刻醒过来，消息的延迟几乎为零。

### 锁冲突处理

1. 直接抛出特定类型的异常
2. sleep
3. 延时队列

## 位图                        

位图的最小单位是比特（bit），每个 bit 的取值只能是 0 或 1。

位图不是特殊的数据结构，它的内容其实就是普通的字符串，也就是 byte 数组。我们可以使用普通的 get/set 直接获取和设置整个位图的内容，也可以使用位图操作 getbit/setbit 等将 byte 数组看成 "位数组"来处理。

### 基本用法

Redis 的位数组是自动扩展的，如果设置了某个偏移位置超出了现有的内容范围，就会自动将位数组进行零扩充。

- 零存整取
```shell
127.0.0.1:6379> setbit s 1 1
(integer) 0
127.0.0.1:6379> setbit s 2 1
(integer) 0
127.0.0.1:6379> setbit s 4 1
(integer) 0
127.0.0.1:6379> setbit s 9 1
(integer) 0
127.0.0.1:6379> setbit s 10 1
(integer) 0
127.0.0.1:6379> setbit s 13 1
(integer) 0
127.0.0.1:6379> setbit s 15 1
(integer) 0
127.0.0.1:6379> get s
"he"
```

- 零存零取
```shell
127.0.0.1:6379> setbit w 1 1
(integer) 0
127.0.0.1:6379> setbit w 2 1
(integer) 0
127.0.0.1:6379> setbit w 4 1
(integer) 0
127.0.0.1:6379> getbit w 1
(integer) 1
127.0.0.1:6379> getbit w 2
(integer) 1
127.0.0.1:6379> getbit w 4
(integer) 1
127.0.0.1:6379> getbit w 5
(integer) 0
```

- 整存零取
```shell
127.0.0.1:6379> set w h
OK
127.0.0.1:6379> getbit w 1
(integer) 1
127.0.0.1:6379> getbit w 2
(integer) 1
127.0.0.1:6379> getbit w 4
(integer) 1
127.0.0.1:6379> getbit w 5
(integer) 0
```

### 统计和查找

bitcount 用来统计指定位置范围内 1 的个数，bitpos 用来查找指定范围内出现的第一个 0 或 1

```shell
127.0.0.1:6379> set w hello
OK
127.0.0.1:6379> bitcount w
(integer) 21
127.0.0.1:6379> bitcount w 0 0  //第一个字符中 1 的位数
(integer) 3
127.0.0.1:6379> bitcount w 0 1
(integer) 7
127.0.0.1:6379> bitpos w 0
(integer) 0
127.0.0.1:6379> bitpos w 1
(integer) 1
127.0.0.1:6379> bitpos w 1 1 1
(integer) 9
127.0.0.1:6379> bitpos w 1 2 2
(integer) 17
```

### 魔术指令 bitfield

bitfield 有三个子指令，分别是 get、set、incrby，它们都可以对指定位片段进行读写，但是最多只能处理 64 个连续的位，就得使用多个子指令，bitfield 可以一次执行多个子指令。
```shell
127.0.0.1:6379> set w hello
OK
127.0.0.1:6379> bitfield w get u4 0 //从第一位开始取 4 个位，结果是无符号数(u)
1) (integer) 6
127.0.0.1:6379> bitfield w get u3 2 //从第三位开始取 3 个位，结果是无符号数(u)
1) (integer) 5
127.0.0.1:6379> bitfield w get i4 0 //从第一位开始取 4 个位，结果是有符号数(i)
1) (integer) 6
127.0.0.1:6379> bitfield w get i3 2
1) (integer) -3
127.0.0.1:6379> bitfield w get u4 0 get u3 2 get i4 0 get i3 2
1) (integer) 6
2) (integer) 5
3) (integer) 6
4) (integer) -3
127.0.0.1:6379> bitfield w set u8 8 97
1) (integer) 101
127.0.0.1:6379> get w
"hallo"
127.0.0.1:6379> set w hello 
OK
127.0.0.1:6379> bitfield w incrby u4 2 1
1) (integer) 11
127.0.0.1:6379> bitfield w incrby u4 2 1
1) (integer) 12
127.0.0.1:6379> bitfield w incrby u4 2 1
1) (integer) 13
127.0.0.1:6379> bitfield w incrby u4 2 1
1) (integer) 14
127.0.0.1:6379> bitfield w incrby u4 2 1
1) (integer) 15
127.0.0.1:6379> bitfield w incrby u4 2 1    //溢出折返了
1) (integer) 0
```
bitfield 指令提供了溢出策略子指令 overflow，用户可以选择溢出行为，默认是折返（wrap）--报错不执行，还可以选择失败（fail），以及饱和截断（sat）--超过了范围就停留在最大或最小值。overflow 指令只影响接下来的第一条指令，这条指令执行完后溢出策略会变成默认折返（wrap）。

- 饱和截断（sat）
```shell
127.0.0.1:6379> set w hello 
OK
127.0.0.1:6379> bitfield w overflow sat incrby u4 2 1
1) (integer) 11
127.0.0.1:6379> bitfield w overflow sat incrby u4 2 1
1) (integer) 12
127.0.0.1:6379> bitfield w overflow sat incrby u4 2 1
1) (integer) 13
127.0.0.1:6379> bitfield w overflow sat incrby u4 2 1
1) (integer) 14
127.0.0.1:6379> bitfield w overflow sat incrby u4 2 1
1) (integer) 15
127.0.0.1:6379> bitfield w overflow sat incrby u4 2 1
1) (integer) 15
```

- 失败不执行（fail）
```shell
127.0.0.1:6379> bitfield w overflow fail incrby u4 2 1
1) (integer) 11
127.0.0.1:6379> bitfield w overflow fail incrby u4 2 1
1) (integer) 12
127.0.0.1:6379> bitfield w overflow fail incrby u4 2 1
1) (integer) 13
127.0.0.1:6379> bitfield w overflow fail incrby u4 2 1
1) (integer) 14
127.0.0.1:6379> bitfield w overflow fail incrby u4 2 1
1) (integer) 15
127.0.0.1:6379> bitfield w overflow fail incrby u4 2 1
1) (nil)
```

## HyperLogLog

HyperLogLog 提供不精确的去重计数方案，标准误差是 0.81%。

### 使用方法

pfadd 用于增加计数，pfcount 用于获取计数。pfadd 和 set 集合的 sadd 的用法是一样的，pfcount 和 scard 的用法是一样的。
```shell
127.0.0.1:6379> pfadd codehole user1
(integer) 1
127.0.0.1:6379> pfcount codehole
(integer) 1
127.0.0.1:6379> pfadd codehole user2
(integer) 1
127.0.0.1:6379> pfadd codehole user3
(integer) 1
127.0.0.1:6379> pfadd codehole user4
(integer) 1
127.0.0.1:6379> pfadd codehole user5
(integer) 1
127.0.0.1:6379> pfadd codehole user6
(integer) 1
127.0.0.1:6379> pfadd codehole user7
(integer) 1
127.0.0.1:6379> pfadd codehole user8 user9 user10
(integer) 1
127.0.0.1:6379> pfcount codehole
(integer) 10
```

### pfmerge 适合的场景

pfmerge 用于将多个 pf 计数值累加在一起形成一个新的 pf 值。

### 注意事项

HyperLogLog 需要占据 12KB 的存储空间，所以不适合统计单个用户相关的数据。Redis 对 HyperLogLog 的存储进行了优化，在计数比较小时，它的存储空间采用稀疏矩阵存储，空间占用很小，仅仅在计数慢慢变大，稀疏矩阵占用空间渐渐超过了阈值时，才会一次性转变成稠密矩阵，才会占用 12KB 的空间。

### pf 的内存占用为什么是 12KB

在 Redis 的 HyperLogLog 实现中用的是 16384 个桶，也就是 $2^{14}$，每个桶的 maxbits 需要 6 个 bit 来存储，最大可以表示 maxbits = 63，于是总共占用内存就是$2^{14} \times 6 / 8$，算出的结果是 12KB。

## 布隆过滤器

### 布隆过滤器是什么

可以把布隆过滤器理解为一个不怎么精确的 set 结构，当你使用它的 contains 方法判断某个对象是否存在时，它可能会误判。

当布隆过滤器说某个值存在时，这个值可能不存在，当它说某个值不存在时，那就肯定不存在。

### 布隆过滤器的基本用法

布隆过滤器有两个基本指令，bf.add 和 bf.exists。bf.add 添加元素，bf.exists 查询元素是否存在，它们的用法和 set 集合的 sadd 和 sismember 差不多。注意 bf.add 只能一次添加一个元素，如果想要一次添加多个，就需要用到 bf.madd 指令。同样如果需要一次查询多个元素是否存在，就需要用到 bf.mexists 指令。
```shell
127.0.0.1:6379> bf.add codehole user1
(integer) 1
127.0.0.1:6379> bf.add codehole user2
(integer) 1
127.0.0.1:6379> bf.add codehole user3
(integer) 1
127.0.0.1:6379> bf.exists codehole user1
(integer) 1
127.0.0.1:6379> bf.exists codehole user2
(integer) 1
127.0.0.1:6379> bf.exists codehole user3
(integer) 1
127.0.0.1:6379> bf.exists codehole user4
(integer) 0
127.0.0.1:6379> bf.madd codehole user4 user5 user6
1) (integer) 1
2) (integer) 1
3) (integer) 1
127.0.0.1:6379> bf.mexists codehole user4 user5 user6 user7
1) (integer) 1
2) (integer) 1
3) (integer) 1
4) (integer) 0
```

Redis 提供了自定义参数的布隆过滤器，需要在 add 之前使用 bf.reserve 指令显式创建。如果对应的 key 已经存在，bf.reserve 会报错。bf.reserve 有三个参数，分别是 key、error_rate（错误率）和 initial_size。
> 1. error_rate 越低，需要的空间越大；
> 2. initial_size 表示预计放入的元素数量，当实际数量超出这个数值时，误判率会上升，所以需要提前设置一个较大的数值避免超出导致误判率升高；
> 3. 如果不使用 bf.reserve，默认的 error_rate 是 0.01，默认的 initial_size 是 100。

### 注意事项

布隆过滤器的 initial_size 设置的过大，会浪费存储空间，设置得过小，就会影响准确率，用户在使用之前一定要尽可能地精确估计元素数量，还需要加上一定的冗余空间以避免实际元素可能会意外高出估计值很多。

### 布隆过滤器的原理

每个布隆过滤器对应到 Redis 的数据结构里面就是一个大型的位数组和几个不一样的无偏 hash 函数。所谓无偏就是能够把元素的 hash 值算的比较均匀，让元素被 hash 映射到位数组中的位置比较随机。

向布隆过滤器中添加 key 时，会使用多个 hash 函数对 key 进行 hash，算得一个整数索引值，然后对位数组长度进行取模运算得到一个位置，每个 hash 函数都会算得一个不同的位置。再把位数组的这几个位置都置 1，就完成了 add 操作。

向布隆过滤器询问 key 是否存在时，跟 add 一样，也会把 hash 的几个位置都算出来，看看位数组中这几个位置是否都为 1，只要有一个位为 0，那么说明布隆过滤器中这个 key 不能存在。如果这几个位置都是 1，并不能说明这个 key 就一定存在，只是极有可能存在，因为这些位被值为 1 可能是因为其他的 key 存在所致。如果这个位数组比较稀疏，判断正确的概率就会很大，如果这个位数组比较拥挤，判断正确的概率就会降低。

### 空间占用估计

布隆过滤器有两个参数，第一个是预计元素的数量 n，第二个是错误率 f。公式根据这两个输入得到两个输出，第一个输出是位数组的长度 l，也就是需要的存储空间大小（bit），第二个输出是 hash 函数的最佳数量 k。

## GeoHash
### GeoHash 算法

GeoHash 算法将二维坐标映射到一维的整数，这样所有的元素都将挂载到一条直线上，距离靠近的二维坐标映射到一维后的点之间距离也会很接近。当我们想要计算"附近的人"时，首先将目标位置映射到这条线上，然后在这个一维的线上获取附近的点就行了。

通常使用二刀法进行划分，设想一个正方形的蛋糕摆在你面前，二刀下去均分分成四块小正方形，这四个小正方形可以分别标记为 00、01、10、11 共四个二进制整数。然后对每一个小正方形继续用二刀法切割，这时每个小小正方形就可以使用 4bit 的二进制整数表示。然后继续切下去，正方形就会越来越小，二进制整数也会越来越长，精确度就会越来越高。

编码之后，每个地图元素的坐标都将变成一个整数，通过这个整数可以还原出元素的坐标，整数越长，还原出来的坐标值的损失程度就越小。对于"附近的人"这个功能而言，损失的一点精确度可以忽略不计。

GeoHash 算法会继续对这个整数做一次 base32 编码（`0~9,a~z,`去掉 a、i、l、o 四个字母）变成一个字符串。在 Redis 里面，经纬度使用 52 位的整数进行编码，放进了 zset 里面， zset 的 value 是元素的 key，score 是 GeoHash 的 52 位整数值。zset 的 score 虽然是浮点数，但是对于 52 位的整数值，它可以无损存储。

### Geo 指令的基本用法
- 增加
```shell
127.0.0.1:6379> geoadd company 116.48105 39.996794 juejin
(integer) 1
127.0.0.1:6379> geoadd company 116.514203 39.905409 ireader
(integer) 1
127.0.0.1:6379> geoadd company 116.489033 40.007669 meituan 
(integer) 1
127.0.0.1:6379> geoadd company 116.562108 39.787602 jd 116.334255 40.027400 xiaomi
(integer) 2
```
Geo 本质上使用的是 zset 存储，删除元素指令可以直接使用 zrem。
- 距离
```shell
127.0.0.1:6379> geodist company juejin ireader km
"10.5501"
127.0.0.1:6379> geodist company juejin meituan km
"1.3878"
127.0.0.1:6379> geodist company juejin jd km
"24.2739"
127.0.0.1:6379> geodist company juejin xiaomi km
"12.9606"
127.0.0.1:6379> geodist company juejin juejin km
"0.0000"
```

- 获取元素位置

geopos 指令可以获取集合中任意元素的经纬度坐标，可以一次获取多个。
```shell
127.0.0.1:6379> geopos company juejin
1) 1) "116.48104995489120483"
   2) "39.99679348858259686"
127.0.0.1:6379> geopos company ireader
1) 1) "116.5142020583152771"
   2) "39.90540918662494363"
127.0.0.1:6379> geopos company juejin ireader
1) 1) "116.48104995489120483"
   2) "39.99679348858259686"
2) 1) "116.5142020583152771"
   2) "39.90540918662494363"
```
观察到获取的经纬度坐标和 geoadd 进去的坐标有少许误差，原因是 GeoHash 对二维坐标进行的一维映射是有损的。

- 获取元素的 hash 值
```shell
127.0.0.1:6379> geohash company ireader
1) "wx4g52e1ce0"
127.0.0.1:6379> geohash company juejin 
1) "wx4gd94yjn0"
```

- 附近的公司
```shell
127.0.0.1:6379> georadiusbymember company ireader 20 km count 3 asc
1) "ireader"
2) "juejin"
3) "meituan"
127.0.0.1:6379> georadiusbymember company ireader 20 km count 3 desc
1) "jd"
2) "meituan"
3) "juejin"
127.0.0.1:6379> georadiusbymember company ireader 20 km withcoord withdist withhash count 3 asc
1) 1) "ireader"
   2) "0.0000"
   3) (integer) 4069886008361398
   4) 1) "116.5142020583152771"
      2) "39.90540918662494363"
2) 1) "juejin"
   2) "10.5501"
   3) (integer) 4069887154388167
   4) 1) "116.48104995489120483"
      2) "39.99679348858259686"
3) 1) "meituan"
   2) "11.5748"
   3) (integer) 4069887179083478
   4) 1) "116.48903220891952515"
      2) "40.00766997707732031"
127.0.0.1:6379> georadius company 116.514202 39.905409 20 km withdist count 3 asc
1) 1) "ireader"
   2) "0.0000"
2) 1) "juejin"
   2) "10.5501"
3) 1) "meituan"
   2) "11.5748"
```

## scan
Redis 提供了一个简单粗暴的指令 keys 用来列出所有满足特定正则字符串规则的 key。这个指令的使用非常简单，只要提供一个简单的正则字符串即可，但是有两个很明显的缺点。
1. 没有 offset、limit 参数，一次性吐出所有满足条件的 key，有可能实例中有几百万个 key 满足条件；
2. keys 算法是遍历算法，复杂度是 `O(n)`，如果实例中有千万级以上的 key，这个指令就会导致 Redis 服务卡顿。

为了解决上述问题，Redis 在 2.8 版本中引入了 scan，其具有以下特点：
1. 复杂度虽然也是 `O(n)`，但是通过游标分步进行，不会阻塞线程；
2. 提供 limit 参数，可以控制每次返回结果的最大条数。
3. 同 keys 一样，它也提供模式匹配功能；
4. 服务器不需要为游标保存状态，游标的唯一状态就是 scan 返回给客户端的游标整数；
5. 返回的结果可能会有重复，需要客户端去重；
6. 遍历的过程中如果有数据修改，改动后的数据能不能遍历到是不确定的；
7. 单词返回的结果是空的并不意味着遍历结束，而要看返回的游标值是否为零。

### 字典结构
在 Redis 中所有的 key 都存储在一个很大的字典中，这个字典的结构和 Java 中的 HashMap 一样，它是一维数组，是二维链表结构。第一维数组的大小总是 $2^n$，扩容一次数组，大小空间加倍，也就是 $2^{n+1}$。

scan 指令返回的游标就是第一维数组的位置索引，将这个位置索引称为槽（slot）。limit 参数就表示需要遍历的槽位数，之所以返回的结果可能多可能少，是因为不是所有的槽位上都会挂接链表，有些槽位可能是空的，还有些槽位上挂接的链表上的元素可能会有多个，每一次遍历都会将 limit 数量的槽位上挂接的所有链表元素进行模式匹配过滤后，一次性返回给客户端。

### scan 遍历顺序
scan 的遍历顺序非常特别。它不是从第一维数组的第 0 位一直遍历到末尾，而是采用了高位进位加法来遍历。之所以使用这样特殊的方式进行遍历，是考虑到字典的扩容和缩容时避免槽位的遍历重复和遗漏。

### 字典扩容
Java 中的 HashMap 有扩容的概念，当 LoadFactor 达到阈值后，需要重新分配一个新的 2 倍大小的数组，然后将所有的元素全部 rehash 挂到新的数组下面。rehash 就是将元素的 hash 值对数组长度进行取模运算。

假设开始槽位的二进制数是 xxx，那么该槽位中的元素将被 rehash 到 0xxx 和 1xxx 中。

### 渐进式 rehash
Java 的 HashMap 在扩容时会一次性将旧数组下挂接的元素全部转移到新数组下面。如果 HashMap 中的元素特别多，就会出现卡顿现象。Redis 为了解决这个问题，采用"渐进式 rehash"。

它会同时保留旧数组和新数组，然后在定时任务中以及后续对 hash 的指令操作中渐渐地将旧数组中挂接的元素迁移到新数组上。