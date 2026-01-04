# System V AMD64 ABI 详解

## 1. 概述

System V AMD64 ABI 是 Linux、macOS、BSD 等类 Unix 系统在 x86-64 架构上使用的标准调用约定。

### 1.1 与 Windows x64 ABI 对比

| 方面 | System V AMD64 | Windows x64 |
|------|----------------|-------------|
| 整数参数寄存器 | RDI, RSI, RDX, RCX, R8, R9 | RCX, RDX, R8, R9 |
| 浮点参数寄存器 | XMM0-XMM7 | XMM0-XMM3 |
| 返回值 | RAX, RDX | RAX |
| 红区 | 128 字节 | 无 |
| 影子空间 | 无 | 32 字节 |
| 栈对齐 | 16 字节 | 16 字节 |

---

## 2. 寄存器约定

### 2.1 通用寄存器分类

```
┌─────────────────────────────────────────────────────────────────┐
│                        寄存器用途分类                            │
├─────────────────────────────────────────────────────────────────┤
│  参数寄存器 (Caller-saved):                                     │
│    RDI, RSI, RDX, RCX, R8, R9  (整数参数 1-6)                  │
│                                                                  │
│  返回值寄存器:                                                   │
│    RAX (返回值低 64 位)                                         │
│    RDX (返回值高 64 位，用于 128 位返回)                        │
│                                                                  │
│  Caller-saved (调用者保存):                                     │
│    RAX, RCX, RDX, RSI, RDI, R8, R9, R10, R11                   │
│    XMM0-XMM15                                                   │
│                                                                  │
│  Callee-saved (被调用者保存):                                   │
│    RBX, RBP, R12, R13, R14, R15                                │
│                                                                  │
│  特殊用途:                                                       │
│    RSP - 栈指针                                                  │
│    RBP - 帧指针（可选）                                          │
│    R10 - 静态链指针（嵌套函数）                                  │
│    R11 - 临时寄存器                                              │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 参数传递寄存器

#### 整数/指针参数

| 参数位置 | 寄存器 |
|----------|--------|
| 第 1 个 | RDI |
| 第 2 个 | RSI |
| 第 3 个 | RDX |
| 第 4 个 | RCX |
| 第 5 个 | R8 |
| 第 6 个 | R9 |
| 第 7+ 个 | 栈 |

#### 浮点参数

| 参数位置 | 寄存器 |
|----------|--------|
| 第 1 个 | XMM0 |
| 第 2 个 | XMM1 |
| ... | ... |
| 第 8 个 | XMM7 |
| 第 9+ 个 | 栈 |

### 2.3 寄存器保存职责

```
函数调用边界:
                    调用前                调用后
                    ──────                ──────
Caller-saved:    调用者保存  ───►  可能被修改
                 (如果之后需要)

Callee-saved:    无需保存    ───►  保证不变
                              (被调用者负责保存/恢复)
```

**Caller-saved 示例**：

```c
void caller() {
    int a = 10;          // a 可能在 RAX
    some_func();         // RAX 可能被修改
    printf("%d", a);     // 必须在调用前保存 a
}
```

```asm
caller:
    push    rbp
    mov     rbp, rsp
    sub     rsp, 16

    mov     dword ptr [rbp-4], 10    ; a 保存到栈
    call    some_func
    mov     edi, dword ptr [rbp-4]   ; 从栈恢复 a
    call    printf

    leave
    ret
```

**Callee-saved 示例**：

```c
int callee(int x) {
    // 使用 RBX（callee-saved）
    int result = complex_calc(x);
    return result;
}
```

```asm
callee:
    push    rbp
    mov     rbp, rsp
    push    rbx                      ; 保存 RBX

    mov     ebx, edi                 ; x 保存到 RBX
    call    complex_calc
    ; RBX 仍然是 x

    pop     rbx                      ; 恢复 RBX
    pop     rbp
    ret
```

---

## 3. 参数传递规则

### 3.1 基本类型分类

ABI 将类型分为以下几类：

| 类别 | 类型 | 传递方式 |
|------|------|----------|
| INTEGER | 整数、指针、枚举 | 整数寄存器 |
| SSE | float, double | XMM 寄存器 |
| SSEUP | __m128 高半部分 | XMM 寄存器（与 SSE 配对） |
| X87 | long double | 栈 + x87 寄存器返回 |
| MEMORY | 大结构体 | 栈（通过指针） |
| NO_CLASS | void, 空结构体 | 不传递 |

### 3.2 参数传递算法

```
对于每个参数:
1. 确定参数的类别
2. 如果是 INTEGER 且有剩余整数寄存器:
   - 使用下一个整数寄存器
3. 如果是 SSE 且有剩余 XMM 寄存器:
   - 使用下一个 XMM 寄存器
4. 如果是 MEMORY 或寄存器用完:
   - 通过栈传递
5. 对于可变参数函数:
   - AL 必须包含使用的 XMM 寄存器数量
```

### 3.3 示例：混合类型参数

```c
void mixed(int a, double b, char *c, float d, long e, double f);
```

**参数分配**：

```
a (int)      -> RDI   (第 1 个整数)
b (double)   -> XMM0  (第 1 个浮点)
c (char*)    -> RSI   (第 2 个整数)
d (float)    -> XMM1  (第 2 个浮点)
e (long)     -> RDX   (第 3 个整数)
f (double)   -> XMM2  (第 3 个浮点)
```

**调用代码**：

```asm
; mixed(1, 2.0, "hello", 3.0f, 4, 5.0)
    mov     edi, 1                   ; a = 1
    movsd   xmm0, [.LC0]             ; b = 2.0
    lea     rsi, [.str]              ; c = "hello"
    movss   xmm1, [.LC1]             ; d = 3.0f
    mov     rdx, 4                   ; e = 4
    movsd   xmm2, [.LC2]             ; f = 5.0
    call    mixed
```

### 3.4 超过寄存器数量的参数

```c
void many_args(int a, int b, int c, int d, int e, int f,
               int g, int h, int i);
```

**参数分配**：

```
a -> RDI
b -> RSI
c -> RDX
d -> RCX
e -> R8
f -> R9
g -> [RSP+0]   (栈)
h -> [RSP+8]   (栈)
i -> [RSP+16]  (栈)
```

**栈布局**（调用后）：

```
高地址
         ┌─────────────────┐
RSP+24   │  返回地址        │
         ├─────────────────┤
RSP+16   │  i              │
         ├─────────────────┤
RSP+8    │  h              │
         ├─────────────────┤
RSP+0    │  g              │  <- RSP (调用前)
         └─────────────────┘
低地址
```

---

## 4. 结构体传递

### 4.1 小结构体（≤16 字节）

小结构体可以通过寄存器传递：

```c
struct Point {
    long x, y;  // 16 字节
};

void process(struct Point p);
```

**分类规则**：

```
Point 的分类:
  - 字节 0-7 (x): INTEGER -> 使用整数寄存器
  - 字节 8-15 (y): INTEGER -> 使用整数寄存器

结果: p.x -> RDI, p.y -> RSI
```

**调用代码**：

```asm
; process((Point){10, 20})
    mov     rdi, 10          ; p.x
    mov     rsi, 20          ; p.y
    call    process
```

### 4.2 混合类型结构体

```c
struct Mixed {
    int a;      // INTEGER
    float b;    // SSE
};

void process(struct Mixed m);
```

**分类**：

```
Mixed 的分类:
  - 字节 0-3 (a): INTEGER
  - 字节 4-7 (b): SSE

整个结构体 8 字节，但混合类型
根据规则: 如果 8 字节块包含混合类型，使用 INTEGER

结果: 整个结构体 -> RDI (作为 64 位整数)
```

### 4.3 大结构体（>16 字节）

```c
struct Large {
    long a, b, c;  // 24 字节
};

void process(struct Large l);
```

**传递方式**：

```
Large > 16 字节，分类为 MEMORY

调用者:
1. 在栈上分配空间
2. 复制结构体到栈
3. 传递指针（隐式）

实际上等价于: void process(struct Large *l)
```

**汇编**：

```asm
; 调用者
    sub     rsp, 24              ; 分配空间
    mov     qword ptr [rsp], 1   ; l.a
    mov     qword ptr [rsp+8], 2 ; l.b
    mov     qword ptr [rsp+16], 3; l.c
    mov     rdi, rsp             ; 传递指针
    call    process
    add     rsp, 24
```

### 4.4 结构体分类算法

```
classify_struct(struct S, size_t size):
    如果 size > 16 字节:
        返回 MEMORY

    对于每个 8 字节块:
        初始化类别为 NO_CLASS

        对于块中的每个成员:
            member_class = classify(member)
            合并类别

    合并规则:
        - MEMORY 优先级最高
        - INTEGER 覆盖 NO_CLASS
        - SSE 覆盖 NO_CLASS
        - 如果一个块既有 INTEGER 又有 SSE，结果是 INTEGER
```

### 4.5 结构体传递示例汇总

```c
// 1. 单个 8 字节块
struct S1 { int a, b; };           // RDI (打包成 64 位)

// 2. 两个 8 字节块
struct S2 { long a, b; };          // RDI, RSI

// 3. 混合 int + float
struct S3 { int a; float b; };     // RDI (合并为 INTEGER)

// 4. 纯浮点
struct S4 { float a, b; };         // XMM0 (64 位，包含两个 float)

// 5. double + int
struct S5 { double a; int b; };    // XMM0, RDI

// 6. 大结构体
struct S6 { long a, b, c; };       // 通过栈（MEMORY）
```

---

## 5. 返回值规则

### 5.1 基本类型返回

| 类型 | 返回寄存器 |
|------|-----------|
| int, long, pointer | RAX |
| 128 位整数 | RAX (低), RDX (高) |
| float, double | XMM0 |
| __m128 | XMM0 |
| long double | ST(0) (x87) |

### 5.2 结构体返回

#### 小结构体（≤16 字节）

```c
struct Point {
    long x, y;
};

struct Point make_point(long x, long y) {
    return (struct Point){x, y};
}
```

**返回方式**：

```asm
make_point:
    mov     rax, rdi     ; x -> RAX
    mov     rdx, rsi     ; y -> RDX
    ret
```

#### 大结构体（>16 字节）

```c
struct Large {
    long a, b, c;
};

struct Large make_large() {
    return (struct Large){1, 2, 3};
}
```

**返回方式**：

调用者提供隐藏的第一个参数（返回值地址）：

```asm
; 调用者
    sub     rsp, 24          ; 分配返回值空间
    mov     rdi, rsp         ; 传递返回地址
    call    make_large
    ; 结果已在 [rsp]

; make_large
make_large:
    mov     qword ptr [rdi], 1    ; 写入调用者提供的地址
    mov     qword ptr [rdi+8], 2
    mov     qword ptr [rdi+16], 3
    mov     rax, rdi              ; 返回地址
    ret
```

### 5.3 返回值寄存器汇总

```
┌─────────────────────────────────────────────────────────────────┐
│                        返回值规则                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  整数/指针类型:                                                  │
│    ≤64 位  -> RAX                                               │
│    ≤128 位 -> RAX:RDX                                           │
│                                                                  │
│  浮点类型:                                                       │
│    float/double      -> XMM0                                    │
│    __m128           -> XMM0                                     │
│    __m256           -> YMM0                                     │
│    complex float    -> XMM0 (实部), XMM1 (虚部)                 │
│    long double      -> ST(0)                                    │
│                                                                  │
│  结构体:                                                         │
│    ≤16 字节, INTEGER+INTEGER -> RAX, RDX                        │
│    ≤16 字节, SSE+SSE        -> XMM0, XMM1                       │
│    ≤16 字节, INTEGER+SSE    -> RAX, XMM0                        │
│    >16 字节                  -> 通过隐藏指针参数                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6. 栈帧布局

### 6.1 标准栈帧

```
高地址
         ┌─────────────────────────┐
         │  调用者的栈帧           │
         ├─────────────────────────┤
  +N     │  栈参数 N               │
         ├─────────────────────────┤
         │  ...                    │
         ├─────────────────────────┤
  +16    │  栈参数 1               │
         ├─────────────────────────┤
  +8     │  返回地址               │  <- 调用后 RSP
         ├─────────────────────────┤
  RBP -> │  保存的 RBP             │  <- 可选
         ├─────────────────────────┤
         │  保存的 callee-saved    │
         │  寄存器                 │
         ├─────────────────────────┤
         │  局部变量               │
         ├─────────────────────────┤
  RSP -> │  红区 (128 字节)        │  <- 仅限叶函数
         └─────────────────────────┘
低地址
```

### 6.2 栈对齐要求

**16 字节对齐规则**：

```
在 CALL 指令之前，RSP 必须是 16 的倍数
CALL 指令压入 8 字节返回地址后，RSP % 16 == 8

函数入口:
    RSP % 16 == 8  (返回地址占 8 字节)

函数体:
    必须确保任何 CALL 之前 RSP % 16 == 0
```

**示例**：

```asm
func:
    ; 入口: RSP % 16 == 8
    push    rbp              ; RSP % 16 == 0
    mov     rbp, rsp

    sub     rsp, 24          ; 分配局部变量
    ; 24 不是 16 的倍数！需要调整

    ; 正确做法:
    sub     rsp, 32          ; 或 sub rsp, 40（包含对齐填充）
```

### 6.3 红区（Red Zone）

**定义**：RSP 之下的 128 字节区域，叶函数可以使用而无需调整 RSP。

```
         ┌─────────────────────────┐
  RSP -> │                         │
         ├─────────────────────────┤
         │                         │
         │      红区               │
         │    (128 字节)           │
         │                         │
         │  叶函数可直接使用        │
         │                         │
         ├─────────────────────────┤
  RSP-128│                         │
         └─────────────────────────┘
```

**叶函数优化**：

```c
int leaf(int a, int b) {
    int x = a + 1;
    int y = b + 2;
    return x + y;
}
```

```asm
; 使用红区，无需 sub rsp
leaf:
    mov     dword ptr [rsp-4], edi   ; x 在红区
    add     dword ptr [rsp-4], 1
    mov     dword ptr [rsp-8], esi   ; y 在红区
    add     dword ptr [rsp-8], 2
    mov     eax, dword ptr [rsp-4]
    add     eax, dword ptr [rsp-8]
    ret
```

**限制**：
- 仅叶函数可用（不调用其他函数）
- 信号处理程序会覆盖红区
- 编译器选项 `-mno-red-zone` 禁用

---

## 7. 可变参数函数

### 7.1 调用约定

```c
int printf(const char *fmt, ...);

printf("%d %f", 42, 3.14);
```

**调用规则**：

```
1. 固定参数: 正常传递
2. 可变参数: 按类型使用对应寄存器/栈
3. AL 寄存器: 必须包含使用的 XMM 寄存器数量
```

**汇编**：

```asm
    lea     rdi, [.fmt]      ; fmt
    mov     esi, 42          ; 第一个可变参数（整数）
    movsd   xmm0, [.pi]      ; 第二个可变参数（浮点）
    mov     al, 1            ; 使用了 1 个 XMM 寄存器
    call    printf
```

### 7.2 va_list 实现

```c
typedef struct {
    unsigned int gp_offset;     // 下一个整数参数在 reg_save_area 中的偏移
    unsigned int fp_offset;     // 下一个浮点参数的偏移
    void *overflow_arg_area;    // 栈上可变参数的地址
    void *reg_save_area;        // 寄存器保存区域
} va_list[1];
```

**寄存器保存区域布局**：

```
reg_save_area:
┌─────────────────┐  offset 0
│  RDI            │
├─────────────────┤  offset 8
│  RSI            │
├─────────────────┤  offset 16
│  RDX            │
├─────────────────┤  offset 24
│  RCX            │
├─────────────────┤  offset 32
│  R8             │
├─────────────────┤  offset 40
│  R9             │
├─────────────────┤  offset 48
│  XMM0           │
├─────────────────┤  offset 64
│  XMM1           │
├─────────────────┤  ...
│  ...            │
├─────────────────┤  offset 160
│  XMM7           │
└─────────────────┘  offset 176
```

### 7.3 可变参数函数的序言

```asm
varargs_func:
    ; 保存所有可能的参数寄存器
    sub     rsp, 176

    ; 保存整数寄存器
    mov     qword ptr [rsp+0], rdi
    mov     qword ptr [rsp+8], rsi
    mov     qword ptr [rsp+16], rdx
    mov     qword ptr [rsp+24], rcx
    mov     qword ptr [rsp+32], r8
    mov     qword ptr [rsp+40], r9

    ; 保存浮点寄存器（根据 AL 的值）
    test    al, al
    je      .no_xmm
    movaps  [rsp+48], xmm0
    movaps  [rsp+64], xmm1
    ; ...
.no_xmm:
    ; 函数体
```

---

## 8. 调用示例

### 8.1 完整的函数调用流程

```c
long foo(long a, long b) {
    return a + b;
}

void caller() {
    long result = foo(10, 20);
}
```

**汇编**：

```asm
foo:
    ; 入口: a 在 RDI, b 在 RSI
    lea     rax, [rdi + rsi]  ; rax = a + b
    ret

caller:
    push    rbp
    mov     rbp, rsp
    sub     rsp, 16           ; 对齐 + 局部变量

    ; 准备参数
    mov     edi, 10           ; a = 10
    mov     esi, 20           ; b = 20

    ; 调用
    call    foo

    ; 返回值在 RAX
    mov     qword ptr [rbp-8], rax  ; result = foo(...)

    leave
    ret
```

### 8.2 复杂参数传递

```c
struct Point { double x, y; };

double distance(struct Point a, struct Point b) {
    double dx = a.x - b.x;
    double dy = a.y - b.y;
    return sqrt(dx*dx + dy*dy);
}
```

**参数分类**：
- `Point a`: 16 字节，两个 SSE 类成员 -> XMM0, XMM1
- `Point b`: 16 字节，两个 SSE 类成员 -> XMM2, XMM3

```asm
; 调用 distance({1.0, 2.0}, {4.0, 6.0})
    movsd   xmm0, [.one]     ; a.x = 1.0
    movsd   xmm1, [.two]     ; a.y = 2.0
    movsd   xmm2, [.four]    ; b.x = 4.0
    movsd   xmm3, [.six]     ; b.y = 6.0
    call    distance
    ; 返回值在 XMM0
```

### 8.3 嵌套函数（静态链）

```c
void outer() {
    int x = 10;

    void inner() {
        printf("%d\n", x);  // 访问外层变量
    }

    inner();
}
```

**R10 用于静态链**：

```asm
outer:
    push    rbp
    mov     rbp, rsp
    sub     rsp, 16

    mov     dword ptr [rbp-4], 10    ; x

    lea     r10, [rbp]               ; 静态链 = outer 的帧
    call    inner

    leave
    ret

inner:
    ; R10 指向 outer 的帧
    mov     esi, dword ptr [r10-4]   ; 读取 x
    lea     rdi, [.fmt]
    call    printf
    ret
```

---

## 9. 异常处理 (DWARF)

### 9.1 CFI (Call Frame Information)

编译器生成 `.eh_frame` 节用于栈展开：

```asm
foo:
    .cfi_startproc
    push    rbp
    .cfi_def_cfa_offset 16       ; CFA 现在在 RSP+16
    .cfi_offset rbp, -16         ; RBP 保存在 CFA-16
    mov     rbp, rsp
    .cfi_def_cfa_register rbp    ; CFA 现在用 RBP

    ; 函数体

    pop     rbp
    .cfi_def_cfa rsp, 8
    ret
    .cfi_endproc
```

### 9.2 异常表

```
.gcc_except_table:
    Landing pads: 异常处理入口点
    Type table: 可捕获的异常类型
    Call site table: 哪些调用点有异常处理
```

---

## 10. Spill 机制

### 10.1 什么是 Spill

Spill（溢出）是指将寄存器中的值保存到栈内存中，以便：
1. 释放寄存器给其他变量使用
2. 在函数调用后恢复变量值
3. 满足某些操作的内存操作数要求

### 10.2 C 的 Spill 策略：按需 Spill

C 编译器使用**寄存器分配算法**（如图着色）来决定哪些变量 spill：

```c
int example(int a, int b, int c) {
    int x = a + 1;      // x 可能在寄存器
    int y = b + 2;      // y 可能在寄存器
    foo();              // 调用后 caller-saved 寄存器可能被破坏
    return x + y + c;   // x, y 需要在调用前保存
}
```

**编译器分析**：

```
变量活跃性分析:
  x: 定义于 L1, 使用于 L4  (跨 foo() 调用)
  y: 定义于 L2, 使用于 L4  (跨 foo() 调用)
  c: 参数, 使用于 L4       (跨 foo() 调用)

结论:
  x, y, c 都跨函数调用存活
  必须 spill 或使用 callee-saved 寄存器
```

**生成的汇编（优化后）**：

```asm
example:
    push    rbp
    mov     rbp, rsp
    push    rbx                  ; 保存 callee-saved
    push    r12
    push    r13

    ; a 在 EDI, b 在 ESI, c 在 EDX
    lea     ebx, [rdi + 1]       ; x = a + 1，用 RBX (callee-saved)
    lea     r12d, [rsi + 2]      ; y = b + 2，用 R12 (callee-saved)
    mov     r13d, edx            ; c 保存到 R13 (callee-saved)

    call    foo                  ; RBX, R12, R13 不会被破坏

    lea     eax, [rbx + r12]     ; x + y
    add     eax, r13d            ; + c

    pop     r13
    pop     r12
    pop     rbx
    pop     rbp
    ret
```

**关键点**：编译器选择使用 callee-saved 寄存器（RBX, R12, R13）而不是 spill 到栈，因为这样更高效。

### 10.3 Caller-saved 寄存器的 Spill

当必须使用 caller-saved 寄存器时：

```c
int compute(int a) {
    int x = expensive_calc(a);   // 返回值在 EAX
    int y = another_calc(a);     // 返回值在 EAX，会覆盖 x
    return x + y;
}
```

```asm
compute:
    push    rbp
    mov     rbp, rsp
    sub     rsp, 16

    mov     dword ptr [rbp-4], edi    ; spill 参数 a

    mov     edi, dword ptr [rbp-4]    ; 加载 a
    call    expensive_calc
    mov     dword ptr [rbp-8], eax    ; spill x (因为下次调用会覆盖 EAX)

    mov     edi, dword ptr [rbp-4]    ; 加载 a
    call    another_calc
    ; y 在 EAX

    add     eax, dword ptr [rbp-8]    ; x + y

    leave
    ret
```

### 10.4 Go 的 Spill 策略：保守 Spill

Go 采用**保守策略**，在函数入口将所有寄存器参数 spill：

```go
func example(a, b, c int) int {
    x := a + 1
    y := b + 2
    foo()
    return x + y + c
}
```

```asm
; Go 生成的汇编
TEXT example(SB), ABIInternal, $32-0
    ; 入口：a 在 AX, b 在 BX, c 在 CX

    ; 立即 spill 所有参数到栈（保守策略）
    MOVQ    AX, a-8(SP)
    MOVQ    BX, b-16(SP)
    MOVQ    CX, c-24(SP)

    ; x := a + 1
    MOVQ    a-8(SP), AX
    ADDQ    $1, AX
    MOVQ    AX, x-32(SP)

    ; y := b + 2
    MOVQ    b-16(SP), AX
    ADDQ    $2, AX
    MOVQ    AX, y-40(SP)

    CALL    foo(SB)

    ; return x + y + c
    MOVQ    x-32(SP), AX
    ADDQ    y-40(SP), AX
    ADDQ    c-24(SP), AX
    RET
```

### 10.5 Spill 策略对比

| 方面 | C (按需 Spill) | Go (保守 Spill) |
|------|----------------|-----------------|
| 时机 | 需要时才 spill | 函数入口全部 spill |
| 决策 | 编译器优化决定 | 固定规则 |
| 分析 | 活跃性分析 + 寄存器分配 | 无需分析 |
| 效率 | 高（最小化内存操作） | 低（多余的内存操作） |
| 实现复杂度 | 高 | 低 |
| GC 友好 | 需要额外信息 | 天然友好 |
| 调试友好 | 变量可能在寄存器 | 变量总在栈上 |

### 10.6 为什么 Go 选择保守策略

```
┌─────────────────────────────────────────────────────────────────┐
│                    Go 保守 Spill 的原因                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. GC 扫描需要                                                  │
│     - GC 需要找到所有指针                                        │
│     - 如果指针在寄存器中，需要额外的元数据                        │
│     - 栈上的指针位置是固定的，容易扫描                           │
│                                                                  │
│  2. Goroutine 抢占                                               │
│     - 任意时刻可能被抢占                                         │
│     - 抢占时需要保存完整状态                                     │
│     - 如果值在栈上，抢占更简单                                   │
│                                                                  │
│  3. Defer/Panic 支持                                             │
│     - defer 需要访问函数参数                                     │
│     - panic 需要栈展开                                           │
│     - 参数在栈上简化了这些操作                                   │
│                                                                  │
│  4. 调试器支持                                                   │
│     - 调试时变量值总能从栈上读取                                 │
│     - 不需要复杂的 DWARF 信息跟踪寄存器位置                      │
│                                                                  │
│  5. 编译速度                                                     │
│     - 不需要复杂的寄存器分配算法                                 │
│     - 编译更快                                                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 10.7 C 的寄存器分配算法

C 编译器使用复杂的算法决定 spill：

```
1. 活跃性分析 (Liveness Analysis)
   - 计算每个变量的活跃区间
   - 识别跨调用点存活的变量

2. 构建干涉图 (Interference Graph)
   - 节点 = 变量
   - 边 = 两个变量同时活跃

3. 图着色 (Graph Coloring)
   - 颜色 = 寄存器
   - 尝试用 K 种颜色着色（K = 可用寄存器数）

4. Spill 决策
   - 如果无法着色，选择一个变量 spill
   - 选择标准：使用频率低、活跃区间长

5. 重新着色
   - spill 后重新尝试着色
   - 重复直到成功
```

**示例**：

```c
int complex(int a, int b, int c, int d, int e, int f, int g) {
    // 7 个参数，只有 6 个参数寄存器
    // g 必须通过栈传递

    int x = a + b;
    int y = c + d;
    int z = e + f;
    foo();              // 调用点
    return x + y + z + g;
}
```

```
干涉图:
  a, b, c, d, e, f, g, x, y, z 都可能互相干涉

寄存器分配:
  - 使用 callee-saved: RBX, R12, R13, R14, R15 (5个)
  - 需要 x, y, z, g 跨调用存活 (4个)
  - 可以分配到 callee-saved 寄存器

  或者:
  - 将 x, y, z 中的一个 spill 到栈
```

### 10.8 Spill 的性能影响

```
              寄存器访问          栈访问 (L1 命中)
延迟:         ~0-1 周期           ~3-4 周期
吞吐:         多个/周期           1-2 个/周期

Spill 代价:
  - 写入栈: 1 条 MOV 指令 + 3-4 周期延迟
  - 从栈读取: 1 条 MOV 指令 + 3-4 周期延迟
  - 总计: ~6-8 周期 vs 寄存器操作的 ~0-1 周期
```

### 10.9 优化：减少 Spill

**C 编译器优化**：

```c
// 原始代码
void process(int *arr, int n) {
    for (int i = 0; i < n; i++) {
        arr[i] = compute(arr[i]);
    }
}
```

**未优化**（每次迭代 spill）：

```asm
.loop:
    mov     [rsp-8], rdi         ; spill arr
    mov     [rsp-16], rsi        ; spill n
    mov     [rsp-24], ecx        ; spill i

    mov     edi, [rdi + rcx*4]
    call    compute

    mov     rdi, [rsp-8]         ; reload
    mov     ecx, [rsp-24]
    mov     [rdi + rcx*4], eax

    inc     ecx
    cmp     ecx, [rsp-16]
    jl      .loop
```

**优化后**（使用 callee-saved）：

```asm
    push    rbx
    push    r12
    push    r13

    mov     rbx, rdi             ; arr in RBX (callee-saved)
    mov     r12d, esi            ; n in R12
    xor     r13d, r13d           ; i in R13

.loop:
    mov     edi, [rbx + r13*4]
    call    compute
    mov     [rbx + r13*4], eax

    inc     r13d
    cmp     r13d, r12d
    jl      .loop

    pop     r13
    pop     r12
    pop     rbx
    ret
```

---

## 11. C 闭包与嵌套函数

C 语言本身没有原生闭包，但有几种实现方式：

### 11.1 GCC 嵌套函数（Nested Functions）

GCC 扩展支持嵌套函数，可以访问外层函数的变量：

```c
void outer(int x) {
    int y = 20;

    int inner(int a) {
        return a + x + y;  // 访问外层变量
    }

    printf("%d\n", inner(5));  // 输出 5 + 10 + 20 = 35
}

outer(10);
```

#### 静态链（Static Chain）

GCC 使用 **R10 寄存器**传递静态链（指向外层栈帧的指针）：

```
┌─────────────────────────────────────────────────────────────────┐
│                    GCC 嵌套函数调用约定                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  调用嵌套函数时:                                                 │
│    R10 = 静态链指针（指向外层函数的栈帧）                        │
│    RDI, RSI, ... = 普通参数（与正常函数相同）                    │
│                                                                  │
│  嵌套函数通过 R10 访问外层变量                                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**生成的汇编**：

```asm
outer:
    push    rbp
    mov     rbp, rsp
    sub     rsp, 32

    mov     dword ptr [rbp-4], edi    ; x = 10
    mov     dword ptr [rbp-8], 20     ; y = 20

    ; 调用 inner(5)
    lea     r10, [rbp]                ; R10 = 静态链 = outer 的帧指针
    mov     edi, 5                    ; a = 5
    call    inner
    ; ...

inner:
    ; 入口: R10 = 静态链, EDI = a
    push    rbp
    mov     rbp, rsp

    mov     eax, edi                  ; eax = a
    add     eax, dword ptr [r10-4]    ; eax += x (通过静态链访问)
    add     eax, dword ptr [r10-8]    ; eax += y (通过静态链访问)

    pop     rbp
    ret
```

**内存布局**：

```
outer 的栈帧:
         ┌─────────────────┐
  RBP -> │  saved RBP      │
         ├─────────────────┤
  -4     │  x = 10         │  <- R10-4 访问
         ├─────────────────┤
  -8     │  y = 20         │  <- R10-8 访问
         └─────────────────┘
               ↑
               │
         R10 指向这里
               │
inner 的栈帧:  │
         ┌─────────────────┐
  RBP -> │  saved RBP      │
         ├─────────────────┤
         │  ...            │
         └─────────────────┘
```

#### Trampoline（蹦床）

当嵌套函数的地址被取出时，GCC 生成 trampoline：

```c
void outer(int x) {
    int inner(int a) {
        return a + x;
    }

    int (*fp)(int) = inner;  // 取嵌套函数地址
    fp(5);                    // 通过函数指针调用
}
```

**问题**：函数指针只有 8 字节，无法同时存储函数地址和静态链。

**解决方案**：Trampoline - 在栈上生成一小段可执行代码：

```asm
; Trampoline 代码（在栈上生成）
trampoline:
    mov     r10, <static_chain>    ; 加载静态链
    jmp     inner                   ; 跳转到真正的函数
```

```
栈布局:
         ┌─────────────────────────────┐
         │  trampoline 代码 (~20字节)   │  <- fp 指向这里
         │    mov r10, imm64           │
         │    jmp inner                │
         ├─────────────────────────────┤
         │  outer 的局部变量           │
         └─────────────────────────────┘
```

**安全问题**：需要栈可执行，现代系统默认禁止。

### 11.2 Apple Blocks

Apple 扩展的 Blocks 是另一种闭包实现：

```c
#include <Block.h>

void example() {
    int x = 10;

    int (^block)(int) = ^(int a) {
        return a + x;
    };

    printf("%d\n", block(5));  // 输出 15
}
```

#### Block 的内存结构

```c
struct Block_layout {
    void *isa;                    // 类指针
    int flags;                    // 标志位
    int reserved;
    void (*invoke)(void *, ...);  // 函数指针
    struct Block_descriptor *descriptor;
    // 捕获的变量紧随其后
    int captured_x;
};
```

**内存布局**：

```
Block 对象（堆上）:
┌─────────────────────────┐  +0
│  isa                    │
├─────────────────────────┤  +8
│  flags                  │
├─────────────────────────┤  +12
│  reserved               │
├─────────────────────────┤  +16
│  invoke (函数指针)       │  ──► Block 函数代码
├─────────────────────────┤  +24
│  descriptor             │
├─────────────────────────┤  +32
│  captured_x = 10        │  ← 捕获的变量
└─────────────────────────┘
```

#### Block 的调用过程

```asm
; block(5) 的调用
; block 指针在某寄存器，假设 RBX

    mov     rdi, rbx              ; 第一个参数 = block 指针本身
    mov     esi, 5                ; 第二个参数 = a
    call    qword ptr [rbx+16]    ; 调用 invoke 函数

; Block 函数内部
block_invoke:
    ; RDI = block 指针
    ; ESI = a
    mov     eax, esi              ; eax = a
    add     eax, dword ptr [rdi+32]  ; eax += captured_x
    ret
```

**关键点**：Block 指针作为第一个隐藏参数传递给 invoke 函数。

### 11.3 手动实现闭包

最通用的方式是手动实现：

```c
// 闭包上下文
typedef struct {
    int x;
    int y;
} Context;

// 闭包函数
int closure_func(Context *ctx, int a) {
    return a + ctx->x + ctx->y;
}

// 使用
void example() {
    Context ctx = {10, 20};
    int result = closure_func(&ctx, 5);  // 35
}
```

**优化版本**：使用 thunk

```c
// 通用闭包结构
typedef struct {
    void *func;      // 函数指针
    void *context;   // 上下文指针
} Closure;

// 调用闭包的宏
#define CALL_CLOSURE(c, ...) \
    ((int (*)(void*, ...))((c)->func))((c)->context, ##__VA_ARGS__)

// 使用
Closure c = {closure_func, &ctx};
int result = CALL_CLOSURE(&c, 5);
```

### 11.4 libffi 动态闭包

libffi 可以在运行时创建闭包：

```c
#include <ffi.h>

void closure_fn(ffi_cif *cif, void *ret, void **args, void *userdata) {
    int *ctx = (int *)userdata;
    int a = *(int *)args[0];
    *(int *)ret = a + *ctx;
}

void example() {
    ffi_cif cif;
    ffi_closure *closure;
    void *code;
    int ctx = 10;

    // 创建闭包
    closure = ffi_closure_alloc(sizeof(ffi_closure), &code);

    ffi_type *arg_types[] = {&ffi_type_sint};
    ffi_prep_cif(&cif, FFI_DEFAULT_ABI, 1, &ffi_type_sint, arg_types);
    ffi_prep_closure_loc(closure, &cif, closure_fn, &ctx, code);

    // 调用
    int (*fp)(int) = (int (*)(int))code;
    printf("%d\n", fp(5));  // 输出 15

    ffi_closure_free(closure);
}
```

### 11.5 闭包上下文传递对比

| 实现方式 | 上下文传递 | 函数指针大小 | 堆分配 |
|----------|-----------|--------------|--------|
| GCC 嵌套函数 | R10 寄存器 | 8 字节 (需 trampoline) | 否（栈） |
| Apple Blocks | 第一个参数 | 8 字节 | 是 |
| 手动实现 | 显式参数 | 16 字节（函数+上下文） | 可选 |
| libffi | 内部处理 | 8 字节 | 是 |
| **Go 闭包** | **DX 寄存器** | **8 字节** | **是** |

### 11.6 与 Go 闭包对比

```
┌─────────────────────────────────────────────────────────────────┐
│                    闭包上下文传递对比                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  GCC 嵌套函数:                                                   │
│    R10 = 静态链（指向外层栈帧）                                  │
│    访问: [R10 + offset]                                         │
│                                                                  │
│  Apple Blocks:                                                   │
│    RDI = Block 指针（隐藏第一个参数）                            │
│    访问: [RDI + offset]                                         │
│                                                                  │
│  Go 闭包:                                                        │
│    DX = 闭包对象指针                                             │
│    访问: [DX + offset]                                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**调用代码对比**：

```asm
; GCC 嵌套函数调用
    lea     r10, [rbp]            ; 静态链
    mov     edi, 5                ; 参数
    call    nested_func

; Apple Block 调用
    mov     rdi, block_ptr        ; Block 指针作为第一个参数
    mov     esi, 5                ; 普通参数
    call    [rdi+16]              ; 调用 invoke

; Go 闭包调用
    mov     dx, closure_ptr       ; 闭包上下文
    mov     ax, 5                 ; 参数
    call    [dx]                  ; 调用函数指针
```

### 11.7 C++ Lambda 与闭包

C++11 的 lambda 与这些机制类似：

```cpp
auto lambda = [x, y](int a) {
    return a + x + y;
};
```

编译器生成的等价代码：

```cpp
class __lambda_1 {
    int x, y;  // 捕获的变量
public:
    __lambda_1(int x_, int y_) : x(x_), y(y_) {}

    int operator()(int a) const {
        return a + x + y;
    }
};

__lambda_1 lambda(x, y);
lambda(5);  // 调用 operator()
```

**调用方式**：this 指针（隐藏第一个参数）+ 普通参数

```asm
; lambda(5) 调用
    lea     rdi, [lambda_obj]    ; this 指针
    mov     esi, 5               ; 参数 a
    call    __lambda_1::operator()
```

---

## 12. 与 Go ABI 对比

| 方面 | System V AMD64 | Go ABIInternal |
|------|----------------|----------------|
| 整数参数 | RDI,RSI,RDX,RCX,R8,R9 (6个) | AX,BX,CX,DI,SI,R8-R11 (9个) |
| 浮点参数 | XMM0-7 (8个) | X0-X14 (15个) |
| 返回值 | RAX,RDX / XMM0,XMM1 | 与参数相同寄存器 |
| 栈对齐 | 16 字节 | 8 字节 |
| 红区 | 128 字节 | 无 |
| 闭包上下文 | R10 (静态链) | DX |
| 特殊寄存器 | 无 | R14 = G 指针 |
| 多返回值 | 最多 2 个 | 任意多个 |

### 10.1 为什么 Go 不直接用 C ABI

1. **多返回值**：Go 函数常返回 `(value, error)`，C ABI 最多返回 2 个值
2. **G 指针**：每个 goroutine 需要快速访问 G 结构体
3. **栈增长**：Go 需要在函数入口检查栈空间
4. **GC 集成**：需要知道寄存器中哪些是指针

### 10.2 cgo 的 ABI 转换

```
Go 代码 (ABIInternal)
    ↓
    参数: AX,BX,CX... -> RDI,RSI,RDX...
    返回: AX... <- RAX...
    ↓
C 代码 (System V)
```

---

## 13. 实用工具

### 13.1 查看调用约定

```bash
# GCC 生成的汇编
gcc -S -O2 -fverbose-asm example.c

# 查看 CFI 信息
objdump -d --dwarf=frames example.o

# readelf 查看符号
readelf -s example.o
```

### 13.2 调试 ABI 问题

```bash
# GDB 查看寄存器
(gdb) info registers

# 查看调用栈
(gdb) bt full

# 单步跟踪调用
(gdb) si
```

---

## 14. 参考资料

1. [System V AMD64 ABI Specification](https://refspecs.linuxbase.org/elf/x86_64-abi-0.99.pdf)
2. [AMD64 Architecture Programmer's Manual](https://developer.amd.com/resources/developer-guides-manuals/)
3. [GCC Internals - Calling Conventions](https://gcc.gnu.org/onlinedocs/gccint/Calling-Conventions.html)
4. [DWARF Debugging Standard](http://dwarfstd.org/)
5. [Linux x86-64 ABI](https://wiki.osdev.org/System_V_ABI)
