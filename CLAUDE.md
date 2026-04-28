# FastNewModel

## 项目概述

FastNewModel 是一个基于 **Unreal Engine 5.7** 的数字人角色定制应用，支持实时角色换装、MetaHuman 集成、热更新内容推送等功能。

- **引擎版本**: UE 5.7
- **项目文件**: `FastModel.uproject`
- **主模块**: `FastModel`（Runtime, Default 加载阶段）
- **开发公司**: 泰金福州信息技术有限公司

## 目录结构

```
FastNewModel/
├── Source/FastModel/          # 主游戏模块
│   ├── Public/
│   │   ├── Gameplay/Core/     # 核心 Gameplay 类（Character, GameMode, PlayerController 等）
│   │   ├── FileUploader.h     # 文件上传功能
│   │   ├── SaveRenderTargetToFile.h      # RenderTarget 截图导出
│   │   └── URuntimeSkeletalMeshAsyncLoader.h  # 运行时骨骼网格异步加载
│   └── FastModel.Build.cs
├── Plugins/                   # 插件目录
│   ├── HotPatcher/            # 热更新打包插件（5 个子模块，已适配 UE5.7）
│   ├── BlueprintWebSocket/    # WebSocket 客户端
│   ├── WebSockeServer/        # WebSocket 服务端
│   ├── VaRestPlugin/          # REST API HTTP 请求
│   ├── HttpRequc65a3f23dd96V8/ # HTTP 请求插件
│   ├── FileDownloader/        # 文件下载
│   ├── FileUploader/          # 文件上传
│   ├── RuntimeImageLoader/    # 运行时图片加载
│   ├── RuntimeLoadTexture/    # 运行时贴图加载
│   ├── FrameCapture/          # 屏幕截图/录屏
│   ├── HDRIBackdrop/          # HDRI 环境光照
│   ├── SwitchLanguage_5.3/    # 多语言切换
│   ├── ColorPicker/           # 颜色选择器
│   ├── OpenBatFileHelper/     # 批处理文件助手
│   ├── CurlWrapperLibrary/    # cURL 封装库
│   └── MyBlueprintFunctionLib/ # 蓝图函数库
├── Content/                   # 资产目录（角色贴图、材质、服装、MetaHuman 等）
├── Config/                    # 项目配置
├── Docs/                      # 技术文档
│   ├── UE5.7_Architecture.md
│   ├── HotPatcher_Reproduction_Guide.md
│   ├── UE5.7_Initialization_Process.md
│   └── UE5.7_Rendering_Pipeline.md
├── Paks/                      # 打包输出
└── .claude/rules/             # Claude Code 规则文件
```

## 核心 Gameplay 类

| 类名 | 文件路径 | 说明 |
|------|----------|------|
| AFastModelCharacter | `Public/Gameplay/Core/FastModelCharacter.h` | 玩家角色 |
| AFastModelGameMode | `Public/Gameplay/Core/FastModelGameMode.h` | 游戏模式 |
| AFastModelPlayerController | `Public/Gameplay/Core/FastModelPlayerController.h` | 玩家控制器 |
| AFastModelGameState | `Public/Gameplay/Core/FastModelGameState.h` | 游戏状态 |
| AFastModelHUD | `Public/Gameplay/Core/FastModelHUD.h` | HUD |
| AFastModelPlayerState | `Public/Gameplay/Core/FastModelPlayerState.h` | 玩家状态 |
| AFastModelPlayerCameraManager | `Public/Gameplay/Core/FastModelPlayerCameraManager.h` | 摄像机管理 |
| UFastModelGI | `Public/Gameplay/Core/FastModelGI.h` | GameInstance |

## 模块依赖

FastModel 主模块依赖：Core, CoreUObject, Engine, InputCore, PakFile, Slate, SlateCore, AssetRegistry, HTTP, Json, JsonUtilities, HairStrandsCore, Niagara

## 关键功能

1. **角色定制** — MetaHuman 集成，支持体型、服装、材质实时调整
2. **热更新** — HotPatcher 插件支持不经过应用商店推送内容补丁
3. **网络通信** — WebSocket + HTTP REST API 与后端交互
4. **截图/渲染** — RenderTarget 导出、FrameCapture 屏幕捕获
5. **文件传输** — 文件上传/下载，支持运行时资源加载
6. **PixelStreaming** — Web 端实时串流

## 项目部署架构

项目存在两个独立的副本，修改源码后需双向同步：

| 项目 | 路径 | 角色 |
|------|------|------|
| FastNewModel | `D:\Work_00\Tech\FastNewModel` | 主开发副本 |
| FastModel57 | `D:\Work_00\Tech\FastModel57` | 实际运行/编译副本 |

- 两个目录**不是 symlink**，是独立副本
- 编译在 FastModel57 项目中进行
- 修改源码后必须确保 FastModel57 中的文件同步更新

## HotPatcher 热更新插件

详细的迁移记录和技术调查见 `Plugins/HotPatcher/CLAUDE.md`。

### 已知限制
- **UE5.6+ 不支持 map 包热更新 Cook** — 需使用 UE 原生打包流程或 HotChunker
- 非 map 资产（.uasset）可正常 Cook
- C4996 弃用警告（UE5.6+ API 变更）

## 构建与开发

- **IDE**: Visual Studio 2022 或 Rider for Unreal
- **解决方案**: `FastModel.sln`
- **格式化**: `.clang-format`（Google 风格修改，4 空格缩进，Allman 大括号）
- **编码规范**: 详见 `.claude/rules/` 下的规则文件
