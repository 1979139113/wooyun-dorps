# TcpDump使用手册

0x01 Tcpdump简介
=====

1.  tcpdump 是一个运行在命令行下的嗅探工具。它允许用户拦截和显示发送或收到过网络连接到该计算机的TCP/IP和其他数据包。tcpdump 是一个在BSD许可证下发布的自由软件。
    
2.  tcpdump是非常强大的网络安全分析工具，可以将网络上截获的数据包保存到文件以备分析。可以定义过滤规则，只截获感兴趣的数据包，以减少输出文件大小和数据包分析时的装载和处理时间。
    
3.  tcpdump 适用于大多数的类Unix系统 操作系统：包括Linux、Solaris、BSD、Mac OS X、HP-UX和AIX 等等。在这些系统中，tcpdump 需要使用libpcap这个捕捉数据的库。其在Windows下的版本称为WinDump；它需要WinPcap驱动，相当于在Linux平台下的libpcap。
    

0x02 Tcpdump用途
=====

tcpdump能够分析网络行为，性能和应用产生或接收网络流量。它支持针对网络层、协议、主机、网络或端口的过滤，并提供and、or、not等逻辑语句来帮助你去掉无用的信息，从而使用户能够进一步找出问题的根源。

也可以使用 tcpdump 的实现特定目的，例如在路由器和网关之间拦截并显示其他用户或计算机通信。通过 tcpdump 分析非加密的流量，如Telnet或HTTP的数据包，查看登录的用户名、密码、网址、正在浏览的网站内容，或任何其他信息。因此系统中存在网络分析工具主要不是对本机安全的威胁，而是对网络上的其他计算机的安全存在威胁。

有很多用户喜欢使用柏克莱数据包过滤器来限制 tcpdump 产生的数据包数量，这样BPF会只把“感兴趣”的数据包到上层软件，可以避免从操作系统 内核向用户态复制其他数据包，降低抓包的CPU的负担以及所需的缓冲区空间，从而减少丢包率。

注:这篇文章只涉及tcpdump的基本用法，请记住tcpdump比我描述的强大的多!

0x03 Tcpdump的安装
=====

做好编译源程序前的准备活动

1.  网上下载获得libpcap和tcpdump
    
    `http://www.tcpdump.org/`
    
2.  安装c编译所需包：`apt-get install build-essential`
    
3.  安装 libpcap的前置：`apt-get install flex,apt-get install bison`
    
4.  安装libpcap。
    
    tcpdump的使用必须有这库。
    
    `tar xvfz libpcap-1.7.3.tar.gz //解压`
    
    进入解压之后的文件目录
    
    ```
    运行./configure  //生成makefile文件`        
    
    make  //进行编译        
    
    make install   //安装   库文件默认安装在目录  /usr/lib,头文件默认安装在  /usr/include
    
    ```
5.  安装tcpdump
    
    `tar xvfz tcpdump.4.7.4.tar.gz //解压`
    
    进入解压之后的文件目录，运行
    
    ```
    ./configure  //生成makefile文件        
    
    make              //进行编译        
    
    make install   //安装   库文件默认安装在目录  /usr/lib,头文件默认安装在  /usr/include
    
    ```

测试是否成功安装：命令行输入 tcpdump有网络信息显示！！

```
Usage: tcpdump [-aAbdDefhHIJKlLnNOpqRStuUvxX#] [ -B size ] [ -c count ]
        [ -C file_size ] [ -E algo:secret ] [ -F file ] [ -G seconds ]
        [ -i interface ] [ -j tstamptype ] [ -M secret ] [ --number ]
        [ -Q in|out|inout ]
        [ -r file ] [ -s snaplen ] [ --time-stamp-precision precision ]
        [ -T type ] [ --version ] [ -V file ]
        [ -w file ] [ -W filecount ] [ -y datalinktype ] [ -z command ]
        [ -Z user ] [ expression ]

```

0x04 Tcpdump的超详细使用命令
=====

```
-A  以ASCII码方式显示每一个数据包(不会显示数据包中链路层头部信息). 在抓取包含
    网页数据的数据包时, 可方便查看数据(nt: 即Handy for capturing web pages).

-c  count
    tcpdump将在接受到count个数据包后退出.

-C  file-size
    (nt: 此选项用于配合-w file 选项使用)
    该选项使得tcpdump 在把原始数据包直接保存到文件中之前, 检查此文件大小是否超过file-size. 如果超过了, 将关闭此文件,
    另创一个文件继续用于原始数据包的记录. 新创建的文件名与-w 选项指定的文件名一致, 但文件名后多了一个数字.
    该数字会从1开始随着新创建文件的增多而增加. file-size的单位是百万字节(nt: 这里指1,000,000个字节,
    并非1,048,576个字节, 后者是以1024字节为1k, 1024k字节为1M计算所得, 即1M=1024 ＊ 1024 ＝ 1,048,576)

-d  以容易阅读的形式,在标准输出上打印出编排过的包匹配码, 随后tcpdump停止.(nt | rt: human readable, 容易阅读的,
    通常是指以ascii码来打印一些信息. compiled, 编排过的. packet-matching code, 包匹配码,含义未知, 需补充)

-dd 以C语言的形式打印出包匹配码.

-ddd    以十进制数的形式打印出包匹配码(会在包匹配码之前有一个附加的'count'前缀).

-D  打印系统中所有tcpdump可以在其上进行抓包的网络接口. 每一个接口会打印出数字编号, 相应的接口名字, 以及可能的一个网络接口
    描述. 其中网络接口名字和数字编号可以用在tcpdump 的-i flag 选项(nt: 把名字或数字代替flag), 来指定要在其上抓包的网络
    接口.

    此选项在不支持接口列表命令的系统上很有用(nt: 比如, Windows 系统, 或缺乏 ifconfig -a 的UNIX系统); 接口的数字
    编号在windows 2000 或其后的系统中很有用, 因为这些系统上的接口名字比较复杂, 而不易使用.

    如果tcpdump编译时所依赖的libpcap库太老,-D 选项不会被支持, 因为其中缺乏 pcap_findalldevs()函数.

-e  每行的打印输出中将包括数据包的数据链路层头部信息

-E  spi@ipaddr algo:secret,...

    可通过spi@ipaddr algo:secret 来解密IPsec ESP包(nt | rt:IPsec Encapsulating Security Payload,
    IPsec 封装安全负载, IPsec可理解为, 一整套对ip数据包的加密协议, ESP 为整个IP 数据包或其中上层协议部分被加密后的数据,
    前者的工作模式称为隧道模式; 后者的工作模式称为传输模式 . 工作原理, 另需补充).

    需要注意的是, 在终端启动tcpdump 时, 可以为IPv4 ESP packets 设置密钥(secret）.

    可用于加密的算法包括des-cbc, 3des-cbc, blowfish-cbc, rc3-cbc, cast128-cbc, 或者没有(none).
    默认的是des-cbc(nt: des, Data Encryption Standard, 数据加密标准, 加密算法未知, 另需补充).
    secret 为用于ESP 的密钥, 使用ASCII 字符串方式表达. 如果以 0x 开头, 该密钥将以16进制方式读入.

    该选项中ESP 的定义遵循RFC2406, 而不是 RFC1827. 并且, 此选项只是用来调试的, 不推荐以真实密钥(secret)来
    使用该选项, 因为这样不安全: 在命令行中输入的secret 可以被其他人通过ps 等命令查看到.

    除了以上的语法格式(nt: 指spi@ipaddr algo:secret), 还可以在后面添加一个语法输入文件名字供tcpdump 使用
    (nt：即把spi@ipaddr algo:secret,... 中...换成一个语法文件名). 此文件在接受到第一个ESP　包时会打开此
    文件, 所以最好此时把赋予tcpdump 的一些特权取消(nt: 可理解为, 这样防范之后, 当该文件为恶意编写时,
    不至于造成过大损害).

-f  显示外部的IPv4 地址时(nt: foreign IPv4 addresses, 可理解为, 非本机ip地址), 采用数字方式而不是名字.
    (此选项是用来对付Sun公司的NIS服务器的缺陷(nt: NIS, 网络信息服务, tcpdump 显示外部地址的名字时会
    用到她提供的名称服务): 此NIS服务器在查询非本地地址名字时,常常会陷入无尽的查询循环).

    由于对外部(foreign)IPv4地址的测试需要用到本地网络接口(nt: tcpdump 抓包时用到的接口)
    及其IPv4 地址和网络掩码. 如果此地址或网络掩码不可用, 或者此接口根本就没有设置相应网络地址和网络
    掩码(nt: linux 下的 'any' 网络接口就不需要设置地址和掩码, 不过此'any'接口可以收到系统中所有接口的
    数据包), 该选项不能正常工作.

-F  file
    使用file 文件作为过滤条件表达式的输入, 此时命令行上的输入将被忽略.

-i  interface

    指定tcpdump 需要监听的接口.  如果没有指定, tcpdump 会从系统接口列表中搜寻编号最小的已配置好的接口(不包括 loopback 接口).
    一但找到第一个符合条件的接口, 搜寻马上结束.

    在采用2.2版本或之后版本内核的Linux 操作系统上, 'any' 这个虚拟网络接口可被用来接收所有网络接口上的数据包
    (nt: 这会包括目的是该网络接口的, 也包括目的不是该网络接口的). 需要注意的是如果真实网络接口不能工作在'混杂'模式(promiscuous)下,
    则无法在'any'这个虚拟的网络接口上抓取其数据包.

    如果 -D 标志被指定, tcpdump会打印系统中的接口编号，而该编号就可用于此处的interface 参数.

-l  对标准输出进行行缓冲(nt: 使标准输出设备遇到一个换行符就马上把这行的内容打印出来).
    在需要同时观察抓包打印以及保存抓包记录的时候很有用. 比如, 可通过以下命令组合来达到此目的:
    ``tcpdump  -l  |  tee dat'' 或者 ``tcpdump  -l   > dat  &  tail  -f  dat''.
    (nt: 前者使用tee来把tcpdump 的输出同时放到文件dat和标准输出中, 而后者通过重定向操作'>', 把tcpdump的输出放到
    dat 文件中, 同时通过tail把dat文件中的内容放到标准输出中)

-L  列出指定网络接口所支持的数据链路层的类型后退出.(nt: 指定接口通过-i 来指定)

-m  module
    通过module 指定的file 装载SMI MIB 模块(nt: SMI，Structure of Management Information, 管理信息结构
    MIB, Management Information Base, 管理信息库. 可理解为, 这两者用于SNMP(Simple Network Management Protoco)
    协议数据包的抓取. 具体SNMP 的工作原理未知, 另需补充).

    此选项可多次使用, 从而为tcpdump 装载不同的MIB 模块.

-M  secret
    如果TCP 数据包(TCP segments)有TCP-MD5选项(在RFC 2385有相关描述), 则为其摘要的验证指定一个公共的密钥secret.

-n  不对地址(比如, 主机地址, 端口号)进行数字表示到名字表示的转换.

-N  不打印出host 的域名部分. 比如, 如果设置了此选现, tcpdump 将会打印'nic' 而不是 'nic.ddn.mil'.

-O  不启用进行包匹配时所用的优化代码. 当怀疑某些bug是由优化代码引起的, 此选项将很有用.

-p  一般情况下, 把网络接口设置为非'混杂'模式. 但必须注意 , 在特殊情况下此网络接口还是会以'混杂'模式来工作； 从而, '-p' 的设与不设,
    不能当做以下选现的代名词:
    'ether host {local-hw-add}' 或  'ether broadcast'(nt: 前者表示只匹配以太网地址为host 的包, 后者表示匹配以太网地址为广播地址的数据包).

-q  快速(也许用'安静'更好?)打印输出. 即打印很少的协议相关信息, 从而输出行都比较简短.

-R  设定tcpdump 对 ESP/AH 数据包的解析按照 RFC1825而不是RFC1829(nt: AH, 认证头, ESP， 安全负载封装,
    这两者会用在IP包的安全传输机制中). 如果此选项被设置, tcpdump 将不会打印出'禁止中继'域(nt: relay prevention field). 另外,
     由于ESP/AH规范中没有规定ESP/AH数据包必须拥有协议版本号域,
    所以tcpdump不能从收到的ESP/AH数据包中推导出协议版本号.

-r  file
    从文件file 中读取包数据. 如果file 字段为 '-' 符号, 则tcpdump 会从标准输入中读取包数据.

-S  打印TCP 数据包的顺序号时, 使用绝对的顺序号, 而不是相对的顺序号.(nt: 相对顺序号可理解为, 相对第一个TCP 包顺序号的差距,
    比如, 接受方收到第一个数据包的绝对顺序号为232323, 对于后来接收到的第2个,第3个数据包, tcpdump会打印其序列号为1, 2分别
    表示与第一个数据包的差距为1 和 2. 而如果此时-S 选项被设置, 对于后来接收到的第2个, 第3个数据包会打印出其绝对顺序号:
    232324, 232325).

-s  snaplen
    设置tcpdump的数据包抓取长度为snaplen, 如果不设置默认将会是68字节(而支持网络接口分接头(nt: NIT, 上文已有描述,
    可搜索'网络接口分接头'关键字找到那里)的SunOS系列操作系统中默认的也是最小值是96).
    68字节对于IP, ICMP(nt: Internet Control Message Protocol,
    因特网控制报文协议), TCP 以及 UDP 协议的报文已足够, 但对于名称服务(nt: 可理解为dns, nis等服务), NFS服务相关的
    数据包会产生包截短. 如果产生包截短这种情况, tcpdump的相应打印输出行中会出现''[|proto]''的标志（proto 实际会显示为
    被截短的数据包的相关协议层次). 需要注意的是, 采用长的抓取长度(nt: snaplen比较大), 会增加包的处理时间, 并且会减少
    tcpdump 可缓存的数据包的数量， 从而会导致数据包的丢失. 所以, 在能抓取我们想要的包的前提下, 抓取长度越小越好.
    把snaplen 设置为0 意味着让tcpdump自动选择合适的长度来抓取数据包.

-T  type
    强制tcpdump按type指定的协议所描述的包结构来分析收到的数据包.  目前已知的type 可取的协议为:
    aodv (Ad-hoc On-demand Distance Vector protocol, 按需距离向量路由协议, 在Ad hoc(点对点模式)网络中使用),
    cnfp (Cisco  NetFlow  protocol),  rpc(Remote Procedure Call), rtp (Real-Time Applications protocol),
    rtcp (Real-Time Applications con-trol protocol), snmp (Simple Network Management Protocol),
    tftp (Trivial File Transfer Protocol, 碎文件协议), vat (Visual Audio Tool, 可用于在internet 上进行电
    视电话会议的应用层协议), 以及wb (distributed White Board, 可用于网络会议的应用层协议).

-t     在每行输出中不打印时间戳

-tt    不对每行输出的时间进行格式处理(nt: 这种格式一眼可能看不出其含义, 如时间戳打印成1261798315)

-ttt   tcpdump 输出时, 每两行打印之间会延迟一个段时间(以毫秒为单位)

-tttt  在每行打印的时间戳之前添加日期的打印

-u     打印出未加密的NFS 句柄(nt: handle可理解为NFS 中使用的文件句柄, 这将包括文件夹和文件夹中的文件)

-U    使得当tcpdump在使用-w 选项时, 其文件写入与包的保存同步.(nt: 即, 当每个数据包被保存时, 它将及时被写入文件中,
      而不是等文件的输出缓冲已满时才真正写入此文件)

       -U 标志在老版本的libcap库(nt: tcpdump 所依赖的报文捕获库)上不起作用, 因为其中缺乏pcap_cump_flush()函数.

-v    当分析和打印的时候, 产生详细的输出. 比如, 包的生存时间, 标识, 总长度以及IP包的一些选项. 这也会打开一些附加的包完整性
      检测, 比如对IP或ICMP包头部的校验和.

-vv   产生比-v更详细的输出. 比如, NFS回应包中的附加域将会被打印, SMB数据包也会被完全解码.

-vvv  产生比-vv更详细的输出. 比如, telent 时所使用的SB, SE 选项将会被打印, 如果telnet同时使用的是图形界面,
      其相应的图形选项将会以16进制的方式打印出来(nt: telnet 的SB,SE选项含义未知, 另需补充).

-w    把包数据直接写入文件而不进行分析和打印输出. 这些包数据可在随后通过-r 选项来重新读入并进行分析和打印.

-W    filecount
      此选项与-C 选项配合使用, 这将限制可打开的文件数目, 并且当文件数据超过这里设置的限制时, 依次循环替代之前的文件, 这相当
      于一个拥有filecount 个文件的文件缓冲池. 同时, 该选项会使得每个文件名的开头会出现足够多并用来占位的0, 这可以方便这些
      文件被正确的排序.

-x    当分析和打印时, tcpdump 会打印每个包的头部数据, 同时会以16进制打印出每个包的数据(但不包括连接层的头部).
      总共打印的数据大小不会超过整个数据包的大小与snaplen 中的最小值. 必须要注意的是, 如果高层协议数据没有snaplen 这么长,
      并且数据链路层(比如, Ethernet层)有填充数据, 则这些填充数据也会被打印.(nt: so for link  layers  that
      pad, 未能衔接理解和翻译, 需补充 )

-xx   tcpdump 会打印每个包的头部数据, 同时会以16进制打印出每个包的数据, 其中包括数据链路层的头部.

-X    当分析和打印时, tcpdump 会打印每个包的头部数据, 同时会以16进制和ASCII码形式打印出每个包的数据(但不包括连接层的头部).
      这对于分析一些新协议的数据包很方便.

-XX   当分析和打印时, tcpdump 会打印每个包的头部数据, 同时会以16进制和ASCII码形式打印出每个包的数据, 其中包括数据链路层的头部.
      这对于分析一些新协议的数据包很方便.

-y    datalinktype
      设置tcpdump 只捕获数据链路层协议类型是datalinktype的数据包

-Z    user
      使tcpdump 放弃自己的超级权限(如果以root用户启动tcpdump, tcpdump将会有超级用户权限), 并把当前tcpdump的
      用户ID设置为user, 组ID设置为user首要所属组的ID(nt: tcpdump 此处可理解为tcpdump 运行之后对应的进程)

      此选项也可在编译的时候被设置为默认打开.(nt: 此时user 的取值未知, 需补充)

```

0x05 Tcpdump表达式详解
=====

```
该表达式用于决定哪些数据包将被打印.  
如果不给定条件表达式, 网络上所有被捕获的包都会被打印,
否则, 只有满足条件表达式的数据包被打印.(nt: all packets, 可理解为, 所有被指定接口捕获的数据包).
表达式由一个或多个表达元组成(nt: primitive, 表达元, 可理解为组成表达式的基本元素). 
一个表达元通常由一个或多个修饰符(qualifiers)后跟一个名字或数字表示的id组成(nt: 即, qualifiers id).
有三种不同类型的 修饰符:type, dir以及 proto.

type 修饰符指定id 所代表的对象类型, id可以是名字也可以是数字. 
可选的对象类型有: host, net, port 以及portrange
(nt: host 表明id表示主机, net 表明id是网络, port 表明id是端口, 而portrange 表明id 是一个端口范围).  
如, host foo, net 128.3, port 20, portrange 6000-6008
(nt: 分别表示主机 foo,网络 128.3, 端口 20, 端口范围 6000-6008). 
如果不指定type 修饰符, id默认的修饰符为host.

dir 修饰符描述id 所对应的传输方向, 即发往id 还是从id 接收
（nt: 而id 到底指什么需要看其前面的type 修饰符）.
可取的方向为: src, dst, src 或 dst, src并且dst.
(nt:分别表示, id是传输源, id是传输目的, id是传输源或者传输目的, id是传输源并且是传输目的).
例如, src foo,
dst net 128.3, src or dst port ftp-data.
(nt: 分别表示符合条件的数据包中, 源主机是foo, 目的网络是128.3, 源或目的端口为 ftp-data).
如果不指定dir修饰符, id 默认的修饰符为src 或 dst.
对于链路层的协议,比如SLIP(nt: Serial Line InternetProtocol, 串联线路网际网络协议), 以及linux下指定
any 设备, 并指定cooked(nt | rt: cooked 含义未知, 需补充) 抓取类型, 或其他设备类型,
可以用inbound 和 outbount 修饰符来指定想要的传输方向.

proto 修饰符描述id 所属的协议. 可选的协议有: 
ether, fddi, tr, wlan, ip, ip6, arp, rarp, decnet, tcp以及 upd.
(nt | rt: ether, fddi, tr, 具体含义未知, 需补充. 可理解为物理以太网传输协议, 光纤分布数据网传输协议,
以及用于路由跟踪的协议.  wlan, 无线局域网协议; ip,ip6 即通常的TCP/IP协议栈中所使用的ipv4以及ipv6网络层协议;
arp, rarp 即地址解析协议, 反向地址解析协议; decnet, Digital Equipment Corporation
开发的, 最早用于PDP-11 机器互联的网络协议; tcp and udp, 即通常TCP/IP协议栈中的两个传输层协议).
 例如, ether src foo, arp net 128.3, tcp port 21, udp portrange 7000-7009分别表示 
从以太网地址foo 来的数据包,发往或来自128.3网络的arp协议数据包, 
发送或接收端口为21的tcp协议数据包, 发送或接收端口范围为7000-7009的udp协议数据包.
如果不指定proto 修饰符, 则默认为与相应type匹配的修饰符. 例如, src foo 含义是 
(ip or arp or rarp) src foo (nt: 即, 来自主机foo的ip/arp/rarp协议数据包, 默认type为host),net bar 含义是
(ip  or  arp  or rarp) net bar(nt: 即, 来自或发往bar网络的ip/arp/rarp协议数据包),
port 53 含义是 (tcp or udp) port 53(nt: 即, 发送或接收端口为53的tcp/udp协议数据包).
(nt: 由于tcpdump 直接通过数据链路层的 BSD 数据包过滤器或 DLPI(datalink provider interface, 数据链层提供者接口)
来直接获得网络数据包, 其可抓取的数据包可涵盖上层的各种协议, 包括arp, rarp, icmp(因特网控制报文协议),
ip, ip6, tcp, udp, sctp(流控制传输协议).
对于修饰符后跟id 的格式,可理解为, type id 是对包最基本的过滤条件: 即对包相关的主机, 网络, 端口的限制;
dir 表示对包的传送方向的限制; proto表示对包相关的协议限制)
fddi(nt: Fiber Distributed Data Interface) 
实际上与ether 含义一样: tcpdump 会把他们当作一种指定网络接口上的数据链路层协议.
如同ehter网(以太网), FDDI 的头部通常也会有源, 目的, 以及包类型, 从而可以像ether

网数据包一样对这些域进行过滤. 此外, FDDI 头部还有其他的域, 但不能被放到表达式中用来过滤
同样, tr 和 wlan 也和 ether 含义一致, 上一段对fddi 的描述同样适用于tr(Token Ring) 和
wlan(802.11 wireless LAN)的头部. 对于802.11 协议数据包的头部, 目的域称为DA, 源域称为 SA;
而其中的 BSSID, RA, TA 域(nt | rt: 具体含义需补充)不会被检测(nt: 不能被用于包过虑表达式中).


除以上所描述的表达元(primitive)， 还有其他形式的表达元, 并且与上述表达元格式不同.
比如: gateway, broadcast, less, greater
以及算术表达式(nt: 其中每一个都算一种新的表达元). 下面将会对这些表达元进行说明.

表达元之间还可以通过关键字and, or 以及 not 进行连接, 从而可组成比较复杂的条件表达式. 
比如,host foo and not port ftp and not port ftp-data
(nt: 其过滤条件可理解为, 数据包的主机为foo,并且端口不是ftp(端口21) 和ftp-data
(端口20, 常用端口和名字的对应可在linux 系统中的/etc/service 文件中找到)).
为了表示方便, 同样的修饰符可以被省略, 如tcp dst port ftp or ftp-data or domain与以下的表达式
含义相同tcp dst port ftp or tcp dst port ftp-data or tcp dst port domain.
(nt: 其过滤条件可理解为, 包的协议为tcp, 目的端口为ftp 或 ftp-data 或 domain(端口53) ).

```

0x06 Tcpdump常用命令实例
=====

默认启动
----

```
tcpdump

```

普通情况下，直接启动tcpdump将监视第一个网络接口上所有流过的数据包。

监听网卡eth0
--------

```
tcpdump -i eth0

```

这个方式最简单了，但是用处不多，因为基本上只能看到数据包的信息刷屏，压根看不清，可以使用ctrl+c中断退出，如果真有需求，可以将输出内容重定向到一个文件，这样也更方便查看。

监听指定的主机
-------

```
tcpdump -i eth0 -nn 'host 192.168.168.2'

```

这样的话，192.168.168.2这台主机接收到的包和发送的包都会被抓取。

```
tcpdump -i eth0 -nn 'src host 192.168.168.2'

```

这样只有192.168.168.2这台主机发送的包才会被抓取。

```
tcpdump -i eth0 -nn 'dst host 192.168.168.2'

```

这样只有192.168.168.2这台主机接收到的包才会被抓取。

监听指定端口
------

```
tcpdump -i eth0 -nnA 'port 80'

```

上例是用来监听主机的80端口收到和发送的所有数据包，结合-A参数，在web开发中，真是非常有用。

监听指定主机和端口
---------

```
tcpdump -i eth0 -nnA 'port 80 and src host 192.168.168.2'

```

多个条件可以用and，or连接。上例表示监听192.168.168.2主机通过80端口发送的数据包。

监听除某个端口外的其它端口
-------------

```
tcpdump -i eth0 -nnA '!port 22'

```

如果需要排除某个端口或者主机，可以使用“!”符号，上例表示监听非22端口的数据包。

抓取特定目标ip和端口的包
-------------

```
tcpdump host 192.168.168.2 and tcp port 8000

```

捕获的数据太多，不断刷屏，可能需要将数据内容记录到文件里，需要使用-w参数：
--------------------------------------

```
tcpdump -X -s 0 -w A.cap host 192.168.168.2 and tcp port 8000

```

则将之前显示在屏幕中的内容，写入tcpdump可执行文件同级目录下的A.cap文件中。

文件查看方式如下，需要使用-r参数：
------------------

```
tcpdump -X -s 0 -r test.cap host 192.168.168.2 and tcp port 8000

```

使用tcpdump抓取HTTP包
----------------

```
tcpdump  -XvvennSs 0 -i eth0 tcp[20:2]=0x4745 or tcp[20:2]=0x4854

```

0x4745 为"GET"前两个字母"GE",0x4854 为"HTTP"前两个字母"HT"。

tcpdump 对截获的数据并没有进行彻底解码，数据包内的大部分内容是使用十六进制的形式直接打印输出的。显然这不利于分析网络故障，通常的解决办法是先使用带-w参数的tcpdump 截获数据并保存到文件中，然后再使用其他程序(如Wireshark)进行解码分析。当然也应该定义过滤规则，以避免捕获的数据包填满整个硬盘。

0x07 tcpdump 与wireshark
=====

Wireshark(以前是ethereal)是Windows下非常简单易用的抓包工具。但在Linux下很难找到一个好用的图形化抓包工具。

还好有Tcpdump。我们可以用Tcpdump + Wireshark 的完美组合实现：在 Linux 里抓包，然后在Windows 里分析包。

tcpdump tcp -i eth1 -t -s 0 -c 100 and dst port ! 22 and src net 192.168.1.0/24 -w ./target.cap

(1)tcp: ip icmp arp rarp 和 tcp、udp、icmp这些选项等都要放到第一个参数的位置，用来过滤数据报的类型

(2)-i eth1 : 只抓经过接口eth1的包

(3)-t : 不显示时间戳

(4)-s 0 : 抓取数据包时默认抓取长度为68字节。加上-S 0 后可以抓到完整的数据包

(5)-c 100 : 只抓取100个数据包

(6)dst port ! 22 : 不抓取目标端口是22的数据包

(7)src net 192.168.1.0/24 : 数据包的源网络地址为192.168.1.0/24

(8)-w ./target.cap : 保存成cap文件，方便用ethereal(即wireshark)分析