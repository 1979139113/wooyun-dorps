# 逆向基础 Finding important/interesting stuff in the code (二)

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

第59章
=====

常量
--

通常人们在生活中或者程序员在编写代码时喜欢使用像10，100，1000这样的整数。

有经验的逆向工程师会对这些数字的十六进制形式很熟悉：10=0xA, 100=0x64, 1000=0x3E8, 10000=0x2710。

常量 0xAAAAAAAA (10101010101010101010101010101010)和0x55555555 (01010101010101010101010101010101)也很常用——构成alternating bits。举个例子，0x55AA在引导扇区，MBR，IBM兼容扩展卡中使用过。

某些算法，特别是密码学方面的使用的常量很有代表性，我们可以在IDA中轻松找到。

举个例子，MD5算法这样初始化内部变量：

```
var int h0 := 0x67452301
var int h1 := 0xEFCDAB89
var int h2 := 0x98BADCFE
var int h3 := 0x10325476

```

如果你在代码中某行发现这四个常量，那么极有可能该处函数与MD5有关。

另一个有关CRC16/CRC32算法的例子，通常使用预先计算好的表来计算：

```
/** CRC table for the CRC-16. The poly is 0x8005 (x^16 + x^15 + x^2 + 1) */
u16 const crc16_table[256] = {
        0x0000, 0xC0C1, 0xC181, 0x0140, 0xC301, 0x03C0, 0x0280, 0xC241,
        0xC601, 0x06C0, 0x0780, 0xC741, 0x0500, 0xC5C1, 0xC481, 0x0440,
        0xCC01, 0x0CC0, 0x0D80, 0xCD41, 0x0F00, 0xCFC1, 0xCE81, 0x0E40,
        ...

```

CRC3预计算表同见：第37节

### 59.1 幻数

许多文件格式定义了标准的文件头，使用了幻数。

举个例子，所有的Win32和MS-DOS可执行文件以"MZ"这两个字符开始。

MIDI文件的开始有"MThd"标志。如果我们有一个使用MIDI文件的程序，它很有可能会检查至少4字节的文件头来确认文件类型。

可以这样实现：

(buf指向内存文件加载的开始处)

```
cmp [buf], 0x6468544D ; "MThd"
jnz _error_not_a_MIDI_file

```

也可能会调用某个函数比如memcmp()或者等同于CMPSB指令(A.6.3节)的代码用于比对内存块。

当你发现这样的地方，你就可以确定的MIDI文件加载的开始处，同时我们可以看到缓冲区存放MIDI文件内容的地方，什么内容被使用以及如何使用。

#### 59.1.1 DHCP

上面的方法对于网络协议也同样适用。举个例子，DHCP协议网络包包含了magic cookie：0x6353826。任何生成DHCP包的代码在某处一定将这个常量嵌入了包中。它在代码中出现的地方可能就与执行这些操作有关，或者不仅是如此。任何接收DHCP的包都会检查这个magic cookie，比对是否相同。

举个例子，我们在Windows 7 x64的dhcpcore.dll文件中搜索这个常量。找到两处：看上去这个常量在名为DhcpExtractOptionsForValidation()和 DhcpExtractFullOptions()函数中使用:

```
.rdata:000007FF6483CBE8 dword_7FF6483CBE8 dd 63538263h ; DATA XREF: ⤦ 
    DhcpExtractOptionsForValidation+79
￼￼￼￼￼￼￼￼￼￼￼.rdata:000007FF6483CBEC dword_7                        DATA XREF: ⤦ 
    DhcpExtractFullOptions+97

```

下面是常量被引用的地址：

```
.text:000007FF6480875F  mov eax, [rsi]
.text:000007FF64808761  cmp eax, cs:dword_7FF6483CBE8
.text:000007FF64808767  jnz loc_7FF64817179

```

还有：

```
.text:000007FF648082C7  mov eax, [r12]
.text:000007FF648082CB  cmp eax, cs:dword_7FF6483CBEC
.text:000007FF648082D1  jnz loc_7FF648173AF

```

### 59.2 搜索常量

在IDA中很容易：使用ALT-B或者ALT-I。如果是在大量文件或者在不可执行文件中搜索常量，我会使用自己编写一个叫[binary grep](http://go.yurichev.com/17017)的小工具。

第60章
=====

寻找合适的指令
-------

如果程序使用了FPU指令但使用不多，你可以尝试用调试器手工逐个检查。

举个例子，我们可能会对用户如何在微软的Excel中输入计算公式感兴趣，比如除法操作。

如果我们加载excel.exe(Offic 2010)版本为14.0.4756.1000 到IDA中，列出所有的条目，查找每一条FDIV指令(除了使用常量作为第二个操作数的——显然不是我们所关心的)：

```
cat EXCEL.lst | grep fdiv | grep -v dbl_ > EXCEL.fdiv

```

然后我们就会看到有144条相关结果。

我们可以在Excel中输入像"=(1/3)"这样的字符串然后对指令进行检查。

通过使用调试器或者tracer(一次性检查4条指令)检查指令，我们幸运地发现目标指令是第14个：

```
.text:3011E919 DC 33        fdiv    qword ptr [ebx]

PID=13944|TID=28744|(0) 0x2f64e919 (Excel.exe!BASE+0x11e919)
EAX=0x02088006 EBX=0x02088018 ECX=0x00000001 EDX=0x00000001
ESI=0x02088000 EDI=0x00544804 EBP=0x0274FA3C ESP=0x0274F9F8
EIP=0x2F64E919
FLAGS=PF IF
FPU ControlWord=IC RC=NEAR PC=64bits PM UM OM ZM DM IM
FPU StatusWord=
FPU ST(0): 1.000000

```

ST(0)存放了第一个参数，[EBX]存放了第二个参数。

FDIV(FSTP)之后的指令在内存中写入了结果：

```
.text:3011E91B DD 1E        fstp    qword ptr [esi]

```

如果我们设置一个断点，就可以看到结果：

```
PID=32852|TID=36488|(0) 0x2f40e91b (Excel.exe!BASE+0x11e91b)
EAX=0x00598006 EBX=0x00598018 ECX=0x00000001 EDX=0x00000001
ESI=0x00598000 EDI=0x00294804 EBP=0x026CF93C ESP=0x026CF8F8
EIP=0x2F40E91B
FLAGS=PF IF
FPU ControlWord=IC RC=NEAR PC=64bits PM UM OM ZM DM IM
FPU StatusWord=C1 P
FPU ST(0): 0.333333

```

我们也可以恶作剧地修改一下这个值：

```
tracer -l:excel.exe bpx=excel.exe!BASE+0x11E91B,set(st0,666)

```

* * *

```
PID=36540|TID=24056|(0) 0x2f40e91b (Excel.exe!BASE+0x11e91b)
EAX=0x00680006 EBX=0x00680018 ECX=0x00000001 EDX=0x00000001
ESI=0x00680000 EDI=0x00395404 EBP=0x0290FD9C ESP=0x0290FD58
EIP=0x2F40E91B
FLAGS=PF IF
FPU ControlWord=IC RC=NEAR PC=64bits PM UM OM ZM DM IM
FPU StatusWord=C1 P
FPU ST(0): 0.333333
Set ST0 register to 666.000000

```

Excel在这个单元中显示666，我们也可以确信的确找到了正确的位置。

![](http://drops.javaweb.org/uploads/images/6866fe94cfe92e33fe0f8b8c0f8b8cdf6b7716d2.jpg)

如果我们尝试使用同样的Excel版本，但是是64位的，会发现只有12个FDIV指令，我们的目标指令在第三个。

```
tracer.exe -l:excel.exe bpx=excel.exe!BASE+0x1B7FCC,set(st0,666)

```

看起来似乎许多浮点数和双精度类型的除法操作都被编译器用SSE指令比如DIVSD(DIVSD总共出现了268次)替换了。

第61章
====

可疑的代码模式
-------

### 61.1 XOR 指令

像XOR op这样的指令，op为寄存器(比如，xor eax，eax)通常用于将寄存器的值设置为零，但如果操作数不同，"互斥或"运算将被执行。在普通的程序中这种操作较罕见，但在密码学中应用较广，包括业余的。如果第二个操作数是一个很大的数字，那么就更可疑了。可能会指向加密/解密操作或校验和的计算等等。

而这种观察也可能是无意义的，比如"canary"(18.3节)。canary的产生和检测通常使用XOR指令。

下面这个awk脚本可用于处理IDA的.list文件：

gawk -e '$2=="xor" { tmp=substr($3, 0, length($3)-1); if (tmp!=$4) if($4!="esp") if ($4!="ebp")⤦ ￼￼￼￼￼￼￼￼