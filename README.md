# personal-ai-archive

> 跨 AI、跨设备通用的个人记忆档案库 —— 一次搭建，全平台复用。

## 这是什么

一个用纯 Markdown 组织的个人档案仓库，任何能读文本的 AI 都可以直接理解，无需格式转换。
仓库公开，AI 可通过 raw 链接免登录读取；写入通过 GitHub API + 最小权限令牌完成。

## 目录结构

| 目录 | 用途 | AI 权限约定 |
|------|------|------------|
| `identity/` | 个人身份、职业、核心标签、人生目标 | 只读（AI-READONLY） |
| `workflow/` | 工作习惯、常用工具、沟通风格、任务优先级规则 | 只读（AI-READONLY） |
| `memory/` | 历史经验、重要决策、AI 对话沉淀的个人偏好 | 可写（AI-WRITABLE，追加为主） |
| `skills/` | 已掌握技能、学习计划、待提升能力清单 | 可写（AI-WRITABLE） |
| `archive/` | 历史项目、旧文档、已完成任务归档 | 可写（AI-WRITABLE，只进不出） |

## AI 识别标签约定

每个文件首行有且仅有一个标签：

- `# AI-READONLY: <用途>` —— AI 只读取，不得修改；修改需本人明确指令。
- `# AI-WRITABLE: <用途>` —— AI 完成任务后可按文件内既定格式追加内容；不得删除或改写已有条目。

## 填写规则

- `{{双花括号}}` 是待填占位符，填好后用真实内容整体替换掉即可。
- 每条记录尽量带日期（YYYY-MM-DD），方便 AI 判断时效。
- 本仓库公开，**不要写入密码、密钥、证件号等敏感信息**。

## 版本保护与回退

- 所有修改都有 git 提交记录；main 分支已开启保护：禁止 force push、禁止删除分支。
- 回退任意文件到历史版本：

```bash
git log --oneline -- memory/experiences.md    # 找到目标提交
git checkout <commit> -- memory/experiences.md
git commit -m "回退 experiences.md 到 <commit>"
```

- 或在 GitHub 网页操作：进入文件 → History → 选择历史版本 → 复制内容恢复。

## 给 AI 的快速接入提示词（复制即用）

```text
我的个人档案库在 https://github.com/henrymak246/personal-ai-archive（公开仓库）。
请先读取 https://raw.githubusercontent.com/henrymak246/personal-ai-archive/main/archive-map.json 获取文件索引，
再按需读取各文件原始内容（raw 链接 = 索引中 raw_base + path）。
规则：AI-READONLY 文件只读；AI-WRITABLE 文件可在我要求时追加记录，禁止删改已有内容。
```

各平台详细接入方式见 [ai-integration.md](ai-integration.md)。
