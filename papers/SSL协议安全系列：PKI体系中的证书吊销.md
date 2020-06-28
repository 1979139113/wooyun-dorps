# SSL协议安全系列：PKI体系中的证书吊销

0x00 前言
=====

在前面的章节我们讨论了部分SSL/TLS握手协议、记录协议中存在的安全问题，针对它们的攻击以及相应的加固方案。在SSL/TLS所依赖的PKI体系中，证书吊销过程同样对SSL通信的安全性有着重要的影响，近几年很多重要安全事件（比如2008年Debian伪随机数发生器问题导致证书公钥易被破解、2014年Heartbleed漏洞导致服务器私钥泄露）都导致了大规模的证书吊销行为。本章我们将介绍传播证书吊销信息的几种方式、它们的优缺点和现状以及针对证书吊销标准的若干攻击1。

0x01 概述
=====

在公钥基础设施体系中，CA负责签发证书，同时，为了维护PKI的完整性，CA也负责吊销证书。当服务器的证书不再合法时（比如服务器提前更换证书、证书的公钥被破解、服务器端存储的证书的私钥泄露等），负责签发这一证书的CA必须要吊销该证书，并通过相关途径将该证书失效信息告知SSL客户端，帮助客户端进行证书合法性校验。CA需要维护它所签发的所有证书的吊销状态。

在客户端建立SSL链接的过程中，服务器会向客户端提供一个证书链作为SSL握手消息的一部分，客户端在校验证书链的过程中，除了检验签名等信息的合法性外，还需要保证证书链中的所有证书均违背吊销，否则链接应该被断开。因此，在吊销证书时，如果被吊销的证书为叶子证书，那么该证书理论上不能通过证书校验；如果被吊销的证书为CA（中间CA或根CA）证书，那么该CA签发的所有叶子证书都不能通过证书校验。通常来说，每个证书都会包含如何检查它是否被吊销的信息，客户端可以根据这个信息来检查该证书是否已被吊销。

目前有两种主要的方法供客户端查询证书的吊销情况：CRLs证书吊销列表（Certificate Revocation Lists）和OCSP在线证书状态协议（Online Certificate Status Protocol）。

0x02 详细介绍
=====

### 1.CRLs

CRLs是应用最广泛的证书吊销检查方案。CRL通常是一个ANS.1编码的文件，包含了一系列元组，每个元组对应一个证书的吊销信息，包含吊销证书的序列号、吊销时间戳、吊销原因等。CRL由CA维护，每个CA通常会维护一个或多个CRL列表，发布它所签发证书的吊销状态。同时，CRL和X.509证书类似，是有一定有效期的，因此CA会阶段性重新签发CRL，即使没有加入新的证书吊销信息，95%的CA更新CRL的时间间隔在24小时之内。客户端可以缓存CRL，但是在它们过期之后还要重新下载更新了的CRL，这也是使用CRL的一个不方便之处。

几乎所有证书都在CRL Distribution Points证书扩展里放置了一个URL告诉客户端能够查询它吊销信息的CRL所在的位置。当一个客户端使用CRL的方式查询证书的吊销状态时，它需要到证书扩展指定的位置下载CRL文件，然后检查当前证书的序列号是否在CRL的列表当中。2014年美国东北大学的一组研究人员对整个IPv4地址空间扫描得到的证书分析发现，仅4,384（0.09%）的证书不包含任何证书吊销检查信息，这部分证书无法被吊销，它们通常是一些自签名证书。图1是百度的SSL证书，它包含了CRL分发点信息；图2是12306网站的证书，它是一个自签名证书，没有包含CRL信息。

![p1](http://drops.javaweb.org/uploads/images/0c388b332106d1270bf4f31b1b0c108247edaecf.jpg)图1 百度的SSL证书CRL信息

![p2](http://drops.javaweb.org/uploads/images/c2f27c03c37e7a33cc9fed6889fe2279b191bf6f.jpg)图2 12306的SSL证书

### 2.OCSP

OCSP的应用时间比CRL稍晚，现在也已被大多数CA采用，如图3，在2012年7月RapidSSL开始支持OCSP后，证书对OCSP技术的支持率也已经达到95%以上。OCSP最初是被设计用来解决CRLs开销过大的问题，它允许客户端生成一个HTTP请求来获取证书的吊销状态，OCSP服务器也被称作OCSP应答者，由CA维护，收到请求后它们会返回一个CA签名了的响应，包含证书的吊销状态（good, revoked或unknown），以及响应的有效时间，通常OCSP响应的有效时间比CRL的要长，可以缓存在客户端。OCSP查询所需的URL被放置在Authority Information Access证书扩展里。

OCSP提供了一种实时查询证书吊销状态的方法，解决了很多CRL效率低下的问题，但是它依然需要客户端向CA发出请求来确定证书是否可以被信任，向OCSP服务器请求证书吊销状态的行为会将用户的浏览行为信息泄露给CA，一定程度上泄露了用户隐私；同时，由于OCSP是实时查询，OCSP服务器的响应速度会影响客户端页面的加载速度，存在一些性能问题。总之，OCSP弥补了CRLs的一些不足，但同时也引入了一些新的问题，研究人员又提出了OCSP stapling技术来解决这些问题。

![p3](http://drops.javaweb.org/uploads/images/3932e32a0c00bb8444470b9674f00c4a0861df1c.jpg)图3 证书对CRLs和OCSP技术的支持情况

### 3.OCSP Stapling

OCSP Stapling是一个SSL/TLS扩展，X.509证书里用status_request这个扩展来表明客户端支持OCSP stapling。OCSP Stapling的工作原理是这样的，SSL服务器将OCSP响应缓存在服务器中，然后把它们作为SSL握手的一部分发送给客户端。因此，客户端在和一个支持OCSP Stapling技术的服务器通信时，它可以同时接收到服务器证书和该证书的吊销状态，这种机制一定程度上解决了OCSP带来的隐私问题和性能问题。

当然OCSP Stapling也并不是完美的，它仅允许SSL服务器缓存叶子证书的OCSP响应，而不允许缓存中间证书的OCSP响应，2013年一个叫做TLS Multiple Certificate Status Request Extension的TLS扩展被提出用来解决这个问题，但是还没有被广泛应用。同时，由于OCSP存在有效期，SSL服务器也未必一直缓存着OCSP响应，可能客户端在和SSL服务器通信时服务器没有在握手消息中插入证书吊销状态信息，需要客户端多访问几次才可能返回OCSP响应信息。图4显示了对于所有支持OCSP Stapling技术的服务器，客户端发起一次请求时，仅82%的服务器在握手消息中插入证书吊销信息，而发起10次请求时几乎所有的服务器都会在握手消息中插入证书吊销消息。

![p4](http://drops.javaweb.org/uploads/images/2237fd5e38b796761cb2812365891a47b5bf286b.jpg)图4 客户端请求次数与SSL服务器OCSP Stapling响应对应关系

0x03 吊销检查方案中存在的问题
=====

### 1.客户端支持问题

证书吊销存在的最大问题就是客户端对证书吊销检查方案的支持不充分。通常来说主流浏览器对SSL/TLS协议各项技术的支持比普通应用程序要好，考虑到浏览器可能在SSL协议建立的不同阶段会使用不同的SSL库（比如Chrome在SSL链接建立时使用Mozilla的NSS库，而在证书校验时依赖操作系统相关的库），美国东北大学的研究人员做了一组实验，将不同的操作系统与不同的浏览器进行组合，操作系统包括Ubuntu 14.04、Windows 8.1、OS X 10.10.2及iOS 6-8、Android4.1-5.1等，测试的浏览器包括Chrome、Firefox、Opera、Safari、IE等覆盖市场所有主流浏览器。它们的测试发现不同浏览器在利用CRL和OCSP查询证书吊销状态时，对于叶子证书和中间证书、普通证书和EV证书的处理情况都不相同，对于不同的OCSP服务器响应各浏览器表现也不同，每种浏览器都存在或多或少的安全问题，桌面平台浏览器在证书吊销处理这方面的表现最好的是IE，其次是Safari以及较新版本的Opera浏览器，Chrome和Firefox存在的问题较多。另外，移动平台上的所有浏览器都没有进行证书吊销状态检查，所以对于那些在Heartbleed漏洞中泄露了私钥的证书，即使它们已经被吊销，攻击者还可以利用它们来攻击移动平台而不会被发现。下表为对不同浏览器证书吊销状态检查行为的测试结果。

![p5](http://drops.javaweb.org/uploads/images/71344dadc580e8325f1f1928c2ebe0f467c4625d.jpg)

### 2.CRL性能问题

CRLs技术一直被诟病的是它的效率问题，客户端需要将完整的CRL文件下载下来进行证书吊销状态检查。随着越来越多安全事件的发生，大量证书被吊销，导致CRL文件的大小增长迅速，比如著名CA GoDaddy的吊销信息在2007年才158KB，到2013年已经增长到41 MB，随着时间的推进，过大的CRL文件将越来越不适合用来进行证书吊销状态检查。

部分CA通过维护多个CRL的方式来解决CRL文件过大的问题，每个CRL只包含所有签发证书的子集，但即便如此，对于那些最流行的CA每个CRL的体积依然很大。下表描述了主要CA的CRL体积情况，可以看到即便GoDaddy使用了322个不同的CRL，平均每个CRL的体积依然超过1MB。

![p6](http://drops.javaweb.org/uploads/images/913624c451df96076e853450c15a7f56c8fca39e.jpg)

除了维护多个CRL这种方式外，有的CA还使用增量CRL的方式来解决CRL文件过大的问题，这种技术通过让客户端下载最新增量 CRL，并将该 CRL 与之前的 CRL 组合在一起，以拥有已吊销证书的完整列表。由于客户端通常在本地缓存 CRL，因此，使用增量 CRL 可潜在改进性能。要使用增量 CRL，客户端应用程序必须了解并明确使用增量 CRL 来进行吊销检查。如果客户端不使用增量 CRL，它会在每次刷新其缓存时从 CA 检索 CRL，不管增量 CRL 是否存在。为此，应验证预期应用程序是否使用增量 CRL，并进行相应的配置。如果客户端不支持使用增量 CRL，则不应将 CRL 配置为发布增量 CRL，或者不应将其配置为于同一间隔发布 CRL 和增量 CRL。这仍允许未来的支持增量 CRL 的应用程序使用它们，同时可为所有应用程序提供当前 CRL。目前，使用 Windows XP 和 Windows Server 2003 家族产品中的 CryptoAPI 的所有应用程序都使用增量 CRL。由于这种技术和现行的CRL技术有一定差别，和CA维护并定时更新CRL的机制并不完全兼容，出微软的产品外别的应用程序使用这种技术的还不多见。

0x04 参考资料
=========

1.  Liu Y, Tome W, Zhang L, et al. An End-to-End Measurement of Certificate Revocation in the Web's PKI[C]//Proceedings of the 2015 ACM Conference on Internet Measurement Conference. ACM, 2015: 183-196.
2.  Zhang L, Choffnes D, Levin D, et al. Analysis of SSL certificate reissues and revocations in the wake of Heartbleed[C]//Proceedings of the 2014 Conference on Internet Measurement Conference. ACM, 2014: 489-502.
3.  [https://msdn.microsoft.com/zh-cn/library/cc782162(v=ws.10).aspx](https://msdn.microsoft.com/zh-cn/library/cc782162(v=ws.10).aspx)
