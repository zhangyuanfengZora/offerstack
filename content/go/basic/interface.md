---
title: 3. 接口（Interface）
weight: 3
---

## Q: 接口的底层实现？

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

## Q: nil 接口和接口的 nil 值有什么区别？

```go
var p *int = nil
var i interface{} = p

fmt.Println(p == nil) // true
fmt.Println(i == nil) // false ← 注意！接口包含类型信息，不为nil
```

{{% callout type="warning" %}}
**面试高频陷阱**：当将一个 nil 指针赋值给接口时，接口本身不为 nil，因为它包含了类型信息。
{{% /callout %}}
