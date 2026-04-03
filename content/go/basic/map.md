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

Map并发访问主要破坏4个核心结构：

#### 1. **桶数组指针**
- **问题**：扩容时指针处于半更新状态
- **后果**：访问无效内存，程序崩溃

#### 2. **桶内键值对**  
- **问题**：写入key-value时被打断，出现不匹配
- **后果**：数据不一致，逻辑错误

#### 3. **扩容状态**
- **问题**：桶数量已更新但数据还没完全迁移
- **后果**：按新规则访问旧数据，内存越界

#### 4. **遍历迭代器**
- **问题**：遍历过程中桶结构被修改
- **后果**：迭代器失效，可能无限循环

{{% callout type="error" %}}
**为什么是fatal error？**  
这些底层结构一旦损坏，map就完全不可信了。继续运行可能导致更严重的内存安全问题，所以Go选择直接终止程序。
{{% /callout %}}

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
