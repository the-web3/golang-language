# Go 语言的 Context 最佳实战

## 一. Context 基本概念

### 1. Context 介绍

`context` 包提供了一个可以在不同的 goroutine 之间传递上下文信息的机制，主要用于处理请求的生命周期，比如取消信号、超时控制和请求范围的数据传递。



有几种方法可以创建 `Context`：

- `context.Background()`: 返回一个空的上下文，通常用于根上下文。
- `context.TODO()`: 返回一个空上下文，表示上下文尚未确定的情况。
- `context.WithCancel(parent Context)`: 返回一个可以取消的上下文。
- `context.WithDeadline(parent Context, deadline time.Time)`: 返回一个在指定时间点到期的上下文。
- `context.WithTimeout(parent Context, timeout time.Duration)`: 返回一个在指定时间段后到期的上下文。



### 2. Context 最佳实战

- 仅在需要的地方使用 `context`。
- 不要将 `Context` 用作一般参数。
- 使用 `context` 存储请求范围的数据，不要存储数据结构或大对象。



## 二. Context 使用场景

### 1. 信号同步与取消

```Go
package main

import (
    "context"
    "fmt"
    "time"
)

func worker(ctx context.Context, id int) {
    for {
        select {
        case <-ctx.Done():
            fmt.Printf("Worker %d: received cancellation signal\n", id)
            return
        default:
            // 模拟工作
            fmt.Printf("Worker %d: working...\n", id)
            time.Sleep(1 * time.Second)
        }
    }
}

func main() {
    ctx, cancel := context.WithCancel(context.Background())

    // 启动多个 worker goroutine
    for i := 1; i <= 3; i++ {
        go worker(ctx, i)
    }

    // 运行一段时间后发送取消信号
    time.Sleep(5 * time.Second)
    cancel() // 发送取消信号

    // 等待一段时间，确保所有 goroutine 完成
    time.Sleep(2 * time.Second)
    fmt.Println("Main: all workers canceled")
}
```

- **Worker 函数**：每个 worker goroutine 会不断循环，执行工作，直到收到取消信号。
- **Context**：在 `main` 函数中创建一个可取消的上下文，并在启动每个 worker 时传递给它。
- **取消信号**：主函数在运行一段时间后调用 `cancel()`，通知所有 worker 停止工作。
- **同步**：通过 `ctx.Done()` 检测取消信号，确保每个 worker 都能优雅退出。





### 2. 默认上下文

`context` 包中最常用的方法还是 `context.Background`、`context.TODO`，这两个方法都会返回预先初始化好的私有变量 `background` 和 `todo`，它们会在同一个 `Go` 程序中被复用

```Go
var (
    background = new(emptyCtx)
    todo       = new(emptyCtx)
)
```

background 通常用在 main 函数中，作为所有 context 的根节点。



todo 通常用在并不知道传递什么 context的情形。例如，调用一个需要传递 context 参数的函数，你手头并没有其他 context 可以传递，这时就可以传递 todo。这常常发生在重构进行中，给一些函数添加了一个 Context 参数，但不知道要传什么，就用 todo “占个位子”，最终要换成其他 context。



### 3.  值传递

```Go
package main

import (
    "context"
    "fmt"
    "time"
)

// 定义常量作为上下文中的键
type key string

const userIDKey key = "userID"

func worker(ctx context.Context) {
    // 从上下文中提取值
    if userID, ok := ctx.Value(userIDKey).(string); ok {
        fmt.Printf("Worker: processing for user %s\n", userID)
    } else {
        fmt.Println("Worker: no user ID found in context")
    }

    // 模拟工作
    time.Sleep(2 * time.Second)
    fmt.Println("Worker: done")
}

func main() {
    // 创建背景上下文
    ctx := context.Background()

    // 将用户 ID 存储到上下文中
    ctx = context.WithValue(ctx, userIDKey, "12345")

    // 启动多个 worker goroutine
    for i := 0; i < 3; i++ {
        go worker(ctx)
    }

    // 等待一段时间，确保所有 goroutine 完成
    time.Sleep(3 * time.Second)
    fmt.Println("Main: all workers done")
}
```

- **键类型**：定义一个 `key` 类型，以避免上下文值的键冲突。
- **上下文值传递**：
  - 在 `main` 函数中，使用 `context.WithValue` 将用户 ID 存储到上下文中。
  - 启动多个 worker goroutine，并将上下文传递给它们。
- **从上下文中读取值**：在 worker 中使用 `ctx.Value` 提取存储的用户 ID，并进行处理。



**注意事项**

- 使用 `context` 传递值时，应该尽量避免存储大型数据结构或复杂对象，避免不必要的内存占用。
- 只在请求范围内使用 `context` 传递信息，不要将其作为常规参数。



# 三. Context 核心实现

### 1. **Context 接口**

`Context` 接口定义了方法来检索截止时间、检查取消状态以及访问与上下文关联的值。方法如下：

- `Deadline()`: 返回上下文的截止时间，如果没有设置截止时间则返回 `ok==false`。
- `Done()`: 返回一个通道，当上下文的工作应该被取消时，该通道会被关闭。如果上下文永远不会被取消，`Done` 可能返回 `nil`。
- `Err()`: 如果 `Done` 尚未关闭，则返回 `nil`。如果 `Done` 已关闭，则返回一个非空错误，解释原因（取消或截止时间已过）。
- `Value(key any) any`: 返回与上下文中键关联的值，如果没有关联的值则返回 `nil`。



### 2. **取消和超时**

- `WithCancel`, `WithDeadline`, `WithTimeout` 函数用于创建派生的上下文，这些上下文可以在父上下文取消或超时时被取消。
- `WithCancelCause` 函数类似于 `WithCancel`，但允许设置取消的原因。



### 3. **上下文传播**

- 上下文应该在函数调用链中显式传递，通常作为第一个参数。
- 不要在结构体中存储上下文，而是将其作为参数传递给需要它的每个函数。


### 4. **上下文值**

- 上下文值应仅用于请求范围内的数据，这些数据在进程和 API 之间传递，而不是用于传递可选参数给函数。



### 5. 主要函数和类型

- **Background 和 TODO**:
  - `Background()` 返回一个非空的、空的上下文，通常用于主函数、初始化和测试，作为传入请求的顶级上下文。
  - `TODO()` 返回一个非空的、空的上下文，当不清楚使用哪个上下文或上下文尚未可用时使用。
- **WithCancel, WithDeadline, WithTimeout**:
  - 这些函数创建派生的上下文，并返回一个 `CancelFunc`，调用该函数可以取消上下文及其子上下文。
- **WithCancelCause**:
  - 类似于 `WithCancel`，但允许设置取消的原因，可以通过 `Cause` 函数检索。
- **WithValue**:
  - 返回一个派生的上下文，其中与键关联的值为 `val`。
  - 

### 6. 实现细节

- **emptyCtx**:：一个永远不会被取消、没有值且没有截止时间的上下文。它是 `backgroundCtx` 和 `todoCtx` 的公共基础。
- **cancelCtx**: 一个可以被取消的上下文。当取消时，它还会取消所有实现 `canceler` 接口的子上下文。
- **timerCtx**: 携带一个定时器和一个截止时间的上下文。它嵌入了一个 `cancelCtx` 来实现 `Done` 和 `Err`。
- **valueCtx**: 携带键值对的上下文。它实现了 `Value` 方法，并将其他调用委托给嵌入的上下文。



# 四. 本章小结

本文讲解了 Context 的最佳实战与底层代码实现，截止到这篇文章，咱们的 Golang 基础编程部分内容就结束了。