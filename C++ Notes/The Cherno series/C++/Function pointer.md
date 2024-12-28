`Function pointer` по сути позволяет назначить функцию в переменную.
```C++
void hw(int i) { cout << "Hello world" << endl; }

auto smth = hw(); // Не работает, возвращаемый тип void 
auto smth = hw;   // Работает, теперь указатель на фцунию
// auto раскроется в тип void(*smth)(int); 

//Чтобы это не выглядило так странно
typedef void(*smthFuncType)(int);
smthFuncType smth = hw;
smth(5);

```
Полезный пример:
```c++
void PrintValue (int value) 
{ 
	std::cout << "Value: " << value << std::endl;
}

void ForEach(const std::vector<int>& values, void (*func)(int))
{
    for (int value : values) 
	    func (value);
}

std::vector<int> values = {1, 5, 6, 4};
ForEach(values, PrintValue);
// С использованием лямбды, т.е. PrintValue можно удалить и будет работать 
ForEach(values, [](int val){ std:: cout << "Value: "<< val << std::endl;} )

```