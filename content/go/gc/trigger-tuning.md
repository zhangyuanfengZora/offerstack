---
title: 3. GC 触发与调优
weight: 3
---

## Q: GC触发的时机有哪些？

### 主动触发
通过调用 `runtime.GC()` 来触发 GC，此调用阻塞式地等待当前 GC 运行完毕。

### 被动触发
分为两种方式：

1. **时间触发**：go后台有一个系统监控线程，当超过两分钟没有产生任何 GC 时，强制触发 GC

2. **内存触发**：内存使用增长一定比例时触发，每次内存分配时检查当前内存分配量是否已达到阈值（环境变量GOGC）：
   - 默认100%，即当内存扩大一倍时启用GC
   - 第一次GC的触发临界值是4MB

```go
// 可以通过以下方式修改GC触发阈值
debug.SetGCPercent(500) // 表示堆大小超过上次标记的500%时触发GC
```

## Q: GC 关注的指标有哪些？

- **CPU 利用率**：回收算法会在多大程度上拖慢程序？有时候，这个是通过回收占用的 CPU 时间与其它 CPU 时间的百分比来描述的
- **GC 停顿时间**：回收器会造成多长时间的停顿？目前的 GC 中需要考虑 STW 和 Mark Assist 两个部分可能造成的停顿
- **GC 停顿频率**：回收器造成的停顿频率是怎样的？目前的 GC 中需要考虑 STW 和 Mark Assist 两个部分可能造成的停顿
- **GC 可扩展性**：当堆内存变大时，垃圾回收器的性能如何？但大部分的程序可能并不一定关心这个问题

## Q: 有了 GC，为什么还会发生内存泄露？

有GC机制的话，内存泄漏其实是预期的能很快被释放的内存其生命期意外地被延长，导致预计能够立即回收的内存而长时间得不到回收。

**Go语言主要有以下两种**：

1. **内存被根对象引用而没有得到迅速释放**：比如某个局部变量被赋值到了一个全局变量map中

2. **goroutine 泄漏**：一些不当的使用，导致goroutine不能正常退出，也会造成内存泄漏

```go
// 内存泄漏示例1：全局变量持有引用
var globalMap = make(map[string]*BigStruct)

func processData(key string) {
    data := &BigStruct{...}
    globalMap[key] = data  // data被全局变量引用，无法回收
    // 忘记在适当时候删除：delete(globalMap, key)
}

// 内存泄漏示例2：goroutine泄漏
func leakGoroutine() {
    ch := make(chan int)
    go func() {
        // 这个goroutine会永远阻塞，造成泄漏
        <-ch
    }()
    // 函数返回，但goroutine仍在运行
}
```

## Q: Go 的 GC 如何调优？

### 基本策略

1. **合理化内存分配的速度、提高赋值器的 CPU 利用率**
2. **降低并复用已经申请的内存**：比如使用 `sync.Pool` 复用经常需要创建的重复对象
3. **调整 GOGC**：可以适量将 GOGC 的值设置得更大，让 GC 触发的时间变得更晚，从而减少其触发频率，进而增加用户代码对机器的使用率

### 具体优化方法

```go
// 1. 使用对象池减少分配
var bufferPool = sync.Pool{
    New: func() interface{} {
        return make([]byte, 1024)
    },
}

func processData() {
    buf := bufferPool.Get().([]byte)
    defer bufferPool.Put(buf)
    // 使用buf处理数据
}

// 2. 预分配切片容量
func badAllocation() []int {
    var result []int
    for i := 0; i < 1000; i++ {
        result = append(result, i)  // 多次扩容
    }
    return result
}

func goodAllocation() []int {
    result := make([]int, 0, 1000)  // 预分配容量
    for i := 0; i < 1000; i++ {
        result = append(result, i)  // 无需扩容
    }
    return result
}

// 3. 调整GOGC
func init() {
    // 设置更高的GC阈值，减少GC频率
    debug.SetGCPercent(200)
}
```

### 监控工具

```bash
# 查看GC统计信息
GODEBUG=gctrace=1 go run main.go

# 进行逃逸分析
go build -gcflags="-m" main.go

# 使用pprof分析内存
go tool pprof http://localhost:6060/debug/pprof/heap
```

{{% callout type="warning" %}}
**调优原则**：先测量，再优化。不要过早优化，应该基于实际的性能数据进行调优。
{{% /callout %}}