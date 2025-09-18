+++
date = '2025-08-21T15:16:35+08:00'
draft = false
title = '空基类优化（EBO）'
tags = ['C++']
toc = true
+++

# 1. 类、结构体的对象大小

在 C++里面，一个类或者结构体的对象大小（`sizeof`）是其内部所有非静态数据成员做完内存对齐之后所占用的大小，例如：

```cpp
struct NotEmpty {
  char c; // 1 byte
  int i;  // 4 bytes
};

std::cout << sizeof(NotEmpty) << std::endl;
```

输出为 8，即`char`对齐后 4 个字节和`int`本身 4 个字节相加 8 字节。那么一个不含任何数据成员的类或结构体的对象呢？例如：

```cpp
struct Empty {};

std::cout << sizeof(Empty) << std::endl;
```

输出是 1，不是 0。这是因为**任何对象或者子对象的大小都必须至少为 1，即使类型为空类类型，以便能够保证同一类型的不同对象的地址始终不同。**

那么问题来了，如果从一个空基类派生出一个带有数据成员的类或结构体，这个派生类的对象大小会是多少呢？例如：

```cpp
struct Base {}; // empty class

struct Derived1 : Base {
    int i;
};

int main() {
    // the size of any object of empty class type is at least 1
    static_assert(sizeof(Base) >= 1);

    // empty base optimization applies
    static_assert(sizeof(Derived1) == sizeof(int));

    std::cout << sizeof(Derived1) << std::endl;

    return 0;
}
```

输出是 4。原因就是发生了**空基类优化（Empty Base Optimization，EBO）**。

# 2. 空基类优化（EBO）

EBO 允许编译器在派生类中不为空基类分配额外的内存空间，从而节省内存。一个使用 EBO 的典型场景是`std::reverse_iterator`的实现（从空基类`std::iterator`派生而来），并且它持有底层迭代器（同样从`std::iterator`派生）作为自己的第一个非静态数据成员。

```cpp
#include <iostream>

struct Base {};  // empty class

// EBO 生效，不为空基类 Base 分配内存空间，因此 Derived1 的大小为4
// 即等于其 int 成员的大小
struct Derived1 : Base {
  int i; // 4 bytes
};

// EBO 生效，不为空基类 Base 分配内存空间，因此 Derived2 的大小为8
struct Derived2 : Base {
  Base c;  // 持有空基类对象，不能使用 EBO，必须占有1字节，后面跟3字节的内存对齐
  int i;   // 4 bytes
};

// EBO 不生效，如果使用 EBO，Derived3 的 Base 子对象和 Derived1 的 Base 子对象
// 会重叠，这会违反 C++ 标准对同类型子对象地址唯一性的要求
struct Derived3 : Base {
  Derived1 c;  // derived from Base, occupies sizeof(int) bytes
  int i;
};

int main() {
  // empty base optimization does not apply,
  // base occupies 1 byte, Base member occupies 1 byte
  // followed by 2 bytes of padding to satisfy int alignment requirements
  static_assert(sizeof(Derived2) == 2 * sizeof(int));

  // empty base optimization does not apply,
  // base takes up at least 1 byte plus the padding
  // to satisfy alignment requirement of the first member (whose
  // alignment is the same as int)
  static_assert(sizeof(Derived3) == 3 * sizeof(int));

  std::cout << "sizeof(Base): " << sizeof(Base) << std::endl;
  std::cout << "sizeof(Derived1): " << sizeof(Derived1) << std::endl;
  std::cout << "sizeof(Derived2): " << sizeof(Derived2) << std::endl;
  std::cout << "sizeof(Derived3): " << sizeof(Derived3) << std::endl;

  return 0;
}
```

输出为

```text
sizeof(Base): 1
sizeof(Derived1): 4
sizeof(Derived2): 8
sizeof(Derived3): 12
```

`Derived1`、`Derived2`和`Derived3`在继承`Base`的时候编译器进行了 EBO，使得它们的大小得以减小。所以`Derived1`的大小等于其持有的`int`大小（4 字节），`Derived2`的大小等于其持有的`Base`对象大小（1 字节）加上`int`大小（4 字节），再加上中间 3 字节的内存对齐，总共 8 字节。

按照相同的逻辑，`Derived3`的大小等于其持有的`Derived1`对象大小（4 字节）加上`int`大小（4 字节），应该也为 8 字节，但实际输出的却是 12。原因是 C++标准要求“同类型子对象的地址必须不同”。

因为 EBO 的本质是把空基类的存储“重叠”到派生类的其它对象中，`Derived3`的第一个非静态数据成员`Derived1`的基类（子对象）已经是`Base`了，如果编译器再使用 EBO 把`Derived3`继承的空基类`Base`的存储重叠到`Derived1`对象上，相当于同一个地址上有两个`Base`子对象，一个是继承来的空基类`Base`，一个是`Derived1`的`Base`子对象，这就违反了“同类型子对象的地址必须不同”这一要求。因此`Derived3`实际的内存布局是：

```text
+--------------+ // Derived3 地址开始的地方
| Base (1B)    | // Derived3 继承而来的 Base 成员，无法使用 EBO，空类大小为1
+--------------+
|              |
| Padding(3B)  | // 内存对齐
|              |
+--------------+
|              |
| Derived1(4B) | // Derived1 使用了 EBO，因此大小为其成员 int 大小
|              |
|              |
+--------------+
|              |
| int(4B)      | // Derived3 的 int 成员
|              |
|              |
+--------------+
// Derived3 总大小为12字节
```

# 3. 总结

空基类优化（EBO）是 C++ 中的一种编译器优化技术，允许在派生类中不为空基类分配额外的内存空间，从而节省内存。派生类触发 EBO 需要满足以下条件：

1. 派生类从空基类派生。基类不能有任何非静态数据成员、不能有虚函数、不能是虚基类，但可以有静态数据成员。
2. 派生类的第一个非静态数据对象（或者其子对象）的类型和空基类（或者其子对象，空基类也可能多次继承）的类型不能相同，否则会导致同类型子对象地址重叠。
