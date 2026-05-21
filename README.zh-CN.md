# openspec-schemas

[English](./README.md) · [繁體中文](./README.zh-TW.md)

社区贡献的 [OpenSpec](https://github.com/Fission-AI/OpenSpec) schema 集合。每个 schema 是一个独立的捆绑包，复制到项目的 `openspec/schemas/` 目录后，可通过 `--schema <name>` 按变更选择使用。

## 本仓库中的桥接

| 桥接 | 用途 | 状态 |
|------|------|------|
| [`superpowers-bridge`](./superpowers-bridge/) | 将 OpenSpec 的工件治理与 [obra/superpowers](https://github.com/obra/superpowers) 执行技能（头脑风暴、编写计划、基于子代理的 TDD、代码审查、收尾）进行桥接。新增了一个以证据为先的 `retrospective` 工件，填补了 Superpowers 原生未覆盖的空白。 | v1 |

## 为什么单独建仓库？

[OpenSpec PR #970](https://github.com/Fission-AI/OpenSpec/pull/970) 最初提议将 `sdd-plus-superpowers` 作为内置 schema。经维护者审查后，该集成移至社区仓库——与 [github/spec-kit 的社区扩展目录](https://speckit-community.github.io/extensions/) 模式相同，将第三方工具集成隔离在核心之外。

好处：
- OpenSpec 核心无需承担 Superpowers 的发布节奏
- 桥接可以独立迭代
- 其他社区 schema 可以作为同级目录加入本仓库

## 安装

每个桥接目录都有自己的 `README.md`，其中包含可一键复制粘贴的 Claude Code 提示词用于快速安装，同时也提供手动 bash 方式。参见 [`superpowers-bridge/README.md#install`](./superpowers-bridge/README.md#install)。

## 路线图

已规划内容见 [`docs/roadmap.md`](./docs/roadmap.md)。

## 许可证

MIT — 详见 [LICENSE](./LICENSE)。
