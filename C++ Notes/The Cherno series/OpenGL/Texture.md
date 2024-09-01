So basically, texture - it's some external data which we load in GPU with description for shader how to draw frame. In this example we will use PNG file as texture which shader should add in rectangle. 

First, we need to somehow load data in CPU. For PNG we can you [stb/stb_image.h](https://github.com/nothings/stb/blob/master/stb_image.h) to save time a little bit. This is single file library, we need only this file and somewhere add (or simply create `stb_image.c` file to generate obj file with implementation)
```C++
#define STB_IMAGE_IMPLEMENTATION
#include "stb_image.h"
```

Next we call [glGenTextures](https://docs.gl/gl4/glGenTextures)(GLsizei amount, GLuint* id) and [glBindTexture](https://docs.gl/gl4/glBindTexture)(GLenum type, GLuint id)
```c++ 
glGenTextures(1, &m_rendererID);
glBindTexture(GL_TEXTURE_2D, m_rendererID);
```

Also we need `stbi_set_flip_vertically_on_load(1);` since OpenGL expect texture pixel starts at bottom left (not top left), bottom left in OpenGL is (0; 0). 

Next we load data in local buffer (here 4 - is amount of channels, rgb+alpha) 
```c++
m_localBuffer = stbi_load(path.c_str(), &m_width, &m_height, &m_bpp, 4);
```

Next is specific some parameters for texture: 
+ Minification filter (when texture should be resampled down / zoom in)
```c++
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
```
+ Magnification filter (resample up / zoom out)
```c++
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
```
+ Horizontal (WRAP_S) Clamp mode (not extend area)
```c++
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);
```
+ and vertical (WRAP_T) clamp
```c++
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);
```
Take note: `glTextureParameteri` - different function which will return error `GL_INVALID_OPERATION`

If not specify this parameters - texture will be black. 

Next is actual load texture [glTexImage2D](https://docs.gl/gl4/glTexImage2D)
```c++
glTexImage2D(GL_TEXTURE_2D, 
			 0,            // Level, but we don't use multilevel here
			 GL_RGBA8,     // internal format, 8 - bit depth
			 m_width, m_height, 
			 0,           // border, we don't have border
			 GL_RGBA,     // pixel format
			 GL_UNSIGNED_BYTE, 
			 m_localBuffer);
```

for stbi we also need to free data
```c++
if (m_localBuffer)
	stbi_image_free(m_localBuffer);
```

We can bind texture to internal slots [glActiveTexture](https://docs.gl/gl4/glActiveTexture)(`GL_TEXTURE0`). Amount of slots depends on platform, for PC it's likely 32 slots. Selecting slot is something like "Select this slot, and if I call bind/something else - this should be related to selected slot"

Now when texture loaded, we need to tell somehow to shader where to find texture. We can done it trough uniform with slot number.
```c++
shader.SetUniform1i("u_TextureSlot", 0);
```

Also we need define some sort of coordinate system to tell our geometry how to rasterize this texture. As we remember pixel shader determine what color each pixel should be. With texture pixel shader need to sample some area from our texture to retrieve what color the pixel should be. In another words, pixel shader need to extract from texture what pixel draw from texture based on screen coordinate. For rectangle we specify for each vertex for area of texture that should be and fragment shader will interpolate between that. 

In this example we add for each vertices additional coordinate, so we will have 4 floats per vertex (which require some adaptation in layout and vertex buffer)
```c++
std::array vertices { -0.5F, -0.5F, 0.0F, 0.0F,   // 0
                       0.5F, -0.5F, 1.0F, 0.0F,   // 1
                       0.5F,  0.5F, 1.0F, 1.0F,   // 2
                      -0.5F,  0.5F, 0.0F, 1.0F};  // 3
```

and apply changes in shader: in vertex shader add new `layout (location = 1) in vec2 texCoord;` in pixel shader specified `uniform sampler2D u_Texture;` and `vec4 texColor = texture(u_Texture, texCoord);`. Now we need somehow transfer data from vertex shader to fragment shader, for this can be used `varying system`.
Writing
```c++
#shader vertex
layout (location = 1) in vec2 texCoord;
out vec2 v_TexCoord;
void main()
{
   v_TexCoord = texCoord;
};

#shader fragment
uniform sampler2D u_Texture;
in vec2 v_TexCoord;
void main()
{
	vec4 texColor = texture(u_Texture, v_TexCoord);
}
```
we will sampling texture according to texture coordinate relative to vertices. Note that if uniform in shader not used - we will get errors when try to set it from CPU.

And final result
```c++
#shader vertex
#version 330 core

layout (location = 0) in vec4 position;
layout (location = 1) in vec2 texCoord;

out vec2 v_TexCoord;

void main()
{
   gl_Position = position;
   v_TexCoord = texCoord;
};

#shader fragment
#version 330 core

layout(location = 0) out vec4 color;

in vec2 v_TexCoord;

uniform vec4 u_Color;
uniform sampler2D u_TextureSlot;

void main()
{
   vec4 texColor = texture(u_TextureSlot, v_TexCoord);
   color = u_Color * texColor;
}
```

	What is missing here - is blending alpha channel. Just for testing of texture we can add 
```c++
glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA);
glEnable(GL_BLEND);
```