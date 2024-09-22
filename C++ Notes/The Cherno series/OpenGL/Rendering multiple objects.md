In general for rendering one object we used vertex buffer, indices buffer and shader. And after we call that shaders for each vertex individually. So here we can notice one way how to draw multiple elements - change vertex buffer. That will require second vertex buffer, rebind them between each other.

But if we take a look at shader, we apply MVP matrix to each vertex. So changing MVP matrix is second solution here. Having two vertex buffer is redundant memory in GPU and redundant hustle to rebind them. 

In out example first of all we need to center our object at 0, 0. 
And then simply 
```c++
glm::vec3 translationA(50, 50, 0);
glm::vec3 translationB(400, 50, 0);
while (glfwWindowShouldClose(pWindow) == GLFW_FALSE)
{
    {
	    glm::mat4 model = glm::translate(glm::mat4(1.F), translationA);
        glm::mat4 mvp = proj * view * model;
        shader.SetUniformMat4f("u_MVP", mvp);
        shader.SetUniform4f("u_Color", {r, 0.5F, 0.5F, 1.0F});
            
        shader.Bind();            
        renderer.Draw(va, ib, shader);
    }

    {
	    glm::mat4 model = glm::translate(glm::mat4(1.F), translationB);
        glm::mat4 mvp = proj * view * model;
        shader.SetUniformMat4f("u_MVP", mvp);
        shader.SetUniform4f("u_Color", {r, 0.5F, 0.5F, 1.0F});
            
        shader.Bind();
        renderer.Draw(va, ib, shader);
    }
}    
```
Its pretty silly solution, and for more advanced scene require batch rendering, since making all calls can be quire slow. Also shader binding quite expensive operation, probably require also to add some protection for avoiding unnecessary bindings. 