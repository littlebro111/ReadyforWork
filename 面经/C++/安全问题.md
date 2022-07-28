# 野指针

野指针就是指尚未初始化的指针，既不指向合法的内存空间，也没有使用NULL/nullptr初始化指针。

举例如下：

```c++
#include <iostream>
using namespace std;
 
int main()
{
    int *p; // 野指针
    int *q = NULL; // 非野指针
    p = new int(5); // p现在不再是野指针
    q = new int(10); 
    free(p);
    free(q);
    return 0;
}
```

# 悬空指针

悬空指针是指指针指向的内存空间已被释放或不再有效。

一般有两种情况：

1. 指针释放后未置空

    ```c++
    #include <iostream>
    using namespace std;
     
    int main()
    {
        int *p = new int(5);
        cout << "p 地址：" << p << endl;
        free(p);  // p在释放后成为悬空指针
        cout << "p 地址：" << p << endl;
        p = NULL; // 非悬空指针
        return 0;
    }
    ```

2. 指针操作超越变量作用域

    ```c++
    #include <iostream>
    using namespace std;
    
    int* func()
    {
        int tmp = 10;
        return &tmp;
    }
    
    int main()
    {
        // 变量tmp的作用域为最近的一层括号内，在括号外引用便超出了变量的作用范围。
        // p在超出了变量的作用范围后就变为了悬空指针。
        int *p; // 野指针
        {
            int tmp = 10;
            p = &tmp;
        }
        
        // 在函数func执行完后，局部变量的内存空间会被释放，而这里q指向了函数内的局部变量，q便成为了悬空指针
    	// 可以通过将tmp变为static来避免这种情况发生。
        int *q = func();
        return 0;
    }
    ```

# 内存泄漏

内存泄漏是指程序在申请内存后，无法释放已申请的内存空间。

一次内存泄漏可能不会有大的影响，但内存泄漏堆积后的后果就是内存溢出。

```c++
#include <stdio.h>
#include <stdlib.h>
 
int main()
{
    int *p = (int*)malloc(sizeof(int));
    *p = 10;
    printf("*p = %d\n", *p);
 
    p = (int*)malloc(sizeof(int));
    *p = 20;
    printf("*p = %d\n", *p);
    free(p);
    return 0;
}
```

**如何防止内存泄漏？**

1. 确保new和delete ，malloc/calloc和free成对出现；
2. 使用智能指针；

# 内存溢出

内存溢出指程序申请内存时，没有足够的内存供申请者使用，或者是，给了你一块存储int类型数据的存储空间，但是你却存储long类型的数据，那么结果就是内存不够用，此时就会报错“OutOfMemory”，即所谓的内存溢出。