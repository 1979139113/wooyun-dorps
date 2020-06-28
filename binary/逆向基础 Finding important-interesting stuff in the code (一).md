# 逆向基础 Finding important/interesting stuff in the code (一)

V 寻找代码中有趣或者重要的部分
=====

现代软件设计中，极简不是特别重要的特性。

并不是因为程序员编写的代码多，而是由于许多库通常都会静态链接到可执行文件中。如果所有的外部库都移入了外部DLL文件中，情况将有所不同。(C++使用STL和其他模版库的另一个原因)

因此，确定函数的来源很重要，是否来源于标准库或者其他著名的库(比如[Boost](http://go.yurichev.com/17036)，[libpng](http://go.yurichev.com/17037))，是否与我们在代码中寻找的东西相关。

通过重写所有的C/C++代码来寻找我们想要的东西是不现实的。

逆向工程师的一个主要的任务是迅速定位到目标代码。

IDA反汇编工具允许我们搜索文本字符串，字节序列和常量。甚至可以导出为.lst或者.asm文件，然后使用grep,awk等工具进一步分析。

当你尝试去理解某些代码的功能时，一些开源库比如libpng会容易理解一些。当你觉得某些常量或者文本字符串眼熟时，值得用google搜索一下。如果你发现他们在某些地方使用了开源项目时，那么只要对比一下函数就可以了。这些方法能够解决部分问题。

举个例子，如果一个程序使用XML文件，那么第一步是确定使用了哪个XML库。通常情况下使用的是标准库(或者有名的库)而非自编写的库。

再举个例子，有一次我尝试去理解SAP 6.0中网络包如何压缩与解压。整个软件很大，但手头有一个包含详细debug信息的.PDB文件，非常方便。最后我找到一个负责解压网络包的函数，叫CsDecomprLZC。我马上就用google搜索了函数名，发现MaxDB(一个开源SAP项目)也使用了这个函数。[http://www.google.com/search?q=CsDecomprLZC](http://www.google.com/search?q=CsDecomprLZC)

然后惊奇的发现，MaxDB和SAP 6.0 使用同样的代码来处理压缩和解压网络包。

第55章
=====

识别可执行文件
-------

### 55.1 Microsoft Visual C++

可导入的MSVC版本和DLL文件如下图：

![](http://drops.javaweb.org/uploads/images/680cd69cec80413d66601aa869b985aad1c9d0cd.jpg)

msvcp*.dll包含C++相关函数，因此如果导入了这类dll，便可推测是C++程序。

#### 55.1.1命名管理

命名通常以问号?开始。

获取更多关于MSVC命令管理的信息：51.1.1节

### 55.2 GCC

除了*NIX环境，Win32下也有GCC，需要Cygwin和MinGW。

#### 55.2.1 命名管理

命名通常以_Z符号开头。

更多关于GCC命名管理的信息：51.1.1节

#### 55.2.2 Cygwin

cygwin1.dll经常被导入。

#### 55.2.3 MinGW

msvcrt.dll可能会被导入。

### 55.3 Intel FORTRAN

libifcoremd.dll,libifportmd.dll和libiomp5md.dll(OpenMP支持)可能会被导入。

libifcoremd.dll中许多函数以前缀名for_开始，表示FORTRAN。

### 55.4Watcom,OpenWatcom

#### 55.4.1 命名管理

命名通常以W符号开始。

举个例子，下面是"class"类名为"method"的方法没有任何参数并且返回void的加密：

`W?method$_class$n__v`

### 55.5 Borland

这里有一个有关Borland Delphi和C++开发者命名管理的例子：

```
@TApplication@IdleAction$qv
@TApplication@ProcessMDIAccels$qp6tagMSG
@TModule@$bctr$qpcpvt1
@TModule@$bdtr$qv
@TModule@ValidWindow$qp14TWindowsObject
@TrueColorTo8BitN$qpviiiiiit1iiiiii
@TrueColorTo16BitN$qpviiiiiit1iiiiii
@DIB24BitTo8BitBitmap$qpviiiiiit1iiiii
@TrueBitmap@$bctr$qpcl
@TrueBitmap@$bctr$qpvl
@TrueBitmap@$bctr$qiilll

```

命名通常以@符号开始，然后是类名、方法名、加密方法的参数类型。

这些名称会被导入到.exe，.dll和debug信息内等等。

Borland Visual Component Libarary(VCL)存储在.bpl文件中，而不是.dll。比如vcl50.dll,rtl60.dll。

其他可能导入的DLL：BORLNDMM.DLL。

#### 55.5.1 Delphi

几乎所有的Delphi可执行文件的代码段都以"Boolean"字符串开始，和其他类型名称一起。 下面是一个典型的Delphi程序的代码段开头，这个块紧接着win32 PE文件头：

```
00000400  04 10 40 00 03 07 42 6f  6f 6c 65 61 6e 01 00 00  |..@...Boolean...|
00000410  00 00 01 00 00 00 00 10  40 00 05 46 61 6c 73 65  |........@..False|
00000420  04 54 72 75 65 8d 40 00  2c 10 40 00 09 08 57 69  |.True.@.,.@...Wi|
00000430  64 65 43 68 61 72 03 00  00 00 00 ff ff 00 00 90  |deChar..........|
00000440  44 10 40 00 02 04 43 68  61 72 01 00 00 00 00 ff  |D.@...Char......|
00000450  00 00 00 90 58 10 40 00  01 08 53 6d 61 6c 6c 69  |....X.@...Smalli|
00000460  6e 74 02 00 80 ff ff ff  7f 00 00 90 70 10 40 00  |nt..........p.@.|
00000470  01 07 49 6e 74 65 67 65  72 04 00 00 00 80 ff ff  |..Integer.......|
00000480  ff 7f 8b c0 88 10 40 00  01 04 42 79 74 65 01 00  |......@...Byte..|
00000490  00 00 00 ff 00 00 00 90  9c 10 40 00 01 04 57 6f  |..........@...Wo|
000004a0  72 64 03 00 00 00 00 ff  ff 00 00 90 b0 10 40 00  |rd............@.|
000004b0  01 08 43 61 72 64 69 6e  61 6c 05 00 00 00 00 ff  |..Cardinal......|
000004c0  ff ff ff 90 c8 10 40 00  10 05 49 6e 74 36 34 00  |......@...Int64.|
000004d0  00 00 00 00 00 00 80 ff  ff ff ff ff ff ff 7f 90  |................|
000004e0  e4 10 40 00 04 08 45 78  74 65 6e 64 65 64 02 90  |..@...Extended..|
000004f0  f4 10 40 00 04 06 44 6f  75 62 6c 65 01 8d 40 00  |..@...Double..@.|
00000500  04 11 40 00 04 08 43 75  72 72 65 6e 63 79 04 90  |..@...Currency..|
00000510  14 11 40 00 0a 06 73 74  72 69 6e 67 20 11 40 00  |..@...string .@.|
00000520  0b 0a 57 69 64 65 53 74  72 69 6e 67 30 11 40 00  |..WideString0.@.|
00000530  0c 07 56 61 72 69 61 6e  74 8d 40 00 40 11 40 00  |..Variant.@.@.@.|
00000540  0c 0a 4f 6c 65 56 61 72  69 61 6e 74 98 11 40 00  |..OleVariant..@.|
00000550  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00000560  00 00 00 00 00 00 00 00  00 00 00 00 98 11 40 00  |..............@.|
00000570  04 00 00 00 00 00 00 00  18 4d 40 00 24 4d 40 00  |.........M@.$M@.|
00000580  28 4d 40 00 2c 4d 40 00  20 4d 40 00 68 4a 40 00  |(M@.,M@. M@.hJ@.|
00000590  84 4a 40 00 c0 4a 40 00  07 54 4f 62 6a 65 63 74  |.J@..J@..TObject|
000005a0  a4 11 40 00 07 07 54 4f  62 6a 65 63 74 98 11 40  |..@...TObject..@|
000005b0  00 00 00 00 00 00 00 06  53 79 73 74 65 6d 00 00  |........System..|
000005c0  c4 11 40 00 0f 0a 49 49  6e 74 65 72 66 61 63 65  |..@...IInterface|
000005d0  00 00 00 00 01 00 00 00  00 00 00 00 00 c0 00 00  |................|
000005e0  00 00 00 00 46 06 53 79  73 74 65 6d 03 00 ff ff  |....F.System....|
000005f0  f4 11 40 00 0f 09 49 44  69 73 70 61 74 63 68 c0  |..@...IDispatch.|
00000600  11 40 00 01 00 04 02 00  00 00 00 00 c0 00 00 00  |.@..............|
00000610  00 00 00 46 06 53 79 73  74 65 6d 04 00 ff ff 90  |...F.System.....|
00000620  cc 83 44 24 04 f8 e9 51  6c 00 00 83 44 24 04 f8  |..D$...Ql...D$..|
00000630  e9 6f 6c 00 00 83 44 24  04 f8 e9 79 6c 00 00 cc  |.ol...D$...yl...|
00000640  cc 21 12 40 00 2b 12 40  00 35 12 40 00 01 00 00  |.!.@.+.@.5.@....|
00000650  00 00 00 00 00 00 00 00  00 c0 00 00 00 00 00 00  |................|
00000660  46 41 12 40 00 08 00 00  00 00 00 00 00 8d 40 00  |FA.@..........@.|
00000670  bc 12 40 00 4d 12 40 00  00 00 00 00 00 00 00 00  |..@.M.@.........|
00000680  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00000690  bc 12 40 00 0c 00 00 00  4c 11 40 00 18 4d 40 00  |..@.....L.@..M@.|
000006a0  50 7e 40 00 5c 7e 40 00  2c 4d 40 00 20 4d 40 00  |P~@.\~@.,M@. M@.|
000006b0  6c 7e 40 00 84 4a 40 00  c0 4a 40 00 11 54 49 6e  |l~@..J@..J@..TIn|
000006c0  74 65 72 66 61 63 65 64  4f 62 6a 65 63 74 8b c0  |terfacedObject..|
000006d0  d4 12 40 00 07 11 54 49  6e 74 65 72 66 61 63 65  |..@...TInterface|
000006e0  64 4f 62 6a 65 63 74 bc  12 40 00 a0 11 40 00 00  |dObject..@...@..|
000006f0  00 06 53 79 73 74 65 6d  00 00 8b c0 00 13 40 00  |..System......@.|
00000700  11 0b 54 42 6f 75 6e 64  41 72 72 61 79 04 00 00  |..TBoundArray...|
00000710  00 00 00 00 00 03 00 00  00 6c 10 40 00 06 53 79  |.........l.@..Sy|
00000720  73 74 65 6d 28 13 40 00  04 09 54 44 61 74 65 54  |stem(.@...TDateT|
00000730  69 6d 65 01 ff 25 48 e0  c4 00 8b c0 ff 25 44 e0  |ime..%H......%D.|

```

数据段(DATA)最开始的四字节可能是00 00 00 00，32 13 8B C0或者FF FF FF FF。在处理加壳/加密的 Delphi可执行文件时这个信息很有用。

### 55.6其他有名的DLLs

*   vcomp*.dll Microsoft实现的OpenMP

第56章
=====

与外部世界通信(win32)
--------------

有时理解函数的功能通过观察函数的输入与输出就足够了。这样可以节省时间。

文件和注册访问：对于最基本的分析，SysInternals的[Process Monitor](http://go.yurichev.com/17301)工具很有用。

对于基本网络访问分析，Wireshark很有帮助。

但接下来你仍需查看内部。

第一步是查看使用的是OS的API哪个函数，标准库是什么。

如果程序被分为主要的可执行文件和一系列DLL文件，那么DLL文件中的函数名可能会有帮助。

如果我们对指定文本调用MessageBox()的细节感兴趣，我们可以在数据段中查找这个文本，定位文本引用处，以及控制权交给我们感兴趣的MessageBox()的地方。

如果我们在谈论电子游戏，并且对里面的事件的随机性感兴趣，那么我们可以查找rand()函数或者类似函数(比如马特赛特旋转演算法)，然后定位调用这些函数的地方，更重要的是，函数执行结果如何被使用。

但如果不是一个游戏，并且仍然使用了rand()函数，找出原因也很有意思。这里有一些关于在数据压缩算法中意外出现rand()函数调用的例子(模仿加密)：<blog.yurichev.com>

### 56.1 Windows API中常用的函数

下面这些函数可能会被导入。值得注意的是并不是每个函数都在代码中使用。许多函数可能被库函数和CRT代码调用。

*   注册表访问(advapi32.dll):RegEnumKeyEx, RegEnumValue, RegGetValue7, RegOpenKeyEx, RegQueryValueEx
*   .ini-file访问(kernel32.dll): GetPrivateProfileString
*   资源访问(68.2.8): (user32.dll): LoadMen
*   TCP/IP网络(ws2_32.dll): WSARecv, WSASend
*   文件访问(kernel32.dll): CreateFile, ReadFile, ReadFileEx, WriteFile, WriteFileEx
*   Internet高级访问(wininet.dll): WinHttpOpen
*   可执行文件数字签名(wintrust.dll): WinVerifyTrust
*   标准MSVC库(如果是动态链接的) (msvcr*.dll): assert, itoa, ltoa, open, printf, read, strcmp, atol, atoi, fopen, fread, fwrite, memcmp, rand, strlen, strstr, strchr

### 56.2 tracer:拦截所有函数特殊模块

这里有一个INT3断点，只触发了一次，但可以为指定DLL中的所有函数设置。

```
--one-time-INT3-bp:somedll.dll!.*

```

我们给所有前缀是xml的函数设置INT3断点吧：

```
--one-time-INT3-bp:somedll.dll!xml.*

```

另一方面，这样的断点只会触发一次。

Tracer会在函数调用发生时显示调用情况，但只有一次。但查看函数参数是不可能的。

尽管如此，在你知道这个程序使用了一个DLL，但不知道实际上使用了哪个函数并且有许多的函数的情况下，这个特性还是很有用的。

举个例子，我们来看看，cygwin的uptime工具使用了什么：

```
tracer -l:uptime.exe --one-time-INT3-bp:cygwin1.dll!.*

```

我们可以看见所有的至少调用了一次的cygwin1.dll库函数，以及位置：

```
One-time INT3 breakpoint: cygwin1.dll!__main (called from uptime.exe!OEP+0x6d (0x40106d))
One-time INT3 breakpoint: cygwin1.dll!_geteuid32 (called from uptime.exe!OEP+0xba3 (0x401ba3))
One-time INT3 breakpoint: cygwin1.dll!_getuid32 (called from uptime.exe!OEP+0xbaa (0x401baa))
One-time INT3 breakpoint: cygwin1.dll!_getegid32 (called from uptime.exe!OEP+0xcb7 (0x401cb7))
One-time INT3 breakpoint: cygwin1.dll!_getgid32 (called from uptime.exe!OEP+0xcbe (0x401cbe))
One-time INT3 breakpoint: cygwin1.dll!sysconf (called from uptime.exe!OEP+0x735 (0x401735))
One-time INT3 breakpoint: cygwin1.dll!setlocale (called from uptime.exe!OEP+0x7b2 (0x4017b2))
One-time INT3 breakpoint: cygwin1.dll!_open64 (called from uptime.exe!OEP+0x994 (0x401994))
One-time INT3 breakpoint: cygwin1.dll!_lseek64 (called from uptime.exe!OEP+0x7ea (0x4017ea))
One-time INT3 breakpoint: cygwin1.dll!read (called from uptime.exe!OEP+0x809 (0x401809))
One-time INT3 breakpoint: cygwin1.dll!sscanf (called from uptime.exe!OEP+0x839 (0x401839))
One-time INT3 breakpoint: cygwin1.dll!uname (called from uptime.exe!OEP+0x139 (0x401139))
One-time INT3 breakpoint: cygwin1.dll!time (called from uptime.exe!OEP+0x22e (0x40122e))
One-time INT3 breakpoint: cygwin1.dll!localtime (called from uptime.exe!OEP+0x236 (0x401236))
One-time INT3 breakpoint: cygwin1.dll!sprintf (called from uptime.exe!OEP+0x25a (0x40125a))
One-time INT3 breakpoint: cygwin1.dll!setutent (called from uptime.exe!OEP+0x3b1 (0x4013b1))
One-time INT3 breakpoint: cygwin1.dll!getutent (called from uptime.exe!OEP+0x3c5 (0x4013c5))
One-time INT3 breakpoint: cygwin1.dll!endutent (called from uptime.exe!OEP+0x3e6 (0x4013

```

第57章
=====

字符串
---

### 57.1 文本字符串

#### 57.1.1 C/C++

普通的C字符串是以零结束的(ASCIIZ字符串)。

C字符串格式(以零结束)是这样的是出于历史原因。[Rit79中]:

```
A minor difference was that the unit of I/O was the word, not the byte, because the PDP-7 was a word- addressed machine. In practice this meant merely that all programs dealing with character streams ignored null characters, because null was used to pad a file to an even number of characters.

```

在Hiew或者FAR Manager中，这些字符串看上去是这样的：

```
int main() {
printf ("Hello, world!\n");
};

```

![](http://drops.javaweb.org/uploads/images/a64ce0497e2bc97a0949873a4a07ca2db9a08a0f.jpg)

#### 57.1.2 Borland Delphi

在Pascal和Borland Delphi中字符串为8-bit或者32-bit长。

举个例子：

```
CODE:00518AC8                 dd 19h
CODE:00518ACC aLoading___Plea db 'Loading... , please wait.',0
...
CODE:00518AFC                 dd 10h
CODE:00518B00 aPreparingRun__ db 'Preparing run...',0

```

#### 57.1.3 Unicode

通常情况下，称Unicode是一种编码字符串的方法，每个字符占用2个字节或者16bit。这是一种常见的术语错误。在许多语言系统中，Unicode是一种用于给每个字符分配数字的标准，而不是用于描述编码的方法。

最常用的编码方法是：UTF-8(在Internet和*NIX系统中使用较多)和UTF-16LE(在Windows中使用)。

**UTF-8**

UTF-8是最成功的字符编码方法之一。所有拉丁符号编码成ASCII，而超出ASCII表的字符的编码使用多个字节。0的编码方式和以前一样，所有的标准C字符串函数处理UTF-8字符串和处理其他字符串一样。

我们来看看不同语言中的符号在UTF-8中是如何被编码的，在FAR中看上去又是什么样的，使用[437内码表](http://go.yurichev.com/17304):

![](http://drops.javaweb.org/uploads/images/42f52a3d3ee7f0cf44e48484d43f9dd0f285f965.jpg)

就像你看到的一样，英语字符串看上去和ASCII编码的一样。匈牙利语使用一些拉丁符号加上变音标志。这些符号使用多个字节编码。我用红色下划线标记出来了。对于冰岛语和波兰语也是一样的。我在开始处使用"Euro"通行符号，编码为3个字节。这里剩下的语言系统与拉丁文没有联系。至少在俄语、阿拉伯语、希伯来语和印地语中我们可以看到相同的字节，这并不稀奇：语言系统的所有符号通常位于同一个Unicode表中，所以他们的号码前几个数字相同。

之前在"How much?"前面，我们看到了3个字节，这实际上是BOM。BOM定义了使用的编码系统。

**UTF-16LE**

在Windows中，许多win32函数带有后缀 -A和-W。第一种类型的函数用于处理普通字符串，第二种类型的函数用于处理UTF-16LE(wide)，每个符号存储类型通常为16比特的short。 UTF-16中拉丁符号在Hiew和FAR中看上去插入了0字节：

```
int wmain() {
wprintf (L"Hello, world!\n");
};

```

![](http://drops.javaweb.org/uploads/images/34a084d1a37954b3d2deb0b84d69b9117e40695c.jpg)

在Windows NT系统中经常可以看见这样的：

![](http://drops.javaweb.org/uploads/images/1e7e3599921bb286dbd086549e7c90afe6beede4.jpg)

在IDA中，占两个字节通常被称为Unicode：

```
.data:0040E000 aHelloWorld:
.data:0040E000                 unicode 0, <Hello, world!>
.data:0040E000                 dw 0Ah, 0

```

下面是俄语字符串在UTF-16LE中如何被编码：

![](http://drops.javaweb.org/uploads/images/840a2b1a8d0573f1a5c76014004e56e6cd0da591.jpg)

容易发现的是，符号被插入了方形字符(ASCII码为4).实际上，西里尔符号位于Unicode第四个平面。因此，在UTF-16LE中，西里尔符号的范围为0x400到0x4FF.

我们回到使用多种语言书写的字符串的例子中吧。下面是他们在UTF-16LE中的样子。

![](http://drops.javaweb.org/uploads/images/3f4f266ff81ff04ee104ab1968b6841ffd1501fb.jpg)

这里我们也能看到开始处有一个BOM。所有的拉丁字符都被插入了一个0字节。我也给一些带有变音符号的字符标注了红色下划线(匈牙利语和冰岛语)。

#### 57.1.4 Base64

Base64编码方法多用于需要将二进制数据以文本字符串的形式传输的情况。实际上，这种算法将3个二进制字节编码为4个可打印字符：所有拉丁字母(包括大小写)、数字、加号、除号共64个字符。 Base64字符串一个显著的特性是他们经常(并不总是)以1个或者2个等号结尾，举个例子：

```
AVjbbVSVfcUMu1xvjaMgjNtueRwBbxnyJw8dpGnLW8ZW8aKG3v4Y0icuQT+qEJAp9lAOuWs=

WVjbbVSVfcUMu1xvjaMgjNtueRwBbxnyJw8dpGnLW8ZW8aKG3v4Y0icuQT+qEJAp9lAOuQ==

```

等号不会在base-64编码的字符串中间出现。

### 57.2 Error/debug messages

调试信息非常有帮助。在某种程度上，调试信息报告了程序当前的行为。通常这些printf类函数，写入信息到log文件中，在release模式下不写任何东西但会显示调用信息。如果本地或全局变量dump到了调试信息中，可能会有帮助，至少能获取变量名。比如在Oracle RDBMS中就有这样一个函数 ksdewt()。

文本字符串常常很有帮助。IDA反汇编器可以展示指定字符串被哪个函数在哪里使用。经常会出现有趣的状况。

错误信息也很有帮助。在Oracle RDBMS中，错误信息会报告使用的一系列函数。

更多相关信息：<blog.yurichev.com>。

快速获知哪个函数在什么情况下报告了错误信息是可以做到的。顺便说一句，这也是copy-protection系统为什么要设置模糊而难懂的错误信息或错误码。没有人会为软件破解者仅仅通过错误信息就快速找到了copy-protection被触发的原因而感到高兴。

一个关于错误信息编码的例子：78.2节

### 57.3 Suspicious magic strings

一些幻数字符串通常使用在后门中，看上去很神秘。举个例子，下面有一个[TP-Link WR740路由器的后门](http://sekurak.pl/tp-link-httptftp-backdoor/)。使用下面的URL可以激活后门：[http://192.168.0.1/userRpmNatDebugRpm26525557/start_art.html](http://192.168.0.1/userRpmNatDebugRpm26525557/start_art.html)。 实际上，"userRpmNatDebugRpm26525557"字符串会在硬件中显示。在后门信息泄漏前，这个字符串并不能被google到。你在任何RFC中都找不到这个。你也无法在任何计算机科学算法中找到使用了这个奇怪字节序列的地方。此外，这看上去也不像错误信息或者调试信息。因此，调查这样一个奇怪字符串的用途是明智的。

有时像这样的字符串可能使用了base64编码。所以解码后再看一遍是明智的，甚至扫一眼就够了。 更确切的说，这种隐藏后门的方法称为“security through obscurity”。

第58章
=====

调用assert
--------

有时，assert()宏的出现也是有用的：通常这个宏会泄漏源文件名，行号和条件。

最有用的信息包含在assert的条件中，我们可以从中推断出变量名或者结构体名。另一个有用的信息是文件名。我们可以从中推断出使用了什么类型的代码。并且也可能通过文件名识别出有名的开源库。

```
.text:107D4B29 mov  dx, [ecx+42h]
.text:107D4B2D cmp  edx, 1
.text:107D4B30 jz   short loc_107D4B4A
.text:107D4B32 push 1ECh
.text:107D4B37 push offset aWrite_c ; "write.c"
.text:107D4B3C push offset aTdTd_planarcon ; "td->td_planarconfig == PLANARCONFIG_CON"...
.text:107D4B41 call ds:_assert
...
.text:107D52CA mov  edx, [ebp-4]
.text:107D52CD and  edx, 3
.text:107D52D0 test edx, edx
.text:107D52D2 jz   short loc_107D52E9
.text:107D52D4 push 58h
.text:107D52D6 push offset aDumpmode_c ; "dumpmode.c"
.text:107D52DB push offset aN30     ; "(n & 3) == 0"
.text:107D52E0 call ds:_assert
...
.text:107D6759 mov  cx, [eax+6]
.text:107D675D cmp  ecx, 0Ch
.text:107D6760 jle  short loc_107D677A
.text:107D6762 push 2D8h
.text:107D6767 push offset aLzw_c   ; "lzw.c"
.text:107D676C push offset aSpLzw_nbitsBit ; "sp->lzw_nbits <= BITS_MAX"
.text:107D6771 call ds:_assert

```

同时google一下条件和文件名是明智的，可能会因此找到开源库。举个例子，如果我们google查找“sp->lzw_nbits <= BITS_MAX”，将会显示一些与LZW压缩有关的开源代码。