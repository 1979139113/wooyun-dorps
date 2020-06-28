# Xcode编译器里有鬼 – XcodeGhost样本分析

作者： 蒸米，迅迪 @阿里移动安全

0x00 序
=====

事情的起因是@唐巧_boy在微博上发了一条微博说到：一个朋友告诉我他们通过在非官方渠道下载的 Xcode 编译出来的 app 被注入了第三方的代码，会向一个网站上传数据，目前已知两个知名的 App 被注入。

![enter image description here](http://drops.javaweb.org/uploads/images/b1ee97ebecdde882cb6f24bc3d1b4120f28f2a0b.jpg)

随后很多留言的小伙伴们纷纷表示中招，@谁敢乱说话表示：”还是不能相信迅雷，我是把官网上的下载URL复制到迅雷里下载的，还是中招了。我说一下：有问题的Xcode6.4.dmg的sha1是：`a836d8fa0fce198e061b7b38b826178b44c053a8`，官方正确的是：`672e3dcb7727fc6db071e5a8528b70aa03900bb0`，大家一定要校验。”另外还有一位小伙伴表示他是在百度网盘上下载的，也中招了。

0x01 样本分析
=====

在@疯狗 @longye的帮助下，@JoeyBlue_ 为我们提供了病毒样本:`CoreService`库文件，因为用带这个库的Xcode编译出的app都会中毒，所以我们给这个样本起名为：`XCodeGhost`。`CoreService`是在”`/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/Library/Frameworks/CoreServices.framework/`”目录下发现的，这个样本的基本信息如下：

```
$shasum CoreService
f2961eda0a224c955fe8040340ad76ba55909ad5  CoreService

$file CoreService
CoreService: Mach-O universal binary with 5 architectures
CoreService (for architecture i386):    Mach-O object i386
CoreService (for architecture x86_64):  Mach-O 64-bit object x86_64
CoreService (for architecture armv7):   Mach-O object arm
CoreService (for architecture armv7s):  Mach-O object arm
CoreService (for architecture arm64):   Mach-O 64-bit object

```

用ida打开，发现样本非常简单，只有少量函数。主要的功能就是先收集一些iPhone和app的基本信息，包括：时间，bundle id(包名)，应用名称，系统版本，语言，国家等。如图所示：

![enter image description here](http://drops.javaweb.org/uploads/images/55eb2325e4a6c9323eeb68517051a1fe3d086702.jpg)

随后会把这些信息上传到 init.icloud-analysis.com。如图所示：

![enter image description here](http://drops.javaweb.org/uploads/images/600605b339c2269cdcfa31271f47ec769ed25ffe.jpg)

`http://init.icloud-analysis.com`并不是apple 的官方网站，而是病毒作者所申请的仿冒网站，用来收集数据信息的。

![enter image description here](http://drops.javaweb.org/uploads/images/56053876e1ffbb29136dd17b2f4ae53592b4af3d.jpg)

目前该网站的服务器已经关闭，用whois查询服务器信息也没有太多可以挖掘的地方。这说明病毒作者是个老手，并且非常小心，在代码和服务器上都没有留下什么痕迹，所以不排除以后还会继续做作案的可能。

0x02 检测方法
=====

为了防止app被插入恶意库文件，开发者除了检测”`/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs`” 目录下是否有可疑的`framework`文件之外，还应该检测一下`Target->Build Setting->Search Paths->Framework Search Paths`中的设置。看看是否有可疑的`frameworks`混杂其中：

![enter image description here](http://drops.javaweb.org/uploads/images/17bc84bca6c328ae26087f7baa003cd33f28d503.jpg)

另外因为最近iOS dylib病毒也十分泛滥，为了防止开发者中招，支付宝的小伙伴还提供了一个防止被dylib hook的小技巧：在Build Settings中找到“`Other Linker Flags`”在其中加上”`-Wl,-sectcreate,__RESTRICT,__restrict,/dev/null`”即可。

![enter image description here](http://drops.javaweb.org/uploads/images/0c9806dac8a995197d56baa6e3c5b1750c569852.jpg)

最后的建议是：以后下载XCode编译器尽可能使用官方渠道，以免app被打包进恶意的病毒库。如果非要用非官方渠道，一定要在下载后校验一下哈希值。

0x03 思考&总结
=====

虽然XCodeGhost并没有非常严重的恶意行为，但是这种病毒传播方式在iOS上还是首次。也许这只是病毒作者试试水而已，可能随后还会有更大的动作，请开发者务必要小心。这个病毒让我想到了UNIX 之父 Ken Thompson 的图灵奖演讲 “`Reflections of Trusting Trust`”。他曾经假设可以实现了一个修改的 tcc，用它编译 su login 能产生后门，用修改的tcc编译“正版”的 tcc 代码也能够产生有着同样后门的 tcc。也就是不论`bootstrap`(用 tcc 编译 tcc) 多少次，不论如何查看源码都无法发现后门，真是细思恐极啊。

0x04 追加更新
=====

1 很多开发者们担心最近下载的Xcode 7也不安全。这里笔者没有使用任何下载工具的情况在苹果官网上下载了Xcode_7.dmg并计算了sha1的值。

```
http://adcdownload.apple.com/Developer_Tools/Xcode_7/Xcode_7.dmg
$ shasum Xcode_7.dmg
4afc067e5fc9266413c157167a123c8cdfdfb15e  Xcode_7.dmg

```

![enter image description here](http://drops.javaweb.org/uploads/images/c1458b0fbfa6667839dc2bb28ee6460ab67addd9.jpg)

所以如果在非App Store下载的各位开发者们可以用shasum校验一下自己下载的Xcode 7是否是原版。

2 @FlowerCode同学通过分析流量发现病毒开发者的服务器是搭建在Amazon EC2的云上的。服务器已经关闭 。

![enter image description here](http://drops.javaweb.org/uploads/images/d95ba816879ec99d67aa0be0d7de1380d9e86e22.jpg)

3 根据palo alto networks公司爆料，被感染的知名App Store应用为”网易云音乐”！该app应用在AppStore上的最新版本2.8.3已经确认被感染。并且该应用的Xcode编译版本为6.4（6E35b），也就是XcodeGhost病毒所感染的那个版本。网易云音乐在AppStore上目前的状态：

![enter image description here](http://drops.javaweb.org/uploads/images/ede3cd3d37516a27762d46e623f9978f27f03ed5.jpg)

![enter image description here](http://drops.javaweb.org/uploads/images/a72d67965892776f6245f6c9411a9bbdfd6caac0.jpg)

"网易云音乐"逆向后的函数列表，可以找到XcodeGhost所插入的函数：

![enter image description here](http://drops.javaweb.org/uploads/images/cfa026e5b5e3917b35032bdf80546fdb20088e9b.jpg)

受感染的"网易云音乐"app会把手机隐私信息发送到病毒作者的服务器”init.icloud-analysis.com”上面：

![enter image description here](http://drops.javaweb.org/uploads/images/eef5c858a8231f9bd70c3d5da0ee834911215c86.jpg)

4 根据热心网友举报，投毒者网名为”coderfun”。他在各种iOS开发者论坛或者weibo后留言引诱iOS开发者下载有毒版本的Xcode。并且中毒的版本不止Xcode 6.4，还有6.1，6.2和6.3等等。

![enter image description here](http://drops.javaweb.org/uploads/images/3fecc180b46dc4a13d033fa2b6879e1dc3416a68.jpg)

![enter image description here](http://drops.javaweb.org/uploads/images/00f46eacd709f29b3819a8ae096229251c227ee3.jpg)

![enter image description here](http://drops.javaweb.org/uploads/images/56a15aec49479e0d08966c7c761e66fbd06e0dd1.jpg)

5 根据热心网友提醒，我们在中信银行信用卡的应用”动卡空间”中也发现了被插入的XcodeGhost恶意代码，受感染的版本为3.4.4。

![enter image description here](http://drops.javaweb.org/uploads/images/9c9de5bee563d9cb1a978af0d53785a3f0e6874a.jpg)

![enter image description here](http://drops.javaweb.org/uploads/images/1ef24be743c0830583b0fddd5f05e540e86cfb97.jpg)

被插入的部分恶意代码如下：

![enter image description here](http://drops.javaweb.org/uploads/images/15b05f24289dbd8ba3d815205814693bc4029739.jpg)

6 在`@Saic`的提示下我们发现受感染的app还拥有接收黑客在云端的命令并按照黑客的指令发送URLScheme的能力：

![enter image description here](http://drops.javaweb.org/uploads/images/b5b43dbf9f1644427a4e4f12f12996022c3e8a7b.jpg)

URLScheme是iOS系统中为了方便app之间互相调用而设计的。你可以通过一个类似URL的链接，通过系统的Openurl来打开另一个应用，并可以传递一些参数。一个恶意的app通过发送URLScheme能干什么呢？

```
<1>. 可以和safari进行通讯打开钓鱼网站。
<2>. 可以和AppStore进行通讯诱导用户下载其他AppStore应用。
<3>. 可以和itms-services服务通讯诱导用户下载无需AppStore审核的企业应用。
<4>. 向支付平台通讯发送欺骗性的支付请求等。

```

7 目前已经官方确认中毒的app有：微信，网易云音乐，豌豆荚的开眼等。

![enter image description here](http://drops.javaweb.org/uploads/images/1809d9498ba987604215cd46a90f355317cc296f.jpg)

![enter image description here](http://drops.javaweb.org/uploads/images/d4cb7c599a50584f782c531a17d1e5acaecef777.jpg)

![enter image description here](http://drops.javaweb.org/uploads/images/3d2addc3040367a8e1955087a01cf3b14c670069.jpg)

0x05 病毒作者现身
=====

微博地址：http://weibo.com/u/5704632164

![enter image description here](http://drops.javaweb.org/uploads/images/82d476ea0eeb7e42dfd81c5472fa0c08134c5b81.jpg)

发表长微博内容：

> "XcodeGhost" Source 关于所谓”XcodeGhost”的澄清
> 
> 首先，我为XcodeGhost事件给大家带来的困惑致歉。XcodeGhost源于我自己的实验，没有任何威胁性行为，详情见源代码:https://github.com/XcodeGhostSource/XcodeGhost
> 
> 所谓的XcodeGhost实际是苦逼iOS开发者的一次意外发现：修改Xcode编译配置文本可以加载指定的代码文件，于是我写下上述附件中的代码去尝试，并上传到自己的网盘中。
> 
> 在代码中获取的全部数据实际为基本的app信息：应用名、应用版本号、系统版本号、语言、国家名、开发者符号、app安装时间、设备名称、设备类型。除此之外，没有获取任何其他数据。需要郑重说明的是：出于私心，我在代码加入了广告功能，希望将来可以推广自己的应用（有心人可以比对附件源代码做校验）。但实际上，从开始到最终关闭服务器，我并未使用过广告功能。而在10天前，我已主动关闭服务器，并删除所有数据，更不会对任何人有任何影响。
> 
> 愿谣言止于真相，所谓的"XcodeGhost"，以前是一次错误的实验，以后只是彻底死亡的代码而已。
> 
> 需要强调的是，XcodeGhost不会影响任何App的使用，更不会获取隐私数据，仅仅是一段已经死亡的代码。
> 
> 再次真诚的致歉，愿大家周末愉快