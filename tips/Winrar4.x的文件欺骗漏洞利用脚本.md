# Winrar4.x的文件欺骗漏洞利用脚本

0x00 背景
-------

* * *

这几天仔细研究了winrar4.x系列的文件扩展名欺骗漏洞的那篇文章，通过一些测试对其有了一些新的想法和建议。（准确的说应该不能算文件扩展名欺骗漏，不止扩展名，整个文件名都是可以欺骗的）

具体的漏洞成因相信文章中都很清楚了，简单说一下：

zip格式中有2个filename，一般情况下，一般应用程序打开zip时，预览使用的是filename2，点击预览也是以filename2方式打开的，只有在解压的时候才会使用filename1。然而在winrar4.x中，点击预览是以预览filename1方式打开的。

这会造成什么结果呢？当第一个filename为readme.exe，第二个filename为readme.txt时，用winrar4.x打开时，你在程序窗口看到的文件名为readme.txt，然后你再点击文件时却是以readme.exe方式打开，这就形成漏洞了。

文章给出了如何利用这个bug的方法，更改filename2即可。但是作者是手动操作的，那么能不能写成利用脚本呢？这个filename2的长度有没有要求，需不需要和filename1长度相同？这正是本文要研究的。

0x01 细节
-------

* * *

在研究这个问题以前，先科普一下zip格式（想看详细版的去网上下载APPNOTE.TXT）。

zip格式由3部分组成：

```
1.文件内容源数据
2.目录源数据
3.目录结束标识结构

```

以只压缩了一个文件的zip文件为例，大致格式为：

```
[file header]
[file data]
[data descriptor]
[central directory file header]
[end of central directory record]

```

其中关键的几个字段为：

```
[file header]: 

Offset           Bytes               Description 
18                 4                   Compressed size 
26                 2                   File name length (n) 
28                 2                   Extra field length (m) 
30                 n                   File name 
30+n               m                  Extra field 


[central directory file header]: 

Offset           Bytes            Description 
28                 2                   File name length (n) 
30                 2                   Extra field length (m) 
34                 2                   File comment length (k) 


[end of central directory record]: 

Offset           Bytes                Description 
12                 4                   Size of central directory (bytes) 
16                 4                   Offset of start of central directory, relative to start of archive 

```

在了解了zip基本格式后，我对winrar压缩生成的zip文件和用windows生成的zip文件进行了分析，它们的区别是winrar的zip文件在Extra field区段都进行了一些数据填充。

![2014033020212771177.jpg](http://drops.javaweb.org/uploads/images/d5a73084c34c33ddc46aae364d731d1956363307.jpg)

由于不清楚Extra field这部分的值会不会影响到winrar的校验，所以根据不同情况做了几个测试，当filename2长度改变时，并且对受filename2长度影响的所有字段（除Extra field）进行修改后，文件可以正常打开。测试结果证明Extra field的值并不会影响winrar打开zip文件。

这样一来，只要按照zip的格式，更改和filename2有关的所有字段，就可以写出一个利用脚本了。

等等，该文章中同时提到了，这个漏洞存在有一个限制：解压。如果你是以右键解压打开这个压缩包的话，那么只会使用filename1，和filename2无关，也就不存在这个漏洞了。作者在文章最后提到了可以利用LRO解决这个限制，那应该如何结合利用RLO呢？

用WinHex对正常zip文件、使用了字符反转的zip文件进行分析：

![2014033020214732907.jpg](http://drops.javaweb.org/uploads/images/69c699c4d856e006970946e42bed8d406228e2de.jpg)

通过对比分析可以看到，当使用含有RLO文件名的文件进行压缩时，压缩的格式有点区别，继续做了几个测试，发现winrar在Extra field添加的信息，不会影响到漏洞的利用。

据此可以将这两个漏洞完美的结合在一起，写成一个利用脚本。

以python为例，具体思路为：

```
1．生成一个带LRO的文件名的文件，并用winrar压缩为zip。在python中可以使用u'\u202e'来构造字符串反转，用os.system()函数来执行winrar命令。 
2．处理zip文件中的数据，将filename2更改为自己需要定义的字符串。按照zip格式依次读取，修改filename2为新的字符串，计算出新的长度，并且修改File name length2字段，Sizeofcentraldirectory 和Offsetofstartofcentraldirectory字段，处理好它们新的偏移位置。 
3．重新生成新的zip。 

```

在文章最后附上完整的利用脚本WinrarExp.py

本程序只用于测试，仅供安全学习、研究所用，请勿用于非法用途，否则造成的一切后果自负。

使用方法：

```
WinrarExp.py [-f <open file>][-s <forged name>][-v <reversed string>] 

```

  
-f表示要压缩的文件，比如1.exe -s表示要伪装的文件名，比如readme.txt -v表示需要反转的字符串，该参数为选用。比如想要文件名反转变成readmeEXE.jpg则参数只要设置为EXE.jpg

下载地址：[WinrarExp.py](http://static.wooyun.org/20141017/2014101712290362071.zip)