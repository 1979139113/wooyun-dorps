# 针对以色列和巴勒斯坦的apt式攻击

0x00 简介
=====

这个简短的报告介绍了一系列针对以色列和巴勒斯坦的攻击行为，通过恶意文件为传播源对大量的具有影响或者政治相关的组织，通过我们的调查，之前并无行为上与此相同的apt记录。不过还是可以找到一些相似的攻击行为。link to (1](2]

话说那是一个2014年的夏天，我们在一些小型的基础设施中获得了恶意样本，这些行为表明，攻击者是穷逼，或者被限制了可利用的资源。

最初我们对文件`ecc240f1983007177bc5bbecba50eea27b80fd3d14fd261bef6cda10b8ffe1e9`进行了分析，并且push到了 malwr.com (3]进行自动分析。顺带说下该样本的原名为`Israel Homeland Defense Directory 2015 _Secured_.exe`一旦运行会显示如下界面。

![enter image description here](http://drops.javaweb.org/uploads/images/efcc13a9b39d654feeb3fecbe452e52016a60ba0.jpg)

实际上最初的文件是一个rar的自解压文件，包含三个组件，事实上后期我们的分析师表示这种恶意软件不属于原来熟知的一些用于apt恶意软件类型，我们进行了进一步的分析希望了解其的目的，细节，与攻击目标。

0x01 进一步分析
=====

该类型的恶意软件最常见的方式是打包成一个rar压缩包，当然并不是仅限于此，攻击者也使用了一些别的手法，比如 Visual Basic 打包，和原始的安装包，我们认为传播的途径是将恶意软件push到第三份下载站点。下面的图显示了一些相关的信息

![enter image description here](http://drops.javaweb.org/uploads/images/f0f0923a8648fa34238f3bcf37971f05080252cd.jpg)

Pomf.se是一个总部设置在瑞典的小型文件共享和托管站点，使用一些小型站点作为传播途径似乎是他们的特征之一，同样的我们也从其他的一些小型站点上发现了类似的恶意软件，并且都是可执行文件。我们先对恶意文件SHA256哈希文件`ecc240f1983007177bc5bbecba50eea27b80fd3d14fd261bef6cda10b8ffe1e9`进行了分析。

我们选择了一个名称`DownExecute`用来指代恶意软件，同时我们对其所有变体进行了确认，下面的图片是共同的打包模式，此外的curl被用于网络连接，或者说用于下载最终的恶意软件。

![enter image description here](http://drops.javaweb.org/uploads/images/5457327f32fe3148203be274bd8c414dff9a0f59.jpg)

另外的发现是其中的一些二进制文件进行了自签名

![enter image description here](http://drops.javaweb.org/uploads/images/8df65d335577cc5fb8637fafac9625e718650d35.jpg)

那么，这个恶意软件能做什么，事实上他的作用只是一个下载器，同时用于进行本地环境的一个检查，其中包括了debugging的检查，使用一个叫IsDebuggerPresent的函数，检查是否存在VirtualBox，通过检查路径.\VBoxMiniRdrDN。

![enter image description here](http://drops.javaweb.org/uploads/images/8a0720b48f78ce2c1c5d5a8d3e2ad56190882a16.jpg)

同时还检查了一系列的反病毒软件的存在，检查甚至包括了存在单词'security'的进程。

![enter image description here](http://drops.javaweb.org/uploads/images/4f9ca08ea405b069f81dd0e36af0dda4237f2753.jpg)

之后恶意软件会开始解密自身的一些数据，其中包括了攻击者的控制服务器。

![enter image description here](http://drops.javaweb.org/uploads/images/a39b0a948ea8ba66f2fac6bdd3cdacfd988ee2c1.jpg)

同时恶意软件会开始连接攻击者服务器，同时会在同一个文件夹建立一个明文文本开始记录日志。。。。。(mlgb.....)

![enter image description here](http://drops.javaweb.org/uploads/images/db1170b9f6d3e39e785ac573498d1eef24869dfb.jpg)

就其初步得到的信息我们可以看出，downloadexcute的作用就跟其名字一样，用于下载另一个恶意软件并且执行，因为其本身的攻击行为几乎没有，downloadexcute像是给攻击者用来站稳脚跟的工具，只有downloadexcute成功了，才会进行下一步入侵。

如上，我们进行了大量的调查，在一些基础设施中发现了一些功能更加全面的恶意软件属于Xtreme RAT和Poison Ivy家族并且使用与downloadexcute相同的域名，我们可以认为其属于在downloadexcute之后第二阶段的而已软件。其中观察到的Poison Ivy使用`admin2014`和`admin!@#$%`作为密码

![enter image description here](http://drops.javaweb.org/uploads/images/a06468353425d36c018e209b1b51bdda7153a78c.jpg)

我们对其中连接的域名进行了调查，不过绝大多数连接的域名为动态域名。主要关联到no-ip.com。通过PwC情报小组的追踪，关联到一些中东的一些规模较小的恶意行为，通过对这些的调查，我们使用了Maltego进行了威胁可视化。

这次的攻击行动与以往的稍许不同，从以往监控到的对于中东的黑客行动，我们已经熟悉其中大部分的ip，相对与一些常被用于用于的主机商，这次主要指向伯利兹的Host Sailor。

如我们之前的报告中提到的(5]，攻击者习惯使用与目标看起来相似的域名，来其看起来稍微正常一些。基于此我们做了一些简单的分析用于这次的目标识别。对于具体的list可以参阅附录B。

![enter image description here](http://drops.javaweb.org/uploads/images/b0eb2ee8c77134594c95a2e158af33e9a1e76139.jpg)

从上面的信息中我们判断，攻击目标可能以以色列的新闻媒体为主。

之后我们对文档的内容进行了调查，其中的部分文档显然是针对以色列为主的，以军事，政治为主题。

![enter image description here](http://drops.javaweb.org/uploads/images/3a594b516041d6fb305fa0d75593529de50d1833.jpg)

其文件的主题关联到巴勒斯坦解放组织领导人Abbas，可以看出攻击者预计目标会使用流利的阿拉伯语。

0x02 结论
=====

事实上，我们无法断定这次的攻击是特别针对中东地区发起的，不过其中的几个行为模式可以给我们一些提醒。

*   不明白为什么这么喜欢使用no-ip.com和其相关的服务。不过我们发现阿拉伯地下论坛中的几个黑客特别推荐使用no-ip的服务。
*   使用公开的恶意软件，而不是自己写一个 (such as Poison Ivy/Xtreme RAT)
*   目标似乎局限于中东，或者说以色列的一些敏感问题。
*   密码模式，之前Fireeye的一篇博文(7]显示了一个团体MoleRats，使用 Poison Ivy 和Xtreme RAT，使用`!@#GooD#@!`,或者说是`!@#`作为密码的模式。

不过攻击者选择自行开发dropper可能表明了如何进入系统是他们最大的问题，(毕竟是可执行。。。),我们能找到最早的样本在2014年6月编译，之后陆续发现的其他样本，都在不久之前进行编译，我们相信这种攻击还会持续12个月。

附录A
=====

```
SHA256

8993a516404c0dd62692f3ce5055d4ddee7e29ad4bb6aa29f67114eeeaee26b9

bfe727f2f238f11eb989e5b76efd24ad2b41df3cf7dabf7077dfaace834e7f03

dad34d2cb2aa9662d4a4148481ae018f5816498f30cc7aee4919e0e9fe6b9e08

2cb9df0d52d09c98f0a97ce71eb8805f224945cadab7d615ef0257b7b09c80d3

f53fd5389b09c6ad289736720e72392dd5f30a1f7822dbc8c7c2e2b655b4dad9

1d533ddaefc7859a3f6c6751114e895b7aa5935eb0ed68b01ec61aa8560ae3d9

95b2f926ae173ab45d6dac4039f0b91eb24699e6d11b621bbcebd860752e5d5e

da63f6392ce6af83f6d944fa1bd3f28082345fec928647ee7ef9939fac7b2e6c

a7aeeead233fcdfe1c7475db982497a82d8ae745ec1c58bd87215e8869c3f9e4

2eb7aa306551d693691d14558c5dc4f6d80ef8f69cf466149fbba23953c08f7f

e945b055fb4057a396506c74f73b873694125e6178a40d10cabf24b2d89d598f

c9e084eb1ce1066ee063f860c13a8f7d2ead97495036855fc956dacc9a24ea68

047e8d542e2fcdf0f4dd45e2b19848771d01abc90d161d05242b79c52cdd248d

25e6bf67410dffb95c527c19dcff5223dbc3bf4c987650e45fbea1267072e8ff

b0edbd0f44df72e0fad3fb73948444a4df5143ed954c9116eb1a7b606841f187

da63f6392ce6af83f6d944fa1bd3f28082345fec928647ee7ef9939fac7b2e6c

de3e25a69ba43b9f236e544ece7f2da82a4fafb4489ad2e263754d9b9d88bc5c

ecc240f1983007177bc5bbecba50eea27b80fd3d14fd261bef6cda10b8ffe1e9

f969bf3b7a9821b3b2d5de889b5af7af25972b25ba59e4e9439f87fe90f1c404

14be3a9a2a4261cb365915e720486a0632dbebb06fe68fb669ae67aa9b18507b

488ba22d6cb8c9b0310c58fa4c4739692cdf45676c3164b357314322542f9dff

b3a47e0bc0af49b46bc0c1158089bf200856ff462a5334df2b5c11e69c8b1ada

324ce011b913feec4adb916f32c743a243f07dccb51b49c0122c4fa4a8e2bded

d6df5943169b48ac58fc28bb665fe8800c265b65fff8a2217b70703a4d3a7277

88e7a7e815565b92af81761ae7b9153b7507677df3d3b77e8ce68787ad1826d4

f51d4155534e10c09b531acc41458e8ff3b7879f4ee7d3ee99f16180c4caf0ee

b3a47e0bc0af49b46bc0c1158089bf200856ff462a5334df2b5c11e69c8b1ada

bc846caa05939b085837057bc4b9303357602ece83dc1380191bddd1402d4a2b

```

附录b 控制服务器
=====

过长 请参考原文。

附录c 签名
=====

Yara

```
rule DownExecute_A

{

meta:

       author = "PwC Cyber Threat Operations :: @tlansec"

       date = "2015-04"

       reference = "http://pwc.blogs.com/cyber_security_updates/2015/04/attacks-against-israeli-palestinian-interests.html"

       description = “Malware is often wrapped/protected, best to run on memory”



strings:

       $winver1 = "win 8.1"

       $winver2 = "win Server 2012 R2"

       $winver3 = "win Srv 2012"

       $winver4 = "win srv 2008 R2"

       $winver5 = "win srv 2008"

       $winver6 = "win vsta"

       $winver7 = "win srv 2003 R2"

       $winver8 = "win hm srv"

       $winver9 = "win Strg srv 2003"

       $winver10 = "win srv 2003"

       $winver11 = "win XP prof x64 edt"

       $winver12 = "win XP"

       $winver13 = "win 2000"



       $pdb1 = "D:\\Acms\\2\\docs\\Visual Studio 2013\\Projects\\DownloadExcute\\DownloadExcute\\Release\\DownExecute.pdb"

       $pdb2 = "d:\\acms\\2\\docs\\visual studio 2013\\projects\\downloadexcute\\downloadexcute\\downexecute\\json\\rapidjson\\writer.h"

       $pdb3 = ":\\acms\\2\\docs\\visual studio 2013\\projects\\downloadexcute\\downloadexcute\\downexecute\\json\\rapidjson\\internal/stack.h"

       $pdb4 = "\\downloadexcute\\downexecute\\"



       $magic1 = "<Win Get Version Info Name Error"

       $magic2 = "P@$sw0rd$nd"

       $magic3 = "$t@k0v2rF10w"

       $magic4 = "|*|123xXx(Mutex)xXx321|*|6-21-2014-03:06PM" wide



       $str1 = "Download Excute" ascii wide fullword

       $str2 = "EncryptorFunctionPointer %d"

       $str3 = "%s\\%s.lnk"

       $str4 = "Mac:%s-Cpu:%s-HD:%s"

       $str5 = "feed back responce of host"

       $str6 = "GET Token at host"

       $str7 = "dwn md5 err"



condition:

       all of ($winver*) or

       any of ($pdb*) or

       any of ($magic*) or

       2 of ($str*)

}

```

Network IDS

```
alert http any any -> any any (msg:"--[PwC CTD] -- Unclassified Middle Eastern Actor - DownExecute URI (/dw/gtk)";  flow:established,to_server; urilen:7; content:"/dw/gtk"; http_uri;  depth:7; content:"GET" ; http_method; content:!"User-Agent:"; http_header; content:!"Referer:"; http_header; reference:md5,4dd319a230ee3a0735a656231b4c9063; classtype:trojan-activity; metadata:tlp WHITE,author @ipsosCustodes; sid:99999901; rev:2015200401;)



alert http any any -> any any (msg:"--[PwC CTD] -- Unclassified Middle Eastern Actor - DownExecute URI (/dw/setup)";  flow:established,to_server; urilen:>8; content:"/dw/setup"; http_uri;  depth:9; content:"POST" ; http_method; reference:md5,4dd319a230ee3a0735a656231b4c9063; classtype:trojan-activity; metadata:tlp WHITE,author @ipsosCustodes; sid:99999902; rev:2015200401;)



alert http any any -> any any (msg:"--[PwC CTD] -- Unclassified Middle Eastern Actor - DownExecute Headers";  flow:established,to_server; urilen:>7; content:"Accept */*";  http_client_body; content:"Content-Type: multipart/form-data\; boundary=------------------------"; http_header; content: "ci_session="; http_cookie; depth:11; content: "POST"; http_method; content:!"Referer:"; http_header; content:!"User-Agent:"; http_header; reference:md5,4dd319a230ee3a0735a656231b4c9063; classtype:trojan-activity; metadata:tlp WHITE,author @ipsosCustodes; sid:99999903; rev:2015200401;)

```

(1] https://github.com/kbandla/APTnotes/blob/master/2012/Cyberattack_against_Israeli_and_Palestinian_targets.pdf

(2] https://www.fireeye.com/blog/threat-research/2013/08/operation-molerats-middle-east-cyber-attacks-using-poison-ivy.html

(3] https://malwr.com/analysis/N2I1YmExMjNkMmM3NGQwMThlNjg5YmI4OGY3Mjc3ZmI

(4] http://curl.haxx.se

(5] See CTO-TAP-20140328-01B or http://pwc.blogs.com/cyber_security_updates/2014/12/apt28-sofacy-so-funny.html for examples of this.

(6] http://pwc.blogs.com/cyber_security_updates/2015/04/attacks-against-israeli-palestinian-interests.html#_ftnref6

(7] https://www.fireeye.com/blog/threat-research/2013/08/operation-molerats-middle-east-cyber-attacks-using-poison-ivy.html