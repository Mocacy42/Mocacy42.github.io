---
title: PBR
date: 2026-03-24
tag: [CG, PBR]
categories: [Tech, CG]
toc: true
comments: true
layout: post
---

# Microfacet Cook-Torrance BRDF

## 1. 渲染方程

$$
L_o = L_e +\int_\Omega f_r \cdot L_i \cdot (n \cdot \omega_i) \cdot d\omega_i
$$

* $L_o$ 表示出射光。
* $L_e$ 表示自发光。
* $f_r$ 表示BxDF，即从入射光方向到出射光方向的反射比例（反射率）。
* $L_i$ 表示入射光。
* $(n \cdot \omega_i)$ 表示入射角度带来了光线衰减。
* $\int_\Omega ...  d\omega_i$ 表示入射方向半球积分。

常规的渲染方程由4个部分组成：直接漫反射 直接镜面反射 间接环境光漫反射 间接环境光镜面反射，最常见的Cook-TorranceBRDF方程如下：

$$
L_o = \int_\Omega (k_d \frac {c_{diff} }{\pi} \space+ \space k_s \frac {DFG} {4 (n , v)(n , \omega_i)})  \cdot L_i \cdot (n \cdot \omega_i) \cdot d\omega_i\\
\space \\
k_d = (1 - F)(1 - Metallic)\\
$$

$ks$ 实际已经由 $F$ 表示，因此实际shader中不需要 $ks$

## 2. 迪士尼原则BxDF

### 2.1 核心理念（艺术导向，不一定要完全物理正确）

1. 应使用直观的参数，而不是物理类的晦涩参数。
2. 参数应尽可能少。
3. 参数在其合理范围内应该为0到1。
4. 允许参数在有意义时超出正常的合理范围。
5. 所有参数组合应尽可能健壮和合理。

### 2.2 迪士尼原则 BRDF 十大参数

* baseColor（基础色）：表面颜色。
* subsurface（次表面）：使用次表面控制漫反射形状。
* metallic（金属度）：（0 = 电介质 ， 1 = 金属）控制两种模型线性混合。
* specular（镜面反射强度）：入射镜面反射量，用于取代折射率。
* specularTint（镜面反射颜色）：对baseColor的入射镜面反射进行颜色控制。
* roughness（粗糙度）：表面粗糙度，控制漫反射和镜面反射。
* anilsotropic（各向异性强度）：控制镜面反射高光的纵横比例。（0 = 各向同性 ， 1 = 各向异性）。
* sheen（光泽度）：额外掠射分量，用于布料。
* sheenTint（光泽颜色）：对sheen的颜色控制
* clearcoat（清漆强度）：有特殊用途的第二个镜面波瓣（specular lobe）。
* clearcoatGloss（清漆光泽度）：控制透明涂层光泽度（0 = 缎面satin ，1 = 光泽gloss）。

### 2.3 迪士尼原则BSDF

## 3. 漫反射BRDF模型（Diffuse BRDF）

Diffuse BRDF分为传统型和基于物理型两种，其中传统型主要为Lambert模型。

基于物理型有许多，以下列举几种：

* Oren Nayar [1994]
* Simplified Oren-Nayar [2012]
* Disney Diffuse [2012]
* Renormalized Disney Diffuse [2012]
* Gotanda Diffuse [2014]
* PBR diffuse for GGX + Smith [2017] (Unity采用此种实现Lit Shader)
* MultiScattering Diffuse BRDF [2018]

在UE中，漫反射BRDF使用如下公式：

$$
f(l,v) = \frac {c_{diff}} {\pi}
$$

Cdiff 表示材质的漫反射率（diffuse albedo）

## 4. 镜面反射BRDF模型（Specular BRDF）

### 4.1 Microfacet Cook-Torrance BRDF（基于微平面理论的BRDF）

$$
f(l,v)=\frac{D(h)F(v,h)G(l,v,h)}{4(n\cdot l)(n\cdot v)}
$$

* $D(h)$ ：法线分布函数 （Normal Distribution Function），描述微面元法线分布的概率，即正确朝向的法线的浓度。即具有正确朝向，能够将来自l的光反射到v的表面点的相对于表面面积的浓度。
* $F(v,h)$ ：菲涅尔方程（Fresnel Equation），描述不同的表面角下表面所反射的光线所占的比率。
* $G(l,v,h)$ ：几何函数（Geometry Function），描述微平面自成阴影的属性，即m = h的未被遮蔽的表面点的百分比。
* $4(n \cdot l) (n \cdot v)$ ：校正因子（correctionfactor），作为微观几何的局部空间和整个宏观表面的局部空间之间变换的微平面量的校正。

注意点：

* 对于分母的点积，不仅要避免负值，还要避免0值，一般通过clamp或绝对值操作后加上一个小正值0.0001来完成。
* Microfacet Cook-Torrance BRDF是实践中使用最广泛的模型，实际上也是人们可以想到的最简单的微平面模型。它仅对几何光学系统中的单层微表面上的单个散射进行建模，没有考虑多次散射，分层材质，以及衍射。Microfacet模型，实际上还有很长的路要走。（来自毛星云文章[(48 封私信 / 80 条消息) 【基于物理的渲染（PBR）白皮书】（一） 开篇：PBR核心知识体系总结与概览 - 知乎](https://zhuanlan.zhihu.com/p/53086060)）

Specular BRDF 的种类相较于Diffuse而言十分复杂，根据D（NDF）、F、G三个阶段的不同处理可以得到许多BRDF模型。

#### 4.1.1 Specualr D

表示法线分布函数，描述微平面发现分布概率（正确朝向的法线浓度），根据光学特性分为以下几种模型（*斜体* 表示业界主流）：

##### 各向同性（Isotropic）

* Beckmann [1963]
* Blinn-Phong [1977]
* *GGX [2007] / Trowbridge-Reitz [1975]*
* Generalized-Trowbridge-Reitz(GTR) [2012]

##### 各向异性(Anisotropic)

* Anisotropic Beckmann [2012]
* Anisotropic GGX [2015]

业界主流使用GGX（Trowbridge-Reitz），因为往往具有更加柔和的高光长尾。

$$
D_{GGX} (n,h) =\frac {\alpha^2}{\pi((n \cdot h)^2(\alpha^2 - 1)+1)^2}
$$

其中 $ \alpha = roughness^2$ 并且 $roughness = 1 - smoothness$

![SpecularBRDF](/img/PBR/SpecularBRDF.png)

#### 4.1.2 Specular F

表示菲涅尔方程，描述不同表面角下所反射的光线占比，有三种模型：

* Cook-Torrance [1982]
* Schlick [1994]
* Gotanta [2014]

业界一般使用Schlick的Fresnel近似：

$$
F_{Schlick}(v,h) = F_0 + (1 - F_0)(1 - (v,h))^5
$$

#### 4.2.3 Specular G

表示几何函数，描述微平面自成阴影与自我遮挡的属性，有以下几种模型：

* Smith [1967]
* Cook-Torrance [1982]
* Neumann [1999]
* Kelemen [2001]
* Implicit [2013]

以及Smith联合遮蔽阴影函数（Smith joint maksing-shadowing function）包括以下四种 :

1.高度相关的掩蔽阴影型 Height-Correlated Masking and Shadowing

2.方向相关的掩蔽阴影型 Direction-Correlated Masking and Shadowing

3.高度-方向相关的掩蔽阴影型 Height-Direction-Correlated Masking and Shadowing

4.分离遮蔽阴影型 Separable Masing and Shadowing 包括如下几种：

* Smith-GGX
* Smith-Beckmann
* Smith-Schlick
* Schlick-Beckmann
* *Schlick-GGX* （UE4）

业界常用最简单的分离遮蔽阴影型，它将G分为两个不放呢：光线方向和视线方向，并对连着使用相同的分布函数描述。根据这种思想，结合法线分布函数NDF与Smith几何阴影函数得到。

UE4使用Schlick-GGX，基于Schlick近似，将k映射为$k = \frac \alpha 2$ 从而匹配GGX Smith方程，也有$ k = \frac {(\alpha + 1)^2}{8}$ 的映射：

$$
k = \frac {\alpha} {2} \\
\space\\
\alpha = (\frac {roughness^2 +1}{2})^2 \\
\space\\
 G_1(v) = \frac {n \cdot v}{(n \cdot v)(1 - k) + k}\\
\space\\
G(l,v,n) = G_1(l)G_1(v)
$$

## 5. 基于物理的环境光照（间接光照）

### 5.1 漫反射间接光

根据辉度环境映射（Irradiance Environment Mapping）分为以下几种方法：

* 立方体映射 Cube Mapping
* 球谐函数 Spherical Harmonics
* 蒙特卡洛积分 Monte Carlo Integration
* 球面高斯 Spherical Gaussians
* H-基底 H-basis

### 5.2 镜面反射间接光

使用基于图像的镜面反射光照 IBL（Specular Image-Based Lighting）：

* 预过滤环境映射 Prefiltering Environment Mapping
* 非对称波瓣 Asymmetric Lobe
* 各向异性波瓣 Anisotropic Lobe
* 分解求和近似 Split Sum Approximation 分为两项：
  1. 预过滤环境贴图 Prefiltered Environment Map
  2. 环境BRDF： 包括 2D LUT、解析拟合、3D LUT等处理方法

基于物理的镜面反射（Specular）环境光照，业界中一般会采用基于图像的光照（IBL）的方案。要将基于物理的BRDF模型与基于图像的光照（IBL）一起使用，需要求解光亮度积分（Radiance Integral），而求解光亮度积分通常会使用重要性采样（Importance Sample）。

重要性采样（Importance Sample）即通过现有的一些已知条件（分布函数），想办法集中于被积函数分布可能性较高的区域(重要的区域)进行采样，进而可高效地计算准确的估算结果的的一种策略。

### 5.3 分解求和近似 Split Sum Approximation

根据蒙特卡洛积分公式，渲染方程可以近似为：

$$
\int_{\Omega}L_i(l)f(l,v)\cos\theta_l\cdot dl \approx \frac 1 N \sum^N_{k = 1} \frac {L_i(l_k)f(l_k,v)\cos\theta_{l_k}}{p(l_k,v)}
$$

在实时渲染中，直接求解对性能消耗过大。因此在业界中，基于分解求和近似思想，上述项可以被分为另外两项：表示光亮度均值的$\frac 1 N \sum^N_{k=1}L_i(l_k)$ 和环境BRDF$ \frac 1 N \sum^N_{k=1} \frac {f(l_k,v)\cos\theta_{l_k}}{p(l_k,v）}$ .

$$
\int_{\Omega}L_i(l)f(l,v)\cos\theta_l\cdot dl \approx \left( \frac 1 N \sum^N_{k=1}L_i(l_k) \right) \left(\frac 1 N \sum^N_{k=1} \frac {f(l_k,v)\cos\theta_{l_k}}{p(l_k,v）}\right)
$$

完成拆分后，分别对两项进行离线预计算，去匹配离线渲染参考值的渲染结果。

而在实时渲染中，分别计算分解求和近似（Split Sum Approximation）方案中几乎已经预计算好的两项，再进行组合，作为实时的IBL物理环境光照部分的渲染结果。下面分别对两项进行简单概括。

### 5.4 预过滤环境贴图$ \frac 1 N \sum^N_{k=1}L_i(l_k)$

$ \frac 1 N \sum^N_{k=1}L_i(l_k)$ 表示对光亮度度求均值，可以使用多级模糊的mipmap存储模糊环境高光

$$
\frac 1 N \sum^N_{k=1}L_i(l_k) \approx Cubemap.Sample(r , miplevel)
$$

### 5.5 环境BRDF$ \frac 1 N \sum^N_{k=1} \frac {f(l_k,v)\cos\theta_{l_k}}{p(l_k,v）}$

$ \frac 1 N \sum^N_{k=1} \frac {f(l_k,v)\cos\theta_{l_k}}{p(l_k,v）}$ 表示镜面反射项的半球反射率，取决与仰角 $ \theta$ 粗糙度 $\alpha$ 和菲涅尔项 $F$ 。通常使用Schlick近似$F$ ，其仅在单个值F0上参数化，从而使Rspec成为三个参数(仰角 $ \theta$（NdotV）、粗糙度 $\alpha$、F0)的函数。

在具体处理中，主要有两种方案，UE4的2D LUT 以及 COD：OP2的解析拟合。

#### 5.5.1 2D LUT

UE4在[Real Shading in Unreal Engine 4, 2013]中提出，第二个求和项 ，使用Schlick近似后， F0可以从积分中分出来：

$$
\begin{align}
\int_\Omega L_i (l)f(l,v)\cos\theta_l \cdot dl =  \space
& F_0 \int_\Omega \frac {f(l,v)} {F(v,h)} (1 - (1 -v \cdot h)^5)\cos\theta_l dl  \nonumber \\
\space\nonumber \\ &+ \int_\Omega \frac {f(l,v)} {F(v,h)} (1 -v \cdot h)^5\cos\theta_l dl \nonumber
\end{align}
$$

上式留下了两个输入（Roughness(v , h) 和 cos θv (l , v)）和两个输出（缩放和向F0的偏差（a scale and bias to F0）），即把上述方程看成是F0 \* Scale + Offset的形式。 我们预先计算此函数的结果并将其存储在2D查找纹理（LUT，look-up texture）中。

![2D_LUT](/img/PBR/2D_LUT.png)

这张红绿色的贴图，输入roughness、cosθ，输出环境BRDF镜面反射的强度。是关于roughness、cosθ与环境BRDF镜面反射强度的固有映射关系。可以离线预计算。

$$
\frac 1 N \sum^N_{k=1} \frac {f(l_k,v)\cos\theta_{l_k}}{p(l_k,v）} = LUT.r \space * F_0 + LUT.g
$$

即UE4是通过把Fresnel公式的F0提出来，组成F0 \* Scale +Offset的方式，再将Scale和Offset的索引存到一张2D LUT上。靠roughness和 NdotV进行查找。

#### 5.5.2 解析拟合

COD：Black Ops 2的做法，是通过数学工具Mathematica（[http://www.**wolfram.com/mathematica**/](https://link.zhihu.com/?target=http%3A//www.wolfram.com/mathematica/)） 中的数值积分拟合出曲线，即将UE4离线计算的这张2D LUT用如下函数进行了拟合：

```csharp
float3 EnvironmentBRDF (float g , float NdotV , float rf0)
{
    float4 t = float4( 1/0.96, 0.475, (0.0275 - 0.25 \* 0.04)/0.96, 0.25 );
    t *= float4( g, g, g, g );
    t += float4( 0, 0, (0.015 - 0.75 * 0.04)/0.96, 0.75 );
    float a0 = t.x * min( t.y, exp2( -9.28 * NoV ) ) + t.z; float a1 = t.w;
    return saturate( a0 + rf0 * ( a1 - a0 ) );
}
```

需要注意的是，上面的方程是基于Blinn-Phong分布的结果，[https://**knarkowicz.wordpress.com**/2014/12/27/analytical-dfg-term-for-ibl/](https://link.zhihu.com/?target=https%3A//knarkowicz.wordpress.com/2014/12/27/analytical-dfg-term-for-ibl/)一文中提出了基于GGX分布的EnvironmentBRDF解析版本：

```Csharp
float3 EnvDFGLazarov( float3 specularColor, float gloss, float ndotv )
{
    float4 p0 = float4( 0.5745, 1.548, -0.02397, 1.301 );
    float4 p1 = float4( 0.5753, -0.2511, -0.02066, 0.4755 );
    float4 t = gloss * p0 + p1;
    float bias = saturate( t.x * min( t.y, exp2( -7.672 * ndotv ) ) + t.z );
    float delta = saturate( t.w );
    float scale = delta - bias;
    bias *= saturate( 50.0 * specularColor.y );
    return specularColor * scale + bias;
}
```

上式中的specularColor即F0。

EnvironmentBRDF函数的输入参数分别为光泽度gloss，NdotV，F0。和UE4的做法有异曲同工之妙，但COD：Black Ops 2的做法不需要额外的贴图采样，这在进行移动端优化时，是不错的选择。

## 参考

[【基于物理的渲染（PBR）白皮书】（一） 开篇：PBR核心知识体系总结与概览 - 知乎](https://zhuanlan.zhihu.com/p/53086060)

[【Unity Graphics】PBR（Physically Based Rendering）手撕PBR（2） - 知乎](https://zhuanlan.zhihu.com/p/4209836850)

[[译]Real Shading in Unreal Engine 4（UE4中的真实渲染)(2) - 知乎](https://zhuanlan.zhihu.com/p/125459331)
