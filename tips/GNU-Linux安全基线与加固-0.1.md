# GNU/Linux安全基线与加固-0.1

**from:https://raw.githubusercontent.com/citypw/DNFWAH/master/4/d4_0x02_DNFWAH_gnu-linux_security_baseline_hardening.txt**

By Shawn

0x00 关于这份文档
-----------

* * *

随着GNU/Linux在各个行业的IT基础架构中的普及，安全问题也成为了关注的焦点， GNU/Linux主要是由GNU核心组建( 编译器GCC, C库Glibc等)和Linux内核组合而成， 在自由开源软件统治着基础平台的大环境下，不少人认为开源一定是安全的，这 是一种不完全正确的观念，Coverity的报告只是说明了开源比闭源更安全，这并 不代表自由开源软件就是牢不可破的，自由开源软件在一定程度上具有一些安全 的特性，这些特性不一定在GNU/Linux发行版中是默认开启，这些特性中有一些是 必须在安全基线中去部署的，有一些可以根据具体需求来定制，这篇文档主要是 介绍一些关于安全基线建设和加固的基本内容。

在实际的安全咨询工作中，很多普通个人用户和企业用户并不是安全领域的黑客， 大多客户都会要求给出一份简单易懂的部署文档，也就是所谓的安全基线，基线 和加固是很大的话题，我会尽力不断更新这篇文档的内容，也希望有社区的朋友 参与，本文所使用的GNU/Linux发行版是Debian。

0x01 安全基线
---------

* * *

在遵循最小安装和最小权限的部署原则下，还有一些地方是需要注意的，我们把 这些部分称为安全基线。

**1.1 安全修复更新**

作为系统管理员，每天干的第1件事情就应该是查看生产环境的机器是否有安全修 复的更新，甚至应该除了生产环境再另外跑一套模拟环境，用于测试升级后是否 影响业务系统。

Debian检查需要安全修复包：

```
sudo apt-get upgrade -s | grep -i security

```

OpenSuSE发行版检查需要安全修复的包：

```
sudo zypper lp | awk '{ if ($7=="security"){ if ($11=="update") {print $13} else{ print $11 }}}' | sed 's/:$//' | grep -v "^$" | sort | uniq

```

**1.2 密码策略**

root密码策略至少应该考虑以下几点：

1，密码强度，至少是字母+数字一共9位以上

2，不同的系统密码不能一样

3，根换密码策略(每90天更换一次）

4，密码分发管理，管理不同业务服务器的系统管理员掌握不同的密码

**1.2.1 业务分离**

生产环境中,不同的业务可以做水平分离,比如把不同的服务运行到不同的虚拟机 中,不需要远程访问的服务器可以绑定到 localhost( 比如只需要访问本机业务的 mysql) 上。

**1.3 SSH安全配置**

openssh目前的默认配置文件相比以前虽然要安全的多，但还是有必要对生产系统 中的ssh服务器进行基线检查。

配置文件：`/etc/ssh/ssh_config`
--------------------------

1，known_hosts保存相关服务器的签名，所以必须把主机名hash:

```
HashKnownHosts yes

```

2，SSH协议v1不安全：

```
Protocol 2

```

3，如果没用X11转发的情况：

```
X11Forwarding no

```

4，关闭rhosts:

```
IgnoreRhosts yes

```

5，关闭允许空密码登录：

```
PermitEmptyPasswords no

```

6，最多登录尝试次数：

```
MaxAuthTries 5

```

7，禁止root登录

```
PermitRootLogin no

```

* * *

(可选)
----

1，关闭密码认证，启用公钥认证：

```
PubkeyAuthentication yes

PasswordAuthentication no

```

2，允许或者禁止用户/组登录:

AllowGroups, AllowUsers, DenyUsers, DenyGroups
----------------------------------------------

**1.4 auditd审计框架**

auditd是接收内核审计模块关于系统调用信息的一个用户态程序，可以通过一些 规则来对一些系统调用或者文件目录进行监控。

安装auditd:

```
sudo apt-get install auditd

```

配置文件:`/etc/audit/auditd.conf`

存储地址`log_file = /var/log/audit/audit.log`

审计规则的配置文件：`/etc/audit/audit.rules`，这里给出一个例子：
--------------------------------------------

```
# First rule - delete all
-D

# Increase the buffers to survive stress events.
# Make this bigger for busy systems
-b 320
-a always,exit -S adjtimex -S settimeofday -S stime -k time-change
-a always,exit -S clock_settime -k time-change
-a always,exit -S sethostname -S setdomainname -k system-locale
-w /etc/group -p wa -k identity
-w /etc/passwd -p wa -k identity
-w /etc/shadow -p wa -k identity
-w /etc/sudoers -p wa -k identity
-w /var/run/utmp -p wa -k session
-w /var/log/wtmp -p wa -k session
-w /var/log/btmp -p wa -k session
-w /etc/apparmor -p wa -k MAC-policy
-w /etc/apparmor.d -p wa -k MAC-policy

```

* * *

上面的例子对一系列关于时间的系统调用进行了监控，一旦时间出现改变就会记 录进如日志，之后对几个跟创建/删除用户和组的文件也进行了监控，最后是对 apparmor的配置文件和规则目录进行监控。在/etc/apparmor下的shell中输入：

```
sudo touch hello

```

用工具ausearch来进行查询：

* * *

```
sudo ausearch -i -k MAC-policy

```

* * *

```
type=CONFIG_CHANGE msg=audit(07/20/2014 20:36:48.397:58) : auid=root ses=2 op="add rule" key=MAC-policy list=exit res=1 

```

* * *

```
type=CONFIG_CHANGE msg=audit(07/20/2014 20:36:48.445:59) : auid=root ses=2 op="add rule" key=MAC-policy list=exit res=1 

```

* * *

```
type=PATH msg=audit(07/20/2014 20:38:42.717:61) : item=1 name=hello inode=799889 dev=08:01 mode=file,644 ouid=root ogid=root rdev=00:00 nametype=CREATE 
type=PATH msg=audit(07/20/2014 20:38:42.717:61) : item=0 name=/etc/apparmor inode=783766 dev=08:01 mode=dir,755 ouid=root ogid=root rdev=00:00 nametype=PARENT 
type=CWD msg=audit(07/20/2014 20:38:42.717:61) :  cwd=/etc/apparmor 
type=SYSCALL msg=audit(07/20/2014 20:38:42.717:61) : arch=i386 syscall=open success=yes exit=3 a0=bfeca8ab a1=8941 a2=1b6 a3=1 items=2 ppid=17704 pid=17876 auid=root uid=root gid=root euid=root suid=root fsuid=root egid=root sgid=root fsgid=root tty=pts1 ses=2 comm=touch exe=/bin/touch key=MAC-policy 

```

* * *

```
type=PATH msg=audit(07/20/2014 20:38:56.017:62) : item=1 name=hello inode=799889 dev=08:01 mode=file,644 ouid=root ogid=root rdev=00:00 nametype=DELETE 
type=PATH msg=audit(07/20/2014 20:38:56.017:62) : item=0 name=/etc/apparmor inode=783766 dev=08:01 mode=dir,755 ouid=root ogid=root rdev=00:00 nametype=PARENT 
type=CWD msg=audit(07/20/2014 20:38:56.017:62) :  cwd=/etc/apparmor 
type=SYSCALL msg=audit(07/20/2014 20:38:56.017:62) : arch=i386 syscall=unlinkat success=yes exit=0 a0=ffffff9c a1=89eb8c0 a2=0 a3=0 items=2 ppid=17704 pid=17889 auid=root uid=root gid=root euid=root suid=root fsuid=root egid=root sgid=root fsgid=root tty=pts1 ses=2 comm=rm exe=/bin/rm key=MAC-policy 

```

* * *

也可以使用aureport来生成报告。

**1.4.1 针对文件的审计**

1, Leo Juranic的详细的分析[]了异常的通配符威胁有多大，：
------------------------------------

```
find / -path /proc -prune  -name "-*"

```

* * *

2, 所谓的world-writable权限的文件是不太合理的，所以这种文件我们必须得提防：
----------------------------------------------

```
find / -path /proc -prune -o -perm -2 ! -type l -ls

```

* * *

3, 一个没有owner的文件是存在潜在威胁的，因为你永远也不知道未来某个时候她的

uid/gid成为了你的敌人：
---------------

```
find / -path /proc -prune -o -nouser -o -nogroup

```

* * *

4, 作为"自主可控"的自由软件用户，你得知道你的生产环境中哪些用户是可用的：
---------------------------------------

```
egrep -v '.*:\*|:\!' /etc/shadow | awk -F: '{print $1}'

```

* * *

5, 你要删除一个用户前，应该先了解一些有哪些文件是他拥有的： ---||||---||||---||||---||||---||||---||||---||||---||||---||||---||||---||||---||||---||||---||||---||||---||||---||||---||||---||||---||||--x

```
find / -path /proc -prune -o -user account -ls

```

然后，安全的删除：

```
userdel -r account

```

* * *

6, 如果不带':x:'的用户肯定是无法正常使用的：
--------------------------

```
grep -v ':x:' /etc/passwd

```

* * *

7, /boot目录权限下至少是644，甚至是600:
---------------------------

```
ls -l /boot

```

* * *

(可选)

1, GNU/Linux的访问控制列表(ACL)也是不错的文件权限管理的途径，获得文件

ACL的信息：
-------

```
getfacl file

```

设置哪些用户对哪些文件有什么样的权限：

setfacl -m u:user:r file
------------------------

0x02 内核安全基线
-----------

* * *

SYN cookies防护主要是为了防止SYN洪水攻击，开启设置为1：
-----------------------------------

```
net.ipv4.tcp_syncookies = 1
/proc/sys/net/ipv4/tcp_syncookies

```

(可选)，如果需要开启SYNPROXY可以直接：

```
iptables -t raw -A PREROUTING -i eth0 -p tcp --dport 80 --syn -j NOTRACK
iptables -A INPUT -i eth0 -p tcp --dport 80 -m state UNTRACKED,INVALID \
     -j SYNPROXY --sack-perm --timestamp --mss 1480 --wscale 7 --ecn

echo 0 > /proc/sys/net/netfilter/nf_conntrack_tcp_loose

```

注意:SYNPROXY是在3.13里加入的NETFILTER特性。
---------------------------------

源路由通常可以用于在IP包的OPTION里设置途经的部分或者全部路由器，这个 特性可以用于网络排错和优化，比如traceroute，攻击者也可以使用这个特性来

进行IP欺骗，关闭设置为0：
--------------

```
net.ipv4.conf.all.accept_source_route = 0
/proc/sys/net/ipv4/conf/all/accept_source_route

```

* * *

ICMP重定向，正常用于选择最优路径，攻击者可以利用开展中间人攻击，关闭设

置为0:
----

```
net.ipv4.conf.all.accept_redirects = 0
/proc/sys/net/ipv4/conf/all/accept_redirects

```

* * *

IP欺骗防护，启动设置为1：
--------------

```
net.ipv4.conf.all.rp_filter = 1
/proc/sys/net/ipv4/conf/all/rp_filter

```

* * *

忽略ICMP请求( PING)，启动设置为1：
-----------------------

```
net.ipv4.icmp_echo_ignore_all = 1
/proc/sys/net/ipv4/icmp_echo_ignore_all

```

* * *

忽略ICMP广播请求，启动设置为1：
------------------

```
net.ipv4.icmp_echo_ignore_broadcasts = 1
/proc/sys/net/ipv4/icmp_echo_ignore_broadcasts

```

* * *

错误消息防护，会警告你关于网络中的ICMP异常，启动设置为1：
-------------------------------

```
net.ipv4.icmp_ignore_bogus_error_responses = 1
/proc/sys/net/ipv4/icmp_ignore_bogus_error_responses

```

* * *

对特定packet(IP欺骗，源路由，重定向）进行审计，启动设置为1：
-----------------------------------

```
/proc/sys/net/ipv4/conf/all/log_martians
net.ipv4.conf.all.log_martians = 1

```

* * *

地址随机化，启动设置为2：
-------------

```
kernel.randomize_va_space=2
/proc/sys/kernel/randomize_va_space

```

* * *

内核符号限制访问，启动设置为1：
----------------

```
kernel.kptr_restrict=1
/proc/sys/kernel/kptr_restrict

```

类似CVE-2014-0196的exploit对于这一项就很难做到“通杀”;-)
----------------------------------------

内存映射最小地址，启动设置为65536：
--------------------

```
vm/mmap_min_addr=65536
/proc/sys/vm/mmap_min_addr

```

* * *

**Apparmor**

安装Apparmor和社区规则：

```
sudo apt-get install -y apparmor-profiles apparmor

```

查看状态是否运行正常：

```
sudo aa-status

```

* * *

0x03 加固
-------

* * *

安全基线是在防御已知的威胁，而加固则是侧重于防御未知的威胁，加固的主要 目的是增加攻击者的成本。

**3.1 内核加固 - Grsecurity/PaX**

GNU/Linux平台从用户空间到内核空间都有一系列的加固措施，但是,对于真正面 临复杂安全环境,确实需要高级安全防护能力的机构,最极端的加固防御是: Grsecurity/PaX，对于注重完美的“老派( old school )”黑客社区而言,没有 Grsecurity/PaX 的定制方案是不完美的 ,这里摘选几段old school社区的调侃：

> "The "better than none" point of view is actually a nice way to false sense of security for those who don't know better. We got better-than-none apparmor, selinux, tomoyo, some poorly maintained and crippled ports of grsec features or alikes, namespaces and containers, rootkit-friendly LSM, the dumb and useless kernel version of SSP, etc. What's the sum of all this shit for end users? False sense of security..."
> 
> "without Grsecurity/PaX, linux security is like monkey can never perform a regular masturbation cu'z lacking of giant pennis;-)"

作为个人完全赞同以上观点，太多的better-than-none可以算是"甜点"，用户使 用后会觉得自high的很爽，但是到了夜幕降临的时候，用户还是很难入睡，因为 "痛点"依然在那里，在安全领域，"甜点"就是"安全感"，安全感绝对不等同于安 全。

Anyway，对于商业客户而言，是否对Grsecurity/PaX定制是一种选择。

安装新的带Grsecurity/PaX补丁的内核，以3.14.1为例，先下载原生内核：
-------------------------------------------

https://www.kernel.org/pub/linux/kernel/v3.x/linux-3.14.1.tar.xz

下载Grsecurity/PaX补丁： https://github.com/citypw/citypw-SCFE/raw/master/security/apparmor_test/grsecurity-3.0-3.14.11-201407072045.patch

解压内核然后打补丁：

```
Patch the kernel with grsecurity:
xz -d linux-3.14.1.tar.xz
tar xvf linux-3.14.1.tar
cd linux-3.14.1/
patch -p1 < ../grsecurity-3.0-3.14.3-201405121814.patch

```

你可以在"make menuconfig"里自己定制符合你口味的内核，也可以使用我测试用 的内核config文件： https://raw.githubusercontent.com/citypw/citypw-SCFE/master/security/apparmor_test/debian-7.4-linux-3.14.1-grsec.config

编译内核(-jx, x通常==你的CPU核数+1):

```
make -j7 deb-pkg

```

安装编译后的内核:

```
dpkg -i ../*.deb

```

* * *

现在内核的部分结束，关于RBAC规则可以使用用户态工具gradm的Learning Mode 来实现，但不在本文的讨论范畴。

**3.2 PHP加固**

1, PHP是常用的WEB开发语言，在WEB生产环境部署的过程中，目录和文件的权限是需 要关注的，通常除了少数用途的目录（比如上传）外，都应该把写入权限禁用：

* * *

```
find -type f -name \*.php -exec chmod 444 {} \;
find -type d -exec chmod 555 {} \;

```

* * *

2, 开启 php 的安全模式,禁用 php 不安全的函数等加固，修改 php 配置文件 /etc/php5/apache2/php.ini :

* * *

// 设置模式为安全模式,此值直接影响 disable_functions 的命令是否生效; [SQL]

```
; http://php.net/sql.safe-mode

sql.safe_mode = On

```

// 禁用不安全的函数

```
disable_functions = system, show_source, symlink, exec, dl, shell_exec,
passthru, phpinfo, escapeshellarg, escapeshellcmd

```

// 避免暴露 php 信息

```
expose_php = Off

```

// 关闭错误信息提示

```
display_errors = Off

```

// 不允许调用 dl

```
enable_dl = Off

```

// 避免远程调用文件

```
allow_url_include = Off

```

* * *

0x04 Reference

* * *

[1] 开源闭源项目代码质量对比 http://www.solidot.org/story?sid=39173

[2] Back To The Future: Unix Wildcards Gone Wild http://www.defensecode.com/public/DefenseCode_Unix_WildCards_Gone_Wild.txt

[3] SYNPROXY：廉价的抗DoS攻击方案 www.solidot.org/story?sid=38791

[4] INTERNET PROTOCOL http://tools.ietf.org/html/rfc791

[5] A simple TCP spoofing attack http://www.citi.umich.edu/u/provos/papers/secnet-spoof.txt

[6] ICMP Attacks Illustrated http://www.sans.org/reading-room/whitepapers/threats/icmp-attacks-illustrated-477

[7] SUSE Linux Enterprise Server 11 SP3 - Security and Hardening https://www.suse.com/documentation/sles11/singlehtml/book_hardening/book_hardening.html

[8] Securing Debian Manual https://www.debian.org/doc/manuals/securing-debian-howto/

[9] A Brief Introduction to auditd http://security.blogoverflow.com/2013/01/a-brief-introduction-to-auditd/

[10] Apparmor RBAC http://wiki.apparmor.net/index.php/Pam_apparmor_example

[11] Hardening PHP from php.ini http://www.madirish.net/199

[12] CVE-2014-0196 exploit http://bugfuzz.com/stuff/cve-2014-0196-md.c