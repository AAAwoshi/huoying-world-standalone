# Nya 开发快照（单机版）全量分析报告

> 对象：[res/火影世界单机版.zip](../../res/火影世界单机版.zip)（**2.1.x 开发期快照**）736,913,093 B｜SHA-256 `52388C6284A8637B33A103CACE700B493FB035DD00CC46CFBBBDA111178E2505`
> 解包位置：`C:\Project\hy_analysis\x\单机版`（下称 **T2**）｜实测 **6,253 文件 / 1,027,802,028 B（980.2 MB）**
> 基线：2.1.1 v8（`C:\Project\hy_analysis\x\Nya211\Nya-2.1.1-本地一键端-无存档-手动云更新-20260805-v8`，下称 **T1**）｜分析日期：2026-10-07
> 姊妹篇：官方发行线见 [03-Nya官方线.md](03-Nya官方线.md)；上游总纲见 [源码对比分析.md](源码对比分析.md) §10（Nya 线源码级全量差异）。

---

## 0. 结论速览

1. **这不是发行版，是作者（与朋友）边玩边开发的工作目录原样打包**：相对 T1 基线 **5,707 文件完全一致 / 10 个内容改动 / 536 个新增（+231.7 MB）/ 22 个删除（−10.4 MB）**。
2. **功能大爆发**：`tools/gm_admin.py` 34,699 → **164,325 B**（动作 14 → **59**）、`www/gm/index.html` 28,425 → **108,342 B**（页签 4 → **9**）、`tools/fake_flash_server.py` 2,471,309 → **2,711,058 B**（+239,749 B，+37 个顶层符号、+8 个 opcode 枚举）。
3. **SQLite 双后端实验**：新增 `tools/storage_backend.py`（8,340 B，JSON/SQLite 双实现）+ `tools/migrate_saves_to_sqlite.py`（762 B），落盘 `save/game.db` **2,170,880 B** + `game.db-wal` **4,556,752 B**，`documents` 表 **20 行**，最后写入 **2026-08-08 22:51:15**（该实验在 2.2.0 被整体放弃）。
4. **携带他人真实游玩存档**：4 个账号 `qwe104510988`/`zxc104510988`/`asd104510988`/`qweqwe`，角色 1001~1004（最大 `1002.json` **471,373 B**），含**两账号结婚**（2026-08-06 21:13:30）、鲜花 99/99/198、家族「木叶村」资金 **1,659,654,240**、单角色在线 3 小时；另有整套 `save_backup_20260807-221025/`。SQLite 库内比平面 JSON **更新**，还多出第 5 个账号 `admin` 与 3 个测试角色。
5. **GM 鉴权被整体关闭**：`static_flash_http.py:535-537` `_require_gm_auth()` 直接 `return True`（docstring 自述 *"Local single-player GM backend skips password verification."*），前端 `www/gm/index.html` 硬编码默认口令 `NyaLocal#208` 免输入。
6. **曾以公网模式运行**：`logs/local-server.out.log` 显示 843/19280/19281/19284/19285/19286 全部绑 **0.0.0.0**，并向客户端公告 **27.184.1.224:19285**——与包内 `PACKAGE_MANIFEST.json` 声明的 `"networkMode": "localhost-only"` 直接相悖；根因是启动器默认 host 解析（见 §6.4）。
7. **完整的 SWF 逆向工作流随包**：`tools/ffdec/` 78 文件 21.4 MB + `tools/jre17/` 316 文件 124.8 MB + `jre17.zip` 43.8 MB + `tools/ffdec_work/` 残留反编译产物 + **10 个带时间戳的 MainLib 备份**（08-06 19:16 ~ 08-08 20:29）+ 7 个体验服 `act_web_tiyan/swf/UISwf/` 素材。
8. **12 个开发脚本全部硬编码开发机路径**（`f:\Nya-2.1.1…`，10 个文件命中，`run_tasks.bat` 一处 11 次），**无一个进入 2.2.0/2.2.3**——这批文件是「T2 是开发沙盒」最直接的物证。

---

## 1. 身份与来源判定

| 判据 | 实测 |
|---|---|
| 版本元数据 | **`.nya-version.json` 已被删除**（`pair_211__dj_removed.txt` 第 1 行，144 B）→ 依赖 versionCode 的云更新检查在开发期失效 |
| 包清单 | `PACKAGE_MANIFEST.json` 仍在，且与 T1 **逐字节相同**（1,311 B，`networkMode: localhost-only`、`includesSaves: false`、`gmPassword: NyaLocal#208`） |
| 启动链 | 10 个顶层 bat + `Nya本地登录器.exe`（1,847,792 B / 1.3.0）**与 T1 逐字节一致**（`cmp` 12 项全部 SAME） |
| 删除项 | `NyaLauncher/` C# 源码 **15 个文件全部消失**（只留编译好的 exe）、`使用说明.txt` 消失、`.nya-version.json` 消失 |
| 存档自证 | 账号 1~3 的 `characters[].savePath` 指向 `F:\Nya-2.1.1-本地一键端-无存档-手动云更新-20260805-v8\save\multiplayer\characters\…`（即 T1 包路径）；账号 4 指向另一台机器 `E:\70-1.0.1端\…` |
| 运行痕迹 | 当前 `MainLib.swf` mtime 2026-08-08 21:24:35；`game.db` 最后写入 2026-08-08 22:51:15；`CefFlashBrowser/Caches/` 由真实运行产生 |
| 判定 | **作者开发机工作目录（含他人游玩档）在 2026-08-08 深夜左右原样压缩**，非可发布状态；它是 T1→2.2.0 之间唯一的过程态标本 |

---

## 2. 包结构与文件账目

### 2.1 顶层结构与体量（实测）

| 顶层项 | 文件数 | 字节/大小 | 说明 |
|---|---:|---:|---|
| `www/` | 5,587 | 552.5 MB | 比 T1 多 14 个（MainLib 备份 + tiyan 素材 + 测试页） |
| `tools/` | 441 | 198.2 MB | 含 ffdec 78 / jre17 316 / ffdec_work 2 / `__pycache__` 20 |
| `CefFlashBrowser_x64_v1.1.7/` | 140 | 199.3 MB | 比 T1 多 `Caches/`、`Logs/`（真实运行痕迹） |
| `runtime/` | 34 | 20.3 MB | 与 T1 逐字节一致（仅 `python313._pth` 无差异） |
| `save/` | 16 | 7.2 MB | 他人存档 + SQLite |
| `save_backup_20260807-221025/` | 12 | 0.7 MB | 08-07 22:10:25 手动备份 |
| `logs/` | 2 | 0.1 MB | `local-server.err.log` 76,435 B + `local-server.out.log` 4,815 B |
| 根目录 | 21 | — | 10 bat + 登录器 + `PACKAGE_MANIFEST.json` + **9 个开发脚本**（`run_setup.py`/`tasks_runner.py`/`run_tasks.bat`/`check_xielunyan_stages.py`/`fix_policy_and_run.py`/`fix_policy.ps1`/`fix_ps_policy.reg`/`test_py.py`/`download_ffdec.py`） |
| **合计** | **6,253** | **1,027,802,028 B（980.2 MB）** | — |

### 2.2 与 T1（2.1.1 v8）的逐文件账目

| 方向 | 数量 | 字节影响 | 明细 |
|---|---:|---:|---|
| 完全一致 | **5,707** | — | 含全部 `runtime/`、10 个 bat、`PACKAGE_MANIFEST.json`、登录器 exe、绝大部分 `www/` 资产 |
| 内容改动 | **10** | +482,083 B | 见 2.3 |
| 新增 | **536** | +242,996,334 B（**+231.7 MB**） | `tools` 195.1 MB、`www` 21.0 MB、`CefFlashBrowser/Caches` 7.6 MB、`save` 7.2 MB、`save_backup_20260807-221025` 0.7 MB、`logs` 0.1 MB |
| 删除 | **22** | −10,944,955 B（−10.4 MB） | `www` 10.1 MB（Ruffle/CEF 相关 + 旧 asset）、`NyaLauncher/` 15 文件 0.3 MB、`使用说明.txt`、`.nya-version.json` |

**+231.7 MB 的构成（本次复核，修正旧文档）**：`tools/` 的 195.1 MB 中 `jre17/` 实测 **124.8 MB**、`jre17.zip` **43.8 MB**、`ffdec/` **21.4 MB**，三者合计 **190.0 MB**——旧稿《Nya线三包源码级全量差异.md》（已并入 [源码对比分析.md](源码对比分析.md) §10）写的「jre17+ffdec 106 MB」低估了约 84 MB（漏计 `jre17.zip` 与解压后的 `jre17/` 目录差）。其余：`www` 21.0 MB 主要是 10 个 MainLib 备份，`save` 7.2 MB（SQLite 2.1 MB + WAL 4.5 MB + JSON）。

### 2.3 10 个内容改动文件（`pair_211__dj_changed.txt` 全文，共 10 行）

| 文件 | T1 → T2 | 性质 |
|---|---|---|
| `tools/fake_flash_server.py` | 2,471,309 → **2,711,058** | 主战场（+239,749 B） |
| `tools/gm_admin.py` | 34,699 → **164,325** | GM 后台大扩写（14→59 动作） |
| `tools/gm_state.py` | 6,951 → 7,263 | 新增 `family_flag_owner_id`，改走 document store |
| `tools/multiplayer_backend.py` | 60,053 → 60,992 | 存储抽象化 + `change_password`/`character_save_exists` |
| `tools/simulated_player_ai.py` | 36,466 → **57,043** | 假人打猎/追击实验 |
| `tools/start_nya_208_isolated.py` | 67,089 → 68,583 | 须佐能乎六脸表 + world-ready 超时 3→8 s 等实验 |
| `tools/static_flash_http.py` | 37,923 → 37,886 | **关闭 GM 鉴权** + 接入 PHP 兼容桥 |
| `www/act_web_nya_208_isolated/Main.swf` | 513,782 → 514,027 | 构建标签（缓存戳 v168→v214） |
| `www/act_web_nya_208_isolated/asset/MainLib.swf` | 2,137,157 → **2,146,418** | SWF 大改批次（32 个 ActionScript 类） |
| `www/gm/index.html` | 28,425 → **108,342** | GM 前端 9 页签 |

> `tools/build_language_packs.py`、`generate_item_dat.py`、`native_flash_host.cs`、`patch_skill_effects_gamedata.py`、`editor_runtime_config.py`、`local_package_server.ps1`、`local_package_update.ps1`、`ruffle_socket_proxy.py`、`nya_208_cash_catalog.json`、`nya_208_runtime_catalogs.json` 在 T1→T2 **零改动**——T2 的新功能全部集中在上面 10 个文件里。

---

## 3. 技术栈与架构

| 项 | 实测 |
|---|---|
| 架构 | 纯 Python 标准库单进程 5 服务（HTTP 19280 / WS 桥 19281 / 游戏 19284 / 频道 19285 / 原生登录 19286 + Flash 策略 843），自带 Python 3.13.14 embeddable（`runtime/python/` 34 文件与 T1 零差异） |
| 存储 | **三层并存**：① 直写 JSON（沿用 T1）；② `tools/storage_backend.py` 抽象层（`get_document_store()`/`scope_key_for_path()`）；③ SQLite 实现（`save/game.db`，WAL） |
| 启动链 | 与 T1 完全一致的 10 个 bat + `local_package_server.ps1`（2,656 B）→ `runtime\python\python.exe -u tools\start_nya_208_isolated.py`；端口守卫、日志重定向、pid 文件机制不变 |
| 前端 | 仍为 GM 面板（本地 Web）+ CefFlashBrowser/Ruffle 双客户端；新增 `www/nya-test-village.html`（1,284 B 的 CEF 村庄检视测试页，写死 `tmp_view` 账号 + 一次性票据） |
| 工具链 | JPEXS FFDec（`tools/ffdec/`）+ Temurin JDK 17 JRE（`tools/jre17/`）+ `tools/swf_tool.py` 封装 + `run_tasks.bat` 自动备份/替换 |

---

## 4. 核心实现（源码级证据）

### 4.1 `tools/fake_flash_server.py`（2,711,058 B，61,299 行）

T1→T2 **新增 37 个顶层符号**，按主题分组：

| 主题 | 代表符号 |
|---|---|
| 飞行道具实验 | `PROJECTILE_TRAVEL_*`（**620 px/s**）、`class PendingProjectileHit`、`_schedule_projectile_hit` / `_tick_pending_projectile_hits` / `_apply_projectile_hit` |
| 影分身强化 | `SHADOW_CLONE_ATTRIBUTE_INHERIT_PERCENT = 100`、`SHADOW_CLONE_SKILL_LEVEL_INHERIT_PERCENT`、`_shadow_clone_skill_level_payload`、`_next_shadow_clone_combat_ninjutsu` |
| 技能升级投影 | `_project_special_skill_upgrade_count`、`project_special_skill_cooldown_ms/cost_scale_bp/mp_cost/zq_cost` |
| Boss 游走 AI | `_boss_reachable_footholds`、`_pick_boss_wander_destination`、`_handle_boss_wander_stuck`、`_advance_boss_wander` |
| 竞技场重做 | `_arena_map_id`、`_recover_arena_location`、`_simulate_arena_outcome`、`_instant_arena_view_result`（并删 `_arena_snapshot_spawn_payload`） |
| SQLite 文档层 | `_read_doc` / `_write_doc` / `_doc_exists` / `_read_document` / `_write_document` |
| 新系统 | 新闻 `DEFAULT_NEWS/handle_news`、活动覆盖 `_activity_override`、兑换码 `_redeem_code`、自动出售 `AUTO_SELL_*`、八咫镜 `YATA_*`、草薙剑写轮眼 `KUSANAGI_SWORD_ITEM_ID`、木叶大战限技 `KONOHA_*`、神器大战日程、家族旗税率 `OFFICIAL_FAMILY_FLAG_RATE_PER_LEVEL = 3`、粮食丸、昆虫 `KUN_CHONG_*`、技能宝石、特效开关 `handle_effect_switch`、境界显示、首登礼包、特阶升阶、宠物技能/皮肤、服务端结算 PVE `_settle_announced_monster_attack` |
| 常量改动 | `MAX_PET_CAPACITY`、`ITEM_STACK_LIMIT = 9999`、`WIRE_DEBUG_ENABLED`、`PROJECT_SPECIAL_SKILL_BALANCE` |

**宠物上限方向修正（旧文档写反）**：实测 `MAX_PET_CAPACITY` **T1 = 10 → T2 = 20 → T3 = 20 → V223 = 20**。旧稿《Nya线三包源码级全量差异.md》（已并入 [源码对比分析.md](源码对比分析.md) §10）与旧报告 report_A 记作「20→10（宠物上限砍半）」，与四棵树的 AST 实测全部相反——真实情况是**开发快照把上限从 10 提到 20，2.2.0/2.2.3 沿用未回退**。

**新增 opcode（AST 枚举差集）**：Client `AUTO_SELL_CONFIG=7779`、`NEWS=7780`、`EFFECT_SWITCH=7781`（+3）；Server `NEWS=7780`、`EFFECT_SWITCH=7781`、`SKILL_CHANGE_TMP=1813`、`UPDATE_CHAR_SKILL_LEVEL=1348`、`WASH_SPECIAL_ATTR_RESULT=2068`（+5）。枚举规模：Client 119→122、Server 158→163。

### 4.2 `tools/gm_admin.py`（34,699 → 164,325 B，动作 14 → 59）

| 分组 | T2 新增动作 |
|---|---|
| 内容运营 | `news`/`update-news`、`activity-overrides`/`update-activity-overrides`、`redemption-codes`/`create-redemption-code`/`delete-redemption-code` |
| 修仙一键满级 ×7 | `cultivation-max-{all,mai,rouquan,anqi,jieyin,xielunyan,lunhuiyan}` |
| 技能 | `learn-all-skills`、`max-skill-levels` |
| 进度 | `complete-quests-achievements`、`complete-card-collection`、`complete-collection-album`、`complete-pve` |
| 装备毕业 ×7 | `max-{born,carve,inlay,myth-attributes,myth-forge,refine,strength}-equipment` |
| 装备改造 ×6 | `evolve-legendary-equipment`、`customize-additional-attributes`、`customize-tale-attribute`、`set-myth-wuxing`、`set-player-attributes`、`set-equipment-special-attributes` |
| 套装/意志/背包 | `highest-purple-job-set`、`highest-pet-equipment-set`、`max-spirit`、`clear-inventory` |
| 存档管理 ×6 | `saves-list`/`save-detail`/`save-repair`/`save-backups`/`save-restore`/`save-restore-all` |
| 假人 | `add-simulated-player`、`remove-simulated-player`（配置持久化到 `simulated_players.json`） |

符号表规模：T1 38 → T2 114 → T3 120。前端 `www/gm/index.html` 同期从 4 页签扩到 9 页签（`player/world/rank/simulated/shortcut/saves/codes/news/activity`），并硬编码默认口令 `sessionStorage.getItem("narutoGmPassword") || "NyaLocal#208"`。

### 4.3 `tools/storage_backend.py`（新增 8,340 B）与 SQLite 实验

- `backend_name()`：环境变量 `NARUTO_STORAGE`(json|sqlite) 优先，否则 `save/game.db` 存在即用 SQLite；
- `JsonDocumentStore`（原子写 + PermissionError 重试 5 次 × 20 ms 递增）与 `SqliteDocumentStore`（WAL、`documents(scope,key,payload,updated_at)` 主键 `(scope,key)`、进程内 RLock、`check_same_thread=False`）；
- `scope_key_for_path()` 做 `save/` 相对路径映射（越界回退 `("root", filename)`）；`migrate_json_to_sqlite()` 幂等迁移；配套 CLI `tools/migrate_saves_to_sqlite.py`（762 B）。

**实测库内容**：仅 1 张表 `documents`，**20 行**，最后写入 **2026-08-08 22:51:15**（≈ T2 最后一次游玩时刻）：

| scope | key（节选） |
|---|---|
| `characters` | `1001.json` 133,350 B、`1002.json` **418,854 B**、`1003.json` 84,456 B、`1004.json` 56,287 B、`1009.json` 15,995 B（木叶忍者 cid8）、两个 `.skill-cooldowns.json` |
| `multiplayer` | `accounts.json` 8,231 B、`accounts-activity-overrides.json`、`accounts-auctions.json`、`accounts-boss-state.json`、`accounts-bounties.json`、`accounts-chunin.json`、`accounts-news.json`、`accounts-redemption-codes.json`、`accounts-stalls.json` |
| `root` | `gm-state.json`、`nya-mainlib-208-isolated.json`、`s.json`（"最终测试" cid5551）、`test.json`（"拍卖测试" cid8888） |

**库内状态领先于平面 JSON**：`accounts.json` 的 `nextAccountId = 6`（平面 JSON 为 5），其中多出**第 5 个账号 `admin`**（id 5，创建 2026-08-08 20:03:39，无角色）；`characters/1002.json` 里角色名已改为 **"单机端免费"**（平面 JSON 仍是「宇智波雾」）——与 `local-server.out.log` 的 `[line-debug] 单机端免费 bind fallback` 互相印证。

### 4.4 `tools/simulated_player_ai.py`（36,466 → 57,043 B）

`SimulatedPlayerActor` 由 447 行扩到 936 行，新增完整「打猎/追击」行为：`_hunt_target`（优先以真实玩家为目标、`_warp_follow_player` 跨图跟随，副本不跟随）、`_tick_hunt`、`_next_cast`、`_attack_range_for`/`_can_attack_target_from_range`/`_move_to`/`_settle_stop`/`_maybe_hop`；身份伪装广播 `_build_identity`/`_sync_identity`/`_apply_identity`；`VISIBLE_TICK_SECONDS` (0.18,0.28)→(0.10,0.16)、新增 `SIM_MOVE_TICK_SECONDS = 0.12`。`MAX_TEST_PLAYERS = 5` 四棵树不变；该实验在 2.2.0 **全量回退**（归一化后与 T1 逐字节一致，0 内容改动）。

---

## 5. 与基线的逐项差异

### 5.1 SWF 逆向工作流（T2 独有）

| 组件 | 实测 |
|---|---|
| `tools/ffdec/` | 78 文件 / 21.4 MB（JPEXS FFDec 全套：`ffdec.jar`、`ffdec-cli.jar`、`lib/`、soleditor） |
| `tools/jre17/` + `jre17.zip` | 316 文件 / **124.8 MB** + 43.8 MB（Temurin `jdk-17.0.20+8-jre`，免装 Java） |
| `tools/swf_tool.py` | 3,715 B，FFDec 命令行封装（反编译/编译） |
| `tools/ffdec_work/` | 2 份残留反编译产物（各 4,320 B）：`mythic_src/scripts/.../EquipAptitudeUtil.as` 与 `check_mythic_ok/scripts/.../EquipAptitudeUtil.as`——「神话锻造（`isMythForgedItem → APTITUDE_MYTH`）+ 彩虹呼吸光」功能的逆向现场 |
| `run_tasks.bat` | 2,127 B：从 `f:\…v8\3\MainLib.swf` 复制覆盖 `www\act_web_nya_208_isolated\asset\MainLib.swf`、从 GitHub 下载 FFDec 24.1.0 解压到 `tools\ffdec` |
| `tools/__pycache__/` | 20 个 pyc / 7.1 MB（cpython-313 为主，另有 cpython-312 双版本，说明开发期换过 Python 运行时） |

### 5.2 MainLib.swf 备份链（10 个带时间戳备份，T3 全部没有）

备份名 = **备份发生时刻**（`before-<YYYYMMDD-HHMMSS>-<动作>`），该文件自身的 mtime = 被替换掉的旧版本写入时刻——两者构成一条自洽的替换链：

| 备份文件 | 字节 | 文件 mtime（=旧版写入时刻） | 该步动作 |
|---|---:|---|---|
| `MainLib.swf.before-20260807-184948-chibaku-cd` | 2,139,482 | 2026-08-06 19:16:04 | 地爆天星(4010) CD 改 3 秒（测试值） |
| `MainLib.swf.before-20260807-205400-badge-yulian` | 2,139,497 | 2026-08-07 18:50:10 | 徽章/玉帘文案 |
| `MainLib.swf.before-20260807-211158-tooltip` | 2,139,994 | 2026-08-07 20:54:12 | 装备 tooltip |
| `MainLib.swf.before-20260807-212126-gouyu` | 2,139,874 | 2026-08-07 21:12:14 | 勾玉 tooltip |
| `MainLib.swf.before-20260807-212751-cell` | 2,139,955 | 2026-08-07 21:21:31 | 品质/格子 |
| `MainLib.swf.before-20260807-215027-quality` | 2,139,497 | 2026-08-07 18:50:10 ← **与 badge-yulian 备份同 mtime** | 该次替换用的是 18:50 版本的副本（**一次回退**） |
| `MainLib.before-20260808-193737-import` | 2,144,257 | 2026-08-08 17:28:22 | 08-08 大改 20 类一次落地（自动出售面板、眼/忍按钮、NEWS/EFFECT_SWITCH/AUTO_SELL 协议、他玩家外观可见…） |
| `MainLib.before-20260808-201628-import` | 2,144,414 | 2026-08-08 19:37:37 | 攻速动画联动 |
| `MainLib.before-20260808-202907-import` | 2,144,910 | 2026-08-08 20:16:28 | 换频道 CD |
| `MainLib.before-20260808-212435-import` | 2,144,982 | 2026-08-08 20:29:07 | 换频道 CD 再调 |
| **当前 `MainLib.swf`** | **2,146,418** | 2026-08-08 21:24:35 | 最后一次替换的产物（他玩家外观 + 勾玉抗性切换） |

另有 3 个非时间戳副本：`MainLib.other-profile-visible-v131.swf`（2,118,120 B，07-31，T2 与 T3 逐字节相同）、`MainLib_plain.swf` 与 `MainLib_pvp.swf`（各 4,393,375 B，2026-07-25 17:03，7/25 原始底版）。**口径说明**：任务书称「11 个时间戳备份」，实测为 **10 个带时间戳的 `before-*` 备份 + 1 个 07-31 变体副本 = 11 个非当前版本 MainLib**；若只数带时间戳者则为 10 个。

### 5.3 体验服素材抄入（7 个，仅 T2 有）

`www/act_web_tiyan/swf/UISwf/`：`UISwf_JueDou.swf` 44,108 B、`UISwf_LiaoLi.swf` 1,113 B、`UISwf_MuYeBattle.swf` 42,367 B、`UISwf_MuYeBattleChengHao.swf` 6,421 B、`UISwf_RouQuanChar1/4/6.swf` 4,269/4,218/4,523 B。T1/T3/V223 的 `act_web_tiyan/` 分别只有 1/1/2 个文件（`localization/t2s.txt`）——开发期把体验服客户端的决斗/料理/木叶大战/称号/肉拳角色 UI 素材逐个抄进来测试，2.2.0 清理。

### 5.4 开发脚本与硬编码路径（12 个，T3/V223 全删）

| 文件 | 字节 | 用途 / 路径证据 |
|---|---:|---|
| `run_setup.py` | 3,106 | 「任务1: 替换 MainLib.swf」等部署步骤 |
| `tasks_runner.py` | 1,778 | 复制 `f:\Nya-2.1.1…\3\MainLib.swf`（**3 处**） |
| `run_tasks.bat` | 2,127 | 同上的 bat 版（`f:\Nya-2.1.1…` **11 处**） |
| `check_xielunyan_stages.py` | 1,543 | 检查写轮眼最大阶段配置 |
| `fix_policy_and_run.py` | 3,665 | winreg 写 PowerShell `ExecutionPolicy=RemoteSigned`（**3 处** f:\ 路径） |
| `fix_policy.ps1` | 75 | `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` |
| `fix_ps_policy.reg` | 152 | 注册表导入 ExecutionPolicy |
| `test_py.py` | 86 | 打印 Python 版本（环境自检） |
| `download_ffdec.py` | 2,890 | 下载/解压 FFDec（**1 处** f:\ 路径） |
| `tools/swf_tool.py` | 3,715 | SWF 替换工具 |
| `tools/migrate_saves_to_sqlite.py` | 762 | JSON→SQLite 迁移 CLI |
| `tools/cef_fullscreen_test.ps1` | 2,943 | CEF 窗口截图/全屏自动化测试（PrintWindow） |

全树 grep `f:\Nya-2\.1\.1`（不区分大小写）命中 **10 个文件**（上述 5 个源文件 + 5 个 `tools/__pycache__/*.cpython-313.pyc`）。

### 5.5 与 2.2.0 的关系：保留 / 回退 / 删除

| 类别 | 内容 |
|---|---|
| **被保留（进入 2.2.0）** | 技能升级投影体系（`PROJECT_SPECIAL_SKILL_BALANCE`）、新闻(7780)、兑换码、活动覆盖、自动出售(7779)、八咫镜、草薙剑写轮眼、木叶大战限技、神器大战日程、家族旗税率、特效开关(7781)、服务端结算 PVE 框架、竞技场重做、掉落覆盖、GM 后台 59 动作中的 56 个（只删 3 个 simulated-player 动作 + 新增 1 个 `pet-potential-boost` → 最终 57） |
| **被回退** | `simulated_player_ai.py` 打猎/追击实验（T3 归一化后与 T1 **逐字节相同**）、飞行道具 `PendingProjectileHit`/`PROJECTILE_TRAVEL_*`、Boss 游走 AI、影子分身继承 100%、`ITEM_STACK_LIMIT = 9999`、`WIRE_DEBUG_ENABLED`、须佐能乎六脸表与 world-ready 超时 8 s、SQLite 双后端（2.2.0 的 `storage_backend.py` 重写为纯 JSON 并自述 *"deliberately keeps saves as ordinary JSON files"*）、`migrate_saves_to_sqlite.py` |
| **被删除** | `NyaLauncher/` C# 源码（15 文件）、`使用说明.txt`、`.nya-version.json`（T2 已删，2.2.0 重建并升级到 vc10）、全部 12 个开发脚本、`tools/ffdec/`、`tools/jre17/`、`tools/jre17.zip`、`tools/ffdec_work/`、`www/nya-test-village.html`、7 个 tiyan `UISwf` 素材、10 个 MainLib 备份、`CefFlashBrowser/` 全部（含 `启动游戏.bat`）、Ruffle 浏览器路线 |
| **未进入 2.2.0 的** | 被人玩出来的存档与 4 个账号（2.2.0 换成作者测试档）、SQLite `game.db`、`save_backup_20260807-221025/`、`logs` |

> 方向提示：T2 **不是** 2.2.0 的直接父目录。2.2.0 出自 `.codex-temp\nya211-rebuild-20260810-v1\…-20260808-v9-final` 重建目录（见 [03-Nya官方线.md](03-Nya官方线.md) §1.3），T2 只是同一开发线在 08-08 左右的一次「打包留档」。

---

## 6. 质量、安全与隐私

### 6.1 他人游玩存档（本次实测）

| 账号 | id | 创建时间 | 角色 | 职业/等级 | 社交证据 |
|---|---:|---|---|---|---|
| `qwe104510988` | 1 | 2026-08-06 01:44:36 | 1001 漩涡雪 | job 100 / lv120 | 配偶=账号 2（`bondAt` 2026-08-06 21:13:30，`bondItem` 19080001）；收花 99 |
| `zxc104510988` | 2 | 2026-08-06 02:01:39 | 1002 宇智波雾（库内已改名「单机端免费」） | job 200 / lv120 | 配偶=账号 1；家族「木叶村」会长，**家族资金 1,659,654,240**；累计送花 198 |
| `asd104510988` | 3 | 2026-08-06 12:03:27 | 1003 神月泉 | job 100 / lv120 | 收花 99 |
| `qweqwe` | 4 | 2026-08-07 17:32:33 | 1004 漩涡朔 | job 400 / lv80 | 无社交；`savePath` = `E:\70-1.0.1端\…` |
| `admin`（**仅存在于 `game.db`**） | 5 | 2026-08-08 20:03:39 | 无角色 | — | SQLite 库内第 5 个账号 |

- 密码统一为 Django 风格 `pbkdf2_sha256$260000$<b64salt>$<b64hash>`（**PBKDF2-HMAC-SHA256 26 万次**），无法直接还原明文，但账号名、角色名、社交关系、资产、在线时长与真实构建路径全部明文可见。
- 角色文件：`characters/1001.json` 142,600 B、**`1002.json` 471,373 B（最大）**、`1003.json` 82,464 B、`1004.json` 64,694 B + 2 个 `.skill-cooldowns.json`；`activity_online_seconds` 分别为 73 / 10,800（3 小时）/1,312/2,467。
- `save_backup_20260807-221025/`（12 文件）是同一批存档 08-07 22:10:25 的整套副本；`save/multiplayer/` 另有 `accounts-boss-state.json`、`accounts-chunin.json`（中忍考试 season 2026-08-03）、`accounts-stalls.json`、`simulated_players.json`（`{"players":[]}`）。
- 库内测试角色：`1009.json`（木叶忍者 cid8）、`test.json`（"拍卖测试" cid8888）、`s.json`（"最终测试" cid5551）。

### 6.2 GM 鉴权被整体关闭（源码）

```python
# tools/static_flash_http.py L535-537
def _require_gm_auth(self) -> bool:
    """Local single-player GM backend skips password verification."""
    return True
```

对照：T1 相同位置（L523-532）是 `hmac.compare_digest(supplied, GM_PASSWORD)` 失败即 401；2.2.0（L599-612）恢复校验并叠加 `/api/ops/*` 门禁。T2 期间 **`/api/gm/status`、`/api/gm/items`、`/api/gm/action` 三个接口对任何能访问 19280 端口的人开放**——结合 §6.4 的 0.0.0.0 绑定，这是一条真实可利用链路（可一键发货币/道具/等级/清空背包/改存档），不是纸面问题。前端同时把默认口令写死（`|| "NyaLocal#208"`）实现免输入登录。

### 6.3 数据校验与完整性

`start_nya_208_isolated.py` 的 GameData.dat SHA-256 权威校验在 T2 仍然生效（`runtimeGameDataSha256 = 091a4ca35ea9b06f`，与 T1 同值、实测 match）；`PACKAGE_MANIFEST.json` 的 6 个关键文件哈希基线也在，但**其声明的 `localhost-only` 与实际运行状态不符**（§6.4）。

### 6.4 公网运行证据

`logs/local-server.out.log`（4,815 B）实测内容节选：

```
Native Flash policy server listening on 0.0.0.0:843
Static Flash HTTP server listening on 0.0.0.0:19280
Ruffle socket proxy listening on 0.0.0.0:19281
Multiplayer Flash login server listening on 0.0.0.0:19284
Native Flash channel server listening on 0.0.0.0:19285
Native Flash login server listening on 0.0.0.0:19286
Returning game-server IP 27.184.1.224:19285
[line-debug] original login zxc104510988 requested_line=1 -> line_id=1
[line-debug] 单机端免费 bind fallback preferred_line=2 -> line_id=2
[original] channel login accepted zxc104510988:单机端免费
```

**根因（源码级）**：`start_nya_208_isolated.py:40-52` 的 `resolve_network_hosts()` 只有在 `NARUTO_LOCAL_ONLY` 为真时才返回 `127.0.0.1`；否则默认 `NARUTO_BIND_HOST = "0.0.0.0"`，且 `DEFAULT_PUBLIC_HOST = "27.184.1.224"`（L33），公告主机为 `127.0.0.1/localhost/0.0.0.0` 时还会强制回落成该公网 IP（L47-48）。**只要不用 `01_启动服务器.bat`（它设 `NARUTO_LOCAL_ONLY=1`）而直接起 Python，就是 0.0.0.0 + 公网 IP 公告模式**——日志正是这样跑出来的。该 IP 与后续 wdnm 修改版里被默认写死的 `27.184.1.224` 同源（见 [源码对比分析.md](源码对比分析.md) §11.2 wdnm 条目）。

### 6.5 日志与运行残留

- `logs/local-server.err.log`（76,435 B）：**0 个 Traceback / 0 个 Exception**，全部是 BaseHTTPRequestHandler 访问行 + 少量 404（favicon 等），时间跨度 2026-08-08 22:42:27→22:51:15（与 `game.db` 最后写入精确吻合，每 3 秒一次的 `GET /api/runtime/config` 心跳）。旧文档「err.log 76 KB 开发期报错」的表述不准确——它是**访问日志写进了 stderr 流**，不是报错。
- `CefFlashBrowser_x64_v1.1.7/Caches/`（含 `LocalPrefs.json` 479 B）证明 CEF 浏览器在开发期被真实使用过。

---

## 7. 完成度与收录建议

| 维度 | 评价 |
|---|---|
| 性质 | **非发行版**：开发机工作目录原样打包，含他人存档、SQLite 实验库、公网运行日志、本机绝对路径 |
| 研究价值 | **极高**：它是 2.1.1 v8 → 2.2.0 之间唯一的过程态标本，可完整还原「GM 后台大扩写 / SWF 逆向工作流 / SQLite 实验 / 假人 AI 实验 / 哪些实验被回退」五条线索 |
| 可玩性 | 可直接运行（启动链完整、10 个 bat 与 T1 逐字节一致），但默认无 `NARUTO_LOCAL_ONLY` 时会暴露到局域网/公网 |
| 收录建议 | **可收录，但必须先清洗**：① 删除 `save/`、`save_backup_20260807-221025/`（他人账号/角色/社交数据）；② 删除 `logs/`（含真实账号名与公网 IP）；③ 若仅作研究标本，建议保留 `tools/ffdec`/`jre17` 之外的原始结构以保住「开发现场」价值；④ 元数据中标注「GM 鉴权已关闭，仅供离线研究，禁止联网运行」 |

---

## 8. 证据与核验

命令在 `C:\Project\huoying-world-standalone` 下用系统 `python`（3.14）执行；T2 树只读访问。

| 关键结论 | 依据文件（绝对路径） | 复核命令（要点） | 实测值 |
|---|---|---|---|
| 包身份 | `C:\Project\huoying-world-standalone\res\火影世界单机版.zip` | `python -c "import hashlib;print(hashlib.sha256(open(p,'rb').read()).hexdigest())"` | 736,913,093 B；`52388C62…178E2505` |
| 文件数/字节 | `hy_analysis\deep\hash_dj.tsv`；或直接 walk `hy_analysis\x\单机版` | 逐行累加第 2 列 / `os.walk` 求和 | 6,253 / 1,027,802,028 B（980.2 MB） |
| 与 T1 的增删改 | `hy_analysis\deep\pair_211__dj_added/changed/removed.txt` | `(Get-Content f).Count`；增删项对第 1 列求和 | 536（+242,996,334 B）/ **10**（+482,083 B）/ 22（−10,944,955 B） |
| 新增 536 项归属 | 同上 `added` 文件 | 按首段路径分组求和 | `tools` 195.1 MB（jre17 124.8 + jre17.zip 43.8 + ffdec 21.4）、`www` 21.0 MB、`CefFlashBrowser` 7.6 MB、`save` 7.2 MB |
| GM 动作数 | `hy_analysis\x\单机版\tools\gm_admin.py` | `ast.parse` → 模块级 `ACTIONS` 字典键 | 59（T1=14、2.2.0=57、2.2.3=60） |
| GM 鉴权关闭 | `hy_analysis\x\单机版\tools\static_flash_http.py` L535-537 | `Select-String '_require_gm_auth' -Context 0,6` | `return True` + docstring `skips password verification` |
| 前端默认口令 | `hy_analysis\x\单机版\www\gm\index.html` | `Select-String 'NyaLocal#208'` | `sessionStorage.getItem("narutoGmPassword") \|\| "NyaLocal#208"` |
| 页签数 | `www\gm\index.html` | 正则 `data-tab="…"` | 4（T1）→ **9**（T2/T3/V223） |
| 宠物上限方向 | 四棵树 `tools\fake_flash_server.py` | 正则 `MAX_PET_CAPACITY\s*=\s*(\d+)` | T1=**10**；T2/T3/V223=**20**（旧文档写反） |
| 实验类是否回退 | `tools\fake_flash_server.py`（T2/T3） | 计数 `PendingProjectileHit`| `PROJECTILE_TRAVEL` | `SHADOW_CLONE_ATTRIBUTE_INHERIT_PERCENT` | `ITEM_STACK_LIMIT` | `WIRE_DEBUG_ENABLED` | T2 = 4/8/3/2/3 处；T3/V223 = **全 0** |
| 他人存档 | `save\multiplayer\accounts.json`、`save\multiplayer\characters\*.json` | `json.load` 后打印 username/characters/spouse/flower | 4 账号 + 角色 1001~1004；`1002.json` 471,373 B；家族资金 1,659,654,240 |
| SQLite 实验 | `save\game.db` | `sqlite3.connect('file:…?mode=ro',uri=True)` → `select scope,key,length(payload),updated_at from documents` | 1 表 / **20 行** / 最后写入 2026-08-08 22:51:15；库内 `nextAccountId=6` 含第 5 账号 `admin` |
| MainLib 备份链 | `www\act_web_nya_208_isolated\asset\MainLib*` | `Get-ChildItem … \| Select Name,Length,LastWriteTime` | 10 个带时间戳 `before-*` + 1 个 `other-profile-visible-v131` + 2 个 7/25 底版 |
| 体验服素材 | `www\act_web_tiyan\swf\UISwf\` | 文件枚举 | 7 个文件（T1/T3/V223 均无） |
| 硬编码开发机路径 | `run_tasks.bat`、`tasks_runner.py`、`fix_policy_and_run.py`、`download_ffdec.py` + 5 个 pyc | 全树正则 `f:\\Nya-2\.1\.1`（忽略大小写） | **10 个文件命中**（`run_tasks.bat` 单文件 11 处） |
| 公网运行 | `logs\local-server.out.log` | `Get-Content` | 6 个端口绑 `0.0.0.0`；`Returning game-server IP 27.184.1.224:19285` |
| 公网模式根因 | `tools\start_nya_208_isolated.py` L33/L40-52 | `Get-Content … -TotalCount 60` | `DEFAULT_PUBLIC_HOST="27.184.1.224"`；无 `NARUTO_LOCAL_ONLY` 时 bind `0.0.0.0` |
| err.log 无报错 | `logs\local-server.err.log` | `Select-String 'Traceback\|Exception'` | 命中 0（76,435 B 全为访问日志） |

---

## 9. 未决问题

1. **`game.db` 与平面 JSON 的分叉时点未逐一还原**：库内 `1002.json` 418,854 B vs 平面 471,373 B、角色名不同（"单机端免费" vs "宇智波雾"），只知库更新（08-08 22:51:15），但哪一次写入导致分叉未定位。
2. **`admin`（id 5）账号的用途未定**：创建于 2026-08-08 20:03:39、无角色，疑似 GM 面板/自动化测试建的账号，未找到调用点。
3. **`MainLib.swf.before-20260807-215027-quality` 与 `badge-yulian` 备份同 mtime**，本报告按「一次回退」解释，但流程细节（是否手工 `copy` 回旧版）无日志佐证。
4. **7 个 tiyan `UISwf` 素材是否被 T2 的 SWF/代码真正引用**未验证（只做了文件级确认）。
5. **T2 存档里的第 4 账号 `qweqwe`（`E:\70-1.0.1端`）来源不明**——可能是另一台机器/另一个版本（"1.0.1端"）的存档被拷进来联机测试，未进一步追溯。
6. **`tools/ffdec_work/` 反编译产物与最终 MainLib 的对应关系**未做逐类反编译比对（该工作由既有 dD 报告覆盖，但仅针对 `EquipAptitudeUtil.as`）。
