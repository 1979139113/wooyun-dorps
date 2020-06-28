# PXN防护技术的研究与绕过

**作者: 陵轩，美成，宋杨**

Linux安全机制简介
=====

近些年来，由于**Android**系统的兴起，作为**Android**底层实现的**Linux**内核其安全问题也是越来越被人们所关注。为了减小漏洞给用户带来的危害和损失，**Linux**内核增加了一系列的漏洞缓解技术。其中包括**DEP**，**ALSR**，更强的**Selinu**x，内核代码段只读，PXN等等。**Linux**中这些安全特性的增加，使得黑客们对漏洞的利用越来越困难。其中，**DEP**，**ALSR**，**Selinux**等技术在**PC**时代就已经比较成熟了。内核代码段只读也是可以通过修改**ptmx_fops**指针表等方案来绕过。那么，**PXN**是什么？它又该如何绕过呢？

PXN简介
=====

**PXN**其实就是**Privileged Execute-Never**的缩写，按字面翻译就是“特权执行从不”。它的开启与否主要由页表属性的**PXN**位来控制，如图所示。[![](http://static.wooyun.org//drops/20150806/2015080621354119110.png)](http://drops.wooyun.org/wp-content/uploads/2015/08/%E5%9B%BE%E7%89%871.png)

在没有PXN安全机制的Android机器上，我们一般的提权思路大致可分为如下几步：

1.  修改**ptmx_fops**表的**fsync**指针地址，使其指向我们用户态的提权代码。
2.  应用层调用**fsync**函数，使得系统触发我们的提权代码。
3.  获取**root**权限。
4.  将**ptmx_fops**表的**fsync**指针置为**NULL**，主要防止其他进程调用**fsync**而使系统崩溃。

上述说到的提权步骤是在没有PXN影响下的一般思路。那么在带有**PXN**的机器上表现又是如何呢？经过我们的多番测试，在不同的机型上表现各不相同。在有的机器上会卡住，**shellcode**不会继续执行；有的机器上会产生**Kernel Panic**，机器直接重启。 所以说，PXN的大致工作原理就是在内核状态下，系统是无法直接执行用户态代码的。因此，我们常用的提权思路也就行不通了。更加坑爹的是在绝大部分的**arm64**系统机型（如三星**s6**，华为**p8**），和一些**arm32**位机型（如三星**note3**，三星**s5**等主流机型）的最新**ROM**都默认开启了**PXN**。

PXN初探
=====

在CVE-2015-3636漏洞的一种利用方式中，我们最终可以控制inet_release函数中 sk->sk_prot->close 指针的值。对应汇编代码如图所示。

[![](http://static.wooyun.org//drops/20150806/2015080621354239027.png)](http://drops.wooyun.org/wp-content/uploads/2015/08/%E5%9B%BE%E7%89%872.png)

图2中X2寄存器的值即为sk->sk_prot->close指针的地址，这里的X2对应着32位系统中的R2。BLR X2指令就是跳转到X2所指向的地址处执行。 在开启了PXN的机型上，如果我们将sk_prot->close指针的地址指向用户态的提权代码，那么系统就直接panic重启了，错误信息如下图所示。在这个panic信息中可以看到PC寄存器的值为0x558d15a2a8，是一个用户态地址。

[![](http://static.wooyun.org//drops/20150806/2015080621354278940.png)](http://drops.wooyun.org/wp-content/uploads/2015/08/%E5%9B%BE%E7%89%873.png)

既然在内核状态下PXN机制能阻止系统运行用户态的代码，那么它会不会阻止内核层的代码执行呢？我们将sk_prot->close指针的地址指向了inet_release函数的返回处。代码如图所示。

[![](http://static.wooyun.org//drops/20150806/2015080621354277473.png)](http://drops.wooyun.org/wp-content/uploads/2015/08/%E5%9B%BE%E7%89%874.png)

此时再触发漏洞，函数能正确返回到用户态，并且系统也没有panic，从而验证了PXN是不会阻止内核态代码执行的。这和PXN自身的原理也是一致的。

再战PXN
=====

**基本思路**

在带有PXN的机器上虽然不能执行用户态的shellcode，但是依然可以执行内核态的代码。因此，我们的策略就是使用内核ROP：构建内核gadget，使得栈指针sp泄露，然后通过sp计算得到thread_info结构的地址，之后patch thread_info结构的addr_limit字段，从而达到用户态任意读写内核态的目的。

**构建gadget**

接下来，我们的主要目的就是验证上述方案的可行性。首要任务就是寻找合适的内核gadget。 假设我们现在已经可以利用漏洞控制X1寄存器的值，如图所示。

[![](http://static.wooyun.org//drops/20150806/2015080621354229131.png)](http://drops.wooyun.org/wp-content/uploads/2015/08/%E5%9B%BE%E7%89%875.png)

因此，在寻找ROP需要的gadget时，我们以X1所指向的内存区域为基础来控制其他寄存器的值。在触发漏洞前，通过mmap函数来创建一块用户态可以控制的内存空间。如下所示

```
void *map_addr_tmp = mmap((void*) 0x30303000, 0x10000, 
PROT_WRITE|PROT_READ|PROT_EXEC, 
MAP_SHARED|MAP_ANONYMOUS|MAP_FIXED, -1, 0);

```

做完了上面的准备工作，接下来，我将上述方案分解成下面三步来完成：

1.  泄露sp的值；
2.  计算addr_limit的地址；
3.  patch addr_limit。

首先，用ROP来完成sp的泄露。因为寄存器X1所指向的地址是我们在用户态自己分配出来，并且可以随意控制的内存块，所以我们在第一段gadget中借由X1来布局各个跳转的地址，然后跳转到下一段gadget开始执行。 第二段gadget主要是将sp栈指针的值赋给X0，然后跳转到下一段gadget。在这个例子里面，用户态是无法直接通过X0来获得sp的。所以我们在gadget3里面通过一条STR指令将X0的值保存到用户态。ROP的大致思路如下图所示。 找gadget在某种程度上是一个体力活，当然也可以通过一些工具来Make life easier，大家可以自行Google。注意这里描述的gadget只是一个大致的思路，和实际中可能会有一些区别。

[![](http://static.wooyun.org//drops/20150806/2015080621354291639.png)](http://drops.wooyun.org/wp-content/uploads/2015/08/%E5%9B%BE%E7%89%876.png)

我们得到sp的值后，就可以计算出addr_limit字段的地址了。在arm64系统上，栈的最大深度为16K。其次，addr_limit字段位于thread_info结构+8的位置。因此，计算方式如下所示：

```
unsigned long thread_info_addr = sp & 0xFFFFFFFFFFFFC000;
unsigned long addr_limit_addr = thread_info_addr + 8;
printf("addr_limit_addr: %p\n", addr_limit_addr); 

```

最后，我们通过ROP来设置addr_limit的值为0xFFFFFFFFFFFFFFFF。同样，X1寄存器我们可以随意控制其内容。第一段gadget用来设置各个跳转地址。第二段gadget完成主要任务，即通过一个STR指令来完成patch addr_limit的目的。最后跳转到inet_release的函数返回处。ROP的大致思路如图所示。

[![](http://static.wooyun.org//drops/20150806/2015080621354279580.png)](http://drops.wooyun.org/wp-content/uploads/2015/08/%E5%9B%BE%E7%89%877.png)

完成了addr_limit字段的patch之后，那么用户态就可以任意读写内核态了。接下来的选择就很多了，一种方法是先确定task_struct的地址，它是在thread_info+0x10的位置。得到了task_struct的地址之后，就可以定位到cred字段，之后patch uid、gid、capability、selinux等即可。 实践中，通过结合已知的漏洞，上述方案在近期发布的一些64位旗舰机型上均能测试成功，如图所示。当然，这个思路在32位机型上也同样适用。

[![](http://static.wooyun.org//drops/20150806/2015080621354224216.png)](http://drops.wooyun.org/wp-content/uploads/2015/08/%E5%9B%BE%E7%89%878.png)

假设我们所使用的是一个任意写的漏洞，那么绕过PXN的方式其实是类似的。首先通过任意写的洞控制Kernel的一个函数指针，指向我们的gadget，泄露出sp的值。接下来，可以直接通过任意写的能力去patch addr_limit（不需要再通过ROP去patch了）。

总结
==

* * *

近些年来随着对Android、Linux的研究越来越深入，通用的Linux平台漏洞，例如：CVE-2013-6282、CVE-2014-3153 （towelroot）以及最新的CVE-2015-3636（pingpong）纷纷被公之于众。 同时，Linux上运用的最新的漏洞缓解机制让系统漏洞的利用越发困难，但绕过这些缓解机制的技术也在推陈出新。攻与防的博弈永远不会结束。

参考文章：

[http://git.kernel.org/cgit/linux/kernel/git/davem/net.git/commit/?id=1d4d37159d013a4c54d785407dd8902f901d7bc5](http://git.kernel.org/cgit/linux/kernel/git/davem/net.git/commit/?id=1d4d37159d013a4c54d785407dd8902f901d7bc5)

[http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.subset.architecture.reference/index.html](http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.subset.architecture.reference/index.html)

[https://bugzilla.redhat.com/show_bug.cgi?id=1218074](https://bugzilla.redhat.com/show_bug.cgi?id=1218074)