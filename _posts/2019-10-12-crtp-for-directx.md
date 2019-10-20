---
layout:     post
title:      CRTP for DirectX
date:       2019-10-12 8:00:00
author:     Kirill Evdakimov
summary:    Some HLSL 11 Tips
categories: C++
thumbnail:  heart
tags:
 - C++
 - CRTP
 - DirectX
 - template
---

# What is a CRTP?
Straight copy from a wikipedia:
> Curiously Recurring Template Pattern - is an idiom in C++ in which a class X derives from a class template instantiation using X itself as template argument.

* [Wikipedia][1]

# Using with DirectX
CRTP is very useful when using it with graphics API. Since almost all of them (not sure about Vulkan and OpenGL) have similar things like blend states, depth stencil states, shaders and so on. (I call em - pipeline states)
Some times when we make a class or function that must do something with our pipeline states we need to set old pipeline state back when we done doing that something.
And writing similar functionality to each pipeline state class is time expensive. 

Why don't we just use some kind of programming pattern and make our lifes easier?
Let's just skip intermidiate part where i write all about CRTP and so on, 'kay?

Here is an example of my [pipeline state class][3] that uses CRTP.

{% highlight C++ %}

template<typename T>
class PipelineState: public DirectXChild {
protected:
    static T *gState;
    static T *gStateOld;

public:
    static T* Current() { return gState;                     }
    static void Push()  { gStateOld = gState;                } /* Store current state */
    static void Pop()   { if( gStateOld ) gStateOld->Bind(); } /* Re-Store old state */
};

template<typename T>
T *PipelineState<T>::gState = 0;

template<typename T>
T *PipelineState<T>::gStateOld = 0;

{% endhighlight %}

It implements 3 functions and adds 2 protected variables. 

Let's begin with understanding how this class is used with classes that inherit from it.
Here's an [example][2]:

{% highlight C++ %}

/* Store state */
DepthStencilState::Push();

/* ... */

/* Restore old state */
DepthStencilState::Pop();

/* Store old state */
BlendState::Push();

/* ... */

/* Restore old states */
BlendState::Pop();
DepthStencilState::Pop();

{% endhighlight %}

First of all we see ___::Push()___ and ___::Pop()___. What do they do? Let's look at [BlendState][4] and [DepthStencilState][5] classes:

{% highlight C++ %}
/* DepthStencilState */
class DepthStencilState: public PipelineState<DepthStencilState> {
private:
    ID3D11DepthStencilState *pState = 0;
    UINT StencilRef = 0;

public:
    void Create(D3D11_DEPTH_STENCIL_DESC pDesc, UINT SRef=0);
    void Bind();
    void Release();
};

/* BlendState */
class BlendState: public PipelineState<BlendState> {
private:
    ID3D11BlendState *pState = 0;
    DirectX::XMFLOAT4 Factor = { 1.f, 1.f, 1.f, 1.f };
    UINT SampleMask = 1;

public:
    void Create(D3D11_BLEND_DESC pDesc, DirectX::XMFLOAT4 f, UINT SampleMask=1);
    void Bind();
    void Release();
};

{% endhighlight %}

* We don't see any functions called ___Push___ or ___Pop___.
* Both classes inherite from PipelineState but with a template of it self?..

> When a class inherits from another one and for the template argument it provides it self - it's a Curiously Recurring Template Pattern.

If you still can't undestand what is CRTP and how does it work, re-read this quote a couple of times.
When i first heard of CRTP i didn't quite understod what was that and i had to read about it and watch at code a couple of times.

We use CRTP in three classes as mentioned above(*BlendState, DepthStencilState, PipelineState*).

# So what does CRTP gives us?
No more identical code that we have to copy over and over again for each class and modify it, so types will be the same as current class.
In example above we saw that PipelineState have a 2 variables with type **T*** gState and gStateOld. 
For all inherited class from PipelineState we give each one of them similar functionality to store a state, return current state and re-store the old state.
This is useful, because we don't have to write something like this:
```cpp
// Store current
BlendState* bsOld = BlendState::Current();

// Bind new state
bsNew->Bind();

// ...

// Re-Store
bsOld->Bind();
```

0. We create additional variable called bsOld with a pointer to current state.
1. Then we bind new BlendState.
2. Do something
3. Re-store a state.

The thing is, that we have to create a variable, get current state and when we done doing something, we only then re-store old state.
This looks kinda ugly to me. Let's add functions Push and Pop in our code:
```cpp
BlendState::Push();

// Bind new state
bsNew->Bind();

// ...

BlendState::Pop();
```

Btw, i didn't mentioned before, but we can add a stack to PipelineState class to store more than one state.
And when we modify a parent class all of it's children going to have same functionality, which prevents us from copy and pasting same code over again.

# Conclusion
* Use patterns in your code, it will make your code cleaner and help you unify it.
* Use CRTP for states, ECS or somewhere else where your need it.


# References
* [Wikipedia][1]
* [Flientcpp][6]
* [Stackoverflow][7]

[1]: https://en.wikipedia.org/wiki/Curiously_recurring_template_pattern
[2]: https://github.com/all500234765/Luna-Engine/blob/27dd077ec6f088edf069d826542aebef4f24d517/DirectX11%20Engine%202019/Effects/SSLRPostProcess.h#L230
[3]: https://github.com/all500234765/Luna-Engine/blob/DeferredMSAA/DirectX11%20Engine%202019/Engine/States/PipelineState.h
[4]: https://github.com/all500234765/Luna-Engine/blob/DeferredMSAA/DirectX11%20Engine%202019/Engine/States/BlendState.h
[5]: https://github.com/all500234765/Luna-Engine/blob/DeferredMSAA/DirectX11%20Engine%202019/Engine/States/DepthStencilState.h
[6]: https://www.fluentcpp.com/2017/05/12/curiously-recurring-template-pattern/
[7]: https://stackoverflow.com/questions/4173254/what-is-the-curiously-recurring-template-pattern-crtp

