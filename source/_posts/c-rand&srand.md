---
title: 每日学说话(●'◡'●)c-rand
---

# c--rand()与srand()

### 引子

​    相信大家对于rand()函数并不陌生，我们常用它来生成伪随机数，但是为什么有时候我们生成的随机数并不符合预期呢？或者说，为什么有时候我们生成的随机数并不随机？如何有效地生成伪随机数呢？

### rand()

    rand()函数是使用线性同余法做的，它并不是真的随机数，因为其周期特别长，所以在一定范围内可以看成随机的。
    rand()函数不需要参数，它将会返回0到RAND_MAX之间的任意的整数。如果我们想要生成一个在区间[0, 1]之内的数，那么我们可以写出如下代码：rand_num = rand()/RAND_MAX;

```c++
#include<iostream>
using namespace std;
int main() {
	for (int i = 0; i < 10; i++)
	{
		cout << rand() << endl;
	}
	return 0;
	system("pause");
}
```

### srand()

 srand()为初始化随机数发生器，用于设置rand()产生随机数时的种子。传入的参数seed为unsigned int类型，通常我们会使用time(0)或time(NULL)的返回值作为seed。下面我们来进行实验，从而对它进行更深入的感知。

```c++
#include<iostream>
using namespace std;
int main() {
	srand(1);
	for(int i = 0; i < 10; i++)
	{
    	cout << rand() << endl;
	}
	return 0;
	system("pause");
}
```

​	这与我们前面不使用srand()设置随机种子时结果一致，因此我们可以看出，如果我们不显示调用srand()的话，将默认为srand(1)。此外，从这次实验中可以看出，如果我们设置某个固定的seed，那么虽然在同一次运行时，会有不同的随机数产生，但是对于这段程序的多次运行所得到的结果是不变的。
​        那我们如何引入变化的种子呢？一般来说，我们会使用time(NULL)或time(0)来表示变化的种子，time(0)的返回的是从1970 UTC Jan 1 00:00到当前时刻的秒数，为unsigned int类型。当我们在不同时刻运行程序时，就会有不同的随机数种子，因此就可以得到不同的结果：

```c++
srand(time(0));
for(int i = 0; i < 10; i++)
{
    cout << rand() << endl;
}
```

​	值得注意的是，如果，我们两次程序运行之间的间隔小于1s，那么会出现下面这种情形（我们通过在代码中两次调用srand(time(0))来模仿这种情形）：

```c++
srand(time(0));
for(int i = 0; i < 10; i++)
{
    cout << rand() << endl;
}
cout << "--------------" << endl;
srand(time(0));
for(int i = 0; i < 10; i++)
{
    cout << rand() << endl;
}
```

两次运行结果一致，这是为什么呢？因为我们两次调用srand()函数设置随机数种子之间的时间间隔不超过1s，这会导致我们重置随机数种子，从而等价于使用了一个固定的随机数种子。我们可以用下面的代码来进行验证：

```c++
srand(time(0));
for(int i = 0; i < 10; i++)
{
    cout << rand() << endl;
}
cout << "--------------" << endl;
sleep(1.0);
srand(time(0));
for(int i = 0; i < 10; i++)
{
    cout << rand() << endl;
}
```

为什么要特意指出这一点？这是为了防止我们不小心将srand(time(0))放入了随机数生成循环中：

```c++
for(int i = 0; i < 10; i++)
{
    srand(time(0));
    cout << rand() << endl;
}
```

 如果，我们在其中引入sleep(1.0)，那么将没有问题：

```c++
for(int i = 0; i < 10; i++)
{
    srand(time(0));
    cout << rand() << endl;
    sleep(1.0);
}
```

！！！φ(゜▽゜*)♪~~~

