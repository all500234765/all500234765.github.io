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
---

# What is a CRTP?
Curiously Recurring Template Pattern. 

# Using with DirectX
CRTP is very useful when using graphics API. Since almost all of them (not really sure about Vulkan and OpenGL)



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

