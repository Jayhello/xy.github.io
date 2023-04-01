---
layout: post
title: "协程原理与实现"
date:   2022-11-22
tags: [协程]
comments: true
author: xy
# toc: true  # 这里加目录导致看到地方太小了
---

本文主要是协程的原理与实现.

<!-- more -->

## 1. 开始
下面给下`linux`下`c`协程封装的主要函数:

```
// 用户上下文的获取和设置
int getcontext(ucontext_t *ucp);
int setcontext(const ucontext_t *ucp);

// 操纵用户上下文
void makecontext(ucontext_t *ucp, void (*func)(void), int argc, ...);
int swapcontext(ucontext_t *oucp, const ucontext_t *ucp);
```
context族是偏向底层的，其实底层就是通过汇编来实现的.寄存器：
%rax作为函数返回值使用
%rsp栈指针寄存器， 指向栈顶
%rdi, %rsi, %rdx, %rcx, %r8, %r9用作函数的参数，从前往后依次对应第1、第2、…

第n参数
%rbx, %rbp, %r12, %r13, %r14, %r15用作数据存储，遵循被调用这使用规- 则，调用子函数之前需要先备份...
ucontext_t是一个结构体变量，其功能就是通过定义一个ucontext_t来保存当前上下文信息的。

## 2. 基本函数

函数：void makecontext(ucontext_t *ucp, void (*func)(), int argc, ...)
功能：修改上下文信息，参数ucp就是我们要修改的上下文信息结构体；func是上下文的入口函数；argc是入口函数的参数个数，后面的…是具体的入口函数参数
这里就是将func的地址保存到寄存器中，把ucp上下文结构体下一条要执行的指令rip改变为func函数的地址。并且将其所运行的栈改为用户自定义的栈


```
void fun(){
    printf("fun()\n");
}
void example_4(){
    int i = 0;
    //定义用户的栈
    char* stack = (char*)malloc(sizeof(char)*8192);

    //定义两个上下文
    //一个是主函数的上下文，一个是fun函数的上下文
    ucontext_t ctx_main, ctx_fun;

    getcontext(&ctx_main);
    getcontext(&ctx_fun);
    printf("i = %d\n", i++);
    sleep(1);

    //设置fun函数的上下文
    //使用getcontext是先将大部分信息初始化，我们到时候只需要修改我们所使用的部分信息即可
    ctx_fun.uc_stack.ss_sp = stack;//用户自定义的栈
    ctx_fun.uc_stack.ss_size = 8192;//栈的大小
    ctx_fun.uc_link = &ctx_main;//该上下文执行完后要执行的下一个上下文
    makecontext(&ctx_fun, fun, 0);//将fun函数作为ctx_fun上下文的下一条执行指令

    setcont
```

![net](https://raw.githubusercontent.com/Jayhello/Jayhello.github.io/master/images/net/recv_1.png)
