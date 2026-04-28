# 首选工作流（UE5）

## 功能开发流程
1. 在 `Docs/Design/` 下写简短设计文档（Markdown）
2. 创建对应 C++ 类，先定义接口/虚函数
3. 在 Editor 中创建 Blueprint 子类做快速验证
4. 验证通过后将逻辑下沉回 C++
5. 写单元测试（Automation Test）
6. 提交前跑 `Full Build + Cook` 确认无错误

## Git 工作流
- 主分支：`main`（保护，禁止直接推送）
- 功能分支：`feature/task-name`
- 修复分支：`fix/issue-description`
- 大型资产（.uasset/.umap）使用 Git LFS 追踪
- Commit message 格式：`[模块] 动词 + 描述`（如 `[Combat] Add hit reaction system`）

## 资产管理
- 所有资产放 `Content/` 下，按模块分目录：`Content/Characters/`、`Content/Environment/`
- 临时测试资产放 `Content/_Dev/YourName/`，不提交主干
- 命名规则：`T_` 贴图、`SM_` 静态网格、`SK_` 骨骼网格、`BP_` Blueprint

## 构建与部署
- 使用 UAT 脚本自动化打包：`RunUAT.bat BuildCookRun`
- CI 环境（如 Jenkins/GitHub Actions）跑 `Shipping` 配置验证