# return2libc学习笔记

0x00 背景介绍
=====

author:万抽抽

r2libc技术是一种缓冲区溢出利用技术，主要用于克服常规缓冲区溢出漏洞利用技术中面临的no stack executable限制(**所以后续实验还是需要关闭系统的ASLR，以及堆栈保护**)，比如PaX和ExecShield安全策略。该技术主要是通过覆盖栈帧中保存的函数返回地址(eip)，让其定位到libc库中的某个库函数(如，system等)，而不是直接定位到shellcode。然后通过在栈中精心构造该库函数的参数，以便达到类似于执行shellcode的目的。

0x01 技术原理
=====

大致技术原理，如图1-1所示：

![p1](http://drops.javaweb.org/uploads/images/6c9d6843d10ee5279f8f42324021f941fe99068d.jpg)

图1-1 r2libc技术原理

简要概括如下：

1) 使用libc库中system函数的地址覆盖掉原本的返回地址；这样原函数返回的时候会转而调用system函数。

获取system函数的返回地址很简单，只需要使用gdb调试目标程序，在main函数下断点，程序运行中断在断点处后，使用`p system`命令即可:

```
>>> p system
$1 = {<text variable, no debug info>} 0xb7e56190 <__libc_system>

```

该方法可以获取任意libc函数的地址。

2) 设置system函数返回后的地址，以及为system函数构造我们预定的参数。

难点主要在第2步中system函数相关的栈帧结构的安排上，比如为什么Filler就是Return Address After system，为什么传递给system的参数紧跟在Fillter之后？这就涉及到函数的调用规则。我们知道函数调用在汇编中通过call指令实现，而函数返回则通过ret指令实现。Call指令可以实现多种方式的函数跳转，这里为了简便，暂且只考虑跳转地址在内存中的call指令的实现

CPU在执行call指令时需要进行两步操作：

1.  将当前的IP(也就是函数返回地址)入栈，即：`push IP`;
2.  跳转，即：`jmp dword ptr 内存单元地址`。

CPU在执行ret指令时只需要恢复IP寄存器即可，因此ret指令相当于`pop IP`。

因此对于正常函数调用而言，其栈帧结构如下图1-2所示：

![p2](http://drops.javaweb.org/uploads/images/265fa1d8d45cf61eb8a602a7bb398c7a053f7e9d.jpg)

图1-2 函数调用栈帧结构图

但是由于我们使用system的函数地址替换了原本的IP寄存器，强制执行system函数，破坏了原程序的栈帧分配和释放策略，所以后续的操作必须基于这个被破坏的栈帧结构实现。

### 1.1 为何Filler是函数的返回地址

首先，解释为何Filler是函数的返回地址。这需要我们查看system函数的汇编实现。通过 gdb调试得到如下信息：

![p3](http://drops.javaweb.org/uploads/images/7d7c7a460b8fa2c457b99f45fa2b5162b4675686.jpg)

图1-3 system函数汇编实现

正常情况下，我们是通过call指令进行函数调用的，因此在进入到system函数之前，call指令已经通过push IP将其返回地址push到栈帧中了，所以在正常情况下ret指令pop到 IP的数据就是之前call指令push到栈帧的数据，也就是说两者是成对的。但是！！！在我们的漏洞利用中，直接通过覆盖IP地址跳转到了system函数，而并没有经过call调用，也即是没有push IP的操作，但是system函数却照常进行了ret指令的pop IP操作。那么这个ret指令pop到IP的是哪一处地址的数据呢？答案就是Filler!

在程序执行到图1-1中的Saved EIP(即system函数地址)的时候，此时的ESP指向了Filler，然后就转而执行system函数。通过分析system函数的汇编实现可以看出：该函数中对ESP的操作都是成对出现的，如`push %ebx`与`pop %ebx`等。所以当执行到ret指令时，ESP还是指向Filler，也就是说ret指令内涵的pop IP操作就是将Filler的数据pop到IP寄存器中！

### 1.2 为何传递给system的参数紧跟在Filler之后

其实通过2.1节的分析，答案已经很明了了。在正常的函数调用中，其栈帧分布如图1-2所示，但是在此漏洞利用中，由于我们强制更改了调用过程，省去了call调用的push IP步骤，因此就造成了**Filler变成了EIP**。但是，需要注意的是，我们也仅仅是省去了push IP这一步而已，其他步骤与正常函数调用并无区别，所以如果我们将Filler看做是保存的返回地址EIP的话，那么它“之后(相对栈增长方向而言)”的数据就自然而然变成了system函数的参数了。

### 1.3构造"/bin/sh"参数

下面唯一的问题就是如何在栈中构造”/bin/sh”参数了。当然，这里用“构造”一词并不准确，“借用”更合适——我们从内存中搜寻此字符串，然后将该字符串在内存中的起始地址赋值到Filler之后即可。那么如何获取到这个字符串的地址呢？我们知道，LINUX的SHELL环境变量的值一般就是“/bin/sh”，所以我们可以通过gdb调试器，暴力搜索`%esp`寄存器之后开始的n个字符串，方法如下图1-4所示：

![p4](http://drops.javaweb.org/uploads/images/6ace494e83924f15cad34559050e4322b67a7680.jpg)

图1-4 GDB暴力搜索字符串

但是这种方法比较“非黑客范”，个人觉得gray hat hacking第四版中提及的方法更犀利，该方法先通过dlopen和dlsym获取system之类的函数的地址，再借助于setjmp与longjmp函数以及信号量机制，以system函数的地址为起点从前、后两个方向搜索字符串，详情见书本LAB11-5。

**还有一种更简单的方法**，详细步骤如下图1-5所示：

![p5](http://drops.javaweb.org/uploads/images/e5d9949faf7dba8025854ba83cf7a25d6540cc3d.jpg)

图1-5 构造“/bin/sh”参数方法三

只要我们将`CHOUCHOU_SH`定义为“/bin/sh”即可。可以用这种方式借助环境变量构造任意的字符串。此方法会在后文中大量使用。这里gtenv的源代码如下：

```
#include <stdio.h>
#include <stdlib.h>

int main(int argc, char *argv[]) {
    char *addr;
    addr = getenv(argv[1]);
    printf("%s is located at %p\n", argv[1], addr);
    return 0;
}

```

构造完参数之后，return to libc利用就算告一段落了，但是该方案是基于system函数实现的，且在一次攻击中只执行一个libc函数，局限性较大，另外system函数有一个致命的缺陷就是：有时候我们并不能利用它成功获取root权限。

因为system函数本质上就是通过fork一个子进程，然后该子进程通过系统自带的sh执行system的命令。而在某些系统中，在启动新进程执行sh命令的时候会将它的特权给剔除掉(如果/bin/sh指向zsh，则不会进行权限降低；如果/bin/sh指向bash则会进行权限降低)，这样我们system就无法获取root权限了。

为了解决这个问题，高手们又研发了一种更高级的攻击技术——基于libc的函数调用链攻击。

0x02 基于libc的函数调用链攻击原理
=====

回顾图1-1，我们知道system函数执行后，其返回地址是Filler，因此如果我们将需要执行的第二个libc函数的地址放置到Filler上，那么不就构造了一个函数调用链么？详情见图2-1：

![p6](http://drops.javaweb.org/uploads/images/933999a7bd5cf045bf9406cb2afa08380fb67532.jpg)

图2-1基于libc的函数调用链攻击原理图

前面说到，单纯通过system函数不一定能够获取到root权限，但是，如果我们在执行system函数之前，先通过setuid(0)函数将正在运行的程序的SET-UID提升至root权限，然后再通过system函数执行sh就一定能够获取root权限了(关于为什么通过setuid(0)设置SET-UID为root权限之后，system执行sh就能获取root权限，超出了本文的范畴，大家可自行google)。

0x03 进阶——使用Execl函数
=====

前文提到，使用system函数必须结合setuid(0)函数才能确保获取root权限，这种处理相对麻烦，好在我们可以直接使用execl函数解决(system是通过fork子进程执行shell，而execl是直接在当前进程中执行shell)。

Execl函数的标准用法如下：`Execl("/bin/sh","/bin/sh", NULL)`。

这引入了一个新的问题——如何构造‘NULL’参数。因为在输入参数中是不能夹杂NULL字节的，否则会截断我们构造的攻击buffer。所以我们得采用迂回的办法——借助格式化字符串漏洞，在漏洞程序进程的堆栈中构造一个‘NULL’参数，而不是直接输入。

### 3.1 构造‘NULL’参数

格式化字符串漏洞主要是利用printf函数的format参数进行攻击，这里我们需要构造一个特定的format参数以实现在“合适的位置”写入’NULL’参数。

很自然地，我们会想到利用格式化字符串中的‘`%#$n`’。这里的“`#`”的具体数值需要根据NULL参数在stack中的offset加以确定。具体的构造方式在后文详细介绍。

### 3.2 构造libc函数调用链

根据我们的攻击逻辑，我们需要完成如下几步：

1.  基于格式化字符串漏洞，利用printf函数构造execl函数的第三个参数“NULL”；
2.  调用execl函数执行sh；
3.  调用exit函数，退出攻击。

因此，我们需要构造如图3-1所示的stack数据：

![p7](http://drops.javaweb.org/uploads/images/456d596cd0abb551024bdc625f95ee18894e189f.jpg)

图3-1 构造实施攻击的栈结构

简要概括上图的执行逻辑：

1.  先通过`printf(“%5$n”)`函数将Addr of HERE处字节设置为NULL;
2.  执行POP/RET指令，跳转到并执行`execl(“/bin/sh”, “/bin/sh”, NULL)`函数；
3.  执行exit函数，完成攻击。

这里我们详细探讨1、2步的实现原理。

**一、为什么用“%5$n”作为参数？**

因为Addr of HERE的地址刚好离Addr of ‘%5$n’有**5个指针**的距离。根据表3-1 Blaess, Grenier, and Raynal提出的magic formula中最后一行的推导公式可以得出以上结论。

表3-1 magic formula

![p8](http://drops.javaweb.org/uploads/images/d9dd32ce1435109146a9180fa4d64ddc91f864f5.jpg)

公式的详细推导见：  
[http://www.cgsecurity.org/Articles/SecProg/Art4/](http://www.cgsecurity.org/Articles/SecProg/Art4/)

另外，提及一点注意事项：在Linux shell中，如果使用双引号`” ”`包含字符串的话，就需要在’`$`’之前加上转义字符`‘\’`，如果使用单引号`’ ’`包含字符串的话，就不需要额外添加转义字符`’\’`了。详情见：

[http://lspgyy.blog.51cto.com/5264172/1282107](http://lspgyy.blog.51cto.com/5264172/1282107)(值得一看！)

**二、POP/RET指令的作用**

为什么它能够让程序正确无误地跳转到execl函数并执行呢？首先我们需要知道这里的POP/RET指令是指两条连续的指令：

```
pop %ebp
ret

```

结合1.1章节的分析，可以知道当执行完printf函数之后，此时的esp指针指向图3-1中的Add of ‘%5$n’处，如图3-2所示：

![p9](http://drops.javaweb.org/uploads/images/7b3d379a3fec569c21cdecc123185870c21b8606.jpg)

图3-2 执行完printf函数后的ESP指针位置

现在开始执行`pop %ebp`指令，将Addr of ‘%5$n’放入ebp寄存器中；然后再执行`ret`指令(即`pop IP`)，将Addr of execl放入IP寄存器中，这样程序下一步就会转而执行execl函数了，而此时的ESP指针指向Addr of exit，如下图3-3所示：

![p10](http://drops.javaweb.org/uploads/images/b04bc0ab7aa620239739952f278655628557ab90.jpg)

图3-3 开始执行execl函数之前ESP指针位置

那么我们如何在目标进程的内存中找到这个POP/RET指令呢？这里需要借助metasploits的msfelfscan程序，假设目标程序为target，那么可以通过如下命令获取：

```
msfelfscan –p –D target
-p, --poppopret，表示Search for pop+pop+ret combinations；
-D, --disasm，表示Disassemble the bytes at this address；

```

输出结果如下：

```
0x080484ae pop edi; pop ebp; ret
0x080484ae  pop edi
0x080484af  pop ebp
0x080484b0  ret

```

现在我们就成功获取了目标程序中POP/RET指令的地址了。

至此利用execl的整个攻击逻辑和关键技术点，就介绍完毕了，下面我们找个例子进行实际操作。

0x04 实战演练
=====

含有缓冲区溢出漏洞的程序源代码如下：

```
/*vuln.c*/
#include <stdio.h>

/*small buffer vuln prog*/
int main(int argc, char* argv[]) {
    char buffer[7];
    strcpy(buffer, argv[1]);
    return 0;
}

```

我们关闭系统的ASLR保护机制之后，再使用root用户编译：

```
# gcc  -fno-stack-protector  -o  vuln  vuln.c

```

然后添加SET-UID：`# chmod u+s vuln`

然后退出root用户，切换到普通用户模式：`# exit`

现在我们可以开始以普通用户权限进行提权攻击了！整个攻击工作可以分为两个大的步骤：1)确定漏洞buffer的大小；2)开始构建攻击buffer。

### 4.1 确定漏洞buffer大小

为方便快速确定大小，我们可以借助metasploits提供的pattern_create.rb和pattern_offset脚本。前者用于生成特殊的输入数据，后者用于帮助确定指定数据在该输入数据中的偏移值。

pattern_create.rb的使用方式如下：

```
Usage: pattern_create.rb length [set a] [set b] [set c]

```

这里我们使用pattern_create.rb 250生成250字节的输入数据:

```
Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag6Ag7Ag8Ag9Ah0Ah1Ah2Ah3Ah4Ah5Ah6Ah7Ah8Ah9Ai0Ai1Ai2A

```

然后使用gdb调试目标程序：

```
chouchou@ubuntu:~/security/flaws/return2libc$ gdb  ./vuln -q
Reading symbols from ./vuln...(no debugging symbols found)...done.
>>> r  Aa0……i2A(省略部分输入数据)
回车之后，gdb输出如下错误数据：
Program received signal SIGSEGV, Segmentation fault.
0x61413661 in ?? ()
Traceback (most recent call last):
  File "<string>", line 701, in lines
  File "<string>", line 116, in run
gdb.error: No function contains program counter for selected frame.

```

显然，目标程序去执行0x61413661的地址时发生错误，这个地址就是被我们使用攻击buffer覆盖后的EIP地址，现在我们使用pattern_offset.rb查找这个地址在攻击buffer中的位置：

```
chouchou@ubuntu:~/security/flaws/return2libc$ pattern_offset.rb 0x61413661 250
[*] Exact match at offset 19

```

好了，现在我们已经确定了用于覆盖的EIP的数据在攻击buffer中的偏移值为19，下面就开始精心构造攻击buffer了。

### 4.2 构建攻击buffer

从前文的分析知道，要使用execl完成基于ret2libc的root提权攻击，我们需要实现搜集如下数据信息：

1.  printf函数的地址，以及printf函数要使用的字符串’%5$n’的地址；
2.  execl函数的地址，以及该函数使用的’/bin/sh’字符串的地址；
3.  exit函数的地址；
4.  POP/RET 指令块的地址。
    
    Printf 0xb7e63280 Execl 0xb7ecbec0 Exit 0xb7e491e0 ‘%5$n’ 0xbffff2b6 ‘/bin/sh’ 0xbffffa6e POP/RET 0x080484af Addr of NULL 0xbfffefc9+18+4*7=0xbfffeff7
    

获取了这些数据之后，将它们各自安插到图3-1的合适位置即可构造对应的攻击buffer。

```
19字节overflow| 0xb7e63280 | 0x080484af | 0xbffff284 | 
   0xb7ecbec0 | 0xb7e491e0 | 0xbffffeea | 0xbffffeea | 0xbfffeff7

```

0x05 对抗ASLR
=====

鉴于ASLR机制只是将既然stack，heap以及共享库文件的起始地址进行了随机化，而文件内部各部分数据之间的偏移值却是固定不变的，所以攻击的整体思路如下：

先泄漏出libc.so某些函数在内存中的地址，然后再利用泄漏出的函数地址根据偏移量计算出system()函数等其他函数的地址，字符串地址类似。

那么怎么才能泄露出Libc库文件的地址呢？考虑到程序本身在内存中的地址并不是随机的，所以只要将返回值设置到程序本身所处的内存段就可以了，当然，**前提是我们必须获取目标系统中的libc.so库文件**。Linux内存随机化分布图如下图所示：

![p11](http://drops.javaweb.org/uploads/images/91d25eb6a3136d7e15e2555cdb3338c71f25d1b3.jpg)

整体攻击思路：

1.观察目标程序，查看其本身使用的、处于Libc库文件的库函数，常见的函数有write, read等；

观察方法很简单：

1) 首先使用`objdump –R`命令查看目标文件的**动态重定位项**，也就是常说的**GOT表项**：

![p12](http://drops.javaweb.org/uploads/images/7eabee3c6d605c278bbd16af05994d80289d9c49.jpg)

2) 然后使用`objdump –j .glt –d`命令反编译目标文件的**plt表**：

```
Disassembly of section .plt:

08048300 <read@plt-0x10>:
 8048300:   ff 35 04 a0 04 08       pushl  0x804a004
 8048306:   ff 25 08 a0 04 08       jmp    *0x804a008
 804830c:   00 00                   add    %al,(%eax)
    ...

08048310 <read@plt>:
 8048310:   ff 25 0c a0 04 08       jmp    *0x804a00c
 8048316:   68 00 00 00 00          push   $0x0
 804831b:   e9 e0 ff ff ff          jmp    8048300 <_init+0x30>

08048320 <__gmon_start__@plt>:
 8048320:   ff 25 10 a0 04 08       jmp    *0x804a010
 8048326:   68 08 00 00 00          push   $0x8
 804832b:   e9 d0 ff ff ff          jmp    8048300 <_init+0x30>

08048330 <__libc_start_main@plt>:
 8048330:   ff 25 14 a0 04 08       jmp    *0x804a014
 8048336:   68 10 00 00 00          push   $0x10
 804833b:   e9 c0 ff ff ff          jmp    8048300 <_init+0x30>

08048340 <write@plt>:
 8048340:   ff 25 18 a0 04 08       jmp    *0x804a018
 8048346:   68 18 00 00 00          push   $0x18
 804834b:   e9 b0 ff ff ff          jmp    8048300 <_init+0x30>

```

熟悉linux中elf文件的动态重定位机制都知道，由于linux采用懒绑定机制，所以对于GOT表中的动态重定位符号，在第一次调用的时候got表项会定位到其对应的plt表项中，再由该plt表项转到真正的函数处。第一次执行完之后，系统会将此时GOT表项的位置加以修改，直接指向真正的函数处，而不再指向之前的plt表项。

2.根据第1步的结果确定想要获取的属于Libc的某个函数的地址，这里我们使用write函数。获取方法并不难，从上面对linux动态重定位机制的介绍可以知道，我们只需要按照如下方式做就可以了：首先构造payload执行目标文件的write函数(怎么执行呢？使用目标文件中的write@plt地址覆盖EIP)，一旦执行了write@plt指令，此时writez@got中的数据就会被系统修改为libc中真正的write函数的地址，所以我们再使用write函数打印出该地址即可：payload构造如下：

![p13](http://drops.javaweb.org/uploads/images/79ac83ae93e0c73963e12e73635a9cf8aaf84204.jpg)

即执行`write(STDOUT, &write@got, 4)`，这样就会将libc中真正的write函数地址打印到标准输出中。这里的vul_func地址可以通过`objdump –d`获取，当然使用IDA更方便。

3.获取了write的真正地址之后，只需要根据libc中write函数与system函数的相对偏移地址就可以计算出system函数真正的地址，计算公式如下：

```
system_real_addr = write_real_addr – ( write_addr_in_libc – system_addr_in_libc)

```

字符串的地址获取方式类似，这里不再细说。

4.执行`system(‘/bin/sh’)`，payload构造如下：

![p14](http://drops.javaweb.org/uploads/images/9ea3c7e4ea9b7719fe7ad2fc52a9bd2f6922c755.jpg)

0x06 参考文献
=====

*   [http://stackoverflow.com/questions/1697440/difference-between-system-and-exec-in-linux](http://stackoverflow.com/questions/1697440/difference-between-system-and-exec-in-linux)
*   [http://blog.sina.com.cn/s/blog_70dd16910100rdgn.html](http://blog.sina.com.cn/s/blog_70dd16910100rdgn.html)

介绍setuid的两篇好文：

*   [http://hubingforever.blog.163.com/blog/static/171040579201372932033844/](http://hubingforever.blog.163.com/blog/static/171040579201372932033844/)
*   [http://nba20717zcx.blog.51cto.com/343890/392063](http://nba20717zcx.blog.51cto.com/343890/392063)
*   [http://www.cs.berkeley.edu/~daw/papers/setuid-usenix02.pdf](http://www.cs.berkeley.edu/~daw/papers/setuid-usenix02.pdf)