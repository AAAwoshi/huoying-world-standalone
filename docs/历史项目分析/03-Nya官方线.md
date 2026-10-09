# Nya 官方线（2.1.1 v8 → 2.2.0 → 2.2.3）全量分析报告

> 对象：
> · [res/火影世界一键端.zip](../../res/火影世界一键端.zip)（内层 **2.1.1 v8**）1,207,190,913 B｜SHA-256 `96F8CF82A99ADD71170881F9A8534CF04F5937161209CDB5FC504223FB000BF5`
> · [res/火影世界单机端2.2.1(九月).zip](../../res/火影世界单机端2.2.1(九月).zip)（实为 **2.2.0**）1,243,146,683 B｜SHA-256 `D1D35AB866ED20021FD86405E43213861766769B83D816D9A16AB7A3E95B5648`
> · [res/单机-九月原版/火影世界单机端2.2.3.zip](../../res/单机-九月原版/火影世界单机端2.2.3.zip)（官方 **2.2.3**）1,237,850,222 B｜SHA-256 `14089052D96C5D4795AA1D0027FC02AC7987C9099255EA3799076CC6B2CD7EC5`
> 解包位置：`C:\Project\hy_analysis\x\Nya211\Nya-2.1.1-本地一键端-无存档-手动云更新-20260805-v8`（下称 **T1**）｜[versions/2.2.1/](../../versions/2.2.1/)（下称 **T3**）｜`C:\Project\hy_analysis\x2\v223`（下称 **V223**）
> 基线：2.1.1 v8（三包中唯一无存档的发行基线）｜分析日期：2026-10-07
> 姊妹篇：开发期快照（`res/火影世界单机版.zip`，下称 **T2**）见 [04-Nya开发快照.md](04-Nya开发快照.md)；上游总纲见 [源码对比分析.md](源码对比分析.md) §10（Nya 线源码级全量差异）。

---

## 0. 结论速览

1. **Nya 官方线是纯 Python 标准库复刻**：无 Java、无 MySQL、无第三方依赖，核心是单文件 `tools/fake_flash_server.py`（2,471,309 B / 57,142 行 → 2,656,566 B / 61,417 行），自带 Python 3.13.14 embeddable 运行时（`runtime/python/` 34 个文件，三版零差异）。
2. **三个包是同一条开发线在 08-05 / 08-10 / 08-12 的三次打包**，不是派生关系：T3 的资产目录里保留着 **7/25~8/4 的 5 个 MainLib 祖先备份**（T1 发行包内一个备份都没有），而 T2 的 11 个 08-06~08-08 备份在 T3 中**全部消失**。
3. **命名与版本号陷阱**：`2.2.1(九月).zip` 内层实为 **2.2.0**（`releaseId nya-local-2.2.0-20260810`，versionCode 10）；真正的 2.2.3 在 `单机-九月原版/`（versionCode 13）。
4. **`.nya-version.json` 的 `fileCount` 不是包内文件总数**，而是「该次更新实际下载替换的文件数」（`tools/local_package_update.ps1:391 Write-VersionState … $changes.Count`）。实测：T1 写 5604 / 实际 5,739；T3 写 5938 / 实际 21,161；V223 写 21,150 / 实际 21,196。
5. **协议规模**：ClientOpcode 119→122（+3）、ServerOpcode 158→164（+6），T1→T3 净增 **9 个枚举成员**（旧文档写「净增 8 个」但列了 9 个名字）；**2.2.0→2.2.3 的 opcode 枚举零变化**（V223 与 T3 完全一致），2.2.3 的协议改动全在**载荷语义**上（客户端伤害声明串、四类攻击包识别）。
6. **GM 后台动作数 14 → 59 → 57 → 60**（2.2.3 在 2.2.0 基础上 +3：`repair-equipment-lines`/`reset-account`/`delete-account`）。GM 鉴权走过山车：v8 常量时间口令校验 → 开发快照**整体关闭**（见 04 篇）→ 2.2.0 恢复双口令 + `/api/ops/*` 三层门禁。
7. **2.2.3 是「文档写明 5 类 + 未宣称约 10 组」的实际落地**：PVE 结算重写、异步出站队列 + 20Hz 独立 Tick、存档代际保护、技能射程全局削顶（魔法 1200→**500**）、狂暴野外 Boss、第一演习场整波次系统（`COPY1_MOBS_ON_MAP=6`）；MainLib.swf 为**纯追加式 ABC 注入**（新增 118 条常量池字符串 / 删除 0）。
8. **包体卫生普遍不达标**：T3 携带作者 3 个测试账号存档与 3 组 `test-*` 日志并保留 3 处指向已删除 Ruffle 的死链；V223 携带 `testfull`/`acpacket0815` 测试账号、565 KB 满配测试角色、35 个 pyc（含 `logic-candidate`/`reference_candidate` 两个无源码候选实现）。公开收录前必须清洗。

---

## 1. 身份与来源判定

### 1.1 版本元数据对照（`.nya-version.json`，逐字段实测）

| 项 | T1 = 2.1.1 v8 | T3 = 2.2.0 | V223 = 2.2.3 |
|---|---|---|---|
| 文件字节 | 144 | 161 | 155 |
| SHA-256 | `7a47205a61b8c333…` | `8e00e4ef5782e92a…` | `4100dfb6bfa2ce01…` |
| `releaseId` | `nya-local-20260805-numeric-version-v8` | `nya-local-2.2.0-20260810` | `nya-local-2.2.3-20260812` |
| `version`（人类可读） | 无此字段 | `"2.2.0"` | `"2.2.3"` |
| `versionCode` | 8 | 10（跳过 9） | 13 |
| `updatedAt` | 2026-08-05T19:12:32+08:00 | 2026-08-10T18:39:45+08:00 | 2026-08-12T19:00:00+08:00 |
| `fileCount`（自述） | 5,604 | 5,938 | 21,150 |
| **实测文件数 / 字节数** | **5,739 / 795,268,566 B（758.4 MB）** | **21,161 / 1,316,577,970 B（1,255.6 MB）** | **21,196 / 1,322,363,344 B（1,261.1 MB）** |
| 当前磁盘再点算 | 5,739（一致） | 21,165（+4，为分析工具生成的 `cpython-314` pyc） | 21,202 / 1,326,319,366 B（+6，同上） |
| 客户端形态 | CefFlashBrowser + 浏览器 Ruffle | NyaMicroClient（原生 Flash 34 + Magpie） | 同 2.2.0 |
| 入口 | `启动Nya登录器.bat`（10 个 bat） | `Nya本地登录器.exe`（0 个 bat） | `Nya本地登录器.exe`（0 个 bat） |

> 口径声明：本报告的「实测」一律指**哈希清单口径**（`hy_analysis\deep\hash_211.tsv` / `hash_221.tsv` / `hash_v223.tsv`）。**21,202 / 1,264.9 MB** 是含 6 个 2026-10-07 01:09-01:10 由分析脚本（CPython 3.14 import `tools/*.py`）生成的 `tools/__pycache__/*.cpython-314.pyc` 的磁盘再点算值——**这 6 个文件不属于原始包**，README 与部分任务书写 21,196 与 21,202 的分歧即源于此，本报告统一以 21,196 / 1,261.1 MB 为准。

### 1.2 `fileCount` 口径的源码级解释（本次实测定案）

`tools/local_package_update.ps1` 是唯一写 `.nya-version.json` 的代码，其 `Write-VersionState`（L53-70）由两处调用：

- L311-312：本次更新 **0 个文件需要替换**时写 `fileCount = 0`；
- L391：`Write-VersionState $resolvedRoot $releaseId $versionCode $changes.Count`，而 `$changes` 是循环中**逐个下载并通过 SHA-256 校验后加入的列表**（L301-305），清单里已与本地一致的条目在 L280-287 被 `continue` 跳过。

即 **`fileCount` = 该次更新实际落盘的文件数，与包内文件总数无关**。三棵树的自述值与实测值的差额（135 / 15,223 / 46）没有统一规律，因为它们是三次不同的更新动作留下的状态值，不能反推包内容。

另注：T1/T3/V223 的 `.nya-version.json` 都**没有** `version` 以外的差异字段，但 `version` 字段**不在** 包内 `local_package_update.ps1` 的写入 schema 里——说明该文件由更新工具链之外的新版打包器写入，包内 ps1 已是孤儿脚本（2.2.0 起 `06_手动检查更新.bat` 与 `PACKAGE_MANIFEST.json` 一并被删，云更新路线整体退场）。

### 1.3 版本谱系（实测证据）

```
作者开发目录（F:\NarutoSingleplayer，2026-07-25 起）
  ├─ 2026-08-05  打包 T1 = 2.1.1 v8（无存档、LF 行尾、带 NyaLauncher C# 源码与 10 个 bat）
  ├─ 2026-08-06~08 继续开发 → T2 单机版快照（带他人存档、SQLite 实验、11 个 MainLib 备份）※见 04 篇
  ├─ 2026-08-10  在 .codex-temp\nya211-rebuild-20260810-v1\…-20260808-v9-final 重建 → 打包 T3 = 2.2.0（全树 CRLF）
  └─ 2026-08-12  继续迭代 → 打包 V223 = 2.2.3（全树 LF，随包附 08-14 修复文档）
```

血缘判据（三条互相独立）：① T3 独有 5 个 07-25~08-04 的 MainLib 备份（`before-v14`/`-v130`/`-v131`/`-v132`/`-v245`），T1 一个都没有；② T2 的 11 个 08-06~08-08 备份在 T3 中全部消失；③ T3 存档 `save/multiplayer/accounts.json` 的 `savePath` 与 `logs/local-server.err.log` 的堆栈各自暴露 `.codex-temp` 构建路径。

---

## 2. 包结构与文件账目

### 2.1 顶层结构与体量

| 顶层项 | T1（2.1.1 v8） | T3（2.2.0） | V223（2.2.3） |
|---|---:|---:|---:|
| 文件总数 / 字节 | 5,739 / 795,268,566 | 21,161 / 1,316,577,970 | 21,196 / 1,322,363,344 |
| `www/` | 5,573 / 541.6 MB | 5,566 / 514.6 MB | 5,572 / 502.5 MB |
| `resources/editor_previews/` | — | 15,325 / 618.0 MB | 15,325 / 618.0 MB |
| `NyaMicroClient/` | — | 194 / 91.4 MB | 193 / 91.3 MB |
| `CefFlashBrowser_x64_v1.1.7/` | 84 / 191.7 MB | —（删除） | — |
| `runtime/python/` | 34 / 20.3 MB | 34 / 20.3 MB | 34 / 20.3 MB |
| `tools/` | 19 / 2.8 MB | 28 / 6.9 MB | 64 / 23.3 MB |
| `NyaLauncher/`（C# 源码） | 15 / 0.3 MB | —（删除） | — |
| `save/` | 0（声明无存档） | 6 / 0.1 MB | 8 / 0.6 MB |
| `logs/` | — | 8 | —（删除） |
| 根目录 | 10 bat + `Nya本地登录器.exe` + 说明 + 清单 | 登录器 + 编辑器 + 说明 + `.nya-version.json` | 登录器 + 编辑器 + 说明 + 版本文件 + **2 份修复文档** |

### 2.2 与基线的增 / 删 / 改账目

| 方向 | 完全一致 | 内容改动 | 新增 | 删除 |
|---|---:|---:|---:|---:|
| T1 → T3 | 5,582 | 37（+1.1 MB；其中 **16 个去 `\r` 后逐字节相同**） | 15,542（+715.8 MB） | 120（−219.8 MB） |
| T3 → V223 | 21,088 | **57**（26 纯行尾 + 18 文本 + 13 二进制） | 51 | 16 |
| T1 → T2（开发快照，详见 04 篇） | 5,707 | 10（+0.46 MB） | 536（+231.7 MB） | 22（−10.4 MB） |

T1→T3 新增的 715.8 MB 里 **618.0 MB 是 `resources/editor_previews`**（15,302 PNG + 24 SVG）、91.4 MB 是 NyaMicroClient；删除的 219.8 MB 里 191.7 MB 是 CefFlashBrowser、27.7 MB 是 `www/ruffle/` 等浏览器路线资产。

T3→V223 新增 51 项的大头：两份修复文档、`tools/make_testfull_complete.py`（13,651 B）、`tools/fake_flash_server.py.before-confirmed-hit-only-20260812-182631`（2,632,261 B，08-12 18:26 中间态备份）、编辑器 history `00000033-44`（12 条）、2 个 `.skill-cooldowns.json` 侧车、`UISwf_RouQuanChar1.swf`、24 个随包 `__pycache__` pyc；删除 16 项为 7 个 `MainLib.before-*/other-profile-*` 开发备份 + 7 个 `logs` + `Thumbs.db`。

### 2.3 行尾（EOL）三版方向——diff 噪声的主要来源

| 树 | `tools/fake_flash_server.py` 字节 | 行数 | CR 数 | 结论 |
|---|---:|---:|---:|---|
| T1 | 2,471,309 | 57,142 | 0 | 全 LF |
| T2（开发快照） | 2,711,058 | 61,299 | 60,821 | 混用（编辑器写回） |
| T3（2.2.0） | 2,645,869 | 59,798 | 59,798 | **全 CRLF** |
| V223（2.2.3） | 2,656,566 | 61,417 | 0 | **全 LF** |

同一文件的「字节变化」里相当一部分是行尾迁移：T1→T3 的 37 个改动文件中 16 个纯行尾；T3→V223 的 57 个改动文件中 26 个纯行尾。**做魔改 diff 前必须先归一化 EOL**，否则 2.2.3 一侧会凭空多出约 6 万行噪声。

---

## 3. 技术栈与架构

### 3.1 运行时与启动链

| 环节 | 实测 |
|---|---|
| 入口（T1） | `启动Nya登录器.bat`(202 B) → `Nya本地登录器.exe` 1.3.0（1,847,792 B）；或 `00_一键启动本地端.bat`(515 B) |
| 入口（T3/V223） | 双击 `Nya本地登录器.exe` 2.2.0 / 2.2.3（1,843,696 B，**0 个 bat**） |
| 服务拉起 | `local_package_server.ps1 -Action Start` → `runtime\python\python.exe -u tools\start_nya_208_isolated.py`，stdout/err 重定向到 `logs\local-server.out.log/.err.log`，并写 `local-server.pid` |
| 端口守卫 | `@(19280,19281,19284,19285,19286)` 占用即退出（exit 2） |
| 端口定义（`start_nya_208_isolated.py:26-32`） | HTTP `19280`、WS 桥 `19281`、游戏 `19284`、频道 `CHANNEL_PORT = GAME_PORT+1 = 19285`、原生登录 `NATIVE_LOGIN_PORT = GAME_PORT+2 = 19286`；另有 Flash 策略端口 **843**（由 `fake_flash_server.py` 起） |
| 绑定与公告 | `resolve_network_hosts()`（L40-52）：`NARUTO_LOCAL_ONLY` 为真 → `(True, 127.0.0.1, 127.0.0.1)`；否则 **默认 `0.0.0.0` + `DEFAULT_PUBLIC_HOST = "27.184.1.224"`**（L33/L45-48） |
| python313._pth | T1/V223：87 B、`cfd93fe96c5fa738…`、LF；T3：93 B、`f8acd16e639451e2…`、CRLF 且多一个空行。内容一律 `python313.zip` / `.` / `#import site`（**site 仍被注释=禁用**） / `..\..\tools`（把包根 tools/ 注入 `sys.path`）；**零功能差异** |
| 运行时版本 | Python 3.13.14（`python.exe` 105,696 B / `python313.dll` 6,132,448 B，三版同哈希） |

> 只有 `01_启动服务器.bat`（设 13 个 `NARUTO_*` 环境变量，含 `NARUTO_LOCAL_ONLY=1`）或登录器才会把服务压回 127.0.0.1。**脱离 bat 直接跑 `start_nya_208_isolated.py` 就是 0.0.0.0 + 公网 IP 公告模式**——这正是开发快照里出现 `0.0.0.0` 日志的根因（见 [04-Nya开发快照.md](04-Nya开发快照.md) §6）。
> T3 对 `local_package_server.ps1` 的唯一实质修复（2,656→3,289 B）：`Start-Process` 前把重复的 `PATH`/`Path` 环境变量去重，修「某些机器上两个大小写变量并存导致 Start-Process 直接中止、Python 起不来」。

### 3.2 数据完整性机制

`www/act_web_nya_208_isolated/nya211-gamedata-authority.json` 提供 `runtimeGameDataSha256` 权威基准，`start_nya_208_isolated.py` 启动时对 `dat/GameData.dat` 现算 SHA-256 比对，**不一致即拒绝启动**。三版实测均自洽：

| 树 | GameData.dat | 实测 SHA-256（前 16） | 权威清单值 | 一致 |
|---|---:|---|---|---|
| T1 | 2,677,996 | `091a4ca35ea9b06f` | `091a4ca35ea9b06f` | ✅ |
| T3 | 2,678,097 | `6e7f7639f4a2ce14` | `6e7f7639f4a2ce14` | ✅ |
| V223 | 2,678,036 | `84d8aface013b77d` | `84d8aface013b77d` | ✅ |

---

## 4. 核心实现（源码级证据）

### 4.1 协议栈

| 项 | 证据 | 实测 |
|---|---|---|
| 包布局 | `tools/fake_flash_server.py` L11924 docstring | `"Recognize the original sid/opcode/body/checksum packet layout."` |
| 校验和 | 同文件 L13107-13108 | `checksum = (binascii.crc_hqx(opcode_payload, 0) * 31 + sid) & 0xFFFF`；`struct.pack(">HH", checksum, sid) + opcode_payload` |
| 密钥材料 | 同文件 L51 | `XOR_KEY = "nehc6666123.."` |
| 载荷置换表 | 同文件 T3 L392-409 | **`ORIGINAL_CLIENT_DECODE_BOX`：256 字节完整置换（0..255）**，仅对上行包 `packet[4:-2]` 逐字节逆替换（详见下段） |
| AMF3 | 同文件 L5097 `class Amf3Reader`；全文 `AMF3` 29 处、`amf3` 16 处 | 自研 AMF3 读写（`.dat` 与登录序列化共用） |
| opcode 表 | `class ClientOpcode` / `class ServerOpcode`（AST 枚举计数） | T1 119/158 → T3 122/164 → **V223 122/164（零变化）** |

**T1→T3 协议净增 9 个枚举成员**（旧稿《Nya线三包源码级全量差异.md》写「净增 8 个 opcode」但正文列了 9 个名字；该旧稿已并入 [源码对比分析.md](源码对比分析.md) §10，本报告以 AST 计数为准）：

| 枚举 | 名称 | 值 | 首次出现 |
|---|---|---:|---|
| Client | `AUTO_SELL_CONFIG` | 7779 | T2 |
| Client | `NEWS` | 7780 | T2 |
| Client | `EFFECT_SWITCH` | 7781 | T2 |
| Server | `NEWS` | 7780 | T2 |
| Server | `EFFECT_SWITCH` | 7781 | T2 |
| Server | `SKILL_CHANGE_TMP` | 1813 | T2 |
| Server | `UPDATE_CHAR_SKILL_LEVEL` | 1348 | T2 |
| Server | `WASH_SPECIAL_ATTR_RESULT` | 2068 | T2 |
| Server | `REMOVE_DROP_ITEM_OWNER` | 1049 | T3 |

**「256 字节 S 盒」已确证存在（表名 `ORIGINAL_CLIENT_DECODE_BOX`）**——本节为复核修正：原判「未验证」是方法缺陷导致的**假阴性**。四棵树逐一实测，表体均为 **256 项、值域 0..255、256 个互异值、`sorted(values) == list(range(256))` → 完整置换**，且四棵树的 256 字节序列**逐字节相同**（序列 SHA-256 = `7826b18d63f08ed3…`）。

| 树 | 定义行（`tools/fake_flash_server.py`） | 上下文注释 | 解码函数 |
|---|---|---|---|
| T1（2.1.1 v8） | `:298-315` | L295-297 | `decode_original_client_payload()` L318，查表 L320 |
| T2（开发快照） | `:411-428` | L408-410 | L431 / L433 |
| T3（2.2.0） | `:392-409` | L389-391 | L412 / L414 |
| V223（2.2.3） | `:402-419` | L399-401 | L422 / L424 |

用途（T3 L389-391 注释原文）：`The unpacked desktop client substitutes every request payload byte before it reaches the socket. Opcodes and framing stay plain; only packet[4:-2] uses this inverse table (CodecUtil.DECODE_BOX in the original MainLib).` 即**仅对客户端上行包的载荷段 `packet[4:-2]` 做逐字节逆替换**，opcode 与帧结构（含 `crc_hqx` 校验和）保持明文；`CodecUtil.DECODE_BOX` 指明该表对应 2012 原版 `MainLib.swf` 的 ActionScript 实现。全树无 `encode_original_client_payload`（0 处），是**单向解码表**。

**此前漏判原因（已更正）**：① 该表排版为 **16 行 × 每行 16 个十进制数**，而当时判据是「单行含 ≥200 个十进制数」，结构上不可能命中；② 关键词判据用的是 `sbox`/`S_BOX`/`S盒`/置换表，而实际命名 `ORIGINAL_CLIENT_DECODE_BOX` 不含 `sbox`/`S_BOX` 子串。两条判据同时失效 → 假阴性。协议栈的完整表述应为：**`crc_hqx*31+sid` 校验和 + 256 字节 `ORIGINAL_CLIENT_DECODE_BOX` 置换表（仅载荷段）+ 13 字符 `XOR_KEY` + AMF3**。

### 4.2 存储与社交

| 系统 | 证据（V223） | 实测 |
|---|---|---|
| 存档格式 | `save/**/*.json` 纯 JSON；`tools/storage_backend.py` 3,609 B（LF） | docstring 明言 `deliberately keeps saves as ordinary JSON`；SQLite 实验已在 T3 放弃 |
| 口令哈希 | `save/multiplayer/accounts.json` | `pbkdf2_sha256$260000$<b64salt>$<b64hash>`（PBKDF2-HMAC-SHA256，**260,000 次迭代**，Django 风格） |
| 家族 | `fake_flash_server.py` 命中 `family` 597 次、`multiplayer_backend.py` 195 次 | 家族/族旗/税率/家族副本 |
| 师徒 / 婚缘 | `tutor` 123/107 次、`spouse` 123/36 次 | 师徒与结婚（`spouseAccountId`/`bondAt`/`bondItem`） |
| 鲜花 | `flower` 40/40 次 | 收送花计数与日志 |
| 摆摊 / 拍卖 / 悬赏 | `stall` 182、`auction` 57、`bounty` 91 次 | 三套独立系统 + `accounts-stalls/auctions/bounties.json` |
| 世界 Boss | `world_boss`/`WORLD_BOSS` 114/91 次 | 刷新 + 广播 + 狂暴（2.2.3 新增） |
| 假人 | `tools/simulated_player_ai.py` | `MAX_TEST_PLAYERS = 5`（**四棵树全部为 5**）；T2 实验版扩到 57,043 B，T3/V223 内容回归基线 36,466 B（仅行尾差） |

### 4.3 GM 后台

| 口径 | T1 | T2 | T3 | V223 |
|---|---:|---:|---:|---:|
| 后端 `ACTIONS`（AST 精确） | **14** | **59** | **57** | **60** |
| 前端唯一 action 字面量（全文正则，口径与旧文档不同） | 13 | 59 | 60 | 63 |
| `gm_admin.py` 字节 | 34,699 | 164,325 | 175,815 | 183,170 |
| `www/gm/index.html` 字节 / 页签 | 28,425 / 4 | 108,342 / 9 | 110,866 / 9 | 111,860 / 9 |
| GM 口令常量 | `NyaLocal#208` | 同（但**校验被关闭**） | 同（双口令兼容 + `/api/ops/*`） | 同 |
| `GM_AP_MAX` | — | `99_999_999` | `65_535` | `65_535` |

2.2.3 相对 2.2.0 的 GM 差集：**+3 动作** `repair-equipment-lines`（按品级补齐装备词条，覆盖背包/仓库/穿戴/宠装）、`reset-account`、`delete-account`（强制离线 + 全账号名确认 + 自动备份 `save_backup_gm_account_*` + 解绑好友/家族/师徒/婚缘），**0 删除**。

---

## 5. 与基线的逐项差异

### 5.1 三版核心文件对照

| 文件 | T1（2.1.1 v8） | T3（2.2.0） | V223（2.2.3） | 性质 |
|---|---:|---:|---:|---|
| `fake_flash_server.py` | 2,471,309（57,142 行） | 2,645,869（59,798 行） | 2,656,566（61,417 行） | 服务端巨石 |
| `start_nya_208_isolated.py` | 67,089 | 69,527 | 68,211（LF） | LIANZHAN 缓冲补发 |
| `gm_admin.py` | 34,699 | 175,815 | 183,170 | GM 主体 |
| `static_flash_http.py` | 37,923 | 53,686 | 53,686 | `/api/ops/*` 门禁 |
| `multiplayer_backend.py` | 60,053 | 61,549 | 60,053（LF） | 内容 2.2.3 = 2.2.0（仅 EOL） |
| `simulated_player_ai.py` | 36,466 | 37,361 | 36,466 | **2.2.0 归一化后 = 2.1.1，零内容改动** |
| `storage_backend.py` | — | 3,724 | 3,609 | SQLite → 纯 JSON + 路径穿越防护 |
| `patch_skill_effects_gamedata.py` | 26,486 | 28,993 | 28,304 | 四个特殊技能等级化 |
| `www/gm/index.html` | 28,425 / 4 页签 | 110,866 / 9 页签 | 111,860 / 9 页签 | GM 前端 |
| `MainLib.swf` | 2,137,157 | 2,147,312 | **2,151,529** | 追加式 ABC 注入 |
| `Main.swf` | 513,782 | 513,983 | — | 构建标签 `no-eye-nin-v179` |
| `GameData.dat` | 2,677,996 | 2,678,097 | 2,678,036 | 技能/装备数据 |

### 5.2 2.2.0 → 2.2.3 的官方变更（逐项落码验证）

官方随包发布两份文档（`2.2.3修复总结-简版-20260814.txt` 2,836 B mtime 08-14 22:29；`2.2.3最新修复清单-20260814.md` 15,663 B mtime 08-14 22:28，含「原问题 / 修复后 / 如何修复 / 当前测试结果」四列方法论表与「自动测试通过 / 结构验证通过 / 实机待确认 / 未宣称修复完成」状态分级）。逐条对照代码：

| # | 官方宣称 | 落码证据（V223 `tools/fake_flash_server.py`） | 与 T3 对比 | 判定 |
|---|---|---|---|---|
| 1 | 关闭施法阶段结算，修「一次攻击两次伤害」 | `PVE_ANNOUNCE_SETTLE_ENABLED = False` | T3/T2 均为 `True` | ✅ 属实 |
| 2 | 命中包带客户端伤害声明则提交声明，否则服务端补算 | `_client_attack_claims()` / `_settle_client_pve_claim()` 各 2 处引用 | T3 均 **0** 处 | ✅ 属实 |
| 3 | 四类攻击（普攻/近战/远程/魔法）统一识别 opcode | `_attack_impact_opcode()` 5 处引用 | T3 0 处 | ✅ 属实 |
| 4 | 本人不重播服务端同一次伤害 | `include_self=not client_has_claims` + SWF `onMobRealAttack` 拦截 | T3 0 处 | ✅ 属实 |
| 5 | 出站改异步队列 | `_enqueue_outbound_locked` 3 处、`_run_outbound_sender` 2 处 | T3 0 处 | ✅ 属实 |
| 6 | 世界 Tick 与收包线程分离（约 20Hz） | `_run_runtime_loop` + `SOCKET_TICK_SECONDS = 0.05` | T3 0 处 | ✅ 属实 |
| 7 | 存档编码/落盘移出角色锁 + 代际保护 | `_snapshot_payload` 3、`_save_generation` 3、`_last_written_generation` 4、`_run_save_worker` 2、`COMBAT_SAVE_MAX_DELAY_SECONDS` | T3 全 0 | ✅ 属实 |
| 8 | 升级属性点改为 70 级前 5 点/级、70 级起 10 点/级并幂等迁移 | `level_attribute_point_total`、`migrate_level_attribute_rules`（特征标记 `_levelAttributePointsV220`） | T3 0 处 | ✅ 属实 |
| 9 | 掉落时固定品级与词条数 | `equipment_plus_attribute_count` 3 处 | T3 0 处 | ✅ 属实 |
| 10 | 死亡/复活/换线清仇恨 + 缩短追击 + 脱战归位 | `_clear_monster_aggro_for_character`（5 个调用点）、`MONSTER_CHASE_DISTANCE = 320` | T3 有同名常量但无清仇恨函数 | ✅ 属实 |
| 11 | 狂暴野外 Boss | `_roll_wild_boss_frenzy`、`_wild_boss_frenzy_config`、`WILD_BOSS_FRENZY_MARKER`（读编辑器 `balance_rules.wild_boss_frenzy`） | T3 0 处 | ✅ 属实（官方自述「是否进入成品需实机核验」） |

**文档未宣称、实测存在的约 10 组改动**：

1. **技能射程全局削顶**：`min(value, 500.0)` 硬上限 + 缺省表下调（魔法 1200→**500**，默认 650→350）——与「canJump 追击限制」配套的反狙击设计，文档只字未提。
2. **VIP 卡与充值 VIP 分离**：持卡者传送免费（`free_vip_card_transfer`），command 1 只反映卡等级。
3. **传送门守卫收紧**：`PORTAL_STALE_REQUEST_GUARD_SECONDS` **8.0 → 2.0**，缺对侧 `tn` 时回落出生门。
4. **昆虫（KunChong）强化补全**：幸运符 `probUp/maxHelpCnt`、保底符 `guarantee`、穿戴位 73-76 槽位映射修正。
5. **宝石合成概率化**：`STONE_MERGE_RESPONSE` 接入 `merge.levelN.prob` 并支持失败分支。
6. **橙装打造门槛放开（疑似误删）**：删掉 `strength_level < 10` 与 `required_level < 60` 两个前置校验，却**保留旧错误文案**「只有60级以上的紫色10星装备」——行为与提示不一致。
7. **传说属性百分比类改摇 1-5**（ATTACK_RATE / DAMAGE_REFLECT / DAMAGE_REDUCE / IGNORE_DEFENCE）。
8. **排行称号改按成就点数**（`CHENG_JIU: achievement_points`）+ 去重保序 + 达标判定边界修正（`<=` → `<`）。
9. **第一演习场（copy_id=1）整波次系统**——规模最大的未宣称改动：`COPY1_MOBS_ON_MAP = 6`（注释明言「降低 Flash 渲染压力」）、`COPY1_MONSTER_CHASE_DISTANCE`、`COPY1_PRESENCE_CAP`、`_prewarm_copy1_combat_caches()`、`_copy1_refill / _copy1_wave / _copy1_boss / _copy1_spawn_cache` 等状态族。**口径修正**：本报告实测带 `def` 定义的 `_copy1_*` **新函数为 4 个**（`_copy1_spawn_positions`、`_copy1_alive_commons`、`_copy1_spawn_cache_build`、`_copy1_next_object_id`），另有 10 个 `self._copy1_*` 实例字段与 3 个 `COPY1_*` 常量；旧报告 E 的「9 个新函数」与本次 AST 计数不符。
10. **副本协议字段 `COPY_ACTION:5` 语义变更**：由 `成员数:0:角色ID:0` 改为 `copy_type:pass_time:角色ID:min_cost_time`，对齐原版副本窗的「首次通关 / 最佳记录」显示。
11. **成就奖励实发 + 通知去重 + 任务 5019 跨图豁免 + 掉落候选缓存**等一批健壮性改动。

**客户端注入（MainLib.swf，反向验证）**：

| 项 | T3（2.2.0） | V223（2.2.3） |
|---|---:|---:|
| 文件字节 | 2,147,312 | 2,151,529 |
| 解压后 SWF body | 5,138,562 | 5,151,059（+12,497） |
| `DoABC2`(tag 82) 载荷 | 4,833,527 | 4,846,024（+12,497） |
| ABC 字节 | 4,833,516 | 4,846,013 |
| 常量池唯一字符串 | 51,824 | **51,942（+118，删 0）** |

`+118` 全部是新增，**删除 0** → 纯追加式注入（RABCDAsm 风格补丁，非重编译）。其中**不可再分的短标识符 24 个**（旧报告 E 的「+24 常量池标识符」即此口径），包括 `applyClientPveAttack`、`clientCombatHp`/`clientCombatMaxHp`、`playLocalMobHit`、`Skill_{100|200|300|400}_skill_*001_hit_0`、`preferActiveQuestMonsters`、`moveToPortal`、`_lastPortalCheckX/Y`、`_recentPortal`、`pinBeside`、`eliteCombatMultiplier`、`isFiveHitMouse` 等；其余为限定名形式（如 `com.mxw.game.world.handlers:DropItemHandler/removeDropItemOwner`、`com.mxw.uifl.controls.copy:SingleModeCopyWindow/remove`）。两条口径不矛盾。

**二进制与版本号**：

| 文件 | T3 | V223 | 结论 |
|---|---|---|---|
| `Nya本地登录器.exe` | 1,843,696（FileVersion 2.2.0.0） | 1,843,696（2.2.3.0） | **同尺寸**，仅 22 字节差异=版本资源串等长替换，本体同一构建 |
| `NyaMicroClient/NyaLauncher.exe` | deps.json `NyaLauncher/2.1.2`（runtimeTarget `.NETCoreApp,Version=v8.0/win-x64`） | `NyaLauncher/2.2.1`（RID 去掉 win-x64） | 重编译 |
| `NyaMicroClient/NyaUpdater.exe`（根 + `Updater/` 两份同哈希） | 151,552 | 151,552 | `.dll` 重编译但**版本号仍 2.1.2**——版本管理粗糙 |
| `火影世界数据编辑器.exe` | 6,506,735（PE 2026-08-11 11:26） | 7,435,149（PE **2026-08-14 23:30**） | PyInstaller 重打包 |
| `tools/native_flash_host.exe` | 10,240（PE 2026-07-28 20:31） | 12,288（PE **2026-08-15 10:51**） | 源码 +88/−3 行，加 HWND 嵌入模式 |

> **活目录快照证据**：编辑器 exe(08-14 23:30) 与 native_flash_host.exe(08-15 10:51) 的重编时间**晚于** 08-14 22:28 写就的修复文档——2.2.3 树是文档写完后仍在继续演化的开发目录，不是冻结发布件。

**游戏数据（.dat）**：三个 `.dat` 顶层节数 56/225/13，键集完全一致，2.2.0→2.2.3 只改了 2 条 schema 规范化（`CaiLiao/12100005` 增 `reqJob:0` 与空 `spec`；`XiaoHao/03000001` 增 `reqJob:0`），`String/*` 解压后 0 差异——**没有任何技能/物品数值改动**，2.2.3 的战斗改动全在服务端与 SWF。

### 5.3 客户端形态演进（三阶段）

| 阶段 | 形态 | 组成（实测） |
|---|---|---|
| T1（2.1.1 v8） | 浏览器 Ruffle + CEF | `www/ruffle/` 7 文件 ≈28.1 MB（`ruffle.js` 461,418 B、两份 `core.ruffle.*.js`、两份 wasm 各 ≈14.2 MB、Apache/MIT 双许可）；`ruffle.html` 30,086 B（含中文输入法桥、WebCrypto AES-GCM 记住密码、BroadcastChannel 单实例待机）；`CefFlashBrowser_x64_v1.1.7` 84 文件 / 191.7 MB |
| T1/T2 过渡 | CefFlashBrowser 运行痕迹 | T2 多出 `Caches/`、`Logs/` 与 `LocalPrefs.json`（开发期真跑过） |
| T3/V223（2.2.0/2.2.3） | NyaMicroClient 原生微端 | `NyaMicroClient/` 194 文件 / 91.4 MB：`NyaLauncher.exe`(.NET 8 WPF，deps.json `NyaLauncher/2.1.2`)、`NyaUpdater.exe` 双份、`Runtime/Flash/Flash.ocx` 12,604,704 B（**Adobe Flash Player ActiveX 34,0,0,251**，二进制内嵌该版本串）与同版离线安装器 11,549,539 B、`Runtime/Magpie/` 0.12.1 含 **3 套 Nya 滤镜**：`Nya FSR 清晰平衡`(FSR_EASU + FSR_RCAS sharpness 0.85)、`Nya CuNNy 动漫增强`(CuNNy-4x12-NVL)、`Nya Anime4K 线条增强`(Anime4K_Upscale_Denoise_L)、`Assets/` 11 首 BGM |
| 桥接 | `tools/ruffle_socket_proxy.py`(6,683 B) → `tools/local_socket_proxy.py`(6,683 B，T3 改名) | 内容三版逐字节一致：WS(18681)→TCP(18684) 桥；T3 从「服务浏览器 Ruffle」转为「服务原生微端」 |

---

## 6. 质量、安全与隐私

### 6.1 GM 鉴权三版演进（源码行号）

| 版本 | 实现 | 判定 |
|---|---|---|
| T1 | `static_flash_http.py:523-532`：`hmac.compare_digest(supplied, GM_PASSWORD)`，失败 401 | 有鉴权 |
| T2 | `static_flash_http.py:535-537`：`"""Local single-player GM backend skips password verification.""" return True` | **鉴权整体关闭**（详见 [04-Nya开发快照.md](04-Nya开发快照.md) §6） |
| T3/V223 | `static_flash_http.py:599-612`：`accepted = (GM_PASSWORD, "NyaLocal#208")` 双口令 + `/api/ops/*` 三层门禁（arm cookie 15 min → 解锁码 SHA-256 `27cd7ff0…c2` → 会话 cookie 15 min；原生票据 30 s 一次性且要求 `X-Nya-Native-Client: NyaMicroClient/2.2.0`；远程需 `NARUTO_OPS_REMOTE_SECRET` + https） | 已加固 |

### 6.2 发行卫生问题（T3 / V223）

| 问题 | 实测 |
|---|---|
| 测试存档（T3） | 3 账号 `pk60v12test`/`111111`/`codexbaseline1`，创建于 2026-08-10 16:14:05~16:16:22（**2 分 17 秒内**），角色 不知火疾风 lv80 / 日向白夜 lv1 / 神月焰 lv9；`savePath` 暴露 `.codex-temp\nya211-rebuild-20260810-v1\…v9-final` |
| 测试存档（V223） | `123123`(旗木枫 lv9) / `testfull`(全技能测试 lv120，08-14 建) / `acpacket0815`(封包测试员 lv1，08-15 建)；`characters/1002.json` **565,542 B** 满配测试号随包 |
| 测试日志（T3） | `logs/` 8 文件：`local-server.err.log` 6,324 B（末次 08-10 19:14→19:25 冒烟测试）、`local-server.out.log` 449 B、`test-launch.*`/`test-v3.*`/`test-v3b.*` 三组；另有 `NyaMicroClient/Runtime/Magpie/logs/magpie.log` 118 KB（08-07 21:54~08-08 00:55 开发机实跑） |
| 文档与现实不符 | `使用说明.txt`（T3 561 B / V223 1,049 B，mtime 08-11 20:18）仍引导「停止服务请运行 99_关闭本地端.bat」，而该 bat 在 T3 已随 10 个 bat 一并删除 |
| Ruffle 死链 | `www/local-control.html:23`（`/ruffle.html` 链接）、`www/nya-208-preview.html:106`（`<script src="/ruffle/ruffle.js">`）、`nya-208-isolated-manifest.json:216`（`"launcher": "www/ruffle.html"`）——三处指向已删除资源，V223 **原样继承未修** |
| 孤儿脚本 | `tools/local_package_update.ps1` 在 T3/V223 中**没有任何调用入口**（bat 全删），却仍随包分发 |
| 候选实现残留（V223） | 35 个随包 pyc（cpython-38 ×17、3.13 ×14、3.12 ×4），其中 `bug_reports`、`season_pass` **无对应源码**，`fake_flash_server.logic-candidate`/`fake_flash_server_reference_candidate` 为两个候选实现；另有 `tools/fake_flash_server.py.before-confirmed-hit-only-20260812-182631` 中间态源码备份（2,632,261 B） |

---

## 7. 完成度与收录建议

| 维度 | 2.1.1 v8 | 2.2.0 | 2.2.3 |
|---|---|---|---|
| 可玩性 | 完整（5 端口 + 843） | 完整 + 微端 | 完整 + 战斗/延迟修复 |
| 干净度 | ★★★★★ 无存档、无日志 | ★★☆ 测试存档 + 日志 + 3 处死链 | ★★ 测试账号 + 满配测试档 + 35 pyc |
| 内容量 | — | +618 MB 编辑器预览缓存 + 编辑器 | 同 2.2.0 |
| 收录优先级 | **1（唯一干净基线）** | 3 | **2（功能最完整 + 官方修复文档齐全）** |

建议：① 三包全部收录，但 V223 与 T3 必须在元数据中标注「含作者测试存档/日志，公开前清洗 `save/`、`logs/`、`tools/__pycache__/`」；② 2.2.3 的编辑器 `resources/editor_previews`（618 MB / 15,325 文件）与 2.2.0 逐字节相同，去重时可只留一份；③ 2.2.0 与 2.2.3 只差 2 天，若要压缩仓库体积，保留 2.2.3 + 2.1.1 v8 即可覆盖大部分谱系（但 2.2.0 是 2.2.3 的 diff 基线，删除会使 `x2pair_221__v223_*.txt` 无法复现）。

---

## 8. 证据与核验

所有命令在 `C:\Project\huoying-world-standalone` 下用系统 `python`（3.14）执行；`T1/T3/V223` 均为只读访问。PowerShell 亦可复现同等结果。

| 关键结论 | 依据文件（绝对路径） | 复核命令（要点） | 实测值 |
|---|---|---|---|
| 三包身份（字节 + SHA-256） | `C:\Project\huoying-world-standalone\res\火影世界一键端.zip` 等 4 个原始包 | `python -c "import hashlib;h=hashlib.sha256(open(p,'rb').read());print(len(open(p,'rb').read()),h.hexdigest().upper())"` | 1,207,190,913 `96F8CF82…000BF5`／1,243,146,683 `D1D35AB8…95B5648`／1,237,850,222 `14089052…2CD7EC5` |
| 三棵树文件数/字节 | `hy_analysis\deep\hash_211.tsv`、`hash_221.tsv`、`hash_v223.tsv` | 逐行累加第 2 列 | 5,739/795,268,566；21,161/1,316,577,970；21,196/1,322,363,344 |
| 磁盘多出的 pyc 是分析副产物 | `versions\2.2.1\tools\__pycache__\*.cpython-314.pyc`（4 个）、`x2\v223\tools\__pycache__\*.cpython-314.pyc`（6 个） | `(Get-Item p).LastWriteTime` | 全部为 2026-10-07 01:09-02:36 |
| `fileCount` 口径 | `versions\2.2.1\tools\local_package_update.ps1` L53-70/L280-312/L391 | `Select-String -Path … -Pattern 'Write-VersionState'` | 写入值是 `$changes.Count`（实际替换文件数） |
| 版本元数据 | 三棵树 `.nya-version.json` | `Get-Content -Raw … | ConvertFrom-Json` | vc 8/10/13；fileCount 5604/5938/21150 |
| opcode 枚举 | `tools\fake_flash_server.py`（三棵树） | `ast.parse` → 取 `class ClientOpcode/ServerOpcode` 成员名集合做差 | C 119→122→122；S 158→163→164；T3→V223 差集为空 |
| GM 动作数 | `tools\gm_admin.py`（四棵树） | `ast.parse` → 模块级 `ACTIONS` 字典键 | 14 / 59 / 57 / 60 |
| PVE 结算开关 | `x2\v223\tools\fake_flash_server.py` | `Select-String 'PVE_ANNOUNCE_SETTLE_ENABLED\s*='` | V223=`False`；T3/T2=`True` |
| 20 Hz 独立 Tick | 同上 | `Select-String 'SOCKET_TICK_SECONDS'` | `0.05`（V223 4 处引用，T3 0 处） |
| 射程削顶 | 同上 | `Select-String 'min\(value, 500\.0\)'` | 存在；`PORTAL_STALE_REQUEST_GUARD_SECONDS = 2.0` |
| 第一演习场参数 | 同上 | `Select-String 'COPY1_MOBS_ON_MAP\|def _copy1_'` | `COPY1_MOBS_ON_MAP = 6`；`def _copy1_*` **4 个** |
| MainLib 追加式注入 | `versions\2.2.1\www\act_web_nya_208_isolated\asset\MainLib.swf` 与 `x2\v223\…\MainLib.swf` | 解 CWS → 遍历 tag → 取 `DoABC2`(82) → 解析 ABC 常量池字符串集做差 | 51,824→51,942，**+118 / −0**；DoABC2 4,833,527→4,846,024 |
| GameData 权威校验 | `www\act_web_nya_208_isolated\nya211-gamedata-authority.json` + `dat\GameData.dat` | 现算 SHA-256 与 `runtimeGameDataSha256` 比对 | 三棵树全部 match |
| 256 字节 `ORIGINAL_CLIENT_DECODE_BOX` **确证存在** | 四棵树 `tools\fake_flash_server.py`（T1 `:298-315`、T2 `:411-428`、T3 `:392-409`、V223 `:402-419`） | `re.search(r'ORIGINAL_CLIENT_DECODE_BOX\s*=\s*\(([^)]*)\)', src, re.S)` 抓取数字后断言 `sorted(vals) == list(range(256))` | 四棵树 len=256、distinct=256、0..255 完整置换、序列逐字节相同（SHA-256 `7826b18d63f08ed3…`） |
| 启动链与端口 | `x2\v223\tools\start_nya_208_isolated.py` L26-52 | `Get-Content … -TotalCount 60` | 19280/19281/19284/19285/19286；`DEFAULT_PUBLIC_HOST="27.184.1.224"` |
| GM 鉴权三态 | `static_flash_http.py` L523-537（T1/T2）、L599-612（T3） | `Select-String '_require_gm_auth' -Context 0,8` | `return True`（T2）／双口令 + ops（T3） |
| 2.2.0→2.2.3 账目 | `hy_analysis\deep\x2pair_221__v223_added/changed/removed.txt` | `(Get-Content f).Count` | 51 / 57 / 16 |
| T1→T3 账目 | `hy_analysis\deep\pair_211__221_*.txt` | 同上 + 第 1 列求和 | 5,582 一致 / 37 改 / 15,542 增（+715.8 MB）/ 120 删（−219.8 MB） |

---

## 9. 未决问题

1. **`ORIGINAL_CLIENT_DECODE_BOX` 的 256 字节内容是否被调整过**：四棵树的置换表**逐字节相同**（已实测，序列 SHA-256 一致），即 v8 → 开发快照 → 2.2.0 → 2.2.3 **全程未改动该表**；仍未验证的是它与 2012 原版 `MainLib.swf` 内 `CodecUtil.DECODE_BOX` 的字节级一致性（需反编译原版 SWF，超出本次范围）。
2. **`fileCount` 的具体更新动作无法还原**：只知其为「某次更新实际替换的文件数」，但三个包分别对应哪次更新（从哪个版本升到哪个版本）在包内没有记录。
3. **2.2.3 的 `_copy1_*` 系统是否实机生效未验证**：代码完整落地且被 `COPY1_*` 常量约束，但包内无对应测试记录；官方文档亦未提及该系统。
4. **`save/**.skill-cooldowns.json` 侧车文件的写入时机**（仅 2.2.3 出现）未定位到具体调用点，仅确认为第一演习场玩法引入。
5. **T3→V223 的 57 个改动中 13 个二进制项的语义**（`NyaLauncher.dll`/`NyaUpdater.dll`/3 个 pyc 等）只做到「重编译/版本资源」级别定性，未做 IL 级 diff。
6. **2.2.0 的 `Main.swf` 构建标签 `no-eye-nin-v179` 与 2.2.3 是否同源**未验证（V223 `Main.swf` 未做反编译比对）。
