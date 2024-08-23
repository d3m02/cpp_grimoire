Shader - is simply a program that runs on GPU. We provide to OpenGL string with our shader program, ask OpenGL to compile and link that program and then run program. In shader we specify for example where vertices are, what color should be used etc. 

Two most common shaders type are vertex shaders and fragment shaders (also known as pixel shaders). Here requires small understanding of  rendering pipeline. Previously we create buffer, bind buffer, set layout - that's initial steps of render pipeline. Next is call draw call and during draw stage called firstly vertex shader and after fragment shader. 

Vertex shader called for each vertex we try to render. And in general, purpose of vertex shader - specify where vertexes should be on screen. Vertex shader will take all vertex attributes which we specified (so shader also can be used to pass that data to next stages). 

Next stage in pipeline is fragment(pixel) shader. Fragment shader runs for each pixel that need to be rasterized (pixel that should be drawn on screen). For triangle example - fragment shader should provide implementation what color pixel should be. 

### Specify shaders and program 
1. Create program [glCreateProgram](https://docs.gl/gl4/glCreateProgram)
Function returns unsinged int with ID?
```c++
GLuint program = glCreateProgram();
```

2. Create shader [glCreateShader](https://docs.gl/gl4/glCreateShader)(GLenum shaderType), also return id, shaderType from predefined options.
```c++
GLuint vs = glCreateShader(GL_VERTEX_SHADER);
GLuint fs = glCreateShader(GL_FRAGMENT_SHADER);
```

3. Specify source of shader [glShaderSource](https://docs.gl/gl4/glShaderSource)(GLuint shader, GLsizei count, const GLchar \*\*string, const GLint \*length)
-shader - id of shader (use value from flCreateShader)
-count - how many source codes we want to specify 
-string - pointer to pointer with string that contain source code (note, pointer to const char*)
-length - if not whole string requires to be used as source code? if null - expect that each string null terminated. 
```c++
glShaderSource(vs, 1, &src, nullptr);
glShaderSource(fs, 1, &src, nullptr);
```

4. Compile, [glCompileShader](https://docs.gl/gl4/glCompileShader)(GLuint shader); Note that function doesn't return any state etc, so error handling - is separate thing.
```c++ 
glCompileShader(vs);
glCompileShader(fs);
```

5. Attach shader to program [glAttachShader](https://docs.gl/gl4/glAttachShader)(GLuint program, GLuint shader)
```c++
glAttachShader(program, vs);
```

6. Link program [glLinkProgram](https://docs.gl/gl4/glLinkProgram)(GLuint program)

7. Validate program [glValidateProgram](https://docs.gl/gl4/glValidateProgram)(GLuint program)
```c++
glLinkProgram(program);
glValidateProgram(program);
```

8. Now we can delete shader [glDeleteShader](https://docs.gl/gl4/glDeleteShader) since we got program, lined shaders and future work will be with program.
```c++
glDeleteShader(vs);
glDeleteShader(fs);
```

9. And last - time to use program, [glUseProgram](https://docs.gl/gl4/glUseProgram)


### Error handling: 
1. Check status with [glGetShaderiv](https://docs.gl/gl4/glGetShader)(GLuint shader, GLenum pname, GLint \*params) 
-shader - shader ID, 
-pname - object parameter (GL_COMPILE_STATUS, GL_DELETE_STATUS etc)
-params - actual result (output)

`iv` in name - specify that we need integer vector
```c++
int result {GL_FALSE};
glGetShaderiv(vs, GL_COMPILE_STATUS, &result);
```

2. If result `false` - next is extract error message. First we need length of error message and allocate memory. 
```c++
int length;
glGetShaderiv(id, GL_INFO_LOG_LENGTH, &length);
// allocate on heap 
char* message = new char[length];
// allocate on stack
char* message = static_cast<char*>(alloca(length * sizeof(char)));
```

3. Get message: [glGetShaderInfoLog](https://docs.gl/gl4/glGetShaderInfoLog)(GLuint shader, GLsizei maxLength, GLsizei \*length, GLchar \*infoLog);
-shader - id,
-maxLength - in general length of message 
-length - same, but in form of pointer
-infoLog - container where to write message. 
```c++
glGetShaderInfoLog(id, length, &length, message);
std::cout << "Failed to compile vertix shader: " << message << std::endl;
```

4. And since we got error - maybe delete shader also suitable here 


Since for vertex and fragment shader everything is same - when can create function for all this steps
```c++
GLuint CompileShader(unsigned int type, const std::string_view source)
{
    auto id = glCreateShader(type);
    const char* src = source.data();

    glShaderSource(id, 1, &src, nullptr);
    glCompileShader(id);

    int result {GL_FALSE};
    glGetShaderiv(id, GL_COMPILE_STATUS, &result);
    if (result == GL_FALSE)
    {
        int length;
        glGetShaderiv(id, GL_INFO_LOG_LENGTH, &length);
        auto* message = static_cast<char*>(alloca(length * sizeof(char)));

        glGetShaderInfoLog(id, length, &length, message);
        std::cout << "Failed to compile " << (type == GL_VERTEX_SHADER ? "vertex" : "fragment")
                  << " shader: " << message << std::endl;

        glDeleteShader(id);
        return 0;
    }
    return id;
}

GLuint CreateShader(const std::string& vertexShader, const std::string& fragmentShader)
{
    unsigned int program = glCreateProgram();
    
    unsigned int vs = CompileShader(GL_VERTEX_SHADER, vertexShader);
    unsigned int fs = CompileShader(GL_FRAGMENT_SHADER, fragmentShader);

    glAttachShader(program, vs);
    glAttachShader(program, fs);

	glLinkProgram(program);
    glValidateProgram(program);

    glDeleteShader(vs);
    glDeleteShader(fs);

    return program;
}
```

### Shaders source code 
vertex shader
```c++
#version 330 core

layout (location = 0) in vec4 position;
void main()
{
   gl_Position = position;
};
```
fragment shader 
```c++
#version 330 core

layout (location = 0) out vec4 color;
void main()
{
   color = vec4(1.0, 0.0, 1.0, 1.0); ///R G B and alpha
};
```

location = 0 matching glVertexAttribPointer(0, ...);
also in glVertexAttribPointer(, 2, ...) mean that we use vec2, but since 
gl_Position use vec4 - we can use `in vec2 position` and then convert to vec4`gl_Position = vec4(position.xy, 0.0, 1.0)`, but we can just let OpenGL handle it for us, it's know that we specified vec2 but try to get vec4. 


In this point also good to write some parser and store shader in file. We can use to different files, one for vertex shader and one for fragment shader. Or merge into one file, leave some bookmarks where start of shader (let's say `#shader vertex` and `#shader fragment`)
```c++

struct ShaderProgramSource
{
	std::string VertexSource;
    std::string FragmentSource;
};
    
ShaderProgramSource OpenGlGym::ParseShader(const std::string& path)
{
    enum class ShaderType
    {
        NONE = -1,
        VERTEX = 0,
        FRAGMENT = 1
    };
    ShaderType type = ShaderType::NONE;

    std::ifstream stream(path);
    std::string line;
    std::stringstream ss[2];

    while (getline(stream, line))
    {
	    //handle keyword
        if (line.find("#shader") != std::string::npos)
        {
            if (line.find("vertex") != std::string::npos)
                type = ShaderType::VERTEX;
            else if (line.find("fragment") != std::string::npos)
                type = ShaderType::FRAGMENT;
        }
        else if (type != ShaderType::NONE) // bypass everything else considering that it's part of code
        {
            ss[static_cast<int>(type)] << line << '\n';
        }
    }
    return {.VertexSource = ss[0].str(), .FragmentSource = ss[1].str()};
}
```