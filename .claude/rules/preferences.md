# 个人编码偏好（UE5）

## 语言选择
- 优先使用 C++，Blueprint 仅用于快速原型或非性能敏感的 UI 逻辑
- 核心系统、AI、物理交互必须用 C++ 实现

## 命名约定
- 类名使用 UE5 标准前缀：`U`（UObject）、`A`（AActor）、`F`（结构体）、`E`（枚举）、`I`（接口）
- 变量使用 PascalCase：`PlayerHealth`、`MaxSpeed`
- 私有成员加前缀：`bIsAlive`（bool）、`MeshComponent`
- Blueprint 可见属性统一加 `UPROPERTY` 宏注释用途

## 编辑器/IDE
- 使用 Rider for Unreal（次选）或 VS2022（主力）
- 启用 UnrealLink 插件保持编辑器同步
- 格式化工具：clang-format，配置文件放在项目根目录 `.clang-format`

## 模块结构
- 每个功能域独立 Module（如 `CombatModule`、`UIModule`）
- 避免循环依赖，依赖方向：Gameplay → Core → Engine