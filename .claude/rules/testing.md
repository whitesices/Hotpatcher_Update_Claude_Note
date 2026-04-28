# 测试约定（UE5）

## 测试框架
- 使用 UE5 内置 **Automation Testing Framework**（`FAutomationTestBase`）
- 复杂系统集成测试使用 **Gauntlet Automation Framework**

## 测试分类
| 类型 | 用途 | 路径 |
|------|------|------|
| Unit Test | 纯逻辑、无 World | `Source/Tests/Unit/` |
| Functional Test | 需要 Actor/World | `Content/Tests/` (Blueprint) |
| Performance Test | 帧率、内存基准 | `Source/Tests/Perf/` |

## 命名规范
```cpp
// 格式：[模块].[类名].[测试场景]
IMPLEMENT_SIMPLE_AUTOMATION_TEST(
    FHealthComponent_TakeDamage_ShouldReduceHealth,
    "Combat.HealthComponent.TakeDamage.ShouldReduceHealth",
    EAutomationTestFlags::ApplicationContextMask |
    EAutomationTestFlags::ProductFilter
)
```

## 覆盖要求
- 所有 public C++ 方法必须有对应单元测试
- 核心 Gameplay 系统（战斗、存档、AI）要求功能测试覆盖
- 每次 PR 合并前必须通过本地 `Run All Tests`

## 运行方式
```bash
# 命令行运行所有测试
UE4Editor-Cmd.exe MyProject -ExecCmds="Automation RunTests Combat" -Unattended -NullRHI
```

## Mock / Stub
- 用接口（`UINTERFACE`）隔离外部依赖，测试时注入假实现
- 避免在测试中启动完整 GameMode，用最小 World 配置