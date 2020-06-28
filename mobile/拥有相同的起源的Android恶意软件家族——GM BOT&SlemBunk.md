# 拥有相同的起源的Android恶意软件家族——GM BOT&SlemBunk

0x00 前言
=====

二月19号，IBM XForce的研究员发布了一个情报报告，这份报告声称GM BOT的源代码在2015年12月被泄漏到了一个恶意软件的论坛上。GM BOT是一个复杂的安卓恶意软件家族，它出现在2014年末俄语地区的网络犯罪地下组织。IBM同时声称几个在最近的安全会议上提出的Android恶意软件家族实际上也是GM BOT的变种，包括[Banskosy](http://www.symantec.com/connect/blogs/androidbankosy-all-ears-voice-call-based-2fa)，[MazarBot](https://www.csis.dk/en/csis/news/4819/)，以及FireEye最近提到的恶意软件[Slembunk](https://www.fireeye.com/blog/threat-research/2015/12/slembunk_an_evolvin.html)。

安全厂商可能在一个恶意软件的“变种”定义上各不相同。这一术语可以指完全相同的代码仅仅做轻微的改动，也可以指表面相似然而其他地方非常不同的代码（比如有相似的网络通信）。

通过使用IBM的报告，我们分析了它们的GM BOT样本和SlemBunk的样本。基于这两个家族的反汇编代码，我们认为有足够多的代码相似性可以表明GM BOT和SlemBunk拥有相同的起源。有意思的是，我们的研究让我们也认为早期的一个恶意软件家族SimpleLocker（Android上最早为人所知的文件加密型勒索软件）也同样具有和之前两个家族相同的起源。

0x01 GM BOT和SlemBunk
=====

我们的分析发现最近被IBM研究者提到的四个GM BOT样本都和SlemBunk拥有相同的主要组件。如下图所示，我们早期的报告展示了SlemBunk主要的组件以及相应的类名：

*   **ServiceStarter**：一旦一个应用被启动或者设备启动就会调用这个Android接收器。它的功能是在后台启动监控服务，MainService。
*   **MainService**：运行在后台并监控所有在设备上运行的进程的Android服务。当一个应用被启动后，它会为用户提供一个覆盖式的类似于合法应用的视图。这项监控服务会通过发送原始设备信息、告知设备状态和应用程序偏好来和一台远程主机进行通信。
*   **MessageReceiver**：处理进入的文本信息的Android接收器。除了拦截来自银行的验证码的功能之外，这个组件也充当着作为一个发送远程命令和进行控制的机器人客户端（C2）。
*   **MyDeviceAdminReceiver**： 当这个应用首次被启动之后，这个接收器就会请求这个Android设备的管理员权限。这会让这个应用更难被移除。
*   **Customized UI views**：呈现虚假登陆页面的Android类，这些虚假的登录页面都是模仿真正的银行应用 或者社交应用的，这样就可以进行钓鱼来获得银行或者社交账户凭证。

![p1](http://drops.javaweb.org/uploads/images/73d9efbeaf08cea6471261fe3cc6cc7de37c95c0.jpg)

前三个GM BOT样本都和我们的SlemBunk样本有相同的包名。除此之外，这些GM BOT样本有5个相同的主要的组件，包括和上图的SlemBunk样本一样，拥有相同的组件名称。

第四个GM BOT有不同的初始化包名，但是会在运行时解压真正的payload。解压的payload拥有和SlemBunk相同的主要组件，只是在类名上有一些小的改变：比如MessageReceiver替换成了buziabuzia，以及MyDeviceAdminReceiver替换成了MDRA。

![p2](http://drops.javaweb.org/uploads/images/b62ed0381bd0cf8bca730c23ad7e9ea11768422e.jpg)

上图展示的是**GM BOT样本**（SHA256 9425fca578661392f3b12e1f1d83b8307bfb94340ae797c2f121d365852a775e）和一个**SlemBunk样本**（SHA256 e072a7a8d8e5a562342121408493937ecdedf6f357b1687e6da257f40d0c6b27）之间代码结构的相似性。从这张图中我们可以发现，我们之前讨论的5个主要的组件也出现在了GM BOT样本里。其他共同的类包括：

*   **Main**: 两个例子中的启动的activity。
*   **MyApplication**：两个例子中在任何其他的activity启动之前启动的应用的类。
*   **SDCardServiceStater**:另外一个监视着MainService的接收器，一旦MainService挂掉了就把它重启。

在上面所有的这些组件和类中，MainService是最重要的一个。它是在启动时被Main类启动的，并持续在后台运行来监控运行最多的进程，当一个受害者的应用（比如一些手机银行APP）被监控到时就将一个钓鱼页面覆盖上去。为了保持MainService持续运行，恶意软件作者添加了两个receiver——ServiceStater和SDCardServiceStater，以此来检查接收到特殊的系统事件时的运行状态。GM BOT和SlemBunk样本都有同样的结构。下图展示的SDCardServiceStater类的主要代码是为了说明GM BOT和SlemBunk使用了同样的机制来维持MainService运行。

![p3](http://drops.javaweb.org/uploads/images/9668d1613c059d69dc74ccd9ebca4d8c46ec5308.jpg)

从这张图中我们可以发现，GM BOT和SlemBunk使用几乎相同的代码来保持MainService运行。值得注意的是两个样本都在系统区域设置中检查了城市，如果是俄罗斯就不启动MainService。唯一的区别就是GM BOT对某些类名、方法和字段进行了重命名混淆。比如，GM BOT中的静态变量“MainService;->a”和SlemBunk中的“MainService;->isRunning”是同样的功能。恶意软件作者使用这种方式来让他的代码更难去理解。然而这并不会改变根本的代码是来自于相同的起源这一事实。

下图展示的MainService类的核心代码说明GM BOT和SlemBunk拥有相同的主服务逻辑。在Android中，当一个服务被启动，就会调用它的onCreate方法。在两个样本中的onCreate方法里，一个静态的变量都会首先被设置为“True”。在GM BOT中，这个变量被命名为“a”，而在SlemBunk中它被命名为“isRunning”。并且它们都会去读取应用程序偏好，并且在两个样本中有同样的命名：“AppPrefs”。这两个主服务的最后的任务也是相同的。如果一个受害者的应用在运行，一个钓鱼的页面就会覆盖到受害者的应用之上。

![p4](http://drops.javaweb.org/uploads/images/7c5210a5d1b8aa374c79860667c3b7651d88441b.jpg)

0x02 SimpleLocker和SlemBunk
=====

IBM声称GM BOT出现在2014年末俄语地区的网络犯罪地下组织。在我们的研究中，我们注意到更早的名为“SimpleLocker”的Android恶意软件也有和GM BOT、SlemBunk相似的代码框架。然而，SimpleLocker有不同的目的：从受害者处索要赎金。在安装到一个Android设备以后，SimpleLocker会扫描设备的特定文件类型，对它们进行加密然后向受害者索要解密的赎金。在SimpleLocker出现之前，也有其他的Android恶意软件会将屏幕锁住；但是SimpleLocker是第一个Android上的文件加密型的勒索软件。

我们知道的更早的关于SimpleLocker的报告是ESET发表于2014年6月的（[Links](http://www.welivesecurity.com/2014/06/04/simplocker/)）。然而，我们也在我们2014年5月的恶意软件数据库中发现了一个更早的样本(SHA256 edff7bb1d351eafbe2b4af1242d11faf7262b87dfc619e977d2af482453b16cb)。这个APP编译的时间是2014年5月20号。我们把这个SimpleLocker的样本和其中一个SlemBunk的样本(SHA256 f3341fc8d7248b3d4e58a3ee87e4e675b5f6fc37f28644a2c6ca9c4d11c92b96)用比较GM BOT和SlemBunk的方法进行了比较。

下图展示的是两个样本代码结构的比较。值得注意的是这个SimpleLocker的变种也有主组件ServiceStater和MainService，这两个也都被SlemBunk所使用。然而，这里的主服务的目的不是监视进程的运行并进行钓鱼来获取银行信息。SimpleLocker的主服务组件会扫描设备来查找受害者的文件，然后会调用文件加密类来对文件进行加密进而进行勒索。SimpleLocker代码中主要的不同如下图红框所示：AesCrypt和FileEncryptor。其他共同的类包括：

*   **Main**, 两个样本中的启动的activity。
*   **SDCardServiceStarter**，另一个receiver监视MainService的状态，当它死掉之后会把它重启。
*   **Tor and OnionKit**，第三方的lib库文件用来进行私密通信。
*   **TorSender, HttpSender and Utils**，支持类来提供代码以进行CnC通信，以及收集设备信息。

![p5](http://drops.javaweb.org/uploads/images/46e0990191e775686c6b21e68c4de9783d9c77b2.jpg)

最后，我们定位到了另一个2014年6月的SimpleLocker样本(SHA256 304efc1f0b5b8c6c711c03a13d5d8b90755cec00cac1218a7a4a22b091ffb30b)，大约在第一个SimpleLocker样本被发现两个月之后。这个新的样本没有使用Tor来进行私密通信，但是和之前的SlemBunk样本(SHA256: f3341fc8d7248b3d4e58a3ee87e4e675b5f6fc37f28644a2c6ca9c4d11c92b96)的5个主要的组件有四个是相同的。下图展示了两个样本的代码结构的比较。

![p6](http://drops.javaweb.org/uploads/images/4077dc40e9bf8908f938c68fe8e3fb2b0ffb6eee.jpg)

正如我们在上图中看到的那样，新的SimpleLocker样本使用了和SlemBunk相似的打包机制，把HttpSender和Utils打包到了名叫“utils”的子包中。它也添加了两个其他的最初只在SlemBunk中看到的主要组件：MessageReceiver和MyDeviceAdminReceiver。总的来说，这个SimpleLocker的变种木马和SlemBunk在5个主要组件里有4个是相同的。

下图展示的是上个例子中MessageReceiver的主要的代码，来证明SimpleLocker和SlemBunk基本上使用了相同的进程和逻辑来和CnC服务器进行通信。首先，MessageReceiver类会先register来处理收到的短消息，这些短消息的到达会触发onReceive方法。正如图中所示，这里的主逻辑基本上是和SimpleLocker和SlemBunk一样的。他们首先会从应用偏好设置中读取一个特殊的键的键值。这个键的名字和共享的偏好设置对于这两个不同的恶意软件家族来说是一样的：键名为“`CHECKING_NUMBER_DONE`”，偏好设置名为“AppPrefs”。下面的步骤调用retriveMessage方法来检索短消息，接下来会把控制流传到SMSProcessor类。这里唯一的不同就是SimpleLocker添加了一个额外的方法processControlCommand来进行控制流传输。

SmsProcessor类定义了恶意软件家族支持的CnC命令。通过对SmsProcessor的观察，我们发现了更多的证据证明SimpleLocker和SlemBunk是有相同的起源的。首先，SimpleLocker支持的CnC命令实际上是SlemBunk支持的命令的子集。在SimpleLocker中，CnC命令包括“`intercept_sms_start`”、“`intercept_sms_stop`”、“`control_number`”和“`send_sms`”，所有的这些都在SlemBunk样本中出现过。并且，在SimpleLocker和SlemBunk中都有共同的前缀“`#`”在实际的CnC命令前。这也是SimpleLocker和SlemBunk有相同起源的非常重要的证据。

![p7](http://drops.javaweb.org/uploads/images/579aa1d3801f23da46db6be35ba703871eafddfe.jpg)

MyDeviceAdminReceiver类的任务是请求设备管理员权限，这会让这些恶意软件很难被移除。SimpleLocker和SlemBunk在这个方面极其类似，支持一套相同的设备管理相关函数。

我们可以发现这些SimpleLocker和SlemBunk的变种木马在主要控件上基本相同，和对相同的设备支持性上的相似性。唯一不同的就是最后的payload，SlemBunk是为了对银行信息进行钓鱼而SimpleLocker是为了加密文件索要赎金。这也让我们相信SimpleLocker和SlemBunk来自于相同的作者之手。

0x03 结论
=====

我们的分析确认这几个Android恶意软件家族拥有共同的来源。更多的研究可能会发现更多相关的恶意软件家族。

网络犯罪地下组织的独立的开发者在编写和定制恶意软件方面是很熟练的。正如我们所知，带有特殊或者多样化目的的恶意软件可以建立在对一些通用功能的共享代码上实现，比如获取管理员权限、启动以及重启服务、以及CnC通信。

随着GM BOT代码的泄露，基于这些代码定制的Android恶意软件家族必然会增多。

0x04 Reference
=====

*   [[1]. Android Malware About to Get Worse: GM Bot Source Code Leaked[10]](https://securityintelligence.com/android-malware-about-to-get-worse-gm-bot-source-code-leaked/)
*   [[2]. Android.Bankosy: All ears on voice call-based 2FA[11]](http://www.symantec.com/connect/blogs/androidbankosy-all-ears-voice-call-based-2fa)
*   [[3]. MazarBOT: Top class Android datastealer[12]](https://www.csis.dk/en/csis/news/4819/)
*   [[4]. SLEMBUNK: AN EVOLVING ANDROID TROJAN FAMILY TARGETING USERS OF WORLDWIDE BANKING APPS[13]](https://www.fireeye.com/blog/threat-research/2015/12/slembunk_an_evolvin.html)
*   [[5]. SLEMBUNK PART II: PROLONGED ATTACK CHAIN AND BETTER-ORGANIZED CAMPAIGN[14]](https://www.fireeye.com/blog/threat-research/2016/01/slembunk-part-two.html)
*   [[6]. ESET Analyzes Simplocker – First Android File-Encrypting, TOR-enabled Ransomware[15]](http://www.welivesecurity.com/2014/06/04/simplocker/)