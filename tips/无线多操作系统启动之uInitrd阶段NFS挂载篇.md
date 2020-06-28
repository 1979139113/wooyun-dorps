# 无线多操作系统启动之uInitrd阶段NFS挂载篇

0x00 背景
-------

* * *

项目组在无线环境下实现了Ubuntu12.04和Android4.0系统在Pandaboard ES开发板上的无线加载、启动及切换。启动过程中操作系统内核uImage和临时根文件系统uInitrd通过wifi网络加载到SD卡中，SD卡中只有轻量级的操作系统引导程序及我们特制的meta OS，系统实际的根文件系统rootfs存储于远程的服务器上，我们使用NFS协议在无线环境下挂载实际的根文件系统完成系统的启动。目前，国内无线多操作系统引导技术文档资料较少，支持多操作系统启动的手机主要有两款：一个是西班牙手机厂商Geeksphone开发的一款代号为Revolution的手机，另外一个是谷歌发布的Nexus系列。在这分享无线多操作系统启动过程中通过NFS协议挂载远程服务器上实际文件系统的一些经验，一种可行的技术手段。目前是在局域网环境，性能有限，仅供大家参考。想分三部分来讲。

```
（1）操作系统引导程序Boot Loader的改造；
（2）系统镜像文件uImage的改造；
（3）系统启动过程中uInitrd阶段NFS挂载相关问题。

```

本文主要讲解uInitrd阶段NFS挂载相关问题，以Ubuntu12.04系统为例进行说明，Android系统先留着。

附上demo图一张：

![enter image description here](http://drops.javaweb.org/uploads/images/9a4d72442c42728fe062c34b9b4a9c3f85b4cc88.jpg)

0x01 NFS概述
----------

* * *

NFS，即Network File System网络文件系统协议。NFS协议可以透过网络，让不同的机器、不同的操作系统彼此分享文件。它与Windows系统中的文件资源共享类似，不同的是NFS是在UNIX/Linux系统下实现的。NFS协议是通过网络将用户远程服务器上的数据挂载到本地目录，从而实现远程数据本地透明获取与访问的一种方式。NFS服务器与客户端挂载示意图如下图所示：

![enter image description here](http://drops.javaweb.org/uploads/images/157c9141b4ee2fc7df762606548e4c2314f1d123.jpg)

服务器将目录`/home/gsm/ubuntu`设为共享目录，其他的客户端主机可以将该共享目录挂在指定的某个目录下。挂载成功后，就可以在挂载目录下看到与`/home/gsm/ubuntu`完全一样的子目录和文件。

要完成uInitrd阶段NFS的挂载，首先了解Linux系统的启动流程。

0x02 Linux系统启动流程分析
------------------

* * *

```
（1）硬件加电自检、初始化；
（2）读取配置文件并执行操作系统引导程序Boot Loader；
（3）读取Boot Loader的参数配置文件boot.scr，将内核uImage加载到内存中执行，uImage开始检测硬件与加载驱动程序；
（4）uImage执行完成后，uInitrd加载到内存中执行，uInitrd解压调用init脚本。注：uInitrd是Linux系统启动过程中使用的临时根文件系统，在Linux启动之前，Boot Loader会将它加载到内存中，内核启动的时候会将这个文件解开作为根文件系统使用；
（5）init脚本创建相应的文件目录，挂载到相应的文件系统。最后完成交权，切换到系统实际的文件系统。

```

由Linux系统的启动流程可知要实现智能移动终端通过无线网络将操作系统内核镜像文件uImage和临时根文件系统uInitrd加载到终端上执行，并在uInitrd阶段通过NFS协议挂载服务器上的实际根文件系统rootfs，完成系统交权、启动。首先要完成的是Boot Loader的改造（这里略过不讲），Boot Loader读取配置文件将对应系统的uImage和uInitrd加载到内存中执行。然后，需要对系统内核uImage进行改造（略过不讲），最后需要对uInitrd进行改造（也就是本文重点要介绍的）。

主要包括三方面：

```
（1）uInitrd无线模块的添加；
（2）uInitrd启动阶段init脚本的修改；
（3）NFS挂载脚本以及无线配置函数等相关函数的定义。

```

0x03 NFS无线挂载实现
--------------

* * *

### Step 1:

Init脚本中有这么几行代码：

```
  maybe_break mount  
   log_begin_msg "Mounting root file system..."  
   . /scripts/${BOOT}  
   parse_numeric ${ROOT}  
   mountroot  

```

Linux系统本地启动时，/scripts/${BOOT}中的BOOT参数默认为local，所以当系统改为NFS挂载远程文件系统启动时，BOOT参数要改为nfs。将/scripts/local 改为/scripts/nfs，即本地启动改为NFS挂载启动。

### Step 2:

Init脚本中添加wifi配置函数wifi_networking，函数在/scripts/functions文件中定义。

```
wifi_networking()
{
    wpa_supplicant -B -iwlan0 -c /etc/wpa_supplicant/wpa_supplicant.conf
    ifconfig wlan0 192.168.1.99 netmask 255.255.255.0 
    route add default gw 192.168.1.1
}

```

### Step 3:

修改scripts目录下nfs文件，指定nfsmount目录，设定挂载参数

### Step 4：

在etc/wpa_supplicant/wpa_supplicant.conf中设定wifi参数

### Step 5：

rootfs中配置无线参数；

### Step 6：

当执行到init脚本最后一行代码时run-init ${rootmnt} ${init}，系统的启动交给将要进入的系统的${init}，并输出，完成系统的交权，这里我们将run-init修改为/sbin/init。

0x04 NFS协议分析
------------

* * *

目前制约性能的一些主要因素。在启动过程中通过wireshark抓包分析可以知道，当终端与服务器的连接出现问题的时候，比如无线网络突然出现问题的时候，终端将不断地尝试与服务器重新建立连接直到与服务器的连接恢复正常。在每次尝试重连不成功后，连接的时间间隔将成指数级增长。（这里我们使用的NFS协议版本为V3，使用的传输协议为TCP/IP协议）

![enter image description here](http://drops.javaweb.org/uploads/images/e5a37c6661abe8bdb64d28489bf2f50a3cd2f701.jpg)

除此之外，在无线网络环境下，数据块丢包是经常出现的，如果出现了一个丢包，这丢失的数据块会被重传。无线网络的不稳定性增加了数据块丢失率。在这段期间，NFS客户端将不断地向NFS服务器请求重传丢失的块直至客户端成功收到丢失的块。（以太网有最大传输单元MTU的限制，为1500字节，所以数据传输过程会进行分块）

![enter image description here](http://drops.javaweb.org/uploads/images/1a55af7e45782c98d15a3ef891a28d34eb92d868.jpg)

![enter image description here](http://drops.javaweb.org/uploads/images/239f0c5a8156e442f1f475514988392d453b25af.jpg)

对NFS协议有一定研究的欢迎交流，包括NFS协议改进，并发环境以及NFS读写块大小，传输协议使用等参数设置问题。了解pNFS协议的求指导。