---
title: Unity Compute Shader
data: 2025-11-23 20:18:00
tags: [Unity,ComputeShader]
categories: [Tech,Unity]
toc: true
comments: true
---
# Compute Shader 计算着色器
## 1. GPGPU
相较于在 CPU 少核架构而言，当代 GPU 多核架构设计决定了其在大规模并行计算问题上的的性能优势。

这种优势本是为了图形渲染准备的，但是也可以将之运用于非图形渲染应用中，叫做 GPGPU ( General Purpose GPU ) 编程。

CPU 和 GPU 之间的通信非常慢，但是与使用 GPU 计算得到的性能提升相比，这不值一提。

![CPU GPU 相对内存带宽速度](/img/ComputeShader/XPUCorrespondingSpeed.png)

用户所需要做的就是
- 在 CPU 准备数据。
- 将数据传给 GPU 。
- GPU 计算( Compute Shader )。
- 返回数据至 CPU 。

Compute Shader ( CS ) 是独立于渲染管线之外的可编程渲染器，可以对 GPU 资源进行读写操作，本身并不绘制任何图形。

CS 本质是显卡以及图形 API 的产物，Unity 在 Direct3D API 的基础上进行了封装。

## 2. Unity 中的 Compute Shader

### 默认文件 Default Compute Shader

Unity 生成的 Compute Shader 文件属于 Asset 文件，以 `.compute`作为后缀。
其默认内容如下：

```cs
#pragma kernel CSMain

RWTexture2D<float4> Result;

[numthreads(8,8,1)]
void CSMain (uint3 id : SV_DispatchThreadID)
{
	Result[id.xy] = float4(id.x & id.y, (id.x & 15)/15.0, (id.y & 15)/15.0, 0.0);
}
```

### 核函数声明

```cs
#pragma kernel CSMain`
```

`CSMain`就是一个函数名，`kernel`表示内核，这一行将`CSMain`声明为核函数，最终会在 GPU 中调用。CS 中至少需要一个`kernel`才可以被调用。

CS 中可以声明多个`kernel`，此外，指令后面可以定义预处理宏命令。（注释不应卸载该命令后面）

### 变量声明

```cs
RWTexture2D<float4> Result;
```

`RW`分别表示`Read`和`Write`，因此这行表示声明了一张可以被 CS 读写的二维纹理，只读不写可以直接使用`Texture2D`，只写不读没有对于 API，需要自己写代码规范。

这样一张 2D 纹理，可以像这样获取单个像素：`Result[uint2(0,0)]`，其中每个像素的值就是`float4`。

在 CS 中，可以读写的类型除了`RWTexture`以外还有`RWBuffer`和`RwStructuredBuffer`。

CS 提供了一系列可读写的类型，这些类型被称为 Unordered Access View (UAV) ：

|           类型            |    含义    |
| :---------------------: | :------: |
|    `RWTexture1D<T>`     | 1D 可读写纹理 |
|  `RWTexture1DArray<T>`  | 1D 可读写纹理 |
|    `RWTexture2D<T>`     | 2D 可读写纹理 |
|  `RWTexture2DArray<T>`  | 2D 可读写纹理 |
|    `RWTexture3D<T>`     | 3D 体素纹理  |
|      `RWBuffer<T>`      |  一维线性缓存  |
| `RWStructuredBuffer<T>` | 一维结构化缓存  |
|  `RWByteAddressBuffer`  | 一维字节寻址缓存 |
|  `RWByteAddressBuffer`  | 一维可写字节缓存 |
此外，还有一系列只读类型，称为 Shader Resource View (SRV) ：

|                 类型                 |    含义     |
| :--------------------------------: | :-------: |
|   `Texture1D` / `Texture1DArray`   |  1D 只读纹理  |
|   `Texture2D` / `Texture2DArray`   |  2D 只读纹理  |
|            `Texture3D`             | 3D 只读体纹理  |
| `TextureCube` / `TextureCubeArray` |   立方体贴图   |
|            `Buffer<T>`             | 一维只读线性缓存  |
|       `StructuredBuffer<T>`        | 一维只读结构化缓存 |
|        `ByteAddressBuffer`         | 一维只读字节寻址  |

### 线程
#### 为线程组分配线程

```cs
[numthreads(tX,tY,tZ)]
// t: thread id
```

这个属性定义了一个**线程组（ThreadGroup）** 中可以被执行的**线程（Thread）** 总数。这里代表有`tX * tY * tZ`个线程。每个核函数前面都应该有该属性，否则编译报错。

要执行的所有线程被划分为数个线程组，每个线程组运行在单个流多处理器（Stream Multiprocessor， SM）上。为了保证 SM 不会空转，每个 SM 至少需要两个线程组，这叫延迟隐藏，SM 可以切换到处理不同组中的线程以提高利用率。

每个线程组内有一个独立的共享内存，组内线程可以访问，线程组之间无法相互访问，这是组内线程同步操作的前提。

tX，tY，tZ 的值在不同 CS 版本中有不同约束：

| Compute Shader Version | tZ Max | tX * tY * tZ Max |
| :--------------------: | :----: | :--------------: |
|         cs_4_x         |   1    |       768        |
|         cs_5_0         |   64   |       1024       |
在 NAVIDIA 显卡（按照 SIMD32 规范）中，每个 SM 在硬件层使用 Warp 执行线程，而一个 Warp 可以控制 32 个线程执行相同指令，因此为了避免空转，线程总数应该为 32 的倍数。在 AMD 显卡中，wavefront 可以控制 64 个线程。综合建议将 `numthreads`值设置为 64 的倍数，以兼容两种显卡。

#### 分配线程组

在 Direct3D12 中，可以使用`ID3D12GraphicsCommandList::Dispatch(gX,gY,gZ)`创建`gX * gY * gZ`个线程组，其在不同 CS 版本中亦有约束：

| Compute Shader Version | gX gY Max | gZ Max |
| :--------------------: | :-------: | :----: |
|         cs_4_x         |   65535   |   1    |
|         cs_5_0         |   65535   | 65535  |

![Thread And Thread Group 分配情况](/img/ComputeShader/ThreadAndThreadGroup.png)

上半部分表示线程组分配，下半部分表示每个线程组中的线程分配。
- `SV_GroupID`：表示线程所在线程组的 ID。
- `SV_GroupThreadID`：表示线程在线程组内的相对 ID。
- `SV_DispatchThreadID`：表示线程在所有线程中的绝对 ID，计算方式：`SV_DispatchThreadID = SV_GroupID * numthreads + SV_GroupThreadID`
- `SV_GroupIdex`：表示线程在当前线程组内的下标，计算方式：`SV_GroupIndex = SV_GroupThreadID.z * numthreads.x * numthreads.y + SV_GroupThreadID.y * numthreads.x + SV_GroupThreadID.x`

在分配线程组的时候可以想象，将大小为（ x, y, z ）（和计算规模直接相关）的立方体用大小为（ tX，tY，tZ ）的立方体填充，得到结果就是 （ gX，gY，gZ ）。

### 核函数

```cs
void CSMain(uint3 id : SV_DispatchThreadID)
{
	Result[id.xy] = float4(id.x & id.y, (id.x & 15)/15.0, (id.y & 15)/15.0, 0.0);
}
```

除了传递`SV_DispatchThreadID`也可以传递其他参数：
```cs
void KernelFunction(uint3 groupId : SV_GroupID,
    uint3 groupThreadId : SV_GroupThreadID,
    uint3 dispatchThreadId : SV_DispatchThreadID,
    uint groupIndex : SV_GroupIndex) { }
```

在本例中，`Texture2D`被分为 64个像素作为一个线程组，每个线程组恰好 64 个线程，因此每个线程处理一个像素。`id.xy`被设计专门在线程组间不会重复访问，在组内遍历每个线程。

### CS 与 C# 脚本交互
#### Texture2D Example

CS 强制使用 C# 驱动，`.compute`可以被 C# 引用。在本例中可以在 C# 中创建一个`RenderTexture`，并且将之赋给`Result`对象。
```C#
public class ComputeShaderExample :　MonoBehaviour
{
	public ComputeShader computeShader;
	public Material material;
	private RenderTexture rt;
	
	void Start()
	{
		// 0 表示不需要深度缓冲
		rt = new RenderTexture(256,256,0);
		rt.enableRandomWrite = true;
		rt.Create();
		
		// 将 _MainTex 设置为创建的 RenderTexture
		material.mainTexture = rt;
		int kernelIndex = computeShader.FindKernel("CSMain");
		computeShader.SetTexture(kernelIndex,"Result",rt);
		computeShader.Dispatch(kernelIndex,256/8,256/8,1);
	}
	void OnDestroy()
	{
		if(rt) rt.Release();
	}
}
```

这个脚本创建了`RenderTexture`并且将之传递给`ComputeShader`计算每个像素的颜色，最后使用`Material`可以赋给其他 Mesh。

#### RWStructuredBuffer Example

CS 也可以获取自定义结构体，而不局限于`int`这样的基本类型。
```cs
#pragma kernel CSMain

struct ParticleData {
	float3 pos;
	float4 color;	
};

RWStructuredBuffer<ParticleData> ParticleBuffer;

float time;

[numthreads(10,10,10)]
void CSMain(uint3 gid : SV_GroupID, uint index : SV_GroupIndex)
{
	int pindex = gid.x * 1000 + index;
	
	float x = sin(index);
	float y = sin(index * 1.2f);
	float3 forward = float3(x, y, -sqrt(1 - x * x - y * y));
	ParticleBuffer[pindex].color = float4(forward.x,forward.y,cos(index) * 0.5f + 0.5f, 1);
	if(time > gid.x)
		ParticleBuffer[pindex].pos += forward * .005f;
}

```

计算后的结果如何使用？这就需要使用`Shader`。
```ShaderLab
Shader "Unlit/ParticleShaderURP"
{
    SubShader
    {
        Tags { "RenderPipeline" = "UniversalPipeline" "RenderType" = "Opaque" }
        LOD 100

        Pass
        {
            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            #pragma multi_compile_instancing
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

            struct particleData
            {
                float3 pos;
                float4 color;
            };

            StructuredBuffer<particleData> _particleDataBuffer;

            struct Attributes
            {
                uint vertexID : SV_VertexID;
            };

            struct Varyings
            {
                float4 positionHCS : SV_POSITION;
                float4 color       : COLOR0;
            };

            Varyings vert(Attributes IN)
            {
                Varyings OUT;
                particleData p = _particleDataBuffer[IN.vertexID];
                float3 worldPos = TransformObjectToWorld(p.pos);
                OUT.positionHCS = TransformWorldToHClip(worldPos);
                OUT.color = p.color;
                return OUT;
            }

            half4 frag(Varyings IN) : SV_Target
            {
                return IN.color;
            }
            ENDHLSL
        }
    }
}

```

接下来需要在 C# 中为`RWStructuredBuffer`赋值。Unity 提供了 `ComputeBuffer`类专门实现这类操作。

```C#
public class StructuredBufferExample : MonoBehaviour
{
	public ComputeShader computeShader;
	public Material material;
	// 结构体数目
	[SerializeField] private int _count = 2000;
	private int _kernelId;
	private ComputeBuffer _buffer;
	
	public struct ParticleData {
		public vector3 pos;
		public Color color;	
	};
	
	void Start()
	{
		// 结构体字节数
		int stride = UnsafeUtility.SizeOf<ParticleData>();
		 
		_buffer = new ComputeBuffer(_count, stride);
		ParticleData[] particleDatas = new ParticleData[count];
		_buffer.SetData(particleDatas);
		_kernelId = computeShader.FindKernel("CSMain");
	}
	void Update()
	{
		computeShader.SetBuffer(_kernelId, "ParticleBuffer", _buffer);
		computeShader.SetFloat("time",Time.time);
		computeShader.Dispatch(_kernelId, _count/1000,1,1);
		material.SetBuffer("_particleDataBuffer",_buffer);
	}
	void OnRenderObject()
	{
		material.SetPass(0);
		Graphics.DrawProceduralNow(MeshTopology.Points,_count);
	}
	void OnDestroy()
	{
		if(_buffer) _buffer.Release(); 
	}
}
```

#### ComputeBuffer

上面使用 ComputeBuffer 使 C# 脚本能够与 CS 、 Shader 沟通，但是并没有设置 ComputeBuffer 的类型。

实际上， Unity 的 ComputeBuffer 类拥有一系列类型`ComputeBufferType.XXX`，这些类型为 CS 提供了对应的 ComputeBuffer 修改方法：

| ComputeBufferType |                                                                                                                 内容                                                                                                                  |
| :---------------: | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------: |
|      Default      |                                                                    ComputeBuffer的默认类型，对应HLSL shader中的StructuredBuffer或RWStructuredBuffer，常用于自定义Struct的Buffer传递。                                                                     |
|        Raw        |                                                     Byte Address Buffer，把里面的内容（byte）做偏移，可用于寻址。它对应HLSL shader中的 ByteAddressBuffer 或RWByteAddressBuffer，用于着色器访问的底层DX11格式为无类型的R32。                                                     |
|      Append       |                                                 Append and Consume Buffer，允许我们像处理 Stack 一样处理 Buffer，例如动态添加和删除元素。它对应 HLSL shader 中的AppendStructuredBuffer 或 ConsumeStructuredBuffer。                                                 |
|      Counter      |                                 用作计数器，可以为 RWStructuredBuffer 添加一个计数器，然后在ComputeShader 中使用 IncrementCounter 或 DecrementCounter 方法来增加或减少计数器的值。由于 Metal 和 Vulkan 平台没有原生的计数器，因此我们需要一个额外的小buffer用来做计数器。                                  |
|     Constant      | constant buffer (uniform buffer)，该 buffer 可以被当做 Shader.SetConstantBuffer 和 Material.SetConstantBuffer 中的参数。如果想要绑定一个 structured buffer 那么还需要添加ComputeBufferType.Structured，但是在有些平台（例如 DX11）不支持一个 buffer 即是 constant 又是 structured 的。 |
|    Structured     |                                                                                             如果没有使用其他的 ComputeBufferType 那么等价于 Default。                                                                                              |
| IndirectArguments |                               被用作 Graphics.DrawProceduralIndirect，ComputeShader.DispatchIndirect 或Graphics.DrawMeshInstancedIndirect 这些方法的参数。buffer 大小至少要 12 字节，DX11底层 UAV 为 R32_UINT，SRV 为无类型的 R32。                                |

注意：Default，Append，Counter，Structured对应的Buffer每个元素的大小，也就是stride的值应该是4的倍数且小于2048。

同一块带 UAV 的 StructuredBuffer 可以被绑定到：
- `AppendStructuredBuffer`：可以使用`.Append()`方法推入元素。
- `ConsumeStructuredBuffer`：可以使用`.Consume()`方法弹出元素。

上述 Default Raw Append Counter 都是自带计数器，可以绑定到`RW/Append/Consum StructuredBuffer`。

下面的例子介绍了如何计数以及如何获取计数。
```cs
#pragma kernel CSMain

struct Particle {
    float3 pos;
    float life;
};

ConsumeStructuredBuffer<Particle> inputParticles;  // 消费输入
AppendStructuredBuffer<Particle> outputParticles;  // 追加输出

[numthreads(64,1,1)]
void CSMain() {
    // 消费一个粒子（线程安全，每个粒子只被消费一次）
    Particle p = inputParticles.Consume();

    // 简单处理：更新位置 + 减少生命
    p.pos += float3(0.1, 0, 0);
    p.life -= 0.1;

    // 只保留活着的粒子
    if (p.life > 0) {
        outputParticles.Append(p);  // 追加到输出
    }
}
```



```c#
public class ConsumeAppendExample : MonoBehaviour
{
    public ComputeShader shader;
    ComputeBuffer inputBuffer;
    ComputeBuffer outputBuffer;
    ComputeBuffer argsBuffer;

    void Start()
    {
        int count = 100;
        int stride = sizeof(float) * 4; 
        
        // 创建输入数据
        Particle[] particles = new Particle[count];
        for (int i = 0; i < count; i++)
        {
            particles[i] = new Particle
            {
                pos = UnityEngine.Random.insideUnitSphere,
                life = UnityEngine.Random.Range(0.5f, 1.0f)
            };
        }

        // 创建 Consume 缓冲区
        inputBuffer = new ComputeBuffer(count, stride, ComputeBufferType.Default);
        inputBuffer.SetData(particles);

        // 创建 Append 缓冲区
        outputBuffer = new ComputeBuffer(count, stride, ComputeBufferType.Append);
        // 初始化计数器的值
        outputBuffer.SetCounterValue(0);

        int kernel = shader.FindKernel("CSMain");
        shader.SetBuffer(kernel, "inputParticles", inputBuffer);   
        shader.SetBuffer(kernel, "outputParticles", outputBuffer); 
        
        shader.Dispatch(kernel, Mathf.CeilToInt(count / 64.0f), 1, 1);

        // 获取输出数量
        argsBuffer = new ComputeBuffer(1, sizeof(int), ComputeBufferType.IndirectArguments);
        ComputeBuffer.CopyCount(outputBuffer, argsBuffer, 0);
        int[] args = new int[1];
        argsBuffer.GetData(args);
        Debug.Log("输出粒子数量: " + args[0]);
    }

    void OnDestroy()
    {
        inputBuffer?.Release();
        outputBuffer?.Release();
        argsBuffer?.Release();
    }

    struct Particle
    {
        public Vector3 pos;
        public float life;
    }
}
```

在 C# 中的代码有两点需要注意：
- `SetCounterValue(0)`：要使用计数器需要提前初始化计数器。
- `ComputeBuffer.CopyCount()`：获取计数，需要通过一个`IndirectArguments`类型的 ComputeBuffer 间接获取计数，否则直接使用`GetData`会直接获取数据数组而不知道其长度。

### 共享内存

通过`groundshared`关键字声明的变量会被放入共享内存中，线程对其的访问几乎与硬件缓存一样快。常常用于缓存像素值，避免不同线程重复采样。

Direct3D 11以来，共享内存支持的最大大小为32kb（之前的版本是16kb），并且单个线程最多支持对共享内存进行256byte的写入操作。

```cs
Texture2D input;
groupshared float4 cache[256];

[numthreads(256, 1, 1)]
void CS(int3 groupThreadID : SV_GroupThreadID, int3 dispatchThreadID : SV_DispatchThreadID)
{
    cache[groupThreadID.x] = input[dispatchThreadID.xy];
    
	// 同步等待组内所有线程到达，并确保内存的写入对后续读取可见。
    GroupMemoryBarrierWithGroupSync();
	
	// 安全读取
    float4 left = cache[groupThreadID.x - 1];
    float4 right = cache[groupThreadID.x + 1];
    ......
}
```

`GroupMemoryBarrierWithGroupSync()`强制同步等待线程组内所有线程完成，一般会导致线程空转。

### shader 变体
CS同样支持[shader变体](https://link.zhihu.com/?target=https%3A//docs.unity3d.com/2020.3/Documentation/Manual/SL-MultipleProgramVariants.html)，用法和普通的shader变体基本相似，示例如下：

```cs
#pragma kernel CSMain
#pragma multi_compile __ COLOR_WHITE COLOR_BLACK

RWTexture2D<float4> Result;

[numthreads(8,8,1)]
void CSMain (uint3 id : SV_DispatchThreadID)
{
#if defined(COLOR_WHITE)
	Result[id.xy] = float4(1.0, 1.0, 1.0, 1.0);
#elif defined(COLOR_BLACK)
	Result[id.xy] = float4(0.0, 0.0, 0.0, 1.0);
#else
	Result[id.xy] = float4(id.x & id.y, (id.x & 15) / 15.0, (id.y & 15) / 15.0, 0.0);
#endif
}
```

然后我们就可以在C#端启用或禁用某个变体了：

- `#pragma multi_compile`声明的全局变体可以使用`Shader.EnableKeyword/Shader.DisableKeyword`或者`ComputeShader.EnableKeyword/ComputeShader.DisableKeyword`
- `#pragma multi_compile_local`声明的局部变体可以使用`ComputeShader.EnableKeyword/ComputeShader.DisableKeyword`

示例如下：

```c#
public class DrawParticle : MonoBehaviour
{
    public ComputeShader computeShader;

    void Start() {
        ......
        computeShader.EnableKeyword("COLOR_WHITE");
    }
}
```
## 参考

[【Unity】 Compute Shader的基础介绍与使用 - 知乎](https://zhuanlan.zhihu.com/p/368307575)

[Unity - Manual: Compute shaders](https://docs.unity3d.com/Manual/class-ComputeShader.html)

