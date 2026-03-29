---
title: 基础语法
weight: 1
---

## 切片（Slice）

### Q: 切片和数组的区别是什么？

**数组**是值类型，长度固定，赋值时会拷贝整个数组。

**切片**是引用类型，底层是对数组的引用，包含三个字段：

```go
type slice struct {
    array unsafe.Pointer  // 指向底层数组的指针
    len   int             // 长度
    cap   int             // 容量
}
```

### Q: 切片扩容机制是什么？

- Go 1.18 之前：容量小于 1024 时翻倍，大于等于 1024 时增长 1.25 倍
- Go 1.18 之后：引入更平滑的扩容策略，避免容量骤增

```go
s := make([]int, 0, 4)
for i := 0; i < 10; i++ {
    s = append(s, i)
    fmt.Printf("len=%d, cap=%d\n", len(s), cap(s))
}
```

---

## Map

### Q: Go 中 Map 的底层实现？

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

### Q: Map 是并发安全的吗？

❌ **不是**，并发读写 map 会触发 panic。

解决方案：
1. 使用 `sync.Mutex` 加锁
2. 使用 `sync.RWMutex` 读写锁
3. 使用 `sync.Map`（适合读多写少场景）

---

## 接口（Interface）

### Q: 接口的底层实现？

接口在底层有两种结构：

```go
// 非空接口 iface
type iface struct {
    tab  *itab          // 类型信息 + 方法表
    data unsafe.Pointer // 指向数据的指针
}

// 空接口 eface
type eface struct {
    _type *_type        // 类型信息
    data  unsafe.Pointer
}
```

### Q: nil 接口和接口的 nil 值有什么区别？

```go
var p *int = nil
var i interface{} = p

fmt.Println(p == nil) // true
fmt.Println(i == nil) // false ← 注意！接口包含类型信息，不为nil
```

{{% callout type="warning" %}}
**面试高频陷阱**：当将一个 nil 指针赋值给接口时，接口本身不为 nil，因为它包含了类型信息。
{{% /callout %}}
