---
title: "c++20 coroutine"
date: 2022-09-11T14:55:59+08:00
---

# 协程

## 什么是协程

- 可以在指定位置暂停，并从暂停点恢复执行
- 可以有多个暂停点，可以多 次返回（generator）
- 协程的本地数据需要持久化

## 协程分类

- 控制传递机制: 对称/非对称（return to caller）
- 是否作为语言的第一类（first class）对象提供：语言特性和编译器支持有栈(stackfull) / 无栈(stackless): 是否有独立的运行时栈

## 协程 coroutine frame

协程的执行被分为 3 个阶段：

- **Suspend:**  暂停当前协程的执行，返回给调用者或者恢复者
- **Resume**：恢复当前协程的执行
- **Destroy**：销毁协程栈，保持的变量，释放栈帧和内存

由于协程可以被暂停，可以被恢复，协程的运行状态需要被保存下来。而栈会随着函数返回被销毁，故栈不能用来保存协程状态。

协程会在堆上保存自己的状态。
可以简单认为运行的协程有两部分组成：

- stack frame：普通的函数栈帧，会随着协作暂停而销毁，会随着协程恢复而创建。
- coroutine frame：堆上的协程栈，会保存跨暂停点的变量，在协程暂停时，会保存暂停点用于后续恢复，保存调用的协程返回值。

# c++20 coroutinue

- 3个语言关键字，co_await, co_yield, co_return
- 一些新的类型：
  - coroutine_handle
  - coroutine_traits<Ts…>
  - suspend_always
  - suspend_never
- 库编写者可以和协程交互以及定义行为的通用机制
- 更容易编写异步代码语言级设施

c++20 没有提供协程库，只提供了用于构造协程库的基础设施，要直接使用现有的协程基础类型编写安全的异步代码还是非常不方便的。

函数体中一定要有 co_return，co_await，co_yield 三个关键字中一个，才算是一个协程。即便返回了 包含内部类 promise_type 的对象，也不是一个协程。不会有 init suspend 和 final suspend 点。

## 协程接口

c++ 标准定义了两个接口：

- **Promise** interfac：用来控制协程的行为
  - 定义协程被调用时的行为
  - 定义协程返回时的行为
  - 定义`co_await` or `co_yield` 表达式的行为

- **Awaitable** interface：用来定义 co_await 表达式的语义
  - 是否暂停当前协程的执行
  - 执行指定的逻辑使得协程在合适的时机恢复
  - 如何获得协程的返回值

## co_await

co_await 是一个一元操作符，co_await awaitable。

awaitable 需要满足特定的条件：

- Normally Awaitable：实现了 operator co_await()， 返回对象实现了 await_ready, await_suspend 和 await_resume，返回的对象被称为 awaiter。

- Contextually Awaitable：co_await 表达式所在协程，有定义 await_transform 函数，用于将 someValue 转换为满足 Normally Awaitable 的对象。

# promise_type

只要在函数体内使用了 co_await, co_yield和co_return 其中之一的关键字，就会触发编译器把该函数编译为协程而不是普通函数。

```C++
{
  co_await promise.initial_suspend();
  try
  {
    <body-statements>
  }
  catch (...)
  {
    promise.unhandled_exception();
  }
FinalSuspend:
  co_await promise.final_suspend();
}
```

promise_type 定义并控制了协程本身的行为。可以自定义协程被调用/返回时，co_await 或 co_yield 表达式的行为。

```C++
class promise_type {
    task<T> get_return_object();
    void unhandled_exception();
    // 控制协程在执行协程体代码之前是应该挂起，还是立即执行
    auto initial_suspend();
    // 在返回到调用者/恢复者之前协程有一个机会来运行一些额外的逻辑
    // 此时协程体已退出，局部变量已析构
    // resume 一个在 final_suspend 点挂起的协程行为是未定义的
    auto final_suspend();
    // co_return 返回 void 时被调用
    void return_void();
    // co_return 返回 T 类型的值时被调用
    void return_value(T& value);
    // 用来生成 co_await 操作符需要的 Awaitable 对象
    template<typename Awaitable>
    Awaitable&& await_transform(Awaitable&& awaitable);
}
```

# coroutine_traits

前面说到，promise_type 用来控制协程的行为。当定义了一个协程时，编译器如何知道该使用哪一个 promise type 呢？

coroutine_traits 是标准库定义的一个 trait，可以根据定义的协程参数，返回类型，得到 promise type。详见 https://en.cppreference.com/w/cpp/coroutine/coroutine_traits。

标准库的 coroutine_traits 要求协程的返回类型中需要定义 promise_type 类型。

我们也可以特化 coroutine_traits，从而可以实现把已有类型作为协程的返回类型。 如 [std::future ](https://en.cppreference.com/w/cpp/coroutine/coroutine_traits)作为协程返回类型。

一般情况下，协程的库都会定义一个协程返回类型，如 cppcoro 定义的 [task](https://github.com/lewissbaker/cppcoro/blob/master/include/cppcoro/task.hpp)。

# awaitable & awaiter

什么是 awaiter?

满足 co_await 运算符必要的函数定义。

```C++
struct Awaiter {
    // 当前协程是否 ready
    // 在操作同步完成而不需要挂起的情况下，可以避免 <suspend-coroutine> 操作的成本
    bool await_ready() const noexcept;
    // 返回值成为 co_await 表达式的结果.
    // 也可以抛出异常，在这种情况下异常从 co_await 表达式中抛出.
    // 如果异常从 await_suspen 抛出，则协程会自动恢复，并且异常会从 co_await 表达式抛出而不调用 await_resume.
    void await_resume() noexcept;
    // void: 无条件地将执行转移回协程的调用者/恢复者
    // bool: 允许 awaiter 对象有条件地返回并立即恢复协程, 
    // false 时，可以立马恢复协程
    // true 时，同 void
    // coroutine_handle: 将 coroutine_handle
    (void | bool | coroutine_handle) await_suspend(coroutine_handle<promise_type> coroutine);
}
```

什么是 awaitable？

能够支持 co_await 运算符。

支持 co_await 运算符的类型，不一定是直接的 awaiter 类型，只要能通过某种方式转换到 awaiter 对象，该类型就是 awaitable。
转换方式：

1. 类型定义了 operator co_await，返回类型就是 awaiter。
2. 定义了到 awaiter 类型的转换。
3. 所在 promise_type 定义了 await_transform，返回的类型可以得到/转换到 awaiter。

转换伪代码：

```C++
template<typename P, typename T>
decltype(auto) get_awaitable(P& promise, T&& expr)
{
   if constexpr (has_any_await_transform_member_v<P>)
      return promise.await_transform(static_cast<T&&>(expr));
   else
      return static_cast<T&&>(expr);
}

template<typename Awaitable>
decltype(auto) get_awaiter(Awaitable&& awaitable)
{
   if constexpr (has_member_operator_co_await_v<Awaitable>)
      return static_cast<Awaitable&&>(awaitable).operator co_await();
   else if constexpr (has_non_member_operator_co_await_v<Awaitable&&>)
      return operator co_await(static_cast<Awaitable&&>(awaitable));
   else
      return static_cast<Awaitable&&>(awaitable);
}
```

# await 协程

一个协程可以 co_await 其他协程，这是两个不同的协程，它们的 coroutine_handle 的地址不同。

c++20 的协程是对称式的，不同于函数返回时，一定会回到调用者的调用栈，协程返回时，下一步该执行什么代码是由协程的实现决定的。当一个协程 ready 时，它可以唤醒等待它的协程，也可以唤醒其他和它没有关系的协程。

当协程被调用时，promise_type 的 initial_suspend 决定了协程是否可以立马被执行。
如果 initial_suspend 返回 true，则协程被创建了，并完成了协程 frame 的分配，后续需要通过 resume 或者 co_await 运算符来运行协程。

当协程运行后，遇到 co_await 时，awaiter 的 await_ready 决定了 awaiter 是否 ready。
如果没有 ready，会暂停当前协程，然后调用 awaiter 的 await_suspend。
如果 ready，则通过 awaiter 的 await_resume 获取被 co_await 协程（awaiter 所属协程）的返回值。

await_suspend 是 await 协程的关键。await_suspend 被调用时，当前协程(并非 awaiter 所属协程)已经被暂停了。根据 awaiter 的 await_suspend 函数返回值，可以控制当前协程被暂后，下一步将控制权移交给谁。

await_suspend 的入参 coroutine_handle 是当前调用栈所属协程：

- 非 FinalAwaiter 时，协程已暂停，为协程调用方的 coroutine_handle。
- FinalAwaiter 时，此时调用方为 FinalAwaiter 所在协程。

```C++
{
    auto&& value = <expr>;
    auto&& awaitable = get_awaitable(promise, static_cast<decltype(value)>(value));
    auto&& awaiter = get_awaiter(static_cast<decltype(awaitable)>(awaitable));
    
    if (!awaiter.await_ready()) {
        using handle_t = std::experimental::coroutine_handle<P>;
        using await_suspend_result_t =
            decltype(awaiter.await_suspend(handle_t::from_promise(p)));
        // 先暂停当前协程(co_await 调用所在协程)，后调用 await_suspend.
        //
        // 编译器生成一些代码来保存协程的当前状态并准备恢复.
        // 这包括存储 <resume-point> 的位置，以及将当前寄存器中的值保存到协程frame.
        <suspend-coroutine> 
        if constexpr (is_void<await_suspend_result_t>) {
            // 无条件暂停协程，返给给调用者，恢复者
            awaiter.await_suspend(handle_t::from_promise(p));
            <return-to-caller-or-resumer>
        } else if (awaiter.await_suspend(handle_t::from_promise(p))) {
            // bool 类型，只有为 true 时，暂停协程
            <return-to-caller-or-resumer> 
        } else {
            // 执行返回的协程
            <return-to-coroutine>
        }
        
        // 恢复点
        <resume-point>
    }
    // 获取操作的结果
    return awaiter.await_resume();
}
```

# 协程例子

调用关系：TopTask -> MiddleTask -> LeafTask

唤醒关系：LeafTask::awaiter::await_suspend ->  LeafTask::FinalAwaiter::await_suspend ->MiddleTask::resume -> MiddleTask::FinalAwaiter::await_suspend -> TopTask::resume

注意，唤醒之后，协程在另一个线程中运行

```C++
#include <experimental/coroutine>
#include <thread>
#include <iostream>

struct TopTask {
    std::experimental::coroutine_handle<> handle_;

    ~TopTask() {
        std::cout << "TopTask  destructed" << std::endl;
    }

    struct promise_type {
        TopTask get_return_object() noexcept {
            auto handle = std::experimental::coroutine_handle<promise_type>::from_promise(*this);
            std::cout << "TopTask handle address: " << handle.address() << std::endl;
            return TopTask{
                    .handle_ = handle
            };
        }

        std::experimental::suspend_always initial_suspend() const noexcept { return {}; }

        std::experimental::suspend_never final_suspend() const noexcept { return {}; }

        void unhandled_exception() noexcept {
        }

        void return_void() noexcept {
        }
    };
};

struct MiddleTask {
    struct FinalAwaiter;
    struct promise_type;

    std::experimental::coroutine_handle<promise_type> handle_;

    struct promise_type {
        std::experimental::coroutine_handle<> continuation_;

        MiddleTask get_return_object() noexcept {
            auto handle = std::experimental::coroutine_handle<promise_type>::from_promise(*this);
            std::cout << "MiddleTask handle address: " << handle.address() << std::endl;
            return MiddleTask{
                    .handle_ = handle
            };
        }

        std::experimental::suspend_never initial_suspend() const noexcept { return {}; }

        FinalAwaiter final_suspend() noexcept {
            auto h = std::experimental::coroutine_handle<promise_type>::from_promise(*this);
            return {.handle_ = h};
        }

        void unhandled_exception() noexcept {
        }

        void return_void() noexcept {
        }
    };


    struct FinalAwaiter {
        std::experimental::coroutine_handle<promise_type> handle_;

        bool await_ready() const noexcept {
            return false;
        }

        void await_suspend(std::experimental::coroutine_handle<> cont) noexcept {
            std::cout << "MiddleTask FinalAwaiter await_suspend: cont address= " << cont.address() << " " << ""
                      << handle_.address() << std::endl;
            std::cout << "MiddleTask FinalAwaiter await_suspend, resume continuation" << std::endl;
            handle_.promise().continuation_.resume();
        }

        void await_resume() noexcept {}
    };

    struct Awaiter {
        std::experimental::coroutine_handle<promise_type> handle_;

        bool await_ready() const noexcept {
            return false;
        }

        void await_suspend(std::experimental::coroutine_handle<> cont) {
            std::cout << "MiddleTask await_suspend: cont= " << cont.address() << " " << "handle= " << handle_.address()
                      << std::endl;
            std::cout << "MiddleTask await_suspend:, do nothing, return to caller" << std::endl;
            handle_.promise().continuation_ = cont;
        }

        void await_resume() {}
    };

    Awaiter operator co_await() {
        return Awaiter{.handle_ = handle_};
    }
};

struct LeafTask {
    struct FinalAwaiter;
    struct Awaiter;
    struct promise_type;
    std::experimental::coroutine_handle<promise_type> handle_;

    struct promise_type {
        std::experimental::coroutine_handle<> continue_;

        LeafTask get_return_object() noexcept {
            auto handle = std::experimental::coroutine_handle<promise_type>::from_promise(*this);
            std::cout << "LeafTask handle address: " << handle.address() << std::endl;
            return LeafTask{.handle_ = handle};
        }

        std::experimental::suspend_always initial_suspend() const noexcept {
            return {};
        }

        FinalAwaiter final_suspend() const noexcept {
            return FinalAwaiter{};
        }

        void unhandled_exception() noexcept {}

        void return_void() noexcept {}
    };

    struct FinalAwaiter {
        bool await_ready() const noexcept {
            return false;
        }

        // cont 是 FinalAwaiter 所在协程, 而不是
        std::experimental::coroutine_handle<>
        await_suspend(std::experimental::coroutine_handle<promise_type> cont) noexcept {
            std::cout << "LeafTask FinalAwaiter await_suspend: cont= " << cont.address() << std::endl;
            std::cout << "LeafTask FinalAwaiter await_suspend, resume caller" << std::endl;
            auto p = cont.promise();
            return p.continue_;
        }

        void await_resume() noexcept {}
    };

    struct Awaiter {
        // 所在协程
        std::experimental::coroutine_handle<promise_type> handle_;

        bool await_ready() const noexcept {
            return false;
        }

        // cont: 调用方协程. awaiter 所在协程已经被暂停
        void await_suspend(std::experimental::coroutine_handle<> cont) {
            handle_.promise().continue_ = cont;
            std::cout << "LeafTask await_suspend: con= " << cont.address() << " " << "handle " << handle_.address()
                      << std::endl;
            auto c = handle_;
            auto f = [c]() mutable {
                std::cout << "LeafTask await_suspend, waker thread resume coroutine after 10 s" << std::endl;
                std::cout << "LeafTask await_suspend, waker thread pid: " << std::this_thread::get_id() << std::endl;
                std::this_thread::sleep_for(std::chrono::seconds{5});
                c.resume();
            };
            std::thread(f).detach();
            std::cout << "LeafTask await_suspend, spawn waker thread, return to caller" << std::endl;
        }

        void await_resume() {}
    };

    Awaiter operator co_await() {
        return Awaiter{.handle_ = handle_};
    }
};

LeafTask bar() {
    std::cout << "befor bar" << std::endl;
    co_await std::experimental::suspend_never{};
    std::cout << "after bar, current pid: " << std::this_thread::get_id() << std::endl;
}

MiddleTask foo() {
    std::cout << "before foo" << std::endl;
    co_await bar();
    std::cout << "after foo" << std::endl;
}

TopTask boo() {
    std::cout << "before boo" << std::endl;
    co_await foo();
    std::cout << "after boo" << std::endl;
}

int main() {
    TopTask t = boo();
    t.handle_.resume();
    std::cout << "TopTask is done: " << t.handle_.done() << std::endl;

    std::this_thread::sleep_for(std::chrono::seconds(15));
    return 0;
}
TopTask handle address: 0x6000038d8210
before boo
MiddleTask handle address: 0x6000023d81c0
before foo
LeafTask handle address: 0x6000036d9120
LeafTask await_suspend: con= 0x6000023d81c0 handle 0x6000036d9120
LeafTask await_suspend, spawn waker thread, return to caller
MiddleTask await_suspend: cont= 0x6000038d8210 handle= 0x6000023d81c0
MiddleTask await_suspend:, do nothing, return to caller
TopTask is done: 0
LeafTask await_suspend, waker thread resume coroutine after 10 s
LeafTask await_suspend, waker thread pid: 0x16d563000
befor bar
after bar, current pid: 0x16d563000
LeafTask FinalAwaiter await_suspend: cont= 0x6000036d9120
LeafTask FinalAwaiter await_suspend, resume caller
after foo
MiddleTask FinalAwaiter await_suspend: cont address= 0x6000023d81c0 0x6000023d81c0
MiddleTask FinalAwaiter await_suspend, resume continuation
after boo
```

# 参考资料：

https://lewissbaker.github.io/2017/11/17/understanding-operator-co-await 系列文章