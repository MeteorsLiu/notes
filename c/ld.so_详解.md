# ld.so 详解 - 动态链接器完全指南

> 记录时间：2025-11-27
> 用途：包管理器开发参考文档

## 目录
- [ld.so 是什么](#ldso-是什么)
- [为什么需要它](#为什么需要它)
- [工作流程详解](#ldso-的工作流程)
- [实际例子](#实际例子)
- [关键概念](#关键概念)
- [为什么 ld.so 和 libc 强耦合](#为什么-ldso-和-libc-强耦合)
- [不同 libc 的 ld.so](#不同-libc-的-ldso)
- [包管理器开发的影响](#包管理器开发的影响)

---

## ld.so 是什么

**`ld.so`** (也叫 **动态链接器** 或 **dynamic linker/loader**) 是一个特殊的程序，负责在程序启动时加载和链接共享库（动态库）。

完整名称通常是：
- Linux x86_64: `ld-linux-x86-64.so.2`
- Linux ARM: `ld-linux-armhf.so.3`
- musl libc: `ld-musl-x86_64.so.1`

## 为什么需要它？

### 对比：静态链接 vs 动态链接

#### **静态链接**（不需要 ld.so）
```
编译时：
[myapp.o] + [libc.a] + [libm.a] → [myapp 可执行文件 (5MB)]
                                     ↑
                                  包含所有代码

运行时：
内核直接加载 myapp → 直接运行
```

优点：
- 不依赖外部库
- 部署简单
- 启动快（无需加载库）

缺点：
- 文件体积大
- 无法共享库代码（浪费内存）
- 更新库需要重新编译

#### **动态链接**（需要 ld.so）
```
编译时：
[myapp.o] + [libc.so 的引用] → [myapp 可执行文件 (20KB)]
                                  ↑
                               只包含引用信息

运行时：
内核加载 myapp → 发现需要 libc.so
              ↓
          调用 ld.so 来处理
              ↓
          ld.so 加载 libc.so 并链接符号
              ↓
          程序开始运行
```

优点：
- 文件体积小
- 多个程序共享同一份库（节省内存）
- 更新库无需重新编译程序

缺点：
- 依赖外部库
- 启动稍慢（需要动态加载）
- 版本兼容性问题

## ld.so 的工作流程

### 详细步骤（10 个阶段）

```
1. 程序启动
   用户执行: ./myapp
   ↓

2. 内核加载 ELF 文件
   内核读取 myapp 的 ELF 头
   发现 PT_INTERP 段: "/lib64/ld-linux-x86-64.so.2"
   ↓

3. 内核启动 ld.so
   内核将控制权交给 ld.so (不是 myapp!)
   ↓

4. ld.so 初始化
   设置自己的内存空间
   准备加载其他库
   ↓

5. ld.so 解析依赖
   读取 myapp 的 DYNAMIC 段
   找到所有 NEEDED 条目:
     - libc.so.6
     - libpthread.so.0
     - libm.so.6
   ↓

6. ld.so 搜索和加载库
   在以下路径搜索（按优先级）:
     1. RPATH (编译时指定)
     2. LD_LIBRARY_PATH (环境变量)
     3. RUNPATH (编译时指定，可被 LD_LIBRARY_PATH 覆盖)
     4. /etc/ld.so.cache (系统缓存)
     5. /lib, /usr/lib (默认路径)

   找到后，加载到内存
   ↓

7. ld.so 处理符号解析
   myapp 调用了 printf()
   ld.so 在 libc.so.6 中找到 printf 的地址
   修改 myapp 的 GOT (Global Offset Table)
   将 printf 的地址填入
   ↓

8. ld.so 执行初始化函数
   调用每个库的 .init 段
   调用 __attribute__((constructor)) 标记的函数
   ↓

9. ld.so 跳转到程序入口
   调用 libc 的 __libc_start_main()
   最终调用你的 main() 函数
   ↓

10. 程序运行
    现在你的代码才真正开始执行
```

### 时间线总结

```
时刻 0ms:  用户执行 ./myapp
时刻 1ms:  内核加载 ELF，启动 ld.so
时刻 2ms:  ld.so 加载 libc.so, libpthread.so 等
时刻 3ms:  ld.so 解析符号，填写 GOT 表
时刻 4ms:  ld.so 调用库的初始化函数
时刻 5ms:  ld.so 跳转到 main()
时刻 5ms+: 你的程序代码开始运行
```

## 实际例子

### 1. 查看程序依赖什么动态链接器

```bash
$ readelf -l /bin/ls | grep interpreter
      [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
```

这告诉内核："启动这个程序时，请先运行 `/lib64/ld-linux-x86-64.so.2`"

### 2. 查看程序依赖什么库

```bash
$ ldd /bin/ls
    linux-vdso.so.1 (0x00007ffd8e9f0000)
    libselinux.so.1 => /lib/x86_64-linux-gnu/libselinux.so.1 (0x00007f8e2a400000)
    libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f8e2a200000)
    /lib64/ld-linux-x86-64.so.2 (0x00007f8e2a600000)
    ↑
    这就是动态链接器本身
```

注意：
- `=>` 左边是程序需要的库名
- `=>` 右边是实际找到的库路径
- 最后一行是动态链接器本身

### 3. 手动运行 ld.so

你可以直接运行 `ld.so` 来启动程序：

```bash
# 直接用 ld.so 启动程序
/lib64/ld-linux-x86-64.so.2 /bin/ls

# 指定库搜索路径
/lib64/ld-linux-x86-64.so.2 --library-path /custom/lib /bin/ls

# 查看 ld.so 的帮助
/lib64/ld-linux-x86-64.so.2 --help
```

### 4. 调试 ld.so 的工作过程

使用 `LD_DEBUG` 环境变量查看详细过程：

```bash
# 查看库搜索过程
LD_DEBUG=libs /bin/ls

# 查看符号绑定过程
LD_DEBUG=bindings /bin/ls

# 查看所有调试信息
LD_DEBUG=all /bin/ls 2>&1 | less

# 查看可用的调试选项
LD_DEBUG=help /bin/ls
```

#### LD_DEBUG 示例输出

```bash
$ LD_DEBUG=libs /bin/ls 2>&1 | head -20
      1234:     find library=libselinux.so.1 [0]; searching
      1234:      search cache=/etc/ld.so.cache
      1234:       trying file=/lib/x86_64-linux-gnu/libselinux.so.1
      1234:
      1234:     find library=libc.so.6 [0]; searching
      1234:      search cache=/etc/ld.so.cache
      1234:       trying file=/lib/x86_64-linux-gnu/libc.so.6
      1234:
      1234:     calling init: /lib/x86_64-linux-gnu/libc.so.6
      1234:
      1234:     transferring control: /bin/ls
```

可以清楚看到 ld.so 的搜索和加载过程。

### 5. 修改程序的动态链接器

```bash
# 查看当前的解释器
patchelf --print-interpreter myapp

# 修改解释器
patchelf --set-interpreter /my/custom/ld.so myapp

# 验证修改
readelf -l myapp | grep interpreter
```

## 关键概念

### 1. GOT (Global Offset Table) - 全局偏移表

程序中对外部函数的调用通过 GOT 间接跳转：

```c
// 你的代码
printf("Hello\n");

// 实际生成的汇编（简化）
call *GOT[printf]  // 从 GOT 表查地址

// ld.so 的工作
// 启动时填写: GOT[printf] = 0x7f8e2a234560 (libc 中 printf 的地址)
```

**为什么需要 GOT？**
- 代码段是只读的，不能修改
- GOT 在数据段，可以修改
- 位置无关代码 (PIC) 的基础

### 2. PLT (Procedure Linkage Table) - 过程链接表

延迟绑定（lazy binding）- 只在第一次调用时解析符号：

```
第一次调用 printf:
  → PLT[printf] (stub 代码)
  → 跳转到 ld.so 的解析函数
  → ld.so 查找 printf 的真实地址
  → 更新 GOT[printf] = printf 的真实地址
  → 跳转到 printf

后续调用 printf:
  → PLT[printf] (stub 代码)
  → 从 GOT[printf] 读取地址（已解析）
  → 直接跳转到 printf (快!)
```

**优势：**
- 只解析实际使用的符号
- 减少启动时间
- 大部分程序不会调用所有导入的函数

**禁用延迟绑定：**
```bash
# 启动时立即解析所有符号
LD_BIND_NOW=1 ./myapp

# 或编译时指定
gcc -Wl,-z,now -o myapp myapp.c
```

### 3. RPATH vs RUNPATH

编译时可以指定库搜索路径：

```bash
# RPATH (老式，优先级高，不可被 LD_LIBRARY_PATH 覆盖)
gcc -Wl,-rpath,/my/lib myapp.c

# RUNPATH (新式，可被 LD_LIBRARY_PATH 覆盖)
gcc -Wl,--enable-new-dtags,-rpath,/my/lib myapp.c

# 查看
readelf -d myapp | grep PATH
objdump -p myapp | grep PATH
```

**搜索优先级：**
```
1. RPATH (如果没有 RUNPATH)
2. LD_LIBRARY_PATH 环境变量
3. RUNPATH
4. /etc/ld.so.cache
5. /lib, /usr/lib
```

**推荐实践：**
- 对于包管理器：使用 RPATH 确保找到正确的库
- 对于系统软件：使用默认路径
- 避免使用 LD_LIBRARY_PATH（容易污染环境）

### 4. ld.so.cache - 库缓存

系统的库缓存文件，加速搜索：

```bash
# 缓存文件位置
/etc/ld.so.cache

# 配置文件
/etc/ld.so.conf
/etc/ld.so.conf.d/*.conf

# 更新缓存
sudo ldconfig

# 查看缓存内容
ldconfig -p | grep libc

# 查看配置
cat /etc/ld.so.conf
```

**工作原理：**
- ldconfig 扫描配置的目录
- 建立库名 → 路径的映射
- 保存为二进制文件（快速查找）
- ld.so 优先查这个缓存

### 5. 符号版本控制

glibc 支持同一个符号的多个版本：

```bash
# 查看符号版本
readelf -s /lib/x86_64-linux-gnu/libc.so.6 | grep GLIBC

# 示例输出：
# memcpy@@GLIBC_2.14
# memcpy@GLIBC_2.2.5
```

**作用：**
- 向后兼容
- 旧程序用旧版本符号
- 新程序用新版本符号
- 同一个库支持多个 ABI

## 为什么 ld.so 和 libc 强耦合

`ld.so` 不仅加载 `libc.so`，它们还深度协作：

### 启动流程的协作

```c
// ld.so 加载所有库后，调用 libc 的入口
libc.so 中的 __libc_start_main():
  1. 设置 TLS (Thread Local Storage - 线程本地存储)
  2. 调用 ld.so 注册的初始化函数
  3. 设置 environ 环境变量
  4. 注册 atexit 处理函数
  5. 调用用户的 main(argc, argv, envp)
  6. 收集 main() 的返回值
  7. 调用 exit(return_value)

// ld.so 需要知道 libc 的内部结构
// libc 需要知道 ld.so 加载了哪些库
// 它们共享很多私有符号和数据结构
```

### 共享的内部接口

```
ld.so 和 libc.so 共享：
- TLS 实现细节
- 线程创建和管理
- 内存分配器的初始化
- 信号处理的设置
- dlopen/dlsym 的实现
- 异常处理机制 (C++)
```

### 版本强绑定

```bash
# glibc 2.35 的 ld.so
$ /lib64/ld-linux-x86-64.so.2 --version
ld.so (Ubuntu GLIBC 2.35-0ubuntu3) stable release version 2.35.
                    ↑
              这是 glibc 的一部分

# 它的内部依赖（私有符号）
$ nm -D /lib64/ld-linux-x86-64.so.2 | grep GLIBC_PRIVATE
00000000000xxxxx T _dl_allocate_tls@@GLIBC_PRIVATE
00000000000xxxxx T _dl_deallocate_tls@@GLIBC_PRIVATE
...
```

**关键点：**
- `GLIBC_PRIVATE` 符号不保证跨版本兼容
- ld.so 和 libc.so 必须是同一个 glibc 构建
- 混用不同版本会导致符号查找失败或崩溃

### 混用版本的后果

```bash
# 系统 glibc 2.35，自己编译 glibc 2.38
# 错误示例：用系统 ld.so 加载自编译 libc

/lib64/ld-linux-x86-64.so.2 (2.35)
  ↓ 尝试加载
/my/lib/libc.so.6 (2.38)
  ↓ 查找私有符号
💥 错误:
  - symbol lookup error: undefined symbol: __libc_init_first, version GLIBC_PRIVATE
  - 或直接 segmentation fault
```

这就是为什么不能混用不同版本的 `ld.so` 和 `libc.so`。

## 不同 libc 的 ld.so

### glibc 的 ld.so

```bash
# 常见路径
/lib64/ld-linux-x86-64.so.2          # x86_64
/lib/ld-linux-armhf.so.3             # ARM 32-bit
/lib/ld-linux-aarch64.so.1           # ARM 64-bit
/lib/ld-linux-riscv64-lp64d.so.1     # RISC-V 64-bit

# 查看版本
/lib64/ld-linux-x86-64.so.2 --version
```

**特点：**
- 功能最全
- 支持复杂的符号版本控制
- 支持 TLS 模型
- 体积较大
- 启动时间较长（但针对常见情况优化）

### musl 的 ld.so

```bash
# 常见路径
/lib/ld-musl-x86_64.so.1
/lib/ld-musl-aarch64.so.1
/lib/ld-musl-armhf.so.1

# 查看版本
/lib/ld-musl-x86_64.so.1 --version
```

**特点：**
- 简洁高效
- 代码更简单（~1000 行 vs glibc 几万行）
- 启动更快
- 没有 lazy binding（总是立即绑定）
- 不支持 GNU 扩展

### 它们完全不兼容

```bash
# ❌ 这会失败 - glibc 的 ld.so 加载 musl 程序
/lib64/ld-linux-x86-64.so.2 ./musl-program
# Error: symbol lookup error

# ❌ 这也会失败 - musl 的 ld.so 加载 glibc 程序
/lib/ld-musl-x86_64.so.1 ./glibc-program
# Error: symbol version not found

# ✅ 正确用法
/lib64/ld-linux-x86-64.so.2 ./glibc-program
/lib/ld-musl-x86_64.so.1 ./musl-program
```

### 对比表

| 特性 | glibc ld.so | musl ld.so |
|------|-------------|------------|
| 代码量 | ~30,000 行 | ~1,000 行 |
| 功能 | 完整 | 简化 |
| 符号版本控制 | ✅ | ❌ |
| Lazy binding | ✅ (默认) | ❌ (总是立即) |
| GNU 扩展 | ✅ | ❌ |
| 启动速度 | 中等 | 快 |
| 内存占用 | 较大 | 小 |
| 适用场景 | 桌面/服务器 | 嵌入式/容器 |

## 静态链接程序不需要 ld.so

### 静态编译

```bash
# 静态链接 glibc
gcc -static -o myapp myapp.c

# 静态链接 musl
musl-gcc -static -o myapp myapp.c

# 查看
$ readelf -l myapp | grep interpreter
(没有输出 - 没有 interpreter)

$ file myapp
myapp: ELF 64-bit LSB executable, statically linked

$ ldd myapp
    not a dynamic executable
```

### 启动流程

```
静态链接程序:
  用户执行 ./myapp
  ↓
  内核读取 ELF 头
  ↓
  没有 PT_INTERP 段
  ↓
  内核直接加载程序到内存
  ↓
  跳转到 _start 符号
  ↓
  _start 调用 __libc_start_main (已静态链接在程序内)
  ↓
  调用 main()
```

**优缺点：**

✅ 优点：
- 不依赖任何外部库
- 部署简单（单一文件）
- 不受系统库版本影响

❌ 缺点：
- 文件体积大（5-10MB vs 20KB）
- 浪费内存（无法共享库代码）
- 安全更新需要重新编译
- glibc 不推荐静态链接（NSS、iconv 等问题）

## 包管理器开发的影响

### 核心结论

**如果你的包管理器要自己编译 libc，必须同时提供配套的 ld.so。**

### 方案对比

| 方案 | 可行性 | 优缺点 |
|------|--------|--------|
| 用系统 ld.so + 自己的 libc (不同版本) | ❌ | 内部 ABI 不兼容，崩溃 |
| 用系统 ld.so + 自己的 libc (相同版本) | ⚠️ | 理论可行但失去隔离意义 |
| 用系统 ld.so + musl libc | ❌ | 完全不兼容 |
| **自己编译 libc + 配套 ld.so** | ✅ | **正确做法，完全隔离** |
| 静态链接，不用 ld.so | ✅ | 可行但文件大 |

### 推荐实现：Nix 模式

#### 目录结构

```
/your-pkg-manager/store/
├── abc123-glibc-2.38/
│   ├── lib/
│   │   ├── libc.so.6               # 主库
│   │   ├── libpthread.so.0         # 其他库
│   │   ├── libm.so.6
│   │   └── ld-linux-x86-64.so.2    # ← 配套的 ld.so
│   └── include/
│
├── def456-musl-1.2.4/
│   └── lib/
│       ├── libc.so
│       └── ld-musl-x86_64.so.1     # ← musl 的 ld.so
│
├── ghi789-python-3.11/
│   ├── bin/
│   │   └── python3                 # ← 已 patch，指向正确的 ld.so
│   └── lib/
│       └── libpython3.11.so
```

#### 编译包的流程

```bash
# 1. 确定使用哪个 libc
LIBC_STORE=/your-pkg/store/abc123-glibc-2.38

# 2. 编译时指定 ld.so 和 rpath
gcc \
  -Wl,--dynamic-linker=$LIBC_STORE/lib/ld-linux-x86-64.so.2 \
  -Wl,-rpath,$LIBC_STORE/lib \
  -o myapp myapp.c

# 3. 或者编译后 patch
gcc -o myapp myapp.c
patchelf \
  --set-interpreter $LIBC_STORE/lib/ld-linux-x86-64.so.2 \
  --set-rpath $LIBC_STORE/lib \
  myapp

# 4. 验证
readelf -l myapp | grep interpreter
readelf -d myapp | grep RPATH
ldd myapp
```

#### 运行时验证

```bash
$ readelf -l python3 | grep interpreter
  [Requesting program interpreter: /your-pkg/store/abc123-glibc-2.38/lib/ld-linux-x86-64.so.2]

$ ldd python3
    libc.so.6 => /your-pkg/store/abc123-glibc-2.38/lib/libc.so.6 (0x...)
    /your-pkg/store/abc123-glibc-2.38/lib/ld-linux-x86-64.so.2 (0x...)

# ✅ 完全隔离，不依赖系统库
```

### 简化方案

如果实现 Nix 模式太复杂，可以考虑：

#### 方案 A: 全局单一 libc

```bash
# 所有包共享一个版本的 libc + ld.so
/your-pkg/lib/
├── libc.so.6
├── ld-linux-x86-64.so.2
└── ...

# 所有包都 patch 到这个路径
patchelf --set-interpreter /your-pkg/lib/ld-linux-x86-64.so.2 ...
```

缺点：升级 libc 影响所有包

#### 方案 B: 静态链接为主

```bash
# 用 musl 静态编译大部分包
musl-gcc -static -o myapp myapp.c

# 优点：完全独立
# 缺点：不支持插件、文件大
```

#### 方案 C: 容器化

```bash
# 每个包在独立的文件系统命名空间
# 使用 chroot 或 容器技术
# 可以使用不同的 libc
```

### 实现工具

```bash
# 查看 ELF 信息
readelf -l <binary>     # 查看段
readelf -d <binary>     # 查看动态段
objdump -p <binary>     # 查看程序头

# 修改 ELF 文件
patchelf --set-interpreter <path> <binary>
patchelf --set-rpath <path> <binary>
patchelf --print-interpreter <binary>

# 调试动态链接
LD_DEBUG=all <binary>
ldd <binary>
```

### Bootstrap 流程

要构建完全独立的工具链：

```
阶段 0: 准备
  ├─ 下载 GCC、binutils、glibc 源码
  └─ 使用系统编译器

阶段 1: 交叉编译临时工具链
  ├─ 编译 binutils (as, ld, ar...)
  ├─ 编译 GCC stage1 (只支持静态链接)
  └─ 编译 glibc

阶段 2: 重新编译工具链
  ├─ 使用 stage1 工具链
  ├─ 重新编译 GCC (完整版)
  └─ 重新编译 glibc

阶段 3: 验证和清理
  ├─ 确保工具链不依赖系统库
  └─ 现在可以用来编译其他包
```

参考：
- Linux From Scratch (LFS) 项目
- musl-cross-make 工具
- crosstool-ng 工具

## 总结

### ld.so 的核心作用

1. **启动时机**：程序启动后、main() 之前
2. **核心任务**：
   - 加载共享库 (.so 文件)
   - 解析和绑定符号（函数地址、全局变量）
   - 执行库的初始化代码
   - 将控制权交给程序

### 为什么它很重要

- 没有它，动态链接的程序无法运行
- 它和 libc 是一个整体，不能分离
- 它决定了程序的库依赖关系

### 包管理器的关键决策

✅ **必须做的：**
- 如果自己编译 libc，必须用配套的 ld.so
- 修改程序的 ELF 头指向正确的 ld.so 路径
- 设置正确的 RPATH 指向你的库目录

⚠️ **可选的：**
- 支持多个 libc 版本（复杂但灵活）
- 提供静态链接选项（简单但文件大）
- 使用容器/命名空间隔离（最彻底）

❌ **不要做的：**
- 混用不同版本的 ld.so 和 libc
- 依赖 LD_LIBRARY_PATH（容易污染环境）
- 假设所有软件都能静态编译（glibc 有问题）

### 学习资源

1. **手册页**：`man ld.so`
2. **ELF 规范**：System V ABI 文档
3. **源码**：
   - glibc: `elf/rtld.c`, `elf/dl-load.c`
   - musl: `ldso/dynlink.c`
4. **项目**：
   - Nix: `nixpkgs/pkgs/development/libraries/glibc/`
   - Linux From Scratch: https://www.linuxfromscratch.org/
5. **工具**：
   - patchelf: https://github.com/NixOS/patchelf
   - musl-cross-make: https://github.com/richfelker/musl-cross-make

---

## 附录：常见问题

### Q: 为什么不能用系统的 ld.so？
A: 因为 ld.so 和 libc 使用私有的内部接口，不同版本或实现（glibc vs musl）不兼容。

### Q: 静态链接是不是最简单的方案？
A: 对单一程序是的，但对包管理器：
- 文件体积大
- 无法使用插件
- glibc 静态链接有已知问题（NSS、locale、iconv）

### Q: 可以同时支持 glibc 和 musl 吗？
A: 可以，Nix 就这样做。每个包指定它需要的 libc，通过 hash 隔离不同版本。

### Q: RPATH 和 LD_LIBRARY_PATH 哪个好？
A: RPATH 更好（不污染环境），但 LD_LIBRARY_PATH 可用于临时调试。

### Q: 如何调试库加载问题？
A: 使用 `LD_DEBUG=libs,bindings ./myapp` 查看详细过程。

### Q: patchelf 会破坏程序吗？
A: 通常不会，但要注意：
- 签名会失效
- 某些加固的程序可能检测到修改

---

**文档版本**: 1.0
**最后更新**: 2025-11-27
**适用于**: Linux 包管理器开发
