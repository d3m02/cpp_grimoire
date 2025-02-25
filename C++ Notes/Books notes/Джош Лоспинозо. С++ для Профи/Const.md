## Const аргументы 
Маркировка аргументам const исключает его модификацию в рамках функции 
```c++
void fun (const int t, int d)
{
	d = 0;   // ок
	t = 0;   // ошибка, нельзя менять t
}
```
## Const методы
Маркировка метода ключевым словом const означает, что мы даем обещание компилятору не изменять текущее состояние объекта в методе. Другим словами, эти методы предназначены только для чтения. Держатели постоянных ссылок и указателей не могут вызывать методы, которые не являются постоянными, поскольку методы, которые не являются постоянным, могут изменять состояние объекта.
```c++
class Age 
{
public:
	Age(int age) : m_age(age) {}
	int getValue() const 
	{ 
		m_age = 10; // - ошибка
		setValue(10); // ошибка, не конст метод
		return m_age; 
	}
	void setValue(int age) { m_age = age; }
private:
	int m_age{};
};

int main()
{
	const Age a1{18};
	a1.setValure(18); //Ошибка
	std::cout << a1.getValue() << std::endl; // ок  
	int age = a1.getValue(); // ок
}
```
## Const переменные-члены
Const переменные-члены не могут быть изменены после инициализации
```c++
class Age
{
	void func() { max++; } // ошибка 
	const int max = 40;
}
```