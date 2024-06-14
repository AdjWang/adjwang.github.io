---
title: "The GLAD Library"
date: 2023-12-14T23:11:03+08:00
---

## Platform compatible

Rendering is not a easy work, it needs both software and hardware support. Unfortunately, there are more than one software/hardware renderer vendor on earth.

### For hardware

OpenGL tries to unify different device driver interfaces. It is **a standard rather than a library**, so OpenGL never provides implementation for rendering. GPU vendors offer the implementation. Any program needs to use OpenGL has to load implementation dynamically, writing some dynamic linking code like this:

```cpp
typedef void (*GENBUFFERS) (GLsizei, GLuint*);
GENBUFFERS glGenBuffers = (GENBUFFERS)wglGetProcAddress("glGenBuffers");
// ... invoke wglGetProcAddress per API
```

Oh, tremendously boring...

### For software

Unfortunately again, different OS provides different interfaces to do that dynamic linking. For APIs, Windows provides `wglGetProcAddress`, Linux provides `glXGetProcAddress`, and MacOS provides `NSGLGetProcAddress`. Code above copies and modifies every time fitting to different systems.

Even worse, some OpenGL version may cause `wglGetProcAddress` fail on windows. ([OpenGL wiki](https://www.khronos.org/opengl/wiki/Load_OpenGL_Functions))

### Glad there is glad

[glad](https://github.com/Dav1dde/glad), also alternatives like [glew](https://github.com/nigels-com/glew), [glee](https://github.com/kallisti5/glee), are used to simplify the checking and linking routine. Now use `gladLoadGLLoader((GLADloadproc)glfwGetProcAddress)` in a nutshell to make the life easier.

> [glfw](https://github.com/glfw/glfw) is a multi-platform window/event manager library.

## Use glad without glfw on windows

Using [glfw](https://github.com/glfw/glfw) to initialize glad is a popular way. But for curiosity, can we use `wglGetProcAddress` to initialize glad?

The answer is yes but not directly. You cannot directly write code like this:

```cpp
if (!gladLoadGLLoader((GLADloadproc)wglGetProcAddress)) {
    ...
}
```

[Someone failed already](https://community.khronos.org/t/glad-without-glfw/105017). The reason is described [here](https://www.khronos.org/opengl/wiki/Load_OpenGL_Functions) that the dynamic linking interface is not available to `wglGetProcAddress` in `OpenGL32.DLL`. The right way is to mix the `wglGetProcAddress` and `GetProcAddress` like this:

```cpp
void *GetAnyGLFuncAddress(const char *name)
{
    void *p = (void *)wglGetProcAddress(name);
    if (p == 0 ||
       (p == (void*)0x1) || (p == (void*)0x2) || (p == (void*)0x3) ||
       (p == (void*)-1) )
    {
        HMODULE module = LoadLibraryA("opengl32.dll");
        p = (void *)GetProcAddress(module, name);
    }

    return p;
}
if (!gladLoadGLLoader((GLADloadproc)GetAnyGLFuncAddress)) {
    ...
}
```

This code works well with the [Windows OpenGL Context](https://www.khronos.org/opengl/wiki/Creating_an_OpenGL_Context_%28WGL%29). Full code snippet is available [here](https://gist.github.com/AdjWang/808422011aee43312f87b15b69e238c6).


## References

- [Learn OpenGL](https://learnopengl.com/Introduction)
- [Open.GL tutorial - Context creation](https://open.gl/context#Onemorething)
- [OpenGL wiki - Load OpenGL Functions](https://www.khronos.org/opengl/wiki/Load_OpenGL_Functions)
- [OpenGL wiki - Creating an OpenGL Context (WGL)](https://www.khronos.org/opengl/wiki/Creating_an_OpenGL_Context_%28WGL%29)
