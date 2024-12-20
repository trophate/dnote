# 数据结构

String、List、Hash、Set、Zset

### 

# 底层结构

Redis 7



## String

SDS（动态字符串）

SDS优点：

1. 获取长度效率为O(1)。
2. 自动扩容，减少空间分配次数。
3. 字符不会产生歧义（\0）。



## List

listpack、quicklist



## Hash

listpack、hashtable



## Set

intset、listpack、hashtable



## Zset

listpack、skiplist



# 持久化

快照、AOF、混合持久化



## 快照

快照存储某时刻的所有数据。

缺点：服务器宕机会丢失最近快照之后的数据。



## AOF

AOF记录操作，将每次操作追加到AOF文件末尾。AOF可以设定同步频率。

缺点：文件巨大、恢复缓慢。

### 重写/压缩

该功能用于压缩AOF文件。



## 混合持久化

混合持久化先将快照数据存入AOF文件，然后在文件末尾追加最新操作。



# 应用问题



## 缓存雪崩

缓存在同一时间失效引发大量数据库访问，导致数据库压力激增。



## 缓存击穿

热点缓存失效导致访问直达数据库，引发数据库压力激增。



## 缓存穿透

大量访问不存在的数据，由于数据不会被缓存，访问直达数据库导致数据库压力激增。