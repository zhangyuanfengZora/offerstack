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

❌ **不是**，map 不是线程安全的。map底层有一个flags标志位，在查找、赋值、遍历、删除过程中都会检测写标志，一旦发现写标志置位（等于1），则直接抛出 **fatal error**。

**解决方案：**
1. 使用 `sync.Mutex` 加锁
2. 使用 `sync.RWMutex` 读写锁  
3. 使用 `sync.Map`（适合读多写少场景）

## Q: 为什么Map并发冲突是fatal error而不是panic？

**fatal error** 和 **panic** 的区别：

| 类型 | 可恢复性 | 处理方式 | 使用场景 |
|------|----------|----------|----------|
| **panic** | 可通过 `recover()` 恢复 | 程序可以继续运行 | 程序逻辑错误，如数组越界 |
| **fatal error** | 不可恢复 | 程序直接终止 | 运行时系统级错误 |

**Map使用fatal error的原因**：

1. **数据竞争的严重性**：并发读写map可能导致数据结构损坏，继续运行会产生不可预测的结果
2. **内存安全**：损坏的map可能导致内存越界访问，威胁程序安全
3. **设计哲学**：Go认为并发安全是程序员的责任，违反这一原则应该立即终止程序

### Q: 具体会破坏Map的哪些数据结构？

Map并发访问可能破坏的核心数据结构包括：

#### 1. **桶数组（buckets）的完整性**
```go
// hmap结构中的关键字段
type hmap struct {
    buckets    unsafe.Pointer // 桶数组指针
    oldbuckets unsafe.Pointer // 扩容时的旧桶数组
    nevacuate  uintptr       // 扩容迁移进度
    // ...
}
```

**破坏场景**：
- 一个goroutine正在扩容，修改`buckets`指针
- 另一个goroutine同时读取，可能读到**半更新状态**的指针
- 结果：访问到无效内存地址，导致程序崩溃

#### 2. **桶内部结构（bmap）的数据一致性**
```go
// 每个桶的内部结构
type bmap struct {
    tophash [8]uint8        // 8个键的哈希高位
    // 紧接着是8个键
    // 然后是8个值  
    // 最后是溢出桶指针
}
```

**破坏场景**：
- goroutine A正在写入key-value对，先写了key，还没写value
- goroutine B同时读取，可能读到**不匹配的key-value对**
- 结果：数据不一致，程序逻辑错误

#### 3. **扩容状态的不一致性**
```go
// 扩容过程中的状态字段
type hmap struct {
    B         uint8          // 桶数量的对数值
    noverflow uint16         // 溢出桶计数
    nevacuate uintptr       // 已迁移的桶索引
}
```

**破坏场景**：
- goroutine A正在扩容，`B`值已更新，但`buckets`还没完全迁移
- goroutine B根据新的`B`值计算桶位置，但访问的还是旧桶结构
- 结果：访问错误的内存位置，可能导致段错误

#### 4. **迭代器状态的破坏**
```go
// 遍历时的迭代器状态
type hiter struct {
    key         unsafe.Pointer // 当前key指针
    elem        unsafe.Pointer // 当前value指针
    bucket      uintptr       // 当前桶索引
    checkBucket uintptr       // 检查桶索引
}
```

**破坏场景**：
- goroutine A正在遍历map，迭代器指向某个桶
- goroutine B同时删除元素，改变了桶的内部结构
- 结果：迭代器指向无效位置，可能无限循环或崩溃

### 实际破坏示例

```go
// 这个例子展示了可能的数据破坏
func demonstrateMapCorruption() {
    m := make(map[int]int)
    
    // 并发写入可能导致的问题：
    go func() {
        for i := 0; i < 1000; i++ {
            m[i] = i * 2  // 可能在写入过程中被打断
        }
    }()
    
    go func() {
        for i := 500; i < 1500; i++ {
            m[i] = i * 3  // 可能读到不一致的桶状态
        }
    }()
    
    // 可能的结果：
    // 1. 程序直接崩溃（fatal error）
    // 2. 数据丢失或错乱
    // 3. 内存泄漏（溢出桶链表被破坏）
    // 4. 无限循环（迭代器状态错乱）
}
```

```go
// 并发访问map会触发fatal error
func concurrentMapAccess() {
    m := make(map[int]int)
    
    // 并发写入
    go func() {
        for i := 0; i < 1000; i++ {
            m[i] = i  // 可能触发: fatal error: concurrent map writes
        }
    }()
    
    go func() {
        for i := 0; i < 1000; i++ {
            m[i] = i  // 可能触发: fatal error: concurrent map writes
        }
    }()
    
    time.Sleep(time.Second)
}

// 正确的并发安全方式
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

{{% callout type="warning" %}}
**重要提醒**：fatal error 无法通过 `recover()` 捕获，程序会直接退出。这是Go故意设计的，强制开发者正确处理并发安全问题。
{{% /callout %}}
