# CSAPP 系统心智模型——AI 时代重读

CSAPP 的内容是物理现实：补码来自加法器电路、缓存来自 SRAM/DRAM 速度差、虚存来自有限物理内存。
不会变。

会变的是载体——语言、调试器、写代码的人。1978 年你必须用 C 直读硬件，今天你可以让
LLM 帮你反汇编、跑 perf、用 Rust 重写关键路径。**原理不变，工具升级**。本项目按照这个思路
重新组织 CSAPP：用现代工具的视角重读它，扔掉 AI 已经能替你做的脏活，专注那些必须靠人理解的
心智模型。

## 课程讲义（12 讲，对应原书 12 章）

每讲按统一结构展开：**物理原理 → C 的设计选择 → 现代视角 → AI 时代怎么验证**。

1. [计算机系统漫游](lect01.md) — 全景图、硬件组成、操作系统抽象
2. [数据即比特，但比特不是数据](lect02.md) — 补码、浮点、隐式转换的陷阱
3. [程序的肉身：汇编](lect03.md) — 寄存器、栈、缓冲区溢出的根源
4. [处理器体系结构](lect04.md) — 流水线、分支预测、投机执行
5. [优化程序性能](lect05.md) — 编译器优化、UB 与确定性
6. [内存层级——速度的代价](lect06.md) — 缓存行、局部性、隐藏的内存分配
7. [链接——程序的拼装](lect07.md) — 头文件的混乱、模块系统的来由
8. [异常与进程——操作系统的入口](lect08.md) — fork、信号、错误处理的演变
9. [虚拟内存——一切的基石](lect09.md) — 页表、mmap、内存管理的噩梦
10. [系统级 I/O](lect10.md) — 文件描述符、缓冲 I/O、显式与隐式
11. [网络编程](lect11.md) — socket API、HTTP、抽象的层次
12. [并发编程](lect12.md) — 线程、竞态条件、死锁、资源清理

## 设计原则

- **原理是物理的**：补码来自加法器、缓存来自速度差、虚存来自有限物理内存。这是不可改变的。
- **C 是观察对象，不是靶子**：C 是 OS/ABI 的事实标准。你读它不是要学它写它，是因为你接触的所有
  系统层都讲 C 的语言。
- **现代工具暴露 C 隐藏的东西**：Rust/Zig/whatever 让 UB、隐式分配、生命周期这些成为编译期错误，
  不是为了替换 C，是为了让你看清 C 哪里在"悄悄做事"。
- **AI 干脏活，你定方向**：反汇编、跑 trace、生成测试用例、读 stack——这些都丢给 LLM。你做的是
  判断它说的对不对、看它漏了什么 case。
- **心智模型 > 手写代码**：理解系统怎么工作，而不是手写每一行。原 CSAPP 的 hack 实验
  （Data Lab 位运算、Bomb Lab 反汇编、Malloc Lab 手写分配器）大部分被 AI 取代了。

## 为什么还看 C？

OS 内核、ABI、系统调用、glibc、Linux 工具链——它们都是 C。你不会写 C，但你要读它。
汇编反编译回来是 C-风格伪代码。LLM 解释系统行为给的例子是 C。所以 C 是这里的"通用语"。
不是因为它好，是因为它无处不在。

## 对比总览

| 主题 | 物理原理 | C 的设计 | 现代视角 | AI 验证 |
|------|----------|----------|----------|---------|
| 数据表示 | 比特无意义，解释有 | 隐式转换、UB | 显式 cast、well-defined ops | 让 AI 解释 `0xFFFFFFFF` 的多种解释 |
| 缓冲区溢出 | 数组无边界就在运行时栈溢 | gets/strcpy 不检查 | slice 带长度、边界检查 | 让 AI 找候选 input 触发 panic |
| 编译器优化 | UB 给优化空间 | `int *restrict`、UB-as-feature | 没有 UB、确定性 ops | 让 AI 对比 `-O0`/`-O3` 汇编差异 |
| 内存管理 | 物理内存有限、要回收 | malloc/free 手动 | defer、Allocator 注入 | 让 AI 找 use-after-free、leak |
| 错误处理 | 错误必然发生 | errno 全局 | 类型系统承载错误 | 让 AI 找未检查的返回值 |
| I/O 缓冲 | 系统调用贵、要省事 | FILE\* 隐藏缓冲 | 显式 bufferedWriter | 让 AI 跑 strace 观察 syscall |
| 链接 | 多个 .o 要拼一个 binary | #include 文本复制 | 模块系统、comptime 引用 | 让 AI 解释 `readelf -a` 输出 |
| 进程/信号 | 隔离 + 异步事件 | fork/exec/signal | 同步原语、async runtime | 让 AI 追踪 SIGCHLD 时序 |
| 虚存 | 物理内存有限、要隔离 | mmap/malloc 手管理 | arena、RAII、GC | 让 AI 分析 page fault trace |
| 网络 | 一切皆文件、socket=fd | sockaddr 强转、大小端忘 | 类型安全、字节序自动 | 让 AI 写 HTTP server 看 syscall |

## 实验（核心 insight + AI 验证路径，不手写代码）

原 CSAPP 实验的"手写反汇编"、"手写位运算"、"手写 malloc"在 AI 时代价值不大。
每个实验重写成：**核心 insight + 用 AI 怎么验证它 + 你应该能回答的判断题**。

| 实验 | 对应章节 | 核心 insight | AI 帮你做什么 |
|------|----------|-------------|---------------|
| [Data Lab](labs/data_lab.md) | Ch2 | 位运算是一套完备计算基础 | 解释每个位 hack 的数学 |
| [Bomb Lab](labs/bomb_lab.md) | Ch3 | 汇编有规律，规律可推 | 反汇编 + 解释每条指令 |
| [Attack Lab](labs/attack_lab.md) | Ch3 | 控制流可被篡改 | 生成 payload、解释 gadget 链 |
| [Cache Lab](labs/cache_lab.md) | Ch6 | 缓存行为是确定性的 | 跑 trace、模拟 cache hit/miss |
| [Shell Lab](labs/shell_lab.md) | Ch8 | Shell 就是 fork + exec 循环 | 解释 fork/exec 时序、strace shell |
| [Malloc Lab](labs/malloc_lab.md) | Ch9 | 分配器是堆上数据结构 | 分析 glibc malloc 源码 |
| [Proxy Lab](labs/proxy_lab.md) | Ch10-12 | 代理是 I/O+网络+并发串 | 生成代理代码、解释 select/epoll |

## 使用方式

每讲约 10-15 分钟阅读。讲义是入口，AI 是工具。读完一章，带着讲义里的判断题去找 LLM 验证。

需要的前置知识：基础编程能力。任何系统语言（C/Rust/Go/Zig）经验。