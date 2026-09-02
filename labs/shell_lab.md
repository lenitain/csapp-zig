# Shell Lab

**对应章节：** 第八讲（异常控制流）

**一句话点题：** Shell 就是一个循环：读命令 → fork → exec → 等待。

## 为什么要做这个实验

CSAPP 的 Shell Lab 让你实现一个简单的 Unix shell，支持前后台进程、信号处理、作业控制。

这个实验的**核心 insight** 是：

**Shell 不是什么神秘的东西，它就是一个程序。** 读取用户输入，解析命令，fork
一个子进程，exec 执行程序，等待子进程结束。就这么简单。

## 最简单的 shell

```c
int main() {
    char cmdline[MAXLINE];
    while (1) {
        printf("> ");
        Fgets(cmdline, MAXLINE, stdin);
        if (feof(stdin)) exit(0);

        eval(cmdline);
    }
}

void eval(char *cmdline) {
    char *argv[MAXARGS];
    int bg = parseline(cmdline, argv);  // 解析命令，bg=1 表示后台

    if (argv[0] == NULL) return;  // 空命令

    pid_t pid = Fork();
    if (pid == 0) {  // 子进程
        Execve(argv[0], argv, environ);
    }

    if (!bg) {  // 前台进程，等待
        int status;
        Waitpid(pid, &status, 0);
    } else {  // 后台进程，不等待
        printf("%d %s", pid, cmdline);
    }
}
```

### 思路

1. `parseline`：把用户输入拆分成命令和参数，判断是否后台运行（`&` 结尾）
2. `Fork`：创建子进程
3. `Execve`：子进程执行命令
4. `Waitpid`：前台进程等待子进程结束

这就是 shell 的核心循环。

## 信号处理：Ctrl-C 怎么工作？

用户按 Ctrl-C，内核给前台进程组发 SIGINT。默认行为是终止进程。但 shell 需要特殊处理：

```c
// 父进程（shell）忽略 SIGINT
Signal(SIGINT, SIG_IGN);
// 子进程继承了这个设置，需要恢复为默认
```

### 思路

Shell 本身不应该被 Ctrl-C 终止——它应该终止前台子进程，然后继续等待输入。所以 shell 忽略
SIGINT，让子进程自己处理。

## 作业控制：前后台切换

```c
// bg %1：把作业 1 放到后台继续运行
// fg %1：把作业 1 放到前台
// jobs：列出所有作业

void builtin_cmd(char **argv) {
    if (!strcmp(argv[0], "quit")) exit(0);
    if (!strcmp(argv[0], "jobs")) list_jobs();
    if (!strcmp(argv[0], "bg")) do_bg(argv);
    if (!strcmp(argv[0], "fg")) do_fg(argv);
}
```

### 思路

Shell 维护一个作业列表，记录每个作业的 PID、状态（运行中/停止/完成）。`SIGCHLD` 信号通知
shell 子进程状态变化，shell 在信号处理函数中更新作业列表。

## Zig 实现的差异

```zig
const posix = std.posix;

fn eval(cmdline: []const u8) void {
    const argv = parseLine(cmdline);
    const bg = isBackground(cmdline);

    const pid = posix.fork() catch return;
    if (pid == 0) {
        // 子进程
        posix.execveZ(argv[0], argv, environ) catch {
            std.debug.print("Command not found\n", .{});
            posix.exit(1);
        };
    }

    if (!bg) {
        _ = posix.waitpid(pid, 0);
    }
}
```

Zig 的 `posix.fork()` 和 `posix.execveZ()` 直接暴露系统调用，没有 C 的 `fork()`/`exec()`
的包装。你看到的就是你得到的。

## 本实验的 takeaway

1. Shell 的核心是 fork + exec——创建子进程，执行程序
2. 信号是异步的，信号处理函数需要小心设计
3. 作业控制维护一个作业列表，用 SIGCHLD 跟踪子进程状态
4. 理解 shell，你就理解了进程管理、信号、I/O 重定向的全部

**核心 insight：Shell 不是魔法，它就是一个普通的程序，用操作系统提供的
API（fork、exec、wait、signal）实现了进程管理。**
