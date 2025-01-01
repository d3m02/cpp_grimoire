## Item 41: Consider pass by value for copyable parameters that are cheap to move and always copied.
Some function parameters are intended to be copied. "Copy" in this discussion means to use it as the source of a copy or move operations. For example, a member function `addName` might copy its parameter into a private container. In C++11 `addName` will be copy constructed only for lvalue. For rvalues, it will be move constructed. 

With that been said, we can imagine what we currently have for `Widget` prototype and possible use cases:
```c++
class Widget {
public:
    void addName (??? newName);
    
private:
    std::vector<std::string> names;
};

Widget w;

std::string name("Bart");
w.addName(name); // call addName with lvalue

w.addName(name + "Jenne"); // call addName with rvalue
```

Firstly, can be used _overloading_. 
```c++
class Widget {
public:
    void addName(const std::string& newName)
    { names.push_back(newName); }
    
    void addName(std::string&& newName)
    { names.push_back(std::move(newName)); }
};
```
In this we create a function for copy lvalues and a function for move rvalue arguments. But it double things to maintain.

An alternative - _universal reference_
```c++
class Widget {
public:
    template<typename T>
    void addName(T&& newName)
    { names.push_back(std::forward<T>(newName)); }
};
```
As a template, `addName`’s implementation must typically be in a header file. It may yield several functions in object code, because it
not only instantiates differently for lvalues and rvalues, it also instantiates differently for `std::string` and types that are convertible to `std::string`. At the same time, there are argument types that can’t be passed by universal reference, and if clients pass improper argument types, compiler error messages can be intimidating.

Third option - _pass by value_. 
```c++
class Widget {
public:
    void addName(std::string newName)
    { names.push_back(std::move(newName)); }
};
```
The fact that there's only one `addName` function explains how we avoid code duplication, both in the source and the object code. We're not using a universal reference, so this approach doesn't lead to bloated header files, odd failure cases, or confounding error messages. But with some performance costs.

Consider the cost, in terms of copy and move operations, of adding a name to a `Widget`.
+ _Overloading_: Regardless of whether an lvalue or an rvalue is passed, the caller's argument is bound to a reference called `newName`. That cost nothing, in terms of copy and move operations. In the lvalue overload, `newName` is copied into `Widget::name`. In the rvalue overload, it's moved. Cost summary: one copy for lvalues, one move for rvalues.
+ _Using a universal reference_: as with overloading, the caller's argument is bound to the reference `newName`. This is a no-cost operation. Due to the use of `std::forward`, lvalue `std::string` arguments are copied into `Widget::names`, while rvalue `std::string` arguments are moved. The cost summary for `std::string` arguments is the same as with overloading: one copy for lvalues, one move for rvalues. If a caller passes an argument of a type other than `std::string`, it will be forwarded to a `std::string` constructor, and that could cause as few as zero `std::string` copy or move operations to be performed. Functions taking universal references can thus be uniquely efficient.
+ _Passing by value_: Regardless of whether an lvalue or an rvalue is passed, the parameter `newName` must be constructed. If an lvalue is passed, this costs a copy construction. If an rvalue is passed, it cists a move construction. In the body of the function, `newName` is unconditionally moved into `Widget::names`. The cost summary is thus one copy plus one move for lvalues, and two moves for rvalues. Compared to the by-reference approaches, that’s one extra move for both lvalues and rvalues.

Suggestion is "_Consider pass by value for copyable parameters that are cheap to move and always copied_".
1. _Consider_ - It's simple, single function, generates only one function in the object code. But it has a higher cost.
2. _for copyable parameters_ - Move-only types failing this test. For them only rvalue arguments need to be supported, and in that case, the “overloading” solution requires only one overload: the one taking an rvalue reference.
3. _cheap to move_ - When moves are cheap, the cost of an extra one may be acceptable.
4. _always copied_. This function incurs the cost of constructing and destroying `newName`, even if nothing is added to names. That’s a price the by-reference approaches wouldn’t be asked to pay.

When there are chains of function calls, each of which employs pass by value because “it costs only one inexpensive move,” the cost for the entire chain of calls may not be something you can tolerate.

If you have a function that is designed to accept a parameter of a base class type or any type derived from it, you don’t want to declare a pass-by-value parameter of that type, because you’ll “slice off” the derived class characteristics of any derived type object that may be passed in.

### Things to Remember 
+ For copyable, cheap-to-move parameters that are always copied, pass by value may be nearly as efficient as pass by reference, it's easier to implement, and it can generate less object code.
+ Copying parameters via construction may be significantly more expensive that coping them via assignment.
+ Pass by value is subject to the slicing problem, so it's typically inappropriate for base parameter types. 



## Item 42: Consider emplacement instead of insertion.
Let's take a look on
```c++
std::vector<std::string> vs;
vs.push_back("xyzzy");
```
Here's what happens at runtime:
1. A temporary `std::string` object is created from the string literal
2. That temporary object is passed to the rvalue overload `push_back`, where it's bound to the rvalue reference parameter `x`. A copy of `x` is then constructed in the memory for the `std::vector`. This construction is what actually creates a new object inside the `std::vector`(the constructor that's used to copy `x` into the `std::vector` is the move constructor).
3. Immediately after `push_back` returns, temporary `std::string` object is destroyed, thus calling the `std::string` destructor. 

`emplace_back` allow to eliminate that inefficiency: it uses whatever arguments are passed to it to construct a `std::string` directly inside the `std::vector` without temporaries. 

`emplace_back` is available for every standard container that supports `push_back`. Similarly, every standard container that supports `push_front` supports `emplace_front`. And every standard container that supports `insert` (which is all but `std::forward_list` and `std::array`) supports `emplace`. The associative containers offer `emplace_hint` to complement their `insert` functions that take a “hint” iterator, and `std::forward_list` has `emplace_after` to match its `insert_after`.

There are also situations where the insertion function run faster. Such situations are not easy to characterize, because they depend on the types of the arguments being passed, the container being used, the locations in the containers where insertion or emplacement is requested, the exception safety of the contained type's constructors, and, for containers where duplicate values are prohibited (i.e., `std::set`, `std::map`, `std:unordered_set`, `std::unordered_map`), whether the value to be added is already in the container. Benchmark both emplacement and insertion. 

If all the following are true, emplacement will almost certainly outperform insertion: 
+ **The value being added is constructed into the container, not assigned**. Node-based containers virtually always use construction to add new values, and most standard containers are node-based. Withing the non-node-based containers, you can rely on `emplace_back` to use construction instead of assignment. But take note, for example, with `std::vector<std::string> vs;` if there is already element and we emplace new object into a location already occupied by an object with `vs.emplace(vs.begin(), "xyzzy");`. For thus code, few implementations will construct the added `std::string` into the memory occupied by `vs[0]`. Instead, they'll move-assign the value into place, but move assignment requires an object to move from, which  mean that a temporary object will need to be created to be the source of the move. But what will be used depends on implementer. 
+ **The argument type(s) being passed differ from the type held by container**. Emplacement's advantage stems for the fact that its interface doesn't require creation and destruction of a temporary object when the argument(s) passed are of the type other than that held by the container. When an object of type `T` is to be added to `contaier<T>`, there's no reason to except emplacement to run faster than insertion, because no temporary needs to be created to satisfy the insertion interface. 
+ **The container is unlikely to reject the new value as a duplicate**. This means that the container either permits duplications or that most of the values you add will be unique. The reason this matter is that in order to detect whether a value is already in the container, emplacement implementations typically create a node with the new value so that they can compare the value of this node with existing container nodes. However, if the value is already present, the emplacement is aborted and the node is destroyed, meaning that the cost of its construction and destruction was wasted. Such nodes are created for emplacement function more often than for insertion functions. 

Sometimes creation of the temporary object is worth far more than it costs.
```c++
std::list<std::shared_ptr<Widget>> ptrs;
void customDeleter(Widget* pWidget);

ptrs.push_back(std::shared_ptr<Widget>(new Widget, customDeleter));
// or, similarly 
ptrs.push_back({ new Widget, customDeleter });
```
Consider the following potential sequence of events:
1. A temporary `std::shared_ptr<Widget>` object is constructed to hold the raw pointer resulting from `new Wdiget`. Call this object _temp_.
2. `push_back` takes _temp_ by reference. During allocation of a list node to hold a copy of _temp_, an out-of-memory exception gets thrown.
3. As the exception propagates out of `push_back`, _temp_ is destroyed, Being the sole `std::shard_ptr` referring to the `Widget` it's managing, it automatically releases that `Widget`, possibly via provided custom deleter. 
With `emplace_back`:
1. The raw pointer resulting from `new Widget` is perfect-forwarded to the point inside `emplace_back` where a list node is to be allocated. That allocation fails, and an out-of-memory exception is thrown. 
2. As the exception propagates out of `emplace_back`, the raw pointer that was the only way to get at the `Widget` on the heap is lost. That `Widget` (and any resources it owns) is leaked. 
Fundamentally, the effectiveness of resource-managing classes like `std::shared_ptr` and `std::unique_ptr` is predicated on resources (such as raw pointers from `new`) being immediately passed to constructors for resource-managing objects. The fact that functions like `std::make_shared` and `std::make_unique` automate this is one of the reasons they’re so important.
```c++
std::shared_ptr<Widget> spw(new Widget, customDeleter); 
ptrs.push_back(std::move(spw)); 

// emplace_back version is similar:
std::shared_ptr<Widget> spw(new Widget, customDeleter);
ptrs.emplace_back(std::move(spw));
```

Noteworthy aspect of emplacement functions is their interaction with `explicit` constructors.
```c++
std::vector<std::regex> regexes;

std::regex r1 = nullptr; // error, won't compile
std::regex r2(nullptr); // compiles

regexes.push_back(nullptr); // error, won't compile

regexes.emplace_back(nullptr); // compiles
```
The `std::regex` constructor taking a `const char*` pointer is`explicit`.
In the call to `emplace_back` we're not claiming to pass a `std::regex` object. Instead, we're passing a _constructor argument_ for a `std::regex` object. That's not considered and implicit conversion request, though it will compile, it has undefined behavior (use `nullptr` for `const char*` constructor).

In the official terminology of the Standard, the syntax used to initialize `r1` (with equal sign) corresponds to what is known as _copy initialization_. In contrast, the syntax used for `r2` (with the parentheses, although braces may be used instead) yields what is called _direct initialization_. Copy initialization is not permitted
to use `explicit` constructors. Direct initialization is.

Emplacement functions use direct initialization, which means they may use `explicit` constructors.

### Things to Remember 
+ In principle, emplacement functions should sometimes be more efficient than their insertion counterparts, and they should never be less efficient.
+ IN practice, they're most likely be faster when the value being added is constructed into the container, not assigned; the argument type(s) passed duffer from the type held by the container; and the container won't reject the value being added due to it being a duplicate. 
+ Emplacement functions may perform type conversions that would be rejected by insertion functions. 