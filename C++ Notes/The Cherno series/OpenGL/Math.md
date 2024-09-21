In graphics math mostly used matrices (set of numbers) and vectors. While in 'traditional' math vector - is some direction and magnitude(length), in graphics programming vector also can be positions. 
Most important usage of vectors and matrices is transformations. Scale, rotation, translation - all that staff. One useful implementation of math functions is GLM. 

Simple example of using math in OpenGL is fixing window aspect ration impact on rendered object. When we draw something, we actually need to place it in some plane/surface instead drawing bare vertices. To transform from 'raw' coordinates to 'window plane' coordinates we can use projections. 
`glm::mat4 proj = glm::ortho();` will create [Orthographic matrix](https://en.wikipedia.org/wiki/Orthographic_projection), for left/right/top/bot we need to somehow specify 3:4 aspect ration (like -2.0f, 2.0f, -1.5f, 1.5f), near/far we don't need here, so (-1.0f, 1.0f) will suits here. Here we defined that we have 3 units in distance from top to bottom and 4 units from left to right. 
Now we can send this matrix in vertex shader (let's call it **Model View Projection** matrix). And we just multiply vertex positions with that matrix.

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

**Projections** are one of biggest problem met in graphics programming, this is part of transformations about how to go from having some kind of arbitrary coordinate system in out 3D/2d world to some positions in window (normalized device coordinates). We could work in that coordinates providing to shaders coordinates in that space (in range -1.0 - 1.0), but than we should worry about calculation (like what it screen is 16:9 instead 4:3 or if resolutions different). Transformation projection allow us to work in suitable for us coordinates (for example, in pixel coordinates). In 3D graphics it's more important since we need somehow also handle perspective. \

Typical examples of projects type are 
* Orthographic
	* usually for 2D, further objects looks bigger 
* Perspective 
	* usually for 3D,  further objects looks smaller

Important note is it mentioned types not "only for 2D/3D" - they can be used for both cases. For example 2D game might use perspective projections to create scene depth, while 3D can you orthographic projections for rendering UI.

Here in glm::ortho we technically define our coordinate system boundaries:  
```c++
   glm::ortho(-2.F,  2.F,   //X 
              -1.5F, 1.5F,  //Y
              -1.F,  1.F);  //Z
```
Coordinates outside that boundaries will not be rendered, while inside - will be converted to normalized device coordinates.

View transformation is view of camera. Model transformation - is transformation (translation, rotation, scale) of model itself. 

In OpenGL camera as it is doesn't exist. When we simulate camera, it's actually something reverse to out scene. If we want to move camera right - we need to move scene left and vice versa. In that case moving camera left with view matrix will look like
```c++ 
glm::mat4 view = glm::translate(glm::mat4(1.F), glm::vec3(100, 0, 0));
```

As for simple model matrix, we can implement move of object. Move similar to camera example, but with positive values: move 200 up and 200 right will be 
```c++
glm::mat4 model = glm::translate(glm::mat4(1.F), glm::vec3(200, 200, 0));
```

Now we combine everything to one 
```c++
    glm::mat4 mvp = proj * view * model;
    shader.SetUniformMat4f("u_MVP", mvp);
```