---
title: 每日学说话(●'◡'●)python-常用模块
---

### sys模块

####  python之sys模块详解

sys模块功能多，我们这里介绍一些比较实用的功能，相信你会喜欢的，和我一起走进python的模块吧！

#### sys模块的常见函数列表

- `sys.argv`: 实现从程序外部向程序传递参数。
- `sys.exit([arg])`: 程序中间的退出，arg=0为正常退出。
- `sys.getdefaultencoding()`: 获取系统当前编码，一般默认为ascii。
- `sys.setdefaultencoding()`: 设置系统默认编码，执行dir（sys）时不会看到这个方法，在解释器中执行不通过，可以先执行reload(sys)，在执行 setdefaultencoding('utf8')，此时将系统默认编码设置为utf8。（见设置系统默认编码 ）
- `sys.getfilesystemencoding()`: 获取文件系统使用编码方式，Windows下返回'mbcs'，mac下返回'utf-8'.
- `sys.path`: 获取指定模块搜索路径的字符串集合，可以将写好的模块放在得到的某个路径下，就可以在程序中import时正确找到。
- `sys.platform`: 获取当前系统平台。
- `sys.stdin,sys.stdout,sys.stderr`: stdin , stdout ,  以及stderr 变量包含与标准I/O 流对应的流对象. 如果需要更好地控制输出,而print 不能满足你的要求, 它们就是你所需要的.  你也可以替换它们, 这时候你就可以重定向输出和输入到其它设备( device ), 或者以非标准的方式处理它们

#### sys.argv

功能：在外部向程序内部传递参数
 示例：`sys.py`

```python
#!/usr/bin/env python

import sys
print sys.argv[0]
print sys.argv[1]
```

运行：

```python
# python sys.py argv1
sys.py
argv1
```

自己动手尝试一下，领悟参数对应关系

#### sys.exit(n)

功能：执行到主程序末尾，解释器自动退出，但是如果需要中途退出程序，可以调用sys.exit函数，带有一个可选的整数参数返回给调用它的程序，表示你可以在主程序中捕获对sys.exit的调用。（0是正常退出，其他为异常）

示例：`exit.py`

```python
#!/usr/bin/env python

import sys

def exitfunc(value):
	print value
	sys.exit(0)

print "hello"

try:
	sys.exit(1)
except SystemExit,value:
	exitfunc(value)

print "come?"
```

运行：

```python
# python exit.py
hello
1
```

#### sys.path

功能：获取指定模块搜索路径的字符串集合，可以将写好的模块放在得到的某个路径下，就可以在程序中import时正确找到。

示例：

```python
>>> import sys
>>> sys.path
['', '/usr/lib/python2.7', '/usr/lib/python2.7/plat-x86_64-linux-gnu', '/usr/lib/python2.7/lib-tk', '/usr/lib/python2.7/lib-old', '/usr/lib/python2.7/lib-dynload', '/usr/local/lib/python2.7/dist-packages', '/usr/lib/python2.7/dist-packages', '/usr/lib/python2.7/dist-packages/PILcompat', '/usr/lib/python2.7/dist-packages/gtk-2.0', '/usr/lib/python2.7/dist-packages/ubuntu-sso-client']
sys.path.append("自定义模块路径")
```

#### sys.modules

功能：`sys.modules`是一个全局字典，该字典是python启动后就加载在内存中。每当程序员导入新的模块，`sys.modules`将自动记录该模块。当第二次再导入该模块时，python会直接到字典中查找，从而加快了程序运行的速度。它拥有字典所拥有的一切方法。

示例：`modules.py`

```python
#!/usr/bin/env python

import sys

print sys.modules.keys()

print sys.modules.values()

print sys.modules["os"]
```

运行：

```python
python modules.py
['copy_reg', 'sre_compile', '_sre', 'encodings', 'site', '__builtin__',......
```

#### sys.stdin\stdout\stderr

功能：stdin , stdout , 以及stderr 变量包含与标准I/O 流对应的流对象. 如果需要更好地控制输出,而print  不能满足你的要求, 它们就是你所需要的. 你也可以替换它们, 这时候你就可以重定向输出和输入到其它设备( device ),  或者以非标准的方式处理它们

勿忘初心，放得始终 (ง •_•)ง！

### os模块

```python
import os
os.sep:取代操作系统特定的路径分隔符
os.name:指示你正在使用的工作平台。比如对于Windows，它是'nt'，而对于Linux/Unix用户，它是'posix'。
os.getcwd:得到当前工作目录，即当前python脚本工作的目录路径。
os.getenv()和os.putenv:分别用来读取和设置环境变量
os.listdir():返回指定目录下的所有文件和目录名
os.remove(file):删除一个文件
os.stat（file）:获得文件属性
os.chmod(file):修改文件权限和时间戳
os.mkdir(name):创建目录
os.rmdir(name):删除目录
os.removedirs（r“c：\python”）:删除多个目录
os.system():运行shell命令
os.exit():终止当前进程
os.linesep:给出当前平台的行终止符。例如，Windows使用'\r\n'，Linux使用'\n'而Mac使用'\r'
os.path.split():返回一个路径的目录名和文件名
os.path.isfile()和os.path.isdir()分别检验给出的路径是一个目录还是文件
os.path.existe():检验给出的路径是否真的存在
os.listdir(dirname):列出dirname下的目录和文件
os.getcwd():获得当前工作目录
os.curdir:返回当前目录（'.'）
os.chdir(dirname):改变工作目录到dirname
os.path.isdir(name):判断name是不是目录，不是目录就返回false
os.path.isfile(name):判断name这个文件是否存在，不存在返回false
os.path.exists(name):判断是否存在文件或目录name
os.path.getsize(name):或得文件大小，如果name是目录返回0L
os.path.abspath(name):获得绝对路径
os.path.isabs():判断是否为绝对路径
os.path.normpath(path):规范path字符串形式
os.path.split(name):分割文件名与目录（事实上，如果你完全使用目录，它也会将最后一个目录作为文件名而分离，同时它不会判断文件或目录是否存在）
os.path.splitext():分离文件名和扩展名
os.path.join(path,name):连接目录与文件名或目录
os.path.basename(path):返回文件名
os.path.dirname(path):返回文件路径
```

文件操作：

```python
os.mknod("text.txt")：创建空文件
fp = open("text.txt",w):直接打开一个文件，如果文件不存在就创建文件
```

open 模式：

```
w 写方式
a 追加模式打开（从EOF开始，必要时创建新文件）
r+ 以读写模式打开
w+ 以读写模式打开
a+ 以读写模式打开
rb 以二进制读模式打开
wb 以二进制写模式打开 (参见 w )
ab 以二进制追加模式打开 (参见 a )
rb+ 以二进制读写模式打开 (参见 r+ )
wb+ 以二进制读写模式打开 (参见 w+ )
ab+ 以二进制读写模式打开 (参见 a+ )
```

```python
fp.read([size])  #size为读取的长度，以byte为单位
 
fp.readline([size])  #读一行，如果定义了size，有可能返回的只是一行的一部分
 
fp.readlines([size])  #把文件每一行作为一个list的一个成员，并返回这个list。其实它的内部是通过循环调用readline()来实现的。如果提供size参数，size是表示读取内容的总长，也就是说可能只读到文件的一部分。
 
fp.write(str)  #把str写到文件中，write()并不会在str后加上一个换行符
 
fp.writelines(seq)  #把seq的内容全部写到文件中(多行一次性写入)。这个函数也只是忠实地写入，不会在每行后面加上任何东西。
 
fp.close()  #关闭文件。python会在一个文件不用后自动关闭文件，不过这一功能没有保证，最好还是养成自己关闭的习惯。 如果一个文件在关闭后还对其进行操作会产生ValueError
 
fp.flush()  #把缓冲区的内容写入硬盘
 
fp.fileno()  #返回一个长整型的”文件标签“
 
fp.isatty()  #文件是否是一个终端设备文件（unix系统中的）
 
fp.tell()  #返回文件操作标记的当前位置，以文件的开头为原点
 
fp.next()  #返回下一行，并将文件操作标记位移到下一行。把一个file用于for … in file这样的语句时，就是调用next()函数来实现遍历的。
 
fp.seek(offset[,whence])  #将文件打操作标记移到offset的位置。这个offset一般是相对于文件的开头来计算的，一般为正数。但如果提供了whence参数就不一定了，whence可以为0表示从头开始计算，1表示以当前位置为原点计算。2表示以文件末尾为原点进行计算。需要注意，如果文件以a或a+的模式打开，每次进行写操作时，文件操作标记会自动返回到文件末尾。
 
fp.truncate([size])  #把文件裁成规定的大小，默认的是裁到当前文件操作标记的位置。如果size比文件的大小还要大，依据系统的不同可能是不改变文件，也可能是用0把文件补到相应的大小，也可能是以一些随机的内容加上去。
 
目录操作
 
os.mkdir("file")　　创建目录
 
shutil.copyfile("oldfile","newfile")　　复制文件:oldfile和newfile都只能是文件
 
shutil.copy("oldfile","newfile")  oldfile只能是文件夹，newfile可以是文件，也可以是目标目录
 
shutil.copytree("olddir","newdir")  复制文件夹.olddir和newdir都只能是目录，且newdir必须不存在
 
os.rename("oldname","newname")  重命名文件（目录）.文件或目录都是使用这条命令
 
shutil.move("oldpos","newpos")  移动文件（目录）
 
os.rmdir("dir")  只能删除空目录
 
shutil.rmtree("dir")  空目录、有内容的目录都可以删
 
os.chdir("path")  转换目录，换路径
```

### 集合、堆和双端队列(heap模块、deque类)

集合(set)、堆(heap)、双端队列(deque)

#### 堆导入 heapq 模块、双端队列导入collections模块。

| 函数                    | 描述                                                         |
| ----------------------- | ------------------------------------------------------------ |
| heappush(heap,value)    | 将value加入堆                                                |
| heappop(heap)           | 将堆中的最小值弹出，并返回该最小值                           |
| heapify(heap)           | 将列表转换为堆，也就是重新安排列表中元素的顺序               |
| heapreplace(heap,value) | 将堆中的最小值弹出，并同时将value入堆                        |
| nlargest(n,iter)        | 返回可迭代对象中前n个最大值，以列表形式返回                  |
| nsmallest(n,iter)       | 返回可迭代对象中前n个最小值，以列表形式返回                  |
| merge(*iter,key)        | 合并多个有序的迭代对象，如果指定key，则对每个元素的排序会利用key指定的函数 |

```python
from collections import deque
```

collections 是 python 内建的一个集合模块，里面封装了许多集合类，其中队列相关的集合只有一个：deque。
deque 是双边队列（double-ended queue），具有队列和栈的性质，在 list 的基础上增加了移动、旋转和增删等。

```python
d = collections.deque([])
d.append('a') # 在最右边添加一个元素，此时 d=deque('a')
d.appendleft('b') # 在最左边添加一个元素，此时 d=deque(['b', 'a'])
d.extend(['c','d']) # 在最右边添加所有元素，此时 d=deque(['b', 'a', 'c', 'd'])
d.extendleft(['e','f']) # 在最左边添加所有元素，此时 d=deque(['f', 'e', 'b', 'a', 'c', 'd'])
d.pop() # 将最右边的元素取出，返回 'd'，此时 d=deque(['f', 'e', 'b', 'a', 'c'])
d.popleft() # 将最左边的元素取出，返回 'f'，此时 d=deque(['e', 'b', 'a', 'c'])
d.rotate(-2) # 向左旋转两个位置（正数则向右旋转），此时 d=deque(['a', 'c', 'e', 'b'])
d.count('a') # 队列中'a'的个数，返回 1
d.remove('c') # 从队列中将'c'删除，此时 d=deque(['a', 'e', 'b'])
d.reverse() # 将队列倒序，此时 d=deque(['b', 'e', 'a'])
```

使用deque解决约瑟夫问题：

```python
""" 约瑟夫算法
据说著名犹太历史学家 Josephus 有过以下的故事：
在罗马人占领桥塔帕特后，39个犹太人与 Josephus 及他的朋友躲到一个洞中，
39个犹太人决定宁愿死也不要被敌人抓到，于是决定了一个自杀方式，41个人排成一个圆圈，
由第1个人开始报数，每报数到第3人该人就必须自杀，然后再由下一个重新报数，
直到所有人都自杀身亡为止。然而 Josephus 和他的朋友并不想自杀，
问他俩安排的哪两个位置可以逃过这场死亡游戏？
"""
import collections
def ysf(a, b):
    d = collections.deque(range(1, a+1)) # 将每个人依次编号，放入到队列中
    while d:
        d.rotate(-b) # 队列向左旋转b步
        print(d.pop()) # 将最右边的删除，即自杀的人

if __name__ == '__main__':
    ysf(41,3) # 输出的是自杀的顺序。最后两个是16和31，说明这两个位置可以保证他俩的安全。
```

###  time模块

time 模块主要包含各种提供日期、时间功能的类和函数。该模块既提供了把日期、时间格式化为字符串的功能，也提供了从字符串恢复日期、时间的功能。

在 Python 的交互式解释器中先导入 time 模块，然后输入 [e for e in dir(time) if not e.startswith('_')] 命令，即可看到该模块所包含的全部属性和函数：

```python
>>> [e for e in dir(time) if not e.startswith('_')]
['altzone', 'asctime', 'clock', 'ctime', 'daylight', 'get_clock_info', 'gmtime', 'localtime', 'mktime', 
'monotonic',
 'perf_counter', 'process_time', 'sleep', 'strftime', 'strptime', 'struct_time', 'time', 'timezone', 'tzname']
```

在 time 模块内提供了一个 time.struct_time 类，该类代表一个时间对象，它主要包含 9 个属性，每个属性的信息如下表所示：

| 字段名   | 字段含义     | 值                      |
| -------- | ------------ | ----------------------- |
| tm_year  | 年           | 如 2017、2018 等        |
| tm_mon   | 月           | 如 2、3 等，范围为 1~12 |
| tm_mday  | 日           | 如 2、3 等，范围为 1~31 |
| tm_hour  | 时           | 如 2、3 等，范围为 0~23 |
| tm_min   | 分           | 如 2、3 等，范围为 0~59 |
| tm_sec   | 秒           | 如 2、3 等，范围为 0~59 |
| tm_wday  | 周           | 周一为 0，范围为 0~6    |
| tm_yday  | 一年内第几天 | 如 65，范围 1~366       |
| tm_isdst | 夏时令       | 0、1 或 -1              |

比如，Python 可以用 time.struct_time(tm_year=2018, tm_mon=5, tm_mday=2, tm_hour=8,  tm_min=0, tm_sec=30, tm_wday=3, tm_yday=1, tm_isdst=0) 很清晰地代表时间。

此外，Python 还可以用一个包含 9 个元素的元组来代表时间，该元组的 9 个元素和 struct_time 对象中 9 个属性的含义是一一对应的。比如程序可以使用（2018, 5, 2, 8, 0, 30, 3, 1, 0）来代表时间。

在日期、时间模块内常用的功能函数如下：

time.asctime([t])：将时间元组或 struct_time 转换为时间字符串。如果不指定参数 t，则默认转换当前时间。

time.ctime([secs])：将以秒数代表的时间转换为时间宇符串。

time.gmtime([secs])：将以秒数代表的时间转换为 struct_time 对象。如果不传入参数，则使用当前时间。

time.localtime([secs])：将以秒数代表的时间转换为代表当前时间的 struct_time 对象。如果不传入参数，则使用当前时间。

time.mktime(t)：它是 localtime 的反转函数，用于将 struct_time 对象或元组代表的时间转换为从 1970 年 1 月 1 日 0 点整到现在过了多少秒。

time.perf_counter()：返回性能计数器的值。以秒为单位。

time.process_time()：返回当前进程使用 CPU 的时间。以秒为单位。

time.sleep(secs)：暂停 secs 秒，什么都不干。

time.strftime(format[, t])：将时间元组或 struct_time 对象格式化为指定格式的时间字符串。如果不指定参数 t，则默认转换当前时间。

time.strptime(string[, format])：将字符串格式的时间解析成 struct_time 对象。

time.time()：返回从 1970 年 1 月 1 日 0 点整到现在过了多少秒。

time.timezone：返回本地时区的时间偏移，以秒为单位。

time.tzname：返回本地时区的名字。

下面程序示范了 time 棋块的功能函数：

```python
import time
# 将当前时间转换为时间字符串
print(time.asctime())
# 将指定时间转换时间字符串，时间元组的后面3个元素没有设置
print(time.asctime((2018, 2, 4, 11, 8, 23, 0, 0 ,0))) # Mon Feb  4 11:08:23 2018
# 将以秒数为代表的时间转换为时间字符串
print(time.ctime(30)) # Thu Jan  1 08:00:30 1970
# 将以秒数为代表的时间转换为struct_time对象。
print(time.gmtime(30))
# 将当前时间转换为struct_time对象。
print(time.gmtime())
# 将以秒数为代表的时间转换为代表当前时间的struct_time对象
print(time.localtime(30))
# 将元组格式的时间转换为秒数代表的时间
print(time.mktime((2018, 2, 4, 11, 8, 23, 0, 0 ,0))) # 1517713703.0
# 返回性能计数器的值
print(time.perf_counter())
# 返回当前进程使用CPU的时间
print(time.process_time())
#time.sleep(10)
# 将当前时间转换为指定格式的字符串
print(time.strftime('%Y-%m-%d %H:%M:%S'))
st = '2018年3月20日'
# 将指定时间字符串恢复成struct_time对象。
print(time.strptime(st, '%Y年%m月%d日'))
# 返回从1970年1970年1月1日0点整到现在过了多少秒。
print(time.time())
# 返回本地时区的时间偏移，以秒为单位
print(time.timezone) # 在国内东八区输出-28800
```

运行上面程序，可以看到如下输出结果：

```python
Fri Feb 22 11:28:39 2019
Mon Feb  4 11:08:23 2018
Thu Jan  1 08:00:30 1970
time.struct_time(tm_year=1970, tm_mon=1, tm_mday=1, tm_hour=0, tm_min=0, tm_sec=30, tm_wday=3, tm_yday=1, 
tm_isdst=0)
time.struct_time(tm_year=2019, tm_mon=2, tm_mday=22, tm_hour=3, tm_min=28, tm_sec=39, tm_wday=4, tm_yday=53, 
tm_isdst=0)
time.struct_time(tm_year=1970, tm_mon=1, tm_mday=1, tm_hour=8, tm_min=0, tm_sec=30, tm_wday=3, tm_yday=1, 
tm_isdst=0)
1517713703.0
0.0
0.140625
2019-02-22 11:28:39
time.struct_time(tm_year=2018, tm_mon=3, tm_mday=20, tm_hour=0, tm_min=0, tm_sec=0, tm_wday=1, tm_yday=79, 
tm_isdst=-1)
1550806119.4960592
-28800
```

time 模块中的 strftime() 和 strptime() 两个函数互为逆函数，其中 strftime() 用于将 struct_time 对象或时间元组转换为时间字符串；而 strptime() 函数用于将时间字符串转换为 struct_time  对象。这两个函数都涉及编写格式模板，比如上面程序中使用 %Y 代表年、%m 代表月、%d 代表日、%H 代表时、%M 代表分、%S  代表秒。这两个函数所需要的时间格式字符串支持的指令如下表所示：

| 指 令 | 含义                                                         |
| ----- | ------------------------------------------------------------ |
| %a    | 本地化的星期几的缩写名，比如 Sun 代表星期天                  |
| %A    | 本地化的星期几的完整名                                       |
| %b    | 本地化的月份的缩写名，比如 Jan 代表一月                      |
| %B    | 本地化的月份的完整名                                         |
| %c    | 本地化的日期和时间的表示形式                                 |
| %d    | 代表一个月中第几天的数值，范固： 01~31                       |
| %H    | 代表 24 小时制的小时，范围：00~23                            |
| %I    | 代表 12 小时制的小时，范围：01~12                            |
| %j    | 一年中第几天，范围：001~366                                  |
| %m    | 代表月份的数值，范围：01~12                                  |
| %M    | 代表分钟的数值，范围：00~59                                  |
| %p    | 上午或下午的本地化方式。当使用 strptime() 函数并使用 %I 指令解析小时时，%p 只影响小时字段 |
| %S    | 代表分钟的数值，范围：00~61。该范围确实是 00~61，60 在表示闰秒的时间戳时有效，而 61 则是由于一些历史原因造成的 |
| %U    | 代表一年中表示第几周，以星期天为每周的第一天，范围：00~53。在这种方式下，一年中第一个星期天被认为处于第一周。当使用 strptime() 函数解析时间字符串时，只有同时指定了星期几和年份该指令才会有效 |
| %w    | 代表星期几的数值，范围：0~6，其中 0 代表周日                 |
| %W    | 代表一年小第几周，以星期一为每周的第一天，范围：00~53。在这种方式下，一年中第一个星期一被认为处于第一周。当使用 strptime() 函数解析时间字符串时，只有同时指定了星期几和年份该指令才会有效 |
| %x    | 本地化的日期的表示形式                                       |
| %X    | 本地化的时间的表示形式                                       |
| %y    | 年份的缩写，范围：00~99，比如 2018 年就简写成 18             |
| %Y    | 年份的完整形式。如 2018                                      |
| %z    | 显示时区偏移                                                 |
| %Z    | 时区名（如果时区不行在，则显示为空）                         |
| %%    | 用于代表%符号                                                |

### random 模块

 ython中的random模块用于生成随机数。

下面具体介绍random模块的功能：

1.random.random()

 \#用于生成一个0到1的

随机浮点数：0<= n < 1.0

```python
1 import random  
2 a = random.random()
3 print (a)  
```

2.random.uniform(a,b) 

\#用于生成一个指定范围内的随机符点数，两个参数其中一个是上限，一个是下限。如果a > b，则生成的随机数n: b <= n <= a。如果 a <b， 则 a <= n <= b。

```python
1 import random  
2 print(random.uniform(1,10))  
3 print(random.uniform(10,1)) 
```

3.random.randint(a, b)

 \#用于生成一个指定范围内的整数。其中参数a是下限，参数b是上限，生成的随机数n: a <= n <= b

```python
1 import random  
2 print(random.randint(1,10))  
```

4.random.randrange([start], stop[, step])

 \#从指定范围内，按指定基数递增的集合中 获取一个随机数。

random.randrange(10, 30, 2)，结果相当于从[10, 12, 14, 16, ... 26, 28]序列中获取一个随机数。

random.randrange(10, 30, 2)在结果上与 random.choice(range(10, 30, 2) 等效。

```python
1 import random  
2 print(random.randrange(10,30,2))
```

 5.random.choice(sequence)

\#random.choice从序列中获取一个随机元素。其函数原型为：random.choice(sequence)。

参数sequence表示一个有序类型。这里要说明 一下：sequence在python不是一种特定的类型，而是泛指一系列的类型。list, tuple, 字符串都属于sequence。

```python
1 import random  
2 lst = ['python','C','C++','javascript']  
3 str1 = ('I love python')  
4 print(random.choice(lst))
5 print(random.choice(str1))  
```

6.random.shuffle(x[, random])

\#用于将一个列表中的元素打乱,即将列表内的元素随机排列。

```python
1 import random
2 p = ['A' , 'B', 'C', 'D', 'E' ]
3 random.shuffle(p)  
4 print (p)  
```

7.random.sample(sequence, k)

\#从指定序列中随机获取指定长度的片断并随机排列。注意：sample函数不会修改原有序列。

```python
1 import random   
2 lst = [1,2,3,4,5]  
3 print(random.sample(lst,4))  
4 print(lst) 
```



 