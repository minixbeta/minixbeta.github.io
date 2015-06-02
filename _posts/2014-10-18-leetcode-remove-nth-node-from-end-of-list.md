---
layout: post
title: Remove Nth Node From End of List
category: 技术
---


## 题目

```
Given a linked list, remove the nth node from the end of list and return its head.

For example,

   Given linked list: 1->2->3->4->5, and n = 2.

   After removing the second node from the end, the linked list becomes 1->2->3->5.
Note:
Given n will always be valid.
Try to do this in one pass.
```

## 解决

* 两个指针，一个指针先走 n 步，然后另一个指针也开始走，一直到前一个指针的下一结点为null，此时后一指针的下一结点就是要删除的
结点
* 但凡涉及删除链表中结点，一定要注意有没有考虑到删除的结点是头结点和尾结点的情况
* 注意 n 超过链表长度是如此处理的
