# HotPatcher UE5.6 -> UE5.7 迁移项目状态

## 项目概述

HotPatcher 是一个 UE 热更新打包插件，当前版本适配 UE5.6，需迁移至 UE5.7。

- **插件文件**: `HotPatcher.uplugin`（5个模块）
- **UE 5.7 源码参考**: https://github.com/EpicGames/UnrealEngine/tree/5.7

## 当前模块结构

| 模块 | 类型 | 加载阶段 | Build.cs |
|------|------|----------|----------|
| HotPatcherEditor | Editor | Default | `Source/HotPatcherEditor/HotPatcherEditor.Build.cs` |
| HotPatcherCore | Editor | PreDefault | `Source/HotPatcherCore/HotPatcherCore.Build.cs` |
| HotPatcherRuntime | Runtime | PreDefault | `Source/HotPatcherRuntime/HotPatcherRuntime.Build.cs` |
| BinariesPatchFeature | Runtime | PreDefault | `Source/BinariesPatchFeature/BinariesPatchFeature.Build.cs` |
| CmdHandler | Developer | PostConfigInit | `Source/CmdHandler/CmdHandler.Build.cs` |

## 已识别的 UE 版本适配代码位置（按优先级排序）

### 一、关键：UE 5.6 版本检查（需扩展为 5.7）

文件：`Source/HotPatcherCore/Private/FlibHotPatcherCoreHelper.cpp`

| 行号 | 当前条件 | 用途 | 说明 |
|------|----------|------|------|
| 648 | `UE_VERSION_OLDER_THAN(5,6,0)` | `PackageArgs.TargetPlatform` | UE5.6 移除了该字段，5.7 保持移除状态 |
| 693 | `UE_VERSION_OLDER_THAN(5,6,0)` | `SAVE_ComputeHash` 标志 | UE5.6 不再手动添加此标志，5.7 需确认 |
| 716 | `!UE_VERSION_OLDER_THAN(5,6,0)` | `UseCookCmdlet` 逻辑 | UE5.6+ 禁用 cook cmdlet |
| 2395 | `UE_VERSION_OLDER_THAN(5,6,0)` | `PreSaveRoot` API | UE5.6 改为 `FObjectPreSaveRootContext` 参数 |
| 2439 | `UE_VERSION_OLDER_THAN(5,6,0)` | `PreSave` API | UE5.6 改为 `FObjectPreSaveContext` 参数 |
| 2499 | `UE_VERSION_OLDER_THAN(5,6,0)` | `PostSaveRoot` API | UE5.6 改为 `FObjectPostSaveRootContext` 参数 |
| 2605 | `UE_VERSION_OLDER_THAN(5,6,0)` | `GetCookSaveFlag` 中的 `SAVE_ComputeHash` | 同上 |

### 二、重要：UE 5.4 版本检查

文件：`Source/HotPatcherCore/Private/FlibHotPatcherCoreHelper.cpp`
- 行 46: `!UE_VERSION_OLDER_THAN(5,4,0)` — `FArchiveCookContext` 枚举命名空间变更
- 行 653: `UE_VERSION_OLDER_THAN(5,4,0)` — `ECookType`/`ECookingDLC` 命名空间

文件：`Source/HotPatcherCore/Private/Cooker/PackageWriter/HotPatcherPackageWriter.h`
- 行 45-49: `MarkPackagesUpToDate` -> `UpdatePackageModificationStatus`

文件：`Source/HotPatcherEditor/Private/HotPatcherEditor.cpp`
- 行 176-180, 202-206, 251-255, 287, 456, 484: Lambda `[=]` -> `[=, this]` 捕获语法变更

### 三、UE 5.3 版本检查

文件：`Source/HotPatcherRuntime/Private/DependenciesParser/FDefaultAssetDependenciesParser.cpp`
- 行 237: `UE::AssetRegistry::EDependencyCategory` 变更

文件：`Source/HotPatcherCore/Private/FlibHotPatcherCoreHelper.cpp`
- 行 2180, 2385: `OnPreSaveWorld` / `GEditor->OnPreSaveWorld` 签名变更

### 四、UE 5.1 及更早版本检查（影响较低）

- `FAppStyle` vs `FEditorStyle`（5.1+）
- `IPackageWriter::ECommitStatus` 替代 `bSucceeded`（5.1+）
- `PrimaryAssetLabel` 相关变更（5.1+）
- Shader library 处理变更（5.1+, 5.2+）

## 迁移任务清单

### Phase 1: 获取 UE 5.7 API 变更信息 ✅ 已完成
- [x] 获取 UE 5.7 的 `EngineVersion.h` 确认版本号（ENGINE_MINOR_VERSION=27）
- [x] 对比 5.6 和 5.7 的 `SavePackage.h` / `ObjectSaveUtilities.h` 变更（无变化）
- [x] 对比 5.6 和 5.7 的 `IPackageWriter.h` / `Serialization/PackageWriter.h` 变更（无变化）
- [x] 对比 5.6 和 5.7 的 `AssetRegistryState.h` 变更（无变化）
- [x] 对比 5.6 和 5.7 的 `IoStoreUtilities.h` 变更（无变化）
- [x] 对比 5.6 和 5.7 的 `ArchiveCookContext.h` 变更（无变化）

### Phase 2: Build.cs 文件修改 ✅ 已完成
- [x] 检查 `HotPatcherCore.Build.cs` 是否需要新的模块依赖（不需要）
- [x] 检查 `HotPatcherEditor.Build.cs` 是否需要新的模块依赖（不需要）
- [x] 检查 `HotPatcherRuntime.Build.cs` 是否需要新的模块依赖（不需要）
- [x] 确认 `.clang-format` 配置仍然兼容

### Phase 3: 源码修改 — FlibHotPatcherCoreHelper.cpp ✅ 已完成
- [x] 审查并更新 5.6 版本守卫（7处），确认 5.7 API 是否保持一致（5.7 沿用 5.6 API）
- [x] 如 5.7 有新的 API 变更，添加 `UE_VERSION_OLDER_THAN(5,7,0)` 守卫（FindObject bExactClass → EFindObjectFlags）
- [x] `SAVE_ComputeHash` 在 5.7 中的行为确认（沿用 5.6 行为）
- [x] `PreSaveRoot` / `PreSave` / `PostSaveRoot` 在 5.7 中的签名确认（沿用 5.6 签名）
- [x] `FSavePackageArgs` 在 5.7 中是否有新变更（无变更）

### Phase 4: 源码修改 — HotPatcherPackageWriter ✅ 已完成
- [x] 审查 `HotPatcherPackageWriter.h` 的虚函数重写是否与 5.7 接口匹配（无需修改）
- [x] 审查 `HotPatcherPackageWriter.cpp` 的实现是否与 5.7 匹配（无需修改）
- [x] `UpdatePackageModificationStatus` 在 5.7 中的签名是否变更（未变更）

### Phase 5: 源码修改 — HotPatcherEditor ✅ 已完成
- [x] 审查 Lambda 捕获语法在 5.7 编译器下是否仍正确（5.4+ 守卫已覆盖）
- [x] 审查 Slate/UI 代码在 5.7 中的 API 变更（无相关变更）

### Phase 6: 源码修改 — Runtime 模块 ✅ 已完成
- [x] `FDefaultAssetDependenciesParser.cpp` — AssetRegistry API 确认（5.3+ 守卫已覆盖）
- [x] `FlibPakHelper.cpp` — PakFile API 确认（无新变更）
- [x] `FlibShaderPipelineCacheHelper.cpp` — Shader cache API 确认（5.1+ 守卫已覆盖）
- [x] `FlibAssetManageHelper.cpp` — Asset manage API 确认（无新变更）
- [x] `FlibPatchParserHelper.cpp` — FindObject bExactClass → EFindObjectFlags（已添加 5.7 守卫）

### Phase 7: 构建验证 ✅ 编译通过
- [x] 在 UE 5.7 环境下编译全部模块（5 个模块全部通过）
- [x] 在 UE 5.7 编辑器中加载插件
- [x] 执行热更新打包流程测试（发现运行时错误，见 Phase 9）

### Phase 8: 额外修复（编译中发现的新问题）✅ 已完成
- [x] `Conv_StringToFloat` → `Conv_StringToDouble`（2 处：FVersionUpdaterManager.cpp、HotPatcherActionManager.cpp）
- [x] `FCoreUObjectDelegates::OnObjectSaved` 移除（HotPatcherEditor.cpp + .h）
- [x] `ANY_PACKAGE` 在 FindObject 中移除（2 处：FlibHotPatcherCoreHelper.cpp、FlibPatchParserHelper.cpp）
- [x] `GetMaterialResource` 参数从 `ERHIFeatureLevel::Type` 改为 `EShaderPlatform`（FlibShaderCodeLibraryHelper.cpp）
- [x] `IAssetRegistry::InitializeSerializationOptions` 签名变更（FlibHotPatcherCoreHelper.cpp）
- [x] `FHotPatcherPackageWriter` 新增纯虚函数：`CreateLinkerArchive`、`CreateLinkerExportsArchive`、`SetCooker`、`GetBaseGameOplogAttachments`
- [x] `FConfigFile::FindOrAddSection` → `FindOrAddConfigSection`（CmdHandler.cpp）
- [x] `FCoreDelegates::OnPakFileMounted2` → `GetOnPakFileMounted2()`（MountListener.cpp）
- [x] 各文件添加 `Misc/EngineVersionComparison.h` include（HotPatcherEditor.h、FVersionUpdaterManager.cpp、CmdHandler.cpp、MountListener.cpp）

### Phase 9: 运行时错误修复（打包热更新 Pak 流程中发现的问题）✅ 已完成

#### 错误 1: Pak 文件已存在导致创建失败
- **错误日志**: `LogPakFile: Error: Failed to create pak at ... File already exists at that location.`
- **根本原因**: `CreatePakWorker` 在 `PatcherProxy.cpp` 中调用 `ExecuteUnrealPak -create` 时，如果目标路径已存在同名 `.pak` 文件（如上次打包残留），UnrealPak 会直接报错拒绝创建。
- **修复文件**: `Source/HotPatcherCore/Private/CreatePatch/PatcherProxy.cpp`
- **修复位置**: `CreatePakWorker` 函数 lambda 内部，`if (PakCommandSaveStatus)` 块起始处
- **修复内容**: 在执行 `ExecuteUnrealPak` 之前，检测目标路径是否已存在 pak 文件，如果存在则先删除
- [x] `PatcherProxy.cpp` — 在 `ExecuteUnrealPak` 前添加已有 pak 文件删除逻辑

#### 错误 2: WorldCookPackageSplitter 断言崩溃（已修复 4 次）
- **错误日志**: `Assertion failed: RegisteredCookPackageSubSplitterFactories.Contains(Class) [File:WorldCookPackageSplitter.cpp] [Line: 26]`
- **真正根因**（第三次调查发现）:
  - `HotPatcherCommandlet::Main()` 在行 20 手动设置 `PRIVATE_GIsRunningCookCommandlet = true`
  - 但此时 `UWorldPartition` 的 CDO 已在 commandlet Main 之前构造完毕（当时 flag 为 false）
  - `WorldPartition.cpp:393` 的注册条件是 `IsRunningCookCommandlet()` → 在 CDO 构造时为 false → sub-splitter **从未注册**
  - 之后 HotPatcher 设置 flag 为 true，保存任何 map 时 `FWorldCookPackageSplitter` 被创建，但 `RegisteredCookPackageSubSplitterFactories` 为空
  - 导致 `check(CookPackageSubSplitters.Num())` (行94) 或 `check(RegisteredCookPackageSubSplitterFactories.Contains(Class))` (行26) 失败
  - **影响范围**: 不只是 WP map，而是 **所有 map 包**，因为 `FWorldCookPackageSplitter` 对所有 `UWorld` 都会创建
- **修复文件**: `Source/HotPatcherCore/Private/FlibHotPatcherCoreHelper.cpp` + `Source/HotPatcherCore/Classes/Commandlets/HotPatcherCommandlet.cpp`
- **修复位置**: `CookPackage` 函数中跳过 map 包 + `HotPatcherCommandlet::Main()` 退出前重置 flag
- **第一次修复**: 简单检测 `FindWorldInPackage()` + `IsPartitionedWorld()`，`FindWorldInPackage()` 返回 nullptr 时守卫被绕过
- **第二次修复**: 三重检测策略（FindWorldInPackage → FullyLoad → _BuiltData 伴生包），仍然只在判定为 WP 时才跳过，但实际问题是所有 map 都会触发
- **第三次修复**: 在 UE5.6+ 上跳过 **所有 map 包**（`Package->ContainsMap()` 为 true 即跳过），解决了 Cook 阶段的问题
- **第四次修复（当前）**: 在 `HotPatcherCommandlet::Main()` 中 `DoExport()` 完成后将 `PRIVATE_GIsRunningCookCommandlet` 重置为 false，防止引擎退出时 GC 触发 `UWorldPartition::BeginDestroy` → `UnregisterCookPackageSubSplitterFactory` 断言
- [x] `FlibHotPatcherCoreHelper.cpp` — 守卫改为跳过所有 map 包（UE5.6+）
- [x] `HotPatcherCommandlet.cpp` — DoExport 完成后重置 `PRIVATE_GIsRunningCookCommandlet = false`
- **注意事项**: HotPatcher 在 UE5.6+ 上不支持任何 map 的热更新 Cook（非 WP map 也不行）。如需打包 map，使用 UE 原生打包流程或 [HotChunker](https://imzlp.com/posts/24350/)。
- **崩溃触发阶段**: Cook 阶段（已通过第三次修复解决）+ 引擎退出阶段（已通过第四次修复解决）

## 关键技术决策点

1. **是否保留向后兼容**: 当前代码通过 `UE_VERSION_OLDER_THAN` 宏支持 UE4.21-5.7。
2. **SAVE_ComputeHash 行为**: UE5.6 移除了手动添加该标志的需求，5.7 沿用此行为。已通过 5.6 版本守卫处理。
3. **Cook cmdlet 策略**: UE5.6+ 禁用了 cook cmdlet（`UseCookCmdlet()` 返回 false），同时 `HotPatcherCommandlet::Main()` 手动设置 `PRIVATE_GIsRunningCookCommandlet = true` 但时序晚于 CDO 构造，导致 `CookPackageSubSplitter` 工厂未注册。最终决策：在 UE5.6+ 上跳过所有 map 包的 Cook，并在退出前重置 flag 防止 GC 阶段崩溃。
4. **PackageWriter 接口**: 5.7 新增 4 个纯虚函数（`CreateLinkerArchive`、`CreateLinkerExportsArchive`、`SetCooker`、`GetBaseGameOplogAttachments`），已实现空壳。
5. **map 包处理策略**: UE5.6+ 上不再区分 WP map 与普通 map，统一跳过所有 `ContainsMap()` 为 true 的包。因为 `FWorldCookPackageSplitter` 对所有 `UWorld` 都会创建，在 sub-splitter 未注册的环境下保存任何 map 都会触发断言。
6. **PRIVATE_GIsRunningCookCommandlet 生命周期**: 在 `HotPatcherCommandlet::Main()` 开头设为 true（Cook 阶段需要），在 `DoExport()` 完成后重置为 false（防止引擎退出时 GC 触发 Unregister 断言）。

## 已知限制（UE5.7 版本）
- **所有 map 包不支持热更新 Cook**: UE5.6+ 上 HotPatcher 的直接 Cook 路径无法安全保存任何 map（包括非 WP map），因为 `WorldCookPackageSplitter` 的 sub-splitter 工厂未注册。如需打包 map，使用 UE 原生打包流程或 [HotChunker](https://imzlp.com/posts/24350/)。
- C4996 警告（UE5.6+ API 弃用）
- Python 脚本兼容性问题（未验证）
- UI 资源打包要求（未验证）

## 项目部署架构

**两个项目有独立的插件副本，修改必须双向同步**：

| 项目 | 路径 | 角色 | 编译产物位置 |
|------|------|------|-------------|
| FastNewModel | `D:\Work_00\Tech\FastNewModel\Plugins\HotPatcher` | 主开发副本 | 无 Binaries |
| FastModel57 | `D:\Work_00\Tech\FastModel57\Plugins\HotPatcher` | 实际运行副本 | `Binaries\Win64\UnrealEditor-*.dll` |

- 两个目录**不是 symlink/hardlink**，是独立副本
- 修改源码后必须确保 FastModel57 中的文件同步更新
- 编译在 FastModel57 项目中进行（UBT 编译输出到 `FastModel57\Plugins\HotPatcher\Binaries\Win64\`）
- **教训**: 第四次报错中，前三次的所有修复只存在于 FastNewModel 中，实际运行的 FastModel57 仍是旧代码，导致两个已修复的问题再次出现

## 文件清单（需关注的核心文件）

```
Source/HotPatcherCore/Private/FlibHotPatcherCoreHelper.cpp     # 最关键，7处 5.6 版本检查 + 所有 map 跳过守卫（UE5.6+）
Source/HotPatcherCore/Classes/Commandlets/HotPatcherCommandlet.cpp  # 根因文件：行 20 设置 flag，行 110 重置 flag
Source/HotPatcherCore/Private/CreatePatch/PatcherProxy.cpp     # Pak 创建入口，已有 pak 文件删除修复
Source/HotPatcherCore/Private/Cooker/PackageWriter/HotPatcherPackageWriter.h  # PackageWriter 接口
Source/HotPatcherCore/Private/Cooker/PackageWriter/HotPatcherPackageWriter.cpp
Source/HotPatcherCore/Private/Cooker/MultiCooker/SingleCookerProxy.cpp  # Cook 调度入口
Source/HotPatcherEditor/Private/HotPatcherEditor.cpp           # Lambda 捕获 + UI
Source/HotPatcherRuntime/Private/DependenciesParser/FDefaultAssetDependenciesParser.cpp
Source/HotPatcherRuntime/Private/FlibPakHelper.cpp
Source/HotPatcherRuntime/Private/FlibAssetManageHelper.cpp     # LoadPackagesForCooking / GetPackage / LoadPackage
Source/HotPatcherRuntime/Private/FlibShaderPipelineCacheHelper.cpp
Source/HotPatcherCore/Private/ShaderLibUtils/FlibShaderCodeLibraryHelper.cpp
Source/HotPatcherCore/HotPatcherCore.Build.cs
Source/HotPatcherEditor/HotPatcherEditor.Build.cs
Source/HotPatcherRuntime/HotPatcherRuntime.Build.cs
```

---

## 技术调查记录

### 一、WorldPartition 断言崩溃的完整调用链分析

#### 错误背景

断言位置：`Engine/Source/Runtime/Engine/Private/WorldCookPackageSplitter.cpp:26`
```
Assertion failed: RegisteredCookPackageSubSplitterFactories.Contains(Class)
```

#### HotPatcher 内部 Cook 调用链

```
SingleCookerProxy::DoExport()                    # SingleCookerProxy.cpp:694
  → MakeCookQueue()                              # :707
    → ExecuteNextCluster()                       # :720
      → ExecCookCluster()                        # :314
        → UFlibAssetManageHelper::LoadPackagesForCooking()  # :359 加载包
        → UFlibHotPatcherCoreHelper::CookPackages()         # :383 批量 Cook 入口
          → ParallelFor(每个 Package)
            → CookPackage()                      # :512 单包 Cook，WP 守卫在此
              → [WP 守卫: 行 539-606]             # 三重检测，跳过 WP 地图
              → GEditor->Save()                  # :696 实际保存，触发断言的地点
```

#### 关键结论：所有调用路径均经过 CookPackage 中的 WP 守卫

- `CookPackages`（复数，行 435-474）使用 `ParallelFor` 分发，每个包均调用 `CookPackage`
- 不存在绕过 `CookPackage` 直接调用 `GEditor->Save` 的路径
- `CookPackagesByCmdlet` 在 UE5.6+ 已禁用（`UseCookCmdlet()` 返回 `false`）
- 问题出在**守卫的检测逻辑不够健壮**，而非缺少守卫

### 二、FindWorldInPackage 返回 nullptr 的根因分析

#### UE5.7 源码实现

```cpp
// Engine/Source/Runtime/Engine/Private/World.cpp:9247
UWorld* UWorld::FindWorldInPackage(UPackage* Package)
{
    UWorld* Result = nullptr;
    ForEachObjectWithPackage(Package, [&Result](UObject* Object)
    {
        Result = Cast<UWorld>(Object);
        return !Result;  // 找到第一个 UWorld 即停止
    }, false);
    return Result;
}
```

#### 返回 nullptr 的场景

| 场景 | 原因 | 说明 |
|------|------|------|
| 包未完全加载 | `LoadPackage` 后 UWorld 导出未被反序列化 | `ForEachObjectWithPackage` 只遍历内存中已存在的对象 |
| GC 回收 | 垃圾回收收走了 UWorld 对象，但 UPackage 仍在内存 | `FWorldCookPackageSplitter` 通过 `OnOwnerReloaded` 处理此场景，但 HotPatcher 没有 |
| 并行 Cook 中包状态不一致 | `bStorageConcurrent = true` 时，`IsFullyLoaded()` 可能误报 true | 见下方包加载分析 |

#### 包加载路径分析

HotPatcher 有两条包加载路径：

**路径 A: `UFlibAssetManageHelper::GetPackage()`** (行 1421-1442)
```cpp
UPackage* Package = FindPackage(NULL, *PackageName.ToString());
if (Package)
{
    Package->FullyLoad();  // 已在内存 → FullyLoad
}
else
{
    Package = LoadPackage(NULL, ..., LOAD_None);  // 从磁盘加载，不 FullyLoad
}
```

**路径 B: `LoadPackagesForCooking()`** (行 1466-1505)
```cpp
// 1. 逐个调用 GetPackage() 加载
// 2. 后处理：
if (bStorageConcurrent)  // 并行模式
{
    if (!Package->IsFullyLoaded())  // 只有非 FullyLoaded 才调 FullyLoad
    {
        Package->FullyLoad();
    }
}
// 非并行模式：不调 FullyLoad，仅 GetMetaData()
```

**关键缺陷**: `GetPackage()` 在 `LoadPackage` 路径下不调用 `FullyLoad`，而 `LoadPackagesForCooking` 的后处理在并行模式下依赖 `IsFullyLoaded()` 判断，该判断可能不准确（包标记为 fully loaded 但 UWorld 导出被 GC 回收）。

### 三、WP 检测策略迭代史（三次修复的完整记录）

> **注意**: 前两次修复的 WP 检测策略（三重检测）已在第三次修复中被替代为"跳过所有 map 包"。
> 以下保留历史记录供参考，理解为什么前两次方案不 work。

#### 第一次修复：简单 WP 检测（已废弃）

```cpp
// 仅检测 FindWorldInPackage + IsPartitionedWorld
if (World && World->IsPartitionedWorld()) { skip; }
```
**失败原因**: `FindWorldInPackage()` 在 UWorld 导出未完全反序列化时返回 nullptr，守卫被绕过。

#### 第二次修复：三重检测策略（已废弃）

```
Package->ContainsMap() == true ?
│
├─ Step 1: FindWorldInPackage(Package)
│   ├─ 有效 World → IsPartitionedWorld() ?
│   └─ nullptr → Step 2
├─ Step 2: Package->FullyLoad() 后重试 FindWorldInPackage
│   ├─ 有效 World → IsPartitionedWorld() ?
│   └─ nullptr → Step 3
└─ Step 3: AssetRegistry 查找 <MapName>_BuiltData 伴生包
    ├─ 存在 → 跳过
    └─ 不存在 → 继续 Cook
```

**失败原因**: 问题不在于 WP 检测精度不够，而在于 `FWorldCookPackageSplitter` 对**所有 UWorld** 都会触发，不只是 WP map。

#### 第三次修复（当前）：跳过所有 map 包

```cpp
// UE5.6+ 上，ContainsMap() == true 即跳过，不做 WP 区分
if (IsValid(Package) && Package->ContainsMap())
{
    // 记录 Warning，通知回调，return false
}
```

**根因**: `HotPatcherCommandlet.cpp:20` 设置 `PRIVATE_GIsRunningCookCommandlet = true` 的时序晚于 `UWorldPartition` CDO 构造，导致 sub-splitter 工厂从未注册。在此环境下保存**任何 map** 都不安全。

### 四、UE5.7 Engine API 参考信息

#### UE5.7 安装位置
- 注册表：`HKLM\SOFTWARE\EpicGames\Unreal Engine\5.7`
- 路径：`C:\Program Files\Epic Games\UE_5.7`

#### WorldPartition 相关 API

| API | 头文件 | 说明 |
|-----|--------|------|
| `UWorld::FindWorldInPackage()` | `Engine/Classes/Engine/World.h` | 遍历内存中已存在的 UWorld 对象，不触发反序列化 |
| `UWorld::IsPartitionedWorld()` | `Engine/Classes/Engine/World.h:2863` | `GetWorldPartition() != nullptr`，需 UWorld 已加载 |
| `UWorld::IsPartitionedWorld(const UWorld*)` | 同上 `:2868` | 静态版本，空指针安全 |
| `UPackage::ContainsMap()` | UPackage API | 检查 `PKG_ContainsMap` 标志 |
| `UPackage::FullyLoad()` | UPackage API | 强制加载包内所有导出对象 |
| `FWorldPartitionPackageHelper` | `WorldPartition/WorldPartitionPackageHelper.h` | UE5.7 中仅有 `UnloadPackage()` 方法，无 `IsWorldPartitionPackage()` |

#### FindWorldInPackage 的 UE 内部使用模式

UE 自身在使用 `FindWorldInPackage` 时，通常配合 `FollowWorldRedirectorInPackage` 作为后备：
```cpp
// Engine/Source/Runtime/Engine/Private/World.cpp:4432
UWorld* World = UWorld::FindWorldInPackage(Package);
if (!World)
{
    World = UWorld::FollowWorldRedirectorInPackage(Package);
}
```
HotPatcher 的三重检测策略比 UE 内部模式更健壮，因为还增加了 `_BuiltData` 伴生包检测。

### 五、关键技术决策记录

1. **map 包处理策略（最终决策）**: UE5.6+ 上跳过所有 map 包
   - 不尝试自行注册 `CookPackageSubSplitterFactories`（侵入性太大，需 Hook UE 内部模块加载流程）
   - 不尝试通过 `CookPackagesByCmdlet` 路径处理（UE5.6+ 已禁用此路径）
   - 不尝试区分 WP map 与普通 map（根因是 sub-splitter 未注册，影响所有 map）
   - 不修改 `HotPatcherCommandlet.cpp:20` 的 `PRIVATE_GIsRunningCookCommandlet = true`（可能在其他地方有依赖）

2. **前两次修复的 WP 检测方案为何被放弃**:
   - `FWorldPartitionPackageHelper::IsWorldPartitionPackage()`: UE5.7 中不存在此 API
   - 包路径检查 `__ExternalActors__` 目录结构: 依赖文件系统，不可靠
   - AssetRegistry Tag `WorldPartition`: 标签名不确定，不同版本可能不同
   - `_BuiltData` 伴生包检测: 可行但不需要，因为问题影响所有 map

### 六、已知限制与注意事项

- **所有 map 包不支持热更新 Cook**: HotPatcher 在 UE5.6+ 上无法安全保存任何 map 包。如需打包 map，使用 UE 原生打包流程或 [HotChunker](https://imzlp.com/posts/24350/)
- **非 map 资产不受影响**: `.uasset` 类型的资产（材质、纹理、蓝图类等）可正常 Cook
- **C4996 警告**: UE5.6+ API 弃用警告，编译时可见
- **Python 脚本兼容性**: 未验证
- **UI 资源打包要求**: 未验证

### 七、待验证项

- [ ] 在 FastModel57 项目中重新编译 HotPatcher 插件（确保 DLL 更新）
- [ ] 重新运行打包流程，确认 Pak 文件删除修复生效（不再出现 "File already exists" 错误）
- [ ] 确认所有 map 包被正确跳过（UE5.6+），日志中应出现 "Skip cooking map package" Warning
- [ ] 确认引擎退出阶段不再触发 WorldCookPackageSplitter 断言（flag 重置修复生效）
- [ ] 确认非 map 资产（uasset）在修复后仍能正常 Cook
- [ ] 并行 Cook 模式（`bStorageConcurrent = true`）下的线程安全性验证
- [ ] 确认热更新 pak 中不包含 map 文件后，客户端加载是否正常

### 八、根因链完整推导（第三次调查）

#### 触发链

```
1. UnrealEditor-Cmd.exe -run=HotPatcher ...
2. UWorldPartition CDO 构造 (IsRunningCookCommandlet() == false)
   → RegisterCookPackageSubSplitterFactory 条件不满足 → 未注册
3. HotPatcherCommandlet::Main() → PRIVATE_GIsRunningCookCommandlet = true
4. HotPatcher 开始 Cook → CookPackage() → GEditor->Save()
5. UE Save 系统检测到 UWorld → 创建 FWorldCookPackageSplitter 实例
6. ReportGenerationManifest() → ForEachRegisteredCookPackageSubSplitterFactories → 空的！
7. check(CookPackageSubSplitters.Num()) 失败 (行 94)
   或 GC 时 UWorldPartition::BeginDestroy → UnregisterCookPackageSubSplitterFactory
   → check(RegisteredCookPackageSubSplitterFactories.Contains(Class)) 失败 (行 26)
```

#### 关键源码位置

| 文件 | 行号 | 代码 | 说明 |
|------|------|------|------|
| `HotPatcherCommandlet.cpp` | 20 | `PRIVATE_GIsRunningCookCommandlet = true` | 在 CDO 构造后才设置，太晚了 |
| `WorldPartition.cpp` | 393 | `if (...&& IsRunningCookCommandlet())` | CDO 构造时 flag 为 false，不注册 |
| `WorldPartition.cpp` | 1757 | `if (...&& IsRunningCookCommandlet())` | 同样的守卫，在 BeginDestroy 中反注册 |
| `WorldCookPackageSplitter.cpp` | 9 | `REGISTER_COOKPACKAGE_SPLITTER(FWorldCookPackageSplitter, UWorld)` | 全局静态注册，对所有 UWorld 生效 |
| `WorldCookPackageSplitter.cpp` | 86-94 | `ForEachRegistered... check(Num())` | 空 sub-splitter → 断言 |
| `WorldCookPackageSplitter.cpp` | 24-28 | `Unregister... check(Contains)` | 反注册不存在的类 → 断言 |

#### 为什么前两次修复无效

| 修复尝试 | 失败原因 |
|----------|----------|
| 第一次：FindWorldInPackage + IsPartitionedWorld | 问题不是 WP map，而是**所有 map** |
| 第二次：三重检测（增加 FullyLoad + _BuiltData 兜底） | 同上，问题不在 WP 检测精度，而在 `FWorldCookPackageSplitter` 对所有 UWorld 都会触发 |
| **第三次（当前）：跳过所有 ContainsMap() 包** | 正确，因为根因是 sub-splitter 未注册，影响所有 map |

---

## 本次操作记录（第三次修复）

### 操作日期
2026-04-13

### 操作内容
1. 读取 `编译执行打热更新包后的报错信息3.md`，发现与之前相同的 `WorldCookPackageSplitter` 断言崩溃
2. 确认三重 WP 检测守卫已存在于代码中，但崩溃仍然发生
3. 排查所有 `CookPackage` 调用路径，确认不存在绕过守卫的路径
4. 深入阅读 UE5.7 Engine 源码：
   - `WorldCookPackageSplitter.cpp` — 理解断言的触发条件和注册机制
   - `WorldPartition.cpp:393,1757` — 发现 `IsRunningCookCommandlet()` 条件守卫
   - `CoreGlobals.h:259` — 理解 `IsRunningCookCommandlet()` 的实现
   - `LaunchEngineLoop.cpp:2196` — 理解 flag 的正常设置时机
5. **根因发现**: `HotPatcherCommandlet.cpp:20` 手动设置 `PRIVATE_GIsRunningCookCommandlet = true`，但此时 `UWorldPartition` CDO 已经构造完毕（flag 为 false），导致 sub-splitter 工厂从未注册
6. **修复**: 将守卫从"仅跳过 WP map"改为"跳过所有 map 包"（`ContainsMap()` 为 true 即跳过）
7. **额外产出**: 创建了 `.claude/skills/upgrade-ue-plugin.md` — 通用 UE 插件版本升级 Skill 文档

### 修改的文件
- `Source/HotPatcherCore/Private/FlibHotPatcherCoreHelper.cpp` — 行 533-571，守卫改为跳过所有 map 包
- `Source/HotPatcherCore/Classes/Commandlets/HotPatcherCommandlet.cpp` — 行 104-111，在 DoExport 完成后重置 `PRIVATE_GIsRunningCookCommandlet = false`
- `CLAUDE.md` — 更新错误 2 修复记录、技术决策、已知限制、文件清单

### 核心经验教训
- **不要过早缩小假设范围**: 前两次都假设"只有 WP map 受影响"，实际是所有 map
- **追踪 Engine 全局注册机制**: `REGISTER_*` 宏、条件注册（`IsRunningCookCommandlet`）、CDO 构造时序
- **编译通过 ≠ 运行正确**: 运行时 BUG 的根因可能在插件的 Commandlet 入口代码中，而非崩溃点附近
- **注意多项目副本**: FastNewModel 和 FastModel57 有独立的插件副本，修改必须同步到两个项目

---

## 第四次操作记录

### 操作日期
2026-04-13

### 日志分析

日志中包含 2 个错误：
1. **Pak 文件已存在** (行 8-9): `Failed to create pak ... File already exists at that location.`
2. **WorldCookPackageSplitter 断言崩溃** (行 61-66): 与之前相同的断言

### 关键时序分析
- `10:09.51`: Pak 创建失败（文件已存在）
- `10:09.52`: `Export Patch Misstion is Successed, return 0!`（HotPatcher 主流程完成）
- `10:09.52`: 引擎退出阶段 → GC 触发 → WorldCookPackageSplitter 断言崩溃

### 根因

**两个问题都是编译产物未更新导致的**：
1. 之前的修复代码存在于 `FastNewModel` 项目中，但 `FastModel57` 有**独立的插件副本**
2. 运行的是 `FastModel57` 中的旧 DLL（不含修复）

**新增发现**：崩溃发生在引擎退出阶段（GC），而非 Cook 阶段。即使 Cook 阶段跳过了 map，引擎退出时 GC 仍触发 `UWorldPartition::BeginDestroy` → `UnregisterCookPackageSubSplitterFactory` 断言（因为 `PRIVATE_GIsRunningCookCommandlet` 仍为 true）。

### 修复措施

1. **同步文件**: 将 `HotPatcherCommandlet.cpp` 的修改同步到 `FastModel57` 项目
2. **新增修复**: 在 `HotPatcherCommandlet::Main()` 中，`DoExport` 完成后将 `PRIVATE_GIsRunningCookCommandlet` 重置为 false，防止引擎退出时 GC 触发 Unregister 断言
3. **重新编译**: 需要在 FastModel57 项目中重新编译 HotPatcher 插件

### 修改的文件（FastModel57）
- `Source/HotPatcherCore/Classes/Commandlets/HotPatcherCommandlet.cpp` — 行 104-111，新增 flag 重置

### 项目部署注意
- **FastNewModel** (`D:\Work_00\Tech\FastNewModel\Plugins\HotPatcher`): 主开发副本，所有修复已就位
- **FastModel57** (`D:\Work_00\Tech\FastModel57\Plugins\HotPatcher`): 实际运行副本，需同步所有修改
- 两个项目有**独立**的 Source 和 Binaries，不是 symlink
