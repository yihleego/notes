# Redis

## 数据类型

### redisObject

在 Redis 中存在名为 redisObject 的结构体，此结构基本上可以表示所有基本的Redis数据类型，如：strings、lists、sets、sorted sets等。
它有一个`type`字段，这样就可以知道给定对象的类型，`encodeing`字段表示对象的储存方式，
还有一个`refcount`，这样就可以在多个位置引用同一个对象，而无需多次分配它。
最后，`ptr`字段指向对象的实际表示，即使对于相同的类型，也可能有所不同，具体取决于所使用的编码。

```
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; /* lru time (relative to server.lruclock) */
    int refcount;
    void *ptr;
} robj;
```

[redis/src/server.h#redisObject](https://github.com/redis/redis/blob/unstable/src/server.h#L848)

```c
/* The actual Redis Object */
#define OBJ_STRING 0    /* String object. */
#define OBJ_LIST 1      /* List object. */
#define OBJ_SET 2       /* Set object. */
#define OBJ_ZSET 3      /* Sorted set object. */
#define OBJ_HASH 4      /* Hash object. */
```

[redis/src/server.h#type](https://github.com/redis/redis/blob/unstable/src/server.h#L639)

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

[redis/src/server.h#encoding](https://github.com/redis/redis/blob/unstable/src/server.h#L825)

### Strings

Strings 是最基本的 Redis 数据类型。
Redis 字符串是二进制安全的，这意味着 Redis 字符串可以包含任何类型的数据，例如：JPEG 图像或序列化的对象。
字符串值的最大长度为 512MB。

字符串类型的数据结构存储方式有三种`int`、`embstr`、`raw`。

#### 1. int

若存储数据的类型是整数，例如`123`这样的数据，就会使用`int`的存储方式进行存储，此时 redisObject 的`type`的值为`OBJ_STRING`，
`encoding`的值为`OBJ_ENCODING_INT`，`ptr`指向数据所在的地址。

#### 2. embstr

若存储数据的类型是字符串，且长度小于等于`44`个字节，就会使用`embstr`的存储方式进行存储，此时 redisObject 的`type`的值为`OBJ_STRING`，
`encoding`的值为`OBJ_ENCODING_EMBSTR`，`ptr`指向数据所在的地址。

_注意：在Redis 3.2 版本的之前是以`39`为界限，之后的版本是以`44`为界限。_

```c
/* Create a string object with EMBSTR encoding if it is smaller than
 * OBJ_ENCODING_EMBSTR_SIZE_LIMIT, otherwise the RAW encoding is
 * used.
 *
 * The current limit of 44 is chosen so that the biggest string object
 * we allocate as EMBSTR will still fit into the 64 byte arena of jemalloc. */
#define OBJ_ENCODING_EMBSTR_SIZE_LIMIT 44
```

[redis/src/object.c#OBJ_ENCODING_EMBSTR_SIZE_LIMIT](https://github.com/redis/redis/blob/unstable/src/object.c#L113)

#### 3. SDS

若存储数据的类型是字符串，且长度大于`44`个字节，就会使用`SDS` (Simple Dynamic Strings) 的存储方式进行存储，此时 redisObject 的`type`的值为`OBJ_STRING`，
`encoding`的值为`OBJ_ENCODING_RAW`，`ptr`指向数据所在的地址。

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
| 提供`空间预分配`和`惰性空间释放`两种策略在为字符串分配空间时，分配的空间比实际多，这样就能减少连续执行字符串增长带来内存重新分配的次数。当字符串被缩短的时候，也不会立即回收不适用的空间，等后面使用的时候再释放 | 固定空间 |
| 拼接字符串前会根据`len`的值判断是否满足条件，若是空间不够会先进行扩容操作 | 字符串拼接时若是没有分配足够长度的内存空间会出现缓冲区溢出的情况 |
| 是二进制安全，除了可以储存字符串以外还可以储存二进制文件（如：图片、音频、视频等） | 字符串是以空字符串作为结束符，文件中可能含有结束符，因此不是二进制安全的 |

### Lists

### Set

### Hashes

### Sorted Sets

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
