# Kemoge病毒分析报告

最近，哈勃分析系统捕获了一类恶意病毒，该类病毒会主动获取root权限，私自安装其他应用，卸载安全软件，给用户带来巨大风险。

0x01 传播途径
=====

伪装成正规应用进行传播。

0x02 恶意行为概述
=====

该病毒监听用户解锁动作和网络连接变化启动自身后，解密资源文件info.mp4，该文件解密后为包含多个root工具和恶意AndroidRTService.apk的zip包，用于获取root权限；一旦获取root权限后，拷贝AndroidRTService.apk到/system/app目录，AndroidRTService.apk会访问服务器获取指令，卸载、下载安装其他应用及弹出各种广告。

0x03 详细分析
=====

3.1 样本监听USER_PRESENT和CONNECTIVITY_CHANGE广播启动后，判断是否已经root：
---------------------------------------------------------

![](http://drops.javaweb.org/uploads/images/1ed7672b80cb360b5cb180072a8908652802f089.jpg)

3.2 如果还未root，就进行root操作：
-----------------------

root所需要的工具都隐藏在由DES加密过的资源文件info.mp4中，样本会先解密info.mp4文件，然后尝试进行root。

### 3.2.1 解密资源文件info.mp4：

资源文件info.mp4由DES加密，然而DES秘钥被再次加密：

![](http://drops.javaweb.org/uploads/images/10559dcc3bf72831273d52e4421bdc73d75ce2cb.jpg)

最终解密后的DES key为：a1f6R:Tu9q8。

由DES key解密资源文件info.mp4为info.mp4.zip，该zip文件需要密码才能被解开：

![](http://drops.javaweb.org/uploads/images/f780a98a2cdd88c3df02095db9d59ae48a05488c.jpg)

![](http://drops.javaweb.org/uploads/images/9a94397fd8efcc377bf526e41de5f72a24ae28db.jpg)

### 3.2.2 获取zip包解压缩密码：

解压缩密码由另外一DES加密：

![](http://drops.javaweb.org/uploads/images/bda7e8245f29865fcee0cf046590c746e9b2f5b6.jpg)

最终得到的解压缩密码为：6f95R:T29q1。

### 3.2.3 解压缩zip包：

![](http://drops.javaweb.org/uploads/images/124eafdf6ed69e0372a13ea78b9bacaa9a02ddc6.jpg)

zip包里包含了各种root工具(root_001~root_008)、权限管理工具及恶意apk。

3.2.1~3.2.3描述的解密过程可表述为：

![](http://drops.javaweb.org/uploads/images/4047cf300f671cb6d0fc71bf0372e004f3d46fbe.jpg)

### 3.2.4 root 操作：

调用root工具root_00*直到获取root权限成功为止：

![](http://drops.javaweb.org/uploads/images/e452d8566a2cc8c3a4c4542c4a27a5e1b3dde160.jpg)

3.3 获取root权限后，将AndroidRTService.apk拷贝到/system/app目录下，并命名/system/app/Launcher**a.apk，以混淆用户，防止被发现：
------------------------------------------------------------------------------------------------

![](http://drops.javaweb.org/uploads/images/da7ee3b21e16992b036ca228cbc3c77a5174abf4.jpg)

![](http://drops.javaweb.org/uploads/images/a0b8739b068e1a6649cb3c2139bebd93d4aff8a3.jpg)

3.4 清理工作，删除root过程中生成的文件，防止被发现：
------------------------------

![](http://drops.javaweb.org/uploads/images/c7da15486716073e7e1f08801be86c4aa58a4eb3.jpg)

![](http://drops.javaweb.org/uploads/images/4862af7404692ca13e6700e71d73c7d74dd26103.jpg)

3.5 恶意AndroidRTService.apk启动后，获取手机基本信息，访问服务器获取指令：
-------------------------------------------------

![](http://drops.javaweb.org/uploads/images/d13a4b4b82dd0fdebe012f80bbe609b9ac0b7c43.jpg)

获取广告信息：

![](http://drops.javaweb.org/uploads/images/9cffb896f83c374e18ace6436cb2e878d77c88c2.jpg)

![](http://drops.javaweb.org/uploads/images/02469021312eb4cd7aac6506d6bdb232514b565c.jpg)

此外，还会获取安装、卸载指令，根据获取到的指令进行相应操作：

![](http://drops.javaweb.org/uploads/images/93c6d2d759bd6ac50dca17fe42f4ea68a3bb7b30.jpg)

![](http://drops.javaweb.org/uploads/images/c388ac92af9c360fb4bc6df017fe1f7c50a31b9e.jpg)

![](http://drops.javaweb.org/uploads/images/5d290dca1764a8f63cb9a41a51c3531dbb2903c6.jpg)

安装推广应用：

![](http://drops.javaweb.org/uploads/images/98e3059809747393f54c32b057632b45989bd3aa.jpg)

0x04 查杀
=====

腾讯哈勃分析系统识别：

![](http://drops.javaweb.org/uploads/images/a916ab5fb6f4a1ed6e55f9fbc31ed61d1d4945aa.jpg)

腾讯电脑管家和手机管家识别：

![](http://drops.javaweb.org/uploads/images/ae3cfa3699fe749d1efd2dde10e09b7bc8895194.jpg)

![](http://drops.javaweb.org/uploads/images/cc9b98d1f22f0ff3ac81aead488abf18a896ceac.jpg)

样本数据：[infected.zip](http://static.wooyun.org/drops/20151013/2015101302183865870infected.zip)