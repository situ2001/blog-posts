---
title: Writeup for bomblab
comments: true
date: 2021-10-23 17:10:00
tags: 
permalink: cb3eed4c3a95/
description: 拆炸弹...
categories: CS:APP
---

## 前言

一些前置知识：常用寄存器的用途，阅读汇编代码的能力，内存概念，C语言

## Phase 1

该阶段不难，读出大概意思就能解

```asm
0000000000400ee0 <phase_1>:
  400ee0: 48 83 ec 08           sub    $0x8,%rsp
  400ee4: be 00 24 40 00        mov    $0x402400,%esi
  400ee9: e8 4a 04 00 00        callq  401338 <strings_not_equal>
  400eee: 85 c0                 test   %eax,%eax
  400ef0: 74 05                 je     400ef7 <phase_1+0x17>
  400ef2: e8 43 05 00 00        callq  40143a <explode_bomb>
  400ef7: 48 83 c4 08           add    $0x8,%rsp
  400efb: c3                    retq   
```

先是准备好了两个参数，然后调用`strings_not_euqal` 函数，然后对比返回值，如果0就不爆炸，不是0就会爆炸。如果是正常人的话，想想就会是，输入的字符串不相等就会爆炸，相等就不爆。

看到汇编里头操作寄存器的语句，容易知道是传入两个字符串——用户输入的与预置的，进行比较。

而再观察一番，发现预置的那个字符串在`0x402400`上。我们便可以用如下命令来读取该地址上的值。

```shell
(gdb) x /s 0x402400
0x402400:       "Border relations with Canada have never been better."
```

所以答案是`Border relations with Canada have never been better.`

## Phase 2

```asm
0000000000400efc <phase_2>:
  400efc: 55                    push   %rbp
  400efd: 53                    push   %rbx
  400efe: 48 83 ec 28           sub    $0x28,%rsp
  400f02: 48 89 e6              mov    %rsp,%rsi
  400f05: e8 52 05 00 00        callq  40145c <read_six_numbers>
  400f0a: 83 3c 24 01           cmpl   $0x1,(%rsp)
  400f0e: 74 20                 je     400f30 <phase_2+0x34>
  400f10: e8 25 05 00 00        callq  40143a <explode_bomb>
  400f15: eb 19                 jmp    400f30 <phase_2+0x34>
  400f17: 8b 43 fc              mov    -0x4(%rbx),%eax
  400f1a: 01 c0                 add    %eax,%eax
  400f1c: 39 03                 cmp    %eax,(%rbx)
  400f1e: 74 05                 je     400f25 <phase_2+0x29>
  400f20: e8 15 05 00 00        callq  40143a <explode_bomb>
  400f25: 48 83 c3 04           add    $0x4,%rbx
  400f29: 48 39 eb              cmp    %rbp,%rbx
  400f2c: 75 e9                 jne    400f17 <phase_2+0x1b>
  400f2e: eb 0c                 jmp    400f3c <phase_2+0x40>
  400f30: 48 8d 5c 24 04        lea    0x4(%rsp),%rbx
  400f35: 48 8d 6c 24 18        lea    0x18(%rsp),%rbp
  400f3a: eb db                 jmp    400f17 <phase_2+0x1b>
  400f3c: 48 83 c4 28           add    $0x28,%rsp
  400f40: 5b                    pop    %rbx
  400f41: 5d                    pop    %rbp
  400f42: c3                    retq  
```

这里可以看到一个函数`read_six_numbers`，顾名思义，这个函数是用来从input里头读取六个数字。我们来分析下这个函数到底做了啥。

```asm
000000000040145c <read_six_numbers>:
  40145c: 48 83 ec 18           sub    $0x18,%rsp
  401460: 48 89 f2              mov    %rsi,%rdx
  401463: 48 8d 4e 04           lea    0x4(%rsi),%rcx
  401467: 48 8d 46 14           lea    0x14(%rsi),%rax
  40146b: 48 89 44 24 08        mov    %rax,0x8(%rsp)
  401470: 48 8d 46 10           lea    0x10(%rsi),%rax
  401474: 48 89 04 24           mov    %rax,(%rsp)
  401478: 4c 8d 4e 0c           lea    0xc(%rsi),%r9
  40147c: 4c 8d 46 08           lea    0x8(%rsi),%r8
  401480: be c3 25 40 00        mov    $0x4025c3,%esi
  401485: b8 00 00 00 00        mov    $0x0,%eax
  40148a: e8 61 f7 ff ff        callq  400bf0 <__isoc99_sscanf@plt>
  40148f: 83 f8 05              cmp    $0x5,%eax
  401492: 7f 05                 jg     401499 <read_six_numbers+0x3d>
  401494: e8 a1 ff ff ff        callq  40143a <explode_bomb>
  401499: 48 83 c4 18           add    $0x18,%rsp
  40149d: c3                    retq   
```

我们去**CppReference**查一下C语言的API

```c
int sscanf( const char *restrict buffer, const char *restrict format, ... ); // (since C99)
```

输入的格式化字符串应该在第二个参数，而上面的汇编代码可以看出，这个字符串在地址`0x4025c3`上，我们用gdb来看看

```shell
(gdb) x/s 0x4025c3
0x4025c3:       "%d %d %d %d %d %d"
```

emm果然是六个数字啊，那我们就要看看，第三到第八个参数是啥了。

首先在调用函数前，`phase_2`将栈顶指针给复制到了`%rsi`。

```asm
400f02: 48 89 e6              mov    %rsp,%rsi
```

然后在读取六个数字的函数调用sscanf之前，准备了一坨参数，可以看到偏移量都是4，妥妥的int整数

```asm
  # %rsi is %rsp
  401463: 48 8d 4e 04           lea    0x4(%rsi),%rcx
  401467: 48 8d 46 14           lea    0x14(%rsi),%rax
  40146b: 48 89 44 24 08        mov    %rax,0x8(%rsp)
  401470: 48 8d 46 10           lea    0x10(%rsi),%rax
  401474: 48 89 04 24           mov    %rax,(%rsp)
  401478: 4c 8d 4e 0c           lea    0xc(%rsi),%r9
  40147c: 4c 8d 46 08           lea    0x8(%rsi),%r8
```

对着寄存器表查了一波，`%rsi`为第一个变量的地址，`%rcx`为第二个变量的地址，`%r8`是第三个的...

那我可以猜测了——这六个数是依次从栈顶排下去的

可以直接实验一下，在爆炸前打个断点，输入`1 2 3 4 5 6`，然后看看此时栈的情况。

```shell
(gdb) x/8wx $rsp
0x7fffffffe350: 0x00000001      0x00000002      0x00000003      0x00000004
0x7fffffffe360: 0x00000005      0x00000006      0x00401431      0x00000000
```

果然如此，那我们继续看看汇编

```asm
  400f0a: 83 3c 24 01           cmpl   $0x1,(%rsp)
  400f0e: 74 20                 je     400f30 <phase_2+0x34>
  400f10: e8 25 05 00 00        callq  40143a <explode_bomb>
```

可以看出，先是看看栈顶是否为1，是就继续，否则爆炸。

然后执行下面这些语句，先把第二个元素的地址放入`%rbx`，然后`%rbp`为`%rsp + 0x18`，刚好是6个int整数的长度

```asm
  400f30: 48 8d 5c 24 04        lea    0x4(%rsp),%rbx
  400f35: 48 8d 6c 24 18        lea    0x18(%rsp),%rbp
  400f3a: eb db                 jmp    400f17 <phase_2+0x1b>
```

然后跳转到这，开始循环，可以看到，前面的这几条语句是初始化循环。

```asm
  400f17: 8b 43 fc              mov    -0x4(%rbx),%eax
  400f1a: 01 c0                 add    %eax,%eax
  400f1c: 39 03                 cmp    %eax,(%rbx)
  400f1e: 74 05                 je     400f25 <phase_2+0x29>
  400f20: e8 15 05 00 00        callq  40143a <explode_bomb>
  400f25: 48 83 c3 04           add    $0x4,%rbx
  400f29: 48 39 eb              cmp    %rbp,%rbx
  400f2c: 75 e9                 jne    400f17 <phase_2+0x1b>
  400f2e: eb 0c                 jmp    400f3c <phase_2+0x40>
```

上面这段汇编的意思是：

把第一个元素放入`%eax`，然后将其翻倍，然后与第二个元素进行比较，如果不相等，就引爆炸弹。然后`%rbx`就继续向上跑，指向第三个元素，继续与第二个元素比较，如果第三个元素是第二个的两倍就继续，否则就爆炸，......，直到`%rbx`与`$rbp`相等为止。

所以后一个元素要是前一个元素的两倍，并且第一个元素要是1。

答案便呼之欲出：`1 2 4 8 16 32`

## Phase 3

```asm
0000000000400f43 <phase_3>:
  400f43: 48 83 ec 18           sub    $0x18,%rsp
  400f47: 48 8d 4c 24 0c        lea    0xc(%rsp),%rcx
  400f4c: 48 8d 54 24 08        lea    0x8(%rsp),%rdx
  400f51: be cf 25 40 00        mov    $0x4025cf,%esi
  400f56: b8 00 00 00 00        mov    $0x0,%eax
  400f5b: e8 90 fc ff ff        callq  400bf0 <__isoc99_sscanf@plt>
  400f60: 83 f8 01              cmp    $0x1,%eax
  400f63: 7f 05                 jg     400f6a <phase_3+0x27>
  400f65: e8 d0 04 00 00        callq  40143a <explode_bomb>
  400f6a: 83 7c 24 08 07        cmpl   $0x7,0x8(%rsp)
  400f6f: 77 3c                 ja     400fad <phase_3+0x6a>
  400f71: 8b 44 24 08           mov    0x8(%rsp),%eax
  400f75: ff 24 c5 70 24 40 00  jmpq   *0x402470(,%rax,8)
  400f7c: b8 cf 00 00 00        mov    $0xcf,%eax
  400f81: eb 3b                 jmp    400fbe <phase_3+0x7b>
  400f83: b8 c3 02 00 00        mov    $0x2c3,%eax
  400f88: eb 34                 jmp    400fbe <phase_3+0x7b>
  400f8a: b8 00 01 00 00        mov    $0x100,%eax
  400f8f: eb 2d                 jmp    400fbe <phase_3+0x7b>
  400f91: b8 85 01 00 00        mov    $0x185,%eax
  400f96: eb 26                 jmp    400fbe <phase_3+0x7b>
  400f98: b8 ce 00 00 00        mov    $0xce,%eax
  400f9d: eb 1f                 jmp    400fbe <phase_3+0x7b>
  400f9f: b8 aa 02 00 00        mov    $0x2aa,%eax
  400fa4: eb 18                 jmp    400fbe <phase_3+0x7b>
  400fa6: b8 47 01 00 00        mov    $0x147,%eax
  400fab: eb 11                 jmp    400fbe <phase_3+0x7b>
  400fad: e8 88 04 00 00        callq  40143a <explode_bomb>
  400fb2: b8 00 00 00 00        mov    $0x0,%eax
  400fb7: eb 05                 jmp    400fbe <phase_3+0x7b>
  400fb9: b8 37 01 00 00        mov    $0x137,%eax
  400fbe: 3b 44 24 0c           cmp    0xc(%rsp),%eax
  400fc2: 74 05                 je     400fc9 <phase_3+0x86>
  400fc4: e8 71 04 00 00        callq  40143a <explode_bomb>
  400fc9: 48 83 c4 18           add    $0x18,%rsp
  400fcd: c3                    retq
```

关键词：switch语句汇编

我们把这汇编拆开几部分来分析，首先很容易看出这个

```asm
400f5b: e8 90 fc ff ff        callq  400bf0 <__isoc99_sscanf@plt>
```

很明显是做输入了，我们去**CppReference**查一下C语言的API，~~哦不用了刚刚已经查过了~~

再看看开头这一段汇编，这是为`sscanf`函数准备参数的过程。

```asm
0000000000400f43 <phase_3>:
  400f43: 48 83 ec 18           sub    $0x18,%rsp
  400f47: 48 8d 4c 24 0c        lea    0xc(%rsp),%rcx
  400f4c: 48 8d 54 24 08        lea    0x8(%rsp),%rdx
  400f51: be cf 25 40 00        mov    $0x4025cf,%esi
  400f56: b8 00 00 00 00        mov    $0x0,%eax
```

参数1在main函数的时候已经存到`%rdi`了。参数2是格式字符串，我们可以通过调试命令`x/s 0x4025cf` 得到`"%d %d"` 。参数3及以后便是变量的地址了，这里参数3 4通过lea计算栈地址，并传入相应的寄存器中。这里我们把这两个参数称之为：第一个数字，第二个数字（

有一说一，~~我之前差点把这个函数当成普通的`scanf`了~~

接着便是比较返回值，如果成功接受的参数数目为2，那么就会跳过爆炸函数的调用。但是如果第一个参数大于7就会爆炸。

```asm
400f60: 83 f8 01              cmp    $0x1,%eax
400f63: 7f 05                 jg     400f6a <phase_3+0x27>
400f65: e8 d0 04 00 00        callq  40143a <explode_bomb>
400f6a: 83 7c 24 08 07        cmpl   $0x7,0x8(%rsp)
400f6f: 77 3c                 ja     400fad <phase_3+0x6a>
```

接着，语句`mov  0x8(%rsp),%eax`便是把arg1给存到%eax里头。紧接着就出现这条语句

```asm
400f71: 8b 44 24 08           mov    0x8(%rsp),%eax
400f75: ff 24 c5 70 24 40 00  jmpq   *0x402470(,%rax,8)
```

`0x400f71`是把第一个参数放入寄存器`%eax`里头

而`0x400f75`看起来就是跳转至该内存区域中存放的地址，那么同理，我们可以用gdb获取到对应地址的值。

```shell
(gdb) x/x 0x402470
0x402470:       0x00400f7c
```

`%rax`的值可以是0~7，这意味着我们要对内存进行偏移，所以我们可以使用命令`x/16x [addr]`，得出所有的displacement后的地址

```shell
(gdb) x/16x 0x402470
0x402470:       0x00400f7c      0x00000000      0x00400fb9      0x00000000
0x402480:       0x00400f83      0x00000000      0x00400f8a      0x00000000
0x402490:       0x00400f91      0x00000000      0x00400f98      0x00000000
0x4024a0:       0x00400f9f      0x00000000      0x00400fa6      0x00000000
```

结合下面的这部分汇编，我们就可以看出这是一个switch语句，根据刚刚得到的信息，那么我们就可以进行标注。

```asm
  # x = 0
  400f7c: b8 cf 00 00 00        mov    $0xcf,%eax
  400f81: eb 3b                 jmp    400fbe <phase_3+0x7b>
  # x = 2
  400f83: b8 c3 02 00 00        mov    $0x2c3,%eax
  400f88: eb 34                 jmp    400fbe <phase_3+0x7b>
  # x = 3
  400f8a: b8 00 01 00 00        mov    $0x100,%eax
  400f8f: eb 2d                 jmp    400fbe <phase_3+0x7b>
  # x = 4
  400f91: b8 85 01 00 00        mov    $0x185,%eax
  400f96: eb 26                 jmp    400fbe <phase_3+0x7b>
  # x = 5
  400f98: b8 ce 00 00 00        mov    $0xce,%eax
  400f9d: eb 1f                 jmp    400fbe <phase_3+0x7b>
  # x = 6
  400f9f: b8 aa 02 00 00        mov    $0x2aa,%eax
  400fa4: eb 18                 jmp    400fbe <phase_3+0x7b>
  # x = 7
  400fa6: b8 47 01 00 00        mov    $0x147,%eax
  400fab: eb 11                 jmp    400fbe <phase_3+0x7b>
  # default
  400fad: e8 88 04 00 00        callq  40143a <explode_bomb>
  400fb2: b8 00 00 00 00        mov    $0x0,%eax
  400fb7: eb 05                 jmp    400fbe <phase_3+0x7b>
  # x = 1
  400fb9: b8 37 01 00 00        mov    $0x137,%eax
  # switch ends
  400fbe: 3b 44 24 0c           cmp    0xc(%rsp),%eax
  400fc2: 74 05                 je     400fc9 <phase_3+0x86>
  400fc4: e8 71 04 00 00        callq  40143a <explode_bomb>
  400fc9: 48 83 c4 18           add    $0x18,%rsp
  400fcd: c3                    retq
```

不过呢，所有case执行过后都把对应的数复制到%eax上，然后跳转到`400fbe` 上，**此时的%eax**便会与输入的第二个数字进行比较，如果相等则成功解除本阶段的炸弹。

所以，我们可以得出其中一个答案为： `1 311`

## Phase 4

```asm
000000000040100c <phase_4>:
  40100c: 48 83 ec 18           sub    $0x18,%rsp
  401010: 48 8d 4c 24 0c        lea    0xc(%rsp),%rcx
  401015: 48 8d 54 24 08        lea    0x8(%rsp),%rdx
  40101a: be cf 25 40 00        mov    $0x4025cf,%esi
  40101f: b8 00 00 00 00        mov    $0x0,%eax
  401024: e8 c7 fb ff ff        callq  400bf0 <__isoc99_sscanf@plt>
  401029: 83 f8 02              cmp    $0x2,%eax
  40102c: 75 07                 jne    401035 <phase_4+0x29>
  40102e: 83 7c 24 08 0e        cmpl   $0xe,0x8(%rsp)
  401033: 76 05                 jbe    40103a <phase_4+0x2e>
  401035: e8 00 04 00 00        callq  40143a <explode_bomb>
  40103a: ba 0e 00 00 00        mov    $0xe,%edx
  40103f: be 00 00 00 00        mov    $0x0,%esi
  401044: 8b 7c 24 08           mov    0x8(%rsp),%edi
  401048: e8 81 ff ff ff        callq  400fce <func4>
  40104d: 85 c0                 test   %eax,%eax
  40104f: 75 07                 jne    401058 <phase_4+0x4c>
  401051: 83 7c 24 0c 00        cmpl   $0x0,0xc(%rsp)
  401056: 74 05                 je     40105d <phase_4+0x51>
  401058: e8 dd 03 00 00        callq  40143a <explode_bomb>
  40105d: 48 83 c4 18           add    $0x18,%rsp
  401061: c3                    retq   
```

递归函数的逆向，看起来，前面这部分依旧是读取两个数字，如果没读齐两个数字，或者第一个数字没有小于等于14的话，炸弹就会爆炸。

```asm
000000000040100c <phase_4>:
  40100c: 48 83 ec 18           sub    $0x18,%rsp
  401010: 48 8d 4c 24 0c        lea    0xc(%rsp),%rcx
  401015: 48 8d 54 24 08        lea    0x8(%rsp),%rdx
  40101a: be cf 25 40 00        mov    $0x4025cf,%esi
  40101f: b8 00 00 00 00        mov    $0x0,%eax
  401024: e8 c7 fb ff ff        callq  400bf0 <__isoc99_sscanf@plt>
  401029: 83 f8 02              cmp    $0x2,%eax
  40102c: 75 07                 jne    401035 <phase_4+0x29>
  40102e: 83 7c 24 08 0e        cmpl   $0xe,0x8(%rsp)
  401033: 76 05                 jbe    40103a <phase_4+0x2e>
  401035: e8 00 04 00 00        callq  40143a <explode_bomb>
```

后半部分是构造好参数到寄存器里头，然后调用函数`func4`，接着比较返回值，如果返回值不为0就会爆炸。如果第二个数字不为0的话，炸弹也会爆炸。

```asm
  40103a: ba 0e 00 00 00        mov    $0xe,%edx
  40103f: be 00 00 00 00        mov    $0x0,%esi
  401044: 8b 7c 24 08           mov    0x8(%rsp),%edi
  401048: e8 81 ff ff ff        callq  400fce <func4>
  40104d: 85 c0                 test   %eax,%eax
  40104f: 75 07                 jne    401058 <phase_4+0x4c>
  401051: 83 7c 24 0c 00        cmpl   $0x0,0xc(%rsp)
  401056: 74 05                 je     40105d <phase_4+0x51>
  401058: e8 dd 03 00 00        callq  40143a <explode_bomb>
  40105d: 48 83 c4 18           add    $0x18,%rsp
  401061: c3                    retq   
```

我们看到函数调用的另外一个函数`func4`

```asm
0000000000400fce <func4>:
  400fce: 48 83 ec 08           sub    $0x8,%rsp
  400fd2: 89 d0                 mov    %edx,%eax
  400fd4: 29 f0                 sub    %esi,%eax
  400fd6: 89 c1                 mov    %eax,%ecx
  400fd8: c1 e9 1f              shr    $0x1f,%ecx
  400fdb: 01 c8                 add    %ecx,%eax
  400fdd: d1 f8                 sar    %eax
  400fdf: 8d 0c 30              lea    (%rax,%rsi,1),%ecx
  400fe2: 39 f9                 cmp    %edi,%ecx
  400fe4: 7e 0c                 jle    400ff2 <func4+0x24>
  400fe6: 8d 51 ff              lea    -0x1(%rcx),%edx
  400fe9: e8 e0 ff ff ff        callq  400fce <func4>
  400fee: 01 c0                 add    %eax,%eax
  400ff0: eb 15                 jmp    401007 <func4+0x39>
  400ff2: b8 00 00 00 00        mov    $0x0,%eax
  400ff7: 39 f9                 cmp    %edi,%ecx
  400ff9: 7d 0c                 jge    401007 <func4+0x39>
  400ffb: 8d 71 01              lea    0x1(%rcx),%esi
  400ffe: e8 cb ff ff ff        callq  400fce <func4>
  401003: 8d 44 00 01           lea    0x1(%rax,%rax,1),%eax
  401007: 48 83 c4 08           add    $0x8,%rsp
  40100b: c3                    retq
```

这个函数看起来是先对做了一定的运算，然后进行比较判断，最后再递归调用自身

由于涉及到了递归调用，增加了~~边注释边脑补的~~难度，所以我们可以逆着写出对应的C代码

```c
int f4(int x, int y, int z)
{
    int t = z - y;
    unsigned tmp = t >> 31;
    t += tmp;
    t >>= 1;
    int sum = t + y;
    if (sum - x == 0)
    {
        return 0;
    }
    if (sum - x > 0)
    {
        return 2 * f4(x, y, sum - 1);
    }
    if (sum - x < 0)
    {
        return 2 * f4(x, sum + 1, z) + 1;
    }
}
```

~~最后我们可以用循环，把答案给搞出来~~

```c
int main()
{
    for (int i = 0; i <= 14; i++)
    {
        if (f4(i, 0, 14) == 0)
        {
            printf("%d ", i);
        }
    }
}
```

最后得出 `0 1 3 7` ，然后又因为第二个数必须为`0`，所以答案可以是 `7 0`

## Phase 5

```asm
0000000000401062 <phase_5>:
  401062: 53                    push   %rbx
  401063: 48 83 ec 20           sub    $0x20,%rsp
  401067: 48 89 fb              mov    %rdi,%rbx
  40106a: 64 48 8b 04 25 28 00  mov    %fs:0x28,%rax
  401071: 00 00 
  401073: 48 89 44 24 18        mov    %rax,0x18(%rsp)
  401078: 31 c0                 xor    %eax,%eax
  40107a: e8 9c 02 00 00        callq  40131b <string_length>
  40107f: 83 f8 06              cmp    $0x6,%eax
  401082: 74 4e                 je     4010d2 <phase_5+0x70>
  401084: e8 b1 03 00 00        callq  40143a <explode_bomb>
  401089: eb 47                 jmp    4010d2 <phase_5+0x70>
  40108b: 0f b6 0c 03           movzbl (%rbx,%rax,1),%ecx
  40108f: 88 0c 24              mov    %cl,(%rsp)
  401092: 48 8b 14 24           mov    (%rsp),%rdx
  401096: 83 e2 0f              and    $0xf,%edx
  401099: 0f b6 92 b0 24 40 00  movzbl 0x4024b0(%rdx),%edx
  4010a0: 88 54 04 10           mov    %dl,0x10(%rsp,%rax,1)
  4010a4: 48 83 c0 01           add    $0x1,%rax
  4010a8: 48 83 f8 06           cmp    $0x6,%rax
  4010ac: 75 dd                 jne    40108b <phase_5+0x29>
  4010ae: c6 44 24 16 00        movb   $0x0,0x16(%rsp)
  4010b3: be 5e 24 40 00        mov    $0x40245e,%esi
  4010b8: 48 8d 7c 24 10        lea    0x10(%rsp),%rdi
  4010bd: e8 76 02 00 00        callq  401338 <strings_not_equal>
  4010c2: 85 c0                 test   %eax,%eax
  4010c4: 74 13                 je     4010d9 <phase_5+0x77>
  4010c6: e8 6f 03 00 00        callq  40143a <explode_bomb>
  4010cb: 0f 1f 44 00 00        nopl   0x0(%rax,%rax,1)
  4010d0: eb 07                 jmp    4010d9 <phase_5+0x77>
  4010d2: b8 00 00 00 00        mov    $0x0,%eax
  4010d7: eb b2                 jmp    40108b <phase_5+0x29>
  4010d9: 48 8b 44 24 18        mov    0x18(%rsp),%rax
  4010de: 64 48 33 04 25 28 00  xor    %fs:0x28,%rax
  4010e5: 00 00 
  4010e7: 74 05                 je     4010ee <phase_5+0x8c>
  4010e9: e8 42 fa ff ff        callq  400b30 <__stack_chk_fail@plt>
  4010ee: 48 83 c4 20           add    $0x20,%rsp
  4010f2: 5b                    pop    %rbx
  4010f3: c3                    retq   
```

这个phase还行，主要考查对指针算术，位运算等的了解

先看看开头

```asm
0000000000401062 <phase_5>:
  401062: 53                    push   %rbx
  401063: 48 83 ec 20           sub    $0x20,%rsp
  401067: 48 89 fb              mov    %rdi,%rbx
  40106a: 64 48 8b 04 25 28 00  mov    %fs:0x28,%rax
  401071: 00 00 
  401073: 48 89 44 24 18        mov    %rax,0x18(%rsp)
  401078: 31 c0                 xor    %eax,%eax
  40107a: e8 9c 02 00 00        callq  40131b <string_length>
  40107f: 83 f8 06              cmp    $0x6,%eax
  401082: 74 4e                 je     4010d2 <phase_5+0x70>
  401084: e8 b1 03 00 00        callq  40143a <explode_bomb>
  401089: eb 47                 jmp    4010d2 <phase_5+0x70>
```

开头这段汇编，比对输入的字符串的长度，如果长度不为5，就会爆炸。这里有一个`%fs:0x28`，只是在后面用来检查栈的而已，这里忽略即可。

如果不爆炸的话，就会去到这里

```asm
  4010d2: b8 00 00 00 00        mov    $0x0,%eax
  4010d7: eb b2                 jmp    40108b <phase_5+0x29>
```

然而这里继续跳转，跳转到下面这段代码，看起来是循环代码，如果条件没达到就循环执行，如果达到了条件就会执行`4010ac`下一句`4010ae`，这段汇编等下贴

```asm
  40108b: 0f b6 0c 03           movzbl (%rbx,%rax,1),%ecx
  40108f: 88 0c 24              mov    %cl,(%rsp)
  401092: 48 8b 14 24           mov    (%rsp),%rdx
  401096: 83 e2 0f              and    $0xf,%edx
  401099: 0f b6 92 b0 24 40 00  movzbl 0x4024b0(%rdx),%edx
  4010a0: 88 54 04 10           mov    %dl,0x10(%rsp,%rax,1)
  4010a4: 48 83 c0 01           add    $0x1,%rax
  4010a8: 48 83 f8 06           cmp    $0x6,%rax
  4010ac: 75 dd                 jne    40108b <phase_5+0x29>
```

那么这段循环代码做了什么呢？我们分析一下，最后写出了如下伪代码

```text
char* input = your input;
for (int i = 0; i < 6; i++) {
  1. retrieve i-th char from input as ch
  2. push ch into stack
  3. mov ch to %rdx
  4. update %rdx with %rdx & 0xf (masking, only retain low 4 bits of ch)
  5. mov the ch on address (0x4024b0 + %rdx) to stack address (%rsp + i) (below stack.top())
}
```

每一次循环，都从`input`里取出第`i`个字符，使其与`0xf`做位运算，产生的结果作为偏移量`offset`，接着取出`0x4024b0 + offset`处的字符，放到栈里。

那么，`0x4024b0`处肯定是有什么东西的吧？试试就知道了，不试不知道，一试，发现是一个数组。

```shell
(gdb) x/s 0x4024b0
0x4024b0 <array.3449>:  "maduiersnfotvbylSo you think you can stop the bomb with ctrl-c, do you?"
```

又刚刚提到了，上面的循环跳出之后，就会执行`4010ac`下一句`4010ae`

```asm
  4010ae: c6 44 24 16 00        movb   $0x0,0x16(%rsp)
  4010b3: be 5e 24 40 00        mov    $0x40245e,%esi
  4010b8: 48 8d 7c 24 10        lea    0x10(%rsp),%rdi
  4010bd: e8 76 02 00 00        callq  401338 <strings_not_equal>
  4010c2: 85 c0                 test   %eax,%eax
  4010c4: 74 13                 je     4010d9 <phase_5+0x77>
  4010c6: e8 6f 03 00 00        callq  40143a <explode_bomb>
  4010cb: 0f 1f 44 00 00        nopl   0x0(%rax,%rax,1)
  4010d0: eb 07                 jmp    4010d9 <phase_5+0x77>
```

那么那一段汇编做了什么工作呢？（注：这里最后一句jmp到结束部分，没有产生什么影响）

看起来是比较两个字符串是否相等，使用函数`string_not_equal`。如果不一样就爆炸，一样就解除该阶段。

同理，我们发现传参的时候，把栈中字符串的指针传进寄存器，准备好了其中一个参数。而另一个参数在准备时，读取了`0x40245e`的值，好，直接用gdb调试看看。

```shell
(gdb) x/s 0x40245e
0x40245e:       "flyers"
```

看来我们需要从上面那一大坨字符串中，构造出与flyers一样的字符串。既然刚刚已经说过怎么从这坨字符串中获取字符了。

那我们先把目标字符的index算出来，分别是`9 15 14 5 6 7`

于是，我们只需构造低4位分别为上述index的字符就行了，经过一番的查ASCII表，我们容易得出其中一个答案为`yon567`

## Phase 6

```asm
00000000004010f4 <phase_6>:
  4010f4: 41 56                 push   %r14
  4010f6: 41 55                 push   %r13
  4010f8: 41 54                 push   %r12
  4010fa: 55                    push   %rbp
  4010fb: 53                    push   %rbx
  4010fc: 48 83 ec 50           sub    $0x50,%rsp
  401100: 49 89 e5              mov    %rsp,%r13
  401103: 48 89 e6              mov    %rsp,%rsi
  401106: e8 51 03 00 00        callq  40145c <read_six_numbers>
  40110b: 49 89 e6              mov    %rsp,%r14
  40110e: 41 bc 00 00 00 00     mov    $0x0,%r12d
  401114: 4c 89 ed              mov    %r13,%rbp
  401117: 41 8b 45 00           mov    0x0(%r13),%eax
  40111b: 83 e8 01              sub    $0x1,%eax
  40111e: 83 f8 05              cmp    $0x5,%eax
  401121: 76 05                 jbe    401128 <phase_6+0x34>
  401123: e8 12 03 00 00        callq  40143a <explode_bomb>
  401128: 41 83 c4 01           add    $0x1,%r12d
  40112c: 41 83 fc 06           cmp    $0x6,%r12d
  401130: 74 21                 je     401153 <phase_6+0x5f>
  401132: 44 89 e3              mov    %r12d,%ebx
  401135: 48 63 c3              movslq %ebx,%rax
  401138: 8b 04 84              mov    (%rsp,%rax,4),%eax
  40113b: 39 45 00              cmp    %eax,0x0(%rbp)
  40113e: 75 05                 jne    401145 <phase_6+0x51>
  401140: e8 f5 02 00 00        callq  40143a <explode_bomb>
  401145: 83 c3 01              add    $0x1,%ebx
  401148: 83 fb 05              cmp    $0x5,%ebx
  40114b: 7e e8                 jle    401135 <phase_6+0x41>
  40114d: 49 83 c5 04           add    $0x4,%r13
  401151: eb c1                 jmp    401114 <phase_6+0x20>
  401153: 48 8d 74 24 18        lea    0x18(%rsp),%rsi
  401158: 4c 89 f0              mov    %r14,%rax
  40115b: b9 07 00 00 00        mov    $0x7,%ecx
  401160: 89 ca                 mov    %ecx,%edx
  401162: 2b 10                 sub    (%rax),%edx
  401164: 89 10                 mov    %edx,(%rax)
  401166: 48 83 c0 04           add    $0x4,%rax
  40116a: 48 39 f0              cmp    %rsi,%rax
  40116d: 75 f1                 jne    401160 <phase_6+0x6c>
  40116f: be 00 00 00 00        mov    $0x0,%esi
  401174: eb 21                 jmp    401197 <phase_6+0xa3>
  401176: 48 8b 52 08           mov    0x8(%rdx),%rdx
  40117a: 83 c0 01              add    $0x1,%eax
  40117d: 39 c8                 cmp    %ecx,%eax
  40117f: 75 f5                 jne    401176 <phase_6+0x82>
  401181: eb 05                 jmp    401188 <phase_6+0x94>
  401183: ba d0 32 60 00        mov    $0x6032d0,%edx
  401188: 48 89 54 74 20        mov    %rdx,0x20(%rsp,%rsi,2)
  40118d: 48 83 c6 04           add    $0x4,%rsi
  401191: 48 83 fe 18           cmp    $0x18,%rsi
  401195: 74 14                 je     4011ab <phase_6+0xb7>
  401197: 8b 0c 34              mov    (%rsp,%rsi,1),%ecx
  40119a: 83 f9 01              cmp    $0x1,%ecx
  40119d: 7e e4                 jle    401183 <phase_6+0x8f>
  40119f: b8 01 00 00 00        mov    $0x1,%eax
  4011a4: ba d0 32 60 00        mov    $0x6032d0,%edx
  4011a9: eb cb                 jmp    401176 <phase_6+0x82>
  4011ab: 48 8b 5c 24 20        mov    0x20(%rsp),%rbx
  4011b0: 48 8d 44 24 28        lea    0x28(%rsp),%rax
  4011b5: 48 8d 74 24 50        lea    0x50(%rsp),%rsi
  4011ba: 48 89 d9              mov    %rbx,%rcx
  4011bd: 48 8b 10              mov    (%rax),%rdx
  4011c0: 48 89 51 08           mov    %rdx,0x8(%rcx)
  4011c4: 48 83 c0 08           add    $0x8,%rax
  4011c8: 48 39 f0              cmp    %rsi,%rax
  4011cb: 74 05                 je     4011d2 <phase_6+0xde>
  4011cd: 48 89 d1              mov    %rdx,%rcx
  4011d0: eb eb                 jmp    4011bd <phase_6+0xc9>
  4011d2: 48 c7 42 08 00 00 00  movq   $0x0,0x8(%rdx)
  4011d9: 00 
  4011da: bd 05 00 00 00        mov    $0x5,%ebp
  4011df: 48 8b 43 08           mov    0x8(%rbx),%rax
  4011e3: 8b 00                 mov    (%rax),%eax
  4011e5: 39 03                 cmp    %eax,(%rbx)
  4011e7: 7d 05                 jge    4011ee <phase_6+0xfa>
  4011e9: e8 4c 02 00 00        callq  40143a <explode_bomb>
  4011ee: 48 8b 5b 08           mov    0x8(%rbx),%rbx
  4011f2: 83 ed 01              sub    $0x1,%ebp
  4011f5: 75 e8                 jne    4011df <phase_6+0xeb>
  4011f7: 48 83 c4 50           add    $0x50,%rsp
  4011fb: 5b                    pop    %rbx
  4011fc: 5d                    pop    %rbp
  4011fd: 41 5c                 pop    %r12
  4011ff: 41 5d                 pop    %r13
  401201: 41 5e                 pop    %r14
  401203: c3                    retq
```

这个可以说是最难phase了吧？这个前前后后花了我快三个钟...做完之后

好家伙，涉及的步骤是真的多——有六个数字的输入，对这些数字进行映射，链表与结构体，有关链表部分涉及到读取节点的值，甚至还有节点重连

近100行的汇编...我觉得还是得一部分一部分地分割开来，单独进行分析

首先还是常见的输入六个数字，从前往后扫描，自顶向下地保存到栈里。

```asm
  4010fc: 48 83 ec 50           sub    $0x50,%rsp
  401100: 49 89 e5              mov    %rsp,%r13
  401103: 48 89 e6              mov    %rsp,%rsi
  401106: e8 51 03 00 00        callq  40145c <read_six_numbers>
  40110b: 49 89 e6              mov    %rsp,%r14
```

然后便会碰到一个嵌套的循环

```asm
  40110e: 41 bc 00 00 00 00     mov    $0x0,%r12d
  401114: 4c 89 ed              mov    %r13,%rbp
  401117: 41 8b 45 00           mov    0x0(%r13),%eax
  40111b: 83 e8 01              sub    $0x1,%eax
  40111e: 83 f8 05              cmp    $0x5,%eax
  401121: 76 05                 jbe    401128 <phase_6+0x34>
  401123: e8 12 03 00 00        callq  40143a <explode_bomb>
  401128: 41 83 c4 01           add    $0x1,%r12d
  40112c: 41 83 fc 06           cmp    $0x6,%r12d
  401130: 74 21                 je     401153 <phase_6+0x5f>
  401132: 44 89 e3              mov    %r12d,%ebx
  401135: 48 63 c3              movslq %ebx,%rax
  401138: 8b 04 84              mov    (%rsp,%rax,4),%eax
  40113b: 39 45 00              cmp    %eax,0x0(%rbp)
  40113e: 75 05                 jne    401145 <phase_6+0x51>
  401140: e8 f5 02 00 00        callq  40143a <explode_bomb>
  401145: 83 c3 01              add    $0x1,%ebx
  401148: 83 fb 05              cmp    $0x5,%ebx
  40114b: 7e e8                 jle    401135 <phase_6+0x41>
  40114d: 49 83 c5 04           add    $0x4,%r13
  401151: eb c1                 jmp    401114 <phase_6+0x20>
```

这个循环用了两个循环进行数字之间的逐一比对

外循环初始化为`i = 0`

```asm
  40110e: 41 bc 00 00 00 00     mov    $0x0,%r12d
```

首先先检查了当前第i个数字是否满足大于0且小于等于6，注意第i个数字被保存到了`%rbp`上

```asm
  401114: 4c 89 ed              mov    %r13,%rbp
  401117: 41 8b 45 00           mov    0x0(%r13),%eax
  40111b: 83 e8 01              sub    $0x1,%eax
  40111e: 83 f8 05              cmp    $0x5,%eax
  401121: 76 05                 jbe    401128 <phase_6+0x34>
  401123: e8 12 03 00 00        callq  40143a <explode_bomb>

```

然后检查一下i是否超出6，如超出则跳出循环

```asm
  401128: 41 83 c4 01           add    $0x1,%r12d
  40112c: 41 83 fc 06           cmp    $0x6,%r12d
  401130: 74 21                 je     401153 <phase_6+0x5f>
```

否则，便使得`j = i + 1`，进入内循环

```asm
  401132: 44 89 e3              mov    %r12d,%ebx
  401135: 48 63 c3              movslq %ebx,%rax
  401138: 8b 04 84              mov    (%rsp,%rax,4),%eax
  40113b: 39 45 00              cmp    %eax,0x0(%rbp)
  40113e: 75 05                 jne    401145 <phase_6+0x51>
  401140: e8 f5 02 00 00        callq  40143a <explode_bomb>
  401145: 83 c3 01              add    $0x1,%ebx
  401148: 83 fb 05              cmp    $0x5,%ebx
  40114b: 7e e8                 jle    401135 <phase_6+0x41>
```

从上可以看出，内循环就是，通过displacement寻址，把第j个元素给放入`%eax`，然后与`%rbp`即第i个数字进行比较。如果相等就会爆炸。如果j超过5的话，就会跳出跳出内循环。

最后外循环进行自增和跳转

```asm
  40114d: 49 83 c5 04           add    $0x4,%r13 # rbp + 4
  401151: eb c1                 jmp    401114 <phase_6+0x20>
```

整理一下，验证的条件有：

1. 所有数字必须大于0（**使用了`jbe`**进行比较）且小于等于6
2. 每个数字之间不能相等

大致的伪代码如下

```c
int n[6] = {...};
for (int i = 0; i < 6; i++) {
  if (n[i] > 6 && n[i] <= 0) {
    boom();
  }
  for (int j = i + 1; j < 6; j++) {
    if (n[j] == n[i]) {
      boom();
    }
  }
}
// pass
```

这还是第一步...我们继续看看第二步，依旧是个循环

```asm
  401153: 48 8d 74 24 18        lea    0x18(%rsp),%rsi
  401158: 4c 89 f0              mov    %r14,%rax
  40115b: b9 07 00 00 00        mov    $0x7,%ecx
  401160: 89 ca                 mov    %ecx,%edx
  401162: 2b 10                 sub    (%rax),%edx
  401164: 89 10                 mov    %edx,(%rax)
  401166: 48 83 c0 04           add    $0x4,%rax
  40116a: 48 39 f0              cmp    %rsi,%rax
  40116d: 75 f1                 jne    401160 <phase_6+0x6c>
```

先是把`0x18(%rsp)`到`%rsi`上（用于在`40116a`处做比较判断循环是否终止），把栈顶指针复制一份给`%r14`，接着把直接数7给了`%ecx`，再临时把7存到`%edx`做后续计算

开始了：把`%edx`的数7给减去`(%rax)`，再把`%edx`复制到`(%rax)`，实现了`x => 7 - x`的映射。

如果我们输入`1 2 3 4 5 6`，那么程序就会处理为`6 5 4 3 2 1`

用JS来表示就相当于

```javascript
const arr = [1, 2, 3, 4, 5, 6].map(x => 7 - x); // [6, 5, 4, 3, 2, 1]
```

好，很有精神，那么我们加大力度！

```asm
  40116f: be 00 00 00 00        mov    $0x0,%esi
  401174: eb 21                 jmp    401197 <phase_6+0xa3>
  401176: 48 8b 52 08           mov    0x8(%rdx),%rdx
  40117a: 83 c0 01              add    $0x1,%eax
  40117d: 39 c8                 cmp    %ecx,%eax
  40117f: 75 f5                 jne    401176 <phase_6+0x82>
  401181: eb 05                 jmp    401188 <phase_6+0x94>
  401183: ba d0 32 60 00        mov    $0x6032d0,%edx
  401188: 48 89 54 74 20        mov    %rdx,0x20(%rsp,%rsi,2)
  40118d: 48 83 c6 04           add    $0x4,%rsi
  401191: 48 83 fe 18           cmp    $0x18,%rsi
  401195: 74 14                 je     4011ab <phase_6+0xb7>
  401197: 8b 0c 34              mov    (%rsp,%rsi,1),%ecx
  40119a: 83 f9 01              cmp    $0x1,%ecx
  40119d: 7e e4                 jle    401183 <phase_6+0x8f>
  40119f: b8 01 00 00 00        mov    $0x1,%eax
  4011a4: ba d0 32 60 00        mov    $0x6032d0,%edx
  4011a9: eb cb                 jmp    401176 <phase_6+0x82>
```



## 完结撒花(并没有)

至此，炸弹被我干掉了

```text
situ@ubuntu:~/Desktop/solutions-csapp/labs/bomb$ ./bomb answer.txt 
Welcome to my fiendish little bomb. You have 6 phases with
which to blow yourself up. Have a nice day!
Phase 1 defused. How about the next one?
That's number 2.  Keep going!
Halfway there!
So you got that one.  Try this one.
Good work!  On to the next...
Congratulations! You've defused the bomb!
```

## Secret Phase

但是你以为就这样结束了？还没有呢！作者特地为我们留下了一个`secret_phase`，你看看`bomb.c`里头这神奇的注释

```c
/* Wow, they got it!  But isn't something... missing?  Perhaps
 * something they overlooked?  Mua ha ha ha ha! */
```

~~你以为拆掉炸弹了，但炸弹没被全拆~~

## 完结撒花(真的)

没错，现在才是真的结束，命令行输出如下结果

## 总结

课业繁重，于是这炸弹有空就拆拆，用了一周左右的时间，其中爆炸了一次，~~是因为我把`phase_6`的大于比较看错成了小于~~
