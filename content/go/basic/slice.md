---
title: 1. 切片与数组
weight: 1
---

## Q: 切片和数组的区别是什么？

**数组**是值类型，长度固定，赋值时会拷贝整个数组。

**切片**是引用类型，底层是对数组的引用，包含三个字段：

```go
type slice struct {
    array unsafe.Pointer  // 指向底层数组的指针
    len   int             // 长度
    cap   int             // 容量
}
```

## Q: 切片扩容机制是什么？

- Go 1.18 之前：容量小于 1024 时翻倍，大于等于 1024 时增长 1.25 倍
- Go 1.18 之后：引入更平滑的扩容策略，避免容量骤增

```go
s := make([]int, 0, 4)
for i := 0; i < 10; i++ {
    s = append(s, i)
    fmt.Printf("len=%d, cap=%d\n", len(s), cap(s))
}
```

## Q: 从一个切片截取另一个切片会改变原来的值吗？

**会影响，但有条件**：在截取完之后，如果新切片没有触发扩容，则修改切片元素会影响原切片；如果触发了扩容则不会。

```go
func main() {
    // 原切片
    original := []int{1, 2, 3, 4, 5}
    
    // 截取切片
    sub := original[1:3]  // [2, 3]
    fmt.Printf("original: %v, sub: %v\n", original, sub)
    
    // 修改截取的切片
    sub[0] = 99
    fmt.Printf("after modify sub[0]:\n")
    fmt.Printf("original: %v, sub: %v\n", original, sub)  // original也被修改了
    
    // 触发扩容
    sub = append(sub, 6, 7, 8, 9, 10)  // 触发扩容，分配新的底层数组
    sub[1] = 88
    fmt.Printf("after expand and modify:\n")
    fmt.Printf("original: %v, sub: %v\n", original, sub)  // original不受影响
}

// 输出：
// original: [1 2 3 4 5], sub: [2 3]
// after modify sub[0]:
// original: [1 99 3 4 5], sub: [99 3]  ← 原切片被影响
// after expand and modify:
// original: [1 99 3 4 5], sub: [99 88 6 7 8 9 10]  ← 原切片不受影响
```

**原理**：
- 截取操作创建的新切片与原切片**共享底层数组**
- 只要没有扩容，修改任一切片都会影响共享的底层数组
- 扩容时会分配新的底层数组，此时两个切片不再共享数据
