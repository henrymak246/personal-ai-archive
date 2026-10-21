# AI-WRITABLE: 历史经验与教训

> 追加格式：`## YYYY-MM-DD 标题`，正文含【背景】【教训】【复用建议】三段。
> 新记录追加到文件末尾，禁止删改已有条目。

## 2026-10-20 搭建跨AI个人记忆档案库（personal-ai-archive）

【背景】从零搭建 GitHub 个人档案库，要求全平台 AI 可读、可回写、带版本保护。全程经 GitHub REST API + git credential 已存令牌完成，零手动操作。

【教训】Git Bash 里 curl -d 直接传含中文的 JSON 会因编码问题被 API 拒收（HTTP 400 Problems parsing JSON）；API 首次调用偶发失败时重试即可，但必须先打印原始响应体再判断。

【复用建议】调 GitHub API 统一用 Python urllib（JSON 强制 UTF-8）；写 Contents API 前必须先 GET 获取文件 sha；任何写入后回读 raw 链接验证。

## 2026-10-21 国产备份软件 SAP HANA backint 集成机制反推与备份/还原实测

【背景】在客户 HANA 2.0 测试机上实测三种备份方式(全备/差异/增量)与三种还原方式(恢复至指定备份、时间点还原 PITR、恢复至最新)。备份软件为国产黑方(ubackup),文档缺失,机制全靠 SSH+hdbsql 实测反推,踩坑多轮后跑通并交付报告。

【教训】①backint 插件固定读默认参数文件(/usr/sap/saphana_backup_DB_<库>_opbackint_param.txt 用于备份、saphana_recovery_<库>_opbackint_param.txt 用于恢复),BACKUP/RECOVER 语句里 USING BACKINT 括号内的路径只决定备份对象命名,不决定配置来源;②参数文件必须放在 /usr/sap/<SID>/SYS/global/hdb/backint/DB_<库>/ 下(否则 110091),不带括号的 USING BACKINT 是语法错;③恢复语法以 nameserver trace 实证为准:`RECOVER DATABASE FOR <库> UNTIL TIMESTAMP '...' CLEAR LOG USING CATALOG BACKINT`,执行前必须先 ALTER SYSTEM STOP DATABASE;④手工 hdbsql 发起的备份不注册备份软件控制台的 job,USING CATALOG BACKINT 目录选不中,会静默回退选控制台注册的最新链——恢复"成功"但数据源不是你的备份,靠探针表才识破;恢复手工备份只能走 `RECOVER DATA ... USING BACKINT('<参数文件basename>')`;⑤恢复依赖本地暂存目录(如 /hanabak/{data,cache}),缺失时报 "File not found in OSVI cache";⑥备份对象按 parent-guid lineage 组织在介质服务器,恢复参数文件的 parent-guids 必须包含备份所在 lineage,自编 lineage 会导致备份"成功"但无处可查;⑦单次全量恢复的实测性能:备份约 8.6GB/min,恢复 dedup 重组段仅 ~2.5GB/min、非重组段可达 60GB/min。

【复用建议】与备份软件(尤其国产)集成 HANA 前先做两件事:看 nameserver 的 executed_statements trace 拿实测语法,看 backint 插件日志(backint.log)反推对象命名与目录注册机制。正式备份/恢复一律从备份软件控制台发起,保证 job 注册与目录一致。任何恢复测试都用"探针表+标记行"(在每类备份之间插一行,恢复后按行集判定还原点),不要只看恢复命令返回成功。

## 2026-10-21 SAP 老系统报表授权图定案:OCR 误读教训 + 角色分配替换语义事故与 CDPOS 恢复法

【背景】客户(广药)ECC 老结构系统 DEV/QAS 双环境做 18 人权限维护。业务用微信发来一张"取消/单独授权 ZMMR001/2/3 报表权限"的截图,按图执行时发现"一直搞不定"。排查发现:OCR 把截图噪声读成了一个不存在的授权项(ZSDCXSTZ),据此在 DEV 误建角色并分配;而该老系统的角色分配 BAPI 是全量替换语义,把目标用户原有 7 个角色全部顶掉,随后的用户比较又把其参数文件(UST04)重建到只剩 1 个档案。

【教训】①微信截图 OCR 必须两次读数交叉验证,凡读出的"多余项"要怀疑是噪声;拿不准就请用户补发清晰图或原表,不要按模糊图直接动手。②老系统(老式 AGR 结构)的 BAPI_USER_ACTGROUPS_ASSIGN 是**全量替换**语义,不是标准系统的增量追加——每次调用必须"先读 AGR_USERS 当前全集,再一次性全量重传"。(本项目 QAS 曾踩过同款坑并写入铁律,DEV 侧未照搬,二次踩坑。)③PRGN_GEN_PROFILES_FOR_ROLES 的 iv_usercompare='X' 在该老系统上也是替换式:只传 1 个角色进去,会把受影响用户的 UST04 整个重建为仅含该角色档案。④角色变更历史:USH04 不记录(那是授权档案变更史),要查 CDPOS(TABNAME='AGR_USERS',TABKEY LIKE '%用户名%',OBJECTCLAS='PFCG')——每笔增删都有 VALUE 新旧值,可完整反推原始角色集。⑤"下结论前先全量实测":本次曾仅抽样 3 个用户就断言"角色建了没人分配",实际 18 人早已全部落地,结论完全错误,被用户当场纠正。

【复用建议】在老式结构 SAP 系统做任何角色/参数文件变更:动手前先 CDPOS 或 AGR_USERS 留底;分配永远走"读全集→一次全传";用户比较必须传该用户全部角色集;完成后用 SUSR_USER_AUTH_FOR_OBJ_GET 逐人实测生效值,不要只验证本次新增项(替换式比较可能顺带清掉其他授权)。
