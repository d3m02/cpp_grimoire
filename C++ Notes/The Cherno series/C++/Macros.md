Макросы дают некоторые возможности для автоматизации некоторых операций в припроцессе. Макрос заменяет некоторый текст на другой, генерируемый с разными параметрами и проч.
```c++
#define WAIT std::cin.get()
{
    WAIT;
}

#ifdef _DEBUG
#define LOG(x) std::cout<<x<<std::endl
#else
#define LOG(x)  // в Release заменит на "ничего"
#endif

{
    LOG ("Hello");
}

```