 > [Back to Basics: Understanding Value Categories - Ben Saks - CppCon 2019](https://www.youtube.com/watch?v=XS2JddPq7GQ)

First of all, values categories aren't really language features. We can't do overload as result. The values categories as a part of how compilers understands and intuprate expressions in the language (semantic properties of expressions).

In early C, there were two categories: lvalues and rvalues. The associated concepts were fairly simple. In C++ this concepts evolved futher more since in early C++ were added classes, const and references. In modern C++ were added rvalues reference, which made concepts more complicated, which is why in C++ added more value categories to specify the associated behaviors. 


In the [C Programming Language, Kernighan and Ritchie](https://en.wikipedia.org/wiki/The_C_Programming_Language) wrote
 > The name "lvalue" comes from the assignment expression 
 > `E1 = E2`, in which the left operand `E1` must be and the lvalue expression
 > An lvalue is an expression referring to ab object. 
 > An object is a region of storage.
 
_rvalue_ - an expression that's not an _lvalue_. 
From example `{cpp}x[i + 1] = abs(p-value);` we can see, that left operand of assignment can be only _lvalue_, while right operand = either _rvalue_ or _lvalue_

The reason of making this distinction between rvalues and lvalues is simply to help compilers assume what don't necessarily occupy storage thus code generations get more freedom to optimize code. 
For example,
```c++
int n; //a definition for an integer object named `n`
n = 1; //an assignment expression.
```

* `n` is sub-expresion referring to an `int` object. It's a _lvalue_.
* `1` is a sub-expresson not referring to and object. It's a _rvalue_
It's possible that compiler might place 1 as named data storage (as if 1 were a lvalue)
```asm
one:           ; a label for following location
	.word 1    ; allocate storage holding the value 1
	mov n, one ; copy the value at one to location n (n = 1;)
```
But also, compiler can use if available in architecture immediate operand (a source operand value can be part of an instruction)
```asm
	mov n, #1        ; copy the value 1 to location n
```

Since rvalue might not occupy space, rvalue can't be used on the left side of assignment expression (even if type is same - where to write value from right operand)
Most literals are rvalues:
* numeric literals, such as 3 and 3.14159
* character literals, such as 'a'
However, string literals such as "xyzzy" are lvalues since they are arrays somewhere in storage and have points to beginning.

lvalues can be used as rvalu, like assigning lvalue expression `n` to lvalue `m`. This is called _lvalue-to-rvalue conversion_.

lvalue and rvalue also apply for operators, like binary `{cpp}operator+`: obviously, those expressions must have suitable types, but each operand can be either a lvalue or rvalue. But result will be compiler-generated temporary rvalues (thus `n + m = 1` equal to `1 = n` error case).


Unary* operator yields to lvalue. A pointer `p` can point to an object, so `*p` is an lvalue. 
```c++
int a[N];
int *p = a;
*p = 3; // OK, since a - is valid existing object

char *s = nullptr;
*s = '\0'; // undifined behavior 
```


rvalues of class a little bit tricky. 
```c++
struct S { int x, y; }
S s1 { 1, 4 };
int i = s1.y;
```
and what compiler might generate - load address of s1 into r1, load s1.y into r2 via offset, store r2 into i.
```asm
s1:  .word 1       ; storage for s1.x
     .word 4       ; storage for s1.y
i:   .word 0       ; storage for i
s1a: .word s1      ; address of s1
ia:  .word i       ; address of i

ldr r1, s1a        ; load address of s1 into r1
ldr r2, [r1, #4]   ; load s1,y into r2 via offset
str r2, ia         ; store r2 into i
```
Now consider code like 
```c++
S foo();            //returns rvalue of type S
int j = foo().m_y;  // access m_y mamber of rvalue
```
Again, uses a base+offset calculation to access `{cpp}foo().y`, therefore, the return value of foo() must have a base address, which mean that it's occupies data storage. 

Also, enums (`{cpp}enum { MAX = 100 };` are rvalues: we can't assign to it anything, we can't take it's address. `MAX` yields an integer rvalue (like pointer but for rvalues?).
On the other hand, `{cpp}int const MAX = 100` - it's non-modifiable lvalue. Still we can't assign anything to it, but it's occupy memory and we can take pointer to it. 


lvalue and rvalues helps to explain #references. References behavior more like pointers in C++. A reference yields an lvalue. A reference is essentially a pointer that's automatically dereferenced each time it's used. 

| reference notation     | equivalent pointer notation                                                  |
| ---------------------- | ---------------------------------------------------------------------------- |
| `{cpp}int &ri = i;`    | `{cpp}int *const cpi = &i;`// `*const - can't change on what it's pointing ` |
| `{cpp}ri = 4;`         | `{cpp}*cpi = 4;`                                                             |
| `{cpp}int j = ri + 2;` | `{cpp}int j = *cpi + 2;*`                                                    |
Reason why pointers exist os to provide friendlier functions interfaces. More specifically, C++ has references so that overloaded operators can look just like built-in operators. 
For example, this code works in C, but not compile in C++
```c++
enum month { Jan, Feb, Mar, ... , Dec, month_end };
typedef enum month month;

for (month m = Jan; m <= Dec; ++m) ;
```
To make it work in C++, we need to overload `++` for `month`:
```c++ 
// by value - but it increments a copy of x. 
void operator++(month x) { x = static_cas<month>(x + 1); }
// also, this make ++Apr; compiles, but it shouldn't (similar to 42++)

// by address? - but it doesn't compile, can't overlad an operator with a para,eter of pointer type
void operator++(month* x) { *x = static_cas<month>(*x + 1); }
// even if comiles - ++&m - looks wrong. 


// by reference - solution 
month &operator++(month& x) { return x = static_cas<month>(x + 1); }
```

Also, 
* const references parameter will accept an argument that's either const or non-const, 
* non-const reference parameter will accept only a non-const argument 
* when it appears in an expression, a reference to const yilds a non-modifiable lvalue. 
* `{cpp}f(const T&) == f(T)`, since T is copy of object, thus modifying doesn't really change it. Passing by const ref might me more efficient than passing by value since making copy consume some time and memory. 

References similar to pointers, yields to lvalue, which mean 
```c++
int *pi = &3;     // can't take rvalue address 
int &ri = 3        // can't bind to rvalue 

int i;
double *pd = &i; // can't convert due to wrong type  
double &rd = i;  // can't bind 
```
There's an exception to the rule that a reference must bind to an lvalue of the referenced type: 
> a "reference to `const T` can bind to an expression `x` that's not an lvalue of type `T` if there's a conversion from `x`'s type to `T`.

Reason why this works - compiler creates a temporary object to hold a copy of `x` converted to `T`. For example `{cpp}const double& rd = 3;`, when program execution reaches this declaration, program converts the value of 3 from int to double, creates a temporary double to hold converted result and binds `rd` to that temporary, when execution leaves scope, program destroy the temporary. This insure that `{cpp}f(const& T) == f(T)` exactly the same. 


Merging information together, we can switch to modern C++ definition: 
there are actually two kinds of rvalues: 
* _prvalues_ or "pure rvalues", which don't occupy data storage 
* _xvalues_ or "expiring values", which occupy data storage. 
The temporary object is created through a temporary materialization conversion. It converts a prvalues to xvalue. 

Since C++11 in standard added also rvalue references. lvalue reference declaration use the &operator, a rvalue reference - &&operator. 
```c++
int&& r = 10;
double&& f(int&& ri);

const int&& rci = 20; //it's compiling, but almost doesn't make sence. 
```
rvalue reference bind only to rvalues. Binding rvalue reference to rvalue triggers a temporary materialization conversion. 
rvalue references in modern C++ uses mainly to implement #move_semantics. Binding rvalue refence to a rvalue creates xvalue. 
However, in move assignment operator 
```c++
string& string::operator=(string&& other)
{
//...
	string temp(other); //calls string (string const&)
//...
}
```
withing operator=m other exists for the duration of the function, in tat context it's an lvalue. In general, rule: if it has a name, it's a lvalue. 

It's safe to move from an lvalue only if it's expiring. 
Compiler can't always recognize and expiring lvalue, so to help compiler - we need to convert it to an xvalue, in other words, convert it to an unnamed rvalue reference. That's what `{cpp}std::move` does


Modern C++ value categories: 
				expression 
		  	/           \\
		     glvalue 	 rvalue
            /       \\     /      \\
		lvalue		xvalues     prvalue
		
* gvalue: generalized lvalues 
* prvalues: pure rvalue 
* xvalue: expiring lvalue