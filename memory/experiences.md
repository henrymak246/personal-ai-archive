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

## 2026-11-01 弘信电子结算号段损坏致 DBSQL_DUPLICATE_KEY_ERROR 洪水:号码段问题诊断与修复全流程

【背景】客户 S/4HANA 生产(HXP, HANA 2.0 SPS05, 双应用服务器)月结日突发 DBSQL_DUPLICATE_KEY_ERROR 洪水:两应用服务器各 5600+ 次 dump(全天 2 万+ SNAP 记录),程序 SAPLKO78/LKO78U01 第 60 行 `INSERT AUAA FROM TABLE AA`(KO88 结算 → K_SETTLEMENT_DOCUMENT_INSERT V1 更新)。ST22 全文实证:新结算凭证 0000011602(订单 OR001000379084)与库里 2022-05-10 过账的同号老凭证(订单 001000006205)撞主键。根因=结算号段对象 CO_ABRECHN 的区间与 level 被破坏:当时为区间 1-1,000,000 + level 11601,而库里凭证已用到 1,000,992+(2022 年 1-2 万、2025-07 的 62.5 万、当天上午成功过账的 1000001-1000992)→ 新号全部落进历史已用区 → 每单撞键。当天 16:03 前 SLOG 无任何传输导入 → 系生产环境手工 SNRO 改动(审计日志未开启,操作人不可查)。业务修复=号段区间改 1-1,999,999 + level 提到 1,001,500(高于历史最大),19:19 后洪水停止。

【教训】①★号码段故障的黄金诊断法=「NRIV 对拍」:拿 SNRO 里该对象的 NRLEVEL 与业务表里 MAX(凭证号) 比大小——level 低于已用最大号=必出重复键,不用看别的。本案例 30 秒锁定:CO_ABRECHN level 11601 vs AUAK 最大 1000992。②dev_w 里 RABAX 块的 `checkVals` 值恒定不变(本案例恒为 4152754217/3462114170,连 BELNR=11602 的 dump 也一样),且该值在任何业务表都查不到——内核语句缓存 dump 的打印缺陷(781 内核),**不可当作失败行的键值证据**,别被带偏;真正的键值证据只能来自 ST22 导出(变量区直接能看到 AK/AA 全表内容)。③一个号段错误会被 V1 更新任务重试机制放大成万级 dump 洪水(同一失败记录反复重执行),因此「dump 量级」不能用来推断「受影响单据量级」——本案例号段只被消耗 ~300 个,受影响单据就是这 ~300 单,不是 1.1 万。④V1 更新失败不落账(整记录回滚),AUAK/AUAA 里查不到失败凭证是正常的;失败记录堆积在 SM13(VBERROR,当天 1.3 万条),**这些记录不能直接重处理**(带着旧重复号会再撞),正确顺序=先修号段→清 SM13→业务重跑结算拿新号。⑤号段被改的时间点可以用「业务表里按日期的最大号分布」反推:当天上午有 1000001+ 成功凭证=那时区间还是对的,11:36 起新号掉回 11602=改动时刻;SLOG 无导入→手工 SNRO,而生产审计日志(SM19)默认不开,事后查不到人——建议客户开审计。⑥修复 level 不能拍脑袋:必须先 SELECT MAX(业务表号段字段) 拿真实最大已用号(注意 NUMC 字段字符串比较要等宽,WHERE BELNR > '0010000' 这种不等宽比较结果全错),再设 level=MAX+缓冲余量。⑦ADT datapreview 只读分析踩坑:MANDT 不能进 WHERE(编译器自动处理 client)、聚合函数/单列 ORDER BY 会报语法错(要双列)、VBERROR 无日期列。⑧生产只读分析通道:ADT 账号(SIE)可直接 curl 8001 的 datapreview 查表 + 读函数组源码(functions/groups/ko78/includes/lko78u01/source/main)——读源码确认报错语句是定位第一环。

【复用建议】号码段故障处置标准五步:①ST22 导出 dump 全文 → 确认失败语句与键字段;②NRIV 对拍(SNRO 看 level/区间 vs 业务表 MAX)——定"号段损坏"性质;③查时间线(SLOG 传输 + 业务表按日最大号分布 + 审计日志)定"谁/何时改的";④修复=SNRO 改区间 TO 覆盖最大已用号 + level 设到 MAX 之上 → **SM56/SNRO 重组清号码段缓冲区**(不清缓冲,内存旧缓冲继续发旧号);应急可 SQL `UPDATE NRIV SET NRLEVEL=...`;⑤收尾=清 SM13 失败记录 + 业务重跑 + ST22 重组清 dump。预防:号码段配置不进传输(NRIV 传输覆盖生产是经典事故源)、client copy 后必查 NRIV、SM19 开审计、把「NRIV level vs 业务表 MAX 对拍」做成监控指标(本案例模式可加进 sap-monitor)。关键 SAP Notes:741698(缓冲区重组)、1019692(NRIV 编号不正确)、1244898(NK_REORGANIZE)、678501(NRIV 表锁)。


## 2026-11-02 老结构 SAP 权限互换:纯 AIR 行级镜像互换法 + 三张授权表字段定案

【背景】老结构 SAP ECC 生产系统(BASIS 750 系)两名员工岗位对调(MM采购岗<->SD销售岗),要求权限互换。常规思路=BAPI_USER_ACTGROUPS_ASSIGN 全量替换+PRGN 用户比较刷新 UST04;实际执行改用「纯 AIR 行级镜像互换」,一条 ABAP 程序单 RFC 原子完成,终验四类镜像逐条相等。

【教训】1)★BAPI 全量替换会抹掉 AGR_USERS 赋值行的 COL_FLAG/有效期等"用户级额外定义"(该 BAPI 只认角色名清单,FROM_DAT 重写为当天、标志位清空) -> 互换场景(要完整镜像)应弃用 BAPI,改用 RFC_ABAP_INSTALL_AND_RUN 整行镜像:SELECT * INTO TABLE itab(经典 `DATA BEGIN OF x OCCURS 0. INCLUDE STRUCTURE <表>.` 式) -> DELETE 两用户原行 -> LOOP 改 BNAME/UNAME 后 INSERT CLIENT SPECIFIED,单 RFC 原子提交。2)表字段定案先查 DD03L 再下结论:老系统 USR12/AGR_1252 可能没有 BNAME/UNAME 字段(即档案级/角色级数据而非用户级),"用户级额外定义"实际只在 AGR_USERS 赋值行(COL_FLAG/ORG_FLAG/EXCLUDE/有效期)——互换时换 AGR_USERS+UST04 即可完整覆盖;RFC_READ_TABLE 对不存在字段的查询报 rc=5,别误判成"无权限"。3)授权生效值镜像验证=SUSR_USER_AUTH_FOR_OBJ_GET 取 S_TCODE 全集,断言「互换后现值 == 对方互换前旧值」逐条相等,比"抽查新增项"强一个量级;改完 master 后须删两用户 USRBF2/USRBF3 缓冲行(派生数据,重登自动重建),否则该 FM 读旧缓冲验证失真。4)UST12 的 AUTH 名=参数文件+00 后缀(如档案 ZCY_MM_WMC 的块叫 ZCY_MM_WMC00),精确 = 查 0 行,LIKE '档案%' 才中。5)互换前侦察覆盖:USR02 存在性/锁定/有效期、AGR_USERS 完整行(含 flag)、UST04 全集、USR05 SET/GET 参数(公司代码类参数参与 $BUKRS 授权替换,换不换要用户拍板)、USR41 在线会话(提醒重登生效)。6)"空挂角色"如实告知勿擅修:无 AGR_1016 映射的角色=定义了但从未生成参数文件,互换后状态对调、行为不变,治理另行立项。

【复用建议】权限互换标准四步:1)快照双方全量状态(AGR_USERS 完整行/UST04/USR05/生效值)存 json 可回滚;2)单 AIR 程序行级镜像互换 AGR_USERS+UST04(整行保留 flag/有效期);3)删 USRBF2/USRBF3 缓冲行;4)终验=角色/档案/S_TCODE 生效值三项镜像断言+USR05 未动+缓冲 0 残留,提醒用户重登生效。


## 2026-11-02 老结构 SAP 新建用户岗位角色校正:审批意见是最高依据 + USH02 定位并行改动

【背景】老结构 SAP 生产系统两名新用户的 OA 权限申请(采购岗+后勤岗 / 销售岗),由另一并行 AI 会话先行建号赋角色,但角色挂错(多挂销售岗/挂成另一公司的开单员角色)。接手按"总表+已有角色、不新建角色"的原则校正。

【教训】1)★角色裁定的最高依据=OA 工单流转记录里的审批人意见(如"分配采购岗和后勤岗权限组"、"权限可参考某某(岗位A+岗位B)"),而不是总表的摘要行——总表摘要可能与工单原文不一致(本例总表写"销售岗+后勤岗",工单原文只批销售岗);泛微类 OA 详情页是 SPA 壳(首屏 HTML 仅几 KB),直接 curl 拿不到正文,但之前的 CDP 无头浏览器会话通常已缓存过全文(按 requestId 命名的 detail_*.txt),先找缓存再考虑现抓。2)★"另一个 AI 刚才动了哪些账号"用 USH02(用户变更历史表)按日期全量扫最可靠:USR02-TRDAT 在部分老系统是幻影字段(大量无关用户齐刷刷显示当天,疑似批量作业刷新),不能当变更日期用;CDPOS 变更文档在老系统可能既缺日期列又对 RFC 报 rc=5,不可用。3)参照人权限本身可能已被清空(离职/冻结),此时按批示中写明的"岗位名"映射到系统里同岗位的存量用户角色集(本例参照人零角色,按岗位名找到同岗位两用户,角色集完全一致,可直接套)。4)并行会话的误建痕迹可复盘:同一个人被建了两个不同公司前缀的账号、工号被直接当账号名(后已删)——接手工单后要先扫一遍近期变更,把前一手会话的中间状态一起清理,并主动提示重复账号的授权成本。5)"双份同名角色"是常见形态:同岗位两个文本完全相同的角色,一个承载参数文件(有效),另一个无参数文件映射(空挂)——按存量用户惯例双挂即可,不要新建角色,也不要只挂空的那个。

【复用建议】新建用户/权限变更单的标准处置链:1)扫 USH02 近 3 天全量变更,定位前一手会话动过的账号与中间状态;2)抓 OA 工单全文,以审批人意见+批示的参照人为准(总表摘要仅作索引);3)岗位→已有角色映射优先查同岗位存量用户(角色集+参数文件集双镜像);4)执行用单程序行级改写+清缓冲+生效值方向验证(应有事务码在/不应有的不在);5)收尾清点:重复账号/未执行的同批工单/参照人异常状态,逐项报用户拍板。


## 2026-11-02 续: 老用户权限批次校正的"按总表执行"口径 + 参照人账号已删的定案

【背景】承接同日前一条(新建用户岗位角色校正)。用户对遗留批次的口径:①医出(JCK)是不同公司,同名不同前缀的账号不是重复号,不删;②"确认的话可以执行"=按总表确认即可执行,无需再抓OA工单原文(抓单动作被用户拒绝);③一律复用已有角色。

【教训】1)权限校正批次的正确裁决链:总表「个人权限变更表」是最新事件日志、「账号」页是状态台账——两者冲突时以**日期更晚的事件**为准(本例一人 08-28 台账标注采购/销售岗删除,但其 10-27 变更单无"取消"字样、语义为申请采购岗+销售岗+保留内勤,故现状全岗=已符合,不动;另两人变更单明确"取消销售岗",执行摘除)。2)变更单摘要的"取消"字样是裁决关键:有"取消"=摘、无"取消"=现状态已符合,不要按邻行类比过度执行。3)参照人账号"零角色"要先查 USR02 是否**账号本身已删**(USH02 在用户删除后仍残留历史行:本例 2024-01-04 被 SU10 批量锁 UFLAG=64,之后账号删除,总表台账未记冻结/删除)——参照人不在系统时,按批示写的岗位名映射到同岗位存量用户,结论仍然成立。4)冻结(离职)处置的现状判断看 UFLAG(本例 64=已锁)+USH02 的 SU01/SU10 锁记录;已锁的用户角色保留是常态,无需再摘角色。5)补角色时先查 UST04:档案可能已挂过(前一会话挂了档案漏了角色),只补角色即可,INSERT 前先判断避免重复键。

【复用建议】批量权限校正四步:①总表两页对读(变更表定最新事件+账号页定状态台账);②按"取消"字样分类(摘/不动/补);③执行前查账号锁定态与档案现状(避免重复插);④终验用生效值方向断言(应有在/不应有无)+缓冲清零。


## 2026-11-02 续二: 新建 SAP 账号的完整性清单(角色之外的部分常被漏)

【背景】接同日前两条(新建用户角色校正)。复核两名"新增用户"工单账号时发现:前一会话建号只做了"建号+挂角色",基础配置缺口一堆——日期格式未对齐、小数格式/登录语言空、姓名国家格式未设、SET/GET 参数空。

【教训】新建账号完整性终验清单(角色之外逐项查):①USR01-DATFM/DCPFM(组织标准, 本例 4=年.月.日/X=1,234.56);②登录语言 LANGU(经 BAPI_USER_GET_DETAIL 的 DEFAULTS 读);③ADDRESS-NAMCOUNTRY/NAMEFORMAT(中文名须 CN/01, 否则部分界面姓名顺序错);④SET/GET 参数 USR05(对照同岗位参照人——本例采购岗三人一致 BSA=ANB/BUK=G0F0/EXP=X/SCL=X;参照人参数分裂时不要替用户猜,留空上报);⑤密码状态: 新号按统一初始密码并实测 RFC 认证(认证通过但无 S_RFC 时 open 报 "No RFC authorization for RFCPING"=成功签名;PWDSTATE=1 首登强制改密), 老号改过密(PWDSTATE=255)不要用默认密码覆盖;⑥有效期/锁定态。技术小坑: 自研 ctypes RFC 封装常缺结构字段写入方法——RfcSetChars 对结构句柄同样适用,可直接复用表行写入函数;BAPI_USER_CHANGE 的 RETURN 是表参数不是结构(按表读 TYPE/NUMBER/MESSAGE)。

【复用建议】接手工单时默认"前一手只做了半套":角色校正后必须跑一遍上述六项完整性终验,把显示格式/参数/密码状态补齐,才叫"新建用户"真正闭环。
