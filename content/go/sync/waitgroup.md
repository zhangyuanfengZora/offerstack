---
title: 2. WaitGroup 等待组
weight: 2
---

## Q: WaitGroup 怎么实现协程等待的？

WaitGroup实现等待，本质上是**一个原子计数器和一个信号量的协作**。

**工作原理**：
- 调用 `Add` 会增加计数值
- `Done` 会减计数值  
- `Wait` 方法会检查这个计数器，如果不为零，就利用信号量将当前goroutine高效地挂起
- 直到最后一个 `Done` 调用将计数器清零，它就会通过这个信号量，一次性唤醒所有在 `Wait` 处等待的goroutine，从而实现等待目的

**结构定义**：
```go
type WaitGroup struct {
    noCopy noCopy
    state1 [3]uint32  // 包含计数器和信号量
}
```

## 使用示例

```go
func main() {
    var wg sync.WaitGroup
    
    for i := 0; i < 5; i++ {
        wg.Add(1)  // 增加计数
        go func(id int) {
            defer wg.Done()  // 减少计数
            fmt.Printf("Worker %d is working\n", id)
            time.Sleep(time.Second)
            fmt.Printf("Worker %d finished\n", id)
        }(i)
    }
    
    wg.Wait()  // 等待所有goroutine完成
    fmt.Println("All workers finished")
}
```

## 常见使用模式

### 批量任务处理
```go
func processBatch(items []string) {
    var wg sync.WaitGroup
    
    for _, item := range items {
        wg.Add(1)
        go func(item string) {
            defer wg.Done()
            processItem(item)
        }(item)
    }
    
    wg.Wait()
}
```

### 带限制的并发
```go
func processWithLimit(items []string, limit int) {
    var wg sync.WaitGroup
    semaphore := make(chan struct{}, limit)
    
    for _, item := range items {
        wg.Add(1)
        go func(item string) {
            defer wg.Done()
            
            semaphore <- struct{}{}        // 获取许可
            defer func() { <-semaphore }() // 释放许可
            
            processItem(item)
        }(item)
    }
    
    wg.Wait()
}
```

{{% callout type="warning" %}}
**使用注意事项**：
- `Add` 必须在 `Wait` 之前调用
- `Add` 的参数可以是负数，但计数器不能变为负数
- 不要在goroutine内部调用 `Add`，应该在启动goroutine之前调用
- WaitGroup 可以重复使用，但必须等待上一轮完成
{{% /callout %}}

## Q: WaitGroup 的零值可以直接使用吗？

**可以**。WaitGroup 的零值是有效的，可以直接使用，不需要初始化。

```go
var wg sync.WaitGroup  // 零值可以直接使用
wg.Add(1)
go func() {
    defer wg.Done()
    // do something
}()
wg.Wait()
```