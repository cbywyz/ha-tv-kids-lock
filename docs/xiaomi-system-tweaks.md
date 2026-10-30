# 小米电视系统级折腾实录（去广告 / 换极简桌面 / 修信号源键）

> 本章是**电视本体**的优化记录（不是 HA 侧），与踩坑 16 / 17 / 18 / 19 互为印证。所有操作走 ADB，全部可逆。
> 正文教程在 [../README.md](../README.md)。

## A.1 设备与结论速览

| 项 | 值 |
|---|---|
| 电视 | 小米电视 MiTV-ASTP0（mulan 固件，Amlogic 平台） |
| 网络 | **有线** `eth0`，IP `192.168.8.103/24`（旁路由下的二级网段；主路由有该网段路由，局域网内可直连，不需要 zerotier） |
| 工具 | [Gathin23/cleanyourtv](https://github.com/Gathin23/cleanyourtv)（MIT，纯 Python ADB 客户端，免装 adb） |
| 去广告 | 26 个包 `pm disable-user` 禁用（可逆，非卸载） |
| 桌面 | **最终用 [Aerial](https://github.com/pengchujin/aerial-tv)（`app.orchard.tv`）**：靠无障碍守护 `HomeGuardService` 接管 Home 键；系统默认 Home 仍是旧桌面（冷开机显示小米原桌面）。早期试装的极简桌面 `com.gathin.tvhome` 已卸载（见 A.6） |
| 信号源键 | 仅 gathin 方案需要自修（见 A.5）；Aerial 方案不涉及此问题 |

## A.2 ADB 连接（mulan 固件特性，重要）

1. 电视：设置里打开「USB 调试」→ `adb connect 192.168.8.103:5555`；
2. **重启后网络 ADB 实测会自起**：`persist.adb.tcp.port=5555` 已固化（重启后仍在，2026-10-26 重启验证 ~10s 恢复连接）；若某次连不上再手动开「USB/网络调试」。旧笔记「重启必关 ADB」是基于未固化 persist 属性的情况，已不再成立。
3. 不想装 adb：用 cleanyourtv 的 `tools/tv.py`（纯 Python 实现 ADB 协议）；
4. **文件传输的坑**：这台电视 `adb pull` 报 `open failed`、往 `/sdcard` 写进去是 0 字节 → 实用解法是 `cat` + base64 分块读回。注意每个 64KB 块 base64 后自带 `==` 补位，**必须按行独立解码再拼字节**，直接拼字符串解码会错位。

## A.3 去广告（pm disable-user）

- **保留**：小爱语音助手、投屏（miplay / airkan / wfdsink / mi_connect）、米家、桌面、屏保、锁屏、输入法、`com.mitv.videoplayer`（本地/USB 播放引擎）、`com.mitv.tvplayer`（直播/信号源）等；
- **小米视频（`com.mitv.mivideoplayer`）已改为禁用**（2026-10-26）：用户不再使用（收费贵 + 应用内贴片广告），它是唯一残留广告源；禁用不影响投屏与本地 USB 播放（走 `com.mitv.videoplayer`）。如需恢复：`pm enable com.mitv.mivideoplayer`。
- **禁用**：统计 / analytics、商城 / 支付、tvpush 推送、天气、日历、卡拉OK、小米音乐、相册、云控、升级、短信推送等共 26 个；
- 脚本自带 **Home 归属保护**（候选名单里即使出现 `com.mitv.tvhome` 也不会真禁）；
- 随时可还原：`tools/restore.sh` + 定制清单，恢复记录在 `~/.cleanyourtv/recovery-<IP>-<id>.json`；
- ⚠️ 最大代价见[踩坑 18](troubleshooting.md)：**别手动禁旧桌面**。

## A.4 换极简桌面（可选，附云端构建）

- 上游 launcher（`com.gathin.tvhome`）不提供预编译 APK，但仓库自带完整 gradle 工程 → **GitHub Actions 云端构建**（本机没有 JDK / Android SDK 时最省事）：建私库推 workflow，约 2 分钟出包；
- 新版 runner 的坑：`sdkmanager` 不在 PATH，`android-actions/setup-android@v3` 自身有 bug 会挂 → 直接用 runner 自带 `/usr/local/lib/android/sdk`；
- **CI 每次构建会生成新的 debug 密钥** → 覆盖安装必被拒（签名不匹配），必须先 `pm uninstall` 再装；
- 设默认 Home 的真相：`pm set-home-activity` 在这固件**写了但不生效**（旧桌面带 `HOME_STATE` 自定义权限压场，`am start` HOME 意图解析仍走旧桌面；`resolve-activity` 在这固件上也不读首选登记，别被它误导）；
- 最终采用的行为（零风险，推荐）：**按 Home → 旧桌面**；新桌面从应用列表打开，进去后按 Home 随时回去。

> ⚠️ **gathin 已被 Aerial 取代并卸载**（2026-10-26）：gathin 作为默认桌面在 mulan 上不生效（`set-home-activity` 写了不生效），且信号源键得自己打补丁；最终改投 **Aerial**（体验更完整、Home 接管更稳），详见 [A.6](#a6-最终桌面aerial取代-gathin)。

## A.5 信号源键修复（本次核心）

**根因**：Input 键（`KEYCODE_TV_INPUT=178`）归**前台应用**管。旧桌面被禁或不在前台 → 按键无人处理 → 失灵。

**无效方案（省你时间）**：无障碍服务按键过滤（`flagRequestFilterKeyEvents` + `canRequestFilterKeyEvents`）——服务绑定成功、能力位齐全，但这固件**就是不过滤**，按键直接透传给窗口。

**有效方案**（Activity 级拦截，旧桌面同款思路），在 `MainActivity.dispatchKeyEvent` 顶部：

```java
if (event.getAction() == KeyEvent.ACTION_UP && !event.isCanceled()
        && event.getKeyCode() == KeyEvent.KEYCODE_TV_INPUT) {
    if (openSourcePicker()) return true;
}
```

`openSourcePicker()` 先试 Google TV 的隐式 `com.android.tv.action.VIEW_INPUTS`（无接收方会抛 `ActivityNotFoundException`），再回退显式组件：

```java
startActivity(new Intent()
        .setComponent(new ComponentName("com.droidlogic.tvinput",
                "com.droidlogic.tvinput.shortcut.ShortCutActivity"))
        .addFlags(Intent.FLAG_ACTIVITY_NEW_TASK));
```

**验证方法**（不用盯屏幕）：

```bash
input keyevent 178          # KEYCODE_TV_INPUT
dumpsys window windows | grep mCurrentFocus
# 期望：mCurrentFocus=Window{... com.droidlogic.tvinput/...ShortCutActivity}
```

> `ShortCutActivity` 是半透明弹出面板（`TRANSIT_TRANSLUCENT_ACTIVITY_OPEN`，视频叠加层渲染），**screencap 截不到它**——判断它是否弹出只能看 `dumpsys window` 的窗口焦点。

## A.6 最终桌面：Aerial（取代 gathin）

> 2026-10-26 收尾：`com.gathin.tvhome` 已 `pm uninstall` 卸载，Aerial 成为唯一第三方桌面。

**为什么换**：gathin 在 mulan 上 `set-home-activity` 写了不生效（默认 Home 仍是旧桌面），且信号源键得自己打补丁；Aerial（`app.orchard.tv`）体验更完整（tvOS 风格、自带屏保 Dream、Home 接管更稳）。

**Aerial 怎么「当」Home（与 gathin 思路不同）**：
- Aerial 注册无障碍服务 `app.orchard.tv/app.orchard.tv.guard.HomeGuardService`，**接管 Home 键**——实体遥控按 Home → 系统把 Home 意图交给 Aerial，而不是旧桌面；
- 该无障碍服务在系统里**开机自启**（重启后 `settings get secure enabled_accessibility_services` 仍含它，进程自动拉起），所以「按 Home → Aerial」在重启后照样成立；
- 同时 Aerial 发 `keep_adb` 广播保持网络 ADB 常开。

**两个必须知道的边界（mulan 固件特性）**：
1. **冷开机先闪一下小米原桌面，随后 Aerial 自动接管**：系统默认 Home 仍是 `com.mitv.tvhome`（`resolve-activity --home` 解析即它，`cmd role` 在这固件没有），所以开机瞬间看到的是原桌面；但 Aerial 的无障碍守护 `HomeGuardService` 开机自启后，会**自动把 Aerial 拉到前台**（无需按 Home），实际效果是「开机约 1~2 秒后自动进入 Aerial」，只是开头有一帧小米桌面的闪现。想要「冷开机零闪现、直接 Aerial」在本固件做不到（默认 Home 改不了）。
2. **模拟 `input keyevent HOME` 验收不可信**：mulan 上这条命令会直接落到系统默认 Home（原桌面），绕过了无障碍对 Home 键的接管，所以 dumpsys 看到的是 `com.mitv.tvhome` 而非 Aerial——**这是假阴性，必须用实体遥控器按 Home 验收**（用户 2026-10-26 实测「按 Home 回 Aerial」通过）。

**补充（2026-10-26 晚）：gathin v3.0.0 试装回归（与 Aerial 共存）**
- cleanyourtv 上游发了 **v3.0.0**（2026-10-25）：20 个官方插件（含广告拦截/信号源）、Launcher 3.0、适配 300 型号。官方 release 依旧**不含 APK**，需自行构建；
- 在构建库 `cleanyourtv-tvhome` 开 `v3-build` 分支：上游 v3.0.0 tag + 我们的 CI workflow（源码已挪到 `launcher/` 子目录，workflow 加 `defaults.run.working-directory: launcher`）；
- **CI 踩坑**：① GitHub 账号级 **artifact 存储配额已满**（upload-artifact 报 `Artifact storage quota has been hit`，删光本仓库旧 artifact 也不够）→ 改用 **Release 资产**发布（`gh release create` 不认 `--clobber`，`gh release upload` 才认；Release 资产不占 Actions 配额）；② 每次 run 都是全新 runner，**debug 签名密钥每次重生成 → 覆盖安装报 `INSTALL_FAILED_UPDATE_INCOMPATIBLE`**，需先 `pm uninstall` 再装；
- 装回电视：`versionName=3.0 / versionCode=9`，仅注册 LEANBACK_LAUNCHER，**系统默认 Home 仍是小米原桌面、Aerial 守护未受影响**——gathin v3.0 作为普通应用从 Aerial 应用列表打开，两桌面共存对比。

**信号源键：v2.7 的老补丁是错的，v3.0.0 要用上游机制（2026-10-26 实测）**
- 老补丁（v2.7 时代）在 `dispatchKeyEvent` 里抢先 `startActivity(VIEW_INPUTS)`，失败后退到**写死 droidlogic 的 `com.droidlogic.tvinput/.shortcut.ShortCutActivity`**。mulan 上该面板弹出后自身报「**该电视不支持该功能**」——补丁把按键引上了死路，还抢在上游逻辑之前执行；
- **上游 v3.0.0 早就适配了这台机器**：`examples/xiaomi-mitv-astp0/` 的 README 明写 "inputs switch only with Xiaomi's `EXTSRC_PLAY`"；
- 正解（已落地并实测通过）：manifest 允许的情况下**仍要在 `dispatchKeyEvent` 拦 TV_INPUT**（InputKeyService 靠无障碍过滤，mulan 不支持，不拦就丢键），但拦到后**调上游自己的 `showInputs()`**，不再跳 droidlogic；`config.xml` 里配
  - `input_action` = `com.xiaomi.mitv.tvplayer.EXTSRC_PLAY`
  - `input_sources` = `HW5=23 / HW6=24 / HW7=25 / HW8=- / HW2=2 / HW3=- / HW4=-`
- 编号来自 `dumpsys tv_input`（HDMI1=HW5）；实测 `EXTSRC_PLAY` + int extra `input=23` → logcat `setCurrentSource 23`、`Hdmi1InputService/HW5`，画面真切；`input=24` 同理切 HDMI2。这组编号与上游 `examples/redmi-mitv-mftp0` **完全一致**（MStar 编号，Amlogic 机器上照样有效）；
- 菜单/切源后无法用 `am start` 冒烟验证时注意：`ExternalSourceActivity` 是视频层，`screencap` 截出来是 0 字节，只能靠 logcat 判；
- **XML 注释坑**：`values/config.xml` 注释里写了 `--ei input 23`，连续两个 `-` 在 XML 注释里非法，`mergeReleaseResources` 直接失败（`The string "--" is not permitted within comments`）——文档里引命令行参数时别写进 XML 注释。

**重启验证（2026-10-26 实测）**：
- 卸载 gathin 后 `adb reboot`；
- 约 10s 后 ADB 自动恢复（`persist.adb.tcp.port=5555` 固化生效）；
- 重启后：`com.gathin.tvhome` 消失、`app.orchard.tv` 仍在、`HomeGuardService` 无障碍仍启用、Aerial 进程自动拉起；
- 实体 Home → Aerial 行为不变（无障碍守护开机自启）。

## A.7 收尾：去广告收口 + 免 ROOT 兜底拦截（已废弃）

> ⛔ **已废弃（2026-10-27 定案，别照抄上线）**：这套最终没保留。`dnsfilter.android` 已从电视卸载，对应的 HA 看门狗自动化（unique_id `tv_adblock_watchdog_20261026`）也已删除、孤儿实体从注册表清掉。放弃原因：电视的网关和 DNS 都指向 `192.168.8.1`（主路由），旁路由上又没有 adblock 插件，想让流量改走旁路由就得改电视静态 IP——而这只能拿遥控器一项项手点，收益不值这个折腾。**下面保留的是当时的完整实现过程，供以后翻查。**

> 2026-10-26 追加：去广告清单已到「最优」状态，再叠加一个免 ROOT 的 DNS 层拦截器兜底。

**1. 禁用小米自带屏保 `com.mitv.screensaver`**
- `pm disable-user com.mitv.screensaver`；禁用总数到 **28**。
- 原因：Aerial 的 `AerialDream` 屏保已接管（`settings get secure screensaver_components = app.orchard.tv/...AerialDream`），小米原屏保无存在意义，且它本就是广告位之一。两者不冲突。

**2. 小米视频 App 也禁了**（见 A.3 同步说明）：`com.mitv.mivideoplayer` 已 `pm disable-user`，掐掉唯一残留的「应用内贴片广告」源（你不用小米视频，收费又贵）。

**3. 装免 ROOT 广告拦截器 [personalDNSfilter](https://github.com/personalDNSfilter/personalDNSfilter)（dnsfilter.android）**
- 来源：F-Droid 官方签名包 `dnsfilter.android_1506103.apk`（SHA256 `3ddd0db322d1ca5fe950ea29dc1788af6e6581bb5f34694085fb4aa2e77b68ad`，约 626 KiB）。
- 原理：**本地 VPN 模式**拦截 DNS 请求，把广告/追踪域名解析到回环地址；免 ROOT、开源、过滤全本地（不联网上传），预置 yoyo / adaway / Steven Black 等列表。对电视上残留的「应用内广告」再挡一层。
- 安装：`adb install -r C:/.../personaldnsfilter.apk` → 成功（`package:dnsfilter.android`）。
- **首次授权**：启动后系统会弹 `com.android.vpndialogs.ConfirmDialog`（「personalDNSfilter 想要设置 VPN 连接」），**必须用遥控器点「确定」**——这是 Android 安全机制，adb 无法代点。点完即开始拦截。
- 建议进 App 开「Autostart / 开机自启」，使拦截在重启后自动恢复。
- 注意：DNS 层只能拦「域名解析型」广告，对把广告域名写死 IP、或走自家 CDN 的应用内广告无效（那种本来就少）；它和已禁的 28 个系统包是**互补**关系，不是重复。

**当时的总状态（2026-10-26，⚠️ 已过期，见本节顶部废弃说明）**：系统层（推送/商城/支付/统计/小米桌面广告位/小米视频）已 28 包禁用全灭；Aerial 接管 Home 看不到原桌面广告；personalDNSfilter 再兜底 DNS 层应用内广告。**全程免 ROOT、全可逆**（`pm enable` 任一项即可恢复）。系统包禁用这部分**至今仍在生效**，废弃的只是 personalDNSfilter 那一层。

## A.8 重启后拦截自动恢复：交给 Home Assistant 看门狗（已废弃）

> ⛔ **同样已废弃（2026-10-27）**：看门狗自动化 `automation.dian_shi_yan_gao_lan_jie_shou_hu_personaldnsfilter_zi_hui_fu` 已删除，电视上也早已没有 `dnsfilter.android`，所以它一直显示 `unavailable`。本节仅作为「用 HA 盯电视端 ADB 状态」的实现范式保留——**里面的 ADB 集成接入方式、`adb_response` 回显拿法、`automation.reload` 热重载这些经验仍然有效**（信号源锁、App 锁都在用），只是不要再去拉起 dnsfilter。

> 痛点：personalDNSfilter 的 **VPN 在电视重启后不会自起**——小米系统级「自启动白名单」会杀它的 `BootUpReceiver`（监听 `BOOT_COMPLETED`），加上 Android 9 后台启动限制、本固件又无 `cmd vpn` 的 always-on 能力。结果：每次重启，拦截静默失效，要人手动进 App 点一下。
> 试过两条路，最终选 **HA**：

**① 电脑看门狗（否决）**：在 Windows 控制机放脚本每 5 分钟拉起过滤。用户明确「看门狗不能装电脑上」，弃。

**② Home Assistant 看门狗（采用）**：HA 常驻、且本就接管这台电视，最自然。

- **前置：重新加回 Android TV(ADB) 集成**。之前因为「重启 ADB 自动关闭」移除了 ADB 遥控器；现在 `persist.adb.tcp.port=5555` 已固化、ADB 重启自起，顾虑消除，遂在 HA 加回 Android TV 集成（host `192.168.8.103:5555`，device_class `androidtv`）。生成实体：
  - `media_player.android_tv_192_168_8_103`（friendly=小米电视）—— **所有 adb 命令都发给它**（看门狗、信号源锁都一样）；
  - `remote.android_tv_192_168_8_103` —— 当年移除的 ADB 遥控回来了，但它的状态常是 `unknown`，**不能用它发 adb 命令**（会静默不执行，见[踩坑 21](troubleshooting.md)）。
  - ⚠️ 首次连接电视会弹「允许 USB 调试吗」，遥控器点「确定」一次性授权（新主机密钥，和电脑 adb 不同）。
- **命令回显拿法（2026-10-27 更正）**：`androidtv.adb_command` 不接受 `response: true`（HTTP 400），但命令输出会写进目标 `media_player` 的 **`adb_response` 属性**——用它就能在 HA 侧读回电视端结果，不必再写"自包含、自己判断"的黑盒命令。下面这条仍保留为"电视端自决"写法，但排障时优先看 `adb_response`：
  ```
  ip addr show tun0 | grep -q inet && echo TUN0_OK ||
  (am start -n dnsfilter.android/.DNSProxyActivity && sleep 4 && input keyevent KEYCODE_HOME)
  ```
  tun0 在 → 什么都不做（**平时零干扰**，连待机唤醒都不影响）；tun0 不在 → 拉起过滤 App、等 4 秒建 VPN、再 `HOME` 退回 Aerial。
- **触发**：`time_pattern` 每 5 分钟兜底巡检 + 电视进入 `playing`/`idle`（一开机）立即查一次；`mode: single`。
- **落地方式（已更正）**：本机 HA 实为 **YAML 模式**——自动化存 `/config/automations.yaml`（bind 到主路由 `/mnt/sata4-4/homeassistant/automations.yaml`，`.storage` 里并无 automations 文件；之前误判成 UI 模式）。经主路由跳板（paramiko + SFTP）把看门狗自动化**直接追加**进 `automations.yaml`（原文件先 `cp` 备份）。**热重载用 `automation.reload`**（`homeassistant.reload_automations` 在 2026.x 已不存在，调它会返回 `service_not_found`，别再用它，也不必重启核心）。加载后实体 `automation.dian_shi_yan_gao_lan_jie_shou_hu_personaldnsfilter_zi_hui_fu`（中文 alias 自动转拼音 slug）state=**on**。
- **实测验证**：本机 adb 强制 `am force-stop dnsfilter.android` 使 `tun0` DOWN；再让 HA 跑该命令 → `tun0` 自动回到 `UP`（`inet 10.0.2.15/24`），`doubleclick.net` 重新解析到 `127.0.0.1`。端到端通过。

**一句话（⚠️ 该承诺已作废，2026-10-27 方案终止）**：~~电视重启后，HA 会在 5 分钟内（或你一开机）自动把广告拦截拉起来~~ —— 现在不装了，也就不需要了。
