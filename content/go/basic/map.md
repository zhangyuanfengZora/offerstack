---
title: 2. Map
weight: 2
---

## Q: Go 中 Map 的底层实现？

Go 的 map 底层使用**哈希表**实现，由 `hmap` 结构体表示：

```go
type hmap struct {
    count     int            // 元素个数
    flags     uint8          // 状态标志
    B         uint8          // 桶数量 = 2^B
    noverflow uint16         // 溢出桶数量
    hash0     uint32         // 哈希种子
    buckets   unsafe.Pointer // 桶数组指针
    oldbuckets unsafe.Pointer // 扩容时旧桶
    nevacuate  uintptr       // 迁移进度
    extra      *mapextra     // 溢出桶管理
}
```

每个桶（bucket）是一个 `bmap` 结构体，包含：
- 8个 tophash（哈希值的高8位）
- 8个键值对
- 指向下一个溢出桶的指针

{{% callout type="info" %}}
为了内存紧凑，`bmap` 中采用先存8个键再存8个值的存储方式。
{{% /callout %}}

## Q: Map 扩容机制是什么？

**触发时机：**
1. **装载因子超过阈值6.5**：触发翻倍扩容
2. **overflow bucket 数量过多**：触发等量扩容，说明哈希冲突严重

**扩容机制：**
- 采用**渐进式扩容**策略，避免一次性迁移影响性能
- 首先分配新的bucket数组，然后在后续map操作时随机将一些key迁移到新bucket
- 通过渐进式 rehash，将 key 从旧 bucket 迁移到新 bucket

## Q: Map 是并发安全的吗？

❌ **不是**，map 不是线程安全的。map底层有一个flags标志位，在查找、赋值、遍历、删除过程中都会检测写标志，一旦发现写标志置位（等于1），则直接 panic。

**解决方案：**
1. 使用 `sync.Mutex` 加锁
2. 使用 `sync.RWMutex` 读写锁  
3. 使用 `sync.Map`（适合读多写少场景）

```go
// 使用互斥锁保护 map
type SafeMap struct {
    mu sync.RWMutex
    m  map[string]int
}

func (sm *SafeMap) Get(key string) (int, bool) {
    sm.mu.RLock()
    defer sm.mu.RUnlock()
    val, ok := sm.m[key]
    return val, ok
}

func (sm *SafeMap) Set(key string, val int) {
    sm.mu.Lock()
    defer sm.mu.Unlock()
    sm.m[key] = val
}
```
