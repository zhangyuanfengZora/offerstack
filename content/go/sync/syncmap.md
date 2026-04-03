---
title: 3. sync.Map 并发安全映射
weight: 3
---

## Q: sync.Map 的基本原理？

`sync.Map` 的底层核心是**空间换时间**，通过冗余的两个Map（read、dirty），读取操作时操作read这个map，写操作时对dirty进行加锁，然后进行操作，实现**读写分离**。

**结构定义**：
```go
type Map struct {
    mu     Mutex             // 用于保护dirty字段的锁
    read   atomic.Value      // 只读字段，实际数据类型是readOnly结构
    dirty  map[interface{}]*entry  // 需要加锁才能访问的map
    misses int               // 计数器，记录从read中读取失败的次数
}
```

**工作机制**：
- **读操作**：优先从 read map 读取（无锁），如果没找到再从 dirty map 读取（加锁）
- **写操作**：直接操作 dirty map（加锁）
- **数据同步**：如果 dirty map 的数据积累到一定程度，或者 read 里面没有某个 key 时，就会把 dirty 里面的数据覆盖掉 read 里面的数据

## Q: sync.Map 的删除操作机制？

sync.Map 使用两种删除标记：**nil**（逻辑删除）和 **expunged**（彻底删除）

### nil 的作用（逻辑删除）
- read map 是只读快照，不可直接删除 key
- 当删除的 key 存在于 read 中时，把 entry 的指针 p 置为 nil 表示逻辑删除
- 在 promotion 之前，仍然可以直接修改这个 entry（CAS 修改），也就是"可复活"
- 这样避免了频繁创建新的 entry，提高效率

### expunged 的作用（彻底删除）
- 当 dirty 升为 read（promotion）时，之前被逻辑删除的 entry 已经不在 map 结构里
- 但其他 goroutine 可能仍持有旧 entry 指针（entry.p == nil）
- 为防止它们修改旧 entry 导致已删除的 key "复活"，promotion 会把 entry.p 标记为 expunged
- 后续对这个 key 的操作必须创建新的 entry，保证结构安全

## 使用示例

```go
func main() {
    var m sync.Map
    
    // 存储
    m.Store("key1", "value1")
    m.Store("key2", "value2")
    
    // 读取
    if value, ok := m.Load("key1"); ok {
        fmt.Println("key1:", value)
    }
    
    // 读取或存储
    actual, loaded := m.LoadOrStore("key3", "value3")
    fmt.Printf("key3: %v, was loaded: %v\n", actual, loaded)
    
    // 删除
    m.Delete("key2")
    
    // 遍历
    m.Range(func(key, value interface{}) bool {
        fmt.Printf("%v: %v\n", key, value)
        return true  // 继续遍历
    })
}
```

## Q: sync.Map 适用于什么场景？

sync.Map 特别适用于以下场景：

1. **读多写少**：大量的读操作，少量的写操作
2. **多个goroutine读写不相交的键集合**：不同的goroutine操作不同的key
3. **键值相对稳定**：不频繁增删键值对

```go
// 适合的场景：配置缓存
type ConfigCache struct {
    cache sync.Map
}

func (c *ConfigCache) GetConfig(key string) (interface{}, bool) {
    return c.cache.Load(key)  // 频繁读取，无锁
}

func (c *ConfigCache) SetConfig(key string, value interface{}) {
    c.cache.Store(key, value)  // 偶尔写入
}
```

## Q: sync.Map vs 普通 map + Mutex 的性能对比？

| 场景 | sync.Map | map + Mutex |
|------|----------|-------------|
| **读多写少** | 优势明显，读操作无锁 | 读写都需要加锁 |
| **写多读少** | 性能较差，需要维护两个map | 性能相对较好 |
| **均匀读写** | 性能一般 | 可能更好 |
| **内存使用** | 较高（双map结构） | 较低 |

{{% callout type="info" %}}
**选择建议**：
- 读操作远多于写操作时，选择 `sync.Map`
- 写操作较多或对内存敏感时，选择 `map + RWMutex`
- 需要复杂操作（如遍历时修改）时，选择 `map + Mutex`
{{% /callout %}}

```go
// 性能测试示例
func BenchmarkSyncMap(b *testing.B) {
    var m sync.Map
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            m.Store("key", "value")
            m.Load("key")
        }
    })
}

func BenchmarkMapWithMutex(b *testing.B) {
    var mu sync.RWMutex
    m := make(map[string]string)
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            mu.Lock()
            m["key"] = "value"
            mu.Unlock()
            
            mu.RLock()
            _ = m["key"]
            mu.RUnlock()
        }
    })
}
```