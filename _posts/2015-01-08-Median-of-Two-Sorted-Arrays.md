---
layout: post
categories: [算法]
tags: [算法]
code: true
title: Median of Two Sorted Arrays
---

题目：  

```
There are two sorted arrays A and B of size m and n respectively. Find the median of the two sorted arrays. The overall run time complexity should be O(log (m+n)).
```  
一开始没看清题目，以为O(m+n)的时间复杂度。。。，然后就直接两个指针从数组头开始指向了，结果正常地超时了。  
明显是用二分法！  
这题其实还是求第K个数，假设两个数组A,B.长度大于k，比较A[k/2 - 1]和B[k/2 - 1]的大小。
三种情况：
1.A[k/2 - 1] = B[k/2 - 1]，找到了所求；    
2.A[k/2 - 1] < B[k/2 - 1]，那么A[k/2 -1]和它之前的元素可以不用找了，不会在那里的，排除他们。这时候k = k - (k/2 -1)即把不可能的数排除后，k的值也在变，这样才可以递归嘛。
3.A[k/2 - 1] > B[k/2 - 1]，同上。
做算法题，还是要注意边界值！！！思考的时候也最好举例子在草稿一边演算，这样思路明确，更加清晰。
网上有段代码写的优美，条理清晰，效率恐怖：    

```
double findKth(int a[], int m, int b[], int n, int k)
{
	//always assume that m is equal or smaller than n
	if (m > n)
		return findKth(b, n, a, m, k);
	if (m == 0)
		return b[k - 1];
	if (k == 1)
		return min(a[0], b[0]);
	//divide k into two parts
	int pa = min(k / 2, m), pb = k - pa;
	if (a[pa - 1] < b[pb - 1])
		return findKth(a + pa, m - pa, b, n, k - pa);
	else if (a[pa - 1] > b[pb - 1])
			return findKth(a, m, b + pb, n - pb, k - pb);
		 else
			return a[pa - 1];
}


class Solution
{
public:
	double findMedianSortedArrays(int A[], int m, int B[], int n) {
		int total = m + n;
		if (total & 0x1)
			return findKth(A, m, B, n, total / 2 + 1);
		else
			return (findKth(A, m, B, n, total / 2) + findKth(A, m, B, n, total / 2 + 1)) / 2;
		}
};
```