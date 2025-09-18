+++
date = '2025-09-15T15:19:36+08:00'
draft = false
title = 'unordered_map源代码笔记 3'
tags = ['C++', 'STL']
toc = true
series = ["STL源代码笔记"]
+++

# 3. 算法

## 3.2 `at()`

`at()`方法继承自`_Map_base`，使用给定的键在哈希表中查找对应的值，如果键不存在则抛出异常。

```cpp
template<typename _Key, typename _Pair, typename _Alloc, typename _Equal,
    typename _H1, typename _H2, typename _Hash,
    typename _RehashPolicy, typename _Traits>
  auto
  _Map_base<_Key, _Pair, _Alloc, _Select1st, _Equal,
      _H1, _H2, _Hash, _RehashPolicy, _Traits, true>::
  at(const key_type& __k)
  -> mapped_type&
  {
    // 获取哈希表实例
    __hashtable* __h = static_cast<__hashtable*>(this);
    // 计算键的哈希值
    __hash_code __code = __h->_M_hash_code(__k);
    // 计算键所在的桶索引
    std::size_t __bkt = __h->_M_bucket_index(__k, __code);
    // 寻找目标节点
    __node_type* __p = __h->_M_find_node(__bkt, __k, __code);

    // 如果节点不存在，抛出异常
    if (!__p)
__throw_out_of_range(__N("_Map_base::at"));
    // 返回节点的值
    return __p->_M_v().second;
  }
```

其中`_M_find_node()`调用了`_M_find_before_node()`，其使用给定的桶索引、键、哈希值查找指定节点的前驱节点。`_M_find_before_node()`会遍历指定的桶，即遍历单链表中指定桶对应的子链表，寻找其中符合要求的节点并返回，否则返回`nullptr`。

```cpp
// Find the node whose key compares equal to k in the bucket bkt.
// Return nullptr if no node is found.
template<typename _Key, typename _Value,
    typename _Alloc, typename _ExtractKey, typename _Equal,
    typename _H1, typename _H2, typename _Hash, typename _RehashPolicy,
    typename _Traits>
  auto
  _Hashtable<_Key, _Value, _Alloc, _ExtractKey, _Equal,
        _H1, _H2, _Hash, _RehashPolicy, _Traits>::
  _M_find_before_node(size_type __bkt, const key_type& __k,
    __hash_code __code) const
  -> __node_base*
  {
    // 获取对应桶，桶中元素是键__key对应子链表头节点的前驱节点
    __node_base* __prev_p = _M_buckets[__bkt];
    if (!__prev_p)
return nullptr;

    // 从子链表桶节点开始遍历
    for (__node_type* __p = static_cast<__node_type*>(__prev_p->_M_nxt);;
    __p = __p->_M_next())
{
  // 比较节点是否匹配
  if (this->_M_equals(__k, __code, __p))
    return __prev_p;

  // 如果到达链表末尾或下一个节点不在同一桶，停止查找
  if (!__p->_M_nxt || _M_bucket_index(__p->_M_next()) != __bkt)
    break;
  __prev_p = __p;
}
    return nullptr;
  }
```

`_M_equals()`从两个方面比较节点是否匹配：

1. 哈希值是否相等。
2. 键是否相等。

```cpp
  bool
  _M_equals(const _Key& __k, __hash_code __c, __node_type* __n) const
  {
    static_assert(__is_invocable<const _Equal&, const _Key&, const _Key&>{},
  "key equality predicate must be invocable with two arguments of "
  "key type");
    return _Equal_hash_code<__node_type>::_S_equals(__c, *__n)
&& _M_eq()(__k, this->_M_extract()(__n->_M_v()));
  }
```

注意`_M_find_before_node()`返回的是目标节点的前驱节点，因此`_M_find_node()`需要返回其后继节点。

## 3.3 \_\*`operator[]()`

`operator[]()`方法继承自`_Map_base`。该方法前半部分和`at()`类似，查找给定键对应的节点，如果找到则返回其值的引用。否则，创建一个新节点并插入哈希表，然后返回新节点的值引用。插入新节点之前会用`_Scoped_node`接管新节点，防止插入过程中抛出异常导致内存泄漏。

```cpp
template<typename _Key, typename _Pair, typename _Alloc, typename _Equal,
    typename _H1, typename _H2, typename _Hash,
    typename _RehashPolicy, typename _Traits>
  auto
  _Map_base<_Key, _Pair, _Alloc, _Select1st, _Equal,
      _H1, _H2, _Hash, _RehashPolicy, _Traits, true>::
  operator[](key_type&& __k)
  -> mapped_type&
  {
    __hashtable* __h = static_cast<__hashtable*>(this);
    __hash_code __code = __h->_M_hash_code(__k);
    std::size_t __bkt = __h->_M_bucket_index(__k, __code);
    // 若找到节点，直接返回其值
    if (__node_type* __node = __h->_M_find_node(__bkt, __k, __code))
return __node->_M_v().second;

    // 否则，创建新节点并插入
    typename __hashtable::_Scoped_node __node {
__h,
std::piecewise_construct,
std::forward_as_tuple(std::move(__k)),
std::tuple<>()
    };
    auto __pos
= __h->_M_insert_unique_node(__k, __bkt, __code, __node._M_node);
    __node._M_node = nullptr;
    return __pos->second;
  }
```

这里构造`_Scoped_node`的时候使用了`std::piecewise_construct`，和使用`unordered_map::insert()`进入`_Hashtable::_M_emplace()`中构造`_Scoped_node`的用法不同。`insert()`接收的参数类型是`std::pair<const Key, T>`，接收之后，直接将其完美转发给`_M_emplace()`，其中`pair`又被完美转发给了`std::pair`的构造函数。

```cpp
_Scoped_node __node { this, std::forward<_Args>(__args)...  };

// 右值作为移动构造的参数
constexpr pair(pair&&) = default;		///< Move constructor

// 左值作为复制构造的参数
template<typename _U1, typename _U2, typename
    enable_if<_PCCFP<_U1, _U2>::template
    _ConstructiblePair<_U1, _U2>()
              && _PCCFP<_U1, _U2>::template
    _ImplicitlyConvertiblePair<_U1, _U2>(),
  bool>::type=true>
  constexpr pair(const pair<_U1, _U2>& __p)
  : first(__p.first), second(__p.second) { }
```

也就是说，对于`insert()`而言，插入元素的过程，至少包含一次构造`std::pair`，一次复制或者移动`std::pair`。

但是对于`operator[]()`而言，用户并没有提供 value，只传入了 key，当哈希表中不存在该 key 时，需要新建一个`std::pair<const Key, T>`。这时有两种选择：

1. 构造一个临时的`std::pair<const Key, T>`，然后将其所有权交给`_Scoped_node`（移动），即一次构造一次移动。
2. 原地构造一个`std::pair<const Key, T>`。

显然原地构造更高效，因此`operator[]()`使用了`std::piecewise_construct`，将 key 和 value 的构造参数分开传递给`std::pair`的构造函数，供其原地构造。这个语法的意思是告诉编译器，把两个`std::tuple`（元组）中的元素作为`std::pair`构造函数的参数进行构造。

这里可能会有一个疑惑，为什么转发给`_Scoped_node`的参数既可以是整个`std::pair`，也可以是`std::piecewise_construct`加两个元组？这是因为`_Scoped_node`的构造函数是一个变参模板，可以接收任意数量和类型的参数。在该构造中，变参模板参数又被转发给了`std::pair`的构造函数。而`std::pair`的构造函数本身也支持多种重载形式，所以放心使用变参模板转发即可，GCC 提供了对应的重载。

```cpp
// Allocate a node and construct an element within it.
template<typename... _Args>
  _Scoped_node(__hashtable_alloc* __h, _Args&&... __args)
  : _M_h(__h),
    _M_node(__h->_M_allocate_node(std::forward<_Args>(__args)...))
  { }
```

使用`std::piecewise_construct`时匹配到的`std::pair`的构造函数如下：

```cpp
// 1. 首先进入这里，这个构造函数的参数是std::piecewise_construct_t和两个tuple，完全匹配
template<class _T1, class _T2>
  template<typename... _Args1, typename... _Args2>
    _GLIBCXX20_CONSTEXPR
    inline
    pair<_T1, _T2>::
    pair(piecewise_construct_t,
      tuple<_Args1...> __first, tuple<_Args2...> __second)
    // 2. 这个构造函数只做转发，内部调用了另外一个构造函数，传入了两个tuple和两个索引序列，便于从tuple中提取参数
    : pair(__first, __second,
        typename _Build_index_tuple<sizeof...(_Args1)>::__type(),
        typename _Build_index_tuple<sizeof...(_Args2)>::__type())
    { }

// 3. 后续进入了这个构造函数
template<class _T1, class _T2>
  template<typename... _Args1, std::size_t... _Indexes1,
            typename... _Args2, std::size_t... _Indexes2>
    _GLIBCXX20_CONSTEXPR inline
    pair<_T1, _T2>::
    pair(tuple<_Args1...>& __tuple1, tuple<_Args2...>& __tuple2,
      _Index_tuple<_Indexes1...>, _Index_tuple<_Indexes2...>)
    // 4. 使用索引序列从tuple中提取参数，并完美转发给first和second的构造函数
    : first(std::forward<_Args1>(std::get<_Indexes1>(__tuple1))...),
      second(std::forward<_Args2>(std::get<_Indexes2>(__tuple2))...)
    { }
```

1. 构造函数匹配过程

当使用了`std::pair<T1, T2> p(std::piecewise_construct, tuple1, tuple2);`时，编译器会优先匹配第一个构造函数（带了 `piecewise_construct_t`, `tuple`, `tuple` 参数的那个），因为参数类型完全对应。

2. 第一个构造函数的作用

它只是一个“转发”构造函数，内部调用了第二个构造函数，并通过 `_Build_index_tuple` 生成索引序列（0, 1, ...），用于后续 `tuple` 元素的解包。

3. 第二个构造函数的作用

这个构造函数用索引序列把 `tuple1` 和 `tuple2` 里的参数分别用 `std::get` 取出，并用 `std::forward` 完美转发给 `first` 和 `second` 的构造函数，实现原地构造，无需临时对象、无复制或移动。

具体来说：

```cpp
: first(std::forward<_Args1>(std::get<_Indexes1>(__tuple1))...),
  second(std::forward<_Args2>(std::get<_Indexes2>(__tuple2))...)
```

这就是把 `tuple1` 的每个元素作为参数传给 `T1` 的构造函数，`tuple2` 的每个元素作为参数传给 `T2` 的构造函数。

另外有些语法比较奇怪，例如

```cpp
typename _Build_index_tuple<sizeof...(_Args1)>::__type(),
typename _Build_index_tuple<sizeof...(_Args2)>::__type())
```

它的意思是从`_Build_index_tuple`模板类中提取嵌套类型`__type`，该类型是一个索引序列类型，表示从 0 开始的整数序列，长度等于传入的模板参数个数。这个索引序列用于从`tuple`中按顺序提取元素。

人话，假设有个`tuple`为`std::tuple<int, double, std::string>`，那么此时`sizeof...(_Args1)`就是 3，`_Build_index_tuple<3>::__type`会生成一个类型，就是`_Index_tuple<0, 1, 2>`。

在`std::pair`的构造函数中，使用这个索引序列配合`std::get`，就能依次从`tuple`中提取出第 0、1、2 个元素，传递给`first`和`second`的构造函数。展开就是类似

```cpp
first(std::forward<Args1>(std::get<0>(tuple1)),
      std::forward<Args1>(std::get<1>(tuple1)),
      std::forward<Args1>(std::get<2>(tuple1)))
second(std::forward<Args2>(std::get<0>(tuple2)))
```

另外，既然要用参数直接构造`pair`，为什么要使用 tuple 传递参数呢？假设现在有个`std::vector<Foo>`，要向其中插入一个`Foo`对象，要原地构造可以使用`emplace_back()`，它的参数是`Foo`的构造函数参数列表：

```cpp
struct Foo {
    Foo(int x, double y, const std::string& z);
    int a;
    double b;
    std::string c;
};

std::vector<Foo> vec;
vec.emplace_back(42, 3.14, "hi"); // 直接传递
```

但是`pair`的情况比较特殊，`pair`有两个成员`first`和`second`，它们可能是不同类型的对象，并且各自的构造函数参数列表也可能不同。例如

```cpp
struct Bar{
  Bar(int x, int y) : a(x), b(y) {}
  int a;
  int b;
};

std::unordered_map<std::string, Bar> umap;

umap.emplace("hello", 1, 2); // NG
```

这种情况下，无法区分哪个参数是传递给`std::string`的，哪个是传递给`Bar`的。是`std::string("hello", 1), Bar(2)`还是`std::string("hello"), Bar(1, 2)`？

因此需要引入一种机制让构造函数既可以接收变参模板参数，又能区分哪些参数是传递给`first`的，哪些是传递给`second`的。好，那现在有两种思路：

1. 使用`{}`初始化列表，传递两个`std::initializer_list`，分别对应`first`和`second`的参数列表。
2. 用容器把参数分开传递，`std::tuple`是一个很好的选择。

GCC 实际选择的是第二种，使用`std::tuple`把给`first`和`second`的参数分开。当然这么做也会引入歧义，例如

```cpp
umap.emplace(std::make_tuple("hello"), std::make_tuple(1, 2)); // Still NG
```

这种写法无法区分`std::unordered_map<std::string, Bar>`和`std::unordered_map<std::tuple<std::string>, std::tuple<int, int>>`。因此，C++11 还引入了`std::piecewise_construct_t`来解决这个歧义。`std::piecewise_construct_t`是一个空类，全局变量`std::piecewise_construct`就是该类型的一个变量，仅作为一个标志，提示将`std::tuple`中的元素解包出来，转发给`first`和`second`的构造函数。结合完美转发，就能实现`std::pair`的原地构造。

```cpp
/// Tag type for piecewise construction of std::pair objects.
struct piecewise_construct_t { explicit piecewise_construct_t() = default; };

/// Tag for piecewise construction of std::pair objects.
_GLIBCXX17_INLINE constexpr piecewise_construct_t piecewise_construct =
  piecewise_construct_t();
```

因此就变成了：

```cpp
struct Bar {
  Bar(int x, int y) : a(x), b(y) {}
  int a;
  int b;
};

std::unordered_map<std::string, Bar> umap;

umap.emplace(std::piecewise_construct, std::make_tuple("hello"),
              std::make_tuple(1, 2));  // OK
```

考虑到`std::make_tuple`会 decay 参数类型，导致`const`和引用丢失，造成不必要的拷贝或移动，C++14 引入了`std::forward_as_tuple`，它不会 decay 参数类型，可以保留`const`和引用（右值性），从而避免不必要的拷贝或移动，可以实现真正的原地构造。这样用`std::forward_as_tuple`包装一个右值，这个右值就能被无损地传递给`std::pair`的构造函数。

因此最终方案就是：

```cpp
umap.emplace(std::piecewise_construct, std::forward_as_tuple("hello"),
              std::forward_as_tuple(1, 2));  // OK
```

这也是 STL 源代码的选择。

> 关于`std::tuple`的使用可以参考另一篇笔记：[如何使用 std::tuple](./如何使用std::tuple.md)。

# 附录

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
