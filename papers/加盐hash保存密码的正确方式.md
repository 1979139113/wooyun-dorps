# 加盐hash保存密码的正确方式

0x00 背景
-------

* * *

大多数的web开发者都会遇到设计用户账号系统的需求。账号系统最重要的一个方面就是如何保护用户的密码。一些大公司的用户数据库泄露事件也时有发生，所以我们必须采取一些措施来保护用户的密码，即使网站被攻破的情况下也不会造成较大的危害。保护密码最好的的方式就是使用带盐的密码hash(salted password hashing).对密码进行hash操作是一件很简单的事情，但是很多人都犯了错。接下来我希望可以详细的阐述如何恰当的对密码进行hash，以及为什么要这样做。

0x01 重要提醒
---------

* * *

如果你打算自己写一段代码来进行密码hash，那么赶紧停下吧。这样太容易犯错了。这个提醒适用于每一个人，**不要自己写密码的hash算法**！关于保存密码的问题已经有了成熟的方案，那就是使用[phpass](http://www.openwall.com/phpass/)或者本文提供的源码。

0x02 什么是hash
------------

* * *

```
hash("hello") = 2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824
hash("hbllo") = 58756879c05c68dfac9866712fad6a93f8146f337a69afe7dd238f3364946366
hash("waltz") = c0e81794384491161f1777c232bc6bd9ec38f616560b120fda8e90f383853542

```

Hash算法是一种单向的函数。它可以把任意数量的数据转换成固定长度的“指纹”，这个过程是不可逆的。而且只要输入发生改变，哪怕只有一个bit，输出的hash值也会有很大不同。这种特性恰好合适用来用来保存密码。因为我们希望使用一种不可逆的算法来加密保存的密码，同时又需要在用户登陆的时候验证密码是否正确。

在一个使用hash的账号系统中，用户注册和认证的大致流程如下：

```
1. 用户创建自己的账号
2. 用户密码经过hash操作之后存储在数据库中。没有任何明文的密码存储在服务器的硬盘上。
3. 用户登陆的时候，将用户输入的密码进行hash操作后与数据库里保存的密码hash值进行对比。
4. 如果hash值完全一样，则认为用户输入的密码是正确的。否则就认为用户输入了无效的密码。
5. 每次用户尝试登陆的时候就重复步骤3和步骤4。

```

在步骤4的时候不要告诉用户是账号还是密码错了。只需要显示一个通用的提示，比如账号或密码不正确就可以了。这样可以防止攻击者枚举有效的用户名。

还需要注意的是用来保护密码的hash函数跟数据结构课上见过的hash函数不完全一样。比如实现hash表的hash函数设计的目的是快速，但是不够安全。只有加密hash函数(cryptographic hash functions)可以用来进行密码的hash。这样的函数有SHA256, SHA512, RipeMD, WHIRLPOOL等。

一个常见的观念就是密码经过hash之后存储就安全了。这显然是不正确的。有很多方式可以快速的从hash恢复明文的密码。还记得那些md5破解网站吧，只需要提交一个hash，不到一秒钟就能知道结果。显然，单纯的对密码进行hash还是远远达不到我们的安全需求。下一部分先讨论一下破解密码hash，获取明文常见的手段。

0x03 如何破解hash
-------------

* * *

### 字典和暴力破解攻击(Dictionary and Brute Force Attacks)

最常见的破解hash手段就是猜测密码。然后对每一个可能的密码进行hash，对比需要破解的hash和猜测的密码hash值，如果两个值一样，那么之前猜测的密码就是正确的密码明文。猜测密码攻击常用的方式就是字典攻击和暴力攻击。

```
Dictionary Attack

Trying apple        : failed
Trying blueberry    : failed
Trying justinbeiber : failed
...
Trying letmein      : failed
Trying s3cr3t       : success!

```

字典攻击是将常用的密码，单词，短语和其他可能用来做密码的字符串放到一个文件中，然后对文件中的每一个词进行hash，将这些hash与需要破解的密码hash比较。这种方式的成功率取决于密码字典的大小以及字典的是否合适。

```
Brute Force Attack

Trying aaaa : failed
Trying aaab : failed
Trying aaac : failed
...
Trying acdb : failed
Trying acdc : success!

```

暴力攻击就是对于给定的密码长度，尝试每一种可能的字符组合。这种方式需要花费大量的计算机时间。但是理论上只要时间足够，最后密码一定能够破解出来。只是如果密码太长，破解花费的时间就会大到无法承受。

目前没有方式可以阻止字典攻击和暴力攻击。只能想办法让它们变的低效。如果你的密码hash系统设计的是安全的，那么破解hash唯一的方式就是进行字典或者暴力攻击了。

### 查表破解(Lookup Tables)

对于特定的hash类型，如果需要破解大量hash的话，查表是一种非常有效而且快速的方式。它的理念就是预先计算(pre-compute)出密码字典中每一个密码的hash。然后把hash和对应的密码保存在一个表里。一个设计良好的查询表结构，即使存储了数十亿个hash，每秒钟仍然可以查询成百上千个hash。

如果你想感受下查表破解hash的话可以尝试一下在[CraskStation](https://crackstation.net/)上破解下下面的sha256 hash。

```
c11083b4b0a7743af748c85d343dfee9fbb8b2576c05f3a7f0d632b0926aadfc
08eac03b80adc33dc7d8fbe44b7c7b05d3a2c511166bdb43fcb710b03ba919e7
e4ba5cbd251c98e6cd1c23f126a3b81d8d8328abc95387229850952b3ef9f904
5206b8b8a996cf5320cb12ca91c7b790fba9f030408efe83ebb83548dc3007bd

```

### 反向查表破解(Reverse Lookup Tables)

```
Searching for hash(apple) in users' hash list...     : Matches [alice3, 0bob0, charles8]
Searching for hash(blueberry) in users' hash list... : Matches [usr10101, timmy, john91]
Searching for hash(letmein) in users' hash list...   : Matches [wilson10, dragonslayerX, joe1984]
Searching for hash(s3cr3t) in users' hash list...    : Matches [bruce19, knuth1337, john87]
Searching for hash(z@29hjja) in users' hash list...  : No users used this password

```

这种方式可以让攻击者不预先计算一个查询表的情况下同时对大量hash进行字典和暴力破解攻击。

首先，攻击者会根据获取到的数据库数据制作一个用户名和对应的hash表。然后将常见的字典密码进行hash之后，跟这个表的hash进行对比，就可以知道用哪些用户使用了这个密码。这种攻击方式很有效果，因为通常情况下很多用户都会有使用相同的密码。

#### 彩虹表 (Rainbow Tables)

彩虹表是一种使用空间换取时间的技术。跟查表破解很相似。只是它牺牲了一些破解时间来达到更小的存储空间的目的。因为彩虹表使用的存储空间更小，所以单位空间就可以存储更多的hash。彩虹表已经能够破解8位长度的[任意md5hash](http://www.freerainbowtables.com/en/tables2/)。彩虹表具体的原理可以参考[http://www.project-rainbowcrack.com/](http://www.project-rainbowcrack.com/)

下一章节我们会讨论一种叫做“盐”(salting)的技术。通过这种技术可以让查表和彩虹表的方式无法破解hash。

0x04 加盐(Adding Salt)
--------------------

* * *

```
hash("hello")                    = 2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824
hash("hello" + "QxLUF1bgIAdeQX") = 9e209040c863f84a31e719795b2577523954739fe5ed3b58a75cff2127075ed1
hash("hello" + "bv5PehSMfV11Cd") = d1d3ec2e6f20fd420d50e2642992841d8338a314b8ea157c9e18477aaef226ab
hash("hello" + "YYLmfY6IehjZMQ") = a49670c3c18b9e079b9cfaf51634f563dc8ae3070db2c4a8544305df1b60f007

```

查表和彩虹表的方式之所以有效是因为每一个密码的都是通过同样的方式来进行hash的。如果两个用户使用了同样的密码，那么一定他们的密码hash也一定相同。我们可以通过让每一个hash随机化，同一个密码hash两次，得到的不同的hash来避免这种攻击。

具体的操作就是给密码加一个随即的前缀或者后缀，然后再进行hash。这个随即的后缀或者前缀成为“盐”。正如上面给出的例子一样，通过加盐，相同的密码每次hash都是完全不一样的字符串了。检查用户输入的密码是否正确的时候，我们也还需要这个盐，所以盐一般都是跟hash一起保存在数据库里，或者作为hash字符串的一部分。

盐不需要保密，只要盐是随机的话，查表，彩虹表都会失效。因为攻击者无法事先知道盐是什么，也就没有办法预先计算出查询表和彩虹表。如果每个用户都是使用了不同的盐，那么反向查表攻击也没法成功。

下一节，我们会介绍一些盐的常见的错误实现。

0x05 错误的方式：短的盐和盐的复用
-------------------

* * *

最常见的错误实现就是一个盐在多个hash中使用或者使用的盐很短。

#### 盐的复用(Salt Reuse)

不管是将盐硬编码在程序里还是随机一次生成的，在每一个密码hash里使用相同的盐会使这种防御方法失效。因为相同的密码hash两次得到的结果还是相同的。攻击者就可以使用反向查表的方式进行字典和暴力攻击。只要在对字典中每一个密码进行hash之前加上这个固定的盐就可以了。如果是流行的程序的使用了硬编码的盐，那么也可能出现针对这种程序的这个盐的查询表和彩虹表，从而实现快速破解hash。

**用户每次创建或者修改密码一定要使用一个新的随机的盐**

#### 短的盐

如果盐的位数太短的话，攻击者也可以预先制作针对所有可能的盐的查询表。比如，3位ASCII字符的盐，一共有95x95x95 = 857,375种可能性。看起来好像很多。假如每一个盐制作一个1MB的包含常见密码的查询表，857,375个盐才是837GB。现在买个1TB的硬盘都只要几百块而已。

基于同样的理由，千万不要用用户名做为盐。虽然对于每一个用户来说用户名可能是不同的，但是用户名是可预测的，并不是完全随机的。攻击者完全可以用常见的用户名作为盐来制作查询表和彩虹表破解hash。

根据一些经验得出来的规则就是盐的大小要跟hash函数的输出一致。比如，SHA256的输出是256bits(32bytes),盐的长度也应该是32个字节的随机数据。

0x06 错误的方式：双重hash和古怪的hash函数
---------------------------

* * *

这一节讨论另外一个常见的hash密码的误解:古怪的hash算法组合。人们可能解决的将不同的hash函数组合在一起用可以让数据更安全。但实际上，这种方式带来的效果很微小。反而可能带来一些互通性的问题，甚至有时候会让hash更加的不安全。本文一开始就提到过，永远不要尝试自己写hash算法，要使用专家们设计的标准算法。有些人会觉得通过使用多个hash函数可以降低计算hash的速度，从而增加破解的难度。通过减慢hash计算速度来防御攻击有更好的方法，这个下文会详细介绍。

下面是一些网上找到的古怪的hash函数组合的样例。

```
md5(sha1(password))
md5(md5(salt) + md5(password))
sha1(sha1(password))
sha1(str_rot13(password + salt))
md5(sha1(md5(md5(password) + sha1(password)) + md5(password)))

```

不要使用他们！

注意：这部分的内容其实是存在争议的！我收到过大量邮件说组合hash函数是有意义的。因为如果攻击者不知道我们用了哪个函数，就不可能事先计算出彩虹表，并且组合hash函数需要更多的计算时间。

攻击者如果不知道hash算法的话自然是无法破解hash的。但是考虑到[Kerckhoffs's principle](https://en.wikipedia.org/wiki/Kerckhoffs%27s_principle),攻击者通常都是能够接触到源码的(尤其是免费软件和开源软件)。通过一些目标系统的密码--hash对应关系来逆向出算法也不是非常困难。

如果你想使用一个标准的"古怪"的hash函数，比如HMAC，是可以的。但是如果你的目的是想减慢hash的计算速度，那么可以读一下后面讨论的慢速hash函数部分。基于上面讨论的因素，最好的做法是使用标准的经过严格测试的hash算法。

0x07 hash碰撞(Hash Collisions)
----------------------------

* * *

因为hash函数是将任意数量的数据映射成一个固定长度的字符串，所以一定存在不同的输入经过hash之后变成相同的字符串的情况。加密hash函数(Cryptographic hash function)在设计的时候希望使这种碰撞攻击实现起来成本难以置信的高。但时不时的就有密码学家发现快速实现hash碰撞的方法。最近的一个例子就是MD5，它的碰撞攻击已经实现了。

碰撞攻击是找到另外一个跟原密码不一样，但是具有相同hash的字符串。但是，即使在相对弱的hash算法，比如MD5,要实现碰撞攻击也需要大量的算力(computing power),所以在实际使用中偶然出现hash碰撞的情况几乎不太可能。一个使用加盐MD5的密码hash在实际使用中跟使用其他算法比如SHA256一样安全。不过如果可以的话，使用更安全的hash函数，比如SHA256, SHA512, RipeMD, WHIRLPOOL等是更好的选择。

0x08 正确的方式：如何恰当的进行hash
----------------------

* * *

这部分会详细讨论如何恰当的进行密码hash。第一个章节是最基础的，这章节的内容是必须的。后面一个章节是阐述如何继续增强安全性，让hash破解变得异常困难。

### 基础：使用加盐hash

我们已经知道恶意黑客可以通过查表和彩虹表的方式快速的获得hash对应的明文密码，我们也知道了通过使用随机的盐可以解决这个问题。但是我们怎么生成盐，怎么在hash的过程中使用盐呢？

盐要使用密码学上可靠安全的伪随机数生成器(Cryptographically Secure Pseudo-Random Number Generator (CSPRNG))来产生。CSPRNG跟普通的伪随机数生成器比如C语言中的rand(),有很大不同。正如它的名字说明的那样，CSPRNG提供一个高标准的随机数，是完全无法预测的。我们不希望我们的盐能够被预测到，所以一定要使用CSPRNG。下表提供了一些常用语言中的CSPRNG。

| Platform | CSPRNG |
| --- | --- |
| PHP | [mcrypt_create_iv](http://php.net/manual/en/function.mcrypt-create-iv.php),[openssl_random_pseudo_bytes](http://php.net/manual/en/function.openssl-random-pseudo-bytes.php) |
| Java | [java.security.SecureRandom](http://docs.oracle.com/javase/6/docs/api/java/security/SecureRandom.html) |
| Dot NET (C#, VB) | [System.Security.Cryptography.RNGCryptoServiceProvider](http://msdn.microsoft.com/en-us/library/system.security.cryptography.rngcryptoserviceprovider.aspx) |
| Ruby | [SecureRandom](http://rubydoc.info/stdlib/securerandom/1.9.3/SecureRandom) |
| Python | [os.urandom](http://docs.python.org/library/os.html) |
| Perl | [Math::Random::Secure](http://search.cpan.org/~mkanat/Math-Random-Secure-0.06/lib/Math/Random/Secure.pm) |
| C/C++ (Windows API) | [CryptGenRandom](http://en.wikipedia.org/wiki/CryptGenRandom) |
| Any language on GNU/Linux or Unix | Read from[/dev/random](http://en.wikipedia.org/wiki//dev/random)or /dev/urandom |

每一个用户，每一个密码都要使用不同的盐。用户每次创建账户或者修改密码都要使用一个新的随机盐。永远不要重复使用盐。盐的长度要足够，一个经验规则就是盐的至少要跟hash函数输出的长度一致。盐应该跟hash一起存储在用户信息表里。

**存储一个密码：**

```
1. 使用CSPRNG生成一个长的随机盐。
2. 将密码和盐拼接在一起，使用标准的加密hash函数比如SHA256进行hash
3. 将盐和hash记录在用户数据库中

```

**验证一个密码：**

```
1. 从数据库中取出用户的盐和hash
2. 将用户输入的密码和盐按相同方式拼接在一起，使用相同的hash函数进行hash
3. 比较计算出的hash跟存储的hash是否相同。如果相同则密码正确。反之则密码错误。

```

在本文的最后，给出了php,C#,Java,Ruby的加盐密码hash的实现代码。

**在web应用中，要在服务端进行hash：**

如果你在写一个web应用，可能会有在客户端还是服务端进行hash的疑惑。是将密码在浏览器里使用javascript进行hash，还是将明文传给服务端，在服务端进行hash呢？

即使在客户端用javascript进行了hash，在服务端依然需要将得到的密码hash再进行hash。如果不这么做的话，认证用户的时候，服务端是获取了浏览器传过来的hash跟数据库里的hash比较。这样子看起来是更安全了，因为没有明文密码传送到服务端。但是事实上却不是这样。

问题在于这样的话，如果恶意的黑客获取了用户的hash，就可以直接用来登陆用户的账号了。甚至都不需要知道用户的明文密码！也就不需要破解hash了。

这并不是说你完全不能在浏览器端进行hash。只是如果你要这样做的话，一定要在服务端再hash一次。在浏览器端进行hash是一个不错的想法，但是在实现的时候一定要考虑到以下几点：

1, 客户端密码hash并不是HTTPS(SSL/TLS)的替代品。如果浏览器和服务器之间的连接是不安全的，中间人(man-in-the-middle)可能通过修改网页的加载的javascript移除掉hash函数来得到用户的明文密码。

2, 有些浏览器可能不支持javascript，有些用户也会禁用javascript。为了更好的兼容性，需要检测用户的浏览器是否支持javascript，如果不支持的话就需要在服务端模拟客户端hash的逻辑。

3, 客户端的hash也需要加盐。一个很容想到的方式就是使用客户端脚本请求服务器或得用户的盐。记住，不要使用这种方式。因为这样恶意攻击者就可以通过这个逻辑来判断一个用户名是否有效。因为我们已经在服务端进行了恰当的加盐的hash。所以这里使用用户名跟特定的字符串(比如域名)拼接作为客户端的盐是可以的。

**使用慢速hash函数让破解更加困难: **

加盐可以让攻击者无法使用查表和彩虹表的方式对大量hash进行破解。但是依然无法避免对单个hash的字典和暴力攻击。高端的显卡(GPUs)和一些定制的硬件每秒可以计算数十亿的hash，所以针对单个hash的攻击依然有效。为了避免字典和暴力攻击，我们可以采用一种称为key扩展(key stretching)的技术。

思路就是让hash的过程便得非常缓慢，即使使用高速GPU和特定的硬件，字典和暴力破解的速度也慢到没有实用价值。通过减慢hash的过程来防御攻击，但是hash速度依然可以保证用户使用的时候没有明显的延迟。

key扩展的实现是使用一种大量消耗cpu资源的hash函数。不要去使用自己创造的迭代hash函数，那是不够的。要使用标准算法的hash函数，比如[PBKDF2](http://en.wikipedia.org/wiki/PBKDF2)或者[bcrypt](http://en.wikipedia.org/wiki/Bcrypt)。PHP实现可以在这里[找到](https://defuse.ca/php-pbkdf2.htm)。

这些算法采用了一个安全变量或者迭代次数作为参数。这个值决定了hash的过程具体有多慢。对于桌面软件和手机APP，确定这个参数的最好方式是在设备上运行一个标准测试程序得到hash时间大概在半秒左右的值。这样就可以避免暴力攻击，也不会影响用户体验。

如果是在web应用中使用key扩展hash函数，需要考虑可能有大量的计算资源用来处理用户认证请求。攻击者可能通过这种方式来进行拒绝服务攻击。不过我依然推荐使用key扩展hash函数，只是迭代次数设置的小一点。这个次数需要根据自己服务器的计算能力和预计每秒需要处理的认证请求次数来设置。对于拒绝服务攻击可以通过让用户登陆的时候输入验证码的方式来防御。系统设计的时候一定要考虑到这个迭代次数将来可以方便的增加或降低。

如果你担心计算机的能力不够强，而又希望在自己的web应用中使用key扩展hash函数，可以考虑在用户的浏览器运行hash函数。[Stanford JavaScript Crypto Library](http://crypto.stanford.edu/sjcl/)包含了PBKDF2算法。在浏览器中进行hash需要考虑上面提到的几个方面。

**理论上不可能破解的hash：使用加密的key和密码hash硬件**

只要攻击者能够验证一个猜测的密码是正确还是错误，他们都可以使用字典或者暴力攻击破解hash。更深度的防御方法是加入一个保密的key(secret key)进行hash，这样只有知道这个key的人才能验证密码是否正确。这个可以通过两种方式来实现。一种是hash通过加密算法加密比如AES，或者使用基于key的hash函数([HMAC](http://en.wikipedia.org/wiki/HMAC))。

这个实现起来并不容易。key一定要做到保密，即使系统被攻破也不能泄露才行。但是如果攻击者获取了系统权限，无论key保存在哪里，都可能被获取到。所以这个key一定要保存在一个外部系统中，比如专门用来进行密码验证的物理隔离的服务器。或是使用安装在服务器上特殊硬件，比如[YubiHSM](https://www.yubico.com/YubiHSM)。

强烈建议所有大型的服务(超过10万用户)的公司使用这种方式。对于超过100万用户的服务商一定得采用这种方式保护用户信息。

如果条件不允许使用专用验证的服务器和特殊的硬件，依然从这种方式中受益。大部分数据库泄露都是利用了[SQL注入技术](http://en.wikipedia.org/wiki/SQL_injection)。sql注入大部分情况下，攻击者都没法读取服务器上的任意文件(关闭数据库服务器的文件权限)。如果你生成了一个随机的key，把它保存在了一个文件里。并且密码使用了加密key的加盐hash，单单sql注入攻击导致的hash泄露并不会影响用户的密码。虽然这种方式不如使用独立的系统来保存key安全，因为如果系统存在文件包含漏洞的话，攻击者就可能读取这个秘密文件了。不过，使用了加密key总归好过没有使用吧。

需要注意使用key的hash并不是不需要加盐，聪明的攻击者总是会找到办法获取到key的。所以让hash在盐和key扩展的保护下非常重要。

0x09 其他的安全措施
------------

* * *

密码hash仅仅是在发生安全事故的时候保护密码。它并不能让应用程序更加安全。对于保护用户密码hash更多的是需要保护密码hash不被偷走。

即使经验丰富的程序也需要经过安全培训才能写出安全的应用。一个不错的学习web应用漏洞的资源是[OWASP](https://www.owasp.org/index.php/Main_Page)。除非你理解了[OWASP Top Ten Vulnerability List](http://owasptop10.googlecode.com/files/OWASP%20Top%2010%20-%202013.pdf),否则不要去写关系到敏感数据的程序。公司有责任确保所有的开发者都经过了足够的安全开发的培训。

通过第三方的渗透测试也是不错的方式。即使最好的程序员也会犯错，所以让安全专家来审计代码总是有意义的。寻找一个可信赖的第三方或者自己招聘一个安全人员来机型定期的代码审计。安全评审要在应用生命周期的早期就开始并且贯穿整个开发过程。

对网站进行入侵监控也十分重要。我建议至少招聘一名全职的安全人员进行入侵检测和安全事件响应。如果入侵没有检测到，攻击者可能让在你的网站上挂马影响你的用户。所以迅速的入侵检测和响应也很重要。

0x0A 经常提问的问题
------------

* * *

**我应该使用什么hash算法**

**可以使用**

1.  本文最后介绍的代码
2.  OpenWall的[Portable PHP password hashing framework](http://www.openwall.com/phpass/)
3.  经过充分测试的加密hash函数，比如SHA256, SHA512, RipeMD, WHIRLPOOL, SHA3等
4.  设计良好的key扩展hash算法，比如[PBKDF2](http://en.wikipedia.org/wiki/PBKDF2)，[bcrypt](http://en.wikipedia.org/wiki/Bcrypt)，[scrypt](http://www.tarsnap.com/scrypt.html)
5.  [crypt](http://en.wikipedia.org/wiki/Crypt_(Unix)#Library_Function_crypt.283.29)的安全版本。($2y$, $5$, $6$)

**不要使用**

1.  过时的hash函数，比如MD5,SHA1
2.  crypt的不安全版本。($1$, $2$, $2x$, $3$)
3.  任何自己设计的算法。

尽管MD5和SHA1并没有密码学方面的攻击导致它们生成的hash很容易被破解，但是它们年代很古老了，通常都认为(可能有一些不恰当)它们不合适用来进行密码的存储。所以我不推荐使用它们。对于这个规则有个例外就是PBKDF2,它使用SHA1作为它的基础算法。

**当用户忘记密码的时候我应该怎样让他们重置**

在我个人看来现在外面广泛使用的密码重置机制都是不安全的，如果你有很高的安全需求，比如重要的加密服务，那么不要让用户重置他们的密码。

大多数网站使用绑定的email来进行密码找回。通过生成一个随机的只使用一次的token，这个token必须跟账户绑定，然后把密码重置的链接发送到用户邮箱中。当用户点击密码重置链接的时候，提示他们输入新的密码。需要注意token一定要绑定到用户以免攻击者使用发送给自己的token来修改别人的密码。

token一定要设置成15分钟后或者使用一次后作废。当用户登陆或者请求了一个新的token的时候，之前发送的token都作废也是不错的主意。如果token不失效的话，那么就可以用来永久控制这个账户了。Email(SMTP)是明文传输的协议，而互联网上可能有很多恶意的路由器记录email流量。并且用户的email账号也可能被盗。使token尽可能快的失效可以降低上面提到的这些风险。

用户可能尝试去修改token，所以不要在token里存储任何账户信息。token应该是一个不能被预测的随机的二进制块(binary blob)，仅仅用来进行识别的一条记录。

永远不要通过email发送用户的新密码。记得用户重置密码的时候要重新生成盐，不要使用之前旧密码使用的盐。

**如果我的用户数据库泄露了，我应该怎么办**

第一要做的就是弄明白信息是怎么泄露的，然后把漏洞修补好。

人们可能会想办法掩盖这次安全事件，希望没有人知道。但是，尝试掩盖安全事件会让你的处境变得更糟。因为你不告知你的用户他的信息和密码可能泄露了会给用户带来更大的风险。一定要第一时间通知用户发生了安全事件，即使你还没有完全搞明白黑客到底渗透到了什么程度。在首页上放一个提醒，然后链接到详细说明的页面。如果可能的话给每一个用户发送email提醒。

向你的用户详细的说明他的密码是如何被保护的，希望是加盐的hash，即使密码进行了加盐hash保护，攻击者依然会进行字典和暴力攻击尝试破解hash。攻击者会使用发现的密码尝试登陆其他网站，因为用户可能在不同的网站都使用了相同的密码(所谓的撞库攻击)。告知你的用户存在的这些风险，建议他们修改使用了相同密码的地方。在自己的网站上，下次用户登陆的时候强制他们修改密码。大部分用户可能会尝试使用相同的密码，为了方便。要设计足够的逻辑避免这样的情况发生。

即使有了加盐的hash，攻击者也可能快速破解一些很弱的弱密码。为了降低这种风险，可以在使用正确密码的前提下，加一个邮件认证，直到用户修改密码。

还要告知你的用户有哪些个人信息存储在网站上。如果数据库包含信用卡信息，你需要通知你的用户注意自己近期的账单，并且最好注销掉这个信用卡。

**应该使用怎样的密码策略，需要强制使用强密码么**

如果你的服务不是有很严格的安全需求，那么不要限制你的用户。我建议在用户输入密码的时候显示它的强度等级。让用户自己决定使用什么强度的密码。如果你的系统有很强的安全需求，那么强制用户使用12位以上的密码，至少包含2个数字，2个字母，2个字符。

每6个月最多强制用户修改一次密码。超过这个次数，用户就会感到疲劳。他们更倾向于选择一个弱密码。更应该做的是教育你的用户，当他们感到自己的密码可能泄露的时候主动修改密码。

**如果攻击者获取了数据库权限，他不能直接替换hash登陆任意账户么**

当然，不过如果他已经或得了数据库权限，很可能已经可以获得服务器上的所有信息了。所以没有什么必要去修改hash登陆别人账户。进行密码hash的目的不是保护网站不被入侵，而是如果入侵发生了，可以更好的保护用户的密码。

在SQL注入攻击中，保护hash不被替换的方式使用两个用户不同权限的用户连接数据库。一个具有写权限，另外一个只具有只读的权限。

**为什么需要一些特别的算法比如HMAC，而不是直接把密码和加密key拼接在一起**

(这部分讲一些密码学的原理，翻译的不好请见谅)

hash函数，比如MD5,SHA1,SHA2使用了[Merkle–Damgård construction](http://en.wikipedia.org/wiki/Merkle%E2%80%93Damg%C3%A5rd_construction)，这导致算法可能长度扩展攻击(length extension attacks)。意思就是说给定一个hash H(X)，攻击者可以在不知道X的情况下，可以找到一个H(pad(X)+Y)的值，Y是个其他的字符串。pad(X)是hash函数使用的填充函数(padding function)。

这就意味者，对于hash H(key + message)，攻击者可以计算 H(pad(key + message) + extension)，并不需要知道加密key。如果这个hash是用在消息认证过程中，使用key为了避免消息被修改。这样的话这个系统就可能失效了，因为攻击者掌握了一个有效的基于 message+extension的hash。

这种攻击对于如何快速破解hash还不是很清楚。但是，基于一些风险的考虑，不建议使用单纯的hash函数进行加密key的hash。也许一个聪明的密码学家一天就可以找到使用这种攻击快速破解hash的方法。所以记得使用HMAC。

**盐应该拼在密码的前面还是后面**

这个不重要。选择一个并且保持风格一致就行了。实际中，把盐放在前面更常见一点。

**为什么本文最后提供的hash代码使用了固定执行时间的函数来比较hash(length-constant)**

使用固定的时间来比较hash是为了防止攻击者在线上的系统中使用基于时间差的攻击。这样攻击者就只能线下破解了。

比较两个字符串是否相同，标准的方式是先比较第一个字节，然后比较第二个字节，一次类推。只要发现有一个字节不同，那么这两个字符串就是不同了。可以返回false的消息了。如果所有字节比较下来都一样，那么这两个字符串就是相同的，可以返回true。这就意味了比较两个字符串，如果他们相同的长度不一样，花费的时间不一样。开始部分相同的长度越长，花费的时间也就越长。

基于这个原理，攻击者可以先找256个字符串，他们的hash都是以不同的字节开头。然后发送到目标服务器，计算服务器返回的时间。时间最长的那一个就是第一个字节hash是正确的。依次类推。攻击者就可能得到hash更多的字节。

这种攻击听起来好像在网络上实现起来比较困难。但是已经有人[实现](https://crypto.stanford.edu/~dabo/papers/ssl-timing.pdf)过了。所以我们在比较hash的时候采用了花费时间固定的函数。

**本文提供的代码中 slowequals 函数是怎么工作的**

上一回答讲到了我们需要比较时间固定的函数，这部分详细讲一下代码的实现。

```
private static boolean slowEquals(byte[] a, byte[] b)
{
    int diff = a.length ^ b.length;
    for(int i = 0; i < a.length && i < b.length; i++)
    diff |= a[i] ^ b[i];
    return diff == 0;
}

```

这段代码使用了异或(XOR)操作符"^"来比较整数是否相等，而没有使用"=="操作符。原因在于如果两个数完全一致，异或之后的值为零。因为`0 XOR 0 = 0, 1 XOR 1 = 0, 0 XOR 1 = 1, 1 XOR 0 = 1`。

所以，第一行代码如果a.length等于b.length，变量diff等于0,否则的话diff就是一个非零的值。然后，让a，b的每一个字节XOR之后再跟diff OR。这样，只有diff一开始是0,并且，a，b的每一个字节XOR的结果也是零，最后循环完成后diff的值才是0,这种情况是a，b完全一样。否则最后diff是一个非零的值。

我们使用XOR而不适用"=="的原因是"=="通常编译成分支的形式。比如C代码"diff &= a == b" 可能编译成下面的X86汇编。

```
MOV EAX, [A]
CMP [B], EAX
JZ equal
JMP done
equal:
AND [VALID], 1
done:
AND [VALID], 0

```

分支会导致代码执行的时间出现差异。

C代码的"diff |= a ^ b"编译之后类似于，

```
MOV EAX, [A]
XOR EAX, [B]
OR [DIFF], EAX 

```

执行时间跟两个变量是否相等没有关系。

**为什么要讨论这么多关于hash的东西**

用户在你的网站上输入密码，是相信你的安全性。如果你的数据库被黑了。而用户密码又没有恰当的保护，那么恶意的攻击者就可以利用这些密码尝试登陆其他的网站和服务。进行撞库攻击。(很多用户在所有的地方都是使用相同的密码)这不仅仅是你的网站安全，是你的所有用户的安全。你要对你用户的安全负责。

0x0B PHP PBKDF2 密码hash代码
------------------------

* * *

[代码下载](https://crackstation.net/source/password-hashing/PasswordHash.php)

```
<?php
/*
 * Password Hashing With PBKDF2 (http://crackstation.net/hashing-security.htm).
 * Copyright (c) 2013, Taylor Hornby
 * All rights reserved.
 *
 * Redistribution and use in source and binary forms, with or without 
 * modification, are permitted provided that the following conditions are met:
 *
 * 1. Redistributions of source code must retain the above copyright notice, 
 * this list of conditions and the following disclaimer.
 *
 * 2. Redistributions in binary form must reproduce the above copyright notice,
 * this list of conditions and the following disclaimer in the documentation 
 * and/or other materials provided with the distribution.
 *
 * THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS" 
 * AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE 
 * IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE 
 * ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE 
 * LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR 
 * CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF 
 * SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS 
 * INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN 
 * CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) 
 * ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE 
 * POSSIBILITY OF SUCH DAMAGE.
 */

// These constants may be changed without breaking existing hashes.
define("PBKDF2_HASH_ALGORITHM", "sha256");
define("PBKDF2_ITERATIONS", 1000);
define("PBKDF2_SALT_BYTE_SIZE", 24);
define("PBKDF2_HASH_BYTE_SIZE", 24);

define("HASH_SECTIONS", 4);
define("HASH_ALGORITHM_INDEX", 0);
define("HASH_ITERATION_INDEX", 1);
define("HASH_SALT_INDEX", 2);
define("HASH_PBKDF2_INDEX", 3);

function create_hash($password)
{
    // format: algorithm:iterations:salt:hash
    $salt = base64_encode(mcrypt_create_iv(PBKDF2_SALT_BYTE_SIZE, MCRYPT_DEV_URANDOM));
    return PBKDF2_HASH_ALGORITHM . ":" . PBKDF2_ITERATIONS . ":" .  $salt . ":" .
        base64_encode(pbkdf2(
            PBKDF2_HASH_ALGORITHM,
            $password,
            $salt,
            PBKDF2_ITERATIONS,
            PBKDF2_HASH_BYTE_SIZE,
            true
        ));
}

function validate_password($password, $correct_hash)
{
    $params = explode(":", $correct_hash);
    if(count($params) < HASH_SECTIONS)
       return false;
    $pbkdf2 = base64_decode($params[HASH_PBKDF2_INDEX]);
    return slow_equals(
        $pbkdf2,
        pbkdf2(
            $params[HASH_ALGORITHM_INDEX],
            $password,
            $params[HASH_SALT_INDEX],
            (int)$params[HASH_ITERATION_INDEX],
            strlen($pbkdf2),
            true
        )
    );
}

// Compares two strings $a and $b in length-constant time.
function slow_equals($a, $b)
{
    $diff = strlen($a) ^ strlen($b);
    for($i = 0; $i < strlen($a) && $i < strlen($b); $i++)
    {
        $diff |= ord($a[$i]) ^ ord($b[$i]);
    }
    return $diff === 0;
}

/*
 * PBKDF2 key derivation function as defined by RSA's PKCS #5: https://www.ietf.org/rfc/rfc2898.txt
 * $algorithm - The hash algorithm to use. Recommended: SHA256
 * $password - The password.
 * $salt - A salt that is unique to the password.
 * $count - Iteration count. Higher is better, but slower. Recommended: At least 1000.
 * $key_length - The length of the derived key in bytes.
 * $raw_output - If true, the key is returned in raw binary format. Hex encoded otherwise.
 * Returns: A $key_length-byte key derived from the password and salt.
 *
 * Test vectors can be found here: https://www.ietf.org/rfc/rfc6070.txt
 *
 * This implementation of PBKDF2 was originally created by https://defuse.ca
 * With improvements by http://www.variations-of-shadow.com
 */
function pbkdf2($algorithm, $password, $salt, $count, $key_length, $raw_output = false)
{
    $algorithm = strtolower($algorithm);
    if(!in_array($algorithm, hash_algos(), true))
        trigger_error('PBKDF2 ERROR: Invalid hash algorithm.', E_USER_ERROR);
    if($count <= 0 || $key_length <= 0)
        trigger_error('PBKDF2 ERROR: Invalid parameters.', E_USER_ERROR);

    if (function_exists("hash_pbkdf2")) {
        // The output length is in NIBBLES (4-bits) if $raw_output is false!
        if (!$raw_output) {
            $key_length = $key_length * 2;
        }
        return hash_pbkdf2($algorithm, $password, $salt, $count, $key_length, $raw_output);
    }

    $hash_length = strlen(hash($algorithm, "", true));
    $block_count = ceil($key_length / $hash_length);

    $output = "";
    for($i = 1; $i <= $block_count; $i++) {
        // $i encoded as 4 bytes, big endian.
        $last = $salt . pack("N", $i);
        // first iteration
        $last = $xorsum = hash_hmac($algorithm, $last, $password, true);
        // perform the other $count - 1 iterations
        for ($j = 1; $j < $count; $j++) {
            $xorsum ^= ($last = hash_hmac($algorithm, $last, $password, true));
        }
        $output .= $xorsum;
    }

    if($raw_output)
        return substr($output, 0, $key_length);
    else
        return bin2hex(substr($output, 0, $key_length));
}
?>

```

0x0C java PBKDF2 密码hash代码
-------------------------

* * *

[代码下载](https://crackstation.net/source/password-hashing/PasswordHash.java)

```
/* 
 * Password Hashing With PBKDF2 (http://crackstation.net/hashing-security.htm).
 * Copyright (c) 2013, Taylor Hornby
 * All rights reserved.
 *
 * Redistribution and use in source and binary forms, with or without 
 * modification, are permitted provided that the following conditions are met:
 *
 * 1. Redistributions of source code must retain the above copyright notice, 
 * this list of conditions and the following disclaimer.
 *
 * 2. Redistributions in binary form must reproduce the above copyright notice,
 * this list of conditions and the following disclaimer in the documentation 
 * and/or other materials provided with the distribution.
 *
 * THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS" 
 * AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE 
 * IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE 
 * ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE 
 * LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR 
 * CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF 
 * SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS 
 * INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN 
 * CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) 
 * ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE 
 * POSSIBILITY OF SUCH DAMAGE.
 */

import java.security.SecureRandom;
import javax.crypto.spec.PBEKeySpec;
import javax.crypto.SecretKeyFactory;
import java.math.BigInteger;
import java.security.NoSuchAlgorithmException;
import java.security.spec.InvalidKeySpecException;

/*
 * PBKDF2 salted password hashing.
 * Author: havoc AT defuse.ca
 * www: http://crackstation.net/hashing-security.htm
 */
public class PasswordHash
{
    public static final String PBKDF2_ALGORITHM = "PBKDF2WithHmacSHA1";

    // The following constants may be changed without breaking existing hashes.
    public static final int SALT_BYTE_SIZE = 24;
    public static final int HASH_BYTE_SIZE = 24;
    public static final int PBKDF2_ITERATIONS = 1000;

    public static final int ITERATION_INDEX = 0;
    public static final int SALT_INDEX = 1;
    public static final int PBKDF2_INDEX = 2;

    /**
     * Returns a salted PBKDF2 hash of the password.
     *
     * @param   password    the password to hash
     * @return              a salted PBKDF2 hash of the password
     */
    public static String createHash(String password)
        throws NoSuchAlgorithmException, InvalidKeySpecException
    {
        return createHash(password.toCharArray());
    }

    /**
     * Returns a salted PBKDF2 hash of the password.
     *
     * @param   password    the password to hash
     * @return              a salted PBKDF2 hash of the password
     */
    public static String createHash(char[] password)
        throws NoSuchAlgorithmException, InvalidKeySpecException
    {
        // Generate a random salt
        SecureRandom random = new SecureRandom();
        byte[] salt = new byte[SALT_BYTE_SIZE];
        random.nextBytes(salt);

        // Hash the password
        byte[] hash = pbkdf2(password, salt, PBKDF2_ITERATIONS, HASH_BYTE_SIZE);
        // format iterations:salt:hash
        return PBKDF2_ITERATIONS + ":" + toHex(salt) + ":" +  toHex(hash);
    }

    /**
     * Validates a password using a hash.
     *
     * @param   password        the password to check
     * @param   correctHash     the hash of the valid password
     * @return                  true if the password is correct, false if not
     */
    public static boolean validatePassword(String password, String correctHash)
        throws NoSuchAlgorithmException, InvalidKeySpecException
    {
        return validatePassword(password.toCharArray(), correctHash);
    }

    /**
     * Validates a password using a hash.
     *
     * @param   password        the password to check
     * @param   correctHash     the hash of the valid password
     * @return                  true if the password is correct, false if not
     */
    public static boolean validatePassword(char[] password, String correctHash)
        throws NoSuchAlgorithmException, InvalidKeySpecException
    {
        // Decode the hash into its parameters
        String[] params = correctHash.split(":");
        int iterations = Integer.parseInt(params[ITERATION_INDEX]);
        byte[] salt = fromHex(params[SALT_INDEX]);
        byte[] hash = fromHex(params[PBKDF2_INDEX]);
        // Compute the hash of the provided password, using the same salt, 
        // iteration count, and hash length
        byte[] testHash = pbkdf2(password, salt, iterations, hash.length);
        // Compare the hashes in constant time. The password is correct if
        // both hashes match.
        return slowEquals(hash, testHash);
    }

    /**
     * Compares two byte arrays in length-constant time. This comparison method
     * is used so that password hashes cannot be extracted from an on-line 
     * system using a timing attack and then attacked off-line.
     * 
     * @param   a       the first byte array
     * @param   b       the second byte array 
     * @return          true if both byte arrays are the same, false if not
     */
    private static boolean slowEquals(byte[] a, byte[] b)
    {
        int diff = a.length ^ b.length;
        for(int i = 0; i < a.length && i < b.length; i++)
            diff |= a[i] ^ b[i];
        return diff == 0;
    }

    /**
     *  Computes the PBKDF2 hash of a password.
     *
     * @param   password    the password to hash.
     * @param   salt        the salt
     * @param   iterations  the iteration count (slowness factor)
     * @param   bytes       the length of the hash to compute in bytes
     * @return              the PBDKF2 hash of the password
     */
    private static byte[] pbkdf2(char[] password, byte[] salt, int iterations, int bytes)
        throws NoSuchAlgorithmException, InvalidKeySpecException
    {
        PBEKeySpec spec = new PBEKeySpec(password, salt, iterations, bytes * 8);
        SecretKeyFactory skf = SecretKeyFactory.getInstance(PBKDF2_ALGORITHM);
        return skf.generateSecret(spec).getEncoded();
    }

    /**
     * Converts a string of hexadecimal characters into a byte array.
     *
     * @param   hex         the hex string
     * @return              the hex string decoded into a byte array
     */
    private static byte[] fromHex(String hex)
    {
        byte[] binary = new byte[hex.length() / 2];
        for(int i = 0; i < binary.length; i++)
        {
            binary[i] = (byte)Integer.parseInt(hex.substring(2*i, 2*i+2), 16);
        }
        return binary;
    }

    /**
     * Converts a byte array into a hexadecimal string.
     *
     * @param   array       the byte array to convert
     * @return              a length*2 character string encoding the byte array
     */
    private static String toHex(byte[] array)
    {
        BigInteger bi = new BigInteger(1, array);
        String hex = bi.toString(16);
        int paddingLength = (array.length * 2) - hex.length();
        if(paddingLength > 0)
            return String.format("%0" + paddingLength + "d", 0) + hex;
        else
            return hex;
    }

    /**
     * Tests the basic functionality of the PasswordHash class
     *
     * @param   args        ignored
     */
    public static void main(String[] args)
    {
        try
        {
            // Print out 10 hashes
            for(int i = 0; i < 10; i++)
                System.out.println(PasswordHash.createHash("p\r\nassw0Rd!"));

            // Test password validation
            boolean failure = false;
            System.out.println("Running tests...");
            for(int i = 0; i < 100; i++)
            {
                String password = ""+i;
                String hash = createHash(password);
                String secondHash = createHash(password);
                if(hash.equals(secondHash)) {
                    System.out.println("FAILURE: TWO HASHES ARE EQUAL!");
                    failure = true;
                }
                String wrongPassword = ""+(i+1);
                if(validatePassword(wrongPassword, hash)) {
                    System.out.println("FAILURE: WRONG PASSWORD ACCEPTED!");
                    failure = true;
                }
                if(!validatePassword(password, hash)) {
                    System.out.println("FAILURE: GOOD PASSWORD NOT ACCEPTED!");
                    failure = true;
                }
            }
            if(failure)
                System.out.println("TESTS FAILED!");
            else
                System.out.println("TESTS PASSED!");
        }
        catch(Exception ex)
        {
            System.out.println("ERROR: " + ex);
        }
    }

}

```

0x0D ASP.NET (C#)密码hash代码
-------------------------

* * *

[代码下载](https://crackstation.net/source/password-hashing/PasswordHash.cs)

```
/* 
 * Password Hashing With PBKDF2 (http://crackstation.net/hashing-security.htm).
 * Copyright (c) 2013, Taylor Hornby
 * All rights reserved.
 *
 * Redistribution and use in source and binary forms, with or without 
 * modification, are permitted provided that the following conditions are met:
 *
 * 1. Redistributions of source code must retain the above copyright notice, 
 * this list of conditions and the following disclaimer.
 *
 * 2. Redistributions in binary form must reproduce the above copyright notice,
 * this list of conditions and the following disclaimer in the documentation 
 * and/or other materials provided with the distribution.
 *
 * THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS" 
 * AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE 
 * IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE 
 * ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE 
 * LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR 
 * CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF 
 * SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS 
 * INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN 
 * CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) 
 * ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE 
 * POSSIBILITY OF SUCH DAMAGE.
 */

using System;
using System.Text;
using System.Security.Cryptography;

namespace PasswordHash
{
    /// <summary>
    /// Salted password hashing with PBKDF2-SHA1.
    /// Author: havoc AT defuse.ca
    /// www: http://crackstation.net/hashing-security.htm
    /// Compatibility: .NET 3.0 and later.
    /// </summary>
    public class PasswordHash
    {
        // The following constants may be changed without breaking existing hashes.
        public const int SALT_BYTE_SIZE = 24;
        public const int HASH_BYTE_SIZE = 24;
        public const int PBKDF2_ITERATIONS = 1000;

        public const int ITERATION_INDEX = 0;
        public const int SALT_INDEX = 1;
        public const int PBKDF2_INDEX = 2;

        /// <summary>
        /// Creates a salted PBKDF2 hash of the password.
        /// </summary>
        /// <param name="password">The password to hash.</param>
        /// <returns>The hash of the password.</returns>
        public static string CreateHash(string password)
        {
            // Generate a random salt
            RNGCryptoServiceProvider csprng = new RNGCryptoServiceProvider();
            byte[] salt = new byte[SALT_BYTE_SIZE];
            csprng.GetBytes(salt);

            // Hash the password and encode the parameters
            byte[] hash = PBKDF2(password, salt, PBKDF2_ITERATIONS, HASH_BYTE_SIZE);
            return PBKDF2_ITERATIONS + ":" +
                Convert.ToBase64String(salt) + ":" +
                Convert.ToBase64String(hash);
        }

        /// <summary>
        /// Validates a password given a hash of the correct one.
        /// </summary>
        /// <param name="password">The password to check.</param>
        /// <param name="correctHash">A hash of the correct password.</param>
        /// <returns>True if the password is correct. False otherwise.</returns>
        public static bool ValidatePassword(string password, string correctHash)
        {
            // Extract the parameters from the hash
            char[] delimiter = { ':' };
            string[] split = correctHash.Split(delimiter);
            int iterations = Int32.Parse(split[ITERATION_INDEX]);
            byte[] salt = Convert.FromBase64String(split[SALT_INDEX]);
            byte[] hash = Convert.FromBase64String(split[PBKDF2_INDEX]);

            byte[] testHash = PBKDF2(password, salt, iterations, hash.Length);
            return SlowEquals(hash, testHash);
        }

        /// <summary>
        /// Compares two byte arrays in length-constant time. This comparison
        /// method is used so that password hashes cannot be extracted from
        /// on-line systems using a timing attack and then attacked off-line.
        /// </summary>
        /// <param name="a">The first byte array.</param>
        /// <param name="b">The second byte array.</param>
        /// <returns>True if both byte arrays are equal. False otherwise.</returns>
        private static bool SlowEquals(byte[] a, byte[] b)
        {
            uint diff = (uint)a.Length ^ (uint)b.Length;
            for (int i = 0; i < a.Length && i < b.Length; i++)
                diff |= (uint)(a[i] ^ b[i]);
            return diff == 0;
        }

        /// <summary>
        /// Computes the PBKDF2-SHA1 hash of a password.
        /// </summary>
        /// <param name="password">The password to hash.</param>
        /// <param name="salt">The salt.</param>
        /// <param name="iterations">The PBKDF2 iteration count.</param>
        /// <param name="outputBytes">The length of the hash to generate, in bytes.</param>
        /// <returns>A hash of the password.</returns>
        private static byte[] PBKDF2(string password, byte[] salt, int iterations, int outputBytes)
        {
            Rfc2898DeriveBytes pbkdf2 = new Rfc2898DeriveBytes(password, salt);
            pbkdf2.IterationCount = iterations;
            return pbkdf2.GetBytes(outputBytes);
        }
    }
}

```

0x0E Ruby (on Rails) 密码hash代码
-----------------------------

* * *

[代码下载](https://crackstation.net/source/password-hashing/PasswordHash.rb)

```
# Password Hashing With PBKDF2 (http://crackstation.net/hashing-security.htm).
# Copyright (c) 2013, Taylor Hornby
# All rights reserved.
# 
# Redistribution and use in source and binary forms, with or without 
# modification, are permitted provided that the following conditions are met:
# 
# 1. Redistributions of source code must retain the above copyright notice, 
# this list of conditions and the following disclaimer.
# 
# 2. Redistributions in binary form must reproduce the above copyright notice,
# this list of conditions and the following disclaimer in the documentation 
# and/or other materials provided with the distribution.
# 
# THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS" 
# AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE 
# IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE 
# ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE 
# LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR 
# CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF 
# SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS 
# INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN 
# CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) 
# ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE 
# POSSIBILITY OF SUCH DAMAGE.

require 'securerandom'
require 'openssl'
require 'base64'

# Salted password hashing with PBKDF2-SHA1.
# Authors: @RedragonX (dicesoft.net), havoc AT defuse.ca 
# www: http://crackstation.net/hashing-security.htm
module PasswordHash

  # The following constants can be changed without breaking existing hashes.
  PBKDF2_ITERATIONS = 1000
  SALT_BYTE_SIZE = 24
  HASH_BYTE_SIZE = 24

  HASH_SECTIONS = 4
  SECTION_DELIMITER = ':'
  ITERATIONS_INDEX = 1
  SALT_INDEX = 2
  HASH_INDEX = 3

  # Returns a salted PBKDF2 hash of the password.
  def self.createHash( password )
    salt = SecureRandom.base64( SALT_BYTE_SIZE )
    pbkdf2 = OpenSSL::PKCS5::pbkdf2_hmac_sha1(
      password,
      salt,
      PBKDF2_ITERATIONS,
      HASH_BYTE_SIZE
    )
    return ["sha1", PBKDF2_ITERATIONS, salt, Base64.encode64( pbkdf2 )].join( SECTION_DELIMITER )
  end

  # Checks if a password is correct given a hash of the correct one.
  # correctHash must be a hash string generated with createHash.
  def self.validatePassword( password, correctHash )
    params = correctHash.split( SECTION_DELIMITER )
    return false if params.length != HASH_SECTIONS

    pbkdf2 = Base64.decode64( params[HASH_INDEX] )
    testHash = OpenSSL::PKCS5::pbkdf2_hmac_sha1(
      password,
      params[SALT_INDEX],
      params[ITERATIONS_INDEX].to_i,
      pbkdf2.length
    )

    return pbkdf2 == testHash
  end

  # Run tests to ensure the module is functioning properly.
  # Returns true if all tests succeed, false if not.
  def self.runSelfTests
    puts "Sample hashes:"
    3.times { puts createHash("password") }

    puts "\nRunning self tests..."
    @@allPass = true

    correctPassword = 'aaaaaaaaaa'
    wrongPassword = 'aaaaaaaaab'
    hash = createHash(correctPassword)

    assert( validatePassword( correctPassword, hash ) == true, "correct password" )
    assert( validatePassword( wrongPassword, hash ) == false, "wrong password" )

    h1 = hash.split( SECTION_DELIMITER )
    h2 = createHash( correctPassword ).split( SECTION_DELIMITER )
    assert( h1[HASH_INDEX] != h2[HASH_INDEX], "different hashes" )
    assert( h1[SALT_INDEX] != h2[SALT_INDEX], "different salt" )

    if @@allPass
      puts "*** ALL TESTS PASS ***"
    else
      puts "*** FAILURES ***"
    end

    return @@allPass
  end

  def self.assert( truth, msg )
    if truth
      puts "PASS [#{msg}]"
    else
      puts "FAIL [#{msg}]"
      @@allPass = false
    end
  end

end

PasswordHash.runSelfTests

```

from:[https://crackstation.net/hashing-security.htm](https://crackstation.net/hashing-security.htm)