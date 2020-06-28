# vvv病毒真相

> 360互联网安全中心监控到，已经沉寂一段时间的CryptoLocker（文档加密敲诈者）类木马，本月初在国内又开始传播感染。本次传播的木马是之前CTB-Locker木马的一个变种，在加密文档之后，会在文档文件名之后加入“.vvv”，该病毒也因此被称为VVV病毒。

0x00 概述
=====

经分析，该木马的核心加密功能，是根据臭名昭著的TeslaCrypt（特斯拉加密者，与那个电动汽车厂商没有任何关系）的最新版本改写而来。TeslaCrypt从今年2月份正式“出道”以来，已经经过了8个版本的迭代。其中前4个版本加密后的文档都可以通过工具恢复，但从第5版开始则无法再恢复。且其第5版和第7版都曾在国内有过不同规模的爆发。

此次病毒事件主要的传播地区在日本，twitter上搜索“vvvウイルス”已经闹翻了天了：

![p1](http://drops.javaweb.org/uploads/images/1f2f0835f8e999f08d0af021427f3fc5c5ef25b0.jpg)

![p2](http://drops.javaweb.org/uploads/images/493cca71b46b1333017cdaa911b72653d1000ebf.jpg)

而我国出现该病毒完全属于“躺枪”。之所以说是“躺枪”，其实看下对方的勒索要求就很明显了。对方给出了四个所谓的“私人页面（PersonalPages）”要求你去访问并获取交易信息。但实际上这四个网站的域名分别为：

```
encpayment23.com
expay34.com
hsh73cu37n1.net
onion.to

```

这其中，只有最后一个onion.to可以访问，但这是一个“TorHiddenServicesGateway”，也就是说这是一个用洋葱路由的方式专门用来隐藏背后真实服务商的网站，并且还需要你安装一个所谓的“TorBrowser”进入洋葱网络才能正常浏览，于是：

![p3](http://drops.javaweb.org/uploads/images/f41f5e5edada5b0c765f4ee801a9eb536f11bdbd.jpg)

几个小时过去了……然后就没有然后了……也就是说即便我想要老老实实的交付赎赎金，也是一个不可能完成的任务。

0x01 传播
=====

不只是不能访问的页面，根据我们的监控数据看，也印证了该木马并非针对中国而来——刚刚进入12月的时候木马感染量有过一次小规模的上涨，而最近几天的传播量更是创了一个新高。但即便如此每天的木马活跃了也没有超过100个，所以说总体而言该木马在我国并没有真正意义上的“爆发”起来。

![p4](http://drops.javaweb.org/uploads/images/86fe8d67538d3f7d782704ad9c06275668d48f24.jpg)

而国内中招用户大部分也是通过比较传统的途径感染的木马——电子邮件：

![p5](http://drops.javaweb.org/uploads/images/6b42d2f527c4603dcf3029b9abec87feb92fedec.jpg)

邮件声称你有一笔未偿欠款，如果逾期不还的话，会产生7%的利息。而附件中所谓的“单据副本”其实就是这个木马。多么典型的诈骗内容——“您有欠款”、“您欠电话费了”、“您有未接收的快递”、“您有法院传票”……看来用这样的说辞并非国人的专利。但全篇的英语又有几个人会去仔细看完然后点开附件呢？这也就是在国内传播量很小的主要原因。

而实际上，该木马在国外的传播途径不仅仅是邮件传播，也包括一部分通过网页挂马（以去年最后的CVE-2014-6332和今年出镜率最高的CVE-2015-5122这两个漏洞的利用为主）来实现的木马传播，但由于经常访问的网站有很大不同，所以在国内因为网页挂马导致感染该木马的情况并不多见。

0x02 样本分析
=====

样本最开始的来源是一个伪装成发票单的js脚本

![p6](http://drops.javaweb.org/uploads/images/165715e68492f1474ea220a3f121e6f05f167460.jpg)

咋看内容是一堆乱码：

![p7](http://drops.javaweb.org/uploads/images/4bb573cea29af5549c317dc97ea4d5d4b0a3b369.jpg)

对其格式稍加调整之后，可以发现，其实就是一些字符的混淆，最后通过eval函数执行。

![p8](http://drops.javaweb.org/uploads/images/12374c4d73a28f7947c036e44db1a62cd6146f28.jpg)

直接将其eval的内容log输出，就能看到真正执行的代码了，功能比较简单，只是一个木马下载器

![p9](http://drops.javaweb.org/uploads/images/62c7ae8bd3538bb7be9af23a3534caf1d637c9cd.jpg)

样本本身带有一个简单的保护壳，启动后，会检测自身路径，如果不在%appdata%下，会将自身拷贝过去，并再次启动，之后删除之前的木马文件，达到隐藏自身的目的。

![p10](http://drops.javaweb.org/uploads/images/66df6cdda2de0e6256a9f0523056413721f09700.jpg)

木马在`%appdata%`下执行之后，会再次启动自身，解密隐藏的代码，注入到启动的这个子进程中：

![p11](http://drops.javaweb.org/uploads/images/b099ae3f6dcb2b30fcfd049ecf4093fb59c899b9.jpg)

木马使用这种方法，试图绕过传统特征码定位引擎的查杀，解码出来的程序是真正的木马工作部分：

![p12](http://drops.javaweb.org/uploads/images/50d4117f76bd1b8d9a2a19d6ec3549b1d005b98d.jpg)

木马的整体流程控制上，和之前的ctb-locker比较相似：

![p13](http://drops.javaweb.org/uploads/images/0aa0ced01057c00aaed375e85a7a9ee86496b5d4.jpg)

由于这个模块是通过注入方式执行的，启动后会通过GetProcAddress找到找到所需的系统API：

![p14](http://drops.javaweb.org/uploads/images/c714affee14b8402c272f5a1bccd0ae109b107ad.jpg)

木马在感染之前，会先将自身写入启动项，保证下次开机仍能够启动！

![p15](http://drops.javaweb.org/uploads/images/b30675d8820d8bf1b53298aec7f61ee6bcd37adb.jpg)

木马中配置等各类地址，都经过了重新编码，用来对抗分析：

![p16](http://drops.javaweb.org/uploads/images/12f865ebb4171ef32ebba0c7387373d2779e1831.jpg)

解码出的内容包括木马控制服务器，密钥交换时提及的信息结构：

```
0012DD0C   00CD1B18  ASCII "Sub=%s&key=%s&dh=%s&addr=%s&size=%lld&version=%s&OS=%ld&ID=%d&gate=%s&ip=%s&inst_id=%X%X%X%X%X%X%X%X"
0012DC98   01062040  ASCII "http://crown.essaudio.pl/media/misc.php"
0012DCF0   01061F38  ASCII "http://graysonacademy.com/media/misc.php"
0012DD04   01061E30  ASCII "http://grupograndes.com/media/misc.php"
0012DD18   01061D28  ASCII "http://grassitup.com/media/misc.php"

```

密钥生成方式和之前的CTB-Locker一致，都是通过ECDH生成，如果没有服务器上的私鈅目前无法获取到加密密钥。

![p17](http://drops.javaweb.org/uploads/images/76cdf8856e97be918cbd7aa821c3b038fbaf3af5.jpg)

![p18](http://drops.javaweb.org/uploads/images/5037781d0974d8fd539b3cfa981f67915f9c5776.jpg)

文件加密过程中，会排除掉带有.vvv扩展名的文件和带有recove的文件：

![p19](http://drops.javaweb.org/uploads/images/28d5897988e1b4ade33a857e4cdbf74d4ad893e0.jpg)

加密如下190种类型的文件：

```
|.r3d|.ptx|.pef|.srw|.x3f|.der|.cer|.crt|.pem|.odt|.ods|.odp|.odm
|.odc|.odb|.doc|.docx|.kdc|.mef|.mrwref|.nrw|.orf|.raw|.rwl|.rw2
|.mdf|.dbf|.psd|.pdd|.pdf|.eps|.jpg|.jpe|.dng|.3fr|.arw|.srf|.sr2
|.bay|.crw|.cr2|.dcr|.ai|.indd|.cdr|.erf|.bar|.hkx|.raf|.rofl|.dba
|.db0|.kdb|.mpqge|.vfs0|.mcmeta|.m2|.lrf|.vpp_pc|.ff|.cfr|.snx
|.lvl|.arch00|.ntl|.fsh|.itdb|.itl|.mddata|.sidd|.sidn|.bkf|.qic
|.bkp|.bc7|.bc6|.pkpass|.tax|.gdb|.qdf|.t12|.t13|.ibank|.sum|.sie
|.zip|.w3x|.rim|.psk|.tor|.vpk|.iwd|.kf|.mlx|.fpk|.dazip|.vtf
|.vcf|.esm|.blob|.dmp|.layout|.menu|.ncf|.sid|.sis|.ztmp|.vdf|.mov
|.fos|.sb|.itm|.wmo|.itm|.map|.wmo|.sb|.svg|.cas|.gho|.syncdb
|.mdbackup|.hkdb|.hplg|.hvpl|.icxs|.docm|.wps|.xls|.xlsx|.xlsm
|.xlsb|.xlk|.ppt|.pptx|.pptm|.mdb|.accdb|.pst|.dwg|.xf|.dxg|.wpd
|.rtf|.wb2|.pfx|.p12|.p7b|.p7c|.txt|.jpeg|.png|.rb|.css|.js|.flv
|.m3u|.py|.desc|.xxx|.wotreplay|wallet|.big|.pak|.rgss3a|.epk|.bik
|.slm|.lbf|.sav|.re4|.apk|.bsa|.ltx|.forge|.asset|.litemod|.iwi
|.das|.upk|.d3dbsp|.csv|.wmv|.avi|.wma|.m4a|.rar|.7z|.mp4|.sql|

```

在对文件加密完成之后，会对加密好的文件进行重命名，加入vvv扩展名：

![p20](http://drops.javaweb.org/uploads/images/bd7572d1e759917da64b251952507570957e77c4.jpg)

![p21](http://drops.javaweb.org/uploads/images/7b5a4973e9d167abdf620cbcf1019ef70d4f6eff.jpg)

和其它木马判断完整进程名不同，这个木马会判断系统是否运行有程序，带有askmg,rocex,egedi,sconfi,cmd这些关键词的程序，如果带有，就结束这些程序，实际上对应的是procexp.exe，taskmgr.exe之类的辅助分析工具等。

![p22](http://drops.javaweb.org/uploads/images/8f756d85328802bc79d3fda0231a3f561deb363f.jpg)

加密完成后，会在被加密文件夹下生成：

![p23](http://drops.javaweb.org/uploads/images/3f25959f773821b1bba074502a8281293ec099ff.jpg)

最后敲诈者展示这个勒索页面：

![p24](http://drops.javaweb.org/uploads/images/e5d30084b844e86a8cdfc1e1bf8f4b4045da7bf2.jpg)

0x03 最后
=====

由于一旦中招，机器中的所有资料全会被加密，并且完全无法恢复（如上所说，即便你愿意付赎金，也可能无处可付），所以对此类木马的警惕性依然是非常有必要的。用户在使用计算机的过程中，对重要文件，最好进行隔离备份，减小各类应病毒木马攻击，程序异常，硬件故障等造成的文件丢失损失。同时也要养成良好的习惯，不要随意打开陌生邮件的附件。安装具有文档防护功能的安全软件，对于安全软件已经提示风险的程序，不要继续执行。