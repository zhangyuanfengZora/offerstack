---
title: 2. make 和 new 的区别
weight: 2
---

## Q: make 和 new 的区别是什么？

make 和 new 都是用于内存分配的内建函数，但使用场景和功能不同：

| 对比项 | make | new |
|--------|------|-----|
| **适用类型** | 只能用于 slice、map、channel | 可用于任何类型 |
| **返回值** | 返回初始化后的数据结构 | 返回指向该内存的指针 |
| **初始化** | 会初始化内部数据结构 | 只分配内存，不初始化 |

## 详细说明

### make 函数

```go
// make 用于初始化并分配内存，只能用于创建 slice、map 和 channel
s := make([]int, 5, 10)    // 创建长度为5，容量为10的切片
m := make(map[string]int)  // 创建空的map
c := make(chan int, 5)     // 创建缓冲区大小为5的channel
```

### new 函数

```go
// new 用于分配内存，但不初始化，返回指向该内存的指针
p := new(int)        // 分配一个int类型的内存，返回*int，值为0
s := new([]int)      // 分配一个[]int类型的内存，返回*[]int，值为nil切片

fmt.Println(*p)      // 输出: 0
fmt.Println(*s)      // 输出: []
fmt.Println(*s == nil) // 输出: true
```

## 示例对比

```go
// 使用 make 创建切片
s1 := make([]int, 3)    // s1 = [0, 0, 0]，可以直接使用
s1[0] = 1

// 使用 new 创建切片
s2 := new([]int)        // s2 = &[]，是指向nil切片的指针
*s2 = append(*s2, 1)    // 需要解引用才能使用

// 使用 make 创建 map
m1 := make(map[string]int)  // m1 是初始化的空map，可以直接使用
m1["key"] = 1

// 使用 new 创建 map
m2 := new(map[string]int)   // m2 是指向nil map的指针
*m2 = make(map[string]int)  // 需要先初始化才能使用
(*m2)["key"] = 1
```

{{% callout type="warning" %}}
**面试重点**：make 返回的是初始化后的数据结构本身，new 返回的是指向零值的指针。
{{% /callout %}}