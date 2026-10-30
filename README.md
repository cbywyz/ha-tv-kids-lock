# 电视「家长管控」教程：音量上限锁 + 信号源锁 + 儿童观看定时锁 + 每日配额 / 分时段模式 / 倒计时暂停（Home Assistant）

> 不改电视系统、不装电视端 App，管控放在电视**外面**——孩子拿遥控器无解。
>
> 方案与品牌无关：小米、索尼、TCL、Apple TV、各类电视盒子……只要电视能接入 Home Assistant，这套教程就能直接套用。**本文以小米电视为例**，其它品牌只是接入方式不同（见「三、准备工作」对照表），自动化部分完全通用。

本仓库四个文件：

| 文件 | 作用 |
|------|------|
| `README.md` | 教程正文（你正在看） |
| `tv_lock_automations.yaml` | 23 条自动化合集，可直接粘贴 |
| `helpers-example.yaml` | 辅助元素（helper）定义 |
| `scripts-example.yaml` | 5 个家长一键情景脚本 |

基础篇（音量锁 / 信号源锁 / 定时锁）看第三～五章；进阶篇（每日配额 / 分时段 / 续杯 / 周报）看[第六章](#六进阶玩法每日配额--分时段--续杯--周报)。**只想快速上手，只做基础篇就够。**

## 相关仓库（同一系列教程）

- [phicomm-t1-android-tv](https://github.com/cbywyz/phicomm-t1-android-tv) —— 斐讯 T1 盒子接入 HA：adb 遥控面板 / 折叠数字键盘 / **中文输入** / 开机优化（同一套 HA 底座，接进来直接套本仓库管控）
- [ha-midea-hualing-ac](https://github.com/cbywyz/ha-midea-hualing-ac) —— 美的/华凌空调接入 HA
- [ha-hualing-fan-broadlink](https://github.com/cbywyz/ha-hualing-fan-broadlink) —— 华凌风扇红外遥控接入 HA
- [gree-yapqf-broadlink-smartir](https://github.com/cbywyz/gree-yapqf-broadlink-smartir) —— 格力空调 Broadlink+SmartIR 接入 HA
- [phicomm-aircat-m1](https://github.com/cbywyz/phicomm-aircat-m1) —— 斐讯悟空 M1 空气检测仪复活记
- [jdcloud-nezha-ax3000-rescue](https://github.com/cbywyz/jdcloud-nezha-ax3000-rescue) —— 京东云哪吒 AX3000 刷机变砖与救砖实录

---

## 一、为什么做这个

几乎所有品牌的「家长管控」只有一个电视端「儿童模式」：设密码、圈内容。但这有个绕不开的难点——**密码长在电视上，孩子盯着看两遍就记住了**；而且没有「锁信号源 / 限时长 / 限音量」这种精细开关。

思路反过来：**把管控从电视里搬到电视外**。电视接入 HA 后就是一个 `media_player` 实体，音量、信号源、开关机都能远程读写，于是能做到电视端做不到的粒度：

- **音量上限锁**：可随意调小，超过上限自动拉回；
- **信号源锁**：锁定某输入（如机顶盒 HDMI 1），被切走秒切回；
- **儿童定时锁**：设定观看时长，到点自动上锁；再设锁定时长，到点自动解锁（或一直锁到家长手动解）。

孩子在电视前无论按什么，都拗不过电视背后的自动化——**没有密码可破，因为根本就没有密码**。

## 二、方案架构

```
┌──────────────┐  集成通道(云/局域网) ┌─────────────────┐
│  电视(被管)   │ ◄─────────────────► │  Home Assistant  │
│              │    media_player     │  (管控大脑)       │
└──────────────┘                     └───────┬─────────┘
                                          │ 自动化
                            ┌─────────────┼─────────────┐
                            ▼             ▼             ▼
                        音量上限锁      信号源锁      儿童定时锁
```

| 管控 | 手段 | 关键点 |
|------|------|--------|
| 音量上限 | 监听 `volume_level`，超限立即拉回 | 0.5% 死区防自触发 |
| 信号源 | 锁定即**无条件强切** + ADB 10s 校验兜底 | 不信任电视上报的 `source`（会停更） |
| 定时锁 | HA 原生 `timer` 倒计时到点开锁 | 手动锁永不自动解除 |

## 三、准备工作

1. **Home Assistant**：任意部署方式均可（我跑在软路由 Docker 里）。
2. **电视接入 HA**（各品牌对照）：

   | 设备类型 | 接入方式 |
   |----------|----------|
   | 小米电视（示例） | Xiaomi Miot 集成（走米家云，偶发丢指令需补发） |
   | 索尼 / TCL / 雷鸟等 Android TV、安卓盒子 | HA 内置 **Android TV** 集成（局域网直连，秒达） |
   | Apple TV | `apple_tv` 集成 |
   | Fire TV | ADB 方式 |
   | 部分 LG / 三星 | webOS / 各自集成（先看 `source_list`） |

   接入后确认实体有 `volume_level` 和 `source` 能力即可整套部署。**小米电视建议加接 Android TV 集成做 ADB 辅助通道**——音量双通道兜底、信号源锁的秒级校验都靠它。

   > 想折腾电视**本体**（去广告 / 换极简桌面 / 修信号源键），见 [docs/xiaomi-system-tweaks.md](docs/xiaomi-system-tweaks.md)。

   > 下文所有 `media_player.living_room_tv` 请换成你自己的电视实体 ID；信号源列表、音量范围同理按你电视实际属性调整。

## 四、第一步：创建辅助元素（Helpers）

整套共用 **8 个辅助元素**，直接加进 `configuration.yaml`（或 UI 逐个建）。完整定义见 `helpers-example.yaml`。

```yaml
input_boolean:
  tv_source_lock:                 # 电视信号源锁定
  dian_shi_yin_liang_suo_ding:    # 电视音量锁定
  ben_ci_wei_dao_ji_shi_suo_ding: # 本次为倒计时锁定（手动锁不标记）
input_select:
  tv_lock_source:                 # 电视锁定信号源（options 按你电视 source_list 填）
input_number:
  er_tong_guan_kan_shi_chang:     # 儿童观看时长（分钟），填几倒计时几分钟，0=取消
  suo_ding_shi_chang:             # 自动解除锁定时长（分钟），0=一直锁到手动解
  dian_shi_yin_liang:             # 电视音量（上限）
timer:
  pin_dao_suo_ding_dao_ji_shi:    # 频道锁定倒计时
  pin_dao_suo_ding_jie_chu:       # 频道锁定解除倒计时
```

## 五、第二步：基础三道锁

### 5.1 音量上限锁

打开「电视音量锁定」后，「电视音量」数值即**上限**：可随意调小，超过就拉回。只认 0.5% 以上超限，避免拉回动作自己触发自己（死循环）。

```yaml
- id: tv_volume_lock
  alias: "[电视] 音量锁定（上限模式，超了拉回）"
  mode: single
  trigger:
    - platform: state
      entity_id: media_player.living_room_tv
      attribute: volume_level
  condition:
    - condition: state
      entity_id: input_boolean.dian_shi_yin_liang_suo_ding
      state: "on"
    - condition: template
      value_template: >-
        {% set target = states('input_number.dian_shi_yin_liang') | float(0) / 100 %}
        {% set cur = state_attr(trigger.entity_id, 'volume_level') %}
        {{ cur is not none and cur - target > 0.005 }}
  action:
    - service: media_player.volume_set
      target: { entity_id: "{{ trigger.entity_id }}" }
      data:
        volume_level: "{{ states('input_number.dian_shi_yin_liang') | float(0) / 100 }}"
```

> 面板放个音量滑条、停手 2 秒下发，见 `tv_lock_automations.yaml` 的 `tv_apply_volume`。

### 5.2 信号源锁定

**锁定生效立即强切**：开关一开就**无条件**切到锁定源（不先判断 `cur != want`，因为云 `source` 属性会停更，见[踩坑 2](docs/troubleshooting.md)）。云端集成指令偶发丢失，5 秒后补发一次（`tv_lock_engage_switch`）。

**换台拦截**：锁定期间被切走自动切回，云通道降为每 5 分钟兜底（`tv_source_lock`）——**千万别每分钟发**，会把小米云通道打爆导致遥控器失灵（[踩坑 15](docs/troubleshooting.md)）。真正的秒级兜底交给下面的 ADB 通道。

**ADB 增强校验（推荐）**：云 `source` 属性有缓存、还会停更，孩子打开电视内 App 时它仍报旧 HDMI → 锁形同虚设（[踩坑 13](docs/troubleshooting.md)、[踩坑 20](docs/troubleshooting.md)）。修法：用 Android TV 集成每 10 秒读电视**本地真实状态**（`dumpsys` + `settings`），不对就本地切回，完全不经过云：

```yaml
- id: tv_source_lock_adb
  alias: "[电视] 信号源锁定 ADB 增强校验"
  mode: single
  max_exceeded: silent
  trigger:
    - platform: time_pattern
      seconds: "/10"
  condition:
    - condition: state
      entity_id: input_boolean.tv_source_lock
      state: "on"
    - condition: template
      value_template: "{{ states('media_player.living_room_tv_adb') in ['playing','paused','idle','on'] }}"
    - condition: template
      value_template: "{{ states('input_select.tv_lock_source') in ['HDMI 1','HDMI 2'] }}"
  action:
    - service: androidtv.adb_command
      continue_on_error: true
      target: { entity_id: media_player.living_room_tv_adb }
      data:
        command: >-
          T={{ {'HDMI 1': 23, 'HDMI 2': 24}.get(states('input_select.tv_lock_source'), 0) }};
          R=$(dumpsys activity activities | grep -m1 mResumedActivity);
          case "$R" in *ExternalSourceActivity*)
          C=$(settings get global mitv.tvplayer.hdmi.last.source | tr -d '\r');
          [ "$C" = "$T" ] && exit 0;; esac;
          am start -a com.xiaomi.mitv.tvplayer.EXTSRC_PLAY
          -n com.xiaomi.mitv.tvplayer/.ExternalSourceActivity --ei input $T
```

> 命令回显看目标 `media_player` 的 `adb_response` 属性；`entity_id` 必须写 `media_player.*`，写 `remote.*` 会静默不执行（[踩坑 21](docs/troubleshooting.md)）。一次只能一个 adb 客户端连电视（[踩坑 22](docs/troubleshooting.md)）。

**锁 App（只锁信号源锁不住内容 App）**：只锁信号源 = 假锁——孩子从桌面打开云视听/B站，`source` 不变就不触发；而且锁回的 HDMI 若无信号会弹回桌面。做法：锁开就 `pm disable-user` 掉娱乐类 App，锁关 `pm enable` 恢复（全程可逆、不碰系统分区、不需 root）：

```yaml
- id: tv_app_lock_adb
  alias: "[电视] 锁定期间禁用娱乐 App"
  mode: single
  max_exceeded: silent
  trigger:
    - platform: state
      entity_id: input_boolean.tv_source_lock
    - platform: time_pattern
      minutes: "/2"        # 兜底对账，命令漏执行 2 分钟内自愈
  action:
    - service: androidtv.adb_command
      continue_on_error: true
      target: { entity_id: media_player.living_room_tv_adb }
      data:
        command: >-
          {% set apps = 'com.xiaodianshi.tv.yst com.ktcp.video com.tencent.qqmusictv com.ixigua.android.tv.wasu com.gitvdemo.video com.dangbeimarket com.cibn.tv' %}
          {% if is_state('input_boolean.tv_source_lock','on') %}
          for p in {{ apps }}; do pm list packages -e | grep -qx "package:$p" && pm disable-user --user 0 $p; done; echo APPS-LOCKED
          {% else %}
          for p in {{ apps }}; do pm list packages -d | grep -qx "package:$p" && pm enable $p; done; echo APPS-UNLOCKED
          {% endif %}
```

> 白名单只动娱乐包，永远不碰桌面/输入法/systemui；先查存在再动手（幂等）。找包名用 `adb shell pm list packages -3 -e`。

### 5.3 儿童观看定时锁

给孩子填「儿童观看时长」= 30 → 30 分钟后自动上锁；「自动解除锁定时长」填 30 → 再过 30 分钟自动解锁（填 0 = 一直锁，家长手动解）。**手动开的锁永不被自动解除**（用一个内部标记区分）。

涉及 5 条自动化（`tv_lock_countdown_start` / `..._cancel` / `..._finish` / `tv_lock_release_finish` / `tv_lock_unlock_cleanup`），完整 YAML 见 `tv_lock_automations.yaml`，逻辑：

- 填时长 → 启动 `timer`；改数字重计、填 0 取消；
- 倒计时结束 → 打开信号源锁定，若解除时长 >0 同时启动解除 `timer`；
- 解除倒计时结束且确为倒计时锁 → 自动解锁；手动解锁清标记 + 取消解除计时。

### 5.4 倒计时可暂停（关机暂停、开机继续）

HA 的 `timer` 是墙钟，电视关机也照走——孩子关电视出去玩，到点照样上锁。用 `timer.pause` / `timer.start`（不带 duration = 从剩余继续）补上：电视关机满 10 秒暂停观看类倒计时，开机继续。**惩罚/冷却倒计时（解除计时）按墙钟走，不暂停**（否则关电视就能赖掉惩罚）。

逻辑见 `tv_lock_automations.yaml` 的 `tv_timer_pause_when_off` / `tv_timer_resume_when_on`。所有开关机判据走 **ADB 实体**而非云实体（云 `state` 也会停更，[踩坑 23](docs/troubleshooting.md)）。

## 六、进阶玩法（每日配额 / 分时段 / 续杯 / 周报）

> 完整自动化见 `tv_lock_automations.yaml` 的「进阶篇」段落；通知部分换成你自己的推送服务（示例用飞书）。

- **每日观看配额**：电视开着每分钟给 `input_number` +1（不用 `history_stats`——它没 reload 服务、必须重启 HA）。今日已看 ≥ 配额 → 上锁 + 推通知。每天 0:01 并入本周累计后清零。
- **分时段模式**：一个 `input_select` 收拢四档——常规 / **作业模式**（上锁且 3 秒内自动锁回、不给续杯）/ **睡眠模式**（一开机就关）/ **今晚自由**（一键全家豁免）。每晚 22:30 自动睡眠、早 7:00 回常规。
- **再看 5 分钟（续杯）**：解锁 → 5 分钟后自动重新锁，**每天最多 2 次**（用完拒绝）。作业模式不给续。
- **每周周报**：周一早 9 点推上周观看（本周累计 + 日均），推完清零。

> 别忘了给「到点自动解锁」加「当前不是作业模式 / 配额未用尽」条件，否则作业模式会被定时解锁悄悄破掉。

## 七、面板卡片建议

放一张 entities 卡，家长一个界面全搞定：

```yaml
type: entities
title: 📺 电视家长管控
entities:
  - entity: input_boolean.tv_source_lock
    name: 信号源锁定
  - entity: input_select.tv_lock_source
    name: 锁定信号源
  - entity: timer.pin_dao_suo_ding_dao_ji_shi
    name: 观看倒计时
  - entity: input_number.er_tong_guan_kan_shi_chang
    name: 儿童观看时长(分)
  - entity: timer.pin_dao_suo_ding_jie_chu
    name: 锁定剩余时间
  - entity: input_number.suo_ding_shi_chang
    name: 自动解除时长(分)
  - entity: input_boolean.dian_shi_yin_liang_suo_ding
    name: 音量锁定
  - entity: input_number.dian_shi_yin_liang
    name: 音量上限
```

进阶做完后加一排一键按钮（`看 30 分钟` / `作业模式` / `睡眠模式` / `今晚自由` / `再看 5 分钟`，对应 `scripts-example.yaml` 的 5 个脚本）。功能多了建议单独建个仪表盘挂侧栏。

> 小技巧：`timer.*` 实体直接放上卡片，前端每秒刷新自带倒计时跳动，比模板卡省事。给家人用的「❓ 使用说明」弹窗（Bubble Card）写法见 `tv_lock_automations.yaml` 顶部注释。

## 八、详细文档（避坑 & 电视本体折腾）

- 📌 [踩坑记录（23 条）](docs/troubleshooting.md) —— 云通道丢指令、`source` 缓存、0.5% 死区、Jinja `{{ set }}` 坑、ADB 单会话、`remote.*` 静默不执行、云 `state` 停更……排错前先翻。
- 🔧 [小米电视系统级折腾](docs/xiaomi-system-tweaks.md) —— 去广告（28 包禁用）、Aerial 极简桌面、信号源键修复、personalDNSfilter 兜底（已废弃说明保留）。全部走 ADB、可逆。

## 九、更新日志

- **2026-10-27** 修「电视关机后还在计时」（云 `state` 也停更，判据换 ADB 实体）；信号源锁 ADB 校验 30s→10s；删除广告拦截看门狗；周报「本周累计」含今日；禁用名单补 CIBN 酷喵（共 7 个娱乐包）；新增「锁 App」。
- **2026-10-26** Aerial 接管桌面 + 系统包精简 28 个 + 去广告收口。

## Star 历史

如果这篇教程帮你省了折腾时间，欢迎点个 Star ⭐ 也欢迎 Issue 交流。
