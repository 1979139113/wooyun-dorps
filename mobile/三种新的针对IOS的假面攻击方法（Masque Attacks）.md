# 三种新的针对IOS的假面攻击方法（Masque Attacks）

国外安全公司FireEye的博客文章，觉得还不错就翻译了一下，如有不足，敬请指正。（视频部分需要翻墙）

原文链接：[https://www.fireeye.com/blog/threat-research/2015/06/three_new_masqueatt.html](https://www.fireeye.com/blog/threat-research/2015/06/three_new_masqueatt.html)

0x01 前言
=====

在最近的IOS8.4版本里，苹果修复了几个漏洞包括允许攻击者部署两种新型的假面攻击（CVE-2015-3722/3725，和CVE-2015-3725）。我们叫这两种利用方法Manifest Masque和Extension Masque,它们可以被用来摧毁apps，包括系统应用程序（比如：Apple Watch，Health，Pay等等），并且可以破坏应用的数据容器。在这篇文章中，我们也不会透露之前修补过的漏洞的细节，而是去关注那些没有被曝光的假面攻击漏洞：Plugin Masque(插件假面攻击)，它可以绕过IOS的强制权限措施并且劫持VPN通信。我们的调查也表明三分之一的IOS设备在发布8.1.3版本5个月之后仍然没有更新到8.1.3及以上版本，而这些设备仍然可能遭受假面攻击。

我们总结了五种假面攻击方法，如下表所示：

<table style="box-sizing: border-box; border-collapse: separate; border-spacing: 0px; background-color: rgb(255, 255, 255); border-top: 1px solid rgb(224, 224, 224); border-left: 1px solid rgb(224, 224, 224); font-family: Helvetica, Arial, &quot;Hiragino Sans GB&quot;, sans-serif; letter-spacing: normal; orphans: 2; text-indent: 0px; text-transform: none; widows: 2; word-spacing: 0px; -webkit-text-stroke-width: 0px; text-decoration-style: initial; text-decoration-color: initial;"><tbody style="box-sizing: border-box;"><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">名称</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">至今为止发现的危害</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">修复状态</td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">App Masque</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">* 替换一个已知的应用<span>&nbsp;</span><br style="box-sizing: border-box;">* 获取隐私数据</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">在IOS 8.1.3被修复[6]</td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">URL Masque</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">* 绕过信任提示<span>&nbsp;</span><br style="box-sizing: border-box;">* 劫持应用程序内部通信</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">在IOS 8.1.3被部分修复 [11]</td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">Manifest Masque</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">* 在无线安装的过程中破坏其他应用程序 (包括. Apple Watch, Health, Pay, 等等.)</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">在iOS 8.4被部分修复</td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">Plugin Masque</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">* 绕过信任提示<br style="box-sizing: border-box;">* 绕过VPN插件的权限<span>&nbsp;</span><br style="box-sizing: border-box;">* 替换一个存在的VPN插件<br style="box-sizing: border-box;">* 劫持设备通信<span>&nbsp;</span><br style="box-sizing: border-box;">* 阻止设备重启<span>&nbsp;</span><br style="box-sizing: border-box;">* 利用更多的内核漏洞</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">在iOS 8.1.3被修复</td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">Extension Masque</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">* 获取其他应用程序数据<span>&nbsp;</span><br style="box-sizing: border-box;">* 或者阻止另外的应用程序获取它自己的数据</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">在iOS 8.4被部分修复</td></tr></tbody></table>

**Manifest Masque**攻击利用了CVE-2015-3722/3725漏洞来破坏在IOS系统内已经存在的应用程序，当受害者使用企业配置文件通过无线来从一个网页上安装IOS内部的应用程序时。被摧毁的应用程序（攻击的目标）可以使一个常规的从官方应用商店下载的应用程序，或者是更重要的系统应用程序，比如Apple Watch，Apple Pay，App Store，Safari，配置文件等。这个漏洞影响所有的IOS7版本以及IOS8.4之前的版本。我们在2014年八月第一次通知苹果公司这个漏洞。

**Extension Masque**可以破坏应用程序数据容器的约束。一个和IOS内部的应用程序一起安装的恶意的扩展应用程序可以获得所有目标应用程序的数据容器，或者也可以组织目标应用程序获得它自己的数据容器。在六月十四号，安全研究者Luyi，Xiaofeng等揭示了几个在OS X上存在的问题，包括一个和Extension Masque相似的问题[5](https://www.fireeye.com/blog/threat-research/2015/06/three/_new/_masqueatt.html)。他们做了很棒的研究，但是却错过了在IOS上的这个漏洞。他们的报告声称：“这个安全风险**在IOS上是不存在的**”。然而，数据容器的问题并不会影响所有的IOS8版本而是IOS8.4之前的版本，并且可以被攻击者利用来偷取目标应用程序的数据容器内的所有数据。我们独自发现了这个在IOS上的漏洞，并且通知了苹果公司在报告[5](https://www.fireeye.com/blog/threat-research/2015/06/three/_new/_masqueatt.html)发布之前，苹果公司也把这个问题作为CVE-2015-3725的一部分修复了。

除了这两个已经在IOS8.4上打过补丁的漏洞，我们还发现了另外一个通过替换VPN插来实现的不可信的代码注入攻击，也就是**Plugin Masque Attack**。我们在2014年九月份向苹果公司提交了这个漏洞，然后苹果公司在IOS8.1.3给原始的假面攻击（App Masque）[6,11]打补丁的时候也修复了这个插件假面攻击漏洞。然而，这个漏洞比原始的假面攻击漏洞要严重得多。这个恶意代码可以被注入到neagent进程并且可以执行特权操作，比如在不引起用户注意的情况下监控所有VPN的通信。我们最早在2015年4月份的Jailbreak Security Summit[7](http://drops.wooyun.org/wp-content/uploads/2015/07/2.jpg)上分析了这种攻击方法。这里我们把这种攻击分类为Plugin Masque Attack。

我们会讨论技术细节并对这三种假面攻击方法做出分析。

0x02 Manifest Masque
=====

为了使用公司配置文件来通过无线分发IOS内的应用程序，必须要创建一个包含一个重定向到XML manifest文件的超链接的web页面，而这个XML manifest文件是被保存在一个http服务器上的[1](http://drops.wooyun.org/wp-content/uploads/2015/07/1.jpg)。这个XML manifest文件也包含了这个内部应用程序的元数据，包括它绑定的标识符，绑定的版本和.ipa文件的下载链接，如下所示。当通过无线安装这个内部的IOS应用程序的时候，IOS会首先下载manifest文件，为安装进程解析元数据。

```
<a href="itms-services://?action=downloadmanifest&url=https://example.com/manifest. plist">Install App</a>


<plist>

  <array>

  <dict>

 ...

 <key>url</key>

 <string>https://XXXXX.com/another_browser.ipa</string>

...

 <key>bundle-identifier</key>

 <string>com.google.chrome.ios</string>

 …

 <key>bundle-version</key>

 <string>1000.0</string>

   </dict>

   <dict>

  … Entries For Another App

   </dict>

   <array>

</plist>

```

根据苹果的官方文档[1](http://drops.wooyun.org/wp-content/uploads/2015/07/1.jpg)，绑定的标识符的范围应该是“你的应用程序的绑定标识符，确切的说是像你的Xcode工程中定义的那样”。然而，我们发现IOS并没有验证在网页XML manifest文件中的绑定的标识符和在应用程序内部的绑定标识符的一致性。如果网页上的XML manifest文件有一个和另一个设备上的真正的应用程序相同的绑定标识符，并且在manifest文件中绑定的版本号高于这个真正的应用程序的版本，原始的应用程序会被卸载到一个虚拟的占位符，然而内部的应用程序回继续使用它内置的绑定id来进行安装。这个虚拟的占位符会在受害者重启设备后消失。并且，如上面的代码所示，一个manifest文件可以包含不同应用程序的元数据入口来在同一时间分发多个应用程序，这也意味着这个漏洞可以导致多个应用程序可以被受害者的一次点击所卸载。

通过利用这个漏洞，一个应用程序开发者可以在安装他自己的应用程序的同时卸载掉其他的应用程序（比如竞争对手的应用程序）。用这种方法，攻击者可以在IOS中实施Dos攻击或者钓鱼攻击。

![enter image description here](http://drops.javaweb.org/uploads/images/7f7bf4673974771b7aeb76cb4b1c9e7acc3ae7b5.jpg)

Figure 1.通过安装“恶意chrome”并卸载掉原始的chrome来进行钓鱼攻击

Figure 1展示了一个钓鱼攻击的例子。当用户点击一个在Gmail应用程序内的URL时，这个URL被用“googlechrome-x-callback://”策略重写，然后会被设备上的Chrome所处理。然而，攻击者可以利用Manifest Masque漏洞来卸载原始的chrome并且安装记录了相同策略的恶意的chrome。和原始的假面攻击不同，原始攻击需要相同的绑定标识符来替换一个原始的应用程序，而这个钓鱼攻击中的恶意的chrome程序使用一个不同的绑定标识符来bypass安装者的绑定标识符验证。然后，当受害者点击Gmail内的URL时，恶意的Chrome程序可以接管重写URL策略并且实施更多复杂的攻击。

更糟糕的是，一个攻击者可以利用这个漏洞来摧毁所有的系统应用（Apple Watch, Apple Pay UIService, App Store, Safari, Health, InCallService, Settings等）。一旦被摧毁，这些系统应用程序将再也不能被受害者使用，既是受害者重启设备。

这里我们在IOS8.3版本来演示一下DoS攻击摧毁所有的系统应用程序以及一个App应用商店应用程序（Gmail）当受害者只是通过无线点击一次来安装一个系统内应用。

**Demo**: iOS Manifest Masque Attack - YouTube**Links**:[https://www.youtube.com/embed/tR9U16krpXs?rel=0](https://www.youtube.com/embed/tR9U16krpXs?rel=0)

0x03 Extension Masque
=====

苹果公司在IOS8系统引入了扩展应用程序特征[2](https://www.fireeye.com/blog/threat-research/2015/06/three/_new/_masqueatt.html)。不同类型的扩展应用程序为开发者提供了各种各样的新方法来扩展IOS8系统上应用的功能。比如，应用程序可以作为窗口小部件出现在今天的屏幕上，可以给Action表增加新的按钮，为IOS的照片应用提供照片滤镜，或者展示一种新的系统风格的键盘[3](http://drops.wooyun.org/wp-content/uploads/2015/07/1.jpg)。另外，iPhone上的手表扩展程序[4](http://drops.javaweb.org/uploads/images/7f7bf4673974771b7aeb76cb4b1c9e7acc3ae7b5.jpg)代表了IOS8.2/8.3上所有手表类应用的逻辑。一个扩展应用程序可以执行代码，并且被限制为只能接近他自己的数据容器的数据。扩展程序是作为IOS应用程序的一部分被分发的，这可以被攻击者利用作为一种潜在的攻击手段。

我们独立地发现了IOS系统应用程序内部的扩展不仅可以获得全部的权限来接触其他的应用的数据容器，还可以阻止其他的应用程序获得他们自己的数据容器。，只要扩展程序和目标应用程序使用了相同的绑定标识符。一个攻击者可以诱惑一个受害者来安装一个内部的应用程序通过使用页面上的公司配置文件，也可以确保恶意扩展程序在受害者的设备上。

这类攻击的影响是和安装有害扩展程序和目标应用程序的顺序有关的。注意一个扩展程序不可以被单独安装，它必须作为应用程序的一部分来被分发。所以在下面的内容中，当我们安装一个扩展程序的时候，就意味着我们正在安装一个带有这个扩展程序的应用。

*   如果恶意的扩展程序在目标应用程序前安装，恶意程序可以破坏数据容器并且可以在不引起使用者注意的情况下获得接近目标应用的数据容器的权限，这个目标应用也会正常的工作。
*   如果恶意的扩展程序是在目标应用之后被安装的，目标应用就再也不能接触到它自己的数据容器。因此目标应用的功能性会被严重的干扰，最严重的时候会崩溃（导致了拒绝服务攻击）。在这个环境中，如果受害者尝试去重新安装目标应用，目标应用就会恢复。但是这就会再次变成上面的那种情况。

这里是一个破坏数据容器的攻击的demo。在这个demo中，一个恶意的扩展程序可以获得所有的Gmail的数据容器内的所有数据，并且上传到攻击者服务器。

**Demo**: iOS Extension Masque Attack - YouTube**Links**:[https://www.youtube.com/embed/rmIp2-k-TCU?rel=0](https://www.youtube.com/embed/rmIp2-k-TCU?rel=0)

0x04 Plugin Masque
------------------

* * *

和IOS扩展程序不同，VPN插件是另外的一种在.ipa文件中绑定的类型。和不需要任何权限就可以嵌入到任何IOS8的应用程序中的扩展程序相比，VPN应用和VPN插件需要被分配“com.apple.networking.vpn.configuration”权限来提供系统内的VPN服务。迄今为止只有极少数的IOS开发者可以在IOS系统上发布这样的VPN客户端（比如Cisco Anyconnect, Junos Pulse, OpenVPN等）。在安装之后，VPN插件就通过一个特权系统进程（neagent[8](http://drops.wooyun.org/wp-content/uploads/2015/07/3.jpg)）被加载，不使用任何的用户接口。

Figure 2展示了Junos Pulse的.ipa文件的目录结构，VPN插件（SSLVPNJuniper.vpnplugin）定位到了这个Payload目录，和应用（Junos Pulse.app）一起为用户提供验证VPN的接口。

![enter image description here](http://drops.javaweb.org/uploads/images/5d3d891c6a4389a3752be2223897e8601f0a24e0.jpg)

Figure 2 Junos Pulse应用的目录结构

我们发现如果一个内部的应用程序嵌入了一个恶意的VPN插件，并且这个恶意插件是和受害者的IOS系统上的正规的VPN插件拥有相同的绑定ID，这个恶意的VPN插件就可以被成功的安装并且将正规的VPN插件替换掉，不需要任何特殊的权限（比如“com.apple.networking.vpn.configuration”）。然后，当受害者启动正常的VPN程序来获得VPN服务的时候，恶意的VPN插件中的不可信代码就会被neagent进程加载并且进行权限操作，比如劫持/监控VPN的通信。注入代码到neagent进程中来逃避沙箱也是在盘古8[8,12]越狱工具中使用的一个关键的利用手段。通过利用VPN插件的漏洞，就可以直接对任何8.1.2版本及以前的设备进行越狱通过无线使用其他的内核EXP。这个漏洞是和CVE-2014-4493相关的，在IOS8.1.3版本被修复，我们首先发现并分析了这个攻击在2015年四月的Jailbreak Security Summit[7](http://drops.wooyun.org/wp-content/uploads/2015/07/2.jpg)上。

这里是一个不可信代码执行攻击的demo。在这个demo中，包含了一个恶意的VPN插件内部的应用程序被安装到了受害者的设备上。在用户使用原始的Junos Pulse应用来验证VPN之后，恶意VPN插件的POC代码就被neagent进程加载和执行了。

```
while(1) {

syslog(LOG_ERR, "[+] ========= ****** PoC DYLIB LOADED ****** ==========");

sleep(3);

}

```

POC code of the malicious VPN Plugin

注意成功实施这次攻击并不需要用户来点击/信任这个内部的应用。即使用户强制卸载了这个被攻击的VPN应用，这个应用也仍然会在重启之后重新安装。这就意味着用户不能轻易的卸载掉这个应用。即使用户尝试去长按关机键来关掉这个手机，运行攻击者代码的neagent进程也会持续在后台运行并且阻止设备进行真正的重启。手机的屏幕会是完全黑色的，并且看起来像在重启。然而，带有攻击者代码的neagent进程可以持续在后台运行。

**Demo**: Plugin Masque Attack - youtube

**Links**:[https://www.youtube.com/embed/alfkkru-RDk?rel=0](https://www.youtube.com/embed/alfkkru-RDk?rel=0)

0x05 IOS更新在企业网络中的空白
=====

IOS8.1.3版本的发布（这个版本App Masque, URL Masque, and Plugin Masque的问题都被修复或者部分修复了）是在2015年的1月份，并且IOS众所周知会快速的采用新版本。然而，我们最近的针对几个比较著名的网络的IOS web通信的监控显示了令人惊讶的结果。如Figure 3所示，我们监控的将近三分之一的IOS通信仍然是在低于8.1.3版本的设备上，即使已经发布这个版本5个月了。在数据通信之后的这些设备对于所有类型的假面攻击仍然是脆弱的，包括App Masque, URL Masque, 和Plugin Masque。我们呼吁每个人，特别是企业用户，要对IOS进行彻底的更新。

![enter image description here](http://drops.javaweb.org/uploads/images/3a376465693e91d33b93e87b35fe5a16214b3eb4.jpg)

Figure 3 根据FireEye监控的网络通信的IOS版本比例

0x06 结论
=====

总结一下，尽管苹果公司已经在IOS8.1.3版本修复或者部分修复了原始的假面攻击漏洞[6,11]，仍然有其他类型的攻击利用IOS安装进程中的漏洞。我们在这篇文章中纰漏了三种假面攻击的变形来帮助用户认识到风险，并且更好的保护他们自己。并且，我们建议所有的IOS用户保持对自己的系统版本进行更新。

Reference:  
[1](https://www.fireeye.com/blog/threat-research/2015/06/three_new_masqueatt.html)https://manuals.info.apple.com/MANUALS/1000/MA1685/en_US/ios_deployment_reference.pdf

[2](https://www.fireeye.com/blog/threat-research/2015/06/three/_new/_masqueatt.html)https://developer.apple.com/app-extensions/

[3](http://drops.wooyun.org/wp-content/uploads/2015/07/1.jpg)https://developer.apple.com/library/prerelease/ios/documentation/General/Conceptual/ExtensibilityPG/NotificationCenter.html

[4](http://drops.wooyun.org/wp-content/uploads/2015/07/2.jpg)https://developer.apple.com/library/ios/documentation/General/Conceptual/WatchKitProgrammingGuide/DesigningaWatchKitApp.html

[5](http://drops.wooyun.org/wp-content/uploads/2015/07/3.jpg)https://drive.google.com/file/d/0BxxXk1d3yyuZOFlsdkNMSGswSGs/view

[6](http://drops.javaweb.org/uploads/images/3a376465693e91d33b93e87b35fe5a16214b3eb4.jpg)https://www.fireeye.com/blog/threat-research/2014/11/masque-attack-all-your-ios-apps-belong-to-us.html

[7](http://drops.wooyun.org/wp-content/uploads/2015/07/2.jpg)http://thecyberwire.com/events/docs/nsmail.pdf

[8](http://drops.wooyun.org/wp-content/uploads/2015/07/3.jpg)https://cansecwest.com/slides/2015/CanSecWest2015_Final.pdf

[9] https://itunes.apple.com/us/app/cisco-anyconnect/id392790924?mt=8

[10] https://itunes.apple.com/us/app/junos-pulse/id381348546?mt=8

[11] https://www.fireeye.com/blog/threat-research/2015/02/ios_masque_attackre.html

[12] http://en.pangu.io/