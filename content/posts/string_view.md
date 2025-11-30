+++ 
date = '2025-11-30T15:57:00+08:00'
draft = false
title = 'TotW #1: string_view'
tags = ['C++']
toc = true
series = [’C++ Tip of the Week‘]
+++

# 1. 什么是`string_view`？

`string_view`是一个轻量级的字符串视图，不拥有字符串的数据本身，只提供对现有字符串缓冲区的只读访问。

本质是一个“不拷贝”的字符串抽象，由两个部分组成：

1. 一个指针指向字符数组的起始位置。
2. 一个长度值，表示字符内容的范围。

## 为什么值得引入？

- 高效性：避免了从`const char*`到`std::string`，或从`std::string`到`const char*`间的隐式转换所需的拷贝（O(n)的时间复杂度）。
- 隐式转换支持：
  - 支持从`const char*`构造（额外进行`strlen`计算，时间复杂度为O(n))。
  - 支持从`const std::string&构造`（时间复杂度仅为O(1)）。

# 2. 使用场景

## 使用场景适合

- 接受只读字符串参数时，推荐优先使用`string_view`。
- 如果函数不需要修改数据或长期存储数据：`string_view`用于临时引用是安全的。

## 不适合的场景

- 基于`string_view`保存数据：因为`string_view`不拥有数据，必须确保底层数据在`string_view`生命周期内有效。
  - 例如：结构体中的成员变量不适合作为`string_view`，此时更合适使用`std::string`。
- 处理需要NUL终止字符串的场景，例如需要传递给C风格API时（不能直接调用printf("%s\n", sv.data());）。

# 3. API转换建议

- 推荐做法：将需要接收字符串的API参数设计为string_view，这样可以方便在调用时处理既有的const char*和std::string类型数据，而不会造成额外的性能损耗：
void TakesStringView(std::string_view s); // 推荐
- 不推荐做法：
  - 使用void TakesCharStar(const char* s); —— 尽管高效，但调用者传入std::string时需要显式调用c_str()，增加不便。
  - 使用void TakesString(const std::string& s); —— 调用者传入const char*时需要生成临时std::string，造成多余的内存分配和拷贝。

# 4. 注意事项

## 4.1 生命周期管理
由于string_view不拥有数据，底层数据必须在视图存活期间有效。否则可能引发未定义行为。
void Test() {
    std::string s = "example";
    std::string_view sv = s; // sv依赖于s的生命周期
} // s超出作用域时，sv将指向无效数据
## 4.2 避免直接存储
不建议将string_view作为结构体或类中的成员变量。除非明确可以保证底层数据超出它的生命周期。
## 4.3 通过值传递
std::string_view是一种轻量级的对象（仅包含两个成员：指针和长度），推荐通过值传递而非const string_view&传递。
void Func(std::string_view sv); // ✅ 推荐
void Func(const std::string_view& sv); // 🚫 不推荐
## 4.4 与const的关系

标记string_view为const不会改变其不可修改底层数据的属性（因为string_view天生不可修改底层数据）。

## 4.5 NULL终止问题
std::string_view的数据可能并不是NUL结尾的，因此无法直接传递给需要C字符串的API：
printf("%s\n", sv.data()); // 🚫 不安全，避免使用
推荐使用Abseil提供的absl::PrintF()或构建一个显式字符串：std::string(sv)。

## 4.6 函数指针问题
如果函数签名中将String类型改为std::string_view，且函数被用作函数指针，会造成编译错误。例如：
void (*funcPtr)(const std::string&) = &SomeFunction;
funcPtr = &AnotherFunctionWithStringView; // 🚫 编译错误

# 5. 总结与最佳实践

如果函数只需要读取字符串（而不是保存或者修改），优先考虑使用string_view。
对仅执行单次操作且不需要存储数据的场景，string_view能显著提升效率。
在现有代码中引入string_view时，从底层工具代码开始逐步替换，而非盲目修改所有代码。
确保不会错误地引用生命周期短于string_view本身的数据。

# 6. 核心语法示例

## 6.1 接受string_view参数的例子
void ProcessData(std::string_view sv) {
    absl::PrintF("Processing: %s\n", sv); // 正确
    std::string s = std::string(sv); // 转为std::string以存储
}

## 6.2 不要将string_view用于存储
struct UserProfile {
    absl::string_view username;  // 不推荐，仅供演示问题
    int score;
};
## 6.3 与隐式转换的便捷性
void PrintString(std::string_view sv) {
    absl::PrintF("Log: %s\n", sv);
}

std::string s = "Hello World";
const char* cs = "Hello C-style";

// 无需显式转换
PrintString(s);
PrintString(cs);
