# Skype逆向之旅

from:

*   Skype逆向之旅: 创世纪[http://www.oklabs.net/skype-reverse-engineering-genesis/](http://www.oklabs.net/skype-reverse-engineering-genesis/)
*   Skype逆向之旅：分析篇[http://www.oklabs.net/skype-reverse-engineering-the-long-journey/](http://www.oklabs.net/skype-reverse-engineering-the-long-journey/)

0x00 背景
=====

我想...在显示分析之前先交代一下背景信息。

But，我得先为我糟糕的英语先道个歉。

在2k6的时候，我在一个法国公司主持一些逆向工程项目，建立一套转有的技术体系。但是这些技术当中最好的/最大的收获便是逆向Skype协议获得的经验。

一年后，由于该公司的战略调整该项目被中止。我离开我的工作，我希望有一天能够继续完成它。虽然在几年后，都没有怎么接触过该项目..

我决定今天发布该项工作，即使它还比较原始。但是它可能会帮助任何其它想学习该协议或满足其好奇心的人。我觉得这比在出租屋的房子里睡上一年觉还美好。

在我离开的时候，逆向工程完成了：能够连接到Skype的分布式网络，认证用户，获取联系人列表，管理用户的存在和接收（响应）文本聊天。

在这里我分享我在该项目中遇到的一些困难，一些工作笔记，逆向过程中的一些工具（包括我自己开发的），并发布一个概念型客户端（名为FakeSkype）和它的事件驱动（已停止开发，名为Epyks）。

我说过，这个项目开始于2k6年，因此，使用的是Skype V2.5客户端协议。

这个项目使用的是Windows XP 32位系统下。

下面是我收集的一些比较好的论文。

*   Silver Needle In The Skype : Philippe Biondi & Fabrice Desclaux
*   Castle In The Skype : Fabrice Desclaux
*   [Vanilla Skype Part I](http://www.oklabs.net/wp-content/uploads/2012/06/vskype-part1.pdf)&[Vanilla Skype Part II](http://www.oklabs.net/wp-content/uploads/2012/06/vskype-part2.pdf): Fabrice Desclaux Kostya Kortchinsky
*   [Skype, Guide for Network Administrators](http://www.oklabs.net/wp-content/uploads/2012/06/guide-for-network-admins-30beta.pdf): Skype Inc.
*   [Malicious Crypto](http://www.oklabs.net/wp-content/uploads/2012/06/esw06-raynal.pdf): Frederic Raynal

首先，我们应该知道从网络流量逆向工程Skype协议在一些国家是合法的，而检查/反汇编二进制文件在Skype的许可协议条款中是禁止的。但是针对文件格式和协议的互操作性有合法的先例。此外，一些国家特别许可那些通过逆向工程而重写的再工程程序（去除复制保护）。

0x01 简要
=====

首先，Skype的网络和现有的网络完全不同，其主要区别在于它的架构上。Skype客户端是基于peer-to-peer网络模型而不是基于经典的client-server结构的VoIP客户端。这就意味着每个Spype客户端涉及到用户间通讯不涉及到集中式的架构。每个Skype客户端同时作为服务端和客户端使用。在一个P2P网络当中，每个客户端被设计成一个“节点”。

_Server Based Network_

![](http://drops.javaweb.org/uploads/images/3fa3c357556c633f0553c8d7fdc0200977e1d9cc.jpg)

_P2P Network_

![](http://drops.javaweb.org/uploads/images/d4bbacdfa393352645db93125937397ad866b5bb.jpg)

Skype有其特定的P2P架构，因为它必须适用所使用的网络情况。举个例子，与Kazaa被设计成文件共享的P2P网络不同，Skype网络被优化到可以传输实时数据。Skype协议实现了用户安全认证，动态联系人列表管理和确保隐私。

Skype的用户目录完全分布在网络中节点之间。这意味着该网络可以很容易得到扩容，且不需要一个复杂昂贵的集中式架构。

Skype还通过其它Skype节点路由通信网络以减缓NAT和防火墙穿越的压力。然而，这会对那些直接连到Internet的用户造成额外的负担。因为他们的计算机和网络带宽还用于路由其他用户的通信。

0x02 逆向工程
=====

正如前面所说，Skype客户端实现的Skype通讯协议带有多重的保护，以防止逆向工程。为了完全研究该协议，我的第一个目标就是清除二进制文件上的每个保护，让我可以从内部观察该协议的工作流程。有许多研究人员试图研究/逆向工程Skype协议而开发第三方客户端，但由于没能够进一步充分地解剖这个协议，导致现在还没有成功过的案例。这里有一个Skype协议分析的知识库：[http://www1.cs.columbia.edu/~salman/skype/](http://www1.cs.columbia.edu/~salman/skype/)。

最完整的论文来自EADS-CCR实验室的两个研究人员，分别是Phillipe Biondi和Fabrice Desclaux。他们公开的研究成果：“Silver Needle in the Skype”带来了绕过Skype主要保护的曙光，同时也透露着该项工作的复杂性。在此基础上我找到了一种方法来绕过这些保护措施。

作为一个闭源程序，分析之前需要把二进制文件翻译成x86会变代码。在此我使用的是众所周知的“OllyDbg（v1.10）”与“Phant0m”插件。这个工具可以加载客户端程序并观察它的汇编代码，还可以通过设置断点单步执行汇编指令。我还使用另一个出名的工具“IDA（v5.00）”。协议分析方面我主要用“OSpy（v1.9.0.0）”。这个工具允许我在二进制文件执行过程中设置指定的hook，以便收集发送到网络的重要数据。

以上就是分析时所需要的工具，总体上，我遇到了以下几种保护：

*   二进制文件的敏感代码使用一个硬编码的密钥进行加密以避免代码被静态分析。Skype客户端（SC）开始处有一个程序负责执行过程中解密敏感代码和加密部分代码。之后，解密器处理刚加密的代码，并擦除另一部分代码。后者的作用是为了防止解密的代码保存在计算机内存中。我当时设计了一个工具，用于从计算机存储器中完整提取出SC程序在运行过程中解密出来的代码，一旦解密程序解密完成，便把解密代码覆盖到保存在硬盘上的SC二进制文件中原有的加密代码。并防止解密程序在解密完成后的进行代码擦除。该程序还会修改导入表，该表包含了程序所使用的每一个导入函数。我设计的那个工具也设法把导入表的修改操作同步到磁盘上的二进制文件上。
    
    _SC Binary Behavior_
    
    ![](http://drops.javaweb.org/uploads/images/e31f19af026e97f6d38066b325835f5ce5cd3ee7.jpg)
    
    ```
    Main lines of the tool to restore encrypted & scrambled code (dirty)
    /* Skype Sucker */
    /* Author : Ouanilo MEDEGAN */        
    
    STARTUPINFO si;
    PROCESS_INFORMATION pi;
    HANDLE hProcess = 0;
    char *Buffer;
    char *BigBuffer;
    long Size;
    FILE *result;        
    
    result = fopen("Skype.exe", "rb");
    fseek(result, 0, SEEK_END);
    Size = ftell(result);
    fseek(result, 0, SEEK_SET);
    BigBuffer = malloc(Size);
    fread(BigBuffer, Size, 1, result);
    fclose(result);        
    
    Buffer = malloc(0x49F000);
    ZeroMemory(Buffer, 0x49F000);
    ZeroMemory(&si, sizeof(si));
    si.cb = sizeof(si);
    hProcess = OpenProcess(PROCESS_VM_READ,FALSE, 0xF0C);
    ReadProcessMemory(hProcess, (LPVOID)0x01FB0000, Buffer, 0x49F000,NULL);
    result = fopen("Skype.exe", "wb");
    memcpy(BigBuffer + 3429792, Buffer, 0x49F000);
    fwrite(BigBuffer, Size, 1, result);
    fclose(result);        
    
    CloseHandle(hProcess);
    system("PAUSE");
    return 0;
    
    ```
*   SC程序也执行一些程序以在运行时探测调试器，并在探测到调试器的时候停止执行程序。我没有使用隐藏调试器的补丁，因为OllyDbg（奇怪）是不在SC的调试器黑名单中。事实上，它只是停止一个调试器，就是老且臭名昭著的“SoftICE”，并且使用各种加密字符串技巧进行检测。
    
    _OllyDbg View_
    
    ![](http://drops.javaweb.org/uploads/images/d5ca2f1d1a41275f9cdf014eafa216bacef6dc1c.jpg)
    
*   现在来看一下SC最有效的保护机制。实际上代码里包含了约300处多态完整性代理（polymorphic integrity agents）。它们的功能是：随机执行，并负责检查并校验各部分代码的完整性，防止在敏感代码上打补丁，甚至可以防止在代码执行过程中设置断点，就如在调试器中插入的软件断点时，在计算机内存中改变其进程运行时的镜像代码。绕过这些代理的主要困难是，它们都来互不相同，如大小，检查方式和检查标准（例如，检查的结果可以用来确定代码的下一个执行路径，或者是另一个代理的地址）。然而，经过深入的分析，发现它们存在相同的设计思想，它们分别是：
    
    1.  初始化地址
    2.  循环
    3.  查找
    4.  测试/计算
    
    Skype Checksum agents generic patterns
    
    ```
    MOV R32, CONST ;Pattern #1
    ANY 8
    MOV R32, R32
    ANY 8
    MOV R32, [R32+CONST]
    JMP OFFSET
    
    XOR R32, CONST ;Pattern #2
    ANY 8
    ANY 8
    ANY 8
    MOV R32, [R32+CONST]
    JMP OFFSET
    
    XOR R32, R32 ;Pattern #3
    ANY 8
    MOV R32, [R32+CONST]
    ANY 8
    DEC R32
    JNZ OFFSET
    
    XOR R32, R32 ;Pattern #4
    ANY 8
    MOV R32, CONST
    ANY 8
    DEC R32
    JNZ OFFSET
    
    ```
    
    意识到这一点，我想我能够使用OllyDbg通过一定的模式找出总体上几乎所有代理。一旦得到它们所有特征点，便使用一个调教好的x86模拟器（名为“x86emu”的IDA插件）以等待每个特征点代理的执行结果。然后，我还设计了一个OD插件（使用插件开发工具包）给SC二进制的代理打上完整的补丁，让它们在最终测试之前直接设置好预期的值。我想我已经能够修复二进制代码，而且可以在代码的每个部分上设置断点，并且还获得一个不需要执行数百个控制点的更快的SC二进制文件。
    
    _Checksum Agent View Exemple#1_
    
    ![](http://drops.javaweb.org/uploads/images/33a57f0595dc0ca7c552e09cf460d96f872de2d6.jpg)
    
    _Checksum Agent View Exemple#2_
    
    ![](http://drops.javaweb.org/uploads/images/0490b311143537cce22ecb9398d730027f6cda4c.jpg)
    
    _Tuned x86 emulator window after execution on spotted agents_
    
    ![](http://drops.javaweb.org/uploads/images/920685e1b83e6d7a03f1f4626d7f2ffb8ffed655.jpg)
    
    Skype Checksum agents Patcher Plugin
    

```
/* Skype Checksum Agents Patcher Plugin */
        /* Author : Ouanilo MEDEGAN */        

        // VERY IMPORTANT NOTICE: COMPILE THIS DLL WITH BYTE ALIGNMENT OF STRUCTURES
        // AND UNSIGNED CHAR!        
        

        #include "plugin.h"        

        HINSTANCE hinst; // DLL instance
        HWND hwmain; // Handle of main OllyDbg window        

        typedef struct patchs_s
        {
            unsigned int wtp;
            unsigned int end;
            char *code;
        } patchs;        

        int checks[] = { 0x74ffdb,
                        0x754220,
                        0x7566d5,
                        0x758331,
                        0x75915e,
                        0x7602df,
                        0x760477,
                        0x7706ed,
                        0x770bf9,
                        0x770ff3,
                        0x771fa5,
                        0x7729b4,
                        0x772a5f,
                        0x77ad1c,
                        0x77baef,
                        0x77d053,
                        0x77ec72,
                        0x780dcd,
                        0x781652,
                        0x782810,
                        0x783412,
                        0x783d58,
                        0x78566c,
                        0x785a82,
                        0x786142,
                        0x7881b7,
                        0x788302,
                        0x7907f5,
                        0x7a2b6e,
                        0x7a2f1d,
                        0x7a34d5,
                        0x7a36ca,
                        0x7a8cfc,
                        0x7a9784,
                        0x7aa229,
                        0x7ac1b6,
                        0x7aede1,
                        0x7af68d,
                        0x7b22ee,
                        0x7b319e,
                        0x7b6d55,
                        0x7b6f55,
                        0x7b7711,
                        0x7b7b19,
                        0x7b8dda,
                        0x7b93c0,
                        0x7bc108,
                        0x7bdeef,
                        0x7be304,
                        0x7bee1d,
                        0x7befaf,
                        0x7bf322,
                        0x7c2b0e,
                        0x7c427d,
                        0x7c4c73,
                        0x7c5d71,
                        0x7c71f0,
                        0x7c7f23,
                        0x7c8e97,
                        0x7cbc3d,
                        0x7d75ca,
                        0x7d8388,
                        0x7da2ce,
                        0x7db581,
                        0x7dcac3,
                        0x7deed3,
                        0x7e502c,
                        0x7e5e8d,
                        0x7e696c,
                        0x7e866b,
                        0x7eb1a9,
                        0x7eb76d,
                        0x7ed530,
                        0x7f5a45,
                        0x7f9b84,
                        0x7f9e99,
                        0x7fa29e,
                        0x7fd27d,
                        0x7fd846,
                        0x7fe497,
                        0x7fec0a,
                        0x7ff609,
                        0x7ffdf1,
                        0x7ffef1,
                        0x800128,
                        0x800deb,
                        0x8014d3,
                        0x802276,
                        0x802893,
                        0x803709,
                        0x804925,
                        0x804eef,
                        0x805005,
                        0x8058fa,
                        0x8066e4,
                        0x806c60,
                        0x806d26,
                        0x806dd4,
                        0x806ec4,
                        0x807702,
                        0x808bab,
                        0x809287,
                        0x809aee,
                        0x809d70,
                        0x80af22,
                        0x80bc37,
                        0x80bf48,
                        0x80c627,
                        0x80c854,
                        0x80c994,
                        0x80cad7,
                        0x80cc87,
                        0x80d0c7,
                        0x80d1c7,
                        0x80d3f9,
                        0x812457,
                        0x813316,
                        0x813c0f,
                        0x814547,
                        0x815a1b,
                        0x846e01,
                        0x8497cb,
                        0x849cb2,
                        0x84a148,
                        0x84d824,
                        0x84da14,
                        0x84ef26,
                        0x854042,
                        0x85489a,
                        0x8552bf,
                        0x861ac8,
                        0x861f18,
                        0x86329a,
                        0x863c8a,
                        0x864185,
                        0x8670c7,
                        0x867d00,
                        0x868400,
                        0x868919,
                        0x869075,
                        0x869e2f,
                        0x86a514,
                        0x86d1de,
                        0x8779db,
                        0x877a5e,
                        0x877f2b,
                        0x87831f,
                        0x87b74f,
                        0x88426a,
                        0x89131c,
                        0x892293,
                        0x892500,
                        0x89453b,
                        0x8986f3,
                        0x898828,
                        0x899162,
                        0x89b328,
                        0x89e972,
                        0x8a3455,
                        0x8a62c7,
                        0x8a71b6,
                        0x8a78cd,
                        0x8a8627,
                        0x8ac0a0,
                        0x8acdba,
                        0x8b1383,
                        0x8b1dcc,
                        0x8b42a7,
                        0x8b48fb,
                        0x8b5c31,
                        0x8ba8eb,
                        0x8bab10,
                        0x8bbfa7,
                        0x8be61a,
                        0x8c3ff0,
                        0x8c90b4,
                        0x8cb8dc,
                        0x8cbaa2,
                        0x8cc105,
                        0x8ce0ef,
                        0x8d01df,
                        0x8d0a41,
                        0x8d1a85,
                        0x8d26c7,
                        0x8d4731,
                        0x8d4efc,
                        0x8d5707,
                        0x8dc0d1,
                        0x8dcb50,
                        0x8e0dd6,
                        0x8e14c4,
                        0x8e227d,
                        0x8e2564,
                        0x8e3722,
                        0x8e4ede,
                        0x8e4fb7,
                        0x8e6b3f,
                        0x8e77b7,
                        0x8e8a34,
                        0x8e9935,
                        0x8ea9ba,
                        0x8ebfc5,
                        0x8ee0c2,
                        0x8ee2b4,
                        0x8ee718,
                        0x8ef236,
                        0x8ef86e,
                        0x8efd29,
                        0x8f01b6,
                        0x8f0b49,
                        0x8f11a1,
                        0x905450,
                        0x924cde,
                        0x9261D5,
                        0x928aec,
                        0x928e78,
                        0x93ae8e,
                        0x93d1e0,
                        0x93f37e,
                        0x940266,
                        0x940577,
                        0x940a3c,
                        0x940ba3,
                        0x946a4d,
                        0x9484b4,
                        0x0 };        

        patchs ptab[] = {
            { 0x74fff9, 0x750009, "MOV ECX, 0c445513e" },
            { 0x75423e, 0x754251, "MOV EDI, 07a51760" },
            { 0x7566ed, 0x7566fd, "MOV ECX, 07f26c1cd" },
            { 0x75834e, 0x75835d, "MOV EDX, 08143833c" },
            { 0x75917b, 0x75918a, "MOV EDI, 0ac10c50d" },
            { 0x7602fd, 0x76030d, "MOV EDI, 07b43df78" },
            { 0x760491, 0x7604a2, "MOV EAX, 013c45465" },
            { 0x770707, 0x770717, "MOV EDX, 02d86e921" },
            { 0x770c19, 0x770c2c, "MOV ECX, 023d836c7" },
            { 0x77100d, 0x77101f, "MOV ESI, 0b7d7ee34" },
            { 0x771fc5, 0x771fd6, "MOV EAX, 08ffdfc46" },
            { 0x7729d4, 0x7729e7, "MOV EAX, 0aa2552d6" },
            { 0x772a7d, 0x772a8d, "MOV EDX, 0dd02dff5" },
            { 0x77ad3a, 0x77ad4b, "MOV EBX, 01f1e9f7b" },
            { 0x77bb0c, 0x77bb1b, "MOV EAX, 056ea89cf" },
            { 0x77d073, 0x77d07f, "MOV ESI, 07d4e5bf" },
            { 0x77ec8d, 0x77eca0, "MOV ECX, 09c78fcfe" },
            { 0x780de7, 0x780df7, "MOV EDI, 02ca58b26" },
            { 0x78166f, 0x781682, "MOV EAX, 02d18848f" },
            { 0x78282b, 0x78283e, "MOV ECX, 0d2d3a1f5" },
            { 0x78342d, 0x78343d, "MOV EDI, 0aa0d1086" },
            { 0x783d76, 0x783d86, "MOV EDI, 054d99c86" },
            { 0x785687, 0x785696, "MOV EDI, 09c9fcf4b" },
            { 0x785aa0, 0x785ab3, "MOV ECX, 03a77aa74" },
            { 0x786162, 0x786173, "MOV EDI, 024367f84" },
            { 0x7881d4, 0x7881e5, "MOV EAX, 0764de354" },
            { 0x78831d, 0x78832f, "MOV EBX, 0c0aeb096" },
            { 0x790812, 0x790822, "MOV EBX, 0e7393655" },
            { 0x7a2b8c, 0x7a2b9c, "MOV ESI, 02cbbbb5a" },
            { 0x7a2f41, 0x7a2f4e, "MOV EBX, 0995c72e3" },
            { 0x7a34f5, 0x7a3505, "MOV EAX, 01fe3bec1" },
            { 0x7a36e2, 0x7a36f5, "MOV EDI, 0d4027302" },
            { 0x7a8d19, 0x7a8d29, "MOV EBX, 0518f95ba" },
            { 0x7a97a8, 0x7a97b8, "MOV EDI, 0c0bad4af" },
            { 0x7aa24a, 0x7aa25a, "MOV EDX, 086760316" },
            { 0x7ac1ce, 0x7ac1dc, "MOV EDI, 0e6503aaa" },
            { 0x7aedfb, 0x7aee0e, "MOV EAX, 02091582b" },
            { 0x7af6aa, 0x7af6b9, "MOV ESI, 0aab731b2" },
            { 0x7b2309, 0x7b2319, "MOV ECX, 0336b4c77" },
            { 0x7b31c1, 0x7b31d0, "MOV EAX, 0a9dc7b7e" },
            { 0x7b6d75, 0x7b6d85, "MOV EAX, 033156dcb" },
            { 0x7b6f6f, 0x7b6f7f, "MOV EDI, 068c30e6" },
            { 0x7b7731, 0x7b7744, "MOV EAX, 03a739a6c" },
            { 0x7b7b36, 0x7b7b49, "MOV EBX, 0dd5eb0ac" },
            { 0x7b8df4, 0x7b8e06, "MOV ECX, 0cda6236b" },
            { 0x7b93e1, 0x7b93f1, "MOV EDX, 0909f211e" },
            { 0x7bc122, 0x7bc12f, "MOV EDX, 0423c1dd2" },
            { 0x7bdf0a, 0x7bdf1a, "MOV EDX, 0dd8ff70f" },
            { 0x7be324, 0x7be334, "MOV EAX, 09cfc6204" },
            { 0x7bee37, 0x7bee47, "MOV EAX, 08033cb72" },
            { 0x7befd0, 0x7befe0, "MOV EBX, 0dd57229d" },
            { 0x7bf33d, 0x7bf34b, "MOV EDI, 0303d25eb" },
            { 0x7c2b2b, 0x7c2b3a, "MOV ESI, 0e5ea18bb" },
            { 0x7c4298, 0x7c42a9, "MOV EDI, 0f9cead47" },
            { 0x7c4c8e, 0x7c4c9f, "MOV ESI, 09ccb01a2" },
            { 0x7c5d8b, 0x7c5d9c, "MOV ECX, 09d859997" },
            { 0x7c720e, 0x7c721e, "MOV EBX, 0d0a84b9f" },
            { 0x7c7f40, 0x7c7f4f, "MOV EDI, 0d3e4f2c7" },
            { 0x7c8eb4, 0x7c8ec6, "MOV ECX, 0427b5851" },
            { 0x7cbc5a, 0x7cbc6d, "MOV EAX, 02d194eb0" },
            { 0x7d75e4, 0x7d75f4, "MOV EBX, 0b024bef9" },
            { 0x7d83a8, 0x7d83b9, "MOV EBX, 0547f095b" },
            { 0x7da2ef, 0x7da2ff, "MOV ESI, 0e5ce5b73" },
            { 0x7db59e, 0x7db5b0, "MOV ECX, 099b9f3a8" },
            { 0x7dcae3, 0x7dcaf1, "MOV EAX, 0b77e8d82" },
            { 0x7deef0, 0x7def03, "MOV EAX, 0e734dc4e" },
            { 0x7e5044, 0x7e5057, "MOV EDX, 07febc729" },
            { 0x7e5eaa, 0x7e5eb7, "MOV EDI, 0ac34b755" },
            { 0x7e698a, 0x7e699d, "MOV ECX, 030143199" },
            { 0x7e8688, 0x7e8699, "MOV EAX, 0149e7219" },
            { 0x7eb1c6, 0x7eb1d8, "MOV EAX, 0929f271f" },
            { 0x7eb78e, 0x7eb79e, "MOV EDX, 08b216d4a" },
            { 0x7ed54b, 0x7ed55e, "MOV ESI, 08017bf81" },
            { 0x7f5a62, 0x7f5a71, "MOV EAX, 099f2503f" },
            { 0x7f9ba4, 0x7f9bb5, "MOV EAX, 05175280f" },
            { 0x7f9eb3, 0x7f9ec3, "MOV EDI, 0a955c470" },
            { 0x7fa2bb, 0x7fa2c8, "MOV EAX, 02dfc1afa" },
            { 0x7fd298, 0x7fd2a7, "MOV ECX, 01f5d41fc" },
            { 0x7fd863, 0x7fd872, "MOV ESI, 0e0812fb9" },
            { 0x7fe4b2, 0x7fe4c0, "MOV EDI, 099e325dd" },
            { 0x7fec24, 0x7fec35, "MOV EBX, 054b340c3" },
            { 0x7ff629, 0x7ff63a, "MOV EDX, 0b45090bf" },
            { 0x7ffe0f, 0x7ffe1d, "MOV ECX, 0e5bc5382" },
            { 0x7fff12, 0x7fff22, "MOV ECX, 08ac85e98" },
            { 0x800146, 0x800157, "MOV ESI, 090201620" },
            { 0x800e05, 0x800e17, "MOV ECX, 0e27686f6" },
            { 0x8014ee, 0x801500, "MOV ECX, 0d2c7b21d" },
            { 0x802290, 0x80229e, "MOV EAX, 0ec8ae7f8" },
            { 0x8028b1, 0x8028c3, "MOV ESI, 054b9f847" },
            { 0x803729, 0x803737, "MOV EBX, 0cddb39d5" },
            { 0x804943, 0x804956, "MOV ESI, 013b8244d" },
            { 0x804f0c, 0x804f1c, "MOV EAX, 06c9aae5b" },
            { 0x805022, 0x805032, "MOV ESI, 055070ae1" },
            { 0x805918, 0x80592b, "MOV EBX, 030124195" },
            { 0x806705, 0x806714, "MOV EBX, 0147affd2" },
            { 0x806c7e, 0x806c8c, "MOV EDX, 06e536285" },
            { 0x806d43, 0x806d53, "MOV ECX, 07a9b5d13" },
            { 0x806df1, 0x806e00, "MOV EAX, 054a8a4b4" },
            { 0x806ee8, 0x806ef6, "MOV EBX, 02e42c699" },
            { 0x80771d, 0x807730, "MOV EBX, 0e638907b" },
            { 0x808bce, 0x808be1, "MOV EDX, 0f9e77f79" },
            { 0x8092a8, 0x8092b6, "MOV EDX, 06efe8faf" },
            { 0x809b0b, 0x809b1b, "MOV EDX, 034f306d1" },
            { 0x809d8a, 0x809d9d, "MOV EAX, 0acf4b4d5" },
            { 0x80af39, 0x80af46, "MOV EDI, 071094df2" },
            { 0x80bc57, 0x80bc68, "MOV EBX, 0b4e5d2d2" },
            { 0x80bf65, 0x80bf72, "MOV ESI, 076f6bf82" },
            { 0x80c644, 0x80c656, "MOV EDI, 0e3df1200" },
            { 0x80c874, 0x80c887, "MOV EAX, 090c5b6fe" },
            { 0x80c9af, 0x80c9be, "MOV EBX, 026599582" },
            { 0x80caf5, 0x80cb02, "MOV EDX, 030a145e7" },
            { 0x80cca8, 0x80ccb8, "MOV ESI, 06bc5d1d1" },
            { 0x80d0e8, 0x80d0f8, "MOV ESI, 055c8cecf" },
            { 0x80d1e5, 0x80d1f4, "MOV EDI, 0ffb2b709" },
            { 0x80d419, 0x80d42b, "MOV ESI, 052956b56" },
            { 0x812478, 0x81248a, "MOV ESI, 0c51f8d42" },
            { 0x813333, 0x813344, "MOV ECX, 018b4b2ca" },
            { 0x813c2f, 0x813c3e, "MOV EDI, 0fb585983" },
            { 0x814568, 0x814577, "MOV EBX, 0a393b14" },
            { 0x815a39, 0x815a49, "MOV ECX, 01552376d" },
            { 0x846e1c, 0x846e29, "MOV EDX, 0895a8f12" },
            { 0x8497e9, 0x8497f9, "MOV ESI, 03c8b3592" },
            { 0x849ccc, 0x849cdf, "MOV EAX, 03fbb1f8a" },
            { 0x84a165, 0x84a172, "MOV EBX, 0e036d6b9" },
            { 0x84d842, 0x84d855, "MOV EDX, 021242c95" },
            { 0x84da31, 0x84da42, "MOV EAX, 0fedb8372" },
            { 0x84ef44, 0x84ef54, "MOV EBX, 0ae164cd1" },
            { 0x85405f, 0x85406f, "MOV EBX, 09a52989" },
            { 0x8548b4, 0x8548c7, "MOV EAX, 0ac0639e0" },
            { 0x8552d9, 0x8552ea, "MOV EAX, 091c4f5b9" },
            { 0x861ae8, 0x861afa, "MOV ESI, 093840a1c" },
            { 0x861f36, 0x861f45, "MOV ESI, 0419edd7b" },
            { 0x8632b1, 0x8632c1, "MOV EBX, 085ab7503" },
            { 0x863ca5, 0x863cb8, "MOV EDX, 07dbfb5b" },
            { 0x8641a6, 0x8641b6, "MOV ECX, 0e173e1e9" },
            { 0x8670e5, 0x8670f8, "MOV ECX, 0e0b9151f" },
            { 0x867d1e, 0x867d2e, "MOV EBX, 09d26fa68" },
            { 0x86841d, 0x86842a, "MOV ESI, 012ec7977" },
            { 0x868939, 0x86894b, "MOV EDX, 068904b7c" },
            { 0x869090, 0x86909e, "MOV EDX, 0f1c6291c" },
            { 0x869e49, 0x869e5c, "MOV EAX, 0f1d2405f" },
            { 0x86a52b, 0x86a53d, "MOV ESI, 06193d267" },
            { 0x86d1f8, 0x86d206, "MOV EAX, 0ebf84584" },
            { 0x8779f8, 0x877a07, "MOV ESI, 0f70a3528" },
            { 0x877a7c, 0x877a8a, "MOV EDX, 0e550009b" },
            { 0x877f4e, 0x877f5d, "MOV EAX, 0cd8400bd" },
            { 0x87833c, 0x87834f, "MOV EBX, 0b44b3cde" },
            { 0x87b769, 0x87b777, "MOV EDX, 04db966e" },
            { 0x884288, 0x884298, "MOV EDX, 01553793c" },
            { 0x89133a, 0x89134a, "MOV ESI, 02a7014c9" },
            { 0x8922ab, 0x8922be, "MOV EBX, 0c38e0a6" },
            { 0x89251d, 0x89252f, "MOV EBX, 0c98a0cfb" },
            { 0x894556, 0x894565, "MOV EBX, 086018a98" },
            { 0x898714, 0x898724, "MOV EDX, 0376c710b" },
            { 0x898845, 0x898853, "MOV ECX, 0f3b765ab" },
            { 0x89917f, 0x89918c, "MOV EAX, 0d6e5d43b" },
            { 0x89b342, 0x89b354, "MOV EDX, 059941057" },
            { 0x89e98d, 0x89e99d, "MOV ESI, 0494ffa1f" },
            { 0x8a3476, 0x8a3487, "MOV EDI, 0d2b785" },
            { 0x8a62ea, 0x8a62fd, "MOV EBX, 08bd76a45" },
            { 0x8a71ce, 0x8a71dc, "MOV EDX, 0a210fda6" },
            { 0x8a78eb, 0x8a78fb, "MOV ECX, 0540c5ef9" },
            { 0x8a8645, 0x8a8655, "MOV EDX, 0627f206c" },
            { 0x8ac0bd, 0x8ac0cc, "MOV ESI, 08e981340" },
            { 0x8acdd5, 0x8acde5, "MOV EDI, 0febe5a7c" },
            { 0x8b13a4, 0x8b13b4, "MOV ECX, 013ce0803" },
            { 0x8b1de9, 0x8b1dfb, "MOV ECX, 04ea685c7" },
            { 0x8b42c4, 0x8b42d5, "MOV EDX, 0476254ba" },
            { 0x8b4919, 0x8b4928, "MOV EBX, 0f84be5e4" },
            { 0x8b5c49, 0x8b5c59, "MOV EDX, 0609bbc80" },
            { 0x8ba905, 0x8ba917, "MOV EDI, 05d22fa5c" },
            { 0x8bab30, 0x8bab42, "MOV EAX, 05be049f2" },
            { 0x8bbfc1, 0x8bbfd1, "MOV EBX, 011ac3604" },
            { 0x8be635, 0x8be645, "MOV EDX, 016c52865" },
            { 0x8c400e, 0x8c401d, "MOV ECX, 0ed9cd2db" },
            { 0x8c90cb, 0x8c90db, "MOV ECX, 027208727" },
            { 0x8cb8fa, 0x8cb90b, "MOV EDI, 0e4a5842d" },
            { 0x8cbabd, 0x8cbad0, "MOV EBX, 0bed28bd1" },
            { 0x8cc120, 0x8cc130, "MOV EBX, 0d4f92d37" },
            { 0x8ce109, 0x8ce119, "MOV ECX, 0c3b20a27" },
            { 0x8d01f7, 0x8d0208, "MOV EDX, 0238a774f" },
            { 0x8d0a5f, 0x8d0a6e, "MOV EDX, 02d698aea" },
            { 0x8d1a9c, 0x8d1aa9, "MOV EAX, 074ec115a" },
            { 0x8d26e1, 0x8d26f0, "MOV ECX, 078b53a47" },
            { 0x8d4752, 0x8d4762, "MOV EBX, 0e8dcb02c" },
            { 0x8d4f17, 0x8d4f26, "MOV ESI, 0733cb8e2" },
            { 0x8d5721, 0x8d5731, "MOV EAX, 07351160b" },
            { 0x8dc0ee, 0x8dc0fd, "MOV EDX, 02337f2eb" },
            { 0x8dcb7e, 0x8dcb8c, "MOV ECX, 055899497" },
            { 0x8e0df1, 0x8e0e01, "MOV ESI, 0911efc6" },
            { 0x8e14e2, 0x8e14f2, "MOV EBX, 0918e6486" },
            { 0x8e2297, 0x8e22a8, "MOV EDX, 0ed078080" },
            { 0x8e257f, 0x8e258d, "MOV EDI, 0d79f6837" },
            { 0x8e3745, 0x8e3756, "MOV EAX, 04ff61b79" },
            { 0x8e4efb, 0x8e4f0c, "MOV ESI, 0951858ca" },
            { 0x8e4fd1, 0x8e4fe1, "MOV EAX, 0373fa366" },
            { 0x8e6b59, 0x8e6b6c, "MOV EAX, 0a4e1d4fa" },
            { 0x8e77d1, 0x8e77df, "MOV EDI, 0df1e173d" },
            { 0x8e8a4e, 0x8e8a5d, "MOV EDI, 08e10e996" },
            { 0x8e9953, 0x8e9963, "MOV ECX, 0777fadde" },
            { 0x8ea9d7, 0x8ea9e9, "MOV ECX, 0f3eadf10" },
            { 0x8ebfe3, 0x8ebff3, "MOV ESI, 050667a1a" },
            { 0x8ee0dc, 0x8ee0ef, "MOV EAX, 01f3501a3" },
            { 0x8ee2d5, 0x8ee2e6, "MOV EDX, 055aad44f" },
            { 0x8ee738, 0x8ee747, "MOV ESI, 0b2fbff36" },
            { 0x8ef253, 0x8ef266, "MOV EAX, 06ad8ec8d" },
            { 0x8ef88f, 0x8ef8a1, "MOV ESI, 025480f72" },
            { 0x8efd44, 0x8efd55, "MOV EDX, 0a4c2184c" },
            { 0x8f01d3, 0x8f01e3, "MOV EDX, 07442d802" },
            { 0x8f0b6d, 0x8f0b7b, "MOV EDX, 03a326175" },
            { 0x8f11b9, 0x8f11c8, "MOV EBX, 04a8ffb13" },
            { 0x90546e, 0x90547e, "MOV EDI, 09c89e0af" },
            { 0x924cff, 0x924d0f, "MOV ESI, 092e232d3" },
            { 0x9261f2, 0x926204, "MOV EDI, 0189bc8ff" },
            { 0x928b10, 0x928b21, "MOV EBX, 07a13a572" },
            { 0x928e93, 0x928ea6, "MOV EDX, 0c026ebb8" },
            { 0x93aea9, 0x93aeb9, "MOV EDI, 01131e7ed" },
            { 0x93d201, 0x93d20f, "MOV EBX, 064a2d76a" },
            { 0x93f39b, 0x93f3ae, "MOV ESI, 0bbe3e8c3" },
            { 0x940284, 0x940294, "MOV ESI, 0a22eb245" },
            { 0x94058e, 0x94059d, "MOV EBX, 02d33939c" },
            { 0x940a5c, 0x940a6c, "MOV EAX, 0456de1a1" },
            { 0x940bbb, 0x940bce, "MOV ESI, 0deffe75a" },
            { 0x946a64, 0x946a72, "MOV ESI, 0aa8e9c06" },
            { 0x9484d4, 0x9484e5, "MOV EDX, 08e7e081c" },
            { 0, 0, NULL }
        };        

        void *build_ext_model(char *cmd_sequence)
        {
            t_extmodel *diamod;
            t_extmodel *pmod;
            char diaasm[ARGLEN];
            char cmd[ARGLEN];
            int n, nseq, errpos, offset, i, j, k, m, good, validcount;
            char *pa;
            char errtext[512];
            char t[512];
            t_extmodel model;        

            m = 0;
            diamod = malloc(sizeof(t_extmodel) * NSEQ * NMODELS);
            memset(t, 0, 512);
            memset(errtext, 0, 512);
            memset(diaasm, 0, ARGLEN);
            if (cmd_sequence == NULL)
                return NULL;
            n = strlen(cmd_sequence);
            if (n == 0)
                return NULL;
            memcpy(diaasm, cmd_sequence, n);
            diaasm[sizeof(diaasm) - 1] = '\0';
            memset(diamod, 0, sizeof(t_extmodel) * NSEQ * NMODELS);
            pa = diaasm;
            errtext[0] = '\0';
            for (nseq = 0; *pa != '\0'; nseq++)
            {
                Addtolist(0, 1, "treat : nseq = %d, where = %s", nseq, pa);
                errpos = 0;
                offset = pa - diaasm;
                if (nseq >= NSEQ)
                {
                    sprintf(errtext, "Sequence is limited to %i commands", NSEQ);
                    break;
                }
                n = 0;
                while (n < TEXTLEN && *pa != '\r' && *pa != '\n' && *pa != '\0')  cmd[n++] = *pa++;  if (n >= TEXTLEN)
                {
                    errpos = TEXTLEN - 1;
                    strcpy(errtext, "Command is too long");
                    break;
                }
                while (*pa == '\r' || *pa == '\n')
                    pa++;
                cmd[n] = '\0';
                for (i = 0; cmd[i] != '\0' && i < (TEXTLEN - 4); i++) { if (cmd[i] != ' ' && cmd[i] != '\t')  break; }  if (memicmp(cmd + i, "ANY", 3) == 0 && (cmd[i + 3] == ' ' || cmd[i + 3] == '\t' || cmd[i + 3] == '\0' || cmd[i + 3] == ';')) {
                    i += 3;  while (cmd[i] == ' ' || cmd[i] == '\t')  i++;  validcount = 0;  for (n = 0; ; i++) {
                        if (cmd[i] >= '0' && cmd[i] <= '9')  n = n * 16 + cmd[i] - '0';  else if (cmd[i] >= 'A' && cmd[i] <= 'F')  n = n * 16 + cmd[i] - 'A' + 10;  else if (cmd[i] >= 'a' && cmd[i] <= 'f')
                            n = n * 16 + cmd[i] - 'a' + 10;
                        else
                            break;
                        validcount++;
                    }
                    while (cmd[i] == ' ' || cmd[i] == '\t')
                        i++;
                    if (cmd[i] != '\0' && cmd[i] != ';')
                    {
                        errpos = i;
                        strcpy(errtext, "Syntax error");
                        break;
                    }
                    if (validcount == 0)
                        n = 1;
                    if (n  8)
                    {
                        errpos = i;
                        strcpy(errtext, "ANY count is limited to 1..8");
                        break;
                    }
                    if (nseq == 0)
                    {
                        strcpy(errtext, "'ANY' can't be the first command in sequence");
                        break;
                    }
                    diamod[nseq * NMODELS].isany = n;
                    diamod[nseq * NMODELS].cmdoffset = offset;
                    continue;
                }
                i = 0;
                for (j = 0; i < NMODELS; j++)
                {
                    Addtolist(0, 1, "building : i = %d, j = %d, errtext = %s", i, j, errtext);
                    good = 0;
                    for (k = 0; k < 4 && i < NMODELS; k++) {
                        if (i >= NMODELS)
                            break;
                        memset(&model, 0, sizeof(model));
                        n = Assemble(cmd, 0, (t_asmmodel *)&model, j | 0x80000000, k, (i == 0 ? errtext : t));
                        if (n > 0)
                        {
                            good = 1;
                            for (m = 0, pmod = diamod + nseq * NMODELS; m < i; pmod++, m++) {
                                if (n == pmod->length && memcmp(model.code, pmod->code, n) == 0 && memcmp(model.mask, pmod->mask, n) == 0 && memcmp(model.ramask, pmod->ramask, n) == 0 && memcmp(model.rbmask, pmod->rbmask, n) == 0)
                                    break;
                            }
                            if (m >= i)
                            {
                                diamod[nseq * NMODELS + i] = model;
                                diamod[nseq * NMODELS + i].cmdoffset = offset;
                                i++;
                            }
                        }
                        else if (i == 0)
                            errpos = -n;
                    }
                    if (good == 0)
                        break;
                }
                Addtolist(0, 1, "end of building : errtext = %s", errtext);
                if (errtext[0] != '\0')
                    break;
                Addtolist(0, 1, "end of building end end");
            }
            if (nseq == 0 && errtext[0] == '\0')
                strcpy(errtext, "Empty sequence");
            if (diamod[(nseq - 1) * NMODELS].isany != 0)
            {
                nseq--;
                strcpy(errtext, "'ANY' can't be the last command in sequence");
            }
            if (errtext[0] != '\0')
            {
                MessageBox(hwmain, errtext, "SkyCks Plugin", MB_OK | MB_ICONERROR);
            }
            errpos = errpos + nseq + m;
            if (errpos)
                return (diamod);
            return (diamod);
        }        

        BOOL WINAPI DllEntryPoint(HINSTANCE hi, DWORD reason, LPVOID reserved)
        {
            if (reason == DLL_PROCESS_ATTACH)
                hinst = hi;
            return 1;
        }        

        extc int _export cdecl ODBG_Plugindata(char shortname[32])
        {
            strcpy(shortname, "Skycks");
            return PLUGIN_VERSION;
        }        

        extc int _export cdecl ODBG_Plugininit(int ollydbgversion, HWND hw, ulong *features)
        {
            if (ollydbgversion < PLUGIN_VERSION)
                return -1;        

            hwmain = hw;        

            Addtolist(0, 0, "Skype CheckSum Agents Patcher Plugin");
            Addtolist(0, -1, " Copyright (C) 2006 Ouanilo MEDEGAN");
            return 0;
        }        

        extc void _export cdecl ODBG_Pluginmainloop(DEBUG_EVENT *debugevent)
        {
        }        

        extc int _export cdecl ODBG_Pluginmenu(int origin, char data[4096], void *item)
        {
            switch (origin)
            {
            case PM_MAIN:
                strcpy(data, "0 &Spot Them All|1 &Update Comments|2 &Dump Comments|3 &Clean Comments|4 &About");
                return 1;
            default:
                break;
            }
            return 0;
        }        

        extc void _export cdecl ODBG_Pluginaction(int origin, int action, void *item)
        {
            t_dump *cpu;
            ulong base;
            char name[TEXTLEN];
            char errtext[TEXTLEN];
            char nop = 0x90;
            int res;
            char *buffer;
            char saddr[9];
            char names[20];
            FILE *file;
            int i, j, k;
            t_asmmodel model;        

            if (origin == PM_MAIN)
            {
                cpu = (t_dump *)Plugingetvalue(VAL_CPUDASM);
                switch (action)
                {
                case 0:
                    i = 0;
                    Dumpbackup(cpu, BKUP_SAVEDATA);
                    while (ptab[i].wtp)
                    {
                        errtext[0] = 0;
                        Assemble(ptab[i].code, ptab[i].wtp, &model, 0, 0, errtext);
                        if (errtext[0])
                            Addtolist(0, 1, "error : %s with %s", errtext, ptab[i].code);
                        else
                            Addtolist(0, -1, "success at %p with %s", ptab[i].wtp, ptab[i].code);
                        Writememory(model.code, ptab[i].wtp, model.length, MM_RESTORE);
                        j = k = 0;
                        j = ptab[i].end - (ptab[i].wtp + model.length);
                        while (j)
                        {
                            Writememory(&nop, ptab[i].wtp + model.length + k, 1, MM_RESTORE);
                            j--;
                            k++;
                        }
                        //Setbreakpoint(ptab[i].wtp, TY_ACTIVE, 0);
                        i++;
                    }
                    Dumpbackup(cpu, BKUP_DELETE);
                    Dumpbackup(cpu, BKUP_LOADCOPY);        

                    break;
                case 1:
                    i = 0;
                    while (checks[i])
                    {
                        memset(names, 0, 20);
                        sprintf(names, "Chk #%d", i);
                        Quickinsertname(checks[i], NM_COMMENT, names);
                        i++;
                    }
                    Mergequicknames();
                    break;
                case 2:
                    buffer = malloc(4096 * 4);
                    memset(buffer, 0, 4096 * 4);
                    strcat(buffer, "int checks[] = {");
                    //Findallsequences(cpu, build_ext_model("XOR R32, R32\r\nANY 8\r\nJMP OFFSET\r\nANY 8\r\nSUB R32, CONST\r\nANY 8\r\nDEC R32"), 0, "Skype CheckSum Layers");
                    memset(name, 0, TEXTLEN);
                    res = 0;
                    base = (ulong)Plugingetvalue(VAL_MAINBASE);
                    res = Findname(base, NM_COMMENT, name);
                    Addtolist(0, -1, "base = %p, res = %d, name = %s", base, res, name);
                    memset(name, 0, TEXTLEN);
                    memset(saddr, 0, 9);
                    while ((res = Findnextname(name)) != 0)
                    {
                        if (strstr(name, "Chk #"))
                        {
                            Addtolist(0, 0, "found : @", name, res);
                            sprintf(saddr, "%#8x", res);
                            strcat(buffer, saddr);
                            strcat(buffer, ",\r\n ");
                        }
                        memset(name, 0, TEXTLEN);
                        memset(saddr, 0, 9);
                    }
                    sprintf(saddr, "%#8x", 0);
                    strcat(buffer, saddr);
                    strcat(buffer, ",\r\n ");
                    strcat(buffer, "}");
                    file = fopen("tab.txt", "wb");
                    fwrite(buffer, strlen(buffer), 1, file);
                    fclose(file);
                    free(buffer);
                    MessageBox(0, "Job Done", "Dump", MB_OK);
                    break;
                case 3:
                    /*memset(name, 0, TEXTLEN);
                    res = 0;
                    base = (ulong)Plugingetvalue(VAL_MAINBASE);
                    res = Findname(base, NM_COMMENT, name);
                    memset(name, 0, TEXTLEN);
                    while ((res = Findnextname(name)) != 0)
                    {
                    if (strstr(name, "Chk #"))
                    {
                    Addtolist(0, 0, "found : @", name, res);
                    Insertname(res, NM_COMMENT, 0);
                    }
                    memset(name, 0, TEXTLEN);
                    memset(saddr, 0, 9);
                    }*/
                    base = (ulong)Plugingetvalue(VAL_MAINBASE);
                    Deletenamerange(base, 0xbeefff, NM_COMMENT);
                    break;
                case 4:
                    MessageBox(hwmain,
                        "Skype CheckSum Agents Patcher Plugin\n"
                        "Copyright (C) 2006 Ouanilo MEDEGAN",
                        "SkyCks Plugin", MB_OK | MB_ICONINFORMATION);
                    break;
                default:
                    break;
                }
            }
        }        

        extc void _export cdecl ODBG_Pluginreset(void)
        {
        }        

        extc int _export cdecl ODBG_Pluginclose(void)
        {
            return 0;
        }        

        extc void _export cdecl ODBG_Plugindestroy(void)
        {
        }

``````
*Checksum Agent After Automatic Patch*    

![](http://www.oklabs.net/wp-content/uploads/2012/06/after_patch.jpg)

```

*   SC程序还定期执行检查是否在调试状态。调试运行比正常运行通常慢上许多。当探测到的时候，程序会给调试器设置陷阱，如改变执行时的关键值，或者直接让它执行到一个错误的路径从而导致进程直接跑飞。这个保护使用OllyDbg的Phant0m插件hook住GetTickCount就过了。
    
*   程序有几个例程执行完整性检查，打上补丁让我们继续。
    
    _Integrity Check Exemple #1_
    
    ![](http://drops.javaweb.org/uploads/images/69937d106ccb8b7aa354866eaffdcbbff1cc9178.jpg)
    
    _Integrity Check Exemple #2_
    
    ![](http://drops.javaweb.org/uploads/images/ce7caa870bba71ab60eaadcf5cd627d38ed03424.jpg)
    
    _Integrity Check Exemple #3_
    
    ![](http://drops.javaweb.org/uploads/images/58c24defff75ef0a8602426f35bec135be5f8a82.jpg)
    
*   SC程序的敏感代码处有大量的花指令混淆正常的阅读。但是这一保护只是减慢了分析进程，并迫使我更加仔细阅读汇编代码。但在代码的某些部分使用一些特殊的技巧，使得不能跟随执行。在这些运行点上，等待是唯一的解决方案。
    
    例：RC4密钥的扩展使用基于“异常重定向（exceptions redirections）”的种子例程生成。代码异常有意调用并设置处理例程，后续的代码在其它地址上。所有运行在Ring 3级别的调试都会丢失这些处理例程。为了解决这个问题，我已经设计了一个工具，让它运行在虚拟机上，并需要一个运行着的Skype客户端附加上，在注入代码中接收种子的扩展需求和应答密钥生成器。
    
    Skype Key Server
    
    ```
    /* Skype Key Server */
    /* Author : Ouanilo MEDEGAN */        
    
    #define RC4_KLEN 88        
    
    HANDLE hProcess = 0;
    LPVOID RemoteAddr, KeyAddr;
    int Size;        
    
    void __declspec(naked) InjectedCode()
    {
        __asm
        {
            jmp BeginOCode
            //KeyAddr:
                INT 3
                INT 3
                INT 3
                INT 3
                KeyAddrGet:
            _emit 0xE8
                _emit 0x00
                _emit 0x00
                _emit 0x00
                _emit 0x00
                pop eax
                sub eax, 0x09
                mov eax, dword ptr[eax]
                ret
                //Seed:
                INT 3
                INT 3
                INT 3
                INT 3
                SeedGet:
            _emit 0xE8
                _emit 0x00
                _emit 0x00
                _emit 0x00
                _emit 0x00
                pop eax
                sub eax, 0x09
                mov eax, dword ptr[eax]
                ret
                BeginOCode :
            call KeyAddrGet
                mov ecx, eax
                call SeedGet
                mov edx, eax
                mov eax, 0x0075D470
                call eax
                //mov eax, 0x7C80C058 //ON COMMON MACHINE : ExitThread Address
                //mov eax, 0x77E54A8F //ON "NEUF" MACHINE
                mov eax, 0x401304
                call eax        
    
                DEC ECX //I
                DEC ESI //N
                DEC EDX //J
                INC EBP //E
                INC EBX //C
                PUSH ESP //T
                DEC ECX //I
                DEC EDI //O
                DEC ESI //N
                POP EDI //_
                INC EBP //E
                DEC ESI //N
                INC ESP //D
        }
    }        
    
    int SizeOfCode()
    {
        int Size;
        char *Proc;
        char Buffer[14] = { 0 };        
    
        Size = 0;
        Proc = (char *)InjectedCode;
        do
        {
            memcpy(Buffer, Proc, 13);
            Size++;
            Proc++;
        } while (strcmp(Buffer, "INJECTION_END"));
        return (Size - 1);
    }        
    
    DWORD GetSkypeProcessHandle()
    {
        HANDLE hProcessSnap;
        PROCESSENTRY32 PE32;
        DWORD SkypeProcess;        
    
        SkypeProcess = -1;
        hProcessSnap = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
        if (hProcessSnap == INVALID_HANDLE_VALUE)
        {
            printf("Error : CreateToolhelp32Snapshot (of processes) failed..\n");
            return (-1);
        }
        PE32.dwSize = sizeof(PROCESSENTRY32);
        if (!Process32First(hProcessSnap, &PE32))
        {
            printf("Error : Process32First failed..\n");
            CloseHandle(hProcessSnap);
            return (-1);
        }
        do
        {
            if (strcmp("Skype.exe", PE32.szExeFile) == 0)
            {
                SkypeProcess = PE32.th32ProcessID;
                break;
            }
        } while (Process32Next(hProcessSnap, &PE32));        
    
        CloseHandle(hProcessSnap);
        return (SkypeProcess);
    }        
    
    int Seed2Key(unsigned char *Key, unsigned int seed)
    {
        /* FIXME */
        /* For the moment based on Skype.exe process in memory. Pick the proc Appart !*/
        DWORD NbWritten, ThID;
        HANDLE hThread;
        unsigned char *CodeBuffer;        
    
        if (!WriteProcessMemory(hProcess, KeyAddr, (LPCVOID)Key, RC4_KLEN, (SIZE_T *)&NbWritten))
        {
            printf("Skype Process WriteProcessMemory (Key) failed.. Aborting..\n");
            return (0);
        }        
    
        CodeBuffer = (unsigned char *)malloc(Size);
        memcpy(CodeBuffer, (void *)InjectedCode, Size);
        memcpy(CodeBuffer + 2, (void *)&KeyAddr, 4);
        memcpy(CodeBuffer + 18, (void *)&seed, 4);        
    
        if (!WriteProcessMemory(hProcess, RemoteAddr, (LPCVOID)CodeBuffer, Size, (SIZE_T *)&NbWritten))
        {
            printf("Skype Process WriteProcessMemory (Code) failed.. Aborting..\n");
            return (0);
        }        
    
        free(CodeBuffer);        
    
        hThread = CreateRemoteThread(hProcess, NULL, NULL, (LPTHREAD_START_ROUTINE)RemoteAddr, NULL, NULL, (LPDWORD)&ThID);
        if (!hThread)
        {
            printf("Skype Process CreateRemoteThread failed.. Aborting..\n");
            return (0);
        }        
    
        WaitForSingleObject(hThread, INFINITE);        
    
        if (!ReadProcessMemory(hProcess, KeyAddr, (LPVOID)Key, RC4_KLEN, (SIZE_T *)&NbWritten))
        {
            printf("Skype Process ReadProcessMemory (Key) failed.. Aborting..\n");
            return (0);
        }        
    
        return (1);
    }        
    
    int InitProc()
    {
        STARTUPINFO Si;
        PROCESS_INFORMATION Pi;        
    
        ZeroMemory(&Si, sizeof(Si));
        ZeroMemory(&Pi, sizeof(Pi));
        Si.cb = sizeof(Si);
        //CREATE_SUSPENDED        
    
        /*if(!CreateProcessA("SkypeKeyServer.exe", "SkypeKeyServer.exe", NULL, NULL, FALSE, NULL, NULL, NULL, (LPSTARTUPINFOA)&Si, &Pi))
        {
        printf("Error creating process..\n");
        return (0);
        }        
    
        hProcess = Pi.hProcess;*/        
    
        hProcess = OpenProcess(PROCESS_ALL_ACCESS, NULL, GetSkypeProcessHandle());        
    
        if (!hProcess)
        {
            printf("Failed Opening process..\n");
            return (0);
        }        
    
        KeyAddr = VirtualAllocEx(hProcess, NULL, RC4_KLEN, MEM_COMMIT, PAGE_EXECUTE_READWRITE);
        if (!KeyAddr)
        {
            printf("Skype Process VirtualAllocEx (Key) failed.. Aborting..\n");
            return (0);
        }        
    
        Size = SizeOfCode();
        RemoteAddr = VirtualAllocEx(hProcess, NULL, Size, MEM_COMMIT, PAGE_EXECUTE_READWRITE);
        if (!RemoteAddr)
        {
            printf("Skype Process VirtualAllocEx (Code) failed.. Aborting..\n");
            return (0);
        }        
    
        return (1);
    }        
    
    void EndProc()
    {
        VirtualFreeEx(hProcess, KeyAddr, 0, MEM_RELEASE);
        VirtualFreeEx(hProcess, RemoteAddr, 0, MEM_RELEASE);        
    
        CloseHandle(hProcess);
    }        
    
    int main(int argc, char* argv[])
    {
        WORD wVersionRequested;
        WSADATA wsaData;
        int err, SelRes, Res, CbSz, i;
        SOCKET Sock;
        sockaddr_in LocalBind, ClientBind;
        fd_set Sockets;
        unsigned char Key[RC4_KLEN] = { 0 };
        unsigned int Seed;        
    
        wVersionRequested = MAKEWORD(2, 2);
        err = WSAStartup(wVersionRequested, &wsaData);
        if (err != 0)
        {
            printf("Unable to start WSA Lib\n");
            return (0xBADF00D);
        }        
    
        if (!InitProc())
            return 0;        
    
        printf("SkypeKeyServer Started..\n");        
    
        Sock = socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP);
        if (Sock == INVALID_SOCKET)
        {
            printf("Could not create socket..\n");
            WSACleanup();
            exit(0);
        }        
    
        ZeroMemory((char *)&LocalBind, sizeof(LocalBind));
        LocalBind.sin_family = AF_INET;
        LocalBind.sin_addr.s_addr = htonl(INADDR_ANY);
        LocalBind.sin_port = htons(33033);
        bind(Sock, (struct sockaddr *)&LocalBind, sizeof(LocalBind));        
    
        while (1)
        {
            FD_ZERO(&Sockets);
            FD_SET(Sock, &Sockets);        
    
            CbSz = sizeof(struct sockaddr_in);        
    
            SelRes = select(FD_SETSIZE, &Sockets, NULL, NULL, NULL);
            if (SelRes)
            {
                Res = recvfrom(Sock, (char *)&Seed, 0x04, 0, (SOCKADDR *)&ClientBind, &CbSz);
                if (Res != 0x04)
                    ZeroMemory((char *)&Key[0], RC4_KLEN);
                else
                {
                    for (i = 0; i < 0x14; i++)
                        *(unsigned int *)(Key + 4 * i) = Seed;
                    Seed2Key(Key, Seed);
                }
                Res = sendto(Sock, (char *)&Key[0], RC4_KLEN, 0, (SOCKADDR *)&ClientBind, CbSz);
            }
        }        
    
        EndProc();
        closesocket(Sock);
        WSACleanup();
        return 0;
    }
    
    ```

到了这里，我就可以获得一个几乎没有保护的二进制文件。接下来就可以真正的开始研究协议了。

这里是我研究过程中的OllyDbg udd文件（包含标签，有意义的断点，调试注释等）：[skype_binary_ollydbg_udd.zip](http://www.oklabs.net/srcdl/skype_binary_ollydbg_udd.zip)（程序路径： C:\Program Files\Skype\Phone\Skype.exe）。

0x03 Skype协议分析
=====

3.1 方法
------

在分析的第一步，是保存各种诸如首次连接没有缓存数据，重连，错误的验证数据（用户名/密码），成功的连接，聊天初始化等等，所有的数据的传送都是通过Skype网络完成。这里已经有调教好的OSpy设置，带有网络功能入口点hook和收集发送到这些功能的参数（参数包括发送到/接收自网络的数据）。

在以前的分析中我注意到一个关键点。二进制文件充斥着大量动态生成的调试字符串。但是在二进制文件中它们并未显示出来（当然，它们仅用于内部调试）。我不得不在二进制文件打上几个补丁，让程序管理那些调试字符串，并用一个特定的函数来接收生成的字符串。在这一点上，我已经调教好“OSpy”让它收集生成的字符串和填补其产生的报告，以同步网络数据发送和接收那些有意义的字符串。。最后，我能得到一个更易于理解的报告，如下图所示（生成的字符串带有“SD”标记）：

![](http://drops.javaweb.org/uploads/images/c31822201cd8ea06b5e8c2d9e86a27edc2ba5cfd.jpg)

oSpy Skype Debug Hook

```
/* oSpy Skype Debug Hook */
/* Author : Ouanilo MEDEGAN */    

#include "stdafx.h"
#include "hooking.h"
#include "logging.h"    

void skype_log(char *str)
{
    if ((str) && (strstr(str, "UI-[") == NULL))
    {
        str[strlen(str) - 1] = 0;
        message_logger_log_message("Sdbg", NULL, MESSAGE_CTX_INFO, "%s", str);
    }
}    

void
hook_skype()
{
    void *func_addr;
    void *var1_addr;
    void *var2_addr;    

    int oldProtect, NbW;    

    if (cur_process_is("Skype.exe"))
    {
        func_addr = (void *)0xB9DFC8;
        var1_addr = (void *)0xB9E150;
        var2_addr = (void *)0xB9E154;
    }
    else
        return ;    

    VirtualProtect(func_addr, 4, PAGE_READWRITE, (PDWORD)&oldProtect);
    *(DWORD *)func_addr = (DWORD)skype_log; //Override built-in & empty logging function
    VirtualProtect(func_addr, 4, oldProtect, (PDWORD)&oldProtect);
    VirtualProtect(var1_addr, 8, PAGE_READWRITE, (PDWORD)&oldProtect);
    *(DWORD *)var1_addr = (DWORD)0x01; //Set logging to YES
    *(DWORD *)var2_addr = (DWORD)0x00; //Dont Log Hour ! (~ 0x01)
    VirtualProtect(var1_addr, 8, oldProtect, (PDWORD)&oldProtect);
}

```

加上这样的详细报告，我就能够确定几乎没有什么网络数据包是伪造的，而且还能帮助理解接收/应答对等网络的数据包。实际上，这些报告还能够让我快速定位部分有趣的汇编码，如通过单步调试以便理解如何数据包是如何构建。报告还帮助我了解Skype会话的不同阶段，从断开状态，初始化通讯会话，首次进入P2P网络，在网络上登录认证本节点，动态联系人列表的获取，根据要求响应P2P协议。

3.2 协议的特性
---------

在继续之前，我发现了一些Skype协议机制，它的一些技术tip我在这里再重申一遍：

*   Skype网络是基于P2P网络模型，Skype客户端同时作为客户端和服务器使用。每个SC在网络表现为一个“node”。然而，那些带宽好，CPU棒的客户端可能会被晋升为“超级node（SN）”。超级node在Skype网络上作为普通node的接口，帮助普通node路由请求和提供网络信息。也有特殊的超级node用于防止直接连接这种特殊的网络情况，每个SC二进制文件都包含适当的超级node和普通node行为的代码。
    
*   每个节点在SC的网络都由一个“NodeID”唯一标识确定，它是一个平台特定的值，如硬盘序列号生成，或开发系统的串行。
    
*   在Skype网络中也存在一些集中式的网络架构：
    
    1.  LoginServer: 该服务用于验证客户端的帐号密码，让客户端可以成为Skype网络上一员。
        
    2.  EventServer: 用于动态缓存每个登录成功的用户信息，如联系人列表，聊天记录等等...
        
    
    _Skype Entities_
    
    ![](http://drops.javaweb.org/uploads/images/74b9c0613090c707624a2dcc4fbc8c9af2a98908.jpg)
    
*   在首次链接当中，SC只使用硬编码在引导程序上的超级node IP地址等信息，并在首次链接之后收集并保存网络数据和用户信息（当前版本保存到“shared.xml”文件）。
    
*   为了进行网络通讯，SC同时使用UDP和TCP协议。传输过程的消息的每一位都经过加密。一个消息有多钟加密算法可以选择。但主要还是用“RC4(128 bit)”算法，虽然AES（Advanced Encryption Standard）和RSA也可以使用。请参考Skype网络管理员指南获得更详细的加密细节[Skype Reverse Engineering : Genesis](http://www.oklabs.net/skype-reverse-engineering-genesis/)。
    
*   Skype协议是一个二进制协议。这意味着，该协议不像HTTP或FTP协议具备较好的可读性。该协议使用数字化的功能ID，参数类型和参数ID。它有点类似于微软的RPC（Remote procedure call）协议。这些使得该协议更加难以理解。此外，让该协议更加难以理解的是，传输的数据采用的是自己的压缩算法。
    
*   SC在一个完整的会话当中有以下几个状态：
    
    1.  “Disconnected” 离线状态。
    2.  “Registering” 注册状态（向Skype网络广播它的存在）。
    3.  “Registered” 就绪状态（准备跟其它用户互动）。
    4.  “Initializing text/file/voice/video session” 初始化状态（与其它用户协商建立通讯会话）。
    5.  “Session in progress” 通讯状态（正与其它用户通讯中）。

3.3 理解协议机制
----------

经过6个月的研究，我已经完全理解了SC客户端怎么从“Disconnected”状态切换到“Registered”状态，以及它在“Registered”状态如何工作，最后如何执行文本/文件/声音/视频会话初始化。

现在让我们总体描述一下SC客户端是怎么从“Disconnected”状态到“Registered”状态：用户从输入用户名和密码之后发起连接的整个过程。

SC客户端首先必须和一个超级node建立连接，并将该超级node作为它的父节点。这个超级node在整个会话期间作为Skype网络接口。具体的实现方法是，SC客户端执行主机扫描：发送UDP探测包给已知的超级node（缓存或引导程序设置的），直到它收到一个确定的探测应答。然后SC将尝试与该超级node建立一个TCP连接。如果失败，SC会继续执行主机扫描探测其它已知的超级node。当超级node发送一个探测拒绝应答，SC客户端也会继续执行主机扫描。

一旦SC客户端成功和超级node建立一个TCP连接，它便向超级node发送一个“client accept”请求。如果收到拒绝应答则继续执行主机扫描。收到确定应答则意味着主机扫描完成且有一个父节点可以作为它的Skype网络接口。

_Host Scan Sample_

![](http://drops.javaweb.org/uploads/images/45f3389b0ec53a23afc14b960b9f193631f5aff9.jpg)

_Successful Parent Node Connection_

![](http://drops.javaweb.org/uploads/images/41571b88120732e0f0aab5d8025343898c02bfdf.jpg)

下一步，SC进行用户验证。它将与一个已知的LoginServer建立TCP连接，并发送一个包含众多hash算法和加密原语（即256位AES长密钥，1536位RSA长密钥，SHA，MD5和RC4 hash）的数据包。这个强大的加密方案用于保护用户访问（用户名/密码）时不被钓鱼。只有具备“Skype Technologies S.A.”的LoginServer才能使用RSA的公钥/私钥解密发送过来的数据包。数据包的加密部分使用“Skype Technologies S.A.”发布的公钥，并硬编码在SC二进制文件中。客户端可以放心地认为只有具有“Skype Technologies S.A.”的私钥才能解密数据包。

注意，在发送数据包到LoginServer之前，SC已经生成一对用于会话期间的RSA公钥/私钥（每个长为1024位）。当该访问是有效时，也就是说如果用户输入的帐号密码是正确的。LoginServer将给SC发送一个包含验证用户名，验证有效期，公钥的数据结构。该结构签署“Skype Technologies S.A.”的私钥以确保它的安全性。通过硬编码在SC二进制文件的“Skype Technologies S.A.”公钥就可以对该结构体进行解密。这个数据结构证明用户已经通过LoginServer身份验证。我们称这个数据结构为“Signed Credentials”。

_Successful Authentication_

![](http://drops.javaweb.org/uploads/images/52d2e38bfb4eb0e603b77c475570e46cc9963f0e.jpg)

一旦通过验证，SC就向Skype网络广播：我的已经上线了哈。具体的实现方法是，SC客户端发送一个包含本node的位置信息（IP:Port，父节点的IP:Port，node ID等等...）的数据包。这里的用户私钥涉及使用RC4和RSA，数据包还包含之前已获得的用户Signed Credentials，这个方案的优点是确保发送者的安全性。数据的加密部分通过包含在验证通过的Signed Credentials的公钥解密出来。也就是说只有通过用户登录验证的用户才能发送数据。

节点位置信息的数据包包含（它也可以包含其他信息，如用户的真实姓名，地理位置，等等）身份验证，User's DirBlob。User's DirBlob的地址被填充为父节点并发送到指定的超级node，接着向Skype网络上发送广播让其它用户可以发现它的存在。到这里，SC处于Registered状态。

在这个状态，SC客户端从EventServer上获取用户联系人列表并检查每个存在的联系人。它的实现方案是：SC客户端与一个已知的EventServer建立TCP连接，然后跟LoginServer进行身份验证，接着请求下载该用户列表包含的每个联系人对应的hash列表。获得hash列表后如果发现没有匹配的hash，SC客户端会向EventServer发送“hash details request”命令请求EventServer回复Contact's DirBlob。这个回复包作为代替之前的User's DirBlob。它包含node位置信息，详细的登录信息，真是姓名，地理信息等等...

现在，无论SC客户端在Skype网络槽上是否上线都会从取得的联系人列表搜索每个联系人。完成之后便向指定的超级node（这个超级node应该在之前就收到SC客户端发送的Contact DirBlob广播）发送“contact search”命令。然后超级node会回复含有它这个节点的位置信息的Contact DirBlob。超级节点会保存广播的节点位置DirBlob72个小时，如果一个联系人在72小时内多次登录，SC客户端搜索的时候会得到更多的Contact DirBlob应答。它发送一个ping命令到接收过DirBlob的每个节点并挑选一个已知确切的位置的联系节点作为应答。

这一步之后SC便处于“Registered”状态，并且联系人列表也更新到最新状态。

3.4 Registered状态
----------------

_Incoming Ping Example_

一旦处于Registered状态，SC客户端便在监听它的网络接口。也就是说，他的父节点可以发送在线联系人会话初始化请求。SC客户端可以自接收父节点的进入ping，好友搜索发送的用户位置验证，网络配置测试和传入申请会话初始化。

![](http://drops.javaweb.org/uploads/images/40fbae9bb37fdff0bd25a52465233329432c6833.jpg)

_Incoming Session Proposition Example_

![](http://drops.javaweb.org/uploads/images/cbcf48ea820edea216622c355f9aae09d4854ef0.jpg)

3.5 会话初始化
---------

为了与另一个用户建立通讯连接，两个SC客户端之间需要进行协商作为通信的channel。channel定义一个安全的AES流作为数据交换使用，无论是否直接或通过超级节点中继通讯。

_Session Initialization Sample #1_

![](http://drops.javaweb.org/uploads/images/5c3471f7c35f79becbba20c35f377914a327ac19.jpg)

_Session Initialization Sample #2_

![](http://drops.javaweb.org/uploads/images/1b1b151d1be49f8d4d702aa5f55f989b41adeed8.jpg)

3.6 会话处理
--------

如果你阅读到这里，我就当作你已经看懂我之前都在说些什么，接下来让代码代替我讲诉后续的分析。

0x04 附录
=====

抱歉，以下都是法语的。

*   有关Skype v2.5协议的相关资料（包含用户验证和联系列表获取）请点击[Skype (v2.5) Protocol Analysis](http://www.oklabs.net/wp-content/uploads/2012/06/skype_protocol_analysis_signed.pdf)查看（法语）。
    
*   有关Skype v2.5协议的主要数据结构请点击[Skype (v2.5) Protocol Data Structures](http://www.oklabs.net/wp-content/uploads/2012/06/skype_data_signed.pdf)查看。
    
*   有关Skype分析项目的笔记请点击[Skype (v2.5) Analysis Project Misc Notes](http://www.oklabs.net/wp-content/uploads/2012/07/skype_analysis_notes_signed.pdf)查看。