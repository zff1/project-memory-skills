# Project Memory Skills

两个可移植的 Agent Skill，用于给编码 Agent 增加项目本地、按话题隔离的记忆：

- `topic-open` —— 在开始工作前加载最相关的项目和话题上下文；
- `topic-save` —— 在工作完成后提炼长期决策、约束、踩坑和代码引用。

职责边界：`topic-open` 可以在项目第一次使用时创建空的 `.project-memory/` 骨架，但不创建业务 topic，也不写入业务记忆；所有业务记忆的创建和写入都由 `topic-save` 负责。

这两个 Skill 遵循参考文章中的轻量级记忆思路：使用可读的 Markdown 文件、按话题隔离、按需渐进加载，并保存会话摘要。不需要 CLI、数据库、向量数据库、后台服务，也不绑定具体模型运行时。

## 安装

在目标项目目录中，从 GitHub 仓库安装两个 Skill：

```bash
npx skills add <owner>/<repo> \
  --skill topic-open \
  --skill topic-save \
  --agent '*'
```

将 `<owner>/<repo>` 替换成发布后的 GitHub 仓库。只安装给某一个 Agent 时，把 `--agent '*'` 替换成具体名称，例如 `claude-code`、`cursor` 或 `codex`。

默认安装范围是当前项目，因此 Skill 会对该项目生效，也可以和项目一起提交到 Git。只有明确希望全局安装时才使用 `-g`。

## 项目记忆

安装后，两个 Skill 使用以下项目本地目录结构：

```text
.project-memory/
├── project.md                    # 项目级上下文
├── index.json                    # 话题索引
└── topics/
    └── <topic-id>/
        ├── TOPIC.md              # 话题范围和代码入口
        ├── MEMORY.md             # 长期记忆
        └── sessions/              # 会话摘要
```

第一次运行 `topic-open` 时，只要当前任务是代码相关任务且项目中没有记忆目录，它会初始化最少的根目录文件，但不会创建具体业务 topic 或写入业务知识。`topic-save` 只有在存在值得长期保存的知识时，才创建或更新 topic。

## 设计规则

- 记忆保存在项目本地，适合使用 Git 管理。
- `MEMORY.md` 保存长期知识；`sessions/` 保存简洁的历史记录。
- 只加载相关 topic，默认不注入完整历史归档。
- 当前代码、仓库规则和用户明确决策优先于过时记忆。
- 绝不保存秘密、凭证、原始聊天记录和未经验证的猜测。
- 高影响或存在冲突的记忆变更必须经过用户确认。

## 仓库结构

```text
skills/
├── topic-open/SKILL.md
└── topic-save/SKILL.md
```

本仓库有意只包含两个 Skill。分发由 Agent Skills 生态及其 `npx skills add` 命令负责。
