# CSAPP 第三讲：程序的肉身——汇编

## 物理原理：CPU 怎么"执行"一条指令

x86-64 的一条 `add %rdi, %rax` 在硬件层面是什么？

```
取指 (Fetch)   ─→  译码 (Decode)  ─→  执行 (Execute)  ─→  访存 (Memory)  ─→  写回 (Writeback)
  从 PC 读       把字节序列解析成      ALU 计算              读/写内存           结果写回寄存器
  下一条指令     "加这个 RADD 是那个"   寄存器值相加
                  μop
```

5 个阶段，每个阶段用独立硬件。这就是**流水线**——上一条还在写回，下一条已经在执行。
理想吞吐：每个时钟周期一条指令。

**关键事实：CPU 是数字逻辑**，不是网。它只看"哪些寄存器的哪些位是高电平"。汇编和机器码是
一对一翻译（CISC），x86 内部会把每条指令拆成 μop 然后用 RISC 风格跑。

## C 的设计选择：栈是 C 的临时工场

### 调用约定是 ABI 的一部分

x86-64 System V ABI（Linux）和 Microsoft x64 ABI（Windows）不同。但共同点：

- 前 6 个整型参数：rdi, rsi, rdx, rcx, r8, r9
- 返回值：rax
- 栈向低地址增长

为啥这样？因为**硬件不强制**。`call` 指令就干两件事：push PC+1，跳到目标地址。剩下的
"参数怎么放、谁负责清理栈"是 ABI 决定的——C 编译器遵守 ABI，但语言层面不强制。

### 函数调用是 push/pop 的连续

```asm
call foo        ; 压栈返回地址，跳到 foo
foo:
  push %rbp     ; 保存调用者的 rbp
  mov %rsp, %rbp ; 建立自己的栈帧
  sub $N, %rsp  ; 分配局部变量空间
  ; ... 函数体 ...
  leave          ; 恢复 rsp/rbp，等价于 mov %rbp, %rsp; pop %rbp
  ret            ; 弹出返回地址，跳回
```

栈帧是临时的——每个函数有自己的临时区域。这件事 1972 年不叫"栈帧"也不叫"作用域"，但 C
函数就是这么实现的。

### 字符串 API 是指针的不安全模式

```c
char buf[64];
gets(buf);                  // 读直到边界——没长度参数
strcpy(buf, user_input);    // 复制直到 \0——没长度参数
```

`gets` 1999 年被 ISO C11 弃用，2011 年删除。`strcpy` 还在用，理由是"性能"。但这两个函数不检查
目标长度——**因为 C 把"长度"的责任推给程序员**。字符串是 `char*` 加 `\0` 终止，没有长度。

这个设计在 1972 年是合理的（节省每一个字节），但时代变了：

- 现代语言字符串都有"长度 + 数据"：Rust 的 `&str`、Go 的 `string`、Zig 的 `[]const u8`
- `?:` 默认 `String` 类型是有显式长度，要查长度是 O(1)
- 写操作会检查 `buf.len >= src.len`，越界就 panic

### 缓冲区溢出：栈溢出的安全灾难

```text
高地址
  [返回地址]      ← call 写入
  [saved rbp]
  [buf[64]]
  [栈底]
低地址
```

如果往 buf 写超过 64 字节，会覆盖 saved rbp 和返回地址。攻击者覆盖返回地址指向自己的代码，
CPU 就跳到攻击者代码执行。

`ASLR` 让地址不可预测，`NX/DEP` 让栈不可执行，`stack canary` 检测溢出，`CFI` 限制跳转——
这些都是**事后补丁**，针对汇编层执行流的保护。

## 现代视角：让"忘记检查"成为编译期错误

### Slice 替代 `Char*`：长度是类型的一部分

```rust
fn safe_func(buf: &mut [u8]) {
    let stdin = std::io::stdin();
    let mut input = String::new();
    stdin.read_line(&mut input).unwrap();
    buf.copy_from_slice(input.as_bytes());  // 编译期检查 buf.len >= input.len()
}
```

`&mut [u8]` 知道自己的长度。`copy_from_slice` 是泛型函数，**编译器能检查两边长度**。这件事，
你能写出来，编译器能检查，不是更好？

### 边界检查默认开，unsafe 是显式选择

```rust
fn get_byte(buf: &[u8], index: usize) -> u8 {
    buf[index]  // 运行时检查：index < buf.len 才返回值，否则 panic
}
```

Rust 的索引是 OZ 而不是裸指针。Zig 的 slice `[]u8` 默认也有运行时边界检查。要绕过边界检查
必须写 `unsafe` 或 `[*]u8`——"unsafe 是显式选择"和"C 默认不安全"是相反的默认。

不是 Rust/C 不允许你做 unsafe——而是**默认安全的代价是性能差一点点，unsafe 显式 opt-in
是阅读时一眼看出哪里危险**。

### Buffer overflow 的现代解药

安全不是靠补丁堆出来的，是靠语言**让越界成为编译期错误**。Rust 的 borrow checker、Zig 的
slice 边界检查，都在**源头**减少攻击面。`canary`、`ASLR`、`NX` 是后续的 belt-and-suspenders。

## AI 时代怎么验证

**1. 让 AI 反汇编你写的 C**

```bash
$ clang -O0 -S t.c         # 看 -O0 汇编
$ clang -O3 -S t.c         # 看 -O3 汇编
$ objdump -d t.o            # 反汇编目标文件
```

让 LLM 解释每条汇编指令。重点确认：你写的 `for` 循环对应什么汇编？`printf` 调用对应什么？
`-O3` 把 `printf("hello\n")` 优化成什么了？

**2. 让 AI 找缓冲区溢出的 candidate input**

给 LLM 一个有 `char buf[64]; gets(buf);` 的函数。让它构造一个 input，能覆盖 saved rbp 和返回
地址。验证思路：64 + 8（rbp）+ 8（返回地址）= 80 字节。

**3. 让 AI 解释 stack canary 的工作原理**

问：canary 在哪生成？放在栈的哪个位置？函数返回前检查什么？为什么 canary 不放返回地址前面
8 字节而是随机值？让 LLM 描述 `__stack_chk_fail` 触发后的 stack unwind。

**4. 让 AI 对比 Rust/Zig 和 C 的栈布局差异**

让 LLM 描述：一个简单 `unsafe` 块里的 Rust 函数生成的汇编，长什么样？内层 unsafe 块的栈
保护是怎样的？

## 本讲要点

1. CPU 是 5 阶段流水线，理想吞吐是每周期一条指令
2. x86 调用约定是 ABI 不是硬件强制；C 遵守 ABI 但语言不强制
3. C 的字符串 API 是 `char*` + `\0`，长度要程序员记
5. 现代 slice 是 "长度+数据"，边界检查默认开，unsafe 是显式 opt-in
6. 缓冲区溢出的解药是语言层默认安全，canary/ASLR 是事后补丁

## 下一讲

汇编告诉你"程序变成了什么"。下一讲看编译器和处理器怎么让它跑得更快——以及为什么 C 的
undefined behavior 给了编译器"合法伤害"你的权力。