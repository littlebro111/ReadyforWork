# 标准模板库（STL）

**容器：**vector、deque、list、set、map

**适配器**：stack、queue

**迭代器**

**String类**

# vector的实现

Vector是一个类，它里面有三个指针myfirst,mylast,myend.分别表示首地址，元素容量地址，容器容量地址。通过这三个指针分别表示容器的所有操作。

**vector的扩容机制**

Vector扩容就是重新申请一段更长的连续内存空间并把以前的数据移动过去，释放以前的内存空间。以前的迭代器都会失效。一般用VS扩容都是扩容现有容器容量的2倍，有的是1.5倍。