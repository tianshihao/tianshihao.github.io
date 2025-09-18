+++
date = '2021-02-05T17:05:00+08:00'
draft = false
title = 'vector的交换技巧'
tags = ['C++', 'STL']
toc = true
+++

面试被问到如何解决 vector 有过多空闲内存的问题。

假定先有一 `vector` 容器 `vec`，它的容量是 10000，大小是 3。

## vector 的内存增长问题

`vector` 申请的是连续内存空间，其实际分配的内存比当前所需的内存要多一些，也就是说，`vector` 容器预留了一些额外的存储区。而当 `vector` 需要分配新的内存时，会申请当前容量二倍的内存，也就是二倍增长。

## resize() 和 reverse()

`resize()` 并不能解决内存浪费的问题。使用 `resize()` 可以将一个容量和大小均为 10000 的 `vector` 的大小改为 3，但 `resize()` 并不是将其余的内存释放掉，后面的内存仍然被占用，因为此时 `vector` 的容量（`capacity()`）仍为 10000。

这时候想到，要是先用 `resize()` 减小容器的大小，再使用 `reverse()` 减少容器的容量，不就好了么？但这样也不行，当使用 `reverse()` 调整容器容量时，若 `new_cap` 大于当前 `capacity()`，则会分配新的存储空间，复制原容器元素，销毁原容器；但**若 `new_cap` 小于当前 `capacity()`，则什么也不做**。

例如以下代码：

```c++
#include <iomanip>
#include <iostream>
#include <vector>
using namespace std;

int main()
{
    // 初始化时预算分配 10000 的大小.
    vector<int> vec(10000, 1);

    cout << setiosflags(ios::left);

    cout << "Initial\n"
         << setw(20) << "vec's size(): " << vec.size() << endl
         << setw(20) << "vec's capacity(): " << vec.capacity() << endl
         << endl;

    // 调整容器大小. 假定现在只需要 3 的大小.
    vec.resize(3);

    cout << "After resize()\n"
         << setw(20) << "vec's size(): " << vec.size() << endl
         << setw(20) << "vec's capacity(): " << vec.capacity() << endl
         << endl;

    // 调整容器容量.
    vec.reserve(3);

    cout << "After reserve()\n"
         << setw(20) << "vec's size(): " << vec.size() << endl
         << setw(20) << "vec's capacity(): " << vec.capacity() << endl
         << endl;

    return 0;
}

```

输出为：

```c++
Initial
vec's size():       10000
vec's capacity():   10000

After resize()
vec's size():       3
vec's capacity():   10000

After reserve()
vec's size():       3
vec's capacity():   10000

```

可以看到 `resize()` 和 `reverse()` 无法处理浪费的空间。

## 交换技巧

交换技巧，使用到了 `swap(vector& other)`，它会将 `vector` 与 `other` 相交换，且不在单独的元素上调用任何移动、复制或交换操作。利用 `swap()` 释放多余的存储空间的操作如下：

```c++
vector<int>(vec).swap(vec);

```

表达式 `vector<int>(vec)` 建立一个临时 `vector`，它是 `vec` 的一份拷贝，`vector` 的复制构造函数做了这个工作。并且，这个复制品只复制了复制元素所需要的内存，所以这个临时 `vector` 没有多余的容量。

然后，让临时 `vector` 和 `vec` 交换数据，`vec` 拥有了临时 `vector` 的修整过的容量，而临时 `vector` 则拥有了发胀的容量。最后语句执行完毕，临时 `vector` 被析构，空间被释放，大功告成。
