# 代码样式指南（UE5 C++）

## 格式化
- 缩进：4 空格（不用 Tab）
- 大括号：Allman 风格（左括号独占一行）
- 每行最大 120 字符
- `.clang-format` 基于 `Google` 风格修改，强制执行

## 头文件
- 使用 `#pragma once`，不用传统 include guard
- 包含顺序：生成的 `.generated.h` 文件必须放最后
```cpp
#pragma once
#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "MyActor.generated.h"
```

## UPROPERTY / UFUNCTION 宏
- 所有编辑器可见属性必须添加 Category 和 ToolTip：
```cpp
UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Combat",
    meta = (ToolTip = "角色最大生命值"))
float MaxHealth = 100.f;
```
- 不对外暴露的成员不加 UPROPERTY（避免垃圾回收误判）

## 内存管理
- Actor/Component 生命周期由 UE 管理，不手动 `delete`
- 非 UObject 的堆对象用 `TSharedPtr` / `TUniquePtr`
- 跨帧持有 UObject 引用必须用 `UPROPERTY` 或 `TWeakObjectPtr`

## 注释
- 公开 API 必须写 Doxygen 注释（`/** */`）
- 复杂逻辑写行内注释，说明"为什么"而非"做什么"