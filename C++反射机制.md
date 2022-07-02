# 一. 前言

Java有着一个非常突出的动态相关机制：Reflection，用在Java身上指的是我们可以于运行时加载、探知、使用编译期间完全未知的classes。换句话说，Java程序可以加载一个运行时才得知名称的class，获悉其完整构造（但不包括methods定义），并生成其对象实体、或对其fields设值、或唤起其methods，然而C++是不支持反射机制。

在 Java 编程中，会经常用到反射，但是C++ 是不支持通过类名称字符串 “ClassXX” 来生成对象的，也就是说我们可以使用`ClassXX* object = new ClassXX;` 来生成对象，但是不能通过`ClassXX* object = new "ClassXX"; `来生成对象。

虽然C++没有自带的语法可以实现，但是我们可以自己通过其他方法来实现类似于`ClassT* obj = FactoryCreate(“ClassT”);`的语法。

# 二. 实现

## 1. 简单工厂模式实现

首先创建一个所有需要实现反射机制的类都需要继承的基类。

```c++
class Object 
{ 
public: 
	virtual string ToString() = 0; 
}; 
```

然后派生出来的类只需要再实现这个ToString即可。

```c++
class MyClass: public Object 
{ 
public: 
	virtual string ToString(){ return "MyClass"; } 
}; 
```

接着就是用于产生对象的工厂。

```c++
Object* FactoryCreate(const string& className) 
{ 
	if (className == "MyClass") 
		return new MyClass; 
	else if (className == "ClassA") 
		return new ClassA; 
	else if(className == "ClassB") 
		return new ClassB; 
	else if(className == "ClassC") 
		return new ClassC; 
	else if(className == "ClassD") 
		return new ClassD; 
}
```

于是我们就可以这样使用：

```c++
int main() 
{ 
	Object* obj = FactoryCreate("MyClass"); 
	cout << obj->ToString(); 
	delete obj; 
	return 0; 
}
```

## 2. 工厂模式结合回调机制

梳理出基本的脉络：

1. 工厂内部需要有个映射,也就是一个字符串对应一个类new的方法。
2. 工厂给出一个接口,我们传入字符串,那么返回这个字符串对应的方法new出来的对象指针。
3. 我们新建的类,如果需要支持反射机制,那么这个类需要自动将自己的new方法和名字注册到工厂的映射中。

基于此，首先我们需要一个Object作为支持反射机制的基类：

```c++
class Object 
{ 
public: 
	Object(){} 
	virtual ~Object(){} 
	static bool Register(ClassInfo *ci);	// 注册传入一个ClassInfo(类信息)，将这个类的信息注册到映射中 
	static Object* CreateObject(std::string name);	// 工厂生产对象的接口 
};
```

然后是实现：

```c++
bool Object::Register(ClassInfo* ci)
{ 
	if (!classInfoMap) {
		classInfoMap = new std::map<std::string, ClassInfo *>(); // 通过map来存储这个映射
	} 
	if (ci) { 
		if (classInfoMap->find(ci->m_className) == classInfoMap->end()) {
            classInfoMap->insert(std::map<std::string, ClassInfo *>::value_type(ci->m_className, ci));
        }else {
            // TODO:原映射中已经存在该名称的类
            return false;
        }
	}
	return true; 
}

Object* Object::CreateObject(std::string name) 
{
	std::map<std::string, ClassInfo *>::const_iterator iter = classInfoMap->find(name);
	if (iter != classInfoMap->end()) { 
		return iter->second->CreateObject();
	} 
	return NULL; 
} 
```

ClassInfo类是用来自动将类的信息注册到映射中的：

```c++
class ClassInfo 
{ 
public: 
	ClassInfo(const std::string className, ObjectConstructorFn ctor)
		: m_className(className), m_objectConstructor(ctor) 
	{ 
		Object::Register(this);	// ClassInfo的构造函数是传入类名和类对应的new函数然后自动注册进map中。 
	} 
	virtual ~ClassInfo(){}
	Object *CreateObject() const { return m_objectConstructor ? (*m_objectConstructor)() : NULL; }
	bool IsDynamic() const { return NULL != m_objectConstructor; }
	const std::string GetClassName() const { return m_className; }
	ObjectConstructorFn GetConstructor() const { return m_objectConstructor; }

public: 
	std::string m_className;
	ObjectConstructorFn m_objectConstructor;
};
```

有了这些类后，我们只需要让需要支持反射的类满足以下要求即可：

1. 继承Object类。
2. 重载一个CreatObject()函数，里面 return new 自身类。
3. 拥有一个classInfo的成员并且用类名和CreatObject初始化。

满足以上三个要求的类就可以利用反射机制来创建对象了。

举例如下：

```c++
class MyObject : public Object 
{ 
public: 
	MyObject(){ std::cout << "MyObject constructor!" << std::endl; } 
	~MyObject(){ std::cout << "MyObject destructor!" << std::endl; }
	virtual ClassInfo *GetClassInfo() const { return &m_classinfo; }
	static Object* CreateObject() { return new MyObject; } 
protected: 
	static ClassInfo m_classinfo; 
}; 
ClassInfo MyObject::m_classinfo("MyObject", MyObject::CreateObject); 
```

具体的使用方法如下所示：

```c++
int main() 
{ 
	Object* obj = Object::CreateObject("MyObject");
    delete obj; 
	return 0;
}
```

通过观察可以发现，包括函数声明、函数定义、函数注册，每个类的代码除了类名外其它都是一模一样的，因此就可以利用宏对代码进行简化。

```c++
// 类申明中添加ClassInfo属性和CreatObject、GetClassInfo方法
#define DECLARE_CLASS(name) \
protected: \
	static ClassInfo m_classinfo; \
public: \
	virtual ClassInfo *GetClassInfo() const; \
	static Object *CreateObject();

// 实现CreatObject和GetClassInfo的两个方法
#define IMPLEMENT_CLASS_COMMON(name) \
	ClassInfo *name::GetClassInfo() const { return &name::m_classinfo; } \
	Object *name::CreateObject() { return new name; }

// ClassInfo属性的初始化
#define IMPLEMENT_CLASS(name) \
	IMPLEMENT_CLASS_COMMON(name) \
	ClassInfo name::m_classinfo(#name, name::CreateObject);
```

使用宏了之后，就可以这样改写MyObject类，效果与之前一样。

```c++
class MyObject2 : public Object
{
DECLARE_CLASS(MyObject2)
public:
	MyObject2(){ std::cout << "MyObject2 constructor!" << std::endl; }
	~MyObject2(){ std::cout << "MyObject2 destructor!" << std::endl; }
};
IMPLEMENT_CLASS(MyObject2)
```

# 三. 小结

以上只是完成了C++反射的部分功能，因为这些并没有完整的实现C++的反射机制，只是实现了反射机制中的一个小功能模块而已，即通过类名称字符串创建类的实例。除此之外，编程语言的反射机制所能实现的功能还有通过类名称字符串获取类中属性和方法，修改属性和方法的访问权限等等。

而对于反射机制的使用：由于在 Java 和 .NET 的成功应用，反射技术以其明确分离描述系统自身结构、行为的信息与系统所处理的信息，建立可动态操纵的因果关联以动态调整系统行为的良好特征，已经从理论和技术研究走向实用化，使得动态获取和调整系统行为具备了坚实的基础。当需要编写扩展性较强的代码、处理在程序设计时并不确定的对象时，反射机制会展示其威力，这样的场合主要有：
（1）序列化（Serialization）和数据绑定（Data Binding）；
（2）远程方法调用（Remote Method Invocation RMI）；
（3）对象/关系数据映射（O/R Mapping）。

当前许多流行的框架和工具，例如 Castor（基于 Java 的数据绑定工具）、Hibernate（基于 Java 的对象/关系映射框架）等，其核心都使用了反射机制来动态获得类型信息。因此，能够动态获取并操纵类型信息，已经成为现代软件的标志之一。

# 参考文献

[1] [C++ 反射机制详解及实例代码](https://www.jb51.net/article/103669.htm)

[2] [我所理解的 C++ 反射机制](https://blog.csdn.net/K346K346/article/details/51698184)

# 附录

下面是本文涉及的完整代码，包括Reflex.h和Reflex.cpp：

```c++
// Reflex.h

#include <iostream>
#include <string>
#include <map>

class ClassInfo;

class Object
{
public:
	Object(){}
	virtual ~Object(){}
	static bool Register(ClassInfo *ci);	// 注册传入一个ClassInfo(类信息)，将这个类的信息注册到映射中
	static Object* CreateObject(std::string name);	// 工厂生产对象的接口
};

typedef Object* (*ObjectConstructorFn)(void);

class ClassInfo
{
public:
	ClassInfo(const std::string className, ObjectConstructorFn ctor)
		: m_className(className), m_objectConstructor(ctor)
	{
		Object::Register(this);	// ClassInfo的构造函数是传入类名和类对应的new函数然后自动注册进map中。
	}
	virtual ~ClassInfo(){}
	Object *CreateObject() const { return m_objectConstructor ? (*m_objectConstructor)() : NULL; }
	bool IsDynamic() const { return NULL != m_objectConstructor; }
	const std::string GetClassName() const { return m_className; }
	ObjectConstructorFn GetConstructor() const { return m_objectConstructor; }

public:
	std::string m_className;
	ObjectConstructorFn m_objectConstructor;
};

// 类申明中添加ClassInfo属性和CreatObject、GetClassInfo方法
#define DECLARE_CLASS(name) \
protected: \
	static ClassInfo m_classinfo; \
public: \
	virtual ClassInfo *GetClassInfo() const; \
	static Object *CreateObject();

// 实现CreatObject和GetClassInfo的两个方法
#define IMPLEMENT_CLASS_COMMON(name) \
	ClassInfo *name::GetClassInfo() const { return &name::m_classinfo; } \
	Object *name::CreateObject() { return new name; }

// ClassInfo属性的初始化
#define IMPLEMENT_CLASS(name) \
	IMPLEMENT_CLASS_COMMON(name) \
	ClassInfo name::m_classinfo(#name, name::CreateObject);
```

```c++
// Reflex.cpp

#include "Reflex.h"

static std::map<std::string, ClassInfo *> *classInfoMap = NULL;

bool Object::Register(ClassInfo* ci)
{
	if (!classInfoMap) {
		classInfoMap = new std::map<std::string, ClassInfo *>(); //这里我们是通过map来存储这个映射的。
	}
	if (ci) {
		if (classInfoMap->find(ci->m_className) == classInfoMap->end()) {
            classInfoMap->insert(std::map<std::string, ClassInfo *>::value_type(ci->m_className, ci)); // 类名 <-> classInfo
        }else {
            // TODO:原映射中已经存在该名称的类
            return false;
        }
	}
	return true;
}

Object* Object::CreateObject(std::string name)
{
	std::map<std::string, ClassInfo *>::const_iterator iter = classInfoMap->find(name);
	if (iter != classInfoMap->end()) {
		return iter->second->CreateObject();     //当传入字符串name后,通过name找到info,然后调用对应的CreatObject()即可
	}
	return NULL;
}

// Example 1
class MyObject : public Object
{
public:
	MyObject(){ std::cout << "MyObject constructor!" << std::endl; }
	~MyObject(){ std::cout << "MyObject destructor!" << std::endl; }
	virtual ClassInfo *GetClassInfo() const { return &m_classinfo; }
	static Object* CreateObject() { return new MyObject; }
protected:
	static ClassInfo m_classinfo;
};
ClassInfo MyObject::m_classinfo("MyObject", MyObject::CreateObject);

// Example 2
class MyObject2 : public Object
{
DECLARE_CLASS(MyObject2)
public:
	MyObject2(){ std::cout << "MyObject2 constructor!" << std::endl; }
	~MyObject2(){ std::cout << "MyObject2 destructor!" << std::endl; }
};
IMPLEMENT_CLASS(MyObject2)

int main()
{
	Object* obj = Object::CreateObject("MyObject");
    Object* obj2 = Object::CreateObject("MyObject2");
    delete obj;
    delete obj2;
	return 0;
}
```

