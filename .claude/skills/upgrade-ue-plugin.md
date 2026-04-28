# UE 插件版本升级 Skill

将一个 Unreal Engine 插件从低版本迁移到高版本的系统化流程，基于 HotPatcher (UE5.6 → UE5.7) 实际迁移经验总结。

---

## 总览

升级分 **四个大阶段、九个子阶段**：

```
阶段 A: 信息收集         阶段 B: 编译适配        阶段 C: 运行时验证      阶段 D: 记忆固化
┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐
│ 1. 确认版本号     │   │ 4. Build.cs 审查  │   │ 7. 编辑器加载     │   │ 9. 更新 CLAUDE.md │
│ 2. 头文件 Diff    │   │ 5. 源码逐模块修改 │   │ 8. 功能流程测试   │   │                  │
│ 3. 定位版本守卫   │   │ 6. 编译排错       │   │   → 发现运行时 BUG │   │                  │
└──────────────────┘   └──────────────────┘   └──────────────────┘   └──────────────────┘
```

---

## 阶段 A: 信息收集

### Step 1: 确认目标版本号

```bash
# 查注册表获取 UE 安装路径
reg query "HKLM\SOFTWARE\EpicGames\Unreal Engine\<版本>" /v InstalledDirectory

# 确认 ENGINE_MINOR_VERSION
# 源码版：读取 Engine/Source/Runtime/Core/Public/Misc/EngineVersion.h
# 安装版：读取 <UE安装目录>/Engine/Source/Runtime/Core/Public/Misc/EngineVersion.h
```

记录确认的 `ENGINE_MAJOR_VERSION` / `ENGINE_MINOR_VERSION` / `ENGINE_PATCH_VERSION`。

### Step 2: 头文件 Diff — 对比目标版本的 API 变更

**核心原则**：逐个对比插件中已使用的 Engine API 对应的头文件，确认签名是否变更。

重点对比的头文件（按优先级排序）：

| 头文件 | 涉及功能 | 关注点 |
|--------|---------|--------|
| `SavePackage.h` / `ObjectSaveUtilities.h` | 资产保存 | FSavePackageArgs 字段增减、Save 函数签名 |
| `IPackageWriter.h` / `Serialization/PackageWriter.h` | 包写入 | 虚函数增减（纯虚函数必须实现） |
| `AssetRegistryState.h` | 资产注册 | 枚举命名空间变更 |
| `ArchiveCookContext.h` | Cook 上下文 | ECookType / ECookingDLC 命名空间 |
| `IoStoreUtilities.h` | IoStore 打包 | 函数签名变更 |
| `ShaderCodeLibrary.h` | Shader 库 | 函数参数类型变更 |

对比方式：
```bash
# 在旧版和新版 Engine 之间 diff
diff "<旧版UE>/Engine/Source/.../Header.h" "<新版UE>/Engine/Source/.../Header.h"
```

### Step 3: 定位插件中所有版本守卫

在插件源码中搜索所有版本检查宏，建立完整清单：

```bash
# 搜索所有版本守卫
grep -rn "UE_VERSION_OLDER_THAN\|ENGINE_MAJOR_VERSION\|ENGINE_MINOR_VERSION" Source/
```

对每个版本守卫记录：

| 字段 | 说明 |
|------|------|
| 文件:行号 | 守卫位置 |
| 版本条件 | 如 `UE_VERSION_OLDER_THAN(5,6,0)` |
| 守卫的 API | 该守卫保护的具体 API 变更 |
| 是否需要扩展 | 目标版本是否引入了新的 API 变更 |

**分类策略**：
- 最高优先级：与目标版本相邻的守卫（如从 5.6 升 5.7，关注所有 `5,6,0` 守卫）
- 中优先级：跨多版本的守卫（如 `5,4,0`、`5,3,0`）
- 低优先级：远古版本守卫（如 `5,1,0` 及更早）

---

## 阶段 B: 编译适配

### Step 4: Build.cs 审查

检查每个模块的 `*.Build.cs`：

1. **模块依赖**：新版 Engine 是否合并/拆分了模块
   ```csharp
   // 检查 PrivateDependencyModuleNames / PublicDependencyModuleNames
   // 常见变更：模块重命名、功能拆分到新模块
   ```

2. **公共定义**：`AddPublicDefinitions` 中的条件编译宏
   ```csharp
   // 如 WITH_UE5、WITH_PACKAGE_CONTEXT 等自定义宏的版本条件
   ```

3. **SDK 路径**：第三方库路径是否仍然有效

### Step 5: 源码逐模块修改

**按模块逐个修改，从最底层（无依赖）到最上层（依赖其他模块）**：

#### 5a: Runtime 模块（最先改，无 Editor 依赖）

常见变更类型及修复模式：

| 变更类型 | 示例 | 修复模式 |
|----------|------|---------|
| 函数签名变更 | `FindObject` 参数变化 | 添加 `UE_VERSION_OLDER_THAN` 守卫 |
| 枚举命名空间 | `ECookType` → `UE::Cook::ECookType` | 条件编译切换命名空间 |
| 函数移除 | `OnObjectSaved` 委托被移除 | 用新版替代 API 或条件编译排除 |
| 参数类型变更 | `ERHIFeatureLevel::Type` → `EShaderPlatform` | 添加版本守卫，使用正确类型 |

#### 5b: Core 模块（Cook/Pack 核心逻辑）

- 虚函数接口变更：检查所有 override 的虚函数，新版是否新增纯虚函数
- Save 流程变更：`FSavePackageArgs` 字段增减
- `CookPackage` / `PackageWriter` 接口签名

#### 5c: Editor 模块（UI/Slate，最后改）

- Lambda 捕获语法：`[=]` → `[=, this]`（C++20 隐式 this 捕获警告变错误）
- Slate 控件 API 变更
- 编辑器委托签名变更

#### 5d: 版本守卫添加原则

```cpp
// 正确模式：在旧版本使用旧 API，新版本使用新 API
#if UE_VERSION_OLDER_THAN(5,7,0)
    // 旧 API
    UClass* Class = FindObject<UClass>(ANY_PACKAGE, *Name, false);
#else
    // 新 API
    UClass* Class = FindObject<UClass>(nullptr, *Name, EFindObjectFlags::None);
#endif
```

**关键原则**：
- 永远保留旧版本支持（不删除旧守卫），除非明确决定放弃
- 新增守卫使用目标版本号（如 `UE_VERSION_OLDER_THAN(5,7,0)`）
- 必须添加 `Misc/EngineVersionComparison.h` include 才能使用版本宏

### Step 6: 编译排错

编译时常见错误及修复：

| 编译错误 | 根因 | 修复 |
|----------|------|------|
| `error C2039: 'XXX' is not a member of 'YYY'` | API 被移除或重命名 | 查新版头文件找替代 API |
| `error C4996: 'XXX' was declared deprecated` | API 被标记废弃 | 可暂时忽略警告，或迁移到新 API |
| `error C2259: cannot instantiate abstract class` | 接口新增纯虚函数 | 实现新增的虚函数 |
| `error C2664: cannot convert argument` | 参数类型变更 | 添加版本守卫切换类型 |
| `error C2065: 'ANY_PACKAGE' undeclared` | 宏/常量被移除 | 新版本使用 `nullptr` 替代 |

**编译排错工作流**：
1. 编译 → 收集所有错误
2. 按错误类型分组
3. 逐组修复（同类错误通常修复模式相同）
4. 重新编译直到通过

---

## 阶段 C: 运行时验证

### Step 7: 编辑器加载测试

1. 在新版 UE 编辑器中启动项目
2. 确认插件正确加载（Output Log 无 Error）
3. 检查插件 UI 是否正常显示

### Step 8: 功能流程测试 — 运行时 BUG 排查

**关键经验**：编译通过不等于运行正确。运行时 BUG 通常比编译错误更隐蔽。

#### 运行时错误排查方法论

**第一步：收集完整错误信息**

```
错误日志中的关键信息：
1. 断言失败的文件名和行号 → 定位到 Engine 源码中的具体 check()
2. 调用栈 → 确认崩溃发生在哪个调用链中
3. 前后文日志 → 了解崩溃时在执行什么操作
```

**第二步：阅读 Engine 源码中的断言上下文**

```cpp
// 不要只看错误消息，要读断言所在函数的完整代码
// 理解：
// 1. 这个 check() 在保护什么不变量？
// 2. 导致 check 失败的条件是什么？
// 3. 什么操作路径会触发这个函数？
```

**第三步：追踪插件调用链**

```
从插件的入口函数开始，追踪到崩溃点的完整调用链：
- 记录每一层的函数名和行号
- 确认每层的参数和状态
- 找到插件的代码与 Engine 代码的交汇点
```

**第四步：分析根因的三个维度**

```
维度 1: API 行为变更
  → 新版 Engine 改变了某个函数的行为，但签名没变
  → 排查方法：对比新旧版本 Engine 中该函数的实现

维度 2: 时序/生命周期问题
  → 全局状态（flag、注册表、单例）的初始化时机变了
  → 排查方法：追踪全局变量的赋值时序，对比 CDO 构造顺序

维度 3: 线程/并发问题
  → 新版 Engine 改变了线程模型或 GC 行为
  → 排查方法：检查并行代码中的共享状态访问
```

**第五步：修复验证**

```
修复后必须回答：
1. 修复是否覆盖了所有触发路径？（不只是最明显的那一条）
2. 修复是否影响非目标场景？（会不会误杀正常功能）
3. 修复是否只在受影响的版本上生效？（版本守卫是否正确）
```

#### 典型运行时问题分类

| 问题类型 | 特征 | 排查要点 |
|----------|------|---------|
| 断言崩溃 | `check()` / `ensure()` 失败 | 读 Engine 源码，理解 check 保护的不变量 |
| 空指针崩溃 | `Access violation` | 追踪对象生命周期，检查 GC/Unload 时序 |
| API 行为变更 | 功能异常但无崩溃 | 对比新旧 Engine 中同函数的实现差异 |
| 模块加载失败 | 函数找不到 / 链接错误 | 检查模块依赖和加载顺序 |
| 全局状态不一致 | 间歇性失败 | 检查全局 flag 的设置时机和 CDO 构造顺序 |

#### 关键教训：不要过早假设影响范围

> HotPatcher 的 WorldCookPackageSplitter 崩溃，前两次修复都假设"只有 World Partition map 受影响"，
> 实际上是因为 `FWorldCookPackageSplitter` 对**所有 UWorld** 都会创建，影响的是**所有 map 包**。
>
> **教训**：在排查根因时，要追踪 Engine 代码的注册机制（`REGISTER_COOKPACKAGE_SPLITTER` 等全局注册）
> 和条件守卫（`IsRunningCookCommandlet()` 等全局 flag），理解它们的实际影响范围，
> 而不是基于表面现象缩小假设。

---

## 阶段 D: 记忆固化

### Step 9: 更新项目记忆文件

在插件的 `CLAUDE.md` 中记录：

1. **模块结构表**：模块名、类型、加载阶段、Build.cs 路径
2. **版本守卫清单**：按版本分类，标注文件位置和用途
3. **迁移任务清单**：每个 Phase 的完成状态和关键结论
4. **运行时错误记录**：
   - 错误日志原文
   - 根因分析（包含完整触发链）
   - 修复方案和影响范围
   - 修复历史（如果多次修复才解决）
5. **Engine API 参考信息**：安装路径、关键 API 头文件位置
6. **已知限制**：升级后丧失的功能
7. **待验证项**：需人工确认的测试点

---

## 常见陷阱清单

### 陷阱 1: 只看编译错误，忽略运行时行为变更

```
编译通过 ≠ 运行正确

常见场景：
- 函数签名没变，但内部行为变了
- 全局 flag 的初始化时机变了
- 对象生命周期/GC 行为变了
- 线程模型变了
```

### 陷阱 2: 版本守卫位置不当

```cpp
// 错误：守卫太窄，只保护了一部分调用
if (condition)
{
#if !UE_VERSION_OLDER_THAN(5,6,0)
    // 新 API
#else
    // 旧 API
#endif
}

// 正确：守卫覆盖完整的逻辑分支
#if !UE_VERSION_OLDER_THAN(5,6,0)
    if (condition) { /* 新 API */ }
#else
    if (condition) { /* 旧 API */ }
#endif
```

### 陷阱 3: 忽略 Engine 内部的全局注册机制

```
Engine 中有很多全局注册机制：
- REGISTER_COOKPACKAGE_SPLITTER → 全局静态对象注册
- FWorldCookPackageSplitter::RegisterCookPackageSubSplitterFactory → 条件注册
- 这些注册通常依赖 IsRunningCookCommandlet() 等全局 flag
- 如果插件改变了这些 flag 的设置时机，可能导致注册/反注册不匹配

排查方法：
1. 搜索 REGISTER_* 宏
2. 追踪 Register/Unregister 的调用条件
3. 确认这些条件在插件运行环境下的值
```

### 陷阱 4: 不理解 CDO 构造时序

```
UClass 的 CDO（Class Default Object）在引擎初始化早期就构造了，
早于任何 Commandlet 的 Main() 函数。
如果某个注册逻辑放在 CDO 构造中且依赖全局 flag，
那么在 Commandlet::Main() 中设置该 flag 是"太晚了"的。

排查方法：
1. 找到注册代码的位置（构造函数？StartupModule？全局静态？）
2. 确认该位置在 Commandlet::Main() 之前还是之后执行
3. 如果在之前，且依赖某个 flag → flag 在 Main() 中设置 = 时序错误
```

### 陷阱 5: 并行代码中的状态假设

```
插件中的 ParallelFor 可能引入竞态条件：
- 共享的 flag / 注册表不是线程安全的
- 对象可能在一个线程被 GC 回收，另一个线程还在访问
- 包的加载状态（IsFullyLoaded）可能在不同线程间不一致

排查方法：
1. 标记所有 ParallelFor 区域
2. 检查区域内对共享状态的读写
3. 确认 UE API 的线程安全性
```

---

## 工具箱

### 快速定位 Engine 源码

```bash
# 查找 Engine 安装路径
reg query "HKLM\SOFTWARE\EpicGames\Unreal Engine" /s

# 查找特定头文件
where /r "<UE安装路径>\Engine\Source" HeaderName.h

# 搜索 API 定义
grep -rn "FunctionName" "<UE安装路径>\Engine\Source\Runtime\...\Public\"
grep -rn "FunctionName" "<UE安装路径>\Engine\Source\Editor\...\Public\"
```

### 版本守卫模板

```cpp
// 包含版本比较宏
#include "Misc/EngineVersionComparison.h"

// 条件编译模式
#if UE_VERSION_OLDER_THAN(5,X,0)
    // 旧版本代码
#else
    // 新版本代码
#endif

// 运行时版本检查（不推荐，优先用编译时）
if (FEngineVersion::Current().GetMinor() < X) { /* ... */ }
```

### 插件中常见的 UE 版本变更模式

| UE 版本 | 典型变更 |
|---------|---------|
| 5.0 → 5.1 | `FAppStyle` 替代 `FEditorStyle`、`IPackageWriter::ECommitStatus` |
| 5.1 → 5.2 | Shader library 处理变更 |
| 5.2 → 5.3 | `UE::AssetRegistry::EDependencyCategory`、`OnPreSaveWorld` 签名 |
| 5.3 → 5.4 | `FArchiveCookContext` 命名空间、Lambda `[=, this]` 强制要求 |
| 5.4 → 5.5 | — （通常是小版本，变更较少） |
| 5.5 → 5.6 | `FSavePackageArgs.TargetPlatform` 移除、`SAVE_ComputeHash` 行为变更、`PreSave/PostSave` 签名 |
| 5.6 → 5.7 | `FindObject` 参数变更（`ANY_PACKAGE` 移除 → `EFindObjectFlags`）、`Conv_StringToFloat` → `Conv_StringToDouble` |
