# 将任意2-36进制数转化为10进制数
## 1. 自己写函数，代码如下：
```cpp
int Atoi(string s,int radix)	// s是给定的radix进制字符串
{
	int ans=0;
	for(int i=0;i<s.size();i++)
	{
		char t=s[i];
		if(t>='0'&&t<='9') ans=ans*radix+t-'0';
		else ans=ans*radix+t-'a'+10;
	}
	return ans;
} 	
```
## 2. strtol()函数
函数原型：`long int strtol(const char *nptr, char **endptr, int base)`<br />nptr是要转化的字符，非法字符会赋值给endptr，base是转化前的进制，例如:
```cpp
#include <cstdio>
int main()
{
    char buffer[20] = "10549stend#12";
    char *stop;
    int ans = strtol(buffer, &stop, 8);	// 将八进制数1054转成十进制，后面均为非法字符
    printf("%d\n", ans);
    printf("%s\n", stop);
    return 0;
}

/*
输出结果：
556
9stend#12
*/
```
注意：
> - 如果base为0，且字符串不是以0x(或者0X)开头，则按十进制进行转化。 
> -  如果base为0或者16，并且字符串以0x（或者0X）开头，那么，x（或者X）被忽略，字符串按16进制转化。 
> - 如果base不等于0和16，并且字符串以0x(或者0X)开头，那么x被视为非法字符。 
> -  对于nptr指向的字符串，其开头和结尾处的空格被忽视，字符串中间的空格被视为非法字符。 

# 将10进制数转换为任意的n进制数
## 1. 自己写函数，代码如下：
```cpp
string intToA(int n,int radix)	// n是待转数字，radix是指定的进制
{
    string ans = "";
    do{
        int t = n % radix;
        if(t >= 0 && t <= 9)	ans += t + '0';
        else ans += t - 10 + 'a';
        n /= radix;
    }while(n != 0);	// 使用do{}while（）以防止输入为0的情况
    reverse(ans.begin(), ans.end());
    return ans;	
}
```
## 2. itoa()函数
函数原型：`char *itoa(int value,char *string,int radix);`<br />value是int型的待转换的十进制数，string是转化后的结果，radix是转化后的目标进制，例如: 
```cpp
#include<cstdio>
#include<cstdlib>
int main()
{
    int num = 10;
    char str[100]; 
    _itoa(num, str, 2);	// c++中一般用_itoa，用itoa也行,
    printf("%s\n", str);
    return 0;
}
```
# 通用方法：使用字符串流stringstream
## 1. 将八，十六进制转十进制
```cpp
#include <iostream>
#include <string>
#include <sstream>
using namespace std;
int main(void)
{
	string s = "20";
	int a;
	stringstream ss;
	ss << hex << s;	// 以16进制读入流中
	ss >> a;	// 10进制int型输出
	cout << a <<endl;
    return 0;
}

// 输出：32
```
## 2. 将十进制转八，十六进制
```cpp
#include <cstdio>
#include <iostream>
#include <string>
#include <sstream>
using namespace std;
int main(void)
{
	string s1,s2;
	int a = 30;
	stringstream ss;
	ss << oct << a;	// 10进制转成八进制读入流中，再以字符串输出
	ss >> s1;
	cout << s1 << endl;	// 输出：36
	ss.clear();	// 不清空可能会出错。
	ss << hex << a;	// 10进制转成十六进制读入流中，，再以字符串输出
	ss >> s2;			
	cout << s2 << endl;	// 输出：1e
    return 0;
}
```

[参考资料](https://blog.csdn.net/Little_Bro/article/details/124913739)
