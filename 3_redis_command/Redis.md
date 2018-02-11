[《Redis开发运维实践指南》原书地址](https://gnuhpc.gitbooks.io/redis-all-about/content/DataStructure/key/list.html)
# Redis简介：
Redis是一个运行在内存中的数据结构服务器。Redis使用的是单进程，所以在配置时，一个实例只会用到一个CPU。

## 数据操作

###key操作
```
keys *list*           //*：通配符，通配任意多个字符
keys *                  
keys l?ist*           //？：通配单个字符
keys li[i]t*          //[]:通配括号内的某个字符
```

注：生产环境已经禁止使用。更安全的做法是采用scan，原理如下图：
<![](https://raw.githubusercontent.com/gnuhpc/All-About-Redis/master/DataStructure/key/scan-intro.png)
![](https://raw.githubusercontent.com/gnuhpc/All-About-Redis/master/DataStructure/key/scan.png)
<![](https://raw.githubusercontent.com/gnuhpc/All-About-Redis/master/DataStructure/key/scan-other.png)
![]()

##### 常用命令：
```
127.0.0.1:6379> scan 0 match *
1) "0"
2) 1) "list2"
   2) "myset"
   3) "guo"
   4) "list1"
----------------------------------------------------
127.0.0.1:6379> sadd myset 1 2 3 foo foobar feelsgoog
(integer) 6
127.0.0.1:6379> sscan myset 0 match f*
1) "0"
2) 1) "feelsgoog"
   2) "foo"
   3) "foobar"
127.0.0.1:6379> scan 0 match *
```

Redis-cli下的扫描：
```
redis-cli --scan --pattern 'list*'
```
设置用scan命令扫描redis中的key，--pattern选项指定扫描的key的pattern。相比key pattern模式，不会藏时间阻塞redis，而导致其他客户端的命令请求一直处于阻塞状态。


[参考地址：](http://redisdoc.com/key/scan.html)

SCAN 命令、 SSCAN 命令、 HSCAN 命令和 ZSCAN 命令都返回一个包含两个元素的 multi-bulk 回复： 回复的第一个元素是字符串表示的无符号 64 位整数（游标）， 回复的第二个元素是另一个 multi-bulk 回复， 这个 multi-bulk 回复包含了本次被迭代的元素。

- SCAN 命令返回的每个元素都是一个数据库键。

- SSCAN 命令返回的每个元素都是一个集合成员。

- HSCAN 命令返回的每个元素都是一个键值对，一个键值对由一个键和一个值组成。

- ZSCAN 命令返回的每个元素都是一个有序集合元素，一个有序集合元素由一个成员（member）和一个分值（score）组成。

本页中采用了小象学院的两张片子，版权归 http://www.chinahadoop.cn/ 所有

##### 1、测试指定的key是否存在
```
# exists guo
```
返回1表示存在，0不存在

##### 2、删除给定key

```
del key1 key2 ...keyn
```
返回1表示存在，0不存在

##### 3、返回给定key的value类型

```
127.0.0.1:6379> type guo
list
127.0.0.1:6379> type myset
set
```
返回none表示不存在的key，String字符类型，list链表类型，set无序集合类型


##### 4、返回从当前数据库中随机选择的一个key
```
127.0.0.1:6379> randomkey
"myset"
```
如果当前数据库为空，返回空串

##### 5、原子的重命名一个key
```
rename oldkey newkey
```
如果newkey存在则会被覆盖，返回1表示成功，0失败。

##### 6、key的超时处理
```
expire key seconds

pexpire key 毫秒数
```
单位是秒，返回1成功，0表示已经设置过过期时间了或者不存在。如果要消除超时则使用persist key。

### 字符串
最大字符串为512M，但是大字符串非常不建议使用。

##### 1、设置key对应的值为string类型的value

```
set key value [ex 秒数] / [px 毫秒数]  [nx] /[xx]

返回1表示成功，0失败

setnx key value  //仅当key不存在时才set，nx表示not exist。

----------------------------------------------------
mset key1 value1   key2 value2 .。。

一次设置多个key，

msetnx key1 value1 .。。。   //不会覆盖已存在的值
```
##### 2、获取key对应的string值

```
get key       //如果key不存在返回null

mget key1 key2 ... keyN     //一次获取多个key的值，如果对于的key不存在，则返回null

getset key value

127.0.0.1:6379> set nihao 'buhao'
OK
127.0.0.1:6379> getset nihao 'hao'
"buhao"

```
原子的设置key的值，并返回key的旧值，如果key不存在返回null。
应用场景：设置新值，返回旧值，配合setnx可实现分布式。

分布式锁的思路：注意该思路要保证多台Client服务器的NTP一致。
- 1、C3发送SETNX lock.foo 想要获得锁，由于C0还持有锁，所以Redis返回给C3一个0
- 2、C3发送GET lock.foo 以检查锁是否超时了，如果没超时，则等待或重试。
- 3、反之，如果已超时，C3通过下面的操作来尝试获得锁：
- 4、GETSET lock.foo
- 5、通过GETSET，C3拿到的时间戳如果仍然是超时的，那就说明，C3如愿以偿拿到锁了。
- 6、如果在C3之前，有个叫C4的客户端比C3快一步执行了上面的操作，那么C3拿到的时间戳是个未超时的值，这时，C3没有如期获得锁，需要再次等待或重试。留意一下，尽管C3没拿到锁，但它改写了C4设置的锁的超时值，不过这一点非常微小的误差带来的影响可以忽略不计。

伪代码：

```
# get lock
lock = 0
while lock != 1:
    timestamp = current Unix time + lock timeout + 1
    lock = SETNX lock.foo timestamp
    if lock == 1 or (now() > (GET lock.foo) and now() > (GETSET lock.foo timestamp)):
        break;
    else:
        sleep(10ms)

# do your job
do_job()

# release
if now() < GET lock.foo:
    DEL lock.foo
```

##### 增减操作

```
incr key             //对key做加加操作，
decr key             //对key做减减操作

incrby key interge   //对key加指定的数

incrbyfloat key floatnumber    //针对浮点数


127.0.0.1:6379> set num 1
OK
127.0.0.1:6379> get num
"1"
127.0.0.1:6379> incr num
(integer) 2
127.0.0.1:6379> incr num
(integer) 3
127.0.0.1:6379> get num
"3"
127.0.0.1:6379> decr num
(integer) 2
127.0.0.1:6379> decr num
(integer) 1
127.0.0.1:6379> get num
"1"
127.0.0.1:6379> incrby num 10000000000000000
(integer) 10000000000000001
127.0.0.1:6379> get num
"10000000000000001"
127.0.0.1:6379> decrby num 999999999999
(integer) 9999000000000002

```

![](https://raw.githubusercontent.com/gnuhpc/All-About-Redis/master/DataStructure/string/incrfloat.png)

可以操作的数
![](https://raw.githubusercontent.com/gnuhpc/All-About-Redis/master/DataStructure/string/incr1.png)

这个操作的应用场景：计数器

##### 追加字符串
```
append key value              //返回新字符的长度

```

##### 截取字符串
```
substr key start end           //并不会修改字符串，返回截取的字符串
```
##### 改写字符串
```
setrange key offset value      //offset表示偏移量
```

##### 返回子串
```
getrange key start end  
```

##### 中文字符串操作
![](https://raw.githubusercontent.com/gnuhpc/All-About-Redis/master/DataStructure/string/chinese.png)

##### 取指定key的value值的长度
```
strlen  key
```

## 列表操作：

##### 添加元素
```
lpush key string                               //在key对应的list的头部添加元素
rpush key string                               //在list的尾部添加元素

 lpushx key value                              //如果key不存在，什么都不做
 rpushx key value                              //同上

linsert key BEFORE|AFTER pivot value            //在list对应的位置之前或之后
```

##### 查看列表对应的长度

```
llen key
```

##### 查看元素
```
lindex key index                          //指定索引的位置，0第一个
```

##### 查看一段列表
```
lrange key start end    
lrange key 0 -1                           // -1表示返回所有数据           
```

##### 截取list
```
ltrim key start end                        //保留指定区间的元素
```
##### 删除元素
```
ldel
```

##### 设置list中指定下标的元素值
```
lset key index value                         //idnex表示指定索引的位置
```

##### 阻塞队列
```
blpop key [key ...] timeout
brpop key [key ...] timeout
```

### 集合操作

##### 添加元素

```
sadd key member               //成功返回1，
```

##### 移除元素
```
srem key member               //成功返回1
```

##### 删除并返回元素

```
spop key                     //如果set是空或者不存在则返回null
```                   
##### 随机返回一个元素
```
srandmember key              //同spop，随机取set中一个元素，但是不删除
```

##### 集合间移动元素  
```
smove srckey dstkey member
```

###### 查看集合的大小
```
scard key                         //如果set是空或者key不存在则返回0
```
##### 判断member是否在set中
```
sismember key member             //存在返回1，0表示不存在或key不存在
```

##### 集合交集
```
sinter key1 key2                  //返回所有给定key的交集
sinterstore dstkey key1 key2      //同sinter，并同时保存并集到dstkey下
```
##### 集合并集
```
sunion key1 key2                 //返回所有给定key的并集
sunionstore dstkey key1 key2      //同sunion，并同时保存并集到dstkey下
```
##### 集合差集
```
sdiff key1 key2                   //返回给定key的差集
sdiffstore dstkey key1 key2       //同sdiff，并同时保存并集到dstkey下
```
##### 获取所有元素
```
smembers  key                      //返回key对应的所有元素，结果是无序的哦
```

### 有序集合

Sorted Set的实现是hash table(element->score, 用于实现ZScore及判断element是否在集合内)，和skip list(score->element,按score排序)的混合体。 skip list有点像平衡二叉树那样，不同范围的score被分成一层一层，每层是一个按score排序的链表。
ZAdd/ZRem是O(log(N))，ZRangeByScore/ZRemRangeByScore是O(log(N)+M)，N是Set大小，M是结果/操作元素的个数。可见，原本可能很大的N被很关键的Log了一下，1000万大小的Set，复杂度也只是几十不到。当然，如果一次命中很多元素M很大那谁也没办法了。

##### 添加元素
```
zadd key score member             //添加元素到集合，元素在集合中存在则更新对应的score
```
##### 删除元素
```
zrem key member                   //1表示成功，如果元素不存在则返回0

zremrangebyrank min max           //删除集合中排名在给定的区间

```
##### 增加score
```
zincrvy key member                //增加对于member的score的值。
```
##### 获取排名
 ```
 zrank key member                 //返回指定元素在集合中的排名，
 zrebrank key member              //同时，但是集合中的元素是按score从大到小排序的
 ```

##### 获取排行榜
```
zrange key start end            //类似lrange操作从集合中去指定区间的元素，返回时有序的。
```

##### 返回给定分数区间的元素
```
zrangebyscore key min max             
```
##### 返回集合中score在给定区间的数量
```
zcount key min max    
```

#####返回集合中元素的个数
```
zcard key  
```
##### 返回给定元素对应的score
```
zscore key element  
```
##### 评分的聚合
```
ZUNIONSTORE destination numkeys key [key ...] [WEIGHTS weight] [AGGREGATE SUM|MIN|MAX]
```

### 哈希操作
底层实现是hash table，一般操作复杂度是O(1)，要同时操作多个field时就是O(N)，N是field的数量。应用场景：土法建索引。比如User对象，除了id有时还要按name来查询。


#####  设置hash值
```
hset key field value              //设置hash field为指定值，如果key不存在，则先创建
hsetnx                             //同时，如果存在返回0，nx是not exist的意思
hmset key filed1 value1 ... filedN valueN      //设置多个值
```
##### 获取hash值
```
hget key field                     //获取指定的hash field
hmget key field1 field2            //获取全部指定的field
```
##### 递增某一个域的值
```
hincrby key field integer          //将指定的hash field加上给定值
```

##### 判断某一个域是否存在
```
hexists key field                     //测试指定field是否存在
```

##### 删除域
```
hdel  key field                         //删除指定的field
```

##### 获取域的数量
```
hlen key                                  //返回会指定hash的field数量
```

##### 获取所有的域
```
hkeys key                               //返回hash的所有field
```

##### 获取所有域的值
```
hvals  key                               //返回hash所有的value
```

##### 获取所有域名和值
```
hgetall                                   //返回hash所有field和value
```
### HyperLogLog操作

#####将元素添加至HyperLogLog
```
PFADD key element
```
##### 返回给定 HyperLogLog 的基数估算值

```
PFCOUNT key [key ...]
```

##### 合并多个HyperLogLog
```
PFMERGE destkey sourcekey [sourcekey ...]
```

## 专题功能

### 排序
redis支持对list，set和sorted set元素的排序。排序命令是sort 完整的命令格式如下：
```
SORT key [BY pattern] [LIMIT start count] [GET pattern] [ASC|DESC] [ALPHA] [STORE dstkey]
```
复杂度为O(N+M*log(M))。(N是集合大小，M 为返回元素的数量)

说明：
- [ASC|DESC] [ALPHA]: sort默认的排序方式（asc）是从小到大排的,当然也可以按照逆序或者按字符顺序排。
- [BY pattern] : 除了可以按集合元素自身值排序外，还可以将集合元素内容按照给定pattern组合成新的key，并按照新key中对应的内容进行排序。例如：
- 127.0.0.1:6379sort watch:leto by severtity:* desc
- [GET pattern]：可以通过get选项去获取指定pattern作为新key对应的值，get选项可以有多个。例如：127.0.0.1:6379sort watch:leto by severtity: get severtity:。 -对于Hash的引用，采用->，例如：sort watch:leto get # get bug: * ->priority。
- [LIMIT start count] 限定返回结果的数量。
- [STORE dstkey] 把排序结果缓存起来

![](https://raw.githubusercontent.com/gnuhpc/All-About-Redis/master/IndependentFunc/sort1.png)

![](https://raw.githubusercontent.com/gnuhpc/All-About-Redis/master/IndependentFunc/sort2.png)

### 事务
用Multi(Start Transaction)、Exec(Commit)、Discard(Rollback)实现。 在事务提交前，不会执行任何指令，只会把它们存到一个队列里，不影响其他客户端的操作。在事务提交时，批量执行所有指令。 一般情况下redis在接受到一个client发来的命令后会立即处理并返回处理结果，但是当一个client在一个连接中发出multi命令后，这个连接会进入一个事务上下文，该连接后续的命令并不是立即执行，而是先放到一个队列中。当从此连接受到exec命令后，redis会顺序的执行队列中的所有命令。并将所有命令的运行结果打包到一起返回给client.然后此连接就结束事务上下文。

Redis还提供了一个Watch功能，你可以对一个key进行Watch，然后再执行Transactions，在这过程中，如果这个Watched的值进行了修改，那么这个Transactions会发现并拒绝执行。

### 流水线

利用流水线（pipeline）的方式从client打包多条命令一起发出，不需要等待单条命令的响应返回，而redis服务端会处理完多条命令后会将多条命令的处理结果打包到一起返回给客户端

```
cat data.txt | redis-cli –pipe
```
在选择开源redis开发库时需要着重注意是否支持pipeline，常见的jedis可以支持。

### 发布订阅

redis作为一个pub/sub server，在订阅者和发布者之间起到了消息路由的功能。订阅者可以通过subscribe和psubscribe命令向redis server订阅自己感兴趣的消息类型，redis将消息类型称为频道(channel)。当发布者通过publish命令向redis server发送特定类型的消息时。订阅该消息类型的全部client都会收到此消息。这里消息的传递是多对多的。一个client可以订阅多个 channel,也可以向多个channel发送消息。



#### 持久化设置
RDB和AOF两者毫无关系，完全独立运行，如果使用了AOF，重启时只会从AOF文件载入数据，不会再管RDB文件。


#### 常见运维操作

##### 启动
```
redis-server redis.conf
```
常见选项： ./redis-server (run the server with default conf) ./redis-server /etc/redis/6379.conf ./redis-server --port 7777 ./redis-server --port 7777 --slaveof 127.0.0.1 8888 ./redis-server /etc/myredis.conf --loglevel verbose

##### 启动redis-sentinel
```
./redis-server /etc/sentinel.conf –sentinel
 ./redis-sentinel /etc/sentinel.conf
```
部署后可以使用sstart对redis 和sentinel进行拉起，使用sctl进行supervisorctl的控制。（两个alias）

##### 停止
```
redis-cli shutdown
```
sentinel方法一样，只是需要执行sentinel的连接端口

注意：正确关闭服务器方式是redis-cli shutdown 或者 kill，都会graceful shutdown，保证写RDB文件以及将AOF文件fsync到磁盘，不会丢失数据。 如果是粗暴的Ctrl+C，或者kill -9 就可能丢失。如果有配置save，还希望在shutdown时进行RDB写入，那么请使用shutdown save命令。

#### 选择数据库

```
select db-index
```
默认连接的数据库所有是0,默认数据库数是16个。返回1表示成功，0失败

#### 清空数据库

```
flushdb                           //删除当前选择数据库中的所有key。生产上已经禁止
flushall                          //删除所有数据库，
```

#### 重命名
```
rename-command
```
例如：rename-command FLUSHALL ""。必须重启。


#### 设置密码
```
config set requirepass  [passw0rd]
```

### 备份

对于RDB和AOF，都是直接拷贝文件即可，可以设定crontab进行定时备份： cp /var/lib/redis/dump.rdb /somewhere/safe/dump.$(date +%Y%m%d%H%M).rdb

## 恢复

如果只使用了RDB，则首先将redis-server停掉，删除dump.rdb，最后将备份的dump.rdb文件拷贝回data目录并修改相关属主保证其属主和redis-server启动用户一致，然后启动redis-server。
































































-
