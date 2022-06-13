# Redis

源码基于[Redis 7.0](https://github.com/redis/redis/blob/7.0/)版本。

## 数据类型

### redisObject

在 Redis 中存在一个名为 redisObject 的结构体，此结构基本上可以表示所有基本的Redis数据类型，如：strings、lists、sets、sorted sets等。

它有一个`type`字段，表示给定对象的类型，`encodeing`字段表示对象的储存方式，还有一个`refcount`，这样就可以在多个位置引用同一个对象，而无需多次分配它。
最后，`ptr`字段指向对象的实际表示，根据所使用的编码，即使对于相同的类型，也可能有所不同。

[server.h#redisObject](https://github.com/redis/redis/blob/7.0/src/server.h#L845)

```
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
#define OBJ_ENCODING_ZIPMAP 3  /* Encoded as zipmap */
#define OBJ_ENCODING_LINKEDLIST 4 /* No longer used: old list encoding. */
#define OBJ_ENCODING_ZIPLIST 5 /* Encoded as ziplist */
#define OBJ_ENCODING_INTSET 6  /* Encoded as intset */
#define OBJ_ENCODING_SKIPLIST 7  /* Encoded as skiplist */
#define OBJ_ENCODING_EMBSTR 8  /* Embedded sds string encoding */
#define OBJ_ENCODING_QUICKLIST 9 /* Encoded as linked list of listpacks */
#define OBJ_ENCODING_STREAM 10 /* Encoded as a radix tree of listpacks */
#define OBJ_ENCODING_LISTPACK 11 /* Encoded as a listpack */
```

_另外，在未来的版本中，`zipmap`和`ziplist`将被弃用。_ [unstable/server.h#encoding](https://github.com/redis/redis/blob/unstable/src/server.h#L831)

```c
#define OBJ_ENCODING_ZIPMAP 3  /* No longer used: old hash encoding. */
#define OBJ_ENCODING_LINKEDLIST 4 /* No longer used: old list encoding. */
#define OBJ_ENCODING_ZIPLIST 5 /* No longer used: old list/hash/zset encoding. */
```

### 0. Strings 字符串

Strings 是最基本的 Redis 数据类型。
Redis 字符串是二进制安全的，这意味着 Redis 字符串可以包含任何类型的数据，例如：JPEG 图像或序列化的对象。
字符串值的最大长度为`512Mb`。

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

```
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

### 1. Lists 列表

Redis 列表是简单的字符串列表，按插入顺序排序，可以将新元素添加到列表的头部（左侧）或尾部（右侧）。

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

```conf
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

```conf
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

### 2. Set 集合

### 3. Sorted Sets (Zset) 有序集合

### 4. Hashes 哈希

### Bitmaps

### HyperLogLogs

### Streams

### Geospatial indexes

## 分布式锁

### Redission

### RedLock

### 看门狗

## 参考链接

- [Redis data types](https://redis.io/docs/manual/data-types/)
