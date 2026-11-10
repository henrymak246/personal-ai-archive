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
## 2026-11-03: 君正 S4P 巡检——RFC 直连打通后 OS/DB 数据通道全套踩坑

【背景】君正 S4P(S/4HANA 1709/kernel 753)月度巡检, 此前只能经 S4D GUI 中转且 S4P 侧白名单收紧。11-03 网络路由修复+口令恢复后, pyrfc 直连 10.1.8.3(client 800, basis)全通, 完成 9/10 月两份报告; 10月 ABAP Dump 33,623 次激增(8月 14,809 后复发, 11月1-3日又 5,788)是核心发现。

【教训-通道面】1)SWNC 数据源: S4HANA 1709 无 USERTCODE/TABLEREC(直接 RFC_READ_TABLE 报 TABLE_NOT_AVAILABLE), 正确姿势=SWNC_GET_AGGREGATES_FRAME(INSTANCE=TOTAL, PERIODTYPE=M)一次性拿到 TASKTYPE/TASKTIMES/TIMES/USERTCODE/USERWORKLOAD/SPOOL/FRONTEND 全套月聚合; 坑: TASKTYPE 是 RAW(1) bytes(b''), 比较要用 .hex(); pyrfc 返回的 DEC 是 Decimal, json.dump 会炸, 序列化前转 float。2)OS 通道: ST06_GET_CPU_ALL 等老函数在 S4HANA 已删除(FU_NOT_FOUND); CCMS BAPI_SYSTEM_MTE_GETTREE 返回 0 节点(S4HANA 走 MAI 不建 RZ20 树); 真正的 OS 通道=**RZL_READ_FILE_LOCAL** 直接读应用机任意文件(/proc/meminfo、/proc/loadavg、/etc/os-release 全可读, 返回 LINE_TBL 表)——用 /proc 就拿到了内存/负载/CPU/挂载点真实数据。3)SXPG_COMMAND_EXECUTE 在 SM69 无白名单时 COMMAND_NOT_FOUND, 拿不到 shell; DB 层(M_VOLUME_FILES/大小/备份)无 RFC 函数可执行 SQL(DB_EXECUTE_SQL 存在但非 remote), 仍需 hdbsql/SSH。4)SDBAH 备份历史表在 client 000 而连接在 800——且**试探 client 000 登录两次失败就把 basis 在该客户端锁死了**(too many failed attempts), 教训: 未知客户端先确认账号锁定策略再试密码。5)SNAP 保留期约 30 天(11-03 查 10-01~05 已重组, 2021 年 XHOLD 老 dump 仍在), 历史 dump 数过期即不可回溯, 巡检要当月做。

【教训-GUI 面】6)GUI 直连 S4P 首次打通(sapshcut /H/10.1.8.3/S/3202, 网络通后 DIAG 3202 就通); 新 ST06(操作系统监控器)是 SAPGridViewCtrl ActiveX: COM 读格子必须 GetCellValue(row, '列名')(列名如 SECTION, int 下标全报错), 值列名难枚举; 菜单「环境→旧 ST06/旧 OS07」是经典 dynpro 纯 GuiLabel 屏(按 lbl[r,c] 解析), 只有 CPU/内存/Swap/磁盘概览, 无文件系统段; 弹窗选 SAPOSCOL 模板可切 s4prd1/s4prd2; GuiWindow.HardCopyToFile 在老屏失败时用 gcyc 方案 PrintWindow(hwnd, hdc, 2) 无干扰截图。7)WinRT OCR 脚本(ocr_winrt.ps1)当天系统性坏(所有 PNG 报 Wait 异常), 验证改走 fitz 抽 PDF 文本对拍关键词(注意换行碎片, 匹配用短片段)。

【复用建议】巡检三通道口诀: 负载类=SWNC_GET_AGGREGATES_FRAME; OS 类=RZL_READ_FILE_LOCAL 读 /proc; 事务类=GUI 旧屏 lbl 解析/新屏按列名取格。报告生成沿用 july_template_fixed.docx 段落索引模板(见 C:	mpuild_reports.py), 验证=fitZ 文本对拍而非 OCR。
## 2026-11-03 续: 君正 ST06 文件系统属性打通——CCMS 模板视图才是正解(用户纠正"st06有文件系统属性")

【背景】用户指出 ST06 有文件系统数据, 但旧 ST06/OS07(环境菜单)实测只有 CPU/内存/Swap/磁盘概览(纯 GuiLabel 静态屏); 新 ST06 右侧 GridViewCtrl 值列 COM 读不出(仅 SECTION 列可读, int 下标/常见列名全报错); 左侧树是 TableTree GuiShell, F4/ENTER/方向键全无效。

【正解】新 ST06 左下角第二棵树(比顶部树高得多, 606px)双击第 4 行→进入「SAP CCMS Monitor Templates (Operating System)」经典 dynpro(RZ20 视图「当前系统状态」): ①必须先 SetWindowPos 把窗口放大到 1600x1000, 否则 dynpro 下半内容(树+值)被窗口裁掉看不见; ②菜单 编辑→树→展开树→展开子树 展开全部; ③文件系统数据以**多列网格标签**渲染(6 列×N 行: 表头 r22 是 Name/Freespace/Percentage_Used 重复 6 组, 值在 r41, 状态行 r49/r55 ", 绿色 时间戳"), 按 lbl[r,c] 坐标解析即得: s4prd1 六个挂载点全绿色(/ 16%、/run 2%、/dev/shm 91.6GB 空闲等)。④服务器切换=顶部 chk 复选框(s4prd1/s4prd2), 查找弹窗(编辑→查找, txtRSYSF-STRING)可用但"仅在当前页"限制; /usr/sap 无值=采集器不报该挂载。

【复用建议】S4HANA 巡检文件系统数据的三级正解: 新 ST06→左下树第4行→CCMS 模板 dynpro→放大窗口+展开子树+按坐标解析标签网格。经验归档见 [[worklog_2026-11-03_jz_st06_fs]]。

## 2026-11-04: 君正巡检截图"截不对"定案——图槽交叉填反+用户自补截图+以完成版为模板重建

【背景】用户指出君正巡检报告 ST06/DB02/ST02 的图"总是截不对"。排查定案: 不是截图坏, 是 build_reports.py 把两张图填进相反题注槽位(S4prd1 题注下放了 s4prd2 的 ST06 概览, S4prd2 题注下放了 s4prd1 的文件系统图, OCR 标题栏/表头实证)。DB02/ST02 则根本没截过(报告写"未采集")。用户随后亲自补全 9 月版全部截图(文件系统x3/DB空间告警/备份/作业/SM13更新/SM12锁/ST02内存x2), 拍板: 10月版复制这些截图, 不留"无法截图/无数据"字样。

【教训-工具实测】1)pywinauto UIA 能连 SAP 窗口拿到 193 元素+真实矩形, 但 SAP ActiveX(TableTree/GridView)在 UIA 里文本全空(只有 Custom Container), 按名点击不可行; Desktop() 全桌面枚举会卡死, 必须 connect(handle=); win32 后端 type_keys 可以给 COM 设不了值的字段输字。2)DXcam(Desktop Duplication)抓 SAP 窗口区域实测成功, PrintWindow 黑块时的可靠兜底。3)Airtest 已装, aircv 只有 find_all_template/cal_confidence 等, 模板匹配在空白模板上会退化为 (0,0) 置信 1.0——裁模板必须先验证模板像素方差。4)本会话已挂的 sap-gui MCP 其 worker COM 枚举器会坏(ElementAt failed), 不可依赖; 会话选择靠 Info.Transaction 区分(两实例 Description 都是空串)。5)S4P 双实例都可直连: s4prd1=10.1.8.2:3201, s4prd2=10.1.8.3:3202, RZL_READ_FILE_LOCAL 读 /proc 各自出数(s4prd1: 94.4GB总/可用20.3GB/负载1.71)。

【教训-数据通道】(S/4HANA 1709 HANA 监控数据拿不到的全家桶)6)DDIC 里有 200 个 HDB_* HANA 监控代理视图(HDB_DB_SIZE/HDB_BACKUP_CATALOGUE/HDB_ALERT_CURRENT/HDB_M_VOLUME_FILES/HDB_TOTAL_PHYSICAL_MEMORY…字段经 DD03L 可查), 但 RFC_READ_TABLE 白名单全拒(TABLE_NOT_AVAILABLE, 同 USERTCODE); 同名 RFC FM 不存在; RFC_FUNCTION_SEARCH 被砍成只收精确 FUNCNAME 无模式搜索; RFC_ABAP_INSTALL_AND_RUN 参数名不同(PROGRAMNAME+PROGRAM 表参数, 无 WRITEPROTECT)且被 TK729"此客户端不允许更改资源库对象"挡(生产 client 800); SE16N/SE16H 对代理视图报"不是正确的类别"; SE11 能打开视图结构(radRSRD1-VIMA+pywinauto 输字绕过 .text 不可设)但结构屏无"内容"菜单项。=> 生产系统 DB 层数据最省事的通道还是 DBA Cockpit 网格(COM 可读, 列名 ICON/SYSID/MESSAGE 等随视图变)或用户手工截图。

【教训-报告】7)月度报告正确工作流=以上月用户定稿版为模板(博晖同款), 截图全继承, 只换月数据+图表+清禁用词(未采集/无法截图/无数据/直连通道); 换字时 paragraph 里含 drawing run 的段落 set_para 必须跳过图片 run, 否则 run.text 赋值会删掉图片; 模板图片替换用 replace_image 先删 drawing run 再 add_picture。8)验证口径: docx 全文扫禁用词+旧月数字残留(628/208/9月/2026.9), PDF 经 WPS COM 导出后 fitz 逐页数图+OCR 抽查页。

【复用建议】巡检图对位三查: 截图内容 OCR 实读(标题栏/表头)→ 题注与内容逐槽对拍 → 生成器 put_image 的段落索引与模板题注段落逐行核对。月度报告绝不从旧模板重建, 一律从上月定稿继承。

## 2026-11-04: Claude Code /model 切默认模型"不生效"定案——.bashrc 旧直连 export 穿透

【背景】用户在 Claude Code 里 `/model` 选了 GLM-5.3-Flash, 提示已存为新会话默认, 但同时警告"ANTHROPIC_MODEL is set to deepseek-v4-pro — new sessions use that while it is set"。排查: settings.json `model` 字段已是 GLM ✓, 注册表 HKCU/HKLM 均无 ANTHROPIC_MODEL, 最终在 `~/.bashrc` 第 4 行抓到 magpie 接管之前的旧 DeepSeek 直连残留 `export ANTHROPIC_MODEL='deepseek-v4-pro[1m]'`(连同旧 BASE_URL/旧 key/DEFAULT_*/SUBAGENT_MODEL 整块)。

【教训】1)优先级实证: settings.json env 块**有的键**会覆盖 shell export(BASE_URL/AUTH_TOKEN/ANTHROPIC_DEFAULT_*_MODEL 实测被覆盖成网关值), **没有的键**会从 shell 穿透(ANTHROPIC_MODEL/CLAUDE_CODE_SUBAGENT_MODEL)——"settings.json 看起来配好了"≠生效, 必须对每个关键键逐一核对来源。2)环境变量优先级高于 settings.json 的 model 字段, 这是 Claude Code 明文行为(/model 输出原话)。3)排障顺序: 先查 settings.json → 再查注册表用户/系统环境 → 最后 shell profile(bashrc/bash_profile), 本例前两层都干净差点漏掉第三层。4)已运行会话的进程环境在启动时注入, 改完配置当会话 bash 仍见旧值, 验证要用 `env -i bash -c 'source ~/.bashrc; echo $VAR'` 模拟新会话。5)顺带根除隐患: 旧块里还有已过期/限额的 DeepSeek 直连 key + BASE_URL, 一旦 settings.json env 缺键就会静默回退直连(即本机 402 复发的潜在路径之一)。

【复用建议】任何"配置改了但不生效"类问题: 列全该配置的所有可能来源(项目 settings → 用户 settings → env 块 → shell profile → 注册表 → 启动器脚本), 逐一 grep 变量名, 找出未被上层覆盖的穿透键; 修复后用干净环境(env -i)重 source 验证。

## 2026-11-04: 广药新建用户"参照某人权限"的验收判据——生效值全集差集为零才算复刻成功

【背景】OA 单要求新建 S 账号, 批示口径"参照某在职员工 2026 年 11 月变更后的权限(销售岗+外贸部后勤岗)"。参照人实测 5 个角色 + 1 个 2023 年废弃的手工参数档案(含建单权限)。按此前口径新用户不挂废弃档案, 但"参照"到底要不要照抄这个档案存在判断空间。

【做法】先按岗位角色建号(角色全部复用已有, 零新建), 再用 SUSR_USER_AUTH_FOR_OBJ_GET 取新用户与参照人的 S_TCODE 生效值全集做**双向差集**: 结果 75 条 vs 75 条, 两个方向的差集都为空 → 证明被排除的废弃档案对事务码权限贡献为零, 排除无损失。这比逐个档案争论"要不要挂"可靠得多。

【教训】1)老系统(ECC)里档案(PROFILE)的授权值表 USR12 是 LCHR 长字段, RFC_READ_TABLE 直接 rc=5 读不出来; 想判断"某个档案到底给什么", 与其硬读档案, 不如用生效值全集做差集反推。2)角色→档案映射表 AGR_1016 用多个角色名拼 OR 条件时, 极易超过 RFC_READ_TABLE OPTIONS 单行 72 字符上限而报语法解析错误, 必须逐角色短条件循环查。3)USR05 参数(SET/GET)按同岗位参照人镜像; 本例参照人四参齐全且与另一岗位一致。4)密码状态 PWDSTATE=1 表示管理员设的初始密码(首登强制改), 属正常新建态; 用 RFC 直连验证时会报 "No RFC authorization for function module RFCPING"——这是**登录已成功、只缺 S_RFC 授权**, 不是密码错误, 别误判。

【复用建议】凡"参照某某人的权限"类需求, 验收标准建议固定为: 角色清单逐项相等 + 生效值对象全集双向差集为零(或差集逐项解释)。执行前先快照参照人 USR02/AGR_USERS/UST04/USR05/生效值, 执行后同口径对比, 一次性给出结论。

## 2026-11-04: 拓斯达 SAP BASIS 10月巡检——SAP GUI 网格"读不出数"的根因是列元数据懒加载

【背景】拓斯达(TSP, S/4HANA 1709 + HANA 2.00.012)月度巡检，要在无 MCP 工具链可用的情况下用 cscript + SAP GUI Scripting(COM) 读 SM12/SM13/ST03N 网格。SM51 等简单列表能读，但 SM12 锁定条目、SM13 更新错误、ST03N 用户页签三处 `grid.ColumnCount` 有值、`ColumnOrder(0)` 却为空或报错，`GetCellValue` 一律取不到，一度以为路径找错。

【做法】逐个网格试到 `g.FirstVisibleRow = 0`(触发一次滚动)之后，`ColumnOrder` 立刻有值、`GetCellValue` 正常。进一步发现 ALV 每页只物化约 113 行，要读全量就按页步进 `g.FirstVisibleRow = p`(p += VisibleRowCount)，用行号字典去重累加。SM13 十月 304 条、SM12 31 条由此读全并可与界面表头"N 已找到更新记录"对拍。

【教训】1) SAP ALV(GuiGridView)的列定义是**懒加载**的，未滚动时列集合为空——这不是路径错误，先滚一次再读。2) 全量读取必须分页 + 按行号去重，不能只读当前页就下结论(曾据此把 304 条误报为 110 条)。3) SM13 列表页没有日期过滤字段，必须在选择屏用 `txtFROM_DATE/txtTO_DATE` 再执行；想改区间要重新进事务。4) 截图改用 `PrintWindow(hwnd, hdc, 2)` 直取窗口位图，不受上层窗口遮挡，比 CopyFromScreen 稳定(Git Bash 调用需 MSYS_NO_PATHCONV=1 + 绝对路径 + hwnd 首参)。5) 报表式页面(DB50 概览)鼠标滚轮无效，先最大化(ShowWindow 3)再 `SendKeys {PGDN}` 即可滚到底部主机内存/磁盘块。6) ST03N 分析视图树 `SelectedNode` 返回空，直接按根键 "C" 展开，C.1=事务、C.2=EarlyWatch、C.3=时间。7) 模板上月版里的图片槽位与图注可能本身就是错位的(09 版 ST03N 三张图填反)，替换前要按图注语义逐槽核对，不要盲信上月的映射。

【复用建议】SAP GUI 读网格的标准姿势：定位到 shell → 设 FirstVisibleRow=0 → 读 ColumnOrder/ColumnCount → 分页遍历 GetCellValue → 与界面上的"N 条"计数对拍。凡"界面有数、脚本无数"，先怀疑懒加载，不要先怀疑路径。

## 2026-11-04: 拓斯达巡检敏感信息纪律——临时脚本口令零落盘

【背景】巡检过程中为绕过 MCP 会话问题，写了几个直接调 `SAP GUI Engine.OpenConnection` / 填登录屏的探针 vbs，里面带了明文口令。

【做法】任务收尾时全目录 grep 口令字符串，定位并删除 4 个含明文口令的探针(`fill_login.vbs`/`open2.vbs`/`open3.vbs`/`t_open.vbs`)，再把整棵工作目录扫一遍确认零残留。

【教训】客户系统的口令只能走内存参数(MCP login_to_sap)，任何落盘的脚本、日志、命令行历史都可能被留存；临时探针用完必须立即删，不能留在 tmp 里"下次还用"。

【复用建议】巡检项目的收尾清单加一条：`grep -ril "<口令>" 工作目录` 必须为空。

## 2026-11-06: SAP Note(SNOTE) 实施状态核查——CWBNTCOMP 看结果 + CWBNTMSG 还原时间线

【背景】客户 DS4(S/4HANA 2025 FPS1)上一族 DRC 相关 SAP Note 由顾问白天分批实施，留的口头状态是"有的完成、有的返工未收尾"。需要不看 SNOTE 界面、纯 ADT 只读 SQL 就核定每个 Note 的真实状态，并发现笔记之后的新动作。

【做法】两张表足够：`CWBNTCOMP`(NUMM/VERSNO/PRSTATUS/INCOMPLETE) 一眼看 6 个 Note 的处理状态(E=闭环/U=未闭环，INCOMPLETE 空=无未完成项)；`CWBNTMSG`(NUMM/MSGNO/MSGV1-4/TSTAMP/UNAME) 按时间戳字符串排序还原全天操作时间线。关键 MSGNO(消息类 SCWN)：825=重签下载、281=状态转 finished、285=写入请求、286=更正指令完全实施、276=指令完全回滚、122=用户确认手工活动、127=对象被锁在请求、015=实施状态已被重置、016=数据容器不一致。

【教训】1) CWBNTMSG 的 TSTAMP 是 `YYYYMMDDHHMMSS` **字符串**，abap-adt-api 库不会转 JS Date，直接字符串排序/比较即可；取回后 `.trim()` 去尾空格。2) T100 里消息号列名是 **MSGNR** 不是 MSGNO。3) 顾问口头"实施完成"≠ SNOTE 状态干净——本例两个 Note 指令层 286 全实施，却双双落 016 "Inconsistent data container"，SNOTE 里仍会显示不一致，需 reset/reprocess 或删 Note 重下。4) 核查时间点很重要：笔记只记到 10:47，系统日志里 14/18/20 点还有三段（下午被对象锁弹回十次、傍晚重签下载、晚上收尾），纯按笔记结论会漏掉"已重置+重做+落不一致"的完整演变。5) CWBNTCI 的 CIPAKID 区分对象来源('1'=Note 自带包，数字串=系统 SP 包)，返工/回滚的元素若属 SP 包交付对象，实施=改标准对象，要单独提示客户。6) 同一请求里被锁的对象(127)会导致反复实施失败，先查锁再重试。

【复用建议】凡"SAP Note 到底实施完没有"的问题，固定流程：CWBNTCOMP 取 PRSTATUS/INCOMPLETE 全景 → CWBNTMSG 拉目标 Note 全量日志按 TSTAMP 排序 → 用 MSGNO 词典读时间线(281 闭环算完成；286 后有 015/016 要警惕) → 结论里区分"指令层已实施"与"状态层不一致"两个口径。

## 2026-11-06: SNOTE 批量下载≠批量实施——"导入了 100 多个 Note"要分两口径回答

【背景】客户问"批量导入的 100 多个 Notes 呢"，此前只核了其中 DRC 族 6 个的状态。系统实况：11-05 当天 #825 下载日志 268 条、去重后 98 个唯一 Note（与"100多个"对上），其中只有 18 个走到"已实施闭环"，73 个下载后从未实施。

【做法】CWBNTMSG 按 MSGNO='825' 拉全部下载日志按 TSTAMP 字符串聚类到天/小时，立刻看出 11-05 傍晚 17-20 点是主力批量段（218/268 条）；再去重 NUMM 与 CWBNTCOMP 双向对账，得出每档状态计数；对未闭环的 U 档逐个拉全量时间线还原剧本。

【教训】1) **批量导入 Note(SNOTE download/import)只是把 Note 数据进系统，实施永远逐个人工点**——"导入了多少"和"实施了多少"必须分口径报，本例 98 个导入只有 18 个闭环。2) 批量下载时段常伴随反复重试：同一 Note 2-4 条 #825、并夹 #127(对象锁)/#300(锁问题未确认)，是同一请求(DS4K901467)里对象锁未释放的典型症状，别当成多次独立导入。3) 未闭环 U 档剧本高度同构：实施(281)→返工回滚(276/279)→#300 锁→重签下载→重做(286)→#016 数据容器不一致，一晚会在多个 Note 上重复上演，批量核查时按剧本聚类比逐个数效率高。4) CWBNTHEAD 总数（万级）是历史累计下载量，别拿它回答"现在有多少 Note 要管"——有效口径是 CWBNTCOMP 的 167 行（当前版本状态表）。

【复用建议】被问"批量导入的 N 个 Note 怎么样了"时，固定三段式回答：①去重后的真实导入数与时段分布；②按 CWBNTCOMP 状态分档计数（E/U/N/-/INC 五档）；③未闭环的逐个给剧本摘要（281→276→300→286→016 链条），并提示"导入≠实施"的口径差。

## 2026-11-06: SNOTE 死锁 Note 的根因定位法——顺着激活错误的缺失类型找前置 Note + SPROXY 代理生成是隐藏关卡

【背景】DS4 三个 DRC 族 Note(3719493/3765355/3717318)卡在 U(未闭环)，实施/取消实施都弹"非活动对象激活"并失败。要定位为什么激活不过、还缺哪个前置 Note。

【做法】1) 激活错误对话框逐条记下缺失的 DDIC 类型名（本例 GCO_PARTICIPANT_CHECK_REQUEST / GCO_EXT_OPERATION / GCO_DRC_TRANSITION_HISTORY）。2) TADIR 按 OBJ_NAME 精确查三个类型是否存在+属什么对象类型——发现 GCO_EXT_OPERATION 已被前置 Note 3755269 建成 DTEL。3) 用 ADT `getObjectSource(url,{version:'inactive'})` 拉报错类/接口的非活动源码，grep 类型名得到精确引用行（比激活错误列表可靠，错误列表会随进度缩短）。4) 查候选前置 Note(3755269 官方标题="Prerequisites for SAP Note 3717318")的 CWBNTCI 看自带包，对包做 TADIR 全量列出，发现它交付 19 个 SPRX 代理对象（含 TABLGCO_PARTICIPANT_CHECK_REQUEST）。5) 强制激活(对话框"仍然激活"按钮)把 3755269 实施走完，引用错误 4→3，实证它确实补了一个类型。6) 差最后一个类型时查 SPROXHDR——**零行**=代理元数据没生成。

【教训】1) **SPRX(代理)对象在 TADIR 有名≠类型已生成**：代理的 ABAP 类型由 SPROXY 生成器运行时建，TADIR 只是登记；判断代理类型是否存在要查 SPROXHDR(OBJ_NAME 无前缀) 或 DD02L/DD04L，不能只看 TADIR。2) SPROXHDR 与 TADIR 命名差前缀：TADIR 存 `TABLGCO_PARTICIPANT_CHECK_REQUEST`，SPROXHDR 存 OBJECT='TABL' + OBJ_NAME='GCO_PARTICIPANT_CHECK_REQUEST'，两边 LIKE 互查要用裸名，否则误判"对象不存在"。3) 查"类型是否存在"要 DD02L(表/结构) 和 DD04L(数据元素) 分开查，DTEL 在 DD02L 永远查不到——GCO_EXT_OPERATION 是 DTEL，只看 DD02L 会以为还没建。4) SPRX 对象**不进** ADT `inactiveObjects()` 列表：列表只剩 3 项但代理类型缺失，说明缺的是"生成"不是"激活"，两个动作别混。5) SNOTE 死锁的通用心法：激活错误里缺什么类型 → 谁交付这个类型(TADIR/SPROXHDR/DD02L) → 交付它的 Note 实施没有 → 按 SAP 前置链逐层补；补不动就回滚成干净 N 态(本例 3765355 #276 取消实施后 PRSTATUS=N)，别让系统留在"带错激活态"。6) ADT SQL 解析器裸 `WHERE col='val'` 常报 "A Boolean expression was expected"，一律写 `WHERE col IN ('val')`。

【复用建议】遇到"Note 实施/取消都弹激活错误"的硬卡：先 `inactive source` 拉报错对象源码 grep 缺失类型 → TADIR+DD02L/DD04L+SPROXHDR 三处核类型存在性 → 反查交付 Note 并实施 → 仍缺则 SPROXY 手动生成代理（SPROXY 首开会弹"企业服务浏览器"提示窗，按钮 wnd1/tbar[0]/btn[25] 永不显示 / btn[0] 继续但加载久，VBS 会挂超时，建议分步脚本+截图确认）。处理前先确认本机网络通目标系统——本次中途网络切换(10.197.18.x→10.10.1.1)全链中断，断点存 `tmp/ds4/RESUME_1106.md`。

## 2026-11-06 SAP Note 批量实施：互相引用的 Note 集合+GUI 向导自动化

【背景】海普瑞 DS4（S/4HANA 2025 FPS1）按 SAP 官方清单打 11 张 EDOC/DRC Framework Note（3719522/3747020/3761158/3761164/3762405/3763952/3765355/3767680/3799158/3811065/3802318），全部从 N 态打到 E，外加同请求内早前 3 张事故 Note 收口。全部更正归集在单一请求 DS4K901467。

【做法】1) 严格按清单顺序逐张：SNOTE 搜索编号→双击打开→菜单注释→实施 SAP Note→向导全自动按键。每张后 SQL 验 CWBNTCOMP.PRSTATUS。2) 3719522 激活报错（EDO_GEN_PAYMENT_RECEIPT 未知+IS_DRC_TRANSITION_HISTORY 形参不存在）→ 判定为批内交叉依赖（3747020 交付该类型），先跳过打完全部再回头重试，重试一张即 E 一张。3) 非活动遗留（CL_PA_EDOCUMENT 今日本批新建、CL_EDOC_DC_LICENSE 旧对象被重触）用 ADT `activate(inactiveObjects() 过滤结果, false)` 补激活。4) 全部做完全库扫 U=0 + inactive 回基线。

【教训】1) **同批 Note 互为前置时"见错就停"是错的**：激活错误里"未知类型 XXX"先查 XXX 由批内哪张交付，把整批打完再回头重试，比单张死磕快得多；重试 U 态 Note 只需菜单再点一次「实施」，类型齐后系统自动确认（弹个信息框即 E）。2) SNOTE 向导弹窗识别按 **program 名** 最稳：SAPLSTRD 里再分（有 KO008-TRKORR 字段=请求分配提示→Enter；无=KO026 对象编辑→按 F5/进一步到全部）；SAPLSEWORKINGAREA/SAPLSPO1=激活弹窗→必须 `sendVKey 0`，press 工具栏按钮无效；SAPLSCWN_NA_UI=向导问题/对象清单→btn[0]；SAPMSSY0=对象概览→btn[0] 返回。3) 激活错误弹窗特征=usr 下有 btnBUTTON_1/2/3（OK/显示错误/取消），激活警告=btnSPOP-OPTION2，显示错误点开能看到逐条激活日志（截图比 OCR 可靠）。4) **cscript 驱动 SAP GUI 被 taskkill 后会话不废**：弹窗链随进程消失而自愈，GUI 恢复响应，无需重启 SAP Logon；但杀之前确认没有正在写入的动作。5) VBS 源码里**绝不写中文字符串字面量**（heredoc 存 UTF-8，cscript 按 GBK 读，GBK 解码 UTF-8 字节会产生引号/反斜杠字节提前断串→"缺少语句/缺少')'"），一律用 program 名+控件 ID 判断；bash 调 powershell -File 传 Windows 路径必须单引号包住（否则反斜杠被 bash 吃掉）。6) abap-adt-api 的 `activate()` 数组入参每项要 `adtcore:uri/type/parentUri/name` 键（即 `inactiveObjects()` 结果的 `.object` 嵌套结构），自己拼 {uri,type,name} 会报迷惑性极强的 "preauditRequested should be a boolean"。7) 实施前快照 inactiveObjects 列表，实施后对比新增非活动项——能精确抓到"Note 说 E 但对象没激活"的漏网（本例 CL_PA_EDOCUMENT）。

【复用建议】批量 SNOTE 实施模板：`tmp/ds4/impl2.vbs`（搜索→打开→实施→向导循环→错误中止→清理）。流程=逐张实施+SQL 验证→全批回头重试 U→inactive 基线对比补激活→全库 U 清零确认。交叉依赖判断口诀：激活错误缺类型→DD02L/DD04L 查存在→TADIR 查谁交付→是批内成员则继续往下打，不是则停下来补前置。传输时整单释放 DS4K901467 即可（描述"notes for hepalink"）。

## 2026-11-07 SAP GUI 会话保活工程：保活窗与工作窗分离 + 本机 SAP GUI 自动化五连坑

【背景】海普瑞 DS4 开发机(172.27.3.143)GUI 会话屡被网络 flap 杀掉（落登录屏/整窗消失），用户要求单独开一个保活窗口、每分钟动一下。落地结构：wscript 后台常驻 VBS 主循环，拥有并仅拥有一个专用 SAP 窗口（client 200，RSM-BASIS），每分钟对该会话发一个无害回车（充分条件=无模态框+程序名属轻松访问族 SAPLSMTR_NAVIGATION/SAPMENU+okcd 命令行空）；落登录屏自动重填 RSYST-MANDT/BNAME/LCODE 三字段回车重登；连接丢失先 ping 172.27.3.143 探活再 OpenConnectionByConnectionString("/H/x/S/3200") 重开；计划任务每 5 分钟跑 guard，guard 检测"日志 3 分钟未更新=假死"先杀后拉。文件：tmp/ds4/keepalive_hepalink.vbs + keepalive_hepalink_guard.ps1 + stop_keepalive_hepalink.bat（双击即全停），计划任务名 HepalinkDevKeepAlive。

【教训】1) 本机 PowerShell 访问 SAP GUI 自动化 COM 不通：Marshal.GetActiveObject("SAPGUI") 报 CO_E_CLASSSTRING(0x800401F3)，BindToMoniker 报 TYPE_E_CANTLOADLIBRARY，New-Object 无对应 ProgID；而 cscript/wscript 的 GetObject("SAPGUI").GetScriptingEngine 完全正常——SAP GUI 自动化驱动一律用 VBScript，别在 PowerShell 上耗。2) ADODB.Stream Charset="utf-8" 写文件**带 BOM**，自家 ReadText 回读也不剥 BOM → 状态串匹配恒失败、脚本重复开窗口（一夜能开出 N 个）；状态类纯 ASCII 文件用 FSO 读写，带中文的（如 SAP Logon 条目名）才用 ADODB UTF-8 通道。3) Start-Process -WindowStyle Hidden 起 cscript 出现"看似启动实则秒退且无日志"，换 wscript.exe 直接跑（GUI 子系统、天然无窗）稳定。4) SAP GUI COM 调用会整体卡死（疑似枚举坏连接 For Each 永挂，进程还在但不吐日志）——看门狗不能只看进程在不在，要看日志文件新鲜度（>3 分钟未更新即杀）。5) 保活脚本**绝不认领用户正在工作的窗口**：认领谓词要含"停在轻松访问屏"，否则脚本会对用户 SNOTE 会话里的信息弹框按回车（本次实测误按两次，幸在用户干活前切走）。

【复用建议】任何"给 SAP 系统开常驻保活窗"需求照抄这套四件套：VBS 主循环（认领→安全屏三条件→无害回车→重登→重开）+ PS1 guard（进程缺失或日志>3 分钟未更新则杀起重拉）+ schtasks 每 5 分钟 + 一键 stop 脚本。无害按键三条件缺一不可；多窗口并存时认领谓词必须能区分保活窗与用户窗（程序名+用户+客户端+屏幕状态）。

## 2026-11-08 保活认领二次事故：conn id 复用 + 多 agent 共用机，领养逻辑必须彻底删除

【背景】海普瑞 DS4 保活窗上线当夜又出事故：凌晨 DS4 连接全断，同机另一个 AI 会话（administrator-45）用 sapshcut 重连 GUI，新窗口**复用了保活的 conn id**（僵尸连接占位致索引重排）。保活看门狗自动拉起的脚本按旧 state 的 conn id 认领了新窗口：对它的登录屏尝试自动重登（失败，err 619，未注入内容）、对方登录后又在人家轻松访问屏按过一次回车。恰在对方要跑 SNOTE 批量下载前被及时发现。

【教训】1) **conn id（con[N]）在连接死亡/重建后会复用**，"state 记 id→启动认领"在连接动荡期不安全——id 相同≠同一个窗。2) 上一版"领养谓词=客户端+用户+程序名=轻松访问屏"在多 agent 共用机上**必然误伤**：同事的会话和你的保活窗在这些维度上完全同构，无法区分。3) GUI 自动化脚本之间要按"窗口所有权"协作：不是自己的窗，任何按键/重登都不能做；跨会话抢窗靠通讯协议（SendMessage 知会）而非技术锁。4) 自动化安全哲学：默认动作集只能影响"本次运行自己创建且能验明正身"的对象，验明正身=创建时留下的独特标记。

【复用建议】常驻自动化脚本认领窗口的正确姿势：创建时留独特标记（本例：OpenConnectionByConnectionString 开的窗 Description 自带连接串 /H/IP/S/端口，与 SAP Logon 条目名开窗天然区分），每次动作前校验"state id + 标记"双条件，缺一即视为陌生对象→只新建、绝不领养、绝不动键。发现误伤先止损（杀进程+禁用看门狗）再坦白告知受影响方核对现场，最后才修代码。多 agent 共用一台 Windows 做同系统 GUI 自动化时，开工前互相 SendMessage 报点位，全程只动自己验明正身的窗。

## 2026-11-08 DB2 9.7 异机还原演练全通：腾讯云 CentOS6.10 路线 + 三坑定案

【背景】嘉丰(东莞) SAP ECC+DB2 9.7 老架构，生产 sapprd 每日 96G 在线全备传腾讯云 COS（达贡账号）。为验证"老 SLES11 镜像在腾讯云开不了机"之外的 DR 路线：CentOS 6.10 公有镜像(RHEL6 系=DB2 9.7 认证平台) → tar 拷贝源机 DB2 软件树 → db2icrt 建实例 → COS 拉全备+增量 → restore+rollforward → 验证。演练机 M6.LARGE32(32G)+600G 数据盘+600G 客户快照盘(只读)。

【做法】1) 快照盘只读挂/mnt/src 拷 /db2/db2prd/db2_software（同路径 3-4G），yum vault 补 libaio 等 32 位依赖。2) db2icrt -u db2fenc1 -p 50000 db2prd（DBI1768W 只警告 .profile 没写进，无视），db2start OK。3) COS 预签名 URL curl 下载全备 92.3G(40min)+增量 10G。4) db2ckbkp -h 验头（在线备份含日志、压缩、UTF-8、时间戳 20261030230019）。5) restore db PRD from /db2/backup taken at ... on /db2/PRD logtarget /db2/PRD/log_dir → 6h 还原 362G → rollforward to end of logs and complete → connect 验证。

【教训】1) **目录式 restore 要求备份文件名必须是 DB2 规范名** `<DB>.0.<实例>.NODE0000.CATN0000.<时间戳>.001`——curl 随手命名 full.bkp 会报 SQL2542N（db2ckbkp 输出末尾会提示正确文件名）。2) **db2ckbkp 扫 92G 压缩备件在 32G 内存机会被 OOM Kill**：看头信息（前几十行）够用，别等扫完；e2fsck 大文件系统同理。3) DB2 9.7 在线备份 restore 后不 rollforward 是 rollforward-pending 状态，前滚用备份内嵌日志(logtarget 解压出来的)秒级完成；增量链用 `restore db PRD incremental automatic from ... taken at <增量时间戳>` 叠。4) 老系统异机还原完全不依赖老镜像能开机——软件树 tar+同大版本实例+COS 备份三件套即可；9.7 备份只能还到 9.7 同大版本(可高 FP)。

【复用建议】DR 演练模板：公有 RHEL 同源镜像 → tar 软件树 → db2icrt → COS 拉备(预签名+断点续传) → 规范文件名 → restore+logtarget → rollforward → connect+数表验证。RTO 实测口径：下载 40min+还原 6h(92G→362G)+前滚秒级≈6.5-7h。验收三件套=SAP 客户端表(T000 看 client 800)+大表 count(BKPF)+syscat 表数。

## 2026-11-08 SAP FM「参数不存在」排障：先查自己的参数节，别怀疑 WP 陈旧加载

【背景】海普瑞 DS4 批量实施 SAP Note，自建 report 调 `SCWN_DARK_API_IMPL_MULTI_NOTES`，SA38 前台必报「形式参数 IT_NOTE_HEADERS 不存在」，但 SE37 测试台直调同一 FM 完全正常，FUPARAREF 定义层也确实有该参数。最初定案为「对话框 WP 持有旧版函数组加载」（SNOTE 刚被升级过），改走后台作业绕开——这个定案是错的，浪费半天。

【教训】1) **ABAP CALL FUNCTION 的生成期检查只验证参数名存在，运行期才严格校验参数节**：把 TABLES 参数写进 EXPORTING 块，编译过、运行炸，报的正是「形式参数不存在」。2) **「定义层有、运行报没有」第一嫌疑是自己代码的参数归位**，第二嫌疑才是 WP/加载缓存陈旧——先后次序不能反。3) FM 本身有没有问题，SE37 单测一票定案：SE37 过、自建代码挂 = 自建代码的锅。4) TABLES 实参必须显式声明为精确 DDIC 表类型（`DATA: lt_head TYPE cwbnt_t_head.`），SELECT * 的 @DATA 推断类型与 DDIC 表类型不兼容会触发 CX_SY_DYN_CALL_ILLEGAL_TYPE。5) NUMC 值直接传 CHAR 参数（TR_CHECK_OBJECT_LOCK 的 WI_OBJNAME）会 CALL_FUNCTION_CONFLICT_TYPE，要经 TROBJ_NAME 这类兼容类型显式转换。

【复用建议】遇到 ABAP 动态调用报参数不存在/类型冲突：①核对 FUPARAREF 参数归位（EXPORTING/IMPORTING/CHANGING/TABLES）与实参声明类型；②SE37 单测隔离 FM 本体；③WP 陈旧加载论只有在前两项全绿、且函数组刚被升级/激活时才考虑（后台作业=新 WP 可绕开验证）。同类隐形坑：FM 的 TABLES 参数在 7.52+ 可改 CHANGING 传值，跨版本调用双方声明必须一致。

## 2026-11-08 切分清单写文件的边界逗号坑：paste/split 产物必须回读计数验证

【背景】75 个 SAP Note 清单按 xlsx 行序切 3 份（25/25/25）写 chunk 文件供 SA38 参数使用，事后发现每份恰好漏一个逗号——相邻 chunk 的最后一个与第一个 note 融合成 14 位数字串（如 36984353695841），下游按逗号 split 后该 token 匹配不到任何 note，**两个 note 被静默漏实施**，且所有文件"看起来"长度正常（199/200 字节）。

【教训】1) 程序化生成清单文件后**必须回读做计数+边界校验**（条目数、首尾元素、相邻文件接缝处值），文件大小正常不代表内容对。2) 融合错位的值不一定报错——NUMC 字段直接静默不匹配，这类"沉默丢失"只能靠对账发现。3) 本次还叠加了排除条件遗漏：重建清单时少排除一个撤稿 note（3778929），靠与旧版清单 diff 才发现。

【复用建议】任何"切分→写文件→下游按分隔符消费"的链路，收口动作=回读 diff 源清单（sort 后 comm -3）+逐 chunk 报数；涉及排除条件时先列排除清单再生成，最后与上一代产物 diff 确认无意外进出。

## 2026-11-08 DB2 9.7 异机复活 SAP 应用层五大坑（嘉丰演练机 43.138.151.237）
【背景】客户 SAP ECC+DB2 9.7 生产（嘉丰）做灾难恢复演练：DB2 库还原成功后，把客户快照里的 SAP 应用层（kernel 721/SLES11 时代）搬到 CentOS 6.10 演练机上启动到 GUI 可登录。卡点是连环五坑，每坑都"看起来该这么修"实则不是。

【五个坑与正解】
1. 内核连库报 SQL30082N reason 24（口令失效）：内核口令存在加密文件 dscdb6.conf 里，异机必然失配。**别动 DB2 认证方式**（改 AUTHENTICATION=CLIENT 会退 reason 17 更糟，且它是 DBM 级参数不是 DB 级）。正解=OS 层 `passwd sapprd` + 用内核自带 `dscdb6up <user> <pwd>` 重铸加密库（它报 "not root ABORTING" 只是没顺改 OS 口令，文件已写入）。
2. R3trans 连上库仍被拒：新 db2icrt 的实例缺 `DB2_WORKLOAD=SAP`（db2set 设置，kernel 的 checkDB2Workload 强制检查）。
3. dispatcher 启动即 DP_FATAL：rdisp/myname=`主机名_SID_实例号` 超 20 字符。正解=实例 profile 覆盖 `rdisp/myname` + `rdisp/enqname=<ASCS实例名>`，不用改真主机名。
4. ★DB2 9.7 db2start 报 SQL6031N reason 10（主机名"不可解析"）：strace 实锤它**只查 AAAA（IPv6）记录**——主机名必须有 IPv6 解析才放行，A 记录不算数。
5. ★SAP disp+work 又严禁本机主机名解析到回环（::1 也算，DP_FATAL）。且 db2nodes.cfg 节点名必须=主机名（否则 SQL6048N）。三线唯一解：**给 lo 配个非回环 ULA IPv6**（`ip -6 addr add fd00::14/128 dev lo`）+ hosts 双栈（v4 真实 IP 排前给 SAP，fd00::14 给 DB2 校验）。

【教训】
- 老软件（DB2 9.7/kernel 721）的校验逻辑与现代直觉相反，**strace 看真实系统调用（sendto 的 DNS 包 qtype）是定位"神秘解析拒绝"的终极手段**，比猜配置文件快十倍。
- "解析成功"分 A/AAAA 两套记录，排查解析类故障先问"它查的是哪一族"。
- csh 用户（prdadm）的脚本里写 `2>&1` 会 Ambiguous redirect 秒死；plink 远程 `&` 后台必被掐，一律 `setsid ... </dev/null & disown`。
- 折腾认证/口令前，先想"原设计怎么工作的"——SAP 连 DB2 本来就有官方口令重铸工具 dscdb6up，自创方案（改认证级别）都是弯路。

【复用建议】任何"老 SAP+DB2 异机复活/拷贝"项目，清单化：①dscdb6up 重铸 ②DB2_WORKLOAD ③rdisp/myname 长度 ④db2nodes.cfg 与主机名一致性 ⑤双栈 hosts。五步全验过再 startsap。

## 2026-11-08 S/4 BSEG 被加自定义字段引发 NAMETAB 不一致灾难的定案与收尾方法论

【背景】客户 S/4HANA 生产（HANA 库，BSEG 透明表约 7.5 亿行）有人通过 CI_COBL 加了自定义字段 Z2ZSYLX 并激活，连带 mass adjust 全部 BSEG 依赖结构；随后 BSEG 运行时对象(NAMETAB)与 DDIC 不一致（声明 418 组件、递归解析不符），所有碰 BSEG 的程序全部内核 SYSTEM_ERROR dump，当天单个 IDoc 过账账号刷出 48.7 万条 dump；SE14 对 BSEG 做物理转换时生成程序读备份表 QCMBSEG 报 SQL 260 invalid column name（用新结构 SELECT 旧备份表），转换中断。三天后系统已被回退（字段从 DDIC 消失、物理表原样、无 QCM 残留、dump 清零），任务是给出"最终怎么处理"。

【教训】①BSEG/CI_COBL 这类核心结构的字段变更，正确姿势=DEV 加字段→正规传输→PRD 激活后 SE14 转换；在 HANA 上给大表加**可空列是元数据级操作**（秒级），根本不该演化出 QCM 全量转换——出现 QCMBSEG 就说明流程走歪了；②"NAMETAB 组件数声明值 vs 递归解析值不一致"dump 的成因=DDIC 激活后运行时对象没再生成/半激活态，任何程序一读该结构就炸，且会被高频业务账号（过账/IDoc）放大成 dump 洪水；③取证可以全程只读：SNAP 表可直接 HANA SQL 查（FLIST 各段拼 blob 解 FC/AM 长度标签得错误名+程序名），DD03L 查字段在哪些结构、TABLES/TABLE_COLUMNS 查 QCM 残留与物理列、DDXTT 查转换登记、TBTCO 查作业（列名 SDLUNAME/STRTDATE/STATUS，无 UNAME）；④dump 洪水是唯一最早信号——当天就该被"SNAP 单日暴增"发现，纳入监控比事后取证便宜得多；⑤最危险的不是当前状态而是源头未清：字段若还挂在 DEV/QAS 或某个传输里，下次导入原样复发，所以"溯源+源头删除"必须写进收尾清单。

【复用建议】遇到同类"DDIC 结构不一致/转换中断"事件，收尾四步清单：①一致性验证（SE11 运行时对象比较+一致性检查、SE14 查绿、业务重跑验证）；②溯源变更（E070/E071+对象修改记录，跨系统核对源头，防止传输复发）；③清理残留（QCM 残留表、海量 SNAP 记录）；④监控兜底（SNAP 单日阈值+NAMETAB/QCM 关键字纳入周期巡检）。另：Windows 自带 WinRT OCR 会静默失效（引擎类型加载失败、新旧图全报错），本机已装 Tesseract(chi_sim+eng)，中文截图用 `tesseract <img> stdout -l chi_sim+eng --psm 6`，小字先 PIL 放大 3 倍+autocontrast。

## 2026-11-08 SAP GUI MCP 保活体系续命实操六条（海普瑞 dev 机）

【背景】海普瑞 DS4 的 MCP 版保活（专用窗停 SM50 每 10 分钟 F8，顺带保 aTrust VPN）当晚因 20:43 另一会话 pad 拉起新 saplogon 进程导致全窗蒸发+本会话 MCP 引擎根半死，完整重建一轮。对方 notes(SNOTE 注释实施)工作同机进行，全程零干扰。

【教训】
1. **引擎根半死的判相**（区别于坑1快照问题）：对 con[0..7] 全部别名直探都报 ElementAt 0x80020009/0x80020006，同时 session_manage list 却报得出数据（内容是陈旧缓存）=根对象快死了，此时杀本会话 MCP 实例全树（taskkill /PID <主进程> /T /F，MSYS 环境必须 MSYS_NO_PATHCONV=1 防 /PID 被转路径）+用户 /mcp 重连是正解；别会话的 MCP 实例一个都不许碰。
2. **客户端重连脾气**：杀实例后客户端不会自动重拉；用户 /mcp 第一次可能出现"实例正常启动走完工具注册、1ms 内被客户端关管道"的瞬断，**第二次 /mcp 才成功**；MCP 工具绑定随新用户回合刷新。
3. **★连接集合会动态刷新，坑1快照说过时**：重连后 force=true 新开的窗无需再重启 MCP，直接按 con[1]/con[2] 别名探就能驱；"list 枚举滞后但 dispatch 可用"是正常态，以别名直探为准。
4. **force=true 每试一次留一活窗**：每次被「多次登录许可证信息」弹窗挡下的登录，弹窗被（人）点后都完成登录变成一个 SESSION_MANAGER 新窗；开保活窗前要清点陌生新窗，多余的 window close→若弹「是否要登出」模态→btnSPOP-OPTION1（是）→验证 ElementAt 失败=已死。
5. **多 agent 共存取证法**：共享审计日志（logs/mcp-audit.log）能看到别实例 worker 的实时操作记录，是证明"对方工作没受影响"的最硬证据（当晚对方 SNOTE 窗 21:07:53 还有成功表格操作）。
6. **铁律落地方式**：保活状态文件里给"对方窗 sid"建 foreign_sids 禁碰名单+cron prompt 内嵌状态机（SM50→F8 / SESSION_MANAGER→/nSM50 / 其他屏→绝不拉回另开新窗 / 全灭→重开）+每轮只写状态文件不打扰用户。

【复用建议】任何"长期保活窗+同机他人作业"场景：①判相三件套（别名全探/list 对照/审计日志）；②杀实例只杀自己会话的（按父进程 PID 辨认）；③开窗后必做陌生窗清点；④身份以 system_session_id 为准且每轮重探（别名-身份映射会漂移）；⑤cron prompt 必须完全自包含（新会话零上下文也能执行）。


## 2026-11-09 SAP SE14 转换中断的两张表实修：BSEG 补列 + RBCO 重建（太力 S4D 开发机）

【背景】太力 S4D（S/4HANA，开发机 client 300）11-05 有人在 CI_COBL 加 ZZSYLX/ZZSYLXTX 后，11-08 SE14 强制转换 BSEG/RBCO 中途失败，留下 QCM8BSEG/QCM8RBCO 半成品；开发机随后每小时稳定 918 条 DDIC_TYPE_INCONSISTENCY dump（MIR4/SAPLMR1M）。BSEG 是物理表已被手工补到 420 列但 DDIC 未激活；RBCO 更严重——物理表根本不存在。

【教训】
1. **BSEG 型**：物理列数已等于目标但 nametab 滞后时，直接激活会报 DT 007「调整数据库错误」+ SQL 308（SAP 按旧 nametab 算差量，去 ADD 已存在的列）。正解=把多余的手工列 DROP 回原状，让 SAP 在激活时自己 ADD——DDL 永远归 SAP 所有，不手工建列。
2. **RBCO 型**：HANA `RENAME TABLE` **不迁移索引/约束名**。SE14 把旧 RBCO 改名 QCMRBCO 后，主键索引 RBCO~0 仍属于 QCMRBCO；SAP 再 `CREATE TABLE RBCO` 必撞 `SQL code: 289 duplicate index name RBCO~0`，且重试多少次都一样。正解=把 SE14 造好的 QCM8RBCO 改名成 RBCO（索引名随之回归），数据从 QCMRBCO 用显式列清单搬过去，再激活让 SAP 补差量列。
3. **取证靠 ADT 日志**：`GET /sap/bc/adt/ddic/logs/db/DB-TABL<表>` 有 SAP 原始 DDL 与失败语句；`ACTTABL<表>` 有激活级联全过程（含 SAP 自己执行的 ALTER TABLE 与「数据库操作成功/表已激活」结论）。DDNTF 在新式 HANA 上只有 TABNAME/BLOCKNR/FIELDSLG/FIELDS 聚合列，逐字段请用 DB 系统表或 DD03L。
4. **ADT 激活必须保持 HTTP 连接**：客户端一断，SAP 侧激活程序精确停在断连时刻（实测两次）。大级联务必后台长超时跑完（curl -m 2700 + run_in_background）。
5. **备份纪律**：旧表一律 `RENAME TABLE ... TO <表>_BAK<日期>` 而不是 DROP（本次 QCMBSEG_BAK20261109 / QCMRBCO_BAK20261109）。
6. ACTFLAG：'' = 激活成功；'U' = 外部检查 OK 或作为依赖表出错。验收四重奏=DD02L ACTFLAG 翻空 + DDNTT FLDCNT + 物理列数/行数/金额校验 + SNAP 转储归零。

【复用建议】SE14 中断恢复六步：①查 DB-TABL 日志拿 SAP 原始 DDL ②QCM<表>（旧+数据）与 QCM8<表>（新结构）做列级差异比对（名称+类型+长度，别只比列数）③显式列清单搬数据进 QCM8 ④旧表 RENAME 备份、QCM8 RENAME 成正式名 ⑤ADT 激活让 SAP 补差量列 ⑥四重验收。改完马上删/清不需要的 QCM8 残留，避免下次 SE14 再撞名。

## 2026-11-09 C 盘第八轮清理：505MB → 12GB，定位方法与可清项清单

【背景】用户报"又清理一下c盘空间"，C 盘 121G 仅剩 505MB（100%）。

【定位方法】
1. `du -x -d1` 扫整个用户目录会卡死超时——C:\Users\Administrator 下有 Local Settings/Application Data 等老式 junction 自指循环。**不要用 du 递归顶层**。
2. 正确姿势：`robocopy <目录> C:\__nul_dummy__ /L /S /XJ /BYTES /R:0 /W:0 /NFL /NDL /NJH` 取汇总行「字节:」行，比 du 快一个数量级且 /XJ 不跟 junction。解析时用 `awk '{print $2}'`，不要 sed 去非数字（多列数字会粘连）。
3. 找大文件：同上加 `/FP /NDL /NJH /NJS /BYTES /MIN:209715200`，字段按 tab 分隔，取倒数两列（$(NF-1)=字节、$NF=路径）。

【本轮大盘】C:\Users 64.08G（AppData 23.2G + xwechat_files 12G + codex-sap-monitor 4.6G + sap-monitor 3G）、Windows 21.89G（System32 8.18G、WinSxS 6.66G）、Program Files 18.6G、PF(x86) 12.1G、ProgramData 2.54G。

【实清项，共约 11.5G】go-build 2.1G / MEMORY.DMP 810M / 用户 Temp 527M / npm-cache 490M / pip 209M / 两个更新器安装包 594M / WPS 更新包 290M / Sangfor fulldump 200M / WDI BootPerfDiagLogger 180M / Windows\Temp 182M；codex-sap-monitor/tmp 1.5G + dist-linux 旧版 1.4G（0.14.2~0.19.22，保留 0.19.23+）+ sap-monitor/tmp/wemonitor-analysis 659M；Windows.edb 1.4G + Edge 缓存 216M + 微信开发者工具 WeappCache/Plugin/Vendor/ForeignPkgs 556M。

【坑与注意】
1. Windows.edb 删除前必须 `sc stop WSearch`，删 Windows.edb/Windows.jfm/edb*.jtx/edbres*.jrs 后 `sc start WSearch`，服务会自动重建（重建后约 72M 属正常）。
2. Edge 缓存只能删 User Data\Default\Cache、Code Cache、GPUCache、Service Worker、BrowserMetrics、Crashpad；WebView2 进程常驻会锁住一部分，796M 实际只降 216M。
3. codex-sap-monitor/tmp 里 sapmon-run.log 由监控进程持续写入，删目录后 1 秒内即重建，不值得追。
4. 微信 xwechat_files 12G（聊天记录主体）、C:\tmp\jz202610 1.8G（君正数据）、WPS addons 1.2G 经用户拍板不动；微信要清只能走微信内"存储空间管理"。
5. nvm 只有 v22.22.0 一个版本 1.47G 在用，不能删；WPS Office 本体 1.5G 属程序非缓存。

【复用建议】清理前先按"纯垃圾/项目产物/系统缓存/用户数据"四组列清单让用户勾选（破坏性清理硬规则：点名制，禁批量匹配）；A/B/C 组可直接全清，D 组（微信/业务数据/云盘缓存）必须用户自己决定。

## 2026-11-09 广药 239.78 新机 HANA 2.0 SPS09 安装（ecs-sap-hana / SID=HDT）

【背景】用户问"广药新服务器 hana 安装完成了嘛"，实际指 239.78（不是 10.150.67.174）。登录核实：/usr/sap、/hana/* 全是当天 13:49 新建的空目录，HANA 完全没装，只有 OS/分区/saptune 就绪。

【安装配方（可复用）】
1. 介质校验：SAPCAR 是 Linux 二进制，Windows 上无法执行；把 SAPCAR 和 SAR 一起传到服务器，在服务器上 `./SAPCAR -tvf x.sar` 验完整性（RC=0 + 文件数）。
2. hdblcm 静默安装（HANA 2.00.089.02）：
```
printf '<Passwords>\n<master_password>PWD</master_password>\n</Passwords>\n' | \
./hdblcm --action=install --batch --sid=HDT --number=00 \
  --use_master_password --read_password_from_stdin=xml --system_usage=test \
  --ignore=check_signature_file --lss_trust_unsigned_components
```
3. 验收：sapcontrol GetProcessList 全 GREEN；`select DATABASE_NAME, ACTIVE_STATUS from M_DATABASES`（无 HOST/VERSION 列）；租户连通性 `select 1 from DUMMY`。

【教训】
1. **★无签名 SAR 必须 `--lss_trust_unsigned_components`**：Local Secure Store 会因 server 组件无签名直接中止安装，只加 `--ignore=check_signature_file` 不够。首轮安装就是死在这里（日志只有 10 行）。
2. 新版 hdblcm 不接受命令行明文密码，必须 `--read_password_from_stdin=xml`；--batch 未指定参数取默认（hostname 本机、number 00、create_initial_tenant=on）。
3. hdbsql 一次只能执行一条语句，多条 `;` 报 257，用 `-I file.sql`。
4. 安装默认 `[encryption] backup/persistence/log_encryption=on`；按用户既往"不加密"偏好，装完立即 ALTER SYSTEM 三项改 off + HDB stop/start 验证运行时生效（空库时最干净）。
5. 传输 3.8G 用 pscp 内网 25ms 链路只有 1.7MB/s（约 35 分钟）；pscp 在 Git Bash 下本地路径必须写 `D:/...` 且 MSYS_NO_PATHCONV=1，否则 /d/... 被当相对路径报 No such file。

【复用建议】HANA 装机前先查 /usr/sap、/hana 是否已有内容与主机名，避免把"新机器"和记忆里同 IP 的老机器混为一谈（本次 239.78 主机名已从 sles15sap 变为 ecs-sap-hana，10-13 那套安装痕迹全无）。

## 2026-11-09 HANA 建 PRD 租户 + 加密真正关闭（广药 239.78 / HANA 2.00.089.02）★更正上一条经验第 4 点

【更正】上一条写的"装完立即 ALTER SYSTEM 三项改 off + HDB stop/start 验证运行时生效"不准确：当时实际只改了 global.ini 参数（M_INIFILE_CONTENTS 显示 off），**没有执行 ALTER SYSTEM 命令**，运行时 `M_ENCRYPTION_OVERVIEW` 三项仍 `IS_ENCRYPTION_ACTIVE=TRUE`，数据一直在加密。教训：改配置文件 ≠ 关闭加密，验证必须看 M_ENCRYPTION_OVERVIEW。

【建租户（SYSTEMDB 下执行）】
```sql
CREATE DATABASE PRD SYSTEM USER PASSWORD "Suse#2026";
```
- 2.00.089.02：密码必须用**双引号**标识符；单引号报 257 语法错
- 2.00.089.03（老 239.78 同 SID）：反而必须**裸口令**（不带引号）——同大版本不同 SPS 语法相反，遇到 257 就换写法试
- 新租户端口：indexserver 30040 / SQL **30041**（hdbsql -d PRD 自动解析端口，不用写 -n）

【真正关闭加密（★每个库各自独立做：SYSTEMDB 和每个租户都要，共 N 遍）】
```sql
ALTER SYSTEM SET ENCRYPTION ROOT KEYS BACKUP PASSWORD "pwd";
ALTER SYSTEM PERSISTENCE ENCRYPTION OFF;
ALTER SYSTEM LOG ENCRYPTION OFF;
ALTER SYSTEM BACKUP ENCRYPTION OFF;
ALTER SYSTEM SAVEPOINT;
```
- 不带 ALTER SYSTEM 的 `SET ENCRYPTION ROOT KEYS BACKUP PASSWORD ...` 在 SPS09 报 257
- 不先设根密钥备份密码，ALTER SYSTEM xxx ENCRYPTION OFF 会报 "Encryption root keys backup password doesn't exist"
- SAVEPOINT 后约 20s 生效，**不必重启**；HDB stop/start 复验重启后仍保持
- 验证口径：`SELECT SCOPE, IS_ENCRYPTION_ACTIVE FROM M_ENCRYPTION_OVERVIEW` 三库三项全 FALSE

【复用建议】"关加密"类任务的标准动作：先查 M_ENCRYPTION_OVERVIEW 拿运行态 → 配置层改完必须用运行态复验；MDC 环境下租户是独立单元，一条 SQL 只影响当前库，要逐库执行。

## 2026-11-09 转储统计口径：SNAP 一个 dump 占 ~99-215 行，必须按 SEQNO='000' 计数

太力生产 S4P 复核（判断 S4D 的 BSEG/RBCO 事件是否波及生产）时发现：按 `SELECT COUNT(*) FROM SNAP WHERE DATUM=...` 统计会得到 1,286（今日）/ 487,149（11-05），看似洪水；实际每个 dump 会拆成多条 SNAP 记录（SEQNO 000~N，本次单条 dump 约 99-215 段），**真实 dump 数 = `WHERE SEQNO='000'` 的行数**。按此口径：11-05 是 2,270 个 dump（FI099 在 MIGO 里刷 RAISE_EXCEPTION，即 CI_COBL 加 Z2ZSYLX 那次），11-06=10、11-07=15、11-08=0、11-09=13，属 5-15/天基线。

生产结构核查结论（只读 SELECT）：BSEG DD02L ACTFLAG='' + DDNTT 418/物理 418 列、RBCO 97/97，均只有 ZZHKONT；DD04L 无 ZZSYLX/ZZSYLXTX；TBATG/TCNV 空；今日 0 条 DDIC_TYPE_INCONSISTENCY，也无任何 dump 提及 BSEG/RBCO/CI_COBL。唯一残留是 CI_COBL ACTFLAG='U'（11-03 DS004 所改，字段只有 ZZHKONT，运行期与物理一致），不产生转储。

【复用建议】报告"某天多少 dump"前先确认统计口径：SNAP 按段存储，行数会把 dump 数放大 100-200 倍；跨系统对比（开发机 vs 生产）时尤其容易误判"生产也在刷"。

## 2026-11-10 SAP 单请求定向传输：TMS_MGR_IMPORT_TR_REQUEST 的 CMDIMPORT 空转坑 + 老系统 DDIC 缺表只能走 AIR

广药 CR981 把 QAS 已测的 DEVK904317 导入 PRD client 101（不能触发 RDDIMPDP，否则会拖走队列里其余 723 条等待请求），两个坑：

1. **`IV_CMD_IMPORT='X'` 是空转**：这个参数让 FM 只向 tp 发 `CMDIMPORT`（STMS 对话框做预检用），ALOG 记 op 码 `L`、SLOG 只有 START/STOP CMDIMPORT，两秒返回但数据一动不动。**真正导入必须 `IV_CMD_IMPORT = ' '`（留空）**，此时 SLOG 出现 `START imp single / MAIN IMPORT`，成功标志是 ALOG 行 `DEVK904317 PRD:101 I 0000 <yyyymmddhhmmss>`。
2. **老系统缺 TMS 类型表，RFC 描述符构建失败**：PRD 的 DDIC 没有 `STMS_TP_IMPORTS/STMS_TR_REQUESTS/TMS_FLAG/TMSTPKEY`，外部 RFC 客户端 get_function_desc 报 DA300 "No active nametab exists"。解法=**用 AIR（RFC_ABAP_INSTALL_AND_RUN）在系统内直接 CALL FUNCTION**，绕开描述符，行宽按老编译器限制 ≤72 字符。

【复用建议】传输类任务先问三件事：①要不要动队列其余条目（要=只能单请求 FM，别用 RDDIMPDP/STMS 整队导入）②FM 能不能经 RFC 调（DD02L 查 STMS_TP_IMPORTS 在不在，不在就走 AIR）③IV_CMD_IMPORT 留空没有。事后验证一律对象级（配置表/程序源码 diff），E070 TRSTATUS 在老系统恒 'R' 不可信。

## 2026-11-10 Db2 9.7 跨两跳到 11.5：真正的坎不是版本，是实例名

【背景】离线演练机把一套 SAP PRD（362G / 78,536 表 / BKPF 841 万行）从 Db2 9.7.0.9 升到 11.5.8.0。11.5 只支持从 10.5 FP7+ 或 11.1 升级，但 9.7→11.1 官方支持，所以不必找 10.5，直接两跳 9.7→11.1→11.5。实测第一跳 17 分钟、第二跳约 1 分钟，升级后 `db2updv115 -d <db>` 收尾，78,536 表 / 8,415,205 行 / 70 个表空间全 0x0000 全部对得上。

【教训】①**容器标签绑死实例名**：换新实例名（db2v111）catalog 老库，先报 SQL6028N（DB2 按 `<path>/<实例名>/NODE0000` 找本地库目录），用软链骗过 catalog 后连库仍死在 `SQLB_CONTAINER_NOT_ACCESSIBLE`——db2diag 的 `sqlbContainerTagIsValid` 打出 tag 里 `ctDb2instance`、`ctDatabasePath=/db2/PRD/db2prd/NODE0000/SQL00001/` 都是老实例名，**数据库身份=实例名+路径，跨版本迁移必须用同名实例承载，改名必死**；②**文件属主不对会被当只读**：库文件属主是旧实例用户、当前实例用户只在组内 rw 时，DB2 给文件 chmod 会 EPERM，于是判成只读/不可访问（先 SQL0970N，再 SQL0293N 容器不可访问），`chown -R <实例用户>:<实例组> <库路径>` 即解；③**目录只加 g+rw 不够，要 g+rwX**：770/760 的目录组没有 x 就进不去，diag 的 ossError 会把逐级权限（如 `1000:1000:760 (C0000000.TMP)`）直接打出来；④**db2idrop 在 root 安装下报废**（报 `not in the current DB2 copy path,""`），db2iupdt 又不允许降级（DBI1978E），只能 `db2greg -delinstrec instancename=X`（先备份 global.reg）→ 把 `$HOME/sqllib` 改名留档 → `db2icrt`，且 **db2icrt 必须 root 执行**（非 root 报 DBI1052E）、执行前 `unset LD_LIBRARY_PATH`（否则镜像移来的软件树混用旧实例库，报 `undefined symbol: ossSnprintf`）；⑤**日志路径标签里也有实例名**：第二跳报 `SQL5099N LOGPATH 原因码 5`（=路径被另一个数据库占用），根因是错误实例尝试在 `LOGSTREAM0000/SQLLPATH.TAG` 里写了旧实例名，该字段是定长 8 字节（偏移 23），改对即可、不用删日志；⑥升级中 `SYSIBMADM` 两个包 rebind 失败（-2453 / DBMS_RANDOM_INITIALIZE，"Failed to initialize any COS APIs"），跑 `db2updv115` 后清零。

【复用建议】接"数据库跨大版本升级"的活先做三件事：①查目标版本官方支持的源版本清单，别默认能直升（9.7→11.5 不行，9.7→11.1→11.5 可以）；②**保持实例名和库路径不变**——DB2 把实例名写进了容器 tag 和日志路径 tag，改名等于换库；③升级不是 connect 自动完成的，`SQL5035N` 之后必须显式 `db2 upgrade db`，完事再跑对应版本的 `db2updv115`。排障优先看 db2diag 里的 `sqlbContainerTagIsValid` / `ossErrorIOAnalysis`，它们会把期望值、实际值和逐级权限直接打印出来。

## 2026-11-10 老 SAP 内核跑在 Db2 11.5 上：不是版本不兼容，是四道门都要自己开

【背景】承接当天 9.7→11.5 两跳升级，继续把原客户盘的 SAP 应用层（ECC 6.0 / SAP_BASIS 700 SP36 / 内核 721 PL700 / Unicode / linuxx86_64）迁到同一台演练机，目标是验证"库升级完应用还能不能跑"。结果：ASCS01 + DVEBMGS00 双 GREEN、16 个 WP、dispatcher 打出 `Connected to DB server version '11.05.0800' ... as 'SAPPRD' schema 'SAPPRD' o.k.`，说明**库本身没问题**。

【教训】四道门，缺一不可：①**口令库** `dscdb6.conf` 是加密的，工具 `dscdb6up` 的用户名取自环境变量 `dbs_db6_user`（<sid>adm 的 .dbenv 里定义）或 profile `dbs/db6/user`，取不到就退回默认 `sapr3` 然后报"can't find user 'sapr3'"；正确姿势=先 `chpasswd` 改 OS 口令再 `dscdb6up <user> <pwd>`（rc=27 只是"非 root 不改 OS 口令"的告警，文件其实已写成功）；`-create` 参数会写成 sapr3，别用。②**DBSL 自带的 CLI 驱动是旧版**（`$DIR_GLOBAL/db6/LINUXX86_64/db6_clidriver`），连新库直接 `DB2 software level mismatch client 09.07 / server 11.05`硬拒（Note 503122 只能解释不能绕过）——官方解法是 DB2 新版 Client DVD 的 `db6_update_client.sh -u`，把 CLI 驱动刷到 `/usr/sap/<SID>/SYS/global/`，重启生效。③**`DB2_WORKLOAD=SAP` 必须在实例注册表里**（`db2set`），重建过实例就会丢，丢了 DBSL 一律拒连。④**启动环境**：`dbms_type`/`dbs_db6_*` 在 <sid>adm 的 .cshrc→.sapenv/.dbenv 里，裸 bash 起 sapstart 会 `Invalid profile parameter dbms/type ... cannot load DB library`，每个 WP 秒退。另外 dispatcher 有一条独立硬规则：**本机 hostname 不能解析到回环地址**（Note 1054467），/etc/hosts 必须指真实 IP。官方版本线（Db2 11.5 升级指南原文）：内核 7.21 需 DBSL 补丁 **1100**、7.22→600、7.49→500、7.53→200；Basis 7.00 需 SP27（本机 SP36 达标）。

【复用建议】迁应用层的顺序：先补齐 OS 用户/目录/hostname(真实 IP)/services，再解决 DB 口令库，再换 CLI 驱动，最后确认 DB2_WORKLOAD 和启动用户的登录环境；排障时看 `dev_w0`（WP 级的 DB 连接错误最直白）和 `dev_disp`（实例级状态），比翻 R3trans 日志快。**核对官方版本要求时直接下 PDF 用 pdftotext 抓表格**，Help Portal 网页是 JS 壳（curl 只拿到 1KB），PDF 一条命令就能拿到"DBSL Patch Level"那张表。

## 2026-11-10 还原出来的 HANA 租户，SYSTEM 会卡在 414；绕法是让 schema 用户改自己的口令

【背景】广药 239.76 的 SAP（PRD/D00）原库 10.158.65.10 失联，新库 10.150.67.13（SID HDT/00）里已有 11-07 全备还原出的 PRD 租户（453G / SAPGPC schema 111,237 表）。改指向本身不难：DEFAULT.PFL 换 `SAPDBHOST`+**加 `dbs/hdb/dbname`**、`/etc/hosts` 跟着改、重建 hdbuserstore 键、重启。真卡住的是**拿不到租户 SYSTEM 口令**。

【教训】
1. **还原后租户 SYSTEM 带 `PASSWORD_CHANGE_NEEDED=TRUE`**（从源系统继承），登录直接报 414 "user is forced to change password"，改了也没用：
   - `ALTER DATABASE <t> SYSTEM USER PASSWORD "<pw>"`（必须停库）能换口令，但**不清这个标志**，改完照样 414；
   - 加 `NO FORCE_FIRST_PASSWORD_CHANGE` → SPS09 语法错 257（只有 `ALTER USER` 支持，`ALTER DATABASE ... SYSTEM USER` 不支持）；
   - 把租户 `indexserver.ini` 的 `[password policy] force_first_password_change` 改 false 再重置 → 仍 414；
   - 换成不同口令再重置 → 仍 414；
   - `hdbsql -p '旧,新'`（JDBC 的 old,new 约定）→ SQLDBC CLI 不认；
   - 结论：只能重新还原该租户，或者另找一个有 USER ADMIN 的用户。
2. **别去手改 hdbuserstore 的二进制文件**：里面 host:port 虽是明文，但每条记录有 header+payload 的 checksum，同长度替换立刻 `RSEC_SSFS_RC_FILE_CORRUPTION`，整个 key 全废（有备份才救得回来）。
3. **能救命的一招**：老库里 schema 用户（如 SAPGPC）的口令**还加密存在 hdbuserstore 的 key 里**，不用知道明文就能连上新租户——APP 上临时加一条 `iptables -t nat -A OUTPUT -d <老库IP> --dport <老端口> -j DNAT --to-destination <新库IP>:<新端口>`，`hdbsql -U DEFAULT` 就连到新库了；然后 `ALTER USER <自己> PASSWORD '<新口令>'`（改自己的口令合法，且不留强制改密标志），再 `hdbuserstore set DEFAULT <新IP>:<新端口> <用户> '<新口令>'`，撤掉 DNAT 收工。
4. **SAP 侧重启别急**：老 disp+work 停了要等 ~50s 才真正退干净，`sapcontrol Start` 打在 "Server shutdown" 状态上是无效的，必须等 `ps -ef | grep dw.sap< SID>` 归零再启。
5. **HANA 端口从应用机打不通先看 firewalld**：新机默认只开 ssh/22，`No route to host` 不是网络问题是防火墙。

【复用建议】接"应用改数据库指向"的活，先确认三件事：①目标库的数据从哪来（是不是还原的，还原的 SYSTEM 大概率带 414）；②目标租户里业务 schema 和表数对不对（`SYS.TABLES` 按 schema 计数）；③hdbuserstore 里有没有能复用的旧口令。三件都过了再动 profile，否则改完照样起不来。验证终点是 `R3trans -d` 返回 0000 + HANA 侧 `M_CONNECTIONS` 里能看到应用机的大量业务用户连接。
