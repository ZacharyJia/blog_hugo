---
title: Peterson算法中turn(will_wait)变量的作用
date: 2016-04-10
categories:
    - 开发
tags:
    - 算法
    - 多线程
---

最近在操作系统上学习了Peterson算法，其使用了3个变量就实现了纯软件的线程同步，即两个线程的标记变量`c1`, `c2`以及一个`turn`变量。
但是对于其为什么要使用第三个变量还不是很了解，今天特意研究了一下，终于发现了第三个变量的作用。

<!--more-->

## 0x0 Peterson算法的实现

以下给出Peterson算法在windows下的一种简单实现：
```
int c1 = 0, c2 = 0, will_wait;

DWORD WINAPI p1(LPVOID p)
{
	int counter = 0;
	int tmp1, tmp2, r;
	do{
		//加锁
		c1 = 1;
		will_wait = 1;
		while(c2 && (will_wait == 1));
		tmp1 = accnt1;
		tmp2 = accnt2;
		r = rand();
		accnt1 = tmp1 + r;
		accnt2 = tmp2 - r;
		counter++;
		
	}while(accnt1 + accnt2 == 0 && (c1 = 0) == 0);
	printf("%d\n", counter);
	return 0;
}

DWORD WINAPI p2(LPVOID p)
{


	int counter = 0;
	int tmp1, tmp2, r;
	do{
		//加锁
		c2 = 1;
		will_wait = 2;
		while(c1 && (will_wait == 2));
		
		tmp1 = accnt1;
		tmp2 = accnt2;
		r = rand();
		accnt1 = tmp1 + r;
		accnt2 = tmp2 - r;
		counter++;
		
	}while(accnt1 + accnt2 == 0 && (c2 = 0) == 0 );
	printf("%d\n", counter);
	return 0;
}


int main()
{
	system("pause");
	HANDLE hFirst = CreateThread(NULL, 0, p1, NULL, 0, NULL);
	HANDLE hSecond = CreateThread(NULL, 0, p2, NULL, 0, NULL);

	WaitForSingleObject(hFirst, INFINITE);
	WaitForSingleObject(hSecond, INFINITE);
	system("pause");
	return 0;
}
```
在该程序中，主程序开辟两个新的线程p1和p2，如果程序运行出现问题，则accnt1和accnt2的值相加不为0，则该线程退出。

## 0x0 简单的情况
从最简单的情况思考，当线程1首先进入临界区(Critical section)时，其值设为1，而当线程2准备进入临界区时，会判断c1是否为1，若为1，则会进入循环等待。当线程1退出后，c1=0，线程2得以进入临界区。
通过这种方式，实现了两个线程之间的同步。

那么，为什么还需要一个turn变量呢？

## 0x1 另一种情况
要理解turn变量的作用，需要考虑另外一种情况。

假设两个线程几乎同时加锁，则其在加锁时的代码可以认为是混杂执行的。c1,c2都被设为1，然后两个线程先后进入阻塞循环当中。此时，如果阻塞循环如下：
```
while(c2); //p1进程

while(c1); //p2进程
```
则两个进程都会进入阻塞状态，互相阻塞，无法继续进行。

## 0x2 引入turn（will_wait）变量
下面考虑存在turn变量的情况。
两个线程分别将c1,c2设为1，然后继续执行，都将turn设为自己对应的序号（1或2）。但是由于其执行顺序必然有先后，因此先赋值的会被覆盖掉，最终的值变为另一个线程的值。假设p1先执行
```
will_wait = 1;  // p1
```
然后p2执行：
```
will_wait = 2; // p2
```
最终，will_wait变量的值将变为2。

而当两个线程进入到阻塞循环当中去时，由于判断条件如下：
```
while(c2 && (will_wait == 1)); //p1
```
```
while(c1 && (will_wait == 2)); //p2
```
对于线程1来说，不满足循环条件（will_wait不等于1），则程序会继续运行，进入临界区。
而对于线程2来说，满足阻塞循环的条件，会阻塞在该循环中，直到p1释放锁。

# 0x3 实验验证
为了验证不使用turn变量时，程序会进入互相阻塞状态。可以将两个线程改为如下代码：
```
//p1代码，p2类似
DWORD WINAPI p1(LPVOID p)
{
    int counter = 0;
    int tmp1, tmp2, r;
    do{
        //加锁
        c1 = 1;
        while(c2);
        tmp1 = accnt1;
        tmp2 = accnt2;
        r = rand();
        accnt1 = tmp1 + r;
        accnt2 = tmp2 - r;
        counter++;
        printf("1: running\n");
    }while(accnt1 + accnt2 == 0 && (c1 = 0) == 0);
    printf("%d\n", counter);
    return 0;
}
```
可以看出，如果程序能够正常进入临界区，则会不断循环输出`1: running`。
另外，需要注意的是，Peterson算法只能在单核系统下生效，如果要在多核系统下生效，需要进行以下操作。
1. 首先在main函数的开始位置，添加一行`system("pause");`
2. 运行程序，程序会等待用户按任意键继续运行。
3. 在任务管理器中找到该程序，点右键（win10下需要转到详细信息页点右键），选择`设置相关性`，然后只选中一个CPU核心。
4. 回到编写的程序，按任意键继续运行。

程序运行结果如下：
![no_turn](http://qn-cdn.zacharyjia.me/no_turn.png)

而添加will_wait变量之后，程序的运行结果如下：
![with_turn](http://qn-cdn.zacharyjia.me/with_turn.png)

可以看出，在没有加turn变量时，线程1执行了两次之后，就进入了相互阻塞状态，两个线程都不继续运行。而加了turn变量之后，程序则会持续运行下去，不断输出，同时也没有出现同步问题导致进程被终止的情况。

# 0x4
由此证明，turn变量在Peterson算法中是为了保证程序不会出现相互阻塞而产生的，不可缺少。