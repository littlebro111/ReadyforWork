# extern关键字的作用：

1. 利用关键字extern，如：`extern int a;`，可以在一个文件中引用另一个文件中定义的变量或者函数。
2. 当extern与 “C” 一起连用时，如：`extern “C” void fun(int a);`，则编译器在编译fun这个函数名时会按C的规则去翻译相应的函数名而不是C++的。

**要调用其它文件中的函数和变量，只需把该文件用#include包含进来即可，为啥要用extern？**

因为⽤ extern 会加速程序的编译过程，这样能节省时间。而且，被包含的文件中的所有的变量和方法都可以被这个文件使用，这样就变得不安全，如果只是希望一个文件使用另一个文件中的某个变量还是使用extern关键字更好。

**C++编译时和C有什么不同？**

由于C++支持函数重载，因此编译器编译函数的过程中会将函数的参数类型也加到编译后的代码中，而不仅仅是函数名；而C语言并不支持函数重载，因此编译C语言代码的函数时不会带上函数的参数类型，一般只包括函数名。

**举例：**

```c++
// a.c
#include<stdio.h>
int global_x = 10;
int func()
{
	printf("call func\n");
}

// b.c
#include<stdio.h>
extern int global_x;
int main()
{
    extern void func();
	func();
    printf("%d", global_x);
	return 0;
}
```

> 注意：extern声明的位置对其作用域也有关系 ，如果是在main函数中进行声明的，则只能在main函数中调用，在其它函数中不能调用。

# extern和stastic

extern表明该变量在别的地方已经定义过了,在这里要使用那个变量；
static表示静态的变量，分配内存的时候, 存储在静态区,不存储在栈上面；

首先，static与extern是一对“水火不容”的家伙，也就是说extern和static不能同时修饰一个变量；
其次，static修饰的全局变量声明与定义同时进行，也就是说当你在头文件中使用static声明了全局变量后，它也同时被定义了；
最后，static修饰全局变量的作用域只能是本身的编译单元，也就是说它的“全局”只对本编译单元有效，其他编译单元则看不到它,例如：

```c++
// test1.h
#ifndef TEST1H
#define TEST1H
#include "iostream"
static char g_str[] = "123456";
void fun1();
#endif

// test1.cpp
#include "test1.h"
void fun1()  { cout << g_str << endl; }

// test2.cpp
#include "test1.h"
void fun2()  { cout << g_str << endl; }
```

以上两个编译单元可以连接成功，打开test1.obj和test2.obj，在两者中都可以找到字符串"123456"。

它们之所以可以连接成功而没有报重复定义的错误是因为虽然它们有相同的内容，但是存储的物理地址并不一样，就像是两个不同变量赋了相同的值一样，而这两个变量分别作用于它们各自的编译单元。

```c++
// test1.cpp
#include "test1.h"
void fun1()
{
    g_str[0] = ''a'';
    cout << g_str << endl;
}

// test2.cpp
#include "test1.h"
void fun2()  { cout << g_str << endl; }
void main() {
    fun1(); // a23456
    fun2(); // 123456
}
```

这个时候再跟踪代码时，就会发现两个编译单元中的g_str地址并不相同，因为只在其中一处修改了g_str，内存中存在了两份拷贝给两个模块中的变量使用。

# extern和const

1. C++中const修饰的全局常量具有跟static相同的特性，即它们只能作用于本编译模块中；
2. 但是const可以与extern连用来声明该常量可以作用于其他编译模块中, 如`extern const char g_str[];`

所以当const单独使用时其特性就与static相同，而当与extern一起使用的时候，它的特性就跟extern的一样了。