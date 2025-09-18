+++
date = '2024-09-12T10:16:00+08:00'
draft = false
title = 'CMake设置include和link时的属性'
tags = ['CMake']
toc = true
+++

# CMake 设置 include 和 link 时的属性

## target_include_directories、target_link_libraries

在 CMake 中，target_include_directories 和 target_link_libraries 命令使用 PRIVATE、PUBLIC 和 INTERFACE 关键字来控制包含目录和链接库的传递性。

- PRIVATE：仅对目标自身可见。其他依赖该目标的目标无法访问这些包含目录或链接库。
- PUBLIC：对目标自身和所有依赖该目标的目标都可见。即，包含目录或链接库会被传递给依赖该目标的所有目标。
- INTERFACE：仅对依赖该目标的目标可见，目标自身不可见。

举个例子：

```cmake
# A库
add_library(A ...)
target_include_directories(A PUBLIC include)

# B库依赖于A库
add_library(B ...)
target_link_libraries(B PUBLIC A)

# C库依赖于B库
add_library(C ...)
target_link_libraries(C PRIVATE B)
```

在这个例子中：

1. `A`库的包含目录是`include`，并且是`PUBLIC`的，所以任何依赖`A`的目标都可以访问`include`目录。
2. `B`库依赖于`A`库，并且是`PUBLIC`的，所以任何依赖`B`的目标也可以访问`A`的包含目录。
3. `C`库依赖于`B`库，并且是`PRIVATE`的，所以`C`库本身可以访问`B`库，但不会将`B`库传递给其他依赖`C`的目标。

因此，`C`库可以访问`A`库的包含目录，因为`B`库以`PUBLIC`方式依赖于`A`库。但是，如果`B`库以`PRIVATE`方式依赖于`A`库，那么`C`库将无法访问`A`库的包含目录。

总结：

- `PRIVATE`：仅目标自身可见，不传递。
- `PUBLIC`：目标自身和依赖目标都可见，传递。
- `INTERFACE`：仅依赖目标可见，不传递给目标自身。

### INTERFACE

`INTERFACE` 属性比较特殊，在 CMake 中，它仅用来指定对依赖目标可见的属性，而对定义这些属性的目标自身不可见。在希望传递某些属性（如 include 目录或者编译选项）给依赖目标，但不希望这些属性影响定义这些属性的目标自身时，特别有用。**另外，可以在声明一个库的时候，使用 INTERFACE 关键字来定义该库为一个 INTERFACE 库。INTERFACE 库不生成实际的二进制文件。**

以下是`INTERFACE`关键字的主要用途：

1. **传递包含目录**：当你希望某个库的包含目录仅对依赖它的目标可见，而不影响该库自身的编译。
2. **传递编译选项**：当你希望某些编译选项仅对依赖目标生效，而不影响该库自身的编译。
3. **传递链接库**：当你希望某些链接库仅对依赖目标生效，而不影响该库自身的链接。

举个例子：

```cmake
# 定义一个接口库
add_library(MyInterfaceLib INTERFACE)

# 设置接口库的包含目录
target_include_directories(MyInterfaceLib INTERFACE ${CMAKE_SOURCE_DIR}/include)

# 设置接口库的编译选项
target_compile_options(MyInterfaceLib INTERFACE -Wall -Wextra)

# 定义一个实际库，依赖于接口库
add_library(MyActualLib src/my_actual_lib.cpp)
target_link_libraries(MyActualLib PUBLIC MyInterfaceLib)
```

在这个例子中：

1. `MyInterfaceLib`是一个接口库，它不会生成任何实际的二进制文件。
2. `MyInterfaceLib`的包含目录和编译选项通过`INTERFACE`关键字指定，这意味着这些属性仅对依赖`MyInterfaceLib`的目标可见。
3. `MyActualLib`依赖于`MyInterfaceLib`，因此`MyActualLib`会继承`MyInterfaceLib`的包含目录和编译选项。

可是看到，`INTERFACE` 库的作用和 C++中的抽象基类有一些类似，它可以定义公共的编译选项、include 目录等属性，供依赖它的目标继承，但不包含实现，即 `INTERFACE` 库不生产实际的二进制文件。

而对于非 `INTERFACE` 库，可以同时使用 `PUBLIC`、`PRIVATE` 和 `INTERFACE` 来目标设置不同的属性，例如：

```cmake
# 定义一个普通库
add_library(MyLibrary src/my_library.cpp)

# 设置包含目录
target_include_directories(MyLibrary
    PRIVATE ${CMAKE_SOURCE_DIR}/src/private_include
    PUBLIC ${CMAKE_SOURCE_DIR}/src/public_include
    INTERFACE ${CMAKE_SOURCE_DIR}/src/interface_include
)

# 设置编译选项
target_compile_options(MyLibrary
    PRIVATE -DPRIVATE_DEFINE
    PUBLIC -DPUBLIC_DEFINE
    INTERFACE -DINTERFACE_DEFINE
)
```

在这个例子中：

`PRIVATE` 属性（如包含目录和编译选项）仅对 `MyLibrary` 自身可见，不会传递给依赖 `MyLibrary` 的目标。
`PUBLIC` 属性对 `MyLibrary` 自身和依赖 `MyLibrary` 的目标都可见，会传递给依赖 `MyLibrary` 的目标。
`INTERFACE` 属性仅对依赖 `MyLibrary` 的目标可见，不会影响 `MyLibrary` 自身。
总结

## include_directories、target_include_directories

设置 include 目录时，一个较为有用的选项是`SYSTEM`，这会告诉编译器将这些目录视为系统包含目录，从而抑制这些目录中同文件的警告。例如系统引入了更多的编译警告的同时使用了第三方库，为了抑制编译时，编译器对第三方库的代码产生多余的警告，可以在 include 第三方库时，使用`SYSTEM`选项。
