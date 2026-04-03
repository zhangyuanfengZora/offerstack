---
title: 1. Mutex 互斥锁
weight: 1
---

## Q: Mutex 的基本原理？

`sync.Mutex` 是一种互斥锁，保证同一时间只有一个goroutine能访问资源。底层是通过**原子操作**和**信号量**来实现的：
- 通过atomic的原子操作来实现加锁的逻辑
- 通过信号量来实现协程的阻塞和唤醒

**结构定义**：
```go
type Mutex struct {
    state int32  // 表示锁定状态，前28位表示等待的协程个数，1位饥饿状态，1位唤醒状态，1位锁定状态
    sema  uint32 // 信号量
}
```

**加锁流程**：
- 无竞争时：state从0变为1（加锁）
- 有锁时：先尝试自旋，如果自旋超过次数就进入等待队列

**解锁流程**：
- 将锁定状态更改为没加锁状态
- 然后去唤醒一个goroutine

## Q: Mutex有几种模式？

### 正常模式（默认）
- **性能优先**：新请求锁的goroutine会和等待队列的第一个goroutine竞争
- **自旋优化**：新来的goroutine会进行几次自旋，如果在自旋期间锁被释放了就可以直接获取锁，减少了协程挂起+唤醒的时间
- **高吞吐量**：但可能导致等待队列的协程等待很久，不公平

### 饥饿模式
- **公平性优先**：当一个goroutine在队列中等待超过1ms之后就会切换到这个模式
- **先来后到**：主打一个先来后到，防止线程被饿死
- **切换回正常模式**：当队列为空或者等待时间少于1ms之后又会切回到正常模式

```go
// 正常模式示例
func normalMode() {
    var mu sync.Mutex
    var wg sync.WaitGroup
    
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            mu.Lock()
            fmt.Printf("Goroutine %d got lock\n", id)
            time.Sleep(time.Millisecond * 10)
            mu.Unlock()
        }(i)
    }
    wg.Wait()
}
```

## Q: Mutex自旋会占用太多资源吗？

**不会**，mutex自旋是有次数和时间限制的：

1. **有限制的自旋**：并不是每次都会自旋，会根据等待队列中的goroutine等待时间进行切换
2. **适用场景**：主要是在竞争不激烈的情况下会进入这种模式，自旋和线程挂起+唤起相比会节省开销
3. **自旋条件**：
   - 锁已被占用，并且锁不处于饥饿模式
   - 积累的自旋次数小于最大自旋次数（active_spin=4）
   - cpu核数大于1
   - 有空闲的P
   - 当前goroutine所挂载的P下，本地待运行队列为空

## 使用示例

```go
type Counter struct {
    mu    sync.Mutex
    value int
}

func (c *Counter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.value++
}

func (c *Counter) Value() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.value
}
```

{{% callout type="warning" %}}
**注意事项**：
- Mutex不可重入，同一个goroutine重复加锁会死锁
- 不要拷贝已使用的Mutex
- 解锁一个未加锁的Mutex会panic
{{% /callout %}}