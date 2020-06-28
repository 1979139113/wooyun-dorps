# 威胁聚焦：CRYPTOWALL4

> 持续更新的恶意软件  
> 原文：[http://blog.talosintel.com/2015/12/cryptowall-4.html](http://blog.talosintel.com/2015/12/cryptowall-4.html)

0x00 摘要
=====

在过去的一年里，Talos花费了大量的时间去研究ransomware的运作原理，它与其他恶意软件的管理，还有它的经济影响。这项研究对开发检测方法和破坏攻击者的攻击有很大的价值。CrytoWall是一款恶意软件，在过去的一年里，先是升级成了CryptoWall2,随后升级为CryptoWall3。尽管大家都在努力检测和破坏它的攻击行为，但是这款恶意软件开发者还是很厉害的，改进了一些技巧，然后现在放出了CryptoWall4这个版本。

为了保证我们能最有效地检测出来，Talos对CryptoWall4进行了逆向，去理解它的行为，还有升级之后的差异，并放在这里分享我们的研究成果。

读者可能不熟悉，ransomware是一款恶意软件，它锁定用户的文件（比如照片，文档，音频文件等等），然后对其进行加密。用户要给赎金才能解密这些文件，查看里面的内容。通常，用户会通过钓鱼邮件感染ransomware。CryptoWall4的核心功能仍然没变，加密用户文件然后要赎金解密。然而，Talos发现了一些新的特性。比如加密算法变了，而且CryptoWall4加入了一项新的技术来禁用并删除了Windows所有的自动备份机制，没有外部的备份想自行恢复文件基本不可能了。

我们还发现CryptoWall4用了一些没有公开的API来获得入侵主机的本地语言设置。在后面的文章里，会更详细地阐述Talos发现的CryptoWall的新特性。

对于有经验的读者，我们建议您继续读下午。我们强烈建议用户和企业遵循安全规范，而且采用多层次的检测手段来规避风险。我们对新版本CryptoWall的深入分析，给了我们保护用户和确定更好的检测方案的技术支持。最后，针对FBI声明说的:用户没办法了还是支付赎金吧。Talos强烈建议用户们不要支付赎金，因为这样会直接让这项恶意行动受益。

0x01 感染
=====

CryptoWall的经营者用网络钓鱼和drive-by-download来将他们的恶意程序传播给用户。一旦CryptoWall4被成功运行，就会从C2服务去下载一个RSA公钥。然后所有的文件都被暂时地先被AES秘钥加密，之后就会被下载下来的RSA公钥加密。然后会用三种不同的方式告诉用户，你被加密了。第一种是一个文本文件，第二种是一张.png的图，第三种是一个HTML文档。所有的都会自动地从受害者的桌面打开。如下面所示。

![Alt text](http://drops.javaweb.org/uploads/images/f801e77a55ab7af097e969248c53e12b0e281d68.jpg)FigureA.1

![Alt text](http://drops.javaweb.org/uploads/images/f7d79d12c5e8560ee4a571e2c8e43b4b20d6267d.jpg)FigureA.2

![Alt text](http://drops.javaweb.org/uploads/images/5b57732b8e857d6f898bd0c4e22b1ec0123b240a.jpg)FigureA.3

![Alt text](http://drops.javaweb.org/uploads/images/724222e338cd416b596b9ce33c0e885cb9ecce9b.jpg)FigureA.4

![Alt text](http://drops.javaweb.org/uploads/images/e2633d206f1a26dbe38ff820334d9ec205e2c1c5.jpg)FigureA.5

一个有趣的发现是，一旦CryptoWall4不能从C2 server检索到RSA公钥，它就会不停地循环来下载公钥。只要公钥得不到，CryptoWall就不会破坏受害者的电脑。Talos同样观察到，这个样本也有一些另外的安全检查，比如如果受害者机器上的语言不受支持，就会自动终止感染进程。下面几种语言设置不会被感染：

_Russian, Kazakh, Ukrainian, Uzbek, Belarusian, Azeri, Armenian, Kyrgyz, Georgian._

很明显，攻击者想要排除一些地区，使之免受感染。

感染进程就如下图描述的一样：

![Alt text](http://drops.javaweb.org/uploads/images/942a842aab41d403b76adbd9a223e889eef0cea6.jpg)Figure B

从上图所示的 ”Delete all shadow copies“开始，CrytoWall4已经注入到了svchost进程。注入到这个进程是为了在受害者机器上获得更高的权限，以此来绕过UAC。有权限了才能静默地删除所有的系统备份，不然会有UAC提示用户来处理。

下图C描述的是其网络通讯，用的是HTTP协议，但是对payload进行了加密。如图D和图E

![Alt text](http://drops.javaweb.org/uploads/images/26394e7e706229ed0eccf888c923d1b29db77660.jpg)Figure C

![Alt text](http://drops.javaweb.org/uploads/images/ef7aedafa6e36782f55a1a4fc37c326dad024d54.jpg)Figure D

![Alt text](http://drops.javaweb.org/uploads/images/b4cddc1758db0e4cf0c33d7f69709823562ccc32.jpg)Figure E

CryptoWall4用了一个新的文件名生成算法来给加密后的文件命名：

1.  扫描磁盘驱动器，排除在白名单里的磁盘驱动器
2.  从目录下得到原始文件，然后跳过白名单里的文件名和扩展名
3.  产生随机的文件名长度（5-10）
4.  由随机的‘a-z’来构建文件名，长度是3得到的。（先得到0-1000的随机数，然后对26取模，再变成‘a-z’的字母）
5.  文件名字符串以Null结尾
6.  size/2和size中间取一个随机数num1。（size为文件名长度）
7.  在1个num1中去一个随机数num2（num2决定接下来往文件名字符串里插入随机数字的个数）
8.  获得一个随机的 0-9（char类型），插入到文件名的随机位置
9.  重复step8,num2次
10.  用同样的算法来生成扩展名，不过长度是2-5
11.  文件名后加上扩展名

CryptoWall4用的CRC32校验算法来排除一些目录，文件名和扩展名。下面是一些白名单：

**Extensions:**  
exe, dll, pif, scr, sys, msi, msp, com, hta, cpl, msc, bat, cmd, scf  
**Directories:**  
windows, temp, cache, sample ,pictures, default pictures, Sample Music, program files, program files (x86), games, sample videos, user account pictures, packages  
**Files:**  
help_your_files.txt, help_your_files.html, help_your_files.png, thumbs.db

完整的白名单在文章末尾Appendix A.

这些在白名单里的目录，文件名以及扩展名都是为了保证操作系统的稳定性。这意味着受害者可以继续用他们的电脑来支付赎金。任何被感染的用户都应该记住，下次开机，加密又会自动运行，然后把新创建的任何文件都加密一次。

生成新文件的文件名后，它的加密算法如图F所示：这也清楚地告诉我们，在CryptoWall4将文件加密之后，如果没有RSA私钥来还原AES秘钥,要还原文件，基本不可能。然而私钥只存在于攻击者的电脑上，而且不会传给用户的电脑。换句话说，用户不支付赎金获得私钥，就没办法还原文件。用户应该备份好自己的重要文件，才能保证遭受这种攻击后，不用支付赎金就能将其恢复。

![Alt text](http://drops.javaweb.org/uploads/images/ed9846836c77d00f654af77ebcfac6723d6ebe8d.jpg)Figure F

0x02 主体
=====

病毒程序主体被压缩，且被不同的壳保护了，壳里面有很多垃圾代码，无用的API调用，还有代码混淆技巧，比如用调用随机的API，参数还很奇怪。

![Alt text](http://drops.javaweb.org/uploads/images/f6eb5dbbf77fe3ba2b4a39b225ad77ea93c8d505.jpg)Figure G

第二层保护用的一种交错跳转形式的代码：

```
INSTRUCTION 1
INSTRUCTION 2
JNO nextSteo

```

0x03 解压缩代码
=====

主要的代码和之前版本的CryptoWall是非常相似的：首先构造自己的IAT,获得需要的系统调用并且创建自己的主事件对象来管理进程同步（名字是workstation的MD5）。这个事件有两个目标：在执行期间，阻止其余的CryptoWall4进程运行而且为感染有关的不同的进程实现同步。这写代码注入到一个新命名为“explorer.exe”进程里。注入到目标进程里的实际代码，用了这两种技术中的一种：

1.  ZwCreateSection和ZwMapViewOfSection
2.  ZwAllocateVirtualMemory , ZwWriteVirtualMemory 和 ZwProtectVirtualMemory

最后，代码被重新安置。用了两种不同的技术来进行注入：

1.  用ZwQueueApcThread这个内部API来将APC队列注入到目标进程
2.  经典的CreateRemoteThread方法

最终注入到新的 宿主“explorer.exr”进程里的代码，会被执行然后用CryptoWall4感染系统并且实现持续感染。程序主体被复制进%APPDATA%目录，然后在用户根目录的"Run"键值后添加进去，实现开机自启。

病毒程序会用一种现在还没被发现方法来禁止所有的系统还原点，和windows备份。首先调用SRRemoveRestorePoint，然后参数是0-1000,直到方法返回`ERROR_INVALID_DATA`。它将注册表`HKLM\Software\Microsoft\Windows Nt\SystemRestore`路径下的“DisableSR”键值设置为1，这样就完全禁止了系统还原。最后它开始执行删除存储备份的标准命令：

```
vssadmin.exe Delete Shadows /All /Quiet

```

接下来的流程在 “svchost.exe”进程里面。代码会重建IAT表，创建另一个事件（只在“svchost.ext”里使用），然后去形成并且打开自己的配置文件。这一步经常会失败，因为配置文件这时候还没有存在。dropper打开并解压位于自己内部的C&C URL列表（用的LZ压缩算法）。最后它尝试发广播去连接其中的某个 C&C server。

可以在IOC段找到恶意程序使用的C&C server的列表。

CryptoWall4的网络包比较特别，是下面这样的：

**| request Id | crypt7 | workstation MD5 [|subRequest Id 1|subRequest 1 Data| … ]**

在写这篇文章的时候，我们成功地了分出了五种不同的类型的包（request ID不同）。

```
1,3  --- Announcement packet --用来告诉C&C server有新的机器被感染了
7　--- Multi purpose packets。第一个sub-request ID用来区分不同类型的包：
 　 1 - Public key request  -用来向 C&C server请求一个新的公钥，为之后的加密做准备
 　 2 -  End announcement packet 用来告诉服务器感染结束了。另一个sub-request ID代表感染如何结束的详细信息：
　　　1　 - Success
　　2, 3 - Unsupported OS language packet - Exit 

```

这些包用标准的HTTPS协议送到网络上，但是在这之前就被加密了。加密算法很普通，用一个随机的字符串作为key,然后形成这样格式的数据流：

**|　Letter　|=|　encryption Key in Hex　|　encrypted stream　|**

例如：  
s=6975376e7a9b0fd24886fbd0c0de32d3ab4dd97174462ca3b06af16a1c840ae893eddacafbd93e56847c23a41352d4f45fc75468e4408

在广播进程成功连接到 C&C server后，就会请求获得公钥。这里有一个CryptoWall4的做的不足的地方：如果防火墙或者IPS足够好，能够拦截CryptoWall4的包，感染就进行不下去了。RSA-2048公钥请求包的request ID是7。C&C server返回的包这样子构成：

1.  付款URL的列表
2.  base64加密的RSA-2048公钥
3.  base64加密的PNG图片（图片由系统的语言设置决定）

![Alt text](http://drops.javaweb.org/uploads/images/d7b323158c60e3aa40c6e26ff2999496f4977420.jpg)Figure H.1

![Alt text](http://drops.javaweb.org/uploads/images/de1d771062b3e32775c79f14bef405adcc9d05d8.jpg)Figure H.2

![Alt text](http://drops.javaweb.org/uploads/images/d0bd7421ede9d65b2115385fef337c053cf697a3.jpg)Figure H.3

公钥用CryptStringToBinary API来解密。解密后的数据被存在一个全局变量中。HTML和text文件（用LZ 压缩算法压缩）由dropper释放出来，最终创建好配置文件并被加密，存储在`C:\Users\[Username]\AppData\Roaming\[Random 8 digits]`中。

CryptoWall4会确认配置文件的完整性，包含执行恶意代码所需要的全部信息。还会保证就算被中断了，恶意程序还是能继续加密文件。这个文件有很多连续的区段，开头是一个DWORD的值，来指定该区段的大小。

CryptoWall4在配置文件里，存储如下的信息：

*   收到的公钥的二进制流数据
*   与用户使用语言相符的HTML页面
*   与用户使用语言相符的text文件
*   与用户使用语言相符的PNG图片 （就是要让受害者看得懂）

文件加密进程完成后，最后三个文件会被写到感染者机器的所有目录下。

配置文件最终被压缩（LZ压缩算法，RtlCompressBuffer API，参数是2 COMPRESSION_FORMAT_LZNT1），然后为写入磁盘中。

一切顺利的话，主线程会被创建，之前的线程被终止（RtlExitUserThread）

0x04 主线程
=====

主线程从导入公钥开始，这会把加密的公钥的二进制的数据解析成能被Windows Crypto APIs识别的数据结构。CryptoWall4用的CryptDecodeObjectEx API来解析加密的公钥。这之后，二进制数据被转化为一个`CERT_PUBLIC_KEY_INFO`结构体。最后，新的数据结构被导入到Crypto APIs中，用的函数是CryptImportPublicKeyInfo，会返回一个xx句柄。然后会计算公钥的MD5,这一步非常重要，因为它会被用来检查受害者的文件是否已经被加密。

这之后，真正的加密进程启动。对每个逻辑分区，会有下面的检查：

```
LPWSTR pngFilePath = new TCHAR[MAX_PATH];
// This produces something like "C:\HELP_YOUR_FILES.PNG"
ComposePngPath(driveName, "HELP_YOUR_FILES.PNG", pngFilePath, MAX_PATH);
if (!FileExists(pngFilePath) == TRUE) {
 // Proceed with the encryption
 // … … …
}

```

一般来说，如果磁盘的根目录下含有`HELP_YOUR_FILES.PNG`文件，这块磁盘会被跳过。我们不知道这是一个bug还是它故意这么做的。对每个过滤掉的磁盘，一个新的加密线程会被启动（线程主函数的参数是一个小的结构体，一部分是公钥，一部分是磁盘名字符串的指针）

主线程会等待所有的加密进程完成。然后把那三个含有解密指示的文件放在两个位置：一是开始菜单的启动目录，而是桌面。

最后，一个end announcement包会被创建并发给C&C server。配置文件被删除，进程被终止（用的ZwTerminateProc）

![Alt text](http://drops.javaweb.org/uploads/images/e8e588f2771f3e094da93dc393751a1093bbbaf1.jpg)Figure I

0x05 加密线程
=====

加密线程有两个主要任务：首先它调用“DoFilesEncryption”将所有白名单之外的文件加密，最后它将`HELP_YOUR_FILES.PNG`写到根目录里。

DoFileEncryption会遍历目标磁盘目录下所有的文件夹和文件。

遇到子目录，会检查目录名字，进行CRC32检验，看是否在白名单里（这样，“windows”，“system32”，“temp”这样的文件夹被过滤掉）。而且检查`HELP_YOUR_FILES.PNG`是否存在，如果不存在，就继续调用DoFileEncryption,参数是当前的目录。

对文件的检查做了两次：扩展名和文件名。没有在白名单里面的，会调用EncryptFile来进行加密。

“EncryptFile”函数，是用来将目标文件加密的。“IsFileAlreadyEncrypted"函数会检查目标文件是否已经被加密：读取最开始的16字节，然后和公钥的MD5值作比较。

这个时候，恶意程序会生成随机的文件名和扩展名。下面是算法：（用的RtlRandomEx API获得每一个可打印字符）

```
// Generate a random value
DWORD GenerateRandValue(int min, int max) {
    if (min == max) return max;
    // Get the random value
    DWORD dwRandValue = RtlRandomEx(&g_qwStartTime.LowPart);
    DWORD dwDelta = max - min + 1;
    dwRandValue = (dwRandValue % dwDelta) + min;
    return dwRandValue;
}
// Generate a Random unicode string
LPWSTR GenerateRandomUString(int minSize, int maxSize) {
    DWORD dwStringSize = 0;             // Generated string size
    DWORD dwNumOfDigits = 0;            // Number of number letters inside the string
    LPWSTR lpRandString = NULL;         // Random unicode string
    // Generate the string size, and alloc buffer
    dwStringSize = GenerateRandValue(minSize, maxSize);
    lpRandString = new TCHAR[dwStringSize+1];
    for (int i = 0; i < (int)dwStringSize; i++) {
          DWORD dwLetter = 0;                       // Generated letter
          dwLetter = GenerateRandValue(0, 1000);
          dwLetter = (dwLetter % 26) + (DWORD)'a';
          lpRandString[i] = (TCHAR)dwLetter;
    }
    // NULL-terminate the string
    lpRandString[dwStringSize] = 0;
    // Now insert the digits inside the string
    DWORD dwUpperHalf = GenerateRandValue(dwStringSize / 2, dwStringSize);
    dwNumOfDigits = GenerateRandValue(1, dwUpperHalf);
    for (int i = 0; i < (int)dwNumOfDigits; i++) {
          DWORD dwValue = 0, dwPos = 0;       // Generated value and position
          dwValue = GenerateRandValue(0, 9) + (DWORD)'0';
          dwPos = GenerateRandValue(0, dwStringSize-1);
          lpRandString[dwPos] = (TCHAR)dwValue;
    }
    return lpRandString;
}
// Generate a random file name starting from a file full path
BOOLEAN GenerateRandomFileName(LPWSTR lpFileFullPath, LPWSTR * lppNewFileFullPath,
 LPWSTR * lppOrgFileName) {
    LPWSTR lpRandFileName = NULL;         // New random file name (without extension)
    LPWSTR lpRandExt = NULL;              // New random file extension
    LPWSTR lpNewFileName = NULL;          // The new file full name
    DWORD dwSize = 0;                     // size of the new filename
    // Check the arguments
    if (!lpFileFullPath || !lppNewFileFullPath || !lppOrgFileName)
          return FALSE;
    // Generate the new file name (without extension)
    lpRandFileName = GenerateRandomUString(5, 10);
    // Generate the random file extension
    lpRandExt = GenerateRandomUString(2,5);
    // Combine the new file name and extension and generate the final new file path
    // ....
    dwSize = wcslen(lpRandFileName) + wcslen(lpRandExt) + 1;
    lpNewFileName = new TCHAR[dwSize+1];
    swprintf_s(lpNewFileName, dwSize+1, L"%s.%s", lpRandFileName, lpRandExt);
    // ....
}

```

新的文件被创建，一个新的AES-CBC 256 key也通过调用CryptGenKey和CryptExportKey生成。这个32位的key会用来加密整个文件。

这时CryptoWall4采取了一个技巧：讲生成的AES key用从C&C server得到的RSA-2048公钥加密，这样就生成了一个256位的key,而且只能被攻击者解密。

RSA公钥的MD5值会写到被加密的文件头部16字节。然后CryptoWall4写入256位的加密字串。原始文件的属性和大小被写入接下来的8字节。原始文件名被得到的AES密钥加密，然后和文件大小一起，被写入到新的加密文件中。

这之后，开始真正的文件内容加密。原始文件每次被读512kb,存到一个大的数据块里。每个数据库会被加密密钥进行AES-CBC 256加密。然后直接写入到新文件里（开头四字节是块的大小）

完成后，CryptoWall4占用的所有资源被释放。原始文件被删除，这个过程比较有趣，见下面代码：

```
// Move the new encrypted file name in the old original position, replacing the old one

bRetVal = MoveFileEx(newEncFileName, lpOrgFileName,
MOVEFILE_WRITE_THROUGH | MOVEFILE_REPLACE_EXISTING);
if (!bRetVal)
// Delete the old file in the standard manner:
    DeleteFile(lpOrgFileName);
else {
    // Rename the original replaced file in the new random file name
    bRetVal = MoveFileEx(lpOrgFileName, newEncFileName, MOVEFILE_REPLACE_EXISTING);
}

```

从伪代码看得出来，存储原始文件的磁盘区被特殊地重写了，这样能保证数据恢复起来非常困难。这是恶意程序的作者来保证高付款率的新奇有趣的方法。减少数据恢复的可能，这样他们更好赚钱。下图是加密后的文件的结构：

![Alt text](http://drops.javaweb.org/uploads/images/0900852ad99ded5f8a502eaf610a7d1418d78bc3.jpg)Figure J

0x06 总结
=====

这篇分析里，我们细致分析了CryptoWall4。恶意程序没有任何创新性的技术，但是还是有几个技术亮点。缺陷是感染进程需要和C2 server进行交互。如果防火墙或者IPS能捕获它用来交互的数据包，感染进程就进行不下去了，因为它需要得到公钥才能加密受害者的文件。然而，一旦CryptoWall4加密了受害者的文件，不给攻击者支付赎金，就没有办法恢复私钥或者是解密文件。因为受害者的机器上得不到RSA私钥。私钥只存在于攻击者手里。

正如我们的分析展示的，CryptoWall的开发者不断在更新这款恶意软件，保证它对用户仍然是有效的。在威胁之上，企业需要认识到，攻击者会不断地改进这款恶意软件。用多层次的自我保护方法，能帮助企业监测到CryptoWall，阻止其威胁。Talos也会继续跟进CryptoWall的研究，找到更好的监测方法，然后为用户建立更好的防护体系。沃恩强烈建议用户和企业遵循安全规范，比如及时安装系统补丁，收到未知的三方信息的时候要谨慎，还要保证有一个给力的备份。这些措施能减少这些恶意程序的威胁，而且被攻击了也能有应急措施。

0x07 防护
=====

![Alt text](http://drops.javaweb.org/uploads/images/648fc459dc09e9ce99cd6a581c55bcbec9b1dba3.jpg)

*   高等级防护（AMP）能很好地阻止这类恶意程序的执行。
*   CWS或者WSA网络扫描能排除攻击者用来进行钓鱼等攻击的恶意网站。
*   IPS和NGFW保持最新来提供网络安全保护，并且 能检测恶意软件的行为。
*   ESA能截获带有恶意行为的email

0x08 others
=====

IOC DETAILS

这里可以下载IOCs[http://blogs.cisco.com/wp-content/uploads/cryptowall-4-iocs.txt](http://blogs.cisco.com/wp-content/uploads/cryptowall-4-iocs.txt)

**样本：**

```
3a73bb154506d8a9a3f4f658bac9a8b38d7590d296496e843503323d5f9b7801

```

**相似样本：**

```
2d04d2a43e1d5a6920a806d8086da9c47f90e1cd25aa99b95af182ee9e1960b3
bf352825a70685039401abde5daf1712fd968d6eee233ea72393cbc6faffe5a2
299b298b433d1cc130f699e2b5c2d1cb3c7e5eb6dd8a5c494a8c5022eafa9223

```

**威胁报告：**

[https://panacea.threatgrid.com/samples/d25f94dc4f2ac59e0428f54d685ead38](https://panacea.threatgrid.com/samples/d25f94dc4f2ac59e0428f54d685ead38)

**C2 URL 列表**

```
abelindia.com/1LaXd8.php
purposenowacademy.com/5_YQDI.php
mycampusjuice.com/z9r0qh.php
theGinGod.com/HS0ILJ.php
yahoosupportaustralia.com/8gX7hN.php
successafter60.com/iCqjno.php
alltimefacts.com/EiFSId.php
csscott.com/YuF59b.php
smfinternational.com/eRs70a.php
lexscheep.com/OIsSCj.php
successafter60.com/r_kfhH.php
posrednik-china.com/etdhIk.php
ks0407.com/VoZQ_j.php
stwholesaleinc.com/yL54uH.php
ainahanaudoula.com/GH09Dp.php
httthanglong.com/yzoLR7.php
myshop.lk/6872VF.php
parsimaj.com/60wEBT.php
kingalter.com/uVRfPv.php
shrisaisales.in/ZUQce4.php
cjforudesigns.com/E8B2gt.php
mabawamathare.org/WEAbCT.php
manisidhu.in/zJE0fD.php
adcconsulting.net/XEGeuI.php
frc-pr.com/dA91lI.php
localburialinsuranceinfo.com/zDJRc8.phpsmfinternational.com/AYNILr.php

```

**附录A**

```
Excluded files CRC32 Checksums
8E87F076h = help_your_files.txt
0A73B295Ch = help_your_files.html
11A8ACA3h = help_your_files.png
88068F93h
775DBED4h
60479578h 
7BD40679h = iconcache.db
48F43013h = thumbs.db
95ED794Ah
884F3F52h
7DAC63A1h
4208466h
0BA069E4Ch
0EC619E8Dh
9B0FD8B3h

Excluded extensions CRC32 Checksums 
6B63B6F0h = exe
3DD3B336h = dll
0BB5EA5C1h = pif
592D276Fh = scr
9E07ED22h = sys
8F3272A8h = msi
0A45BDDC1h = no three letter ext
0B65F578Ah = no three letter ext
0EB59DA68h = msp
64B6C6E6h = com
0C863AEB6h = hta
0DEEBF8EEh = cpl
6FE79BB6h = msc
9F9C299Fh = bat
2F5C1CC0h = cmd
43F7F312h = scf

Excluded directories CRC32 Checksums
0E3E7859Bh = windows
0B5385CAh = temp
0ED4E242h
9608161Ch
41476BE7h = cache
0F5832EB4h
0D8601609h
1DF021B7h
0B91A5F78h = sample pictures
0A622138Ah = default pictures
3FF79651h = sample Music
62288CBBh = program files
224CD3A8h = program files (x86)
72D480B3h
0FF232B31h = games
0A33D086Ah = sample videos
78B7E09h = user account pictures
9BB5C0A7h = packages
24FA8EBDh
```