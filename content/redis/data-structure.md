---
title: 数据结构与底层实现
weight: 1
---

## 五大基本数据类型

### String

底层编码：
- `int`：整数值，直接存储
- `embstr`：短字符串（≤44字节），一次内存分配
- `raw`：长字符串（>44字节），两次内存分配

```bash
SET counter 100
INCR counter    # 原子自增
GET counter     # "101"
```

---

### Hash

底层编码：
- `listpack`（旧版 ziplist）：元素少且值小时使用，内存紧凑
- `hashtable`：元素多或值大时转换

```bash
HSET user:1 name "Alice" age 25
HGET user:1 name
HGETALL user:1
```

---

### List

底层编码：
- `listpack`：元素少时使用
- `quicklist`：双向链表 + listpack，兼顾性能和内存

```bash
LPUSH queue task1 task2  # 左侧入队
RPOP queue               # 右侧出队（实现队列）
LRANGE queue 0 -1        # 查看所有元素
```

---

### Set

底层编码：
- `listpack`：元素少且为整数时
- `hashtable`：元素多时

```bash
SADD tags "go" "redis" "mysql"
SISMEMBER tags "go"  # 判断成员
SUNION tags1 tags2   # 求并集
```

---

### ZSet（有序集合）

底层编码：
- `listpack`：元素少时
- `skiplist + hashtable`：元素多时

```bash
ZADD leaderboard 100 "Alice" 95 "Bob"
ZRANGE leaderboard 0 -1 WITHSCORES  # 按分数升序
ZREVRANK leaderboard "Alice"         # 排名（降序）
```

### Q: 跳表为什么比平衡树更适合 ZSet？

| 对比 | 平衡树（红黑树）| 跳表 |
|------|----------------|------|
| 范围查询 | 中序遍历，复杂 | 链表顺序扫描，简单 |
| 实现复杂度 | 旋转操作复杂 | 随机层数，简单 |
| 内存占用 | 较少 | 较多（多级指针）|
| 时间复杂度 | O(log n) | O(log n) |

{{% callout type="info" %}}
Redis 作者 antirez 表示选择跳表的原因：实现简单、范围查询友好、性能相近。
{{% /callout %}}
