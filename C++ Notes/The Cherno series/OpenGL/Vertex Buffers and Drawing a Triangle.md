So for drawing a triangle with modern OpenGL requires vertex buffer and shader. 
Vertex buffer is just a buffer, vertex in name is almost meaningless. It's just blob of video card memory in which we can push bytes. After pushing data in buffer - before we call draw function we need to specify how to ineprete that data and how put it on screen. 
Shader - is program on GPU in which we describe how to intepret and work with data. 
OpenGL work as state machine: drawing process looks like "select this buffer, load such amout of bytes, select this shader, draw". 

1. Create buffer ([glGenBuffers](https://docs.gl/gl4/glGenBuffers))
```c++
/**
 * @param n       [in] - number of buffer names to be generated 
 * @param bufers [out] - id of genereated buffer 
 */
glGenBuffers(GLsizei n, GLuint* buffers);

GLuint buffer;
glGenBuffers(1, &buffer); 
```
2. Select buffer ([glBindBuffer](https://docs.gl/gl4/glBindBuffer))
```c++
/**
 * @param target [in] - purpose of buffer, should be one of value of 
 *                      enum 
 * @param buffer [in] - id of genered buffer
 */
glBindBuffer(GLenum target, GLuint buffer);

glBindBuffer(GL_ARRAY_BUFFER, buffer);
```
3. Specify data in buffer ([glBufferData](https://docs.gl/gl4/glBufferData))
```c++ 
/**
 * @param target [in] - same as target used in buffer selection function?
 * @param size   [in] - size in bytes of data to be pushed in buffer
 * @param data   [in] - pointer on data array
 * @param usage  [in] - expected usage pattern of the data store from gl 
 *                      enum
 */
glBufferData(GLenum target, GLsizeiptr size, const void* data, GLenum usage)

std::array<float, 6> vertices {-0.5F, -0.5F, 0.0F, 0.5F, 0.5F, -0.5F};
glBufferData(GL_ARRAY_BUFFER, 
            static_cast<GLsizeiptr>(sizeof(float) * vertices.size()),
            vertices.data(),
            GL_STATIC_DRAW);
```
`usage` - quite important thing, it's combination of `GL_<usage>_<access>`. In general it's hit for OpenGL. So `<usage>` can be:
* `STREAM` - contents will be modified once and used at most a few times (in general not used often)
* `STATIC` - contents will be modified once and used many times (means we create it once but it will be drawn every frame)
* `DYNAMIC` - contents will be modified once and used many times (modify every frame and draw every frame?)
and `<access>`: `DRAW`, `READ`, `COPY`

At current point for OpenGL data is just something like `void*` - it doesn't have context how to separate bytes, how intepreate it etc. And vertex data not always 2 floats - vertex != coordinates and can contain different data (like norms, color etc.). 

First we need [glVertexAttribPointer](https://docs.gl/gl4/glVertexAttribPointer):
* `GLuint index` - index of the generic vertex attribute to be modified. It's used by shader and provide index attribute (for triangle we have 3 vertices and each vertex have 2 attributes). 
* `GLint size` - number of attributes must be 1, 2, 3, 4. It's not size in byts, it's count per one vertex (for triangle example = 2)
* `GLenum type` - type of data (GL_FLOAT, GL_BYTE etc)
* `GLboolean normalized` - specifies should data be normalized (GL_TRUE) or converted directly as fixed-point values (GL_FALSE) when they are accessed
* `GLsizei stride` - amount of bytes between each vertex. In general - size in bytes of each vertex (2 * sizeof(float) for triangle)
* `const GLvoid* pointer` - simply offset in bytes where attribute in buffer beggins (GLintptr vertex_texcoord_offset = 3 * sizeof(float); pointer = (GLvoid*)vertex_texcoord_offset). Normally layout stored in `struct` and used macro `offsetof`). 

And now we need to enable that attribe with [glEnableVertexAttribArray](https://docs.gl/gl4/glEnableVertexAttribArray), providing index from glVertexAttrivPoiner
```c++
glVertexAttribPointer(0, 2, GL_FLOAT, GL_FALSE, sizeof(float) * 2, nullptr);
glEnableVertexAttribArray(0);
```

Next stage - deal with shaders: [[How Shaders Work]]