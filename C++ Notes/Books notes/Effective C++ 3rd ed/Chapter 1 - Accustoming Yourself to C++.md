## Item 2: Prefer consts, enums and inlines to `#defines`.
Defines may be removed by the preprocessor before source code ever gets compiler. 

Defines values in headers for other developers can be questionable, where that value came from. 

Preprocessor's blind substitution of macro could result in multiple copies of it's value in object code, while the use of `const` should never result in more than one copy. 

When replacing defines with constants, two special cases are worth mentioning: 
	- defining constant pointers, usually declared as const to what pointer points to (`const char* const title = "Example"`); 
	- class-specific constants, to limit the scope of a constant to a class, it should be made it a member, and to ensure there's at most one copy of the constant, it should be made _static_ member
	- `static const int NumTurns = 5` - is _declaration_ of NumTurns. For static class-specific integral (int, char, bool) type works exception and they can be used without providing a definition, unless their address not taken (Usually in C++ for anything to be used requires declaration). If requires to take the address of a class constant or if compiler incorrectly insists on a definitions, can be provided separate definitions `const int GamePlayer::NumTurns;`. In-class initialization is allowed only for integral types and only for constants (another exception but from statics now, static members can't be provided with initial value). Therefor, in other cases in header - `static onst double FudgeFactor`, in implementation file - `const double CostEstimate::FudgeFactor = 1.35`

defines don't respect scope, can't be used for classes and can't provide any kind of encapsulation

When value of constant required during compilation (such as in the declaration of the array), the accepted way to compensate for compiler that forbid in-class specification of initial values for static integral class constant is to use "the enum hack": `enum { NumTurns = 5 };`. 

Enum as hack can be used where ints are expected; enum acts more like \#define and sometimes it's can be used as advantage (it's not legal to take the address of and enum while taking _const_ address is legal)

Another common (mis)use of the \#define directive is using it to implement macros that look like functions but that don't incur the overhead of a function call. 
```c++
#define CALL_WITH_MAX(a, b) f((a) > (b) ? (a) : (b))
int a = 5, b = 0;
CALL_WITH_MAX(++a, b);      // a is incremented twice
                           // f((++a) > (b) ? (++a) : (b));
CALL_WITH_MAX(++a, b + 10); // a is incremented once

/* correct way will be */
template<typename T>
inline void CallWithMax (const T& a, const T& b)
{
    f(a > b ? a : b)
}
```



## Item 3: Use const whenever possible.
Syntax for pointers: if _const_ appears to the left of the asterisk, what's _pointed to_ is constant; if _const_ appears to the right of the asterisk, the _pointer itself_ is constant. As help, pointer declarations can be read right to left: to read `const char* const p` as "p is constant pointer to constant chars".

Declaring an  _iterator_ const is like declaring a pointer const (i.e. `T* const pointer`): the iterator isn't allowed to point to something different, but the thing it points to may be modified. _const_iterator_ is an iterator that points to something that can't be modified 

`const Rational operator* (const Rational& lhs, const Rational& rhs)` can looks strange (why should the result of operator* be a const object), but without const someone can commit something like `(a * b) = c`. It's makes more sense with typo like `if (a * b = c)`. Making result of operator* be a const object also one of hallmarks of good user-defined types that avoid gratuitous incompatibilities with the built-ins. 

### const member functions
The purpose of const on member functions is to identify which member functions may be invoked on const objects. In most cases const objects most often arise as a result of being passed by pointer- or reference-to-const. Thus, sometimes might add in class both, const and non-const function versions. 

In some cases member functions might not act very const. For example if const-member function modify on what member pointer is points to without modifying pointer itself.
```c++
class C {
public:
    void cFunc(int stolen) const 
    { *pTreasure = stolen; }
private:
    int* pValue {};
};
```
Or if function declared as const member but return non-const reference to member. 
```c++
class CTextBlock {
public:
    char& operator[](std::size_t position) const 
    { return pText[position]; }
private:
    char* pText;
};
const CTextBlock cctb("Hello");
char* pc = &cctb[0];
*pc = 'J';     // cctb now "Jello"
```

Sometimes it's required for const member functions to modify some of the bits n object (like flag or length), but only in ways that clients cannot detect. In this case should be used `mutable`

### Avoiding duplication in const and non-const member functions 
Creating const and non-const versions of functions can become a huge code duplications. This can be solved with casting away constness. Casting in general is a bad idea, but code duplication also not better solution. 

Casting away the const on the return value is safe if non-const version does exactly what const version, because whoever called the non-const version must have had a non-const object in the first place. 
```c++
class TextBlock {
public: 
    const char& operator[](std::size_t pos) const 
    {
        // some checks, logs etc
        return text[position];
    }
    char& operator[](std::size_t pos)
    {
        return 
            const_cast<char&>(           //cast away const on return type
                static_cast<const TextBlock&>(*this) // add const
                    [position]         // call const version of operator[]
            );
    }
}
```
We want the non-const `operator[]` to call const one, but if inside the non-const we just call `operator[]`, we'll recursively call ourselves, that's why we cast `*this` to it's const version. 

A const member function promises never to change the logical state of its objects, but a non-const member function makes no such promise. Thus having const member function call a non-const one is wrong: the object could be changed. 



## Item 4: Make sure that objects are initialized before they're used. 
Rules that describe when object initialization is guaranteed and when not are complicated: C-array isn't necessarily guaranteed to have it's contents initialized, but a stl-vector is. The bests way to deal with this is to always initialize objects before use them. 

For non-member objects of built-in types it's should be done manually. For almost everything else, the responsibility for initialization falls on constructors: make sure that all constructors initialize everything in the object.  

Assignment not a initialization. It's typical case for class members: writing in constructor body `m_id = 0` - it's assignment, and `m_id` will be initialized with something default, which make some extra cost in redundancy. The rules of C++ stipulate that data members of an object are initialize before the body of a constructor is entered. 

For class-member initialization initialization list preferable. Members in initialization list better to place in same way how they defined in class since class member initialization order match declaration order. 

Sometimes the initialization list must be used, for example, data members that are const or are references - must be initialized. 

Base classes are initialized before derived classes, and within a class, data members are initialized in the order in which they are declared. 

C++ guarantee that local static objects are initialized when the object's definition is first encountered during a call to that function. 

Static objects exists from the time it's constructed until the end of the program. Stack and heap-based objects are thus **excluded**. If initialization of a non-local static object in one translation unit uses a non-local static object in a different translation unit, the object it uses could be uninitialized, because the relative order of initialization of non-local static objects defined in different translation units **is undefined**. But, this can be fixed with singletons: non-local static object moved to own functions, function return reference to now local static object and initialization is combined with function call. As a bonus, if function never called - cost of constructing and destructing the object never incur.  

Any kind of non-const static object - local or non-local - is trouble waiting to happen in the presence of multiple threads. One way to deal with such trouble is to manually invoke all the reference-returning functions during the single-threaded startup portion of the program. This eliminates initialization-related race conditions. 