# 劫持GPS定位&劫持WIFI定位

0x00 概述
-------

* * *

在刚刚举行的Black Hat Europe 2015大会上，来自阿里移动安全的Wang Kang & Shuhua Chen & Aimin Pan 展示了劫持GPS定位以及劫持WIFI定位的技术

于是我用朋友的Hackrf试了一下

0x01 GPS劫持步骤
------------

* * *

在Ubuntu 15.10中安装gnuradio以及hackrf工具

```
apt-get install gnuradio gr-osmosdr hackrf

```

请注意，这里目前无法使用kali的hackrf软件包，因为需要-R选项来进行repeat，而kali暂时没有更新hackrf软件包至最新

编译开源的gps模拟工具

```
git clone https://github.com/osqzss/gps-sdr-sim.git
cd gps-sdr-sim
make

```

设置经纬度并生成数据样本

```
 ./gps-sdr-sim -e brdc3540.14n -l 30.286502,120.032669,100

```

然后插上Hackrf，开始伪造GPS信号

```
hackrf_transfer -t gpssim.bin -f 1575420000 -s 2600000 -a 1 -x 0 -R 

```

测试时发现由于Hackrf本身做工问题，只能坚持连续工作5分钟，之后噪声过大导致信噪比无法满足GPS定位的要求，所以大家在测试时一定要等到准备完毕再插上Hackrf，执行命令后立即查看效果

效果如图所示：

![enter image description here](http://drops.javaweb.org/uploads/images/e5b42ef9cf56b42f973c543def7a44bfb3182137.jpg)

所有的卫星GPS都是伪造出来的，因此也比较整齐，不够整齐是因为山寨的Hackrf硬件原因

![enter image description here](http://drops.javaweb.org/uploads/images/c15c781c215988e2f2fb6e59b46a1ecab119252c.jpg)

这里的坐标我随便定的，定在吉林省了，上文中给出的坐标是杭州

0x02 WIFI定位劫持步骤
---------------

* * *

安装aircrack系列工具

apt-get install aircrack-ng -y

安装mdk3

```
wget ftp://ftp.hu.debian.org/pub/linux/distributions/gentoo/distfiles/mdk3-v6.tar.bz2
tar -jxvf mdk3-v6.tar.bz2
cd mdk3-v6

```

这里需要修改一下Makefile

-lpthread 改成 -pthread

之后再编译安装

```
make
make install

```

把以下内容保存为wifi-mdk3.awk

```
$1 == "BSS" {
 MAC = $2
 wifi[MAC]["enc"] = "Open"
}
$1 == "SSID:" {
 wifi[MAC]["SSID"] = $2
}
$1 == "freq:" {
 wifi[MAC]["freq"] = $NF
}
$1 == "signal:" {
 wifi[MAC]["sig"] = $2 " " $3
}
$1 == "WPA:" {
 wifi[MAC]["enc"] = "WPA"
}
$1 == "WEP:" {
 wifi[MAC]["enc"] = "WEP"
}
END {
 for (BSSID in wifi) {
 printf "%s %s\n",BSSID,wifi[BSSID]["SSID"]
 }
}

```

然后扫描wifi（wlan0为无线网卡interface，我自己的是wlp3s0）

```
sudo iw wlan0 scan |awk -f wifi-mdk3.awk > result.txt

```

然后是带着电脑换个没有WIFI的地方，因为要伪造WIFI定位，在自己的地点伪造自己没有效果……而如果身边的真实WIFI过多则会干扰伪造的可信度

开启无线网卡的monitor模式，wlan0是无限网卡的interface

```
sudo airmon-ng check kill
sudo airmon-ng start wlan0

```

然后用mdk3伪造刚才iw scan到ssid 的信号，wlan0-mon是用airmon-ng开出的monitor interface

```
sudo mdk3 wlan0-mon b -v result.txt

```

如果身边没有其他的WIFI，伪造开始之后就可以欺骗WIFI定位了，不过如果身边有其他正常WIFI时，一般情况下很难伪造成功

0x03 GPS定位与劫持原理
---------------

* * *

这里是用Hackrf伪造的GPS信号

以下翻译内容摘自https://www.blackhat.com/docs/eu-15/materials/eu-15-Kang-Is-Your-Timespace-Safe-Time-And-Position-Spoofing-Opensourcely-wp.pdf 有删减

![enter image description here](http://drops.javaweb.org/uploads/images/a98a6bb5120e44b2b3c13ead99b22610772b5886.jpg)

1) GPS定位原理

首先，让我们明确我们的需求。我们想要知道的是我们的位置坐标(x,y,z)，如果从一个已知坐标(x1,y1,z1)的点A（这个点在现实情况下是卫星）广播一个信号，比如说光和声音或者电磁波，然后我们试着去测量信号发送至到达的时间差τ1（在gps系统中我们用的是电磁波，我们知道它的速度），然后我们就能得出下面的等式：

![enter image description here](http://drops.javaweb.org/uploads/images/b24745096241354df0039f7b1fd62a71733256a0.jpg)

这个等式有3个未知变量，因此单单一个等式解不出来，我们可以再加两个已知位置的点（卫星），我们把它们记作(x2,y2,z2) 和 (x3,y3,z3)，然后就是下面的方程组

![enter image description here](http://drops.javaweb.org/uploads/images/a1b80b4b724b3f38aba73f68068475def1507c36.jpg)

现在我们就能解出我们的位置(x,y,z)了

但在工程应用中这样还不够。为了测量电磁波发送至到达的时间差τ1，需要在电磁波发送的时候写一个时间戳t1，然后是卫星上的时钟时间参考值，当信号到达我们这里时，我们提取出时间戳t1，然后计算t1和当地时间t2的差值来计算时间差τ1。然而当地时间和卫星时间并不是同步的，会出现一个时间偏移量∆t1，所以这个时间偏移量也要被考虑进去，于是修正后的方程式如下所示：

![enter image description here](http://drops.javaweb.org/uploads/images/f01aa9fdf0dba057a0c715e040a32dbe1f5c388b.jpg)

译者注：所以有4个变量，就需要4个卫星来创造4个等式啦，以下高等数学内容略，以上内容说明我们需要伪造至少4颗卫星的信号才能使gps定位

2) GPS信号帧：GPS信号帧的结构如图所示：

![enter image description here](http://drops.javaweb.org/uploads/images/e2cfe903c4dbce32087e23bb446c881bd1cdeb9a.jpg)

GPS信号的比特率为50 bps。GPS卫星在不同的频率和幅度上广播GPS信号，民用最常用的是L1 信号。

GPS信号的强度非常弱，大约在 -130 dBm 左右，并且大多数GPS接收器在室内不起作用，这就使得GPS信号可以轻易地被嗅探或伪造，至少攻击者不用为了盖过真实的卫星信号去发射大功率信号了

3) BRSC 数据：BRSC（广播星历数据）文件包含了每天独特的GPS卫星ephemeris数据，Ephemeris数据提供了每颗卫星确切的位置信息(xi(t),yi(t),zi(t))，所以接收者就可以以此计算位置数据。你可以以RINEX (与接收器无关的交互格式)格式从ftp://cddis. gsfc.nasa.gov/gnss/data/daily/下载BRDC档案

这些档案按照如下格式命名（如下表所示）：

![enter image description here](http://drops.javaweb.org/uploads/images/f02448b9958111ea8aa2468828d41113c53f2c75.jpg)

举个例子：‘brdc3540.14n’意味着2014年的12月20号

译者注：之后是介绍开源项目以及各种sdr平台的使用，由于我只有hackrf，所以把自己用hackrf实际测试的过程和结果放在了0x01部分，如果您有其他sdr平台，请参照原作者文章进行实际测试

0x04 WIFI定位劫持原理
---------------

* * *

由于GPS定位在室内无法使用，各种定位平台厂商比如苹果地图、谷歌地图、百度地图通常会使用WIFI信号来帮助用户获得更好的定位

原理很简单，手机的无线芯片可以提供周围热点的扫描结果，最能用来定位的关键信息是SSID和BSSID，SSID是热点的名称，而BSSID则是AP的MAC地址。定位平台厂商搜集SSID及BSSID和GPS数据一起存入他们的数据库，有些时候信息搜集是通过终端用户的手机来实现的，比如在苹果的位置服务问答上写着：

“相反，我们维护了一个在您当前位置附近Wi-Fi热点和基站的数据库，其中一些可能离你的iPhone一百多英里远，来帮助你的iPhone快速准确地计算出当前的位置。如果只用手机GPS来做卫星定位，最多可能会用到几分钟的时间。 iPhone可以缩短这个时间，在短短的几秒钟内通过使用Wi-Fi热点和基站数据快速定位到第二颗GPS卫星，甚至只使用Wi-Fi热点和基站数据进行三角测量来获取其位置，尤其是GPS不可用（如室内或地下室）的时候特别有用。这些计算使用的是由数以千万计的iPhone发送的匿名和加密的附近的Wi-Fi热点和基站的地理标记位置，以产生Wi-Fi热点和基站数据的数据库。 ”

所以，我们要是伪造一些SSID和BSSID会发生什么呢？最简单的方式当然是手动扫描出一堆热点的信心，然后买一大堆路由器来伪造出这些热点，但是这么弄难度有些大，所以我们需要一个更好的办法。

译者注：就是伪造一堆SSID名称和MAC地址，然后这个列表上传到地图数据库作对比，然后就伪造定位了，原文下面是具体的iw scan和mdk3的介绍，就不全部翻译了，经本人实际测试的过程和结果如前文0x02所示

译者再注：要想伪造成功，你伪造出的SSID必须要远远多于正常SSID数目，也就是说你在一个能够WIFI定位的地方要想伪造定位基本不可能成功，因为身边正常的SSID太多了，不过你仍然可以通过手机WIFI连接菜单看到我们伪造出的SSID名称，能看到名称就说明实验原理成功了

0x05 参考文献&致谢
------------

* * *

《Time and Position Spoofing with Open Source Projects》 Kang Wang ，Shuhua Chen ，Aimin Pan

[1](http://drops.wooyun.org/wp-content/uploads/2015/11/16.jpg)Dong L. IF GPS signal simulator development and veriﬁcation[M]. National Library of Canada= Bibliothque nationale du Canada, 2005.

[2](http://drops.wooyun.org/wp-content/uploads/2015/11/22.jpg)Akos, D. M. (1997), A Software Radio Approach To Global Navigation Satellite System Receiver Design, Dissertation, Ohio University.

[3](http://drops.wooyun.org/wp-content/uploads/2015/11/320.png)Kaplan, E. D. (1996), Understanding GPS, Principles and Applications, Boston: Artech House, Inc.

[4](http://drops.wooyun.org/wp-content/uploads/2015/11/419.png)https://play.google.com/store/apps/details?id=com.chartcross.gpstest

[5](http://drops.wooyun.org/wp-content/uploads/2015/11/513.png)http://www.pseudocode.info/post/50127404555/ beacons-beacons-everywhere-using-mdk3-for-ssid

[6](http://drops.wooyun.org/wp-content/uploads/2015/11/610.png)https://github.com/osqzss/gps-sdr-sim

[7](http://drops.wooyun.org/wp-content/uploads/2015/11/79.png)https://en.wikipedia.org/wiki/Global Positioning System

[8](http://drops.wooyun.org/wp-content/uploads/2015/11/87.png)https://en.wikipedia.org/wiki/GPS signals