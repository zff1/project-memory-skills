<div align="center">

# Project Memory Skills

**给编码 Agent 使用的中文项目记忆工作流**

[快速安装](#快速安装) · [工作原理](#工作原理) · [项目记忆结构](#项目记忆结构) · [限制](#限制)

</div>

Project Memory Skills 提供两个中文 Agent Skill，帮助编码 Agent 在不同会话之间保留项目知识，并按话题加载相关上下文：

- `topic-open`：代码任务开始前，初始化空记忆骨架并加载相关项目记忆。
- `topic-save`：任务完成后，提炼决策、约束、踩坑和代码引用，并保存到项目记忆。

记忆使用项目内可读、可编辑、可 Git 管理的 Markdown 和 JSON 文件保存。不需要数据库、向量数据库、后台服务或特定模型。

## 快速安装

在需要启用项目记忆的项目根目录执行：

```bash
npx skills add zff1/project-memory-skills \
  --skill topic-open \
  --skill topic-save \
  --agent '*' \
  --yes
```

这会把两个 Skill 安装到当前项目中，并配置给本机检测到的 Agent。默认是项目级安装，不会全局修改其他项目。

只安装给某一个 Agent 时，例如 Claude Code：

```bash
npx skills add zff1/project-memory-skills \
  --skill topic-open \
  --skill topic-save \
  --agent claude-code \
  --yes
```

> [!TIP]
> 推荐项目级安装。这样 Skill 的使用约定和项目本身绑定，团队成员也能在项目配置中看到这套工作流。

查看可安装的 Skill：

```bash
npx skills add zff1/project-memory-skills --list
```

查看已安装的 Skill：

```bash
npx skills list
```

## 为什么需要项目记忆

跨会话开发时，Agent 容易丢失之前已经确认的内容：

- 模块在哪里，以及相关代码入口是什么；
- 已经做过哪些架构决策；
- 哪些方案已经被否定；
- 项目有哪些约束和编码约定；
- 过去遇到过什么坑，以及如何规避。

这两个 Skill 把工作流程变成：

```text
任务开始
  ↓
topic-open：加载相关项目和话题上下文
  ↓
Agent：在明确约束下完成任务
  ↓
任务完成 / 用户要求“记住”
  ↓
topic-save：提炼并保存长期知识
  ↓
下一次任务继续使用这些知识
```

它保存的是经过筛选的项目知识，不是所有聊天记录。

## 工作原理

### `topic-open`：读取记忆

`topic-open` 是一个“读取为主”的 Skill，代码相关任务开始前使用。它会：

1. 定位当前项目根目录；
2. 检查 `.project-memory/` 是否存在；
3. 如果不存在，创建最小的空记忆骨架；
4. 读取项目级上下文和话题索引；
5. 根据任务选择最相关的 topic；
6. 加载该 topic 的范围说明和长期记忆；
7. 只有在需要追溯历史原因时才读取会话摘要。

首次使用只会创建：

```text
.project-memory/
├── project.md
├── index.json
└── topics/
```

`topic-open` 不创建具体业务 topic，不创建 `TOPIC.md` 或 `MEMORY.md`，也不把刚刚读到的代码信息直接写入长期记忆。这样可以避免“打开记忆”本身产生隐式的业务知识变更。

### `topic-save`：写入记忆

`topic-save` 是项目业务记忆的唯一写入方。任务完成后，或用户明确要求“记住这件事”时使用。它会：

1. 确定任务涉及的 topic；
2. 检查代码、测试结果和变更，区分事实与猜测；
3. 提炼值得长期复用的知识；
4. 创建或更新 topic 的 `TOPIC.md`、`MEMORY.md`；
5. 必要时保存简洁的会话摘要；
6. 更新 `index.json` 的路由信息；
7. 合并重复内容、处理过时内容和检查冲突。

适合保存的内容包括：

- 已确认的技术决策；
- 未来任务必须遵守的约束；
- 已验证的 Bug 原因和规避方式；
- 重要文件、类、函数或服务入口；
- 跨模块依赖关系；
- 尚未解决、需要后续跟进的问题。

## 项目记忆结构

安装的 Skill 只提供工作规则；每个实际项目自己的记忆保存在项目根目录的 `.project-memory/` 中：

```text
.project-memory/
├── project.md                    # 项目级上下文和全局约束
├── index.json                    # topic 索引和路由信息
└── topics/
    └── <topic-id>/
        ├── TOPIC.md              # 话题范围、边界和代码入口
        ├── MEMORY.md             # 跨会话长期记忆
        └── sessions/              # 历史任务摘要
            └── <timestamp>-<description>.md
```

### `project.md`

保存整个项目通用的信息，例如技术栈、启动和测试命令、目录职责，以及全局约束。

### `index.json`

只负责 topic 路由，不复制大量记忆内容。初始化内容如下：

```json
{
  "version": 1,
  "topics": []
}
```

有业务 topic 后，可以记录关键词和路径：

```json
{
  "version": 1,
  "topics": [
    {
      "id": "authentication",
      "name": "登录与鉴权",
      "keywords": ["登录", "鉴权", "token", "session"],
      "path": "topics/authentication",
      "status": "active",
      "updated_at": "2026-08-05"
    }
  ]
}
```

### `TOPIC.md`

描述某个话题的范围、不包含的内容、相关代码和依赖，帮助 Agent 判断是否应该加载这个 topic。

### `MEMORY.md`

保存该话题中跨会话有复用价值的知识，建议按以下类别组织：

- 决策（Decision）；
- 约束（Constraint）；
- 踩坑（Pitfall）；
- 代码引用（Reference）；
- 项目约定（Convention）；
- 跨模块依赖（Dependency）；
- 待确认问题（Open question）。

### `sessions/`

保存简洁的任务历史，例如目标、结果、修改文件、验证方式和后续工作。默认不加载全部历史，只在需要追溯历史原因时查看。

## 设计特点

### 按话题隔离

大型项目可以拆成多个相互独立的 topic，例如：

```text
authentication
payment-api
frontend-navigation
data-pipeline
build-and-deploy
```

实现支付功能时只加载支付相关记忆，避免无关模块的历史污染上下文。

### 渐进式加载

记忆按以下顺序逐层加载：

```text
project.md
  ↓
index.json
  ↓
TOPIC.md
  ↓
MEMORY.md
  ↓
sessions/（仅在必要时）
```

这样可以在提供必要背景的同时，减少无关内容和上下文过载。

### 记忆与代码一起版本化

`.project-memory/` 位于项目目录中，可以和代码一起提交、审查和回滚。多人协作时可以通过 Git 同步，但仍需要正常处理合并冲突和错误记忆审查。

### 当前证据优先

记忆不是绝对真相。当前代码、测试结果、仓库规则和用户明确决策优先于可能过时的记忆。发现冲突时，应记录差异并修正记忆，而不是盲目服从旧内容。

### 最小侵入

项目只需要安装两个 Skill，记忆就是普通文件。无需引入常驻进程、独立数据库或复杂的 Agent 编排系统。

## 记忆写入规则

### 建议保存

- 已确认的架构和实现决策；
- 明确的项目级或模块级约束；
- 经过代码或测试验证的解决方案；
- 可复现的故障原因和规避方式；
- 未来任务需要知道的代码入口；
- 用户明确要求长期遵守的项目规则。

### 不要保存

- 完整聊天记录；
- 一次性进度，例如“改了一个按钮”；
- 未验证的模型推测；
- 临时变量、调试输出和无复用价值的细节；
- API Key、Token、Cookie、密码、私钥等秘密；
- 未脱敏的个人信息。

### 重要变更需要确认

涉及以下内容时，`topic-save` 应先展示拟保存的变更并请求用户确认：

- 替换已有架构；
- 修改 API 或数据库结构；
- 修改鉴权、权限或安全规则；
- 修改生产环境约束；
- 与已有有效记忆冲突；
- 未来实现可能依赖、但目前没有充分证据的说法。

## 一次完整使用示例

```text
1. 在项目根目录执行安装命令
2. 开始代码相关任务
3. topic-open 创建空骨架或加载已有记忆
4. Agent 阅读代码并完成任务
5. 任务完成后执行 topic-save
6. topic-save 创建或更新相关 topic
7. 下一次任务由 topic-open 加载这些知识
```

用户也可以在任务中直接提出：

```text
以后所有 API 请求都必须经过 apiClient，请记住这一点。
```

此时 Agent 应通过 `topic-save` 判断这是否是长期规则，检查冲突和证据；涉及重要架构或 API 约束时，先确认再写入。

## 限制

这两个 Skill 是 Agent 的工作规范，不是操作系统级别的强制钩子。安装后是否在每次任务开始和结束时自动触发，取决于具体 Agent 对 Skills 的加载方式。

通常应遵守：

```text
代码任务开始前：执行 topic-open
代码任务完成、准备回复前：检查是否需要 topic-save
用户说“记住”时：立即进入 topic-save 流程
```

如果某个 Agent 没有主动触发，可以在该 Agent 的项目规则文件中补充相同要求。不要把这两个 Skill 理解成会自动监听所有聊天的后台记忆服务。

## 卸载和更新

卸载：

```bash
npx skills remove topic-open topic-save
```

更新：

```bash
npx skills update topic-open topic-save
```

## 仓库结构

本仓库只发布两个 Skill：

```text
skills/
├── topic-open/SKILL.md
└── topic-save/SKILL.md
```

项目记忆不保存在本仓库中，而是保存在每个使用它的项目自己的 `.project-memory/` 目录中。
