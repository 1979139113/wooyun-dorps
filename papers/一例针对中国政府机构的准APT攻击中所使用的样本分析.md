# 一例针对中国政府机构的准APT攻击中所使用的样本分析

作者:安天

微信公众号:Antiylab

博文地址:http://www.antiy.com/response/APT-TOCS.html

0x00 背景
=====

安天近期发现一例针对中国政府机构的准APT攻击事件，在攻击场景中，攻击者依托自动化攻击测试平台Cobalt Strike生成的、使用信标（Beacon）模式进行通信的Shellcode，实现了对目标主机进行远程控制的能力。这种攻击模式在目标主机中体现为:无恶意代码实体文件、每60秒发送一次网络心跳数据包、使用Cookie字段发送数据信息等行为，这些行为在一定程度上可以躲避主机安全防护检测软件的查杀与防火墙的拦截。鉴于这个攻击与Cobalt Strike平台的关系，我们暂时将这一攻击事件命名为APT-TOCS（TOCS，取Threat on Cobalt Strike之意。）

APT-TOCS这一个攻击的核心步骤是:加载Shellcode的脚本功能,通过命令行调用powershell.exe将一段加密数据加载到内存中执行。解密后的数据是一段可执行的Shellcode，该Shellcode由Cobalt Strike（一个自动化攻击测试平台）所生成。安天分析小组根据加载Shellcode的脚本进行了关联，亦关联出一个可能在类似攻击中的作为脚本前导执行文件的PE程序，但由于相关脚本可以通过多种方式来执行，并不必然依赖前导PE程序加载，且其是Cobalt Strike所生成的标准攻击脚本，因此无法判定该前导PE文件与本例攻击的关联。

这种基于脚本+Shellcode的方式注入内存执行没有硬盘写入操作，使用信标（Beacon）模式进行通信，支持多信标通信，可以同时和多个信标工作。这种攻击方式可以不依赖载体文件进行攻击，而可以依靠网络投放能力和内网横向移动按需投放，这将会给取证工作带来极大的困难,而且目前的一些沙箱检测产品也对这种攻击无效。

APT-TOCS攻击尽管看起来已经接近APT水准的攻击能力，但并非更多依赖攻击团队自身的能力，而是依托商业的自动化攻击测试平台来达成。

0x01 事件样本分析
=====

2.1 前导文件与样本加载
-------------

* * *

APT-TOCS是利用了"powershell.exe"执行Shellcode的脚本实现对目标系统的远程控制。安天分析人员认为攻击者掌握较多种最终可以达成多种脚本加载权限的远程注入手段，如利用安全缺陷或漏洞直接实现脚本在主机上执行。同时，通过关联分析，发现如下的二进制攻击前导文件（以下简称Sample A），曾被用于类似攻击：

```
病毒名称 Trojan/Win32.MSShell

原始文件名 ab.exe

MD5 44BCF2DD262F12222ADEAB6F59B2975B

处理器架构 X86

文件大小 72.0 KB (73,802 字节)

文件格式 BinExecute/Microsoft.EXE[:X86]

时间戳 2009-05-10 07:02:12

数字签名 NO

加壳类型 未知

编译语言 Microsoft Visual C++

```

该PE样本中嵌入的脚本，与安天获取到的Shellcode脚本功能代码完全相同，但加密数据存在差异，该PE样本曾在2015年5月2日被首次上传到Virustotal。

![enter image description here](http://drops.javaweb.org/uploads/images/63b9ee3723666b641d2dba634cd02e0e4463a2dc.jpg)

图1 PE文件内嵌的使用powershell.exe加载加密数据

该PE样本使用WinExec运行嵌入的恶意代码：

![enter image description here](http://drops.javaweb.org/uploads/images/fe17f2131de681c9fbac165c100947d37e7c8210.jpg)

图2使用WinExec函数调用powershell.exe加载加密数据

由此可以初步看到，这一"前导文件"可以被作为类似攻击的前导，依托系统和应用漏洞，不依赖类似文件，依然可以实现脚本的执行与最终的控制。 从目前来看，并不能确定这一前导样本与本起APT事件具有关联关系。

2.2 关键机理
--------

* * *

APT-TOCS攻击远控的核心部分是依托PowerShell加载的加密数据脚本（以下简称Sample_B），图1为脚本各模块之间的衍生关系和模块主要功能：

![enter image description here](http://drops.javaweb.org/uploads/images/75cccc3c622c03ccd6d4bddce0c96694333e7901.jpg)

2.3 APT-TOCS的主样本（SampleB）分析
---------------------------

* * *

Sample B文件的内容（base64的内容已经省略）如下：

![enter image description here](http://drops.javaweb.org/uploads/images/466c06ecf2e3bc613531e4e3e1ba70b0e822af77.jpg)

图4 Sample B的内容

该部分脚本的功能是：将base64加密过的内容进行解密，再使用Gzip进行解压缩，得到模块1，并使用PowerShell来加载执行。

2.4 脚本1分析
---------

* * *

脚本1的内容（base64的内容已经省略）如下：

![enter image description here](http://drops.javaweb.org/uploads/images/102016e2e965d3fd7e4787bc446bfd7843de6ee4.jpg)

图5脚本1的内容

此部分内容的功能是将经过base64加密的数据解密，得到模块1，写入到powershell.exe进程内，然后调用执行。

2.5 模块1分析
---------

* * *

该模块的主要功能是调用wininet模块的函数，进行连接网络，下载模块2的操作；并加载到内存中执行。

![enter image description here](http://drops.javaweb.org/uploads/images/98d1e5df3a114ace49b30c3cceb2a6da1fbb5bfc.jpg)

图6HTTP GET请求

上图为使用HTTPGET请求，获取文件：http://146.0.**_._**/hfYn。

2.6 模块2分析
---------

* * *

模块2创建并挂起系统进程rundll32.exe：

![enter image description here](http://drops.javaweb.org/uploads/images/5e38bdc7168b4ea21cc0a8fdda8414d59e00941b.jpg)

图7创建挂起系统进程rundll32.exe

写入模块3的数据：

![enter image description here](http://drops.javaweb.org/uploads/images/97f0a48301cb3d1aff65d40475e3e78dda6d7c71.jpg)

图8写入模块3的数据

模块3的数据虽然是以"MZ"开头，但并非为PE文件，而是具有后门功能的Shellcode。

![enter image description here](http://drops.javaweb.org/uploads/images/f6c2f32c5c9e4a25697e9c7bb651893f99fa166c.jpg)

图9以MZ（4D 5A）开头的Shellcode

2.7 模块3分析
---------

* * *

该模块会连接两个地址，端口号为80：

```
146.0.***.***   （罗马尼亚） 

dc.******69.info (146.0.***.***)    （罗马尼亚）

```

发送请求数据，接收返回数据。

![enter image description here](http://drops.javaweb.org/uploads/images/aac6b22f5c2484eac0a1a605762db85a9e85a4c3.jpg)

图10发送请求数据

上述IP、域名和访问地址的解密方式是"异或0x69"。 从该模块的字符串与所调用的系统函数来判断，该模块为后门程序，会主动向指定的地址发送GET请求，使用Cookie字段来发送心跳包，间隔时间为60秒。心跳包数据包括校验码、进程ID、系统版本、IP地址、计算机名、用户名、是否为64位进程，并将该数据使用RSA、BASE64加密及编码。

![enter image description here](http://drops.javaweb.org/uploads/images/49e70c2d6ac30a1ada4c8415eb00aa63ccfabcdb.jpg)

图11心跳包原始数据

由于进程ID与校验码的不同，导致每次传输的心跳包数据不相同。校验码是使用进程ID和系统开机启动所经过的毫秒数进程计算得出的。算法如下：

![enter image description here](http://drops.javaweb.org/uploads/images/c10ab3250e3494bbcee8977ae41bbd8433a6859c.jpg)

图12校验码算法

加密后的心跳包使用Cookie字段进行传输：

![enter image description here](http://drops.javaweb.org/uploads/images/17a85093f8081a2d2de4dd6311b582b7ddd7133c.jpg)

图13数据包内容

0x03 攻击技术来源的验证分析
=====

安天CERT分析人员关联的PE前导文件Sample_A和Sample B利用PowerShell的方法和完全相同，但正应为相关脚本高度的标准化，并不排除Sample_A与本次攻击没有必然联系。而基于其他的情况综合分析，我们依然判断是一个系列化的攻击事件，攻击者可能采用了社工邮件、文件捆绑、系统和应用漏洞利用、内网横向移动等方式实现对目标主机的控制。

而在分析"模块1"时，我们发现了"Beacon"等字符串，依托过往分析经验，怀疑该Shellcode与自动化攻击测试平台Cobalt Strike密切相关。于是，分析人员对使用Cobalt Strike生成的Beacon进行对比分析，验证两者之间的关系。 Cobalt Strike 是一款以metasploit（一个渗透测试平台）为基础的GUI的框架式渗透工具，Cobalt Strike的商业版，集成了服务扫描、自动化溢出、多模式端口监听、多种木马生成方式（dll木马、内存木马、office宏病毒和Beacon通信木马等）、钓鱼攻击、站点克隆、目标信息获取，浏览器自动攻击等。

3.1 模块1比较
---------

* * *

我们将模块1与使用Beacon生成的Payload进行比较，发现只有三处数据不同，分别为：Get请求时所发送的Head数据、请求的文件名称和IP地址。

![enter image description here](http://drops.javaweb.org/uploads/images/b0fb07957a612d7f7ffa5595f688e18950ef39b4.jpg)

图14模块1对比

左侧为样本模块1，右侧为由Beacon所生成的模块，从图中对比来看，可以得出结论，模块1是由Beacon所生成。 在请求时的数据包截图如下：

![enter image description here](http://drops.javaweb.org/uploads/images/7832d698b7db429bc838709655a7c8ee6d7e3cae.jpg)

图15模块1发包数据对比

3.2 模块2反汇编指令比较
--------------

* * *

分析人员将样本的模块2与Beacon相关的文件进行比较，发现两者的反汇编指令除了功能代码不同之外，其它指令完全一致，包括入口处的异或解密、加载系统DLL、获取函数地址、函数调用方式等，下面列举三处：

样本模块2

![enter image description here](http://drops.javaweb.org/uploads/images/979157755a62ecfe06c69b5738edd9f220a85d65.jpg)

Beacon相关文件

![enter image description here](http://drops.javaweb.org/uploads/images/141be72dca6cc0c889ac5094fbe0da7f9b24bfe4.jpg)

入口处异或解密（使用x86/shikata_ga_nai变形）

![enter image description here](http://drops.javaweb.org/uploads/images/756bdff8d5b5d8beafa9cdb07fe8cfe094edffd6.jpg)

![enter image description here](http://drops.javaweb.org/uploads/images/e337c071327b00ff049cdac01eeaf5583427506b.jpg)

解密后入口处代码

![enter image description here](http://drops.javaweb.org/uploads/images/7df6d6c56e3f57444c936286c1ad5aa3ec1dbd02.jpg)

![enter image description here](http://drops.javaweb.org/uploads/images/51eeab8bbc83d74d72ff534709ff50b64b70c818.jpg)

函数调用

3.3 模块3数据包对比分析
--------------

* * *

下面是样本模块3与Beacon所生成模块的Get请求比较，可以看出，两者都是使用Cookie来传输信息，该信息进行了加密，每间隔60秒主动发送请求，该数据为上线包/心跳包。

![enter image description here](http://drops.javaweb.org/uploads/images/2e0f8ff3d2c47a6f635e031b1196910f8a871f2d.jpg)

图16模块3数据包对比

3.4 Cobalt Strike特点
-------------------

* * *

利用Cobalt Strike的攻击可以在目标系统中执行多种操作，如：下载文件、上传文件、执行指定程序、注入键盘记录器、通过PowerShell执行命令、导入PowerShell脚本、通过CMD执行命令、利用mimikatz抓取系统密码等。 Cobalt Strike具有以下特点：

*   穿透沙箱
*   躲避白名单机制与云检测
*   内网渗透
*   持久化攻击
*   攻击多种平台

0x04 总结
=====

使用自动化攻击测试平台Cobalt Strike进行攻击渗透方式具有穿透防火墙的能力，其控制目标主机的方式非常隐蔽，难以被发现；同时具备攻击多种平台，如Windows、Linux、Mac等；同时与可信计算环境、云检测、沙箱检测等安全环节和手段均有对抗能力。从安天过去的跟踪来看，这种威胁已经存在近五年之久，但依然却缺乏有效检测类似威胁的产品和手段。

安天CERT分析小组之所以将APT-TOCS事件定位为准APT事件，是因为该攻击事件一方面符合APT攻击针对高度定向目标作业的特点，同时隐蔽性较强、具有多种反侦测手段。但同时,与我们过去所熟悉的很多APT事件中，进攻方具备极高的成本承担能力与巨大的能力储备不同，其成本门槛并不高，事件的恶意代码并非由攻击者自身进行编写构造，商业攻击平台使事件的攻击者不再需要高昂的恶意代码的开发成本，相关攻击平台亦为攻击者提供了大量可选注入手段，为恶意代码的加载和持久化提供了配套方法，这种方式降低了攻击的成本，使得缺少雄厚资金、也没有精英黑客的国家和组织依托现即有商业攻击平台提供的服务即可进行接近APT级攻击水准，而这种高度"模式化"攻击也会让攻击缺少鲜明的基因特点，从而更难追溯。

我们不仅要再次引用信息安全前辈Bruce Schiner的观点"一些重大的信息安全攻击事件时，都认为它们是网络战的例子。我认为这是无稽之谈。我认为目前正在发生而且真正重要的趋势是：越来越多战争中的战术行为扩散到更广泛的网络空间环境中。这一点非常重要。通过技术可以实现能力的传播，特别是计算机技术可以使攻击行为和攻击能力变得自动化。"显然，高度自动化的商业攻击平台使这种能力扩散速度已经超出了我们的预测。

我们需要提醒各方关注的是，鉴于网络攻击技术具有极低的复制成本的特点，当前已经存在严峻的网络军备扩散风险。商业渗透攻击测试平台的出现，一方面成为高效检验系统安全的有利工具，但对于缺少足够的安全预算、难以承担更多安全成本的国家、行业和机构来说，会成为一场噩梦。在这个问题上，一方面需要各方面建立更多的沟通和共识；而另一方面毫无疑问的是当前在攻防两端均拥有全球最顶级能力的超级大国，对于有效控制这种武器级攻击手段的扩散，应该负起更多的责任。

同时，APT-TOCS与我们之前所发现的诸多事件一样，体现了一个拥有十三亿人口、正在进行大规模信息化建设的国家，所面对的严峻的网络安全挑战；当然，也见证着中国用户与安全企业为应对这种挑战所做的努力。

附录一：关于Cobalt Strike及其作者的参考资料
=====

Cobalt Strike是Armitage的商业版本。Armitage是一款Java写的Metasploit图形界面的渗透测试软件，可以用它结合Metasploit已知的exploit来针对存在的漏洞自动化攻击。bt5、kali linx下集成免费版本Armitage，最强大的功能是多了个Beacon的Payload。

![enter image description here](http://drops.javaweb.org/uploads/images/1603bc40bd53ad6561c8072eb16f9f23b2e55b42.jpg)

Cobalt Strike作者：Raphael Mudge（美国），他是Strategic Cyber LLC（战略网络有限责任公司）创始人，基于华盛顿的公司为RED TEAM开发软件，他为Metaslpoit创造了Armitage，sleep程序语言和IRC客户端jIRCii。此前作为美国空军的安全研究员，渗透实验的测试者。他设置发明了一个语法检测器卖给了Automattic。发表多篇文章，定期进行安全话题的演讲。给许多网络防御竞赛提供RED TEAM，参加2012-2014年黑客大会。

![enter image description here](http://drops.javaweb.org/uploads/images/5b9cf23393b499cf5119d0017e46b4e2c7614289.jpg)

教育背景：Syracuse University 美国雪城大学，密歇根科技大学

目前就职：Strategic Cyber LLC（战略网络有限责任公司）；特拉华州空军国民警卫队

技能：软件开发信息安全面向对象的设计分布式系统图形界面计算机网络设计博客系统社会工程学安全研究等等

![enter image description here](http://drops.javaweb.org/uploads/images/98c7e6d9a9564cd2c049d87ed8f4e25ce9d7b28f.jpg)

支持的组织机构：

```
大学网络防御竞赛（CCDC）
东北North East CCDC 2008-2015
东部地区Mid Atlantic CCDC 2011-2015
环太平洋Pacific Rim CCDC 2012, 2014
东南South East CCDC - 2014
西部Western Regional CCDC - 2013
国家National CCDC 2012-2014

```

所做项目：

```
Sleep脚本语言（可扩展的通用语言，使用受JAVA平台启发的Perl语言）sleep是开源的，受LGPL许可。
jIRCii（可编写脚本的多人在线聊天系统客户端，Windows, MacOS X, and Linux平台，开源）

```

出版作品：

```
《使用Armitage 和Metasploit的实弹安全测试（Live-fire Security Testing with Armitage and Metasploit）》linux杂志
《通过后门入侵：使用Armitage的实施漏洞利用》（Get in through the backdoor: Post exploitation with Armitage）Hakin9杂志
《教程：用Armitage进行黑客攻击Linux》（Tutorial: Hacking Linux with Armitage）ethicalhacker.net
《校对软件服务设计》（The Design of a Proofreading Software Service）NAACL HLT2010计算机的语言学和写作研讨会
《基于代理的流量生成》（Agent-based Traffic Generation）Hakin9杂志等

```

贡献：

```
cortana-scripts
metasploit-loader
malleable-c2-profiles
layer2-privoting-client
armitage
项目：
商业合资企业类
After the Deadline 
Feedback Army 
Cobalt Strike
开源软件
Armitage
Far East 
jIRCii
Moconti
One Hand Army Man s
phPERL Same Game
Sleep

```

信息参考链接：

```
https://plus.google.com/116899857642591292745/posts （google+）
https://github.com/rsmudge (GitHub)
https://www.youtube.com/channel/UCJU2r634VNPeCRug7Y7qdcw (youtube)
http://www.oldschoolirc.com/
https://twitter.com/rsmudge
http://www.hick.org/~raffi/index.html 
http://www.blackhat.com/html/bh-us-12/speakers/Raphael-Mudge.html
http://www.linkedin.com/in/rsmudge
```