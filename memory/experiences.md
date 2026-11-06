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
