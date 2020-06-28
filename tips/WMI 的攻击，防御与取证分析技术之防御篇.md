# WMI 的攻击，防御与取证分析技术之防御篇

近日，FireEye 安全公司的高级逆向工程团队（FLARE）发布了一份标题为《 WMI 攻击，防御与取证分析技术 》的 PDF 文档，该文档页数多达 90 页，文档内容主要从攻击，防御和取证分析这三个角度分篇对 WMI 技术做了详细的描述。其中不乏有很多值得学习思考的地方。于是，我利用业余时间翻译整理了此文档，拿出来与大家共分享 :），如有纰漏，望各位不吝赐教。

为了对原文档内容进行全面的翻译和解读，我按照文章的分析角度对原文档进行了分段式的翻译，本篇文章是分段式里面的第二篇，其余两篇译文的标题分别为：

*   **[《 WMI 的攻击，防御与取证分析技术之攻击篇 》](http://drops.wooyun.org/tips/9973)**
*   **《 WMI 的攻击，防御与取证分析技术之取证分析篇 》**

0x00 WMI 防御
=====

对于每一种 WMI 的攻击方式，都有相同数量的潜在防御措施。

现有的检测工具
-------

下列工具可以用来检测和删除 WMI 的持久性恶意攻击程序：

*   Sysinternals Autoruns
*   [Kansa](https://github.com/davehull/Kansa/)–– 一个用于事件响应的 PowerShell 模块

这些工具的缺点是只能在一个特定时间的快照里检测 WMI 持久性代码。一旦攻击者执行完他们的操作，他们就会清理掉相关的持久性代码。然而可以使用 WMI 永久订阅实时的捕捉攻击者的 WMI 持久性操作。

通过 EventConsumers 对 WMI 持久性的检测是微不足道的。图 11 中的 PowerShell 代码会查询远程系统上的所有 WMI 持久性项目。

![](http://drops.javaweb.org/uploads/images/8c7bc3c91c530cb6a66779962637c12778cf3299.jpg)

图 11：检测远程系统的 WMI 持久性的 PowerShell 代码

使用 WMI 检测 WMI 攻击
----------------

目前在 WMI 中有极其强大的事件处理子系统，因此， WMI 可以被认为是一个你从来不知道的并且已存在的微软的免费主机 IDS 。考虑到几乎所有的系统操作都可以触发 WMI 事件，所以，WMI 可以实时的捕捉许多攻击行为。请细想下面的攻击活动以及在 WMI 中它们各自的影响：

1.攻击者使用 WMI 作为持久性机制

*   影响:`__EventFilter`、`__EventConsumer`和`__FilterToConsumerBinding`的实例被创建。`__InstanceCreationEvent`事件被触发。

2.WMI Shell 工具集被用于 C2 的通道

*   影响:`__Namespace`对象的实例被创建和修改。因此，`__NamespaceCreationEvent`和`__NamespaceModificationEvent`事件被触发了。

3.创建 WMI 类来存储攻击者的数据

*   影响：`__ClassCreationEvent`事件被触发。

4.攻击者安装恶意的 WMI 提供程序

*   影响：`__Provider`类的实例被创建，`__InstanceCreationEvent`事件被触发。

5.攻击者通过 “开始菜单” 和 “注册表” 进行持续攻击

*   影响：`Win32_StartupCommand`类实例被创建，`__InstanceCreationEvent`事件被触发。

6.攻击者通过其他额外的注册表值进行持续攻击

*   影响：`RegistryKeyChangeEvent`和`RegistryValueChangeEvent`事件被触发。

7.攻击者安装服务

*   影响：`Win32_Service`类实例被创建，`__InstanceCreationEvent`事件被触发。

所有的攻击和上述描述的影响都可以用 WMI 事件查询来表示。在与事件消费者结合使用时，防御者可以极具创造力的选择如何检测和响应攻击者的操作。例如，防御者可以选择在创建 Win32_StartupCommand 的任意实例后接收一封电子邮件。

在创建了用于警报攻击者行为的 WMI 事件时，必须认识到攻击者也有可能很熟悉 WMI ，并且可以检查和删除现有的 WMI 防御事件订阅。因此，猫和老鼠的游戏就接着开始了。作为最终所采取的防御机制以防止攻击者删除你的防御事件订阅，你可以注册一个事件订阅来检测`__InstanceDeletionEvent`事件的`__EventFilter`，`__EventConsumer`和`__FilterToConsumerBinding`对象。然后，如果攻击者成功地删除了 WMI 永久防御事件订阅，防御者将会在删除时有最后一次机会得到攻击警报。

缓解措施
----

除了部署 WMI 永久防御事件订阅之外，还有几种可能防止部分或所有正在发生的 WMI 攻击的缓解措施。

1.  系统管理员可以禁用 WMI 服务。这对一个组织要考虑它对 WMI 的需要是非常重要的。一定要考虑清楚停止 WMI 服务后所带来的任何意外情况。因为，Windows 已经越来越依赖于 WMI 和 WinRM 的管理任务。
2.  考虑阻止 WMI 的协议端口。如果没有正当的理由需要远程使用 WMI，可以考虑配置 DCOM 使用[单一端口](https://msdn.microsoft.com/en-us/library/bb219447(v=vs.85).aspx)，然后阻止该端口。这是一个比较明智的缓解措施，因为它会阻止远程 WMI ，但允许该服务在本地运行。
3.  WMI、 DCOM 和 WinRM 的事件会被记录为下列事件日志：  
      a. Microsoft-WindowsWinRM/Operational  
        i. 显示含有来源 IP 地址的失败的 WinRM 连接尝试  
      b.Microsoft-Windows-WMI-Activity/Operational  
        i. 包含失败的 WMI 查询和可能包含攻击者活动证据的方法调用  
      c.Microsoft-WindowsDistributedCOM  
        i. 显示含有来源 IP 地址的失败的 DCOM 连接尝试

0x01 公共信息模型 (CIM)
=====

"公共信息模型 (CIM) 是一个开放的标准，它定义了如何将托管的元素在 IT 环境中表示为一组公共的对象以及它们之间的关系。分布式管理任务组维护 CIM ，允许这些托管的元素独立于其制造商或供应商的一致管理。WMI 使用 CIM 标准来表示其管理的对象。例如，系统管理员通过 WMI 查询的系统必须通过标准化的 CIM 命名空间来获取一个处理对象实例。

WMI 在 CIM 储存库中维护了一个所有可管理对象的登记册。CIM 储存库是一个持久性的数据库，它存储在运行 WMI 服务的本地计算机上。在使用 CIM 时，它维护了所有可管理对象的定义，包括它们之间的关系以及由谁来提供它们的实例。例如，当软件开发者通过 WMI 公开自定义应用程序的性能统计时，他们必须先注册性能指标的描述。这使得 WMI 能够正确解释查询并用良好的格式化数据来作响应。

CIM 是面向对象的，支持的特性有 (单一) 继承、抽象和静态属性、默认值并可以将任意键-值对附加到被称作"限定符"的项目中。

相关的类被分组式的放在分层命名空间之下。类声明的属性和方法通过可托管对象公开。属性是一个包含在类实例中的特定类型数据的名称字段。类定义描述了有关属性的元数据，类的实例包含了具体的由 WMI 提供程序填充的值。方法是一个已命名的例程，它在一个类的实例中执行，并且是由 WMI 提供程序实现的。类定义描述了其原型 (返回值的类型、 名称、 参数类型)，但没有实现过程。限定符是一个可以附加到命名空间、类、属性和方法的元数据的键-值对。常见的限定符提供了一个字符串类型的提示的值，以指示客户端如何去解释枚举项和语言代码页。

例如，图 12 列出了一些安装在纯净版 Windows 10 上的命名空间。请注意，它们可以很简单的被表示为树状。当客户端没有声明其本身的命名空间时， WMI 会选择 ROOT\CIMV2 命名空间作为默认的命名空间。

![](http://drops.javaweb.org/uploads/images/16f6a66e6f7bcafe458a7a3f2fd15a56b2453933.jpg)

图 12 命名空间 示例

在此安装的 Windows 中，ROOT\CIMV2 命名空间包含 1,151 个类定义。图 13 列出了一些在此命名空间中发现的类。请注意每个类具有一个名称和一个用于唯一标识类的路径。按照约定，某些类拥有名为 Description 的限定符，它包含了一个用以描述应如何托管这个类的人类可读字符串。 WMI Explorer 这个工具提供了用户友好的界面来获取 Description 限定符并在网格视图中显示它的值。

![](http://drops.javaweb.org/uploads/images/ea465766ec0896a0df9d03a3ae9b97b59141efae.jpg)

图 13 类 示例

图 14 列出了一些`Win32_LogicalDisk`类实例中已公开的属性。此类定义声明了 40 个属性， Win32_LogicalDisk 类实例包含了每个属性具体的值。例如，DeviceID 属性是字符串类型，用于磁盘的唯一标识，所以一个 WMI 客户端可以枚举类实例并期望得到像 A:、C:，D: 这样的值。

![ ](http://drops.javaweb.org/uploads/images/a5a0a68d7dc27708e6f9565bb1c742b76ea37167.jpg)

图 14 属性 示例

图 15 列出了`Win32_LogicalDisk`类实例中已公开的方法。此类定义声明了五种方法并且相关联的 WMI 提供程序允许客户端能够调用`Win32_LogicalDisk`实例的这些方法。面板底部的两个窗口描述了调用方法必须提供的参数和返回的数据。在此示例中，Chkdsk 方法需要五个布尔型参数并返回了一个描述操作状态的 32 位整数。请注意， Description 限定符附加到这些方法后其参数将作为 WMI 客户端开发者的 API 文档。

![](http://drops.javaweb.org/uploads/images/d1f6a45b77496da191f901aac9d01f2142c8270d.jpg)

图 15 方法 实例

在此安装的 Windows 中，有三个 Win32_LogicalDisk 类的实例。图 16 列出了使用它们唯一的实例路径的类实例。此路径是通过组合在类名称中具有限定符的特殊属性的名称和值所构成的。在这里，有一个属性: DeviceID 。每个类实例都是用来自同一个逻辑项目的具体数据填充的。

![](http://drops.javaweb.org/uploads/images/0e8417e5f49429787bcd8a69ec232a25b3d8c69e.jpg)

图 16 类实例 示例

图 17 列出了 Win32_LogicalDisk 类实例中 C: 磁盘卷的具体的值。注意，并非 40 个属性都列出在了这里;在类定义中，没有明确的值的属性也是没有默认值的。

![](http://drops.javaweb.org/uploads/images/787b215720789cc2059c5dafd908511524870f39.jpg)

图 17 Win32_LogicalDisk 类实例中 C: 磁盘卷的属性值

0x02 托管对象格式 （MOF）
=====

WMI 使用托管对象格式（MOF）用于描述 CIM 类的语言。MOF 文件是一个文本文件，包含了指定的可被查询的事物的语句如该事物的名称，复杂类的字段类型，对象组相关的权限。语言的结构类似于 Java，相当于受限的 Java 接口声明。系统管理员可以通过 WMI 使用 MOF 文件扩展 CIM 支持并且 mofcomp.exe 工具可以在 CIM 资料库中将被格式化的数据插入到 MOF 文件中。WMI 提供程序通常是由提供的 MOF 文件定义的，它定义了数据和事件类和提供数据的 COM DLL 文件。

MOF 是一种面向对象的语言，它由以下部分组成：

*   命名空间
*   类
*   属性
*   方法
*   限定符
*   实例
*   引用
*   注释

所有在 "公共信息模型 (CIM)" 章节中所阐述的实体都可以使用 MOF 语言进行描述。以下各章节将会说明如何使用 MOF 语言来描述 CIM 实体。

MOF 中的命名空间
----------

要在 MOF 中声明一个 CIM 命名空间，可以使用`#pragma namespace (\\computername\path)`指令。通常，这个语句被发现在文件的最开始，并且会应用到同一文件内剩余部分的语句。

MOF 语言允许通过声明父命名空间并定义一个 __namespaceclass 的新实例来创建新的命名空间。例如，我们可以使用在图 18 中列出的 MOF 文件创建`\\.\ROOT\default\NewNS`命名空间。

![](http://drops.javaweb.org/uploads/images/f431af45c665080db4cced36e6ff7df8e9529052.jpg)

图 18 在 MOF 中创建命名空间

MOF 中的类定义
---------

要在 MOF 中声明一个类，首先需要定义当前命名空间，然后再使用 class 关键字。提供新的类名和所继承的类。大多数类都有一个父类，新的 WMI 类的开发人员应该找到相应的要继承的类。接下来，描述新的类所支持的属性和方法。将限定符附加到类，属性和方法时，会有额外的元数据关联到一个实体上，如使用目的或枚举的解释。使用动态修饰符用于指示提供程序动态的创建类的实例。抽象类限定符用于指示该类没有实例可以创建。读取属性限定符用于指示该值是只读的。

MOF 支持程序员使用的常见的数据类型，包括字符串、数字类型 (uint8、sint8、uint16、sint16 等)、日期 (datetime) 和其他数据类型的数组。

图 19 列出了在 MOF 中定义类的语句的结构，而图 20 列出了一个定义了两个新类: ExistingClass 和 NewClass 的 MOF 示例文件。在`\\.\ROOT\default`命名空间中，可以找到这两个类。ExistingClass 类有两个属性：Name 和 Description。Name 属性的 Key 限定符指示了它被用于唯一标识此类的一个实例。NewClass 类有四个显式属性: Name, Buffer, Modified 和 NewRef。NewClass 也继承了其基类 ExistingClass 的 Description 属性。NewClass 被标记了动态限定符，这意味着相关联的 WMI 提供程序将会按需要动态创建此类的实例。NewClass 有一个名为 FirstMethod 的方法，该方法接受一个 32 位无符号整数的参数，并返回一个 8 位无符号的整数值。

![](http://drops.javaweb.org/uploads/images/1aa0db1164b4832ef708f2851814fe84977b84f7.jpg)

图 19 MOF 类定义结构

![](http://drops.javaweb.org/uploads/images/8e0107579569fa82f1b06876b9b8a270b46908cc.jpg)

图 20 在 MOF 中创建类定义

MOF中的实例
-------

在 MOF 中定义类的一个实例，需要在类名后使用 instance 关键字, 名称-值的键值对列表用来填充具体的属性值。图 21 列出了一个创建`\\.\ROOT\default\ExistingClass`类的实例的 MOF 文件，并分别给 Name 和 Description 属性提供了具体的值 SomeName 和 SomeDescription 。其余字段将会默认填充为 nil 。

![](http://drops.javaweb.org/uploads/images/f9305093a33912b329ce83473c28a191c912a8b0.jpg)

图 21 在 MOF 中创建一个类实例

MOF 中的引用
--------

CIM 类属性可以通过实例对象路径引用其他类的现有实例。这被称为引用。在 MOF 中定义类实例的引用，需要使用 ref 关键字作为属性的数据类型的一部分。例如，图 22 列出的 MOF 语句用于声明一个命名为 NewRef 的类引用，它指向了 ExistingClass 类的一个实例。

![ ](http://drops.javaweb.org/uploads/images/80c6c0b9636f86436de03b3977f50ea7b4b458d6.jpg)

图 22 在 MOF 中定义类的实例引用

若要设置引用属性的值，请将属性的值设置为实例对象路径用于标识现有的类实例。例如，图 23 列出的 MOF 语句可以将 NewRef 的属性设置为带有 Name 属性的 ExistingClass 类的实例，这等同于直接使用 SomeName 对 ExistingClass 类实例的 Name 属性进行赋值。

![](http://drops.javaweb.org/uploads/images/eb87272de159dd98d00111608145afda284b1f3c.jpg)

图 23 在 MOF 中设置类实例的引用

MOF 中的注释
--------

MOF 格式支持单行和多行 C 语言风格的注释。图 24 列出了各种风格的 MOF 语句定义的注释。

![](http://drops.javaweb.org/uploads/images/c815a3c90de9386d80fea340e5231470634911d9.jpg)

图 24 MOF 中的注释

　MOF 自动恢复　
----------

WMI 的 CIM 储存库实现了 MOF 文件的事务缓存式的插入，以确保数据库不会被破坏。如果在插入的时候系统发生崩溃或停止， MOF 文件可以被注册为后续自动恢复。若要启用此功能，可以在 MOF 文件顶部使用 #pragma autorecover 语句。这时，WMI 服务将会把 MOF 文件的完整路径添加到 MOF 文件自动恢复列表中，此列表存储在以下注册表项中：

**`HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\WBEM\CIMOM\Autorecover MOFs`**