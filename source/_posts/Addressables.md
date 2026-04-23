---
title: Unity的Addressables
date: 2025-11-10 1:52:00
tags: [Unity, Addressables, 动态资源管理]
categories: [Tech, Unity]
toc: true
comments: true
---
# Addressables可寻址资源管理系统
## 1. 传统资源管理方式`Resouces`
传统的资源加载方式一般通过将资源放置在一个特殊的`Resource`文件夹中，使用如下方式使用：

```C#
// 加载资源
GameObject prefab = Resources.Load<GameObject>("Characters/MyCharacter");
// 卸载资源
Resources.UnloadAsset(prefab);
```

使用`Resources`方式管理资源有几点缺点：
- 内存管理十分不灵活，容易造成内存泄漏或资源丢失。
- 无法热更新。所有资源在构建时会被打包一个特殊不变归档文件中，无法在资源发布后更改。
- `Resources`文件增大会直接影响游戏启动速度。
- 一切资源，只要存在于`Resources`文件夹中，都会被包含在最终构建中。
- 
## 2. 什么是Addressables

`Addressables Asset System`是Unity开发的资源管理插件，主要用于运行时游戏资源的加载和卸载。资源（例如预制件）标记为“可寻址”后，就会生成一个可从任何地方调用的地址。无论资源位于何处（本地还是远程），系统都会找到资源及其依赖项，然后将其返回。
`Addressable`相比与`Resources`的优势在于：
- 灵活的打包模式：
	- Packed Together：组内所有资源打包为一个资源包。
	- Separately：每个资源单独打包。
- 热更新：可以只更新修改过的资源包，避免对所有构建内容一起修改。
- 优化异步加载。
- 
## 3. 导入Addressables

`Addressables`插件可以在`Package Manager`中找到。（`Addressables.CN`增加了对`AssetBundle`的加密，无法通过`AssetStudio`工具逆向）。


## 4. 资源设置

### 4.1 统一资源管理界面`Addressables Groups`
`Addressables Groups`的概念只存在于`Editor`中。也就是说无法在代码中根据`Groups`获取资源。但是可以通过使用给一个组的所有资源添加`Label`管理。
通过下列目录结构创建`Addressables Groups`。

```
Window
- Asset Management
  - Addressables
    - Groups
```

### 4.2 创建`Addressables Settings`
1. 在`Addressables Groups`中点击`Create Addressables Settings`
2. 在`Inspector`中将任意资源设置为`Addressable`

工程目录中会出现名为`AddressableAssetsData`的文件夹，并且`Addressables Groups`中默认创建组`Default Local Group (Default)`。
`Addressables`通过对每个`Group`打包为一个或多个`AssetBundle`。

![Addressable Asset Settings](/img/Addressables/AddressableAssetSettings.png)

- `Profile`：选择激活的`Profile`。
- `Diagnostics`：
	- `Send Profiler Events`：向`Unity Profiler`发送详细的资源加载/卸载事件。
	- `Log Runtime Exception`：错误日志开关，控制当资源加载/卸载失败时，是否将异常信息记录到`Unity Console`，发布时最好禁用。
- `Catalog`:
	- `Player Versino Override`：控制远程`Catalog`命名，决定客户端识别不同版本资源目录。
	- `Compression Local Catalog`：将 `Catalog.json`打包成`AssetBundle`，减小文件体积
	- `Build Remote Catalog`：是否生成用于远程加载的`Catalog`文件，使已安装的应用能够检测并下载更新的资源。
	- `Only update catalogs manually`：是否允许自动从远程服务器下载并更新`Catalog`，禁用后强制所有`Catalog`更新必须通过代码显式调用。
- `Update a Previous Build`：
	- `Check for Update issues`：手动触发内容更新限制检查工具，分析当前项目中哪些资源被修改、哪些资源需要移动到新`Bundle`或重建。
	- `Content State Build Path`：指定 `Content Update Build`工具用于跟踪资源变更的状态文件的存储路径。
	- `Path Preview`：实时显示`Build Path`和`Load Path`的实际解析路径，帮助验证配置是否正确。
- `Downloads`：
	- `Custom certificate handler`：自定义 `SSL/TLS` 证书验证逻辑。
	- `Max Concurrent Web Requests`：限制同时活跃的`UnityWebRequest`实例数量
	- `Catalog Download Timeout`：设置下载远程`Catalog`文件的最大等待时间，一般小于`Bundle Timeout`。
- -`Build`：
	- `Build Addressables On Player Build`：
		- `Use global Settings`
		- `Build Addressables content on Player Build`：构建时自动执行`Addressables`构建。
		- `Do not Build Addressables content on Player Build`：跳过`Addressables`构建，使用最后一次手动构建的结果。
	- `Ignore Invalid/Unsupported Files in Build`： 控制`Addressables`构建过程中遇到无效或不支持的文件时是忽略并继续构建还是报错终止。
	- `Unique Bundle IDs`：开启后`Bundle Naming Mode`应该使用携带`Hash`的模式。
	- `Contiguous Bundles`：将资源在物理上连续存储，将随机 I/O 转换为顺序 I/O，对 Requested Asset 模式有巨大性能提升，默认开启。
	- `Non-Recursive Dependency Calculation`：控制`Bundle`依赖关系的计算方式，决定是深度遍历所有依赖还是仅处理直接依赖。默认禁用。
	- `Shader Bundle Naming Prefix`：控制着色器`AssetBundle`命名的设置。
	- `MonoScript Bundle Naming Prefix`：控制`MonoScript AssetBundle`命名的设置。`MonoScript`是`Unity`引擎内部用于表示一个脚本文件本身的类对象，用于创建`MonoBehavior`实例。
	- `Strip Unity Version from AssetBundle`：开启后，序列化文件头中的版本号会被写成 `0.0.0`，使得不同`Unity`小版本间、内容完全相同的`Bundle`得到一致的`CRC`，从而避免玩家因版本升级而被迫全量重新下载。
	- `Disable Visible Sub Asset Representations`：阻止子资源被单独打包。
- `Build and Play Mode Scripts`：构建以及`Play Mode Scripts`配置文件。
- `Asset Group Templates`：包含所有`Asset Group Templates`。
- `initialzation Objects`：配置项目的初始化对象。初始化对象是实现了`IObjectInitializationDataProvider`接口的`ScriptableObject`类。可以在运行时创建这些对象以向`Addressables`初始化过程传递数据。
- `Cloud Content Delivery`：
	- `Enable CCD Features`：开启`Unity`云内容分发（`Cloud Content Delivery, CCD`） 集成开关。打开后，会在构建时自动把资源包上传到在 `Unity Dashboard`创建的`CCD`存储桶（`bucket`），并生成远程目录，而不再需要手动用`CLI` 或网页传包。

### 4.3 将资源设置为`Addressables`
两种方式：
- 在资源的`Inspector`窗口设置为`Addressable`。
- 将资源直接拖入`Addressables Groups`窗口的组中。

注意：`Addressables`对于代码无效。

### 4.4 对组的操作
`Group`的本质是`AddressableAssetsData`下的`ScriptObject`配置文件，使用`YAML`格式描述信息。
-  在`Create`选项中创建组：
	- `Packed Assets`：打包资源分组（自带默认打包加载相关设置信息）。
	- `Blank(no schema)`：空白（打包加载相关设置信息需哎哟自己关联）。
- `Remove Group`：移除组，将组中所有资源变为不可寻址资源。
- `Simplify Addressable Names`：简化可寻址名称。
- `Set as Default`：设置为默认组，当直接勾选资源的`Addressable`时会添加进默认组。
- `Rename`：重命名。
- `Create New Group`：创建新组

下面将介绍`Addressables Groups`对应的配置`ScriptObject`。

#### `Content Packing & Loading`

![Content Packing & Loading](/img/Addressables/ContentPacking&Loading.png)

`Build & Load Paths`是一组配置路径对，定义了`Addressables`构建和加载路径。
可以选择`Local`、`Remote`和`Custom`三种模式。

#### `Advanced Options`

![Advanced Options](/img/Addressables/AdvancedOptions.png)
- `Asset Bundle Compresson`：指示了该包的压缩方式。
- `Include in Build`：决定该组资源是否打包进最终游戏构建，如果设置为`false`需要远程打包，适用于游戏DLC热更新。
- `Force Unique Provider`：该组会创建独立`Provider`实例，使用独特资源加载逻辑。
- `Use Asset Bundle Cache`：缓存远程资源，加速下一次加载。
- `Asset Bundle CRC`：加载前验证完整性。
- `Use UnityWebRequest for Local Asset Bundles`：是否使用 `UnityWebRequest` 替代默认的 `AssetBundle.LoadFromFileAsync` 来加载本地`Bundle`。
- `Request Timeout`：控制`UnityWebRequest`最大等待时间。
- `Use Http Chunked Transfer`：是否使用 HTTP /1.1分块传输编码（`Chunked Transfer Encoding`）下载远程 `AssetBundle`。
- `Http Redirect Limit`：控制 `UnityWebRequest` 在下载远程 `AssetBundle` 或 `Catalog` 时允许跟随 HTTP 重定向的最大次数。
- `Retry Count`：控制当远程资源加载失败时（超时、网络错误、HTTP 错误等），自动重新尝试的最大次数。
- `Include Addresses in Catalog`：是否在 `Addressables Catalog`文件中包含资源的地址字符串（`Address`），用于减小 Catalog 文件体积，禁用后无法通过地址字符串加载。
- `Include GUIDs in Catalog`：禁用后无法通过`AssetReference`加载。
- `Include Labels in Catalog`：禁用后无法通过`Label`加载。
- `Internal Asset Naming Mode`：决定资源在 `AssetBundle` 内部 和 `Catalog` 中如何被标识和命名。
- `Internal Bundle Id Mode`：决定 `AssetBundle` 在内部如何被唯一标识。
- `Cache Clear Behavior`：决定已安装的应用程序何时从设备缓存中清除 `AssetBundle`文件。
- `Bundle Mode`：控制组打包模式，可以组内资源打包为一个`AssetBundle`,亦可以每个资源单独打包为`AssetBundle`，还可以按照`Label`打包为多个`AssetBundle`。
- `Bundle Naming Mode`：控制`AssetBundle`文件名的命名规则。`Filename`模式仅限开发使用，因为容易违背`Unique Bundle IDs`功能。
- `Asset Load Mode`：决定加载资源时是按需加载还是全量加载。
- `Asset Provider`：决定了资源加载到内存时使用的底层提供程序`Provider`类型。使用`Bundled Asset Provider`后底层会使用`AssetBundle.LoadFromFile`加载整个`Bundle`（适合旧项目迁移）。
- `Asset Bundle Provider`：底层加载器，决定如何从磁盘或网络加载 `AssetBundle` 文件本身（`.bundle` 文件）。同上，一样作为旧项目迁移手段时使用`Bundle Asset Provider`模式。

#### Content Update Restriction schema reference

![Content Update Restriction schema reference](/img/Addressables/ContentUpdateRestrictionSchemaReference.png)

控制内容热更新构建时资源的处理策略，决定修改过的资源是移动到新`Bundle`还是原地重建整个`Bundle`。
- `Prevent Updates = Enabled`：不移动资源而是重建整个`Bundle`。
- `Prevent Updates = Disabled`：修改资源时，`Check for Content Update Restrictions`工具会将修改的资源移动到新组`GroupName_Update`，配置为远程加载。

注意：设置该项，并且完整构建`Build`后，应该保证该项不变，否则无法生成正确变更。

### 4.5 对`Addressables`资源的操作
- `Move Addressables to Group`：将资源移入另一个组。
- `Move Addressables to New Group`：新建组并移入资源。
- `Remove Addressables`：移除资源，将其变为不可寻址资源（取消`Addressable`）。
- `Simplify Addressables Names`：删除路径和拓展，简化名称。
- `Copy Address to Clipboard`：复制地址名称至剪切板。
- `Change Address`：为资源改名。
- `Create New Group`：创建新组。
- 可以为每一个`Addressable`资源添加标签`Label`。

`Addressables`依赖资源路径寻找资源，因此资源可以同名，但是路径不可相同。

### 4.6 `Label`标签

可以为每一个`Asset`设置一个或多个`Label`。
在`Addressables`中，`Label`有以下功能：
- 代码中通过`Label`寻找`Asset`。
- 根据`Label`将一个`Group`构建为多个`AssetBundle`。
- 在`Group Window`中的过滤器中根据`Label`迅速查找`Asset`。 

### 4.7 配置概述相关`Profile`
`Profile`定义了一系列供`Addressables build scripts`使用的变量，可以为项目自定义多个`profile`用于开发、测试、发布阶段。

![Profile Window](/img/Addressables/ProfileWindow.png)

`Manage Profile`：使用`Addressables Profiles`窗口管理配置文件（可以配置打包目标、本地远程打包加载路径等信息）。
- `BuildTarget`：`[UnityEditor.EditorUserBuildSettings.activeBuildTarget]`，设置了构建目标，一般为`Android`或`StandaloneWindows64`。
- `Local.BuildPath`：`[UnityEditor.AddressableAssets.Addressables.BuildPath]/[BuildTarget]`，定义了资源构建本地路径。
- `Local.LoadPath`：`｛UnityEngine.AddressableAssets.Addressables.RuntimePath｝/[BuildTarget]`，定义了资源加载路径，默认情况下位于`StreamingAssets`中。
- `Remote.BuildPath`：`ServerData/[BuildTarget]`，定义了远程分发时要将资源文件构建的目标路径。
- `Remote.LoadPath`：`http://localhost/[BuildTarget]`，定义了从哪个`URL`上下载资源。

这是默认`path values`，`Unity`的`build system`期望将`AssetBundles`和其他文件放在默认位置，如果更改了本地路径，需要在构建前将文件从构建路径复制到加载路径并且加载路径应该始终位于`StreamingAsset`文件夹内。

除了这些变量外，也可以创建自定义的变量和路径对。
- `[]`表示构建期间内容，可以是其他配置条目`BuildTarget`或者代码变量`[UnityEditor.EditorUserBuildSettings.activeBuildTarget]`。
- `{}`表示运行期间内容，可以是运行时类的代码变量，如`{UnityEngine.AddressableAssets.Addressables.RuntimePath}`。

一般来说，设置为`{MyNamespace.MyClass.MyURL}/content/[BuildTarget]`的加载路径对应构建期间`catalog`的加载路径为`{MyNamespace.MyClass.MyURL}/content/Android/*.bundle`，`[BuildTarget]`为`Android`。运行期间，最终加载路径为`http://xxx.com/content/Android/*。bundle`。

在开发阶段，`LocalBundles`和`RemoteBundles`都在本地路径存储，可以都使用`Built-In`模式。
后期将`RemoteBundles`移动到服务器端，可以更改`Remote.LoadPath`为服务器地址。

在构建期间，`Addressables`系统仅将数据从`Addressables.BuildPath`复制到`StreamingAssets`文件夹。它不处理通过`LocalBuildPath`或`LocalLoadPath`变量指定的任意路径。如果你将数据构建到默认位置以外的位置，或者从默认位置以外的位置加载数据，你必须手动复制数据。
类似地，你必须手动将远程`AssetBundles`及其相关的目录和哈希文件上传到你的服务器，以便它们可以通过`RemoteLoadPath`定义的URL被访问。

### 4.8 工具相关
```
Tools
- Inspect System Settings 选中配置文件
- Check for Content Update Restrictions 检查内容更新限制
- Window Addressables 窗口相关
    Profiles
    Labels
    Analyze
    Hosting Services
    Event Viewer
-Groups view 分组视图
	Show Sprite and Subobject Addresses 显示可寻址对象的Sprite和子对象，如图集资源等。
    Group Hierarchy with Dashes 组层次结构
```
### 4.9 `Play Mode Script`
```
Play Mode Script
- Use Asset Database (fast)
- Simulate Groups (Advanced)
- Use Existing Build (requires built groups)
```
`play Mode Script`决定了在编辑器播放模式下运行游戏时，可寻址系统如何访问资源。
- `Use Asset Database (fast)`：使用资源数据库，一般在开发阶段使用，不必构建打包；实际开发基本不使用。
- `Simulate Groups (Advanced)`：测试阶段使用，通过`ResourceManager`从资产数据库加载资产，引入时间延迟，模拟远程资产绑定的下载速度和本地绑定的文件加载速度，实际未绑定资产，以脚本方式加载资源。实际开发中常常使用该模式。
- `Use Existing Build (requires built groups)`：最终发布测试阶段使用。使用现有构建，从创建的捆绑包加载资产，使用之前需要用生成脚本打包资源。远程内容需要托管在用于神抽狗内容的配置文件的`RemoteLoadPath`上。
### 4.10 `Build commands`
```
Build
- New Build
   Default Buid Script 使用默认打包脚本
- Update a Previous Build 更新过去版本
- Clean Build 清空之前的构建资源
  - 
```

## 5. 构建`Build`
### `Unity Player Build`和`Addressable Build`
- `Player Build`：将项目构建为安装包。
- `Addressables Build`：打包为`AssetBundle`。

`Unity`在构建`Player Build`时：
- 如果启用`Build Addressables on Player Build`，`Unity`则会在`Player Build`时先自动执行一次`Addressable Build`，将结果复制到`StreamingAssets`文件夹，然后在打包。
- 如果启用`Do not Build Addressables on Player Build`，`Unity`需要手动进行`Addressables Build`，`Player Build`只会将最后一次构建的`AssetBundle`存入`StreamingAssets`。

以上只是`Local Content`的构建，对于`Remote content`，需要设置好`RemoteLoadPath`指定的`URL`才能在应用安装后下载。

`Build commands`中使用了`Build Script`配置具体的`Build`设置。

### 资源依赖

#### 引用子物件`Reference sub-objects`
如果`AssetReference`指向`Addressable Asset`，那么其子对象也会被构建为同一个包内。
如果`AssetReference`指向拥有子对象的父对象如`Scene GameObject ScriptObject`（其子对象不是`Addressable`），则只将子对象作为隐式依赖项`implicit dependency`构建到`Bundle`中。

#### 捆绑包依赖`AssetBundle dependencies`
当`AssetBundle A`引用另一个`AssetBundle B`中的资源，如`Material`，就叫`AssetBundle dependencies`。
- 在传统`AssetBundle`工作流中，此时`Unity`会生成`AssetBundleManifest`，需要先根据它加载依赖项。
- `Addressables`会使用`Catalog.json`记录依赖关系，可以直接使用`API`而不关心依赖。但是，隐式依赖项无法检测。此外，循环依赖可以被自动检测。

#### 引用多个隐式资源

![Reference Multiple Implicit Asset](/img/Addressables/ReferenceMultipleImplicitAsset.png)

如果多个可寻址资源引用同一个隐式资源，那么每个包含引用的可寻址资源的 bundle 中都会包含该隐式资源的副本。
为防止这种重复，可以将隐式资源设为`Addressable`资源，并将其包含在现有的`AssetBundle`中，或添加到不同的`AssetBundle`中。当加载引用该资源的`Addressable`时，该资源所在的`AssetBundle`会被加载。如果`Addressable`被打包到与被引用资源不同的`AssetBundle`中，那么包含被引用资源的`AssetBundle`就是一个`AssetBundle`依赖。

非`Addressable`对象和`Addressable`对象引用同一份`Addressable`资源时，他们之间的依赖会导致`Unity`复制资产；同样，不同`Bundle`中的`Addressable`对象引用同一个`Addressable`对象时也会导致相同的问题。
有一些策略可以减轻该问题：
- 尽量将`Resources`文件夹中的资源设置为`Addressable`。
- 尽量保证`Shared Addressable Asset`位于单独`Bundle`。

### 完整构建`New Build`
#### 构建和加载路径
本地构建路径默认为`Addressables.BuildPath`提供的路径，该路径位于`Unity`项目的`Library`文件夹中。`Addressables`会根据你当前的平台构建目标设置在本地构建路径中追加一个文件夹。当你为多个平台构建时，构建会将每个平台的工件放置在不同的子文件夹中。

同样，本地加载路径默认为`Addressables.RuntimePath`提供的路径，该路径解析为 `StreamingAssets`文件夹。`Addressables`再次将平台构建目标添加到路径中。

当你将本地资源包构建到默认构建路径时，构建代码在构建`Player`时会暂时将资源从构建路径复制到`StreamingAssets`文件夹，并在构建完成后将其删除。

`Addressables`将默认的远程构建路径设置为任意选择的一个文件夹名称，`ServerData` ，该文件夹创建在你的项目文件夹下。构建会将当前平台目标作为子文件夹添加到路径中，以分离不同平台的唯一资源。

默认的远程加载路径是 `http://localhost/` 加上当前配置文件`BuildTarget`变量。你必须将此路径更改为你计划加载`Addressable`资源的基`URL`。

#### 设置远程内容构建
1. `Window > Asset Management > Addressables > Settings`。
2. 在`Catalog`下，开启`Build Remote Catalog`。
3. 对于每个组，将`Build Path`和`Load Path`设置为`RemoteBuildPath`和`RemoteLoadPath`。
4. `Window > Asset Management > Addressables > Profiles`
5. 将`RemoteLoadPath`设置为计划托管远程内容的`URL`。

#### 执行构建
在`Group Window`选择`Build > NewBuild > Default Build Script`。

### 更新构建`Update Build`
`Update Build`会对先前发布的构建进行差异更新，需要上次完整构建留下的`addressables_content_state.bin`文件，这个文件默认位置在`Assets/AddressableAssetsData/TargetPlatform`。

以下行为会导致`MonoScript`更新，从而导致需要完整构建`Addressables`：
- 将脚本移动到不同程序集。
- 更改程序集定义文件名称。
- 添加/更改类命名空间。
- 更改类名。

### 构建脚本`Build Scripting`
叫恩启动构建需要调用方法`AddressableAssetSettings.BuildPlayerContent`方法，该方法会考虑一下信息：
- `AddressableAssetSettingsDefaultObject`
- `ActivePlayerDataBuilder`
- `addressables_content_state.bin`
- 
#### 从脚本构建

加载自定义设置对象：
```C#
static void getSettingsObject(string settingsAsset)
{
	// Use Default Settings
	settings = AddressableAssetSettingsDefaultObject.Settings;
	
	// Use Custom Settings
	settings = AssetDatabase.LoadAssetAtPath<ScriptableObject>(settingAsset) 
		as AddressableAssetSettings;
		
	if(settings == null) Debug.Log($"{settingsAsset} couldn't be found or isn't a settings object.");
}
```
设置配置文件：`settings`包含`profile`列表，根据名称获取指定`profileId`并将该`profile`设置为`active`活动配置文件：
```C#
static void setProfile(string profile)
{
	string profileId = settings.profileSettings.getProfileId(profile);
	if(String.IsNullOrEmpty(profileId)) Debug.LogWarning($"Couldn't find a profile named {profile}");
	else
		settings.activeProfiledId = profileId;
}
```

设置构建脚本：`BuildContent`会根据`ActivePlayerDataBuilder`设置启动构建，构建脚本必须是实现了`IDataBuilder`的`ScriptableObject`，并且必须将其添加到`AddressableAssetSettings` 实例中的`DataBuilders`列表中。
```C#
static void setBuilder(IDataBuilder builder)
{
	int index= settings.DataBuilders.IndexOf((ScriptableObject)builder);
	if(index > 0)
		settings.ActivePlayerDataBuilderIndex = index;
	else
		Debug.LogWarining($"{builder} must be added to the " +
                         $"DataBuilders list before it can be made " +
                         $"active. Using last run builder instead.");
}
```

启动构建
```C#
static bool buildAddressableContent()
{
	AddressableAssetSettings.BuildPlayerContent(out AddressablesPlayerBuildResult result);
	bool success = string.IsNullOrEmpty(result.Error);
	if(!success) Debug.LogError("Addressables build error" + result.Error);
	return success;
}
```

在编辑器中的示例脚本，实现了2个命令，分别完成构建`Addressable`和`Player`。
```C#
#if UNITY_EDITOR
    using UnityEditor;
    using UnityEditor.AddressableAssets.Build;
    using UnityEditor.AddressableAssets.Settings;
    using System;
    using UnityEngine;

    internal class BuildLauncher
    {
        public static string build_script
            = "Assets/AddressableAssetsData/DataBuilders/BuildScriptPackedMode.asset";

        public static string settings_asset
            = "Assets/AddressableAssetsData/AddressableAssetSettings.asset";

        public static string profile_name = "Default";
        private static AddressableAssetSettings settings;

        static void getSettingsObject(string settingsAsset)
        {
            // This step is optional, you can also use the default settings:
            //settings = AddressableAssetSettingsDefaultObject.Settings;

            settings
                = AssetDatabase.LoadAssetAtPath<ScriptableObject>(settingsAsset)
                    as AddressableAssetSettings;

            if (settings == null)
                Debug.LogError($"{settingsAsset} couldn't be found or isn't " +
                               $"a settings object.");
        }


        static void setProfile(string profile)
        {
            string profileId = settings.profileSettings.GetProfileId(profile);
            if (String.IsNullOrEmpty(profileId))
                Debug.LogWarning($"Couldn't find a profile named, {profile}, " +
                                 $"using current profile instead.");
            else
                settings.activeProfileId = profileId;
        }


        static void setBuilder(IDataBuilder builder)
        {
            int index = settings.DataBuilders.IndexOf((ScriptableObject)builder);

            if (index > 0)
                settings.ActivePlayerDataBuilderIndex = index;
            else
                Debug.LogWarning($"{builder} must be added to the " +
                                 $"DataBuilders list before it can be made " +
                                 $"active. Using last run builder instead.");
        }
        
        static bool buildAddressableContent()
        {
            AddressableAssetSettings
                .BuildPlayerContent(out AddressablesPlayerBuildResult result);
            bool success = string.IsNullOrEmpty(result.Error);

            if (!success)
            {
                Debug.LogError("Addressables build error encountered: " + result.Error);
            }

            return success;
        }
        
        [MenuItem("Window/Asset Management/Addressables/Build Addressables only")]
        public static bool BuildAddressables()
        {
            getSettingsObject(settings_asset);
            setProfile(profile_name);
            IDataBuilder builderScript
                = AssetDatabase.LoadAssetAtPath<ScriptableObject>(build_script) as IDataBuilder;

            if (builderScript == null)
            {
                Debug.LogError(build_script + " couldn't be found or isn't a build script.");
                return false;
            }

            setBuilder(builderScript);

            return buildAddressableContent();
        }

        [MenuItem("Window/Asset Management/Addressables/Build Addressables and Player")]
        public static void BuildAddressablesAndPlayer()
        {
            bool contentBuildSucceeded = BuildAddressables();

            if (contentBuildSucceeded)
            {
                var options = new BuildPlayerOptions();
                BuildPlayerOptions playerSettings
                    = BuildPlayerWindow.DefaultBuildMethods.GetBuildPlayerOptions(options);

                BuildPipeline.BuildPlayer(playerSettings);
            }
        }
    }
#endif


```

如果脚本化构建过程涉及`Domain reloads`，则不应在编辑器中运行，而是应该使用`Unity`的命令行参数。
在编辑器中以交互方式运行触发域重载的脚本时，例如使用菜单命令，编辑器脚本会在域重载发生之前完成执行。因此，如果立即开始`Addressables`构建，代码和导入的资源仍然处于原始状态。必须等待域重载完成后再开始内容构建。

关于自定义构建脚本内容不做补充。

### 部分资源如何被构建
#### 图集`Sprite Atlases`

对于位于不同组的`Addressable Texture`，会被分别打包。

将其放入非`Addrsssable`的`Sprite Atlas`，其中一个`AssetBundle`会包含所有`Texture`，其余`AssetBundle`只包含`Metadata`并且将之作为依赖。

将其放入`Addressable`的`Sprite Atlas`，`Altas`所在`AssetBundle`会包含所有`Texture`，其余`AssetBundle`只包含`Metadata`。

在上述前提基础上，当引用`Texture`的`Prefab`被设置为`Addressable`，而`Atlas`设置为`Non-Addressable`，`Prefab`所在`AssetBundle`会包含`Atals`的所有`Texture`，造成严重重复。

应该将`Atlas`设置为`Addressable`，此时，`Prefab`所在`AssetBundle`中仅包含引用，依赖自`Atlas`所在`AssetBundle`。

#### `Shaders`
默认情况下，`Unity`会移除未在任何场景中使用的着色器变体。这可能会排除仅在`AssetBundles`中使用的变体。为确保某些变体不会被移除，请在图形设置中的着色器移除属性中包含它们。

例如，如果您有使用与光照贴图相关的着色器（如`Mixed Lights`）的 Addressable 资产，请前往`Edit > Project Settings > Graphics > Shader Stripping`，并将光照贴图模式属性设置为自定义。

质量设置也会影响`AssetBundles`中使用的着色器变体。

## 6. 分发
## 7. 运行时使用`Addressables`
## 8. 加载资源
### `Addressables`加载流程
1. 根据给出的`key`寻找资源
2. 创建资源依赖列表
3. 下载远程`AssetBundle`
4. 将`AssetBundle`载入内存
5. 将加载的`object`赋给`Result`
6. 更新`Status`，执行`Completed`事件回调

这里提到的`key`是`Addressables`寻找资源的依据，可以是下面几种类型：
- `Address`：资源的实际地址字符串，是资源的唯一标识。
- `Label`：标签字符串，可以指向多个资源。
- `AssetReference object`：`AssetReference`的实例。
- `IResourceLocation instance`：包含加载资源及其依赖所需信息的中间`object`。

加载成功：
- `Status`置为`Succeeded`。
- `Result`被赋值。

加载失败：
- `Status`置为`Failed`。
- `OperationException`获取错误信息，可以使用`ResourceManager.ExceptionHandler`处理信息，或者通过勾选`Log Runtime Exceptions`打印信息至`Unity Conole`。

### 异步加载`Asynchronous loading`

异步加载方法不会直接返回资源本身，而是返回一个`AsyncOperationHandle`，可以使用该对象访问加载的资源。

除非主动释放这些资源，它们都会一直存在与内存当中，所以使用`Addressables.Release`释放空间，一些方法会自动释放如`UnloadSceneAsync`。

加载失败时，`Handle`仍然需要被释放，不过加载多个资源的方法如`LoadAssetsAsync`允许在加载任意资源失败时保留已经加载的资源或者释放所有的资源。

第一次加载资源最少需要一帧时间才能完成操作。之后再次加载时，不同的异步操作需要不同时间：
- `Coroutine`：不论内容是否加载完成，执行前至少延迟一帧。
- `Completed`事件回调：如果内容尚未加载，至少延迟一帧，否则同帧调用回调函数。
- `Await AsyncOperationHandle.Task`：如果内容尚未加载，至少延迟一帧，否则同帧执行。

这三种操作也是`Addressables`推荐使用的异步加载方式。

#### `Coroutine`
```C#
using System.Collections;
using UnityEngine;
using UnityEngine.AddressableAssets;
using UnityEngine.ResourceManagement.AsyncOperations;

internal class LoadWithIEnumerator : MonoBehaviour
{
    public string address;
    AsyncOperationHandle<GameObject> opHandle;

    public IEnumerator Start()
    {
        opHandle = Addressables.LoadAssetAsync<GameObject>(address);

        // yielding when already done still waits until the next frame
        // so don't yield if done.
        if (!opHandle.IsDone)
            yield return opHandle;

        if (opHandle.Status == AsyncOperationStatus.Succeeded)
        {
            Instantiate(opHandle.Result, transform);
        }
        else
        {
            Addressables.Release(opHandle);
        }
    }

    void OnDestroy()
    {
        Addressables.Release(opHandle);
    }
}
```
如果希望取消`LoadAssetAsync`，很遗憾这样的操作是不被允许的。但是可以在加载途中`Release`以减少`Handle`引用计数，在完成加载后就会自动释放资源。

####  基于事件的操作`Event-based operation handling`
```C#
using UnityEngine;
using UnityEngine.AddressableAssets;
using UnityEngine.ResourceManagement.AsyncOperations;

internal class LoadWithEvent : MonoBehaviour
{
    public string address;
    AsyncOperationHandle<GameObject> opHandle;

    void Start()
    {
        // Create operation
        opHandle = Addressables.LoadAssetAsync<GameObject>(address);
        // Add event handler
        opHandle.Completed += Operation_Completed;
    }

    private void Operation_Completed(AsyncOperationHandle<GameObject> obj)
    {
        if (obj.Status == AsyncOperationStatus.Succeeded)
        {
            Instantiate(obj.Result, transform);
        }
        else
        {
            Addressables.Release(obj);
        }
    }

    void OnDestroy()
    {
        Addressables.Release(opHandle);
    }
}
```
#### 基于`Task`的操作`Task-based operation handling`
`AsyncOperationHandle`提供了`Task`对象，因此可以通过`C#`的`await`和`async`关键字调用异步方法。

注意：`WebGL`平台无法使用`AsyncOperationHandle.Task`。

```C#
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.AddressableAssets;
using UnityEngine.ResourceManagement.AsyncOperations;

internal class LoadWithTask : MonoBehaviour
{
    // Label or address strings to load
    public List<string> keys = new List<string>() {"characters", "animals"};

    // Operation handle used to load and release assets
    AsyncOperationHandle<IList<GameObject>> loadHandle;

    public async void Start()
    {
        loadHandle = Addressables.LoadAssetsAsync<GameObject>(
            keys, // Either a single key or a List of keys 
            addressable =>
            {
                // Called for every loaded asset
                Debug.Log(addressable.name);
            }, Addressables.MergeMode.Union, // How to combine multiple labels 
            false); // Whether to fail if any asset fails to load

        // Wait for the operation to finish in the background
        await loadHandle.Task;

        // Instantiate the results
        float x = 0, z = 0;
        foreach (var addressable in loadHandle.Result)
        {
            if (addressable != null)
            {
                Instantiate<GameObject>(addressable,
                    new Vector3(x++ * 2.0f, 0, z * 2.0f),
                    Quaternion.identity,
                    transform); // make child of this object

                if (x > 9)
                {
                    x = 0;
                    z++;
                }
            }
        }
    }

    private void OnDestroy()
    {
        Addressables.Release(loadHandle);
        // Release all the loaded assets associated with loadHandle
        // Note that if you do not make loaded addressables a child of this object,
        // then you will need to devise another way of releasing the handle when
        // all the individual addressables are destroyed.
    }
}
```
`AsyncOperationHandle.Task`可以使用`C#`的`Task`类中的方法如`WhenAll`控制流程。
```C#
// Load the Prefabs
var prefabOpHandle = Addressables.LoadAssetsAsync<GameObject>(
    keys, null, Addressables.MergeMode.Union, false);

// Load a Scene additively
var sceneOpHandle
    = Addressables.LoadSceneAsync(nextScene,
        UnityEngine.SceneManagement.LoadSceneMode.Additive);

await System.Threading.Tasks.Task.WhenAll(prefabOpHandle.Task, sceneOpHandle.Task);
```

### 同步加载`Synchronous loading`
#### `WaitForCompletion`

要在`Addressables`中使用同步加载，需要调用`WaitForCompletion`方法，这样会阻塞主线程直到操作完成。

```C#
using UnityEngine;
using UnityEngine.AddressableAssets;
using UnityEngine.ResourceManagement.AsyncOperations;

internal class LoadSynchronously : MonoBehaviour
{
    public string address;
    AsyncOperationHandle<GameObject> opHandle;

    void Start()
    {
        opHandle = Addressables.LoadAssetAsync<GameObject>(address);
        opHandle.WaitForCompletion(); // Returns when operation is complete

        if (opHandle.Status == AsyncOperationStatus.Succeeded)
        {
            Instantiate(opHandle.Result, transform);
        }
        else
        {
            Addressables.Release(opHandle);
        }
    }

    void OnDestroy()
    {
        Addressables.Release(opHandle);
    }
}
```

`WaitForCompletion`会返回异步加载的`Result`，如果加载失败则会返回`default(TObject)`。对于`autoReleaseHandle`设置为`true`和`Addressables.InitializeAsync`这种会自动释放句柄的操作，`WaitForComletion`同样会返回`default(TObject)`。

注意：`WaitForCompletion`可能造成一些性能问题，尤其是资源未缓存在本地的时候，不要在需要下载远程资源时使用。

另外，`WebGL`不支持`WaitForCompletion`。

#### 场景同步加载限制

在`Unity`中，场景无法同步加载，也无法同步卸载。

简单来说，加载场景需要经历2个阶段，首先序列化数据，然后注册到`SceneManager`、`Physics`、`Rendering`等引擎系统，`WaitForCompletion`只完成一阶段内容，而没有激活场景。

另外，由于`WaitForCompletion`会阻碍主线程，因此如果连续加载`Single`场景（场景覆盖时会销毁上一个场景，调用`UnloadUnusedAsstes`）并且使用该方法会导致死锁问题。
```C#
var handle1 = Addressables.LoadSceneAsync("Scene1", LoadSceneMode.Single);
var scene1 = handle1.WaitForCompletion(); // 主线程被阻塞

var handle2 = Addressables.LoadSceneAsync("Scene2", LoadSceneMode.Single);
var scene2 = handle2.WaitForCompletion(); // 再次阻塞主线程

```

### 自定义操作`Custom Operations`

如果要自定义操作，需要继承自`AsyncOperationBase`类并且重写其虚方法。
#### 注册
如果要使用自定义操作，还需要将它注册给`ResourceManager.StartOperation`方法。
#### 执行
`ResourceManager`会调用`AsyncOperationBase.Excute`方法。
#### 完成
当自定义操作完成时，应该调用`AsyncOperationBase.Complete`，这个方法告诉`ResourceManager`操作已经结束，应该触发自定义操作实例的`AsyncOperationHandle.Completed`事件。
#### 销毁
当`AsyncOperationBase.ReferenceCount`归零时，`ResourceManager`会调用`AsyncOperationBase.Destroy`。`Addressables.Release`和`AsyncOperationBase.DecrementReferenceCount`都会减少引用数。
#### 类型转换
大多数`Addressables`方法返回通用`AsyncOperationHandle<T>`类型。也可以使用非通用类型`AsyncOperationHandle`，并在两种类型间转换。但是将`AsyncOperationHandle`转换为`Handle`信息未记录的类型是不被允许的。

```C#
// Load asset using typed handle:
AsyncOperationHandle<Texture2D> textureHandle = Addressables.LoadAssetAsync<Texture2D>("mytexture");

// Convert the AsyncOperationHandle<Texture2D> to an AsyncOperationHandle:
AsyncOperationHandle nonGenericHandle = textureHandle;

// Convert the AsyncOperationHandle to an AsyncOperationHandle<Texture2D>:
AsyncOperationHandle<Texture2D> textureHandle2 = nonGenericHandle.Convert<Texture2D>();

// This will throw and exception because Texture2D is required
AsyncOperationHandle<Texture> textureHandle3 = nonGenericHandle.Convert<Texture>();
// 因为 nonGenericHandle 已经记录了类型为 Texture2D
```

### 报告加载进度

`Addressables`提供了一些方法可以监控报告操作的进度：
- `GetDownloadStatus`：返回`DownloadStatus`结构体，包含已下载的`bytes`和未下载的`bytes`，`DownloadStatus.PercentComplete`提供了下载百分比。当资源在本地缓存，则`Percent`直接返回`100%`。
- `AsyncOperationHandle.PercentComplete`：返回任务完成百分比，每个子任务权重相同。

### 加载资产 `Load Assets`

#### 加载单个资产

使用`LoadAssetAsync`加载单个资产，可以使用`key`值筛选，如果有多个目标符合，则只会返回第一个资产。

```C#
using System.Collections;
using UnityEngine;
using UnityEngine.AddressableAssets;
using UnityEngine.ResourceManagement.AsyncOperations;

internal class LoadAddress : MonoBehaviour
{
    public string key;
    AsyncOperationHandle<GameObject> opHandle;

    public IEnumerator Start()
    {
        opHandle = Addressables.LoadAssetAsync<GameObject>(key);
        yield return opHandle;

        if (opHandle.Status == AsyncOperationStatus.Succeeded)
        {
            GameObject obj = opHandle.Result;
            Instantiate(obj, transform);
        }
    }

    void OnDestroy()
    {
        Addressables.Release(opHandle);
    }
}
```

#### 加载多个资产

使用`LoadAssetsAsync`方法加载多个资产，可以使用`key`列表匹配资产，有几种匹配方式：
- `Union`：符合任意`key`。
- `Intersection`：符合所有`key`。
- `UseFirst`：返回符合第一个`key`的第一个资产。

```C#
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.AddressableAssets;
using UnityEngine.ResourceManagement.AsyncOperations;

internal class LoadMultiple : MonoBehaviour
{
    // Label strings to load
    public List<string> keys = new List<string>() {"characters", "animals"};

    // Operation handle used to load and release assets
    AsyncOperationHandle<IList<GameObject>> loadHandle;

    // Load Addressables by Label
    public IEnumerator Start()
    {
        float x = 0, z = 0;
        loadHandle = Addressables.LoadAssetsAsync<GameObject>(
            keys,
            addressable =>
            {
                //Gets called for every loaded asset
                Instantiate<GameObject>(addressable,
                    new Vector3(x++ * 2.0f, 0, z * 2.0f),
                    Quaternion.identity,
                    transform);

                if (x > 9)
                {
                    x = 0;
                    z++;
                }
            }, Addressables.MergeMode.Union, // How to combine multiple labels 
            false); // Whether to fail and release if any asset fails to load

        yield return loadHandle;
    }

    private void OnDestroy()
    {
        Addressables.Release(loadHandle);
        // Release all the loaded assets associated with loadHandle
        // Note that if you do not make loaded addressables a child of this object,
        // then you will need to devise another way of releasing the handle when
        // all the individual addressables are destroyed.
    }
}
```
#### 加载多个资产失败

当加载失败时，根据`releaseDependenciesOnFailure`，如果设置为`true`则会释放已加载资源，设置为`false(default)`则会尝试加载所有资产，加载失败的资产以`null`代替。

#### 根据`Location`加载资产

使用`key`值查找资产时，`Addressables`首先查询资产`location`，并使用`IResourceLocation`实例下载需要的`AssetBundle`以及依赖项，加载时可以使用`LoadResourceLocationsAsync`。

`LoadResourceLocationsAsync`如果没有找到资产就会返回空列表。

此外，`LoadResourceLocationsAsync`的参数`type`可以指定查找的资产类型。


```C#
AsyncOperationHandle<IList<IResourceLocation>> handle
    = Addressables.LoadResourceLocationsAsync(
        new string[] {"knight", "villager"},
        typeof(Texture2D),
        Addressables.MergeMode.Union);

yield return handle;

//...

Addressables.Release(handle);
```

对于具有子对象的资产，每个子对象都会生成`IResourceLocation`实例，可以使用`type`选择性获取`location`。有以下几种写法：
- `LoadResourceLocationsAsync("myFBXObject", typeof(Mesh))`
- `LoadResourceLocationsAsync("myFBXObject[Mesh]")`

### 加载场景 `Load Scene`

`Addressables.LoadSceneAsync`方法可以加载场景，这个方法是对`SceneManager.LoadSceneAsync`的封装。

`Addressables.LoadSceneAsync`方法有几个参数：
- `LoadSceneMode`：控制加载场景是额外加载`Additive`还是覆盖加载`Single`。
- `LocalPhysicsMode`：控制额外加载场景的物理是否与当前场景物理产生交互还是单独隔离作用（该参数被包含在`LoadSceneParameters`中）。
- `LoadSceneParameters`：包含上面两个参数。
- `activateOnLoad`：控制场景激活流程，如果设置为`false`需要调用`SceneInstance.ActivateAsync()`方法激活。
- `priority`：`Unity`的加载线程/主线程调度器会把所有异步操作按 `priority`降序排队，数字大的先执行，默认为`100`。

```C#
using UnityEngine;
using UnityEngine.AddressableAssets;
using UnityEngine.ResourceManagement.AsyncOperations;
using UnityEngine.ResourceManagement.ResourceProviders;
using UnityEngine.SceneManagement;

internal class LoadSceneByAddress : MonoBehaviour
{
    public string key; // address string
    private AsyncOperationHandle<SceneInstance> loadHandle;

    IEnumerator Start()
    {
	    LoadSceneParameters param = new LoadSceneParameters(
		    LoadSceneMode.Additive 
		    , LocalPhysicsMode.None);
	    
        loadHandle = Addressables.LoadSceneAsync(
	        key, 
	        param,
		    activateOnLoad:false,
		    priority:100);
		yield return loadHandle;
		
		SceneInstance sceneInstance = loadHandle.Result;
		yield return sceneInstance.ActivateAsync();
    }

    void OnDestroy()
    {
        Addressables.UnloadSceneAsync(loadHandle);
    }
}
```
### 加载 AB 包 `Load AssetBundles

`Addressables`通过两种方式加载 **AB** 包，默认采用`AssetBundle.LoadFromFileAsync`加载本地资产，采用`UnityWebRequest.GetAssetBundle`加载远程资产。

`AssetBundle.LoadFromFileAsync`对不同压缩方式处理：

| 压缩方式 |        实际行为        |  成本   |
| :--: | :----------------: | :---: |
| LZ4  |    读取文件头，按需解压块     |  最低   |
| LZMA |     整包解压进内存再返回     | 首次加载慢 |
| 未压缩  | 直接把磁盘文件直接映射到内存地址空间 |  最快   |

`UnityWebRequest.GetAssetBundle`对不同压缩方式处理：

| 压缩方式 |       实际行为       |
| :--: | :--------------: |
| LZ4  |       原样缓存       |
| LZMA | 先下载解压，再本地压缩为 LZ4 |

### 卸载资产 `Unload Addressable assets`

`Addressables`会自动释放引用计数为 0 的资产。

卸载一个场景会导致其所在整个 AB 包都被卸载，也会将其中所有的`GameObject`卸载，即使将之移动到其他场景。

使用`LoadSceneMode.Single`会调用`UnloadUnusedAssets`方法，如果想要防止场景被卸载，需要保持对其的引用，可以对其句柄使用`ResourceManager.Acquire`。

对于`Addressables`的卸载方法，`Object.DontDestroyOnLoad`或者`HideFlags.DontUnloadUnusedAsset`等保存资产的方法都不起作用。

`Addressables`单独加载的资产无关场景，不会被释放，除非以`Single`模式加载新场景调用`Resource.UnloadUnusedAssts`或者`UnloadAsset`主动释放。
## 9. 诊断工具
## 参考
[Unity Addressables——bilibili](https://www.bilibili.com/video/BV19T4y127co?spm_id_from=333.788.player.switch&vd_source=0c4720590a80024b8323b2bb6910d392&p=3)
[Unity Addressables 1.21.14 Manaual](https://docs.unity.cn/Packages/com.unity.addressables@1.21/manual/ProfileVariables.html)
[Unity 2022.3 Document]((https://docs.unity.cn/cn/2022.3/Manual/AssetBundles-Dependencies.html))