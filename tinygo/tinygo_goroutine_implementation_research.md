# TinyGo Goroutine 实现调研

## 概述

TinyGo 针对嵌入式/WASM 场景，放弃了标准 Go 的动态栈增长和抢占式调度，采用固定栈 + 协作式调度。核心实现分三部分：

1. **编译期**：`compiler/goroutine.go` - 生成 wrapper + 参数打包
2. **栈初始化**：`src/internal/task/task_stack.go` + 架构相关代码 - 栈分配 + 寄存器初始化
3. **调度运行**：`src/runtime/scheduler_*.go` - 协作式/多核调度器

## 编译期转换

### go 语句的 IR 转换

`compiler/goroutine.go:16 createGo()`

```go
// 输入 SSA: go add(x, y)
// 输出 LLVM IR:
//   %params = pack(x, y)  // emitPointerPack
//   %wrapper = add$gowrapper
//   %stacksize = getGoroutineStackSize() | DefaultStackSize
//   call @internal/task.start(%wrapper, %params, %stacksize)
```

关键点：
- 所有参数打包成一个连续内存块（避免可变参数传递）
- 生成 wrapper 统一接口为 `func(*unsafe.Pointer)`
- 栈大小：静态分析（`-opt=2`）或编译时常量

### Wrapper 生成：接口适配层

#### 设计动机

运行时启动接口固定为 `func start(fn uintptr, args unsafe.Pointer, stackSize uintptr)`，要求：
- `fn` 必须是 `func(*unsafe.Pointer)` 类型
- 只接受单个指针参数
- 这是汇编层 `tinygo_startTask` 的硬编码约定

但用户代码千变万化：
```go
go func(x, y int) { ... }          // 多参数
go obj.Method(s string)            // 方法调用
go funcVar(a, b, c int)            // 函数指针
go func() { println(captured) }()  // 闭包
```

TinyGo 方案：
```go
// 编译时生成
func add$gowrapper(ptr *unsafe.Pointer) {
    args := (*struct{x, y int64})(ptr)
    add(args.x, args.y)  // 零开销
}
```

优势：
- 编译时确定类型 → 零运行时开销
- 内联后等价于直接调用
- 不依赖 reflect
- 二进制体积小

代价：
- 每个不同签名生成一个 wrapper
- 稍增大二进制（但远小于 reflect）

#### Wrapper 的本质

**编译时类型擦除 + 运行时类型恢复**：

```
编译时：
  go add(x, y) → pack{x, y} + add$gowrapper

运行时：
  task.start → tinygo_startTask → add$gowrapper(pack) → unpack → add(x, y)
```

这是 TinyGo "复杂度从运行时转移到编译时" 的典型体现。

## 栈初始化

### 内存布局

`src/internal/task/task_stack.go:41 state.initialize()`

```
高地址 (stack + stackSize)
├─────────────────────────┤
│ calleeSavedRegs (56B)   │ ← sp (初始)
│   rbx, rbp, r12-r15     │
│   pc = &tinygo_startTask│
├─────────────────────────┤
│                         │
│   未使用栈空间           │
│                         │
├─────────────────────────┤
│ canary (8B)             │ ← canaryPtr
│ = 0x670c1333b83bf575    │
└─────────────────────────┘
低地址 (stack)
```

实现：
```go
stack := runtime_alloc(stackSize, nil)
*(*uintptr)(stack) = stackCanary  // 栈底哨兵
r := (*calleeSavedRegs)(unsafe.Add(stack, stackSize-unsafe.Sizeof(calleeSavedRegs{})))
s.archInit(r, fn, args)
```

### 架构相关初始化

`src/internal/task/task_stack_amd64.go:24`

```go
func (s *state) archInit(r *calleeSavedRegs, fn uintptr, args unsafe.Pointer) {
    s.sp = uintptr(unsafe.Pointer(r))
    r.pc = uintptr(unsafe.Pointer(&startTask))  // 汇编入口
    r.r12 = fn      // wrapper 函数指针
    r.r13 = uintptr(args)  // 参数包
}
```

寄存器约定：
- **AMD64**: r12=fn, r13=args
- **ARM**: r4=fn, r5=args
- **ARM64**: x19=fn, x20=args

## 上下文切换

执行流：
```
scheduler → swapTask → tinygo_startTask → wrapper → 用户函数
                              ↑                          ↓
                              └────── tinygo_task_exit ──┘
                                      (调用 Pause)
```

### swapTask 实现

`src/internal/task/task_stack_amd64.S:45`

```asm
tinygo_swapTask:
    # 参数: %rdi=newStack, %rsi=&oldStack
    pushq %r15; pushq %r14; pushq %r13; pushq %r12
    pushq %rbp; pushq %rbx
    movq %rsp, (%rsi)     # 保存当前 SP
    movq %rdi, %rsp       # 切换栈
    popq %rbx; popq %rbp; popq %r12; popq %r13
    popq %r14; popq %r15
    ret                   # 跳转到新任务
```

关键：
- 仅保存 callee-saved 寄存器（ABI 保证 caller-saved 由调用者处理）
- `ret` 弹出栈顶作为返回地址：
  - 新 goroutine → `tinygo_startTask`
  - 恢复的 goroutine → `swapTask` 调用后的下一条指令

### Goroutine 启动

`src/internal/task/task_stack_amd64.S:3`

```asm
tinygo_startTask:
    .cfi_undefined rip    # 标记栈底（调试器用）
    movq %r13, %rdi       # args → 第一参数
    callq *%r12           # 调用 wrapper
    jmp tinygo_task_exit  # 自动退出
```

## 调度器

### 协作式调度器 (scheduler.tasks)

`src/runtime/scheduler_cooperative.go:138`

数据结构：
```go
var (
    runqueue   task.Queue   // 就绪队列 (FIFO 链表)
    sleepQueue *task.Task   // 睡眠队列 (按唤醒时间排序)
    timerQueue *timerNode   // 定时器队列
)
```

调度循环核心：
```go
func scheduler(returnAtDeadlock bool) {
    for !mainExited {
        now := ticks()

        // 1. 唤醒到期的睡眠任务
        if sleepQueue != nil && now >= sleepQueue.Data {
            runqueue.Push(sleepQueue)
            sleepQueue = sleepQueue.Next
        }

        // 2. 触发到期定时器
        if timerQueue != nil && now >= timerQueue.whenTicks() {
            timerQueue.callback(...)
        }

        // 3. 执行就绪任务
        if t := runqueue.Pop(); t != nil {
            t.Resume()  // → swapTask(t.sp, &systemStack)
            continue
        }

        // 4. 计算下次唤醒时间并休眠
        sleepTicks(min(sleepQueue.wakeup, timerQueue.wakeup) - now)
    }
}
```

让出点：
- `time.Sleep()` → `addSleepTask` + `Pause`
- `runtime.Gosched()` → `runqueue.Push(current)` + `Pause`
- Channel 阻塞 → `Pause`
- Mutex 竞争 → `Pause`

限制：无抢占，长时间计算会饿死其他 goroutine。

### 多核调度器 (scheduler.cores)

`src/runtime/scheduler_cores.go:162`

增强：
```go
var (
    cpuTasks [numCPU]*task.Task  // 每核当前任务
    schedulerLock Mutex          // 全局锁
)

const (
    RunStatePaused   = 0  // 已暂停
    RunStateRunning  = 1  // 运行中
    RunStateResuming = 2  // 运行中但待恢复（伪抢占标记）
)
```

伪抢占机制：
```go
func scheduleTask(t *task.Task) {
    schedulerLock.Lock()
    if t.RunState == RunStateRunning {
        t.RunState = RunStateResuming  // 标记
        // 下次 Pause() 会检查并立即恢复
    } else {
        runqueue.Push(t)
    }
    schedulerLock.Unlock()
}
```

实现细节：
- 进入调度器前必须持有 `schedulerLock`
- Resume 前解锁，返回后重新加锁（避免任务执行时持锁）
- 核间唤醒：`schedulerWake()` 发送 IPI

## 关键数据结构

### Task

`src/internal/task/task.go:8`

```go
type Task struct {
    Next  *Task           // 队列链接
    Ptr   unsafe.Pointer  // 通用指针（channel 等待队列等）
    Data  uint64          // 通用数据（时间戳等）
    state state           // {sp, canaryPtr}
    RunState uint8
    DeferFrame unsafe.Pointer
    gcData gcData
}
```

字段复用：
- `Data` 在 sleepQueue 中存时间戳
- `Ptr` 在 channel 中存等待节点

### Queue

`src/internal/task/queue.go:9`

```go
type Queue struct {
    head, tail *Task
}

func (q *Queue) Push(t *Task) {
    mask := lockAtomics()  // 禁中断或自旋锁
    if q.tail != nil { q.tail.Next = t }
    q.tail = t; t.Next = nil
    if q.head == nil { q.head = t }
    unlockAtomics(mask)
}
```

无锁优化：单核通过中断禁用，多核用 `atomicsLock` 自旋锁。

## 与标准 Go 差异

| 特性 | 标准 Go | TinyGo |
|-----|---------|--------|
| 栈策略 | 连续栈，动态增长 | 固定栈，编译时确定 |
| 栈溢出处理 | 拷贝栈 + 指针调整 | Panic (canary 检测) |
| 调度 | 抢占式 (信号/协作) | 纯协作式 (tasks) / 伪抢占 (cores) |
| 创建开销 | ~150 cycles | ~100 cycles |
| 切换开销 | 10-30 cycles | 10-20 cycles |
| 最大并发 | 百万级 | 数千级 (内存限制) |

核心 tradeoff：
- 放弃栈增长 → 可预测内存，但需人工调优
- 放弃抢占 → 简化运行时，但需程序员自律
- 固定调度策略 → 减小二进制体积


## 栈溢出检测机制

检测点：每次切换回调度器时

```go
// task_stack.go:47
func (s *state) resume() {
    swapTask(s.sp, &systemStack)
    if *s.canaryPtr != stackCanary {
        runtimePanic("stack overflow")
    }
}
```

Canary 值选择：随机数，避免被意外数据匹配。
d64.S`
- `src/runtime/scheduler_cooperative.go`
