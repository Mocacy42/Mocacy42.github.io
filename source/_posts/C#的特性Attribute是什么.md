---
title: C#的特性Attribute是什么
date: 2026-04-22
tag: [C#]
categories: [Tech, Program]
toc: true
comments: true
---
## 什么是特性(Attribute)

微软的文档中给出：

公共语言运行时使你能够添加类似于关键字的描述性声明（称为特性），以便批注编程元素（如类型、字段、方法和属性）。 当您为运行时编译代码时，它会被转换为公共中间语言（CIL），并与编译器生成的元数据一起放置在可移植可执行文件（PE）中。 通过属性，可以将额外的描述性信息放入可以使用运行时反射服务提取的元数据中。

简单理解，特性就是给类、结构体、方法等元素添加元数据描述，这个元数据会在编译时被处理并加入最终文件。

## 添加特性

添加特性可以使用如下模式，为`element`添加特性。

```C#
//  模式
[attribute(positional_parameters 必需构造信息 ， name_parameter 可选信息)]
element
```

## 预定义特性

`.Net`为用户提供了三种预定义特性：

- `AttributeUsage`
- `Conditional`
- `Obsolete`

### `AttributeUsage` 自定义特性

`AttributeUsage`用于用户自定义特性。

```c#
[AttributeUsage(
  validon, 
  AllowMultiple=false,
  Inherited=false  
)]
```

- `validon`规定了该特性可以应用的元素类型，如`AttributeTargets.Class | AttributeTargets.Field | AttributeTargets.Method `通过位枚举表示该特性可以应用于类、字段、方法。
- `AllowMultiple`是可选`bool`参数，表示特性是否可以多用。
- `Inherited`是可选`bool`参数，表示特性是否可以被派生类继承。

如下是一个创建自定义特性的示例：

```c#
[AttributeUsage(AttributeTargets.Method | AttributeTargets.Class)]
public class CustomAttribute : System.Attribute
{
  private int id;
  private string user;
  public message;
  
  public CustomAttribute(int id, string user)
  {
    this.id = id;
    this.user = user;
  }
  public int Id { get { return id; } }
  public string User { get { return user; } }
  public string Message { get { return message; } set { message = value; } }
} 

// 使用特性
[CustomAttibute(0, "class" , Message = ("This is a custom attribute applied on class"))]
class Debugclass
{
  private int a, b;
  [CustomAttibute(0, "method" , Message = ("This is a custom attribute applied on method"))]
  public int add () { return a + b;}
}


```

可以看到，`AttributeUsage`的类，构造函数的参数成为必需参数，未在构造函数中的参数成为可选参数，相当于使用一个类来记录元数据。

### `Conditional`条件编译特性

`Conditional`会标记一个条件方法，执行依赖预处理标识符。

```c#
[Conditional("DEBUG")]
string Message(string msg) { Console.WriteLine(msg); }

// 在作用效果上相当于
string Message(string msg) {  
  #ifdefine DEBUG 
    Console.WriteLine(msg); 
  #endif
}
```

### `Obsolete`废弃特性标记

使用该特性标记的元素表示这个元素不应该被使用（预备废弃或已被废弃）。

```c#
[Obsolete( message , iserror = false)]
```

- `message`：字符串，表示为什么被废弃。
- `iserror`：`bool`参数，`false`生成警告，`true`生成错误。
