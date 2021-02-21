---
title: IPC 管道(pipe)测试
date: 2020-01-20 15:04:08
tags:
- IPC
- C/C++
categories:
- OS
---

# 基础知识梳理

## 文件共享
unix系统支持在不同进程间共享打开文件。下面两个函数都可以用来复制一个现有的文件描述符
```cpp
#include <unistd.h>
int dup(int fd);
int dup2(int fd, int fd2);
```
由dup返回的新文件描述符一定是当前可用文件描述符中的最小数值。
对于dup2，可以用fd2参数指定新描述符的值。
    - 如果fd2已经打开，则先将其关闭；
    - 如若fd等于fd2，则dup2返回fd2，而不关闭它；
    - 否则，fd2的FD_CLOEXEC文件描述符标志就被清除，这样fd2在进程调用exec时是打开状态
这些函数返回的新文件描述符与参数fd共享同一个文件表项。

## fork进程
一个现有的进程可以调用fork函数创建另一个新进程。
```c
#include <unistd.h>
pid_t fork(void);
```
由fork创建的新进程被称为子进程(child process)。fork函数被调用一次，但返回两次。两次返回的区别是子进程返回值是0，父进程的返回值是新建子进程的进程ID。因为一个父进程可以有多个子进程，并且没有一个函数使一个进程可以获得其所有子进程的ID；而子进程只有一个父进程，子进程总是可以通过调用getppid获得其父进程的进程ID。

子进程和父进程继续执行fork调用之后的指令。子进程是父进程的副本，可以获得父进程的数据空间、堆和栈的副本。父进程的所有打开文件描述符都被复制到子进程中，我们说“复制”是因为对每个文件描述符来说，就好像执行了dup函数。父进程和子进程每个相同的打开文件描述符共享一个文件表项。一般来说，在fork之后是父进程先执行还是子进程先执行是不确定的，这取决于内核所使用的调度算法。

在fork之后处理文件描述符有以下两种常见的情况：
1. 父进程等待子进程完成。在这种情况下，父进程无需对其描述符做任何处理，当子进程终止后，它曾进行读、写操作的任何共享描述符的文件偏移量已做了相应更新。
2. 父进程和子进程各自执行不同的程序段。在这种情况下，在fork之后，父进程和子进程各自关闭它们不需要使用的文件描述符，这样就不会干扰对方使用的文件描述符。这种方法是网络服务进程经常使用的。

## 函数exec
用fork函数创建新的子进程之后，子进程往往要调用一种exec函数以执行另一个程序。当进程调用一种exec函数时，该进程执行的程序完全替换为新程序，而新程序则从其main函数开始执行。因为调用exec并不创建新进程，所以前后的进程ID并未改变。exec只是用磁盘上的一个新程序替换了当前进程的正文段、数据段、堆段和栈段。
```c
#include <unistd.h>
int execv(const char *pathname, char* const argv[]);
```
用fork可以创建新进程，用exec可以初始执行新的程序；exit函数和wait函数处理终止和等待终止。这些是我们需要的基本的进程控制原语。

## 管道IPC
管道是unix系统IPC的最古老形式，所有unix系统都提供此种通信机制。管道又分为匿名管道和命名管道：
1. 匿名管道(pipe)，是一种半双工的通信方式，数据只能单向流动，而且只能在具有公共祖先的进程间使用，通常是指父子进程间。
2. 命名管道(named pipe)，又叫FIFO(First In, First Out)，通常也是半双工的通信方式，不同的是，命名管道可以支持没有亲缘关系的进程之间通信。
    - FIFO (First in, First out)为一种特殊的文件类型，它在文件系统中有对应的路径，因此可以通过文件的路径来识别管道，从而让没有亲缘关系的进程之间建立连接，可以用函数mkfifo()创建。

大家通常说的管道默认情况下是指匿名管道，可以通过pipe函数创建：
```c
#include <unistd.h>
int pipe(int fd[2]);
```
经由参数fd返回两个文件描述符：fd[0]为读而打开，fd[1]为写而打开.fd[1]的输出是fd[0]的输入。如下图所示：
![pipe](/images/c++/pipe/pipe.png)
对于从父进程到子进程的管道，父进程关闭管道的读端(fd[0])，子进程关闭管道的写端(fd[1])；对于从子进程到父进程的管道，父进程关闭fd[1]，子进程关闭fd[0]。当管道的一段被关闭后，下列两条规则起作用：
1. 当读(read)一个写端已被关闭的管道时，在所有数据都被读取后，read返回0，表示文件结束
2. 如果写(write)一个读端已被关闭的管道时，则产生信号SIGPIPE。如果忽略该信号或者捕捉该信号并从其处理程序返回，则write返回-1，errno被设置为EPIPE。

# 管道测试
下面主要测试下父子进程通过匿名管道进行通信的过程：子进程写，父进程读，因此关闭了父进程的写文件描述符和子进程的读文件描述符；主要想看一下，如果管道中没有数据，读操作是否会被阻塞，还是直接返回。
子进程代码如下：
```cpp
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <memory>
#include <iostream>
#include <string>
#include <string>
#include <time.h>
using namespace std;

string get_time()
{
    char buff[30];          // sizeof("2018-04-19 19:49:23") == 20;
    time_t now = time(NULL);
    struct tm *local_time = NULL;
    local_time = localtime(&now);
    strftime(buff, sizeof(buff), "%Y-%m-%d %H:%M:%S ", local_time);
    return string(buff);
}

void test() {
    cerr << get_time() << "[sub process] begin to sleep" << endl;
    sleep(30);
    cerr << get_time() << "[sub process] begin to output" << endl;
    int cnt = 0;
    while (1) {
        ++cnt;
        printf("word %10d", cnt);
        //cout << "word " << to_string(cnt) << endl;
        cerr << get_time() << "[sub process] begin to usleep" << endl;
        usleep(100000);
    }
}

int main() {
    test();
    return 0;
}

# 编译链接
/opt/compiler/gcc-8.2/bin/g++ --std=c++11 ./sub_process.cpp -o sub_process
```
主进程代码如下：
```cpp
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <memory>
#include <iostream>
#include <string.h>
#include <string>
#include <time.h>
using namespace std;

string get_time()
{
    char buff[30];          // sizeof("2018-04-19 19:49:23") == 20;
    time_t now = time(NULL);
    struct tm *local_time = NULL;
    local_time = localtime(&now);
    strftime(buff, sizeof(buff), "%Y-%m-%d %H:%M:%S ", local_time);
    return string(buff);
}

void test() {
    cerr << get_time() << " main process" << endl;
    cerr << get_time() << " fork sub process" << endl;
    FILE* sub_stdin; // user progress stdin --> can write
    FILE* sub_stdout; // user progress stdout --> can read
    int infd[2]; //文件描述符, [0]读管道，[1]写管道
    pipe(infd);
    const char* cmd = "./sub_process";
    int pid = fork();
    if (pid == -1) {
        printf("fork error happens: %s\n", strerror(errno));
        close(infd[0]);
        close(infd[1]);
    } else if (pid == 0) {
        cerr << get_time() << "-----[sub process]-----" << endl;
        // copy to std
        int temp_fd;
        if ((temp_fd = dup2(infd[1], 1)) != 1) {
            fprintf(stderr, "err when dup2 infd 0, return= %d, infd0= %d\n", temp_fd, infd[0]);
        }
        close(infd[0]);
        close(infd[1]);
        fprintf(stderr, "BEFORE execv, cmd=[ %s ]\n", cmd);
        // start user cmd
        const char* args[] = { "/bin/bash", "-c", cmd, NULL };
        if (-1 == execv(args[0], (char * const*)args)) {
            fprintf(stderr, "Error when execv: %s\n", strerror(errno));
        }
    } else {
        cerr << get_time() << "-----[parent process]-----" << endl;
        close(infd[1]); // only read, close write
        fprintf(stderr, "SUCCESS fork, pid=[ %d ], cmd=[ %s ]\n", pid, cmd);
        sub_stdout = fdopen(infd[0], "r");
        while (1) {
            char buf[20];
            cerr << get_time() << "[parent process] before read pipe" << endl;
            int ret = fread(buf, 15, 1, sub_stdout);
            cerr << get_time() << "[parent process] after read pipe" << endl;
            if (ret != 1) {
                cerr << get_time() << "[parent process] read error " << ret << endl;
                cerr << get_time() << "[parent process] begin to usleep 1000" << endl;
                usleep(1000);
            } else {
                buf[16] = '\0';
                fprintf(stderr, "%s [parent process] read buf: %s\n", get_time().c_str(),  buf);
            }
        }
    }
}

int main() {
    test();
    return 0;
}

# 编译链接
/opt/compiler/gcc-8.2/bin/g++ --std=c++11 ./main_process.cpp -o main_process
```
运行结果如下：
![pipe test result](/images/c++/pipe/pipe_run_res.png)

# 总结
实验表明，通过匿名管道进行父子进程的数据通信时，如果管道为空，读操作会被阻塞。

[未完待续！]

# Refer
- 《UNIX环境高级编程》


