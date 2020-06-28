# WireShark黑客发现之旅（5）—扫描探测

**作者：**Mr.Right、K0r4dji**申明：**文中提到的攻击方式仅为曝光、打击恶意网络攻击行为，切勿模仿，否则后果自负。

0x00 简单介绍
=====

“知己知彼，百战不殆。”扫描探测，目的就是“知彼”，为了提高攻击命中率和效率，基本上常见的攻击行为都会用到扫描探测。

扫描探测的种类和工具太多了，攻击者可以选择现有工具或自行开发工具进行扫描，也可以根据攻击需求采用不同的扫描方式。本文仅对`Nmap`常见的几种扫描探测方式进行分析。如：地址扫描探测、端口扫描探测、操作系统扫描探测、漏洞扫描探测（不包括`Web`漏洞，后面会有单独文章介绍`Web`漏洞扫描分析）。

0x01 地址扫描探测
=====

地址扫描探测是指利用`ARP`、`ICMP`请求目标网段，如果目标网段没有过滤规则，则可以通过回应消息获取目标网段中存活机器的`IP`地址和`MAC`地址，进而掌握拓扑结构。

如：`192.1.14.235`向指定网段发起`ARP`请求，如果`IP`不存在，则无回应。

![enter image description here](http://drops.javaweb.org/uploads/images/e1948c98d1394c97934fcb19d63e6be8b6c4d5d0.jpg)

如果`IP`存在，该`IP`会通过`ARP`回应攻击`IP`，发送自己的`MAC`地址与对应的`IP`。

![enter image description here](http://drops.javaweb.org/uploads/images/a48eaafe807a4e62652bc0ca67901ef7046b4084.jpg)

`ARP`欺骗适用范围多限于内网，通过互联网进行地址扫描一般基于`Ping`请求。

如：`192.1.14.235`向指定网段发起`Ping`请求，如果`IP`存在，则返回`Ping reply`。

![enter image description here](http://drops.javaweb.org/uploads/images/452ccd860603de61cf1b1fffdfe2025921f21603.jpg)

![enter image description here](http://drops.javaweb.org/uploads/images/a75556eaa3b5afab14be7ece33b48545fbdbdc54.jpg)

0x02 端口扫描探测
=====

端口扫描是扫描行为中用得最多的，它能快速获取目的机器开启端口和服务的情况。常见的端口扫描类型有全连接扫描、半连接扫描、秘密扫描和`UDP`扫描。

### 1、全连接扫描

全连接扫描调用操作系统提供的`connect()`函数，通过完整的三次`TCP`连接来尝试目标端口是否开启。全连接扫描是一次完整的TCP连接。

**1）如果目标端口开启**攻击方：首先发起`SYN`包；

目标：返回`SYN ACK`；

攻击方：发起`ACK`；

攻击方：发起`RST ACK`结束会话。

**2）如果端口未开启**攻击方：发起`SYN`包；

目标：返回`RST ACK`结束会话。

如：`192.1.14.235`对`172.16.33.162`进行全连接端口扫描，首先发起`Ping`消息确认主机是否存在，然后对端口进行扫描。

![enter image description here](http://drops.javaweb.org/uploads/images/dd4204bfbc87466646f735c401bfaa69ecd739d8.jpg)

下图为扫描到`TCP3389`端口开启的情况。

![enter image description here](http://drops.javaweb.org/uploads/images/f9ab145fdda86118f94990271d641ca2f4f0c13a.jpg)

下图为扫描到`TCP1723`端口未开启的情况。

![enter image description here](http://drops.javaweb.org/uploads/images/e37fcd2f0e0f47d181a550388a3d5e134b634fc6.jpg)

### 2、半连接扫描

半连接扫描不使用完整的`TCP`连接。攻击方发起`SYN`请求包；如果端口开启，目标主机回应`SYN ACK`包，攻击方再发送`RST`包。如果端口未开启，目标主机直接返回`RST`包结束会话。

如：`192.1.14.235`对`172.16.33.162`进行半连接端口扫描，首先发起`Ping`消息确认主机是否存在，然后对端口进行扫描。

![enter image description here](http://drops.javaweb.org/uploads/images/7e79feecc18f1c463dfe8fe29ccf8893c507b0c5.jpg)

扫描到`TCP80`端口开启。

![enter image description here](http://drops.javaweb.org/uploads/images/8f8568fe5e5361186fadf916021fbd5c4a973242.jpg)

`TCP23`端口未开启。

![enter image description here](http://drops.javaweb.org/uploads/images/86b287f5be93f96d1d47614b10f5d1943dc68dcd.jpg)

### 3、秘密扫描TCPFIN

`TCP FIN`扫描是指攻击者发送虚假信息，目标主机没有任何响应时认为端口是开放的，返回数据包认为是关闭的。

如下图，扫描方发送`FIN`包，如果端口关闭则返回`RST ACK`包。

![enter image description here](http://drops.javaweb.org/uploads/images/fa96cf629bf0550bacafeadb65f162f9f3e293fa.jpg)

### 4、秘密扫描TCPACK

`TCP ACK`扫描是利用标志位`ACK`，而`ACK`标志在`TCP`协议中表示确认序号有效，它表示确认一个正常的`TCP`连接。但是在`TCP AC`K扫描中没有进行正常的`TCP`连接过程，实际上是没有真正的`TCP`连接。所以使用`TCP ACK`扫描不能够确定端口的关闭或者开启，因为当发送给对方一个含有`ACK`表示的`TCP`报文的时候，都返回含有`RST`标志的报文，无论端口是开启或者关闭。但是可以利用它来扫描防火墙的配置和规则等。

![enter image description here](http://drops.javaweb.org/uploads/images/238b8f9af5713772fc197c0a44855d767dd0eebf.jpg)

### 5、UDP端口扫描

前面的扫描方法都是针对`TCP`端口，针对`UDP`端口一般采用`UDP ICMP`端口不可达扫描。

如：`192.1.14.235`对`172.16.2.4`发送大量`UDP`端口请求，扫描其开启`UDP`端口的情况。

![enter image description here](http://drops.javaweb.org/uploads/images/bc2eaa84e0bde499335684a40782205f85a3c061.jpg)

如果对应的`UDP`端口开启，则会返回`UDP`数据包。

![enter image description here](http://drops.javaweb.org/uploads/images/42a9e1ff348789a104733ce2550d8aaba3c7d4ba.jpg)

如果端口未开启，则返回“`ICMP`端口不可达”消息。

![enter image description here](http://drops.javaweb.org/uploads/images/05670c17974cb2698ee4190bfdec92df6fa578c0.jpg)

0x03 操作系统的探测
=====

`NMAP`进行操作系统的探测主要用到的是`OS`探测模块，使用`TCP/IP`协议栈指纹来识别不同的操作系统和设备。`Nmap`内部包含了`2600`多种已知操作系统的指纹特征，根据扫描返回的数据包生成一份系统指纹，将探测生成的指纹与`nmap-os-db`中指纹进行对比，查找匹配的操作系统。如果无法匹配，则以概率形式列举出可能的系统。

如：`192.168.1.50`对`192.168.1.90`进行操作系统的扫描探测。首先发起`Ping`请求，确认主机是否存在。

![enter image description here](http://drops.javaweb.org/uploads/images/b9955f09febd9fcee07178b27e8d9ed22286eba8.jpg)

发起`ARP`请求，获取主机`MAC`地址。

![enter image description here](http://drops.javaweb.org/uploads/images/6ff8bc4dcb0bbbf9900154a6741b04b89d578616.jpg)

进行端口扫描。

![enter image description here](http://drops.javaweb.org/uploads/images/6b625051cf153227adf677d211a3a005035a7fb0.jpg)

根据综合扫描情况，判断操作系统类型。

![enter image description here](http://drops.javaweb.org/uploads/images/884d70519657abfe320974c9657f183fd3caab88.jpg)

0x04 漏洞扫描
=====

操作系统的漏洞探测种类很多，本文针对“`smb-check-vulns`”参数就`MS08-067`、`CVE2009-3103`、`MS06-025`、`MS07-029`四个漏洞扫描行为进行分析。

攻击主机：`192.168.1.200`（Win7），目标主机：`192.168.1.40`（WinServer 03）；

`Nmap`扫描命令：`nmap --script=smb-check-vulns.nse --script-args=unsafe=1 192.168.1.40`。

![enter image description here](http://drops.javaweb.org/uploads/images/9b962b0df502b85d0aa8ca1cceceabb95b4f11f1.jpg)

### 1、端口扫描

漏洞扫描前，开始对目标主机进行端口扫描。

![enter image description here](http://drops.javaweb.org/uploads/images/33111e358878087174262aceba832966faddd62b.jpg)

### 2、SMB协议简单分析

由于这几个漏洞多针对SMB服务，下面我们简单了解一下`NAMP`扫描行为中的SMB命令。

SMB Command:`Negotiate Protocol`(0x72)：SMB协议磋商

SMB Command:`Session Setup AndX`(0x73)：建立会话，用户登录

SMB Command:`Tree Connect AndX`(0x75)：遍历共享文件夹的目录及文件

SMB Command:`NT Create AndX`(0xa2)：打开文件，获取文件名，获得读取文件的总长度

SMB Command:`Write AndX`(0x2f)：写入文件，获得写入的文件内容

SMB Command:`Read AndX`(0x2e)：读取文件，获得读取文件内容

SMB Command:`Tree Disconnect`(0x71)：客户端断开

SMB Command:`Logoff AndX`(0x74)：退出登录

![enter image description here](http://drops.javaweb.org/uploads/images/58cd1fdba3382ef38f5e63556fca227202a12f11.jpg)

### 3、MS08-067漏洞

**（1）MS08-067漏洞扫描部分源码如下：**

```
function check_ms08_067(host)
    if(nmap.registry.args.safe ~= nil) then
        return true, NOTRUN
    end
    if(nmap.registry.args.unsafe == nil) then
        return true, NOTRUN
    end
    local status, smbstate
    local bind_result, netpathcompare_result

    -- Create the SMB session  \\创建SMB会话
    status, smbstate = msrpc.start_smb(host, "\\\\BROWSER")
    if(status == false) then
        return false, smbstate
    end

    -- Bind to SRVSVC service
    status, bind_result = msrpc.bind(smbstate, msrpc.SRVSVC_UUID, msrpc.SRVSVC_VERSION, nil)
    if(status == false) then
        msrpc.stop_smb(smbstate)
        return false, bind_result
    end

    -- Call netpathcanonicalize
--  status, netpathcanonicalize_result = msrpc.srvsvc_netpathcanonicalize(smbstate, host.ip, "\\a", "\\test\\")

    local path1 = "\\AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA\\..\\n"
    local path2 = "\\n"
    status, netpathcompare_result = msrpc.srvsvc_netpathcompare(smbstate, host.ip, path1, path2, 1, 0)

    -- Stop the SMB session
    msrpc.stop_smb(smbstate)

```

**（2）分析**

尝试打开“`\\BROWSER`”目录，下一包返回成功。

![enter image description here](http://drops.javaweb.org/uploads/images/b6f2425a80b7093fa3f9bfebe27ce2ba04d8d174.jpg)

同时还有其它尝试，均成功，综合判断目标存在`MS08-067`漏洞。通过`Metasploit`进行漏洞验证，成功溢出，获取Shell。

![enter image description here](http://drops.javaweb.org/uploads/images/2a00b655456e320f5fbc4a23d8e2f032886f3dde.jpg)

### 4、CVE-2009-3103漏洞

**（1）CVE-2009-3103漏洞扫描部分源码如下：**

```
host = "IP_ADDR", 445
buff = (
"\x00\x00\x00\x90" # Begin SMB header: Session message
"\xff\x53\x4d\x42" # Server Component: SMB
"\x72\x00\x00\x00" # Negociate Protocol
"\x00\x18\x53\xc8" # Operation 0x18 & sub 0xc853
"\x00\x26"# Process ID High: --> :) normal value should be "\x00\x00"
"\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\xff\xff\xff\xfe"
"\x00\x00\x00\x00\x00\x6d\x00\x02\x50\x43\x20\x4e\x45\x54"
"\x57\x4f\x52\x4b\x20\x50\x52\x4f\x47\x52\x41\x4d\x20\x31"
"\x2e\x30\x00\x02\x4c\x41\x4e\x4d\x41\x4e\x31\x2e\x30\x00"
"\x02\x57\x69\x6e\x64\x6f\x77\x73\x20\x66\x6f\x72\x20\x57"
"\x6f\x72\x6b\x67\x72\x6f\x75\x70\x73\x20\x33\x2e\x31\x61"
"\x00\x02\x4c\x4d\x31\x2e\x32\x58\x30\x30\x32\x00\x02\x4c"
"\x41\x4e\x4d\x41\x4e\x32\x2e\x31\x00\x02\x4e\x54\x20\x4c"
"\x4d\x20\x30\x2e\x31\x32\x00\x02\x53\x4d\x42\x20\x32\x2e"
"\x30\x30\x32\x00"
)

```

**（2）分析**

十六进制字符串“`0x00000000`到`202e30303200`”请求，通过`ASCII`编码可以看出是在探测`NTLM`和`SMB`协议的版本。无响应，无此漏洞。

![enter image description here](http://drops.javaweb.org/uploads/images/4907b7e22f490718957fb9957247ac3610e138c5.jpg)

### 5、MS06-025漏洞

**（1）MS06-025漏洞扫描部分源码如下：**

```
--create the SMB session
--first we try with the "\router" pipe, then the "\srvsvc" pipe.
local status, smb_result, smbstate, err_msg
status, smb_result = msrpc.start_smb(host, msrpc.ROUTER_PATH)
if(status == false) then
err_msg = smb_result
status, smb_result = msrpc.start_smb(host, msrpc.SRVSVC_PATH) --rras is also accessible across SRVSVC pipe
if(status == false) then
    return false, NOTUP --if not accessible across both pipes then service is inactive
end
end
smbstate = smb_result
--bind to RRAS service
local bind_result
status, bind_result = msrpc.bind(smbstate, msrpc.RASRPC_UUID, msrpc.RASRPC_VERSION, nil)
if(status == false) then 
msrpc.stop_smb(smbstate)
return false, UNKNOWN --if bind operation results with a false status we can't conclude anything.
End

```

**（2）分析**

先后尝试去连接“`\router`”、“`\srvsvc`”路径，均报错，无`RAS RPC`服务。

![enter image description here](http://drops.javaweb.org/uploads/images/da8a2435fa821983e42ff5c6c18db6f11d29fb8f.jpg)

![enter image description here](http://drops.javaweb.org/uploads/images/62efda9ae1ff96d6f2252a24dc28aa2a101a1590.jpg)

### 6、MS07-029漏洞

**（1）MS07-029漏洞扫描部分源码如下：**

```
function check_ms07_029(host)
    --check for safety flag  
if(nmap.registry.args.safe ~= nil) then
        return true, NOTRUN
end
if(nmap.registry.args.unsafe == nil) then
return true, NOTRUN
end
    --create the SMB session
    local status, smbstate
    status, smbstate = msrpc.start_smb(host, msrpc.DNSSERVER_PATH)
    if(status == false) then
        return false, NOTUP --if not accessible across pipe then the service is inactive
    end
    --bind to DNSSERVER service
    local bind_result
    status, bind_result = msrpc.bind(smbstate, msrpc.DNSSERVER_UUID, msrpc.DNSSERVER_VERSION)
    if(status == false) then
        msrpc.stop_smb(smbstate)
        return false, UNKNOWN --if bind operation results with a false status we can't conclude anything.
    end
    --call
    local req_blob, q_result
    status, q_result = msrpc.DNSSERVER_Query(
        smbstate, 
        "VULNSRV", 
        string.rep("\\\13", 1000), 
        1)--any op num will do
    --sanity check
    msrpc.stop_smb(smbstate)
    if(status == false) then
        stdnse.print_debug(
            3,
            "check_ms07_029: DNSSERVER_Query failed")
        if(q_result == "NT_STATUS_PIPE_BROKEN") then
            return true, VULNERABLE
        else
            return true, PATCHED
        end
    else
        return true, PATCHED
    end
end

```

**（2）分析**

尝试打开“`\DNSSERVER`”，报错，未开启`DNS RPC`服务。

![enter image description here](http://drops.javaweb.org/uploads/images/a71e4f4c9aa3862e25935662f14fb347b8d58a78.jpg)

0x05 总结
=====

1、扫描探测可以说是所有网络中遇到最多的攻击，因其仅仅是信息搜集而无实质性入侵，所以往往不被重视。但扫描一定是有目的的，一般都是攻击入侵的前兆。

2、修补漏洞很重要，但如果在扫描层面进行防御，攻击者就无从知晓你是否存在漏洞。

3、扫描探测一般都无实质性通信行为，同时大量重复性动作，所以在流量监测上完全可以做到阻止防御。