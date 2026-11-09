# 停用 ColorOS 系统浏览器 `com.heytap.browser`：分析、取舍与可复现流程

> **已脱敏版本**（公开分发用）。文中设备型号、序列号、IP / 端口、配对码、构建号、安装时间戳、APK 指纹、本机路径与日志文件名均已模糊处理；技术结论、命令、判定逻辑与数值结构（类数、字节量级、进程内存）保持原样，不影响复现。
> 复现时以你自己设备上的实测值为准。

> 设备类别：realme / OPPO 系（ColorOS 16，Android 16 / SDK 36）
> 执行窗口：`<执行日期> 18:45 – 18:50`
> 执行方式：本机 adb（无线调试配对，**非 root**）
> 结论：**已挂起（suspend）成功，浏览器进程全部消失，系统核心功能无回归。**

---

## 0. 一句话结论

`com.heytap.browser` 是**出厂预装、开机即运行、长期未停、由应用商店持续更新的完整 Chromium 浏览器**，其商业模式是信息流与原生广告变现。它并非默认浏览器（默认是 Chrome），却长期占用约 1.1 GB 内存并持有大批高危权限。在不使用 root 的前提下，`pm suspend` 是唯一能真正让它停止运行的手段；代价是一个灰色图标与少量浏览器附属功能。

---

## 1. 对象识别

| 字段          | 值                                                                                          | 来源                        |
| ----------- | ------------------------------------------------------------------------------------------ | ------------------------- |
| 包名          | `com.heytap.browser`                                                                       | `dumpsys package`         |
| 版本（运行时）     | `xx.xx.xx.x`（三段式常规版本号）                                                                     | 同上                        |
| 出厂 APK      | `/product/app/Browser/Browser.apk`，约 **275 MB**                                            | `ls -l` on device         |
| 实际运行 APK    | `/data/app/~~<hash>==/com.heytap.browser-<hash>==/base.apk`，约 **219 MB**，sha256 `<SHA256>` | `pm path` + pull + sha256 |
| 更新时间线       | `firstInstallTime` = 购机同期（2023 年 8 月）；`lastUpdateTime` = 停用前约 3 周                          | 同上                        |
| 更新来源        | `installerPackageName=com.heytap.market`（应用商店 vxx.xx.x_CN）                                 | 同上                        |
| 包标志         | `SYSTEM HAS_CODE UPDATED_SYSTEM_APP LARGE_HEAP`                                            | 同上                        |
| appId / uid | `10***` / `u0_a***`                                                                        | `dumpsys package` + `ps`  |
| targetSdk   | 33                                                                                         | APK manifest              |
| 体量          | 21 个 dex，dex 合计约 **150 MB**，**59,944 个类**，15,693 个资源                                       | 解包统计                      |

**要点**：`firstInstallTime` 与购机/首次开机时间对齐；`stopped=false notLaunched=false` 说明它从装机起就没被停过。它是"系统的一部分"，不是用户装的第三方应用。

---

## 2. 它到底是什么（静态取证）

### 2.1 内核与业务能力

Chromium 内核 + 独立渲染沙箱进程（`org.chromium.content.app.SandboxedProcessServiceNBr<Brand>Browser...`），并自带完整业务栈：

- 账号：`com.heytap.usercenter.accountsdk.*`、`com.platform.usercenter.account.ams.*`
- 支付：`com.client.platform.opensdk.pay.*`、`AtlasPayFileProvider`
- 下载：`com.xunlei.downloadprovider.service.XService`（迅雷组件）、自实现 `com.android.providers.downloads.DownloadProvider`
- 书签/历史：`browser.db`、`PropertyContentProvider`
- 备份：`BrowserBRPluginService`（action `com.heytap.br.service`）
- 快应用桥：`org.hapjs.features.channel.ChannelService`

### 2.2 变现层（这才是重点）

| 关键词（DEX 内计数）                                 | 含义                                          |
| -------------------------------------------- | ------------------------------------------- |
| `iflow` 19,650                               | 信息流（含 `com.heytap.heyiflow.net`，域名出现 200 次） |
| `openadsdk` 3,549                            | 字节穿山甲广告 SDK                                 |
| `pangle` 166                                 | 穿山甲海外品牌名                                    |
| `embedapplog` 204、`oaid` 39、`com.asus.msa` 8 | 字节归因/设备标识                                   |
| `com.bytedance.sdk` 24                       | 字节 SDK 基础包                                  |

上报/广告域名（APK 内字符串计数）：

```
com.heytap.heyiflow.net                       200
com.heytap.browser.iflow.net                   53
sf3-fe-tos.pglstatp-toutiao.com                49   ← 穿山甲素材 CDN
cpc-comment-cn.heytapimage.com                 24
mbd.baidu.com                                  18   ← 百度搜索/广告
browser.heytapmobi.com                         13
bcc-szth.baidu.com                              8
com.heytap.nearx.com                            7
log.snssdk.com                                  6   ← 字节埋点
ads.heytapmobi.com                              6
cpc.heytapmobi.com                              6
```

配套组件：付费墙 `TaskWallSharedProvider`、投放统计 `UrlStatProvider`、搜索分成 `SearchEngineProvider`、火山 `zeus` / `playlet`（短剧）。

### 2.3 反证：它不是"纯广告壳"

它有 59,944 个类、完整 Chromium 栈、下载器、书签库、支付与账号 SDK。**它是真浏览器，广告是它的盈利方式**，不是"套了层壳的 SDK"。任何"除了广告啥都没有"的说法都不准确。

---

## 3. 风险面

### 3.1 权限（`dumpsys package` 实测）

```
MANAGE_EXTERNAL_STORAGE      QUERY_ALL_PACKAGES
SYSTEM_ALERT_WINDOW          PACKAGE_USAGE_STATS
MANAGE_ACCOUNTS              RECORD_AUDIO
REQUEST_INSTALL_PACKAGES     GET_TASKS
INTERACT_ACROSS_USERS
```

### 3.2 三个 exported 且无权限保护的保活服务

| 服务                  | action                                             |
| ------------------- | -------------------------------------------------- |
| `DaemonService`     | `oplus.intent.action.browser.keep.alive`           |
| `SlideService`      | `heytap.intent.action.browser.window.server`       |
| `SlideLightService` | `heytap.intent.action.browser.window.server.light` |

任何应用都能通过这三个 action 把它拉起来——这正是它常年常驻、且能被"设置"等无关入口唤起的机制基础。

### 3.3 实测的常驻成本

```
644 MB   com.heytap.browser
226 MB   com.heytap.browser:downloads
228 MB   com.heytap.browser:sandboxed_process0:org.chromium...SandboxedProcessServiceNBr...
```

内存 RSS 排名系统第三（仅次于 system_server 与微信），而默认浏览器是 Chrome——**它是在为一个没人用的功能持续消耗内存**。

### 3.4 被外部拉起的实证

```
<日期> 18:31:04  ACTIVITY_RESUMED  com.heytap.browser/.BrowserActivity
                 taskRootPackage=com.android.settings
                 taskRootClass=...OplusSettingsHomepageActivity
```

从"设置"首页的 task 里被拉起来过一次，不是用户点击图标。

---

## 4. 处置方式的选择

### 4.1 为什么不用 `pm disable-user`

实测两次（user 0 / user 10）：

```
$ pm disable-user --user 0  com.heytap.browser
Package com.heytap.browser new state: default     ← 未生效
```

原因：该包存在**两条 Package 记录**（`/data/app` 更新副本 + `/product/app/Browser` 出厂副本）。`disable-user` 打在更新副本上，状态被回滚为 `default`。

同一现象在 `com.opos.ads` 上伴随明确报错：

```
PackageManager: package shell:1000 uid 2000 call SetEnabledSetting(com.opos.ads component = unknown, 3, 0)
java.lang.SecurityException: Shell cannot change component state for null to 2
```

`com.oplus.statistics.rom` 则是单记录，`disable-user` 正常返回 `disabled-user`（对比样本）。

### 4.2 为什么不用 `pm uninstall --user 0`

可行（图标会彻底消失），但对 `UPDATED_SYSTEM_APP` 属于"卸载"语义，且整包 OTA 后需重做；`suspend` 与它在持久性上等价而语义更轻，回滚更简单（一条 `unsuspend`）。

### 4.3 采用的方案：`pm suspend` + `force-stop`

```
pm suspend --user 0  com.heytap.browser   → new suspended state: true
pm suspend --user 10 com.heytap.browser   → new suspended state: true
am force-stop com.heytap.browser
```

语义：包仍然安装、数据完好、图标变灰且不可启动、进程不允许创建。**可逆**。

---

## 5. 先决条件：工具、品牌、适用对象

### 5.1 必备工具

| 工具                                 | 版本/要求                          | 用途                    | 获取与校验                                                            |
| ---------------------------------- | ------------------------------ | --------------------- | ---------------------------------------------------------------- |
| **adb（platform-tools）**            | `1.0.41 (37.0.1-15733141)` 或更新 | 唯一必需工具，全部操作经由它        | Google 官方 `platform-tools-latest-<os>.zip`，解压即用；校验 `adb version` |
| 数据线 / 无线调试                         | Android 11+ 支持无线配对             | 建立 shell 通道           | 开发者选项 → 无线调试 → 配对码 + `IP:端口`                                     |
| 终端                                 | Windows 10+ / macOS / Linux    | 执行命令                  | —                                                                |
| `sha256sum`                        | 任意                             | 记录 APK 指纹             | 系统自带                                                             |
| **可选**：zip 解包 + `grep`（`LC_ALL=C`） | 任意                             | DEX 关键词计数、域名提取        | 无需安装                                                             |
| **可选**：jadx / JADX-headless        | 需 Java                         | 看 manifest、组件导出状态、类结构 | —                                                                |
| **可选**：apktool                     | 2.9+                           | 反编译资源与 manifest       | —                                                                |

**不需要**：root、Magisk、Xposed、任何安装到手机的 App、电脑端抓包工具。

### 5.2 环境要求

- 手机：开发者选项 + USB 调试；无线调试需手机与电脑同一 Wi-Fi 且 mDNS 可通
- **易踩坑**：`adb connect <PAIR_IP>:<PAIR_PORT>` 被拒是**正常现象**——配对端口 ≠ 连接端口，配对成功后靠 mDNS 自动发现连上即可，不要在配对端口上反复 connect
- Android 版本：**≥ 7.0**（`pm suspend` 自 API 24 引入）；无线调试 ≥ 11
- 权限：shell（uid 2000），**无需 root**
- 时间成本：10–15 分钟（不含静态取证）

### 5.3 品牌 / 机型适用矩阵

| 品牌                                   | 系统                                       | 浏览器包名                                        | 能否直接套用本文流程                                 |
| ------------------------------------ | ---------------------------------------- | -------------------------------------------- | ------------------------------------------ |
| **realme / OPPO / 一加（国行 ColorOS 版）** | ColorOS 12–16、realme UI 4–6、OxygenOS 13+ | `com.heytap.browser`                         | **可以，本文即此场景**（实测 ColorOS 16 / Android 16）  |
| 一加（国际版 OxygenOS）                     | OxygenOS 13+                             | `com.heytap.browser` 或 `com.oneplus.browser` | 先 `pm list packages \| grep -i browser` 确认 |
| 华为 / 荣耀                              | EMUI 12 / HarmonyOS                      | `com.huawei.browser`                         | 命令通用，**包名必须换**；HarmonyOS NEXT 无 adb 包管理语义  |
| 小米 / 红米                              | HyperOS / MIUI 12–15                     | `com.android.browser`                        | 命令通用，换包名；**注意小米常把它设为默认浏览器**                |
| vivo / iQOO                          | OriginOS 3–6                             | `com.vivo.browser`                           | 命令通用，换包名                                   |
| 三星                                   | One UI 5–7                               | `com.sec.android.app.sbrowser`               | 命令通用；部分机型 suspend 后图标仍可点（厂商定制）             |
| 原生 AOSP / Pixel                      | Android 12+                              | 无预装广告浏览器（Chrome 即默认）                         | 不适用                                        |

**通用判定公式**（任何品牌）：

```
包存在  +  cmd role get-role-holders android.app.role.BROWSER 不是它  +  DEX 里有广告 SDK/信息流域名
        ⇒ 适用本文流程
```

任一条不满足就停手（见 §5.5）。

### 5.4 同一套流程可复用的对象（ColorOS 实测清单）

**A 档（低风险，可同法处理）**

| 包名                         | 角色             | 备注                                                                          |
| -------------------------- | -------------- | --------------------------------------------------------------------------- |
| `com.opos.ads`             | 系统广告总闸         | 大量 exported 组件；`disable-user` 同样失效，用 `suspend`                              |
| `com.oplus.statistics.rom` | **DCC 数据捐献中心** | 托管 `DonateContentProvider` 等 12 个 Provider，是短信捐献链的落库端；单记录，`disable-user` 可用 |
| `com.heytap.browser`       | 本文对象           | —                                                                           |

**B 档（中风险，有功能损失）**

| 包名                               | 损失                           |
| -------------------------------- | ---------------------------- |
| `com.heytap.quicksearchbox`      | 全局搜索；内含读短信/上传的 `SmsLoader` 链 |
| `com.coloros.sceneservice`       | 负一屏 / 场景推荐                   |
| `com.oplus.deepthinker`          | 应用预加载（可能影响冷启动速度）             |
| `com.coloros.remoteguardservice` | 家人守护                         |

**不要动（C 档）**：`com.coloros.phonemanager`、`com.oplus.dmp`、`com.oplus.ota`、`com.coloros.findmyphone`、`com.oplus.beaconlink`、`com.oplus.encryption`、`com.coloros.backuprestore`。

### 5.5 停止条件（命中任一条就别做）

1. `cmd role get-role-holders android.app.role.BROWSER` 返回的就是它 → 先装替代浏览器并设为默认
2. 手机上没有 Chrome 或任何第三方浏览器 → 会直接失去上网能力
3. Android < 7.0 → 无 `pm suspend`，只能 `disable-user`（且可能因双记录失败）
4. 用户依赖它的下载器、离线缓存、书签同步
5. 设备受 MDM/EMM 管理 → 策略可能把 suspend 回滚，并触发合规告警
6. 它是某个**系统入口的硬编码依赖**（无法静态确认，只能跑 §7 的行为回归来收口）

---

## 6. 可复现流程

> 约定：`PKG=com.heytap.browser`，`DEV=<serial>`。

### Step 0 — 连接

```bash
adb devices                      # 确认 DEV 在线
adb -s DEV shell getprop ro.build.version.release
```

### Step 1 — 只读侦察（务必先做）

```bash
# 身份与时间线
adb -s DEV shell dumpsys package $PKG | grep -E "versionName|firstInstallTime|lastUpdateTime|installerPackageName|pkgFlags|User 0:"

# 当前进程与内存
adb -s DEV shell ps -A | grep -i heytap.browser

# 是否默认浏览器（关键：若不是，禁用不影响日常开链接）
adb -s DEV shell cmd role get-role-holders android.app.role.BROWSER

# 高危权限
adb -s DEV shell dumpsys package $PKG | grep -E "MANAGE_EXTERNAL_STORAGE|QUERY_ALL_PACKAGES|SYSTEM_ALERT_WINDOW|PACKAGE_USAGE_STATS|REQUEST_INSTALL_PACKAGES"

# 导出 APK 做静态取证
adb -s DEV pull "$(adb -s DEV shell pm path $PKG | tr -d '\r' | sed 's/package://')" browser.apk
sha256sum browser.apk
```

### Step 2 — 影响面核查

```bash
# 它注册了哪些 ContentProvider（判断是否被别的进程当数据中心用）
adb -s DEV shell dumpsys activity providers | grep "com.heytap.browser/"

# 谁在 bind 它的服务
adb -s DEV shell dumpsys activity services | grep -A3 "ServiceRecord.*com.heytap.browser/"

# 默认浏览器解析是否指向别处（应为 com.android.chrome）
adb -s DEV shell cmd package resolve-activity --brief -a android.intent.action.VIEW -d http://a.com
```

判定门槛：`cmd role get-role-holders android.app.role.BROWSER` 若返回它自己，**停下**（见 §5.5）。

### Step 3 — 执行

```bash
for u in 0 10; do adb -s DEV shell pm suspend --user $u $PKG; done
adb -s DEV shell am force-stop $PKG
```

若 `disable-user` 返回 `new state: default`，不要再重试，直接走 `suspend`（原因见 §4.1）。

### Step 4 — 验证

```bash
# 状态：应见 suspended=true stopped=true
adb -s DEV shell dumpsys package $PKG | grep -E "^\s+User (0|10):|versionName"

# 进程：应为空
adb -s DEV shell ps -A | grep -c heytap.browser

# 功能回归
adb -s DEV shell cmd package resolve-activity --brief -a android.intent.action.VIEW -d http://a.com   # → com.android.chrome
adb -s DEV shell cmd package resolve-activity --brief -a android.intent.action.DIAL                  # → com.android.contacts/.DialtactsActivityAlias
adb -s DEV shell cmd package resolve-activity --brief -a android.intent.action.SENDTO -d sms:10086    # → com.android.mms/.ui.conversation.LaunchConversationActivity
adb -s DEV shell cmd package resolve-activity --brief -a android.settings.SETTINGS                   # → OplusSettingsHomepageActivity
adb -s DEV shell ping -c 2 223.5.5.5
```

### Step 5 — 断掉后台复活通道（可选）

```bash
adb -s DEV shell cmd deviceidle whitelist -com.heytap.browser
adb -s DEV shell am set-standby-bucket com.heytap.browser restricted   # 注意：ColorOS 上常被回成 5(EXEMPTED)，不保证生效
```

### Step 6 — 记录

把 Step 1/2/4 的原始输出整段落盘（路径自定），作为日后比对基线。

---

## 7. 利害分析

### 优势（有证据）

| 项             | 说明                                                     |
| ------------- | ------------------------------------------------------ |
| 去掉一整套广告投放链    | 穿山甲 3,549 + pangle 166 + 信息流 19,650 + 付费墙 + 11 个广告域名   |
| 去掉 3 个免鉴权保活入口 | `keep.alive` / `window.server` / `window.server.light` |
| 去掉高权限面        | 存储全盘、全包可见、悬浮窗、使用统计、安装包请求                               |
| 释放常驻内存        | 实测 644+226+228 ≈ 1.1 GB RSS                            |
| 切断被动拉起路径      | 实证存在"从设置 task 拉起"的行为                                   |
| 可逆            | 一条 `unsuspend` 回到原状，数据不丢                               |

### 代价

| 项                                                 | 严重度           |
| ------------------------------------------------- | ------------- |
| 桌面出现灰色图标（系统不隐藏 suspended 应用）                      | 观感问题          |
| 失去一个浏览器                                           | 低——默认是 Chrome |
| 手机搬家/云备份中"浏览器数据"一项失效                              | 低，仅影响它自己的书签历史 |
| `BrowserOpushService` 被 `com.heytap.mcs` 绑定的推送链断开 | 只影响浏览器推送      |

### 未验证的残余风险（**不要当作"没有"**）

1. 快应用（hapjs）若经 `ChannelService` 跳转，个别快应用调起可能异常。
2. 系统内是否存在硬编码 `com.heytap.browser` 的入口（如某处"用户协议"）。
   - 尝试用全量 `dumpsys package` 交叉引用归因失败：825 处命中几乎全分布在 Activity Resolver 表，无法确定宿主包。
   - 因此这条**既不能证明存在，也不能证明不存在**。
3. 该机 `dumpsys usagestats` 返回空，**无法证明"用户从不使用它"**。这是本次决策最大的证据缺口。

### 建议的收口验证（约两分钟，实际点击）

1. 设置 → 关于手机 → 法律信息 / 用户协议
2. 语音助手："打开浏览器" / "搜索 xx"
3. 相册 → 分享 → "用浏览器打开"
4. 手机搬家 / 云备份页面是否有报错
5. 微信/QQ/短信里点一条外链

同时开 `adb logcat | grep -E "ActivityNotFoundException|com.heytap.browser"` 观察是否有人调用它。

---

## 8. 回滚

```bash
adb -s DEV shell pm unsuspend --user 0  com.heytap.browser
adb -s DEV shell pm unsuspend --user 10 com.heytap.browser
# 若曾移出省电白名单，需要时恢复：
adb -s DEV shell cmd deviceidle whitelist +com.heytap.browser
```

回滚后图标恢复、可直接启动、书签历史完好（数据未动）。

---

## 9. 关于"禁止再被更新"

已确认与已排除：

| 通道                                                                 | 状态                                                                                                                     |
| ------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------- |
| `com.oplus.sau`（系统应用更新器）                                           | 已禁用                                                                                                                    |
| `com.oplus.ota` / `com.oplus.cota` / `com.oplus.customize.coreapp` | 已禁用                                                                                                                    |
| 应用商店 `com.heytap.market`                                           | **仍是唯一活的更新通道**                                                                                                         |
| 收回市场 `INSTALL_PACKAGES`                                            | 失败：`revokeRuntimePermission: Neither user 2000 nor current process has android.permission.REVOKE_RUNTIME_PERMISSIONS.` |
| AppOps 阻断                                                          | 不可行，市场持 `INSTALL_PACKAGES: granted=true`（系统级静默安装）                                                                      |
| restricted 待机桶                                                     | 不稳定，ColorOS 回写为 `5`(EXEMPTED)                                                                                          |

**安全网**：`suspended` 是 per-user 持久状态，`pm install -r` 不会清除它——即使被更新，新版本依然无法运行。版本号可能变化，行为不会恢复。

若要彻底断链，可选（代价：商店不可用）：

```bash
adb -s DEV shell pm suspend --user 0 com.heytap.market
adb -s DEV shell pm suspend --user 10 com.heytap.market   # 回滚 pm unsuspend
```

---

## 10. 原始证据文件

本文所有数值来自下列实测记录的当场输出（文件名已脱敏）：

| 记录                     | 内容                  |
| ---------------------- | ------------------- |
| `<LOG-1>`              | 执行前后状态、进程、禁用列表、功能回归 |
| `<LOG-2>`              | 市场权限收回尝试与报错         |
| `<LOG-3>`              | 白名单移除、待机桶值、最终锚点     |
| `<SAMPLE-APK-BROWSER>` | 约 219 MB 静态样本       |
| `<SAMPLE-APK-MARKET>`  | 约 94 MB（应用商店样本）     |

---

## 11. 判定口径（供日后复查）

一条命令，两个数：

```bash
adb -s DEV shell dumpsys package com.heytap.browser | grep -E "versionName|User 0:"
```

- 基线：`<原版本号>` / `suspended=true stopped=true enabled=0`
- **`suspended=false` → 被解除挂起，需重做 Step 3**
- **`versionName` 变化 → 商店仍在更新（此时行为仍被 suspend 拦住，但应处理商店）**
- 进程数 > 0 → 异常，立即 `am force-stop` 并复查

---

## 12. 方法论备注（对复现者）

1. **先证明它不是默认浏览器**，否则禁用等于砍掉上网能力。
2. `disable-user` 在 `UPDATED_SYSTEM_APP` 上可能静默失败并回滚成 `default`；**看到 `default` 就是失败**，不要当成成功。
3. `suspend` 与 `disable` 的区别：前者保留图标（灰色）与数据，后者隐藏且更"永久"；在无法 disable 的场景下 suspend 是等价手段。
4. 静态取证（DEX 计数 + 域名）只能证明"具备能力"，**不能证明"正在调用"**；判断运行时行为必须看 `dumpsys activity` / `logcat` / 流量。
5. 判断"能不能安全禁"，看三点：是否默认应用、是否被别的进程绑定、是否有系统入口硬编码。前两点可查，第三点往往查不清——需要行为回归兜底。
6. 跨品牌复用时，只改 `PKG` 与"默认浏览器"判定两处，其余步骤与判定口径不变。

---

##

|     |     |
| --- | --- |
|     |     |
|     |     |
|     |     |
|     |     |
|     |     |
|     |     |
|     |     |
|     |     |
