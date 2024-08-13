#copying 

Базовая вещь

`int a = 2; int b = a;`
 в `b` копируется значение `а`, это два разных участка памяти, изменение `b` не изменит `a`

В случае, если
```c++
Entity* a = new Entity();
Entity* b = a;
```
`a` теперь указатель и при присвоение `b` значения a – копируется указатель, а не все содержимое класса `Entity`. Но при этом изменение `b` окажет влияние и на `а`, они оба ссылаются на один участок памяти, в котором выполняется изменение.

Ситуация с копированием указателей может вызвать проблемы при попытке удаления памяти уже удаленного ранее указателя – возникает исключение в runtime.
```c++
class String 
{
    char* m_Buffer;
    unsigned int m_Size;
    String (const char* string) 
    {
        m_Size = strlen (string);
        m_Buffer = new char[m_Size+1];
        memcpy (m_Buffer,string,m_Size);
    }
    ~String(){ delete[] m_Buffer; }
};

{
    String s1 = "Test";
    String s2 = s1; // Копируется указатели char* m_Buffer, а не содержимое по указателю_
}

// Выход за Scope, удаляется s1, удаляется m_Buffer
// Попытка удалить s2, но m_Buffer уже удален
```
Проблема решается с помощью Copy Constructor, который по умолчанию имеет вид `String(const String& other)`
По умолчанию он делает `memcpy`:
```c++
    String(const String& other) 
	    : m_Buffer(other.m_Buffer)
	    , m_Size(other.m_Size)
	{
	}
    // Или
    String(const String& other) 
    { 
	    memcpy (this, &other, sizeof(String));
	}
```
Конструкт копирования по умолчанию можно отключить, добавив `= delete`.
Чтобы решить проблему с копирование контекста по умолчанию – можно написать свой конструктор
```c++
 String(const String& other)
    : m_Size(other.m_Size) //m_Size не указатель и можно просто копировать
{    
	m_Buffer = new char [m_Size];
    memcpy (m_Buffer, other.m_Buffer, m_Size
}
```

С помощью конструктора копирования можно так же и отслеживать излишние копирования. К примеру, если используется функция для вывода строки и в качестве входного параметра используется не ссылка\указатель – будет проводится лишнее копирование.