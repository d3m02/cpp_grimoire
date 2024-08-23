#move #cpp11

Small example - string class and consumer of that class:
```c++
class String 
{
public:
	String() = default;
	String(const cahr* string)
	{
		m_size = strlen(string);
		m_data = new char[m_size];
		memcpy(m_data, string, m_size);
	}
	String(const Srtring& other)
	{ // Copy constructor
		m_size = other.m_size;
		m_data = new char[m_size];
		memcpy(m_data, other.m_data, m_size);
	}
	~String() { delete m_data; }
	
private:
	char* m_data {};
	uint32_t m_size {};
}

class Entity
{
public:
	Entity(const Strign& name)
		: m_name(name) {}
private: 
	String m_name{}; 
}

Entity entity(String("Example"));
// What's happen here - called String const char* constructor 
// then called copy constructor. 
```
