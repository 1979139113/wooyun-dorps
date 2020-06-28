# 密码管理利器：Linux - KeePassX

随着安全性问题变得越来越重要，密码当然是越安全越理想（比如多步认证）。

于是我最近就试用了几个安全密码管理器，试图找到一款比较安全，易于使用并且跨平台的应用软件。

首先，我尝试了[LastPass](https://lastpass.com/)。LastPass大概最为人们所熟知，因为它是基于WEB管理密码的，是所有软件中平台无关性最强的。但是我发现它的界面简陋，而且提供太多的工具和选项，比较繁琐。

接下来，我又试了试[KeePass 2](http://keepass.info/index.html)。尽管是一款功能相当完善的应用软件，非常类似于下面我将要描述的，但是官方没有提供Linux的安装包，而且社区移植的版本，虽然可用，但仍然算不上最好的。所以我又尝试了其他应用。

在所有的试用过的软件中，我最喜欢的是 KeePassX 。 它原来是KeePass的在Linux上的移植版，但是后来演变成了独立的应用。凭借更漂亮、更原生的外观，KeePassX 打败了KeePass 2。

在ubuntu中使用KeePassX
------------------

* * *

方便的是，KeePassX已经提供在ubuntu上安装的软件包。

从命令行安装KeePassX或者 从软件管理中心安装：

从Ubuntu软件中心安装[KeePassX](http://apt.ubuntu.com/p/keepassx)

安装后打开它，你会看到一个空白窗口。点击工具条上的第一个按钮来创建一个新数据库。你可以使用密钥文件或者密码保护这个刚刚创建的数据库。一般你会使用密码，因为只需要记住它并输入就行了 - 你应该输入较长的密码，这样你就可以防止其他人使用你的数据库。

接下来，你得把它存到某个位置。我保存在我的Dropbox里面，这样就可以从多个地方获取。

Dropbox使用双因子认证，所以如果有人想进到我的Dropbox里面，他就得拿到我的手机，这样的方式是还是相当安全的。

或者你也可以使用其他的服务，比如Google Drive和Skydrive，它们都可以使用[认证器](https://play.google.com/store/apps/details?id=com.google.android.apps.authenticator2&hl=en)应用；也可以使用Box，它用短信进行双因子认证。

当然，如果你真的很在意自己的密码，你很可能不想把密码存到其他的地方，因为理论上密码是可以被他们获取到的。

![2014030320181077268.png](http://drops.javaweb.org/uploads/images/ee4b8c8cf2e5faf3ce86f954761c4f893000b8e1.jpg)Ubuntu中KeePassX的主界面

使用该应用还是相当直观明了的。你可以添加分组，然后在分组里添加密码。KeePassX带有一个很方便的密码生成器，当你

需要输入一个密码的时候可以使用该生成器，而不用自己构思一个。我倾向于使用所有基本的字符以及挑选的特殊的字符来生成我的密码，20个字符的长度，当然 这得看你访问的网站接不接受了。

需要注意一点，有些网站并不告诉你他们接受多长字符的密码，倾向于只在输入框限制输入长度。如果你粘贴进去的密码看起来没那么长，很可能就不是你要输入的口令，而被截断了。这种情况我碰到过几次。

![2014030320184359989.png](http://drops.javaweb.org/uploads/images/b9ba013a95147895096d116459780ff855df6618.jpg)KeePassX 密码生成器

根据日常的使用经验，我积累了一些小的技巧，使得操作KeePassX更简单一些：

### 关于复制粘贴的问题

像这样复制粘贴密码，你可能会比较担心。可以肯定的是这比手动输入高效多了。默认情况下，KeePassX会在一分钟后清空粘贴板，你也可以设置更短的时间，所以不必担心有人会在你电脑上把密码粘贴下来查看。你也可以开启一个AutoType的特性，不过这对于我来说没用。 Chris Zuber 在这个[评论](http://www.omgubuntu.co.uk/2013/10/manage-passwords-securely-keepassx#comment-1080345241)里面说明了如何使用 AutoType 。

### 数据库的困境

如果你把数据库存放到云端，就不要为云端服务设置完全随机的密码。如果你不能进入到云，但是又把云密码存储到云里边，这是完全没有用的。这看起来似乎很明显，但是刚开始我却没有意识到这一点。

### 确保所有的密码都是安全的

为了查看常用的账号，工作或者学习的时候要频繁地掏手机，这也是一件挺痛苦的事儿，所以设置密码的时候不妨想象一下这种情形，哈。

未来
--

* * *

如果你以前也深入了解过KeePass 2和KeePassX，或许会注意到二者使用不同的数据库格式。

KeePass 2使用一种新的版本格式，比如允许自定义字段。尽管KeePassX目前还不支持新的.kdbx格式，不过正在开发中的新的版本会支持的。

可以预览一下新版本的KeePassX，界面大为改善。你也可以从[GitHub](https://github.com/keepassx/keepassx)下载后自己安装。

![2014030320193421040.png](http://drops.javaweb.org/uploads/images/128d3b4c8bd7c0f4cc06bd3a4557363b4a032b76.jpg)

其他建议
----

* * *

windows下此软件叫"keepass"

正如本文开头所说，我在寻找能够跨平台的东西。这正是.kdb格式的优点 - 很多应用都支持这种格式。KeePassX 在 Mac OS X上运行起来要比KeePass 2容易得多，在windows上也可以。

Android系统上，我使用[KeePassDroid](https://play.google.com/store/apps/details?id=com.android.keepass&hl=en_GB)，在我的手机和平板上运行都很稳定。

via:[http://www.omgubuntu.co.uk/2013/10/manage-passwords-securely-keepassx](http://www.omgubuntu.co.uk/2013/10/manage-passwords-securely-keepassx)