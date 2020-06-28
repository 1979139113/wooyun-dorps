# Cybercrime in the Deep Web

0x00 序言
=====

深网（Deep Web）覆盖的内容包罗万象，其中包括有动态网页，已屏蔽网站（需要你回答问题或填写验证码进行访问），个人网站（需要登录凭证才能进行访问），非HTML/contextual/script内容和受限访问网络等等。但由于各种原因，Google等搜索引擎无法索引到暗网的内容。

诸如.BIT域名这一类受限访问的网站所注册的DNS（域名服务系统）根服务器不受ICANN（互联网名称与数字地址分配机构）管理。这些网站运行在非标准顶级域名的标准DNS服务器当中。想要访问这些地下网络（Darknets）需要通过Tor这类软件来进行访问。而这些地下网络活动是组成深网大部分共同的利益的基础。

**深网的用途**

聪明人在网上购买毒品的时候是不会在普通浏览器输入这些敏感的关键字的。因此，他需要一种不公开的IP地址和物理地址的匿名上网方式进行非法活动。同样的，毒品的卖家也不想在网上开店后被人查出具体位置。如果注册的域名或网站的IP地址是真实存在，那么很容易就会被查水表。

除了购买毒品的需求以外还有其它很多原因需要使用到匿名上网。比如：有人想要从政府的监控中跟他人进行秘密通讯，知情人想要向记者透露爆炸性新闻但不想暴露自己的身份，某些政治制度严格的国家的不同政见人士想要安全地向全世界告知他们国家正在发生什么事情。这些原因都致使他们用到深网的匿名功能。

另外，那些公众人物想密谋暗杀他人需要保证自身不会留下什么尾巴。其它需要保持匿名的非法服务还有类似贩卖非法护照和信用卡。同样，那些要泄露他人的私人信息的猥琐佬也要通过匿名的方式保证自身安全。

**表网 VS 深网**

讨论深网的时候一个不得不说的概念就是“表网（Clear Web）”。它与深网完全相当，能够被传统的搜索引擎索引，可以通过无需任何特殊配置的标准Web浏览器浏览Internet。这种称之为“可搜索互联网（searchable Internet）”便是表网。

**暗网 VS 深网**

很多人误解暗网（Dark Web）与深网（Deep Web）两个概念，甚至一些研究人员把它们当成等价关系。但是！！暗网不是深网！！它仅仅只是深网的一部分。暗网依赖于地下网络。在暗网，两者之间的通讯网络是受信的。Tor的“无形的互联网”项目（Invisible Internet Project（I2P））便是一个暗网系统的例子。

0x01 深网分析器
=====

深网分析器（DeWA）是为了追查恶意软件的作者，探索新的恶意威胁，提取深网中有意义的数据，搜查新的恶意软件活动等目标而设计的。

深网分析器包含五个部分：

1.  数据收集模块，负责从多个来源中搜索和保存新的URL。
2.  通用网关，解决那些私人DNS地址，并允许用户像使用Tor和I2P这些软件一样去访问隐藏的资源。
3.  页面侦查模块，负责爬取新网址。
4.  数据富集分析模块，整合从其它源的侦查信息。
5.  存储索引模块，让数据方便进一步分析。
6.  可视化分析工具。

![System Overview](http://drops.javaweb.org/uploads/images/8dd7406193e7a5e2b5c153b4608734137d8b975c.jpg)

_System Overview_

**数据分析模块**

深网分析器的第一个模块是数据收集模块，数据收集模块通过下面的主站爬取新的URL：

*   TOR和I2P隐藏的服务主机
*   Freenet资源定位器
*   .bit域名
*   非标准TLD（顶级域名）的其它域名，从已知的代理域名注册商获取顶级域名列表

我们的监测系统的数据基于：

*   用户数据，检查HTTP链接到隐藏的服务或非标准域名
*   类似Pastebin的网站，检查文本中包含深网网址的片段
*   公众论坛（reddit等一类网站），查找包含深网网址的帖子
*   包含深网域名的网站，比如deepweblinks.com，darkspider.com等
*   TOR网关的统计信息，比如tor2web.org这类网站支持用户无需安装TOR便可以访问隐藏的服务并统计每天的域名访问信息
*   I2P解析文件，作为一种加快I2P主机名解析方法，它可以从一些隐蔽的网站下载一些预先准备好的主机列表。我们可以在这个列表找到一些有趣的新域名
*   Twiter，从Twiter查找包含深网域名的URL

数据收集模块在发现新域名后生成数据索引，同时还对各个URL组件进行流量分析。这些分析操作能够使我们发现新的恶意软件活动。

**通用深网网关**

前面我们已经提到过，深网的资源很难爬取。需要通过TOR和I2P这类专用软件代替DNS和TLD作为网络地址解析工具。为了方便快速访问深网的资源，我们部署了一个Charon（一个可以使用URL发送HTTP请求到目标服务器的透明代理服务器）。

根据URL的种类，Charon连接到：

*   TOR负载均衡器
*   I2P
*   Freenet节点
*   能够解析私人TLD的私人DNS服务器

**页面侦察**

对于每个收集到的URL都要执行“侦察”操作。即尝试连接到URL并保存响应的数据。当发生错误的时候，侦察器保存所有错误信息供使用者查看错误是由域名解析，服务器端错误或传输失败等原因造成的。HTTP请求失败之后，侦察器会保存整个HTTP头部，这个头部可以用来侦察恶意软件对应的主机。当然，这种情况只是针对特定的HTTP请求。

当成功的时候，侦察器使用无界面浏览器（Headless Browser）从下载下来的页面提取相关的信息：

*   记录所有HTTP头，并追踪所有重定向链接。
*   执行网页DOM渲染（为了获取动态javascript页面）
*   获取网页的快照
*   计算网页的大小和MD5
*   提取网页的元数据：标题，标签，资源，关键字等等
*   提取网页的文本内容
*   提取网页所有链接
*   收集网页中所有email地址
*   提取URL并反馈给数据收集模块，然后作为附加数据源的索引。

**数据富集**

数据富集（Data Enrichment）由侦察的数据组成，针对每个侦察成功的页面执行以下操作：

*   检测页面的语言
*   使用Google翻译所有非英文网页
*   通过Web信誉系统针对链接进行评级分类
*   使用语义聚类算法分析生成WordCloud

聚类算法生成的WordCloud就已经包含了重要的信息。该算法的工作流程如下：

1.  记录页面上的特殊单词和每个单词词频
2.  筛选单词，只保留名词，其它如动词，形容词都去掉。名词只保留单数形式
3.  计算语义距离矩阵：这个矩阵记录词与词彼此之间的分类距离。这个矩阵称之为WordNet矩阵。WordNet矩阵测量每个单词的分类距离。例如，“棒球”和“篮球”的距离就非常接近，因为两者都属于“体育”。同样，“猫”和“狗”的距离也很相近，因为它们都是属于“动物”。而另一方面，“狗”和“棒球”的距离就很远了
4.  词集的单词距离由内向外增加。一旦我们拥有每个词对的距离，便可以创造一组具有意义相似的单词组
5.  词集使用的第一个词的字母顺序为标签标注，并计算词集中每个单词的词频
6.  使用词集里面分数前20名的标签，绘制生成WordCloud

数据富集模块让分析人员可以快速从一个网页当中获得主旨。

**存储和索引**

订阅的URL和侦察的信息都根据不同标准的索引方式保存到Elasticsearch集群。侦察信息作为每个网页文档的索引，并由Elasticsearch提供搜索功能。这种关联关键字的方式通过文本查询就可以搜索成千上万的网页。每个URL组件的URL信息也作为统计信息保存起来，它可以用于确定一个系统的主机名以及查看这个URL的流行程度。其它用途还有：给定一个主机名和参数就可以查看它第一次访问情况，或者找出哪些URL访问次数最频繁等等。

**UI和可视化**

为了访问和操作数据，我们需要借助三个不同的前端系统：

*   为了进行定性分析，我们开发了一个深网门户网站。这个工具可以方便调查人员通过不同的方式搜索深网的内容。我们提供不同的可视化效果：一个网站分类，它允许用户通过主机名，路径，字符串等方式浏览所有深网的URL。一个URL概要视图，用于显示所有收集的URL。一个侦察概要视图，提供一个单独的侦察网页用于搜索网页内容。
*   为了进行定量分析，我们依靠Kibana提供的先进数据统计功能和实时数据计算功能。它提供了一个数据挖掘标签和可视化标签。可视化视图标签根据不同的数据指标和聚合进行绘制图表。
*   对于更高级的数据检验，我们使用了IPython Notebook。它含有丰富程序库，方便嵌入到Elasticsearch集群当中检验本地数据和编译详细的报告信息。

0x02 深网的状况
=====

在本节当中，我们将展示一些用我们的系统收集到的深网应用场景。

首先先来看下在过去两年间收集到的所有现有深网网页的语言分布情况。

有两种方法可以进行语言检测：一是使用Python的第三方guess_language模块，它基于Trigram算法实现，并支持离线使用。二是使用Google翻译。在使用的时候需要比较两者的探测质量避免造成数据偏差。例如，Google翻译有“未知语言（当网页没有数据的时候）”的概念。而且默认情况下是使用英语。因此一个不慎就容易造成巨大的数据偏差。

下图显示网页语言的分布情况，在统计的时候我们已经过滤掉小于1KB的数据量的语言（因为数据量太小说不上话）。

可以看到深网网页主要以英文为主，在所有域名当中占到75%。第二是俄国，然后是法国（可能包括法国和加拿大）。

![](http://drops.javaweb.org/uploads/images/5eaab07d57d9283a0ffc377e07824e3b4f86fb04.jpg)

接下来我们看一下过去两年间收集到的所有域名的URL调用方法（HTTP，HTTPS，FTP...）。HTTP（s）协议占到了22.000。如果过滤掉这些数据，可以看到如下图所示的有趣数据：

![](http://drops.javaweb.org/uploads/images/bca89c509c03a1b5800259ee8edf820454a033e0.jpg)

超过100个站点使用了IRC(S)协议。这些都是正常的聊天服务器。当然，它们也可以作为进行违法交流场或作为僵尸网络（Botnet）的通信渠道使用。同种类型的还有运行在TOR的聊天服务器的7 XMPP（类似Jabber所使用的）域名。

**一些深网犯罪活动的例子**

深网里面提供非常好的翻译环境供人们交易商品或服务，并提供保证人们在交易时的匿名性。虽然缺乏身份证的交易虽然存在很大的风险，但同时也提供了相对的安全性。这种方式使得深网网民可以自由地贩卖交易非法商品或服务。此外，不同于地下网络犯罪，深网大多数活动都对“真实世界”起着重大的影响作用。

在这里我们无法担保这些商品或服务的真实性，只针对性讨论那些真实存在的网站广告。而且我们无法覆盖所有产品和服务，在这里主要介绍几个重要的交易类型。

**贩卖护照和国籍**

即使是假的护照或身份证也是非常好用的证件。这些证件不单单可以用于出国（包括买家不容易出现交叉），也可以用于开设银行账户，申请贷款，购买房地产等等。所以毫无疑问，护照和身份证都是一种很有价值的商品。有几个深网网站都声称它们出售正式的护照和身份证，价格在不同国家和不同卖家之间也各不相等。

这类服务很难保证说没人购买。特别是那些在异国他乡但护照身份证被骗/盗/丢失的人为了继续留在该国家可能就会购买这些非法证件。

![](http://drops.javaweb.org/uploads/images/4eaa2919e74371f81598e772e732f130fef75075.jpg)

**USA Citizenship for sale for under 6000 USD http://xfnwyig7olypdq5r.onion/**

![](http://drops.javaweb.org/uploads/images/77dc06a34a7c1872c9e0b95063f1bbc1c87c7d51.jpg)

![](http://drops.javaweb.org/uploads/images/511e3d6bb4a201efd2867fe89ccb90edab5772d6.jpg)

**Pricing information and samples for fake passports and other documents http://fakeidigyiumbgpu.onion**

参考：

1.  [USA Citizenship](http://xfnwyig7olypdq5r.onion/)
2.  [UK Passports](http://vfqnd6mieccqyiit.onion/)
3.  [Fake Passports, many countries](http://fakeidigyiumbgpu.onion/)

**盗卖帐号**

盗卖帐号绝不仅限于深网，表网地底下这种类型的交易也很常见。在过去我们写了大量关于俄罗斯和中国这方面的报告。其中，信用卡、银行账户，在线拍卖网站和游戏可能是最常见的盗卖帐号类型。

表网上不同的网站之间价格也相差甚大。但成熟的商品往往都会有一个人们普遍接受的定价标准。通常会有两种售卖方式：高质量经过已验证的帐号，但需要提供明确的帐号余额。大量未经验证的帐号，但需要保证至少一部分有效。第一种销售方式成本虽然高了一些，但可能带来更多的高质量的买家。而批发帐号售价会相对便宜一些。

![](http://drops.javaweb.org/uploads/images/e5d896f0dc218608eceeed7d414686a43baee1c9.jpg)

**Unverified accounts sold in bulk – 80% valid or replacement offered http://3dbr5t4pygahedms.onion/**

可以发现深网出售的商品都能在表网找到对应的商品。所以说表网不是没有这种类型论坛，只是深网上看起来逼格更高一些。

![](http://drops.javaweb.org/uploads/images/b42d5eba39a74f9bdf4b88575c9d970a98b7cb95.jpg)

**Replica credit cards created with stolen details http://ccccrckysxxm6avu.onion/**

参考：

1.  [http://www.trendmicro.com/cloud-content/us/pdfs/security-intelligence/white-papers/wp-russianunderground-101.pdf](http://www.trendmicro.com/cloud-content/us/pdfs/security-intelligence/white-papers/wp-russianunderground-101.pdf)
2.  [http://www.trendmicro.com/cloud-content/us/pdfs/security-intelligence/white-papers/wp-russianunderground-revisited.pdf](http://www.trendmicro.com/cloud-content/us/pdfs/security-intelligence/white-papers/wp-russianunderground-revisited.pdf)
3.  [http://www.trendmicro.com/cloud-content/us/pdfs/security-intelligence/white-papers/wp-thechinese-underground-in-2013.pdf](http://www.trendmicro.com/cloud-content/us/pdfs/security-intelligence/white-papers/wp-thechinese-underground-in-2013.pdf)
4.  [Stolen Paypal accounts](http://paypal4ecnf7eyqa.onion/)
5.  [Unverified stolen accounts](http://3dbr5t4pygahedms.onion/)
6.  [Replica stolen credit cards](http://ccccrckysxxm6avu.onion/)

**暗杀服务**

这也是深网里面最黑暗的服务之一，这类服务提供暗杀服务和杀手出租服务，如果放在表网上那绝对是愚蠢至极。深网存在几个这样的服务提供商，而且在他们网站也公开说明他们是如何保证业务的机密性。一个网站明确说明：它们不提供杀手们过去的工作证明，以及以往的客户反馈情况和暗杀成功的证明。相反，他们使用比特币作为信誉象征。最后，只有当杀手展开暗杀并提供证明，才能获得佣金。

![](http://drops.javaweb.org/uploads/images/b475e8a56b42c74ab4d2c61b493a3096b5092444.jpg)

**C’thulu Resume – Assassination Services for Hire http://cthulhuuap7ch47k.onion**

从上图可以看到，服务的价格随着目标的死亡方式，受伤方式和地位的不同而不同。最近，Ross Ulbricht就因利用丝绸之路进行贩毒被判刑而企图雇佣五个杀手干掉他的合伙人。

还有另外一种不同的服务，称之为“众包暗杀”。在DeadPool这个网站里面，用户提出潜在的暗杀目标，然后其他人向“死亡之池”扔比特币。暗杀者预测目标大概什么时候以什么方式死亡。如果这个人确实死了，而且符合预测的结果，那么暗杀者就可以获得这笔钱。至今为止已经提出了四个名字，然而还没有钱进入池中。我们可以猜测这是一个钓鱼网站。

![](http://drops.javaweb.org/uploads/images/0cd08638614886db434de1fff4b29b2b96812954.jpg)

**Deadpool – Crowd Sourced Assassination http://deadpool4x4a25ys.onion**

参考：

1.  [http://www.wired.com/2015/02/read-transcript-silk-roads-boss-ordering-5-assassinations/](http://www.wired.com/2015/02/read-transcript-silk-roads-boss-ordering-5-assassinations/)
2.  [Contract Killers (C’thulu Resume)](http://www.wired.com/2015/02/read-transcript-silk-roads-boss-ordering-5-assassinations/)
3.  [Crowdsourced assassination](http://deadpool4x4a25ys.onion/)

**比特币和洗钱**

比特币（Bitcoin）本身是为了匿名流通而设计的货币。因此它经常使用在购买非法商品或服务上面（当然也可以购买合法的东西）。虽然只要不把比特币跟你的真实身份打上挂钩就可以保证在交易的匿名性。但是，每笔比特币的交易都是完全公开的。所以，尽管比较困难，调查人员追查资金的流通情况还是可行的。

有一些服务可以提高你的货币在系统中的匿名性，使得这些货币流通情况更难以追查。这些服务通常把你的货币在网络蜘蛛上进行微交易后再返回到你手上。在这个过程你会丢失少许货币（通常减去少量的手续费），但可以使得你的交易过程变得更加难以追查。

![](http://drops.javaweb.org/uploads/images/7ca630d6a1ab1ac009e55d93dcbda630f805aebb.jpg)

**EasyCoin – Bitcoin laundery service http://easycoinsayj7p5l.onion**

比特币洗钱服务可以提高资金在比特币系统流通的匿名性。但人们最希望的还是从系统从把比特币通过其它方式转换为现金。深网有转换现金的匿名服务：它们基本都是通过Paypal，ACH，西联汇款或者直接发送邮件给你现金。

![](http://drops.javaweb.org/uploads/images/b956a40b4d11f43a0d0d516a73c5c138d483b03a.jpg)

**WeBuyBitcoins – Exchanging Bitcoin for cash or electronic payments http://jzn5w5pac26sqef4.onion**

像WeBuyBitcoins这类网站在表网提供非匿名但相对较高的汇率的交易。对于犯罪分子来说可能原意承担更大的风险获得更多的现金。另外还有一种选择是：使用比特币购买假币。

![](http://drops.javaweb.org/uploads/images/7173b3e0e38ec90d41e3a04b82adb163d87562c5.jpg)

**Buying counterfeit 20 USD for approximately half the price of face value http://usjudr3c6ez6tesi.onion**

参考：

1.  [Bitcoin used to by a Tesla Model S](http://www.wired.com/2013/12/tesla-bitcoin/)
2.  [EasyCoin – Bitcoin Wallet with free Bitcoin Mixer / Laundery](http://easycoinsayj7p5l.onion/)
3.  [OnionWallet – Bitcoin Wallet with free Bitcoin Mixer / Laundery](http://ow24et3tetp6tvmk.onion/)
4.  [WeBuyBitcoins – Sell Bitcoins for Cash (USD), ACH, WU/MG, LR, PayPal and others](http://jzn5w5pac26sqef4.onion/)
5.  [Counterfeit $20 USD / Euro Bills](http://usjudr3c6ez6tesi.onion/)
6.  [Counterfeit $50 Euro Bills](http://y3fpieiezy2sin4a.onion/)
7.  [Counterfeit $50 USD Bills](http://qkj4drtgvpm7eecl.onion/)

**泄漏政府，执法部门，法人的信息**

黑客文化是一种一群志同道合的人组成的松散式或封密式的组织。由于这种性质，组织之间很容易发生竞争冲突。发生冲突时“Dox”对方是一种常见的做法，Dox是指通过计算机检索，黑客等行为把对方的个人信息发布到网络上。获取对方个人信息方法有很多，但通常会结合公共数据，社会工程学和黑客攻击几种方法收集对方的个人信息。

![](http://drops.javaweb.org/uploads/images/292603dee56500c2955a9aa0058586577d65a716.jpg)

**Cloudnine Doxing site – note it requests SSN, medical & financial info and more http://cloudninetve7kme.onion**

但是Dox现象不仅限于黑客之间，针对敌手公司，名人，公众人物的Dox也是很常见的。暴露的信息也不仅限于黑客获取到的信息，也可能是内部人员透露的。一般情况下都把信息提交到维基解密（Wikileak）上。深网也有这种类型的网站，允许提交这些信息。

很难保证这些信息的真实性。但通过泄漏的信息包括：生日，SSN，个人email地址，手机号码，居住地址等等。Cloud Nine这个网站列出了一些可能“Dox”信息：

*   几个FBI特工
*   Bill，Hillary Clinton，Barack，Michelle Obama，Sarah Palin，美国参议员还有其它一些政府人员。
*   Angelina Jolie，Bill Gates，Tom Cruise，Lady Gaga，Beyonce，Dennis Rodman等名人。

![](http://drops.javaweb.org/uploads/images/4f491d44b5a366d52ed1323ab8ec50b54435f783.jpg)

**Apparent personal email account of Barack Obama (unverified) http://cloudninetve7kme.onion**

![](http://drops.javaweb.org/uploads/images/c8fb10e6639bccc62cbcafbe65c780e2f8cca7b8.jpg)

**Apparent leaks of LEA (unverified) http://cloudninetve7kme.onion**

![](http://drops.javaweb.org/uploads/images/6708533f9125eeb494c6d5602f2403dc85ec747e.jpg)

**A leak for Kim Kardashian among other hacker related dox http://cloudninetve7kme.onion**

参考：

1.  [Doxing archive](http://cloudninetve7kme.onion/)
2.  [Wikileaks clone](http://gjlng65kwikileax.onion/)
3.  Wikileaks submission portal
4.  [Possible Judge Forrest leak](http://uhwikih256ynt57t.onion/wiki/index.php?title=Dox_-*Katherine\_Bolan\_Forrest*(Silk\_Road\_Judge)&oldid=5764)

**病毒**

正如前面提到过的，深网最常见的就是贩卖毒品和武器。但在这篇文章中我们不打算深入探讨这些细节，因为已经有很多文章报告了深网贩卖病毒的事情。但我们想强调的是，即使是运维“丝绸之路”贩卖毒品的Ross Ulbricht最近也被判无期徒刑。贩卖毒品对于本文分析深网的分量来说并不是很重要。

深网里面贩卖的毒品类型众多，有烟草，大麻，迷药，可卡因等等。

![](http://drops.javaweb.org/uploads/images/203b78433af3fb9d865c99e0a677ddc84ab91c2b.jpg)

**The Peoples Drug Store – selling Heroin, Cocaine, Ectasy and more http://newpdsuslmzqazvr.onion**

![](http://drops.javaweb.org/uploads/images/607ae6563a4f22eda091a10683c0f8020ce90970.jpg)

**Grams – the Deepwebs search engine for drug http://grams7enufi7jmdl.onion**

除了专门的商店和讨论外，还有一个非常受欢迎的网站“Grams”。网站风格有些类似Google，而且提供简单的搜索引擎允许搜索毒品。它在深网里面已经成为那些想购买毒品的人的旗帜性网站。

我们甚至发现TOR里面有些网站还提供大麻的培植环境：现场的温度，水分，还有植物的生命周期。

![](http://drops.javaweb.org/uploads/images/271c386b76b11e24b2b17566b31568a6d2c03207.jpg)

**Growhouse – showing temperature and live streaming of Cannabis plant http://growboxoo2uacpkh.onion**

![](http://drops.javaweb.org/uploads/images/844f1151129ca9be1ec4cb9f0ed9aa813f7803da.jpg)

**Drugs dealer in the Deep Web**

我们只所以要在这一节介绍深网里面的毒品报告是因为想强调：就像丝绸之路一样，它会记录下你的犯罪行为。深网根本上并不是一个好的解决方案。一方面买家希望向你购买毒品，另一方面还需要有卖家提供货源。市场和论坛只是作为一个交易转接点，你要是不想使用它，那么只要商品的双方需求量够大，立刻会有其它市场伴随需求而诞生。

参考：

1.  [http://www.forbes.com/sites/katevinton/2015/05/29/ulbricht-sentencing-silk-road/](http://www.forbes.com/sites/katevinton/2015/05/29/ulbricht-sentencing-silk-road/)
2.  [Contraband Tobacco](http://cigs7cviqbi4bvuy.onion/)
3.  [Cannabis](http://smoker32pk4qt3mx.onion/)
4.  [Psychedelics](http://ll6lardicrvrljvq.onion/)
5.  [Heroin, Cocaine and others](http://newpdsuslmzqazvr.onion/)
6.  [Grams – Deep Web drug search engine](http://grams7enufi7jmdl.onion/)
7.  [Live feed from a Cannabis Growhouse](http://growboxoo2uacpkh.onion/)
8.  [Expert Insight video Series – The Deep Web](http://www.trendmicro.com/vinfo/us/security/news/cybercrime-and-digital-threats/the-deep-webanonymizing-technology-good-and-bad)

**恶意软件**

深网和恶意软件之间在许多方面上能够完美结合在一起。特别是当使用深网作为C&C控制服务器基础设施使用的时候能够利用TOR和I2P强大的加密功能隐藏位置信息保证网站和服务的匿名性。这使得调查人员很难使用传统的方式检查服务器IP地址和登录详情等等。此外，这些网站和服务使用起来很简单。所以不必惊讶为什么那么多网络犯罪分子使用TOR作为C&C。通常恶意软件捆绑了TOR的客户端。这种趋势最早在2013年开始，当时MEVADE恶意软件还造成了TOR流量剧增，2014年之后流行的是类ZBOT恶意软件家族。

举个例子，VAWTRAK恶意软件是一种通过钓鱼邮件进行扩散的银行木马。每个样本都使用C&C服务器提供的IP地址列表进行通讯，IP地址列表向TOR主机网站下载（通常是一个icon文件，一般命名为favicon.ico）。这种方式的好处是保证犯罪服务器的匿名性。但这不是所有人都能访问，只有那些受到病毒感染的系统才能访问C&C服务器。

![](http://drops.javaweb.org/uploads/images/944bcf0ae88711230cfc406d7c6f40465bb9bee6.jpg)

**Vawtrak C&C showing the legitimate looking Favicon http://4bpthx5z4e7n6gnb.onion/favicon.ico**

web服务器通过favicon.ico文件配置C&C控制服务器（大多数运行在openresty/1.7.2.1）。我们可以通过搜索这些网站的完整列表下载每天最新的C&C。

![](http://drops.javaweb.org/uploads/images/144dca1a882ad5d635e3617137c60ebb4c434fa9.jpg)

**Example of fetched HTTP headers from C&Cs**

![](http://drops.javaweb.org/uploads/images/b974554dcbec1bdec491d29a93b0e45643636186.jpg)

**Identified TOR-based C&Cs (1)**

![](http://drops.javaweb.org/uploads/images/51af488d1d840891af194d186d9c1564d94e0eda.jpg)

**Identified TOR-based C&Cs (2)**

另一个使用深网的恶意软件是CryptoLocker。CryptoLockeree是一款ransomware勒索软件的变种，它通过加密受害者的个人文档和资料，并在受害者再次访问的时候重定向到它的网站以达到勒索目的。CryptoLocker可以自动调整付款页面的语言和支付手段。TorrentLocker是CryptoLocker的变种，它使用TOR作为主机，并使用比特币作为支付方式。这就说明了为什么犯罪分子为什么要使用深网作为基础设施，因为它确实更加安全。下面的截图是深网分析器捕获到的两种语言的付款页。

![](http://drops.javaweb.org/uploads/images/da3853a49cfb7635b36a971e0e97cf99b1d3ac06.jpg)

![](http://drops.javaweb.org/uploads/images/93c7d139e8861e992b11598fbfbb02ea0030c40a.jpg)

**Cryptolocker C&C automatically formatted for a victim in Taiwan and Italy http://ndvgtf27xkhdvezr.onion**

![](http://drops.javaweb.org/uploads/images/8aed804144511e26cf9a593dd9fb0f7eb164f59a.jpg)

**Breakdown by Victims and Countries**

下面是一个有关恶意软件盗取机密信息的例子。在我们的搜索方法当中，我们使用一个最近和最短的时间窗口作为查询字符串，这样我们可以快点发现深网里面新的威胁。

在这个例子中，xu和xd两个参数在过去一周人气剧增。xu关联超过1700个的字典值并组成二进制对象文件。进一步观察发现，xu使用NionSpy窃取授权凭证（通常是网上银行等），然后收集键盘记录并发送到深网中。与此同时，xd用于注册感染新的僵尸网络。注册信息包含受害者机器名和操作系统版本号，通信的参数类似下面的JSON字符串：

```
[REDACTED]2xx.onion:80/si.php?xd={“f155”:”MACHINE IP”,”f4336”:”MACHINE NAME”,”f7035”:”5.9.1.1”,”f1121”:”windows”,”f6463”:””,”f2015”:”1”}

```

通过泄漏出来的数据收集分析注册相关的信息，构建显示每天新增的受害者图表。

![](http://drops.javaweb.org/uploads/images/19b99bd41b8e9a2cc5a0a9c4d21e21a1fc7b05cc.jpg)

**Automated Analysis on Prevalent Query-String Parameters**

![](http://drops.javaweb.org/uploads/images/2f4aa23ca11dd752b553a3f84f2e0f2c39293842.jpg)

**Number of new Infections (and Leaked data, in bytes) per day.**

最后值得一提的是：一款名为Dyre的木马将I2P作为C＆C服务器的备份选项。正常情况下则使用表网的DGA。这个木马作为一个BHO的MiTMs运行在浏览器的网上银行上。攻击者可以通过后门访问受到感染的受害者银行门户。DeWA介绍这个恶意软件的时候说到：在过去的6个月间，受到I2P感染的受害者的数量明显增加。

![](http://drops.javaweb.org/uploads/images/6e5690752c0bf9e34f35db8fe7f260859ebb0cca.jpg)

**Traffic to Dyre’s I2P infrastructure.**

参考：

1.  [http://blog.trendmicro.com/trendlabs-security-intelligence/the-mysterious-mevade-malware/](http://blog.trendmicro.com/trendlabs-security-intelligence/the-mysterious-mevade-malware/)
2.  [http://blog.trendmicro.com/trendlabs-security-intelligence/defending-against-tor-using-malware-part-1/](http://blog.trendmicro.com/trendlabs-security-intelligence/defending-against-tor-using-malware-part-1/)
3.  [http://blog.trendmicro.com/trendlabs-security-intelligence/defending-against-tor-using-malware-part-2/](http://blog.trendmicro.com/trendlabs-security-intelligence/defending-against-tor-using-malware-part-2/)
4.  [http://blog.trendmicro.com/trendlabs-security-intelligence/steganography-and-malware-why-and-how/](http://blog.trendmicro.com/trendlabs-security-intelligence/steganography-and-malware-why-and-how/)
5.  [Vawtrak / Neverquest C&C](http://4bpthx5z4e7n6gnb.onion/favicon.ico)
6.  [Cryptolocker C&C](http://ndvgtf27xkhdvezr.onion/)