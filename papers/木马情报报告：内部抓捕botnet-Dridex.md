# 木马情报报告：内部抓捕botnet-Dridex

**Author: AnubisNetworks****from:https://www.anubisnetworks.com/dridex-botnet-report**

0x00 执行概括
=====

2015年3月， Anubis网络实验室开始着手于分析Dridex木马样本，以便：

1.  确定与botnet相关的感染；
2.  理解其通讯信道的复杂程度；
3.  枚举出所有的可利用漏洞。

根据研究，我们总结出了下面的几点：

1.  Dridex部署了一个混合型的点对点网络，通过不同的网络层以避免被取缔；
2.  Dridex生态系统由少量独立的botnet构成，其攻击目标主要是不同地区的金融机构；
3.  目前，Dridex主要从大量的在线服务中窃取凭证，包括多个国家的银行服务；
4.  Dridex P2P网络协议中存在一些漏洞，允许bot通讯被拦截，也可用于破坏和取缔botnet；
5.  通过逆向Dridex通讯协议，我们部署了一个流氓节点，可以窃听bot与超级节点（admin_nodes）之间的botnet通讯，从而枚举所有的bot，并确定botnet的感染散播情况。

0x01 范围
=====

本文详细地介绍了Dridex木马在受感系统上运行并上线后，所使用的通讯信道，P2P网络，加密方法以及相关的C2基础设施。我们主要研究了木马的CC通讯方法，并且确定了木马用于支持监控功能的协议上都存在哪些漏洞。

本文中没有记录木马的传播机制，也没有详细介绍受感染系统上的木马行为，而只是谈到了木马的网络通讯以及支持这些通讯的各种操作（比如，加密操作）。与Dridex相关的botnet感染扩散情况并不在本文的范围内。

本文中也没有列出任何属性标识。

0x02 Dridex 综述
=====

Dridex最初出现于2014年11月。Dridex实际是Bugat、Geodo、Feodo和Cridex木马的升级版本。

Dridex主要是通过邮件传播，利用MS Word文档作为附件，一旦用户打开了这个附件，木马就会下载第二阶段的有效载荷来感染系统。

Dridex木马的主要目标是家用银行用户。这个木马还具备多种功能，包括浏览器中间人攻击，键盘记录器，代理和VNC。木马最突出的特点就是使用了点对点（P2P）网络和加密的通讯信道。

根据木马操作员的标记，我们发现Dridex生态系统由8个botnet组成。在这些botnet中，我们确定了主botnet可能是120， 125，200， 220和320，并且121，122和123也似乎是botnet 120的别名，或至少是从120的逻辑层上隔离出来的，因为它们使用了相同的CC系统。

目前，botnet 120，200和220是最活跃的，主要的感染活动都分布在欧洲、北美和亚洲。

Dridex的bot主控非常活跃，一直在发动新的活动来攻击不同地方的目标，并利用新的应对策略和CC系统来加强botnet基础设施。

![](http://static.wooyun.org//drops/20160422/2016042203022713957.com/blob/foeaaawobgd/06shzid9uw6ucbtgrdupya?s=wyoyaawai16p)

Dridex 220分布（取样）

0x03 点对点网络
=====

这个botnet使用了一个混合型的P2P网络来进行通讯，也使用了不同类型的节点和协议。这一部分就对其进行了详细地解释。

网络结构
----

其网络结构如下：

![](http://static.wooyun.org//drops/20160420/2016042007220182919.com/blob/foeaaawobgd/yh2vi1ix6gw5r7i0ovedjq?s=wyoyaawai16p)

由下面的元素组成：

*   bots - bot是最常用的网络元素。指的是那些没有直接连接互联网，但是启用了防火墙或NAT的受感染系统；
*   节点 - 节点指的是那些直接连接了互联网，没有用防火墙或NAT，并且成功提升到这节点层的受感染系统；
*   超级节点admin_nodes - 超级节点由被攻破的服务器组成，一般是作为C2后端和节点之间的一个代理层；
*   服务器 - 服务器是木马首先使用的C2通信点，一般是硬编码到木马文件中，然后木马与服务器通信获取初始的点列表；
*   c2后端 - CC后端是由botnet主控负责操作的。注意，我们还没有发现网络的这一层，所以，我们只是推测这一层是存在的。

网络通讯
----

木马在不同时间使用的网络协议和加密方法也是在变化的。本文中只涵盖了我们最近发现的方案，这个方案是从2015年4月30日开始投入使用。

### 网络协议

Dridex使用了XML信息来进行所有网络元素之间的通讯。但是，这些信息都会通过不同的加密协议，使用独立的通讯上下文进行封装。在发送XML信息时，Dridex会使用下面的几种协议：

*   **服务器通讯协议**- 在感染系统阶段使用，用于获取botnet中初始的节点列表 ；
*   **节点间通讯协议**- 当通讯中不包含与C2相关的信息时使用，比如，当信息仅仅是用于维持网络结构时；
*   **C2通讯协议**- 当信息需要前往C2后端时使用，比如，当发送窃取数据或请求新命令时。

#### 服务器通讯协议

服务器通讯使用了HTTPoverSSL协议来进行传输。HTTP请求的主体中包含有一个4字节的异或秘钥和异或加密的XML信息。下图显示的就是HTTP的主体结构：

![](http://static.wooyun.org//drops/20160420/2016042007220484507.com/blob/foeaaawobgd/4y-t_3zh88rcwt0zrkjjxw?s=wyoyaawai16p)

*   **异或秘钥（4字节）**- 异或加密数据时使用的秘钥；
*   **异或加密的XML信息**- 通过使用秘钥，异或信息的每个4字节来加密XML信息；

#### 节点间通讯协议

节点间通讯直接通过TCP socket进行传输，并且使用了下面的结构：

![](http://static.wooyun.org//drops/20160420/2016042007220651454.com/blob/foeaaawobgd/a5kljgzwc5p8wpnrun-_qq?s=wyoyaawai16p)

*   **随机数据（124字节）**- 随机数据是通过使用Windows API函数 CryptGenRandom生成的；
*   **校验和（4字节）**- 校验和是根据随机数据得到的。随机数据中的124个字节里包含有314字节的整数值总和，反向的这个总和就是校验和；下面是一个Python实现例子：
    
    ```
    struct.pack(">I",-uint32(sum(struct.unpack('<31I',rnd1))))[::-1]
    
    ```
*   **数据长度 （N字节）**- 数据长度以一种特殊的格式储存在两个签名短整数中（Signed Short Integer，双字节）。第一个短整数中包含有数据长度除以30000后的反向结果，第二部分中包含了余数除以30000后的反向结果。如果相除以后的结果小于1，第一个短整数就会使用一个随机的双字节，导致的结果会是一个负值。下面是一个Python实现例子：
    
    ```
    datalen=len(xoredpayload+xorkey) 
    hpart=datalen/30000 
    lpart=datalen%30000
    if (hpart):
    hpartbytes=struct.pack('<h',-hpart) 
    else:
    hpartbytes="\x00\x00" 
    if (lpart):
    lpartbytes=struct.pack('<h',-lpart) 
    else:
    lpartbytes="\x00\x00" 
    lenbytes=hpartbytes+lpartbytes
    
    ```
*   **异或秘钥（4字节）**- 用于异或加密的秘钥。 这个秘钥是一个随机的4字节秘钥，通过Windows API函数CryptGenRandom获得；
    
*   **异或加密并使用gzip压缩的XML信息**- 需要发送到节点的XML信息会使用gzip格式压缩。然后，压缩数据的每个4字节都会使用异或秘钥进行加密。
    

#### C2 通讯协议

C2通讯会通过HTTPoverSSL协议进行传输（bot并不会检查SSL证书的有效性）。

每当bot向C2发送信息时，HTTP请求中就会包含一个特殊的标头，用于识别请求bot的唯一ID。下面就是bot向C2发送请求时的一个HTTP标头示例：

```
Id: ABCDEFG_12345678901234567890123456789012

```

HTTP请求主体中包含有下列格式的XML加密信息：

![](http://static.wooyun.org//drops/20160420/2016042007220893543.com/blob/foeaaawobgd/uosahqslkljx4ynbrehxyw?s=wyoyaawai16p)

*   **RSA 签名（128字节）**- HTTP 请求中其余部分的RSA签名使用了本地生成的Bot RSA私钥；
*   **Microsoft SimpleKey Blob 标头（12字节）**- Microsoft SimpleKey Blob的标头。这是Microsoft Crypto Library用于交换加密秘钥时使用的一部分格式。在这个例子中，使用了下面的字节：
    
    ```
    Ox010200000168000000a40000
    
    ```
*   **RSA 加密的RC4 秘钥（128字节）**- 给每条要发送的信息生成一个随机的RC4秘钥，并使用硬编码在木马文件中的C2公钥来加密这个秘钥。
    
*   **RC4加密并压缩后的XML信息**- 使用gzip格式压缩要发送到节点的XML信息。然后使用为这条信息生成的RC4秘钥来加密压缩后的XML。
    

#### 信息协议

其中一个信息使用了Base64编码，并且会通过XML信息发送。这个信息就是秘钥信息（比如，软件模块，初始节点列表，XML信息内容），秘钥信息会经过双重加密并使用C2秘钥来前面。在解码了base64加密后，其内容就会按照下面的格式储存：

![](http://static.wooyun.org//drops/20160420/2016042010012423912.com/blob/foeaaawobgd/baupao8wafobllz5mixfig?s=wyoyaawai16p)

*   **RSA 签名（128字节）**-其他有效载荷（异或秘钥和异或后的有效载荷）使用的RSA签名；
*   **异或秘钥（4字节）**- 用于加密数据的秘钥，这是一个随机四字节秘钥；
*   **异或并使用gzip压缩的数据**- 发送到节点的数据会使用gzip格式进行压缩。然后，压缩数据的每4个字节都会使用异或秘钥进行异或；

### 通讯流程

这一部分描述了bot首次感染后，进行的典型通讯流程，在感染后，bot首先会尝试提升到节点，同时还在向C2发信号。

#### 加入网络

下图表示的是在系统感染后，一直到完全加入Dridex点对点网络期间，所发生的通讯顺序。

![](http://static.wooyun.org//drops/20160420/2016042007221299086.com/blob/foeaaawobgd/mkv6xhccj084k-yaicbcpw?s=wyoyaawai16p)

当一个新系统感染后，系统会执行下面的操作（接下来我们会详细解释不同XML信息的内容）：

1.  第一阶段，一个恶意的MS Office文档会通过一个常规的HTTP请求从攻破的web服务器上下载第二阶段的木马；
2.  第二阶段木马会使用Dridex 服务器协议（前面说明过）来连接其硬编码服务器，并发送一个 “loader”XML信息，请求第三阶段的木马/bot DLL和初始的节点列表；
3.  bot按顺序连接列表上所有的节点，直到出现正确响应的节点，然后发送一条 “becaon with pub key”XML信息，这样，在响应中就会包含bot的外部IP地址；
4.  bot使用C2通信协议，发送一条“beacon with pub key” XML信息，其中包含有从节点上获取到的IP地址，以及bot的RSA秘钥；
5.  bot向显示空白配置文件哈希值的C2服务器发送一条“beacon with empty hash” XML信息。这样，C2就会响应，并发送最新的配置文件，想要bot执行的命令，已经最近的节点列表；
6.  bot向节点发送一条 “beacon” XML信息。节点会响应并回复可用的模块列表；
7.  bot发送一条“beacon with get_module”XML细心，从C2请求可用的模块；
8.  bot发送一条“beacon with get_module”XM信息，从节点请求可用的模块；

#### 提升至节点

一旦bot注册到了网络上（在bot成功把公钥发送到服务器并获取到了积极的回应后），bot就会开始提升至节点的过程，同时继续通过相关的请求来获取配置文件和可用的模块。这个过程会允许没有启用NAT的bot作为P2P网络上的节点工作。下图显示了在提升过程中发生的通讯顺序：

![](http://static.wooyun.org//drops/20160420/2016042007221580799.com/blob/foeaaawobgd/txbkojli7exqoxt3jkawkg?s=wyoyaawai16p)

在这一过程中，发生了下面的这些通讯：

1.  bot将 “checkme”XML信息发送给从节点列表中随机选择的3个节点，并重复这一过程直到测试完所有的节点；
2.  每个获取到 “checkme”请求的节点会尝试连接到 “checkme”XML信息中列出的TCP端口，并发送一个 “Ping”XML信息；
3.  如果节点接收到了 一条“Pong”XML信息，为了回应 “Ping”，节点会发送一条“beacon with newnode”XML信息给C2，警告检测到了一个新的潜在节点；
4.  C2会响应合适的 “Info” XML信息，发送到新的节点上。这条信息中一般会包含新节点需要使用的admin_node；
5.  节点会连接bot并发送 “Info”XML信息；
6.  bot会直接连接到admin_node，并发送一条“beacon with pub key”XML信息，没有IP，包含有bot的公钥；
7.  最后，新节点会直接连接到admin_node，并向包含有XML类型节点字段的admin发送一条“node beacon” XML信息；

#### 周期性签入

bot成功注册到网络后，会尝试提升至节点，下载配置文件和所有可用的模块；还会定期嵌入到C2上，从而：

*   发送任意的命令输出结果；
*   发送窃取的数据；
*   检查配置文件的更新
*   检查模块的更新

下图中就是感染后的周期性通讯：

![](http://static.wooyun.org//drops/20160420/2016042010012638223.com/blob/foeaaawobgd/oiki5a73of4kcj_gorikfg?s=wyoyaawai16p)

一旦受感染的系统注册成为一个新的bot，系统就会执行下面的操作来开始节点提示过程：

1.  bot向节点发送一条 “beacon”XML信息。如果bot IP地址更改了或出现了新模块，在响应中，bot就能获取到更新后的IP地址。如果接收到新模块或新版本，bot就会发送信息请求新的模块；
2.  bot向C2发送一条 “beacon”XML信息。这条信息中包含一个字段，里面有窃取到的数据，命令输出和调试信息。响应中可能包含有更新后的配置文件，额外的命令，更新后的节点列表或更新后的admin_node列表。

### 信息格式

Dridex使用了前面提到的协议来传输和接收XML信息。分布在不同国家的bot通过接收这些信息和响应，从而让botnet发挥作用。这些信息可以分为下面的几类：

*   服务器信息
*   节点信息
*   C2信息

这些信息大多都有共同的属性。相关性最强的有：

*   unique: 唯一的bot ID，根据计算机名称和注册表键值的哈希创建，格式如下_；
*   botnet: bot属于botnet。每个botnet都是一个独立的Dridex P2P网络，具有独立的配置，模块，bot，节点和admin_node。
*   version: bot模块的软件版本；
*   system: 操作系统版本；
*   type: 受感染系统的角色（bot还是节点）

#### 服务器信息

下面的这些信息会在Dridex第二阶段组件和硬编码服务器之间传递。这些信息使用了前文提到的服务器通讯协议进行加密和传输。

**Loader**

当第二阶段木马加载到系统时，木马会连接到硬编码IP地址（服务器），并下载bot和节点列表。

下面是请求和响应示例：

![](http://static.wooyun.org//drops/20160420/2016042010012777789.com/blob/foeaaawobgd/-aszpnfr4ikobui346mltg?s=wyoyaawai16p)

![](http://static.wooyun.org//drops/20160420/2016042007222192800.com/blob/foeaaawobgd/7ifdcdb_-ved8ybnqqv4ua?s=wyoyaawai16p)

请求中会发送bot ID，用于以后与botnet和安装在系统上的软件进行通讯。

响应中包含有节点和适用于系统架构的bot模块。节点列表和模块都使用前面提到的信息协议进行了加密。

#### 节点信息

下面的信息会在bot与节点或节点与节点之间传递。这些信息使用了前文提到的节点间通讯协议进行加密和传输。

**Beacon**

bot会向节点发送beacon信息，在初始感染后，每隔20分钟就进行一次周期性签入。在beacon信息中，没有公钥，但是包含有bot最后已知的外部IP地址。响应中会包括节点已知的模块和bot 的IP地址，如果与请求中发送的不同的话。

下面是请求和响应示例：

![](http://static.wooyun.org//drops/20160420/2016042007222394527.com/blob/foeaaawobgd/0bb8way6cm2zgo6aj7-jkg?s=wyoyaawai16p)

**Beacon with pub key 信息**

beacon with pub key信息是受感染系统在首次与节点通讯时，向节点发送的。这条信息不同于常规的beacon信息，因为其中包含有bot的公钥，并且没有bot的外部IP地址。

这条信息会导致节点发送下面的信息：

1.  bot当前的外部IP地址
2.  节点已知的最近模块

下面是请求和响应示例：

![](http://static.wooyun.org//drops/20160420/2016042010013023604.com/blob/foeaaawobgd/ya-qhi5cnehqcdelj2i0cw?s=wyoyaawai16p)

**Beacon with get_module 信息**

beacon with get_module信息用于从节点中请求一个软件模块。节点会储存并传输软件模块，从而避免C2服务器上加载额外与软件模块传播相关的流量。

下面是请求和响应示例：

![](http://static.wooyun.org//drops/20160420/2016042010013287363.com/blob/foeaaawobgd/hvggv9ebuqgkjxl_mhmmqg?s=wyoyaawai16p)

**Checkme 信息**

在提升至节点过程中，bot会用到checkme信息，用于请求一个现有节点检查这个bot是否能直接连接到网络（没有防火墙，没有NAT），是否有资格成为一个节点。

下面是请求和响应示例：

![](http://static.wooyun.org//drops/20160420/2016042007223059734.com/blob/foeaaawobgd/pu3hd47fltj9kmymspg8mq?s=wyoyaawai16p)

**Ping 信息**

在提升至节点过程中，当节点接收到了bot发出的checkme请求，节点就会尝试向checkme信息中指定的端口发送一个ping信息。如果节点接收到了pong信息，那么bot就有资格成为一个节点。

下面是请求和响应示例：

![](http://static.wooyun.org//drops/20160420/2016042007223397230.com/blob/foeaaawobgd/xxxyafpni01afvntcdszkw?s=wyoyaawai16p)

**Info信息**

Info信息会发送给新的节点，通知它们它们应该联系哪个admin_node。

下面是请求和响应示例：

![](http://static.wooyun.org//drops/20160420/2016042010013567679.com/blob/foeaaawobgd/mhlz_kpfki_hjximib72ya?s=wyoyaawai16p)

Info标签中的主体使用了Info协议进行编码，并且包含了bot要使用的admin_node，格式如下：

```
<root><admin_node>1.2.3.4:443</admin_node></root>

```

#### C2 信息

下面的信息会在bot和C2之间传递。这些信息会使用HTTPS协议发送通过一个极点，其中包含有一个HTTP标头，在标头中有bot ID，然后节点在不检查的情况下就会把信息转发给admin node。HTTP请求的主体会使用C2通讯协议进行加密。

**Beacon**

在发送到C2的信息中，关键部分就是beacon。在bot初始感染和提升过程完成后，bot就会通过发送如下的一个beacon，周期性的签入C2。如果C2上配置文件的哈希与bot发送的哈希不同，新的配置文件就会通过响应返回到bot。

![](http://static.wooyun.org//drops/20160420/2016042010013741926.com/blob/foeaaawobgd/wvrrzby71ur-qe9zypjc5q?s=wyoyaawai16p)

**Beacon with pub key**

当bot在注册过程中首次与C2通讯时，bot会发送一条包含有公钥信息的beacon。这个beacon种包含有bot密码对儿的公钥，但是没有包含配置文件的哈希。在响应中，bot会接收到一个公钥。但是，收到的公钥似乎没有在任何通讯加密或签名有效性例程中使用。

![](http://static.wooyun.org//drops/20160420/2016042010013961604.com/blob/foeaaawobgd/stamgl-7b-atix2rc2jtmq?s=wyoyaawai16p)

![](http://static.wooyun.org//drops/20160420/2016042010014266148.com/blob/foeaaawobgd/fvlfsx5_i-bntaunlvguoa?s=wyoyaawai16p)

**Beacon with empty hash**

在bot的初始注册过程中，bot会发送一个包含空哈希值的beacon。在响应中，服务器会发送当前的配置文件和一开始需要执行的一些命名。

![](http://static.wooyun.org//drops/20160420/2016042007224448995.com/blob/foeaaawobgd/zfsqadzxbupjcffgmbjpzg?s=wyoyaawai16p)

![](http://static.wooyun.org//drops/20160420/2016042010014436479.com/blob/foeaaawobgd/5gjyyhlqkxbtv-u0iyifcg?s=wyoyaawai16p)

**Beacon with get_module**

在bot初始注册的过程中，或每当bot找到新的可以软件模块时，bot就请求向节点或服务器来获取模块。为此，bot会发送一个包含 get_module标签的beacon。在响应中，服务器会发送请求的模块。

![](http://static.wooyun.org//drops/20160420/2016042010014762447.com/blob/foeaaawobgd/gjmhhjddncnou-fm5d3ucw?s=wyoyaawai16p)

![](http://static.wooyun.org//drops/20160420/2016042010014953548.com/blob/foeaaawobgd/idmvmuqgldtnkt4zwnocrg?s=wyoyaawai16p)

**Beacon with data**

每当bot有数据要发送到C2时，bot就会发送一个包含数据标签的beacon，其中包括了要发送的数据。要发送的数据取决于数据的收集方式，但是，可以包括：

*   键盘记录数据
*   浏览器中间人攻击数据（格式，密码等），截图
*   调试数据
*   其他数据

![](http://static.wooyun.org//drops/20160420/2016042010015132244.com/blob/foeaaawobgd/t_gi49x9_sqiigvfcslela?s=wyoyaawai16p)

**节点beacon**

节点会周期性地向admin_node发送节点beacon（每20分钟一次），从而保证节点状态是活动的。 下面是一个节点beacon示例：

![](http://static.wooyun.org//drops/20160420/2016042010015484963.com/blob/foeaaawobgd/fvlnu8oww4jygo6l2qlobg?s=wyoyaawai16p)

0x04 在Dridex Botnet中运行一个虚假节点
=====

在执行逆向过程的过程中，AnubisNetwork的主要目标是研究一种方法来实时地监控botnet，及其感染情况，配置文件，模块和数据窃取活动。

为了进一步了解Dridex及其操作，我们决定使用我们已知的协议知识用Python开发一个虚假的bot，这个bot要具备下面的功能：

*   **自我注册到botnet**- 这是必须的第一个步骤。虚假bot需要与dridex协议 “交流”，并且和其他的bot要保持无差别；
*   **提升至节点**- 在提升至节点后，我们就可以连接到其他的bot；
*   **拦截通讯**-如果能解密连接，我们就能知道能从bot中提取出哪些确切的信息类型，以及bot主控的可能行为模式；
*   **接收更新后的配置文件**- 这是一个很有趣的功能，因为这样能允许我们实时监控新的目标（比如，银行机构），无论它们是在什么时候添加或移除的；
*   **接收更新后的软件模块**- 这样能允许我们监控现有模块的更新，以及botnet中新出现的模块；
*   **在节点之间发送任意消息并伪造C2回复**- 这样能允许我们进一步测试botnet并理解其操作行为。

架构
--

下图中表示的是虚假节点的架构：

![](http://static.wooyun.org//drops/20160420/2016042010015678759.com/blob/foeaaawobgd/hs_arwjuvrfofgd_tnl78q?s=wyoyaawai16p)

系统包含下面的组件：

*   Bot模块 - 这个模块负责把bot注册到P2P网络。这个模块会和常规bot一样与节点通讯，在提升过程后，会作为节点与admin_node通讯；
*   节点模块 - 这个模块负责接收所有的输入连接并处理所有的节点间通讯。如果输入连接没有使用节点间协议，模块就会将其转发给SSL MITM模块；
*   SSL MITM模块 - 这个模块负责拦截SSL通讯，通过向客户端呈现一个虚假的证书，并重建到admin服务器的SSL连接。这个模块还负责使用C2私钥来解密所有的C2协议通讯。

部署
--

我们的虚假节点实现了所有的目标。运营这个节点是一个挑战，因为bot操作者的高机动性，以及他们随时部署的对抗手段，请看下面的反制措施部分。

我们的虚假节点成功的监控了botnet几个月的时间，但是这种方法的维护成本很高，因为botnet也是在不断进化的。不过，通过这个软件，我们更清楚了botnet的内部工作原理，也发现了其中的几处漏洞。

为了追踪不同的Dridex botnet，不同时间的C2变化和bot感染，我们开发了另一个应用，叫做Dridex Tracker。下图中就是Dridex Tracker 的GUI：

![](http://static.wooyun.org//drops/20160420/2016042010015899429.com/blob/foeaaawobgd/rhxsd-iqjwdqp6dszswh6w?s=wyoyaawai16p)

Dridex Tracker截图

反制措施时间线
-------

在2015年4月到7月的上半月，这个虚假的节点能够启用所有的功能。在这段时间内，botnet和bot模块也更新了几次，bot主控新增了几种反制措施来防止虚假节点进入网络。 下面是一些值得注意的事件记录：

*   4月22日 - 虚假节点提升至节点并开始接收其他bot的连接；
*   4月30日 - Dridex协议发生变化，我们无法再加入botnet。于是我们逆向了新协议并改进了虚假节点的代码；
*   5月20日 - bot主控开始通过在节点列表中移除虚假节点，并禁用虚假节点的IP地址注册到网络。我们选择在同一时间内只在一个botnet中使用虚假节点；
*   5月27日 - bot主控新增了一项功能，在拦截虚假节点IP时，发送127.0.0.1作为admin_node和节点的IP地址，这样导致虚假节点会进入一个无限循环；
*   5月30日 - bot主控开始尝试通过发送 “killer模块”来清除系统MBR并造成重启，从而禁用虚假节点；
*   6月3日 - bot主控开始更频繁的拦截虚假节点，一天就好几次；
*   7月10日- 节点提升过程似乎可以手动控制了，或是通过其他的方法来阻止自动地快速节点提升。

0x05 结果
=====

感染分布
----

在考虑到受感染系统的规模上，相较于其他botnet，Dridex botnet的规模并不大，但是仍然是一项严重的威胁。这一点很不正常，因为Dridex的邮件传播活动很频繁。

在2015年4月，我们在Dridex主botnet上找到了下面的这些bot：

![](http://static.wooyun.org//drops/20160420/2016042007231235301.com/blob/foeaaawobgd/lxfgkkgwfzcgy7m-avargg?s=wyoyaawai16p)

在7月，我们观察到数量开始下降。下面的与bot相关的统计使我们的虚假节点在2015年6月9日-7月10日之间从Dridex主botnet上拦截的。

**Dridex-220**

![](http://static.wooyun.org//drops/20160420/2016042010020175700.com/blob/foeaaawobgd/2sa1dxyn_wap0mkmfqeofq?s=wyoyaawai16p)

Dridex 220的地理位置分布和前十大受感染国家（不同bot：3770）

**Dridex-120**

![](http://static.wooyun.org//drops/20160420/2016042010020310102.com/blob/foeaaawobgd/bfoqgv6orovc-icygkv6bg?s=wyoyaawai16p)

Dridex 120的地理位置分布和前十大受感染国家（不同bot：995）

目标
--

Dridex的主要目标是几个国家的银行。在我们分析期间，我们在主botnet（120，200和220）更新时，实时收集到了几个不同版本的配置文件。下面是这些配置文件的哈希。从文件数量上就能反映出更新数量，从中也能看出botnet操作者的活动很频繁。

![](http://static.wooyun.org//drops/20160420/2016042010020591440.com/blob/foeaaawobgd/h2kkggolzzpw85vtd_4sja?s=wyoyaawai16p)

![](http://static.wooyun.org//drops/20160420/2016042010020894695.com/blob/foeaaawobgd/ijmllhxkmsumxtaocrslnw?s=wyoyaawai16p)

![](http://static.wooyun.org//drops/20160420/2016042010021112306.com/blob/foeaaawobgd/s5ywubt4u6cfvd7x6ndg8w?s=wyoyaawai16p)

这些配置文件中包含有上百个URL的webinject设置，大多数都属于银行应用和其他与金融相关的应用。除了webinject数据，这些bot还会记录下列应用的数据：

![](http://static.wooyun.org//drops/20160421/2016042102514299065.com/blob/foeaaawobgd/y6eg4b6nfypls86j7squmq?s=wyoyaawai16p)

很有趣的一点是，Didex不仅仅从银行网站上收集信息，还会从其他的一些网站上收集信息，而这些网站并没有直接包括在配置文件的webinject设置中。其中包括企业外联网的登录凭据，VPN，webmails和几项在线服务。在botnet上运行虚假节点时，我们总共观察到了2069个不同的域名，这些域名都是从用户的浏览器中收集并传输到了C2。

窃取到的信息
------

我们的虚假节点能解密bot与C2服务器之间的通讯。借此，我们更好地理解了bot采集到的信息类型，虽然，其中有失窃的银行凭据，但是大多都与网上银行没有直接关联。我们主要观察到了3类提交数据：

### 键盘记录数据

这些数据来自一些被配置记录的应用。另外，因为其中包括一些通用软件，比如jp2launcher.exe（java applets的进程），几个聊天应用和其他基于applet的接口也都被记录了。下图中就是一个传输的键盘记录会话：

![](http://static.wooyun.org//drops/20160420/2016042010021593415.com/blob/foeaaawobgd/kg1jfo4e2fdky1ioe-5jdw?s=wyoyaawai16p)

发送给Dridex C2的一个聊天会话转储

### Web Inject数据

Web injects能够从大量不同的域名中提取特定HTTP所提交数据 。下面是一些发送到C2的在线应用凭据：

![](http://static.wooyun.org//drops/20160420/2016042010021971826.com/blob/foeaaawobgd/rkbbdkyh_djbfpqoqig1gq?s=wyoyaawai16p)

发送给Dridex C2的登录凭据

### 屏幕截图

一些应用的屏幕截图也可以被获取。下面就是一个传输到C2的屏幕截图：

![](http://static.wooyun.org//drops/20160420/2016042012575441623.com/blob/foeaaawobgd/dznqabtnc4e7s0encwe9rg?s=wyoyaawai16p)

感染了Dridex的系统向C2发送的截图

0x06 入侵标识
=====

已分析样本
-----

下面是我们在研究过程中分析过的样本，以SHA-256 哈希作为参考：