Vertex arrays a little bit OpenGL special thing. It's a way to bind vertex buffers with some kind of specification for layout of that vertex buffer. 

In our example we have buffer which contain all vertex data 
```c++
unsigned int v_buff;
glGenBuffers(1, &v_buff);
glBindBuffer(GL_ARRAY_BUFFER, v_buff);
std::array vertices { -0.5F, -0.5F,   // 0
                       0.5F, -0.5F,   // 1
                       0.5F,  0.5F,   // 2
                      -0.5F,  0.5F }; // 3

glBufferData(GL_ARRAY_BUFFER, 
			 static_cast<GLsizeiptr>(sizeof(vertices)), 
			 vertices.data(),
             GL_STATIC_DRAW);
```

and we enabled attribute array where specified layout of data. 
```c++
glEnableVertexAttribArray(0);
glVertexAttribPointer(0, 2, GL_FLOAT, GL_FALSE, 
					  sizeof(float) * 2, nullptr);
```

Vertex array allow as to bind vertex specification that we specified (by using `glVertexAttribPointer`) to an actual buffer (or series of buffers) so this can help handling with explicit layout defining (`glVertexAttribPointer` actually not connected with `gl.*Buffer.*` and if we have multiple objects/buffers - we need to rebind/redefine layout before each draw?)

In general, each draw our draw flow something like 'bind shader -> bind vertex buffer -> set up vertex layout -> bind index buffer -> draw'. Instead, we change it to 'bind shader -> bind vertex array -> bind index buffer -> draw'. Vertex array will contain all actual state what we need.

Vertex arrays are actually mandatory, OpenGL create vertex array objects by default in Compatibility Profile (this what 0 means in `glEnableVertexAttribArray`?). The Core Profile doesn't create vertex arrays thus we need to explicitly create it. To enable Core Profile requires to add before creating window
```c++
glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 3);
glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 3);
glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE);
```
With Core profile we will get error in `glEnableVertexAttribArray` since no Vertex array object exist.  

The fix is quite simple: we create vertex array [glGenVertexArrays](https://docs.gl/gl4/glGenVertexArrays)(GLsizei n, GLuint \*arrays) and [glBindVertexArray](https://docs.gl/gl4/glBindVertexArray) it 
```c++
unsigned int vao;
glGenVertexArrays(1, &vao);
glBindVertexArray(vao);
```

So now in render loop we can replace 
```c++
glBindBuffer(GL_ARRAY_BUFFER, v_buff);
glEnableVertexAttribArray(0);
glVertexAttribPointer(0, 2, GL_FLOAT, GL_FALSE, 
					  sizeof(float) * 2, nullptr);
```
with
```c++
glBindVertexArray(vao);
```

So under hood OpenGL make bind when we specified `glVertexAttribPointer` in index 0 of vertex array `vao` with currently bind `GL_ARRAY_BUFFER`. 
If we call
```c++
glEnableVertexAttribArray(1);
glVertexAttribPointer(1, 2, GL_FLOAT, GL_FALSE, 
					  sizeof(float) * 2, nullptr);
```
we bind current `GL_ARRAY_BUFFER` and layout to index 1 in `vao` 

Last question is what use to better performance - one global VAO and one for each object - there is no clear answer, if it's requires to squish all performance from device - better to benchmark it with tests. In the past global VAO was faster, but currently it's not clear and depended from environment to environment 