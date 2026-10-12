# skills

面向编码智能体（coding agents）的技能仓库，包含两类内容：

- **原创 skill**：将特定技术领域的默认选型、工作流程与编码规范固化为结构化文档，由智能体在执行任务时按需加载，使不同项目、不同会话保持一致的工程路径。
- **收录 skill**：精选社区中经过实践检验的优质 skill，收录时保留上游原始结构，并注明出处与许可信息。

## 目录结构

```text
skills/
├── py-dev/               # 一个 skill 一个目录
│   ├── SKILL.md          # 入口：frontmatter + 工作流程 + 核心规则
│   └── references/       # 按需加载的详细参考文档
│       ├── setup.md
│       ├── stack.md
│       └── ...
└── LICENSE               # 仓库默认许可（MIT）
```

约定：

- **一个 skill 一个顶层目录**，目录名与 frontmatter 中的 `name` 一致。
- **`SKILL.md` 是唯一入口**，带 YAML frontmatter（`name` + `description`），正文保持精简，只放工作流程、铁律和速查表。
- **细节下沉到 `references/`**。SKILL.md 按任务场景指路，智能体只读当前任务需要的那份，避免撑爆上下文。
- **`description` 写触发条件**。说清楚"什么时候用"，包括用户点名旧库、旧工具时的兜底场景，智能体靠它决定是否加载。
- **收录的第三方 skill 保持原样**。不重写上游的 SKILL.md 与目录结构，便于跟踪上游更新；出处（仓库链接、作者）记录在下方一览表的"来源"列，上游附带独立许可时，将该 LICENSE 文件保留在对应 skill 目录内。

## Skills 一览

| Skill | 来源 | 说明 |
|---|---|---|
| [py-dev](./py-dev/) | 原创 | Python 项目的默认栈与写法规范：uv + ruff + pyright + pytest 工具链，httpx/fastapi/polars/structlog 等默认选型，以及类型、异常、并发、测试等代码要点。新建 Python 项目、编写或审查 Python 代码时使用。 |

## 收录原则

第三方 skill 满足以下条件时才会收录：

1. 解决通用问题，不绑定特定私有环境或内部系统；
2. 有清晰的工作流程与明确的触发条件，而非零散的知识片段；
3. 结构符合（或可低成本适配）SKILL.md 格式；
4. 许可允许再分发，且不与仓库整体许可冲突。

## 使用方式

Skills 是纯 Markdown，不依赖任何特定平台。把整个目录复制（或 symlink）到你的智能体 skills 目录即可：

```bash
# 例如
ln -s /path/to/skills/py-dev ~/.agents/skills/py-dev
```

支持 SKILL.md 格式的智能体（如 pi、Claude Code / Claude Agents）会根据 `description` 自动决定何时加载；不支持的平台也可以直接把文档当作规范阅读。

## License

原创内容以 [MIT](./LICENSE) 发布。收录 skill 的版权归上游作者所有，以各 skill 目录内标注的许可与出处为准。
