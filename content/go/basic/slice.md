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
