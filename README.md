# Project Memory Skills

两个中文 Agent Skill，用于让编码 Agent 在不同会话之间持续记住项目知识，并且按照话题隔离上下文：

- `topic-open`：在代码任务开始前，找到并加载当前任务相关的项目记忆。
- `topic-save`：在任务完成后，从代码、测试、变更和对话中提炼值得长期保留的知识，并保存到项目记忆中。

这两个 Skill 的设计来自“Markdown 即知识”的思路：记忆保存在项目目录中，内容可读、可编辑、可 Git 管理，不依赖数据库、向量数据库、后台服务或特定模型。

## 快速安装

在你想启用项目记忆的项目根目录中执行：

```bash
npx skills add zff1/project-memory-skills \
  --skill topic-open \
  --skill topic-save \
  --agent '*' \
  --yes
```

这条命令会从 GitHub 仓库下载并安装两个 Skill。

说明：

- `zff1/project-memory-skills` 是本项目的 GitHub 仓库地址。
- `--skill topic-open` 和 `--skill topic-save` 表示只安装这两个 Skill。
- `--agent '*'` 表示安装到当前机器上检测到的所有支持的 Agent。
- `--yes` 表示跳过安装确认。
- 不加 `-g` 时，默认是“当前项目级安装”，只对当前项目生效。

如果你只使用某一个 Agent，可以指定它，避免安装到其他 Agent：

```bash
# Claude Code
npx skills add zff1/project-memory-skills \
  --skill topic-open \
  --skill topic-save \
  --agent claude-code \
  --yes

# Cursor
npx skills add zff1/project-memory-skills \
  --skill topic-open \
  --skill topic-save \
  --agent cursor \
  --yes

# Codex
npx skills add zff1/project-memory-skills \
  --skill topic-open \
  --skill topic-save \
  --agent codex \
  --yes
```

如果希望全局安装，让所有项目都能发现这两个 Skill，可以增加 `-g`：

```bash
npx skills add zff1/project-memory-skills \
  --skill topic-open \
  --skill topic-save \
  --agent '*' \
  --global \
  --yes
```

一般建议使用项目级安装，便于团队成员知道这个项目启用了哪些工作约定；如果你希望以后所有项目都默认具备这两个 Skill，再使用全局安装。

## 安装后如何确认

查看仓库中有哪些可安装的 Skill：

```bash
npx skills add zff1/project-memory-skills --list
```

查看当前已安装的 Skill：

```bash
npx skills list
```

如果 Skill 已正确安装，应该能看到：

```text
topic-open
topic-save
```

## 这两个 Skill 解决什么问题

普通的 AI 编程会话通常是彼此独立的：

```text
第一次会话：讨论支付模块的架构和历史问题
第二次会话：继续修改支付模块

Agent 可能不知道：
- 支付模块在哪里；
- 之前确定了哪些架构决策；
- 哪些方案已经被否决；
- 之前踩过什么坑；
- 项目有哪些不能违反的约束。
```

安装这两个 Skill 后，工作流程变成：

```text
任务开始
  ↓
topic-open 读取相关话题记忆
  ↓
Agent 基于项目上下文完成任务
  ↓
任务结束或产生重要决策
  ↓
topic-save 提炼并保存长期知识
  ↓
下一次任务继续使用这些知识
```

它保存的不是所有聊天记录，而是经过筛选的项目知识，例如：

- 已确认的技术决策；
- 未来任务必须遵守的约束；
- 真实验证过的踩坑和规避方式；
- 重要的代码入口、文件路径和符号；
- 跨模块依赖关系；
- 尚未解决、但需要后续继续跟进的问题。

## 两个 Skill 的职责

### `topic-open`：任务开始时读取记忆

`topic-open` 是“读取为主”的 Skill，负责在开始代码相关任务前准备上下文。

它会：

1. 定位当前项目根目录；
2. 检查项目是否有 `.project-memory/`；
3. 如果没有，创建最小的空记忆骨架；
4. 读取项目级上下文；
5. 根据当前任务选择相关 topic；
6. 加载相关 topic 的长期记忆；
7. 只有在确实需要时才查看历史会话；
8. 区分已验证事实、历史记录和待确认事项。

第一次使用时，它只创建空骨架：

```text
.project-memory/
├── project.md
├── index.json
└── topics/
```

它不会在任务开始时直接创建 `authentication`、`payment` 等业务 topic，也不会把刚刚读到的代码信息直接写入长期记忆。

这样设计是为了避免“打开记忆”本身产生隐式修改，也避免 Agent 在还没有理解项目时，把猜测写入长期记忆。

### `topic-save`：任务完成时写入记忆

`topic-save` 是项目业务记忆的唯一写入方，负责：

1. 判断当前任务涉及哪个 topic；
2. 从任务结果、代码、测试和 diff 中提炼长期知识；
3. 创建新 topic（如果这个领域会持续存在）；
4. 创建或更新 `TOPIC.md` 和 `MEMORY.md`；
5. 保存必要的会话摘要；
6. 更新 `index.json` 的路由信息；
7. 合并重复内容；
8. 标记过时内容；
9. 检查冲突；
10. 对高影响变更请求用户确认。

如果用户在对话中明确说“记住这件事”，也应该进入 `topic-save` 流程，而不必等到整个任务结束。

职责边界可以概括为：

```text
topic-open = 创建空骨架 + 读取已有记忆
              不创建业务 topic，不写业务知识

topic-save = 提炼知识 + 创建 topic + 写入和整理记忆
```

## 项目记忆目录结构

安装 Skill 后，记忆会保存在当前项目自己的 `.project-memory/` 目录中：

```text
.project-memory/
├── project.md                    # 项目级上下文和全局约束
├── index.json                    # topic 索引和路由信息
└── topics/
    └── <topic-id>/
        ├── TOPIC.md              # 话题范围、边界、代码入口
        ├── MEMORY.md             # 跨会话长期记忆
        └── sessions/              # 历史任务摘要
            └── <timestamp>-<description>.md
```

### `project.md`

保存整个项目都适用的基础信息，例如：

- 技术栈；
- 项目启动、测试、构建命令；
- 顶层目录职责；
- 全局编码约定；
- 全局安全或架构约束。

它不适合保存某个单独模块的细节。模块细节应该放到对应 topic 的 `MEMORY.md` 中。

### `index.json`

这是一个轻量的 topic 路由索引，用来帮助 Agent 判断当前任务应该打开哪些 topic。初始化时内容很简单：

```json
{
  "version": 1,
  "topics": []
}
```

当项目中有业务 topic 后，可以变成：

```json
{
  "version": 1,
  "topics": [
    {
      "id": "authentication",
      "name": "登录与鉴权",
      "keywords": ["登录", "鉴权", "token", "session", "permission"],
      "path": "topics/authentication",
      "status": "active",
      "updated_at": "2026-08-05"
    }
  ]
}
```

`index.json` 只负责路由，不应该复制 `MEMORY.md` 中的大量内容。

### `TOPIC.md`

说明某个 topic 的边界和定位，例如：

```markdown
# 登录与鉴权

## 负责范围

- 登录流程
- Token 刷新
- 会话状态
- 权限判断

## 不负责

- 商品业务逻辑
- 页面视觉样式

## 相关代码

- src/services/auth.ts
- src/stores/session.ts
- src/middleware/permission.ts
```

### `MEMORY.md`

保存这个话题中跨会话有复用价值的知识，例如：

```markdown
# 登录与鉴权记忆

## 已确认的决策

### Token 刷新统一放在请求层

- 状态：verified
- 详情：业务组件不能自行刷新或拼接 Token。
- 证据：`src/services/http.ts`
- 更新时间：2026-08-05

## 约束

- 登录失效必须和普通业务错误区分处理。

## 已知踩坑

- 并发请求必须共享同一次 Token 刷新操作。

## 代码引用

- 请求封装：`src/services/http.ts`
- 登录状态：`src/stores/auth.ts`

## 待确认问题

- 是否需要支持多账号同时登录？
```

### `sessions/`

保存简洁的任务历史，而不是完整聊天记录。适合记录：

- 某次任务解决了什么问题；
- 修改了哪些文件；
- 使用了什么验证方式；
- 哪些工作还没有完成；
- 当时做了什么重要决策。

历史会话默认不全部加载，只有在需要追溯某个历史原因时才读取。

## 推荐的使用流程

### 第一次在新项目中使用

```text
1. 在项目根目录安装两个 Skill
2. 开始一个代码相关任务
3. Agent 执行 topic-open
4. topic-open 创建空的 .project-memory/ 骨架
5. Agent 阅读代码并完成任务
6. Agent 执行 topic-save
7. topic-save 创建合适的 topic 并保存第一批长期记忆
```

### 继续已有模块

```text
1. 开始任务，例如“继续优化登录过期后的请求重试”
2. topic-open 读取 project.md 和 index.json
3. 根据 authentication topic 的关键词定位话题
4. 读取 authentication/TOPIC.md 和 MEMORY.md
5. Agent 基于已有决策和踩坑继续工作
6. 完成后由 topic-save 更新记忆
```

### 对话中明确要求记住

当你说：

```text
以后所有 API 请求都必须经过 apiClient，请记住这一点。
```

Agent 应该：

1. 判断这是否是长期有效的项目规则；
2. 判断属于哪个 topic；
3. 检查是否和已有记忆冲突；
4. 记录内容、类型、状态和证据；
5. 对重要架构、API、安全规则向你确认；
6. 通过 `topic-save` 写入对应记忆。

## 什么内容会被保存

### 适合保存

- 已确认的架构决策；
- 明确的编码约束；
- 经过测试验证的解决方案；
- 真实复现过的 Bug 原因和规避方式；
- 重要文件、类、函数或服务入口；
- 跨模块的依赖关系；
- 用户明确要求长期遵守的项目规则；
- 尚未解决但后续需要继续跟进的问题。

### 不应该保存

- 完整聊天记录；
- “今天改了一个按钮”这类一次性进度；
- 未验证的模型猜测；
- 临时变量和一次性调试输出；
- 已经没有价值的重复信息；
- API Key、Token、Cookie、密码、私钥等秘密；
- 未脱敏的个人信息。

## 冲突和重要变更

以下内容写入前应该要求用户确认：

- 替换已有架构；
- 修改 API 约定；
- 修改数据库结构；
- 修改鉴权、权限或安全规则；
- 修改生产环境约束；
- 与当前有效记忆冲突；
- 未来实现可能依赖、但目前还没有证据的说法。

理想的确认方式是展示一个简短提案：

```text
准备更新 authentication topic：

旧记忆：Token 由页面组件自行处理
新记忆：Token 统一由请求层处理
证据：src/services/http.ts
影响：后续所有 API 请求实现

是否确认写入？
```

普通的、低风险且有明确证据的代码引用或踩坑，可以直接保存，但仍然应该检查重复和最终 diff。

## 重要限制：Skill 不等于强制钩子

这两个 Skill 是给 Agent 的工作规范，不是操作系统级别的后台服务。安装后能否在每次任务开始和结束时自动执行，取决于具体 Agent 对 Skills 的加载和触发方式。

通常应遵守：

```text
涉及代码的任务开始前：执行 topic-open
代码任务完成、准备最终回复前：检查是否需要执行 topic-save
用户明确说“记住”时：立即进入 topic-save 流程
```

如果某个 Agent 没有主动触发 Skill，可以在该 Agent 的项目规则文件中补充类似要求：

```text
所有代码相关任务开始前，先读取并遵循 topic-open Skill；
所有代码任务完成后，检查并遵循 topic-save Skill。
```

不要因此把这两个 Skill 当成完全自动的记忆系统。它们提供的是一套跨 Agent 可复用的项目记忆工作协议。

## 与参考文章中设计优点的对应关系

这两个 Skill 参考了文章中“自进化记忆”的公开设计，但不是对文章内部工具 `@ali/ai-coding-assistant` 的源码复刻。下面只列出当前版本实际支持或能够明确做到的部分：

| 文章中的设计点 | 当前支持情况 | 说明 |
|---|---|---|
| 按话题隔离，避免无关记忆互相污染 | 已支持 | 使用 `topics/<topic-id>/` 独立保存每个话题的 `TOPIC.md`、`MEMORY.md` 和会话摘要。 |
| 分层、按需加载，控制上下文大小 | 已支持 | 按 `project.md → index.json → TOPIC.md → MEMORY.md → sessions/` 的顺序渐进加载，历史会话默认不加载。 |
| Markdown 即知识，内容可读、可编辑 | 已支持 | 记忆使用 Markdown 保存，不依赖专用数据库或不可读的内部格式。 |
| 随项目一起 Git 版本化 | 已支持 | `.project-memory/` 位于项目目录中，可以和代码一起提交、审查、回滚。多人同步可以通过 Git 实现，但仍需要正常处理合并冲突。 |
| 只保存有价值的知识 | 已支持 | `topic-save` 明确排除原始聊天、一次性进度、临时调试信息、重复内容和未验证猜测。 |
| 记忆持续进化 | 已支持，但依赖 Agent 执行 | `topic-save` 会合并重复内容、更新过时内容、记录新决策和踩坑；它不是后台自动监听服务。 |
| 最小侵入，不改变原有开发流程 | 已支持 | 只有两个 Skill，项目记忆是普通文件，不需要 CLI、数据库、向量数据库或常驻服务。 |
| 大变更需要人工确认 | 已支持 | 涉及架构、API、数据、安全、生产约束或现有记忆冲突时，`topic-save` 要求先展示提案并获得确认。 |
| 新任务开始自动打开、任务结束自动保存 | 部分支持 | Skill 定义了这套工作流程，但是否每次自动触发取决于具体 Agent；Skills 本身不是强制生命周期钩子。 |
| 多人研发中的记忆同步 | 条件支持 | 将记忆文件提交到 Git 后可以随项目同步，但权限、合并冲突、错误记忆审查仍由团队负责。 |

因此，当前版本真正提供的是一套“项目记忆工作协议”：它把文章中的话题隔离、渐进加载、Markdown 持久化和经验提炼落成两个可安装 Skill，但不承诺后台自动运行、跨 Agent 一致触发或自动解决团队协作冲突。

### 按话题隔离

一个大型项目可以拆成多个 topic，例如：

```text
authentication
payment-api
frontend-navigation
data-pipeline
build-and-deploy
```

实现支付功能时只加载支付相关记忆，不把鉴权、前端和部署的全部历史都塞进上下文。

### 渐进式加载

加载顺序是：

```text
project.md
  ↓
index.json
  ↓
TOPIC.md
  ↓
MEMORY.md
  ↓
sessions/（只有必要时）
```

这样可以避免上下文过长，也减少无关记忆对当前任务的干扰。

### 记忆与代码一起版本化

`.project-memory/` 位于项目目录中，可以和代码一起使用 Git 管理。你可以查看谁在什么时候增加了哪条决策，也可以在错误更新后回滚。

### 当前证据优先

记忆不是绝对真相。当前代码、测试结果、仓库规则和用户明确决策，优先于可能已经过时的记忆。发现冲突时，Agent 应该记录并修正，而不是盲目服从旧记忆。

## 常见问题

### 是否需要 CLI、数据库或向量数据库？

不需要。这个项目只提供两个 Skill，记忆由 Agent 通过文件工具读写。Markdown 和 JSON 足以支持第一版；只有当项目记忆规模很大时，才需要考虑额外检索方案。

### `.project-memory/` 是否会被安装命令下载？

不会。安装命令只安装 `topic-open` 和 `topic-save`。`.project-memory/` 是每个具体项目自己的运行数据，由 `topic-open` 在第一次代码任务中创建。

### 可以手动编辑记忆吗？

可以，而且这是设计目标之一。记忆是普通 Markdown 和 JSON，建议通过 Git diff 审查重要修改。

### 可以把 `.project-memory/` 提交到 Git 吗？

可以。项目记忆的目标就是和项目一起版本化。但提交前必须确认其中没有 Token、密码、Cookie、内部地址或其他敏感信息。

### 一个项目应该创建多少 topic？

不要为每个文件、工单或会话创建 topic。只有会被反复使用的模块、业务领域或基础设施才适合成为 topic。

### 安装后会不会自动记住所有聊天？

不会。系统只应该保存经过提炼、有长期价值的项目知识，不保存完整聊天记录。是否自动触发 open/save 取决于具体 Agent。

## 卸载和更新

卸载两个 Skill：

```bash
npx skills remove topic-open topic-save
```

从远程仓库更新已安装的 Skill：

```bash
npx skills update topic-open topic-save
```

## 仓库结构

本仓库有意只包含两个 Skill：

```text
skills/
├── topic-open/SKILL.md
└── topic-save/SKILL.md
```

项目记忆不存放在这个 Skill 仓库中，而是存放在每个使用它的项目自己的 `.project-memory/` 目录中。
