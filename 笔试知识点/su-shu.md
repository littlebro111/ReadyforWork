# 素数

## 朴素算法

时间复杂度：$O(n^2)$

根据素数的定义，试除以从2开始到sqrt(n)的整数，若均不能整除，则该数必为素数。

```
#include <iostream>
#include <cmath>
using namespace std;

bool isPrime(int n)
{
    for(int i = 2; i * i <= n; ++i)
    {
        if (n % i == 0) return false;
    }
    return true;
}

int main()
{
    int n;
    cin >> n;
    for(int i = 2; i <= n; ++i){
        if (isPrime(i)){
            cout << i << " is a prime." << endl;
        }else{
            cout << i << " isn't a prime." << endl;
        }
    }
    return 0;
}
```

## Eratosthenes筛法（埃氏筛法）

时间复杂度：$O(n_lglgn)$（约等于 $O(1.5_n)$）

首先将2到n范围内的整数写下来，其中2是最小的素数。将表中所有的2的倍数划去，表中剩下的最小的数字就是3，他不能被更小的数整除，所以3是素数。再将表中所有的3的倍数划去……以此类推，如果表中剩余的最小的数是m，那么m就是素数。然后将表中所有m的倍数划去，像这样反复操作，就能依次枚举n以内的素数。

```
#include <iostream>
#include <unordered_set>
#define Max 1000000
using namespace std;

bool flags[Max];
unordered_set<int> st;

void findPrime1(int n){
	for(int i = 0; i <= n; ++i) flags[i] = true;
	for(int i = 2; i <= n; ++i){
		if (flags[i]){
            st.insert(i);
            for(int j = i * i; j <= n; j += i){
                flags[j] = false;
            }
        }
	}
}
int main() {
    int n;
    cin >> n;
	findPrime1(n);
	for(int i = 2; i <= n; ++i){
        if (st.count(i)){
            cout << i << " is a prime." << endl;
        }else{
            cout << i << " isn't a prime." << endl;
        }
    }
	return 0;
}
```

## 欧拉筛法

时间复杂度：$O(n)$

欧拉筛法改进了埃式筛法的一些冗余，避免了重复的标记，其思想基础是“任何一个合数都可以由两个质数相乘得到”，那么对于每一个n我们就都可以用比它小的某一个质数来筛去。

```
#include <iostream>
#include <vector>
#include <unordered_set>
#define Max 1000000
using namespace std;

bool flags[Max];
vector<int> primes;
unordered_set<int> st;

void findPrime2(int n) {
    for(int i = 0; i <= n; ++i) flags[i] = 1;
	for(int i = 2; i <= n; ++i){
		if (flags[i]){
            primes.emplace_back(i);
            st.insert(i);
        }
        int len = primes.size();
        for(int j = 0; j < len && i * primes[j] <= n; ++j) {
			flags[i * primes[j]] = false;
			if(i % primes[j] == 0) break;
		}
	}
}
int main() {
	int n;
    cin >> n;
	findPrime2(n);
	for(int i = 2; i <= n; ++i){
        if (st.count(i)){
            cout << i << " is a prime." << endl;
        }else{
            cout << i << " isn't a prime." << endl;
        }
    }
	return 0;
}
```

> 注：欧拉筛的难点就在于对`if (i % primes[j] == 0) break;`这步的理解，当i是primes\[j]的整数倍时，记 m = i / primes\[j]，那么 i \* primes\[j+1] 就可以变为 (m \* primes\[j+1]) \* primes\[j]，这说明 i \* primes\[j + 1] 是 primes\[j] 的整数倍，不需要再进行标记(在之后会被 primes\[j] \* 某个数 标记)，对于 prime\[j+2] 及之后的素数同理，直接跳出循环，这样就避免了重复标记。

[参考资料](https://blog.csdn.net/Little\_Bro/article/details/124913587)
