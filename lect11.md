# CSAPP 第十一讲：网络编程

## 网络也是文件

Unix 的"一切皆文件"哲学延伸到了网络。网络连接通过**套接字** (socket) 表示，套接字也是一个文件描述符。你用 `read()`/`write()` 收发数据，就像读写文件一样。

这是 CSAPP 第十一章的核心 insight：**网络编程不是什么神秘的东西，它只是 I/O 的一种特殊形式。**

## 客户端-服务器模型

网络应用的基本架构是**客户端-服务器**：

- **服务器**：等待连接，提供服务
- **客户端**：主动连接服务器，请求服务

```
客户端                    服务器
   |                        |
   |--- 连接请求 ---------->|
   |<-- 接受连接 -----------|
   |--- 发送请求 ---------->|
   |<-- 返回响应 -----------|
   |--- 关闭连接 ---------->|
```

## Socket API

### 创建套接字

```c
int sockfd = socket(AF_INET, SOCK_STREAM, 0);
// AF_INET: IPv4
// SOCK_STREAM: TCP（可靠的、面向连接的）
// 0: 自动选择协议
```

`socket()` 返回一个文件描述符。之后所有的网络操作都通过这个文件描述符进行。

### 绑定地址（服务器端）

```c
struct sockaddr_in addr;
addr.sin_family = AF_INET;
addr.sin_port = htons(8080);           // 端口号
addr.sin_addr.s_addr = INADDR_ANY;     // 监听所有接口

bind(sockfd, (struct sockaddr *)&addr, sizeof(addr));
```

`bind()` 把套接字绑定到一个地址和端口。服务器必须绑定，客户端通常不需要（系统自动分配）。

### 监听连接（服务器端）

```c
listen(sockfd, 10);  // 最多 10 个等待连接
```

`listen()` 把套接字从"主动"变成"被动"，开始监听连接请求。

### 接受连接（服务器端）

```c
struct sockaddr_in client_addr;
socklen_t client_len = sizeof(client_addr);
int connfd = accept(sockfd, (struct sockaddr *)&client_addr, &client_len);
```

`accept()` 从等待队列中取出一个连接，返回一个新的文件描述符。这个新的文件描述符用于和客户端通信。

### 连接服务器（客户端）

```c
struct sockaddr_in addr;
addr.sin_family = AF_INET;
addr.sin_port = htons(8080);
inet_pton(AF_INET, "127.0.0.1", &addr.sin_addr);

connect(sockfd, (struct sockaddr *)&addr, sizeof(addr));
```

`connect()` 主动连接服务器。成功后，套接字就可以收发数据了。

## 收发数据

连接建立后，用 `read()`/`write()` 收发数据：

```c
// 客户端发送请求
write(sockfd, request, strlen(request));

// 服务器接收请求
char buf[1024];
ssize_t n = read(connfd, buf, sizeof(buf));

// 服务器发送响应
write(connfd, response, strlen(response));
```

这就是"一切皆文件"的威力：网络 I/O 和文件 I/O 用同一套 API。

## C 的 socket API 的问题

### 类型不安全

```c
struct sockaddr_in addr;
// 必须强制转换为 struct sockaddr *
bind(sockfd, (struct sockaddr *)&addr, sizeof(addr));
```

C 的 socket API 用 `struct sockaddr *` 作为通用类型，实际传入的是 `struct sockaddr_in` 或 `struct sockaddr_in6`。这种强制转换绕过了类型系统。

### 大小端转换容易忘

```c
addr.sin_port = htons(8080); // 主机字节序转网络字节序
```

网络协议用大端字节序 (big-endian)，x86 用小端字节序 (little-endian)。你必须用 `htons()`/`htonl()` 转换。如果忘了，端口号会被错误解释。

### IPv4 和 IPv6 不兼容

```c
// IPv4
struct sockaddr_in addr4;
// IPv6
struct sockaddr_in6 addr6;
// 两套不同的结构体，不同的函数
```

### Zig 的态度：更安全的抽象

```zig
const net = std.net;

// 创建 TCP 服务器
const address = try net.Address.resolveIp("127.0.0.1", 8080);
var server = try address.listen(.{
    .reuse_address = true,
});

// 接受连接
const conn = try server.accept();
const reader = conn.stream.reader();
const writer = conn.stream.writer();

// 收发数据——和文件 I/O 一样的 API
const request = try reader.readUntilDelimiterAlloc(allocator, '\n', 1024);
try writer.print("HTTP/1.1 200 OK\r\n\r\nHello\n", .{});
```

Zig 的 `std.net` 提供了类型安全的 socket API：
- `net.Address` 统一了 IPv4 和 IPv6
- 字节序转换由库处理
- 没有强制类型转换

## HTTP：最常用的应用层协议

HTTP (HyperText Transfer Protocol) 是 Web 的基础。它建立在 TCP 之上，使用简单的文本格式：

### HTTP 请求

```
GET /index.html HTTP/1.1
Host: www.example.com
Connection: close
```

### HTTP 响应

```
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 13

Hello, World!
```

HTTP 的设计体现了 Unix 哲学：**简单、文本、可组合。** 你可以用 `telnet` 或 `curl` 手动发送 HTTP 请求，用 `tcpdump` 抓包查看。

### 简单的 HTTP 服务器

```zig
const std = @import("std");

pub fn main() !void {
    const address = try std.net.Address.resolveIp("127.0.0.1", 8080);
    var server = try address.listen(.{ .reuse_address = true });
    defer server.deinit();

    while (true) {
        const conn = try server.accept();
        handleConnection(conn);
    }
}

fn handleConnection(conn: std.net.Server.Connection) void {
    defer conn.stream.close();

    var buf: [1024]u8 = undefined;
    const n = conn.stream.reader().read(&buf) catch return;
    const request = buf[0..n];

    // 解析请求行
    if (std.mem.startsWith(u8, request, "GET")) {
        const response = "HTTP/1.1 200 OK\r\nContent-Length: 13\r\n\r\nHello, World!";
        _ = conn.stream.writer().write(response) catch return;
    }
}
```

这个服务器只有 20 多行代码，但它能处理真实的 HTTP 请求。

## 网络编程的挑战

### 协议解析

网络数据是字节流，没有"消息边界"。你必须自己解析协议，处理不完整的数据。

```c
// read() 可能只返回部分数据
char buf[1024];
ssize_t n = read(fd, buf, sizeof(buf));
// buf 里可能只有 HTTP 请求的一部分
// 你必须循环读取直到收到完整的请求
```

### 并发处理

服务器需要同时处理多个客户端。选择：
- 多进程/多线程：每个连接一个进程/线程
- I/O 多路复用：一个线程处理多个连接
- 异步 I/O：事件驱动

每种方式都有 trade-off，没有最好的方案。

### 错误处理

网络是不可靠的。连接可能中断、数据可能丢失、超时可能发生。你必须处理所有这些情况。

## 本讲要点

1. 网络套接字是文件描述符，网络 I/O 和文件 I/O 用同一套 API
2. 客户端-服务器模型是网络应用的基本架构
3. C 的 socket API 类型不安全，大小端转换容易忘
4. Zig 的 std.net 提供了更安全的抽象
5. HTTP 是最常用的应用层协议，设计简单、文本、可组合

## 下一讲

网络让程序可以和远程的程序通信。但本地的程序之间也需要通信——下一讲看并发编程，以及为什么"共享内存"既是并发的基础，也是并发 bug 的根源。
