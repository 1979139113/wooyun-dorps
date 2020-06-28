# 智能设备逆向工程之外部Flash读取与分析篇

**author: rayxcp**

0x00 前言
=====

目前智能家居设备的种类很多，本文内容以某智能豆浆机为例完成对其的固件提取和分析。

究其分析内部逻辑的原因可能会有很多种，和安全相关的原因主要有：

1.  了解设备内部运行逻辑，逆向后有条件更改原有逻辑
2.  通过逆向后的代码找到可利用的漏洞或原有隐藏功能

0x01 读取Flash
=====

首先，准备好螺丝刀，镊子等工具。把设备拆解。取出设备主控板。如下图：

![](http://drops.javaweb.org/uploads/images/dfe604c940b8c1441cdbbd27e851ddb0b927549a.jpg)

正面，其中红圈所在的黑色小板是Wi-Fi模块，上面的8脚Flash的芯片已通过吹焊台的热风机取下。

![](http://drops.javaweb.org/uploads/images/e6d4cc3a3ceb608fb39f03bdfca13a342bd4ddb0.jpg)

反面图

当目标为取得设备的运行逻辑时，首先需要找到运行逻辑存放的位置。以STM32 MCU的开发板为例，会有多种boot模式，其中一种boot模式是从MCU内部的Flash中启动，也就是说系统启动初期的逻辑在这种模式下是从MCU内部Flash读出的。但是这部分Flash的大小受限，通常容量情况为32KB ~ 512KB。那么其他的逻辑会通过SPI或者总线接口连接的外部Flash或SD卡等存储设备存放。针对这一款设备，除了MCU内部Flash外，板上还预制了一颗容量为2MB的外部Flash。

本篇关注外部Flash上的逻辑获取，通过SWD方式读取MCU内部Flash和调试等方法会在后续章节介绍。智能家居设备上一般的外部Flash为SOP8宽体芯片。见下图插座上的8脚芯片：

![](http://drops.javaweb.org/uploads/images/92f7f5def687fadf47233c60e6330cda1d57f2a4.jpg)

图中红圈的芯片就是从设备Wi-Fi模块上取下的Flash

Flash厂商品牌常见的有: winbond，gigadevice等。

以gigadevice GD25Q16名称为例

*   GD为厂商名
*   25为芯片系列号
*   16为MBit容量 也就2MB字节大小的Flash

Flash的内容可以通过编程器读取，首先在PC端安装相关的驱动和软件(在购买时会由商家提供)后，就可以读取Flash内容了。对编程器的选择上建议一定要带上插座，这样可以减少芯片的吹焊次数。此外还有芯片夹，可以夹住板上的Flash芯片，直接尝试读取，免于取下的过程，但是实测成功率不是很理想。

一般连接PC后，编程器配套的软件会自动识别出芯片类型和大小等信息。如果没有可手工尝试选择近似的型号（根据芯片名称）。

![](http://drops.javaweb.org/uploads/images/f9fc381e81e3fe54482d95bf16fb3d421f2d5332.jpg)

点击读取，待读取结束后，可以将读出的内容保存为bin文件。至此就获取到了Wi-Fi模块上Flash中的二进制内容。

0x02 逆向分析
=====

在得到bin文件后就可以开始逆向分析了。

从设备MAC地址以及一些bin文件中特征字串和Wi-Fi模块的外观形态判断，该设备Wi-Fi模块使用的是上海汉枫公司（之后简称HF）提供的HF-LPT100S-10。该模块包含一枚HF自研的MCU(HF-MC101，内置128KB SRAM)。而模块上的Flash大小2MB。

将bin文件放入IDA，在加载的第一步中设置CPU类型为ARM Little End（根据该款设备的CPU类型）。之后会出现内存选项，如下图。这里可选设置RAM的起始位置和大小，这一步将方便之后设置RAM中的变量名，建议根据芯片实际情况进行设置。这里RAM是128KB，起始位置是0x20000000。

![](http://drops.javaweb.org/uploads/images/7be91fc571db764c4271b1b5ba9214316017b133.jpg)

现在需要分析bin中包含的逻辑。在HF官网上提供了SDK的下载(基于Keil开发环境)。通过查看SDK包含的文档，得到如下针对ROM内容的结构描述。

![](http://drops.javaweb.org/uploads/images/de4739152f6ede363df19f4a6757581829bdaff2.jpg)

这时可以建立一个ida py脚本，将得到的信息添加其中，在IDA中标注出各个segment：

![](http://drops.javaweb.org/uploads/images/3dd7146848e5a7fafea757be5c35760a8a757dae.jpg)

当然也可以直接按长度，提取出各个段，例如提取出bin文件前512KB的app_main内容（见下面的附件）。

在进一步分析代码前可以先标记出基础库，例如标记出文件管理，内存操作等相关函数。一般有两种方式，第一种从文件自身的调试打印信息或者符号表包含的信息反推出函数；第二是加载获取的FLIRT类信息到当前文件。

FLIRT是Fast Library Identification and Recognition Technology的缩写，可以在[https://www.hex-rays.com/products/ida/tech/flirt/index.shtml](https://www.hex-rays.com/products/ida/tech/flirt/index.shtml)中找到相关的描述。简单的说就是通过建立库函数中使用汇编指令串以及各类引用的特征值匹配到当前当目标文件，标注出已知的各种函数名。当然定位函数，除了单纯指令串特征值外，诸如bindiff类插件也会通过函数内逻辑，和相互间调用关系来确定函数，这样的好处是提高库函数的命中范围和提高准确度。类似技术运用在函数定位上可以参见下面的插件：

[https://github.com/devttys0/ida/tree/master/plugins/rizzo](https://github.com/devttys0/ida/tree/master/plugins/rizzo)

针对LPT100S，有两类库文件可以导入，一类是Keil自身的一些库，还有就是在HF提供的SDK中包含的库LPB100Kernel.lib。通过FLIRT工具包，可以从这些库中提取出sig文件。步骤如下（以LPB100Kernel.lib为例）：

1.  下载ida的sdk文件，解压其中的flirt.zip
2.  在bin目录下找到对应系统的目录，运行
    1.  Pelf LPB100Kernel.lib LPB100Kernel.pat
    2.  Sigmake LPB100Kernel.pat LPB100Kernel.sig
        1.  这时会报错，打开LPB100Kernel.exc，处理冲突函数并删掉开头处的注释
        2.  重新运行该条命令
3.  将生成的LPB100Kernel.sig复制到ida安装目录\sig\arm\下

在IDA界面中通过Load File -> FLIRT signature file方式加载通过上面步骤生成的sig文件。之后就能生成如下效果了。

![](http://drops.javaweb.org/uploads/images/a1ef146574b4a756a0142039e185bc5303ae95a6.jpg)

![](http://drops.javaweb.org/uploads/images/7b5a42b4fc6c741ec1eff776a63acb385a48a79d.jpg)

除了SDK中的库文件，Keil自带的C基础库在Keil安装目录下的ARMCC\arm\lib\中，可以按照同样过程生成sig文件，并加载至分析文件中。这样mem，str等基本的函数调用也会匹配上。

至此就能更方便的查看内部逻辑了。

0x03 感谢
=====

piaca和redfree

http://www.devttys0.com/

0xFF 版权声明
=====

本文独家首发于乌云知识库(drops.wooyun.org)。本文并没有对任何单位和个人授权转载。如本文被转载，一定是属于未经授权转载，属于严重的侵犯知识产权，本单位将追究法律责任。