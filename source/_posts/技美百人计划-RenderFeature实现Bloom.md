---
title: 技美百人计划-RenderFeature实现Bloom
date: 2026-03-24
tag: [Unity]
categories: [Tech]
toc: true
comments: true
---

# RF实现简易泛光Bloom效果

首先先看看Bloom效果对比

原图：

![](/img/技美百人计划-RenderFeature实现Bloom/1741614850486.png)

Bloom处理图：

![](/img/技美百人计划-RenderFeature实现Bloom/1741614889449.png)

## Bloom原理解析

Bloom可以分为三个步骤：

1. 提取高亮部分Prefilter
2. 模糊处理Downsmaple
3. 混合Upsample

### Prefilter

泛光可以认为是一个高亮物的颜色向周围延伸的结果，因此需要先提取出画面中的亮部。

我使用了自定义的亮度计算公式luminance

```csharp
float luminance(float4 color)
        {
            return 0.2125 * color.r + 0.7154 * color.g + 0.0721 * color.b;
        }
```

（0.2125 , 0.7145 , 0.0721）是人眼对RGB颜色亮度的感知灵敏度，通过将之与RGB相乘可以得到人眼感知的亮度。

在BloomShader中，提取亮度部分的代码如下

```csharp
Pass
        {
            Name "PASS0"
            HLSLPROGRAM
            #pragma vertex Vert
            #pragma fragment FragB
            half4 FragB(Varyings i ):SV_TARGET
            {
                half4 col = SAMPLE_TEXTURE2D(_BlitTexture,sampler_LinearClamp,i.texcoord);
                float bloom = clamp( luminance(col) - _Threshold,0,1);
                return col * bloom ;
            }
            ENDHLSL
        }
```

```csharp
//提取高光
                Blitter.BlitCameraTexture(cmd ,passSource,buffer1,passMat,0);
```

PASS0共有几个步骤：

1. 采样_BlitTexture（ 一张与`BlitTexture()`相关联的全局纹理，具体来说，当我们在RF中调用`BlitTexture()`或者`BlitCameraTexture()`时会引入一张记录 SourceRTHandle 的名为 _BlitTexture 的全局纹理，在过去的`Blit()`操作中，这个纹理名为 _MainTex ）
2. 计算亮度，并减去阈值后将所得值bloom限制在[0 , 1]。
3. 将采样颜色与bloom系数相乘。

提取亮度后所得图如下:

![](/img/技美百人计划-RenderFeature实现Bloom/1741614931222.png)

### Downsample

降采样可以通过使用MipMap的方式不断去将原图（1920 ，1080 ）向下采样为更小的纹理图，如（960 ， 540）

![](/img/技美百人计划-RenderFeature实现Bloom/1741615426126.png)

其具体数值由设定的参数Downsample决定。

在降采样后再使用高斯模糊使纹理的亮度变化更加平滑。

```csharp
//模糊处理
                for ( int i = 0; i < m_iterations; i++ )
                {
                    cmd.SetGlobalFloat(BloomSie , 1f + i * m_blurSpread);
                    Blitter.BlitCameraTexture(cmd,buffer1,buffer2,passMat,1);
                    cmd.SetGlobalTexture("_TempTex",buffer2);
                    ( buffer1, buffer2 ) = ( buffer2, buffer1 );
                }
```

在RenderFeature中，通过两个临时buffer（假称A， B）相互拷贝，不断将_TempTex进行shader模糊并叠加。

其逻辑如下A-~模糊叠加~->B-~模糊叠加~->A-~模糊叠加~->/...

![](/img/技美百人计划-RenderFeature实现Bloom/1741616006550.png)

模糊的shader代码如下：

```csharp
Pass
        {
            Name "PASS1"
            HLSLPROGRAM
            #pragma vertex Vert
            #pragma fragment FragB

            TEXTURE2D(_TempTex);
            SAMPLER(sampler_TempTex);
  
            half4 FragB(Varyings i ):SV_TARGET
            {
                float2 uv = i.texcoord;
                half4 col = float4(0,0,0,0);
                col += 0.060 * SAMPLE_TEXTURE2D(_BlitTexture, sampler_LinearClamp, uv + float2(-1,-1)*_BlitTexture_TexelSize.xy*_BlurSize);
                col += 0.098 * SAMPLE_TEXTURE2D(_BlitTexture, sampler_LinearClamp, uv + float2(0,-1)*_BlitTexture_TexelSize.xy*_BlurSize);
                col += 0.060 * SAMPLE_TEXTURE2D(_BlitTexture, sampler_LinearClamp, uv + float2(1,-1)*_BlitTexture_TexelSize.xy*_BlurSize);
                col += 0.098 * SAMPLE_TEXTURE2D(_BlitTexture, sampler_LinearClamp, uv + float2(-1,0)*_BlitTexture_TexelSize.xy*_BlurSize);
                col += 0.162 * SAMPLE_TEXTURE2D(_BlitTexture, sampler_LinearClamp, uv);
                col += 0.098 * SAMPLE_TEXTURE2D(_BlitTexture, sampler_LinearClamp, uv + float2(1,0)*_BlitTexture_TexelSize.xy*_BlurSize);
                col += 0.060 * SAMPLE_TEXTURE2D(_BlitTexture, sampler_LinearClamp, uv + float2(1,-1)*_BlitTexture_TexelSize.xy*_BlurSize);
                col += 0.022 * SAMPLE_TEXTURE2D(_BlitTexture, sampler_LinearClamp, uv + float2(1,0)*_BlitTexture_TexelSize.xy*_BlurSize);
                col += 0.060 * SAMPLE_TEXTURE2D(_BlitTexture, sampler_LinearClamp, uv + float2(1,1)*_BlitTexture_TexelSize.xy*_BlurSize);

                half4 col2 = SAMPLE_TEXTURE2D(_TempTex,sampler_LinearClamp,uv);
                return col + col2;
            }
            ENDHLSL
        }
```

### Upsample

一般而言，Upsample需要混合每一级的模糊图像，但是我在Downsample中其实已经完成了这个操作，只不过与通常所用的逻辑不同（A-~模糊~->A1-~模糊~->A2, ...    A1-~叠加~->B, A2-~叠加~->B, ...）

因此，我的Upsample只有混合操作：

RF如下

```csharp
                cmd.SetGlobalTexture(BloomTex,buffer1);
                //混合RT
                Blitter.BlitCameraTexture(cmd,passSource , passSource , passMat,2);
```

shader中操作如下：

```csharp
Pass
        {
            Name "PASS2"
            HLSLPROGRAM
            #pragma vertex VertB
            #pragma fragment FragB
            struct Varying
            {
                float4 uv : TEXCOORD0;
                float4 posCS :SV_POSITION;
            };
            Varying VertB(Attributes v)
            {
                Varying o;
                o.posCS = GetFullScreenTriangleVertexPosition(v.vertexID);
                o.uv.xy = GetFullScreenTriangleTexCoord(v.vertexID);
                o.uv.zw = o.uv.xy;

                #if UNITY_UV_STARTS_AT_TOP
                if(_BlitTexture_TexelSize.y<0.0)
                    o.uv.w = 1.0 - o.uv.w;
                #endif
                return o;
            }
            half4 FragB(Varying i ):SV_TARGET
            {
                half4 bloomCol = SAMPLE_TEXTURE2D(_BloomTex,sampler_BloomTex,i.uv.zw);
                half4 texCol = SAMPLE_TEXTURE2D(_BlitTexture,sampler_LinearClamp,i.uv.xy);
 
                return saturate( texCol + bloomCol * _Intensity);
            }
            ENDHLSL
        }
```

## 最后附上代码

```csharp
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;
using UnityEditor;

public class BloomRF : ScriptableRendererFeature
{
    BloomPass m_BloomPass;
    public RenderPassEvent m_RenderPassEvt = RenderPassEvent.BeforeRenderingPostProcessing;
    public Material mat;
    [Range(1,10)]public int iterations;
    [Range(0,5)]public float blurSpread = 0.7f;
    [Range(1,10)]public int downSample = 1;
    [Range( 0, 10 )] public float intensity = 1;
    [Range( 0, 1 )] public float threshold = 0.1f;
  
    public override void Create()
    {
        m_BloomPass = new BloomPass();
        m_BloomPass.renderPassEvent = m_RenderPassEvt;
        m_BloomPass.passMat = mat;
    }
    public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
    {
        if(!ShouldExecute(renderingData)) return;
        renderer.EnqueuePass(m_BloomPass);
    }

    public override void SetupRenderPasses( ScriptableRenderer renderer, in RenderingData renderingData )
    {
        if(!ShouldExecute(renderingData)) return;
        m_BloomPass.SetUp(renderer.cameraColorTargetHandle,iterations,blurSpread,downSample,intensity,threshold);
    }

    protected override void Dispose( bool disposing )
    {
        m_BloomPass.OnDispose( );
    }
    bool ShouldExecute(in RenderingData data )
    {
        if ( data.cameraData.cameraType != CameraType.Game )
        {
            return false;
        }

        return true;
    }
}
    class BloomPass : ScriptableRenderPass
    {
        private const string ProfileTag = "BloomPass";
        private ProfilingSampler m_ProfilingSampler = new ProfilingSampler( ProfileTag );
  
        private int m_iterations;
        private float m_blurSpread;
        private int m_downSample;
        private float m_Intensity;
        private float m_Threshold;
  
        private const string k_BloomTex = "_BloomTex";
        private const string k_BloomSize = "_BloomSize";
        private const string k_Threshold = "_Threshold";
        private const string k_Intensity = "_Intensity";
        private int BloomTex = Shader.PropertyToID( k_BloomTex );
        private int BloomSie = Shader.PropertyToID( k_BloomSize );
        private int Threshold = Shader.PropertyToID( k_Threshold );
        private int Intensity = Shader.PropertyToID( k_Intensity );
  
        public Material passMat = null;
        private RTHandle passSource;
        private RTHandle buffer1;
        private RTHandle buffer2;

        public void SetUp(RTHandle rt , int iterations = 1,float blurSpread = 0.7f, int downSample = 1,float Intensity = 1 , float Threshold =0.1f )
        {
            this.passSource = rt;
            this.m_iterations = iterations;
            this.m_downSample = downSample;
            this.m_blurSpread = blurSpread;
            m_Intensity = Intensity;
            m_Threshold = Threshold;
        }
        public override void OnCameraSetup(CommandBuffer cmd, ref RenderingData renderingData)
        {
            RenderTextureDescriptor desc = renderingData.cameraData.cameraTargetDescriptor;
            desc.colorFormat = RenderTextureFormat.ARGB32;
            desc.depthBufferBits = 0;
  
            desc.width >>= m_downSample;
            desc.height >>= m_downSample;
  
            RenderingUtils.ReAllocateIfNeeded( ref buffer1, desc ,FilterMode.Bilinear,TextureWrapMode.Clamp,name:"BloomBuffer1");
            RenderingUtils.ReAllocateIfNeeded( ref buffer2, desc ,FilterMode.Bilinear,TextureWrapMode.Clamp,name:"BloomBuffer2");
            ConfigureInput(ScriptableRenderPassInput.Color);
            ConfigureTarget(passSource);
  
  
        }

  
        public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
        {

            CommandBuffer cmd = CommandBufferPool.Get( ProfileTag );
  
            using ( new ProfilingScope(cmd,m_ProfilingSampler) )
            {
                cmd.SetGlobalFloat(Threshold,m_Threshold);
                cmd.SetGlobalFloat( Intensity, m_Intensity );
                //提取高光
                Blitter.BlitCameraTexture(cmd ,passSource,buffer1,passMat,0);
      
                //模糊处理
                for ( int i = 0; i < m_iterations; i++ )
                {
                    cmd.SetGlobalFloat(BloomSie , 1f + i * m_blurSpread);
                    Blitter.BlitCameraTexture(cmd,buffer1,buffer2,passMat,1);
                    cmd.SetGlobalTexture("_TempTex",buffer2);
                    ( buffer1, buffer2 ) = ( buffer2, buffer1 );
                }
      
                cmd.SetGlobalTexture(BloomTex,buffer1);
                //混合RT
                Blitter.BlitCameraTexture(cmd,passSource , passSource , passMat,2);
      
      
            }
            context.ExecuteCommandBuffer(cmd);
            cmd.Clear();
  
            CommandBufferPool.Release(cmd);
        }

        public override void OnCameraCleanup(CommandBuffer cmd)
        {
        }
        public void OnDispose( )
        {
            passSource?.Release();
            buffer1?.Release();
            buffer2?.Release();
  
#if UNITY_EDITOR
            if ( EditorApplication.isPlaying )
            {
                if(passMat != null) Object.Destroy(passMat);
            }
            else
            {
                if(passMat != null) Object.DestroyImmediate(passMat);
            }
#else
            if(passMat != null) Object.Destroy(passMat);
  
#endif
        }
    }
```

BloomShader代码如下：

```csharp
Shader "Unlit/Bloom"
{
    Properties
    {
    }
    SubShader
    {
        Tags { "RenderType"="Transparent" "RenderPipeline"="UniversalRenderPipeline"}
        Cull Off
        ZWrite Off
        ZTest Always

        HLSLINCLUDE
        #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
        #include "Packages/com.unity.render-pipelines.core/Runtime/Utilities/Blit.hlsl"

        CBUFFER_START(UnityPerMaterial)
        float _BlurSize;
        float4 _BlitTexture_TexelSize;
        float _Threshold;
        float _Intensity;
        CBUFFER_END
  
        TEXTURE2D(_BloomTex);
        SAMPLER(sampler_BloomTex);

        float luminance(float4 color)
        {
            return 0.2125*color.r + 0.7154 * color.g + 0.0721 * color.b;
        }
  
        ENDHLSL

        Pass
        {
            Name "PASS0"
            HLSLPROGRAM
            #pragma vertex Vert
            #pragma fragment FragB
            half4 FragB(Varyings i ):SV_TARGET
            {
                half4 col = SAMPLE_TEXTURE2D(_BlitTexture,sampler_LinearClamp,i.texcoord);
                float bloom = clamp( luminance(col) - _Threshold,0,1);
                return col * bloom ;
            }
            ENDHLSL
        }
        Pass
        {
            Name "PASS1"
            HLSLPROGRAM
            #pragma vertex Vert
            #pragma fragment FragB

            TEXTURE2D(_TempTex);
            SAMPLER(sampler_TempTex);
  
            half4 FragB(Varyings i ):SV_TARGET
            {
                float2 uv = i.texcoord;
                half4 col = float4(0,0,0,0);
                col += 0.060 * SAMPLE_TEXTURE2D(_BlitTexture, sampler_LinearClamp, uv + float2(-1,-1)*_BlitTexture_TexelSize.xy*_BlurSize);
                col += 0.098 * SAMPLE_TEXTURE2D(_BlitTexture, sampler_LinearClamp, uv + float2(0,-1)*_BlitTexture_TexelSize.xy*_BlurSize);
                col += 0.060 * SAMPLE_TEXTURE2D(_BlitTexture, sampler_LinearClamp, uv + float2(1,-1)*_BlitTexture_TexelSize.xy*_BlurSize);
                col += 0.098 * SAMPLE_TEXTURE2D(_BlitTexture, sampler_LinearClamp, uv + float2(-1,0)*_BlitTexture_TexelSize.xy*_BlurSize);
                col += 0.162 * SAMPLE_TEXTURE2D(_BlitTexture, sampler_LinearClamp, uv);
                col += 0.098 * SAMPLE_TEXTURE2D(_BlitTexture, sampler_LinearClamp, uv + float2(1,0)*_BlitTexture_TexelSize.xy*_BlurSize);
                col += 0.060 * SAMPLE_TEXTURE2D(_BlitTexture, sampler_LinearClamp, uv + float2(1,-1)*_BlitTexture_TexelSize.xy*_BlurSize);
                col += 0.022 * SAMPLE_TEXTURE2D(_BlitTexture, sampler_LinearClamp, uv + float2(1,0)*_BlitTexture_TexelSize.xy*_BlurSize);
                col += 0.060 * SAMPLE_TEXTURE2D(_BlitTexture, sampler_LinearClamp, uv + float2(1,1)*_BlitTexture_TexelSize.xy*_BlurSize);

                half4 col2 = SAMPLE_TEXTURE2D(_TempTex,sampler_LinearClamp,uv);
                return col + col2;
            }
            ENDHLSL
        }
        Pass
        {
            Name "PASS2"
            HLSLPROGRAM
            #pragma vertex VertB
            #pragma fragment FragB
            struct Varying
            {
                float4 uv : TEXCOORD0;
                float4 posCS :SV_POSITION;
            };
            Varying VertB(Attributes v)
            {
                Varying o;
                o.posCS = GetFullScreenTriangleVertexPosition(v.vertexID);
                o.uv.xy = GetFullScreenTriangleTexCoord(v.vertexID);
                o.uv.zw = o.uv.xy;

                #if UNITY_UV_STARTS_AT_TOP
                if(_BlitTexture_TexelSize.y<0.0)
                    o.uv.w = 1.0 - o.uv.w;
                #endif
                return o;
            }
            half4 FragB(Varying i ):SV_TARGET
            {
                half4 bloomCol = SAMPLE_TEXTURE2D(_BloomTex,sampler_BloomTex,i.uv.zw);
                half4 texCol = SAMPLE_TEXTURE2D(_BlitTexture,sampler_LinearClamp,i.uv.xy);
  
                return saturate( texCol + bloomCol * _Intensity);
            }
            ENDHLSL
        }
    }
}
```

## 小憩

* 本次Bloom效果其实并不算好的，与URP自带的Bloom效果相比仍有较大差距，URP自带后处理Bloom（第一张）与本人自写Bloom效果（第二张）对比。自写Bloom中有几个问题：锯齿 、 光的延伸效果不好（模糊范围太小）、Downsample值会影响画面质量

![](/img/技美百人计划-RenderFeature实现Bloom/1741616882770.png)

![](/img/技美百人计划-RenderFeature实现Bloom/1741614889449.png)

* 我还在项目中遇到一些问题：如临时Buffer没有复制原纹理，经过反复测试才发现，临时Buffer的Release位置放在了`OnCameraCleanUp()`，导致每一帧Buffer都被释放，无法存储纹理，正确的位置应该放在RF的`Dispose()`里面调用`Buffer?.Realease()`。

之后应该会参考Unity的Bloom源码和参考更多资料，将Bloom效果改进。

## 参考

[[urp Bloom学习笔记 - 知乎](https://zhuanlan.zhihu.com/p/10316984143)]

[高质量Bloom——URP后处理实现 - 知乎](https://zhuanlan.zhihu.com/p/785962352)

[用Unity实现Bloom - 知乎](https://zhuanlan.zhihu.com/p/558854248)
