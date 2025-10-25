## Template function to concatenate strings 
It's use [Fold expressions (since C++17)](https://en.cppreference.com/w/cpp/language/fold) to calculate string length to reserve. 
It's 1.7 times faster than concatenating using `std::string::operator+`/`std::string::append` and 1.3 times slower than initializing `std::string` with result after concatenating.  

```c++
template<typename... Ts>
std::string stringConcat(Ts&&... ts)
{
    const auto size = (std::string_view{ts}.size() + ...);
    std::string res;
    res.reserve(size);
    (res.append(ts), ...);
    return res;
}
```


## C++ suffixes and literals

| Suffix      | Type               | Example | Note                    |
| ----------- | ------------------ | ------- | ----------------------- |
| `u`/`U`     | unsigned int       | `42U`   |                         |
| `l`/`L`     | long               | `42L`   |                         |
| `ul`/`UL`   | unsigned long      | `42UL`  | Combination `u` and `l` |
| `ll`/`LL`   | long long          | `42LL`  | C++11+                  |
| `ull`/`ULL` | unsigned long long | `42ULL` | C++11+                  |
| `f`/`F`     | float              | `3.14f` |                         |
| `l`/`L`     | long double        | `3.14L` |                         |
| `z`/`Z`     | std::size_t        | `42z`   | C++23                   |
| `uz`/`UZ`   | std::size_t        | `42uz`  | C++23                   |
String literals 

| Suffix/prefix | Type                                         | Example        | note                                                                                                                         |
| ------------- | -------------------------------------------- | -------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| `s`           | `std::string`                                | `"hello"s`     | C++14, required using namespace `std::string_literals` / `std::string_view_literals` / `std::literals::string_view_literals` |
| `sv`          | `std::string_view`                           | `"hello"sv`    | C++17, required using namespace `std::string_literals` / `std::string_view_literals` / `std::literals::string_view_literals` |
| `u8`          | UTF-8 string, `char8_t[]`                    | `u8"hello"`    | C++11+, starting from C++20+ typeR `char8_t[]`                                                                               |
| `u`           | UTF-16 string, `char16_t[]`                  | `u"hello"`     | C++11+                                                                                                                       |
| `U`           | UTF-32 string, `char32_t[]`                  | `U"hello"`     | C++11+                                                                                                                       |
| `L`           | Wide string, `wchar_t[]`                     | `L"hello"`     |                                                                                                                              |
| `R"()"`       | `const char[]`                               | `R"(hello)"`   | C++11+, do not escape any character                                                                                          |
| `LR"()"`      | `const wchar_t[]`                            | `LR"(hello)"`  | C++11+, combination of `L` and `R`                                                                                           |
| `u8R"()"`     | `const char[]`/`const char8_t[]`(from C++20) | `u8R"(Hello)"` | C++11+                                                                                                                       |
| `uR"()"`      | `const char16_t[]`                           | `uR"(Hello)"`  | C++11+                                                                                                                       |
| `UR"()"`      | `const char32_t[]`                           | `UR"(Hello)"`  | C++11+                                                                                                                       |
Single characters

| Суффикс | Тип         | Пример  | Примечание                    |
| ------- | ----------- | ------- | ----------------------------- |
| `u8`    | UTF-8 char  | `u8'a'` | C++17+, тип `char8_t` (C++20) |
| `u`     | UTF-16 char | `u'a'`  | C++11+, тип `char16_t`        |
| `U`     | UTF-32 char | `U'a'`  | C++11+, тип `char32_t`        |
| `L`     | Wide char   | `L'a'`  | Тип `wchar_t`                 |

Chrono-literals (C++14, `std::chrono`)

| Suffix | Type         | Example  | Note                                    |
| ------ | ------------ | -------- | --------------------------------------- |
| `h`    | hours        | `24h`    | `using namespace std::chrono_literals;` |
| `min`  | minutes      | `60min`  |                                         |
| `s`    | seconds      | `30s`    |                                         |
| `ms`   | milliseconds | `100ms`  |                                         |
| `us`   | microseconds | `500us`  |                                         |
| `ns`   | nanoseconds  | `1000ns` |                                         |
| `y`    | years        | `2024y`  | C++20                                   |
| `d`    | days         | `7d`     | C++20                                   |

Complex numbers (C++14, `std::complex`), require using namespace `std::complex_literals` /`std::literals` / `std::literals::complex_literals`

| Suffix | Type                   | Example |
| ------ | ---------------------- | ------- |
| `i`    | `complex<double>`      | `3.0i`  |
| `if`   | `complex<float>`       | `3.0if` |
| `il`   | `complex<long double>` | `3.0il` |
User literals can be created with implementation of `operator""`:

```cpp
constexpr long double operator""_deg(long double deg) {
    return deg * 3.14159 / 180;
}

auto angle = 90.0_deg;  // пользовательский суффикс
```

**Note:** User literals can only start with `_`.

---
## C++ operators' signatures 
### Unary Arithmetic operators

|                     | Inside class              | Outside class              |
| ------------------- | ------------------------- | -------------------------- |
| Plus `+a`           | `T T::operator+() const;` | `T operator+(const T& a);` |
| Minus ` -a `        | `T T::operator-() const;` | `T operator-`              |
| Bitwise NOT<br>`~a` | `T T::operator~() const;` | `T operator~(const T& a);` |
### Binary Arithmetic operators
It's good practice to implement non-member binary operators overloads as callers of members assignment operator

|                                  | Inside class                          | Outside class                            |
| -------------------------------- | ------------------------------------- | ---------------------------------------- |
| Addition `a + b`                 | `T T::operator+(const T2& b) const;`  | `T operator+(const T& a, const T2& b);`  |
| Subtraction <br>`a - b`          | `T T::operator-(const T2& b) const;`  | `T operator-(const T& a, const T2& b);`  |
| Multiplication<br>`a * b`        | `T T::operator*(const T2& b) const;`  | `T operator*(const T& a, const T2& b);`  |
| Division `a / b`                 | `T T::operator/(const T2& b) const;`  | `T operator/(const T& a, const T2& b);`  |
| Remainder `a % b`                | `T T::operator%(const T2& b) const;`  | `T operator%(const T& a, const T2& b);`  |
| Bitwise AND `a & b`              | `T T::operator&(const T2& b) const;`  | `T operator&(const T& a, const T2& b);`  |
| Bitwise OR `a \| b`              | `T T::operator\|(const T2& b) const;` | `T operator\|(const T& a, const T2& b);` |
| Bitwise XOR `a ^ b`              | `T T::operator^(const T2& b) const;`  | `T operator^(const T& a, const T2& b);`  |
| Bitwise left <br>shift `a << b`  | `T T::operator<<(const T2& b) const;` | `T operator<<(const T& a, const T2& b);` |
| Bitwise right <br>shift `a >> b` | `T T::operator>>(const T2& b) const;` | `T operator>>(const T& a, const T2& b);` |

### Assignment operators

| Operator name                     | Inside class                       | Inside class                          |
| --------------------------------- | ---------------------------------- | ------------------------------------- |
| Simple `a = b`                    | `T& T::operator =(const T2& b);`   | N/A                                   |
| Addition `a += b`                 | `T& T::operator +=(const T2& b);`  | `T& operator +=(T& a, const T2& b);`  |
| Subtraction<br>`a -= b`           | `T& T::operator -=(const T2& b);`  | `T& operator -=(T& a, const T2& b);`  |
| Multiplication<br>`a *= b`        | `T& T::operator *=(const T2& b);`  | `T& operator *=(T& a, const T2& b);`  |
| Division `a /= b`                 | `T& T::operator /=(const T2& b);`  | `T& operator /=(T& a, const T2& b);`  |
| Remainder `a %= b`                | `T& T::operator %=(const T2& b);`  | `T& operator %=(T& a, const T2& b);`  |
| Bitwise AND `a &= b`              | `T& T::operator &=(const T2& b);`  | `T& operator &=(T& a, const T2& b);`  |
| Bitwise OR `a \|= b`              | `T& T::operator \|=(const T2& b);` | `T& operator \|=(T& a, const T2& b);` |
| Bitwise XOR `a ^= b`              | `T& T::operator ^=(const T2& b);`  | `T& operator ^=(T& a, const T2& b);`  |
| Bitwise left <br>shift `a <<= b`  | `T& T::operator <<=(const T2& b);` | `T& operator <<=(T& a, const T2& b);` |
| Bitwise right <br>shift `a >>= b` | `T& T::operator >>=(const T2& b);` | `T& operator >>=(T& a, const T2& b);` |
### Increment/decrement operators

|                      | Inside class            | Outside class              |
| -------------------- | ----------------------- | -------------------------- |
| Pre-increment `++a`  | `T& T::operator++();`   | `T& operator++(T& a);`     |
| Pre-decrement `--a`  | `T& T::operator--();`   | `T& operator--(T& a);`     |
| Post-increment `a++` | `T T::operator++(int);` | `T operator++(T& a, int);` |
| Post-decrement `a--` | `T T::operator--(int);` | `T operator--(T& a, int);` |
### Logical operators

|                         | Inside class                               | Outside class                                 |
| ----------------------- | ------------------------------------------ | --------------------------------------------- |
| Negation `!a`           | `bool T::operator!() const;`               | `bool operator!(const T &a);`                 |
| AND `a && b`            | `bool T::operator&&(const T2 &b) const;`   | `bool operator&&(const T &a, const T2 &b);`   |
| Inclusive OR `a \|\| b` | `bool T::operator\|\|(const T2 &b) const;` | `bool operator\|\|(const T &a, const T2 &b);` |
### Comparison operators
Implement `operator<` (it's actively used in STL), `operator==` (possible from `operator<`, but for simplicity and better separately) and from them other operators. 

|                                           | Inside class                            | Outside class                              |
| ----------------------------------------- | --------------------------------------- | ------------------------------------------ |
| Equal to `a == b`                         | `bool T::operator==(const U& b) const;` | `bool operator==(const T& a, const U& b);` |
| Not equal to `a != b`                     | `bool T::operator!=(const U& b) const;` | `bool operator!=(const T& a, const U& b);` |
| Less than `a < b`                         | `bool T::operator<(const U& b) const;`  | `bool operator<(const T& a, const U& b);`  |
| Greater than `a > b`                      | `bool T::operator>(const U& b) const;`  | `bool operator>(const T& a, const U& b);`  |
| Less than <br>or equal to `a <= b`        | `bool T::operator<=(const U& b) const;` | `bool operator<=(const T& a, const U& b);` |
| Greater than <br>or equal to `a >= b`     | `bool T::operator>=(const U& b) const;` | `bool operator>=(const T& a, const U& b);` |
| Three-way comparison <br>C++20  `a <=> b` | `R T::operator<=>(const U& b) const;`   | `R operator<=>(const T& a, const U& b);`   |

### Member access operators

|                                          | Inside class              | Outside                     |
| ---------------------------------------- | ------------------------- | --------------------------- |
| Subscript `a[b]`                         | `R& T::operator[](S b);`  | N/A                         |
| C++23 subscript `a[...]`                 | `R& T::operator[](...);`  | N/A                         |
| Indirection `*a`                         | `R& T::operator*();`      | `R& operator*(T a);`        |
| Address-of `&a`                          | `R* T::operator&();`      | `R* operator&(T a);`        |
| Member of object `a.b`                   | not overloadable          | not overloadable            |
| Member of pointer `a->b`                 | `R* T::operator->();`     | N/A                         |
| Pointer to <br>member of object `a.*b`   | not overloadable          | not overloadable            |
| Pointer to <br>member of pointer `a->*b` | `R& T::operator->*(S b);` | `R& operator->*(T a, S b);` |
### Other operators

|                                  | Inside class                                | Outside class                       |
| -------------------------------- | ------------------------------------------- | ----------------------------------- |
| Function call `a(a1, a2)`        | `R T::operator()(Arg1 &a1, Arg2 &a2, ...);` | N/A                                 |
| Comma `a, b`                     | `T2& T::operator,(T2 &b);`                  | `T2& operator,(const T &a, T2 &b);` |
| conditional operator `a ? b : c` | not overloadable                            | not overloadable                    |

---
