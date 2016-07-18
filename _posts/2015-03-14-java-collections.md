---
layout: post
categories: [Java]
tags: [Java]
code: true
title: 
---

&emsp;&emsp;自从接触ConcurrentHashMap后，对java集合越来越有兴趣。今天上午在宿舍楼道上看了一上午thinking in java，晚上做下笔记以便以后翻阅。  
其中有些继承关系图片上没有完全给出。比方LinkedList实现了双队列Deque,而Deque extends Queue。Java容器现在已经很完全了，那些抽象类可以不用记忆。   

### 队列  
Java SE5中，Queue只有两个实现LinkedList和PriorityQueue，还有其他子接口及其子接口的实现。  

### 双向队列
LinkedList支持，但是Java木有显示的用于的用于双向队列的接口。可以创建一个类组合LinkedList，直接从LinkedList中暴露其Deque接口方法。  

### Map
映射表，也称为关联数组，基本思想是维护键—值对。  
散列的思想是用key产生一个数，这个数和key的hashCode相关，作为数组的下标，然后准备添加到该数组指向的链表上，于是遍历链表，看看有没有已经也是相同hashCode映射到这里的，如果有的话，那么要通过equals方法比较有没有相同的对象，如果有会覆盖该元素，返回旧Value。通过hashCode()映射马上可以访问到该数组，然后数组指向的链表HashMap的put方法用了hash（Key），而hash（）方法是用key.hashCode()。
所以作为key的类要重写equals方法和hashCode方法。因为put方法，是通过该的hashCode映射到数组，然后通过equals方法判断两个对象是否相同。  
重写equals要满足5个条件：自反性，对称性，传递性，一致性，equals（null）要返回FALSE。其中注意一致性：调用x.equals(y)多次返回的结果应是一样的。   
重写hashCode方法很重要，因为好的散列设计会使得value分布均匀，而要非常注意的是：同一个对象hashCode（）应该产生相同的值。如果put()后，hashCode（）产生的值不同，get()方法怎么找回该value。如果hashCode依赖变量，那么要注意了，数据的修改是会改变hashCode的值。默认的hashCode方法使用的对象的地址。  

