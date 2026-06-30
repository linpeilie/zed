# Zed 二次开发文档

这一组中文文档面向基于当前仓库做二次开发的人。它不是对
`docs/src` 用户文档的翻译，而是补充一层代码导览、开发路径和验证清单。

## 阅读顺序

1. [资料基线与可信边界](./00-source-basis.md)
2. [构建、运行与本地环境](./01-build-and-run.md)
3. [仓库结构与模块地图](./02-repo-map.md)
4. [启动流程与应用状态](./03-startup-and-app-state.md)
5. [GPUI 与界面开发](./04-gpui-ui.md)
6. [编辑器、项目、语言与 LSP](./05-editor-project-language-lsp.md)
7. [设置、Action 与快捷键](./06-settings-actions-keymaps.md)
8. [扩展系统](./07-extensions.md)
9. [协作、云服务与 AI 模块](./08-collab-ai-cloud.md)
10. [测试、调试与质量门禁](./09-testing-debugging-quality.md)
11. [二次开发工作流](./10-secondary-development-playbook.md)
12. [主界面设计与实现分析](./11-main-interface-design-and-implementation.md)
13. [主编辑器设计与实现分析](./12-main-editor-design-and-implementation.md)
14. [AI 设计与实现分析](./13-ai-design-and-implementation.md)

## 使用方式

- 先读构建和仓库地图，确认你的开发环境和目标模块。
- 改 UI 或主程序行为时，重点看启动流程、GPUI、设置和 action。
- 改语言支持或编辑器行为时，重点看主编辑器、项目、语言/LSP 和扩展系统。
- 改协作、AI、云端能力时，先读协作/云服务与 AI 模块，再读 AI 设计与实现分析；不要默认本地可完全替代线上服务。
- 每次修改后按测试文档选最小但有效的验证范围。

## 文档状态

这些文档依据当前工作区文件编写。当前工作区存在 `.codegraph/` 索引；涉及主界面
和主编辑器、AI 分析时已先用 CodeGraph 定位，再用本地源码核对关键实现。没有逐行审计
全部 236 个 crate；未验证的细节不会写成确定结论。需要深入某个具体功能时，应从
对应的“依据文件”继续追源码。
