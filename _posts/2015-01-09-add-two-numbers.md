---
layout: post
categories: [算法]
tags: [算法]
code: true
title: LeetCode | Add Two Numbers
---

题目：  

```
You are given two linked lists representing two non-negative numbers. The digits are stored in reverse order and each of their nodes contain a single digit. Add the two numbers and return it as a linked list.   

Input: (2 -> 4 -> 3) + (5 -> 6 -> 4)   
Output: 7 -> 0 -> 8   
```

这题边界情况要处理好：空值；两个链表长度可能不等；最后两个为空还可能留有一个进位木有处理。  
这次做题，自己做的不太好：变量名取的渣，取个next，搞的自己逻辑混乱，所以取变量名要取好；还有如果一个为null可以不往下循环，直接指针指向不为空的内个链表就可以了。  
代码：  

```
/**
 * Definition for singly-linked list.
 * public class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode(int x) {
 *         val = x;
 *         next = null;
 *     }
 * }
 */
public class Solution {
    public ListNode addTwoNumbers(ListNode l1, ListNode l2) {
        if(l1 == null || l2 == null)
            return null;
        int temp1 = 0;
        int temp2 = 0;
        int count = 0;
        int sum = 0;
        ListNode first = null;
        ListNode next = null;
        
        while(l1 != null || l2 != null) {
            temp1 = l1 != null ? l1.val : 0;
            temp2 = l2 != null ? l2.val : 0;
            sum = temp1 + temp2 + count;
            if(sum > 9) {
                sum -= 10;
                count = 1;
            } else {
                count = 0;
            }
            ListNode list = new ListNode(sum);
            if(next == null) {
               first = list;
               next = first;
            } else {
               next.next = list;
               next = list;
            }

            l1 = l1 == null ? null :l1.next;
            l2 = l2 == null ? null :l2.next;
           
        }
        if(count == 1) {
            next.next = new ListNode(1);
        }
        return first;
    }
}
```