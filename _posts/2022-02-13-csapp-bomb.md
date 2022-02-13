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

### 0x1.2 phase_3

接着需要输入第三段字符，首先可以通过 `b phase_3` 直接在第三阶段开始的时候打上断点，然后输入一段字符以进入这个函数。

通过阅读下面这段汇编代码，可以发现，`scanf` 的返回值需要大于`1`，否则炸弹就会爆炸，这也就是说至少需要输入`2`个数字。

接着继续查看下面的代码，发现需要比较 `$0x7` 和 `0x8(%rsp)`，通过 `x (%rsp+0x8)` ，可以发现`0x8(%rsp)` 表示第一个数字的地址，进一步可以发现 `0xc(%rsp)`表示第二个数字的地址，地址差距是4个字节，说明是 int32 的数据类型。

```assembly
  400f6a:	83 7c 24 08 07       	cmpl   $0x7,0x8(%rsp)
  400f6f:	77 3c                	ja     400fad <phase_3+0x6a>
  ...
  400fad:	e8 88 04 00 00       	callq  40143a <explode_bomb>
```

所以这一段代码即表示如果输入的第一个参数比`7`大，则会跳到 `0x400fad` 这个位置继续执行，而这个位置执行的函数为 `explode_bomb`，这也就是间接说明了需要输入的第一个数字`<=7`。

接着继续执行下面的代码，首先将第一个数字放到寄存器 `$eax` 中，然后执行 `jmpq` 指令，跳转的地址是 `0x402470 + 8 * $rax` 位置的指针指向的内容，这里我们可以通过在 `gdb` 中输入 `ni` 进行单步执行，来判断最终代码跳转的位置，如果输入的第一个数字是`1`，那么也可以通过`p/x *(0x402470+8)`或者是 `x/x 0x402470+8` 来计算出具体的位置。

```assembly
  400f71:	8b 44 24 08          	mov    0x8(%rsp),%eax
  400f75:	ff 24 c5 70 24 40 00 	jmpq   *0x402470(,%rax,8)
```

接着直接跳到对应的位置去查看代码，可以发现下面的代码需要比较输入的第二个数字和 `0x137` 的大小，因为首先通过 `mov $0x137,%eax` 将 `0x137` 移动到寄存器 `$eax` 中，然后通过 `cmp 0xc(%rsp),%eax` 来比较寄存器 `$eax` 的值以及地址 `0xc(%rsp)`  指向内存的值，从前面可以知道这是第二个输入的数字。

```assembly
  400fb9:	b8 37 01 00 00       	mov    $0x137,%eax
  400fbe:	3b 44 24 0c          	cmp    0xc(%rsp),%eax
  400fc2:	74 05                	je     400fc9 <phase_3+0x86>
  400fc4:	e8 71 04 00 00       	callq  40143a <explode_bomb>
```

所以输入的第二个数字换成十进制则是 `311` ，于是输入 `1 311` 即可接触第三个炸弹，因为要求第一个数字 `<=7`，所以这里只描述了一种可能性，也可以尝试第一个数字输入是`2`，会发现程序会跳到 `0x400f83`，需要输入的第二个数字是 `707`.

### 0x1.3 phase_4

下一个阶段主要考察了对递归函数的 reverse engineering，首先在 `phase_4` 开始阶段打上断点，然后运行进入，首先查看下面这段汇编代码

```assembly
  40101a:	be cf 25 40 00       	mov    $0x4025cf,%esi
  40101f:	b8 00 00 00 00       	mov    $0x0,%eax
  401024:	e8 c7 fb ff ff       	callq  400bf0 <__isoc99_sscanf@plt>
  401029:	83 f8 02             	cmp    $0x2,%eax
  40102c:	75 07                	jne    401035 <phase_4+0x29>
  40102e:	83 7c 24 08 0e       	cmpl   $0xe,0x8(%rsp)
  401033:	76 05                	jbe    40103a <phase_4+0x2e>
  401035:	e8 00 04 00 00       	callq  40143a <explode_bomb>
```

可以发现需要我们通过 `scanf` 输入一些内容，通过在 `gdb` 中查看 `scanf` 需要的输入个数和类型如下

```javascript
(gdb) x /s 0x4025cf
0x4025cf:       "%d %d"
```

说明需要输入两个 `int` 类型的数字，同时通过 `cmp $0x2,%eax` 也可以证实这点，另外还通过 `cmpl $0xe,0x8(%rsp)` 要求输入的第一个数字必须要 `<=14`，接着代码会跳到地址 `40103a` 继续执行。

然后查看对应位置的汇编代码，发现整体逻辑是调用 `func4`，判断返回值是否为`0`，如果不是 `0`，那么直接跳到 `0x401058` 就触发爆炸，如果是 `0` 则可以继续判断输入的第二个数字是否是 `0`，如果是 `0`，那么顺利解除炸弹。

```assembly
  40103a:	ba 0e 00 00 00       	mov    $0xe,%edx
  40103f:	be 00 00 00 00       	mov    $0x0,%esi
  401044:	8b 7c 24 08          	mov    0x8(%rsp),%edi
  401048:	e8 81 ff ff ff       	callq  400fce <func4>
  40104d:	85 c0                	test   %eax,%eax
  40104f:	75 07                	jne    401058 <phase_4+0x4c>
  401051:	83 7c 24 0c 00       	cmpl   $0x0,0xc(%rsp)
  401056:	74 05                	je     40105d <phase_4+0x51>
  401058:	e8 dd 03 00 00       	callq  40143a <explode_bomb>
```

看到这里，整体已经比较清晰了，输入两个数字，第一个需要不比 `14` 大，第二个必须是`0`，然后调用 `func4` 需要保证返回值是 `0`，而其输入的参数分别在寄存器 `%rdi, %rsi, %rdx`，继续分析这些寄存器的获得方法，可以发现，第一个参数就是我们输入的第一个数字，第二个参数是 `0`，第三个参数是 `14`，所以接下来只需要进一步分析 `func4` 即可。

在汇编代码中搜索 `func4` 即可找到对应的代码片段

```assembly
0000000000400fce <func4>:
  400fce:	48 83 ec 08          	sub    $0x8,%rsp
  400fd2:	89 d0                	mov    %edx,%eax
  400fd4:	29 f0                	sub    %esi,%eax
  400fd6:	89 c1                	mov    %eax,%ecx
  400fd8:	c1 e9 1f             	shr    $0x1f,%ecx
  400fdb:	01 c8                	add    %ecx,%eax
  400fdd:	d1 f8                	sar    %eax
  400fdf:	8d 0c 30             	lea    (%rax,%rsi,1),%ecx
  400fe2:	39 f9                	cmp    %edi,%ecx
  400fe4:	7e 0c                	jle    400ff2 <func4+0x24>
  400fe6:	8d 51 ff             	lea    -0x1(%rcx),%edx
  400fe9:	e8 e0 ff ff ff       	callq  400fce <func4>
  400fee:	01 c0                	add    %eax,%eax
  400ff0:	eb 15                	jmp    401007 <func4+0x39>
  400ff2:	b8 00 00 00 00       	mov    $0x0,%eax
  400ff7:	39 f9                	cmp    %edi,%ecx
  400ff9:	7d 0c                	jge    401007 <func4+0x39>
  400ffb:	8d 71 01             	lea    0x1(%rcx),%esi
  400ffe:	e8 cb ff ff ff       	callq  400fce <func4>
  401003:	8d 44 00 01          	lea    0x1(%rax,%rax,1),%eax
  401007:	48 83 c4 08          	add    $0x8,%rsp
  40100b:	c3                   	retq   
```

整体代码并不长，但是里面有两处存在递归调用，如果使用 `gdb` 进行单步调试会比较绕，所以这里直接对汇编代码进行反向工程，获得对应的c代码如下

```c
int func4(int x, int a1, int a2) {
  int m = (a2 - a1) / 2;
  int n = m + a1;

  if (n == x) return 0;
  if (n < x) {
    m = func4(x, n + 1, a2);
    return 2 * m + 1;
  } else {
    m = func4(x, a1, n - 1);
    return 2 * m;
  }
}
```

因为该函数需要返回的值是 0，所以只能走 `n==x` 和 `n>x` 这两个分支，否则会返回 `2*m+1`，那么结果必然不会是 `0`，而由前面知道 `a1=0, a2=14`，所以不断计算 `n` 的值即可求出所有符合条件的输入 x，如果嫌麻烦，也可以直接把运行这段代码来找符合条件的值。

最终符合条件的值是 `0,1,3,7`，输入这四个值中的任意一个作为第一个数字输入，均可成功拆除炸弹。

### 0x1.4 phase_5

进入到 `phase_5` 之后，注意到下面这一段汇编代码

```assembly
  401078:	31 c0                	xor    %eax,%eax
  40107a:	e8 9c 02 00 00       	callq  40131b <string_length>
  40107f:	83 f8 06             	cmp    $0x6,%eax
  401082:	74 4e                	je     4010d2 <phase_5+0x70>
  401084:	e8 b1 03 00 00       	callq  40143a <explode_bomb>
```

其中 `xor %eax,%eax` 其实等价于将 `%eax` 置零，然后通过 `string_length` 来判断输入的字符串长度，后面通过 `cmp $0x6,%eax` 比较字符串长度和 `6` 的大小，如果不相等则会引发爆炸，如果相等，可以跳到地址为 `0x4010d2` 继续执行。

继续往后执行，可以发现会进入到下面这一段循环代码中

```assembly
  40108b:	0f b6 0c 03          	movzbl (%rbx,%rax,1),%ecx
  40108f:	88 0c 24             	mov    %cl,(%rsp)
  401092:	48 8b 14 24          	mov    (%rsp),%rdx
  401096:	83 e2 0f             	and    $0xf,%edx
  401099:	0f b6 92 b0 24 40 00 	movzbl 0x4024b0(%rdx),%edx
  4010a0:	88 54 04 10          	mov    %dl,0x10(%rsp,%rax,1)
  4010a4:	48 83 c0 01          	add    $0x1,%rax
  4010a8:	48 83 f8 06          	cmp    $0x6,%rax
  4010ac:	75 dd                	jne    40108b <phase_5+0x29>
```

我们可以逐步分析每一行代码的含义，其中 `%rbx` 表示输入的字符串开头，在 `gdb`  中运行 `p %rax` 获得初始值是 `0`，那么这里相当于首先把输入字符串的第一个字符放到寄存器 `%ecx` 中；

接着两行 `mov` 指令将 `%ecx` 的最低 `8` 位寄存器 `%cl` 的结果放到寄存器 `%rdx` 中，因为不能直接从一个寄存器到另外一个寄存器，所以中间需要一次内存的转换，最终 `%rdx` 里面存放了刚才获得的字符；

下面一条指令 `and $0xf,%edx` 会将 `%edx` 在 `4` 位之后的值全部设为 `0`，因为 `0xf` 转换成二进制是 `0b1111`；

 然后程序会以地址 `0x4024b0` 为起点，计算移动之后的地址所指向的内容，用寄存器 `%edx` 进行保存，可以通过在 `gdb` 中访问这个地址所对应的内存，得到下面的结果

```javascript
(gdb) x /s 0x4024b0
0x4024b0 <array.3449>:  "maduiersnfotvbylSo you think you can stop the bomb with ctrl-c, do you?"
```

最后程序会将寄存器 `%edx` 的低 `8` 位寄存器 `%dl` 中的值放到地址为 `0x10(%rsp,%rax,1)` 的内存中，接着继续遍历后续的字符，直到遍历完输入的 `6` 个字符为止。

直到此刻位置，我们仍然不清楚程序的要求，按照上面的流程，会根据输入字符的编码数字，经过一系列运算得到一个 `offset`，最后根据这个 `offset` 从一个比较长的字符串里面获得对应的字符，接下来继续跟着程序往后运行。

在跳出循环好，代码会进入到这个地方

```assembly
  4010ae:	c6 44 24 16 00       	movb   $0x0,0x16(%rsp)
  4010b3:	be 5e 24 40 00       	mov    $0x40245e,%esi
  4010b8:	48 8d 7c 24 10       	lea    0x10(%rsp),%rdi
  4010bd:	e8 76 02 00 00       	callq  401338 <strings_not_equal>
  4010c2:	85 c0                	test   %eax,%eax
  4010c4:	74 13                	je     4010d9 <phase_5+0x77>
  4010c6:	e8 6f 03 00 00       	callq  40143a <explode_bomb>
```

首先获得地址为 `$0x40245e` 的字符串，用寄存器 `%esi` 保存它的起始地址，同时在 `%rdi` 中保存了刚才循环生成的字符串，接着调用 `strings_not_equal` ，通过名字可以知道它是在判断这两个字符串是否相等。

通过 `gdb` 查看以地址 `$0x40245e` 为起点的字符串是

```javascript
(gdb) x /s 0x40245e
0x40245e:       "flyers"
```

最终的问题就变成了要从序列中 `maduiersnfotvbylSo you think you can stop the bomb with ctrl-c, do you?` 中选择对应位置的字符，使得最终字符是 `flyers`，这里存在多种选择方式，比如 `l` 在序列中出现在了多个位置，我们可以选择第一次出现对应字符的位置。

重新查看上面对输入字符的运算流程，每次输入的字符被转变成二进制，最终只会保留最低的 `4` 位，比如输入 `a=97=1100001`，经过计算最后会变成 `0001`，这其实相当于进行 `mod 16` 的运算。

分析 `flyers` 分别在序列中最先出现的位置是 `9, 15, 14, 5, 6, 7`，如果我们都输入小写字母，那么起始点就是 `96`，因为在 `anscii` 编码中 `a=97`，比他小且最靠近它的 `16` 的倍数就是 `96`，所以上面的值分别加上 `96` 可以得到下面的结果 `105, 111, 110, 101, 102, 103`，再将其转换成字符可以获得最后输入的 `6` 个字符分别是 `ionefg`。

### 0x1.5 phase_6

第6阶段会比较复杂，主要考察了 `linked list` 这种数据结构，我们还是像之前一样，逐步分析。

首先进入的代码如下

```assembly
  401100:	49 89 e5             	mov    %rsp,%r13
  401103:	48 89 e6             	mov    %rsp,%rsi
  401106:	e8 51 03 00 00       	callq  40145c <read_six_numbers>
  40110b:	49 89 e6             	mov    %rsp,%r14
  40110e:	41 bc 00 00 00 00    	mov    $0x0,%r12d
  401114:	4c 89 ed             	mov    %r13,%rbp
  401117:	41 8b 45 00          	mov    0x0(%r13),%eax
  40111b:	83 e8 01             	sub    $0x1,%eax
  40111e:	83 f8 05             	cmp    $0x5,%eax
  401121:	76 05                	jbe    401128 <phase_6+0x34>
  401123:	e8 12 03 00 00       	callq  40143a <explode_bomb>
  401128:	41 83 c4 01          	add    $0x1,%r12d
```

先通过 `read_six_numbers` 读取 `6` 个数字，而 `$rsp, $r13, $rsi` 这三个寄存器里面都存放了相同的地址，可以通过在 `gdb` 中使用 `x /6 $rsi` 验证这里存放的是输入的 `6` 个数字的起始地址。

`mov 0x0(%r13),%eax` 会将输入的第一个数字放到寄存器 `$eax` 中，后面会对他做 `-1` 的运算，然后和 `0x5` 比较大小，必须要比他小或相等，否则炸弹会引爆，即输入的第一个数字必须要 `<=6`。

然后代码会进入到下面的地方

```assembly
  401128:	41 83 c4 01          	add    $0x1,%r12d
  40112c:	41 83 fc 06          	cmp    $0x6,%r12d
  401130:	74 21                	je     401153 <phase_6+0x5f>
  401132:	44 89 e3             	mov    %r12d,%ebx
  401135:	48 63 c3             	movslq %ebx,%rax
  401138:	8b 04 84             	mov    (%rsp,%rax,4),%eax
  40113b:	39 45 00             	cmp    %eax,0x0(%rbp)
  40113e:	75 05                	jne    401145 <phase_6+0x51>
  401140:	e8 f5 02 00 00       	callq  40143a <explode_bomb>
  401145:	83 c3 01             	add    $0x1,%ebx
  401148:	83 fb 05             	cmp    $0x5,%ebx
  40114b:	7e e8                	jle    401135 <phase_6+0x41>
```

注意到前面的代码中将 `$r12d` 置为 `0`，然后这里会 `+1` 和 `6` 进行比较，如果相等则会直接跳到地址 `0x401153`，说明这是在遍历输入的 `6` 个数。

`mov (%rsp,%rax,4),%eax` 不断遍历输入的数字，此时 `$rax=1`，所以是把第二个数字放到寄存器 `$eax` 中，然后和 `0x0(%rbp)` 进行比较，前面我们知道 `$rbp` 存放了第一个数字的地址，也可以在 `gdb` 中进行确认，所以这是将后面输入的数字都和第一个数字比较，需要他们不等，否则就会引爆炸弹。

跳出这个循环之后，会进入到下面这段代码

```assembly
  401114:	4c 89 ed             	mov    %r13,%rbp
  401117:	41 8b 45 00          	mov    0x0(%r13),%eax
  40111b:	83 e8 01             	sub    $0x1,%eax
  40111e:	83 f8 05             	cmp    $0x5,%eax
  401121:	76 05                	jbe    401128 <phase_6+0x34>
  401123:	e8 12 03 00 00       	callq  40143a <explode_bomb>
  401128:	41 83 c4 01          	add    $0x1,%r12d
  40112c:	41 83 fc 06          	cmp    $0x6,%r12d
  401130:	74 21                	je     401153 <phase_6+0x5f>
  ...
  40114d:	49 83 c5 04          	add    $0x4,%r13
  401151:	eb c1                	jmp    401114 <phase_6+0x20>
```

首先会对 `$r13` 增加 `4`个字节，因为输入的是 `int`，所以这里等价于让 `$r13` 指向第二个输入的数字，然后跳转到地址 `0x401114`。

接着用 `$eax` 保留第二个数字，又跳到了第一段代码的地方，说明这里要求输入的所有数字都要 `<=6`。

通过 `$r12d` 和 `6` 比较的大小来决定是否遍历完所有的数字，接着进入到上一个循环，判断后面的数字是否和当前数字相等，这里的当前数字是第二个输入的数字，所以上面的代码总结起来的需求是要输入 `6` 个数字，每个数字需要 `<=6`，并且他们互相不相等。

然后代码会进行到下面这里

```assembly
  401153:	48 8d 74 24 18       	lea    0x18(%rsp),%rsi
  401158:	4c 89 f0             	mov    %r14,%rax
  40115b:	b9 07 00 00 00       	mov    $0x7,%ecx
  401160:	89 ca                	mov    %ecx,%edx
  401162:	2b 10                	sub    (%rax),%edx
  401164:	89 10                	mov    %edx,(%rax)
  401166:	48 83 c0 04          	add    $0x4,%rax
  40116a:	48 39 f0             	cmp    %rsi,%rax
  40116d:	75 f1                	jne    401160 <phase_6+0x6c>
```

注意到 `0x18` 在十进制下是 `24`，`$rsi` 地址在 `$rsp` 之后的 `24`个字节，刚好差了 `6` 个类型是 `int` 的输入。

这里会遍历输入的每一个数字，首先通过 `7-num` 获得结果，保存在 `$edx` 中，然后 `mov %edx,(%rax)` 将结果放回 `$rax` 指向的内存地址，接着 `add $0x4,%rax` 遍历下一个数字，所以整体流程类似于下面的代码

```c
int a[6] {};
for (int i; i < 6; ++i) {
  a[i] = 7 - a[i];
}
```

继续运行，会进入到下面的代码

```assembly
  401197:	8b 0c 34             	mov    (%rsp,%rsi,1),%ecx
  40119a:	83 f9 01             	cmp    $0x1,%ecx
  40119d:	7e e4                	jle    401183 <phase_6+0x8f>
  40119f:	b8 01 00 00 00       	mov    $0x1,%eax
  4011a4:	ba d0 32 60 00       	mov    $0x6032d0,%edx
  4011a9:	eb cb                	jmp    401176 <phase_6+0x82>
```

`$rsi` 初始为 `0`，`mov (%rsp,%rsi,1),%ecx` 将第一个数字放到 `$ecx` 中，然后和 `1` 进行比较，如果 `<=1`，进入地址 `0x401183` 这个位置，否则往后执行，会进入 `0x401176` 这个地址。

注意到`mov $0x6032d0,%edx` 将地址放到寄存器 `$edx` 中，可以通过 `x $edx` 查看其中的内容如下 `0x6032d0 <node1>: 0x0000014c`，不过目前我们还不知道这个数据结构的含义。

继续运行到地址是 `0x401176` 的代码如下

```assembly
  401176:	48 8b 52 08          	mov    0x8(%rdx),%rdx
  40117a:	83 c0 01             	add    $0x1,%eax
  40117d:	39 c8                	cmp    %ecx,%eax
  40117f:	75 f5                	jne    401176 <phase_6+0x82>
  401181:	eb 05                	jmp    401188 <phase_6+0x94>
  401183:	ba d0 32 60 00       	mov    $0x6032d0,%edx
  401188:	48 89 54 74 20       	mov    %rdx,0x20(%rsp,%rsi,2)
```

`$rdx` 表示 `node1`，开始先对地址 `+8`，然后 `$eax+1` 和 `$ecx` 比较大小，如果小于 `$ecx` 则继续对 `node1` 地址 `+8`，而`$ecx==6` 表示输入的 `6` 个数字，先从第一个数字开始取。

通过 `x/24 $rdx` 可以看到下面的结果

```javascript
0x6032d0 <node1>:       0x0000014c      0x00000001      0x006032e0      0x00000000
0x6032e0 <node2>:       0x000000a8      0x00000002      0x006032f0      0x00000000
0x6032f0 <node3>:       0x0000039c      0x00000003      0x00603300      0x00000000
0x603300 <node4>:       0x000002b3      0x00000004      0x00603310      0x00000000
0x603310 <node5>:       0x000001dd      0x00000005      0x00603320      0x00000000
0x603320 <node6>:       0x000001bb      0x00000006      0x00000000      0x00000000
```

注意到 `node1` 的第三个数字即为 `node2` 的开始地址，说明这是一个 `linked list`，而第一个数字标注节点的 `val`，比如  `node1` 里面的 `val` 是 `332`。

总结起来就是 `linked list` 初始地址为 `0x6032d0`，用户输入的 `6` 个数字，首先用 `7-`，然后遍历每个数字，数字决定了 `linked list` 向后移动的次数，最后将每次取到的 `node` 依次放到以 `0x20(%rsp)` 为起始地址的内存中。

接着会进行下面这段代码

```assembly
  4011ab:	48 8b 5c 24 20       	mov    0x20(%rsp),%rbx
  4011b0:	48 8d 44 24 28       	lea    0x28(%rsp),%rax
  4011b5:	48 8d 74 24 50       	lea    0x50(%rsp),%rsi
  4011ba:	48 89 d9             	mov    %rbx,%rcx
  4011bd:	48 8b 10             	mov    (%rax),%rdx
  4011c0:	48 89 51 08          	mov    %rdx,0x8(%rcx)
  4011c4:	48 83 c0 08          	add    $0x8,%rax
  4011c8:	48 39 f0             	cmp    %rsi,%rax
  4011cb:	74 05                	je     4011d2 <phase_6+0xde>
  4011cd:	48 89 d1             	mov    %rdx,%rcx
  4011d0:	eb eb                	jmp    4011bd <phase_6+0xc9>
```

这段代码是修改每个 `node` 指向的内容，重新建立新的 `linked list`，让取到的前一个 `node` 指向取到的下一个 `node` 的地址，可以通过 `mov  %rdx,0x8(%rcx)` 看出来，同时 `add $0x8,%rax` 即为不断的向后遍历 `linked list`。

最后会运行到下面这段代码

```assembly
  4011d2:	48 c7 42 08 00 00 00 	movq   $0x0,0x8(%rdx)
  4011d9:	00 
  4011da:	bd 05 00 00 00       	mov    $0x5,%ebp
  4011df:	48 8b 43 08          	mov    0x8(%rbx),%rax
  4011e3:	8b 00                	mov    (%rax),%eax
  4011e5:	39 03                	cmp    %eax,(%rbx)
  4011e7:	7d 05                	jge    4011ee <phase_6+0xfa>
  4011e9:	e8 4c 02 00 00       	callq  40143a <explode_bomb>
```

这段代码最后决定了是否会引爆炸弹，`$rbx` 是第一次取到的 `node`，`0x8(%rbx)` 是第二次取到的 `node`，这里的比较需要前一次取到的 `nodel` 里面的值比第二次取到的 `node` 中的值大，否则炸弹爆炸。

看到这里，要求已经很清楚了，我们需要输入 `6` 个数字，每个数字 `7-` 之后表示取 `linked list` 中的第几个 `node`，依次取出来值之后，需要降序排列，所以通过比较上面 `linked list` 不同 `node` 的值，最终需要输入的结果为 `4 3 2 1 6 5`。

