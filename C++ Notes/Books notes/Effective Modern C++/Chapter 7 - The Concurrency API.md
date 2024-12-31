## Item 35: Prefer task-based programming to thread-based. 
There are two basic choices to run a function asynchronously: 
+ Create `std::thread` and run function on it, thus employing a _thread-based_ approach
+ Pass function to `std::async`, a strategy known as _task-based_. In such calls, the function object is considered a _task_.

```c++
int doAsyncWork();

std::thread t(doAsyncWork);
auto fut = std::async(doSyncWork);
```

With the thread-based invocation, there's no straightforward way to get access to function return value. With the task-based approach, it's easy, because the future return from `std::async` offers the `get` function. The `get` function is even more important if `doAsyncWork` emits an exception, because `get` provides access to that, too. With the thread-based approach, if `doAsyncWork` throws, the program dies (via call to `std::terminate`).

In general, there are three meanings of "thread" in concurrent C++ software:
+ _Hardware threads_ are the threads that actually perform computation. Contemporary machine architectures offer one or more hardware threads per CPU core.
+ _Software threads_ (also known as _OS threads_ or _system threads_) are the threads that the operating system manages across all processes and schedule for execution on hardware threads. It's typically possible to create more software threads than hardware threads, because when a software thread is blocked (e.g., on I/O waiting for a mutex or condition variable), throughput can be improved by executing other, unblocked, threads.
+ _`std::threads`_ are objects in a C++ process that act as handles for underlying software threads. Some `std::thread` objects represent "null" handles, i.e., correspond to no software thread, because they're in a default-constructed state (hence have no function to execute), have been moved from (the moved-to `std::tread` then acts as the handle to the underlying software thread), have been `join`ed (the function they were to run has finished), or have been `detach`ed (the connection between them and their underlying software thread has been severed). 

Software threads are a limited resource. If you try to create more than the system can provide, a `std::system_error` exception is thrown. Thus us true even if the function to run can't throw.

Well-written software must somehow deal with this possibility. One approach is to run `doAsyncWork` on the current thread, but that could lead to unbalanced loads, and, if the current thread is a GUI thread, responsiveness issues. Another option is to wait for some existing software threads to complete and then try to create a new `std::thread` again, but it's possible that the existing threads are waiting for an action that `doAsyncWork` is supposed to perform (e.g., produce a result or notify condition variable).

Even without out of threads problem, there is possible _oversubscription_. That's when there are more ready-to-run (i.e., unblocked) software threads than hardware threads. When that happens, the thread scheduler (typically part of the OS) time slices the software threads on the hardware. When on thread's time-slice is finished and another's begins, a context switch is performed. Such context switches increase the overall thread management overhead of the system, and they can be particularly costly when the hardware thread on which a software thread is scheduled is on a different core than was the case for the software thread during its last time-slice. In that case, the CPU caches are typically cold for that software thread (i.e., they contain little data and few instructions useful to it) and the running of the "new" software thread on that core "pollutes" the CPU caches for "old" threads that had been running on that core are likely to be scheduled to run there again. Avoiding oversubscription is difficult. 

These problems can be avoided using `std::async`.

`std::async` when called doesn't guarantee that it will create a new software thread. Rather, it permits the scheduler to arrange for the specified function to be run on the thread requesting `doAsyncWork` result, and reasonable schedulers take advantage of that freedom if the system is oversubscribed or is out of threads. 

With `std::async`, responsiveness on a GUI thread can still be problematic, because the scheduler has no way of knowing which of your threads has tight responsiveness requirements. In that case, you'll want to pass the `std::launch::async` launch policy to `std::async`. That will ensure that the function will run on a different thread. 

If program directly with `std::thread`s, you assume the burden of dealing with thread exhaustion, oversubscription, and load balancing yourself, not to mention how your solution to these problems mesh with the solution implemented in programs running in other processes on the same machine. 

There are some situations where using threads directly may be appropriate. They include: 
+ **You need access to the API of the underlying threading implementation**. To provide access to the API of the underlying threading implementation, `std::thread` objects typically offer the `native_handle` member function.
+ **You need to and are able to optimize thread usage for your application**. For example, with known execution profile that will be deployed
+ **You need to implement threading technology beyond the C++ concurrency API**, e.g.,, thread pool on platform where your C++ implementations doesn't offer them.

### Thing to Remember 
+ The `std::thread` API offers no direct way to get return values from asynchronously run functions, and if those functions throw, the program is terminated.
+ Thread-based programming calls for manual management of thread exhaustion, oversubscription, load balancing, and adaptation to new platforms.
+ Taks-based programming via `std::async` with the default launch policy handles most of these issues for you. 