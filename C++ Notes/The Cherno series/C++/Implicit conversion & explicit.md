#explicit #implicit_conversion 

– магия на уровне объявления класса `Entity new = 55`, где класс `Entity` имеет конструктор, принимающие значение `int` или вызывать функцию, которая имеет входным параметром ссылку на класс и конструктор с `int`, и в качестве входного параметра указать `int`
```c++
class Entity
{
public:
	Entity (int age)
		: m_age(age)
	{
	}

	void PrintEntity(const Entity& entity)
	{
	}
}

PrintEntity(22);
Entity new = 55;
```

`explicit`  – отменяет функционал неявного преобразования. Ставится перед конструктором класса

```c++
explicit Entity (int num)
```


Тогда `Entity new = 55` не будет работать, хотя можно перекантовать:  `Entity new = (Entity) 55;`