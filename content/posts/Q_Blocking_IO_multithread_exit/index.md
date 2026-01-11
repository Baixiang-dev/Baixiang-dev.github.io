---
title: "使用阻塞I/O的多线程服务退出问题"
date: 2026-01-11
draft: false
tags:
    - debug
math: true
---
今天在实现一个使用阻塞I/O通信的多线程服务时，遇到一个奇怪的问题: 直接运行程序时，使用 `Ctrl+C` 停止程序，程序输出退出提示但是没有正常停止，而使用 gdb 附加调试时，`Ctrl+C` 可以正常退出。

复现此问题的最小 demo 如下：
```cpp
#include <arpa/inet.h>
#include <atomic>
#include <csignal>
#include <cstring>
#include <iostream>
#include <thread>
#include <unistd.h>

int               listen_fd;
std::atomic<bool> quit{false};

void sigint_handler(int)
{
 std::cout << "[main] SIGINT received\n";
    quit = true;
}

void accept_thread()
{
 std::cout << "[accept] calling accept()...\n";

    sockaddr_in cli{};
 socklen_t   len = sizeof(cli);

 int fd = accept(listen_fd, (sockaddr*)&cli, &len);
 if (fd < 0)
 {
 std::cout << "[accept] accept returned -1, errno=" << errno << " (" << strerror(errno)
 << ")\n";
 }
 else
 {
 std::cout << "[accept] accepted\n";
 close(fd);
 }

 std::cout << "[accept] exit\n";
}

int main()
{
 signal(SIGINT, sigint_handler);

    listen_fd = socket(AF_INET, SOCK_STREAM, 0);

    sockaddr_in addr{};
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = htonl(INADDR_LOOPBACK);
    addr.sin_port = htons(7777);

 bind(listen_fd, (sockaddr*)&addr, sizeof(addr));
 listen(listen_fd, 16);

 std::thread t(accept_thread);

 while (!quit) pause();

 std::cout << "[main] server stopped\n";

    /**
     * 1. 使用 close 关闭，不能唤醒阻塞的 accept
     * - 直接运行时 此处应该看到程序卡在 t.join() 处
     * - 使用 gdb 调试发送 SIGINT信号时，ptrace会中断accept调用，再次执行时，在已经 close 的 socket
     *  上 accept 返回错误，errno=9 (Bad file descriptor)
     */
 close(listen_fd);

    // 2. 使用 shutdown 关闭 socket，会唤醒阻塞的 accept, 此处应该看到返回的错误是 invalid argument
    // shutdown(listen_fd, SHUT_RDWR);

    t.join();   // ← 直接运行时会卡在这里
 std::cout << "[main] exit\n";
}
```

最终定位下来，造成此现象的原因有两个：

1. `close` 不能唤醒阻塞状态的系统调用: `close` 关闭socket，不能保证唤醒阻塞状态的系统调用，监听线程阻塞在 `accept` ，主线程等待监听线程，所以程序不会结束。
2. 直接运行和 GDB 调试的行为差异。直接运行时、内核向进程发送 SIGINT，只保证至少有一个线程收到信号，GDB 附加调试时，`ptrace` 会中断唤醒所有阻塞的系统调用。
## close的局限性
在 Linux 中，一个 socket fd 的结构是像下面这样的：
```txt
fd (文件描述符)
 └── struct file
      └── struct socket
           └── struct sock (协议相关，如 TCP)
```

fd 是一个整数索引，指向进程打开的文件对象 file ，当进程在调用 `close` 关闭 fd 的时候，只是减少对这个文件对象的引用计数 `file->fcount`。 只有当引用计数为 0 的时候，内核才会真正关闭这个文件对象，内核在阻塞调用（比如 `accept` ）中，会增加fd的引用计数。

在上面的 demo 中，程序开始后，主线程创建 `listen_fd`，把它实际指向的文件对象称为 listen_file，listen_file 的引用计数是 1，监听线程陷入 `accept` 时，listen_file的引用计数是 2 。

在终端输入 `Ctrl+C` 后，主线程进入退出流程，`close(fd)` 之后，listen_file 的引用计数为 1 ，此时内核不会关闭 listen_file，也不会唤醒监听线程阻塞中的 `accept`，主线程等待监听线程退出，但是监听线程阻塞，所以程序没法正常退出。

为了解决这个问题，可以使用 `shutdown` 关闭 `listen_fd`，`shutdown` 是协议语义（图示中的sock）层面的操作，`shutdown` 之后，内核会显式唤醒阻塞在该 socket 上的线程并返回错误 `EINVAL` (Invalid argument) 。

直接运行和 GDB 的行为差异
直接运行时，在终端发送SIGINT，内核只保证进程中至少有一个线程收到此信号，通常是主线程收到信号并处理，但是这不会打断其他线程阻塞中的系统调用，在 demo 中，监听线程继续 `accept`，不受影响。

GDB 基于 `ptrace` 实现，在使用 GDB 进行附加调试时发送 SIGINT 信号，`ptrace` 会捕获信号并中断所有阻塞的系统调用，listen_file 的引用计数为 1，在使用`signal SIGINT`注入信号后再继续执行时，主线程先执行 `close`，内核关闭 listen_file, 之后监听线程 `accept` 一个已经关闭的fd，从而触发 `EBADF` (Bad file descriptor) 错误并返回。
