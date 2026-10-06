# CSAPP 第七讲：链接——程序的拼装

## 物理原理：多个翻译单元要拼成一个 binary

你的程序分三个文件：`main.c`、`utils.c`、`utils.h`。编译器把它们各自翻译成 `.o`，然后
链接器把它们拼成一个可执行文件。

**为啥要拼？** 因为每个 `.c` 是独立编译的——编译器一次只能看见一个文件。**为了减少重编译
时间**——改 `utils.c` 不需要重编 `main.c`。这件事在 1972 年重要（CPU 慢、磁盘小），现在也重要
（Linux 内核编译要几分钟）。

链接器干两件事：

1. **符号解析**：每个函数/全局变量名要找到唯一定义
2. **重定位**：把每个 `.o` 的代码/数据放到最终地址，修正所有引用

**这是物理实现**：编译器一次编译一个文件，地址是相对的（从 0 开始）。链接器决定最终地址，
重定位是把相对地址改成绝对地址。

### 目标文件格式 (ELF)

Linux 用 ELF（macOS 是 Mach-O，Windows 是 PE/COFF）。ELF 文件由：

- `.text`：代码段
- `.data`：已初始化全局变量
- `.bss`：未初始化全局变量（不占文件空间，运行时分配）
- `.symtab`：符号表，函数名/全局变量名→地址
- `.rel.text`：回写表，需要链接器修正的地址

`readelf -a` 看 ELF 全部，`objdump -d` 看反汇编，`nm` 看符号表。**这些是你看 ELF 的工具**。

## C 的设计选择：头文件是文本复制

### C 的 `#include` 不是模块，是物理拷贝

```c
// utils.h
int add(int a, int b);

// main.c
#include "utils.h"
int main() { return add(1, 2); }
```

预处理后 `main.c` 变成：

```c
// 复制粘贴进来的 utils.h 内容
int add(int a, int b);
int main() { return add(1, 2); }
```

`#include` 是**物理文本复制**。这个设计在 1972 年合理——`#include` 比 LISP 性能强。现代语言
的模块系统是 ELF 解析过的元数据，不是文本。

### `static`/`extern`/`inline` 是链接可见性

```c
// 翻译单元私有
static int x = 0;

// 跨翻译单元可见（声明）
extern int y;

// 跨翻译单元（定义）
int y = 1;

// 内联（不强制）
inline int square(int x) { return x * x; }
```

C 给了你三档（private/declaration/definition）。但**没类型的可见性是字符串级别的**——`extern int y`
可以声明一个不存在的变量，链接时才报。

### 弱符号：C 的隐蔽坑

```c
// a.c
int x;  // 暂定定义

// b.c
int x;  // 另一个暂定定义
```

两个未初始化的全局变量同名。链接器随便选一个。**你以为是两个变量，其实是同一个**。这是
C 的"暂定定义 + 弱符号"规则的代价。

### 头文件包含污染

```c
// a.h
#define MAX 100

// b.h
#define MAX 200  // 重定义

// c.c
#include "a.h"
#include "b.h"  // 警告或错误
```

宏是全局文本替换，include 顺序影响结果。这是**原理缺陷**——宏没有作用域。

## 现代视角：模块系统，编译过的元数据

### Rust/Zig 的模块是真正的模块

```rust
// utils.rs
pub fn add(a: i32, b: i32) -> i32 { a + b }

// main.rs
mod utils;
use crate::utils::add;
fn main() { add(1, 2); }
```

Rust 的 `mod utils` 不是文本复制——它读取 `utils.rs` 的**编译过的元数据**（哪个函数可见、参数
类型）。这件事**不可重新定义**——`utils` 是一个命名空间。

Zig 的 `mod` 一样，公开的也是编译过的元数据。

### 显式可见性，默认私有

```rust
fn helper() { ... }  // 默认私有
pub fn api() { ... }  // 显式 pub
```

Rust/Zig 的可见性是显式 opt-in。C 的 `static int x` 是默认私有但**通过宏复杂化**。

### `comptime` 引用替代 `#include`

```zig
const utils = @import("utils.zig");
```

Zig 的 `@import` 是编译时引用——读 `utils.zig` 的元数据。**不是文本复制**。这件事**编译时
类型验证**——`utils.add` 用了不存在的函数，编译期就报错。

## AI 时代怎么验证

**1. 让 AI 解释 `readelf -a` 输出**

```bash
$ readelf -a main.o
```

让 LLM 解释：哪些 section？哪些符号？回写表里哪些地址要修正？

**2. 让 AI 解释链接错误**

```bash
$ cc main.c utils.c -o main
// undefined reference to `add`
```

让 LLM 找问题：哪个文件没定义 `add`？是函数名打错了，还是没链接相应 .c？

**3. 让 AI 解释 PIC（位置无关代码）**

```bash
$ readelf -d lib.so | grep BIND_NOW
$ objdump -d lib.so | grep -A 3 "plt>"
```

让 LLM 解释：PLT/GOT 是什么？为啥动态库需要 PIC？性能开销在哪？

**4. 让 AI 对比 Rust 和 C 的模块解析时间**

```bash
$ time cargo build --release
$ time cc -c a.c utils.c -o main.o
```

让 LLM 解释：Rust 模块系统为什么比 C include 快？元数据缓存是什么？

## 本讲要点

1. 链接器做符号解析 + 重定位——物理上要把多个 `.o` 拼成一个 binary
2. C 的 `#include` 是文本复制——不是现代意义上的模块
3. `static`/`weak` 是 C 的可见性，宏全局污染是原理缺陷
4. Rust/Zig 的模块系统是编译过的元数据，可见性显式 opt-in
5. AI 时代读 ELF 是必备技能——`readelf`/`nm`/`objdump` 是你的工具

## 下一讲

程序运行起来了，但"运行"到底是什么意思？下一讲看进程和异常控制流——操作系统怎么管理多个程序
"同时"运行，以及 C 的 errno 机制为什么是一个设计得很糟糕的错误处理方式。