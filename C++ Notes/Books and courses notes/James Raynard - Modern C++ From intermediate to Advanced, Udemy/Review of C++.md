## `using IntVec = vector<int>` vs `typedef vector<int> IntVec;`
In general results and idea similar - define alias which compiler will use to replace types.

#### (research with chatGPT)
But difference is that `typedef` can't be used with templates similarly to 
```c++
template<typename T>
{
    using Vec = std::vector<T>;
    ...
}
```
because `T` is not type, but template parameter, set of types.


## Class functions in global scope 
Class functions are actually located in global scope and have hidden first argument class object (`this`). 
### (research with chatGPT)
And technically it's possible to call
```c++
A::f(&a, 42);
```

Also this is why static functions don't have `this` - they don't have that hidden argument. 

Virtual functions also implemented via `vtable` which stores pointer to such global functions - during object construction `vtable` pointer filled with correct pointers to functions which lately allow to execution correct function when class initialized with Base class type: 
```c++
Base* b = new Derived;
// (under hood) 
// init:
// 1. call Base::Base()
// 2. this->vptr point to Base_vtable;
// 3. call Derived::Derived()
// 4. update this->vptr = &Derived_vtable; 

void** table = b->vptr;
void (*func)(Derived*) = (void (*)(Derived*))table[0];
// or 
auto func = reinterpret_cast<void(*)(Derived*)>(table[0]);

// call function 
func(static_cast<Derived*>(b));
```

Vtables generated at compilation time in RO data segment (code segment) of executable file. For non virtual functions compiler determine which global function to call based on type analyze at compilation time.

Access to class members made through `this->`. C++ provide syntax sugar to not type `this->` always, but `this->i` is always used to access class members. Decision about add `this->` or not based on order of visibility (Local variable -> class member -> global variable), so this will be added if variable not 'shadowed' by local variable 

Access to private/public method checked by compiler at compilation time and `this->` also help compiler to identify - is access allowed or not.

## Special member functions
* Copy constructors uses const references since if use object not as reference it might end up with recursion since we need to create a copy of object to pass it as argument to copy costructor

### (research with claude)
And compiler might not allow it
><source>:9:5: error: invalid constructor; you probably meant 'A (const A&)'
   9 |     A(A other) : b{other.b}

In general it's possible to use as argument as non-const reference - in this case we not able to use temporary objects (rvalues), but we can't create situation where this create compilation error due to copy elision/RVO (instead copy constructor will be called normal constructor) and it's in general reasonable - we don't need to copy temporary object and just initialize right with it. But in this case we can't copy const object (`error: binding reference of type 'A&' to 'const A' discards qualifiers`) and can't use with stl container (`stl_construct.h:133:7: error: cannot bind non-const lvalue reference of type 'A&' to an rvalue of type 'A'`) 

  
* Assignment operator better to return non const value, mostly because built-it types return non-const and it's best practice. It's possible to return const, but in that case `a1 = a2 = a3` will not work

* For assignment constructor  used as return value reference to (`*this`) for optimization (to avoid creation of temporary object and possible copy constructions) and to provide some level of safety (`this` exist and will be exist after)

## Types and Literals
The size of C++ types depends on the implementation (beside types from `cstdint`)

| Type          | Characteristic                                           |
| ------------- | -------------------------------------------------------- |
| `char`        | the only fixed, 8 bits                                   |
| `int`         | at least 16 bits                                         |
| `long`        | at least 32, at least as many as `int`                   |
| `long long`   | from c++11, at least 64 bits, at least as many as `long` |
| `float`       | usually 6 digits precision                               |
| `double`      | usually 15 digits precision                              |
| `long double` | usually 20 digits pecision                               |

| Literals                        | Example           |
| ------------------------------- | ----------------- |
| Numeric (no literals)           | `int dec = 42`    |
| Hexadecimal(`0x`/`0X` in front) | `int hex = 0x2a`  |
| Octal **(`0` in front)**        | `int oct = 052`;  |
| Binary (`0b`/`0B` in front)     | `int bin = 0b100` |
By default integer literals are `int` (`long` and `long long` used if capacity not enough), floating-point literals are double by default. Suffix can change type of a literal

| Suffix                           | Example |
| -------------------------------- | ------- |
| float (`f`/`F`)                  | 3.1415f |
| unsigned long long (`ull`/`ULL`) | 123ULL  |
(more info in [[Notes#C++ suffixes and literals]])

C-style string literals (`"Hello worlds"`) which are just array of `char` and C++style string literals (`"Hellow worlds"s`) which create `std::sting` require to add `\` to escape some symbols (like new line of `"`). 
To simplify work with strings where lots of escapes required (like URLs) can be used raw-sting literal `R"(hello world)"` and raw sting literal with delimiter when string contain ')' `R"x(Hello (worlds))x"`

## `auto`
Originally `auto` used in K&R C (1978) to specify that a variable should be created on stack, but it was useless since local variables by default created on stack 
```c
auto int x = 5;     // variable on stack
static int y = 10;  // static variable 
register int z = 1; // CPU register
```
In **C++11** it was reinvented as type interface - compiler have to deduce type from the variable's initial value (use same rule as template deduction?)

## Iterators and vector
To modify array elements it's require to dereference iterator
```C++
std::vector<int> vec {1, 2, 3, 4};

std::cout << "Vec: \n";
for (const auto el : vec)
    std::cout << el << ", ";
for (auto& el : vec)
    el++;
std::cout << "\nVec: \n";
for (const auto el : vec)
    std::cout << el << ", ";
```

Iterators library also provide several arithmetic functions:
```c++
// itr+n (don't change iterator, just return Nth iter after)
template<class InputIt>
InputIt std::next(InputIt it, typename std::iterator_traits<InputIt>::difference_type n = 1);

// itr-n
template<class BidirIt>
BidirIt prev(BidirIt it, typename std::iterator_traits<BidirIt>::difference_type n = 1);

// lastItr - firstItr
template<class InputIt>
typename std::iterator_traits<InputIt>::difference_type std::distance(InputIt first, InputIt last);
    
// move itarator by distance n, changes interator
template<class InputIt, class Distance>
void std::advance(InputIt& it, Distance n);
```

## Constructor Template Argument Deduction
(known as "CTAD"), starting from C++17 template can deduce parameter type based on initialization. It's also appliable for STL containers 
```c++
std::vector<int> vec{1, 2, 3}; // C++11
std::vector vec{1, 2, 3};      // C++17
```

## Namespace 
Every symbol declared inside the namespace will have the namespace's name automatically prefixed to it b the compiler 
```c++
namespace abc {
    class Test; // this defined the symbol abc::Test
}
```

If a name is not in any namespace, its is to be in the "global namespace". The global namespace has no name, if we want to specify that we are referring to the global namespace (probably inside some namespace), we use `::` on it's own before the name 
```c++
class Test {...};
::Test test;
```


## Function pointer 
Function pointer is obviously a pointer to memory where function stored, any function's executable code is stored in memory. It's type complicated and better to use `auto` or `using`. 
```c++
void func (int x, int y) {
    std::cout << x << " + " << y << " = " << x + y;
}

using pfunc = void (*)(int, int); // The * is not optional.

void some_func(int x, int y, pfunc func_ptr) {
    (*func_ptr)(x, y);
    // or (func_ptr)(x, y);/func_ptr(x, y)* is optional 
}
```

### (research with claude)
Reason why asterisk is optional due to fact that any _function decay to pointer_ by compiler and it's logical to have symmetry in function call
```c
void process(int x, int y);
process(1, 2);  // 'natural' syntax

// function decay to pointer
void (*func_ptr)(int, int) = process;

func_ptr(1, 2);  // symetrical to original function 
```
Syntax `(*func_ptr)(int, int)` was added right from C89 to make it clear when used regular pointer and where function pointer
```c
void *ptr;           // pointer to void
void (*func_ptr)();  // pointer to function
```
