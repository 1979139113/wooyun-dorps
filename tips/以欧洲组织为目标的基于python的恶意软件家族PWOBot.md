# 以欧洲组织为目标的基于python的恶意软件家族PWOBot

Links:[http://researchcenter.paloaltonetworks.com/2016/04/unit42-python-based-pwobot-targets-european-organizations/](http://researchcenter.paloaltonetworks.com/2016/04/unit42-python-based-pwobot-targets-european-organizations/)

0x00 前言
=====

我们发现了一个叫做“PWOBot”的恶意软件家族，这个恶意软件家族相当的独特，因为它完全是用Python编写的，并通过[PyInstaller](http://www.pyinstaller.org/)进行编译来生成一个windows下的可执行程序。这个恶意软件被证实影响了大量的欧洲组织，特别是在波兰。此外，这个恶意软件只通过一个流行的波兰文件共享web服务来传播的。

这个恶意软件本身提供了丰富的功能，包括能够下载和执行文件，执行Python代码，记录键盘输入，生成一个HTTP服务器，并且通过受害者的CPU和GPU挖掘比特币。

现在至少有PWOBot的12个变种，并且这个恶意软件早在2013年末的攻击中就开始活跃了。更多最近的影响欧洲组织的攻击是在2015年中到年末。

0x01 目标
=====

在过去的半年中，我们发现PWOBot影响了以下的组织：

*   波兰国家研究机构
*   波兰航运公司
*   大型的波兰零售商
*   波兰信息技术公司
*   丹麦建筑公司
*   法国光学设备提供商

大部分PWOBot的样本都是从`chomikuj.pl`（波兰流行的文件共享web服务）上下载的。下面的这些奇特的URL被发现提供了PWOBot的副本：

```
s6216.chomikuj[.]pl/File.aspx?e=Pdd9AAxFcKmWlkqPtbpUrzfDq5_SUJBOz
s6102.chomikuj[.]pl/File.aspx?e=Hc4mp1AqJcyitgKbZvYM4th0XwQiVsQDW
s8512.chomikuj[.]pl/File.aspx?e=h6v10uIP1Z1mX2szQLTMUIoAmU3RcW5tv
s6429.chomikuj[.]pl/File.aspx?e=LyhX9kLrkmkrrRDIf6vq7Vs8vFNhqHONt
s5983.chomikuj[.]pl/File.aspx?e=b5Xyy93_GHxrgApU8YJXJlOUXWxjXgW2w
s6539.chomikuj[.]pl/File.aspx?e=EH9Rj5SLl8fFxGU-I0VZ3FdOGBKSSUQhl
s6701.chomikuj[.]pl/File.aspx?e=tx0a8KUhx57K8u_LPZDAH18ib-ehvFlZl
s6539.chomikuj[.]pl/File.aspx?e=EH9Rj5SLl8fFxGU-I0VZ3ISlGKLuMnr9H
s6539.chomikuj[.]pl/File.aspx?e=EH9Rj5SLl8fFxGU-I0VZ3OFFAuDc0M9m0
s6179.chomikuj[.]pl/File.aspx?e=Want-FTh0vz6www2xalnT1Nk6O_Wc6huR
s6424.chomikuj[.]pl/File.aspx?e=o_4Gk0x3F9FWxSDo4JWYuvGXDCsbytZMY

```

另外，有一次这个恶意软件被从http://[108.61.167.105](https://www.virusbook.cn/ip/108.61.167.105)/favicon.png。这个IP地址是和[tracking.huijang.com](https://www.virusbook.cn/domain/tracking.huijang.com)有联系的，而这个域名被相当数量的PWOBot所使用。

下面的这些文件名字被发现用来传播PWOBot：

*   favicon.png
*   Quick PDF to Word 3.0.exe
*   XoristDecryptor 2.3.19.0 full ver.exe
*   Easy Barcode Creator 2.2.6.exe
*   Kingston Format Utility 1.0.3.0.exe
*   uCertify 1Z0-146 Oracle Database 8.05.05 Premium.exe
*   Six Sigma Toolbox 1.0.122.exe
*   Fizjologia sportu. Krtkie wykady.exe [Physiology of sports. Short lectures.exe]

正如我们能从使用的文件名中所看到的，相当一部分的PWOBot样本伪装成了各种各样的软件。在某些情况下，波兰语被认为是更容易被当成目标的文件名。

目前尚不清楚这个恶意软件最初是怎样被发送到终端用户的。我们可以根据文件名作出推论，这个恶意软件很可能是在终端用户下载其他软件时被传播的。因此，钓鱼攻击可能被用来引诱受害者下载这些文件。

0x02 恶意软件分析
=====

正如最开始提到的，PWOBot是完全用Python编写的。攻击者利用PyInstaller来把Python代码转换成Windows可执行程序。因此，因为Python被使用了，所以它可以很简单的被移植到其他操作系统，比如Linux或者OSX。

除了最初的运行之外，PWOBot会首先卸载掉它可能会发现的之前的PWOBot的版本。它会查询`Run`注册表项，判断是否存在之前的版本。主要的版本针对注册表项`Run`使用了一种`pwo[VERSION]`的格式，在这里`[version]`代表的是PWOBot的版本号。

![](http://drops.javaweb.org/uploads/images/e965c744d52934ab3db38679ef3f01639281f417.jpg)

图一 PWOBot uninstalling previous versions

在所有之前的版本被卸载之后，PWOBot会进行自我安装并创建一个它自己的可执行文件的副本，存在以下位置：

`%HOMEPATH%/pwo[version]`

接下来它会设置以下的注册表键值来把它指向到新拷贝过来的可执行文件上：

`HKCU/SOFTWARE/Microsoft/Windows/CurrentVersion/Run/pwo[VERSION]`

如果这是这个恶意软件第一次运行，PWOBot会在一个新的进程里执行新复制的文件。

在安装完毕之后，PWOBot会对各种各样的键盘和鼠标事件进行HOOK，这会在接下来的键盘记录活动中被用到。PWOBot是用模块化的风格编写的，允许攻击者在运行时包含各种模块。基于对当前已有的样本的分析，以下的服务被发现带有PWOBot：

*   PWOLauncher : 下载/执行文件，或者执行本地文件
*   PWOHTTPD : 在受害者机器上大量生成HTTP服务器
*   PWOKeyLogger : 在受害者机器上进行键盘记录
*   PWOMiner : 使用受害者机器的CPU/GPU挖掘比特币
*   PWOPyExec : 运行Python代码
*   PWOQuery : 查询远程URL并返回结果

PWOBot有两个配置文件，其中一指定了这个恶意软件的各种配置，另一个确定了PWOBot在执行的时候应该连接哪个远程服务器。

![](http://drops.javaweb.org/uploads/images/dd0d8025e7c22d50845b8bb43d03e79a13bc6af1.jpg)

图二 PWOBot settings configuration

![](http://drops.javaweb.org/uploads/images/46822739725c3899cd3733adc93db18c38aba5f7.jpg)

图三 PWOBot remote server configuration

正如在配置图中所可以看到的，PWOBot包含了很多Windows的可执行文件，这些可执行文件是在攻击者使用PyInstaller来对代码进行编译的时候包含进去的。这些可执行文件被用来进行比特币挖掘以及利用TOR发送代理服务器请求。比特币挖掘是`minerd`和`cgminer`的一个编译好的版本。这些文件分别被用来作为CPU和GPU的比特币挖掘。

PWOBot也使用了Tor匿名网络来对攻击者的远程服务器的通信进行加密。PWOBot使用了一个Python字典作为网络协议。每一个特定的时间段PWOBot都会发送一段通知信息到远程服务器上去。这样的通知消息的例子可以如下所示：

```
{
  1: '16ea15e51a413f38c7e3bdb456585e3c', 
  3: 6, 
  4: '[REDACTED-USERNAME]', 
  5: True, 
  6: {
       1: 'Darwin', 
       2: 'PANHOSTNAME', 
       3: '14.5.0', 
       4: 'Darwin Kernel Version 14.5.0: Tue Sep  1 21:23:09 PDT 2015; root:xnu-2782.50.1~1/RELEASE_X86_64', 
       5: 'x86_64', 
       6: 'i386', 
       7: 8
     }, 
  7: {
       1: 'en_US', 
       2: 'UTF-8', 
       3: 25200
     }
}

```

针对上面的例子中列举的各个数据都有不同的枚举类型。替换之后我们可以看到更完整的被发送的数据。

```
{
  BOT_ID: '16ea15e51a413f38c7e3bdb456585e3c', 
  VERSION: 6, 
  USER: '[REDACTED-USERNAME]', 
  IS_ADMIN: True, 
  PLATFORM: {
       SYSTEM: 'Darwin', 
       NODE: 'PANHOSTNAME', 
       RELEASE: '14.5.0', 
       VERSION: 'Darwin Kernel Version 14.5.0: Tue Sep  1 21:23:09 PDT 2015; root:xnu-2782.50.1~1/RELEASE_X86_64', 
       MACHINE: 'x86_64', 
       PROCESSOR: 'i386', 
       CORES: 8
     }, 
  LOCALE: {
       LANGUAGE: 'en_US', 
       ENCODING: 'UTF-8', 
       TIMEZONE: 25200
     }
}

```

在通知被发送之后，攻击者可能会选择提供一条指令来让PWOBot来执行之前定义好的其中一项服务。上述行为的结果会在随后使用相同的格式上传给攻击者。

总的来说，基于Palo Alto Networks的Unit 42发现的最近的版本，目前存在12个PWOBot的变种。在这12个版本之中我们已经在网络上发现了其中的第5、6、7、9、10和12个版本。不同的版本之间的差别很小，以及存在不同的性能上的区别。

0x03 结论
=====

PWOBot作为一个恶意软件家族是非常有意思的，因为它是完全用python写的。尽管在历史上它只影响过windows平台，但是因为它的底层的代码是跨平台的，它可以很简单的被移植到Linux和OSX上去。这个事实以及它的模块化的设计，让PWOBot成为一个潜在的重要的威胁。

这个恶意软件家族在之前并没有被公开的披露过。当前它被证实影响了一些欧洲的组织。

Palo Alto Networks的用户从以下的几方面被保护：

*   所有的PWOBot样本都被WildFire服务恰当的判定为恶意的。
*   和PWOBot相关的域名被划分为恶意的。
*   AutoFocus用户可以使用PWOBot tag来监控这次威胁。

相关的文件：

1.  http://www.pyinstaller.org/
2.  https://www.torproject.org/
3.  https://github.com/pan-unit42/iocs/tree/master/pwobot/hashes.txt