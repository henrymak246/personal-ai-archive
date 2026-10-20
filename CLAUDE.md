# CLAUDE.md — 本仓库工作约定

这是用户的跨 AI 个人记忆档案库（详见 README.md 与 ai-integration.md）。
Claude 在本仓库工作时遵守：

1. 先读 `archive-map.json` 获取文件索引与读写标签。
2. `# AI-READONLY` 文件只读；`# AI-WRITABLE` 文件只追加、不删改已有条目。
3. 所有记录带日期（YYYY-MM-DD），遵循各文件顶部注明的追加格式。
4. 写入前 `git pull`，写入后提交并推送，提交信息用中文说明沉淀内容。
5. 本仓库公开，绝不写入密码、密钥、证件号等敏感信息。
