## Item 18: Use `std::unique_ptr` for exclusive-ownership resource management.
It's reasonable to assume that, default, `std::unique_ptr`s are the same size as raw pointers, and for most operations (including dereferencing) they execute exactly the same instructions.

`std::unique_ptr` embodies _exclusive ownership_ semantics. Copying isn't allowed, because if it was allowed, that mean that two `std::unique_ptr` will point to the same resource, each thinking is owner of it. Upon destruction, a non-null `std::unique_ptr` destroy its resource, by default, resource destruction is accomplished by applying delete to the raw pointer inside the `std::unique_ptr`.

If the ownership chain got interrupted due to an exception or other atypical control flow (e.g., early function return), the `std::unique_ptr` owning the managed resource would eventually have its destructor called, and the resource it was managing would thereby be destroyed. There are a few exceptions to this. If an exception propagates out of a thread's primary function (e.g., `main`, for the program's initial thread) of a `nonexcept` specification is violated, local object may not be destroyed, and if `std::abort` or an exit function (i.e. `std::exit`) is called, they defiantly won't be.

By default, object destruction would take place via `delete`, but, during construction, `std::unique_ptr` objects can be configurated to use _custom deleters_: arbitrary functions (or function objects, including those arising from lambda expressions) to be invoked when it's time for their resource to be destroyed. 

All custom deletion functions accept a raw pointer to the object to be destroyed, then do what it necessary to destroy that object. When a custom deleted is to be used, its type must be specified as the second type argument to `std::unique_ptr`. When a custom deleter is to be used, it's type must be specified as the second type argument to `std::unique_ptr`. For factory functions that return `std::unique_ptr` the basic strategy  is to create a null `std::unique_ptr`, make it point to an object of the appropriate type, and then return it. Attempting to assign a raw pointer to a `std::unique_ptr` won't compile, because it would constitute an implicit conversation from a raw to a smart pointer. `std::unique_ptr::reset` member function can be used to assume ownership of the object created via `new`. With each `new` use of `std::forward` for perfect-forward the arguments passed makes all the information provided by callers available to the constructor of the objects being created. 
```c++
auto delInvm = [](Investment* pInvestment)
               {
                   makeLogEntry(pInvestment);
                   delete pInvestment;
               };
template<typename... Ts>
std::unique_ptr<Investment, decltype(delInvmt)>
makeInvestment(Ts&&... params)
{
    std::unique_ptr<Investment, decltype(delInvmt)>
      pInv(nullptr, delInvmt);
    if (/* a Stock object should be created */)
        pInv.reset(new Stock(std::forward<Ts>(params)...))
    else if (/* a Bond object should be created */)
	     pInv.reset(new Bond(std::forward<Ts>(params)...))

    return pInv;
}
```

Deleters that are function pointers generally cause the size of a `std::unique_ptr` to grow form one word to two. For deleters that are functions objects, the change in size depends on how much state is stored in the function object. Stateless objects (e.g., from lambda expressions with no captures) incur no size penalty, and this means that when a custom deleter can be implemented as either a function or a captureless lambda expression, the lambda is preferable. 

`std::unique_ptr` comes in two forms, one for individual objects (`std::unique_ptr<T>`) and one for arrays (`std::unique_ptr<T[]>)`). As a results, there's never any ambiguity about what kind of entity a `std::unique_ptr` points to. The `std::unique_ptr` API is design to match the form was used. 

`std::unique_ptr` is the C++11 way to express exclusive ownership, but one of its features is that it easily and efficiently converts to a `std::shared_ptr`.

### Things to Remember
+ `std::unique_ptr` is a small, fast, move-only smart pointer for managing resources with exclusive-ownership semantics. 
+ By default, resource destruction takes place via `delete`, but custom deleters can be specified. Stateful deleters and function pointers as deleters increase the ize of `std::unique_ptr` objects. 
+ Converting a `std::unique_ptr` to a `std::shared_ptr` is easy.



## Item 19: use `std::shared_ptr` for shared-ownership resource management. 
An object accessed via `std::shjared_ptr`s has its lifetime managed by those pointers through _shared ownership_. No specific `std::shared_ptr` owns the object. Instead, all `std::shared_ptr`s pointing to it collaborate to ensure its destruction at the point where it's no longer needed.

A `std::shared_ptr` can tell whether it's the last one pointing to a resource by consulting the resource's _reference count_, a value associated with the resource that keeps track of how many `std::shared_ptr`s point to it. `std::shared_ptr` constructors increment this count (except move-constructing), destructors decrement it and copy assignment operators do both. 

The existence of the reference count has performance implications: 
+ `std::shared_ptr`s are twice the size of a raw pointer, because they internally contain a raw pointer to the resource as well as a raw pointer to the resource's reference count (this implementation is not required by the Standard, but every STL implementation employs it).
+ Memory for reference count must be dynamically allocated.
+ Increments and decrements of the reference count must be atomic. Atomic operations are typically slower than non-atomic, so even though reference counts are usually only a word in size, reading and writing them is comparatively costly.

Like `std::unique_ptr`, `std::shared_ptr` uses `delete` as its default resource-destruction mechanism and also supports custom deleters. The design of this support differs from that for `std::unique_ptr`, however. 
For `std::unique_ptr`, the type of the deleter is part of the type of the smart pointer, for `std::shared_ptr` it's not. 
```c++
auto loggingDel = [](Widget* pw) 
                  {
                     makeLogEntry(pw);
                     delete pw; 
                  };
std::unique_ptr<Widget, decltype(loggingDel)> 
upw(new Widget, loggingDel);

std::shared_ptr<Widget>
spw(new Widget, loggingDel);
```

The `std::shared_ptr` design is more flexible. Two `std::shared_ptr` with custom, but different deleters will have same type, thus they can be placed in a container of objects of that type. They could also be assigned to one another, and they could each be passed to a function taking a parameter of type `std::shared_ptr<Widget>`. 
None of these things can be done with `std::unique_ptr`s that differ in the types of their custom deleters, because the type of the custom deleter would affect the type of the `std::unique_ptr`.
In another difference from `std::unique_ptr`, specifying a custom deleter doesn't change the size of a `std::shared_ptr` object. Regardless of deleter, a `std::shared_ptr` object is two pointers in size. Custom deleters can be function objects, and function objects can contain arbitrary amounts of data. That means they can be arbitrary large. 

Trick is that deleter isn't part of the `std::shared_ptr` object. It's on the heap or, if the creator of the `std::shared_ptr` took advantage of `std::shared_ptr` support for custom allocators, it's wherever the memory managed by the allocator is located. 

Reference count is part of data structure _control block_. There's a control block for each object managed by `std::shared_ptr`s. The control block contains, in addition to the reference count, a copy of the custom deleter, if one has ben specified. If a custom allocator was specified, the control block contains a copy of that, too. The control block may also contain additional data, weak counter. 

An object's control block is set up by the function creating the first `std::shared_ptr` to the object in following cases: 
+ `std::make_shared` always creates a control block.
+ A control block is created when a `std::shared_ptr` is constructed from a unique-ownership pointer (`std::unique_ptr` or `std::auto_ptr`)
+ When a `std::shared_ptr` constructor is called with a raw pointer, it creates a control block.
A consequence of these rules is that constructing more than one `std::shared_ptr` from a single raw pointer gives undefined behavior, because the pointed-to object will have multiple control blocks. 

Therefor, general recommendation to avoid passing raw pointer and use `std::make_shared` (but this doesn't allow to set custom deleter) or pass raw pointer returned from `new` directly to `std::make_shared` constructor.

Another example for undefined behavior:
```c++
std::vector<std::shared_ptr<Widget>> processedWidgets;
class Widget {
public:
  void process ()
  {
   ...
   processedWidgets.emplace_back(this);
  }
};
```
This code will compiler, `std::shared_ptr` thus constructed will create a new control block for the pointed-to `Widget`(`*this`). That doesn't sound harmful until case where there are `std::shared_ptr`s outside the member function that already point to that `Widget`.
The `std::shared_ptr` API includes a facility for just his kind of situation, `std::enable_shared_from_this`. That's a template to a base class that allow safely create `std::shared_ptr` from `this` pointer if class inherited from this base class. The design pattern behind it called _The Curiously Recurring Template Pattern_ [(CRTP)](https://en.cppreference.com/w/cpp/language/crtp). `std::enable_shared_from_this` defines a member function `shared_from_this()` that creates a `std::shared_ptr` to the current object, but it does it without duplicating control block.

Internally, `shared_from_this` looks up the control block for the current object, and it creates a new `std::shared_ptr` that refers to that control block. The design relies on the current object having an associated control block. For that to be the case, there must be an existing `std::shared_ptr` that points to the current object. If no such `std::shared_ptr` exists, behavior is undefined, although `shared_from_this` typically throw an exception. 

To prevent clients from calling member function that invoke `shared_from_this` before a `std::shared_ptr` points to the object, classes inheriting from `std::enable_shared_from_this` often declare their constructors `private` and have clients create objects by calling factory functions that return `std::shared_ptr`s. 
```c++
class Widget : public std::enable_shared_from_this<Widget> {
public:
    template<typename... Ts>
    static std::shared_ptr<Widget> create(Ts&&... params);
    
    void process ()
    {
        ...
        processedWidgets.emplace_back(shared_from_this());
    }
private: 
    Widget();
};
```

A control block is typically only a few words in size, although custoim deleters and allocators may make it larger. The usual control block implementation is more sophisticated than might be expected. It makes use of inheritance, and there's even a virtual functions. (It's used to ensure that the pointed-to object is properly destroyed). That means that using `std::shared_ptr`s also incurs the cost of the machinery for the virtual function used by the control block. 

Under typical conditions, where the default deleter and default allocator are used and where the `std::shared_ptr` is created by `std::make_shared`, the control block is only about three words in size, and it's allocation is essentially free. 

Dereferencing a `std::shared_ptr` is no more expensive than dereferencing a raw pointer. Performing an operation requiring a reference count manipulation (e.g. copy construction or copy assignment, destruction) entails one or two atomic operations, but these operations typically map to individual machine instructions, so although they may be expensive compared to non-atomic instructions, they're still just single instructions. The virtual functions machinery in the control block is generally used only per object managed by `std::shred_ptr`s: when the object is destroyed. 

There's no `std::shared_ptr<T[]>`.

It's possible to use `std::shared_ptr<T>` to point to an array, specifying a custom deleter to perform an array `delete[]`, but it's a horrible idea. For one thing, `std::shared_ptr` offers no `operator[]`, so indexing into the array requires awkward expressions based on pointer arithmetic. For another, `std::shared_ptr` support derived-to-base pointer conversions that makes sense for single objects, but that open holes in the type system when applied to arrays. 

### Things to Remember 
+ `std::shared_ptr`s offer convenience approaching that of garbage collection for the shared lifetime management of arbitrary resources.
+ Compared to `std::unique_ptr`, `std::shared_ptr` objects are typically twice as big, incur overhead for control blocks, and require atomic reference count manipulations.
+ Default resource destruction is via `delete`, but custom deleters are supported. The type of the deleter has no effect on the type of the `std::shared_ptr`.
+ Avoid creating `std::shared_ptr`s from variables of raw pointer types. 



## Item 20: Use `std::weak_ptr` for `std::shared_ptr` like pointers that can dangle. 
`std::weak_ptr` is kind of smart pointer that track _pointer dangle_, i.e. when the object it is supposed to pointer to no longer exists; solve problem unknown to `std::shared_ptr`s: the possibility that what it points to has been destroyed. 

`std::weak_ptr` can't be dereferenced, nor can they be tested for nullness. That's because `std::weak_ptr` isn't a standalone smart pointer, it's an augmentation of `std::shared_ptr`.

`std::weak_ptr`s are typically created from `std::shared_ptr`s. They point to the same place as the `std::shared_ptr`s initializing them, but they don't affect the reference count of them object they point to. 
 
`std::weak_ptr`s that dangle are said the have _expired_. This can be check, but even if `std::weak_ptr` not expired, object can't be access through it. Separating the check and the dereference would introduce a race condition: between the call to `expired` an the dereferencing actions, another thread might reassign or destroy the last `std::shared_ptr` pointing to the object, thus causing that object to be destroyed. In that case dereference would yield undefined behavior. 

What is needed is an atomic operation that checks to see if the `std::weak_ptr` has expired and, if not, gives access to the object it points to. This is done by crating a `std::shared_ptr` from `std::weak_ptr`. The operation comes in two forms, depending on how expired `std::weak_ptr` should be handled. 
In one form the `std::shared_ptr` in null if the `std::weak_ptr` has expired
```c++
auto spw = std::nake_shared<Widget>();
std::weak_ptr(Widget) wpw (spw);
...
std::shared_ptr<Widget> swp1 = wpw.lock();
// or 
auto swp1 = wpw.lock();
```
The other form if `std::weak_ptr` expired, an exception is thrown:
```c++
std::shared_ptr<Widget> spw2(wpw);
```

`std::weak_ptr` can be used as cache pointers, since cache's pointers need to be able to detect when they dangle (since the corresponding cache enty will dangle when clients finished using an object and destroy it), another case is the Observer design pattern. 
The primary components of this pattern are subjects (objects whose state may change) and observers (objects to be notified when state change occur). In most implementations, each subject contains a data member holding pointers to its observers. That makes it easy for subjects to issue state change notification. Subjects have no interest in controlling the lifetime of their observers (i.e. when they're destroyed), but they have a great interest in making sure that if an observer gets destroyed, subject don't try to subsequently access it. A reasonable design is for each subject to hold a container of `std::weak_ptr`s to its observers, thus making it possible for the subject  to determine whether a pointer dangles before using it. 

In strictly hierarchal data structures such as trees, child nodes are typically owned only by their parents. Links from parent to chiildren are thus generally best represented by `std::unique_ptr`s, while backlinks with from children to parents can be safely implemented as raw pointers, because a child node should never have a lifetime longer than its parent. 

`std::weak_ptr` objects are the same size as `std::shared_ptr` objects, they make use if the same control block as `std::shared_ptr`, and operations such as construction, destruction, and assignment involve atomic reference weak-count manipulations.

### Things to Remember 
+ Use `std::weak_ptr` for `std::shared_ptr`-like pointers that can dangle.
+ Potential use cases for `std::weak_ptr` including caching, observer lists, and the prevention of `std::shared_ptr` cycles. 



## Item 21: Prefer `std::make_unique` and `std::make_shared` to direct use of `new`.
`std::make_unique` and `std::make_shared` perfect-forwards its parameters to the constructor of the object being created constructs a corresponding smart pointer from the raw pointer `new` produces, and returns the smart pointer so created. This form of the function doesn't support arryas or custom deleters. 

There also `std::allocate_shared`, it acts just like `std::make_shared`, except its first argument is an allocator object to be used for the dynamic memory allocation.

Use `new` instead of make-function could cause potential resource leak:
```c++
void processWidget(std::shared_ptr<Widget> spw, int priprity)
{ .. }
int main()
{
    processWidget(std::shared_ptr<Widget>(new Widget), computePriority());
}
```
At runtime, the arguments for a function must be evaluated before the function can be invoked, so in the call to `processWidget`, the following things must occur before function execution:
+ The expression `new Widget` must be evaluated.
+ The constructor for the `std::shared_ptr<Widget>` executed.
+ `computePriority` must run.
Compilers are not required to generated code that executes them in this order, so `computePriority` can executes between `std::shared_ptr` creating and emit exception.
Using `std::make_shared` avoid this problem.

A special feature of `std::make_shared` (compare to direct use of `new`) is improved efficiency. Using `std::make_shared` allows compilers to generate smaller, faster code that employs leaner data structures. 

Every `std::shared_ptr` points to a control block containing, among other things, the reference count for the pointed-to object. Memory for this control block is allocated in the `std::shared_ptr` constructor. Direct use of `new` then, requires one memory allocation for the `Widget` and a second allocation for the control block, while `std::make_share` may use one allocation. 

The make functions for perfect forwarding code uses parentheses, not braces. That means that if require to construct pointed-to object using a brace initializer, `new` must be used directly - braced initializers can't be perfect-forwarded. However, use `auto` type deduction to create a `std::initlizer_list` object from a brace initializer, then pass the `auto`-created object through the make function can hack limitation:
```c++
auto initList = { 10, 20 };
auto spv = std::make_shared<std::vector<int>>(initList);
```

Some classes define their own versions of `operator new` and `operator delete`. The presence of these function implies that the global memory allocation and deallocation routines for objects of these types are inappropriate. Such routines are a poor fit for `std::shared_ptr`'s support for custom allocation (via `std::allocate_shared`) and deallocation (via custom deleters), because the amount of memory that `std::allocate_shared` requests isn't the size of the dynamically allocated 
objects, it's size of that object plus the size of control block. 

The reference count tracks how many `std::shared_ptr`s refer to the control block, but the control block contains a second reference count, one that tallies how many `std::weak_ptr`s refer to the control block - weak counter. In practice, the value of the weak count isn't always equal to the number of `std::weak_ptr`s referring to the control block, because library implementers have found way to slip additional information into the weak count that facilitate better code generation. When `std::weak_ptr` checks to see if it has expired, it does so by examining the reference count (not the weak count) in the control block that it refers to. If the reference count is zero, the `std::weak_ptr` has expired. 
As long as `std::weak_ptr`s refer to a control block (i.e., the weak count is greater than zero), that control block must continue to exist. And as long as a control block exists, the memory containing it must remain allocated. The memory allocated by a `std::share_ptr` make function, then, can't be deallocated until the last `std::shared_ptr` and the last `std::weak_ptr` referring to it have been destroyed. 

In situation where use of make-function are impossible or inappropriate, the best way to use `new` is to male suer that when `new` used directly, result immediately passed to a smart pointer constructor in a statement that does nothing else. This prevent compilers from generating code that could emit an exception between the use of `new` and invocation of the constructor for the smart pointer that will manage the object. `std::shared_ptr` assume ownership of the raw pointer passed to its constructor, even if that constructor yields an exception.
```c++
std::shared_ptr<Widget> spw(new Widget, cusDel);
processWidget(std::move(spw), computePrioroty());
```
Because `processWidget`'s `std::shared_ptr` parameter is passed by value, construction from an rvalue entails only a move, while construction from an lvalue requires a copy. For `std::shared_ptr` the difference can be significant, because copying a `std::shared_ptr` require an atomic increment of its reference count, while moving not.

### Things to Remember 
+ Compared to direct use of `new`, make functions eliminate source code duplication, improve exception safety, and, for `std::make_shared` and `std::allocate_shared`, generate code that's smaller and faster.
+ Situations where use of make functions is inappropriate include the need to specify custom deleters and a desire to pass braced initializers.
+ For `std::shared_ptr`s, additional situations where make functions may be ill-advised classes with custom memory management and systems with memory concerns, very large objects, and `std::weak_ptr`s that outlive the corresponding `std::share_ptrs`.



## Item 22: When using the Pimpl idiom, define special member functions in the implementation file.
_Pimpl_ ("pointer to implementation) _Idiom_ us the technique whereby data members of a class replaces with a pointer to an implementation class (or struct), data members that used to be in the primary class put into implementation class, and those data members access through the pointer. 

When as data members used types like `std::string`, `std::vector`, user-defined types, etc., headers for those types must be present for class to compile, and that means that class clients must include them as well. That speeds compilation and reduce clients dependencies on headers.  

Applying the Pimpl Idiom in C++98 could replace data members with a raw pointer to a struct that has been declared, but not defined. 
```c++
// widget.h
class Widget {
public:
    Widget();
    ~Widget();
    ..
private:
    struct Impl;
    Impl* pImpl;
};
```
Part 2 is the dynamic allocation and deallocation of the object that holds the data members that used to be in the original class.
```c++
// widget.cpp

#include "widget.h"
#include "gadget.h"
#include <string>
#include <vector>

struct Widget::Impl {
    std::string name;
    std::vector<double> data;
    Gadget g1, g2, g3;
};

Widget::Widget()
    : pImpl(new Impl)
{}

Widget::~Widget()
{ delete pImpl; }
```

Applying offer by C++11 smart pointers a little bit tricky. Simple use `std::unique_ptr<Impl> pImpl` will cause compilation error during attempt create `Widget` object. The issue arises due the code that's generated for `Widget` object destruction. In the class definition using `std::unique_ptr` we don't need to declare a destructor since default generated destructor suppose to work fine, the compiler inserts code that call the destructor for class members. `std::unique_ptr<Impl> pImpl` will use default deleter, which uses `delete` on the raw pointer inside the `std::unique_ptr`. Prior to using `delete`, implementations typically have the default deleter employ `static_assert` to ensure that the raw pointer doesn't point to an incomplete type, which will fails in our case, since `Widget's` destructor, like all compiler-gernerted special member functions, is implicitly `inline`. 

To fix the problem, code that destroy `std::unique_ptr<Widget::Impl>` should be put in place where `Widget::Impl` is complete type - after definition. That means that `Widget` destructor should be implemented in cpp file. Destructor body also can be defined with `= default`. 
The declaration of a destructor prevents compilers from generating the move operations, if move support required - move function should be declared "manually". Since the compiler-generated move assignment operator need to destroy the object pointed by `pImpl` before reassigning it, incomplete type assert can rise again, solution same as for destructor.
```c++
// widget.h
class Widget {
public:
    Widget();
    ~Widget();

    Widget(Widget&& rhs);
    Widget& operator=(Widget&& rhs);
    ..
private:
    struct Impl;
    std::unique_ptr<Impl> pImpl;
};
```

```c++
// widget.cpp
struct Widget::Impl {
    std::string name;
    std::vector<double> data;
    Gadget g1, g2, g3;
};

Widget::Widget()
    : pImpl(std::make_uniquie<Impl>())
{}

Widget::~Widget() = default;

Widget::Widget(Widget&& rhs) = default;
Widget& Widget::operator=(Widget&& rhs) = default;
```

Copy operation supports require also manual implementation, since compilers won't generate copy operations for move-only types like `std::unique_ptr` and even if they did, the generated functions would copy only the `std::unique_ptr` (i.e., perform a _shallow copy_), while we want to copy what the pointer points to (i.e., perform a _deep copy_). Rather than copy the fiends one by one, better take advantage of the fact that compilers will create the copy operation for `Impl`, and these operations will copy each field automatically 
```c++
Widget::Widget(const Widget& rhs)
    : pImpl(std::make_unique<Impl>(*rhs.pImpl))
{}
Widget& Widget::operator=(const Widget& rhs)
{
    *pImpl = *rhs.pImpl;
    return *this;
}
```

If use `std::shared_ptr` instead `std::unique_ptr`, mentioned advices no longer applied: there'd be no need to declare a destructor, and without a user-declarted destructor compiler will generate move operation.

The difference in behavior between `std::unique_ptr` and `std::shared_ptr` for `pImpl` pointers stems from the differing way these smart pointers supports custom deleters. For `std::unique_ptr`, the type of the deleter is part of the type of the smart pointer, and this makes it possible for compilers to generate smaller runtime data structures and faster runtime code. A consequence of this is that pointer-to types must be complete when compiler-generated special functions (e.g., destructors or move operations) are used. For `std::shared_ptr`, the type of the delete is not part of the type of the smart pointer. This necessitates larger runtime data structures and somehow slower code, but pointed-to types need not be complete when compiler-genereated special functions are employed. 

### Things to Remember
+ The Pimpl Idiom decreases build times by reducing compilation dependencies between class clients and class implementations.
+ For `std::unique_ptr` Pimpl pointers, declare special member functions, in the class header, but implement them in the implementation file. Do this even if the default function implementation are acceptable.
+ The above advice applies to `std::unique_ptr`, but not to `std::shared_ptr`