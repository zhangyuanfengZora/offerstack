---
title: 2. Map
weight: 2
---

## Q: Go 中 Map 的底层实现？

Go 的 map 底层使用**哈希表**实现，由 `hmap` 结构体表示：

```go
type hmap struct {
    count     int            // 元素个数
    B         uint8          // 桶数量 = 2^B
    buckets   unsafe.Pointer // 桶数组指针
    oldbuckets unsafe.Pointer // 扩容时旧桶
    // ...
}
```

## Q: Map 是并发安全的吗？

❌ **不是**，并发读写 map 会触发 panic。

解决方案：
1. 使用 `sync.Mutex` 加锁
2. 使用 `sync.RWMutex` 读写锁
3. 使用 `sync.Map`（适合读多写少场景）
