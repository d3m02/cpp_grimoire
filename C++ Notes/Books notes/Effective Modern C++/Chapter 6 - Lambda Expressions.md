A _lambda expression_ is just that: an express. It's part of the source code 
```c++
std::find_if(container.begin(), container.end(),
    [](int val) { return 0 < val && val < 10; }); // <- expression
```
A _closure_ is the runtime object created by a lambda. Depending on the capture mode, closures hold copies of or references to the captured data. In the `std::find_if` above, the closure is the object that's passed at runtime as the third argument of `std::find_if`.

A _closure class_ is a class from witch a closure is instantiated. Each lambda causes compiler to generate a unique closure class. The statements inside a lambda become executable instructions in the member function of its closure class. 

A lambda is often used to create a closure that's used only as an argument to a function. 
> Closure class can be observed with [cppinsights.io](cppinsights.io):
```c++
int main()
{
    std::array <int, 2>  arr { 1, 2 }; 
    auto res = std::find_if(arr.begin(), arr.end(),
                            [](int val) { return val < 10; });
}
```
>transforms to 
```c++
int main()
{
  std::array<int, 2> arr = {{1, 2}};
      
  class __lambda_8_29
  {
  public: 
    inline bool operator()(int val) const
    { return val < 10; }
    
    using retType_8_29 = bool (*)(int);
    inline operator retType_8_29 () const noexcept
    { return __invoke; };
    
  private: 
    static inline bool __invoke(int val)
    { return __lambda_8_29{}.operator()(val); }
  };
  
  int* res = std::find_if(arr.begin(), arr.end(),  
                          __lambda_8_29(__lambda_8_29{}));
  return 0;
}
```
> name `__lambda_8_29` - 8 is line of code where lambda expression created, 29 - column in line. 


## Item 31: Avoid default capture modes. 
There are two default capture modes C++11: by-reference and by-value. 

A by-reference capture causes a closure to contain a reference to a local variable or to a parameter that's available in the scope where the lambda is defined. If the lifetime of a closure created from that lambda exceeds the lifetime of the local variable or parameter, the reference in the closure will dangle.
```c++
std::vector<std::function<bool(int)>> filters;

void addDivisorFilter()
{
    auto calc1 = computeSomeValue1();
    auto calc2 = computeSomeValue2();

    auto divisor = computeDivisor(calc1, calc2);
    filters.emplace_back(
        [&](int value) { return value % divisor == 0;}
    ); // danger, ref to divisor will dangle. 
}
```
The lambda refers to the local variable `divisor`, but that variable ceases to exist when `addDivisorFilter` returns. 

If you know that a closure will be used immediately(e.g., by being passed to an STL algorithm) and won't be copied, there is no risk that references it holds will outlive the local variables and parameters in the environment where its lambda is created. In that case, there's no risk of dangling references, hence no reason to avoid a default bt-reference capture mode. 

One way to solve problem above would be a default by-value capture mode. 
The problem is that if captured a pointer by value, pointer will be copied into the closures arising from the lambda, but it doesn't prevent code outside the lambda from `delete`ing the pointer and causing copy to dangle. 

Captures apply only to non-`static` local variables (including parameters) visible in the scope where the lambda is created. It can't capture members of class as well. 

To capture class member variable in lambda created in member function, `this` pointer can be used. Every non-`static` member function has a `this` pointer. With default capture lambda will capture `this` pointer instead `this->variable`. 
This means that the viability of the closures arising from such lambda is tied to the lifetime of the class whose `this` pointer they contain a copy of. This particular problem can be solved by making a local copy of the data member and then capturing the copy:
```c++
class Widget {
public:
    .. 
    void addFilter() const 
    {
        auto divisorCopy = divisor;
        filters.emplace_back(
            [divisorCopy](int value)
            { return value % divisorCopy == 0; }
        );
        // capture with [=](int value) will also work
    }
private:
    int divisor;
}
```

In C++14, a better way to capture a data member is to use generalized lambda capture, it's works in between lambda scope (left from '`=`') and local function scope ('`=`' from right) 
```c++
filters.emplace_back(
    [divisor = divisor](int value)
    { return value % divisor == 0; }
);
```

An additional drawback to default by-value captures is that they can suggest that the corresponding closures are self-contained and insulated from changes to data outside the closures. In general, that's not true, because lambdas may be dependent not just on local variables and parameters (which may be captured), but also on objects with _static storage duration_. Such objects are defined at global or namespace scope or are declared `static` inside classes, functions or files. These objects can be used inside lambdas, but they can't be captured. Yet specification of a default by-value capture mode lend the impression that they are. This can create some trick:
```c++
void addDivisorFilter()
{
    static auto calc1 = computeSomeValue1();
    static auto calc2 = computeSomeValue2();

    static auto divisor = computeDivisor(calc1, calc2);
    filters.emplace_back(
        [=](int value) { return value % divisor == 0; }
    ); //^-> even though we use copy    ^^^^^^^ -> not a copy
}
```

### Things to Remember 
+ Default by-reference capture can lead to dangling references.
+ Default by-value capture is susceptible to dangling pointers (especially `this` pointer), and it misleadingly suggest that lambdas are self-contained.



## Item 32: Use init capture to move objects into closures.
C++14 offers direct support for moving objects into closures, called _init capture_. Using an init capture makes it possible to specify 
1. _the name of a data member_ in the closure class generated from the lambda
2. _an expression_ initializing that data member.
```c++
auto func = [pw = std::move(pw)]
            { return pw->isValidated() 
                    && pw->isArchived(); }; 
```
To the left of the "`=`" is the name of the data member in the closure class to be specified, and to the right is initializing expression.

The scope of the left is that of the closure class. The scope on the right is the same as where the lambda is being defined. 

C++14 notation of "capture" is considerably generalized from C++11m because in C++11, it's not possible to capture the result of an expression. As a result, another name for init capture is _generalized lambda capture_.

Move capture can be emulated in C++11 by moving the object to be captured into a function object produced by `std::bind` and giving the lambda reference a reference to the "captured" object
```c++
std::vector<double> data;

// C++11 emulation
auto func = std::bind(
                [](const std::vector<double>& data)
                { /* uses of data */},
                std::move(data)
);

// equal to C++14
auto func = [data = std::move(data)]
            { /* uses of data */ };
```

Like lambda expressions, `std::bind` produce function objects, _bind objects_. 
A bind object contains copies of all the arguments passed to `std::bind`. For each lvalue argument, the corresponding object in the bind object is copy constructed. For each rvalue, it's move constructed. 
When a bind object is "called" (i.e., its function call operator is invoked) the argument it stores are passed to the callable object originally passed to `std::bind`. In this example, that means that when `func` (the bind object) is called, the move-costructed copy of `data` inside `func` is passed as an argument to the lambda that was passed to `std::bind`.

By default, the `operator()` member function inside the closure class generated from a lambda is `const`. That has effect of rendering all data members in the closure `const` within the body of the lambda. The move-constructed copy of a `data` inside the bind object is not `const`, however, so to prevent that copy of `data` from being modified inside the lambda, the lambda's parameter is declared reference-to-const. If the lambda were declared `mutable`, `operator()` in the closure class would not be declared `const`, and it would be appropriate to omit `const` in the lambda parameters declaration.

Because a bind object stores a copies of all the arguments passed to `std::bind`, the bind object contains a copy of the closure produced by the lambda that it its first argument. The lifetime of the closure is therefore the same as the lifetime of the bind object. That's important, because it means that as long as the closure exists, the bind object containing the pseudo-moved-captured object exists, too.

These fundamental points should be clear:
+ It's not possible to move-construct an object into a C++11 closure, but it is possible to move-construct an object into a C++11 bind object.
+ Emulating move-capture in C++11 consists of move-constructing an object into a bind object, then passing the move-constructed object to the lambda by reference.
+ Because the lifetime of the bind object is the same of the closure, it's possible to treat object in the bind object as if they were in the closure. 

### Things to Remember 
+ Use C++14's init capture to move objects into closures.
+ In C++11, emulate init capture via hand-written classes or `std::bind`.



## Item 33: Use `decltype` on `auto&&` parameters to `std::forward` them.
One of the most exciting feature if C++14 is _generic lambdas_ - lambdas that use `auto` in their parameter specifications. 
```c++
auto f = [](auto x) { return func(normalze(x)); }

// the closure class's function call opertor looks like 
class __lambda_6_10
{
public: 
    template<class type_parameter_0_0>
    inline auto operator()(type_parameter_0_0 x) const
    {
      return func(normalze(x));
    }  
};
```

Here lambda not written properly, because it always passes an lvalue (the parameter `x`) to `normilize`, even if the argument that was passed to the lambda was an rvalue.

The correct way to write the lambda is to have it perfect-forward `x` to `normalize`.

Normally, when we employ perfect forwarding, in a template function type parameter `T` is reused in `std::forward<T>`, however in the generic lambda there's no type parameter `T` available. 

If an lvalue argument is passed to a universal reference parameter, the type of that parameter becomes an lvalue reference. If an rvalue is passed, the parameter becomes an rvalue reference. This means that in our lambda, we can determine whether the argument passed was an rvalue or an lvalue by inspecting the type of the parameter. `decltype` gives us a way to do that. 
When calling `std::forward`, convention dictates that the type argument be an lvalue reference to indicate an lvalue and non-refernce to indicate an rvalue (due to [[Chapter 5 - Rvalue References, Move Semantics, and Perfect Forwarding#Item 28 Understand reference collapsing.|reference collapsing]]). In lambda, `x` is bound to an lvalue, `decltype(x)` will yield an lvalue reference, however if `x` is bound o an rvalue, `decltype(x)` will yield and rvalue reference instead of the customary non-reference. 
Instantiating `std::forward` with an rvalue reference type yields the same result as instantiating it with non-refernce type. That's means that `decltype(x)` yielding is suitable here 
```c++
auto f = [](auto&& param)
{ return func(normalize(std::forward<decltype(param)>(param))); };
```



## Item 34: Prefer lambdas to `std::bind`.
The most important reason to prefer lambdas over `std::bind` is that lambdas more readable. 
```c++
using Time = std::chrono::steady_clock::time_point;
using Duration = std::chrone::steady_clock::duration;
enum class Sound { Beep, Siren, Whistle };

void setAlarm(Time t, Sound s, Duration d);

auto setSoundL = 
    [](Sound s)
    {
    using namespace std::chrono;
    
    #if __cplusplus == 201103L // C++11 version

        setAlarm(steady_clock::now() + hours(1),
                s,
                seconds(30));

    #elif __cplusplus >= 201402L // since C++14
    
        using namespace std::literals;
        setAlarm(steady_clock::now() + 1h, 
                 s, 
                 30s);
    
    #endif
    };
```
`std::bind` alternative will looks something like 
```c++
using namespace std::chrono;

using namespace std::placeholders;

#if __cplusplus >= 201402L
using namespace std::literals;
#endif

auto setSoundB = 
#if __cplusplus == 201103L // C++11 version
    
    std::bind(setAlarm,
              std::bind(std::plus<steady_clock::time_point>(),
                        steady_clock::now(),
                        hours(1)),
                _1,
                seconds(30));
                
#elif __cplusplus >= 201402L // since C++14

    // In C++14 template type argumrnt for the standard operator templates can be ommitted
    std::bind(setAlarm,
             std::bind(std::plus<>(), steady_clock::now(), 1h),
             _1,
             30s);
             
#endif
```
From reader perspective, for a uninitiated the placeholder `_1` is essentially magic, but even readers who knows, they have to mentally map from the number in the placeholder to its position in the `std::bind` parameter list in order to understand that the first argument in a call to `setSoundB` is passed as the second argument to `setAlarm`. The type of this argument is not identified in the call to `std::bind`, so readers have to consult the `setAlarm` declaration to determine what kind of argument to pass to `setSoundB`. 

In the lambda, it's clear that the expression `stead_clock::now() + 1h` is an argument to `setAlarm`. In the `std::bind` call, however, it's passed as an argument to `std::bind`, not to `setAlarm`. That means that the expression will be evaluated when `std::bind` is called, and the time resulting from that expression will be stored inside the resulting bind object. Fixing the problem requires telling `std::bind` to defer evaluation of the expression until `setAlarm` is called, and that's why is second nested call to `std::bind` is for. `std::plus<>` is template from C++98 that simply return `static_cast<_Ty1&&>(_Left) + static_cast<_Ty2&&>(_Right);`

When `setAlarm` is overloaded, a new issue arises. The problem is that compilers have no way to determine which of the overload functions they should pass to `std::bind`. All they have is a function name, and the name alone is ambiguous. To get the `std::bind` call to compile, function must be cast to the proper function type 
```c++
using SetAlarm3ParamType = void(*)(Time t, Sound s, Duration d);
auto setSoundB = 
    std::bind(static_cast<SetAlarm3ParamType(setAlarm),
              std::bind(std::plus<>(),
                        steady_clock::now(),
                        1h),
              _1,
              30s);
```

`std::bind` always copies its arguments, but callers can achieve the effect of having an argument stored by reference by applying `std::ref` to it. All arguments passed to bind object are passed by reference, because the function call operator for such objects uses perfect forwarding.

_In C++14 there are no reasonable use cases for `std::bind`_. In C+11, however, `std::bind` can be justified in two constrained situations:
+ **Move capture**. C++11 lambdas don't offer move capture, but it can be emulated through a combination of a lambda and `std::bind`.
+ **Polymorphic function objects**. Because the function call operator on a bind object uses perfect forwarding, it can accept arguments of any type. This can be useful when you want to bind an object with a templatized function call operator.

### Things to Remember 
+ Lambdas are more readable, more expressive, and may be more efficient than using `std::bind`.
+ In C++11 only, `std::bind` may be useful for implementing move capture or for binding objects with templatized function call operators.