+++
date = '2025-09-17T16:24:13+08:00'
draft = false
title = 'unordered_map源代码笔记 4'
tags = ['C++', 'STL']
toc = true
series = ["STL源代码笔记"]
+++

# 3. 算法

## 3.4 `emplace()`

`emplace()`的实现和[`insert()`](./unordered_map源代码笔记2.md#312-构造_scoped_node)是完全一样，两个函数最终都会调用`_Hashtable::_M_emplace()`进而使用`_Hashtable::_M_insert_unique_node()`插入新节点。它们二者的区别在于接收的参数不同，`emplace()`接收的是参数包，其中是构造哈希表底层`pair`所需的参数。哈希表可以使用这些参数原地构造节点。而`insert()`接收的是一个已经构造好的`std::pair`，多了`pair`的构造这一骤。

```cpp
struct Person {
  std::string name{"unknown"};
  std::string address{"unknown"};

  Person() {
    std::cout << "Default constructor called for Person\n";
    std::cout << "  Name: " << name << ", Address: " << address << '\n';
  }
  Person(Person const& other) : name(other.name), address(other.address) {
    std::cout << "Copy constructor called for Person\n";
    std::cout << "  Name: " << name << ", Address: " << address << '\n';
  }
  Person(Person&& other) noexcept
      : name(std::move(other.name)), address(std::move(other.address)) {
    std::cout << "Move constructor called for Person\n";
    std::cout << "  Name: " << name << ", Address: " << address << '\n';
  }

  Person(std::string const& n, std::string const& a) noexcept
      : name(n), address(a) {
    std::cout << "Parameterized constructor called for Person\n";
    std::cout << "  Name: " << name << ", Address: " << address << '\n';
  }

  friend std::ostream& operator<<(std::ostream& os,
                                  const Person& person) noexcept {
    return os << "Name: " << person.name << ", Address: " << person.address;
  }
};

void Emplace() {
  using Code = std::string;
  std::unordered_map<Code, Person> umap;

  umap.insert({"Alice", Person("Alice Abraham", "789 Pine St")});
  umap.emplace("Bob", Person("Bob Blake", "123 Main St"));
  umap.emplace(std::piecewise_construct, std::forward_as_tuple("Charlie"),
               std::forward_as_tuple("Charlie Chapman", "456 Oak St"));

  // NG
  // umap.insert(std::piecewise_construct, std::forward_as_tuple("Charlie"),
  //             std::forward_as_tuple("Charlie", "101 Maple St"));
  // NG
  // umap.emplace("David", "David", "999 Elm St"); // 直接传递参数
}
```

这段代码的输出是：

```bash
Parameterized constructor called for Person
  Name: Alice Abraham, Address: 789 Pine St
Move constructor called for Person
  Name: Alice Abraham, Address: 789 Pine St
Move constructor called for Person
  Name: Alice Abraham, Address: 789 Pine St
Parameterized constructor called for Person
  Name: Bob Blake, Address: 123 Main St
Move constructor called for Person
  Name: Bob Blake, Address: 123 Main St
Parameterized constructor called for Person
  Name: Charlie Chapman, Address: 456 Oak St
```

对于`insert()`，需要先构造一个临时的`Person`对象，再将其移动到临时的`std::pair`中，最后再将`pair`移动到哈希表节点中。总共涉及一次构造和两次移动构造。

而对于`emplace()`，可以直接将构造哈希表底层`pair`节点所需的原料传递给哈希表。例如`umap.emplace("Bob", Person("Bob", "123 Main St")`，这里首先构造了一个临时的`Person`对象，然后`std::string`和`Person`分别被转发给`std::pair`，用于构造`first`和`second`成员，所以`Person`被移动了一次。这种方式需要一次构造和一次移动构造。

甚至通过`std::piecewise_construct`，`emplace()`连`Person`的移动构造都可以省略。例如`umap.emplace(std::piecewise_construct, std::forward_as_tuple("Charlie"), std::forward_as_tuple("Charlie", "456 Oak St"))`，这里`std::piecewise_construct`告诉哈希表分别将`std::tuple`中的元素传递给`std::pair`的每个成员进行原地构造，所以`Person`只被构造了一次，没有任何移动构造，效率最高。

仔细看一下调用栈，对于`insert()`：

```cpp
// 1. 先构造 Person
Person(std::string const& n, std::string const& a) noexcept
    : name(n), address(a) {
  std::cout << "Parameterized constructor called for Person\n";
  std::cout << "  Name: " << name << ", Address: " << address << '\n';
}

// 2. 把构造好的 Person 移动到 std::pair 中
    template<typename _U1, typename _U2, typename
        enable_if<_PCCP::template
        _MoveConstructiblePair<_U1, _U2>()
      && _PCCP::template
        _ImplicitlyMoveConvertiblePair<_U1, _U2>(),
                        bool>::type=true>
constexpr pair(_U1&& __x, _U2&& __y)
: first(std::forward<_U1>(__x)), second(std::forward<_U2>(__y)) { }


// 3. 把 std::pair 从 std::unordered_map 完美转发到_Hashtable，实际上调用的是Map_base 中的 insert
std::pair<iterator, bool>
insert(value_type&& __x)
// 注：这里的 value_type 实际是 std::pair<const Key, T> &&，已经是右值了，所以使用
// std::move 转发
{ return _M_h.insert(std::move(__x)); }


// 4. Map_base再转发到_Hashtable
    template<typename _Pair, typename = _IFconsp<_Pair>>
__ireturn_type
insert(_Pair&& __v)
{
  __hashtable& __h = this->_M_conjure_hashtable();
  return __h._M_emplace(__unique_keys(), std::forward<_Pair>(__v));
}

// 5. _Hashtable，其中把Person移动到节点中
template<typename _Key, typename _Value,
    typename _Alloc, typename _ExtractKey, typename _Equal,
    typename _H1, typename _H2, typename _Hash, typename _RehashPolicy,
    typename _Traits>
  template<typename... _Args>
    auto
    _Hashtable<_Key, _Value, _Alloc, _ExtractKey, _Equal,
    _H1, _H2, _Hash, _RehashPolicy, _Traits>::
    _M_emplace(true_type, _Args&&... __args)
```

对于使用了`std::piecewise_construct`的`emplace()`：

```cpp


// 1. 参数被从std::unordered_map完美转发到_Hashtable
    template<typename... _Args>
std::pair<iterator, bool>
emplace(_Args&&... __args)
{ return _M_h.emplace(std::forward<_Args>(__args)...); }


// 2. 在_Hashtable中再次完美转发
    // Emplace
    template<typename... _Args>
__ireturn_type
emplace(_Args&&... __args)
// 这里的unique key是获取一下哈希表的key是否唯一，这个属性在一开始就确定了
{ return _M_emplace(__unique_keys(), std::forward<_Args>(__args)...); }


// 3. _Hashtable
template<typename _Key, typename _Value,
    typename _Alloc, typename _ExtractKey, typename _Equal,
    typename _H1, typename _H2, typename _Hash, typename _RehashPolicy,
    typename _Traits>
  template<typename... _Args>
    auto
    _Hashtable<_Key, _Value, _Alloc, _ExtractKey, _Equal,
    _H1, _H2, _Hash, _RehashPolicy, _Traits>::
    _M_emplace(true_type, _Args&&... __args)
```
