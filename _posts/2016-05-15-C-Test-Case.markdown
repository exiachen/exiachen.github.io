---
layout: default
title:  "C语言知识小记"
categories: C
---

###C语言知识点小记

一套C语言题目的分析：[链接](http://www.nowamagic.net/librarys/veda/detail/775)

对其中自己欠缺的C语言知识点进行学习和纪录。

####setjmp与longjmp的使用方法
setjmp和longjmp通过保存和恢复程序的执行环境，来实现更复杂的执行流控制（可以在函数之间进行跳转）。大致原理是在第一次调用setjmp时将执行环境的上下文（寄存器，setjmp函数的返回地址等）保存在jmp_buf格式的变量中，后续调用longjmp时，需指定一个jmp_buf格式的变量和一个返回值，调用后继而跳转到setjmp返回后的第一条指令进行执行，并且返回longjmp所指定的返回值。一个简单的例子：

```
#include <stdio.h>
#include <stdlib.h>
#include <setjmp.h>    

static jmp_buf env;

int main()
{
    int ret = 0;
    if ((ret = setjmp(env)))
    {
        printf("long jmp return, ret = %d\n", ret);
        exit(0);
    }
    else
    {
        printf("first time to call setjmp, ret = %d\n", ret);
    }


    printf("after setjmp\n");
    longjmp(env, 1);

    return 0;
}
```
执行结果为：

```
first time to call setjmp, ret = 0
after setjmp
long jmp return, ret = 1
```


####对数组名取地址所表示的含义
在C语言中数组名一般作为常量指针使用（指向数组第一个元素的常量指针），但有两种情况下不作为这种方式使用：

* 使用sizeof作用与数组名时，得到的结果是整个数组的大小。
* 对数组名取地址时得到的是***指向数组类型的指针***。

```
int a[4] = {1, 2, 3, 4};

int main()
{
    printf("sizeof(a) = %lu\n", sizeof(a));
    printf("a = %p\n", a);
    printf("&a = %p\n", &a);
    printf("a + 1 = %p\n", (a + 1));
    printf("&a + 1 = %p\n", (&a + 1));

    return 0;
}
```
得到的结果为：

```
sizeof(a) = 16
a = 0x102d76020
&a = 0x102d76020
a + 1 = 0x102d76024
&a + 1 = 0x102d76030
```

####逗号操作符（其优先级，与等号的优先级）
逗号优先级低于等号，所以像以下类似的操作：

```
int a = 1, b = 2, c = 3;

a = b, c;	// a = b

a = (b, c)	// a = c
```
较为详细的C语言运算符优先级及结合型总结见[这篇博客](http://c.biancheng.net/cpp/html/462.html)


####sizeof操作符，不会对参数中的表达式进行计算
```
int i = 0;
printf("size = %lu, i = %d\n", sizeof(i++ + ++i), i);
```

结果为：

```
size = 4, i = 0
```

sizeof仅关心表达式结果的类型，并不实际对表达式进行求值。
使用sizeof需要注意的点：

* sizeof对于void类型
网上看到sizeof不能对void类型使用，但实际测试了一下（分别用clang-703.0.29 和 gcc-4.6.3）`sizeof(void)`的结果是1，C++98和C++11标准文档中规定，对于基本类型求sizeof，只有char, unsigned char进行了规定，必须是1，其他的都是编译器决定。

* 当表达式作为sizeof的参数时，返回表达式结果的类型大小，不对表达式求值。
* 对函数调用sizeof时，返回函数返回值的类型大小，不进行函数调用。 

