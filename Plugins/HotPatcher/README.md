# HotPatcher UE5.6 分支

> 本版本基于 [HotPatcher](https://github.com/hxhb/HotPatcher) 适配 **Unreal Engine 5.6.x**（已测试 5.6.1）
>
> **维护者：泰金（福州）信息技术有限公司**

---

## 简介

HotPatcher 是 Unreal Engine 中涵盖热更新、构建提升、包体优化、资源审计等全流程的资源管理框架。

本分支针对 **UE5.6** 进行了适配和优化，确保在最新版本的 Unreal Engine 中稳定运行。

### 核心功能

- **热更新管理**：管理热更版本和资源打包，追踪工程版本的原始资源变动
- **版本控制**：支持资源版本管理、版本间的差异对比和打包
- **多平台支持**：支持导出多平台的基础包信息，支持 Cook 和打包多平台的 Patch
- **包体优化**：提供多种开箱即用的包体优化方案
- **命令行支持**：全功能的 Commandlet 支持，可与 CI/CD 平台无缝集成

---

## 版本要求

| 项目 | 要求 |
|------|------|
| Unreal Engine | 5.6.x（推荐 5.6.1） |
| Visual Studio | VS2019 或更高版本 |
| Windows | Windows 10/11 64-bit |

---

## 安装步骤

### 1. 复制插件到项目

将插件目录复制到项目的 `Plugins` 文件夹下：

- 源路径：`<当前项目>/Plugins/HotPatcher`
- 目标路径：`<你的项目>/Plugins/HotPatcher`

只需要复制 `Source` 文件夹即可。

### 2. 配置 .uproject 文件

确保项目的 `.uproject` 文件中包含 `HotPatcher` 插件引用：

1. 打开编辑器 -> `Edit` -> `Plugins`
2. 找到 `HotPatcher`
3. 勾选启用并点击「Enabled」

确保项目编译时未勾选 `Disabled`。

### 3. 生成 Visual Studio 项目文件

在项目目录下：

- 右键 `.uproject` -> `Generate Visual Studio project files`

### 4. 编译项目

#### 编辑器目标（推荐首次）

```
Build.bat <ProjectName>Editor Win64 Development -Project="<YourProject>.uproject" -WaitMutex -NoUBA -MaxParallelActions=4
```

#### 游戏目标

```
Build.bat <ProjectName> Win64 Development -Project="<YourProject>.uproject" -WaitMutex -NoUBA -MaxParallelActions=4
```

### 5. 验证安装

启动编辑器后检查：

1. 菜单/窗口可打开 `HotPatcher`
2. 无报错信息提示
3. 可访问 HotPatcher 资源管理器

---

## 快速开始

### 创建首个补丁包

1. 修改资源并保存项目
2. 在 HotPatcher 预设中加载 `ok.json` 配置
3. 设置版本号 `VersionId`（如 `1.0.0.1`）
4. 点击 `Diff` -> `GeneratePatch`
5. 生成的补丁文件位于配置的输出目录

### 推荐配置（基于 `ok.json`）

关键配置项：

```json
{
    "bByBaseVersion": true,
    "baseVersion": {
        "filePath": "J:/UEPJ/hotpacher/update/1.0.0.0/1.0.0.0_Release.json"
    },
    "bCookPatchAssets": true,
    "serializeAssetRegistryOptions": {
        "bSerializeAssetRegistry": true,
        "bSerializeAssetRegistryManifest": true
    },
    "assetScanConfig": {
        "assetRegistryDependencyTypes": ["All"],
        "bForceSkipContent": false,
        "forceSkipContentRules": []
    },
    "savePath": {
        "path": "J:/UEPJ/hotpacher/update"
    },
    "pakTargetPlatforms": ["Windows"]
}
```

> 注意：`assetRegistryDependencyTypes=All` 和 `SerializeAssetRegistry=true` 对加载新图/子关卡非常重要

---

## 已知问题

### 编译警告

- 可能出现 `C4996` 警告（UE5.6 API 弃用通知）
- 这些警告目前不影响功能正常使用

### Python 脚本兼容性问题

- 部分 Python 脚本可能存在 `EMatchRule` 或 `MatchRule` 相关问题

### UI 资源打包

- 确保只打包需要更新的 UMG/UI 资源
- 资源路径使用正确格式（如 `/All/...` 而非 `/Game/...`）

---

## 常见问题排查

### A. 编译失败

如遇到 `AutomationUtils/GaGauntlet/NU190x` 相关错误，有 4 种解决方式（只编译项目目标）。

### B. 清理并重新编译

- 删除 `Binaries/Intermediate` 文件夹
- 重新生成项目文件
- 重新编译

### C. 查看编译日志

- 日志位置：`C:\Users\<User>\AppData\Local\UnrealBuildTool\Log.txt`
- 搜索 `error Cxxxx` 定位错误

---

## 项目结构

```
HotPatcher/
├── Source/
│   ├── HotPatcherEditor/     # 编辑器模块
│   ├── HotPatcherCore/       # 核心逻辑模块
│   ├── HotPatcherRuntime/   # 运行时模块
│   ├── BinariesPatchFeature/# 二进制补丁功能
│   └── CmdHandler/           # 命令行处理
├── Binaries/                 # 预编译二进制文件
├── Intermediate/             # 编译中间文件
├── HotPatcher.uplugin        # 插件描述文件
└── README_*.md               # 文档
```

---

## 相关文档

- [UE5.6 使用说明](README_UE56_Usage_CN.md)
- [补丁工作流程](README_UE56_PatchWorkflow_CN.md)
- [补丁服务器技术规格](README_PatchServer_TechSpec_CN.md)

---

## 作者

**泰金（福州）信息技术有限公司**

本 UE5.6 分支由泰金（福州）信息技术有限公司维护和适配。

- 电话：15659933310
- 微信：15659933310（同电话）

---

## 致谢

- 原始项目：[hxhb/HotPatcher](https://github.com/hxhb/HotPatcher)
- 社区支持：QQ 群 958363331

---

## 开源协议

本软件遵循原始 HotPatcher 开源协议：允许在商业项目中免费使用功能，但不允许任何第三方基于该插件进行任何形式的二次收费。
