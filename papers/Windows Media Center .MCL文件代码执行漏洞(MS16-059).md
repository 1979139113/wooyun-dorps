# Windows Media Center .MCL文件代码执行漏洞(MS16-059)

0x00 简介
=====

漏洞作者EduardoBraun Prado在今年早期发现了WMP的.MCL文件又存在一个可以导致远程代码执行的漏洞。为什么要说又呢，因为这个东西实在是“不争气”，同一个地方出现了多次绕过导致远程代码执行的问题。

0x01 历史A――MS15-100
=====

2015年的MS15-100（CVE-2015-2509，Aaron Luo, Kenney Lu, and Ziv Chang of TrendMicro）漏洞便是MCL“噩梦”的开始，研究者发现只要在MCL文件中指定：

```
<application run="c:\windows\system32\cmd.exe"></application>

```

便可以让WMP运行cmd.exe，代码层面看，由于MCL是一个XML文件，WMP加载时只是简单的解析了该XML，便直接运行了“run”中指定的内容。看起来相当令人无语。

0x02 历史B――MS15-134
=====

紧随着MS15-100中微软将run的内容做了过滤之后，MS15-134（CVE-2015-6127，Francisco Falcon of CoreSecurity）又诞生了。这回作者利用的倒不是run这个属性，而是另一个url属性。

通过application的url属性指定一个文件后，WMP会在自己的WebBrowser中加载这个URL。但是研究者发现，WMP的exe并不在Local Machine Lockdown的列表（`HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\InternetExplorer\MAIN\FeatureControl\FEATURE_LOCALMACHINE_LOCKDOWN`）中，因此在WMP的WebBrowser中打开的本地页面中的脚本将会自动运行。

但是总归不能分发两个文件到用户电脑中吧，于是作者想到了更“优雅”的方式：在application标签中加入脚本。因为WMP的XMLParser把不认识的APPLICATION的子元素全部忽略，如果攻击者构造攻击文件a.mlc：

```
<application url="a.mlc">
    <script>alert(1)</script>
</application>

```

注意url指向自身，这样这个文件又会被当作HTML文件加载，由于WebBrowser（IE）处理HTML的宽松的特性――不认识的标签会被忽略，只去渲染认识标签的含义。就导致WebBrowser中直接渲染并在file域下执行了文件包含的脚本。接下来能干什么事情想必也不用细说了。

0x03 主角――MS16-059
=====

在微软拦截的七七八八的时候，又有人发现了新玩法：

```
<application run="file:///\\127.0.0.1\c$\programdata\cpl.lnk"/>

```

因为微软对UNC作出了拦截，攻击者使用了`file:///`来绕过过滤。同时，微软也对传递的文件后缀进行了拦截，因此作者又传入了lnk（快捷方式）文件进行了绕过。

作者在lnk中指定了调用一个CPL文件，CPL其实也是一个实现了特定接口的DLL文件，因此作者成功地绕过了MCL的拦截，执行了任意代码。

借助于UNC，作者仍然只需要传入一个MCL文件给用户即可，其他的所有恶意代码完全依靠远程拉取。

0x04 MCL是如何被处理的？（原版・14年版本）
=====

因为虚拟机打补丁实在是太慢了，所以这里就直接用14年的DLL来演示了，看一个原始风貌。

MCL文件如何处理？打开注册表，查看MCL文件相关关联，可知是由ehshell.exe打开的：`HKEY_LOCAL_MACHINE\SOFTWARE\Classes\.mcl\OpenWithList\ehshell.exe`。

观察ehshell.exe的代码可以发现，ehshell.exe简单地加载了ehshell.dll，之后所有逻辑都是再ehshell.dll去做的。

从调用栈也可以得出这个结论（双击mcl文件之后的栈）：

```
0:025> bp kernel32!CreateFileW
0:025> g
Breakpoint 0 hit
kernel32!CreateFileW:
00000000`779b1b88 ff2522b90700    jmp     qword ptr [kernel32!_imp_CreateFileW (00000000`77a2d4b0)] ds:00000000`77a2d4b0={KERNELBASE!CreateFileW (000007fe`fd944600)}
0:000> dqs esp
00000000`0021e108  00000000`779a0dad kernel32!CreateFileWImplementation+0x7d
00000000`0021e110  00000000`80000000
00000000`0021e118  00000000`00000000
00000000`0021e120  00000000`0000006c
00000000`0021e128  00000000`0039d2e0
00000000`0021e130  00000000`00000003
00000000`0021e138  00000000`00100000
00000000`0021e140  00000000`00000000
00000000`0021e148  00000000`00000000
00000000`0021e150  00000000`00440042
00000000`0021e158  00000000`03a7ed88
00000000`0021e160  00000000`00100000
00000000`0021e168  000007fe`f116bec7 mscorwks!DoNDirectCall__PatchGetThreadCall+0x7b
00000000`0021e170  00000000`00000000
00000000`0021e178  00000000`0021e1d0
00000000`0021e180  00000000`00000003
0:000> du 00000000`03a7ed88
00000000`03a7ed88  "C:\Users\BlastTS\Desktop\test.mc"
00000000`03a7edc8  "l"

```

可以看到LaunchMediaCenter之后，ehshell基本就啥都不干了。

```
# Child-SP          RetAddr           : Args to Child                                                           : Call Site
00 00000000`0021e108 00000000`779a0dad : 00000000`80000000 00000000`00000000 00000000`0000006c 00000000`0039d2e0 : kernel32!CreateFileW
01 00000000`0021e110 000007fe`f116bec7 : 00000000`00000000 00000000`0021e1d0 00000000`00000003 000007fe`f0331830 : kernel32!CreateFileWImplementation+0x7d
*** WARNING: Unable to verify checksum for C:\Windows\assembly\NativeImages_v2.0.50727_64\mscorlib\c3beeeb6432f004b419859ea007087f1\mscorlib.ni.dll
02 00000000`0021e170 000007fe`f0207f71 : 000007fe`f0331830 000007fe`eff69210 00000000`03a7ee70 000007fe`f0287aea : mscorwks!DoNDirectCall__PatchGetThreadCall+0x7b
03 00000000`0021e230 000007fe`f0207e2e : 00000000`03a7edd8 00000000`03a7edd8 00000000`03a7ee00 000007fe`f0288e86 : mscorlib_ni+0x317f71
……………………………………………………
17 00000000`0021f3b0 000007ff`00250222 : 00000000`02681eb0 00000000`02681f30 00000000`0021f4a0 00000000`0021f060 : 0x7ff`00252817
18 00000000`0021f5b0 000007fe`f116a21a : 40ca06f5`582e02ef bdb2bd08`2c45789a 00000000`00000000 00000000`00000000 : 0x7ff`00250222
19 00000000`0021f5e0 00000001`3f7e161b : 00000000`01e10020 00000000`003bcfa8 00000000`0021f6a0 00000000`00000000 : mscorwks!UMThunkStubAMD64+0x7a
1a 00000000`0021f670 00000001`3f7e153a : 00000000`00312462 00000000`00000000 00000000`003283c0 00000000`01e10020 : ehshell!LaunchMediaCenter+0x14e
1b 00000000`0021f6c0 00000001`3f7e148d : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : ehshell!wWinMain+0xc3
1c 00000000`0021f920 00000000`779a59ed : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : ehshell!InitializeCOMSecurity+0x382
1d 00000000`0021f9e0 00000000`77adb371 : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : kernel32!BaseThreadInitThunk+0xd
1e 00000000`0021fa10 00000000`00000000 : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : ntdll!RtlUserThreadStart+0x1d

```

由于MediaCenter以及ehshell.dll都是C# DLL，因此WinDbg看就显得比较捉急了，我们将其反编译，生成源代码直接看：

![pic1](http://drops.javaweb.org/uploads/images/8fcbc4f24edba5e6721bfd66ef78a1284cb8667c.jpg)

工程十分庞大，在漫长的生成过程后，搜索关键字可以看到这样的逻辑：在加载MCL之后，首先程序解析.MCL文件（通过System.XML），并在InternalExtensibilityAppInfo中为包含有Run属性的MCL创建一个特殊的“AppEntryPoint”。

```
namespace MediaCenter.Extensibility
{
……
public ExtensibilityEntryPointInfo AddExternalAppEntryPoint(string strRun,string strTitle, string strDescription, string strCategory, object objContext,string strNowPlayingDirective)
{
IDictionarydictEntryPoint = CreateBaseEntryPointDict(Guid.NewGuid(), strTitle,strDescription, strCategory, objContext, strNowPlayingDirective);
dictEntryPoint[Run] = strRun;
return this.AddEntryPoint(dictEntryPoint);
}

```

在处理AppEntryPoint时，程序啥校验都不做，直接Process.Start启动程序。当然，这是最初的版本，因此没有任何弹框或判断。

```
namespace MediaCenter.Extensibility
{
………………

 internal classExtensibilityExternalAppEntryPointInfo : ExtensibilityEntryPointInfo
 {
………………

internal override LaunchResult Launch(ref object objState)
{
………………
Process process = null;
try
{
process = Process.Start(base._strRun);
}
catch (Exception exception)
………………

```

0x05 新版如何过滤？（16年6月补丁后版本）
=====

简单粗暴，所有包含Run/Url的全部弹窗二次确认。这也是微软对前几个严重问题所做的非常暴力的“折中”处理。

![pic2](http://drops.javaweb.org/uploads/images/6387edc5de47bd35b197b91abe03a46bcf59310b.jpg)

0x06 十年以上的漏洞
=====

而有趣的是，这个漏洞看起来感觉有十年以上。MCL文件最初就是为了在WMP中执行一个程序用的，这有点像HTA文件，在查找相关信息的时候，我发现了06年（肯定还有更早的）论坛帖子就有关于MCL如何启动程序的讨论了。非常有意思的是，当时的人们并不觉得这个功能是一个安全漏洞，而只是一个功能而已，但是到了2015年第一次被人报告开始，微软就给了这个问题最高的安全评级――但是这个原本是正常的“功能设计”。可见安全的概念在十年间转换了很多。

例如下面的代码是曾经一个帖子“如何在Xbox 360上运行Zsnes”的解决代码，和MS15-100的利用代码一模一样，发帖时间是2012年。

```
<application
name = "Zsnes"
SharedViewport = ""
NowPlayingDirective = ""
run = "C:/Zsnes/Zsnes.exe">
<capabilitiesrequired
directx="True"
audio="False"
video="False"
intensiverendering="True"
console="False"
cdburning="False" />
</application>

```