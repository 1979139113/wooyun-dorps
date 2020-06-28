# 一款结合破壳(Shellshock)漏洞利用的Linux远程控制恶意软件Linux/XOR.DDoS 深入解析

原文：[http://blog.malwaremustdie.org/2015/07/mmd-0037-2015-bad-shellshock.html](http://blog.malwaremustdie.org/2015/07/mmd-0037-2015-bad-shellshock.html)

0x00 背景
=====

昨天又是忙碌的一天，因为我获知近期有破壳漏洞利用攻击，所以我们团队集合起来对所有近期的捕获到的互联网交叉流量中的ELF文件威胁 (ELF threats)进行检测。我查看了一套shell脚本的命令部分，立刻就知道它基本上就是`Linux/XOR.DDoS`木马。我又检查了`payload`，并且深入查看了细节。最终确定这就是`Linux/XOR.DDoS`。

相关信息可以查看：[http://blog.malwaremustdie.org/2015/06/mmd-0033-2015-linuxxorddos-infection_23.html](http://blog.malwaremustdie.org/2015/06/mmd-0033-2015-linuxxorddos-infection_23.html)

[https://www.virustotal.com/en/file/dced727001cbddf74303de20211148ac8fad0794355c108b87531b3a4a2ad6d5/analysis/](https://www.virustotal.com/en/file/dced727001cbddf74303de20211148ac8fad0794355c108b87531b3a4a2ad6d5/analysis/)

因为已经过我的睡觉时间很久了，我需要休息了。所以我让我们生活在地球另一边的专家同事们`MMD ELF Team`来看看这个样本是否有什么新特性。

为了搞定这些新的发现，我几乎一夜没睡。这篇文章将会告诉你为什么。

0x01 shellshock
=====

这次`shellshock`攻击是从13/07/2015开始的。让我们直接来看看利用`shellshock`植入的一段`bash`命令的代码：

![enter image description here](http://drops.javaweb.org/uploads/images/73af0a681e7d858598c70547fd6c9b8abd42f92e.jpg)

这段代码的主要功能是： 使用`web server`的UID来执行代码，这段脚本删除了sftp守护进程的pid文件(pid文件是某些程序启动时记录一些进程ID信息而创建的文件)，检查当前目录是否有6000.rar这个文件存在，如果有就删除。然后，在根目录通过wget或者curl命令从多个IP地址(43.255.188.2/103.20.195.254/122.10.85.54)下载`6000.rar`文件。检查是否下载成功，如果下载成功了就添加可执行权限并执行，最后打印一个“ExecOK”消息。如果下载不成功，它也会继续探测系统的版本信息，`HDD status`，进程信息(如果是linux系统将会显示`UID，PID，PPID，Time和Command`，但是在BSD系统中会显示一些不同的有趣结果:D)并且会检查以太网连接，显示本地和远程地址，接入互联网的程序。然后打印一个“`ExecOK`”消息。最后，这个脚本会打印`sftp，mount，gcc`的PID内容。再打印一个“InstallOK”消息。

最后，它从上面列出的IP地址的其中一个下载`g.rar`文件(使用wget或crul命令)。添加可执行权限并执行。需要指出的是，在大多数`Linux/XOR.DDoS shellshock`案例中，最后这一步都没有执行。被`“#”`注释掉了。

0x02 Linux/XOR.DDoS payloads
=====

至今，我们`MalwareMustDie`是第一个发现这类ELF威胁样本的，并且我们给其命名为`Linux/XOR.DDoS`，我们的博客中有分析文章，发布于29/09/2014[[link]](http://blog.malwaremustdie.org/2014/09/mmd-0028-2014-fuzzy-reversing-new-china.html).该恶意软件在2015年初造成了很大的影响，很多IT媒体，SANS ISC都有相关报道，参见[[-1-]](http://www.infosecurity-magazine.com/news/massive-ddos-bruteforce-targets/)[[-2-]](http://www.scmagazine.com/malware-targets-linux-and-arm-architecture/article/391497/)[[-3-]](http://www.pcworld.com/article/2881152/ddos-malware-for-linux-systems-comes-with-sophisticated-custombuilt-rootkit.html)[[-4-]](https://isc.sans.edu/forums/diary/XOR+DDOS+Mitigation+and+Analysis/19827/)。

这次威胁变种使用的加解密技术和之前其他相似的中国`ELF DDOS`变种使用很多解码技术(对安装脚本进行编码)和加密技术(基于XOR)有明显的不同。所有我们的团队在处理威胁的时候就像和恶意软件作者在进行一场CTF比赛。这对下班或放学后的我们有好处，使我们的大脑不会闲着。我们在`kernelmode`[[link]](http://www.kernelmode.info/forum/viewtopic.php?f=16&t=3509)论坛上开辟并维护一个版块来讨论`Linux/XOR.DDoS`。最新事件也可以在我们的报告中找到->[[link]](http://blog.malwaremustdie.org/2015/06/mmd-0033-2015-linuxxorddos-infection_23.html)。该恶意软件总是试图连接并且会在受害者机器上安装`rootkit`。

我们可以在之前给出的IP地址列表中随意挑选一个来下载`Linux/XOR.DDoS`的`payload`。就像我下面做的，有点点暴力。(并没下载全部，只是演示)。

![enter image description here](http://drops.javaweb.org/uploads/images/624d3c29a2f1ee2768161f65911ace6557c92ba5.jpg)

下面的是收集到的payload。感谢伟大的团队合作。

![enter image description here](http://drops.javaweb.org/uploads/images/524c8d6cb9abcbb2517547bd92e72d6578bc9cc5.jpg)

可以发现有两种样本。一种是`unstripped`的，还有`encoded+stripped`的。我选择了两个来进行检测，hash分别是73fd29f4be88d5844cee0e845dbd3dc5和758a6c01402526188f3689bd527edf83。

样本73fd29f4be88d5844cee0e845dbd3dc5是典型的`XOR.DDoS ELF x32`变种，由以下的代码工程编译：

![enter image description here](http://drops.javaweb.org/uploads/images/ce2fd37beb9574a8448064927fccd5e0bd5062ad.jpg)

可以发现熟悉的代码段，看看下面的解密方法：

![enter image description here](http://drops.javaweb.org/uploads/images/274b3a00b6e11b0ee412b9d0825ae6b33513e13d.jpg)

看看完全解密后的结果：

![enter image description here](http://drops.javaweb.org/uploads/images/1822d2a2018d5b6b208a70d74d175aec6b3d51f4.jpg)

成功解密之后你就可以直接看到`Linux/XOR.DDoS`用作`CNC(command and control)`服务器的域名。在本例中是：`www1.gggatat456.com in 103.240.141.54`该域名在ENOM域名商注册，使用的受隐私保护的联系ID注册。

![enter image description here](http://drops.javaweb.org/uploads/images/0e5e13d623eb45aec334f4371adc31e7a65745bf.jpg)

```
Domain Name: GGGATAT456.COM
Registrar: ENOM, INC.
Registry Domain ID: 1915186707_DOMAIN_COM-VRSN
Sponsoring Registrar IANA ID: 48
Whois Server: whois.enom.com
Referral URL: http://www.enom.com
Name Server: DNS1.NAME-SERVICES.COM
Name Server: DNS2.NAME-SERVICES.COM
Name Server: DNS3.NAME-SERVICES.COM
Name Server: DNS4.NAME-SERVICES.COM
Name Server: DNS5.NAME-SERVICES.COM
Status: clientTransferProhibited http://www.icann.org/epp#clientTransferProhibited
Updated Date: 31-mar-2015
Creation Date: 31-mar-2015
Expiration Date: 31-mar-2016
Last update of whois database: Wed, 15 Jul 2015 00:59:06 GMT
Tech Email: QJJLPSYSWP@WHOISPRIVACYPROTECT.COM
Name Server: DNS1.NAME-SERVICES.COM
Name Server: DNS2.NAME-SERVICES.COM
Name Server: DNS3.NAME-SERVICES.COM
Name Server: DNS4.NAME-SERVICES.COM
Name Server: DNS5.NAME-SERVICES.COM

```

和之前的样本不同，我们分析这次捕获的样本发现，它与CNC服务器建立了ssh连接并下载加密数据：

![enter image description here](http://drops.javaweb.org/uploads/images/19822e846c4d22c205596aae2fd75bcfd4d200ad.jpg)

因为在此篇文章中我有很多需要解释的内容，该恶意软件已经在`kernelmode`论坛里讨论分析了很多次了，在之前的MMD博客中，和一些其他研究网站中也讨论了很多了，所以我不会在本文中涉及过多的细节。将关注重点主要放在恶意代码的解密和为了达到分解和停止目的而采取的策略。

hash为`758a6c01402526188f3689bd527edf83`的`g.rar`文件有一些不同。它是`ELF-stripped`二进制文件(仅仅使逆向工程增加了2%的难度)。可以通过`code-mapping`方法来解决逆向分析问题。

![enter image description here](http://drops.javaweb.org/uploads/images/eedcb16f88966ed261c44d8d52d0ad2ebf326ab0.jpg)

其中包含`deflate.c`代码(`ver 1.2.1.2`)[[link]](http://pastebin.com/mgQ8x66r)：

![enter image description here](http://drops.javaweb.org/uploads/images/1949209463867dd880caa49096cbac77753eeade.jpg)

用zlib压缩混淆一些文本值：

![enter image description here](http://drops.javaweb.org/uploads/images/c1870d625b3ede81ef9835304aa3eb7bbb54d68b.jpg)

这些安装时使用的字符串会带领我们找到CNC服务器

剩下的一些功能都是已知的，并没有什么新东西。结果看起来也使用了同样的映射方法。我推荐使用很不错的`pipe code`[[link]](http://www.zlib.net/zpipe.c)来解压这些文本值。

对剩下的二进制文件进行逆向，将会看到CNC主机的域名:`GroUndHog.MapSnode.CoM in 211.110.1.32`下面的图进一步证实了，该域名和我想的一样，和之前提到的IP地址一样是`Linux/XOR.DDoS`木马的一部分。

![enter image description here](http://drops.javaweb.org/uploads/images/ccc3b1df80040463401ce147bdf5e929391bd27d.jpg)

该域名是同一个人在ENOM注册的：

```
Domain Name: MAPSNODE.COM
Registrar: ENOM, INC.
Sponsoring Registrar IANA ID: 48
Whois Server: whois.enom.com
Referral URL: http://www.enom.com
Name Server: DNS1.NAME-SERVICES.COM
Name Server: DNS2.NAME-SERVICES.COM
Name Server: DNS3.NAME-SERVICES.COM
Name Server: DNS4.NAME-SERVICES.COM
Name Server: DNS5.NAME-SERVICES.COM
Status: clientTransferProhibited http://www.icann.org/epp#clientTransferProhibited
Updated Date: 11-may-2015
Creation Date: 11-may-2015
Expiration Date: 11-may-2016
Last update of whois database: Wed, 15 Jul 2015 07:16:23 GMT
Tech Email: RRRPBHYTFS@WHOISPRIVACYPROTECT.COM
Name Server: DNS1.NAME-SERVICES.COM
Name Server: DNS2.NAME-SERVICES.COM
Name Server: DNS3.NAME-SERVICES.COM
Name Server: DNS4.NAME-SERVICES.COM
Name Server: DNS5.NAME-SERVICES.COM

```

下面是更多关于shellshock和CNC使用的IP地址的细节：

```
{
  "ip": "43.255.188.2",
  "country": "HK",
  "loc": "22.2500,114.1667",
  "org": "AS134176 Heilongjiang Province hongyi xinxi technology limited"

  "ip": "103.20.195.254",
  "country": "HK",
  "loc": "22.2500,114.1667",
  "org": "AS3491 Beyond The Network America, Inc."

  "ip": "122.10.85.54",
  "country": "HK",
  "loc": "22.2500,114.1667",
  "org": "AS55933 Cloudie Limited"

  "ip": "211.110.1.32",
  "country": "KR",
  "loc": "37.5700,126.9800",
  "org": "AS9318 Hanaro Telecom Inc."
}

```

这些样本都已经在VirusTotal网站进行了检测，可以搜索下列hash：

```
MD5 (3503.rar) = 238ee6c5dd9a9ad3914edd062722ee50
MD5 (3504.rar) = 09489aa91b9b4b3c20eb71cd4ac96fe9
MD5 (3505.rar) = 5c5173b20c3fdde1a0f5a80722ea70a2
MD5 (3506.rar) = d9304156eb9a263e3d218adc20f71400
MD5 (3507.rar) = 3492562e7537a40976c7d27b4624b3b3
MD5 (3508.rar) = ba8cc765ea0564abf5be5f39df121b0b
MD5 (6000.rar) = 73fd29f4be88d5844cee0e845dbd3dc5
MD5 (6001.rar) = a5e15e3565219d28cdeec513036dcd53
MD5 (6002.rar) = fd908038fb6d7f42f08d54510342a3b7
MD5 (6003.rar) = ee5edcc4d824db63a8c8264a8631f067
MD5 (6004.rar) = 1aed11a0cbc2407af3ca7d25c855d9a5
MD5 (6005.rar) = 2edd464a8a6b49f1082ac3cc92747ba2
MD5 (g.rar) = 758a6c01402526188f3689bd527edf83

```

0x03 "Linux/killfile" ELF (downloader, kills processes & runs etc malware)
=====

我们在之前的样本分析中并没有见过这些功能，所以我需要解释一下。`Linux/XOR.DDoS`通过加密会话从CNC服务器下载其他的恶意文件。在CNC服务器上，有一系列的ELF恶意软件下载者程序，分别适用于被感染主机的不同系统环境。下面的ELF二进制文件(x32或x64)的其中之一将会在被感染的机器上运行，该ELF可执行文件是实现killfile功能的模块，可以下载对应配置文件来结束特定进程或者其他需要结束的恶意软件。它可以从配置的文本文件(`kill.txt`或`run.txt`)中读取出配置信息，这些信息使用`“|”`来逻辑分割`kill/run`功能，`killfile`功能模块可以对此配置进行逻辑解析并执行。

```
MD5 (killfile32) = e98b05b01df42d0e0b01b97386a562d7  15282 Apr  3  2014 killfile32*
MD5 (killfile64) = 57fdf267a0efd208eede8aa4fb2e1d91  20322 Apr  3  2014 killfile64

```

接下来说明几个重要的模块函数。

killfile模块用C编写，下面是源文件名：`'crtstuff.c' 'btv1.2.c' 'http_download.c' 'libsock.c'`

它将自己伪装成一个`bluetooth`进程：

![enter image description here](http://drops.javaweb.org/uploads/images/5e2401ea8a41238a2e5b6db0a1568b2a38bb233c.jpg)

也会伪装为Microsoft(试着读下面的代码，它是自解释的)：

![enter image description here](http://drops.javaweb.org/uploads/images/1ec0392aa9be7ce6243d408878ce088e68591cab.jpg)

在我分析的这前两个样本中，它们都会从下面的域名和IP地址下载进程列表并结束这些进程：`kill.et2046.com sb.et2046.com 115.23.172.31`

下图是对结束列表的逆向分析：

![enter image description here](http://drops.javaweb.org/uploads/images/a7f95cccb458bbbd5745ac5169c036980d0b8c6f.jpg)

这个IP地址位于韩国电信，看起来是被黑客控制的服务器：

```
{
  "ip": "115.23.172.31",
  "hostname": "kill.et2046.com",
  "city": Seoul,
  "country": "KR",
  "loc": "37.5700,126.9800",
  "org": "AS4766 Korea Telecom"
}

```

et2046.com应该也不是一个正常的域名，在GoDaddy注册，信息如下：

```
Domain Name: ET2046.COM
Registrar: GODADDY.COM, LLC
Sponsoring Registrar IANA ID: 146
Whois Server: whois.godaddy.com
Referral URL: http://registrar.godaddy.com
Name Server: A.DNSPOD.COM
Name Server: B.DNSPOD.COM
Name Server: C.DNSPOD.COM
Status: clientDeleteProhibited http://www.icann.org/epp#clientDeleteProhibited
Status: clientRenewProhibited http://www.icann.org/epp#clientRenewProhibited
Status: clientTransferProhibited http://www.icann.org/epp#clientTransferProhibited
Status: clientUpdateProhibited http://www.icann.org/epp#clientUpdateProhibited
Updated Date: 21-dec-2014
Creation Date: 27-nov-2012
Expiration Date: 27-nov-2016
Last update of whois database: Thu, 16 Jul 2015 22:17:47 GMT <<<

```

email地址为tuhao550@gmail.com:

```
Registry Registrant ID:
Registrant Name: smaina smaina
Registrant Organization:
Registrant Street: Beijing
Registrant City: Beijing
Registrant State/Province: Beijing
Registrant Postal Code: 100080
Registrant Country: China
Registrant Phone: +86.18622222222
Registrant Email: "tuhao550@gmail.com"

```

这域名和email地址都是我的朋友`Dr.DiMino`在`DeepEndResearch`[[link]](http://www.deependresearch.org/2015/02/linuxbackdoorxnote1-indicators.html)找到的。 以上域名的DNS解析如下：

![enter image description here](http://drops.javaweb.org/uploads/images/70e02fa26a3c9bcfad5a957ba2468c3202e1d26d.jpg)

重新回到`Linux/KillFile ELF`恶意软件的行为分析。它随后从之前的URL下载`run.txt`文件，下载执行该文件里面列出的其他恶意文件。在本样本中，run.txt里面的内容是一款臭名昭著ELF DDoS恶意软件 "`IptabLes|x`" 。难道，`Xor.DDoS`和`IptabLes|x`合作了。我想只有ChinaZ(译者：这个单词不知道如何准确翻译，难道是代表中国人)才会与`IptabLes|x`勾结。是否`IptabLes|x`在中国的恶意软件黑市上已经是开源的了？为什么近期那些危险分子开始变化他们的工具？这些答案将会在以后才能得到揭晓。

![enter image description here](http://drops.javaweb.org/uploads/images/1c4f19f6e555e723202c6f7d901bfca6cbdf5429.jpg)

该二进制文件是明文的，你可以清楚看见下载源地址，下载的文件和虚假的用户集(注意字符串：“`TencentTraveler`”)，这些信息可以迅速用来帮助抵御威胁：

![enter image description here](http://drops.javaweb.org/uploads/images/71d1b811b85b0c3ccecb9fde1a5bb4740057a105.jpg)

这前两个`Linux/KillFile ELF`恶意软件是在2014年编译的老版本，但是看起来已经参与了几次感染行动了。但是VT评分还是0.可以参考->[[-1-]](https://www.virustotal.com/en/file/35b2c1592fe394fc8d12d7a67fa54d5c7e7cf1b7ad3dc78e530d761628374006/analysis/1436962486/)和[[-2-]](https://www.virustotal.com/en/file/82ba1e7c02b91ee4298717f9a8ba20aae3063107c8d818463ceb5829e5746b48/analysis/1436962528/)。这次感染中，攻击者还准备了另外一个 Linux/KillFile。我们将在下一节深入解析中来说明。

我们在kernelmode论坛添加了新的ELF Linux/killfile讨论帖： [http://www.kernelmode.info/forum/viewtopic.php?f=16&t=3944&p=26309#p26309]

0x04 深入解析(Under the hood)
=====

也许你读过很多次与本文之前讨论相似的报告。也许你已经对这类恶意软件，CNC服务器，IP地址，DDoS等重复的故事感到厌烦。但是这次，我们带来的深入解析将是全新的。先说明，所有的内容已经由HTTP协议带到你面前，通过浏览器展示给你，其中没有黑客攻击细节，没有攻击的方法描述，所以如果你期待的比这更多，我只能说声抱歉。所有这些都是由我们ELF Team的团队合作努力完成的。

我们已经被`Linux/XOR.DDoS`攻击了多次，都还不知道他们到底是谁。大家都充满了好奇，所以我们随机的选择可以接触到的IP地址，尝试看看通过合法手段能不能有什么收获。从上面给出的IP的其中一个，我们拿到了被加密了的特殊文件：

![enter image description here](http://drops.javaweb.org/uploads/images/d8af4e1a60fac1d5c342babaf7f133055dec8fad.jpg)

部分文件被制作者加密了，但是是可以破解的:-)。我们不会在这里披露密码。因为它里面包含了rootkits，漏洞扫描，CNC程序，ELF病毒和木马(下载者和回连)。一系列攻击者用来攻击我们的工具。这些工具依然在使用：

0x05 RDP(Remote Desktop Protocol)扫描工具集rdp.rar
=====

这是一个实实在在的`ELF RDP`扫描工具集，扫描引擎(`rdp ELF`可执行文件)用来扫描指定范围的IP段，探测开启了RDP的主机，发现了之后使用字典进行暴力攻击。以得到用户名和密码。可以设置扫描的线程数和发现匹配用户后终止扫描的条件。可以明显的看出来，工具代码的编写者不是讲英语地区的人。这种情况我们在windows平台上见太多了。我们认为这个ELF版本的工具只是被限制给少数人使用。下图是该工具的界面截图：

![enter image description here](http://drops.javaweb.org/uploads/images/33a30565e5da3c902ff250898f7df598d1627701.jpg)

rdp文件是整个rdp工具集的核心，为了能够实现自动化攻击，作者使用了脚本。我们发现了shell脚本“a”和“start”来实现自动化。下图给出具体脚本代码片段，请注意，如果没有rdp程序，这些脚本就什么用都没有。

![enter image description here](http://drops.javaweb.org/uploads/images/d105b36ec5e578f855ba8583d334b6ed957fed5f.jpg)

`Linux/XOR.DDoS`使用者在工具集里的文本文件中留下一些数据，你可以看到这些数据包含用来暴力攻击的用户名和单词库。还有一些扫描结果保存在`vuln.txt`文件中：

![enter image description here](http://drops.javaweb.org/uploads/images/d2949227417061524a53a97cf89a69fa81ba53bf.jpg)

我想上图非常清楚的说明了rdp能干些什么和它是怎么被`Linux/XOR.DDoS`攻击者所使用的，为了他们的黑客行动，用rdp来暴力破解windows网络。所以我就不做视频了。这个`rdp ELF`是个新发现，我们将它命名为`Linux/rdp`，作为一款ELF黑客工具。同时我们在`kernelmode`论坛上开一个版块来讨论它。

这就是`rdp.rar`的全部？不，这里还有个名叫psc的家伙，它也是一个ELF可执行文件，细节如下：

```
16840 Nov 24 psc fcd078dc4cec1c91ac0a9a2e2bc5df25
psc: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), 
     dynamically linked (uses shared libs), for GNU/Linux 2.2.5, not stripped

```

如果你将psc放到`Virus Total`检测，将会得到(37/56)的`Linux ELF`病毒感染的检测结果。被判定为一种已知的恶意软件`Linux/RST`变种。该软件可以感染其它的ELF文件，然后留下一个后门[[link]](https://www.symantec.com/security_response/writeup.jsp?docid=2004-052312-2729-99)。是不是很坏。

但是psc其实是一款名为pscan的黑客工具。是用来进行端口扫描的。黑帽子使用它来扫描服务器的SSH端口，我也知道有白帽子拿它来进行渗透测试。MMD博客曾经讨论过黑客工具，参见MMD-0023-2014[[link]](http://blog.malwaremustdie.org/2014/05/a-payback-to-ssh-bruting-crooks.html)。

我们测试运行了它，发现和pscan是一样的。应该是`Linux/XOR.DDoS`的攻击者使用psc缩写来命名。

![enter image description here](http://drops.javaweb.org/uploads/images/5438adcffa9d17d41dd510e3d109ff6014c69f36.jpg)

下图显示了psc里面有很多的字符串包含“pscan”字符串：

![enter image description here](http://drops.javaweb.org/uploads/images/b6cde8bcaf4be01141fd7571cbe5a781b6ff909c.jpg)

然而，为什么VT报告它被病毒感染了呢？谢谢`Miroslav Legen`的建议，我检查了程序入口点。程序原始入口点被修改为`0x08049364`，恶意入口点取代了原始入口点(`0x080487a8`)，这么做的目的就是运行提前运行恶意指令，然后再跳转到原始入口点。下图是我在检查入口点替换时的逆向笔记：

![enter image description here](http://drops.javaweb.org/uploads/images/9878df98f143678be1fae870b8b2f0bcf503944a.jpg)

进一步还发现，`rdp ELF`二进制文件也进行了入口点替换：

![enter image description here](http://drops.javaweb.org/uploads/images/761c3bf85f3bc131c1d16167c9bff52753477436.jpg)

事实上，我们发现IP位于中国主机上的恶意软件集和黑客工具都被另一种恶意软件所感染。所以本文介绍的案例只是其中一个，毕竟，它本来就是来自一个不可信的环境。所以我们强烈建议不要运行任何从不可信环境中的到的任何东西。我不敢肯定是否`Linux/XOR.DDoS`使用者知道这情况或者是故意将被感染的ELF黑客工具放进他们的压缩包里(呵呵，我想他们应该是不知道，因为他们给压缩包设置了密码)，但是我真心希望他们自己被这病毒感染:D

0x06 XOR.DDoS CNC服务器中的.lptabLes僵尸网络集
=====

在job.rar压缩包中，攻击者将攻击我们网络所需的各种工具都打包进了这一个压缩包里面。它里面包含下载模块`killfile`和用于结束进程的`kill.txt`，用于下载启动文件`run.txt`。我们已经在前面的章节中介绍过这个模块化的`killfile ELF`模块。

还有一个`rootkit`，我将会在最后一节中用一个视频来对代码进行说明。但是现在我想指出的是：在job.rar压缩包中有一个.lptabLes|x客户端(ELF)和它的服务端，僵尸网络CNC服务器。job.rar包确实需要引起注意：

![enter image description here](http://drops.javaweb.org/uploads/images/a889f3d59d4a08e7580def6cf65a21102f71ac89.jpg)

该`.lptabLes|x ELF`客户端二进制文件(ELF僵尸网络恶意软件)我们已经在之前的文档中分析过-->[[-1-]](http://blog.malwaremustdie.org/2015/06/mmd-0035-2015-iptablex-or-iptables-on.html)和[[-2-]](http://blog.malwaremustdie.org/2014/06/mmd-0025-2014-itw-infection-of-elf.html)。这些二进制文件没有任何新东西，只是用来连接到CNC僵尸网络(Windows PE)软件运行的IP地址。我们直接将注意力集中到僵尸网络CNC软件上来。

这个二进制文件是用.net编写的。我们已经将其上传到Virus Total上[[link]](https://www.virustotal.com/en/file/37e6f378156bfeb5a95fe396baf97331bb547a63aa712a9931a9dcc4bde0d2fe/analysis/)。我制作了一个视频，以更加直观的方式介绍CNC僵尸网络工具是怎么样工作的：

为了了解僵尸网络CNC工具的细节，可以通过阅读我们制作的逆向源代码来学习，我们对代码进行了处理(为了让代码不能被重用，但是很容易阅读)。通过下面链接可以获得： https://pastebin.com/xDCipBY1

下面是 IptabLes|x 客户端和CNC 僵尸网络工具的MD5列表：

```
MD5 (777.hb) = "e2a9b9fc7d5e44ea91a2242027c2f725"
MD5 (888.hb) = "ff1a6cc1e22c64270c9b24d466b88540"
MD5 (901.hb) = "c0233fc30df55334f36123ca0c4d4adf"
MD5 (903.hb) = "f240b3494771008a1271538798e6c799"
MD5 (905.hb) = "603f16c558fed2ea2a6d0cce82c3ba3a"
MD5 (Control.exe) = "315d102f1f6b3c6298f6df31daf03dcd"

```

![enter image description here](http://drops.javaweb.org/uploads/images/c5f91dffe2536a90c9098d60bc134ac5062a33c1.jpg)

0x07 “Linux/KillFile”工具集：xxz.rar,kill.txt,run.txt
=====

在job.rar[[link]](https://lh3.googleusercontent.com/-EWCmW0g-Bls/VadPw7kKL4I/AAAAAAAASkY/E-RoEMDbsO8/s2720-Ic42/034_phixr.png)中,有一个`Linux/KillFile`恶意软件，伪装成为一个`xxz.rar`的压缩文件。`Linux/KillFile`的工作逻辑和之前章节介绍的一模一样[[link]](http://blog.malwaremustdie.org/2015/07/mmd-0037-2015-bad-shellshock.html),区别在于，之前的两个样本都是执行其他与感染无关的功能，但是这个是为了感染而生的。它所扮演的角色就是下载和安装前面介绍的`Linux/IptabLes|x`僵尸网络客户端。

这个版本的`Linux/KillFile`，当执行了下载的文件之后，它使用虚假的Microsoft版本信息来进行伪装。

![enter image description here](http://drops.javaweb.org/uploads/images/7e8702bb369a43bfd11a53eb74fd0e9c405a2f11.jpg)

另外一个大的不同是，与host连接时的数据和从远程主机获取的数据：

![enter image description here](http://drops.javaweb.org/uploads/images/26927ae9c4c51d5198a271fdb484aeac06b139f5.jpg)

我们据此获得新的ip：61.33.28.194和115.23.172.47，同样的都是位于韩国：

```
{
  "ip": "61.33.28.194",
  "hostname": "No Hostname",
  "city": null,
  "country": "KR",
  "loc": "37.5700,126.9800",
  "org": "AS3786 LG DACOM Corporation"
}

```

我非常肯定`et2046.com`域名是被控制的，如果是攻击者控制的，下面的按时间序排列的IP数据连接到et2046.com，一定就会连接到攻击者。

```
115.23.172.31 (current) 
115.23.172.6 (May 2014) 
115.23.172.47 (current)

```

我很奇怪的是为什么使用了这么多的韩国IP地址，韩国的网络执法部门需要对此引起重视。

0x08 “xwsniff rootkit”源代码
=====

本节中，我制作了一个视频来讲解`xwsniff rootkit`源代码，展示了所有的源码，不方便用语言表示的。这就是一份网络犯罪的证据。该rootkit在CNC集里面的多个地方被找到。包括在针对目标的压缩包job.rar里。不用怀疑的是，其中一个功能就是获得被感染服务器的root权限。这对读者来说也是学习了解rootkit最安全的方法，为了在以后能更好的防护我们自己。一样的，我们也对源代码进行了处理来进行保护，防止被坏人利用。

这份xwsniff rootkit安装包里面，有FTP守护进程(现在我们知道了为什么他们要停止ftp PIDl )，OpenSSH和PAM源代码，加上`stealth rootkit`的一部分，通过编译结合在一起，并拥有自己的内核代码。被感染的NIX系统将会受到极大的破坏，没有手工清除该rootkit的方法。所以我建议读者重装。这个rootkit是为linux系统设计的，但是经过少量修改就可以用于所有NIX系统。

为了实现功能，它拥有威力强大的函数，能让受害者无法知道是不是被感染了，在shell里面的安插后门和通过http下载，只需要添加几个函数。好了，说的太多了，我们直接看视频吧。

0x09 CNC僵尸网络的基础资源
=====

下面的IP地址都是用来实施感染的：

```
"43.255.188.2" (shellshock landing)
"103.20.195.254" (shellshock landing)
"122.10.85.54"  (shellshock landing)
"103.240.141.54" (Xor.DDoS CNC server)
"211.110.1.32" (Xor.DDoS CNC server)
"115.23.172.31" (.IptabLes|x download server)
"115.23.172.47" (.IptabLes|x download server)
"61.33.28.194" (.IptabLes|x download server)
"115.23.172.6" (Iptables|x previous IP record)

```

他们的地址：

![enter image description here](http://drops.javaweb.org/uploads/images/7b96f0c3a90a07b8c4feb56ff01a835fe83208b9.jpg)

下面是用于感染的主机域名：

```
kill.et2046.com
sb.et2046.com
www1.gggatat456.com
GroUndHog.MapSnode.CoM

```

0x0a 下一步打算
=====

对于这次事件，还有很多工作需要做。举个例子，我们上传所有样本到VT，包括`killfile，XOR.DDoS`下载者ELF模块。但是我们不分享rootkit，除了反病毒公司和执法部门。因为他是危险的工具。请阅读我们的法律声明-->[link](http://blog.malwaremustdie.org/p/the-rule-to-share-malicious-codes-we.html)。请给我们时间来准备新的分享。在下面的评论区表达你们的希望与要求，别忘了附上你完整的email地址。

我们会在kernelmode论坛分享样本，只为了研究目的。同样上传给`VirusTotal`。`Linux/KillFile`已经上传到kernelmode[[link]](http://www.kernelmode.info/forum/viewtopic.php?f=16&t=3944)和VT[[link]](https://www.virustotal.com/en/file/6e5dcfbc9d8368470bd846b6f1cf4faed9aff64ff2fdf8aa023425cb6ac5d535/analysis/1437120536/)。`.IptabLes|x botnet CNC tool WInPE(.NET)`在kernelmode做了限制性分享[[link]](http://www.kernelmode.info/forum/viewtopic.php?f=16&t=3468&p=26320)。其他的都上传到VT但是没有分享到kernelmode，因为都感染了其他病毒。 Rootkie源代码从20/07/2015开始分享，现在开始接受请求。

推特账号： @MalwareMustDie

* * *

感谢原作者辛勤的工作！ 结尾一段，与正文关系不大，略去没翻译。 本人水平有限，错误难免，还请指出！拜谢！