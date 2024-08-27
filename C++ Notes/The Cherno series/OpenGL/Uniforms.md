Uniforms are simply way to get from CPU(C++ code) into a shader (in GPU), so we can you then that data as a variable. Take for example color. We can set color via [[Vertex Buffers]] (attributes) as well.

First of all, uniforms set per draw (before call `glDrawElements\glDrawArrays`) 

1) We declare unform in shader (this also should be different name from output color.)
```c++ 
#shader fragment
layout(location = 0) out vec4 color;
uniform vec4 u_Color;

void main()
{
	color = u_Color;
}
```

2) After shader bind (so OpenGl will know in which shader send data) we need to get location id via [glGetUniformLocation](https://docs.gl/gl4/glGetUniformLocation). If function return -1 - it mean that OpenGL wasn't able to find our uniform. Note that -1 also can be returned if variable was unused. 
 ```c++
 glUseProgram(shader);
 int location = glGetUniformLocation(shader, "u_Color");
 assert(location != -1);
```
3) And after we ready for call [glUniform](https://docs.gl/gl4/glUniform) (series of glUniform{1|2|3|4}{f|i|ui} functins, f-float, i - int, ui - unsigned)
```c++
assert(location != -1);
glUniform4f(location, 0.2F, 0.3F, 0.8F, 1.0F);
```