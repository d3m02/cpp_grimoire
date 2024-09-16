In graphics math mostly used matrices (set of numbers) and vectors. While in 'traditional' math vector - is some direction and magnitude(length), in graphics programming vector also can be positions. 
Most important usage of vectors and matrices is transformations. Scale, rotation, translation - all that staff. One useful implementation of math functions is GLM. 

Simple example of using math in OpenGL is fixing window aspect ration impact on rendered object. When we draw something, we actually need to place it in some plane/surface instead drawing bare vertices. To transform from 'raw' coordinates to 'window plane' coordinates we can use projections. 
`glm::mat4 proj = glm::ortho();` will create [Orthographic matrix](https://en.wikipedia.org/wiki/Orthographic_projection), for left/right/top/bot we need to somehow specify 3:4 aspect ration (like -2.0f, 2.0f, -1.5f, 1.5f), near/far we don't need here, so (-1.0f, 1.0f) will suits here. Here we defined that we have 3 units in distance from top to bottom and 4 units from left to right. 
Now we can send this matrix in vertex shader (let's call it ModelViewProjection matrix). And we just multiply vertex positions with that matrix.

```c++
#shader vertex
#version 330 core

layout (location = 0) in vec4 position;
layout (location = 1) in vec2 texCoord;

out vec2 v_TexCoord;

uniform mat4 u_MVP;

void main()
{
   gl_Position = u_MVP * position;
   v_TexCoord = texCoord;
};

```
Also we need to set uniform
```c++
    void SetUniformMat4f(const std::string& name, const glm::mat4& matrix)
    {
        glUniformMatrix4fv(GetUniformLocation(name), 
                           1,              // 1 matrix
                           GL_FALSE,       //tranpose not needed for GLM
                           &matrix[0][0]); //address of first element
    }

```