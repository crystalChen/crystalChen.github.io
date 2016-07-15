---
layout: post
categories: [算法]
tags: [JMM]
code: true
title: Maximum Product Subarray
---

&emsp;&emsp;我上网查找资料的时候，看到一个东西，喜欢去查内个东西，然后发现和自己最初想查的东西越来越远，不过没关系，都是知识，而且无心插柳看到的知识面更广，好吧，然后昨天遇到了leetcode，其实早先看到过这个词，但是没去注册然后刷题目，昨天注册了，刷了题目，第一题如下：

```
Find the contiguous subarray within an array (containing at least one number) which has the largest product.  
For example, given the array [2,3,-2,4],  
the contiguous subarray [2,3] has the largest product = 6.  
```

&emsp;&emsp;这个和最大子数组之和的最大值很像，可是昨天晚上做了两三个钟头都没做出来，想了想，自己算法根基不太好吧，还一边听歌，看电影。。。太影响效率了，今天早上再战，当然是先看了《编程之美》。有了明确的思路，写起来就是快，代码如下：  

```
class Solution {
public:
    int maxProduct(int A[], int n) {
        int start1[n]; 	//记录以A[i]开头的最大连续正整数乘积
        int start2[n];	 //记录以A[i]开头的最小连续负整数乘积
        int All[n];   	 //记录A[i]-A[n-1]之间的最大乘积
        start1[n-1] = A[n-1];
        start2[n-1] = A[n-1];
        All[n-1] = A[n-1];
        for(int i=n-2; i>=0; i--)
        {
            start1[i] = max(A[i], A[i]*start1[i+1], A[i]*start2[i+1]);
            start2[i] = min(A[i], A[i]*start1[i+1], A[i]*start2[i+1]);
            All[i] = start1[i] > All[i+1] ? start1[i] : All[i+1];
            
        }
        return All[0];
    }

    int max(int a,int b,int c){
        int max = a;
        if(max<b) max = b;
        if(max<c) max = c;
        return max;
    }
    
    int min(int a,int b,int c){
        int min = a;
        if(min>b) min = b;
        if(min>c) min = c;
        return min;
    }
};
```