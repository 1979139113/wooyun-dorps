# Remaiten-一个以路由器和IoT设备为目标的Linux bot

[http://www.welivesecurity.com/2016/03/30/meet-remaiten-a-linux-bot-on-steroids-targeting-routers-and-potentially-other-iot-devices](http://www.welivesecurity.com/2016/03/30/meet-remaiten-a-linux-bot-on-steroids-targeting-routers-and-potentially-other-iot-devices)

0x00 前言
=====

ESET的研究人员正在积极地检测以嵌入式系统为攻击目标的木马，受影响的有路由器，网关和无线访问点。近期，我们已经发现了一个相关的bot，这个bot整合了Tsunami（也叫作Kaiten）和Gafgyt的功能，并相较于前者做出了一些改进，提供了新的功能。这个新威胁就是Linux/Remaiten。截至目前，我们已经发现了三个版本的Linux/Remaiten，版本号分别是2.0，2.1和2.2。根据其代码来看，木马作者称之为“KTN-Remastered”或“KTN-RM”。

在本文中，我们会说明Linux/Remaiten的特殊传播机制，新增功能，以及不同版本之间的差别。

0x01 改进后的传播机制
=====

Linux/Gafgyt最突出的功能就是Telnet扫描。在执行Telnet扫描时，木马会尝试通过互联网端口23连接到随机的IP地址。如果连接成功，木马会根据内置的一份用户名/密码列表，尝试猜测登录凭证。登录成功后，木马会发出一个shell命令，下载多个不同架构的bot可执行文件，并尝试运行这些bot。这种感染方式虽然简单，但是会产生很多干扰，因为只有一个二进制能够运行在当前架构下。

Linux/Remaiten通过携带downloader，改进了上述的传播机制。木马的downloader是专门针对嵌入式Linux设备的CPU架构，比如ARM和MIPS。在通过telnet登录了受害设备后，木马会尝试判断设备平台，并传输适用于该平台的downloader。这个downloader的任务是联系CC服务器，请求适用于设备平台的Linux/Remaiten bot二进制。然后，在新的受害设备上运行bot二进制，创建一个新的bot供攻击者使用。

0x02 技术分析downloader
=====

Linux/Remaiten downloader是一个小型的ELF可执行文件，内嵌在bot二进制中。当执行时，downloader会连接到bot的CC服务器，并发送下面的某个请求，然后另起一行：

*   mips
*   mipsel
*   armeabi
*   armebeabi

CC服务器会根据请求的架构，响应一个ELF bot二进制。注意，这里用于连接CC服务器的TCP端口并不是bot的IRC服务器。

![p2](http://drops.javaweb.org/uploads/images/a5acaa4f002fb460ebe6f21fe3bbfaf6cae30eb6.jpg)图1-downloader向CC请求一个bot二进制

![p3](http://drops.javaweb.org/uploads/images/cc9ae09cea1ac6e7a3245cb13c058f0b9cd3b3d7.jpg)图2-downloader正在连接CC

downloader的唯一任务就是向CC服务器发送前面提到的某条命令并将响应写入到stdout。在这个例子中，发送的命令是mips。

![p4](http://drops.javaweb.org/uploads/images/0fc890c23b87405ed6864b11350189255bd98e3c.jpg)图3-downloader向CC请求一个mips架构的bot

0x03 bot分析
==========

在执行时，bot默认在后台运行。使用“-d”命令运行bot时，bot会保持在前台。一旦启动，进程会伪装成合法的名称，比如“-bash” 或“-sh”。我们观察发现，2.0和2.1版本使用的是“-bash” ，2.2版本使用的是“-sh”。

![p5](http://drops.javaweb.org/uploads/images/e47579179b688c7c569a71c428a082151b21bd31.jpg)图4-bot启动

接下来，函数create_daemon 会创建一个名称是“.kpid”的文件，创建位置是下面的某个预设守护进程目录（第一个具有写入权限），函数还会讲其PID写入到这个文件。

![p6](http://drops.javaweb.org/uploads/images/a776487415bce5450ae26b542ecbcd592d12a191.jpg)图5-守护进程文件目录

如果已经存在“.kpid”文件，根据文件中的PID，运行中的木马进程就会被杀掉。然后，这个文件会被移除，接着，创建新的“.kpid”，并继续执行。

![p7](http://drops.javaweb.org/uploads/images/85064e31226033454bfad1f60b01e6dd74fbb5e8.jpg)图6-跟踪pid文件的创建

0x04 连接到CC服务器
=====

在bot二进制中，硬编码了一个CC服务器IP地址表。bot会随机选择一个地址，并通过一个硬编码端口连接到选中的CC。不同的变种会使用不同的端口。

![p8](http://drops.javaweb.org/uploads/images/8b8530fea5c02df601990006891520531507dca3.jpg)图7-bot连接一个CC服务器

如果连接成功，bot随后会进入IRC通道。CC则会响应一条welcome信息和后续指令。bot会在受感染设备上解析并执行这些指令。

![p9](http://drops.javaweb.org/uploads/images/ec28e735ad8d00169a9367a8f0a8582d69f6372f.jpg)图8-CC响应给bot的欢迎信息

0x05 处理IRC命令
=====

bot可以处理多种通用的IRC命令。这些命令和函数处理程序都会以阵列的形式列出。

![p10](http://drops.javaweb.org/uploads/images/1e78afa3f59029ec3e135f545c15b07da677a00e.jpg)图9-IRC命令

其中最有意思的是“PRIVMSG”命令。这个命令会要求bot执行一些恶意操作，比如，flooding，下载文件，telnet扫描等。通过“PRIVMSG”发送的命令也是以静态阵列的形式呈现。

![p11](http://drops.javaweb.org/uploads/images/f7e4361aa0c7547be80c8938d4fccbc6f2d350bd.jpg)图10-可用的bot命令

大部分的功能都是来自Linux/Tsunami和Linux/Gafgyt。二进制中的下列字符串与恶意行为有关。这些详细的介绍，能让我们知道这些字符串的作用。

![p12](http://drops.javaweb.org/uploads/images/363451a00a3f3bef0b25cfb9c150523face863af.jpg)图11-flooding功能

![p13](http://drops.javaweb.org/uploads/images/14dfde9929fe1ca3145f5aaf1feef7e560529bba.jpg)图12-Telnet扫描，下载文件，杀掉其他bots

0x06 内置downloader
=====

我们前面提到过，Linux/Remaiten 的特别之处在于携带了多个小型的downloader，如果有符合受害设备架构的版本，木马就会把相应的downloader传输到设备上。在执行时，downloader会联系CC，请求一个bot二进制。

![p14](http://drops.javaweb.org/uploads/images/6b82d25e61004c082531dff42c73bbc2466e2d1b.jpg)图13-内置的有效载荷

![p15](http://drops.javaweb.org/uploads/images/6f9754ad3b958f76d2e2701e1b153f947af8a11b.jpg)图14-有效载荷结构

0x07 Telnet scanner
=====

![p16](http://drops.javaweb.org/uploads/images/9490491a397528d1a1d7f8db20c75fcc8820a1f8.jpg)图15-猜测telnet登录凭证

当CC发出“QTELNET” 命令时，Remaiten的telnet scanner就会启动。分析发现，木马作者提供的命令描述是正确的：这个telnet的确是一个增强版的Gafgyt telnet scanner。

Telnet扫描是分阶段完成的，可以归结为：

1.  选择一个随机的公共IP地址，并将其连接到端口23
2.  尝试用户名/密码组着
3.  确定受害设备的架构
4.  发送并执行相应的downloader

木马会通过执行“cat $SHELL”命令来判断设备的架构，并解析其结果。SHELL环境变量中包含有可执行文件的路径，这个可执行文件目前还是作为一个命令行翻译器。如果这个文件是一个ELF可执行文件，则解析文件标头来判断其架构。

![p17](http://drops.javaweb.org/uploads/images/fa72dcc09ca2aa10f590848fb6e10b6f90a21414.jpg)图16-判断受害平台&检查是否有适合该平台的downloader

![p18](http://drops.javaweb.org/uploads/images/b1aeaf10fab3610da6af867a1efd3a435575d076.jpg)图17-负责解析ELF标头的部分函数

然后，bot会选择合适的有效载荷并发送到新的受害设备上。

![p19](http://drops.javaweb.org/uploads/images/0b626511eefd54294727861d46daf97808d9c7ef.jpg)图18-负责根据设备架构选择有效载荷的函数

第一步是找到一个可写入的目录。Linux/Remaiten中有一个常用的可写入路径表。

![p20](http://drops.javaweb.org/uploads/images/91e11f990649f2ddcdc7fe5151db500c75dbb75e.jpg)图19-downloader的保存目录

创建了几个空的可执行程序：“.t”，“retrieve”和“binary”。“retrieve”文件中会包含有downloader，“binary”是从CC服务器上请求到的bot。似乎2.2版本之前都没有使用“.t”文件。

![p21](http://drops.javaweb.org/uploads/images/14e6cc0063af93f456baae7ef6a9dda90fe8d40d.jpg)图20-准备传输和执行有效载荷

Linux/Remaiten 采用了一种很奇怪的方式来创建空的可执行文件：木马会复制busybox二进制（出现在大多数嵌入式设备上），然后使用>file命令做空这个二进制。

downloader是通过telnet传输的，通过发出echo命令，每个字节都编码了16进制“\x” 转序字节。我们此前就见过有木马利用这种技术在嵌入式Linux设备中进行传播，比如Linux/Moose。

![p22](http://drops.javaweb.org/uploads/images/79a027cc1cb6b957f10821aec80edef673672be3.jpg)图21-传输带有echo的有效载荷hexstring

传输完成后，downloader就会启动并获取完整的Linux/Remaiten有效载荷。downloader会从CC请求一个bot二进制，并将其写入到标准输出，而部署命令会把这个输出重定向到“binary” 文件。最后，启动“binary”文件，激活新的IRC bot。

![p23](http://drops.javaweb.org/uploads/images/d23955425cc7977bba169349fcc947d977a86485.jpg)图22-执行downloader和bot

0x08 向CC发送状态
=====

在恢复telnet扫描之前，bot会向CC服务器报告其进度。bot会发送新的设备IP，正确的用户名/密码，以及是否感染了其他设备。如果自动感染方式失败，僵尸网络管理员可能会手动感染其他的设备或从不受支持的架构上收集数据？

![p24](http://drops.javaweb.org/uploads/images/d1c909435187259b127d485046380a0de4af29b7.jpg)图23-通知CC服务器bot部署状态

0x09 杀掉其他bot
=====

还有一个很有趣的命令是“KILLBOTS”。发出这个命令后，bot会枚举正在运行的进程，然后会根据一定的标准，主要是进程名称，来决定是忽略还是杀掉这个进程。不同版本的bot可能会选择不同的进程名称。

![p25](http://drops.javaweb.org/uploads/images/5f208391ac247ddae68b75f19d70691a9cf73255.jpg)图24-需要杀掉的进程名称

![p26](http://drops.javaweb.org/uploads/images/f83785cc8c983a9d4e930eb12ef639b1696c6a61.jpg)图25-需要忽略的进程名称

Linux/Remaiten会根据进程的tty设备号，只杀掉由一个交互shell启动的进程。另外，木马还会向CC服务器报告都杀掉了哪些进程，这样所或许是为了修改其进程白名单或黑名单。

![p27](http://drops.javaweb.org/uploads/images/3e55a7129396ee16dccb8b4c44ef4a41bd7bb80b.jpg)图26-bot正在杀进程

0x0A Linux/Remaiten变更日志
=====

不同的bot客户端版本之间只是稍有不同，比如，修改进程白名单和黑名单，downloader目录有变化等等。我们有理由怀疑，各个编译版本之间可能会有区别，即使版本号没有变化。在我们分析过的bot中，其downloader二进制都是一样的，但是IP地址和端口有差别。

和Gafgyt一样，v2.2版本仍然会通过执行wget/tftp命令来下载shell脚本，然后，由这个shell脚本下载bot二进制。在传输downloader之前，传播代码首先会尝试这种方法。

![p28](http://drops.javaweb.org/uploads/images/e9a54902a4c79ae74f812dd670d920b800c11ea6.jpg)图27-通知CC通过wget/tftp部署bot

shell脚本是由另一个服务器发布的，这个服务器还会负责发送Gafgyt bot。

![p29](http://drops.javaweb.org/uploads/images/60e458f6723132c9b744fc9114bbd4bc69b81a80.jpg)图28-由另一个服务器发布shell脚本

从al.sh文件来看，这是我们首次发现针对PowerPC和SuperH等平台的bot。虽然，有跨平台编译工具，但是，我们还是很惊讶攻击者会在编译自己的木马时遇到问题。我们不清楚是哪台设备运行在PowerPC或SuperH。

![p30](http://drops.javaweb.org/uploads/images/c714cc4c1ea3b3995ad7bacd7b04558b1dff8e0d.jpg)图29-shell脚本下载的bot

![p31](http://drops.javaweb.org/uploads/images/5c6ddaf056b99973a22b18de5456b0b22d4eb1c3.jpg)图30-shell脚本的开头

2.0版本使用的CC服务器给出了一条出乎我们预料的welcome信息：其中引用了一篇MalwareMustDie博客。

![p32](http://drops.javaweb.org/uploads/images/23c3bbd4bc31c852e7506d4d6fe4210d9c0a737b.jpg)图31-2.0版本的CC引用了MalwareMustDie博客

或许这是为了报复MalwareMustDie 曝光Gafgyt等木马。