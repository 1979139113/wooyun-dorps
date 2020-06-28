# SSL/TLS协议安全系列：SSL的Padding Oracle攻击

0x00 前言
-------

* * *

在上一讲中，我们已经介绍了Padding Oracle攻击的背景知识和基本原理。简单来说，攻击就是利用密文解密后填充是否正确这一信息，以此得到明文的部分信息甚至恢复出全部明文。在这一讲里，我们会按照时间序梳理一遍与此攻击相关并影响SSL协议安全性的重要学术文章，简单介绍文章的贡献，给出其影响及对现实应用的指导意义。

0x01 Padding Oracle的学术线
-----------------------

* * *

1998年，贝尔实验室研究员瑞士密码学家Daniel Bleichenbacher在顶级密码学会议CRYPTO上发表了一篇名[Chosen Ciphertext Attacks Against Protocols Based on the RSA Encryption Standard PKCS#1](http://link.springer.com/chapter/10.1007/BFb0055716)的文章，这算是Padding Oracle攻击的开山之作。文章给出了可用于实际攻击那些使用了PKCS#1v1.5填充规范的RSA的协议的一种有效方法，这些协议中就包含了当时已被广泛使用的SSL协议。此外，这篇文章的学术意义还在于，它也第一次表明了考虑加密方案的适应性选择密文攻击安全性是有实际意义的，实际中可以发起选择密文攻击（对于使用1024bit密钥加密的密文，恢复出对应明文需要大约一百万次Oracle查询，作者对此的描述为though not entirely as efficient）。文中给出的解决方案是，不要使用PKCS#1 1.5版本中的填充方案，而是转而使用OAEP。

然而在2001年，James Manger在CRYPTO上发表了文章[A Chosen Ciphertext Attack on RSA Optimal Asymmetric Encryption Padding (OAEP) as Standardized in PKCS#1 v2.0](http://link.springer.com/chapter/10.1007/3-540-44647-8_14)，结果就是OAEP这种填充机制也是可以进行Padding Oracle攻击的，并且解密一条1024bit加密的密文，只需要大约1000次Oracle查询。这里需要额外提一句的是，SSL协议并不支持OAEP填充，所以目前TLS1.2中，使用的填充仍旧是v1.5中的方案。

以上都针对RSA的攻击，对应在SSL协议中，握手过程交换预主密钥的消息是攻击的点。而对于对称加密方案，首先提出Padding Oracle攻击的是Serge Vaudenay。在2002年EUROCRYPT上，在瑞士联邦理工大学（Swiss Federal Institute of Technologies，EPFL）领导安全及密码实验室（Security and Cryptography Laboratory，LASEC）的Serge Vaudenay教授在发表了名为[Security Flaws Induced by CBC Padding Applications to SSL, IPSEC, WTLS . EUROCRYPT 2002](http://www.iacr.org/cryptodb/archive/2002/EUROCRYPT/2850/2850.pdf)的文章，论文中表明Padding Oracle的攻击思想在对称密码领域也是可行的。这打开了Padding Oracle攻击的一扇新的大门，其意义从后面的文章直接将CBC模式下的Padding Oracle攻击称为Vaudenay攻击就可见一斑。

CBC模式下Padding Oracle的详细攻击原理和方法在上一讲中已给出了详细讲述，这里还要再提一下的是，在02年的这篇文章中，攻击者判断Padding是否正确的依据，是利用Padding检查不通过和消息验证码（Authentication）检查不通过时，SSL回复的Alert值不同这一特点。但其实Alert值是加密传送的，所以这个方法并不实际。尽管这样，在这之后，各个SSL相关的库都将Padding检查失败和Authentication检查失败返回的Alert值不做区分。

紧接着在03年，Serge Vaudenay在CRYPTO上发表了文章[Password interception in a SSL/TLS Channel](http://link.springer.com/chapter/10.1007/978-3-540-45146-4_34)，在文章中提出使用时间旁路信息来区分padding检查失败和authentication检查失败，这是因为在实现时，如果padding检查失败，就不会做authentication检查，因此padding不正确时，服务器的响应时间会比正确时要短，利用这个旁路信息，攻击者就可以获得了一个有效的padding oracle。

很自然地，各个库对应的响应方式就是无论padding检查正确或失败，都进行authentication检查，这样程序所表现出的时间差异就被弥补了。

这之后很长一段时间，对于SSL协议的Padding Oracle攻击因为无法在实际中获取到可用的oracle，就一直沉寂了。直到2013年，Lucky13攻击的出现才又将padding oracle攻击带回大众的视野，并且引发了很强烈的反响。下面我们就来详细介绍Lucky13攻击，看看它是如何又打开一个信息旁路信息通道，使得padding oracle再次生效。

这里需要多提一下的是，14年的POODLE攻击又将Padding Oracle攻击推到了风口浪尖，并宣告了SSLv3的死刑，但是这个攻击只是一个并没有利用其它的旁路信息，通过连接的继续与断开作为padding合法与否的反馈信息，在学术上没有新的贡献点，因此只是在工业界掀起了一阵风潮。

0x02 Lucky13攻击
--------------

* * *

下面我们先来简单回顾一下，Vaudenay攻击要求攻击者能够识别出何时填充是正确的，在TLS中，即使消息验证失败也要可以。通常情况下，验证应该在解密之前做，而TLS显然没有这么做，这就导致了在CBC模式下的Padding Oracle攻击。

Vaudenay最初的攻击利用的是在填充检查失败和验证失败时TLS会返回不同的警告值，但显然这无法构成实际攻击，因为警告值是加密传送的。随后，Vaudenay又发现可以利用时间旁路信息来识别填充是否正确，这是因为若填充不正确则不会执行验证，服务器所用的时间定会多于填充正确时的时间。修补方法就是不论填充检查是否成功，都进行验证。

但是在这种情况下，攻击者仍旧可以精心构造特定长度的消息，使得服务器在处理填充合法和不合法的情况下，所用的时间是不同的，也就是仍旧可以得到一条时间旁路通道进行攻击。这就是我们接下来要详细介绍的Lucky13攻击。

2013年，AlFardan和Paterson在顶级会议Security&Privacy上发表了文章[Lucky Thirteen: Breaking the TLS and DTLS Record Protocols](http://www.ieee-security.org/TC/SP2013/papers/4977a526.pdf)，文中提出了对于全部版本的SSL协议的一种Padding Oracle攻击方法。

文中利用的旁路信息，仍旧是服务器处理padding正确和不正确的消息时所用的时间，只不过这个时间的不同是因处理不同长度消息时进行消息验证过程中调用压缩函数的次数不同造成的。换句话说，压缩函数的调用次数与消息长度有关，而处理的消息长度值又与填充长度有关，因此填充的合法与否会间接导致压缩函数的调用次数不同，从而导致服务器处理消息的时间不同。

### 原理解析

假设使用TLS_RSA_WITH_AES_128_CBC_SHA密码套件，有一条待解密消息如下：

```
ContentType  ||   Version   ||   Length   ||   Civ||C1||C2||C3||C4 

|-----1字节-----|-----2字节----|-----2字节----|------16字节 * 5--------|

```

服务器收到消息后，将5字节的头消息去掉，解密Civ||C1||C2||C3||C4，解密后的消息长度为64字节，格式如下：

```
Message || MAC || Padding

```

在之前的设定中，为消除时间旁路信息，服务器在检查Padding后，不论Padding是否正确，都进行验证。因此服务器首先检查Padding，如果Padding合法，则去掉相应长度的Padding，否则，认为Padding长度为0。之后去掉20字节的MAC，然后验证MAC，去做HMAC的消息如下：

```
HMAC(     Seq     || ContentType || Version || Length_M || Message） 
         |--8字节--|-----1字节-----| --2字节--|---2字节----|

```

可以看到在Message前面附加了13字节的内容。下面我们重点关注计算HMAC过程中调用的压缩函数的次数。HMAC的计算如下：

```
HMAC(K, m) = H( (K ⊕ opad) || H((K ⊕ ipad)||m) )

```

若使用SHA1作为哈希函数，那么每个压缩函数处理的消息长度是64字节（下称为一个block）。 (K ⊕ ipad)是一个block，如果消息m的长度不是64的倍数，则要先对消息进行填充，填充机制如下：

```
m || 0x80 || 0x00 … 0x00 || Length(8字节)

```

可以看出，若消息需要填充，则至少填充9字节。

(K ⊕ ipad)是一个block，因此外层哈希会调用2次压缩函数，若m填充后是一个block（也就是m至多有55字节）时，则内层哈希函数调用2次压缩函数，HMAC共有4次调用；否则会共5次调用。

因此攻击者在看到消息ContentType || Version || Length || Civ||C1||C2||C3||C4后，可以实施如下攻击：

*   将C3的最后两个字节异或某一个值后发送给服务器解密
*   服务器解密后，Message||MAC||Padding共64字节，此时检查Padding：
    
    1.  如果Padding不合法（很大可能），则当做Padding长度为0，去掉20字节MAC，剩余44字节Message，Hmac的接收到m的长度为44+13=57字节，因此会调用5次压缩函数；
    2.  如果最后一个字节为0x00，那么padding合法，去掉Padding和MAC后Message长度为43字节，m长度为56字节，仍旧有5次压缩函数调用；
    3.  而如果Padding最后两个字节都为0x01的话，m的长度就刚好是55字节，只会有四次压缩函数调用。
*   攻击者将一次压缩函数调用的时间差作为旁路信息来区分Padding正确与否，进行常规的Padding Oracle攻击。
    

### 漏洞修补

* * *

很容易想到，针对Lucky13的修补，就是不论可能的Padding是什么，都要用恒定的时间计算MAC。实际中，不同的密码库有其不同的实现方法，大概可以分为两类：

*   加入dummy function处理，比如当padding不正确时，PolarSSL库就会计算需要额外执行的压缩函数数目，然后调用一个叫做md_process的函数;`for(j = 0; j < extra_run; j++)`
    
    md_process(&ssl->transform_in->md_ctx_dec, ssl->in_msg);
    
*   加入dummy data，直接执行压缩函数，使不论Padding正确与否，都执行可能的最大数目压缩函数调用，OpenSSL和Mozilla NSS就是采用了这种方式。
    

这两种方法单从时间上看，都成功封堵掉了时间旁路这个通道，但是对于利用dummy function的处理方式，若攻击者能够通过某种方式得到dummy function是否执行了的信息，则可以仍旧发起Lucky13攻击，将这个信息用作旁路信息，得到一个Padding Oracle。

0x03 Lucky13卷土重来？
-----------------

* * *

上一小节讲到，如果攻击者能够得到dummy function是否被执行的信息，Lucky13就还可以攻击成功。今年AsiaCCS上的一篇文章[Lucky 13 Strikes Back](http://dl.acm.org/citation.cfm?id=2714625)，就成功地得到了dummy function执行与否的信息，从而让Lucky13复活。

文章作者构建的攻击场景是在云中，攻击者与攻击目标是运行在同一台物理机器上的两个虚拟机。攻击者运用`Flush+Reload`的技术，获取到目标机执行dummy function的情况。从作者给出的结果来看，这个新的旁信息比原本Lucky13攻击利用的网络时间旁路信息要更加准确，平均恢复每个字节需要查询的次数要少。

下面就来简单说明一下这个关键的Flush+Reload技术。它是一种强有力的基于cache的旁路攻击方法。我们知道cache位于CPU和RAM之间，利用时空局域性原则（principle of locality），将最近访问的memory中的内容放到缓存中来，以减少平均访存时间。访问cache要比访问memory快很多。因此，Flush+Reload的大致做法就是先将特定的代码从cache中清除掉，等目标程序执行后，再执行那段代码，通过加载代码的时间长短来判断程序是否在cache中，若是，则说明目标程序也执行了这段代码。

此外，发起这个攻击需要满足以下两个要求：

1.  攻击者与目标机运行在同一台物理机器上
2.  物理机器上开启了内存去重（Memory Deduplication）特性，这是一种最初被作为Kernel Samepage Merging（KSM）引入Linux的内存优化技术，不同程序使用了相同的页会被指向同一块物理内存。

这个攻击相比原本的Lucky13攻击来说，不需要通过收到服务器的响应去测算所用时间，因此避免了很难处理的网络通道噪声，但是仍旧需要处理来自微架构的噪声（如指令预取等）。此外，对于补丁做得比较到位的库（如OpenSSL，Mozilla NSS等）此攻击无法奏效。目前确认受影响的库包括GnuTLS，PolarSSL和CyaSSL。

要使这个攻击失效，可以从两个方面入手，一是密码库的实现，对于填充合法和不合法的情况，采用同样的函数处理，并且保证同样的执行流（分支不依赖于消息）；另一方面可使Flush+Reload技术失效，比如可以关闭去重功能，对缓存进行分区，或者载入缓存时进行掩码（基于硬件)。

0x04 POODLE攻击
-------------

* * *

以上攻击讲完，似乎Padding Oracle攻击所需的padding oracle已无从寻找，从实现上来讲，在padding正确与否时，服务器都尽量做到了响应保持一致。但这并不意味着无法找到差别，毕竟服务器在收到正确的消息后，要完成正常的业务处理，而消息不对时则会拒绝服务。因此，找到一个特殊修改密文的方法使得服务器正常接受，同时最后一个字节可以预测，则就得到了一个可用的padding oracle。

14年的贵宾犬攻击用到的就是这个思想，但只能对sslv3攻击成功，是因为它利用了SSLv3消息的不确定填充机制，也就是填充的最后一个字节表示填充的长度，而对填充的其他部分则不做要求。对于密文Civ||C1||C2||C3||C4，若明文长度刚好为分组的倍数，也就是最后一个分组全部都是填充那么将C2到C4的位置，若替换后的密文被服务器接受，则可得知解密后最后一个字节，也就恢复除了C2所对应的明文的最后一个字节。

可见这个攻击的要求还是较为苛刻的，首先要保证最后一个分组全部都是填充，此外因只能恢复分组的最后一个字节，因此还要设法使待解密字节处在分组的最后一个字节，同时保证最后一个分组全部是填充。

附：[这里](http://seclist.us/poodle-attack-poc-implementation-of-the-poodle-attack.html)是一个可用的poc。

0x05 结语
-------

* * *

SSL的Padding Oracle攻击就介绍到这里，在下一讲中，我们会对SSL中弱PRNG带来的安全问题进行深入剖析，敬请期待。