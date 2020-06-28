# 2014年澳大利亚信息安全挑战 CySCA CTF 官方write up Crypto篇

**from:https://www.cyberchallenge.com.au/CySCA2014_Crypto.pdf**

0x00 背景
-------

* * *

Fortcerts开发了不少涉及加密算法的程序并已经自行测试了这些程序，现在他们想要了解其他人对于这些加密程序安全性的看法。

0x01 标准银河字母（Standard Galactic Alphabet）
---------------------------------------

* * *

**Question**请对Fortcerts自定义加密算法的Slightly Secure Shell程序进行白盒测试。找出漏洞并证明其可以利用来获取机密信息。服务器运行在172.16.1.20:12433

**Source**

```
a580fd052a2f1ef9a0753ee36ad6bd51-crypt01.py
=== snip... ===
def execute_command(command,plain,coded):
    print "Running command: bash %s" % command
    proc = subprocess.Popen(("/bin/bash","-c",command),stdout=subprocess.PIPE,stderr=subprocess.PIPE)
    proc.wait()
    stdoutdata = proc.stdout.read()
    stdoutdata += proc.stderr.read()
    output = ""
    for letter in stdoutdata:
        if letter in plain:
            output += coded[plain language=".find(letter)"][/plain]
        else:
            output += letter

    return output

def handle_client(conn,addr):
    plain = "`1234567890-=~!@#$%^&*()_+[]\{}|;':\",./<>?abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ "
    conn.send("You have connected to the Slightly Secure Shell server of Fortress Certifications.\n")
    coded = shuffle_plain(plain)
    command = ""
    conn.send("#>")

    while 1:
        data = conn.recv(1)
        if not data: break
        if data == "\x7f" or data == "\x08":
            if len(command) > 0:
                command = command[:-1]
            continue
        if data == "\n" or data =="\r":
            if len(command) == 0: continue
            conn.send("Running command: '%s'\n" % command)
            cmd_stdout = execute_command(command,plain,coded)
            conn.sendall(cmd_stdout+"\n")
            command = ""
            conn.sendall("Key reset\n")
            coded = shuffle_plain(plain)
            conn.sendall("#>")
        else:
            if data not in plain:
                continue
            command += data
        conn.sendall(data)
    conn.close()
=== snip... ===

```

**Designed Solution**选手可以在自定义命令之前发送一个echo命令来获得加密密钥并对输出值进行破译，然后使用多个命令来查找包含key的文件，并查看其内容。**Write Up**首先阅读系统提供的源码，明确程序读取用户的输入值，直到它获到一个换行符或回车符，然后通过bash来执行命令，并使用单码代换对命令的执行结果加密后将输出返回给用户。在此之后，密钥被重置。 我们可以通过管道命令来在单个密钥的使用过程中执行多个命令，基于此来恢复密钥和对输出进行解密。 下面连接到服务器来发送一些命令以验证我们的假设：可以执行多个命令并且密钥仅在命令都执行完后被重置。

```
#>nc 172.16.1.20 12433
You have connected to the Slightly Secure Shell server of Fortress Certifications.
#>echo AAAA
echo AAAARunning command: 'echo AAAA'
6666
Key reset
#>echo AAAA;echo AAAA
echo AAAA;echo AAAARunning command: 'echo AAAA;echo AAAA'
zzzz
zzzz
Key reset
#>echo AAAA;echo BBBB;echo ABAB
echo AAAA;echo BBBB; echo ABABRunning command: 'echo AAAA;echo BBBB; echo ABAB'
8888
5555
8585
Key reset

```

可以看出A和B总是被相同字符替换，因此可以确定这是一个单码代换加密，同时可以得知密钥是在所有命令的输出结果显示之后再更新。 我们通过两个bash命令来执行任意命令，第一个echo明文字符串，第二个执行自定义命令。这样输出结果的开始将是密文字符表，剩下的内容就是第二个命令的输出。

```
#>echo abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ; echo TestString
echo abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ; echo TestStringRunning command: 'echo abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ; echo estString'
_u;pSw"y|r^QFJ <Lk\+{KnEX$#%=UDb/6[HlBOaPRA54(-!'xq: <<< Ciphertext alphabet
(S\+4+k|J" <<< Test string

```

由此可以使用密文字符表来解密测试字符串。 明文字符表`abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ`密文字符表:`_u;pSw"y|r^QFJ <Lk\+{KnEX$#%=UDb/6[HlBOaPRA54(-!'xq:`密文测试字符串:`(S\+4+k|J"`明文测试字符串:`TestString`这种方法看来行之有效，下面使用ls命令来定位flag，并用cat命令来显示它。

```
#>echo abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ1234567890.;ls -1
echo abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ1234567890.;ls -1Running command: 'echo abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ1234567890.;ls -1'
 qaF)-:"XzJ^#Ehp2H_*]iDMUP\@8`rNdIm6AW3&5e%0}[Rk/VYS>{tl<;KfG4n <<Key
qXE       << bin
F)i       << dev
)*a       << etc
-^ :n*M*  << flag.txt
^Xq       << lib
Key reset
#>echo abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ1234567890.; cat flag.txt
echo abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ1234567890.; cat flag.txtRunning command: 'echo
abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ1234567890.; cat flag.txt'
q^0sE`"{G3KSo& @):nUZ=;+[BM?Lx*]CwIdmOg#2,VX-!Plav'e\9N4/_F(Wf7
Lq=Gq:?qKEs-{qoEW__ << CaviarBakedShame966

```

执行这些命令并对响应解码后得到flag：CaviarBakedShame966  

0x02 压缩会话（Compression Session）
------------------------------

* * *

**Question**请对Fortcerts的高安全性密钥生成服务器进行白盒测试。找出能造成秘密数据泄漏的漏洞。服务器运行在172.16.1.20:9999**Source**5a92cb8141992b7b71497a3bc920c7a5-crypt02.py

```
=== snip... ===
def compress(d):
    c = zlib.compressobj()
    return(c.compress(d) + c.flush(zlib.Z_SYNC_FLUSH)).encode("hex")

def encrypt(aes,d):
    return aes.encrypt(d).encode("hex")

def handle_client(conn, addr):
    AESKey = os.urandom(32)
    ctr = Counter.new(128)

    aes = AES.new(AESKey, AES.MODE_CTR, counter=ctr)

    BANNER = "\t\tWelcome to the Keygen server.\n\t\t"
    BANNER += "="*30
    BANNER += "\n[+]All access is monitored and any unauthorised access will be treated as CRIME\n"
    conn.send(BANNER)

    SECRET = 'Key:' + helpers.load_flag()
    data = conn.recv(2048).strip()

    while len(data):
        data = compress(SECRET + data)
        data = encrypt(aes, data)
        conn.send(data)
        data = conn.recv(2048).strip()
    conn.close()
=== snip... ===

```

**Designed Solution**在这里，选手使用被认为是犯罪的攻击方式。由于在使用流密码加密有关被泄漏的私钥之前对数据进行了压缩，选手可以自写利用泄露信息的脚本**Write Up**首先分析系统提供的源码，服务器把我们的输入值，附加到密钥后面并压缩，然后加密，最后返回一个经过加密的16进制编码的字符串。它使用了AES的CTR模式，说明这里的AES是流密码方式加密而不是分组加密。 使用netcat连接到远程服务。标识表明这是一个密钥服务器，并警告说任何未经授权的访问都将被视为是犯罪，这里的CRIME特意用了大写。服务器既不提供任何提示来与用户进行交互，也没有提供help功能。

```
#>nc 172.16.1.20 9999
        Welcome to the Keygen server.
        ==============================
[+]All access is monitored and any unauthorised access will be treated as CRIME

```

为了验证在查看源码片段时所得出有关代码功能的结论是否正确，下面发送一些输入到服务器。从这里我们可以看出，无论CTR模式中使用的计数器是怎么增加的，每次提交之间的密钥不会被重置，所以加密的数据不同。同时也能看出密文长度变化很小，这也表明使用的是流密码而不是分组密码。

```
AAAA
0a7a99948f447f19b22d7a9a74e25d591b8ff864fa0d3b80804f18b2798219cdb4429e39c8bd6594bc
50f640432fa7db9368db69ce769d2fe4ca30a1bc728c4d0ccdf701b8ae79000db82220
AAAA
ae387373a2131e25e26793aae4d3d1e61a74680c0c654dcef76ec0c27667bf861bd87d272f8d70e03f
b55a515be355f0030bae31beef06903ee6da654a96dbe41d134bea66d4993fbb414512
AAAA
7ff3513568e314e954bb872b604d3d9518fb64382fc6129d80465f27f99ab53a0300aea122627cc2fa
2d5d06600626079c20406f51fe9dcd9a680a9c9041c421308ddc322aac3b2f7dc9f253
AAAABBBB
b1217dacfc0e7927f3519baedb290a14b791b3b97c821158e84e956d071c887f97afd85213476712ff
da16fab035d54a72ee8f8608f94557e7e8866480aa5252eb1ed533c8d006c647824fc95123782b
AAAAAAAA
a3c7b8ae0f2628e126914284f62db7f60122daa23fc39cfc7f3cbd8999c2f979a3f8a214ebd225c48e
5ec5b303a672a45450ef13c00c110758e14d3f2981b95356c537f678357340be941877

```

注意到8个A加密的输出长度和4个A一样。与通常的不一样的是，8个A一般会被压缩成和4个A一样的长度。被压缩的数据通过流密码加密，因此加密数据的长度等于被压缩数据的长度。我们可以使用压缩数据的长度来获取数据被压缩的有关信息，这是一个有关CRIME攻击的类似准则。你现在明白CRIME为什么要大写了吗？ 再次审查源码，我们明确了数据压缩的方式。假设flag是“TestFlag”，输入ABCD到服务器，压缩数据将是“Key:TestFlagABCD”。这个数据不会被压缩，我们也得到了一个较长的响应字符串。但是，如果输入“Key:”，将要压缩的数据就是“Key:TestFlagKey:”。当算法压缩数据时，它将得到两个“Key:”，然后有效地压缩并返回一个较短的响应。 我们发送“Key:”来验证推断。每次发送后都得到了不同的响应内容，但是响应的长度没有变化。继续发送“Key:A”，响应的长度增加了；发送“Key:B”和“Key:C”都得到了与“Key:A”相同的响应长度。当发送“Key:D”给服务器后，响应变短了，长度与发送“Key:”的响应相同。

```
Key:
8bec8291d51787058bfe80c1303c3024f60b84812b4ec6a0d2a818dc74fc340bc45d99f89e29fbd7ee
c47f9091ebe518bd85b6e3d5c583719ee438ae4d597772afc9506aec2214865d9750ca
Key:A
964c86d2d100faaee0a920dee7716a29493cd3e11e7a33e1b0cfbe866e01f53208cab473a3eebc8607
dbf2ab25d197b6f6684c70f9a51b15debc4f282edf048cfb57c47b94e7b147b0f08fa4f27c
Key:B
85deee163f4e0982f67688fb0cdb7c735ccbe8f276796aadfdb7585a0528e5af9e107f5901637fdbc2
9c4622d821a91f4a4da9302115f601aeb97a120ffb86350797ed1e801a7e8c9f00f97e28dd
Key:C
af147c047eb3d0e84472cc95670919617ae4d10623b990ced591f074bae9d85f58bce66491ccc2a9c5
6ff9e438e78ce02f9c4d61086a5f52f16806a2eddc6d7daca5498c15ac854018a53491ab8d
Key:D
dd008730d0524e3af6332e4c69e69d7428968b7b21d32ebef9e35d8b908026b69023b09e2a9f978721
017b254b0b1b08ec7cd460a681e404eedb027c88e843f288ba5a15d276208aa86792b0

```

从这里我们得知flag可能是以大写的D开头。知道了这个缺陷，就可以使用Python脚本来连接服务器，然后暴力破解flag的每一个字符。

```
import socket
import string
known_part_of_the_key = "Key:"
def Checker(s, bs):
    global known_part_of_the_key
    strings = string.letters + string.digits + "%/+=:"
    while 1:
        for ch in strings:
            s.send(known_part_of_the_key + ch)
            resp = s.recv(2048)
            if len(resp) == bs:
                print known_part_of_the_key
                known_part_of_the_key = known_part_of_the_key + ch

host = "172.16.1.20"
port = 9999
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.connect((host, port))
banner = sock.recv(1024) #Receive the banner
print banner
sock.send(known_part_of_the_key)
resp = sock.recv(2048)
base_size = len(resp)
Checker(sock, base_size)

```

运行脚本之后就得到flag：DrizzleVerandaFinger576

```
#>python soln.py
        Welcome to the Keygen server.
        ==============================
[+]All access is monitored and any unauthorised access will be treated as CRIME
Key:
Key:D
Key:Dr
Key:Dri
Key:Driz
****SNIP****
Key:DrizzleVerandaFinger576
****SNIP****
 

```

0x03 杂项（Chop Suey）
------------------

* * *

**Question**

Fortcerts的高管均表示反对使用白盒测试的方法，他们说真正的攻击者不可能获得源码的访问权。请对Fortcerts的一个非常安全的加密服务进行黑盒测试。找出服务中能够恢复加密数据的漏洞。这个非常安全的加密服务运行在172.16.1.20:1337

**Designed Solution**

选手需要分析服务器以确定它是否使用了基于流密码的算法。一旦他们是这样做的，需要确定是否使用了能产生重复密钥流的IV。选手可以建立一个密钥流的数据库，然后通过重复的密钥流来对flag进行解密。

**Write Up**

由于开始时没有任何文件提供，我们直接连接到加密服务看是否能明确它的功能。首先是一句“Key Reset!”的消息，然后从标识处得知程序使用了非常安全的加密。另外还列出了两个命令并告诉我们输出的格式。

```
#>nc 172.16.1.20 1337
Key Reset!
    Welcome to the FortCerts Certified Data Encryption Service
            This program uses very secure encryption
Commands:
E - Encrypt specified data
D - Dump service stored data
Output is in format <IV>:<Encrypted Data>

```

我们输入这两个命令来确定其功能。输入E，得到一个Invalid use of encrypt，这说明该命令需要附带参数。使用D命令时得到的字符串表明加密结果与D命令下面的简介信息相符，最后得到一个“Key Reset!”的消息。

```
E
Invalid use of encrypt. Usage E,<valuetoencrypt>
D
226e:01ef3044bac49aed75821296a10c060f0d0225386936
Key Reset!

```

依次输入a到z，A到Z看是否存在未公开的命令。但是，除了E和D之外都显示无效的命令，然后连接断开。 我们尝试给D和E命令附加一些参数来查看其输出是否有变化。 结果表明：无论给D命令什么参数，尽管字符串发生了变化，但是它的输出格式不变。加密数据的长度以及IV的长度总是恒定的，同时最后总有一个“Key Reset!”消息。

```
D
035b:3e4e9892b1ab053426dfa7168a0cc9827acf33d3480b
Key Reset!
D,
171f:73674b54f57f52874aed3027ece58d0f97780c88223e
Key Reset!
D,,,,
8e01:ba6b8ce2fa0e4d9f6b1888889df0110f4015b10a60fe
Key Reset!
D,AAAAA
a6ec:8f040a25951bdb727a974b7087c9cf733e44b427e2dd
Key Reset!
D,Test
b20f:0c823c92d80102913af7353254717d60b68f5e3d2a0e
Key Reset!
D
d4c0:b99815314cf8d5bec3d92be1204e66a044fc7050c48a
Key Reset!
D,BBBBBB
1026:bd9c781a6cd7d766040c0d539a13ebb9a590b1cebac1
Key Reset!

```

对于E命令，服务要求使用的正确命令格式为E,，任何其它的命令都显示Invalid use of encrypt，同时返回一个用法。当向E命令提供正确参数值时，输出的格式和D命令一致，所以D命令很有可能只是执行E命令存储在服务中的数据。

```
E.
Invalid use of encrypt. Usage E,<valuetoencrypt>
E,,,
Invalid use of encrypt. Usage E,<valuetoencrypt>
E,,,,,,
Invalid use of encrypt. Usage E,<valuetoencrypt>
E..
Invalid use of encrypt. Usage E,<valuetoencrypt>
E,
Invalid use of encrypt. Usage E,<valuetoencrypt>
E,A
fb51:9e
E,AAAA
78c9:f469f095
E,AAAA
9d16:001b2784
E,AAAAAAAA
dbfb:87a24590fd8ac117
E,TEST
1038:8283d469
E,TEST
7e3d:0c057546
E,1234567890
65e4:fdba65422ede91d82f10

```

当使用E命令时，输出字符都是16进制范围内的0-9和a-f，因此字符串很可能是16进制编码的字符。另外没有显示“Key Reset”的消息，这也表明可能每次使用E功能时密钥将不会重置。 注意到每个IV都是不同的而且加密后的数据长度总是两倍于输入的数据。由于加密数据的长度是随着输入字符长度的增加而增加，我们确信这是一个流密码而不是分组密码。 在流密码和流密码攻击中，我们知道WEP不安全的根源在于其IV仅有24位长。但是这里的IV仅仅只有2个16进制编码字符即16位长。因此有可能得到重复使用的密钥和IV，这将导致密钥流的重用，同时允许我们能够解密D命令返回的数据。我们尝试一下。 对5个A加密500次，然后对结果进行排序并计数，最终有3个IV的碰撞而且每个碰撞的响应数据是相同的。

```
#>for i in {0..500}; do echo "E,AAAAA" >> test.txt; done; echo "Q" >> test.txt
#The Q is so we get disconnected

#>tail test.txt
E,AAAAA
E,AAAAA
E,AAAAA
E,AAAAA
E,AAAAA
Q

#>cat test.txt | nc 172.16.1.20 1337 | sort | uniq -c | grep " [2-9] "
2 63b2:67c7b06d66
2 a315:beece8713b
2 f7db:08db5207b7

```

现在我们确认该服务器有IV重用攻击的漏洞，只需要收集一些结果以便能够恢复密钥流并且对服务中存储的数据进行解密。 我们需要明确的是，目的不是恢复密钥而是密钥流，因此需要从D命令的输出中得到尽可能多的响应字节。查看D命令的输出值，我们知道至少需要22字节的密钥流。

```
D
3868:3bff03fff1fe385eb6d68c9e84f6771564968d7b017

```

我们写一个脚本来加密25个A，并将结果存下来。当拥有10000个不同的IV值时，我们将能转储flag并尝试进行解密。

```
import socket
clisock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
clisock.connect(("172.16.1.20",1337))
respdict = dict()
#Recv Key Reset message and Banner
print clisock.recv(1024)
print clisock.recv(1024)

while len(respdict) < 10000:
    #Send encrypt
    clisock.send("E,"+"A"*25+"\n")
    resp = clisock.recv(1024).strip()
    parts = resp.split(":")
    respdict[parts[0]] = parts[1]
    #print parts
    if len(respdict) % 200 == 0:
        print str(len(respdict))+"/10000"

#Dump the service secret
clisock.send("D\n")
secretdata = clisock.recv(1024).strip()
parts = secretdata.split(":")

#Look for the IV in the dict
if parts[0] in respdict:
    print "Found secret iv in dict. Dumping responses"
    print " A's response:",respdict[parts[0]]
    print "Secret response:",parts[1]
else:
    print "Didn't find secret iv in dict. Try again"

```

对服务器运行脚本，等待其对IV值的收集。最后收到一条消息表示dump命令的输出值中有一个IV与存放在字典中的某个IV一致。

```
#>python solv.py
Connecting
Key Reset!
Welcome to the FortCerts Certified Data Encryption Service
This program uses very secure encryption
Commands:
E Encrypt specified data
D Dump service stored data
Output is in format <IV>:<Encrypted Data>

200/10000
**** SNIP (about 7 minutes) ****
9600/10000
9800/10000
10000/10000
Found secret iv in dict. Dumping responses
A's response: 32fe79d33e7055190ec47c1778c1029946c9fc47398c613936
Secret response: 35cd51f711555a373bec5e337be52fbe66fbc9334af8

```

![enter image description here](http://drops.javaweb.org/uploads/images/5fa0f3dc4b4e5541803f8997ef5d237d86a03934.jpg)

复现图

现在已有带重用密钥流的响应，首先恢复密钥流。流密码生成密钥流，然后逐字节与明文进行异或得到密文。已知明文A的响应，只需要将响应数据与明文进行异或就能得到实际的密钥流。

```
A's response: 32fe79d33e7055190ec47c1778c1029946c9fc47398c613936
A's plaintest: 41414141414141414141414141414141414141414141414141
Shared keystream: 73bf38927f3114584f853d56398043d80788bd0678cd207877

```

由已恢复的密钥流很容易就能解密dump命令的响应，并得到flag：FriendNoticeBelfast525

```
Dump response: 35cd51f711555a373bec5e337be52fbe66fbc9334af8
Shared keystream: 73bf38927f3114584f853d56398043d80788bd0678cd207877
Dump plaintext: 467269656e644e6f7469636542656c66617374353235
Plaintext(ASCII): FriendNoticeBelfast525

```