## Item 49: Understand the behavior of the new-handler.
When operator new can’t satisfy a memory allocation request, it throws an exception. Long ago, it returned a null pointer.

Before operator new throws an exception in response to an unsatisfiable request for memory, it calls a client-specifiable error-handling function called a new-handler.
```c++
namespace std {
    typedef void (*new_handler)();
    new_handler set_new_handler(new_handler p) throw();
}

void outOfMem()
{
    std::cerr << "Unable to satisfy request for memory\n";
    std::abort();
}
std::set_new_handler(outOfMem);
```
`set_new_handler`’s parameter is a pointer to the function operator new should call if it can’t allocate the requested memory. The return value of `set_new_handler` is a pointer to the function in effect for that purpose before `set_new_handler` was called.

When operator new is unable to fulfill a memory request, it calls the new-handler function repeatedly until it can find enough memory.

A well-designed new-handler function must do one of the following:
+ **Make more memory available.**
+ **Install a different new-handler**. The current new-handler can install the other new-handler in its place.
+ **Deinstall the new-handler**.
+ **Throw an exception**.
+ **Terminate application**.

Example how to implement for specific class custom new-handler based on _the curiously recurring template pattern (CRTP)_: 
```c++
template<typename T> 
class NewHandlerSupport { 
public: 
    static std::new_handler set_new_handler(std::new_handler p) throw();
    static void* operator new(std::size_t size) throw(std::bad_alloc);

private:
    static std::new_handler currentHandler;
};

template<typename T>
std::new_handler
NewHandlerSupport<T>::set_new_handler(std::new_handler p) throw()
{
    std::new_handler oldHandler = currentHandler;
    currentHandler = p;
    return oldHandler;
}

template<typename T>
void* NewHandlerSupport<T>::operator new(std::size_t size)
throw(std::bad_alloc)
{
    NewHandlerHolder h(std::set_new_handler(currentHandler));
    return ::operator new(size);
}
// this initializes each currentHandler to null
template<typename T>
std::new_handler NewHandlerSupport<T>::currentHandler = 0;

class Widget: public NewHandlerSupport<Widget> {
...
};
```



## Item 50: Understand when it makes sense to replace `new` and `delete`.
These are three of the most common reasons:
+ **To detect usage errors.** Custom operator `new`s can overallocate blocks so there’s room to put known byte patterns (“signatures”) before and after the memory made available to clients. Operator `delete`s can check to see if the signatures are still intact. If they’re not, an overrun or underrun occurred
+ **To collect statistics about the use of dynamically allocated memory.** Custom versions of operator `new` and operator `delete` can collect information like distribution of allocated block sizes; distribution of their lifetimes; tend to be allocated and deallocated in FIFO, LIFO or something closer to random order; do the usage patterns change over time, e.g., does software have different allocation/deallocation patterns in different stages of execution; maximum amount of dynamically allocated memory in use at any one time; etc
+ **To increase the speed of allocation and deallocation.** General purpose allocators are often (though not always) a lot slower than custom versions, especially if the custom versions are designed for objects of a particular type.
+ **To reduce the space overhead of default memory management.** General-purpose memory managers are often (though not always) not just slower than custom versions, they often use more memory, too.
+ **To compensate for suboptimal alignment in the default allocator.** For example case, when the operator `new` that ship with some compilers don’t guarantee eight-byte alignment for dynamic allocations of doubles, replacing the default operator `new` with one that guarantees eight-byte alignment could yield big increases in program performance.
+ **To cluster related objects near one another.**
+ **To obtain unconventional behavior.** For example, write a custom operator `delete` that overwrites deallocated memory with zeros in order to increase the security of application data.



## Item 51: Adhere to convention when writing `new` and `delete`.
Implementing a conformant operator `new`requires having the right return value, calling the new-handling function when insufficient memory is available, and being prepared to cope with requests for no memory.

Operator `new` actually tries to allocate memory more than once, calling the new-handling function after each failure. The assumption here is that the new-handling function might be able to do something to free up some memory. Only when the pointer to the new-handling function is null does operator `new` throw an exception. C++ requires that operator new return a legitimate pointer even when zero bytes are requested.

```c++
// migth have additional parameters
void* operator new(std::size_t size) throw(std::bad_alloc)
{
    using namespace std;
    if (size == 0)
        size = 1; 

    while (true) {
        //attempt to allocate size bytes;
        if (/*the allocation was successful*/)
            return /*(a pointer to the memory)*/;
        // allocation was unsuccessful; find out what the
        // current new-handling function is
        new_handler globalHandler = set_new_handler(0);
        set_new_handler(globalHandler);
        if (globalHandler) 
            (*globalHandler)();
        else 
            throw std::bad_alloc();
    }
}
```
Class member version should also check in base class that for `new` can be called derived class
```c++
class Base {
public:
    static void* operator new(std::size_t size) throw(std::bad_alloc);
    //...
};
class Derived: public Base // Derived doesn’t declare
{ ... }; // operator new
Derived *p = new Derived; // calls Base::operator new!

void* Base::operator new(std::size_t size) throw(std::bad_alloc)
{
    if (size != sizeof(Base)) // if size is “wrong,”
        return ::operator new(size);
    //..
}
```

For operator `delete`, things are simpler. About all you need to remember is that C++ guarantees it’s always safe to delete the null pointer, so you need to honor that guarantee.
```c++
void operator delete(void *rawMemory) throw()
{
    if (rawMemory == 0) 
        return; // do nothing if the null
                // pointer is being deleted
    /* deallocate the memory pointed to by rawMemory; */
}

class Base { 
public: 
    static void operator delete(void *rawMemory, std::size_t size) throw();
    //...
};

void Base::operator delete(void *rawMemory, std::size_t size) throw()
{
    if (rawMemory == 0) 
        return; 
    if (size != sizeof(Base)) { // if size is “wrong,”
        ::operator delete(rawMemory); // have standard operator
        return; // delete handle the request
    }
    /* deallocate the memory pointed to by rawMemory; */
}
```



## Item 52: Write placement `delete` if you write placement `new`.
When you’re using only the normal forms of `new` and `delete`, then, the runtime system has no trouble finding the delete that knows how to undo what new did. The which-delete-goes-with-this-new issue does arise when you start declaring non-normal forms of operator `new` — forms that take additional parameters.

The runtime system looks for a version of operator `delete` that takes the same number and types of extra arguments as operator `new`, and, if it finds it, that’s
the one it calls _only if an exception arises from a constructor call that’s coupled to a call to a placement `new`._ This means that must be provided both the normal operator `delete` and a placement version that takes the same extra arguments as operator `new` does.



