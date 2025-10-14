**Move semantics** makes it possible for compilers to replace expensive copying operations with less expensive moves. In the same way that copy constructors and copy assignment operators give control over what it means to copy objects, move constructors and move assignment operators control over the semantics of moving. Move semantics also enables the creation of move-only types. 
**Perfect forwarding** makes it possible to write function templates that take arbitrary arguments and forward them to other functions such that the target functions receive exactly the same arguments as were passed to the forwarding functions. 

`std::move` doesn't move anything, and perfect forwarding is imperfect. Move operations aren't always cheaper than copying; when they are, they're not always as cheap as expected; they're not always called in a context where moving is valid. The construct "`type&&`" doesn't always represent and rvalue reference. 

## Item 23: Understand `std::move` and `std::forward`.
`std::move` and `std::forward` are merely functions (function templates for precise) that perform casts.  `std::move` unconditionally casts its argument to an rvalue, while `std::forward` performs this cast only if a particular condition is fulfilled. 

`std::move` takes a reference to an object (a universal reference) and it returns a reference to the same object.
The "`&&`" part of the function's return type implies that `std::move` returns an rvalue refence, but, if type `T` happens to be an lvalue reference, `T&&` would become and lvalue reference. To prevent this from happening, the type trait `std::remove_reference` is applied to `T`, thus ensuring that "`&&`" is applied to a type that isn't a reference. That guarantees that `std::move` truly returns an rvalue reference, and that's important, because rvalue reference returned from functions are rvalues. 

```c++
class Annotation {
public:
    explicit Annotation (const std::string text)
        : values(std::move(text))
    { ... }
private:
    std::string value;
};
```
In this example `text` is not moved into `value`, it's copied. Sure, `text` is cast to and rvalue by `std::move`, but `text` is declared `const strd::string`, so before the cast, `text` is an lvalue `const st::string`, and the result of the cast is an rvalue `const std::string`, but throughout it all, the constness remains.
Consider the effect that has when compilers have to determine which `std::string` constructor to call. There are two possibilities: 
```c++
class string {
public:
    ...
    string(const string& rhs);
    string(string&& rhs);
    ...
};
```
The rvalue can't be passed to move constructor, because the move constructor an rvalue to non-const. Thus rvalue from `const std::string` will pass to copy constructor. 

There are two lessons to be drawn. First, don't declare objects `const` if move ability desirable. Second, `std::move` not only doesn't actually move anything, it doesn't even guarantee that the objects it's casting will be eligible to be moved. 

The story for `std::forward` similar to that for `std::move`, but whereas `std::move` _unconditionally_ casts its arguments to an rvalues, `std::forward` does it only under certain conditions, `std::forward` is a _conditional_ cast - it casts to an rvalue only if its argument was initialized with an rvalue. 

The most common scenario for `std::forward` use is a function template taking a universal reference parameters that is to be passed to another function:
```c++
void process(const Widget& lvalArg);
void process(Widget&& rvalArg);

template<typename T>
void logAndProcess(T&& param)
{
    makeLogEntry(/*...*/)
    process(std::forward<T>(param));
}
```
When we call function with lvalue, we naturally except that lvalue to be forwarded to `process` as lvalue, and when we call with an rvalue, we except the rvalue overload of `process` to be invoked. 
But `param`, like all function parameters, us an lvalue. Every call to `process` inside will thus want to invoke the lvalue overload for `process`. To prevent this, we need a mechanism for `param` to be cast to an rvalue of and only if the arguments with which `param` was initialized - was an rvalue. This is precisely what `std::forward` does. 

Given that both `std::move` and `std::forward` boil down to casts, the only difference being that `std::move` always casts, while `std::forward` only sometimes does. 

Technically, `std::forward` can to all what `std::move` does and thus can be used everywhere. Writing cast everywhere not a good idea, but beside `std::move`'s attractions are convenience, reduced likelihood of error and greater clarity. `std::move` requires only a function argument, while `std::forward` require both a function argument and a template type parameter. The type pass to `std::forward` should be a non-reference, because that's the convention for encoding that the argument being passed is an rvalue. Together, that means that `std::move` require less typing than `std::forward`, and it spares us the trouble of passing a type argument that encodes that the arguments we're passing is an rvalue. It also eliminates the possibility of our passing an incorrect type.

### Things to Remember 
+ `std::move` performs an unconditional cast to an rvalue. In and of itself, it doesn't move anything.
+ `std::forward` cast its arguments to an rvalue only if that argument is bound to an rvalue.
+ Neither `std::move` nor `std::forward` do anything at runtime. 



## Item 24: Distinguish universal references from rvalue references. 
"`T&&`" has two different meanings. One is rvalue reference. Such references bind only to rvalues, and their primary reason for being is to identify objects that may be moved from. 
The other meaning is _universal references_ (sometimes also called _forwarding references_ since in almost all cases `std::forward` applied to them). Such references looks like rvalue references in the source code, but they can behave as if they were lvalue references. They can bind to `const` or non-`const` objects, to `volatile` or non-`volatile` objects, they can bind to virtually anything. 

Universal references arise in two contexts. The most common is function template parameters, the second context is `auto` declarations.
```c++
template<typename T>
void f(T&& param); // universal reference

auto&& var2 = var1; // universal reference 
```

The initializer for a universal reference determines whether it represents an rvalue reference or an lvalue reference. The from of the reference declaration must also be correct, it must be precisely "`T&&`"; `const T&&` will be rvalue reference.

In template `T&&` not always universal references, because being in a template doesn't guarantee the presence of type deduction. For example, in `std::vector::push_back`
```c++
template<class T, class Allocator = allocator<T>>
class vector {
public: 
    void push_back(T&& x);
};
```
there's no type deduction in this case, `push_back` can't exist without particular `vector` instantiation for it to be part of, and the type of that instantiation fully determines the declaration for `push_back`. When `std::vector` instantiated with some time, it's clearly that `push_back` always declares a parameter of type rvalue-reference-to-T
```c++
std::vector<Widget> v;
// cause 
class vector<Widget, allocator<Widget>> {
public:
    void push_back(Widget&& x);
};
```

In contrast, `emplace_back` does employ type deductions - type parameter `Args` is independent of `vector`'s type parameter `T`, so `Args` must be deduced each time `emplace_back` is called.
```c++
template<class T, class Allocator = allocator<T>>
class vector {
public: 
    template <class... Args>
    void emplce_back(Args&&... args);
};
```

### Things to Remember
+ If a function template parameter has type `T&&` for a deduced type `T`, or if an object is declared using `auto&&`, the parameter or object is a universal reference. 
+ If the form of the type declaration isn't precisely _type&&_, or of type deduction doesn't occur, _type&&_ denotes an rvalue reference. 
+ Universal reference correspond to rvalue references if they're initialized with rvalues. They correspond to lvalue reference if they're initialized with lvalues. 



## Item 25: Use `std::move` on rvalue references, `std::forward` on universal references.
Rvalue references bind only to objects that are candidates for moving. A universal references on the other hand _might_ be bound to an object that's eligible for moving. Universal references should be cast to rvalues only if they were initialized with rvalues.

Rvalue references should be _unconditionally cast_ to rvalues (via `std::move`) when forwarding them to other functions, because they're always bound to rvalues, and universal references should be _conditionally cast_ to rvalues (via `std::forward`) when forwarding them, because they're _sometimes_ bound to rvalues. 

`std::forward` on rvalue refences can be made to exhibit the proper behavior, but the source code is wordy, error-prone, and unidiomatic, so using `std::forward` with rvalue references should be avoided. Even worse is the idea of using `std::move` with universal references, because that can have the effect of unexpectedly modifying lvalues (e.g., local variables):
```c++
class Widget {
public:
    template<typename T>
    void setName(T&& name) //universal reference
    { m_name = std::move(newName); }

private:
    std::string m_name;
}
std::string getWidgetName(); // factory function
Widget w;
auto n = getWidgetName();
w.setName(n); // moves n into w!
.. // n's value now unkown. 
```
Unconditionally cast its reference parameter to an rvalue, `n`'s value will be moved into `w.m_name`, and `n` will come back from the call to `setName` with an unspecified value. 

Typically classes implement two overloads for const lvalues and rvalue references. That would certainly work in this case, but there are drawbacks. First, it's more source code to write and maintain. Second, it can be less efficient. For example, `w.setName("Adela Noval")`, with version of `setName` taking a universal refence, the string literal would be conveyed to the assignment operator for the `std::string` inside `w`, `w`'s `m_name` would thus be assigned directly from the string literals; no temporary `std::string` object would arise. With the overload versions, a temporary `std::string` object would be created for `setName` parameter to bind to, and this temporary `std::string` would then be moved into `w`'s data member. A call to `setName` would thus entail execution of one `std::string` constructor, one `std::string` move assignment operator, and one `std::string` destructor. Replacing a template taking a universal reference with a pair of functions overloaded on lvalue and rvalue references is likely to incur a runtime cist in some cases. 

The most serious problem with overloading however, is poor scalability of the design. Some function, function templates actually, take an _unlimited_ number of parameters, each of which could be an lvalue or rvalue. For functions like these, overloading on lvalues and rvalues is not an option. 
```c++
template<class T, class... Args>
shared_ptr<T> make_shared(Args&&... args);
```

If function return _by value_ and returning an object bound to an rvalue refence or universal reference, better to apply `std::move` or `std::forward` when returning the reference. By casting to an rvalue in the `return` statement, object will be moved into the function's return value location. If the call to `std::move` were omitted, the fact that object can be an lvalue would force compilers to instead copy it into the return value location. If object does not support moving, casting it to an rvalue won't hurt, because the rvalue will simply be copied by copy constructor.
```c++
Matrix operator+(Matrix&& lhs, const Matrix& rhs)
{
    lhs += rhs;
    return std::move(lhs);
}
```

The situation is similar for universal references and `std::forward`. If the original object is an rvalue, its value should be moved into the return value, but if the original is an lvalue, an actual copy must be created. 

It was recognized long ago that the local object return copying can be avoided if construct it in the memory alloted for function's return value. This is known as the _return values optimization_. Compilers may elide the copying (or moving) of a local object in a function that returns by value if the type of the local object is the same as that returned by the function and the local object is what's being returned. Eligible local objects include most local variables as well as temporary objects created as part of a `return` statement. Function parameters don't qualify. Some people draw a distinction between application of the RVO to named and unnamed (i.e., temporary) local objects, limiting the term RVO to unnamed objects and calling its application to name objects _named return value optimization_ (NVRO).
```c++
Widget makeWidget()
{
    Widget w;
    ...
    return w; // RVO may be employed.
}
```
If use `return std::move(w)`, compilers probably won't use RVO to eliminate the move - they can't do it. RVO may be performed only if what's being returned is a local object, but `return std::move` instead return _reference_ to local object. 

If the conditions for the RVO are met, but compilers choose not to perform elision, the object being returned _must be threated as an rvalue_, meaning that `std::move` in this case is implicitly applied to local 
object being returned. 

### Things to Remember
+ Apply `std::move` to rvalue references and `std::forward` to universal references the last tine each is used.
+ Do the same thing for rvalue references and universal references being returned from function that return by value.
+ Never apply `std:move` or `std::forward` to local objects if they would otherwise be eligible for the return value optimization. 



## Item 26: Avoid overloading on universal references.
_Combining overloading and universal reference in almost a bad idea: the universal reference overload vacuums up far more argument type than the developer doing the overloading generally expects._ 

Suppose function that takes name as a parameter.
```c++
std::multiset<std::string> names;

void logAndAdd(const std::string& name)
{
    auto now = std::chrono::system_clock::now();
    log(now, "LogAndAdd");
    names.emplace(name);
}

std::string petName("Darls");

logAndAdd(petName);                      // pass lvalue
logAndAdd(std::string("Persephone"));    // pass rvalue 
logAndAdd("Patty Dog");                  // pass string literal
```
In the fist call `logAndAdd`'s parameter `name` is bound to the variable `petName`. Because `name` is an lvalue, it's copied into `names`, there's no way to avoid that copy. 
In the second call, parameter `name` is bound to an rvalue, the temporary `std::string` explicitly created from "`Persephone`". `name` itself is an lvalue, so it's copied into `names`, but in principle, its value could be moved into `names`, but in this call, we pay for a copy.
In the third call, the parameter `name` is again bound to rvalue, but this time it's to a temporary `std::string` that's implicitly created from "`Patty Dog`".  Had the string literal been passed directly to `emplace`, there would been no need to create a temporary `std::string` at all. 

We can eliminate the inefficiencies in the second and third calls by rewritting `logAndAdd` to take a universal reference and use `std::forward`:
```c++
template<typename T>
void logAndAdd(T&& name)
{
    auto now = std::chrono::system_clock::now();
    log(now, "LogAndAdd");
    names.emplace(std::forward<T>(name)); 
}
```
But if some clients doesn't have direct access to the `names`, only an index to look up. To support such clients, `logAndAdd` could be overloaded:
```c++
std::string nameFromIdx(int idx);

void logAndAdd(int idx)
{
    auto now = std::chrono::system_clock::now();
    log(now, "LogAndAdd");
    names.emplace(nameFromIdx(idx)); 
}
```
Suppose a client has a `short` holding an index and passes that to `logAndAdd` - this can cause error. There are two `logAndAdd` overloads. The one taking a universal reference can deduce `T` to `short`, thus yielding an exact match. The overload with an `int` parameter can match the `short` argument only with a promotion. Per the normal overload resolution rules, an exact match beats a match with a promotion, so the universal reference overload is invoked.
Within that overload, the parameter `name` is bound to the `short` that's passed in, `name` is then `std::forward`ed to the `emplace` member function on `names` (a `std::multiset<std::string>`), which, in turn, dutifully forward it to the `std::string` constructor. There is no constructor for `std::string` that takes a `short`. 

_Even worse case can create write a prefect forwarding constructor._
A small modification to the `logAndAdd` example demonstrate the problem. Instead of writing a free function that can take either a `std::string` and and index that can be used to look up, imagine a class `Person` with constructors that do the same thing
```c++
class Person {
public:
    template<typename T>
    explicit Person(T&& n)
        : name(std::forward<T>(n))
    {}

    explicit Person(int idx)
        : name(nameFromIdx(idx))
    {}

private:
    std::string name;
}
```
As was case with `logAndAdd`, passing an integral type other than `int` will call the universal reference constructor overload instead of the `int` overload, and that will lead to compilation failures. 

C++ will in addition generate both copy and move constructors, and this is true even if the class contains a templatized constructor that could be instantiated to produce the signature of the copy or move constructor. This leads to behavior that's intuitive only if you've spend to much time around compilers and compiler-writers, you've forgotten what it's like to be human: 
```c++ 
Person p("Nancy");
auto cloneOfP(p);
```
Here we're trying to create a `Person` from another `Person`, which seems like about as obvious a case for copy constructors. But this code won't call the copy constructor, it will call the perfect-forwarding constructor. That function will then try to initialize `Person`'s `std::string` member with a `Person` object. Compilers reason as follows. `cloneOfP` is being initialized with a non-`const` lvalue, and that means that the templatized constructor can be instantiated to take a non-`const` lvalue of type `Person`.

If we change the example slightly so that the object to be copied is `const`, it's now an exact match for the parameter taken by the copy constructor. The templatized constructor can be instantiated to have the same signature, but this doesn't matter, because _one of the overload-resolution rules in C++ is that in situations where a template instantiation and a non-template functions are equally good matches for a function call, the normal function is preferred_. The copy constructor thereby trumps an instantiated template with the same signature. 

The interaction among perfect-forwarding constructors and compiler-generated copy and move operation create even more troubles when inheritance enters the picture. The derived class copy and move constructors don't call their base class's copy and move constructors, they call the base class's perfect-forwarding constructor! Derived class function are using arguments of own type to pass to their base class. Ultimately, the code won't compile, because there's no `std::string` constructor taking a derived class own type when perfect-forwarding base template constructor instantiated.

_Overloading on universal reference parameters is something that should be avoid if at all possible_.

### Things to Remember 
+ Overloading on universal references almost always leads to the universal reference overload being called more frequently than expected.
+ Perfect-forwarding constructors are especially problematic, because they're typically better matches than copy constructors for non-`const` lvalues, and they can hijack derived class calls to base class copy and move constructors. 



## Item 27: Familiarize yourself with alternatives to overloading on universal refences. 
#### Abandon overloading
Avoid the drawbacks of overloading on universal reference by simply using different names for the would-be overloads.

#### Pass by `const T&`
Giving up some efficiency to keep things simple might be a more attractive trade-off than it initially appeared. 

#### Pass by value
An approach that often allow to dial up performance without any increase in complexity is to replace pass-by-reference parameters with pass by value. 

#### Use Tag dispatch
Neither pass by lvalue-reference-to-const nor pass by value offers support for perfect forwarding. If the motivation for the use of a universal reference is perfect forwarding, we have to use a universal reference. 
A universal reference parameter generally provides an exact match for whatever's passed in, but if the universal reference is part of a parameter list containing other parameters that are not universal references, sufficiently poor matches on the non-universal reference parameters can knock an overload with a universal reference out of the running. That's the basis behind the _tag dispatch_ approach. 

Rather than adding the overload, we'll make function to delegate to two other functions. For example, if `int` overload required, each function will also take a second parameter, one that indicated whether the argument being passed is integral with `std::is_integral<T>`. Important note that _if an lvalue argument is passed to the universal reference, the type deduced for `T` will be an lvalue reference (`int&`), which is not and integral type_. `std::remove_reference` also will required for this example
```c++
template<typename T>
void logAndAdd(T&& name)
{
    logAndAddImpl(std::forward<T>(name),
                  std::is_integral<typename std::remove_reference<T>::type>());
}

template<typename T>
void logAndAddImpl(T&& name, std::false_type)
{
    auto now = std::chrone::system_clock::now();
    log(now, "logAndAdd");
    names.emplace(std::forward<T>(name));
}

std::string nameFromIdx(int idx)
{ .. }
void logAndAddImpl(T&& name, std::true_type)
{
    logAndAdd(nameFromIdx(idx));
}
```
In C++14 can be used `std::remove_reference_t<T>` instead `typename std::remove_reference<T>::type`.

Conceptually, `logAndAdd` passes a boolean to `logAndAddImpl` indicating whether an integral type was passed to `logAndAdd`, but `true` and `false` are _runtime_ values, and we need to use overload resolution - a _compile-time_ phenomenon - to choose the correct `logAndAddImpl` overload. The call to the overloaded implementation functions inside `logAndAdd` "dispatches" the work to the correct overload by causing the proper tag object to be created. 

#### Constraining templates that take universal references
Compilers may generate copy and move constructors themselves, so even if only one constructor written and used tag dispatch within it, some constructor calls may be handled by compiler-generated function that bypass the tag dispatch system. The real problem is that they don't always pass it by.
Providing a constructor taking a universal reference causes the universal reference constructor (rather than the copy constructor) to be called when copying non-const lvalues. When a base class declares a perfect-forwarding constructor, that constructor will typically be called when derived classes implement their copy and move constructors in the conventional fashion, even though the correct behavior is for the base class's copy and move constructors to be invoked. 

`std::enable_if` gives a way to force compilers to behave as if a particular template didn't exist. By default, all templates are "enabled", but a template using `std::enable_if` is enabled only if the specified condition is satisfied. In out case, we'd like to enable perfect-forwarding constructor in the type being passed isn't class own type, since in otherwise case we want to handle passed argument with copy or move constructor. What makes `std::enable_if` here is technology called [SFINAE](https://en.cppreference.com/w/cpp/language/sfinae). 
Take in mind that type deduction for a universal reference initialized with an lvalue is always an lvalue reference. When we say that the templatized constructor should be enabled only if `T` isn't a class, we'll realize that when we're looking at `T`, we want to ignore
+ _Whether it's a reference._ For the purpose of determining whether the universal reference constructor should be enabled for class `Person`, the types `Person`, `Person&` and `Person&&` are all the same as `Person`.
+ _Whether it's const or volatile_. A `const Person`, a `volatile Person` and a `const volatile Person` are all the same as a `Person`.
This means we need to strip any references, `const`s and `volatile`s from `T` before passing to `std::enable_if`. That is what `std::decay` is for.
`!std::is_same<Person, typename std::decay<T>::type>::value`

Last thing to consider is inheritance, passing `SpecialPerson` type inhered from `Person` isn't same as `Person` even after application of `std::decay`. We also don't want to enable the templatized constructor for any argument type other than `Person` or _a type derived from_ `Person`.
Among the standard type traits there is one that determines whether one type is derived from another, `std::is_base_of<T1, T2>::value`.

There always one more things - we start from troubles with overloading. All we need to do - add a `Person` constructor overload to handle integral arguments.

Tying all together,
```c++
class Person {
public:
    template<
      typename T,
      typename = typename std::enable_if<
                    !std::is_base_of<Person, 
                                     typename std::decay<T>::type
                     && !std::is_integral<std::remove_refernce<T>::type
                     >::value
                    >::value
                 >::type
    > 
    explicit Person(TT& n)
        : name(std::forward<T>(n))
    { .. }

    explicit Person(int idx) // Although, passing size_t variable will not work
        : name(nameFromIdx idx)
    { .. }
    ...
};
```
In C++14 we can simplify it a little bit
```c++
class Person {
public:
    template<
      typename T,
      typename = std::enable_if_t<
        !std::is_base_of<Person, std::decay_t<T>>::value
         && !std::is_integral<std::remove_reference_t<T>>::value
      >        
    >
    explicit Person(TT& n);
    ...
};
```

> In C++17 also possible to use `std::is_base_of` and `is_integral_v`, with C++20 it's possible to replace SFINAE with more readable [concepts library](https://en.cppreference.com/w/cpp/concepts)
```c++
class Person {
public:
    template<typename T>
    requires (!std::derived_from<std::decay_t<T>, Person> 
              && !std::integral<std::remove_reference_t<T>>)
    explicit Person(T&& n);
};

// or furthermore
template<typename T>
concept ValidPersonArg = 
    !std::derived_from<std::decay_t<T>, Peson>
    && !std::integral<std::remove_reference_t<T>>;

class Person {
public:
    template<ValidPersonArg T>
    explicit Person(T&& n);
}
```

#### Trade-offs
As a rule, perfect forwarding is more efficient, because it avoids the creation of temporary objects solely for the purpose of conforming the type of a parameter declaration. 
But perfect forwarding has drawbacks. One is that some kinds of arguments can't be perfect-forwarded, even though they can be passed to functions taking specific types. 
A second issue is the comprehensibility of error messages when clients pass invalid arguments. The more times universal reference is forwarded, the more baffling the error message may be when something goes wrong. Even if add message via compile-time test through `static_assert` and `std::is_constructible` type trait, like 
```c++
class Person {
    template<
      typename T,
      typename = std::enable_if_t<
        !std::is_base_of<Person, std::decay_t<T>>::value
         && !std::is_integral<std::remove_reference_t<T>>::value
      >        
    >
    explicit Person(TT& n)
        : name (std::forward<T>(n))
    {
        static_assert(
            std::is_constructible<std::string, T>::value,
            "Parameter n can't be used to construct a std::string"
        );
        ...
    }
};
```
error message appears only _after_ the usual error message, since `static_assert` is in the body of the constructor, but the forwarding code is being part of the member initialization list, precedes it. 

### Things to Remember
+ Alternatives to the combination of universal references and overloading including the use of distinct function names, passing parameters by lvalue-reference-to-const, passing parameters by value, and using tag dispatch.
+ Constraining template via `std::enable_if` permits the use of universal references and overloading together, but it control the conditions under which compilers may use the universal reference overloads.
+ Universal reference parameters often have efficiency advantages, but they typically have usability disadvantages.  



## Item 28: Understand reference collapsing.
When an argument is passed to a template function, the type deduced for the template parameter encodes whether the argument is an lvalue or an rvalue. This happens only when the argument is used to initialize a parameter that's a universal reference, but there's a good reason for the omission: universal references. 

The encoding mechanism is simple. When an lvalue is passed as an argument, `T` is deduced to be an lvalue reference. When an rvalue is passed, `T` is deduced to be a non-refernce. (Note the asymmetry: lvalue are encoded as lvalue references, but rvalues are encoded as non-refrences.). It's also the underlying mechanism through which `std::forward` dies it work. 

References to references are illegal in C++.

When an lvalue is passed to a function template taking a universal reference, 
```c++ 
template<typename T>
void func(T&& param);
```
if we take the type deduced for `T` (i.e., `Widget&`) and use it to instantiate the template, we get `void func(Widget& && param)`, a reference to a reference!
How does the compiler get from the result of taking the deduced type for `T` and substituting it into the template to the following, which is the ultimate function signature? The answer is _reference collapsing_. When compilers generate references to references, reference collapsing dictates what happens next.

If a reference to a reference arises in a context where this is permitted (e.g., during template instantiation), the reference _collapse_ to a since reference according to this rule:
_If either reference is an lvalue reference, the result is an lvalue reference. Otherwise (i.e., if both are rvalue references) the result is an rvalue reference_.

Reference collapsing is a key part of what makes `std::forward` work. `std::forward`'s job is to cast an lvalue to an rvalue if and only if `T` encodes that the argument passed to function was an rvalue, i.e., if `T` is a non-reference type. 
```c++
template<typename T>
T&& forward(typename 
             remove_reference<T>::type& param)
{ return static_cast<T&&>(param); }
```
Suppose that the argument passed to template function `f` with universal reference parameter is an lvalue of type `Widget`. Plugging `Widget&` into the `std::forward` implementation yields 
```c++
Widget& && forward(typename
                     remove_refernce<Widget&>::type& param)
{ return static_cast<Widget& &&>(param); }
```
The type trait `std::remove_refernce<Widget&>::type` yields `Widget`, so `std::forward` becomes
```c++
Widget& && forward(Widget& param)
{ return static_cast<Widget& &&>(param); }
```
Reference collapsing is also applied to the return type and the cast, and the final result 
```c++
Widget& forward(Widget& param)
{ retun static_cast<Widget&>(param); }
```
So, casting `Widget&` has no effect, an lvalue argument passed to `std::forward` will thus return an lvalue reference. 
If an rvalue of type `Widget` passed to `f`, the deduced type for `f`'s parameter `T` will be simply `Widget`. The call inside `f` to `std::forward` will thus be `std::forward<Widget>`
```c++
Widget&& forward(typename 
                  remove_reference<Widget>::type& param)
{ return static_cast<Widget&&>(param); }

// Applying std::remove_refernce
Widget&& forward(Widget& param)
{ return static_cast<Widget&&>(param); }
```
there are no references to references here, so there's no reference collapsing, and this is the final instantiated version of `std::forward` for the call.

A universal references isn't a new kind of references, it's actually an rvalue reference is a context where two conditions are satisfied: 
+ _Type deduction distinguishes lvalue from rvalues._ Lvalues of the type `T` are deduced to have type `T&`, while rvalues of type `T` yields `T` as their deduced type.
+ _Reference collapsing occurs_.

Reference collapsing occurs in four contexts.
+ The first and most common is template instantiation. 
+ The second is type generation for `auto` variables, the details are essentially the same as for templates, because type deduction for `auto` variable is essentially the same as type deduction for templates. 
+ The third is the generation and use of `typedef`'s and alias declarations. If, during creation or evaluation of a `typedef`'s, refences to references arise, reference collapsing intervenes to eliminate them.
+ The final context in which reference collapsing takes place is uses of `decltype`. If, during analysis of a type involving `decltype`, a reference to a reference arises, reference collapsing will kick in to eliminate it. 

### Thing to Remember 
+ Reference collapsing occurs in four contexts: template instantiation, `auto` type generation, creation and use of `typedef`s and alias declarations, and `decltype`.
+ When compilers generate a reference to a reference in a reference collapsing context, the result becomes a single reference. If either of the original references is an lvalue reference, the result is an lvalue reference. Otherwise, it's an rvalue reference. 
+ Universal references are rvalue references in context where type deduction distinguishes lvalues from rvalues and where reference collapsing occurs. 



## Item 29: Assume that move operations are not present, not cheap, and not used. 
It would be a mistake to assume that moving all containers is cheap. For some containers, this is because there's not truly cheap way to move their contents. For others, it's because the truly cheap move operations the containers offer come with caveats the container elements can't satisfy. 

Consider `std::array`, it's essentially a built-in array with an STL interface. This is fundamentally different from the other standard containers (such `std::vector`), each of which stores its contents on the heap. Objects of such container type hold (as data members), conceptually, only a pointer to the heap memory staring the contents of the container. (the reality is more complex, but for purposes of this analysis, the differences are not important). The existence of this pointer makes it possible to move the contents of an entire container in constant time: just copy the pointer to the container's contents from the source container to the target, and set the source's pointer to null.
`std::aray` objects lack such a pointer, because the data for a `std::array`'s contents are stored directly in the `std::array` objects. Assuming that content is a type where moving is faster than copying, moving a `std::array` will be faster than copying. Yet both moving and copying a `std::array` have liner-time computational complexity, because each element in the container must be copied or moved. 

On the other head, `std::string` offers constant-time moves and linear-time copies. That makes it sound like moving is faster than copying, but that may not be the case. Many string implementations employ the _small string optimization_ (SSO). With the SSO, "small" strings (e.g., those with a capacity of no more than 15 characters) are stored in a buffer withing the `std::string` objects; no heap-allocated storage is used. Moving small string using an SSO-based implementation is no faster than copying them, because the copy-only-a-pointer trick that generally underlies the performance advantage of moves over copies isn't applicable.

Some container operations in the STL offer the strong exception safety guarantee and that to ensure that legacy C++98 dependent on that guarantee isn't broken when upgrading to C++11, the underlying copy operations may be replaces with move operations only if the move operations are known to not throw. 

There are thus several scenarios in which C++11's move semantics do you no good:
+ **No move operations**: The object to be moved from fails to offer move operations. The move request therefore becomes a copy request. 
+ **Move not faster**: The object to be moved from has move operations that are no faster than its copy operations. 
+ **Move not usable:** The context in which the moving would take place require a move operation that emits no exceptions, but that operation isn't declared `noexcept`
+ **Source objects is lvalue**: With very few exception only rvalues may be used as source of a move operation 

Often, however, the types code uses are known, and it possible to rely on their characteristics not changing (e.g., whether they support inexpensive move operations). When that's the case, you don't need to make assumption, simply look up the move supports details for the types, if those types offer cheap move operations and objects is been used in contexts where those move operations will be invoked, it's safe to rely on move semantics to replace copy operations with their less expensive more counterparts.    

### Things to Remember 
+ Assume that move operations are not present, not cheap, and no used.
+ In code with known types or support for move semantics, there is no need for assumptions. 



## Item 30: Familiarize yourself with perfect forwarding failure cases.
"Forwarding" just means that one function passes, _forwards_, its parameters to another function. The goal is for the second function (the one being forwarded to) to receive the same objects that the first function received. That rules out by-value parameters, because they're copies of what the original caller passed in. Pointer parameters are also ruled out, because we don't want to force callers to pass pointers. 

_Perfect forwarding_ means we don't just forward objects, we also forward their salient characteristics: their types, whether they're lvalues or rvalues, and whether they're `const` or `volatile`. 

Forwarding functions are generic. A logical extension of this genericity is for forwarding function to be _variadic_ templates, accepting any number of arguments
```c++
template<typename... Ts>
void fwd(Ts&&... params)
{
    f(std::forward<Ts>(params)...);
}
```

Given our target function `f` and our forwarding function `fwd`, perfect forwarding _fails_ if calling `f` with a particular arguments does one thing, but calling `fwd` with the same argument does something different. 

Several kind of arguments lead to this kind of failure. 

#### Braced initializers
Suppose `f` is declared like `void f(const std::vector<int>& v)`.
In that case, `f({ 1, 2, 3 });` compiles, but `fwd({ 1, 2, 3 });` doesn't compile, because the use of braced initializer is a perfect forwarding failure case.

All such failure cases have the same cause. In a direct call to `f`, compilers see the arguments passed at the call site, and they see the types of the parameters declared by `T`. They compare the arguments at the call site to the parameter declarations to see if they're compatible, and, if necessary, they perform implicit conversation. When calling `f` indirectly through the forwarding function template `fwd`, compilers no longer compare the arguments passed at `fwd`'s call site to the parameter declarations in `f`, instead the _deduce_ the types of the arguments being passed to `fwd`, and they compare the deduced types to `f`'s parameter declarations. Perfect forwarding fails when either of the following occurs:
+ **Compilers are unable to deduce a type**.
+ **Compilers deduce the "wrong" type.** One source of such divergent behavior would be if `f` were an overloaded function name, and, due to "incorrect" type deduction, the overload of `f` called inside `fwd` were different from the overload that would be invoked if `f` were called directly.

Compilers are forbidden from deducing a type for the experession `{ 1, 2 , 3 }`, because `fwd`'s parameter isn't declared to be a `std::initializer_list`.

Interestingly, type deduction succeed for `auto` variables initialized with a braced initializer. Such variables are deemed to be `std::initializer_list` objects, and this affords a simple workaround for cases where the type the forwarding function should deduce is a `std::initializer_list` - declare a local variable using `auto`, then pass the local variable to the forwarding function. 

#### `0` or `NULL` as null pointers
When you try to pass `0` or `NULL` as null pointer to a template, type deduction goes awry, deducing an integral type. The result is that neither `0` nor `NULL` can be perfect-forwarded as a null pointer. 

#### Declaration-only integral `static const` data members
As a general rule, there's no need to defined integral `static const` data member in classes; declarations alone suffice. That's because compilers performs _const propagation_ on such member's value, thus eliminating the need to set aside memory for them.
```c++
class Widget {
public:
    static const std::size_t MinVals = 28; // declaration
};
.. //no defn. for MinVals

std::vector<int> wudgetData;
widgetData.reserve(Widget::MinVals); // use of MinVals
```
The fact that no storage has been set aside for `MinVals`' value in unproblematic. If `MinVals`' address were to be taken (e.g., if somebody created a pointer to `MinVals`), then `MinVals` would require storage (so that the pointer had something to point to), and the code above, though it would compile, would fail at link-time until a definition for `MinVals` was provided. 
With that in mind, `f(Widget::MinVals)` will compile like `f(28)`, while `fwd(Widget::MinVals)` shouldn't link. 

Passing integral `static const` data members by reference generally requires that they be defined, and that requirement can cause code using perfect forwarding to fail where the equivalent code without perfect forwarding succeeds. 

According to the Standard, passing `MinVals` by reference requires that it be defined. But not all implementations enforce his requirement. Simply provide a definition for the integral `static const` data member in question. Note that definition doesn't repeat the initializer

#### Overloaded function names and template names 
Suppose our function `f` (the one we keep wanting to forward argument to via `fwd`) can have its behavior customized by passing it a function that does same of its work.
```c++
void f(int (*pf)(int)); // pf = "processing function"

int processVal(int value);
int processVal(int value, int priority);

f(processVal)    // fine
fwd(processVal); // error
```
Compilers know which `processVal` they need: the one matching `f`'s parameter type. They thus choose the `processVal` taking on `int`, and pass the function's address to `f`.
What makes this work, is that `f`'s declaration lets compilers figure out which version of `processVal` is required. `fwd`, however, being a function template, doesn't have any information about what type it needs, and that makes it impossible for compilers to determine which overload should be passed. Without a type, there can be no type deduction, and without type deductions, we're left with another perfect forwarding failure case. 

The same problem arises if we try to use a function template instead of (or in addition to) an overloaded function name. 
```c++
template<typename T>
T workOnVal(T param)
{ ... } 

fwd(workOnVal); // error, which instantiation?
```

The way to get a perfect-forwarding function template to accept an overloaded function name or a template name is to manually specify the overload or instantiation to have forwarded. 
```c++
using ProcessFuncType = int (*)(int);

ProcessFuncType processValPtr = processVal;

fwd(processValPtr);
fwd(static_cast<ProcessFuncType>(workOnVal));
```

#### Bitfields
The final failure case for perfect forwarding is when a bitfield is used as a function argument. 
```c++
struct IPv4Header {
    std::uniunt32_t version:4,
                    IHL:4,
                    DSCP:6,
                    ECN:2,
                    totalLength:16;
};

void f(std::size_t sz);

IPv4Header h;
f(h.totalLength);   //fine
fwd(h.totalLength); //error
```
The problem is that `fwd`'s parameter is a reference, and `h.totalLength` is a non-const bitfield. The C++ Standard condemns that "_A non-const reference shall not be bound to a bit-field_". Bitfields may consist of arbitrary parts of machine words (e.g., bits 3-5 of a 32-bit `int`), but there's no way to directly address such things. 

Working around the impossibility of perfect-forwarding a bitfield is easy, once you realize that any function that accepts a bitfield as an argument will receive a _copy_ if of the bitfield's value.
```c++
auto length = static_cast<std::uinit16_t>(h.totalLength);
fwd(length);
```

### Things to Remember 
+ Perfect forwarding fails when template type deduction fails or when it deduces the wrong type.
+ The kinds of arguments that lead to perfect forwarding failure are brace initializers, null pointers as `0` or `NULL`, declaration-only integral `const static` members, template and overloaded functions names, and bitfields. 