[toc]

# Redis笔记

> redis版本：6.0.16
>
> 主要根据B站上的相关课程[^ref]进行组织，部分底层实现自己阅读了源码。

## Redis 介绍

- Redis 是一个开源的 key-value 存储系统。
- 和 Memcached 类似，它支持存储的 value 类型相对更多，包括 string (字符串)、list (链表)、set (集合)、zset (sorted set –有序集合) 和 hash（哈希类型）。
- 这些数据类型都支持 push/pop、add/remove 及取交集并集和差集及更丰富的操作，而且这些操作都是原子性的。
- 在此基础上，Redis 支持各种不同方式的排序。
- 与 memcached 一样，为了保证效率，数据都是缓存在内存中。
- 区别的是 Redis 会周期性的把更新的数据写入磁盘或者把修改操作写入追加的记录文件。
- 并且在此基础上实现了 master-slave (主从) 同步。

## 应用场景

- 高频次，热门访问的数据，降低数据库 IO。

- 分布式架构，做 session 共享。

- NoSQL适用场景：
    - 需要高并发
    - 海量数据读写
    - 对数据有高扩展性需求
- 不适合使用NoSQL的场景：
    - 需要事务支持
    - 需要结构化查询存储

 

## 相关技术

Redis使用单线程（命令执行器是单线程的）+IO多路复用（网络模块）。因为Redis命令的执行引擎是单线程的，因此每个Redis自带的命令操作本身都是原子性的。

IO多路复用，IO就是网络IO（socket）连接，多路复用指的是使用有限（较少）数量的线程/进程去管理这些连接，复用的是线程/进程。

- select
- poll
- epoll（Redis采用epoll）

## 安装

```bash
cp redis.conf /etc/redis.conf
vim /etc/redis.conf
```

修改内容如下：

```bash
daemonize yes # 后台运行

requirepass yourpassword # 配置密码
```

添加到系统服务：

```bash
vim /usr/lib/systemd/system/redis.service 
```

```bash
Description=Redis Service
Documentation=https://redis.io
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/local/bin/redis-server /etc/redis.conf
ExecStop=/usr/local/bin/redis-cli shutdown
Restart=always
StandOutput=syslog

StandError=inherit

[Install]
WantedBy=multi-user.target
```

## Redis 操作及数据类型

[官网：Redis data types](https://redis.io/docs/manual/data-types/)

### Redis 键操作（Key）

- `keys *`                          查看当前库所有key
- `exist key_name`           判断key是否存在
- `type key`                      查看key类型
- `del key`                        删除指定的key数据
- `unlink key `                  根据value选择非阻塞删除
- `expire key time `         为key设置过期时间，单位秒
- `ttl key`                        查看key还有多少秒过期，-1表示永不过期，-2表示已过期
- `select`                          切换数据库
- `dbsize`                         查看当前数据库key的数量
- `flushdb `                       清空当前库
- `flushall`                     清空所有库

### Redis 字符串（String）

- 简介

    - String是Redis最基本的数据类型，一个key对应一个value。
    - String类型是二进制安全的。意味着 Redis 的 string 可以包含任何数据。比如 jpg 图片或者序列化的对象。
    - 一个Redis中的字符串value最多可以是512M。

- 常用命令

    - `set K V [option1] [NX|XX]`         添加键值对 `[option1]` = `[EX seconds|PX milliseconds|KEEPTTL]`
    - `get K`                                              获取K的对应的value
    - `append K V`                                     原来K的value后追加
    - `strlen K`                                         获取K对应V的长度
    - `setnx K V`                                       只有K不存在时，才去设置值
    - `incr K`                                             将K中存储的数字值增加1，只能对数字值操作，如果为空，新增值为1。
    - `decr K`                                             将K中存储的数字值减1，只能对数字值操作，如果为空，新增值为-1。
    - `incrby K step`                                将K中存储的数字值增加step，只能对数字值操作，如果为空，新增值为step。
    - `decrby K step`                                将K中存储的数字值减step，只能对数字值操作，如果为空，新增值为-step。
    - `mset K1 V1 K2 V2 ...`                   可以同时设置一个或多个KV对
    - `mget K1 K2 K3 ...`                        可以同时获取一个或多个value
    - `msetnx K1 V1 K2 V2 ...`               可以同时设置一个或多个KV对，当且仅当所有给定key都不存在。
    - `getrange K start end`                   获取[start , end]范围的值，左闭右闭。
    - `setrange K start V`                      从start开始覆盖原来的值。
    - `setex K time V`                             设置KV并带过期时间，time单位秒
    - `getset K V`                                     设置K的值，并且会返回旧值。

- 底层数据结构[^1]

    string的底层实现可以是int、raw、embstr。int 编码是用来保存整数值，raw编码是用来保存长字符串，而embstr是用来保存短字符串。创建源码位于`object.c`

    - int，当数据值为数字且长度小于20时，就会采用int格式进行存储，此时不需要用到sds结构体，直接将redisObject中ptr字段的值改为该数字(**注:当数值为0-10000时,key会指向预先创建好的redisObject，而不需要新建redisObject**)。`object.c::439行`

    - raw，长度大于44的使用raw编码格式。

    - embstr，代表 embstr 格式的 SDS(Simple Dynamic String 简单动态字符串)，长度小于44的字符串或者介于20到44之间的数字使用这种格式存储。**该种编码格式是将sds字符串与其对应的redisObject对象分配在同一块连续的内存空间，彷佛sds字符串嵌入在redisObject中一样，在内存上是连续的**。对应的源码位于`object.c::createEmbeddedStringObject::84行`。

        简单动态字符串 (Simple Dynamic String, 缩写 SDS)，是可以修改的字符串，内部结构实现上类似于 Java 的 ArrayList，采用预分配冗余空间的方式来减少内存的频繁分配.

        ![](https://haiqiang-picture.oss-cn-beijing.aliyuncs.com/blog/redis01.png)

        如图中所示，内部为当前字符串实际分配的空间 capacity 一般要高于实际字符串长度 len。当字符串长度小于 1M 时，扩容都是加倍现有的空间，如果超过 1M，扩容时一次只会多扩 1M 的空间。需要注意的是字符串最大长度为 512M。

    

    raw格式和embstr格式的区别如下图，即embstr是“嵌入”redisObject中的，在内存上连续，而raw格式下不是连续的。

    ![https://blog.csdn.net/WuLex/article/details/112528351](https://haiqiang-picture.oss-cn-beijing.aliyuncs.com/blog/redis02.png)

    > 图片来源：https://blog.csdn.net/WuLex/article/details/112528351

### Redis 列表（List）

- 简介

    单键多值：Redis 列表是简单的字符串列表，按照插入顺序排序。你可以添加一个元素到列表的头部（左边）或者尾部（右边）。它的底层实际是个双向链表，对两端的操作性能很高，通过索引下标的操作中间的节点性能会较差。


    ![image-20220518163018649](../../../Documents/tmp/redis03.png)

- 常用命令

    - `lpush/rpush K V1 V2 ...`                   从左边/右边插入一个或多个值。
    - `lpop/rpop K`                                         从左边/右边pop出一个值。当所有值都被pop后，对应的K也会被删除。
    - `rpoplpush K1 K2`                                  从K1列表的右边pop一个值，插入到K2列表左边
    - `lrange K start stop`                          按照索引下标获取[start, stop]之间的元素。stop为-1代表取所有值
    - `llen`                                                      获取列表长度
    - `linsert K before|after V NewV `       在V的前面/后面插入NewV
    - `lrem K n V`                                          从左边删除n个V
    - `lset K index V`                                   将K中下标为index的值替换为V

- 底层数据结构[^2]

    redis list数据结构底层采用quicklist来存储，quicklist本质上是ziplist+linkedlist的组合。旧版本的Redis中List的底层是ziplist+linkedlist，数据量少（128个）且单个数据不是很大（64字节）时使用ziplist，否则使用linkedlist存储。

    ziplist使用的是一块连续的内存，它并不是linkedlist，即两个节点之间不需要使用指针连接，所以节省了空间。

    ziplist整体结构如下：
    

![在这里插入图片描述](https://haiqiang-picture.oss-cn-beijing.aliyuncs.com/blog/redis04.png)
    
每个节点由以下信息构成
    
![在这里插入图片描述](https://haiqiang-picture.oss-cn-beijing.aliyuncs.com/blog/redis05.png)
    
- `previous_entry_length`：记录了前一个节点占用的字节数
    - `encoding`：记录了 节点的 content属性 所保存数据的类型以及长度
- `content`：负责记录节点的内容。值的类型和长度 由 节点的`encoding`属性决定
  

`zlentry`源码，位于`ziplist.c::282行`：
    
    ```c
    typedef struct zlentry {
    unsigned int prevrawlensize; /* Bytes used to encode the previous entry len*/
        unsigned int prevrawlen;     /* Previous entry len. */
    unsigned int lensize;        /* Bytes used to encode this entry type/len.
                                        For example strings have a 1, 2 or 5 bytes
                                        header. Integers always use a single byte.*/
        unsigned int len;            /* Bytes used to represent the actual entry.
                                        For strings this is just the string length
                                        while for integers it is 1, 2, 3, 4, 8 or
                                        0 (for 4 bit immediate) depending on the
                                        number range. */
        unsigned int headersize;     /* prevrawlensize + lensize. */
        unsigned char encoding;      /* Set to ZIP_STR_* or ZIP_INT_* depending on
                                        the entry encoding. However for 4 bits
                                        immediate integers this can assume a range
                                        of values and must be range-checked. */
        unsigned char *p;            /* Pointer to the very start of the entry, that
                                        is, this points to prev-entry-len field. */
    } zlentry;
    ```
    
    quicklist结构如下：
    
    ![image-20220518164459005](https://haiqiang-picture.oss-cn-beijing.aliyuncs.com/blog/redis06.png)

Redis 将链表和 ziplist 结合起来组成了 quicklist。也就是将多个 ziplist 使用双向指针串起来使用。这样既满足了快速的插入删除性能，又不会出现太大的空间冗余。


### Redis 集合（Set）

- 简介

    - Redis set 对外提供的功能与 list 类似，是一个列表的功能，特殊之处在于 set 是可以自动排重的，当你需要存储一个列表数据，又不希望出现重复数据时，set 是一个很好的选择，并且 set 提供了判断某个成员是否在一个 set 集合内的重要接口，这个也是 list 所不能提供的。
    - Redis 的 Set 是 string 类型的无序集合。它底层其实是一个 value 为 null 的 hash 表，所以添加、删除、查找的 **复杂度都是 O (1)**。
    - 一个算法，随着数据的增加，执行时间的长短，如果是 O (1)，数据增加，查找数据的时间不变。

- 常用命令

    - `sadd K V1 V2`                              将一个或多个元素加入到集合中，已经存在的元素将被忽略
    - `smembers K`                                  取出该集合的所有值
    - `sismember K V`                             判断集合K中是否含有V，有则返回1，否则返回0
    - `scard K`                                        返回集合的元素个数
    - `srem K V1 V2`                               删除集合里指定元素
    - `spop K`                                          随机pop一个值
    - `srandmember K n`                         随机取出n个值，不会从集合中删除
    - `smove source dest V`                  把集合中一个值移动到另一个集合中
    - `sinter K1 K2`                               取交集
    - `sunion K1 K2`                               取并集
    - `sdiff K1 K2`                                 取差集，K1中有而K2中不存在的

- 底层数据结构

    当value是整数且数据量不大（小于512）时，使用intset存储，其他情况用字典dict存储，底层是哈希表。

    intset本质上就是一个数组，并且存储数据是有序的，因此查找时可以通过二分查找来实现。

    redis中intset的定义，源码位于`intset.h`：

    ```c
    typedef struct intset {
        uint32_t encoding;
        uint32_t length;
        int8_t contents[];
    } intset;
    
    intset *intsetNew(void);
    intset *intsetAdd(intset *is, int64_t value, uint8_t *success);
    intset *intsetRemove(intset *is, int64_t value, int *success);
    uint8_t intsetFind(intset *is, int64_t value);
    int64_t intsetRandom(intset *is);
    uint8_t intsetGet(intset *is, uint32_t pos, int64_t *value);
    uint32_t intsetLen(const intset *is);
    size_t intsetBlobLen(intset *is);
    ```

    插入过程（如果不升级为字典且插入值不重复）就是一次直接插入排序的过程，会先找到插入位置，然后将插入位置的数据依次移动到数组尾部。源码位于`intset.c::intsetAdd::206行`

### Redis 哈希（Hash）

- 简介

    - Redis hash 是一个键值对集合。
    - Redis hash 是一个 string 类型的 field 和 value 的映射表，hash 特别适合用于存储对象。

- 常用命令

    - `hset K field value` 给K集合中field键赋值value
    - `hget K field` 从K集合中取出field对应的值value
    - `hmset K field1 value1 field2 value2 ...` 批量赋值
    - `hexists K field` 查询K中是否有field键值对
    - `hkeys K` 列出该K下的所有field
    - `hvals K` 列出该K下所有的value
    - `hincrby K field increment` 为K中field的值加上增量increment
    - `hsetnx K field value` 只有field不存在时才设置值。

- 底层数据结构

    Hash对应ziplist和hashtable。

    满足下面两个条件时，使用ziplist：

    - 保存的所有键值对的键和值的字符串长度都小于 64 字节；
    - 保存的键值对数量小于512个。

    使用ziplist存储时，一个kv对占用两个ziplist节点，第一个节点存key，第二个节点存value。数据较少时，使用ziplist存储既能节省空间，又能保持不错的性能。

    redis里hashtable是redis自己用c实现的，源码位于`dict.h`和`dict.c`中

    ```c
    typedef struct dictEntry {
        void *key;
      	// 内部value是一个共用体union
        union {
            void *val;
            uint64_t u64;
            int64_t s64;
            double d;
        } v;
        struct dictEntry *next;
    } dictEntry;
    
    typedef struct dictType {
        uint64_t (*hashFunction)(const void *key);
        void *(*keyDup)(void *privdata, const void *key);
        void *(*valDup)(void *privdata, const void *obj);
        int (*keyCompare)(void *privdata, const void *key1, const void *key2);
        void (*keyDestructor)(void *privdata, void *key);
        void (*valDestructor)(void *privdata, void *obj);
    } dictType;
    
    /* This is our hash table structure. Every dictionary has two of this as we
     * implement incremental rehashing, for the old to the new table. */
    typedef struct dictht {
        dictEntry **table;
        unsigned long size;
        // 用于计算索引值的，总是等于size-1
        // 源码中计算索引的方式  idx = h & d->ht[table].sizemask; 
        unsigned long sizemask; 
        unsigned long used;
    } dictht;
    
    
    typedef struct dict {
        dictType *type;
        void *privdata;
        // 定义了两个dictht，其中ht[1]用于动态扩容
        dictht ht[2];
        long rehashidx; /* rehashing not in progress if rehashidx == -1 */
        unsigned long iterators; /* number of iterators currently running */
    } dict;
    ```

    hashtable结构示意图如下，使用的是拉链法来解决冲突：

    ![image-20220508200026820](https://haiqiang-picture.oss-cn-beijing.aliyuncs.com/blog/redis07.png)

    此外hashtable还有以下内容

    - `dictEntry** table`的大小一定是2的整数次幂，从源码中计算索引值的代码就可以得出`idx = h & d->ht[table].sizemask; `，其中sizemask=size-1，只有哈希表的总长度len为2的n次幂时，才能用`hash & (len - 1)`这样来计算索引值。这是因为，当len为2的n次幂时，它的二进制一定是`0000...x...0`，而len-1的二进制一定是`0000...0111...1`(从x下一位开始全为1)，这样`hash & (len - 1)`就等价于`hash % len`了，而我们知道位运算速度是很快的，而计算索引在hash结构中又是调用非常频繁的api，因此hash桶（`dictEntry** table`）的长度一定是2的整数次幂。源码中注释也说明了这点。

        ```c
        /* Our hash table capability is a power of two */
        static unsigned long _dictNextPower(unsigned long size)
        ```

    - `typedef struct dict`结构体中有个`dictht ht[2]`数组，`ht[1]`是用来扩容使用的，当哈希表的负载因子（load factor）达到一定的数值后，就需要扩容或缩小，需要rehash，这个时候就需要`ht[1]`来辅助rehash过程。

        - 如果是扩容，就为`ht[1]`分配一个大小为大于`ht[0].used*2`且为2的n次幂的大小的`dictEntry`数组。

            扩容源码位置：`dict.c::969行`

            ```c
            return dictExpand(d, d->ht[0].used*2);
            ```

            源码位置：`dict.c::975行`

            ```c
            /* Our hash table capability is a power of two */
            static unsigned long _dictNextPower(unsigned long size)
            {
                unsigned long i = DICT_HT_INITIAL_SIZE;
            
                if (size >= LONG_MAX) return LONG_MAX + 1LU;
                while(1) {
                    if (i >= size)
                        return i;
                    i *= 2;
                }
            }
            ```

        - 如果是收缩操作，那么就为`ht[1]`分配一个大小为大于`ht[0].used`且为2的n次幂且不小于`DICT_HT_INITIAL_SIZE`（4）且数值最小的大小的`dictEntry`数组。可能比较绕，其实就是找到一个最接近`ht[0].used`且比`ht[0].used`大的数。

            收缩源码位置：`t_hash.c::306行`

            ```c
            if (dictDelete((dict*)o->ptr, field) == C_OK) {
                deleted = 1;
            
                /* Always check if the dictionary needs a resize after a delete. */
                if (htNeedsResize(o->ptr)) dictResize(o->ptr);
            }
            ```

            `dictResize`源码位置：`dict.c::135行`

            ```c
            int dictResize(dict *d)
            {
                unsigned long minimal;
            
                if (!dict_can_resize || dictIsRehashing(d)) return DICT_ERR;
                minimal = d->ht[0].used;
                if (minimal < DICT_HT_INITIAL_SIZE)
                    minimal = DICT_HT_INITIAL_SIZE;
                return dictExpand(d, minimal);
            }
            ```

### Redis 有序集合（ZSet，Sorted Set）

- 简介

    - Redis 有序集合 zset 与普通集合 set 非常相似，是一个没有重复元素的字符串集合。
    - 不同之处是有序集合的每个成员都关联了一个评分（score），这个评分（score）被用来按照从最低分到最高分的方式排序集合中的成员。集合的成员是唯一的，但是评分可以是重复了 。
    - 因为元素是有序的，所以你也可以很快的根据评分（score）或者次序（position）来获取一个范围的元素。
    - 访问有序集合的中间元素也是非常快的，因此你能够使用有序集合作为一个没有重复成员的智能列表。

- 常用命令

    - `zadd K score1 v1 score2 v2 ...` 将一个或者多个mamber元素及其score值加入到K集合中
    - `zrange K start stop [withscores]`返回ZSet中下标在[start, end]之间的元素，如果带`withscores`则会同时返回分数。
    - `zrangebyscore K min max [withscores] [limit offset count]` 返回ZSet的K中，所有分数介于[min, max]的元素，从小到大。
    - `zrevrangebyscore K max min [withscores] [limit offset count]` 同上，结果从大到小排序。
    - `zincrby K increment member` 给member的score加上increment
    - `zrem K member [member ...]` 删除指定K下的member
    - `zcount K min max` 统计集合中，位于[min, max]之间的元素的个数
    - `zrank K member` 返回该值在集合中的排名，从0开始。

- 底层数据结构

    如果满足下面的条件，则用ziplist来存储：

    - 元素数量小于128个
    - 所有member的长度都小于64字节。

    如果不满足，则使用hashtable+skiplist来存储。

    源码位置：`server.h::913行`

    ```c
    /* ZSETs use a specialized version of Skiplists */
    typedef struct zskiplistNode {
        sds ele;  // member
        double score;  // member对应的分数，允许相同
        struct zskiplistNode *backward;  // 指向前驱节点
        struct zskiplistLevel { 
            struct zskiplistNode *forward;  // 指向节点当层的下一个节点
            unsigned long span;  // 从当前节点到下一个节点的跨度
        } level[];
    } zskiplistNode;
    
    typedef struct zskiplist {
        struct zskiplistNode *header, *tail;
        // 元素个数
        unsigned long length;
        // 头节点的层数，永远和跳表中最大的层数相等
        int level;
    } zskiplist;
    
    typedef struct zset {
        dict *dict;
        zskiplist *zsl;
    } zset;
    ```

    从`typedef struct zset`的源码就可以看出ZSet使用一个字典`dict`和一个`zskiplist`的数据结构来实现。字典的实现上面讨论了，这里不再赘述。字典`dict`用于保存`member`到`score`的映射，这样就可以在O(1)的时间复杂度下获取`member`对应的`score`，此外字典还可以完成去重的功能。

    源码位置：`t_zset.c::1419行`

    ```c
    dictAdd(zs->dict,ele,&znode->score)
    ```

    可以看到，字典里添加的键值对是`<member:score>`

    ZSet初始化节点层数的代码（`t_zset.c::122行`）：

    ```c
    int zslRandomLevel(void) {
        int level = 1;
        while ((random()&0xFFFF) < (ZSKIPLIST_P * 0xFFFF))  // ZSKIPLIST_P的值是0.25
            level += 1;
        return (level<ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;  // ZSKIPLIST_MAXLEVEL 的值是32，最多32层
    }
    ```

    可以看到有`1/4`的概率生成新的一层。

    ```c
    // 位于t_zset.c 第132行
    zskiplistNode *zslInsert(zskiplist *zsl, double score, sds ele) {
        zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
        unsigned int rank[ZSKIPLIST_MAXLEVEL];
        int i, level;
    
        serverAssert(!isnan(score));
        x = zsl->header;
        // 从最高层往下找合适的插入位置
        for (i = zsl->level-1; i >= 0; i--) {
            /* store rank that is crossed to reach the insert position */
            // rank用于计算排名
            rank[i] = i == (zsl->level-1) ? 0 : rank[i+1];
            // 根据这里的判断可以得出，插入ZSet时，如果两个元素的score一样，
            // 则按照sdscmp()函数的结果来排序，底层调用memcmp()，完成字典序排序
            while (x->level[i].forward &&
                    (x->level[i].forward->score < score ||
                        (x->level[i].forward->score == score &&
                        sdscmp(x->level[i].forward->ele,ele) < 0)))
            {
                rank[i] += x->level[i].span; // 当前排名需要加上跳过的节点的个数，被跳过的节点排名一定小于当前节点
                x = x->level[i].forward; // 跳到当前层指向的下一个节点继续找
            }
            update[i] = x;  // 至此，找到了当前层需要更新的位置
        }
        /* we assume the element is not already inside, since we allow duplicated
         * scores, reinserting the same element should never happen since the
         * caller of zslInsert() should test in the hash table if the element is
         * already inside or not. */
        level = zslRandomLevel();
        // 如果新的节点的层数大于头节点的层数，需要更新头节点
        if (level > zsl->level) {
            // 为新节点更新信息
            for (i = zsl->level; i < level; i++) {
                // 新节点高于原来总层数的所有层的rank都是0
                rank[i] = 0;
                update[i] = zsl->header;
                // 跳过的个数就是所有
                update[i]->level[i].span = zsl->length;
            }
            // 更新总的层数
            zsl->level = level;
        }
        x = zslCreateNode(level,score,ele);
        for (i = 0; i < level; i++) {
            x->level[i].forward = update[i]->level[i].forward;
            update[i]->level[i].forward = x;
    
            /* update span covered by update[i] as x is inserted here */
            x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]);
            update[i]->level[i].span = (rank[0] - rank[i]) + 1;
        }
    
        /* increment span for untouched levels */
        for (i = level; i < zsl->level; i++) {
            update[i]->level[i].span++;
        }
    
        x->backward = (update[0] == zsl->header) ? NULL : update[0];
        if (x->level[0].forward)
            x->level[0].forward->backward = x;
        else
            zsl->tail = x;
        zsl->length++;
        return x;
    }
    
    // 位于 sds.c 799行
    int sdscmp(const sds s1, const sds s2) {
        size_t l1, l2, minlen;
        int cmp;
    
        l1 = sdslen(s1);
        l2 = sdslen(s2);
        minlen = (l1 < l2) ? l1 : l2;
        cmp = memcmp(s1,s2,minlen);
        if (cmp == 0) return l1>l2? 1: (l1<l2? -1: 0);
        return cmp;
    }
    ```

    关于跳表的实现网上博客比价多，就不多比比了。简单来说跳表就是一种有序表的实现，类似的实现还有`SizeBalancedTree`、`红黑树`、`AVL`。这几种数据结构都能做到O(logn)时间复杂度的增删查改。跳表在插入的常数时间上是要优于其他几种结构的，因为它不需要复杂的调整过程，另外跳表相对其它几种结构更简单，更容易去维护。

### Bitmaps

- 简介

    Redis提供bitmap数据类型。bitmap可以理解为一个bit类型的数组，每个数组元素都有两个值：0和1。可以表示两种状态。底层是使用String实现的，String的最大值是512M，因此bitmap最大的下标值是2^32。但是实际应用中，可以用多个bitmap拼起来来实现一个更大的bitmap。

- 常用命令

    - `setbit K offset V`             给K代表的Bitmaps偏移量为offset处的bit设置值V，V只能是0或者1。offset可以理解为bit数组的下标。 
    - `getbit K offset`                 获取K代表的Bitmaps偏移量为offset处的bit值。
    - `bitcount K [start end]`    计算[start end]之间1的bit的个数。
    - `bitop operation destkey key [key ...]`对一个或多个key进行位操作，并将结果保存到destkey中。

- 底层数据结构

    底层是String，字符串类型。其实不管底层是啥数据，最终都是按2进制位来存储的，所以理论上任何数据结构多可以拿来实现Bitmaps。

### HyperLogLog

- 简介

    基数计数(cardinality counting)，通常用来统计一个集合中不重复的元素个数。

    什么是基数？

    比如数据集 {1, 3, 5, 7, 5, 7, 8}，那么这个数据集的基数集为 {1, 3, 5 ,7, 8}，基数 (不重复元素) 为 5。 基数估计就是在误差可接受的范围内，快速计算基数。

    在工作当中，我们经常会遇到与统计相关的功能需求，比如统计网站 PV（PageView 页面访问量），可以使用 Redis 的 incr、incrby 轻松实现。但像 UV（UniqueVisitor 独立访客）、独立 IP 数、搜索记录数等需要去重和计数的问题如何解决？这种求集合中不重复元素个数的问题称为基数问题。

    解决基数问题有很多种方案：

    - 数据存储在 MySQL 表中，使用 distinct count 计算不重复个数。
    - 使用 Redis 提供的 hash、set、bitmaps 等数据结构来处理。

    以上的方案结果精确，但随着数据不断增加，导致占用空间越来越大，对于非常大的数据集是不切实际的。能否能够降低一定的精度来平衡存储空间？Redis 推出了 HyperLogLog。HyperLogLog其实是一种基数计数概率算法，并不是Redis特有的。

    HyperLogLog 的优点是，在输入元素的数量或者体积非常非常大时，计算基数所需的空间总是固定 的、并且是很小的。在 Redis 里面，每个 HyperLogLog 键只需要花费 12 KB 内存，就可以计算接近 2^64 个不同元素的基 数。这和计算基数时，元素越多耗费内存就越多的集合形成鲜明对比。但是，因为 HyperLogLog 只会根据输入元素来计算基数，而不会储存输入元素本身，所以 HyperLogLog 不能像集合那样，返回输入的各个元素。[^3][^4]

- 常用命令

    - `pfadd key element [element ...]` 添加元素到HyperLogLog中。
    - `pfcount key [key ...]`                   计算HyperLogLog的近似基数。
    - `pfmerge destkey sourcekey [sourcekey ...]` 将一个或多个HyperLogLog合并后的结构存储到另一个HyperLogLog中。

- 底层数据结构

    [Redis-HyperLogLog内部实现原理源码解析](https://blog.csdn.net/zanpengfei/article/details/86519324)

### Geospatial

Redis 3.2 中增加了对 GEO 类型的支持。GEO，Geographic，地理信息的缩写。该类型，就是元素的 2 维坐标，在地图上就是经纬度。redis 基于该类型，提供了经纬度设置，查询，范围查询，距离查询，经纬度 Hash 等常见操作。

## Redis配置文件

### Units

- **配置数据单位换算关系**

    ```bash
    ##################  该部分用于指定存储单位的大小换算关系，不区分大小写，只支持bytes，不支持bits
    # 1k => 1000 bytes
    # 1kb => 1024 bytes
    # 1m => 1000000 bytes
    # 1mb => 1024*1024 bytes
    # 1g => 1000000000 bytes
    # 1gb => 1024*1024*1024 bytes
    #
    # units are case insensitive so 1GB 1Gb 1gB are all the same.
    ```

- **包含其它配置文件的信息  include path**

    对于公共部分配置，可以按以下方式配置引入

    ```bash
    # include /path/to/local.conf
    # include /path/to/other.conf
    ```

### Network 网络相关

- **bind IP1 [IP2 ...]**

    这项配置绑定的IP并不是远程访问的客户端的IP地址，而是本机的IP地址。

    ```bash
    # bind 192.168.1.100 10.0.0.1
    # bind 127.0.0.1 ::1
    bind 127.0.0.1
    ```

    本机的IP地址是和网卡（network interfaces）绑定在一起的，配置这项后，Redis只会接收来自指定网卡的数据包。比如我的主机有以下网卡：

    ```bash
    root@VM-4-5-ubuntu:~# ifconfig 
    eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
            inet 10.0.4.5  netmask 255.255.252.0  broadcast 10.0.7.255
            inet6 fe80::5054:ff:fe0b:843  prefixlen 64  scopeid 0x20<link>
            ether 52:54:00:0b:08:43  txqueuelen 1000  (Ethernet)
            RX packets 283943  bytes 28027507 (28.0 MB)
            RX errors 0  dropped 0  overruns 0  frame 0
            TX packets 280878  bytes 43033240 (43.0 MB)
            TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
    
    lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
            inet 127.0.0.1  netmask 255.0.0.0
            inet6 ::1  prefixlen 128  scopeid 0x10<host>
            loop  txqueuelen 1000  (Local Loopback)
            RX packets 35168  bytes 2582220 (2.5 MB)
            RX errors 0  dropped 0  overruns 0  frame 0
            TX packets 35168  bytes 2582220 (2.5 MB)
            TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
    ```

    如果我想要让Redis可以远程连接的话，就需要让Redis监听`eht0`这块网卡，也就是要加上配置`bind 127.0.0.1 10.0.4.5`，这样既可以本地访问，也能够远程访问。（主要`bind`只能有一行配置，如果有多个网卡要监听，就配置多个`ip`，用空格隔开，否者只有配置的最后一个`bind`生效）。

- **保护模式  protected-mode**

    从注释信息就可以看到，如果`protected-mode`是`yes`的话，如果没有指定`bind`或者没有指定密码，那么只能本地访问。

    ```bash
    protected-mode yes
    ```

- **端口号  port**

    配置Redis监听的端口号，默认6379。

    ```bash
    # Accept connections on the specified port, default is 6379 (IANA #815344).
    # If port 0 is specified Redis will not listen on a TCP socket.
    port 6379
    ```

- **TCP半连接队列长度配置 tcp-backlog**

    在进行TCP/IP连接时，内核会维护两个队列

    - `syns queue`用于保存已收到`sync`但没有接收到`ack`的TCP半连接请求。由`/proc/sys/net/ipv4/tcp_max_syn_backlog`指定，我的系统（Ubuntu20.04）上是1024。
    - `accept queue`，用于保存已经建立的连接，也就是全连接。由`/proc/sys/net/core/somaxconn`指定。

    根据配置里的注释，需要同时提高`somaxconn`和`tcp_max_syn_backlog`的值来确保生效。

    ```bash
    tcp-backlog 511
    ```

- **是否超时无操作关闭连接  timeout **

    客户端经过多少时间（单位秒）没有操作就关闭连接，0代表永不关闭。

    ```bash
    timeout 0
    ```

- **TCP连接保活策略  tcp-keepalive**

    TCP连接保活策略，可以通过tcp-keepalive配置项来进行设置，单位为秒，假如设置为60秒，则server端会每60秒向连接空闲的客户端发起一次ACK请求，以检查客户端是否已经挂掉，对于无响应的客户端则会关闭其连接。如果设置为0，则不会进行保活检测。

    ```bash
    tcp-keepalive 300
    ```

### GENERAL 通用配置

- **启动方式  daemonize **

    是否以守护（后台）进程的方式启动，默认no。

    ```bash
    daemonize yes 
    ```

- **进程pid文件  pidfile**

    redis启动后会把pid写入到pidfile指定的文件中。

    ```bash
    pidfile /var/run/redis_6379.pid
    ```

- **日志相关  loglevel   logfile**

    ` loglevel `用于配置日志打印机别，默认`notice`：

    - `debug`：能设置的最高的日志级别，打印所有信息，包括debug信息。
    - `verbose`：打印除了`debug`日志之外的所有日志。
    - `notice`：打印除了`debug`和`verbose`级别的所有日志。
    - `warning`：仅打印非常重要的信息。

    ```bash
    loglevel notice
    
    logfile ""
    ```

- **指定数据库的数量  databse**

    redis默认有16个数据库，编号从0开始。

    ```bash
    databases 16
    ```

- **启动是否显示logo**

    ```bash
    always-show-logo yes
    ```

### SNAPSHOTTING 快照相关

### SECURITY 安全相关

```bash
################################## SECURITY ###################################

# Warning: since Redis is pretty fast, an outside user can try up to
# 1 million passwords per second against a modern box. This means that you
# should use very strong passwords, otherwise they will be very easy to break.
# Note that because the password is really a shared secret between the client
# and the server, and should not be memorized by any human, the password
# can be easily a long string from /dev/urandom or whatever, so by using a
# long and unguessable password no brute force attack will be possible.
```

大致意思就是redis很快，所以被破解密码时，性能也很好，如果你的密码太渣渣了，那么可能很快就被破解了，因此尽量使用长且不容易被猜到的密码作为redis的访问密码。

- **配置ACL**

    `ACL`：访问控制列表。

    有两种方法配置ACL：

    - 在命令行通过ACL命令进行配置
    - 在Redis配置文件中开始，可以直接在`redis.conf`中配置，也可以通过外部`aclfile`配置。`aclfile path`。

    配置语法：`user <username> ... acl rules ...`，例如` user worker +@list +@connection ~jobs:* on >ffa9203c493aa99`

    redis默认有一个`default`用户。如果`default`具有`nopass`规则（就是说没有配置密码），那么新连接将立即作为`default`用户登录，无需通过`AUTH`命令提供任何密码。否则，连接会在未验证状态下启动，并需要`AUTH`验证才能开始工作。

    ____

    **描述用户可以做的操作的ACL规则如下：**

    

    - **启用或禁用用户（已经建立的连接不会生效）**
        - `on`        启用用户，该用户可以验证身份登陆。
        - `off`      禁用用户，该用户不允许验证身份登陆。
    - **允许/禁止用户执行某些命令**
        - `+<command>`                                允许用户执行`command`指示的命令✅
        - `-<command>`                                禁止用户执行`command`指示的命令🈲
        - `+@<category>`                            允许用户执行`category`分类中的所有命令✅
        - `-@<category>`                            禁止用户执行`category`分类中的所有命令🈲
        - `+<command>|subcommand`           允许执行某个被禁止的`command`的子命令`subcommand`。没有对应的`-`规则。✅
        - `allcommands`                              `+@all`的别名，允许执行所有命令。✅
        - `nocommands`                               `-@all`的别名，禁止执行所有命令。🈲
    - **允许/禁止用户访问某些Key**
        - `~<pattern> `                              添加用户可以访问的Key，比如`~*`代表用户可以访问所有key，`~K*`代表用户可访问以`K`开头的key。✅
        - `allkeys`                                   等价于`~*`✅
        - `resetkeys ~<pattern> `           刷新允许模式的列表，会覆盖之前设置的模式。例如 `user test ~* resetkeys ~K*`，则只允许访问匹配`K*`的键，`~*`失效。✅
    - **为用户配置有效密码**
        - `><password>`              将密码添加到用户的有效密码列表中。例如`user test >mypass on`，则用户test可以使用`mypass`验证。
        - `<<password>`              将密码从用户的有效密码列表中删除，即不可以使用该密码验证。
        - `nopass`                        使用任意密码都可以成功验证。
        - `resetpass`                  清除用户可用密码列表的数据，并清除 nopass 状态。之后该用户不能登陆。直到重新设置密码/设置`nopass`。
        - `reset`                          重置用户到初始状态。该命令会执行以下操作：`resetpass`,` resetkeys`,` off`,`-@all`。

- **ACL日志配置**

    设置ACL日志最大长度，默认128个记录。这个日志是存在内存里的。

    ```bash
    acllog-max-len 128
    ```

- **外部ACL文件配置**

    默认位置`etc/redis/users.acl`，我们可以在这个文件中定义所有用户的`ACL`控制信息。

    ```bash
    aclfile /etc/redis/users.acl
    ```

- **配置默认用户default的密码**

    该配置只对默认用户`default`生效。

    ```bash
    requirepass yourpass 
    ```

### CLIENTS 客户端配置

- **设置最大同时客户端连接数**

    设置可以同时连接客户端的最大数量。默认该项设置为 10000 个客户端。达到限制值后的连接会被拒绝并会返回错误信息。

    ```bash
    maxclients 10000
    ```

### MEMORY MANAGEMENT 内存管理

- **最大内存限制**

    指定Redis最大内存限制。达到内存限制时，Redis将尝试删除已到期或即将到期的Key。

    ```bash
    maxmemory <bytes>
    ```

- **达到最大内存限制时的策略**

    配置达到最大内存限制后，Redis进行何种操作。默认`noeviction`

    ```bash
    maxmemory-policy noeviction
    ```

    总共有8种策略可供选择。

    - `volatile-lru`            只对设置了过期时间的Key进行淘汰，淘汰算法近似的LRU。
    - `allkeys-lru`              对所有Key进行淘汰，LRU。
    - `volatile-lfu`            只对设置了过期时间的Key进行淘汰，淘汰算法近似的LFU。
    - `allkeys-lfu`              对所有Key进行淘汰，LFU。
    - `volatile-random`      只对设置了过期时间的Key进行淘汰，淘汰算法为随机淘汰。
    - `allkeys-random`        对所有Key进行淘汰，随机淘汰。
    - `volatile-ttl`            只对设置了过期时间的Key进行淘汰，删除即将过期的即ttl最小的。
    - `noeviction`                永不删除key，达到最大内存再进行数据装入时会返回错误。
    
    对于可以通过删除key来释放内存的策略，如果没有key可以删除了，那么也会报错。
    
- **使用LRU/LFU/TTL算法时采样率**

    Redis使用的是近似的LRU/LFU/minimal TTL算法。主要是为了节约内存以及提升性能。Redis配置文件有`maxmemory-samples`选项，可以配置每次取样的数量。Redis每次会选择配置数量的key，然后根据算法从中淘汰最差的key。

    ```bash
    maxmemory-samples 5
    ```

    可以通过修改这个配置来获取更高的淘汰精度或者更好的性能。默认值5就可以获得很好的结果。选择10可以非常接近真是的LRU算法，但是会耗费更多的CPU资源。3的话更快但是淘汰结果不是特别准确。

- **从库不淘汰数据**

    配置Redis主从复制时，从库超过`maxmemory`也不淘汰数据。这个配置主要是为了保证主从库的一致性，因为Redis的淘汰策略是随机的，如果允许从库自己淘汰key，那么会导致主从不一致的现象出现（master节点删除key的命令会同步给slave节点）。

    ```bash
    replica-ignore-maxmemory yes
    ```

- **过期keys驻留在内存中的比例**

    设置过期keys仍然驻留在内存中的比重，默认是为1，表示最多只能有10%的过期key驻留在内存中，该值设置的越小，那么在一个淘汰周期内，消耗的CPU资源也更多，因为需要实时删除更多的过期key。

    ```bash
    active-expire-effort 1
    ```

### LAZY FREEING 懒惰删除

```bash
# Redis has two primitives to delete keys. One is called DEL and is a blocking
# deletion of the object. It means that the server stops processing new commands
# in order to reclaim all the memory associated with an object in a synchronous
# way. If the key deleted is associated with a small object, the time needed
# in order to execute the DEL command is very small and comparable to most other
# O(1) or O(log_N) commands in Redis. However if the key is associated with an
# aggregated value containing millions of elements, the server can block for
# a long time (even seconds) in order to complete the operation.
# 
# For the above reasons Redis also offers non blocking deletion primitives
# such as UNLINK (non blocking DEL) and the ASYNC option of FLUSHALL and
# FLUSHDB commands, in order to reclaim memory in background. Those commands
# are executed in constant time. Another thread will incrementally free the
# object in the background as fast as possible.
# DEL, UNLINK and ASYNC option of FLUSHALL and FLUSHDB are user-controlled.
# It's up to the design of the application to understand when it is a good
# idea to use one or the other. However the Redis server sometimes has to
# delete keys or flush the whole database as a side effect of other operations.
# Specifically Redis deletes objects independently of a user call in the
# following scenarios:
#
# 1) On eviction, because of the maxmemory and maxmemory policy configurations,
#    in order to make room for new data, without going over the specified
#    memory limit.
# 2) Because of expire: when a key with an associated time to live (see the
#    EXPIRE command) must be deleted from memory.
# 3) Because of a side effect of a command that stores data on a key that may
#    already exist. For example the RENAME command may delete the old key
#    content when it is replaced with another one. Similarly SUNIONSTORE
#    or SORT with STORE option may delete existing keys. The SET command
#    itself removes any old content of the specified key in order to replace
#    it with the specified string.
# 4) During replication, when a replica performs a full resynchronization with
#    its master, the content of the whole database is removed in order to
#    load the RDB file just transferred.
#
# In all the above cases the default is to delete objects in a blocking way,
# like if DEL was called. However you can configure each case specifically
# in order to instead release memory in a non-blocking way like if UNLINK
# was called, using the following configuration directives.
```

翻译上面的话就是：

Redis有两个删除keys的原语。一个是`DEL`并且它是一个阻塞的删除对象的操作。意味着server会停止处理新的`command`以便以同步的方式回收与对象关联的所有内存。如果被删除的key关联的是一个小对象，那么执行`DEL`命令所需要的时间非常短，与Redis中其它`O(1)`或`O(log_N)`的命令时间开销几乎一样。然鹅，如果key与包含了数百万个元素的大对象相关联，那么服务器为了完成删除命令会阻塞很长时间（甚至几秒钟）。

出于以上原因，Redis提供了非阻塞的删除原语，例如`UNLINK`(非阻塞式的`DEL`)和`FLUSHALL`、`FLUSHDB`命令的`ASYNC`选项，以便在后台回收内存。这些命令会在常量（固定的）时间内执行。另外一个线程会在后台尽可能快的以渐进式的方式释放对象。

使用`DEL`,`UNLINK`以及`FLUSHALL`和`FLUSHDB`的`ASYNC`选项是由用户来控制的。这应该由应用程序的设计来决定使用其中的哪一个。 然鹅，作为其它操作的副作用，Redis server有时不得不去删除keys或者刷新整个数据库。具体来说，Redis在以下情况下会独立于用户调用而删除对象：

1） 由于`maxmemory` 和`maxmemory policy`的设置，为了在不超出指定的内存限制而为新对象腾出空间而逐出旧对象；

2） 因为过期：当一个key设置了过期时间且必须从内存中删除时；

3） 由于在已经存在的key上存储对象的命令的副作用。例如，`RENAME`命令可能会删除旧的key的内容，当该key的内容被其它内容代替时。类似的，`SUNIONSTORE`或者带`STORE`选项的`SORT`命令可能会删除已经存在的keys。`SET`命令会删除指定键的任何旧内容，以便使用指定字符串替换。

4）在复制过程中，当副库与主库执行完全重新同步时，整个数据库的内容将被删除，以便加载刚刚传输的RDB文件。

在上述所有情况下，默认情况是以阻塞方式删除对象，就像调用DEL一样。但是，你可以使用以下配置指令专门配置每种情况，以非阻塞的方式释放内存，就像调用`UNLINK`一样。

____

相关的配置：

```bash
# 内存达到设置的maxmemory时，是否使用惰性删除，对应上面 1）
lazyfree-lazy-eviction no
# 过期keys是否惰性删除，对应上面 2）
lazyfree-lazy-expire no
# 内部删除选项，对应上面选项 3）的情况是否惰性删除
lazyfree-lazy-server-del no
# slave接收完RDB文件后清空数据是否是惰性的，对应上面情况 4）
replica-lazy-flush no

# It is also possible, for the case when to replace the user code DEL calls
# with UNLINK calls is not easy, to modify the default behavior of the DEL
# command to act exactly like UNLINK, using the following configuration
# directive:

# 是否将DEL调用替换为UNLINK，注释里写的从user code里替换DEL调用为UNLINK调用可能并不是一件
# 容易的事，因此可以使用以下选项，将DEL的行为替换为UNLINK
lazyfree-lazy-user-del no
```

### THREADED I/O

Redis大体上是单线程的，但是也有一些场景使用额外的线程去做的，比如`UNLINK`、`slow I/O accesses`。

**现在还可以在不同的I/O线程中处理Redis客户端socket读写**。（只是网络IO这块儿成了多线程，执行命令的那个家伙，还是单线程！）特别是因为写操作很慢，通常Redis的用户使用pipeline来提升每个核心下的Redis性能，并且运行多个Redis实例来实现扩展。使用多线程I/O，不需要使用pipeline和实例切分，就可以轻松的提升两倍的性能。

默认情况下，多线程是禁用的，我们建议只在至少有4个或更多内核的机器中启用多线程，至少保留一个备用内核。使用超过8个线程不太可能有多大帮助。我们还建议仅当您确实存在性能问题时才使用线程化I/O，因为除非Redis实例能够占用相当大的CPU时间，否则使用此功能没有意义。

> redis.conf相关配置翻译

- **配置IO线程数**

如果你的机器是4核的，可以配置2个或者3个线程。如果你有8核，可以配置6个线程。通过下面这个参数来配置线程数：

```bash
io-threads 4
```

将`io-threads`设置为1将只使用主线程。当启用I/O线程时，我们只使用多线程进行写操作，也就是说，执行`write(2)`系统调用并将Client缓冲区传输到套接字。但是，也可以通过将以下配置指令设置为`yes`来启用读取线程和协议解析：

```bash
io-threads-do-reads no
```

通常情况下多线程的read并没有什么卵用。

需要注意的两点是：

- 这两个配置不能运行时通过`CONFIG SET`来改变，而且开启SSL功能时，多线程I/O同样不会生效。
- 如果你想用benchmark脚本测试多线程下的性能提升，确保benchmark也是多线程模式，在后面加上`--threads`参数，来匹配Redis的线程数。不然看不到什么性能提升。

### KERNEL OOM CONTROL 设置OOM时终止哪些进程

在Linux上，可以提示内核OOM killer在OOM发生时应该首先终止哪些进程。

启用此功能可使Redis根据其角色主动控制其所有进程的oom_score_adj值。默认分数将尝试在所有其他进程之前杀死背景子进程，并在主进程之前杀死从节点进程。

Redis支持三个选项：

- `no`：对`oom-score-adj`不做任何修改（默认值）
- `yes`：`relative`的别名
- `absolute`：`oom-score-adj-values`配置的值将写入内核
- `relative`：当服务器启动时，使用相对于`oom_score_adj`初始值的值，然后将其限制在-1000到1000的范围内。因为初始值通常为0，所以它们通常与绝对值匹配。

```bash
oom-score-adj no
```

当使用`oom-score-adj`选项（不为`no`）时，该指令控制用于**主、从和后台子进程**的特定值。数值范围为-2000到2000（越高意味着死亡的可能性越大）。

非特权进程（不是根进程，也没有CAP_SYS_RESOURCE功能）可以自由地增加它们的价值，但不能将其降低到初始设置以下。这意味着将oom score adj设置为“相对”，并将oom score adj值设置为正值将始终成功

```bash
# 分别控制主进程、从进程和后台子进程的值
oom-score-adj-values 0 200 800
```

### APPEND ONLY MODE AOF持久化配置

- **开始/关闭aof**

    ```bash
    appendonly no
    ```

- **aof文件名称**

    ```bash
    appendfilename "appendonly.aof"
    ```

- **执行fsync()系统调用刷盘的频率**

    ```bash
    appendfsync everysec
    ```

    - `everysec`：每秒执行，可能会丢失最后一秒的数据。
    - `always`：每次写操作执行，数据最安全，但是对性能有影响。
    - `no`：不强制刷盘，由内核决定什么时候刷盘，数据最不安全，性能最好。

- **当有后台保存任务时，关闭appendfsync**

    当后台在执行save任务或者aof文件的rewrite时，会对磁盘造成大量I/O操作，在某些Linux配置中，Redis可能会在`fsync()`系统调用上阻塞很长时间。需要注意的是，目前还没有很好的解决方法，因为即使是在不同的线程中执行`fsync()`调用也会阻塞`write(2)`调用。

    为了缓解上述问题，可以使用以下选项，防止在进行`BGSAVE`或者`BGREWRITEAOF`时在主进程中调用`fsync()`。

    这意味这如果有其它子进程在执行saving任务时，Redis的行为相当于配置了`appendfsync none`。实际上，这意味着在最坏的情况下（使用Linux默认设置），可能丢失最多30s的日志。

    如果您有延迟的问题（性能问题），将此设置为“yes”，否则，设置为“no”。从持久化的角度看，这是最安全的选择。

    ```bash
    no-appendfsync-on-rewrite no
    ```

- **自动重写aof文件**

    在AOF文件大小增长到了指定的百分比（相对于上次AOF文件大小的增长量）或者最小体积时，自动调用`BGREWRITEAOF`命令重写AOF文件。

    ```bash
    auto-aof-rewrite-percentage 100
    auto-aof-rewrite-min-size 64mb
    ```

- **AOF文件末尾被截断**

    在Redis启动过程的最后，当AOF数据加载回内存时，可能会发现AOF文件被截断。当运行Redis的系统崩溃时，可能会发生这种情况，尤其是在安装ext4文件系统时，没有`data=ordered`选项（然而，当Redis本身崩溃或中止，但操作系统仍然正常工作时，这种情况不会发生）。

    Redis可以在出现这种情况时带着错误退出，也可以加载尽可能多的数据（现在是默认值），并在发现AOF文件在最后被截断时启动。以下选项控制此行为。

    如果aof load truncated设置为yes，则会加载一个被截断的aof文件，Redis服务器开始发送日志，通知用户该事件。否则，如果该选项设置为“no”，服务器将因错误而中止并拒绝启动。当选项设置为“no”时，用户需要使用“redis-check-aof”实用程序修复AOF文件，然后才能重新启动服务器。

    请注意，如果在中间发现AOF文件已损坏，服务器仍将退出并出现错误。此选项仅适用于Redis尝试从AOF文件读取更多数据，但找不到足够字节的情况。

    ```bash
    aof-load-truncated yes
    ```

- **开启混合持久化**

    当重写AOF文件时，Redis能够在AOF文件中使用RDB前导，以更快地重写和恢复。启用此选项后，重写的AOF文件由两个不同的节组成：

    \[RDB file][AOF tail]

    加载时，Redis识别出AOF文件以“Redis”字符串开头，并加载带前缀的RDB文件，然后继续加载AOF尾部。

    ```bash
    aof-use-rdb-preamble yes
    ```

### LUA SCRIPTING-LUA脚本相关

- **配置LUA脚本最大执行时长**

    单位毫秒，默认5s。当脚本运行时间超过限制后，Redis将开始接受其他命令当不会执行，而是会返回`BUSY`错误。

    ```bash
    lua-time-limit 5000
    ```

### REDIS CLUSTER 集群配置

- **允许集群模式**

    只有以集群模式启动的Redis实例才能作为集群的节点

    ```bash
    cluster-enabled yes
    ```

- **集群配置文件**

    由Redis创建维护，不需要我们关心内容，只需要配好位置即可

    ```bash
    cluster-config-file nodes-6379.conf
    ```

- **节点超时时间**

    集群模式下，master节点之间会互相发送`PING`心跳来检测集群master节点的存活状态，超过配置的时间没有得到响应，则认为该master节点主观宕机。

    ```bash
    cluster-node-timeout 15000
    ```

- **设置副本有效因子**

    副本数据太老旧就不会被选为故障转移的启动者。

    副本没有简单的方法可以准确测量其“数据年龄”，因此需要执行以下两项检查：

    - 如果有多个复制副本能够进行故障切换，则它们会交换消息，以便尝试为具有最佳复制偏移量的副本提供优势（已经从master接收了尽可能多的数据的节点更可能成为新master）。复制副本将尝试按偏移量获取其排名，并在故障切换开始时应用与其排名成比例的延迟（排名越靠前的越早开始故障迁移）。
    - 每个副本都会计算最后一次与其主副本交互的时间。这可以是最后一次收到的`PING`或命令（如果主机仍处于“已连接”状态），也可以是与主机断开连接后经过的时间（如果复制链路当前已关闭）。如果最后一次交互太旧，复制副本根本不会尝试故障切换。

    第二点的值可以由用户调整。特别的，如果自上次与master交互以来，经过的时间大于`(node-timeout * cluster-replica-validity-factor) + repl-ping-replica-period`，则不会成为新的master。

    较大的`cluster-replica-validity-factor`可能允许数据太旧的副本故障切换到主副本，而太小的值可能会阻止群集选择副本。

    为了获得最大可用性，可以将`cluster-replica-validity-factor`设置为0，这意味着，无论副本上次与主机交互的时间是什么，副本都将始终尝试故障切换主机。（不过，他们总是会尝试应用与其偏移等级成比例的延迟）。

    0是唯一能够保证当所有分区恢复时，集群始终能够继续的值（保证集群的可用性）。

    ```bash
    cluster-replica-validity-factor 10
    ```

- **设置master故障转移时保留的最少副本数**

    群集某个master的slave可以迁移到孤立的master，即没有工作slave的master。这提高了集群抵御故障的能力，因为如果孤立master没有工作slave，则在发生故障时无法对其进行故障转移。

    只有在slave的旧master的其他工作slave的数量至少为给定数量时，slave才会迁移到孤立的master。这个数字就是`cluster-migration-barrier`。值为1意味着slave只有在其master至少有一个其他工作的slave时才会迁移，以此类推。它通常反映集群中每个主机所需的副本数量。

    默认值为1（仅当副本的主副本至少保留一个副本时，副本才会迁移）。要禁用迁移，只需将其设置为非常大的值。可以设置值0，但仅对调试有用，并且在生产中很危险。

    ```bash
    cluster-migration-barrier 1
    ```

- **哈希槽全覆盖检查**

    默认情况下，如果Redis群集节点检测到至少有一个未覆盖的哈希槽（没有可用的节点为其提供服务），它们将停止接受查询。这样，如果集群部分关闭（例如，一系列哈希槽不再被覆盖），那么所有集群最终都将不可用。一旦所有插槽再次被覆盖，它就会自动返回可用状态。

    然而，有时您希望正在工作的集群的子集继续接受对仍然覆盖的密钥空间部分的查询。为此，只需将`cluster-require-full-coverage`选项设置为no。

    ```bash
    cluster-require-full-coverage yes
    ```

- **是否自动故障转移**

    当设置为“yes”时，此选项可防止副本在主机故障期间尝试故障切换master。但是，如果被迫这样做，主机仍然可以执行手动故障切换。

    这在不同的场景中很有用，尤其是在多个数据中心运营的情况下，如果不在DC（DataCenter？）完全故障的情况下，我们希望其中一方永远不会升级为master。

     ```bash
     cluster-replica-no-failover no
     ```

- **集群失败时允许节点处理读请求**

    此选项设置为“yes”时，允许节点在集群处于关闭状态时提供读取流量，只要它认为自己拥有这些插槽。

    这对两种情况很有用。第一种情况适用于在节点故障或网络分区期间应用程序不需要数据一致性的情况。其中一个例子是缓存，只要节点拥有它应该能够为其提供服务的数据。

    第二个用例适用于不满足三个分片集群，但又希望启用群集模式并在以后扩展的配置。不设置该选项而使用1或2分片配置中的master中断服务会导致整个集群的读/写服务中断。如果设置此选项，则只会发生写中断。如果达不到master的quorum（客观宕机）数值，插槽所有权将不会自动更改。

    ```bash
    cluster-allow-reads-when-down no
    ```

### CLUSTER DOCKER/NAT support

- **声明访问IP、port**

    以下三项设置对NAT网络或者Docker的支持。

    因为NAT端口映射的IP地址在局域网之外是没办法访问到的，因此在这种情况下，要声明集群的公网网关（NAT映射）/宿主机的IP地址，以便局域网之外也可以访问到NAT映射后的/Docker容器内的Redis集群中的每个实例。

    `cluster-announce-bus-port`集群节点之间进行数据交换的额外端口。

    ```bash
    cluster-announce-ip
    cluster-announce-port
    cluster-announce-bus-port
    ```

### SLOW LOG 慢日志

Redis的慢查询日志功能用于**记录执行时间超过给定时长的命令请求**，用户可以通过这个功能产生的日志来监视和优化查询速度

- **设置慢日志记录阈值**

    超过这个值的命令会被记录到慢日志中，默认10000微秒。

    ```bash
    slowlog-log-slower-than <microseconds>
    ```

- **慢日志文件大小**

    可以通过这个配置改变慢日志文件的最大长度，超过这个长度后最旧的记录会被删除。默认128。

    ```bash
    slowlog-max-len 128
    ```

### LATENCY MONITOR 延迟监控

Redis延迟监控子系统在运行时对不同的操作进行采样，以收集与Redis实例可能的延迟源相关的数据。

通过延迟命令，用户可以打印图表和获取报告。

系统仅记录在等于或大于通过延迟监视器阈值配置指令指定的毫秒数的时间内执行的操作。当其值设置为零时，延迟监视器将关闭。

默认情况下，延迟监控是禁用的，因为如果没有延迟问题，通常不需要延迟监控，而且收集数据会对性能产生影响，虽然影响很小，但可以在大负载下进行测量。如果需要，可以在运行时使用命令`CONFIG SET latency-monitor-threshold <millists>`轻松启用延迟监控。

- **设置延迟阈值**

    ```bash
    latency-monitor-threshold 0
    ```

### EVENT NOTIFICATION 事件通知

[Redis keyspace notifications](https://redis.io/docs/manual/keyspace-notifications/)

实时的监控keys和values的更改。

Redis可以将key space中发生的事件通过发布/订阅通知客户端。

例如，如果`notify-keyspace-events`已经启用，并且客户端对数据库0中存储的键`foo`执行DEL操作，则将通过Pub/Sub发布两条消息：

- PUBLISH \__keyspace@0__:foo del
- PUBLISH \__keyevent@0__:del foo

可以在一组类中选择Redis将通知的事件。每个类由一个字符标识：

- `K`		 Keyspace事件，通过`__keyspace@<db>__`前缀发布。
- `E`		 Keyevent事件，通过`__keyevent@<db>__ `前缀发布。
- `g`		 通用命令（非特定类型），例如`DEL`，`EXPIRE`，`RENAME`...
- `$`		 String相关命令
- `l`		 List相关命令
- `s`		 Set相关命令
- `h`		 Hash相关命令
- `z`		 Sorted Set（ZSet）相关命令
- `x`		 过期事件（每次key过期时生成的事件）
- `e`		 回收事件（达到`maxmemory`时回收key的事件）
- `t`		 Stream相关命令
- `m`		 Key-miss events，访问的key不存在时触发
- `A`		 `g$lshzxet`的别名，因此`AKE`代表了除了`m`之外的所有事件。

默认情况下所有事件通知都是关闭的，因为大多数用户不需要这些特性。且需要至少有`K`或者`E`时事件通知才会生效。

```bash
notify-keyspace-events ""
```

### GOPHER SERVER  Gopher协议

开启`Gopher`协议，大体意思就是说这是一个90年代很流行的Web协议，客户端和服务端实现都非常简单，Redis服务器只需要100行代码就能支持它。一些人想要一个更简单的互联网，另一些人认为主流互联网变得过于受控，为想要一点新鲜空气的人创造一个替代空间是很酷的。总之，为了庆祝🎉Redis诞生10周年，Redis的作者将这个协议支持作为礼物🎁送给了Redis。

```bash
# Redis contains an implementation of the Gopher protocol, as specified in
# the RFC 1436 (https://www.ietf.org/rfc/rfc1436.txt).
#
# The Gopher protocol was very popular in the late '90s. It is an alternative
# to the web, and the implementation both server and client side is so simple
# that the Redis server has just 100 lines of code in order to implement this
# support.
#
# What do you do with Gopher nowadays? Well Gopher never *really* died, and
# lately there is a movement in order for the Gopher more hierarchical content
# composed of just plain text documents to be resurrected. Some want a simpler
# internet, others believe that the mainstream internet became too much
# controlled, and it's cool to create an alternative space for people that
# want a bit of fresh air.
#
# Anyway for the 10nth birthday of the Redis, we gave it the Gopher protocol
# as a gift.
#
# --- HOW IT WORKS? ---
#
# The Redis Gopher support uses the inline protocol of Redis, and specifically
# two kind of inline requests that were anyway illegal: an empty request
# or any request that starts with "/" (there are no Redis commands starting
# with such a slash). Normal RESP2/RESP3 requests are completely out of the
# path of the Gopher protocol implementation and are served as usual as well.
#
# If you open a connection to Redis when Gopher is enabled and send it
# a string like "/foo", if there is a key named "/foo" it is served via the
# Gopher protocol.
#
# In order to create a real Gopher "hole" (the name of a Gopher site in Gopher
# talking), you likely need a script like the following:
#
#   https://github.com/antirez/gopher2redis
#
# --- SECURITY WARNING ---
#
# If you plan to put Redis on the internet in a publicly accessible address
# to server Gopher pages MAKE SURE TO SET A PASSWORD to the instance.
# Once a password is set:
#
#   1. The Gopher server (when enabled, not by default) will still serve
#      content via Gopher.
#   2. However other commands cannot be called before the client will
#      authenticate.
#
# So use the 'requirepass' option to protect your instance.
#
# Note that Gopher is not currently supported when 'io-threads-do-reads'
# is enabled.
#
# To enable Gopher support, uncomment the following line and set the option
# from no (the default) to yes.
#
# gopher-enabled no
```

### ADVANCED CONFIG  高级设置

- **设置Hash底层数据结构由ziplist转为hashtable的阈值**

    当Hash类型的keys只包含了少量的实体并且实体的大小没有超过给定的阈值时，Hash底层会使用ziplist来存储数据而不是使用hashtable以节省空间。

    ```bash
    hash-max-ziplist-entries 512
    hash-max-ziplist-value 64
    ```

    当一个Hash类型的key包含的实体数量超过了`hash-max-ziplist-entries`的值或者某个实体的大小超过了`hash-max-ziplist-value`的值（单位字节），那么底层编码就会升级成hashtable。

- **设置List底层数据结构quicklist中单个ziplist的大小**

    Redis中List数据结构的底层使用的是quicklist的数据结构，本质上是ziplist作为节点串起来的linkedlist。可以通过该项设置来改变每个ziplist的最大大小（ziplist中的`fill`属性，超过这个值就会开启一个新的ziplist）。总共提供了-5到-1五个选项：

    - `-5`：最大大小为64Kb，不推荐作为正常情况下的负载
    - `-4`：最大大小为32Kb，不推荐
    - `-3`：最大大小为16Kb，大概可能估计好像不是很推荐（原话：probably not recommended）
    - `-2：最大大小为8Kb，good（原话）
    - `-1`：最大大小为4Kb，good（原话）

    默认值是`-2`

    ```bash
    list-max-ziplist-size -2
    ```

- **设置压缩List中ziplist为quicklistLZF结构**

    大神们觉着ziplist不够zip啊，所以再压缩一下吧。实际上是考虑了这样的场景，即List数据结构两端访问频率比较高，但是中间部分访问频率不是很高的情况，那么使用ziplist存放这部分结构就有点浪费，是不是可以把这部分结构进行压缩（LZF算法压缩）呢？这个选项就是进行这个操作的。有下面几个值：

    - `0`：代表不压缩，默认值
    - `1`：两端各一个节点不压缩
    - `2`：两端各两个节点不压缩
    - `...`：依次类推。。。

    ```bash
    list-compress-depth 0
    ```

- **设置Set底层intset最大entities个数/intset升级为hashtable的阈值**

    Set数据结构只有在一种情况下会使用intset来存储：set由能转成10进制且数值在64bit有符号整形数值组成时。下面的配置设置了intset能存储的最大entities数量，超过这个数量会转成hashtable存储。默认512个。

    ```bash
    set-max-intset-entries 512
    ```

- **设置ZSet底层数据结构由ziplist转为skiplist的阈值**

    当超过下面设置的阈值时，ZSet底层存储结构会由ziplist转为skiplist。

    ```bash
    zset-max-ziplist-entries 128
    zset-max-ziplist-value 64
    ```

- **设置HyperLogLog底层稀疏矩阵转为稠密矩阵的阈值**

    HyperLogLog当在计数比较小时会使用稀疏矩阵来存储，只有当计数达到阈值时，才会转为稠密矩阵。

    超过16000的值是完全无用的，因为这种情况下使用稠密矩阵更加节省内存。

    建议的值是3000左右，以便在不降低太多PFADD速度的情况下获取空间有效编码的好处，稀疏编码的PFADD的时间复杂度为O(N)。当不考虑CPU占用时而考虑内存占用时，这个值可以升到10000左右。

    ```bash
    hll-sparse-max-bytes 3000
    ```

- **自定义Stream宏节点大小**

    可以通过`stream-node-max-bytes`选项修改Stream中每个宏节点能够占用的最大内存，或者通过`stream-node-max-entries`参数指定每个宏节点中可存储条目的最大数量。

    ```bash
    stream-node-max-bytes 4096
    stream-node-max-entries 100
    ```

- **开启Rehash**

    Redis将在每100毫秒时使用1毫秒的CPU时间来对redis的hash表进行重新hash，可以降低内存的使用。当你的使用场景中，有非常严格的实时性需要，不能够接受Redis时不时的对请求有2毫秒的延迟的话，把这项配置为no。如果没有这么严格的实时性要求，可以设置为yes，以便能够尽可能快的释放内存。

    ```bash
    activerehashing yes
    ```

- **客户端输出缓存控制**

    客户端输出缓冲区限制可用于强制断开由于某种原因从服务器读取数据速度不够快的客户端（一个常见原因是发布/订阅客户端不能像发布服务器生成消息那样快地使用消息）。

    对于三种不同类型的客户端，克制设置不同的限制：

    - `normal`：一般客户端包含监控客户端
    - `replica`：副本客户端（slave）
    - `pubsub`：客户端至少订阅了一个pubsub通道或模式。

    每个客户端输出缓冲区限制指令语法：

    ```bash
    client-output-buffer-limit <class> <hard limit> <soft limit> <soft seconds>
    ```

    一旦达到`<hard limit>`限制或者达到`<soft limit>`之后又过了`<soft seconds>`秒，那么客户端会立即被断开连接。

    例如，如果`<hard limit>`为32兆字节，`<soft limit>`和`<soft seconds>`分别为16兆字节，10秒，则如果输出缓冲区的大小达到32兆字节，客户端将立即断开连接，但如果客户端达到16兆字节并连续超过限制10秒，客户端也将断开连接。

    默认情况下，普通客户端不受限制，因为它们不会在没有请求（以推送方式）的情况下接收数据，而是在请求之后接收数据，因此只有异步客户端可能会创建一个场景，其中请求数据的速度比读取数据的速度快。

    相反，pubsub和副本客户端有一个默认限制，因为订阅者和副本以推送方式接收数据。

    硬限制或软限制都可以通过将其设置为零来禁用。

    ```bash
    client-output-buffer-limit normal 0 0 0
    client-output-buffer-limit replica 256mb 64mb 60
    client-output-buffer-limit pubsub 32mb 8mb 60
    ```

- **配置客户端query buffer大小**

    客户端query buffer大小不能超过该项配置的值。

    > 每个Client都有一个query buffer(查询缓存区或输入缓存区), 它用于保存客户端的发送命令，redis server从query buffer获取命令并执行。如果程序的Key设计不合理，客户端使用大量的query buffer，这会导致redis server比较危险，很容易达到maxmeory限制，导致缓存数据被清空、数据无法写入和oom.
    >
    > https://blog.csdn.net/u012271526/article/details/107208295

    ```bash
    client-query-buffer-limit 1gb
    ```

- **Redis协议批量请求单个字符串限制**

    默认512mb，可以通过下面选项修改

    ```bash
    proto-max-bulk-len 512mb
    ```

- **Redis执行任务频率**

    Redis调用一个内部函数来执行许多后台任务，比如在超时时关闭客户端连接，清楚从未被请求过的过期key...

    并非所有任务都已相同的频率执行，但Redis根据指定的`hz`值检查要执行的任务。

    默认情况下，`hz`的值为10.提高这个值会让Redis在空闲的时候占用更多的CPU，但同时也会让Redis在有很多keys同时过期时响应更快并且可以更精确的处理超时。

    范围在1到500之间，但是超过100通常不是一个好主意。大多数用户应该使用缺省值10，只有在需要非常低延迟的环境中才应该将值提高到100。

    ```bash
    hz 10
    ```

- **动态hz配置**

    根据客户端连接的数量动态的调整hz的值，当有更多的客户端连接时，会临时以`hz`设置基准提高该`hz`的值。默认开启。

    ```bash
    dynamic-hz yes
    ```

- **AOF重写时执行fsync刷盘策略**

    当一个子系统重写AOF文件时，如果启用了以下选项，则该文件将每生成32MB的数据进行fsync同步。这对于以更增量的方式将文件提交到磁盘并避免较大的延迟峰值非常有用。

    ```bash
    aof-rewrite-incremental-fsync yes
    ```

- **保存RDB文件时执行fsync刷盘策略**

    当redis保存RDB文件时，如果启用以下选项，则每生成32 MB的数据，文件就会同步一次。这对于以更增量的方式将文件提交到磁盘并避免较大的延迟峰值非常有用。

    ```bash
    rdb-save-incremental-fsync yes
    ```

- **LFU设置**

    设置Redis LFU相关。Redis LFU淘汰策略实现有两个可调整参数：`lfu-log-factor`和`lfu-decay-time`。

    ```bash
    lfu-log-factor 10
    lfu-decay-time 1
    ```

### ACTIVE DEFRAGMENTATION  碎片整理

主动（在线）碎片整理允许Redis服务器压缩内存中数据的少量分配和释放之间的空间（内存碎片），从而回收内存。

碎片化是每个分配器（幸运的是，Jemalloc比较少发生这种情况）和某些工作负载都会发生的自然过程。通常需要重启服务器以降低碎片，或者至少清除所有数据并重新创建。然而，多亏了Oran Agra为Redis 4.0实现的这一功能，这个过程可以在服务器运行时以“hot”的方式在运行时发生（类似热部署的意思，不需要停止服务）。

基本上，当碎片超过某个级别（参见下面的配置选项）时，Redis将通过利用特定的Jemalloc功能（以了解分配是否导致碎片并将其分配到更好的位置）开始在连续内存区域中创建值的新副本，同时释放数据的旧副本。对所有键递增地重复该过程将导致碎片降至正常值。

需要了解的重要事项：

1.默认情况下，此功能被禁用，并且仅当您编译Redis以使用我们随Redis源代码提供的Jemalloc副本时，此功能才有效。这是Linux版本的默认设置。

2.如果没有碎片问题，则无需启用此功能。

3.一旦遇到内存碎片，可以在需要时使用命令`CONFIG SET activedefrag yes`启用此功能。

配置参数能够微调碎片整理过程的行为。如果你不确定它们是什么意思，最好不要改变默认值。

- **开启活动碎片整理**

    ```bash
    activedefrag no
    ```

- **启动活动碎片整理的最小内存碎片阈值**

    ```bash
    active-defrag-ignore-bytes 100mb
    ```

- **启动活动碎片整理的最小内存碎片百分比**

    ```bash
    active-defrag-threshold-lower 10
    ```

- **尝试释放的最大百分比**

    ```bash
    active-defrag-threshold-upper 100
    ```

- **最少CPU使用率**

    ```bash
    active-defrag-cycle-min 1
    ```

- **最大CPU使用率**

    ```bash
    active-defrag-cycle-max 25
    ```

- **最大扫描量**

    ```bash
    # Maximum number of set/hash/zset/list fields that will be processed from
    # the main dictionary scan
    active-defrag-max-scan-fields 1000
    ```

- **使用后台线程**

    ```bash
    jemalloc-bg-thread yes
    ```

## Redis 发布与订阅

发布订阅（pub/sub）是一种消息通信模式：发送者（pub）发送消息，订阅者（sub）接收消息。Redis客户端可以订阅任意数量的频道。

![image-20220518211300027](https://haiqiang-picture.oss-cn-beijing.aliyuncs.com/blog/redis08.png)



![image-20220518211401669](https://haiqiang-picture.oss-cn-beijing.aliyuncs.com/blog/redis09.png)

____

**Redis发布订阅命令**

- `subscribe channel [channel ...]` 订阅通道
- `publish channel message` 向通道发送消息
- `psubscribe pattern [pattern ...]` 按通配符订阅频道，需要注意的是，如果两个不同的模式匹配了同一个channel，那么消息会被向客户端发送两次。

## Redis 事务、锁机制

### Redis事务

- **概念**

    Redis 事务是一个单独的隔离操作：事务中的所有命令都会序列化、按顺序地执行。事务在执行的过程中，不会被其他客户端发送来的命令请求所打断。

    Redis 事务的主要作用就是**串联多个命令**防止别的命令插队。

- **事务命令**

    - `multi`        标记一个事务块的开始，执行后，在执行`EXEC`或`DISCARD`之前的命令会放入队列缓存，而不会直接执行。
    - `exec`          执行事务块
    - `discard`     放弃事务块

    需要注意的是，Redis事务中某个命令执行（执行`exec`之后）失败，其余命令仍然会被执行。如果执行`multi`后执行`exec`之前的某个命令报错，那么执行`exec`时事务会失败。
    
- Redis事务的三大特性

    - 单独的隔离操作 ：事务中的所有命令都会序列化、按顺序地执行。事务在执行的过程中，不会被其他客户端发送来的命令请求所打断。
    - 没有隔离级别的概念 ：队列中的命令没有提交之前都不会实际被执行，因为事务提交前任何指令都不会被实际执行。
    - 不保证原子性 ：事务中如果有一条命令执行失败，其后的命令仍然会被执行，没有回滚 。


### 事务冲突问题

例子：

账户金额总共有10000，现在有三个请求

- 一个请求想给金额减 8000；
- 一个请求想给金额减 5000；
- 一个请求想给金额减 1000；

最终结果是-4000，显然是不合理的

![image-20220519094341090](https://haiqiang-picture.oss-cn-beijing.aliyuncs.com/blog/redis10.png)

可以通过加锁来解决。

### 悲观锁

![image-20220519094633896](https://haiqiang-picture.oss-cn-beijing.aliyuncs.com/blog/redis11.png)

**悲观锁 (Pessimistic Lock)**，顾名思义，就是很悲观，每次去拿数据的时候都认为别人会修改，所以每次在拿数据的时候都会上锁，这样别人想拿这个数据就会 block 直到它拿到锁。**传统的关系型数据库里边就用到了很多这种锁机制**，比如**行锁**，**表锁**等，**读锁**，**写锁**等，都是在做操作之前先上锁。 

### 乐观锁

![image-20220519094741479](https://haiqiang-picture.oss-cn-beijing.aliyuncs.com/blog/redis12.png)

**乐观锁 (Optimistic Lock)**，顾名思义，就是很乐观，每次去拿数据的时候都认为别人不会修改，所以不会上锁，但是在更新的时候会判断一下在此期间别人有没有去更新这个数据，可以使用版本号等机制。**乐观锁适用于多读的应用类型，这样可以提高吞吐量**。Redis 就是利用这种 **check-and-set 机制**实现事务的。

### Watch机制

- ` watch key [key ...]  `        在执行 multi 之前，先执行`watch key1 [key2]`，可以监视一个 (或多个) key ，如果在事务**执行之前这个 (或这些) key 被其他命令所改动，那么事务将被打断。**
- `unwatch`                               取消 WATCH 命令对所有 key 的监视。如果在执行 WATCH 命令之后，EXEC 命令或 DISCARD 命令先被执行了的话，那么就不需要再执行 UNWATCH 了。

 ### LUA脚本

[Redis Lua API reference](https://redis.io/docs/manual/programmability/lua-api/)

[Scripting with Lua](https://redis.io/docs/manual/programmability/eval-intro/)

 Redis使用同一个Lua解释器来执行所有命令，同时，Redis保证以一种原子性的方式来执行脚本：当lua脚本在执行的时候，不会有其他脚本和命令同时执行，这种语义类似于 MULTI/EXEC。从别的客户端的视角来看，一个lua脚本要么不可见，要么已经执行完。

然而这也意味着，执行一个较慢的lua脚本是不建议的，由于脚本的开销非常低，构造一个快速执行的脚本并非难事。但是你要注意到，当你正在执行一个比较慢的脚本时，所以其他的客户端都无法执行命令。

___

**Redis 脚本相关命令**

- `EVAL script numkeys [key [key ...]] [arg [arg ...]]` 使用内嵌的Lua解释器执行脚本。
    - `script`               一段Lua脚本，它会被运行在Redis中内嵌的Lua5.1解释器上，这段脚本不必（也不应该）定义为一个Lua函数。
    - `numkeys`             用于指定键名参数的个数，之后跟着对应数量的Lua脚本所使用的Redis keys的个数。在脚本中用KEYS[n]来访问，基准从1开始( KEYS[1] ， KEYS[2] ，以此类推)。
    - `arg [arg ...]`  附加参数，在 Lua 中通过全局变量 ARGV 数组访问，访问的形式和 KEYS 变量类似( ARGV[1] 、 ARGV[2] ，诸如此类)。
- `EVALSHA sha1 numkeys key [key ...] arg [arg ...]`  根据给定的sha1校验码，执行缓存在服务器中的脚本。
    - `sha1`             通过 SCRIPT LOAD 生成的 sha1 校验码。
    - `numkeys`        用于指定键名参数的个数，之后跟着对应数量的Lua脚本所使用的Redis keys的个数。在脚本中用KEYS[n]来访问，基准从1开始( KEYS[1] ， KEYS[2] ，以此类推)。
    - `arg [arg ...]`  附加参数，在 Lua 中通过全局变量 ARGV 数组访问，访问的形式和 KEYS 变量类似( ARGV[1] 、 ARGV[2] ，诸如此类)。
- `SCRIPT EXISTS sha1 [sha1 ...]`       用于校验指定的脚本是否已经被保存在缓存当中。返回值为一个列表，包含0和1,0代表不存在缓存中，后者标着已经在缓存里。
- `SCRIPT FLUSH `                                      清除所有 Lua 脚本缓存。
- `SCRIPT KILL `                                        杀死正在执行的Lua脚本，当且仅当这个脚本没有执行过任何写操作时，命令才生效。
- `SCRIPT DEBUG`                                      开启对脚本的调试。可以单步执行，设置断点，观察变量等等。
- `SCRIPT LOAD`                                        将脚本 script 添加到脚本缓存中，但并不立即执行这个脚本。



## Redis 持久化

Redis 提供了两种不同形式的持久化方式：

- RDB（Redis DataBase）
- AOF（Append Of File）

### RDB

- **简介**

    在指定的时间间隔内将内存中的数据集快照写入磁盘， 也就是行话讲的 Snapshot 快照，它恢复时是将快照文件直接读到内存里。

- **RDB备份执行过程**

    Redis 会单独创建（fork）一个子进程来进行持久化，首先会将数据写入到一个临时文件中，待持久化过程都结束了，再用这个临时文件替换上次持久化好的文件。整个过程中，主进程是不进行任何 IO 操作的，这就确保了极高的性能。如果需要进行大规模数据的恢复，且对于数据恢复的完整性不是非常敏感，那 RDB 方式要比 AOF 方式更加的高效。**RDB 的缺点是最后一次持久化后的数据可能丢失**。

    ![image-20220519103155712](https://haiqiang-picture.oss-cn-beijing.aliyuncs.com/blog/redis13.png)

- **RDB文件**

    在Redis.conf中配置文件名称以及保存路径，默认为启动目录下`./dump.rdb`。可以通过`dir`配置修改。

- **触发RDB快照保存策略**

    可以在Redis.conf中配置相关策略，也可使使用命令触发。

    - `SAVE` ： 执行一个同步保存操作，将当前 Redis 实例的所有数据快照(snapshot)以 RDB 文件的形式保存到硬盘。阻塞。
    - `BGSAVE`：执行一个异步保存操作，将当前 Redis 实例的所有数据快照(snapshot)以 RDB 文件的形式保存到硬盘。不阻塞。
    - `LASTSAVE`： 获取最后一次成功执行快照的时间。

- **优势**

    - 适合大规模的数据恢复
    - 对数据完整性和一致性要求不高的场景适合使用
    - 节省磁盘空间
    - 恢复速度快

    ![image-20220519104039665](https://haiqiang-picture.oss-cn-beijing.aliyuncs.com/blog/redis14.png)

- **劣势**

    - Fork 的时候，内存中的数据被克隆了一份，大致 2 倍的膨胀性需要考虑。
    - 虽然 Redis 在 fork 时使用了**写时拷贝技术**，但是如果数据庞大时还是比较消耗性能。
    - 在备份周期在一定间隔时间做一次备份，所以如果 Redis 意外 down 掉的话，就会丢失最后一次快照后的所有修改。

- **关闭RDB**

    通过`redis-cli config set save ""` 来在运行时禁用保存策略，也可以在Redis.conf中提前配置。

- **总结**

    ![image-20220519104309571](https://haiqiang-picture.oss-cn-beijing.aliyuncs.com/blog/redis15.png)

### AOF

- **简介**

    以**日志**的形式来记录每个写操作（增量保存），将 Redis 执行过的所有写指令记录下来 (**读操作不记录**)， **只许追加文件但不可以改写文件**，redis 启动之初会读取该文件重新构建数据，换言之，redis 重启的话就根据日志文件的内容将写指令从前到后执行一次以完成数据的恢复工作。

- **AOF持久化流程**

    - 客户端的请求写命令会被append追加到AOF缓冲区内；

    - AOF缓冲区根据AOF持久化策略[aloways, everysec, no]将操作 sync同步到磁盘的AOF文件中；

    - AOF文件大小超过重写策略或手动重写时，会对AOF文件rewrite重写，压缩AOF文件容量；

    - Redis服务重启时，会重新load加载AOF文件中的写操作达到数据恢复的目的。

        ![image-20220519104813563](https://haiqiang-picture.oss-cn-beijing.aliyuncs.com/blog/redis16.png)

- **开启AOF**

    AOF默认是不开启的，可以在Redis.conf中配置，默认保存名称为appendonly.aof，保存路径跟RDB相同，同样可以通过`dir`配置修改。

    AOF和RDB同时开启时，系统默认选取AOF，因为AOF数据不会丢失。

- **AOF 启动、修复、恢复**

    - AOF 的备份机制和性能虽然和 RDB 不同，但是备份和恢复的操作同 RDB 一样，都是拷贝备份文件，需要恢复时再拷贝到 Redis 工作目录下，启动系统即加载。
    - 正常恢复
        - 修改默认的 appendonly no，改为 yes。
        - 将有数据的 aof 文件复制一份保存到对应目录 (查看目录：config get dir)。
        - 恢复：重启 redis 然后重新加载。
    - 异常恢复
        - 修改默认的 appendonly no，改为 yes。
        - 如遇到 **AOF 文件损坏**，通过 /usr/local/bin/ **redis-check-aof–fix appendonly.aof** 进行恢复。
        - 备份被写坏的 AOF 文件。
        - 恢复：重启 redis，然后重新加载。

- **AOF 同步频率设置**

    - `appendfsync always`：始终同步，每次 Redis 的写入都会立刻记入日志；性能较差但数据完整性比较好。
    - `appendfsync everysec`：每秒同步，每秒记入日志一次，如果宕机，本秒的数据可能丢失。
    - `appendfsync no`：redis 不主动进行同步，把同步时机交给操作系统。

- **AOF Rewrite压缩** 

    AOF 采用文件追加方式，文件会越来越大为避免出现此种情况，新增了重写机制，当 AOF 文件的大小超过所设定的阈值时，Redis 就会启动 AOF 文件的内容压缩，只保留可以恢复数据的最小指令集，可以使用命令 `bgrewriteaof`手动调用。

    ____

    **重写原理**：

    AOF 文件持续增长而过大时，会 fork 出一条新进程来将文件重写 (也是先写临时文件最后再 rename)，redis4.0 版本后的重写，是指把 rdb 的快照，以二进制的形式附在新的 aof 头部，作为已有的历史数据，替换掉原来的流水账操作。

    **no-appendfsync-on-rewrite**

    - 如果`no-appendfsync-on-rewrite=yes`，不写入aof文件只写入缓存，用户请求不会阻塞，但是在这段时间如果宕机会丢失这段时间的缓存数据。（降低数据安全，提高性能）
    - 如果`no-appendfsync-on-rewrite=no`，还是会把数据往磁盘里刷，但是遇到重写操作，可能会发生阻塞。（数据安全，但是性能降低）

    **触发机制、时机**

    Redis 会记录上次重写时的 AOF 大小，默认配置是当 AOF 文件大小是上次 rewrite 后大小的一倍且文件大于 64M 时触发。

    重写虽然可以节约大量磁盘空间，减少恢复时间。但是每次重写还是有一定的负担的，因此设定 Redis 要满足一定条件才会进行重写。

    - `auto-aof-rewrite-percentage`：设置重写的基准值，文件达到 100% 时开始重写（文件是原来重写后文件的 2 倍时触发）。
    - `auto-aof-rewrite-min-size`：    设置重写的基准值，最小文件64MB。达到这个值开始重写。
    - 系统载入时或者上次重写完毕时，Redis会记录此时AOF大小，设为bash_size。
    - 如果Redis的AOF当前大小>=base_size + base_size*100%（默认）且当前大小>=64MB（默认）的情况下，Redis会对AOF进行重写。例如，文件达到70MB开始重写，降到50MB，下次到达100MB时才会再次重写。

    **重写流程**

    - bgrewriteaof 触发重写，判断是否当前有 bgsave 或 bgrewriteaof 在运行，如果有，则等待该命令结束后再继续执行；
    - 主进程fork出子进程执行重写操作，保证主进程不会阻塞；
    - 子进程遍历redis内存中数据到临时文件，客户端的写请求同时写入aof_buf缓冲区和aof_rewrite_buf重写缓冲区，保证原AOF文件完整以及新AOF文件生成期间的新的数据修改动作不会丢失；
    - 子进程写完新的AOF文件后，向主进程发信号，父进程更新统计信息。主进程把aof_rewrite_buf中的数据写入到新的AOF文件；
    - 使用新的AOF文件覆盖旧的AOF文件，完成AOF重写。

    ![image-20220519110418254](https://haiqiang-picture.oss-cn-beijing.aliyuncs.com/blog/redis17.png)

- **优势**

    ![image-20220519110452361](https://haiqiang-picture.oss-cn-beijing.aliyuncs.com/blog/redis18.png)

    - 备份机制更稳健，丢失数据概率更低。
    - 可读的日志文本，通过操作AOF文件，可以处理误操作。

- **劣势**

    - 比起RDB占用更多的磁盘空间。
    - 恢复备份速度要慢。
    - 每次读写都同步的话，有一定的性能压力。
    - 存在个别Bug，造成无法恢复数据。

- **小总结**

    ![image-20220519110629704](https://haiqiang-picture.oss-cn-beijing.aliyuncs.com/blog/redis19.png)

### 选择RDB还是AOF

**官网推荐两个都启用：**

- 如果对数据安全不敏感，可以选择单独使用RDB。
- 不建议单独使用AOF，因为可能会出现Bug。
- 如果只是用做纯内存缓存，可以都不用。

**官网建议：**

- RDB 持久化方式能够在指定的时间间隔能对你的数据进行快照存储。
- AOF 持久化方式记录每次对服务器写的操作，当服务器重启的时候会重新执行这些命令来恢复原始的数据，AOF 命令以 redis 协议追加保存每次写的操作到文件末尾。
- Redis 还能对 AOF 文件进行后台重写，使得 AOF 文件的体积不至于过大。
- 只做缓存：如果你只希望你的数据在服务器运行的时候存在，你也可以不使用任何持久化方式。
- 同时开启两种持久化方式：在这种情况下，当 redis 重启的时候会优先载入 AOF 文件来恢复原始的数据，因为在通常情况下 AOF 文件保存的数据集要比 RDB 文件保存的数据集要完整。
- RDB 的数据不实时，同时使用两者时服务器重启也只会找 AOF 文件。那要不要只使用 AOF 呢？
- 建议不要，因为 RDB 更适合用于备份数据库 (AOF 在不断变化不好备份)，快速重启，而且不会有 AOF 可能潜在的 bug，留着作为一个万一的手段。
- 性能建议：
    - 因为 RDB 文件只用作后备用途，建议只在 Slave 上持久化 RDB 文件，而且只要 15 分钟备份一次就够了，只保留 save 9001 这条规则。
    - 如果使用 AOF，好处是在最恶劣情况下也只会丢失不超过两秒数据，启动脚本较简单，只 load 自己的 AOF 文件就可以了。
    - aof 代价：一是带来了持续的 IO，二是 AOF rewrite 的最后，将 rewrite 过程中产生的新数据写到新文件造成的阻塞几乎是不可避免的。
    - 只要硬盘许可，应该尽量减少 AOF rewrite 的频率，AOF 重写的基础大小默认值 64M 太小了，可以设到 5G 以上。默认超过原大小 100% 大小时重写可以改到适当的数值。

## Redis 主从复制、集群

主机数据更新后根据配置和策略， 自动同步到备机的 master/slaver 机制，**Master 以写为主，Slave 以读为主**，主从复制节点间数据是全量的。

**作用：**

- 读写分离，性能扩展

- 容灾快速恢复

    ![image-20220519111652745](https://haiqiang-picture.oss-cn-beijing.aliyuncs.com/blog/redis20.png)

____

### **主从复制原理**

通过执行`SLAVEOF host port`命令或者`REPLICAOF`（前者的别名）或者在redis.conf中配置`slaveof`/`replicaof`选项，让一个服务器去复制另一个服务器的内容。（旧版配置文件中配置`slaveof`，新版`replicaof`）。当一个slave执行`REPLICAOF`成为某个节点的从节点后，它会给它的master发送`SYNC`/`PSYNC`命令来请求同步数据。

> psync命令和sync命令的区别是，psync命令可以根据主从节点当前状态的不同决定执行全量复制还是执行增量复制，而sync同步的方式是全量复制。

____

**全量同步过程**

- slave启动成功链接到master后会发送一个psync命令请求同步（旧版是sync），如果判断执行全量同步则执行以下流程；
- master接到命令启动后台的存盘进程（BGSAVE）生成RDB文件，同时收集所有接收到的用于**修改**数据集命令存放到缓冲区中，在后台进程执行完毕之后，master将传送整个数据文件到slave，slave加载RDB文件后，同步到master节点执行BGSAVE时的状态。
- 主服务器将缓冲区中的所有**写命令**发送给从服务器，从服务器执行这些**写命令**，同步主节点更新自己的状态。

![image-20220519112440704](https://haiqiang-picture.oss-cn-beijing.aliyuncs.com/blog/redis21.png)



当slave第一次连接master，slave不知道master的replicationid，也不知道自己偏移量，这时候会传一个问号和-1，告诉master节点是第一次同步。格式为`psync ? -1`

![image-20220514101947629](https://haiqiang-picture.oss-cn-beijing.aliyuncs.com/blog/redis23.png)

**增量同步过程**

增量同步是`PSYNC`命令的功能，用于处理断连后重新连接时数据同步的情况。如果条件允许，那么master只会将断连这段时间的数据发送给slave节点，从而达成一致状态。

左下角是repl_baklog的组织形式，是一个环形缓冲复制队列，如果master节点的offset已经超过slave一个环的量，那么就不能执行增量复制了，只能执行全量复制。这是因为如果master节点的offset大于slave节点一个环的量，说明有一部分未同步的数据已经被覆盖了，是无法从repl_baklog中找到的了，所以只能全量复制了。

![image-20220514102511238](https://haiqiang-picture.oss-cn-beijing.aliyuncs.com/blog/redis24.png)



**master如何判断slave是不是第一次同步数据？**

- Replication Id：简称replid，是数据集的标记，id一致说明是同一数据集。每一个master都有唯一的replid，slave会继承master节点的replid。
- offset：偏移量，随着记录在repl_baklog中的数据增多而逐渐增大。slave完成同步时也会记录当前同步的offset。如果slave的offset小于master的offset，说明slave数据落后与master，需要更新。

### **哨兵模式（sentinel）**

在主从模式下，如果cluster宕机，可以重启后同步数据，并且在重启过程中redis服务是可用的（一主多从），但是如果master宕机，那么在master节点重启的过程中，整个redis服务是不能写的。为了解决这个问题，引入了哨兵模式来实现主从集群的自动故障恢复。

在哨兵模式下，当一个 master 宕机后，后面的 slave 可以立刻升为 master，其后面的 slave 不用做任何修改。只需要用` slaveof no one `指令将从机变为主机。哨兵模式能够后台监控主机是否故障，如果故障了根据投票数自动将从库转换为主库。哨兵本身也是集群。

![image-20210619154258222](https://haiqiang-picture.oss-cn-beijing.aliyuncs.com/blog/redis22.png)

**哨兵的作用**

- **监控：**Sentinel会不断检查master和slave是否按照预期工作。

    Sentinel是基于心跳的机制检测服务状态，每隔1秒向集群的每个实例发送`PING`命令（返回PONG那个命令）。

    - 主观下线：如果某Sentinel节点发现某实例未在规则时间响应，则认为该实例主观下线。
    - 客观下线：如果超过指定数量（quorum）的Sentinel认为该实例主机下线，则该实例客观下线。quorum值最好超过Sentinel实例数量的一半。

- **自动故障恢复：**如果master故障，Sentinel会将一个slave提升为master。当故障实例恢复后也以新的master为整个主从集群的master，旧的master重启后会变成slave。

    - Sentinel集群选举新的master。
    - Sentinel向新选举出的master发送`REPLICAOF no one`，让该节点上升为master；
    - Sentinel给其它所有slave节点发送`REPLICAOF new_host port`命令，让其它slave成功新选举出的master节点的是slave。
    - 最后将故障节点（原master）标记为slave，故障节点恢复后会自动称为新master节点的slave。

- **通知：**Sentinel充当Redis客户端的服务发现来源，当集群发生故障转移时，会将最新的信息推送给Redis的客户端。

**当主机挂掉，从机选举产生新的主机**

- 首先会判断slave节点与master节点断开时间的长短，如果超过指定值（down-after-milliseconds * 10）则会排除该slave节点。

- 然后根据优先级别：replica-priority（旧版slave-priority），值越小优先级越高，如果是0则永不参与选举。默认100。
- 如果priority一样，则判断offset值，值越大说明数据越新，优先级越高。
- 最后判断运行id，越小优先级越高。（不重要）
- 原主机重启后会变为从机。

#### Sentinel集群搭建

**配置文件sentinel.conf**：

- **配置端口号**：

    这里需要注意的是Sentinel本质上也是redis实例，跟redis.conf配置一样默认情况下不能从非localhost的网卡访问，因此需要关闭保护模式或者使用`bind`配置显式绑定本地网卡。

    ```bash
    # *** IMPORTANT ***
    #
    # By default Sentinel will not be reachable from interfaces different than
    # localhost, either use the 'bind' directive to bind to a list of network
    # interfaces, or disable protected mode with "protected-mode no" by
    # adding it to this configuration file.
    #
    # Before doing that MAKE SURE the instance is protected from the outside
    # world via firewalling or other means.
    #
    # For example you may use one of the following:
    #
    # bind 127.0.0.1 192.168.1.1
    #
    # protected-mode no
    
    # port <sentinel-port>
    # The port that this sentinel instance will run on
    port 26379
    ```

- **配置后台运行/开启守护进程**

    ```bash
    daemonize yes
    ```

- **pid文件位置**

    ```bash
    pidfile /var/run/redis-sentinel.pid
    ```

- **日志文件位置**

    ```bash
    logfile ""
    ```

- **指定访问ip和tcp port**

    在使用NAT端口映射的环境下，本机（Sentinel）使用的内网ip地址可能只在本局域网内才能访问，外网的实例要通过某个公网ip通过NAT转换才能访问（Sentinel）。但是默认情况下Sentinel会告诉对方以检测到的内网IP通信，这就会导致外部的机器没办法访问到Sentinel。这就类似你在你自己的局域网下是访问不到我这边的局域网ip地址`172.22.29.134`的，除非你和我位于同一个局域网。

    可以使用下面的配置指定其它实例访问的地址。两个选项不需要一起使用。如果使用了`sentinel announce-ip <ip>`，那么Sentinel实例会使用声明的ip地址和`port`选项指定的端口号。如果只提供后者，那么Sentinel会自动检测本地ip和指定端口。

    ```bash
    sentinel announce-ip <ip>
    sentinel announce-port <port>
    ```

- **指定工作目录**

    ```bash
    dir /tmp
    ```

- **指定监控master的信息**

    ```bash
    sentinel monitor <master-name> <ip> <redis-port> <quorum>
    ```

    - `<master-name>`：master实例的名称，可以自己命名master节点的名称。
    - `<ip>`：master实例的ip地址
    - `<redis-port>`： master实例的端口号
    - `<quorum>`：投票数，只有当num个以上的Sentinel实例认为master宕机了，Sentinel集群才会真的认为master宕机。建议值在Sentinel集群实例数量的一半以上。

- **配置Redis访问认证信息**

    配置实例访问密码：

    ```bash
    sentinel auth-pass <master-name> <password>
    ```

    - `<master-name>`：`sentinel monitor`配置中定义的master节点的名称
    - `<password>`：访问密码

    配置实例访问用户：

    ```bash
    sentinel auth-user <master-name> <username>
    ```

    - `<master-name>`：`sentinel monitor`配置中定义的master节点的名称
    - `<username>`：认证用户名称

- **配置主观宕机时间**

    当某个Sentinel实例给master发送`PING`命令后，如果指定的时间内未收到响应，则主观上认为master宕机，如果超过一定数量的Sentinel主观上认为master宕机，那么master客观宕机，此时Sentinel集群会切换master。

    ```bash
    sentinel down-after-milliseconds <master-name> <milliseconds>
    ```

    - `<master-name>`：`sentinel monitor`配置中定义的master节点的名称
    - `<milliseconds>`：未响应的时间，默认30秒

- **配置Sentinel实例访问密码**

    需要注意的是，如果给Sentinel配置了访问密码，那么Sentinel实例访问其它所有Sentinel实例时，会以同样的密码尝试去认真，因此所有的Sentinel都需要配置一样的密码。

    ```bash
    requirepass <password>
    ```

- **配置故障转移期间多少个slave进行数据同步**

    ```bash
    sentinel parallel-syncs <master-name> <numreplicas>
    ```

    - `<master-name>`：`sentinel monitor`配置中定义的master节点的名称
    - `<numreplicas>`：进行数据同步的slave的数量。这个数字越小，完成故障转移的时间就越长；如果数字越大，意味着会有多个slave在故障转移期间因为数据同步而不可用。可以通过设置值为1（默认1）来保证每次只有一个slave处于不能处理命令请求的状态。

- **故障转移超时时间**

    ```bash
    sentinel failover-timeout <master-name> <milliseconds>
    ```

    - `<master-name>`：`sentinel monitor`配置中定义的master节点的名称
    - `<milliseconds>`：超时时间，默认180s。

    `failover-timeout`有多种用途：

    - 在给定的Sentinel已对同一主机尝试上一次故障转移后，重新启动故障转移的超时时间是`failover-timeout`的两倍。
    - slave根据Sentinel当前配置在master节点宕机后，到同步新的master数据的时间（从Sentinel检测到错误配置的那一刻起计算）。
    - 取消已在进行但未产生任何配置更改的故障切换所需的时间（`SLAVEOF no one`尚未得到升级slave的确认）。
    - 正在进行的故障切换等待所有slave重新配置为新master副本的最长时间。然而，即使在这段时间之后，slave仍将由哨兵重新配置，但不会按照规定的精确并行同步进程进行配置。

    如果以上几种情况的执行时间超过了`failover-timeout`，那么这次故障转移就会被标记为失败。

- **配置通知脚本**

    当有任何警告⚠️级别的时间发生时（比如说redis实例的主观失效和客观失效等等），将会去调用这个脚本，这时这个脚本应该通过邮件，SMS等方式去通知系统管理员关于系统不正常运行的信息。调用该脚本时，将传给脚本两个参数，一个是事件的类型，一个是事件的描述。如果sentinel.conf配置文件中配置了这个脚本路径，那么必须保证这个脚本存在于这个路径，并且是可执行的，否则sentinel无法正常启动成功。

    ```bash
    sentinel notification-script <master-name> <script-path>
    ```

    客户端重写：

    ```bash
    sentinel client-reconfig-script <master-name> <script-path>
    ```

    拒绝客户端重写：

    ```bash
    sentinel deny-scripts-reconfig yes
    ```

#### 脑裂问题

[Redis 的脑裂现象和解决方案](https://blog.csdn.net/j1231230/article/details/121500055)

### **Redis 集群（cluster 模式）**

主从和哨兵模式可以解决高可用、高并发读的问题，但是仍然有两个问题没有解决：

- 海量数据存储问题
- 高并发写问题

Redis 集群（包括很多小集群）实现了对 Redis 的水平扩容，即启动 N 个 redis 节点，将整个数据库分布存储在这 N 个节点中。

- 集群中有多个master，每个master保存不同的数据；
- 每个master都可以有多个slave节点；
- master之间通过`PING`命令检测彼此的健康状态；
- 客户端请求可以访问集群任意节点，最终都会被转发到正确的节点。

____

**Hash Slot 哈希插槽**

Redis会把每一个master节点映射到0~16383共16384个哈希插槽上，也就是说每个master节点会负责一部分的哈希槽。

集群模式下，Redis中数据的key不与节点绑定，而是与插槽绑定。redis会根据key的有效部分计算插槽值，分两种情况：

- key中包含"{}"，且“{}”中至少包含一个字符，“{}”中的部分是有效部分；
- key中不包含"{}"，则整个key是有效部分。

具体的过程就是，redis会根据CRC16算法对key的有效部分计算hash，然后对16384取余，得到的结果就是slot的值，根据这个值就能找到对应的master节点。

**集群伸缩**

可以动态的添加或者删除节点。增加或删除节点后可以通过redis有关集群的命令将哈希槽重新分配以达到集群动态伸缩的目的。

**集群故障转移**

当集群中如果有一半以上的节点任意一个master宕机，就会让它的slave上升为master。

**集群进入fail状态**

- 如果某个master节点和它所有的slave都挂掉了，那么集群就会进入fail状态。
- 如果集群超过半数以上master挂掉了，那么无论是否有slave，集群都会进去fail状态。
- 如果集群任意master挂掉，且当前master没有slave，集群进入fail状态。

____


[^ref]: [【尚硅谷】Redis 6 入门到精通 超详细 教程](https://www.bilibili.com/video/BV1Rv41177Af)           [黑马程序员Redis入门到实战教程，全面透析redis底层原理+redis分布式锁+企业解决方案+redis实战](https://www.bilibili.com/video/BV1cr4y1671t)
[^1]: [Redis数据类型及编码格式——介绍及String篇](https://blog.csdn.net/qq_33983753/article/details/123004078)
[^2]: [Redis数据结构(六)-压缩列表ziplist](https://blog.csdn.net/weixin_38405646/article/details/120569304)
[^3]: [菜鸟教程](https://www.runoob.com/redis/redis-hyperloglog.html)
[^4]: [初识Redis的数据类型HyperLogLog](https://baijiahao.baidu.com/s?id=1669618381679082000&wfr=spider&for=pc)

