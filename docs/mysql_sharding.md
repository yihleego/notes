# 分库分表

## 基因法

```python
db = 0


def next_id():
    global db
    db = db + 1
    return db


def hash(shards: int, user_id: int):
    if (shards & (shards - 1)) != 0:
        raise InvalidArgumentException
    return user_id & (shards - 1)


def gen_order_id(shards: int, user_id: int):
    idx = hash(shards, user_id)
    return (next_id() << (shards.bit_length() - 1)) | idx


if __name__ == '__main__':
    shards = 16

    for _ in range(1, 100):
        oid = gen_order_id(shards, 1)
        print(f"db_idx: {oid:020b}", oid)

    for user_id in range(1, 100):
        h = hash(16, user_id)
        print(f"hash: {h:04b}, {h}")

    for user_id in range(1, 100):
        oid = gen_order_id(shards, user_id)
        print(f"db_idx: {oid:020b}", oid)
```

订单表分片`16`，假设用户ID为`666`，二进制为`0b1010011010`，则定位表索引为`666 & (16 - 1)`，

```
0b1010011010 # 用户ID：666
0b0000001111 # 分片Mod：16 - 1
---
0b0000001010 # 表索引：10
```

生成下一个订单`next_id` << $\log_2{shards}$，然后再加上用户基因，即分片索引

```
0b0000000001 << 4
0b0000010000
---
0b0000010000 加用户基因
0b0000001010
0b0000011010 最终结果
```
