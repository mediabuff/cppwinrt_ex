# Overview of `cppwinrt_ex` Library

This library is a small (and I hope growing) collection of utilities that complement the functionality of [`cppwinrt`](https://github.com/Microsoft/cppwinrt) library.

To start using this library, clone or download it and then `#include` the single header "cppwinrt_ex/core.h". Library has no external dependencies, except `cppwinrt` itself.

## Code Structure

This repository consists of the following subdirectories:

* **include**
  * Contains `core.h` header .
* **sample**
  * Contains an example project that illustrates the library usage.

## Compiler Support

The library has been tested on Microsoft Visual Studio 2017 Version 15.1 (26403.7).

## Documentation

`cppwinrt_ex` library has all its classes and functions defined in `winrt_ex` namespace.

### `async_action` and `async_operation<T>` Classes

`cppwinrt` library defines two classes: `winrt::Windows::Foundation::IAsyncAction` and `winrt::Windows::Foundation::IAsyncOperation<T>` as promise classes to be used by C++ coroutines.

The only problem with these classes is that they force continuation on original thread's context, which sometimes is not desirable or even at all possible.

`cppwinrt_ex` adds simplified versions of those classes, `async_action` and `async_operation<T>` that do not force return of execution control to original thread's context. This also slightly improves performance.

**Note that coroutines directly called from UI thread should continue to use original versions.**

### `start` and `start_async` Functions

`cppwinrt` provides a number of convenient utility classes to initiate asynchronous waits and I/O, among other things. The only problem with those classes is that the operation does not start until the caller begins _awaiting_ its result. Consider the following:

```
IAsyncAction coroutine1()
{
    // ...
    co_await 3s;
    // ...
}
```

In this code snippet, `co_await 3s;` is translated to `co_await resume_after{3s};`. `resume_after` is an _awaitable_ that starts a timer on thread pool and schedules a continuation. The problem is that you cannot *start* a timer and continue your work.

The same problem exists with `resumeable_io` class:

```
resumable_io io{handle};
/// ...
IAsyncAction coroutine2()
{
    co_await io.start([]
    {
       check(::ReadFile(handle,...));
    });
    // ...
}
```

And again, you cannot start an I/O and do other stuff before you _await_ for operation result.

`winrt_ex` provides simple wrapper function that starts an asynchronous operation for you:

```
resumable_io io{handle};
/// ...
IAsyncAction coroutine3()
{
    auto running_io_operation = winrt_ex::start(io.start([]
    {
       check(::ReadFile(handle,...));
    });
    // do other stuff
    // here we finally need to wait for operation to complete:
    auto result = co_await running_io_operation;
    // ...
}
```

`winrt_ex::start` supports awaitables that produce no result or awaitables that produce some result. Therefore, it behaves either as it has `IAsyncAction` return type or `IAsyncOperation<T>` return type.

Lirary also has `winrt_ex::start_async` version that has `async_action` or `async_operation<T>` as its return type.

### `async_timer` Class

This is an awaitable cancellable timer. Its usage is very simple:

```
winrt_ex::async_timer timer;
// ...
IAsyncAction coroutine4()
{
    try
    {
        co_await timer.wait(20min);
    } catch(hresult_cancelled)
    {
        // the wait has been cancelled
    }
}
// ...
void cancel_wait()
{
    timer.cancel();
}
```

**Note that current version runs the timer continuation inside the call to the `cancel` method. This might be changed in the future.**

### `resumable_io_timeout` Class

This is a version of `cppwinrt`'s `resumable_io` class that supports timeout for I/O operations. Its `start` method requires an additional parameter that specifies the I/O operation's timeout. If operation does not finish within a given time, it is cancelled and `hresult_cancelled` exception is propagated to the continuation:

```
winrt_ex::resumable_io_timeout io{ handle_to_serial_port };
// ...
IAsyncAction coroutine5()
{
    try
    {
        auto bytes_received = co_await io.start([]
        {
            check(::ReadFile(handle_to_serial_port, ... ));
        }, 10s);
        // Operation succeeded, continue processing
    } catch(hresult_cancelled)
    {
        // operation timeout, data not ready
    } catch(hresult_error)
    {
        // other I/O errors
    }
}
```

### `when_all` Function

`when_all` function accepts any number of awaitables and produces an awaitable that is completed only when all input tasks are completed. If at least one of the tasks throws, the first thrown exception is rethrown by `when_all`.

All input parameters must be `IAsyncAction`, `async_action`, `IAsyncOperation<T>` or `async_operation<T>`.

If all input tasks produce no result, `when_all` also produces no result, otherwise, it produces an `std::tuple<>` of all input parameter types.

```
winrt_ex::async_action void_timer(TimeSpan duration)
{
	co_await duration;
}

winrt_ex::async_operation<bool> bool_timer(TimeSpan duration)
{
	co_await duration;
	co_return true;
}

winrt_ex::async_operation<int> int_timer(TimeSpan duration)
{
	co_await duration;
	co_return 10;
}

IAsyncAction coroutine6()
{
    // The following operation will complete in 30 seconds and produce void
    co_await winrt_ex::when_all(void_timer(10s), void_timer(20s), void_timer(30s));

    // The following operation will complete in 30 seconds and produce std::tuple<bool, bool, int>
    std::tuple<bool, bool, int> result = co_await winrt_ex::when_all(bool_timer(10s), bool_timer(20s), int_timer(30s));
}
```

### `when_any` Function

`when_any` function accepts any number of awaitables and produce an awaitable that is completed when at least one of the input tasks is completed. If the first completed task throws, the thrown exception is rethrown by `when_any`.

All input parameters must be `IAsyncAction`, `async_action`, `IAsyncOperation<T>` or `async_operation<T>` and *must all be of the same type*.

**`when_any` does not cancel any non-completed tasks.** When other tasks complete, their results are silently discarded. `when_any` makes sure the control block does not get destroyed until all tasks complete.

If all input tasks produce no result, `when_any` produces the index to the first completed task. Otherwise, it produces `std::pair<T, size_t>`, where the first result is the result of completed task and second is an index of completed task:

```
IAsyncAction coroutine7()
{
    // The following operation will complete in 10 seconds and produce 0
    size_t index = co_await winrt_ex::when_any(void_timer(10s), void_timer(20s), void_timer(30s));

    // The following operation will complete in 10 seconds and produce std::pair<bool, size_t> { true, 0 }
    std::pair<bool, size_t> result = co_await winrt_ex::when_all(bool_timer(10s), bool_timer(20s), bool_timer(30s));
}
```