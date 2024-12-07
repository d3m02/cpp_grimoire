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