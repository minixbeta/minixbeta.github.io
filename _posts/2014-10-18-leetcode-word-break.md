---
layout: default
title: Word Break
---

## 题目 

```
Given a string s and a dictionary of words dict, determine if s can be segmented into a space-separated sequence of one or more dictionary words.

For example, given
s = "leetcode",
dict = ["leet", "code"].

Return true because "leetcode" can be segmented as "leet code".
```

## 解答

* 检查 s[0, i] 是否在 dict 里，以及子问题 s[i:] 是否能由 dict 构成
* 使用缓存，避免子问题重复求解
