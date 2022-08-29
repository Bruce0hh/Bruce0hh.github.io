[TOC]

# 并发编程

## Context

### context.Context 实现方法

- Deadline：返回 context 被取消的时间，即完成工作的截止日期
- Done：返回一个 Channel，这个 Channel 会在当前工作完成或 context 被取消后关闭，多次调用 Done 后会返回同一个 Channel 。
- Err：返回 context 结束的原因，它只会在 Done 方法对应的 Channel 关闭时返回非空值
  - 如果 context 被取消，会返回 Canceled 错误；
  - 如果 context 超时，会返回 DeadlineExceeded 错误。
- Value：从 context 中获取键对应的值，对于同一个上下文来说，多次调用 Value 并传入相同的 Key 会返回相同的结果，该方法可以用来传递特定的数据。

### 设计原理

- contex.Context 最大作用是，在 Goroutine 构成的树形结构中同步信号以减少计算资源的浪费。
- 我们可能会创建多个 Goroutine 来处理一次请求，而 context 的作用是在不同的 Goroutine 之间同步请求特定数据、取消信号以及处理请求的截止日期。
- 每一个 context 都会从最顶层的 Goroutine 逐层传递到最底层。context 可以在上层 Goroutine 执行出现错误时将信号及时同步给下层。当最上层的 Goroutine 因为某些原因执行失败时，下层的 Goroutine 由于没有接收到这个信号所以会继续工作。但是当我们正确使用 context 时，就可以在下层及时停掉无用的工作以减少额外的资源消耗。

## 同步原语和锁

### 基本原语

### Mutex

```go
type Mutex struct {
    state int32	 //表示当前互斥锁的状态
    sema  uint32 //用于控制锁状态的信号量
}
```

#### mutex 状态（4个状态位）

- 默认所有状态位都为0，int32 中不同位分别表示不同状态。
- mutexLocked 表示互斥锁的锁定状态；
- mutexWoken 表示被从正常模式唤醒；
- mutexStarving 表示当前互斥锁进入饥饿状态；
- waiterCount 表示当前互斥锁上等待的 Goroutine 个数。

#### 正常模式和饥饿模式

- **正常模式**：锁的等待者会按照先进先出的顺序获取锁。但是刚被唤醒的 Goroutine 与新创建的 Goroutine 竞争时，大概率获取不到锁。为了减少这种情况的出现，一旦 Goroutine 超过 1ms 没有获取到锁，它就会将当前互斥锁切换到饥饿模式，防止部分 Goroutine 被饿死。
- **饥饿模式**：饥饿模式下，互斥锁会直接交给等待队列最前面的 Goroutine。新的 Goroutine 在该状态下不能获取锁，也不会进入自旋状态，它们只会在队列末尾等待。如果一个 Goroutine 获得了互斥锁并且它在队列末尾或者它等待的时间少于 1ms，那么当前互斥锁就会切换回正常模式。
- 正常模式下的互斥锁能够提供更好的性能；饥饿模式则能避免 Goroutine 由于陷入等待无法获取锁而造成的高**尾延时**。

#### **加锁和解锁**

加锁 `sync.Mutex.Lock`，当锁的状态是0时，将`mutexLocked`设置成1；如果互斥锁的状态不是0，就会调用`sync.Mutex.lockSlow`尝试通过**自旋**等方式等待锁的释放。

**获取锁过程：**

1. 判断当前 Goroutine 能否进入自旋；
2. 通过自旋等待互斥锁的释放；
3. 计算互斥锁的最新状态；
4. 更新互斥锁的状态并获取锁。

**自旋条件：**

- 互斥锁只有在普通模式下才能进入自旋；
- `runtime.sync_runtime_canSpin`需要返回 true：
  - 在有多个 CPU 的机器上运行；
  - 当前 Goroutine 为了获取该锁进入自旋的次数少于4；
  - 当前机器上至少存在一个正在运行的处理器 P 并且处理的运行队列为空。



