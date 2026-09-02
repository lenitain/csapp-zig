# Proxy Lab

**对应章节：** 第十、十一、十二讲（I/O、网络编程、并发）

**一句话点题：** Web 代理就是"中间人"——接收客户端请求，转发给服务器，把响应返回给客户端。

## 为什么要做这个实验

CSAPP 的 Proxy Lab 让你实现一个支持 HTTP/1.0 的 Web 代理服务器，需要处理并发连接和缓存。

这个实验把前面学到的三章内容串在一起：
- **系统级 I/O**：读写 socket
- **网络编程**：解析 HTTP 协议
- **并发编程**：处理多个客户端同时请求

## 最简单的代理

```c
int main(int argc, char **argv) {
    int listenfd = Open_listenfd(argv[1]);  // 监听端口

    while (1) {
        struct sockaddr_in clientaddr;
        socklen_t clientlen = sizeof(clientaddr);
        int connfd = Accept(listenfd, (SA *)&clientaddr, &clientlen);

        doit(connfd);   // 处理请求
        Close(connfd);
    }
}

void doit(int fd) {
    char buf[MAXLINE], method[MAXLINE], uri[MAXLINE], version[MAXLINE];
    char hostname[MAXLINE], pathname[MAXLINE], port[MAXLINE];

    Rio_readlineb(fd, buf, MAXLINE);
    sscanf(buf, "%s %s %s", method, uri, version);

    parse_uri(uri, hostname, pathname, port);  // 解析 URI

    // 连接服务器
    int serverfd = Open_clientfd(hostname, port);

    // 转发请求
    sprintf(buf, "GET %s HTTP/1.0\r\n", pathname);
    Rio_writen(serverfd, buf, strlen(buf));
    Rio_writen(serverfd, "\r\n", 2);

    // 转发响应
    size_t n;
    while ((n = Rio_readn(serverfd, buf, MAXLINE)) > 0) {
        Rio_writen(fd, buf, n);
    }

    Close(serverfd);
}
```

### 思路

1. 监听端口，等待客户端连接
2. 读取客户端的 HTTP 请求
3. 解析 URI，提取主机名、路径、端口
4. 连接目标服务器
5. 转发请求给服务器
6. 转发响应给客户端

就这么简单。**代理就是一个"中间人"，不修改请求和响应，只做转发。**

## 并发处理

单线程代理一次只能处理一个请求。如果一个请求需要 1 秒，10 个客户端就要等 10 秒。

### 方案一：多进程

```c
while (1) {
    int connfd = Accept(listenfd, ...);
    if (Fork() == 0) {
        Close(listenfd);
        doit(connfd);
        Close(connfd);
        exit(0);
    }
    Close(connfd);
    // 父进程不等待，继续接受新连接
}
```

### 方案二：多线程

```c
while (1) {
    int connfd = Accept(listenfd, ...);
    Pthread_create(&tid, NULL, thread, &connfd);
}

void *thread(void *vargp) {
    int connfd = *((int *)vargp);
    Pthread_detach(pthread_self());
    Free(vargp);
    doit(connfd);
    Close(connfd);
    return NULL;
}
```

### 思路

多进程简单但开销大（fork 复制地址空间）。多线程轻量但需要注意线程安全。选择哪个取决于场景。

## 缓存

如果多个客户端请求同一个资源，代理可以缓存响应，避免重复请求服务器。

```c
// 简单的缓存结构
struct cache_entry {
    char uri[MAXLINE];      // 缓存的 URI
    char *response;         // 缓存的响应
    size_t size;            // 响应大小
    struct cache_entry *next;
};

// 缓存查找
char *cache_find(char *uri) {
    for (struct cache_entry *e = head; e; e = e->next) {
        if (!strcmp(e->uri, uri)) {
            return e->response;  // 命中
        }
    }
    return NULL;  // 未命中
}
```

### 思路

缓存的本质是"用空间换时间"。缓存命中时，省去了网络往返。但缓存有大小限制，需要淘汰策略（LRU、LFU）。

并发环境下，缓存需要加锁保护：

```c
// 读写锁：多个读者可以并发，写者独占
pthread_rwlock_t cache_lock = PTHREAD_RWLOCK_INITIALIZER;

char *cache_find(char *uri) {
    pthread_rwlock_rdlock(&cache_lock);  // 读锁
    char *result = lookup(uri);
    pthread_rwlock_unlock(&cache_lock);
    return result;
}

void cache_insert(char *uri, char *response) {
    pthread_rwlock_wrlock(&cache_lock);  // 写锁
    insert(uri, response);
    pthread_rwlock_unlock(&cache_lock);
}
```

## 本实验的 takeaway

1. Web 代理就是"中间人"：接收请求，转发给服务器，返回响应
2. HTTP 协议是文本的，可以用 `sscanf` 解析
3. 并发处理用多进程或多线程，各有 trade-off
4. 缓存是"用空间换时间"，需要淘汰策略和并发保护

**核心 insight：代理把前面三章的内容串在一起了——I/O（读写 socket）、网络（HTTP 协议）、并发（多线程处理）。理解代理，你就理解了一个完整的网络应用是怎么工作的。**
