#cpp17
Новая фича в С++17 стандарте, позволяющая возвращать несколько типов данных. 
Проблема для возврата нескольких переменных решается с помощью `tuple`\\`pair`, но в случае с `tuple` существует проблема, что для того, чтоб «выцепить» переменные по отдельность – требуется использовать `std::get<0>`, `std::get<1>` и т.д., что слегка делает код не читаемым и запутанным, особенно если используется auto.
`Structured bindings` решает эту проблему следующим образом:
```c++
std::tuple <std::string, int> Fun() { return {"Fun", 24}; }

// Before
// std::tuple <std::string, int> test= Fun();
auto test = Fun ();
std::string test_str = std::get<0>(test);
int test_int = std::get<1>(test);

// after c++17
auto[test_str, test_int] = Fun();

```