# 【安天CERT】利用路由器传播的DYREZA家族变种分析

**微信公众号:Antiylab**

0x00 概述
=====

安天CERT（安全研究与应急处理中心）近期收到大量用户反馈，称其收到带有可疑附件的邮件，经过安天CERT研究人员分析发现，这是一类利用垃圾邮件进行传播和下载的木马家族Dyreza的变种，其目的是窃取银行账号和比特币。该变种通过Upatre下载者进行下载，下载Dyreza变种的服务器均为路由器。攻击者将入侵的路由器作为Dyreza变种的传播服务器，在路由器中存放的文件均为加密文件。此外，该变种还具有反虚拟机功能。在分析过程中，安天CERT研究人员发现大量的路由器被植入了Dyreza的最新变种。

0x01 事件样本分析
=====

1.1 传播过程

![enter image description here](http://drops.javaweb.org/uploads/images/48de759692c59d15c78a83640a829b59c8e94393.jpg)

图1 Dyreza木马的传播过程

不法分子首先利用弱口令等方法入侵互联网中的路由器，在路由器中存放加载的恶意代码程序，这些恶意代码程序的后缀名包括： .AVI、.ZIP、.TAR、.RAR、.PNG、.PDF；然后通过散布带有社会工程学性质的垃圾邮件，诱使用户运行附件中的Upatre下载者；Upatre下载者连接被入侵的路由器，下载路由器中存放的加密的恶意代码程序，在用户系统中解密后得到Dyreza木马。

1.2 样本标签

![enter image description here](http://drops.javaweb.org/uploads/images/6bfe93fdcac29d11698370178cd0d4bb89c7eb92.jpg)

本次事件中，Upatre下载者负责下载窃取银行账号和比特币的Dyreza木马程序。Upatre下载者主要通过电子邮件进行传播，目前以Upatre家族为载体的木马家族有Zeus、Rovnix、Dyreza、勒索软件和僵尸网络等。

1.3 Upatre样本分析

1.  样本使用了多层混淆技术来阻止反病毒工程师对其进行分析。Upatre运行后，首先创建进程svchost.exe，并将自身代码注入到svchost.exe进程中执行，同时结束自身进程；通过ZwQueryInformationProcess函数检测自身是否在调试器下运行；最后通过ZwResumeThread函数恢复线程启用。

![enter image description here](http://drops.javaweb.org/uploads/images/e0dcac02f8c1cc150bbd48ee0973ecb373c9578d.jpg)

图2 将自身代码注入到创建的svchost.exe进程中

1.  通过HTTPS进行下载，如下载失败会循环下载其它路由器的IP地址进行下载，直到下载成功。

![enter image description here](http://drops.javaweb.org/uploads/images/fdba9f65c680a6f7bfcce37c295e0c4c8e3dd1e0.jpg)

图3 下载加密的恶意代码文件

1.  目前安天CERT研究人员发现通过路由器IP地址下载的文件均为加密的Dyreza变种。其文件后缀名如下：

![enter image description here](http://drops.javaweb.org/uploads/images/6526c5f739025477d81c7c2d6ad17a7ed2244ba2.jpg)

经安天CERT研究人员测试发现在同一个路由器上存在多个加密文件。其中某路由器上存放着99个加密恶意代码，文件名列表如下：

![enter image description here](http://drops.javaweb.org/uploads/images/3307b8a799814dc714baa287beb4c9bc836dadc4.jpg)

放置加密恶意代码的路由器登陆界面：

![enter image description here](http://drops.javaweb.org/uploads/images/71c3dfe246fc811cfc67d59bdeb8df33c704905f.jpg)

图4 放置加密恶意代码的路由器连接界面

1.  在下图，全球被入侵路由器的IP地址的地理位置中，排在前三位的分别是：美国、波兰、乌克兰；其中欧洲国家较多。

![enter image description here](http://drops.javaweb.org/uploads/images/f66bfe4c177c97772aeea68e406866cea33c72a5.jpg)

图5 存放加密恶意代码的路由器的IP地址分布

1.  Upatre下载者将加密文件下载到本地后进行解密

![enter image description here](http://drops.javaweb.org/uploads/images/950e9e82e41109cedfa50a8ebf9818ef4fc33ce7.jpg)

图6 解密代码

首先略过加密文件头部的4个字节，然后每次取出4个字节与Key进行异或，每次运算后Key减1。依此循环进行解密，直至解密完成，解密后的文件是一个可执行的PE文件，即Dyreza木马。

0x02 Dyreza家族变种分析
=====

2.1 样本标签

![enter image description here](http://drops.javaweb.org/uploads/images/9f0102c1b0a42fa8bfdf4b7a1a59bb77b2303d56.jpg)

Dyreza家族非常类似于臭名昭著的Zeus僵尸网络，它们都是利用浏览器中间人攻击，当被感染的用户访问特定的网站（这类网站通常为金融机构或金融服务的登录页面），则会注入恶意Javascript代码来进行捕获用户所输入的账号密码等信息。Dyreza家族新变种的主要功能是窃取用户银行账号和比特币。

2.2 Dyreza家族变种分析

1.  反虚拟机功能：

2006年英特尔发布双核处理器后，现今市场及用户电脑中已经很难见到单核处理器了。而自动化分析平台为节省资源，常常将虚拟机的处理器设置为单核处理器。Dyreza家族的新变种正是利用这种情况，对感染系统的处理器个数进行判断，如果少于2个，则样本直接退出。该判断在样本中出现多次，在后面核心的DLL模块中，也存在此功能。

![enter image description here](http://drops.javaweb.org/uploads/images/acadc41ce11337444da287369ea6aab00dbe5e1a.jpg)

图7 判断处理器是否为单核CPU

1.  资源解密： 样本带有多个资源文件，将这些资源读取到内存后，根据内置的一个100字节的shellcode表来进行解密操作，得到核心的DLL功能模块。

![enter image description here](http://drops.javaweb.org/uploads/images/01caa8e4e59e22ca54867e8a4b2e802cb960ae7d.jpg)

图8 解密资源文件

1.  创建虚假的google升级服务：

![enter image description here](http://drops.javaweb.org/uploads/images/28bf3fa3b04ca0110436b09e31082c46affb814f.jpg)

图9 创建虚假的google升级服务

4.使用任务计划执行程序，每分钟执行一次：

![enter image description here](http://drops.javaweb.org/uploads/images/e2086b9e371e9523b794df39470062b029ecee2e.jpg)

图10 每分钟运行一次自身

1.  在核心模块中，使用了跨平台的文件加密库bcrypt，将获取的本地信息进行加密操作。

![enter image description here](http://drops.javaweb.org/uploads/images/4cd5c5ba2d58e799f73d4e111e29091e4ae92f08.jpg)

图11 使用的加密函数

1.  Dyreza家族具有远程控制用户系统的功能，使用POST和GET方式与远程服务器进行通信。

![enter image description here](http://drops.javaweb.org/uploads/images/97e6d18d917960395a74b7f491a3fb2275b0a8e7.jpg)

图12 使用POST和GET方式进行通信

1.  Dyreza家族的新变种可以在被感染的系统中添加一个管理员账号和密码，账号名为“lname0”,密码为“1qazxsw2”。通过本地登陆验证是否添加成功。

![enter image description here](http://drops.javaweb.org/uploads/images/eba9027d4f0cbab703c7777b0c516491811b525c.jpg)

图13 创建管理员账号和密码

0x03 路由器防护建议
=====

Dyreza家族木马通过入侵路由器的方式进行传播，恶意程序放置在路由器中很难被用户发现，而反病毒产品通常无法直接扫描路由器。因此，安天CERT建议用户使用以下几种路由器防护措施：

不要使用路由器默认的口令，修改为高强度的口令。

定期更新路由器固件，修复已出现的安全漏洞。

关闭SSID广播，防止SSID被嗅探。

使用安全性较高的WPA2协议、AES加密算法。

禁用DHCP，开启MAC地址过滤，仅允许绑定的MAC地址访问无线网络。

0x04 总结
=====

Dyreza家族以窃取用户银行账号和比特币为目的，以利用入侵的路由器进行传播为特点，应引起用户和企业的关注。在此之前，2014年4月份的CVE-2014-0160（心脏出血）漏洞即可入侵大量的路由器设备。安天CERT判定，Dyreza家族与Rovnix家族有着必然的联系，它们使用相同的Upatre下载者进行传播，并使用相似的下载地址。在本报告发布前，安天又捕获到一个更新的Dyreza变种，在传播方式上，它使用与Rovnix攻击平台[g1]相似的下载地址，都使用WordPress搭建的网站，或入侵由WordPress搭建的第三方正常网站。继HaveX首次使用这种传播方式后，这已成为安天CERT发现的第三个使用这种传播方式的家族了。安天CERT会继续跟踪使用这种方式传播的恶意代码，并持续关注HaveX、Rovnix、Dyreza这三个家族。

[g1 http://www.antiy.com/response/ROVNIX.html

[博文地址](http://drops.wooyun.org/papers/7478)