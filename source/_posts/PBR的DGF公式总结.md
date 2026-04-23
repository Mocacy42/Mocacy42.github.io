---
title: PBR的DGF公式总结
date: 2026-03-24
tag: [CG, PBR]
categories: [Tech, CG]
toc: true
comments: true
---

# PBR的DGF公式总结

## 1 法线分布函数NDF

### 1.1 各向同性

若一个各向同性NDF具备形状不变性`shape-invariant`则可以表示为如下形式

$$
D(m)= \frac {1}{ \alpha^2 (n \cdot m)^4}g({\frac { \sqrt{ 1-(n \cdot m)^2}} {\alpha(n \cdot m)^2}})
$$

具备形状不变性的NDF易于推导出各向异性NDF的等价公式。

#### 1.1.1 Blinn-Phong

$$
D_p(h) = \frac {1}{\pi \alpha^2} (n \cdot h)^{\frac 2 {\alpha^2} - 2}
$$

#### 1.1.2 Beckmann

$$
D_b(h) = \frac 1 {\pi \alpha^2 (n \cdot h)^4} exp^{\frac {(n \cdot h) ^2 - 1}{\alpha^2 (n \cdot h)^2}}
$$

#### 1.1.3 GGX

$$
D_{GGX}(h) = \frac {\alpha^2} {\pi ((n \cdot h)^2(\alpha^2 -1) + 1)^2}
$$

#### 1.1.4 GTR

$$
D_{GTR}(h) = \frac {c} {((n \cdot h)^2(\alpha^2 - 1) + 1)^\gamma}
$$

当$\gamma=1$时，$c = \alpha^2 -1$。

当$\gamma = 2$时，$c=\alpha^2$，等同于GGX分布。

在实际实现中，一般只会使用这两种。

### 1.2 各向异性

各向异性的NDF分布可以有具有*形状不变性* 的各向同性NDF方程推导得到，上述各向同性NDF中只有Beckmann和GGX满足*形状不变性*的条件。

其一般形式为

$$
D(m)= \frac {1}{ \alpha_x \alpha_y (n \cdot m)^4}g({\frac { \sqrt{ \frac {(t \cdot m)^2} {\alpha_x^2} + \frac {(b \cdot m)^2} {\alpha_y^2}}} {(n \cdot m)^2}})
$$

#### 1.2.1 Beckmann

$$
D_{Baniso}(m)= \frac {1}{\pi \alpha_x \alpha_y (n \cdot m)^4}exp^{- \frac { \frac {(t \cdot m)^2} {\alpha_x^2} + \frac {(b \cdot m)^2} {\alpha_y^2}} {(n \cdot m)^2}}
$$

#### 1.2.2 GGX

$$
D_{GGXaniso}(m)=\frac {1}{\pi\alpha_x\alpha_y} \frac {1}{(\frac {(t \cdot m)^2} {\alpha_x^2} + \frac {(b \cdot m)^2} {\alpha_y^2} +(n\cdot m)^2)^2}
$$

## 2 几何函数G

### 2.1 Smith遮蔽函数

#### 2.1.1 G1函数形式

$$
G_1(m,v) = \frac{\chi^+(m \cdot v)}{1 + \Lambda(v)}
$$

其中

$$
\chi^+(x) =\begin{cases} 1 & x > 0 \\ 0 &x<=0 \end {cases}
$$

#### 2.1.2 G2的四种形式

##### 2.1.2.1 分离的遮蔽阴影函数`Separable Masking and Shadowing Function`

$$
G_2(v,l,m) = G_1(v,m)G_1(l,m) = \frac {\chi^+(v \cdot m)} {1 + \Lambda(v)} \frac {\chi^+(l \cdot m)} {1 + \Lambda(l)}
$$

##### 2.1.2.2 高度相关的遮蔽阴影函数`Height-Correlated Masking and Shadowing Function`

$$
G_2(v,l,m) =  \frac {\chi^+(v \cdot m) \chi^+(l \cdot m)} {1 + \Lambda(v) + \Lambda(l)}
$$

##### 2.1.2.3 方向相关的遮蔽阴影函数`Direction-Correlated Masking and Shadowing Function`

$$
G_2(v,l,m) =  \lambda(\phi) G_1(v,m)G_1(l,m) + (1-\lambda(\phi)) \min(G_1(v,m),G_2(l,m))
$$

其中$\lambda(\phi)$为经验因子。

##### 2.1.2.4 高度方向相关的遮蔽阴影函数`Height-Direction-Correlated Masking and Shadowing Function`

$$
G_2(v,l,m) =  \frac {\chi^+(v \cdot m) \chi^+(l \cdot m)} {1 + \max (\Lambda(v) , \Lambda(l)) + \lambda(v,l) \min(\Lambda(v),\Lambda{l})}
$$

其中$\large\lambda = \frac{4.41\phi}{4.41\phi + 1}$表示经验因子。

### 2.2 常见NDF对应的$\Lambda$解析形式

#### 2.2.1 Beckmann

$$
\Lambda(v) = \frac {erf(\alpha) - 1} {2} + \frac {1}{2\alpha \sqrt\pi}exp^{-\alpha^2} \approx \begin {cases}\frac {1-1.259\alpha + 0.396\alpha^2}{3.535\alpha + 2.181\alpha^2} & \alpha < 1.6 \\ \ \ \ \ \ \ \ \ \ \ \ 0 & otherwise  \end{cases}
$$

#### 2.2.2 GGX

$$
\Lambda(v) = \frac {-1 + \sqrt{1 + \frac 1 {a^2}}}{2}
$$

其中，$\large a = \frac{1}{\alpha \tan\theta_o}$

### 2.3 业界对Smith GGX的实现

#### 2.3.1 Disney [SIGGRAPH 2012]

$$
G(l,v,h) = G_1(l)G_1(v) \\ \
\\G_1(v) = \frac {2(n \cdot v)} {(n \cdot v) + \sqrt{\alpha^2 + (1 - \alpha^2) (n \cdot v)^2}} \\ \
\\\alpha = (0.5 + roughness/2)^2
$$

#### 2.3.2 UE4 [SIGGRAPH 2013]

$$
G(l,v,h) = G_1(l)G_1(v) \\ \ \\
G_1(v) = \frac {(n \cdot v)} {(n \cdot v)(1 - k) + k} \\ \ \\
k = \frac {\alpha} 2 \\ \ \\
\alpha = (\frac {roughness + 1} {2})^2
$$

#### 2.3.3 Frostbite的GGX-Smith Joint 近似 [Lagarde 2014]

$$
\frac {G_2(l,v)} {4|n \cdot l||n \cdot v|} \Rightarrow \frac {0.5} {\mu_o \sqrt{\alpha^2 + \mu_i(\mu_i - \alpha^2\mu_i)} + \mu_i \sqrt{\alpha^2 + \mu_o(\mu_o - \alpha^2\mu_o)}} \\ \ \\\mu_i = (n \cdot l)^+ \\ \ \\ \mu_o = (n \cdot v)^+
$$

#### 2.3.4 UE4的GGX-Smith Correlated Joint 近似 [2014]

$$
G_2(l,v,h) = \frac {0.5} {\Lambda(v) + \Lambda(l)} \\ \ \\
\Lambda(v) = (n \cdot l)((n \cdot v)(1 - \alpha) + \alpha) \\ \ \\
\Lambda(l) = (n \cdot v)((n \cdot l)(1 - \alpha) + \alpha) \\ \ \\
\alpha = roughness^2
$$

#### 2.3.5 Unity HDRP的 GGX-Smith Correlated Joint 近似

$$
G_2(l,v,h) = \frac {0.5} {\Lambda(v) + \Lambda(l)} \\ \ \\
\Lambda(v) = (n \cdot l)((n \cdot v)(1 - \alpha) + \alpha) \\ \ \\
\Lambda(l) = (n \cdot v)\sqrt{ (-(n \cdot l)\alpha^2 +(n \cdot l))(n \cdot l) + \alpha^2} \\ \ \\
\alpha = roughness
$$

#### 2.3.6 Google Filament渲染器的 GGX-Smith Joint 近似

$$
G_2(l,v,h) = \frac {0.5} {\Lambda(v) + \Lambda(l)} \\ \ \\
\Lambda(v) = (n \cdot l)\sqrt{ (-(n \cdot v)\alpha^2 +(n \cdot v))(n \cdot v) + \alpha^2} \\ \ \\
\Lambda(l) = (n \cdot v)\sqrt{ (-(n \cdot l)\alpha^2 +(n \cdot l))(n \cdot l) + \alpha^2} \\ \ \\
\alpha = roughness
$$

#### 2.3.7 Respawn Entertainment的 GGX-Smith Joint 近似

$$
\frac {G_2(l,v)} {4|n \cdot l||n \cdot v|} \approx \frac {0.5} {lerp(2|n \cdot l||n \cdot v|,|n \cdot l|+|n \cdot v|,\alpha )}
$$

## 参考文献

[PBR中BRDF常用的各类法线分布函数、几何函数总结（unity）-CSDN博客](https://blog.csdn.net/yx314636922/article/details/129123740)

[【基于物理的渲染（PBR）白皮书】（四）法线分布函数相关总结 - 知乎](https://zhuanlan.zhihu.com/p/69380665)

[【基于物理的渲染（PBR）白皮书】（五）几何函数相关总结 - 知乎](https://zhuanlan.zhihu.com/p/81708753)
