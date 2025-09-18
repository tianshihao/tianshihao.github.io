+++
date = '2021-02-05T17:24:00+08:00'
draft = false
title = '手动实现 shared_ptr'
tags = ['C++', '数据结构', '面试']
toc = false
+++

面试写了一个基础的 `scoped_ptr`，被面试官要求写 `shared_ptr`，一时语塞。面试官不断提示我说在现有的基础上实现 `shared_ptr` 很简单，真的很简单，宛如在不断暗示我 1+1 就是把两个数加在一起 😂。我知道简单，也知道引用计数原理，但没有写过代码啊，不知道具体是怎么实现引用计数的，当时只能放弃了。

今天研究了一下 `shared_ptr`，手写了简单实现，现在记录一下实现的重点。

## 引用计数

面试时，我想是不是用 `static` 实现的引用计数？其实不能这样做，静态变量是同属一个类所有对象，这样的话这个类的所有对象，不管指向的是不是相同的内存，它们共享的都是同一个计数器。

引用计数，就是在内存中开辟一块内存专门用于存储资源被引用的次数，自己保留一个指向计数器的指针，计数器初始化为 1。每次用已有的 `shared_ptr` 构造新的 `shared_ptr` 或者给已有的 `shared_ptr` 赋值时，通过指针将计数器加 1，所有指向同一个资源的 `shared_ptr` 也指向同一个计数器。

## 释放所有权

当 `shared_ptr` 析构或主动释放所有权时，应将引用计数减 1，然后判断引用计数是否归零，如果归零，那么就释放指针指向的内存，此后将计数器指针置为空。需要注意的是，如果将计数器指针置为空，为了防止指针主动释放所有权之后计数器指针为空，导致判断计数是否归零，需要在判断计数归零前先判断计数指针是否为空。

## 参考代码

```c++
#include <iostream>
#include <string>

template <typename T>
class SharedPtr
{
private:
    // 裸指针.
    T *_ptr;
    // 指向引用计数器的指针.
    int *_pcount;

public:
    SharedPtr(T *ptr) : _ptr(ptr), _pcount(new int(1))
    {
        std::cout << "SharedPtr " << this << " constructed!\n\n";
    }

    // 复制构造函数.
    SharedPtr(const SharedPtr<T> &s) : _ptr(s._ptr), _pcount(s._pcount)
    {
        std::cout << "Entering copy constructor\n";

        std::cout << "\tSharedPtr " << this
                  << " copied from SharePtr " << &s << std::endl;

        // 引用计数加一.
        std::cout << "\tReference count increased by one\n";
        ++(*(_pcount));

        std::cout << "Leaving copy constructor\n\n";
    }

    ~SharedPtr()
    {
        std::cout << "Entering destructor\n";

        // 重置指针为空.
        this->_ptr = nullptr;

        std::cout << "\tReference count decreased by one\n";

        // 引用计数减一.
        if (_pcount != nullptr && --(*(this->_pcount)) == 0)
        {
            delete this->_ptr;
            _ptr = nullptr;
            delete _pcount;
            _pcount = nullptr;

            std::cout << "\tCount zero, memory released\n";
        }

        // 将引用计数指针置为空.
        this->_pcount = nullptr;

        std::cout << "SharedPtr " << this << " destructed\n";
    }

public:
    // 返回引用计数的个数.
    int use_count() const
    {
        return *(this->_pcount);
    }
    int use_count()
    {
        return *(this->_pcount);
    }
    // 返回指针是否具有独占权, 即引用计数是否为 1.
    bool unique() const
    {
        return *(this->_pcount) == 1 ? true : false;
    }
    bool unique()
    {
        return *(this->_pcount) == 1 ? true : false;
    }
    // 返回裸指针.
    T *get() const
    {
        return this->_ptr;
    }
    // 释放当前指针的资源所有权, 计数减一.
    void release()
    {
        std::cout << "\tRelease resource ownership\n";

        // 重置指针为空.
        this->_ptr = nullptr;

        std::cout << "\tReference count decreased by one\n";

        // 将当前资源引用计数减一.
        if (_pcount != nullptr && --(*(this->_pcount)) == 0)
        {
            delete this->_ptr;
            delete this->_pcount;
            this->_pcount = nullptr;
            std::cout << "\tCount zero, memory released\n";
        }

        // 将引用计数指针置为空.
        this->_pcount = nullptr;
    }

public:
    SharedPtr<T> &operator=(const SharedPtr<T> &s)
    {
        std::cout << "Entering assignment operator\n";

        if (this == &s)
        {
            return *this;
        }

        // 释放原来的旧资源.
        release();

        // 新资源计数加一.
        std::cout << "\tReference count increased by one\n";
        this->_ptr = s._ptr;
        this->_pcount = s._pcount;
        ++(*(this->_pcount));

        std::cout << "Leaving assignment operator\n\n";

        return *this;
    }
    T &operator*()
    {
        return *(this->_ptr);
    }
    T *operator->()
    {
        return this->_ptr;
    }
};

int main()
{
    SharedPtr<std::string> p1(new std::string("Hello world!"));
    SharedPtr<std::string> p2(p1);
    SharedPtr<std::string> p3 = p2;
    p3 = p2;

    std::cout << p2.use_count() << std::endl;
    p1.release();
    std::cout << p2.use_count() << std::endl;

    std::cout << *(p2.get()) << std::endl;

    return 0;
}

```

输出如下

```
SharedPtr 0x7ffffffedba0 constructed!

Entering copy constructor
        SharedPtr 0x7ffffffedbb0 copied from SharePtr 0x7ffffffedba0
        Reference count increased by one
Leaving copy constructor

Entering copy constructor
        SharedPtr 0x7ffffffedbc0 copied from SharePtr 0x7ffffffedbb0
        Reference count increased by one
Leaving copy constructor

Entering assignment operator
        Release resource ownership
        Reference count decreased by one
        Reference count increased by one
Leaving assignment operator

3
        Release resource ownership
        Reference count decreased by one
2
Hello world!
Entering destructor
        Reference count decreased by one
SharedPtr 0x7ffffffedbc0 destructed
Entering destructor
        Reference count decreased by one
        Count zero, memory released
SharedPtr 0x7ffffffedbb0 destructed
Entering destructor
        Reference count decreased by one
SharedPtr 0x7ffffffedba0 destructed

```
