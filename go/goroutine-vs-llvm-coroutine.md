# Goroutine 与 LLVM Coroutine 原理及 ABI 对比

## 1. 概述

| 特性 | Goroutine | LLVM Coroutine |
|------|-----------|----------------|
| 类型 | 有栈协程 (Stackful) | 无栈协程 (Stackless) |
| 实现层 | 运行时 | 编译器 |
| 栈模型 | 独立栈 | 无独立栈 |
| 挂起能力 | 任意位置 | 仅限 suspend point |

---

## 2. Goroutine 实现原理

### 2.1 核心数据结构

```go
// G - Goroutine
type g struct {
    stack       stack   // 栈范围 [lo, hi)
    stackguard0 uintptr // 栈溢出检查

    sched       gobuf   // 挂起时保存的上下文
    atomicstatus uint32 // 状态

    m           *m      // 绑定的 M
    // ...
}

// gobuf - 执行上下文
type gobuf struct {
    sp   uintptr  // 栈指针
    pc   uintptr  // 程序计数器
    g    guintptr // G 指针
    bp   uintptr  // 帧指针
    // ...
}
```

### 2.2 栈结构

每个 Goroutine 拥有独立的连续栈：

```
┌────────────────────────────┐ stack.hi
│        可用空间             │
├────────────────────────────┤
│  栈帧 N (当前函数)          │
│  ├── 返回地址              │
│  ├── 局部变量              │
│  └── spill 区域            │
├────────────────────────────┤
│  栈帧 N-1                  │
├────────────────────────────┤
│  ...                       │
├────────────────────────────┤ <- SP
│        栈保护区             │
└────────────────────────────┘ stack.lo
```

### 2.3 挂起机制 (mcall)

```asm
TEXT runtime·mcall(SB), NOSPLIT, $0-8
    ; 保存当前上下文到 gobuf
    MOVQ    0(SP), AX                    ; 返回地址
    MOVQ    AX, gobuf_pc(R14)            ; 保存 PC
    LEAQ    8(SP), AX
    MOVQ    AX, gobuf_sp(R14)            ; 保存 SP
    MOVQ    BP, gobuf_bp(R14)            ; 保存 BP

    ; 切换到 g0 栈
    MOVQ    g_m(R14), BX
    MOVQ    m_g0(BX), SI
    MOVQ    gobuf_sp(SI), SP

    ; 调用调度函数
    CALL    fn(FP)
```

**关键点**：只保存 SP/PC/BP，不保存通用寄存器。编译器确保挂起前所有活跃值已 spill 到栈。

### 2.4 恢复机制 (gogo)

```asm
TEXT runtime·gogo(SB), NOSPLIT, $0-8
    MOVQ    buf+0(FP), BX        ; gobuf 指针

    ; 恢复上下文
    MOVQ    gobuf_sp(BX), SP     ; 恢复 SP
    MOVQ    gobuf_bp(BX), BP     ; 恢复 BP
    MOVQ    gobuf_pc(BX), AX     ; 恢复 PC

    ; 设置 G 指针
    MOVQ    gobuf_g(BX), R14

    ; 跳转到保存的 PC
    JMP     AX
```

**关键点**：直接跳转到任意 PC 地址继续执行。

### 2.5 栈增长

```go
// 编译器在函数入口插入栈检查
func someFunc() {
    // if SP < stackguard0 { morestack() }
}
```

```asm
TEXT ·someFunc(SB), $128-0
    MOVQ    (TLS), CX            ; 当前 G
    CMPQ    SP, stackguard0(CX)  ; 检查栈空间
    JLS     need_more            ; 空间不足则扩容

    ; 函数体...

need_more:
    CALL    runtime·morestack(SB)
    JMP     someFunc(SB)         ; 重新执行
```

---

## 3. LLVM Coroutine 实现原理

### 3.1 编译器变换

LLVM Coroutine 是**纯编译器变换**，将协程函数转换为状态机：

```
源代码                              编译后
─────────────────────────────────────────────────────────
task<int> foo() {                  foo() → frame*
    int x = 1;                     foo.resume(frame*)
    co_await bar();      ──►       foo.destroy(frame*)
    co_return x + 1;                    +
}                                  coroutine_frame 结构
```

### 3.2 协程帧结构

```cpp
struct foo_frame {
    // 固定头部
    void (*resume_fn)(foo_frame*);   // +0: resume 入口
    void (*destroy_fn)(foo_frame*);  // +8: destroy 入口
    int32_t index;                    // +16: 挂起点索引

    // promise 对象
    promise_type promise;

    // 跨挂起存活的变量
    int x;
    // ...
};
```

### 3.3 函数分割 (CoroSplit Pass)

原始协程函数被分割为三个函数：

**1. Ramp Function (入口)**：
```cpp
task<int> foo() {
    auto* frame = new foo_frame;
    frame->resume_fn = &foo_resume;
    frame->destroy_fn = &foo_destroy;
    frame->index = 0;

    foo_resume(frame);  // 首次执行

    return task<int>{frame};
}
```

**2. Resume Function (恢复)**：
```cpp
void foo_resume(foo_frame* frame) {
    switch (frame->index) {
    case 0: goto entry;
    case 1: goto resume_1;
    case 2: goto resume_2;
    }

entry:
    frame->x = 1;
    // co_await bar()
    frame->index = 1;
    if (!bar_ready()) {
        bar_suspend(frame);
        return;  // 挂起
    }
    [[fallthrough]];

resume_1:
    // co_return x + 1
    frame->promise.return_value(frame->x + 1);
    frame->index = -1;  // 完成
    return;

resume_2:
    // final suspend
    return;
}
```

**3. Destroy Function (销毁)**：
```cpp
void foo_destroy(foo_frame* frame) {
    // 析构 promise 和保存的变量
    frame->promise.~promise_type();
    delete frame;
}
```

### 3.4 挂起点分析

编译器识别每个 `co_await`/`co_yield` 作为挂起点：

```cpp
task<void> example() {
    int a = 1;           // 挂起点 0 之前
    co_await op1();      // 挂起点 1
    int b = a + 1;       // 挂起点 1 之后
    co_await op2();      // 挂起点 2
    use(b);              // 挂起点 2 之后
}
```

变量活跃性：
- `a`: 跨挂起点 1 存活 → 保存到帧
- `b`: 跨挂起点 2 存活 → 保存到帧

### 3.5 LLVM IR 表示

```llvm
define void @foo() presplitcoroutine {
entry:
    %id = call token @llvm.coro.id(...)
    %size = call i32 @llvm.coro.size.i32()
    %alloc = call ptr @malloc(i32 %size)
    %hdl = call ptr @llvm.coro.begin(token %id, ptr %alloc)

    ; 协程体
    %x = add i32 1, 0

    ; 挂起点
    %save = call token @llvm.coro.save(ptr %hdl)
    %sus = call i8 @llvm.coro.suspend(token %save, i1 false)
    switch i8 %sus, label %suspend [
        i8 0, label %resume
        i8 1, label %cleanup
    ]

resume:
    ; 继续执行...

cleanup:
    %mem = call ptr @llvm.coro.free(token %id, ptr %hdl)
    call void @free(ptr %mem)
    br label %suspend

suspend:
    call i1 @llvm.coro.end(ptr %hdl, i1 false)
    ret void
}
```

---

## 4. 核心原理对比

### 4.1 变量保存机制

#### Go：Spill 到栈

**原理**：编译器在函数入口将寄存器参数 spill（溢出）到栈上，挂起时栈帧保持不变。

```
参数 (寄存器)  ──►  函数入口 spill  ──►  栈帧 (持久)  ──►  挂起时保持  ──►  恢复后原地继续
     AX, BX           MOVQ AX, -8(SP)       [a, b, x, ...]      栈帧不变          MOVQ -8(SP), AX
```

```asm
; Go 编译生成
TEXT foo(SB), ABIInternal, $32-0
    MOVQ    AX, a-8(SP)       ; 入口 spill（保守：全部 spill）
    MOVQ    BX, b-16(SP)

    ; ... 函数体 ...

    CALL    runtime.gopark    ; 挂起时栈帧完整保持

    MOVQ    a-8(SP), AX       ; 恢复后从原位置读取
```

**特点**：
- 保守策略：所有参数/局部变量都在栈上
- 不做活跃性分析
- 栈帧在整个生命周期内持久存在

#### LLVM Coroutine：LVA (Live Variable Analysis)

**原理**：编译器分析哪些变量**跨越挂起点存活**，只保存这些变量到协程帧。

```
源代码            LVA 分析              协程帧
───────────────────────────────────────────────────────
int x = a + b;    x: 跨挂起存活        struct frame {
int temp = x * 2; temp: 挂起前死亡  →      int x;    // 只有 x
use(temp);        a, b: 挂起前死亡        // 无 temp, a, b
co_await ...;
return x;
```

**LVA 分析过程**：

```
1. 构建 CFG:  entry → co_await → return
2. 计算活跃变量:
   - LiveIn[return] = {x}
   - LiveOut[co_await] = {x}
3. 跨挂起存活 = LiveOut[挂起点] = {x}
4. 帧布局: frame.x @ offset N
```

#### 对比

| 方面 | Go Spill | LLVM LVA |
|------|----------|----------|
| 策略 | 保守（全部保存） | 精确（只保存跨挂起） |
| 分析 | 无 | 数据流分析 |
| 保存位置 | 栈 | 协程帧（堆） |
| 空间 | 较大 | 最小化 |

---

### 4.2 控制流恢复机制

#### Go：PC 直接跳转

**原理**：保存程序计数器 (PC)，恢复时直接跳转到任意地址。

```
挂起:                              恢复:
─────────────────────────────      ─────────────────────────────
MOVQ  ret_addr, gobuf_pc(G)        MOVQ  gobuf_pc(G), AX
MOVQ  SP, gobuf_sp(G)              MOVQ  gobuf_sp(G), SP
                                   JMP   AX   ; 跳转到任意 PC
```

**特点**：
- PC 可以是任意函数内部地址
- 恢复点不固定
- 支持任意调用深度挂起

```
调用链: A() → B() → C() → gopark()
                           ↓
                    保存 PC = C 内部某行
                           ↓
                    恢复时 JMP 到 C 继续
```

#### LLVM Coroutine：状态机 Switch

**原理**：挂起点编号为状态，恢复时通过 switch 分发到对应位置。

```cpp
void foo_resume(foo_frame* frame) {
    switch (frame->index) {     // 状态分发
        case 0: goto entry;
        case 1: goto resume_1;  // 固定的恢复点
        case 2: goto resume_2;
    }
}
```

**特点**：
- 恢复入口固定（resume 函数）
- 状态是整数索引
- 只能在 co_await 点挂起

#### 对比

| 方面 | Go PC 跳转 | LLVM Switch |
|------|------------|-------------|
| 恢复入口 | 任意 PC 地址 | 固定 resume 函数 |
| 分发方式 | 直接 JMP | switch 语句 |
| 挂起位置 | 任意 | 仅 co_await 点 |
| 分支预测 | 间接跳转（较慢） | switch（可优化） |

---

### 4.3 栈帧生命周期

#### Go：栈帧持久

```
时间 ──────────────────────────────────────────────────►

调用     栈帧创建      挂起        恢复         返回
 │          │           │          │            │
 ▼          ▼           ▼          ▼            ▼
        ┌───────┐   ┌───────┐  ┌───────┐   ┌───────┐
        │ 栈帧  │   │ 栈帧  │  │ 栈帧  │   │ 栈帧  │
        │ 创建  │   │ 保持  │  │ 同一个│   │ 销毁  │
        └───────┘   └───────┘  └───────┘   └───────┘
              │           │          │
              └───────────┴──────────┘
                    栈帧存活期间
```

**特点**：
- 栈帧从创建到返回一直存在
- 挂起期间占用栈空间
- 所有变量位置不变

#### LLVM Coroutine：栈帧瞬态

```
时间 ──────────────────────────────────────────────────────────────────────►

创建协程    首次执行      挂起        恢复执行       完成
    │          │           │            │            │
    ▼          ▼           ▼            ▼            ▼
┌───────┐  ┌───────┐   ┌───────┐   ┌───────┐   ┌───────┐
│ 帧对象 │  │ 栈帧1 │   │ (无)  │   │ 栈帧2 │   │ (无)  │
│ 分配  │  │ 创建  │   │       │   │ 创建  │   │       │
└───────┘  └───────┘   └───────┘   └───────┘   └───────┘
    │          │           │            │            │
    └──────────┴───────────┴────────────┴────────────┘
                      帧对象存活期间
                     （栈帧随用随建）
```

**特点**：
- 帧对象持久，栈帧瞬态
- 挂起时栈帧消失
- 恢复时创建新栈帧

#### 对比

| 方面 | Go | LLVM Coroutine |
|------|-----|----------------|
| 栈帧生命 | 函数全程 | 仅执行期间 |
| 挂起时 | 栈帧保持 | 栈帧销毁 |
| 内存模型 | 每协程独立栈 | 共享调用栈 + 帧对象 |
| 栈空间 | 持续占用 | 动态复用 |

---

### 4.4 挂起点识别

#### Go：隐式挂起点

**原理**：任何可能调用 runtime 函数的点都是潜在挂起点。

```go
func foo() {
    x := 1
    bar()           // 可能挂起（bar 内部可能调用 gopark）
    println(x)
    ch <- 1         // 可能挂起（channel send）
    mutex.Lock()    // 可能挂起（锁竞争）
    time.Sleep(1)   // 必定挂起
}
```

**编译器处理**：
- 不需要识别具体挂起点
- 所有函数调用前，活跃值已 spill
- 调用约定保证恢复后状态正确

#### LLVM Coroutine：显式挂起点

**原理**：只有 `co_await`/`co_yield`/`co_return` 是挂起点。

```cpp
task<void> foo() {
    int x = 1;
    bar();              // 不是挂起点！（普通函数）
    println(x);
    co_await ch.send(); // 显式挂起点
    co_await lock();    // 显式挂起点
    co_await sleep(1s); // 显式挂起点
}
```

**编译器处理**：
- 静态识别所有挂起点
- 为每个挂起点生成状态
- 只在挂起点做变量保存

#### 对比

| 方面 | Go | LLVM Coroutine |
|------|-----|----------------|
| 挂起点 | 隐式（任意调用） | 显式（co_await） |
| 识别方式 | 运行时 | 编译时 |
| 普通函数 | 可能挂起 | 不可挂起 |
| 调用约定 | 统一处理 | 协程/非协程区分 |

---

### 4.5 嵌套调用挂起

#### Go：支持任意深度

```go
func A() {
    x := 1
    B()      // B 内部可能挂起
    use(x)   // x 仍然可用
}

func B() {
    y := 2
    C()      // C 内部可能挂起
    use(y)   // y 仍然可用
}

func C() {
    gopark()  // 实际挂起点
}
```

**栈状态**：
```
挂起时栈:
┌─────────┐
│  A 栈帧  │  x 在这里
├─────────┤
│  B 栈帧  │  y 在这里
├─────────┤
│  C 栈帧  │  gobuf.pc 指向这里
└─────────┘

恢复后：整个栈保持，A/B/C 都能继续
```

#### LLVM Coroutine：需要传染

```cpp
task<void> A() {
    int x = 1;
    co_await B();  // 必须 co_await
    use(x);
}

task<void> B() {
    int y = 2;
    co_await C();  // 必须 co_await
    use(y);
}

task<void> C() {
    co_await suspend();  // 挂起
}
```

**帧状态**：
```
三个独立的协程帧:
┌─────────┐
│ A 的帧  │  x 在这里
└─────────┘
┌─────────┐
│ B 的帧  │  y 在这里
└─────────┘
┌─────────┐
│ C 的帧  │  状态在这里
└─────────┘

每个帧独立管理，A 等待 B，B 等待 C
```

**问题**：如果 B 是普通函数：

```cpp
task<void> A() {
    int x = 1;
    B();           // 普通调用
    use(x);        // 如果 B 内部挂起，这里不会执行！
}

void B() {         // 普通函数
    C_sync();      // 不能 co_await
    // 如果想挂起，做不到！
}
```

#### 对比

| 方面 | Go | LLVM Coroutine |
|------|-----|----------------|
| 嵌套挂起 | 透明支持 | 需要 co_await 链 |
| 普通函数内挂起 | 可以 | 不可以 |
| 调用链保存 | 自动（栈） | 手动（多个帧） |
| API 设计 | 阻塞风格 | 异步风格 |

---

### 4.6 栈增长 vs 固定帧

#### Go：动态栈增长

```go
func recursive(n int) {
    if n == 0 { return }
    var buf [1024]byte  // 大局部变量
    use(buf)
    recursive(n - 1)    // 递归
}
```

**栈管理**：
```
初始栈: 2KB
       ┌─────────┐
       │ 可用    │
       ├─────────┤
       │ 栈帧1   │
       └─────────┘

递归深了:
       ┌─────────┐  ← 检测到 SP < stackguard
       │ 满了！   │
       └─────────┘
             │
             ▼ morestack()
       ┌─────────┐
       │         │
       │ 新栈 4KB │  ← 分配更大的栈
       │         │
       │ 复制内容 │
       └─────────┘
```

**机制**：
```asm
; 函数入口栈检查
CMPQ    SP, stackguard0(G)
JLS     need_more_stack

; ... 函数体 ...

need_more_stack:
    CALL    runtime.morestack
    JMP     function_start  ; 重新执行
```

#### LLVM Coroutine：固定帧大小

```cpp
task<void> foo() {
    char buf[1024];  // 编译时确定大小
    co_await op();
}
```

**帧管理**：
```
编译时确定帧大小:

struct foo_frame {
    void (*resume)(foo_frame*);   // 8
    void (*destroy)(foo_frame*);  // 8
    int index;                     // 4
    char buf[1024];               // 1024
};                                // 总计 ~1044 bytes

运行时一次性分配，不能增长
```

#### 对比

| 方面 | Go | LLVM Coroutine |
|------|-----|----------------|
| 栈/帧大小 | 动态 (2KB-1GB) | 编译时固定 |
| 增长机制 | morestack | 无（重新分配需拷贝） |
| 大局部变量 | 自动处理 | 占用帧空间 |
| VLA | 支持 | 不支持 |

---

### 4.7 GC 与指针追踪

#### Go：栈扫描

```go
func foo() {
    ptr := new(Object)  // 堆指针
    gopark()            // 挂起
    use(ptr)            // ptr 必须被 GC 追踪
}
```

**GC 处理**：
```
1. GC 开始时扫描所有 goroutine 栈
2. 通过 FUNCDATA 找到栈上的指针位置
3. ptr 在栈帧的已知偏移处
4. GC 标记 ptr 指向的对象为活跃

┌─────────────┐
│  foo 栈帧   │
│  ptr ─────────────► Object (标记为活跃)
└─────────────┘
```

#### LLVM Coroutine：帧对象追踪

```cpp
task<void> foo() {
    Object* ptr = new Object();
    co_await op();  // 挂起
    use(ptr);       // ptr 在协程帧中
}
```

**问题**：谁持有帧对象？
```
协程帧在堆上:
┌─────────────┐
│  foo_frame  │
│  ptr ─────────────► Object
└─────────────┘
      ↑
      谁持有这个帧？

通常: task 对象持有
task<void> t = foo();  // t 持有帧指针
                       // t 被追踪 → 帧被追踪 → ptr 被追踪
```

#### 对比

| 方面 | Go | LLVM Coroutine |
|------|-----|----------------|
| 指针位置 | 栈帧（已知布局） | 协程帧（编译器生成） |
| GC 扫描 | 栈扫描 | 帧对象作为根 |
| 所有权 | G 结构体持有栈 | 用户代码持有帧 |
| 追踪难度 | 运行时支持 | 需要用户正确管理 |

---

### 4.8 异常/Panic 处理

#### Go：栈展开

```go
func foo() {
    defer cleanup()
    gopark()      // 挂起
    mayPanic()    // 可能 panic
}
```

**panic 处理**：
```
panic 发生:
1. 沿着栈帧链展开
2. 执行每个帧的 defer
3. 栈帧完整，defer 链正确

┌─────────┐
│  foo    │  defer: cleanup
├─────────┤
│  bar    │  defer: ...
├─────────┤
│  baz    │  defer: ...
└─────────┘
     ↑
   panic 从这里开始展开
```

#### LLVM Coroutine：异常困难

```cpp
task<void> foo() {
    auto guard = ScopeGuard(cleanup);
    co_await op();      // 挂起，栈帧消失
    mayThrow();         // 抛异常
}
```

**问题**：
```
挂起后栈帧消失:
┌─────────────┐
│ resume 调用 │  ← 新栈帧
└─────────────┘

异常抛出时，原来的栈帧不在了！
ScopeGuard 怎么触发？

解决方案: 编译器在每个挂起点后插入 try-catch
```

#### 对比

| 方面 | Go | LLVM Coroutine |
|------|-----|----------------|
| 机制 | 栈展开 | try-catch 包装 |
| defer/RAII | 自然工作 | 需要特殊处理 |
| 实现复杂度 | 运行时支持 | 编译器生成代码 |

---

### 4.9 总结表

| 原理方面 | Go Goroutine | LLVM Coroutine |
|----------|--------------|----------------|
| 变量保存 | Spill 到栈（保守） | LVA（精确） |
| 恢复机制 | PC 直接跳转 | Switch 状态机 |
| 栈帧生命 | 持久 | 瞬态 |
| 挂起点 | 隐式（任意） | 显式（co_await） |
| 嵌套挂起 | 透明 | 需传染 |
| 栈增长 | 动态 | 固定 |
| GC 集成 | 栈扫描 | 帧对象追踪 |
| 异常处理 | 栈展开 | try-catch |

---

## 5. ABI 调用差异

### 5.1 函数签名

| Goroutine | LLVM Coroutine |
|-----------|----------------|
| 签名不变 | 原函数分割为 3 个函数 |
| `func foo(a int) int` | `foo() -> frame*` |
| 直接返回结果 | `foo.resume(frame*)` |
|  | `foo.destroy(frame*)` |

### 5.2 调用方式

**Goroutine**：
```asm
; 同步调用
MOVQ    $1, AX
CALL    foo
; 结果在 AX
```

**LLVM Coroutine**：
```asm
; 创建协程
mov     edi, 1
call    foo                    ; 返回 frame*
mov     [rsp+8], rax

; 恢复执行
mov     rdi, [rsp+8]
call    qword ptr [rdi]        ; 调用 frame->resume

; 可能多次 resume
```

### 5.3 参数传递

**Goroutine (ABIInternal)**：

```
参数: 寄存器 (AX, BX, ...)
         ↓
      函数入口 spill 到栈
         ↓
      挂起时栈帧保持
         ↓
      恢复后从栈 reload
```

**LLVM Coroutine**：

```
参数: 寄存器
         ↓
      入口复制到协程帧
         ↓
      挂起时栈帧消失，帧对象保持
         ↓
      恢复后从帧对象读取
```

### 5.4 挂起/恢复

| 操作 | Goroutine | LLVM Coroutine |
|------|-----------|----------------|
| 挂起保存 | SP, PC, BP | state index |
| 恢复入口 | JMP 到任意 PC | call resume + switch |
| 栈帧 | 保持存在 | 消失 |

**Goroutine 挂起**：
```asm
MOVQ    PC, gobuf_pc(G)
MOVQ    SP, gobuf_sp(G)
MOVQ    BP, gobuf_bp(G)
; 切换到 g0
```

**Goroutine 恢复**：
```asm
MOVQ    gobuf_sp(G), SP
MOVQ    gobuf_bp(G), BP
JMP     gobuf_pc(G)        ; 直接跳转
```

**LLVM Coroutine 挂起**：
```asm
mov     [frame+index], N   ; 保存状态
ret                        ; 普通返回
```

**LLVM Coroutine 恢复**：
```asm
call    [frame+resume]     ; 调用 resume
; resume 内部:
mov     eax, [frame+index]
switch  eax, ...           ; 状态分发
```

### 5.5 寄存器保存

**Goroutine**：
- 通用寄存器 (AX-R11): 不保存，编译器在挂起前 spill 到栈
- R14: G 指针，调度器维护
- SP, BP: 保存到 gobuf

**LLVM Coroutine**：
- 跨挂起存活变量: 保存到帧对象
- 不跨挂起变量: 不保存

### 5.6 变量访问

**Goroutine**：
```asm
; 始终用栈偏移
MOVQ    x-8(SP), AX        ; 挂起前后相同
```

**LLVM Coroutine**：
```asm
; 跨挂起变量用帧偏移
mov     rax, [rdi+24]      ; rdi = frame*

; 不跨挂起变量用栈偏移
mov     rax, [rsp-8]       ; 临时变量
```

### 5.7 返回值

**Goroutine**：
```asm
MOVQ    result, AX         ; 返回值在寄存器
RET
```

**LLVM Coroutine**：
```cpp
// 通过 promise 返回
frame->promise.return_value(result);

// 调用者获取
int result = task.get();   // 从 promise 读取
```

### 5.8 闭包上下文寄存器冲突

#### Go 闭包调用约定

Go 使用 **DX 寄存器**传递闭包上下文：

```go
func outer() func(int) int {
    captured := 42
    return func(x int) int {  // 闭包
        return x + captured
    }
}
```

**生成的调用代码**：
```asm
; 调用闭包
MOVQ    closure, DX          ; DX = 闭包上下文指针
MOVQ    $10, AX              ; AX = 参数 x
MOVQ    0(DX), BX            ; BX = 函数指针
CALL    BX                   ; 调用
```

**闭包函数入口**：
```asm
TEXT closure_func(SB), ABIInternal, $0
    ; DX 已包含闭包上下文
    MOVQ    8(DX), CX            ; 读取 captured 变量
    ADDQ    CX, AX               ; AX = x + captured
    RET
```

#### LLVM Coroutine 帧指针传递

LLVM Coroutine 的 resume 函数需要接收帧指针：

```cpp
void foo_resume(foo_frame* frame) {  // frame 通过第一个参数寄存器
    switch (frame->index) { ... }
}
```

**C ABI (System V AMD64)**：
```asm
; frame 指针在 RDI（第一个参数）
foo_resume:
    mov     eax, [rdi+16]        ; 读取 frame->index
    ; ...
```

**Go ABIInternal**：
```asm
; frame 指针在 AX（第一个参数）
foo_resume:
    MOVQ    16(AX), CX           ; 读取 frame->index
    ; ...
```

#### 冲突分析

| 场景 | Go 约定 | LLVM Coroutine 需求 | 冲突 |
|------|---------|---------------------|------|
| 普通协程 | AX=参数1 | AX=frame 指针 | **无冲突**（可复用） |
| 协程+闭包 | DX=闭包上下文, AX=参数1 | 需要 frame 指针 | **有冲突** |
| 协程方法 | AX=receiver, BX=参数1 | 需要 frame 指针 | **有冲突** |

**问题场景：协程闭包**

```go
func outer() func() task {
    captured := 42
    return func() task {  // 这是一个协程闭包
        co_await something()
        return captured
    }
}
```

**需要同时传递**：
1. `DX` = 闭包上下文（访问 captured）
2. `?` = 协程帧指针（resume 时需要）

**可能的解决方案**：

| 方案 | 描述 | 问题 |
|------|------|------|
| 帧内嵌闭包上下文 | 将闭包上下文复制到协程帧 | 增加帧大小，复制开销 |
| 帧指针用其他寄存器 | 如 R12/R13 | 需要修改 Go ABI |
| 帧指针通过栈传递 | 不用寄存器 | 性能下降 |
| 闭包上下文存帧中 | frame->closure_ctx | 需要编译器特殊处理 |

**Go 的选择**：

Go 选择了有栈协程，完全避免了这个问题：
```
闭包上下文在 DX ──► 函数入口保存到栈 ──► 挂起时栈保持 ──► 恢复后从栈读取
                    不需要额外的帧指针
```

### 5.9 cgo 兼容性

#### Go cgo 调用链

```
Go 代码 (ABIInternal)
    ↓ ABI wrapper
cgo trampoline (ABI0)
    ↓
C 代码 (System V AMD64 ABI)
```

**ABI 差异**：

| 方面 | Go ABIInternal | System V AMD64 |
|------|----------------|----------------|
| 整数参数 | AX,BX,CX,DI,SI,R8-R11 | RDI,RSI,RDX,RCX,R8,R9 |
| 返回值 | AX,BX,... | RAX,RDX |
| 浮点参数 | X0-X14 | XMM0-XMM7 |
| 栈对齐 | 8 字节 | 16 字节 |
| 红区 | 无 | 128 字节 |
| 调用者保存 | AX-R11 | 大部分 |
| G 指针 | R14 固定 | 无此概念 |

#### Goroutine 中的 cgo

```go
func goFunc() {
    x := 1
    C.cFunc()    // cgo 调用
    println(x)   // x 仍然可用
}
```

**处理流程**：
```
1. goFunc 入口：参数 spill 到栈
2. cgo 调用前：
   - 保存 G 指针
   - 切换到 g0 栈（系统栈）
   - 转换参数到 C ABI
3. C 函数执行
4. cgo 返回后：
   - 恢复 G 指针到 R14
   - 切换回 goroutine 栈
5. 继续执行：x 在原来的栈位置
```

**关键点**：cgo 会切换到系统栈，但 goroutine 自己的栈帧保持不变。

#### LLVM Coroutine 中的 cgo（假设场景）

```cpp
task<void> coroFunc() {
    int x = 1;
    co_await async_c_call();  // 调用 C 代码
    use(x);                    // x 从哪里读取？
}
```

**问题 1：帧指针传递**

```
Go resume 函数使用 Go ABI:
    frame 指针在 AX

C 代码如果要 resume:
    需要按 C ABI 调用
    frame 指针应该在 RDI

需要 wrapper 转换!
```

**问题 2：挂起点在 C 代码中**

```
如果 C 代码内部需要挂起协程：

Go 协程:
    C.blocking_call()  ──► runtime 自动处理，切换到其他 goroutine

LLVM Coroutine:
    C 代码不是协程，无法 co_await
    必须返回到 Go 代码才能挂起
```

**问题 3：G 指针丢失**

```
LLVM Coroutine resume 时：

普通函数调用:
    调用者确保 R14 = G

从 C 代码 resume:
    C 不知道 R14 的含义
    R14 可能被 C 代码修改
    需要从 TLS 重新加载 G
```

#### cgo + 协程的 ABI 边界

```
┌─────────────────────────────────────────────────────────────┐
│                        Go Runtime                           │
├─────────────────────────────────────────────────────────────┤
│  Goroutine 方案:                                            │
│                                                             │
│  go func() ──► C.func() ──► 返回 ──► 继续                   │
│       │            │           │         │                  │
│    栈帧1         系统栈       栈帧1     栈帧1               │
│    (保持)       (临时)       (恢复)    (继续)               │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│  LLVM Coroutine 方案（假设）:                               │
│                                                             │
│  coro() ──► 挂起 ──► C 要 resume ──► ???                    │
│     │         │           │                                 │
│   帧对象    返回C       C 调用 resume                       │
│   (创建)   (栈消失)     (ABI 不匹配!)                       │
│                                                             │
│  解决方案: C 不能直接 resume，必须通过 Go wrapper           │
│                                                             │
│  C 代码 ──► cgo_resume_wrapper(frame) ──► Go resume(frame)  │
│               (C ABI)                      (Go ABI)         │
└─────────────────────────────────────────────────────────────┘
```

#### 总结：寄存器与 cgo 约束

| 约束 | Goroutine | LLVM Coroutine (假设移植到 Go) |
|------|-----------|-------------------------------|
| 闭包 DX 寄存器 | 自然兼容 | 需要特殊处理帧+闭包 |
| 方法 receiver | 自然兼容 | 需要特殊处理帧+receiver |
| G 指针 R14 | 栈切换时保持 | resume 时需重新加载 |
| cgo 调用 | 运行时处理 | 需要 ABI wrapper |
| C resume Go | 不需要 | 需要额外 trampoline |
| 栈对齐 | 自动处理 | 需要考虑 16 字节对齐 |

---

## 6. 帧结构对比

### 6.1 Goroutine gobuf

```
┌──────────────────┐ +0
│  sp              │
├──────────────────┤ +8
│  pc              │
├──────────────────┤ +16
│  g               │
├──────────────────┤ +24
│  ctxt            │
├──────────────────┤ +32
│  ret             │
├──────────────────┤ +40
│  lr              │
├──────────────────┤ +48
│  bp              │
└──────────────────┘

大小: 56 bytes (固定)
不包含局部变量
```

### 6.2 LLVM Coroutine Frame

```
┌──────────────────┐ +0
│  resume_fn (8B)  │
├──────────────────┤ +8
│  destroy_fn (8B) │
├──────────────────┤ +16
│  index (4-8B)    │
├──────────────────┤
│  promise         │
├──────────────────┤
│  变量 a          │
├──────────────────┤
│  变量 b          │
├──────────────────┤
│  ...             │
└──────────────────┘

大小: 可变，取决于跨挂起变量
包含所有跨挂起存活的值
```

---

## 6. 栈帧生命周期

### Goroutine

```
调用 ──► 栈帧创建 ──► 挂起 ──► 栈帧保持 ──► 恢复 ──► 继续 ──► 返回 ──► 栈帧销毁

         SP 保存到 gobuf       SP 恢复
```

### LLVM Coroutine

```
调用 ──► 帧对象分配 ──► 挂起 ──► 栈帧消失 ──► 恢复 ──► 新栈帧 ──► 完成 ──► 帧对象销毁
         + 栈帧创建      帧对象保持    call resume
```

---

## 7. 总结表

| 方面 | Goroutine | LLVM Coroutine |
|------|-----------|----------------|
| 栈模型 | 独立栈 (2KB-1GB) | 无栈，共享调用栈 |
| 挂起位置 | 任意函数调用点 | 仅 co_await 点 |
| 函数形态 | 签名不变 | 分割为 3 个函数 |
| 上下文保存 | SP, PC, BP | state + 跨挂起变量 |
| 恢复方式 | JMP 到 PC | call resume + switch |
| 变量存储 | 栈上 (持久) | 帧对象 (跨挂起) + 栈 (临时) |
| 内存开销 | ~2KB/goroutine | 几十~几百 bytes/coroutine |
| 切换开销 | ~100-200ns | ~10-50ns |
