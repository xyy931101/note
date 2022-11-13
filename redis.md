# Redis键值对

​	为了实现从键到值的快速访问，Redis 使用了一个哈希表来保存所有键值对。一个哈希表，其实就是一个数组，数组的每个元素称为一个哈希桶。所以，我们常说，一个哈希表是由多个哈希桶组成的，每个哈希桶中保存了键值对数据。

在下图中，可以看到，哈希桶中的 entry 元素中保存了*key和*value指针，分别指向了实际的键和值，这样一来，即使值是一个集合，也可以通过*value指针被查找到

![](image\redis\Redis全局hash表结构.jpg)

因为这个哈希表保存了所有的键值对，所以，我也把它称为全局哈希表。哈希表的最大好处很明显，就是让我们可以用 O(1) 的时间复杂度来快速查找到键值对——我们只需要计算键的哈希值，就可以知道它所对应的哈希桶位置，然后就可以访问相应的 entry 元素。

## Redis hash冲突

​		Redis 解决哈希冲突的方式，就是链式哈希。链式哈希也很容易理解，就是指同一个哈希桶中的多个元素用一个链表来保存，它们之间依次用指针连接。

## 渐进式 rehash

Redis 默认使用了两个全局哈希表：哈希表 1 和哈希表 2。一开始，当你刚插入数据时，默认使用哈希表 1，此时的哈希表 2 并没有被分配空间。随着数据逐步增多，Redis 开始执行 rehash，**在rehash过程中，更删改等操作都会在两个表进行，查则会现在ht[0]中，不存在再去ht[1]中查找**。这个过程分为以下5步：

1. 为ht[1]分配空间（第一个大于等于ht[0].used*2的2的N次方幂），让字典同时拥有ht[0]和ht[1]两个哈希表
2. 在字典中维持所以rehashidx，并将其值设置为0，表示rehash工作正式开始
3. 在rehash期间，每次对哈希表进行添加、删除、更新或者查找操作时，程序出来执行指定的操作外，还会顺带将ht[0]上的值rehash到ht[1]中，当rehash操作完成时2，将rehashidx属性的值增1.
4. 随着字典的不断操作，最终在某个时间点上，ht[0]的所有键值对都会被rehash到ht[1]上，这是将rehashidx的值设为-1，表示操作完成
5. 将ht[1]设置为ht[0],然后为ht[1]分配一个空白的哈希表

![](image\redis\渐进式rehash.jpg)

# Redis的数据结构

## SDS 字符串

1. 常数复杂度获取字符串长度

   相较于C字符串，获取字符串长度的复杂度是O1，切设置SDS长度是调用API时自动完成的

2. 杜绝缓冲区溢出

   C字符串不记录本身字符串的长度，假设调用stack函数，如果某一字符串空间不够，则有可能将内存中其他字符串内容进行覆盖，从而产生缓冲区溢出。而SDS在对内容进行修改时，会检查空间是否满足修改所需的需求，不满足的话，会先将SDS的空间拓展至所需的大小，而后再进行修改

3. 减少修改字符串长度时所需的内存重新分配

   1. 空间预分配：对SDS字符进行修改时，若SDS 小于1MB，则会分配len同等长度的未使用空间。若大于1MB则会分配多1MB的未使用空间。在拓展SDS空间之前，会检查free属性是否够用，如果够用则不再进行预分配。

   2. 惰性释放空间，使用free属性记录未使用的字节数量，避免了缩短字符时所需的对内存重新分配操作，并未将来可能有可能的增长操作提供了优化。并且有提供API进行SDS空间的释放，所以不必担心造成内存浪费

      **在一般程序中，修改字符串长度不太常见，那么每次修改都执行一次是可以接收的。但是redis作为数据库，对于速度要求严苛，字符串被修改的常见也经常出现**

4. 二进制安全

5. 兼容部分C字符串函数

```c
struct sdshdr {

    // buf 中已占用空间的长度
    int len;

    // buf 中剩余可用空间的长度
    int free;

    // 字节数组
    char buf[];
};
```



## LinkList

特性：

- 双端：链表节点带有prev和next指针，获取前节点和后节点复杂度是O(1)
- 无环：表头节点prev和表尾节点next都是指向NULL
- 带有头指针和尾指针：通过list节后的head、tail指针，获取链表头节点和尾节点是O(1)
- 带有长度计数器，获取链表长度复杂度为O(1)
- 多态：链表节点使用void*指针来保存节点属性，并且可以通过list结构的dup、free、match三个属性节点设置特定函数，所以节点可以保存不同类型的值

```c
//链表结构
typedef struct list{
    listNode *head;  			//头节点
    listNode *tail;				//尾节点
    unsigned long len; 			//链表长度
    void *(*dup)(void *ptr);	//节点复制函数
    void (*free)(void *ptr);	//节点释放函数
    int (*match)(void *ptr, void *key);	//节点值对比函数
}list;

//链表节点
typedef struct listNode{
    struct listNode *prev;	//前置节点
    struct listNode *next;	//后置节点
    void * value;			//节点的值
}listNode；
```



## Hashtable

- 字典被广泛应用于Redis，其中包括数据库和哈希键
- Redis'中的字段使用hashtable作为底层实现，每个字段带有两个哈希表，一个平时使用，一个仅在rehash时使用。及渐进式hash
- 当字段被用作数据库的底层实现，或者哈希键的底层实现是，使用MurmurHash2来计算键的哈希值
- 哈希表使用链地址法来解决键冲突，被分配到同一个索引上的键值会采用**头插法**形成单向链表
- 在进行哈希表扩展或收缩时，程序需要将现有哈希表包含的所有键值对rehash到新的hash表中，并且这个rehash过程并不是一次性完成的，而是渐进式的。

```c

typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];  
    long rehashidx;		//当rehash不在进行时,值为-1,如果不为-1,则代表当前rehash的进度
    int16_t pauserehash;//
} dict;

//定义一个hash桶，用来管理hashtable
typedef struct dictht {		//管理hashtable
    dictEntry **table;		//指针数组，这个hash的桶
    unsigned long size;		//元素个数
    unsigned long sizemask;	//此字段的作用是当使用下标访问数据时，确保下标不越界。
    unsigned long used;		//存在多少个元素
} dictht;

//hash节点
typedef struct dictEntry {//hash节点
    void *key;//键，一半是sds
    union {//联合体牛逼，导致值可以多用途，可以存储指针或存储整数或浮点数。必定可以存储上面类型之前，通过不同的引用，告诉编译器如何解释对应的内存块。
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next;//下一个节点，解决碰撞冲突
} dictEntry;
```

### hash表的扩展与收缩

​	当以下条件任意一个被满足时候，程序会对哈希表进行拓展操作，其中负载因子计算公式 load_factor = ht[0].used / ht[0].size

1. 服务器目前没有执行BGSAVE或者BGREWRITEAOF命令，并且哈希表的负载因子大于1
2. 服务器目前正在进行BGSAVE或BGREWRITEAOF命令，并且哈希表的负载因子大于5
3. 当负载因子小于**0.1**时，会对哈希表进行收缩操作

## Ziplist(压缩列表)

ziplist是由一系列特殊编码的连续内存块组成的顺序存储结构，类似于数组，ziplist在内存中是连续存储的，但是不同于数组，为了节省内存 ziplist的每个元素所占的内存大小可以不同（数组中叫元素，ziplist叫节点entry，下文都用“节点”），每个节点可以用来存储一个整数或者一个字符串。

![](image\redis\ziplist.png)

```c
typedef struct zlentry {
    unsigned int prevrawlensize;
    unsigned int prevrawlen;
     
    unsigned int lensize;                                 
    unsigned int len;
            
    unsigned int headersize; 
    
    unsigned char encoding;
      
    unsigned char *p;           
} zlentry;
```



zlbytes:	 ziplist的长度（4字节)，是一个32位无符号整数，在对压缩列表进行内存重分配或计算zlend时使用
zltail:	 	ziplist最后一个节点的偏移量（4字节)，反向遍历ziplist或者pop尾部节点的时候有用。
zllen:	 	ziplist的节点（entry）个数（2字节）
entry:		节点
zlend: 		值为0xFF，用于标记ziplist的结尾

ziplist将一些必要的偏移量信息记录在了每一个节点里，使之能跳到上一个节点或下一个节点。
接下来我们看看节点的布局

节点的布局(entry)

每个节点由三部分组成：<prevlen> <encoding> <entry-data>

## Skiplist(跳跃表)

​	主要用于有序集合键(**有序集合节点数>128或值的长度大于64**)，集群节点中用作内部数据结构

![image-20210329234923723](image\skiplist.jpg)

**实现**

为了满足自身的功能需要， Redis 基于 William Pugh 论文中描述的跳跃表进行了以下修改：

1. 允许重复的 `score` 值：多个不同的 `member` 的 `score` 值可以相同。
2. 进行对比操作时，不仅要检查 `score` 值，还要检查 `member` ：当 `score` 值可以重复时，单靠 `score` 值无法判断一个元素的身份，所以需要连 `member` 域都一并检查才行。
3. 每个节点都带有一个高度为 1 层的后退指针，用于从表尾方向向表头方向迭代：当执行 [ZREVRANGE](http://redis.readthedocs.org/en/latest/sorted_set/zrevrange.html#zrevrange) 或 [ZREVRANGEBYSCORE](http://redis.readthedocs.org/en/latest/sorted_set/zrevrangebyscore.html#zrevrangebyscore) 这类以逆序处理有序集的命令时，就会用到这个属性。
4. 每个节点的层高都是1-32之前的随机数
5. 跳跃表是一种随机化数据结构，查找、添加、删除操作都可以在对数期望时间下完成。
6. 跳跃表目前在 Redis 的唯一作用，就是作为有序集类型的底层数据结构（之一，另一个构成有序集的结构是字典）。
7. 为了满足自身的需求，Redis 基于 William Pugh 论文中描述的跳跃表进行了修改，包括：
   1. `score` 值可重复。
   2. 对比一个元素需要同时检查它的 `score` 和 `memeber` 。
   3. 每个节点带有高度为 1 层的后退指针，用于从表尾方向向表头方向迭代。

这个修改版的跳跃表由 `redis.h/zskiplist` 结构定义：

```c
typedef struct zskiplist {
    // 头节点，尾节点
    struct zskiplistNode *header, *tail;
    // 节点数量
    unsigned long length;
    // 目前表内节点的最大层数
    int level;
} zskiplist;

```

跳跃表的节点由 `redis.h/zskiplistNode` 定义：

```c
typedef struct zskiplistNode {
    // member 对象
    robj *obj;
    // 分值
    double score;
    // 后退指针
    struct zskiplistNode *backward;
    // 层
    struct zskiplistLevel {
        // 前进指针
        struct zskiplistNode *forward;
        // 这个层跨越的节点数量
        unsigned int span;
    } level[];

} zskiplistNode;
```

## IntSet（整数集合）

- 整数集合是集合键的底层实现之一
- 整数集合作为数组，这个数组以**有序**、**无重复**的方式保存集合元素。在有需要时，程序会根据新添加的元素的类型改变这个数组的类型
- 升级操作作为整数集合带来了操作上的灵活性，**并尽可能的节约了内存**
- 整数集合只支持升级操作，**不支持降级操作**

```c
typedef struct intset {
    uint32_t encoding;	//intset的类型编码
    uint32_t length; 	//成员元素的个数
    int8_t contents[];	//用来存储成员的柔性数组
}
```

| 编码方式 | 范围                   |
| -------- | ---------------------- |
| int16_t  | -32768~32767           |
| int32_t  | -2147483648~2147483647 |
| int64_t  | -2^63 ~ 2^63-1         |

### intset升级操作

1. 根据新元素的类型，扩展整个整数集合底层数组的空间大小，并为新元素分配空间
2. 将底层数组现有的所有元素都转换成与新元素相同的类型，并将类型转换后的元素放置在正确的位上，而且在放置元素的过程中，需要继续维持底层数据的有序性
3. 将新元素添加到底层数组里面



------

# Redis的数据类型

## redis对象

- Redis数据库中的每个键值对的键和值都是一个对象
- Redis共有字符串、列表、哈希、集合、有序集合五种类型的对象，每种对象都有两种以上的编码，不同对象在不同的使用场景上优化对象的使用
- 服务器在执行某些命令之前，会先检查给定键的类型是否能执行指定的命令，而检查一个键的类型就是检查键的值得对象的类型
- Redis的对象系统带有引用计数器实现内存的回收，当一个对象不再被使用时，该对象所占用的内存就会被自动释放
- Redis会共享0-9999的字符串对象
- 对象会记录自己的最后一次被访问的时间，这个时间可以用于计算对象的空转时间

```c
typedef struct redisObject {
    unsigned type:4;			//数据类型(String, List, Set. Hash, Zset)
    unsigned encoding:4;		//编码格式(int, emstr,raw,hash,ziplist,skiplist)
    unsigned lru:24; 				/* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    int refcount; 				//引用计数
    void *ptr;					//对象指针
} robj;
```



## String（字符串对象）

- **int** 当一个字符串是整数值得话，是以int类型保存的

- **raw** SDS (Simple Dynamic String) 简单动态字符串  保存的是字符串值，且长度大于**44**

  **redisObj  sdshdr两个结构都会申请内存**

- **embstr**  字符串长度小于44

  1. embstr创建时，内存分配次数从raw的两次降为一次
  
     **减少了sdshdr的内存分配,在创建redisObj的时候直接申请了小于64字节的内存，基于缓存行**

  2. 释放时，只需要调用一次内存释放函数，raw需要两次
  
  3. 所有数据都保存在一块，所以速度更快
  
  **embstr实际上是只读的，当我们对embstr进行修改的时候，无论长度多少，总会变成raw**

## lsit（列表对象）

列表对象的编码可以是ziplist也可以是linkedlist.

当列表对戏满足一下条件是，对象列表使用ziplist

- 所有字符串长度小于64字节
- 列表保存的元素小于512个

## hash（哈希对象）

哈希对象可以是ziplist或者hashtable

**ziplist**：每当有新的键值对插入时，程序会把保存了键的压缩列表推入到压缩列表表尾，然后再将保存了值的压缩列表节点推入到压缩列表表位

- 保存了同一键值对的两个节点总数在一起，保存键的节点在前，保存值得节点在后。
- 先添加到哈希对象中的键值，会被放到压缩列表的表头方向而后添加的键值会被放在压缩列表的表尾

**hashtable** 使用字典作为底层实现

- 字典的每个键作为键值对的键
- 字典的值作为键值对的值

当hash对象满足一下条件时，使用ziplist编码

1. 哈希对象的所有键值对的键和值得长度都小于64字节

2. 哈希对象键值对数量小于512个。

   <!--以上条件可以通过配置文件hash-max-ziplist-value、hash-max-ziplist-enties修改-->

## set(集合对象）

集合对象的编码是**intset**或**hashtable**

当满足一下条件时，作为IntSet保存

- 集合对象所有元素都整数
- 集合对象保存的元素不超过512个

## ZSET(有序集合对象)

有序集合的编码可以是ziplist和skiplist

ziplist作为有序集合时候，每个集合使用紧紧相邻的两个节点来保存，第一个节点保存元素的成员(member)，第二个节点保存节点的元素分数(source)

使用ziplsit作为有序集合的条件

- 有序集合的数量小于128个

- 有序集合所有元素成员长度都小于64

  

## Redis数据结构与数据类型的关系

![](image\redis\redis数据类型.jpg)

# 单机数据库

## 数据库键空间

```c
typedef struct redisDb {
    dict *dict;                 /* The keyspace for this DB */
    dict *expires;              /* Timeout of keys with a timeout set */
    dict *blocking_keys;        /* Keys with clients waiting for data (BLPOP)*/
    dict *ready_keys;           /* Blocked keys that received a PUSH */
    dict *watched_keys;         /* WATCHED keys for MULTI/EXEC CAS */
    int id;                     /* Database ID */
    long long avg_ttl;          /* Average TTL, just for stats */
    unsigned long expires_cursor; /* Cursor of the active expire cycle. */
    list *defrag_later;         /* List of key names to attempt to defrag one by one, gradually. */
} redisDb;
```

服务器中每个数据库都是一个redisDb结构的dict字典保存了所有数据库的所有键值对，我们将这个字段称为**键空间**（key space）

- 键空间的所有的键都是字符串对象
- 键空间的值可以是字符串对象，列表对象，哈希对象、集合对象和有序集合对象

## Redis的过期键的删除策略

我们都知道，Redis是key-value数据库，我们可以设置Redis中缓存的key的过期时间。(四种命令,但是最后走的都是PEXPIREAT )

- EXPIRE 将键key的生存时间设置为ttl秒
- PEXPIRE 将键key的生存时间设置为ttl毫秒
- EXPIREAT 将键key的过期时间设置为timestamp所设置的秒数时间戳
- PEXPIREAT 将键key的过期时间设置为timestamp锁设置的毫秒时间戳

Redis的过期策略就是指当Redis中缓存的key过期了，在RedisDB中，专门有expires字典保存过期时间与键值的关系

过期策略通常有以下三种：

- **定时过期**：每个设置过期时间的key都需要创建一个定时器，到过期时间就会立即清除。该策略可以立即清除过期的数据，对内存很友好；但是会占用大量的CPU资源去处理过期的数据，从而影响缓存的响应时间和吞吐量。
- **惰性过期**：只有当访问一个key时，才会判断该key是否已过期，过期则清除。该策略可以最大化地节省CPU资源，却对内存非常不友好。极端情况可能出现大量的过期key没有再次被访问，从而不会被清除，占用大量内存。
- **定期过期**：每隔一定的时间，会扫描一定数量的数据库的expires字典中一定数量的key，并清除其中已过期的key。该策略是前两者的一个折中方案。通过调整定时扫描的时间间隔和每次扫描的限定耗时，可以在不同情况下使得CPU和内存资源达到最优的平衡效果。

(expires字典会保存所有设置了过期时间的key的过期时间数据，其中，key是指向键空间中的某个键的指针，value是该键的毫秒精度的UNIX时间戳表示的过期时间。键空间是指该Redis集群中保存的所有键。)

Redis中同时使用了**惰性过期和定期过期**两种过期策略。

惰性删除：所有的读写操作会经过expireIfNeeded函数,如果过期就了删除,不执行下面的流程

```c
robj *lookupKeyWriteWithFlags(redisDb *db, robj *key, int flags) {
    expireIfNeeded(db,key);
    return lookupKey(db,key,flags);
}

int expireIfNeeded(redisDb *db, robj *key) {
    if (!keyIsExpired(db,key)) return 0;

    if (server.masterhost != NULL) return 1;
    if (checkClientPauseTimeoutAndReturnIfPaused()) return 1;
    /* Delete the key */
    deleteExpiredKeyAndPropagate(db,key);
    return 1;
}
```

定期删除：每当redis服务器执行周期性函数 **redis.c/sercerCron** 会执行**activeExpireCycle** 从过期字典(**expires**)中随机获取一部分过期的键进行删除

## Redis的内存淘汰策略

Redis的内存淘汰策略是指在Redis的用于缓存的内存不足时，怎么处理需要新写入且需要申请额外空间的数据。

**全局的键空间选择性移除**

- **noeviction**：当内存不足以容纳新写入数据时，新写入操作会报错。
- **allkeys-lru**：当内存不足以容纳新写入数据时，在键空间中，移除最近最少使用的key。（这个是最常用的）
- **allkeys-random**：当内存不足以容纳新写入数据时，在键空间中，随机移除某个key。

**设置过期时间的键空间选择性移除**

- **volatile-lru**：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，移除最近最少使用的key。
- **volatile-random：**当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，随机移除某个key。
- **volatile-ttl：**当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，有更早过期时间的key优先移除。

# Redis的持久化

## AOF(Append Only File)日志

​	redis的日志回写顺序是：

1. 执行数据库命令，将数据写入内存
2. 写入aof_buf缓冲区
3. serverCorn将缓冲区内容写进AOF文件

**好处**

1. 写日志时候不检查语法,在写入命令正确后，再记录日志。避免额外开销。
2. 避免对当前命令的阻塞

#### 三种回写策略

1. **Always** 命令执行完之后立即回写
2. **Everysec**  每写入一个命令之后，把日志写入到AOF文件缓冲区,然后每隔一秒把缓冲区的内容写回磁盘(默认值),**并且这个线程是由一个专门的线程执行的**
3. **No** 操作系统控制的回写，只是把日志写到AOF文件缓冲区，由操作系统决定何时回写磁盘

#### 重写机制

为了避免AOF文件过大,redis有AOF文件重写机制。(为了避免执行命令时造成客户端输入缓冲区溢出，重写程序在处理列表、哈希、集合、有序集合时会先检查键所饱含的元素个数，如果超过了redis.h/REDIS_AOF_REWRITE_PRE_CMD:默认64，那么重写程序会使用多条命令来记录键的值，而不是一条命令)

​		**原理**：通过直接读取当前数据库的所有的KEY值(如果键已经过期了,则直接忽略)，直接生成一条命令(当键的所包含的元素数量超过64时，会生成多条命令)，替换之前记录这个键操作的所有记录。

​		**auto-aof-rewrite-percentage** ：配置了当 aof 文件相较于上一版本的 aof 文件大小的百分比达到多少时触发 AOF 重写。举个例子，auto-aof-rewrite-percentage 选项配置为 100，上一版本的 aof 文件大小为 100M，那么当我们的 aof 文件达到 200M 的时候，触发 AOF 重写。

- AOF重写期间，服务器进程可以(父进程)可以继续处理命令请求
- 子进程带有服务器进程的数据副本,使用子进程而不是线程，**可以在避免使用锁的情况下，保证数据的安全**

重写过程由后台子进程bfrewriteaof执行,会fork主线程的内存记录重写日志，还会把当前主线程在重写期间的回写日志记录到重写日志的缓冲区。当AOF重写工作后，会向父进行发送信号，当父进行接收到信号后，会进行以下操作

1. 将AOF重写缓冲区所有数据写入到新的AOF文件，保持数据库跟AOF文件的状态一致
2. 对新的AOF文件进行改名，原子的（atomic）覆盖现有的AOF文件，完成新旧AOF文件的替换

![6b054eb1aed0734bd81ddab9a31d0be8](image\6b054eb1aed0734bd81ddab9a31d0be8.jpg)



## RDB(内存快照)

Redis 提供了两个命令来生成 RDB 文件，分别是 **SAVE** 和 **BGSAVE**

**SAVE**：在主线程中执行，会导致阻塞；

**BGSAVE**：创建一个子进程，专门用于写入 RDB 文件，避免了主线程的阻塞，这也是 Redis RDB 文件生成的默认配置。在此过程中Redis 就会借助操作系统提供的写时复制技术（**Copy-On-Write, COW**），在执行快照的同时，正常处理写操作。子进程是由主线程 fork 生成的，可以共享主线程的所有内存数据。bgsave 子进程运行后，开始读取主线程的内存数据，并把它们写入 

### BGSAVE的触发条件

数据结构：

```c
struct redisServer{
    //修改计数器(上次SAVE或BGSAVE之后，每次数据库进进行修改，都会加1)
    long long dirty;
    //上次成功执行SAVE或BGSAVE操作的时间
    time_t lastsave; 
    //记录保存条件的数据
    struct saveparam *saveparams;
}
struct saveparam{
    //秒数
    time_t seconds; 
    //修改次数
    int changes;
}
```

redis的serverCorn会遍历redisServer中的saveparams数组，只要有其中一个条件满足了，就会触发其BGSAVE命令：

比如说save的选项为以下条件：

```c
save 900 1		//服务器在900s之内，对数据库至少进行了一次修改
save 300 10		//服务器在300s之内，对服务器至少进行了10次修改 
save 60 1000	//服务器在60s之内，对服务器至少进行了10000次修改
```

伪代码

```apl
def serverCorn():
#...
#遍历所有保存条件
for saveparam in server.saveparams:
	#计算距离上次保存时间有多少秒
	save_interval = unixtime_now() - server.lastsave;
	#如果数据库状态修改次数超过条件所设置的次数
	#并且距离上次修改的时间超过条件设置的时间
	if server.dirty >= saveparam.changes and save_interval > saveparam.seconds
		#则执行保存操作
		BGSAVE();
#...
	
```



### RDB文件结构

| REDIS | db_version | databases              | EOF  | check_sum     |
| ----- | ---------- | ---------------------- | ---- | ------------- |
| REDIS | 0001       | database 0\|database 1 | EOF  | check_sum的值 |

database 0部分内容：

| SELECTDB | db_number | key_value_pairs       |
| -------- | --------- | --------------------- |
| SELECTDB | 0         | key_value_pairs的内容 |

1. 五个REDIS字符,标识是redis的RDB文件

2. 四个字节的版本号

3. database 标识多少数据库，及具体的数据库内容（其中database0代表0号数据库的具体内容）

   1. SELECTDB 0(这里标识具体的费控数据库)

   2. key_value_pairs 保存了所有数据的键值对

      组成：EXPIRETIME_MS(过期时间标识，如果没有过期时间则没有)、ms(具体的过期时时间戳)、TYPE(value的对象类型)、key(键)、value(值的具体对象)

4. EOF常量 标识RDB文件的正文结束

5. check_sum 校验和

它拍的是一张内存的“大合影”，不可避免地会耗时耗力。虽然，Redis 设计了 bgsave 和写时复制方式，尽可能减少了内存快照对正常读写的影响，但是，频繁快照仍然是不太能接受的。而混合使用 RDB 和 AOF，正好可以取两者之长，避两者之短，以较小的性能开销保证数据可靠性和性能。



![4dc5fb99a1c94f70957cce1ffef419cc](image\4dc5fb99a1c94f70957cce1ffef419cc.jpg)



# **Redis线程模型**

Redis基于Reactor模式开发了网络事件处理器，这个处理器被称为文件事件处理器（file event handler）。它的组成结构为4部分：多个套接字、IO多路复用程序、文件事件分派器、事件处理器。因为文件事件分派器队列的消费是单线程的，所以Redis才叫单线程模型。

- 文件事件处理器使用 I/O 多路复用（multiplexing）程序来同时监听多个套接字， 并根据套接字目前执行的任务来为套接字关联不同的事件处理器。
- 当被监听的套接字准备好执行连接应答（accept）、读取（read）、写入（write）、关闭（close）等操作时， 与操作相对应的文件事件就会产生， 这时文件事件处理器就会调用套接字之前关联好的事件处理器来处理这些事件。
- 文件事件处理器以单线程方式运行， 但通过使用 I/O 多路复用程序来监听多个套接字， 文件事件处理器既实现了高性能的网络通信模型， 又可以很好地与 redis 服务器中其他同样以单线程方式运行的模块进行对接， 这保持了 Redis 内部单线程设计的简单性

虽然文件事件处理器以单线程方式运行， 但通过使用 I/O 多路复用程序来监听多个套接字， 文件事件处理器既实现了高性能的网络通信模型， 又可以很好地与 redis 服务器中其他同样以单线程方式运行的模块进行对接， 这保持了 Redis 内部单线程设计的简单性。

![](image\redis\redis线程模型.png)

![](image\redis\redis.png)

# 事件

## 文件事件

**accpetTcpHandler**:连接应答处理器

​	当有客户端连接时，会产生AE_READABLE事件，引发连接应答处理器，并执行相应的套接字应答处理操作。

**readQueryFromClient:** 命令请求处理器

​	负责从套接字中读取客户端发送的命令

**sendReplyToClient**:命令回复处理器

​	负责将服务器执行命令后得到的命令，命令回复处理器将套接字返回给客户端。

## 时间事件

- 定时事件：让一段程序在指定的时间后执行，
- 周期性事件：让一段程序在

事件的调度与执行

```c
def aeProcessEvents
  #获取到达事件离当前事件最近的时间事件
  time_event = aeSearchNearestTimer();

	#计算最接近的时间事件还有多少好眠
	remaind_ms = time_event.when - unixts_now();

	#若果事件已经到达,那么remaind_ms的值可能为负数,将其设置为0
	if remaind_ms < 0
    remaind_ms = 0
    
  #根据remaind_ms的值，创建timeval结构
  timeval = create_timeval_with_ms(remaind_ms);

	#阻塞并等待文件事件产生，最大阻塞时间由传入的timeval结构决定
	#如果remaind_ms值为0,那么aeApiPoll调用之后立马返回,不阻塞
	aeApiPoll(timeval);
	
	#处理所有产生的文件事件
	processFileEvents();

	#处理所有已到达的时间事件
	processTimeEvents();
```

## 命令请求过程

1. 客户端发送命令请求
2. 读取命令请求
   - 读取套接字中协议格式命令请求，并将其保存到客户端的输入缓冲区
   - 对输入缓冲区中的命令请求进行分析，提取出命令参数及命令个数
   - 调用执行命令，执行客户端指定的命令
3. 命令执行器
   1. 查找命令实现
   2. 执行预备操作
      - 检查客户端的状态，cmd指针是否指向NULL
      - 检查客户端是否已经通过了身份验证，未通过只能执行AUTH命令
      - 如果服务器打开了maxmemory功能，在执行命令前先检查服务器内存，并在有需要时进行内存回收
   3. 调用命令函数的实现
   4. 执行后续工作
      - 如果开启了慢查询日志功能，那么慢查询日志模块会检查是否需要为刚刚执行的命令添加慢查询日志
      - 根据刚才命令耗时时长，更新被执行命令的redisCommand结构的milliseconds属性
      - 如果开启了AOF，那么AOF模块会将刚刚执行的命令写入AOF缓冲区
      - 如果有其他从服务器在复制当前服务器，则将刚才执行的命令传播给所有从服务器，且写入复制积压缓冲区
4. 将命令回复给客户端
5. 客户端接收并打印命令回复

# redisServerCorn

是redis的周期性操作的关键，其每100毫秒会执行一次（ Redis 2.8 开始， 用户可以通过修改 `hz` 选项来调整 `serverCron`的每秒执行次数），进而执行下面的相关操作（在执行完文件事件之后会处理时间事件，时间事件就是serverCorn，这个也是**主线程执行**的）

- 更新服务器的各类统计信息，比如时间、内存占用、数据库占用情况等
- 清理数据库中的过期键值对
- 对不合理的数据库进行大小调整
- 关闭和清理连接失效的客户端
- 尝试进行 AOF 或 RDB 持久化操作
- 如果服务器是主节点的话，对附属节点进行定期同步
- 如果处于集群模式的话，对集群进行定期同步和连接测试

# Redis高可用

## 单机问题

1. 单机故障
2. 容量有限
3. socket,cpu压力

## 主从同步

- 读操作：主库、从库都可以接收；
- 写操作：首先到主库执行，然后，主库将写操作同步给从库。
- 过期key: 主库会向从库追加一个DEL的命令

### 主从库同步过程

- 第一阶段是主从库间建立连接、协商同步的过程，主要是为全量复制做准备。在这一步，从库和主库建立起连接，并告诉主库即将进行同步，主库确认回复后，主从库间就可以开始同步了。
- 在第二阶段，主库将所有数据同步给从库。从库收到数据后，在本地完成数据加载。这个过程依赖于内存快照生成的 RDB 文件。
- 最后，也就是第三个阶段，主库会把第二阶段执行过程中新收到的写命令，再发送给从库。具体的操作是，当主库完成 RDB 文件发送后，就会把此时 replication buffer 中的修改操作发给从库，从库再重新执行这些操作。这样一来，主从库就实现同步了。

![redis主从过程](image\redis\redis主从过程.jpg)

增量复制时，主从库之间具体是怎么保持同步的呢？这里的奥妙就在于 repl_backlog_buffer (默认大小为1MB)这个缓冲区。我们先来看下它是如何用于增量命令的同步的。当主从库断连后，主库会把断连期间收到的写操作命令，写入 replication buffer，同时也会把这些操作命令也写入 repl_backlog_buffer 这个缓冲区。repl_backlog_buffer 是一个环形缓冲区(**FIFO**)，主库会记录自己写到的位置，从库则会记录自己已经读到的位置。当从节点的内容已经不在环形缓冲区内了，则会触发BGSAVE

![redis主从增量复制](image\redis\redis主从增量复制.jpg)

------

### 心跳检查

在命令传播阶段，从服务器默认会以每秒一次的频率向主服务器发送命令：

REPLCONF ACK <replication_offest>    replication_offest为从服务器当前的复制偏移量

发送REPLCONF ACK命令对于主服务器有3个作用

- 检测主从服务器的网络连接状态
- 辅助实现min-slaves选项（min-slaves-to-write、min-slaves-max-lag 即大于toWrite的从服务器延迟大于lag的值，则拒绝执行命令）
- 检测命令丢失

## **哨兵集群  Redis  Sentinel**

### seninel的网络连接

基于 pub/sub 机制的哨兵集群组成哨兵实例之间可以相互发现，要归功于 Redis 提供的 pub/sub 机制，也就是发布 / 订阅机制。即：

1. 向master发送连接命令，并接受命令回复
2. 订阅连接，__sentinel__:hello，获取其他sentinel的信息。

#### 获取主服务器信息

每10秒钟发送INFO命令,获取当前主服务器的信息，通过命令回复可以获取以下信息

1. 主服务器本身信息，包括run_id及role域记录的角色信息
2. 获取从服务器信息
   - 从服务器的运行ID run_id
   - 从服务器的角色role
   - 从服务的master_hodt+master_port
   - 从服务器的连接状态master_link_status
   - 从服务器的优先级slave_priorty
   - 从服务器的复制偏移量slave_repl_offset

哨兵只要和主库建立起了连接，就可以在主库上发布消息了，比如说发布它自己的连接信息（IP 和端口）。同时，它也可以从主库上订阅消息，获得其他哨兵发布的连接信息。当多个哨兵实例都在主库上做了发布和订阅操作后，它们之间就能知道彼此的 IP 地址和端口。

![](image\redis哨兵.jpg)



sentinel，中文名是哨兵。哨兵是 redis 集群机构中非常重要的一个组件，主要有以下功能：

- **集群监控**：负责监控 redis master 和 slave 进程是否正常工作。
- **消息通知**：如果某个 redis 实例有故障，那么哨兵负责发送消息作为报警通知给管理员。
- **故障转移**：如果 master node 挂掉了，会自动转移到 slave node 上。
- **配置中心**：如果故障转移发生了，通知 client 客户端新的 master 地址。

sentinel在对判断服务器是否下线有两种：即主观下线与客观下线。

​	默认情况下，sentinel会每秒一次的频率想服务器发送消息，当超过配置的时间内没有收到回复，则认为是主观下线了。之后向其他sentinel进行询问，当超过半数的sentinel认为服务器下线时，会判定服务器为客观下线，继而进行故障转移。

### Sentinel的leader的选举

确定由哪个哨兵执行主从切换的过程，和主库“客观下线”的判断过程类似，也是一个“投票仲裁”的过程。在具体了解这个过程前，我们再来看下，判断“客观下线”的仲裁过程。哨兵集群要判定主库“客观下线”，需要有一定数量的实例都认为该主库已经“主观下线”了。我在上节课向你介绍了判断“客观下线”的原则，接下来，我介绍下具体的判断过程。任何一个实例只要自身判断主库“主观下线”后，就会给其他实例发送 is-master-down-by-addr 命令。接着，其他实例会根据自己和主库的连接情况，做出 Y 或 N 的响应，Y 相当于赞成票，N 相当于反对票。

![](image\redis\哨兵leader选举.jpg)

1. 判断主库“主观下线”后，给其他实例发送 is-master-down-by-addr 命令

2. 接着，其他实例会根据自己和主库的连接情况，做出 Y 或 N 的响应，Y 相当于赞成票，N 相当于反对票。

3. 判断所获得的赞成票是否大于哨兵配置文件中的 quorum配置项，之后标记主库为“客观下线”

4. 向其他哨兵实例发送leader选举，并且给自己投一票赞成票Y，其他实例回复Y/N。

5. 当获得半数以上的赞成票，成为leader。开始redis的主从切换

   > 哨兵对主从库进行的在线状态检查等操作，是属于一种时间事件，用一个定时器来完成，一般来说每100ms执行一次这些事件。每个哨兵的定时器执行周期都会加上一个小小的随机时间偏移，目的是让每个哨兵执行上述操作的时间能稍微错开些，也是为了避免它们都同时判定主库下线，同时选举Leader。

在投票过程中，任何一个想成为 Leader 的哨兵，要满足两个条件：

- 第一，拿到半数以上的赞成票；
- 第二，拿到的票数同时还需要大于等于哨兵配置文件中的 quorum 值。

### **哨兵选出新的master的过程**

1. 删除列表中所有处于下线状态的从服务器
2. 删除列表中所有最近5秒内没有回复sentinel leader的从服务器
3. 删除所有与下线主服务器连接断开超过down-after-milliseconds * 10毫秒的从服务器
4. 根据从服务器的优先级，对列表中的从服务器进行排序，并选出优先级最高的从服务器
5. 若具有多个相同的最高优先级服务器，则按照从服务器的复制偏移量进行排序，并选出偏移量最大的服务器
6. 若有多个优先级最高，复制偏移量最大的从服务器，则根据运行时ID进行排序，选出其中run_id最小的

当哨兵把新主库选择出来后，客户端就会看到下面的 switch-master 事件。这个事件表示主库已经切换了，新主库的 IP 地址和端口信息已经有了。这个时候，客户端就可以用这里面的新主库地址和端口进行通信了。

```c++
switch-master <master name> <oldip> <oldport> <newip> <newport>
```



## **切片集群   Redis Cluster**

![](image\redis cluster.jpg)

**简介**

Redis Cluster是一种服务端Sharding技术，3.0版本开始正式提供。Redis Cluster并没有使用一致性hash，而是采用slot(槽)的概念，一共分成16384个槽。将请求发送到任意节点，接收到请求的节点会将查询请求发送到正确的节点上执行

集群构建过程：利用 CLUSTER MEET <IP> <PORT>命令

收到命令的节点A 将会与节点B进行握手，以此来确认彼此的存在，并为下一步通信打好基础

1. 节点A为节点B创建一个clusterNode结构，并将该结构添加到自己的clusterState.node字典里面
2. 节点A根据CLUSTER MEET命令给定的IP和端口号，向节点B发送MEET命令
3. 如果一切顺利，节点B会为节点A创建clusterNode结构，并将该结构添加到自己的clusterState.node字典里面
4. 之后，节点B将向节点A返回PONG消息
5. 如果一切顺利，节点A将接受节点B返回的PONG命令，从而知晓节点B已经成功接受自己的MEET命令
6. 之后节点A将向节点B发送一条PING命令
7. 如果一切顺利，节点B将接受节点A的PING命令，从而知道节点A已经成功接受自己的返回消息，握手成功

**槽指派 方案说明**

- 通过哈希的方式(CRC-16)，将数据分片，每个节点均分存储一定哈希槽(哈希值)区间的数据，默认分配了16384 个槽位
- 每份数据分片会存储在多个互为主从的多节点上(clusterState.slots数组,存储槽位属于那个分片)
- 数据写入先写主节点，再同步到从节点(支持配置为阻塞同步)
- 同一分片多个节点间的数据不保持一致性
- 读取数据时，当客户端操作的key没有分配在该节点上时，redis会返回转向指令，指向正确的节点
- 扩容时时需要需要把旧节点的数据迁移一部分到新节点

在 redis cluster 架构下，每个 redis 要放开两个端口号，比如一个是 6379，另外一个就是 加1w 的端口号，比如 16379。

16379 端口号是用来进行节点间通信的，也就是 cluster bus 的东西，cluster bus 的通信，用来进行故障检测、配置更新、故障转移授权。cluster bus 用了另外一种二进制的协议，gossip 协议，用于节点间进行高效的数据交换，占用更少的网络带宽和处理时间。

### **节点间的内部通信机制**

（基本通信原理）集群元数据的维护有两种方式：集中式、Gossip 协议。redis cluster 节点间采用 gossip 协议进行通信。

#### 重定向机制

所谓的“重定向”，就是指，客户端给一个实例发送数据读写操作时，这个实例上并没有相应的数据，客户端要再给一个新实例发送操作命令。那客户端又是怎么知道重定向时的新实例的访问地址呢？当客户端把一个键值对的操作请求发给一个实例时，如果这个实例上并没有这个键值对映射的哈希槽，那么，这个实例就会给客户端返回下面的 MOVED 命令响应结果，这个结果中就包含了新实例的访问地址

```
GET hello:key
(error) MOVED 13320 172.16.19.5:6379
```

其中，MOVED 命令表示，客户端请求的键值对所在的哈希槽 13320，实际是在 172.16.19.5 这个实例上。通过返回的 MOVED 命令，就相当于把哈希槽所在的新实例的信息告诉给客户端了。这样一来，客户端就可以直接和 172.16.19.5 连接，并发送操作请求了。

MOVED错误格式为:

​	`<slot> <ip>:<port>`

**一个客户端实际上会与集群的所有节点建立套接字连接，所谓的节点转向，其实就是换一个套接字来发送命令**

#### 重新分片

当集群节点数量增加或较少时，集群需要进行重新分片，从而处理槽位。重新分片操作是有Redis的集群管理软件Redis-trib负责执行的。其步骤如下：

1. redis-trib对目标节点发送CLUSTER SETSLOT <slot> IMPORTING <source_id> 命令，让目标节点准备好从源节点导入（import）属于槽slot的键值对
2. 对源节点发送CLUSTER SETSLOT <slot> MIGRATING <slot> <count> 命令，让源节点准备好将属于槽slot的键值对迁移(migrate)至目标节点
3. 对源节点发送CLUSTER GETKEYSINSLOT <slot> <count> 命令，获得最多count个属于槽slot的键值对的键名（key name）
4. 对步骤3获得的每个键名，都向源节点发送 MIGRATE <target_ip> <target_port> <key_name> 0 <timeoit>命令，将被选中的键原子的迁移至目标节点。
5. 重复执行步骤3和步骤4，直到源节点保存的所有槽slot都迁移完
6. redis-trib向集群中的任一几点发送CLUSTER SETSLOT<slot> NODE <target_id>命令，将槽指派给目标节点，这一指派信息会通过消息发送至整个集群。

#### ASK错误

需要注意的是，在节点进行重新分片期间，如果 Slot 2 中的数据比较多，就可能会出现一种情况：客户端向实例 2 发送请求，但此时，Slot 2 中的数据只有一部分迁移到了实例 3，还有部分数据没有迁移。在这种迁移部分完成的情况下，客户端就会收到一条 ASK 报错信息，如下所示

```c++

GET hello:key
(error) ASK 13320 172.16.19.5:6379
```

这个结果中的 ASK 命令就表示，客户端请求的键值对所在的哈希槽 13320，在 172.16.19.5 这个实例上，但是这个哈希槽正在迁移。此时，客户端需要先给 172.16.19.5 这个实例发送一个 ASKING 命令。这个命令的意思是，让这个实例允许执行客户端接下来发送的命令。然后，客户端再向这个实例发送 GET 命令，以读取数据。看起来好像有点复杂，我再借助图片来解释一下。在下图中，Slot 2 正在从实例 2 往实例 3 迁移，key1 和 key2 已经迁移过去，key3 和 key4 还在实例 2。客户端向实例 2 请求 key2 后，就会收到实例 2 返回的 ASK 命令。

ASK 命令表示两层含义：

​	第一，表明 Slot 数据还在迁移中；

​	第二，ASK 命令把客户端所请求数据的最新实例地址返回给客户端，此时，客户端需要给实例 3 发送 ASKING 命令，然后再发送操作命令。

![](image\redis\cluster迁移过程.jpg)





### **主节点选举流程**

原理分析：
当slave发现自己的master变为FAIL状态时，便尝试进行Failover，以期成为新的master。由于挂掉的master可能会有多个slave，从而存在多个slave竞争成为master节点的过程， 其过程如下：
1.slave发现自己的master变为FAIL
2.将自己记录的集群currentEpoch加1，并广播FAILOVER_AUTH_REQUEST信息
3.其他节点收到该信息，只有master响应，判断请求者的合法性，并发送FAILOVER_AUTH_ACK，对每一个epoch只发送一次ack
4.尝试failover的slave收集FAILOVER_AUTH_ACK
5.超过半数后变成新Master
6.广播Pong通知其他集群节点。

### **分布式寻址算法**

- hash 算法（大量缓存重建）
- 一致性 hash 算法（自动缓存迁移）+ 虚拟节点（自动负载均衡）
- redis cluster 的 hash slot 算法

### **优点**

- 无中心架构，支持动态扩容，对业务透明
- 具备Sentinel的监控和自动Failover(故障转移)能力
- 客户端不需要连接集群所有节点，连接集群中任何一个可用节点即可
- 高性能，客户端直连redis服务，免去了proxy代理的损耗

### **缺点**

- 运维也很复杂，数据迁移需要人工干预
- 只能使用0号数据库
- 不支持批量操作(pipeline管道操作)
- 分布式逻辑和存储模块耦合等

# Redis事务

## Redis事务功能是通过MULTI、EXEC、DISCARD和WATCH

Redis会将一个事务中的所有命令序列化，然后按顺序执行。即事务开始、事务入队、事务执行。

**1）**redis 不支持回滚，“Redis 在事务失败时不进行回滚，而是继续执行余下的命令”， 所以 Redis 的内部可以保持简单且快速。

**2）**如果在一个事务中的命令出现错误，那么所有的命令都不会执行；

**.3）**如果在一个事务中出现运行错误，那么正确的命令会被执行。

- WATCH 命令是一个乐观锁，可以为 Redis 事务提供 check-and-set （CAS）行为。可以监控一个或多个键，一旦其中有一个键被修改（或删除），之后的事务就不会执行，监控一直持续到EXEC命令。
- MULTI命令用于开启一个事务，它总是返回OK。MULTI执行之后，客户端可以继续向服务器发送任意多条命令，这些命令不会立即被执行，而是被放到一个队列中，当EXEC命令被调用时，所有队列中的命令才会被执行。
- EXEC：执行所有事务块内的命令。返回事务块内所有命令的返回值，按命令执行的先后顺序排列。当操作被打断时，返回空值 nil 。
- 通过调用DISCARD，客户端可以清空事务队列，并放弃执行事务， 并且客户端会从事务状态中退出。
- UNWATCH命令可以取消watch对所有key的监控

Redis 是单进程程序，并且它保证在执行事务时，不会对事务进行中断，事务可以运行直到执行完所有事务队列中的命令为止。因此，Redis 的事务是总是带有隔离性的。

## Redis事务其他实现

- 基于Lua脚本，Redis可以保证脚本内的命令一次性、按顺序地执行，其同时也不提供事务运行错误的回滚，执行过程中如果部分命令运行错误，剩下的命令还是会继续运行完
- 基于中间标记变量，通过另外的标记变量来标识事务是否执行完成，读取数据时先读取该标记变量判断是否事务执行完成。但这样会需要额外写代码实现，比较繁琐。



# **Redisson实现Redis分布式锁的底层原理**

![](image\redis分布式锁.png)



咱们来看上面那张图，现在某个客户端要加锁。如果该客户端面对的是一个redis cluster集群，他首先会根据hash节点选择一台机器。

紧接着，就会发送一段lua脚本到redis上，那段lua脚本如下所示：



# **Redis6.0多线程的实现机制**

**Redis6.0之前的版本真的是单线程吗？**

​		Redis在处理客户端的请求时，包括获取 (socket 读)、解析、执行、内容返回 (socket 写) 等都由一个顺序串行的主线程处理，这就是所谓的“单线程”。但如果严格来讲从Redis4.0之后并不是单线程，除了主线程外，它也有后台线程在处理一些较为缓慢的操作，例如清理脏数据、无用连接的释放、大 key 的删除等等。

从Redis自身角度来说，因为读写网络的read/write系统调用占用了Redis执行期间大部分CPU时间，瓶颈主要在于网络的 IO 消耗, 优化主要有两个方向:

• 提高网络 IO 性能，典型的实现比如使用 DPDK 来替代内核网络栈的方式
• 使用多线程充分利用多核，典型的实现比如 Memcached。

**Redis6.0默认是否开启了多线程？**

Redis6.0的多线程默认是禁用的，只使用主线程。如需开启需要修改redis.conf配置文件：**io-threads-do-reads yes**

开启多线程后，还需要设置线程数，否则是不生效的。同样修改redis.conf配置文件

## **Redis6.0多线程的实现机制**

![](image\redis\redis多线程实现机制.png)

**流程简述如下**：

1、主线程负责接收建立连接请求，获取 socket 放入全局等待读处理队列
2、主线程处理完读事件之后，通过 RR(Round Robin) 将这些连接分配给这些 IO 线程
3、主线程阻塞等待 IO 线程读取 socket 完毕
4、主线程通过单线程的方式执行请求命令，请求数据读取并解析完成，但并不执行
5、主线程阻塞等待 IO 线程将数据回写 socket 完毕
6、解除绑定，清空等待队列

![](image\redis\redis线程执行流程.png)

# redis穿透、击穿、雪崩如何应对

## 击穿

​		某个热点key,在并发情况下正好缓存过期,导致大量请求没有命中缓存，接着查询数据库，虽然数据库是有数据的，但是大量请求情况下也可能出现压垮数据库。

解决思路：

​	1.根据情况可以设置热点key永不过期,配合定时任务更新cache,或者数据有更新再主动更新。

​	2.结合互斥锁处理，先去请求一遍redis，然后再去请求DB(这样就能解决DB的穿透了)

​			业界比较常用的做法，是使用mutex。简单地来说，就是在缓存失效的时候（判断拿出来的值为空），不是立即去load db，而是先使用缓存工具的某些带成功操作返回值的操作（比如Redis的SETNX或者Memcache的ADD）去set一个mutex key，当操作返回成功时，再进行load db的操作并回设缓存；否则，就重试整个get缓存的方法。

## 缓存穿透

​		key对应的数据在缓存和数据源都不存在，导致每次请求都会出现没有命中缓存,接着查询数据源,从而有可能压垮数据源。比如用一个不存在的用户id获取用户信息，在并发情况下就可能压垮数据库。

解决思路：

​	1.用同一个不存在用户id去查询这种情况属于恶意请求，可以在nginx层根据ip做拦截。

​	2.对用户的请求参数加强过滤，比如id<1就return false。

​	3.缓存和数据都查不到时也可以给这个key设置一个null值,设置一个短一点的过期时间,比如30秒,也可以减少请求打到db上。

​	4.也可以结合布隆过滤器(**Bloom Filter**)对key做查询,如果返回没有 就一定没有，如果返回有 表示有可能有。



## 缓存雪崩

​		同一时间缓存大面试过期，会导致数据库在同一时间收到大量请求，可能出现压垮数据库。

解决思路：

​		1.分散过期时间，在设置缓存过期时间时加上一个随机的数字。

​		2.或者设置缓存永不过期，数据有更新再主动更新。