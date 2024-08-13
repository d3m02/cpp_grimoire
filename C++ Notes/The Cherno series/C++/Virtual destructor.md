#virtual #destructor
Сценарий, когда два класса, один наследник другого
```c++
class Base 
{
public:
    Base(){ cout << "Base Constructor\n"; }
    ~Base(){cout << "Base Destructor\n"; }
}
class Derived : public Base 
{
public:
    Derived(){ cout << "Derived Constructor\n"; }
    ~Derived(){ cout << "Derived Destructor\n"; }
}
```
Рассмотрим две ситуации: тут результат соответствует ожиданию
```c++
Base* base = new Base();
delete base;
cout <"---\n";
Derived* derived = new Derived();
delete derived;

/*
Base Constructor 
Base Destructor
----
Base Constructor
Derived Constructor
Derived Destructor
Base Destructor
*/
```

Но если `Base` указателю дать `Derived` - то деструктор `Derived` не вызовется, т.е. будет утечка памяти.
```c++
Base* base = new Derived();
delete base;

/*
Base Constructor 
Derived Constructor
Base Destructor
*/
```
Решается с помощью виртуального деструктора. 