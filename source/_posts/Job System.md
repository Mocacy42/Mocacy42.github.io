---
title: Job System
date: 2025-11-25 15:27:00
tags: [Unity, JobSystem, 多线程]
categories: [Tech, Unity]
toc: true
comments: true
---
# Job System 

## Job System 介绍

Unity 的 Job System 是多线程任务调度框架，让开发者可以只用 C# 写出安全高效的多线程代码，并与 Unity 引擎内部的原生作业系统共享线程池，避免创建过多线程导致 CPU 争用。

### 核心思想

其核心思想就是将每个任务做成一个 Struct ，让其实现 `IJob`,`IJobParallelFor`等接口，其中仅包含数据和对数据的处理逻辑而无引用类型。
在主线程中调用 `Schedule()`将 Job 推入队列，将 Job 分发给子线程，并使用`Complete()`等待 Job完成。
### 安全性系统

为了避免子线程对主线程内容操作从而造成竞争，Job System 会采用数据拷贝的方式提供子线程数据，即：Job 中操作的数据都是主线程中数据的拷贝而非引用。

这样的方式也导致 Job 只能访问可位块传输的数据类型( Blittable data type )。
基本类型：
- `byte \ sbyte`
- `short \ ushort`
- `int \ uint`
- `long \ ulong`
- `float (System.Single)`
- `double (System.Double)`
- `IntPtr \ UIntPtr `

复杂类型：
- 一维数组
- 结构体 \ 类（只包含 Blittable 类型 并使用 `[StructLayout(LayoutKind.Sequential)]`或`Explicit`显式标识）

非 Blittable 类型：
- `string`
- `char`
- `bool`
- `object`
- ...

拷贝数据的缺陷是 Job 计算与外界隔离，为了获取结果，可以分配一块共享内存 —— NativeContainer。

### NativeContainer

默认提供的 NativeContainer 是`NativeArray<T>`。其安全性有两项保障
- `DisposeSentinel`：检测内存泄漏，保证释放内存。
- `AtomicSafetyHandle`：进行 NativeContainer 所有权转让，避免两个 Job 同时写入同一块内存。

除此之外，编辑器环境下会检查下标边界检查、内存释放检查、资源竞争检查。

一般，NativeContainer 开放读写权限，如果希望只读或只写，应该约束：
- `[ReadOnly]`
- `[WriteOnly]`

访问静态数据可以绕过所有的安全性系统并可能导致Unity奔溃。

```C#
NativeArray<float> result = new NativeArray<float>(arraySize, Allocator.TempJob);
```
为 NativeArray 分配内存时有三种分配器可选：
- `Allocator.Temp`是最快的分配类型。它适用于分配一个生命周期只有一帧或更短时间的操作。你不应当把一个分配器为Temp类型分配的NativeContainer传递给jobs使用。你同时需要在函数返回之前调用Dispose方法(例如MonoBehaviour.Update，或者其他从原生到托管代码的调用)
- `Allocator.TempJob`是相比于 Temp 是一个较慢的分配类型但它比 Persistent 要快。这是一个生命周期为四帧的内存分配而且它是线程安全的。如果你在四帧之内没有调用 Dispose，控制台会打印一个由原生代码生成的警告信息。绝大部分小 jobs 使用这种类型的 NativeContainer 分配器。
- `Allocator.Persistent`是最慢的分配类型但，它可以持续存在到你需要的时间，如果必要的话可以贯穿应用程序的整个生命周期。它是直接调用malloc 的一个封装。长时间的 jos 可以使用这种分配类型。当性能比较紧张的时候你不应当使用 Persistent。

## 创建 Job

### Simple Job Example

要创建一个 Job，只需要创建一个实现了`IJob`的结构体并添加 Blittable 类型数据，创建 `Execute`方法即可。

```C#
public struct AddJob : IJob
{
	public float a;
	public float bl
	public NativeArray<float> result;
	
	public void Execute()
	{
		result[0] = a + b;
	}
}
```

要在主线程中调用，需要使用`Schedule`方法。

```C#
NativeArray<float> result = new NativeArray<float>(1 , Allocator.TempJob);

AddJob jobData = new AddJob();
jobData.a = 10;
jobData.b = 20;
jobData.result = result;

JobHandle handle = jobData.Schedule();

handle.Complete();

float finalResult = result[0];

result.Dispose();
```

### Dependencies Job Example

如果一个 Job B 需要另外一个 Job A 的计算结果，就称 Job B 依赖 Job A。
可以使用`JobHandle.CombineDependencies`方法合并依赖项，并且将之传递给`Schedule`方法。

```C#
NativeArray<JobHandle> handles = new NativeArray<JobHandle>(numJobs, Allocator.TempJob);

JobHandle combineHandle = JobHandle.CombineDependencies(handles);

newHandle.Schedule(combineHandle);
```

### 阻塞主线程

使用`Complete`方法可以阻塞主线程直到 Job 完成，并将 NativeContainer 类型的归属权交给主线程。

可以调用 Job A 的`Complete`方法，或者你可以调用依赖于 Job A 的 Job B的`Complete`方法。两种方法都可以在调用`Complete`后在主线程安全访问 Job A 使用的 NativeContainer 类型。

```C#
// Job 
public struct MyJob : IJob
{
    public float a;
    public float b;
    public NativeArray<float> result;

    public void Execute()
    {
        result[0] = a + b;
    }
}
public struct AddOneJob : IJob
{
    public NativeArray<float> result;

    public void Execute()
    {
        result[0] = result[0] + 1;
    }
}
// Main Thread
NativeArray<float> result = new NativeArray<float>(1, Allocator.TempJob);

MyJob jobData = new MyJob();
jobData.a = 10;
jobData.b = 10;
jobData.result = result;
JobHandle firstHandle = jobData.Schedule();

AddOneJob incJobData = new AddOneJob();
incJobData.result = result;
JobHandle secondHandle = incJobData.Schedule(firstHandle);

secondHandle.Complete();

float aPlusB = result[0];

result.Dispose();
```

如果不需要对数据的访问，但需要明确地刷新这个批次的 Job。为了做到这点，调用静态方法`JobHandle.ScheduleBatchedJobs`。注意这个调用会对性能产生负面的影响。

这个方法常常用于分帧任务中。

```C#
void Update()
{
    if (!handle.IsCompleted)
    {
        // 帧末先推一次
        JobHandle.ScheduleBatchedJobs();
        return;
    }

    handle.Complete();  // 下一帧再来检查
    UseResults();
}
```

## ParallelFor Job

Unity 中常常需要对大量数量做相同操作，`IJobParallelFor`专为此情况设计。使用 ParallelFor Job 调度时，只能有一个 Job。

```C#
// ParallelFor Job
struct IncrementByDeltaTimeJob : IJobParallelFor
{
    public NativeArray<float> values;
    public float deltaTime;

    public void Execute(int index)
    {
        float temp = values[index];
        temp += deltaTime;
        values[index] = temp;
    }
}
// Main Thread
NativeArray<float> values = new NativeArray<float>(100, Allocator.TempJob);

IncrementByDeltaTimeJob jobData = new IncrementByDeltaTimeJob
{
    values = values,
    deltaTime = Time.deltaTime
};

JobHandle handle = jobData.Schedule(values.Length, 1); 
handle.Complete(); 
```

`Schedule(int length ,int innerLoopBatchCount)`的参数控制如何并行处理。
- `length`：表示源数组大小，方便`index`遍历。
- `innerLoopBatchCount`：控制线程最大处理的批次上限，就是单个工作线程处理最大任务数。

Parallel Job 有为 Transforms 专门提供的类型 —— `ParallelForTransform`。

## 接口

| 接口                       | 执行模式 | 核心方法                                | 用途                  | 关键点                                          |
| ------------------------ | ---- | ----------------------------------- | ------------------- | -------------------------------------------- |
| IJob                     | 单任务  | Execute()                           | 单一独立任务，整体操作数据       | 最简单，一次执行                                     |
| IJobParallelFor          | 数据并行 | Execute(int index)                  | 处理大型数组/列表，无数据竞争     | 最常用，高性能，需注意线程安全                              |
| IJobParallelForTransform | 数据并行 | Execute(int index, TransformAccess) | 高效并行处理大量Transforms  | IJobParallelFor 的特化版，需用 TransformAccessArray |
| IJobFor                  | 串行循环 | Execute(int index)                  | 需要Burst优化但存在数据依赖的循环 | 不是并行的，用于Burst优化单线程循环                         |

## 参考

[Unity相关技术 - 知乎](https://zhuanlan.zhihu.com/c_1078686690062942208)

[Jobs overview - Unity 手册](https://docs.unity.cn/cn/2022.3/Manual/job-system-jobs.html)

[ Dots实战七 Job System的使用 - 知乎](https://zhuanlan.zhihu.com/p/1960755007281411274)

[Unity JobSystem使用手册 - 知乎](https://zhuanlan.zhihu.com/p/1944344142680392088)