# RAII机制

RAII是Resource Acquisition Is Initialization（即“资源获取就是初始化”）的简称，是C++语言的一种管理资源、避免泄漏的惯用法。利用的就是C++构造的对象最终会被销毁的原则。RAII的做法是使用一个对象，在其构造时获取对应的资源，在对象生命期内控制对资源的访问，使之始终保持有效，最后在对象析构的时候，释放构造时获取的资源。

RAII是用来管理资源、避免资源泄漏的方法。那么资源是如何定义的？在计算机系统中，资源是数量有限且对系统正常运行具有一定作用的元素。比如：网络套接字、互斥锁、文件句柄和内存等等，它们属于系统资源。由于系统的资源是有限的，就好比自然界的石油，铁矿一样，不是取之不尽，用之不竭的，所以，我们在编程使用系统资源时，都必须遵循一个步骤：

1. 申请资源；
2. 使用资源；
3. 释放资源。

第一步和第三步缺一不可，因为资源必须要申请才能使用的，使用完成以后，必须要释放，如果不释放的话，就会造成资源泄漏。

但是如果程序很复杂的时候，需要为所有的 new 分配的内存 delete 掉，导致极度臃肿，效率下降，更可怕的是，程序的可理解性和可维护性明显降低了，当操作增多时，处理资源释放的代码就会越来越多，越来越乱。如果某一个操作发生了异常而导致释放资源的语句没有被调用，怎么办？这个时候，RAII 机制就可以派上用场了。

由于系统的资源不具有自动释放的功能，而C++中的类具有自动调用析构函数的功能。如果把资源用类进行封装起来，对资源操作都封装在类的内部，在析构函数中进行释放资源。当定义的局部变量的生命结束时，它的析构函数就会自动的被调用，如此，就不用程序员显示的去调用释放资源的操作了。

```c++
template<class... _Mutexes>
class lock_guard
{	// class with destructor that unlocks mutexes
public:
	explicit lock_guard(_Mutexes&... _Mtxes)
		: _MyMutexes(_Mtxes...)
		{	// construct and lock
			_STD lock(_Mtxes...);
		}
 
	lock_guard(_Mutexes&... _Mtxes, adopt_lock_t)
		: _MyMutexes(_Mtxes...)
		{	// construct but don't lock
		}
 
	~lock_guard() _NOEXCEPT
		{	// unlock all
		_For_each_tuple_element(
			_MyMutexes,
			[](auto& _Mutex) _NOEXCEPT { _Mutex.unlock(); });
		}
 
	lock_guard(const lock_guard&) = delete;
	lock_guard& operator=(const lock_guard&) = delete;
private:
	tuple<_Mutexes&...> _MyMutexes;
};
```

在使用多线程时，经常会涉及到共享数据的问题，C++ 中通过实例化 std::mutex 创建互斥量，通过调用成员函数 lock() 进行上锁，unlock() 进行解锁。不过这意味着必须记住在每个函数出口都要去调用 unlock()，也包括异常的情况，这非常麻烦，而且不易管理。C++ 标准库为互斥量提供了一个 RAII 语法的模板类 std::lock_guard，其会在构造函数的时候提供已锁的互斥量，并在析构的时候进行解锁，从而保证了一个已锁的互斥量总是会被正确的解锁。上面的代码正是 \<mutex\> 头文件中的源码，其中还使用到很多 C++11 的特性，比如 delete/noexcept 等，有兴趣的同学可以查一下。