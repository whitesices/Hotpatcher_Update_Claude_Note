# 安全要求（UE5）

## 网络安全（多人游戏）
- **所有游戏逻辑在 Server 端验证**，客户端只做表现
- 使用 `HasAuthority()` 判断执行权限，关键逻辑加断言：
```cpp
if (!HasAuthority()) return;
```
- `UFUNCTION(Server, Reliable, WithValidation)` 必须实现 `_Validate` 函数
- 永远不信任客户端传来的伤害值、坐标、物品数量

## 输入验证
- 所有从网络/外部接收的字符串在使用前做长度和字符集校验
- 控制台命令（Exec）仅在开发/编辑器版本开放，Shipping 关闭

## 反作弊
- 敏感数值（血量、金币）存放在 `GameState`/`PlayerState`，Server Authoritative
- 移动验证使用 UE 内置 `CharacterMovementComponent` 的服务端校正机制
- 不在客户端暴露爆头/伤害倍率等参数

## 数据安全
- 玩家存档加密：使用 AES-256，密钥不硬编码（从安全存储读取）
- 不在日志中打印玩家 ID、账号信息（Shipping 构建关闭详细日志）
- 第三方 SDK（支付、社交）调用必须走后端代理，不在客户端直接请求

## 代码审查检查项
- [ ] RPC 函数是否有 `_Validate` 实现？
- [ ] 是否有裸指针跨网络传递？
- [ ] Shipping 构建是否禁用了调试命令？
- [ ] 所有 `LoadObject` / `StaticLoadClass` 的路径是否来自可信来源？