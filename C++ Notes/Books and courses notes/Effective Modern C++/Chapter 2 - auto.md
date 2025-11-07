## Item 5: Prefer `auto` to explicit type declarations.
`auto` variables have their type deduced from their initializer, so they must be initialized. That force to avoid a host of uninitialized variable problems.

Said highway lacks the potholes associated with declaring a local variable whose value is that of a dereferenced iterator
```c++
template<typename It>
void dwim(It b, It e)
{
    while (b != e)
        auto currValue = *b;
}
```

Because `auto` uses type deduction, it can represent types known only to compilers:
```c++
auto derefUPLess = 
    [](const std::unique_ptr<Widget>& pr1,
       const std::unique_ptr<Widget>& pr2)
    { return *p1 < *p2; };
```

In C++14 lambda expressions may involve `auto`:
```c++
auto derefLess = [](const auto& p1, const auto& p2)
                 { return *p1 < *p2; };
```

`auto`-declared objects functions invoking here can replace `std::function`, which is generally bigger, slower and may yield out-of-memory exceptions.

>`std::function` is a template in the C++11 STL that generalizes the idea of a function pointer. Whereas function pointers can point only to functions, however, `std::function` objects can refer to any callable object, i.e., to anything that can be invoked like a function. Because lambda expressions yield callable objects, closures can be stored as `std::function` objects. Without using `auto` it looks like 
```c++
std::function<bool(const std::unique_ptr<Widget>&,
                   const std::unique_ptr<Widget>&)>
    derefUPLess = [](const std::unique_ptr<Widget>& pr1,
                     const std::unique_ptr<Widget>& pr2)
                  { return *p1 < *p2; };
```
>The type of a `std::function`-declared variable holding a closure is an instantiation of the `std::function` template, and that has fixed size for any given signature, This size may not be adequate for the closure it's asked to store, and when that's the case, the `std::function` constructor will allocate heap memory to store the closure. The result is that the `std::function` object typically uses more memory than the `auto`-declared objects, thanks to implementation details that restrict inlining and yields indirect function calls, invoking a closure via `std::function` object is almost certain to be slower than calling it via an `auto`-declared object. 

`auto` can also improve code portability. 
>For example, the official return type of `std::vector<int> v; v.size();` is `std::vector<int>::size_type`, it's specified to be an unsigned integral type, so a lot of programmers figure that `unsigned` is good enough and use `unsigned sz = v.size();`. However, on 32-bit Windows both `unsigned` and `std::vector<int>::size_type` are the same size, but on 64-bit Windows `unsigned` is 32 bits, while `std::vector<int>::size_type` is 64 bits. 

`auto` can prevent implicit conversations that might be unwanted or unexpected due to type mismatch (which can also cause performance issues).
> `std::unordered_map` key part is `const`, so the type of `std::pair` (which is what `std::unordered_map` is). In case for 
```c++
std::unordered_map<std::string, int> m;
for (const std::pair<std::string, int>& p : m) ...
```
> it's not `std::pair<std::string, int>`, it's `std::pair<const std::string, int>`. As result, compilers will strive to find a way to convert `std::pair<const std::string, int>` objects to `std::pair<std::string, int>` objects. They'll succeed by creating a temporary object of type that `p` wants to bind to by coping each object in `m`, then binding the reference `p` to that temporary object.

### Things to Remember 
+ `auto` variable must be initialized, are generally immune to type mismatches that can lead to portability or efficiency problems, can ease the process of refactoring, and typically require less typing than variables with explicitly specified types. 
+ `auto`-typed variable are subject to the pitfalls due to deduction like template or deduction "proxy" types. 



## Item 6: Use the explicitly typed initializer idiom when `auto` deduces undesired types. 
Through `std::vector<bool>` conceptually holds bools, `operator[]` doesn't return a reference to an element of container (which is what it returns for every type except bool). Instead, it returns an object of type `std::vector<bool>::refrence`(a class nested inside `std::vector<bool>`). It's exist because C++ forbids references to bits. Not able to return a `bool&`, `operator[]` returns an object that _acts_ like a `bool&`.
As result, `auto highPriority = features(w)[5]` might be not what it's expected to (`bool highPriority = features(w)[5]`).
Furthermore, it will copy of `std::vector<bool>::reference` temporary object, which in at the end of statement can be destroyed, therefore `highPriority` contain a dangling pointer, and that's cause of undefined behavior. 

`std::vector<bool>::reference` is an example of a _proxy class_: class that exists for the purpose of emulating and augmenting the behavior of some other type. Smart pointers types are also proxy classes. 

As a general rule, "invisible" proxy classes don't play well with `auto`. Objects of such classes are often not designed to live longer than a single statement, so creating variables of those types tends to violate fundamental library design assumptions. 

The solution is to force a different type deduction, _the explicitly typed initializer idiom_. 
`auto highPriority = static_cast<bool>(features(w)[5]);`

### Things to Remember 
+ "Invisible" proxy types can cause `auto` to deduce the "wrong" type for an initializing expression
+ The explicitly type initializer idiom forces `auto` to deduce wanted type
