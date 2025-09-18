+++
date = '2024-08-30T10:14:00+08:00'
draft = false
title = '如何正确读取RTI中的enum'
tags = ['C++', 'RTI']
toc = true
+++

# 如何正确读取 RTI 中的 enum

## 背景

假设录制的 RTI 数据保存.dat 文件中，使用 DB browser for SQLite 查看.dat 文件，其中包含了要读取的 topic 数据，这个 topic 对应的 IDL 中含义 enum 类型，假设其定义如下：

```idl
module Sample {
    enum State {
        UNKNOWN = -1,
        MANUAL = 1,
        STARTED = 2,
        READY = 3
    };

    struct Info {
        Header header;
        State state;
        string name;
        boolean valid;
    };
};
```

之后使用 RTI 的接口从.dat 中读取了数据，返回的数据类型为`dds::core::xtypes::DynamicData dynamic_raw`，然后从中读取需要的字段，将其储存起来。

对于其他类型的数据，例如`string`或者`boolean`，可以通过下面的方式读取：

```cpp
inline void SampleReader::FillData(dds::core::xtypes::DynamicData dynamic_raw) {
  if (dynamic_raw.member_exists("name")) {
    idl_message_.name(dynamic_raw.value<std::string>("name"));
  }

  if (dynamic_raw.member_exists("valid")) {
    idl_message_.valid(dynamic_raw.value<bool>("valid"));
  }
}
```

但是对于`enum`类型的数据，直接通过`value`读取是不行的，

```cpp
inline void SampleReader::FillData(dds::core::xtypes::DynamicData dynamic_raw) {
  if (dynamic_raw.member_exists("state")) {
    idl_message_.state(dynamic_raw.value<Sample::State>("state"));
  }
}
```

这样做会提示未定义符号，例如：

```bash
undefined symbol: _ZNK3rti4core6xtypes15DynamicDataImpl5valueIN3dds4core9safe_enumIN15Sample8State_defENS9_4typeEEEEET_RKNSt7__cxx1112basic_stringIcSt11char_traitsIcESaIcEEE
```

这个原因比较奇怪，实际上这个类型`Sample::State`被导出了，`idl_message_`就是`Sample::State`类型的，可在读取的时候就不能这么使用。考虑到 enum 可能是一个 int 类型，所以可以通过下面的方式读取：

```cpp
auto state{dynamic_raw.value<int32_t>("state")};
idl_message_.state(static_cast<Sample::State>(state));
```

这样也不行，编译都过不了，报错为：

```bash
error: invalid conversion from ‘int’ to ‘dds::core::safe_enum<Sample::State_def>::inner_enum’ {aka ‘Sample::State_def::type’} [-fpermissive]
```

也就是不能将`int`类型直接转换为`Sample::State`类型。那么应该如何正确读取 DDS 中的 enum 呢？

## 解决方案

查看源代码，发现枚举`Sample::State`的定义为

```cpp
struct State_def {
    enum type {
        UNKNOWN = -1,
        MANUAL = 1,
        STARTED = 2,
        READY = 3
    };
    static type get_default(){ return UNKNOWN;}
};

typedef ::dds::core::safe_enum< State_def > State;
NDDSUSERDllExport std::ostream& operator << (std::ostream& o,const State& sample);
```

即`Sample::State`实际是`dds::core::safe_enum`的实例化，而`safe_enum`是一个模版类，其定义如下：

```cpp
template<typename def, typename inner = typename def::type>
class UserDllExport safe_enum : public def {
public:

    /**
     * The underlying \p enum type
     *
     * To convert from an integer, \p static_cast it to the member type \p inner_enum:
     *
     * \code
     * int reliability_as_int = 2;
     * ReliabilityKind reliability = static_cast<ReliabilityKind::inner_enum>(reliability_as_int);
     * \endcode
     */
    typedef inner inner_enum;

public:

    /**
     * @brief Initializes the enumeration to zero
     *
     * @note Zero may not be a valid enumerator. It's best to assign a valid
     * enumerator whenever possible:
     * \code
     * Reliability reliability = Reliability::RELIABLE;
     * \endcode
     */
    safe_enum() : val(static_cast<inner>(0))
    {
    }

    /**
     * @brief Copy constructor
     */
    safe_enum(inner_enum v) : val(v) {}

    /**
     * @brief Retrieves the actual \p enum value
     */
    inner_enum underlying() const { return val; }


private:
    inner_enum val;
};
```

`inner_enum`的注释中提到，可以通过`static_cast`将`int`转换为`inner_enum`，所以可以通过下面的方式读取：

```cpp
inline void SampleReader::FillData(dds::core::xtypes::DynamicData dynamic_raw) {
  if (dynamic_raw.member_exists("state")) {
    auto state{dynamic_raw.value<int32_t>("state")};
    idl_message_.state(static_cast<Sample::State::inner_enum>(state));
  }
}
```

原来解决方案就在源代码里面……
