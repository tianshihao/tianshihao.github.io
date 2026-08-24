+++
date = '2024-03-09T08:14:00+08:00'
draft = true
title = 'LeetCode做题笔记'
tags = ['C++', 'LeetCode']
toc = true
+++

# 二分查找

关键是找到满足的条件，例如nums[pos - 1] < target <= nums[pos]、target == nums[pos]，然后代入模板即可。

题目：

1. [35. Search Insert Position](https://leetcode.cn/problems/search-insert-position/description/)
2. [349. Intersection of Two Arrays](https://leetcode.cn/problems/intersection-of-two-arrays/description/)

# 动态规划

转移方程是关键，一般是dp[i] = f(dp[i - 1], dp[i - 2], ...)。

题目：


1. **[70. Climbing Stairs](https://leetcode.cn/problems/climbing-stairs/description/)**
   1. 自己的解法：递归+记忆话搜索、动态规规划
2. [62. Unique Paths](https://leetcode.cn/problems/unique-paths/description/)
2. [2312. Selliing Pieces of Wood](https://leetcode.cn/problems/selling-pieces-of-wood/description/)

## 记忆化搜索

就是开个visited数组，存储已经计算过的结果。常和递归一起使用，进入新的调用栈之后检查一下visited，用于优化递归的时间复杂度。

## 二叉树中序遍历

1. 94
2. 98，二叉搜索树
3. 230，二叉搜索树
4. 872，二叉搜索树
5. 700，二叉搜索树

## 二叉搜索树

1. 450

## 二叉树先序遍历

1. 1448
2. 114

## 二叉树的层序遍历

1. 1161
2. 199

## 前缀和

1. 437

## 深度优先搜索

1. 437
2. 124、543（二叉树后序遍历）
3. 841，递归
4. 547，递归

## 回溯

1. 17，不允许重复
2. 46，不允许重复
3. 39，允许重复
4. 40，不允许重复
5. 216，不允许重复

## 单调栈

1. 901
2. 739