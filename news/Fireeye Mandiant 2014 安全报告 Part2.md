# Fireeye Mandiant 2014 安全报告 Part2

0x00 进化中的攻击生命周期
=====

在我们调查过的多数案子中都出现了相似的行为模式，我们叫它攻击生命周期。

安全就像猫捉老鼠，安全团队加了些新的防御措施，然后攻击者换了种攻击方式，绕过你的防御体系。在2014年里还是这样。我们发现更多攻击使用了VPN来连接受害方的网络。同时也出现了许多很聪明的新方法来绕过检测，新工具和技巧从已攻下的环境中传出信息。

0x01 VPN劫持
=====

获取了目标网络的VPN登录权限给了攻击者两个非常有利的优势。首先他们可以在不用部署后门的情况下持续连接到目标网络内。其次他们可以像正常用户那样登录内部网络。

在最近几年中，我们发现一些组织在内网站住脚之后，立刻瞄准了VPN服务器和VPN账号。在2014年，这种趋势达到了有史以来最高，越来越多的攻击通过受害方的VPN服务进行。

我们发现了两种常见的攻击VPN的方式:

·**单因素认证**: 如果对方的VPN只需要用户名和密码就可以登录，攻击者只要简单的用之前收集到的账号或者从AD获取的账号密码就可以了。

·**基于账号的双因素认证**: 如果对VPN需要双因素认证比如用户证书，攻击者会使用常见的工具比如Mimikatz来从用户终端中获取证书。我们还发现了一些被窃取的用户证书是通过不安全的方式分发给用户的，比如通过email附件发过去(论社工目标邮箱的重要性)或者放在开放的网络共享中。

在另一些比较少见的场景中，攻击者使用了一些漏洞来绕过VPN，比如通过Heartbleed，可以从服务器的内存中获取64kB的数据。安全研究员一开始对这个漏洞的影响有些怀疑，比如是否能在实际攻击中窃取到敏感信息，例如，加密密钥，密码和证书等等。

之后他们的恶梦成真了，在Heartbleed漏洞公开后一周内，我们发现了一起利用这个漏洞获取了VPN的session的安全事件，不需要用户名和密码。在之后的几周里，攻击者通过Heartbleed攻击了其他受害者的VPN系统。

道高一尺魔高一丈。在2014年我们在攻击生命周期里发现了一些新的技巧，有些比较有特点。

在这些案例中，VPN记录给了我们一些提示: 被攻击者利用的VPN账号的登录源地址经常变动，比如在不同ISP的IP段中切换，甚至在不同国家中切换等等。(论SIEM的重要性)

0x02 在眼皮底下隐藏的恶意软件
=====

恶意软件检测是一个永不停歇过程。在2014年我们发现攻击者使用了很多新技巧来隐藏他们的操作和新的安装长期后门的方式。

隐藏webshell 我们发现了几起案例，攻击者把webshell放到了有SSL的服务器上，因为对方的网络结构不支持SSL流量检查，所以攻击者的webshell绕过了基于流量的识别。

另一个趋势是在合法网页里嵌入一句话webshell，比如eval之类的。 最后一种很狡诈的后门隐藏方式是在配置文件比如web.config里加载恶意模块。例如:

```
<!--HTTP Modules -->
 <modules>
 <add type=”Microsoft.Exchange.Clients.BadModule” name=”BadModule” />
 </modules>

```

攻击者利用配置文件加载了一个伪装成真实微软DLL文件名的恶意模块，并且把配置文件和dll文件的时间戳也改了。

0x03 WMI后门
=====

Windows Management Instrumentation (WMI)是一个Windows的核心组件，它提供了广泛的系统管理功能和接口。程序和各种脚本语言都可以用WMI来收集数据，和系统底层交互和执行命令。WMI还提供了一个基于事件(Event)的触发器功能，可以被用来在特定对象的状态发生改变的时候触发恶意软件。

在前几年，我们没有发现有多少攻击者通过WMI来绕过检测，这很可能是因为和WMI交互比较复杂，另外一些基础的绕过方式已经足够了。但是在2014年，我们发现了一些组织通过WMI来实现隐蔽的后门。

这些技巧用了WMI的三个部分，一般通过powershell来执行:

```
Event filter 事件过滤器: 利用系统定时轮询的功能来达到长期执行的目的，比如每天x点x分执行一次。
Event consumer事件消费者: 在发生特定事件的时候执行特定的程序或者命令。
Filer to Consumer Binding: 把consumer绑定到filter上，确保在事件发生的时候event consumer得到执行。

```

![enter image description here](http://drops.javaweb.org/uploads/images/97479443401302ffb91b74b0d5438970bd0354e8.jpg)

Figure7:

1. 攻击者通过powershell命令建立3个WMI事件对象:
--------------------------------

一个consumer用来执行一个命令或脚本 一个filter来轮询系统 一个binding来把filter绑定到consumer

2. WMI定期向系统轮询event filter中的事件，在这个例子中，filter的事件是当时间 = 每天8:05的时候。
---------------------------------------------------------------

```
SELECT * FROM __InstanceModificationEvent WITHIN 60 WHERE TargetInstance ISA 'Win32_LocalTime' AND TargetInstance.Hour = 08 AND TargetInstance.Minute = 05 GROUP WITHIN 60

```

3. 当filter被触发的时候，WMI自动执行绑定的consumer。这个例子显示了consumer执行的一部分命令，通过powershell来执行base64编码后的恶意代码。
------------------------------------------------------------------------------------------

```
CommandLineTemplate="C:\WINDOWS\System32\WindowsPowerShell\v1.0\powershell.exe
–NonInteractive –enc SQBuAHYAbwBrAGUALQBDAG8AbQBtAGEAbgBkACAALQBDAG8AbQBwAHUAdABl...

```

下面是通过powershell建立WMI命令执行consumer的例子，利用powershell.exe执行传入的base64编码后的参数，base64编码后的内容也是powershell脚本，比如下载执行一些程序等等。如果绑定的是可以重复执行的filter，那么也可以定时执行命令.

```
Set-WmiInstance -Namespace “root\subscription” -Class ‘CommandLineEventConsumer’
-Arguments @{ name=’EvilWMI’;CommandLineTemplate=”C:\WINDOWS\System32\WindowsPowerShell\v1.0\powershell.exe
–enc SQBuAHYAbwBrAGUALQBDAG8AG8AbQ...<SNIP>”;RunInteractively=’false’}

```

基于WMI的持久后门给取证带来了一些难度。WMI filter和consumer没有在注册表里留下痕迹。 WMI对象存在于硬盘上的一个复杂的数据库中(objects.data)，给分析带来了难度。另外，Windows只会在debug级的日志打开后才会记录WMI执行的审核日志，此外这也不是长久之计，因为日志量非常大(windows本身也有很多功能用到MWI)。

0x04 恶意安全模块(Security Packages)
=====

我们还观察到几个案例，攻击者使用了Windows Local Security Authority (LSA) security packages. 一种少见的基于注册表的长期后门。用来隐蔽的自动加载恶意软件。 安全包是一些在系统启动时由LSA自动加载的一些DLL。这些安全模块通过注册表里的 HKLM\ SYSTEM\CurrentControlSet\Control\Lsa 键值来加载。 每个键值都包括了一个字符串列表，指向%SYSTEMROOT%\system32\下需要加载的文件名（不含扩展名）

因为LSA模块是通过lsass.exe自动加载的，具有管理员权限的攻击者可以添加或者修改这些参数，加载恶意DLL。在2014年，我们发现有攻击者通过安全包的功能在系统上加载了一个多级后门，tspkgEX.dll

HKLM\SYSTEM\CurrentControlSet\Control\Lsa\Security Packages中加载的模块:

```
SECURITY PACKAGES (修改前): kerberos msv1_0 schannel wdigest tspkg pku2u
SECURITY PACKAGES (修改后): kerberos msv1_0 schannel wdigest tspkg pku2u tspkgEx

```

这导致了系统启动后自动加载`C:\WINDOWS\system32\tspkgEx.dll`

因为LSA的可扩展性，自定义的安全模块可以在用户登录的时候抓到明文密码。

0x05 Mimikatz是个 女子 东 西
=====

在几乎全部的调查中我们都遇到了能绕过杀软的各种Mimikatz的改版，攻击者一般都修改和重新编译了源码来绕过检测。 Mimikatz也包括了一个邪恶的LSA模块 mimilib ssp 用来获取密码。

简单的得到密码

2014年用的比较多的两个方法: Pass the hash，通过获取的hash来认证。

用Mimikatz从内存获取明文密码。 微软在windows server 2012R2和Win8里减少了上面两个方法的有效性，但是我们遇到的大部分客户都还在用Server 2008和win7.

hash传递还是很好用的，尤其是很多系统都用相同的本地管理员密码的时候。Mimikatz更加进一步，能直接从内存中得到明文密码。

在一台员工的电脑上，能获取的仅限于员工账号的密码。在一个有很多交互式登录的服务器上，能获取到的东西多的多。受害者们很快发现，从几台系统被攻下到整个域被拿下只需要很短的时间。

在我们的调查中，几乎从来没有杀软检测到了Mimikatz，甚至还有基于powershell的脚本在内存中执行Mimikatz。

Mimikatz还能够生成Kerberos golden ticket，在攻击者拿下域控之后，生成的一种可以无限期生效的金钥匙，可以用来代表任何账号登录，即使那个账号密码改过了。有了这个金钥匙，攻击者只要到了内网环境，就可以随时重新获取整个域的管理权限。

唯一的防御Kerberos金钥匙的方法是，除了别让你的域被攻下之外，重置服务Kerberos Key Distribution账号的krbtgt密码两次。这样即可让生成的金钥匙失效。

0x06 通过WMI和Powershell后期渗透
=====

在过去，在windows内网环境中渗透用的都是一些windows自带的工具，比如net，at，还有自己写的小工具，脚本或者vbs，要么就是psexec。这些手段又快又好用，但是会留下一些证据还有记录。

2013到2014年，我们发现了我们跟踪的APT组织的行为发生了显著的变化，他们越来越多的使用了WMI和powershell进行后期渗透，收集密码和信息等等。 同样，安全研究和渗透测试工具也广泛的使用了powershell，这或许开源了许多信息和源码，让攻防两方都学了不少东西。

0x07 本篇总结,攻击者的新技巧:
=====

VPN劫持: 在这一年里我们发现了更多受害方的VPN权限被攻击者成功获取。

明文密码: 攻击者重新编译了Mimikatz，打造了各种各样的工具来从内存中获取明文密码，并且可以绕过杀毒软件。

Webshell隐藏: 攻击方继续用着各种新颖的方式来隐藏web shell。我们发现了一些很隐蔽的方式: 把webshell放到可以用SSL访问的地方来躲避网络检测 在正常网页里的一句话eval后门(老外你们弱爆了，eval一句话我们早玩腻了) 通过服务器配置文件来加载恶意DLL(类似Apache的模块后门)

恶意安全模块: 利用Windows security package的可扩展性来加载后门和key logger

使用WMI和powershell: 越来越多的攻击利用了WMI和Powershel，两个windows内置的强大工具/功能，用来保持长期后门，收集数据和持续渗透。

Kerberos攻击: 在拿到了域管理员权限后，攻击者使用了Kerberos golden ticket attack来登录任意账号，即使域管理员重置了密码。