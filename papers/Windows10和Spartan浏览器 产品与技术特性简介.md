# Windows10和Spartan浏览器 产品与技术特性简介

目录

0x00 Windows10产品新特性
-------------------

* * *

1、自动选择默认桌面 2、开始菜单回归 3、Metro应用窗口化 4、虚拟桌面 5、智能分屏 6、全局搜索 7、其它产品新特性 8、Windows10产品新特性总结

0x01 Windows10内置特性和安全特性
-----------------------

* * *

1、In-place升级 2、Stay Current保持最新 3、UPAO (User Protection Always On) 4、Secure ETW Channel（安全ETW通道） 5、Lockdown Mode和虚拟化安全 6、AMSI (Antimalware Scan Interface) 7、WinRE Offline扫描

0x02 微软为什么要开发新浏览器
-----------------

0x03 Spartan产品特性
----------------

* * *

1、取代IE 2、网页标注 3、阅读模式 4、全新的EDGE渲染引擎 5、网络模型 6、号称将支持新的扩展

0x04 Spartan安全特性
----------------

1、框架进程也运行在了AppContainer完整性权限 2、渲染进程（tabs）缺省运行在EPM下 3、IE下的Toolbars、BHO等浏览器扩展都不再支持 4、ActiveX控件受到更加严格的限制，权限极低 5、禁用vbscript，减少攻击利用手段，安全性更高 6、更完善的堆保护 7、CFG默认开启 8、沙箱

0x05 参考文档
---------

0x00 Windows10产品新特性
=====

1、自动选择默认桌面
----------

Windows10在启动时会对用户的硬件设备进行判断，对于PC进入的是经典操作界面（如下图所示），而智能设备（支持触屏）则进入支持触屏手势的开始屏幕。

Windows8的开始屏幕（metro UI）一直被诟病，传统的PC用户显然不买账，对于办公用户来说无疑更喜欢传统桌面，而对于普通用户来说可能对于这个新桌面不知所措。也许windows8和8.1就类似Vista一样是一个实验版本，微软终于明白应该怎么做。赢得用户口碑才是最重要的。

![enter image description here](http://drops.javaweb.org/uploads/images/f9cd019b4559eb4860deab4f34af4ae332704d93.jpg)

2、开始菜单回归
--------

开始菜单从Windows8，8.1到Windows10经历了几次大的演变，最开始竟然把开始菜单整没了，到了8.1的时候搞了一个半吊子开始菜单，点开之后竟然跳转到metroUI，用户显然不买账，几十年的使用习惯，没有革命性的变化，是不太可能改变的。Windows10里面微软终于妥协了，乖乖把开始菜单放出来了，虽然结合了一些metroUI的特点，但是和Windows7相比在使用上已经没有大的区别了。

![enter image description here](http://drops.javaweb.org/uploads/images/fccf303bd8233894f9b82c034b103f5391d8366c.jpg)

3、Metro应用窗口化
------------

在Windows10里面，Metro应用默认以窗口化方式运行，窗口可以最大化，最小化，和普通的窗口程序在操作体验上其实已经很一致了。有些metro应用还提供了全屏功能。 如下面就是“著名”的Spartan浏览器的窗口界面。

![enter image description here](http://drops.javaweb.org/uploads/images/e06fafe97c606badc0a7d7be2085f81461fbbeee.jpg)

而下面这款天气metro应用则提供了全屏功能。

![enter image description here](http://drops.javaweb.org/uploads/images/262d543e065458ea40ddebd5360cc03b2b0aaea5.jpg)

4、虚拟桌面
------

Windows10的任务栏上面新增一个“任务视图”按钮（可以用快捷键Win+Tab直接调出，其实在Window7上面，这个快捷键也可以调出一个3D的任务选择界面出来，大家可以试试），点击后可查看当前桌面正在运行的程序，在右下角有一个新建桌面的＋，可以新建出新的桌面（最多9个），同时在底部区可以快捷的添加、切换、关闭虚拟桌面。虚拟桌面本身不是什么新的技术，有些小软件就可以做到，只能说是对桌面操作系统的一个完善吧。好处就是当一个桌面太繁杂的时候，用户可以在一个新的桌面去做一些新的任务，或者把不同性质的事情放在不同的桌面，提高工作效率，有点类似多显示器体验。

![enter image description here](http://drops.javaweb.org/uploads/images/ba2a0d7e89d1a246b8b2105fe79454dd9766f22b.jpg)

5、智能分屏
------

整个屏幕可以分成4个屏幕来使用，实际上就是把几个任务放到显示器的4个角来分别处理，如果显示器不够大，建议就不要使用了，估计只有那些超大屏幕的用户比较感兴趣吧。操作方法，通过把不同的窗口拖拽到桌面的四个角来分屏放置。

![enter image description here](http://drops.javaweb.org/uploads/images/bf9f831226d5e37bfd4a5b54afbc99fae43eb23b.jpg)

6、全局搜索
------

任务栏上新增了搜索功能，点击后会打开一个小的搜索窗口。默认先搜索本机程序，然后再搜索互联网内容，也会给出相应的搜索建议，相当智能。

![enter image description here](http://drops.javaweb.org/uploads/images/216a0d57cd9b69b3b0e51261e5514a016f45e367.jpg)

7、其它产品新特性
---------

通知中心：后续应该会做成类似于Android和IOS的通知栏。

语音助手Cortana：补齐和竞争对手的差异，平板上面应该更广泛。

SpartanBrowser：全新的浏览器，后面会详细介绍。

8、Windows10产品新特性总结
------------------

明显开始照顾传统用户的感受。

PC端和终端的体验开始大一统。

设计上更加趋于扁平化。

0x01 Windows10内置特性和安全特性
=====

1、In-place升级
------------

为使Windows能够方便地升级到Windows10，微软对Windows7、8和8.1的用户提供了保留软件、配置和数据的In-place升级方式。它通过Windows Update实现系统升级，类似IOS和Android的系统升级方式。

第三方软件要提前布局的是，开发出兼容Windows10的版本，提示升级或自动升级到用户的当前系统，避免系统升级后无法使用。对于杀毒软件，因为有驱动等可以造成新系统Crash的模块，需要额外满足一些要求。

2、Stay Current保持最新
------------------

微软希望尽可能多的用户保持Windows10系统最新，解决当前系统和浏览器碎片化问题，但同时也给企业用户控制更新策略的能力。

Win7及之前的操作系统聚焦于生产力，较少关注用户是否使用最新特性；而Win8、8.1则聚焦于普通用户的设备，尤其是移动设备，对桌面版的效率方面带来负面效应，如Metro开始菜单。

Win10在二者之间做了平衡，并定期持续更新。系统更新主要有两类：

(1) 每月的Patch Tuesday - 安全和可靠性相关的补丁

(2) 定期的Rollup – 新特性、内置软件更新。

3、UPAO (User Protection Always On)
----------------------------------

Windows8以上的用户都安装有安全软件，但并不是所有的用户的安全软件都处于保护状态。微软有过一份统计数据表明，收费安全软件处于非保护状态的比例更大。

根据另外一份根据统计数据，处于安全软件保护状态的用户恶意软件中招率明显低于不处于安全软件保护状态的用户。

因此在Windows10中，引入一个新流程，希望保证用户一直处于AV保护状态。主要用了两个方法，一个是通知提示，一个是过期自动启用Windows Defender。

其中的通知比以前版本做了很大的优化，安全软件过期前5天，系统会通知用户即将过期，引导更新，最后一天的通知是模态的。过期后3天通知用户已过期，引导更新，最后一天的通知也是模态的。

过期当天，Windows Defender会自动启用，同时禁用原安全软件。用户更新原安全软件或安装一个新的安全软件之后，Windows Defender才会失效（兼容性考虑）。

4、Secure ETW Channel（安全ETW通道）
-----------------------------

安全软件为实现某些防护功能，常会Hook内核，来监视系统里进程的各种行为。这种Hook方式容易产生兼容性问题，会对系统内核稳定性造成负面影响。遇到内核更新，可能需要相应更新才能使功能继续生效。

安全ETW通道，是Windows10的一个新安全特性，可扩展，可提供进程行为的实时数据。安全软件的受保护进程，不需内核层Hook就可以监听内核、安全、Win32事件。

5、Lockdown Mode和虚拟化安全
---------------------

为了安全和管理，设备可配置为仅允许受信软件运行的模式。特别适用于财经、政府等安全要求高的行业，这个模式会在企业版和教育版SKU中提供。

启用Lockdown模式，需要虚拟化特性支持Virtual Secure Mode(VT-d)、UEFI 2.4及更新的版本支持安全启动、打开UEFI安全启动选项并禁止用户修改。

基于虚拟化的KMCI（Kernel Mode Code Integrity），为防止一些内核Exploits，Hypervisor层可以执行签名校验检查。Page只有在通过了校验后才会被标记为可执行，动态分配的代码被阻止。

6、AMSI (Antimalware Scan Interface)
-----------------------------------

新增Win32 API，支持软件集成已安装的安全软件扫描能力。当前支持对文件、内存、流、URL、IP的检查。

![enter image description here](http://drops.javaweb.org/uploads/images/8317abdc1afa322dc7192a4d0934a67478efaa63.jpg)

新增Provider接口IAntimalwareProvider提供安全相关服务。

7、WinRE Offline扫描
-----------------

内核rootkit难以在受感染的系统内清除，当前的Offline扫描工具使用比较复杂。因此WSC给第三方AV提供了WinRE执行环境做扫描、清除。

优点是Offline清除Rootkit较容易，不再需要安全软件自己建立Offline环境和使用所需的设备（如CD、USB）。主要要求有，扫描模块需要WHQL签名，兼容WinRE环境，放置在WSC可读取位置等。

0x02 微软为什么要开发新浏览器
=====

IE的一些不好评价，尤其是安全性上的评价，已经直接在影响业界对微软技术能力的评价，很多软件公司甚至不愿意开发与IE兼容的软件。事实上，由于IE浏览器在网页响应速度、抵挡黑客或病毒攻击、对最新技术的兼容度、人性化浏览设置等方面一直存在缺陷，使得近年来软件业围绕浏览器的争夺战愈演愈烈。谷歌、火狐、360等中外软件企业都开发出了自己的浏览器，并各自吸引了一批用户，导致IE的市场份额持续下滑。

新的Spartan浏览器主打轻快安全特点，刚好对应IE的重慢不安全等不良因素，微软计划凭借Spartan扭转在浏览器市场的地位，同时塑造更好的企业形象。

全球浏览器市场份额一览（时间点：2014年11月）。

![enter image description here](http://drops.javaweb.org/uploads/images/b02eaebe208c427fba7a201edf9def9b10d84091.jpg)

统计方法不同导致市场份额有较大差异，但是不变的是IE的市场份额在持续下滑。

![enter image description here](http://drops.javaweb.org/uploads/images/99aac569cfb599d9f15f36b1cac1c604563b5536.jpg)

Spartan在这场角力中将扮演继续沉沦还是逆袭的角色，让我们拭目以待。

![enter image description here](http://drops.javaweb.org/uploads/images/98d60460a63561e93e8fbc0e3587b60ed1ad25a3.jpg)

0x03 Spartan产品特性
=====

1、取代IE
------

取代IE这是肯定的，正式版本将取代IE11的位置，被固定在任务栏。

![enter image description here](http://drops.javaweb.org/uploads/images/d4ef987e3c426827ce5b9b6e246bf27cecf6db53.jpg)

2、网页标注
------

Spartan支持网页标注，并且可以与其它Spartan用户分享。这是要给浏览器带上社交属性了，但不得不说这是一种很好的分享途径。

![enter image description here](http://drops.javaweb.org/uploads/images/d94008e19047bbb7a3716feda7e78c82e4d7a6a3.jpg)

标注的结果可以存放在本地，也可以放到oneNote，前提是你安装了oneNote（Windows默认集成），拥有这一个功能之后，你可以和别人分享你的标注结果。

![enter image description here](http://drops.javaweb.org/uploads/images/295695bcd5004767d354bdcf47e5fa482fc65c74.jpg)

3、阅读模式
------

用过Chrome的扩展“印象笔记”的应该很熟悉这一应用，简直就是逆天。开启阅读模式之后，浏览器通过摘要算法提取当前页面的正文内容之后，覆盖原页面新起一个窗口进行浏览，广告，杂乱的信息都没了，只剩下干净的正文，大家可以通过下面的图片对比体验一下。

![enter image description here](http://drops.javaweb.org/uploads/images/a17a627a5ac0e832b60797e3cd46a006e4daabb9.jpg)

开启阅读模式之后。

![enter image description here](http://drops.javaweb.org/uploads/images/58f476e1d247761b8f85938d44cc972deb76dd98.jpg)

4、全新的edge渲染引擎
-------------

![enter image description here](http://drops.javaweb.org/uploads/images/abe5537c4d16142b43ebb1e3fda74ae4aa0de052.jpg)

抛弃IE的兼容性包袱，跑得更加轻快，对网页新特性支持得也更好，估计能更好的吸引Web开发者。另外值得称道的是Edge引擎的接口（包括EdgeHtml.dll渲染引擎和Chakra.dll JS引擎）完全和IE的引擎一致（MsHtml.dll和jscript*.dll），因此实际上IE是可以直接使用Edge引擎的，实际上也确实可以的，目前Windows10上面的IE11是通过实验功能提供的。在IE地址栏输入about:flags，然后将Enable Experimental Web Platform Features功能设置为已启用，然后应用更改（要禁用设置为Disabled即可）。即上演替代大法，IE的渲染引擎就完全被Edge替代了。

![enter image description here](http://drops.javaweb.org/uploads/images/df680beaa7309df7798af5115dfab9c24470ccf7.jpg)

从这里也可以看出国内其它的IE内核浏览器（譬如QQ浏览器）要使用Edge引擎也是轻而易举的。

5、网络模型
------

网络模型上沿用了IE11的最新架构，效率极高，IE11的成功给了Spartan很多借鉴。

HTTP请求使用了完成端口模型，并用系统高效线程池优化数据收发，每次收数据时，预收1K数据，合理控制收包数量。对HTTP Response也进行了解析优化，只认为第一个回包是HTTP头，其它都是数据，显然这是一个策略上合理改变，同时也提升了解析效率。

6、号称将支持新的扩展
-----------

截至发文前Spartan浏览器还不支持扩展和插件（现在还是Project Spartan呢）。微软之前曾经号称Spartan要兼容Chrome的扩展，如果微软真的能兼容一部分Chrome的扩展，已经很不错了。Chrome的扩展已经被证明是易开发，易使用的，而且不像Native插件饱受漏洞的困扰。

0x04 Spartan安全特性
=====

对于IE11已经启用的安全特性，Spartan也是继续延续的，希望能甩掉IE的不安全帽子，每月的补丁日都来个IE累积安全更新，一年还来一两次IE紧急更新也是醉了。 下面我们了解下这些新的安全特性。

1、框架进程也运行在了AppContainer完整性权限
----------------------------

完整性权限并不是一个新东西。从已有一些浏览器的架构来说，框架进程一般都是Medium权限，Spartan的做法是再次从系统层面降低了框架进程的权限，框架进程运行在AppContainer完整性权限的好处是整个浏览器访问系统资源更加受限。在进行沙箱突破时也会更加困难，在突破Chrome的沙箱时，一种思路就是利用Chrome的框架进程和渲染进程的IPC通信来进行突破，因为Chrome的框架进程的权限还是Medium，权限相对AppContainer来说高了几个等级。

2、渲染进程（tabs）缺省运行在EPM下
---------------------

IE11一般情况下需要启动EPM（增强保护模式）才能让渲染进程运行在AppContainer权限（进沙箱），默认进入显然更加安全。

3、IE下的Toolbars、BHO等浏览器扩展都不再支持
-----------------------------

减少攻击平面，安全性更高。BHO等Native插件本身可能没有什么漏洞，但是BHO使用的一些技术手段，譬如常用的detours等库，因为本身有一些实现上的缺陷，导致漏洞利用变得容易，譬如detours修改了PE头的可执行属性，跳板代码放置在内存中的固定位置等等，直接不支持这些插件，可能对插件作者是一个打击，但是对安全性来说显然是更高的，不用再去考虑这些不可控的东西。

4、ActiveX控件受到更加严格的限制，权限极低
-------------------------

ActiveX控件的泛滥严重影响着IE的安全，虽然一些ActiveX控件的漏洞严格来说不属于浏览器的漏洞，但是严重影响着IE的口碑，看来Spartan是要在这块下很手了。

5、禁用vbscript，减少攻击利用手段，安全性更高
---------------------------

利用vbscript来绕过nozzle已经是比较成熟的技术，直接禁用vbscript则是从源头上堵死了这种利用手段。2014年有一个神一般的漏洞被曝光，CVE-2014-6332，这个漏洞通杀IE3.0~IE11，破坏性可见一斑，这个漏洞本身并不是vbscript的漏洞，但是它利用vbscript的一些语言上的特性，关闭了vbscript的SafeMode模式（传说中的开启上帝模式），然后直接获得系统权限，不用绕什么DEP，ASLR了，也不用布置什么精巧的shellcode，在IE里面想干啥就干啥。

6、更完善的堆保护
---------

在微软没有启用内存保护机制之前，对象和字符串都是从进程堆中分配的，对于UAF漏洞利用来说，一个对象被释放后，可以分配同样大小的字符串，占住之前对象的内存，由于字符串可以随意修改，因此可以修改对象的虚表，在这个对象再次被使用时，将会调用到我们修改后的虚表函数，实现漏洞利用。之后微软启用了隔离堆，对象在隔离堆中分配，字符串在进程堆中分配，这样使得对象被释放后，不能用字符串来占坑修改。缓解UAF漏洞利用。

斯巴达中新增加了一种内存保护堆机制（MemProtectHeap），由chakra.dll自己管理内存分配，不再使用系统的堆机制。首先调用VirtualAlloc分配一大片内存，每次从中取一页，从0开始依次分配。对象释放时，只是简单标记一下，并且清0。只有当一页中所有对象被释放时，这个页才被释放。这种管理机制的特点是内存申请非常快，并且难以占坑，因为需要释放一页中所有的对象，并且要重新申请到这一页。缺点也是显而易见的，它的内存消耗很大，堆喷很方便，并且没有随机性。

7、CFG默认开启
---------

CFG（控制流保护），是对CFI（控制流完整性）的一个实用性实现。完全实现CFI需要在jmp、call 一个寄存器（或者使用寄存器间接寻址）的时候，改写目的地址内容，目的地址有时必须通过动态获得，且改写的开销又很大，这些都给CFI的实际应用造成了一定的困难。但CFG使用一种相对比较巧妙的方法，降低了内存消耗和对系统的影响。

微软在最新的操作系统Windows10当中，对基于执行流防护的实际应用中采用了CFG技术。它是一种编译器和操作系统相结合的防护手段，目的在于防止不可信的间接调用。对基于虚表进行攻击的利用手段可以有效防御。

漏洞攻击过程中，常见的利用手法是通过溢出覆盖或者直接篡改某个寄存器的值，篡改间接调用的地址，进而控制了程序的执行流程。CFG通过在编译和链接期间，记录下所有的间接调用信息，并把他们记录在最终的可执行文件中，并且在所有的间接调用之前插入额外的校验，当间接调用的地址被篡改时，会触发一个异常，操作系统介入处理。

8、沙箱
----

沙箱的发展历史经历了几个阶段，最开始的基于Hook的沙箱应该是最原始的应用，这种沙箱与系统容易产生兼容性问题，维护不方便，基本上系统更新一次（甚至打一个补丁），就可能要更改实现，否则就有可能导致兼容问题。

后来Chrome的沙箱没有使用Hook的机制，而是尽量利用借助系统的一些安全机制，通过控制权限的方式来保护自己。这种沙箱机制已经在黑客大赛中被证明最难攻破，Spartan浏览器显然借鉴了这一做法，而且做得更彻底，连框架进程也被包含在了整个安全体系里面，通过利用系统安全机制，降低浏览器的进程权限，限制它访问资源的权限，达到防攻击的目的。

当然Spartan上面应用的安全特性远远不止上面这些，还有很多已经在IE11里面实现，Spartan直接就沿用过来了，这里就不再做过多解读，有兴趣的童鞋可以去看看隔离堆（反heapspay），内存延迟释放（反UAF攻击）等等安全知识。

电脑管家的网页防火墙已经第一时间对Spartan浏览器进行安全支持，管家用户可以安全浏览上网。

0x05 参考文档
=====

(1] http://www.mydrivers.com/

(2] http://www.cnbeta.com/articles/381555.htm

(3] Living on the edge – our next step in helping the web just work ttp://blogs.msdn.com/b/ie/archive/2014/11/11/living-on-the-edge-our-next-step-in-interoperability.aspx

(4] Project Spartan and the Windows 10 January Preview Build http://blogs.msdn.com/b/ie/archive/2015/01/22/project-spartan-and-the-windows-10-january-preview-build.aspx

(5] http://www.199it.com/archives/261940.html

(6] 有关IE VB神洞CVE-2014-6332 http://blog.vulnhunt.com/index.php/2014/11/18/about_cve-2014-6332/

(7] 斯巴达浏览器 http://baike.baidu.com/item/%E6%96%AF%E5%B7%B4%E8%BE%BE%E6%B5%8F%E8%A7%88%E5%99%A8