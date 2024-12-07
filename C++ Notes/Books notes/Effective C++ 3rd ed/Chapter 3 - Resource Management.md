## Item 13: Use objects to manage resources.
There are several ways function could fail to delete object: there might be a premature `return` statement somewhere inside function; in loops - loop can prematurely existed by `break` or `gpoto` statement; or some exception thrown. Skipping `delete` expression is memory leak.
To make sure that the created resource is always released, we need to put that resource inside an object whose destructor will automatically release the resource when control leaves function (smart pointers).

The idea of using objects to manage resources is often called _Resource Acquisition Is Initialization (RAII)_. Resources are acquired and immediately turned over to resource-managing objects. 

Examples of such resource-managin object is `std::auto_ptr`.
> Note: `std::auto_ptr` deprecated in C++11 and removed in C++17. In modern C++ `std::unique_ptr` should be used instead.

An alternative is a _reference-counting smart pointer (RCSP)_. RCSP keeps track of how many objects point to a particular resource and automatically deletes the resource when nobody is pointing to it any longer. It's similar to garbage collection, but unlike garbage collection, RCSPs can't break cycles of references (e.g. two otherwise unused objects that points to one another). In STL as RCSP presented `std::shared_ptr` (and `std::weak_ptr`).

Both smart pointers from STL use `delete` in their destructors, not `delete[]`. That means that using smart pointer with dynamically allocated arrays is a bad idea. 



## Item 14: Think carefully about copying behavior in resource-managing classes.
RAII classes bring with them one important questions: what should happen when an RAII object is copied? 
Most of the time, required to choose one of the following possibilities: 
+ **Prohibit copying**
+ **Reference-count the underlying resource**. This used by `std::shared_ptr` (and `std::weak_ptr`). Important note that `std::shared_ptr` default behavior is to delete what it points to when the reference count goes to zero; in some cases that' nit what desired (for example, with Mutex we want to unlock it, not delete it). `std::shared_ptr` allows specification of a "deleter" - a function or function object to be called when the reference count goes to zero.
+ **Copy the underlying resource**. Worth mentioning that when making copy of pointer - also required to make copy of memory it points to, otherwise deleting of one copy will invalidate all other such incomplete copies.
+ **Transfer ownership of the underlying resource**



## Item 15: Provide access to raw resources in resource-managing classes.
RAII classes don't exist to encapsulate something; they exist to ensure that a particular actions - resource release - takes place. Some functions might take raw pointer to some object as argument. 

There are two general ways to convert an object of the RAII class into the raw resource it contains: explicit conversion and implicit conversion. 

Explicit conversion - return (a copy of) raw pointer inside smart pointer object (`get()` function).

Implicit conversion - overloading the pointer dereferencing operators (`operator->` and `operator*`)

In general, explicit conversion is safer, but implicit conversion is more convenient for clients. 



## Item 16: Use the same form in corresponding uses of `new` and `delete`.
When we employ a `new` expression, two things happen: first, memory is allocated; second, one or more constructors are called for that memory. When we employ a `delete` expression, two other things happen: one or more destructors are called for the memory, then the memory is deallocated. The big question for `delete` us this: how many objects reside in the memory being deleted?

If used `[]` in `new` expression, `[]` must be used in the corresponding `delete` expression. If `[]` don't used in a `new` expression - don't use `[]` in the matching `delete` expression. 



## Item 17: Store newed objects in smart pointers in standalone statements.
Mindful of the wisdom of using objects to manage resources, such function call can be used:

```c++
int priority();
void processWidget(std::shared_ptr<Widget> pw, int priority);

processWidget(std::shared_ptr<Widget>(new Widget), priority());

```
Surprisingly, although we're using object-managing resources everywhere here, this call may leak resources. First argument consists of two parts: 
+ Execution of the expression `new Widget`
+ A call to the `std::shared_ptr` constructor. 
C++ compilers are granted considerable latitude in determining the order in which these things are to be done. The `new Widget` expression must be executed before the `std::shared_ptr` constructor can be called, because the result of the expression is passed as an argument to the `std::shared_ptr` constructor, but the call to `priority` can be performed first, second, or third. 
But what will happen if `priority` called between `new Widget` and `std::shared_ptr` constructor and `priority` yields an exception - the pointer returned from `new Widget` will be lost. 

The way to avoid problems like this is simple: use a separate statement to create the Widget and store it in a smart pointer, then pass the smart pointer:
```c++
std::shared_ptr<Widget> pw(new Widget);
processWidget(pw, priority());
```

> From C++11 also available `std::make_shared`, which can be used in function call and solve descripted problem. 