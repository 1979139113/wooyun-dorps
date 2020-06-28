# TruSSH Worm分析报告

最近百度X-Team捕获到一个利用SSH弱口令漏洞构建僵尸网络的蠕虫，该蠕虫具有自动扫描、自动传播、并依托公共社交网络服务作为获取Command and Control(后文简称C&C)控制信息等特点；蠕虫作者为保证控制方式的独享性，上线地址的变化性以及隐蔽性做了大量工作，C&C上线地址能够做到每天一换。根据其上线特点，我们将此蠕虫命名为：TruSSH Worm。目前此蠕虫已经在全世界范围内大规模传播。鉴于此蠕虫的编写和控制方式有些特殊，特拿出来和大家分享。

0x01 蠕虫主体特征
=====

所有的蠕虫主体执行文件均通过upx壳进行压缩，但通过破坏upx header等方式防止upx –d的自动化脱壳，这在linux类的恶意样本中并不多见。

![](http://drops.javaweb.org/uploads/images/2942883794452c98f8bbf3db42c872dce8aa5220.jpg)

手工脱壳后继续分析，整个ELF文件静态链接，并且被stripped。蠕虫支持在i686、mips、arm架构的linux上运行，能够适应在各种小型被裁剪过的路由器上进行传播。通过对脱壳后的bin中关键字符串进行查看，发现其集成了openssl、libssh2、libevent, libcurl等库，其中openssl的库给出了版本号以及发布时间等信息，让我们得知此恶意蠕虫属于被近期投放，要晚于2015年6月12日。X-Team在捕获改样本时，VT上并没有发现有过历史提交。

![](http://drops.javaweb.org/uploads/images/0e0a7a7bd98ea4a90bdbab5708f685abcf057ac9.jpg)

0x02 蠕虫传播方式
=====

蠕虫运行成功后，会开启大量的线程，随机选择生成一个B段IP段，扫描其中22端口的开放情况，成功连接的IP地址保存到名为list2列表中，当一次扫描完成后，会读取list2数据，并尝试使用事先设置好的弱口令集进行破解，该蠕虫仅仅依靠三个弱口令root:root, admin:admin 以及ubnt:ubnt三个用户名密码进行破解。其中ubnt属于近期被DDOS集团重点关注airos系统的SSH默认账户名密码。通过zoomeye和shodan，我们也可以看到全网的此类设备的量级是非常可观的。

生成IP地址列表

![](http://drops.javaweb.org/uploads/images/8ca0461c41f71e133f33303bdaace855a119bf77.jpg)

一旦破解成功，会将该IP地址的信息保存到good2文件中,并将当前目录下的所有.mod文件全部复制到远程服务器的/tmp/.xs目录下，然后设置可执行属性并依次执行这些文件。

![](http://drops.javaweb.org/uploads/images/922f0a1a1894bf52cbdfd90a1f48bca99cb0bdeb.jpg)

为了防止目标环境没有wget或者tftp等命令，这里蠕虫采用了一个比较tricky的方式方法，直接在server端使用cat > xxxx.mod的方式传送文件，下图是我们抓取到的命令执行内容：

![](http://drops.javaweb.org/uploads/images/110d733c22dffd6afadbfef29f4ac93c7ab23d7b.jpg)

蠕虫同时会监听9000和1337端口接受外界请求，其中9000端口是一个非常重要的感染标志。蠕虫周期性的会检查good2中机器的存活情况，确保感染率，在对SSH进行爆破前，蠕虫先会向9000端口发送post 请求时，如果其响应“{status: 1}” 则表示该机属于存活状态，跳过SSH密码尝试过程，如下所示:

![](http://drops.javaweb.org/uploads/images/fe49973be5f556e67343164b9adea6e8ad60d853.jpg)

9000端口还提供update和download等功能，这些功能的用途在后面会看到。

0x03 蠕虫的上线方式
=====

蠕虫利用了公共社交网络平台进行控制，并采用两阶段获取C&C IP的上线方式，这是我们之前捕获的蠕虫中没有发现过的。该蠕虫通过www.twitter.com、www.reddit.com、my.mail.ru等网站上搜索特定的信息，解析页面内容来找到第一阶段控制IP信息，来看下这个流程是怎么进行的：

![](http://drops.javaweb.org/uploads/images/4192062c405ea19231268e8f2ec1176e7a2d5cec.jpg)

通过在twitter上搜索关键字获取一阶段IP地址,其他连接www.mail.ru以及www.reddit.com的情况类似。在这些社交平台上搜索的关键字，蠕虫通过一定的算法来进行得到，有点fastflux的感觉。准备连接前，蠕虫会根据内置的算法从www.google.com返回Server Response中的Date域中提取出来的值作为变量生成随机数，再使用随机数从预先定义好的词表字典中来选择两个对应的词，然后加上随机数拼接成一个合适的url，如下所示：

![](http://drops.javaweb.org/uploads/images/5ec59a55876d9ece9236847e334c34247aaa17e6.jpg)

![](http://drops.javaweb.org/uploads/images/e517a48c810f7e630589652ca0fb99e75e21dea9.jpg)

用来构造请求的词表：

![](http://drops.javaweb.org/uploads/images/e96c99cb7a604d5a6ae1e77d4b55e9c763e0d256.jpg)

下图是逆向出来随机数生成算法的C实现

![](http://drops.javaweb.org/uploads/images/40f5e946f8c0730617706bb4269daa775a62c490.jpg)

蠕虫在收到Response code 200的返回后，在回复的页面中尝试查找base64 特征的字符串，并结合蠕虫内置的KEY，使用openssl中椭圆曲线算法（ECDSA）来验证数据的有效性。此时蠕虫会得到一个二进制文件，该文件格式如下所示。

![](http://drops.javaweb.org/uploads/images/58532a741426fe42f4037e226c9ba4ed63ee7ed9.jpg)

第一部分红线标注的即为上线IP地址，蠕虫此时会连接IP的9000端口，获取实际的上线地址，并连接到该地址，请求名为http://IP:9000/srv_report&ver=0的URL， 并从这个url中得到实际的C&C地址，下图是我们捕获到的srv_report，格式如下所示：

![](http://drops.javaweb.org/uploads/images/2409b7bd31102d1313fd55a0fa319b8e14c1cbf2.jpg)

其组成也是分成三部分，第一部分为最终的上线URL，后紧跟着一个04开头的值，然后是4位的时间戳。最后是一个32位的签名校验。在校验URL的有效性后，蠕虫会连接该URL地址，通过请求该地址，得到实际需要执行的命令。

![](http://drops.javaweb.org/uploads/images/bd646779c65d50aa60d6a4b855781068afe38338.jpg)

![](http://drops.javaweb.org/uploads/images/450de434af50701c4be68780336409f60e3e637a.jpg)

![](http://drops.javaweb.org/uploads/images/4092442769b0d33ef163962d9356d1081aa92de8.jpg)

0x04 感染范围
=====

我们根据之前提到的9000端口特征，8月底的时候在全网范围内进行了一次排查,结果如下：全球共感染主机23367台，其中中国是受影响最多的国家 达到7000+

![](http://drops.javaweb.org/uploads/images/6a70bd5b5b92aab2e928749a684c749219b74c8c.jpg)

0x05 其他
=====

在分析代码的过程中，我们发现了一些有趣的信息，蠕虫的编写者有使用truecrypt的习惯，编译在bin中的路径字符串泄露了这一点。

![](http://drops.javaweb.org/uploads/images/12ab0e32543e6abe604cdee619143899f64c53dc.jpg)

从连续监控twitter等社交网络数据来看，原作者大约是在2015年7月17 到2015年7月23 日左右放出了控制信息，之后未有新控制信息放出。