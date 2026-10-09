# wdnm 修改版（Nya-wdnmd 修改版 V2.6 分发版）全量分析报告
> 对象：@res/单机-wdnm修改版/Nya-wdnmd修改版V2.6-分发版.zip@（909,330,692 B，SHA-256 @458102A5C209FF4314B6206151819A61F3CDB81A45F608889D2886FB1B0CD3AE@）｜解包位置：@C:\Project\hy_analysis\x2\wdnm@｜基线：**Nya 2.1.1 v8**（官方 2.2.0/2.2.3 仅作代际参照）｜分析日期：2026-10-07

---

## 0. 结论速览

1. **血缘：2.1.1 v8 发布后约 3 天分叉的私有平行分支**。全树 5,635 个文件与 2.1.1 v8 哈希一致、48 个改动、3,995 个新增（+401,908,536 B / 383.3 MiB）、56 个删除（−26,009,610 B）；对照官方 2.2.0 缺 15,564 个文件、对照 2.2.3 缺 15,590 个 → **不存在整体移植**。@.nya-version.json@ 被作者删除（在删除清单中，144 B），官方版本戳被抹掉。
2. **代际共享：本包 150 个新增函数中恰有 34 个与官方 2.2.3 的 123 个新增函数同名**（异步存档写手、写轮眼显示、技能宝石、copy1 英雄副本、兑换码、境界显示、锻造 120、效果开关等）——分叉点在"2.1.1 之后、2.2.0 之前"的官方开发线上；但**函数体独立演化**：以 2.2.0 为参照，双方相对 2.1.1 都改过的 90 个函数里，**65 个函数体文本不同、25 个相同**。官方 2.2.3 另有 **90 个函数本包完全没有**（PVE 结算 worker、竞技场大改、仓库排序、掉落覆盖等）。
3. **双抽奖系统**：官方式 @/api/lottery/*@（木叶大奖，15 处引用）+ 自建 @www/lottery-shop/@ 刮刮乐（Vue/Pinia 构建，@/api/lottery-shop/{currency,buy,win}@ 三接口）。**@/win@ 直接采信客户端 @gain@，不校验票面，可凭空加元宝**（@static_flash_http.py:1200-1233@）；交接文档声称的"加解密结算"在交付代码中**不存在**（@crypto/encrypt/decrypt/AES@ 命中数均为 0）。
4. **联机主机模式**：新增 @bat/联机_主机模式.bat@ 设 @NARUTO_PUBLIC_HOST/NARUTO_ADVERTISE_HOST@；@DEFAULT_PUBLIC_HOST@ 从官方硬编码的 **27.184.1.224** 改为 **127.0.0.1**（@start_nya_208_isolated.py:33@），平时单机零跨网延迟。
5. **捆绑第三方游戏火影战记 V2**：@senki/NarutoSenkiV2@ 1,170 文件 / 167,766,770 B（160.0 MiB）；后端 @SENKI_REWARD = 50000@（@static_flash_http.py:71@），以"进程退出 + 存档 SHA-256 变化 + 模式全开"判定打了一局即发 5 万元宝。
6. **客户端大改**：@MainLib.swf@ 2,137,157 → 2,170,970 B，标识符 28,739 → 29,238（**恰好 +499，0 个删除**）；自制 @Char_10000119.swf@（须佐能乎·别天神）**881,974 → 7,502,509 B**（官方 2.2.0/2.2.3 仍是 881,974）；@Main.swf@ 内新增 @skillbar-pages@ 标记串并把尾兽解锁串 @一尾$0@ 改为 @一尾$1@。**GM 后台符号（gm_admin.py 的函数定义）35 → 98**。
7. **分发形态重构**：删除 @NyaLauncher/@ C# 源码 15 文件与 @Nya本地登录器.exe@，换成 **71,888,062 B 的 .NET 8 自包含 @火影世界客户端.exe@（v3.0.0.0）**；云更新被禁（@local_package_update.ps1@ 第 13-15 行打印提示后直接 @exit 0@）；bat 迁入 @bat/@。
8. **隐私重灾区（必须清洗才能收录）**：@launcher.json@ 明文 3 组账密（含疑似作者 QQ 号 1649493707 及其派生口令）、@save/game-client-account.json@ 明文、@logs/login-debug.log@ 145 行逐行明文账密与 token、3 账号 4 角色存档、@save/game-client-cache@ 86 文件 24.2 MiB 浏览器缓存；**打包脚本 @make_friend_dist_zip.py@ 自称会清空账密/排除 save 与 logs，但本分发树全部保留**。

---

## 1. 身份与来源判定

### 1.1 包与版本

| 项 | 值 |
|---|---|
| 外层 zip | 909,330,692 B / SHA-256 @458102A5…0CD3AE@ |
| 自称 | "Nya-wdnmd修改版V2.6-分发版"（@launcher.json@、@make_friend_dist_zip.py@、@docs/彩票店交接文档.md@ 三处一致） |
| 官方版本戳 | @.nya-version.json@ **被删除**（见 §5 删除清单首行） |
| 解包树 | 9,678 文件 / 1,178,712,807 B |

### 1.2 血缘证据链（为什么是"2.1.1 后 3 天分叉的私有平行分支"）

| 证据 | 实测 |
|---|---|
| 与 2.1.1 v8 的账目 | same=5,635 / changed=48 / added=3,995 / removed=56 |
| 与 2.2.0 的账目 | 基线独有 15,564 个文件在本包**不存在**（主要是 editors/预览资源与 2.2.x 新增 www 资产） |
| 与 2.2.3 的账目 | 基线独有 15,590 个文件不存在；新增函数交集恰为 **34** |
| 代际函数（名称级） | 本包新增 def 150 个 ∩ 2.2.3 新增 def 123 个 = **34 个**（@_write_payload@、@_async_writer_loop@… 见 §4.1 清单）；2.2.3 独有 def = **90 个** |
| 代际函数（函数体级，基准 2.1.1） | 双方都改过的函数 90 个：**65 个函数体不同（独立演化）、25 个完全相同** |
| tools/ 形态 | 保留 2.1.x 形态：有 @ruffle_socket_proxy.py@，**无** @storage_backend.py@ / @local_socket_proxy.py@ / @nya_client_api_bridge.py@，**无** @NyaMicroClient/@（官方 2.2.3 三者齐全） |
| 时间锚点 | manifest 内 @modified_by: "20260808-mythglow+mythanim+scrollpane-height28"@（分叉时点 08-08，早于 2.2.3 的 08-12/08-14）；源码内 @[MOD 日期]@ 共 127 处，日期分布 **08-26×21、08-27×28、08-29×2、09-02×1、09-03×6、09-08×19、09-10×3、09-12×2、09-13×9** |
| 客户端资源 | @Char_10000119.swf@ 7.5 MB、@Char_10000040.swf@ 678,603 B、@Skill_300_*@ 重制 —— 官方 2.2.0/2.2.3 均为旧版小文件，证明是作者自制而非移植 |

> 结论：本包与官方 2.2.x 是**同源平行、互不包含**的关系；"V2.6"是作者自有版本序列，与官方 versionCode 无关。

---

## 2. 包结构与文件账目

| 项 | 值 |
|---|---:|
| 解包文件数 | 9,678 |
| 解包总字节 | 1,178,712,807 |
| vs 2.1.1 相同 / 改动 / 新增 / 删除 | 5,635 / **48** / 3,995 / 56 |
| 新增字节 | +401,908,536（383.3 MiB） |
| 删除字节 | −26,009,610（24.8 MiB） |

### 2.1 新增 3,995 文件聚类（Top）

| 目录 | 文件数 | 体积 | 内容 |
|---|---:|---:|---|
| @senki/NarutoSenkiV2@ | 1,170 | 160.0 MiB | 捆绑第三方游戏"火影战记 V2" |
| @www/act_web_nya_208_isolated@ | 83 | 73.8 MiB | 新/改 SWF（Skill 20 个约 40 MB、Char 9 个、Pet 4 个、dat 备份） |
| 根目录 | 4 | 68.7 MiB | @火影世界客户端.exe@ 等 |
| @save/game-client-cache@ | 86 | 24.2 MiB | **CEF 浏览器缓存随包分发** |
| @logs/@ | 10 | 11.3 MiB | 作者运行日志 |
| @tools/patch_fixed_slot@ | 1,902 | 11.3 MiB | FFDec 导出的 MainLib 全量 AS |
| @tools/@ 其余（含 @__pycache__@） | ~120 | ~40 MiB | 补丁工作区（@patch_lottery/patch_9034/patch_two_yi_jue@…） |
| @assets/carousel@ + @assets/bgm@ | 2 | 11.4 MiB | 登录器 UI 素材 |

### 2.2 删除 56 文件（24.8 MiB）

1. **官方登录器整体移除**：@NyaLauncher/@ 15 个 C# 源文件 + @Nya本地登录器.exe@（1,847,792 B）+ @启动Nya登录器.bat@ + @编译Nya登录器.bat@；
2. **版本戳与清单**：@.nya-version.json@、@PACKAGE_MANIFEST.json@、@使用说明.txt@；根目录 8 个 bat 迁入 @bat/@；
3. **开发备份清理**：@MainLib.before-*@ ×5、@MainLib_plain.swf@、@MainLib_pvp.swf@（各约 2.1–4.4 MB）、@back_GameData.dat@、@swf/Thumbs.db@ 等；
4. **编辑器痕迹清扫**：@www/editor_workspace/history/00000001-15.json@ 15 个 + @project.json@ 被清空到 86 B。

---

## 3. 技术栈与架构

- 服务端：纯 Python（@fake_flash_server.py@ 64,775 行 + @static_flash_http.py@ 102,561 B + @gm_admin.py@ + @multiplayer_backend.py@）+ 内嵌 @runtime/python@。
- 客户端：Flash SWF + CefFlashBrowser（exe 被瘦身：2,008,576 → 292,864 B，CefSharp 拆到 @Assets@，原版留作 @CefFlashBrowser.original.exe@）。
- 数据：@GameData.dat@ / @JiNengShu.dat@ / @JiNengCanJuan.dat@ / @LiBao.dat@ / @version.dat@ 均为"明文键 + zlib 段"容器。
- 新增 Web 子系统：@www/lottery-shop/@（Vite 构建的 Vue/Pinia SPA）、@www/gm/@ 后台、@bat/@ 启动脚本组、.NET 8 自包含登录器。

---

## 4. 核心实现（源码级证据）

### 4.1 服务端 @fake_flash_server.py@（2,471,309 → 2,857,155 B，64,775 行）

- 新增 def **150** 个，其中与官方 2.2.3 同名 **34** 个：@_write_payload@、@_async_writer_loop@、@flush@、@_flush_locked@、@_broadcast_xielunyan_status@、@_display_xielunyan_stage@、@_display_realm@、@_display_skill_level@、@_skill_gem_ids@、@_send_skill_gem_bonuses@、@_refresh_skill_gem_levels@、@_start_copy1_lines@、@_spawn_next_copy1_boss@、@_copy1_spawn_positions@、@_leave_hero_copy@、@_redeem_code@、@_forge_stage_up_120@、@handle_effect_switch@、@_use_pet_skill_symbol@、@_auto_learn_pet_skills@、@_pet_dream_multiplier@、@_total_skill_level@、@_skill_jingjie_realm@、@_family_flag_attr_rates@ 等。
- 本包独有 116 个，代表：@_tick_czhuodong_activity@ 系列（限时充值大奖）、@_kamui_*@（神威）、@_ensure_two_yi_jue@（两仪决）、@_start_copy7_boss_and_waves@（copy7）、@_apply_lianzhan_boss_damage@、@_muye_mingyun_*@（木叶大战称号）、@_kun_chong_base_value@ 等。

### 4.2 双抽奖系统

**(a) 木叶大奖 @/api/lottery/*@**（@static_flash_http.py@ 内 15 处引用）：@request-open@ / @pending-open@ / @draw@（服务端真实开奖 + 保底计数 @state.lottery_pity@）/ @claim@（@_grant_lottery_items@）/ @recycle@（按品级回收元宝 N=1000 / R=2000 / SR=3000 / SSR=4000 / UR=5000）。

**(b) HTML 刮刮乐 @/api/lottery-shop/*@**（@[MOD 2026-09-13]@，代码即上文注释）：

| 接口 | 行号 | 行为 |
|---|---|---|
| @POST /api/lottery-shop/currency@ | @static_flash_http.py:1148@ | 读 @cash_money@ |
| @POST /api/lottery-shop/buy@ | :1163 | @cost=max(0,int(cost))@，余额不足返回 400，否则扣款 + @handle_cash_money("")@ 广播 |
| @POST /api/lottery-shop/win@ | :1200 | **@gain=int(payload.get("gain"))@ 直接入账** |

**可刷经济漏洞（实证）**：@/win@ 只做 @new_cash = max(0, cash_money + gain)@，**没有任何票面/流水校验**；接口鉴权仅靠请求头 @X-Naruto-Account-Id@ 匹配在线角色（匹配不到就回退"当前第一个已进入游戏的角色"）。因此任何能访问 127.0.0.1:19280 的进程可无限加元宝。代码注释自认"票面结果由前端本地生成，服务端只负责读余额/扣/加"。

**(c) 玩法与价格（与文档冲突）**：交付 JS（@www/lottery-shop/assets/index-DJynCdv6.js@，5,544 行）内基础价 @10/15/20/20/25/30@ 再统一乘 @__S=100@ → 实际 **1000 / 1500 / 2000 / 2000 / 2500 / 3000** 六档：
| 玩法 | type | 基础价×100 | 交接文档所写 |
|---|---|---:|---:|
| 福星高照 | lucky | 1000 | 1000 |
| 幸运数字 | match | 1500 | 2000 ✗ |
| 一键三连 | trinity | 2000 | 2500 ✗ |
| 数字加加 | plus | 2000 | 2500 ✗ |
| 暴击图案 | crit | 2500 | 2500 |
| 大负翁 | dice | 3000 | 3000 |

**文档 vs 代码冲突（两处，均以代码为准）**：① @docs/彩票店交接文档.md@ 称 @www/lottery-shop/index.html@ 为"787 行、含加解密结算"，实测 **index.html 只有 22 行 / 1,132 B**（逻辑全在 231,699 B 的构建产物 JS 中），且 **无任何加解密**；② 文档的四档价表与实际六档不符（见上表）。

### 4.3 联机主机模式

- @bat/联机_主机模式.bat@：@set "NARUTO_PUBLIC_HOST=27.184.1.224"@ / @NARUTO_ADVERTISE_HOST@，注释指导改成自己的公网 IP 后双击。
- @DEFAULT_PUBLIC_HOST = "127.0.0.1"@（@start_nya_208_isolated.py:33@，注释 @[MOD 2026-09-02]@）vs 2.1.1 官方 @DEFAULT_PUBLIC_HOST = "27.184.1.224"@（同文件 :33） → 默认直连本机，联机时由主机 bat 覆盖为公网 IP；@:1534@ 把 @NARUTO_ADVERTISE_HOST@ 写入环境供票据广播；@:1501@ 写 @local-server.pid@。
- 免密自动登录：@multiplayer_backend.auto_login(username)@ + 路由 @/api/account/native-auto-login@（@register_with_hash@ 用于"存档转入器"，只按用户名签发）。风险限于本机监听。

### 4.4 火影战记 V2 捆绑闭环

| 环节 | 证据 |
|---|---|
| 资源 | @senki/NarutoSenkiV2/@ 1,170 文件 / 167,766,770 B；含 @NarutoSenkiV2.exe@ 等 |
| 奖励常量 | @SENKI_REWARD = int(os.environ.get("NARUTO_SENKI_REWARD", "50000"))@ @static_flash_http.py:71@ |
| 判定 | @_senki_save_hash()@（sha256，:114-118）→ @_senki_wait_settled_hash()@（:137）→ @_senki_read_modes()@（:163）→ @_senki_all_modes_on()@（:210） |
| 前端 | @flash-native.html@ 的 @senkiLaunch/senkiPoll/senkiClaim/senkiSweep@：2 s 轮询 @/api/senki/status@，退出后 @/api/senki/claim@，"胜利！+50,000 元宝已发放"；@senkiSweep@ 每 4 s 兜底 + @/api/senki/comp@ 一次性补发 |

### 4.5 GM 后台（35 → 98 个函数）

@tools/gm_admin.py@ 34,699 → 82,079 B，函数定义 **35 → 98**，ACTIONS **14 → 55**（本报告解析口径）。新增能力：一键通关（@cultivation-max-*@、@learn-all-skills@、@max-skill-levels@、@complete-pve@）、限时大奖活动编辑器、掉落/礼包编辑器（PNG → 34×34 SWF 图标）、存档管理（@saves-list/save-restore/…@）、兑换码、@spawn-furious-boss@（狂暴野怪），以及 **须佐能乎锚点编辑器** @yuzo-faces / yuzo-face-detail / yuzo-origin-set / yuzo-origin-apply-all@ —— 用 @Amf3Reader@（7 处出现）**直接读写 @GameData.dat@ 的 AMF3+zlib 段**，这就是 GameData.dat +28,144 B 的主要写入路径。

### 4.6 SWF 客户端改动（ABC 常量池/原始体实测）

| SWF | 2.1.1 → wdnm | 实测结论 |
|---|---|---|
| @asset/MainLib.swf@ | 2,137,157 → 2,170,970 | 标识符集合 28,739 → 29,238（**+499，−0**）；原始体含 wdnm 独有串 @KAMUI_WARP_*@、@castTwoYiJueBuffChain@、@createMuYeDaJiangPane@、@createFixedCell@、@pagerHotspots@ |
| @Main.swf@ | 513,782 → 514,171 | 原始体含 wdnm 独有 @skillbar-pages@ 标记串；尾兽解锁串 @一尾$0@ → @一尾$1@（初始解锁一尾/二尾） |
| @asset/MainLoginServer*.swf@ ×3 | 395,595/393,152/391,601 → 393,394/391,013/389,208 | 各减约 2 KB，同为解锁串改动 |
| @swf/Char/Char_10000119.swf@ | 881,974 → **7,502,509** | 自制"须佐能乎·别天神"整卡；官方 2.2.0/2.2.3 仍为 881,974 |
| @swf/Char/Char_10000040.swf@ | 191,825 → 678,603 | 自制改写 |
| @Skill_300_skill_3001001~3001003@ ×8 | 65 KB–495 KB → 145 KB–1.14 MB | 技能 effect/hit/ball 全套重制 |
| @UISwf/UISwf_GetHuoyingBiIcon.swf@ | 3,579,781 → 3,579,781（同尺寸异哈希） | 重压缩产物 |

> **与旧文档的出入**：@deep\dG\report_G_wdnm.md@ 称 @Main.swf@ 新增"@-forge-special-attr-v182-skillbar-pages-v1..v21@ 21 轮迭代痕迹"。本次实测：@skillbar-pages@ 仅见于 wdnm 原始体（正确），但 @forge-special-attr@ 在 2.1.1 中**已存在**，且 dG 的 @211_Main.txt@ 与 @wdnm_Main.txt@ 两个 dump **MD5 完全相同**（@c5c56f0b0bd08bd197915e17fe1f9f08@）——该 dump 无法区分两者，字符串证据需以原始 SWF 体为准。

---

## 5. 与基线的逐项差异（48 个改动文件分组）

| 组 | 文件（字节 基线→wdnm） | 摘要 |
|---|---|---|
| 品牌改名 | @www/local-control.html@ 3,043→3,098、@www/ruffle.html@ 30,086→30,103、@www/flash-native.php@ 676→701、@CefFlashBrowser.exe.config@ | 标题改"单机火影世界——wdnmd修改版"等 |
| 云更新/进程 | @tools/local_package_update.ps1@ 17,056→17,260（第 13-15 行打印"云更新已禁用"后 @exit 0@）、@tools/local_package_server.ps1@ 2,656→6,192（清残留 + 日志轮转） | 云更新禁用，官方更新服务器 @47.95.121.98@ 被切断 |
| 服务端核心 | @tools/fake_flash_server.py@ 2,471,309→2,857,155 | §4.1 |
| GM | @tools/gm_admin.py@ 34,699→82,079、@www/gm/index.html@ 28,425→87,211、@tools/gm_state.py@ 6,951→7,832 | §4.5 |
| 后端/编排 | @tools/multiplayer_backend.py@ 60,053→62,482、@tools/start_nya_208_isolated.py@ 67,089→71,221、@tools/static_flash_http.py@ 37,923→102,561、@tools/editor_runtime_config.py@ 22,860→33,903 | 免密登录、主机模式、彩票店、编辑器配置 |
| 清单/目录 | @nya-208-isolated-manifest.json@ 24,872→25,434、@original-client-manifest.json@ 766,297→769,651、@nya211-gamedata-authority.json@ 2,502→2,573、@nya_208_cash_catalog.json@ 6,839→5,583、@nya_208_runtime_catalogs.json@ 94,969→82,068、@www/editor_workspace/project.json@ 4,215→86 | 权威哈希同步、商城位删除、编辑器痕迹清空 |
| 数据 dat | @GameData.dat@ 2,677,996→2,706,140（+28,144）、@JiNengShu.dat@ 20,054→21,552、@JiNengCanJuan.dat@ 13,674→15,064、@LiBao.dat@ 232,515→233,088、@version.dat@ 629→652 | 须佐锚点/两仪决/新技能/礼包 |
| SWF | 见 §4.6（16 个） | 客户端内容 |
| 浏览器 | @CefFlashBrowser.exe@ 2,008,576→292,864 | ILMerge 单体 → 瘦身 + @Assets\CefSharp@ probing |

---

## 6. 质量、安全与隐私

### 6.1 隐私清单（逐文件，路径 + 内容摘要）

| 文件 | 实测 | 内容摘要 |
|---|---|---|
| @launcher.json@（381 B） | **明文** | @RememberedUsername: "222"/"2222"/"1649493707"@ + @RememberedPassword@（"222" 组口令为同名重复 6 位；"2222" 组为 6 位重复；**1649493707 组的口令即 @qq<用户名>@ 形式，用户名本身是 QQ 号** → 高度疑似作者本人凭据）。**本报告不复述口令原文**，复核命令见 §8 |
| @save/game-client-account.json@（38 B） | **明文** | @{"username":"222","password":"222222"}@ |
| @logs/login-debug.log@（40,080 B / 145 行） | **明文** | 逐行 @route=/api/account/login username=[…] password=[…]@，并回显 @playerApiToken@ / @nativeLaunchToken@ 会话票据 |
| @save/multiplayer/accounts.json@（4,978 B） | 哈希（合规） | 3 个账号 @111/222/2222@，PBKDF2-SHA256；但角色 @savePath@ 泄露作者盘符 @E:\djhuoying\Nya-wdnmd修改版V2.6-分发版\…@ |
| @save/multiplayer/characters/@（7 文件，883,129 B） | 存档 | 3 账号 4 角色：1001 lisye 61 级、1002 秋道朔 120 级、1003 波风水门 74 级；含 4 份 @*.bak_*@ 测试期备份 |
| @save/game-client-cache/@（86 文件） | **24.2 MiB** | Chromium 缓存（@Cache@ 52 文件 23.5 MiB、@GPUCache@、@Cookies@、@Visited Links@），含 5 份完整 SWF 缓存副本 |
| @logs/@（10 文件 / 11,871,034 B） | 日志 | @swf-requests.log.1.log@ 10.5 MB 请求流水（含 @game-launch.html?username=…&characterId=…@）、@game-client-cef.log@ 泄露另一路径 @E:\A单机火影\…@ |
| @docs/彩票店交接文档.md@ | 作者文档 | 交接文档，暴露开发流程与私有测试账号使用习惯 |
| @.git/@（18 文件） | 版本骨架 | 无 objects，仅 hooks/config |

### 6.2 打包脚本"洗过但没洗净"（实证）

@make_friend_dist_zip.py@（3,735 B）文档头明写：

> 排除: save(个人存档)/logs/.workbuddy/所有 *.bak*/__pycache__/yuzo_editor/editor_workspace
> **launcher.json 记住的账号密码会被清空**
> save/ 目录以空目录形式保留(朋友首启=全新存档)

代码里确实实现了（@cfg["RememberedUsername"] = ""@、@EXCLUDE_DIRS@ 含 @save/logs@、@excluded_file()@ 过滤 @*.bak*@/@*.log@/@*.pyc@/@*.tmp@）。但**本分发树里 @save/@（103 文件 26,298,219 B）、@logs/@（10 文件 11.87 MB）、@launcher.json@ 三组明文凭据全部保留**——说明实际外发的是作者工作目录，而不是脚本产出的"朋友版"zip；输出名也从文档头的 @…_20260903.zip@ 变成了代码里的 @…_20260912.zip@。

### 6.3 安全

| 问题 | 证据 | 风险 |
|---|---|---|
| 元宝可凭空增加 | @/api/lottery-shop/win@ 无票面校验（§4.2b） | 单机可控；按主机模式开公网即经济系统失效 |
| 免密登录任意账号 | @/api/account/native-auto-login@ + @auto_login(username)@ | 限 127.0.0.1，但同机进程可提权到任意账号 |
| 首尾一致性 | 56 个删除中含 @.nya-version.json@ 与 @PACKAGE_MANIFEST.json@ | 官方版本溯源链被切断 |
| 交付代码 vs 文档 | "加解密结算"不存在 | 文档误导二次开发者 |

---

## 7. 完成度与收录建议

| 维度 | 评价 |
|---|---|
| 功能完成度 | 高：双抽奖、捆绑小游戏经济闭环、主机联机、GM 全套编辑器、自制角色/技能/地图资源均可用 |
| 工程质量 | 中：单文件 6.4 万行 + 150 个新函数；补丁工作区（@patch_*@ 1,902 文件）与 @.bak@ 随包 |
| 分发形态 | 已做"成品化"重构（自包含登录器、离线、bat 归位），但**隐私与缓存未清洗** |
| 收录价值 | 高：是"功能型魔改 + 分发重打包"的完整样本，彩票店与服务端协议、SWF 标识符增量、GM 直改 GameData 三条线都有教学价值 |
| 收录红线 | **不满足**：明文凭据 + 真实（作者）存档 + 日志。必须清洗后才可收录 |

**建议**：先按 @make_friend_dist_zip.py@ 的声明真正执行一次清洗（清空 @launcher.json@、删除 @save/*@、@logs/*@、@*.bak*@、@__pycache__@、@CefFlashBrowser …/Caches@），再收录；文档需修正"787 行 / 加解密 / 四档价"三处不实描述。

---

## 8. 证据与核验

| 关键结论 | 依据文件（绝对路径） | 复核命令 | 实测值 |
|---|---|---|---|
| 包身份 | @C:\Project\huoying-world-standalone\res\单机-wdnm修改版\Nya-wdnmd修改版V2.6-分发版.zip@ | @(Get-FileHash -Algorithm SHA256 -LiteralPath '<p>').Hash@ | 909,330,692 B / @458102A5…0CD3AE@ |
| 三向账目 | @C:\Project\hy_analysis\deep\hash_211.tsv@、@hash_wdnm.tsv@ | @pair.py@（同 05 报告） | same 5,635 / changed 48 / added 3,995 / removed 56 |
| 34 个代际函数 | @…\x2\wdnm\tools\fake_flash_server.py@、@…\x2\v223\tools\fake_flash_server.py@、@…\x\Nya211\…\tools\fake_flash_server.py@ | @symdiff.py@（见下方） | 150 ∩ 123 = **34**；2.2.3 独有 90 |
| @/win@ 无校验 | @…\x2\wdnm\tools\static_flash_http.py:1200-1233@ | @Get-Content '<p>' \| Select-Object -Index (1195..1233)@ | @gain=int(payload.get("gain"))@ 直接 @cash_money + gain@ |
| 六档价格 | @…\x2\wdnm\www\lottery-shop\assets\index-DJynCdv6.js@ | @Select-String -Path '<p>' -Pattern 'price: \d+'@ + 乘数 @__S = 100@ | 10/15/20/20/25/30 → 1000/1500/2000/2000/2500/3000 |
| 无加解密 | 同上 | @Select-String -Path '<p>' -Pattern 'crypto\|encrypt\|decrypt\|AES'@ | 0 命中 |
| 主机模式默认值 | @…\x2\wdnm\tools\start_nya_208_isolated.py:33@ vs @…\x\Nya211\…\tools\start_nya_208_isolated.py:33@ | @Select-String -Path '<p>' -Pattern 'DEFAULT_PUBLIC_HOST'@ | @127.0.0.1@（wdnm，@[MOD 2026-09-02]@） vs @27.184.1.224@（官方） |
| 火影战记奖励 | @…\x2\wdnm\tools\static_flash_http.py:71@ | @Select-String -Path '<p>' -Pattern 'SENKI_REWARD'@ | @50000@ |
| senki 体积 | @…\x2\wdnm\senki@ | @(Get-ChildItem -Recurse -File '<p>' \| Measure-Object Length -Sum)@ | 1,170 文件 / 167,766,770 B |
| MainLib +499 | @…\x2\wdnm\www\act_web_nya_208_isolated\asset\MainLib.swf@、@…\x\Nya211\…\asset\MainLib.swf@ | 对 CWS 解压后提取 ABC 字符串集合做差 | 28,739 → 29,238（+499/−0） |
| Char_10000119 | 两包同路径 @swf/Char/Char_10000119.swf@ | @(Get-Item '<p>').Length@ | 881,974 → 7,502,509 |
| GM 35→98 | @…\x2\wdnm\tools\gm_admin.py@ | @([regex]::Matches((Get-Content '<p>' -Raw),'(?m)^\s*def\s+')).Count@ | 35 → 98；ACTIONS 14 → 55 |
| 明文凭据（本地复核，勿外泄） | @…\x2\wdnm\launcher.json@、@save\game-client-account.json@、@logs\login-debug.log@ | @Get-Content '<p>'@ | 3 组账密 / 1 组 / 145 行明文 |
| 打包脚本未生效 | @…\x2\wdnm\make_friend_dist_zip.py@ | @Get-Content '<p>' -TotalCount 12@ | 文档头声明清空账密与排除 save/logs；实际树内全部保留 |

```python
# symdiff.py —— 代际函数名称级交集（Python 3.14，只读）
import re, os
ROOT = {
    "211":  r"C:\Project\hy_analysis\x\Nya211\Nya-2.1.1-本地一键端-无存档-手动云更新-20260805-v8",
    "v223": r"C:\Project\hy_analysis\x2\v223",
    "wdnm": r"C:\Project\hy_analysis\x2\wdnm",
}
def defs(root):
    src = open(os.path.join(root, "tools", "fake_flash_server.py"), encoding="utf-8").read()
    return set(re.findall(r"^[ \t]*def\s+([A-Za-z_]\w*)\s*\(", src, re.M))
n, v, w = defs(ROOT["211"]), defs(ROOT["v223"]), defs(ROOT["wdnm"])
new_w, new_v = w - n, v - n
print(len(new_w), len(new_v), len(new_w & new_v))   # -> 150 123 34
print(len(v - w))                                   # -> 90（官方 2.2.3 独有、未移植）
```

```powershell
# 隐私要素快速取证（只读；输出含明文，请勿张贴）
Select-String -Path 'C:\Project\hy_analysis\x2\wdnm\launcher.json' -Pattern 'Username|Password'
Select-String -Path 'C:\Project\hy_analysis\x2\wdnm\logs\login-debug.log' -Pattern 'route=/api/account/login' | Select-Object -First 3
(Get-ChildItem 'C:\Project\hy_analysis\x2\wdnm\save' -Recurse -File | Measure-Object Length -Sum).Sum   # -> 26298219
```

---

## 9. 未决问题

1. **@www/editor_workspace/project.json@ 被清空**：4,215 → 86 B，作者删掉了编辑历史；清空前的覆盖内容无法还原（只能确认这一动作本身）。
2. **@GameData.dat@ +28,144 B 的内部结构差异**：本报告只确认写入路径（GM @yuzo-*@ + @patch_skill_effects_gamedata.py@）与字节增量，**未逐段解码**；段数比对需要 AMF3 结构化解析，列为未验证。
3. **彩票店前端玩法概率分布的期望收益**：未做数学建模（@fuDist/prizes@ 等表已在 JS 内，可后续离线计算）。
4. **@senki/@ 第三方游戏本体**：仅做体积/文件数与调用链核实，未审计其自身安全性（属第三方同人游戏，超出范围）。
5. 作者身份（"wdnmd"、QQ 群 1097398812）未做外部信息追溯；@launcher.json@ 中 1649493707 是否确为作者本人 QQ **未验证**（仅依"用户名即 QQ 号 + 口令为 @qq+用户名@ 派生"推断）。
