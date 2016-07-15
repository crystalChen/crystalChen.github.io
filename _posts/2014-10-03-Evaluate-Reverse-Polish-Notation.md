---
layout: post
categories: [算法]
tags: [JMM]
code: true
title: Evaluate Reverse Polish Notation
---
题目：  

```
Evaluate the value of an arithmetic expression in Reverse Polish Notation.  
Valid operators are +, -, *, /. Each operand may be an integer or another expression.  
Some examples:  
  ["2", "1", "+", "3", "*"] -> ((2 + 1) * 3) -> 9  
  ["4", "13", "5", "/", "+"] -> (4 + (13 / 5)) -> 6
```
后缀表达式，用栈做，遇到运算符出栈两个元素做运算，然后把结果入栈。switch（“字符串”）是jdk1.7的新特性。

```
public class Solution {
    public int evalRPN(String[] tokens) {
        int temp = 0;
        MyStack ms = new MyStack();
        for(int i=0; i<tokens.length; i++){
            switch(tokens[i]){
                case "+": temp = ms.pop() + ms.pop(); ms.push(temp);break;
                case "-": temp = -ms.pop() + ms.pop(); ms.push(temp);break;
                case "*": temp = ms.pop() * ms.pop(); ms.push(temp);break;
                case "/": temp = ms.pop();temp = ms.pop() / temp; ms.push(temp);break;
                default: temp = Integer.parseInt(tokens[i]); ms.push(temp);
            }
        }
        return ms.pop();
    }
}
class MyStack{
    int[] stack = new int[1000000];
    int top = 1;
    int pop(){
        if(top == 0) return 0;
        top--;
        return stack[top];
    }
    void push(int a){
        if(top > stack.length)
            return;
        stack[top] = a;
        top++;
    }
}
```