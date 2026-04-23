---
title: 技美百人计划-Unity空间坐标系
date: 2026-03-24
tag: [Unity]
categories: [Tech]
toc: true
comments: true
---

# Unity空间坐标系

在Unity渲染流程中，常用的空间坐标系包括：物体空间、世界空间、观察空间、齐次裁剪空间、NDC、屏幕空间。

## 物体空间ObjectSpace

物体空间是记录模型顶点信息的空间（如FBX空间），拥有自己的原点和坐标系。

* PosOS [x y z w] xyz的范围为[-1 , 1] ，w[1]为齐次量。

## 世界空间WorldSpace

世界空间指的是Unity世界空间坐标系，即：Unity的Scene空间。

* PosWS [x y z w] xyz为Scene世界坐标，w[1]为齐次量。

## 观察空间ViewSpace

观察空间是以摄像机为原点的一个坐标系。

* PosVS [x y z w] xyz为观察空间坐标，w[1]为齐次量。

## 齐次裁剪空间Homogeneous Clip Space

齐次裁剪空间是观察空间坐标通过投影矩阵投影得到。

PosCS [x y z w]

* 在透视投影下 xyz范围为[-w , w], w[near far]表示近裁剪平面到远裁剪平面的范围
* 在正交投影下 xyz范围为[-1 , 1], w[1]

## NDC空间

NDC空间为HCP归一化所得空间

PosNDC [x y z w]

* xy范围为  PosCS.xy/PosCS.w
* zw = PosCS.zw

## 屏幕空间ScreenSpace

屏幕空间，名副其实，表示的是我们在Unity窗口中观察的屏幕。

ScreenPos [x y z w]

* x范围是[-1 , 1]
* y范围是[-1 , 1]
* zw = PosCS.zw

```csharp
inline float4 ComputeScreenPos (float4 posCS)
{
    float4 o = posCS * 0.5f;
    o.xy = float2(o.x, o.y*_ProjectionParams.x) + o.w;
    o.zw = pos.zw;
    return o;
}
```

对于ShaderGraph而言，其中有一个节点名为 SceneDepth , 获取的是场景到摄像机的深度，就是通过 ScreenPos.z 获取的，它与深度图中获取的深度不同之处在与深度图中只有Opaque的深度信息，而SceneDepth则没有这个限制可以是任意物体。

在水体渲染中，常常用它们的差值获取水体深度。

* ScreenUV = float2(1, _ProjectionParams.x ) * ScreenPos.xy/ScreenPos.w
* ScreenUV = PosCS.xy /( width , height)  （这里的PosCS是具有SV_POSITION语义语以的，会自动将xy值映射为屏幕坐标（[ 0 , width ] , [ 0 , height ]））
* ScreenUV = float2 （1 , _ProjectionParams.x）（ i.posCS.xy / i.posCS.w ） * 0.5 + 0.5（这里的PosCS是不具有SV_POSITION语义的）

ScreenUV常用于采样屏幕纹理，在一些全屏后处理效果中会发挥作用。
