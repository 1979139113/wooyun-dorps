# SSL/TLS协议安全系列：CBC 模式的弱安全性介绍(一)

编辑标注: 因drops对数学符号的支持存在问题，所以部分内容采取图片的形式显示，我们会经快解决这个问题，希望见谅。

一 前言
=====

在SSL的一些版本中，使用CBC分组模式对数据进行加密，而CBC分组模式在使用中常会出现一些问题。本文对CBC分组模式中Padding Oracle Attack做基本介绍，主要包括Padding Oracle Attack的背景知识、在多种填充模式下的攻击原理以及现实场景应用。

Padding Oracle Attack，顾名思义：Padding是“填充”，这里指在CBC分组加密模式中的填充数据；Oracle是“提示”，意指服务器返回的提示信息。Padding Oracle Attack可理解为“填充提示攻击”，是一种逻辑上的旁路攻击。简要来说，Padding Oracle Attack是指以明文分组和填充为根源，依据系统解密密文时泄露的填充信息，通过不断尝试填充字节直到获得有效填充信息，从而恢复出明文或者构造任意明文对应的密文这样一种攻击方式。利用Padding Oracle Attack，恢复一块长度为b字节的密文块，只需要平均128*b次oracle回应，而不是尝试2^k次（k指密钥长度为多少bit）。

Padding Oracle Attack最初是Serge Vaudenay在EUROCRYPT 2002会议上提出， Vaudenay介绍了CBC模式中基本的Padding Oracle Attack攻击原理；在 2010 年的 BlackHat 欧洲大会上，Juliano Rizzo 与 Thai Duong 把这种攻击应用到了一些流行的Web应用框架上，包括JavaServer Faces，Ruby on Rails 以及 ASP.NET，他们在2011年的Symposium on Security and Privacy大会上发表文章详细分析ASP.NET中的Padding Oracle问题；同年，在Pwnie Rewards中，ASP.NET的Padding Oracle漏洞被评为“最具价值的服务器漏洞”；次年，Romain Bardou和Riccardo Focardi等人发现，这种攻击对于一些安全设备同样有效；在 2014年，谷歌的安全人员又提出了针对 SSL v3协议中Padding Oracle漏洞的POODLE攻击，99%以上使用 https 服务的网站受到影响。

二 背景知识
=====

在介绍Padding Oracle Attack原理之前，这里首先简单介绍一些相关的背景知识。

CBC(Cipher Block Chaining)分组模式

常用的对称加密算法(如AES、3DES等)，一般使用分组加密模式，其中CBC是最常见的分组模式之一。CBC分组模式在对消息进行加/解密时，会根据使用的不同对称加密算法，将消息进行分块（block），每块大小可被分为16字节(AES)、8字节(3DES等)。

CBC分组模式加密流程如下图：

![enter image description here](http://drops.javaweb.org/uploads/images/1cd2b34774cdd17e592477da6e9d85389006ec70.jpg)

CBC分组模式中引入初始化随机向量IV，使得相同明文在不同的加密次数中产生不同的密文，IV随密文一起发送给接收方。整个加密过程可以表示为：

![enter image description here](http://drops.javaweb.org/uploads/images/9066b16e84d42246c830905ef6b2ff739c074bd6.jpg)

其中C表示密文块，P表示明文块。从加密过程中，我们可以看出，每一块明文都需要是整齐的，由于在现实中明文不一定可以整齐的分成块，因此在最后一块明文中需要加入填充(Padding)。

CBC分组模式解密流程图：

![enter image description here](http://drops.javaweb.org/uploads/images/14566bd033f0569beb518283080cfe2ecc70e7a9.jpg)

首先每块密文经过对称算法解密成中间值，之后同前一块密文异或得到明文。解密过程可以表示为：![enter image description here](http://drops.javaweb.org/uploads/images/724b10357979584d65aa177d0d3721766ea40059.jpg)根据CBC分组模式加/解密过程可以看出，假如改变C_i的任何一位，解密时会影响到P_i和P_(i+1)，同时，如果有一块密文改变，也只会影响到它对应的以及之后的一块明文的内容。

几种填充方式

前面我们提到，明文信息可以是任意长度，不一定是block的整数倍，这样难免明文分组后最后一块长度不足。解决的方法就是对最后一块内容进行填充，这里我们介绍几种常见的填充规范。

1.  PKCS#5

最常见填充规范是PKCS#5，PKCS#5标准的全称是“Public-Key Cryptography Standards”，在 RFC2898里有说明。简单的说，如果最后一块明文缺少N个字节，就用N这个数填充到完整的块。我们以8字节为一块的分组模式举例：

![enter image description here](http://drops.javaweb.org/uploads/images/52b324120abcd083ad3a9c3b53d92f25ceafa9a1.jpg)

1.  ISO/IEC 9797-1

PKCS#5只是常用的填充标准之一，ISO/IEC也提供了两种填充的标准，分别为ISO/IEC 9797-1和ISO/IEC 10118-1，这里我们具体介绍前一种填充标准，并在后面给出对应的Padding Oracle Attack方法。

ISO/IEC 9797-1标准有三种填充方式：

第一种在明文后面全部补‘0’，直到是block size的整数倍；

第二种在明文后面紧跟着的一位补‘1’，之后补‘0’，直到是block size的整数倍；

第三种方式：

![enter image description here](http://drops.javaweb.org/uploads/images/9dc90b6e354d358c1761c1af5be65af33886020e.jpg)

假设分组模式每块长度为n(bit)，首先右填充，在明文后面全部补充‘0’，直到n的整数倍，然后再进行左填充，(L_D)_2是未填充的明文长度(用bit表示)的二进制表示，然后在该长度左边补充‘0’直到长度为n。这种填充方式如果数据为空，仍然会有一个全‘0’的块作为填充。针对第三种填充方法，我们后面会给出对应的攻击原理。

上面介绍的两种标准，第一种是按字节填充，第二种是按位填充。除了这两种标准，还有很多其他的填充方式，比如John Black和Hector Urtubia在USENIX 2002年大会上提出的，这里我们以4字节填充为例介绍：

![enter image description here](http://drops.javaweb.org/uploads/images/7f91f893b7bd08ec64486e342f199c576de87272.jpg)

XY-PAD：选取固定的X和Y，在明文最后填充一个X，之后填充足够的Y；

ESP-PAD：在明文最后填充123..n补齐；

PAIR-PAD:与XY-PAD类似，这里的X和Y不是固定值，每次随机选取；

OZ-PAD：按位填充，在明文最后填充‘1’bit，之后以‘0’补齐；

ABIT-PAD：按位填充，发送者检测明文最后一位是‘0’还是‘1’，在最后以相反的位进行填充；

ABYT-PAD：发送者检测明文最后一个字节，然后选取任意不同的字节‘Y’进行填充；

BOZ-PAD：同XY-PAD，只不过X取的是0x80，Y取0x00。

三 攻击原理
=====

**1 攻击实施条件：**

Padding Oracle Attack依赖于两个假设，也就是实施攻击需要满足的先决条件：

*   攻击者可以窃听通信，并且拦截CBC分组模式的密文。
*   攻击者能够访问一个padding oracle O，并且能够区分返回的响应信息表示填充‘有效’或者‘无效’。

另一点是基于服务器的响应，服务器解密和验证padding时返回的不同信息：

*   当信息解密和填充都合法时，服务器会返回类似200 OK这样的有效信息；
*   如果解密过程中填充正确，但是消息内容不正确，也会返回200 OK；
*   如果出现填充错误，我们就会得到一个密码学异常和一个类似 500 Internal Server Error的响应。比如JSF view states填充错误时，会返回 javax.crypto.BadPaddingException: Given final block not properly padded。这就很明显的告诉我们此时填充不正确。

由此可见，攻击者能够很容易的根据错误消息判断填充是否有效，相当于能够访问一个padding oracle。

**2、如何解密**

(1)PKCS#5

这里我们首先以PKCS#5填充标准为例，介绍Serge Vaudenay在EURO-CRYPT 2002上提出的Padding Oracle Attack的基本原理。

首先给出几个符号定义，‘b’表示block的字节长度，比如AES中b=16，DES中b=8；N表示明文包含的block数量；在PKCS#5中，一个block序列x1, x2,……xN填充有效，是指xN最后一个字节为0x01，或者最后两个字节为0x02，0x02，以此类推。

<1 Last Word Oracle：

Last Word Oracle，能够获得任意密文块y最后一个字节对应的明文内容。如图所示：

![enter image description here](http://drops.javaweb.org/uploads/images/4fd60fc427ca3905e2492ca0177a6bf8c8263523.jpg)

![enter image description here](http://drops.javaweb.org/uploads/images/4461ce1255776bdd3b47e8f1a6925177dffc5abb.jpg)

当然也会出现填充为0x02,0x02或者0x03,0x03,0x03这样的情况，这时也会返回valid，此时从r的最左端一个字节更改，观察是否返回仍然为valid，如果是说明从这个字节往后都是填充，否则向右移动更改r的字节。Vaudenay给出了具体的实施过程：

![enter image description here](http://drops.javaweb.org/uploads/images/4f3e81f77e16dec643406ee943fa63456cf79d07.jpg)

这个过程的思想就是枚举‘IV’，密文y解密的中间值不变，通过枚举r，使得异或后的明文产生变化，根据填充方式的特征以及返回的Oracle信息，判断异或后明文最后字节的内容，从而获得中间值对应字节的内容，进一步与前一密文块对应字节异或得到y的最终明文。

<2 Block Decryption Oracle

获得密文块y最后一个字节明文内容后，可以通过修改r_b的值，使得C^(-1) (y)⊕r最后一个字节值为0x02，然后枚举r_(b-1)，如果得到O(r||y)=valid，说明倒数第二字节内容也为0x02，进一步得到倒数第二字节的内容，依此恢复出整个block的内容。

![enter image description here](http://drops.javaweb.org/uploads/images/461bf85320e0c4b640827ffa56c3a631a42eeffe.jpg)

以上Padding Oracle Attack攻击原理，是Vaudenay于EUROCRYPT 2002大会上提出，主要针对PKCS#5填充标准。下面介绍对于其他填充方式的Padding Oracle Attack。

(2) ISO/IEC 9797-1

ISO/IEC 9797-1第三种填充方式上面我们介绍过：

![enter image description here](http://drops.javaweb.org/uploads/images/6934b2cb2026bf3ebc75f4a17ff2cf95e2873a15.jpg)

针对ISO/IEC 9797-1第三种填充方式的Padding Oracle Attack，是Kenneth G. Paterson等在2004年CT-RSA大会上提出，攻击方式分为两个步骤：

<1 确定L_D

![enter image description here](http://drops.javaweb.org/uploads/images/97c2d23a54ecd20a81edc6448ddba7ee594f77a6.jpg)

**_第一种情况：填充完之后数据block数量q>=3（包括L_D）_**

在CBC分组模式中，改变密文块C_j中第i位，同时会影响解密后的明文P_(j+1)的第i位。当更改倒数第二块密文块C_(q-1)的任意一位，最后一块明文P_q对应的位也会改变。因为q>=3，所以倒数第二块不会是L_D对应的密文，L_D内容不会被修改。将更改后的密文发送给Oracle会有两种情况发生：

*   更改到的位是原始的非填充内容。填充信息仍然是完整正确的，这时Oracle会返回valid。
*   更改的位置是padding内容。这时根据L_D计算的填充信息会错误，Oracle返回invalid。

对于返回valid的情况，将更改位向右移动继续尝试，invalid情况，更改位向左移动，直到移动到填充有效和无效的边界，这时就可以得知填充的位数，从而计算出L_D的内容。

**_第二种情况：填充之后数据block数量q=2_**

![enter image description here](http://drops.javaweb.org/uploads/images/95de942fad7b14617976fb3eb44af939624bcb3b.jpg)

**_第三种情况：q=2, L_D=0或者L_D=n_**

构造![enter image description here](http://drops.javaweb.org/uploads/images/b74436aad21dd4e6bb1b1a95bb132f544b18107b.jpg)

当L_D=0时，未填充明文数据长度为3n，Oracle会返回invalid；L_D=n时，未填充明文数据长度为2n，Oracle会返回valid。从这种方法判断L_D内容。 <2>解密数据

![enter image description here](http://drops.javaweb.org/uploads/images/2cc29b1a297db3159370e6bcf0a861796be02b21.jpg)

![enter image description here](http://drops.javaweb.org/uploads/images/a1ad44d2fc5ae94c12856d635137253539913a7d.jpg)

![enter image description here](http://drops.javaweb.org/uploads/images/8847edd58e0286ddf0d12763ee602ac9a9d9d64e.jpg)

以上就是ISO/IEC9797-1标准第三种填充方式的解密原理。

John Black和Hector Urtubia提出的一些填充方式，可以使用类似PKCS#5的攻击方式进行解密，这里不再详细描述。

**3、CBC-R**

CBC-R是由John Black和Hector Urtubia 在Blackhat Europe 2010大会上提出，攻击者可以在不知道密钥K的情况下，构造任意明文对应的密文。

回顾一下CBC解密公式：![enter image description here](http://drops.javaweb.org/uploads/images/0dd6c680d2cf7d2ba167231237605addd1bc94d4.jpg)

如果攻击者获得了C_(i-1)和D_k (C_i)，他就可以控制明文P_i的内容。由于攻击者目的是获得任意明文P_i对应的密文C_i，那么对C_(i-1)可以做任意更改。攻击过程如下：

*   攻击者从截获的数据中任意选择C_i；
*   根据前面介绍的Block Decryption Oracle方法，获得D_k (C_i )；
*   异或D_k (C_i )和P_i就可以获得C_(i-1)的内容。

用这种方法，会使P_(i-1)的内容混乱，但是依此修改C_(i-2)的值可以构造P_(i-2)，依此类推构造整个密文块。具体算法如下：

![enter image description here](http://drops.javaweb.org/uploads/images/526135fa074a210b1d6dd0e04e80c06a6de6375c.jpg)

在CBC-R攻击中，会出现IV问题。攻击者在构造任意明文对应的密文时，依此类推最后需要修改C_0也就是IV的值，而在现实应用中，IV可能并不能被攻击者控制。IV在现实应用中可能有以下几种情况：

*   IV是密文的一部分，可以完全被攻击者控制；
*   IV是固定且公开的值，不能被攻击者修改；
*   IV保密且固定；
*   IV保密且会变化。

对于IV不受攻击者控制的情况，也有解决办法。

假如服务器希望收到头部(第一块明文)含有某些特定信息的数据，并且此时IV不能被控制，这时候可以牺牲中间某块数据的含义，获得有效的头部数据：

![enter image description here](http://drops.javaweb.org/uploads/images/0008e72ca6cc30e9dbe12e16039360644b99f5ad.jpg)

C_capture是攻击者截获的头部密文，这样解密后只是中间某个block失去意义。

四 现实应用
=====

**_1. POODLE攻击_**

CVE-2014-3566描述了POODLE(Padding Oracle On Downgraded Legacy Encryption)攻击。在SSL v3.0版本中，CBC分组模式填充方式为：·······N，最后一个字节表示填充内容的长度。这种攻击利用SSL v3.0版本对数据先认证后加密的漏洞，使得攻击者可以拿到服务器返回的填充有效（无效）信息，同时攻击者可以截获修改密文，满足实施Padding Oracle Attack的先决条件，从而利用Padding Oracle Attack获取明文内容。攻击者通过控制HTTP请求的路径和主体，使需要解密的敏感信息与分组块对齐，并将对应的密文块放在密文最后，通过多次发送该请求，获得该密文块最后一个字节内容，然后修改HTTP请求的路径，移动敏感数据下一个字节到密文块边缘。依此类推获得敏感信息。

**_2. Lucky 13_**

CVE-2013-0169描述了Lucky 13攻击。简单来说，该攻击利用服务器验证HMAC产生的时间差异，进行明文恢复。当服务器检测完CBC分组模式padding时，并不马上返回Oracle，此时会计算HMAC是否正确。而在计算HMAC时，会因为明文长度不同，计算时间也不同。攻击者通过修改密文，利用HMAC计算时间不同，猜测修改后的密文对应的明文的padding内容，依此获得明文信息。

对于Padding Oracle Attack的应用场景还有很多，比如ASP.NET，JSF view states等，这些很多资料中讨论较多，这里不再详细说明。

五 防范方法
=====

Padding Oracle Attack的实施条件中提到，攻击者需要截获修改密文内容，另外能够从服务器获取Oracle信息。在使用CBC分组模式时，如果先进行数据填充和加密，然后进行MAC完整性验证，解密之前先检测MAC，再检查padding，就不会出现padding oracle，这样就阻止了攻击者进行此类攻击。

在TLS1.2版本中，使用的是AEAD (Authenticated Encryption with Additional Data)，也就是AES-GCM模式，避免了使用CBC分组模式，同时避免了类似Padding Oracle Attack攻击。