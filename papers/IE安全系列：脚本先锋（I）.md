# IE安全系列：脚本先锋（I）

回顾一下，前两篇概述了一下IE的以下内容：IE的历史，各个版本新增的功能、简单的HTML渲染逻辑和网站挂马对IE安全带来的挑战。

从这章开始，将继续以网马为契机，逐渐深入讲述IE的漏洞分析与安全对抗的相关知识。 脚本先锋系列将持续4章，前2章内会介绍网马中常见的加密方式和处理对策。后续会介绍对Shellcode的分析，共2章。

III.1 HTML与网马攻击2 — 权限问题
=====

网马是离不开脚本的，上一章中也介绍了最基础的混淆，或者更准确的说是编码，因为escape的确就是为了编码用的。

我想从实际发生过的网马的例子来介绍这章的内容。

![enter image description here](http://drops.javaweb.org/uploads/images/ecfa4c96925f3b58618cf3e5984902e119eb5345.jpg)

请看以上代码。这个是发生在真实世界中的挂马页面。这些挂马的服务器随时有可能停止，所以我已将该页面存档，请见参考资料(1]（解压密码www.wooyun.org），附件1。由于是由挂马页面直接抓取，杀毒软件可能会报告病毒，如果担心安全问题，建议在虚拟环境内处理样本。

这个网马利用了VBScript整数溢出的漏洞（CVE-2014-6332）。这个著名的漏洞网上也已经有很多分析，如果不了解的话，大家可以参考下这些分析文章。 本章由于只是介绍脚本层面的内容，所以二进制分析将在后续章节进行，视所占篇幅挑选部分漏洞进行分析。

一下将对这一部分涉及到的脚本和攻击点进行简单讲解。

在页面最开始可以看到有一串META的标记，这是因为Internet Explorer 11中（Edge模式）已经不支持VBScript，可以看到为了兼容，挂马代码在最前面要求IE模拟IE8，这样就可将渲染模式强行改为IE8，从而支持VBScript的执行。

```
<meta http-equiv="X-UA-Compatible" content="IE=EmulateIE8" >

```

接着，这个页面中出现了两块SCRIPT标签：

![enter image description here](http://drops.javaweb.org/uploads/images/a7bae8f8ee0d300afef5e77ed3378539052d957d.jpg)

可能有的同学会对代码执行的优先顺序产生疑惑，在IE中脚本的执行顺序是：

（1）谁的块（SCRIPT）在前面谁先执行；

![enter image description here](http://drops.javaweb.org/uploads/images/9d18c70d913eb62e5ac4f40f64484a374903b91a.jpg)

（2）各个分块中，函数优先被解析，但是不执行，函数解析完成后，从最外层的Public代码的第一行开始执行；

![enter image description here](http://drops.javaweb.org/uploads/images/ff421db0371ff2ffdebe54c8d9c30fedfbad9fc4.jpg)

（3）各个分块中如果没有容错语句，遇到错误的代码之后，该分块后面的代码不会运行。但是不会影响到其他分块的内容；

![enter image description here](http://drops.javaweb.org/uploads/images/1b109109351d9c0630c8ed84a2efb9d5920a13d6.jpg)

（4）在Javascript中，容错代码是try{}catch(...){}，在VBScript中，容错代码即为：On Error Resume Next；

![enter image description here](http://drops.javaweb.org/uploads/images/f8fd76b6a9591a8ed9fdab62d2967a7ea6f81cbe.jpg)

![enter image description here](http://drops.javaweb.org/uploads/images/9a7300ea90e11909fe4ec59272906be8e426a204.jpg)

这样，可以看到第一节中定义了一个函数，函数中会调用CreateObject创建wscript.shell， Microsoft.XMLHTTP，ADODB.Stream三个对象。

![enter image description here](http://drops.javaweb.org/uploads/images/b7cffd377152e0193b1b933891f14705386e66e5.jpg)

这三个对象的GUID分别是：

```
Wscript.shell: 72C24DD5-D70A-438B-8A42-98424B88AFB8

Microsoft.XMLHTTP: ED8C108E-4349-11D2-91A4-00C04F7969E8 *(progid: Microsoft.XMLHTTP.1.0)

ADODB.Stream: 00000566-0000-0010-8000-00AA006D2EA4

```

在注册表中查看这Wscript.shell、ADODB.Stream的guid，你会不约而同的发现：

![enter image description here](http://drops.javaweb.org/uploads/images/e5dc15ddd6ad7d042f00a26b01d91c27c2f6e662.jpg)

![enter image description here](http://drops.javaweb.org/uploads/images/e5b85c737b50dd32ed25608bc849bd9bfdae6149.jpg)

对了，这两个ActiveX对象的Safe for script和Safe for init均被设置为了False，这使得他们无法在Internet zone被加载，而在本地域下，使用它们的话，Internet Explorer会弹出：

![enter image description here](http://drops.javaweb.org/uploads/images/aee05c7d8acde825323f3c4f3001c957e9f66c0e.jpg)

Internet 域默认的是中-高级别，在此级别下，这类脚本会被禁止执行（同时触发一个异常）。

![enter image description here](http://drops.javaweb.org/uploads/images/9d10f3b355d07f6904b36f751b2583cc9964e3aa.jpg)

漏洞如果成功执行，会把安全检查的开关关闭，导致这些原先被标记为不安全的对象全部都可以成功创建并运行，这样甚至不需要理会任何现有防御机制（例如ASLR等）即可攻击用户电脑。

![enter image description here](http://drops.javaweb.org/uploads/images/8bd7bcae2f9e3c039fcff2630aa2888aa8ffd63a.jpg)

可以看到该网马最后运行了这个函数。这个函数即会下载一个EXE并运行。该EXE即为木马程序。

而其他的，该网马最后还有这么一句，首先是创建一个iframe指向木马exe：hxxp://116.255.195.114/server.exe，然后又用window.open()打开一个新窗口，url也是这个exe。

![enter image description here](http://drops.javaweb.org/uploads/images/b8c634f524a29390bfd87471288953d620d0b036.jpg)

这两个语句的结果都是让IE（其他浏览器也是如此）弹出来一个下载提示：

![enter image description here](http://drops.javaweb.org/uploads/images/764d2677f2108b4fe8b7973e108685676cf6f6c2.jpg)

这是坑用户的最后一步，只要用户不点运行就没有问题，而且这个URL几乎被国内的所有杀软都入库了，想必也不会骗到多少人。

![enter image description here](http://drops.javaweb.org/uploads/images/d7be5189f08378acbb28853b9209020c5f53dc7b.jpg)

需要注意的是该网站还有一个1.js，这个js文件404了，内容我们不得而知。不过看前面这么明显的“js挂马”字样，也许作者是把这个js的内容也一起并到了这个网页中也说不定。

最终，我们知道了，这个网页是想要让用户下载server.exe。仅仅从杀软的报告来看，这个exe是一个远控程序。针对此类体积适中的木马程序，比较简易和方便的调试环境是Sandboxie+OllyDbg，或者VMWare+调试工具，后者可能占内存较多，前者比较轻量也很容易整理。但是需要注意不要被有些病毒样本穿出了沙箱从而影响到真实系统。这部分内容不在本系列的介绍范围内，不详细叙述了。

![enter image description here](http://drops.javaweb.org/uploads/images/eda804fd6263ce9817a23d53801ec1035415b837.jpg)

在搜刮今天的挂马网站列表的时候，我还看到了如下例子（附件 2），这是类似熊猫烧香的一个病毒Ramnit感染的，他就是我之前说的，给每个html都挂上一段脚本，这个脚本需要本地权限才可以提示执行，不过万一用户点击允许了呢，这样，这个病毒就会重新死而复生了（也许这段代码和上面CVE-2014-6332配合起来才更好）：

![enter image description here](http://drops.javaweb.org/uploads/images/26d3c487d919dfc65657b7a0b2170517473db4ec.jpg)

图：Ramnit感染的例子

该病毒感染的文件几乎可以被所有杀毒软件修复。

III.2 HTML与网马攻击3 — 反混淆
=====

以上针对简单的例子进行了讲解，现在我们将看一些带有混淆的例子。

这一节中的例子并不是十分困难，只要仔细观察，是一定可以轻松解开的。一些比较难解的例子将在下一章中介绍。

1. JS压缩后的结果；
------------

* * *

![enter image description here](http://drops.javaweb.org/uploads/images/9464fc20079996deebcc8461f78a5868f11fc422.jpg)

让我们看一看这个例子，详情见附件3。这个脚本是CVE-2004-0204的一个利用代码，取自以前的某个网马记录，乍一看这个东西代码复杂而恶心无从下手，但是其实如果你记得上一章所说的，eval最终会执行第一个参数内的函数，而这里第一个参数就是一个function，因此只需要将eval替换为alert，执行即可得到内容：

![enter image description here](http://drops.javaweb.org/uploads/images/4939a2ab595f108b530a3afa5ae85f7671dadb73.jpg)

红框处即为木马要下载的文件地址。

不过知其然不知其所以然不太好，让我们简单阅读一下代码：

![enter image description here](http://drops.javaweb.org/uploads/images/01146dd2457085aa17e05df616a4fb69ee9badd7.jpg)

可以看出来函数实际为：

```
eval(
function (p,a,c,k,e,d){}    (p, a , c ,k ,e ,d)
)

```

这实际上是把这个拥有6个参数的匿名函数的返回值传给了eval执行，因此返回值至少是解密过1次的代码。

如果还是不了解，这么看你就明白了：

```
var a = function(){return 1}();
    alert(a.toString());

```

![enter image description here](http://drops.javaweb.org/uploads/images/d6d487b29542b5367ff79573118bf193dcfb1167.jpg)

最初（2007），除了极少的JS库之外，这种代码大多数都被用在挂马上，不过之后，jspack倒是由于它有压缩代码的功能，被很多网站采用了。如果你也想生成这样的代码，不妨百度搜索一下eval压缩。

2、简单的代码阅读
---------

* * *

![enter image description here](http://drops.javaweb.org/uploads/images/408d49fbfcd4ca78082e5d3b633c20d8f40f9e5d.jpg)

这个页面中（附件4），我们看到有一段加密的代码十分奇怪，

![enter image description here](http://drops.javaweb.org/uploads/images/3d7f0bd962403ed6247415999920a8599df83ce0.jpg)

通过阅读代码可知这段代码其实就两段：

```
定义函数xViewState()；
调用函数xViewState()。

```

通过阅读函数xViewState，我们可以发现前半段都是在解密数据，而与页面或者脚本有交互的地方仅有document.write一处，因此，将document.write替换成alert即可知道最后它要写入页面的内容。

请注意，document.write写入DOM的内容会立即被渲染执行。

![enter image description here](http://drops.javaweb.org/uploads/images/6e133420becf687e39c5d80c742942360741a1ee.jpg)

看来它是在写入一段style信息，将.nemonn移动到-9999px top的地方，这表示这个内容将不在页面的可视范围内。为什么要这么做呢？想必你也知道了：挂黑链。

该页面中另一处隐藏的地方，阅读这个代码也许你就更清楚它想干什么了：

![enter image description here](http://drops.javaweb.org/uploads/images/72d4e4862bf5d0e79cbd37182de492cddb391b68.jpg)

![enter image description here](http://drops.javaweb.org/uploads/images/89aee32d22f58d8f00478796433f3a11c70b5bab.jpg)

图：黑链不显示在首页的代码

这种方式已经被Google列为打击对象。用脚本加密的方式倒是可以算作是与Google的一个“对抗”。

3、工具处理
------

* * *

鉴于javascript中可以轻易地劫持一个对象，因此我提供的工具中也有简单的替换功能：

![enter image description here](http://drops.javaweb.org/uploads/images/732d7ef68eb8f7ab2a50c89fb83559126482c9e0.jpg)

![enter image description here](http://drops.javaweb.org/uploads/images/a508ac059c6c758d9055ab4a3f6d1eba0227683e.jpg)

4、Exploit Kit示例
---------------

这是臭名昭著的Nuclear Exploit Kit的载入页（学名Landing Page）一个较简单的例子。

![enter image description here](http://drops.javaweb.org/uploads/images/8b2bac5b888d31235e75ab7a467a87b53e88afdf.jpg)

图：Nuclear EP的Landing Page 观察页面（附件5），可以发现页面结构类似：

```
<SCRIPT> ... </SCRIPT>
 <ELEMENTS>  DATA  </ELEMENTS>
<SCRIPT> ... </SCRIPT>

```

三段，由于ELEMENTS的内容必然是不能执行的，所以分析的重点应该放在SCRIPT中间的内容。

![enter image description here](http://drops.javaweb.org/uploads/images/0a09ec39b5e9d2f33f4d7a3ca4f01ef0e5d80124.jpg)

先对第一段去混淆。可以发现代码中注释占了一半的篇幅，所以先批量删除。

![enter image description here](http://drops.javaweb.org/uploads/images/beb5fc9ba6390d3a1d06ef6cd1ac5a43c2ef66d5.jpg)

然后将JS代码格式化，

![enter image description here](http://drops.javaweb.org/uploads/images/43abb4e0febffc7c8118e4a0e10a73827ebeb33a.jpg)

然后稍作整理（附件6），可以看到此时几乎已经很容易就能知道这段代码做的什么了：

页面的第三个SCRIPT块（附件6， LN78）

```
<script>aiTsnQh(EOHCnD("iaTyv"));</script>

```

事实上调用了EOHCnD 这个函数，这个函数的定义是：

![enter image description here](http://drops.javaweb.org/uploads/images/b17976d5812db33abf0250b0f5ab9f346c40eae7.jpg)

阅读可知，

```
LN29：生成对象document；
LN30：调用document[”getElementById”](divId).innerHTML[”replace”](/[ ]/g,’’)将空格删除；
LN32-33： 实际是substr的混淆；
LN37-50：从第一个字节开始，每2个字substr一下，转为数字，如果小于10原样不动，大于10的话-2，然后保存在MvBLCx变量中
LN52： 返回解密后的字符。

```

也就是说，很简单，这个EOHCnD就是解密的函数，因此，我们执行页面并把它的返回值输出即可。

第三个SCRIPT块改为Console.log：

![enter image description here](http://drops.javaweb.org/uploads/images/b618b8a30945107d223c6671c4382ad843898979.jpg)

得到解密后的内容（附件 7）。该脚本会将参数传给漏洞利用程序（SWF， 附件8）来执行。SWF的内容之后再提。

以上就是本章内所有解密内容，大家可以对照附件的恶意脚本进行一些解密试验。接下来，再概述一下IE中ActiveX的一些知识。

III.3 IE渲染网页时ActiveX处理方式和安全限制
=====

在IE渲染网页时，ActiveX对象一直是漏洞挖掘者喜闻乐道的东西。ActiveX控件是指基于COM（微软的组件对象模型）设计出来的一种可以重用的组件。因为它能“Active”在各种东西上，所以大概就因此叫了这个名字。

ActiveX控件可以通过`<OBJECT>`标签，或者脚本中CreateObject或者new ActiveXObject的方式创建一个实例。

![enter image description here](http://drops.javaweb.org/uploads/images/edb9e68736d8c1ad380973441a81d65433670bc8.jpg)

图：XP下CVE-2010-0886溢出漏洞的利用代码，正在向对象传入一个过长的docbase参数

ActiveX对象是一个二进制文件，那么如果这个二进制文件中包含有一些危险操作，那么必然可以对用户机器做一些不好的事情。因为ActiveX控件几乎可以做所有普通程序可以做的事情，所以恶意的ActiveX将是十分致命的，尤其是在IE中加载起网页指定的ActiveX控件，安全和便利又因此发生了冲突，是好是坏赞否两论。

关于它的反面说法其一是由于“历史问题”，XP下ActiveX与IE权限等同，且大部分人都是管理员权限登录的，所以导致ActiveX也有管理员权限。该问题在Vista引入的IE保护模式中得到改善。

简单介绍一下ActiveX的安全标记Safe For Scripting。标记为Safe For Scripting的控件理应不会被任何不信任的脚本（简单说就是别人提供的，开发者也无法预见的内容）恶意利用，比如泄露隐私，执行文件，或者干脆干扰了其他软件正常功能。

还有一个就是参数的传入，当传入的初始化数据是不可信的时候（比如我指定一个控件背景色是RGB(999, -1, "abc")的时候），插件也不能崩或者因此就不工作了（Safe For Initializing），谁知道用户会传给你什么呢。

要想让ActiveX可以参与IE的脚本互动，必须要确保对任何脚本宿主来说这个插件都能安全执行，也要把插件注册为“Safe For Scripting”。

有两种方式可以这么来，一是在注册表中写入键值，二是继承IObjectSafety接口（ATL也提供了个IObjectSafetyImpl方便你操作）。

![enter image description here](http://drops.javaweb.org/uploads/images/a42dd91d9cee93b1bd01d8e4ff2b74d2957b14f9.jpg)

图：一个控件使用IObjectSafety的例子

IObjectSafety有GetInterfaceSafetyOptions、SetInterfaceSafetyOptions这对函数，GetInterfaceSafetyOptions应当返回ActiveX控件的安全特性（Safe For Init？Safe For Scripting？），SetInterfaceSafetyOptions由宿主调用，告诉控件应当具有什么安全特性。

![enter image description here](http://drops.javaweb.org/uploads/images/37aa0870b6d987dcfed822b0cb7d1a0de0403781.jpg)

图：IObjectSafety的定义，参考VC运行库中objsafe.h的具体代码

![enter image description here](http://drops.javaweb.org/uploads/images/8ff4b198092d79ab747612dd4650a61a398364cb.jpg)

图：GetInterfaceSafetyOptions的实现，参考VC运行库中atlctl.h的具体代码

这部分的具体内容可以参考《COM本质论》。

让我们简单讨论一下ActiveX在IE中的实现，之前说到，`<XX>`括起来的元素多是继承了CElement，`<OBJECT>`也不例外，OBJECT对应的类为CObjectElement，继承于CElement。

在解析到OBJECT时，该对象会：

```
1. 试图读取参数，找到CLSID和其他参数信息；

2. 读取CODEBASE的值并解析存入Property Bag，这个值可以是： 

2.1 绝对URL；（http://drops.wooyun.org/xx.cab#version=xxx）

2.2 相对URL；（xx.cab#version=xxx）

2.3 无URL；（#version=xxx）

3. 读取其他参数并解析，存入Property Bag；

4. 加载该OBJECT；

```

在加载OBJECT时，IE会：

```
1. 检查缓存，这个缓存会缓存一些指向IDispatch的指针，如果已经命中缓存了，这次就不需要再去Query了；

2. 确保ActiveX控件可以安全加载（SafeForScripting）且可以访问；如果这步检测失败，IE会返回E_ACCESSDENIED。

```

IsSafeToScript 是COleSite的一个函数，该函数会：

```
1. 检查用户是否已经关闭了ActiveX的安全检测（检查域是否设置了URLACTION_ACTIVEX_OVERRIDE_SCRIPT_SAFETY）；

```

![enter image description here](http://drops.javaweb.org/uploads/images/ae41af1c8fbe5d1cff3c86109ca9756349bba85c.jpg)

图：MSDN，https://msdn.microsoft.com/en-us/library/ms537178.aspx

```
2. 如果当前内容是Java Applet，检查用户是否允许Applet加载，不允许直接返回禁止；

3. 检查控件的IObjectSafety属性，标记为Safe For Scripting时通过；

4. 当标记为不通过且用户选择提示时，弹出提醒，告诉用户要加载的ActiveX插件不安全；

5. 安全时载入该控件。

```

以上就是IE加载ActiveX控件时的通用动作，下面我们再简单介绍一下IE针对ActiveX控件做的一些安全改进：

自IE7开始，引进了Activex Opt-In，这个功能的作用是：默认关闭大部分的ActiveX，当网站请求执行某个ActiveX控件的时候，IE会弹一个信息条：

![enter image description here](http://drops.javaweb.org/uploads/images/8863e671947aa2c980b159ae703b0fab933fedec.jpg)

图：信息条，via Google Image

![enter image description here](http://drops.javaweb.org/uploads/images/d3ceb16561f39dafaff53ffa0f29c8e210a85cfd.jpg)

图：安全警告，via Google Image

当用户确定时，这个控件才会加载。

这些插件不属于“大部分”之列：

```
a. 升级到IE7之前已经用过的插件；
b. IE7预存了一个白名单，这里面都是经过检验的，而且很多都是常见的ActiveX；
c. 用户在浏览器中下载并安装的控件；

```

![enter image description here](http://drops.javaweb.org/uploads/images/af0cdad6f0ca3cea71a0e0a30624fd508fcf2bc8.jpg)

图：白名单，位置：HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Ext\PreApproved

2、IE8 （+Vista）引入了Per-User (Non-Admin) ActiveX，它允许用户以非管理员权限安装ActiveX控件，微软声称此举是为了让用户更好的使用UAC特性，因为如果你以普通用户权限安装了一个恶意的ActiveX控件，除了影响当前用户，整体的系统安全倒不会受到严重影响，因为这个ActiveX控件也是和当前用户一个权限。

IE8中，ActiveX也可以按网站开启。从这个时候开始，KillBits功能还被整合进了Windows Update，这样微软就可以在ActiveX出现问题之后收拾残局。

Vista中IE还引入了保护模式，保护模式下IE运行在低完整性级别，这意味着即使ActiveX被攻破，也不能写入一些敏感数据。

3、IE9中，增加了ActiveX Filtering功能，可以让用户在所有网站禁止运行控件而不会弹出提示。

4、IE10 中ActiveX控件的加载会经历多个检测，包括组策略，权限检查等；ActiveX控件有和浏览器等同的权限。当开启EPM之后，只有支持EPM（同时有支持32/64位文件，且兼容AppContainers）的ActiveX控件才会被加载。

5、IE11 （+Windows 8）会自动扫描ActiveX并阻止恶意ActiveX运行。

同时，微软还推送了一些Out-Of-Date ActiveX功能，这个估计是学的Chrome和Firefox，把一些过时的ActiveX屏蔽掉。

6、 Spartan（IE12），不支持ActiveX和BHO。

而是用一个“扩展系统”来安装。“旧风格”（内网，需要旧版本支持的网站）网站使用IE11来渲染。(4]

可见，微软对自己的这套东西是有多爱就有多恨，至于Spartan到底能不能完成这一系列的安全进步和兼容性过渡，就要看微软之后到底怎么完善它的“扩展系统”了，是变得更安全还是又多一个烂摊子，让我们拭目以待。

附 参考资料
------

* * *

(1][下载地址](http://drops.wooyun.org/wp-content/uploads/2015/04/maliciousscripts.rar)本文中所有例子，解压密码www.wooyun.org

(2][下载地址](http://drops.wooyun.org/wp-content/uploads/2015/04/redoce.rar)自己写的Redoce 3解密工具，2013年3月最后一版，已不再维护

(3] https://www.blackhat.com/presentations/bh-usa-08/Kim/BH_US_08_Kim_Vista_and_ActiveX_control_WhitePaper.pdf

(4] http://blogs.msdn.com/b/ie/archive/2015/01/22/project-spartan-and-the-windows-10-january-preview-build.aspx