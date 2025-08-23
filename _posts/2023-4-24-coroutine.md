---
title: "初探 C++20 Coroutine"
date: 2023-4-24 12:57:36 +0800
categories: [教程, Cpp]
tags: [c++, 编程语言, c++20, 教程]     # TAG names should always be lowercase
---

## 前言

近段时间研究了一下 C++20 的协程（Coroutine），大概了解了其中的工作原理，做一下记录。

初次接触 Coroutine 时，给我的感觉是一脸懵逼的。和其他语言简单的 async、await 不同，想要使用 C++20 的 Coroutine，它要求你定义一个包含 `promise_type` 的类型，其中 `promise_type` 又需要至少包含 `get_return_object`, `initial_suspend`, `final_suspend`, `return_void` 和 `unhandled_exception` 函数；还没完，`co_await` 表达式还要你实现一个 `awaitable` 类型，这个 `awaitable` 类型至少需要实现 `await_ready`, `await_suspend` 和 `await_resume`。这一大片东西呼过来，相信很少有人能不晕。让我们一个一个来看它们究竟是什么。

## `co_await` 与 `awaitable` 对象

在程序中，我们很容易遇到阻塞的情况，例如，等待 socket 数据包，等待数据库返回查询结果等等，通常为了避免这些阻塞的操作影响主线程，我们会单独开一个新的线程去做这些操作。

我们简单模拟一个阻塞操作：

```cpp
#include <chrono>
#include <functional>
#include <thread>

class AddOne {
 public:
  // 注意构造函数本身不阻塞主线程
  AddOne(int x, std::function<void(int)> result_ready_cb)
      : _thread{[=, result_ready_cb = std::move(result_ready_cb)]() mutable {
          // 假装阻塞了 5s 才得到结果
          std::this_thread::sleep_for(std::chrono::seconds(5));
          result_ready_cb(x + 1);
        }} {}

  ~AddOne() {
    // 如果 AddOne 析构时线程还没有完成，我们 detach 这个线程
    if (_thread.joinable()) _thread.detach();
  }

  // 但是 wait_for_result 有可能阻塞主线程
  void wait_for_result() { _thread.join(); }

 private:
  std::thread _thread;
};
```

需要注意的是：我们在 AddOne 中使用 `std::thread` 是为了**模拟一个阻塞操作**。

为了避免这个阻塞操作影响我们的主线程，所以我们开一个单独的线程去执行它：

```cpp
#include <chrono>
#include <functional>
#include <iostream>
#include <thread>

class AddOne {
 public:
  // 注意构造函数本身不阻塞主线程
  AddOne(int x, std::function<void(int)> result_ready_cb)
      : _thread{[=, result_ready_cb = std::move(result_ready_cb)]() mutable {
          // 假装阻塞了 5s 才得到结果
          std::this_thread::sleep_for(std::chrono::seconds(5));
          result_ready_cb(x + 1);
        }} {}

  ~AddOne() {
    // 如果 AddOne 析构时线程还没有完成，我们 detach 这个线程
    if (_thread.joinable()) _thread.detach();
  }

  // 但是 wait_for_result 有可能阻塞主线程
  void wait_for_result() { _thread.join(); }

 private:
  std::thread _thread;
};

int main(int argc, char* argv[]) {
  int value = 1;
  int result1 = 0;
  bool result1_ready = false;
  int result2 = 0;
  bool result2_ready = false;
  std::function<void(int)> result_handle1 = [&](int _result) mutable {
    result1 = _result + 1;
    result1_ready = true;
  };
  std::function<void(int)> result_handle2 = [&](int _result) mutable {
    result2 = _result * _result;
    result2_ready = true;
  };

  // 为了避免 add_one.wait_for_result
  // 阻塞主线程，所以我们将它放到一个新线程中执行
  std::thread thread([&]() {
    AddOne add_one(value, result_handle1);
    add_one.wait_for_result();
  });

  // 如果还需要等待另一个阻塞数据，那就需要再开一个线程
  std::thread thread2([&]() {
    AddOne add_one(value, result_handle2);
    add_one.wait_for_result();
  });

  // std::thread 不会阻塞主线程，我们继续做其他事情...
  // ...
}
```

注意到 `AddOne` 有一个特点，它可以注册一个回调函数用来通知我们是否已经准备好了数据。在线程模型中，我们除了在线程中干等它结束别无他法，但是在协程模型中，我们可以在阻塞时暂停这个任务，让执行这个任务的线程先去执行其他的任务，等到 `AddOne` 调用回调函数通知我们时，我们再恢复这个任务。

执行“**暂停**”这一操作的运算符是 **`co_await`**，它需要一个 **`awaitable` 对象**，当调用 `co_await awaitable` 时，它会暂停该任务，让当前线程去处理其他任务（准确地说，是将控制返回给当前协程的调用方），直到被暂停的任务被恢复（通过协程句柄的 `resume()` 函数）——需要注意的是，被恢复时可能在另一个线程上，这是协程的特点之一，可以在被暂停时自由地在线程之间传递。

为此，我们需要定义一个 `awaitable` 类型，该类型主要需要实现下面的函数：

* **`await_ready()`**：在被 `co_await` 时是否已经准备好。如果返回 `false`，那么 `co_await awaitable` 就会立即暂停该协程，然后调用 `awaitable.await_suspend()`。如果返回 `true`，说明数据已经准备好了，那么 `co_await` 就没必要暂停协程了。因为我们的 `AddOne` 必然阻塞 5s，所以这里我们直接返回 `false` 就行了。
* **`await_suspend()`**：如上所述，如果协程被暂停，该函数会被调用，我们需要在这个函数中注册回调函数，并且在回调函数中调用协程句柄的 `resume()` 函数以表示数据已经准备好，恢复协程。该函数返回 `void` 或 `true` 时，协程暂停，将控制返还给调用者；返回 `false` 时，立即恢复该协程。如果返回其他协程的协程句柄，则立即恢复那个协程。
* **`await_resume()`**：当协程句柄的 `resume()` 函数被调用时，该函数被调用。该函数的返回值就是 `co_await awaitable` 的返回值。

整合上述信息，我们为 `AddOne` 实现一个 `AddOneAwaitable` 类型：

```cpp
#include <chrono>
#include <functional>
#include <iostream>
#include <thread>
#include <coroutine>

class AddOne {
  ...
};

class AddOneAwaitable {
 public:
  AddOneAwaitable(int x) : _x(x) {}
  bool await_ready() const { return false; }
  void await_suspend(std::coroutine_handle<> handle) {
    AddOne add_one(_x, [=, this](int result) {
      // 保存结果，用在 await_resume 中
      _result = result;
      // 在回调函数中，我们恢复当前协程
      handle.resume();
    });
  }
  int await_resume() const {
    // 返回结果
    return _result;
  }

 private:
  int _x;
  int _result;
};
```

## 协程对象与承诺对象

除了 `awaitable` 对象以外，我们还需要定义一个**协程对象**（我们用 `task` 作为它的类名）。

一个最简单的协程对象可以什么都没有，只包含一个 `promise_type` 类型：

```cpp
class task {
 public:
  class promise_type { ... };
};
```

每一个协程对象都与一个**承诺（Promise）对象**关联。简单来说，承诺对象由协程操作，协程通过承诺对象提交结果或执行异常处理；协程对象则由调用者操作。

承诺对象需要至少实现下面的五个函数：

* **`get_return_object()`**：获得协程对象。我们可以在此处通过 `std::coroutine_handle::from_promise()` 获取关联的协程句柄，然后将协程句柄传递给协程对象。
* **`initial_suspend()`**：该函数返回一个 `awaitable` 对象，协程在初始化时将会 `co_await` 它。对于惰性启动的协程，可以返回 `std::suspend_always`，对于立即启动的协程，可以返回 `std::suspend_never`。这两个类型的实现非常简单，它们的 `await_suspend()` 和 `await_resume()` 都是空的，而 `await_ready()` 分别返回 `false` 和 `true`。我们知道，当 `await_ready()` 返回 `true` 的时候，`co_await` 不会暂停协程，因此通过返回 `std::suspend_always` 或 `std::suspend_never` 可以决定协程是否在初始化时暂停。
* **`final_suspend()`**：和上面类似，只不过是在协程结束时 `co_await` 它返回的 `awaitable` 对象。需要注意的是，如果该函数返回 `std::suspend_never`，即协程不在结束时暂停，那么协程完成 `final_suspend()` 之后会释放相关资源；而如果返回 `std::suspend_always`，即协程在结束时暂停，那么调用者仍然可以从协程对象中获取相关的资源。注意，不管暂不暂停，之后再恢复该协程都是未定义行为。另外，**该函数必须是 `noexcept` 的**，也就是说它不能引发异常。
* **`unhandled_exception()`**：当协程因异常而结束时，调用该函数处理异常。
* **`return_void()`**/**`return_value()`**：该函数与 `co_return` 有关，我们暂且按下不表。

用上面的信息，我们来实现协程和承诺类型：

```cpp
class task {
 public:
  class promise_type {
   public:
    // 获得协程对象
    task get_return_object() {
      // 把协程句柄交给协程对象的构造函数
      return {std::coroutine_handle<promise_type>::from_promise(*this)};
    }

    // 没有必要在初始化时暂停，所以返回 std::suspend_never
    std::suspend_never initial_suspend() { return {}; }

    // 我们还需要获取协程的相关信息，因此让协程在结束时暂停，见 task::done 函数
    std::suspend_always final_suspend() noexcept { return {}; }

    // 异常处理，我们目前不关心它，留空
    void unhandled_exception() {}

    // 和 co_return 有关，暂且按下不表
    void return_void() {}
  };

  // 保存一下协程句柄
  task(std::coroutine_handle<promise_type> handle) : _handle(handle) {}

  // 协程对象的调用者可以通过该函数获取协程是否执行完成
  bool done() {
    // 需要注意，协程句柄的 done() 函数要求协程在结束时暂停，
    // 也就是说，承诺类型的 final_suspend() 要返回 std::suspend_always
    return _handle.done();
  }

 private:
  std::coroutine_handle<promise_type> _handle;
};
```

我们写一个协程函数，对应在线程模型中传递给线程执行的函数：

```cpp
// 返回值 task 将会自动生成，可以充当返回值的类型 T 一定要有 T::promise_type
task add_one_coroutine(int x, std::function<void(int)> result_handle) {
  // co_await 一个 AddOneAwaitable，
  // 协程会在此处暂停，然后将控制交还给 add_one_coroutine 的调用者。
  // 整个表达式的返回值是 AddOneAwaitable::await_resume() 的返回值。
  int result = co_await AddOneAwaitable(x);
  // 协程句柄的 resume() 被 AddOneAwaitable 注册给 AddOne 的回调函数调用，协程函数恢复执行，可以处理数据了
  result_handle(result);
}
```

然后，我们将线程模型替换为协程模型：

```cpp
int main(int argc, char* argv[]) {
  int value = 1;
  int result1 = 0;
  int result2 = 0;
  std::function<void(int)> result_handle1 = [&](int _result) mutable {
    result1 = _result + 1;
  };
  std::function<void(int)> result_handle2 = [&](int _result) mutable {
    result2 = _result * _result;
  };

  // 把线程模型改成协程模型：
  task task1 = add_one_coroutine(value, result_handle1);
  task task2 = add_one_coroutine(value, result_handle2);

  // 由于上面两个任务都会阻塞暂停协程，所以 main 函数还可以继续做其他事情
  std::cout << "hello world" << std::endl;

  // main 需要做的事情已经做完了，等待协程完成
  while (true) {
    if (task1.done() && task2.done()) {
      // 处理结果
      std::cout << result1 << result2 << std::endl;
      break;
    }
    else
      std::this_thread::sleep_for(std::chrono::milliseconds(10));
  }
  return 0;
}
```

完整的代码如下：

```cpp
#include <chrono>
#include <coroutine>
#include <functional>
#include <iostream>
#include <thread>

class AddOne {
 public:
  // 注意构造函数本身不阻塞主线程
  AddOne(int x, std::function<void(int)> result_ready_cb)
      : _thread{[=, result_ready_cb = std::move(result_ready_cb)]() mutable {
          // 假装阻塞了 5s 才得到结果
          std::this_thread::sleep_for(std::chrono::seconds(5));
          result_ready_cb(x + 1);
        }} {}

  ~AddOne() {
    // 如果 AddOne 析构时线程还没有完成，我们 detach 这个线程
    if (_thread.joinable()) _thread.detach();
  }

  // 但是 wait_for_result 有可能阻塞主线程
  void wait_for_result() { _thread.join(); }

 private:
  std::thread _thread;
};

class AddOneAwaitable {
 public:
  AddOneAwaitable(int x) : _x(x) {}
  bool await_ready() const { return false; }
  void await_suspend(std::coroutine_handle<> handle) {
    AddOne add_one(_x, [=, this](int result) {
      // 保存结果，用在 await_resume 中
      _result = result;
      // 在回调函数中，我们恢复当前协程
      handle.resume();
    });
  }
  int await_resume() const {
    // 返回结果
    return _result;
  }

 private:
  int _x;
  int _result;
};

class task {
 public:
  class promise_type {
   public:
    // 获得协程对象
    task get_return_object() {
      // 把协程句柄交给协程对象的构造函数
      return {std::coroutine_handle<promise_type>::from_promise(*this)};
    }

    // 没有必要在初始化时暂停，所以返回 std::suspend_never
    std::suspend_never initial_suspend() { return {}; }

    // 我们还需要获取协程的相关信息，因此让协程在结束时暂停，见 task::done 函数
    std::suspend_always final_suspend() noexcept { return {}; }

    // 异常处理，我们目前不关心它，留空
    void unhandled_exception() {}

    // 和 co_return 有关，暂且按下不表
    void return_void() {}
  };

  // 保存一下协程句柄
  task(std::coroutine_handle<promise_type> handle) : _handle(handle) {}

  // 协程对象的调用者可以通过该函数获取协程是否执行完成
  bool done() {
    // 需要注意，协程句柄的 done() 函数要求协程在结束时暂停，
    // 也就是说，承诺类型的 final_suspend() 要返回 std::suspend_always
    return _handle.done();
  }

 private:
  std::coroutine_handle<promise_type> _handle;
};

// 返回值 task 将会自动生成，可以充当返回值的类型 T 一定要有 T::promise_type
task add_one_coroutine(int x, std::function<void(int)> result_handle) {
  // co_await 一个 AddOneAwaitable，
  // 协程会在此处暂停，然后将控制交还给 add_one_coroutine 的调用者。
  // 整个表达式的返回值是 AddOneAwaitable::await_resume() 的返回值。
  int result = co_await AddOneAwaitable(x);
  // 协程句柄的 resume() 被 AddOneAwaitable 注册给 AddOne 的回调函数调用，协程函数恢复执行，可以处理数据了
  result_handle(result);
}

int main(int argc, char* argv[]) {
  int value = 1;
  int result1 = 0;
  int result2 = 0;
  std::function<void(int)> result_handle1 = [&](int _result) mutable {
    result1 = _result + 1;
  };
  std::function<void(int)> result_handle2 = [&](int _result) mutable {
    result2 = _result * _result;
  };

  // 把线程模型改成协程模型：
  task task1 = add_one_coroutine(value, result_handle1);
  task task2 = add_one_coroutine(value, result_handle2);

  // 由于上面两个任务都会阻塞暂停协程，所以 main 函数还可以继续做其他事情
  std::cout << "hello world" << std::endl;

  // main 需要做的事情已经做完了，等待协程完成
  while (true) {
    if (task1.done() && task2.done()) {
      // 处理结果
      std::cout << result1 << result2 << std::endl;
      break;
    }
    else
      std::this_thread::sleep_for(std::chrono::milliseconds(10));
  }
  return 0;
}
```

我们来分析一下协程的执行顺序：

1. 进入 `main` 函数
2. `main` 函数调用 `add_one_coroutine()` 函数
3. 创建一个 `task::promise_type` 类型的承诺对象，假设为 `promise1`
4. 调用 `promise1.get_return_object()`，构造 `task` 类型的协程对象，获得 `task1`
5. `promise1.initial_suspend()` 被调用，由于返回 `std::suspend_never`，因此协程继续执行
6. 进入函数 `add_one_coroutine()` 内部，执行 `co_await AddOneAwaitable(x)`
7. 构造 `AddOneAwaitable` 对象，假设为 `awaitable1`
8. 调用 `awaitable1.await_ready()`，由于我们返回 `false`，因此协程立即暂停
9. 调用 `awaitable1.await_suspend()`，之后协程将控制交还给 main 函数
10. `main` 函数再次调用 `add_one_coroutine()` 函数
11. 同 3~9，协程对象 `task2` 也做一样的事情，最后将控制交还给 main 函数
12. main 函数继续做自己的事情
13. `AddOne` 阻塞结束，调用 `task1` 的协程句柄的 `resume()` 函数，`task1` 协程恢复。需要注意的是，**这里 `task1` 会被转移到 `AddOne` 创建的线程中恢复执行**，这一点我们后面再说
14. 调用 `awaitable1.await_resume()`，并将该函数的返回值作为 `co_await AddOneAwaitable(x)` 的返回值
15. 继续执行 `add_one_coroutine` 剩下的内容
16. 函数执行结束，调用 `promise1.return_void()`，该函数与 `co_return` 有关，我们后面再讲
17. 调用 `promise1.final_suspend()`，协程执行结束
18. 等待 `task2` 的阻塞结束，然后类似 12~16 步骤完成 `task2` 协程

有几点是上述例子中没有体现出来的，需要注意：

1. 同一个协程内可以出现多个 `co_await` 表达式
2. 同一个 `awaitable` 对象可以被多次 `co_await`
3. 协程对象 `task` 也可以实现 `await_ready()`, `await_suspend()` 和 `await_resume()` 然后被其他协程 `co_await`。

另一方面，我们注意到协程并不是在主线程恢复的，而是在 `AddOne` 创建的线程中恢复执行的。这是因为哪个线程调用协程句柄的 `resume()`，哪个线程就会接手该协程的执行。

**我们当然也可以在主线程中调用 `resume()`，让所有协程都在同一个线程中进行**。例如，我们写一个简单的计数器协程：

```cpp
#include <coroutine>
#include <iostream>

class task {
 public:
  class promise_type {
   public:
    task get_return_object() {
      return {std::coroutine_handle<promise_type>::from_promise(*this)};
    }
    std::suspend_never initial_suspend() { return {}; }
    // 这个例子中不需要在协程结束后保留相关资源，因此直接返回 std::suspend_never
    std::suspend_never final_suspend() noexcept { return {}; }
    void unhandled_exception() {}
    void return_void() {}
  };

  task(std::coroutine_handle<promise_type> handle) : _handle(handle) {}

  // 通过调用协程对象的 resume() 主动恢复协程
  void resume() { _handle.resume(); }

 private:
  std::coroutine_handle<promise_type> _handle;
};

// 一个简单的计数器，功能就是在每次恢复时计数一次
task counter() {
  std::suspend_always awaitable;
  for (size_t count = 0;; ++count) {
    // 同一个 awaitable 可以多次 co_await
    // 每次 co_await，都会将控制交还给调用者，也就是 main 函数
    co_await awaitable;
    std::cout << "current count: " << count << std::endl;
  }
}

int main(int argc, char* argv[]) {
  task counter_task = counter();
  for (int i = 0; i < 3; ++i) {
    std::cout << "resume counter task" << std::endl;
    // 主动调用 resume 以恢复协程
    counter_task.resume();
  }
  return 0;
}
```
{: run="cpp" }

## 协程计时器

根据上面的信息，我们实现协程计时器有两个思路：

1. 在 `awaitable` 的 `await_suspend` 函数中开辟新的线程，在新线程中计时，并注册唤醒协程的回调函数。该方案的优点是精确，能确保协程在指定时间被唤醒。
2. 不开辟新线程，而是在主线程中主动调用 `handle.resume()` 来推动计时器。该方案的优点是不需要开辟新的线程，但是缺点是只有在主线程主动调用 `handle.resume()` 时，才能知道计时器究竟有没有计时结束，因此这种计时器的计时不够精准，其精确程度取决于主线程调用 `handle.resume()` 的频率。

第一种方案和 `AddOneAwaitable` 的实现大同小异，我们就不再写一遍了。我们主要来说说第二种计时器。

首先 `task` 类型没啥特殊的地方，照着以前的用就行：

```cpp
#include <coroutine>

class task {
 public:
  class promise_type {
   public:
    task get_return_object() {
      return {std::coroutine_handle<promise_type>::from_promise(*this)};
    }
    std::suspend_never initial_suspend() { return {}; }
    std::suspend_always final_suspend() noexcept { return {}; }
    void unhandled_exception() {}
    void return_void() {}
  };

  task(std::coroutine_handle<promise_type> handle) : _handle(handle) {}
  bool done() { return _handle.done(); }
  void resume() { _handle.resume(); }

 private:
  std::coroutine_handle<promise_type> _handle;
};
```

然后是 `awaitable` 类型，由于我们不打算开辟新线程来计时，因此我们不能使用形如 `std::this_thread::wait_for()` 的方法来计时，否则会直接阻塞主线程。

那么我们要做的其实就很简单了，只在 `awaitable` 里做超时判断，返回 `true`/`false` 给外部：

```cpp
#include <chrono>
#include <coroutine>

class SleepFor {
 public:
  template <typename DurationT>
  SleepFor(DurationT duration) {
    _time_wait_for = std::chrono::steady_clock::now() + duration;
  }
  bool await_ready() const { return _timeout(); }
  bool await_suspend(std::coroutine_handle<>) const { return !_timeout(); }
  // 这里是关键，将当前是否超时返回给 co_await 的调用者
  bool await_resume() const { return _timeout(); }

 private:
  inline bool _timeout() const {
    return std::chrono::steady_clock::now() >= _time_wait_for;
  }
  std::chrono::time_point<std::chrono::steady_clock> _time_wait_for;
};
```

那么，我们可以这样来实现协程计时器函数：

```cpp
template <typename DurationT>
task sleep_for(DurationT duration, std::function<void(void)> callback) {
  SleepFor sleep_awaitable{duration};
  while (!co_await sleep_awaitable)
    ;
  callback();
}
```

由于是单线程，我们在 `main` 函数中需要不断地推动计时器，因此最好的实现方式就是事件循环：

```cpp
int main(int argc, char* argv[]) {
  task t =
      sleep_for(5s, []() { std::cout << "5 seconds passed" << std::endl; });

  for (;;) {
    // 做主线程该做的事情。在这里，我们使用 sleep 100ms 的方法来代表主线程做了
    // 100ms 的其他事情
    std::this_thread::sleep_for(100ms);

    // 我们在 resume() 的时候才知道计时器有没有跑完，因此这个计时器是不准确的，
    // 其精确程度取决于每一轮事件循环中，主线程需要做多久其他的事情。
    // 但这种方法并不是没有优点，考虑数据竞争问题，这种方案可以有效地避免主线程
    // 在处理数据的过程中，协程被唤醒并且导致数据竞争。
    if (!t.done()) {
      t.resume();
    }
  }
}
```

我们整合一下代码，可以把 `SleepFor` 藏在 `sleep_for` 函数的内部以隐藏其实现细节：

```cpp
#include <chrono>
#include <coroutine>
#include <functional>
#include <iostream>
#include <thread>

using namespace std::chrono_literals;

class task {
 public:
  class promise_type {
   public:
    task get_return_object() {
      return {std::coroutine_handle<promise_type>::from_promise(*this)};
    }
    std::suspend_never initial_suspend() { return {}; }
    std::suspend_always final_suspend() noexcept { return {}; }
    void unhandled_exception() {}
    void return_void() {}
  };

  task(std::coroutine_handle<promise_type> handle) : _handle(handle) {}
  bool done() { return _handle.done(); }
  void resume() { _handle.resume(); }

 private:
  std::coroutine_handle<promise_type> _handle;
};

template <typename DurationT>
task sleep_for(DurationT duration, std::function<void(void)> callback) {
  class SleepFor {
   public:
    SleepFor(DurationT duration) {
      _time_wait_for = std::chrono::steady_clock::now() + duration;
    }
    bool await_ready() const { return _timeout(); }
    bool await_suspend(std::coroutine_handle<>) const { return !_timeout(); }
    bool await_resume() const { return _timeout(); }

   private:
    inline bool _timeout() const {
      return std::chrono::steady_clock::now() >= _time_wait_for;
    }
    std::chrono::time_point<std::chrono::steady_clock> _time_wait_for;
  };

  SleepFor sleep_awaitable{duration};
  while (!co_await sleep_awaitable)
    ;
  callback();
}

int main(int argc, char* argv[]) {
  task t =
      sleep_for(5s, []() { std::cout << "5 seconds passed" << std::endl; });

  for (;;) {
    // 做主线程该做的事情。在这里，我们使用 sleep 100ms 的方法来代表主线程做了
    // 100ms 的其他事情
    std::this_thread::sleep_for(100ms);

    if (!t.done()) {
      t.resume();
    }
  }
}
```

## `co_return`

我们知道，协程函数的返回值是一个协程对象，因此我们没法简单地通过 `return value;` 将返回值从协程函数内传递给协程调用者。

因此，C++20 给了我们 `co_return` 关键字，类似普通函数中的 `return`，它可以结束协程，并将值从协程函数内传递给协程调用者。

`co_return` 表达式的核心就是我们之前刻意忽略掉的承诺类型的 `return_void()` 和 `return_value()` 函数。当 `co_return;` 不携带返回值调用时，`promise.return_void()` 被调用，类似普通函数，在协程函数末尾会隐含一个 `co_return;`；当 `co_return expr` 被调用时，`promise.return_value(expr)` 被调用。

举个简单的例子：

```cpp
#include <coroutine>
#include <iostream>

class task {
 public:
  class promise_type {
   public:
    task get_return_object() {
      return {std::coroutine_handle<promise_type>::from_promise(*this)};
    }
    std::suspend_never initial_suspend() { return {}; }
    std::suspend_always final_suspend() noexcept { return {}; }
    void unhandled_exception() {}
    void return_value(int value) {
      // 把返回值存起来
      _value = value;
    }
    int _value;
  };

  task(std::coroutine_handle<promise_type> handle) : _handle(handle) {}

  bool done() { return _handle.done(); }
  void resume() { _handle.resume(); }

  // 包装一个函数供 main 函数获取返回值
  int get_value() { return _handle.promise()._value; }

 private:
  std::coroutine_handle<promise_type> _handle;
};

// 一个简单协程，仅仅只是返回 1
task just_get_1() { co_return 1; }

int main(int argc, char* argv[]) {
  task get_1_task = just_get_1();
  // 确保协程已经执行完
  if (!get_1_task.done()) get_1_task.resume();
  // 获取协程的返回值
  std::cout << "value is " << get_1_task.get_value() << std::endl;
  return 0;
}
```
{: run="cpp" }

## `co_yield`

**`co_yield expr` 实质是 `co_await promise.yield_value(expr)` 的语法糖**，很容易看出，想要使用 `co_yield`，我们需要给承诺类型实现一个 `yield_value()` 函数，并且该函数返回一个 `awaitable` 对象。这个语法通常用来实现**惰性生成器**，例如，我们做一个斐波那契数列的生成器：

```cpp
#include <coroutine>
#include <iostream>

class task {
 public:
  class promise_type {
   public:
    task get_return_object() {
      return {std::coroutine_handle<promise_type>::from_promise(*this)};
    }
    // 由于是惰性生成器，我们在协程初始化时就暂停，所以返回 std::suspend_always
    std::suspend_always initial_suspend() { return {}; }
    std::suspend_always final_suspend() noexcept { return {}; }
    void unhandled_exception() {}
    void return_void() {}
    // 每次生成值后都暂停，所以我们返回 std::suspend_always
    std::suspend_always yield_value(size_t value) {
      // 类似 co_return，我们把每次 yield 的值都保存起来
      _value = value;
      return {};
    }
    size_t _value;
  };

  task(std::coroutine_handle<promise_type> handle) : _handle(handle) {}

  // 我们重载一个 operator()，当然重载一个其他函数名也是一样的
  size_t operator()() {
    // 恢复协程的执行
    _handle.resume();
    // 将本次 yield 的值返回给调用者
    return _handle.promise()._value;
  }

 private:
  std::coroutine_handle<promise_type> _handle;
};

task fibonacci() {
  // yield 斐波那契数列的第一项给调用者
  co_yield 1;
  // yield 斐波那契数列的第二项给调用者
  co_yield 1;

  size_t n1 = 1, n2 = 1;
  while (true) {
    // 计算斐波那契数列
    size_t value = n1 + n2;
    // yield 给调用者
    co_yield value;
    // 为下一次计算做准备
    n1 = n2;
    n2 = value;
  }
}

int main(int argc, char* argv[]) {
  task fib = fibonacci();
  for (int i = 0; i < 10; ++i)
    // 调用一次 task::operator()，恢复协程的运行并获得一个 yield 出来的值
    std::cout << "fibonacci[" << i << "] is " << fib() << std::endl;
  return 0;
}
```
{: run="cpp" }

## 让 `task` 也可 `co_await`

基于上面的 `AddOneAwaitable` 的例子，我们试着将 `task` 也改造成 `awaitable` 类型。

不过我们先简化一下上面的例子，不再用回调函数了，而是用 `co_return` 返回结果：

```cpp
task add_one_coroutine(int x) {
  int result = co_await AddOneAwaitable(x);
  co_return result;
}
```

我们的目标是，让 `task` 也可以被 `co_await`，这意味着我们可以写：

```cpp
task add_one_coroutine(int x) {
  int result = co_await AddOneAwaitable(x);
  co_return result;
}

task add_two_coroutine(int x) {
  int result = co_await add_one_coroutine(x);
  result = co_await add_one_coroutine(result);
  co_return result;
}

task add_three_coroutine(int x) {
  int result = co_await add_two_coroutine(x);
  result = co_await add_one_coroutine(result);
  co_return result;
}
```

首先我们先把 `task::promise_type` 的 `return_void()` 换成 `return_value()` 以满足 `co_return` 的需求：

```cpp
class task {
 public:
  class promise_type {
   public:
    ...
    void return_value(int value) { _value = value; }
    int _value;
  };
  ...
  int get_value() const { return _handle.promise()._value; }
};
```

然后我们为 `task` 实现 `awaitable` 类型所需的三个函数：

```cpp
class task {
 public:
  class promise_type {
   public:
    ...
    std::suspend_always final_suspend() noexcept {
      // 这里是关键，我们需要在协程函数结束之后唤醒父协程
      if (_parent_handle) _parent_handle.resume();
      return {};
    }
    // 父协程的句柄
    std::coroutine_handle<promise_type> _parent_handle;
  };
  ...
  bool await_ready() const { return false; }
  void await_suspend(std::coroutine_handle<promise_type> handle) {
    // 把父协程的句柄传递给 promise，这样在当前协程执行结束之后可以调用它的
    // resume() 来恢复父协程
    _handle.promise()._parent_handle = handle;
  }
  int await_resume() const { return get_value(); }

 private:
  std::coroutine_handle<promise_type> _handle;
};
```

完整代码如下，该例子在启动 15 秒后输出 4：

```cpp
#include <chrono>
#include <coroutine>
#include <functional>
#include <iostream>
#include <thread>

class AddOne {
 public:
  AddOne(int x, std::function<void(int)> result_ready_cb)
      : _thread{[=, result_ready_cb = std::move(result_ready_cb)]() mutable {
          std::this_thread::sleep_for(std::chrono::seconds(5));
          result_ready_cb(x + 1);
        }} {}

  ~AddOne() {
    if (_thread.joinable()) _thread.detach();
  }

  void wait_for_result() { _thread.join(); }

 private:
  std::thread _thread;
};

class AddOneAwaitable {
 public:
  AddOneAwaitable(int x) : _x(x) {}
  bool await_ready() const { return false; }
  void await_suspend(std::coroutine_handle<> handle) {
    AddOne add_one(_x, [=, this](int result) {
      _result = result;
      handle.resume();
    });
  }
  int await_resume() const { return _result; }

 private:
  int _x;
  int _result;
};

class task {
 public:
  class promise_type {
   public:
    task get_return_object() {
      return {std::coroutine_handle<promise_type>::from_promise(*this)};
    }
    std::suspend_never initial_suspend() { return {}; }
    std::suspend_always final_suspend() noexcept {
      // 这里是关键，我们需要在协程函数结束之后唤醒父协程
      if (_parent_handle) _parent_handle.resume();
      return {};
    }
    void unhandled_exception() {}
    void return_value(int value) { _value = value; }
    int _value;
    // 父协程的句柄
    std::coroutine_handle<promise_type> _parent_handle;
  };

  task(std::coroutine_handle<promise_type> handle) : _handle(handle) {}
  bool done() { return _handle.done(); }
  int get_value() const { return _handle.promise()._value; }
  bool await_ready() const { return false; }
  void await_suspend(std::coroutine_handle<promise_type> handle) {
    // 把父协程的句柄传递给 promise，这样在当前协程执行结束之后可以调用它的
    // resume() 来恢复父协程
    _handle.promise()._parent_handle = handle;
  }
  int await_resume() const { return get_value(); }

 private:
  std::coroutine_handle<promise_type> _handle;
};

task add_one_coroutine(int x) {
  int result = co_await AddOneAwaitable(x);
  co_return result;
}

task add_two_coroutine(int x) {
  int result = co_await add_one_coroutine(x);
  result = co_await add_one_coroutine(result);
  co_return result;
}

task add_three_coroutine(int x) {
  int result = co_await add_two_coroutine(x);
  result = co_await add_one_coroutine(result);
  co_return result;
}

int main(int argc, char* argv[]) {
  int value = 1;
  task task = add_three_coroutine(value);

  // 由于上面两个任务都会阻塞暂停协程，所以 main 函数还可以继续做其他事情
  std::cout << "hello world" << std::endl;

  // main 需要做的事情已经做完了，等待协程完成
  while (true) {
    if (task.done()) {
      // 处理结果
      std::cout << task.get_value() << std::endl;
      break;
    } else
      std::this_thread::sleep_for(std::chrono::milliseconds(10));
  }
  return 0;
}
```

## 展望 C++23

从上面的例子不难看出，在 C++ 中想要写一份协程代码非常的麻烦，需要定义非常多的东西。而 C++23 就在着手解决这个问题，标准库将会提供一些通用的协程类型，让我们可以更简单地上手操作协程。例如，上述斐波那契数列生成器的例子，在 C++23 中可以这样写：

```cpp
#include <generator>
#include <ranges>
#include <iostream>

std::generator<size_t> fibonacci() {
  co_yield 1;
  co_yield 1;

  size_t n1 = 1, n2 = 1;
  while (true) {
    size_t value = n1 + n2;
    co_yield value;
    n1 = n2;
    n2 = value;
  }
}

int main(int argc, char* argv[]) {
  for (auto const [i, item] : fibonacci() | std::views::enumerate | std::views::take(10))
    std::cout << "fibonacci[" << i << "] is " << item << std::endl;
  return 0;
}
```

## 参考

[c++20协程入门 - 觅新笑话中](https://zhuanlan.zhihu.com/p/59178345)

[协程 - cppreference](https://en.cppreference.com/w/cpp/language/coroutines.html)
