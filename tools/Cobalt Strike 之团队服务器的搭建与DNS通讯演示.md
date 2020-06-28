# Cobalt Strike 之团队服务器的搭建与DNS通讯演示

0x00 背景
-------

* * *

Cobalt Strike 一款以metasploit为基础的GUI的框框架式渗透工具，Armitage的商业版，集成了端口发、服务扫描，自动化溢出，多模式端口监听，win exe木马生成，win dll木马生成，java木马生成，office宏病毒生成，木马捆绑，mac os 木马生成，钓鱼攻击包括：站点克隆，目标信息获取，java执行，游览器自动攻击等等。

Cobalt Strike 官网为 [Cobalt Strike](http://www.advancedpentest.com/) 程序只不接受大天朝的下载，各位自行想办法。[作者博客](http://blog.strategiccyber.com/) 有很多好东西，推荐大家收藏。

Cobalt Strike 在1.45和以前是可以连接本机windows的metasploit的，在后来就不被支持了，必须要求连接远程linux的metasploit。

Cobalt Strike 还有个强大的功能就是他的团体服务器功能，它能让多个攻击者同时连接到团体服务器上，共享攻击资源与目标信息和sessions。

这篇文章就给大家分享下我大家团体服务器的方法（我的不一定是最好的，参考下就行了）

0x01 搭建
-------

* * *

### 1.服务器

服务器强烈建议大家选择ubuntu的，内存1G以上，带宽8M以上，虽然在Centos上也帮朋友成功搭建过，但是很不推荐，稳定性和维护性都没有用ubuntu好。

### 2.安装metasploit

metasploit有3个版本 专业版，社区版，和git上面的版本，当然大家用社区版就行了，专业版的功能比起社区版要多，但是要给钱，只能免费使用一段时间，以前找到的无限免费使用专业版也被官方封锁了，git上的版本适合高级安装的用户，具体可以自己去玩玩。

下载安装好社区版的metasploit后接下来就是要激活第一次访问metasploit的web管理页面必须是localhost，这好似规定死的，我这里有两种方法1.给服务器开启VNC，然后上去激活（不推荐）2.连接ssh的时候开启socks5然后游览器设置下就可以访问了。

还有一中快捷的安装方式就是上传Cobalt Strike搭服务器，在里面有个quick-msf-setup 的脚本，它可以帮你快熟部署团体服务器环境，不过我不喜欢这种方式，我比较喜欢折腾，嘿嘿。

### 3.部署Cobalt Strike

将下载好的Cobalt Strike上传到服务器，解包后会有这些文件

![enter image description here](http://drops.javaweb.org/uploads/images/95139658eca46a188203f9b4a589f23bbdda9d1c.jpg)

Cobalt Strike是JAVA写的，服务器还得有JAVA环境，这里我们没必要去下载JAVA来安装，metasploit已经有了JAVA环境，我们只需要配置下环境变量就行了 打开root目录下的.bashrc文件，建议先备份，在最下面添加：

```
#JAVA     
export JAVA_HOME=/opt/metasploit/java

```

  export PATH=$JAVA_HOME/bin:$JRE_HOME/bin:$PATH

然后在执行

```
source .bashrc

```

最后看看是否成功

![enter image description here](http://drops.javaweb.org/uploads/images/757a53798f59eb150aca1dbea2fd55062a297d57.jpg)

回到Cobalt Strike目录

执行./teamserver 服务器IP 连接密码

![enter image description here](http://drops.javaweb.org/uploads/images/5be1e6b92c2691a1f809dd813771130776477195.jpg)

启动的过程中会有很多警告，不用理会它，大概几分钟后出现这个就OK了

![enter image description here](http://drops.javaweb.org/uploads/images/b9dec4a1abf95becee80b7e5e9edb79e32cd32e9.jpg)

这里不要关闭，然后本机启动Cobalt Strike 连接测试

```
地址是:192.168.10.62
端口是:55553
用户名是:msf
密码是:luom  (也就是我们刚才设置的)

```

![enter image description here](http://drops.javaweb.org/uploads/images/bc554abc3dd675d10ebf69114e8e6d693dc25151.jpg)

点击连接，会弹出一个服务器认证，确认，然后弹出设置你昵称（Cobalt Strike是可以在线聊天的）

![enter image description here](http://drops.javaweb.org/uploads/images/9e89c8cf73ba4d30830783cd2942f1e84f63adcc.jpg)

回到SSH 你可以看到各种日志，但是这个一关闭团队服务器也就关闭了，这里我们可以把他置于后台来运行

```
nohup ./teamserver 192.168.10.62 luom &

```

这样就可以了，这个目录下会生成一个nohup.out的文件，这个是程序运行的日志文件 注： 在启动Cobalt Strike的时候报错 如下

![enter image description here](http://drops.javaweb.org/uploads/images/fe1959ac717a047621f8e30dae8f687c3ee9d96e.jpg)

这个原因是你的内存使用超过%50 无法启动java虚拟机。

在结束Cobalt Strike的时候也要同时结束所有 msfrpcd 进程，不要下次启动会启动不了的。

0x02 实例之Cobalt Strike通过DNS控制目标
------------------------------

* * *

通过DNS来控制目标和渗透好处不多说把，大家都知道，不开端口，能绕过大部分防火墙，隐蔽性好等等。Cobalt Strike有个beacons的功能，它可以通过DNS,HTTP,SMB来传输数据，下面我以DNS为例演示下。

### 1. 域名设置

首先我们的有个域名，并且创建一条A记录指向我们的metasploit服务器，记住不要用CDN什么的

![enter image description here](http://drops.javaweb.org/uploads/images/37389098f99da4258681f7a2f3db015195dad4b7.jpg)

然后再创建2个或3个ns记录指向刚才创建的A记录

![enter image description here](http://drops.javaweb.org/uploads/images/0cb701f1dc206c5df56cb56982b972d7c8944112.jpg)

这样我们就可以通过dns找到我们的metasploit服务器了

### 2. Cobalt Strike设置

在Cobalt Strike中我们添加一个listener

![enter image description here](http://drops.javaweb.org/uploads/images/87ee7ff1ec2f40cc7653a9a88ce391bd7d6819cb.jpg)

HOST填写的是metasplit服务的IP，在点击Save的时候会要求填写你的NS记录，这里写入我们刚才创建的3个

![enter image description here](http://drops.javaweb.org/uploads/images/f2c2e2270b4c9dd7c374ada564ae8b08716fd5d3.jpg)

监听我们设置好了，接下来创建一个木马测试下。

### 3. 木马生成

在`attack->packages`中找到windows木马生成

![enter image description here](http://drops.javaweb.org/uploads/images/27a8d8c8c0f308929bc08a80b02f238d70286e3d.jpg)

Listener选择我们刚才创建的(有两个，选择有DNS的那个)，输出的有exe，带服务的EXE，dll等。（我测试过连接方式以DNS生成的DLL木马能过掉很大一部分杀毒软件）

我们把生成的DNS.EXE放到虚拟机中运行。

运行前的端口情况

![enter image description here](http://drops.javaweb.org/uploads/images/efc518b4adbde3e6593776a29c0c523e820088ff.jpg)

运行后的端口情况

![enter image description here](http://drops.javaweb.org/uploads/images/3833eeddd27e98b89772241ea15d216442731856.jpg)

没有开新的端口，在来抓包看看

![enter image description here](http://drops.javaweb.org/uploads/images/be8830f465234413efe3f3d3122038b34f299d92.jpg)

走的是DNS。

回到Cobalt Strike打开beacons管理器发现有一个服务端响应了我们

![enter image description here](http://drops.javaweb.org/uploads/images/a3d1842291ba4bfdd2a3dd67b78fee14bbfd2fa6.jpg)

右键是管理菜单，选择sleep设置相应的时间，然后选择interact来到操作界面

![enter image description here](http://drops.javaweb.org/uploads/images/507e2b7613fcb9b790165bcb33347eec60642c36.jpg)

首先来设置的是传输的模式，有dns、dns-txt，http,smb四种，我们这里用的是DNS就在dns、dns-txt中选择把，前者传送的数据小后者传送的数据多 这里我设置为`mode dns-txt`（这里可以用TAB补齐命令的）

![enter image description here](http://drops.javaweb.org/uploads/images/9069ecd41f5f2a4663fbfca3a5304008d8942d5e.jpg)

键入help可以看到支持的命令

```
    Command                   Description
    -------                   -----------
    bypassuac                 Spawn a session in a high integrity process
    cd                        Change directory
    checkin                   Call home and post data
    clear                     Clear beacon queue
    download                  Download a file
    execute                   Execute a program on target
    exit                      Terminate the beacon session
    getsystem                 Attempt to get SYSTEM
    getuid                    Get User ID
    help                      Help menu
    inject                    Spawn a session in a specific process
    keylogger start           Start the keystroke logger
    keylogger stop            Stop the keystroke logger
    message                   Display a message to user on desktop
    meterpreter               Spawn a Meterpreter session
    link                      Connect to a Beacon peer over SMB
    mode dns                  Use DNS A as data channel (DNS beacon only)
    mode dns-txt              Use DNS TXT as data channel (DNS beacon only)
    mode http                 Use HTTP as data channel
    mode smb                  Use SMB peer-to-peer communication
    rev2self                  Revert to original token
    shell                     Execute a command via cmd.exe
    sleep                     Set beacon sleep time
    socks                     Start SOCKS4a server to relay traffic
    socks stop                Stop SOCKS4a server
    spawn                     Spawn a session 
    spawnto                   Set executable to spawn processes into
    steal_token               Steal access token from a process
    task                      Download and execute a file from a URL
    timestomp                 Apply timestamps from one file to another
    unlink                    Disconnect from parent Beacon
upload                    Upload a file

```

这里就演示几个常用的命令把

```
Getuid  获取当前用户

```

![enter image description here](http://drops.javaweb.org/uploads/images/c75166e9d02529ec6ea8c47555ac4c81cf6813ca.jpg)

Execute 运行可执行程序（不能执行shell命令）

```
Shell  执行shell命令

```

![enter image description here](http://drops.javaweb.org/uploads/images/8b5c518e5cc829879029c00ddbc51cfd1b9a657e.jpg)

```
Meterpreter  返回一个meterpreter会话

```

剩下的命令就等大家自己去看吧。

这东西好处在于比较对控制目标主机比较隐蔽，缺点在每次的命令我返回结果比较慢，在过防火墙方面还是不错的。