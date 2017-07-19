# Multiplicative Hashing
Multiplicative Hashings是一种基于取余运算的高效哈希算法。
假设我们有一个基于链表的hash map，其伪代码结构如下：
```C
struct HashMap {
...
private:
  array<List> elements;
...
}
```
令2^d = elements.size(), 那么哈希函数定义如下：
```
hash(x) = (z * x mod (2^w)) div (2^(w-d))
```
其中z是在{1...2^w-1}中随机选取的一个奇数且w > d.

下面给出两个定理来说明该哈希函数为什么是高效的。

 定理1
 ```
 对于任意的x, y∈{1...2^w-1}, 且x != y, 那么有
 P{hash(x) == hash(y)} <= 2 / (2^d). P{t}表示t出现的概率。
 ```
 
 定理2
 ```
 对于任意的x，elements[hash(x)].size() <= Nx + 2, 其中Nx表示x在hash map中出现的次数。
 ```
 
 定理的证明参见《open data structures》中HashMap一节。
