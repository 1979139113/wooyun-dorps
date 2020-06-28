# 当失控的预装行为以非正当手段伸向行货机时_北京鼎开预装刷机数据统计apk（rom固化版）分析

作者：395B5B28E44BAAE80F68968A88ADC6CAD702851B

0x00 背景
-------

* * *

本报告仅代表个人观点，与所在公司、所属团队、所在职务无关。因考虑人身安全，本节选由wooyun知识库代为匿名发布；全文待咨询相关人士后再决定是否公布。

2013年8月，在一大型电商网购联想A820t（属移动定制机型号）行货后，发现预装应用出现了异常：按道理，该机器本应有可卸载的移动预装应用，但实际上并没有，而多出了许多无法卸载的其它应用。经过技术分析，发现和鼎开存在关联。在接下来的数周内，连续或零散进行了实体店和网络调查，并形成了约50页报告并持续反馈给CNVD。

2013年9月后，该事件连同报告移交由CNCERT和CNVD处置；2014-2-14，ANVA进行初步处置披露（[http://www.anva.org.cn/webInfo/show/1181](http://www.anva.org.cn/webInfo/show/1181)）；2014-3-15，央视315晚会进行了部分曝光。本文为本报告的技术部分和普通用户处理方法节选。

本报告致谢如下人员：CNCERT、CNVD 相关工作人员；dex2jar、apktool、jdgui作者；大型城市公共交通系统（地铁、巴士、县际班车等）；手机实体店员工；其它在网络上进行相关主题讨论和问题咨询的网友。

本报告不希望成为新一轮口水战的起点，更希望成为每个参与互联网事业的人员反思自身的一个驿站——在这场抢占移动互联网用户和入口的大战中，每个人都有罪过。

0x01 应用名称、版本、权限演化历史、渠道区分
------------------------

* * *

com.android.tunkoo.scan一般被放置于/system/app/目录下，故而这类被恶意刷机的的行货无法对其卸载。其名称具有伪装性，目前已经发现两个。

| 应用名称 | 版本号 | 文件sha1值 |
| :-: | :-: | :-: |
| SystemScan | 1.1 | 1485f3944756655ff23361ee7f4ab0ace9f110c1 |
| SystemServer | 1.2 | 43cc56679117fcc58d1b1f2515ff163f26301026 |

表：com.android.tunkoo.scan版本历史

这两个版本的apk均以“DINGKAI.RSA”签名， openssl打印信息均一样。以签名颁发者“beijing dingkai”为关键词搜索，初步认为是北京鼎开联合信息技术有限公司。

两者所申请的权限如下，其中 “SystemServer”作为高版本，比“SystemScan”多申请了联系人和短信读取权限。从Android 3.1开始，应用要接收Android广播（比如开机自动启动），必须要让用户至少运行一次，并且没有在设置中“强制停止”；但com.android.tunkoo.scan被放置于/system/app，因此即使没有入口让用户启动过，也可以无视该规定，作为系统应用自动开机运行。

![enter image description here](http://drops.javaweb.org/uploads/images/0cc801631ca5152851969996b97ce565746ce71f.jpg)

表：com.android.tunkoo.scan所申请的权限列表和演化

为了区分不同的刷机渠道，每个com.android.tunkoo.scan中的AndroidManifest.xml中，使用友盟SDK的UMENG_CHANNEL字段定义了刷机ROM标识字符串，同时还有一个SHOP_ID字段标识推广渠道id。

![enter image description here](http://drops.javaweb.org/uploads/images/a6d271782e844191d2cb0cf2b12025e8c176615c.jpg)

图：两个样本的AndroidManifest.xml值

0x02 上报分析
---------

* * *

### 概述

com.android.tunkoo.scan除了使用友盟SDK作简单渠道统计外，还有一套自己的上报机制，主要目的是持续监测用户行为并进行上报。为了逃避快速沙箱检测，com.android.tunkoo.scan组合使用定时器（Alarm）、广播（Broadcast）和服务（Service）机制，对监测行为进行了延后或轮训；为了阻止手工快速分析，这些上报信息在手机存储和发送到服务器中，均运用了不同的对称加密方法和密钥。

通过对发送加密前的日志进行分析，相关的上报内容被分成三大类发往不同的统计接口。

第一类为手机基础信息上报，上报时机为插入SIM卡且每次连接网络。上报内容有：

```
A. 手机硬件信息：手机型号、手机WIFI MAC地址
B. 运营商识别信息：IMEI、IMSI（若有插入SIM卡）、手机号码
C. 手机软件基础信息：安卓版本号、root情况
D. 手机软件列表信息：已安装应用列表
E. 鼎开刷机标识：ROMID、shop_id、device_IOS

```

第二类为用户使用行为信息上报，上报时机为每次连接网络、或手机开机两小时后每两小时。上报内容有：

```
A. 运营商识别信息：运营商信息
B. 手机上网运行信息：上网类型、历史上网ip 
C. 手机软件运行信息：正在运行的应用列表、曾经连接过网络的应用和流量
D. 手机软件其它信息：安装卸载应用历史
E. 定位信息：通过网络定位获取用户的地理坐标

```

第三类仅存在于“SystemServer”，为特定信息挖掘上报，上报时机为每次连接网络、或手机开机两小时后每两小时。上报内容有：

```
A. 15个短信关键词命中计数
B. 11个联系人关键词命中计数
C. 手机QQ应用信息：使用该手机登录过的QQ号、所有QQ联系人的头像图片文件名、所有QQ群的头像图片文件名

```

以下详细讲解其上报时机、格式和运作机制。

### 手机基础信息上报

这是预装统计最基础的内容上报，依据各种条件判断，上报到以下url中的其中一个：

```
http://x.xxxx.com/api.aspx?t=1 （“SystemScan”、“SystemServer”）
http://xxxxxxx.xxxx.com/?p=daoda3 （“SystemServer”）
http://x.xxxx.com/api.aspx?t=6 （“SystemScan”）
http://xxxx.xxxx.com/rec.aspx?t=0&iszip=false （SystemScan，SystemServer）

```

一条完整的上报统计格式如下：

```
ROMID={ROMID}||device_IOS={device_IOS}||device_IMEI={device_IMEI}||MAC={MAC}||os_Version={ os_Version }||shop_id={ shop_id }||device_IMSI={device_IMSI}||device_Model={device_Model}||device_Number={device_Number}||device_app_list={device_app_list}||nettype={nettype}||isrooted={isrooted}||postdataindex={postdataindex}||allsendindex={ allsendindex}||TIME1={TIME1}||TIME2={TIME2}||TIME3={TIME3}||

```

![enter image description here](http://drops.javaweb.org/uploads/images/30d0b237450e015e44724dd60760e61d1462abc2.jpg)

表：手机基础信息上报字段名含义和示例值

主要的上报逻辑有：

```
1. 在开机后，周期性检测手机是否插入SIM卡（TelephonyManager.getSimState() != SIM_STATE_ABSENT）。如果条件不成立，则潜伏不运行（即使存在wifi连接）。
2. 插入SIM卡后，如果发现有网络连接（GPRS/WiFi等），并且距离上一次发送已经超过10分钟，则在连接后发送一个统计心跳维持包。该心跳包仅包含ROMID、IMEI、MAC三个字段信息，其余字段均赋值为null。一个日志信息如下（敏感信息已用示例值替代）：
3. 如果期间有安装或者卸载过应用，会在下一次连接上网络后发送完整统计包，此时心跳包内所有null值均被替换为完整信息。此处上报的手机个人敏感信息最为详细。一个日志信息如下（敏感信息已用示例值替代）：
4. 如果周期内统计发送失败，会延长时间重试3次，延长时间分别为2分钟、20分钟、2小时。

```

### 用户使用行为信息上报

这是持续跟踪用户使用行为并进行上报的机制。依据不同版本，上报到以下url中的其中一个：

```
http://xxxxxxx.xxxx.com/?p=caiji3（“SystemServer”）
http://x.xxxx.com/api.aspx?t=3（“SystemScan”）

```

当用户点击连接网络（GPRS/WiFi）、或手机开机两小时后每两小时，都会进行一次这样的上报。如果上报失败，会将待上报的内容，以数字编号文件形式存入“`/data/data/com.android.tunkoo.scan/files/s`”目录，并在后续逻辑中按文件顺序重新上报。故如果用户频繁开关机、或者长时间待机，会在此目录积累大量的待上报内容。

![enter image description here](http://drops.javaweb.org/uploads/images/75ddadadf03f2405725ff35f75c214d97fb82692.jpg)

图： 用户使用行为上报信息缓存文件

一条上报统计示例如下，根据上报时间时采集到不同的用户行为，会有所变化：

```
TIME=2013-09-20 18:27:05||IOS=com.android.tunkoo.scan_3_8c81a411||IMEI=477777777777777||IMSI=83333333333333333333||SHOPID=8798||ROMID=8798_201308111529_471.0||MAC=00:0d:08:07:c6:a5||MODEL=Lenovo A820t||IP=172.0.0.1||LA=0||LO=0||APPUSE=com.speedsoftware.rootexplorer,d2481498,2013-09-20 18:25:53,2013-09-20 18:26:26&&||APPFLOW=||NETTYPE=,WIFI,net,2013-09-20 18:25:51,2013-09-20 18:26:26&&||APPINSTALL=||IP=172.0.0.1,2013-09-20 18:25:51&&||

```

![enter image description here](http://drops.javaweb.org/uploads/images/45cc87e0e61dc7db7417c78772493b4f39f7dd8d.jpg)

表：用户使用行为上报字段名含义和示例值

由于每种用户行为的频率不一样，故部分参数的行为采集频率（定时器设置间隔）、存储位置并不一致。

![enter image description here](http://drops.javaweb.org/uploads/images/221a7185cfb2cf76055600c33b2df5c02a935bec.jpg)

表：用户使用行为上报部分字段名的采集频率、目的和存放位置

### 特定信息挖掘上报（仅存在于SystemServer）

这是“SystemServer”新增的信息上报类别，为此该版本新增请求权限“android.permission.READ_CONTACTS”（读取联系人）和“android.permission.READ_SMS”（读取短信）。当用户点击连接网络（GPRS/WiFi）、或手机开机两小时后每两小时，都会进行一次上报检测，若检测到两次上报之间大于5小时，则向http://xxxxxxx.xxxx.com/?p=xincaiji3 进行一次上报。

一条上报统计示例如下：

```
IMEI=477777777777777||MAC=00:0d:08:07:c6:a5||MODEL=Lenovo A820t||Contact=1,0,0,0,0,0,0,0,0,0,0,||MSGSend=0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,||MSGRec=0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,||NumberFoldList=555555,||_HDList=AEEE,BEEE,||_THDList=KEEE,LEEE,

```

![enter image description here](http://drops.javaweb.org/uploads/images/aa124e83d7d1abdccfc4a4a7493610edf1d95dde.jpg)

表：特定信息挖掘上报字段名含义和示例值

该上报有两个关键要点：

1.  该上报涉及读取手机联系人和短信这两个强隐私信息，但只是进行关键词计数上报，而没有上报具体内容。
    
    其中对联系人列表进行扫描时，使用11个关键词进行包含匹配测试并计数：“老婆、宝贝、孩子、女儿、儿子、老公、老爸、老妈、爸爸、妈妈、老师”。 比如说，如果Contact上报的内容为“1,0,0,0,0,0,0,0,0,0,0,”，则表示所有联系人中，有1个号码的联系人名称包含“老婆”这个词，其它关键词均没有命中。 而对发送短信和接收短信的扫描中，除了上面11个联系人关键词之外，还新增了4个关键词专门匹配短信内容：“先生、女士、消费、还款”。 比如说，如果MSGRec上报的内容为“0,0,0,0,0,0,0,0,0,0,0,3,0,0,0,”，则表示自上次上报新接收的短信中，有3条短信内容包含“先生”这个词，其它关键词均没有命中。
    
2.  该上报非常关注手机QQ软件在sdcard的目录信息。
    

由于手机QQ运行时的文件较多，难以放到/data对应的应用目录下，故这些文件都放到了sdcard目录下。但sdcard目录允许被任何应用读取，故导致可被“SystemServer”扫描相关目录并上报。猜测这个目的是为了尝试收集该手机用户的QQ使用习惯和用户喜好。

![enter image description here](http://drops.javaweb.org/uploads/images/d87a79dc4bf04915414931cb8f9f5cc68b042e37.jpg)

图：“SystemServer”扫描手机QQ目录获取所有登录过的QQ号

![enter image description here](http://drops.javaweb.org/uploads/images/e656e2e171abf0883a17228398ff03ca4eba42dc.jpg)

图：“SystemServer”扫描手机QQ的联系人头像缓存文件名列表

![enter image description here](http://drops.javaweb.org/uploads/images/75b4b92d261615f3ee082c75087b2ac373d24376.jpg)

图：“SystemServer”扫描手机QQ的群头像缓存文件名列表

0x03 危害和查杀情况
------------

* * *

com.android.tunkoo.scan的危害主要在于持续监测和上报，泄露了用户各种隐私信息和行为。在这个过程中由于大量使用定时器扫描和进行网络连接，故会持续消耗电量和网络流量；特别的，如果没有申请运营商流量套餐又误用GPRS上网，将会出现扣费问题。另外在某些情况下，可能会由于大量的扫描，引起系统缓慢和失去响应。

![enter image description here](http://drops.javaweb.org/uploads/images/6d24db8cdf91bd98a6fc1a2f58d6db1e0e836198.jpg)

图：百度知道中有关“SystemScan”流量问题的提问

对com.android.tunkoo.scan的查杀情况，则进行了两次验证。第二次验证为2013-9，在测试机上进行了“SystemServer”（原版apk）查杀测试，仍然无法检出。截图从左到右：腾讯手机管家（病毒库20130922B）、金山毒霸（库文件版本2013.9.12.1947 + 启发扫描）、LBE（病毒库20130918.f）、360（病毒库20130922B）。此处对手机毒霸有个疑问，那就是无论扫描结果中的“耗电软件”（后台自启动应用列表）、还是应用行为管理，均找不到“SystemServer”应用可供管理或禁用。

![enter image description here](http://drops.javaweb.org/uploads/images/26a7d05de2ca45d083b6b4295b64d3f35ff84c97.jpg)

图：2013-9在测试机上对“SystemServer”的查杀情况

0x04 鼎开的行货刷机手段
--------------

* * *

鼎开的恶意刷机，是在/system/vendor/operator/app/内的应用全部删除（此目录下的定制应用可被用户卸载），再在/system/app内放入无法卸载的应用。

![enter image description here](http://drops.javaweb.org/uploads/images/0871ee3fe3715ecb7222e56d51d62cfc5388758d.jpg)

图：联想A820t恶意刷机前后/system/vendor/operator/app/目录对比。左：S115厂商包；右：问题手机

![enter image description here](http://drops.javaweb.org/uploads/images/89f701920dc934f62a0024a56f0dbe93ad8d664f.jpg)

图：高端手机预装联盟服务流程

在新浪微博搜索中，以“鼎开 刷机”进行搜索的结果显示，鼎开似乎成为了行货刷机的公开幕后渠道，日刷机量至少5万，其最初的主要目标是三星行货（但后来也许也兼营其它品牌，比如联想），并且可能有大型互联网公司的参与。而这些信息，至少从去年底就开始存在，但并没有引起很大的反响。

![enter image description here](http://drops.javaweb.org/uploads/images/2583df4f183ca42368fc95878bf8da53ae3e0dd0.jpg)

![enter image description here](http://drops.javaweb.org/uploads/images/225524bb49d30c59640916bfc042d4f02ad3bf71.jpg)

![enter image description here](http://drops.javaweb.org/uploads/images/370a27e8a8335f4a52f7048164c00f9f8f05acb0.jpg)

图：微博上搜索“鼎开 刷机”信息

根据百度搜索的结果，鼎开至少从2012年开始，在智联招聘、赶集网、58同城等网站发布不同地区的招聘信息，比如广东深圳的rom编辑人员（2013-08-22），还有河南郑州（2012-05-29）、河北石家庄（2013-08-22）、广东深圳（2013-08-05）等地的安卓系统刷机操作专员，其中明显提到，刷机的操作流程需要有“拆包-封包”等破坏和伪造行货包装完整性的行为。

![enter image description here](http://drops.javaweb.org/uploads/images/5f93d70f922b29cf219bbbb54dc3ce4730f01a33.jpg)

图：鼎开在石家庄招聘安卓系统刷机操作专员，其中提到的刷机流程

根据游戏葡萄的文章，除了鼎开之外还有其它的刷机公司，分布在不同的品牌和价位；至于刷机方法，已有一整套成熟的自动化工具。

![enter image description here](http://drops.javaweb.org/uploads/images/561384e3649c4809471b0f8ec94191f361d89923.jpg)

图：刷机公司的工具

综合以上信息，可以看到失控的预装行为，早已经以非正当的方式，伸向了已渐趋理性、并且消费者一贯信赖的行货机。而从鼎开的合作模式和刷机操作来看，可以肯定渠道出现问题的可能性极大——有某些渠道商非法联合刷机商，私自拆开行货包装重刷一遍，刷完后用行货的封条缝合，外观上没看出动过，以此欺骗下游渠道商和最终用户。这样的行为已经明显损害了购买行货的消费者权益。

0x05 基于网络和实体店的感染情况与分布调查
-----------------------

* * *

为了摸清楚受鼎开刷机所影响的范围和分布情况，从2013年8月开始，针对城乡结合部区域进行了持续一周的实体店调查，在后续的周末中，也断续进行零散调查；同时在这个过程中，针对性进行网络搜索。

调查的结果如下：

```
1. 无论线上电商购买还是线下实体店购买，都有可能购买到被鼎开或其他刷机商恶意刷机的联想或三星行货。
2. 网络购物中，几乎所有知名电商，都或多或少存在以“行货正品”的形式销售被刷机手机。
3. 实体店调查中，有销售鼎开刷机的手机店，其区域分布广泛，且发现存在其它形式（非鼎开）的刷机。
4. 每部被恶意刷机的预装应用列表，随刷机时间、手机型号和推广渠道的不同而不同。这其中确实有大型互联网公司参与其中。
5. 如果一部运营商定制型号行货被刷机，不明就里的消费者会更大概率怪罪运营商、手机制造商，而不是真正的刷机商。

```

0x06 普通用户的解决方法
--------------

* * *

为了让用户确定自己是否遭受鼎开刷机，作者开发了一个“应用安装信息报告”（com.example.forensics.systemapp）。该应用可安装在Android 2.2及以上机器，在进入应用后点击“收集…”按钮并稍候。如果结果中出现“[重要提示！]发现疑似鼎开刷机的证据”、并且下面列出的安装包中出现“com.android.tunkoo.scan”或者“com.easyandroid.widgets”等（暂不判断是否为系统应用），那么恭喜你，很大概率中招了……

![enter image description here](http://drops.javaweb.org/uploads/images/8647a17921aab22a10d70cd4ecf60165f54e0637.jpg)

图：“应用安装信息报告”运行结果

![enter image description here](http://drops.javaweb.org/uploads/images/0ce857712093be2e45be693fc432c28eba61cb15.jpg)

表：“应用安装信息报告”的结果文件夹含义

中招的解决之道，简单的方法是在设置中停用“SystemScan”或者“SystemServer”；但唯一最彻底的方法是重新刷机，如果对刷机不熟悉，最好带上发票，直接找当地的售后服务网点进行。即便自己懂得刷机，也需要做好个人资料（比如手机通讯录、短信等）的备份，同时也要承担操作失误而变砖、或者因为root/刷机计数增加等而失去保修的风险。另外：

```
1. 如果手机是MTK芯片的话，会容易出现因为MTK手机芯片特性而导致刷机后丢失IMEI号、手机SN号、wifi mac地址、蓝牙mac地址等这些手机唯一特征值的问题，从而引发系统问题，故建议使用纸笔或电子文件备份好以上手机唯一特征值。
2. 三星安卓手机会检查手机是否在官方行货状态，包括是否没有 root、是否为官方 recovery 等。一旦发现不符合条件，系统状态将会从“官方”变成“定制”状态。这种改变的结果，会导致失去保修资格。另外， 三星安卓手机存在一个刷机计数器（ODIN 刷机界面中的“CUSTOM BINARY DOWNLOAD”）， 当用户自行刷机时，计数器加 1并在开机界面显示黄色感叹号告警，这样的结果同样是失去质保。

```

在刷机之外，消费者也可以积极做好和最终购买渠道的售后沟通和投诉，如有必要，也可考虑向手机制造商举报这个购买渠道存在问题。

对于还没有购买手机的用户而言，如果无法承受网购所带来的风险，那么去实体店购买的时候，一定要到当地知名的正规连锁店购买，并且仔细考察店内各种展示机里面的应用列表，如发现存在有疑问的应用，应及时咨询；在购买时，仔细检验包装，开机后检查应用列表是否有问题，并提防基于门店的渠道安装。当然比起网购，实体店由于种种原因，价格会比较贵——尤其是一款有一段上市时间的手机，此时网上的价格会比实体零售店的价格差距，很有可能会拉得非常大。

如果对自己网购能力有信任，那么请务必不要贪图小便宜，同时认真看网购评价，尤其是中差评，有可能会看出一些端倪。

0x07 附录：关于利用刷机方式植入“手机预装马” 进行用户信息窃取的情况通报（2014-2-14）（节选）
------------------------------------------------------

* * *

通过对该手机预装马进行持续监测，截至2014年1月CNCERT发现全国范围内感染该手机预装马的用户数达2167148个。感染用户按照区域分布和运营商分布情况分别如图2和图3所示。

![enter image description here](http://drops.javaweb.org/uploads/images/de484d56bfe2df644f7c22bd961d6ecc02b9561c.jpg)

图2 感染“手机预装马”用户按照区域分布情况

![enter image description here](http://drops.javaweb.org/uploads/images/40381e11802b6144b399a71713baa15aec124bda.jpg)

图3 感染“手机预装马”用户按照运营商分布情况

目前，CNCERT已协调域名注册商中国万网对“手机预装马”所用于采集用户信息的3个服务器域名x.xxxx.com，xxxxxxxxxx.xxxx.com，xxxxxx.xxxx.com进行停止解析处理，并协调电信增值企业网宿科技对服务器所使用的IP地址进行停止接入处理。

由于“手机预装马”在运行时具有明显特征，安卓手机用户可通过检查当前手机进程信息中是否含有名为“com.android.tunkoo.scan”的“SystemScan”进程，如果含有该进程则说明手机中已被植入“手机预装马”，用户须及时卸载“SystemScan”应用。

![enter image description here](http://drops.javaweb.org/uploads/images/300f1aef0a76ce8312cc1b206233ac28fdd5e681.jpg)

图4 “手机预装马”所运行的恶意程序信息

同时用户可以通过由CNCERT负责日常运营的中国反网络病毒联盟ANVA（www.anva.org.cn）网站或者邮箱msample@cert.org.cn进行举报，[[email protected]](http://drops.com:8000/cdn-cgi/l/email-protection)信息，以便CNCERT开展处置工作。

检测APK：[com.example.forensics.systemapp.zip](http://static.wooyun.org/20141017/2014101719404351418.zip)