```c++
class Singleton
{
public:
    static Singleton& Get(){ return *s_Instance; }
    void Print(){}

private:
    static Singlton* s_pInstance;
};

Singleton* Singleton::s_pInstance = nullptr;
int main()
{
    Singleton::Get().Print();
}
```
И схожий результат, но с применением локальных статика:
```c++
class Singleton
{
public:
    static Singleton& Get()
    {
        static Singleton instance;
        return instance;
     }
    void Print(){}
};

int main()
{
    Singleton::Get().Print();
}
```