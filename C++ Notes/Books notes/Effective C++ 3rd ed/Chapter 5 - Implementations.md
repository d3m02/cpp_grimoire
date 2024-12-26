## Item 26: Postpone variable definitions as long as possible. 
```c++
// this function defines the variable "encrypted" too soon 
std::string encryptPassword(const std::string& password) 
{
    using namespace std; 
    string encrypted; 
    if (password.length() < MinimumPasswordLength) 
        throw logic_error("Password is too short");
    ...
    return encryped;
}
```
The object `encrypted` isn’t completely unused in this function, but it’s
unused if an exception is thrown. Problem is that we pay for the construction and of `encrypted` even if exception thrown.
Simply moving definition closer to variable use reduce possible redundancy. 

Also preferable to initialize variable instead defining and assigning, thus we skip pointless default construction.

With loops situation not that easy. It's depend on data sensitivity (how long should it live) and if constructor-destructor pair less expansive. 
```c++
// Approach A
// 1 constructor + 1 destructor + n assignments
// name w visible in a larger scope
Widget w;
for (int i = 0; i < n; ++i)
{
    w = something(i);
    ...
}

// Approach B
// n construcrtors + n destructors. 
for (int i = 0; i < n; i++)
{
    Widget w(someting(i));
    ...
}
```



## Item 27: Minimize casting. 
The rules of C++ are designed to guarantee that type errors are impossible. Unfortunately, casts subvert the type system. Casting not always telling compilers to threat one type as another. Type conversions of any kind often lead to code that is executed at runtime. 

```c++
class Base {...};
class Derived : public Base { .. };
Derived d;
Base* pb = &d;
```
Here we're just creating a base class pointer to a derived class object, but sometimes, the two pointer values will not be the same. When that's the case, an offset is applied at runtime to the Derived pointer to get correct Base pointer value. This demonstrates that a single object might have more than one address (e.g., its address when pointed to by a `Base*` pointer and its address when pointed to by a `Derived*` pointer).

In next example intend to implement in virtual member function implementations in derived classes call their base class counterparts first. 
```c++
class SpecialWindow : public Window {
public: 
    virtial void onResize() {
        static_cast<Window>(*this).onResize();
        ...
    }
}
```
What actually happening here - cast creates a new temporary copy of the bass class part of `*this`, then invokes onResize on the copy. Instead should be used simple `Window::onResize();`

`dynamic_cast` is slow. The need for `dynamic_cast` generally arises because sometime required to perform derived class operations but only available  a pointer or reference-to-base through which to manipulate the object. 
```c++
std::vector <Base> collection;
// note: in book used for with iterator and iter++; range-based loops 
//and auto in C++ was added after book publication.
for (auto item : collection)
{
    if (auto* pDerived = dynamic_cast<Derived*>(item))
        pDerived->blink();
}
```
it's can cost a lot in runtime, better instead have separate container for Derived classes. Or declare in base class the function that does nothing 
```c++ 
class Base {
public:
    virtual void blink() {}
};
class Derived : public Base {
public:
    virtual void blink() { ... }
};
std::vector <Base> collection;
for (auto item : collection)
{
    item->blink();
}
```



## Item 28: Avoid returning "handles" to object internals. 
Assume that we have class 
```c++ 
struct RectData {
    Point ulhc;
    Point lrhc;
};
class Rectnagle {
private:
    std::shared_ptr<RectData> pData;
};
```
If add functions like 
```c++
class Rectangle 
{
public:
    Point& upperLeft() const { return pData->ulhc; }
    Pointer& lowerRight() const { return pData->lrhc; }
}

const Rectange rec(coord1, coord2);
rec.upperLeft().setX(50); // will change const object private member
```

Important not to return handles to object "internals". This means never have a member functions that return a pointer to a less accessible member function. 

We can add `const` to return value, this might prevent from modifying. Even so, they are still returning handles to an object's internals, and that can be problematic in other ways. It can lead to dangling handles: handles that refer to parts of objects that don't exist any longer. 
```c++
class GUIObject { .. };
const Rectangle boundingBox(const GUIObject& obj);

GUIObject* gpo;
const Point* oUpperLeft = &(boundingBox(*pgo).uppeLeft());
// the call to boundingBox will return a new temporary Rectangle object
// upperLeft will then be called on temp and that call will return a refernce to and internal part of temp, that will be destroyed.
```



## Item 29: Strive for exception-safe code.
When an exception is thrown, excpetion-safe functions:
+ **Leak no resource**. 
+ **Don't allow data structures to become corrupted**.

Exception-safe functions offer one of three guarantees:
+ **Basic guarantee** - promise that if an exception is thrown, everything in program remains in a valid state. For example, if during changing background exception thrown - object stay with old background or might provide some default background. 
+ **Strong guarantee** - promise that if an exception is thrown, the state of the program is unchanged. 
+ **Nothrow guarantee** - promise never to  throw exceptions.

There is a general design strategy that typically leads to the strong guarantee - "copy and swap".

When function have side effects on non-local data, it's much harder to offer the strong guarantee. If a side effect of calling f1 function, for example, is that a database is modified, it will be hard to make someFunc (which call f1 inside) strongly exception-safe. A function can usually offer a guarantee no stronger than the weakest guarantee of the functions it calls. 



## Item 30: Understand the ins and outs of inlining. 
The idea behind an inline function is to replace each call of that function with its code body, which is likely increase the size of object code. 

On the other hand, if an inline function body is very short, the code generated for the function body may be smaller than the code generated for a function call. 

Inlining is request, not  command. The request can be given implicitly or explicitly. 

The implicit way is to define a function inside a class definition:
```c++
class Person {
public:
    int age()const { return mAge; }
};
```
Friend functions can also be defined inside classes, they're also implicitly declared inline.

The explicit way is to precede its definition with `inline` keyword. 

Inline functions must typically be in headers files, because most build environments do inlining during compilation, compilers must know what the function looks like. 

Compiler can refuse to make inlining, if function looks complicated (e.g. functions that contain loops or are recursive, if code might take address of function). Whether a given inline function is actually inlined depends on the build environment, primarily on the compiler. 

Inline in library can restrict binary upgrades: if some function is inline, clients of library compile the body of f into their application and then f implementation changed - clients must recompile f, while non-inline version require only relink.



## Item 31: Minimize compilation dependencies between files.
The problem is that C++ doesn't do a very good job of separating interfaces from implementations. A class definition specifies not only a class interface but also a fair implementations details. 
```c++
class Person {
public:
    Person(const std::srting& name, const Address& address);
    std::string name() const;
    std::string address() const;

private: 
    std::string m_name; //implementation detail, require include <string>
    Address m_address;  //implementation detail, require include "adress.h"
}
```

Forward declaring can partially solve this problem, but not everything can forward-declared. For example, `std::string` can't be forward-declared since it's not a class, it's `typedef` (for `basic_string<char>`). Also compiler need to know size of objects during compilation. Generally speaking, forward-declaring only works for pointers and references. 

#### Handle classes
One way to solve this problem is separating into two classes - interface (_Handle class_) class and implementation. 
```c++
#include <string>
#include <memory> 

class PersonImpl;
class Address;

class Person {
public:
    Person(const std::srting& name, const Address& address);
    std::string name() const;
    std::string address() const;
private: 
    std::shared_ptr<PersonImpl> pImpl;
};
```
The key to this separation is replacement of dependencies on definitions with dependencies on declarations. 

Never needed a class definition to declare a function using that class, not even if the function passes or returns the class type by value:
```c++
class Date;
Date today();
void clearAppointments(Date d);
// fine - no defintiono of Date is needed.
```

In order to facilitate adherence to the above guidelines, header files need to come in pairs: one for declarations, the other for definitions. If declaration if changed in one place, it must be changed in both. As a result, library clients should always include a declaration file instead of forward-declaring themselves.
```c++
#include "datefwd.h"

Date today();
void clearAppointments(Date d);
```
The name of declarion-only header `"datefhd.h"` is based on the header `<iosfwd>` from STL which contains declarations of iostream components. 

#### Interface classes 
An alternative to handle class approach is to make a special kind of abstract base class called _Interface class_. The purpose of such is to specify an interface for derived classes, typically interface class has no data members, not constructors, only a virtual destructor and set of pure virtual functions that specify the interface.
```c++
class Person {
public:
    virtual ~Person();
    virtual std::string name() const = 0;
    virtual std::string address() const = 0;
}
```

Clients of an interface class must have a way to create new objects. They typically do it by calling a function that plays the role of the constructor for the derived classes that are actually instantiated. Such function are typically called _factory_ or _virtual constructors_. They return pointer to dynamically allocated objects that support the interface class's interface.
```c++
class Person {
public:
    static std::shared_ptr<Person> create(const std::string& name, const Address& addr);
};

std::shared_ptr<Person> pp (Person::create(name, address));
```
Concrete classes supporting the interface class's interface must be defined and real constructor must be called. That all happens behind the scenes inside the files containing the implementations of the virtual constructors.
```c++
class RealPerson : public Person {
public:
    RealPerson(const std::string& name, const Address& addr);
    virtual ~RealPerson();
    // note: override added in standart after book publication
    std::string name() const override; 
    std::address() const override;
private:
    std::string m_name;
    Address m_address;
};
std::shhared_ptr<Person> Person::create(const std::string& name, 
                                        const Address& addr)
{
    return std::shared_ptr<Person>(new RealPerson(name, addr));
}
```