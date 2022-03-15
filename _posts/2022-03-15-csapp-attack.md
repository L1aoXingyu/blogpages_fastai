---
toc: true
layout: post
description: 记录在阅读和学习 CSAPP 过程中，完成 AttackLab 的相关内容
comments: true
categories: [operation system, c, csapp, assembly, attack, disassembly]
title: "CSAPP 之 Attack Lab"
image: images/csapp-attack/attack_preview.png
---

### 0x0 Introduction

在学习完汇编和系统安全相关的内容之后，为了更好的理解相关的内容，csapp 提供了一个配套的 lab，提供了两种方式的攻击，一种是直接写入 code，让程序在 return 时执行我们传入的另外一个函数地址；另外一种方式是通过组合代码中已有的片段，让代码按照我们想要的方式运行，最终被我们接管。

通过完成 attack lab，可以更好地理解 buffer overflow 相关的内容，同时也可以真正理解 code-injection 和 return-oriented 这两种攻击方式，最终可以更好地帮助我们写出安全漏洞更小的代码
。

在完成 lab 的同时，也可以让我们对 `gdb` 以及 `objdump` 等相关工具更熟练的掌握，对 stack frame 有深入的理解。

下面我们先介绍一下完成这个 lab 之前的一些工具准备工作，之后再分别介绍两种不同的攻击方式是如何实现的。

#### 0x0.1 Preparation

1. 第一个需要使用的工具是 `gcc`，需要用它来编译汇编代码，因为我们要在程序中插入自己的代码，所以需要用 `gcc` 将这段汇编代码编译，等待后续的使用；

   ```shell
   gcc -c test.s
   ```

2. 接着需要用到 `objdump`，这是用来将汇编代码转换成实际的机器码，因为需要传递给计算机执行，不能将汇编代码直接传进去，需要先通过 `gcc` + `objdump` 将汇编代码变成机器码；

    ```shell
    objdump -d test.o
    ```

3. 最后需要使用的是 lab 中提供的工具 `hex2raw`，这个工具可以将我们在前两个步骤中获得的机器码转换成2进制的字节码，这正是计算机真正执行的代码；

    ```shell
    ./hex2raw < exploit.txt > exploit.bin
    ```

通过这三个工具我们就可以将需要执行的汇编代码转换成字节码，最终送到计算机中进行执行，这三个工具也给了我们做 code-injection 攻击的基础。

### 0x1 Code Injection Attacks

#### 0x1.1 Level 1

#### 0x1.2 Level 2

#### 0x1.3 Level 3

### 0x2 Return-Oriented Programming

#### 0x2.1 Level 4

#### 0x2.2 Level 5

### 0x3 Conclusion

### 0x4 Reference
