# new和malloc的区别

|      malloc（差）       |               new（好）               |
| :---------------------: | :-----------------------------------: |
|           堆            |    自由存储区（堆/静态存储区/空）     |
|    返回void*,需强转     |           返回对象类型指针            |
|    需要指定内存大小     |           无需指定内存大小            |
|  分配内存失败返回NULL   |         失败抛出bad_alloc异常         |
|  内存不够可以重新分配   |            不可以重新分配             |
| 不调用构造函数/析构函数 | （new一些自定义类时）调用构造析构函数 |
| 不能初始化数组元素对象  |          初始化数组元素对象           |

联系：new的底层实现可以调用malloc，并且可以实现重载。



使用new操作符来分配对象内存时会经历三个步骤：

* 第一步：调用operator new 函数（对于数组是operator new[]）分配一块足够大的，原始的，未命名的内存空间以便存储特定类型的对象。

- 第二步：编译器运行相应的构造函数以构造对象，并为其传入初值。
- 第三步：对象构造完成后，返回一个指向该对象的指针。

使用delete操作符来释放对象内存时会经历两个步骤：

- 第一步：调用对象的析构函数。
- 第二步：编译器调用operator delete(或operator delete[])函数释放内存空间。

总之来说，new/delete会调用对象的构造函数/析构函数以完成对象的构造/析构，而malloc则不会。



# 关于placement new

operator new有三种形式：

```c++
throwing (1)
void* operator new (std::size_t size) throw (std::bad_alloc);
nothrow (2) 
void* operator new (std::size_t size, const std::nothrow_t& nothrow_value) throw();
placement (3)
void* operator new (std::size_t size, void* ptr) throw();
```

(1)(2)的区别仅是是否抛出异常，当分配失败时，前者会抛出bad_alloc异常，后者返回null，不会抛出异常。它们都分配一个固定大小的连续内存。

```
A* a = new A; //调用throwing(1)
A* a = new(std::nothrow) A; //调用nothrow(2)
```

（3）是placement new，它也是对operator new的一个重载，定义于#include <new>中，它多接收一个ptr参数，但它只是简单地返回ptr。