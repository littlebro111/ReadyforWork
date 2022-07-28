# struct和class的区别

1. 从来源上看，C++中的struct是沿用了C中的struct并进行了扩充，现在可以拥有静态成员、成员函数、初始化列表、继承甚至是多态。

2. 用法上，struct一般用于描述一个数据结构集合，而class是对一个对象数据的封装；

3. struct中默认的访问控制权限是public的，而class中默认的访问控制权限是private的；

4. 在继承关系中，struct默认是公有继承，而class是私有继承；

    > struct可以继承class，同样class也可以继承struct。而默认是public继承还是private继承，取决于子类而不是基类，即默认的继承访问权限是看子类到底是用的struct还是class。

5. class关键字可以用于定义模板参数，就像typename，而struct不能用于定义模板参数，例如：

    ```
    template<typename T> // 可以把typename换成class
    int Func(const T& t) {
    	//TODO
    }
    ```

6. 在使用大括号进行初始化上：
    *  class和struct如果定义了构造函数的话，都不能用大括号进行初始化；
    * 如果没有定义构造函数，struct可以用大括号初始化；
    * 如果没有定义构造函数，且所有成员变量全是public的话，class可以用大括号初始化；