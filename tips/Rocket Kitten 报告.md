# Rocket Kitten 报告

from:http://blog.checkpoint.com/wp-content/uploads/2015/11/rocket-kitten-report.pdf

0x00 概括
=====

网络间谍小组“Rocket Kitten”，由伊朗人组成，他们的首次活动可以追溯到2014年，主要是通过钓鱼活动来传播木马，从而感染目标受害者。据报道，这个小组现在仍在活动中，最新的攻击活动出现在了2015年10月。

一些安全机构和安全专家已经分析了Rocket Kitten小组和他们的攻击活动，有大量相关的报告也介绍这个小组的行动方法、以及他们使用过的工具和技术。

这个小组使用的技术并不复杂，经常是利用钓鱼活动来攻击中东地区（包括伊朗）、欧洲和美国的个人目标和组织目标。他们已经利用定制编写的木马成功地入侵了其中的大量目标。虽然与基础设施相关的识别工作一直在进行，但是， 他们也在一次又一次的调整自己的工具和钓鱼域名，以逃避拦截。

Check Point已经从攻击者的服务器上获取到了一份 目标名单，其中确定的目标有高级别的国防官员，多个国家的大使馆，伊朗研究人员，人权活动人士，媒体和记者，学术研究所，以及物理与核科学领域的学者。

*   在这份报告中，我们总结了下列发现：
*   Check Point在调查攻击者的基础设施期间，发现了新的证据，包括此前未公布的木马标识。
*   获取到的一些信息，能完整披地露攻击活动的范围，能帮助我们了解受害者档案与攻击活动的内部关联
*   通过分析攻击数据，获取到了详细的受害者信息和产业信息，这些情况可能与伊朗的政治和军事利益息息相关
*   通过分析攻击者的失误，首次发现了主要开发者的真实身份（也就是“Wool3n.H4T”）

我们希望，这份报告以及其中提出的方法能有效地遏制攻击者的行动（应对当前的工具和基础设施）。虽然，Check Point的客户不会受到这个威胁的影响，我们还是呼吁安全厂商和木马研究专家扩大当前防护基础设施中恶意IoC（入侵标识）的涵盖范围。

0x01 调查时间回顾
=====

（如果你已经看过了以前出版的报告，可以跳过这一部分）

有几家不同的厂商，威胁情报小组和个人研究者都已经研究和分析过了Rocket Kitten的活动。在木马研究领域中，有一个老生常谈的问题，就是我们经常会在不同的报告中看到不同的代码名称和行动名称，但是实际上，这些代号有可能指的是同一起活动或同一组攻击者。

虽然木马命名方式不同，但是所有的报告都认为这些活动是从伊朗发源的。根据活动的攻击目标，以及各种间接和直接证据都能支持这一结论。尽管，我们知道攻击者可以伪造和篡改数字证据来欺骗分析者，但是根据在这几年的活动中发现的大量证据来看，这些数字信息不可能是伪造的。

虽然报道和公布了相关的恶意标识，但是Check Point发现仍然有一些攻击活动 在使用相同的方法和基础设施。其他安全厂商和Check Point的研究伙伴也都证实了这些发现。

看起来，虽然这些攻击使用的技术不复杂，但是他们完全不在意西方安全领域的研究和发现。他们往往会通过更换域名和更新他们的木马攻击， 继续按部就班地实施他们的行动。

我们现在就回顾和简要总结一些有趣的发现。

![](http://static.wooyun.org//drops/20160420/2016042019135925484.com/blob/fudaaay7cmi/wt31bsvo0ubpffnl8ftkdg?s=raybawkd2kaf)

2014年发布的‘Operation Saffron Rose’报告把这个伊朗攻击小组命名为了‘Ajax Security’（CrowdStrike使用的代码名称是‘Flying Kitten’）。报告中称，这个小组通过钓鱼攻击了伊朗的异议分子（那些企图规避政府交通监控的人）。这个小组与最近发生的Rocket Kitten活动很可能存在联系（不同的工具，但是行动模式和钓鱼域名的命名方式类似）。不过，目前还没有确凿的证据能证实这样的关联。

在同一个月里，iSight发布了BNewscaster报告，详细地说明了相似的钓鱼活动，这次的钓鱼活动利用了虚假的社交媒体账号，伪装成了‘newsonair.org.’网站的一名记者。据说，iSight与FBI合作，发现这些攻击活动都是伊朗人实施的。报告中指出，攻击者的目标有美国、英国和以色列的政客，资深军事人员和国防组织。我们没有发现有直接证据能表明Rocket Kitten与这些活动有关联。

ClearSky在2014年9月发布的文章中，首次说明了这些攻击活动使用了一种叫做‘Gholee’ 的木马（这个名称出现在了恶意有效载荷的导出函数中，可能是根据一名伊朗歌星来命名的）。研究人员发现了一些与其他攻击活动相关的线索，并且注意到大量的AV产品都无法检测出这个木马。

![](http://static.wooyun.org//drops/20160420/2016042013323161938.com/blob/fudaaay7cmi/fxkfexetzpscqvghw_nabg?s=raybawkd2kaf)

图1-ClearSky注意到的_‘gholee’_导出名称

Gadi & Tillman在31c3大会（第31届Chaos通讯大会，德国）上，通过报告首次明确了Rocket Kitten的身份，紧接着CrowdStrike确定了对伊朗攻击小组的命名方式。在报告中，研究人员介绍了黑客‘Wool3n.H4t’以及其他成员的身份。

![](http://static.wooyun.org//drops/20160420/2016042013015693474.com/blob/fudaaay7cmi/fy663nlxlt7liisfbzecga?s=raybawkd2kaf)

图2-_Tillman Werner & Gadi Evron_注意到恶意文档中的人工痕迹能指示文件创建者

接着，研究人员又介绍了攻击者使用的两个木马：

*   在深入研究了ClearSky所说的“Gholee”木马后，我们发现这个木马实际上是一个渗透测试工具的“Wrapper”，这个测试工具最初是阿根廷公司Core Security开发的，是一个合法的渗透测试工具，叫做“Core Impact”。攻击者非法修改了这个工具，然后，用在了Rocket Kitten小组的恶意攻击活动中。
*   一个基于.NET的凭据窃取器，负责窃取在受感染计算机上储存的某些凭据，并通过邮件的方式发送到‘wool3n.h4t@gmail.com (mailto:wool3n.h4t@gmail.com)’。攻击者给这个工具起的名字似乎是‘FireMalv’。

Trend Micro在2015年的报告中重新介绍了‘Gholee’木马（GHOLE）活动和‘Operation Woolen Goldfish’行动，以及一个不太复杂的键盘记录器‘CWoolger’-这个程序的名称似乎是‘woolger’（可能是从‘wool3n keylogger’这个词而来），使用了C++进行编写，现有证据表明，这个程序是从2011年开始出现的。

```
C:\Users\Wool3n.H4t\Documents\Visual Studio 2010\Projects\C-CPP\CWoolger\Release\CWoolger.pdb

```

研究人员还称，Wool3n.H4T很可能就是木马的作者，这个人只登录过伊朗的一个博客平台。

![](http://static.wooyun.org//drops/20160420/2016042019140269027.com/blob/fudaaay7cmi/w2g1poyk6hruv_ytdzmoeq?s=raybawkd2kaf)

图4-_Trend Micro_的研究人员发现了_wool3nh4t.blog.ir_

在这次的报告中，Trend Micro的研究人员记录了Rocket Kitten如何修改了Gholee木马（把‘gholee’函数重新命名为了‘function’），应该是为了避开ClearSky公布的Yara签名。另外，研究人员还记录了日期为2011年3月的Gholee木马样本，并作为了调查早期攻击活动的证据。

ClearSky仍然在继续调查这个小组的活动，并且在2015年6月公布的报告中介绍了 ‘Thamar Reservoir’活动，这次活动使用了Thamar E. Gindin来命名，因为她也是Rocket Kitten小组的攻击目标。ClearSky的研究人员指出，在这次活动中，钓鱼网站托管在了以色列的一家学术研究网站上，接着，ClearSky的研究人员发现攻击者犯了一次OPSEC（行动安全）失误，从而发现了一份详细的目标名单（一部分）。

![](http://static.wooyun.org//drops/20160420/2016042013020489073.com/blob/fudaaay7cmi/fmdpdztj5jobhqpzraw2tg?s=raybawkd2kaf)

图5-ClearSky在钓鱼服务器日志中发现的部分目标国家分布

我们在分析了这份名单后发现，这些目标都关系着国家政治利益，有的受害者是伊朗的对头， 有的则是具有重要的情报价值。ClearSky还引用了美国财政部无意在网上公开的一份备忘录，证明了攻击者来自伊朗。

ClearSky展示了大量私人定制的钓鱼邮件和通讯，其中涉及到了一些电话通话，攻击者利用了这些通话来诱惑受害者打开邮件中的附件。由此可见，这个攻击小组的毅力和行动广度不一般。

![](http://static.wooyun.org//drops/20160420/2016042013020727400.com/blob/fudaaay7cmi/az7wngcm4gqi0j8opfckjq?s=raybawkd2kaf)

图6-ClearSky发现了定制的钓鱼页面

Citizen Lab在2015年8月发布了一份报告，详细地说明了攻击者如何利用这种电话通话骗局，来诱骗受害者交出他们的双重认证令牌。很显然，攻击者会首先研究受害者，然后拨打电话，督促他们赶快处理接收到的邮件。在受到攻击的受害者中，Citizen Lab提到了EFF的国际言论自由主管Jillian York。Citizen Lab在报告中说道，在这次钓鱼攻击中，有一些钓鱼域名是已经报道过的，已经确认了与Rocket Kitten存在关联。

![](http://static.wooyun.org//drops/20160420/2016042019140765569.com/blob/fudaaay7cmi/vwa_m8mtrexwki_esfv66w?s=raybawkd2kaf)

图7-Citizen Lab发现的谷歌密码重置钓鱼页面

有趣的是，Citizen Lab后来又在报告中新增了一份来自一家新闻媒体的回应，据称，这家媒体与伊朗情报机构的关系很密切。在这篇回应下面，他们还附上了伊朗流亡记者Omid Memarian的断言，称这些攻击“无疑”是伊朗革命卫队发动的。媒体回应中嘲笑道“西方媒体这是在浑水摸鱼”，所有这些断言都是“可笑的”。

在Trend Micro和ClearSky最新发布的文章中（2015年9月），详细地说明了这个小组的情况以及现今的行动模式，并介绍了另外的几起攻击事件，以及最新的“downloader”木马。

0x02 Rocket Kitten使用的工具&基础设施
=====

Rocket Kitten攻击小组的主要攻击途径是钓鱼。有效的钓鱼攻击只需要一个定制的钓鱼页面，托管在一个价格较低的web服务器上就足够了。正如先前的报告中所说的，Rocket Kitten小组利用了各种各样的钓鱼方式，有时候是通过邮件往来与受害者通讯，有时候甚至会拨打电话来诱使受害者打开恶意附件。

在此次活动中检测到的恶意附件有的是定制编写木马，有的是“downloader”组件，而“downloader”组件会从远程服务器上获取木马，然后在受害者的机器上执行。

除此之外，我们还遇到了一些利用了“web 入侵”工具和套件的攻击活动，这些攻击活动企图入侵受害者的网站。 此前报道过的定制木马包括：

*   CWoolger—一个基于C++ 的 ‘woolen 键盘记录器’。这个木马会记录所有的键盘输入并把数据发送到木马中硬编码的FTP服务器。
*   Wrapper/Gholee—这个木马实际是重新改造后的Core Impact 渗透测试工具，允许远程访问某个平台，用于平行感染或随后的木马安装。
*   FireMalv—一个基于.NET 的Firefox 凭据窃取器。这个工具会复制储存在Firefox浏览器中的密码。

Check Point在调查中还发现攻击者使用了下面的工具：

*   .NETWoolger—一个基于.NET 的 “woolen 键盘记录器”。这个木马的功能类似于 CWoolger。攻击者似乎是在交换使用这两个工具，作为备用的感染机制（以防受害者检测到了计算机上的木马）。
*   MPK—一个多功能的定制RAT木马。这个木马能记录键盘输入，允许远程命令执行，截图和流量监控。详细的MPK木马介绍请参考附录B。

除了定制编写的木马，我们还发现攻击者使用了多种入侵和扫描工具来攻击受害者的网站。

*   Metasploit—一个开源的多功能渗透测试平台。Metasploit的 ‘meterpreter’ 有效载荷封装在了一个可执行文件中，攻击者会把这个工具添加到钓鱼邮件的附件中，当做一个RAT木马来传播。
*   Havij & SQLMap—SQL注入工具； Havij 是伊朗人开发的，而SQLMap是一个开源项目。
*   Acunetix & Netsparker—现有的web漏洞扫描器，能自动发现和利用常用web平台上的漏洞。
*   WSO Web Shell—一个无人不知的web shell- PHP 脚本，允许攻击者通过后门访问遭入侵的服务器。一般是在成功入侵后部署，以便进一步的操作。
*   NIM-Shell—伊朗黑客小组开发的一个web shell，功能与上一个类似，在遭入侵的服务器上额外使用了一个Perl脚本。

我们检测到这个Web入侵企图是从多个IP范围发起的，与Rocket Kitten C&C服务器的地址很接近。我们估计，发动这次活动的攻击者要么直接使用了这些服务器，要么就是把这些服务器配置成了代理/VPN端点来引导其攻击活动。

结合目前完成的研究工作以及Check Point所观察到的攻击活动，我们绘制了一幅攻击者的基础设施概况图。

![](http://static.wooyun.org//drops/20160421/2016042102472021667.com/blob/fudaaay7cmi/wcj7y5ogc5nxhkq8pzhoww?s=raybawkd2kaf)

*   我们不认为上述的任何提供商参与了这次恶意活动。很可能是攻击者伪装成了合法的消费者或入侵了这些服务器，而服务提供商并不知情。
*   有一些特定的范围可能是作为整体分配给了攻击者。由于IP地址分配的动态特性，在报告公布时，这些地址可能已经过期了。
*   因为卫星通讯的工作方式，显示位于德国的基础设施可能实际并没有安放在德国。更合理的猜测是，这些服务器实际位于伊朗。有一些证据是能证明我们的猜测的，比如注册人信息。

0x03 GEFILTE PHISH—永志之仇
=====

在得知了某个客户网络遭到了Rocket Kitten小组的攻击后，Check Point的研究人员决定主动参与调查。虽然，在Trend Micro和ClearSky最近公布的报告中(‘The Spy Kittens Are Back: Rocket Kitten 2’)已经详细地介绍了这次活动，但是我们想要确认我们分析的攻击活动与这次活动存在关系，并发现其他有价值的信息。

在得知了这次攻击后，我们尝试与钓鱼服务器通讯，并进行初步的勘察。我们了解到有好几个恶意域名都是用了相同的IP地址。 我们注意到这个服务器上的IP地址是活动的，接着，我们就开始探测和思考这个服务器的目的到底是什么。

结果让我们大吃一惊。

在web探测时，我们首先制作了一个GET请求脚本，尝试浏览已知的路径。一分钟后，我们发现在发出了几个请求后，包括/xampp和/phpmyadmin(!)，返回了一条200 OK响应。

我们怀疑脚本出现了误报结果，所以，我们在浏览器中输入了/xampp，结果令我们很惊讶：

![](http://static.wooyun.org//drops/20160420/2016042021091552939.com/blob/fudaaay7cmi/aar3mta6tpkkmiyx5qetqa?s=raybawkd2kaf)

图8-XAMPP的默认配置-在一个活动的攻击服务器上

我们很好奇地在浏览器中输入了直接路径，并加载了phpmyadmin界面。

直到我们把查询提交到服务器上，我们才明白了phpmyadmin被配置成了允许任何访客不使用密码就能获取root权限。

我们立刻想到“这样的疏忽一定是一个圈套”。国家级别的攻击者怎么可能会犯这么业余的错误来暴露自己的钓鱼服务器数据库呢？他们会犯这样的失误吗？

如果他们注意到了‘XAMPP Security’页面：

![](http://static.wooyun.org//drops/20160421/2016042103325251588.com/blob/fudaaay7cmi/iw8qmd4jxygnrgtmgbdb5w?s=raybawkd2kaf)图9-MySQL管理用户root不需要密码-不安全

在浏览了整个数据库后，我们很快发现了大量的图示结构，但是大多数都是空的（是为了用于测试？），其中有一个很突出：‘phakeddb’。

![](http://static.wooyun.org//drops/20160421/2016042102472686333.com/blob/fudaaay7cmi/jimrg0finb4hqiv-vfutzg?s=raybawkd2kaf)

图10-phakeddb方案-注意几个列表中的_utf8_persian_ci collation_

’phakeddb’中包含了一些非常有趣的列表和数据集；这些数据集又丰富了研究人员对木马活动的猜想。在浏览了这些列表后，我们发现了一个phishing web application，很可能是Rocket Kitten小组定制开发的。这个web应用会根据操作者的要求，生成针对特定目标定制的钓鱼页面，伪装成目标服务（Gmail，Youtube，Hotmail等）。

后来我们得知，攻击者把这个平台命名为了‘Oyun Management System’。

我们首先看看‘users’ 列表：

![](http://static.wooyun.org//drops/20160421/2016042102472932136.com/blob/fudaaay7cmi/dkk8vjaixxuvshlptzezna?s=raybawkd2kaf)

图11-‘users’ 列表

攻击者登录这个应用，也是为了像其他平台一样来设置其钓鱼活动。这个服务器似乎是在2014年8月部署的，在当时创建了所有的用户。

密码字段使用的哈希类型是什么？他们使用了MD5哈希，这点不是很意外。

但是，在这个系统中，这还不是最明显的失误。用户名 “super admin”（分配有所有的权限）的哈希是e10adc3949ba59abbe56e057f20f883e。解密爱好者都认识这个字符串就是“123456”的MD5哈希。

通过观察用户名，我们发现了几个波斯语姓名和化名，比如merah,，kaveh，ahzab 和amirhosein。这些名字可能就是活动的“策划者”-针对目标都有社交工程和钓鱼任务。（提示：“123456”并不是这份列表中唯一一个容易破解的密码）

接下来是“conversation”一栏，这好像是攻击者之间通讯使用的一个实验功能。不过很少使用。

![](http://static.wooyun.org//drops/20160420/2016042023042910643.com/blob/fudaaay7cmi/obselx97_2fslwkslmob1q?s=raybawkd2kaf)

图12-“conversation”列表

大部分信息都是指向钓鱼页面的链接，而这些页面都是攻击活动中使用的，也证明了这个数据库确实与攻击活动有关联。

有意思的是，我们可以看到用户ID 55（对应着用户表中的用户名‘attache’）发出了一个请求：

```
please 20 subject for me. 
tank you
attache

```

随后，用户ID 60（(‘john’)请求了：

```
seyeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeedddddddddddddd
ddddddddd
please
please
please
support me
please
I need make a new project please add 50 subject tank you so match fadaaaaaaaaaaaaaaaaaaaaat bos bos
;0

```

boos’是波斯语的’_kiss_'，'bos bos'可能是波斯语的'_xoxo_'

这是什么意思呢？这个系统的用户指的是什么项目和话题呢？我们通过调查‘requiretypes’更加明确了这个系统的目的：

![](http://static.wooyun.org//drops/20160420/2016042013022165017.com/blob/fudaaay7cmi/oludidyziqejkfzi1g44rw?s=raybawkd2kaf)

图13-‘requiretypes’列表

我们发现了钓鱼页面使用的代码样板，包括“Victim Full Name” “Victim User Name”等字段值。甚至我们还获取到每个字段所使用的样本。Wool3n.H4T再次引起了我们的注意（键盘记录木马的作者），因为在这一栏中不停地提到了这个人。我们有理由怀疑Wool3n.H4T编写了这个“钓鱼应用”来支持其行动。

这里还有一个非常有趣的‘supervisor@ybsoft.com (mailto:supervisor@ybsoft.com)’，但是ybsoft.com目前注册到了中国的一家电商，所以这个方向上就没什么好运了。

但是，真正的大奖还在前面。

当我们打开“projects”时，我们都忘了喘气了。很明显每个“project”代表的就是一个受害者（目标的邮箱地址），每个“project”还会分配有一个“proj_id”，一名操作员和一个特定的链接，这个链接就会发送给这名受害者。我们总共发现了1842条记录，包括从2014年8月到2015年8月期间（我们访问这个数据库的时间），所有遭到攻击的受害者。

![](http://static.wooyun.org//drops/20160420/2016042023043356110.com/blob/fudaaay7cmi/79gd9o9ipbm__h8jevr3mq?s=raybawkd2kaf)

图14-_‘projects’列表_

我们不仅仅获取到了所有受害者的邮箱地址，还获取到了每个钓鱼页面的样本值（在‘projectmailrequirevalues’中）！例如，在一个‘Google Sign-In’页面上，一般会显示受害者的全名，以及用户自己定义的公开头像。攻击者必须要复制网站的外观，感受，并在数据库中填充每个目标的全名，地址和照片。

不出所料，我们验证并获取到了已经报道过的受害者姓名和照片。

![](http://static.wooyun.org//drops/20160420/2016042019143439362.com/blob/fudaaay7cmi/x36nxkw27alspfmqauyjog?s=raybawkd2kaf)

图15-_‘projectmailrequirevalues’_列表

但是，‘projectlogs’里面有什么呢？

是我想的那样吗？

![](http://static.wooyun.org//drops/20160421/2016042103522563161.com/blob/fudaaay7cmi/bz_wpvduhzcgh01wod1hpq?s=raybawkd2kaf)

图16-每条访问钓鱼页面的记录

在这个列表里，保存着每个钓鱼页面的每一次访问记录，如果受害者上当了，这里面还会有受害者提供的凭据。现在，我们可以利用这些数据，进一步了解从2014年8月到2015年8月期间的钓鱼活动。具体请参考报告中的攻击日志分析部分。

我们在继续探索这个服务器时，发现了一个类似暴露的‘Webalizer’界面，这个界面提供了一些实用的分析工具，包括计数器和一些经常访问的链接。

![](http://static.wooyun.org//drops/20160420/2016042021093492699.com/blob/fudaaay7cmi/jppdqtlrucm1pdhzabcqag?s=raybawkd2kaf)

图17-2015年8月的Webalizer统计

Webalizer界面给我们提供了大量有用的元数据，包括“前40个访客IP”-我们能很清楚的知道是哪个攻击者访问了这个网站，并且这个界面还能为接下来的调查提供很多线索。很有趣，我们还发现了一些来源标头，这些标头指向了相同服务器上的一个路径：

![](http://static.wooyun.org//drops/20160420/2016042013023885658.com/blob/fudaaay7cmi/kljluek_dnci34zpihnvfw?s=raybawkd2kaf)

图18-登录界面

在一个‘hacker secret access’门户中，我们似乎遇到了钓鱼平台的web界面。通过测试我们此前“破解的”“admin”凭据，我们获取到了：

![](http://static.wooyun.org//drops/20160420/2016042013024025512.com/blob/fudaaay7cmi/ouxefhifibyelepa8lqawg?s=raybawkd2kaf)

图19-_“Oyun Mangement System (OMS)”_“Oyun”管理系统

现在，我们知道攻击者把这个系统命名为了“Oyun”，并且使用了Larry Page的照片作为管理员头像。这个界面上的其他部分还能允许读写phakeddb数据库，包括插入并编辑“projects”（目标）和内部聊天平台-“conversations”。

0x04 WOOLGERED—搬起石头砸自己的脚
=====

凭借在woolen键盘记录器中硬编码的凭据，我们获取到了大量的woolger DAT文件（键盘记录日志），是世界各地的受害者上传的。

同样明显的是，同样的硬编码FTP凭据实际上是C&C Windows服务器上的管理员凭据，在这上面有C$和D$ NetBIOS/SMB管理员共享，可以通过WAN访问。

![](http://static.wooyun.org//drops/20160420/2016042013330824638.com/blob/fudaaay7cmi/tigy06vfwdp0swgeyi4weg?s=raybawkd2kaf)

图20-如果你不想让研究人员获取你的CC服务器_<captain_hindsight.png>_上的管理员权限，你不应该把管理员凭据硬编码到你的木马中

在众多包含有窃取数据的键盘日志中，我们找到了一些令人震惊的发现：Rocket Kitten的攻击活动实际上感染了自己的工作站，似乎是“测试运行”了woolger。攻击者没有从C&C服务器上清除这些文件，由此可见，攻击者缺少OPSEC意识。

我们最感兴趣的还是‘Wool3n.H4t’自己的日志：

![](http://static.wooyun.org//drops/20160420/2016042013024492435.com/blob/fudaaay7cmi/rarp-ql-fryt9uisdgyu9g?s=raybawkd2kaf)

图21-测试成功

对于接下来在同一个日志文件中发现的东西，你感到惊讶吗？

![](http://static.wooyun.org//drops/20160420/2016042019144383454.com/blob/fudaaay7cmi/inttnlndvvtpgyhkrc7qvw?s=raybawkd2kaf)

图22-攻击者测试了自己的工具

是的，我们实际上刚刚发现Wool3n.H4T更换了自己打开的窗口，包括了一个‘CWoolger’项目的Microsoft Visual Studio调试会话。

在另一个日志中，我们观察到了一个特定的程序会加载‘wsc.vbs’脚本，符合Trend Micro和其他厂商公布的报告。此时，毫无疑问，我们现在查看的就是木马作者的开发工作站。

![](http://static.wooyun.org//drops/20160420/2016042014334787025.com/blob/fudaaay7cmi/y5-k5mxgju8lvah0itnh6w?s=raybawkd2kaf)

图23-互斥量和线程-你最不应该担心的就是安全性

下一个日志告诉我们，攻击者想要测试他的工具能否准确地捕捉输入到Firefox HTTP认证窗口的凭据，因此，他输入了自己的C&C服务器…

![](http://static.wooyun.org//drops/20160420/2016042021093788594.com/blob/fudaaay7cmi/dxkwoacfkbqaobjor_aljw?s=raybawkd2kaf)

图24

Wool3n.H4T的所有日志都是2014年10月获取的。

然后，我们就注意到了这个日志区段：

![](http://static.wooyun.org//drops/20160420/2016042014335487110.com/blob/fudaaay7cmi/ljtesuixnfmuxwcibzos1q?s=raybawkd2kaf)

tu 25-_‘AOL Mail’_已经缩小了范围

在Wool3n.H4T名称下的一条记录显示，一名用户使用用户名‘yaserbalaghi’登录了AOL邮箱。

这名用户是否是近期Trend Micro和ClearSky报告中指出的‘Yaser’呢？ (‘D:\Yaser Logers\CWoolger’...) 能否解释Phakeddb对“ybsoft”的引用呢？ 目前我还不清楚，我们还需要继续深入。

‘yaserbalaghi@aol.com (mailto:yaserbalaghi@aol.com)’在伊斯兰立法年1389年（2010-2011）时， 在伊朗的一个程序员论坛(“Barname Nevis”)上通过一个C++线程给出了自己的技术回答：

![](http://static.wooyun.org//drops/20160420/2016042019145022828.com/blob/fudaaay7cmi/jtidm7kvitgp0yyjyajbcq?s=raybawkd2kaf)

这名yaserbalaghi用户还发了另外几个帖子，与多个编程教程视频存在关联，涉及到的主题有ASP.NET，AJAX，jQuery 和 SQL 注入，他要求用户都使用截屏软件。

在仔细看完了这些视频后，我们发现了几处有意思的细节。首先，Yaser Balaghi是一名Microsoft Visual Studio 2010用户，很熟悉在‘Rocket Kitten’活动中使用的几个工具。

![](http://static.wooyun.org//drops/20160421/2016042107501515530.com/blob/fudaaay7cmi/xojtnkt4thmbwerq8vaahw?s=raybawkd2kaf)

图26-Yaser Balaghi_(Engineer Balaghi)_教程视频中的截图

![](http://static.wooyun.org//drops/20160421/2016042102494715766.com/blob/fudaaay7cmi/hkgyjf2-g78zznhonj9v3q?s=raybawkd2kaf)

图27-_Engineer Balaghi_主机名

通过进一步调查屏幕截图中的用户名和主机名，我们注意我们获取到的键盘记录实际上来自一台“受感染的计算机”，这台计算机上的用户就是“Engineer Balaghi”，这一点更加深了我们的猜测。但是，我们目前还无法确定；Yaser Balaghi可能就是一个很普通的名字，或者是与Wool3n.H4T和攻击者相关的某个人。

几分钟后，我们在SQLi教程视频中发现了一处OPSEC错误，这就是我们要找的关键证据：

![](http://static.wooyun.org//drops/20160421/2016042102473898629.com/blob/fudaaay7cmi/v3bljne8jv4ietcbyyjhtw?s=raybawkd2kaf)

图28-我们在看了一个小时的SQL注入教程后有了发现

![](http://static.wooyun.org//drops/20160420/2016042013333037572.com/blob/fudaaay7cmi/i66tfwxze9u0bfhu8humcw?s=raybawkd2kaf)

Wool3n.H4t当场现行。在他犯的众多错误之一，他现在竟然在教程中登录了自己的秘密化名，如果他没有这样做的话，我们根本不会联系到他的真实身份。这些视频是在2014年2月拍摄的，早于 Rocket Kitten在年中实施的首次攻击。

我们看了一眼W00l3n.Hat的桌面，发现桌面上的攻击工具就是Rocket Kitten所使用的。

![](http://static.wooyun.org//drops/20160421/2016042103325874861.com/blob/fudaaay7cmi/rymayzw8ki_yzepgtcc6ha?s=raybawkd2kaf)

图29-_Havij, Acunetix, Netsparker, SQLMap, wamp,_那是一个正常的IDA许可吗？

随后凭借几个在线查询，我们获取到了很多结果，现在我们要像验证Yaser Balaghi一样，交叉验证Wool3n.H4T的身份。

_Engineer_Yaser Balaghi不仅仅活跃在一些编程论坛上-他还有一个网站（[www.eng-balaghi.com在2014年8月下线，仍然可以通过Wayback](http://www.eng-balaghi.xn--com20148%2Cwayback-fr9y23hbnh64f0opk11bei4bcc1c6y2d9sye92b/)Machine访问）。在网站介绍中，他称自己是一名“程序员，分析师，顾问和讲师”，并且接受聘用。

![](http://static.wooyun.org//drops/20160420/2016042013333649671.com/blob/fudaaay7cmi/ed0lzxd8-_hhfbxzks47ig?s=raybawkd2kaf)

图30-Yaser Balaghi的stackoverflow账户

如果所有这些还不够的话，我们还获取到了一份最新的简历，上面显示Balaghi的所在地是德黑兰：

![](http://static.wooyun.org//drops/20160420/2016042014341541448.com/blob/fudaaay7cmi/lqlkagl3wks-ydb1gekedw?s=raybawkd2kaf)

图31-Yaser Balaghi的的简历（2013）

Balaghi毕业于伊斯兰阿萨德大学计算机软件专业，曾经担任“软件开发团队的技术总监和团队领导（Private）”和“安全与进攻主管（合法）（Private）”。随后，他又列出了曾经完成的项目，包括为一家“安全组织”开发和设计了一个“钓鱼攻击系统”。

![](http://static.wooyun.org//drops/20160421/2016042103330240388.com/blob/fudaaay7cmi/v6hvkb3tuv-0vzc3-jpnsa?s=raybawkd2kaf)

应一家网络组织的要求，设计爆破软件  
应一家网络组织的要求，设计钓鱼攻击系统  
应一家网络组织的要求，设置文件装订软件  
应一家网络组织的要求，设计一个用Python写的Windows木马  
应一家网络组织的要求，完成大量的攻击项目  
设置并执行了大量的软件项目，入侵工具和其他项目

图32-（原文和翻译）-我们不骗你

从这一部分中你能学到的一课是：如果你不想让人知道你为政府开发木马，就不要把这事写在你的简历上。

0x05 收线—分析钓鱼日志
=====

据目前的报道，攻击者会向受害者发送邮件，拨打电话，并根据不同的受害者伪装成相应的身份。很显然，攻击者们会阅读相关的分析报告，并修改自己的战术。

有一份报告中曾写到，攻击者伪装成了一名ClearSjy的研究员，引用了最近的Rocket Kitten报告，附上了一个“检测软件”，而这个软件实际上是一个木马。这种策略很有意思，在社会工程课堂上值得一提。此时，我们要说一下，这份报告在发布时并没有提供任何检测或防御工具，只有Check Point的 Software Blade。如果你收到的报告中附上了一个可执行程序，那么这个程序很可能是一个恶意诱饵。

在另一个例子中，攻击者以一名受害者的身份发送了恶意附件。一名以色列的用户在接收到附件后，很怀疑邮件的来源，于是他回了一封邮件，问“这是你发的，还是伊朗人又控制了你的电脑？”攻击者回应道（绝对是希伯来语，不是用Google翻译的 ）：“伊朗人再也无法返回我的电脑了！”

德黑兰操作中心可能已经对此进行了讨论，甚至在主餐厅中贴出了这份邮件。

由于先前的报告（阅读TrendMicro和ClearSky的近期报告）已经很好地确定了Rocket Kitten小组的行为特征，所以，我们主要是通过分析‘Oyun’系统的受害者数据库来获取新的见解。我们知道这个 数据库中包含有一份局部图，开始于2014年8月，一直到2015年8月。虽然，这些数据能成功关联到我们从其他服务器上收集的日志，但是我们无法查看包含有恶意附件的邮件（不同于用于窃取凭据的钓鱼链接），或任何描述攻击活动的完整web入侵日志。

从目标数据库的数量来看，这几个月以来，攻击行动很频繁 ，工作量也很大。这些日志中包括有访问IP的所属国家。我们通过分析，确定了下面的IP分布：

![](http://static.wooyun.org//drops/20160420/2016042013031339184.com/blob/fudaaay7cmi/p-qw7j20b8dzutlkgvnvuq?s=raybawkd2kaf)

表1-钓鱼访客的国家分布

我们研究了访客数据，判断出有很多攻击者都访问这个网站并测试了网站的功能性。我们了解到攻击者使用了伊朗的地址，美国，德国，沙特阿拉伯和荷兰的VPN。这些数据必须要拦截，并纳入参考。

我们过滤掉了系统中大约25%的日志和15%的“测试运行”项目。下表就是根据每条有效项目创建的。

通过绘制出钓鱼日志图，我们可以观察其时间线：

![](http://static.wooyun.org//drops/20160420/2016042013031515051.com/blob/fudaaay7cmi/wlcyznqcm0bxs1s_gxlekw?s=raybawkd2kaf)

表2-钓鱼日志和成功事件

我们通过研究这些数据，发现了下面几点有趣的地方：

*   平均来说，这个服务器上26%的钓鱼页面都成功地欺骗受害者输入了他们的凭据。这个结果已经相当高了，很可能就是长期和定制邮件的功劳。
*   在2015年5月26日，出现了一次网站访问高峰，并没有取得多成功。在分析时，我们发现在几分钟内出现了3批访问，‘project_ids’也越来越高，并且没有提供任何数据，这些IP地址显示都来自以色列。我们可以忽略这些访问，因为这些访问都是来自研究人员的测试。我们尝试“爆破”这些钓鱼页面，紧接着就在6月发布了Clear Sky报告。
*   似乎，攻击者在6月和7月关停了他们的平台（可能是由于报告的发布），并在8月时恢复了行动。我们发现有证据表明，这个数据库是从先前的一个服务器中移植过来的。

我们根据user_id对项目进行了分类，借此我们更了解了攻击者的任务分配；虽然我们的目标分析还远远无法得出结论，我们可以评估每名用户的作用和任务：

![](http://static.wooyun.org//drops/20160420/2016042019151013874.com/blob/fudaaay7cmi/nns2jkqvt_nupi38go_gza?s=raybawkd2kaf)

虽然，我们的了解有限，但是我们可以确定大量的攻击都是成功的-攻击者从世界各地的目标那里收集到了大量的机密信息。

0x06 后记
=====

我们认为，在木马研究行业中，Rocket Kitten是一个非常有趣的研究案例，代表了我们在这两年中亲历的国家级攻击者趋势；现在，对于网络间谍活动，不仅仅是财力雄厚的组织会雇佣上千名的网络战士来破解超级计算机的密码或通过高级研究来感染你的硬盘固件。攻击者经常会通过更简单的方式来实现更有效的入侵，比如钓鱼和简单地定制木马。

在这种情况下，与先前报道的案例一样，政府机构会雇佣当地的黑客来攻击网站或实施针对性的间谍活动。因为这些人缺乏经验，所以往往会缺少行动安全意识，留下可以追踪的痕迹，从而暴露攻击来源和他们的真实身份（比如Yaser Balaghi, Mehdi Mahdavi等）。

虽然安全公司公布了报告，代码名称和文章，但是这个攻击小组仍然在继续发动攻击，并且只有一小部分被拦截了。

这里不得不重提一个行业问题，稍微修改一下现有的木马就能绕过目前的大部分防护解决方案。如何有效地阻止攻击者的行动需要分析努力。

我们会通过与CERT合作，继续帮助托管服务提供商。我们希望这些努力能取得成果，并且能帮助减少攻击基础设施。

附录A
---

### 样本

所有的哈希都是MD5或SHA1

```
诱饵文档/Dropper
01c9cebbc39e273ac1f5af8b629a7327
08273c8a873c5925ae1563543af3715c
1685ba9dbdb0e136d68e0b1a80a969b5
177ef7faab3688572403730171ffb9c4
1ceca1757cb652ba7e5b0d45f2038955
266cfe755a0a66776df9fd8cd2fee1f1
271a5f526a638a9ae712e6a5a64f3106
2cb23916ca60a63a67d974f4ddeb2a11
393bd2fd420eecf2d4ca9d61df75ff0c
395461588e273fab5734db56fa18051b
48573a150562c57742230583456b4c02
4bf2218eb068385ca1bfff8d609c0104
50d3f1708293f40a2c0c1f151c2c426f
54ee31eb1eed79d4ddffd1423d5f5e28
55ff220e38556ff902528ac984fc72dc
5a009a0d0c5ecaac1407fb32ee1c8172
5af0cbc18c6f8ed4fd1a3f68961f5452
60f5bc820cf38e78b51e1e20fed290b5
61a808ce0b645c4824d79865be8888ed
85b79953bf2b33fb6118dc04e4c30910
8ed01ac79680d84c0ee7a5f027d8b86a
9fc345c25e6ab94bca2db6ee95d2c861
ac94ee83c91ca784a88ff26cf85e273a
aeb9d12ecbe73bfa91616ebacf24831b
c9ea312c35e9ac0809f1c76044929f2f
d0c3f4c9896d41a7c42737134ffb4c2e
d14b3e0b82e3b5d6b9cc69b098f8126d
e1a5b4ffc612270425d5d31f4c336aa9
f68a0a3784a7edfc60ad9333ec209cbf
f8547010eb4238f8fb76f4e8a756e36d
0482fc2e332918456b9c97d8a9590781095b2b53
0f4bf1d89d080ed318597754e6d3930f8eec49b0
1a999a131144afe8cb7316ebb842da4f38101ac5
2627cdc3324375e6f41f93597a352573e45c0f1e
2c3edde41e9386bafef248b71974659543a3d774
46a995df8d9918ca0793404110904479b6adcb9f
4711f063a0c67fb11c05efdb40424377799efafd
476489f75fed479f19bac02c79ce1befc62a6633
64ba130e627dd85c85d6534e769d239080e068dd
6571f2b9a0aea89f45899b256458da78ac51e6bb
788d881f3bb2c82e685a98d8f405f375c0ac2162
9579e65e3ae6f03ff7d362be05f9beca07a8b1b3
a9245de692c16f90747388c09e9d02c3ee34577e
ad6c9b003285e01fc6a02148917e95c780c7d751
ae18bb317909e16f765ba2e88c3d72d648db2798
b67572a18282e79974dc61fffb8ca3d0f4fca1b0
c485b0d59b28d37a1ac80380b0d7774bdb9d8248
c727b8c43943986a888a0428ae7161ff001bf603
e2728cabb35c210599e248d0da9791991e38eb41
e6964d467bd99e20bfef556d4ad663934407fd7b
ec692cf82aef16cf61574b5d15e5c5f8135df288
ed5615ffb5578f1adee66f571ec65a992c033a50
f51de6c25ff8e1d9783ed5ac13a53d1c0ea3ef33
f7f69c5ed94a03f6d57e9afd33c2627ff69205f2    

Wrapper/Gholee
05523761ca296ec09afdf79477e5f18d
08e424ac42e6efa361eccefdf3c13b21
0b67ebed08f09c0584b92f4e94ced778
13039118daadbe87e337310403e64454
14f2e86f11114c083856c92095d79256
1b02ac8c0e1102faaee70f4026cad291
223feb91efbe265696f318fb7c89c3fd
3dd221b0ea6f863e086868b246a6a104
4215d029dd26c29ce3e0cab530979b19
48573a150562c57742230583456b4c02
4b0edcd1d2953c26b6fc4298e8bf9150
4cdc28ab6e426dc630638488743accfb
58bcfe673d21634616d898c3127bd1bc
60f5bc820cf38e78b51e1e20fed290b5
63558e2980d1c6aaf34beefb657866fe
8a45dfec98dd96c86d933d9c1d6ef296
8bd58db9c29c53197dd5d5f09704296e
916be1b609ed3dc80e5039a1d8102e82
a42cea20439789bd1d9a51d9063ae3e4
b7de8927998f3604762096125e114042
b884f67c247d3dd6c559372a8a31a898
b8fb83d76eb67cbeed0b54c02a68256b
c222199c9a7eb0d162d5e96955739447
d5517542b5f8dc2010933ee17a846569
da976a502a3afc4ba63611d47c625738
ee41e7c97f417b07177ea420afe510a1
f3c3ed556072209b60c3342ddefba0f9
f89a4d4ae5cca6d69a5256c96111e707
02b04563ef430797051aa13e48971d3490c80636
07a77f8b9f0fcc93504dfba2d7d9d26246e5878f
0b0cdf47363fd27bccbfba6d47b842e44a365723
0b880fb3414374dbbf582217ee0288a76c904e9b
22f6a61aa2d490b6a3bc36e93240d05b1e9b956a
25d3688763e33eac1428622411d6dda1ec13dd43
37ad0e426f4c423385f1609561422a947a956398
476489f75fed479f19bac02c79ce1befc62a6633
47b1c9caabe3ae681934a33cd6f3a1b311fd7f9f
53340f9a49bc21a9e7267173566f4640376147d9
58045d7a565f174df8efc0de98d6882675fbb07f
62172eee1a4591bde2658175dd5b8652d5aead2a
6e30d3ef2cd0856ff28adce4cc012853840f6440
729f9ce76f20822f48dac827c37024fe4ab8ff70
7ad0eb113bc575363a058f4bf21dbab8c8f7073a
7fef48e1303e40110798dfec929ad88f1ad4fbd8
8074ed48b99968f5d36a494cdeb9f80685beb0f5
86222ef166474e53f1eb6d7e6701713834e6fee7
c1edf6e3a271cf06030cc46cbd90074488c05564
c6db3e7e723f20ed3bcf4c53fc4748e9591f4c40
cabdfe7e9920aeaa5eaca7f5415d97f564cdec11
ce03790d1df81165d092e89a077c495b75a14013
e6964d467bd99e20bfef556d4ad663934407fd7b
e8dbcde49c7f760165ebb0cb3452e4f1c24981f5
efd1c6a926095d36108177045db9ad21df926a6e
fa5b587ceb5d17f26fe580aca6c02ff2e20ad3c4
fd8793ce4ca23988562794b098b9ed20754f8a90
fe3436294f302a93fbac389291dd20b41b038cba
ffead364ae7a692afec91740d24649396e0fa981    

FireMalv凭据窃取器
0b0e2c4789b895e8ac44b6ada284aec1
29d93b156bcfbcecf79c5ba389094796a1ba76ee    

Woolen-Keylogger
0a22232c1d5add9d7aabdf630b6ed5af
0e2dc1cb6bda45d68ee9c751e37df73b
1a2b18cb40d82dc279eb2ef923c3abd0
1f7688653c272d5205f9070c2541a68c
3c6c1722acfb70bfa4453b69e99c98bb
662d094799e9c7108f35c00eb894205f
b4790618672197cab31681994bbc10a4
c72dce99e892bbf2537f5285a01985c0
f7e093d721d2616ecb9067934a615f70
f898eef9dfa04820bb2f798e063645a7
f9b235067b1c607b5b26896d465b6665
29968b0c4157f226761073333ff2e82b588ddf8e
5d334e0cb4ff58859e91f9e7f1c451ffdc7544c3
8e1bd64acd8bbe819ac60650eb1fa4f501d330ec
a42f1ad2360833baedd2d5f59354c4fc3820c475
a65b39d3919f15649106a039469013479a31ba4b
b9842058c88170cc45183aaaae4206c74e6c7351
c8096078f0f6c3fbb6d82c5b00211802168f9cba
d5b2b30fe2d4759c199e3659d561a50f88a7fb2e
db2b8f49b4e76c2f538a3a6b222c35547c802cef
eeb67e663b2fa980c6b228fc2e04304c8992401d
faf0fe422259d36494a0b2c9ccefe40dee978f31    

MPK
014bf8a588f614883d3d8b96024cd278
5c66b560f70c0b756bfc840b871864ce
d1b526770abb441d771f4681872d2fcb
eb6a21585899e702fc23b290d449af846123845f
f2ed8cd0154ae4d6ecf52a0bcf5fa80c7095dcd2
f710bd9ea40fd94c06d704c00e16a5941544378f    

网络流量
Wrapper/Gholee
HTTP/HTTPS [80/443]
index\.php\?c=\w+&r=\d+
Woolger
FTP [21] to 107.6.181.116, 107.6.172.54, 5.145.151.6
MPK
raw [8900,8899,8987,9090,1993] -
\/\/\[mpk\]\\\d{4} example: //[mpk]\2012 \/\/\[smpk\]}\\\d{4} example: //[smpk]}1992    

域名
account.login.gfimail.us
accounts.google.uk.to
account-user.com
drive-google.co
drives-google.co
gfimail.us
gmail-member.us.to
google-setting.com
google-verify.com
login.miicrosoftonline.us.to
login.office365.uk.to
logins-verify.com
login-users.com
mail.mail2.mod.gov.af.mail.al
mail-verify.com
my.idc.ac.il.my.to
outlook.profile.com.hmail.us
outlook.tau.ac.il.mail.al
owa.inss.mises.org.il
owas.haifa.ac.il.info.gf
owas.haifa.us.to
profile.gmail.us.to
profile.google.uk.to
profiles.faceboek.in
profiles.googel.com.inc.gs
profiles.googlemembers.com.home.kg
profiles-google.uk.to
qooqle.co
secure.www.cfr.us.to
service-logins.com
signin-users.com
signin-verify.com
signs-service.com
verification.google-it.info
video.qooqle.co
webmail.tau.ac.il.us.to
webmail.technion.ac.il.us.to
yahoo-profiles.uk.to
youtube.com.now.im    

IP地址
[107.6.181.96-127]
[107.6.172.50-62]
[107.6.154.224-231]
107.6.181.116
107.6.172.54
107.6.172.55
107.6.181.114
107.6.172.51
107.6.172.53
107.6.181.100
107.6.172.52
107.6.154.230
5.39.223.227
31.192.105.10
[5.145.151.1-7]
5.145.151.6
[84.11.146.52-63]
84.11.146.55
84.11.146.62
84.11.146.61
[109.169.22.69-72]
[109.169.61.4-8]
109.169.61.8
109.169.22.69
109.169.22.71
109.169.22.72
162.223.90.148
162.223.91.226
162.222.194.51
212.118.118.100

```

附录B-MPK技术介绍
-----------

攻击者似乎把这个木马命名为了‘MPK’，可能是根据wool3n.h4t名下的伊朗博客网站“Masoud_PK”起的名。

### 安装

木马会把自己添加到“explorer”项目下的autorun：

```
HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Run
HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Run

```

木马中包含一个Visual Basic脚本（‘tmp.vbs’），这个脚本会首先尝试把木马的可执行程序复制到目标位置：

```
Sub CopyFile(SourceFile, DestinationFile)
Set fso = CreateObject("Scripting.FileSystemObject") Dim wasReadOnly
wasReadOnly = False
If fso.FileExists(DestinationFile) Then
If fso.GetFile(DestinationFile).Attributes And 1 Then fso.GetFile(DestinationFile).Attributes = fso.GetFile(DestinationFile).Attributes - 1 wasReadOnly = True
End If
fso.DeleteFile DestinationFile, True End If
fso.CopyFile SourceFile, DestinationFile, True
If wasReadOnly Then
fso.GetFile(DestinationFile).Attributes = fso.GetFile(DestinationFile).Attributes + 1
End If
Set fso = Nothing End Sub
copyme = WScript.Arguments.Item(0) copyto = WScript.Arguments.Item(1) CopyFile copyme,copyto,0

```

另外，木马还会执行下面的Wscript，这个脚本会在9s后启动木马。

```
WScript.Sleep 9000
CreateObject("WScript.Shell").Run "iexplorer.exe [1]"

```

### 主要操作

从本质上看，这个木马是一个RAT木马（远程访问木马）。木马可以实现的功能有键盘记录，嗅探TCP和UDP流量，获取屏幕截图，并作为一个远程命令shell。

此外，这个木马还能收集关于目标系统的信息，比如文件枚举，磁盘，服务，进程信息，并且还能把文件发送到CC服务器。

木马还可以提取一些不太重要但是很敏感的信息：

*   主显示器的分辨率
*   是否具有管理员权限
*   处理器信息
*   主机名信息
*   Windows版本
*   Service Pack 版本
*   目标系统上安装的内存容量
*   网络适配器和网络配置信息
*   TCP 连接表

必须创建下面的互斥量：

```
[2]opened

```

然后，木马会检查下面的互斥量是否存在：

```
MyApp1.0

```

如果存在，木马就会退出，这样每次只能运行一个木马实例，然后会继续主要的操作。

### Keylogger

Keylogger 会把键盘输入储存到下面的文件中：

```
%TEMP%\log%d.txt

```

下面是一个简单的键盘日志输出：

```
(((((((Hello new File)))))))))
+++++++++++++
Window= VMware Accelerated AMD PCNet Adapter (Microsoft's Packet Scheduler) : Capturing - Wireshark
+++++++++++++ [UP][DOWN][DOWN][UP][UP][DOWN][UP][DOWN][DOWN][UP][UP][DOWN][DOWN][DOWN][UP][UP][UP][UP] [DOWN][DOWN][DOWN][DOWN]r
+++++++++++++
Window= Run
+++++++++++++
cmd[ENTER]
+++++++++++++
Window= C:\WINDOWS\system32\cmd.exe +++++++++++++
notepad[ENTER]
+++++++++++++
Window= VMware Accelerated AMD PCNet Adapter (Microsoft's Packet Scheduler) : Capturing - Wireshark
+++++++++++++
[DOWN][UP]
+++++++++++++
Window= Untitled - Notepad
+++++++++++++
test test test
+++++++++++++

```

随后，这个文件会被发送到远程的CC服务器。

如果木马检测到打开的“Gmail”, “Yahoo” 或“Outlook”窗口，木马就会进行特殊的处理，这样攻击者就能识别最优价值的数据。下面的字符串会附到输出文件中：

```
\r/////////////\r\nMail Find

```

### 捕捉Webcam

木马可能会捕捉Webcam中的照片。首先，文件会储存成test.bmp，随后转换成JPEG并保存成Cam.jpg，最后发送给CC服务器。

### TCP 连接表

木马会使用GetTcpTable API收集与当前TCP连接相关的可用元数据，并把获取到的数据整理成某种格式，发送给CC服务器。

### 截图

木马可能会获取截图。截图使用的文件名是Screeny.jpeg。

### 远程Shell (活动的命令执行)

木马会创建下面的进程作为一个活动的命令窗口：

```
cmd.exe /c cmd.exe

```

这个进程的输出和输入都是通过远程CC服务器的管道连接和重定向的，允许攻击者输入命令来控制受害者的计算机。首先发送给服务器的是下面的内容：

```
Welcome To mpkshell Command Line (This Message Send From Server)

```

流量监控
----

木马可能会嗅探机器上的所有TCP和UDP流量。这是通过使用RAW socket实现的。下面的状态字符串会发送到CC服务器：

```
Initializing Winsock 2.2...
Creating RAW socket...
Configuring socket for packet interception Starting the sniffing process...
UDP Packet Information:
Source IP: %s DESTINATION IP: %s SOURCE PORT: %d DESTINATION PORT: %d PACKET DATA:
#############################################################
TCP Packet Information:
Source IP: %s DESTINATION IP: %s SOURCE PORT: %d DESTINATION PORT: %d
#############################################################

```

如果当前用户权限不能执行这个操作，就会出现下面的错误：

```
the processs is not admin try after restart to while mpkProcess To Admin...

```

### 文件提取

木马可以向远程CC服务器发送任何数据。木马还能枚举系统上的所有文件，并查找攻击者指定的文件。

在文件提取时，木马会检查文件的大小。这样是为了发送4Kb“区块”的文件，每个区块框架的大小都是0x1014h字节。

在上传任何文件到CC服务器之前，木马会报告文件的大小：

```
length: %d

```

在发送了每个区块后，木马会报告当前的传输状态：

```
%d Bytes / %d Bytes

```

当传输结束后，木马会使用下面的字符串报告传输结束：

```
Completed: %d Bytes Downloaded.

```

如果有任何问题，就会报告这个字符串：

```
Failed to open %s, %s not found.

```

### 通讯协议

木马会使用IP协议上的原始socket（IPPROTO_IP标记），有效地实现其自己的协议来进行数据传输。

可执行程序自己的“文件版本信息”会被解析，用于获取服务器的IP，编码到“Company”值：

![](http://static.wooyun.org//drops/20160420/2016042013031877563.com/blob/fudaaay7cmi/-wnfrzp-cu5mklefkti5ua?s=raybawkd2kaf)

这个数据中包含有硬编码的IP地址和CC服务器的端口。在这个样本中是：

```
83.170.33.67:9090

```

木马会连接到这个IP，并定期发送‘keep-alive’信息，包含这6个字节：

```
123456

```

文件提取数据包有0x1014h个字节。前两个字节指示了需要提取的文件类型：

```
• 0811h—logs(initialpacket)
• 0810h—logs(subsequentpackets)
• 080Fh—logs(finalpacket)
• 0BCDh—webcamimages(initialpacket)
• 0BCFh—webcamimages(subsequentpackets) 
• 0BCEh—webcamimages(finalpacket)
• 0803h—screenshots(initialpacket)
• 0805h—screenshots(subsequentpackets) 
• 0804h—screenshots(finalpacket)
• 13C2h—errorwithfile

```

文件名称位于第一个数据包的偏移0x08h。接下来的数据包只会包含文件内容。