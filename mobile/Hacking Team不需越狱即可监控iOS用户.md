# Hacking Team不需越狱即可监控iOS用户

0x00 简介
=====

近日总部位于意大利的监控软件开发公司HackingTeam被黑，415GB文件被泄露，HackingTeam泄漏的数据至少涉及多个针对Android 4.4以下版本的远程代码执行和提权漏洞、多个针对Java、Word的浏览器沙箱逃逸漏洞的完整攻击代码(`exploit`)以及`MacOS X、iOS、Android、WP8`等系统的恶意软件代码，里面有`Flash 0day, Windows字体0day, iOS enterprise backdoor app, Android selinux exploit, WP8 trojan`，更为严重的是，Hacking Team 的终极远控系统RCS，能够感染包括云平台在内的几乎所有平台或介质，实现了全平台的RSC系统（包括windows phone）。

在HackingTeam泄漏的文件，我们发现了有针对IOS进行监控的代码，一旦用户点击运行，就会请求获取一些数据的访问权限并追踪用户的位置，日历和联系人。在这个过程中，手机不需要越狱即可实现。

0x01 监控行为分析
=====

相关代码在`\core-ios-master.zip\core-ios-master\ios-newsstand-app\newsstand-app`文件夹下，通览全部源码之后，我们发现，监控的实现，主要是通过在目标设备上安装一个报刊杂志应用，该应用安装后显示为一个空白应用，也没有图标。起代码文件如下所示：

![enter image description here](http://drops.javaweb.org/uploads/images/97707e645f15da2cce9f592dcd279ae6d4620370.jpg)

首先查看一下应用的信息文件`info.plist`，通过该文件知道该应用为一个报刊杂志类(`NewsstandApp`)应用，并指定了应用的显示风格和图标，部分信息如下所示：

![enter image description here](http://drops.javaweb.org/uploads/images/804c43e91533372456fb647e96ef935176afabae.jpg)

然后看程序入口模块Main.m，Main函数直接调用了AppDelegate类模块启动应用。

![enter image description here](http://drops.javaweb.org/uploads/images/bbfd5c335cac9a6ea8d2f4d0167c8c9b17d3ff3e.jpg)

AppDelegate模块主要功能是任务的派发以及后台刷新。包括后台获取访问权限，不断刷新获取用户日历，联系人以及照片任务，键盘任务等。

![enter image description here](http://drops.javaweb.org/uploads/images/e487c34c0b9213f4c5b195d4ff816f6450e17fec.jpg)

同时该模块调用接口ViewController创建了一个nil类型的gMainView，当view需要被展示而它却是nil时，viewController会调用该方法

![enter image description here](http://drops.javaweb.org/uploads/images/ccff63bc277ad7da14629bc068b85d8829d015ae.jpg)

该方法中实现了获取用户信息的主要功能，其功能模块为ViewConroller.m，该应用被加载后开始获取日历，联系人，GPS位置信息以及用户的照片。启动代码如下所示：

![enter image description here](http://drops.javaweb.org/uploads/images/3c55ae8680d3371f559b6c9538c724a5b6ba3074.jpg)

所有获取到的信息都会通过RCS系统发送到远程服务器，其RCS模块以RCS开头，传输过程中采用一系列的加密方式，加密模块主要为以NS开头的模块，起后边跟上其加密方式，剩下的还有键盘的实现模块和网络传输模块，如下所示：

![enter image description here](http://drops.javaweb.org/uploads/images/8cf05aecf0cbed714a41347a90ad25a275c419b3.jpg)

0x02 实测行为
=====

将程序编译，然后在手机上运行。安装过后，会创建报刊杂志隐藏应用，长按应用即可显示出来，在设置的应用列表中也可以看到，如下图所示：

![enter image description here](http://drops.javaweb.org/uploads/images/155873f84c00b721cffe1001b218a5593a4dfdad.jpg)![enter image description here](http://drops.javaweb.org/uploads/images/05494c86f6477eaacea12f3170a882e14e88fddb.jpg)![enter image description here](http://drops.javaweb.org/uploads/images/0dc4fd02b26dc72c961d5c4907a9656ce0843100.jpg)

点击该应用后，其所有的监控服务就会启动起来，越狱用户是没有任何提示直接启动，而非越狱用户需要添加信任，如下图所示：

![enter image description here](http://drops.javaweb.org/uploads/images/28decc9c3966090cae2d7f37bfd009f8e9298112.jpg)

但是该黑客团队拥有企业证书，因此它可以通过类似于网络链接的方式引导用户下载安装而不被察觉。

服务一旦启动起来，应用开始请求他所有想获取的数据权限，如下图所示：

![enter image description here](http://drops.javaweb.org/uploads/images/37daf51a04804e07e321194ff981ca833e8c9881.jpg)![enter image description here](http://drops.javaweb.org/uploads/images/359d9f96c6ec7f12271fa6cba06ce26174aabc4b.jpg)![enter image description here](http://drops.javaweb.org/uploads/images/af15206c948bc255f8bd7b2fe4830bc523078cb4.jpg)

同时该应用增加了一个新的键盘，键盘界面和原生的IOS内置键盘相同，因此被攻击用户会在没有察觉的情况下将他们所有的输入信息发送到远程服务器上。键盘如下所示：

![enter image description here](http://drops.javaweb.org/uploads/images/ad19ff9c76091183b83f8c94e86710f3a04ca836.jpg)![enter image description here](http://drops.javaweb.org/uploads/images/b07400b8fbcdbf8ec24faa76e9a6378f133dccb8.jpg)

需要注意的是，苹果公司针对第三方键盘做了一些保护措施，他不允许第三方键盘运行在具有密码标记的区域，因此该工具并不能够从应用程序和网站中窃取用户的输入密码，但它可以窃取用户名，电子邮件等其他敏感信息。

0x03 感染途径
=====

苹果公司做了大量的工作去保护了非越狱用户远离恶意软件，公共报道的也是监控软件只能去感染越狱的IOS设备，看似非越狱用户是安全的。

Hacking Team拥有苹果的企业证书，而企业证书是由苹果公司发布给企业，并且允许企业不经过appstore审核直接将自己的应用发布到自己的网站上。其他人可以直接下载不用设备授权即可直接安装，并且不限设备上限，因此使得该证书签名的任何应用，不论目标IOS设备是否越狱，都可以安装上。并且该监控工具是一个隐藏的报刊应用，因此可以分发到任何一台IOS设备上。但是苹果公司对此也做了一些安全警告，需要未越狱用户点击信任才可安装，但是从企业网站上下载的应用，用户一般都会忽略掉。

另外还可以通过捆绑越狱工具直接安装到用户手机中，或者通过点击一些下载链接，email等也可以安装在用户设备中。

0x04 总结
=====

苹果公司做了大量的工作去保护了非越狱用户远离恶意软件，公共报道的也是监控软件只能去感染越狱的IOS设备，看似非越狱用户是安全的。而Hacking Team企业证书的滥用导致监控工具恶意传播，对非越狱用户也造成了极大的危害，而苹果公司也在前不久吊销了该团队的企业证书，但是潜在的威胁依然存在，用户在平时下载第三方应用时候也要多注意程序的来源是否可信。

确认自己手机是安装远程监控可通过如下方法查看：

```
1.检测应用中是否具有空名称的应用程序
2.设置->通用->键盘查看是否有名为app.keyboard的第三方键盘

```

一些安全建议：

```
1.添加手机密码。很多间谍软件安全都需要物理接触，添加密码使得它们的攻击更艰难。
2.不要下载来自第三方市场或者链接的应用。
3.尽量不要越狱手机，如果不清楚请求权限的软件是什么，不要添加信任
4.下载安全程序，定期扫描手机系统，如cm security等。
```