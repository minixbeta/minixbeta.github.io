---
layout: default
title: 4Sum
---

## 题目 

```
Given an array S of n integers, are there elements a, b, c, and d in S such that a + b + c + d = target? Find all unique quadruplets in the array which gives the sum of target.

Note:
Elements in a quadruplet (a,b,c,d) must be in non-descending order. (ie, a ≤ b ≤ c ≤ d)
The solution set must not contain duplicate quadruplets.
    For example, given array S = {1 0 -1 0 -2 2}, and target = 0.

    A solution set is:
    (-1,  0, 0, 1)
    (-2, -1, 1, 2)
    (-2,  0, 0, 2)
```

## 解答

* 基本思想：计算出两两相加的结果，构成一个有序数组，将问题转化为一个 2Sum 问题
* 使用一个 HashMap 保存 (sum, (i,j))，其中 (i, j) 是相加为 sum 的两个数在原数组中的下标
* 如果找到 (sum1, (i1, j1)), (sum2, (i2, j2))，保证 sum1 + sum2 == target，并且 (j1 < i2) ，那么可以将 (i1, j1, i2, j2)
添加进结果集
* 结果集需要使用 HashSet，避免重复
