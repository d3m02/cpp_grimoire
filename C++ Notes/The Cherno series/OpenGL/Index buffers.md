Triangle is graphics is almost basic element for any figures (since to define plane requires 3 points). To draw square (or rectangle) we can you two triangles: 
```
(-0.5, 0.5)__________ (0.5, 0.5)
          |         /|
          |      /   |
          |   /      |
          |/_________|
(-0.5, -0.5)        (0.5, -0,5)
```
As we can see, 2 vertices draw twice. We can simply use instead 
```c++ 
std::array vertices {-0.5F, -0.5F, 0.0F, 0.5F, 0.5F, -0.5F};
glBufferData(GL_ARRAY_BUFFER, 
            static_cast<GLsizeiptr>(sizeof(vertices)),
            vertices.data(),
            GL_STATIC_DRAW);
glDrawArrays(GL_TRIANGLES, 0, 3);
```
just 
```c++
std::array vertices {-0.5F, -0.5F,
                      0.5F, -0.5F,
                      0.5F,  0.5F,

                      0.5F,  0.5F,
                     -0.5F,  0.5F,
                     -0.5F, -0.5F};
glBufferData(GL_ARRAY_BUFFER, 
             static_cast<GLsizeiptr>(sizeof(vertices)),
             vertices.data(),
             GL_STATIC_DRAW);
glDrawArrays(GL_TRIANGLES, 0, 6);
```
But we just duplicate data and waste GPU memory. In this example it's not huge loss, but if each vertex contain more - this become more and more noticeable. So instead duplicating - we can create just index buffer. Note that we can use only unsigned here. 
```c++
std::array vertices {-0.5F, -0.5F,  // 0
                      0.5F, -0.5F,  // 1
                      0.5F,  0.5F,  // 2
                     -0.5F,  0.5F}; // 3
glBufferData(GL_ARRAY_BUFFER, 
             static_cast<GLsizeiptr>(sizeof(vertices)),
             vertices.data(),
             GL_STATIC_DRAW);

unsigned int ibo;  
glGenBuffers(1, &ibo);  
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, ibo);
std::array indices {0u, 1u, 2u,
			        2u, 3u, 0u};
			        
glBufferData(GL_ELEMENT_ARRAY_BUFFER, 
	         static_cast<GLsizeiptr>(sizeof(indices)),
	         indices.data(),
	         GL_STATIC_DRAW);

glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_INT, nullptr); // nullptr since we bind GL_ELEMENT_ARRAY_BUFFER, 
```

