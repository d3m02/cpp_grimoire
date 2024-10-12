>[CppCon 2015: Bjarne Stroustrup “Writing Good C++14”](https://youtu.be/1OEu9C51K2A?si=mZs5JN17QzawT4SR)

> Disclaimer: it's just what I found interesting/useful for myself, not full conspectus of record.

## Dangling pointer problem
Problem where  delete called with pointer where function is not owning pointer. It general ownership problem is about when pointer can be used without doubts that it was deleted.
```c++ 
void f(Obj* ptr)
{
	//...
	delete p;
}

void g()
{
	Obj* ptr = new Obj;
	f(ptr);
	//... Some other staff
	ptr->use(); // - Ouch, ptr deleted!;
}
```
Problem solved with set of rules, like `Distinguish owner from non-owners`, `Assume raw pointers to be nont-owners`, `catch all attempts for a pointer to escape into a scope enclusing its owner's scope (return, throw, out-parameters, long-lived containers, ..)`, `something that holds an owner is an owner (e.g. vector<owner<int*>>m owner<int*>[]`. 

Representing ownership quite tricky thing. Language by default doesn't have it, since it's require extra bit redundancy, breaking ABI, require rewrite lots of code. #GSL provide owner implementation  - it's idea just a handler for static analysis, cost free, ABI compatible, simply `template<typename T> owner = T`

```c++
void func (gsl::owner<int*>); // func requirs an owner

void bar(gls::owner<int*> o_ptr, int* raw_ptr, 
		 owner<int*> o_ptr2, int* raw_ptr2)
{
	func(o_ptr); // OK: transfer ownership
	func(raw_ptr); // bad: not an owner
	delete o_ptr2; // necessary
	delete raw_ptr2; // bad: not an onwer.
}
```

Another simple rules:
```c++
int* f(int* p)
{
	int x = 4;
	return &x; // No, it is pointer to variable destroyed after leaving scope 
	return new int{7}; // Ok, but where to delete? 
	return p; // Ok, pointer come from caller 

	// don't mix different owenership in an array
	int* q = new int{7};
	vector<int*> res {p, &x, q}; // Bad {unknown, pointer to local, owner}
}
```

## Range errors 
GSL array_view<T\>
```c++
// common style
void f(int*p, int n)
{
	p[7] = 9; // no check is index 7 exsit 
	for (int i = 0; i < n; ++i) p[i] = 7; // maybe ok
}
 // better 
 void f(array_view<int>a)
{
	a[7] = 9; // OK? checkable against a.size()
	for (itn x : a) a = 7; // OK
}
 
```
Advantage for array_view - simpler than common style (can be use `f(a)` instead `f(a, 100)`), shorter, at least as fsat, sometimes using the GSL, sometimes using STL

## nullptr problems 
Pointers can be nullptr. But this raise some questionable things: what if call `func(nullptr)`, is it safe in that function? possibly function can have `if (ptr == nullptr)`, but caller can't be sure about that, also it's require additional checks in functions and cost is crash in runtime. 
GLS provide `not_null<T>` which can help static analysis to catch `func(nullptr)` and extra checks for nullptr not necessary now.

## (Mis)uses of smart pointers 
+ smart pointers (like `shared_ptr` ) can be expensive
+ can mess up interface fore otherwise simple functions (`unique_ptr`, `shared_ptr`)
+ often, we don't need a pointer (Scoped objects prevered, we need pointer for OO interfaces and when we need to change the object referred to)

In this example smar pointers redundant?
```c++ 
void f(X*); // just uses X; no ownership transfer or sharing 
void g(std::shared_ptr<X>); // just uses X - bad 
std::unique_ptr<X> h(std::unique_ptr<X>); // just use X - bad (give pointer back to provent destruction)

void use()
{
	auto p = std::make_shared<X>P{};
	f(p.get());  // extract raw pointer 
	g(p);        // mess with use count (probably a mistake)
	auto q = h(std::make_unique<X>(p.get())); // transer ownership to use (a mistake), extact raw pointer, then wrap it and copy prevent destruction.
}
```

