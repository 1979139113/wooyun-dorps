# windows安全日志分析之logparser篇

0x01 前言
=====

工作过程中，尤其是应急的时候，碰到客户windows域控被入侵的相关安全事件时，往往会需要分析windows安全日志，此类日志往往非常的大；此时，高效分析windows安全日志，提取出我们想要的有用信息，就显得尤为关键，这里给大家推荐一款我常用的windows日志分析工具，logparser，目前版本是2.2。

0x02 logparser使用介绍
=====

首先，让我们来看一下Logparser架构图，熟悉这张图，对于我们理解和使用Logparser是大有裨益的

![enter image description here](http://drops.javaweb.org/uploads/images/61733a50a3222c24dbc23d6defedf9dbdcfb494b.jpg)

简而言之就是我们的输入源（多种格式的日志源）经过 SQL语句（有SQL引擎处理）处理后，可以输出我们想要的格式。

**1、输入源**

从这里可以看出它的基本处理逻辑，首先是输入源是某一种固定的格式，比如EVT（事件），Registry（注册表）等，对于每一种输入源，它所涵盖的字段值是固定的，可以使用logparser –h –i:EVT查出（这里以EVT为例）：

![enter image description here](http://drops.javaweb.org/uploads/images/e29947582559bbf7da1b1bc6de5e349fa0ac25e0.jpg)

这里是一些可选参数，在进行查询的时候，可对查询结果进行控制，不过我们需要重点关注的是某一类日志结构里含有的字段值（在SQL查询中匹配特定的段）：

![enter image description here](http://drops.javaweb.org/uploads/images/388cc1605d8ccedaceaec53b7e400b58925ef14a.jpg)

对于每一类字段值的详细意义，我们可以参照logparser的自带文档的参考部分，这里以EVT（事件）为例：

![enter image description here](http://drops.javaweb.org/uploads/images/9a6b495039b5f30213b65de37aadf2ffe5c2372a.jpg)

**2、输出源**输出可以是多种格式，比如文本（CSV等）或者写入数据库，形成图表，根据自己的需求，形成自定的文件（使用TPL）等，比较自由

0x03 基本查询结构
=====

了解了输入和输出源，我们来看一则基本的查询结构

```
Logparser.exe –i:EVT –o:DATAGRID “SELECT * FROM E:\logparser\xx.evtx”

```

这是一则基本的查询，输入格式是EVT（事件），输出格式是DATAGRID（网格），然后是SQL语句，查询E:\logparser\xx.evtx的所有字段，结果呈现为网格的形式:

![enter image description here](http://drops.javaweb.org/uploads/images/1349b3e6768fbf8bfcc0fc0dfb16f9dfb78f16ea.jpg)

看到这里，想必你已经明白了，对于windows的安全日志分析，我们只需要取出关键进行判断或者比对，就可以从庞大的windows安全日志中提取出我们想要的信息。

0x04 windows安全日志分析
=====

对于windows安全日志分析，我们可以根据自己的分析需要，取出自己关心的值，然后进行统计、匹配、比对，以此有效获取信息，这里通过windows安全日志的EVENT ID迅速取出我们关心的信息，不同的EVENT ID代表了不同的意义，这些我们可以在网上很容易查到，这里列举一些我们平常会用到的

<table style="box-sizing: border-box; border-collapse: separate; border-spacing: 0px; background-color: rgb(255, 255, 255); border-top: 1px solid rgb(224, 224, 224); border-left: 1px solid rgb(224, 224, 224); font-family: Helvetica, Arial, &quot;Hiragino Sans GB&quot;, sans-serif; letter-spacing: normal; orphans: 2; text-indent: 0px; text-transform: none; widows: 2; word-spacing: 0px; -webkit-text-stroke-width: 0px; text-decoration-style: initial; text-decoration-color: initial;"><tbody style="box-sizing: border-box;"><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">Event</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">任务类别</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">解释</td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">ID</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);"></td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);"></td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">540</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">登陆/注销</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">Event 540 gets logged when a user elsewhere on the network connects to a resource (e.g. shared folder) provided by the Server service on this computer. The Logon Type will always be 3 or 8, both of which indicate a network logon.</td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">538</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">登陆/注销</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">Ostensibly, event 538 is logged whenever a user logs off, whether from a network connection, interactive logon, or other&nbsp;logon type<br style="box-sizing: border-box;">For network connections (such as to a file server), it will appear that users log on and off many times a day. This phenomenon is caused by the way the Server service terminates idle connections.&nbsp;</td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">528</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);"></td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">![enter image description here][6]</td></tr><tr style="box-sizing: border-box;"><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">675</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">账户登陆</td><td style="box-sizing: border-box; padding: 0px 3px; border-bottom: 1px solid rgb(224, 224, 224); border-right: 1px solid rgb(224, 224, 224);">When a user attempts to log on at a workstation and uses a valid domain account name but enters a bad password, the DC records event ID 675 (pre-authentication failed) with Failure Code&nbsp;24. By reviewing each of your DC Security logs for this event and failure code, you can track every domain logon attempt that failed as a result of a bad password. In addition to providing the username and domain name, the event provides the IP address of the system from which the logon attempt originated.</td></tr></tbody></table>

有了这些我们就可以对windows日志进行分析了 比如我们分析域控日志的时候，想要查询账户登陆过程中，用户正确，密码错误的情况，我们需要统计出源IP，时间，用户名时，我们可以这么写（当然也可以结合一些统计函数，分组统计等等）：

```
LogParser.exe -i:EVT "SELECT TimeGenerated,EXTRACT\_TOKEN(Strings,0,'|') AS USERNAME,EXTRACT\_TOKEN(Strings,2,'|') AS SERVICE\_NAME,EXTRACT\_TOKEN(Strings,5,'|') AS Client_IP FROM 'e:\logparser\xx.evtx' WHERE EventID=675"

```

查询结果如下：

![enter image description here](http://drops.javaweb.org/uploads/images/bbf14bf13b61e801fcf45b8c86f2e42e6f81ec3c.jpg)

如果需要对于特定IP进行统计，我们可以这么写（默认是NAT输出）：

```
LogParser.exe -i:EVT "SELECT TimeGenerated,EXTRACT\_TOKEN(Strings,0,'|') AS USERNAME,EXTRACT\_TOKEN(Strings,2,'|') AS SERVICE\_NAME,EXTRACT\_TOKEN(Strings,5,'|') AS Client\_IP FROM 'e:\logparser\xx.evtx' WHERE EventID=675 AND EXTRACT\_TOKEN(Strings,5,'|')='x.x.x.x'"

```

或者将查询保存为sql的格式：

```
SELECT TimeGenerated,EXTRACT\_TOKEN(Strings,0,'|') AS UserName,EXTRACT\_TOKEN(Strings,1,'|') AS Domain ,EXTRACT\_TOKEN(Strings,13,'|') AS SouceIP,EXTRACT\_TOKEN(Strings,14,'|') AS SourcePort FROM 'E:\logparser\xx.evtx' WHERE EXTRACT_TOKEN(Strings,13,'|') ='%ip%'

```

然后在使用的时候进行调用

```
logparser.exe file:e:\logparser\ipCheck.sql?ip=x.x.x.x –i:EVT –o:NAT

```

查询结果为：

![enter image description here](http://drops.javaweb.org/uploads/images/2900ea110066b12151fb56dce9abfe3a1907edf1.jpg)

怎么样？是不是一目了然呢？根据特定登陆事件，直接定位到异常IP，异常时间段内的连接情况。

同样我们也可以选择其他输出格式，对日志分析和统计。上述所有操作都是在命令行下完成的，对于喜欢图形界面的朋友，We also have choices！这里我们可以选择使用LogParser Lizard。 对于GUI环境的Log Parser Lizard，其特点是比较易于使用，甚至不需要记忆繁琐的命令，只需要做好设置，写好基本的SQL语句，就可以直观的得到结果，这里给大家简单展示一下 首先选取查询类型

![enter image description here](http://drops.javaweb.org/uploads/images/2e6a10cc5074781c4523e9d89db815173d282bda.jpg)

这里我们选择windows event log，然后输入刚才的查询语句： 比如：

```
SELECT TimeGenerated,EXTRACT\_TOKEN(Strings,0,'|') AS USERNAME,EXTRACT\_TOKEN(Strings,2,'|') AS SERVICE\_NAME,EXTRACT\_TOKEN(Strings,5,'|') AS Client\_IP FROM 'e:\logparser\xx.evtx' WHERE EventID=675 AND EXTRACT\_TOKEN(Strings,5,'|')='x.x.x.x'

```

得到的查询结果为（并且这里我们可以有多种查询格式）：

![enter image description here](http://drops.javaweb.org/uploads/images/dad106add7f6a84bb39e92840040c4c33d9ab7e5.jpg)

具体其他功能，大家可以去尝试一下~

0x05 总结
=====

这里简单和大家介绍了在windows安全日志分析方面logparser的一些使用样例，logparser的功能很强大，可进行多种日志的分析，结合商业版的Logparser Lizard，你可以定制出很多漂亮的报表展现，图形统计等，至于其他的功能，留给大家去探索吧~~~~~