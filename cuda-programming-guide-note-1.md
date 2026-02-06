+++
date = '2026-02-06T14:18:38+08:00'
draft = false
title = 'CUDA Programming Guide Note 1'
tags = ['CUDA']
toc = true
series = ["CUDA Programming Guide"]
+++

> 原文：https://docs.nvidia.com/cuda/cuda-programming-guide/01-introduction/programming-model.html

# CUDA编程模型

## 1. Thread Blocks and Grids

当程序启动一个kernel时，会使用很多很多的线程去处理这个kernel，实际上是数百万个线程。这些线程被组织到很多 Block 里面。一组线程被称做线程块。线程块又被组织进一个Grid里面。一个Grid里面所有的线程块Blocks，有相同的大小和维度。

这里补充一张图。

线程块和grid的维度可以是1、2或者3。这些维度可以帮助把每一个线程映射到对应的工作单元或者对应的数据。

kernel启动的时候，它需要一些特定的配置来指定grid和block的维度。配置中还可以包含诸如cluster size、stream和SM设置等参数。

使用内建的变量，每一个线程可以知道它在block中的相对位置（`threadIdx`）。同样，每一个block也可以知道它在grid中的相对位置（`blockIdx`）。
线程还可以使用这些内建变量来确定block的维度（`blockDim`）以及grid的维度（`gridDim`）。

这些加起来，每一个线程就在所有运行kernel的线程里面拥有了一个唯一的标识（索引）。这个标识经常被用来确定一个线程正在操作哪些数据。
