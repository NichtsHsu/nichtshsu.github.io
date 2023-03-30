---
title: "[C++] std::function 是如何实现 lambda 递归的"
date: 2023-3-30 14:20:18 +0800
categories: [杂记, Cpp]
tags: [c++, 编程语言]     # TAG names should always be lowercase
---

我们都知道，C++ 是不允许 lambda 函数递归调用自己的，如果想要递归，最好的办法就是使用 `std::function`{:.language-cpp}：

```cpp
#include <functional>
#include <iostream>

int main() {
  std::function<unsigned(unsigned)> fact = [&fact](unsigned x) -> unsigned {
    return x <= 1u ? 1.0 : x * fact(x - 1);
  };
  std::cout << fact(10) << std::endl;

  return 0;
}
```
{: run="cpp" }

`std::function`{:.language-cpp} 究竟有什么魔力，可以让 lambda 进行递归？鲁迅说的好，**要想明白一个东西的原理，最好的办法就是从头实现一遍**。鲁迅：这是周树人说的，不是我说的。

让我们先简单分析一下，假设我们实现的类名为 `Closure`，那么我们需要知道 lambda 的**返回值类型**和**参数类型**，这样才能重载 `Closure` 的 `operator()`{:.language-cpp} 以调用 lambda 函数（注：为了保持参数的引用类型，后续所有代码都会使用万能引用以及 `std::forward`{:.language-cpp} 进行完美转发）：

```cpp
template <typename Ret, typename... Args>
class Closure {
 public:
  typedef Ret (*Func)(Args...);
  Closure(Func&& fp) : _fp(std::forward<Func>(fp)) {}

  Ret operator()(Args&&... args) const { return _fp(std::forward<Args>(args)...); }

 private:
  Func _fp;
};
```

当然，事情远没有那么简单，上述 `Closure` 类型虽然可以包装无捕捉的简单 lambda 函数，但是**无法包装有捕捉的 lambda 函数**，更别说递归：

```cpp
#include <iostream>

template <typename Ret, typename... Args>
class Closure {
 public:
  typedef Ret (*Func)(Args...);
  Closure(Func&& fp) : _fp(std::forward<Func>(fp)) {}

  Ret operator()(Args&&... args) const { return _fp(std::forward<Args>(args)...); }

 private:
  Func _fp;
};

int main() {
  // 可以包装无捕捉的 lambda 函数
  Closure<int, int> addOne = (Closure<int, int>::Func)[](int x) { return x + 1; };
  std::cout << addOne(10) << std::endl;

  // 但是不能包装有捕捉的 lambda 函数
  // int m = 12;
  // Closure<int, int> addm = (Closure<int, int>::Func)[m](int x) { return x + m; };
  // std::cout << addm(10) << std::endl;

  return 0;
}
```
{: run="cpp" }

不过我们先不着急，我们先处理两个小问题。

一是，我们使用 `Closure<int, int>`{:.language-cpp} 而不是 `Closure<int(int)>`{:.language-cpp} 声明类型，这个问题可以用**模板特化**来解决：

```cpp
#include <iostream>

template <typename T>
class Closure {
  Closure() = delete;
};

template <typename Ret, typename... Args>
class Closure<Ret(Args...)> {
 public:
  typedef Ret (*Func)(Args...);
  Closure(Func&& fp) : _fp(std::forward<Func>(fp)) {}

  Ret operator()(Args&&... args) const { return _fp(std::forward<Args>(args)...); }

 private:
  Func _fp;
};

int main() {
  Closure<int(int)> addOne = (Closure<int(int)>::Func)[](int x) { return x + 1; };
  std::cout << addOne(10) << std::endl;

  return 0;
}
```
{: run="cpp" }

二是，我们不得不将闭包进行**显式类型转换**，这是因为复制初始化（即使用等号初始化）对于隐式类型转换的要求比直接初始化（即使用括号初始化）更严格。既然不允许在这里隐式类型转换，那么我们可以换个思路，我们把构造函数改成**泛型构造函数**，把强制类型转换放到构造函数中进行：

```cpp
#include <iostream>

template <typename T>
class Closure {
  Closure() = delete;
};

template <typename Ret, typename... Args>
class Closure<Ret(Args...)> {
 public:
  typedef Ret (*Func)(Args...);

  template<typename Lambda>
  Closure(Lambda&& fp) : _fp(std::forward<Lambda>(fp)) {}

  Ret operator()(Args&&... args) const { return _fp(std::forward<Args>(args)...); }

 private:
  Func _fp;
};

int main() {
  Closure<int(int)> addOne = [](int x) { return x + 1; };
  std::cout << addOne(10) << std::endl;

  return 0;
}
```
{: run="cpp" highlight-lines="13, 14" }

现在我们该来处理一下有捕捉的 lambda 函数的问题了。

我们知道，lambda 函数可以被视为一个匿名类，其成员变量就是被捕捉的环境变量或其引用，然后还重载了 `operator()`{:.language-cpp} 用来执行 lambda 函数的函数体。这就意味着，我们要有办法可以**保存一个可能是任意结构的匿名类型**。

注意直接将 lambda 函数的类型作为泛型类型，然后使用该泛型类型来声明一个成员变量是**不可行**的，因为 lambda 函数想要捕捉一个环境变量，这个环境变量的类型大小必须是已知的，如果 `Closure` 类型依赖 lambda 函数的类型，那么它就无法在 lambda 函数的函数体中被捕捉。

既然如此，我们还是只能储存 lambda 函数的指针，因为指针的大小是固定的。但是由于我们不知道该指针的具体类型，因此我们考虑使用 `void *`{:.language-cpp}。但是使用 `void *`{:.language-cpp} 意味着丢失了 lambda 函数的类型，我们还需要通过其他方法再拿回它的类型。

我们先把想到的东西先写下，一步一步走：

```cpp
template <typename T>
class Closure {
  Closure() = delete;
};

template <typename Ret, typename... Args>
class Closure<Ret(Args...)> {
 public:
  template <typename Lambda>
  Closure(Lambda&& fp) {
    // 我们通过 new 把 lambda 函数转移到堆上，这样我们就不需要直接持有它
    _fp = new Lambda(std::forward<Lambda>(fp));
  }

  // 禁止拷贝构造
  Closure(const Closure&) = delete;

  // 允许移动构造
  Closure(Closure &&other) {
    _fp = other._fp;
    other._fp = nullptr;
  }

  Ret operator()(Args... args) const {
    // 我们不知道 lambda 的实际类型，因此这里暂时无法实现
  }

 private:
  void* _fp;
};
```

现在的关键是我们无法在 `operator()`{:.language-cpp} 中得到 lambda 函数的实际类型，该类型只有在构造函数中才知道。因此，我们需要借助另一个类型来保存 lambda 函数的类型，假设名为 `_Base`：

```cpp
template <typename Ret, typename Lambda, typename... Args>
class _Base {
 public:
  // TODO
};
```

显然，我们也不能在 `Closure` 类中直接持有 `_Base` 类型的对象或者其指针，因为我们无法知道 `Lambda` 泛型究竟是什么。这样的话，`_Base` 类必须**向外提供一个 static 的变量或者函数**，并且这个 static 的变量或函数的类型**不包含 `Lambda` 泛型**，这样才能让 `Closure` 类可以持有 `_Base` 类的 static 变量或函数指针。

其实到这里答案就很明显了。不妨停下来，尝试一下自己能否补全这个 `_Base` 类。

好了，不卖关子了，我们只需要将 `_Base` 类实现为：

```cpp
template <typename Ret, typename Lambda, typename... Args>
class _Base {
 public:
  static Ret invoke(void* fp, Args&&... args) {
    // 在函数体内使用 Lambda 泛型类型，而不是让函数的调用者提供
    return (*reinterpret_cast<Lambda*>(fp))(std::forward<Args>(args)...);
  }
};
```

注意到 `_Base::invoke`{:.language-cpp} 函数，它是一个 static 函数，并且该函数的类型是 `Ret (*)(void*, Args&&...)`{:.language-cpp}，这意味着我们可以不依赖 `Lambda` 泛型就能储存它的函数指针：

```cpp
template <typename Ret, typename Lambda, typename... Args>
class _Base {
 public:
  static Ret invoke(void* fp, Args&&... args) {
    // 在函数体内使用 Lambda 泛型类型，而不是让函数的调用者提供
    return (*reinterpret_cast<Lambda*>(fp))(std::forward<Args>(args)...);
  }
};

template <typename T>
class Closure {
  Closure() = delete;
};

template <typename Ret, typename... Args>
class Closure<Ret(Args...)> {
 public:
  template <typename Lambda>
  Closure(Lambda&& fp) {
    _fp = new Lambda(std::forward<Lambda>(fp));
    // 我们通过储存 _Base 类的 invoke 函数的指针来间接储存 lambda 的类型
    _invoke = _Base<Ret, Lambda, Args...>::invoke;
  }
  Closure(const Closure&) = delete;
  Closure(Closure &&other) {
    _fp = other._fp;
    other._fp = nullptr;
    _invoke = other._invoke;
  }

  Ret operator()(Args&&... args) const {
    // 最终实际调用的是 _Base 类的 invoke 函数
    return _invoke(_fp, std::forward<Args>(args)...);
  }

 private:
  void* _fp;
  // _invoke 指针不依赖 lambda 函数的实际类型
  Ret (*_invoke)(void*, Args&&...);
};
```

当然，由于我们用到了 `new`{:.language-cpp} 来储存 lambda 函数，因此我们也需要一个合适的方法去 `delete`{:.language-cpp} 掉它。和 `invoke` 同理，我们可以给 `_Base` 实现一个 `free` 函数：

```cpp
template <typename Ret, typename Lambda, typename... Args>
class _Base {
 public:
  static Ret invoke(void* fp, Args&&... args) {
    // 在函数体内使用 Lambda 泛型类型，而不是让函数的调用者提供
    return (*reinterpret_cast<Lambda*>(fp))(std::forward<Args>(args)...);
  }

  static void free(void* fp) {
    // 在函数体内使用 Lambda 泛型类型，而不是让函数的调用者提供
    delete reinterpret_cast<Lambda*>(fp);
  }
};

template <typename T>
class Closure {
  Closure() = delete;
};

template <typename Ret, typename... Args>
class Closure<Ret(Args...)> {
 public:
  template <typename Lambda>
  Closure(Lambda&& fp) {
    _fp = new Lambda(std::forward<Lambda>(fp));
    // 我们通过储存 _Base 类的 invoke 函数和 free 函数的指针来间接储存 lambda 的类型
    _invoke = _Base<Ret, Lambda, Args...>::invoke;
    _free = _Base<Ret, Lambda, Args...>::free;
  }
  Closure(const Closure&) = delete;
  Closure(Closure &&other) {
    _fp = other._fp;
    other._fp = nullptr;
    _invoke = other._invoke;
    _free = other._free;
  }
  ~Closure() {
    // 通过 _Base 类的 free 函数来释放堆内存
    _free(_fp);
  }

  Ret operator()(Args&&... args) const {
    // 最终实际调用的是 _Base 类的 invoke 函数
    return _invoke(_fp, std::forward<Args>(args)...);
  }

 private:
  void* _fp;
  // _invoke 和 _free 的类型不依赖 lambda 函数的实际类型
  Ret (*_invoke)(void*, Args&&...);
  void (*_free)(void*);
};
```

我们可以将 `_Base` 类移动到 `Closure` 类内部定义，这样可以避免 `_Base` 类被直接访问，也可以少些两个泛型参数：

```cpp
template <typename T>
class Closure {
  Closure() = delete;
};

template <typename Ret, typename... Args>
class Closure<Ret(Args...)> {
 private:
  // 模板参数 Ret 和 Args 由 Closure 类提供，这里可以不用写了
  template <typename Lambda>
  class _Base {
   public:
    static Ret invoke(void* fp, Args&&... args) {
      // 在函数体内使用 Lambda 泛型类型，而不是让函数的调用者提供
      return (*reinterpret_cast<Lambda*>(fp))(std::forward<Args>(args)...);
    }

    static void free(void* fp) {
      // 在函数体内使用 Lambda 泛型类型，而不是让函数的调用者提供
      delete reinterpret_cast<Lambda*>(fp);
    }
  };

 public:
  template <typename Lambda>
  Closure(Lambda&& fp) {
    _fp = new Lambda(std::forward<Lambda>(fp));
    // 我们通过储存 _Base 类的 invoke 函数和 free 函数的指针来间接储存 lambda 的类型
    _invoke = _Base<Lambda>::invoke;
    _free = _Base<Lambda>::free;
  }
  Closure(const Closure&) = delete;
  Closure(Closure &&other) {
    _fp = other._fp;
    other._fp = nullptr;
    _invoke = other._invoke;
    _free = other._free;
  }
  ~Closure() {
    // 通过 _Base 类的 free 函数来释放堆内存
    _free(_fp);
  }

  Ret operator()(Args&&... args) const {
    // 最终实际调用的是 _Base 类的 invoke 函数
    return _invoke(_fp, std::forward<Args>(args)...);
  }

 private:
  void* _fp;
  // _invoke 和 _free 的类型不依赖 lambda 函数的实际类型
  Ret (*_invoke)(void*, Args&&...);
  void (*_free)(void*);
};
```

让我们来测试一下它能不能做到闭包递归：

```cpp
#include <iostream>

template <typename T>
class Closure {
  Closure() = delete;
};

template <typename Ret, typename... Args>
class Closure<Ret(Args...)> {
 private:
  // 模板参数 Ret 和 Args 由 Closure 类提供，这里可以不用写了
  template <typename Lambda>
  class _Base {
   public:
    static Ret invoke(void* fp, Args&&... args) {
      // 在函数体内使用 Lambda 泛型类型，而不是让函数的调用者提供
      return (*reinterpret_cast<Lambda*>(fp))(std::forward<Args>(args)...);
    }

    static void free(void* fp) {
      // 在函数体内使用 Lambda 泛型类型，而不是让函数的调用者提供
      delete reinterpret_cast<Lambda*>(fp);
    }
  };

 public:
  template <typename Lambda>
  Closure(Lambda&& fp) {
    _fp = new Lambda(std::forward<Lambda>(fp));
    // 我们通过储存 _Base 类的 invoke 函数和 free 函数的指针来间接储存 lambda 的类型
    _invoke = _Base<Lambda>::invoke;
    _free = _Base<Lambda>::free;
  }
  Closure(const Closure&) = delete;
  Closure(Closure &&other) {
    _fp = other._fp;
    other._fp = nullptr;
    _invoke = other._invoke;
    _free = other._free;
  }
  ~Closure() {
    // 通过 _Base 类的 free 函数来释放堆内存
    _free(_fp);
  }

  Ret operator()(Args&&... args) const {
    // 最终实际调用的是 _Base 类的 invoke 函数
    return _invoke(_fp, std::forward<Args>(args)...);
  }

 private:
  void* _fp;
  // _invoke 和 _free 的类型不依赖 lambda 函数的实际类型
  Ret (*_invoke)(void*, Args&&...);
  void (*_free)(void*);
};

int main() {
  Closure<unsigned(unsigned)> fact = [&fact](unsigned x) {
    return x <= 1 ? 1 : x * fact(x - 1);
  };
  // 输出 3628800
  std::cout << fact(10) << std::endl;

  return 0;
}
```
{: run="cpp" }

Perfect! `fact(10)`{:.language-cpp} 成功输出了 10 的阶乘。

当然，`std::function`{:.language-cpp} 的东西远不止这么点，不过最核心的功能已经在本文中介绍了，剩下那些边角料，有兴趣的话就请自行翻阅源代码吧~

最后，在 C++23 的提案中，有一个新的语法称为 **deducing this**，它允许类成员函数将一个显式对象作为其第一个参数，如果该语法得到实施，这意味着我们可以简单的写出这样的代码来递归 lambda函数：

```cpp
#include<iostream>

int main()
{
    auto fact = [](this auto &&self, unsigned x) {
        return x <= 1 ? 1 : x * self(x - 1);
    };
    std::cout << fact(10) << std::endl;
    return 0;
}
```