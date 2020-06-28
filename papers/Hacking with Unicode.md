# Hacking with Unicode

from：https://speakerdeck.com/mathiasbynens/hacking-with-unicode

* * *

0x00 Unicode简介
--------------

* * *

很多人经常会把Unicode和utf-8的概念混淆在一起，甚至会拿它们去做比较。实际上这种比较是非常的荒谬的。这就犹如拿“苹果”和“僵尸”去做比较，当然我觉得可以是更夸张的例子。所以，首先需要声明的是Unicode不是字符编码。我们可以把Unicode看做是一个数据库，不同的码点(code point)会指向不同的符号（如下图所示）。这是一种很简单想法，当你说出一个符号的码点时，别人就会知道你在说的是什么符号。当然了，如果他不知道也可以去查询Unicode表。

![enter image description here](http://drops.javaweb.org/uploads/images/4a3b28f30b6680a4e147e46918643d9ed6417d3e.jpg)

下面以拉丁字母中大写的A为例：

![enter image description here](http://drops.javaweb.org/uploads/images/3c6b8fb779703e0b301b7d0f6f9e7c1dea1685a8.jpg)

拉丁字母大写A的码点就是: U+0041。又比如码点U+2603就会是指向一个雪人(snowman)的符号：

![enter image description here](http://drops.javaweb.org/uploads/images/338f2848b5a59bb22cb254c55eb33135e1edfb57.jpg)

当然还有一些更有趣的符号，比如下面这位：

![enter image description here](http://drops.javaweb.org/uploads/images/6c9e06ec20a74c8b51286ed9619a3a60282b0a38.jpg)

这个符号看上去是一坨屎在向你微笑，这符号虽然很搞笑，但这并不是重点。因为我们发现这次的c码点从4位变成了5位。那么码点的长度究竟应该是多少，又到底存在多少个码点呢？

```
U+000000 -> U+10FFFF

```

上面这个范围就是码点的范围了，长度一目了然。至于真实存有符号的码点的数量就不是所有的码点的总和了。因为其中也包括一些预留位。 除此之外还需要我们了解的就是平面（plane）的概念。Unicode字符被分成17组编排，而其中的每一组我们都称其为平面（Plane）。 其中的第一组被称为基本多语言面（Basic Multilingual Plane)也可以简称为BMP。是我们平时最常用的平面。它的范围是: U+0000 -> U+FFFF 举个例子，如果你正在使用英语来来编写一篇文章，那么你所用的字符就可能全都来自这一个平面。其余的平面2-17的范围则是:

```
U+010000 -> U+10FFFF

```

这16个平面占据了整个Unicode的75%，我相信来自这些平面的字符很容易就可以和基本多语平面做区分。因为只要码点的长度大于4，你就应该想到这个字符是来自2-17的平面。

0x01 编码
-------

* * *

在我们对Unicode的基础知识有了一点了解之后，现在再让我们来谈谈编码。

![enter image description here](http://drops.javaweb.org/uploads/images/3e92ae462bd60489d1899e052be2483247636cd9.jpg)

上图中所展示的UTF-8,UTF-16,UTF-32,UTF-EBCDIC和GB18030都是一些完全支持Unicode的编码。我们发现这种编码数量并不多，而且它们也存在一些差异。拿UTF-32来讲，我们发现不论需要编码的字符是什么，这个编码都会使用4 bytes。从存储空间的角度来讲，这种设计是十分的不合理的。相对而言UTF-16会显得更合理一些，不过不难看出其中最合理的还是属UTF-8。比如，我们用utf-8编码的一个字符，如果是属于第一个区域(U+000000-U+00007F)那么它就只会占用1 bytes。 在探索编码的过程当中，你最多可能会被搜索引擎带到的可能是下面的页面：

![enter image description here](http://drops.javaweb.org/uploads/images/9884711ae3f92b29ec20319d3d44c568244eb79b.jpg)

如同你所看到的，这是IETF发布的RFC文档。很多人在去翻阅文档时，都会选择看RFC文档。原因很简单因为RFC是标准。在RFC文档中我们甚至可以查阅到Unicode是如何被设计的。但是作为黑客，我想这些应该不是你所感兴趣的。在这里我会更加推荐你去看下面的网页：

![enter image description here](http://drops.javaweb.org/uploads/images/042ded7c99ffa5c03a4853d947850eb84a82d751.jpg)

在Encoding Living Standard你可以阅读到一些更贴近真实世界的东西。不是因为对RFC有偏见，而是因为RFC中的标准和实际应用到浏览器中的情况往往都会存在一些差异。

0x02 JavaScript中的Unicode
------------------------

* * *

在这个部分中，我们主要会谈论JavaScript中一些奇怪的现象，一些程序员会犯的错误以及如何去修复。就像很多其它语言一样，JavaScript中也会有很多奇怪的现象。比如有这样的网站来专门纰漏PHP中一些奇怪的现象。

![enter image description here](http://drops.javaweb.org/uploads/images/dd22ab210b1390f9d53544e09931191a935d9faf.jpg)

同样的我们也有wtfjs。上面会纰漏一些JavaScript中的奇怪现象:

![enter image description here](http://drops.javaweb.org/uploads/images/5bc091e2a59150a50c48b2e4d4340f5d12247738.jpg)

当然还有一些其它的问题。比如，比较著名的Punycode攻击，常用于注册看上去相似的域名进行钓鱼攻击(如下图所示)。对于Punycode攻击也有一些非常有意思的文档。比如，Mutillidae的作者Adrian在不久前发表的” Fun and games with Unicode”。

![enter image description here](http://drops.javaweb.org/uploads/images/a4eade9fab489139364bf18c01791f8cb8202717.jpg)

我们都知道在Javascript当中我们可以使用base16来表示一个字符，比如： 用 ‘x41’ 来表示大写拉丁字母”A” 我们也可以用Unicode来表示一个字符，比如： 用’u0041’也可以表示大写拉丁字母”A” 那我们之前提到的U+1F4A9（那个在向你微笑的一坨屎）又是怎么样的呢？这个问题说简单也简单，说复杂也复杂。在ECMAScript 6（下面将简称为ES）中我们可以这样来解决这个问题：

![enter image description here](http://drops.javaweb.org/uploads/images/a34edf9e4d030806e59af3a31c7cc0367ca73a08.jpg)

当然，我们也可以使用代理编码对（surrogate pairs）:

![enter image description here](http://drops.javaweb.org/uploads/images/038e362eb1cd4827616c7fa15b4c23674efb5afa.jpg)

下面再附上代理编码对的编码。当然了，你不需要真的去记住它。但如果你是一个经常会和Unicode打交道的人，那么你至少应该去去了解一下。

![enter image description here](http://drops.javaweb.org/uploads/images/154efbebd5fa40ab38bf0d5846afaf1c3aeb7026.jpg)

现在我们再聊一下字符串长度的问题。这也是一些开发者会常犯的错误。在这里需要我们记住的是，字符串长度并不等于字符个数。下面这张图应该很容基就能帮助我们理解到这个问题。

![enter image description here](http://drops.javaweb.org/uploads/images/39dd0a122387c574b41bae66173d05036a168300.jpg)

不要认为没有人会犯这样的错误。Twitter上就有这样的问题，我实际上应该被允许发140坨屎,但实际上我只能发70坨……。虽然这不是安全问题，但这仍然是个逻辑错误。

![enter image description here](http://drops.javaweb.org/uploads/images/b90826a296b9c886788f45eef59b6edaada03b99.jpg)

对于处理这个问题你可以使用我编写的punycode.js:

![enter image description here](http://drops.javaweb.org/uploads/images/4d630ab1f53e277743e35e1e5ee38cfe4bee1741.jpg)

当然在ES6中也有别的解决方案：

![enter image description here](http://drops.javaweb.org/uploads/images/89ca06bb3bb77ae472d9bb8a22e703f4e120d20f.jpg)

但实际上问题往往没有上面描述的那么简单。因为javascript还对会一些字符进行escape：

![enter image description here](http://drops.javaweb.org/uploads/images/34718843eeb7cb7c4f51b7bb048d1e560b9b872a.jpg)

这个问题甚至会比我们想象的还要更复杂一些：

![enter image description here](http://drops.javaweb.org/uploads/images/bc99ee862ab5bb351caeee794fb887f5fe2b2d9c.jpg)

所以ES6又为我们提供了Unicode标准化：

![enter image description here](http://drops.javaweb.org/uploads/images/c905d567e554f8905df14b2060f6f0507c458f31.jpg)

那么这个方案已经完美了么？其实也不然。因为我们往往会面临更复杂的挑战。比如下面的例子，我们看到的只有9个字符，然而结果会是116。修复这类问题我们还需要用到epic regex-fu.

![enter image description here](http://drops.javaweb.org/uploads/images/22626b04d4f4bd0f5b385fdd0ee540ce5292bdf9.jpg)

同样的问题也存在于reverse函数中（看图中第三个reverse结果）

![enter image description here](http://drops.javaweb.org/uploads/images/354911715bd5bd03fba087cdec2e647e2e141f48.jpg)

我们可以通过下图中提供的esrever来解决这个问题：

![enter image description here](http://drops.javaweb.org/uploads/images/571e3d4a73cb295e011e02e6858e9e4ccf4df99b.jpg)

又比如String.fromCharCode()函数也只能处理U+0000~U+FFFF范围内的字符。在解决这个问题上时，我们依旧可以使用之前提到的代理编码对，Punycode.js又或者是ES6

![enter image description here](http://drops.javaweb.org/uploads/images/babcd2316c5d9c91dffd7029ac9c0ac94119e27b.jpg)

除此之外还有charAt()和charCodeAt()函数只能取一半的unicode的问题（U+1F4A9只能取到U+D83D)也在ES6和ES7当中得到了解决。在ES7可以用at()来替代charAt（据说这个问题没赶上ES6的修正）,在ES6中的可以使用codePointAt来代替charCodeAt()。存在类似问题的函数还有很多，比如：slice(),substring()等等等等。 然后就是一些正则匹配的问题。也分别在ES6当中得到了解决。（翻到这儿感觉像是推ES6的有没有……）

![enter image description here](http://drops.javaweb.org/uploads/images/b4648a0e509b2d2f2dbd2b38251be5f503e855aa.jpg)

当然这些也许只是JS开发者在开发的过程当中会碰到的问题的一小部分。对于测试这类问题我推荐你使用“那坨屎”来进行验证^_^

0x03 Mysql中的Unicode
-------------------

* * *

谈完了JavaScript中Unicode问题之后，让我们再来看看Mysql中Unicode的表现又是如何的。下面是个很常见的例子，网上也有很多Mysql教程是教你这么去建数据库的。

![enter image description here](http://drops.javaweb.org/uploads/images/5125c25a20a2cc75447f9fa97aace36988a0df66.jpg)

请注意这里的编码选择的是UTF-8。这也会有问题？请看下图：

![enter image description here](http://drops.javaweb.org/uploads/images/cd2ea4008142397438356ae1512070db607c571f.jpg)

从上图中我们可以看到当我们使用UTF－8编码来建立数据库，并试图更新id=9001的字段column_name为’fooU+1F4A9foo’时，虽然更新成功了，但出现了警告。当我们对当前表的该字段进行查询时，发现内容居然被截断了。因为这里的UTF-8并不是我们所认识的支持所有unicode的UTF-8。所以作为开发者我们需要记住，应该使用utf8mb4来对数据库的编码进行设定，而不是使用utf-8.这种截断在某些场景下将导致很严肃的安全问题。

0x04 Hacking with Unicode
-------------------------

* * *

在做了这么多的铺垫之后，让我们来看一下真实生活当中存在的Hacking.

### <1> UTF-16/UTF-32可能导致XSS问题（谷歌实例）

![enter image description here](http://drops.javaweb.org/uploads/images/d5459e1530dc22db5f5abcf480f37cc0a2cfdd4f.jpg)

### <2>Unicode将导致用户名欺骗问题

![enter image description here](http://drops.javaweb.org/uploads/images/06f8a2c1e9d1365b2e0b3f1cffc796861b760cb7.jpg)

### <3>Unicode换行符将导致问题XSS

![enter image description here](http://drops.javaweb.org/uploads/images/4c998f6857c7b4b7cfbdca3524d0dde5fdd24c4a.jpg)

### <4>Mysql unicode截断问题，导致注册邮箱限制绕过

![enter image description here](http://drops.javaweb.org/uploads/images/8bf53dd2cec76a875c688497b7c731e51ea7cbf0.jpg)

### <5>Mysql unicode截断问题，导致WP插件被爆菊：

![enter image description here](http://drops.javaweb.org/uploads/images/dd308a61f40897fc1d246a9187cf01bca29fbd2a.jpg)

### <6>Unicode问题导致Stackoverflow的HTML净化器被绕过：

![enter image description here](http://drops.javaweb.org/uploads/images/b1af160cabd626fdd1fd693898e04efeb0bb0770.jpg)

0x05 写在最后
---------

* * *

如果你是一个JavaScript开发者，我觉得这是一个你不容错过的文章。文章中不但给出了一些在开发过程中很有可能碰到的问题，也给出了相应的解决方案。如果你是黑客，文章中给出的检验这种缺陷的方法也许可以给你带来不少的收获。至少Mysql的截断问题，在我看来还是十分有趣的。