#Union
Похож на `class`\\`struct`, но может иметь в себе только одну переменную. Если объявлено несколько переменных – они все будет указывать на один участок памяти. В основном `union` используют для назначения разных названий типов для одной переменной, условно класс вектора с `X, Y, Z` и с помощью `union` может ему дать так же название как `RGB`. Иными словами, `union` дают возможность по разным типам данных обращаться к одному участку памяти.
```c++
struct Union
{
    union 
    {
        float a;
        int b;
    };
};

Union u;
u.a = 2.0f;
cout << u.a << "," << u.b << end; //2, 1073741824

```
Более продвинутый пример
```c++
struct Vector2 {float x,y;}
//struct Vector4 { float x,y,z,w;}
// У нас есть функция вывода Vector2 и Vector4 по сути\два Vector2

struct Vector4  {
    union 
    {
        struct 
        {
            float x, y, z, w;
        };
        struct 
        {
            Vector2 left, right;
        };
    };
} 

void printVector2 (const Vector2& vector) 
{
    cout << vector.x << "," << vector.y << endl;
}

Vector4 vector = { 1.0f, 2.0f, 3.0f, 4.0f };
float test = vector.x; // все еще работает, т.к. структруы анонимные

PrintVector2 (vector.a);
```