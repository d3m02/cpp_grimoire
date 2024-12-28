## Item 1: Understand template type deduction.
Deduction for templates is the bases for one the modern C++'s most compelling features: `auto`.

In templates like 
```c++
template<typename T> 
void f(ParamType param);

f(expr); // Call f with some expression
```
during compilation, compilers use passed `expr` to deduce two types: one for `T` and one for `ParamType`. These types are frequently different, because `ParamType` often contains adornments, e.g., `const` or reference qualifiers.

There are three cases:
#### Case 1: `ParamType` is a Reference or Pointer, but not a Universal Reference. 
```c++
template<typename T>
void f1(T& param);

template<typename T>
void f2(const T& param);

template<typename T>
void f3(T* param);

int x = 27;
const int cx = x;
const int& rx = x;
const int* px = &x;
```

If `expr` type is a reference - ignore the reference part. 

|           | passed type | `T` type  | `ParamType` type |
| --------- | ----------- | --------- | ---------------- |
| `f1(x);`  | int         | int       | int&             |
| `f1(cx);` | const int   | const int | const int&       |
| `f1(rx);` | const int&  | const int | const int&       |
| `f2(x);`  | int         | int       | const int&       |
| `f2(cx);` | const int   | int       | const int&       |
| `f2(rx);` | const int&  | int       | const int&       |
| `f3(x);`  | int         | int       | int*             |
| `f3(px)`  | const int*  | const int | const int*       |
These examples all show _lvalue_ reference parameters, but type deduction works exactly the same way for _rvalue_ reference parameters. 
Because in `f2` `param` now a reference-to-const, there's no longer a need for `const` to be deducted as part of T.
For pointer (or a pointer to const), things would work essentially like for reference. 

#### Case 2: `ParamType` is a Universal Reference
Universal reference parameters are declared like rvalue references, but they behave differently when lvalue arguments are passed in.
```c++
template<typename T>
void f(T&& param);

int x = 27;
const int cx = x;
const int& rx = x;
```

If `expr` is an lvalue, both `T` and `ParamType` are deduced to be lvalue references. This is only situation in template type deduction where `T` is deducted to be a reference. Although `ParamType` is declared using the syntax for an rvalue reference, its deduced type is an lvalue reference. 
If `expr` is an rvalue, the "normal" (i.e., Case 1) rules apply. 

|          | passed type | `T` type   | `ParamType` type | Category |
| -------- | ----------- | ---------- | ---------------- | -------- |
| `f(x);`  | int         | int&       | int&             | lvalue   |
| `f(cx);` | const int   | const int& | const int&       | lvalue   |
| `f(rx);` | const int&  | const int& | const int&       | lvalue   |
| `f(23);` | int         | int        | int&&            | rvalue   |
#### Case 3: `ParamType` is Neither a Pointer nor a Reference
Dealing with pass-by-value, which means that `param` will be a copy (completely new object) of whatever is passed. 

```c++
template<typename T>
void f(T param);

int x = 27;
const int cx = x;
const int& rx = x;
```

If `expr` is const or volatile - ignore that, if reference - ignore the reference part. 

|          | passed type | `T` type | `ParamType` type |
| -------- | ----------- | -------- | ---------------- |
| `f(x);`  | int         | int      | int              |
| `f(cx);` | const int   | int      | int              |
| `f(rx);` | const int&  | int      | int              |
If passed `const char* const` by value - `param` type will be `const char*`.

##### Array Arguments
Array parameter (`int param[]`) declarations are treated as if they were pointer parameters - `param` deducted as pointer type (`int*`). 
There is no such thing as an array function pointer, although it's a legal syntax, but `void f(int param[])` <=> `void f(int* param)`.

Although functions can't declare parameters that are truly arrays, they can declare parameters that are references to arrays:
```c++
template<typename T>
void f(T& param);

const char name[] = "J. P. Briggs";
```

|            | `T` type       | `ParamType` type  |
| ---------- | -------------- | ----------------- |
| `f(name);` | const char[13] | const char(&)[13] |
The ability to declare references to arrays enables creation of a template that deduces the number of elements that an array contains. 

##### Function arguments 
Function types can decay into function pointers, and everything mentioned about type deduction for arrays applies to type deduction for arrays applies to type deduction for functions and their decay into function pointers. 

```c++
void someFunc(int, double);

template<typename T>
void f1(T param);

template<typename T>
void f2(T& param);
```

|                | passed type       | `ParamType` type      |
| -------------- | ----------------- | --------------------- |
| `f1(someFunc)` | void(int, double) | void(\*)(int, double) |
| `f2(someFunc)` | void(int, double) | void(&)(int, double)  |

### Thing to Remember
+ During template type deduction, arguments that are references are threaded as non-refrences, i.e., their rerence-ness is ignored.
+ When deducting types for universal reference parameters, lvalue arguments get special treatment.
+ When deducing types for by-value parameters, `const`-and/or `violatile` arguments are ignore.
+ During template type deduction, arguments that are array or function names decay to pointers, unless they're used to initialize references. 



## Item 2: Understand `auto` type deduction.
`auto` deduction _almost_ is template deduction. Same behavior as for Universal references, array and function names decay into pointers for non-reference type specifiers.

Except case with uniform initialization with `{}`. In this case might appear `std::initializer_list` (for example, in `auto x = { 27 };`)
> Note: `auto x { 27 };` before C++17 probably might return also `std::initializer_list` and use it as example as well, but it's more likely outdated. 

For `std::initializer_list` type can't be deducted if in list different types, like `auto x = { 1, 2, 3.0 };` is compilation error. 

Also this effect template behavior 
```c++
template<typename T>
void f(T param);

f({ 11, 23, 9 }); // error, can't deduce type for T

template<typename T>
void f(std::initializer_list<T> initList);
f({ 11, 23, 9 }); // T - int, initList - std::initializer_list
```

Since C++14, `auto` can be used for deduce return type, which use template type deduction. As result
```c++
auto createInitList()
{
    return { 1, 2, 3 }; // error - can't deduce type. 
}
```

### Things to Remember 
+ `auto` type deduction is usually the same as template type deduction, but `auto` deduction assumes that a braced initializer represents a `std::initializer_list`, and template type deduction doesn't.
+ `auto` in a function return type or a lambda parameter implies template type deduction, not `auto` type deduction. 



## Item 3: Understand `decltype`.
In contrast to what happens during type deduction for templates and `auto`, `decltype` typically parrots back the exact type of the name or expression given. 
In C++11, perhaps, the primary use for `decltype` is declaring function templates where the function's return type depends on its parameter types. 

```c++
template<typename Container, typename Index>
auto authAndAccess(Container& c, Index i) -> decltype(c[i])
{
    authenticateUser();
    return c[i];
}
```
Here example of `decltype` and _trailing return type_ syntax. Container's `operator[]` typically return `T&`, but `std::vector<bool>` doesn't return `bool&`, it returns `std::vecotr<bool>::reference`. A trailing return type has the advantage that the function's parameters _can be used in the specification of the return type_. 

In C++ we can omit the trailing return type, with that form of declaration, `auto` does mean that type deduction will take place. Since `auto` deduction can strip reference, `decltype(auto)` makes perfect sense: `auto` specifies that the type is to be deduced, and `decltype` says that decltype rules should be used during the deduction.
```c++
template<typename Container, typename Index>
decltype(auto) authAndAccess(Container& c, Index i)
{
    authenticateUser();
    return c[i];
}
```

In example the container is passed by lvalue-reference-to-non-const, because returning a reference to an element of the container permits clients to modify that container. But this mean it's not possible to pass rvalue containers to this function. Rvalue can't bind to lvalue references (unless they're lvalue-references-to-const). Overloading for support both lvalue and rvalue would work, but than we'd have two functions to maintain. A way to avoid that is to employ a reference parameter that can bind to lvalues and rvalues - _universal references_. Using universal references also suggested with `std::forward` for taking advantages of _reference collapsing_.
```c++
// C++14 version
template<typename Container, typename Index>
decltype(auto) authAndAccess(Container&& c, Index i)
{
    authenticateUser();
    return std::forward<Container>(c)[i];
}
// C++11 version
template<typename Container, typename Index>
auto authAndAccess(Container&& c, Index i) -> decltype(std::forward<Container>(c)[i])
{
    authenticateUser();
    return std::forward<Container>(c)[i];
}
```

Applying `decltype` to a name yields the declared type for that  name. Names are lvalue expressions, but that doesn't affect decltype's behavior. For lvalue expressions more complicated than names, however, decltype ensure that the type reported is always an lvalue reference. That is, if an lvalue expression other than a name has type `T`, `decltype` reports that type is `T&`. This seldom has any impact, because the type of most lvalue expressions inherently includes an lvalue refrence qualifier. Functions returning lvalues, for example, always return lvalue references. 

Thus, putting parentheses around a name can change the type that decltype report for it. 
```c++
decltype(auto) f1() 
{
    int x = 0;
    return x; //decltype(x) is int
}

decltype(auto) f2() 
{
    int x = 0;
    return (x); //decltype((x)) is int&
}
```
Here `(x)` - is lvalue expression.

### Things to Rememver
+ `decltype` almost always yields the type of a variable expression without any modifications.
+ For lvalue expression of type T other than names, `decltype` always reports a type of `T&`
+ C++14 supports `decltype(auto)`, which, like `auto`, deduces a type from its initializer, but it performs the type deduction using the `decltype` rules. 



## Item 4: Know how to view deduced types.
#### IDE Editors 
Code editors in IDEs often show the types of program entities (e.g., variables, parameters, functions, etc.). IDEs mostly get this information from compiler running inside the IDE. For simple types information from IDEs is generally fine, however, when more complicated types are involved, the information displayed by IDEs may not be particularly helpful.

#### Compiler Diagnostics
An effective way to get a compiler to show a type it has deduced is to use type in a way that leads to compilation problems. 
For example, attempts to instantiate this template will elicit an error message, because there's no template definition to instantiate 
```c++
template<typename T> class TD;

auto x = { 27 };
TD<decltype(x)> td;
// error: aggregate 'TD<std::initializer_list<int> > td' has incomplete type and cannot be defined
```

#### Runtime Output
Defined in `#include <typeinfo>` member function [`std::type_info::name`](https://en.cppreference.com/w/cpp/types/type_info/name) can return implementation defined null-terminated character string containing the name of the type. 
Calls to `std::type_info::name` are not guaranteed to return anything sensible, but implementations try to be helpful, level of helpfulness varies from compiler to compiler. For example, _clang_ and _gcc_ can output "PKi" which means "pointer to ~~konst~~ const int", _msvc_ - "int const \* ptr64".

In more complex cases results might not be reliable. For example
```c++
template<typename T>
void f(const T& param)
{
    std::cout << "T = " << typeid(T).name() << '\n';
    std::cout << "param = " << typeid(param).name() << '\n';
}

class Widget {};

int main()
{
    std::vector<Widget> createdVec{};
    f(&createdVec[0]);  
}
```
_gcc_ and _clang_ reports for both `T` and `param` type "PK6Widget" (pointer to const, 6 characters in name, Widget), _msvc_ - "class Widget const \*", while `param`'s type is `const Widget* const&` - due to template type deduction - refrence-ness is ignored, and if the type after reference removal is `const` (or `volatile`), it's also ignored. 

More often _Boost.TypeIndex_ can succeed in type determine ([type_id_with_cvr](https://beta.boost.org/doc/libs/1_67_0/doc/html/boost/typeindex/type_id_with_cvr.html)) 
```c++
#include <iostream>
#include <boost/type_index.hpp>
#include <vector>

template<typename T>
void f(const T& param)
{
    using boost::typeindex::type_id_with_cvr;
    std::cout << "T = " << type_id_with_cvr<T>().pretty_name() << '\n'; 
    std::cout << "param = " << type_id_with_cvr<decltype(param)>().pretty_name() << '\n'; 
}

class Widget {};
int main()
{
    std::vector<Widget> createVec{};
    f(&createVec[0]);   
}
/*======== output ==========
  T = Widget*
  param = Widget* const&
  ==========================*/
```

### Things to Remember
+ Deduced types can often be seen using IDE editors, compiler error message, and the Boost TypeIndex library.
+ The result of some tools may be neither helpful nor accurate, so an understanding of C++'s type deduction rules remains essential.
