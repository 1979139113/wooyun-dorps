# IE安全系列之：中流砥柱（II）—Jscript 9处理浅析

本文将简要介绍Jscript 9 （Charka）引擎处理数字、对象、函数操作的一些内容。

0x00 Jscript9 的安全机制
=====

IE9开始，微软使用了全新的JavaScript引擎—工程代号Charka版本9的JavaScript引擎。而IE8中它的版本还是5，也就是说5到9之间其实没有什么其他版本了，既然版本号跳跃如此之大，那Charka（下均称Jscript 9）到底有什么改进呢？

首先，与Jscript 5不同的是，JScript9引入了一个新的JIT，由此指令的执行变得更加线性化，不用像之前一样跳来跳去的执行各个分支，源代码经过解析器生成AST，然后经过字节码生成器产生字节码并由其JIT解释成机器码并执行。也达到了之前类似Malzilla Firefox所称Javascript执行性能已经达到原生C++编译出的代码的水平。

本文中我们将关注：

*   1、常量（数字、字符串）是如何表现在字节码中的以及代码优化和MSVC的编译器的异同之处；
*   2、代码的跳入跳出逻辑，对象以及函数的调用；

大改之后的JScript，由于引入了字节码执行方式，相对于之前的解释型执行，必然会为攻击者提供一些新的思路—ROP。

字节码的编译过程中，如果像编译普通EXE一样产生字节码，必然会遇到如下的问题：

*   1、生成代码固定、可预测；
*   2、内存中执行的这些代码基本受到攻击者控制；

在微软的层层保护之下，如果字节码不设防，那么从字节码中进行ROP将会是一个十分轻松愉悦的过程。由攻击者控制的JS代码（和生成的字节码）中产生Gadget将会十分简单，这个可能给UAF类型的漏洞的利用常严重的影响，会大幅降低攻击者攻击的难度。

因此，在JScript9的JIT引擎中，引入了大量的安全概念[1](http://drops.wooyun.org/wp-content/uploads/2015/08/117.png)：

（1）代码是默认开启DEP的；

（2）代码对齐随机化，字节码存放在各个新申请的内存中，“顶格”放可能会被预测地址，因此在对齐方面预留了这样的一手；

（3）随机NOP插入，也是同样防止攻击者预测指令地址的，在调试JIT生成的字节码中，你也许会发现有很多莫名其妙的指令，例如mov ecx,ecx， lea ecx, [ecx]，不用理会，这些都是它插入的NOP；

（4）常量加密存储，你会发现DWORD值在字节码中并不是按照原样存放的，而是xor了一个随机的值存放，并在使用它的时候再次xor解开；

（5）以及JIT内存分配的上限，以及页的随机化。

让我们看一下最为直观的（4），在阅读NDSS15的一篇论文时，我关注到了作者所述“Charka中小于2字节的常量都是XOR一个值并存放起来，并在使用时解开”。但是在之后的调试中我发现这个描述是不正确的。看起来也是作者的笔误。

经过调试，可知整数的存取整理如下：

32位IE中，

（1）2 byte以上32位整数常量全部以(n * 2 + 1) xor (some value)过的形式存在内存中；xor key随机，需要一段代码解开才能继续使用；

（2）n > 0x7fffffff的值无法* 2 + 1放在EXX寄存器中了，这个时候会转入SSE寄存器中（xmmX）来处理；

（3）2 byte以下常量将以n * 2 + 1的方式保存，并在需要使用时将其右移一位再使用。

以下是具体的例子：

#### 1、“2Byte” 以下的正整数和负整数。

2 byte以下常量将以n * 2 + 1的方式保存，并在需要使用时将其右移一位再使用。

正常来说，32位整数，包括负数在Chakra中均采用有符号数来处理，那-1即0xffffffff，让我们通过实例来确定是否存在这个情况。

考虑如下JS代码：`var a = 1; var b = -1; a--; b++;`

挂上调试器，栈中看到这种解释不出来名字的，跟在`Js::JavascriptFunction::CallFunction`的，就是编译出来的字节码：

![enter image description here](http://drops.javaweb.org/uploads/images/463d807cbaa697ed30292bbd69c1825fcdd3fa10.jpg)

该地址所在的内存页的Allocation Base基本就是函数开头，观察该函数。

首先可以看到大量的NOP插入，如箭头处，感觉没有什么大作用的语句会让想从这里拿到ROP的片段的攻击者大为头疼。

![enter image description here](http://drops.javaweb.org/uploads/images/018590d54e0443e4c820f17920fb866b5cf16454.jpg)

继续往下，可以看到在Chakra（11.0.9600.17689）中，这两个数字被解释为：

![enter image description here](http://drops.javaweb.org/uploads/images/9d10f4699c258efc4eefb9189b909bb55e72fec7.jpg)

1被存储为了3 ，也即 1 * 2 + 1 （左移一位加一）。 -1 被存储为了0xffffffff， -1 * 2 + 1 = -1 （同上）。

![enter image description here](http://drops.javaweb.org/uploads/images/747acde429084fe886bb4aa0a326dead152fd7b3.jpg)

而由于编译执行，所以编译器自然地做了优化，将a++ 的值2按 * 2 + 1编码后5 和 b— 的值 -2 ，编码后-2 * 2 + 1 = -3 （0xfffffffd）提前计算好写入了字节码中，如上图，这个和流行的编译器的优化逻辑如出一辙，只不过并没有像是编译器那样那么彻底，有点像Release开启了O1 O2优化之后的感觉。但是明显效率高于Debug版的。

让我们看一下同样代码在VS2008 Release开启O2和Debug 默认以及Chakra的结果：

![enter image description here](http://drops.javaweb.org/uploads/images/4d649bbdb3a2bf349ea30de8316f7ba6bb2bc3be.jpg)

代码在Debug Vs2008生成如下操作：

![enter image description here](http://drops.javaweb.org/uploads/images/038190bccab1140e1be24a1927b849ccf2e37afe.jpg)

可以看到就是逐句翻译，真·机翻的感觉。

![enter image description here](http://drops.javaweb.org/uploads/images/27057acc4b9ad4caa170a150da06fc2a5d6f2361.jpg)

![enter image description here](http://drops.javaweb.org/uploads/images/7eb1b71e635622a1f1d9b23cd5d90792afa0ad0d.jpg)

Release 开启O2之后的效果，这里已经把加加减减的值算好了，这个和Chakra的动作一样。

![enter image description here](http://drops.javaweb.org/uploads/images/7ec59e4768755cdd6da04f54c35812a711dd993c.jpg)

![enter image description here](http://drops.javaweb.org/uploads/images/eac6215231f56b6edeca2615c15bdeae38a3a2b1.jpg)

代码比较简单，所以开启O1也是一个效果，所以用脚趾头也可以得出结论，Chakra的字节码编译过程中确实有优化。

不过不同于VS，如果你直接写个变量加加减减最后哪儿也不用他，开启OX优化的时候，这个变量根本就不会编译进EXE，而Chakra则稍有不同，即使没用，还是编译进来了，只不过做了最大可能的优化处理。

如果全优化了，F12你怎么调试呢。

#### 2、“2Byte”～“4Byte”之间的整数存取

这个区间内的数字比较特殊，采用了异或的方式存储。异或的随机密钥在生成字节码时由编译器确定。

考虑如下JS代码：

var a = 0x32132132; var b = -0x12312312;

a++; b--;

调试可知，这里的数字被“加密”了：

![enter image description here](http://drops.javaweb.org/uploads/images/5a9494ed87d86eafbf4a5d2662d86d982a88ed6c.jpg)

以

```
062a003f bad8359075      mov     edx,offset USP10!pScriptProperties+0xd0 (759035d8)
062a0044 81f2bd77b611    xor     edx,11B677BDh

```

为例`（a = 0x32132132;）`

```
0:008> g
Breakpoint 0 hit
eax=9ef4b531 ebx=03f2bad0 ecx=03419720 edx=db9db9db esi=03f2b8bc edi=062a0000
eip=062a003f esp=03f2b8a0 ebp=03f2b8a8 iopl=0         nv up ei pl zr na pe nc
cs=0023  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00000246
062a003f bad8359075      mov     edx,offset USP10!pScriptProperties+0xd0 (759035d8)
0:008> p
eax=9ef4b531 ebx=03f2bad0 ecx=03419720 edx=759035d8 esi=03f2b8bc edi=062a0000
eip=062a0044 esp=03f2b8a0 ebp=03f2b8a8 iopl=0         nv up ei pl zr na pe nc
cs=0023  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00000246
062a0044 81f2bd77b611    xor     edx,11B677BDh
0:008> 
eax=9ef4b531 ebx=03f2bad0 ecx=03419720 edx=64264265 esi=03f2b8bc edi=062a0000
eip=062a004a esp=03f2b8a0 ebp=03f2b8a8 iopl=0         nv up ei pl nz na pe nc
cs=0023  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00000206
062a004a 899108060000    mov     dword ptr [ecx+608h],edx ds:002b:03419d28=67422664

```

将edx右移一位，得到0x32132132即a的值。

负数的处理也是一样，以第二句为例：

```
062a0056 ba170c6687      mov     edx,87660C17h
062a005b 81f2cab5fb5c    xor     edx,5CFBB5CAh

0:008> 
eax=9ef4b531 ebx=03f2bad0 ecx=03419720 edx=64264265 esi=03f2b8bc edi=062a0000
eip=062a0056 esp=03f2b8a0 ebp=03f2b8a8 iopl=0         nv up ei pl nz na pe nc
cs=0023  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00000206
062a0056 ba170c6687      mov     edx,87660C17h
0:008> 
eax=9ef4b531 ebx=03f2bad0 ecx=03419720 edx=87660c17 esi=03f2b8bc edi=062a0000
eip=062a005b esp=03f2b8a0 ebp=03f2b8a8 iopl=0         nv up ei pl nz na pe nc
cs=0023  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00000206
062a005b 81f2cab5fb5c    xor     edx,5CFBB5CAh
0:008> 
eax=9ef4b531 ebx=03f2bad0 ecx=03419720 edx=db9db9dd esi=03f2b8bc edi=062a0000
eip=062a0061 esp=03f2b8a0 ebp=03f2b8a8 iopl=0         nv up ei ng nz na pe nc
cs=0023  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00000286
062a0061 89910c060000    mov     dword ptr [ecx+60Ch],edx ds:002b:03419d2c=dbb99ddb

```

得到edx后，将其右移一位得到`0xEDCEDCEE`(记得符号位不要也一起右移了)。正是`var b = -0x12312312`;。

下方a++运算结果和2byte的例子一样，我们就简单看一个

```
062a006d ba30059380      mov     edx,80930530h
062a0072 81f25747b5e4    xor     edx,0E4B54757h

```

异或后：64264267 右移一位：32132133 （32132132 + 1）

#### 3、4byte以上的结果

作为32位有符号数，大于4byte的就不好弄了，让我们看看

```
var a=0x90909090;
var b=0x99999999;
a++;b++;

```

的结果：

针对它的加法，JS已经选择用SSE2的函数了：

```
07030080 68b8b31803      push    318B3B8h
07030085 53              push    ebx
07030086 e88539e46b      call    jscript9!Js::SSE2::JavascriptMath::Increment_Full (72e73a10)
0703008b 8bf8            mov     edi,eax
0703008d e9b8ffffff      jmp     0703004a

```

它们的值被提前算好存储起来了：

```
063b0067 47              inc     edi
063b0068 8b1504612c03    mov     edx,dword ptr ds:[32C6104h]
063b006e 81fa804f2c03    cmp     edx,32C4F80h
063b0074 0f8562000000    jne     063b00dc

063b007a 8b1508612c03    mov     edx,dword ptr ds:[32C6108h]
063b0080 8bff            mov     edi,edi
063b0082 899a08060000    mov     dword ptr [edx+608h],ebx
063b0088 8b1508612c03    mov     edx,dword ptr ds:[32C6108h]
063b008e 89b20c060000    mov     dword ptr [edx+60Ch],esi
063b0094 8b1508612c03    mov     edx,dword ptr ds:[32C6108h]
063b009a 898208060000    mov     dword ptr [edx+608h],eax
063b00a0 8b1508612c03    mov     edx,dword ptr ds:[32C6108h]
063b00a6 898a0c060000    mov     dword ptr [edx+60Ch],ecx

```

ebx、esi等其实是存储着mmword 的指针，但这看起来并不直观，所以我们需要有更直观的方式：

```
var a=0x90909090;
var b=0x99999999;
b = a+b;

068f0026 8d6424f4        lea     esp,[esp-0Ch]
068f002a 57              push    edi
068f002b 56              push    esi
068f002c 53              push    ebx
068f002d bbf0805503      mov     ebx,35580F0h
068f0032 be00815503      mov     esi,3558100h
068f0037 33ff            xor     edi,edi
068f0039 f20f10056c490003 movsd   xmm0,mmword ptr ds:[300496Ch]
068f0041 f20f100db4f96e00 movsd   xmm1,mmword ptr ds:[6EF9B4h]
068f0049 f20f58c1        addsd   xmm0,xmm1
068f004d 8b0538b8fb02    mov     eax,dword ptr ds:[2FBB838h]
068f0053 8d4810          lea     ecx,[eax+10h]
068f0056 3b0d34b8fb02    cmp     ecx,dword ptr ds:[2FBB834h]
068f005c 0f875b000000    ja      068f00bd

```

这回就看的很清楚了

![enter image description here](http://drops.javaweb.org/uploads/images/10b093502ff90c0f3279a9d59f95b91663bd22cd.jpg)

![enter image description here](http://drops.javaweb.org/uploads/images/0e3d930217d3a0efa0d0ac3a5dac9a49ef614c12.jpg)

这正是我们的第一个数，0x90909090；

![enter image description here](http://drops.javaweb.org/uploads/images/aabdcde6ce466ac394eb27fdb06555f656dc6dbd.jpg)

第二个数也不例外，

![enter image description here](http://drops.javaweb.org/uploads/images/cef4631bccdcd39248c2badfdbca6c8f45eee123.jpg)

可见，使用xmm的时候，Chakra并没有给它们做任何的混淆，事实上也没有什么太大的必要混淆这类数字了，因此做法可以接受。

VIII.2 对象的操作与结构

作为一个面向对象的语言，JScript中宽松的定义使得对象操作简单但是又“复杂” ，考虑如下代码。

```
var a = new Object;  
a.X = 3; a.Y = 4;

```

此时a是一个含有X、Y两个Number成员的Object，它的编译方式是怎样呢？

请一定要记得上一节的内容，3和4在内存里面是7和9。

![enter image description here](http://drops.javaweb.org/uploads/images/775a8e87b270e4f278cb9a67052ed07ad0ea5340.jpg)

只有这样，我们才能愉快地找到后两句的位置，阅读代码可知这个对象的内存分布是：

```
+0: jscript9!Js::ArrayBufferParent::`vftable' 

```

![enter image description here](http://drops.javaweb.org/uploads/images/52a0f3a6539e813b3a2976d436c737d407510d08.jpg)

左边是Object，右边是成员的两个Number，在Windbg中可以跟踪得到：

```
0:007> dd [edi+8]
0b69a030  00000007 00000009 030eb040 030eb040
0b69a040  00000000 00000000 00000000 00000000

```

这样当然是不够的，让我们再看看将这个对象（或者更形象的说“Property Bag”）复制后再扩展一个成员（Slot），代码如下

```
function xObj(x,y)
{
this.X = x; 
this.Y = y;
} 
var a=new xObj(3,4); 
var b=new xObj(3,4); 
b.Z = 5;

```

首先，找到五个赋值语句的位置：

![enter image description here](http://drops.javaweb.org/uploads/images/21a9005f13fb347dfa212366258f8ed696a1859e.jpg)

第一句的3，4

![enter image description here](http://drops.javaweb.org/uploads/images/1a4e2cd6bc5a94eebb754e2383740c3324ea801d.jpg)

第二句的3，4

到此为止，这两个对象都是一样的，

![enter image description here](http://drops.javaweb.org/uploads/images/c8e15b67d1dace17043653d55cee8cb0180e5669.jpg)

直到这里，可以看出来它给b的第三个元素（应为Z）赋值5，这里a、b的情况我们在控制台输入一下。

**a：**

![enter image description here](http://drops.javaweb.org/uploads/images/c977a436b5b6fa9b930c867c167092a3843e8c0e.jpg)

**b：**

![enter image description here](http://drops.javaweb.org/uploads/images/8210377a3631ce5d443937fe9e0e3df451eabcfa.jpg)

可以看出来a、b除了成员变量之外别无二致，这正是Chakra中为了对象的快速存取而制作的多态特性：

![enter image description here](http://drops.javaweb.org/uploads/images/6753d1e0abf324f03f7922188a26b17135d7836d.jpg)

VIII.3 函数的调用

最后，让我们回到这个问题上，想在一章之内说完Chakra绝对无异于痴人说梦，所以这里挑一个常见的地方来描述：在Chakra中的函数调用是如何的？

首先是基本的函数，这个和JScript5差不多，一些具体功能的操作符（或者在JS中称为Token），例如new Object这样无参数的new操作符，对应的函数是`Js::JavascriptOperators::NewJavascriptObjectNoArg`。

而有些简单的内容被编译器直接计算值后显示，例如Math.sqrt(5)在这里直接存为了`2.23606xxxxx`，而不会再现场调用函数来运算：

![enter image description here](http://drops.javaweb.org/uploads/images/69606b69a8180d6f7852fb0093cee61867212da5.jpg)

其他的一些无法优化的函数调用还是会照常调用，还是类似JScript5一样看名字即可知道是做啥的，例如取随机数的函数即为Js::SSE2::JavascriptMath::Random：

![enter image description here](http://drops.javaweb.org/uploads/images/790e96b33fe37118df5b954755a4cd717abf417b.jpg)

通过符号很容易看到各个函数的名称和参数类型等：

![enter image description here](http://drops.javaweb.org/uploads/images/259eda9d94bbee025d6f2a8b67eb4c9971ccd140.jpg)

而用户自定义的函数、循环体等通常在字节码中不会作为内联函数存在，例如下列伪代码，它们的调用有点像是：

![enter image description here](http://drops.javaweb.org/uploads/images/1ad4a5f9c77e611624d5ddbb7b41ee1f02885e1b.jpg)

在生成字节码的时候它的范围通常是：

![enter image description here](http://drops.javaweb.org/uploads/images/f21e344a177a542cd22a9cadd4f272e8ace732d3.jpg)

颜色代表立场，呸，颜色代表不同的函数块，上述函数在调用时最可能的栈像是：

![enter image description here](http://drops.javaweb.org/uploads/images/d2ec66fe2707f30fe4f16a4311b334bfe2d2b00c.jpg)

Chakra的内容介绍到此结束，限于个人水平以及篇幅内容，先介绍的就是这么多，其实基本调试方法和调试之前的代码没啥两样，这里的调试很容易就能够举一反三。

自Chakra开始，IE的脚本执行速度比先前提升了十多倍（微软自己的SunSpider跑分），原图我找不着了，但是在微软的某个文章里面还是看到了小图……如下，IE9面世不久后的测试得分和当时的Chrome21的脚本跑分不相上下。可惜悲剧的它出来的时候大家在用IE6，IE11出来的时候大家还是在用IE6，就是卡的不行啊怎么办，赶紧升级吧:)。

![enter image description here](http://drops.javaweb.org/uploads/images/8acdad9a7cf0cea395b0b757f263e48084343a87.jpg)

[1](http://drops.wooyun.org/wp-content/uploads/2015/08/117.png)[http://files.accuvant.com/web/files/images/webbrowserresearch_v1_0.pdf](http://files.accuvant.com/web/files/images/webbrowserresearch_v1_0.pdf)

[2](http://drops.wooyun.org/wp-content/uploads/2015/08/218.png)[http://users.ics.forth.gr/~elathan/papers/ndss15.pdf](http://users.ics.forth.gr/~elathan/papers/ndss15.pdf)

[3](http://drops.wooyun.org/wp-content/uploads/2015/08/315.png)[https://blog.indutny.com/5.allocating-numbers](https://blog.indutny.com/5.allocating-numbers)