---
title: "C++ Concept 重载决议探讨"
date: 2022-12-16 14:39:21 +0800
categories: [杂记, Cpp]
tags: [c++, 编程语言, c++20]     # TAG names should always be lowercase
---

## 测试环境

* OS: Ubuntu 22.04
* CC: GCC 11.2.0

## 重载决议

重载决议（Overload Resolution）的定义，摘自 [cppreference](https://en.cppreference.com/w/cpp/language/overload_resolution.html):

> 为了编译函数调用，编译器必须首先进行[名字查找](https://en.cppreference.com/w/cpp/language/lookup.html)，对于函数可能涉及[实参依赖查找](https://en.cppreference.com/w/cpp/language/adl.html)，而对于函数模板可能后随[模板实参推导](https://en.cppreference.com/w/cpp/language/template_argument_deduction.html)。如果这些步骤产生了多个候选函数，那么需要进行重载决议选择将要实际调用的函数。

说人话就是，由于 C++ 支持函数重载、隐式类型转换、模板函数等特性，因此一个函数调用可能可以匹配上多个函数实现，编译器需要通过一些规则来选择一个最优实现参与编译。

重载决议规则比较复杂，但是可以总结几种常见的情况。

1. 无论如何，最优先选择完全匹配的函数实现。
2. 必须通过隐式转换匹配时，优先选择隐式转换少的函数实现，例如：

   ```cpp
   #include <iostream>

   void print(int a, char b) { std::cout << "int char" << std::endl; }
   void print(int a, int b) { std::cout << "int int" << std::endl; }

   int main() {
       // 选择 (int char) 的重载版本
       print('a', 'b');
       return 0;
   }
   ```
   {: run="cpp" }

3. 由于模板函数会生成完全匹配的函数实现，因此，如果没有模板特化，那么模板函数的优先级高于隐式转换：

   ```cpp
   #include <iostream>

   void print(int a, char b) { std::cout << "int char" << std::endl; }
   void print(int a, int b) { std::cout << "int int" << std::endl; }
   template <typename T, typename U>
   void print(T a, U b) {
       std::cout << "template" << std::endl;
   }

   int main() {
       // 选择模板函数版本
       print('a', 'b');
       return 0;
   }
   ```
   {: run="cpp" }

## 带有 Concept 的重载决议

### 重载决议失败

```cpp
#include <concepts>
#include <iostream>

template <typename T>
concept Addable = requires(T a) { a + a; };

template <std::integral T>
void print(const T &a) {
    std::cout << "integral" << std::endl;
}

template <Addable T>
void print(const T &a) {
    std::cout << "Addable" << std::endl;
}

int main() { print(12); }
```

上述代码在 GCC 11 中会编译失败，原因是 **call of overloaded ‘print(int)’ is ambiguous**。

代码中定义了一个新的概念 `Addable`，并且使用 `Addable` 概念与标准库概念 `std::integral`{:.language-cpp} 分别实现了一个 `print` 的重载，对于 `print(12)`{:.language-cpp} 而言，`12`{:.language-cpp} 既满足 `Addable` 也满足 `std::integral`{:.language-cpp}，这两种重载对于编译器而言是同等优先级的，因此编译器无法在其中择出一个最优实现。

想要解决这个问题也很简单，只需要引入 `!`{:.language-cpp} 来令两种重载互斥即可，例如在第二个重载中增加 `!std::integral<T>`{:.language-cpp}：

```cpp
#include <concepts>
#include <iostream>

template <typename T>
concept Addable = requires(T a) { a + a; };

template <std::integral T>
void print(const T &a) {
    std::cout << "integral" << std::endl;
}

template <typename T>
  requires Addable<T> && (!std::integral<T>)
void print(const T &a) {
    std::cout << "Addable" << std::endl;
}

int main() {
    // 选择 std::integral 重载版本
    print(12);
}
```
{: run="cpp" highlight-lines="13"}

由于第二个重载限定了 `T` 不能满足 `std::integral`{:.language-cpp}，因此编译器为 `print(12)`{:.language-cpp} 选择第一个重载。

### 合取引出的特异性

然而，Concept 之间并不总是同等优先级的。我们尝试引入一个新的概念 `IntAddable`：

```cpp
#include <concepts>
#include <iostream>

template <typename T>
concept Addable = requires(T a) { a + a; };

template <typename T>
concept IntAddable = std::integral<T> && Addable<T>;

template <std::integral T>
void print(const T &a) {
    std::cout << "integral" << std::endl;
}

template <IntAddable T>
void print(const T &a) {
    std::cout << "IntAddable" << std::endl;
}

int main() {
    // 选择 IntAddable 重载版本
    print(12);
}
```
{: run="cpp" highlight-lines="7-8"}

我们会发现，这回编译器不会报 ambiguous 了，并且选择了 `IntAddable` 的重载版本。我们从这里可以看出，由于 `IntAddable` 概念将 `std::integral`{:.language-cpp} 概念包含在内，因此对于 `print(12)`{:.language-cpp} 而言，`IntAddable` 比 `std::integral`{:.language-cpp} 更加具有特异性，在编译器的眼中 `IntAddable` 的优先级就要高于 `std::integral`{:.language-cpp}。这对于学习过 css 选择器的同学来说应该是非常熟悉的。

### 那么析取？

Concept 想要引入包含关系，除了合取语法之外还有析取语法。不过与合取相反的是，合取是 `A && B`{:.language-cpp} 包含 `A`，但是析取是 `A` 包含 `A || B`{:.language-cpp}。

下面的代码可以佐证这一点：

```cpp
#include <concepts>
#include <iostream>

template <typename T>
concept Addable = requires(T a) { a + a; };

template <typename T>
concept IntOrAddable = std::integral<T> || Addable<T>;

template <std::integral T>
void print(const T &a) {
    std::cout << "integral" << std::endl;
}

template <IntOrAddable T>
void print(const T &a) {
    std::cout << "IntOrAddable" << std::endl;
}

int main() {
    // 选择 std::integral 重载版本
    print(12);
}
```
{: run="cpp" highlight-lines="7-8"}

### 结论

对于任意两个没有关联的概念 `A` 和 `B`，在重载决议时的优先级为：`A && B`{:.language-cpp} > `A` == `B` > `A || B`{:.language-cpp}。
