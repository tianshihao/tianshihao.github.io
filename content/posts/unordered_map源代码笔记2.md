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

`std::hash<_Key>`的实现非常简单，对于内置类型，会直接调用了`std::hash`的特化版本（如果有的话），否则调用内置类型的哈希实现：

```cpp
/** @defgroup hashes Hashes
  *  @ingroup functors
  *
  *   Hashing functors taking a variable type and returning a @c std::size_t.
  *
  *  @{
  */

template<typename _Result, typename _Arg>
  struct __hash_base
  {
    typedef _Result     result_type _GLIBCXX17_DEPRECATED;
    typedef _Arg      argument_type _GLIBCXX17_DEPRECATED;
  };

// Explicit specializations for integer types.
#define _Cxx_hashtable_define_trivial_hash(_Tp) 	\
template<>						\
  struct hash<_Tp> : public __hash_base<size_t, _Tp>  \
  {                                                   \
    size_t                                            \
    operator()(_Tp __val) const noexcept              \
    { return static_cast<size_t>(__val); }            \
  };
```

例如对于`_Tp`为`int`以及所有的整数类型，`std::hash`的实现就是返回其数值本身（转换为`size_t`类型）。下面是`std::hash`所有的内置类型的特化：

| 类型               | 默认实现（文字说明）                                                       |
| ------------------ | -------------------------------------------------------------------------- |
| bool               | 直接将值转换为 `size_t` 并返回。                                           |
| char               | 直接将值转换为 `size_t` 并返回。                                           |
| signed char        | 直接将值转换为 `size_t` 并返回。                                           |
| unsigned char      | 直接将值转换为 `size_t` 并返回。                                           |
| wchar_t            | 直接将值转换为 `size_t` 并返回。                                           |
| char8_t            | 直接将值转换为 `size_t` 并返回。                                           |
| char16_t           | 直接将值转换为 `size_t` 并返回。                                           |
| char32_t           | 直接将值转换为 `size_t` 并返回。                                           |
| short              | 直接将值转换为 `size_t` 并返回。                                           |
| int                | 直接将值转换为 `size_t` 并返回。                                           |
| long               | 直接将值转换为 `size_t` 并返回。                                           |
| long long          | 直接将值转换为 `size_t` 并返回。                                           |
| unsigned short     | 直接将值转换为 `size_t` 并返回。                                           |
| unsigned int       | 直接将值转换为 `size_t` 并返回。                                           |
| unsigned long      | 直接将值转换为 `size_t` 并返回。                                           |
| unsigned long long | 直接将值转换为 `size_t` 并返回。                                           |
| 枚举类型           | 先转换为枚举的底层整数类型，然后用该类型的 hash 实现。                     |
| 指针类型 (T\*)     | 直接将指针地址转换为 `size_t` 并返回。                                     |
| float              | 如果值为 0 或 -0，返回 0；否则用 `_Hash_impl::hash` 对其二进制表示做哈希。 |
| double             | 如果值为 0 或 -0，返回 0；否则用 `_Hash_impl::hash` 对其二进制表示做哈希。 |
| long double        | 用 `_Hash_impl::hash` 对其二进制表示做哈希（具体实现见源码）。             |
| nullptr_t          | 总是返回 0。                                                               |

对于非内置类型呢？比如`std::string`而言，其会提供自己的`std::hash<std::string>`特化版本，使用字符串内容计算哈希值：

```cpp
/// std::hash specialization for string.
template<>
  struct hash<string>
  : public __hash_base<size_t, string>
  {
    size_t
    operator()(const string& __s) const noexcept
    { return std::_Hash_impl::hash(__s.data(), __s.length()); }
  };
```

对于没有默认哈希实现的类型，比如`std::vector`、`std::unordered_map`本身，是不能作`std::unordered_map`的键的。因为每个关联容器不仅需要`Key`作为参数，还需要满足哈希要求的`Hash`函数对象作为参数，该函数对象需要能接受`Key`类型的参数并返回`size_t`类型的哈希值。同时还需要二元谓词`Pred`作为参数，能对`Key`类型的值上定义一个等价关系：

> Each unordered associative container is parameterized by Key, by a function object type Hash that meets the Hash requirements (17.6.3.4) and acts as a hash function for argument values of type Key, and by a binary predicate Pred that induces an equivalence relation on values of type Key.

用户可以通过特化`std::hash`为自定义类型提供哈希实现，或者传递自定义的哈希函数对象给`std::unordered_map`的模板参数`Hash`。例如对于使用`std::vector<int>`作为键的哈希表，可以这样定义 hasher：

```cpp
#include <functional>
#include <iostream>
#include <string>
#include <unordered_map>
#include <vector>

// Method 1: Custom hasher for std::vector<int>
struct MyVectorHash {
  size_t operator()(const std::vector<int>& v) const {
    size_t h = 0;
    for (int x : v) h ^= std::hash<int>()(x) + 0x9e3779b9 + (h << 6) + (h >> 2);
    return h;
  }
};

// Method 2: Template hasher for any container whose elements are hashable
template <typename Container>
struct ContainerHash {
  size_t operator()(const Container& c) const {
    size_t h = 0;
    for (const auto& elem : c)
      h ^= std::hash<typename Container::value_type>()(elem) + 0x9e3779b9 +
           (h << 6) + (h >> 2);
    return h;
  }
};

// Method 3: Specialize std::hash for std::vector<int> in the std namespace
namespace std {
template <>
struct hash<std::vector<int>> {
  size_t operator()(const std::vector<int>& v) const {
    size_t h = 0;
    for (int x : v) h ^= std::hash<int>()(x) + 0x9e3779b9 + (h << 6) + (h >> 2);
    return h;
  }
};
}  // namespace std

void test_hasher() {
  // Method 1: Custom hasher object
  std::unordered_map<std::vector<int>, std::string, MyVectorHash> umap1;
  umap1[{1, 2, 3}] = "custom hasher";
  std::cout << "Method 1: " << umap1[{1, 2, 3}] << std::endl;

  // Method 2: Template hasher
  std::unordered_map<std::vector<int>, std::string,
                     ContainerHash<std::vector<int>>>
      umap2;
  umap2[{4, 5, 6}] = "template hasher";
  std::cout << "Method 2: " << umap2[{4, 5, 6}] << std::endl;

  // Method 3: std::hash specialization
  std::unordered_map<std::vector<int>, std::string> umap3;
  umap3[{7, 8, 9}] = "std::hash specialization";
  std::cout << "Method 3: " << umap3[{7, 8, 9}] << std::endl;
}
```

#### 3.1.3 计算桶索引

有了哈希值之后，就可以计算桶索引了。计算桶索引的方法同样被封装到了`__hash_code_base`当中，其默认值是`__detail::_Mod_range_hashing`，用户不能通过`std::unordered_map`的模板参数设置该方法。`__detail::_Mod_range_hashing`的实现非常简单，就是使用键的哈希值`__code`对桶数量`_M_bucket_count`取模，符合使用 EBO 的条件。首次插入元素时，由于桶数量为 0，桶索引总是 0。这并不是说第一个元素会被放到 0 号桶中，因为第一次插入也会触发扩容（初始负载因子为 1，超出限制），扩容之后桶的数量不为 0，此时会重新计算桶索引。

```cpp
// 默认分桶哈希函数对象
struct _Mod_range_hashing
{
  typedef std::size_t first_argument_type;
  typedef std::size_t second_argument_type;
  typedef std::size_t result_type;

  result_type
  operator()(first_argument_type __num,
        second_argument_type __den) const noexcept
  { return __num % __den; }
};


_M_emplace() {
  // ...
  size_type __bkt = _M_bucket_index(__k, __code);
}

size_type
_M_bucket_index(const key_type& __k, __hash_code __c) const
{ return __hash_code_base::_M_bucket_index(__k, __c, _M_bucket_count); }
```

#### 3.1.4 检查等价节点

在插入新节点之前，哈希表需要先判断当前桶中是否已经存在一个“等价节点”。这个检查由 `_M_find_node()` 完成：

```cpp
if (__node_type* __p = _M_find_node(__bkt, __k, __code))
  // There is already an equivalent node, no insertion
  return std::make_pair(iterator(__p), false);
```

具体实现上，`_M_find_node()` 会先调用 `_M_find_before_node()`，在指定桶内查找与目标键相等的节点的前驱节点。如果找到了，则通过前驱节点的 next 指针获取等价节点，并在 `_M_emplace()` 返回对应迭代器；如果没找到，则返回 `nullptr`，继续执行插入逻辑。

```cpp
    __node_type*
    _M_find_node(size_type __bkt, const key_type& __key,
      __hash_code __c) const
    {
__node_base* __before_n = _M_find_before_node(__bkt, __key, __c);
if (__before_n)
  return static_cast<__node_type*>(__before_n->_M_nxt);
return nullptr;
    }
```

`_M_find_before_node()` 的核心流程如下：

核心流程分为以下几步：

1. **定位桶**：

- 通过桶索引 `__bkt` 定位到对应的桶。
- 如果桶不存在，直接返回 `nullptr`，表示没有找到等价节点，可以插入新节点。

2. **遍历桶内链表**：

- 如果桶不为空，则遍历桶内的链表。
- 对每个节点，执行以下判断：
  1. 用 `_M_equals()` 检查当前节点是否与待插入节点相等。
  - 若相等，返回该节点的前驱节点，表示已存在等价节点，不能插入。
  2. 若不相等，则检查后继节点：
  - 如果后继节点存在且属于当前桶，则继续遍历。
  - 如果后继节点不存在或不在当前桶，说明已遍历完当前桶，未找到等价节点，返回 `nullptr`，可以插入新节点。

桶数组 `_M_buckets` 是一个指向桶数组的指针，每个元素都是指向链表头节点的指针，因此可以高效地定位和返回等价节点的前驱节点。

#### 3.1.5 插入新节点

如果没有找到等价节点，说明可以插入新节点。插入逻辑由 `_M_insert_unique_node()` 完成，其核心流程如下：

```cpp
template<typename _Key, typename _Value,
      typename _Alloc, typename _ExtractKey, typename _Equal,
      typename _H1, typename _H2, typename _Hash, typename _RehashPolicy,
      typename _Traits>
auto
_Hashtable<_Key, _Value, _Alloc, _ExtractKey, _Equal,
        _H1, _H2, _Hash, _RehashPolicy, _Traits>::
_M_insert_unique_node(const key_type& __k, size_type __bkt,
               __hash_code __code, __node_type* __node,
               size_type __n_elt)
-> iterator
{
  const __rehash_state& __saved_state = _M_rehash_policy._M_state();
  std::pair<bool, std::size_t> __do_rehash
   = _M_rehash_policy._M_need_rehash(_M_bucket_count, _M_element_count,
                          __n_elt);

  if (__do_rehash.first)
  {
   _M_rehash(__do_rehash.second, __saved_state);
   __bkt = _M_bucket_index(__k, __code);
  }

  this->_M_store_code(__node, __code);

  // Always insert at the beginning of the bucket.
  _M_insert_bucket_begin(__bkt, __node);
  ++_M_element_count;
  return iterator(__node);
}
```

插入节点的具体步骤如下：

1. **保存当前 rehash 状态**

- 首先通过 `const __rehash_state& __saved_state = _M_rehash_policy._M_state();` 保存当前 rehash 策略的状态，即当前要扩容的桶数量，初始是 0。
- 这样做的目的是：如果后续判断需要扩容（rehash），但是扩容失败的话，可以用这个状态恢复扩容策略之前的状态。

2. **判断是否需要扩容（rehash）**

详细见[3.1.6 重新哈希](#316-重新哈希)。

- 通过 `_M_rehash_policy._M_need_rehash(_M_bucket_count, _M_element_count, __n_elt)` 判断当前哈希表是否需要扩容。
- 其中`_M_bucket_count`是当前桶的数量，`_M_element_count`是当前元素的数量，`__n_elt`是即将插入的元素数。`_M_need_rehash`会根据这三个变量判断是否需要 rehash 以及新的桶数量。一般而言，第一次插入节点，都是需要扩容的。
- 如果需要扩容，则`_M_need_rehash`返回`make_pair(true, n)`记为`__do_rehash`，其中`n`是新的桶的数量。然后调用 `_M_rehash(__do_rehash.second, __saved_state)` 进行 rehash，并重新计算桶索引 `__bkt`。

3. **存储哈希码**

- 调用 `this->_M_store_code(__node, __code)`，将哈希码存储到节点中。

4. **插入节点到桶头**

- 始终将新节点插入到对应桶的链表头部（即 O(1) 时间复杂度）。
- 通过 `_M_insert_bucket_begin(__bkt, __node)` 完成插入。

这里把一开始的例子做一些修改来说明（[向哈希表中插入元素](#31-insert)）插入过程，假设现在向哈希表中先后插入键值对`(1, "one")`、`(4, "four")`、`(5, "five")`。那么一开始哈希表桶的数量是 1，即`_M_bucket_count = 1`，这是因为一开始有一个单桶（todo 后面解释），`_M_before_begin`为`nullptr`，`_M_buckets`为`nullptr`，`_M_element_count`为 0，初始负载因子是 1。

因此当插入第一个键值对`(1, "one")`时，会触发扩容，扩容后桶数组`_M_buckets`的大小`_M_bucket_count`变为质数 13。此时链表的头节点是`nullptr`，其前驱节点是`_M_before_begin`。

![_M_before_begin -> nullptr](/_M_insert_bucket_begin_1.svg)

扩容之后会重新计算桶索引`__bkt`。由于 key 是 1，`std::hash<int>`计算得到的哈希值也是 1，桶索引`__bkt = 1 % 13 = 1`。按照头插法，节点`(1, "one")`被插入到头节点`nullptr`之前，头节点前驱节点`_M_before_begin`之后。

![_M_before_begin -> 1 -> nullptr](/_M_insert_bucket_begin_2.svg)

此时桶数组中每个元素仍然是`nullptr`，元素还没有被放入到桶中。为了满足桶中元素（指针）指向的是桶对应链表的前驱节点，需要将`_M_buckets[1]`的值设置为`&_M_before_begin`，表示 1 号桶的链表头节点的前驱节点是`_M_before_begin`。注意，`_M_buckets[]`是指针数组，设置数组中指针的值，即让指针指向某个节点。这和`_M_before_begin._M_nxt`指向节点`(1, "one")`是两回事。此时链表的头节点变成了`(1, "one")`。

![_M_before_begin=buckets[1] -> 1 -> nullptr](/_M_insert_bucket_begin_3.svg)

接着，插入第二个键值对`(4, "four")`。其桶索引是 4，按照头插法被插入到头节点`(1, "one")`之前，头节点前驱节点`_M_before_begin`之后。此时问题出现了，节点 4 在 1 号桶中，它应该在 4 号桶中才对，而 1 号桶指向节点的后继节点是 4，也不是 1。

![_M_before_begin=buckets[1] -> 4 -> 1 -> nullptr](/_M_insert_bucket_begin_4.svg)

总结问题有两个，一个是新插入的节点不在正确的桶中，一个是前链表头节点尽管在桶中，但是桶已经不再指向前头节点的前驱节点。因此需要执行两步操作：

- 更新前头节点，为了防止误操作，需要判断前头节点不为空。
- 将新插入的节点放到正确的桶中，这一步操作和以前一样，更新`_M_buckets`即可。

所以需要执行两步，

1. 检查前首节点`(1, "one")`不为空，更新其所在桶的指针`_M_buckets[1]`，让其指向`(1, "one")`的前驱节点`(4, "four")`即`__node`。

```cpp
if (__node->_M_nxt)
  // We must update former begin bucket that is pointing to
  // _M_before_begin.
  _M_buckets[_M_bucket_index(__node->_M_next())] = __node;
```

2. 更新`_M_buckets`，让`_M_buckets[4]`指向`&_M_before_begin`，这时，其为节点 4 的前驱节点，即这么做之后`_M_buckets[4]`指向了节点 4 的前驱节点。

```cpp
_M_buckets[__bkt] = &_M_before_begin;
```

结果如下图所示，桶 1 所指链表头节点的前驱节点是节点 4，首节点是节点 1；桶 4 所指的链表头节点前驱节点是`_M_before_begin`，头节点是节点 4。这是符合预期的。

![_M_before_begin=buckets[4] -> 4 -> 1 -> nullptr](/_M_insert_bucket_begin_6.svg)

同样，继续插入节点`(8, "eight")`。一开始节点 8 会在`_M_before_begin`之后，节点 4 之前，相当于桶 4 指向`_M_before_begin`，是节点 8 的前驱节点，节点 4 变成了前头节点。因此更新桶 4，让其指向节点 8，相当于桶 4 指向了节点 4 的前驱节点。再设置桶 8，让其指向`_M_before_begin`，即桶 8 指向了节点 8 的前驱节点。结果如下图所示：

![_M_before_begin=buckets[8] -> 8 -> 4 -> 1 -> nullptr](/_M_insert_bucket_begin_7.svg)

节点`(12, "twelve")`也是一样，插入之后形成一个很有意思的结构：

![_M_before_begin=buckets[12] -> 12 -> 8 -> 4 -> 1 -> nullptr](/_M_insert_bucket_begin_8.svg)

todo，这里需要讲解插入过程，最好画图实现。

1. **更新元素计数并返回迭代器**

- `++_M_element_count;`，元素计数加一。
- `return iterator(__node);`，返回指向新节点的迭代器。

这种插入方式保证了哈希表的高效性：

- 插入始终是 O(1) 操作（不考虑 rehash）。
- 桶内链表头插入，避免遍历。
- rehash 时会重新分配桶并迁移节点，确保负载因子合理。

#### 3.1.6 重新哈希

哈希表在插入新元素时，可能会触发重新哈希（rehash）操作，以保证负载因子在合理范围内，从而维持高效的查找性能。负载因子（load factor）是衡量哈希表空间利用率和性能的关键指标，定义为：

> 负载因子 = 元素数量 / 桶数量
>
> `__builtin_ceill(__n / (long double)_M_max_load_factor)`

负载因子越高，说明每个桶平均存储的元素越多，哈希冲突概率也越大，查找效率会下降。为了保证哈希表的高效性，通常会设置一个最大负载因子（如 1.0）。当插入新元素后，负载因子超过该阈值时，就会自动进行 rehash（扩容并重新分配桶）。一般而言，在插入第一个元素时，哈希表总会进行扩容。

**`std::unordered_map` 默认采用 `_Prime_rehash_policy` 作为 rehash 策略**，**该策略生成的新的桶的数量通常是一个可以使负载因子保持足够小的质数**。有两个数据成员和一个静态数据成员：

- `float _M_max_load_factor`：最大负载因子，默认值为 1.0。
- `mutable std::size_t _M_next_resize`：下次扩容的桶数量阈值，初始为 0，表示第一次插入元素时一定会扩容。
- `static const std::size_t _S_growth_factor`：增长因子，默认值为 2。

其设计目标和主要功能如下：

- 动态判断何时需要扩容，保证负载因子不超过最大值。
- 计算合适的桶数量，并保证桶数量为质数，从而提升哈希分布的均匀性，减少冲突。
- 管理扩容阈值和内部状态，自动完成 rehash，无需用户干预。

具体流程如下：

1. **最大负载因子管理**：

- 通过 `max_load_factor()` 获取最大负载因子（默认 1.0），即每个桶允许的最大平均元素数。

2. **扩容判断**：

- 每次插入新元素前，调用 `_M_need_rehash(__n_bkt, __n_elt, __n_ins)` 判断插入后负载因子是否超标。`__n_bkt` 是当前桶数量，`__n_elt` 是当前元素数量，`__n_ins` 是即将插入的元素数。
- 如果需要扩容，返回 `make_pair(true, n)`，其中 n 是新的桶数量。不过不需要则返回 `make_pair(false, 0)`。
- 没有找到`_M_need_rehash()`的定义，只有声明。

3. **桶数量计算与选取**：

- `_M_bkt_for_elements(__n)` 根据元素数量和负载因子计算所需桶数。
- `_M_next_bkt(__n)` 返回不小于 n 的下一个质数，保证桶数量为质数，有助于哈希分布均匀。

4. **扩容与节点迁移**：

如果判断需要扩容，则调用 `_Hashtable::_M_rehash(size_type __bkt_count, const __rehash_state& __state)` 进行扩容。其核心流程如下：

```cpp
void _Hashtable::_M_rehash(size_type __bkt_count, const __rehash_state& __state) {
  try {
    _M_rehash_aux(__bkt_count, __unique_keys());
  } catch (...) {
    // 如果分配桶失败，恢复 hash policy 的旧状态并抛出异常
    _M_rehash_policy._M_reset(__state);
    throw;
  }
}
```

1. **异常安全处理**：

   - 扩容时，先保存当前 rehash policy 的状态。
   - 如果分配新桶失败（如内存不足），会恢复旧状态并重新抛出异常，保证哈希表状态不被破坏。

2. **节点迁移与链表重建**：
   - 具体迁移逻辑在 `_M_rehash_aux()` 中实现：

```cpp
void _Hashtable::_M_rehash_aux(size_type __bkt_count, true_type) {
  // 1. 分配新的桶数组
  __bucket_type* __new_buckets = _M_allocate_buckets(__bkt_count);
  // 获取链表头节点
  __node_type* __p = _M_begin();
  _M_before_begin._M_nxt = nullptr;
  std::size_t __bbegin_bkt = 0;
  // 2. 遍历所有节点
  while (__p) {
    // 记录后继节点
    __node_type* __next = __p->_M_next();
    // 使用range_hash计算新桶索引
    std::size_t __bkt = __hash_code_base::_M_bucket_index(__p, __bkt_count);
    if (!__new_buckets[__bkt]) {
      // 桶为空，将节点插入链表头
      __p->_M_nxt = _M_before_begin._M_nxt;
      _M_before_begin._M_nxt = __p;
      __new_buckets[__bkt] = &_M_before_begin;
      if (__p->_M_nxt)
        __new_buckets[__bbegin_bkt] = __p;
      __bbegin_bkt = __bkt;
    } else {
      // 桶不为空，将节点插入桶链表头
      __p->_M_nxt = __new_buckets[__bkt]->_M_nxt;
      __new_buckets[__bkt]->_M_nxt = __p;
    }
    __p = __next;
  }

  _M_deallocate_buckets();
  _M_bucket_count = __bkt_count;
  _M_buckets = __new_buckets;
}
```

迁移过程分为以下几步：

- 分配新的桶数组。
- 遍历所有节点，重新计算每个节点的桶索引，将节点插入新桶对应的链表头。
- 维护链表头指针 `_M_before_begin`，保证所有桶的链表结构正确。
- 释放旧桶数组，更新桶数量和桶指针。

这种迁移方式保证了扩容后哈希表的结构和查找效率不会受到影响。

5. **扩容阈值与状态管理**：

- `_M_state()` 和 `_M_reset()` 用于保存和恢复 rehash 策略的内部状态（如下次扩容的阈值），保证扩容逻辑的正确性。

这种设计的优点：

- 桶数量为质数，减少哈希冲突。
- 动态调整负载因子和桶数量，保证性能和空间利用率。
- 插入时自动判断是否需要扩容，无需用户手动干预。

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
