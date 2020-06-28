# 利用勒索软件Locky的漏洞来免疫系统

From：[https://www.lexsi.com/securityhub/abusing-bugs-in-the-locky-ransomware-to-create-a-vaccine/?lang=en](https://www.lexsi.com/securityhub/abusing-bugs-in-the-locky-ransomware-to-create-a-vaccine/?lang=en)

0x00 简介
=====

在2009年，我们利用免疫系统的概念来保护工作站或者服务器免受快速传播的蠕虫病毒Conficker的入侵。让我们来看一下是否能把这一防御概念用在勒索软件Locky上。

下面我们将对系统进行一些小的修改，来打造一个免疫系统，在不需要用户交互的情况下，部分或者完全地阻止恶意程序在执行的时候对系统造成的危害。很明显，我们必须要让“疾病”到来之前使免疫系统取得控制权.......

这些小的改动可以是一个特别的互斥体或者一个注册表键的创建或者一个简单的系统参数的修改，但是这些改动不能对用户造成任何不便。这里有一个反例，Locky在开始执行的时候回检查系统语言，它不会感染系统语言是俄语的系统：

![p1](http://drops.javaweb.org/uploads/images/8152237a9e24647251164b81f11c0a42f61326b8.jpg)

因此，把系统语言改为俄语可以免受其感染，但是这样会给很多人造成使用上的不方便。

0x01 利用访问控制列表ACL 来阻止Locky注册表项的创建
=====

在检查完语言后，Locky会尝试创建注册表项HKCU\Software\Locky ，如果创建失败，它就会立即终止。

![p2](http://drops.javaweb.org/uploads/images/a697edbd9ea2b5b4449bafd2f07faac35e547881.jpg)

当这个键被创建以前我们利用ACLs来阻止任何人打开这个键，这样系统就得到了免疫。

![p3](http://drops.javaweb.org/uploads/images/07b5a32db05184a25c1a11467789ed6b92facb66.jpg)

0x02 注册表键completed 的值
=====

Locky会检查注册表项HKCU\Software\Locky的键，查找ID（被感染机器系统标识），pubkey（服务器上的公钥，后面详述）， paytext（以它指定的语言格式，显示给用户的文字） 和completed值。最后一项的值代表着对用户系统的加密过程是否结束。按照Locky的定义：如果completed被设置为1,同时id值为正确的系统标识符，那么它将停止执行。

![p4](http://drops.javaweb.org/uploads/images/4e2a1f3f0871d6f86c62a8b324e69b2f899ebc95.jpg)

这个标识符生成的算法比较简单，我们在测试机器上产生的结果如下：

1.  GetWindowsDirectoryA() : C:\WINDOWS
2.  GetVolumeNameForVolumeMountPointA(C:\WINDOWS) :`\\?\Volume{ b17db400-ae8a-11de-9cee-806d6172696f}`
3.  md5({b17db400-ae8a-11de-9cee-806d6172696f}) : 1d9076e6fd853ab665d25de4330fee06
4.  把上面的结果里字母转为大写，并取前16位: 1D9076E6FD853AB6

创建这样两个键值，其中id的值随着系统不同而不同，这样就能阻止Locky加密系统文件。

![p5](http://drops.javaweb.org/uploads/images/faab211d0264045d4b3b08829c6ca11cd0e15f75.jpg)

0x03 破坏RSA密钥
=====

在加密文件前，Locky会发送一个HTTP POST请求到它的C&C服务器，发送明文如下：

```
(gdb) hexdump 0x923770 0x65
88 09 0c da 46 fd 2c de 1d e8 e4 45 89 18 ae 46 |....F.,....E...F|
69 64 3d 31 44 39 30 37 36 45 36 46 44 38 35 33 |id=1D9076E6FD853|
41 42 36 26 61 63 74 3d 67 65 74 6b 65 79 26 61 |AB6&act=getkey&a|
66 66 69 64 3d 33 26 6c 61 6e 67 3d 66 72 26 63 |ffid=3&lang=fr&c|
6f 72 70 3d 30 26 73 65 72 76 3d 30 26 6f 73 3d |orp=0&serv=0&os=|
57 69 6e 64 6f 77 73 2b 58 50 26 73 70 3d 32 26 |Windows+XP&sp=2&|
78 36 34 3d 30                                  |x64=0

```

第一行是后面字符串的MD5值，这个数据在发送前会进行简单的编码：

![p6](http://drops.javaweb.org/uploads/images/e1d1d1ec23efa3c661297b1e6e1089738728a449.jpg)

用类似的算法可以解码返回数据：

![p7](http://drops.javaweb.org/uploads/images/87945f9e852ee56068f6ccdd51d83dbac4bf120f.jpg)

这两个算法可以用几行Python代码实现：

```
def encode(buff):
  buff = md5(buff).digest() + buff
  out = ""
  key = 0xcd43ef19
  for index in range(len(buff)):
    ebx = ord(buff[index])
    ecx = (ror(key, 5) - rol(index, 0x0d)) ^ ebx
    out += chr(ecx & 0xff)
    edx = (rol(ebx, index & 0x1f) + ror(key, 1)) & 0xffffffff
    ecx = (ror(index, 0x17) + 0x53702f68) & 0xffffffff
    key = edx ^ ecx
  return out
def decode(buff):
  out = ""
  key = 0xaff49754
  for index in range(len(buff)):
    eax = (ord(buff[index]) - index - rol(key, 3)) & 0xff
    out += chr(eax)
    key += ((ror(eax, 0xb) ^ rol(key, 5) ^ index) + 0xb834f2d1) & 0xffffffff
  return out

```

解码后数据如下：

```
00000000: 3af6 b4e2 83b1 6405 0758 854f b971 a80a :.....d..X.O.q..
00000010: 0602 0000 00a4 0000 5253 4131 0008 0000 ........RSA1....
00000020: 0100 0100 2160 3262 90cb 7be6 9b94 d54a ....!`2b..{....J
00000030: 45e0 b6c3 f624 1ec5 3f28 7d06 c868 ca45 E....$..?(}..h.E
00000040: c374 250f 9ed9 91d3 3bd2 b20f b843 f9a3 .t%.....;....C..
00000050: 1150 5af5 4478 4e90 0af9 1e89 66d2 9860 .PZ.DxN.....f..`
00000060: 4b60 a289 1a16 c258 3754 5be6 7ae3 a75a K`.....X7T[.z..Z
00000070: 0be4 0783 9f18 46e4 80f7 8195 be65 078e ......F......e..
00000080: de62 3793 2fa6 cead d661 e7e4 2b40 c92b .b7./....a..+@.+
00000090: 23c9 4ab3 c3aa b560 2258 849c b9fc b1a7 #.J....`"X......
000000a0: b03f d9b1 e5ee 278c bf75 040b 5f48 9501 .?....'..u.._H..
000000b0: 80f6 0cbf 2bb4 04eb a4b5 7e8d 30ad f4d4 ....+.....~.0...
000000c0: 70ba f8fb ddae 7270 9103 d385 359a 5a91 p.....rp....5.Z.
000000d0: 4995 9996 3620 3a12 168e f113 1753 d18b I...6 :......S..
000000e0: fdac 1eed 25a1 fa5c 0d54 6d9c dcbd 9cb7 ....%..\.Tm.....
000000f0: 4b8e 1228 8b70 be13 2bfd face f91a 8481 K..(.p..+.......
00000100: dc33 185e b181 8b0f ccbd f89d 67d3 afa8 .3.^........g...
00000110: c680 17d8 0100 6438 4eba a7b7 04b1 d00f ......d8N.......
00000120: c4fc 94ba                               ....

```

前16个字节是后面字节的MD5值，从偏移0x10处开始我们可以发现一个[BLOB_HEADER](https://technet.microsoft.com/fr-fr/aa387453)结构：

*   type 0x06 = PUBLICKEYBLOB
*   version 0x02
*   2 reserved bytes
*   ALG_ID 0xa400 = CALG_RSA_KEYX

这是一个RSA公钥，因此下面是[RSAPUBKEY](https://msdn.microsoft.com/en-us/library/windows/desktop/aa387685(v=vs.85).aspx)结构：

*   magic RSA1 = public key
*   key size: 0x800 = 2048 bits
*   exponent 0x10001 = 65537
*   modulus 2160…94ba

这个结构（除去MD5 hash部分）保存在上面提到的pubkey键值里，如果这个值存在，并且是一个错误的值，那么系统里的文件既不会被改名，也不会被加密。下图中，把pubkey用一个NULL字节填充，然后在测试机器上运行Locky，尽管Locky对txt格式的文件具有针对性，但是这时候桌面上的文件monfichier.txt 并没有发生变化。

![p8](http://drops.javaweb.org/uploads/images/d0e8b31c5bc9c90ed5f02a4c0140783076b43521.jpg)

另外如果pubkey是一个RSA 1024密钥，文件将会被改名，但是不会被加密(“ceci est un secret”，法语：“这是一个秘密”):

![p9](http://drops.javaweb.org/uploads/images/f7ced2d16d074dbbec355614aa0288252ac0ec95.jpg)

0x04 获取RSA 私钥
=====

Locky有另外一个设计漏洞：如果pubkey的值存在，Locky用pubkey加密文件的时候不做任何验证，允许我们强制使用我们自己掌控的一个公钥来加密文件，同时我们又知道这个公钥相对应的私钥。

下面的C代码将生产一个BLOB_HEADER 格式的RSA 2048 密钥对，用来作为pubkey的值：

```
#define RSA2048BIT_KEY 0x8000000
CryptAcquireContext(&hCryptProv, "LEXSI", NULL, PROV_RSA_FULL, 0);
CryptGenKey(hCryptProv, AT_KEYEXCHANGE, RSA2048BIT_KEY|CRYPT_EXPORTABLE, &hKey);
// Public key
CryptExportKey(hKey, NULL, PUBLICKEYBLOB, 0, NULL, &dwPublicKeyLen);
pbPublicKey = (BYTE *)malloc(dwPublicKeyLen);
CryptExportKey(hKey, NULL, PUBLICKEYBLOB, 0, pbPublicKey, &dwPublicKeyLen);
hPublicKeyFile = CreateFile("public.key", GENERIC_WRITE, 0, NULL, CREATE_ALWAYS, FILE_ATTRIBUTE_NORMAL, NULL);
WriteFile(hPublicKeyFile, (LPCVOID)pbPublicKey, dwPublicKeyLen, &lpNumberOfBytesWritten, NULL);

// Private key
CryptExportKey(hKey, NULL, PRIVATEKEYBLOB, 0, NULL, &dwPrivateKeyLen);
pbPrivateKey = (BYTE *)malloc(dwPrivateKeyLen);
CryptExportKey(hKey, NULL, PRIVATEKEYBLOB, 0, pbPrivateKey, &dwPrivateKeyLen);
hPrivateKeyFile = CreateFile("private.key", GENERIC_WRITE, 0, NULL, CREATE_ALWAYS, FILE_ATTRIBUTE_NORMAL, NULL);
WriteFile(hPrivateKeyFile, (LPCVOID)pbPrivateKey, dwPrivateKeyLen, &lpNumberOfBytesWritten, NULL);

```

通过上述代码，我们生成一个.reg文件，导入注册表后，我们就可以强制Locky用我们设置的RSA公钥。

让我们仔细看看Locky是怎么加密文件的。当通过CryptAcquireContext()函数获得一个指向PROV_RSA_AES结构的CSP（密码库）句柄后，通过CryptImportKey()函数导入pubkey键值里包含有公钥的二进制数据，然后它把文件按`<id><16个随机字符>.locky`格式改名。它通过CryptGenRandom()函数来产生16个随机字符：

![p10](http://drops.javaweb.org/uploads/images/ff6dec350fc139b71505d532f9233827b43f367a.jpg)

```
(gdb) hexdump 0x009ef8a0 16
9d 86 d3 42 48 3a 45 04 1a cb 95 1c 77 90 8f 7c

```

这些字节将会被复制到由CryptImportKey()函数导入的BLOB_HEADER 结构的后面。

```
(gdb) hexdump 0x009ef784 0x1c
08 02 00 00 0e 66 00 00 10 00 00 00 9d 86 d3 42 
48 3a 45 04 1a cb 95 1c 77 90 8f 7c

```

*   type 0x08 = PLAINTEXTKEYBLOB
*   version 0x02
*   2 reserved bytes
*   ALG_ID = 0x660e = CALG_AES_128
*   key size = 0x10 bytes

这是一个AES-128 密钥，然后GetSetKeyParam()函数会被调用，对其下断，这里我们可以获得更多关于这个密钥是如何被使用的信息：

```
(gdb) x/w $esp+4
0x9ef830: 0x00000004

(gdb) x/w *(int*)($esp+4+4)
0x9ef858: 0x00000002

```

内存值4在第二个参数里，代表 KP_MODE ，可以允许自定义操作模式；内存值2在第三个参数里，代表CRYPT_MODE_ECB。CryptEncrypt()函数用RSA公钥来加密AES的密钥，获取到的结果如下：

```
(gdb) hexdump 0x9ef8a0 256
64 ab 20 75 75 56 ae f4 af 20 7f 38 81 d7 d6 56 |d. uuV... .8...V|
22 89 92 6e 30 e0 61 d2 24 f0 a1 d6 2a 20 7f 6c |"..n0.a.$...* .l|
e0 10 cc ab 26 62 33 66 71 8d 93 4c 04 61 8a 9a |....&b3fq..L.a..|
86 e7 f4 75 58 ae 8a 68 96 1f a8 69 15 aa 2f e7 |...uX..h...i../.|
8b cd ca 2e b0 7b e1 89 5f 3e 65 61 4c 0b 43 5e |.....{.._>eaL.C^|
60 3b 17 48 0e d2 08 80 bd 4d e2 38 5b 51 c9 82 |`;.H.....M.8[Q..|
26 bf 94 8a 45 40 82 62 1e 88 42 aa 35 2a 3e 58 |&...E@.b..B.5*>X|
d2 7d 03 4d cd d4 e6 3b 7d 44 e9 5f dc 4d 1c 4b |.}.M...;}D._.M.K|
27 a9 39 0c 74 ed 46 97 60 af 3a 97 3f 89 33 28 |'.9.t.F.`.:.?.3(|
bf 27 67 57 f8 c5 4e 03 72 45 60 88 03 e5 11 98 |.'gW..N.rE`.....|
6f 49 af 92 72 69 db ec b7 c7 51 9a 05 f2 34 e0 |oI..ri....Q...4.|
17 e4 1b 7e c5 97 ff 3d 42 5d ff a5 69 a4 58 f8 |...~...=B]..i.X.|
3b bd 9f 84 6e a5 c7 81 4e 0e aa 5d 40 ff 06 01 |;...n...N..]@...|
e9 ee 3c e5 0f b2 b4 80 af 56 c5 b8 25 af 11 2e |..<......V..%...|
22 82 c1 f1 93 50 b2 a4 76 98 46 2e db 6c 76 bb |"....P..v.F..lv.|
b5 1e 70 44 41 e2 15 31 f9 02 7d 92 7a e5 73 17 |..pDA..1..}.z.s.|

```

这个数据和.locky文件里的一致。

然后Locky产生一个0x800字节用0填充的缓冲空间，并且每行结尾用一字节计数：

```
(gdb) hexdump 0x00926598 0x800
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 |................|
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 01 |................|
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 02 |................|
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 03 |................|
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 04 |................|
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 05 |................|
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 06 |................|
[...]
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 7a |...............z|
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 7b |...............{|
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 7c |...............||
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 7d |...............}|
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 7e |...............~|
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 7f |................|

```

然后CryptEncrypt()函数会使用上面产生的密钥来对这些字节进行AES-128-ECB加密，然后得到如下数据（它是用来进行异或加密的K）：

```
0x00926598: fc d3 bb 90 ac 1e 1e 6e 76 88 09 52 66 76 71 fc 
0x009265a8: d5 e2 07 fd a5 0c 02 50 d0 83 4e 9b 95 1c 0b 60 
0x009265b8: 3f c5 49 e5 df b2 05 56 bd ce eb f6 0d 70 9f 62 
0x009265c8: 98 f1 e8 b7 e2 8e d8 97 7f a1 83 14 2b db 82 98 
0x009265d8: 5b 4a 94 f7 fb 60 81 cd bb c7 a2 33 60 b1 c0 c7 
0x009265e8: 1c c5 c7 40 af 7c ea 4b e2 74 b0 32 c2 37 5e fa 
0x009265f8: cf 40 69 9b 81 92 b8 f1 77 79 83 97 32 19 75 a6 
[...]
0x009267c8: 96 9a 1d bd 9b 03 33 2f d5 e7 a7 fc ac fc 09 c9 
0x009267d8: f6 bd c5 73 ce 9e ce bc fd e4 ef 6f 06 dd 7d 15 
0x009267e8: 7d 95 e6 18 78 87 46 ba 75 5e 58 2e f8 ba 5c 14 
0x009267f8: 3d a9 f3 d3 af ef 0b 39 00 ae 0c 32 2b fd 37 eb 
0x00926808: 3f 3a 68 11 b8 d1 ae e7 28 40 0a 20 33 31 8f 7e 
0x00926818: c3 8f 55 2a 5f b5 31 26 02 41 d7 e3 84 c5 79 9b 
[...]

```

第一个要加密的元素就是文件名。Locky先分配0x230字节空间，用0填充，

先把文件名复制进缓冲区，然后再与K进行异或加密。例如，如下我们观察前3字节：

```
(gdb) p/x $edx // 加密KEY
$3 = 0x90bbd3fc
(gdb) hexdump $edi 64 //XOR 加密前
2a a1 1b d4 6d 00 6f 00 6e 00 66 00 69 00 63 00 |*...m.o.n.f.i.c.|
68 00 69 00 65 00 72 00 2e 00 74 00 78 00 74 00 |h.i.e.r...t.x.t.|
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 |................|
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 |................|
(gdb) hexdump $edi 64 //XOR 加密后 #1
d6 72 a0 44 6d 00 6f 00 6e 00 66 00 69 00 63 00 |.r.Dm.o.n.f.i.c.|
68 00 69 00 65 00 72 00 2e 00 74 00 78 00 74 00 |h.i.e.r...t.x.t.|
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 |................|
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 |................|

```

再看4-7字节

```
(gdb) p/x $eax // key 
$4 = 0x6e1e1eac
(gdb) hexdump $edi 64 // XOR 加密后#2
d6 72 a0 44 c1 1e 71 6e 6e 00 66 00 69 00 63 00 |.r.D..qnn.f.i.c.|
68 00 69 00 65 00 72 00 2e 00 74 00 78 00 74 00 |h.i.e.r...t.x.t.|
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 |................|
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 |................|

```

以此类推，加密结果与加密后的.locky文件一致。

然后Locky以同样的方式来加密文件内容，但是KEY 从偏移0x230处（本场景中在0x009267c8处）开始。这不可能是一个随机的选择，因为如果Locky的开发者采用相同KEY来加密，那么文件内容解密将会是非常顺利成章的。实际上，保存文件名的0x230字节几乎全是0，对这些字节进行异或操作后，在.locky文件里可以看到结果，这样在不知道AES相应的密钥的情况下，我们可以分析出相当大比例的KEY。

文件内容加密如下：

```
(gdb) p/x $edx
$5 = 0xbd1d9a96 // key stream from offset 0x230 (0x009267c8)
(gdb) hexdump $edi 64 //before XOR
63 65 63 69 20 65 73 74 20 75 6e 20 73 65 63 72 |ceci est un secr|
65 74 00 00 00 00 00 00 00 00 00 00 00 00 00 00 |et..............|
(gdb) hexdump $edi 64 //after XOR #1
f5 ff 7e d4 20 65 73 74 20 75 6e 20 73 65 63 72 |..~. est un secr|
65 74 00 00 00 00 00 00 00 00 00 00 00 00 00 00 |et..............|

```

这里和.locky文件开头部分一致。

如果文件大小大于初始化时候的缓冲区，Locky将通过CryptEncrypt()函数以同样的布局加密其余的缓冲空间，并增加计数：

```
(gdb) hexdump *(int*)($esp+4+4+4+4) 128
0x009255e0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 80
0x009255f0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 81
0x00925600: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 82
[...]
(gdb) hexdump *(int*)($esp+4+4+4+4) 128
0x009255e0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 01 00
0x009255f0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 01 01
0x00925600: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 01 02
[...]

```

最终一个.Locky文件布局如下：

![p11](http://drops.javaweb.org/uploads/images/50286f59caa43d7e4a9f990847a2da5b3bc0b4e0.jpg)

可以按以下步骤来解密：

1.  用我们的私钥来解密RSA加密的缓冲区，获得16字节数据
2.  用这16字节数据作为AES-128-ECB密钥来加密一个增量常数缓冲区（用0填充，结尾计数的数据）
3.  用上面结果的前0x230字节作为KEY来对文件名进行异或解密
4.  把从0x230处开始的数据作为KEY 来对文件内容进行异或解密

第一步用C代码实现如下：

```
// Importing the RSA private key
hPrivateKeyFile = CreateFile("private.key", GENERIC_READ, FILE_SHARE_READ, NULL, OPEN_EXISTING, FILE_FLAG_SEQUENTIAL_SCAN, NULL);
dwPrivateKeyLen = GetFileSize(hPrivateKeyFile, NULL);
pbPrivateKey = (BYTE *)malloc(dwPrivateKeyLen);
ReadFile(hPrivateKeyFile, pbPrivateKey, dwPrivateKeyLen, &dwPrivateKeyLen, NULL);
CryptImportKey(hCryptProv, pbPrivateKey, dwPrivateKeyLen, 0, 0, &hKey);

// Reading the RSA buffer
hEncryptedFile = CreateFile("encrypted.rsa", GENERIC_READ, FILE_SHARE_READ, NULL, OPEN_EXISTING, FILE_FLAG_SEQUENTIAL_SCAN, NULL);
dwEncryptedDataLen = GetFileSize(hEncryptedFile, NULL);
pbEncryptedFile = (BYTE *)malloc(dwEncryptedDataLen);
ReadFile(hEncryptedFile, pbEncryptedFile, dwEncryptedDataLen, &dwEncryptedDataLen, NULL);

// Decrypting the AES key
CryptDecrypt(hKey, NULL, TRUE, 0, pbEncryptedFile, &dwEncryptedDataLen);
hClearFile = CreateFile("aeskey.raw", GENERIC_WRITE, 0, NULL, CREATE_ALWAYS, FILE_ATTRIBUTE_NORMAL, NULL);
WriteFile(hClearFile, (LPCVOID)pbEncryptedFile, dwEncryptedDataLen, &lpNumberOfBytesWritten, NULL);

```

获取到AES密钥：

```
$ xxd aeskey.raw
9d86 d342 483a 4504 1acb 951c 7790 8f7c

```

通过上面的密钥，用Python代码实现解密文件名和文件内容如下：

```
#! /usr/bin/env python
from Crypto.Cipher import AES
print "UnLocky - Locky decryption tool, CERT-LEXSI 2016
key = "9d86d342483a45041acb951c77908f7c".decode("hex")
# NB: small files only here
counter = ""
for i in range(0x80):
  counter += "\x00"*15 + chr(i)
keystream = AES.new(key, AES.MODE_ECB).encrypt(counter)
data = open("1D9076E6FD853AB6C931AFE2B33C3AF9.locky").read()
enc_size = len(data) - 0x230 - 0x100 - 0x14
enc_filename = data[-0x230:]
enc_content  = data[:enc_size]
clear_filename = ""
for i in range(0x230):
  clear_filename += chr(ord(enc_filename[i]) ^ ord(keystream[i]))
print "[+] File name:"
print clear_filename
clear_content = ""
for i in range(enc_size):
  clear_content += chr(ord(enc_content[i]) ^ ord(keystream[0x230+i]))
print "[+] Content:"
print clear_content

```

我们看看它是否能解密文件：

```
$ ./unlocky.py
UnLocky - Locky decryption tool, CERT-LEXSI 2016
[+] File name:
monfichier.txt // "myfile.txt"

[+] Content:
ceci est un secret // "this is a secret"

```

0x05 总结
=====

Locky已经肆虐很久了，这里提供了一些简单的方法，在不需要任何反病毒或者安全工具的前提下，可以防止用户系统文件被加密，并保护系统为原样。比如，可以使用本文提到的4个“疫苗”中的一个“疫苗”：强制替换用来加密AES KEY的RSA公钥。

通过上面的分析，Locky并不是直接通过AES-128-ECB来加密文件，而是通过AES-128-ECB来加密一个增量缓冲区，并把结果分为两部分，分别作为异或加密文件名和文件内容的KEY，看上去和CTR非常类似。