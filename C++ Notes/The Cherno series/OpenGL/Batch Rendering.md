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

