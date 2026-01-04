# Go ABI0 与 ABIInternal 深度解析

## 1. 背景与动机

### 1.1 什么是 ABI

ABI（Application Binary Interface）定义了：
- 函数调用时参数如何传递
- 返回值如何传递
- 寄存器如何分配
- 栈帧如何布局
- 调用者/被调用者保存哪些寄存器

### 1.2 为什么需要新的 ABI

Go 1.17 之前，Go 使用基于栈的调用约定（ABI0）：

```
调用 func Add(a, b int) int:

         ┌─────────────┐
   SP+16 │  返回值      │
         ├─────────────┤
   SP+8  │  参数 b      │
         ├─────────────┤
   SP+0  │  参数 a      │
         └─────────────┘
```

**问题**：
1. 每次函数调用都需要栈读写操作
2. 栈操作比寄存器操作慢 3-10 倍
3. 增加内存带宽压力
4. 缓存效率低

**基准测试显示**：切换到寄存器 ABI 后，整体性能提升 5-10%。

---

## 2. ABI0 详解

### 2.1 核心规则

```
所有参数从右到左压栈
所有返回值从右到左压栈（在参数之上）
调用者负责分配和清理栈空间
```

### 2.2 栈帧布局

```
高地址
         ┌─────────────────────┐
         │   调用者的栈帧       │
         ├─────────────────────┤
         │   返回地址 (CALL指令) │
         ├─────────────────────┤
         │   返回值 N           │
         ├─────────────────────┤
         │   ...               │
         ├─────────────────────┤
         │   返回值 1           │
         ├─────────────────────┤
         │   参数 N             │
         ├─────────────────────┤
         │   ...               │
         ├─────────────────────┤
  SP ->  │   参数 1             │
         └─────────────────────┘
低地址
```

### 2.3 汇编示例

```go
// Go 代码
func Add(a, b int) int {
    return a + b
}
```

```asm
// ABI0 汇编实现
TEXT ·Add(SB), NOSPLIT, $0-24
    // 参数布局: a(+0), b(+8), 返回值(+16)
    MOVQ    a+0(FP), AX      // 从栈读取 a
    MOVQ    b+8(FP), BX      // 从栈读取 b
    ADDQ    BX, AX           // AX = a + b
    MOVQ    AX, ret+16(FP)   // 写回栈上的返回值
    RET
```

### 2.4 FP 和 SP 寄存器

```
FP (Frame Pointer): 伪寄存器，指向参数/返回值区域
SP (Stack Pointer): 栈顶指针

                    ┌─────────────┐
          FP+16 ->  │  返回值      │
                    ├─────────────┤
          FP+8  ->  │  参数 b      │
                    ├─────────────┤
          FP+0  ->  │  参数 a      │
                    ├─────────────┤
                    │  返回地址    │  <- 硬件SP（调用后）
                    ├─────────────┤
          SP    ->  │  局部变量    │
                    └─────────────┘
```

---

## 3. ABIInternal 详解

### 3.1 设计目标

1. 使用寄存器传递参数和返回值
2. 保持向后兼容（汇编代码仍用 ABI0）
3. 不破坏 runtime 的栈扫描
4. 支持 defer、panic、reflect 等特性

### 3.2 寄存器分配规则（amd64）

#### 整数寄存器（按顺序使用）

| 顺序 | 寄存器 | 用途 |
|------|--------|------|
| 1 | RAX | 整数参数/返回值 1 |
| 2 | RBX | 整数参数/返回值 2 |
| 3 | RCX | 整数参数/返回值 3 |
| 4 | RDI | 整数参数/返回值 4 |
| 5 | RSI | 整数参数/返回值 5 |
| 6 | R8  | 整数参数/返回值 6 |
| 7 | R9  | 整数参数/返回值 7 |
| 8 | R10 | 整数参数/返回值 8 |
| 9 | R11 | 整数参数/返回值 9 |

#### 浮点寄存器

| 顺序 | 寄存器 |
|------|--------|
| 1-15 | X0-X14 |

#### 特殊寄存器

| 寄存器 | 用途 |
|--------|------|
| RSP | 栈指针 |
| RBP | 帧指针（可选） |
| R12 | 保留 |
| R13 | 保留 |
| R14 | 当前 G (goroutine) 指针 |
| R15 | GOT 指针（位置无关代码） |

### 3.3 参数分配算法

```go
// 伪代码：参数分配逻辑
func assignParams(params []Type) Assignment {
    intRegs := []Reg{RAX, RBX, RCX, RDI, RSI, R8, R9, R10, R11}
    floatRegs := []Reg{X0, X1, ..., X14}

    intIdx := 0
    floatIdx := 0
    stackOffset := 0

    for _, param := range params {
        switch {
        case isInteger(param) || isPointer(param):
            if intIdx < len(intRegs) {
                assign(param, intRegs[intIdx])
                intIdx++
            } else {
                assign(param, stack[stackOffset])
                stackOffset += size(param)
            }

        case isFloat(param):
            if floatIdx < len(floatRegs) {
                assign(param, floatRegs[floatIdx])
                floatIdx++
            } else {
                assign(param, stack[stackOffset])
                stackOffset += size(param)
            }

        case isStruct(param):
            // 结构体拆分处理
            assignStruct(param, &intIdx, &floatIdx, &stackOffset)
        }
    }
}
```

### 3.4 结构体传递规则

```go
type Point struct {
    X, Y int64
}

func Process(p Point) int64 {
    return p.X + p.Y
}
```

**分配结果**：
```
p.X -> RAX (第1个整数寄存器)
p.Y -> RBX (第2个整数寄存器)
返回值 -> RAX
```

**复杂结构体**：
```go
type Large struct {
    A, B, C, D, E, F, G, H, I, J int64  // 10个字段
}
```

超过寄存器数量时，剩余字段通过栈传递。

### 3.5 大结构体传递详解

#### Go 的策略：字段拆分

Go **不使用隐藏指针**，而是将结构体拆分为独立字段传递：

```go
type Large struct {
    A, B, C, D, E, F, G, H, I, J int64  // 80 字节
}

func Process(s Large) int64 {
    return s.A + s.J
}
```

**参数分配**：

```
s.A -> AX  (寄存器 1)
s.B -> BX  (寄存器 2)
s.C -> CX  (寄存器 3)
s.D -> DI  (寄存器 4)
s.E -> SI  (寄存器 5)
s.F -> R8  (寄存器 6)
s.G -> R9  (寄存器 7)
s.H -> R10 (寄存器 8)
s.I -> R11 (寄存器 9)
s.J -> 栈  (寄存器用完，溢出到栈)
```

**生成的汇编**：

```asm
; 调用者
    MOVQ    s.A, AX
    MOVQ    s.B, BX
    MOVQ    s.C, CX
    MOVQ    s.D, DI
    MOVQ    s.E, SI
    MOVQ    s.F, R8
    MOVQ    s.G, R9
    MOVQ    s.H, R10
    MOVQ    s.I, R11
    MOVQ    s.J, 0(SP)        ; 第 10 个字段通过栈
    CALL    Process

; Process 函数
TEXT Process(SB), ABIInternal, $0
    ; s.A 在 AX, s.J 在栈上
    ADDQ    0(SP), AX         ; AX = s.A + s.J (从栈读取 s.J)
    RET
```

#### C 的策略：隐藏指针

C 对于 >16 字节的结构体使用**隐藏指针**：

```c
struct Large {
    long a, b, c, d, e, f, g, h, i, j;  // 80 字节
};

long process(struct Large s) {
    return s.a + s.j;
}
```

**实际传递**：

```
调用者:
1. 在栈上分配 80 字节
2. 复制整个结构体到栈
3. 传递栈地址作为隐藏第一个参数

实际签名变为: long process(struct Large *s)
```

```asm
; C 调用者
    sub     rsp, 80              ; 分配空间
    ; 复制结构体到 [rsp]
    mov     rdi, rsp             ; RDI = 结构体指针
    call    process

; C process 函数
process:
    ; RDI = 结构体指针
    mov     rax, [rdi]           ; rax = s->a
    add     rax, [rdi+72]        ; rax += s->j
    ret
```

#### 策略对比

| 方面 | Go (字段拆分) | C (隐藏指针) |
|------|---------------|--------------|
| 传递方式 | 逐字段传寄存器+栈 | 整体复制到栈，传指针 |
| 小结构体 | 全寄存器 | ≤16 字节用寄存器 |
| 大结构体 | 部分寄存器+部分栈 | 全部栈+指针 |
| 复制开销 | 调用者复制到寄存器/栈 | 调用者复制整体 |
| 被调用者访问 | 直接从寄存器/栈读取 | 通过指针间接访问 |
| 修改原值 | 不可能（值传递） | 不可能（复制后传指针） |

#### 具体示例对比

```go
// Go: 24 字节结构体
type Triple struct {
    X, Y, Z int64
}

func Sum(t Triple) int64 {
    return t.X + t.Y + t.Z
}
```

**Go 传递**：

```asm
; 调用 Sum(Triple{1, 2, 3})
    MOVQ    $1, AX        ; t.X
    MOVQ    $2, BX        ; t.Y
    MOVQ    $3, CX        ; t.Z
    CALL    Sum

; Sum 函数
    ADDQ    BX, AX        ; X + Y
    ADDQ    CX, AX        ; + Z
    RET
```

```c
// C: 24 字节结构体 (> 16 字节)
struct Triple {
    long x, y, z;
};

long sum(struct Triple t) {
    return t.x + t.y + t.z;
}
```

**C 传递**：

```asm
; 调用 sum((Triple){1, 2, 3})
    sub     rsp, 24
    mov     qword ptr [rsp], 1      ; t.x
    mov     qword ptr [rsp+8], 2    ; t.y
    mov     qword ptr [rsp+16], 3   ; t.z
    mov     rdi, rsp                ; 传递指针
    call    sum

; sum 函数
    mov     rax, [rdi]              ; t->x
    add     rax, [rdi+8]            ; + t->y
    add     rax, [rdi+16]           ; + t->z
    ret
```

#### 返回大结构体

**Go**：返回值也拆分到寄存器

```go
func MakeTriple() Triple {
    return Triple{1, 2, 3}
}
```

```asm
TEXT MakeTriple(SB), ABIInternal, $0
    MOVQ    $1, AX        ; 返回值 X
    MOVQ    $2, BX        ; 返回值 Y
    MOVQ    $3, CX        ; 返回值 Z
    RET
```

**C**：调用者传递隐藏返回地址

```c
struct Triple make_triple() {
    return (struct Triple){1, 2, 3};
}
```

```asm
; 调用者
    sub     rsp, 24
    mov     rdi, rsp          ; 传递返回值地址
    call    make_triple
    ; 结果在 [rsp]

; make_triple
make_triple:
    mov     qword ptr [rdi], 1
    mov     qword ptr [rdi+8], 2
    mov     qword ptr [rdi+16], 3
    mov     rax, rdi          ; 返回地址
    ret
```

#### 性能影响

```
┌─────────────────────────────────────────────────────────────────┐
│                    大结构体传递性能                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Go 字段拆分:                                                    │
│    优点:                                                         │
│      - 前 9 个字段在寄存器中，访问快                             │
│      - 无需指针解引用                                            │
│      - 编译器可以优化未使用的字段                                │
│    缺点:                                                         │
│      - 调用时需要多条 MOV 指令                                   │
│      - 寄存器压力大                                              │
│                                                                  │
│  C 隐藏指针:                                                     │
│    优点:                                                         │
│      - 只传递一个指针，调用开销固定                              │
│      - 适合非常大的结构体                                        │
│    缺点:                                                         │
│      - 所有访问都需要内存读取                                    │
│      - 可能有更多的缓存未命中                                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

#### 阈值对比

| 结构体大小 | Go | C |
|------------|-----|-----|
| ≤8 字节 | 1 个寄存器 | 1 个寄存器 |
| 9-16 字节 | 2 个寄存器 | 2 个寄存器 |
| 17-72 字节 | 拆分到寄存器+栈 | **隐藏指针** |
| >72 字节 | 拆分到寄存器+栈 | 隐藏指针 |

**关键区别**：C 在 16 字节处切换策略，Go 没有这个阈值，一直使用字段拆分。

### 3.6 接口和切片

```go
// 接口 = (type, data) 两个指针
type iface struct {
    tab  *itab   // -> 寄存器1
    data unsafe.Pointer  // -> 寄存器2
}

// 切片 = (ptr, len, cap) 三个值
type slice struct {
    ptr *T   // -> 寄存器1
    len int  // -> 寄存器2
    cap int  // -> 寄存器3
}
```

### 3.7 实际生成的汇编

```go
func Add(a, b int) int {
    return a + b
}
```

```asm
// ABIInternal 生成的代码
TEXT main.Add(SB), ABIInternal, $0-0
    // a 在 AX, b 在 BX, 返回值放 AX
    ADDQ    BX, AX      // AX = a + b
    RET                 // 返回值已在 AX
```

对比 ABI0：
- 少了 3 次内存读写
- 代码更短
- 执行更快

---

## 4. ABI 包装器（ABI Wrapper）

### 4.1 为什么需要包装器

当 ABIInternal 代码需要调用 ABI0 代码时（如汇编函数），需要进行 ABI 转换。

### 4.2 包装器工作流程

```
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│  Go 代码          │     │  ABI Wrapper     │     │  汇编代码        │
│  (ABIInternal)   │ --> │  寄存器->栈      │ --> │  (ABI0)          │
│  参数在寄存器     │     │  栈->寄存器      │     │  参数在栈        │
└──────────────────┘     └──────────────────┘     └──────────────────┘
```

### 4.3 包装器生成

编译器自动生成包装器：

```asm
// 自动生成的 wrapper 示例
TEXT ·Add·abi0(SB), NOSPLIT, $24-0
    // 保存寄存器参数到栈
    MOVQ    AX, arg0-8(SP)   // a
    MOVQ    BX, arg1-16(SP)  // b

    // 调用真正的 ABI0 函数
    CALL    ·Add(SB)

    // 从栈读取返回值到寄存器
    MOVQ    ret-24(SP), AX
    RET
```

### 4.4 //go:noescape 和 //go:nosplit

```go
//go:noescape  // 告诉编译器参数不会逃逸，避免堆分配
//go:nosplit   // 禁止栈分裂检查，用于低级代码
func asmFunc(p *byte)
```

---

## 5. 实现细节

### 5.1 编译器变更（cmd/compile）

```
src/cmd/compile/internal/abi/
├── abi.go           # ABI 类型定义
├── abi_amd64.go     # amd64 特定实现
└── abi_arm64.go     # arm64 特定实现
```

核心结构：

```go
// ABIConfig 定义 ABI 配置
type ABIConfig struct {
    IntArgRegs   []Register  // 整数参数寄存器
    FloatArgRegs []Register  // 浮点参数寄存器
    IntRetRegs   []Register  // 整数返回值寄存器
    FloatRetRegs []Register  // 浮点返回值寄存器
}

// ABIParamAssignment 描述参数分配
type ABIParamAssignment struct {
    Type   *types.Type
    Regs   []Register  // 使用的寄存器
    Offset int64       // 栈偏移（如果溢出）
}
```

### 5.2 链接器变更（cmd/link）

链接器需要：
1. 识别函数的 ABI 类型
2. 生成必要的 wrapper
3. 处理跨 ABI 调用

```go
// 符号的 ABI 标记
const (
    ABI0       = 0
    ABIInternal = 1
)
```

### 5.3 Runtime 变更

#### 5.3.1 栈扫描

GC 需要知道活跃指针的位置：

```go
// 寄存器中的指针需要特殊处理
type regArgs struct {
    Ints   [9]uint64   // 整数寄存器副本
    Floats [15]uint64  // 浮点寄存器副本
    Ptrs   [9]bool     // 标记哪些是指针
}
```

#### 5.3.2 Defer 实现

```go
// defer 需要保存寄存器状态
type _defer struct {
    // ...
    regs regArgs  // 保存寄存器参数
}
```

#### 5.3.3 Reflect

```go
// reflect.Call 需要处理两种 ABI
func (v Value) Call(in []Value) []Value {
    // 检查目标函数的 ABI
    if fn.isABIInternal() {
        return callABIInternal(fn, in)
    }
    return callABI0(fn, in)
}
```

### 5.4 Spill 区域

即使使用寄存器传参，仍需要栈空间保存寄存器：

```
┌─────────────────────┐
│   调用者保存区域     │
├─────────────────────┤
│   Spill 区域        │  <- 保存寄存器参数（GC/抢占时需要）
├─────────────────────┤
│   局部变量          │
├─────────────────────┤
│   被调用者保存区域   │
└─────────────────────┘
```

**为什么需要 Spill**：
1. GC 需要扫描所有指针
2. 抢占时需要保存完整状态
3. defer/panic 需要恢复参数

---

## 6. ABI0 vs ABIInternal 调用约定详细对比

### 6.1 参数传递对比

```
                ABI0                              ABIInternal

func foo(a, b, c int)                     func foo(a, b, c int)

         ┌─────────┐                              寄存器:
  FP+16  │    c    │                              RAX = a
         ├─────────┤                              RBX = b
  FP+8   │    b    │                              RCX = c
         ├─────────┤
  FP+0   │    a    │                              栈: (空)
         └─────────┘
```

### 6.2 返回值对比

```
                ABI0                              ABIInternal

func bar() (int, error)                   func bar() (int, error)

         ┌─────────┐                              寄存器:
  FP+8   │  error  │                              RAX = int
         ├─────────┤                              RBX = error.type
  FP+0   │   int   │                              RCX = error.data
         └─────────┘
```

### 6.3 寄存器职责对比

| 寄存器 | ABI0 用途 | ABIInternal 用途 |
|--------|-----------|------------------|
| RAX | 临时/返回值 | 参数1/返回值1 |
| RBX | 临时 | 参数2/返回值2 |
| RCX | 临时 | 参数3/返回值3 |
| RDI | 临时 | 参数4/返回值4 |
| RSI | 临时 | 参数5/返回值5 |
| R8-R11 | 临时 | 参数6-9/返回值6-9 |
| R14 | 当前 G | 当前 G（相同） |
| SP | 栈顶 | 栈顶（相同） |
| BP | 帧指针 | 帧指针（相同） |

### 6.4 Caller-saved vs Callee-saved

**ABIInternal 的约定**：

| 类型 | 寄存器 | 说明 |
|------|--------|------|
| Caller-saved | RAX, RBX, RCX, RDI, RSI, R8-R11, X0-X14 | 调用者负责保存 |
| Callee-saved | R12, R13, R14, R15, RBP, RSP | 被调用者负责保存 |
| 固定用途 | R14 (G), RSP, RBP | 不可用于参数 |

```go
func caller() {
    a := 1                    // a 在 RAX

    // 调用前：如果 a 之后还需要，必须 spill
    // （编译器自动处理）
    callee()

    // 调用后：RAX 已被覆盖，需要 reload
    println(a)
}

func callee() {
    // 可以自由使用 RAX, RBX, RCX...
    // 但必须保存/恢复 R12, R13, RBP...
}
```

### 6.5 函数调用指令对比

**ABI0 调用**：
```asm
# 调用 foo(1, 2)

MOVQ    $1, 0(SP)      # 参数 a 入栈
MOVQ    $2, 8(SP)      # 参数 b 入栈
CALL    foo(SB)        # 调用
MOVQ    16(SP), AX     # 读取返回值
```

**ABIInternal 调用**：
```asm
# 调用 foo(1, 2)

MOVQ    $1, AX         # 参数 a 到 RAX
MOVQ    $2, BX         # 参数 b 到 RBX
CALL    foo(SB)        # 调用，返回值已在 AX
```

### 6.6 栈帧大小对比

```go
func example(a, b, c, d int) (int, int) {
    return a+b, c+d
}
```

| 项目 | ABI0 | ABIInternal |
|------|------|-------------|
| 参数区域 | 32 字节 (4 × 8) | 0 字节 |
| 返回值区域 | 16 字节 (2 × 8) | 0 字节 |
| Spill 区域 | 0 字节 | 32 字节 (可选) |
| **典型总大小** | **48 字节** | **0-32 字节** |

### 6.7 闭包调用对比

```go
fn := func(x int) int { return x + captured }
fn(42)
```

**ABI0**：
```asm
# 闭包上下文在 DX
MOVQ    closure+0(FP), DX    # 加载闭包
MOVQ    $42, 0(SP)           # 参数入栈
MOVQ    0(DX), AX            # 获取函数指针
CALL    AX                    # 调用
```

**ABIInternal**：
```asm
# 闭包上下文在 DX，参数在 AX
MOVQ    closure, DX          # 闭包上下文
MOVQ    $42, AX              # 参数
MOVQ    0(DX), BX            # 获取函数指针
CALL    BX                    # 调用
```

---

## 7. 闭包上下文深度分析

### 7.1 闭包的内存结构

当函数返回一个闭包时，Go 编译器会生成一个**闭包对象**：

```go
func outer(a int) func(int) int {
    b := a * 2
    return func(x int) int {
        return x + a + b  // 捕获 a 和 b
    }
}
```

**编译器生成的闭包结构**：

```
闭包对象 (堆上分配)
┌────────────────────────┐ +0
│  函数指针 (code ptr)    │  ──► 闭包函数的机器码
├────────────────────────┤ +8
│  捕获变量 a            │
├────────────────────────┤ +16
│  捕获变量 b            │
└────────────────────────┘

大小 = 8 (函数指针) + 8*N (捕获变量数)
```

**等价的伪结构体**：

```go
// 编译器内部表示
type closure_outer_func1 struct {
    F    func(int) int  // 函数指针（实际指向闭包函数代码）
    a    int            // 捕获的变量
    b    int            // 捕获的变量
}
```

### 7.2 闭包的创建过程

```go
func outer(a int) func(int) int {
    b := a * 2
    return func(x int) int {
        return x + a + b
    }
}

// 调用
fn := outer(10)
```

**编译器生成的代码（伪汇编）**：

```asm
TEXT outer(SB), ABIInternal, $24-8
    ; 入口：a 在 AX
    MOVQ    AX, a-8(SP)           ; spill a 到栈

    ; b := a * 2
    SHLQ    $1, AX                ; AX = a * 2
    MOVQ    AX, b-16(SP)          ; b 保存到栈

    ; 分配闭包对象 (24 字节)
    MOVQ    $24, AX               ; 大小
    CALL    runtime.newobject     ; 返回指针在 AX

    ; 填充闭包对象
    LEAQ    outer.func1(SB), CX   ; 闭包函数地址
    MOVQ    CX, 0(AX)             ; closure[0] = 函数指针
    MOVQ    a-8(SP), CX
    MOVQ    CX, 8(AX)             ; closure[1] = a
    MOVQ    b-16(SP), CX
    MOVQ    CX, 16(AX)            ; closure[2] = b

    ; 返回闭包指针（在 AX 中）
    RET
```

**内存布局**：

```
outer 调用后：

栈:                              堆:
┌─────────────┐                  ┌─────────────────────┐
│ outer 栈帧  │                  │ 闭包对象            │
│  a = 10     │                  │  [0] = &outer.func1 │
│  b = 20     │                  │  [8] = 10 (a)       │
└─────────────┘                  │ [16] = 20 (b)       │
      ↓                          └─────────────────────┘
   栈帧销毁                              ↑
                                   fn 指向这里
```

### 7.3 闭包的调用过程

```go
fn := outer(10)  // fn 是闭包指针
result := fn(5)  // 调用闭包
```

**调用者代码**：

```asm
; fn(5) 的调用
; fn 在某个寄存器或栈位置，假设在 R12

MOVQ    R12, DX               ; DX = 闭包上下文指针（关键！）
MOVQ    $5, AX                ; AX = 参数 x
MOVQ    0(DX), BX             ; BX = 闭包对象[0] = 函数指针
CALL    BX                    ; 调用闭包函数
; 返回值在 AX
```

**DX 寄存器的特殊角色**：

```
┌─────────────────────────────────────────────────────────┐
│  Go 调用约定中，DX 专门用于传递闭包上下文               │
│                                                         │
│  调用闭包时：                                           │
│    DX = 闭包对象指针                                    │
│    AX, BX, CX... = 普通参数                            │
│                                                         │
│  闭包函数通过 DX 访问捕获的变量                         │
└─────────────────────────────────────────────────────────┘
```

### 7.4 闭包函数内部访问捕获变量

**闭包函数代码**：

```asm
TEXT outer.func1(SB), ABIInternal, $0-0
    ; 入口时：
    ;   DX = 闭包上下文指针
    ;   AX = 参数 x

    ; 读取捕获的变量
    MOVQ    8(DX), BX             ; BX = a (offset 8)
    MOVQ    16(DX), CX            ; CX = b (offset 16)

    ; 计算 x + a + b
    ADDQ    BX, AX                ; AX = x + a
    ADDQ    CX, AX                ; AX = x + a + b

    RET                           ; 返回值在 AX
```

**变量访问方式**：

```
DX (闭包指针)
    │
    ▼
┌─────────────────────┐
│ [0] = 函数指针      │  ← 调用时已用过
├─────────────────────┤
│ [8] = a             │  ← MOVQ 8(DX), reg
├─────────────────────┤
│ [16] = b            │  ← MOVQ 16(DX), reg
└─────────────────────┘
```

### 7.5 值捕获 vs 引用捕获

#### 值捕获（默认）

```go
func outer(a int) func() int {
    return func() int {
        return a  // 值捕获，a 被复制到闭包
    }
}
```

```
闭包对象:
┌─────────────────┐
│ 函数指针        │
├─────────────────┤
│ a 的副本        │  ← 值被复制
└─────────────────┘
```

#### 引用捕获（变量被修改时）

```go
func outer() func() int {
    a := 0
    return func() int {
        a++       // 修改 a，需要引用捕获
        return a
    }
}
```

**编译器处理**：将 `a` 提升到堆上

```go
// 编译器转换后的等价代码
func outer() func() int {
    a := new(int)  // a 变成指针，分配在堆上
    *a = 0
    return func() int {
        *a++
        return *a
    }
}
```

```
闭包对象:                    堆:
┌─────────────────┐         ┌─────────┐
│ 函数指针        │         │ a = 0   │
├─────────────────┤         └─────────┘
│ &a (指针)       │ ────────────┘
└─────────────────┘
```

**汇编差异**：

```asm
; 值捕获：直接读取
MOVQ    8(DX), AX             ; AX = a

; 引用捕获：需要两次间接寻址
MOVQ    8(DX), BX             ; BX = &a
MOVQ    0(BX), AX             ; AX = *(&a) = a
```

### 7.6 闭包作为参数传递

```go
func caller() {
    fn := outer(10)
    process(fn)
}

func process(f func(int) int) {
    result := f(5)
}
```

**process 调用 f 时**：

```asm
TEXT process(SB), ABIInternal, $0-0
    ; 入口：f 在 AX（func 类型 = 闭包指针）

    ; 调用 f(5)
    MOVQ    AX, DX                ; DX = 闭包上下文
    MOVQ    $5, AX                ; AX = 参数
    MOVQ    0(DX), BX             ; BX = 函数指针
    CALL    BX                    ; 调用

    ; 返回值在 AX
    RET
```

**关键点**：`func` 类型的值本身就是闭包指针，调用时：
1. 将闭包指针放入 DX
2. 从闭包对象读取函数指针
3. 调用该函数

### 7.7 普通函数 vs 闭包函数

```go
// 普通函数
func add(a, b int) int {
    return a + b
}

// 闭包函数
func makeAdder(a int) func(int) int {
    return func(b int) int {
        return a + b
    }
}
```

**调用差异**：

| 方面 | 普通函数 | 闭包函数 |
|------|----------|----------|
| 调用指令 | `CALL add(SB)` | `CALL (闭包指针)` |
| DX 寄存器 | 不使用 | 必须设置为闭包指针 |
| 参数位置 | AX, BX, ... | AX, BX, ...（相同） |
| 函数地址 | 编译时确定 | 运行时从闭包读取 |

### 7.8 闭包与接口的关系

```go
type Adder interface {
    Add(int) int
}

func getAdder(base int) Adder {
    return adderFunc(func(x int) int {
        return base + x
    })
}

type adderFunc func(int) int

func (f adderFunc) Add(x int) int {
    return f(x)
}
```

**内存布局**：

```
接口值:                      闭包对象:              堆:
┌─────────────┐             ┌─────────────┐        ┌──────┐
│ itab        │             │ 函数指针    │        │ base │
├─────────────┤             ├─────────────┤        └──────┘
│ data ───────────────────► │ &base ──────────────────┘
└─────────────┘             └─────────────┘
```

### 7.9 闭包调用的完整流程图

```
1. 创建闭包
   outer(10) 调用
        │
        ▼
   ┌─────────────────────────────────────┐
   │ 分配闭包对象 (runtime.newobject)    │
   │ 填充: [函数指针, 捕获变量...]        │
   └─────────────────────────────────────┘
        │
        ▼
   返回闭包指针 (在 AX)
        │
        ▼
   fn := ... (保存到变量)


2. 调用闭包
   fn(5) 调用
        │
        ▼
   ┌─────────────────────────────────────┐
   │ MOVQ fn, DX        ; 闭包上下文     │
   │ MOVQ $5, AX        ; 参数           │
   │ MOVQ 0(DX), BX     ; 读取函数指针   │
   │ CALL BX            ; 间接调用       │
   └─────────────────────────────────────┘
        │
        ▼
   ┌─────────────────────────────────────┐
   │ 闭包函数入口:                        │
   │   DX = 闭包指针                      │
   │   AX = 参数 x                        │
   │                                      │
   │ MOVQ 8(DX), BX     ; 读取 a         │
   │ ADDQ BX, AX        ; x + a          │
   │ RET                ; 返回           │
   └─────────────────────────────────────┘
        │
        ▼
   返回值在 AX
```

### 7.10 为什么 DX (Caller-saved) 可以用于闭包上下文

#### 问题

DX 是 caller-saved 寄存器，意味着任何函数调用都可能破坏它。那为什么 Go 选择用它传递闭包上下文？

#### 答案：DX 只需要在"入口那一瞬间"有效

```
┌─────────────────────────────────────────────────────────────────┐
│                    DX 的生命周期                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  调用者:                                                         │
│    MOVQ  closure_ptr, DX    ─┐                                  │
│    MOVQ  $5, AX              │  DX 有效                         │
│    CALL  closure_func       ─┘                                  │
│                                                                  │
│  闭包函数入口:                                                   │
│    ; DX 此时有效             ─┐                                  │
│    MOVQ  DX, -8(SP)          │  立即 spill 到栈                 │
│    ; DX 不再需要了           ─┘                                  │
│                                                                  │
│  闭包函数体:                                                     │
│    ; 从栈读取闭包上下文                                          │
│    MOVQ  -8(SP), R12         ; 用 callee-saved 寄存器缓存       │
│    MOVQ  8(R12), AX          ; 读取捕获变量                      │
│    CALL  other_func          ; DX 可能被破坏，但我们不在乎      │
│    MOVQ  8(R12), AX          ; 继续使用 R12                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**关键点**：
1. DX 只在 `CALL` 指令到闭包函数入口的第一条指令之间需要有效
2. 闭包函数**立即**将 DX spill 到栈或复制到 callee-saved 寄存器
3. 之后的代码从栈/callee-saved 寄存器读取，不再依赖 DX

#### 为什么不用 Callee-saved 寄存器？

| 方案 | 问题 |
|------|------|
| 用 RBX (callee-saved) | 调用者需要先保存 RBX，开销大 |
| 用 R12 (callee-saved) | 同上，且 R12-R15 是珍贵资源 |
| 用 DX (caller-saved) | 调用者无需保存，闭包入口保存即可 |

**Caller-saved 的优势**：调用者可以直接设置，无需保存/恢复。

#### 对比 C 的 R10 静态链

C (GCC) 用 R10 传递嵌套函数的静态链，R10 也是 caller-saved：

```asm
; C 嵌套函数调用
    lea     r10, [rbp]        ; 设置静态链
    call    nested_func

; nested_func 入口
nested_func:
    mov     [rsp-8], r10      ; 立即保存（如果需要跨调用）
    ; ...
```

**相同的设计理念**：上下文寄存器只在入口有效，被调用者负责保存。

#### 实际编译器行为

```go
func outer(a int) func() int {
    return func() int {
        fmt.Println(a)  // 调用其他函数
        return a
    }
}
```

**编译器生成的代码**：

```asm
TEXT outer.func1(SB), ABIInternal, $24-0
    ; 序言：保存 DX 到栈
    MOVQ    DX, closure-8(SP)     ; 立即 spill！

    ; 或者：保存到 callee-saved 寄存器
    ; MOVQ  DX, R12               ; 如果后续频繁使用

    ; 读取捕获变量
    MOVQ    closure-8(SP), CX     ; 从栈加载闭包指针
    MOVQ    8(CX), AX             ; 读取 a

    ; 调用 Println
    CALL    fmt.Println           ; DX 可能被破坏，没关系

    ; 再次读取 a（从栈，不是从 DX）
    MOVQ    closure-8(SP), CX
    MOVQ    8(CX), AX

    RET
```

#### 为什么这样设计更好

```
方案 A: DX (caller-saved)
─────────────────────────────────────
调用者:   设置 DX (1 条指令)
被调用者: 保存 DX (1 条指令，仅在需要时)
总开销:   1-2 条指令

方案 B: RBX (callee-saved)
─────────────────────────────────────
调用者:   保存 RBX, 设置 RBX, 恢复 RBX (3 条指令)
被调用者: 无需保存
总开销:   3 条指令

方案 A 更优！
```

#### 总结

| 问题 | 答案 |
|------|------|
| 为什么用 caller-saved DX？ | 调用者无需保存，开销更小 |
| DX 会被破坏怎么办？ | 闭包函数入口立即 spill 到栈 |
| 为什么不用 callee-saved？ | 调用者需要保存/恢复，开销更大 |
| 这和 C 一样吗？ | 是的，C 用 R10 也是 caller-saved |

### 7.11 DX 寄存器的保存与恢复

当闭包函数内部调用其他函数时：

```go
func outer(a int) func() int {
    return func() int {
        fmt.Println(a)  // 调用 fmt.Println
        return a
    }
}
```

**问题**：`fmt.Println` 可能破坏 DX

**解决**：编译器在闭包入口将 DX spill 到栈

```asm
TEXT outer.func1(SB), ABIInternal, $16-0
    ; 入口保存闭包上下文
    MOVQ    DX, closure-8(SP)     ; spill DX 到栈

    ; 读取 a 用于 Println
    MOVQ    8(DX), AX
    CALL    fmt.Println           ; DX 可能被破坏

    ; 恢复闭包上下文
    MOVQ    closure-8(SP), DX     ; reload DX
    MOVQ    8(DX), AX             ; 再次读取 a

    RET
```

### 7.12 与 C 语言闭包对比

| 方面 | Go 闭包 | C (GCC nested functions) |
|------|---------|--------------------------|
| 上下文传递 | DX 寄存器 | R10 寄存器 (trampoline) |
| 分配位置 | 堆 | 栈 (可能堆) |
| 生命周期 | GC 管理 | 手动/栈作用域 |
| 安全性 | 类型安全 | 需要特殊处理 |

---

### 7.12 Method 调用对比

```go
type T struct{ val int }
func (t *T) Add(x int) int { return t.val + x }

var t T
t.Add(10)
```

**ABI0**：
```asm
LEAQ    t(SB), AX            # receiver 地址
MOVQ    AX, 0(SP)            # receiver 入栈
MOVQ    $10, 8(SP)           # 参数入栈
CALL    T.Add(SB)
```

**ABIInternal**：
```asm
LEAQ    t(SB), AX            # receiver 在 RAX
MOVQ    $10, BX              # 参数在 RBX
CALL    T.Add(SB)            # 直接调用
```

### 6.9 性能关键差异

| 操作 | ABI0 延迟 | ABIInternal 延迟 |
|------|-----------|------------------|
| 传递 1 个 int 参数 | ~3 周期 | ~1 周期 |
| 读取返回值 | ~3 周期 | ~0 周期 |
| 典型函数调用开销 | ~10-15 周期 | ~3-5 周期 |

**为什么寄存器更快**：
1. 寄存器访问 ≈ 0-1 周期
2. L1 缓存访问 ≈ 3-4 周期
3. 避免了 store-to-load forwarding 延迟
4. 减少了指令数量

---

## 8. 源码导读

### 8.1 关键文件

```
go/src/
├── cmd/compile/internal/
│   ├── abi/
│   │   └── abi.go              # ABI 分配算法
│   ├── ssagen/
│   │   └── abi.go              # SSA 阶段 ABI 处理
│   └── ir/
│       └── func.go             # 函数 IR 表示
├── cmd/internal/obj/
│   ├── link.go                 # 目标文件格式
│   └── x86/
│       └── asm6.go             # x86 汇编生成
└── runtime/
    ├── asm_amd64.s             # 运行时汇编
    ├── stubs.go                # 运行时存根
    └── traceback.go            # 栈回溯
```

### 8.2 核心代码片段

#### ABI 配置（amd64）

```go
// src/cmd/compile/internal/abi/abi.go

var AMD64 = ABIConfig{
    IntArgRegs: []Register{
        {Name: "AX"}, {Name: "BX"}, {Name: "CX"},
        {Name: "DI"}, {Name: "SI"}, {Name: "R8"},
        {Name: "R9"}, {Name: "R10"}, {Name: "R11"},
    },
    FloatArgRegs: []Register{
        {Name: "X0"}, {Name: "X1"}, {Name: "X2"},
        // ... X3-X14
    },
    // 返回值使用相同的寄存器
    IntRetRegs:   /* same as IntArgRegs */,
    FloatRetRegs: /* same as FloatArgRegs */,
}
```

#### 参数分配

```go
// 简化的分配逻辑
func (a *ABIConfig) ABIAnalyze(ft *types.Func) *ABIParamResultInfo {
    result := &ABIParamResultInfo{}

    intReg := 0
    floatReg := 0
    stackOffset := int64(0)

    // 处理每个参数
    for _, param := range ft.Params {
        switch {
        case param.IsInteger() || param.IsPointer():
            if intReg < len(a.IntArgRegs) {
                result.InRegs = append(result.InRegs,
                    ABIParamAssignment{Reg: a.IntArgRegs[intReg]})
                intReg++
            } else {
                result.InStack = append(result.InStack,
                    ABIParamAssignment{Offset: stackOffset})
                stackOffset += param.Size()
            }
        // ... 处理其他类型
        }
    }

    return result
}
```

---

## 9. 调试与观察

### 9.1 查看生成的汇编

```bash
# 查看带 ABI 标注的汇编
go build -gcflags="-S" main.go 2>&1 | less

# 输出示例
"".Add STEXT nosplit size=4 args=0x10 locals=0x0 funcid=0x0 align=0x0 leaf
        0x0000 00000 (main.go:3)  TEXT    "".Add(SB), NOSPLIT|ABIInternal, $0-16
        0x0000 00000 (main.go:3)  FUNCDATA        $0, gclocals·g2BeySu+wFnoycgXfElmcg==(SB)
        0x0000 00000 (main.go:4)  ADDQ    BX, AX
        0x0003 00003 (main.go:4)  RET
```

### 9.2 使用 go tool objdump

```bash
go build -o main main.go
go tool objdump -S main | grep -A 20 "TEXT.*Add"
```

### 9.3 使用 delve 调试

```bash
dlv debug main.go

(dlv) break main.Add
(dlv) continue
(dlv) regs
# 查看 RAX, RBX 中的参数值
```

### 9.4 GOSSAFUNC 查看 SSA

```bash
GOSSAFUNC=Add go build main.go
# 打开生成的 ssa.html 查看编译过程
```

---

## 10. 性能对比

### 10.1 微基准测试

```go
func BenchmarkAdd(b *testing.B) {
    for i := 0; i < b.N; i++ {
        _ = Add(1, 2)
    }
}
```

典型结果：
```
ABI0:       ~2.5 ns/op
ABIInternal: ~0.8 ns/op (提升 ~70%)
```

### 10.2 实际应用

| 场景 | 性能提升 |
|------|----------|
| 小函数密集调用 | 10-15% |
| 普通应用 | 5-10% |
| IO 密集型 | 2-5% |
| 已优化的热路径 | 1-3% |

---

## 11. 与其他语言对比

### 11.1 C (System V AMD64 ABI)

| 特性 | Go ABIInternal | System V AMD64 |
|------|----------------|----------------|
| 整数寄存器 | 9个 | 6个 (RDI,RSI,RDX,RCX,R8,R9) |
| 浮点寄存器 | 15个 | 8个 (XMM0-7) |
| 返回值 | 多个 | RAX+RDX |
| 栈对齐 | 8字节 | 16字节 |
| 红区 | 无 | 128字节 |

### 11.2 为什么 Go 使用更多寄存器

1. Go 函数通常返回多个值（值+error）
2. 接口和切片占用多个寄存器
3. 方法调用需要额外的 receiver 参数

---

## 12. 常见问题

### 12.1 汇编代码如何兼容

```asm
// 方法1：继续使用 ABI0（推荐）
TEXT ·OldFunc(SB), NOSPLIT, $0-24
    MOVQ arg+0(FP), AX
    // ...

// 方法2：明确指定 ABI
//go:build go1.17
TEXT ·NewFunc<ABIInternal>(SB), NOSPLIT, $0
    // 参数已在寄存器中
    // ...
```

### 12.2 cgo 调用

cgo 使用 ABI0，编译器自动生成 wrapper：

```
Go 代码 (ABIInternal)
    -> cgo wrapper (ABI转换)
    -> C 代码 (C ABI)
```

### 12.3 reflect.MakeFunc

```go
// reflect 需要同时支持两种 ABI
fn := reflect.MakeFunc(typ, func(args []Value) []Value {
    // ...
})
```

内部实现会根据目标 ABI 生成相应的 trampoline。

---

## 13. 未来发展

1. **更多平台支持**：RISC-V、MIPS 等
2. **寄存器分配优化**：更智能的溢出决策
3. **尾调用优化**：与寄存器 ABI 协同
4. **Profile-Guided 优化**：热路径使用更多寄存器

---

## 14. 参考资料

1. [Go 1.17 Release Notes - Register ABI](https://go.dev/doc/go1.17#compiler)
2. [Proposal: Register-based Go calling convention](https://go.googlesource.com/proposal/+/master/design/40724-register-calling.md)
3. [Go Internal ABI Specification](https://go.dev/src/cmd/compile/abi-internal.md)
4. [Austin Clements - Register ABI Design](https://go.googlesource.com/proposal/+/master/design/27539-internal-abi.md)
5. [System V AMD64 ABI](https://refspecs.linuxbase.org/elf/x86_64-abi-0.99.pdf)
