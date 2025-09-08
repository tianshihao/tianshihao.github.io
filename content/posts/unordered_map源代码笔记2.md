+++
date = '2025-09-08T15:08:08+08:00'
draft = false
title = 'unordered_map源代码笔记 2'
tags = ['C++', 'STL']
toc = true
series = ["STL源代码笔记"]
+++

# `std::unordered_map`源代码笔记 2

## 3. 算法

### 3.1 `insert()`

常见操作，向哈希表中插入一个新的键值对。考虑这样一段代码：

```cpp
#include <iostream>
#include <string>
#include <unordered_map>

struct MyStruct {
  std::string tag;

  MyStruct() = default;

  MyStruct(std::string const& t) noexcept : tag(t) {
    std::cout << "Invoking constructor for: " << tag << '\n';
  }

  MyStruct(MyStruct const& other) noexcept : tag(other.tag) {
    std::cout << "Invoking copy constructor for: " << tag << '\n';
  }

  MyStruct(MyStruct&& other) noexcept : tag(std::move(other.tag)) {
    std::cout << "Invoking move constructor for: " << tag << '\n';
  }

  friend std::ostream& operator<<(std::ostream& os,
                                  const MyStruct& ms) noexcept {
    return os << ms.tag;
  }
};

void Insert() {
  std::unordered_map<int, MyStruct> umap;

  // insert with initializer list
  umap.insert({1, MyStruct("initializer list")});

  // insert with lvalue
  auto p{std::make_pair(4, MyStruct("lvalue"))};
  umap.insert(p);

  // insert with rvalue
  umap.insert(std::make_pair(5, MyStruct("rvalue")));

  for (const auto& pair : umap) {
    std::cout << "Key: " << pair.first << ", Value: " << pair.second << '\n';
  }
}

int main() {
  Insert();

  return 0;
}
```

程序的输出是：

```text
Invoking constructor for: initializer list
Invoking move constructor for: initializer list
Invoking move constructor for: initializer list
Invoking constructor for: lvalue
Invoking move constructor for: lvalue
Invoking copy constructor for: lvalue
Invoking constructor for: rvalue
Invoking move constructor for: rvalue
Invoking move constructor for: rvalue
Key: 5, Value: rvalue
Key: 4, Value: lvalue
Key: 1, Value: initializer list
```

其中，向 `insert()` 传递 `initializer_list` 和传递 `std::make_pair()` 的效果类似：

- 首先构造一个 `MyStruct` 临时对象；
- 该对象被移动到 `std::pair` 中（调用一次移动构造）；
- 然后 `std::pair` 作为右值传递给 `insert()`，后被完美转发到`_Hashtable`的`_M_emplace()`，其中数据被构造为哈希表内部的数据结构（又一次移动构造）。

对于插入左值的情况，流程略有不同：

- 先构造 `MyStruct` 临时对象，并移动到 `std::pair`（一次移动构造）；
- 但 `insert(p)` 传递的是左值，节点内只能拷贝构造（调用一次拷贝构造）。

#### 3.1.1 构造`_Scoped_node`

观察`insert()`的调用栈，发现不论是插入左值还是右值，数据实际都是在`_Hashtable::_M_emplace()`中被真正传递给哈希表的。`_M_emplace()`首先会使用传入的数据构造成一个 RAII 的`_Scoped_node`，确保在插入过程中资源的正确管理。

```cpp
  template<typename _Key, typename _Value,
	   typename _Alloc, typename _ExtractKey, typename _Equal,
	   typename _H1, typename _H2, typename _Hash, typename _RehashPolicy,
	   typename _Traits>
    template<typename... _Args>
      auto
      _Hashtable<_Key, _Value, _Alloc, _ExtractKey, _Equal,
		 _H1, _H2, _Hash, _RehashPolicy, _Traits>::
      _M_emplace(true_type, _Args&&... __args)
      -> pair<iterator, bool>
      {
	// First build the node to get access to the hash code
	_Scoped_node __node { this, std::forward<_Args>(__args)...  };
	const key_type& __k = this->_M_extract()(__node._M_node->_M_v());
	__hash_code __code = this->_M_hash_code(__k);
	size_type __bkt = _M_bucket_index(__k, __code);
	if (__node_type* __p = _M_find_node(__bkt, __k, __code))
	  // There is already an equivalent node, no insertion
	  return std::make_pair(iterator(__p), false);

	// Insert the node
	auto __pos = _M_insert_unique_node(__k, __bkt, __code, __node._M_node);
	__node._M_node = nullptr;
	return { __pos, true };
      }
```

#### 3.1.2 计算哈希值

有了`_Scoped_node`之后，就可以计算哈希值了。首先使用`_M_extract()`提取出键，其默认值为 `__detail::_Select1st`，从`pair`中取出第一个元素，然后使用`_M_hash_code()`计算哈希值，其默认值为`std::hash<_Key>` 。这两个方法都是`_Hashtable`从`_Hashtable_base`继承来的。

代码中关于哈希函数存储和调用的机制有些晦涩但也很巧妙，这里仔细分析一下。

首先，不论是`_ExtractKey`、`_Equal`还是`_Hash`、`_H1`、`_H2`，它们都是函数对象（Funtor，又称仿函数）。函数对象是重载了`operator()`的类或结构体，因此可以像函数一样被调用。函数对象的好处是可以存储状态（存储到数据成员中），但是如果没有数据成员的话，这个函数对象就是空类，对其就可以使用 EBO（[Empty Base Optimization，空基类优化](空基类优化EBO.md)），使其在派生类中不占空间。

理论上，函数指针、`std::function`以及 lambda 表达式都可以用来替代函数对象，但它们既不能存储状态，也不能使用 EBO 优化做到空间零开销。所以从存储空间角度来讲，函数对象比`std::function`、lambda 表达式以及函数指针更节省空间、更具优势。

但 STL 不能假设用户自定义的函数对象一定能被 EBO 优化。为此，STL 设计了`_Hashtable_ebo_helper<int _Nm, typename _Tp, bool __use_ebo>`模板辅助类来完成 EBO：

- 如果用户的函数对象`_Tp`是空类（无非静态数据成员）且可继承（不为`final`），则`_Hashtable_ebo_helper`通过继承`_Tp`的方式存储它，实现零空间开销。
  - 此时`__use_ebo`的值计算结果为`true`，即可以提供一个`__use_ebo`为`true`的特化，实现 EBO。
- 否则就作为成员变量存储，保证功能完整。
  - 此时`__use_ebo`的值计算结果为`false`，即提供一个`__use_ebo`为`false`的特化，实现普通存储。
- 并且该模板辅助类提供统一的 getter 接口访问函数对象，方便使用。

还需要注意的是，`_Hashtable`并不是通过持有`_Hashtable_ebo_helper`数据成员的方式来存储函数对象的，而是通过继承链间接“持有”所有函数对象的。这样，所有被使用了 EBO 的函数对象都不会占用空间，最大化节省内存。因为每存储一个使用了 EBO 的`_Hashtable_ebo_helper`特化实例对象，就会占用 1 字的空间（C++标准要求每个对象有唯一地址），持有多个函数对象，就会占用更多空间。如持有的函数对象没有被 EBO，占用的存储空间之会更多。

**`_Hashtable`是通过继承的方式“持有”函数对象的**。`_Hashtable`直接继承`_Hashtable_base`，`_Hashtablebase`再直接以及间接继承了多个`_Hashtable_ebo_helper`，每一个`_Hashtable_ebo_helper`继承了用户提供的函数对象，最终，实现了函数对象在`_Hashtable`中的存储。这样，使用了 EBO 的函数对象就不会占用任何空间，实现了空间零开销。继承关系如下图所示：

![hashtable_base](/hashtable_base.svg)

> 总之，STL 通过模板类`_Hashtable_ebo_helper`封装函数对象，并利用 EBO 优化，既保证了空间零开销，又能为所有函数对象的访问提供了统一的接口。

```cpp
  /**
   *  Primary class template _Hashtable_ebo_helper.
   *
   *  Helper class using EBO when it is not forbidden (the type is not
   *  final) and when it is worth it (the type is empty.)
   */
  template<int _Nm, typename _Tp,
	   bool __use_ebo = !__is_final(_Tp) && __is_empty(_Tp)>
    struct _Hashtable_ebo_helper;

  // 两种特化

  /// Specialization using EBO.
  template<int _Nm, typename _Tp>
    struct _Hashtable_ebo_helper<_Nm, _Tp, true>
    // 使用 EBO 的特化继承 _Tp，_Tp 在 _Hashtable_ebo_helper 中不占空间
    : private _Tp
    {
      _Hashtable_ebo_helper() noexcept(noexcept(_Tp())) : _Tp() { }

      template<typename _OtherTp>
	_Hashtable_ebo_helper(_OtherTp&& __tp)
	: _Tp(std::forward<_OtherTp>(__tp))
	{ }

      const _Tp& _M_cget() const { return static_cast<const _Tp&>(*this); }
      _Tp& _M_get() { return static_cast<_Tp&>(*this); }
    };

  // 不使用 EBO 的特化
  /// Specialization not using EBO.
  template<int _Nm, typename _Tp>
    struct _Hashtable_ebo_helper<_Nm, _Tp, false>
    {
      _Hashtable_ebo_helper() = default;

      template<typename _OtherTp>
	_Hashtable_ebo_helper(_OtherTp&& __tp)
	: _M_tp(std::forward<_OtherTp>(__tp))
	{ }

      const _Tp& _M_cget() const { return _M_tp; }
      _Tp& _M_get() { return _M_tp; }

    private:
      // 老老实实持有对象
      _Tp _M_tp{};
    };
```

模板辅助类`_Hashtable_ebo_helper`的特化除了会检测模板参数是否可以 EBO 之外，还会提供统一的访问接口`_M_get()`返回存储的函数对象，方便`Hashtable`中调用函数对象。如`_Hash_code_base`继承了`_Hashtable_ebo_helper<0, _ExtractKey>`和`_Hashtable_ebo_helper<1, _Hash>`，同时`_Hash_code_base`提供了两个接口`_M_extract()`和`_M_ranged_hash()`，它们分别使用对应的`_Hashtable_ebo_helper::_M_get()`和`_Hashtable_ebo_helper::_M_cget()`返回存储存储的`_ExtractKey`和`_Hash`。

很妙的设计。

```cpp
  // 其中一个特化
  /// Specialization: hash function and range-hashing function, no
  /// caching of hash codes.
  /// Provides typedef and accessor required by C++ 11.
  template<typename _Key, typename _Value, typename _ExtractKey,
	   typename _H1, typename _H2>
    struct _Hash_code_base<_Key, _Value, _ExtractKey, _H1, _H2,
			   _Default_ranged_hash, false>
    : private _Hashtable_ebo_helper<0, _ExtractKey>,
      private _Hashtable_ebo_helper<1, _H1>,
      private _Hashtable_ebo_helper<2, _H2>
    {
    private:
      using __ebo_extract_key = _Hashtable_ebo_helper<0, _ExtractKey>;
      using __ebo_h1 = _Hashtable_ebo_helper<1, _H1>;
      using __ebo_h2 = _Hashtable_ebo_helper<2, _H2>;

    protected:
      // ...
      // We need the default constructor for _Hashtable default constructor.
      _Hash_code_base() = default;
      _Hash_code_base(const _ExtractKey& __ex,
		      const _H1& __h1, const _H2& __h2,
		      const _Default_ranged_hash&)
      : __ebo_extract_key(__ex), __ebo_h1(__h1), __ebo_h2(__h2) { }

      __hash_code
      _M_hash_code(const _Key& __k) const
      {
	static_assert(__is_invocable<const _H1&, const _Key&>{},
	    "hash function must be invocable with an argument of key type");
	return _M_h1()(__k);
      }

      // ...

      const _ExtractKey&
      _M_extract() const { return __ebo_extract_key::_M_cget(); }

      // H1 是哈希函数对象，负责将键映射为哈希值
      const _H1&
      _M_h1() const { return __ebo_h1::_M_cget(); }

      // H2 是分桶哈希函数对象，将哈希值映射为桶索引
      const _H2&
      _M_h2() const { return __ebo_h2::_M_cget(); }
    };
```

注意，`_M_hash_code()`、`_M_extract()`、`_M_h1()`、`_M_h2()`以及`_Hash_code_base`的构造函数，它们的访问等级都是`protected`，这意味着它们只能在`_Hash_code_base`及其派生类中访问，不允许单独使用。这符合`_Hash_code_base`作为抽象出来的策略基类的设计。`_Hashtable`会间接继承这个`_Hash_code_base`，从而可以访问这些成员。

了解了`_Hashtable_ebo_helper`的设计之后，理解计算哈希值的过程就很简单了。

首先，`_M_emplace()`使用`_M_extract()`提取出键，其默认值为 `__detail::_Select1st`，从`pair`中取出第一个元素。这里的`_M_extract()`实际上是调用了`_Hash_code_base`的`_M_extract()`。

```cpp
	const key_type& __k = this->_M_extract()(__node._M_node->_M_v());
```

然后就是使用`_M_hash_code()`计算哈希值。`_M_hash_code()`间接调用`__ebo_h1::_M_cget()`，即`_Hashtable_ebo_helper<1, _Hash>::_M_cget()`，返回函数对象`_Hash`。`_Hash`接收键`__k`，计算并返回其哈希值。`_Hash`的默认值是`std::hash<_Key>`，这是 C++ 标准库提供的通用哈希函数对象。

```cpp
	__hash_code __code = this->_M_hash_code(__k);
```

#### 3.1.3 计算桶索引

有了哈希值之后，就可以计算桶索引了。使用`_M_bucket_index()`方法，根据键和哈

确定元素应该放入哪个桶中。

## 附录

1. 哈希和数组的对比

你的问题很有代表性。可以这样解释：

如果直接用数组存储 key-value 对，通常有两种方式：

1. 用 key 作为数组下标（如 C 数组、std::vector），只适用于 key 是连续整数且范围不大（如 0~1000），否则会浪费大量空间，无法支持任意类型的 key。
2. 用数组存储所有元素，每次查找都遍历整个数组，查找/插入/删除的时间复杂度是 O(n)，效率低下。

哈希表（unordered_map）的优势在于：

- 支持任意类型的 key（只要能哈希和比较相等）
- 查找、插入、删除的平均时间复杂度是 O(1)，远快于数组的 O(n)
- 空间利用率高，不需要为每个可能的 key 都分配空间

总结：  
哈希表相比直接用数组，能高效支持大范围、任意类型的 key，并且查找/插入/删除都非常快，是实现“键值对映射”最常用的数据结构之一。

2. 理解了，也就是哈希表的查询其实是 O(1)复杂度？算出哈希，然后就能定位到是哪一个桶了？不过桶里面可能不止一个元素，还是 O(1)么？况且最坏的情况，size 为 n 的哈希，只有一个桶，里面 n 个键值对，这种情况是 O(n)吧？一般所说的常数复杂度是指 O(n)还是 O(1)？

你的理解非常准确！

- 哈希表的查找、插入、删除操作，平均情况下是 O(1) 复杂度。因为通过哈希函数可以直接定位到桶，然后在桶内查找元素。
- 但桶内可能有多个元素（哈希冲突），需要在桶内遍历，桶内元素越多，查找越慢。
- 最坏情况下，所有元素都被哈希到同一个桶（极端哈希冲突），查找就退化为 O(n)。

所以：

- 一般所说的“常数复杂度”是指**平均情况下**的 O(1)，即哈希函数分布均匀、负载因子合理时的表现。
- 最坏情况下确实是 O(n)，但实际应用中只要哈希函数设计得好，这种极端情况很少见。

总结：  
哈希表的高效性依赖于哈希函数和负载因子的合理设计，平均复杂度 O(1)，最坏复杂度 O(n)。常说的“常数复杂度”指的是平均情况。

分配器也要仔细看一下。
