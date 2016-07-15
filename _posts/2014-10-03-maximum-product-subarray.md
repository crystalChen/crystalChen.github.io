---
layout: post
categories: [算法]
tags: [JMM]
code: true
title: LeetCode|Reverse Words in a String
---

leetcode原题：  

```
For example,  
Given s = "the sky is blue",  
return "blue is sky the".  

Clarification:  
What constitutes a word?  
A sequence of non-space characters constitutes a word.  
Could the input string contain leading or trailing spaces?  
Yes. However, your reversed string should not contain leading or trailing spaces.  
How about multiple spaces between two words?  
Reduce them to a single space in the reversed string.  
```
此题也是九月26号晚上清华大学百度笔试第一大题，昨天晚上也木有调试出来，还是内个习惯，不雅一边听歌一边在电脑上写代码，今天晚饭后回来手写一便代码，思路清晰明了，解决了其中的bug。 
主要思路是先将连续空白字符去除，用的是StringBuffer接收，因为append方法不会创建新的对象。然后全部反转整个句子，再把一个一个单词反转回来。代码如下： 

``` 
public class Solution {
    public String reverseWords(String s) {
        //去除连续重复的空格
       StringBuffer str = new StringBuffer();
       int i = 0;
       while(i<s.length())
       {
           while(i<s.length() && s.charAt(i) != ' '  )
           {
               str.append(s.charAt(i));
               i++;
           }
           str.append(" ");
           while(i<s.length() && s.charAt(i) == ' ')
           {
               i++;
           }
       }
        char[] a = str.toString().trim().toCharArray();
        int j=a.length-1;
        int k=0;
        char temp=' ';
        i = 0;
        //reverse all the words
        while(i<j){
            temp = a[i];
            a[i] = a[j];
            a[j] = temp;
            i++;
            j--;
        }
        i = 0;
        j = 0;
        //reverse every word
        while(i < a.length){   
            //find the end char index of the word
            while(j<a.length && a[j] != ' ')  j++;
            k = j+1;    // remark the the first char of the next word 
            j--;
            //reverse the word
            while(i<j){    
                temp = a[i];
                a[i] = a[j];
                a[j] = temp;
                i++;
                j--;
            }
            i = k;
            j = k;
        }
        
       s = new String(a);
        return s;
    }
} 
```