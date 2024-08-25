Most common way to get errors from OpenGL is use [glGetError](https://docs.gl/gl4/glGetError). 
But also exist [glDebugMessageCallback](https://docs.gl/gl4/glDebugMessageCallback)which provide more useful 
information but it's available only from OpenGL 4.3+. 

First thing about glGetError to notice - is that it's guarantee to return `GL_NO_ERROR` (which equal 0) and it's returns and clears an arbitrary error flag value. Thus, `glGetError` should always be called in a loop, until it returns `GL_NO_ERROR`.

As for starting point - we create two functions, one to clear possible errors from other functions and one to print error messages
```c++
void GlClearErrors()
{
	  while (glGetError());
}

void GlCheckErrors()
{
	while (auto error = glGetError()) 
	{
		std::cout << "[OpenGL error]" << error << std::endl;
	}
}

```
And when debug our program - simply 
```c++ 
GlClearErrors();
glDrawElements(GL_TRIANGLES, 6, GL_INT, nullptr);
GlCheckErrors();
```
Here we will get as output "1280", that's error code. Not much useful - to find what it's an exact error - we can search for value in OpenGL header, but in some cases it will be in hex, so we need to convert 1280 to 0x0500 which is `GL_INVALID_ENUM` (and in this example - we used `GL_INT` instead `GL_UNSINGED_INT`). 

We can improve it a little bit: 
```c++
#include <cassert>
#define DebugGlCall(x) while (glGetError());\
    x;\
    assert(GlLogCall());

bool GlLogCall()
{
    while (auto error = glGetError())
    {
        std::cout << "[OpenGL error]" << error << std::endl;
        return false;
    }
    return true;
}
```
and use as 
```c++
DebugGlCall(glDrawElements(GL_TRIANGLES, 6, GL_INT, nullptr));
```
As result we will get additional information about line of code etc (but cassert might bring some redundancy, in MSVC we can use something like `#define ASSERT(x) __debugbreak();` and add pretty prints manually with `GlLogCall(#x, __FILE__, __LINE__)`)
