---
layout: post
categories: [算法]
tags: [算法]
code: true
title: 求数组的子数组之和的最大值（分治法）
---

&nbsp;&nbsp;&nbsp;&nbsp;记得在上《软件工程》的时候的一个夏天的某个下午，吴一帆老师出过这道题目，当时自己没什么研究，只知道O(n^2)的解法，现在已经在网上看到很多次了，解法也看过好多种，但是都没有手写然后在机器上调试过，今天手痒痒想用递归思想分治法做一下，测试的比较成功，记录如下  
&nbsp;&nbsp;&nbsp;&nbsp;分治方法根据最大连续子数组存在三种位置的可能    
&nbsp;&nbsp;&nbsp;&nbsp;1. 出现在数组中间的左边（a[0],a[1]...a[n/2]);  
&nbsp;&nbsp;&nbsp;&nbsp;2.出现在数组中间的右边(a[n/2+1],...a[n-1]);  
&nbsp;&nbsp;&nbsp;&nbsp;3.子数组经过了中间。  
对于第三种情况，我们分两边计算，因为穿过了中间，不重于第一第二情况，那么必然结果中含有a[n/2]和a[n/2+1]，记a[n/2]以左以a[n/2]开头的连续数组最大值为S1，记a[n/2+1]以右以a[n/2]开头的连续数组最大值为S2，因为要考虑到可能都是负数的情况，S1和S2的初始值都要以开始第一个数组为初值。代码如下:
    
		int MaxSum(int* a,int i, int j)
		{
			if(i == j) return a[i];
			int mid = (i+j)/2;
			int num1 = MaxSum(a,i,mid);
			int num2 = MaxSum(a,mid+1,j);
			int num3 = 0;
			int max1 = a[mid],max2 = a[mid+1];
			int s1 = 0,s2 = 0;
			for(int k=mid; k>=i; k--)
			{
				s1 += a[k];
				if(s1 > max1) max1 = s1;
			}
			for(int k=mid+1; k<=j; k++){
				s2 += a[k];
				if(s2 > max2) max2 = s2;
			}
			num3 = max1 + max2;
			int max = 0;
			max = (num1>num2) ? num1:num2;
			max = (num3>max) ? num3:max;
			return max;
		}
		int main(){
			int a[] = { 1,-2,3,5,-3,2 };
			int maxsum = MaxSum(a,0,5);
			printf("%d\n",maxsum);
		} 
		
算法时间复杂度和堆排序一样.


