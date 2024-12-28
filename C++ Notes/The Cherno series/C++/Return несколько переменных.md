Классические варианты – поместить возвращаемые варианты в массив, в `std::array` \\ `std::vector`, в структуру\класс.

Так же существует `std::tuple` – класс, который позволяет возвращать несколько переменных.
```c++
std::tuple<char, int> fun1 ()
{
    return std::make_pair('a', 1);
}

std::tuple<char, int> res = fun1();

char resChar = std::get<0>(res);
int resInt = std:get<1>(res);
```
Подобным образом работает `std::pair`, но становится возможным использовать `res.first` `res.second`