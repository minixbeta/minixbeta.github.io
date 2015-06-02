---
layout: post
title: Largest Rectangle in Histogram
category: 技术
---


## 题目

![demo](http://www.leetcode.com/wp-content/uploads/2012/04/histogram_area.png)

```
Given n non-negative integers representing the histogram's bar height where the width of each bar is 1, 
find the area of largest rectangle in the histogram.




Above is a histogram where width of each bar is 1, given height = [2,1,5,6,2,3].


The largest rectangle is shown in the shaded area, which has area = 10 unit.

For example,
Given height = [2,1,5,6,2,3],
return 10.
```

## 思路 

维护一个栈，栈中存放元素位置，如果当前元素比栈顶元素指向的元素大，则当前元素位置入栈。否则，出栈。每次出栈后，当前栈顶元素就是
以当前元素为高的矩形的左边界，当前元素位置即为右边界。 `当前元素` 只有入栈后，当前位置才会跳到下一个，否则不变，并且每次都要
与栈顶元素指向元素比较。

栈中维护的其实是一个元素位置列表，这些位置指向的元素由小到大排列，所以列表中元素左边的位置就是左边第一个比他小的元素的位置，
而右边界是右边第一个比出栈位置指向元素小的元素。
