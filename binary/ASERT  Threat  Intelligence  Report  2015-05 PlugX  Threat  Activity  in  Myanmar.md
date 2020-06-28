# ASERT  Threat  Intelligence  Report  2015-05 PlugX  Threat  Activity  in  Myanmar

0x01 概要
=====

缅甸目前是一个从事重要政治活动的国家。2011年的民主改革更是帮助政府创造了一个有利于吸引投资者的氛围。该国资源丰富，拥有多种自然资源和稳定劳动力供给【1】。尽管近来有所发展，该国还需要长期进行种族斗争和内战。分析人士认为，中国和美国对缅甸的内部斗争有很大的影响，尤其是中国有海上通道，港口贸易和重要的燃料管道等优势。地理政治学家认为，美国可能会因为自己的利益而在这里阻挠中国的野心【2】，【3】，【4】。

APT组织成员来自多个国家，其中包括中国。该组织的战略利益是以恶意软件作为基础进行间谍活动。使用的是恶意软件家族中的著名的PlugX（也称为Korplug），该恶意软件允许完全访问受害者的机器和网络。最近观察到在缅甸政府主站上托管了多个PlugX相关的恶意软件。在2015年八月初的时候，Arbor ASERT已经提供信息帮助缅甸CERT处理这种情况。初步局势现已得到处理。我们可以更广泛地对外发布此信息。这篇报告并不打算全面揭露持续整个活动的威胁，但是这些威胁源TPP's（Tactics, Techniques, Procedures）信息可以帮助其他组织机构提高防范意识，增强对这种攻击活动的防范和检测。

对恶意软件的初步调查发现缅甸选举的相关网站是PlugX恶意软件的宿主。缅甸针对这次攻击讨论结果是类似于Palo Alto Networks在2015年六月份发布的受到Evilgrab恶意软件影响的策略性网络感染攻击（也称“watering hole”）。他们的研究还表明9002 RAT事例也是使用相同的基础设备。由于威胁环境的性质所限，归因的查找相当困难，尤其是当多个威胁源组织可能分布在一个集中地方使用相同的恶意软件。但不论是谁发起的攻击，了解对方的目标和TPP's能够在突变情况下成为重要资料帮助工作人员抵御敌人。这是一场持久的智力战争。

0x02 缅甸政府站点分布的PlugX和加载器
=====

截至2015-07-30，缅甸政府的Web服务器上保存了几个PlugX相关的恶意软件。确切来说，Ministry of Information(MOI)网站托管了Myanma Motion Picture Development Department(MMPDD)相关网站URL[http://www.moi.gov[.]mm/mmpdd/sites/default/files/field](http://www.moi.gov[.]mm/mmpdd/sites/default/files/field)

**Figure  1: Screenshot of website containing malware as of July 30, 2015**

![](http://drops.javaweb.org/uploads/images/797ef9ab594896a0264c49f1ac5a28a45f7c73f1.jpg)

PlugX的二进制文件包括moigov.exe，fields.exe，fibmapp.exe三个文件。field目录最后的修改时间是2015-07-15 23:33，可以推测该网站在这个日期或之前就遭受黑手。网站是运行在Drupal，但相关的危害分析技术超出文本的范围。

**Figure 2: Parent directory reveals last modification of the directory used to store PlugX artifacts**

![](http://drops.javaweb.org/uploads/images/b891ce7eb6287f5be02c517e7c14abeaef0ed018.jpg)

**Table  1: A variety of PlugX malware-related content was observed on the Myanmar site**

![](http://drops.javaweb.org/uploads/images/4fbf3b3e1fecb5438fd7bdd437e261fe6a5c1e2c.jpg)

0x03 PlugX moigov.exe样本
=====

加载程序（MD5:a30262bf36b3023ef717b6e23e21bd30）下载一个叫moigov.exe的PlugX二进制文件（MD5: d0c5410140c15c8d148437f0f7eabcf7）。该PlugX样本具有多种配置属性。分析人员，事故救援人员和研究人员可以通过使用Volatility memory forensics framework（[https://github.com/arbor-jjones/volatility_plugins/blob/master/plugx.py](https://github.com/arbor-jjones/volatility_plugins/blob/master/plugx.py)和[https://github.com/arbor-jjones/volatility_plugins/blob/master/plugx_structures.py](https://github.com/arbor-jjones/volatility_plugins/blob/master/plugx_structures.py)）获取其内部信息。

应该注意到这种情况下PlugX是P2P的变种，它的P2P功能在配置里面是禁用的。这些配置elements对Indicators of Compromise（IOCs）非常有用，还有可以帮助连接该PlugX样本到其它攻击活动。分析人员必须谨慎使用，因为不是所有elements都是无害的。例如，谷歌开放的DNS服务器8.8.8.8和8.8.4.4是无害的，但是通常使用PlugX或其它恶意软件会阻碍DNS过滤/探测或者使之探测到有危害的DNS服务器。同往常一样，分析时需要多加谨慎考虑。

**PlugX configuration for moigov.exe, MD5 d0c5410140c15c8d148437f0f7eabcf7**

```
md5:  d0c5410140c15c8d148437f0f7eabcf7
cnc:  usacia.websecexp.com:53
cnc:  appeur.gnway.cc:90
cnc:  webhttps.websecexp.com:443
cnc:  usafbi.websecexp.com:25
cnc1:  webhttps.websecexp.com:443  (TCP  /  HTTP)
cnc2:  usafbi.websecexp.com:25  (UDP)
cnc3:  usacia.websecexp.com:53  (HTTP  /  UDP)
cnc4:  appeur.gnway.cc:90  (TCP  /  HTTP)
cnc5:  usafbi.websecexp.com:25  (TCP  /  HTTP)
cnc6:  webhttps.websecexp.com:443  (HTTP  /  UDP)
dns:  180.76.76.76
dns:  168.126.63.1
dns:  203.81.64.18
dns:  8.8.8.8
enable_icmp_p2p:  0
enable_ipproto_p2p:  0
enable_p2p_scan:  0
enable_tcp_p2p:  0
enable_udp_p2p:  0
flags1:  4294967295
flags2:  0
hide_dll:  -1
http:  http://epn.gov.co/plugins/search/search.html
icmp_p2p_port:  1357
injection:  1
inject_process:  %ProgramFiles%\Internet  Explorer\iexplore.exe
inject_process:  %windir%\system32\svchost.exe
inject_process:  %windir%\explorer.exe
inject_process:  %ProgramFiles(x86)%\Windows  Media  Player\wmplayer.exe
install_folder:  %AUTO%\McAfeeemOS
ipproto_p2p_port:  1357
keylogger:  -1
mac_disable:  00:00:00:00:00:00
mutex:  Global\VdeBueElStlKd
persistence:  Service  +  Run  Key
plugx_auth_str:  open
reg_hive:  2147483649
reg_key:  Software\Microsoft\Windows\CurrentVersion\Run
reg_value:  OmePlus
screenshot_folder:  %AUTO%\McAfeeemOS\NtBXvdMGwtDwrfHs
screenshots:  0
screenshots_bits:  16
screenshots_keep:  3
screenshots_qual:  50
screenshots_sec:  10
screenshots_zoom:  50
service_desc:  McAfee  OmePlus  Module
service_display_name:  McAfee  OmePlus  Module
service_name:  McAfee  OmePlus  Module
sleep1:  83886080
sleep2:  0
tcp_p2p_port:  1357
uac_bypass_inject:  %windir%\system32\rundll32.exe
uac_bypass_inject:  %windir%\system32\dllhost.exe
uac_bypass_inject:  %windir%\explorer.exe
uac_bypass_inject:  %windir%\system32\msiexec.exe
uac_bypass_injection:  1
udp_p2p_port:  1357

```

0x04 PlugX fields.exe样本
=====

有几个原因使得fields.exe样本比较出名。它同样存在缅甸的moi.gov网站，和上述的moigov.exe样本有差不多一样的结构。但是有些elements是不同的，比如C2认证字符串，注入的进程表，安装文件夹等方面。

**PlugX configuration for fields.exe, MD5 809976f3aa0ffd6860056be3b66d5092**

```
md5:  809976f3aa0ffd6860056be3b66d5092
cnc: appeur.gnway.cc:90
cnc: webhttps.websecexp.com:443
cnc: usacia.websecexp.com:53
cnc: usafbi.websecexp.com:25
cnc1: webhttps.websecexp.com:443 (TCP / HTTP)
cnc2: usafbi.websecexp.com:25 (UDP)
cnc3: usacia.websecexp.com:53 (HTTP / UDP)
cnc4: appeur.gnway.cc:90 (TCP / HTTP)
cnc5: usafbi.websecexp.com:25 (TCP / HTTP)
cnc6: webhttps.websecexp.com:443 (HTTP / UDP)
cnc_auth_str: Kpsez-htday
dns: 168.126.63.1
dns: 180.76.76.76
dns: 8.8.8.8
dns: 203.81.64.18
enable_icmp_p2p: 0
enable_ipproto_p2p: 0
enable_p2p_scan: 0
enable_tcp_p2p: 0
enable_udp_p2p: 0
flags1: 4294967295
flags2: 0
hide_dll: -1
http: http://epn.gov.co/plugins/search/search.html
icmp_p2p_port: 1357
injection: 1
inject_process: %windir%\system32\svchost.exe
inject_process: %ProgramFiles%\Internet Explorer\iexplore.exe
inject_process: %windir%\explorer.exe
inject_process: %ProgramFiles(x86)%\Windows Media Player\wmplayer.exe
install_folder: %AUTO%\MybooksApp
ipproto_p2p_port: 1357
keylogger: -1
mac_disable: 00:00:00:00:00:00
mutex: Global\EStZmOzInezFVydxhdE
persistence: Service + Run Key
plugx_auth_str: open
reg_hive: 2147483649
reg_key: Software\Microsoft\Windows\CurrentVersion\Run
reg_value: OSEMInfo
screenshot_folder: %AUTO%\MybooksApp\hIZu
screenshots: 0
screenshots_bits: 16
screenshots_keep: 3
screenshots_qual: 50
screenshots_sec: 10
screenshots_zoom: 50
service_desc: Windows OSEMinfo Service
service_display_name: McAfee OSEM Info
service_name: McAfee OSEM Info
sleep1: 83886080
sleep2: 0
tcp_p2p_port: 1357
uac_bypass_inject: %windir%\explorer.exe
uac_bypass_inject: %windir%\system32\dllhost.exe
uac_bypass_inject: %windir%\system32\msiexec.exe
uac_bypass_inject: %windir%\system32\rundll32.exe
uac_bypass_injection: 1
udp_p2p_port: 1357

```

一个有趣的element是C2验证字符串"Kpsez-htday"，它可能引用了缅甸Rakhine State的Kyaukphyu Township。这是一个经济特区（SEZ）。其相关信息可以参看下图【6】：

**Figure 3: About KP SEZ from http://kpsez.org/en/about-us-2/**

![](http://drops.javaweb.org/uploads/images/8f1dc1836392e7be05c8f90a2948d895de52241c.jpg)

基于先来者们使用PlugX的历史【7】【8】【9】【10】和该经济特区特性，为选择有利于民族国家利益而实施泄露和间谍行动，具体情况还要进一步调查。

0x05 PlugX fibmapp.exe样本
=====

**PlugX configuration, MD5 69754b86021d3daa658da15579b8f08a**

```
md5:  69754b86021d3daa658da15579b8f08a
cnc:  appeur.gnway.cc:90
cnc:  webhttps.websecexp.com:443
cnc:  usacia.websecexp.com:53
cnc:  usafbi.websecexp.com:25
cnc1:  webhttps.websecexp.com:443 (TCP / HTTP)
cnc2:  usafbi.websecexp.com:25 (UDP)
cnc3:  usacia.websecexp.com:53 (HTTP / UDP)
cnc4:  appeur.gnway.cc:90 (TCP / HTTP)
cnc5:  usafbi.websecexp.com:25 (TCP / HTTP)
cnc6:  webhttps.websecexp.com:443 (HTTP / UDP)
cnc_auth_str:  EDMS GM716
dns:  168.126.63.1
dns:  180.76.76.76
dns:  8.8.8.8
dns:  203.81.64.18
enable_icmp_p2p:  0
enable_ipproto_p2p:  0
enable_p2p_scan:  0
enable_tcp_p2p:  0
enable_udp_p2p:  0
flags1:  4294967295
flags2:  0
hide_dll:  -1
http:  http://epn.gov.co/plugins/search/search.html
icmp_p2p_port:  1357
injection:  1
inject_process:  %windir%\system32\svchost.exe
inject_process:  %ProgramFiles%\Internet Explorer\iexplore.exe
inject_process:  %windir%\explorer.exe
inject_process:  %ProgramFiles(x86)%\Windows Media Player\wmplayer.exe
install_folder:  %AUTO%\EDMSinfos
ipproto_p2p_port:  1357
keylogger:  -1
mac_disable:  00:00:00:00:00:00
mutex:  Global\qZlDbiNRvrLXkhFTgAhdIeESC
persistence:  Service + Run Key
plugx_auth_str:  open
reg_hive:  2147483649
reg_key:  Software\Microsoft\Windows\CurrentVersion\Run
reg_value:  EDMSinfos
screenshot_folder:  %AUTO%\EDMSinfos\NHY
screenshots:  0
screenshots_bits:  16
screenshots_keep:  3
screenshots_qual:  50
screenshots_sec:  10
screenshots_zoom:  50
service_desc:  Windows EDMSinfos Service
service_display_name:  EDMSinfos Module
service_name:  EDMSinfos Module
sleep1:  83886080
sleep2:  0
tcp_p2p_port:  1357
uac_bypass_inject:  %windir%\explorer.exe
uac_bypass_inject:  %windir%\system32\dllhost.exe
uac_bypass_inject:  %windir%\system32\msiexec.exe
uac_bypass_inject:  %windir%\system32\rundll32.exe
uac_bypass_injection:  1
udp_p2p_port:  1357

```

0x06 近来缅甸受到Evilgrab的攻击的TTP's
=====

根据配置的信息来看，这件事情可能和六月份Palo Alto Networks【5】发布的使用Evilgrab恶意软件对缅甸总统网站发起的策略性网络感染攻击（SWC）相关。

这种攻击在目标网站上添加一个iframe。这次攻击中，一个iframe也添加到www.moi.gov.mm网站，通过分析MOI网站可以发现以下脚本被index.html?q=content%2Fmmpdd-e-services页面调用。

![](http://drops.javaweb.org/uploads/images/88a64dc159a22edac5ca0efa7eb8be4264d2963e.jpg)

custom.js最后一行包含一个隐藏的iframe，指向Drupal themes文件夹里面html5.php：

![](http://drops.javaweb.org/uploads/images/dc1d3d3da9072b4dab60c1b7a04fa44fbcefbef6.jpg)

html5.php在目标网站上已经不再有效（可能是被攻击者、网站维护人员移除或者限定到指定的目标），VirusTotal检查结果表明其它html5.php文件可能提供其它服务。特别是一个MD5为a1c0c364e02b3b1e0e7b8ce89b611b53的html5.php文件包含了一个捆绑了PlugX的Firefox浏览器插件。这个浏览器插件只是复制PlugX二进制文件（名字为 Components.exe）到C:\Windows\tasks目录然后执行它。具体代码在boostrap.js里面：

![](http://drops.javaweb.org/uploads/images/63979865a1e631bc1e0c9b6b67a892cfbc091747.jpg)

Components.exe的MD5是1c7fafe58caf55568bd5f28cae1c18fd。这个特殊的二进制文件似乎没有攻击缅甸的相关动作。然而它捆绑PlugX的策略、文件名和web感染方法都和Palo Alto Networks发布的Evilgrab攻击方法吻合。

如本文所述的攻击者利用浏览器插件的方法同时在Shadowserver上被发现。不久之后将发布他们的研究结果，Defenders支持审查这些资料，当发布的时候可以获得额外的攻击活动信息。

除了上面提到的技术，在Evilgrab攻击和PlugX配置观察到相同的C2服务器，这表明一些攻击者团队或者攻击者团队之间共享攻击时的基础设备，相关资料如下：

*   usafbi.websecexp[.]com (port 53)
*   usacia.websecexp[.]com (UDP/53)
*   webhttps.websecexp[.]com (port 443)
*   appeur.gnway[.]cc (TCP/90)

0x07 可能相关的PlugX恶意软件
=====

配置清单还列出了另一个DNS服务器（203.81.64.18）。运行在缅甸邮电可能是因为比起其它DNS服务器会受到较少的怀疑。至少了四个PlugX样本使用这个DNS服务器。CA验证字符串可以参看下表：

![](http://drops.javaweb.org/uploads/images/54df8446f7a067a2171c2f84f55228d096196606.jpg)

在#1样本的C2验证字符串分析表明其日期可能是4月9日（04-09）和4月20日（04-20），样本#2包含了2015-02-24的时间戳。样本#3的验证字符串可能指的是3月12日和3月20日。但目前ASERT缺乏证据证明这些活动是在这些日期。样本#4（在上述的Myanmar.gov网站发现）可能指的是通用术语“Electronic Document  Management  System”，GM716可能是7月16日。EDMS可能跟缅甸政府的EDMS相关【11】，【12】，虽然没证据证明这种说法。

这个表的第一个恶意软件（eeb631127f1b9fb3d13d209d8e675634）在[http://the-casgroup[.]com/Document/doc/dxls.exe](http://the-casgroup[.]com/Document/doc/dxls.exe)发现并于2015年4月20日首次提交到VirusTotal。

这个网站似乎属于“CAS GROUP INTERNATIONAL LIMITED”，其自我介绍下：“The CAS Group brings along a number of world innovative home automation,audio speakers and digital signag products from the USA under one roof into Hong Kong”[http://the-casgroup.com/about.php](http://the-casgroup.com/about.php])。2015-03-30的时候分析发现这个domain连接到我们与其它恶意样本（MD5:  e2eddf6e7233ab52ad29d8f63b1727cd），其功能似乎是下载[http://the-casgroup[.]com/Document/doc.zip](http://the-casgroup[.]com/Document/doc.zip)。恶意软件伪装成一个假的JPEG文件（invitations.jepg），正如我们可以从截图看到RUNDLL试图运行invitations.jpeg.dll。一些有趣的字符串（sanitized）样本如下所示：

```
Downer.dll
%s\Thumbs.db
%s//%s?%d
http://the-casgroup[.]com/Document
http://the-casgroup[.]com/Document/doc.zip
%s\doc.zip
Java  Sun
Thumbs.db Mode
Sun_FlashUpdate.lnk

```

![](http://drops.javaweb.org/uploads/images/8d370915c3237cf21c853746fa151623d345e661.jpg)

这个恶意样本发现自缅甸的Naypyitaw联邦选举委员会网站[http://www.uecmyanmar[.]org/dmdocuments/invitations.rar](http://www.uecmyanmar[.]org/dmdocuments/invitations.rar)RAR文件（MD5: d055518ad14f3d6c40aa6ced6a2d05f2）。2015-07-30的时候，这个RAR文件仍然还在这个网站上。档案的名称是“Preliminary discussions about the election,invitations\Preliminary discussions about the election,invitations.lnk”。.lnk文件显示的修改时间是2015-03-25 2:35:36PM 星期三。.lnk的目标地址是：“C:\WINDOWS\system32\rundll32.exe invitations.jpeg Mode”，在DLL里面使用Mode功能执行invitations.jepg。

档案还包含一个Readme.txt文件，为确保恶意软件的执行包含了以下的措辞。

![](http://drops.javaweb.org/uploads/images/ec1b42846370397755c0effa0feb858cf5127185.jpg)

在同一个网站发现[http://www.uecmyanmar[.]org/dmdocuments/PlanProposal.rar](http://www.uecmyanmar[.]org/dmdocuments/PlanProposal.rar)。当解压之后得到另一个PlugX样本。RAR包含的三个文件如下：

**Figure 4: Malware found inside RAR file hosted on uecmyanmar site**

![](http://drops.javaweb.org/uploads/images/b32a53ebe99279b75a238395db302d33eef2a181.jpg)

这三个文件解压到“PlanProposal\new questionnaire\Voter Plan Proposal”目录，说明他们的目标可能是操作缅甸的参与投票。

还有PlugX配置还包含了一个Epn.gov.co(the National Penitentiary School for the National Penitentiary and Prison Institute (INPEC) in Colombia)的HTTP的配置element，这个字段的意图是当C2没有反应的时候提供一个命令或控制服务。这个URL的内容是这样的：

**Figure 5: External site hosting PlugX Command & Control servers in an encoded form**

![](http://drops.javaweb.org/uploads/images/83c384a6b01ec927399d9059eeb0d240de674dad.jpg)

在其他一些诸如：“PlugX:some uncovered points"的elements被Airbus Defence and Space【13】和应急人员【14】作为讨论的焦点。Volatility memory forensics framework可能是使用ASERT的plugx.py和plugx_structures.py Volatility插件分析PlugX配置elements【15】。每行的开头和结尾四个字节包含C2编码信息，之前的版本和端口信息。

Fireeye在2014年八月份开源的解码脚本【16】只需稍作修改便可用于这里。ASERT修改了FireEye python脚本（plugx_c2_decode.py）的header，版本信息，端口信息，开始几个字节和返回唯一的主机名。

```
import sys
s = sys.argv[1][10:-4]
rvalue = ""
for x in range(0, len(s), 2):
  tmp0 = (ord(s[x+1]) - 0x41) << 4
  rvalue += chr(ord(s[x]) + tmp0 - 0x41)
print rvalue

```

python  plugx_c2_decode.py  DZKSEAAAJBAAFHDHBGGGCGJGOCHHFGCGDHFGDGFGIHAHOCDGPGNGDZJS**usafbi.websecexp.com**

python  plugx_c2_decode.py  DZKSGAAAFDAAFHDHBGDGJGBGOCHHFGCGDHFGDGFGIHAHOCDGPGNGDZJS**usacia.websecexp.com**

PlugX使用者用Colombian政府网站指向该网站，但Colombian的分析目标在这个报告之外。

0x08 建议
=====

缅甸内部相关的机构应该意识到，攻击者的目的是本文所述的所有特殊邮件信息或网络流量。相关机构应当了解PlugX的网络流量，并应监控本文所述的PlugX配置数据相关的主机和网络。另外，监控JPCERT【17】所说的P2P PlugX也是一个明智的选择，尽管这个简单的PlugX样本P2P功能是被禁用的。PlugX只是恶意软件系列中的一员，攻击者通常有树种恶意软件可以选择。鉴于PlugX攻击活动具有针对性，攻击活动可能会持续。IOCs包含本书所述可能遭到攻击的组织，发现系统受损程度、以及所造成的损失以及组织应急措施。

应急处理人员应该了解目标的geopolitical，采取适当的方式阻碍他们的行动。在这种情况下适当的处理方式是寻找PlugX（和其它恶意软件）和任何已经有入侵迹象的系统和网站。如果包含恶意活动的日志文件可用，可以利用他们来确定威胁活动。这使得救援人员可以跟踪spear-phish和其它攻击方式，从而了解一些可以帮助组织更好地防御危害的信息，进而限制他们泄漏敏感数据。

0x09 附录1：”Connection Test.exe“恶意软件下载器技术分析
=====

2015年7月2日，一个名为moigov.exe的文件试图下载来自MOI网站一个2015年6月23日编译的恶意软件下载器。分析时，如果下载这个文件返回404错误，则该文件很可能已经被攻击者删除了。恶意软件下载器原有的名字被VirusTotal检测到是”Connection Test.exe“，更多细节请参考【18】。这个程序伪装成IBM安全软件AppScan，并且使用相同的文件名，版权，版本号，出版商和产品值 (8.0.650.113)（参考binarydb.com 【19】）。

因为AppScan常常被开发人员和安全人员用来查找漏洞，很可能这场攻击活动需要更高级别的资源访问权限。

AppScan和恶意软件的比较如下所示：

**Figure 6: Legitimate AppScan binary**

![](http://drops.javaweb.org/uploads/images/4b6a75a6be6f1842a6d1f3f1b0dc0ad1b1b405e6.jpg)

**Figure 7: Bogus AppScan binary that downloads PlugX**

![](http://drops.javaweb.org/uploads/images/f4a766457d3630d12adb8a48b642ecb423d33323.jpg)

两个程序的图标：

![](http://drops.javaweb.org/uploads/images/5aeb9b5b99922d41fc5d83a4d249e5b41030b4fb.jpg)

我们这里选择介绍的恶意软件下载器非常普通，使用的是2013年Arbor ASERT的成员Jason Jones所说的字节串技术【21】。字节串技术常常被各种恶意软件用于混淆代码。Deobfuscation可以手动为之，然而有一个python脚本实现快速Deobfuscation【22】。独特的字节串技术可以很容易混淆代码。

值得一提的是，一些中国APT恶意软件被检测到也使用该技术，但不仅限于中国。有几个基于Delphi的中国恶意软件已经使用了该技术【20】。它们分别是Gh0st RAT，Poison lvy，IXESHE，Etumbot（关于Etumbot更多的细节请查看2014-07 ASERT的简报“Illuminating  the  Etumbot  APT  Backdoor”）等。

Deobfuscation之前我们看到WinMain函数的第一部分使用的字节串技术。这种情况下，AL寄存器存放“L”字符，CL寄存器存放“E”，BL寄存器存放“0”，DL寄存器存放“o”。这些值结合各种静态字符串来调用LoadLibraryA和GetProcAddress。这和ASERT中描述的技术一样：使用IDAPython查找字节串 - “我们也看到类似的代码：把字符加载到一个8位寄存器然后和变量混合起来进一步混淆代码”【21】。

**Figure  8: Byte strings technique fills AL, CL, BL, and DL one byte registers (hex on the left, ASCII on the right) with characters**

![](http://drops.javaweb.org/uploads/images/d772e45545b9202c33c1f2bd24c5ddb9400f7780.jpg)

**Figure 9: Strings being built (obfuscated on the left, deobfuscated on the right)**

![](http://drops.javaweb.org/uploads/images/6754cad33590bb0fc6759bb52a9a4e133561f0e1.jpg)

可以看到数据字符串组合成：kernel32.dll，urlMon.dll（作为调用LoadLibraryA的参数）。

**Figure 10: Strings being processed after deobfuscation**

![](http://drops.javaweb.org/uploads/images/cfdca7ec724ccee8faff283337cd14b950bb3405.jpg)

同样还可以看到CreateProcessA，URLDownloadToFileA(作为调用GetProcAddress的参数)。字符串加工之后，PlugX恶意下载器通过 这个URL调用UrlMod.URLDownloadToFileA函数。

**Figure 11: Downloader URL target which downloads PlugX**

![](http://drops.javaweb.org/uploads/images/59d98d1a7930ea5dfa18ae431297e8f15fe24980.jpg)

**Figure 12: Calls to LoadLibraryA and GetProcAddress receive the dynamically created strings**

![](http://drops.javaweb.org/uploads/images/c2e0ac2ea47d6402e5460a4ddabae374b83aafa3.jpg)

**Figure 13: The next function (4011F0) specifies the malware download location**

![](http://drops.javaweb.org/uploads/images/e78b3962df746f5270f996e16ac97936bab262f8.jpg)

虽然通过IDA pro或者其它静态分析器可以获取到下载器的详细信息。但更方便的方法是使用调试器。地址00403000显示了下载的URL。顺便说一下，里面还包含了“Hello，World！”字符串。

**Figure 14: .data section reveals downloader details**

![](http://drops.javaweb.org/uploads/images/1c713bd0d9deff7c23cd1bd73c614c6c78009099.jpg)

此字符串叠加技术有一个方便的特点是，它留下的的机器码很独特，可以通过一定的规则来检测。虽然这些规则有用，但是生成这些恶意下载器可能一些使用随机的寄存器或随机的字节，导致代码进一步的混乱。

0x0A 参考
=====

1.  [http://thediplomat.com/2014/06/changing-dynamics-in-myanmar-impact-bangladeshs-geopolitics/](http://thediplomat.com/2014/06/changing-dynamics-in-myanmar-impact-bangladeshs-geopolitics/)
2.  [http://www.wantchinatimes.com/news-subclass-cnt.aspx?id=20150226000021&cid=1501](http://www.wantchinatimes.com/news-subclass-cnt.aspx?id=20150226000021&cid=1501)
3.  [http://www.ipcs.org/article/peace-and-conflict-database/the-role-of-geopolitics-in-myanmars-resource-curse-4852.html](http://www.ipcs.org/article/peace-and-conflict-database/the-role-of-geopolitics-in-myanmars-resource-curse-4852.html)
4.  [http://www.eastasiaforum.org/2015/03/06/china-and-myanmar-when-neighbours-become-good-friends/](http://www.eastasiaforum.org/2015/03/06/china-and-myanmar-when-neighbours-become-good-friends/)
5.  [http://researchcenter.paloaltonetworks.com/2015/06/evilgrab-delivered-by-watering-hole-attack-on-president-ofmyanmars-website/](http://researchcenter.paloaltonetworks.com/2015/06/evilgrab-delivered-by-watering-hole-attack-on-president-ofmyanmars-website/)
6.  [http://kpsez.org/en/about-us-2/](http://kpsez.org/en/about-us-2/)
7.  [https://www.fireeye.com/blog/threat-research/2014/07/pacific-ring-of-fire-plugx-kaba.html](https://www.fireeye.com/blog/threat-research/2014/07/pacific-ring-of-fire-plugx-kaba.html)
8.  [http://www.esecurityplanet.com/malware/report-plugx-is-rat-of-choice-for-nation-states.html](http://www.esecurityplanet.com/malware/report-plugx-is-rat-of-choice-for-nation-states.html)
9.  [http://www.infosecurity-magazine.com/news/china-vietnam-and-plugx-dominate/](http://www.infosecurity-magazine.com/news/china-vietnam-and-plugx-dominate/)
10.  [https://blogs.sophos.com/tag/plugx/](https://blogs.sophos.com/tag/plugx/)
11.  [http://www.mcit.gov.mm/content/egovernment.html](http://www.mcit.gov.mm/content/egovernment.html)
12.  [http://www.mcit.gov.mm/sites/default/files/edms%20manual.pdf](http://www.mcit.gov.mm/sites/default/files/edms%20manual.pdf)
13.  [http://blog.cassidiancybersecurity.com/post/2014/01/plugx-some-uncovered-points.html](http://blog.cassidiancybersecurity.com/post/2014/01/plugx-some-uncovered-points.html)
14.  [http://www.slideshare.net/takahiroharuyama5/i-know-you-want-me-unplugging-plugx](http://www.slideshare.net/takahiroharuyama5/i-know-you-want-me-unplugging-plugx)
15.  [https://github.com/arbor-jjones/volatility_plugins](https://github.com/arbor-jjones/volatility_plugins)
16.  [https://www.fireeye.com/blog/threat-research/2014/08/operation-poisoned-hurricane.html](https://www.fireeye.com/blog/threat-research/2014/08/operation-poisoned-hurricane.html)
17.  [http://blog.jpcert.or.jp/2015/01/analysis-of-a-r-ff05.html](http://blog.jpcert.or.jp/2015/01/analysis-of-a-r-ff05.html)
18.  [https://www.virustotal.com/en/file/ac5db170487d1a789e8b5fb1cb52f7b84086b1768b25083c50309a88a7229545/analysis](https://www.virustotal.com/en/file/ac5db170487d1a789e8b5fb1cb52f7b84086b1768b25083c50309a88a7229545/analysis)
19.  [http://binarydb.com/file/Connection-Test.exe-v2568653.html](http://binarydb.com/file/Connection-Test.exe-v2568653.html)
20.  [http://www.ibm.com/developerworks/library/se-sql-injection-attacks/](http://www.ibm.com/developerworks/library/se-sql-injection-attacks/)
21.  [https://asert.arbornetworks.com/asert-mindshare-finding-byte-strings-using-idapython/](https://asert.arbornetworks.com/asert-mindshare-finding-byte-strings-using-idapython/)
22.  [https://github.com/arbor/reversing/blob/master/find_byte_strings.py](https://github.com/arbor/reversing/blob/master/find_byte_strings.py)

0x0B 关于ASERT
=====

Arbor NetWorks的ASERT（Arbor Security Engineering & Response Team）是一个提供世界级的网络安全的研究和分析，帮助当今企业维护利益的网络运营商。ASERT的工程师和研究人员更是机构里面的精英团队，他们被成为“super remediators”，代表着最好的信息安全团队。其知名度和修复能力在全球的大多数网络服务提供商可以反映出来。

ASERT分享给几百名国际计算机应急反应小组（CERTs）和成千上万的网络运营商情报简要和安全内容供稿。ASERT也运营当今世界上最大的分布式Honeynet，全天候监控全球的网络威胁[http://atlas.arbor.net](http://atlas.arbor.net/)。这一使命和Arbor Networks相关的资源给全球网络安全问题带一个创新和研究的动力。

要查看Arbor近期的研究，新闻和ASERT信息安全社区请访问我们的门户[http://www.arbornetworks.com/threats/](http://www.arbornetworks.com/threats/)。