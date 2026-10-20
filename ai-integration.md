# AI-READONLY: 跨平台 AI 接入配置说明

> 本文件说明如何把本仓库接入各主流 AI 平台。最后更新：2026-10-20

## 0. 两个基本原理

**读取**：本仓库为公开仓库，所有文件可通过 raw 链接免认证读取：

```text
https://raw.githubusercontent.com/henrymak246/personal-ai-archive/main/<文件路径>
```

文件索引（建议 AI 先读这个）：
`https://raw.githubusercontent.com/henrymak246/personal-ai-archive/main/archive-map.json`

**写入**：任何 AI / 脚本要修改档案，走 GitHub Contents API：

```text
PUT https://api.github.com/repos/henrymak246/personal-ai-archive/contents/<文件路径>
```

需要 Personal Access Token（PAT）。**强烈建议使用 fine-grained PAT：只授权本仓库、只给 Contents: Read and write 权限。**
创建入口：GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens。

---

## 1. Claude

- **claude.ai 网页 / 桌面版**：Settings → Integrations → 连接 GitHub，授权后在对话中直接引用本仓库文件即可。
- **Claude Code / Kimi CLI 等终端 Agent**：`git clone` 本仓库到本地，AI 直接读写文件并提交推送。这是写入能力最完整的方式，且每次修改都有 git 记录。

## 2. ChatGPT（含 Dots）

- **轻量方式（只读）**：Settings → Personalization → Custom instructions，粘贴 README 中的"快速接入提示词"和核心 raw 链接，ChatGPT 可按链接读取档案。
- **写入方式**：创建 Custom GPT → Configure → Actions → 配置 GitHub Contents API（PAT 放在 Action 的鉴权头中），GPT 即可按规则回写档案。
- **Dots（always-on Agent）**：给你的 Dot 设定长期目标时，把"快速接入提示词"写进目标描述，并在其应用连接器中确认 GitHub 是否可用；若可用，授权后即可持续同步偏好。Dots 的连接器目录在持续扩充，以产品内实际列表为准。

## 3. Gemini

- **Gem 自定义助手**：在 Gem 的指令中粘贴核心档案内容，或粘贴 raw 链接清单让其按需读取。
- **AI Studio / API**：把档案内容放入 system instruction；写入需通过自己的脚本调 GitHub API。

## 4. Meta Muse

Muse 通过其"连接器"体系访问外部应用（首批已公布 Asana、Notion、Slack、Shopify 等）。接入方式：

- 在 Muse 设置中查看连接器目录：
  - 若已有 **GitHub 连接器**：授权本仓库，并在你的 Muse 目标 / 偏好设置中写入"快速接入提示词"。
  - 若暂无 GitHub 连接器：可让 Muse 通过其云电脑的浏览器 / 终端访问 raw 链接读取；写入建议交给终端 Agent（见第 1 节）。
- 该产品的接入能力以 Meta 官方最新说明为准。

## 5. 任何其他 AI 的通用模式

只要该 AI 满足任一条件即可接入：

1. 能访问 URL → 用 raw 链接读取（只读）。
2. 能调 HTTP API → 用 Contents API + PAT 读写。
3. 两者都不能 → 复制粘贴档案内容进去；让它把更新内容输出为 Markdown，由你本人提交。

---

## 6. 自动更新规则（跨平台通用，复制即用）

把下面这段粘贴到各平台的自定义指令 / Agent 长期目标中：

```text
你是我的长期助手。我的个人档案库：https://github.com/henrymak246/personal-ai-archive
规则：
1. 档案中 # AI-READONLY 文件只读；# AI-WRITABLE 文件可追加，禁止删改已有条目。
2. 每次完成重要任务后，主动问我："是否把这次的经验/偏好沉淀到档案库？"
3. 经我同意后，按目标文件既定格式追加一条带日期（YYYY-MM-DD）的记录：
   - 经验教训   → memory/experiences.md
   - 重要决策   → memory/decisions.md
   - 个人偏好   → memory/preferences.md
   - 技能进展   → skills/ 下对应文件
   - 完结项目   → archive/ 归档并在 archive/README.md 登记索引
4. 若你没有写入权限，输出整理好的 Markdown 文本块，由我本人提交。
5. 追加记录前先读取目标文件最新内容，避免覆盖他人/其他 AI 的更新。
```

> 注意：消费级 AI（网页版 ChatGPT / Gemini / Muse 等）不会在每次对话后"自动"写入仓库——
> 写入动作必须由你授权触发（指令触发 / Action 调用 / Agent 执行），上面的规则就是这个触发契约。

---

## 7. 安全须知

- PAT 只授权本仓库、最小权限；一旦泄露，立即在 GitHub → Settings → Developer settings 吊销重建。
- 档案是公开仓库，**绝不写入密码、密钥、身份证、银行卡等敏感信息**。
- main 分支已开启保护（禁止 force push、禁止删除分支），所有修改可回退，操作见 README。
- 建议每月检查一次 GitHub → Settings → Applications / Integrations，清理不再使用的授权。
