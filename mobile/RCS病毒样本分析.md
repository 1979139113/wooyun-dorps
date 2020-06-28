# RCS病毒样本分析

0x00 简介
=====

近日总部位于意大利的监控软件开发公司HackingTeam被黑，415GB文件被泄露，其监控软件源码和漏洞源码流出，对网络安全造成了巨大冲击。猎豹移动专门对HackingTeam开发的适用于安卓系统的监控软件RCS系统进行了分析，并对肯能出现的利用该系统源码的病毒针对性的展开查杀，目前已经拦截了数十个利用该技术的变种病毒，这些病毒在隐蔽性，危害性上都非常的强大，已经对用户的隐私，数据安全构成了非常严重的威胁。

0x01 利用RCS技术的病毒分析：
=====

目前出现的病毒，很大程度上还只是在RCS系统源码上做了部分更改，有些变种已经开始采用一些较不常见的手段规避杀软的查杀，下面就以一个比较典型的病毒变种为例，来看看黑客们是怎么利用RCS技术攻击我们的。

0x02 目录结构
=====

![enter image description here](http://drops.javaweb.org/uploads/images/956ba680a8c125862089ab4ad4307ea5b7b7c3fe.jpg)

Librunner.so内容如下：可以看出只定义了一个函数invokeRun，做了打印输出和utf的编码解码函数的调用.。

![enter image description here](http://drops.javaweb.org/uploads/images/ffe82ca41e00dac33b32b9c8c117cdda0a3ca1dc.jpg)

![enter image description here](http://drops.javaweb.org/uploads/images/caa0b9167663d995630b851529c4b0509b4f279f.jpg)

0x03 使用的主要加密文件
=====

![enter image description here](http://drops.javaweb.org/uploads/images/f8ef07556ff30c92a0efd080b2fa5148d826a5a8.jpg)

Messages.bin文件使用AES对称加密，里面包含了一些敏感的可被作为特征的字符串和api关键字。文件内容是使用键值对的方式存储的，在使用的时候通过键值（如：13.0）取出对应的字符串（对应13.0的为Content://mms/sent），部分截图如下：下图中并不是文件原本的存储方式，文件中是使用“=”做的分割，这里为了让大家看清楚经过了作者的处理，其中key为键，value为值.

![enter image description here](http://drops.javaweb.org/uploads/images/b5d5ff1106bcb63756b56ac6f15a27afe503c18b.jpg)

上图展示了masseges.bin文件的一部分内容，如想获取全部内容请自行解密文件

0x04 程序结构
=====

下图中列出了病毒样本的主要程序结构

![enter image description here](http://drops.javaweb.org/uploads/images/a8fc6d104847dd3e3bf11bd505de0b264bd705c3.jpg)

0x05 AndroidManifest.xml文件结构
=====

应用权限对比：

![enter image description here](http://drops.javaweb.org/uploads/images/1b02cb1b0a531e5d6fd699001f66a5cb4227d534.jpg)

通过两图对比，可以看出病毒样本比RCS源码申请了更多的权限。

组件对比：

![enter image description here](http://drops.javaweb.org/uploads/images/363fe3465fc2244e61ff5281884bf174ef1d7d02.jpg)

从组件对比可以看出，RCS源码和病毒样本注册的广播接收器receiver和服务service名称完全相同，只是在监听的消息上存在微小差异。

0x06 病毒行为
=====

**病毒逻辑**

病毒逻辑主要分为两部分，一部分使用注册的广播监听器，监听特定的广播，手机如短信，来电等信息，将收集到的广播保存在特定的类中共其他类调用。另一部分通过名为core的主线程，进行手机状态（是否root，sd卡上是否存在相关目录并创建文件等）判断，获取硬件信息，用户位置，发送短信，链接远程服务器上传收集到的信息或更新病毒代码等。

**广播监听器部分**

病毒被安装后，会注册广播接收器监听手机广播（短信广播，来电广播，充电广播，屏幕解锁广播等），并通过服务获取用户的隐私信息，包括拦截短信，监听电话等。 这里以拦截短信为例进行说明：

注册的广播接收器，通过拦截android.provider.Telephony.SMS_RECEIVED系统广播，在用户收到短信时使用BroadcastMonitorSms类拦截到用户短信并通过abortBroadcast()阻断广播子继续传播。

![enter image description here](http://drops.javaweb.org/uploads/images/865bbb1ac1a50a976b9100edca9cd0d6610b5657.jpg)

被感染手机接收到的短信被拦截读取。

![enter image description here](http://drops.javaweb.org/uploads/images/43e2c0f0dbcbd90fc5423ca2090514db0d9d63ae.jpg)

上图中x.a(“26.0”)返回字符串“pdus”，从而获取intent中额puds结构体重保存的短信息。x这个类中保存了，masseges.bin文件解密后得到的键值对，x.a()方法可以通过键（如“26.0”）获取到字符串（如“pdus”） 获取到拦截的短信内容（包括号码，内容，时间）后，保存在一个类中：

![enter image description here](http://drops.javaweb.org/uploads/images/e82cb0d7ad4b5b83d67e2aacb96f7cf9430f6b04.jpg)

将保存信息的类加入到列表中，这部分大量使用继承、多态等技术，使得通过这些服务收集到的信息可被其他类获取、保存并最终上传。

![enter image description here](http://drops.javaweb.org/uploads/images/e51cb31d87f9c6eb2335ba2188ee79b12f95ea0b.jpg)

0x07 主要服务Core部分
=====

病毒在BroadcastMonitor广播接收器被触发后，通过发送action为com.android.service.app的intent，启动主要服务ServiceCore，

![enter image description here](http://drops.javaweb.org/uploads/images/526df8f3b1345feca0dd71e53530fb711cfe30df.jpg)

在ServiceCore中会首先解密保存的加密文件，并将文件内容以键值对的方式保存起来，方便后面取用。检查、安装并执行后门程序，启动主要线程CORE:

![enter image description here](http://drops.javaweb.org/uploads/images/1d9a14e7e52bcd7cea7ea329f10fc03816c00e63.jpg)

解密massege.bin代码：在下图可以看到massege.bin文件，是使用秘钥为"0x5A3D00448D7A912B" 加密方式为："AES/CBC/PKCS5Padding"的对称加密算法。程序在解密文件后，将文件中保存的关键字符串信息以键值对的形式保存在一个map中，以便在后面使用。

![enter image description here](http://drops.javaweb.org/uploads/images/2b1b3f6cea192f5914c48c1ecd782d51a43af054.jpg)

在屏幕关闭后保持程序运行：在用户使用电源键关闭手机屏幕后，手机的cpu实际上保持着运行，也就是说病毒的程序仍然在执行，使得用户不知不觉仍然处在监控中：

![enter image description here](http://drops.javaweb.org/uploads/images/faa2c58d807efd0319a03ca724b9f8956594dbf1.jpg)

下面是启动的Core线程，在该线程中会检查手机的root状态，被感染状态，信息手机状态，并收集手机的各种基本信息，访问远程服务器，发送短信，控制进程的休眠启动等等，这里仅就信息收集部分做说明：进一步调用信息收集，上传的方法如下图，由于样本高度混淆，再加上使用了很多的继承和多态，多线程机制使得代码逻辑跟踪比较复杂，这里只描述一条逻辑，具体情况请以具体反编译代码为准。下面仅对几个关键类进行截图说明，其余跳转则不在此处赘述。

![enter image description here](http://drops.javaweb.org/uploads/images/60b5437a2641ed63e3e4355f49dd78c464e52547.jpg)

![enter image description here](http://drops.javaweb.org/uploads/images/2906de031cb9f65e771a51e6887eed02a3a7dd39.jpg)

上图变量V5为类h的对象

![enter image description here](http://drops.javaweb.org/uploads/images/40033d4667eed8842bc7a90c3f1aa9a99cf6045a.jpg)

图中的数字对应的就是病毒企图隐藏起来的字符串，经过替换后该语句如下：

```
this.d=”SID:”+V0_3.e+“, NID:” + v0_3.f+”” , BID:”+v0_3.g;

```

上图收集了用户的IMSI、IMEI，SID等硬件信息以及所在位置的经纬度信息等， 具体获取方法在U.f（）中，截图如下：

![enter image description here](http://drops.javaweb.org/uploads/images/e9b6f37d1cc383596764ab224b35aa98721c85d1.jpg)

病毒程序在获取上述信息后以短信的形式发送出去

![enter image description here](http://drops.javaweb.org/uploads/images/9f2bee42b2cda356d15a01f22c230a4620dcd528.jpg)

0x08 总结
=====

可以看出，由于HackingTeam事件发生没有多久，很多黑客就已经做出了反应，利用该事件暴露出来的工具和技术对用户进行攻击并且该类病毒对用户隐私威胁极大，属于高危间谍软件，尤其是使用了比较新颖的使用文件给关键字符串加密，在程序运行中解密文件，并以键值对的方式保存关键字符串，在相关位置使用键取值，从而躲避杀软的查杀，这说明黑客在利用网上暴露的公开的技术的同时更自己进行了改造、升级，以更加危险的形式出现。

建议用户安装杀软，在日益严峻的移动安全形势下与安全厂商携手保护好自己的权益。