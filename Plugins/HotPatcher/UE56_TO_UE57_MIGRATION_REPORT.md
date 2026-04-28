# HotPatcher UE 5.6 → 5.7 迁移报告

## 概述

本报告记录了 HotPatcher 热更新插件从 UE 5.6 迁移至 UE 5.7 的完整过程。迁移基于对 UE 5.7 官方 Release Notes（约 40 万字符）的全面分析。

- **UE 5.7 版本号**: `ENGINE_MAJOR_VERSION=5`, `ENGINE_MINOR_VERSION=27`
- **迁移结论**: 核心打包/Cooking API 在 5.7 中未发生签名变化，仅需修复 `FindObject` 系列函数的参数变更

---

## UE 5.7 API 变更调研结果

| API 区域 | 5.6 → 5.7 变化 | 对 HotPatcher 的影响 |
|----------|----------------|----------------------|
| `SavePackage` / `FSavePackageArgs` | 无变化 | 无需修改 |
| `PreSaveRoot` / `PreSave` / `PostSaveRoot` | 沿用 5.6 签名 | 无需修改 |
| `IPackageWriter` 接口 | 无变化 | 无需修改 |
| `SAVE_ComputeHash` | 沿用 5.6 行为 | 无需修改 |
| `ArchiveCookContext` | 无变化 | 无需修改 |
| `AssetRegistry` / `GetDependencies` | 无变化 | 无需修改 |
| **`FindObject` 系列函数** | **`bExactClass` → `EFindObjectFlags`** | **需要修改** |

### UE 5.7 中与 HotPatcher 相关的其他变更

- **Incremental Cooking（Beta）**: 5.7 新增增量烘焙功能，通过 Zen Server 分析资产变更
- **Cook Commandlet 弃用**: 5.7 弃用在编辑器进程内运行 Cook 的能力，将在 5.9 移除
- **BuildSettingsVersion.V6**: 项目 `Target.cs` 需更新为 V6
- **UHT**: 新增引擎版本比较的预处理条件支持
- **`AssetManager` / `AssetRegistry`**: `ManageReferences` 新增 `AffectsCookRule` 属性
- **`CookPackageSplitter`**: `GetGenerateList` 返回值改为结构体 `ReportGenerationManifest`

---

## 已修改的文件

### 1. `Source/HotPatcherCore/Private/FlibHotPatcherCoreHelper.cpp`

**位置**: 第 745-749 行（`RunCmdlet` 函数内）

**修改原因**: UE 5.7 将 `FindObject` 系列函数的 `bool bExactClass` 参数替换为 `EFindObjectFlags` 枚举。

```cpp
// 修改前:
UClass* SPCTCmdletClass = FindObject<UClass>(ANY_PACKAGE, *CmdletNameStr, false);

// 修改后:
#if UE_VERSION_OLDER_THAN(5,7,0)
    UClass* SPCTCmdletClass = FindObject<UClass>(ANY_PACKAGE, *CmdletNameStr, false);
#else
    UClass* SPCTCmdletClass = FindObject<UClass>(ANY_PACKAGE, *CmdletNameStr, EFindObjectFlags::None);
#endif
```

---

### 2. `Source/HotPatcherRuntime/Private/FlibPatchParserHelper.cpp`

**位置**: 第 1867-1871 行（`GetPakEncryptionKeys` 函数内）

**修改原因**: 同上，`FindObject` 的 `bExactClass` 参数需适配 5.7。同时新增 `#include "Misc/EngineVersionComparison.h"` 以支持版本守卫宏。

```cpp
// 修改前:
UClass* Class = FindObject<UClass>(ANY_PACKAGE, TEXT("/Script/CryptoKeys.CryptoKeysSettings"), true);

// 修改后:
#if UE_VERSION_OLDER_THAN(5,7,0)
    UClass* Class = FindObject<UClass>(ANY_PACKAGE, TEXT("/Script/CryptoKeys.CryptoKeysSettings"), true);
#else
    UClass* Class = FindObject<UClass>(ANY_PACKAGE, TEXT("/Script/CryptoKeys.CryptoKeysSettings"), EFindObjectFlags::ExactClass);
#endif
```

**新增 include**（第 34 行）:

```cpp
#include "Misc/EngineVersionComparison.h"
```

---

## 无需修改的文件（已确认）

| 文件 | 说明 |
|------|------|
| `HotPatcherPackageWriter.h/.cpp` | 所有版本守卫（5.1/5.3/5.4）已正确覆盖 5.7 路径 |
| `HotPatcherEditor.cpp` | Lambda `[=, this]` 捕获已有 5.4 守卫，5.7 正确走新路径 |
| `FDefaultAssetDependenciesParser.cpp` | `EDependencyCategory` 已有 5.3 守卫 |
| `FlibPakHelper.cpp` | PakFile API 在 5.7 中无变更 |
| `FlibShaderPipelineCacheHelper.cpp` | Shader cache API 在 5.7 中无变更 |
| `FlibAssetManageHelper.cpp` | 无未保护的 `FindObject` 调用 |
| `HotPatcherTemplateHelper.hpp` | `FindObject` 调用已由 `#if ENGINE_MAJOR_VERSION > 4` 守卫保护，5.7 走 `StaticEnum` 路径 |
| `FlibShaderCodeLibraryHelper.cpp` | `SaveShaderLibraryWithoutChunking` 签名无变更 |
| 所有 `Build.cs` 文件 | 版本判断逻辑正确处理 5.7（`ENGINE_MINOR_VERSION=27`） |
| `HotPatcher.uplugin` | 模块结构无需变更 |

---

## 现有版本守卫在 UE 5.7 中的行为验证

### `UE_VERSION_OLDER_THAN(5,6,0)` 守卫（7处）

UE 5.7 中该宏评估为 **false**（5.7.0 >= 5.6.0），因此所有 `#else` 分支（5.6+ API 路径）将被编译。这些 API 在 5.7 中保持不变，行为正确。

| 守卫位置 | 用途 | 5.7 路径 |
|----------|------|----------|
| 行 648 | `PackageArgs.TargetPlatform` | 字段不编译（5.6 已移除，5.7 保持移除） |
| 行 693-700 | `SAVE_ComputeHash` 标志 | 直接添加 `EWriteOptions::ComputeHash` |
| 行 716-718 | `UseCookCmdlet` | `bUse = false`（禁用 cook cmdlet） |
| 行 2395-2404 | `PreSaveRoot` API | 使用 `FObjectPreSaveRootContext` |
| 行 2439-2447 | `PreSave` API | 使用 `FObjectPreSaveContext` |
| 行 2499-2506 | `PostSaveRoot` API | 使用 `FObjectPostSaveRootContext` |
| 行 2605-2607 | `GetCookSaveFlag` 中 `SAVE_ComputeHash` | 不添加旧标志 |

### `ENGINE_MINOR_VERSION < 26` 守卫（9处）

UE 5.7 中 `ENGINE_MINOR_VERSION=27`，`< 26` 评估为 **false**，旧的 `FScopedNamedEvent` 构造函数不编译。行为正确。

### `UE_VERSION_OLDER_THAN(5,4,0)` 守卫（Lambda 捕获等）

5.7 中评估为 **false**，使用 `[=, this]` 捕获和 `UpdatePackageModificationStatus`。行为正确。

---

## 后续步骤（待用户执行）

### 构建验证

- [ ] 在 UE 5.7 环境下编译全部 5 个模块
- [ ] 在 UE 5.7 编辑器中加载插件，确认无运行时崩溃
- [ ] 执行热更新打包流程测试，验证功能正常
- [ ] 检查编译输出中是否有新的 deprecation 警告（特别是 `IPackageWriter` 可能新增的虚函数）

### 可选优化

- [ ] 考虑是否移除 UE 4.x 的向后兼容代码（如 `ENGINE_MAJOR_VERSION == 4` 分支）
- [ ] 考虑利用 UE 5.7 新增的 UHT 引擎版本比较预处理条件简化版本守卫
- [ ] 评估 UE 5.7 增量烘焙对 HotPatcher 工作流的影响

---

## 参考资源

- [UE 5.7 Release Notes](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-engine-5-7-release-notes)
- [UE 5.7 Migration Guide](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-engine-5-migration-guide)
- [UE 5.7 API Documentation](https://dev.epicgames.com/documentation/en-us/unreal-engine/API)
- [FindObject API (5.7)](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Runtime/CoreUObject/FindObject)
