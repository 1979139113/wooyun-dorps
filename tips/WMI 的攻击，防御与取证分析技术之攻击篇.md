# WMI 的攻击，防御与取证分析技术之攻击篇

近日，FireEye 安全公司的高级逆向工程团队（FLARE）发布了一份标题为《 WMI 攻击，防御与取证分析技术 》的 PDF 文档，该文档页数多达 90 页，文档内容主要从攻击，防御和取证分析这三个角度分篇对 WMI 技术做了详细的描述。其中不乏有很多值得学习思考的地方。于是，我利用业余时间翻译整理了此文档，拿出来与大家共分享 :），如有纰漏，望各位不吝赐教。

为了对原文档内容进行全面的翻译和解读，我按照文章的分析角度对原文档进行了分段式的翻译，本篇文章是分段式里面的第一篇，其余两篇译文的标题分别为：

*   **《 WMI 的攻击，防御与取证分析技术之防御篇 》**
*   **《 WMI 的攻击，防御与取证分析技术之取证分析篇 》**

0x00 WMI 简介
=====

WMI 的全称是 Windows Management Instrumentation，即 Windows 管理规范，在 Windows 操作系统中，随着 WMI 技术的引入并在之后随着时间的推移而过时，它作为一项功能强大的技术，从 Windows NT 4.0 和 Windows 95 开始，始终保持其一致性。它出现在所有的 Windows 操作系统中，并由一组强大的工具集合组成，用于管理本地或远程的 Windows 系统。

尽管已被大众所知并且从其创始以来，已经被系统管理员大量使用，但当WMI技术在震网病毒中被发现以后，它开始在安全社区变得非常流行。从那之后， WMI 在攻击中变得日益普及，其作用有执行系统侦察，反病毒和虚拟机检测，代码执行，横向运动，权限持久化以及数据窃取。

随着越来越多的攻击者利用 WMI 进行攻击，他将会是安全维护人员，事件响应人员，取证分析师必须掌握的一项重要技能，并且要明白如何发挥它的优势。本白皮书介绍了 WMI 技术，并且演示了在实际攻击当中使用 WMI 构造 POC 和如何使用 WMI 作为一个基本的 IDS 以及提出了如何在 WMI 存储库文件格式中进行取证分析。

0x01 修订历史
=====

在 Win2K 之前的操作系统中，就已经支持了 WMI 技术，只是当时需要下载并安装一个开发包。从 Win2K 开始，系统自带了 WMI ，并且 WMI 成为了系统的一个重要组件。随着 XP、2003、Vista、Win7 等的发布， WMI 所能提供的功能也在不断的增强和完善中。

下面是操作系统版本中对应的 WMI 的版本：

*   Nt 4.0 1.01
*   Sms 2.0 1.1
*   Win2000 1.5
*   WinXP/2003 2.0

0x02 WMI 体系结构
=====

WMI是微软实现的由分布式管理任务组（DMTF）发布的基于 Web 的企业管理（WBEM）和公共信息模型（CIM）标准。这两个标准的目的是提供工业不可知论者手段，收集和传播在企业中有关的任何托管组件中的信息。

在一个较高的水平上，微软所实现的这些标准可以总结如下：

**托管组件**

托管组件被表示为 WMI 对象 —— 表示高度结构化的操作系统数据的类实例。微软提供了丰富的 WMI 对象用来与操作系统相关的信息进行通信。例如：`Win32_Process,Win32_Service,AntiVirusProduct,Win32_StartupCommand`等等。

**使用 WMI 数据**

微软提供了几种方式来使用 WMI 数据和执行 WMI 方法。例如， PowerShell 提供了一种非常简单的方式与 WMI 进行交互。

**查询 WMI 数据**

所有的WMI对象都使用类似于一个 SQL 查询的语言称为 WMI 查询语言（WQL）。 WQL 能够很好且细微的控制返回给用户的 WMI 对象。

**填充 WMI 数据**

当用户请求特定的 WMI 对象时，WMI 服务 (Winmgmt) 需要知道如何填充被请求的 WMI 对象。这个过程是由 WMI 提供程序去实现的。WMI 提供程序是一个基于 COM 的 DLL 文件 ，它包含一个在注册表中已经注册的相关联的 GUID 。 WMI 提供程序的功能 —— 例如查询所有正在运行的进程，枚举注册表项等等。

当 WMI 服务填充 WMI 对象时，有两种类型的类实例: 动态对象和持久性对象。动态对象是在特定查询执行时在运行过程中生成的。例如，Win32_Process 对象就是在运行过程中动态生成的。持久性对象存储在位于`%SystemRoot%\System32\wbem\Repository\`的 CIM 数据库中，它存储着 WMI 类的实例，类的定义和命名空间的定义。

**结构化 WMI 数据**

绝大多数的 WMI 对象的架构是在托管对象格式 (MOF) 文件中描述的。MOF 文件使用类似于 C++ 的语法并为一个 WMI 对象提供架构。因此，尽管 WMI 提供程序产生了原始数据，但是 MOF 文件为其产生的数据提供了被格式化的模式。从安全维护人员的角度来看，值得注意的是， WMI 对象定义可以在没有 MOF 文件的情况下被创建。相反，他们可以使用 .NET 代码直接插入到 CIM 资料库中。

**远程传输 WMI 数据**

Microsoft 提供了两个协议用于远程传输 WMI 数据: 分布式组件对象模型 (DCOM) 和 Windows 远程管理 (WinRM)。

**执行 WMI 操作**

部分 WMI 对象包括可执行的方法。例如，攻击者进行横向运动时执行的一个常用方法是在`Win32_Process`类中的静态`Create`方法，此方法可以快速创建一个新的进程。另外， WMI 提供了一个事件系统，使用户可以使用注册事件处理函数进行创建，修改或删除任何 WMI 对象实例。

图 1 提供了微软实现 WMI 的一个高级别概述以及微软实现的组件和实现的标准之间的关系。

![](http://drops.javaweb.org/uploads/images/5c0f397ae7c5af2cdc0d9b0dea6db8ac2ba07d2d.jpg)

图 1: WMI 体系结构的高级别概述

0x03 WMI 的类与命名空间
=====

WMI 代表着大多数与操作系统信息以及以对象的形式操作有关的数据。一个 WMI 对象是高度结构化定义的信息被如何表示的类的实例。在 MSDN 上，有很多常用的 WMI 类的详细介绍。例如，常见的、有据可查的 WMI 类是`Win32_Process`。还有很多未文档化的 WMI 类，幸运的是，所有的 WMI 类都可以使用 WMI 查询语言 (WQL) 进行查询。

WMI 类的命名空间的层次结构非常类似于传统的，面向对象的编程语言的命名空间。所有的命名空间都派生自根命名空间，在用脚本语言查询对象并未显式指定命名空间时，微软使用 ROOT\CIMV2 作为默认的命名空间。在下面的注册表项中包含所有 WMI 设置，也包括已定义的默认命名空间:

**`HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\WBEM`**

可以使用下面的 PowerShell 代码递归查询所有的 WMI 类和它们各自的命名空间。

![](http://drops.javaweb.org/uploads/images/022dce4f479f0065f5d179b416f8b6a994949952.jpg)

图 2：列举所有 WMI 类和命名空间的 PowerShell 示例代码

我们在 Windows 7 系统上测试后发现已经有 7950 个 WMI 类，这意味着有大量的操作系统数据可被检索。

下面是由上述脚本执行后返回的完整 WMI 类路径的一部分结果：

![](http://drops.javaweb.org/uploads/images/3d14bc231971e84aed2bb66b2e3e6145ea648326.jpg)

0x04 查询 WMI
=====

WMI 提供了一种简单的语法用于查询 WMI 对象实例、 类和命名空间 — —[WMI 查询语言 (WQL)](https://msdn.microsoft.com/en-us/library/aa392902(v=vs.85).aspx)。

有三种类别的 WQL 查询:

1.实例查询 —— 用于查询 WMI 类的实例  
2.事件查询 —— 用于一个 WMI 事件注册机制，例如 WMI 对象的创建、 删除或修改  
3.元查询 —— 用于查询 WMI 类架构

实例查询
----

实例查询是最常见的用于获取 WMI 对象实例的 WQL 查询。基本的实例查询采用以下形式:

**`SELECT [Class property name|*] FROM [CLASS NAME] <WHERE [CONSTRAINT]>`**

以下查询将返回所有正在运行的进程的可执行文件名称中包含"Chrome"的结果。具体的说是，此查询将返回 Win32_Process 类的每个实例的所有属性的名称字段中包含字符串"Chrome"的结果。

**`SELECT * FROM Win32_Process WHERE Name LIKE "%chrome%"`**

事件查询
----

事件查询提供了报警机制，触发事件的类。在 WMI 类实例被创建时被用于常用的事件查询触发器。事件查询将采取以下形式:

**`SELECT [Class property name|*] FROM [INTRINSIC CLASS NAME] WITHIN [POLLING INTERVAL] <WHERE [CONSTRAINT]>`**

**`SELECT [Class property name|*] FROM [EXTRINSIC CLASS NAME] <WHERE [CONSTRAINT]>`**

内部和外部的事件将在事件章节中进一步详细解释。

下面是交互式用户登录的事件查询触发器。根据[MSDN 文档描述](https://msdn.microsoft.com/en-us/library/aa394189(v=vs.85).aspx)，交互式登录的LogonType值为 2。

**`SELECT * FROM __InstanceCreationEvent WITHIN 15 WHERE TargetInstance ISA 'Win32_LogonSession' AND TargetInstance.LogonType = 2`**

下面是在可移动媒体插入时的事件查询触发器：

**`SELECT * FROM Win32_VolumeChangeEvent WHERE EventType = 2`**

元查询
---

元查询提供一个 WMI 类架构发现和检查机制。元查询采用以下形式:

**`SELECT [Class property name|*] FROM [Meta_Class<WHERE [CONSTRAINT]>`**

以下查询将列出所有以字符串 "Win32" 开头的 WMI 类：

**`SELECT * FROM Meta_Class WHERE __Class LIKE "Win32%"`**

当执行任何 WMI 查询时，除非显式提供命名空间，否则将隐式使用默认的命名空间 ROOT\CIMV2。

0x05 与 WMI 进行交互
=====

Microsoft 和第三方供应商提供了丰富的客户端工具使您可以与 WMI 进行交互。以下是此类客户端实用程序的非详尽清单：

PowerShell
----------

PowerShell 是功能极其强大的脚本语言，包含了丰富的与 WMI 进行交互的功能。截至 PowerShell V3，以下 cmdlet (PowerShell 命令术语) 可用于与 WMI 进行交互：

*   Get-WmiObject
*   Get-CimAssociatedInstance
*   Get-CimClass
*   Get-CimInstance
*   Get-CimSession
*   Set-WmiInstance
*   Set-CimInstance
*   Invoke-WmiMethod
*   Invoke-CimMethod
*   New-CimInstance
*   New-CimSession
*   New-CimSessionOption
*   Register-CimIndicationEvent
*   Register-WmiEvent
*   Remove-CimInstance
*   Remove-WmiObject
*   Remove-CimSession

WMI 和 CIM 的 cmdlet 也提供了类似的功能。然而，CIM cmdlet 引入了 PowerShell V3，并通过[WMI cmdlets](http://blogs.msdn.com/b/powershell/archive/2012/08/24/introduction-to-cim-cmdlets.aspx)提供了一些额外的灵活性。使用 CIM cmdlet 的最大优点是它们工作在 WinRM 和 DCOM 协议之上。WMI cmdlet 只工作在 DCOM 协议之上。但是并不是所有的系统都将安装 PowerShell v3+。PowerShell v2 是默认安装在 Windows 7 上的。因此，它被攻击者视为最小公共程序。

wmic.exe
--------

wmic.exe 是一个与 WMI 进行交互的强大的命令行实用工具。它拥有大量的 WMI 对象的方便记忆的默认别名，但你还可以执行更为复杂的查询。wmic.exe 还可以执行 WMI 方法，攻击者经常用来通过调用 Win32_Process 的 Create 方法来进行横向运动。Wmic.exe 的局限性之一是不能接受调用嵌入的 WMI 对象的方法。在 PowerShell 不可用的情况下,使用 wmic.exe 足够用于执行系统侦察和基本方法的调用。

wbemtest.exe
------------

wbemtest.exe 是一个功能强大的带有图形界面的 WMI 诊断工具。它能够枚举对象实例、执行查询、注册事件、修改 WMI 对象和类，并且可以在本地或远程去调用方法。它的接口对大多数用户来说不是特别友好，但从攻击者的角度来看，在其他工具不可用时，它完全可以作为替代选项 —— 例如，如果应用程序白名单机制阻止了 wmic.exe 和 powershell.exe，那么 wbemtest.exe 将是一个带有一个不太理想的 UI （如图 3 所示）但是功能却很强大的实用工具。

![](http://drops.javaweb.org/uploads/images/d20e0cf37931ac921328308be0640115580bdc09.jpg)

图 3 wbemtest的图形接口

WMI Explorer
------------

WMI Explorer 是一个很好的 WMI 类发现工具。它提供了一个优雅的 GUI （图 4 所示），你可以使用分层次的方式探索 WMI 存储库。它也能够连接到远程的 WMI 存储库，并执行查询。对安全研究人员寻找可用于攻击或防御的 WMI 类来说，像这样的 WMI 类发现工具是非常有价值的。

![](http://drops.javaweb.org/uploads/images/ebb8b20110c79725cc2d8dd45eff16902f0ffa57.jpg)

图 4 WMI Explorer

CIM Studio　
-----------

CIM Studio 是 Microsoft 遗留的一个免费工具，你可以方便地浏览 WMI 存储库。像 WMI Explorer 一样，此工具也可以很好的进行 WMI 类发现。

![](http://drops.javaweb.org/uploads/images/e69d51405fa1c0f8246fdda6a0fca79739c9cf66.jpg)

Windows 脚本宿主（WSH）语言
-------------------

Microsoft 提供了两个 WSH 脚本语言，VBScript 和 JScript。尽管它们比较过时，也算不上高雅的编程语言，但是说到与 WMI 进行交互时，它们的确都是功能强大的脚本语言。事实上，使用 VBScript 和 JScript 编写的利用 WMI 作为主要的 C&C 机制的后门已经出现了。此外，如后面将要解释的，它们是唯一支持`ActiveScriptEventConsumer`事件消费者组件的语言，该组件对于攻击者和防御者来说都是一个非常有价值的 WMI 组件。最后，从攻击的角度来看， VBScript 和 JScript 是在未安装 PowerShell 的老版本的 Windows 系统上的最小公共程序。

C/C++ 调用 IWbem* COM API
-----------------------

如果你需要使用非托管语言如 C 或 C++ 与 WMI 进行交互，你将需要使用[WMI 的 COM API](https://msdn.microsoft.com/en-us/library/aa389276(v=vs.85).aspx)。逆向工程师将需要非常熟悉此接口以及每一个 COM Guid 才能充分理解与 WMI 交互的恶意软件。

.NET System.Management 类
------------------------

.NET 类库在 System.Management 命名空间中提供了几个与 WMI 相关的类，可以相对简单的使用如 C#、VB.Net 和 F# 语言编写与 WMI 交互的程序。在后续的示例中，这些类将用于在 PowerShell 代码中补充现有的 WMI/CIM cmdlet。

winrm.exe
---------

winrm.exe 可以在运行 WinRM 服务的本地和远程计算机上进行枚举 WMI 对象实例、调用方法，并创建和删除对象实例。也可以用 winrm.exe 来配置 WinRM 设置。  
下面的示例显示了 winrm.exe 可用于执行命令、枚举对象的多个实例，并检索单个对象实例:

*   **`winrm invoke Create wmicimv2/Win32_Process @{CommandLine="notepad.exe";CurrentDirectory="C:\"}`**
*   **`winrm enumerate http://schemas.microsoft.com/wbem/wsman/1/wmi/root/cimv2/Win32_Process`**
*   **`winrm get http://schemas.microsoft.com/wbem/wsman/1/wmi/root/cimv2/Win32_OperatingSystem`**

Linux版本的 wmic 和 wmis-pth
------------------------

wmic 是一个简单的 Linux 命令行实用工具，用于执行 WMI 查询。wmis 是 Win32_Process 类的 Create 方法的远程调用命令行包装程序，并且支持使用 NTLM 哈希进行连接远程计算机，因此， wmis 已经被渗透测试人员大量使用。

0x06 远程使用 WMI
=====

虽然可以与本地的 WMI 进行交互，但是通过网络才能显示出 WMI 的真实能量。目前，由于 DCOM 和 WinRM 这两个协议的存在，将使得远程对象的查询，事件注册， WMI 类方法的执行，以及类的创建都能够被支持。

这些协议看起来非常有利于攻击者，因为大多数组织机构和安全供应商一般不审查这些恶意活动所传输的内容。攻击者需要有效的利用远程 WMI 则需要有特权用户的凭据。在 Linux 平台中的 wmis-pth 实用工具，则需要的是受害者的用户的哈希。

分布式组件对象模型 (DCOM)
----------------

从 DCOM 出现以来它一直是 WMI 所使用的默认协议，通过 TCP 的 135 端口建立初始连接。后续的数据交换则使用随机选定的 TCP 端口。可以通过 dcomcnfg.exe 并最终修改下面的注册表项来配置此端口的范围：

**`HKEY_LOCAL_MACHINE\Software\Microsoft\Rpc\Internet –Ports (REG_MULTI_SZ)`**

在 PowerShell 中内置的所有 WMI cmdlets 都是使用 DCOM 进行通信的。

![](http://drops.javaweb.org/uploads/images/52b54c4eac257e3eb11ec3fdb8013cfddedf2456.jpg)

Windows 远程管理 (WinRM)　
---------------------

最近， WinRM 取代了 DCOM 并成为 Windows 推荐的远程管理协议。WinRM 的构建基于 Web 服务管理 (WSMan) 规范 —— 一种基于 SOAP 的设备管理协议。此外，PowerShell 的远程传输协议也是基于 WinRM 规范的，同时 PowerShell 提供了极其强大的 Windows 企业级的远程管理。WinRM 也支持 WMI，以及通过网络执行 CIM 操作。

默认情况下，WinRM 服务监听的 TCP 端口是 5985 （HTTP），并且在默认情况下是加密的。还可以配置证书使其支持 HTTPS ，此时监听的 TCP 端口为 5986。

WinRM 的设置很容易配置，可以使用 GPO ， winrm.exe ，或 PowerShell 中的 WSMan PSDrive 来配置，如下所示：

![](http://drops.javaweb.org/uploads/images/8d06bfbe0a7b98655251468facc7c7b7e42b5198.jpg)

PowerShell 提供了一个 cmdlet 可以很方便的验证 WinRM 服务是否正在侦听 ——**`Test-WSMan`**。如果**`Test-WSMan`**返回了结果，则表明该系统的 WinRM 服务正处于监听状态。

![](http://drops.javaweb.org/uploads/images/223897f681e11a8b80bc8f5ca8aae6bc0f5e48cd.jpg)

为了与系统的 WMI 进行交互以便运行 WinRM 服务，唯一支持远程 WMI 交互的内置工具是 winrm.exe 和 PowerShell 的 CIM cmdlet。此外，对于没有运行 WinRM 服务的系统还可以使用 CIM cmdlet 来配置使用 DCOM 。

![](http://drops.javaweb.org/uploads/images/96279cc0f23b9aa825019fe320d82ce27cf0fbaf.jpg)

0x07 WMI 事件
=====

从攻击者或防御者的角度来看， WMI 最强大的功能之一就是对 WMI 事件的异步响应的能力。除了少数例外，WMI 事件几乎可以用于对操作系统的任何事件作出响应。例如，WMI 事件可能用于触发一个进程创建的事件。这种机制可随后被用作在任何 Windows 操作系统上执行命令行审计。

有两类 WMI 事件 —— 它们都运行在本地的单个进程和 WMI 永久事件订阅的上下文中。本地事件可以维持宿主进程的生存期，尽管 WMI 永久事件订阅存储在 WMI 存储库中，但是作为 SYSTEM 权限运行后依旧可以在重新启动之后继续持续运行。

事件触发条件
------

要安装一个永久事件订阅，下面三件事情是必须要做的：

1.事件筛选器 —— 筛选出感兴趣的事件  
2.事件消费者 —— 要在事件被触发时执行的操作  
3.消费者绑定筛选器 — — 将筛选器绑定到消费者的注册机制

事件筛选器
-----

事件筛选器描述了感兴趣的事件并且执行了 WQL 事件查询。一旦系统管理员配置了筛选器，他们就可以使用它在创建新的事件时接收到通知。举一个例子，事件筛选器可能用于描述以下一些事件:

*   创建一个具有特定名称的进程
*   将 DLL 加载到进程中
*   创建具有特定 ID 的事件日志
*   插入可移动媒体
*   用户注销
*   创建、修改、删除任何文件或目录

事件筛选器都被存储为一个 ROOT\subscription:__EventFilter 对象的实例。事件筛选器查询支持以下类型的事件:

### 内部事件

内部事件表示的是创建、修改和删除任何 WMI 类，对象或命名空间的事件。它们也可被用于计时器或 WMI 方法执行的警报。以下内部事件采用了系统类 (以两个下划线开头的那些) 的形式，并存在于每一个 WMI 命名空间:

*   __NamespaceOperationEvent
*   __NamespaceModificationEvent
*   __NamespaceDeletionEvent
*   __NamespaceCreationEvent
*   __ClassOperationEvent
*   __ClassDeletionEvent
*   __ClassModificationEvent
*   __ClassCreationEvent
*   __InstanceOperationEvent
*   __InstanceCreationEvent
*   __MethodInvocationEvent
*   __InstanceModificationEvent
*   __InstanceDeletionEvent
*   __TimerEvent

这些事件的作用非常强大，因为它们可以被用于在操作系统中几乎任何可以想见的事件的触发器。例如，如果触发了一个基于交互式登录的事件则可以形成下面的内部事件查询:

此查询被转换为创建一个登录类型为 2 （交互式）的`Win32_LogonSession`类的一个实例。

由于触发的内部事件有一定的频率，所以必须在 WQL 查询语句的 WITHIN 子句中指定事件轮询间隔。这就是说，它有时可能错过事件。例如，如果事件查询的形式目的是创建 WMI 类的实例，如果该实例的创建和销毁 (如常见的一些进程 —— Win32_Process 实例) 在轮询间隔内，那么则会错过这一事件。创建内部 WMI 查询时，必须考虑这种可能出现的情况。

**`SELECT * FROM __InstanceCreationEvent WITHIN 15 WHERE TargetInstance ISA 'Win32_LogonSession' AND TargetInstance.LogonType = 2`**

### 外部事件

外部事件解决了和内部事件有关的潜在的轮询问题，因为它们在事件发生时立刻被触发。然而美中不足的是在 WMI 中并没有太多的外部事件，不过，所有已经存在的外部事件的作用很强大，性能也很高。下面的外部事件对于攻击者和防御者来说可能是有用的：

*   ROOT\CIMV2:Win32_ComputerShutdownEvent
*   ROOT\CIMV2:Win32_IP4RouteTableEvent
*   ROOT\CIMV2:Win32_ProcessStartTrace
*   ROOT\CIMV2:Win32_ModuleLoadTrace
*   ROOT\CIMV2:Win32_ThreadStartTrace
*   ROOT\CIMV2:Win32_VolumeChangeEvent
*   ROOT\CIMV2: Msft_WmiProvider*
*   ROOT\DEFAULT:RegistryKeyChangeEvent
*   ROOT\DEFAULT:RegistryValueChangeEvent

以下外部事件查询形式可以用来捕获每一个进程已加载的所有可执行模块（用户模式和内核模式）：  
**`SELECT * FROM Win32_ModuleLoadTrace`**

### 事件消费者

事件消费是一个派生自 __EventConsumer 系统类的类，它表示了在事件触发时的动作。系统提供了以下有用的标准事件消费类：

*   LogFileEventConsumer - 将事件数据写入到指定的日志文件
*   ActiveScriptEventConsumer - 执行嵌入的 VBScript 或 JScript 脚本 payload
*   NTEventLogEventConsumer - 创建一个包含事件数据的事件日志条目
*   SMTPEventConsumer - 发送一封包含事件数据的电子邮件
*   CommandLineEventConsumer - 执行一个命令行程序

攻击者在响应他们的事件时，大量使用 ActiveScriptEventConsumer 和 CommandLineEventConsumer 类。这两个事件消费者为攻击者提供了极大的灵活性去执行他们想要执行的任何 payload 并且无需写入一个恶意的可执行文件或脚本到磁盘。

### 恶意的 WMI 持久化示例

图 5 中的 PowerShell 代码是修改过的存在于[SEADADDY](https://github.com/pan-unit42/iocs/blob/master/seaduke/decompiled.py#L887)恶意软件家族中的 WMI 持久性代码实例。该事件筛选器取自 PowerSploit 的持久性模块，目的是在系统启动后不久触发，事件消费者只需执行一个具有系统权限的可执行文件。图 5 中的事件筛选器在系统启动后的 200 和 320 秒之间被当作一个触发器。在事件被触发时事件消费者会执行已指定好的可执行文件。通过指定筛选器和一个 __FilterToConsumerBinding 实例将筛选器和消费者注册并将二者绑定在一起。

![](http://drops.javaweb.org/uploads/images/d5e50a6bd1e5e4392b327ad27c35e8abdafee3f5.jpg)

图 5 ：SEADADDY 恶意软件的 WMI 持久性 PowerShell 代码

0x08 WMI 攻击技术
=====

在攻击者的各个阶段的攻击生命周期中，WMI 都是极其强大的工具。系统提供了丰富的 WMI 对象、方法和事件，它们的功能极其强大，可以执行很多东西，从系统侦察、反病毒、虚拟机检测、代码执行、横向运动、隐蔽存储数据到持久性。它甚至可以打造一个纯粹的 WMI 后门且无需写入文件到磁盘。  
攻击者使用 WMI 有很多优势：

*   它被默认安装在所有的 Windows 操作系统中，并且可以追溯到 Windows 98 和 NT4.0。
*   对于执行代码，它可以隐蔽的运行 PSEXEC。
*   WMI 永久事件订阅是作为系统权限运行的。
*   防御者通常没有意识到 WMI 可以作为一个多用途的攻击向量。
*   几乎每一个系统操作都能够触发一个 WMI 事件。
*   除了在 WMI 存储库中存储之外不会对磁盘进行任何操作。

以下列表显示了几个如何使用 WMI 在攻击的各个阶段执行操作的例子。

系统侦察
----

许多恶意软件操纵者和渗透测试人员所做的第一件事情就是系统侦察， WMI 包含有大量的类可以帮助攻击者去感知他们的目标的环境。  
下面的 WMI 类是在攻击的侦察阶段可以收集数据的子集:

*   主机/操作系统信息:Win32_OperatingSystem, Win32_ComputerSystem
*   文件/目录列举: CIM_DataFile
*   磁盘卷列举: Win32_Volume
*   注册表操作: StdRegProv
*   运行进程: Win32_Process
*   服务列举: Win32_Service
*   事件日志: Win32_NtLogEvent
*   登录账户: Win32_LoggedOnUser
*   共享: Win32_Share
*   已安装补丁: Win32_QuickFixEngineering

反病毒/虚拟机检测
---------

### 杀毒引擎检测

已安装的 AV 产品通常会将自己注册在 WMI 中的 AntiVirusProductclass 类中的 root\SecurityCenter 或者是 root\SecurityCenter2 命名空间中，具体是哪一个命名空间则取决于操作系统的版本。

一个 WMI 客户端可以通过执行下面的 WQL 查询示例来获取已安装的 AV 产品:

**`SELECT * FROM AntiVirusProduct`**

如下图所示：

![](http://drops.javaweb.org/uploads/images/b3e188b89831e65a0fcbbb95e65fc5cecd0cbacf.jpg)

### 通用的虚拟机/沙盒检测

恶意软件可以使用 WMI 对通用的虚拟机和沙盒环境进行检测。例如，如果物理内存小于 2 GB 或者是单核 CPU ，那么很可能操作系统是在虚拟机中运行的。

WQL 查询示例如下：

**`SELECT * FROM Win32_ComputerSystem WHERE TotalPhysicalMemory < 2147483648`**

**`SELECT * FROM Win32_ComputerSystem WHERE NumberOfLogicalProcessors < 2`**

图 6 显示了使用 WMI 和 PowerShell 对通用的虚拟机进行检测的操作：

![](http://drops.javaweb.org/uploads/images/85e6d179c14d53cc41b4637c0c9f4c5bc020f9e3.jpg)

图 6 ：检测通用的虚拟机的 PowerShell 代码

### VMware 虚拟机检测

下面的查询示例试图查找 VMware 字符串是否出现在某些 WMI 对象中并且检查 VMware tools 的守护进程是否正在运行：

**`SELECT * FROM Win32_NetworkAdapter WHERE Manufacturer LIKE "%VMware%"`**  
**`SELECT * FROM Win32_BIOS WHERE SerialNumber LIKE "%VMware%"`**  
**`SELECT * FROM Win32_Process WHERE Name="vmtoolsd.exe"`**  
**`SELECT * FROM Win32_NetworkAdapter WHERE Name LIKE "VMware%"`**

图 7 演示的是使用 WMI 和 PowerShell 对 VMware 虚拟机进行检测的操作：

![](http://drops.javaweb.org/uploads/images/a9c077013ceac83923e3a29f691c4bb6175edbc9.jpg)

图 7 ：检测 VMware 虚拟机的 PowerShell 代码

代码执行和横向运动
---------

有两种常用的方法可以实现 WMI 的远程代码执行: Win32_Process 的 Create 方法和事件消费者。

### Win32_Process 的 Create 方法

Win32_Process 类包含一个名为`Create`的静态方法，它可以在本地或远程创建进程。这种情况下 WMI 就等同于运行 psexec.exe 一样，只是没有了不必要的取证操作，如创建服务。下面的示例演示了在远程机器上执行进程：

![](http://drops.javaweb.org/uploads/images/b29f3a8cf68bd314cbc6690f73037f3675feef2e.jpg)

一个更切实可行的的恶意使用案例是调用 Create 方法并且使用 powershell.exe 调用包含嵌入的恶意脚本。

### 事件消费者

实现代码执行的另外一个方法是创建一个 WMI 永久事件订阅。通常情况下， WMI 永久事件订阅被设计为对某些事件持续的做出响应。然而，如果攻击者想要执行一个 payload ，他们可能只需要配置事件消费者去删除其相应的事件筛选器、消费者和绑定到消费者的筛选器。这种技术的优点是 payload 是作为系统进程运行的并且避免了以明文方式在命令行审计中显示 payload。例如，如果采用了一个 VBScript 的 ActiveScriptEventConsumer payload，那么唯一创建的进程是以下 WMI 脚本宿主进程：

**`%SystemRoot%\system32\wbem\scrcons.exe -Embedding`**

作为攻击者，为了使用这个类作为攻击向量，将会遇到的挑战是去选择一个智能的事件筛选器。如果他们只是想要在几秒钟后触发运行 payload，那么可以使用`__IntervalTimerInstruction`类。攻击者可能会选择在用户的屏幕锁定后执行 payload，在这种情况下，可以使用外部的 Win32_ProcessStartTrace 事件作为创建 LogonUI.exe 的触发器。攻击者可以在他们选择的一个适当的事件筛选器中获得创意。

### 隐蔽存储数据

攻击者巧妙的利用了 WMI 存储库本身作为一种来存储数据的手段。其中一种方法是可以通过动态创建 WMI 类并可以将任意数据存储作为该类的静态属性的值。图 8 演示了将一个字符串存储为静态的 WMI 类属性的值：

![](http://drops.javaweb.org/uploads/images/557db4d0a30bec113d38cc09298447ae805eecbd.jpg)

图 8 ：创建 WMI 类的 PowerShell 代码示例

前面的示例演示了创建本地的 WMI 类。然而，也有可能可以创建远程的 WMI 类，这个将会在下一节进行说明。远程的创建和修改类的能力将使攻击者能够存储和检索任意数据，并将 WMI 变成 C2 的有效通道。

这取决于攻击者决定他们想用 WMI 存储库中存储的数据来做什么。接下来的几个例子阐述了攻击者如何利用此攻击机制的几个切实可行的例子。

使用 WMI 作为 C2 通道
---------------

使用 WMI 作为一种来存储和检索数据的机制，同样也可以使得 WMI 能作为一个纯粹的 C2 通道。这种使用 WMI 的聪明想法是由 Andrei Dumitrescu 在他[WMI Shell](http://2014.hackitoergosum.org/slides/day1_WMI_Shell_Andrei_Dumitrescu.pdf)工具中被首次公开——利用创建和修改 WMI 的命名空间作为 C2 的通道。

实际上还有很多 C2 暂存机制可以采用， 如刚才讨论过的 WMI 类的创建。同样也有可能使用注册表进行数据转储作为 WMI C2 的通道。下面的示例演示了一些利用 WMI 作为 C2 通道的 POC 代码。

![](http://drops.javaweb.org/uploads/images/d2687103946bd546c1c6137789282c113e5e94ee.jpg)

### “Push” 攻击

图 9 演示了如何远程创建 WMI 类来存储文件数据。之后可以远程使用 powershell.exe 将该文件数据写入到远程文件系统。

![](http://drops.javaweb.org/uploads/images/5e4d1b3ba96e60e463d722061a70a8d3742dcd29.jpg)

图 9 ： 远程创建 WMI 类并写入到远程文件系统的 PowerShell 代码

### “Pull” 攻击

图 10 演示了如何使用注册表来收取 PowerShell 命令的结果。此外，许多恶意工具试图捕捉只是将输出转换为文本的 PowerShell 命令的输出。本示例利用了 PowerShell 对象序列化和反序列化方法来保持目前在 PowerShell 对象中丰富的类型信息。

![](http://drops.javaweb.org/uploads/images/f806709321a0c9f4ca22d26071a4aaaa4c0a869f.jpg)

图 10 : 从 WMI 类属性拉回命令数据的 PowerShell 代码

0x09 WMI 提供程序
=====

提供程序是 WMI 的主干部分。几乎所有 WMI 类以及他们各自的方法都是在提供程序中实现的。提供程序是一个用户模式的 COM DLL 或者是一个内核驱动程序。每个提供程序都具有各自的 CLSID 用于在注册表中区别相关联的 COM 。此 CLSID 用于查找实现该提供程序的真正的 DLL 。此外，所有已注册的提供程序都有各自的 __Win32Provider WMI 类实例。例如，请思考以下已注册的处理注册表操作的 WMI 提供程序:

![](http://drops.javaweb.org/uploads/images/b98d9648d67240840a4814dc85879a543d6774c4.jpg)

通过引用以下注册表值来找到 RegistryEventProvider 提供程序对应的 DLL：

**`HKEY_CLASSES_ROOT\CLSID\{fa77a74e-e109-11d0-ad6e-00c04fd8fdff}\InprocServer32 : (Default)`**

![](http://drops.javaweb.org/uploads/images/860f771c385eedd2b68a4dae0d1f312148a9a8c1.jpg)

恶意的 WMI 提供程序
------------

WMI 提供程序仅用来向用户提供合法的 WMI 功能，因此，恶意的 WMI 提供程序就可以被攻击者用于扩展 WMI 的功能。

[Casey Smith](https://github.com/jaredcatkinson/EvilNetConnectionWMIProvider)和[Jared Atkinson](https://github.com/davehull/Kansa/)两人发布了恶意的 WMI 提供程序的 POC ，这些恶意的提供程序能够远程执行 Shellcode 和 PowerShell 脚本。恶意的 WMI 提供程序作为一种有效的持久性机制，它允许攻击者远程执行代码，只要攻击者拥有有效的用户凭据。