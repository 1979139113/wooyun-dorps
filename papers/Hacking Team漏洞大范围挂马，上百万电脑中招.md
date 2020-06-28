# Hacking Team漏洞大范围挂马，上百万电脑中招

0x01 概况
=====

近日，腾讯反病毒实验室拦截到一个恶意推广木马大范围传播，总传播量上百万，经分析和排查发现该木马具有以下特征：

1） 该木马是通过网页挂马的方式传播的，经分析黑客用来挂马的漏洞是前段时间Hacking Team事件爆出的flash漏洞CVE-2015-5122。新版本的Flash Player已经修复了该漏洞，但是国内仍有大量的电脑未进行更新，给该木马的传播创造了条件。

2） 经分析和追踪，发现挂马的主体是一个广告flash，大量存在于博彩类网站、色情类网站、外挂私服类网站、中小型下载站等，以及部分流氓软件的弹窗中，影响广泛。

3） 挂马的漏洞影响Windows、MacOSX和Linux平台上的IE、Chrome浏览器等主流浏览器。经测试，在未打补丁电脑上均可触发挂马行为，国内主流浏览器均未能对其进行有效拦截和提醒。

4） 木马更新变种速度快，平均2-3小时更换一个新变种，以此逃避安全软件的检测，同时可降低单个文件的广度，逃过安全软件的广度监控。

5） 该木马主要功能是静默安装多款流氓软件，部分被安装的流氓软件具有向安卓手机静默安装应用的功能，危害严重。同时该木马还玩起“黑吃黑”——能够清除已在本机安装的常见的其它流氓软件，达到独占电脑的目的。

![](http://drops.javaweb.org/uploads/images/ce06209e1eeebca00d009d649709410880008631.jpg)

图1. Flash 漏洞挂马示意图

0x02 挂马网站分析
=====

经分析和追踪，发现挂马的主体是一个广告flash，当访问到挂马网站时，该flash文件会被自动下载并播放，从而触发漏洞导致感染木马。如图2所示。

![](http://drops.javaweb.org/uploads/images/221ad088a063f8967bfd4fdc5b1d87dd051da992.jpg)

图2. 被挂马的网站之一

图3所示为带木马的Flash文件，有趣的是如果当前电脑flash已经打补丁，则显示正常的广告，如果flash未打相应补丁则会触发漏洞，在浏览器进程内执行ShellCode。

![](http://drops.javaweb.org/uploads/images/7bb9a4619c4146d20ae392f51ace26054f1b0073.jpg)

图3. 挂马的flash文件

通过反编译flash文件可以发现，该flash是在Hacking Team泄漏代码的基础上修改而来的，该flash使用doswf做了加密和混淆，以增加安全人员分析难度。如图4所示为挂马flash代码。

![](http://drops.javaweb.org/uploads/images/c2ee01df8053d555d24343dbb6f38af4f1548a94.jpg)

图4. 挂马的flash文件反编译代码

漏洞触发后，直接在浏览器进程中执行ShellCode代码，该ShellCode的功能是下载`hxxp://222.186.10.210:8861/calc.exe`到本地，存放到浏览器当前目录下，文件名为`explorer.exe`并执行。

![](http://drops.javaweb.org/uploads/images/dc02d2c0d36baa1acd4c008d325bc77129bd3b57.jpg)

图5. ShellCode经混淆加密，其主要功能是下载执行

`Explorer.exe`为了逃避杀软的查杀，其更新速度非常块，平均2-3小时更新变种一次，其变种除了修改代码以逃避特征外，其图标也经常变动，以下是收集到的`explorer.exe`文件的部分图标，可以发现主要是使用一些知名软件的图标进行伪装。

![](http://drops.javaweb.org/uploads/images/be17f89802198052c07eaa23a217bb21dbea455a.jpg)

图6. 木马使用的伪装图标列表

该木马传播量巨大，为了防止样本广度过高被安全厂商发现，变种速度飞快，该木马10月中旬开始传播，截至目前总传播量上万的变种MD5统计如下：

![](http://drops.javaweb.org/uploads/images/d95f7453dd25588b1ebf02ee71b9592462575d62.jpg)

图7. 截至目前总传播量上万的变种MD5列表

0x03 木马行为分析
=====

木马运行后会通过检测判断是否存在`c:\okokkk.txt`文件，如果存在则表示已经感染过不再感染，木马退出，如果不存在则创建该文件，然后创建两个线程，分别用于安装推广流氓软件和清理非自己推广的流氓软件等。

![](http://drops.javaweb.org/uploads/images/6aeeceb16be8d64f4f115acfba9b9936173735c0.jpg)

图8. 通过判断“`C:\okokkk.txt`”文件来防止重复感染

“黑吃黑”是此木马的一大特色，木马运行后专门创建了一个线程，用于检测和删除其它软件的桌面快捷方式、开始菜单项中的目录等，删除的项目大部分为流氓软件，其内置的列表堪称流氓软件大全，涵盖大量的流氓软件，以下只是列表的一部分。

![](http://drops.javaweb.org/uploads/images/a650c15c6d4a2d35ab91deffb865e4ce1ac23613.jpg)

图9. 部分被删除的桌面快捷方式

![](http://drops.javaweb.org/uploads/images/f42ca3f98992e7c909bd7a9f2714990199e5a3d9.jpg)

图10. 木马要删除的部分菜单项列表

循环结束指定进程，包括`taskmgr`（任务管理器）、`QQPCDownload`（管家在线安装）等多款软件的安装程序。

![](http://drops.javaweb.org/uploads/images/4bc0edb0108deab4a8e6d0e7913dc4bdf25a93eb.jpg)

图11. 循环结束指定进程

向统计页面发送信息，统计安装量，随后下载并静默安装以下软件（共11款），其中`hbsetup64.exe`是一款流氓软件，能够静默向连接电脑的安卓手机安装手机应用，危害严重

![](http://drops.javaweb.org/uploads/images/6b830edfd17b1a5b53a7366fac190c61cf1b60bb.jpg)

图 12.该木马静默推广的软件列表及安装统计地址

0x04 后记
=====

被称作“黑客核武库”的Hacking Team泄漏数据大大降低了黑客攻击的门槛，把整个黑色产业链的技术水平提高一个档次，攻击者仅需要对线程的代码做少量的修改便可生成强大的攻击“武器”，对整个互联网安全构成了严重的威胁，还希望广大互联网用户及时安装相关安全补丁。