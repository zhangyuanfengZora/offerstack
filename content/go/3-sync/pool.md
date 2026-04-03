---
title: 4. sync.Pool
weight: 4
---

## Q: sync.Pool 是什么？

`sync.Pool` 是一个对象池，用于存储和复用临时对象，减少内存分配和GC压力。

**核心特点**：
- **临时性**：Pool中的对象可能随时被GC清理
- **并发安全**：多个goroutine可以安全地存取对象
- **自动扩缩容**：根据使用情况自动调整池大小

## Q: sync.Pool 的使用场景？

1. **频繁创建的临时对象**：如缓冲区、临时结构体
2. **减少GC压力**：复用对象避免频繁分配
3. **性能敏感场景**：网络库、JSON解析等

```go
// 创建对象池
var bufferPool = sync.Pool{
    New: func() interface{} {
        return make([]byte, 1024)  // 创建新对象的函数
    },
}

// 使用对象池
func processData() {
    buf := bufferPool.Get().([]byte)  // 获取对象
    defer bufferPool.Put(buf)         // 归还对象
    
    // 使用buf处理数据...
}
```

## Q: sync.Pool 的实现原理？

**双层结构**：
- **本地池（local pool）**：每个P都有自己的本地池，减少锁竞争
- **共享池（shared pool）**：当本地池为空时，从其他P的池中偷取

**GC交互**：
- 每次GC时，Pool中的对象都可能被清理
- 通过`runtime_registerPoolCleanup`注册清理函数

## Q: 使用 sync.Pool 的注意事项？

1. **不要依赖Pool中的对象状态**：对象可能随时被GC清理
2. **Put前要重置对象**：避免数据泄漏到下次使用
3. **适用于无状态临时对象**：不适合需要保持状态的对象

```go
// 错误示例：依赖对象状态
type Buffer struct {
    data []byte
    important bool  // 不应该依赖这个状态
}

// 正确示例：重置对象
func (b *Buffer) Reset() {
    b.data = b.data[:0]
    b.important = false
}

func useBuffer() {
    buf := bufferPool.Get().(*Buffer)
    defer func() {
        buf.Reset()  // 重置后再归还
        bufferPool.Put(buf)
    }()
    
    // 使用buf...
}
```

{{% callout type="info" %}}
**性能提示**：sync.Pool 主要优化的是内存分配，而不是对象创建的CPU开销。对于创建成本很低的对象，使用Pool可能得不偿失。
{{% /callout %}}