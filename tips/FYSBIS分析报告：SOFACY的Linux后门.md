# FYSBIS分析报告：SOFACY的Linux后门

**本文翻译自：**[http://researchcenter.paloaltonetworks.com/2016/02/a-look-into-fysbis-sofacys-linux-backdoor/](http://researchcenter.paloaltonetworks.com/2016/02/a-look-into-fysbis-sofacys-linux-backdoor/)  
**原作者：Bryan Lee 和 Rob Downs**  
**版权归原作者所有**

0x00 简介
=====

Sofacy组织，也被称为APT28或者Sednit，是一个相当知名的网络攻击间谍组织，据信跟俄罗斯有关。他们的攻击目标遍布全世界，主要针对政府、防御组织和多个东欧国家政府。已经有很多关于他们活动的报告，以至于已经有[维基百科](https://en.wikipedia.org/wiki/Sofacy_Group)的词条。

从这些报告里，我们发现该组织有丰富的工具和策略，包括利用0day漏洞攻击通用应用程序，例如JAVA或者Microsoft Office；大量使用鱼叉式网络钓鱼；利用合法网站进行水坑式攻击并且目标包括各类操作系统--Windows、OSX、Linux、iOS。

Linux下的恶意软件Fysbis是Sofacy很喜欢使用的一个工具，虽然这个工具不是特别的精巧复杂。但由于Linux安全总体上是一个不是很成熟的领域，特别是恶意软件方面。所以，这个工具帮助了Sofacy组织进行成功攻击是完全有可能的。

0x01 恶意软件评估
=====

Fysbis是一个模块化的Linux木马/后门，将插件和控制模块作为不同的类来实现。一些分析把这个恶意软件归类到Sednit组织命名名称里。这个恶意软件包括32位和64位的[ELF](http://www.linuxjournal.com/article/1059)文件。此外，Fysbis在有或者没有root权限的情况下都可以把自己植入目标系统。当需要选择安装的账户时，这增加了攻击的选择。

对3个样本的总结信息如下：

**Table 1: Sample 1 – Late 2014 Sofacy 64-bit Fysbis**

<table style="box-sizing: border-box; border-collapse: separate; border-spacing: 0px; background-color: transparent; border-top: 1px solid rgb(224, 224, 224); border-left: 1px solid rgb(224, 224, 224);"><tbody style="box-sizing: border-box;"><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">MD5</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">364ff454dcf00420cff13a57bcb78467</td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">SHA-256</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">8bca0031f3b691421cb15f9c6e71ce193355d2d8cf2b190438b6962761d0c6bb</td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">ssdeep</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">3072:n+1R4tREtGN4qyGCXdHPYK9l0H786O26BmMAwyWMn/qwwiHNl:n+1R43QcILXdF0w6IBmMAwwCwwi</td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">Size</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">141.2 KB (144560 bytes)</td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">Type</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">ELF 64-bit (stripped)</td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">Install as root</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">/bin/rsyncd</td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">Root install desc</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">synchronize and backup service</td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">Install as non-root</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">~/.config/dbus-notifier/dbus-inotifier</td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">Non-root install desc</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">system service d-bus notifier</td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">C2</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">azureon-line[.]com (TCP/80)</td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">Usage Timeframe</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">Late 2014</td></tr></tbody></table>

**Table 2: Sample 2 – Early 2015 Sofacy 32-bit Fysbis**

<table style="box-sizing: border-box; border-collapse: separate; border-spacing: 0px; background-color: transparent; border-top: 1px solid rgb(224, 224, 224); border-left: 1px solid rgb(224, 224, 224);"><tbody style="box-sizing: border-box;"><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">MD5</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">075b6695ab63f36af65f7ffd45cccd39</td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">SHA-256</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">02c7cf55fd5c5809ce2dce56085ba43795f2480423a4256537bfdfda0df85592</td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">ssdeep</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">3072:9ZAxHANuat3WWFY9nqjwbuZf454UNqRpROIDLHaSeWb3LGmPTrIW33HxIajF:9ZAxHANJAvbuZf454UN+rv eQLZPTrV3Z</td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">Size</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">175.9 KB (180148 bytes)</td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">Type</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">ELF 32-bit (stripped)</td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">Install as root</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">/bin/ksysdefd</td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">Root install desc</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">system kernel service defender</td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">Install as non-root</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">~/.config/ksysdef/ksysdefd</td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">Non-root install desc</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">system kernel service defender</td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">C2</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">198.105.125[.]74 (TCP/80)</td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">Usage Timeframe</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">Early 2015</td></tr></tbody></table>

**Table 3: Sample 3 – Late 2015 Sofacy 64-bit Fysbis**

<table style="box-sizing: border-box; border-collapse: separate; border-spacing: 0px; background-color: transparent; border-top: 1px solid rgb(224, 224, 224); border-left: 1px solid rgb(224, 224, 224);"><tbody style="box-sizing: border-box;"><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">MD5</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">e107c5c84ded6cd9391aede7f04d64c8</td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">SHA-256</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">fd8b2ea9a2e8a67e4cb3904b49c789d57ed9b1ce5bebfe54fe3d98214d6a0f61</td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">ssdeep</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">6144:W/D5tpLWtr91gmaVy+mdckn6BCUdc4mLc2B9:4D5Lqgkcj+</td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">Size</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">314.4 KB (321902 bytes)</td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">Type</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">ELF 64-bit (not stripped)</td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">Install as root</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">/bin/ksysdefd</td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">Root install desc</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">system kernel service defender</td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">Install as non-root</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">~/.config/ksysdef/ksysdefd</td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">Non-root install desc</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">system kernel service defender</td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">C2</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">mozilla-plugins[.]com (TCP/80)</td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">Usage Timeframe</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">Late 2015</td></tr></tbody></table>

总的来说，这些样本不是很复杂但是却很有效。这些样本表明了一个事实：APT攻击者并不需要高级的手段来攻击目标。相反，攻击者把高级的恶意软件和0day利用保留在手里而只使用刚好能达到目的的资源进行攻击。因此分析人员有理由用一些捷径或者trick来缩短评估威胁的时间。也就是说，分析人员应该总是通过一些方法来更有效的工作而不是一味蛮干。

0x02 利用字符串充分获取信息
=====

字符串本身就可以体现很多信息，提高了诸如静态分析分类的效率（例如使用[Yara](https://plusvic.github.io/yara/)）。表1和表2Fysbis样本安装和目标平台信息就是很好的例子。

![p1](http://drops.javaweb.org/uploads/images/393d49663bc483011c1b022704399ca5cd94b627.jpg)图1：从字符串中获得的Fysbis安装和目标平台信息

从这个例子中，我们可以发现文件的安装路径和通过匹配来确定具体的Linux版本。后面跟着的是一系列延长在目标上的存活时间的Linux shell命令。

另一个例子是跟样本功能相关的信息。

![p2](http://drops.javaweb.org/uploads/images/b083b9eabb6ab636fe4614b3ae6e0cd713e79b4a.jpg)图2：从字符串中泄露的功能信息

图2 表明了交互状态和返回的信息，让分析人员对样本功能有个大概的印象。除了可以帮助静态分析，这也可以作为后面事件响应优先级和评估威胁的出发点。

0x03 符号信息可以缩短分析时间
=====

有趣地是，最新的ELF 64位文件(表3的样本)在使用前没有strip，这导致在文件中会有额外的符号信息。对Windows[PE](https://msdn.microsoft.com/en-us/library/ms809762.aspx)比较熟悉的分析人员可以认为就是Debug版本和Release版本的区别。作为比较，如果我们分析下Fysbis strip过的样本跟 "RemoteShell" 相关的字符串，就只能发现下面的字符串：

![p3](http://drops.javaweb.org/uploads/images/d17b6f19ef9b4b36bd58aa9bc219c1309758d9d6.jpg)图3：Fysbis strip后样本跟RemoteShell功能相关的字符串

跟没有strip的样本比较：

![p4](http://drops.javaweb.org/uploads/images/f6b417bdfceb6b5155dedfad065cefc1f496d896.jpg)图4：Fysbis 没有strip的样本跟RemoteShell功能相关的字符串

一些像这样的静态分析技巧可以帮助分析人员快速分析样本的功能，更重要的是，在后续相似样本关联和发现也有用处。

此外，最新的样本表明恶意样本进行了小的改进，最显著的就是进行了混淆。表1和表2的样本都很清晰地泄露了安装信息。这跟表3的样本是不同的。用反汇编工具看下这个没有strip的样本，下面展示了在有root权限的账户中解密安装信息的相关信息。

![p5](http://drops.javaweb.org/uploads/images/462bb2e8436772a36a1589d8eff3af319844e2be.jpg)图5：样本3安装信息的汇编代码

在这个例子中，从符号信息可以看出解密的方法，有mask，路径，名字和byte数组。

![p6](http://drops.javaweb.org/uploads/images/2f9f48f7abb0d758a2db86fc676cf33fb422ea0e.jpg)图6：样本3跟root权限安装相关的byte数组的汇编代码

这个解密的算法是，用一个byte数组作为mask，作用到另外的byte数组上，使用循环并且有2个key的异或算法来生成恶意样本的安装路径、文件名和Linux root账户相关的信息。由于INSTALLUSER byte数组的存在，可以在非root情况下安装恶意样本。相同的解密方法也可以用来解密样本配置的C2信息，这进一步说明很少的符号进行就可以在很大程度上提高样本分析的完整性。

如果你想知道更多关于Fysbis的信息，样本的分析报告可以在[这里](http://vms.drweb.ru/virus/?i=4276269)获得。

0x04 基础设施分析
=====

就像Unit 42在其他文章里说的一样，我们发现攻击者好像不太愿意更换他们的基础设施。这可能是因为不想增加额外的资源，或者仅仅是因为保持对原有设施的熟悉程度来保证时效性。在Sofacy组织使用的Fysbis样本中都发现了上述2种情况。

最老的一个样本（表1），跟域名azureon-line[.]com进行交互，这个域名已经被广泛证实是Sofacy组织使用的进行控制命令的域名。通过被动DNS，我们发现这个域名解析到2个初始的IP 193.169.244[.]190 and 111.90.148[.]148，也被映射到Sofacy在这段时间内使用的其他域名。

![p7](http://drops.javaweb.org/uploads/images/3fde575f74e380f97f542aa01330b506a503f338.jpg)图7：样本1的C2信息

表2的样本，关联的IP也是Sofacy组织使用的198.105.125[.]74。这个IP跟一个叫CHOPSTICK的工具有关，具体可以[查看](https://www2.fireeye.com/rs/fireye/images/rpt-apt28.pdf)。

![p8](http://drops.javaweb.org/uploads/images/b541a4cfe7550d57bab761a6f4c90014a0d83037.jpg)图8：样本2的C2信息

最新的样本（表3），是一个之前未知的域名 mozilla-plugins[.]com。这跟前文Sofacy组织的策略相互印证，即使用跟合法公司相似的名字来作为基础设施的名字。这个域名和IP反查结果在之前都没有发现过，表明表3的样本可能跟新的团体相关。把样本3的二进制文件跟另外2个比较，发现在代码层面和行为层面都有很大的相似性。

![p9](http://drops.javaweb.org/uploads/images/c0f67feaa93f001623740dbfd2887f8958121dd5.jpg)图9：样本3的C2信息

0x05 结论
=====

Linux是在商业和家庭中常见的操作系统，并且有很多版本。数据中心、云服务都喜欢使用Linux，在网络和应用服务器市场也越来越受欢迎。Linux也是Android和其他几种嵌入式系统的基础。使用Linux的好处——特别是在商业公司——可以总结到3点：低成本的TCO、安全性和功能丰富。数据统计和比较可以清楚地评估TCO和功能，但是安全性需要深入地研究。Linux方面的知识在工业界的各个应用上都非常需要，从系统管理、大数据分析和事件响应等等。

大部分的商业活动还是在Windows环境下，这也是说核心基础设施也使用Windows服务器（例如Active Directory、SharePoint等等应用）。这表明，从实际的情况看，大部分情况下还是在支持和保护Windows下的设施。一部分公司的IT专家对Linux还不是很熟悉，特别是对于网络防护人员来说。识别和确认潜在的威胁，需要对什么是正常操作有一定的熟悉程度，这样才能发现异常情况。对于环境中的其他软件也是一样的，正常操作完全依赖于指定软件在公司中扮演的角色和功能。

对非Windows平台缺少专业知识和了解对公司的安全形势增加了很大的危险。最近的一个例子就是，Linux上的漏洞[CVE-2016-0728](http://www.pcworld.com/article/3023870/security/linux-kernel-flaw-endangers-millions-of-pcs-servers-and-android-devices.html)表明相关平台潜在威胁的广度。专业或者投机的攻击者，尽管他们有不同的动机，都增加了平台暴露的风险。尽管很多人认为Linux的特点让它有更高的安全性（实际上并不正确），但Linux上的恶意软件和漏洞确实存在，并且已经被攻击者在实际中用来攻击。