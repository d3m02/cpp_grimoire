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
What's needed is to specify that when `data` is invoked on an rvalue `Widget`, the result should also be an rvalue. Using refernce qualifiers to overload `data` for lvalue and rvalue makes that possible 
```c++
DataType& data() & 
{ return values; }
DataType data() &&
{ return std::move(values); }
```

### Things to Remember
+ Declare overriding functions `override`.
+ Member function reference qualifiers make it possible to treat lvalue and rvalue objects (`*this`) differently. 



