## Item 18: Make interfaces easy to use correctly and hard to use incorrectly. 
Developing interfaces that are easy to use correctly and hard to use incorrectly requires consider the kinds of mistake that clients might make.
For example simple Date class
```c++
class Date { 
public:
	Date(int month, int day, int year);
};
```
user can swap arguments and use constructor as `Date d(30, 2, 1995);`, user 
can pass an invalid value `Date d(2, 40, 1995);`.
This can be fixed with introduction of new types, like `struct Day;` `struct Month;`, `struct Year;` with `explicit` constructor, now `Date d(Day(30), Month(3), Year(1995));` will generate error since wrong types used.

Sometimes it's reasonable to restrict values of new defined types. For example enum can be used to restrict month, but enums are not as type-safe as we might like (like, enums can be used like `int`).
> Note: since C++11 available `enum class` which are don't have implicit int conversion and provide type safety. 

A safer solution is to predefine the set of all valid Months:
```c++
class Month{
public:
	static Month Jan() { return Month(1); }
	...
	static Month Dec() { return Month(12); }
private:
	explicit Month (int m);
}
Date d(Month::Mar(), Day(30), Year(1995));
```

Another way to prevent likely client errors is to restrict what can be done with a type. For example, adding `const` for `operator=` return value. Users already known how types like `int` behave, so better if custom crafted types behave the same way whenever reasonable. 

Function `Investement* createInvestment();` can cause give user two opportunities for error: failure to delete a pointer and perform deletion of the same pointer more than once. Raw pointer as return type can be replaced with `std::shared_ptr<Investement>`.

An especially nice feature of `std::shared_ptr` is that it automatically uses its per-pointer deleter to eliminate another potential client error, the "cross-DLL problem". This problem crops up when an object is created using `new` in one dynamically linked library, but is deleted in different DLL. On many platforms such case lead to runtime errors. `std::shared_ptr` avoids the problem, because its default deleter uses delete from the same DLL where it was created. 



## Item 19: Treat class design as type design. 
Virtually every classes requires that developer confront the following questions, answer to which often lead to constraints on design: 
* **How should objects of new type be created and destroyed?**
* **How should object initialization differ from object assignment?** 
* **What does it mean for objects of new type to be passed by value?**
* **What are the restrictions on legal values for new type?** 
* **Does new type fit into an inheritance graph?** (if it inherited from another class, should be paid attention to what is virtual and what not in base class, should it be allowed for other classes to inherit from new class)
* **What kind of type conversions are allowed in new type?**
* **What operators and functions make sense for the new type?**
* **What standard functions should be disallowed?**
* **Who should have access to the members of new type?**
* **What is the "undeclared interface" of new type?** What kind of guarantees does it offer with respect to performance, exception safety, resource usage. 
* **How general is new type?** Perhaps should be used template class.
* **Is a new type really what needed?** Perhaps better to achieve goals by simply defining one or more non-member functions or templates. 



## Item 20: Prefer pass-by-reference-to-const to pass-by-value.
Passing by reference to const makes code much efficient: not constructors or destructors are called and replacing in general doesn't change for user function behavior: non-const value lifetime limited with function scope thus user already don't except any change of object. But it's important to make pass-by-reference const, since otherwise callers would have to worry about changing object they passed in. 

Passing parameters by reference also avoids _slicing problem_. When a derived class object is passed (by value) as a base class object, the base class copy constructor is called, and the specialized features that make the object behave like a derived class object are "sliced" off.

Even for small types that little more than a pointer - copy of such types can cause copying everything they point to, which can be very expensive. Also, size of small user-defined types can change in future. 



## Item 21: Don't try to return a reference when you must return an object. 
Never return a pointer or reference to a local stack object, a reference to a heap-allocated object, or a pointer or reference to a local static object if there is a chance that more than one such object will be needed. It's simply still require calling constructor plus object can be destroyed.



## Item 22: Declare data members `private`.
Making data members private allow to reserve the right to change implementation decisions later. If don't hide such decisions, changing anything public will brake client code if they use direct access to members. Better to use setters and getters to limit amount of problems. 
Protected data members are thus as unencapsulated as public ones, because in both cases, if the data members are changed, an unknowably large amount of client code is broken. Practically speaking, unencapsulated means unchangeable, especially for classes that are widely used. Yet widely used classes are most in need of encapsulation, because they are the ones that can most benefit from the ability to replace one implementation with a better one



## Item 23: Prefer non-member non-friend functions to member functions. 
If something is encapsulated, it's hidden from view. The more something is encapsulated, the fewer things can see it. The fewer things can see it, the greater flexibility we have to change it, because our changes directly affect only those things that can see what we change. The greater something is encapsulated, then, the greater our ability to change it. 

Given a choice between a member function (which can access not only the private data of a class, but also private functions, enums, typedefs, etc,) and a non-member non-friend function (which can access none of these things) providing the same functionality, the choice yielding greater encapsulation is the non-member non-frind function, because it doesn't increase the number of functions that can access the private parts of the class.

In C++, a more natural approach would be to make a non-member functions in the same namespace as related object. Namespaces, unlike classes, can be spread across multiple source files. 
Note that this is exactly how the standard C++ library is organized. Rather than having a single monolithic header containing everything in the std namespace, there are dozens of headers (e.g. `<vector>`, `<algorithm>`, `<memory>`, etc), each declaring some of the functionality in std. Clients who use only vecotr-related functionality aren't rquired to include `<memory>`. This allows clients to be compilation dependent only on the parts of the system they actually use. Partitioning functionality in this way is not possible when it comes from a class's member functions, because a class must be defined in it's entirety; it can'y be split into pieces.  



## Item 24: Declare non-member functions when type conversions should apply to all parameters. 
```c++
class Rational {
public: 
	Rational(int numeratpr = 0, int denominator = 1);
};
```
It's require now for new type to add `operator*`. Design decision 
```c++
const Rational operator*(const Rational& rhs) const;
Rational oneHalf(1, 2);
```
will work with `result = oneHalf * 2;` (if constructor not marked as `explicit`), but will not work with `result = 2 * oneHalf;`
Instead, better to implement non-member function 
```c++
const Ratioanl operator*(const Rational& lhs, const Rational& rhs)
{ ... }
```



## Item 25: Consider support for a non-throwing swap.
`swap` function is a mainstay of exception-safe programming and a common mechanism for coping with the possibility of assignment to self. 
Typical implementation is (something similar implemented in STL) 
```c++
template<typename T>
void swap(T& a, T& b)
{
	T temp(a);
	a = b;
	b = temp;
}
```
As long objects support copying (via copy constructor and copy assignment operator), the default swap implementation will let objects be swapped without having to do any special work. But sometimes making 3 copies might be expensive and custom swap will be better. For example if class implemented with design _pimpl idiom_ (pointer to implantation) - it's enough to swap `pImpl`. 
```c++
class Widget {
public:
	void swap(Widget& other)
	{
		using std::swap;
		swap (pImpl, other.pImpl);
	}
private:
	WidgetImpl* pImpl;
};
namespace std {
	template<>
	void swap<Widget>(Widget& a, Widget& b)
	{
		a.swap(b);
	}
}
```

If `Widget` and `WidgetImpl` are class templates, implementation will be changed a little bit since with implementation above partial specialization will not compile. Instead we declare a non-member swap that calls the member swap, we just don’t declare the non-member to be a specialization or overloading of `std::swap`
```c++
namespace WidgetStuff {
	template<typename T>
	class WidgetImpl { ... };
	template<typename T>
	class Widget { ... };

	template<typename T>
	void swap(Widget<T>& a, Widget<T>& b)
	{
		a.swap(b);
	}
}
```

Adding `using std::swap` allow to make STL swap "visible".  When compilers see the call to swap, they search for the right swap to invoke. C++’s name lookup rules ensure that this will find any T-specific swap at global scope or in the same namespace as the type T. If no T specific swap exists, compilers will use swap in std, thanks to the using declaration that makes std::swap visible in this function. Even then, however, compilers will prefer a T-specific specialization of std::swap over the general template, so if std::swap has been specialized for T, the specialized version will be used. This also reason why better to hide non-member swap under namespace. 
