---
title: Data Structrue
categories: Data Structure and Algorithm
---
# Linked List(链表)

线性结构，与数组的区别是可以不连续存储，链表变换时不必移动一整块内存。元素的顺序一般由链表对象中的指针决定。

搜索链表在最坏情况下的运行时间为$\theta(n)$。

插入一个元素进入链表的运行时间一般为$\theta(1)$

删除一个链表中的元素的运行时间一般为$\theta(1)$

## Singly Linked List(单向链表)

链表中元素的指针仅包含指向下一个元素的指针。

## Doubly Linked List(双向链表)

链表中元素的指针包含指向下一个元素的指针和上一个元素的指针

## Circular Linked List(双向循环链表)

包含双向链表的特性，但是链表的头元素会包含一个指向尾元素的指针，尾元素的指针包含一个指向头元素的指针。

# Stack(栈)

线性结构，但是后进先出(Last-In First-Out, LIFO)的线性结构，仅在结构的一端进行动作。

栈的插入操作被称为压入(Push)，而删除操作被称为弹出(Pop)，且一般在实际的实现中不要求参数。

试图对空栈执行弹出成为栈下溢(Underflow)，对已满的栈执行压入称为栈上溢(Overflow)。

查询、压入、弹出的运行时间均为$\theta(1)$

# Queue(队列)

线性结构，但是先进先出(First-In First-Out, FIFO)的线性结构，在结构的两端进行动作。

队列的插入操作被称为入队(Enqueue)，而删除操作被称为出队(Dequeue)，且一般在实际的实现中不要求参数。

队列下溢是指试图对空队列执行出队，队列上溢是指试图对已满队列执行入队。

入队、出队的运行时间均为$\theta(1)$。

# Hash(哈希表，散列表)

以常数平均时间执行插入、删除、查找的数据结构。该数据结构的元素被映射到从0至表的大小-1这个范围中的某个数，并且被放到相应的结构单元中，这个过程由散列函数(Hash Function)负责。这个过程中会不可避免地遇到散列值冲突的问题，如果碰上表的大小和元素具有一些不理想的性质，那么散列函数还需要处理这些性质带来的问题。

## Separate Chaining(分离链接)

将散列到同一个散列值的元素用一个链表保存

## Open Addressing(开放定址)



# Binary Tree(二叉搜索树)



# Splay Tree(伸展树)



# B-Tree(B树)



# Self-Balancing Binary Tree(自平衡二叉树)



## Single Rotation(单旋转)



## Double Rotation(双旋转)



# Red Black Tree(红黑树)



# Reference

1. Data Structures and Algorithm Analysis in C [Second Edition] - Mark Allen Weiss, 冯舜玺译
2. Introduction to Algorithm [Third Edition] - Thomas H.Cormen, Charles E.Leiserson, Ronald L. Rivest, Clifford Stein, 殷建平、徐云、王刚、刘晓光、苏明、邹恒明、王宏志译