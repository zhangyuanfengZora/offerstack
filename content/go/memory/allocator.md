---
title: 1. 内存分配器
weight: 1
---

## Q: Go 内存管理的整体思路？

Go 采用**分级分配**的策略，按照不同大小的对象和不同内存层级来分配管理内存。

**内存管理层级**：
```
mspan → mcache → mcentral → mheap → heapArena
```

**分配流程**：
```
每个 P 的 mcache 缓存每种规格的一个mspan（无锁访问）
    ↓
mcentral（8字节规格） → mspan链表 [8B][8B][8B]...
mcentral（16字节规格）→ mspan链表 [16B][16B][16B]...
mcentral（32字节规格）→ mspan链表 [32B][32B][32B]...
    ↓
mheap（全局堆）
    ↓
    OS
```

**分配策略**：
1. **小于16B的对象**：使用mcache微对象进行分配
2. **16B-32KB的对象**：首先计算出需要使用的span大小规格，然后使用mcache中相同大小规格的mspan分配
3. **如果mcache没有可用的mspan**：就会向mcentral申请
4. **如果mcentral中没有可用的mspan**：就向mheap申请
5. **如果mheap中没有可用的span**：就会向操作系统申请一系列新的页（最小1MB）
6. **大于32KB的大对象**：直接从mheap分配

## Q: mspan 是什么？

mspan是golang内存分配管理的基本单位，是一个双向链表。

```go
type mspan struct {
    next *mspan        // 链表指针
    prev *mspan        
    
    startAddr uintptr  // 管理的内存起始地址
    npages    uintptr  // 占用的 page 数量（1 page = 8KB）
    
    spanclass spanClass // 规格类别（如：8字节、16字节...）
    elemsize  uintptr   // 每个 object 的大小
    
    nelems     uint16   // 总共有多少个 object
    allocCount uint16   // 已分配多少个
    freeindex  uint16   // 下一个空闲索引
    
    allocBits  *gcBits  // 分配位图
    allocCache uint64   // 位图缓存
}
```

**基本特点**：
- mspan共有8B-80KB（67种不同规格），其中class=0代表上不封顶，用于大对象分配
- 同样规格的mspan才可以连接在一起

**mspan的优势**：

1. **消除外部碎片**：传统内存分配器（如C的malloc）频繁分配释放不同大小的对象后，会产生大量不连续的小空闲内存——这就是外部碎片。mspan通过固定规格切割解决这个问题：每个mspan只存放一种固定大小的对象，分配时只要任意一个空槽就行，不要求连续空间

2. **优化GC性能**：可以通过spanclass快速判断，如果这个span是noscan类型（里面全是非指针对象），可以跳过扫描，提升GC效率

3. **快速分配状态管理**：mspan使用位图记录object分配状态，可以快速知道哪些分配了哪些没有分配

## Q: mcache 是什么？

```go
type mcache struct {
    alloc [numSpanClasses]*mspan
}
```

**基本介绍**：mcache内部维护了不同规格的mspan，每个Processor(P)都有一个mcache。

**优势**：
1. **避免锁竞争**：goroutine运行时会绑定到一个P上，每个P都有自己的mcache，goroutine在分配内存时只会访问自己P的mcache，避免了跨P的锁竞争
2. **高并发优势**：多个goroutine可以同时从自己的mcache获取内存，提升性能

## Q: mcentral 是什么？

```go
type mcentral struct {
    lock      mutex     // 锁，由于每个P关联的mcache都可能会向mcentral申请空闲的span
    spanclass spanClass // mcentral负责的span规格
    nonempty  mSpanList // 已经使用的span列表(链表)
    empty     mSpanList // 空闲span列表(链表)
    nmalloc   uint64    // mcentral已分配的span计数
}
```

**基本介绍**：一个mcentral维护一个规格的mspan，当mcache中没有可用的mspan时就会向mcentral申请，不同的P可能会向同一个mcentral申请内存，所以需要加锁。

**分配流程**：
1. **查看empty列表**：如果有空闲的列表，就会直接把这个mspan分配给goroutine
2. **如果没有空闲列表，查看nonempty列表**：这里的是部分使用的，有些mspan已经被分配出去了，但还会有空闲的
3. **回收空闲内存单元**：然后把这些空闲的放到empty的列表里面，然后进行分配
4. **如果mcentral不能满足分配要求**：就会向mheap去申请新的内存页

## Q: 内存逃逸是什么？

**逃逸场景**：
1. **返回局部变量指针**
2. **interface类型**
3. **闭包引用外部变量**
4. **切片/map动态扩容**
5. **大对象**

**影响**：因为堆对象需要垃圾回收机制来释放内存，栈对象会随着函数结束被回收，所以大量的内存逃逸会给GC带来压力。

```go
// 逃逸示例
func escape() *int {
    x := 10
    return &x  // x逃逸到堆上
}

func noEscape() int {
    x := 10
    return x   // x在栈上分配
}
```

{{% callout type="info" %}}
**优化建议**：通过 `go build -gcflags="-m"` 可以查看逃逸分析结果，帮助优化内存分配。
{{% /callout %}}