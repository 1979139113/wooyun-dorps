# 近期js敲诈者的反查杀技巧分析

0x00 前言
=====

最近不少网友反映电脑中了敲诈者病毒（又名“Locky勒索软件”），电脑中的文档，图片等重要资料被病毒加密。此类病毒载体为js脚本，由js脚本下载远程服务器的pe文件，并使此pe文件在本地运行，从而完成对受害电脑数据的加密。

根据360安全中心监测，js敲诈者病毒主要通过网页挂马和钓鱼邮件两种途径进行传播，本文将对此类病毒的传播方法及反查杀技巧进行分析。

0x01 网页挂马传播
=====

黑客会利用部分网站的漏洞，将js敲诈者病毒植入到网页中，当用户访问带有恶意代码的网页时，电脑就会自动下载并执行此病毒。

![p1](http://drops.javaweb.org/uploads/images/39a6a73c5b234259f70325b439ceb7d68f7dd66d.jpg)图 1：样本1

如图1所示的样本，它利用十六进制对代码进行简单的加密，因此对它进行解密相对来说比较简单，解密后的代码如图2所示：

![p2](http://drops.javaweb.org/uploads/images/1b4f4689c732e74c1950e2a2bfa1436233095695.jpg)图 2：解密后的样本1

通过分析解密后的代码可以看出，它利用IE的ActiveX控件来获取远程的PE文件，执行过程包括下载文件，保存文件和运行文件三个步骤。它首先创建MSXML2.XMLHTTP对象来与远程服务器进行通信，获取服务器中的数据，然后使用创建的ADODB.Stream对象将获取的数据保存到用户的TEMP目录下，最后利用创建的WScript.Shell对象直接运行此文件。

样本1由于加密方式比较简单，所以很容易被杀毒软件查杀，为了进行反查杀，它的变种进行更加复杂的加密，如图3所示的样本就是目前比较流行的加密方式。

![p3](http://drops.javaweb.org/uploads/images/977a63b6913928a6ae6c1f673afcd620e6ef118d.jpg)图 3：样本2

首先它为字符串定义一个daughters函数，通过此函数来完成对字符串的截取。

![p4](http://drops.javaweb.org/uploads/images/e796c7ff491c9ededaf2b98ccac94c439d5b3565.jpg)图 4：字符截取操作

然后在代码中插入一些无意义的变量进行混淆，如图5所示的变量abeUtGplX、ojfdmCwgalh、yHoFUfYVm和GapGRiqoRoK就是起到混淆代码的作用。

![p5](http://drops.javaweb.org/uploads/images/a50c515158c24fdcd7ca157d111256d8fff23dfa.jpg)图 5：代码混淆

最后为了进一步达到免查杀目的，它将代码中将要使用的关键字定义到了一个数组nUvahxKnc中，它或者将关键字与和一些无意义的字符进行组合，或者将一个关键字拆分成几个不同的字符，在使用的时候再对字符进行拆分或者拼接操作。它还会在数组中插入一些无意义的字符进行代码混淆，而在脚本执行中动态地修改数组的长度从而去除那些无意义的字符，具体代码如图6所示。

![p6](http://drops.javaweb.org/uploads/images/9fd8b1018032e637c52ad2f4826907c021ec7a03.jpg)图 6：使用数组进行代码混淆

样本最终的解密结果如图7所示：

![p7](http://drops.javaweb.org/uploads/images/bee74bd284212df265865e08f7b326e8237363bf.jpg)图 7：解密后的样本2

0x02 邮件传播
=====

黑客通过社会工程学，利用人们的好奇心，精心构造一封钓鱼邮件，将js敲诈者脚本放入到邮件附件中，当用户双击运行js文件时便会中招，常见的邮件形式如下图所示。

![p8](http://drops.javaweb.org/uploads/images/d67ba805c1a18e9b43eb8c079648d7841e1bb763.jpg)图 8：钓鱼邮件1

![p9](http://drops.javaweb.org/uploads/images/f3509a5e9beaf7e0ea3de70c205c5c4bf82b9a01.jpg)图 9：钓鱼邮件2

为了达到反查杀的目的，钓鱼邮件1中的js文件首先用到了字符拆分和拼接的方法，这些方法由于前面已经提到，不再进行分析。

其次它将主要的恶意代码放到一个if条件表达式中，通过调用Date.getMilliseconds和WScript.Sleep函数来获取几个不同的毫秒数，然后通过判断这几个变量值是否相等来决定是否执行if条件中的内容，如图10。

![p10](http://drops.javaweb.org/uploads/images/5343207612a014ebb475f758556f685346d48e6a.jpg)图 10：if表达式

钓鱼邮件1中的样本解密后如图11所示。

![p11](http://drops.javaweb.org/uploads/images/995a01ede9046a2a6c8e0834d059ed00f19e561a.jpg)图 11：解密后的样本

通过分析加密后的代码，可以看出通过钓鱼邮件方式传播的js敲诈者和挂马传播方式不同的，它没有利用ActiveX控件来进行运行，而是选择使用WScript对象。由于windows操作系统中wscript.exe会为js脚本文件提供一个宿主环境，因此当邮件保持到本地后，双击js文件便会直接运行。

钓鱼邮件2中的样本是钓鱼邮件1的变种，为了达到进一步的反查杀效果，它在如下几个方面发生了变化。

首先它的if条件表达式发生了变化，它使用了`/*@cc_on @*/`条件编译，首先将Kcm赋值为false，然后在条件编译中将Kcm的值赋值为true，如果杀毒软件对此没有特殊处理的话，就很难检测到下面的内容，具体代码见图12。

![p12](http://drops.javaweb.org/uploads/images/0d4a897a17ca168b0d5dd89f225c27693a0593d8.jpg)图 12：变化后的if条件表达式

其次在计算毫秒时，它使用的不再是Date.getMilliseconds函数，而换成了Date.getUTCMilliseconds函数来进行计算，具体见图12。

钓鱼邮件2的样本解密后如图13所示。

![p13](http://drops.javaweb.org/uploads/images/c0cdc2f42b0561a0379bceb65d66d356110bdf1e.jpg)图 13：解密后的样本

还有一种类型的js样本，样本中经过加密的关键字需要使用特定的函数进行解密。如图14，样本中加密的值要调用adjurepe6函数才能进行解密，而它为了提高复杂性，在adjurepe6函数中需要再次调用btoa函数，只有经过这两个函数的解密才能得到最终的结果。

![p14](http://drops.javaweb.org/uploads/images/3a412312a0b99779735024c574fdc9ca46eea78e.jpg)图 14：使用函数加密

js代码使用escape函数进行加密而躲避查杀的情况也比较常见，此函数会同eval函数一起使用。使用时首先通过unescape函数对字符串进行解码，然后通过eval函数将字符串转化为js代码。此种手法在以前的js敲诈者样本中也经常碰到，但是以前是对整个js脚本进行加密，而现在它们只对部分代码进行加密，而未加密的部分又要使用加密部分定义的函数和变量，如图15和16所示。

![p15](http://drops.javaweb.org/uploads/images/d9411a5618071b3c00a88dcc46677721a3328cff.jpg)图 15：加密的代码

![p16](http://drops.javaweb.org/uploads/images/26748beee382782e1431bbda735eb52978612e5c.jpg)图 16：未加密的代码

可以看到detectDuplicates、matchesSelector、fragment和string函数中所使用的addHandle在未加密代码中是无法找到的，而加密代码解密后，可以看到addHandle值为eval。代码经过解密后，主要的代码如图17所示。

![p17](http://drops.javaweb.org/uploads/images/a89ad4956b2799147c21c4b939a620efc912cc7a.jpg)图 17：解密后的脚本

通过上面的分析可以看到为了达到反查杀的目的，js敲诈者使用了各种代码混淆和加密的方法，甚至使用了类似于`/*@cc_on @*/`条件编译的用法。由于这种病毒有经济上的利益，因此它们的更新速度特别快，360安全中心会密切关注这类病毒的最新动向，第一时间为用户提供有效的防护方案。