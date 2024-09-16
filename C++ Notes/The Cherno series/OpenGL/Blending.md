Blending takes place when we talk about transparency. When we render something with transparency - we need to specify how this should impact on other object on scene.

> Blending determines how we combine out output color with what is already in out target buffer. Output = the color we output from our fragment shader (know as source). Target buffer = the buffer our fragment shader is drawing to (known as destination).
  
OpenGL provide three ways to control blending 
+ [glEnable(GL_BLEND)](https://docs.gl/gl4/glEnabl)// [glDisable(GL_BLEND)](https://docs.gl/gl4/glEnable) 
+ [glBlendFunc(src, dest)](https://docs.gl/gl4/glBlendFunc) 
	+ src = how the src RGBA factor in computed (default is GL_ONE)
	+ dest = how the dest RGBA factor is computed (default is GL_ZERO)
	We take each channel of pixel and multiply it with some value we specified. GL_ONE mean that we take source and multiple with 1 (which have no affect). GL_ZERO in destination mean to erase data, "throw away data in destination and overwrite with value from source".  
+ [glBlendEquation(mode)](https://docs.gl/gl4/glBlendEquation)
	+ mode specify how we combine src and dest colors. Default is GL_FUNC_ADD 

Let's take example. We use`glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA)`,
source is (1.0, 1.0, 1.0, 0.5), destination (1.0, 0.0, 1.0, 1.0).
With such input we will get 
R = (1.0 * 0.5) + (1.0 * (1 - 0.5)) = 1.0
G = (1.0 * 0.5) + (0.0 * (1 - 0.5)) = 0.5
B = (1.0 * 0.5) + (1.0 * (1 - 0.5)) = 1.0
A = (0.5 * 0.5) + (1.0 * (1 - 0.5)) = 0.75


