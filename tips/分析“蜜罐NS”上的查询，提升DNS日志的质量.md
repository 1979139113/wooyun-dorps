# 分析“蜜罐NS”上的查询，提升DNS日志的质量

from:http://securityintelligence.com/analyzing-queries-on-a-honeypot-name-server-for-better-dns-log-quality/

0x00 网络杂讯
=====

“蜜罐”是统计“网络杂讯”的一种常用方法，并且这种方法也比较简单。你对“网络杂讯”了解的程度越高，那么你就能[更好地做安全分析](http://securityintelligence.com/five-steps-for-better-security-analytics-in-2015/)。 我好奇的是，在公有云上，NS蜜罐能接收哪些流量；我的研究如下：

0x01 设置NS蜜罐
=====

这个服务器的系统是默认设置下的Ubuntu 14.04.1 LTS，由法兰克福亚马逊弹性计算云（Frankfurt EC Amazon cloud）托管，并且服务器配置了一个IPv4地址（我没有查看IPv6数据）。因为公共渠道上并没有公布服务器的IP地址和DNS（域名服务器）服务，所以，任何进入这个服务器的请求都可被视作可疑请求。目前，我还没有调查云提供商[重复使用IP地址](http://arxiv.org/pdf/1204.0764.pdf)对此造成的影响。

这个服务器还安装了下面这些“蜜罐”：[dionaea](http://dionaea.carnivore.it/)、[Glastopf](http://glastopf.org/)、[Conpot](http://conpot.org/)、SNMP、NTP和[Kippo](https://github.com/desaster/kippo)。 Kippo是一个SSH蜜罐，用于吸引黑客的注意并提升安全性。在[Github](https://github.com/cudeso/cudeso-honeypot/tree/master/elk)网站上的一个存储库中，可以获取Kippo的相关设置和数据收集（通过ELK）。

我曾经使用过最热门的一个[NS软件](http://en.wikipedia.org/wiki/Comparison_of_DNS_server_software)-[BIND](https://www.isc.org/downloads/bind/)。 其中多数设置都是默认设置。我禁用了递归，IPv6和转发器（forwarder），启用了日志，还自定义了服务器的版本号，在该服务器中含有一个zone file。Zone file包含了IP地址和域名的映射关系，以及可用的子域。Zone file中包含一条记录，这条记录指向着同一个主机。所以，任何人只要查询这个NS蜜罐（“a.b.c.d”），就会得到一条包含蜜罐服务器IP地址的应答。

0x02 Bind配置
=====

```
options { directory "/var/cache/bind"; dnssec-validation auto; recursion no; allow-transfer { none; }; auth-nxdomain no; # conform to RFC1035 // listen-on-v6 { any; }; statistics-file "/var/log/named/named_stats.txt"; memstatistics-file "/var/log/named/named_mem_stats.txt"; version "9.9.1-P2";}; logging{ channel query_log { file "/var/log/named/query.log"; severity info; print-time yes; print-severity yes; print-category yes; }; category queries { query_log; };};

```

  

```
$TTL 10<br/>@ IN SOA localhost. root.localhost. (<br/> 1 ; Serial<br/> 10 ; Refresh<br/> 10 ; Retry<br/> 10 ; Expire<br/> 10 ) ; Negative Cache TTL<br/>;<br/><br/> IN NS localhost<br/>* IN A a.b.c.d

```

0x03 统计数据
=====

第一组统计数字表示的是从日志文件中发现的原始数据。

0x04 时间范围
=====

如果我们根据这些数据映射查询，我们就会发现，在1月15日和1月20日、1月末和2月初，查询量猛增。

![enter image description here](http://drops.javaweb.org/uploads/images/c31cfc493add16bc61c76584fb856283d58394b6.jpg)

通过进一步的调查，我们发现这些查询都是由一个IP地址引起的；这个IP地址属于德国波鸿鲁尔大学。在波鸿鲁尔大学的校网站上解释说明了一个“[DDos放大攻击追踪者项目](http://scanresearch.syssec.ruhr-uni-bochum.de/)（Amplification DDoS Tracker Project）”。这个网站通过获取扫描数据，提醒网络所有者可能出现的问题。我们发现有两个IP地址进行了这些扫描：其中一个属于德国波鸿鲁尔大学，另一个属于德国萨尔大学。

我们之后还发现，是由于open resolver项目查询了蜜罐DNS，所以才造成了大量查询。

0x05 这些查询来自何方？
=====

我提取出了日志中的IP地址，然后使用[Team Cymru](http://asn.cymru.com/)公司的ASN，获取了这些IP的对应国家。自治系统（AS）可以告诉我们IP地址所属的网络区段。在[Github](https://github.com/cudeso/cudeso-honeypot/tree/master/elk)网站上也可以找到这个脚本。

![enter image description here](http://drops.javaweb.org/uploads/images/187b81157d135a5494279ea019a338ca8c3d6bac.jpg)

![enter image description here](http://drops.javaweb.org/uploads/images/ddfcfebf6463388b0112115f370dbc997ac59e69.jpg)

扫描主要来自德国，美国和中国这三个国家。其中，DFN（德国）和湖南Chinanet这两个AS的查询较多。有超过半数的查询来自欧洲（RIPE）。考虑到这中间有大量的请求来自德国鲁尔大学，出现这样的结果也就不足为奇了。

![enter image description here](http://drops.javaweb.org/uploads/images/1d50384e2fd37b8265d6b05a6f467eb3c6761191.jpg)

0x06 他们在找什么？
=====

多数查询都是为了获得域名的A记录，18%以上的查询是为了获得域名的ANY记录。TXT请求主要是为了获取DNS服务器的版本。

![enter image description here](http://drops.javaweb.org/uploads/images/fe73d09eeb1c300671d8ebffaaa3ba0e3619d74c.jpg)

这些查询是为了获取Google和Shadowsever的IP地址，或者是Bind NS的版本。不出意外，最普遍的[TLD](http://en.wikipedia.org/wiki/Top-level_domain)是.com和.org。在你创办企业时，[要谨慎选择域名](http://securityintelligence.com/malicious-domains-can-give-you-the-biz/)和TLD。这份数据中，最有趣的部分是.ru和.cn这两个TLD所占的比例。

![enter image description here](http://drops.javaweb.org/uploads/images/9eacab2d9745d7f5e0980f3608670d4ecc27f88a.jpg)

![enter image description here](http://drops.javaweb.org/uploads/images/c51fd81bbc44731975d4a1ef8f6f5e307a8277fc.jpg)

0x07 open resolver扫描
=====

如上文所述，open resolver项目导致了大量的请求。事实上，大约56%的查询都是来自一些参与open resolver的组织。

![enter image description here](http://drops.javaweb.org/uploads/images/96666d38f1d76a204eca354af89e5793accba6b3.jpg)

虽然，这类查询的数量很大，但是在日志中可以很容易地辨别。

```
01-Feb-2015 04:57:49.352 queries: info: client x.x.x.x#34341 (dnsscan.shadowserver.org): query: dnsscan.shadowserver.org IN A + (x.x.x.x)02-Feb-2015 19:15:44.507 queries: info: client x.x.x.x#41248 (www.goOGLe.co.in): query: www.goOGLe.co.in IN A + (x.x.x.x)07-Jan-2015 06:36:04.149 queries: info: client x.x.x.x#33481 (7f14f6df.openresolvertest.net): query: 7f14f6df.openresolvertest.net IN A + (x.x.x.x)11-Jan-2015 14:54:03.692 queries: info: client x.x.x.x#43656 (openresolver.com): query: openresolver.com IN A +E (x.x.x.x)01-Feb-2015 06:42:54.797 queries: info: client x.x.x.x#46018 (7f14f6df.openresolverproject.org): query: 7f14f6df.openresolverproject.org IN A + (x.x.x.x)08-Feb-2015 04:12:45.562 queries: info: client x.x.x.x#28207 (9h2y.96bf5d36.wc.syssec-research.mmci.uni-saarland.de): query: 9h2y.96bf5d36.wc.syssec-research.mmci.uni-saarland.de IN A + (x.x.x.x)<br/>

```

这类查询很令人反感，数量也很庞大。如果你不在日志监控系统中过滤掉这些，就很难发现真正的恶意请求。如果你想时刻关注DNS服务器，你必须要做的就是应用合适的过滤程序，在处理日志前，首先去除网络杂讯，拦截这类请求，或者是要求他们停止扫描你。这样还能增多你从日志监控或[SIEM](http://securityintelligence.com/gartner-2014-magic-quadrant-siem-security/)解决方案中获得的结果。

这类扫描还可能涉及法律问题。虽然，多数open resolver项目的请求通常都不是恶意的，但是也不难想象，有些不熟悉这些组织的人会认为这些扫描是恶意的。安德鲁·柯麦科（Andrew Cormack）在他的论文《[扫描漏洞：这是合法的吗？](https://www.terena.org/activities/tf-csirt/meeting43/Scanning%20for%20Vulnerabilities.pdf)》 中解释了某些法律问题。

0x08 过滤open resolver扫描后的结果
=====

在过滤了open resolver查询后，得到了下列结果：

![enter image description here](http://drops.javaweb.org/uploads/images/4985ff7d171cb4a72b092c7c3bc61f9c87a0d578.jpg)

在这些时间范围中，并没有显著地突增。多数查询都是来自美国和中国。RISs Ripe，Arin和Apnic之间的查询分布也都大致相当。

![enter image description here](http://drops.javaweb.org/uploads/images/253c9b55e577a9646ff9c6b0f0b6a6381976c2b3.jpg)

在这一类别中，请求类型不是A记录查询就是ANY资源查询。多数都是谷歌域名查询。

![enter image description here](http://drops.javaweb.org/uploads/images/bc086883eb6fcf8fb7447a296318bd52cb833aff.jpg)

![enter image description here](http://drops.javaweb.org/uploads/images/68838bd8315db5fd5e423c0c228b3600037988be.jpg)

在剔除了open resolver的干扰后，数据集揭露了两个特别主机的行为：一个来自中国(AS 63835)，另一个来自俄罗斯(AS 2848)。

中国主机只定期查询Bind NS的版本，然后查找www.google.it和www.google.com的A记录：

```
05-Feb-2015 18:35:21.888 queries: info: client x.x.x.x#56334 (VERSION.BIND): query: VERSION.BIND CH TXT + (x.x.x.x)06-Feb-2015 01:19:13.674 queries: info: client x.x.x.x#39664 (www.google.it): query: www.google.it IN A + (x.x.x.x)06-Feb-2015 16:49:14.384 queries: info: client x.x.x.x#51102 (www.google.com): query: www.google.com IN A + (x.x.x.x)07-Feb-2015 01:57:22.995 queries: info: client x.x.x.x#45938 (VERSION.BIND): query: VERSION.BIND CH TXT + (x.x.x.x)07-Feb-2015 14:35:58.562 queries: info: client x.x.x.x#41664 (www.google.it): query: www.google.it IN A + (x.x.x.x)07-Feb-2015 23:00:43.537 queries: info: client x.x.x.x#49252 (www.google.com): query: www.google.com IN A + (x.x.x.x)08-Feb-2015 13:27:10.678 queries: info: client x.x.x.x#34047 (VERSION.BIND): query: VERSION.BIND CH TXT + (x.x.x.x)<br/>

```

俄罗斯主机只定期查询com：

```
06-Feb-2015 08:45:17.256 queries: info: client x.x.x.x#42795 (com): query: com IN ANY +E (x.x.x.x)08-Feb-2015 15:44:01.787 queries: info: client x.x.x.x#33207 (com): query: com IN ANY +E (x.x.x.x)<br/>

```

0x09 'ANY'请求出了什么问题？
=====

大约半数的请求是“ANY”请求、

```
10-Feb-2015 07:48:38.565 queries: info: client x.x.x.x#32767 (isc.org): query: isc.org IN ANY +ED (x.x.x.x)

```

通常，这是使用虚假查询 的DNS放大攻击。 所有这些查询都具有递归()设置标志，说明这个查询来自客户端，或者是服务器转发的请求。只有极小数量的主机要求()在应答中支持DNSSEC。

除了Google和ISC域名，我们还发现了更多“外来的”域名；攻击者利用这些域名，测试了DNS放大攻击的可能性。在此前的攻击中发现了这些域名：

*   globe.gov
*   ohhr.ru
*   Egransy.com
*   uzuzuu.ru
*   sema.cz
*   vlch.net

[DNS放大攻击观察者](http://dnsamplificationattacks.blogspot.be/)掌握了攻击中使用的域名，并且还能提供一个[IP表单规则集](https://github.com/smurfmonitor/dns-iptables-rules/blob/master/domain-blacklist-string.txt)，使用“黑名单”拦截这些域名。

但是，为什么还会有人使用这些请求呢？用正常的网络行为无法解释发送这些请求的原因。然而，如果你看一看某些请求返回的应答，情况就会一目了然。你可以使用下列命令测试：

```
dig -t ANY @8.8.8.8 mydomain

```

某些应答的大小超过了6,000字节。

```
;; MSG SIZE rcvd: 6800;; MSG SIZE rcvd: 6584

```

对于常规用途而言（区域传送除外），DNS使用端口53上的UDP协议传输数据包。这种方法要求DNS数据包的大小相对较小（512字节，不考虑不同的表头）。需要注意的是，DNSSEC通常要求体积更大的数据包。DNS使用[EDNS0](http://en.wikipedia.org/wiki/Extension_mechanisms_for_DNS)，就可以处理体积更大的数据包。然而，使用EDNS0还是对数据包的大小有限制（可能是服务器不支持EDNS0），所以，又重新使用TCP协议处理这些大数据包的DNS请求。输出显示：

```
;; Truncated, retrying in TCP mode.

```

总之，就是一个通过简单协议传输，占用资源较少的小型请求，变成了一个通过复杂协议传输，占用大量资源的大型应答。

UDP协议和TCP协议的区别在于，UDP没有“握手过程”。你发送了请求，也就结束了；这是一个“无状态”协议。而TCP协议则具有“握手过程”，它需要更多的资源。此外，这种协议下，还可以很轻松的假冒IP地址的来源。

综合上述两点，放大攻击就有了很好的攻击向量。

[OpenDNS](https://labs.opendns.com/2014/03/17/dns-amplification-attacks/)的同仁们描述了这些DNS攻击是如何运作的。[CloudFlare](https://blog.cloudflare.com/deep-inside-a-dns-amplification-ddos-attack/)公司也观察到了相同的模式。

为了防御这种类型的攻击，需要需要在多个层面上努力。DNS管理员需要通过限制列表上可以执行递归查询的客户端，确保他们的递归NS不是开放的解析器。对于授权的NS，管理员应该设置[应答频率限制](https://deepthought.isc.org/article/AA-00994/0/Using-the-Response-Rate-Limiting-Feature-in-BIND-9.10.html)（RRL）。 网络管理员则可以只允许已知的网络前缀离开他们的网络（执行[BCP 38](http://tools.ietf.org/html/bcp38)。)这样可以防御所有基于UDP协议的DDoS放大攻击（DNS、SNMP、NTP等）。

0x10 结论
=====

如果一个随机的DNS服务器快速接受了大量针对开放解析器测试的请求，那么这些网络杂讯就会污染日志；这样你就很难检测到DNS服务器上的攻击。

你可以使用DNS服务器“蜜罐”，捕捉这些网络杂讯。

仅仅使用几个脚本，你就可以根据“蜜罐”数据集提供的信息，轻松地过滤掉主要的扫描者。然后，你就可以使用这个白名单，把他们从真正的DNS日志中去除，使你的日志监控或SIEM解决方案更具价值。但是，如果你想要阻止某些高级黑客入侵你的白名单，手动设置白名单认证还是很有必要的。