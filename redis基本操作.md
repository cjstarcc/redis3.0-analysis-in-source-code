# Redis 基本操作

[TOC]

# Redis key

`exists key`:判断键是否存在

`del key`:删除键值对

`move key db`:将键值对移动到制定的数据库

`expire key second`:设置键值对的过期时间

`type key`:查看value的数据类型

```bash
127.0.0.1:6379> keys * # 查看当前数据库所有key
(empty list or set)
127.0.0.1:6379> set name qinjiang # set key
OK
127.0.0.1:6379> set age 20
OK
127.0.0.1:6379> keys *
1) "age"
2) "name"
127.0.0.1:6379> move age 1 # 将键值对移动到指定数据库
(integer) 1
127.0.0.1:6379> EXISTS age # 判断键是否存在
(integer) 0 # 不存在
127.0.0.1:6379> EXISTS name
(integer) 1 # 存在
127.0.0.1:6379> SELECT 1
OK
127.0.0.1:6379[1]> keys *
1) "age"
127.0.0.1:6379[1]> del age # 删除键值对
(integer) 1 # 删除个数


127.0.0.1:6379> set age 20
OK
127.0.0.1:6379> EXPIRE age 15 # 设置键值对的过期时间

(integer) 1 # 设置成功 开始计数
127.0.0.1:6379> ttl age # 查看key的过期剩余时间
(integer) 13
127.0.0.1:6379> ttl age
(integer) 11
127.0.0.1:6379> ttl age
(integer) 9
127.0.0.1:6379> ttl age
(integer) -2 # -2 表示key过期，-1表示key未设置过期时间

127.0.0.1:6379> get age # 过期的key 会被自动delete
(nil)
127.0.0.1:6379> keys *
1) "name"

127.0.0.1:6379> type name # 查看value的数据类型
string
```

关于`TTL`命令

Redis的key，通过TTL命令返回key的过期时间，一般来说有3种：

1. 当前key没有设置过期时间，所以会返回-1.
2. 当前key有设置过期时间，而且key已经过期，所以会返回-2.
3. 当前key有设置过期时间，且key还没有过期，故会返回key的正常剩余时间.

关于重命名`RENAME`和`RENAMENX`

- `RENAME key newkey`修改 key 的名称
- `RENAMENX key newkey`仅当 newkey 不存在时，将 key 改名为 newkey 。



# Redis 五大基本类型

## string(字符串)

```bash
127.0.0.1:6379> FLUSHALL
OK
127.0.0.1:6379> keys *
(empty list or set)
127.0.0.1:6379> set name hxx
OK
127.0.0.1:6379> keys *
1) "name"
127.0.0.1:6379> set age 1
OK
127.0.0.1:6379> keys *
1) "age"
2) "name"
127.0.0.1:6379> EXISTS name
(integer) 1
127.0.0.1:6379> get name
"hxx"
127.0.0.1:6379> move name 1  #移除key到另一个数据库中
(integer) 1
127.0.0.1:6379> keys *
1) "age"
127.0.0.1:6379> set name hxx
OK
127.0.0.1:6379> EXPIRE name 10  #设置超时时间
(integer) 1
127.0.0.1:6379> ttl name   #查看当前key的剩余时间
(integer) 8
127.0.0.1:6379> ttl name
(integer) 6
127.0.0.1:6379> ttl name
(integer) 5
127.0.0.1:6379> ttl name
(integer) 4
127.0.0.1:6379> ttl name
(integer) -2
127.0.0.1:6379> set name hxx   
OK
127.0.0.1:6379> type name  #查看key的类型
string
127.0.0.1:6379> append name xsg  #将value追加到key的value后面
(integer) 6
127.0.0.1:6379> get name
"hxxxsg"
127.0.0.1:6379> set views 0
OK

##################incr incrby key的自增######################

127.0.0.1:6379> incr views  #将value加一
(integer) 1
127.0.0.1:6379> decr views #将value减一
(integer) 0
127.0.0.1:6379> incrby views 10 #将views加10
(integer) 10
#################getrange 获取key的范围##################
127.0.0.1:6379> set key1 helloword 
OK
127.0.0.1:6379> get key1  
"helloword"
127.0.0.1:6379> getrange key1 0 3  #获取key的范围
"hell"
127.0.0.1:6379> getrange key1 0 -1
"helloword"
127.0.0.1:6379> set keys2 abcdefg
OK
127.0.0.1:6379> get keys2
"abcdefg"
127.0.0.1:6379> setrange keys2 0 xx #替换指定位置的字符串
(integer) 7
127.0.0.1:6379> get keys2
"xxcdefg"
##################setex 设置key并设置过期时间###########
127.0.0.1:6379> setex key3 20 hello #设置过期时间
OK
127.0.0.1:6379> ttl key3
(integer) 16
#################setnx 如果不存在就设置，如果存在就返回0###########
127.0.0.1:6379> setnx key4 redis  #如果不存在key则设置value，如果存在那么设置失败
(integer) 1
127.0.0.1:6379> setnx key4 mongoDB
(integer) 0
###############mset 批量设置key###########################
127.0.0.1:6379> mset k1 v1 k2 v2 k3 v3 #批量设置key，value
OK
127.0.0.1:6379> keys *
1) "k1"
2) "k2"
3) "k3"
127.0.0.1:6379> mget k1 k2 k3 #批量获取
1) "v1"
2) "v2"
3) "v3"
127.0.0.1:6379> msetnx k1 v1 k4 v4 #msetnx 是一个原子操作，要么一起成功，要么一起失败
(integer) 0
####################设置对象######################
127.0.0.1:6379> mset user:1:name zhangsan user:1:age 2  #设置一个user:1对象，name，age是他的field值
OK
127.0.0.1:6379> mget user:1:name
1) "zhangsan"
#################getset 设置新值，返回旧值###############
127.0.0.1:6379> getset db redis  #如果没有值，返回失败并设置
(nil)
127.0.0.1:6379> get db
"redis"
127.0.0.1:6379> getset db mysql #如果有值，那么返回旧值
"redis"
```

## list

```bash
############## lpush 增加的元素顺序是反的##############
127.0.0.1:6379> lpush list one
(integer) 1
127.0.0.1:6379> lpush list two
(integer) 2
127.0.0.1:6379> lpush list three
(integer) 3
127.0.0.1:6379> lrange list 0 1
1) "three"
2) "two"
127.0.0.1:6379> lrange list 0 -1
1) "three"
2) "two"
3) "one"
127.0.0.1:6379> keys *
1) "list"
127.0.0.1:6379> rpush list four
(integer) 4
127.0.0.1:6379> lrange list 0 -1
1) "three"
2) "two"
3) "one"
4) "four"
##################### lpop 弹出value并显示  ##################
127.0.0.1:6379> lpop list
"three"
##################### lrange 按范围查看 key ####################
127.0.0.1:6379> lrange list 0 -1
1) "two"
2) "one"
3) "four"
127.0.0.1:6379> rpop list
"four"
127.0.0.1:6379> lrange list 0 -1
1) "two"
2) "one"
#################### lindex 根据索引获取value##########################
127.0.0.1:6379> lindex list 1 
"one"
127.0.0.1:6379> lindex list -1
"one"
127.0.0.1:6379> llen list
(integer) 2
127.0.0.1:6379> lrange list 0 -1
1) "two"
2) "one"
127.0.0.1:6379> lpush list two
(integer) 3
127.0.0.1:6379> lpush list two
(integer) 4
127.0.0.1:6379> lrange list 0 -1
1) "two"
2) "two"
3) "two"
4) "one"
#################按value删除##################
127.0.0.1:6379> lrem list 1 two  #删除一个value为two的值
(integer) 1
127.0.0.1:6379> lrange list 0 -1
1) "two"
2) "two"
3) "one"
127.0.0.1:6379> lrem list 2 two  #删除两个value为two的值
(integer) 2
127.0.0.1:6379> lrange list 0 -1
1) "one"
127.0.0.1:6379> lpush list two
(integer) 2
127.0.0.1:6379> lpush list two three four five 
(integer) 6
127.0.0.1:6379> lrange list 0 -1
1) "five"
2) "four"
3) "three"
4) "two"
5) "two"
6) "one"
#######################ltrim 截取长度##################### 
127.0.0.1:6379> ltrim list 1 2 #截取指定的长度
OK
127.0.0.1:6379> lrange list 0 -1
1) "four"
2) "three"
127.0.0.1:6379> rpoplpush list list1  #移除列表最后一个value并移动到新的列表中
"three"
127.0.0.1:6379> lrange list 0 -1
1) "four"
127.0.0.1:6379> lrange list1 0 -1
1) "three"
127.0.0.1:6379> exists list  #存在list
(integer) 1
127.0.0.1:6379> lset list 0 item  # 替换0位置的value为item
OK
127.0.0.1:6379> lrange list 0 -1
1) "item"
127.0.0.1:6379> rpush mylist hello
(integer) 1
127.0.0.1:6379> rpush mylist world
(integer) 2
#######################3linsert 在value之前或之后插入数据##################
127.0.0.1:6379> linsert mylist  before world other  #在world之前插入value
(integer) 3
127.0.0.1:6379> lrange mylist 0 -1
1) "hello"
2) "other"
3) "world"
127.0.0.1:6379> linsert mylist  after world user
(integer) 4
127.0.0.1:6379> lrange mylist 0 -1
1) "hello"
2) "other"
3) "world"
4) "user"
```

## set

```bash

127.0.0.1:6379> sadd myset hello
(integer) 1
127.0.0.1:6379> sadd myset world
(integer) 1
127.0.0.1:6379> sadd myset hello,world
(integer) 1
127.0.0.1:6379> smembers myset   #查看set中的元素
1) "world"
2) "hello,world"
3) "hello"
127.0.0.1:6379> sismember myset hello  #判断value是否在set中
(integer) 1
127.0.0.1:6379> scard myset   #获取集合中的元素个数
(integer) 3
127.0.0.1:6379> srem myset hello
(integer) 1
127.0.0.1:6379> smembers myset #删除set中的元素
1) "world"
2) "hello,world"
127.0.0.1:6379> srandmember myset 3  #随机取出set中的三个元素
1) "world"
2) "hello,world"
127.0.0.1:6379> srandmember myset 2
1) "world"
2) "hello,world"
127.0.0.1:6379> srandmember myset 1
1) "hello,world"
127.0.0.1:6379> srandmember myset
"world"
127.0.0.1:6379> smembers myset
1) "world"
2) "hello"
3) "hello,world"
127.0.0.1:6379> spop myset  #随机删除set中的一个元素
"world"
127.0.0.1:6379> spop myset
"hello,world"
127.0.0.1:6379> smembers myset
1) "hello"
127.0.0.1:6379> smove myset myset2 hello #移出set中的一个元素到另一个set中
(integer) 1
127.0.0.1:6379> smembers myset
(empty list or set)
127.0.0.1:6379> smembers myset2
1) "hello"
##############sdiff，sinter，sunion 差集并集交集######################
127.0.0.1:6379> sadd key1 a
(integer) 1
127.0.0.1:6379> sadd key1 b
(integer) 1
127.0.0.1:6379> sadd key1 c
(integer) 1
127.0.0.1:6379> sadd key2 c
(integer) 1
127.0.0.1:6379> sadd key2 d
(integer) 1
127.0.0.1:6379> sadd key2 e
(integer) 1
127.0.0.1:6379> sdiff key1 key2  #计算key1 和key2的差集，key1 - key2
1) "b"
2) "a" 
127.0.0.1:6379> sinter key1 key2  #交集
1) "c"
127.0.0.1:6379> sunion key1 key2  #并集
1) "a"
2) "e"
3) "c"
4) "b"
5) "d"
127.0.0.1:6379> sinterstore key3 key1 key2
(integer) 1
127.0.0.1:6379> smembers key3
1) "c"
##########################################
```

## zset

```bash
-------------------ZADD--ZCARD--ZCOUNT--------------
127.0.0.1:6379> ZADD myzset 1 m1 2 m2 3 m3 # 向有序集合myzset中添加成员m1 score=1 以及成员m2 score=2..
(integer) 2
127.0.0.1:6379> ZCARD myzset # 获取有序集合的成员数
(integer) 2
127.0.0.1:6379> ZCOUNT myzset 0 1 # 获取score在 [0,1]区间的成员数量
(integer) 1
127.0.0.1:6379> ZCOUNT myzset 0 2
(integer) 2

----------------ZINCRBY--ZSCORE--------------------------
127.0.0.1:6379> ZINCRBY myzset 5 m2 # 将成员m2的score +5
"7"
127.0.0.1:6379> ZSCORE myzset m1 # 获取成员m1的score
"1"
127.0.0.1:6379> ZSCORE myzset m2
"7"

--------------ZRANK--ZRANGE-----------------------------------
127.0.0.1:6379> ZRANK myzset m1 # 获取成员m1的索引，索引按照score排序，score相同索引值按字典顺序顺序增加
(integer) 0
127.0.0.1:6379> ZRANK myzset m2
(integer) 2
127.0.0.1:6379> ZRANGE myzset 0 1 # 获取索引在 0~1的成员
1) "m1"
2) "m3"
127.0.0.1:6379> ZRANGE myzset 0 -1 # 获取全部成员
1) "m1"
2) "m3"
3) "m2"

#testset=>{abc,add,amaze,apple,back,java,redis} score均为0
------------------ZRANGEBYLEX---------------------------------
127.0.0.1:6379> ZRANGEBYLEX testset - + # 返回所有成员
1) "abc"
2) "add"
3) "amaze"
4) "apple"
5) "back"
6) "java"
7) "redis"
127.0.0.1:6379> ZRANGEBYLEX testset - + LIMIT 0 3 # 分页 按索引显示查询结果的 0,1,2条记录
1) "abc"
2) "add"
3) "amaze"
127.0.0.1:6379> ZRANGEBYLEX testset - + LIMIT 3 3 # 显示 3,4,5条记录
1) "apple"
2) "back"
3) "java"
127.0.0.1:6379> ZRANGEBYLEX testset (- [apple # 显示 (-,apple] 区间内的成员
1) "abc"
2) "add"
3) "amaze"
4) "apple"
127.0.0.1:6379> ZRANGEBYLEX testset [apple [java # 显示 [apple,java]字典区间的成员
1) "apple"
2) "back"
3) "java"

-----------------------ZRANGEBYSCORE---------------------
127.0.0.1:6379> ZRANGEBYSCORE myzset 1 10 # 返回score在 [1,10]之间的的成员
1) "m1"
2) "m3"
3) "m2"
127.0.0.1:6379> ZRANGEBYSCORE myzset 1 5
1) "m1"
2) "m3"

--------------------ZLEXCOUNT-----------------------------
127.0.0.1:6379> ZLEXCOUNT testset - +
(integer) 7
127.0.0.1:6379> ZLEXCOUNT testset [apple [java
(integer) 3

------------------ZREM--ZREMRANGEBYLEX--ZREMRANGBYRANK--ZREMRANGEBYSCORE--------------------------------
127.0.0.1:6379> ZREM testset abc # 移除成员abc
(integer) 1
127.0.0.1:6379> ZREMRANGEBYLEX testset [apple [java # 移除字典区间[apple,java]中的所有成员
(integer) 3
127.0.0.1:6379> ZREMRANGEBYRANK testset 0 1 # 移除排名0~1的所有成员
(integer) 2
127.0.0.1:6379> ZREMRANGEBYSCORE myzset 0 3 # 移除score在 [0,3]的成员
(integer) 2


# testset=> {abc,add,apple,amaze,back,java,redis} score均为0
# myzset=> {(m1,1),(m2,2),(m3,3),(m4,4),(m7,7),(m9,9)}
----------------ZREVRANGE--ZREVRANGEBYSCORE--ZREVRANGEBYLEX-----------
127.0.0.1:6379> ZREVRANGE myzset 0 3 # 按score递减排序，然后按索引，返回结果的 0~3
1) "m9"
2) "m7"
3) "m4"
4) "m3"
127.0.0.1:6379> ZREVRANGE myzset 2 4 # 返回排序结果的 索引的2~4
1) "m4"
2) "m3"
3) "m2"
127.0.0.1:6379> ZREVRANGEBYSCORE myzset 6 2 # 按score递减顺序 返回集合中分数在[2,6]之间的成员
1) "m4"
2) "m3"
3) "m2"
127.0.0.1:6379> ZREVRANGEBYLEX testset [java (add # 按字典倒序 返回集合中(add,java]字典区间的成员
1) "java"
2) "back"
3) "apple"
4) "amaze"

-------------------------ZREVRANK------------------------------
127.0.0.1:6379> ZREVRANK myzset m7 # 按score递减顺序，返回成员m7索引
(integer) 1
127.0.0.1:6379> ZREVRANK myzset m2
(integer) 4


# mathscore=>{(xm,90),(xh,95),(xg,87)} 小明、小红、小刚的数学成绩
# enscore=>{(xm,70),(xh,93),(xg,90)} 小明、小红、小刚的英语成绩
-------------------ZINTERSTORE--ZUNIONSTORE-----------------------------------
127.0.0.1:6379> ZINTERSTORE sumscore 2 mathscore enscore # 将mathscore enscore进行合并 结果存放到sumscore
(integer) 3
127.0.0.1:6379> ZRANGE sumscore 0 -1 withscores # 合并后的score是之前集合中所有score的和
1) "xm"
2) "160"
3) "xg"
4) "177"
5) "xh"
6) "188"

127.0.0.1:6379> ZUNIONSTORE lowestscore 2 mathscore enscore AGGREGATE MIN # 取两个集合的成员score最小值作为结果的
(integer) 3
127.0.0.1:6379> ZRANGE lowestscore 0 -1 withscores
1) "xm"
2) "70"
3) "xg"
4) "87"
5) "xh"
6) "93"
```



## hash

```bash
127.0.0.1:6379> hset myhash field1 hxx
(integer) 1
127.0.0.1:6379> hget myhash field1
"hxx"
127.0.0.1:6379> hmset myhash field2 xsg
OK
127.0.0.1:6379> hmset myhash field2 xsg field1 hxx
OK
127.0.0.1:6379> hmget myhash field1 field2
1) "hxx"
2) "xsg"
127.0.0.1:6379> hmset myhash field2 xsg field1 shuaige
OK
127.0.0.1:6379> hmget myhash field1 field2
1) "shuaige"
2) "xsg"
127.0.0.1:6379> hgetall myhash
1) "field1"
2) "shuaige"
3) "field2"
4) "xsg"
127.0.0.1:6379> hdel myhash field1 #删除hash中的key-value
(integer) 1
127.0.0.1:6379> hgetall myhash #获取所有的hash
1) "field2"
2) "xsg"
127.0.0.1:6379> hlen myhash  #获取hash的长度
(integer) 1
127.0.0.1:6379> hexists myhash field1  #判断key值是否存在
(integer) 0
127.0.0.1:6379> hexists myhash field2
(integer) 1
127.0.0.1:6379> hvals myhash   #获取hash中的value
1) "xsg"
127.0.0.1:6379> hkeys myhash   #获取hash中的key
1) "field2"
127.0.0.1:6379> hset myhash field3 5 
(integer) 1
127.0.0.1:6379> hincrby myhash field3 1  #hash自增
(integer) 6
127.0.0.1:6379> hsetnx myhash field4 hello  #如果有可以设置，如果没有则不能设置
(integer) 1
127.0.0.1:6379> hsetnx myhash field4 world
(integer) 0
127.0.0.1:6379> hset user:1 name qinjiang #按照对象存储
(integer) 1
127.0.0.1:6379> hget user:1 name
"qinjiang"
```

# 事务

```bash
127.0.0.1:6379> MULTI  # 开始事务
OK
127.0.0.1:6379> exec
(empty list or set)
127.0.0.1:6379> flushdb
OK
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379> set k1 v1
QUEUED
127.0.0.1:6379> set k2 v2
QUEUED
127.0.0.1:6379> get k2
QUEUED
127.0.0.1:6379> set k3 v3
QUEUED
127.0.0.1:6379> exec  #执行事务
1) OK
2) OK
3) "v2"
4) OK
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379> set k1 v1
QUEUED
127.0.0.1:6379> set k2 v2
QUEUED
127.0.0.1:6379> set k4 v4
QUEUED
127.0.0.1:6379> DISCARD #放弃事务
OK
####################
127.0.0.1:6379> set money 100
OK
127.0.0.1:6379> set out 0
OK
127.0.0.1:6379> watch money #监控事务
OK
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379> decrby money 20
QUEUED
127.0.0.1:6379> incrby out 20
QUEUED

*******************************与此同时在另一个客户端执行
127.0.0.1:6379> set money 80
OK
*******************************回到之前客户端
127.0.0.1:6379> exec   #当有修改时事务修改不成功
(nil)
####################

```



# 引用

https://blog.csdn.net/DDDDeng_/article/details/108118544

