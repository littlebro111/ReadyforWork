# sort函数

使用：

```c++
#include <algorithm>
using namespace std;
```

作用：排序

时间复杂度：$O(nlog{n})$

空间复杂度：$O(log{n})$

实现原理：其实STL中的sort()并非只是普通的快速排序，除了对普通的快速排序进行优化，它还结合了**插入排序**和**堆排序**。根据不同的数量级别以及不同情况，能自动选用合适的排序方法，这并不是说它每次排序只选择一种方法，它是在一次完整排序中针对不同的情况选用不同方法。当数据量较大时采用快速排序，分段递归。一旦分段后的数据量小于某个阀值，为避免递归调用带来过大的额外负荷，便会改用插入排序。而如果递归层次过深，有出现最坏情况的倾向，还会改用堆排序。

# 代码分析

下面是完整的`SGI STL sort()`源码（使用默认`<`操作符版）

```c++
template <class _RandomAccessIter>
inline void sort(_RandomAccessIter __first, _RandomAccessIter __last) {
  __STL_REQUIRES(_RandomAccessIter, _Mutable_RandomAccessIterator);
  __STL_REQUIRES(typename iterator_traits<_RandomAccessIter>::value_type,
                 _LessThanComparable);
  if (__first != __last) {
    __introsort_loop(__first, __last,
                     __VALUE_TYPE(__first),
                     __lg(__last - __first) * 2);
    __final_insertion_sort(__first, __last);
  }
}
```

其中，`__introsort_loop`便是上面介绍的内省式排序，其第三个参数中所调用的函数`__lg()`便是用来控制分割恶化情况，代码如下：

```c++
template <class Size>
inline Size __lg(Size n) {
	Size k;
	for (k = 0; n > 1; n >>= 1) ++k;
	return k;
}
```

即求`lg(n)`（取下整），意味着快速排序的递归调用最多 2*lg(n) 层。

内省式排序算法如下：

```cpp
template <class _RandomAccessIter, class _Tp, class _Size>
void __introsort_loop(_RandomAccessIter __first,
                      _RandomAccessIter __last, _Tp*,
                      _Size __depth_limit)
{
  while (__last - __first > __stl_threshold) {
    if (__depth_limit == 0) {
      partial_sort(__first, __last, __last);
      return;
    }
    --__depth_limit;
    _RandomAccessIter __cut =
      __unguarded_partition(__first, __last,
                            _Tp(__median(*__first,
                                         *(__first + (__last - __first)/2),
                                         *(__last - 1))));
    __introsort_loop(__cut, __last, (_Tp*) 0, __depth_limit);
    __last = __cut;
  }
}
```

1. 首先判断元素规模是否大于阀值`__stl_threshold`，`__stl_threshold`是一个常整形的全局变量，值为16，表示若元素规模小于等于16，则结束内省式排序算法，返回`sort`函数，改用插入排序。
2. 若元素规模大于`__stl_threshold`，则判断递归调用深度是否超过限制。若已经到达最大限制层次的递归调用，则改用堆排序。代码中的`partial_sort`即用堆排序实现。
3. 若没有超过递归调用深度，则调用函数`__unguarded_partition()`对当前元素做一趟快速排序，并返回枢轴位置。`__unguarded_partition()`函数采用的便是上面所讲的使用两个迭代器的方法，代码如下：

```cpp
template <class _RandomAccessIter, class _Tp>
_RandomAccessIter __unguarded_partition(_RandomAccessIter __first, 
                                        _RandomAccessIter __last, 
                                        _Tp __pivot) 
{
    while (true) {
        while (*__first < __pivot)
            ++__first;
        --__last;
        while (__pivot < *__last)
            --__last;
        if (!(__first < __last))
            return __first;
        iter_swap(__first, __last);
        ++__first;
    }
}
```

4. 经过一趟快速排序后，再递归对右半部分调用内省式排序算法。然后回到while循环，对左半部分进行排序。源码写法和我们一般的写法不同，但原理是一样的，需要注意。

递归上述过程，直到元素规模小于`__stl_threshold`，然后返回`sort`函数，对整个元素序列调用一次插入排序，此时序列中的元素已基本有序，所以插入排序也很快。至此，整个`sort`函数运行结束。