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

### 0x1.1 phase_2

接着需要输入第二段字符，首先可以通过 `b phase_2` 直接在第二阶段开始时打上断点，然后随意输入一段字符以进入这个函数。

通过下面的汇编代码片段，发现调用了函数 `read_six_numbers`，其实通过函数名就可以发现这个函数需要输入6个数字，接着紧跟着的代码是执行 `cmpl $0x1, (%rsp)`，如果相等直接跳到 `0x400f30`，否则会调用 `explode_bomb`，这说明了要求输入的第一个数字必须是 1，否则炸弹会爆炸。

```assembly
0000000000400efc <phase_2>:
  400efc:	55                   	push   %rbp
  400efd:	53                   	push   %rbx
  400efe:	48 83 ec 28          	sub    $0x28,%rsp
  400f02:	48 89 e6             	mov    %rsp,%rsi
  400f05:	e8 52 05 00 00       	callq  40145c <read_six_numbers>
  400f0a:	83 3c 24 01          	cmpl   $0x1,(%rsp)
  400f0e:	74 20                	je     400f30 <phase_2+0x34>
  400f10:	e8 25 05 00 00       	callq  40143a <explode_bomb>
  400f15:	eb 19                	jmp    400f30 <phase_2+0x34>
```

为了进一步确定 `read_six_numbers` 是要求我们输入6个数字，我们搜索对应的函数名，可以获得下面的汇编代码段

```assembly
000000000040145c <read_six_numbers>:
  40145c:	48 83 ec 18          	sub    $0x18,%rsp
  401460:	48 89 f2             	mov    %rsi,%rdx
  401463:	48 8d 4e 04          	lea    0x4(%rsi),%rcx
  401467:	48 8d 46 14          	lea    0x14(%rsi),%rax
  40146b:	48 89 44 24 08       	mov    %rax,0x8(%rsp)
  401470:	48 8d 46 10          	lea    0x10(%rsi),%rax
  401474:	48 89 04 24          	mov    %rax,(%rsp)
  401478:	4c 8d 4e 0c          	lea    0xc(%rsi),%r9
  40147c:	4c 8d 46 08          	lea    0x8(%rsi),%r8
  401480:	be c3 25 40 00       	mov    $0x4025c3,%esi
  401485:	b8 00 00 00 00       	mov    $0x0,%eax
  40148a:	e8 61 f7 ff ff       	callq  400bf0 <__isoc99_sscanf@plt>
  40148f:	83 f8 05             	cmp    $0x5,%eax
  401492:	7f 05                	jg     401499 <read_six_numbers+0x3d>
  401494:	e8 a1 ff ff ff       	callq  40143a <explode_bomb>
  401499:	48 83 c4 18          	add    $0x18,%rsp
  40149d:	c3                   	retq   
```

注意 `0x40148f` 对应的代码行，scanf 返回的结果和 5 进行比较，需要比 5 大，否则炸弹会爆炸。

接着代码会跳到 `0x400f30`，继续往下找对应的代码，会发现这一段代码比较重要。

```assembly
  400f17:	8b 43 fc             	mov    -0x4(%rbx),%eax    
  400f1a:	01 c0                	add    %eax,%eax
  400f1c:	39 03                	cmp    %eax,(%rbx)
  400f1e:	74 05                	je     400f25 <phase_2+0x29>
  400f20:	e8 15 05 00 00       	callq  40143a <explode_bomb>
  400f25:	48 83 c3 04          	add    $0x4,%rbx
  400f29:	48 39 eb             	cmp    %rbp,%rbx
  400f2c:	75 e9                	jne    400f17 <phase_2+0x1b>
  400f2e:	eb 0c                	jmp    400f3c <phase_2+0x40>
  400f30:	48 8d 5c 24 04       	lea    0x4(%rsp),%rbx
  400f35:	48 8d 6c 24 18       	lea    0x18(%rsp),%rbp
  400f3a:	eb db                	jmp    400f17 <phase_2+0x1b>
```

其中 `(%rsp)` 表示第一个输入的数字，因为是 int 类型，所以内存中占用的长度是**4个字节**，那么`0x4(%rsx)` 表示第二个输入的数字，`0x18(%rsp)` 并不是表示第六个输入的数字，而是表示第六个数字之后的地址，后面会讲为什么需要这样设置。

可以发现代码又跳转到 `0x400f17` 地址，首先 `mov  -0x4(%rbx),%eax` 即可将第一个输入的数字放到寄存器 `%eax` 中，然后 `add %eax,%eax` 其实就是将数字`*2`，然后判断 `%eax` 和 `(%rbx)` 是否相等，如果不相等则炸弹会爆炸，由此可以看出我们需要输入6个数字，第一个是1，然后每一个后面的数字都是前面数字的两倍。

最后可以通过 `add $0x4,%rbx` 不断遍历下一个元素，然后通过 `%rbp` 和 `%rbx` 是否相等的判断来决定跳出循环，注意 `(%rsp)` 表示第一个数字，那么`0x14(%rsp)`表示第六个数字，所以`0x18(%rsp)`表示第六个数字之后的地址，通过和这个地址作比较，可以确保遍历完所有的6个数字。

最后这6个数字为 `1 2 4 8 16 32`，输入这些数字即可解除第二阶段的炸弹。

