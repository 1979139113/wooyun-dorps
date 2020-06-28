# DUSTSTORM

[https://www.cylance.com/hubfs/2015_cylance_website/assets/operation-dust-storm/Op_Dust_Storm_Report.pdf?t=1456259131512](https://www.cylance.com/hubfs/2015_cylance_website/assets/operation-dust-storm/Op_Dust_Storm_Report.pdf?t=1456259131512)

Cylance SPEAR发现了一起针对日本、韩国、美国、欧洲以及其他几个东南亚国家的威胁行动，在上述国家中，有大量的行业部门都遭到了攻击。

0x00 多样的权利形式
=====

我们研究发现DustStorm最早从2010年开始活动，使用了大量不同的作战技术，包括钓鱼、水坑攻击和0-day漏洞。最初，有几家杀毒公司还能检测到他们使用的一些早期木马样本-被标记为Misdat，但是，经过多年的发展，现在已经很难再检测到他们的踪迹。

攻击数据表明，DustStorm小组不再像以前一样收集政府和国防情报，而是更加集中的寻找与日本关键基础设施和资源相关的组织机构信息。

近期，这个攻击小组大范围攻击了下面的这些行业：电力，石油和天然气，金融，运输和建筑。SPEAR研究发现，DustStorm小组当前的唯一关注点是日本企业和大型的日本外资机构。

0x01 前期：钓鱼
=====

与DustStorm相关的活动信息，最早是根据Misdat样本中可执行资源部分的编译时间确定的。所有的早期后门都是使用某个版本的Delphi编译的，这个版本的Delphi会把编译时间戳修改成June 19, 1992 22:22:17 UTC。利用可执行资源部分的时间戳，SPEAR准确的推测除了这些样本的真正编译时间，并跟踪了其中的一个样本”bc3b36474c24edca4f063161b25bfe0c90b378b9c19c”，直到2010年1月。

在2010年的时候，虽然DustStorm针对亚洲的攻击活动引起了一定的关注，但是，与这个威胁相关的公共信息仍然非常少。可能的解释是因为他们一开始大量使用了动态DNS域名来部署其CC服务器，以及使用了像Poison Ivy和Gh0st RAT这样的公共RAT作为第二阶段的植入木马。一直到，这些攻击者还在大量使用免费的动态DNS服务来部署其服务器，包括No-IP ([http://www.noip.com](http://www.noip.com/))，Oray (<http:/ www.oray.com/>) 和 3322 ([http://www.pubyun.com/](http://www.pubyun.com/))；SPEAR最早发现的一些后门就会与 “323332.3322.org” 和“1stone.zapto.org”通讯。

直到2011年，DustStorm开始利用未修复的IE 8漏洞（CVE-2011-1255）来入侵受害者的网络。在发动攻击时，攻击者会在钓鱼邮件中添加一个漏洞链接，这封邮件会装成来自某国的一名学生，向目标寻求建议或解答问题。

Symantec在报告中也提到了，攻击者使用的次级CC服务器是“honeywells.tk”；这个域名在2011年6月之前，会解析到“111.1.1.66”。这个地址也是第一个Misdat样本在同一时间范围内使用的一个地址。

在2011年4月的一次攻击活动中，攻击者在钓鱼邮件中添加了一个Word文档，而这个文档中内嵌了一个Flash 0-day漏洞（CVE-2011-0611）。其中使用的最终有效载荷经过证实就是Misdat样本，这个样本会连接“msejake.7766. org”，在攻击过程中，这个域名首先解析到了“125.46.42.221”，随后又解析到了“218.106.246.220”。

根据其他报告的描述，攻击者在入侵了目标机器后的几分钟内，就会开始手动枚举目标网络以及网络中的主机。

在2011年10月，DustStorm小组借助利比亚危机，开始钓鱼攻击在2011年10月20日报道了卡扎菲死亡消息的媒体。似乎除了美国国防目标，这次活动还攻击了一些维吾尔人。这次，DustStorm小组使用了一个经过定制的恶意Windows Help文件（.hlp），利用的漏洞是CVE- 2010-1885。这个hlp文件在打开时，会通过“mshta.exe”执行一些JavaScript代码，从而使用Windows脚本程序来启动第二部分Visual Basic Script。然后，VBS代码就会负责解码hlp文件中的有效载荷并执行。

在这些攻击中，使用的第一阶段有效载荷是Misdat变种，以Base64编码的方式储存在hlp文件中。SPEAR识别出的样本会与“msevpn.3322.org”通讯，当时这个域名使用的IP地址是“218.106.246.195”。通过调查这个IP地址，又发现了其他几个用作CC服务器的动态DNS域名，以及攻击者在2010年5月-2015年12月期间使用的几个标准域名。

![p1](http://drops.javaweb.org/uploads/images/df2297d02d70d520216744dc5227d6c4b0c02dde.jpg)

图1-2011-2011年的域名注册

在2010-2011年期间，攻击者主要使用了"wkymyx (at) 126.com” 和 “duomanmvp (at) 126.com”这两个邮箱来注册域名。

这些攻击者要么使用了随机的4字符子域名，要么就是使用了一些常用词，比如image, blog, ssl, pic, mail, news等。有证据表明，在2011年7-8月，这个小组通过钓鱼域名尝试了收集Yahoo，Windows Live以及其他的用户凭证。

虽然SPEAR无法获取到最初的钓鱼页，但是这些钓鱼页的域名包括：“login.live.adobekr.com”，“login.live.wih365.com”和“yahoomail.adobeus.com”。每个域名使用的IP地址都是经常变化的，没有一个IP能用到一个月以上的时间。

0x02 身份危机：0-day攻击
=====

SPEAR发现了另一起在2012年6月实施的DustStorm行动，这次行动利用了先前曾经使用过的Flash 0-day CVE-2011-0611和一个IE 0-day CVE- 2012-1889。攻击者使用了域名“mail.glkjcorp.com”来投放这些漏洞，并且在当时，这个域名会解析到“114.108.150.38”。SPEAR无法确定这个漏洞网站具体属于哪一次的水坑攻击或钓鱼活动，但是，其他的一些XX-APT小组也在相同时间使用这两种技术利用了IE漏洞。漏洞域名 “glkjcorp.com”是在2012年5月25日的一起攻击活动之前创建的。在注册这个域名时，使用了两个不同的邮箱“effort09 (at) hotmail.com” 和“zaizhong16 (at) 126.com”。

这次攻击率先使用了文件“DeployJava. js”来获取受害者系统上已安装软件的指纹，然后，才会部署一个有效的漏洞。“DeployJava.js”会与漏洞页上内嵌的另一个脚本配合使用，如果是IE8或9，会投放Flash漏洞；如果是IE6或7，则投放IE 0-day。

![p2](http://drops.javaweb.org/uploads/images/c46aa2c584aa8164ab792e1ab3da141f9c904716.jpg)

图2-选择漏洞投放时的JS代码段

在2012和2013年，主要是其他APT小组在使用“DeployJava.js”脚本，Nitro小组在同年8月也使用过。

![p3](http://drops.javaweb.org/uploads/images/fb690e1f73d50f7c4848fd4ecd03b6bc9d097484.jpg)

图3-其他利用了“DeployJava.js”的针对性攻击

另外还要注意的是，在这次攻击中：最终的有效载荷(`hxxp:/mail.glkjcorp.com/pic/win.exe`)使用了一个单字节对0x95个字节进行了异或，并且没有异或秘钥本身和0，这样做是为了避免暴露秘钥。在当时，这种混淆方法能够保证有效载荷通过大多数IDS/IPS系统。未经过编码的有效载荷是一个旧版Misdat后门和下一代后门-S型后门的结合体。这个后门首先会尝试使用旧版的Misdat网络协议与 “smtp. adobekr.com”通讯。如果失败，则会使用新型的基于HTTP的S型协议与“mail.glkcorp.com”通讯。

在2013年时，DustStorm完全放弃了旧版的Misdat后门作为第一阶段的植入木马，并转向主要使用S型后门。

0x03 未来预测：日本目标
=====

SPEAR注意到，DustStorm在2013年3月-2013年8月的活动数量出现了大幅下降。巧合的是（也许不是巧合），Mandiant在2013年2月19日公布了APT 1报告([https://www.fireeye.com/blog/threat-research/2013/02/mandiant-exposes-apt1-chinas-cyber-espionage-units.html](https://www.fireeye.com/blog/threat-research/2013/02/mandiant-exposes-apt1-chinas-cyber-espionage-units.html))。虽然DustStorm的活动并没有完全停止，但是在这段时间中，SPEAR能够收集到的木马数量也在大幅减少。

在这段时间中，注册了几个新的域名，这些域名可能是为了接下来几年的行动做准备。

![p4](http://drops.javaweb.org/uploads/images/023a0983faba5fc187df097ad07345e2bf43db91.jpg)

图4-在2013年新注册的C2域名

有一些没有经过证实的证据表明，DustStorm利用了一个Ichitaro 0-day “CVE-2013- 5990”来攻击日本的受害者。这个0-day最早是在2013年11月12日公开报道 过。Ichitaro是由JustSystems开发的一个日文处理程序。

在整个2013年中，SPEAR发现所有的攻击活动中都部署的是S型后门。今年，攻击者还使用了两个位置来保证木马的持久性，这样是为了防止受害者的权限不够，无法执行某些特定的操作或无法访问特定的文件位置，比如写入注册表。在这段时间中，一些老技术，像使用“Startup”文件夹再次得到了启用。

2014年2月初开始，有确切的证据表明，DustStorm小组水坑攻击了一家常用软件销售商，旨在向目标投放IE 0-day CVE-2014-0322。这个漏洞本身托管在“hxxp:/krtzkj.bz.tao123.biz/error/pic.html”，当时，这个域名解析到了“126.85.184.190”。在同一时间，域名“js.amazonwikis.com”也指向了这个IP，并且利用了此前的一个web漏洞。有效载荷“Erido.jpg”是一个经过异或的可执行程序，与其他利用CVE-2014-0322的攻击类似，最终投放的还是S型后门。

DustStorm在2014年开始尝试通过其他方式来维持受害系统上的木马。SPEAR发现了几个第二阶段样本需要作为ServiceDLL安装才能保证正常的运行，有的也会安装为路由和路由访问服务的路由器管理器。只需要简单的搜索注册表键值“`HKLM\System\CurrentControlSet\ Services\RemoteAccess\RouterManagers\IP\DllPath`”就能找到大量其他的木马；但是，SPEAR只发现了一个样本利用了这种技术。攻击者还在2014年新注册了几个域名来支持行动扩张。

![p5](http://drops.javaweb.org/uploads/images/c02efe41dee4d16c389bb57154a2994753236026.jpg)

图5-在2014和2015年新注册的C2域名

0x04 现状：遭到入侵的企业
=====

**DustStorm**在2015年的活动更加有趣。SPEAR发现了大量的二阶段后门都使用了硬编码的代理地址和凭证。这些代理地址表明，攻击者入侵了大量的日本企业，涉及到能源业，石油与天然气，建筑，金融和交通业。

在2015年2月初的一个案例中，SPEAR恢复了一个S型后门变种投放的二阶段植入。

这个二阶段植入也是使用Microsoft Visual Studio 6编译的，似乎木马作者很喜欢这个版本的Visual Studio。尽管使用了旧版的Visual Studio，这个后门的设计仍然很出色，并且提供了攻击者需要的所有功能。

似乎没有任何杀毒厂商能够稳定的检测出**SPEAR**发现的这个变种。

更有趣的是，在2015年初，这个小组采用并最终定制了几个Android后门来实现他们的目的。到2015年3月，DustStorm迅速扩张了他们针对移动端的行动。最开始的几个后门都比较简单，并且能够将所有的SMS信息和通话信息发动给CC服务器。随后的一些版本变得更加复杂，并且新增了直接从受感染设备上枚举和传输文件的功能。

Android木马的所有攻击目标都来自日本或韩国。和先前的活动相比，用于支持Android端活动的基础设施具有更大的规模。目前为止，已经确定了有200个域名。

SPEAR发现了另外两拨攻击活动，分别开始于2015年7月和10月。有趣的是，其中的一个主要目标是一家韩国电力公司在日本设立的子公司。另外，SPEAR还发现攻击者单独入侵了一家日本石油天然气公司。我们并不清楚其目的，但是，如果和以前一样，可能是为了勘察和长期的间谍活动。

0x05 总结
=====

目前，SPEAR认为这些攻击活动的破坏性并不是很强。但是，我们认为这些攻击活动在将来还会以日本的关键基础设施和资源作为目标。

注：大量早期使用的_Misdat_域名已经在_2015_年_12_月末放入污水池了。目前，这些域名都指向了**IP地址“58.158.177.102”**。

0x06 植入木马分析
=====

### MISDAT 后门 (2010-2011)

最早的Misdat样本是没有封装的；但是，接下来就值得安全厂商注意了，在2010年末和2011年出现的样本都使用UPX version 3.03(hxxp:/upx.sourceforge.net/)进行了封装。SPEAR识别出的所有Misdat样本都是使用Borland Delphi编写的，Borland Delphi会修改默认的PE时间戳；所以，SPEAR不得不利用样本的资源编译时间来推测后门的真实编译时间。

**文件特征**

![p6](http://drops.javaweb.org/uploads/images/71b7d22640050bbd32629bc47920351384587ead.jpg)

已知Misdat样本的文件类型和资源编译时间

**主机标识**

**证据**

*   会根据卷序列号的MD5哈希，解密后的网络配置数据和一个经过编码的活动标识符，创建一个32位的互斥量

**文件系统修改：**

*   后门会把自己复制到`%CommonFiles%\{Unique Identifier}\msdtc.exe`
*   取决于样本，可能会打开文件`C:\2.hiv`,`c:\t2svzmp.kbp`或`c:\tmp.kbm`；然后删除。2011年以后的版本都使用了`c:\t2svzmp.kbp`

**注册表修改**

*   木马可能会创建注册表键值`HKCU\Software\dnimtsoleht\StubPath`,`HKCU\Software\snimtsOleht\StubPath`或`HKCU\Software\Backtsaleht\StubPath`来作为维持机制
*   在SPEAR的测试中，StubPath总是指向在`%CommonFiles%`目录下新创建的msdtc.exe二进制，根据不同的样本，可能使用“`/ok`”或“`/start`” 来启动
*   可能创建注册表键值`HKLM\SOFTWARE\Microsoft\Active Setup\Installed. Components\{3bf41072-b2b1-21c8- b5c1-bd56d32fbda7}`或`HKLM\SOFTWARE\Microsoft\Active Setup\Installed Components\{3ef41072-a2f1-21c8-c5c1- 70c2c3bc7905}`

**网络标识**

网络流量一直是base 64加密的明文，通过一个原始socket发送到常用端口上，比如80，443或1433。下面是一个初始数据包。

```
logon|{Hostname}|Windows XP|100112|bd56d32fbda703a98c87689c92325d90|

```

图6-在Base64解码后的初始beacon数据包

“logon”字符串永远会排在其他信息的前面。在上面的这个例子中，受害系统的主机名，操作系统版本，样本标示符（SPEAR 认为这是一个日期：1/12/2010），以及互斥量使用的MD5都会发送给服务器。一旦注册到C2，后门就会发送字符串“`YWN0aXZlfA==`”，在解码后就是 “`active|`”。然后，后门会继续发送这个字符串，直到接收到C2发出的下面某条命令。

![p7](http://drops.javaweb.org/uploads/images/10f4d112da7722d5fdcbe9003f4d5e9c8c105d01.jpg)

图7-Misdat后门支持的网络命令

**详情**

这些后门相对来说比较简单，并且能够上传和下载文件，操作和枚举文件，执行shell命令，断开与C2的连接，卸载后门，关闭或重启系统。这些后门还可以获取命令行参数“`/ok`” 或 “`/start`”；这些参数开关能够更改进程运行的用户环境。如果在执行时没有提供开关，后门就会把自己复制到“`%CommonFiles%\ {Unique Identifier}\msdtc.exe`”，其中的标识符就是作为互斥量使用的MD5哈希的前十个字符。然后，后门会配置Active设置以及相关的注册表键值来实现维持机制。

SPEAR逆向了用于混淆网络回调信息的编码机制和活动标识符。下面的脚本可以用于解码这些经过混淆的字符串。

![p8](http://drops.javaweb.org/uploads/images/ed1afb09b1243820dd815ac25c1b5581266ee61e.jpg)

图8-用于解码Misdat字符串的Python脚本

**文件特征**

![p9](http://drops.javaweb.org/uploads/images/923bbe9c994cb0b694b5c37967424c7bb18db0c8.jpg)

图9-解码后的活动标识符和网络回调

另外，所有的样本都会尝试调用Windows API “GetKeyboardType”来 检测受害者使用的是不是日文键盘，并报告给攻击者。

### MIS类型的混合后门

在2012年，DustStorm逐渐开始使用一种混合型后门，这种后门的同一个二进制中实际上包含了两个独立的后门。这种后门首先会使用base64编码的网络协议，通过一个原始TCP socket，尝试创建一个交互式shell。如果连接第一个C2失败，后门就会尝试通过一种基于HTTP的协议联系一个次级C2，并与这个C2通讯。SPEAR识别的这个混合型变种使用了UPX version 3.03进行压缩。

**文件特征**

![p10](http://static.wooyun.org//drops/20160229/2016022914542075177106.png)
===========================================================================

图10-混合后门的文件详情

**主机标识**

**证据：**

*   会根据卷序列号，解密后的网络配置数据，经过编码的网络配置数据和经过编码的活动标识符，创建一个32位的互斥量
*   可能会在系统上创建一个临时用户，名称：“`Lost_{Unique Identifier}`”，密码：“`fuck~!@6”{Unique Identifier}`”
*   可能临时创建文件夹`%System%\{Unique Identifier}`
*   可能在`%AppData%\{Unique Identifier}`中创建以“tmp.exe”结尾的文件
*   可能创建文件
*   `%AppData%\{Unique Identifier}\HOSTRURKLSR`
*   包含有 “`cmd.exe /c ipconfig /all`”命令的运行结果
*   `%AppData%\{Unique Identifier}\NEWERSSEMP`
*   包含有“`cmd.exe /c net user {Username}`”命令的运行结果

**文件系统修改：**

*   后门会把自己复制到`%AppData%\{Unique Identifier}\msdtc.exe`– 其中的标示符是MD5哈希的前十个字符

**注册表修改：**

*   木马会创建注册表键值`HKCU\Software\bkfouerioyou`
*   创建指向`%AppData%\{Unique Identifier}\msdtc.exe`的StubPath值
*   会创建其中一个注册表键值作为维持机制：
*   `HKLM\SOFTWARE\Microsoft\Active Setup\Installed Components\{6afa8072-b2b1-31a8-b5c1-`
*   `{Unique Identifier} – First 12bytes`
*   `HKLM\SOFTWARE\Microsoft\Active Setup\Installed Components\{3BF41072-B2B1-31A8-B5C1-`
*   `{Unique Identifier} – First 12bytes`

**网络标识**

木马会DNS请求域名“smtp.adobekr.com” 和 “mail.glkjcorp.com” 或 “auto.glkjcorp.com”。这两个样本都会首先使用Misdat网络协议，通过TCP端口80，443和25来与“smtp.adobekr.com”通讯。如果没有接收到C2响应，样本会使用次级HTTP请求通过相同的TCP端口与备用C2通讯。

![p11](http://drops.javaweb.org/uploads/images/95fadad1767ae70dba3098e955da5b1761d268f9.jpg)

图11-初始的S型Beacon

初始的POST请求总是会使用静态的User-Agent “FirefoxApp”并包含有操作系统信息，用户信息，几个权限测试结果，文件系统和文件运行时间。如果后门没有接收到响应，然后会尝试通过一个GET请求在URI中传输经过base64加密的信息。

![p12](http://drops.javaweb.org/uploads/images/9bcf61a9d8bd4a191eb79c71c72a4d31222132eb.jpg)

图12-从上图中，解码后的POST请求

接下来的请求会使用系统上默认浏览器的User-Agent，如下。

![p13](http://drops.javaweb.org/uploads/images/0a59c2f5cc59e3a726aa67ceb96b0eb40267e86d.jpg)

图13-后续的HTTP流量

上图中的id和mmid字段都使用了为互斥量创建的标识符的前16个字符。Misdat协议能够为攻击者提供一个功能完整的后门，而次级协议似乎主要是用作更新机制来加载木马上的其他木马。

**网络标识**

这个后门可以通过3个不同的参数开关来启动：“/ok”，“/Start”或 “/fuck”。这些开关影响了进程的运行上下文，以及二进制在执行后是否要立即删除。

![p14](http://drops.javaweb.org/uploads/images/3a72ca99523720991765d46b1e908cf65821a84d.jpg)

图14-后门使用的命令行执行开关

后门会尝试运行大量的测试来判断受害者用户的权限级别，包括能否向系统中添加用户，能否在%System%文件夹中创建一个目录，以及用户能否通过调用“OpenSCManagerA”来访问服务管理器。

用户权限测试时通过利用NetUserAdd和NetUserDel Windows API执行的；这些测试会尝试创建临时用户“`Lost_{Unique Identifier}`”，使用密码“`fuck~!@6{Unique Identifier}`”。如果激活了次级网络协议，后门还会通过命令编译器执行两个命令来收集系统信息：“`cmd.exe /c ipconfig /all`” 和 “`cmd.exe /c net user {Username}`”。后门会临时把这些命令的输出结果写入到文件中，分别是“`%AppData%\{Unique Identifier}\HOSTRURKLSR`”和“`%AppData%\{Unique Identifier}\NEWERSSEMP`”。然后，这些信息会经过base64加密，并通过GET请求在URI中传输给C2服务器。下面更详细介绍了S型网络协议。另外还值得注意的是，即使能够与次级C2建立连接，这个后门会继续尝试连接端口25上的“smtp.adobekr.com”。

在这些后门中包含有配置信息，使用图8中提供的脚本可以解码。

**文件特征**

![p15](http://drops.javaweb.org/uploads/images/cbdef7d8681115e58b2e698729cf2d736aeb535e.jpg)

图15-次级C2服务器和活动标识符

### S型后门 (2013-2014)

在实验了结合Misdat和S型后门的混合型后门后，DustStorm在2013年完全弃用早期的Misdat网络协议。所有识别出的样本都使用了Borland Delphi编写，并且利用了自定义类来实现常用的后门功能。在2013年的大多数样本都使用了UPX version 3.03封装，而在2014年的变种就没有。

**文件特征**

![p16](http://drops.javaweb.org/uploads/images/a93cebed0e0daaf3b422513ed92a1e86286194f5.jpg)

图16-S型后门的文件特征

**主机标识**

**证据：**

*   可能创建一个名为 “`{Unique Identifier}_KB10B2D1_CIlFD2C`” 的互斥量
*   可能创建在系统上创建一个临时用户，名称 ：“`Lost_{Unique Identifier}`” ，密码：“`pond~!@6”{Unique Identifier}`”
*   可能创建临时文件夹`%System%\{Unique Identifier} temporarily`

**文件系统修改：**

*   后门会把自己复制到`%CommonFiles%\{Unique Identifier}\msdtc.exe`，而其他的变种使用了`%Appdata%\{Unique Identifier}\msdtc.exe`
*   可能创建文件`%HOMEPATH%\Start Menu\Programs\Startup\Realtek {Unique Identifier}.lnk`
*   这个快捷方式会指向`%CommonFiles%`中的msdtc.exe文件，使用“/Start”来启动
*   可能在`%temp%\{random numbers}.tmp`中创建临时文件

**注册表修改：**

*   可能临时创建注册表键值`HKCU\SOFTWARE\AdobeSoft`
*   可能创建注册表键值`HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run\ IMJPMIJ8.1{3 characters of Unique Identifier}`
*   可能创建注册表键值：
    
    *   `HKCU\Software\Microsoft\Windows\CurrentVersion\Internet Settings\ZoneMap\Domains\ssl.projectscorp.net\http`
    *   `HKCU\Software\Microsoft\Windows\CurrentVersion\Internet Settings\ZoneMap\Domains\ssl.projectscorp.net\https`

**网络标识**

这个后门主要在端口80上与"ssl.projectscorp.net" 和 “pic.elecarrow.com”通讯；但是，如果初始通讯失败，则会与端口443或8080通讯。后门使用了HTTP与C2服务器通讯；数据传是通过GET请求在URI中经过base64加密后传输，或在POST请求主体中传输。请求中使用的硬编码User-Agent有两个：在初始请求中使用“FirefoxApp” 和 “Mozilla/4.0 (compatible; MSIE 8.0; Windows NT 5.1; SV1)，然后，在后续通讯中使用系统的默认User-Agent。

下图中是一些HTTP请求样本。

![p17](http://drops.javaweb.org/uploads/images/c823360d534871824c312da4bccd5bbcf6d749fe.jpg)

图17-S型后门发送的初始POST请求

![p18](http://drops.javaweb.org/uploads/images/707cae08a074f16aa98c7071ce0d763af3f50af5.jpg)

图18-S型后门发送的Get请求

![p19](http://drops.javaweb.org/uploads/images/da39808ee92f83d279dca9678901f76e386f1b64.jpg)

图19-从上图中，解码后的数据参数

这个后门会尝试运行大量的测试来判断受害者用户的权限级别，包括能否向系统中添加用户，能否在%System%文件夹中创建一个目录，以及用户能否通过调用“OpenSCManagerA”来访问服务管理器。这些信息会与文件系统类型和代理信息一并传输。后门也可能会在URI中使用下面的变量来发起网络请求：“`&type=ie&`”, “`stype=info&data=`”, “`stype=srv&data=`”, “`stype=con&data=`”, “`stype=user&data=`”, “`mmid=`”, “`&type=post&stype=`”, 或“`&status=`”。

**详情**

与先前的DustStorm后门类似，这个后门也会通过“GetKeyboardType” API检测受害者使用的是否是日文键盘。后门本身能够执行shell命令，枚举系统和网络信息，操作文件，下载和执行任意文件。通过观察返现，这个后门以前是一个侦查平台，随后，攻击者将其升级为了具备完整功能的后门。

后门会首先尝试使用NetUserAdd API添加用户“`Lost_{Unique Identifier}`”，使用的密码是“`pond~!@ {Unique Identifier}`”；如果成功，木马会通过NetUserDel API移除这个用户。然后，后门会尝试使用CreateDirectoryA API创建文件夹“`%System%\{Unique Identifier}`”，并使用RemoveDirectoryA API来移除这个文件夹。一旦完成了这两个测试，木马会尝试通过调用“OpenSCManagerA”来访问WindowsService Control Manager。一旦通过POST请求传输了这些信息以及代理和文件系统信息，后门会尝试执行一系列的命令来枚举与系统和本地网络相关的信息。

![p20](http://drops.javaweb.org/uploads/images/bfe97050b71e0fa88f01798bc3ab86e3bbdf6e56.jpg)

图20-后门在系统上执行的初始命令

这些命令的运行结果会经过base64编码，作为URI中的数据参数进行传输，“`/pic/index. asp?id={Unique_Identifier}&type=ie&stype=info&data=`”。一旦将运行结果传输给C2服务器，后门就会继续连接URI“`/pic/index.asp?mmid={Unique Identifier}`”，并等待执行命令，或二进制更新。任何从C2下载下来的文件都是base64编码的，并且名称是“{Unique Identifier}.txt”。如果文件是一个二进制，则会作为 “tmp.exe”写到磁盘上并通过WinExec执行。如果成功，后门会向C2发送 “`&status=run succeed`”；如果出现错误，则发送“`&status=Error Code`”。

上面的"{Unique Identifier}”是一个8字符的十六进制字符串，这个通过加上C盘的卷序列号和后门配置中某个CRC32哈希的前0x90个字节计算得到。这一点与之前的Misdat变种不同，因为可以通过逆向来获取到磁盘的序列号。后门会从偏移0xE9FC解码其配置信息，跳过前4个字节，然后每个字节减去0x2，使用配置区块的第一个字节，这里是0x58，来异或最终值。

![p21](http://drops.javaweb.org/uploads/images/bba6dd9df4f034289bd6fd5ae18aaec8dacfe3c4.jpg)

图21-经过编码的配置区块

![p22](http://drops.javaweb.org/uploads/images/aca02cbfcffea5b3ab1cb7344835c9ff25b5cd51.jpg)

图22-解码后的配置区块

在已解码区块，首先是用16字节表示的字符串长度，然后是文本字符串。除了URL“hxxp:/ssl.projectscorp.net/pic/index.asp”和域名“pic.elecarrow.com”，上图中的绿色，蓝色，紫色分别对应端口70，443和8080。使用下面的Python函数可以解码这些配置区块。

![p23](http://drops.javaweb.org/uploads/images/8f7d40a6cbde54e29799dd62f1c1da4827b97b4f.jpg)

图23-用于解码w型配置数据的Python脚本

### ZLIB 后门 (2014-2015)

在2014年和2015年，DustStorm小组更喜欢用这个木马作为第二阶段植入。这个木马具备完整的功能，并且内置了NTLM代理认证支持，这一设计是为了将其作为ServiceDLL运行。SPEAR已经识别的所有样本都会根据受害者的具体环境来定制，使用了Microsoft Visual C++ 6编译。

**文件特征**

![p24](http://drops.javaweb.org/uploads/images/f123a22312750e3a08e08d011ff3ce1cbd7b75e1.jpg)

图24-共同的文件特征

**其他文件细节：**

*   导出函数DriveDev，DriverInit，DriverLaunch，DriverProc
*   将资源版本信息伪装成一个合法的Realtek Semiconductor 模块，或Nvidia模块，或Synaptics模块
*   PE校验和为0

**主机标识**

**文件系统修改：**

**后门位置：**

*   %WINDIR%\system32\cryptpol.dll
*   All Users %AppData%\cryptpol.dll
*   All Users %AppData%\wdd.ocx
*   All Users %AppData%\athmgmt.dll
*   All Users %AppData%\rasctl.dll
*   All Users %AppData%\rtcomdll.dll
*   All Users %AppData%\msnt.dll
*   可能在%AppData%下创建随机命名的.tmp文件
*   可能在%temp%目录下创建以“tmp”开头的临时文件

**注册表修改：**

*   会创建必要的键值来配置后门作为ServiceDll运行，包括重新定义ServiceMain来指向另一个后门的导出函数-服务名称如下：
*   CryptPol – Cryptography Policy Control Service
*   AtherosMgMt – Atheros Communications Management Service
*   WDDSVC – Windows Display Driver
*   RASCtrl – Remote Access Control Center

**网络标识**

后门会通过HTTP POST和GET请求与预先配置的C2服务器通讯。通讯内容会使用标准的Zlib压缩库 (http:/www.zlib.net/)进行压缩。在SPEAR的测试中，User-Agent是静态的 “Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.1; SV1)”。

![p25](http://drops.javaweb.org/uploads/images/d389a4a83eff349d1a5e4a8ea6da2a3d9c09789b.jpg)

图25-Zlib后门发送的初始POST请求

![p26](http://drops.javaweb.org/uploads/images/ace801d31e1daa40f11984d9f2e13c0cdae76cdf.jpg)

![p27](http://drops.javaweb.org/uploads/images/608c8079a374d20e9cf40ef8211ab0be50331efa.jpg)

图26-解压缩后的初始POST请求

在一个可控测试中，主机名，后门运行的上下文，操作系统信息和用户信息都传回了C2。

**详情**

有证据表明，攻击者对后门做出了小幅修改，会按需要简单的更新配置信息。所以，大多数后门的PE校验和并不是计算出的值。这一后门能够允许攻击者上传和下载文件，枚举文件和磁盘，枚举系统信息，枚举和操作Windows服务，枚举和模拟登录会话，模拟键盘和鼠标输入，捕捉屏幕截图并执行shell命令。

后门本身中没有多少明文字符串，除了导入表以外也没有任何身份信息。后门会通过每次向栈内推入一个字符的方式来初始化感兴趣的字符串；有越来越多的木马作者开始使用这种方法来绕过启发式检测。这个后门的配置信息使用Zlib压缩在二进制中，压缩后的数据大小保存为一个双子组，就放置在标头开头“0x78 0x9C”的后面。解压缩后的数据中会有Windows服务名称，Windows显示名称和服务描述。其中还有后门的文件名称，域名，端口以及内部企业代理。

![p28](http://drops.javaweb.org/uploads/images/24cbf93f5354b60bba79b8c5f784fec45ed49f85.jpg)

图27-解码后的配置数据示例