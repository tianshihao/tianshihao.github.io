+++
date = '2021-02-05T17:09:00+08:00'
draft = false
title = 'Erase-Remove惯用法'
tags = ['C++', 'STL']
toc = true
+++

看到《Effective STL》条款 9 的时候想到了我以前复习的“如何正确使用迭代器删除元素”，我面试时使用的也是里面的方法，看面试官的反应好像也没有什么问题，还问了我一些我早已整理过的考点。但看到条款 9 之后，我就觉得自己以前回答得没什么水平了。

文本参考了条款 9 和条款 32。

## remove()

`<algorithm>` 中的 `remove()` 的声明为：

```c++
template<class ForwardIt, class T>
ForwardIt remove(ForwardIt first, ForwardIt last, const T& value);

```

也就是从范围 [first, last) 内移除所有满足特定判别标准的元素，并返回新结尾的尾后迭代器。需要注意的是，**`remove()` 并不真正删除东西，因为它做不到**。`remove()` 移动指定区间中的所有元素直到所有“不删除的”元素在区间的开头（相对位置和原来的一样）。它返回一个指向最后一个“不删除的”元素的下一个元素的迭代器。返回值是区间的“新**逻辑**终点”。

```c++
vector<int> v;
v.reserver(10);
for(int i = 1; i <= 10; ++i)
{
    v.push_back(i);
}

v[3] = v[5] = v[9] = 99;
vecitor<int>::iterator newEnd(remove(v.begin(), v.end(), 99));

```

调用 `remove()` 之前：

![调用remove之前](https://gitee.com/tianshihao4944/gitee-images-bed/raw/master/20210205142005.png)

调用 `remove()` 之后：

![调用remove之后](https://gitee.com/tianshihao4944/gitee-images-bed/raw/master/20210205142051.png)

## erase-remove 惯用法

如果要真正删除东西的话，应该在 `remove()` 后面接上 `erase()`。要 `erase()` 的元素很容易识别：它们是从区间的“新逻辑终点”开始持续到区间真正重点的原来区间的元素。要除去那些元素，要做的事情就是用两个迭代器调用 `erase()` 的区间形式。

```c++
vector<int> v;
v.erase(remove(v.begin(), v.end(), 99), v.end());

```

把 `remove` 的返回值作为 `erase()` 区间形式第一个实参传递很常见，这是个惯用法。

## remove_if() 和 remove_copy_if()

### 序列容器

如果要删除的不是每个有特定值的物体，而是消除下面的判断式

```c++
bool badValue(int x); // 返回 x 是否是 "bad".

```

对于序列容器，要做的只是把每个 `remove()` 替换成 `remove_if()`，然后就完成了：

```c++
c.erase(remove_if(c.begin(), c.end(), badValue), c.end());

```

这样面试题“从 `vector` 中删除 3 的倍数”就变得更简单了：

```c++
bool ThreeTimes(int num)
{
    return num % 3 == 0 ? true : false;
}

int main()
{
    // ...
    // 一行代码, 没有循环.
    vec.erase(std::remove_if(vec.begin(), vec.end(), ThreeTimes), vec.end());
    // ...
}

```

真是好笑，听到面试官问我这道题，我心中窃喜，还想着秀一秀我关于“如何正确使用迭代器删除元素”的清晰思路，哈哈，真是贻笑大方 😂。

更完美的版本：

```c++
#include <algorithm>
#include <iostream>
#include <vector>
using namespace std;

int main()
{
    vector<int> vec{19, 21, 28, 13, 16, 2, 3, 29, 12,
                    4, 25, 17, 26, 5, 20, 18, 30, 6, 22,
                    11, 24, 14, 7, 23, 1, 15, 27, 8, 9, 10};

    // 使用 lambda 函数作为谓词. 当然 -> bool 可以省略.
    vec.erase(
        remove_if(vec.begin(),
                  vec.end(),
                  [&](int num) -> bool { return num % 3 == 0; }),
        vec.end());

    cout << "after erase-remove\n";
    cout << "size: " << vec.size() << "\n";
    cout << "capacity: " << vec.capacity() << "\n\n";

    // 使用交换技巧收缩空间.
    vector<int>(vec).swap(vec);

    cout << "after swap trick\n";
    cout << "size: " << vec.size() << "\n";
    cout << "capacity: " << vec.capacity() << "\n";

    return 0;
}

```

输出如下：

```
after erase-remove
size: 20
capacity: 30

after swap trick
size: 20
capacity: 20

```

### 关联容器

对于关联容器，`remove_if()` 不是很直截了当。有两种方法处理该问题：

1. `remove_copy_if()`，更容易，效率较低；
2. 循环，更高效。

关于 `remove_copy_if()`，它**把需要的值拷贝到另一个新容器中**，然后把原容器的内容和新的交换：

```c++
#include <algorithm>
#include <set>

bool ThreeTimes(int num)
{
    return num % 3 == 0 ? true : false;
}

int main()
{
    std::set<int> c{1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    std::set<int> notThreeTimes; // 存储非 3 的倍数的临时容器.

    // 从 c 复制不删除的值到 notThreeTimes.
    std::remove_copy_if(c.begin(), c.end(),
                        std::inserter(notThreeTimes, notThreeTimes.end()),
                        ThreeTimes);

    c.swap(notThreeTimes); // 交换 c 和 notThreeTimes.

    return 0;
}

```

这种方法的缺点是它复制了所有不删除的元素，而这样的复制开销花费很大。

`for` 循环很简单，注意 `erase()` 时处理好迭代器就行：

```c++
for(std::set<int>::iterator i = c.begin(); i != c.end();)
{
    if(ThreeTimes(*i))
    {
        // 还可以向日志文件中写入一条消息.
        // logFile << "Erasing " << *i << '\n';
        c.erase(i++);
    }
    else
    {
        ++i;
    }
}

```

当然，要是序列容器也需要在 `erase()` 时有其它操作，同样可以使用 `for` 循环，只需注意序列式容器使用迭代器删除元素的技巧就行。

## 结论

去除一个容器中特定值的所有对象：

1. 如果容器是 `vector`、`stirng` 或 `deque`，使用 erase-remove 惯用法；
2. 如果容器是 `list`，使用 `list::remove`；
3. 如果容器是标准关联容器，使用它的 `erase()` 成员函数。

去除一个容器中满足一个特定判定式的所有对象：

1. 如果容器是 `vector`、`string` 或 `deque`，使用 erase-remove_if 惯用法；
2. 如果容器是 `list`，使用 `list::romove_if()`；
3. 如果容器是标准关联容器，使用 `remove_copy_if()` 和 `swap()`，或写一个循环来遍历容器元素，但把迭代器传给 `erase()` 时记得后置递增它。

在循环内做某些事情（除了删除对象之外）：

1. 如果容器是标准序列容器，写一个循环来遍历元素，每当调用 `erase()` 时记得用它的返回值更新迭代器；
2. 如果容器是标准关联容器，写一个循环来遍历元素，当把迭代器传给 `erase()` 时记得后置递增它。
