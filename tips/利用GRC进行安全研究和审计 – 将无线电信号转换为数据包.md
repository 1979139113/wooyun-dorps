# 利用GRC进行安全研究和审计 – 将无线电信号转换为数据包

0x00 介绍
=====

InGuardians作为一家从事信息安全研究和咨询的公司，自创立以来不但关注着web应用的渗透测试，网络取证，嵌入式设备等领域也致力于无线网络的评估方法上面的研究。在期间无线网络评估也从起初单一的企业无线网络部署慢慢地发展到开始涉及一些通用或自定义的蓝牙，zigbee等网络的分析。

InGuardians和其它一些企业，安全机构一样会一直通过参考其它人发表的一些研究结果来扩充自己的知识。在利用別人发表的内容来提高自我水平的同时，他们也从来没有在分享自己的一些经验和研究结果上有过止步。如果你有关注BH，DC等会议你们应该有在这些会议上看到过他们的身影。

在寻求充分理解无线电审计背后的技术过程当中，InGuardians意识到市面上缺乏一些一步一步指导人们如何分析无线电信号的文稿。其中最大的缺口出现在教会人们如何利用GRC（GNU Radio Companion）来对无线电信号进行彻底地分析，最终再将无线电信号转换为更易于我们理解和分析的数据包。虽然我们没有办法将个别案例中的分析方法原样地应用到任意的项目当中，但是了解一些分析案例中各个步骤背后的种种不但可以帮助人们开发出更好的工具，在某些时候也可以帮助人们完成其它类似的研究。作为工具 GRC Bit Converter3(GRC Bit Converter: https://github.com/cutaway/grc_bit_converter ) 就是个很好的例子。它可以很好地帮助安全研究人员和无线电爱好者完成他们的项目和工作。

0x01 安装与配置
=====

市面上有很多关于如何购买适合自己的无线电设备，如何安装GRC和如何安装频谱分析应用的教程。所以再重复叙述那些东西应该是没有任何意义的。在这里只给出这些硬件和软件的相关链接。

. 硬件资源

USRP . http://home.ettus.com/

HackRF Jawbreaker . https://github.com/mossmann/hackrf/wiki (.1)

bladeRF - http://nuand.com/

RTL-SDR . http://en.wikipedia.org/wiki/List_of_software-defined_radios

CC1111EMK - http://www.ti.com/tool/cc1111emk868-915

Ubertooth . http://ubertooth.sourceforge.net/

Atmel RZUSBstick - http://www.atmel.com/tools/rzusbstick.aspx

软件资源

GNU Radio Companion –

http://gnuradio.org/redmine/projects/gnuradio/wiki/GNURadioCompanion

SDR# - http://sdrsharp.com/

GQRX – http://gqrx.dk/

Baudline - http://www.baudline.com/

RFcat – http://code.google.com/p/rfcat/

Ubertooth - http://ubertooth.sourceforge.net/

KillerBee - http://code.google.com/p/killerbee/

通过学习并使用GRC(GNU Radio Companion)你可以很轻易地敲开通往无线电世界的大门。GRC十分易用的图形操作界面可以让你很轻易地实现一些复杂的无线电功能。GRC中的许多特定操作都重度依赖于那些可配置的模块（block）。通过对不同的block进行连接和配置，最终就可以实现我们想要的功能。市面上有大量的教程教授人们如何使用GRC和它的主要模块来捕捉并显示我们所需要的无线电信号。了解并探索如何入门GRC对于本文的读者来说应该是不错的锻炼机会。InGuardians建议大家可以试着用一下我们在上面所列出的那些软件。使用SDR#，GQRX等软件可以帮助读者对无线电传输的可视化和调整有一个初步的了解。这也许可以帮助读者更快地进入GRC的大门。

0x02 管理DC Spike
=====

HackRF,BladeRF或其它一些RTL-SDR的使用者在使用一些频谱分析工具进行调频时会看到一个很大的尖峰（Spike）。这个尖峰就被称做DC Spike。(图2中心处的尖峰)

![enter image description here](http://drops.javaweb.org/uploads/images/8bd753385b5a786d4f016d5eea26bb88851a2dab.jpg)

图2 DC Spike

当一些用户第一次看到这些Spike时可能会担心自己的设备是否存在缺陷，在这里可以告诉大家这和你选购的硬件或该硬件所使用的固件并没有任何的关系，只要你使用无线电DC Spike就会与你同在。关于DC Spike，在这https://github.com/mossmann/hackrf/wiki/FAQ你可以看到一些很好的解释。

正如HackRF FAQ当中所描绘的那样，在多数情况下我们是不需要考虑DCSpike的。但是，你如果想要成功地解调你所捕获的信号或捕捉被发射的信号你就需要尽力去保证这个信号是干净的。因此，为了避免受到DC Spike的干扰我们可以使用“DC Offset”来解决此类的问题。我们需要做的就是通过正确的GRC模块将接收频率调至无线电的传输带宽之外。唯一需要注意的是，我们需要选择一个合理的offset将DC Spike移动到传输的信号之外，但同时要保证不要将offset设置过大导致我们不得不使用我们本不需要的带宽。

通常确定如何去配置DC Offset的方法有两种。第一种方法，需要我们对产品手册中的数据进行分析来确定无线电设备的性能。在手册当中你通常可以找到关于信道间隔的标注。简单的来说，信道间隔就是从中心频率到两个不同传输区域之间的距离。借助这个信息我们就可以很好的调整我们的DC Spike。信道间隔和传输的大小没有必然的联系。例如,在观察wifi的信号传输时我们会发现，Wifi虽然有14条信道，但是在无线电适配器的传输当中会有6条信道被吞噬。除此之外，通过频谱分析工具对传输带宽进行观察可以帮助我们调节DCOffset，这也是我们之前提到的两种方法中的第二个。图3就是将信道间隔应用到无线网格网络中实现FHSS很好的一个例子。在GRC FFT当中使用“PeakHold”选项，就可以图中看到DC Spike会出现的位置。

![enter image description here](http://drops.javaweb.org/uploads/images/1074874865de2937fee6f272af589f0080d191a5.jpg)

图3

无线电设备会通过设置信道间隔来防止信道之间的相互干扰。默认的信道间隔也会因设备和制造商而异。但幸运的是，因为这些值都是预设的，所以通常我们可以在制造商所提供的文档中找到这些值。如果你无法从产品附属的手册或产品供应商那里获取这些值，你也许可以在论坛,产品源代码或一些专利申请中找到你想了解的参数的值。

![enter image description here](http://drops.javaweb.org/uploads/images/f035741a885fea80d4ee774562f2fd30e7691236.jpg)

图4 DC Spike出现在传输信号内

如果你找到的文档不全或需要验证信道间隔的值，图形化分析可以帮助确定一个粗略的信道间隔。图4就是通过图形化界面来目测信道间隔很好的例子。其中的蓝色尖峰即为DC Spike，绿色波峰为传输的信号，而用红色标注出来的距离则为频道的大小。其中出现的个传输信号的边缘之间间隔就正好是信道间隔。

GRC当中通常会使用变量模块(variable blocks)来对一些值进行修改。在下面的例子当中我们将使用变量模块来定义信道间隔的值。一旦channel_spacing设定完毕我们就可以在其他模块当中使用我们所设定的值。为了使用信道间隔来控制DC Spike，我们将定义另外一个变量模块freq_offset并让其值等于(channel_spacing / 2) * (channel_spacing * .1)。借助这个公式我们就可以把DC Spike推移到信号的边缘部分。如果我们进一步将此处的1调整到10就可以将DC Spike推移到传输信号之外了（如图5所示）。图6中将告诉大家应该如何去设置相关的osmocom Source。

![enter image description here](http://drops.javaweb.org/uploads/images/84577a3ccd43141c9249566b97bdcb1b5566824b.jpg)

图5 freq_offset的相关配置

![enter image description here](http://drops.javaweb.org/uploads/images/36f4bc951ad4b18e4d41411ab565b22b5c798b71.jpg)

图6 设置相对应的osmocom Source模块

一旦我们完成了这些模块的配置后，我们将会所对应的区域中看到公式计算厚的结果呈现在其中。被计算后的的结果如图7所示。

![enter image description here](http://drops.javaweb.org/uploads/images/9ad09bfc9030859d7bc2b0382693a6e86f423936.jpg)

图7 GRC捕获配置图

最后在图8中可以看到我们已经成功地将DC Spike推移到了传输之外。

![enter image description here](http://drops.javaweb.org/uploads/images/a9623844d997fe2a5a2488b2585a36d582bb7221.jpg)图8

0x03 传输信号的分离
=====

一旦DC Spike的问题得到解决，我们就可以在不受它干扰的情况下对信号进行处理了。接下来需要我们去做的就是信号的分离。GRC当中存在两个模块可以帮助我们对信号进行分离。它们分别是Low Pass Filter（LPF）和FrequencyXlating FIR Filter（FXFF）模块。从功能上来将两者能为我们提供的几乎是相同的。但是相对于LPF，FXFF为我们提供了更多的选项可以让我们在分离或对传输的信号进行操作时有一个更好的开始。在这里我们先对LPF做一个简单的介绍。

在LPF的配置当中，第一个可调节选项是Decimation。通过对该值进行调节，我们可以修改即将输入进来的信号的采样率（Sampling Rate）。你会在众多FM解调的GRC例子当中看到Decimation。这个参数的取值通常会和图5中的freq_offset一样由计算公式来完成。虽然使用Decimation在有些信号解调中是需要使用到的，但是在我们这次的例子当中我们不会用到它。所以我们可以让它保持在默认的值“1”,这样就不会对我们的采样率（Sample Rate）有任何的影响。这种做法也能同时保证我们不会去打破采样定理中提到的每一个正弦周期需要进行两次采样的说法。

下一个需要我们去设置的是“Window”这个选项。这个选项的默认值是“Hamming”。但是在Balint Seeber的“Blind signal analysis with GNU Radio”演讲当中他曾经提到我们应该将Window的值设置成“BlackMan”，因为这相对于另外一个来说是更优秀的算法。

在设置完Decimation和Window参数之后，还需要我们设置SampleRate，Cutoff Frequency和Transition Width来完成我们的信号分离。Sample Rate的设置很简单，它是输入采样率并应当和前面所提到的一样被默认的设置成“samp_rate”。这里需要注意的是即使我们使用decimation修改了输出的采样率，Sample Rate应当依旧设置成我们的输入采样率。Cutoff Frequency为我们之前提到的图4中所显示的频道大小。然而频道间隔的设定也应当和我们在之前所提到的一样，应该有同名的变量模块进行设定。

其中最为麻烦的可能就属Transition Width的设定了。在设定这个值之前我们需要做一些测试来给它赋予一个正确的值。更多的经验和调查将会对于你在起初如何设定这个值有很大的帮助。当然这也会取决于中心频率，带宽，无线电设备类型等因素的影响。有些时候传输的信号不会恰好地集中在中心频率。它时常都有可能受到天气，Power，甚至是其它信号的干扰。总而言之，该值的不合理设定可能会导致多余信号的产生，又或者是信号的丢失。根据我们的经验，我们会将该值设定为频道间隔（Channel Spacing）的４０％～５０％之间的值。在后面我们也会看到这个设定可以为我们带来最好的结果。在图9中我们将使用下述的参数来对LPF进行配置。

Frequency Offset 120,000

Channel Spacing 200,000

Channel Transition 80,000

Window BlackMan

![enter image description here](http://drops.javaweb.org/uploads/images/f38b0079d3225753d73fc734a6de83bc10291d6e.jpg)

当我们用上述的参数观察我们分离出来的信号时，我们发现我们的信号并没有位于FFT Plot的中心处。然而为了正确地解调信号我们需要把信号移动到FFT Plot的中心处。这里就需要我们使用Frequency Xlating FIRFilter（FXFF）来实现这个操作。

FXFF和我们前面所提到的那样，和LPF有着许多相似之处。所以在这里我们可以将Decimation和Sample Rate设置成与之前在LPF中同样的值。在FXFF当中还有一个叫Center Frequency的变量。这个变量和DC_Offset一样通常用来矫正信号并将其调整到中心处。如果在设置过程当中我们没有使用DC_Offset那么该值应当设置成０，如果不是它的值应当设置成freq_offset。最后一个需要我们去设置的是Taps变量。对于这个变量的设定，我们使用Dragorn在他的博客Playing with the HackRF – Keyfobs中所提到的公式：“ firdes.low_pass(1, samp_rate,channel_spacing, channel_trans, firdes.WIN_BLACKMAN, 6.76) ”。（详见图10）

![enter image description here](http://drops.javaweb.org/uploads/images/31cd58a6733bd1179589548a9368ec576725474d.jpg)

对于FXFF进行了上述的设定之后我们就可以在图11中看到我们所预期的结果。

![enter image description here](http://drops.javaweb.org/uploads/images/c0092ba77bf647b8bcf5d388059e5e5a8a2485d3.jpg)

0x04 实际调解
=====

使用LPF和FXFF对信号进行分离可以帮助我们更有效的对信号进行解调。Michael Ossmann在他的无线电培训课程当中讲述了从这个点开始应当如何进行解调。在他的课程的当中他不但讲述了相关的概念和数学知识，也告诉了我们为了解调不同的ASK（amplitude-shift key）和FSK(frequency-shifykey)所需要的模块。解调ASK我们可以使用“Complex to Mag” 或“Complex to Mag ^ 2”。然而解调FSK（尤其是2FSK和GFSK）我们可以使用“Quadrature Demod”。

在这个例子当中我们需要解调的是通过GFSK传输的信号。在此之前的图片当中我们所看到的一些结果都是Texas Instrument (TI) Chronos Watch和IChronos black dongle之间的数据传输结果。在TI的站点上我们可以找到和这个dongle相关的一些数据，甚至是它的源代码。最终我们在这个产品的源代码中看到了我们想了解的信息（Mudulation=（1）GFSK），如图12所示。

![enter image description here](http://drops.javaweb.org/uploads/images/398d305cf2a31b3177c488f7804ac59cb9fdecaa.jpg)

现在我们已经知道了Modulation type，那么剩下的就是需要知道Deviation了。这个值我们可以从源代码中获取（可以看到在图12 当中 Deviation=32Khz）。

当我们对在需要解调GFSK索要用到的“Quadrature Demod”模块进行配置时，我们会看到的默认的Gain变量将通过这样的公式进行计算：samp_rate/(2_math.pi_fsk_deviation_hz/8.0)，这意味着我们还需要添加一个变量模块fsk_deviation_hz 来协助Gain的计算。这个fsk_deviation_hz就正好是我们之前提到的需要我们知道并且设定的第二个变量Deviation（如图13所示）。这也是为什么我们会说除了Modulation type之外我们还需要知道Deviation的值。

![enter image description here](http://drops.javaweb.org/uploads/images/d0f98f83f0088343281bb8e326d6d4a901b808d2.jpg)

在图13当中我们可以看到模块Quadrature Demod处理结果最终会输出到File Sink模块当中。这是因为从这一步开始我们需要将解调过后的结果放到其它的工具里来完成我们之后的步骤。在这里需要提到的一点是，关于文件的命名和输出的位置。有时候我们需要在很短的时间内将很大的文件写入到磁盘。如果你是Linux用户，我们建议你将文件写入到“/dev/shim”，这样做会比直接写入到硬盘会快一些。关于文件命名你可以参考将一些可能会用到和重要的信息放到文件名称当中，方便日后的分析和处理工作。如，

“/tmp/blog_demod_1e6_905.99e6_gfsk_76721_hackrf.cfile.”

在完成了这些设定之后，运行图13中的GRC会将调节后的信号存储到我们预设定的文件当中。将此文件导入到频谱分析工具Baudline（Baudline:http://www.baudline.com/）中，就可以继续我们下面的工作了。如果你对此处的一些细节有疑问，可以在Dragorn发布的Keyfob blog post中看到一些更详细的解释（评论中有很多亮点）。图14所展示的是，我们将文件导入到Baudline之后所呈现的FFT和波形图。

![enter image description here](http://drops.javaweb.org/uploads/images/bb023688b14aef0efe7d5cea38bb6de9a4614663.jpg)

通过使用Baudline我们可以在把无线电信号转换成数据包之前对我们得到的结果进行分析和判断。现在我们将上图中的波形图进行放大来进一步观察。（如图15 所示）

![enter image description here](http://drops.javaweb.org/uploads/images/e40112a79d06c2ec2c9eb95a87549647fd7d8165.jpg)

通过上图对我们解调过后的信号进行观察，我们可以发现一些很有价值的信息。首先我们可以从图中，看到我们获取的信号并没有我们预期的那么干净。其次，波形图整体偏下，且并没有穿过中心轴（X-axis）.然而这些对于我们将无线电信号转换成0和1是十分的不利的。为了实现信号的转换我们需要在Quadrature Demodulation和File Sink blocks之间加入另外一个LPF。就像在之前所提到的那样，这个模块需要我们配置它的“Cutoff Freq”和“Transition Width”变量。然而问题是，现在还并没有什么好的文档能告诉我们该如何去设定这个LPF当中的这两个变量。这些变量的取值一般可能会受到频率，Modulation和其它一些配置的影响。根据经验，我们通常会从100,000开始设定“Cutoff Freq”并将“Transition Width”的设置为它一半的值。通过重复调节这些参数并将输出文件导入到Baudline进行观察，再经过几轮甚至是更多次的调试之后我们将会在Baudline当中看到看上去更像是数据的波形图。图16中的波形图，就是我们将“Cutoff Freq”设置成80,000，“Transition Width”设置成50,000后的结果。

![enter image description here](http://drops.javaweb.org/uploads/images/d98d369476f6789630803f6534cfdbd87f20501e.jpg)

现在我们可以看到我们的信号看上去好了很多。但是波形和中心轴的问题依然没有得到解决。解决这个问题，只需要我们在LPF和File Sink当中再加入一个Add Const模块来对waveform进行整体上移。关于如何设置AddConst的值，你可以参考之前LPF的调节方式，对其值进行逐步增加，直到出现如图17所示的画面（在这里我们将contast设定为6）。

![enter image description here](http://drops.javaweb.org/uploads/images/8fbe9860bcfdd864f34325787fd531ce5f117945.jpg)

了解如何在不丢失数据的情况下对wave pattern进行操作在我们分析其它的信号时也十分的重要。举个例子来说，如果厂商的文档中有提到数据是倒置之后进行传送的，那么我们就可以通过设定contast值为-1,来进行反向的数据反转。

0x05 位计数
=====

在完成了解调之后就需要我们从wave中检测出所有的0和1.在GRC中我们可以使用“Clock Recovery MM”和“Binary Slicer”来实现它。其中的“Clock Recovery MM”负责从wave中找出高位和地位，再由“Binary Slicer”负责将它们转换成1和0。首先，我们需要配置“Clock Recovery MM”模块。在打开“Clock Recovery MM”的配置界面后（如图18所示）你可能会被这些多的有点让人难受的变量给吓到。但是实际上其中的“Gain Omega,”“Mu,” “Gain Mu,” ， “Omega Relative Limit”等变量在一般情况下是不需要我们去重新设定的。所以只要让它们保持默认的值就可以了。 所以在这里唯一需要我们去设定的是，“Omega”变量。在“Omega”的默认值当中我们可以看到，这样的设定：samp_per_sym*(1+0.0)。这意味着我们需要和之前一样添加一个变量模块samp_per_sym来帮助“Clock Recovery MM”计算每帧抽样次数(Samples Per Symbol)。

![enter image description here](http://drops.javaweb.org/uploads/images/406ab3c76d51b4d991e1f8d8888cb1af3de8b498.jpg)

这里的samp_per_sym可以通过公式“int(samp_rate / data_rate)”进行计算（如图19所示）。samp_rate对我们来说是已知的，但是data_rate呢？这个值我们可以从图12中找到。所以为了计算出samp_per_sym我们还需要再添加一个变量模块“data_rate”，并将其值设定为图12中所显示的76721.191。

![enter image description here](http://drops.javaweb.org/uploads/images/2cd6252b4e557e62910c0e2da691e7e00f70f071.jpg)

最终我们将配好的模块接入到我们之前做好的GRC当中（从图20中可以看到原始的File Sink处于Disabled状态）就应该是图20所呈现的样子

![enter image description here](http://drops.javaweb.org/uploads/images/b4c9f47b2b6e5907ed643f3bb460ce4752659d54.jpg)

0x06 分析解调后的数据
=====

在我们运行图20所展现的GRC之后我们会得到一个文件。通过Linux的xxd或任何平台上的hex editor打开我们所获得文件后你会看到如图21的内容。

![enter image description here](http://drops.javaweb.org/uploads/images/39c408bb8e9b5710a2e32da73eb092fe6b2d4946.jpg)

和我们预期的一样我们可以在上图当中看到满满的0和1。其中每8个bit可以视作是一个bit的数据。比如第一行的0101 0101 0101 0101 0101 0101 0101 0101就应该是0b1111111111111111或0xff。第三行的0101 01010101 0101 0001 0100 0100 0101就应该是0b1111111101101011或0xff6b。

但这些输出可能是被传输的数据，同时也可能是任何一些可以被0和1所表示的其它一些东西。所以在这里找出真正的数据将会是我们最大的挑战。而且根据经验，图21中所展现的数据在我们看来和空气中传播的杂音（noise）有极高的相似度。在我们现在配置当中“Clock Recovery MM”会将所有的高位和低位扔给“Binary Slicer”进行处理。然而从现在的数据来看和实际数据对比，我们收到了太多的杂音（noise）。也就说我们现在的GRC配置还不具备辨别数据，杂音和来字其它无线电设备的信号的功能。也许我们可以通过定位数据包的头bit来对实际产生的数据进行定位，但是GRC是否有这样的模块可以实现我们想要的这个功能呢？

除此之外，我们在图21中看到的数据还只是raw data。所以我们需要将这些数据转换成byte再去讨论如何辨别杂音或找到真实数据的问题。在这里可以借助InGuardians开发的GRC Bit Converter（https://github.com/cutaway/grc_bit_converter）将数据转换成bytes（如图22所示）。

![enter image description here](http://drops.javaweb.org/uploads/images/d0efa9cf8cd0272d4770f0c5e68a93960e8d9abc.jpg)

使用默认设置运行grc_bit_converter.py，脚本会将每2000个bits转换成250个bytes并将其标记为一个数据包。这个流程会一直的重复，直到处理完所有的数据。其中的参数“Occurrences”会告诉我们有多少个包内的数据是一模一样的。这个参数在我们找到数据包之后会有很大的用处。

在完成转换之后我们需要确定传输的数据是如何被格式化的。这将有助于我们从这一坨数据当中找到它们。很多无线电设备（并不是所有的，大多数都是这样）在进行数据传输之前会发送前导码（preamble）和同步码（SyncWord）。前导码是一系列从高到低传输，如果从binary文件来看的话它通常会是这样“0b1010101010101010” 或这样 “0xaaaa”。前导码byte数取决与设备本身或它的相关配置。由于没有办法确定前导码的首位是什么所以在我们对binary文件进行过处理之后，你在解析过后的文件当中看到的前导码可能已经变成了“0101010101010101”或“0x5555”。 前导码通常只负责告诉设备接下来的数据是需要处理的。而真正告诉告诉设备数据是从哪里开始责由同步码负责。在真实世界的无线电传输当中最常用的同步码是“0xd391”。但是就和之前我们提到的前导码问题一样，在我们解析完之后“0xd391”很可能已经变成了其它的值。这时就需要我们使用ipython来对这个“0xd391”进行小小的处理了。在图23中你们会看到可能会变成0xa722, 0x4e44或0x9c88出现在我们解析过后的文件当中。

![enter image description here](http://drops.javaweb.org/uploads/images/30ac67c69fa1f51fc777a9b0ae9628605a385269.jpg)

在有了一点线索之后，现在让我们试着从grc_bit_converter.py的输出结果中查找我们的同步码（0xd391）来定位我们的数据包(如图24所示）。

![enter image description here](http://drops.javaweb.org/uploads/images/3236538049272a16e334c864a98fc319e7dddf9f.jpg)

这下我们好像终于找到了看上去像数据的东西。当然了，我们也可以通过再此回顾图12中关于同步码和前导码的描述来确定我们的操作的准确性。在这里我们可以看到我们之前所提到的前导码是“\xaa\xaa\xaa\xaa”这个图12中叙说的4 bytes完全吻合。然而我们的同步码“\xd3\x91\xd3\x91”正好是32bits，也和源代码中描述检测32bits的同步码的30位这一说相吻合。然后在源代码中我们还看到同步码后面的第一个byte表示整个数据包的长度。从上图中我们可以看到同步码后面的第一个byte正好是\x0f也就是15。从这里开始数15个byte我们会发现还会剩下两个byte，这两个byte就是CyclicRedundancy Check (CRC)了。这两位被称为数据错误，循环冗余检查通常用于检查数据的正确性和完整性。

0x07 对数据包进行标记
=====

当我们手头上的信息十分的有限时，手工或通过脚本对数据进行解析有时候可能会是我们唯一的出路。但是这个案例当中我们拥有足够的信息来运用相关的GRC模块进行数据包的标记。

这个动作可以使用GRC中的“Correlate Access Code”模块完成。“Correlate Access Code”模块需要我们配置其中的两个变量来完成标记的操作。它们分别是“Access Code” 和 “Threshold”。其中的“Access Code”是我们需要定义的特征向量来对需要我们进行标记的包进行表示。然而另外一个参数“Threshold”是特征向量中可能会发生变化的部分的位数。在图12当中我们看到在检同步码时，只会检测32位当中的30位，这也就意味着其中的两位可能是不一样的。那么我们这里的“Threshold”的值就应该是2了。特征向量“Access Code”和我们之前提到的一样应当填入同步码所对应的值（如图25所示）。

![enter image description here](http://drops.javaweb.org/uploads/images/d9d5cbae492fb6d9dc9b1861e69d3165c479aeb9.jpg)

和之前所提到的一样，“Correlate Access Code”标记的是“Binary Slicer”处理后的数据。相关的位置关系将在图26展示。

![enter image description here](http://drops.javaweb.org/uploads/images/d9df65933b7942a1eb0d6d652c35c4917db97443.jpg)

在完成文件的输出之后，我们可以像之前一样用xxd或hex editor打开输出的文件，并通过搜索02或03来查找我们标记的数据包。（如图27所示）

![enter image description here](http://drops.javaweb.org/uploads/images/29633ae6bde17ad99ff7fb3e5fa1ce9d7d166468.jpg)

就和前面使用grc_bit_converter.py对输出的文件进行分析一样，这次我们同样可以使用这个脚本。这次需要我们在脚本后面增加几个参数来检测“Correlate Access Code”模块标记过的数据包，并修改数据包的大小（如图28所示）。

![enter image description here](http://drops.javaweb.org/uploads/images/450c87698ca9bab73adaf3ba11a81bcba3066c43.jpg)

0x08 验证
=====

在这个例子中的GRC配置对于我们的TI Chronos Watch来说工作的还算不错。但是文中提到的这些技术是否能同样的应用到其它的或类似的无线电传输当中呢？为了验证，我们会使用http://lists.gnu.org/archive/html/discussgnuradio/2014-04/msg00097.html中Jay为我们提供的使用DC Offset抓取的包进行验证。

为了对这个包进行分析我们需要将原来的sources模块替换成“File Source”和“Throttle”模块。在这个案例当中capture rate是500,000，然而DC Offset是200,000 Hz。这个GFSK信号在903 MHz进行传输（如图29所示）。

![enter image description here](http://drops.javaweb.org/uploads/images/48597ac98be703d5abc062a32fb3ed175b12f87e.jpg)

从图29可以看出，我们捕获到的信号靠近图形的左边缘。我们还可以发现频道大小从902,950,000 到903,100,000将近为150,000 Hz。随后再将FXFF中的频道间隔设置成100,000，“Channel Transition”设置成50,000后我们可以可以在图30中看到变化。

![enter image description here](http://drops.javaweb.org/uploads/images/155ca302a4e8af16247513ebb1653871c18eeb54.jpg)

通过Jay的Post我们还可以知道其它的一些信息。比如：

. Deviation 16,500

. data rate 19200

在使用这些值完成我们前面所做过的步骤后，我们又可以到可以将信号导入到baudline进行调整和分析的步骤了。经过多次的调整我们最终将“CutOffFreq”和“Transition Width”分别设定成了25,000 和 10,000。调整过后的波形图如图31所示。

![enter image description here](http://drops.javaweb.org/uploads/images/9a32d54ce7e31aae9c87e2a4b1d44c8b546d5c74.jpg)

从图中我们知道波形图还需要做一些上下的调整。在这里我们需要将“Add Constant”设置成-6,来让wave向下进行移动。在完成了wave的调整之后需要我们像之前那样验证同步码，最终得知只会发送一个"0xd391"。接住这个信息我们就设置“Correlate Access Code”来对包进行标记。再将包的大小设定为80之后我们再次调用grc_bit_converter.py对结果进行观察（如图32所示）。

![enter image description here](http://drops.javaweb.org/uploads/images/bac1bed79524c362d2a326a274c374dec9f77c52.jpg)

在完成了这些之后下一个步骤将会是对判定在这些数据传输中所使用的协议。从包的长度，CRC bytes，是否使用数据白话技术等入手将对我们分析这些数据会有很大的帮助。作为额外的数据捕获我们还可以尝试在接上电源，关闭电源，配对开始的情境下进行包的捕获。捕获的数据信息越多,就越有利于我们了解它们是如何进行通讯的，是什么中端在进行数据传输，我们应当如何去设置GRC来实现我们所需要的通讯。

0x09 结论
=====

通过适当的设备和软件我们可以轻易的捕捉到我们想捕捉的信号。然而理解如何使用GRC来对捕捉到的信号进行操作会稍微困难一些。这使得对信号进行解调并转换成数据包变得更加复杂。但幸运的是，我们可以通过互联网阅读许多人分享的分析案例。通过理解这些案例可以使得我们的分析过程变得轻松许多。