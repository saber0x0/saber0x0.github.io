# Linux快速入门

## 常用基础命令

```
ls                  用来显示目标列表

cd [path]           用来切换工作目录

pwd                 以绝对路径的方式显示用户当前工作目录

man [command]       查看Linux中的指令帮助、配置文件帮助和编程帮助等信息

apropos [whatever]  在一些特定的包含系统命令的简短描述的数据库文件里查找关键字

echo [string]       打印一行文本，参数“-e”可激活转义字符

cat [file]          连接文件并打印到标准输出设备上

less [file]         允许用户向前或向后浏览文字档案的内容

mv [file1] [file2]  用来对文件或目录重新命名，或者将文件从一个目录移到另一个目录中

cp [file1] [file2]  用来将一个或多个源文件或者目录复制到指定的目的文件或目录

rm [file]           可以删除一个目录中的一个或多个文件或目录，也可以将某个目录及其下属的所有文件及其子目录均删除掉

ps                  用于报告当前系统的进程状态

top                 实时查看系统的整体运行情况

kill                杀死一个进程

ifconfig            查看或设置网络设备

ping                查看网络上的主机是否工作

netstat             显示网络连接、路由表和网络接口信息

nc(netcat)          建立 TCP 和 UDP 连接并监听

su                  切换当前用户身份到其他用户身份

touch [file]        创建新的空文件

mkdir [dir]         创建目录

chmod               变更文件或目录的权限

chown               变更某个文件或目录的所有者和所属组

nano / vim / emacs  字符终端的文本编辑器

exit                退出 shell
管道命令符 "|"       将一个命令的标准输出作为另一个命令的标准输入
```

使用变量：

```
var=value         给变量var赋值value

$var, ${var}      取变量的值

`cmd`, $(cmd)     代换标准输出

'string'          非替换字符串

"string"          可替换字符串        
$ var="test";
$ echo $var
test
$ echo 'This is a $var';
This is a $var
$ echo "This is a $var";
This is a test

$ echo `date`;
2017年 11月 06日 星期一 14:40:07 CST
$ $(bash)

$ echo $0
/bin/bash
$ $($0)
```



## Bash 快捷键

```
Up(Down)          上（下）一条指令

Ctrl + c          终止当前进程

Ctrl + z          挂起当前进程，使用“fg”可唤醒

Ctrl + d          删除光标处的字符

Ctrl + l          清屏

Ctrl + a          移动到命令行首

Ctrl + e          移动到命令行尾

Ctrl + b          按单词后移（向左）

Ctrl + f          按单词前移（向右）

Ctrl + Shift + c  复制

Ctrl + Shift + v  粘贴
```

## 根目录结构



```
$ uname -a
Linux manjaro 4.11.5-1-ARCH #1 SMP PREEMPT Wed Jun 14 16:19:27 CEST 2017 x86_64 GNU/Linux
$ ls -al /
drwxr-xr-x  17 root root  4096 Jun 28 20:17 .
drwxr-xr-x  17 root root  4096 Jun 28 20:17 ..
lrwxrwxrwx   1 root root     7 Jun 21 22:44 bin -> usr/bin
drwxr-xr-x   4 root root  4096 Aug 10 22:50 boot
drwxr-xr-x  20 root root  3140 Aug 11 11:43 dev
drwxr-xr-x 101 root root  4096 Aug 14 13:54 etc
drwxr-xr-x   3 root root  4096 Apr  8 19:59 home
lrwxrwxrwx   1 root root     7 Jun 21 22:44 lib -> usr/lib
lrwxrwxrwx   1 root root     7 Jun 21 22:44 lib64 -> usr/lib
drwx------   2 root root 16384 Apr  8 19:55 lost+found
drwxr-xr-x   2 root root  4096 Oct  1  2015 mnt
drwxr-xr-x  15 root root  4096 Jul 15 20:10 opt
dr-xr-xr-x 267 root root     0 Aug  3 09:41 proc
drwxr-x---   9 root root  4096 Jul 22 22:59 root
drwxr-xr-x  26 root root   660 Aug 14 21:08 run
lrwxrwxrwx   1 root root     7 Jun 21 22:44 sbin -> usr/bin
drwxr-xr-x   4 root root  4096 May 28 22:07 srv
dr-xr-xr-x  13 root root     0 Aug  3 09:41 sys
drwxrwxrwt  36 root root  1060 Aug 14 21:27 tmp
drwxr-xr-x  11 root root  4096 Aug 14 13:54 usr
drwxr-xr-x  12 root root  4096 Jun 28 20:17 var
```

由于不同的发行版会有略微的不同，我们这里使用的是基于 Arch 的发行版 Manjaro，以上就是根目录下的内容，我们介绍几个重要的目录： - `/bin`、`/sbin`：链接到 `/usr/bin`，存放 Linux 一些核心的二进制文件，其包含的命令可在 shell 上运行。 - `/boot`：操作系统启动时要用到的程序。 - `/dev`：包含了所有 Linux 系统中使用的外部设备。需要注意的是这里并不是存放外部设备的驱动程序，而是一个访问这些设备的端口。 - `/etc`：存放系统管理时要用到的各种配置文件和子目录。 - `/etc/rc.d`：存放 Linux 启动和关闭时要用到的脚本。 - `/home`：普通用户的主目录。 - `/lib`、`/lib64`：链接到 `/usr/lib`，存放系统及软件需要的动态链接共享库。 - `/mnt`：这个目录让用户可以临时挂载其他的文件系统。 - `/proc`：虚拟的目录，是系统内存的映射。可直接访问这个目录来获取系统信息。 - `/root`：系统管理员的主目录。 - `/srv`：存放一些服务启动之后需要提取的数据。 - `/sys`：该目录下安装了一个文件系统 sysfs。该文件系统是内核设备树的一个直观反映。当一个内核对象被创建时，对应的文件和目录也在内核对象子系统中被创建。 - `/tmp`：公用的临时文件存放目录。 - `/usr`：应用程序和文件几乎都在这个目录下。 - `/usr/src`：内核源代码的存放目录。 - `/var`：存放了很多服务的日志信息。



## 进程管理

- top
- 可以实时动态地查看系统的整体运行情况。
- ps
- 用于报告当前系统的进程状态。可以搭配 kill 指令随时中断、删除不必要的程序。
- 查看某进程的状态：`$ ps -aux | grep [file]`，其中返回内容最左边的数字为进程号（PID）。
- kill
- 用来删除执行中的程序或工作。
- 删除进程某 PID 指定的进程：`$ kill [PID]`

## UID 和 GID

Linux 是一个支持多用户的操作系统，每个用户都有 User ID(UID) 和 Group ID(GID)，UID  是对一个用户的单一身份标识，而 GID 则对应多个 UID。知道某个用户的 UID 和 GID 是非常有用的，一些程序可能就需要 UID/GID 来运行。可以使用 `id` 命令来查看：

```
$ id root
uid=0(root) gid=0(root) groups=0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel),19(log)
$ id firmy
uid=1000(firmy) gid=1000(firmy) groups=1000(firmy),3(sys),7(lp),10(wheel),90(network),91(video),93(optical),95(storage),96(scanner),98(power),56(bumblebee)
```

UID 为 0 的 root 用户类似于系统管理员，它具有系统的完全访问权。我自己新建的用户 firmy，其 UID 为 1000，是一个普通用户。GID 的关系存储在 `/etc/group` 文件中：

```
$ cat /etc/group
root:x:0:root
bin:x:1:root,bin,daemon
daemon:x:2:root,bin,daemon
sys:x:3:root,bin,firmy
......
```

所有用户的信息（除了密码）都保存在 `/etc/passwd` 文件中，而为了安全起见，加密过的用户密码保存在 `/etc/shadow` 文件中，此文件只有 root 权限可以访问。

```
$ sudo cat /etc/shadow
root:$6$root$wvK.pRXFEH80GYkpiu1tEWYMOueo4tZtq7mYnldiyJBZDMe.mKwt.WIJnehb4bhZchL/93Oe1ok9UwxYf79yR1:17264::::::
firmy:$6$firmy$dhGT.WP91lnpG5/10GfGdj5L1fFVSoYlxwYHQn.llc5eKOvr7J8nqqGdVFKykMUSDNxix5Vh8zbXIapt0oPd8.:17264:0:99999:7:::
```

由于普通用户的权限比较低，这里使用 `sudo` 命令可以让普通用户以 root 用户的身份运行某一命令。使用 `su` 命令则可以切换到一个不同的用户：

```
$ whoami
firmy
$ su root
# whoami
root
```

`whoami` 用于打印当前有效的用户名称，shell 中普通用户以 `$` 开头，root 用户以 `#` 开头。在输入密码后，我们已经从 firmy 用户转换到 root 用户了。



## 权限设置

在Linux 中，文件或目录权限的控制分别以读取、写入、执行 3 种一般权限来区分，另有 3 种特殊权限可供运用。

使用 `ls -l [file]` 来查看某文件或目录的信息：

```
$ ls -l /
lrwxrwxrwx   1 root root     7 Jun 21 22:44 bin -> usr/bin
drwxr-xr-x   4 root root  4096 Jul 28 08:48 boot
-rw-r--r--   1 root root 18561 Apr  2 22:48 desktopfs-pkgs.txt
```

第一栏从第二个字母开始就是权限字符串，权限表示三个为一组，依次是所有者权限、组权限、其他人权限。每组的顺序均为 `rwx`，如果有相应权限，则表示成相应字母，如果不具有相应权限，则用 `-` 表示。 - `r`：读取权限，数字代号为 “4” - `w`：写入权限，数字代号为 “2” - `x`：执行或切换权限，数字代号为 “1”



通过第一栏的第一个字母可知，第一行是一个链接文件 （`l`），第二行是个目录（`d`），第三行是个普通文件（`-`）。

用户可以使用 `chmod` 指令去变更文件与目录的权限。权限范围被指定为所有者（`u`）、所属组（`g`）、其他人（`o`）和所有人（`a`）。 - -R：递归处理，将指令目录下的所有文件及子目录一并处理； - <权限范围>+<权限设置>：开启权限范围的文件或目录的该选项权限设置 - `$ chmod a+r [file]`：赋予所有用户读取权限 - <权限范围>-<权限设置>：关闭权限范围的文件或目录的该选项权限设置 - `$ chmod u-w [file]`：取消所有者写入权限 - <权限范围>=<权限设置>：指定权限范围的文件或目录的该选项权限设置； - `$ chmod g=x [file]`：指定组权限为可执行 - `$ chmod o=rwx [file]`：制定其他人权限为可读、可写和可执行

![img](http://study.ctfcaict.com/media/course/html/3/site/linux/pic/1.3_file.png)

## 字节序

目前计算机中采用两种字节存储机制：大端（Big-endian）和小端（Little-endian）。

> MSB (Most Significan Bit/Byte)：最重要的位或最重要的字节。
>
> LSB (Least Significan Bit/Byte)：最不重要的位或最不重要的字节。

Big-endian 规定 MSB 在存储时放在低地址，在传输时放在流的开始；LSB  存储时放在高地址，在传输时放在流的末尾。Little-endian 则相反。常见的 Intel 处理器使用 Little-endian，而  PowerPC 系列处理器则使用 Big-endian，另外 TCP/IP 协议和 Java 虚拟机的字节序也是 Big-endian。

例如十六进制整数 0x12345678 存入以 1000H 开始的内存中：

![img](http://study.ctfcaict.com/media/course/html/3/site/linux/pic/1.3_byte_order.png)

我们在内存中实际地看一下，在地址 `0xffffd584` 处有字符 `1234`，在地址 `0xffffd588` 处有字符 `5678`。

```
gdb-peda$ x/w 0xffffd584
0xffffd584:     0x34333231
gdb-peda$ x/4wb 0xffffd584
0xffffd584:     0x31    0x32    0x33    0x34
gdb-peda$ python print('\x31\x32\x33\x34')
1234

gdb-peda$ x/w 0xffffd588
0xffffd588:     0x38373635
gdb-peda$ x/4wb 0xffffd588
0xffffd588:     0x35    0x36    0x37    0x38
gdb-peda$ python print('\x35\x36\x37\x38')
5678

gdb-peda$ x/2w 0xffffd584
0xffffd584:     0x34333231      0x38373635
gdb-peda$ x/8wb 0xffffd584
0xffffd584:     0x31    0x32    0x33    0x34    0x35    0x36    0x37    0x38
gdb-peda$ python print('\x31\x32\x33\x34\x35\x35\x36\x37\x38')
123455678
db-peda$ x/s 0xffffd584
0xffffd584:     "12345678"
```



## 输入输出

- 使用命令的输出作为可执行文件的输入参数
- `$ ./vulnerable 'your_command_here'`
- `$ ./vulnerable $(your_command_here)`
- 使用命令作为输入
- `$ your_command_here | ./vulnerable`
- 将命令行输出写入文件
- `$ your_command_here > filename`
- 使用文件作为输入
- `$ ./vulnerable < filename`

## 文件描述符

在 Linux 系统中一切皆可以看成是文件，文件又分为：普通文件、目录文件、链接文件和设备文件。文件描述符（file descriptor）是内核管理已被打开的文件所创建的索引，使用一个非负整数来指代被打开的文件。

标准文件描述符如下：

| 文件描述符 | 用途     | stdio 流 |
| ---------- | -------- | -------- |
| 0          | 标准输入 | stdin    |
| 1          | 标准输出 | stdout   |
| 2          | 标准错误 | stderr   |

当一个程序使用 `fork()` 生成一个子进程后，子进程会继承父进程所打开的文件表，此时，父子进程使用同一个文件表，这可能导致一些安全问题。如果使用 `vfork()`，子进程虽然运行于父进程的空间，但拥有自己的进程表项。

## 核心转储

当程序运行的过程中异常终止或崩溃，操作系统会将程序当时的内存、寄存器状态、堆栈指针、内存管理信息等记录下来，保存在一个文件中，这种行为就叫做核心转储（Core Dump）。

#### 会产生核心转储的信号

| Signal  | Action | Comment                  |
| ------- | ------ | ------------------------ |
| SIGQUIT | Core   | Quit from keyboard       |
| SIGILL  | Core   | Illegal Instruction      |
| SIGABRT | Core   | Abort signal from abort  |
| SIGSEGV | Core   | Invalid memory reference |
| SIGTRAP | Core   | Trace/breakpoint trap    |

#### 开启核心转储

- 输入命令 `ulimit -c`，输出结果为 `0`，说明默认是关闭的。

- 输入命令 `ulimit -c unlimited` 即可在当前终端开启核心转储功能。

- 如果想让核心转储功能永久开启，可以修改文件 

  ```
  /etc/security/limits.conf
  ```

  ，增加一行：  

  ```
  #<domain>      <type>  <item>         <value>
  *               soft    core            unlimited
  ```

#### 修改转储文件保存路径

- ```
  /proc/sys/kernel/core_uses_pid
  ```

  ，可以使生成的核心转储文件名变为 

  ```
  core.[pid]
  ```

   的模式。  

  ```
  # echo 1 > /proc/sys/kernel/core_uses_pid
  ```

- 还可以修改 

  ```
  /proc/sys/kernel/core_pattern
  ```

   来控制生成核心转储文件的保存位置和文件名格式。  

  ```
  # echo /tmp/core-%e-%p-%t > /proc/sys/kernel/core_pattern
  ```

    此时生成的文件保存在 

  ```
  /tmp/
  ```

   目录下，文件名格式为 

  ```
  core-[filename]-[pid]-[time]
  ```

#### 使用 gdb 调试核心转储文件

```
$ gdb [filename] [core file]
```

#### 例子

```
$ cat core.c
#include <stdio.h>
void main(int argc, char **argv) {
    char buf[5];
    scanf("%s", buf);
}
$ gcc -m32 -fno-stack-protector core.c
$ ./a.out
AAAAAAAAAAAAAAAAAAAA
Segmentation fault (core dumped)
$ file /tmp/core-a.out-12444-1503198911
/tmp/core-a.out-12444-1503198911: ELF 32-bit LSB core file Intel 80386, version 1 (SYSV), SVR4-style, from './a.out', real uid: 1000, effective uid: 1000, real gid: 1000, effective gid: 1000, execfn: './a.out', platform: 'i686'
$ gdb a.out /tmp/core-a.out-12444-1503198911 -q
Reading symbols from a.out...(no debugging symbols found)...done.
[New LWP 12444]
Core was generated by `./a.out'.
Program terminated with signal SIGSEGV, Segmentation fault.
#0  0x5655559b in main ()
gdb-peda$ info frame
Stack level 0, frame at 0x41414141:
 eip = 0x5655559b in main; saved eip = <not saved>
 Outermost frame: Cannot access memory at address 0x4141413d
 Arglist at 0x41414141, args:
 Locals at 0x41414141, Previous frame's sp is 0x41414141
Cannot access memory at address 0x4141413d
```

## 调用约定

函数调用约定是对函数调用时如何传递参数的一种约定。关于它的约定有许多种，下面我们分别从内核接口和用户接口介绍 32 位和 64 位 Linux 的调用约定。

#### 内核接口

**x86-32 系统调用约定**：Linux 系统调用使用寄存器传递参数。`eax` 为 syscall_number，`ebx`、`ecx`、`edx`、`esi`、`ebp` 用于将 6 个参数传递给系统调用。返回值保存在 `eax` 中。所有其他寄存器（包括 EFLAGS）都保留在 `int 0x80` 中。

**x86-64 系统调用约定**：内核接口使用的寄存器有：`rdi`、`rsi`、`rdx`、`r10`、`r8`、`r9`。系统调用通过 `syscall` 指令完成。除了 `rcx`、`r11` 和 `rax`，其他的寄存器都被保留。系统调用的编号必须在寄存器 `rax` 中传递。系统调用的参数限制为 6 个，不直接从堆栈上传递任何参数。返回时，`rax` 中包含了系统调用的结果。而且只有 INTEGER 或者 MEMORY 类型的值才会被传递给内核。

#### 用户接口

**x86-32 函数调用约定**：参数通过栈进行传递。最后一个参数第一个被放入栈中，直到所有的参数都放置完毕，然后执行 call 指令。这也是 Linux 上 C 语言函数的方式。

**x86-64 函数调用约定**：x86-64 下通过寄存器传递参数，这样做比通过栈有更高的效率。它避免了内存中参数的存取和额外的指令。根据参数类型的不同，会使用寄存器或传参方式。如果参数的类型是 MEMORY，则在栈上传递参数。如果类型是 INTEGER，则顺序使用 `rdi`、`rsi`、`rdx`、`rcx`、`r8` 和 `r9`。所以如果有多于 6 个的 INTEGER 参数，则后面的参数在栈上传递。

## 环境变量

- 按照生命周期划分
- 永久环境变量：修改相关配置文件，永久生效。
- 临时环境变量：使用 `export` 命令，在当前终端下生效，关闭终端后失效。
- 按照作用域划分
- 系统环境变量：对该系统中所有用户生效。
- 用户环境变量：对特定用户生效。

#### 设置方法

1. 在文件 `/etc/profile` 中添加变量，这种方法对所有用户永久生效。如：

   ```
   # Set our default path
   PATH="/usr/local/sbin:/usr/local/bin:/usr/bin"
   export PATH
   ```

   添加后执行命令 

   ```
   source /etc/profile
   ```

    使其生效。

   

2. 在文件 `~/.bash_profile` 中添加变量，这种方法对当前用户永久生效。其余同上。

3. 直接运行命令 `export` 定义变量，这种方法只对当前终端临时生效。

#### 常用变量

使用命令 `echo` 打印变量：

```
$ echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/bin:/usr/lib/jvm/default/bin:/usr/bin/site_perl:/usr/bin/vendor_perl:/usr/bin/core_perl
$ echo $HOME
/home/firmy
$ echo $LOGNAME
firmy
$ echo $HOSTNAME
firmy-pc
$ echo $SHELL
/bin/bash
$ echo $LANG
en_US.UTF-8
```

使用命令 `env` 可以打印出所有环境变量：

```
$ env
```

使用命令 `set` 可以打印处所有本地定义的 shell 变量：

```
$ set
```

使用命令 `unset` 可以清楚环境变量：

```
$ unset $变量名
```



#### LD_PRELOAD

该环境变量可以定义在程序运行前优先加载的动态链接库。在 pwn 题目中，我们可能需要一个特定的 libc，这时就可以定义该变量：

```
$ LD_PRELOAD=/path/to/libc.so ./binary
```

一个例子：

```
$ ldd /bin/true
    linux-vdso.so.1 =>  (0x00007fff9a9fe000)
    libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f1c083d9000)
    /lib64/ld-linux-x86-64.so.2 (0x0000557bcce6c000)
$ LD_PRELOAD=~/libc.so.6 ldd /bin/true
    linux-vdso.so.1 =>  (0x00007ffee55e9000)
    /home/firmy/libc.so.6 (0x00007f4a28cfc000)
    /lib64/ld-linux-x86-64.so.2 (0x000055f33bc50000)
```



注意，这种方法得根据实际情况来用，大概就是使用的发行版要相同（`interpreter` 相同），上面的例子中两个 libc 是这样的：

```
$ file /lib/x86_64-linux-gnu/libc-2.23.so
/lib/x86_64-linux-gnu/libc-2.23.so: ELF 64-bit LSB shared object, x86-64, version 1 (GNU/Linux), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=088a6e00a1814622219f346b41e775b8dd46c518, for GNU/Linux 2.6.32, stripped
$ file ~/libc.so.6
/home/firmy/libc.so.6: ELF 64-bit LSB shared object, x86-64, version 1 (GNU/Linux), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=088a6e00a1814622219f346b41e775b8dd46c518, for GNU/Linux 2.6.32, stripped
```

都是 `interpreter /lib64/ld-linux-x86-64.so.2`，所以可以替换。



而下面的例子是在 Arch Linux 上使用一个 Ubuntu 的 libc，就会出错：

```
$ ldd /bin/true
        linux-vdso.so.1 (0x00007ffc969df000)
        libc.so.6 => /usr/lib/libc.so.6 (0x00007f7ddde17000)
        /lib64/ld-linux-x86-64.so.2 => /usr/lib64/ld-linux-x86-64.so.2 (0x00007f7dde3d7000)
$ LD_PRELOAD=~/libc.so.6 ldd /bin/true
Illegal instruction (core dumped)
$ file /usr/lib/libc-2.26.so
/usr/lib/libc-2.26.so: ELF 64-bit LSB shared object, x86-64, version 1 (GNU/Linux), dynamically linked, interpreter /usr/lib/ld-linux-x86-64.so.2, BuildID[sha1]=458fd9997a454786f071cfe2beb234542c1e871f, for GNU/Linux 3.2.0, not stripped
$ file ~/libc.so.6
/home/firmy/libc.so.6: ELF 64-bit LSB shared object, x86-64, version 1 (GNU/Linux), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=088a6e00a1814622219f346b41e775b8dd46c518, for GNU/Linux 2.6.32, stripped
```

一个在 `interpreter /usr/lib/ld-linux-x86-64.so.2`，而另一个在 `interpreter /lib64/ld-linux-x86-64.so.2`。



## /proc/pid

proc 文件系统是 Linux 内核提供的，为访问系统内核数据的操作提供接口。在该文件系统下，有一些以数字命名的目录，这些数字是进程的 PID 号，而这些目录是进程目录。

目录下的所有文件如下，然后会介绍几个比较重要的：

```
$ cat - &
[1] 2865
$ ls /proc/2865/
attr             cpuset   limits      ns             root          statm
autogroup        cwd      map_files   numa_maps      sched         status
auxv             environ  maps        oom_adj        schedstat     syscall
cgroup           exe      mem         oom_score      setgroups     task
clear_refs       fd       mountinfo   oom_score_adj  smaps         timers
cmdline          fdinfo   mounts      pagemap        smaps_rollup  timerslack_ns
comm             gid_map  mountstats  personality    stack         uid_map
coredump_filter  io       net         projid_map     stat          wchan

[1]+  Stopped                 cat -
```



#### /proc/[pid]/maps

这个文件大概是最常用的，用于显示进程的内存区域映射信息：

```
$ cat /proc/2865/maps
5580631c6000-5580631ce000 r-xp 00000000 08:01 4981196                    /usr/bin/cat
5580633cd000-5580633ce000 r--p 00007000 08:01 4981196                    /usr/bin/cat
5580633ce000-5580633cf000 rw-p 00008000 08:01 4981196                    /usr/bin/cat
558063c7d000-558063c9e000 rw-p 00000000 00:00 0                          [heap]
7f6301cd7000-7f6302027000 r--p 00000000 08:01 4993768                    /usr/lib/locale/locale-archive
7f6302027000-7f63021d5000 r-xp 00000000 08:01 4982395                    /usr/lib/libc-2.26.so
7f63021d5000-7f63023d5000 ---p 001ae000 08:01 4982395                    /usr/lib/libc-2.26.so
7f63023d5000-7f63023d9000 r--p 001ae000 08:01 4982395                    /usr/lib/libc-2.26.so
7f63023d9000-7f63023db000 rw-p 001b2000 08:01 4982395                    /usr/lib/libc-2.26.so
7f63023db000-7f63023df000 rw-p 00000000 00:00 0
7f63023df000-7f6302404000 r-xp 00000000 08:01 4982398                    /usr/lib/ld-2.26.so
7f63025c1000-7f63025c3000 rw-p 00000000 00:00 0
7f63025e1000-7f6302603000 rw-p 00000000 00:00 0
7f6302603000-7f6302604000 r--p 00024000 08:01 4982398                    /usr/lib/ld-2.26.so
7f6302604000-7f6302605000 rw-p 00025000 08:01 4982398                    /usr/lib/ld-2.26.so
7f6302605000-7f6302606000 rw-p 00000000 00:00 0
7fff2ab81000-7fff2aba2000 rw-p 00000000 00:00 0                          [stack]
7fff2abef000-7fff2abf2000 r--p 00000000 00:00 0                          [vvar]
7fff2abf2000-7fff2abf4000 r-xp 00000000 00:00 0                          [vdso]
ffffffffff600000-ffffffffff601000 r-xp 00000000 00:00 0                  [vsyscall]
```



#### /proc/[pid]/stack

这个文件表示当前进程的内核调用栈信息：

```
$ sudo cat /proc/2865/stack
[<ffffffffa008d05e>] do_signal_stop+0xae/0x1f0
[<ffffffffa008e50c>] get_signal+0x18c/0x5a0
[<ffffffffa002ac26>] do_signal+0x36/0x610
[<ffffffffa0003019>] exit_to_usermode_loop+0x69/0xa0
[<ffffffffa00038eb>] syscall_return_slowpath+0x9b/0xb0
[<ffffffffa06926e4>] entry_SYSCALL_64_fastpath+0x7b/0x7d
[<ffffffffffffffff>] 0xffffffffffffffff
```



#### /proc/[pid]/auxv

该文件包含了传递给进程的解释器信息，即 auxv(AUXiliary Vector)，每一项都是由一个 unsigned long 长度的 ID 加上一个 unsigned long 长度的值构成：

```
$ xxd -e -g8 /proc/2865/auxv
00000000: 0000000000000021 00007fff2abf2000  !........ .*....
00000010: 0000000000000010 00000000bfebfbff  ................
00000020: 0000000000000006 0000000000001000  ................
00000030: 0000000000000011 0000000000000064  ........d.......
00000040: 0000000000000003 00005580631c6040  ........@`.c.U..
00000050: 0000000000000004 0000000000000038  ........8.......
00000060: 0000000000000005 0000000000000009  ................
00000070: 0000000000000007 00007f63023df000  ..........=.c...
00000080: 0000000000000008 0000000000000000  ................
00000090: 0000000000000009 00005580631c8290  ...........c.U..
000000a0: 000000000000000b 00000000000003e8  ................
000000b0: 000000000000000c 00000000000003e8  ................
000000c0: 000000000000000d 00000000000003e8  ................
000000d0: 000000000000000e 00000000000003e8  ................
000000e0: 0000000000000017 0000000000000000  ................
000000f0: 0000000000000019 00007fff2ab9ff39  ........9..*....
00000100: 000000000000001a 0000000000000000  ................
00000110: 000000000000001f 00007fff2aba1feb  ...........*....
00000120: 000000000000000f 00007fff2ab9ff49  ........I..*....
00000130: 0000000000000000 0000000000000000  ................
```

每个值具体是做什么的，可以用下面的办法显示出来，对比看一看，更详细的可以查看 `/usr/include/elf.h` 和 `man ld.so`：

```
$ LD_SHOW_AUXV=1 cat -
AT_SYSINFO_EHDR: 0x7fff6afb3000
AT_HWCAP:        bfebfbff
AT_PAGESZ:       4096
AT_CLKTCK:       100
AT_PHDR:         0x557b68217040
AT_PHENT:        56
AT_PHNUM:        9
AT_BASE:         0x7f41e5689000
AT_FLAGS:        0x0
AT_ENTRY:        0x557b68219290
AT_UID:          1000
AT_EUID:         1000
AT_GID:          1000
AT_EGID:         1000
AT_SECURE:       0
AT_RANDOM:       0x7fff6aedc0a9
AT_HWCAP2:       0x0
AT_EXECFN:       /usr/bin/cat
AT_PLATFORM:     x86_64
```

值得一提的是，`AT_SYSINFO_EHDR` 所对应的值是一个叫做的 VDSO(Virtual Dynamic Shared Object) 的地址。在 ret2vdso 漏洞利用方法中会用到（参考章节6.1.6）。



#### /proc/[pid]/environ

该文件包含了进程的环境变量：

```
$ strings /proc/2865/environ
```



#### /proc/[pid]/fd

该文件包含了进程打开文件的情况：

```
$ ls -al /proc/2865/fd
total 0
dr-x------ 2 firmy firmy  0 12月 30 11:13 .
dr-xr-xr-x 9 firmy firmy  0 12月 30 11:13 ..
lrwx------ 1 firmy firmy 64 12月 30 12:31 0 -> /dev/pts/2
lrwx------ 1 firmy firmy 64 12月 30 12:31 1 -> /dev/pts/2
lrwx------ 1 firmy firmy 64 12月 30 12:31 2 -> /dev/pts/2
```



#### /proc/[pid]/status[¶](http://study.ctfcaict.com/media/course/html/3/863846a6-3305-4770-9bb5-ecaa12171a2e.html#procpidstatus)

该文件包含了进程的状态信息：

```
$ cat /proc/2865/status
Name:   cat
Umask:  0022
State:  T (stopped)
Tgid:   2865
Ngid:   0
Pid:    2865
PPid:   2059
TracerPid:      0
Uid:    1000    1000    1000    1000
Gid:    1000    1000    1000    1000
FDSize: 256
Groups: 3 7 10 56 90 91 93 95 96 98 1000
NStgid: 2865
NSpid:  2865
NSpgid: 2865
NSsid:  2059
VmPeak:     7828 kB
VmSize:     7828 kB
VmLck:         0 kB
VmPin:         0 kB
VmHWM:       788 kB
VmRSS:       788 kB
RssAnon:              64 kB
RssFile:             724 kB
RssShmem:              0 kB
VmData:      312 kB
VmStk:       132 kB
VmExe:        32 kB
VmLib:      1876 kB
VmPTE:        40 kB
VmPMD:        12 kB
VmSwap:        0 kB
HugetlbPages:          0 kB
Threads:        1
SigQ:   2/47723
SigPnd: 0000000000000000
ShdPnd: 0000000000000000
SigBlk: 0000000000000000
SigIgn: 0000000000000000
SigCgt: 0000000000000000
CapInh: 0000000000000000
CapPrm: 0000000000000000
CapEff: 0000000000000000
CapBnd: 0000003fffffffff
CapAmb: 0000000000000000
NoNewPrivs:     0
Seccomp:        0
Cpus_allowed:   ff
Cpus_allowed_list:      0-7
Mems_allowed:   00000001
Mems_allowed_list:      0
voluntary_ctxt_switches:        1
nonvoluntary_ctxt_switches:     0
```

#### /proc/[pid]/syscall

该文件包含了进程正在执行的系统调用：

```
$ sudo cat /proc/2865/syscall
0 0x0 0x7f63025e2000 0x20000 0x22 0xffffffffffffffff 0x0 0x7fff2ab9f958 0x7f630210ea11
```

第一个值是系统调用号，后面跟着是六个参数，最后两个值分别是堆栈指针和指令计数器的值。