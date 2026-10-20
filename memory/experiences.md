# AI-WRITABLE: 历史经验与教训

> 追加格式：`## YYYY-MM-DD 标题`，正文含【背景】【教训】【复用建议】三段。
> 新记录追加到文件末尾，禁止删改已有条目。

## 2026-10-20 搭建跨AI个人记忆档案库（personal-ai-archive）

【背景】从零搭建 GitHub 个人档案库，要求全平台 AI 可读、可回写、带版本保护。全程经 GitHub REST API + git credential 已存令牌完成，零手动操作。

【教训】Git Bash 里 curl -d 直接传含中文的 JSON 会因编码问题被 API 拒收（HTTP 400 Problems parsing JSON）；API 首次调用偶发失败时重试即可，但必须先打印原始响应体再判断。

【复用建议】调 GitHub API 统一用 Python urllib（JSON 强制 UTF-8）；写 Contents API 前必须先 GET 获取文件 sha；任何写入后回读 raw 链接验证。
