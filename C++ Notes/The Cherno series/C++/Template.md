Шаблоны по факту – это способ позволить компилятору дописать код по определенным правилам.

К примеру, есть функция вывода текста. Для разных типов входных данных потребуется писать свои перегрузки, при это меняя только тип данных. Вместо копировать кода можно оставить только одну копию и использовать шаблон.
```c++
template <typename T>
void log(T value){ cout <<"Log: " << value << endl; }
```
Будет работать как вариант `log(5)`, но тогда могут возникать сложности с пониманием кода, поэтому можно так же использовать вызов `log<int>(5)`
Вместо `typename` можно так же указать `class` и это будет работать.
Другой пример – использовать для создания массива в `Stack`.
```c++
template <int N>
class Array
{
    int m_Array[N];
};
Array<5> arr;
```
Можно так же и объединить оба приема:
```c++
template <typename T, int N>
class Array
{
    T m_Array[N];
};
Array<int,5> arr;
```

