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

## 2026-10-22 腾讯云两节点 HANA scale-out 搭建:介质断点救援 + 127.0.1.1 回环大坑 + hdbmodify 三连坑

【背景】在腾讯云两台 CVM(每机 4C/32G+独立 200G 数据盘)上从零搭 HANA 2.0 SPS08.9 双节点 scale-out(共享 /hana/shared 走节点 NFS,数据/日志各机本地)。当天完成从"介质截断"到"双主机全绿、租户每机一个 indexserver"的全链路,踩坑密度极高,全部留档。

【教训】①介质文件先验完整性再用:SAPCAR 解压中途 error 5 报"读错误"是 SAR 被截断的典型症状,解压前应 md5 双端比对、SAPCAR -tvf 看成员表(截断文件成员表也截断,对比未截断版本可算出应解出多少成员)。②介质断点救援组合拳:源机器有 rsync 且可直通目标机公网时,把目标机 SSH 私钥临时放源机,`rsync --append` 按已有前缀断点续传,远比重新全量快;4GB 级文件不要用一次性 cat/单 PUT(会整体失败无续传)。③★127.0.1.1 回环陷阱(最狠的一坑):SLES 云镜像的 cloud-init 会在 /etc/hosts 写 `127.0.1.1 <hostname>`,HANA 服务随之绑定回环地址,addhost 后主节点 nameserver 失联、新节点 indexserver 建卷卡死十几分钟无报错。修法=cloud.cfg 设 `manage_etc_hosts: false` + 模板删 127.0.1.1 行 + /etc/hosts 只留真实 IP,再重启 HANA 才真正在真实 IP 上监听。任何装 HANA 的机器先 `getent hosts $(hostname)` 验证。④hdblcm SPS08 参数坑:--storage_cfg 要传目录不是文件(标准布局干脆不传,datapath/logpath 会自动沿用到 addhost);--ignore=check_signature_file 必加;LSS 默认装 /lss/shared(单机可用,多机必须改到共享路径);hdblcm update 动作会跳过 addhosts("only when installing components"),加节点要用 hdbmodify 直接干(密码经 stdin 传 XML)。⑤hdbmodify addhosts 三连坑:新节点 /hana/data|log/<SID> 目录要先建且 777;数据/日志非共享时 hdbnsutil 报 5514 → global.ini `[persistence] basepath_shared=no`(放弃 standby 可接受);失败残留会让主节点拓扑里该主机 NO/ERROR,清注册需要本机 nameserver 活着,实践上修好根因后直接 StartSystem 自愈(二进制/账号/注册都已在位)。⑥pkill -f 的远程自杀陷阱:命令串本身含匹配词会把自己 shell 杀掉,远程批量脚本里的 pkill 用 `pkill -f 'xxx[y]'` 括号技巧。

【复用建议】云端从零装多节点 HANA 的检查清单:装机后先验 hosts 解析(回环必杀)→ 再验时钟/互信 → 介质 md5 → 共享盘(单机 NFS 即可)→ hdblcm 装主节点 → hdbmodify addhosts(卷目录 777 + basepath_shared 决策)。诊断 addhost 卡死看三处:主节点 nameserver alert(连接被拒的 endpoint 地址直接暴露绑定问题)、新节点 stderr1、nameserver_history。遇到"一直 Initializing"先 `ss -tlnp` 看端口绑在哪个地址,别干等。

## 2026-10-23 SAP 三系统权限镜像:参照账号不存在导致的"克隆抽风"+老系统 SET/GET 参数只有直写表一条路

【背景】客户申请给新员工账号(三系统 DEV/QAS/PRD)按指定参照人镜像权限。生产侧发现前一天"按另一参照人克隆建号"的账号只有空壳(0 角色 0 参数文件),当日会话只修了地址/默认值,角色缺失未被发现。排查后把三系统对齐到业务指定的新参照人,并做了密码重置与参数镜像。

【教训】①"工具读取参照人抽风"的根因往往不是工具问题,是**参照账号在目标系统根本不存在**——克隆前必须先在目标系统做 USR02 存在性+角色集侦察,而不是假设同名账号三系统都有(同一人的账号名在不同系统可能不同,甚至缺号)。②老结构 ECC 系统上 **BAPI_USER_CHANGE 不能改 SET/GET 用户参数**(PARAMETER、PARAMETER1 两张表都返回 029 "No changes",该实现只认 CREATE1 的 PARAMETER1)→ 唯一可靠路径是写小 ABAP 程序经 RFC_ABAP_INSTALL_AND_RUN 直写 USR05(DELETE+INSERT×N,幂等);该通道老编译器不认 `TABLES:` 声明(报 data object has no component),要用 `DATA wa TYPE <表>` 结构体+`INSERT INTO ... CLIENT SPECIFIED VALUES wa`。③老系统 BAPI_USER_PROFILES_ASSIGN 的 PROFILES 表字段名是 BAPIPROF 不是 PROFN(标准名在部分版本不生效,直接 rc=20)。④权限镜像的最强验证=找目标系统里与参照账号**恰好同角色集**的存量用户做对拍:S_TCODE 生效值逐项比(SUSR_USER_AUTH_FOR_OBJ_GET)+UST04 档案集+USR05 参数集,三项零差异才算过,比"检查本次新增项在不在"强得多。⑤"重置过并验证过的密码"不等于"现在还是这个密码":用户登录后可能自行改密(TRDAT 变化+登录报错可反推);初始密码失效态(PWDSTATE=255)也能通过重置恢复并正常登录。⑥本机 sing-box 代理客户端(10809)会不定期静默死亡,任何"git push 不通"先查端口监听,起服务后再走代理。

【复用建议】三系统账号镜像标准流程:①先侦察参照账号在三系统的存在性与角色集(可能同名不同号/缺号,定"口径"要用业务指定的那个系统为准);②目标账号已有角色先 CDPOS/AGR_USERS 留底;③BAPI_USER_ACTGROUPS_ASSIGN 全量替换语义,必须"读全集→一次全传";④PRD 需 PRGN_GEN_PROFILES_FOR_ROLES(iv_usercompare='X')刷新 UST04,DEV/QAS 自动写;⑤手工参数文件走 BAPI_USER_PROFILES_ASSIGN(BAPIPROF);⑥SET/GET 走 AIR 直写 USR05;⑦终验四件套(角色/UST04/USR05/密码登录)+同角色集对照用户对拍生效值。凡"参照克隆"失败,第一怀疑对象永远是参照账号本身在不在目标系统,而非工具。
