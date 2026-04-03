---
title: 3. 函数传参机制
weight: 3
---

## Q: Go 函数传参是值类型还是引用类型？

在 Go 语言中**只存在值传递**，要么是值的副本，要么是指针的副本。无论是值类型的变量还是引用类型的变量亦或是指针类型的变量作为参数传递都会发生值拷贝，开辟新的内存空间。

{{% callout type="info" %}}
**重要概念区分**：值传递、引用传递和值类型、引用类型是两个不同的概念，不要混淆。引用类型作为变量传递可以影响到函数外部是因为发生值拷贝后新旧变量指向了相同的内存地址。
{{% /callout %}}

## 类型分类

**值类型**：`int`、`uint`、`float32`、`string`、`bool`、`byte(uint8)`

**引用类型**：`slice`、`map`、`channel`、指针

## 示例分析

### 值类型传参

```go
func modifyInt(x int) {
    x = 100  // 只修改副本，不影响原值
}

func main() {
    a := 10
    modifyInt(a)
    fmt.Println(a) // 输出: 10，原值未改变
}
```

### 引用类型传参

```go
func modifySlice(s []int) {
    s[0] = 100  // 修改底层数组，影响原切片
    s = append(s, 4) // 如果触发扩容，不会影响原切片
}

func main() {
    slice := []int{1, 2, 3}
    modifySlice(slice)
    fmt.Println(slice) // 输出: [100, 2, 3]
}
```

### 指针传参

```go
func modifyByPointer(p *int) {
    *p = 100  // 通过指针修改原值
}

func main() {
    a := 10
    modifyByPointer(&a)
    fmt.Println(a) // 输出: 100，原值被修改
}
```

## Q: 如果所有函数入参和返回值都使用指针传递会有什么问题？

1. **空指针错误**：如果指针没有被初始化，可能会出现空指针操作
2. **函数指针入参**：如果对参数进行修改，可能导致参数在外部使用时发生变化
3. **指针返回值**：每次返回都用 `&` 创建新结构体，会导致频繁的内存分配，增加 GC 压力

```go
// 问题示例
func badExample() *User {
    user := User{Name: "Alice"}  // 每次都在堆上分配
    return &user                 // 返回指针，增加GC负担
}

// 更好的做法
func goodExample() User {
    return User{Name: "Alice"}   // 直接返回值，可能在栈上分配
}
```

{{% callout type="warning" %}}
**面试重点**：Go 中只有值传递，但引用类型的"值"是指向底层数据的指针，所以看起来像引用传递。
{{% /callout %}}