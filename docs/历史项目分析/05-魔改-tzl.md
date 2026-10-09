# tzl 修改版（单机-tzl修改版）全量分析报告
> 对象：@res/单机-tzl修改版/基础完整修复版（九月版基础）.zip@（645,297,510 B，SHA-256 @069A1D39031AD5E298A836353D0D3C0748B3EEF9F6F1EE3FC0B087D56C3B3638@）；附随 @res/单机-tzl修改版/已经完成的修复及修改内容(1).docx@（19,458 B，SHA-256 @64AFEE09E277FD0CB3881584DC91981EAF22E21C67C717B117B2C1BE1A3591DE@）｜解包位置：@C:\Project\hy_analysis\x2\tzl\tzl_root@（另有 @tzl_tmp@，为解包中间目录）｜基线：**Nya 2.1.1 v8**（非"九月版/2.2.3"，见 §1）｜分析日期：2026-10-07

---

## 0. 结论速览

1. **基座是 Nya 2.1.1 v8，不是包名暗示的"九月版/2.2.3"**：解包树 @.nya-version.json@ 与 2.1.1 v8 逐字节相同（@releaseId=nya-local-20260805-numeric-version-v8@、@versionCode=8@、@updatedAt=2026-08-05T19:12:32+08:00@），5,739 个基线文件中 **5,730 个哈希一致、仅 9 个内容改动、0 个删除**；若与官方 2.2.3 比则有 **15,570 个文件缺失**——两者不可能是同源父子关系。
2. **改动面极小**：全部内容改动集中在 **9 个文件**（1 个 Python 服务端主文件、4 个工具/后端、2 个登录页、1 个 GM 页面、1 个 GameData.dat），新增 316 个文件（+96,806,692 B，几乎全是浏览器缓存、@__pycache__@、存档与约 212 个开发脚本），删除 0 个。
3. **@tools/fake_flash_server.py@ 2,471,309 → 2,837,035 B（57,143 → 63,528 行）**，忽略行尾差异后 **+7,344 / −959 行、308 个 hunk**；顶层符号仅 +2 常量（@AUTO_ACTION_DURATION/AUTO_AUCTION_*@）与 2 处签名变更，其余全部以**类方法**形式插入现有类内。
4. **@GameData.dat@ −35 字节的全部原因是一条 zlib 流**：5,135 条流中**仅偏移 22 处的 1 条流（Spirit/items，解压后 8,138 B）不同**，且只有 **30 个字节位置**不同；顶层 @modify_802_v2.py@ 中硬编码的 20 个新值与流内实际字节**逐值一致**（@[40,34,32,38,36,26,42,24,30,20,18,28,16,14,12,22,10,6,8,4]@）。
5. **docx 修改清单兑现度高**：本报告复核 12 项，**10 项逐字落地、1 项部分落地（神话打造改为线性增长而非"翻倍"）、1 项无服务端实现（"攻击回血 2% 传说特性"）**，另有 5 类 docx 未记载的暗改。
6. **安全面显著收缩（本包最严重的问题）**：GM 密码校验被整段删除（@_require_gm_auth@ 直接 @return True@）、启动器对 @GameData.dat@ 的 SHA-256 完整性校验被跳过；两个登录页把**明文账号密码写入浏览器 localStorage**。
7. **开发痕迹完全未清洗**：根目录 20 个 @find_*/modify_*/verify_*.py@ + @tools/@ 下 192 个脚本（@fix_*@ 108 个）、13 张调试截图、@trait_debug.log@、@local-server.pid@、两个作者自己的 docx 随包分发；脚本中 @D:\游戏\火影世界\单机\…v8@ 实机路径出现 **392 次**。
8. **收录建议**：作为"最小侵入式数值/玩法魔改"的样本有参考价值（9 文件 diff 可直接当补丁读），但**必须先清洗脚本群与截图、恢复 GM 鉴权与完整性校验**才可公开收录；不建议作为可运行分发版本直接收录。

---

## 1. 身份与来源判定（基线判定）

### 1.1 三层证据

| 层 | 证据 | 实测值 |
|---|---|---|
| L1 包级 | 外层 zip 大小 / SHA-256 | 645,297,510 B / @069A1D39…B3638@（与 @hy_analysis\res_sha256.tsv@ 登记值一致） |
| L2 清单级 | @.nya-version.json@ 内容 | 与 2.1.1 v8 **逐字节相同**（sha256 @7a47205a61b8c333…@） |
| L3 账目级 | 全树三向哈希账目 | vs 2.1.1 v8：same=5,730 / changed=9 / added=316 / removed=0 |

### 1.2 为什么不是"九月版（2.2.0/2.2.3）"

| 候选基线 | 相同文件 | 改动 | 新增 | **本包缺失（基线独有）** | 判定 |
|---|---:|---:|---:|---:|---|
| **2.1.1 v8** | **5,730** | **9** | 316 | **0** | ✅ 真实基座 |
| 单机版快照 | 5,728 | 56 | 271 | 469 | ✗ |
| 2.2.0（"九月版"内层） | 5,582 | 42 | 431 | 15,537 | ✗ |
| 2.2.3 | 5,585 | 41 | 429 | 15,570 | ✗ |

> 关键否证：若基座是 2.2.x，则 2.2.x 独有的 @resources/editor_previews@（15,325 文件）、@NyaMicroClient@、@tools/storage_backend.py@ 等应全部存在；实际本包**一个都没有**，缺失量 1.5 万个文件。而 2.1.1 v8 的 5,739 个文件中只有 9 个不同、0 个缺失。**包名"九月版基础"是作者对"下载时用的整合包"的口语称呼，与内部版本号无关**（判定规则：版本号以 @.nya-version.json@ 为准）。

### 1.3 作者与开发环境

- 实机工作目录：脚本内硬编码 @D:\游戏\火影世界\单机\…v8\tools\fake_flash_server.py@（@tools/*.py@ 中该前缀出现 **392 次**），说明作者在 2.1.1 v8 的本地副本上直接改。
- 开发方式：AI 辅助的"定位锚点 → 生成补丁脚本 → 运行 → 截图/GM 验证 → 失败再出 v2/v3"迭代（同一功能多版本脚本、按天整文件备份、中文业务描述注释）。
- 时间戳线索：@tools@ 下 25 份 @fake_flash_server_backup_2026081x_<中文功能名>.py@ 备份，日期集中在 2026-08-10 ~ 08-14；zip 内文件时间 2026-08-14。

---

## 2. 包结构与文件账目

### 2.1 总量

| 项 | 值 |
|---|---:|
| 解包文件数 | 6,055 |
| 解包总字节 | 892,505,234 |
| vs 基线 相同 / 改动 / 新增 / 删除 | 5,730 / **9** / 316 / **0** |
| 新增字节 | +96,806,692（92.3 MiB） |
| 删除字节 | 0 |

### 2.2 顶层目录（文件数 / 说明）

| 顶层 | 文件数 | 说明 |
|---|---:|---|
| @www/@ | 5,581 | 客户端 + 登录页 + GM 页面 + GameData.dat |
| @tools/@ | 210 | 服务端脚本 19 个官方文件 + 191 个新增 dev 脚本 |
| @CefFlashBrowser_x64_v1.1.7/@ | 145 | 浏览器运行时 + 运行缓存 |
| 根目录 | 51 | 20 个 @.py@、13 张 @.png@、8 个 bat、2 个作者的 docx、@local-server.pid@、@trait_debug.log@ 等 |
| @runtime/@ / @NyaLauncher/@ | 34 / 15 | 与基线一致（未改） |
| @save/@ | 14 | 新增 12 个存档文件 |
| @logs/@ | 5 | 新增运行日志 |

### 2.3 新增 316 文件聚类

| 目录 | 文件数 | 体积 | 内容 |
|---|---:|---:|---|
| @tools/@（含 @__pycache__@） | 191 | 70.3 MiB | 开发脚本 + 字节码缓存 |
| @www/act_web_nya_208_isolated@ | 4 | 10.2 MiB | 新增 SWF/数据 |
| @CefFlashBrowser…/Caches@ + @Logs@ | 61 | 6.1 MiB | **运行缓存被原样打包** |
| 根目录 | 37 | 2.8 MiB | 脚本、截图、作者 docx |
| @logs/@ | 5 | 2.7 MiB | http/调试日志 |
| @save/multiplayer@ | 12 | 0.3 MiB | 作者测试存档 |

---

## 3. 技术栈与架构

与 Nya 官方线一致，未引入任何新技术：

- **服务端**：单文件 Python（@tools/fake_flash_server.py@，63,528 行）+ @static_flash_http.py@（本地 HTTP + GM API）+ @multiplayer_backend.py@（账号/存档）+ @start_nya_208_isolated.py@（编排/拉起）+ @gm_admin.py@（GM 动作）。运行环境 @runtime/python@（内嵌 CPython）。
- **客户端**：Flash SWF（@MainLib.swf@ 等，本包**完全未改**）+ CefFlashBrowser/Ruffle 宿主。
- **数据**：@GameData.dat@ = AMF3 对象 + 多段 @zlib@ 流容器（0x0C + u29 长度 + @78 9c/da@），共 5,135 条流；@www/act_web_nya_208_isolated/nya211-gamedata-authority.json@ 持有整文件 SHA-256 权威值。
- **Web**：@www/login.php@（PHP 登录壳）+ 静态页。

---

## 4. 核心实现（源码级证据）

### 4.1 9 个改动文件一览

| # | 文件 | 字节 基线→tzl | 行数 基线→tzl | 性质 |
|---|---|---:|---:|---|
| 1 | @tools/fake_flash_server.py@ | 2,471,309 → 2,837,035 | 57,143 → 63,528 | 玩法主体（+365 KB） |
| 2 | @tools/gm_admin.py@ | 34,699 → 65,716 | — | ACTIONS 14→38，函数 35→62 |
| 3 | @tools/multiplayer_backend.py@ | 60,053 → 65,884 | — | 新增 @delete_account@（86 行） |
| 4 | @tools/start_nya_208_isolated.py@ | 67,089 → 68,653 | — | **跳过 GameData 完整性校验** |
| 5 | @tools/static_flash_http.py@ | 37,923 → 37,676 | — | **移除 GM 鉴权** |
| 6 | @www/login.php@ | 2,855 → 12,991 | — | UI 重做 + localStorage 记住密码 |
| 7 | @www/flash-native.html@ | 10,716 → 17,600 | — | 同上 |
| 8 | @www/gm/index.html@ | 28,425 → 37,525 | — | GM 后台"快捷功能"页签 |
| 9 | @www/act_web_nya_208_isolated/dat/GameData.dat@ | 2,677,996 → 2,677,961 | — | −35 B（仅 1 条流变化） |

### 4.2 @fake_flash_server.py@ 新增功能与锚点（行号均指 tzl 版）

| 功能块 | 关键符号 / 常量 | 行号 | 摘要 |
|---|---|---|---|
| 竞拍场自动上架 | @AUTO_AUCTION_ITEMS@（9051）、@AUTO_AUCTION_INTERVAL=3600@、@AUTO_ACTION_DURATION=8*3600@ | 9051-9074 | 每小时自动上架一件，持续 8 小时 |
| NPC 22 神话打造菜单 | @NPC_SERVICE_MENUS[22]@、@_handle_myth_forge@ | 10505、56582 | 三代火影菜单 |
| 自来也/三代元宝召唤 | @_summon_special_spirit_teacher@（cost=2000、每日上限 100 次） | 19699-19711 | 元宝召唤 |
| 召唤品质概率 | @{"pueple":80,"pink":19,"red":1}@ | 19898-19902 | 紫/粉/红 |
| 特殊三代池 | @cost = 5000@（@teacher_index==4 and spirit_sandaime_special@） | 19793-19799 | 5000 元宝 |
| 野外 BOSS 分级掉落 | @_roll_wild_boss_drop@（≤20/≤40/≤60/≤80/≤100/100+ 六档） | 27387 | 概率表逐档 |
| 击杀固定奖励 | @cash_money += 5; coupon_money += 5; money += 100000@ | 44687-44700 | 5 元宝 + 5 点券 + 10 万金币 |
| 副本/英雄副本/自定义 BOSS 掉落 | @_roll_copy_boss_drop@ / @_roll_hero_copy_boss_drop@ / @_roll_custom_boss_drop@ | 45865 / 45998 / 46256 | 三类副本掉落表 |
| 装备部位词条限制 | @_get_allowed_plus_attributes@ | 56111 | 部位 → 允许词条集合 |
| 神话属性锻造 | @_forge_myth_attribute@：@success_rates_by_quality={1:50,2:40,3:30}@（54143）、@value_ranges={1:(1,3),2:(2,4),3:(3,5)}@（54113） | 54110-54330 | 品质分级 |
| 神话重置 | @_reset_myth_forge@ | 54332-54420 | 1~5 颗→第 N 条、6→特技、7→全部 |
| 特性洗练 | @_wash_special_attributes@：@randrange(100) >= 70@ 判失败（56861） | 56842-56930 | 固定 70% 成功 |
| 资源上限 | @MAX_ZQ = 500_000 → 2_000_000@ | 544 | ×4 |
| 八门去倒计时 | 注释"取消 10 秒倒计时，成功后立即生效" | 53563 | 立即生效 |

### 4.3 GameData.dat：唯一变动的流

- 结构复核（Python @zlib.decompressobj@ 逐流扫描）：**基线 5,135 条流 / tzl 5,135 条流**，逐条比对后**只有 section#0（文件偏移 22）不同**。
- 该流：解压后 **8,138 B → 8,138 B**（长度不变），**30 个字节位置**不同；ASCII 键为 @801 / spec / attrType / damage_reflect / color / pink@，即称"意志物品 Spirit/items"流。
- 该流内 AMF3 整数序列：基线 @[40,38,36,34,32,30,28,26,24,22,20,18,16,14,12,10,8,6,4,2]@ → tzl @[40,34,32,38,36,26,42,24,30,20,18,28,16,14,12,22,10,6,8,4]@；**数值集合由"2~40 偶数"变为"4~42 偶数"**。
- 与顶层脚本互证：@modify_802_v2.py@ 中 @new_values = [40, 34, 32, 38, 36, 26, 42, 24, 30, 20, 18, 28, 16, 14, 12, 22, 10, 6, 8, 4]@ 与流内实际字节**逐值一致**；脚本同时显式调用 @zlib.compress(bytes, 9)@ 并用 @encode_u29(new_len << 1 | 1)@ 重写 AMF3 长度头——这正是 −35 字节的来源（@78 9c@ → @78 da@ 高压缩等级）。
- **与旧文档的出入（重要）**：@deep\dF\report_F_tzl.md@ 称新版"统一为随等级严格递增：4,6,8,…,42（level19=40，level20=42）"。**本次实测该序列并非递增**，而是上述乱序排列；只有"值集合上移 +2（2..40 → 4..42）"和"20 个槽位值全部改变"可从字节流直接证实。脚本注释写的是"属性顺序和 801 的一样"，即这 20 个槽位是**属性槽序**而非等级序；level→value 的映射需要解析 AMF3 键引用才能确定，**本报告不宣称"按等级递增"**。

---

## 5. 与基线的逐项差异（docx 声称 vs 代码）

docx 全文 6,088 字（已转文本）逐项复核：

| # | docx 声称 | 代码证据（tzl 行号） | 结论 |
|---|---|---|---|
| 1 | 洗附加属性按部位限制词条（武器/手套=攻击暴击，衣服/护额=生命闪避，护腰/忍鞋=生命防御，斗笠/项链=攻击生命，戒指/手环=攻击暴击生命，忍包/斗篷=攻击生命防御） | @_get_allowed_plus_attributes@（56111）部位→集合硬编码 | **落地**；斗笠/项链代码为"攻击、生命、**闪避**"，docx 少写闪避 |
| 2 | 神话锻造成功率 50%/40%/30%；词条范围普通 1-3%、精良 2-4%、完美 3-5% | @success_rates_by_quality={1:50,2:40,3:30}@（54143）、@value_ranges={1:(1,3),2:(2,4),3:(3,5)}@（54113） | **完全吻合** |
| 3 | 高级锻造石可覆盖低级词条直至全部替换 | @_forge_myth_attribute@ 内 @lower_quality_indices@ 逐条提品质 | **吻合** |
| 4 | 重置：1 颗→第 1 条…6 颗→特技，7 颗→全部；五行重置随机而非清零 | @_reset_myth_forge@（54332-54420） | **完全吻合** |
| 5 | 特性洗炼固定 70%；变身卡 6 词条+1 传说特性，装备前 3+后 3；百分比池与数值池取值范围 | @_wash_special_attributes@：@randrange(100) >= 70@（56861）与池常量 | **完全吻合**（区间逐值核对一致） |
| 6 | 传说特性固定"攻击回血 2%" | 全文件检索 @回血@ 仅命中"自动回血/回蓝阈值配置"（49382），**无传说特性实现** | **未落地**（客户端表现或纯文档） |
| 7 | 神话装备打造：NPC 三代火影，1 打造石 + 1 亿金币，10% 成功率，成功后基本属性翻倍 | @_handle_myth_forge@（56582）：@gold_cost = 100_000_000@（56638）、10% 判定 | **部分落地**：实际**另收 1000 元宝**，成功效果为"(次数+1) 倍线性增长、上限 20 次"而非"翻倍" |
| 8 | 传说洗练石修复（移速 1-2、攻速 1-10、反弹/减免 1-4%） | @_refine_tale_attribute@ | **落地** |
| 9 | 自来也 2000 元宝/次：80% 紫 / 19% 粉 / 1% 红；30% 点亮特殊三代，5000 元宝同概率 | @_summon_special_spirit_teacher@（19699，cost=2000）、@probabilities@（19898）、特殊池 @cost=5000@（19799） | **完全吻合**；代码另加**每日上限 100 次**（docx 未写） |
| 10 | 野外 BOSS 六档分级掉落表（各档概率与数量） | @_roll_wild_boss_drop@（27387-27663） | **完全吻合**（逐档概率/数量核对） |
| 11 | 野外小怪固定掉 5 元宝；八门取消 10 秒等待 | @cash_money += 5@（44688）；"取消10秒倒计时"（53563） | **吻合** |
| 12 | 结印一键消耗全部查克拉精华、宠物经验/进阶修复、家族商城等 | 均可在代码中找到对应分支 | **落地** |

**兑现度结论：抽查 12 项 → 10 项逐字落地、1 项部分落地（#7 数值口径已变）、1 项无服务端实现（#6）。docx 是"承诺清单"，代码整体兑现，且多处代码比 docx 更新（线性增长、加收元宝、每日上限、斗笠项链多闪避）。**

### docx 未记载的改动（暗改清单）

| 类别 | 内容 | 证据 |
|---|---|---|
| 安全 | GM 密码校验删除 | @static_flash_http.py@ @_require_gm_auth@ → @return True@ |
| 安全 | @GameData.dat@ 运行时 SHA-256 校验被短路 | @start_nya_208_isolated.py@ @verify_nya211_runtime_game_data()@ |
| 隐私 | 登录页明文记住账号密码（localStorage 键 @nya_saved_accounts@） | @www/login.php@、@www/flash-native.html@ |
| 玩法 | 竞拍场（自动上架/持久化/断线修复）、VIP、连斩、怪物掉血实时显示、护送日期、八咫镜、对戒、限时大奖、家族证明分类兑换、家徽限制取消 | @tools/fake_flash_server_backup_2026081x_<中文名>.py@ 25 份备份文件名链 |
| 数值 | @MAX_ZQ@ 50 万 → 200 万 | 544 行 |

---

## 6. 质量、安全与隐私

### 6.1 安全（高）

| 问题 | 证据 | 影响 |
|---|---|---|
| **GM API 完全无鉴权** | @tools/static_flash_http.py@：@_require_gm_auth@ 内原 @hmac.compare_digest@ 分支被替换为注释"取消GM密码验证，直接通过" + @return True@ | 任何能访问本地 HTTP 端口的进程可重置/删除任意账号、一键满修炼、直接发放装备 |
| **完整性校验被绕过** | @tools/start_nya_208_isolated.py@：@verify_nya211_runtime_game_data()@ 开头插入"跳过校验，允许修改GameData.dat" + @return@ | 权威哈希机制失效，@GameData.dat@ 被任意篡改也不再拒启动（本包正是靠它改 GameData） |
| **明文凭据落浏览器** | @www/login.php@、@www/flash-native.html@：勾选"记住账号密码"后把明文账号+口令写入 @localStorage['nya_saved_accounts']@ | 同机任意页面/脚本可读取；与免鉴权 GM 组合放大风险 |
| 新增端点攻击面 | @gm_admin.py@ 新增 24 个 ACTIONS（@reset-*@、@cultivation-max-*@、@learn-all-skills@、@delete-account@…） | 与无鉴权叠加 |

### 6.2 隐私（中高）

- 根目录携带作者本人 **2 个 docx**（@已经完成的修复及修改内容.dox.docm@、@礼包.docx@）。
- @local-server.pid@、@trait_debug.log@、@logs/@ 5 个日志、13 张实机截图（@debug_screenshot.png@、@error*.png@、@trait_result.png@ 等）随包分发。
- @save/multiplayer@ 新增 12 个存档文件（作者测试号），未做清洗。
- 脚本内 @D:\游戏\火影世界\单机\…v8@ 实机路径出现 392 次（@tools/*.py@），暴露作者本机目录结构。

### 6.3 代码质量

- 魔改以"整函数文本替换/锚点插入"落地，@tools/@ 下 @fix_*@ 脚本 108 个、@replace_*_func@ 系列直接换函数体——可维护性差，但**改动集中、可增量阅读**。
- 8/9 个改动文件存在整文件行尾差异（LF↔CRLF），任何 diff 必须先 @--strip-trailing-cr@，否则虚增改动量。

---

## 7. 完成度与收录建议

| 维度 | 评价 |
|---|---|
| 功能完成度 | 高。docx 12 项中 10 项逐字落地，含全部概率常量与掉落表数值；暗改的竞拍场/VIP/连斩等亦为完整功能 |
| 稳定形态 | 是作者自用工作目录的打包快照（含 pid、日志、截图、多轮备份），**不是发布态** |
| 安全合规 | 差。GM 无鉴权 + 完整性校验关闭 + 明文记住密码 |
| 收录价值 | 中高：是"最小侵入式数值魔改"的最佳样本（9 文件 diff 可直接作为补丁参考），GameData 单流改写与顶层脚本互证是很好的"数据层逆向"素材 |

**建议**：收录"差异分析"结论，不直接收录压缩包；若必须收录，需（1）删除 20+192 个 dev 脚本、13 张截图、@local-server.pid@、@trait_debug.log@、@logs/*@、@CefFlashBrowser …/Caches@、作者的 docx；（2）恢复 @_require_gm_auth@ 与 @verify_nya211_runtime_game_data@；（3）移除两处 localStorage 明文记住密码。

---

## 8. 证据与核验

| 关键结论 | 依据文件（绝对路径） | 复核命令 | 实测值 |
|---|---|---|---|
| 外层包身份 | @C:\Project\huoying-world-standalone\res\单机-tzl修改版\基础完整修复版（九月版基础）.zip@ | @(Get-FileHash -Algorithm SHA256 -LiteralPath '<path>').Hash@ | 645,297,510 B / @069A1D39…B3638@ |
| 基座 = 2.1.1 v8 | @C:\Project\hy_analysis\x2\tzl\tzl_root\.nya-version.json@ vs @C:\Project\hy_analysis\x\Nya211\Nya-2.1.1-本地一键端-无存档-手动云更新-20260805-v8\.nya-version.json@ | 两文件 @Get-FileHash@ 比较 | @versionCode=8@、@releaseId=nya-local-20260805-numeric-version-v8@，哈希相同 |
| 三向账目 9/316/0 | @C:\Project\hy_analysis\deep\hash_211.tsv@、@hash_tzl.tsv@ | 下方 @pair.py@ | same 5,730 / changed 9 / added 316 / removed 0 |
| 9 个改动文件与字节数 | @C:\Project\hy_analysis\deep\x2pair_211__tzl_changed.txt@ | @Get-Content '<path>'@ | 9 行，含 @2471309→2837035 tools/fake_flash_server.py@ |
| GameData 仅 1 条流变 / −35 B | @…\x\Nya211\…\dat\GameData.dat@、@…\x2\tzl\tzl_root\www\act_web_nya_208_isolated\dat\GameData.dat@ | 下方 @gamedata.py@ | 5,135→5,135 条流；仅 section#0（off=22）不同，8,138 B，30 字节位置不同；文件 −35 B |
| 顶层脚本与流内值互证 | @C:\Project\hy_analysis\x2\tzl\tzl_root\modify_802_v2.py@ | @Select-String -Path '<path>' -Pattern 'new_values'@ | @[40,34,32,38,36,26,42,24,30,20,18,28,16,14,12,22,10,6,8,4]@ |
| 锻造/洗练概率常量 | @…\tzl_root\tools\fake_flash_server.py@ | @Select-String -Path '<path>' -Pattern 'success_rates_by_quality'@ | @{1:50,2:40,3:30}@（54143）、@>= 70@（56861） |
| GM 无鉴权 | @…\tzl_root\tools\static_flash_http.py@ | @Select-String -Path '<path>' -Pattern '_require_gm_auth' -Context 0,8@ | 函数体 @return True@，原校验被删除 |
| dev 脚本与截图 | @…\tzl_root\@、@…\tzl_root\tools\@ | @(Get-ChildItem '<root>' -Filter *.py).Count@ | 根 20 个 @.py@ + 13 张 @.png@；@tools@ 192 个 @.py@（@fix_*@ 108 个） |

```python
# pair.py —— 三向账目（Python 3.14，只读）
import os
def load(name, base=r"C:\Project\hy_analysis\deep"):
    d = {}
    for line in open(os.path.join(base, "hash_%s.tsv" % name), encoding="utf-8"):
        line = line.rstrip("\n")
        if line:
            h, sz, rel = line.split("\t", 2)
            d[rel] = (h, int(sz))
    return d
A, B = load("211"), load("tzl")
same = changed = added = 0
for r, (h, s) in B.items():
    if r not in A: added += 1
    elif A[r][0] == h: same += 1
    else: changed += 1
removed = len(set(A) - set(B))
print(same, changed, added, removed)   # -> 5730 9 316 0
```

```python
# gamedata.py —— GameData.dat 逐流比对（Python 3.14，只读）
import zlib
def secs(p):
    data = open(p, "rb").read(); out = []; i = 0
    while i < len(data) - 1:
        if data[i] == 0x78 and data[i+1] in (0x01, 0x5e, 0x9c, 0xda):
            d = zlib.decompressobj()
            try:
                out.append((i, d.decompress(data[i:])))
                i += max(len(data) - i - len(d.unused_data), 2); continue
            except Exception:
                pass
        i += 1
    return data, out
a, sa = secs(r"C:\Project\hy_analysis\x\Nya211\Nya-2.1.1-本地一键端-无存档-手动云更新-20260805-v8\www\act_web_nya_208_isolated\dat\GameData.dat")
b, sb = secs(r"C:\Project\hy_analysis\x2\tzl\tzl_root\www\act_web_nya_208_isolated\dat\GameData.dat")
diff = [k for k in range(len(sa)) if sa[k][1] != sb[k][1]]
print(len(a), len(b), len(sa), len(sb), diff)                 # -> 2677996 2677961 5135 5135 [0]
print(len(sa[0][1]), sum(1 for x, y in zip(sa[0][1], sb[0][1]) if x != y))   # -> 8138 30
```

---

## 9. 未决问题

1. **level→value 映射未解**：意志流中 20 个属性槽与 1-20 级的对应关系需要解析 AMF3 键引用（key ref）才能确认；@report_F_tzl.md@ 的"按等级严格递增"结论**本次未能复现**，仅确认值集合由 2..40 上移到 4..42。
2. **物品 602 系未单独核对**：任务书提到"801/802/602 系"，实测全文件只有 1 条流变化，ASCII 键中仅见 801（@damage_reflect@）与 802 相关字段；602 是否在同一流内、是否被改动**未验证**。
3. **"攻击回血 2% 传说特性"**：服务端无实现；本包 @MainLib.swf@ 与基线逐字节相同，故更可能只是 docx 的未实现条目（客户端侧未另行验证）。
4. **@tzl_tmp@ 目录**：解包中间产物，未纳入本次账目。
5. 作者身份未做外部信息追溯（超出任务范围）。
