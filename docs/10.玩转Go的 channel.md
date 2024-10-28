# 玩转Go的 channel

Golang在并发编程上有两大利器，分别是`channel`和`goroutine`，本节内容主要讲解`channel`。

## 一. Channel 的基本用法

从字面上看，`channel`的意思大概就是管道的意思。`channel`是一种go协程用以接收或发送消息的安全的消息队列，`channel`就像两个go协程之间的导管，来实现各种资源的同步。

暂时无法在Lark文档外展示此内容

使用`channel`时有几个注意点：

- 向一个`nil` `channel`发送消息，会一直阻塞；
- 向一个已经关闭的`channel`发送消息，会引发运行时恐慌`（panic）`；
- `channel`关闭后不可以继续向`channel`发送消息，但可以继续从`channel`接收消息；
- 当`channel`关闭并且缓冲区为空时，继续从从`channel`接收消息会得到一个对应类型的零值。



### 1. Channel 类型

- **无缓冲 channel**：接收者和发送者必须同时存在，发送和接收会阻塞，确保数据的同步。

![img](https://github.com/the-web3/golang-language/tree/main/docs/imgs/1.png)

```Go
package main

import (
    "fmt"
)

func main() {
    ch := make(chan int) // 无缓冲区的 channel
    go func() {
       ch <- 42 // 发送数据
    }()

    // 接收数据
    value := <-ch
    fmt.Println("接收到的值:", value) // 输出: 接收到的值: 42
}
```

- **缓冲 channel**：允许在没有接收者的情况下发送数据，直到达到缓冲区的大小。发送数据会阻塞直到有空间可用，接收数据会阻塞直到有数据可接收。

![img](https://github.com/the-web3/golang-language/tree/main/docs/imgs/2.png)

```Go
package main

import "fmt"

func main() {
    tasks := make(chan int, 10)

    for i := 0; i < 10; i++ {
       fmt.Println("task ", i)
       tasks <- i
    }
    close(tasks)

    for task := range tasks {
       fmt.Println("处理任务:", task)
    }
}
```

### 2. Channel 的关闭

- 关闭 channel 的好处是接收方可以通过遍历来判断何时结束。通过 `for value := range ch` 语法可以有效处理。
- 注意：在关闭 channel 后，不要再向其发送数据，这样会引发 panic。



### 3. channel 的遍历

#### 3.1 for ... range 遍历无缓冲区的 Channel



```Go
package main

import "fmt"

func main() {
    ch := make(chan int)

    // 启动一个 goroutine 向 channel 发送数据
    go func() {
       for i := 0; i < 5; i++ {
          ch <- i
       }
       close(ch) // 发送完毕后关闭 channel
    }()

    // 遍历 channel 接收数据
    for value := range ch {
       fmt.Println("接收到:", value) // 输出: 接收到: 0, 1, 2, 3, 4
    }

    fmt.Println("channel 已关闭")
}
```

**遍历的注意事项**

- **关闭 channel**：在使用 `for range` 遍历 channel 时，必须确保 channel 在发送完所有数据后被关闭。如果 channel 没有关闭，遍历会一直阻塞，直到有数据可接收。
- **零值处理**：如果尝试从已关闭的 channel 接收数据，接收到的将是该类型的零值。例如，接收 int 类型的 channel，关闭后再接收将返回 0。



#### 3.2 for ... range 遍历有缓冲区的 Channel

```Go
package main

import (
    "fmt"
)

func main() {
    ch := make(chan int, 3) // 创建一个缓冲大小为 3 的 channel

    go func() {
        for i := 0; i < 5; i++ {
            ch <- i
        }
        close(ch)
    }()

    for value := range ch {
        fmt.Println("接收到:", value) // 可能输出: 0, 1, 2, 3, 4
    }
}
```

#### 3.3 遍历时的性能考虑

- 在高并发场景中，遍历 channel 时要考虑性能，避免阻塞和长时间等待。
- 如果需要进行复杂处理，建议将处理逻辑放在 goroutine 中，以保持遍历的流畅性。



### 4. Channel 和 Select 联合使用

```Go
package main

import (
    "context"
    "fmt"
    "sync"
    "time"
)

type TimeTask struct {
    Ctx          context.Context
    PollInterval time.Duration
    TimeOut      time.Duration
    wg           sync.WaitGroup
    cancel       func()
}

func NewTimeTask(ctx context.Context, pollInterval, timeout time.Duration) (*TimeTask, error) {
    _, cancelFunc := context.WithTimeout(ctx, timeout)
    return &TimeTask{
       Ctx:          ctx,
       PollInterval: pollInterval,
       cancel:       cancelFunc,
    }, nil
}

func (tt *TimeTask) Start() error {
    tt.wg.Add(1)
    go func() {
       err := tt.SyncChain()
       if err != nil {
          fmt.Println("Sync chain err", "err", err)
       }
    }()
    return nil
}

func (tt *TimeTask) Stop() error {
    tt.cancel()
    tt.wg.Wait()
    return nil
}

func (tt *TimeTask) SyncChain() error {
    defer tt.wg.Done()
    ticker := time.NewTicker(tt.PollInterval)
    defer ticker.Stop()
    for {
       select {
       case <-ticker.C:
          fmt.Println("Sync Chain Normal")
          // implement your business
       case err := <-tt.Ctx.Done():
          fmt.Println("Sync error and go exit", "err", err)
          return nil
       }
    }
}
package main

import (
    "context"
    "fmt"
    "time"
)

func main() {
    task, err := NewTimeTask(context.Background(), time.Second*2, time.Second*15)
    if err != nil {
       fmt.Println("New time task error", "err", err)
       return
    }
    err = task.Start()
    if err != nil {
       fmt.Println("start time task error", "err", err)
       return
    }

    <-(chan struct{})(nil) // 不推荐的写法
}
```

### 5. 组合使用多个 channel

可以通过组合多个 channel 来实现复杂的逻辑，比如将多个 goroutine 的结果合并到一个 channel。

```Go
package main

import (
    "fmt"
)

func merge(ch1, ch2 <-chan int) <-chan int {
    out := make(chan int)
    go func() {
       defer close(out)
       for {
          select {
          case v, ok := <-ch1:
             if ok {
                out <- v
             } else {
                ch1 = nil // 关闭 ch1 后，将其设置为 nil，以避免再接收
             }
          case v, ok := <-ch2:
             if ok {
                out <- v
             } else {
                ch2 = nil // 关闭 ch2 后，将其设置为 nil
             }
          }
          // 如果两个 channel 都关闭了，退出循环
          if ch1 == nil && ch2 == nil {
             break
          }
       }
    }()
    return out
}

func main() {
    ch1 := make(chan int)
    ch2 := make(chan int)

    // 启动 goroutine 向 ch1 发送数据
    go func() {
       for i := 0; i < 5; i++ {
          ch1 <- i
       }
       close(ch1) // 关闭 ch1
    }()

    // 启动 goroutine 向 ch2 发送数据
    go func() {
       for i := 5; i < 10; i++ {
          ch2 <- i
       }
       close(ch2) // 关闭 ch2
    }()

    // 调用 merge 函数
    merged := merge(ch1, ch2)

    // 从合并后的 channel 中接收数据
    for value := range merged {
       fmt.Println("接收到:", value)
    }
}
```

**说明**

- **创建输入 channel**：
  - `ch1` 和 `ch2` 是输入 channel，用于接收数据。
- **启动 goroutines**：
  - 启动两个 goroutine，分别向 `ch1` 和 `ch2` 发送数据，发送完成后关闭对应的 channel。
- **调用** **merge** **函数**：
  - 通过 `merge(ch1, ch2)` 合并两个 channel，返回一个新的输出 channel。
- **接收合并后的数据**：
  - 使用 `for range` 循环遍历合并后的 channel `merged`，接收并打印数据。

### 6. 通道的内存管理

确保 goroutine 在完成工作后能够正确退出，可以使用 `sync.WaitGroup` 来等待所有 goroutine 完成。

```Go
var wg sync.WaitGroup
ch := make(chan int)

for i := 0; i < 5; i++ {
    wg.Add(1)
    go func(i int) {
        defer wg.Done()
        ch <- i
    }(i)
}

go func() {
    wg.Wait()
    close(ch)
}()

for value := range ch {
    fmt.Println(value)
}
```

### 7. 使用 Context 进行取消控制

通过 context 可以优雅地控制 goroutine 的生命周期，便于取消操作和处理超时。

```Go
ctx, cancel := context.WithCancel(context.Background())
defer cancel()

go func() {
    select {
    case <-ctx.Done():
        fmt.Println("取消操作")
        return
    case value := <-ch:
        fmt.Println("接收数据:", value)
    }
}()
```

### 8. Channel 的主要特点

- **数据传输**：Channel 用于发送和接收数据，可以传输各种数据类型（如基本数据类型、结构体等）。
- **同步机制**：发送和接收操作是阻塞的：发送操作会在接收方准备好接收数据之前阻塞，接收操作会在有数据可接收之前阻塞。这种特性使得 goroutines 之间能够安全地交换数据。
- **类型安全**：每个 channel 都有特定的数据类型，确保只有这种类型的数据可以被发送和接收，避免了类型错误。
- **缓冲和无缓冲**：
  - **无缓冲 channel**：发送和接收必须同时进行，确保数据同步。
  - **缓冲 channel**：允许在没有接收者的情况下发送数据，直到达到缓冲区的容量。
- **关闭 channel**：Channel 可以被关闭，关闭后不再发送数据，但接收方仍然可以接收已发送的数据直到 channel 中没有更多数据。关闭后，接收方可以通过 `for range` 循环来判断何时结束接收。



## 二. Channel 底层实现

### 1. Channel 底层数据结构

```Go
type hchan struct {
    qcount   uint           // total data in the queue
    dataqsiz uint           // size of the circular queue
    buf      unsafe.Pointer // points to an array of dataqsiz elements
    elemsize uint16
    closed   uint32
    timer    *timer // timer feeding this chan
    elemtype *_type // element type
    sendx    uint   // send index
    recvx    uint   // receive index
    recvq    waitq  // list of recv waiters
    sendq    waitq  // list of send waiters

    // lock protects all fields in hchan, as well as several
    // fields in sudogs blocked on this channel.
    //
    // Do not change another G's status while holding this lock
    // (in particular, do not ready a G), as this can deadlock
    // with stack shrinking.
    lock mutex
}
```

`channel`的缓冲区其实是一个环形队列

- **qcount**：当前通道队列中的元素总数。
- **dataqsiz**：分配给通道的循环队列的大小。
- **buf**：指向存储通道实际数据的缓冲区的指针。
- **elemsize**：通道中每个元素的大小（以字节为单位）。
- **closed**：表示通道是否已关闭（通常用整数表示布尔值）。
- **timer**：指向用于通道阻塞操作的定时器的指针。
- **elemtype**：指向通道可以保存的元素类型的指针。
- **sendx**：下一个要发送到通道的元素的索引。
- **recvx**：下一个要从通道接收的元素的索引。
- **recvq**：等待接收通道的 goroutine 的等待队列。
- **sendq**：等待发送到通道的 goroutine 的等待队列。
- **lock**：一个互斥锁，用于保护 `hchan` 结构体中的所有字段，确保线程安全访问。



### 2. Channel 的创建

`makechan`的代码逻辑还是比较简单的，首先校验元素类型和缓冲区空间大小，然后创建`hchan`，分配所需空间。这里有三种情况：当缓冲区大小为0，或者`channel`中元素大小为0时，只需分配`channel`必需的空间即可；当`channel`元素类型不是指针时，则只需要为`hchan`和缓冲区分配一片连续内存空间，空间大小为缓冲区数组空间加上`hchan`必需的空间；默认情况，缓冲区包含指针，则需要为`hchan`和缓冲区分别分配内存。最后更新`hchan`的其他字段，包括`elemsize`，`elemtype`，`dataqsiz`。

```Go
func makechan(t *chantype, size int) *hchan {
    elem := t.Elem

    // compiler checks this but be safe.
    if elem.Size_ >= 1<<16 {
       throw("makechan: invalid channel element type")
    }
    if hchanSize%maxAlign != 0 || elem.Align_ > maxAlign {
       throw("makechan: bad alignment")
    }

    mem, overflow := math.MulUintptr(elem.Size_, uintptr(size))
    if overflow || mem > maxAlloc-hchanSize || size < 0 {
       panic(plainError("makechan: size out of range"))
    }

    // Hchan does not contain pointers interesting for GC when elements stored in buf do not contain pointers.
    // buf points into the same allocation, elemtype is persistent.
    // SudoG's are referenced from their owning thread so they can't be collected.
    // TODO(dvyukov,rlh): Rethink when collector can move allocated objects.
    var c *hchan
    switch {
    case mem == 0:
       // Queue or element size is zero.
       c = (*hchan)(mallocgc(hchanSize, nil, true))
       // Race detector uses this location for synchronization.
       c.buf = c.raceaddr()
    case !elem.Pointers():
       // Elements do not contain pointers.
       // Allocate hchan and buf in one call.
       c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
       c.buf = add(unsafe.Pointer(c), hchanSize)
    default:
       // Elements contain pointers.
       c = new(hchan)
       c.buf = mallocgc(mem, elem, true)
    }

    c.elemsize = uint16(elem.Size_)
    c.elemtype = elem
    c.dataqsiz = uint(size)
    lockInit(&c.lock, lockRankHchan)

    if debugChan {
       print("makechan: chan=", c, "; elemsize=", elem.Size_, "; dataqsiz=", size, "\n")
    }
    return c
}
```

#### 2.1 函数概述

- **参数**：
  - `t *chantype`：通道的类型信息，包含元素的类型和大小。
  - `size int`：通道的大小，即可以存储的元素数量。
- **返回值**：返回一个指向 `hchan` 结构体的指针，代表创建的通道。

#### 2.2 关键步骤

- **元素类型检查**：
  - 检查元素大小是否超过 `1<<16`，如果是，则抛出错误。
  - 检查 `hchan` 大小的对齐要求，确保其符合最大对齐约束。
- **内存分配计算**：
  - 使用 `math.MulUintptr` 计算总所需内存，判断是否溢出。
  - 如果内存溢出、超出最大分配限制，或者通道大小小于 0，则引发恐慌。
- **通道分配**：
  - 如果 `mem` 为 0（表示元素大小或通道大小为 0），则直接分配一个 `hchan` 结构体。
  - 如果元素类型不包含指针，则可以在一次调用中分配 `hchan` 和缓冲区 `buf` 的内存。
  - 如果元素类型包含指针，则单独创建一个 `hchan`，并为元素分配内存。
- **设置通道属性**：
  - 设置 `elemsize` 为元素的大小。
  - 设置 `elemtype` 为元素的类型。
  - 设置 `dataqsiz` 为通道的大小。
  - 初始化锁以保证线程安全。

### 3. Channel 发送

`channel`的发送操作调用了运行时方法`chansend1`, 在`chansend1`内部又调用了`chansend`，直接来看`chansend`的实现

```Go
/*
 * 通用单通道发送/接收
 * 如果 block 不是 nil，
 * 则协议不会睡眠，而是在无法完成时返回。
 *
 * 当与之相关的通道已经关闭时，sleep 可以在 g.param == nil 的情况下唤醒。
 * 最简单的方法是循环并重新执行操作；我们会发现通道现在已关闭。
 */
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
    if c == nil {
       // 如果是非阻塞，直接返回发送不成功
       if !block {
          return false
       }
       gopark(nil, nil, waitReasonChanSendNilChan, traceBlockForever, 2)
       throw("unreachable")
    }

    if debugChan {
       print("chansend: chan=", c, "\n")
    }

    if raceenabled {
       racereadpc(c.raceaddr(), callerpc, abi.FuncPCABIInternal(chansend))
    }

    // 快速路径：在不获取锁的情况下检查非阻塞操作是否失败。
    //
    // 在观察通道未关闭之后，我们会检查通道是否准备好进行发送。每次观察是一次单个字大小的读取
    // （先读取 c.closed，然后读取 full()）。
    // 因为一个已关闭的通道不会从“准备发送”状态转变为“未准备发送”状态，
    // 即使通道在这两次观察之间关闭，也意味着在两次观察之间有一个时刻，通道既没有关闭，
    // 也没有准备好发送。我们假设自己在这个时刻观察到了通道，
    // 并报告发送操作无法继续。
    //
    // 即使这些读取发生重排序也是可以的：如果我们先观察到通道未准备好发送，
    // 然后观察到它没有关闭，这意味着在第一次观察期间通道并没有关闭。
    // 然而，这里没有保证前进的进度。我们依赖于 chanrecv() 和 closechan() 中锁释放的副作用
    // 来更新该线程对 c.closed 和 full() 的观察。
    if !block && c.closed == 0 && full(c) {
       return false
    }

    var t0 int64
    if blockprofilerate > 0 {
       t0 = cputicks()
    }

    lock(&c.lock)

    if c.closed != 0 {
       unlock(&c.lock)
       panic(plainError("send on closed channel"))
    }

    if sg := c.recvq.dequeue(); sg != nil {
       // Found a waiting receiver. We pass the value we want to send
       // directly to the receiver, bypassing the channel buffer (if any).
       send(c, sg, ep, func() { unlock(&c.lock) }, 3)
       return true
    }

    if c.qcount < c.dataqsiz {
       // Space is available in the channel buffer. Enqueue the element to send.
       qp := chanbuf(c, c.sendx)
       if raceenabled {
          racenotify(c, c.sendx, nil)
       }
       typedmemmove(c.elemtype, qp, ep)
       c.sendx++
       if c.sendx == c.dataqsiz {
          c.sendx = 0
       }
       c.qcount++
       unlock(&c.lock)
       return true
    }

    if !block {
       unlock(&c.lock)
       return false
    }

    // Block on the channel. Some receiver will complete our operation for us.
    gp := getg()
    mysg := acquireSudog()
    mysg.releasetime = 0
    if t0 != 0 {
       mysg.releasetime = -1
    }
    // No stack splits between assigning elem and enqueuing mysg
    // on gp.waiting where copystack can find it.
    mysg.elem = ep
    mysg.waitlink = nil
    mysg.g = gp
    mysg.isSelect = false
    mysg.c = c
    gp.waiting = mysg
    gp.param = nil
    c.sendq.enqueue(mysg)
    // Signal to anyone trying to shrink our stack that we're about
    // to park on a channel. The window between when this G's status
    // changes and when we set gp.activeStackChans is not safe for
    // stack shrinking.
    gp.parkingOnChan.Store(true)
    gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceBlockChanSend, 2)
    // Ensure the value being sent is kept alive until the
    // receiver copies it out. The sudog has a pointer to the
    // stack object, but sudogs aren't considered as roots of the
    // stack tracer.
    KeepAlive(ep)

    // someone woke us up.
    if mysg != gp.waiting {
       throw("G waiting list is corrupted")
    }
    gp.waiting = nil
    gp.activeStackChans = false
    closed := !mysg.success
    gp.param = nil
    if mysg.releasetime > 0 {
       blockevent(mysg.releasetime-t0, 2)
    }
    mysg.c = nil
    releaseSudog(mysg)
    if closed {
       if c.closed == 0 {
          throw("chansend: spurious wakeup")
       }
       panic(plainError("send on closed channel"))
    }
    return true
}
```

- **空通道检查**：如果通道 `c` 为 `nil`，根据 `block` 的值决定是否阻塞，如果不允许阻塞则返回 `false`。
- **调试信息**：如果启用了调试模式，打印通道信息。
- **竞态条件检测**：如果启用了竞态检测，则记录当前的程序计数器。
- **快速路径**：在不获取锁的情况下检查通道是否关闭以及是否已满。如果通道未关闭且已满，返回 `false`。
- **锁定通道**：获取通道的锁，确保线程安全。
- **关闭通道检查**：如果通道已关闭，释放锁并引发错误。
- **查找等待接收者**：如果有接收者在等待，则将要发送的值直接传递给接收者，跳过通道缓冲区。
- **检查缓冲区空间**：如果通道缓冲区有空间，则将数据放入缓冲区，并更新相关状态（如发送索引和计数）。
- **阻塞发送**：
  - 如果没有空间且不允许阻塞，释放锁并返回 `false`。
  - 如果允许阻塞，创建一个 `sudog`（代表等待的 goroutine），并将其放入发送队列中，之后调用 `gopark` 进行阻塞。
- **保持值的存活**：在阻塞期间，确保要发送的值在接收之前不会被回收。
- **被唤醒后的处理**：
  - 被唤醒后检查 `mysg` 是否仍然有效，如果不一致，抛出错误。
  - 检查通道是否已关闭并处理相关逻辑。
- **返回结果**：如果通道被关闭，抛出错误，否则返回 `true` 表示发送成功。

#### 3.1 存在等待接收的Goroutine

如果等待接收的队列`recvq`中存在`Goroutine`，那么直接把正在发送的值发送给等待接收的`Goroutine`

![img](https://github.com/the-web3/golang-language/tree/main/docs/imgs/4.png)

具体看一下`send`方法：

```Go
// send 处理在空通道 c 上的发送操作。
// 发送者发送的值 ep 被复制到接收者 sg。
// 然后唤醒接收者，让其继续执行。
// 通道 c 必须为空且已锁定。send 使用 unlockf 解锁 c。
// sg 必须已经从通道 c 中出队。
// ep 必须为非 nil，并指向堆或调用者的栈。
func send(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
    if raceenabled {
       if c.dataqsiz == 0 {
          racesync(c, sg)
       } else {
          // 假装我们是通过缓冲区进行发送，即使我们是直接复制数据。
         // 请注意，只有在启用竞态检测时，我们才需要增加头/尾位置。
          racenotify(c, c.recvx, nil)
          racenotify(c, c.recvx, sg)
          c.recvx++
          if c.recvx == c.dataqsiz {
             c.recvx = 0
          }
          c.sendx = c.recvx // c.sendx = (c.sendx+1) % c.dataqsiz
       }
    }
    if sg.elem != nil {
       sendDirect(c.elemtype, sg, ep)
       sg.elem = nil
    }
    gp := sg.g
    unlockf()
    gp.param = unsafe.Pointer(sg)
    sg.success = true
    if sg.releasetime != 0 {
       sg.releasetime = cputicks()
    }
    goready(gp, skip+1)
}
```

- **竞态检测**：
  - 如果启用了竞态检测：
    - 当通道为空时，调用 `racesync(c, sg)` 进行同步。
    - 如果通道有缓冲，模拟通过缓冲区进行发送，即使我们实际上是直接复制数据。此时需要增加接收索引 `recvx` 和发送索引 `sendx`。
- **更新接收索引**：
  - 如果通道不为空且启用了竞态检测，调用 `racenotify` 来通知当前的接收状态。
  - 更新接收索引 `recvx`，并循环回到起始位置（当索引到达通道大小时）。
- **发送数据**：
  - 如果 `sg.elem` 不为 `nil`，调用 `sendDirect(c.elemtype, sg, ep)` 将要发送的数据从 `ep` 复制到接收者 `sg` 中。
  - 发送后，将 `sg.elem` 设为 `nil`，以避免悬空指针。
- **解锁通道**：调用 `unlockf()` 解锁通道，允许其他操作。
- **唤醒接收者**：
  - 获取当前的 goroutine `gp`。
  - 将 `sg` 指针赋值给 `gp.param`，以便在唤醒时能获取到相应的值。
  - 将 `sg.success` 设置为 `true`，表示发送成功。
  - 如果 `sg.releasetime` 不为零，更新其值为当前 CPU 时钟滴答数（表示释放时间）。
- **准备 goroutine**：调用 `goready(gp, skip+1)` 将 `gp` 标记为就绪，准备调度执行。
  - ```Go
    _Gidle = iota // goroutine刚刚分配，还没有初始化
    _Grunnable    // goroutine处于运行队列中, 还没有运行，没有自己的栈
    _Grunning     // goroutine在运行中，拥有自己的栈，被分配了M(线程)和P(调度上下文)
    _Gsyscall     // goroutine在执行系统调用
    _Gwaiting     // goroutine被阻塞
    _Gdead        // goroutine没有被使用，可能是刚刚退出，或者正在初始化中
    _Gcopystack   // 表示g当前的栈正在被移除并分配新栈
    ```

当调用`goready`时，将`Goroutine`的状态从 `_Gwaiting`置为`_Grunnable`，等待下一次调度再次执行。

#### 3.1 当缓冲区未满时

当缓冲区未满时，找到`sendx`所指向的缓冲区数组的位置，将正在发送的值拷贝到该位置，并增加`sendx`索引以及释放锁，示意图如下：

![img](https://github.com/the-web3/golang-language/tree/main/docs/imgs/5.png)

##### 3.1.1 阻塞发送

如果是阻塞发送，那么就将当前的`Goroutine`打包成一个`sudog`结构体，并加入到`channel`的发送队列`sendq`里。示意图如下：

![img](https://github.com/the-web3/golang-language/tree/main/docs/imgs/6.png)

之后则调用`goparkunlock`将当前`Goroutine`设置为`_Gwaiting`状态并解锁，进入阻塞状态等待被唤醒（调用`goready`）；如果被调度器唤醒，执行清理工作并最终释放对应的`sudog`结构体。



### 4. Channel 接收

`channel`的接收有两种形式：两种方式分别调用运行时方法`chanrecv1`和`chanrecv2`；两个方法最终都会调用`chanrecv`方法, 该方法负责处理通道的接收操作，包括从发送者处接收数据、处理通道关闭的情况、以及在没有数据时阻塞等待发送者。在这个过程中，考虑了竞态条件，并确保在并发环境下的安全性和正确性。

```Go
// chanrecv receives on channel c and writes the received data to ep.
// ep may be nil, in which case received data is ignored.
// If block == false and no elements are available, returns (false, false).
// Otherwise, if c is closed, zeros *ep and returns (true, false).
// Otherwise, fills in *ep with an element and returns (true, true).
// A non-nil ep must point to the heap or the caller's stack.
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
    // raceenabled: don't need to check ep, as it is always on the stack
    // or is new memory allocated by reflect.

    if debugChan {
       print("chanrecv: chan=", c, "\n")
    }

    if c == nil {
       if !block {
          return
       }
       gopark(nil, nil, waitReasonChanReceiveNilChan, traceBlockForever, 2)
       throw("unreachable")
    }

    if c.timer != nil {
       c.timer.maybeRunChan()
    }

    // Fast path: check for failed non-blocking operation without acquiring the lock.
    if !block && empty(c) {
       // After observing that the channel is not ready for receiving, we observe whether the
       // channel is closed.
       //
       // Reordering of these checks could lead to incorrect behavior when racing with a close.
       // For example, if the channel was open and not empty, was closed, and then drained,
       // reordered reads could incorrectly indicate "open and empty". To prevent reordering,
       // we use atomic loads for both checks, and rely on emptying and closing to happen in
       // separate critical sections under the same lock.  This assumption fails when closing
       // an unbuffered channel with a blocked send, but that is an error condition anyway.
       if atomic.Load(&c.closed) == 0 {
          // Because a channel cannot be reopened, the later observation of the channel
          // being not closed implies that it was also not closed at the moment of the
          // first observation. We behave as if we observed the channel at that moment
          // and report that the receive cannot proceed.
          return
       }
       // The channel is irreversibly closed. Re-check whether the channel has any pending data
       // to receive, which could have arrived between the empty and closed checks above.
       // Sequential consistency is also required here, when racing with such a send.
       if empty(c) {
          // The channel is irreversibly closed and empty.
          if raceenabled {
             raceacquire(c.raceaddr())
          }
          if ep != nil {
             typedmemclr(c.elemtype, ep)
          }
          return true, false
       }
    }

    var t0 int64
    if blockprofilerate > 0 {
       t0 = cputicks()
    }

    lock(&c.lock)

    if c.closed != 0 {
       if c.qcount == 0 {
          if raceenabled {
             raceacquire(c.raceaddr())
          }
          unlock(&c.lock)
          if ep != nil {
             typedmemclr(c.elemtype, ep)
          }
          return true, false
       }
       // The channel has been closed, but the channel's buffer have data.
    } else {
       // Just found waiting sender with not closed.
       if sg := c.sendq.dequeue(); sg != nil {
          // Found a waiting sender. If buffer is size 0, receive value
          // directly from sender. Otherwise, receive from head of queue
          // and add sender's value to the tail of the queue (both map to
          // the same buffer slot because the queue is full).
          recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
          return true, true
       }
    }

    if c.qcount > 0 {
       // Receive directly from queue
       qp := chanbuf(c, c.recvx)
       if raceenabled {
          racenotify(c, c.recvx, nil)
       }
       if ep != nil {
          typedmemmove(c.elemtype, ep, qp)
       }
       typedmemclr(c.elemtype, qp)
       c.recvx++
       if c.recvx == c.dataqsiz {
          c.recvx = 0
       }
       c.qcount--
       unlock(&c.lock)
       return true, true
    }

    if !block {
       unlock(&c.lock)
       return false, false
    }

    // no sender available: block on this channel.
    gp := getg()
    mysg := acquireSudog()
    mysg.releasetime = 0
    if t0 != 0 {
       mysg.releasetime = -1
    }
    // No stack splits between assigning elem and enqueuing mysg
    // on gp.waiting where copystack can find it.
    mysg.elem = ep
    mysg.waitlink = nil
    gp.waiting = mysg

    mysg.g = gp
    mysg.isSelect = false
    mysg.c = c
    gp.param = nil
    c.recvq.enqueue(mysg)
    if c.timer != nil {
       blockTimerChan(c)
    }

    // Signal to anyone trying to shrink our stack that we're about
    // to park on a channel. The window between when this G's status
    // changes and when we set gp.activeStackChans is not safe for
    // stack shrinking.
    gp.parkingOnChan.Store(true)
    gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanReceive, traceBlockChanRecv, 2)

    // someone woke us up
    if mysg != gp.waiting {
       throw("G waiting list is corrupted")
    }
    if c.timer != nil {
       unblockTimerChan(c)
    }
    gp.waiting = nil
    gp.activeStackChans = false
    if mysg.releasetime > 0 {
       blockevent(mysg.releasetime-t0, 2)
    }
    success := mysg.success
    gp.param = nil
    mysg.c = nil
    releaseSudog(mysg)
    return true, success
}
```

- **竞态检测**：如果启用了竞态检测，`ep` 总是在堆或调用者的栈上，因此不需要检查 `ep` 的有效性。
- **通道为空检查**：
  - 如果通道 `c` 为 `nil` 且 `block` 为 `false`，则返回。
  - 如果 `c` 有定时器，调用 `maybeRunChan()` 来检查是否需要执行定时任务。
- **快速路径**：
  - 如果 `block` 为 `false` 并且通道为空：
    - 检查通道是否关闭，如果没有关闭，返回。
    - 如果通道已经关闭并且没有待接收的数据，清空 `ep`（如果不为 `nil`），并返回 `(true, false)`。
- **获取锁**：
  - 调用 `lock(&c.lock)` 锁定通道，以保护后续操作。
- **处理关闭的通道**：
  - 如果通道已关闭且缓冲区为空，释放锁并清空 `ep`，返回 `(true, false)`。
  - 如果通道关闭但缓冲区有数据，继续处理。
- **处理等待的发送者**：
  - 如果找到了等待的发送者，直接接收数据并解锁通道。
- **从缓冲区接收数据**：
  - 如果缓冲区有数据，从队列中接收数据并清空相应的缓冲区位置。
- **阻塞等待发送者**：
  - 如果没有发送者且 `block` 为 `true`，则阻塞当前 goroutine。
  - 创建一个 `sudog` 结构体，分配给 `gp.waiting`，将接收操作的上下文保存在其中。
- **设置计时器**：如果通道有定时器，调用 `blockTimerChan` 进行处理。
- **信号与唤醒**：
  - 使用 `gopark` 阻塞当前 goroutine，直到有数据可接收。
  - 唤醒后，检查 `mysg` 是否仍然在等待列表中，处理定时器。



#### 4.1 存在等待发送的 Goroutine

如果等待发送的队列`sendq`里存在挂起的`Goroutine`，那么有两种情况：当前`channel`无缓冲区，或者当前`channel`已满。从`sendq`中取出最先阻塞的`Goroutine`，然后调用`recv`方法：

```Go
// recv processes a receive operation on a full channel c.
// There are 2 parts:
//  1. The value sent by the sender sg is put into the channel
//     and the sender is woken up to go on its merry way.
//  2. The value received by the receiver (the current G) is
//     written to ep.
//
// For synchronous channels, both values are the same.
// For asynchronous channels, the receiver gets its data from
// the channel buffer and the sender's data is put in the
// channel buffer.
// Channel c must be full and locked. recv unlocks c with unlockf.
// sg must already be dequeued from c.
// A non-nil ep must point to the heap or the caller's stack.
func recv(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
    if c.dataqsiz == 0 {
       if raceenabled {
          racesync(c, sg)
       }
       if ep != nil {
          // copy data from sender
          recvDirect(c.elemtype, sg, ep)
       }
    } else {
       // Queue is full. Take the item at the
       // head of the queue. Make the sender enqueue
       // its item at the tail of the queue. Since the
       // queue is full, those are both the same slot.
       qp := chanbuf(c, c.recvx)
       if raceenabled {
          racenotify(c, c.recvx, nil)
          racenotify(c, c.recvx, sg)
       }
       // copy data from queue to receiver
       if ep != nil {
          typedmemmove(c.elemtype, ep, qp)
       }
       // copy data from sender to queue
       typedmemmove(c.elemtype, qp, sg.elem)
       c.recvx++
       if c.recvx == c.dataqsiz {
          c.recvx = 0
       }
       c.sendx = c.recvx // c.sendx = (c.sendx+1) % c.dataqsiz
    }
    sg.elem = nil
    gp := sg.g
    unlockf()
    gp.param = unsafe.Pointer(sg)
    sg.success = true
    if sg.releasetime != 0 {
       sg.releasetime = cputicks()
    }
    goready(gp, skip+1)
}
```

- 如果无缓冲区，那么直接从`sender`接收数据；
- 如果缓冲区已满，从`buf`队列的头部接收数据，并把`sender`的数据加到buf队列的尾部；
- 最后调用`goready`函数将等待发送数据的`Goroutine`的状态从`_Gwaiting`置为`_Grunnable`，等待下一次调度。

![img](https://github.com/the-web3/golang-language/tree/main/docs/imgs/7.png)

##### 4.1.1 缓冲区buf中还有数据

如果缓冲区`buf`中还有元素，那么就走正常的接收，将从`buf`中取出的元素拷贝到当前协程的接收数据目标内存地址中。值得注意的是，即使此时`channel`已经关闭，仍然可以正常地从缓冲区`buf`中接收数据。这种情况比较简单，示意图就不画了。



#### 4.2 阻塞接收

如果是阻塞模式，且当前没有数据可以接收，那么就需要将当前`Goroutine`打包成一个`sudog`加入到`channel`的等待接收队列`recvq`中，将当前`Goroutine`的状态置为`_Gwaiting`，等待唤醒。示意图如下：

![img](https://github.com/the-web3/golang-language/tree/main/docs/imgs/8.png)

如果之后当前`Goroutine`被调度器唤醒，则执行清理工作并最终释放对应的`sudog`结构体。



### 5. Channel 关闭

```Go
func closechan(c *hchan) {
    if c == nil {
       panic(plainError("close of nil channel"))
    }

    lock(&c.lock)
    if c.closed != 0 {
       unlock(&c.lock)
       panic(plainError("close of closed channel"))
    }

    if raceenabled {
       callerpc := getcallerpc()
       racewritepc(c.raceaddr(), callerpc, abi.FuncPCABIInternal(closechan))
       racerelease(c.raceaddr())
    }

    c.closed = 1

    var glist gList

    // release all readers
    for {
       sg := c.recvq.dequeue()
       if sg == nil {
          break
       }
       if sg.elem != nil {
          typedmemclr(c.elemtype, sg.elem)
          sg.elem = nil
       }
       if sg.releasetime != 0 {
          sg.releasetime = cputicks()
       }
       gp := sg.g
       gp.param = unsafe.Pointer(sg)
       sg.success = false
       if raceenabled {
          raceacquireg(gp, c.raceaddr())
       }
       glist.push(gp)
    }

    // release all writers (they will panic)
    for {
       sg := c.sendq.dequeue()
       if sg == nil {
          break
       }
       sg.elem = nil
       if sg.releasetime != 0 {
          sg.releasetime = cputicks()
       }
       gp := sg.g
       gp.param = unsafe.Pointer(sg)
       sg.success = false
       if raceenabled {
          raceacquireg(gp, c.raceaddr())
       }
       glist.push(gp)
    }
    unlock(&c.lock)

    // Ready all Gs now that we've dropped the channel lock.
    for !glist.empty() {
       gp := glist.pop()
       gp.schedlink = 0
       goready(gp, 3)
    }
}
```

- 关闭`channel`时，会遍历`recvq`和`sendq`（实际只有`recvq`或者`sendq`），取出`sudog`中挂起的`Goroutine`加入到`glist`列表中，并清除`sudog`上的一些信息和状态。
- 然后遍历`glist`列表，为每个`Goroutine`调用`goready`函数，将所有`Goroutine`置为`_Grunnable`状态，等待调度。
- 当`Goroutine`被唤醒之后，会继续执行`chansend`和`chanrecv`函数中当前`Goroutine`被唤醒后的剩余逻辑。



## 三. 小结

本文讲解了 Channel 的基本用法和简单的代码实战，同时深入讲解了 Channel 的底层实现原理。

