## Item 5: Know what functions C++ silently writes and calls. 
The default constructor and the destructor primarily give compilers a place to put "behind the scenes" code such as invocation of constructors and destructors of base classes and non-static data members. Note that the generated destructor is non-virtual unless it's for a class inheriting from a base class that itself declares a virtual destructor (in which case the function's virtualness comes from the base class).

As for copy constructor and copy assignment operator, the compiler-generated versions simply copy each non-static data member of the source objects over to the target object. 

If defined constructor with arguments, compilers won't generate a default constructor.

Compiler-generated copy assignment operators behave as described above only when the resulting code is both legal and has a reasonable chance of making sense. 

Compilers reject implicit copy assignment operators in derived classes that inherit from base classes declaring the copy assignment operator private. 

> Note: since book based on standards before C++11, move semantics not described. 



## Item 6: Explicitly disallow the use of compiler-generated functions you do not want. 
 The key to the solution is that all the compiler generated functions are
 public. To prevent these functions from being generated, you must
 declare them yourself, but there is nothing that requires that you
 declare them public. Instead, declare the copy constructor and the
 copy assignment operator private.
 > Note: book based on standards before C++11, from C++11 available `= delete` 



## Item 7: Declare destructors virtual in polymorphic base classes. 
A factory function - a function that returns a base class pointer to a newly-creatged derived class object. If used such functions and base class doesn't contain virtual destructor - derived class destructor of object that is being deleted via a base class pointer will not be called, results are undefined (what typically happens at runtime is that the derived part of the object is never destroyed).

If a class does not contain virtual functions, that often indicates it is not meant to be used as a base class. When a class is not intended to be a bas class, making the destructor virtual is usually a bad idea due to vtable overhead. 

The implementation of virtual functions requires that objects carry information that can be used at runtime to determine which virtual functions should be invoked on the object. This information typically takes form of a pointer called a _vptr_ ("virtual table pointer"). The _vptr_ to an array of function pointers called a _vtbl_ ("virtual table"); each class with virtual functions has an associated _vtbl_. When a virtual function is invoked on an object, the actual function called is determined by following the obhject's _vptr_ to a _vtbl_ and then looking up the appropriate function pointer int the _vtbl_

Occasionally it can be convenient to give a class a pure virtual destructor. Recall that pure virtual functions result in abstract classes - classes that can't be instantiated (i.e., can't be used to create objects of that type). Sometimes might require to make class as abstract, but class don't have any pure virtual functions - in this case destructor can be made as pure virtual, however in this case definition for the pure virtual destructor must be provided: 
```c++
class AWOV {
public: 
    virtual ~AWOV() = 0;
};
AWOV::~AWOV() {}
```

The way destructor works is that the most derived class's destructor is called first, then the destructor of each base class is called. Compilers will generate a call even for pure virtual destructor and without definition the linker will complain. 

The rule for giving base classes virtual destructors applies only to polymorphic base classes - to base classes designed to allow the manipulation of derived class types through base class interfaces. Not all base classes are designed to be used polymorphically. 



## Item 8: Prevent exceptions from leaving destructors. 
Destructors that emit exceptions are dangerous, always running the risk of premature program termination or undefined behavior. If functions called in a destructor may throw, the destructor should catch any exceptions, then swallow them or terminate the program.



## Item 9: Never call virtual functions during construction or destruction.
During base class construction, virtual functions never go down into derived classes. Instead, the object behaves as if it were of the base type. Informally speaking, during base class construction, virtual functions aren't.

During base class construction of a derived class object, the type of the object is that of the base class.

The same reasoning applies during destruction. Once a derived class destructor has run, the object's derived class data members assume undefined values, so C++ threats them as if they no longer exist. 

Since we can't use virtual functions to call down from base classes during construction, we can compensate by having derived classes pass necessary construction information up to base class constructors instead. 
```c++
class Transaction {
 public:
     explicit Transaction(const std::string& logInfo);

     void logTransaction(const std::string& logInfo) const;
 };
 Transaction::Transaction(const std::string& logInfo)
 {
     logTransaction(logInfo);
 }
 
 class BuyTransaction: public Transaction {
 public:
     BuyTransaction( parameters )
         : Transaction(createLogString( parameters ))
     { ... }
 ...
 private:
     static std::string createLogString( parameters );
 };
```



## Item 10: Have assignment operators return a reference to `*this`.
In C++ assignments can be chain together, in this case assignment is right-associative: 
```c++
int x, y, z;
x = y = z = 15; // x = (y = (z = 15))
```
The way this is implemented is that assignment returns a reference to its left-hand argument. In general it's good idea to make user-defined types behavior similar to basic types, thus better to implement similar behavior in all assignment operators for classes:
```c++
Widget& operator=(const Widget& rhs)
{
    //...
    return *this;
}
Widget& operator+=(const Widget& rhs)
{
    //...
    return *this;
}
Widget& operator=(int rhs)
{
    //...
    return *this;
}
```



## Item 11: Handle assignment to self in operator=
An assignment to self occurs when an object is assigned to itself. 
This can lead to trap of accidentally releasing a resource before done using it. 
```c++
class Widget {
public:
    Widget& operator=(const Widget& rhs)
    { //unsafe impl. 
        delete pBitmap;
        pBitmap = new Bitmap(*rhs.pBitmap);
        return *this;
    }
private:
    Bitmap* pBitmap; 
};
```

Another problem - if some exception emitted.
```c++
Widget& Widget::operator=(const Widget& rhs)
{
    if (this == &rhs) return *this;

    delete pBitmap;
    pBitmap = new Bitmap(*rhs.pBitmap);
    return *this;
}
```
If `new` yields an exception, the Widget will end up holding a pointer to deleted pBitmap. Such pointers are toxic, they can't safely be deleted, they can't be safely read.

Making `operator=` exception-safe typically renders it self-assignment-safe. 
```c++
Widget& Widget::operator=(const Widget& rhs)
{
    Bitmap* pOrig = pBitmap;
    pBitmap = new Bitmap(*rhs.pBitmap);
    delete pOrig;

    return *this;
}
```
In this example self-identity test can be added, but it's possibly redundant: self-assignment test isn't free and will make actual profit in unrealistic cases (how often we expect that self-assignment will occur). 

An alternative to manually ordering statements in `operator=` to make sure the implementation is both exception- and self-assignment-safe is to use the technique known as _copy and swap_
```c++
class Widget 
{
    void swap(Widget& rhs);
    Widget& operator=(const Widget& rhs)
    {
        Widget temp(rhs); // passing by reference require manual copy
        swap(temp);
        return *this;
    }
    Widget& operator=(Widget rhs) // passing by value makes a copy of it
    {
        swap(rhs);
        return *this;
    }
}
```



## Item 12: Copy all parts of an object. 
When implementing copying functions for some inherited class it's easy to miss fact that not only members of derived class requires to be copied, but also member of base class. Every inherited class contains a copy of data members of it's base class. In this example those data members are not being copied at all and for base class member default initialization will be used  
```c++
class Customer 
{
public:
    Customer(const Customer& rhs)
        : name(rhs.name) 
    {}
    
    Customer& operator=(const Customer& rhs)
    {
        name = rhs.name;
        return *this;
    }

private:
    std::string name;
};


class PriorityCustomer : public Customer 
{
public:
    PriorityCustomer(const PriorityCustomer& rhs)
        : priority(rhs.priority) 
    {}
    
    PriorityCustomer& operator=(const PriorityCustomer& rhs)
    {
        priority = rhs.priority;
        return *this;
    }

private:
    int priority;
};
```

Any time class implements own copying functions for a derived class, base class parts should be taken in account as well. For this in inherited class can be invoked base class corresponding class functions:
```c++
    PriorityCustomer(const PriorityCustomer& rhs)
        : Custmor(rhs)
        , priority(rhs.priority) {}
    
    PriorityCustomer& operator=(const PriorityCustomer& rhs)
    {
        Customer::operator=(rhs);
        priority = rhs.priority;
        return *this;
    }
```

When copy constructor and copy assignment operator have similar code bodies, code duplication can be eliminated by creating member function (typically called `init`) that both call. 