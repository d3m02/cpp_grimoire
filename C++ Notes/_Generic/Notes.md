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