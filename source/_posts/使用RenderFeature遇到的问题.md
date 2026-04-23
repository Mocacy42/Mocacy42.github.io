---
title: 使用RenderFeature遇到的问题
date: 2026-03-24
tag: [Unity]
categories: [Tech]
toc: true
comments: true
---

### 在后处理的Blit过程中，如果要使用自定义的着色器，顶点着色器中应该如此获取UV及CS坐标：

```csharp
                o.posCS = TransformObjectToHClip（GetFullScreenTriangleVertexPosition(v.vertexID)）;
                o.uv = GetFullScreenTriangleTexCoord(v.vertexID);
```

否则在片元着色器中无法正确采样 _BlitTexture ，导致无效。
