+++
date = '2025-08-18T01:40:40+08:00'
draft = false
title = 'unordered_map源代码阅读'
tags = ['C++', 'STL']
toc = true
+++

# `std::unordered_map`源代码阅读

## 1. 概括

`std::unordered_map` 是一种关联容器，内部存储唯一 key 的键值对。查找、插入和删除元素的操作平均具有常数时间复杂度。

在内部，元素不会按照任何特定顺序排列，而是根据 key 的哈希值被组织到不同的桶（bucket）中。元素被放入哪个桶，完全取决于其 key 的哈希值。具有相同哈希值的 key 会出现在同一个桶里。这种设计使得一旦计算出哈希值，就能快速定位到包含该元素的桶，从而实现对单个元素的高效访问。

当 map 的 key 等价谓词（key equality predicate）对两个 key 返回 true 时，这两个 key 被认为是等价的。如果两个 key 等价，那么哈希函数也必须为它们返回相同的哈希值。

`std::unordered_map`本质上就是一个基于哈希桶数组实现的哈希表。

### 1.1 `std::unordered_map`和`__umap_hashtable`的定义

先观察一下`std::unordered_map`的定义：

```cpp
  template<typename _Key, typename _Tp,
     typename _Hash = hash<_Key>,
     typename _Pred = equal_to<_Key>,
     typename _Alloc = allocator<std::pair<const _Key, _Tp>>>
    class unordered_map
    {
      // 可以看到这里使用的其实是模板__umap_hashtable
      typedef __umap_hashtable<_Key, _Tp, _Hash, _Pred, _Alloc>  _Hashtable;
      _Hashtable _M_h;
    }

```

这些模板参数的含义分别是：

- `_Key`：映射中键的类型。
- `_Tp`：映射中值的类型。
- `_Hash`：用于计算哈希值的函数对象。
- `_Pred`：用于比较键是否相等的函数对象，即所谓的等价谓词（Predicate）。
- `_Alloc`：用于分配内存的分配器类型。

其中用户必须指定的模板参数是`_Key`和`_Tp`，其他的模板参数都有默认值。

还可以发现`std::unordered_map`持有的实际是`__umap_hashtable`，继续看`__umap_hashtable`的定义，发现`__umap_hashtable`是`_Hashtable`实例化的别名，**所以实际上的哈希表是`_Hashtable`**。通过构造`__umap_hashtable`把`std::unordered_map`的模板参数传递给了`_Hashtable`。

```cpp
  /// Base types for unordered_map.
  template<bool _Cache>
    using __umap_traits = __detail::_Hashtable_traits<_Cache, false, true>;

  template<typename _Key,
     typename _Tp,
     typename _Hash = hash<_Key>,
     typename _Pred = std::equal_to<_Key>,
     typename _Alloc = std::allocator<std::pair<const _Key, _Tp> >,
     typename _Tr = __umap_traits<__cache_default<_Key, _Hash>::value>>
    // 这里，__umap_hashtable是_Hashtable实例化的别名
    using __umap_hashtable = _Hashtable<_Key, std::pair<const _Key, _Tp>,
                                        _Alloc, __detail::_Select1st,
                _Pred, _Hash,
                __detail::_Mod_range_hashing,
                __detail::_Default_ranged_hash,
                __detail::_Prime_rehash_policy, _Tr>;
```

### 1.2 `_Hashtable`的定义

继续看`_Hashtable`的类型，`_Hashtable`的类型定义在`/usr/include/c++/10/bits/hashtable.h`里面。

```cpp
  template<typename _Key, typename _Value, typename _Alloc,
     typename _ExtractKey, typename _Equal,
     typename _H1, typename _H2, typename _Hash,
     typename _RehashPolicy, typename _Traits>
    class _Hashtable
    : public __detail::_Hashtable_base<_Key, _Value, _ExtractKey, _Equal,
				       _H1, _H2, _Hash, _Traits>,
      public __detail::_Map_base<_Key, _Value, _Alloc, _ExtractKey, _Equal,
				 _H1, _H2, _Hash, _RehashPolicy, _Traits>,
      public __detail::_Insert<_Key, _Value, _Alloc, _ExtractKey, _Equal,
			       _H1, _H2, _Hash, _RehashPolicy, _Traits>,
      public __detail::_Rehash_base<_Key, _Value, _Alloc, _ExtractKey, _Equal,
				    _H1, _H2, _Hash, _RehashPolicy, _Traits>,
      public __detail::_Equality<_Key, _Value, _Alloc, _ExtractKey, _Equal,
				 _H1, _H2, _Hash, _RehashPolicy, _Traits>,
      private __detail::_Hashtable_alloc<
	__alloc_rebind<_Alloc,
		       __detail::_Hash_node<_Value,
					    _Traits::__hash_cached::value>>>,
      private _Hashtable_enable_default_ctor<_Equal, _H1, _Alloc>
    {
    private:
      __bucket_type*		_M_buckets		= &_M_single_bucket;
      size_type			_M_bucket_count		= 1;
      __node_base		_M_before_begin;
      size_type			_M_element_count	= 0;
      _RehashPolicy		_M_rehash_policy;

      // A single bucket used when only need for 1 bucket. Especially
      // interesting in move semantic to leave hashtable with only 1 bucket
      // which is not allocated so that we can have those operations noexcept
      // qualified.
      // Note that we can't leave hashtable with 0 bucket without adding
      // numerous checks in the code to avoid 0 modulus.
      __bucket_type		_M_single_bucket	= nullptr;
    }
```

`_Hashtable`的定义乍一看很复杂，但大体可以分 3 个角度来解读：模板参数、继承、数据成员。

#### 1.2.1 模板参数

其中`_Hashtable`模板参数的含义分别是：

- `_Key`：映射中键的类型。
- `_Value`：映射中值的类型。类型一般是`std::pair<const _Key, _Tp>`，见`__umap_hashtable`定义。
- `_Alloc`：用于分配内存的分配器类型。
- `_ExtractKey`：用于从值中提取键的函数对象。
- `_Equal`：用于比较键是否相等的函数对象。
- `_H1`、`_H2`：用于计算哈希值的函数对象。
- `_Hash`：用于计算哈希值的函数对象。
- `_RehashPolicy`：用于控制重新哈希策略的类型。
- `_Traits`：用于提供额外类型信息的特性类。

再结合上面`__umap_hashtable`的定义可以发现，`__umap_hashtable`是一个特定配置的`_Hashtable`，而`std::unordered_map`持有`__umap_hashtable`，因此`std::unordered_map`是一个特定配置的`_Hashtable`。

STL 正是通过这种方式，把底层`_Hashtable`复杂的模板参数都封装到了`__umap_hashtable`里面，然后`std::unordered_map`只暴露给用户最常用、最需要关心的参数（键类型、值类型、哈希函数、等价谓词和分配器）。这么做隐藏了实现细节，简化了接口。

或者还一种说法：STL 通过 typedef/using 机制，将底层复杂的 `_Hashtable` 封装为 `__umap_hashtable`，并由`std::unordered_map` 持有，从而只暴露用户关心的参数，隐藏实现细节，简化了接口。

#### 1.2.2 继承

`unordered_map` 封装的 `_Hashtable` 继承结构非常复杂，主要是为了实现高效、灵活且可扩展的哈希表。

**总`_Hashtable` 继承结构的设计目的**

`_Hashtable` 通过多重继承，将哈希表的各个功能模块（如基础操作、插入、重哈希、分配器管理等）分离到不同的基类中。这种设计有助于代码复用、功能解耦和灵活扩展，同时也便于针对不同容器（如 `unordered_map`、unordered_set）进行定制。

**主要基类的作用**

1.  **`__detail::_Hashtable_base`**  
    负责哈希表的基础操作，如键值提取、哈希函数、相等比较等。它为哈希表的查找、哈希分布等提供了核心逻辑。

2.  **`__detail::_Map_base`**  
    为 map 类型的哈希表（如 `unordered_map`）提供特有的接口和行为，比如 `operator[]`、`at()` 等。它区分了 map 和 set 的不同需求。

3.  **`__detail::_Insert`** （在 GCC-15 去掉了）
    封装了元素插入相关的逻辑，包括插入单个元素、批量插入、插入时的哈希冲突处理等。这样插入逻辑可以独立于其他功能进行维护和优化。

4.  **`__detail::_Rehash_base`**  
    负责 rehash（重哈希）相关的策略和实现。当哈希表负载因子过高时，自动扩容并重新分布元素。

5.  **`__detail::_Hashtable_alloc`**  
    私有继承，用于管理哈希表节点的内存分配。通过分配器适配器（如 `__alloc_rebind`）实现 allocator 的灵活配置。

这种多重继承的设计，将哈希表的各个功能模块化，便于代码复用和维护。每个基类只关注自己的职责，减少了耦合，提高了可读性和可扩展性。对于不同类型的哈希容器（如 map、set），只需组合不同的基类即可实现定制化行为。这也是 STL 容器内部常用的“策略模式”与“分层继承”思想的体现。

#### 1.2.3 数据成员

主要的数据成员为：

```cpp
    {
    private:
        __bucket_type*    _M_buckets    = &_M_single_bucket; // 桶数组
        size_type         _M_bucket_count = 1; // 桶的数量，初始会有一个空桶，所以大小是1
        __node_base       _M_before_begin; // 头节点，指向第一个元素的前一个节点，todo，后面补充
        size_type         _M_element_count = 0; // hashtable中元素的数量，即size
        _RehashPolicy     _M_rehash_policy; // rehash策略

        // A single bucket used when only need for 1 bucket. Especially
        // interesting in move semantic to leave hashtable with only 1 bucket
        // which is not allocated so that we can have those operations noexcept
        // qualified.
        // Note that we can't leave hashtable with 0 bucket without adding
        // numerous checks in the code to avoid 0 modulus.
        __bucket_type   _M_single_bucket  = nullptr; // 单个桶的指针，初始时只有一个空桶
    }
```

## 2. 数据结构

### 2.1 节点

哈希表不可避免地会遇到哈希冲突：不同的 key 经过哈希函数后，可能会被分配到同一个桶。为了解决冲突，`std::unordered_map`（底层 `_Hashtable`）采用**链表法**，即每个桶存储一个链表的头指针，链表中的每个节点存储一个实际的键值对，并通过 next 指针串联。

这些节点的实现并不是简单的结构体，而是通过**继承体系**来设计的：

1. **节点基类 `__node_base`**：只包含链表结构所需的最基本成员（如 next 指针），为所有节点类型提供统一的接口。
2. **实际节点类型 `__node_type`**：继承自 `__node_base`，在其基础上增加了存储 key-value 对等成员。需要特别指出的是，`__node_type` 就是链表节点的实际类型。

桶数组的类型 `__bucket_type` 实际上也是 `__node_base*`，即节点基类的指针类型，而不是直接指向 `__node_type`。这样设计的好处是：

- 桶数组可以用统一的基类指针来管理所有实际的节点类型，实现多态和灵活扩展。
- 即使将来节点类型有变化，主逻辑也无需修改，只需依赖基类接口即可。

这种继承和指针抽象，使得哈希表的节点管理既高效又灵活。

#### 2.1.1 `__node_base`

`__node_base` 是所有哈希表节点的基类，主要用于实现链表结构，就是个简单的链表节点，其定义如下：

```cpp
  // 节点基类，只包含 next 指针，一个很简单的链表节点
  struct _Hash_node_base {
    _Hash_node_base* _M_nxt;

    _Hash_node_base() noexcept : _M_nxt(nullptr) { }
    _Hash_node_base(_Hash_node_base* __next) noexcept : _M_nxt(__next) { }
  };

  // _Hashtable_alloc 负责分配内存，管理节点的生命周期
  template<typename _NodeAlloc>
    struct _Hashtable_alloc : private _Hashtable_ebo_helper<0, _NodeAlloc>
    {
      using __node_base = __detail::_Hash_node_base;
      using __bucket_type = __node_base*;
    }
```

这个基类只包含一个指向下一个节点的指针 `_M_nxt`，用于将同一个桶内的节点串成链表。所有实际存储数据的节点类型最终都会继承自这个基类。这样，哈希表可以用基类指针统一管理所有节点，实现链表操作的多态性和灵活性。

#### 2.1.2 `_Hash_node_value_base`

`_Hash_node_value_base`继承自 `_Hash_node_base`，即`__node_base`，在`__node_base`的基础上多了一个存储键值对的成员，但它并不是实际的节点类型`__node_type`，而是中间的过渡类型。里面的`_Value`其实就是`std::unordered_map`中`Key`和`_Tp`构成的`pair`，类型为`std::pair<const _Key, _Tp>`，在前面[`std::unordered_map`和`__umap_hashtable`的定义](#stdunordered_map和__umap_hashtable的定义)中可以看到，它是`std::unordered_map`实际使用的`_Hashtable`实例的别名。

```cpp
  /**
   *  struct _Hash_node_value_base
   *
   *  Node type with the value to store.
   */
  template<typename _Value>
    struct _Hash_node_value_base : _Hash_node_base
    {
      typedef _Value value_type;

      __gnu_cxx::__aligned_buffer<_Value> _M_storage;

      _Value*
      _M_valptr() noexcept
      { return _M_storage._M_ptr(); }

      _Value&
      _M_v() noexcept
      { return *_M_valptr(); }

      // ...
    };
```

#### 2.1.3 `__node_type`

`__node_type` 是哈希表链表节点的实际类型，其类型为 `_Hash_node`的特化版本，特化版本继承自 `_Hash_node_value_base`，其定义如下：

```cpp
  /**
   *  Primary template struct _Hash_node.
   */
  template<typename _Value, bool _Cache_hash_code>
    struct _Hash_node;

  /**
   *  Specialization for nodes with caches, struct _Hash_node.
   *
   *  Base class is __detail::_Hash_node_value_base.
   */
  template<typename _Value>
    struct _Hash_node<_Value, true> : _Hash_node_value_base<_Value>
    {
      std::size_t  _M_hash_code;

      _Hash_node*
      _M_next() const noexcept
      { return static_cast<_Hash_node*>(this->_M_nxt); }
    };

  /**
   *  Specialization for nodes without caches, struct _Hash_node.
   *
   *  Base class is __detail::_Hash_node_value_base.
   */
  template<typename _Value>
    struct _Hash_node<_Value, false> : _Hash_node_value_base<_Value>
    {
      _Hash_node*
      _M_next() const noexcept
      { return static_cast<_Hash_node*>(this->_M_nxt); }
    };

  using __node_type = __detail::_Hash_node<_Value, __hash_cached::value>;
```

`_Hash_node`添加了`_M_next()`方法，用于获取下一个节点的指针。至此，`_Hash_node`的接口就完整了，可以被链表使用了。

到这里，梳理一下派生顺序：

- `__node_base`，基类，提供最基本的 next 指针，实现链表的功能。
- `_Hash_node_value_base`，中间基类，增加了存储键值对的成员。
- `_Hash_node`最后的派生类，实际节点类型，提供访问链表 next 指针的接口。

> 另外，注意`_Hash_node`的声明没有继承任何类，但是特化继承了`_Hash_node_value_base`，这是模板元编程的一个特性，允许在特化时改变继承关系。不仅特化可以继承，相同模板的不同特化还可以继承不同的类。因为模板的本质就是替换。
>
> 从替换的角度理解模板特化可能会困扰，因为都是同一个模板的特化，替换之后名字是相同的，结果两个特化的实例继承的却是不同的类，这难道不会冲突么？在普通类上是看不到这种现象的。但其实不用担心，因为模板特化的实例化是独立的，编译器会为每个特化生成独立的类型信息，所以不会冲突。

`_Hash_node`的第一个模板参数`_Value`类型也是`std::pair<const _Key, _Tp>`，即存储的键值对类型。

`_Hash_node`的第二个模板参数 `__hash_cached::value` 决定了节点类型是否缓存哈希值。

`_Hash_node` 的定义分为两种特化：

1. **带哈希缓存的节点**（`_Hash_node<_Value, true>`）：

   - 继承自 `_Hash_node_value_base<_Value>`，后者又继承自 `_Hash_node_base`。
   - 除了存储键值对，还多了一个 `std::size_t _M_hash_code` 成员，用于缓存该节点的哈希值。
   - 提供 `_M_next()` 方法，返回下一个节点指针（类型安全地转换）。

2. **不带哈希缓存的节点**（`_Hash_node<_Value, false>`）：
   - 继承自 `_Hash_node_value_base<_Value>`，后者又继承自 `_Hash_node_base`。
   - 没有 `_M_hash_code` 成员。
   - 也有 `_M_next()` 方法。

**hash code 缓存与 cache 参数说明：**

- `__hash_cached::value` 决定是否为每个节点缓存哈希值（即 `_M_hash_code`）。
- 如果缓存，查找和重哈希时可以直接用节点里的哈希值，无需重新计算，提高效率。
- 是否缓存由容器的 traits 决定（如 key 类型、哈希函数等），它 traits 在`__umap_traits`中设置（可以参考[`std::unordered_map`和`__umap_hashtable`的定义](#stdunordered_map和__umap_hashtable的定义)），其值其实是由一系列模板元编程逻辑和类型推导在编译器自动计算出来的，用户不可以直接决定。
- 这种设计让节点结构在空间和效率之间灵活权衡。

总结：

- `__node_type` 既存储键值对，也可能存储哈希值（如果开启缓存）。
- 桶数组存储的是 `__node_base*`，实际指向 `__node_type`。
- 通过继承体系和模板特化，STL 实现了节点结构的灵活扩展和高效访问。
- `_Hash_node_base`（`__node_base`）派生`_Hash_node_value_base`派生`_Hash_node`（`__node_type`）。

##### 如何决定`_Hash_node`是否缓存哈希值？

这一过程完全由类型萃取（traits）和模板参数自动推导，无需用户干预。其核心逻辑如下：

1. **传递 traits 给`_Hashtable`**

- `__umap_hashtable` 是 `_Hashtable` 的别名，最后一个模板参数 `_Tr` 实际上传递的是 `__umap_traits<__cache_default<_Key, _Hash>::value>`，`_Tr`会作为`_Hashtable`的最后一个模板参数`_Traits`传递进去。
- `__umap_traits` 是 `__detail::_Hashtable_traits<_Cache, false, true>` 的别名，三个参数分别控制：是否缓存哈希值、是否 constant 迭代器、key 是否唯一，即`_Hashtable`的`_Tr`为`__detail::_Hashtable_traits<>`的`_Cache`。
- 在`__umap_traits`的定义中，`_Cache` 由 `__cache_default<_Key, _Hash>::value` 决定，`__cache_default` 是 type traits，`::value` 取出布尔值。

即`__umap_traits`的第一个模板参数`_Cache`由`_Key`和`_Hash`通过`__cache_default`计算决定，其值为`__cache_default::value`，作为`_Tr`传递给`_Hashtable`的`_Traits`。后两个模板参数固定为 `false` 和 `true`。

重点在`_Hashtable_traits`中第一个参数`_Cache`的计算。

2. **`_Hashtable_traits`的作用**

- `struct _Hashtable_traits`是一个特性类，里有三个成员类型：
  - `__hash_cached`：是否缓存哈希值，通过`__cache_default`计算得到。
  - `__constant_iterators`：是否 constant 迭代器
  - `__unique_keys`：key 是否唯一

```cpp
  // __cache_default计算得到_Cache，传递给_Hashtable_traits
  template<bool _Cache_hash_code, bool _Constant_iterators, bool _Unique_keys>
    struct _Hashtable_traits
    {
      using __hash_cached = __bool_constant<_Cache_hash_code>;
      using __constant_iterators = __bool_constant<_Constant_iterators>;
      using __unique_keys = __bool_constant<_Unique_keys>;
    };
```

- 这些成员类型都是 `__bool_constant<...>`，即 `std::integral_constant<bool, ...>`，有静态成员 `value`，值为传入的模板参数。

3. **`_cache_default`的逻辑**

- `__cache_default<_Key, _Hash>` 的定义：

  ```cpp
  template<typename _Tp, typename _Hash>
  using __cache_default = __not_<__and_<
    __is_fast_hash<_Hash>,
    __is_nothrow_invocable<const _Hash&, const _Tp&>
  >>;
  ```

- 逻辑：只有当哈希函数既快又 `noexcept` 时，节点**不缓存**哈希值（节省空间）；否则**缓存**（提高效率/安全）。
- 例如：

  - `Key=int` 且 `Hash=std::hash<int>`，则 `__is_fast_hash` 和 `__is_nothrow_invocable` 都为 true，最终 `__cache_default` 为 false，不缓存。
  - `Key=自定义类型` 或 `Hash=慢/可能抛异常`，则 `__cache_default` 为 true，缓存。

- traits 在 STL 语境下通常译为“特性类”或“类型萃取”，也可理解为“类型属性集合”。
- `__umap_traits` 可译为“unordered_map 的特性类”或“unordered_map 的类型萃取”。

4. **`_Hashtable`内部的节点类型推导**

拿到完整的`_Hashtable_traits`即`_Traits`后，就可以在`_Hashtable`里面使用了：

```cpp
using __traits_type = _Traits; // `__umap_hashtable`的_Tr
using __hash_cached = typename __traits_type::__hash_cached;
using __node_type = __detail::_Hash_node<_Value, __hash_cached::value>;
```

- 这里 `__hash_cached::value` 就是最终决定节点类型是否缓存哈希值的布尔值，即前面提到的 `__cache_default` 求出的值。

5. **总结**

- 这套设计让类型信息和行为（如是否缓存哈希值）在模板参数层面自动推导，用户无需关心，代码更灵活高效。

#### 2.1.4 `_Scoped_node`

然而，在实际操作过程中，比如`std::unordered_map`的`insert()`（对应`_Hashtable()`的`_M_insert()`或者`_M_emplace()`），并不会直接对`__node_type`进行操作，而是使用一个名为 `_Scoped_node` 的 RAII 类来管理节点的生命周期。它定义在`_Hashtable`内部，其定义如下：

```cpp
      // Simple RAII type for managing a node containing an element
      struct _Scoped_node
      {
  // Take ownership of a node with a constructed element.
  _Scoped_node(__node_type* __n, __hashtable_alloc* __h)
  : _M_h(__h), _M_node(__n) { }

  // Allocate a node and construct an element within it.
  template<typename... _Args>
    _Scoped_node(__hashtable_alloc* __h, _Args&&... __args)
    : _M_h(__h),
      _M_node(__h->_M_allocate_node(std::forward<_Args>(__args)...))
    { }

  // Destroy element and deallocate node.
  ~_Scoped_node() { if (_M_node) _M_h->_M_deallocate_node(_M_node); };

  _Scoped_node(const _Scoped_node&) = delete;
  _Scoped_node& operator=(const _Scoped_node&) = delete;

  __hashtable_alloc* _M_h;
  __node_type* _M_node;
      };
```

`_Scoped_node`的设计是典型的 RAII，有两个数据成员，两种构造方式。

##### 2.1.4.1 数据成员

- `__hashtable_alloc* _M_h`：指向分配器，用于分配和释放节点。因为`_Hashtable`的资源管理全部使用 Allocator，所以这个指针是必须的。
  - `_Scoped_node`在`_Hashtable`内部使用，而`_Hashtable`继承了`_Hashtable_alloc`，因此这也是为什么在构造`_Scoped_node`的时候传入`_Hashtable`的`this`指针作为`__hashtable_alloc`的原因。
- `__node_type* _M_node`：指向的节点。

##### 2.1.4.2 构造函数

**1. 直接接管已有节点**

这种方式构造方式主要用于“接管”一个已经分配并构造好的`__node_type`节点，`_Scoped_node`负责其后续的释放。由于`_Hashtable`的资源管理全部使用 Allocator，因此为了能够正确释放资源，`_Scoped_node`需要`_Hashtable`的 Allocator 指针。

常用于`insert()`等操作中，已经有了一个节点（可能是通过`_M_allocate_node()`分配的），现在需要将其交给`_Scoped_node`来管理。todo 如何接管呢？

```cpp
_Scoped_node(__node_type* __n, __hashtable_alloc* __h)
: _M_h(__h), _M_node(__n) { }
```

**2. 分配新节点并构造元素**

这种构造方式会通过分配器`__h`，传递参数`__args`，直接分配并构造一个新的`__node_type`节点。这里的`__args`是构造节点时需要的参数，通常是键值对。

常用于`emplace()`操作中，需要新建一个节点并在其中构造元素。

```cpp
template<typename... _Args>
_Scoped_node(__hashtable_alloc* __h, _Args&&... __args)
: _M_h(__h),
  // 通过 allocator的 _M_allocate_node 分配一个新的节点，并构造元素
  _M_node(__h->_M_allocate_node(std::forward<_Args>(__args)...))
{ }
```

##### 2.1.4.3 其它主要函数

1. `_Scoped_node`的析构函数也是通过 allocator 释放节点的。
2. `_Scoped_node`禁止复制操作，这样设计是为了防止同一个节点被多个`_Scoped_node`管理，避免重复释放或悬垂指针，确保资源的唯一所有权。这是典型的 RAII 独占语义。
   1. todo，`__node_type`是可以被复制的么？`__node_base`持有一个指针，是 trivial 的，`_Hash_node_value_base`持有的`typedef _Value value_type;`是 trivial 的么？
3. 虽然`_Scoped_node`没有显示定义移动构造函数和移动赋值运算符，但由于其成员都是指针类型，编译器会自动生成默认的移动构造和移动赋值操作。这样可以安全地转移节点所有权。

### 2.2 分配器

前面讲了哈希表中节点的定义、继承关系、数据成员，以及管理其生命周期的 `_Scoped_node` RAII 类。`Scoped_node`负责管理`__node_type`的生命周期，在其构造函数中，使用分配器完成内存分配和`__node_type`的构造。

`Scoped_node`继承了`_Hashtable_alloc`，因此在构造`Scoped_node`的时候直接传入`_Hashtable`的`this`指针作为分配器。

`Scoped_node`继承的`_Hashtable_alloc`其实是一个分配器适配器，它通过`__alloc_rebind`将用户传入的分配器类型（如`std::allocator<std::pair<const _Key, _Tp>>`）重新绑定为适用于节点类型（`_Hash_node`）的分配器类型。这样，哈希表就可以使用用户指定的分配器来管理节点的内存。

### 2.3 insert

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
