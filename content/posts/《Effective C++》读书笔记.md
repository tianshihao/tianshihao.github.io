+++
date = '2024-04-21T13:44:00+08:00'
draft = false
title = '《Effective C++》读书笔记'
tags = ['C++']
toc = true
+++

# 《Effective C++》读书笔记

之前看过一遍，不过草草了事。近日看了《深度探索 C++对象模型》，想起《Effective C++》中的内容已经有些忘记了，所以重新温习一下。这篇笔记只挑选书中的一些重要内容进行记录。

## 条款 07：为多态基类声明 virtual 析构函数

这一个条款几乎是面试中的高频问题，只需要回答，如果不将基类中的析构函数声明为虚函数，那么当我们通过基类指针删除派生类对象时，只会调用基类的析构函数，而不会调用派生类的析构函数，这样就会导致派生类中的资源无法释放，即“局部销毁”，造成内存泄漏。可是，为什么会这样呢？以及为什么将析构函数声明为虚函数就可以解决这个问题呢？

让我们结合《深度探索 C++对象模型》中的内容来解释这个问题。-我们知道 C++通过虚拟机制实现多态，而在表现上，多态就是“以一个 public base class 的指针或者引用寻指出一个 derived class object”的意思。因此通过基类指针调用函数时，编译器会检查该函数是否在虚表中出现，以将该调用替换成正确的形式。下面来讨论基类析构函数不是虚函数时，会发生什么。

### 基类析构函数不是虚函数

#### 不展现多态：基类中无其它虚函数

这种情况下，派生类只是简单继承了基类，没有重写基类的任何函数，派生类对象中也不会有 vptr。因此，当我们通过基类指针删除派生类对象时，只会调用基类的析构函数，派生类的析构函数不会被调用，这样就会导致派生类中的资源无法释放，即“局部销毁”，造成内存泄漏。

#### 展现多态：基类中有其它虚函数

这种情况下，派生类重写了基类的某个虚函数，派生类对象中会有 vptr，vptr 指向派生类的虚表。而因为基类的析构函数不是虚函数，即其没有被重写，所以派生类的虚表中析构函数那一项指向的是基类的析构函数。“局部销毁”的情况同样会发生。

### 基类的析构函数是虚函数

那么如果基类的析构函数是虚函数，按照条款 07，就不会发生“局部销毁”的情况，可是，为什么呢？直觉上，基类的析构函数为虚的话，按照虚拟机制的一般规则，基类指针会直接调用派生类的析构函数，析构掉派生类的资源，可是基类呢？为什么基类的数据成员也会被析构掉呢？

这里就要引入《深度探索 C++对象模型》中的内容了。在《深度探索 C++对象模型》中，作者介绍了析构语意学，提到了析构函数的扩展形式：

1. destructor 的函数本体首先被执行。
2. 如果 class 拥有 member class objects，而后者拥有 destructors，那么它们会以其声明顺序的相反顺序被调用。
3. 如果 object 内含一个 vptr，那么首先重设（reset）相关的 virtual table。
4. 如果有任何直接的（上一层）nonvirtual base class 拥有 destructor，它们会以其声明顺序的相反顺序被调用。
5. 如果有任何 virtual base classes 拥有 destructor，而目前讨论的这个 class 是最尾端（most-derived）的 class，那么它们会以其原来的构造顺序的相反顺序被调用。

即析构函数会被以该顺序进行扩展，生成一个完整的析构函数。因此，当我们通过基类指针删除派生类对象时，会按照上述顺序调用析构函数，即首先调用派生类的析构函数，然后调用基类的析构函数，这样就可以保证派生类中的资源也会被释放。

下面是示例代码：

```cpp
#include <cstdlib>
#include <iostream>

class Base {
 public:
  Base() {
    _base_data = static_cast<char*>(malloc(100));
    std::cout << "Base constructor" << std::endl;
  }
  virtual ~Base() {
    free(_base_data);
    _base_data = nullptr;
    std::cout << "Base destructor" << std::endl;
  }

 private:
  char* _base_data;
};

class Derived : public Base {
 public:
  Derived() {
    _derived_data = static_cast<char*>(malloc(100));
    std::cout << "Derived constructor" << std::endl;
  }
  virtual ~Derived() override {
    free(_derived_data);
    _derived_data = nullptr;
    std::cout << "Derived destructor" << std::endl;
  }

 private:
  char* _derived_data;
};

int main() {
  Base* base{new Derived()};
  delete base;
  return 0;
}
```

## 条款 09：绝不在构造和析构过程中调用 virtual 函数

这个条款的内容也是面试中的高频问题，只需要回答，因为在构造和析构过程中，对象的类型是不断变化的，如果在这两个过程中调用虚函数，那么调用的是当前对象的虚函数，而不是最终对象的虚函数，这样就会导致错误的发生。

## 条款 27：尽量少做转型动作

1. const_cast 通常被用来将对象的常量性转除（cast away the constness）。它也是唯一有此能力的 C++-style 转型操作符。
2. dynamic_cast 主要用来执行“安全向下转型”（safe downcasting），也就是用来决定某个对象是否归属继承体系中的某个类型。它是唯一无法由旧式语法执行的动作，也是唯一可能耗费重大运行成本的转型动作。将 pointer-to-base 转为 pointer-to-derived。
3. reinterpret_cast 意图执行低级转型，实际动作（及结果）可能取决于编译器，这也就表示它不可移植。例如将一个 pointer to int 转型为一个 int。这一类转型在低级代码以外很少见。
4. static_cast 用来强迫隐式转换（implicit conversions），例如将 non-const 对象转成 const 对象，或将 int 转为 double 等等。它也可以用来执行上述多种转型的反向转换，例如将 void \*指针转为 typed 指针，将 pointer-to-derived 转为 pointer-to-base。但是它无法将 const 转为 non-const——这个只有 const_cast 才能办到。

## 条款 30：透彻了解 inlining 的里里外外

## 条款 41：了解隐式接口编程和编译期多态

OOP 中常见显示接口和运行期多态。

```cpp
void doProcessing(Widget& w) {
  if (w.size() > 10 && w != sameNastyWidget) {
    Widget temp(w);
    temp.normalize();
    temp.swap(w);
  }
}
```

1. w 必须支持 Widget 这个接口，我们可以在代码中找到这个接口，比如在 widget.h 中，可以看到它是什么样子，所以我们称其为显示接口。这个接口就是签名。
2. 由于 Widget 的某些成员函数是 virtual 的，因此对这些函数的调用必须等到运行时才能确定，这就是运行期多态。

```cpp
template <typename T>
void doProcessing(T& w) {
  if (w.size() > 10 && w != sameNastyWidget) {
    T temp(w);
    temp.normalize();
    temp.swap(w);
  }
}
```

1. w 必须支持哪一种接口，系由 template 中执行于 w 身上的操作来决定的。本例来看 w 的类型 T 好像必须支持 size、normalize 和 swap 成员函数、copy 构造函数、不等比较操作符。我们看不到 T 的定义，所以我们称其为隐式接口。
2. 凡是涉及 w 的任何函数调用，都有可能造成该 template 的具现化（instantiated），使这些调用成功。这样的具现行为发生在编译期。“以不同的 template 参数具现化 function templates”会导致调用不同的函数，这表示所谓的编译期多态（compile-time polymorphism）。
