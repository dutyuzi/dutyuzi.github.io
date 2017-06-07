---
layout:     post
title:      "蓄水池抽样算法"
subtitle:   "Reservoir sampling "
date:       2016/10/8 17:47:13 
author:     "MaJ"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - 算法
---


### 题目
如何随机从未知大小的数据流中选择***1个***数据
### 思路
1. 假设数据流只有一个数，则概率为1。若数据流中有两个数，编号为分别为1、2，则生成一个0到1的随机数R,如果R小于0.5我们就返回第一个数据，如果R大于0.5，返回第二个数据。



2. 若数据流中有三个数，编号分别为1、2、3。当收到1号和2号数据时，则按照二分之一的概率淘汰一个，若淘汰了1号数据，则继续读入3号数据，此时有2、3两个数据，只有保证选择3号数据的概率为1/3,选择结果才是正确的，所以在剩下2号和3号数据中，以1/3的概率选择3号数据，则以2/3的概率选择2号数据，最终概率如下：


	数据1被留下：（1/2）*(2/3) = 1/3；

	数据2被留下概率：（1/2）*(2/3) = 1/3；

	数据3被留下概率：1/3

### 解法

 我们总是选择第一个对象，以1/2的概率选择第二个，以1/3的概率选择第三个，以此类推，以1/m的概率选择第m个对象。当该过程结束时，每一个对象具有相同的选中概率，即1/n，证明如下。

证明：第m个对象最终被选中的概率P=选择m的概率*其后面所有对象不被选择的概率，即

![img](/img/zm1.jpg)

对应的该问题的伪代码如下：

    i = 0  
	while more input items  
        with probability 1.0 / ++i  
                choice = this input item  
	print choice  
	

C++代码如下：

    #include <iostream>  
	#include <cstdlib>  
	#include <ctime>  
	#include <vector>  
	  
	using namespace std;  
	  
	typedef vector<int> IntVec;  
	typedef typename IntVec::iterator Iter;  
	typedef typename IntVec::const_iterator Const_Iter;  
	  
	// generate a random number between i and k,  
	// both i and k are inclusive.  
	int randint(int i, int k)  
	{  
	    if (i > k)  
	    {  
	        int t = i; i = k; k = t; // swap  
	    }  
	    int ret = i + rand() % (k - i + 1);  
	    return ret;  
	}  
	  
	// take 1 sample to result from input of unknown n items.  
	bool reservoir_sampling(const IntVec &input, int &result)  
	{  
	    srand(time(NULL));  
	    if (input.size() <= 0)  
	        return false;  
	  
	    Const_Iter iter = input.begin();  
	    result = *iter++;  
	    for (int i = 1; iter != input.end(); ++iter, ++i)  
	    {  
	        int j = randint(0, i);  
	        if (j < 1)  
	            result = *iter;   
	    }  
	    return true;  
	}  
	  
	int main()  
	{  
	    const int n = 10;  
	    IntVec input(n);  
	    int result = 0;  
	  
	    for (int i = 0; i != n; ++i)  
	        input[i] = i;  
	    if (reservoir_sampling(input, result))  
	        cout << result << endl;  
	    return 0;  
	}  
	    
### 扩展
如何随机从未知大小的数据流中选择***k个***数据？这就是**蓄水池抽样问题**，即如何在海量未知数据中等概率的抽取k个数据。

类比下即可得到答案：
将最先读入的k个数据放入**蓄水池中**，从第k+1个数据开始以k/k+1的概率选择该数据，以k/k+2的概率选择第k+2个数据，依次类推，到第m（m>k）个数据时，以k/m的概率来选择第m个数据。如果m被选中，则将m放入蓄水池中，随机替换蓄水池中的一个数据。这样每个数据的被选中的概率就为k/n。

证明：第m个对象被选中的概率=选择m的概率 * （其后元素不被选择的概率+其后元素被选择的概率 * 不替换第m个对象的概率），即

![img](/img/zm2.jpg)

对应的该问题的伪代码如下：

    array S[n];    //source, 0-based  
	array R[k];    // result, 0-based  
	integer i, j;  
	  
	// fill the reservoir array  
	for each i in 0 to k - 1 do  
	        R[i] = S[i]  
	done;  
	  
	// replace elements with gradually decreasing probability  
	for each i in k to n do  
	        j = random(0, i);   // important: inclusive range  
	        if j < k then  
	                R[j] = S[i]  
	        fi  
	done  

C++代码如下：

	#include <iostream>  
	#include <cstdlib>  
	#include <ctime>  
	#include <vector>  
	  
	using namespace std;  
	  
	typedef vector<int> IntVec;  
	typedef typename IntVec::iterator Iter;  
	typedef typename IntVec::const_iterator Const_Iter;  
	  
	// generate a random number between i and k,  
	// both i and k are inclusive.  
	int randint(int i, int k)  
	{  
	    if (i > k)  
	    {  
	        int t = i; i = k; k = t; // swap  
	    }  
	    int ret = i + rand() % (k - i + 1);  
	    return ret;  
	}  
	  
	// take m samples to result from input of n items.  
	bool reservoir_sampling(const IntVec &input, IntVec &result, int m)  
	{  
	    srand(time(NULL));  
	    if (input.size() < m)  
	        return false;  
	  
	    result.resize(m);  
	    Const_Iter iter = input.begin();  
	    for (int i = 0; i != m; ++i)  
	        result[i] = *iter++;  
	  
	    for (int i = m; iter != input.end(); ++i, ++iter)  
	    {  
	        int j = randint(0, i);  
	        if (j < m)  
	            result[j] = *iter;  
	    }  
	    return true;  
	}  
	  
	int main()  
	{  
	    const int n = 100;  
	    const int m = 10;  
	    IntVec input(n), result(m);  
	  
	    for (int i = 0; i != n; ++i)  
	        input[i] = i;  
	    if (reservoir_sampling(input, result, m))  
	        for (int i = 0; i != m; ++i)  
	            cout << result[i] << " ";  
	    cout << endl;  
	    return 0;  
	}  




----------
参考：

[https://en.wikipedia.org/wiki/Reservoir_sampling](https://en.wikipedia.org/wiki/Reservoir_sampling "https://en.wikipedia.org/wiki/Reservoir_sampling")

[http://blog.csdn.net/huagong_adu/article/details/7619665](http://blog.csdn.net/huagong_adu/article/details/7619665 "http://blog.csdn.net/huagong_adu/article/details/7619665")

[http://blog.jobbole.com/42550/](http://blog.jobbole.com/42550/ "http://blog.jobbole.com/42550/")



