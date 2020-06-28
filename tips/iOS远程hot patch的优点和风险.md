# iOS远程hot patch的优点和风险

**原文地址：**  
[https://www.fireeye.com/blog/threat-research/2016/01/hot_or_not_the_bene.html](https://www.fireeye.com/blog/threat-research/2016/01/hot_or_not_the_bene.html)

0x00 前言
=====

苹果做了非常多的努力来建造和维持一个健康并且干净的应用环境。其中对现在的现状起到很大作用的部分就是苹果APP STORE，它是被一个十分周密的对所有提交的应用进行检查的审批程序所保护的。尽管这个程序是被设计为用来保护IOS用户并且确保应用程序符合苹果的安全性和完整性的要求，体验过这个流程的开发者可能会觉得它太复杂了并且花费了大量的时间。发布一个新的release版本或者发布一个已经存在的APP的补丁也要遵守这个流程，这对于一个想要给一个影响现有APP用户的重要bug或者安全漏洞打补丁的开发者来说就会非常困扰。

开发者社区已经在寻找相应的替代方案，并且取得了一些进展。这一系列的解决方案现在提供更有效率的IOS APP开发体验，让APP开发者去更新他们的代码只要他们认为是合适的，并且立即把补丁部署到用户的设备上去。尽管这些技术提供更自由的开发体验，它们并不符合苹果试图去维持的相同的安全规范。更糟糕的是，这些理念可能会成为苹果APP STORE坚固“城墙”的阿喀琉斯之踵。

在这篇文章中，FireEye手机安全研究员将会分析采用这些可替代方案的IOS APP的安全风险，并且试图阻止IOS APP生态环境做出意外的安全妥协。

在这篇文章最开始的部分，我们会关注一个开源的解决方案：JSPatch。

0x01 JSPatch
=====

JsPatch是一个开源项目——它基于苹果公司的JavaScriptCore框架——目的是为苹果的费力以及不可预测的审批程序提供一种可替换的方案，当及时发布重要bug的补丁是非常重要的时候。用作者自己的话说（**加粗**的部分作强调）：

JSPatch使用Objective-C的运行环境来联系Objective-C和JavaScript。你只需要包含一个小的引擎，就可以在JS中调用**任何**Objective-C的类和方法。这就让这个APP获得了脚本语言的能力：**动态地**添加模块或者替换Objective—C代码来修复bug。

JSpatch的作者，在个人博客中提供了一个样例，来说明JSPatch是如何被用来更新一个有缺陷的APP：

下图展示了一个UITableViewController的实现，类名为JPTableViewController,通过`tableView:didSelectRowAtIndexPath:`来提供数据。在第5行，它会从后端源的字符串数组中检索数据，这个字符串数组带有一个映射到选定行数字的目录。在很多情况下，这个功能是没有问题的；但是当行目录超过了源数据数组的范围时，这也是很有可能发生的，这个程序就会抛出一个异常并且最终导致APP crash掉。而APP的崩溃对用户来说是绝对不希望发生的。

![p1](http://drops.javaweb.org/uploads/images/1462e61ef5da9f7b2bb273a4ba1e99cc254e416e.jpg)

在苹果提供的技术范围内，修复这个状况的方法就是使用更新的代码来修复bug并且重建这个应用，并且把最新创建的APP提交给App Store作申请。尽管审批流程对于更新的应用相比最开始提交的审查来说往往花费更少的时间，这个流程仍然是花费大量时间的，不可预期的，并会导致一些亏损，如果APP的修复没有能以一种及时并且可控的模式进行发布。

然而，如果原始的应用嵌入了JSPatch引擎，它的行为就可以按照在运行时被加载的JavaScript代码来改变。这里的JS文件(hxxp://cnbang.net/bugfix.JS in the above example)是被APP开发者远程控制的。它通过网络通信被发送到APP。

下图展示了JSPatch在一个IOS应用中被创建的基本方法。这些代码将会允许下载一个JavaScript补丁并在APP刚启动的时候执行。

![p2](http://drops.javaweb.org/uploads/images/9f4efdcab507a33ae21ada6567cf80748a17e82b.jpg)

JSPatch确实是非常轻量级的。在这个例子中，确保它能执行做的额外的工作就只是在`application:didFiishLaunchingWithOptions: selector`中添加了7行代码。下图展示的是从hxxp://cnbang.net/bugfix.JS下载的用于给有缺陷的代码打补丁的JS代码。

![p3](http://drops.javaweb.org/uploads/images/906b5aebf0aca9da0a2dced7d1868a3bc7d6a640.jpg)

0x02 恶意功能展示
=====

JSPatch对IOS开发者来说是一份不错的福利。从好的一方面来说，它可以被用来迅速并有效地部署补丁和代码更新。但是在我们这样一个非乌托邦的世界里，总会有人去利用这种技术来实现预料之外的目的。特别是，如果一个攻击者能够篡改最终被APP加载的JavaScript的文件，就可以靠一个苹果APP STORE的应用来实施大规模的攻击。

**目标应用：**

我们随机挑选了一个合法的APP，它使用JSPatch并且可以从APP STORE下载到。如图所示，建立JSPatch平台的逻辑以及补丁的源代码被打包在`[AppDelegate excuteJSPatch:]`这个流程中。

![p4](http://drops.javaweb.org/uploads/images/0bf84b4ec92abc681e4519f664f0fd8abeca0ac8.jpg)

这里有一系列的流程，从应用程序入口点（在这个例子中就是AppDelegate类）到JavaScript文件包含更新或者补丁代码被写到文件系统的地方。这个流程包含了与远程服务器通信获得补丁代码。在我们的测试设备上，我们最终发现JavaScript的补丁代码是被hash处理的并且存储在如下图所示的位置。hash过的内容也如下图所示，它使用base64格式加密的。

![p5](http://drops.javaweb.org/uploads/images/2e57b67d3519a74b6afd3448cf3b882ff92f98cd.jpg)（下载的JavaScript文件在目标机器上存储的位置）

![p6](http://drops.javaweb.org/uploads/images/af7cfb78d070be20fe664875bed0e0a58aad18f9.jpg)（加密的补丁内容）

尽管这个目标应用的开发者已经采取了一些措施来确保这些隐私数据不被非法窃取，比如在对称加密的基础上使用Base64编码，攻击者可以通过在[Cycript](http://www.cycript.org/)上运行几条指令来使这些安全措施无效。加密的补丁代码如下图所示：

![p7](http://drops.javaweb.org/uploads/images/fb022972c83e66002bf222ef8126f8bb3d82d533.jpg)

这就是被JPEngine加载和运行的内容，JPEngine是由JSPatch框架提供并嵌入到目标应用中的。为了改变正在运行的APP的行为，我们就要去修改一些JavaScript的内容。在下面我们展示了对抗苹果审查的恶意行为的几种可能。尽管下面的样例是来自于一台越狱过的设备，我们同样也会证实这些行为也可以在非越狱设备上完成。

**Example 1**:加载任意的**public**framework到应用进程内

**a.**public framework样例：`/System/Library/Frameworks/Accounts.framework`

**b.**一些被public framework使用的private API：`[ACAccountStore init], [ACAccountStore allAccountTypes]`

上面讨论的目标应用，当被运行的时候，就回家再如下图所示的frameworks进入到进程内存中去：

![p8](http://drops.javaweb.org/uploads/images/1b64362ca08162642ee0422bdb10c5a9d2fabe41.jpg)

注意上面的清单——从苹果允许的IOS应用的二进制文件生成的——并没有包含Accounts.framework。因此，任何依赖这个框架提供的这些API进行的“危险”或者“有风险”的操作都是不希望发生的。然而，下图所示的JavaScript代码让这种假设没有任何意义。

![p9](http://drops.javaweb.org/uploads/images/d2cfc10af8139d021cb4d1209fd1321d7d621367.jpg)

如果这些JavaScript代码作为hot patch被发送到目标应用，它就会动态加载一个public framework（Accounts.framework）到运行的进程中去。一旦这个框架被加载，这个脚本就有权限接触这个框架所有的API。下图展示了执行private API（ACAccountStore allAccountTypes）的输出，它输出了36个账户类型在测试设备上。添加的行为不需要要求这个应用进行重建，也不需要在APP store进行额外的审查。

![p10](http://drops.javaweb.org/uploads/images/78f5468ddcfa25649c7c5ddb6cce141068668888.jpg)

上面的证明强调了对于IOS APP用户和APP开发者来说可能存在的严重的安全风险。JSPatch技术可能会允许个人有效的绕过APP store的审查流程，并且在设备上执行任意的有威力的行为并且不需要用户的同意。代码动态的本质也让在一次恶意的行为中抓住恶意的攻击者变得十分困难。我们并不会在这个博文中提供任何有意义的EXP，只是指出这个可能性来避免攻击者对这个漏洞进行利用。

**Example 2**：将任意private的framework加载到应用进程中去

**a.**framework样例：

`/System/Library/PrivateFrameworks/BluetoothManager.framework`

**b.**样例的framework使用的private API：

`[BluetoothManager connectedDevices], [BluetoothDevice name]`

和上一个例子很像，一个恶意的JSPatch JavaScript脚本可以让一个应用加载任意的private framework，比如BluetoothManager.framework，并且会进一步调用private API来改变设备的状态。IOS private framework是被设计为只能被苹果提供的应用所使用。尽管关于private framework的使用并没有相关官方的文件，但是关于它们众所周知的是它们中的大部分可以提供私有的接近底层系统功能的权利，这也可能会让一个应用绕过操作系统设置的安全控制。APP STORE有严格的策略禁止第三方APP使用任何private framework。然而，很有必要指出的是操作系统并没有区分苹果应用的private framework使用和第三方应用的private framework使用。禁止第三方使用仅仅只是苹果APP STORE自己的策略。

当我们使用JSPatch，这些约束就会变得毫无疑义，因为JavaScript文件并不属于苹果APP store的审查范围。下图展示的代码是通过加载BluetoothManager.framework并利用API来读和改变主机设备的蓝牙状态。然后后一张图展示的是相匹配的控制台的输出。

![p11](http://drops.javaweb.org/uploads/images/2c72e1b6189f51787fdfaf180502f141a7431a89.jpg)

![p12](http://drops.javaweb.org/uploads/images/86d51088196931f216404d50cdfd20211341646d.jpg)

**Example 3**：通过private API改变系统属性

**a.**样例依赖的framework：

`/System/Library/Frameworks/CoreTelephony.framework`

**b.**样例framework使用的private API：

`[CTTelephonyNetworkInfo updateRadioAccessTechnology:]`

考虑一个使用public framework（CoreTelephony.framework）创建的目标应用的情况。按照苹果的文件说明，这个framework允许获得一个用户的家庭移动电话服务提供者的信息。它暴露了几个public API给开发者来获得这些，但是`[CTTelephonyNetworkInfo updateRadioAccessTechnology:]`并不是其中的一个。然而，如下图所示，我们可以成功的在不经过苹果同意的情况下使用这个private API来更新目标设备的移动电话服务状态。

![p13](http://drops.javaweb.org/uploads/images/6db2883b20f3cddc8477a14e2bedff9149393987.jpg)

![p14](http://drops.javaweb.org/uploads/images/6049e7e87be225174d9c9f8f789e3926b7556814.jpg)

**Example 4**：通过public API获取相册隐私数据

**a.**样本加载的framework：

`/System/Library/Frameworks/Photos.framework`

**b.**public apis:

`[PHAsset fetchAssetsWithMediaType:options:]`

对于手机用户来说关注的问题主要是隐私侵犯的问题。任何在一个设备上执行的涉及到接触和使用用户隐私数据的行为（包括联系人，信息，照片，音频，记事本，通话记录等等）都应该被证明是在应用提供的服务的范围之内。然而，如下图所示，我们可以利用private API接触到用户的相册，通过从内置的Photo.framework来采集照片的元数据。如果再多一点代码，攻击者就可以在用户不知情的情况下把照片数据导出。

![p15](http://drops.javaweb.org/uploads/images/0e06d287604d50a40b26488ecd85c78e7979b828.jpg)

![p16](http://drops.javaweb.org/uploads/images/de8f16c7cd9f86f680c006e7a43ae8059c71b83f.jpg)

**Example 5**:实时获得剪贴板的数据

**a.**framework样例：`/System/Library/Frameworks/UIKit.framework`

**b.**API：`[UIPasteboard strings], [UIPasteboard items], [UIPasteboard string]`

IOS的剪贴板是允许应用之间数据传递的机制之一。一些安全研究者已经提出一些关于安全方面的担忧，因为剪贴板可以被用来传递隐私数据，比如账户和证书。下图展示了一个简单的JavaScript的demo函数，当在JSPatch framework上运行时，就会从剪贴板获取所有的字符串内容并显示到控制台上去。

![p17](http://drops.javaweb.org/uploads/images/1b2f1a0072350f2b841ad216f34559590556045d.jpg)

![p18](http://drops.javaweb.org/uploads/images/34417a6d278772bc579f08e630ff3552cc6cb392.jpg)

我们已经展示了5个利用JSPatch作为攻击途径的例子，并且会有更多的攻击的可能性。

0x03 未来的攻击方法
=====

IOS大部分的本地的功能是依赖于C函数的（比如dlopen(),UIGetImageScreen())。由于C函数不能被反射调用，JSPatch不支持Objective C到JavaScript的直接映射。为了在JavaScript中使用C的函数，一个应用就必须支持JSExtension，JSExtension打包了C函数到相关接口并进一步导出到JavaScript中去。

依赖额外的Objective C代码来使用C函数功能让恶意的攻击者受到了约束（比如后台屏幕截取，在不被发现的情况下拦截文本消息，窃取照片隐私数据，或者后台录音）。但是这些约束可以被很简单的绕过。事实上，JSPatch的作者可以在未来提供给应用开发者更可用和方便的接口，确保有足够的命令。在这个情况下，以上的所有操作都不会被苹果公司所感受到。

0x04 安全影响
=====

众所周知IOS设备是比运行其他操作系统的手机更安全的；然而，我们必须认识到维持这个现状的因素是多方面的。苹果安全控制的核心是为IOS用户和开发者提供和维持一个安全的生态环境，一个“带有围墙的花园”——APP STORE。通过APP STORE分发的应用在一次有意义的攻击过程中将会是更难被利用的。时至今日，两个主要的攻击途径组成了所有之前披露过的对IOS平台发起的攻击：

1.  由于禁止了签名检查函数，越狱的IOS设备允许未签名的或者错误签名的应用进行安装。在一些情况下，沙箱的约束被移除，这也就允许应用在沙箱外运行。
2.  在非越狱设备上，应用程序可以借助企业证书进行side loading。FireEye发布了一系列关于利用这个攻击面的详细的攻击的报告，并且最近的报告都在持续关注这个已知的攻击途径。

并且正如我们在这篇报告中强调的，JSPatch提供了一个不需要side loading或者一台越狱的设备作为攻击必须要素的攻击途径。不难断定JSPatch里的JavaScript的内容可能将会是这个应用开发框架的阿喀琉斯之踵。因为几乎没有什么安全措施来确保这个文件的安全属性，以下的攻击场景是可能会发生的：

● 前提条件： 1）应用内嵌入了JSPatch平台；2）应用开发者有恶意的企图。

○ 影响： 应用开发者可以利用加载的framework提供的所有的private API来实施不想被苹果公司或者用户知道的行为。因为开发者拥有对JavaScript代码的控制，所以恶意的行为可能会是暂时的、动态地、秘密的和具有逃避性的。这样的攻击一旦发生，就会给所有牵涉到的利益相关者带来巨大的风险。

○ 下图阐述了一个这种类型的攻击场景：

![p19](http://drops.javaweb.org/uploads/images/663d1cf0ee47008b4293f09aaa21a5e40b60e714.jpg)

● 前提条件： 1）第三方广告SDK嵌入了JSPatch平台；2）主机应用程序使用了这个广告SDK；3）广告SDK对主机应用程序有恶意的企图。

○ 影响：1）广告SDK可能会从应用沙盒泄露数据；2）广告SDK可能会改变主机应用程序的行为；3）广告SDK可以以主机应用程序的名义对操作系统进行各种操作。

○ 下图阐述了一个这种类型的攻击场景：

![p20](http://drops.javaweb.org/uploads/images/569d5b1980bfb07002c88a18b930623edb8fb6aa.jpg)

FireEye在2015年的关于[iBackdoor](https://www.fireeye.com/blog/threat-research/2015/11/ibackdoor_high-risk.html)的发现就是一个IOS开发社区内的可信替换攻击的例子，并且可以作为这种被忽视的威胁的一个很好的例子。

● 前提条件：1）应用嵌入了JSPatch平台；2）应用开发者是合法的；3）应用没有保护从客户端到服务器之间JavaScript内容通信的安全；4）一个恶意的攻击者实施中间人攻击篡改JavaScript内容。

○ 影响：中间人攻击可以泄露沙盒内的应用的内容；中间人攻击可以通过private API利用主机应用作为代理,实施各种恶意行为。

○ 下图阐述了一个这种类型的攻击场景：

![p21](http://drops.javaweb.org/uploads/images/16a1c3346a47ec7b528c1efe713b44a414593173.jpg)

0x05 相关资料
=====

JSPatch起源于中国。自从2015年发布以来，已经在中国地区获得了成功。按JSPatch所说，很多流行的中文应用已经采用了这个技术。FireEye应用扫描在APP STORE中发现了1220个使用JSPatch的应用。

我们也发现了中国之外的开发者采用了这个框架。一方面，这也说明JSPatch在IOS开发环境中是一个很有用并且令人满意的技术。另一方面，这也标志着这些用户有很大的威胁可能被攻击——尤其是没有采取预防措施来确保所有涉及到的各方的安全。尽管JSPatch暴露出了一些安全风险，FireEye没有发现任何上述的应用是恶意的。