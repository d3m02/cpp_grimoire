Batch rendering quite might be difficult to describe since it's can mean from something simple up to complicated techniques. 

Simple meaning - batch rendering is way how we can batch together geometry, how we can render more geometry with a single draw call. So simplest way to render something - define vertex buffer with vertices, index buffer with indices and render them together with some GlDrawElements. If we want some another geometry to be rendered - we basically repeat that process. Sometime we can reuse same index and vertex buffers and apply some transformation, of cause. But this can not work when we need to render really a lot of objects. Worth mentioning that modern (especially) dedicated GPU can handle thousands of items - here point it that it's not so efficient and it's possible to improve. 

In nutshell, batch rendering - is when we batch together all geometry into a single vertex buffer and index buffer and draw it at once. 

Let's take look at quad rendering. 
1 quad - 
* it's Vertex Array
	* Vertex Buffer    -   4x vertices
	* Index Buffer     -   6x indices 
	and it's actually 2 triangles (3 * 2 indices, where 2 vertices - shared between two triangles). 

2 quads - it's rendering Vertex Array twice with different transform. 
But what if merge them together. In that case 
* Vertex Array 
	* Vertex Buffer    -   2 * 4x vertices
	* Index Buffer     -   2 * 6x indices 
	* Problem is position of each quad - that information can be injected into Vertex Buffer. Color for example also can be put into Vertex Buffer. 
	* If some object moves - Index Buffer become dynamic where we stream data. 

It' can be achieved simply by adding new vertices in Vertex Buffer and indices in Index Buffer:
```c++
 std::array vertices { -50.F, -50.F, 0.0F, 0.0F,   // 0
                        50.F, -50.F, 1.0F, 0.0F,   // 1
                        50.F,  50.F, 1.0F, 1.0F,   // 2
                       -50.F,  50.F, 0.0F, 1.0F,   // 3

                          150.F, 150.F, 0.0F, 0.0F,   // 4
                           50.F, 150.F, 1.0F, 0.0F,   // 5
                           50.F,  50.F, 1.0F, 1.0F,   // 6
                          150.F,  50.F, 0.0F, 1.0F }; // 7
    
    
std::array indices { 0U, 1U, 2U, 2U, 3U, 0U,
                    4U, 5U, 6U, 6U, 7U, 4U };

```

Next stage of modifications - add color. One example how color can be added to each quad - is adding color in vertex. 
Vertex shader gonna read color value from Vertex Buffer and pass it to fragment shader. Also, we need to modify layout: add new slot with color and modify stride. 

```c++
  /*                        x      y     z      R     G      B     A*/
    std::array vertices { -50.F, -50.F, 0.0F, 0.18F, 0.6F, 0.96F, 1.0F,   
                           50.F, -50.F, 0.0F, 0.18F, 0.6F, 0.96F, 1.0F,   
                           50.F,  50.F, 0.0F, 0.18F, 0.6F, 0.96F, 1.0F,   
                          -50.F,  50.F, 0.0F, 0.18F, 0.6F, 0.96F, 1.0F,   

                          150.F, 150.F, 0.0F, 1.F, 0.93F, 0.24F, 1.0F,   
                           50.F, 150.F, 0.0F, 1.F, 0.93F, 0.24F, 1.0F,   
                           50.F,  50.F, 0.0F, 1.F, 0.93F, 0.24F, 1.0F,   
                          150.F,  50.F, 0.0F, 1.F, 0.93F, 0.24F, 1.0F }; 

	/* vertex possition */
	glEnableVertexArrayAttrib(m_quad, 0)
	glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 7 * sizeof(float), 0);
	
	/* vertex color */
	glEnableVertexArrayAttrib(m_quad, 1)
	glVertexAttribPointer(1, 4, GL_FLOAT, GL_FALSE, 7 * sizeof(float), (const void*)12);
	
```

As for shaders parts:
```c++
#shader vertex
#version 330 core

layout (location = 0) in vec3 a_Position;
layout (location = 1) in vec4 a_Color;

uniform mat4 u_MVP;

out vec4 v_Color;

void main()
{
   v_Color = a_Color;
   gl_Position = u_MVP * vec4(a_Position, 1.0);
};

#shader fragment
#version 330 core

layout(location = 0) out vec4 o_Color;

in vec4 v_Color;

void main()
{
	o_Color = v_Color;
}
```


Next, textures. One simples solution - use texture add list of sprites grid. It's commonly used in pixelart-style games. From some perspective - it's also batching, but textures. But it's more like general case. Instead, we can do something similar to what we do with color - store information in Vertex Buffer - we can store texture coordinates and texture slot. We can bind multiple textures in different slots, not only one (max amount of slots depends from GPU). Also we need uniform with sample2D. 
Now vertices contain enough information, but it's now hard to read. So better to wrap everything with struct. 

```c++
struct Vertex
{
    std::array<float, 3> positions;
    std::array<float, 4> color;
    std::array<float, 2> textureCoord;
    float textureSlot;
};

std::array<Vertex, 8> oldvertices {{ 
        { .positions =    {-50.F, -50.F, 0.0F}, 
          .color =        {0.78F, 0.6F, 0.96F, 1.0F}, 
          .textureCoord = {0.0F, 0.0F}, 
          .textureSlot =  0.0F },
        { .positions =    { 50.F, -50.F, 0.0F}, 
          .color =        {0.18F, 0.6F, 0.16F, 1.0F}, 
          .textureCoord = {1.0F, 0.0F}, 
          .textureSlot =  0.0F }, 
        { .positions =    { 50.F,  50.F, 0.0F}, 
          .color =        {0.78F, 0.6F, 0.16F, 1.0F}, 
          .textureCoord = {1.0F, 1.0F},
          .textureSlot =  0.0F }, 
        { .positions =    {-50.F,  50.F, 0.0F}, 
          .color =         {0.18F, 0.6F, 0.96F, 1.0F}, 
          .textureCoord =  {0.0F, 1.0F},
          .textureSlot =   0.0F },

         { .positions =    {150.F, 150.F, 0.0F}, 
           .color =        {1.F, 0.23F, 0.24F, 1.0F}, 
           .textureCoord = {0.0F, 0.0F}, 
           .textureSlot =  1.0F },   
         { .positions =    { 50.F, 150.F, 0.0F}, 
           .color =        {1.F, 0.93F, 0.94F, 1.0F}, 
           .textureCoord = {1.0F, 0.0F}, 
           .textureSlot =  1.0F },   
         { .positions =    { 50.F,  50.F, 0.0F}, 
           .color =        {1.F, 0.23F, 0.84F, 1.0F}, 
           .textureCoord = {1.0F, 1.0F}, 
           .textureSlot =  1.0F },   
         { .positions =    {150.F,  50.F, 0.0F}, 
           .color =        {0.F, 0.93F, 0.24F, 1.0F}, 
           .textureCoord = {0.0F, 1.0F}, 
           .textureSlot =  1.0F } 
         }};

/* vertex possition */
glEnableVertexArrayAttrib(m_quad, 0)
glVertexAttribPointer(0, Vertex.positions., GL_FLOAT, GL_FALSE, sizeof(Vertex), 0);
	
/* vertex color */
glEnableVertexArrayAttrib(m_quad, 1)
glVertexAttribPointer(1, 4, GL_FLOAT, GL_FALSE, sizeof(Vertex), (const void*)offsetof(Vertex, color));
	
/* vertex texture coord */
glEnableVertexArrayAttrib(m_quad, 2)
glVertexAttribPointer(2, 2, GL_FLOAT, GL_FALSE, sizeof(Vertex), (const void*)offsetof(Vertex, textureCoord));
	
/* vertex texture slot */
glEnableVertexArrayAttrib(m_quad, 3)
glVertexAttribPointer(3, 1, GL_FLOAT, GL_FALSE, sizeof(Vertex), (const void*)offsetof(Vertex, textureSlot));

/* load textures*/
m_texture1 = std::make_unique<Texture>("texture1.png");
m_texture2 = std::make_unique<Texture>("texture2.png");

/* sampler2D */
auto loc = glGetUniformLocation(shader->GetRendererID(), "u_Textures");
int samplers[2] = { 0, 1 };
glUniform1iv(loc, 2, samplers);

```

and relative changes in shader 
```c++
#shader vertex
#version 330 core

layout (location = 0) in vec3 a_Position;
layout (location = 1) in vec4 a_Color;
layout (location = 2) in vec2 a_TexCoord;
layout (location = 3) in float a_TexSlot;

uniform mat4 u_MVP;

out vec4 v_Color;
out vec2 v_TexCoord;
out float v_TexSlot;

void main()
{
   v_Color = a_Color;
   v_TexCoord = a_TexCoord;
   v_TexSlot = a_TexSlot;
   gl_Position = u_MVP * vec4(a_Position, 1.0);
};

#shader fragment
#version 330 core

layout(location = 0) out vec4 o_Color;

in vec4 v_Color;
in vec2 v_TexCoord;
in float v_TexSlot;

uniform sampler2D u_Textures[2];


void main()
{
   int slot = int(v_TexSlot);
   vec4 texColor = texture(u_Textures[slot], v_TexCoord);
    o_Color = v_Color * texColor;
}
```

This technic have noticeable limitations. First, to render object without texture - we need to add some offset or negative value and somehow determine - when we need use color, when need use texture, when both. Also, number of texture slots - platform-depended. For this situation when can split batch on multiple batches based on amount of slots. 

And now, let's make it dynamic. In most cases we want no changes objects on some specific frames. And this what dynamic arrays are for. 
To achieve this, firstly we replace Vertex Buffers definition with allocation by simply put in glBufferData nullptr instead pointer to Vertex Buffer and replace GL_STATIC_DRAW with GL_DYNAMIC_DRAW.
And now by simply initializing vertex array in render loop and using 
```c++
glBufferSubData(GL_ARRAY_BUFFER, 0, sizeof(indexArray), indexArray);
```
we can dynamically change objects inside batch. 