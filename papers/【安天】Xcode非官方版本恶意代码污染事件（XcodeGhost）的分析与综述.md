# 【安天】Xcode非官方版本恶意代码污染事件（XcodeGhost）的分析与综述

**微信公众号:Antiylab**

0x00 摘要
=====

Xcode 是由苹果公司发布的运行在操作系统Mac OS X上的集成开发工具（IDE），是开发OS X 和 iOS 应用程序的最主流工具。

2015年9月14日起，一例Xcode非官方版本恶意代码污染事件逐步被关注，并成为社会热点事件。多数分析者将这一事件称为“XcodeGhost”。攻击者通过对Xcode进行篡改，加入恶意模块，并进行各种传播活动，使大量开发者使用被污染过的版本，建立开发环境。经过被污染过的Xcode版本编译出的App程序，将被植入恶意逻辑，其中包括向攻击者注册的域名回传若干信息，并可能导致弹窗攻击和被远程控制的风险。

本事件由腾讯相关安全团队发现，并上报国家互联网应急中心，国家互联网应急中心发出了公开预警，此后PaolAlto Network、360、盘古、阿里、i春秋等安全厂商和团队机构，对事件进行了大量跟进分析、处理解读。目前，已有分析团队发现著名的游戏开发工具Unity 3D也被同一作者进行了地下供应链污染，因此会影响更多的操作系统平台。截止到本版本报告发布，尚未发现“XcodeGhost”对其他开发环境的影响，但安天分析小组基于JAVA代码和Native代码的开发特点，同样发出了相关风险预警。

截止到2015年9月20日，各方已经累计发现当前已确认共692种（如按版本号计算为858个）App曾受到污染，受影响的厂商中包括了微信、滴滴、网易云音乐等著名应用。

从确定性的行为来看，尽管有些人认为这一恶意代码窃取的信息“价值有限”，但从其感染面积、感染数量和可能带来的衍生风险来看，其可能是移动安全史上最为严重的恶意代码感染事件，目前来看唯有此前臭名昭著的CarrierIQ能与之比肩。但与CarrierIQ具有强力的“官方”推广方不同，这次事件是采用了非官方供应链（工具链）污染的方式，其反应出了我国互联网厂商研发“野蛮生长”，安全意识低下的现状。长期以来，业界从供应链角度对安全的全景审视并不足够，但供应链上的各个环节，都有可能影响到最终产品和最终使用场景的安全性。在这个维度上，开发工具、固件、外设等“非核心环节”的安全风险，并不低于操作系统，而利用其攻击的难度可能更低。因此仅关注供应链的基础和核心环节是不够的，而同时，我们必须高度面对现实，深刻分析长期困扰我国信息系统安全的地下供应链问题，并进行有效地综合治理。

0x01 背景
=====

Xcode是由苹果公司开发的运行在操作系统Mac OS X上的集成开发工具（IDE），是开发OS X 和 iOS 应用程序的最快捷的方式，其具有统一的用户界面设计，同时编码、测试、调试都在一个简单的窗口内完成。【1】

自2015年9月14日起，一例Xcode非官方供应链污染事件在国家互联网应急中心发布预警后，被广泛关注，多数分析者将这一事件称之为“XcodeGhost”。攻击者通过对Xcode进行篡改，加入恶意模块，进行各种传播活动，使大量开发者获取到相关上述版本，建立开发环境，此时经过被污染过的Xcode版本编译出的App程序，将被植入恶意逻辑，其中包括向攻击者注册的域名回传若干信息，并可能导致弹窗攻击和被远程控制的风险。

截止到2015年9月20日，各方已经累计发现共692种（如按版本号计算为858个）App确认受到感染。同时，有分析团队认为，相同的攻击者或团队可能已经对安卓开发平台采用同样的思路进行了攻击尝试。从其感染面积、感染数量和可能带来的衍生风险来看，其可能是移动安全史上最为严重的恶意代码感染事件之一，从影响范围上来看能与之比肩的仅有此前臭名昭著的CarrierIQ]【2】。

目前，Unity 3D也被发现由同一作者采取攻击Xcode手法类似的思路进行了地下供应链污染。

鉴于此事态的严重性，安天安全研究与应急处理中心（Antiy CERT）与安天移动安全公司（AVL Team）组成联合分析小组，结合自身分析进展与兄弟安全团队的分析成果，形成此报告。

0x02 作用机理与影响
=====

安天根据Xcode非官方供应链污染事件的相关信息形成了图2-1，其整体污染路径为官方Xcode被攻击者植入恶意代码后，由攻击者上传到百度云网盘等网络位置，再通过论坛传播等方式广播下载地址，导致被App开发者获取，同时对于攻击者是否利用下载工具通过下载重定向方式扩大散布，也有较多猜测。有多个互联网公司采用被污染过的Xcode开发编译出了被污染的App，并将其提交至苹果App Store，且通过了苹果的安全审核，在用户获取相关App进行安装后，相关应用会被感染并回传信息至攻击者指定域名。

![](http://drops.javaweb.org/uploads/images/76e4796ae4f1e495523970708190589b1ebb7838.jpg)

图2‑1 Xcode非官方供应链污染事件示意图

2.1 作用机理
--------

### 2.1.1 样本信息

*   文件名：CoreService
*   位于Xcode位置：（iOS、iOS模拟器、MacOSX三个平台）
    
    ```
    ./Developer/Platforms/iPhoneOS.platform/Developer/SDKs/Library/Frameworks/CoreServices.framework/CoreService
    ./Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/Library/Frameworks/CoreServices.framework/CoreService
    ./Developer/Platforms/MacOSX.platform/Developer/SDKs/Library/Frameworks/CoreServices.framework/CoreService
    
    ```
*   样本形态：库文件（iOS、iOS模拟器、MacOSX三个平台）
    
    表2‑1 样本信息
    
    ![](http://drops.javaweb.org/uploads/images/2c630b094d2411e3b322ff738e440f623776a5bc.jpg)
    

### 2.1.2 感染方式

#### 2.1.2.1攻击机理

这次攻击本质上是通过攻击Xcode间接攻击了自动化构建和编译环境，目前开发者不论是使用XcodeServer还是基于第三方工具或自研发工具都需要基于Xcode。而这次如此大面积的国内产品中招，则只能反映大量产品研发团队在产品开发和构建环境的维护以及安全意识上都呈现出比较大的问题。

![](http://drops.javaweb.org/uploads/images/4fcd1184dbbb34ee6f905844c1ea748c3a27a78f.jpg)

图2‑2 基于Xcode的开发流程（第三方图片）【4】

**1. 恶意插件植入Xcode方式**

也许出于对Xcode稳定性和植入方便性的考虑，恶意代码实际并没有对Xcode工具进行太多修改，主要是添加了如下文件：

*   **针对 iOS**
    
    *   Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/Library/Frameworks/CoreServices.framework/CoreService
    *   Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/Library/PrivateFrameworks/IDEBundleInjection.framework
*   **针对 iOS 模拟器**
    
    *   Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/Library/Frameworks/CoreServices.framework/CoreService
    *   Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/Library/PrivateFrameworks/IDEBundleInjection.framework
*   **针对 Mac OS X**
    
    *   Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/Library/Frameworks/CoreServices.framework/CoreService
    *   Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/Library/PrivateFrameworks/IDEBundleInjection.framework

以及修改了配置文件：

*   Xcode.app/Contents/PlugIns/Xcode3Core.ideplugin/Contents/SharedSupport/Developer/Library/Xcode/Plug-ins/CoreBuildTasks.xcplugin/Contents/Resources/Ld.xcspec

**2.恶意插件植入App方式**

*   被攻击的开发环节：编译App项目部分；
*   恶意代码植入机理：通过修改Xcode配置文件，导致编译Linking时程序强制加载恶意库文件；
*   修改的配置文件：Xcode.app/Contents/PlugIns/Xcode3Core.ideplugin/Contents/SharedSupport/Developer/Library/Xcode/Plug-ins/CoreBuildTasks.xcplugin/Contents/Resources/Ld.xcspec
*   添加的语句： “-force_load$(PLATFORM_DEVELOPER_SDK_DIR)/Library/Frameworks/CoreServices.framework/CoreService”

![](http://drops.javaweb.org/uploads/images/c8e4b01255b3f6bff0cd04e83ae126e25bde44b9.jpg)

![](http://drops.javaweb.org/uploads/images/18f53695ea7e45553ce328ca716f6b5b6ad5dc3a.jpg)

图2‑3 受感染Xcode与官方版本配置文件对比

#### 2.1.2.2 恶意代码运行时间

*   恶意代码植入位置：UIWindow (didFinishLaunchingWithOptions)；
*   恶意代码启动时间：App启动后，开启准备展示第一个页面时恶意代码已经执行了；
*   iOS应用启动流程：从代码执行流程来看，图2-4中每一步都可以作为恶意代码植入点，且其框架基本都是由模板自动生成，而UIWindow为iOS App启动后展示页面时执行。

![](http://drops.javaweb.org/uploads/images/7b0fec3f49c0f140948c2ad13cd290aacbd32ac7.jpg)

图2‑4 iOS应用启动流程（第三方图片）【5】

*   UIWindow生成方式：通常通过模板建立工程时，Xcode会自动生成一个Window，然后让它变成keyWindow并显示出来；由于是模板自动生成，所以很多时候开发人员都容易忽略这个UIWindow对象，这也是此次Xcode被植入恶意代码位置的原因之一。
*   恶意代码启动时机分析：恶意代码植入于UIWindow (didFinishLaunchingWithOptions)中，其入口点为： __UIWindow_didFinishLaunchingWithOptions\__makeKeyAndVisible_；UIWindow是作为包含了其他所有View的一个容器，每一个程序里面都会有一个UIWindow；而didFinishLaunchingWithOptions里面的代码会在UIWindow启动时执行，即被感染App在启动时的开始准备展示界面就已经在执行被植入的恶意代码了。

![](http://drops.javaweb.org/uploads/images/34bd3e5b3793f0e620de651975ddac0bd0362735.jpg)

图2‑5 恶意插件植入的代码入口点

### 2.1.3 危害分析

#### 2.1.3.1 上传隐私

恶意代码上传的信息主要有：时间戳、应用名、包名、系统版本、语言、国家、网络信息等；同时还有被感染App的运行状态：launch、runing、suspend、terminate、resignActive、AlertView。

所有信息通过DES加密后POST上传到服务器：

*   init.icloud-analysis.com
*   init.crash-analytics.com
*   init.icloud-diagnostics.com

其中6.0版本URL为字符串形式，而到7.0版本URL则进化到字符拼接，7.0版本中还可以获取应用剪贴板信息。

![](http://drops.javaweb.org/uploads/images/e30af51c510e3a57b2a648b06f8e668cac18c237.jpg)

图2‑6 获取上传的信息

![](http://drops.javaweb.org/uploads/images/2b0420450b24cb2769d8cd7fa79c5f0a53370fb3.jpg)

图2‑7 6.0版本中URL为字符串形式

![](http://drops.javaweb.org/uploads/images/b2a63434e4a050c4a2a6c1ec17d7e896f57020c9.jpg)

图2‑8 7.0版本中URL字符拼接形式

![](http://drops.javaweb.org/uploads/images/9c1870ad156b9b56499a9926387d042598b5c4da.jpg)

图2‑9 7.0版本获取应用剪贴板信息

#### 2.1.3.2 任意弹窗

恶意代码可以远程设置任意应用的弹窗信息，包括弹框标题、内容、推广应用ID、取消按钮、确认按钮；需要注意的是该部分代码并没有使用输入框控件，同时也并无进一步的数据回传代码，因此是不能直接高仿伪造系统弹窗，钓鱼获取Apple ID输入和密码输入的（这点在诸多已公开分析报告中均有一定误导性）。

![](http://drops.javaweb.org/uploads/images/611c57a16b069d7ab3e4541a0267fdef197f9244.jpg)

图2‑10 远程设置弹窗信息

但是由于弹窗内容可以任意设定，攻击者完全可以使用弹窗进行欺诈通知。

#### 2.1.3.3 远控模块

**1.OpenURL远控**

恶意代码包含了一个使用OpenURL的远控模块，该模块可以用来执行从服务器获取到的URL scheme，其使用canOpenURL获取设备上定义的URL scheme信息，并从服务器获取URL scheme通过OpenURL执行。

![](http://drops.javaweb.org/uploads/images/b545c803f67958d66a77dc620dbcc757381ca2b5.jpg)

图2‑11 canOpenURL获取信息，执行从服务器获取的URLscheme

**2.URL scheme能力**

URL scheme功能强大，通过OpenURL可用实现很多功能；但需要注意的是，URL scheme所能达到的功能与目标App权限有关，如拨打电话、发送短信需要被感染应用具有相应权限。但如果其他App或系统组件有URL scheme 解析漏洞、Webview漏洞等，则能相应执行更多行为。

由于服务器已经关闭，同时该样本也没有明显证据表明具体使用了哪些URL scheme，以下为我们分析的恶意代码所能做到的行为：

*   调用App
*   拨打电话
*   发送短信
*   发送邮件
*   获取剪贴板信息
*   打开网页，如打开高仿Apple的钓鱼网站
*   结合弹窗推广应用，App Store&企业证书应用皆可

![](http://static.wooyun.org//drops/20150924/2015092405341031535pic141.jpegg)

图2‑12 设置推广应用appID和对应URL scheme

另外推广应用时候，由于弹窗所有信息皆可远程设置，当把”取消“按钮显示为”安装“，”安装“按钮显示为”取消“，极易误导用户安装推广的应用。

### 2.1.4 中间人利用

虽然恶意代码使用的域名已经被封，但由于其通信数据只是采用了DES简单加密，很容易被中间人重定向接管所有控制。当使用中间人攻击、DNS污染时，攻击者只需要将样本中服务器域名解析到自己的服务器，即可接管利用所有被感染设备，进而获取隐私、弹窗欺诈、远程控制等。

其中恶意代码使用的DES密钥生成比较巧妙，是先定义一个字符串”stringWithFormat“，再截取最大密钥长度，即前八个字符“stringWi”。

DES标准密钥长度为56位，加上8个奇偶校验位，共64bit即8个字节。

![](http://drops.javaweb.org/uploads/images/32dd2d19012437666f44de47ea0567a68e065c23.jpg)

图2‑13 DES密钥生成方式

所以即便恶意代码服务器已经失活，依然建议用户及时更新被感染App版本，若仍未更新版本的App，建议立即卸载或尽量不要在公共WiFi环境下使用，请等待新版本发布。

2.2 影响面分析
---------

截止到2015年9月22日凌晨3时，通过各安全厂商累计发现的数据显示，当前已确认共692种（按照版本号计算858个）App受到感染，其中影响较大的包括微信、高德地图、滴滴出行（打车）、58同城、豆瓣阅读、凯立德导航、平安证券、网易云音乐、优酷、天涯社区、百度音乐等应用的多个版本。（详情请参见附录三。）

**注：需要说明的是，根据相关消息，这一事件是腾讯在自查中发现并上报给CNCERT的。**

安天依托国内一份2014年度iOSTOP 200排行的信息【3】进行了App检测，共发现有6款App受到影响，在表2-1中已用红色字体突出显示。

表2‑2 App Store Top200中受影响App统计，红色为受到影响软件

![](http://drops.javaweb.org/uploads/images/ca0218231b99ee16fd7a115bc03421e08c14f360.jpg)

目前，已有其他分析团队示警著名手机游戏开发平台Unity 3D也被同一作者植入了恶意代码，与攻击Xcode手法一样，但安天分析小组没有跟进分析。

0x03 扩散、组织分析
============

3.1 传播分析
--------

攻击者使用了多个账号，在多个不同网站或论坛进行传播被植入恶意代码的Xcode，以下是我们对其传播账号及传播信息的脑图化整理：

![](http://drops.javaweb.org/uploads/images/bc996b664f01c8d6364efb12e940662dbba83665.jpg)

图3‑1 传播马甲关联分析图

被植入恶意代码的Xcode由攻击者上传至百度云网盘，然后通过国内几个知名论坛发布传播，传播的论坛包括：51CTO技术论坛、威锋网论坛、unity圣典、9ria论坛和swiftmi。我们通过跟踪发现其最早发布是在“unity圣典”，进行传播的时间为2015年3月16日，攻击者在每个论坛的ID及所发的百度云盘下载地址都不相同，详情参见表3-1。

表3‑1 传播论坛情况

![](http://drops.javaweb.org/uploads/images/0a452d3ad5fa600c4ca1532426d7dcf054f57bf2.jpg)

![](http://drops.javaweb.org/uploads/images/8a371a38ab6c9e31f3727828c0ddcbb7f200a13d.jpg)

图3‑2 作者百度云网盘

以威锋论坛传播为例，这个帖子本身是一个旧帖，首次发布于2012年，并在2015年6月15日进行最后编辑，作者用旧帖“占坑”的目的，是为了增加下载者对此帖信任度，如图3-3。

![](http://drops.javaweb.org/uploads/images/ee6325101eab41661917b313ccac95c8a51ff35a.jpg)

图3‑3 通过威锋网传播

此外，微博上有网友说Xcode传播是通过迅雷下载重定向传播，会导致输入官方下载地址下载到错误版本，但后来同一网友又澄清是自己看错了导致，并删除了原帖。有关截图参见图3-4、图3-5：

![](http://drops.javaweb.org/uploads/images/942c9fb8ad6ff8d0277d37a8905d1d1268ce0b0c.jpg)

图3‑4 证明截图1微博发贴已删

![](http://drops.javaweb.org/uploads/images/d6f3e90e450af347a69e7108d22c35c29efafc36.jpg)

图3‑5 证明截图2微博发贴已删

2015年9月19日该网友公开澄清是自己记错了，误把百度云盘链接直接拷贝到迅雷下载地址里了，以下是网友的公开声明。

![](http://drops.javaweb.org/uploads/images/53e51ea29ce03f92cece0c6d3bd60a638d59aa19.jpg)

图3‑6 网友澄清证明

但在地下社区中，确实一直存在针对迅雷污染，使迅雷下载到重定向恶意代码的方法讨论，而且由于类似感染可能获得巨大的经济利益，我们并不能完全排除这种下载污染，有可能通过流量劫持等方式来配合。同时亦有网友反馈迅雷存在直接下载和离线下载所得到的文件大小不同的情况。受时间所限，安天分析小组未对这些传言进行进一步验证。

3.2 攻击者情况猜测
-----------

此前根据相关安全团队的信息检索，认为攻击者可能是X工业大学的一名学生。从攻击者具备污染了多个平台的开发能力，并已经确定对iOS用户产生了严重影响，开发能力覆盖前台、后台，掌握社工技巧，具备SEO优化的意识。从其综合能力来看，有可能不是个体作业，而是一个小的团伙、或者存在其他方式协同的可能性。同时，根据相对可信的消息，相关云资源使用者每月向数据汇聚点亚马逊支付数千美元，其应该有较高的获益。

但此前关于攻击者的亚马逊资费每月数十万美元的猜测，我们认为有误，因为亚马逊从计费上是单向收费的，而相关猜测是双向计算的。

3.3 开发环节的安全问题分析
---------------

导致大量原厂发布的App遭到污染的重要原因是，开发团队未坚持原厂下载，也并未验证所下载的开发工具的数字签名。

我们发现有很多分析团队向开发者提供了完整的官方Xcode文件的Hash，但显然直接对应用进行数字签名验证可能是更高效的方法。

### 3.3.1 Mac&iOS app签名方式

OS X和iOS应用使用相同的签名方式，即：

1.  在程序包中新建 _CodeSignature/CodeResources 文件，存储了被签名的程序包中所有文件的摘要信息；
2.  使用私钥 Private Key 对摘要进行加密，完成代码签名。

### 3.3.2 Mac上官方签名工具codesign验证App方式

3.3.2 Mac上官方签名工具codesign验证App方式

Mac上可以使用官方codesign工具对Mac&iOS App进行签名验证。

codesign工具属于Xcode Command Line Tools套件之一，可以使用如下方式获取安装，推荐使用1、2方式：

1.  Terminal里执行：Xcode-select –install；
2.  Mac AppStore里安装；
3.  安装 brew 后执行 brew doctor 自动安装；
4.  在Developer Apple网站下载安装：[https://developer.apple.com/downloads/](https://developer.apple.com/downloads/)

![](http://drops.javaweb.org/uploads/images/6ec36ec4a17f1e2e05ecd8df80df92583b7a145a.jpg)

图3‑7 Developer Apple上的Command LineTools工具

#### 3.3.2.1 获取app签名证书信息

使用codesign -vv-d xxx.app 指令可以获取app的签名信息。

如验证官方Xcode7.0，方框内Authority信息即该应用的证书信息，其中：

*   Authority=Apple Root CA，表示发布证书的CA机构，又称为证书授权中心；
*   Authority=Apple WorldwideDeveloper Relations Certification Authority，表示证书的颁发中心，为Apple的认证部门；
*   Authority=Apple Mac OSApplication Signing，表示证书所有者，即该应用属于Mac App Store签名发布。

![](http://drops.javaweb.org/uploads/images/e5471db5a90ba825ebc186e16dabd6437961991a.jpg)

图3‑8 官方Xcode7.0签名信息

而验证含XcodeGhost恶意插件的Xcode6.4应用，其所有者为“Software Signing”，说明非Mac App Store官方渠道发布。

![](http://drops.javaweb.org/uploads/images/3fea83018bbe961a1ab73731cdbfa66f09396d0f.jpg)

图3‑9 恶意Xcode签名信息

#### 3.3.2.2 验证app的合法性

为了达到给所有文件设置签名的目的，签名的过程中会在程序包（即Example.app）中新建一个叫做 _CodeSignatue/CodeResources 的文件，这个文件中存储了被签名的程序包中所有文件的签名。

使用codesign--verify xxx.app 指令可以根据_CodeSignatue/CodeResources文件验证App的合法性

校验官方Xcode应用，验证通过，将没有任何提示。

![](http://drops.javaweb.org/uploads/images/eea07a8512b5920d367aa34dbeeb422dfda10fa0.jpg)

图3‑10 官方Xcode签名校验合法

校验含XcodeGhost恶意插件的Xcode应用，验证失败，会出现提示，说明该应用在签名之后被修改过。

![](http://drops.javaweb.org/uploads/images/700555a5905e7d8dfaa4649f48e2361e9118d7e5.jpg)

图3‑11 恶意Xcode签名校验失败

### 3.3.3 官方推荐Xcode验证工具spclt

鉴于此事件影响重大，苹果官方于2015年9月22日发布文章[6](http://drops.javaweb.org/uploads/images/7b0fec3f49c0f140948c2ad13cd290aacbd32ac7.jpg)推荐使用spclt工具校验Xcode合法性。

![](http://drops.javaweb.org/uploads/images/3cc7e2ff1eb7976020e9fa25437715b8cca5f001.jpg)

图3‑12 官方推荐使用spctl工具校验Xcode合法性

如图3-13，上面为正版Xcode验证信息，表明来源为Mac App Store；而下面为恶意Xcode工具验证信息，没有任何来源信息；

![](http://drops.javaweb.org/uploads/images/2ae9a8dfe703ca6b1872b57efb8010d4e994bdc4.jpg)

图3‑13 使用spctl工具校验Xcode来源

关于此前Xcode加载较慢的问题一直被诟病，其存在国内网络设施方面的问题，但苹果未投入足够CDN资源也是一个因素。我们注意到苹果在申明中提及会改善国内相关下载体验，但无论体验如何，原厂获取、本地可信分发建立与维护，都应该是开发者需要建立的规则。

0x04 Android风险预警
=====

4.1 预警背景
--------

在XcodeGhost事件发生后，安天分析人员尝试进行了其他平台的非官方通道开发工具审查，但由于我们的资源获取能力等因素所限，尽管检查了大量包和镜像，却并未在其他开发平台发现更多问题。但正如兄弟团队发现Unity 3D被同样污染的问题一样，这并不意味着其他开发平台不存在其他问题。

由于Android的开发环境在国内获取较为困难，使得部分开发者会选择从在线网盘等渠道下载离线更新包的方式来取代在线更新，因此我们将Android开发生产环境作为重点预警对象。

目前Android下的开发生产环境如图4-1所示，其可以分成开发流程、自动化构建和发布三部分。

![](http://drops.javaweb.org/uploads/images/e28a3873a1681d7304ed635fb7e1cba33e63d1b2.jpg)

图4‑1 Xcode安卓开发生产环境示意图

通过分析，无论开发者是否使用IDE环境进行开发或者自动化构建，都会使用到Android SDK和Android NDK，并且默认官网下载的zip中只包含有SDK Manager，不同API版本的构建工具和lib库均需要开发者在线下载，并且在在线云盘中有大量的离线包分享。

![](http://drops.javaweb.org/uploads/images/0ae212a0cded7fcb33cda3f8a194a6c78d1d59ee.jpg)

![](http://drops.javaweb.org/uploads/images/54ae8027a7ada34047860d0ec723dad1c7acc8b0.jpg)

![](http://drops.javaweb.org/uploads/images/56be8e149831ab974176a55111e127c24e40545f.jpg)

图4‑2 通过搜索引擎可以看到在百度网盘资源中有多份Android开发工具包

下文我们将分别说明JAVA代码和Native代码的开发生产环境面临的污染风险，为避免我们的分析被攻击者利用，分析小组对公开版本报告中的本部分做了大篇幅删减，相关论述是希望兄弟团队共同对污染的可能性进行排查，以降低风险。

4.2 JAVA代码开发生产环境的风险
-------------------

在Android开发中，JAVA代码开发和编译主要由Android SDK提供，其中除了编译工具和Ant编译脚本外，还提供了系统jar库，用于版本兼容的support库，以及一些官方其他支持库。

JAVA开发生产环境被污染的风险，可能存在如下情况：

1.  污染代码在编译过程中被植入，或者随apk打包的jar库植入；
2.  污染代码需要被主动调用，如在一个类被加载的时候去调用。

Android SDK中最大的风险是jar库没有普遍采用签名验证机制，存在被篡改风险。

![](http://drops.javaweb.org/uploads/images/1e4db65cd46ed7b9aa644a409454a7af3bf89e22.jpg)

图4‑3 验证部分.jar的数字签名

4.3 Native代码开发生产环境的风险
---------------------

同样，Native代码开发生产环境被污染的风险，可能存在如下情况：

1.  污染代码跟随编译过程进入构建的模块，Native代码开发生产环境存在如下污染问题：
    
    1.  污染头文件被污染；
    2.  ndk-build编译脚本被污染；
    3.  crt运行的.o库被污染。
2.  污染代码必须能够自动执行，污染代码可能被编译到.init_proc或者.init_array节；
    
3.  存在某处主动调用静态库的extern方法。
    

0x05 全景的安全视野才能减少盲点
=====

如果对这一事件进行定性，我们将其称之为一系列严重的“地下供应链”（工具链）污染事件，在当前移动互联网研发过度追求效率、安全意识低下的现状下，连锁形成了重大后果。从目前分析来看其污染源可能是地下黑产，并穿透了若干道应有的安全篱笆。同时值得深思的是，在今年3月份的时候斯诺登曝光的一份文档显示：美国情报机构曾考虑通过对Xcode(4.1)SDK进行污染，从而绕过苹果App Store的安全审查机制，最终将带毒App放到正规的苹果应用商店里。长期起来，国内安全的焦虑过多地集中围绕着CPU和操作系统等少数环节的的自主化展开，但有其他几方面问题没有得到足够关注。

**1.从供应链上看：**供应链上的各个环节，都有可能影响到最终产品和最终使用场景的安全性。在这个维度上，开发工具、固件、外设等“非核心环节”的安全风险，并不低于操作系统，而利用其攻击的难度可能更低。因此仅关注供应链的基础和核心环节是不够的。而同时，在实际应用场景中，往往存在着因盗版、汉化、破解等问题带来的地下供应链，以及下载重定向、第三方分发源等带来的不确定性，这些因素，在过去已经给信息系统制造了大量隐患。

**2.从场景和时域关联上看：**开发商、分销商、配送通道的安全性与使用场景的安全同样重要，因此只关注最终场景的安全是不够的。苹果在中国缺少足够的CDN资源投入，尽管因其产品的强势，让中国的开发者和用户都能容忍，但无疑也助动了地下工具链的成长。而国内的网络速度和效率问题，看似一个体验问题，但其最终也同样可以转化为安全问题。

**3.从权限上看：**在数据读取能力上，居于底层的操作系统和居于上层的应用程序可能是一致的，即使操作系统做出种种分层访问、沙箱隔离等限制，一旦受到污染程序进入系统，其攻击难度就瞬间从远程利用降低为提权，因此只关注操作系统的“底层”安全是不够的，追求系统数据安全和持续运维才是目的。而同时，在移动平台上，无论是安卓提供的安全产品与应用平权的策略，还是苹果彻底将反病毒产品逐出App Store的方法，最终都导致安全厂商难以给予其更多的支持、防护和协同。从而使这些互联网霸王龙陷于与寄生虫们不可能取胜的战争当中。

**4.从信息链上看：**互联网模式不是传统的信息流转，而是基于用户的方便性追求而进行的信息采集、聚合与分析，用户需要用主动提供信息来置换对应服务，因此，仅用传统的控制思维是不够的。而从这一恶意代码所表现出的信息采集和提交的特性来看，其似乎与常态的互联网客户端十分接近，这或许也是其未能被更早发现的原因。

从被现行发现的Xcode到之后被关注到的U3D，以及我们预警的安卓开发平台的可能性，一系列非官方版本污染事件涉及到了上述每个问题的层面，其正是通过工具链污染绕过了多个开发厂商的自我安全审核，与号称非常严格的苹果应用商店的上架审核（也许对苹果来说，这个“多余”的模块就像一个新增的广告联盟的插件）。而一批开发者不坚持原厂获取开发工具，不审查工具的数字签名，这些都暴露了App开发领域的野蛮生长，忽视安全的现状。而这种被污染的App到达用户终端后，并不需要依赖获取更高权限，依然可以获取大量有价值的信息，但一旦与漏洞利用结合，就有可能形成巨大的威力。而同时，其也采用了与互联网客户端类似的信息采集聚合方式，而数据的聚合点，则位于境外的云服务平台上。从而使事件变成的多边、多角的复杂关系。

对于苹果公司在此事中的表现，我们除了期待苹果在中国有更多的CDN投入来改善下载体验外，我们想引用我们的同事在2012年所写的两段文字：“作为一个选择全封闭、不兼容产业模式的商业帝国，苹果公司取得的巨大成就在于其对体验极致的追求和近乎完美的产业定位设计。但如果把第三方安全支撑能力完全摒弃在外，一个自身已经成为复杂巨系统的体系，怎么可能具备独善其身的能力呢”、“对安全厂商高度排斥，如果不求变革，这些都将导致其陷入漫长的一个人的战斗。”

这一问题再度提醒我们，要跳出单点迷思，从完整供应链、信息链角度，形成全景的安全视野、安全建模与评价、感知能力。同时，我们也要再度提醒，我们需要警惕“建立一个完全自闭合的供应链与自循环的信息链方能安全自保”的小农安全观的卷土重来，在互联网和社会生活场景下，这本身就不可能实现。而中国政企网络的市场空间，完全不足以保证一个健康的闭环供应链的有效生存，更谈不上还需要支撑这个闭环供应链安全性的持续改进。

同时，我们也需要看到，网络地下黑产的泛滥与无孔不入，其不仅将危害网民的安全，也会带来更多纵深的安全风险，其让国家安全的防护边界变得模糊。因此在这一问题上投入更多的资源，进行更为强力、有效的综合治理，于国于民都非常必要。

[博客地址](http://mp.weixin.qq.com/s?__biz=MjM5MTA3Nzk4MQ==&mid=211284160&idx=1&sn=a2de7a525b668f78b55dc318d3e7cf2a&scene=1&srcid=0924TlNwItkd9zHm2wEPKzS9&key=2877d24f51fa53844e66b561382922075e89fbbaf58738438dbe31082dac08aaaebfdfdca955daad4525bbff608f3e78&ascene=0&uin=Mjg2OTM2MjIwMw==&devicetype=iMac%20MacBookPro11,1%20OSX%20OSX%2010.10%20build%2814A389%29&version=11020012&pass_ticket=tQfXLJpYeYEEZgekCHBViFVSXHz%2bil8zfyoNXRmNdwsDgtADGS95Pgf7yV5uUTPq)