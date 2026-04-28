# HotPatcher UE 5.7 适配 — 当前状态与待办清单

> 最后更新：2026-04-10
> 状态：**已识别全部编译错误，尚未开始代码修改**

---

## 1. 编译环境

- **项目路径**: `D:\Work_00\Tech\FastNewModel`
- **UE 5.7 引擎路径**: `C:\Program Files\Epic Games\UE_5.7`
- **编译命令**: `"C:/Program Files/Epic Games/UE_5.7/Engine/Build/BatchFiles/Build.bat" FastModelEditor Win64 Development "D:/Work_00/Tech/FastNewModel/FastModel.uproject" -WaitMutex`
- **编译结果**: Failed (OtherCompilationError)，235 个编译单元中部分失败

---

## 2. 已确认的全部编译错误（15 个）

### 错误 1: `ANY_PACKAGE` 宏已移除
- **文件**: `Source/HotPatcherCore/Private/FlibHotPatcherCoreHelper.cpp:748`
- **错误**: `error C2065: 'ANY_PACKAGE': 未声明的标识符`
- **原因**: UE 5.7 移除了 `ANY_PACKAGE` 宏（头文件 `AnyPackagePrivate.h` 已标记 deprecated 且不再导出该宏）
- **当前代码**（已有版本守卫但使用了 `ANY_PACKAGE`）:
  ```cpp
  // 行 745-749
  #if UE_VERSION_OLDER_THAN(5,7,0)
      UClass* SPCTCmdletClass = FindObject<UClass>(ANY_PACKAGE, *CmdletNameStr, false);
  #else
      UClass* SPCTCmdletClass = FindObject<UClass>(ANY_PACKAGE, *CmdletNameStr, EFindObjectFlags::None);  // ERROR: ANY_PACKAGE 不存在
  #endif
  ```
- **UE 5.7 正确 API**:
  ```cpp
  // FindObject 签名（UE 5.7）:
  template<class T>
  inline T* FindObject(UObject* Outer, FStringView Name, EFindObjectFlags Flags = EFindObjectFlags::None);
  // 或使用 FTopLevelAssetPath 版本:
  template<class T>
  inline T* FindObject(FTopLevelAssetPath InPath, EFindObjectFlags Flags = EFindObjectFlags::None);
  ```
- **修复方案**: 将 `ANY_PACKAGE` 替换为 `nullptr`（在 UE 5.7 中 `nullptr` 表示搜索所有顶层对象）:
  ```cpp
  #else
      UClass* SPCTCmdletClass = FindObject<UClass>(nullptr, *CmdletNameStr, EFindObjectFlags::None);
  #endif
  ```

### 错误 2: `ANY_PACKAGE` 宏已移除（第二处）
- **文件**: `Source/HotPatcherRuntime/Private/FlibPatchParserHelper.cpp:1870`
- **错误**: `error C2065: 'ANY_PACKAGE': 未声明的标识符`
- **当前代码**:
  ```cpp
  // 行 1867-1871
  #if UE_VERSION_OLDER_THAN(5,7,0)
      UClass* Class = FindObject<UClass>(ANY_PACKAGE, TEXT("/Script/CryptoKeys.CryptoKeysSettings"), true);
  #else
      UClass* Class = FindObject<UClass>(ANY_PACKAGE, TEXT("/Script/CryptoKeys.CryptoKeysSettings"), EFindObjectFlags::ExactClass);  // ERROR
  #endif
  ```
- **修复方案**: 使用 `FTopLevelAssetPath` 版本替代:
  ```cpp
  #else
      UClass* Class = FindObject<UClass>(FTopLevelAssetPath(TEXT("/Script/CryptoKeys"), TEXT("CryptoKeysSettings")), EFindObjectFlags::ExactClass);
  #endif
  ```
- **需新增 include**: `#include "UObject/TopLevelAssetPath.h"` (如果尚未包含)

### 错误 3: `FConfigFile::FindOrAddSection` 已移除
- **文件**: `Source/CmdHandler/Private/CmdHandler.cpp:41`
- **错误**: `error C2039: "FindOrAddSection": 不是 "FConfigFile" 的成员`
- **当前代码**:
  ```cpp
  // 行 40-42
  auto EngineIniIns = GConfig->FindConfigFile(GEngineIni);
  auto MultiCookerDDCBackendSection = EngineIniIns->FindOrAddSection(MultiCookerDDCBackendName);
  MultiCookerDDCBackendSection->Empty();
  ```
- **UE 5.7 变更**: `FindOrAddSection` 改名为 `FindOrAddConfigSection`，且返回 `const FConfigSection*`（不可直接修改）。同时 `FConfigSection` 的迭代器也改为只读。
- **UE 5.7 可用 API**:
  - `const FConfigSection* FindOrAddConfigSection(const FString& Name)` — 返回只读指针
  - `FConfigSection* FindOrAddSectionInternal(const FString& Name)` — 内部方法，可修改但不可外部访问
  - `GConfig->SetString(Section, Key, Value, Filename)` — 通过 GConfig 设置值
  - `GConfig->SetInSection(Section, Key, Value, Filename)` — 替换 key/value
  - `GConfig->EmptySection(Section, Filename)` — 清空 section
- **修复方案**: 由于代码需要修改 section 内容，最简方案是使用 `GConfig->SetString` + `GConfig->EmptySection` 重写逻辑:
  ```cpp
  // 替代方案：直接通过 GConfig 操作
  GConfig->EmptySection(*MultiCookerDDCBackendName, GEngineIni);
  auto UpdateKeyLambda = [&MultiCookerDDCBackendName](const FString& Key, const FString& Value)
  {
      GConfig->SetString(*MultiCookerDDCBackendName, *Key, *Value, GEngineIni);
  };
  UpdateKeyLambda(TEXT("MinimumDaysToKeepFile"), TEXT("7"));
  UpdateKeyLambda(TEXT("Root"), TEXT("(Type=KeyLength, Length=120, Inner=AsyncPut)"));
  // ... 其余同理
  ```
- **同样涉及**: 行 112-113 的 `EngineIniIns->Remove(MultiCookerDDCBackendName)` 也需要改为 `GConfig->EmptySection`

### 错误 4: `UKismetStringLibrary::Conv_StringToFloat` 已移除（第一处）
- **文件**: `Source/HotPatcherEditor/Private/SVersionUpdater/FVersionUpdaterManager.cpp:123`
- **错误**: `error C2039: "Conv_StringToFloat": 不是 "UKismetStringLibrary" 的成员`
- **当前代码**:
  ```cpp
  Result = UKismetStringLibrary::Conv_StringToFloat(ValueStr);
  ```
- **UE 5.7 变更**: `Conv_StringToFloat` 重命名为 `Conv_StringToDouble`，返回 `double`
- **修复方案**:
  ```cpp
  #if UE_VERSION_OLDER_THAN(5,7,0)
      Result = UKismetStringLibrary::Conv_StringToFloat(ValueStr);
  #else
      Result = static_cast<float>(UKismetStringLibrary::Conv_StringToDouble(ValueStr));
  #endif
  ```

### 错误 5: `UKismetStringLibrary::Conv_StringToFloat` 已移除（第二处）
- **文件**: `Source/HotPatcherEditor/Private/HotPatcherActionManager.cpp:158`
- **错误**: `error C2039: "Conv_StringToFloat": 不是 "UKismetStringLibrary" 的成员`
- **当前代码**:
  ```cpp
  return UKismetStringLibrary::Conv_StringToFloat(Version);
  ```
- **修复方案**: 同错误 4

### 错误 6: `UMaterial::GetMaterialResource` 签名变更
- **文件**: `Source/HotPatcherCore/Private/ShaderLibUtils/FlibShaderCodeLibraryHelper.cpp:279`
- **错误**: `error C2665: 'UMaterial::GetMaterialResource': 没有重载函数可以转换所有参数`
- **当前代码**:
  ```cpp
  if (FMaterialResource* Res = Material->GetMaterialResource((ERHIFeatureLevel::Type)FeatureLevel))
  ```
- **UE 5.7 新签名**:
  ```cpp
  FMaterialResource* GetMaterialResource(EShaderPlatform InShaderPlatform, EMaterialQualityLevel::Type QualityLevel = EMaterialQualityLevel::Num);
  const FMaterialResource* GetMaterialResource(EShaderPlatform InShaderPlatform, EMaterialQualityLevel::Type QualityLevel = EMaterialQualityLevel::Num) const;
  ```
- **变更**: 第一个参数从 `ERHIFeatureLevel::Type` 改为 `EShaderPlatform`
- **修复方案**: 将 `ERHIFeatureLevel::Type` 转换为 `EShaderPlatform`:
  ```cpp
  #if UE_VERSION_OLDER_THAN(5,7,0)
      if (FMaterialResource* Res = Material->GetMaterialResource((ERHIFeatureLevel::Type)FeatureLevel))
  #else
      EShaderPlatform ShaderPlatform = GShaderPlatformForFeatureLevel[FeatureLevel];
      if (FMaterialResource* Res = Material->GetMaterialResource(ShaderPlatform))
  #endif
  ```
  或者使用 `UE::GetShaderPlatformForFeatureLevel` 辅助函数（如果存在）。

### 错误 7: `FCoreDelegates::OnPakFileMounted2` 变更
- **文件**: `Source/HotPatcherRuntime/Private/MountListener.cpp:20`
- **错误**: `error C2039: "OnPakFileMounted2": 不是 "FCoreDelegates" 的成员`
- **当前代码**:
  ```cpp
  #if ENGINE_MAJOR_VERSION >4 || ENGINE_MINOR_VERSION >=26
      FCoreDelegates::OnPakFileMounted2.AddLambda([this](const IPakFile& PakFile){this->OnMountPak(*PakFile.PakGetPakFilename(),0);});
  #endif
  ```
- **UE 5.7 变更**: `OnPakFileMounted2` 从 `static TMulticastDelegate<...>` 改为 getter 函数 `static TTSMulticastDelegate<...>& GetOnPakFileMounted2()`，返回线程安全委托引用
- **修复方案**:
  ```cpp
  #if ENGINE_MAJOR_VERSION >4 || ENGINE_MINOR_VERSION >=26
      FCoreDelegates::GetOnPakFileMounted2().AddLambda([this](const IPakFile& PakFile){this->OnMountPak(*PakFile.PakGetPakFilename(),0);});
  #endif
  ```

### 错误 8: `FCoreUObjectDelegates::OnObjectSaved` 已移除
- **文件**: `Source/HotPatcherEditor/Private/HotPatcherEditor.cpp:98`
- **错误**: `error C2039: "OnObjectSaved": 不是 "FCoreUObjectDelegates" 的成员`
- **当前代码**（已有 deprecation pragma 但仍编译失败）:
  ```cpp
  PRAGMA_DISABLE_DEPRECATION_WARNINGS
  FCoreUObjectDelegates::OnObjectSaved.AddRaw(this,&FHotPatcherEditorModule::OnObjectSaved);
  PRAGMA_ENABLE_DEPRECATION_WARNINGS
  ```
- **UE 5.7 变更**: `FCoreUObjectDelegates::OnObjectSaved` 已完全移除，替换为 `FCoreUObjectDelegates::OnObjectPreSave`
- **新 API**:
  ```cpp
  // UObjectGlobals.h:3327
  DECLARE_MULTICAST_DELEGATE_TwoParams(FOnObjectPreSave, UObject*, FObjectPreSaveContext);
  static FOnObjectPreSave OnObjectPreSave;
  ```
- **关键差异**: 新回调签名是 `(UObject*, FObjectPreSaveContext)` 而非 `(UObject*)`
- **修复方案**:
  ```cpp
  // 修改头文件中的方法签名
  // HotPatcherEditor.h:93
  // void OnObjectSaved(UObject* ObjectSaved);
  // 改为:
  void OnObjectSaved(UObject* ObjectSaved, FObjectPreSaveContext SaveContext);

  // 修改绑定
  FCoreUObjectDelegates::OnObjectPreSave.AddRaw(this, &FHotPatcherEditorModule::OnObjectSaved);
  ```
  注意: 新的 `OnObjectPreSave` 在保存**之前**触发（不是之后），语义有差异。需评估功能影响。`FObjectPreSaveContext` 定义在 `UObject/SavePackage.h` 中。

### 错误 9: `FHotPatcherPackageWriter` 无法实例化（抽象类）
- **文件**: `Source/HotPatcherCore/Private/FlibHotPatcherCoreHelper.cpp:359`
- **错误**: `error C2259: 'FHotPatcherPackageWriter': 无法实例化抽象类`
- **当前代码**:
  ```cpp
  PackageWriter = new FHotPatcherPackageWriter;
  ```
- **原因**: UE 5.7 的 `ICookedPackageWriter` 接口新增了以下纯虚函数，`FHotPatcherPackageWriter` 未实现：
  - `GetBaseGameOplogAttachments()` — 新增的纯虚函数
  - `CookerBeginCacheForCookedPlatformData()` — 替代了旧的 `BeginCacheForCookedPlatformData()`
  - `RegisterDeterminismHelper(ICookedPackageWriter*, UObject*, ...)` — 新增
  - `IsDeterminismDebug()` — 新增的纯虚函数
  - `SetCooker(UE::PackageWriter::Private::ICookerInterface*)` — 新增的纯虚函数
  - `IsPreSaveCompleted()` — 需确认是否为纯虚
  - `UpdateSaveArguments()` — 新增虚函数（有默认实现，非纯虚）
  - `IsAnotherSaveNeeded()` — 新增虚函数（有默认实现，非纯虚）
- **修复方案**: 在 `HotPatcherPackageWriter.h` 和 `.cpp` 中添加缺失的虚函数实现。大部分可返回默认值:
  ```cpp
  // 需要新增的方法:
  virtual void GetBaseGameOplogAttachments(TArrayView<FName> PackageNames,
      TArrayView<FUtf8StringView> AttachmentKeys,
      TUniqueFunction<void(FName, FUtf8StringView, FCbObject&&)>&& Callback) override
  {
      // 同 GetOplogAttachments 实现
  }

  virtual EPackageWriterResult CookerBeginCacheForCookedPlatformData(
      FBeginCacheForCookedPlatformDataInfo& Info) override
  { return EPackageWriterResult::Success; }

  virtual void RegisterDeterminismHelper(ICookedPackageWriter* PackageWriter,
      UObject* SourceObject, const TRefCountPtr<UE::Cook::IDeterminismHelper>&) override {}

  virtual bool IsDeterminismDebug() const override { return false; }

  virtual void SetCooker(UE::PackageWriter::Private::ICookerInterface*) override {}
  ```
  **注意**: 需要确认 `ICookedPackageWriter` 的命名空间。在 UE 5.7 中 `ICookedPackageWriter` 和 `IPackageWriter` 均定义在 `Runtime/Core/Public/Serialization/PackageWriter.h` 中。`SetCooker` 的参数类型 `UE::PackageWriter::Private::ICookerInterface` 需要检查是否可前向声明或需 include。

### 错误 10: `IAssetRegistry::InitializeSerializationOptions` 签名变更
- **文件**: `Source/HotPatcherCore/Private/FlibHotPatcherCoreHelper.cpp:1606`
- **错误**: `error C2665: 'IAssetRegistry::InitializeSerializationOptions': 没有重载函数可以转换所有参数`
- **当前代码**:
  ```cpp
  AssetRegistry->InitializeSerializationOptions(SaveOptions, TargetPlatform->IniPlatformName());
  ```
- **UE 5.7 变更**: 接受 `FString` 的重载已被弃用。新签名为:
  ```cpp
  virtual void InitializeSerializationOptions(
      FAssetRegistrySerializationOptions& Options,
      const ITargetPlatform* TargetPlatform = nullptr,
      UE::AssetRegistry::ESerializationTarget Target = UE::AssetRegistry::ESerializationTarget::ForGame) const = 0;
  ```
- **修复方案**:
  ```cpp
  #if UE_VERSION_OLDER_THAN(5,7,0)
      AssetRegistry->InitializeSerializationOptions(SaveOptions, TargetPlatform->IniPlatformName());
  #else
      AssetRegistry->InitializeSerializationOptions(SaveOptions, TargetPlatform);
  #endif
  ```

---

## 3. 修改清单（按文件分组）

### 文件 1: `Source/HotPatcherCore/Private/FlibHotPatcherCoreHelper.cpp`
| 行号 | 错误编号 | 修改内容 |
|------|----------|----------|
| 748 | #1 | `ANY_PACKAGE` → `nullptr`（FindObject 第一个参数） |
| 359 | #9 | `FHotPatcherPackageWriter` 抽象类（需在 PackageWriter 头文件中补实现） |
| 1606 | #10 | `InitializeSerializationOptions` 改传 `ITargetPlatform*` |

### 文件 2: `Source/HotPatcherRuntime/Private/FlibPatchParserHelper.cpp`
| 行号 | 错误编号 | 修改内容 |
|------|----------|----------|
| 1870 | #2 | `ANY_PACKAGE` → `FTopLevelAssetPath` 版 FindObject |

### 文件 3: `Source/CmdHandler/Private/CmdHandler.cpp`
| 行号 | 错误编号 | 修改内容 |
|------|----------|----------|
| 40-42 | #3 | `FindOrAddSection` → 使用 `GConfig->SetString` / `EmptySection` |
| 112-113 | #3 | `Remove` → `GConfig->EmptySection` |

### 文件 4: `Source/HotPatcherEditor/Private/SVersionUpdater/FVersionUpdaterManager.cpp`
| 行号 | 错误编号 | 修改内容 |
|------|----------|----------|
| 123 | #4 | `Conv_StringToFloat` → `Conv_StringToDouble` + 版本守卫 |

### 文件 5: `Source/HotPatcherEditor/Private/HotPatcherActionManager.cpp`
| 行号 | 错误编号 | 修改内容 |
|------|----------|----------|
| 158 | #5 | `Conv_StringToFloat` → `Conv_StringToDouble` + 版本守卫 |

### 文件 6: `Source/HotPatcherCore/Private/ShaderLibUtils/FlibShaderCodeLibraryHelper.cpp`
| 行号 | 错误编号 | 修改内容 |
|------|----------|----------|
| 279 | #6 | `GetMaterialResource(ERHIFeatureLevel)` → `GetMaterialResource(EShaderPlatform)` + 版本守卫 |

### 文件 7: `Source/HotPatcherRuntime/Private/MountListener.cpp`
| 行号 | 错误编号 | 修改内容 |
|------|----------|----------|
| 20 | #7 | `OnPakFileMounted2.AddLambda` → `GetOnPakFileMounted2().AddLambda` + 版本守卫 |

### 文件 8: `Source/HotPatcherEditor/Private/HotPatcherEditor.cpp`
| 行号 | 错误编号 | 修改内容 |
|------|----------|----------|
| 98 | #8 | `OnObjectSaved` → `OnObjectPreSave` + 修改回调签名 |

### 文件 9: `Source/HotPatcherEditor/Public/HotPatcherEditor.h`
| 行号 | 错误编号 | 修改内容 |
|------|----------|----------|
| 93 | #8 | `OnObjectSaved(UObject*)` 签名改为 `OnObjectSaved(UObject*, FObjectPreSaveContext)` |

### 文件 10: `Source/HotPatcherCore/Private/Cooker/PackageWriter/HotPatcherPackageWriter.h`
| 行号 | 错误编号 | 修改内容 |
|------|----------|----------|
| - | #9 | 新增 `GetBaseGameOplogAttachments`、`SetCooker`、`IsDeterminismDebug`、`RegisterDeterminismHelper`、`CookerBeginCacheForCookedPlatformData` 等虚函数声明 |

### 文件 11: `Source/HotPatcherCore/Private/Cooker/PackageWriter/HotPatcherPackageWriter.cpp`
| 行号 | 错误编号 | 修改内容 |
|------|----------|----------|
| - | #9 | 新增上述虚函数的空实现 |

---

## 4. UE 5.7 API 参考速查表

| 旧 API（5.6） | 新 API（5.7） | 位置 |
|---------------|---------------|------|
| `FindObject<T>(ANY_PACKAGE, Name, bool)` | `FindObject<T>(nullptr, Name, EFindObjectFlags)` 或 `FindObject<T>(FTopLevelAssetPath, EFindObjectFlags)` | `UObjectGlobals.h` |
| `FConfigFile::FindOrAddSection()` | `GConfig->SetString()`, `GConfig->EmptySection()`, `GConfig->SetInSection()` | `ConfigCacheIni.h` |
| `UKismetStringLibrary::Conv_StringToFloat()` | `UKismetStringLibrary::Conv_StringToDouble()` | `KismetStringLibrary.h` |
| `UMaterial::GetMaterialResource(ERHIFeatureLevel)` | `UMaterial::GetMaterialResource(EShaderPlatform)` | `Material.h` |
| `FCoreDelegates::OnPakFileMounted2` | `FCoreDelegates::GetOnPakFileMounted2()` | `CoreDelegates.h` |
| `FCoreUObjectDelegates::OnObjectSaved` | `FCoreUObjectDelegates::OnObjectPreSave` | `UObjectGlobals.h` |
| `IAssetRegistry::InitializeSerializationOptions(Options, FString)` | `IAssetRegistry::InitializeSerializationOptions(Options, ITargetPlatform*)` | `IAssetRegistry.h` |

---

## 5. 额外警告（非阻断，但将在未来版本变为错误）

| 警告 | 文件 | 说明 |
|------|------|------|
| C4996 `EAssetRegistryDependencyType::Type` | `FlibAssetManageHelper.h:108,109,111,214` | 已弃用，需迁移到新 API |
| C4996 `SListView::ItemHeight` | `SHotPatcherCookMaps.cpp:35`, `SHotPatcherCookSetting.cpp:36`, `SHotPatcherCookedPlatforms.cpp:33` | 仅用于 Tile 模式 |
| C4996 `FJsonObject::GetStringField(ANSI)` | `SHotPatcherCookSetting.cpp:112` | 改用 TCHAR 版本 |
| C4996 `FRHITextureCreateDesc::SetBulkData` | RuntimeImageLoader 插件（非 HotPatcher） | 改名为 `SetInitActionBulkData` |
| C4996 `RHIUpdateTextureReference` | RuntimeImageLoader 插件（非 HotPatcher） | 需传入 immediate command list |
| C4996 `FImageUtils::CompressImageArray` | `SaveRenderTargetToFile.cpp`（项目代码） | 改用 `PNGCompressImageArray` |

---

## 6. 已完成的先前修改（保留）

以下修改在之前的迁移分析中已完成（添加了 `UE_VERSION_OLDER_THAN(5,7,0)` 版本守卫），但 `FindObject` 的 `ANY_PACKAGE` 参数仍需修正：

1. `FlibHotPatcherCoreHelper.cpp:745-749` — FindObject bExactClass → EFindObjectFlags（已完成，但 ANY_PACKAGE 需修正）
2. `FlibPatchParserHelper.cpp:1867-1871` — 同上（已完成，但 ANY_PACKAGE 需修正）
3. `FlibPatchParserHelper.cpp:34` — 新增 `#include "Misc/EngineVersionComparison.h"`（已完成）

---

## 7. 执行计划（按优先级排序）

### Phase 1: 修复所有编译错误（阻断性）

1. **修复 FindObject + ANY_PACKAGE**（错误 #1, #2）
   - 两个文件，替换 `ANY_PACKAGE` 为 `nullptr` 或 `FTopLevelAssetPath`

2. **修复 FConfigFile API**（错误 #3）
   - 重写 `CmdHandler.cpp` 的 DDC 配置逻辑

3. **修复 Conv_StringToFloat**（错误 #4, #5）
   - 两个文件，添加版本守卫

4. **修复 GetMaterialResource**（错误 #6）
   - 添加 EShaderPlatform 转换

5. **修复 OnPakFileMounted2**（错误 #7）
   - 改用 `GetOnPakFileMounted2()` getter

6. **修复 OnObjectSaved**（错误 #8）
   - 改用 `OnObjectPreSave`，修改回调签名

7. **修复 FHotPatcherPackageWriter 抽象类**（错误 #9）
   - 在 `.h` 和 `.cpp` 中补全缺失的纯虚函数实现
   - 这是最复杂的修改，需要仔细检查 UE 5.7 的 `ICookedPackageWriter` 接口

8. **修复 InitializeSerializationOptions**（错误 #10）
   - 改传 `ITargetPlatform*`

### Phase 2: 重新编译验证

```bash
"C:/Program Files/Epic Games/UE_5.7/Engine/Build/BatchFiles/Build.bat" FastModelEditor Win64 Development "D:/Work_00/Tech/FastNewModel/FastModel.uproject" -WaitMutex
```

### Phase 3: 处理 C4996 弃用警告

---

## 8. 关键注意事项

1. **`OnObjectSaved` → `OnObjectPreSave` 语义变化**: `OnObjectSaved` 在对象保存**之后**触发，而 `OnObjectPreSave` 在保存**之前**触发。HotPatcher 使用此回调来追踪保存的资产，需确认时机差异不影响功能。

2. **`FConfigFile` 的线程安全**: UE 5.7 的 `FConfigFile` 新增了读写锁（`ConfigFileMapLock`），公共 API 全部返回 `const` 指针。直接修改 section 不再推荐，应使用 `GConfig` 的方法。

3. **`GetMaterialResource` 的 EShaderPlatform 转换**: 需要确认 `GShaderPlatformForFeatureLevel` 数组在 UE 5.7 中的可用性，或使用 `GMaxShaderPlatform` 等替代方案。

4. **`ICookedPackageWriter` 接口扩展**: UE 5.7 对 `ICookedPackageWriter` 添加了多个新方法。`SetCooker` 的参数类型 `UE::PackageWriter::Private::ICookerInterface` 是内部类型，可能需要特殊处理。
