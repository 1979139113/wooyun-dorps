# 解密MSSQL链接数据库的密码

0x00 背景
-------

* * *

from:[https://www.netspi.com/blog/entryid/221/decrypting-mssql-database-link-server-passwords](https://www.netspi.com/blog/entryid/221/decrypting-mssql-database-link-server-passwords)

建议在看之前先了解下什么是linked server。比如参考微软的相关学习资料：

[http://technet.microsoft.com/zh-cn/library/ms188279.aspx](http://technet.microsoft.com/zh-cn/library/ms188279.aspx)

文章很多专有名词，而且老外说话的方式和中国的差别还是太大了点……所以有很多地方翻译的不是很好，见谅。

从重要系统中获取明文密码的过程总是充满着乐趣。MSSQL服务器对本地保存的密码是进行了加密操作的，链接服务器的相关凭证也不例外，但与此同时MSSQL也是有自己的手段可以对相关的密码凭证进行解密。而你可以直接参考使用本文发布的Powershell脚本对相关的凭证进行解密。以进攻者的角度来看，如果想要解密相关的凭证，我们需要有MSSQL的Sysadmin权限和本地服务器系统管理员的权限。从防御的层面来看，这篇文章的目的主要是想提醒下相关的管理员，不必要的数据库链接、高权限数据库链接和使用SQL server的身份认证会比使用集成的身份认证产生更多不必要的风险。这篇博文推荐给对数据库感兴趣的黑客及打算深入学习的管理员们。

0x01 链接服务器
----------

* * *

Microsoft SQL Server允许用户使用MSSQL管理其它不同类型的数据库，一般情况下链接服务器用来管理与本地不同版本的MSSQL服务。当链接建立之后，它们可以被配置为使用安全的上下文和静态sql server凭据。如果sql server的凭据被添加使用了，相关的用户名和密码会被加密存储到相关的表内，而这一加密是可逆的。单向不可逆的hash是不可以用于链接服务器的，因为sql server必须要使用明文的密码信息去访问其它的数据库。所以，如果密码信息是使用了对称加密而不是单向hash的话，sql server自然会有方法去解密相关的密文凭证。本文主要介绍这一加密、解密的过程及其工具化的实现。

0x02 链接服务器的密码存储方式
-----------------

* * *

MSSQL将链接服务器的信息（包含加密的密码）保存在master.sys.syslnklgns表中。我们重点关注的加密密码是保存在字段“pwdhash”中的（虽然写着hash但是他不是一个普通的hash），下图是一个例子：

![2014031313584244331.jpg](http://drops.javaweb.org/uploads/images/441846499cd6e9072eac4d8456da681e67158b4b.jpg)

master.sys.syslnklgns表在正常的数据库连接情况下是无法访问的，必须在专用管理员连接（DAC）下才可以访问（更多关于DAC的信息请查看[http://technet.microsoft.com/en-us/library/ms178068%28v=sql.105%29.aspx](http://technet.microsoft.com/en-us/library/ms178068%28v=sql.105%29.aspx)）。打开专用管理员连接有两个条件：一是需要有mssql的Sysadmin权限，二是本地服务器的管理员权限。

如果本地管理员没有获得Sysadmin的权限，你只需要将MSSQL权限修改为本地系统账户即可。更多信息请参考[https://www.netspi.com/blog/entryid/133/sql-server-local-authorization-bypass](https://www.netspi.com/blog/entryid/133/sql-server-local-authorization-bypass)

0x03 MSSQL加密
------------

* * *

下面介绍一些MSSQL加密的基本原理。首先我们可以先了解一下服务主密钥（SMK）（更多信息请参考[http://technet.microsoft.com/en-us/library/ms189060.aspx](http://technet.microsoft.com/en-us/library/ms189060.aspx)）。根据微软的描述“服务主密钥为 SQL Server 加密层次结构的根。

服务主密钥是首次需要它来加密其他密钥时自动生成的”可知SMK保存在master.sys.key_encryptions表中，而他的key_id对应的值是102。SMK是使用DPAPI来加密的，而这里他有两个可用的版本，第一种是使用的LocalMachine加密，另一种是CurrentUser的相关上下文（意思是SQL Server服务运行的账户）。这里我们挑使用LocalMachine的Machinekey来单独加密的、解密时不依赖于SQL Server账户的格式来讨论。下面是一个例子：

![2014031313592481620.jpg](http://drops.javaweb.org/uploads/images/3af3b2b49773557ac0addaa3ff57e82a176146d5.jpg)

为了增强加密的强度，算法中会加入熵（密码学用语，可自行查阅），不过算法中使用到的熵字节我们可以从注册表`HKLM:\SOFTWARE\Microsoft\Microsoft SQL Server\[instancename]\Security\Entropy`中找到。再次提醒，访问此表项需要本地系统的管理员权限。下图是熵的一个例子：

![2014031313595995821.jpg](http://drops.javaweb.org/uploads/images/f990764ab77dedaf8a5ad769d195324ecc7ff44b.jpg)

搞定上面这些之后（同时必须去除填充字节等）我们就可以使用DPAPI来解密SMK了。

0x04 解密链接服务器的密码
---------------

* * *

从SMK的长度（或MSSQL的版本）我们可以看出两种不同的加密算法：MSSQL 2012使用的是AES，而早期的版本使用的是3DES。另外，pwdhash必须解析为bit才可以找到相关的加密密码。版本使用的算法参考了高级T-SQL程序员发布的文章，见[http://stackoverflow.com/questions/2822592/how-to-get-compatibility-between-c-sharp-and-sql2k8-aes-encryption](http://stackoverflow.com/questions/2822592/how-to-get-compatibility-between-c-sharp-and-sql2k8-aes-encryption)

即使数据的格式和文章内给出的不太一致，但是也并不难发现正确的加密数据。到此，我们已经有办法使用SMK解密出所有的明文凭证了（当使用的是SQL Server账户而不是windows身份认证）。

0x05 使用脚本解密链接服务器的密码
-------------------

* * *

下面给出powershell的自动解密脚本Get-MSSQLLinkPasswords.psm1的源码链接（译者提示：该脚本必须在powershell2.0下运行）：

[https://github.com/NetSPI/Powershell-Modules/blob/master/Get-MSSQLLinkPasswords.psm1](https://github.com/NetSPI/Powershell-Modules/blob/master/Get-MSSQLLinkPasswords.psm1)

该脚本必须在MSSQL服务器本地运行（DPAPI必须可以访问到local machine key），同时运行该脚本的用户必须有数据库的sysadmin权限（在DAC下）可以访问所有的数据库实例，而且该账户也必须拥有系统管理器权限（用于读取注册表内的熵）。另外，如果启用了UAC，脚本必须以管理员身份运行。下面几个是脚本运行的关键步骤概要：

```
1. 获取本服务器所有MSSQL的实例
2. 为每个实例打开DAC访问
3. 获取每个实例的pwdhash字段值
4. 从master.sys.key_encryptions表中读取出所有key_id值为102的行所对应的SMK，并根据thumbprint字段来判断其版本
5. 读取HKLM:\SOFTWARE\Microsoft\Microsoft SQL Server\[instancename]\Security\Entropy 中的熵
6. 使用以上信息解密SMK
7. 程序根据MSSQL的版本和SMK的长度来确定使用AES算法或3DES算法
8. 使用SMK明文去解密链接服务器的凭证
9. 若程序运行成功则会回显相关的密码信息，比如下图就是一个例子：

```

![2014031314002642166.jpg](http://drops.javaweb.org/uploads/images/4d780bc51c6d4c8046ff1f1ad8a7ae0283eaf36e.jpg)

0x06 译者自测
---------

* * *

在测试这个的过程中很多问题的产生，要不是DAC登录不上，就是powershell的脚本一直有问题。

目前测试了两个环境：

```
1. win2003 + mssql2005
2. win2008 + mssql2008

```

除了脚本有问题，其它都是可以获取到的。脚本大部分语法是类似.NET的，基本都是在调用.NET的库，由于一些时间问题，我就不再献丑去改了等大牛们来一个“一键获取”。

先看点2003的测试图：

1.pwdhash

![2014031314004519325.png](http://drops.javaweb.org/uploads/images/062590cb0256b13c0ef74cf91f191b18968a59e2.jpg)

2.读取加密的SMK

![2014031314010729749.png](http://drops.javaweb.org/uploads/images/29ca610a2fbe3201d593eeb3bb0d1c4a6001f2f7.jpg)

3.查看注册表key

![2014031314012465783.png](http://drops.javaweb.org/uploads/images/eda7aa44ebc3984c3a522c4763d681f479f82d0f.jpg)

但是在运行脚本的时候（注意要2.0以上）还是报错了：

![2014031314014787183.png](http://drops.javaweb.org/uploads/images/8f3e4cd179b95c275d62c7bf9ea11f5d0fcc64b7.jpg)

已经测试过两个参数都非空，而且应该是正确的。

在2008下也是类似的情况。时间关系不再深入，等有兴趣的牛牛们再搞搞。

另外，这也是一种“找数据库连接信息”的方法。虽然有点奇葩，但是预计还是可行的，以后如果可以安排得过来，而且没有大牛修改上面的脚本的话……我再来献丑。

第一次搞翻译，翻译的不好大家多多原谅。