---
toc: true
layout: post
description: 记录在阅读和学习 CSAPP 过程中，完成 BombLab 的相关内容
comments: true
categories: [operation system, c, csapp, assembly]
title: "CSAPP 之 Bomb Lab"
---

### 0x0 Introduction

该实验提供了一个 "binary bomb"，这是一个编译好的二进制程序，在运行过程中需要用户提供6次输入，如果有任何一次不正确，炸弹就会爆炸。

在这个实验中，我们需要做逆向工程对炸弹进行拆解，对这个二进制程序进行反汇编后，通过阅读他的汇编代码，获取需要输入的6个字符串是什么。

在这次实验中，主要使用的工具如下：

- `objdump`: 将二进制文件进行反汇编；
- `vscode`：阅读和注解汇编代码的编辑器；
- `cgdb(gdb)`：单步调试汇编代码的 debugger；

通过下面的命令即可对二进制程序进行反汇编，最终获得 `bomb.asm` 代码，还可以在 vscode 中安装 asm 插件实现代码的高亮。

```shell
objdump bomb -d > bomb.asm
```

<img src="{{ site.baseurl }}/images/csapp-bomb/bomb_preview.png" width="700"/>

### 0x1.0 phase_1

下面的c代码包含了第一阶段的所有内容

```c
/* Hmm...  Six phases must be more secure than one phase! */
input = read_line();             /* Get input                   */
phase_1(input);                  /* Run the phase               */
phase_defused();                 /* Drat!  They figured it out!
            * Let me know how they did it. */
/* Border relations with Canada have never been better. */
```



代码非常简单，程序读取输入之后，送到 `phase_1` 这个函数中进行处理，如果函数顺利退出则表示炸弹被拆掉，输入指令正确。

通过在汇编代码中搜索 `phase_1` 即可找到其在汇编代码中的片段

```assembly
0000000000400ee0 <phase_1>:
  400ee0:	48 83 ec 08          	sub    $0x8,%rsp
  400ee4:	be 00 24 40 00       	mov    $0x402400,%esi
  400ee9:	e8 4a 04 00 00       	callq  401338 <strings_not_equal>
  400eee:	85 c0                	test   %eax,%eax
  400ef0:	74 05                	je     400ef7 <phase_1+0x17>
  400ef2:	e8 43 05 00 00       	callq  40143a <explode_bomb>
  400ef7:	48 83 c4 08          	add    $0x8,%rsp
  400efb:	c3                   	retq   
```



简单阅读这一段汇编代码，主要是调用了 `strings_not_equal` 这个函数，而通过名字可以大概了解这就是一个判断字符串是否相等的函数。

而上面有一段代码将地址 `0x402400` 移动到寄存器 `%esi` 中，通过查表可以了解到 `%esi` 是函数的第二个参数，而调用 `phase_1` 时，汇编代码如下，发现是通过 `read_line` 这个函数，将用户输入的结果移动到寄存器 `%rdi` 中，而这恰好是函数的第一个参数，所以可以得到第一阶段即要对比用户输入的字符串和 `0x402400` 这个地址存放的字符串是否相等。

```assembly
  400e32:	e8 67 06 00 00       	callq  40149e <read_line>
  400e37:	48 89 c7             	mov    %rax,%rdi
  400e3a:	e8 a1 00 00 00       	callq  400ee0 <phase_1>
```

通过 `cgdb` 启动程序，在 `phase_1` 处打断点，注意这里断点需要打到汇编代码中，可以直接通过 `b phase_1` 打断点，也可以通过地址进行打断点，比如 `b *0x400ee0`，接着直接运行。

在 gdb 中通过 `ni` 可以在汇编代码中进行单步运行，我们可以运行到 `strings_not_equal` 这个函数调用前，接着通过 `p (char*)0x402400` 即可获得这个地址对应的字符串如下

```javascript
(gdb) p (char*)(0x402400)
$6 = 0x402400 "Border relations with Canada have never been better."
```

接着可以尝试在 `phase_1` 输入这段字符串，可以获得下面的结果，说明 `pahse_1` 的输入指令正确，炸弹已经成功拆除了。

```javascript
Phase 1 defused. How about the next one?
```

