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



## Item 36: Specify `std::launch::async` if asynchronicity is essential. 
`std::async` is really a request that the function be run in accord with `std::async` _launch policy_. There are tow standard policies, each represent by an enumerator in `std::launch` scoped enum.
+ _The `std::launch::async` launch policy_ means that `f` must be run asynchronously, i.e., on a different thread. 
+ _The `std::launch::deferred` launch policy_ means that `f` may run only when `get` or `wait` is called on the future returned by `std::async`. This is simplification. what matters isn't the future on which `get` or `wait` is invoked, it's the shared state to which the future refers. Because `std::futures` support moving and can also be used to construct `std::shared_futures`, and because `std::shared_futures` can be copied, the future object referring to the shared state arising from the call to `std::async` to which `f` was passed is likely to be different from the one returned by `std::async`. That's a mouthful, however, so it's common to simply talk about invoking `get` or `wait` on the future returned from `std::async`.

`std::async`'s def-ault launch policy - is neither of these, it's these or-ed together. The default policy thus permits `f` to be run either asynchronously or synchronously.

Given a thread `t` execution `auto fut = std::async(f)`,
+ **It's no possible to predict whether `f` will run concurrently with `t`**, because `f` might be scheduled to run deferred.
+ **It's not possible to predict whether `f` runs on a thread different from the thread invoking `get` or `wait` on `fut`.** If that thread is `t`, the implication is that it's not possible to predict whether `f` runs on a thread different from `t`.
+ **It may not be possible to predict whether  `f` runs at all**, because it may not be possible to guarantee that `get` or `wait` will be called on `fut` along every path through the program. 

The default launch policy's scheduling flexibility often mixes poorly with the us of `thread_local` variables, because it means that if `f` reads or writes such _thread-local storage_ (TLS), it's not possible to predict which thread's variable will be accessed. It also affect `wait`-based loops using timeouts, because calling `wait_for` or `wait_until` on a task that's deferred yields the value `std::launch::deferred`. This means that the loop, which looks like it should eventually terminate, may, in reality, run forever:
```c++
using namespace std::literals;

void f() { std::this_thread::sleep_for(1s); }

auto fut = std::async(f);

while (fut.wait_for(100ms) != std::future_status::ready)
{ 
.. // loop until f has finished running, which may never happen!
}
```

The fix is simple: just check the future corresponding to the `std::async` call to see whether the task is deferred, and, if so, avoid entering the timeout-based loop. Unfortunately, there's no direct way to ask a future whether its task is deferred. Instead, you have to call a timeout-based function - a function such as `wait_for`.
```c++
auto fut = std::async(f);

if (fut.wait_for(0s) == std::future_status::deferred)
{
    // use wait or get on fut to call f synchronously 
}
else 
{
    while (fut.wait_for(100ms) != std::future_status::ready)
    {
        // task is neither deferred nor ready, so do concurrent work until it's ready
    }
    // fut is ready.
}
```

The upshot of these various considerations is that using `std::async` with the default launch policy for a task is fine as long as the following conditions are fulfilled:
+ The task need not run concurrently with the thread calling `get` or `wait`.
+ It doesn't matter which thread's `thread_local` variables are reed or written.
+ Either there's a guarantee that `get` or `wait` will be called on the future returned by `std:async` or it's acceptable that the task may never execute.
+ Code using `wait_for` or `wait_until` takes the possibility of deferred status into account. 

### Things to Remember 
+ The default  launch policy for `std::async` permit both asynchronous and synchronous task executions.
+ This flexibility leads to uncertainty when accessing `thread_local`s, implies that the task may never execute, and affects program logic for timeout-based `wait` calls.
+ Specify `std::launch:async` if asynchronous task execution is essential. 



## Item 37: Make `std::thread`s unjoinable on all paths. 
A joinable `std::thread` corresponds to an underlying asynchronous thread of execution that is or could be running. A `std::thread` corresponding to an underlying thread that's blocked or waiting to be scheduled is joinable, for example, `std::thread` object corresponding to underlying threads that have run to completion are also considered joinable.

Unjoinable `std::thread` objects include:
+ Default-constructed `std::thread`s. 
+ `std::thread` objects that have been moved from.
+ `std::thread`s that have been `join`ed.
+ `std::thred`s that have been `detach`ed.

One reason a `std::thread`'s joinability is important is that if the destructor for a joinable thread is invoked, execution of the program us terminated. 

```c++
constexpr auuto tenMillion = 10'000'000;

bool doWork(std::function<bool(int)> filter, 
            int maxValue = tenMillion)
{
    std::vector<int> goodVals;
    
    std::thread t([&filter, maxVal, &goodVals]
                 {
                     for (auto i = 0; i <= maxVal; ++i)
                         if (filter(i))
                             goodVals.push_back(i);
                 });

    auto nh = t.native_handle();

    if (conditionsAreSatisfied())
    {
        t.join();
        preformComputation(goodVals);
        retur true;
    }
    
    return false;
}
```

If `conditionsAreSatisfied()` return `true`, all is well, but if it return `false` or throws an exception, the `std::thread` object `t` will be joinable when its destructor is called at the end of `doWork`. That would cause program execution to be terminated. Such destruction behavior is because the two other obvious following options arguably worse:
+ **An implicit join**. In this case, a `std::thread`'s destructor would wait for its underlying asynchronous thread of execution to complete. That sound reasonable, but it could lead to performance anomalies that would be difficult to track down.
+ **An implicit detach**. In this case, a `std::thread`'s destructor would sever the connection between the `std::thread` object and its underlying thread of execution. The underlying thread would continue to run. 

This is reason to ensure that `std::thread` objects unjoinable on every path out if the scope. 

In C++14 yet there is no standard RAII class for `std::thread` objects. 
> In C++20 added `std::jthread`s.

```c++
class ThreadRAII {
public:
    enum class DtorAction { join, detach };

    ThreadRAII(std::thread&& t, DtorAction a)
        : action(a)
        , t(std::move(t))
    {}
    
    ~ThreadRAII()
    {
        if (t.joinable()) 
        {
            if (action == DtorAction::join)
                t.join();
            else
                t.detach();
        }
    }

    ThreadRAII(ThreadRAII&&) = default;
    ThreadRAII& operator=(ThreadRAII&&) = default;

    std::thread& get() { return t; }
private:
    DtorAction action;
    std::thread t;
};
```

The parameter order in the constructor is designed to be intuitive to callers, but the member initialization list is designed to match the order of the data members' declaration. That order puts the `std::thread` object in the end since it's possible for member initialization of one data member to depend on anther, and because `std::thread` object may start running a function immediately after they are initialized. 

Before the `ThreadRAII` destructor invokes a member function on the `std::thread` object `t`, it checks to make sure that `t` is joinable. This is necessary, because invoking `join` or `detach` on an unjoinable thread yield undefined behavior. 

A `std::thread` object can change state from joinable to unjoinable only through a member function call, e.g., `join`, `detach`, or a move operations. At the time a `ThreadRAII` object's destructor is invoked, no other thread should be making member function calls on that object. If there are simultaneous calls, there is certainly a race, but isn't inside the destructor, it's in the client code that is trying to invoke two member functions (the destructor and something else) on one object at the same time.
The "proper" solution to these kinds of problems would be to communicate to the asynchronously running lambda that we no longer need its work and that it should return early. 

### Things to Remember 
+ Make `st::thread`s unjoinable on all paths. 
+ `join`-on-destruction can lead to defficult-to-debug performance anomalies. 
+ `detach`-on-destruction can lead to difficult-to-debug undefined behavior.
+ Declare `std::thread` objects last in lists of data members.




## Item 38: Be aware of varying thread handle destructor behavior. 
Both `std::thread` objects and future objects can be thought of as _handles_ to system threads. But `std::thread` and futures have different behaviors in their destructors.

Destruction of a joinable `std::thread` terminates program, because the two obvious alternatives - an implicit `join` and an implicit `detach` - were considered worse choices. Yet the destructor for a future sometimes behaves as if it did an implicit `join`, sometimes as if it did an implicit `detach`, and sometimes neither. It never cause program termination. 

Future is one end of a communication channel through which a callee transmits a result to a caller. The callee (usually running asynchronously) writes the result of its computation into the communications channel (typically via a `std::promise` object), and the caller reads that result using a future. 

The callee could finish before the caller invokes `get` on a corresponding future, so the result can't be stored in the callee's `std::promise`, that object, being local to the callee, would be destroyed when the callee finished. 

The result can't be stored in the caller's future, either, because a `std::future` may be used to create a `std::shared_future` (thus transferring ownership of the callee's result from `std::future` to the `std::shared_future`), which may then be copied many times after the original `std::future` is destroyed. 

Result is stored in a location outside the caller or the callee, in _shared state_.

The existence of the shared state is important, because the behavior of a future's destructor is determined by the shared state associated with the future:
+ _The destructor for the last future referring to a shared state for a non-deferred task launched via `std::async` blocks until task completers_. In essence, the destructor for such a future does an implicit `join` on the thread on which the asynchronously executing task is running. 
+ _The destructor for all other futures simply destroys the future object_. For asynchronously running tasks, this is akin to an implicit `detach` on the underlying thread. For deferred tasks for which this is the final future, it means that the deferred task will never run.

The normal behavior is that a future's destructor destroys the future object and also decrements the reference count inside the shared state that's manipulated by both the futures referring to it and the callee's `std::promise`. This reference count makes it possible for the library to know when the shared state can be destroyed. 

The exception to this normal behavior arises only for a future for which all of the following apply:
+ _It refers to a shared state that was created due to a call to `std::async`_
+ _The task's launch policy is `std::launch::async`_, either because that was chosen by the runtime system or because it was specified in the call to `std::async`
+ _The future is the last future referring to the shared state._ For `std::futures` thus will always be the case. For `std::shared_futures`, if other `std::shared_futures` refer to the same shared state as the future being destroyed, the future being destroyed follows the normal behavior (i.e., it simply destroy its data members).

Only when all of these conditions are fulfilled does a future's destruction exhibit special behavior, and that behavior is to block until the asynchronously running task completes. Practically speaking, this amounts to an implicit `join` with the thread running the `std::async`-created task.

The API for futures offers no way to determine whether a future refer to a shared state arising from a call to `std::async`, so given an arbitrary future object, it's not possible to known whether it will block in its destructor waiting for an asynchronously running task to finish.

Shared states arising from calls to `std::async` qualify for the special behavior, but there are other ways that shared states get created. One is use of `std::packaged_task`. A `std::packaged_task` object prepares a function (or other callable object) for asynchronous execution by wrapping it such that its result is put into a shared state. A future referring to that shared state can then be obtained via `std::packaged_task`'s `get_future` function. At this point, we known that the future doesn't refer to a shared state created by a call to `std::async`, so its destructor will behave normally. 

After moving obtained with such way future into `std::thread`, after creating such thread object `t` following possibilities can happen:
+ _Nothing happens to `t`_. In this case `t` will be joinable at the end of the scope. That will cause the program to be terminated. 
+ _A `join` is done to `t`_. In this case, there would be no need for future to block in its destructor, because the `join` is already present in the calling code.
+ _A `detach` is done on `t`_. In this case, there would be no need for future to `detach` in its destructor, because the calling code already does that.
In other words, for future corresponding to a shared state that arose due to a `std::packaged_task` there's usually no need to adopt a special destruction policy, because the decision among termination, joining or detaching will be made in the code that manipulates the `std::thread` on which the `std::packaged_task` is typically run.

### Things to Remember 
+ Futures destructors normally just destroy the future's data members. 
+ The final future referring to a shared state for a non-deferred task launched via `std::async` blocks until the task completes. 



## Item 39: Consider `void` futures for one-shot event communications. 
If we call the task that detects the condition the _detecting task_ and the task reacting to the condition the _reacting task_, the strategy is simple: the reaction task waits on a condition variable, and the detecting thread notifies that condvar when the event occurs. 

The code for the reacting task before calling `wait` on the condvar, it must lock a mutex through a `std::unique_lock` object. Locking a mutex before waiting on a condition variable is typical for threading libraries. The need to lock the mutex through a `std::unique_lock` object is simply part of C++11 API. 

There are two problems to pay attention to:
+ **If the detecting task notifies the condvar before the reacting task `wait`s, the reacting task will hang***
+ **The `wait` statement fails t account for spurious wakeups**. Code waiting on condition variables may be awakened even if the condvar wasn't notified. Such awakenings are known as _spurious wakeups_. The C++ condvar API make this exceptionally easy, because it permits a lambda (or other function objects) that test for the waited-for condition to be passed to `wait`. Taking advantage of this capability requires that the reacting task be able to determine whether the condition it's waiting for is true. But in the scenario we've been considering, the condition it's waiting for is the occurrence of an event that detecting thread is responsible for recognizing. The reacting thread may have no way of determining whether the event it's waiting for has taken place. That's why it's waiting on a conditional variable!

Another solution is flag-based design. When the detecting thread recognize the event it's looking for, it sets the flag (better `std::atomic<bool>`) and reacting thread simply polls the flag `while(!flag)`. But it's consume hardware thread that another task might be able to make use of. 

It's common to combine the condvar and flag-based designs. A flag indicates whether the event of interest has occurred, but access to the flag is synchronized by a mutex. It works regardless of whether the reacting task `wait`s before the detecting task notifies, it works in the presence of spurious wakeups, and it doesn't require polling. 

```c++

std::conditional_variable cv;
std::mutex m;

// detecting task
bool flag { false };
...
{
    std::lock_guard<std::mutex> g(m);
    flag = true;
}

cv.notify_one();

// reacting task
{
    std::unique_lock<std::mutex> lk(m);
    cv.wait(lk, [] { return flag; });
    ..
}
..
```

An alternative is to avoid condition variables, mutexes, and flags by having the reacting task `wait` on a future that's set by the detecting task. A future represents the receiving end of a communications channel from a callee to a caller, even though there's no callee-caller relationship between the detecting and reacting tasks. However, a communication channel whose transmitting end is a `std::promise`, can be used in any situation where required to transmit information from one place in program to another. In this case, we'll use it to transmit information from the detecting task to the reacting task, and the information we'll convey will be that the event of interest has taken place.

When the detecting task sees that the event it's looking for has occurred, it _sets_ the `std::promise` (i.e., writes into the communication channel). Meanwhile, the reacting task `wait`s on its future. That `wait` blocks the reacting task until `std::promise` has been set. Since there is no data, `void` is suitable type for such promise. 

```c++
std::promise<void> p;
// detecting task
...
p.set_value();

// reacting task
...
p.get_future().wait();
...
```

Between a `std::promise` and a future is a shared state, and shared states are typically dynamically allocated. Thus, this design incurs the cost of heap-based allocation and deallocation.

`std::promise` may be set only once. The communication channel between a `std::promise` and a future is a _one-shot_ mechanism: it can't be used repeatedly. 

Assuming you want to suspend a thread only once (after creating, but before it's running its thread function), a design using a `void` future is a reasonable choice.
```c++
std::promise<void> p;

void react();

void detect()
{
    std::thread t([]
                  {
                      p.get_future().wait();
                      react();
                  });
    ... // here t is suspended
    
    p.set_value(); // unsuspend t (and thus call react)
    
    .. // to addditional work
    
    t.join();
}
```

`std::future::share` member function transfers ownership of its shared state to the `std::shared_future` object produced by `share`. Each reacting thread needs its own copy of the `std::shared_future` that refers to the shared state, so the `std::shared_future` obtained from `shar` is captured by value by the lambdas running on the reacting threads
```c++
std::promise<void> p;

void detect()
{
    auto sf = p.get_future().share();

    std::vector<std::thread> vt;

    for (int i = 0; i < threadsToRun; ++i)
        vt.emplace_back([sf]
                        {
                            sf.wait();
                            react(); 
                        });
    ... // detect hangs if this code throws!

    p.set_value(); //unsuspend all threads
    ..
    for (auto& t : vt)
        t.join();
}
```

### Things to Remember 
+ For simple event communication, condvar-based design require a superfluous mutex, impose constraints on the relative progress of detecting and reacting tasks, and require reacting tasks to verify that the event has taken place.
+ Design employing a flag avoid those problems, but are based on pooling, not blocking.
+ A condvar and flag can be used together, but the resulting communications mechanism is somewhat stilted.
+ Using `std::promise`s and futures dodges these issues, but the approach uses heap memory for shared states, and it's limited to one-shot communication. 



## Item 40: Use `std::atomic` for concurrency, `volatile` for special memory.
`std::atomic` templates offer operations that are guaranteed to be seen as atomic by other threads. Once a `std::atomic` object has been constructed, operations on it behave as if they were inside a mutex-protected critical section, but the operations are generally implemented using special machine instructions that are more efficient than would be the case if a mutex were employed. 

`std::atomic` guarantees only that the read is atomic. There is no guarantee that the entire statement proceeds atomically. For example, during printing of `std::atomic<int> ai(0);` in `std::cout << ai;` between time `ai`'s value is read and `operator<<` is invoked, another thread may have modified `ai`'s value. However, `++ai` and `--ai`, these are each read-modify-write (RMW) operations, yet they execute atomically. 

Compilers are permitted to reorder assignments to independent variables. However, the use of `std::atomic`s imposes restriction on how code can be reordered, and one such restriction is that no code that, in the source code, precedes a write of a `std::atomic` variable may take place (or appear to other cores to take place) afterwards, meaning that no only compilers retain the order of the assignment, they must generate code that ensures that the underlying hardware does, too. This is true only for `std::atomic`s using _sequential consistency_ (`std::memory_order_seq_cst`), which is both the default and the only consistency mode for `std::atmoic` object that use the syntax `ai = 10`. C++11 also supports consistency models with more flexible code reordering rules (like `std::memory_order_relaxed` etc), which make it possible to create software that runs faster on some hardware architectures, but the use of such models yields software that is much more difficult to get right, to understand, and to maintain. Subtle errors in code using relaxed atomics is not uncommon, even for experts, so you should stick to sequential consistency if at all possible.

Declaring variable as `volatile` doesn't impose the same reordering restrictions. But `volatile` impose another restriction for compiler. In a nutshell, `volatile` telling compilers that they're dealing with memory that doesn't behave normally. "Normal" memory has the characteristic that if you write a value to a memory location, the value remains there until something overwrites it. Compiler can optimize the generated code by eliminating _redundant loads_ and _dead store_.
```c++
int x;

auto y = x;
y = x;  // redundant, already used x in assignment

x = 10; 
x = 20; // rewrite x without reading 10 from x

// Compiler can optimize it like 
auto y = x;

x = 20;
```

`volatile` disable any optimizations on operations on memory. It's can be used for "special" memory, like _memory-mapped I/O_, where such reading and writing make sense (like, reading sensor value).

Compilers are permitted to eliminate such redundant operations on `std::atomic`s.

For `std::amotic` copy operations are deleted. In order for the copy construction of `y` from `x` to be atomic, compilers would have to generated code to read `x` and write `y` in a single atomic operation. Hardware generally can't do that, so copy construction isn't supported for `std::atomic` types. Copy assignment is deleted for the sane reason. 

It's possible to "copy" `x` to `y` separating read and write on two operations via `std::atomic` member functions `load` and `store`
```c++
std::atomic<int> y(x.load());
y.store(x.load());
```

The situation should thus be clear:
+ `std::atomic` is useful for concurrent programming, but not for accessing special memory
+ `volatile` is useful for accessing special memory, but not for concurrent programming.
Because `std::atomic` and `volatile` serve different purposes, they can be used together. This could be useful if `volatile std::atomic<int>` corresponded to a memory-mapped I/O location that was concurrently accessing by multiple threads. 

### Things to Remember 
+ `std::atmoic` is for data accessed from multiple threads without using mutexes. It’s a tool for writing concurrent software.
+ `volatile` is for memory where reads and writes should not be optimized away. It’s a tool for working with special memory.