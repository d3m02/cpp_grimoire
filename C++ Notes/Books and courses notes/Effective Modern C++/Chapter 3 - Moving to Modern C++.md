## Item 7: Distinguish between `()` and `{}` when creating objects. 
For built-in types like `int`, the difference is academic, but for user-defiend types, it's important to distinguish initialization from assignment, because different function calls are involved: 
```c++
Widget w1;      // call default constructor
Widget w2 = w1; // not an assignment; calls copy ctor
w1 = w2;        // an assignment; calls copy operator=
```

As a general rule, initialization values may be specified with parentheses, an equal sign, or braces:
```c++
int x(0);
int y = 0;
int z { 0 };
int w = { 0 };
int v = (0);
```

To address the confusion of multiple initialization syntaxes, as well as fact that they don't cover all initialization scenarios, C++11 introduced _uniformal initialization:_ a single initialization syntax that can, at least in concept, be used anywhere and express everything. It's based on braces.

Braces initialization can be used for specifying the initial contents of container - 
```c++
std::vector<int> v{ 2 }; // v will initialized with 2, [2]
std::vector<int> v(2);   // call constructor with specified size, value [0 0]
```

Braces can also be used to specify default initialization values for non-static data members. This capability is shared with "=" initialization syntax, but not with parentheses:
```c++
class Widget { 
..
private:
    int x { 0 };
    int y = 0;
    int z(0); // error
}
```

Uncopyable objects (e.g., `std::atomic`s) may be initialized using braces or parentheses, but not using "=":
```c++
std::atomic<int> ai1 { 0 };
std::atomic<int> ai2(0);
std::atomic<int> ai3 = 0; // error
```

Brace initialization prohibits _implicit narrowing conversions_ among build-in types. 

Braced initialization is immune to C++'s _most vexing parse_ - a side effect of C++'s rule that anything that can be pared as a declaration must be interpreted as one, the most vexing parse most frequently afflicts developers when they want to default-construct an object, but inadvertently end up declaring a function instead. 
```c++
Widget w1(10); //call Widget ctor with argument 10
Widget w2(); // declares a function named w2 that returns a Widget
```

The drawback to braced initialization is the sometimes-surprising behavior from relationship among braced initializers, `std::initializer_lists` and constructor overload resolution. 

If one or more constructors declare a parameter of type `std::initializer_list`, calls using the brace initialization syntax strongly prefer the overloads taking `std::initializer_list`. Even if best-match `std::initializer_list` constructor can't be called.
```c++
class Widget {
public:
    Widget(int i, double d);
    Widget(std::initializer_list<bool> il);
};
Widget w{10, 5.0}; //error - requires narrowing conversions.
```
Only if there's no way to convert the types of the arguments in a braced initializer to the type of a `std::initializer_list` (for example, `std::initializer_list<std::string>` and `Widget w{10, 5.0};`) do compilers fall back on normal resolution. 

If required to call a `std::initializer_list` constructor with an empty `std::initializer_list`, it can be done by putting empty braces inside the parentheses or braces:
```c++
Widget w4({});
Widget w5 {{}};
```

Constructors with `std::initializer_list` also require from class designing perspective aware that if set of overloaded constructors includes one or more functions taking a `std::initializer_list`, client code using braced initialization may see only the `std::initializer_list` overload. As a result, it's best to design constructors so that the overload called isn't affected by whether clients use parentheses or braces. An implication is that if class have no `std::initializer_list` constructor, after one was add - client code using braced initialization may find that calls that used to resolve non-`std::initializer_list` constructor now resolve to the new function. 

### Things to Remember 
+ Braced initialization is the most widely usable initialization syntax, it prevents narrowing conversions, and it's immune to C++'s most vexing parse.
+ During constructor overload resolution, braced initializers are matched to `std::initializer_list` parameters if at all possible, even if other constructors offer seemingly better matches.
+ An example of where the choice between parentheses and braces can make a significant difference is creating a `std::vector<numberic type>` with two arguments 
+ Choosing between parentheses and braces for object creation inside templates can be challenging. 



## Item 8: Prefer `nullptr` to `0` and `NULL`
First note - neither `0` nor `NULL` has a pointer type. Passing `0` or `NULL` never called a pointer overload. In C++98, the primary implication of this was that overloading on pointer. Counterintuitive behavior "I'm calling `f` with `NULL` - the null pointer" is actually "I'm calling `f` with some kind of integer - not the null pointer", is what leads to the guideline from C++98 to avoid overloading on pointer and integral types. 

If `NULL` is defined to be, say, `0L`, the call is ambiguous, because conversation from `long` to `int`, `long` to `bool`, and `0L` to `void*` are considered equally good.

`nullptr`'s advantage is that it doesn't have an integral type, it's actual type is `std::nullptr_t`. This type implicitly converts to all raw pointer types. 

Using `nullptr` instead of `0` or `NULL` thus avoid overload resolution surprises. It also can improve code clarity, especially when `auto` variables are involved (`auto result = findRecord(input);` `if (result == 0)` vs `if (result == nullptr)`)

`nullptr` especially useful with templates. Template type deduction can deduces "wrong" type for `0` and `NULL` when requires to refer to a null pointer. With `nullptr` template pose no special challenge with that.

### Things to Remember
+ Prefer `nullptr` to `0` and `NULL`.
+ Avoid overloading on integral and pointer types.



## Item 9: Prefer alias declarations to `typedef`s
Using STL containers with smart pointers from STL sometime can create needs to use types like `std::unique_ptr<std::unordered_map<std::string, std::string>>`. Such constructed types can be reduced with `typedef`:
```c++
typedef 
   std::unique_ptr<std::unordered_map<std::string, std::string>>
   UPtrMapSS;
```

C++11 also offers _alias declarations_: 
```c++
using UPtrMapSS =
   std::unique_ptr<std::unordered_map<std::string, std::string>>;
```

Alias declaration can be used for easier dealing with types involving function pointers:
```c++
typedef void (*FP)(int, const std::string&);
using FP = void (*)(int, const std::string&);
```

Alias declarations may be templatized (in which case they're called _alias templates_), while `typedef`s cannot. 
```c++
template<typename T>
using MyAllocList = std::list<T, MyAlloc<T>>;

MyAllocList<Widget> lw;
```
With a `typedef` it's 
```c++
template<typename T>
struct MyAllocList {
    typedef std::list<Tm MyAlloc<T>> type;
};

MyAllocList<Widget>::type lw;
```

It case when required to use the `typedef` inside a template for the purpose of creating a linked list holding objects of a type specified by a template parameter, `typedef` name needs to precede with `typename`:
```c++
tempalte<typename T>
class Widget {
private:
    typename MyAllocList<T>::type list;
};
```
Here, `MyAllocList<T>::type` refers to a type that's dependent on a template type parameter (T). `MyAllocList<T>::type` is thus a _dependent type_, and one of C++'s many endearing rules is that the names of dependent types must be preceded by `typename`. 
If `MyAllocList` is defined as an alias template, this need for `typename` vanishes (as does the "`::type` suffix):
```c++
tempalte<typename T>
using MyAllocList = std::list<T, MyAlloc<T>>;

tempalte<typename T>
class Widget {
private:
   MyAllocList<T> list;
};
```

In contrast with `typedef`, which is dependent type, alias template is _non-dependent type_, and `typename` specifier is neither required nor permitted. 

Alias template also allow to avoid such coalition when `type` can be interpreted as class member
```c++
class Wine {};
template<>
class MyAllocList<Wine>{
private:
    enum class WineType
    { White, Red, Rose };

    WineType type;
};
```
If `Widget` were to be instantiated with `Wine`, `MyAllocList<T>::type` inside the `Widget` template would refer to a data member, not a type. Inside the `Widget` template, then, whether `MyAllocList<T>::type` refers to a type is honestly dependent on what `T` is, and that's why compilers insist on developer asserting that it is a type by preceding it with `typename`.

C++11 give the tools to perform transformations with types like strip off `const`, refernce-qualifiers, add `const` or turn into lvalue reference etc via _type traits_ assortment of templates inside the header `<type_traits>`. Given a type `T` the resulting type is `std::transformation<T>::type`, for example
```c++
std::remove_const<T>::type         // yields T from const T
std::remove_refernce<T>::type      //yields T from T& and T&&
std::add_lvalue_reference<T>::type // yilds T& from T
```
In C++14 additionally added aliases for each C++11 transformation named `std::transformation_t`:
```c++
std::remove_const_t<T>
std::remove_reference_t<T>
std::add_lvalue_refernce_t<T>
```
Writing such alias templates quite easy using:
```c++
template<class T>
using remove_const_t = typename remove_const<T>::type;
```

### Things to Remember
+ `typedef`s don't support templatization, but alias declarations do.
+ Alias templates avoid the "`::type`" suffix and, in templates, the "
  `typename`" prefix often required to refer to `typedef`s.
+ C++14 offers alias templates for all the C++11 type traits transformations. 



## Item 10: Prefer scoped enums to unscoped enums.
Names in C++98-style `enum`s belong to the scope containing the `enum` and that means that nothing else in that scope may have the same name. The fact that these enumerator names leak into the scope containing their `enum` definition gives rise to the official term for this kind of `enum`: _unscoped enums_. 
```c++
enum Class { black, white, red };
auto white = false; // error, white already declared in this scope
```

In C++11 introduced their counterparts, _scoped enums_, which uses syntax `enum class`.

Firstly, scoped `enum` don't leak names like unscoped does.

Enumerators for unscoped `enum` implicitly convertible to integral types. Scoped `enum` doesn't have implicit conversions from enumerators to any other type.
```c++
enum Color { black, white, red };
Color c = red; 
if (c < 14.5) // compare Color to double (!?)

...

enum class Color { black, white, red };
Color c = Color::red;
if (static_cast<double>(c) < 14.5) //odd, but it's valid
```

Scoped `enum` may be forward-decalred, i.e. their names may be declared without specifying their enumerators:
```c++
enum Color; //error 
enum class Color; // fine
```
In C++11 unscoped `enum` may also be forward-declrated, but only after a bit of additional work. Every `enum` in C++ has an integral _underlying_ type that is determined by compilers. Compiler might chose `char` underlying type, if `enum` contain much larger value (e.g. `0xFFFFFFFF`) compiler probably select an integral type larger than `char`. Compilers often want to choose the smallest underlying type to make efficient use of memory, to make that possible, C++98 supports only `enum` definitions (where all enumerators are listed) and not allowing `enum` declarations. In C++11 underlying type for unscoped `enum` can be specify, enabling forward declaration (while for scoped `enum` underlying type is always known, is default to `int`):
```c++
enum Color : std::uint8_t;
```

The most notable drawback of inability to forward-declare `enums` is compilation dependencies. If new value is introduced, it's likely that entire system when `enum` used will have to be recompiled, even if only a single subsystem, possibly only a single function, uses the new enumerator. Ability to specify underlying type (in other words, make forward declaration) is eliminate such case. 

There's at least one situation where unscoped `enum` maybe be useful. That's when referring to fields withing C++11's `std::tuples`
```c++
enum UserInfoFields { uiName, uiEmail, uiReputation };
using UserInfo = std::tuple<std::string, std::string, std::size_t>;

UserInfo uInfo;
auto val = std::get<uiEmail>(uInfo);
```

> Note, in C++17 this changed with using structured binding (from C++26 unused variables can be replaced with placeholder variables `_`) or `std::tie`
```c++
auto [_, val, _] = uInfo;
//or 
std::string val; 
std::tie(std::ignore, val, std::ignore) = uInfo;
```

### Things to Remember 
+ C++98-style `enum` are now known as unscoped `enum`
+ Enumerator of scoped `enum` are visible only withing the `enum`. They convert to other types only with a cast.
+ Both scoped and unscoped `enum` support specification of the underlying type. The default underlying type for scoped `enum` is int. Unscoped `enum` have no default underlying type.
+ Scoped `enum` may always be forward-declrated. Unscoped `enum` may be forward-declared only if their declaration specifies an underlying type.



## Item 11: Prefer deleted functions to private undefined ones. 
From C++98 declaring functions as `private` prevents clients from calling them, Deliberately failing to define them means that if code still has access to them (i.e., member functions or `friend` of the class) uses them, linking will fail due to missing function definitions. 
```c++
template <class charT, class traits = char_traits<charT> >
class basic_ios : public ios_base {
public: 
    ...
private:
    basic_ios(const basic_ios&);            // not defined
    basic_ios& operator=(const basic_ios&); // not defined
};
```

In C++11, there's better way to achieve essentially the same end: use `= delete`. Deleted function may not be used in any way, so even code that's in member and `friend` function will fail to compile. 

By convention, deleted functions are declared `public`, not `private`. When client code tries to use a member function, C++ checks accessibility before deleted status. When client code tries to use a deleted `private` function, some compilers complain only about the function being `private`, even though the function's accessibility doesn't really affect whether it can be used.

An important advantage of deleted functions is that any function may be deleted, while only member functions may be private. For example, this can be used to disable implicit conversation for `int`:
```c++
bool isLucky(int number);
bool isLucky(char) = delete;
bool isLucky(bool) = delete;
bool isLucky(double) = delete;
```
In this example deleting overload for `double` also guard from `float` overload since given a choice between converting a `float` to an `int` or to a `double`, C++ prefers the conversation to `double`.

`delete` also can prevent use of template instantiations that should be disabled. 
```c++
template<typename T>
void processPointer(T* ptr);

template<>
void processPointer<void>(void*) = delete;
template<>
void processPointer<char>(char*) = delete;
```

If a function template inside a class need some instantiations need to be disabled, declaring them as `private` will not work since it's not possible to give a member function template specialization a different access level from that of the main template. The problem is that template specializations must be written at namespace scope, not class scope. This issues doesn't arise for deleted function, because they don't need a different access level.
```c++
class Widget {
public:
    template<typename T>
    void processPointer(T* ptr)
    { .. }
};

template<>
void Widget::processPointer<void>(void*) = delete;
```

### Things to Remember 
+ Prefer deleted functions to private undefined ones.
+ Any function may be deleted, including non-member functions and template instantiations. 



## Item 12: Declare overriding function `override`.
For overriding to occur, several requirements must be met:
+ The base class function must be virtual.
+ The base and derived function name must be identical (except in the case of destructors).
+ The parameter types of the base and derived functions must be identical.
+ The constness of the base and derived functions must be identical.
+ The return types and exception specifications of the base and derived functions must be compatible. 
To these constraints, which were also part of C++98, C++11 adds one more:
+ The functions' _reference qualifiers_ must be identical. Reference qualifiers make possible to limit use of a member function to lvalue only or to rvalue only. 
```c++ 
class Widget {
public:
    void doWork() &; 
              //  ^ only when *this is an lvalue
    void doWord() &&;
               // ^^ only when *this is an rvalue 
};
```

Code containing overriding errors is typically valid, but its meaning isn't what might be expected. 

In C++11 was added `override` _contextual keyword_ to make explicit that a derived class function is supposed to override a base class version. 

A policy of using `override` on all derived class overrides can do more than just enable compilers to tell when would-be overrides aren't overriding. It can also help gauge the ramifications if contemplating changing the signature of a virtual function on a base class: changing base function signature will brake compilation for all derived functions marked as `override`. 

C++11 also introduce another contextual keyword `final`. Applying `final` to a virtual function prevents the function from being overridden in derived classes. `final` may also be applied to a class, in which case the class is prohibited from being used as a base class. 
Both keywords `final` and `override` have the characteristic that they are reserved, but only in certain context. In the case of `override`, it has a reserved meaning only when it occurs at the end of a member function declaration. That means that if some legacy code already uses name override (for function name, for example), it will not brake anything after enabling C++11. 

The need for reference-qualified member function is not common, but it can arise. For example
```c++
class Widget {
public:
    using DataType = std::vector<double>;
    DataType& data() { return values; }
private:
    DataType values;
};
Widget w;
auto vals1 = w.data(); //copy w.values into vals1
```
The return type of `Widget::data` is an lvalue reference and because lvalue references are defined to be lvalues, `vals1` initializing from an lvalue - `vals1` is thus copy constructed from `w.values`.
Now suppose factory function that create `Widgets` and we want to initialize a variable with the `std::vector` inside the `Widget` returned from this factory function:
```c++
Widget makeWidget();
auto vals2 = makeWidget().data();
```
`Widgets::data` returns an lvalue reference, lvalue reference is an lvalue, so, `vals2` is copy constructed from `values` inside `Widget`. This time `Widget` is temporary object returned from `makeWidget` (i.e. rvalue) which mean that copying `std::vector` inside it is waste of time. It'd be preferable to move it, but, because `data` returning an lvalue reference, the rules of C++ require that compilers generate code for a copy. (There's some wiggle room for optimization through what is known as the "as if rule", but it's might be foolish to rely on compilers finding way to take advantage of it). 
What's needed is to specify that when `data` is invoked on an rvalue `Widget`, the result should also be an rvalue. Using reference qualifiers to overload `data` for lvalue and rvalue makes that possible 
```c++
DataType& data() & 
{ return values; }
DataType data() &&
{ return std::move(values); }
```

### Things to Remember
+ Declare overriding functions `override`.
+ Member function reference qualifiers make it possible to treat lvalue and rvalue objects (`*this`) differently. 



## Item 13: Prefer `const_iterator`s to `iterator`s.
`const_iterator`s are the STL equivalent of pointers-to-const - they point to values that may not be modified.

In C++98 `const_iterator`s had only halfhearted support: it wasn't that easy to create them, the way of use were limited. For example, in case when `std::find` used to find iterator for insertion in `std::vector`, `std::find` return just `iterator` but since `insert` of `std::vector` doesn't modifies what an iterator points to, it's reasonable to use `const`. Conceptually this approach suitable, though still not correct:
```c++
typedef std::vector<int>::iterator IterT;
typedef std::vector<int>const_iterator ConstIterT;

std::vector<int> values;
ConstIter ci = std::find(static_cast<ConstIterT>(values.begin()),
                         static_cast<ConstIterT>(values.end()),
                         1983);
values.insert(static_cast<IterT>(ci), 1998);
```

The cast in the call to `std::find` are present because values in a non-const container and in C++98, there was no simple way to get a `const_iterator` from non-const container. The cast aren't strictly necessary, because it was possible to get `const_iterator` in other ways (e.g. bind to a reference-to-const variable, then use that variable), but the process of getting `const_iterator`s to elements of non-const container involved some amount of contorting. Since in C++98 locations for insertions (and erasures) could be specified only by iterators and `const_iterator` weren't acceptable backward cast required as well. At the end, this code might not compile because there's no portable conversion from `const_iterator` to an `iterator`, and example here to show troubles with `const_iterator` in C++98 that cause possible avoidance of `const_iterators`. 

From C++11 `const_iterator` can be easily get from container member functions `cbegin` and `cend`, STL uses more `const_iterator`:
```c++
std::vector<int> values;
auto it = std::find(values.cbegin(), values.cend(), 1983);
values.insert(it, 1998);
```

About the only situation in which C++11's support for `const_iterator`s come up a little lacking is when required to write maximally generic library code, which takes into account that some containers and conteiner-like data structures offer `begin` and `end` (plus `cbegin`, `cend`, `rbegin`, etc) as non-member functions, rather than members. Maximally generic code this uses non-member functions rather than assuming the existence of member versions, but in C++11 added non-member functions only for `begin` and `end`. In C++14 this situation mostly fixed and `cbegin`, `cend` also available. 
```c++
template<typename C, typename V>
void findAndInsert(C& container, 
                   const V& targetVal, 
                   const V& insertVal)
{
    using std::cbegin;
    using std::cend;

    auto it = std::find(cbegin(container), cend(container), targeVal);
    container.insert(it, insertVal);
}
```
For C++11 it might require implement own implementation for `cbegin` and `cend`:
```c++
template <class C>
auto cbegin(const C& container)->decltype(std::begin(container))
{
    return std::begin(container);
}
```
This template accepts any type of argument representing a container-like data structure and it access this argument through its reference-to-const parameter `container`. In `C` is a conventional container type (e.g., a `std::vector<int>`), container will be a reference to a const version of that container (e.g., a `const std::vector<int>&`). Invoking the non-member `begin` function in C++11 on a `const` container yields a `const_iterator`, and that iterator is what this template returns. The advantage of implementing thing this way is that it works even for containers that offer a `begin` member function, but fail to offer a `cbegin` member. 

### Thing to Remember 
+ Prefer `const_iterator` to `iterator`
+ In maximally generic code, prefer non-member version of `begin`, `end`, `cbegin`, etc., over their member function counterparts.



## Item 14: Declare functions `noexcept` if they won't emit exceptions. 
In C++98 when function's implementation was modified, the exception specification might require revision, changing an exception specification could break client code, because callers might be dependent on the original exception specification.

During work on C++11, a consensus emerged that the truly meaningful information about a function's exception-emitting behavior was whether function might emit an exception or guaranteed that it wouldn't. That's what `noexcept` is designed for. As such, whether a function is `noexcept` is as important a piece of information as whether a member function is `const`. Failure to declare a function `noexcept` when it's know that it won't emit an exception is simply poor interface specification. 

Consider a function `f` that promises callers they'll never receive an exception. There are two ways of expressing this:
```c++
int f(int x) throw();
int f(int x) noexcept; // since C++11
```
If, at runtime, an exception leaves `f`, `f`'s exception specification is violated. With `throw` exception specification, the call stack is unwound to `f`'s caller, and, after some actions not relevant here, program execution is terminated. With the C++11 exception specification, runtime behavior is slightly different: the stack is only possibly unwound before execution is terminated. In a `noexcept` function, optimizers need not keep the runtime stack in an unwindable state if an exception would propagate out of the function, nor must they ensure that objects in a `noexcept` function are destroyed in the inverse order of construction should an exception leave the function. Functions with `throw` exception specifications lack such optimization flexibility, as do functions with no exception specification at all.

When a new element is added to a `std::vector`, it's possible that the `std::vector` lacks space for it, i.e., that the `std::vector`'s size is equal to its capacity. When that happens, the `std::vector` allocates a new, larger, chunks of memory to hold its elements, and it transfers the elements from the existing chunks of memory to the new one. In C++98, the transfer was accomplished by copying each element from the old memory to the new memory, then destroying the objects in the old memory. This approach enabled `push_back` to offer the strong exception safety guarantee: if an exception was thrown during the copying of the elements, the state of the `std::vector` remained unchanged, because none of the elements in the old memory were destroyed until all elements had been successfully copied into the new memory.
In C++11, a natural optimization would be to replace the copying with moves. Unfortunately, doing this runs the risk of violating `push_back`'s exception safety guarantee. If n elements have been moved from the old memory and an exception is thrown moving element n+1, the `push_back` operation can't run to completion, but the original `std::vector` has been modified: n of its elements have been moved from. Restoring their original state may not be possible, because attempting to move each object back into the original memory may itself yield an exception. 
This is a serious problem, because the behavior of legacy code could depend on `push_back`'s strong exception safety guarantee. Therefore, C++11 implementation can't silently replace copy operations inside `push_back` with moves unless it's known that move operation won't emit exception. In that case, having moves replace with copies would be safe, and the only side effect would be improved performance. 
`std::vector::push_back` takes advantage of this "move if you can, but copy if you must" strategy, checking if the operation declared `noexcept`. The checking is typically rather roundabout. Functions like `std::vector::push_back` call `std:move_if_noexcept`, a variation of `std::move` that conditionally cast to an rvalue depending on whether the type's move constructor is `noexcept`. In turn, `std::move_if_noexcept` consults `std::is_nothrow_move_constructible`, and the value of this type trait is set by compilers, based on whether the move constructor has a `noexcept` (or `throw()`) designation.

`swap` function is another case where `noexcept` is particularly desirable. Interestingly, whether swaps in the STL are `noexcept` is sometimes dependent on whether user-defined swaps are `noexcept`
```c++
template<class T, size_t N>
void swap(T (&a)[N],
          T (&b)[N]) noexcept(noexcept(swap(*a, *b)));
template <class T1, class T2>
struct pair { 
..
void swap(pair& p) noexcept(noexcept(swap(first, p.first)) &&
                            noexcept(swap(second, p.second)));
};
```
These functions are _conditionally noexcept_: whether they are `noexcept` depends on whether the expressions inside the `noexcept` clauses are `noexcept`.

If function declared `noexcept` and later regret that decisions, options are bleak. Removing `noexcept` from the function's declaration (i.e., change its interface) can cause risk of breaking client code. Function implementation might be changes in such way that an exception could escape, yet keeping the original (now incorrect) exception specification - in this case program will be terminated if an exception tries t o leave the function. 

Most functions are _exception-neutral_. Such functions throw no exceptions themselves, but functions they call might emit one. When that happens, the exception-neutral function allow the emitted exception to pass through on its way to a handler further up the call chain. Exception-neutral functions are never `noexcept`, because they may emit such "just passing through" exceptions. 

If a straightforward function implementations might yield exception, hide that from callers (e.g., catching all exceptions and replacing them with status codes or special return values) and declaring function then as `noexcept` - will not only complicated function's implementation, it will typically complicate code at all call sites too. 
For example, callers may have to check for status codes or special return values. The runtime cost of those complications (e.g., extra branches, larger functions that put more pressure ion instruction caches, etc.) could exceed any speedup that was hoped to achieve via `noexcept`, plus code will be more difficult to comprehend and maintain. 

By default, all memory deallocation functions and all destructors, both user-defined and compiler-generated - are implicitly `noexcept`. The only time a destructor is not implicitly `noexcept` is when a data member of  the class is of a type that expressly states that its destructor may emit exceptions (e.g., declares it `noexcept(false)`). 

It's worth nothing that some library interface designers distinguish functions with _wide contracts_ from those with _narrow contracts_. A function with wide contract has no preconditions. Such a function may be called regardless of the state of the program, and it imposes no constraints on the arguments that callers pass it. Functions with wide contracts never exhibit undefined behavior. "Regardless of the state of the program" and "no constraints" doesn't legitimize programs whose behavior is already undefined. For example, `std::vector::size` has a wide contract, but that doesn't require that it behave reasonably if it's invoked on random chunk of memory that was casted to `std::vector`. The result of the cast is undefined, so there are no behavioral guarantees for the program containing the cast. 

Functions without wide contracts have narrow contracts. For such functions, if a precondition is violated, results are undefined. For such function declaring `noexcept` can be tricky. For example, if function `f` has a precondition: the length of its `std::string` parameter doesn't exceed 32 characters, and if function marked `noexcept` - it makes difficult to detect precondition violation, since throwing exception would lead to program termination. For this reason, library designers who distinguish wide from narrow contracts generally reserve `noexcept` for functions with wide contracts. 

Compilers typically offer no help in identifying inconsistencies between function implementations and their exception specifications. For example, if function declared `noexcept`, even though it calls the non-`noexcept` functions. 

### Things to Remember. 
+ `noexcept` is part of a function's interface, and that means that callers may depend on it.
+ `noexcept` functions are more optimizable than non-`noexcept` functions.
+ `noexcept` is particularly valuable for the move operations, swaps, memory deallocation functions and destructors.
+ Most functions are exception-neutral rather than `noexcept`.



## Item 15: Use `consexpr` whenever possible.
When `constexpr` applied to objects, it's essentially a beefed-up form to `const`. Conceptually, `constexpr` indicates a value that's not only constant, it's known during compilation. `const` doesn't offer the same guarantee as `constexpr`, because `const` objects need not be initialized with values during compilation. 

Integral values that are constant and known during compilation can be used in contexts where C++ requires an _integral constant expression_. Such contexts include specification of array sizes, integral template arguments (including lengths of `std::array` objects), enumerator values, alignment specifiers, and more. 

`constexpr` functions produce compile-tiem constants _when they called with compile-time constants_. If the values of the arguments passed to `constexpr` functions known during compilation, the result will be computed during compilation. When a `constexpr` function is called with one or more values that are not known during compilation, they acts like a normal functions, computing its result at runtime. 

In C++11 `constexpr` functions may contain no more than a single executable statement: a `return`. This doesn't count conditional "?:" operator, also recursion can be used instead of loops.
```c++
constexpr int pow(int base, int exp) noexcept
{
    return (exp == 0 ? 1 : base * pow(base, exp - 1));
}
```
In C++14, the restrictions on `constexpr` functions are substantially looser. 

`constexpr` functions are limited to taking and returning literal types, which essentially means types that can have values determined during compilation. In C++11, all build-in types except `void` qualify, but user-defiend types my be literal, too, because constructors and other member functions may be `constexpr`. For such cases, constructor cab be declared `constexpr`, because if the arguments passed to it are known during compilation, the value of the data members of the constructed object can also be known during compilation, getters can be `constexpr`, because if they're invoked on object with a value known during compilation, the values of the data members can be known during compilation. In C++11 setter functions can't be declared `constexpr` - first, they modify the object they operate on, and in C++11, `constexpr` member functions are implicitly `const`, second, they have `void` return types. Both restrictions are lifted in C++14, so in C++14 setters also can be `constexpr`.

> Note, with newer C++ standards restriction and requirements for `constexpr` actively changing: for example from C++17 `constexpr` available with static data member declaration; from C++20 virtual functions might be used in `constexpr` context, `try`/`catch` can be used with some limitations, `new`/`delete` allowed in `constexpr` functions, `constexpr` destructors; in C++23 more STL algorithm and `std::vector` available in `constexpr` context; in C++26 planned `constexpr` for exceptions, structed bindings and references to `constexpr` variables

### Things to Remember
+ `constexpr` objects are `const` and are initialized with values known during compilation.
+ `constexpr` functions can produce compile-time results when called with arguments whose values are known during compilation.
+ `constexpr` objects and functions may be used in a wider range of contexts than non-`constexpr` objects and functions.
+ `constexpr` is part of an object's or function's interface. 



## Item 16: Make `const` member functions thread safe.
One catchy word - `mutable`. Even if function is `const` (promised to not change member variables), reading operations without synchronization are (supposed to be) safe - if for example `const` function change some `mutable` variable for caching - it's might become thread unsafe
```c++
class Polynomial {
public:
    using RootType = std::vector<double>;

    RootType roots() const 
    {
        if (!rootAreValid)
        {
            ... //compute roots, store them in rootVals
            rootsAreValid = true;
        }
      
        return rootVals;
    }
private:
    mutable bool rootsAreValid { false };
    mutable RootsTyope rootVals {};
}
```
Here if function `roots` used with multiple threads - date race might accure. 

The easiest way to fix is employ a mutex, which also in this case will be `mutable`, but since `std::mutex` is a move-only type, a side effect of adding `std::mutex` is that class loses the ability to be copied. 

In some situations, a mutex is overkill and `std::atomic` will solve situation and often be a less expensive than mutex acquisition and release. 

For a single variable or memory location requiring synchronization, use of `std::atomic` is adequate, but dealing with two or more variables or memory location that require manipulation as a unit, probably should be used mutex. 
```c++
class Widget {
public: 
    int magicValue() const
    {
        if (cacheValid)
            return cachedValue;
          
        auto val1 = expensiveComputation1();
        auto val2 = expensiveComputation2();
        cachedValue = val1 + val2; // !
        cacheValid = true; // !
        return cachedValue;
    }
private:
    mutable std::atomic<bool> cacheValid { false };
    mutable std::atmoic<int> cacheValue;
}
```
What can happen here - first thread sees `cacheValid` is false, perform two expensive computations and assigns their sum to `cachedValue`; at that point second thread also sees `cacheValid` as false, also carries same expensive computation that the first thread has just finished. 
Swapping assignment for `cachedValue` and `cahceValid` can make situation even worse since second thread can find that `cachedValid` is true and return `cachedValue` even if first thread has not yet made an assignment. 

### Things to Remember
+ Make `const` member functions thread safe unless you're certain they'll never be used in a concurrent context.
+ Use of `std::atomic` variables may offer better performance than a mutex, but they're suited for manipulation of only a single variable or memory location.



## Item 17: Understand special member function generation. 
C++98 has for _special member functions_ that might be generated by compilers: the default constructor, the destructor, the copy constructor, and the copy assignment operator. These functions are generated only if they're needed, i.e. if some code uses them without their being expressly declared in the class. A default constructor is generated only if the class declares no constructors at all. Generated special member functions are implicitly public and `inline`, and they're nonvirtual unless the function in question is a destructor in a derived class inheriting from a base class with a virtual destructor.
> Virtualness and inlining remains consistent across modern C++ standards. 

As of C++11, the special member functions was extended with move constructor and move assignment operator. The rules governing their generation and behavior are analogies to those for their copy siblings. The move operations are generated only if they're needed, and if they are generated, they perform "memberwise moves" on the non-statiic data members of the class. There is no guarantee that a move will actually take place, it's move request and "move" for some fields can be done via copy operations.

_The two copy operations are independent_: declaring one doesn't prevent compiler from generating the other. So if copy constructor declared, but no copy assignment operator, then write code that requires copy assignment - compilers will generate the copy assignment operator. 

_The two move operations are not independent_. Declared a move constructor prevents a move assignment operator from being generated, and declaring a move assignment operator prevents compilers from generating a move constructor.

Furthermore, _move operations won't be generated for any class that explicitly declares a copy operation_. The justification is that declaring a copy operation (construction or assignment) indicates that the normal approach to copying an object isn't appropriate. _Copy operations won't be generated for any class that explicitly declare a copy operation_. The copy operations are disabled by deleting them. 

_User-defined destructor had no impact on compilers' willingness to generate copy operations_, even though exist "Rule of Three" that suggests that if a class declares explicitly a destructor, copy operations should also explicitly declare, since non-default destructor probably means some kind of resource management. Although, since C++11 generating copy assignment operator (copy constructor) when declared user-defined destructor or copy constructor (copy assignment operator) is deprecated. 
> Note: still, in C++26 it's just marked as deprecated. Some compilers may yields warnings about generating deprecated copy operators.   

_C++11 does not generate move operations for a class with a user-decleared destructor_. 

So move operations are generated for classes (when needed) only if these three things are true:
+ No copy operations are declared in the class.
+ No move operations are declared in the class.
+ No destructor is declared in the class.

C++11 also provide `= default` keyword that allow to say compiler to generate member function even if it's automatic generation disabled (at least, compiler can try to generate). This approach is often useful in polymorphic base classes, since normally they have virtual destructors. Unless a class inherits a destructor that's already virtual, the only way to make a destructor virtual is to explicitly declare it that way. User-declared destructor suppresses generation of the move operations, declaring the move operations disables the copy operations. In this case `= default` can be handy.
```c++
class Base {
public:
    virtual ~Base() = default;
    // support moving
    Base(Base&&) = default;
    Base& operator=(Base&&) = default;
    // support copying
    Base(const Base&) = default;
    Base& operator=(const Base&) = default;
};
```

Also worth mentioning move via copy cases. Suppose that in source code exists class with only constructor, compiler will generate all 5 member functions. If such class used for creating objects that in serval places are moving, adding in such class destructor might looks like braking compilation (since automatic generating of move operations after declaring destructor will be disabled), but since destructor declare doesn't disable copy operations generating, code therefore likely will compile, run, and pass its functional testing - move requests will be done via copying. 

The C++11 riles governing the special member functions are thus:
+ **Default constructor**: Generated only if the class contains no user-declared constructors.
+ **Destructor**: virtual only if a base class destructor is virtual, `noexcept` by default.
+ **Copy constructor**: Memberwise copy construction of non-static data members in runtime. Generated only if the class lacks a user-declared copy constructor. Deleted if the class declares a move operation. Generation of this function in a class with a user-declared copy assignment operator or destructor is deprecated.
+ **Copy assignment operator**: Memberwise copy assignment of non-static data members in runtime. Generated only if the class lacks a user-declared copy assignment operator. Deleted if the class declares a move operation. Generation of this function in a class with user-declared copy constructor or destructor is deprecated. 
+ **Move constructor** and **Move assignment operator**: Each performs memberwise moving request of non-static data members. Generated only if the class contains no user-declared copy operations, move operations, or destructor. 

Note that there's nothing in the rules about the existence of a member function _template_ preventing compilers from generating the special member functions. Compiler will generate copy and move operations (assuming the usual conditions governing their generation are fullflled), even though these templates could be instantiated to produce the signatures for the copy constructor and copy operator.
```c++
class Widget {
    ..
    template<typename T>
    Widget(const T& rhs);

    template<typename T>
    Widget& operator=(const T& rhs);
}
```

### Things to Remember
+ The special member functions are those compilers may generate on their own: default constructor, destructor, copy operations, and move operations.
+ Move operations are generated only for classes lacking explicitly declared move operations, copy operations, and a destructor.
+ The copy constructor is generated only for classes lacking an explicitly declared copy constructor, and it's deleted if a move operation is declared. The copy assignment operator is generated only for classes lacking an explicitly declared copy assignment operator, and it's deleted if a move operation is declared. Generation of the copy operations in classes with an explicitly declared destructor is deprecated. 
+ Member function templates never suppress generation of special member functions. 