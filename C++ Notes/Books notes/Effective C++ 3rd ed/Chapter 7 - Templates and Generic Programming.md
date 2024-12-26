The world of object-oriented programming revolves around _explicit_ interfaces and _runtime_ polymorphism: we can see exactly what class derived class inherits and what virtual functions will be called resolves in runtime.

The world of templates and generic programming it is world, where explicit interfaces and runtime polymorphism continue to exist, but they're less important, instead _implicit_ interfaces and _compile-time_ polymorphism move to the fore. 

An explicit interface typically consists of function signatures, i.e. function names, parameter types, return types, etc. 

An implicit interface rather consists of valid expressions. 
```c++
template<typename T>
void doProcessing(T& w)
{
    if (w.size() > 10 && w != someNastyWidget) {
    ...
}
```
`T` must support `size` member function, but this member function need not return an integral (or numeric) type, it need not even return a type for which `operator>` defined. All it needs to do is return an object of some type `X` such that there is an `operator>` that can be called with an object of type `X` and `int` (because 10 is of int type). The `operator>` need not take a parameter of type `X`, because it could take a parameter of type `Y`, and that would be okay as long as there were an implicit conversion from objects of type `X` to objects of type `Y`.



## Item 42: Understand the two meaning of `typename`.
Between `template<class T>` and `template<typename T>` there is no difference.
However, C++ doesn't always view `class` and `typename` as equivalent. 

```c++
template<typename C> // typename allowrd (as is "class")
void print2nd(const C& container) //typename not allowed
{
    if (container.size() >= 2) {
        typename C::const_iterator iter(container.begin()); //typename required
    }
}
```
`iter` is depended on template parameter `C`, it's _nested dependent type name_. Anytime we refer to a nested dependent type name in a template, it's must be precede by the `typename` keyword. `typename` should be used to identify only nested dependent type names, exception is that `typename` must not precede nested dependent type name in a list of base classes or as a base class identifier in a member initialization list.  
```c++
template <typename T> 
class Dervied : public Base<T>::Nested // typename not allowed in base class list
{
public:
    explicit Derived (int x)
        : Base<T>::Nested(x)  // not allowed: initialization list
    {
        typename Base<T>::Nested temp; // required. 
    }
```



## Item 43: Know how to access name in templatized base classes.
When compilers encounter the definition for the class template, they don't know what class it inherits from. In some sense, when we cross from Object-Oriented C++ to Template C++, inheritance stops working.
To reset it, we have to somehow disable C++'s "don't look in templatized base classes" behavior:
* preface calls to base class function with `this->`
* employ a `using` declaration. 
* explicitly specify that the function being called is in the base class (this is generally the least desirable way to solve the problem, because if the function being called is virtual, explicit qualification turns off the virtual binding behavior)

```c++
class CompanyA {
public: 
    ...
    void sendCleartext(const std::string& msg);
};
class CompanyB {
public: 
    void sendCleartext(const std::string& msg);
};
class MsgInfo { .. };

template<typename Company>
class MsgSender {
public:
    ...
    void sendClear(const MsgInfo& info)
    {
        // create msg from info
        Company c;
        c.sendCleartext(msg);
    }
};

template<typename Company>
class LoggingMsgSender : public MsgSender<Company> {
public:
    void SendClearMsg(const MsgInfo& info)
    {
        // log before

        /* sendClear(info); < will not compile */

        this->sendClear*(info);
        //or 
        MsgSender::sendClear(info);

        //log after 
    }

    // or  
    using MsgSender::sendClear; 
    void sendClearMsg(const MsgInfo& info)
    {
        ...
        sendClear(info);
        ...
    }
};
```

In same case type can not contain some function or require specific behavior. `template<>` syntax is signifies specialized version of template to be used when the template argument is something specific. This is known as a _total template specialization_: the template is specialized for the some type, and the specialization is _total_ - once the type parameter has been defined to specified type, to other aspect of the template's parameters can vary.
```c++
class CompanyZ {
public:
    //no sendCleartext function 
    void sendEncrypted(const std::string& msg);
}

template<>
class MsgSender<CompanyZ> {
public:
    // sendClear is omitted
    void sendSecret(const MsgInfo& info) { ... }
};
LoggingMsgSender<CompanyZ> zMsgSender;
MsgInfo msgData;
/* zMsgSender.sendClearMsg(msgData); < won't comple */
```



## Item 44: Factor parameter-independet code out of templates.
In non-template code, replication is implicit: duplication can be seen between two functions or two classes. In template code, replication is implcitit: replications may teke place when a template is instanitated multiple times.
```c++
template<typename T, std::size_t n>
classs SquareMatric {
public:
    void invert();
};

```

Non-type paramater (`size_t`) can cause instantiation of diferent copies: `SquareMatrix<double, 5> sm1;` and `SquareMatrix<double, 10> sm2;`

In this case - base class templatized only on type of obhjects, hence all matrices holding a given type will share single base class, thus will share a single copy of that class's version of invert.

```c++
template<typename T>
class SquareMatrixBase {
protected:
    void invert(std::size_t matrixSize);
};

template<typename T, std::size_t n>
class SquareMatrix : private SquareMatrixBase<T> {
private:
    using SquareMatrixBase<T>::invert; // make base class version visible

public:
    void invert() { invert(n); }
};
```

If it's inlined, each instantaion of SquareMatrix::invert will get a copy of SquareMatrixBase::invert’s code - this will step back in object code replication.



## Item 45: Use member function templates to accept "all compatible types".
_Member function templates_ (member templates) - templates that generate member functions of a class:
```c++
template<typename T>
class SmartPtr {
public:
    template<typename U>
    SmartPtr(const SmartPtr<U>& other);
};
```
Constructors that create one object from another object whose type is a different instantiation of the same template are sometimes known as _generalized copy constructors_. 

Declaring a generalized copy constructor in a class doesn't keep compilers from generating their own copy constructor (a non-template).



## Item 46: Define non-member functions inside templates when type conversions are desired.
Inside a class template, the name of the template can be used as shorthand for the template and its parameters, so instead `Rational<T>` can be used `Rational`. That saves only a few characters, but when there are multiple parameters or longer parameters name, it can both save typing and make the resulting code clearer.

When implementing class, it's recommended to [[Chapter 4 - Design and Declarations#Item 24 Declare non-member functions when type conversions should apply to all parameters.|Declare non-member functions when type conversions should apply to all parameters]], for example for `operator*`, for templates class it's can be a little bit tricky due to type deduction. 

Simplest way to solve is to merge the body of `operator*` into its declaration:
```c++
template<typename T> 
class Rational {
public:
    ...
    friend const Rational operator*(const Rational& lhs, const Rational& rhs)
    {
        return Rational(lhs.numerator() * rhs.numerator(), 
                        lhs.denominator() * rhs.denominator());
    }
}
```
Note that this will be inline.

The fact that class is a template means that the helper function will usually also be a template
```c++
template<typename T> class Rational;
template<typename T>
const Rational<T> doMultiply(const Rational<T>& lhs, 
                             const Rational<T>& rhs)
{
    return Rational<T>(lhs.numerator() * rhs.numerator(), 
                       lhs.denominator() * rhs.denominator());
}

template<typename T>
class Rational {
public:
    ...
    friend const Rational operator*(const Rational& lhs, const Rational& rhs)
    { return doMultiply(lhs, rhs); }
}
```
As a template, `doMultiply` will not support mixed-mode multiplication, but it doesn't need to, since `operator*` does support mixed-mode operations. 



## Item 47: Use traits classes for information about types.
In some cases might be required to have ability to get some information about a type. That's what _traits_ let do.
Traits aren't a keyword or a predefined construct in C++, they're a technique and a convention followed by C++ programmers. 

The traits information for a type must be external to the type (since there's no way  to nest information inside pointers)

One of the convention - traits are always implemented as struct. Structs used to implement traits are known as traits classes. 

Example from STL iterators: 
```c++
tempalte<typename iterT>
struct iterator_traits {
    typedef typename iterT::iterator_category iterator_category;
};

template <...>
class deque {
public:
    class iterator {
    public:
        typedef random_access_iterator_tag iterator_category;
    };
    ...
}
```
The way iterator_traits works is that for each type `iterT` a typedef `iterator_category` is declared in the struct `iteraotor_traits<iterT>` which identified the iterator category of `iterT`.
To support iterators-pointers `iterator_trits` offers a _partial template specialization_ for pointer types 
```c++
tempalte<typename T>
struct iterator_traits<T*> {
    typedef random_access_iterator_tag iterator_category;
};
```

To design and implement a traits class:
+ Identify some information about types to make available (e.g., for iterators, their iterator category).
+ Choose a name to identify that information (e.g. iterator_category)
+ Provide a template and set if specializations that contain the information for the types to support. 

Issue now is that for example if statement is evaluated at runtime. Instead should be used overloading, which require to create multiple versions of an overloaded functions
```c++
template<typename IterT, typename DistT>
void doAdvance(IterT& iter, DistT d, std::random_access_iterator_tag)
{
...
}
template<typename IterT, typename DistT>
void doAdvance(IterT& iter, DistT d, std::bidirectional_iterator_tag)
{
...
}

template<typename IterT, typename DistT>
void advance(IterT& iter, DistT d)
{
    doAdvance(iter, d, 
              typename std::iterator_traits<IterT>::iteraotr_category());
}
```

Summary how to use a traits class:
+ Create a set of overloaded “worker” functions or function templates that differ in a traits parameter. Implement each function in accord with the traits information passed.
+ Create a “master” function or function template that calls the workers, passing information provided by a traits class.



## Item 48: Be aware of template metaprogramming.
A template metaprogram is a program written in C++ that executes inside the C++ compiler. 

TMP makes some things easy that would otherwise be hard or impossible. And because template metaprograms execute during C++ compilation, they can shift work from runtime to compile-time. One consequence is that some kinds of errors that are usually detected at runtime can be found during compilation. Another is that C++ programs making use of TMP can be more efficient in just about every way: smaller executables, shorter runtimes, lesser memory requirements.

TMP has been shown to be Turing-complete, which means that it is powerful enough to compute anything. Using TMP, you can declare variables, perform loops, write and call functions, etc. But such constructs look very different from their “normal” C++ counterparts.

