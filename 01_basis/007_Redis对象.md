## 对象

​		在前面我们介绍了Redis用到的所有主要数据结构，比如简单动态字符串（SDS）、双端链表、字典、压缩列表、整数集合等等。		

​		Redis并没有直接使用数据结构来实现键值对数据库，而是基于这些数据结构创建了一个对象系统，这个系统包含字符串对象、列表对象、哈希对象、集合对象和有序集合对象这五种类型的对象。Redis的对象系统还实现了基于引用计数的内存回收机制，当程序不再使用某个对象的时候，这个对象所占用的内存就会被自动释放；另外，Redis还通过引用计数技术实现了对象共享机制，这一机制可以再适当的条件下，通过让多个数据库键共享同一个对象来节约内存。

#### 1、对象的类型与编码

​		Redis使用对象来表示数据库中的键和值，每次当我们再Redis的数据库中新创建一个键值对时，我们至少会创建两个对象，一个对象用作键值对的键（键对象），另一个对象用作键值对的值（值对象）。

​		Redis中的每个对象都由一个redisObject结构表示：

```c
typedef struct redisObject {
    // 类型
    unsigned type:4;
    // 编码
    unsigned encoding:4;
    unsigned lru:REDIS_LRU_BITS; /* lru time (relative to server.lruclock) */
    int refcount;
    // 指向底层数据结构实现的指针
    void *ptr;
} robj;
```



##### 1.1、类型

​		对象的type属性记录了对象的类型。这个属性的值可以是下面列出的常量的其中一个。

|   类型常量   |  对象的名称  |
| :----------: | :----------: |
| REDIS_STRING |  字符串对象  |
|  REDIS_LIST  |   列表对象   |
|  REDIS_HASH  |   哈希对象   |
|  REDIS_SET   |   集合对象   |
|  REDIS_ZSET  | 有序集合对象 |

​		对于Redis数据库保存的键值对来说，键总是一个字符串对象，而值则可以是字符串对象、列表对象、哈希对象、集合对象或者有序集合对象的其中一种。

​		$TYPE$命令的实现方式也与此类似，当我们对一个数据库键执行TYPE命令时，返回的结果为数据库对应的值对象的类型，而不是键对象的类型：

```shell
# 键为字符串对象，值为字符串对象
127.0.0.1:6379> set msg "hello world"
OK
127.0.0.1:6379> type msg
string
# 键为字符串对象，值为列表对象
127.0.0.1:6379> rpush numbers 1 3 5
(integer) 3
127.0.0.1:6379> type numbers
list
```



##### 1.2、编码和底层实现

​		对象的ptr指针指向对象的底层实现数据结构，而这些数据结构由对象的encoding属性决定。

​		encoding属性记录了对象所使用的编码，也即是说这个对象使用了什么数据结构作为对象的底层实现。

```c
// 简单动态字符串
#define REDIS_ENCODING_RAW 0     		/* Raw representation */
// embstr编码的简单动态字符串
#define REDIS_ENCODING_EMBSTR
// long类型的整数
#define REDIS_ENCODING_INT 1     		/* Encoded as integer */
// 字典
#define REDIS_ENCODING_HT 2      		/* Encoded as hash table */
#define REDIS_ENCODING_ZIPMAP 3  		/* Encoded as zipmap */
// 双端链表
#define REDIS_ENCODING_LINKEDLIST 4 /* Encoded as regular linked list */
// 压缩列表
#define REDIS_ENCODING_ZIPLIST 5 		/* Encoded as ziplist */
// 整数集合
#define REDIS_ENCODING_INTSET 6  		/* Encoded as intset */
// 跳跃表和字典
#define REDIS_ENCODING_SKIPLIST 7  	/* Encoded as skiplist */
```



#### 2、字符串对象

​		字符串对象的编码可以是int、raw或者embstr

