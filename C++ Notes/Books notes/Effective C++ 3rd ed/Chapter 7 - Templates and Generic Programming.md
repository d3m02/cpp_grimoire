## Item 41: Understand implicit interfaces and compile-time polymorphism. 
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
	vpod sendCleartext(const std::string& msg);	
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