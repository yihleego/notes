# Redis

源码基于[Redis 7.0](https://github.com/redis/redis/blob/7.0/)版本。

## 数据类型

### redisObject

在 Redis 中存在一个名为 redisObject 的结构体，此结构基本上可以表示所有基本的Redis数据类型，如：strings、lists、sets、sorted sets等。

它有一个`type`字段，表示给定对象的类型，`encodeing`字段表示对象的储存方式，还有一个`refcount`，这样就可以在多个位置引用同一个对象，而无需多次分配它。
最后，`ptr`字段指向对象的实际表示，根据所使用的编码，即使对于相同的类型，也可能有所不同。

[server.h#redisObject](https://github.com/redis/redis/blob/7.0/src/server.h#L845)

```c
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    /* LRU time (relative to global lru_clock) or
     * LFU data (least significant 8 bits frequency
     * and most significant 16 bits access time). */
    unsigned lru:LRU_BITS; 
    int refcount;
    void *ptr;
} robj;
```

[server.h#object](https://github.com/redis/redis/blob/7.0/src/server.h#L638)

```c
/* The actual Redis Object */
#define OBJ_STRING 0    /* String object. */
#define OBJ_LIST 1      /* List object. */
#define OBJ_SET 2       /* Set object. */
#define OBJ_ZSET 3      /* Sorted set object. */
#define OBJ_HASH 4      /* Hash object. */
```

[server.h#encoding](https://github.com/redis/redis/blob/7.0/src/server.h#L825)

```c
/* Objects encoding. Some kind of objects like Strings and Hashes can be
 * internally represented in multiple ways. The 'encoding' field of the object
 * is set to one of this fields for this object. */
#define OBJ_ENCODING_RAW 0     /* Raw representation */
#define OBJ_ENCODING_INT 1     /* Encoded as integer */
#define OBJ_ENCODING_HT 2      /* Encoded as hash table */
#define OBJ_ENCODING_ZIPMAP 3  /* No longer used: old hash encoding. */
#define OBJ_ENCODING_LINKEDLIST 4 /* No longer used: old list encoding. */
#define OBJ_ENCODING_ZIPLIST 5 /* No longer used: old list/hash/zset encoding. */
#define OBJ_ENCODING_INTSET 6  /* Encoded as intset */
#define OBJ_ENCODING_SKIPLIST 7  /* Encoded as skiplist */
#define OBJ_ENCODING_EMBSTR 8  /* Embedded sds string encoding */
#define OBJ_ENCODING_QUICKLIST 9 /* Encoded as linked list of listpacks */
#define OBJ_ENCODING_STREAM 10 /* Encoded as a radix tree of listpacks */
#define OBJ_ENCODING_LISTPACK 11 /* Encoded as a listpack */
```

### 字符串 String

String 是最基本的 Redis 数据类型，它是二进制安全的，这意味着它可以包含任何类型的数据，例如：JPEG 图像或序列化的对象。
字符串值的最大长度为`512MB`。

字符串类型的数据结构存储方式有三种：`int`、`embstr`、`raw`。

- `int`:
  若存储数据的类型是整数，例如`123`这样的数据，就会使用`int`的存储方式进行存储，此时 redisObject 的`type`的值为`OBJ_STRING`，
  `encoding`的值为`OBJ_ENCODING_INT`。数据修改后类型不再是整数或者长度超过2^63-1时，会将`int`编码修改为`raw`编码。

- `embstr`:
  若存储数据的类型是字符串，且长度小于等于`44`个字节，就会使用`embstr`的存储方式进行存储，此时 redisObject 的`type`的值为`OBJ_STRING`，
  `encoding`的值为`OBJ_ENCODING_EMBSTR`。

- `raw`:
  若存储数据的类型是字符串，且长度大于`44`个字节，就会使用`raw`的存储方式进行存储，此时 redisObject 的`type`的值为`OBJ_STRING`，
  `encoding`的值为`OBJ_ENCODING_RAW`。

#### SDS (Simple Dynamic Strings) 简单动态字符串

`embstr`和`raw`相同点：

- `embstr`和`raw`都使用 redisObject 和 sdshdr 来表示字符串对象。

`embstr`和`raw`不相同点：

- `embstr`会申请一块连续的内存，使得 redisObject 和 sdshdr 处于同一个连续的空间，所以在释放内存时 redisObject 和 sdshdr 会被同时释放。
- `embstr`字符串是不可修改的，所以修改字符串会重新申请内存。
- `raw`会分别为 redisObject 和 sdshdr 申请内存，所以内存不一定是连续的。
- `raw`字符串是可修改的，当修改后的长度溢出时，会为 sdshdr 重新申请内存。

`SDS`数据结构如下：

```c
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; /* used */
    uint64_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
```

`SDS`与C语言的字符串相比有以下几点优势：

| SDS | C语言字符串 |
|:----|:-----------|
| 获取字符串长度读取`len`的值即可，时间复杂度变为`O(1)` | 不记录字符串长度，因此需要每次遍历，时间的复杂度是`O(n)` |
| 提供`空间预分配`和`惰性空间释放`两种策略在为字符串分配空间时，分配的空间比实际多，<br/>这样就能减少连续执行字符串增长带来内存重新分配的次数。<br/>当字符串被缩短的时候，也不会立即回收不适用的空间，等后面使用的时候再释放 | 固定空间 |
| 拼接字符串前会根据`len`的值判断是否满足条件，若是空间不够会先进行扩容操作 | 字符串拼接时若是没有分配足够长度的内存空间会出现缓冲区溢出的情况 |
| 是二进制安全，除了可以储存字符串以外还可以储存二进制文件（如：图片、音频、视频等） | 字符串是以空字符串作为结束符，文件中可能含有结束符，因此不是二进制安全的 |

---

#### 为什么限制以44个字节为临界值？

Redis 为`type`是`embstr`的 redisObject 分配内存时，会申请64个字节，
在 Redis 3.2 之前的版本中，根据源码`sizeof(robj)+sizeof(struct sdshdr)+len+1`可得字符串实际可用长度为`64-16-8-1=39`。

_额外减`1`是因为字符串结束符`\0`占1个字节。_

```c
typedef struct redisObject {
    unsigned type:4;      // 0.5 字节
    unsigned encoding:4;  // 0.5 字节
    unsigned lru:LRU_BITS;// 3 字节
    int refcount;         // 4 字节
    void *ptr;            // 8 字节
} robj;                   // 共 16 字节
```

```c
// Redis < 3.2
struct sdshdr {
    unsigned int len; // 4 字节
    unsigned int free;// 4 字节
    char buf[];       // ? 字节
};                    // 共 8 字节
```

在 Redis 3.2 之后的版本中，对`sdshdr`结构体进行了优化，由原来的`8`字节变成了`3`字节，所以字符串实际可用长度为`64-16-3-1=44`

```c
// Redis >= 3.2
struct sdshdr8 {
    uint8_t len;         // 1 字节
    uint8_t alloc;       // 1 字节
    unsigned char flags; // 1 字节
    char buf[];          // ? 字节
};                       // 共 3 字节
```

_实际源码中直接定义了`embstr`临界值值，而非通过计算获得。_

[object.c#OBJ_ENCODING_EMBSTR_SIZE_LIMIT](https://github.com/redis/redis/blob/7.0/src/object.c#L119)

```c
/* Create a string object with EMBSTR encoding if it is smaller than
 * OBJ_ENCODING_EMBSTR_SIZE_LIMIT, otherwise the RAW encoding is
 * used.
 *
 * The current limit of 44 is chosen so that the biggest string object
 * we allocate as EMBSTR will still fit into the 64 byte arena of jemalloc. */
#define OBJ_ENCODING_EMBSTR_SIZE_LIMIT 44
```

总结：在 Redis 3.2 版本的之前是以`39`为界限，之后的版本是以`44`为界限。

### 列表 List

List 用于存储字符串，按插入顺序排序，可以将新元素添加到列表的头部（左侧）或尾部（右侧）。

在 Redis 3.2 之前的版本的列表是使用`ziplist`和`linkedlist`进行实现的。

- 当列表元素个数比较少并且每个元素占用空间比较小的时候使用`ziplist`。
- 当列表元素个数比较多或者某个元素占用空间比较大的时候使用`linkedlist`。

在 Redis 3.2 之后的版本使用了`quicklist`代替了`ziplist`和`linkedlist`，原因是`linkedlist`的每节点附加空间相对太高，
例如每个节点的`prev`和`next`指针占`16`个字节（64位系统的指针占`8`个字节），且分别为每个节点申请内存，导致内存碎片化，进而影响内存管理效率。

实际上，`quicklist`是以`ziplist`为节点的链表，将链表按段切分，每一段使用内存连续的`ziplist`进行存储，多个`ziplist`通过`prev`和`next`指针组成的双向链表。
它结合了`ziplist`和`linkedlist`的优势，压缩了内存的使用量，进一步提高了性能。

#### linkedlist 链表

源码：[adlist.h](https://github.com/redis/redis/blob/7.0/src/adlist.h#L47)

![linkedlist](images/redis_linkedlist.jpg)

#### ziplist 压缩列表

源码：[ziplist.c](https://github.com/redis/redis/blob/7.0/src/ziplist.c)

![ziplist](images/redis_ziplist.jpg)

#### quicklist 快速列表

源码：[quicklist.h](https://github.com/redis/redis/blob/7.0/src/quicklist.h#L105)

![quicklist](images/redis_quicklist.jpg)

既然`quicklist`本质上是将`ziplist`连接起来，那么每个`ziplist`存放多少的元素比较合适呢？

- 太少：起不到应有的作用，若每个`ziplist`只储存`1`个元素，就退化成了`linkedlist`。
- 太多：性能差，若当前`quicklist`只存在`1`个`ziplist`，就退化成了`ziplist`。

Redis 默认配置的每个`ziplist`的大小为`8Kb`，超过这个大小时会创建一个新的`ziplist`。
该大小可以通过修改`redis.conf`文件的`list-max-ziplist-size`配置来调整。

[redis.conf#list-max-listpack-size](https://github.com/redis/redis/blob/7.0/redis.conf#L1929)

```bash
# Lists are also encoded in a special way to save a lot of space.
# The number of entries allowed per internal list node can be specified
# as a fixed maximum size or a maximum number of elements.
# For a fixed maximum size, use -5 through -1, meaning:
# -5: max size: 64 Kb  <-- not recommended for normal workloads
# -4: max size: 32 Kb  <-- not recommended
# -3: max size: 16 Kb  <-- probably not recommended
# -2: max size: 8 Kb   <-- good
# -1: max size: 4 Kb   <-- good
# Positive numbers mean store up to _exactly_ that number of elements
# per list node.
# The highest performing option is usually -2 (8 Kb size) or -1 (4 Kb size),
# but if your use case is unique, adjust the settings as necessary.
list-max-listpack-size -2
```

为了进一步节约内存，还可以使用`LZF`压缩算法对`ziplist`进行压缩存储。
若一个`ziplist`被压缩，那么从中读取数据前需要先解压，因此性能会有所下降。

- 压缩深度为`0`时，不会对任何节点进行压缩。
- 压缩深度为`1`时，`quicklist`的`head`和`tail`节点不会被压缩，这样可以有效地提高`push`和`pop`操作的性能。
- 压缩深度为`2`时，`quicklist`的首尾各`2`个节点（共`4`个）不会被压缩。
- 压缩深度为`3`时，`quicklist`的首尾各`3`个节点（共`6`个）不会被压缩。
- 以此类推。

压缩深度可以通过修改`redis.conf`文件的`list-compress-depth`配置来调整。
默认值为`0`，即不启用压缩功能。

[redis.conf#list-compress-depth](https://github.com/redis/redis/blob/7.0/redis.conf#L1945)

```bash
# Lists may also be compressed.
# Compress depth is the number of quicklist ziplist nodes from *each* side of
# the list to *exclude* from compression.  The head and tail of the list
# are always uncompressed for fast push/pop operations.  Settings are:
# 0: disable all list compression
# 1: depth 1 means "don't start compressing until after 1 node into the list,
#    going from either the head or tail"
#    So: [head]->node->node->...->node->[tail]
#    [head], [tail] will always be uncompressed; inner nodes will compress.
# 2: [head]->[next]->node->node->...->node->[prev]->[tail]
#    2 here means: don't compress head or head->next or tail->prev or tail,
#    but compress all nodes between them.
# 3: [head]->[next]->[next]->node->node->...->node->[prev]->[prev]->[tail]
# etc.
list-compress-depth 0
```

#### LZF 压缩算法

_TODO_

### 集合 Set

Set 与 List 类似，都可以用于存储字符串。不同的是，List 可以存储相同的字符串，是有序的；而 Set 不允许存储重复的字符串，是无序的。

Set 底层实现是基于`hash table`和`intset`实现的，分别对应`encoding`为`OBJ_ENCODING_HT`和`OBJ_ENCODING_INTSET`。

#### 整数集合 intset

`intset`用于保存整数类型的数据结构，源码如下：

```c
/* Note that these encodings are ordered, so:
 * INTSET_ENC_INT16 < INTSET_ENC_INT32 < INTSET_ENC_INT64. */
#define INTSET_ENC_INT16 (sizeof(int16_t))
#define INTSET_ENC_INT32 (sizeof(int32_t))
#define INTSET_ENC_INT64 (sizeof(int64_t))

typedef struct intset {
    uint32_t encoding;
    uint32_t length;
    int8_t contents[];
} intset;
```

结构体中，定义了三个属性：

- `encoding`：编码方式，支持`int16_t`、`int32_t`和`int64_t`整数类型
- `length`：集合长度
- `contents`：储存元素的数组

因为`intset`是有序的，所以通过对比`contents`中首尾两个数（即最小值和最大值）便可以初步判断元素否存在。然后通过二分法在数组中可以快速定位元素的位置。

应用场景：去重、抽奖、点赞、关注

#### hash table

![Hash 哈希](#Hash-哈希)

### 有序集合 Sorted Set

Sorted Set 与 Set 类似，都不允许存储重复的字符串。不同的是，Sorted Set 是有序的。

Sorted Set 底层实现是基于`skiplist`和`ziplist`实现的，对应`encoding`为`OBJ_ENCODING_SKIPLIST`和`OBJ_ENCODING_ZIPLIST`。

#### skiplist

`skiplist`是一种有序的数据结构，它通过每一个节点维持多个指向其它节点的指针，从而达到快速访问的目的。

相比一般的链表，有更高的查找效率，链表并不能像数组随机访问，只能从头一个个遍历。跳表为节点设置了快速访问的指针，可以跨越节点进行访问，这也是跳表的名字的含义。
跳表的性能可比拟二叉查找树，平均期望的查找、插入、删除时间复杂度都是`O(logn)`，许多知名的开源软件（库）中的数据结构均采用了跳表这种数据结构。

数据结构如下：

![redis_zset_skiplist](images/redis_zset_skiplist.png)

当插入一个数据时，随机生成这个节点的高度，每新增一层的概率为`p`，为了防止极端情况高度过高，会设定一个最大高度。

```java
int randomLevel() {
    int level = 1;
    while (rand() >= p)
        level++;
    return level > MAX_LEVEL ? MAX_LEVEL : level;
}
```

![redis_zset_skiplist_search](images/redis_zset_skiplist_search.png)

搜索时从最高的层级的节点开始查找，通过节点进行跳跃，而不是一个个的遍历。

#### 作者用 skiplist 而不用 Red–black tree 或 AVL tree 或 B-tree 的原因

1. `skiplist`不是很占用内存。这基本上取决于你。更改有关节点具有给定数量级别的概率的参数将使内存密集度低于`b-tree`。
2. `Sorted Set`往往是很多`ZRANGE`或`ZREVRANGE`操作的目标，即把`skiplist`作为链表遍历,此操作的缓存局部性至少与其他类型的平衡树一样好。
3. `skiplist`更易于实现、调试。例如，由于跳过列表的简单性，我们收到了一个补丁，实现了复杂度为`O(logn)`的`ZRANK`的扩展，只需要对代码进行少量更改。

我们的经验表明，Redis 主要受 I/O 限制。假设您的操作速度如此之快以至于可以使单个核心饱和，可以运行多个 Redis 实例，并使用 Redis Cluster 解决方案。

应用场景：排行榜、滑动窗口限流

### 哈希 Hash

Hash 底层实现是基于`hash table`和`ziplist`，`hash table`的实现原理类似于`HashMap`的底层原理。

### 位图 Bitmap

Bitmap 是用一个比特位来映射某个元素的状态。由于一个比特位只能表示 0 和 1 两种状态，所以 Bitmap 能映射的状态有限，但是使用比特位的优势是能大量的节省内存空间。
在 Redis 中，可以把 Bitmap 想象成一个以比特位为单位的数组，数组的每个单元只能存储0和1，数组的下标在 Bitmap 中叫做偏移量。

![redis_bitmap](images/redis_bitmap.png)

应用场景：用户签到、统计数量、用户在线状态、实现布隆过滤器

### HyperLogLog

HyperLogLog 是用来做基数统计的算法，HyperLogLog 的优点是，在输入元素的数量或者体积非常非常大时，计算基数所需的空间总是固定的、并且是很小的。

在 Redis 里面，每个 HyperLogLog 键只需要花费 12 KB 内存，就可以计算接近 2^64 个不同元素的基数。这和计算基数时，元素越多耗费内存就越多的集合形成鲜明对比。
但是，因为 HyperLogLog 只会根据输入元素来计算基数，而不会储存输入元素本身，所以 HyperLogLog 不能像集合那样，返回输入的各个元素。

### Stream

TODO

### Geospatial indexes

GEO 主要用于存储地理位置信息，并对存储的信息进行操作。

## 命令

### 键 key

- `DEL key`：该命令用于在 key 存在时删除 key
- `DUMP key`：序列化给定 key ，并返回被序列化的值
- `EXISTS key`：检查给定 key 是否存在
- `EXPIRE key seconds`：为给定 key 设置过期时间，以秒计
- `EXPIREAT key timestamp`：EXPIREAT 的作用和 EXPIRE 类似，都用于为 key 设置过期时间。 不同在于 EXPIREAT 命令接受的时间参数是 UNIX 时间戳(unix timestamp)
- `PEXPIRE key milliseconds`：设置 key 的过期时间以毫秒计
- `PEXPIREAT key milliseconds-timestamp`：设置 key 过期时间的时间戳(unix timestamp) 以毫秒计
- `KEYS pattern`：查找所有符合给定模式( pattern)的 key
- `MOVE key db`：将当前数据库的 key 移动到给定的数据库 db 当中
- `PERSIST key`：移除 key 的过期时间，key 将持久保持
- `PTTL key`：以毫秒为单位返回 key 的剩余的过期时间
- `TTL key`：以秒为单位，返回给定 key 的剩余生存时间(TTL, time to live)
- `RANDOMKEY`：从当前数据库中随机返回一个 key
- `RENAME key newkey`：修改 key 的名称
- `RENAMENX key newkey`：仅当 newkey 不存在时，将 key 改名为 newkey
- `SCAN cursor [MATCH pattern] [COUNT count]`：迭代数据库中的数据库键
- `TYPE key`：返回 key 所储存的值的类型

### 字符串 String

- `SET key value`：设置指定 key 的值
- `GET key`：获取指定 key 的值
- `GETRANGE key start end`：返回 key 中字符串值的子字符
- `GETSET key value`：将给定 key 的值设为 value ，并返回 key 的旧值(old value)
- `GETBIT key offset`：对 key 所储存的字符串值，获取指定偏移量上的位(bit)
- `MGET key1 [key2..]`：获取所有(一个或多个)给定 key 的值
- `SETBIT key offset value`：对 key 所储存的字符串值，设置或清除指定偏移量上的位(bit)
- `SETEX key seconds value`：将值 value 关联到 key ，并将 key 的过期时间设为 seconds (以秒为单位)
- `SETNX key value`：只有在 key 不存在时设置 key 的值
- `SETRANGE key offset value`：用 value 参数覆写给定 key 所储存的字符串值，从偏移量 offset 开始
- `STRLEN key`：返回 key 所储存的字符串值的长度
- `MSET key value [key value ...]`：同时设置一个或多个 key-value 对
- `MSETNX key value [key value ...]`：同时设置一个或多个 key-value 对，当且仅当所有给定 key 都不存在
- `PSETEX key milliseconds value`：这个命令和 SETEX 命令相似，但它以毫秒为单位设置 key 的生存时间，而不是像 SETEX 命令那样，以秒为单位
- `INCR key`：将 key 中储存的数字值增一
- `INCRBY key increment`：将 key 所储存的值加上给定的增量值（increment）
- `INCRBYFLOAT key increment`：将 key 所储存的值加上给定的浮点增量值（increment）
- `DECR key`：将 key 中储存的数字值减一
- `DECRBY key decrement`：key 所储存的值减去给定的减量值（decrement）
- `APPEND key value`：如果 key 已经存在并且是一个字符串， APPEND 命令将指定的 value 追加到该 key 原来值（value）的末尾。

### 列表 List

- `BLPOP key1 [key2 ] timeout`：移出并获取列表的第一个元素， 如果列表没有元素会阻塞列表直到等待超时或发现可弹出元素为止
- `BRPOP key1 [key2 ] timeout`：移出并获取列表的最后一个元素， 如果列表没有元素会阻塞列表直到等待超时或发现可弹出元素为止
- `BRPOPLPUSH source destination timeout`：从列表中弹出一个值，将弹出的元素插入到另外一个列表中并返回它； 如果列表没有元素会阻塞列表直到等待超时或发现可弹出元素为止
- `LINDEX key index`：通过索引获取列表中的元素
- `LINSERT key BEFORE|AFTER pivot value`：在列表的元素前或者后插入元素
- `LLEN key`：获取列表长度
- `LPOP key`：移出并获取列表的第一个元素
- `LPUSH key value1 [value2]`：将一个或多个值插入到列表头部
- `LPUSHX key value`：将一个值插入到已存在的列表头部
- `LRANGE key start stop`：获取列表指定范围内的元素
- `LREM key count value`：移除列表元素
- `LSET key index value`：通过索引设置列表元素的值
- `LTRIM key start stop`：对一个列表进行修剪(trim)，就是说，让列表只保留指定区间内的元素，不在指定区间之内的元素都将被删除
- `RPOP key`：移除列表的最后一个元素，返回值为移除的元素
- `RPOPLPUSH source destination`：移除列表的最后一个元素，并将该元素添加到另一个列表并返回
- `RPUSH key value1 [value2]`：在列表中添加一个或多个值
- `RPUSHX key value`：为已存在的列表添加值

### 集合 Set

- `SADD key member1 [member2]`：向集合添加一个或多个成员
- `SCARD key`：获取集合的成员数
- `SDIFF key1 [key2]`：返回第一个集合与其他集合之间的差异
- `SDIFFSTORE destination key1 [key2]`：返回给定所有集合的差集并存储在 destination 中
- `SINTER key1 [key2]`：返回给定所有集合的交集
- `SINTERSTORE destination key1 [key2]`：返回给定所有集合的交集并存储在 destination 中
- `SISMEMBER key member`：判断 member 元素是否是集合 key 的成员
- `SMEMBERS key`：返回集合中的所有成员
- `SMOVE source destination member`：将 member 元素从 source 集合移动到 destination 集合
- `SPOP key`：移除并返回集合中的一个随机元素
- `SRANDMEMBER key [count]`：返回集合中一个或多个随机数
- `SREM key member1 [member2]`：移除集合中一个或多个成员
- `SUNION key1 [key2]`：返回所有给定集合的并集
- `SUNIONSTORE destination key1 [key2]`：所有给定集合的并集存储在 destination 集合中
- `SSCAN key cursor [MATCH pattern] [COUNT count]`：迭代集合中的元素

### 有序集合 Sorted Set

- `ZADD key score1 member1 [score2 member2]`：向有序集合添加一个或多个成员，或者更新已存在成员的分数
- `ZCARD key`：获取有序集合的成员数
- `ZCOUNT key min max`：计算在有序集合中指定区间分数的成员数
- `ZINCRBY key increment member`：有序集合中对指定成员的分数加上增量 increment
- `ZINTERSTORE destination numkeys key [key ...]`：计算给定的一个或多个有序集的交集并将结果集存储在新的有序集合 destination 中
- `ZLEXCOUNT key min max`：在有序集合中计算指定字典区间内成员数量
- `ZRANGE key start stop [WITHSCORES]`：通过索引区间返回有序集合指定区间内的成员
- `ZRANGEBYLEX key min max [LIMIT offset count]`：通过字典区间返回有序集合的成员
- `ZRANGEBYSCORE key min max [WITHSCORES] [LIMIT]`：通过分数返回有序集合指定区间内的成员
- `ZRANK key member`：返回有序集合中指定成员的索引
- `ZREM key member [member ...]`：移除有序集合中的一个或多个成员
- `ZREMRANGEBYLEX key min max`：移除有序集合中给定的字典区间的所有成员
- `ZREMRANGEBYRANK key start stop`：移除有序集合中给定的排名区间的所有成员
- `ZREMRANGEBYSCORE key min max`：移除有序集合中给定的分数区间的所有成员
- `ZREVRANGE key start stop [WITHSCORES]`：返回有序集中指定区间内的成员，通过索引，分数从高到低
- `ZREVRANGEBYSCORE key max min [WITHSCORES]`：返回有序集中指定分数区间内的成员，分数从高到低排序
- `ZREVRANK key member`：返回有序集合中指定成员的排名，有序集成员按分数值递减(从大到小)排序
- `ZSCORE key member`：返回有序集中，成员的分数值
- `ZUNIONSTORE destination numkeys key [key ...]`：计算给定的一个或多个有序集的并集，并存储在新的 key 中
- `ZSCAN key cursor [MATCH pattern] [COUNT count]`：迭代有序集合中的元素（包括元素成员和元素分值）

### 哈希 Hash

- `HDEL key field1 [field2]`：删除一个或多个哈希表字段
- `HEXISTS key field`：查看哈希表 key 中，指定的字段是否存在
- `HGET key field`：获取存储在哈希表中指定字段的值
- `HGETALL key`：获取在哈希表中指定 key 的所有字段和值
- `HINCRBY key field increment`：为哈希表 key 中的指定字段的整数值加上增量 increment
- `HINCRBYFLOAT key field increment`：为哈希表 key 中的指定字段的浮点数值加上增量 increment
- `HKEYS key`：获取所有哈希表中的字段
- `HLEN key`：获取哈希表中字段的数量
- `HMGET key field1 [field2]`：获取所有给定字段的值
- `HMSET key field1 value1 [field2 value2 ]`：同时将多个 field-value (域-值)对设置到哈希表 key 中
- `HSET key field value`：将哈希表 key 中的字段 field 的值设为 value
- `HSETNX key field value`：只有在字段 field 不存在时，设置哈希表字段的值
- `HVALS key`：获取哈希表中所有值
- `HSCAN key cursor [MATCH pattern] [COUNT count]`：迭代哈希表中的键值对

### 位图 Bitmap

- `SETBIT key offset value`：设置 value
- `GETBIT key offset`：获取 value
- `BITCOUNT key [start] [end]`：获取指定范围内值为 1 的个数
- `BITOP [operations] [result] [key ...]`：运算操作
- `BITPOS key value`：获取第一次出现 value 的位置

### HyperLogLog

- `PFADD key element [element ...]`：添加指定元素到 HyperLogLog 中。
- `PFCOUNT key [key ...]`：返回给定 HyperLogLog 的基数估算值。
- `PFMERGE destkey sourcekey [sourcekey ...]`：将多个 HyperLogLog 合并为一个 HyperLogLog

## 过期策略

Redis 的过期策略采用定期删除和惰性删除结合的方式。

### 定期删除

默认配置每隔 100 毫秒随机抽取一些设置过期时间的`key`，发现过期就删除。该操作是一种主动操作。

### 惰性删除

客户端操作某个指定的`key`时，先判断该的`key`是否过期，如果已过期则删除且返回空，如果没过期则返回数据给客户端。

## 持久化

### RDB Redis Database

以指定的时间间隔将内存中的数据集以快照形式写入磁盘。

默认情况下，Redis 将数据集的快照保存在磁盘上的一个名为`dump.rdb`的二进制文件中。

Redis提供了`SAVE`或`BGSAVE`命令来创建 RDB 文件

- `SAVE`：会阻塞Redis服务器进程，直到 RDB 文件创建完毕为止，在服务器进程阻塞期间，服务器不能处理任何命令请求。
- `BGSAVE`（默认）：会派生出一个子进程，然后由子进程负责创建RDB文件，服务器进程（父进程）继续处理命令请求。

我们可以通过 Redis 服务器配置文件 的`save`选项设置多个保存条件，只要其中任意一个条件被满足，服务器就会执行`BGSAVE`命令，默认设置如下：

```text
# 在 900 秒内，至少进行了1次修改
save 900 1
# 在 300 秒内，至少进行了10次修改
save 300 10
# 在 60 秒内，至少进行了10000次修改
save 60 10000
```

在新版本中已经将`save`选项改为了一行配置：

```text
# 在 3600 秒内，至少进行了1次修改
# 在 300 秒内，至少进行了100次修改
# 在 60 秒内，至少进行了10000次修改
save 3600 1 300 100 60 10000
```

#### 优点

RDB 是 Redis 数据的一个非常紧凑的单文件。

RDB 文件非常适合备份。例如，每小时归档一次 RDB 文件，并在 30 天内每天保存一个 RDB 快照。这使得可以在发生灾难时轻松恢复不同版本的数据集。

RDB 最大限度地提高了 Redis 的性能，因为 Redis 父进程为了持久化而需要做的唯一工作就是派生一个将完成所有其余工作的子进程。父进程永远不会执行磁盘 I/O 或类似操作。

#### 缺点

如果您需要在 Redis 由于任何原因在没有正确关闭的情况下停止工作时（例如断电后）将数据丢失的可能性降到最低，那么 RDB 并不适合。
可以配置生成 RDB 的不同保存点。例如，在对数据集至少 5 分钟和 100 次写入之后。

RDB 需要经常`fork()`以便使用子进程在磁盘上持久化。如果数据集很大，`fork()`可能会很耗时，并且如果数据集很大并且 CPU 性能不是很好，可能会导致 Redis 停止为客户端服务几毫秒甚至一秒钟。

### AOF Append Only File

以日志的形式记录服务器收到的每个写入操作，将在服务器启动时再次播放，重建原始数据集。

Redis 服务器会根据配置文件中`appendfsync`选项来决定何时将 AOF 缓冲区中的内容写入文件中：

- `always`：每次将新命令附加到 AOF 时调用 fsync。效率最差，安全性最好。
- `everysec`（默认）：每秒调用 fsync。性能较好，如果发生故障，最多会丢失 1 秒的数据。
- `no`：永远不调用 fsync，只是把数据交给操作系统，通常，Linux 将使用此配置每 30 秒刷新一次数据。性能最佳，但是安全性最差。

#### 优点

使用 AOF Redis 更加持久：可以有不同的 fsync 策略：不使用 fsync、每秒 fsync、每次查询时 fsync。使用每秒 fsync 的默认策略，写入性能仍然很棒。
fsync 是使用后台线程执行的，当没有 fsync 正在进行时，主线程将执行写入，因此最差的情况会丢失一秒钟的数据。

AOF 日志是一个仅附加日志，因此不会出现寻道问题，也不会在断电时出现损坏问题。即使由于某种原因（磁盘已满或其他原因）日志以写一半的命令结束，redis-check-aof 工具也能够轻松修复它。

当 AOF 变得太大时，Redis 能够在后台自动重写 AOF。重写是完全安全的，因为当 Redis 继续附加到旧文件时，会使用创建当前数据集所需的最少操作集生成一个全新的文件，一旦第二个文件准备就绪，Redis 就会切换两者并开始附加到新的那一个。

AOF 以易于理解和解析的格式依次包含所有操作的日志。甚至可以轻松导出 AOF 文件。

#### 缺点

对于相同数量的数据集而言，AOF 文件通常要大于 RDB 文件。RDB 在恢复大数据集时的速度比 AOF 的恢复速度要快。

根据同步策略的不同，AOF 在运行效率上往往会慢于 RDB。

## 高可用

## 一致性

## 分布式锁

### Redission

### RedLock

### 看门狗

## 参考链接

- [Redis data types](https://redis.io/docs/manual/data-types/)
