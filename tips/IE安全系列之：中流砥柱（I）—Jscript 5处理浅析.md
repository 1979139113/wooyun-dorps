# IE安全系列之：中流砥柱（I）—Jscript 5处理浅析

本文将简要介绍Jscript5.8引擎处理相关代码、字符操作、函数操作的一些内容，下一篇中流砥柱（II）将以同样的方式介绍Jscript 9 （Charka）的处理方式。

VII.1 Jscript5 处理浅析
=====

IE中的JavaScript解释引擎有两种，一种是IE9（不含）之前的浏览器，采用的Jscript 5.8-，包括现在的WSH都是采用这个解释器。该解释器是典型的switch-dispatch threading interpreter，与之相关的也可以参考参考资料[1](http://drops.wooyun.org/wp-content/uploads/2015/06/image00110.png)QQ大讲堂的文章，这个类型的解释器就相当于提前预设好了一堆case，待词法分析器分析到上面之后，会采用dispatch的方式跳转到各个处理函数中去。

这里，以我的电脑上的JScript（版本：5.8.9600.16428，Win 7 Sp1 X86）为例，简单看一下JScript的相关代码结构。

JScript解释代码采用了解析代码（COleScript::ParseScriptText）时先生成抽象语法树（AST），该树的节点即为Javascript 的token（可以认为是保留字或操作符，例如=、!=、+、for等等），然后调用解释器进行解释执行的模式，JScript的解释器是CScriptRuntime::Run(VAR *)，这是一个体积巨大的函数，这给我们阅读代码带来了一些不便，至于它体积巨大的原因，首先我们需要知道，一个switch dispatch的解释器，它的代码结构是类似于下图的：

![enter image description here](http://drops.javaweb.org/uploads/images/2b916abe88206a4c135c6c089992c8a1493abc03.jpg)

因此，让我们从实际例子观察一下这个解释器。

针对以下代码：

```
var p=0x11223344;
p.toString();

```

可以看到JScript的调用栈如下（无关栈内容有省略）：

```
0:000> k
ChildEBP RetAddr  
0016e244 757efbd0 msvcrt!_ui64toa_s+0x42
0016e254 60eab798 msvcrt!_ltow+0x22
0016e274 60eab7d7 jscript!GetStringForNumber+0x32
0016e4c8 60eacc92 jscript!ConvertToString+0xb9
0016e504 60e95950 jscript!JsNumToString+0x7b
0016e56c 60e95876 jscript!NatFncObj::Call+0xce
0016e5f4 60e94546 jscript!NameTbl::InvokeInternal+0x1d6
0016e7a0 60e95396 jscript!VAR::InvokeByName+0x445
0016e7ec 60e9534e jscript!VAR::InvokeDispName+0x3e
0016e8cc 60e92bbb jscript!AutBlock::AddRef+0x7df3
0016ec64 60e92ea6 jscript!CScriptRuntime::Run+0xe59
0016ed50 60e92d19 jscript!ScrFncObj::CallWithFrameOnStack+0x170
0016ed9c 60e93317 jscript!ScrFncObj::Call+0x7b
0016ee30 60e9fb9f jscript!CSession::Execute+0x1f6
0016ee90 60e9dd02 jscript!COleScript::ExecutePendingScripts+0x293
0016eeac 00c219ee jscript!COleScript::SetScriptState+0x51

```

此处，可以猜测JScript的Run过程通过Name搜索其对应的Dispatch，下层通过一个Name Table来找到对应的函数，并调用函数，然后获取返回结果。这看起来像一个非常标准的流程。让我们从代码中阅读一下它的逻辑。

回去找找它的vPC++指针是在哪儿移动的吧。通过跟踪代码或者直接阅读即可得知“vPC”的移动过程事实上发生在：

![enter image description here](http://drops.javaweb.org/uploads/images/b0fe6623b730c1925dfbc116295494799bfae7a2.jpg)

上述loc_63382650段正是switch(*vPC++){}的代码，该switch中共有0x0 ～ 0x9f（159 DEC）个switch项目，及一个default项目。IDA也计算好了共有160 DEC cases。在静态阅读的时候，可以找找Switch Table，align 4后开始第一项为0，后面依次数一下就可以手动切换到对应的case上（下文也称dispatch）。

![enter image description here](http://drops.javaweb.org/uploads/images/e0afc2311e70ba84fc43fd6322dfcb984f7c46ee.jpg)

各个DispatchCode对应某个或某些模式，例如赋值（assignment，例如p=1）对应0x8的Dispatch：

![enter image description here](http://drops.javaweb.org/uploads/images/af95422ed8a8c1e099a876ae93cb3db29cbf5af6.jpg)

再来，为了完成Number.toString()之类的方法调用，. (accessfield)对应的dispatch id为0x2d。包括它之后的函数等，这个完整的语法解析后会将方法名传送给VAR::InvokeByName以完成调用toString()。

![enter image description here](http://drops.javaweb.org/uploads/images/87a322d2849bf8a826396f958aadb349a8e371a5.jpg)

![enter image description here](http://drops.javaweb.org/uploads/images/150d21aa19e345101c3f96a1d746a7781ff67b63.jpg)

![enter image description here](http://drops.javaweb.org/uploads/images/9748763a1e0f1acde29c112565947c90d0fac333.jpg)

IDA由Symbol进行反crack后得到的函数及类型是VAR::InvokeByName(CSession *,SYM *,ulong,VAR *,int,VAR *)。由它让我们看看它的分发过程。可能看到InvokeByName这个名字的时候已经有一些疑惑，在展开它的代码后，阅读前面一点即可知道，这个过程和自动化过程中的IDispatch::InvokeByName几乎一样。

VAR::InvokeByName的原型实际上正是 ：

```
HRESULT GetIDsOfNames(  
  REFIID  riid,                  
  OLECHAR FAR* FAR*  rgszNames,  
  unsigned int  cNames,          
  LCID   lcid,                   
  DISPID FAR*  rgDispId          
);

```

可以简单的证实一下：

```
0:000> bp  VAR::InvokeByName
0:000> g
Breakpoint 1 hit
eax=003ce9fc ebx=00000000 ecx=003cea10 edx=00000080 esi=00223960 edi=00000000
eip=52f333f1 esp=003ce9dc ebp=003cea24 iopl=0         nv up ei pl zr na pe nc
cs=0023  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00000246
jscript!VAR::InvokeByName:
52f333f1 8bff            mov     edi,edi
0:000> kvn 1
 # ChildEBP RetAddr  Args to Child              
00 003ce9d8 52f35396 00223960 003ce9fc 00000001 jscript!VAR::InvokeByName (FPO: [6,95,4])
0:000> du poi(poi(esp+8))
00225998  "toString"

```

从代码中也可以看到，这里它正是调用了IDispatch::GetIDsOfNames，获得了DISPID （v84 / v94）：

![enter image description here](http://drops.javaweb.org/uploads/images/8ecfa6129371ae838ede53d0aa543cb148a8dc0d.jpg)

并最后InvokeDispatch调用对应的函数：

![enter image description here](http://drops.javaweb.org/uploads/images/5771a9c876da7544617018675f3e7bcb042f909c.jpg)

VII.2 简单的字符串的处理
=====

看完了上述对解释器的简单介绍，让我们再着眼看一下JS中另一个重要的部分：字符串。

由于大量采用了COM，因此即使之前没接触，大致也是能猜出来的，JS中的字符串是与BSTR有关的。

BSTR是一个与自动化兼容的字符串类型，操作系统提供类似SysAllocString的函数来管理它。BSTR实际上可以看作一个COM字符串（但是如果你深究的话，实际上它本质就是PWCHAR）。

定义在wtypes.h中的BSTR如下：

```
typedef wchar_t WCHAR; 
typedef WCHAR OLECHAR; 
typedef OLECHAR *BSTR;

```

普通字符串一般使用\0结尾，但是这样的字符串在COM 组件间传递不方便。因此有了这个BSTR的规范，标准的BSTR是一个有长度前缀（前4字节，长度计算时不包含结尾的\0）和\0的OLECHAR数组。

![enter image description here](http://drops.javaweb.org/uploads/images/394e367ca3cb4c2142bd60d2716f75a7ef82564a.jpg)

让我们在JS中验证一下。为了方便定位，我们执行escape(“ABCD”)。然后断下函数jscript!JsEscape。 这一块VII.3还会提到，这里先如此继续调试。

![enter image description here](http://drops.javaweb.org/uploads/images/0f5c38a9e12b14fa64bd31c475ce80d96048e3ff.jpg)

jscript!JsEscape函数是一个简单的Helper，会做简单的判断，然后将内容传给jscript!ScrEscape。

![enter image description here](http://drops.javaweb.org/uploads/images/9f3eac63fe594e0f6c31115c9af674d57ba4bd57.jpg)

为了方便调试，在ScrEscape处断下。

```
0:008> bp jscript!ScrEscape
0:008> g
Breakpoint 0 hit
eax=00000000 ebx=00000003 ecx=005f35c8 edx=005f56b8 esi=005f5130 edi=001aef70
eip=68d61627 esp=001aea58 ebp=001aea7c iopl=0         nv up ei pl zr na pe nc
cs=0023  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00000246
jscript!ScrEscape:
68d61627 8bff            mov     edi,edi

```

ScrEscape原型有4个参数，这个函数是fastcall (MS)，因此前两个参数的传递使用edx,ecx传递。它的第一个参数就是str，因此不出意外的话，edx即为BSTR strToEscape： 

```
005f56b8  "ABCD"
0:000> du edx
005f56b8  "ABCD"
0:000> dd edx-4
005f56b4  00000008 00420041 00440043 baad0000
005f56c4  ba99838a 00000000 00000008 0000000f
005f56d4  00000014 00000120 00000001 00000001
005f56e4  00000000 00000000 00000027 00000000
005f56f4  00000001 0053004a 00720063 00700069
005f5704  00200074 0020002d 00630073 00690072
005f5714  00740070 00620020 006f006c 006b0063
005f5724  baad0000 00680077 006c0069 00280065

```

果不其然，情况正是如此。为了更直观一点，我再手动加一个字，ABCDE，此时，长度DWORD@(edx-4)应该是0000000A。

执行即可知道确实如此：

```
0:008> bp jscript!ScrEscape;g;du edx;dd edx-4;
Breakpoint 0 hit
004656b8  "ABCDE"
004656b4  0000000a 00420041 00440043 00000045
004656c4  ba9a838a 00000000 00000008 0000000f
004656d4  00000015 00000124 00000001 00000001
004656e4  00000000 00000000 00000028 00000000
004656f4  00000001 0053004a 00720063 00700069
00465704  00200074 0020002d 00630073 00690072
00465714  00740070 00620020 006f006c 006b0063
00465724  baad0000 00680077 006c0069 00280065

```

知道了字符串的存储，我们还需要知道和字符串相关的操作，例如连接。

考虑下列代码：

```
var p = "ABCD";
var q = "EFGH";
var r = p + q;

```

有这样一个疑惑，就是一般情况下，p+q的结果是否会基于p或者q的内存进行扩充？调试可知：

```
0:000> du 004d5e10
004d5e10  "ABCD"
0:000> !heap -x 0x4d5e10
Entry     User      Heap      Segment       Size  PrevSize  Unused    Flags
-----------------------------------------------------------------------------
004d5da8  004d5db0  004d0000  004d0000       2c8       908        1c  busy extra fill 


0:000> du 004d5e3c
004d5e3c  "EFGH"
0:000> !heap -x 0x4d5e3c
Entry     User      Heap      Segment       Size  PrevSize  Unused    Flags
-----------------------------------------------------------------------------
004d5da8  004d5db0  004d0000  004d0000       2c8       908        1c  busy extra fill 

0:000> du 0097a6ec
0097a6ec  "ABCDEFGH"
0:000> !heap -x 0x97a6ec
Entry     User      Heap      Segment       Size  PrevSize  Unused    Flags
-----------------------------------------------------------------------------
0097a6e0  0097a6e8  00930000  00930000        38        28        18  busy extra fill 

```

可见明显两个常量“相加”（+， case 0x7）之后的结果并没有扩充之前的内存。原因是其中有两个常量，这两个常量的值在解析时就已经分配好了，而连接后的结果是现场生成的。

使用 gflags -p /enable [image filename] 可以打开页堆，断到地址后可使用!heap -p -a ADDRESS观察该内存对应堆的分配栈。

常量的分配栈大致如下，可见实在分析语法的时候就已经分配好的，这个不难理解，要不然语法树也没法建立了：

```
 74ad9d45 msvcrt!malloc+0x0000008d
 68d3845e jscript!Parser::GenerateCode+0x000001b2
 68d3955d jscript!Parser::ParseSource+0x0000053d
 68d38e24 jscript!COleScript::Compile+0x00000145
 68d420d4 jscript!COleScript::CreateScriptBody+0x000000e5
 68d41f7c jscript!COleScript::ParseScriptTextCore+0x0000021c
 68d4065f jscript!COleScript::ParseScriptText+0x00000029
 00772986 wscript!CScriptingEngine::Compile+0x00000067

```

再做这样一个测试，测试都是非常量的情况会如何：

```
var r = p + q;
var r = r + r;

0:000> !heap -x 005c1d28
Entry     User      Heap      Segment       Size  PrevSize  Unused    Flags
-----------------------------------------------------------------------------
005c1d28  005c1d30  00560000  00560000        58        50        10  busy extra 

0:000> !heap -x 005c1e04
Entry     User      Heap      Segment       Size  PrevSize  Unused    Flags
-----------------------------------------------------------------------------
005c1dd8  005c1de0  00560000  00560000        68        58        10  busy extra 

```

结果也是重新分配了。而Jscript 5中，针对字符串的操作最初还是比较低效率的，每一次+（Plus）操作都会导致分配新内存，这包括“a”+“b”+“c”，假定“a”是1个单位长度，这个操作会先在堆上分配2个单位长度的缓冲区，将“ab”复制进去后，又申请3个单位的缓冲区，将“abc”复制进去并返回。toString()的操作也是类似，也是分配一个缓冲区再复制进去，即使是String.toString()也是如此。

JScript中还有一个AString类，是用于处理多字节的一个类，如果你在调试中遇到了它，将其da poi(poi(ADDR) + 8)即可看到AString中存储的ANSI String（特别是你在调试调用了VAR::ConvertASTRtoBSTR的函数时可能会想要通过这个方式了解它转的字符串是什么）。限于篇幅，这块之后如果有介绍到相关具体漏洞的时候再详细介绍。

VII.3 一见即知：JScript5引擎的函数与JScript脚本的函数
=====

JScript中存储了许多Helper（Wrapper），这些Helper的名字其实和之前几章说到的CElement一样有规律可循，通常一些JScript的原生函数都会形成有JsXXXX的Helper函数，这些Helper会对传入的参数做简单处理并调用具体逻辑的实现的函数。

这些函数的名称在微软的Symbol 中，所以使用IDA或者Windbg（x jscript!Js*）即可读取到这些函数。

![enter image description here](http://drops.javaweb.org/uploads/images/413f68e73246d89ffe79b6c90d605717bc09849f.jpg)

举例来看，例如一些数学函数，仅仅是直接调用对应的函数，例如JsExp（对应JScript： exp()）

![enter image description here](http://drops.javaweb.org/uploads/images/5515dd5ef458133b91a35729dd5f37cc587715ed.jpg)

图：Javascript exp()方法

```
（IDA生成的伪代码，有少量改动）
double __stdcall Exp(double a1)
{
  return _exp(a1);
}

int __stdcall DoMathFnc(double a1, int a2, int a3, void (__stdcall *pFunc)(_DWORD, _DWORD))
{
  int result; // eax@2
  __int16 v1; // [sp+18h] [bp-10h]@1
  double arg_1; // [sp+20h] [bp-8h]@2

  GetParam(0, &v1);
  if ( 5 == v1 || (result = ConvertToScalar(&v1, 5, 1), result >= 0) )
  {
    pFunc(LODWORD(arg_1), HIDWORD(arg_1));
    VAR::SetNumber(arg_1);
    result = 0;
  }
  return result;
}

int __stdcall JsExp(int a1, int a2, int a3, int a4, int a5)
{
  return DoMathFnc(a3, a4, a5, Exp);
}

```

也有直接在里面进行实现的，例如JsRandom （对应Math.random()）

![enter image description here](http://drops.javaweb.org/uploads/images/7a201d630d21f819d5222e9616629f2753227aea.jpg)

![enter image description here](http://drops.javaweb.org/uploads/images/de1594663e0369003dbec18ca30d445df0200efb.jpg)

图：Jscript random()方法

以及做一些预处理后调用处理方法的函数，例如JsJSONStringify，对应JSON.stringify().

![enter image description here](http://drops.javaweb.org/uploads/images/caa82120a8eb15b0b90f71c18f5748fa39d95392.jpg)

![enter image description here](http://drops.javaweb.org/uploads/images/a7511167984c6dc2177a78dcce885c773e5a9191.jpg)

图：JSON.stringify()

除去需要分析这些函数本身的用途之外，这些函数还有一些比较方便的作用，那就是可以作为一个类似中断的东西而存在。抛开alert()之类的阻塞型，如果想要跟踪某个字符串，那么完全可以通过：

```
var s =”string that blast want to track how it goes”;
escape(s);

```

然后，在脚本未执行时下断点：bp jscript!JsEscape，等跟踪到这个函数时，其参数就是我们想要拿到的这个字符串。这样会方便调试代码，尤其是一些不容易跟踪的东西，例如一个数字之类的。

本篇到此为止介绍了Jscript 5中的一些简单实现，一些具体的内容大家也可以参考Microsoft Javascript Blog/ Microsoft IE Blog的文章，当然他们不会说的很细，如果对这个感兴趣的话，可以使用wscript.exe或者iexplore.exe（推荐IE8-）来跟踪调试代码。下篇将介绍Jscript 9 （Charka）的一些简单实现。

(1] http://security.tencent.com/index.php/blog/msg/43

(2] 浅析JavaScript引擎的技术变迁 http://djt.qq.com/article/view/489

(3] http://www.w3school.com.cn/jsref/jsref_exp.asp

(4] https://msdn.microsoft.com/library/cc836459(v=vs.94).aspx