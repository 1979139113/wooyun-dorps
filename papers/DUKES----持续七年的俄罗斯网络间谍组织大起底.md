# DUKES----持续七年的俄罗斯网络间谍组织大起底

from:[https://www.f-secure.com/documents/996508/1030745/dukes_whitepaper.pdf](https://www.f-secure.com/documents/996508/1030745/dukes_whitepaper.pdf)

0x00 执行概括
=====

Duke网络间谍小组手头掌握有充足的资源，他们目标明确，行动有条理。我们认为，Duke小组至少从2008年开始就为俄罗斯联邦效力，帮助俄罗斯收集情报，从而制定外交和安全政策。

Duke的主要目标是西方政府以及与西方政府有关系的一些组织，比如政府部门和机构，政府智囊团和政府性承包商。他们的目标还有英联邦国家；亚洲，非洲和中东政府；与车臣极端分子有关系的组织，以及非法参与药品毒品贸易的俄罗斯人。

Duke小组最出名的一点就是持有大量的恶意软件工具，比如我们已经发现的MiniDuke, CosmicDuke, OnionDuke, CozyDuke, CloudDuke, SeaDuke, HammerDuke, PinchDuke, 和GeminiDuke。近年来，Duke每年都会发动两次大规模的钓鱼活动，攻击上百名甚至上千名与政府机构及其附属组织相关的人员。

这些活动利用了“打砸抢掠”方法，用最少的时间收集和窃取尽可能多的数据。如果入侵目标有价值，Duke就会立刻还用另外的工具，转而采用长期隐藏策略，长时间的收集情报。

除了这些大规模的活动，Duke同时还在不停的实施一些小规模活动，这些活动的针对性更强，使用了不同的工具。这些针对性活动已经至少持续了7年。其目标和持续时间都是根据俄罗斯联邦当时的外交和安全政策而确定。

Duke会时刻关注与他们的工具相关的研究报告，并采取相应的对策。但是，因为Duke（或他们的赞助者）非常重视他们的行动，所以他们会不断的修改自己的工具，避免检测并重新隐藏起来。他们不会中断自己的行动，而是会在修改工具的同时，持续执行原定的计划。

在一些极端情况下，即使安全公司和媒体注意到了一些工具时，Duke还可能会继续使用这些不经修改的工具来参与一些活动。从中不难看出，即使他们的工具被曝光，Duke还是相信自己能成功地完成入侵。

0x01 Duke小组的故事
=====

正如现在所熟知的，Duke小组的故事以PinchDuke恶意软件工具集为开篇。这个工具集包括多个loader和信息窃取木马。最重要的是，PinchDuke木马样本中总是会出现一个引人注意的文本字符串，我们认为Duke小组会用这个字符串来区分并行的攻击活动。在这些活动标识符中，经常会标明活动日期和活动目标，借此我们可以了解Duke小组更早期的行动。

2008: 车臣
--------

我们确定Duke小组最早的行动是2008年11月开始的两次PinchDuke行动。这些活动中使用了PinchDuke样本，其时间戳显示为2008年11月5日和12日。这两个样本中的活动标识符分别是“alkavkaz.com20081105” 和 “cihaderi. net20081112”。

样本中的第一个活动标识符，是在5号编译的，引用了alkavkaz.com，这个域名关联到了一个土耳其网站，宣称是“车臣信息中心”（图1，第5页）。第二个活动标识符是在12号变异的，引用了cihaderi.net，这也是一家土耳其网站，宣称提供“伊斯兰圣战新闻”，并且网站上有一个车臣板块。

由于缺少其他从2008年开始或更早期的PinchDuke样本，我们也无法估计Duke小组最早的活动是从什么时间开始的。但是，根据我们对2008年PinchDuke样本的技术分析，我们认为PinchDuke是在2008年夏天部署的。

事实上，我们认为到2008年秋天，Duke已经开发了至少两个不同木马工具集。我们这一推断是建立在另一个最早的Duke相关工具集-GeminiDuke，这个工具集的编译时间是2009年1月26日。这个样本，类似于早期的PinchDuke样本，已经是一个很成熟的样本，这也是我们认为GeminiDuke是在2008年秋天开发的原因。

Duke在2008年下半年就已经开发和操作了至少两个木马工具集，因此，我们推测有两种情况，一种是当时Duke的间谍活动规模就已经足够大，需要这样的工具；另一种是Duke预计他们的行动规模会大幅扩张，在未来有必要开发这样的工具。接下来，我们在本文的“工具与技术”章节，更详细地检查了这些Duke工具集。

词源：命名
-----

Duke工具集的命名可以追溯到卡巴斯基实验室，卡巴斯基的研究员把他们发现的首个Duke木马命名为了“MiniDuke”。他们在白皮书中说到，他们惊奇的发现MiniDuke后门在传播时利用的漏洞，就是ItaDuke使用的漏洞。名称中的“Duke”是因为研究员联想到了臭名昭著的Duqu威胁。虽然，在名称上有历史渊源，但是，并不是说Duke工具集和ItaDuke木马或Duqu有任何形式的联系。

![](http://drops.javaweb.org/uploads/images/114e6ded20314f36fee10bf5cca37d82d404f8ca.jpg)

随着研究员继续发现其他由MiniDuke的创建小组开发和使用的工具集，“Duke”也一直沿用了下来，所以，我们也用“Dukes”来代指操作这些工具集的那个小组。另外“APT29”指的也是这个小组。

当然，并不是所有的名称都是严格符合这样的命名习惯，也有例外，具体还要看特定的Duke工具集，其他常用的名称都列在了“工具与技术”章节。

2009:已知的首次针对西方的攻击活动
-------------------

根据在2009年PinchDuke样本中发现的活动标识符，Duke小组在2009年的攻击目标有格鲁吉亚国防部，土耳其和乌干达外交部等组织。另外，从中也可以看出，Duke小组在2009年的时候就已经开始注意与美国和北约相关的政治事件，因为Duke运行了很多针对美国对外政策智囊团的活动，另外的一系列活动针对的是北约在欧洲的一些演习，第三次活动的目标是格鲁吉亚“在北约的信息中心”。

在这些活动中，最突出的是两类的活动。第一个系列的活动是从2009年4月16日-17日，目标是美国外交政策智囊团，以及波兰和捷克共和国的政府研究所（图1）.这些活动利用了特别制作的恶意Microsoft Word文档和PDF文件，通过邮件附件发送给不同的人员，尝试渗透目标组织。

我们认为这类活动有一个共同的目标，就是收集目标国家对美国在波兰设立“欧洲拦截站”导弹防御基地和在捷克设立雷达站的看法。考虑到这些活动的时间，有趣的是，就在奥巴马总统刚刚在4月5日发表了演讲表明其部署这些导弹防御系统的意图，就在之后的11天，他们就开始了相关行动。

第二类活动包括两次行动，主要是想收集关于格鲁吉亚与北约关系的信息。第一次行动实用的活动标识符是“natoinfo_ge”，引用的www.natoinfo.ge网站属于格鲁吉亚政府实体，之后重命名为“北约和欧盟信息中心”。虽然，这个标识符本身没有包含日期，我们认为活动开始的日期在2009年6月7日左右，也是PinchDuke的编译日期。我们的论断是通过分析了所有其他PinchDuke样本得到的，样本标识符的日期和编译日期是在同一天中。我们怀疑的第二个活动标识符是一个月后的“mod_ge_2009_07_03”，其目标是格鲁吉亚国防部。

![](http://drops.javaweb.org/uploads/images/37712feb324b3cc619e5682c826f245c0a0bc93d.jpg)

图1-2008年和2009年的早期活动（左：alkavkaz.com网站的截图，由2008年的PinchDuke样本引用；下：2009年，PinchDuke活动中的诱饵文档，攻击目标有波兰，捷克和美国智囊团。内容好像是从BBC新闻中复制的）

2010:高加索地区的CosmicDuke危机
-----------------------

2010年春天，PinchDuke活动攻击了土耳其和格鲁吉亚，但是还有大量的活动攻击了其他英联邦独立国家成员，比如哈萨克斯坦，吉尔吉斯斯坦，阿塞拜疆和乌兹别克斯坦。在这些活动中，有一个活动的标识符是“kaz_2010_07_30”，其目标可能是哈萨克斯坦，值得注意的是这是我们观察到的最后一次PinchDuke活动。我们认为，在2010年上半年，Duke小组慢慢地不再使用PinchDuke，转而利用了一个全新的信息窃取木马，我们称之为CosmicDuke。

第一个已知的CosmicDuke工具集是在2010年1月16日编译的。在当时，CosmicDuke任然还没有后来加入的凭据窃取功能。我们认为，在2010年春天，PinchDuke的凭证和文件窃取功能正逐渐的加入到CosmicDuke上，PinchDuke就完全弃用了。

在过渡期间，CosmicDuke经常会内嵌PinchDuke，这样在执行时，CosmicDuke会写入到磁盘上，并执行PinchDuke。然后，PinchDuke和CosmicDuke会在受害主机上独立运行，包括执行独立的信息收集，数据窃取和CC通信- 虽然这两个木马经常使用同一个CC服务器。我们认为其目的是“实地考察”最新的CosmicDuke工具，同时用已经经过考验的PinchDuke来保证行动能成功。

在测试和开发CosmicDuke期间，Duke作者还开始实验利用权限提升漏洞。特别是在2010年1月19日，安全研究员Tavis Ormandy发现了一个本地权限提升漏洞（CVE-2010-0232）影响了微软Windows系统。Ormandy还附上了漏洞的POC利用源代码。就在七天后，1月26日，Duke编译了一个CosmicDuke组件来利用这个漏洞，允许工具在高权限下运行。

**一个loader加载一切**

Duke小组除了编译组件，在2010年的时候他们还主动开发了和测试了一个新的loader-这个组件会打包所有的核心木马代码并提供另一层混淆。

这个loader的第一个样本是在2010年6月26日编译的，也就是“MiniDuke”loader的前身，之后的版本在MiniDuke和CosmicDuke得到了广泛的应用。

我们在2014年发布了一份CosmicDuke白皮书，在其中我们提了提“MiniDuke loader”的历史，在当时，我们观察到这个loader在与MiniDuke使用之前，已经与CosmicDuke使用过。这个loader的第一个样本在2010年夏天投入使用，最近的样本大多是在2015年春天投入使用。

在这5年来，这个loader与Dukes小组的很多工具都有联系，比如这一版的loader就是用来从三个Dukes工具集中加载木马，包括CosmicDuke，PinchDuke和MiniDuke。

![](http://drops.javaweb.org/uploads/images/fbc9e91fb7cc0a6ea9732f1a27050b6141ac532f.jpg)

图2-WHOIS注册信息对比（左：natureinhome.com的原始whois注册信息，是Duke CC服务器的一个域名，由John Kasa在2011年1月29日注册；右：修改后的whois信息，感受一下Dukes小组的幽默）

2011:奥地利克拉根富特的John Kasai
------------------------

在2011年，Dukes又丰富了自己的木马工具集和CC基础设施。虽然Dukes应用了黑掉的网站和有意租来的服务器来作为其CC基础设施，Duke小组很少注册自己的域名，而是通过IP地址来连接他们自己管理的服务器。

但是，在2010年初，Duke打破常规，分两批注册了大量域名；第一批是在1月29日注册的，第二批是在2月13日注册的。所有这些域名都是用同一个化名注册的：“John Kasai of Klagenfurt, Austria”（图2）。Duke小组在很多活动中都使用了这些域名和木马，一直到2014年。和“MiniDuke loader”一样，这些“John Kasai”域名还提供了一个常用的线程来绑定这些工具和Dukes的基础设施。

2011: 继续扩张Dukes武器库
------------------

到2011年，Duke已经开发了至少3个不同的木马工具集，包括一些支持组件，比如loader和维持模块。事实上，在开发了新的工具集后，他们会完全弃用一些木马工具集，由此可见他们的武器库范围很广。

在2011年的时候，Dukes继续扩展了他们的武器库，新增了另外两个工具集：MiniDuke和CozyDuke。前期的工具集-GeminiDuke，PinchDuke和CosmicDuke都是围绕一个信息窃取组件设计的，而MiniDuke是一个简单的后门组件，其目的是在受害主机上实现远程命令执行。我们发现的第一个MiniDuke后门组件样本是在2011年5月发现的。但是，这个后门组件的技术与GeminiDuke非常接近，从某种程度上我们认为它们共用了一样的源代码。因此，MiniDuke的起源可以追溯到GeminiDuke，最早发现的编译日期是2009年1月。

不同于简单的MiniDuke工具集，CozyDuke是一个用途丰富，模块化的木马“平台”，其功能并不集中于一个核心组件，而是可以通过指令从CC服务器上下载的一系列模块。这些模块用于有选择地给CozyDuke提供必要的功能来完成手头上的任务。CozyDuke的模块平台明显区别于之前的Duke工具集设计风格。

CozyDuke和先前工具集的风格区别在编码方式上体现的更明显。前面提到的4个工具集都是用木马常用的极简化风格编写的；MiniDuke使用了很多用汇编语言编写的组件。但是，CozyDuke则完全相反。CozyDuke没有用汇编语言或C语言，而是用了C++，抽象性增加了，代价就是复杂性更高了。

不同于木马，早期的CozyDuke版本还没有混淆或隐藏自己的特性。事实上，它们很开放，功能很详细-比如，早期的样本中有一个没有加密的日志信息。对比来说，甚至是最早期的GeminiDuke样本会加密所有的字符串，避免暴露木马的真实特性。

最后，早期的CozyDuke版本还有很多元素都会让人联系到传统软件开发项目上，而不是木马。比如，最早的CozyDuke版本利用了Microsoft Visual C++的一个特性-运行时错误检查。这个特性能让程序执行的关键部分自动检查错误，但是，对于木马来说，代价就是木马的功能更容易逆向了。

![](http://drops.javaweb.org/uploads/images/561f6a49036322969e5995ca125e0154eb02d140.jpg)

图3-MINIDUKE使用的诱饵，以乌克兰为主题的诱饵文档，2013年2月，MiniDuke活动利用

根据CozyDuke与其他工具集之间的的风格差异，我们推测以前的Duke系列的作者可能有编写木马的背景（或有黑客经历），CozyDuke的作者可能有软件开发经历。

2012:藏身暗处
---------

我们还了解一些关于2012年Duke活动的情况。根据2012年的Duke木马样本，Duke小组似乎在继续使用和开发他们的工具。在这之中，CosmicDuke和MiniDuke的使用越来越频繁，但是只接收镜像更新。从另一方面来看，GeminiDuke和CozyDuke很少用于真正的行动中，但是一直在开发中。

2013: MiniDuke飞的和太阳太近了
----------------------

在2013年2月12日，FireEye公布了一份博客，提醒用户注意新发现的Adobe Readers 0-day漏洞CVE-2013-0640 和CVE-2013-0641，有很多攻击活动都在利用这两个漏洞。在FireEye发布文章的8天后，卡巴斯基发现这个漏洞被用于传播另一个不同的木马系列。在2月27日，卡巴斯基和CrySyS实验室公布了一份关于这个木马系列的研究报告，并称之为MiniDuke。

如我们所知，到2013年2月，在至少4年半的时间中，Dukes小组一直在操作MiniDuke和其他工具集。但是，在这期间他们的木马就被检测到了。事实上，在2009年，AV产品测试组织开始用木马集来对比测试AV产品，其中就有PinchDuke样本。但是直到2013年，更早的Duke工具集才开始放入了合适的上下文。这一点从2013年开始变化。

使用这些漏洞传播的MiniDuke样本是在2月20号编译的，当时人们已经都知道了这个漏洞的存在。你可能会说既然是在漏洞公布之后，Duke小组只需要复制就可以了。但是我们不是这样认为的。正如卡巴斯基所说的，虽然MiniDuke活动利用的这些漏洞与FireEye公布的基本一致，但是还是有小区别的。其中，最关键的就是MiniDuke漏洞中存在的PDB字符串。当使用特定的编译设置时，编译器就会生成这些字符串，也就是说MiniDuke使用的漏洞组件必须独立编译，这点不同于FireEye的描述。

我们不知道Duke是不是自己编译的这些组件，还是找别人变异的。但是，Duke不可能像FireEye所说的那样，简单的复制了漏洞二进制，然后再重新利用。

我们看来，Duke小组在漏洞遭曝光后还坚持使用有三点原因。第一，Duke小组可能对自己的能力相当自信（认为对手的应对速度不够快），他们不在意自己的目标是不是在检查自己有没有被人盯上。第二，Duke认为MiniDuke活动能提供的价值值得去冒险。第三，在FireEye发出提醒之前，Duke小组可能在活动中投入了很多，他们可能无法承受中途而费的代价。

我们认为，从某种情况上说这三种情况可能是共存的。在这份报告中，你会注意到，这种情况不是个例，而是经常发生的，即使这样，他们还是要按计划执行他们的行动，而不是采取应对策略。

与卡巴斯基白皮书中说明的一样，MiniDuke活动从2013年2月开始通过钓鱼邮件来发送恶意的PDF附件。这些PDF文件会偷偷地用MiniDuke感染受害人，同时显示一个诱饵文档来迷惑受害者。这些文档的标题有“Ukraine’s NATO Membership Action Plan (MAP) Debates乌克兰加入北约成员行动计划辩论”, “The Informal Asia-Europe Meeting (ASEM) Seminar on Human Rights非正式亚欧会议人权研讨会”和 “Ukraine’s Search for a Regional Foreign Policy乌克兰探求区域外交政策” （图3）。这些活动的目标，据卡巴斯基级称，有的位于比利时，匈牙利，卢森堡和西班牙。根据从MiniDuke的CC服务器上获取的日志文件，卡巴斯基还发现了一些高价值受害者，分别来自乌克兰，比利时，罗马尼亚，捷克共和国，冰岛，美国和匈牙利。

2013: 令人好奇的案例OnionDuke
----------------------

在2月的活动之后，MiniDuke的活动数量有所减少，但是在2013年中并没有完全停止。不过，整个Dukes小组并没有要停下来的迹象。事实上，我们发现了另一个Duke木马工具集-OnionDuke，首次出现在2013年。和CozyDuke类似，OnionDuke设计有丰富的功能，也采用了类似的模块化平台方法。OnionDuke工具集中包括多种模块，用于窃取密码，信息收集，DDoS攻击，甚至是向俄罗斯媒体的网络VKontakte发送垃圾邮件。OnionDuke工具集中还有一个投放器，一个信息窃取器变体和多个不同版本的核心组件，负责与不同的模块交互。

OnionDuke奇怪的原因是它从2013年夏天开始使用的感染载体。为了传播这个工具集，Dukes小组使用了一个封装器在合法应用程序中整合OnionDuke，创建的种子文件包含有植入了木马的应用。然后上传到托管种子的网站上（图4）。使用这些种子文件下载应用的受害者会感染上OnionDuke。

![](http://drops.javaweb.org/uploads/images/277d023f5f25b7024403530732ead5aacf74a38a.jpg)

图4-ONIONDUKE的木马种子，在这个种子文件中包含有一个植入了OnionDuke工具集的可执行程序

对于我们观察到的大多数OnionDuke组件，我们注意到的第一个版本是在2013年夏天编译的，说明在这段时间中Duke在开发这个工具集，。但是，我们观察到的第一个OnionDuke 投放器样本，只会和这个工具集中的组件配合使用，这个样本是在2013年2月编译的。这点很重要，因为由此表明在任何Duke行动进入公众视野前，OnionDuke就已经在开发中了。所以，OnionDuke的开发绝对不可能简单的是为了替换过时的Duke木马，而是为了配合其他工具集的使用。由此可见，Dukes计划并行使用这5个木马工具集，Dukes掌握有庞大的资源和能力。

2013: Dukes与乌克兰
---------------

在2013年，Dukes采用的多数诱饵文档都涉及到乌克兰，包括乌克兰外交部首席副部长签署的一封信，由冰岛驻乌克兰大使馆发给乌克兰外交部的一封信，以及一份名为“乌克兰探寻地区性外交政策”的文档。

但是，这些诱饵文档是在2013年11月欧洲抗议乌克兰动荡之前编写的。这点非常重要，因为不同于预测，我们真正观察到当Dukes注意到乌克兰局势后，与乌克兰相关的活动量开始下降了，而不是增长。

这点不同于其他的俄罗斯攻击者（比如Operation Pawn Storm），他们在乌克兰危机后增加了针对乌克兰的活动。这也和我们的分析一致，Dukes的目标是收集情报来支持外交政策的制定。在乌克兰危机之前，当时俄罗斯还在权衡乌克兰的选择，所以，当时乌克兰是Dukes的目标。但是当俄罗斯直接采取行动后，Dukes对乌克兰的态度就不一样。

![](http://drops.javaweb.org/uploads/images/a7725cebdd5efa6201dfd47f73673409a1d4738f.jpg)

图5-COSMICDUKE的诱饵，诱饵文档的截图，似乎是一份购买生长激素的订单，用于2013年9月的CosmicDuke活动

2013: CosmicDuke对毒品宣战
---------------------

在2013年9月，出现了一次转折性事件，CosmicDuke活动攻击了参与非法化学物质贸易的俄罗斯人（图5）。

卡巴斯基实验室，有时会用‘Bot Gen Studio’来代指CosmicDuke，我们推测‘Bot Gen Studio’是一个木马平台，也叫做“合法间谍软件”工具；因此，他们认为可能是有两个独立的机构在使用CosmicDuke分别来打击毒品经销商和政府部门。但是，我们感觉，攻击毒品销售商和政府的CosmicDuke操控者一定不是两个完全独立的部门。可能是因为公用了同一家木马提供商，但是这也无法解释行动技术和CC基础设施上为什么会出现这么多的重叠。另外，我们感觉攻击毒品经销商是Duke分部的一个新任务，可能是因为毒品贸易涉及到了安全政策问题。我们还任务这项任务是临时的，因为在2014年春天之后，就再也没有观察到类似的活动目标。

2014: MiniDuke浴火重生
------------------

在2013年的时候，研究人员都注意到了MiniDuke活动，因此其数量大幅的减少了，但是2014年初，这个工具集又全力回归了。所有的MiniDuke组件，从loader到下载器到后门，都经过了简单的升级和修改。有意思的是，通过这些修改我们能了解到其主要目的是重新隐藏自己，避免被检测到。

在这些修改中，最重要的就要数loader的修改。正是因为这些修改才有了后来的“Nemesis Gemina loader”，因为我们在很多样本中都找到了PDB字符串。但是，这还是迭代后的早期MiniDuke loader。

我们发现的第一个Nemesis Gemina loader样本（2013年11月14日）是用于加载更新后的MiniDuke后门，但是在2014年春，CosmicDuke中也使用了Nemesis Gemina loader。

2014: CosmicDuke的兴盛与衰落
----------------------

在MiniDuke曝光后，CosmicDuke也遭到了曝光，F-Security在2014年7月2日公布了一份关于CosmicDuke的白皮书。第二天，卡巴斯基也公布了自己的木马研究报告。值得注意的是，虽然CosmicDuke已经使用了超过4年，并且也经过了几次镜像修改和更新，甚至最近的CosmicDuke样本中也常常内嵌维持模块，其中有些能追溯到2012年。这些样本的功能也涵盖了2010年时候CosmicDuke的功能。

因此，能观察到在7月初，Dukes小组又开始恢复CosmicDuke的一些活动，这也是很有价值的。在当月月底，我们发现了一些在7月30日编译的CosmicDuke样本，在这些样本中，一些在以前版本的样本中出现的无用代码都被删除了。同样，这么多年以来，CosmicDuke样本中也有很多硬编码值是没有修改过的。我们认为木马作者修改或移除了一些他们觉得可能有助于识别和检测的部分，从而避免工具被检测到。

在修改CosmicDuke的同时，Dukes小组也在修改他们的loader。类似于CosmicDuke工具集，从2010年第一个样本之后，MiniDuke和CosmicDuke使用的loader也经过了重大更新（Nemesis Gemina更新）。同样，大部分的修改工作也是集中在移除多余的代码，尝试区别于旧版本的loader。但是，有意思的是，木马作者还尝试了另外的逃逸技巧-伪造loader的编译日期。

![](http://drops.javaweb.org/uploads/images/8a284cb0dda6cd00888a564d2a9357c2195bd260.jpg)

图6-COZUDUKE诱饵，左：伪装成美国信件传真的诱饵文档，用于CozyDuke活动；右-猴子视频诱饵，也是用于CozyDuke活动

第一个CosmicDuke样本是我们在首次研究了CosmicDuke之后发现的，这个样本是在2014年7月30日编译的。这个样本使用的loader据说是在2010年3月25日编译的。但是，根据在编译中遗留下来的一些痕迹，我们知道这个loader使用的Boost库版本是1.54.0，只在2013年7月1日公布过。因此，这个编译时间戳一定是伪造的。或许，Dukes小组认为伪造一个更早的时间可能会迷惑到研究人员。

在2014年到2015年春，Dukes继续在修改CosmicDuke来逃避检测，同时也在实验异或loader的方法。但是，在实验的同时，Dukes小组还在开发全新的loader，我们在2015年春的CosmicDuke活动中发现了首个全新的样本。

虽然，不出意外，Dukes会根据各大安全公司发布的报告来修改自己的工具集，但是，他们的应对方式还是值得注意的。和2013年2月的MiniDuke报告类似，Dukes小组似乎又优化了接下来的活动优先级来隐藏自己。他们本可以停止使用所有的CosmicDuke（至少是在他们开发出新的loader之前），或让CosmicDuke完全退役，因为他们还有一些其他的工具集是可以使用的。但是，他们只沉寂了很短的时间，在稍微对工具集进行修改后，就又继续行动了。

2014: CozyDuke和猴子视频
-------------------

虽然，我们知道CozyDuke从2011年末出现以来，一直在发展，但是知道2014年7月初，首次大规模的CozyDuke活动才进入了我们的视野。这次活动，和之后的CozyDuke活动一样，首先是用钓鱼钓鱼邮件来模拟成常见的垃圾邮件。这些钓鱼邮件的一些链接最终会导致用户感染CozyDuke。

在7月初，一些CozyDuke钓鱼邮件会伪装成e-fax送达通知，一种垃圾邮件惯用的主题；并且还使用了“US letter fax test page”诱饵文档，一年后的CloudDuke也是使用了这个诱饵。但是，在不止一个例子中，邮件中没有使用链接，而是用了压缩文档“Office Monkeys LOL Video.zip”，这个文件托管在DropBox云服务上。有趣的是，这个例子没有使用常规的虚假PDF文件，而是用了一个Flash视频作为诱饵，更具体的说就是2007年的超级碗广告，办公室里的猴子（图6）。

2014: OnionDuke使用了恶意的Tor节点
--------------------------

在2014年10月23日，Leviathan 安全集团公布了一份文章，描述了他们发现的一个恶意Tor退出节点。他们注意到这个节点似乎在恶意篡改通过HTTP连接从这个节点上下载的所有可执行程序。如果执行了这些经过篡改的应用，受害者可能就会感染木马。在11月14日，F-Security公布了一份文章，把木马命名为OnionDuke，并发现这个木马与CosmicDuke和其他的Duke工具集存在关系。

根据我们对OnionDuke的调查，我们认为从2014年4月开始到Leviathan在2014年10月公布报告的大约7个月中，研究人员发现的这个Tor退出节点被被用于包装与OnionDuke相关的可执行程序（图7）。这种方式与2013年夏天通过种子文件来传播木马应用的方式类似。

在调查通过Tor节点传播的这个OnionDuke变体期间，我们又发现了另一个OinionDuke变体在2014年春成功入侵了一个受害者，这名受害者来自某个东欧国家的外交部。这个变体与通过Tor节点传播的木马在功能上存在很大的差别，也就是说这些不同的功能是为了针对不同的受害者。

我们认为，这个通过Tor节点传播的OinionDuke变体，不是为了执行攻击活动，而是为了构成小型的botnet，以便日后使用。这个OnionDuke变体与2013年夏天通过种子文件传播的一个木马有关联。这两种感染途径，与Dukes经常使用的钓鱼邮件相比，都非常随意，也没有针对性。

另外， 这个OnionDuke变体的功能是由很多模块提供的。有的是收集信息，有的会尝试窃取受害者的用户名和密码，这些功能都是针对性攻击木马经常具备的。另外还有两个已知的OnionDuke模块则完全相反，一个设计用于DoS攻击，另一个是为了向俄罗斯VKontakte社交网站发送预定的信息。这种功能在犯罪型botnet中更常见，而不是国家赞助的针对性攻击。

至此，我们已经至少识别了两个独立的OnionDuke botnet。我们认为第一个botnet是在2014年1月开始形成的，使用了未知的感染途径和已知的恶意Tor节点，并且一直持续到11月份，直到我们发布了文章。对于第二个botnet，我们认为是在2014年8月开始形成的，并且一直持续到了2015年1月。我们还没能确定第二个botnet使用的感染途径，但是其使用的CC服务器已经公开了目录列表，允许我们获取文件中存在的受害人IP列表。这些IP的地理分布（图8）验证了我们的猜测，这些OnionDuke变体不是为了在针对性攻击中打击高价值目标。

有一种看法是认为这些botnet是Dukes小组的犯罪业务部分。但是，如果是为了商业DoS攻击或垃圾邮件，这个botnet的规模（大约1400个bots）就太小了。不过，OnionDuke也会窃取受害人的用户凭据，作为另外的收入来源。但是，反对这种声音的原因是因为这种方式取得的价值在黑市上太低了。

2015: Dukes加大赌注
---------------

在2015年1月，Duke活动数量开始显著增加，有上千人都接收到了包含CozyDuke恶意链接的钓鱼邮件。我们好奇的是，这些钓鱼邮件和e-fax主题的垃圾邮件非常类似，都是勒索软件和其他犯罪软件常用的传播方式。由于接收到邮件的用户数量太多，Dukes很可能没有向其他小数量活动一样，专门的定制邮件。

但是，与常用垃圾邮件的相似性可能是为了更险恶的目的。不难想象，安全分析师在面对着沉重的网络攻击负担时，会把这种常见的垃圾邮件简单地看作 “一个犯罪活动”，致使这样的活动隐藏到 “芸芸众生”中。

CozyDuke活动表现出Dukes行动有长期进行的趋势，使用了多种木马工具集来打击一个单一的目标。在这种情况下，Dukes首先会使用CozyDuke（以一种更明显的方式）尝试感染大量的潜在目标。然后，他们会使用工具集来收集关于受害者的初始信息，接着再判断应该继续攻击哪些受害者。对于感兴趣的受害者，Dukes接着会部署不同的工具集。

我们认为，这种战术的主要目的是为了尝试躲避目标网络中的检测。即使受害者的组织注意到了最初的CozyDuke活动，或者遭到了曝光，防御者首先会查看的就是与CozyDuke工具集相关的入侵标志（IOC），但是，如果当时Dukes已经在受害者的网络中运行了，只要使用另一种工具集，而这个工具集的IOC与曝光的不一样，那么有理由相信，受害者的组织会用更长的时间才会注意到渗透。

在先前的例子中，Dukes小组就曾经更换过木马工具集，无论是作为初始阶段或后续阶段的工具集。然而，对于这些CozyDuke活动，Dukes小组似乎应用了两个特殊的后阶段工具集-SeaDuke和HamerDuke，有意设计用于在攻破网络上留下维持后门。Hammeruke是一个后门系列，首先在野外发现是在2015年2月；而SeaDuke是一个跨平台后门，据赛门铁克称，首次在野外发现是在2014年10月。这两个工具集都是CozyDuke活动部署的。

SeaDuke的特殊性在于它是用Python编写的，设计能在Windows和Linux系统上运行；这是我们首次发现Dukes使用了跨平台工具。其中一个可能的原因可能是开发这样灵活的木马可能会更多的遇到使用Linux作为操作系统的受害者。

![](http://drops.javaweb.org/uploads/images/06ef959b694963cae0527205dfeda81b435fcca6.jpg)

图7-OnionDuke使用恶意Tor节点感染受害者的流程

![](http://drops.javaweb.org/uploads/images/3d84cd756bc84c35b88aa98080a572ab51b936e2.jpg)

图8-OnionDuke Botnet的地理分布

![](http://drops.javaweb.org/uploads/images/f5a73428e23235c18aefce6185f556dc41d6cc03.jpg)

图9-OnionDuke CC推文，OnionDuke使用的推文，里面的链接指向了一个图像文件，这个图像文件中潜入了一个OnionDuke版本更新

同时，HammerDuke是一个只能在Windows上运行的木马（用.NET编写），有两个变体。简单的那个会通过HTTP或HTTPS连接一个硬编码服务器，下载执行命令。更高级的那一个，会使用算法来生成定期更改的Twitter账号，然后尝试搜索从这个账号发出的推文，根据里面的链接下载执行命令。这样，高级的HammerDuke变体就能通过合法的Twiter使用，来隐藏其网络流量。这种方法并不是HammerDuke独家使用的，MiniDuke, OnionDuke和CozyDuke都会类似的利用Twitter（图9）来获取链接，另外下载payload或命令。

2015: CloudDuke
---------------

在2015年7月初，Dukes小组开始执行另一次大规模钓鱼活动。在这次活动中使用的木马工具集是以前没有出现过的，并且我们认为7月份的这次活动，标志着Dukes首次开始真正的部署工具集，而不是小规模的测试了。

CloudDuke工具集至少包括一个loader，一个downloader和两个后门变体。这两个后门（作者内部称之为“BastionSolution” 和“OneDriveSolution”），允许操作员远程在入侵设备上执行命令。但是，这两个后门完成这一操作的方式是截然不同的。BastionSolution变体会简单的从一个受Dukes控制的硬编码服务器上获取命令，OneDriveSolution是利用微软的OneDrive云存储服务与主控通信，从而增加了防御者注意并拦截通信信道的难度。最重要的还要数2015年7月，CloudDuke活动的时间线。这次活动似乎包括两次独立的钓鱼攻击，一次是在7月初，另一次是从7月20号开始的。Palo Alto Networks在7月14日，发布了关于第一波攻击的CloudDuke详细技术分析报告。卡巴斯基随后又在7月16日公布了更多细节。

这些公布都在第二波攻击之前，并且引起了公众的广泛关注。尽管引起了公众的注意，并曝光了大量的技术细节（包括IOC），Dukes还是继续发动了第二波钓鱼攻击，包括继续使用CloudDuke。Dukes小组并没有更改钓鱼邮件的内容，也没有更换邮件的格式，而是又恢复成了他们以前用过的efax主题格式，甚至重新使用了在一年前CozyDuke活动（2014年7月）中使用的诱饵文档。

这次更加突出了Dukes小组的行为特征。首先，与2013年2月的MiniDuke和2014年夏天的CosmicDuke活动一样，Dukes又一次清楚地优化了行动顺序来保持隐秘性。其次，又突出了Dukes小组的胆大，傲慢和自信；他们对自己的能力有信心，能成功的入侵目标，即使自己的工具和技术都已经曝光，这样的情况下，他们也坚信自己的行动不会受到影响。

2015: 继续用 CosmicDuke执行外科手术式的打击
------------------------------

除了CozyDuke和CloudDuke这样肆无忌惮的大规模活动外，Dukes还在继续用CosmicDuke来执行更加毫不隐瞒，外科手术式的攻击活动。我们发现的最新活动开始于2015年春和2015年夏天的早些时候，这些活动使用了恶意文档来利用近期未修复的一些漏洞。

波兰安全公司Prevenit发表文章详细地介绍了这些活动，他们称这两次活动的目标是波兰的一些机构，在钓鱼邮件中使用了用波兰语作为名称的恶意附件。类似的，我们观察到第三次活动攻击了格鲁吉亚的一些机构，使用了用格鲁吉亚语作为名称的恶意附加，翻译过来就是 “NATO加强对黑海的控制.docx”。

据此，我们并不认为Dukes会抛弃原来隐秘的针对性活动，转而投向公开的投机性CozyDuke和CloudDuke式的活动。相反，我们认为他们只是在通过增加新的工具和技术来扩展自己的活动。

![](http://drops.javaweb.org/uploads/images/ae563393e46b62e8ce5687c5d8bbe63f141ce416.jpg)

图10-Duke工具集的活动时间分布

Dukes工具与技术
----------

![](http://drops.javaweb.org/uploads/images/a9d6ebe62ae47b85b532ccaa02053adfbb371859.jpg)

PinchDuke工具集包含多过loader和一个核心的信息窃取木马。与PinchDuke工具集关联的loader已经发现与CosmicDuke配合使用过。

PinchDuke工具集包含多过loader和一个核心的信息窃取木马。与PinchDuke工具集关联的loader已经发现与CosmicDuke配合使用过。

PinchDuke信息窃取木马会收集系统配置信息，窃取用户凭据并从入侵的主机上收集用户文件，然后通过HTTP(S)把这些文件转发到一个CC服务器上。我们认为PinchDuke的凭据窃取功能是基于Pinch凭据窃取木马的源代码（也就是LdPinch），最早是在21世纪初开发的，后来在地下论坛上传播过。

PinchDuke的目标凭据与下面的这些软件或服务有关：

*   The Bat!
*   Yahoo!
*   [Mail.ru](http://mail.ru/)
*   Passport.Net
*   Google Talk
*   Netscape Navigator
*   Mozilla Firefox
*   Mozilla Thunderbird
*   Internet Explorer
*   Microsoft Outlook
*   WinInet Credential Cache
*   Lightweight Directory Access Protocol (LDAP)

PinchDuke还会查找在预定时间范围内创建的文件和在预定列表中出现的文件扩展。

令人好奇的是，多数PinchDuke样本中都包含有下面的俄语错误信息： “Ошибка названия модуля! Название секции данных должно быть 4 байта!”

大体意思是： “模块名有错误！数据节名称的长度必须是4字节！”

![](http://drops.javaweb.org/uploads/images/4b2404a35febd9e1b3507a2a423d81d7bae2bd59.jpg)

GeminiDuke工具集包括一个核心的信息窃取器，一个loader和多个与维持相关的组件。不同于CosmicDuke和PinchDuke，GeminiDuke主要是收集受害者计算机上的配置文件。收集的信息包括：

*   本地用户账户
*   网络设置
*   网络代理设置
*   已安装的驱动
*   运行中的进程
*   用户曾经执行过的程序
*   在启动时自动运行的程序和服务
*   环境变量值
*   在任何用户的home文件夹中出现的文件和文件夹
*   在任何用户的My Docments中出现的文件和文件夹
*   安装到Program Files文件夹的程序
*   近期访问的文件，文件夹和程序

在木马中很常见，GeminiDuke的信息窃取器使用了一个互斥量来确保只有一个实例是在运行的。不常见的是，这个互斥量经常使用一个时间戳作为自己的名称。我们认为这些时间处是在GeminiDuke的编译期间，根据计算机的本地时间确定的。

比较GeminiDuke的编译时间发现，它经常引用UTC+0时区的时间，把本地时间作为互斥量的名称，并根据预设的时区调整时差，我们注意到所有的互斥量名称都引用了一个时间和日期，这个日期就是在样本编译时间戳的几秒钟之内。另外，所有在冬天编译的GeminiDuke样本都使用了UTC+3作为时间戳的时区，而在夏天的编译的样本，就把UTC+4作为时间戳的时区。

已经观察到的时区符合2011年以前的莫斯科标准时间（MSK）的定义，就是在冬天时区是UTC+3，在夏天是UTC+4。在2011年，MSK时间不再遵守夏令时间（DST），并且开始全年使用UTC+4，并在2014年全年使用UTC+3。某些GemiiDuke样本使用的互斥量就是当MSK还遵守夏时令时编译的，这些样本的时间戳符合当时的莫斯科时间定义。

![](http://drops.javaweb.org/uploads/images/0b569c2aa40a80edf40bf9b9ac14a3f1106d7ff0.jpg)

莫斯科的时区图：粉色：MSK(UTC+3)，橙色：UTC+4

但是，在MSK修改后编译的GeminiDuke样本还是冬天用UTC+3，夏天用UTC+4。虽然，使用Windows的计算机会自动调整DST，但是更改时区需要安装Windows更新。因此，我们认为Dukes小组没有更新用来编译GeminiDuke样本的计算机，所有在之后出现的样本时间戳还是遵循以前的莫斯科标准时间定义。

GeminiDuke信息窃取器有时还会封装一个loader，这可能是GeminiDuke都有的，以前也从未在其他的Duke工具集中发现过。GeminiDuke有时还会内嵌其他的可执行程序，尝试维持在受害者的计算机上。这些维持组件是经过特别定制的，为了配合GeminiDuke使用，但是他们还使用了相同的技术来作为CosmicDuke的维持组件。

![](http://drops.javaweb.org/uploads/images/bad724bf7e2f5e039267b71b1a8eecda1e3da7b4.jpg)

CosmicDuke工具集是围绕一个主要的信息窃取组件设计的。这个信息窃取器附带了大量的组件，操作员会有选择的在主要的组件上添加其他的组件来提供额外的功能，比如多种建立维持的方法，以及尝试利用权限提升漏洞的模块，从而用更高的权限来执行CosmicDuke。CosmicDuke的信息窃取功能包括：

*   键盘记录
*   获取屏幕截图
*   窃取粘贴板内容
*   窃取符合定义列表的文件扩展
*   导出用户的加密证书，包括私钥
*   收集用户凭据，包括多种聊天和邮件程序，web浏览器的密码

CosmicDuke可能使用HTTP，HTTPS，FTP或WebDav来把收集到的数据传输到一个硬编码CC服务器。虽然我们认为CosmicDuke是一个完全自定义编写的工具集，不会共用其他Duke工具集的代码，但是其很多高级功能的实现都与其他的Dukes武器有相通之处。

具体来说，CosmicDuke用于从目标软件提取用户凭据和检测分析软件的技术，都是基于PinchDuke的技术。同样，许多CosmicDuke的维持组件使用的技术也用在了与GeminiDuke和CozyDuke相关的组件上。在所有的这些例子中，技术都是相同的，但是代码都经过了修改，导致在最后的实现上存在小区别。

我们发现，有一小部分CosmicDuke样本中还有的组件会尝试利用近期公布的漏洞 CVE-2010-0232或CVE-2010- 4398权限提升漏洞。以CVE-2010-0232为例，这个利用似乎是直接建立在安全研究员Tavis Ormandy在披露漏洞时公布的概念验证代码上。我们认为，CVE- 2010-4398利用也是建立在公开的概念验证上。

除了经常内嵌维持或权限提升组件外，CosmicDuke还偶尔会内嵌PinchDuke，GeminiDuke或MiniDuke的组件。应该注意的是，CosmicDuke并不会与后者交互操作，内嵌的木马会把木马写入到磁盘上执行。之后，CosmicDuke和第二个木马会独立操作，包括各自联系自己的CC服务器。有时候，两个木马会使用同一个CC服务器，但是其他情况下，甚至是使用的服务器也都不是一样的。

最后值得注意，虽然多数CosmicDuke的编译时间戳看起来是真实的，但是我们还是注意到，有几个是伪造的。一起一个就是前文中提到的，可能是为了逃避检测。另一个是在2010年秋天与CosmicDuke和PinchDuke联合使用的一个loader变体。这些loader样本的编译时间戳都是在2001年9月24日或25日。但是，这些loader样本大都内嵌了CosmicDuke变体来利用CVE-2010- 0232权限提升漏洞，所以其编译时间戳不可能是真的。

深度阅读：

1.  Timo Hirvonen; F-Secure Labs; CosmicDuke: Cosmu with a Twist of MiniDuke; published 2 July 2014;`https://www.f-secure.com/ documents/996508/1030745/cosmicduke_ whitepaper.pdf`
2.  GReAT; Securelist; Miniduke is back: Nemesis Gemina and the Botgen Studio; published
3.  July 2014;`https://securelist.com/blog/ incidents/64107/miniduke-is-back-nemesis- gemina-and-the-botgen-studio/`

![](http://drops.javaweb.org/uploads/images/3be046f8933b5db267fd136e5eda87d63fb6fe03.jpg)

MiniDuke工具集包括多个downloader和后门组件，卡巴斯基在最初的MiniDuke白皮书中成这些组件为MiniDuke “阶段1” “阶段2”和 “阶段3”组件。另外，有一个特别的loader经常会关联到MiniDuke工具集，称作 “MiniDuke loader”。

这个loader经常与其他的MinDuke组件一起使用的同时，也还是经常与CosmicDuke和PinchDuke组件联用。事实上，我们发现这个loader样本最早还是与PinchDuke使用。但是，为了避免混淆，我们还是称这个loader为 “MiniDuke loader”。

有两处关于MiniDuke组件的细节需要值得注意。首先，一些MiniDuke组件是用汇编语言编写的。虽然，很多木马都是在 “以前的那种好奇心驱使下”用汇编语言编写的，但是现在越来越少见。第二，一些MiniDuke组件中没有包含硬编码的CC服务器地址，而是通过Twitter来获取当前的CC服务器地址。使用Twitter来获取CC服务器的地址（或作为备份，如果没有硬编码的主要CC服务器响应）也是OnionDuke，CozyDuke和HammerDuke的特性。

深度阅读：

1.  Costin Raiu, Igor Soumenkov, Kurt Baumgartner, Vitaly Kamluk; Kaspersky Lab; The MiniDuke Mystery: PDF 0-day Government Spy Assembler 0x29A Micro Backdoor; published 27 February 2013;`http:// kasperskycontenthub.com/wp-content/ uploads/sites/43/vlpdfs/themysteryofthepdf0- dayassemblermicrobackdoor.pdf`
2.  CrySyS Blog; Miniduke; published 27 February 2013;`http://blog.crysys.hu/2013/02/miniduke/`
3.  Marius Tivadar, Bíró Balázs, Cristian Istrate; BitDefender; A Closer Look at MiniDuke; published April 2013;`http://labs.bitdefender. com/wp-content/uploads/downloads/2013/04/ MiniDuke_Paper_Final.pdf`
4.  CIRCL - Computer Incident Response Center Luxembourg; Analysis Report (TLP:WHITE) Analysis of a stage 3 Miniduke sample; published 30 May 2013;`https://www.circl.lu/files/tr-14/ circl-analysisreport-miniduke-stage3-public.pdf`
5.  ESET WeLiveSecurity blog; Miniduke still duking it out; published 20 May 2014;`http://www. welivesecurity.com/2014/05/20/miniduke-still- duking/`

![](http://drops.javaweb.org/uploads/images/2d95ee390cd5c12e7b84a5da5e4445dd628d43ca.jpg)

CozyDuke不是一个简单的木马工具集，而是一个模块化的木马平台，围绕一个核心的后门组件形成。CC服务器可以指令这个组件来下载并执行任意模块。并且这些模块能提供给CozyDuke丰富的功能。已知的CozyDuke模块有：

*   命令执行模块，用于执行任意的Windows命令提示符命令
*   木马窃取模块
*   NT LAN管理器（NTLM）哈希窃取模块
*   系统信息收集模块
*   屏幕截图模块

除了模块，CozyDuke还可以通过指令，来下载并执行其他独立的可执行程序。在某些观察到的例子中，这些可执行程序是自解压文档，其中包含着入侵工具，比如，PSExe和Mimikatz，还有执行这些工具的脚本文件。在其他情况下，CozyDuke还会下载和执行其他Duke工具集的工具，比如OnionDuke，SeaDuke和HamerDuke。

深度阅读：

1.  Artturi Lehtio; F-Secure Labs; CozyDuke; published 22 April 2015;`https://www.f-secure. com/documents/996508/ 1030745/CozyDuke`(PDF)
2.  Kurt Baumgartner, Costin Raiu; Securelist; The CozyDuke APT; 21 April 2015;`https://securelist. com/blog/research/69731/the-cozyduke-apt/`

CozyDuke中的PDB字符串实例

*   **E:\Visual Studio 2010\Projects\Agent_NextGen\Agent2011v3\Agent2011\Agent\tasks\bin\ GetPasswords\exe\GetPasswords.pdb**
*   **D:\Projects\Agent2011\Agent2011\Agent\tasks\bin\systeminfo\exe\systeminfo.pdb**
*   **\192.168.56.101\true\soft\Agent\tasks\Screenshots\agent_screeshots\Release\agent_ screeshots.pdb**

![](http://drops.javaweb.org/uploads/images/09776653aeeae7a4847456f9a6a62fb8f5409cca.jpg)

OnionDuke工具集至少包括一个dropper，一个loader一个信息窃取器和多个模块变体。

OnionDuke首先引起我们的注意是因为它是通过一个恶意的Tor退出节点进行传播。这个Tor节点会拦截所有下载下来的未加密可执行文件，并通过添加嵌入了OnionDuke的恶意封装器来篡改这些可执行文件。一旦受害者下载了这些文件并执行，封装器就会用OnionDuke感染受害者的计算机，然后执行原始的合法可执行文件。

这个封装器还会用于封装合法的可执行文件，然后让用户从种子网站上下砸。这样，如果受害者下载的种子中包含封装后的可执行文件，他们就会感染OnionDuke。

最后，我们还观察到，感染了OnionDuke的受害者首先已经感染了CozyDuke。在这种情况下，CozyDuke就会接收到CC服务器的指令，下载并执行OnionDuke工具集。

深度阅读：

1.  Artturi Lehtio; F-Secure Weblog; OnionDuke: APT Attacks Via the Tor Network; published 14 November 2014;`https://www.f-secure.com/ weblog/archives/00002764.html`

![](http://drops.javaweb.org/uploads/images/529ad44db53662e387758e1f3a6c6b2648f1775e.jpg)

SeaDuke是一个简单的后门，主要是执行从CC服务器上获取的命令，比如上传和下载文件，执行系统命令并评估额外的Python代码。SeaDuke很有趣的地方是它是用Python写的，并且是跨平台的，可以在Windows和Linux上运行。

已知SeaDuke的唯一感染途径是通过现有的CozyDuke感染，由CozyDuke下载和执行SeaDuke工具集。

类似于HammerDuke，当CoyDuke完成了初始感染并获取到可用信息后，SeaDuke似乎主要用作CozyDuke受害者上的一个二级后门。

深度阅读：

1.  Symantec Security Response; “Forkmeiamfamous”: Seaduke, latest weapon in the Duke armory; published 13 July 2015;`http://www.symantec.com/connect/blogs/ forkmeiamfamous-seaduke-latest-weapon- duke-armory`
2.  Josh Grunzweig; Palo Alto Networks; Unit 42 Technical Analysis: Seaduke; published 14 July 2015;`http://researchcenter.paloaltonetworks. com/2015/07/unit-42-technical-analysis- seaduke/`
3.  Artturi Lehtio; F-Secure Weblog; Duke APT group’s latest tools: cloud services and Linux support; published 22 July 2015;`https://www.f- secure.com/weblog/archives/00002822.html`(`http://secure.com/weblog/archives/00002822.html`)

在SeaDuke源代码中发现的跨平台支持

![](http://drops.javaweb.org/uploads/images/cfba759a589f16bfddde450c18db2012a94f2757.jpg)

![](http://drops.javaweb.org/uploads/images/97ccf4cd557751da8ab0f214fcc3c73c5eca1ee4.jpg)

HammerDuke是一个简单的后门，设计的作用类似与SeaDuke。具体来说，HammerDuke的唯一已知感染途径是由CozyDuke下载和执行，这样，再结合HammerDuke简单的后门功能，表明Dukes小组主要是将其用作二级后门，首先是由CozyDuke执行初始感染，并窃取可用的受害者信息。

但是，HammerDuke也是挺有意思的，因为是用.NET编写的，并且还偶尔会用Twitter作为CC通信信道。一些HammerDuke变体中只包含硬编码的CC服务器地址，从这上面获取命令，但是其他的HammerDuke变体首先会使用一个自定义算法根据当前日期，来生成一个Twitter账号。如果账号已经存在，HammerDuke就会搜索这个账户推文中的图像链接，其中包含有执行命令。

HammerDuke使用Twitter和特制的图像文件和其他的Duke工具类似。OnionDuke和MiniDuke也使用了日期算法来生成Twitter账户，并搜索从这个账户中发表的推文，查找到像文件的链接。但是，对比来说，OnionDuke和MiniDuke的图像链接中嵌入了的是可以下载和执行的木马，而不是指令。

类似的，GeminiDuke也会下载图像文件，但是这些文件中可能会嵌入工具集本身的配置信息。但是不同于HammerDuke，GeminiDuke下载的URL是硬编码在初始配置中的，而不是从Twitter中获取的。

深度阅读：

1.  FireEye; HAMMERTOSS: Stealthy Tactics Define a Russian Cyber Threat Group; published July 2015;`https://www2.fireeye.com/rs/848-DID-242/ images/rpt-apt29-hammertoss.pdf`* APT29就是Dukes小组

![](http://drops.javaweb.org/uploads/images/0c6e2ba43670beae8c82f58ce5a48bda4f92346a.jpg)

CloudDuke这个工具集至少包括一个downloader，一个loader和两个后门变体。CloudDuke downloader会从预先定义的位置下载和执行另外的木马。有趣的是，这个位置可能是一个web地址，或一个OeDrive账户。

这两个CloudDuke后门变体都支持简单的后门功能，与SeaDuke类似。其中一个变体会通过HTTP或HTTPS使用一个预先配置的CC服务器，另一个变体会使用OneDrive账户来交换命令和窃取到的数据。

深度阅读：

1.  Artturi Lehtio; F-Secure Weblog; Duke APT group’s latest tools: cloud services and Linux support; published 22 July 2015;`https://www.f-secure. com/weblog/archives/00002822.html`
2.  Brandon Levene, Robert Falcone and Richard Wartell; Palo Alto Networks; Tracking MiniDionis: CozyCar’s New Ride Is Related to Seaduke; published 14 July 2015;`http://researchcenter. paloaltonetworks.com/2015/07/tracking- minidionis-cozycars-new-ride-is-related-to- seaduke/`
3.  Segey Lozhkin; Securelist; Minidionis – one more APT with a usage of cloud drives; published
4.  16 July 2015;`https://securelist.com/blog/ research/71443/minidionis-one-more-apt-with- a-usage-of-cloud-drives/`

感染途径
----

Dukes主要是利用钓鱼邮件让受害者感染木马。这些钓鱼邮件有的特别伪装成垃圾信息来传播常见的犯罪软件，并发送给大量的接收人，有的是高度针对性的邮件，发送给少量的接收人（或一个人），其中的内容也是与预定目标高度相关的。在有些情况下，Dukes会利用已经被攻破的受害者向其他目标发送钓鱼邮件。

Dukes小组利用的钓鱼邮件中，要么使用特制的恶意附件，或托管着木马的URL链接。当使用恶意附件时，这些附件有的是利用常用软件中的漏洞，比如Microsoft Word 或 Adobe Reader，或者是附件中包含有图标和文件名，经过混淆后看起来不像是一个可执行文件。

我们只在一些实例中发现，Dukes小组没有使用钓鱼作为初始的感染途径，就是某些OnionDuke变体。这些变体在传播木马时，要么使用恶意的Tor节点在合法的应用中植入OnionDuke工具集，要么通过种子文件来下载植入了木马的合法应用。

最后，需要注意，Dukes有时候在感染了受害者后，还会用另外的木马工具再感染这名受害者。比如，使用感染了CozyDuke的受害者又感染了SeaDuke， HammerDuke或 OnionDuke；感染了CosmicDuke的受害者还会感染PinchDuke，GeminiDuke或MiniDuke。

诱饵
--

Dukes小组通常会在感染途径中使用诱饵。这些诱饵可能是图像文件，文档，Flash视频或类似的文件，在感染过程中会显示给用户，从而迷惑受害者，防止他们注意到恶意活动。这些诱饵的内容有的是非针对性的材料，比如电视商业广告-办公室猴子；针对性文档，内容是与受害者相关的，比如报告，邀请或参数某项事件的人员名单。

一般情况下，这些诱饵的内容可能来自公共来源，有的是复制的公共材料，比如新闻报道，或公开的合法文件。但是，在某些情况下，高度针对性的诱饵会使用非公共来源的材料。这些内容可能是窃取的其他的Duke工具集受害人。

漏洞利用
----

Dukes小组既在感染途经中利用漏洞，也在木马中利用了漏洞。但是，我们只发现了一个实例-利用CVE-2013-0640来部署MiniDuke-我们认为这个漏洞在当时是一个0-day漏洞。在所有已知的漏洞利用案例中，我们认为，Dukes并不是自己发现漏洞，也不是设计漏洞来利用，而是购买了漏洞。在其他的案例中，我们认为他们只是购买了漏洞或概念验证。

![](http://drops.javaweb.org/uploads/images/fc39c8a1add21aef7069500ee525e06a5d3f070a.jpg)

俄罗斯时区图，粉色：MSK（UTC+3）;橙色：UTC+4

归属和赞助国家
-------

归属判断一直是一个很难的问题，但是在理解这类威胁和与之对抗时，这个问题的答案又是很重要的。在文中，我们已经表达了观点，我们认为Dukes小组是俄罗斯赞助的一个间谍小组。为了得到这一结论，我们首先分析了这个小组的目的和动机。

根据我们目前了解到的，在这7年中，Dukes小组是如何选择目标的，他们的目标中一直都有与外交政策和安全政策相关的实体。这些目标包括一些组织，比如外交部，大使馆，参议院，议会，国防部，国防承包商和智囊团。

在其中一个有趣的例子中，Dukes的目标中还出现过涉及非法药品运输的实体。但是，即使是这样的目标也和他们的一贯目标相一致，考虑到毒品也与安全政策有关系。据此，我们对自己的判断有信心，Dukes小组的主要任务是通过收集情报来支持外交和安全政策的确定。

很自然的，这就牵扯到了国家赞助的问题。根据这个小组的主要任务，我们认为主要的受益者是政府。但是Dukes小组或部门是属于政府机构的吗？还是外部承包商？只认钱的犯罪团伙？一群有技术的爱国分子？我们不清楚。

根据Dukes的活动长度，我们推测在这些行动中投入的资源数量和活动数量只会越来越多。我们认为，这个小组掌握有大量，关键的，稳定的经济支撑。Dukes一直在实施大规模的活动，攻击高价值目标，同时也会参与小规模的目标更广的活动，很显然，他们协调的很好，没有遇到行动冲突。所有，我们认为Dukes是一家独立的，协调能力强的大型组织，有着明确的责任划分和目标。

Dukes小组有时候会为了保持隐蔽性而重新安排行动的继续情况。他们的2015年CozyDuke和CloudDuke活动将这特点发挥到了极致。最极端的例子就是CloudDuke活动，当时的多家安全厂商都认为Dukes都曝光了CloudDuke活动后，他们又在2015年7月继续进行了活动。所以，我们认为Dukes的主要任务很有价值，其赞助者认为行动继续比一切都重要。

就我们看来，Dukes小组对公众关注的无所谓，是因为其赞助者具有权势，并且一直紧密联系着Dukes，所以Dukes才能肆无忌惮的进行行动。我们认为唯一有权力提供这样全面的保护的就只能是国家政府。所以我们认为Dukes或者在政府内工作或直接就是效力于政府，绝不可能是犯罪团伙或其他第三方。

最后一个问题：哪个国家？我们无法负责人的证明是哪个具体的国家在支持Dukes。但是，在我们看来，所有的证据都指向了俄罗斯。而且，目前我们也没有发现能有证据来驳斥这一理论。

卡巴斯基实验室先前已经注意到，在一些Dukes木马样本中出现了俄语。我们也在很多PinchDuke样本中发现了俄语的错误信息：“Ошибка названия модуля! Название секции данных должно быть 4 байта!”，大体意思是 “模块名称有错误，数据节名称必须是4字节！”

另外，卡巴斯基还指出，根据编译时间戳，Duke木马的作者主要是在周一至周五工作，从早上6点到下午4点， UTC+0时间。对应着UTC+3时区，也就是莫斯科时间的早9点和下午7点，基本涵盖了大部分俄罗斯西部，包括莫斯科和圣彼得堡。

卡巴斯基实验室对Duke木马作者的工作时间分析也印证了我们自己的分析，以及FireEye的分析。很多GeminiDuke样本中的时间戳也支持这种根据时区的推断，这种相似性表明这个小组的工作地点位于莫斯科标志时区，详情请参考GeminiDuke技术分析部分。

最后，Dukes小组的目标有-西欧国家的外交部，西方智囊团和政府组织，甚至是俄罗斯毒枭-都与俄罗斯外交和安全政策相关。虽然，Dukes的目标看起来是全球的政府，但是我们并没有发现他们针对俄罗斯政府。但是证据不足不代表就是没有证据，这是很有趣的一点。

根据列出的证据和分析，我们认为，Duke工具集是一个独立的，掌握有大量资源的大型组织开发的（我们称之为Dukes小组），旨在为俄罗斯政府提供与外交和安全政策相关的情报，用于交换支持和保护。 归属判断一直是一个很难的问题，但是在理解这类威胁和与之对抗时，这个问题的答案又是很重要的。在文中，我们已经表达了观点，我们认为Dukes小组是俄罗斯赞助的一个间谍小组。为了得到这一结论，我们首先分析了这个小组的目的和动机。

根据我们目前了解到的，在这7年中，Dukes小组是如何选择目标的，他们的目标中一直都有与外交政策和安全政策相关的实体。这些目标包括一些组织，比如外交部，大使馆，参议院，议会，国防部，国防承包商和智囊团。

在其中一个有趣的例子中，Dukes的目标中还出现过涉及非法药品运输的实体。但是，即使是这样的目标也和他们的一贯目标相一致，考虑到毒品也与安全政策有关系。据此，我们对自己的判断有信心，Dukes小组的主要任务是通过收集情报来支持外交和安全政策的确定。

很自然的，这就牵扯到了国家赞助的问题。根据这个小组的主要任务，我们认为主要的受益者是政府。但是Dukes小组或部门是属于政府机构的吗？还是外部承包商？只认钱的犯罪团伙？一群有技术的爱国分子？我们不清楚。

根据Dukes的活动长度，我们推测在这些行动中投入的资源数量和活动数量只会越来越多。我们认为，这个小组掌握有大量，关键的，稳定的经济支撑。Dukes一直在实施大规模的活动，攻击高价值目标，同时也会参与小规模的目标更广的活动，很显然，他们协调的很好，没有遇到行动冲突。所有，我们认为Dukes是一家独立的，协调能力强的大型组织，有着明确的责任划分和目标。

Dukes小组有时候会为了保持隐蔽性而重新安排行动的继续情况。他们的2015年CozyDuke和CloudDuke活动将这特点发挥到了极致。最极端的例子就是CloudDuke活动，当时的多家安全厂商都认为Dukes都曝光了CloudDuke活动后，他们又在2015年7月继续进行了活动。所以，我们认为Dukes的主要任务很有价值，其赞助者认为行动继续比一切都重要。

就我们看来，Dukes小组对公众关注的无所谓，是因为其赞助者具有权势，并且一直紧密联系着Dukes，所以Dukes才能肆无忌惮的进行行动。我们认为唯一有权力提供这样全面的保护的就只能是国家政府。所以我们认为Dukes或者在政府内工作或直接就是效力于政府，绝不可能是犯罪团伙或其他第三方。

最后一个问题：哪个国家？我们无法负责人的证明是哪个具体的国家在支持Dukes。但是，在我们看来，所有的证据都指向了俄罗斯。而且，目前我们也没有发现能有证据来驳斥这一理论。

卡巴斯基实验室先前已经注意到，在一些Dukes木马样本中出现了俄语。我们也在很多PinchDuke样本中发现了俄语的错误信息：“Ошибка названия модуля! Название секции данных должно быть 4 байта!”，大体意思是 “模块名称有错误，数据节名称必须是4字节！”

另外，卡巴斯基还指出，根据编译时间戳，Duke木马的作者主要是在周一至周五工作，从早上6点到下午4点， UTC+0时间。对应着UTC+3时区，也就是莫斯科时间的早9点和下午7点，基本涵盖了大部分俄罗斯西部，包括莫斯科和圣彼得堡。

卡巴斯基实验室对Duke木马作者的工作时间分析也印证了我们自己的分析，以及FireEye的分析。很多GeminiDuke样本中的时间戳也支持这种根据时区的推断，这种相似性表明这个小组的工作地点位于莫斯科标志时区，详情请参考GeminiDuke技术分析部分。

最后，Dukes小组的目标有-西欧国家的外交部，西方智囊团和政府组织，甚至是俄罗斯毒枭-都与俄罗斯外交和安全政策相关。虽然，Dukes的目标看起来是全球的政府，但是我们并没有发现他们针对俄罗斯政府。但是证据不足不代表就是没有证据，这是很有趣的一点。

根据列出的证据和分析，我们认为，Duke工具集是一个独立的，掌握有大量资源的大型组织开发的（我们称之为Dukes小组），旨在为俄罗斯政府提供与外交和安全政策相关的情报，用于交换支持和保护。