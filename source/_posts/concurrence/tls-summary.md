---
title: 初识Thread Local Storage
date: 2019-09-30 20:42:35
tags: concurrence
categories: concurrence
---

> 当程序运行时，函数的局部变量是在线程的栈上进行分配，虽然线程共享进程的虚拟地址空间，但因为每个线程有自己的线程栈，所以栈中的数据是互相隔离的，互不侵扰；而全局变量在`heap`上进行分配，`heap`在各个线程间是共享的，所以在对共享的资源进行读写时，需要有同步机制来确保线程安全；然而有一种多线程下的编程方式，可以使得全局变量或静态变量只对单个线程可见，而对其它线程不可见，这就是`Thread Local Storage`，又叫`线程本地存储`或`线程局部存储`

## Thread Local Storage

`维基百科`上对Thread Local Storage的解释如下:

> Thread-local storage (TLS) is a computer programming method that uses static or global memory local to a thread.

翻译下来就是：`线程本地存储(TLS)，对于线程来讲是一种对本地化使用静态或全局内存的计算机编程方法。`

线程局部存储(TLS)是一个后来者, 产生于多线程概念之后.而在软件发展的早期, 全局变量经常用在库函数中, 用于存储全局信息, 比如errno, 多线程程序产生之后, 全局变量errno就成为所有线程都共享的一个变量, 而实际上, 每个线程都想维护一份自己的errno, 隔离于其他线程.这个时候, 没人愿意去修改库函数的接口. 于是线程局部存储就诞生了, 它主要是为了避免多个线程同时访存同一全局变量或者静态变量时所导致的冲突，尤其是多个线程同时需要修改这一变量时，而这些变量逻辑上又可以在各个线程中独立，也就是说线程并不共享这些变量。

 为了解决这个问题，我们可以通过TLS机制，为每一个使用该全局变量的线程都提供一个变量值的副本，每一个线程均可以独立地改变自己的副本，而不会和其它线程的副本冲突。从线程的角度看，就好像每一个线程都完全拥有该变量。而从全局变量的角度上来看，就好像一个全局变量被克隆成了多份副本，而每一份副本都可以被一个线程独立地改变。
## TLS简单使用

### C语言实现

编写如下一段程序：
```c
#define _MULTI_THREADED

#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <unistd.h>

__thread int TLS_data1;
__thread int TLS_data2;

//int TLS_data1;
//int TLS_data2;

#define                 NUMTHREADS   4


void *theThread(void *a) {
    int arg = *(int *)a;
    printf("Thread %lu before change: arg:%d TLS data=%d %d\n",
           pthread_self(), arg,  TLS_data1, TLS_data2);
    TLS_data1 = arg;
    TLS_data2 = arg +1;
    printf("Thread %lu after change: arg:%d TLS data=%d %d\n",
           pthread_self(), arg,  TLS_data1, TLS_data2);
    return NULL;
}


int main(int argc, char **argv) {
    pthread_t thread[NUMTHREADS];
    int rc = 0;
    int i;

    int ar[NUMTHREADS];

    printf("Enter Testcase - %s\n", argv[0]);

    printf("Create/start threads\n");
    for (i = 0; i < NUMTHREADS; i++) {
        /* Create per-thread TLS data and pass it to the thread */
        ar[i] = i;
        rc = pthread_create(&thread[i], NULL, theThread, &ar[i]);
    }

    printf("Wait for the threads to complete, and release their resources\n");
    for (i = 0; i < NUMTHREADS; i++) {
        rc = pthread_join(thread[i], NULL);
    }
    printf("Main completed\n");
    return 0;
}
```
该段程序使用`__thread`声明了两个变量`TLS_data1`和`TLS_data2`为线程局部存储，然后分别在4个线程中修改他们的值，观察运行结果：

```shell
Create/start threads
Thread 139919563978496 before change: arg:0 TLS data=0 0
Thread 139919563978496 after change: arg:0 TLS data=0 1
Thread 139919547193088 before change: arg:2 TLS data=0 0
Thread 139919547193088 after change: arg:2 TLS data=2 3
Wait for the threads to complete, and release their resources
Thread 139919538800384 before change: arg:3 TLS data=0 0
Thread 139919538800384 after change: arg:3 TLS data=3 4
Thread 139919555585792 before change: arg:1 TLS data=0 0
Thread 139919555585792 after change: arg:1 TLS data=1 2
Main completed
```
可以看到每个线程可以从容的修改他们。并且相互之间没有造成干扰，那么我们去掉`__thread`而使用普通的全局变量的话，就会使这段程序变得线程不安全：
```shell
Create/start threads
Thread 140450580477696 before change: arg:0 TLS data=0 0
Thread 140450580477696 after change: arg:0 TLS data=0 1
Thread 140450572084992 before change: arg:1 TLS data=0 1
Thread 140450572084992 after change: arg:1 TLS data=1 2
Thread 140450563692288 before change: arg:2 TLS data=1 2
Thread 140450563692288 after change: arg:2 TLS data=2 3
Wait for the threads to complete, and release their resources
Thread 140450555299584 before change: arg:3 TLS data=2 3
Thread 140450555299584 after change: arg:3 TLS data=3 4
Main completed
```

可以试着删除`__thread`关键字，再编译运行观察，你会看到一个错乱的运行结果。

### python实现

把上面的例子用python来实现：
```python
import threading

local = threading.local()
local.TLS_data1 = 0
local.TLS_data2 = 0


def func(info):
    myname = threading.currentThread().getName()
    local.TLS_data1 = info
    local.TLS_data2 = info + 1
    print('Thread {0} after change TLS data: {1}, {2}'.format(myname, local.TLS_data1, local.TLS_data2))


t = [0, 1, 2, 3]
for i in t:
    t[i] = threading.Thread(target=func, args=[i])
    t[i].start()


for i, v in enumerate(t):
    v.join()


print('Thread {0} TLS data: {1}, {2}'.format("main", local.TLS_data1, local.TLS_data2))
```

执行结果：
```shell
Thread Thread-1 after change TLS data: 0, 1
Thread Thread-2 after change TLS data: 1, 2
Thread Thread-3 after change TLS data: 2, 3
Thread Thread-4 after change TLS data: 3, 4
Thread main TLS data: 0, 0
```

## TLS的误区

网上有很多文章误将TLS当成是编写线程安全代码的银弹，其实哪里是这样，这都取决于`global variable`在你的线程之间是不是`shared`，如果你的本意就是共享，那么TLS反而使你南辕北辙，你仍然需要`mutex`之类的锁去同步你的操作，那么TLS的本质到底是什么？

我认为TLS的本质就是填补了全局变量和局部变量之间的空白，它不像全局变量那样在多个线程中可见，也不像局部变量那样仅仅生存在在函数的作用域之内，它的可见度，大于局部变量，又小于全局变量。

## TLS适用场景

综上所述，线程本地存储并不是解决多线程变量共享的并发问题，而是限制变量仅在当前线程中可见，可想而知这样带来的好处之一就是线程内各个方法之间不用再通过传参就可以共享变量；另外一个可想而知的使用场景就是可以实现每个线程需要单独拥有一个实例的情况。

还有一个也是[wiki](https://en.wikipedia.org/wiki/Thread-local_storage)上提到的，就是对一个`global variable`进行累加的情况，为了避免`race condition`的传统做法是使用`mutex`，但也可以使用TLS先在每个线程本地累加，然后再讲每个线程的累加结果同步到一个真正的`global variable`之上。

当然，我认为TLS的适用场景肯定远不止这些，只是我个人平时工作当中，编码并不是很多，其中多线程编程便又少了一些，而在多线程编程中适用TLS的情况更是为零，本篇文章仅当做学习过程中的记录，以后有更深层次的思考会随时补充，也希望大家可以共同探讨。


**参考文章：**

1. [Thread-local storage](https://en.wikipedia.org/wiki/Thread-local_storage)
2. [线程局部存储漫谈](http://ju.outofmemory.cn/entry/66238)
3. [A Deep dive into (implicit) Thread Local Storage](https://chao-tic.github.io/blog/2018/12/25/tls)