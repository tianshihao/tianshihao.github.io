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

# 一维动态规划

转移方程是关键，一般是dp[i] = f(dp[i - 1], dp[i - 2], ...)。

题目：

1. 70，爬楼梯
1. 746，带cost的爬楼梯
1. 1137，斐波那契数列
上面三个最优相同的模版。
1. 198，打家劫舍

同一个模版，外层遍历更新每个dp位置，内层遍历所有选择项
1. 279，完全平方数
1. 322，零钱兑换


1. 139，两层循环
2. 300，两层循环

3. [62. Unique Paths](https://leetcode.cn/problems/unique-paths/description/)
4. [2312. Selliing Pieces of Wood](https://leetcode.cn/problems/selling-pieces-of-wood/description/)

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
5. 1466，递归

## 回溯

1. 17，不允许重复
2. 46，不允许重复
3. 78
4. 39，允许重复
5. 40，不允许重复
6. 216，不允许重复
7. 22，两个条件判断
8. 79，单词搜索，分方向

## 单调栈

1. 901
2. 739
3. 84

## 图论

1. 207，拓扑排序，BFS
2. 994，腐烂橘子，BFS
3. 1926，迷宫出口，BFS，类似腐烂橘子，需要修改原数组

4. 200，岛屿数量，DFS