---
title: UnityShader
date: 2026-03-24
tag: [Unity, Shader, CG]
categories: [Tech, Unity]
toc: true
comments: true
---

Unity Shader

# 光照模型

## 漫反射

$$
C_{diffuse} = C_{light}  \cdot C_{color} \cdot saturate(\vec n \cdot \vec l_{light} )
$$

saturate（）函数可以将值截取至[0 ,1]。

## 高光

$$
\vec {half} = normalize(\vec {viewDir} + \vec l_{light})  
\\C_{specular} = C_{light} \cdot C_{color} \cdot saturate(\vec n \cdot \vec{half})
$$



## 半兰伯特模型

对漫反射颜色进行调整，不是使用saturate()，而是通过
$$
C_{diffuse} = C_{light}  \cdot C_{color} \cdot (0.5 \cdot (\vec n \cdot \vec l_{light} ) + 0.5)
$$


使得暗部变亮（0.5作为比例系数是可以调整的）