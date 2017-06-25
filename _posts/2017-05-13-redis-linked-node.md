layout: post
categories: [Redis]
tags: [Redis]
code: true
title: 《Redis设计与实现》链表




integers 列表键的底层实现就是一个链表， 链表中的每个节点都保存了一个整数值。

除了链表键之外， 发布与订阅、慢查询、监视器等功能也用到了链表， Redis 服务器本身还使用链表来保存多个客户端的状态信息， 以及使用链表来构建客户端输出缓冲区（output buffer）。

每个链表节点使用一个 adlist.h/listNode 结构来表示：

    typedef struct listNode {
    
        // 前置节点
        struct listNode *prev;
    
        // 后置节点
        struct listNode *next;
    
        // 节点的值
        void *value;
    
    } listNode;

多个 listNode 可以通过 prev 和 next 指针组成双端链表， 如图 3-1 所示。



虽然仅仅使用多个 listNode 结构就可以组成链表， 但使用 adlist.h/list 来持有链表的话， 操作起来会更方便：

    typedef struct list {
    
        // 表头节点
        listNode *head;
    
        // 表尾节点
        listNode *tail;
    
        // 链表所包含的节点数量
        unsigned long len;
    
    } list;

list 结构为链表提供了表头指针 head 、表尾指针 tail ， 以及链表长度计数器 len 。

因为链表表头节点的前置节点和表尾节点的后置节点都指向 NULL ， 所以 Redis 的链表实现是无环链表。


