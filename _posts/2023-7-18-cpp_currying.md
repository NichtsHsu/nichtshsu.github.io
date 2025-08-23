---
title: "[C++] 不使用标准库和 lambda 实现柯里化"
date: 2023-7-18 13:17:17 +0800
categories: [教程, Cpp]
tags: [c++, 编程语言, 教程, 模板元编程]     # TAG names should always be lowercase
math: true
---

## 限制

由于 C++ 提供了 lambda 语法和强大的 `functional` 库，如果不做任何限制的话，那么实现柯里化是一件非常简单的事情。本文注重于介绍原理和过程，而不是最终结果，因此，让我们来做一些大胆的限制：

* **禁止使用任何标准库组件**
* **禁止使用 lambda 语法**

## 实现 `Function`

事实上，在我之前的文章 [\[C++\] std::function 是如何实现 lambda 递归的]({{ site.baseurl }}{% link _posts/2023-3-30-std_function.md %})中对 `std::function`{:.language-cpp} 的核心原理有过十分详细的介绍并给出了一份实现，我们可以直接把它搬过来并稍微改一改命名：

```cpp
#include <functional>
#include <iostream>

template <typename T>
class Function {
  Function() = delete;
};

template <typename Ret, typename... Args>
class Function<Ret(Args...)> {
 private:
  template <typename Callable>
  static Ret invoke(void* callable, Args... args) {
    return std::invoke(
        *reinterpret_cast<std::remove_reference_t<Callable>*>(callable),
        std::forward<Args>(args)...);
  }
  template <typename Callable>
  static void free(void* callable) {
    delete reinterpret_cast<std::remove_reference_t<Callable>*>(callable);
  }
  template <typename Callable>
  static void* copy(void* callable) {
    return new std::remove_reference_t<Callable>(
        *reinterpret_cast<std::remove_reference_t<Callable>*>(callable));
  }

 public:
  template <typename Callable,
            std::enable_if_t<!std::is_same_v<std::remove_reference_t<Callable>,
                                             Function<Ret(Args...)>>,
                             bool> = true>
  Function(Callable&& callable) {
    _callable =
        new std::remove_reference_t<Callable>(std::forward<Callable>(callable));
    _invoke = invoke<Callable>;
    _free = free<Callable>;
    _copy = copy<Callable>;
  }
  Function(const Function& other) {
    _callable = other._copy(other._callable);
    _invoke = other._invoke;
    _free = other._free;
    _copy = other._copy;
  };
  Function(Function&& other) {
    _callable = other._callable;
    other._callable = nullptr;
    _invoke = other._invoke;
    _free = other._free;
    _copy = other._copy;
  }
  ~Function() { _free(_callable); }

  Ret operator()(Args... args) const {
    return _invoke(_callable, std::forward<Args>(args)...);
  }

 private:
  void* _callable;
  Ret (*_invoke)(void*, Args...);
  void (*_free)(void*);
  void* (*_copy)(void*);
};

int main() {
  Function<unsigned(unsigned)> fact = [&fact](unsigned x) {
    return x <= 1 ? 1 : x * fact(x - 1);
  };
  // 输出 3628800
  std::cout << fact(10) << std::endl;

  Function<unsigned(unsigned)> fib = [&fib](unsigned x) {
    return x == 0 ? 0 : x <= 2 ? 1 : fib(x - 1) + fib(x - 2);
  };
  // 输出 55
  std::cout << fib(10) << std::endl;

  return 0;
}
```

不过这个实现用到了标准库的东西，不符合我们的限制，因此我们下一步就是需要想办法自己实现这些组件。

## `remove_reference`  与模板元编程

关于**模板元编程（Template Metaprogramming）**，网上有太多的教程，我不打算从头到尾彻底把它讲清楚，我们仅关注我们会用到的部分。

在模板元编程中，最依赖的特性之一就是**模板特化（Template Specialization）**，这里我们以 `remove_reference` 为例：

```cpp
template <typename T>
struct remove_reference {
  using type = T;
};

template <typename T>
struct remove_reference<T&> {
  using type = T;
};

template <typename T>
struct remove_reference<T&&> {
  using type = T;
};

template <typename T>
using remove_reference_t = typename remove_reference<T>::type;
```

上述是标准库中 `std::remove_reference` 和 `std::remove_reference_t` 的实现，它的功能是输入任何一个类型，得到它去除引用后的类型。

例如，`remove_reference_t<Test>`{:.language-cpp}, `remove_reference_t<Test&>`{:.language-cpp} 和 `remove_reference_t<Test&&>`{:.language-cpp} 的类型都是 `Test`。它的原理我相信各位一眼就能看出，就是通过模板特化匹配左值引用和右值引用的类型，然后将它们去除引用后的类型重命名为 `type`。

## 引用折叠与完美转发

由于 C++ 右值引用的特殊性（即**具名的右值引用是左值**，具体可参考我之前写的 [\[C++\] 深入了解左值与右值]({{ site.baseurl }}{% link _posts/2023-6-27-cpp_lvalue_rvalue.md %})这篇文章），在传递函数参数的过程中会丢失右值引用类型，这可能会导致在 `invoke` 时函数的行为不一致。这时我们就需要用到**完美转发（Perfect Forwarding）**，让参数在传递的过程中保持自身的右值引用类型，而实现这一机制的标准库函数就是 `std::forward`{:.language-cpp}。

在谈论如何实现完美转发之前，我们需要先了解什么是**引用折叠（Reference Collapsing）**。我们知道，C++ 的引用分为左值引用类型 `T&`{:.language-cpp} 和右值引用类型 `T&&`{:.language-cpp}，但是，如果此时的 `T` 也是左值引用或右值引用类型，会发生什么呢？C++ 为此定义了以下规则：

| `T` 的类型 | `T&` 的类型 | `T&&` 的类型 |
| :-: | :-: | :-: |
| `U&` | `U&` | `U&` |
| `U&&` | `U&` | `U&&` |

让我们来看看如何实现 `forward`：

```cpp
template <typename T>
T &&forward(remove_reference_t<T> &t) {
  return static_cast<T &&>(t);
}

template <typename T>
T &&forward(remove_reference_t<T> &&t) {
  return static_cast<T &&>(t);
}
```

当 `T` 是 `U&`{:.language-cpp} 时，`forward` 返回 `U& &&`{:.language-cpp}，根据引用折叠规则，折叠为 `U&`{:.language-cpp}；当 `T` 是 `U&&`{:.language-cpp} 时，`forward` 返回 `U&& &&`{:.language-cpp}，根据引用折叠的规则，折叠为 `U&&`{:.language-cpp}，这样就保证了参数传递时的右值引用类型。

当然，标准库的实际实现会多一点东西：

```cpp
template <typename T>
constexpr T &&forward(remove_reference_t<T> &t) noexcept {
  return static_cast<T &&>(t);
}

template <typename T>
constexpr T &&forward(remove_reference_t<T> &&t) noexcept {
  return static_cast<T &&>(t);
}
```

另外，引用折叠的规则也可以反过来用，实现**万能引用**。具体来说，当 `T` 是模板类型时，`T&&`{:.language-cpp} 就是一个万能引用，它不仅可以匹配 `U&&`{:.language-cpp} 的右值引用类型，还可以匹配 `U&`{:.language-cpp} 的左值引用类型，其原理就是将 `U& &&`{:.language-cpp} 折叠为 `U&`{:.language-cpp} 来匹配左值类型。

相比之下，`move` 的实现就比较简单，因为 `move` 本质上只是对强制类型转换的包装：

```cpp
template <typename T>
constexpr remove_reference_t<T> &&move(T &&t) noexcept {
  return static_cast<remove_reference_t<T> &&>(t);
}
```

## 实现 `invoke`

回想一下我们使用 `std::invoke` 的根本原因是什么。

大体上来说，C++ 的可调用类型分为**函数（指针）类型**，**成员函数（指针）类型**，以及**重载了 `operator()`{:.language-cpp} 的类型**。其中第一种和第三种都可以简单通过 `f(args...)`{:.language-cpp} 的形式进行调用，而第二种必须以 `(t.*f)(args...)`{:.language-cpp} 或 `(t->*f)(args...)`{:.language-cpp} 的形式来调用。而 `std::invoke`{:.language-cpp} 则隐藏了这个细节，将三种可调用类型通过 `std::invoke(f, args...)`{:.language-cpp} 以及 `std::invoke(f, t, args...)`{:.language-cpp} 的形式统一起来。

现在让我们来看看如何**实现 `invoke`**。

由于成员函数的特殊性，我们需要将 `invoke` 的参数分为三部分，一是可调用类型的对象，二可能是一个用来调用成员函数的对象也可能是一个参数，三是剩下的参数。因此，我们需要考虑三种不同的重载：

```cpp
template <typename Callable>
decltype(auto) invoke(Callable &&callable) {}

template <typename Callable, typename First>
decltype(auto) invoke(Callable &&callable, First &&first) {}

template <typename Callable, typename First, typename... Rest>
decltype(auto) invoke(Callable &&callable, First &&first, Rest &&...rest) {}
```

注意：我们在此处使用 `decltype(auto)`{:.language-cpp} 作为返回值类型，是因为我们不知道返回值类型是否包含引用，而仅凭 `auto`{:.language-cpp} 是无法处理引用的。当 `decltype(auto)`{:.language-cpp} 作为返回值时，约等于将返回值表达式替换到 `auto`{:.language-cpp} 位置进行类型推导，因此可以自动处理返回值为引用的情况。

其中第一种最为简单，因为这种形式的调用没有任何参数，必不可能是成员函数，所以我们直接正常调用即可：

```cpp
template <typename Callable>
decltype(auto) invoke(Callable &&callable) {
  return callable();
}
```

接下来是第二种和第三种。在这两种情况下，`First` 类型既有可能是用来调用成员函数的对象，也有可能是一个普通参数，因此我们需要有一个办法可以区分普通函数和成员函数。

在这里我们需要引入一个**特殊模板类型 `F T::*`{:.language-cpp}**。该类型的特殊之处在于，**它可以匹配任何一个成员函数类型**，例如，对于 `void (Test::*)(int)`{:.language-cpp} 类型，`F` 匹配 `void (int)`{:.language-cpp} 而 `T` 匹配 `Test`{:.language-cpp}。

有了这个知识点之后，我们可以对 `invoke` 函数再次进行模板特化，以适配成员函数类型。

不过，在这里我想给大家介绍一下模板元编程中一种非常常见的做法：

```cpp
// 用空结构体来表示一种 tag
struct normal_function {};
struct member_function {};

// 普通函数类型
template <typename Callable>
struct callable_tag {
  using type = normal_function;
};

// 通过模板特化来区分成员函数类型
template <typename F, typename T>
struct callable_tag<F T::*> {
  using type = member_function;
};

// 简化一下调用形式
template <typename Callable>
using callable_tag_t =
    typename callable_tag<remove_reference_t<Callable>>::type;

// 注意该重载的第一个参数是 normal_function
template <typename Callable, typename... Args>
decltype(auto) invoke_inner(normal_function, Callable &&callable,
                            Args &&...args) {
  return callable(forward<Args>(args)...);
}

// 而该重载的第一个参数是 member_function，通过这个 tag 来区分成员函数
template <typename Callable, typename Caller, typename... Args>
decltype(auto) invoke_inner(member_function, Callable &&callable,
                            Caller &&caller, Args &&...args) {
  return (caller->*callable)(forward<Args>(args)...);
}

// 真正的 invoke 函数，使用 callable_tag_t 作为 invoke_inner
// 的第一个参数来区分普通函数与成员函数
template <typename Callable, typename... Args>
decltype(auto) invoke(Callable &&callable, Args &&...args) {
  return invoke_inner(callable_tag_t<Callable>{}, callable,
                      forward<Args>(args)...);
}
```

在这种实现下，无需为只有一个 `Callable` 参数的情况写一个重载，因为 `Args` 可以为空。

之后我们将 `Function` 实现中的 `std::invoke`{:.language-cpp} 替换成我们自己实现的即可：

```cpp
template <typename Ret, typename... Args>
class Function<Ret(Args...)> {
 private:
  template <typename Callable>
  static Ret invoke(void *callable, Args... args) {
    // 这里使用 ::invoke 的意思是使用全局定义的 invoke 函数
    // 而不是 Function 的成员函数 invoke
    return ::invoke(*reinterpret_cast<remove_reference_t<Callable> *>(callable),
                    forward<Args>(args)...);
  }

  ...
};
```

## `enable_if` 与 SFINAE

在我之前的文章 [\[C++\] std::function 是如何实现 lambda 递归的]({{ site.baseurl }}{% link _posts/2023-3-30-std_function.md %})中有提到过，在 C++20 引入 concept 之前，可以使用 **`std::enable_if`{:.language-cpp}** 来防止入参 `Function &`{:.language-cpp} 类型匹配到 `Callable &&`{:.language-cpp} 构造函数而不是 `const Function &`{:.language-cpp} 拷贝构造函数。

`std::enable_if`{:.language-cpp} 的核心原理是利用了 C++ 的 **SFINAE 原则**。所谓 SFINAE，即**替换失败不是错误（Substitution Failure Is Not An Error）**，简单来说，就是编译器在解析多种可能的重载时，其中一些重载可能是由模板实例化而来的，如果在替换模板参数时发生错误，则编译器**不会**报错并停止，而只是将其从候选列表中移除。

基于这一原理，我们实现 `enable_if` 的思路就很明确了：当 `enable_if` 的条件成立时，正常编译，而当 `enable_if` 的条件不成立时，令其发生编译错误，这样编译器就会把当前模板从候选列表中移除。

这样，我们就可以使用非常简单的方法实现 `enable_if`：

```cpp
template <bool, typename T = void>
struct enable_if {};

template <typename T>
struct enable_if<true, T> {
  using type = T;
};

template <bool Condition, typename T>
using enable_if_t = typename enable_if<Condition, T>::type;
```

当 `Condition` 为 `true`{:.language-cpp} 时，`enable_if` 包含 `type` 类型，而当 `Condition` 为 `false`{:.language-cpp} 时，不包含 `type` 类型，导致编译错误，使编译器抛弃该重载。

当然，标准库里的 `enable_if` 还允许同时使用多个条件，这涉及到模板元编程的另一个模块：即如何使用模板实现 and, or, not 等逻辑运算。其核心原理还是模板特化那一套，因为我们暂且用不到这个功能，所以我们就不深入去讲了。

接下来我们还需要一个 **`is_same`** 来判断两个类型是否是同一个类型，这个实现很简单，直接上代码：

```cpp
template <typename, typename>
struct is_same {
  constexpr static bool value = false;
};

// 模板特化，当两个类型一致时 value 为 true
template <typename T>
struct is_same<T, T> {
  constexpr static bool value = true;
};

// 简化调用接口
template <typename T, typename U>
constexpr static bool is_same_v = is_same<T, U>::value;
```

在实际使用时，我们声明的形式是 `enable_if_t<!is_same_v<remove_reference_t<Callable>, Function<Ret(Args...)>>, bool> = true`{:.language-cpp}，注意到 `enable_if_t` 在模板中实际声明了一个模板常量，因此这里的 `= true`{:.language-cpp} 主要目的是给这个模板常量赋予一个默认值，否则就需要调用时来提供值。当然，这里即使把 `true`{:.language-cpp} 改成 `false`{:.language-cpp}，或者把 `bool`{:.language-cpp} 改成其他类型都不会影响最终结果。

## `tuple`, `get`, `apply` 与 `tuple_cat`

### `tuple` 和 `get`

再回来看看柯里化需要什么：我们需要临时保存一些参数，直到被柯里化的函数被调用时，将这些被保存的参数和剩余的其他参数一起传递给柯里化前的函数。

这个需求实际上说白了就是要做一个 lambda 函数。然而我们开篇就禁止使用 lambda 函数了，所以我们要想办法做一个类似 lambda 函数的玩意，也就是说它具有保存任意数量的任意类型的对象的能力。万幸的是，标准库里确实有那么个玩意满足我们的需求，他就是 **`std::tuple`**。

`std::tuple`{:.language-cpp} 的核心原理并不复杂，以 `std::tuple<T1, T2, T3, T4>`{:.language-cpp} 来举例，该类型实际只储存了 `T1` 的对象，但是它会继承 `std::tuple<T2, T3, T4>`{:.language-cpp}，然后由 `std::tuple<T2, T3, T4>`{:.language-cpp} 储存 `T2` 对象；再进一步地，`std::tuple<T2, T3, T4>`{:.language-cpp} 继承 `std::tuple<T3, T4>`{:.language-cpp}，以此类推。

另一方面，为了更好地定位 `tuple` 中的值，模板参数中会添加一个索引 `Idx`，该索引会随着一次次的继承而增大。

一个简易的 `tuple` 实现如下：

```cpp
// 用来储存一个值的简易包装类型
template <size_t Idx, typename T>
class tuple_storage {
 public:
  tuple_storage(const T &value) : value(value) {}
  tuple_storage(T &&value) : value(forward<T>(value)) {}
  T value;
};

// 使用 tuple_inner 的主要目的是让 tuple 类型可以隐藏 Idx
template <size_t Idx, typename... Elements>
class tuple_inner {};

template <size_t Idx, typename T>
class tuple_inner<Idx, T> : public tuple_storage<Idx, T> {
 public:
  tuple_inner(const T &value) : tuple_storage<Idx, T>(value) {}
  tuple_inner(T &&value) : tuple_storage<Idx, T>(forward<T>(value)) {}
};

// 对于多个值的 tuple，每次继承 tuple_inner 都会让 Idx 加一
template <size_t Idx, typename T, typename... Rest>
class tuple_inner<Idx, T, Rest...> : public tuple_inner<Idx + 1, Rest...>,
                                     public tuple_storage<Idx, T> {
 public:
  tuple_inner(const T &value, Rest &...rest)
      : tuple_inner<Idx + 1, Rest...>(rest...), tuple_storage<Idx, T>(value) {}
  tuple_inner(T &&value, Rest &&...rest)
      : tuple_inner<Idx + 1, Rest...>(forward<Rest>(rest)...),
        tuple_storage<Idx, T>(forward<T>(value)) {}
};

// 对外暴露的 tuple 类型，本身只是 tuple_inner 的包装，没有什么特殊之处
template <typename... Elements>
class tuple : public tuple_inner<0, Elements...> {
 public:
  tuple(const Elements &...elements)
      : tuple_inner<0, Elements...>(elements...) {}
  tuple(Elements &&...elements)
      : tuple_inner<0, Elements...>(forward<Elements>(elements)...) {}
};

template <>
class tuple<void> {};
```

这时候有人就要问了，你搞 `Idx` 我能勉强理解为为了定位值，你搞 `tuple_inner` 我也可以理解为是为了对外隐藏 `Idx` 参数，但是你**搞 `tuple_storage` 是为了什么，为什么不直接在 `tuple_inner` 里存一个对象**？

这就问到点子上了。我们现在需要一个能够获取 `tuple` 中值的方法，我们可以使用标准库里的这种形式：**`get<0>(tuple)`{:.language-cpp} 获取 `tuple` 的第一个值，`get<1>(tuple)`{:.language-cpp} 获取 `tuple` 的第二个值，以此类推**。

那么这个 `get` 方法究竟要如何实现呢？答案是：

```cpp
template <size_t Idx, typename T>
T &get(tuple_storage<Idx, T> &t) {
  return t.value;
}
```

啊？就这？是的，就这。这就是为什么要引入 `tuple_storage`。

其核心的原理在于，C++ 的引用和指针一样，**父类型的引用可以接受子类型的对象**。举例来说，对于 `tuple<T1, T2, T3, T4>`{:.language-cpp} 而言，它继承了 `tuple_storage<0, T1>`{:.language-cpp}, `tuple_storage<1, T2>`{:.language-cpp}, `tuple_storage<2, T3>`{:.language-cpp} 以及 `tuple_storage<3, T4>`{:.language-cpp}，当调用 `get<1>(tuple)`{:.language-cpp} 时，编译器寻找能够匹配 `tuple_storage<1, ?>`{:.language-cpp} 的类型，最终找到 `tuple_storage<1, T2>`{:.language-cpp}，然后通过父类型的引用拿到它所储存的值。

另一方面，我们可以再实现 `const` 版本和右值版本：

```cpp
template <size_t Idx, typename T>
const T &get(const tuple_storage<Idx, T> &t) {
  return t.value;
}

template <size_t Idx, typename T>
T &&get(tuple_storage<Idx, T> &&t) {
  return move(t.value);
}

template <size_t Idx, typename T>
const T &&get(const tuple_storage<Idx, T> &&t) {
  return move(t.value);
}
```

### `apply` 与 `make_index_sequence`

接下来我们再说说 `std::apply`{:.language-cpp}。简单来说，这个函数就是一个 `tuple` 版的 `invoke` 函数，只不过它将输入的参数打包成了 `tuple`。例如，`invoke(f, 1, 2, 3)`{:.language-cpp} 等效于 `apply(f, tuple<int, int, int>(1, 2, 3))`{:.language-cpp}。

遗憾的是，这里出现了一个黑魔法。具体来说，是 **`std::make_index_sequence`{:.language-cpp}**，它的作用很简单，`std::make_index_sequence<N>`{:.language-cpp} 将得到一个 `std::integer_sequence<0, 1, 2, ..., N-1>`{:.language-cpp} 类型。而具体是怎么展开的，编译器大都选择了“开洞”，例如 GCC，通过内建的 `__integer_pack(N)...`{:.language-cpp} 指令，在编译器编译源码时手动将这个指令展开为数组序列。

不过幸运的是，即使不使用黑魔法，我们仍然有机会实现这一功能，参考 [stackoverflow 上的这个问题的回答](https://stackoverflow.com/questions/49669958/details-of-stdmake-index-sequence-and-stdindex-sequence)：

```cpp
template <std::size_t... Ns>
struct integer_sequence {};

template <std::size_t N, std::size_t... Is>
auto make_index_sequence_inner() {
  if constexpr (N == 0)
    return integer_sequence<Is...>();
  else
    return make_index_sequence_inner<N - 1, N - 1, Is...>();
}

template <std::size_t N>
using make_index_sequence =
    remove_reference_t<decltype(make_index_sequence_inner<N>())>;
```

在这里，**`if constexpr`{:.language-cpp}** 是 C++17 引入的新语法，在编译时就完成判断，并且抛弃不匹配的另一条路径。这意味着，两条路径可以返回不一样的类型，因为编译时始终只有一条路径参与编译。需要注意的是，即使不使用 `if constexpr`{:.language-cpp} 语法，我们仍然可以通过类型继承的方法来达成相同的目的，就像我贴出的那篇 stackoverflow 的第一条回答那样。

我们来理解一下这里究竟发生了什么：

1. 调用 `make_index_sequence<3>`{:.language-cpp}
2. 扩展到 `remove_reference_t<decltype(make_index_sequence_inner<3>())>`{:.language-cpp}
3. 在 `make_index_sequence_inner<3>()`{:.language-cpp} 中，`N` 匹配到 `3`{:.language-cpp}，而 `Is` 没有匹配到任何东西
4. `if constexpr`{:.language-cpp} 断言为 `false`{:.language-cpp}，编译器选择 `else`{:.language-cpp} 路径
5. 调用 `make_index_sequence_inner<2, 2>()`{:.language-cpp}
6. 此时 `N` 匹配到 `2`{:.language-cpp}，`Is` 也匹配到 `2`{:.language-cpp}
7. `if constexpr`{:.language-cpp} 仍然断言 `false`{:.language-cpp}，编译器选择 `else`{:.language-cpp} 路径
8. 调用 `make_index_sequence_inner<1, 1, 2>()`{:.language-cpp}
9. 此时 `N` 匹配到 `1`{:.language-cpp}，`Is` 匹配到 `1, 2`{:.language-cpp}
10. `if constexpr`{:.language-cpp} 继续断言 `false`{:.language-cpp}，编译器选择 `else`{:.language-cpp} 路径
11. 调用 `make_index_sequence_inner<0, 0, 1, 2>()`{:.language-cpp}
12. 此时 `N` 匹配到 `0`{:.language-cpp}，`Is` 匹配到 `0, 1, 2`{:.language-cpp}
13. `if constexpr`{:.language-cpp} 断言为 `true`{:.language-cpp}，`return integer_sequence<Is...>()`{:.language-cpp} 被调用
14. 由于 `Is` 匹配到 `0, 1, 2`{:.language-cpp}，因此返回的类型是 `integer_sequence<0, 1, 2>`{:.language-cpp}。

有了 `integer_sequence`，我们就很好做 `apply` 了，我们只需要通过 `integer_sequence` 对 `get<Idx>(tuple)`{:.language-cpp} 进行展开即可。

所以首先，我们需要先有办法得到 `tuple` 元素的数量。这个不难，C++11 添加了 **`sizeof...()`{:.language-cpp} 运算符**，可以得到模板参数包的大小，具体来说：

```cpp
template <typename T>
struct tuple_size {};

template <typename... Elements>
struct tuple_size<tuple<Elements...>> {
  constexpr static size_t size = sizeof...(Elements);
};

// 简单封装一下以方便调用
template <typename T>
constexpr size_t tuple_size_v = tuple_size<remove_reference_t<T>>::size;
```

之后就可以愉快地进行包展开了：

```cpp
// 用 apply_inner 隐藏一下 integer_sequence
template <typename Callable, typename Tuple, size_t... Idx>
decltype(auto) apply_inner(Callable &&callable, Tuple &&tuple,
                           integer_sequence<Idx...>) {
  return invoke(forward<Callable>(callable),
                get<Idx>(forward<Tuple>(tuple))...);
}

template <typename Callable, typename Tuple>
decltype(auto) apply(Callable &&callable, Tuple &&tuple) {
  return apply_inner(forward<Callable>(callable), forward<Tuple>(tuple),
                     make_index_sequence<tuple_size_v<Tuple>>{});
}
```

### `tuple_cat`

最后我们来说一下 **`tuple_cat`**。该函数的作用是，对于任意数量的 `tuple<T...>`{:.language-cpp}, `tuple<U...>`{:.language-cpp}, `tuple<X...>`{:.language-cpp} ...，将它们合并为 `tuple<T..., U..., X..., ...>`{:.language-cpp}。

`tuple_cat` 的原理和 `apply` 比较类似，总而言之我们最终需要将它展开为 `tuple<T..., U..., X..., ...>(get<0>(t1), get<1>(t1), ..., get<0>(t2), get<1>(t2), ...)`{:.language-cpp} 的形式，这样的话，我们仍然需要依赖递归来实现。

我们首先来定义一下递归调用的最后一步，即所有参数都已经展开后的处理：

```cpp
template <typename... T>
auto tuple_cat_inner(T &&...values) {
  tuple<T...>(forward(values)...);
}
```

并不复杂，就是把所有展开后的值重新打包成一个新的 `tuple`。然后我们再想想它的上一级是怎么样的，首先需要输入已经展开完成的值，其次输入一个 `tuple` 类型，然后还要输入一个 `integer_sequence` 辅助它展开；对于更高层级，还需要多添加一个模板类型包来代表其他还没经过处理的 `tuple`，因此我们需要定义：

```cpp
template <typename... T, typename Tuple, size_t... Idx, typename... OtherTuples>
auto tuple_cat_inner(Tuple &&current, OtherTuples &&...others,
                     integer_sequence<Idx...>, T &&...values) {}
```

等等等，等一下，这个函数的模板类型未免也太多了吧！让我们做一点包装让它好看一点：

```cpp
template <typename IdxSeq, typename... Tuples>
struct TupleConcator {};

template <>
struct TupleConcator<integer_sequence<>> {
  template <typename... T>
  static auto tuple_cat_inner(T &&...values) {
    return tuple<T...>(forward(values)...);
  }
};

template <size_t... Idx, typename Tuple, typename... OtherTuples>
struct TupleConcator<integer_sequence<Idx...>, Tuple, OtherTuples...> {
  template <typename... T>
  static auto tuple_cat_inner(Tuple &&current, OtherTuples &&...others,
                              T &&...values) {}
};
```

通过加入 `TupleConcator` 的包装，我们可以把 `Idx...`{:.language-cpp} 到 `integer_sequence<Idx ..>`{:.language-cpp} 的过程移到类模板上完成。

接下来考虑如何实现 `tuple_cat_inner`。由于我们递归调用下一个 `tuple_cat_inner` 时，需要提供对应的 `integer_sequence`，因此我们需要有办法得到 `OtherTuples...`{:.language-cpp} 中第一个 `tuple` 类型的元素数量。实现起来不算复杂：

```cpp
template <typename...>
struct tuple_first_index_sequence;

template <>
struct tuple_first_index_sequence<> {
  using type = integer_sequence<>;
};

template <typename First, typename... Rest>
struct tuple_first_index_sequence<First, Rest...> {
  using type = make_index_sequence<tuple_size_v<First>>;
};

template <typename... Tuples>
using tuple_first_index_sequence_v =
    typename tuple_first_index_sequence<Tuples...>::type;
```

这样，我们就很容易定义出递归调用的下一个 `TupleConCator` 的类型：

```cpp
using next_idx = tuple_first_index_sequence_v<OtherTuples...>;
using next_concator = TupleConcator<next_idx, OtherTuples...>;
```

之后调用 `next_concator` 的 `tuple_cat_inner` 完成递归：

```cpp
template <size_t... Idx, typename Tuple, typename... OtherTuples>
struct TupleConcator<integer_sequence<Idx...>, Tuple, OtherTuples...> {
  template <typename... T>
  static auto tuple_cat_inner(Tuple &&current, OtherTuples &&...others,
                              T &&...values) {
    using next_idx = tuple_first_index_sequence_v<OtherTuples...>;
    using next_concator = TupleConcator<next_idx, OtherTuples...>;

    return next_concator::tuple_cat_inner(forward<OtherTuples>(others)...,
                                          forward<T>(values)...,
                                          get<Idx>(forward<Tuple>(current))...);
  }
};
```

然后再封装 `tuple_cat` 函数作为入口：

```cpp
template <typename... Tuples>
auto tuple_cat(Tuples &&...tuples) {
  using first_idx = tuple_first_index_sequence_v<Tuples...>;
  using first_concator = TupleConcator<first_idx, Tuples...>;

  return first_concator::tuple_cat_inner(forward<Tuples>(tuples)...);
}
```

然而这里有一个坑。当使用 `forward` 来传递值时，会导致 `tuple_cat_inner` 的 `T...`{:.language-cpp} 类型也被推导为引用类型，导致最后生成的 `tuple` 全变成了引用类型。因此我们需要额外传递一个模板类型用来指示生成的 `tuple` 类型。

获取组合后的 `tuple` 的类型可以通过递归包展开来实现：

```cpp
template <typename...>
struct combine_tuples;

template <>
struct combine_tuples<> {
  using type = tuple<>;
};

template <typename... T>
struct combine_tuples<tuple<T...>> {
  using type = tuple<T...>;
};

template <typename... T1, typename... T2, typename... Rest>
struct combine_tuples<tuple<T1...>, tuple<T2...>, Rest...> {
  // 通过递归 using 的形式来依次展开 tuple 的参数
  using type = typename combine_tuples<tuple<T1..., T2...>, Rest...>::type;
};

// 简化调用接口
template <typename... Tuples>
using tuple_cat_result =
    typename combine_tuples<remove_reference_t<Tuples>...>::type;
```

之后，将新的模板参数添加到 `tuple_cat` 中：

```cpp
template <typename Ret, typename IdxSeq, typename... Tuples>
struct TupleConcator {};

template <typename Ret>
struct TupleConcator<Ret, integer_sequence<>> {
  template <typename... T>
  static Ret tuple_cat_inner(T &&...values) {
    // 用传过来的 Ret 类型构造 tuple
    return Ret(forward<T>(values)...);
  }
};

template <typename Ret, size_t... Idx, typename Tuple, typename... OtherTuples>
struct TupleConcator<Ret, integer_sequence<Idx...>, Tuple, OtherTuples...> {
  template <typename... T>
  static Ret tuple_cat_inner(Tuple &&current, OtherTuples &&...others,
                              T &&...values) {
    using next_idx = tuple_first_index_sequence_v<OtherTuples...>;
    using next_concator = TupleConcator<Ret, next_idx, OtherTuples...>;

    return next_concator::tuple_cat_inner(forward<OtherTuples>(others)...,
                                          forward<T>(values)...,
                                          get<Idx>(forward<Tuple>(current))...);
  }
};

// 这里用到了一个后置返回值类型语法，可以把较长的返回值类型放在函数声明的末尾
template <typename... Tuples>
auto tuple_cat(Tuples &&...tuples) -> tuple_cat_result<Tuples...> {
  using first_idx = tuple_first_index_sequence_v<Tuples...>;
  using tuple_ret = tuple_cat_result<Tuples...>;
  using first_concator = TupleConcator<tuple_ret, first_idx, Tuples...>;

  return first_concator::tuple_cat_inner(forward<Tuples>(tuples)...);
}
```

### 引用类型的处理

最后，C++ 还有一个大坑，就是当模板类型只有左值引用类型时，例如 `tuple<int &>`{:.language-cpp}，此时 `tuple_storage` 的构造函数变成了：`tuple_storage(int &const)`{:.language-cpp} 以及 `tuple_storage(int & &&)`{:.language-cpp}，注意，和指针类型一样，前者并不是 `const int &`{:.language-cpp}，而是 `int &const`{:.language-cpp} 这个实际不存在的类型，因此会被视为 `int &`{:.language-cpp}；而后者也会折叠成 `tuple_storage(int &)`{:.language-cpp}，从而导致二者冲突。

这一点有一个非常巧妙的办法解决，只需要为所有使用 `T &&`{:.language-cpp} 的构造函数再加一层模板即可：

```cpp
template <size_t Idx, typename T>
class tuple_storage {
 public:
  tuple_storage(const T &value) : value(value) {}
  template <typename U>
  tuple_storage(U &&value) : value(std::forward<U>(value)) {}
  T value;
};

template <size_t Idx, typename... Elements>
class tuple_inner {};

template <size_t Idx, typename T>
class tuple_inner<Idx, T> : public tuple_storage<Idx, T> {
 public:
  tuple_inner(const T &value) : tuple_storage<Idx, T>(value) {}
  template <typename U>
  tuple_inner(U &&value) : tuple_storage<Idx, T>(std::forward<U>(value)) {}
};

template <size_t Idx, typename T, typename... Rest>
class tuple_inner<Idx, T, Rest...> : public tuple_inner<Idx + 1, Rest...>,
                                     public tuple_storage<Idx, T> {
 public:
  tuple_inner(const T &value, const Rest &...rest)
      : tuple_inner<Idx + 1, Rest...>(rest...), tuple_storage<Idx, T>(value) {}
  template <typename U, typename... Inputs>
  tuple_inner(U &&value, Inputs &&...rest)
      : tuple_inner<Idx + 1, Rest...>(std::forward<Inputs>(rest)...),
        tuple_storage<Idx, T>(std::forward<U>(value)) {}
};

template <typename... Elements>
class tuple : public tuple_inner<0, Elements...> {
 public:
  tuple(const Elements &...elements)
      : tuple_inner<0, Elements...>(elements...) {}
  template <typename... Inputs>
  tuple(Inputs &&...elements)
      : tuple_inner<0, Elements...>(std::forward<Inputs>(elements)...) {}
};
```

当 `tuple` 的类型仅为左值引用时，由于 `const`{:.language-cpp} 约束对引用类型没有意义，因为引用类型自身不可改变指向，因此相比于模板构造函数，`const Element &...`{:.language-cpp} 版本属于模板特化，编译器选择该版本。

另一方面，`get` 的右值版本实现也需要改一改，使用 `forward` 替换 `move`:

```cpp
template <size_t Idx, typename T>
T &&get(tuple_storage<Idx, T> &&t) {
  return forward<T>(t.value);
}

template <size_t Idx, typename T>
const T &&get(const tuple_storage<Idx, T> &&t) {
  return forward<T>(t.value);
}
```

## 柯里化

事实上，当你看到一个程序员在说**柯里化（Currying）**时，他很可能说的是**部分应用（Partial Application）**。

部分应用指的是，提供函数所需的部分参数，你可以得到一个可以接受剩下参数的函数：

```cpp
auto g = partial(f, a, b, c);
auto result = g(d, e);
```

而柯里化指的是，将一个多参函数转换为一系列接受单个参数的函数：

```cpp
auto g = curry(f);
auto h = g(a)(b)(c);
auto result = h(d)(e);
```

可以看出来，相比于柯里化，实际上部分应用更符合我们编程的直觉。另一方面，正如我以前在 [Rust 中的闭包递归与 Y 组合子]({{ site.baseurl }}{% link _posts/2021-6-7-rust_closure_and_y.md %})提到过，数学上尤其是在 lambda 演算中，对于函数应用这一操作的定义并非那么严格：lambda 演算 `f a b` 和 `(f a) b` 是等效的。这也就是说，在数学的视角中 `f(a, b)` 和 `f(a)(b)` 只是函数应用的两种不同写法而已。这也就意味着，事实上我们可以将部分应用视为是在某个固定点对函数进行柯里化，可以为 `curry` 函数添加下标来表示其固定点位置，即 $$partial(f)_a = curry_1(f)(a)$$，反过来说，我们也可以将柯里化视为递归地对函数的第一个参数进行部分应用。

实现真正的柯里化是很简单的，因为对于每个中间函数对象而言都只需要处理一个模板类型参数。而实现部分应用要麻烦的很多，因为部分应用所固定的参数数量，和生成的函数对象能接受的剩余参数的数量都是不定的，对于写模板类型而言更具有挑战性，因此，下文中我们将**默认柯里化为部分应用**来实现代码逻辑。

首先我们需要构造一个类似 lambda 的类型，用来表示柯里化之后的函数，因此这个类型需要保存原始函数和柯里化的参数，然后再提供 `operator()`{:.language-cpp} 接受那些未参与柯里化的参数。因此，先写出基本的结构：

```cpp
template <typename Callable, typename... CurriedArgs>
class Curried {};

template <typename Ret, typename... CurriedArgs, typename... UncurriedArgs>
class Curried<Function<Ret(CurriedArgs..., UncurriedArgs...)>, CurriedArgs...> {
 public:
  // 使用模板类型处理左值和右值的问题
  template <typename Callable>
  Curried(Callable &&callable, tuple<CurriedArgs...> &&args)
      : _callable(forward<Callable>(callable)), _curriedArgs(move(args)) {}

  // 提供 operator() 接受剩余的参数
  Ret operator()(UncurriedArgs... args) const {
    auto uncurriedArgs =
        tuple<UncurriedArgs...>(forward<UncurriedArgs>(args)...);
    return apply(_callable, tuple_cat(_curriedArgs, move(uncurriedArgs)));
  };
 private:
  // 保存原始函数
  Function<Ret(CurriedArgs..., UncurriedArgs...)> _callable;
  // 保存柯里化的参数，此处加上 mutable 主要是方便 const Curried 时的调用
  mutable tuple<CurriedArgs...> _curriedArgs;
};
```

不过这种写法没法正常使用，编译器几乎无法对 `CurriedArgs` 和 `UncurriedArgs` 进行推导。我们可以降低一下模板类型的复杂度，把 `Function` 类型用一个模板类型来表示，并且将模板参数包用 `tuple` 封装起来：

```cpp
template <typename Callable, typename ArgsTuple, typename UncurriedArgsTuple>
class Curried {};

// 用 Callable 来表示 Function 类型，降低复杂度
template <typename Callable, typename... CurriedArgs, typename... UncurriedArgs>
class Curried<Callable, tuple<CurriedArgs...>, tuple<UncurriedArgs...>> {
 public:
  template <typename CallableT>
  Curried(CallableT &&callable, tuple<CurriedArgs...> &&args)
      : _callable(forward<CallableT>(callable)), _curriedArgs(move(args)) {}

  decltype(auto) operator()(UncurriedArgs... args) const {
    auto uncurriedArgs =
        tuple<UncurriedArgs...>(forward<UncurriedArgs>(args)...);
    return apply(_callable, tuple_cat(_curriedArgs, move(uncurriedArgs)));
  };

 private:
  Callable _callable;
  mutable tuple<CurriedArgs...> _curriedArgs;
};
```

但是当我们想要封装一个柯里化函数时，又有新的问题出现了：

```cpp
template <typename Ret, typename... CurriedArgs, typename... UncurriedArgs>
auto curry(Function<Ret(CurriedArgs..., UncurriedArgs...)> callable,
           UncurriedArgs... args) {}
```

我们尝试写出柯里化函数的声明，但是这里有一个严重的问题：对于类型 `Function<Ret(CurriedArgs..., UncurriedArgs...)>`{:.language-cpp} 而言，编译器无法判断 `CurriedArgs` 和 `UncurriedArgs` 的分界线究竟在哪，即使我们后面还引入了 `UncurriedArgs ...args`{:.language-cpp}。

那么我们需要一个办法来让编译器知道如何划分两个参数包的界限，我们引入一个新的参数包:

```cpp
template <typename Ret, typename... CurriedArgs, typename... UncurriedArgs, typename... Args>
auto curry(Function<Ret(CurriedArgs..., UncurriedArgs...)> callable,
           Args &&...args) {}
```

`Args` 参数包是可以正常推导出来的，并且我们知道 `Args` 的包大小等于 `UncurriedArgs` 的包大小。这样的话，我们需要有办法可以得到一个模板参数包的前 N 个参数，以及去除前 N 个参数之后剩余的参数。

当然，有了前面一系列的模板元编程的经验，这对我们来说并不算难，我们可以借助 `tuple` 和 `integer_sequence` 的帮助来实现 `args_head`：

```cpp
template <typename IdxSeq, typename... Args>
struct args_head {};

// 当索引序列为空时，类型为空 tuple
template <typename... Args>
struct args_head<integer_sequence<>, Args...> {
  using type = tuple<>;
};

// 借助索引序列递归拆分参数包，并使用 tuple_cat_result 连接参数列表
template <typename T, typename... Rest, size_t N, size_t... Idx>
struct args_head<integer_sequence<N, Idx...>, T, Rest...> {
  using type = tuple_cat_result<
      tuple<T>, typename args_head<integer_sequence<Idx...>, Rest...>::type>;
};

// 包装一个好看的接口给外部调用
template <size_t N, typename... Args>
using args_head_t = typename args_head<make_index_sequence<N>, Args...>::type;
```

这样，`args_head_t<2, T1, T2, T3, T4>`{:.language-cpp} 的类型就会解析为 `tuple<T1, T2>`{:.language-cpp}。

对于去除前 N 个参数，保留剩下的参数的情况则更加简单。为了避免误会 N 的含义，我们将其命名为 `args_except`：

```cpp
template <typename IdxSeq, typename... Args>
struct args_except {};

// 索引序列为空时，剩下的类型就是所有我们需要的类型
template <typename... Args>
struct args_except<integer_sequence<>, Args...> {
  using type = tuple<Args...>;
};

// 递归参数包和索引列表，以跳过前 N 个类型
template <typename T, typename... Rest, size_t N, size_t... Idx>
struct args_except<integer_sequence<N, Idx...>, T, Rest...> {
  using type = typename args_except<integer_sequence<Idx...>, Rest...>::type;
};

// 包装一个好看的接口给外部调用
template <size_t N, typename... Args>
using args_except_t =
    typename args_except<make_index_sequence<N>, Args...>::type;
```

这样，我们的 `curry` 函数可以通过下面的方法来实现：

```cpp
template <typename CurriedArgsTuple, typename UncurriedArgsTuple>
struct Curry {};

// 包装类型，主要目的是将 tuple<T...> 中的 T... 拆出来
template <typename... CurriedArgs, typename... UncurriedArgs>
struct Curry<tuple<CurriedArgs...>, tuple<UncurriedArgs...>> {
  // 左值版本
  template <typename Ret, typename... Args>
  static auto curry_inner(
      const Function<Ret(CurriedArgs..., UncurriedArgs...)> &callable,
      Args &&...args) {
    // 定义 Callable 类型为 Function 类型
    using Callable = Function<Ret(CurriedArgs..., UncurriedArgs...)>;
    // 定义对应的 Curried 类型
    using CurriedT =
        Curried<Callable, tuple<CurriedArgs...>, tuple<UncurriedArgs...>>;

    return CurriedT(callable, tuple<CurriedArgs...>(forward<Args>(args)...));
  }

  // 右值版本，和上面那个函数唯一的区别就是 callable 是右值
  template <typename Ret, typename... Args>
  static auto curry_inner(
      Function<Ret(CurriedArgs..., UncurriedArgs...)> &&callable,
      Args &&...args) {
    using Callable = Function<Ret(CurriedArgs..., UncurriedArgs...)>;
    using CurriedT =
        Curried<Callable, tuple<CurriedArgs...>, tuple<UncurriedArgs...>>;

    return CurriedT(move(callable),
                    tuple<CurriedArgs...>(forward<Args>(args)...));
  }
};

// 左值版本
template <typename Ret, typename... FullArgs, typename... Args>
auto curry(const Function<Ret(FullArgs...)> &callable, Args &&...args) {
  using CurriedArgsTuple = args_head_t<sizeof...(Args), FullArgs...>;
  using UncurriedArgsTuple = args_except_t<sizeof...(Args), FullArgs...>;
  using CurryWrapper = Curry<CurriedArgsTuple, UncurriedArgsTuple>;

  return CurryWrapper::curry_inner(callable, forward<Args>(args)...);
}

// 右值版本，和上面那个函数唯一的区别就是 callable 是右值
template <typename Ret, typename... FullArgs, typename... Args>
auto curry(Function<Ret(FullArgs...)> &&callable, Args &&...args) {
  using CurriedArgsTuple = args_head_t<sizeof...(Args), FullArgs...>;
  using UncurriedArgsTuple = args_except_t<sizeof...(Args), FullArgs...>;
  using CurryWrapper = Curry<CurriedArgsTuple, UncurriedArgsTuple>;

  return CurryWrapper::curry_inner(move(callable), forward<Args>(args)...);
}
```

我们给普通函数写一个包装，将创建 `Function` 隐藏起来：

```cpp
template <typename Ret, typename... FullArgs, typename... Args>
auto curry(Ret(*func)(FullArgs...), Args &&...args) {
  Function<Ret(FullArgs...)> callable = func;
  return curry(move(callable), forward<Args>(args)...);
}
```

简单验证一下功能：

```cpp
#include <iostream>

void print(int a, double b, char c, const char *d) {
  std::cout << a << " " << b << " " << c << " " << d << std::endl;
}

int main() {
  auto curried = curry(print, 1919, 114.514);
  curried('a', "hello world");
  return 0;
}
```

该例子在 GCC 和 Clang 环境下以 C++20 编译均无报错，且运行结果都正确。

然后我们还可以给成员函数也写一份重载，需要注意，我们需要额外考虑不提供柯里化参数的情况：

```cpp
// 没有柯里化参数的情况需要额外考虑
template <typename Ret, typename T, typename... FullArgs>
auto curry(Ret (T::*func)(FullArgs...)) {
  Function<Ret(T *, FullArgs...)> callable = func;
  return curry(move(callable));
}

// 正常情况
template <typename Ret, typename T, typename... FullArgs, typename... Args>
auto curry(Ret (T::*func)(FullArgs...), T *caller, Args &&...args) {
  Function<Ret(T *, FullArgs...)> callable = func;
  return curry(move(callable), caller, forward<Args>(args)...);
}
```

最后，也是最关键的，当被柯里化的函数是 `Curried` 本身时，我们有两种方法，简单的方法是把 `Curried` 打包到 `Function` 里视为一个普通函数再次进行柯里化包装；但是我们用另一种方法，**利用 `tuple_cat` 生成一个新的 `Curried`**。

因为要访问 `Curried` 的内部变量，因此我们先给 `Curried` 实现一个 `curry` 成员函数：

```cpp
template <typename Callable, typename... CurriedArgs, typename... UncurriedArgs>
class Curried<Callable, tuple<CurriedArgs...>, tuple<UncurriedArgs...>> {
 public:
  template <typename... Args>
  // 当 Curried 自身是左值时
  auto curry(tuple<Args...> &&args) const & {
    using NewCurried =
        Curried<Callable,
                tuple_cat_result<tuple<CurriedArgs...>, tuple<Args...>>,
                args_except_t<sizeof...(Args), UncurriedArgs...>>;

    return NewCurried(_callable, tuple_cat(_curriedArgs, args));
  }

  // 当 Curried 自身是右值时，可以 move 自己的 _callable 和 _curriedArgs
  template <typename... Args>
  auto curry(tuple<Args...> &&args) && {
    using NewCurried =
        Curried<Callable,
                tuple_cat_result<tuple<CurriedArgs...>, tuple<Args...>>,
                args_except_t<sizeof...(Args), UncurriedArgs...>>;

    return NewCurried(move(_callable), tuple_cat(move(_curriedArgs), args));
  }

  ...
};
```

然后封装全局 `curry` 函数：

```cpp
template <typename... CurriedArgsList, typename... Args>
auto curry(const Curried<CurriedArgsList...> &curried, Args &&...args) {
  return curried.curry(tuple<Args...>(forward<Args>(args)...));
}

template <typename... CurriedArgsList, typename... Args>
auto curry(Curried<CurriedArgsList...> &&curried, Args &&...args) {
  return curried.curry(tuple<Args...>(forward<Args>(args)...));
}
```

最后，你会发现，尽管我们实现的是部分应用，但是却隐含了实现真正的柯里化。柯里化无非是 `g(a)(b)(c)`{:.language-cpp}，而 `g(a)`{:.language-cpp} 这一操作在 C++ 中的本质是 `g.operator()(a)`{:.language-cpp}，它的功能实际上正是对应了我们的 `curried.curry(a)`{:.language-cpp}，只需要我们把 `Curried::curry`{:.language-cpp} 函数的实现转移到 `Curried::operator()`{:.language-cpp} 上，并且使用 `if constexpr`{:.language-cpp} 判断如果是最后一个所需的参数就调用内部函数，那么我们就轻易地把它改造成了真正的柯里化。

## 最终实现

由于完整代码实在太长，因此我将其上传到 [Github Gist](https://gist.github.com/NichtsHsu/9b2f490675c71cda641a3a137778f70e) 上以方便查看。

另外，我还上传了一个[使用标准库的版本](https://gist.github.com/NichtsHsu/030271c510249edd3bbd11cb90468003)以供参考。

## 参考

GCC 标准库源码

[cppreference](https://en.cppreference.com/)

[details of std::make_index_sequence and std::index_sequence - stack overflow](https://stackoverflow.com/questions/49669958/details-of-stdmake-index-sequence-and-stdindex-sequence)
