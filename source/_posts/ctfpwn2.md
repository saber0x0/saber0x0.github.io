```
 学习日记2
```

**格式化字符串**

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625195611418.png" alt="image-20210625195611418" style="zoom:60%;" />

**格式化占位符**

格式化字符串中的占位符用于指明输出的参数值如何格式化。

| **字符** | **描述**                                                     |
| -------- | ------------------------------------------------------------ |
| n$       | n是用这个格式说明符（specifier）显示第几个参数；这使得参数可以输出多次，使用多个格式说明符，以不同的顺序输出。  如果任意一个占位符使用了parameter，则其他所有占位符必须也使用parameter。这是POSIX扩展，不属于ISO  C。例：printf("%2$d %2$#x; %1$d %1$#x",16,17) 产生"17 0x11; 16  0x10" |

 Flags可为0个或多个：

| **字 符** | **描述**                                                     |
| --------- | :----------------------------------------------------------- |
| +         | 总是表示有符号数值的'+'或'-'号，缺省情况是忽略正数的符号。仅适用于数值类型。 |
| 空格      | 使得有符号数的输出如果没有正负号或者输出0个字符，则前缀1个空格。如果空格与'+'同时出现，则空格说明符被忽略。 |
| -         | 左对齐。缺省情况是右对齐。                                   |
| #         | 对于'g'与'G'，不删除尾部0以表示精度。对于'f',  'F', 'e', 'E', 'g', 'G', 总是输出小数点。对于'o', 'x', 'X', 在非0数值前分别输出前缀0, 0x, and  0X表示数制。 |
| 0         | 如果width选项前缀以0，则在左侧用0填充直至达到宽度要求。例如printf("%2d",  3)输出" 3"，printf("%02d",  3)输出"03"。如果0与-均出现，则0被忽略，即左对齐依然用空格填充。 |

**格式化占位符**

Length指出浮点型参数或整型参数的长度。可以忽略，或者是下述：

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625200440768.png" alt="image-20210625200440768" style="zoom:50%;" />

**漏洞利用**

- 读取数据 

- - %x
  - %d
  - %s

- 写入数据 

- - %n

**读取数据**

**$操作符**

读取一个特定的参数

printf("%3$s", 1, "b", "c", 4);

将显示3而不是对应顺序的1。其他例子：

- printf("%3$d",1,2,3);，显示3
- printf("%3$d     %2$d %1$d",1,2,3);，显示3 2 1

读取栈上第100个dword数据

$ python -c "print '%100$08x'" | ./print2
 ffa3cad0

通过$操作符读取任意地址数据！！！

**写入数据**

- 任意地址任意写

- - 任意地址：$控制
  - 任意值：      %xd通过控制x改变输出的字符个数

- getshell

- - 任意地址任意写------>getshell?

- - 覆盖got表！！！

**AAR和AAW**

- AAR：任意地址读（arbitrary     address read）
- AAW：任意地址写（arbitrary     address write）
- 能做到这两点就非常强了，重点是如何拿到shell
- Q：写哪些地址，写什么值进去，可以劫持控制流？
- A：GOT表

**GOT表**

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625200629787.png" alt="image-20210625200629787" style="zoom:50%;" />

**防御**

- 调用约定 

- - caller将所有参数放在栈上
  - callee假设所有参数都在栈上

- 漏洞确实存在，假设caller传的参数和callee期望的参数个数不同

- 解决方案：在caller和callee之间做验证     

- - caller一共传了多少参数？ 

  - - 编译器在编译阶段可以知道

  - callee使用了多少参数？ 

  - - 仅在运行时方可知道

### **Glibc堆分配机制**

**堆的使用**

| **分类**   | **函数**                                              |
| ---------- | ----------------------------------------------------- |
| 程序员应用 |                                                       |
| libc 函数  | malloc(),  realloc(), free(), brk(), mmap(), munmap() |
| 内核调用   | brk,  mmap, munmap                                    |

**Linux 进程布局**

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625200833180.png" alt="image-20210625200833180" style="zoom:50%;" />

没错！又是它o(*￣▽￣*)ブ！！！

**堆 VS 栈**

- 堆 
  - 动态分配 
  - 对象，大缓冲区，结构体... 

- 更慢，手动 

- - malloc / calloc /      recalloc / free
  - new / delete

- 栈

- - 在编译阶段就固定的内存分配
  - 局部变量，返回地址，函数地址

- 快，自动

- - 编译阶段就完成
  - 抽象出分配/取消分配的概念

**堆的实现**

不同的堆实现

- dlmalloc
- ptmalloc (glibc)
- tcmalloc (Chrome,     replaced)
- jemalloc     (Firefox/Facebook)
- nedmalloc
- Hoard

**glibc的堆实现**

chunk
 堆的基本单位是一个一个chunk块，chunk的结构如下：

```c++
struct malloc_chunk {
    INTERNAL_SIZE_T    prev_size;     /* Size of previous chunk (if free).  */
    INTERNAL_SIZE_T    size;          /* Size in bytes, including overhead. */
    struct malloc_chunk* fd;          /* double links -- used only if free. */
    struct malloc_chunk* bk;                                                                           / * Only used for large blocks: pointer to next larger size.  */
    struct malloc_chunk* fd_nextsize; /* double links -- used only if free. */
    struct malloc_chunk* bk_nextsize;
 };
```

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625201205824.png" alt="image-20210625201205824" style="zoom:50%;" />

**chunk size**

为了执行效率，chunk的长度会进行对齐

- 32bit： （x + 4）对齐到8

| **data size** | **0 ~ 4** | **5 ~ 12** | **13 ~ 20** | **...** | **53 ~ 60** | **61 ~ 68** | **69 ~ 76** | **77 ~ 84** |
| ------------- | --------- | ---------- | ----------- | ------- | ----------- | ----------- | ----------- | ----------- |
| chunk  size   | 16        | 16         | 24          | ...     | 64          | 72          | 80          | 88          |

- 64bit： （x + 4）对齐到16

| **data size** | **0 ~ 8** | **9 ~ 24** | **25 ~ 40** | **...** | **105 ~ 120** | **121 ~ 136** | **137 ~ 152** | **153 ~ 168** |
| ------------- | --------- | ---------- | ----------- | ------- | ------------- | ------------- | ------------- | ------------- |
| chunk  size   | 32        | 32         | 24          | ...     | 64            | 72            | 80            | 88            |

**In used chunk**

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625201332771.png" alt="image-20210625201332771" style="zoom:50%;" />

- 如果前面的chunk是使用状态，prev_size则属于前一个堆块。
- prev_in_use表明前一个堆块是否free状态（且不在fastbins内）
- is_mmaped说明内存来自于mmap()
- non_main_arena表明该堆块由非主线程分配

**free chunk**

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625202247694.png" alt="image-20210625202247694" style="zoom:50%;" />

如果前面的chunk是空闲状态，prev_size则记录前面chunk的大小。

下一个堆块的prev_size记录本堆块的大小。

其中，size不同于pre_size的是，size中的低三位作为flag。

- 最低位:前一个 chunk         是否正在使用
- 倒数第二位:当前 chunk     是否是通过 mmap 方式产生的
- 倒数第三位:这个 chunk     是否属于一个线程的 arena
  需要注意的是，如果该free chunk落在fastbin中，这些数据将不会被设置。

**chunk**

- 正在被使用的chunk 

- - 被程序的数据指针指向
  - 长度是对于chunk本身是可知的
  - free()因此得以工作

- free状态的chunk 

- - 不再被程序指向
  - 稍后会被重复使用来回应内存分配
  - 需要有效的组织管理free  chunk以便于maclloc()来分配
  - how？

**free chunks的组织管理**

- 链表，通常的，一个链表拥有的chunk长度相同     

- - 需要多个链表
  - 以管理不同长度的chunk
  - 单链表或双向链表
  - 链表被分组为几个BINs
  - fastbins,unsorted_bin,smallbins,largebins

**Arena(32bit)**

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625203040407.png" alt="image-20210625203040407" style="zoom:50%;" />

**Get Arena**

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625203114779.png" alt="image-20210625203114779" style="zoom:50%;" />

**fastbin**

管理16bytes-64bytes（64位下为32bytes~128bytes）的chunk的单向链表数据结构，由于需要加速程序执行速度的原因，linux对于fastbin的检查较为松散，因此利用起来也较为方便。

- 单向链表 

- - 特点：快
  - 释放后的chunk插入fastbin表头
  - 只使用fd指针，bk指针，pre_size,p bit无效

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625203309172.png" alt="image-20210625203309172" style="zoom:50%;" />

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625203322743.png" alt="image-20210625203322743" style="zoom:40%;" />

**unsorted_bin**

- malloc() 

- - 如果搜索到unsorted_bin，对它管理的chunk一一处理      

  - - 把chunk归放到small/large  bin（唯一增加small/large bin的地方）
    - 如果长度刚好满足要求，就返回chunk并结束搜索

- free() 

- - 如果free的chunk不能放到fastbin，就放到unsorted_bin。

  - 向前面的空闲的chunk合并 

  - - 通过检查p bit

  - 向后面的空闲的chunk合并 

  - - 通过检查它后一个chunk p       bit

  - 设置fd，bk指针，后一个chunk的prev_size,p  bit

**Small-bins**

small-bins特点：

- malloc() 

- - 如果找到对应长度的chunk，则直接返回

- free() 

- - 不会直接把chunk放进small-bin

- fd/bk被设置 

- - 当chunk从unsorted_bin删除时。

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625203558989.png" alt="image-20210625203558989" style="zoom:50%;" />

**Large-bins**

- malloc() 

- - 如果large-bin被搜索到 

  - - 寻找满足要求的最小chunk
    - 或者分裂大chunk，一份返回给用户，一份加入到unsorted_bin

- free() 

- - 不会直接放到large-bin

- fd/bk, fd_nextsize/bk_nextsize 被设置 

- - 当其从unsorted_bin删除时。

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625203631401.png" alt="image-20210625203631401" style="zoom:50%;" />

**top chunk**

初次malloc()后，没有被分出去的堆。

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625203756685.png" alt="image-20210625203756685" style="zoom:50%;" />

**malloc过程**

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625203823867.png" alt="image-20210625203823867" style="zoom:50%;" />

- 检查长度是否符合fastbin的范围，如果在其中找到，则     

- - 将其从单链表删除
  - 更新fd指针

- 检查长度是否符合smallbins的范围，如果在其中找到，则     

- - 从双向链表中删除
  - 更新fd/bk指针

- 如果尝试分配长度很大的chunk，不直接搜索large-bin，而是进行chunk整理     

- - 把fastbin放到unsorted_bin中
  - 更新下一个chunk的prev_size和p      bit
  - 合并空闲堆快

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625203848532.png" alt="image-20210625203848532" style="zoom:50%;" />

- 在unsorted_bin中搜索 

- - 如果last_reminder      cache足够大，就分一块出来
  - 如果找到合适大小的chunk就返回
  - 对于搜索的每个chunk，将其按照范围放进small/large      bins

- 更新fd/bk指针

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625203902464.png" alt="image-20210625203902464" style="zoom:50%;" />

- 在small/large中搜索 

- - 在最小的非空small/large      bin中找到足够大的chunk 

  - - unlink，split

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625203923167.png" alt="image-20210625203923167" style="zoom:50%;" />

- 从TOP chunk中获取内存 

- - 如果top      chunk足够大，就分一部分出来
  - 如果没有fastbin就从系统获取内存
  - 再次整理fastbin和unsorted_bin

**free**

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625203941475.png" alt="image-20210625203941475" style="zoom:50%;" />

- 检查元数据的安全性 

- - e.g. :      ptr->fd->bk == ptr

- 如果大小在fastbin范围内就放到fastbin里面     

- - 更新fastbin，fd

- 合并前一个free状态的chunk 

- - unlink

- 合并后一个free状态的chunk 

- - unlink，top

**unlink**

- 从链表中删除结点，并更新BK/FD

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625204020527.png" alt="image-20210625204020527" style="zoom:50%;" />

\#define unlink(p,BK,FD)
 {
   FD = p -> fd;
   BK = p -> bk;
   FD -> bk = BK;
   BK -> fd = FD;
 }

**什么时候进行unlink**

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625204048786.png" alt="image-20210625204048786" style="zoom:50%;" />

- free chunk从双向链表中被取出     

- - 一种情况：与它物理相邻的堆块合并
  - 另一种情况：回应系统的分配请求

- 如果空闲堆块被损害？

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625204119490.png" alt="image-20210625204119490" style="zoom:50%;" />

**当chunk被unlink时**

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625204132422.png" alt="image-20210625204132422" style="zoom:50%;" />

**main_areana 地址**

- 运行时libc基地址

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625204159171.png" alt="image-20210625204159171" style="zoom:50%;" />

编译时确定的main_arena的偏移

![image-20210625204227122](C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625204227122.png)

运行时main_arena的地址

0xf7e16000 + 0x001BA420= 0xf7fd0420

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625204302602.png" alt="image-20210625204302602" style="zoom:50%;" />

**main_arena内容**

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625204321373.png" alt="image-20210625204321373" style="zoom:50%;" />

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625204358309.png" alt="image-20210625204358309" style="zoom:50%;" />

- free()和malloc()依赖于元数据(chunk的头部信息)来     

- - 遍历      bins/lists/chunks
  - 将内存返回给用户

- 但是这些元数据没有很好的保护 

- - 堆溢出，use-after-free

**堆溢出利用**

**Glibc：Chunk In Use**

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625204532029.png" alt="image-20210625204532029" style="zoom:50%;" />

- prev_size 区域 

- - 属于前一个chunk
  - 如果前一个chunk是free状态（并且不在fastbin）就会被设置

- prev_in_use 

- - 若前一个chunk是free状态（并且不在fastbin）则置0

- is_mmaped 

- - 是否是通过mmap()产生的

- non_main_arena

**Glibc:被释放的chunk**

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625204722491.png" alt="image-20210625204722491" style="zoom:50%;" />

- 下一个chunk的prev_size 这个free chunk的长度
- 下一个chunk的prev_in_use 为0
- 但是 如果这个被释放的chunk在fastbin，这些东西就不会被设置
- 这个 chunk 是否属于一个线程的     arena

**堆溢出**

- 基本上与栈溢出基本类似

- 栈溢出的目标 

- - 返回地址
  - 保存在栈帧的指针
  - 局部函数变量
  - 异常句柄
  - 其他敏感数据

- 堆溢出的目标 

- - 堆的元数据
  - 对象中的函数指针
  - 对象中的vtable指针
  - 其他敏感数据

**例子**

```c++
	struct toystr()
{
    void (* message)(char *);
    char buffer[20];
}
	coolguy = malloc(sizeof(struct toystr))
	lameguy = malloc(sizeof(struct toystr))
	coolguy -> message = &print_cool;
	lameguy -> message = &print_meh;
	puts("Input coolguy's name:");
	fgets(coolguy->fuffer,200,stdin);
	coolguy->buffer[strcspn(coolguy->buffer,"\n")] = 0;
	puts("Input lameguy's name:");
	fgets(coolguy->fuffer,200,stdin);
	coolguy->buffer[strcspn(coolguy->buffer,"\n")] = 0;
	coolguy->message(coolguy->buffer);
	lameguy->message(lameguy->buffer);
```

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625205042187.png" alt="image-20210625205042187" style="zoom:50%;" />

```c++
coolguy = malloc(sizeof(struct toystr))
lameguy = malloc(sizeof(struct toystr))
coolguy -> message = &print_cool;
lameguy -> message = &print_meh;
puts("Input coolguy's name:");
fgets(coolguy->fuffer,200,stdin);
coolguy->buffer[strcspn(coolguy->buffer,"\n")] = 0;
puts("Input lameguy's name:");
fgets(coolguy->fuffer,200,stdin);
coolguy->buffer[strcspn(coolguy->buffer,"\n")] = 0;
coolguy->message(coolguy->buffer);
lameguy->message(lameguy->buffer);//<--------被溢出覆盖的函数指针
```

- 堆溢出有什么其他的利用方法？ 

- - 对象中的函数指针
  - 管理堆的元数据
  - VTable指针

**溢出一个正在被使用的chunk**

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625205135042.png" alt="image-20210625205135042" style="zoom:50%;" />

- 溢出 p bit（prev_in_use） 

- - 从0改变成1 

  - - 当这个chunk被free，前一个chunk不会被合并
    - note：只有被free的chunk比fast-bin-size大时才会发生合并操作

  - 从1改变成0 

  - - 当这个chunk被free时，前一个chunk会被合并
    - Q1:前一个"free"状态chunk的大小？
    - Q2:什么时候会发生合并？
    - 如果合并的free       chunk接下来被分配给其他对象，那么两个不同的对象会共享同一段内存

- 溢出 N bit（non_main_arena） 

- - 从0改变成1 

  - - 会标志其为non_main_arena，切换arena
    - 当这个chunk被释放，将会被添加到fastbin或者unsorted_bin
    - 指向fastbin/unsorted_bin的指针会更新

**溢出一个free chunk**

- 溢出 P bit     （prev_in_use） 

- - 从1改变成0 

  - - Q1：如果被分配给对象，P bit会被堆管理器纠正吗？
    - Q2:如果这个chunk被其他free chunk合并了呢？
      e.g.：该free chunk之后的chunk被释放，其会尝试向前面的chunk合并，得到一个非预期大小的free chunk
    - 这个free       chunk可以在稍后分配新的对象，新的对象会和前面的chunk共享内存
    - Q3:如果这个chunk在bins之间移动？

  - 从0改变成1，前面的free      chunk将不会被合并

  - 被释放的堆快由arena/fastbin追踪记录

  - 被释放的堆快可以在稍后分配给新的对象，或者被新的free chunk合并，或者按照 fastbin->unsorted_bin->smallbin/largebin的顺序移动	

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625205424699.png" alt="image-20210625205424699" style="zoom:50%;" />

- 溢出 size 区域 

- - 如果该chunk被分配？
  - 如果该chunk被物理相邻的chunk合并？
  - 如果这个chunk在bins间移动？

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625205445914.png" alt="image-20210625205445914" style="zoom:50%;" />

- 溢出 fd/bk指针 

- - 产生 unlink 问题

- 溢出 prev_size 区域 

- - 如果缩小prev_size 

  - - 当下一个chunk被释放时，会发生什么？       

    - - 会尝试合并当前chunk（的一部分）
      - 当前chunk（的一部分）将被unlink，导致unlink操作到假的fd/bk指针，并且当前        free chunk和合并的chunk重叠。

  - 如果放大prev_size 

  - - 当下一个chunk被释放时，会发生什么？       

    - - 会尝试合并当前chunk（大于实际）
      - （大于实际）的当前堆快将被unlink，导致unlink操作到假的fd/bk指针，并且前面（正在使用）的chunk会和新合并的free        chunk共享内存

**溢出TOP chunk的size区域**

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625205532518.png" alt="image-20210625205532518" style="zoom:50%;" />

- 缩小size 

- - 浪费内存

- 放大size 

- - 新的分配请求总是能被TOP      chunk满足
  - 这个（比实际大的）TOP      chunk可以覆盖包括 code/data,e.g.,got表，那么任意地址都可以被分配请求返回

**防御堆溢出**

- cookie 

- - 与StackGuard/GS（在返回地址附近插入cookie）类似，我们可以在堆对象附近插入cookie。

- 但是 

- - 运行时开销非常大

  - 没有一个好的时间节点进行cookie检查：      

  - - 对于栈，在函数返回时进行检查
    - 对于堆，一个还算合理的适当malloc/free被调用时进行检查，但其可能无法对被损害的对象进行操作。

**Use-After-Free (UAF)**

- 漏洞 

- - 被释放的内存被再次释放？
  - 被释放的内存被损坏？

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625205716558.png" alt="image-20210625205716558" style="zoom:50%;" />

- 攻击
  - 最常见的UAF攻击：VTable Hijacking

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625205758688.png" alt="image-20210625205758688" style="zoom:50%;" />

**UAF vulnerability(MS12-063)**

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625205915719.png" alt="image-20210625205915719" style="zoom:50%;" />

**UAF vulnerability(MS12-063)**<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625205933444.png" style="zoom:50%;" />

**动态分配的VTable(c++)**

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625205958106.png" alt="image-20210625205958106" style="zoom:50%;" />

```c++
void foo(Base2* obj){
    obj->vg4();
}
void main(){
    Base2* obj = new Sub();
    foo(obj);
}
```

```assembly
code section
; Function main()
push SIZE
call malloc()
mov ecx, eax
call Sub::Sub()
; now ECX points to the Sub object
add ecx, 8
; now ECX points to the Sub::Base2 object
call foo()
ret
; Function foo()
mov eax, [ecx]      ; read vfptr of Base2
mov edx, [eax+0x0C] ; get vg4() from vtable
call edx            ; call Base2::vg4()
ret
```

**VTable Hijacking 分类**

- 损坏VTable 

- - 覆盖VTable

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625210119072.png" alt="image-20210625210119072" style="zoom:50%;" />

- VTable注入 

- - 覆盖vfptr
  - 指向假的VTable

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625210150396.png" alt="image-20210625210150396" style="zoom:50%;" />

- VTable重用 

- - 覆盖vfptr
  - 指向包括VTable，数据等

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625210205114.png" alt="image-20210625210205114" style="zoom:50%;" />

**VTable Hijacking 实战**

- Pwn2Own 2014  firefox
- Pwn2Own 2014  chrome
- CVE-2014-1772   IE

```assembly
code section
; Function main()
push SIZE
call malloc()
mov ecx, eax
call Sub::Sub()
; now ECX points to the Sub object
add ecx, 8
; now ECX points to the Sub::Base2 object
call foo()
ret
; Function foo()
mov eax, [ecx]      ; read vfptr of Base2
mov edx, [eax+0x0C] ; get vg4() from vtable
call edx            ; call Base2::vg4()
ret
```

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625210304071.png" alt="image-20210625210304071" style="zoom:50%;" />

**防御VTable Hijacking**

**VTint**

| **-**        | **Attack**                      | **Requirement**              |
| ------------ | ------------------------------- | ---------------------------- |
| VTable  破坏 | 覆盖  VTable                    | VTable可写                   |
| VTable  注入 | 覆盖vfptr，指向被注入的VTable   | VTable可写                   |
| VTable  重用 | 覆盖vfptr，指向包括VTable，数据 | 形似VTable的数据，包括VTable |

| **-**        | **Attack**                      | **Requirement**              | **解决方案**      |
| ------------ | ------------------------------- | ---------------------------- | ----------------- |
| VTable  破坏 | 覆盖  VTable                    | VTable可写                   | VTable只读        |
| VTable  注入 | 覆盖vfptr，指向被注入的VTable   | VTable可写                   | VTable只读        |
| VTable  重用 | 覆盖vfptr，指向包括VTable，数据 | 形似VTable的数据，包括VTable | 不同的VTable/数据 |

**VTint vs. DEP**

| **-**        | **VTint**         |
| ------------ | ----------------- |
| VTable  破坏 | VTable只读        |
| VTable  注入 | VTable只读        |
| VTable  重用 | 不同的VTable/数据 |

| **-**        | **DEP**                          |
| ------------ | -------------------------------- |
| VTable  破坏 | 只读的代码段                     |
| VTable  注入 | 只读的代码段(可写的段不会被执行) |
| VTable  重用 | NO                               |

- 与DEP的相同之处 

- - 轻量级，可以是二进制可兼容的。

- 不同 

- - 加固后，可攻击面更小

**二进制级防护：vfGuard**

NDSS’15: Dynamic binary instrumentation

\- 在运行时检查虚拟调用指令（使用PIN）

\- 针对指令使用不同策略

- 优势 

- - 易于扩展，去执行不同策略

- 劣势 

- - 开销大（PIN的开销*118%）
  - 无法防御VTables重用攻击

**源代码级防护：vfGuard**

- Microsoft IE10,     Core Objects 

- - 在VTable结束地方的特殊cookies

- 优势 

- - 轻量级

- 劣势 

- - 只防御核心对象Core      Objects
  - 不能应付VTable注入
  - 信息泄漏

<img src="C:\Users\86153\AppData\Roaming\Typora\typora-user-images\image-20210625210454607.png" alt="image-20210625210454607" style="zoom:50%;" />

**源代码级防护：SafeDispatch**

- NDSS’14, LLVM-based     

- - 静态计算一系列合法目标
  - 根据计算结果进行动态验证

- 策略1: VTable Check 

- - 劣势： 

  - - 编译阶段需要耗费大量时间去分析
    - 高运行时开销（~30%）

- 策略2: moethod Check 

- - 劣势： 

  - - 编译阶段需要耗费大量时间去分析
    - 高运行时开销（~7%）

**源代码级防护：Forward Edge CFI**

- GCC-VTV     [Usenix’14], whitelist-based

  \- 在编译阶段计算一组不完整的合法目标
  \- 在加载是通过初始化函数合并不完整的数据
  \- 在运行时检验是否合法

- 优势 

- - 支持增量构建

- 劣势 

- - 运行时操作繁琐

**源代码级别的防护：RockJIT**

- CCS’15, CFI-based 

- - 在编译阶段收集类型信息
  - 根据收集到的信息计算加载时转移目标的等价类别
  - 更新CFI检查，在加载时只允许间接传输到一个等价类

- 优势 

- - 支持增量构造

- 劣势 

- - 加载时开销大

头疼≧ ﹏ ≦~~~
