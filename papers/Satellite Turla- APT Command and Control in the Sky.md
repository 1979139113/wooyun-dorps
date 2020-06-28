# Satellite Turla: APT Command and Control in the Sky

你有看过卫星电视吗？对里面丰富的电视频道和广播电台是否感到非常惊讶？你有没有想过了解卫星电话或卫星网络（satellite-based Internet connections）是如何运作的？如果我告诉你，除了娱乐，交通和天气外卫星Internet还有更多的使用方法。更多，更多的...

[Turla: Hiding Traces High in the Skies](https://youtu.be/Du3rBVZqKkk)

如果你是APT团队的成员，那么你每天都需要处理各种不同的问题。其中一个，也可能是最大的任务就是不断地入侵服务器作为自己的C&C（command-and-control）。这些已经被入侵的服务器通常被相关执法部门移除或被IPS关闭掉。有时候会被利用来跟踪攻击者的物理位置。

一些高级的攻击者和商业黑客工具用户找到了一个更好的解决方案--使用基于卫星Internet链路（satellite-based Internet links）。在过去，我们已经看到三个不同的攻击者使用这些链路隐藏他们的业务。最特殊有趣的是Turla团队。

Turla也称之为Snake或者Uroburos，其名称来自于最高级的rootkit。Turla网络间谍团队已经活跃了8年之久。虽然已经有几篇发表的论文阐述关于该团队运作，但直到卡巴斯基实验发表[Epic Turla](https://securelist.com/analysis/publications/65545/the-epic-turla-operation/)的研究报告才提供了它们一些比较特殊资料，如第一阶段的watering hole感染攻击。

Turla团队之所以特殊不仅是他的工具复杂，包括Uroboros rootkit（又称为Snake），以及通过内部LAN的多级代理网络绕过网闸的机制，但这次攻击的后期阶段使用的是基于卫星的C&C机制。

在这篇博客中，我们希望清楚地阐明基于卫星C&C机制的APT组织，包括Turla/Snake组织，被他们所控制的重要受害者。当这些机制变得越来越流行的时候，系统管理员部署正确的防御策略来减轻这种攻击尤为重要。具体详情请查看附录。

![](http://drops.javaweb.org/uploads/images/ad8def8ede1a065a0afec97f6f93440209b64c0c.jpg)

0x01 技术细节
=====

虽然比较罕见，但自2007年以来，几个顶尖的APT团队一直在使用卫星链路来管理他们的业务。大多数情况下使用的是C&C基础设施，Turla就是其中之一。使用这个方法有一些优点，比如很难分辩出发起攻击的背后运营商，但它同时也给攻击者带来一些风险。

一方面，C&C服务器的真实位置不容易被发现。基于卫星Internet的接收器可以位于任何卫星覆盖区，而这个区域通常是非常大的。Turla团队使用这个方法劫持下游链路不仅高度匿名，而且还不需要交卫星网络费用。

另一方面，卫星Internet有速度慢，不稳定等缺点。

开始的时候，我们和其它研究人员都不清楚观察到的那些链路是否都是攻击者购买的卫星商业网络，或者攻击者违反了ISP使用路由层MitM（Man in the Middle）攻击劫持流量。现在我们已经分析了这些机制，并得出了惊人的结论，Turla团队使用的方法非常简单明了，而且高度匿名，操作简单，管理方便。

0x02 购买卫星网络链路，还是使用MitM攻击或BGP劫持?
=====

购买卫星Internet链路是APT团队确保C&C业务运转的选择之一。然而，全双工的卫星链路是非常昂贵的：一个简单的双工为1Mbit up/down卫星链路的费用可能高达每星期7000美刀。对于较长期的合同，这笔费用可能会大幅度降低，但带宽仍然非常昂贵。

另一种获得卫星IP范围内的C&C服务器方法是在卫星运营商和受害者注入沿途包劫持流量。这需要卫星提供商本身或者途径的其它ISP的协助。

这种类型的劫持攻击在2013年已经被[Renesys的博客](http://research.dyn.com/2013/11/mitm-internet-hijacking/)记录了下来。

据Renesys说：“各个提供商的BGP路由被劫持，其结果是他们部分网络流量被误导流经白俄罗斯和冰岛的ISP。我们有BGP路由数据记录了2013年2月21日和5月份俄罗斯事件和2013年七月到八月份冰岛事件的演进过程。”

在2015年博客文章中，Dyn研究人员指出：“安全分析人员检查警报日志的时候要意识到发生事件的IP地址来源通常是可以伪造的。例如，来自New Jersey的Comcast IP地址的攻击，其攻击者可能正位于Eastern Europe，只是简单征用一下Comcast IP地址。有趣的是，上面讨论的六个样例均来自Europe或Russia。”

显然，这种极其明显和大规模的攻击几乎无法幸存太长时间，而这是运行APT的关键要求之一。因此，通过MitM劫持流量不是一个可行的方案，除非攻击者直接控制一些诸如骨干路由器和光纤高流量的网络点。有迹象表明，这种攻击越来越普遍，但有一个更简单的方法--劫持卫星网络。

![](http://drops.javaweb.org/uploads/images/99756fb29f6599cccdb902e8d98439e479970e77.jpg)

0x03 劫持卫星链路（DVB-S）
=====

S21Sec的研究人员Leonardo Nve Egea在过去已经介绍了几次卫星DVB-S链路劫持，详情查看：[hijacking satellite DVB links was delivered at BlackHat 2010](https://www.blackhat.com/presentations/bh-dc-10/Nve_Leonardo/BlackHat-DC-2010-Nve-Playing-with-SAT-1.2-wp.pdf)。

劫持卫星DVB-S链路，需要执行以下操作：

*   卫星盘 - 大小取决于地理位置和卫星
*   低噪声下变频器（LNB）
*   专用的DVB-S调谐器（PCIe卡）
*   一台PC，最好是运行Linux的PC。

卫星盘和LNB最少要符合标准。TBS卡可能是最重要的组件。目前最好的DVB-S卡是[TBS Technologies](http://www.tbsdtv.com/)生产的。[TBS-6922SE](http://www.tbsdtv.com/products/tbs6922se-dvb-s2-tv-tuner-pcie-card.html)也许是最好的入门级卡。

**TBS-6922SE PCIe card for receiving DVB-S channels**![](http://drops.javaweb.org/uploads/images/d0fc2caf9a065a136c307b9bcb02bad85702bcca.jpg)

上面所述的TBS卡特别适合这项任务，因为它有专门的Linux内核驱动并提供一个暴力扫描函数，允许测试宽频范围的信号。当然，使用其它PCI或PCIe卡可能也没问题。但是基于USB的卡相对较差，应该避免使用他们。

不同于双全工卫星网络，Internet的downstream-only链路是用于加速Internet下载速度，它非常便宜而且容易部署。而且它们本身也是不安全的，流量没有经过加密处理。这创造了被利用的可能性。

公司提供Internet downstream-only访问权限传送点跟卫星来往。卫星广播流量较大的在地面上，Ku波段（12-18Ghz）则通过某些IP类传送点路由。

0x04 如何劫持卫星网络？
=====

![](http://drops.javaweb.org/uploads/images/622efc05f42cd69a51e57993ea0a1f7ecfb12b32.jpg)

为了攻击基于卫星的Internet，这些链路的合法用户以及攻击者自己的卫星盘需要指向特定卫星广播流量，攻击者利用这些未加密的数据包，确定通过卫星的下行链路路由的IP地址，然后监听这个来自Internet的IP地址的数据包，一旦接收到TCP/IP的SYN包，他们就可以伪造一个TCP的SYN ACK应答包重新建立Internet链路。

同时，链路的合法用户只是忽略所述的包，因为它流向没有开放的端口，例如80或10080端口。这里有个值得观察的地方：通常情况下，如果一个数据包到达一个未开放的端口，会返回一个RST或FIN包到源地址表示这里并不期待有数据包过来。然而对于慢速链路来说，防火墙通常是简单地丢弃了那些发送到未开放端口的数据包。这将创造一个滥用的机会。

0x05 受到侵害的Intenet范围
=====

在分析过程中，我们观察到Turla团队利用几个卫星DVB-S Internet提供商，其中大部分是提供中东和非洲地区的downstream-only链路。有趣的是，覆盖地区并不包括欧洲和非洲。这意味着在中东或非洲需要一个卫星盘。另外，一个更大的卫星盘（3米+）可于增强其它地区信号。

为了计算卫星盘的大小，可以使用各种工具，包括诸如satbeams.com的在线工具：

**Sample dish calculation – (c) www.satbeams.com**![](http://drops.javaweb.org/uploads/images/7ffc86a48da9fe85e95e32ced74b0c10cb552e8d.jpg)

下表显示了Turla攻击者用域名解析器获得的卫星Internet服务提供商那些与C&C服务器相关的IP地址。

**Note: 84.11.79.6 is hardcoded in the configuration block of the malicious sample.**![](http://drops.javaweb.org/uploads/images/da060f8fd5d2f436ea8fae293383e1f651e2ca03.jpg)

下面是所观察到的卫星IP地址相关信息：

![](http://drops.javaweb.org/uploads/images/f748355243d01a8de22f12403c6d99eca7b0a2ed.jpg)

84.11.79.6这个IP比较有趣，它属于IABG mbH的卫星IP范围。

这个C&C服务器IP地址是加密的，Turla团队在一个Agent.DNE后门中使用到：

**Agent.DNE C&C configuration**![](http://drops.javaweb.org/uploads/images/992de32634e8bdc6c1560519991cb53958eaa868.jpg)

这个Agent.DNE样本中的编译时间戳是2007年11月22日14点34分15秒星期四。说明Turla团队已使用了近八年的卫星Internet链路。

0x06 结论
=====

这些链路通常都高达数个月，但从来不会太久。具体多久无法确定，因为有可能是服务器自身的业务安全限定或者是其它方由于要进行其它恶意行为而关闭。

实现这些Internet链路的技术方法依赖于劫持各个ISP的下行带宽。这是一种在技术上比较容易实现的方法。并可能提供比任何其它常规租用的VPS和黑客合法的服务器更高程度的匿名性。

要实现这种攻击方法，初期投资少于1000美刀。一年的定期维护费少于1000美刀。考虑到该方法简单又便宜，但令人惊讶的是，我们还没看到有更多的其它APT组织使用它，尽管它提供了无与伦比的匿名性。原因是它需要依赖于防弹主机（bullet-proof hosting），多级代理或肉鸡网站。事实上，Turla团队已经知道使用所有这些技术，使其成为一个非常通用的，动态的，灵活的网络间谍活动运转机器。

最后应该指出的是，Turla不是使用基于卫星Internet链路唯一的APT团队。基于C&C的黑客团队可以参看上面的卫星IP，里面有Xumuxu团队和新近的Rocket kitten APT团队。

如果此方法在APT团队和网络犯罪组织大量使用，这将给IT安全和counter-intelligence社区造成严重后果。

_Turla团队使用的基于卫星的互联网链路的论文全文适用于卡巴斯基情报服务的客户_

0x07 IOCs
=========

*   **IPs**：
    
    ```
    84.11.79.6
    41.190.233.29
    62.243.189.187
    62.243.189.215
    62.243.189.231
    77.246.71.10
    77.246.76.19
    77.73.187.223
    82.146.166.56
    82.146.166.62
    82.146.174.58
    83.229.75.141
    92.62.218.99
    92.62.219.172
    92.62.220.170
    92.62.221.30
    92.62.221.38
    209.239.79.121
    209.239.79.125
    209.239.79.15
    209.239.79.152
    209.239.79.33
    209.239.79.35
    209.239.79.47
    209.239.79.52
    209.239.79.55
    209.239.79.69
    209.239.82.7
    209.239.85.240
    209.239.89.100
    217.194.150.31
    217.20.242.22
    217.20.243.37
    
    ```
*   **Hostnames**:
    
    ```
    accessdest.strangled[.]net
    bookstore.strangled[.]net
    bug.ignorelist[.]com
    cars-online.zapto[.]org
    chinafood.chickenkiller[.]com
    coldriver.strangled[.]net
    developarea.mooo[.]com
    downtown.crabdance[.]com
    easport-news.publicvm[.]com
    eurovision.chickenkiller[.]com
    fifa-rules.25u[.]com
    forum.sytes[.]net
    goldenroade.strangled[.]net
    greateplan.ocry[.]com
    health-everyday.faqserv[.]com
    highhills.ignorelist[.]com
    hockey-news.servehttp[.]com
    industrywork.mooo[.]com
    leagueoflegends.servequake[.]com
    marketplace.servehttp[.]com
    mediahistory.linkpc[.]net
    music-world.servemp3[.]com
    new-book.linkpc[.]net
    newgame.2waky[.]com
    newutils.3utilities[.]com
    nhl-blog.servegame[.]com
    nightstreet.toh[.]info
    olympik-blog.4dq[.]com
    onlineshop.sellclassics[.]com
    pressforum.serveblog[.]net
    radiobutton.mooo[.]com
    sealand.publicvm[.]com
    securesource.strangled[.]net
    softstream.strangled[.]net
    sportacademy.my03[.]com
    sportnewspaper.strangled[.]net
    supercar.ignorelist[.]com
    supernews.instanthq[.]com
    supernews.sytes[.]net
    telesport.mooo[.]com
    tiger.got-game[.]org
    top-facts.sytes[.]net
    track.strangled[.]net
    wargame.ignorelist[.]com
    weather-online.hopto[.]org
    wintersport.mrbasic[.]com
    x-files.zapto[.]org
    
    ```
*   **MD5s**:
    
    ```
    0328dedfce54e185ad395ac44aa4223c
    18da7eea4e8a862a19c8c4f10d7341c0
    2a7670aa9d1cc64e61fd50f9f64296f9
    49d6cf436aa7bc5314aa4e78608872d8
    a44ee30f9f14e156ac0c2137af595cf7
    b0a1301bc25cfbe66afe596272f56475
    bcfee2fb5dbc111bfa892ff9e19e45c1
    d6211fec96c60114d41ec83874a1b31d
    e29a3cc864d943f0e3ede404a32f4189
    f5916f8f004ffb85e93b4d205576a247
    594cb9523e32a5bbf4eb1c491f06d4f9
    d5bd7211332d31dcead4bfb07b288473
    
    ```
*   **卡巴斯基实验室检测到上述的Turla样本**：
    
    ```
    Backdoor.Win32.Turla.cd
    Backdoor.Win32.Turla.ce
    Backdoor.Win32.Turla.cl
    Backdoor.Win32.Turla.ch
    Backdoor.Win32.Turla.cj
    Backdoor.Win32.Turla.ck
    Trojan.Win32.Agent.dne
    
    ```

0x08 参考
=====

1.  [Agent.btz: a Source of Inspiration?](https://securelist.com/blog/virus-watch/58551/agent-btz-a-source-of-inspiration/)
2.  [The Epic Turla operation](https://securelist.com/analysis/publications/65545/the-epic-turla-operation/)
3.  [The ‘Penquin’ Turla](https://securelist.com/blog/research/67962/the-penquin-turla-2/)