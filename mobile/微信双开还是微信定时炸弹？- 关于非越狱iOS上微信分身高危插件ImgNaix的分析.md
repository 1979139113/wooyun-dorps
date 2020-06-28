# 微信双开还是微信定时炸弹？- 关于非越狱iOS上微信分身高危插件ImgNaix的分析

蒸米@阿里移动安全

0x00 序
=====

微信作为手机上的第一大应用，有着上亿的用户。并且很多人都不只拥有一个微信帐号，有的微信账号是用于商业的，有的是用于私人的。可惜的是官方版的微信并不支持多开的功能，并且频繁更换微信账号也是一件非常麻烦的事，于是大家纷纷在寻找能够在手机上登陆多个微信账号的方法，相对于iOS，Android上早就有了很成熟的产品，比如360 OS的微信双开和LBE的双开大师就可以满足很多用户多开的需求。但是在iOS上，因为苹果的安全机制，并没有任何知名的IT厂商推出微信多开的产品，反而是各种小公司的微信双开产品满天飞。但使用这些产品真的安全吗？今天我们就来看看这些产品的真面目。

0x01 ”倍推微信分身”初探
=====

这次要分析的产品名字叫”倍推微信分身”，可以实现非越狱iOS上的微信多开。这个app的安装是通过itms-services，也就是企业证书的安装模式进行安装的。服务器是架在59os.com。可以看到除了微信分身以外，还有很多别的破解应用提供下载：

![null](http://drops.javaweb.org/uploads/images/e22f6c1d2b582d128ed4b09a505b8ba413aafa32.jpg)

app安装完后的图标和微信的一模一样，只是名字变成了“倍推微信分身”：

![](http://drops.javaweb.org/uploads/images/7dae988c8964973108f019fd35ff1fdf926828b9.jpg)

下载完倍推微信分身，并登陆后，可以看到首页与原版微信并没有太大的变化，只是左上角多了一个VIP的标志：

![](http://drops.javaweb.org/uploads/images/4dd57afd3843cd1a7f02b3152ad741efa34bb2d5.jpg)

我们知道，根据苹果的系统机制，一台iOS设备上不允许存在多个Bundle ID一样的app。因此，我们猜测这个微信分身app是修改过Bundle ID的。于是我们查看一下Info.plist，果然Bundle ID已经做了修改：

![](http://drops.javaweb.org/uploads/images/1a2eeee32217845e2db741a79ecdb89e5342366d.jpg)

但是研究过iOS上微信分身的人一定知道，微信app在启动以及发送消息的时候会对Bundle ID做校验的，如果不是” com.tencent.xin”就会报错并退出。那么”倍推微信分身”是怎么做到的呢？经过分析，原来”倍推微信分身”是通过hook的手段，在app启动的时候对BundleID做了动态修改。至于怎么进行非越狱iOS上的hook可以参考我之前写的两篇文章：

[iOS冰与火之歌番外篇 - 在非越狱手机上进行App Hook](http://drops.wooyun.org/papers/12803)

[iOS冰与火之歌番外篇 - App Hook答疑以及iOS 9砸壳](http://drops.wooyun.org/papers/13824)

于是我们对”倍推微信分身”的binary进行分析，发现这个binary在启动的时候会load一个伪装成一个png文件的第三方的dylib – wanpu.png：

![](http://drops.javaweb.org/uploads/images/e47c6a979aa19219db01fd7174e8e59b8a7852ec.jpg)

用file指令可以看到这个伪png文件其实是一个包含了armv7和arm64的dylib：

![](http://drops.javaweb.org/uploads/images/83be8a72f347bdfe160cb56e860456ffe97e97f4.jpg)

我们看到这个伪图片就像是一个寄生虫一样存在于微信app的体内，特别像dota里的Naix（俗称小狗）的终极技能 - 寄生，因此我们把这个高危样本称之为ImgNaix。

![](http://drops.javaweb.org/uploads/images/db0c2865ee87d549b823fc685d2f698f8000ba2b.jpg)

0x02 wanpu.png分析
=====

用ida打开wanpu.png，可以看到这个dylib分别对BundleID，openURL和NewMainFrameViewController进行了hook：

![](http://drops.javaweb.org/uploads/images/dfa0ff45c471b1280669eb2cae500affe9cea147.jpg)

BundleID不用说，是为了让app在运行的时候改回”com.tencent.xin”。

NewMainFrameViewController的hook函数就是在微信主页上显示VIP的图片，以及传输一些非常隐私的用户数据（ssid, mac, imei等）到开发者自己的服务器上：

![](http://drops.javaweb.org/uploads/images/dcbcda32fcf3b168a3b022fbfd9dd84b06ec1c47.jpg)

![](http://drops.javaweb.org/uploads/images/1157c73edde3607d167c06504dad3e2f5ae5ea5b.jpg)

OpenURL这个hook就很有意思了，这个函数本身是用来处理调用微信的URL Schemes的。看过我之前写过的《iOS URL Scheme 劫持》的文章的人一定知道这个”倍推微信分身”是有能力进行URL Scheme劫持的，如果在Info.plist里进行了声明，手机上所有使用的URL Schemes的应用都有可能被hijack。

除了这些hook以外，我们在竟然在”倍推微信分身”的逆向代码里，发现了Alipay的SDK！一个没想到，在”倍推微信分身”的帮助下，支付宝和微信支付终于走到了一起：

![](http://drops.javaweb.org/uploads/images/b0d8da99b82d1b0e0b1dda7c1b1fad903e3d7f01.jpg)

因为捆绑了支付宝的SDK，”倍推微信分身”可以调用支付宝的快捷支付功能：

![](http://drops.javaweb.org/uploads/images/41ee3d6feb57081e98f09bc4f28a74eefef41f8f.jpg)

通过网络抓包分析，我们可以看到”倍推微信分身”会发送一些服务收费的数据到手机上：

![](http://drops.javaweb.org/uploads/images/9958bd890efc89dab6c23ca9aaad06e8df457226.jpg)

经分析，”倍推微信分身”之所以加入支付宝sdk是为了对这个微信多开app进行收费。因为天下没有免费的午餐，软件开发者之所以制作腾讯的盗版软件”倍推微信分身”就是为了能够获取到一定的收入，所以才会接入支付SDK的。

0x03 高危接口分析
=====

需要注意的是，”倍推微信分身”打开的url数据都是服务端可控的，并且没有进行加密，黑客可以使用MITM (Man-in-the-middle attack) 随意修改推送的内容，进行钓鱼攻击等操作。比如我通过DNS劫持就能够随意修改推送给用户的数据，以及诱导用户去下载我自己设定的企业app，简直和XcodeGhost一模一样（具体细节可以参考我之前发表的[《你以为服务器关了这事就结束了？ - XcodeGhost截胡攻击和服务端的复现，以及UnityGhost预警》](http://drops.wooyun.org/papers/9024)）。

这里我们进行DNS劫持并修改了推送的内容，同时我们把URL替换成了另一个企业应用的下载plist：

![enter image description here](http://drops.javaweb.org/uploads/images/fc815a4acda481505f514ac9d4821c07a94db3ce.jpg)

可以看到我们在启动”倍推微信分身”的时候弹出了更新对话框,还无法取消:

![enter image description here](http://drops.javaweb.org/uploads/images/21ade13a6c4b2b650ac79bfe5fca3aaa91d1abe6.jpg)

点击后,”倍推微信分身”下载了我们替换后的企业应用,一个伪装成微信的假 app:

![enter image description here](http://drops.javaweb.org/uploads/images/f0706a6bcfbc56e9880b2b88a0f81f49c134ee69.jpg)

除此之外,在分析的过程中,我们还发现”倍推微信分身”app 还存在非常多的高危接口,并 且可以利用第三方服务器的控制进行远程调用:

(1). “倍推微信分身”app利用动态加载的方式调用了很多私有API。比如app使用了MobileCoreServices里的`[LSApplicationWorkspace allInstalledApplications]`来获取手机上安装的应用：

![](http://drops.javaweb.org/uploads/images/e7ba3444d2eeb46216fe04582664e403a0842edb.jpg)

![](http://drops.javaweb.org/uploads/images/c007a0b264744e388c295b810ea0a656c23c2b8a.jpg)

![](http://drops.javaweb.org/uploads/images/117b874817ec44d546ba6708591564be5ffc416a.jpg)

比如app使用了SpringBoardServices的SBSLaunchApplicationWithIdentifier。这个API 可以在不需要url scheme的情况下调起目标app：

![](http://drops.javaweb.org/uploads/images/33cd8fb3a5b55dc2bb94ecbab9ae2370edb57526.jpg)

![](http://drops.javaweb.org/uploads/images/68d5d4a0e44b5bceecf0ce0d2b7946535d677ebe.jpg)

比如app加载了和应用安装有关的私有Framework MobileInstallation以及预留了通过URL Scheme安装企业app的接口：

![](http://drops.javaweb.org/uploads/images/0822a1777a8229597b37082b720f2ccb09798315.jpg)

![](http://drops.javaweb.org/uploads/images/540a4a9fa21b7de0961261bd368b9030747964c2.jpg)

(2). “倍推微信分身”app预留了一整套文件操作的高危接口，可以直接对微信app内的所有文件进行操作，这些文件包括好友列表，聊天记录，聊天图片等隐私信息。

![](http://drops.javaweb.org/uploads/images/b6d2f626adbce9937abc60fc80d5b3d0a359f922.jpg)

要知道在iOS上，聊天记录等信息都是完全没有加密的保存在MM.sqlite文件里的：

![](http://drops.javaweb.org/uploads/images/56205832ba0dbdbcb9169d244482fa085ce1400d.jpg)

0x04 总结
=====

虽然我们在样本分析的过程中除了获取用户隐私外，暂时没有捕获到恶意攻击的行为，但这个”倍推微信分身”预留了大量高危的接口（私有API，URL Scheme Hijack，文件操作接口等），并且破解者是可以随便修改客户端的内容，因此不要说推送任意广告和收费信息了，连窃取微信账号密码的可能性都有，简直就像一颗定时炸弹装在了手机上。这样的微信双开你还敢用吗？

从这个样本中，我们已经看到在非越狱iOS上的攻防技术已经变的非常成熟了， 无论是病毒（XcodeGhost）还是破解软件（ImgNaix）都利用了很多苹果安全机制的弱点，并且随着研究iOS安全的人越来越多，会有更多的漏洞会被发现 (e.g., 利用XPC漏洞过App沙盒 http://drops.wooyun.org/papers/14170)。此外，iOS上的app不像Android，简直一点防护措施都没有，当遇到黑客攻击的时候几乎会瞬间沦陷。正如同我在MDCC 2015开发者大会上所讲的，XcodeGhost只是一个开始而已，随后会有越来越多的危机会出现在iOS上，请大家做好暴风雨来临前的准备吧！