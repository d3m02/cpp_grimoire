## Item 32: Make sure public inheritance models "is-a".
Every object of derived class is also an object of base class, but not vice versa. Base class represents a more general concept than derived; derived represents a more specialized concept than base. Inheritance "asserting" that anywhere an object of base class be used, an object of derived class can be used as well. Derived - is a Base. 



## Item 33: Avoid hiding inherited names. 
Let's take example 
```c++
class Base {
public: 
	virtual void mf1() = 0;
	virtual void mf2();
	void mf3();
};
class Derived : public Base {
public: 
	virtual void m1();
	void mf4();
};
```
Suppose that in derived class `mf4` implemented as 
```c++
void Derived::mf4()
{
..
	mf();
}
```

When compilers see the use of name `mf2`, they have to figure out what it refers to. They do that by searching scopes for a declaration of something named `mf2`. First they look in the local scope, then search the containing scope (that of the class `Derived`), then move on to the next containing scope (base class) - here they will find `mf2`. If there were no `mf2` - search would continue, first to namespace(s) and finally to the global scope. 

Now we add in example overload functions 
```c++
class Base {
public: 
	virtaul void mf1() = 0;
	virutal void mf1(int);
	void mf3();
	void mf3(double);
};
class Derived : public Base {
public: 
	virtual void m1();
	void mf3();
	void mf4();
};
```
All functions named `mf1` and `mf3` in the base class are hidden by the functions named `mf1` and `mf3` in the derived class now. 
```c++
Derived d;
d.mf1(); // fine, calls Derived::mf1
d.mf1(2); // error, Derived::mf1 hides Base::mf1
d.mf2(); //fine, calls Base::mf2
d.mf3(); // fine, calls Dervied::mf3
d.mf3(2); // error, Derived::mf3 hides Base::mf3
```

The rationale behind this behavior is that it prevents from accidentally inheriting overloads from distant base classes when created a new derived class. Unfortunately, typically expected to inherit the overloads, in fact if using public inheritance but not inherit the overloads, this is violating the is-a relationship. 

For fixing used `using` keyword:
```c++
class Derived : public Base {
public: 
	using Base::mf1;
	using Base::mf3;
	
	virtual void m1();
	void mf3();
	void mf4();
};
```

Sometimes we won't want to inherit all the functions from base classes. Under public inheritance, this should be never be the case. Under private inheritance, however, it can make sense.  
For example, suppose Derived privately inherits from Base, and the only version of mf1 that Derived wants to inherit is the one taking no parameters. Here can be used _a simple forwarding function_ technique (inline forwarding function)
```c++
class Base {
public: 
	virtaul void mf1() = 0;
	virutal void mf1(int);
};
class Derived : private Base {
public: 
	virtual void m1()
	{ Base::mf1(); }
};
```




## Item 34: Differentiate between inheritance of interface and inheritance of implementation. 
As a class designer, sometimes require to allow for derived classes to inherit only the interface (declaration) of a member function; sometimes derive both a function's interface and implementation, but allow them to override the implementation they inherit; sometimes require for derived classes allow only to inherit a function's interface and implementation without allowing to override anything. 

+ The purpose of declaring a pure virtual function is to have derived classes _inherit a function interface only_
```c++
class Shape {
public:
	// pure virtual function
	virtual void draw() const = 0;
}
```

+ The purpose of declaring a simple virtual function is to have derived classes _inherit a function interface and default implementation_
```c++
class Shape {
public:
	virtual void error(const std::string& msg);
}
```
Sometimes it can be dangerous to allow simple virtual functions to specify both a function interface and a default implementation. For example if plane class `Model A` and `Model B` both are flown in exactly same way and fly function - but into `Airplane` base class as simple virtual function, but then added new, `Model C` that flown differently and fly function wasn't redefined my mistake - this will be attempt to made `Model C` fly as `Model A`. 
As solution can be used separate protected non-virtual function 
```c++
class Airplane {
public: 
	virtual void fly() = 0;
protected:
	void defaultFly();
}
class ModelA : public Airplane {
public:
	virtual void fly() { defaultFly(); }
}
```

Or provide default implementation for pure virtual function:
```c++
class Airplane {
public: 
	virtual void fly() = 0;
}
void Airplane::fly () 
{
	// default code for flying
}
class ModelA : public Airplane {
public:
	virtual void fly() { Airplane::fly(); }
}
```


A non-virtual member function specifies an _invariant over specialization_, because it identifies behavior that is not supposed to change. 
+ The purpose of declaring a non-virtual function is to have derived classes _inherit a function interface as well as mandatory implementation_.
```c++
class Shape { 
public:
	int objectID() const;
};
```



## Item 35: Consider alternative to virtual functions. 
#### The Template Method Pattern via the Non-Virtual Interface Idiom

^4b7e05

_Non-virtual interface (NVI) idiom_ - having clients call private virtual functions indirectly through public non-virtual member functions.
```c++
class GameCharacter {
public:
	int healthValue() const 
	{
		//...
		int retVal = doHealthValue();
		//...
		return retVal;
	}
private: 
	virtual int doHealtValue() const { /* ... */ }
}
```
It' a particular manifestation of the more general design pattern _Template Method_ (nothing to do with C++ templates).
An advantage - wrapping virtual call with guaranteed "do before" and "do after" (it can be mutex lock, setting context, validation etc). The NVI idiom give derived classes control how functionality is implemented, but the base class reserves for itself the right to say when the function will be called. 

#### The Strategy Pattern via Function Pointers 
_Strategy pattern_ - design pattern when function pointer passed as argument to function. 
```c++
class GameCharacter {
public:
	typedef int(*HealthCalcFunc)(const GameCharacter&)
	explicit GameCharacter(HeathCalcFuntion hcf = defaultHeatlhCalc)
		: healthFunc(hcf)
	{}
	
	int healthValue() const { return healthFunc(*this); }
private:
	HealthCalcFunc healthFunc;
};
```
This approach offer interesting flexibility: 
+ Different instances of the same character type can have different health calculation functions
+ Health calculation function for a particular character may be changed at runtime. 
On the other hand, since health calculation function is no longer member function, it has no special access to the internal parts of the object. 
Also, for Strategy approach can be used `std::function` wrapper.

#### The "Classic" Strategy Pattern
```c++
class GameCharacter; // forward declaration
class HealthCalcFunc { 
public:
	virtual int calc(const GameCharacter& gc) const { .. }
};
HealthCalcFunc defaultHeathCalc;
class GameCharacter {
public:
	explicit GameCharacter(HealthCalcFunc* phcf = &defaultHealthCalc)
		: m_pHealthCalc(phcf)
	{}
	int healthValue() const { return pHealthCalc->calc(*this); }
private:
	HealthCalcFunc* m_pHealthCalc;
};
```
This also offers the possibility to tweak health function by adding a derived class. 



## Item 36: Never redefine an inherited non-virtual function. 
This can cause some unexpected for user behavior when accessing such function through pointer to base class 
```c++
class B {
public:
	void mf();
 };
class D: public B {
public: 
	 void mf(); 
};
D x;
B* pB = &x;
pB->mf();    //calls B::mf

D* pD = &x;
pD->mf();    //calls D::mf
```
This violate is-a rule or rule that class D must inherit both the interface and the implementation of mf, because mf is non-virtual in B.



## Item 37: Never redefine a function's inherited default parameter value. 
Virtual functions are dynamically bound (late binding), but default parameter values are statically bound(early binding). What can happen - invoking a virtual function defined in a derived class but using a default parameter value form a base class.  
Reason why C++ act in such manner is runtime efficiency. If default parameter values were dynamically bound, compilers would have to come up with a way to determine the appropriate default value(s) for parameters of virtual functions at runtime, which would be slower and more complicated.

[[#The Template Method Pattern via the Non-Virtual Interface Idiom]] can provide solution.
```c++
class Shape {
public:
	enum ShapeColor { Red, Green, Blue };
	void draw(ShapeColor color = Red) const
	{
		doDraw(color);
	}

private:
	virtual void doDraw(ShapeColor color) const = 0;
};

class Rectangle: public Shape {
public:
...
private:
	virtual void doDraw(ShapeColor color) const;
};
```




## Item 38: Model "has-a" or "is-implemented-in-term-of" through composition. 
_Composition_ is the relationship between types that arises when objects of one type contain objects of another type. Composition means either "has-a" or "is-implemented-in-term-of". 

Simple "has-a" example:
```c++
class PhoneNumber { };

class Person {
public:
	...
private:
	std::string name;
	PhoneNumber voiceNumber;z
}
```

"is-implemented-in-term-of" might be something like 
```c++
template<class T>
class Set {
public: 
	bool member(const T& item) const;
	void insert(const T& item);
	void remove(const T& item);

	std::size_t size() const;
private: 
	std::list<T> m_represent;
};

template<typename T>
bool Set<T>::member(const T& item) const
{
	return std::find(m_represent.begin(), m_represent.end(), item) != rep.end();
}
template<typename T>
bool Set<T>::insert(const T& item)
{
	if (!member(item)) m_represent.push_back(item);
}
...
```