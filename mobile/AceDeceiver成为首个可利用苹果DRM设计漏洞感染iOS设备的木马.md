# AceDeceiver成为首个可利用苹果DRM设计漏洞感染iOS设备的木马

from ：[http://researchcenter.paloaltonetworks.com/2016/03/acedeceiver-first-ios-trojan-exploiting-apple-drm-design-flaws-to-infect-any-ios-device/](http://researchcenter.paloaltonetworks.com/2016/03/acedeceiver-first-ios-trojan-exploiting-apple-drm-design-flaws-to-infect-any-ios-device/)

0x00 简介
=====

近日，我们发现了一个全新系列的 iOS 恶意软件，这个恶意软件叫做“AceDeceiver”，能够成功感染任何非越狱设备。

与过去两年中某些 iOS 恶意软件利用企业证书发动攻击不同，AceDeceiver 无需企业证书即可自行安装。究其原因是，AceDeceiver 利用了 Apple DRM 机制上的设计漏洞，就算苹果将其从 App Store 内移除，AceDeceiver也可以借助全新的攻击途径进行传播。

AceDeceiver 是我们发现的首个利用了苹果 FairPlay DRM 保护机制漏洞的 iOS 恶意软件，能够在 iOS 设备上安装恶意应用，无论这些 iOS 设备是否已经越狱。这项技术也叫“FairPlay 中间人攻击”（FairPlayMan-In-The-Middle, MITM），自2013年起就被用来传播盗版 iOS 应用，但这是我们第一次发现这项技术被用来传播恶意软件。（虽然，早在在2014年，USENIX安全会议上就曾经演示过“FairPlay 中间人攻击”技术，但是，直到今天，利用这种技术还是能成功的实施攻击活动）。

Apple 允许用户通过 iTunes 客户端购买及下载 iOS 应用程序，然后通过电脑把这些应用程序安装到自己的 iOS 设备上，而 iOS 设备会要求每个应用程序提供一个权限代码以证明该应用程序是合法购买的。在“FairPlay 中间人攻击”中，攻击者首先会在 App Store 上购买一个应用程序，然后拦截并储存该权限代码。随后，他们会开发一个PC软件来模拟iTunes客户端的行为，从而欺骗 iOS 设备相信此应用程序是受害者购买的。这样，用户就可以免费安装一些收费的app，而该 PC 软件的作者则可以在用户不知情的情况下将潜在的恶意应用安装到用户的 iOS 设备上。

![p1](http://drops.javaweb.org/uploads/images/7e764e50d17eee1252e264c229e612aa86177d19.jpg)图1-FairPlay中间人攻击流程

在 2015 年七月至 2016 年二月，AceDeceiver 家族旗下三个不同的 iOS 应用程序被上传到了官方的 App Store，并显示为壁纸应用。这些应用程序使用与ZergHelper 的相似方法，通过在不同的地理位置执行不同的操作，至少七次成功地绕过了 Apple 的代码审核（包括首次上传，四轮代码更新）。在目前情况下，AceDeceiver 只向位于中国的用户表现出恶意行为，但攻击者能轻易地改变应用的行为。Apple 在收到我们的报告后，于2016 年二月下旬从 App Store 中删除了这三个应用。然而，由于“FairPlay 中间人攻击”要求这些应用程序只需在 App Store 中存在过一次，即使app下架，这种攻击方式仍然能生效。只要攻击者取得了 Apple 认证的副本，那么就不需要再使用 App Store来传播这些应用程序。

为了发动攻击，木马作者创造了一个叫“爱思助手”（Aisi Helper）的 Windows 软件来执行“FairPlay中间人攻击”……爱思助手能够为iOS设备提供支持服务，例如，系统重装，越狱，系统备份，设备管理和系统清理。但是，当iOS设备连接到安装了爱思助手的PC时，爱思助手也会悄悄的安装恶意应用。（注意：在感染时，只会在iOS设备上安装最新的一个应用，而不是三个应用都安装。）这些恶意iOS中提供了一个第三方软件商店的连接，诱使用户从中下载iOS应用和游戏，但是这个商店是由木马作者控制的。这些应用会要求用户输入自己的Apple ID和密码来解锁更多功能，我们怀疑这些凭证会在加密后，上传到AceDeceiver的C2服务器。另外，我们还发现了几个早期版本的AceDeceiver，有的最早使用了日期为2015年3月的企业证书。

在截稿时，AceDeceiver只会感染位于中国大陆的用户。但是，更严峻的问题在于，AceDeceiver的成功证明了一种相对简单的方式来感染未越狱设备。因此，我们很可能会在未来发现更多其他地区的用户也会感染这种木马，无论是由这些攻击者，还是其他模仿者来发动。从危险程度来看，这种新型的攻击技术也更具威胁性，原因如下：

1.  不需要企业证书，因为这类木马并不受到MDM解决方案的控制，并且木马执行也不再需要用户确定可信性。
2.  还未修复，即使被修复了，这种攻击方式也能够针对旧版的iOS系统。
3.  虽然，App Store中已经移除了这些应用，但是对攻击活动没有任何影响。攻击者并不需要让恶意app一直存在于App Store中，而是只要在App Store中上架一次并要求用户在PC上安装爱思客户端就够了。但是，ZergHelper和AceDeceiver已经说明了Apple的代码审核过程形同虚设，恶意app可以轻易地进入到App Store中。
4.  不需要用户手动安装恶意app，恶意app会自行安装。这就是为什么，恶意app虽然只进入了几个地区的App Store，而没有影响到攻击的成功。另外，这样还可以增加Apple和其他研究iOS漏洞的安全公司发现恶意app的难度。
5.  虽然，攻击需要用户的PC首先感染木马，但是，在此之后，iOS设备的感染是完全在后台进行的，用户根本无法察觉。唯一的疑点是，恶意app在新安装后，其图标会出现在用户的主屏上，所以有可能会引起用户的注意。

分析了AceDeceiver，我们认为FairPlay中间人攻击活动会成为一种针对未越狱iOS设备的常用攻击途径，也会因此威胁到全球的iOS用户。Palo Alto Networks已经放出了IPS签名（38914, 38915），也更新了URL过滤和威胁防护机制来帮助其客户防御AceDeceiver木马，以及FairPlay中间人攻击技术。

在接下来的部分，我们会详细地分析AceDeceiver的传播方式，攻击技术和实现方法。另外，我们还谈到了FairPlay中间人攻击是如何生效的，并讨论了Apple DRM技术中存在的安全漏洞。

0x01 威胁的出现时间
=====

2013年1月：FairPlay中间人攻击被用于传播盗版iOS app

2014年8月：在第23届USENIX安全会议上，研究人员分析了FairPlay 中间人攻击技术

2015年3月26日：AceDeceiver的iOS app使用了企业证书作为签名，并新增了密码盗取功能。这些app都是植入在Windows版的爱思助手客户端中

2015年7月10日：香港和新西兰App Store中出现了AceDeceiver的iOS版“爱思助手”

2015年7月24日：爱思助手Windows客户端通过升级，植入了与App Store中版本相同的爱思助手

2015年11月7日：美国App Store中出现了AceDeceiver的iOS app-“AS Wallpaper”

2016年1月30日：美国和英国App Store中出现了AceDeceiver的iOS app-“i4picture”

2016年2月21日：Palo Alto Networks公布了ZergHelper研究报告

2016年2月24日：Palo Alto Networks向Apple举报了AceDeceiver

2016年2月25日：App Store中移除了AceDeceiver的应用

2016年2月26日：Palo Alto Networks向Apple举报了AceDeceiver的FairPlay中间人攻击

0x02 关于AceDeceiver的背景
=====

爱思助手中的AceDeceiver木马是由中国深圳的一家公司开发的。AceDeceiver的C2服务器域名就是爱思助手的官网`www.i4[.]cn`。木马还利用了这个域名的三级URL进行下载和更新。

![p2](http://drops.javaweb.org/uploads/images/f40a7995ab5ebc5437bb454e4d1ef05614e44cca.jpg)图2-爱思助手的官网

爱思助手是一个Windows版本的客户端，能够向iOS设备提供诸如系统重装，系统备份，设备管理和系统清理等服务。当中国大陆的用户安装了iOS版的爱思助手后，无论其iOS设备是否越狱，都可以从第三方软件商店中下载应用和游戏，只是这个软件商店是在攻击者的控制下。攻击者提供的大部分iOS app都是盗版的。有意思的是，据中国数据库及商业信息服务网站-IT桔子称，爱思助手最初在2014年1月发布，当时还不具有任何恶意功能。截至，2014年12月，爱思助手的用户量超过了1500万人，每月的活动用户有660万以上。而恶意功能是在2015年才新增的。

用户通过电脑访问爱思官网时，无论使用的是什么操作系统，网站都会提示用户安装爱思助手的PC客户端。一旦安装，爱思PC客户端就会自动向连接到计算机的iOS设备安装最新的恶意iOS app。但是，如果用户是通过iPhone或iPad访问其网站，浏览器就会被重定向到移动版的官网 (m.i4[.]cn)，并推荐用户安装一个使用了企业证书签名的iOS版爱思助手。在2016年2月，我们调查发现，所有从爱思官网下载的爱思助手，无论是Winsows版本还是iOS版本中，都植入了AceDeceiver木马。

根据我们对AceDeceiver功能的了解，爱思网站的法律声明有点可疑。第七条中写到“任何由于黑客攻击、计算机病毒侵入或发作，因政府管制而造成的暂时性关闭等影响网络正常浏览的不可抗力而造成的个人资料泄露、丢失、被盗用或被窜改等，本网站均得免责。”

![p3](http://drops.javaweb.org/uploads/images/705c1e29573e78a5c0c38286dde1e50ed4e93b2c.jpg)图3-个人资料泄露、丢失、被盗用或被窜改等，其产品免责

0x03 AceDeceiver多次上架App Store
=====

我们在官方App Store中发现了3个属于AceDeceiver家族的iOS app。第一个app是在2015年7月10日发布的，第二个是2015年11月7日，第三个是在2016年1月30日。所有这几个app都伪装成了合法的壁纸应用，但是每个在App Store中都显示了不同的名称，使用了不同的应用标识符和不同的开发者账户。

![p4](http://drops.javaweb.org/uploads/images/695cc262872d3fb35b18e2b7a4975b2da8afc6d1.jpg)图4-App Store中第三个来自AceDeceiver家族的恶意app

| App Store中的显示名称 | 版本 | Bundle ID | 发布时间 | 地区 | 开发者 |
| --- | --- | --- | --- | --- | --- |
| 壁纸助手 | 6.0.x | com.aisi.aisiring | 7/10/2015 | 香港、新西兰 | fangwen huang |
| AS Wallpaper | 7.0.x | com.aswallpaper.mito | 11/7/2015 | 美国 | Yuzu He |
| i4picture | 7.1.x | com.i4.picture | 1/30/2016 | 美国、英国 | liu xiaolong |

表1-App Store中的3个AceDeceiver app

正常情况下，任何app在申请上架App Store时，Apple首先会审核其代码。如果开发者想要更新现有的iOS app，Apple会要求再次审核新版本应用的代码。目前，我们发现在App Store中的所有AceDeceiver app都经过了更新。也就是说，AceDeceiver成功7次绕过了Apple的代码审核。

![p5](http://drops.javaweb.org/uploads/images/16d84dd999e34fe5306fcd5b29fce21319ac8bbf.jpg)图5-面向非中国用户的界面

这些app在启动时，首先会访问URL`tool.verify.i4[.]cn/toolCheck.xhtml`，也就是AceDeceiver的C2服务器，然后才会显示用户界面。如果URL返回“0”，用户界面就会显示攻击者控制的第三方软件商店。但是，如果URL返回“1”或0以外的结果，用户则会看到一个壁纸应用的界面，如图5。

![p6](http://drops.javaweb.org/uploads/images/397fbe5eb0f73eb8ff7cbbfc37c91c78b28cf971.jpg)图6-iOS app访问C2来判断要显示哪个用户界面

![p7](http://drops.javaweb.org/uploads/images/97f827b86ecb7c035a62e455b70bb6f2e0b47911.jpg)图7-iOS首次联系C2

在2月份时，我们对这些app进行了分析，结果显示，如果设备的IP地址来自中国大陆，AceDeceiver的C2服务器就会返回“0”。如果这些app并不是利用此种方法来绕过Apple的代码审核，那么攻击者可能是在审核期间，把服务器设置成总返回“1”，这样不管Apple的审查人员身处何处，看到的总是一个壁纸应用的用户界面。

除了这种利用不同用户地区的技巧，AceDeceiver还做出了其他一些努力来避免恶意应用的暴露。

首先，攻击者选择了只向一个选定地区提交应用。苹果面向全球的155个不同地区提供了Apple Store服务。用户通常只会选择从当地的App Store中下载应用，并且无法安装其他地区的app。前面提到的ZergHelper作者向所有155个地区的App Store提交了app，但是AceDeceiver的作者并没有这样做。比如，在我们调查期间，第三个app-“i4picture”只在美国和英国的App Store中是可用的。由于这种分发策略，木马作者很可能猜测Apple不会审查来自中国的app。这样能够大幅降低安全研究人员发现恶意app的几率，因为任何其他位置的用户即使从App Store中下载了这个app，也不会发现任何恶意功能。

其次，除了根据IP地址显示不同的用户界面，根据我们的测试，AceDeceiver还会通过收集信息来记住一台设备。这些app会把设备的唯一ID上传给C2服务器。如果有一台设备曾经在中国以外的地区使用过，那么即使这台设备随后又更改回中国的IP地址，恶意功能也会是隐藏的。

第三，这些app也会在不同的情景下显示不同的名称。例如，6.0.4版本使用了与ZergHelper相同的技巧，但是，在App Store页面上，显示的名称是“壁纸助手(Wallpaper Helper)”；而在iOS设备上，显示的名称会变成“爱思助手(Aisi Helper)”。我们是通过检查IPA文件的iTunesMetadata.plist才发现了这些app在名称显示上的差异。在之后的版本7.1.2中，这个app在使用了简体中文的iOS设备上会显示为“爱思助手(Aisi Helper)”，而在所有其他设备上，其名称是“i4picture”。

![p8](http://drops.javaweb.org/uploads/images/cb0f7fc22a9ee48ba1589bc6d4901179931eb71e.jpg)图8-iOS和App Store页面上显示不同的名称

![p9](http://drops.javaweb.org/uploads/images/9b6dc1555511a394b4ec232d60b15299f856cc01.jpg)图9-使用不同语言显示不同名称

0x04 FairPlay中间人攻击
=====

在我们调查AceDeceiver时，最奇怪的部分是，木马的app并不在中国大陆的App Store中，但是其恶意功能针对的是中国大陆的用户。带着这一点疑问，我们发现了FairPlay中间人攻击的利用方式。通过中间人攻击，这些app不需要通过App Store就可以安装到iOS设备上。

![p10](http://drops.javaweb.org/uploads/images/6c15f5e005a9a90f14ed352d24d41955e5b97d12.jpg)图10-在iTunes中认证一台计算机

FairPlay是Apple在多年以前开发的一种数字版权管理（DRM）技术，旨在保护通过iTunes和iPods上的歌曲使用。基于DRM，Apple用户可以通过iTunes将iOS app下载到PC或Mac上，然后再通过计算机将这些app安装到他们的设备上。在安装之前，用户必须要用在购买这些app时使用的Apple ID来“认证这台计算机”。Apple严格限制，每个Apple ID只能用于认证最多5台计算机。攻击者正是滥用了这一过程。

在app安装时，认证过程中有一个很关键的步骤就是已连接的iPhone或iPad会把afsync.rq和afsync.rq.sig发送到计算机上，并且计算机上的iTunes软件需要把正确的afsync.rs和afsync.rs.sig文件作为响应，返回给iOS设备。只有响应正确的afsync.rs，iOS设备才可以安装这个app，并解密其DRM防护（真正的协议要比我们描述的复杂；为了便于说明，我们做出了一些简化。）

![p11](http://drops.javaweb.org/uploads/images/454baa570b5772bd3b3a4dd0230efb3134c174ae.jpg)图11-在app安装时，正常的认证过程

但是，这一过程中存在一些设计漏洞

首先，Apple只限制了需要使用购买iOS app时的Apple ID来验证这台计算机，而没有限制这些app可以安装到多少台iOS设备上。

第二，Apple允许不使用iOS设备上的Apple ID来购买app。

第三，用于保护app 安装器文件（IPA文件）的DRM总是一样的，与app下载到了哪台计算机，购买app时使用的Apple ID无关。总的来说，问题在于FairPlay DRM防护只关注了app本身，而没有关联Apple ID和iOS设备。在Apple设计的这种防护机制中，其安全性依靠的是计算机认证和这台计算机与相应设备之间的物理连接。

这些设计漏洞致使FairPlay中间人攻击成为了可能。在攻击时，攻击者可以认证一台计算机，然后从任意地区的App Store中购买一个iOS app，这样就能生成正确的afsync.rs响应。然后，攻击者可以把这个afsync.rs部署成C2服务器上的一个服务。现在，攻击者就可以创建PC或Mac软件了。客户端软件需要模拟iTunes连接iOS设备和安装iOS应用的功能。当客户端接收到iOS设备发来的afsync.rq时，就会把这个文件上传到C2服务器，然后从C2服务器获取正确的afsync.rs响应，接着再把这个响应发送到iOS设备。

通过在C2服务器上部署已经通过认证的计算机，并使用客户端软件作为中间代理，攻击者就可以将这个已购买的iOS app部署到无限数量的iOS设备上。

![p12](http://drops.javaweb.org/uploads/images/73ac6203a05b9e48db42449bef392d87f4e3b985.jpg)图12-FairPlay中间人攻击中的认证过程

考虑到攻击者需要逆向iTunes客户端才可以知道Apple采用的协议/算法，要想实现FairPlay中间人攻击也不简单。即使是这样，2013年1月，有媒体报道称，这种攻击技术已经被用来传播了盗版iOS应用。2014年8月，王铁磊和佐治亚理工学院的分析人员联合发表了论文《论大范围感染iOS设备的可行性》“On the Feasibility of Large-Scale Infections of iOS Devices”，并在第23届USENIX安全会议上提出了这一论点。文中详细的介绍了这种攻击方式，通过POC实验证明了这一攻击技术，并评估了这种攻击方法如何能在大范围内使用。

从2013年的报道至今，已经过去了3年，在分析AceDeceiver的过程中，我们意识到Apple仍然没有修复这一复杂问题。更为严重的是，当前的iOS设备和旧版iOS在面对相同的攻击方式时，仍然是不堪一击。

0x05 AceDeceiver如何利用了FairPlay中间人攻击
=====

我们调查了从5.38版开始到最新版（6.15）的爱思助手Windows客户端，在客户端的 “files/i4/60.ipa”路径下包含有App Store版本的AceDeceiver iOS IPA文件。根据爱思助手官网上的更新日志来看，5.38版本是在2015年7月24日发布的。而那三个有问题的iOS app就是在14天前上架了App Store。有些版本的Windows客户端中还包含有一个“files/i4/i4.ipa”文件，说明 AceDeceiver的iOS app使用了企业证书来签名。

在安装了Windows客户端后，只要用户将iOS设备连接到PC，并启动了客户端，客户端中植入的 AceDeceiver app就会利用FairPlay中间人攻击自动安装到iOS设备上。这一过程不需要用户确认。在用户界面上，会有一个进度条在显示“Installing Aisi Helper…（正在安装爱思助手）”，但是没有任何暂停或取消选项。

![p13](http://drops.javaweb.org/uploads/images/2ca852eca59717ed4f2cd7909199de012904c679.jpg)图13-爱思助手Windows客户端自动安装木马app

下面的分析是基于6.12 版本的Windows客户端。其安装包是在2016年2月24日，下载于`http://d.app6.i4[.]cn/i4tools/v6/i4tools_v6.12_Setup.exe`。这个安装包的SHA-256值是ad313d8e65e72a790332280701bc2c2d68a12efbeba1b97ce3dde62abbb81c97。

安装了Windows客户端后，i4Tools.exe文件中的函数sub_4A9530会负责实现安装和漏洞利用，并且会调用函数sub_4A8CE0来安装IPA文件，然后调用函数auth来执行FairPlay中间人攻击。

函数`sub_4A8CE0`会读取储存在\files\i4\60.ipa中的IPA文件，然后调用`am_install_app`。`am_install_app`函数是在i4m.dll文件中实现的，通过与`com.apple.mobile.installation_proxy`服务通信，从而将IPA文件安装到iOS设备上。这种方法非常经典。

![p14](http://drops.javaweb.org/uploads/images/5ae63d839558a279be0da59be7fb9a52cb11d37b.jpg)图14-将IPA文件安装到iOS设备

然后，sub_4A9530会调用auth函数，同样是在i4m.dll文件中实现。

![p15](http://drops.javaweb.org/uploads/images/85df97d3f6bd697c80164b38c3f07fd8053039f8.jpg)图15-通过C2 URL调用函数auth，实现FairPlay中间人攻击

最终是由函数auth实现了FairPlay中间人攻击。auth函数收到已连接的iOS设备发来的 /AirFair/sync/afsync.rq，通过HTTP POST将这个文件发送给C2服务器`auth3.i4[.]cn`（在函数sub_10005DB0中实现），从C2服务器接收afsync.rs，然后把这个响应发回到iOS设备的`/AirFair/sync/_afsync.rs_`。

![p16](http://drops.javaweb.org/uploads/images/69ab3c4a4f37562f700f2dd9137368f2adac200b.jpg)图16-Auth函数读取afsync.rq文件

![p17](http://drops.javaweb.org/uploads/images/81e6141526998bf99a37492002796a65e5f4acae.jpg)图17-通过libcurl将afsync.rq发送给C2服务器

![p18](http://drops.javaweb.org/uploads/images/97cb6940adfc67d3d6f0b9bc13dc19d3ed638f69.jpg)图18-发送afsync.rq文件的流量

![p19](http://drops.javaweb.org/uploads/images/e0971e387622ab62e8854a4fcc957e40f4654d55.jpg)图19-把afsync.rs复制回iOS

0x06 窃取Apple ID和密码
=====

如果用户来自中国，AceDeceiver的iOS app主要是作为一个第三方软件商店。注意，在这个商店中提供的有些app和游戏也都是通过FairPlay中间人攻击安装的。另外，这些app会强烈建议用户输入Apple ID和密码，这样用户就能够“直接从App Store中免费安装app，执行应用内购买，并登录到Game Center”。

在输入Apple ID和密码的页面上，AceDeciver提供了“绑定说明”和“免责条款”内容，并声称他们绝不会把用户的密码发送至自己的服务器，只是在解密后储存在本地。但是，事实并非如此。注意，免责声明的最后一条中写道，如果用户的Apple ID出现异常活动，爱思助手不负任何责任。

![p20](http://drops.javaweb.org/uploads/images/2962a03f1678a548076bb8d4c226752170a595eb.jpg)图20-iOS app的安装和Apple ID输入界面

![p21](http://drops.javaweb.org/uploads/images/d1868121c8bdc9b249db26f48076ff33d17fed08.jpg)图21-输入Apple ID时显示绑定说明和免责条款

事实上，我们发现所有版本的AceDeceiver都会把Apple ID和密码上传到C2服务器。用户只要输入了Apple ID和密码，这些iOS app就会调用[LoginEntity getLoginAppleId:withPassword:block:]方法来获取界面上的Apple ID和密码，然后调用[UserInfo initAppleId:WithPassword]。通过这种方法，Apple ID和密码都经过了RC4算法的加密，使用的秘钥是一个固定值。这个秘钥值本身就十分有意思：

```
i4.cn/forum.php?mod=viewthread&tid=m&fromuid=n

```

![p22](http://drops.javaweb.org/uploads/images/e95e13c69071387c26661a57216fdc7b1d4c75f6.jpg)图22-Apple ID和密码使用RC4算法加密

在加密后，Apple ID和密码会继续经过Base64编码和URL编码，然后设置成一个LoginEntity实例的appleId_RC4属性和password_RC4属性。现在，就需要调用[LoginEntity getLoginInfo]。在这种方法下，加密后的Apple ID和密码会通过URL “`http://buy.app.i4[.]cn`”发送回AceDeceiver的C2服务器。

![p23](http://drops.javaweb.org/uploads/images/d6a3d10e470e7cc436d53f906fe57f7ac7018517.jpg)图23-加密后的Apple ID和密码发送到C2服务器

在分析时，，我们输入了一个无效的Apple ID：“`example@example.com`”和密码：“`example123`”。通过网络流量，我们可以看到服务器接收到了下面这个凭证（因为服务器响应了HTTP 200）。

![p24](http://drops.javaweb.org/uploads/images/adab23472ccdcfc3a1b8022920e5bd03bddc6f7a.jpg)图24-上传Apple ID和密码的流量

0x07 降低风险
=====

于2016年2月下旬，Apple移除了App Store中的这3个AceDeceiver木马app。即使如此，爱思助手Windows客户端仍然可以利用FairPlay中间人攻击，在未越狱的iOS设备上安装这些木马app。我们再次向Apple举报了使用旧版企业证书签名的AceDeceiver app。所有已知被滥用的企业证书都已经失效了。

我们建议安装了在2015年3月后安装了爱思助手Windows 客户端或爱思助手 iOS app的用户立即删除这些软件和应用，并更改自己的 AppleID和密码。我们亦建议所有 iOS 用户启用 Apple ID 双重认证。

我们建议企业用户，检查苹果设备上是否安装了使用下面程序标识符的iOS应用：

*   aisi.aisiring
*   aswallpaper.mito
*   i4.picture

由于 AceDeceiver 也能通过企业证书来传播，我们还建议企业检查未知或异常的条款声明。

目前为止，所有已知的恶意流量都来自或发送至`i4[.]cn`下的子域名。企业也可以查看该域名的流量以识别潜在的 AceDeceiver 流量。

0x08 总结
=====

AceDeceiver反映出来，攻击者正在采用另一种途径来绕过Apple的安全措施，尤其是向未越狱设备上安装恶意app。移动端是恶意攻击者越来越重视的一片领地，安卓设备多年以来一直吸引着大众的关注，然而随着iOS设备的风靡，攻击者也开始注意iOS设备。在撰写本文时， AceDeceiver主要攻击的是中国大陆的iOS设备，但是，攻击者可以很容易的把攻击活动扩张到世界其他地区。我们已经注意到，这种攻击技术对于Apple而言可不是分分钟就可以修复的。

Palo Alto Networks已经发布了一个IPS签名（38915）来拦截AceDeceiver家族的C2流量。URL过滤和威胁防护已经将相关的URL已经被标记为恶意URL。Palo Alto Networks WildFire会将所有的爱思助手Windows客户端识别为恶意程序。

Palo Alto Networks还公布了另一个IPS签名（38914）来应对FairPlay中间人攻击。

0x09 致谢
=====

我们非常感激来自Palo Alto Networks的Jen Miller-Osborn和Chad Berndtson能够协助我们撰写这份报告。同时感谢来自Palo Alto Networks的Yi Ren, Tyler Halfpop，Yuchen Zhou和Rongbo Shao，正是因为他们的努力，我们的客户才能幸免于难。