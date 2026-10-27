# 电视「家长管控」教程：音量上限锁 + 信号源锁 + 儿童观看定时锁 + 每日配额 / 分时段模式 / 倒计时暂停（Home Assistant）

> 不改电视系统、不装电视端 App，管控放在电视**外面**——孩子拿遥控器无解。
>
> 方案与品牌无关：小米、索尼、TCL、Apple TV、各类电视盒子……只要电视能接入 Home Assistant，这套教程就能直接套用。**本文以小米电视为例**，其他品牌只是接入方式不同（见「三、准备工作」对照表），自动化部分完全通用。
>
> **本仓库四个文件**：`README.md`（教程正文）｜`tv_lock_automations.yaml`（23 条自动化合集，可直接粘贴）｜`helpers-example.yaml`（辅助元素定义）｜`scripts-example.yaml`（5 个家长一键情景脚本）。
>
> 基础篇（音量锁 / 信号源锁 / 单次定时锁 / 倒计时暂停）看第五～七章；进阶篇（每日配额 / 分时段模式 / 续杯 / 周报）看第八～十章。**只想快速上手的话，只做基础篇就够了。**

## 相关仓库（同一系列教程）

- [ha-midea-hualing-ac](https://github.com/cbywyz/ha-midea-hualing-ac) —— 美的/华凌空调接入 Home Assistant（云端集成教程+折腾实录）
- [ha-hualing-fan-broadlink](https://github.com/cbywyz/ha-hualing-fan-broadlink) —— 华凌风扇红外遥控接入 HA（Broadlink + 8 键编码库）
- [gree-yapqf-broadlink-smartir](https://github.com/cbywyz/gree-yapqf-broadlink-smartir) —— 格力空调 Broadlink+SmartIR 接入 HA
- [phicomm-aircat-m1](https://github.com/cbywyz/phicomm-aircat-m1) —— 斐讯悟空 M1 空气检测仪复活记（自建集成）

---
- [cbywyz/jdcloud-nezha-ax3000-rescue](https://github.com/cbywyz/jdcloud-nezha-ax3000-rescue) —— 京东云哪吒 AX3000 (RE-OS-03U) 刷机变砖与救砖完整实录（TTL/U-Boot 救援、固件选型、eMMC 修复）

## 一、为什么做这个

现在的智能电视，硬件和内容体验都相当成熟，但「家长管控」在几乎所有品牌上都是同一副面孔——**只有一个电视端的「儿童模式」**：设个密码，把不想让孩子看的内容圈起来。

这个思路方向是对的，只是「密码长在电视上」有个绕不开的共性难点（与品牌无关）：

1. **家长输密码时，孩子就在旁边盯着**——几位数字看两遍就记住了，防护力随时间递减；
2. **没有「锁定信号源 / 限制观看时长 / 限制音量」这类精细化开关**——孩子把信号源一切、或者想控制"今天只看 30 分钟"，电视端大多没有对应入口；
3. 儿童模式是整体开关：要么全锁要么全开，没法只锁某个信号源、也没法只限制音量。

这不怪厂商——家长管控是典型的小众细分需求，不值得为它加重电视系统，没做深做细是合理的取舍。

所以思路反过来：**把管控从电视里搬到电视外**。孩子能摸到的只有电视和遥控器，而管控大脑在家庭服务器上，他连密码长什么样都没见过。

Home Assistant（HA）把电视接入后，电视对 HA 来说就是一个 `media_player` 实体——音量、信号源、开关机全都能远程读写。在这个实体上做自动化，就实现了电视端方案做不到的管控粒度：

- **音量上限锁**：可以随便调小，超过上限自动拉回，绝不允许比上限大；
- **信号源锁**：锁定某个输入（比如机顶盒所在的 HDMI 1），被切走自动秒切回来；
- **儿童观看定时锁**：设定观看时长，倒计时到点自动上锁；再设定锁定时长，到点自动解锁（也可以一直锁到家长手动解）。

孩子在电视前无论按什么，都拗不过电视背后的自动化——**没有密码可破，因为根本就没有密码**。

## 二、方案架构

```
┌──────────────┐  集成通道(云/局域网) ┌─────────────────┐
│  电视(被管设备)│ ◄─────────────────► │  Home Assistant  │
│              │    media_player     │  (管控大脑)       │
└──────────────┘                     └───────┬─────────┘
                                          │ 自动化
                            ┌─────────────┼─────────────┐
                            ▼             ▼             ▼
                        音量上限锁      信号源锁      儿童定时锁
                       (超限拉回)    (切走秒回+强切)  (倒计时上/解锁)
```

核心思路三条：

| 管控 | 手段 | 关键点 |
|------|------|--------|
| 音量上限 | 监听 `volume_level` 变化，超限立即拉回 | 0.5% 防抖，避免拉回动作自己触发自己 |
| 信号源 | 锁定开关打开**无条件强切** + 换台拦截 + ADB 10 秒增强校验（云通道只留 5 分钟兜底） | 不信任电视上报的 source 属性（会停更），以 ADB 读到的前台/输入口为准 |
| 定时锁 | HA 原生 `timer` 倒计时 → 到点打开锁定开关 | 到点自动解锁，手动锁永不自动解除 |

## 三、准备工作

1. **Home Assistant**：任意部署方式均可（我跑在软路由 Docker 里）。
2. **电视接入 HA**——各品牌对照表：

| 设备类型 | 接入方式 | 备注 |
|----------|----------|------|
| 小米电视（本文示例） | Xiaomi Miot 集成 | 走米家云，偶发丢指令需补发（见踩坑 1） |
| 索尼 / TCL / 雷鸟等 Android TV 电视、安卓盒子 | HA 内置 **Android TV** 集成 | 局域网直连，指令秒达，连补发都可省 |
| Apple TV | `apple_tv` 集成 | 支持 `volume_set` |
| Fire TV | ADB 方式 | 音量可用 |
| 部分 LG / 三星电视 | webOS / 各自集成 | `select_source` 支持程度因型号而异，接入后先看 `source_list` 属性 |

   接入后先确认电视实体有 `volume_level`（音量）和 `source`（信号源）能力，有就能整套部署。

   **本文以小米电视为例**：小米电视本身体验一直在线——画质、系统流畅度、内容生态都很能打，走 Xiaomi Miot 接入 HA 也省心，接入后得到 `media_player.xxx` 实体，音量、信号源、播放状态都能读写。
3. （可选，小米电视体验更好）**Android TV / ADB 辅助通道**：小米电视本质是 Android TV，加接 Android TV 集成后音量控制可以双通道下发，云指令偶发丢失时有兜底。本教程主链路只依赖主集成，可选通道不做硬性要求。

   > 想更进一步折腾电视**本体**（去广告 / 换极简桌面 / 修信号源键），见[附录 A](#附录-a小米电视系统级折腾实录去广告--换极简桌面--修信号源键)——全部走 ADB、可逆，与踩坑 16/17/18/19 互为印证。

> 下文所有 YAML 里的 `media_player.living_room_tv` 请替换成你自己的电视实体 ID（开发者工具 → 状态里查）；示例中的信号源列表、音量范围同理按你电视的实际属性调整。

## 四、第一步：创建辅助元素（Helpers）

整套系统共用 **8 个辅助元素**。推荐直接把下面内容加进 `configuration.yaml`（或用 UI 逐个创建）：

```yaml
input_boolean:
  tv_source_lock:
    name: 电视信号源锁定
    icon: mdi:television-classic-lock
  dian_shi_yin_liang_suo_ding:
    name: 电视音量锁定
    icon: mdi:volume-high
  # 内部标记：本次锁定是倒计时自动上的锁（手动开的锁不标记）
  ben_ci_wei_dao_ji_shi_suo_ding:
    name: 本次为倒计时锁定

input_select:
  tv_lock_source:
    name: 电视锁定信号源
    icon: mdi:import
    options:
      - TV
      - DTMB
      - AV
      - HDMI 1
      - HDMI 2

input_number:
  # 每天给孩子填一个数，填几就倒计时几分钟，填 0 取消
  er_tong_guan_kan_shi_chang:
    name: 儿童观看时长（分钟）
    min: 0
    max: 300
    step: 1
    icon: mdi:timer-outline
  # 上锁后再锁多久自动解，填 0 = 一直锁到家长手动解
  suo_ding_shi_chang:
    name: 自动解除锁定时长（分钟）
    min: 0
    max: 300
    step: 1
    icon: mdi:timer-lock-outline
  dian_shi_yin_liang:
    name: 电视音量
    min: 0
    max: 100
    step: 1
    icon: mdi:volume-high

timer:
  pin_dao_suo_ding_dao_ji_shi:
    name: 频道锁定倒计时
    icon: mdi:timer-sand
  pin_dao_suo_ding_jie_chu:
    name: 频道锁定解除倒计时
    icon: mdi:timer-lock
```

> `options` 里的信号源列表请按你自己电视的 `source_list` 属性填写。

## 五、第二步：音量上限锁

### 5.1 面板改音量自动下发（可选，但强烈建议）

在 HA 面板上放个音量滑条，改完自动下发到电视。**停手 2 秒**才下发，避免拖动过程刷屏：

```yaml
- id: tv_apply_volume
  alias: "[电视] 应用自定义音量"
  description: 面板改「电视音量」数值 → 停手 2 秒后自动下发到电视
  mode: single
  trigger:
    - platform: state
      entity_id: input_number.dian_shi_yin_liang
      for: "00:00:02"
  action:
    - service: media_player.volume_set
      target:
        entity_id: media_player.living_room_tv
      data:
        volume_level: "{{ states('input_number.dian_shi_yin_liang') | float(0) / 100 }}"
      continue_on_error: true
    # 如有第二通道（如 Android TV 集成）可再来一发（不接就删掉这条）
    - service: media_player.volume_set
      target:
        entity_id: media_player.android_tv
      data:
        volume_level: "{{ states('input_number.dian_shi_yin_liang') | float(0) / 100 }}"
      continue_on_error: true
```

### 5.2 音量上限锁（核心）

打开「电视音量锁定」后，「电视音量」数值即**上限**：可以随便调小，超过就拉回。只认 **0.5%** 以上的超限——拉回动作本身造成的微小回弹不会再触发，防死循环：

```yaml
- id: tv_volume_lock
  alias: "[电视] 音量锁定（上限模式，超了拉回）"
  description: >-
    打开「电视音量锁定」后，「电视音量」里填的数值就是音量上限：
    可以随便调小，但只要超过上限就自动拉回到上限值。
    只认 0.5% 以上的超限，避免自己拉回时反复触发。
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
      target:
        entity_id: "{{ trigger.entity_id }}"
      data:
        volume_level: "{{ states('input_number.dian_shi_yin_liang') | float(0) / 100 }}"
```

## 六、第三步：信号源锁定

### 6.1 锁定生效立即强切（关键！）

锁定开关一打开就**无条件**切到锁定源。注意：不看电视上报的 `source` 属性——云端集成缓存的这个属性经常不准（实际换了源它还报旧值），依赖它就会漏切：

```yaml
- id: tv_lock_engage_switch
  alias: "[电视] 锁定生效立即强切信号源"
  description: 锁定开关打开且电视开着 → 立刻切到锁定源，5 秒后补发一次；不依赖 source 属性。
  mode: single
  trigger:
    - platform: state
      entity_id: input_boolean.tv_source_lock
      to: "on"
  condition:
    - condition: state
      entity_id: media_player.living_room_tv
      state: "on"
  action:
    - service: media_player.select_source
      target:
        entity_id: media_player.living_room_tv
      data:
        source: "{{ states('input_select.tv_lock_source') }}"
    - delay: "00:00:05"
    - service: media_player.select_source
      target:
        entity_id: media_player.living_room_tv
      data:
        source: "{{ states('input_select.tv_lock_source') }}"
```

> **为什么发两次？** 云端集成（如小米走米家云）的指令偶发限流/丢失，实测单发有时石沉大海，5 秒后补发一次基本 100% 命中。**走局域网直连的集成（Android TV 等）指令可靠，第二条补发可以删掉。**

### 6.2 换台拦截（云通道，5 分钟兜底）

锁定期间任何人切走信号源，电视状态一变就自动切回；云指令偶发限流，保留 5 分钟兜底：

```yaml
- id: tv_source_lock
  alias: "[电视] 信号源锁定"
  description: 锁定开启时信号源被切走自动切回（云通道）；云指令偶发限流，降为每 5 分钟兜底。
  mode: single
  max_exceeded: silent
  trigger:
    # 锁定开关打开时也切一次（条件判断版本，与 6.1 的强切互为补充）
    - platform: state
      entity_id: input_boolean.tv_source_lock
      to: "on"
    - platform: state
      entity_id: media_player.living_room_tv
    - platform: time_pattern
      minutes: "/5"
  condition:
    - condition: state
      entity_id: input_boolean.tv_source_lock
      state: "on"
    - condition: template
      value_template: >-
        {% set want = states('input_select.tv_lock_source') %}
        {% set cur = state_attr('media_player.living_room_tv','source') %}
        {{ states('media_player.living_room_tv') == 'on'
           and (cur is none or cur != want) }}
  action:
    - service: media_player.select_source
      target:
        entity_id: media_player.living_room_tv
      data:
        source: "{{ states('input_select.tv_lock_source') }}"
```

> **为什么从每分钟降到 5 分钟**：见踩坑 15。每分钟一条云端 `select_source` 会把米家云通道打爆（遥控器失灵、电视卡死）。真正的秒级兜底交给 6.3 的 ADB 通道。

### 6.3 ADB 增强校验（推荐，专治"切几次就锁不住"）

这一节是本教程 2026-10 新增的核心修正。老版本只靠云实体的 `source` 属性判断，而云属性有两个硬伤：

1. **有缓存、会停更**：电视切到 App、切了信号源，云属性可能几分钟甚至几小时不刷新；
2. **看不见 App 盲区**：孩子打开云视听/B 站，云属性仍报原来的 HDMI，HA 认为"没切走"，永远不触发。

结果就是：**刚上锁时好用，切两三次之后锁就形同虚设**。这不是自动化写错了，是判据本身失效。

修法：用 `androidtv` 集成（局域网 ADB 直连）每 10 秒读一次电视**本地真实状态**，发现不对就本地切回，完全不经过云：

```yaml
- id: tv_source_lock_adb
  alias: "[电视] 信号源锁定 ADB 增强校验"
  description: 锁开且电视开着 → 每 10 秒用 ADB 读真实信号源；被切到别的 HDMI 或切到 App，立刻本地切回。
  mode: single
  max_exceeded: silent
  trigger:
    - platform: time_pattern
      seconds: "/10"
  condition:
    - condition: state
      entity_id: input_boolean.tv_source_lock
      state: "on"
    # ⚠️ 别用云实体判断开关机：云 state 也会停更，关机后照样报 on（踩坑 23）
    - condition: template
      value_template: "{{ states('media_player.living_room_tv_adb') in ['playing', 'paused', 'idle', 'on'] }}"
    # 只处理已验证过 input 编号的源
    - condition: template
      value_template: "{{ states('input_select.tv_lock_source') in ['HDMI 1', 'HDMI 2'] }}"
  action:
    - service: androidtv.adb_command
      continue_on_error: true
      target:
        # ⚠️ 必须是 androidtv 的 media_player 实体，不能写 remote.*（见下方踩坑 21）
        entity_id: media_player.living_room_tv_adb
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

**怎么验证它真的在执行（不用看画面）**

`media_player.android_tv_<ip>` 这个实体带一个 `adb_response` 属性，值就是上一条 adb 命令的
stdout/stderr。所以可以先发一条"只读诊断"命令，直接从属性里读回电视端算出来的结论：

```yaml
service: androidtv.adb_command
target: { entity_id: media_player.living_room_tv_adb }
data:
  command: >-
    T=23; R=$(dumpsys activity activities | grep -m1 mResumedActivity);
    C=$(settings get global mitv.tvplayer.hdmi.last.source | tr -d '\r');
    echo "FG=$R C=$C"
```

属性返回 `FG=...MainActivity C=23` 就说明 HA→电视的命令链是通的，且读到了真实值。
另一条更硬的验证：让 HA `settings put global x 123`，再用电脑 `adb shell settings get global x`
读回，能读到就证明不是"假成功"。

**判据（小米电视实测）**

| 项 | 命令 / 值 | 说明 |
| --- | --- | --- |
| 切源入口 | `am start -a com.xiaomi.mitv.tvplayer.EXTSRC_PLAY -n com.xiaomi.mitv.tvplayer/.ExternalSourceActivity --ei input N` | `23/24/25` = HDMI 1/2/3，实测 23↔24 可来回切 |
| 当前是否在信号源界面 | `dumpsys activity activities \| grep mResumedActivity` | 含 `ExternalSourceActivity` 才是在看 HDMI |
| 当前是哪个 HDMI | `settings get global mitv.tvplayer.hdmi.last.source` | 切源即更新；**只有在前台是 ExternalSourceActivity 时才代表当前源** |
| 在 App 里 | 前台是别的包名 | 直接切回目标 HDMI，这正是云通道的盲区 |

**三个关键设计点**

1. **必须前台判断 + source 判断双条件**：只看 `last.source` 会误判——孩子切到 App 时这个值还停在最后看的 HDMI，看起来"没被切走"。
2. **只对验证过 input 编号的源生效**：不同品牌/型号的编号不一样，没验证过的源宁可不动，也不要乱切（切错会黑屏）。
3. **自包含一行命令**：`androidtv.adb_command` 不回传 stdout（`response: true` 在部分版本报 400），所以让电视自己判断、自己决定切不切，HA 只负责发。

**已验证 / 未验证**

- ✅ 已实测：`23↔24` 能真实切换；`HA → androidtv.adb_command` 服务调用返回 200；电视端 `settings` 会同步更新。
- ⚠️ 待实测：HA 与电视的 ADB 长连接稳定性。很多电视的 `adbd` **只允许一个客户端**——电脑 `adb` 连着时 HA 连不上（反之亦然），日志表现为 `Connection reset by peer` / `Reading timed out`。排障前先 `adb kill-server` 断开电脑，再看 HA 日志；首次连接还要在电视上点「允许调试」。
- ⚠️ 待实测：电视待机时 ADB 不可达，自动化会失败（已用云实体 `state == on` 过滤），`continue_on_error: true` 保证不刷错误日志。

### 6.4 补充：锁 App（只锁信号源锁不住内容 App）

> 2026-10-27 实测补的第二个洞。**只锁信号源 = 假锁**，两种情况都会漏：
> ① 孩子直接从桌面打开云视听/B站，`source` 属性根本不变（踩坑 13）；
> ② 锁回的那个 HDMI 如果**没有信号源**，电视十几秒后自己弹回桌面 —— 实测 20:35:30 锁回 HDMI 1，
>    20:35:47 就回到 Aerial 桌面，形成"黑屏 15 秒 → 桌面 15 秒"的循环，孩子在桌面那段时间照样能点开 App。

**做法：锁开就 `pm disable-user` 掉娱乐类 App，锁关 `pm enable` 恢复。** 最土但最有效，全程可逆、
不碰系统分区、不需要 root：

```yaml
- id: tv_app_lock_adb
  alias: "[电视] 锁定期间禁用娱乐 App"
  mode: single
  max_exceeded: silent
  trigger:
    - platform: state
      entity_id: input_boolean.tv_source_lock
    - platform: time_pattern      # 兜底对账，手滑/HA 重启/命令漏执行 2 分钟内自愈
      minutes: "/2"
  action:
    - service: androidtv.adb_command
      continue_on_error: true
      target:
        entity_id: media_player.living_room_tv_adb
      data:
        command: >-
          {% set apps = 'com.xiaodianshi.tv.yst com.ktcp.video com.tencent.qqmusictv com.ixigua.android.tv.wasu com.gitvdemo.video com.dangbeimarket com.cibn.tv' %}
          {% if is_state('input_boolean.tv_source_lock', 'on') %}
          for p in {{ apps }}; do pm list packages -e | grep -qx "package:$p" && pm disable-user --user 0 $p; done; echo APPS-LOCKED
          {% else %}
          for p in {{ apps }}; do pm list packages -d | grep -qx "package:$p" && pm enable $p; done; echo APPS-UNLOCKED
          {% endif %}
```

**这份禁/启脚本的三个安全点**

1. **先查存在再动手**：`pm list packages -e | grep -qx` 命中才 disable，`-d` 命中才 enable —— 包不存在/已禁用都不会报错，可反复执行（幂等）。
2. **只动白名单里的包**：脚本里的包名是硬编码白名单，永远不碰桌面、输入法、systemui、管家（禁了电视只能 adb 救）。
3. **解锁是"恢复"不是"保证启用"**：`pm enable` 只对白名单里的包生效，不会把系统里其它本来就被禁的包一并放开。

**怎么找该禁哪些包**：`adb shell pm list packages -3 -e`（已启用的第三方 App）先看全量，再按包名判断；
小米电视实测：`com.xiaodianshi.tv.yst`=B站小电视、`com.ktcp.video`=云视听极光、
`com.tencent.qqmusictv`=QQ音乐、`com.ixigua.android.tv.wasu`=西瓜视频、
`com.gitvdemo.video`=电视家、`com.dangbeimarket`=当贝市场、**`com.cibn.tv`=CIBN酷喵影视（优酷）**。
**学习类（如洪恩识字 `com.hongen.app.word.ott.vip`）别一刀切**。

> 认包名的小技巧：名字看不出品牌的，`dumpsys package <包名> | grep -iE "activity|scheme"` 看内部类名/URL scheme 就知道是谁。例如
> `com.cibn.tv` 里出现 `com.youku.tv.home.activity.YkottHomeActivity` 和 scheme `cibntv_yingshi` → 就是酷喵。

**实测结论（2026-10-27）**

| 步骤 | 结果 |
| --- | --- |
| 锁开 +14s | 6 个包全部出现在 `pm list packages -d`，后台进程被强停、正在放的 B站 被踢出 |
| 锁开时 `am start` B站 | 拉不起来（前台始终不是该 App） |
| 锁关 +14s | 6 个包全部回到 `pm list packages -e` |
| 锁关后再 `am start` | `FG=com.xiaodianshi.tv.yst/.ui.main.MainActivity`，正常播放 |


## 七、第四步：儿童观看定时锁

玩法：给孩子填「儿童观看时长」= 30 → 30 分钟后自动上锁；「自动解除锁定时长」填 30 → 再过 30 分钟自动解锁（填 0 = 一直锁着，家长手动解）。**手动开的锁定永不被自动解除**。

> 补充：配合 7.4 的暂停机制后，这个倒计时统计的是"真正看电视的时间"——中途关机的时间不算在内。

### 7.1 启动倒计时 / 取消

```yaml
- id: tv_lock_countdown_start
  alias: "[电视] 频道锁定倒计时启动"
  description: 填「儿童观看时长（分钟）」即启动倒计时；改数字重新计时，填 0 取消。
  mode: restart
  trigger:
    - platform: state
      entity_id: input_number.er_tong_guan_kan_shi_chang
  condition:
    - condition: template
      value_template: "{{ states('input_number.er_tong_guan_kan_shi_chang') | int(0) > 0 }}"
  action:
    - service: timer.start
      target:
        entity_id: timer.pin_dao_suo_ding_dao_ji_shi
      data:
        duration: >-
          {% set m = states('input_number.er_tong_guan_kan_shi_chang') | int(0) %}
          {{ '{:02d}:{:02d}:00'.format(m // 60, m % 60) }}

- id: tv_lock_countdown_cancel
  alias: "[电视] 取消频道锁定倒计时"
  description: 时长调到 0 立刻取消倒计时。
  mode: single
  trigger:
    - platform: numeric_state
      entity_id: input_number.er_tong_guan_kan_shi_chang
      below: 1
  action:
    - service: timer.cancel
      target:
        entity_id: timer.pin_dao_suo_ding_dao_ji_shi
```

### 7.2 到点自动上锁（并按需启动解除倒计时）

```yaml
- id: tv_lock_countdown_finish
  alias: "[电视] 倒计时到点自动锁频道"
  description: 倒计时结束 → 打开信号源锁定；若「自动解除锁定时长」>0 则同时启动解除倒计时。
  mode: single
  trigger:
    - platform: event
      event_type: timer.finished
      event_data:
        entity_id: timer.pin_dao_suo_ding_dao_ji_shi
  action:
    - service: input_boolean.turn_on
      target:
        entity_id: input_boolean.tv_source_lock
    - service: input_boolean.turn_on
      target:
        entity_id: input_boolean.ben_ci_wei_dao_ji_shi_suo_ding
    - choose:
        - conditions:
            - condition: template
              value_template: "{{ states('input_number.suo_ding_shi_chang') | int(0) > 0 }}"
          sequence:
            - service: timer.start
              target:
                entity_id: timer.pin_dao_suo_ding_jie_chu
              data:
                duration: >-
                  {% set m = states('input_number.suo_ding_shi_chang') | int(0) %}
                  {{ '{:02d}:{:02d}:00'.format(m // 60, m % 60) }}
```

### 7.3 到点自动解锁 / 手动解锁清理

```yaml
- id: tv_lock_release_finish
  alias: "[电视] 定时锁定到点自动解除"
  description: 解除倒计时结束，且本次确实是倒计时锁的 → 自动解锁（手动锁不受影响）。
  mode: single
  trigger:
    - platform: event
      event_type: timer.finished
      event_data:
        entity_id: timer.pin_dao_suo_ding_jie_chu
  condition:
    - condition: state
      entity_id: input_boolean.ben_ci_wei_dao_ji_shi_suo_ding
      state: "on"
  action:
    - service: input_boolean.turn_off
      target:
        entity_id: input_boolean.tv_source_lock
    - service: input_boolean.turn_off
      target:
        entity_id: input_boolean.ben_ci_wei_dao_ji_shi_suo_ding

- id: tv_lock_unlock_cleanup
  alias: "[电视] 手动解锁时清理定时标记"
  description: 锁定被关掉（手动解锁）→ 清标记 + 取消解除倒计时，防止到点误解锁。
  mode: single
  trigger:
    - platform: state
      entity_id: input_boolean.tv_source_lock
      to: "off"
  action:
    - service: input_boolean.turn_off
      target:
        entity_id: input_boolean.ben_ci_wei_dao_ji_shi_suo_ding
    - service: timer.cancel
      target:
        entity_id: timer.pin_dao_suo_ding_jie_chu
```

### 7.4 倒计时可暂停：关机暂停、开机继续

HA 的 `timer` 是真·墙钟，启动那一刻就开始走，**不管电视是不是还开着**。孩子下去吃个饭、或者嫌吵顺手关了电视出去玩，倒计时照样走完——到点照样上锁，"看 30 分钟"实际只看了 10 分钟，孩子觉得亏，家长还得手动解锁，来回折腾。

HA 的 `timer` 自带 `pause` / `start`（不带 `duration` 时从剩余时间继续），所以只要盯住电视开关状态就能补上这块：

```yaml
# 只暂停「观看」类倒计时（单次观看倒计时 + 续杯倒计时）；
# 锁定后的「解除倒计时」是惩罚时间，保持按墙钟走，关机不暂停。
- id: tv_timer_pause_when_off
  alias: "[电视] 关机暂停观看倒计时"
  description: 电视关机满 10 秒 → 暂停进行中的观看倒计时/续杯倒计时，不再空耗时间。判据走 ADB 实体（云实体 state 会停更）。
  mode: single
  trigger:
    # ⚠️ 必须用 template trigger：state trigger 的 to: off 抓不到 unavailable（ADB 断开）
    - platform: template
      value_template: "{{ states('media_player.living_room_tv_adb') not in ['playing', 'paused', 'idle', 'on'] }}"
      for: "00:00:10"
  action:
    - choose:
        - conditions:
            - condition: state
              entity_id: timer.pin_dao_suo_ding_dao_ji_shi
              state: "active"
          sequence:
            - service: timer.pause
              target:
                entity_id: timer.pin_dao_suo_ding_dao_ji_shi
    - choose:
        - conditions:
            - condition: state
              entity_id: timer.xu_bei_dao_ji_shi
              state: "active"
          sequence:
            - service: timer.pause
              target:
                entity_id: timer.xu_bei_dao_ji_shi

- id: tv_timer_resume_when_on
  alias: "[电视] 开机继续观看倒计时"
  description: 电视开机 → 把暂停中的倒计时接着跑（从剩余时间继续，不重置）。判据走 ADB 实体。
  mode: single
  trigger:
    - platform: template
      value_template: "{{ states('media_player.living_room_tv_adb') in ['playing', 'paused', 'idle', 'on'] }}"
  action:
    - choose:
        - conditions:
            - condition: state
              entity_id: timer.pin_dao_suo_ding_dao_ji_shi
              state: "paused"
          sequence:
            - service: timer.start
              target:
                entity_id: timer.pin_dao_suo_ding_dao_ji_shi
    - choose:
        - conditions:
            - condition: state
              entity_id: timer.xu_bei_dao_ji_shi
              state: "paused"
          sequence:
            - service: timer.start
              target:
                entity_id: timer.xu_bei_dao_ji_shi
```

三个设计细节：

1. **为什么加 `for: "00:00:10"`**：部分电视（尤其是走云通道的）在换台、投屏切换、短暂休眠时会甩出一个瞬时 `off`，不加延时就会被这种假关机误暂停。10 秒足够过滤掉误判，又不影响真实场景。
2. **为什么用 `choose` 判状态**：`timer.pause` / `timer.start` 对不在对应状态的 timer 会报错（日志里刷一片红）。先用 `condition: state` 过滤一遍，只对 `active` 的暂停、只对 `paused` 的继续，日志干净。写成两个独立的 `choose` 而不是一个带多条 condition 的，是因为两个 timer 之间不需要"同时成立"的关系，各自独立才符合条件不佳时互不牵连。
3. **记住 `timer.start` 的两种用法**：带 `duration` = 重新开始；**不带 `duration` = 从暂停处继续**。这是整个暂停机制的支点，别写顺手给它补个 duration 上去，那就变成"开机重置 15 分钟"了。

哪些倒计时该暂停、哪些不该，是这样的：

| 倒计时 | 含义 | 关机是否暂停 |
|---|---|---|
| `timer.pin_dao_suo_ding_dao_ji_shi` | 单次观看时长 | ✅ 暂停（统计真看了多久） |
| `timer.xu_bei_dao_ji_shi` | 续杯的 5 分钟 | ✅ 暂停 |
| `timer.pin_dao_suo_ding_jie_chu` | 上锁后的惩罚/冷却时间 | ❌ 不暂停（否则关掉电视就能无限坐牢豁免） |

最后一条尤其重要：**惩罚时间必须按墙钟走**。要是允许暂停解除倒计时，孩子关了电视这 30 分钟就永远不流动，等于花一手"关机"就能赖掉惩罚。

> 实测记录：设 5 分钟 → 播到 4:51 时关电视 → 10 秒后timer 状态变 `paused`，剩余停在 4:51 → 干等 8 秒仍然是 4:51（确认暂停有效）→ 再开电视 → 从 4:51 继续往下走。

## 八、进阶篇：每日观看配额（堵住"看完一轮又一轮"）

第五章到第七章解决的是"单次看多久"，但它有个天然漏洞：**看满 15 分钟锁上 → 家长解锁 → 又能看 15 分钟**，一天可以无限循环。

要堵死它，就得有"今天总共看了多久"这个数。

### 8.1 为什么不用 history_stats

网上教程清一色用 `history_stats` 统计开机时长，它确实准，但有个致命不便：**`sensor` 域没有 reload 服务**，加了之后必须**重启 HA** 才生效。对一台正在给孩子放动画片的电视来说，重启是要命的。

本教程改用一个"土办法"——**每分钟累加器**：电视开着就每分钟给 `input_number` 加 1。精度足够（±1 分钟），而且 `input_number` 是热重载的，改完立刻生效。

### 8.2 需要的辅助元素

| 辅助元素 | 作用 | 建议值 |
|---|---|---|
| `input_number.mei_ri_guan_kan_pei_e` | 每天最多看多少分钟，**0 = 不限** | 60 |
| `input_number.jin_ri_yi_guan_kan` | 今日已看（系统自动累加，别手动改） | — |
| `input_number.ben_zhou_lei_ji_guan_kan` | 本周累计（**只含到昨天**，展示时用值 + 今日） | — |
| `input_boolean.jin_ri_pei_e_yi_yong_jin` | "今天配额已用尽"标记，防重复上锁/重复推送 | — |

### 8.3 三条自动化

1. **`tv_watch_minutes_acc`**：每分钟触发一次，条件是 **ADB 实体**在 `['playing','paused','idle','on']`（**别用云实体 `state == on`**：云 state 会停更，关机后照样报 on，一天能虚高两百多分钟，见踩坑 23），动作是给"今日已看"+1；
2. **`tv_quota_enforce`**：`今日已看 ≥ 配额` 且不在"今晚自由"模式且未标记过 → 打标记 + 上锁 + 推通知；
3. **`tv_daily_reset`**：每天 0:01 把"今日已看"并入"本周累计"后清零，并清掉标记。**注意这意味着 `ben_zhou` 只含到昨天**，面板/周报要展示"本周累计"时得用 `ben_zhou + jin_ri`，否则会出现"今日 385 > 本周 295"的怪现象。

> 完整 YAML 见 `tv_lock_automations.yaml` 的「进阶篇」段落，`tv_quota_enforce` 里的通知部分换成你自己的推送服务即可（示例用的是飞书）。

## 九、进阶篇：分时段模式（作业 / 睡眠 / 今晚自由）

一个 `input_select` 把日常场景收拢成四档，家长不用每次挨个拨开关：

| 模式 | 行为 |
|---|---|
| 常规 | 只执行音量锁、信号源锁、单次倒计时、每日配额 |
| **作业模式** | 立刻上锁；期间**任何解锁都会在 3 秒内被自动锁回**；不给续杯 |
| **睡眠模式** | 电视一开机就自动关掉 |
| **今晚自由** | 一键全家豁免：解锁 + 停掉所有倒计时 + 清掉配额标记 |

外加两个自动切换：**每晚 22:30 自动进睡眠模式，早上 7:00 自动回常规**。

实现要点（三条自动化）：

- `tv_homework_mode_relock`：作业模式下一旦检测到锁定开关变成 `off`，3 秒内立刻 `turn_on` 拉回来；
- `tv_sleep_enforce`：睡眠模式下电视变**开机**（ADB 实体进白名单）→ 立刻 `media_player.turn_off`；
- `tv_mode_night_auto` / `tv_mode_morning_auto`：按时间改 `input_select`。

> ⚠️ 别忘了把第七章的"到点自动解锁"（`tv_lock_release_finish`）**加上"当前不是作业模式 / 今日配额未用尽"的条件**，否则作业模式会被定时解锁悄悄破掉。

## 十、进阶篇：再看 5 分钟（续杯）与每周周报

### 10.1 续杯

孩子磨你的时候，与其每次手动解锁，不如给一个"再看 5 分钟"按钮：解锁 → 5 分钟后自动重新锁上，**每天最多 2 次**（用完推通知拒绝）。作业模式下不给续。

- 计数用 `counter.jin_ri_xu_bei_ci_shu`（每天 0:01 随统计一起 `reset`）；
- 倒计时用 `timer.xu_bei_dao_ji_shi`，`tv_xu_bei_finish` 到点重新上锁。

### 10.2 周报

每周一早上 9 点推送上周观看情况（本周累计 + 日均），推完把"本周累计"清零。家长不用盯面板也心里有数。

### 10.3 五个情景脚本

把常用组合做成脚本，面板上放按钮一点即达，完整内容见 `scripts-example.yaml`：

| 脚本 | 干什么 |
|---|---|
| `script.tv_mode_30min` | 看 30 分钟（设时长 + 启动倒计时） |
| `script.tv_mode_homework` | 作业模式（切模式 + 上锁） |
| `script.tv_mode_sleep` | 睡眠模式（切模式 + 关电视） |
| `script.tv_mode_free` | 今晚自由（切模式 + 解锁 + 停所有倒计时） |
| `script.tv_xu_bei` | 再看 5 分钟（判断次数上限 → 解锁 → 启动续杯倒计时） |

> 脚本域支持热重载：改完 `scripts.yaml`，开发者工具 → 服务 → `script.reload` 即可，不用重启。

## 十一、面板卡片建议

在 HA 仪表盘放一张 entities 卡（或塞进 Bubble Card 弹窗），家长一个界面全搞定：

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

> 小技巧：把 `timer.*` 实体直接放上卡片，前端每秒刷新自带倒计时跳动，比任何模板卡都省事。

### 11.1 进阶版：家长一键按钮

做完进阶篇后，面板上再加一排按钮，日常操作一个都不用翻菜单：

```yaml
type: horizontal-stack
cards:
  - type: button
    name: 看 30 分钟
    icon: mdi:timer
    icon_height: 32px
    tap_action:
      action: call-service
      service: script.turn_on
      target: { entity_id: script.tv_mode_30min }
  - type: button
    name: 作业模式
    icon: mdi:book-open-variant
    icon_height: 32px
    tap_action:
      action: call-service
      service: script.turn_on
      target: { entity_id: script.tv_mode_homework }
  - type: button
    name: 睡眠模式
    icon: mdi:sleep
    icon_height: 32px
    tap_action:
      action: call-service
      service: script.turn_on
      target: { entity_id: script.tv_mode_sleep }
```

第二排放「今晚自由」「再看 5 分钟」两个按钮，加一张 entities 卡显示今日已看 / 配额 / 续杯次数 / 倒计时即可。

> 面板入口小建议：功能多了以后，单独建一个仪表盘（HA 里「设置 → 仪表盘 → 添加」）挂到侧栏，比塞在某个面板的标签页里好找。

### 11.2 使用说明弹窗（家人不用问你）

功能一多，家里其他人根本不知道每个按钮是干嘛的。在面板上放一条「❓ 使用说明」，点开弹窗看说明——配合 Bubble Card 插件（HACS 安装）实现：

**触发按钮**（放在面板里）：

```yaml
type: custom:mushroom-template-card
primary: ❓ 使用说明
secondary: 点按查看各功能说明
icon: mdi:help-circle
icon_color: blue
tap_action:
  action: navigate
  navigation_path: "#help"
```

**说明弹窗**（同一视图里追加，`card_type` 必须是带连字符的 `pop-up`）：

```yaml
type: custom:bubble-card
card_type: pop-up
hash: "#help"
name: 电视管控使用说明
icon: mdi:help-circle
popup_mode: fit-content
width_desktop: 520px
cards:
  - type: markdown
    content: >
      ## 📺 电视管控使用说明

      **看 30 分钟**：立即开 30 分钟观看时间

      **作业模式**：上锁且禁止解锁，写完作业家长手动切回

      **睡眠模式**：开机即关；每晚 22:30 自动进入，07:00 恢复

      **今晚自由**：今晚不限时，明天自动恢复

      **再看 5 分钟**：倒计时结束后的续杯，每天最多 2 次

      **倒计时暂停**：关机超 10 秒自动暂停计时，开机接着看剩余时间
```

说明内容按你自己实际做的功能写就行，写清楚「每个按钮点了会发生什么」，家里人点开看一眼就懂。

## 十二、踩坑记录（省你半天排错）

1. **云通道指令偶发丢失/限流**：走云的集成（如小米 Miot）`select_source`、`volume_set` 单发有时不生效。对策：动作里加 `continue_on_error: true`，关键切换 5 秒后补发一次，再加 5 分钟兜底（别用每分钟，见踩坑 15）。局域网直连的集成没有这个问题。
2. **`source` 属性有缓存，不可全信**：实际换了信号源，属性可能还报旧值。所以「锁定生效那一刻」必须无条件强切，不能先判断 `cur != want` 再切——否则永远不触发。这也是本教程 6.1 与 6.2 分成两条自动化的原因。
3. **音量锁防自触发**：拉回动作本身会再触发一次音量变化事件，条件里用 `cur - target > 0.005`（0.5% 死区）掐掉回弹，不然会无限循环拉回。
4. **锁定开关（input_boolean）本身不触发任何东西**：只开开关不切源是新手最常见的坑，务必有一条「开关打开 → 执行切换」的自动化。
5. **手动锁与倒计时锁要区分**：用一个内部标记 `input_boolean`（7.2 里第 2 个动作）标记"这次是定时上的锁"，自动解锁前检查它——手动上的锁就永远不会被定时器偷偷解掉。
6. **Jinja 里 `{{ set }}` 是错的，必须 `{% set %}`**：写错的那条自动化会被判为 invalid **并自动 disabled**（状态变 `unavailable`），界面上不报错，只能从 `home-assistant.log` 里翻。改完 YAML 先本地搜一遍有没有 `{{ set`。
7. **自动化被自动禁用后，改 id 重装会留 residual 实体**：旧的 `automation.xxx` 会一直是 `unavailable`，要重启 HA 才清得掉（或用 WS `config/entity_registry/remove`）。所以改自动化时尽量保持 id 不变。
8. **`sensor` 域（history_stats、template sensor 里的部分平台）没有 reload 服务**：这是本教程坚持用 `input_number` 做累加器的原因——能热重载的才是家庭里敢动的东西。
9. **`timer.start` 带不带 `duration` 是两种完全不同的语义**：带 = 重新开始计时；不带 = 从 `paused` 的剩余时间继续。做暂停功能时千万别顺手补上 duration。
10. **`timer.pause` / `timer.start` 对状态不符的 timer 会报错**：`idle` 的 timer 调 `pause` 会在日志里刷红。动作外面套一层 `condition: state`（暂停只看 `active`、继续只看 `paused`）就干净了。
11. **区分"该暂停的时间"和"不该暂停的时间"**：观看类倒计时可以随关机暂停，但上锁后的冷却/惩罚时间必须按墙钟走——否则孩子关个电视就能把惩罚无限期冻结。
12. **Bubble Card 的弹窗 `card_type` 是 `pop-up`（带连字符）**：写成 `popup` 不报错，就是点了没反应，非常隐蔽。另外触发弹窗别用 bubble 自带的 button 卡（部分版本点了没反应），用普通卡片 `tap_action: navigate` 到 `#hash` 最稳（见 11.2）。

13. **信号源锁锁不住「电视内 App」（云视听 / 爱奇艺等）——这是机制盲区，先说结论**：锁只认 `media_player` 的 `source` 属性（物理信号源 HDMI / TV）。孩子在电视桌面里打开云视听这类内容 App，**不会切换信号源**，`source` 属性不变 → HA 判定"还在锁定的 HDMI 1"，不触发切回。所以单靠信号源锁**只能锁物理信号源（机顶盒 / 游戏机所在的 HDMI 口），锁不住电视自带的流媒体 App**。要在 App 层也管住，得另走 ADB / 应用层方案（**2026-10-27 已落地：锁开时 `pm disable-user` 娱乐 App，见 [6.4](#64-补充锁-app只锁信号源锁不住内容-app)**）。
14. **小米 Miot 云端通道极不稳定，「切回」依赖它，断线锁就暂时失效**：实测一晚断 3 次（其中一次长达 85 分钟零上报）。家里局域网 ping 电视 **0 丢包、0.36ms**，排除了自家网络——断的是小米云。后果：云端断开期间孩子切走信号源，HA 既收不到变化、也下不了切回指令，锁等于暂时失效。**局域网直连的电视（索尼 / TCL / 雷鸟等走 Android TV 集成）没有这问题**，这是"走云"必须接受的代价。要缓解，只能在锁定时加飞书告警（失联即通知家长），而不是硬扛云端。
15. **⛔ 红线：别对小米云实体做 `time_pattern` 高频 `select_source`（会打爆电视）**：为补第 13 条 App 盲区的漏，我曾加"每分钟无条件切回 HDMI 1"。结果：每分钟一条云端指令把小米电视（经 xiaomi_home 云端通道）打爆 → **遥控器失灵、电视卡死**。实测当时手动 `select_source(HDMI 2)` 下发返回 200，电视毫无反应，HA 侧 `source` 110+ 秒不更新 → 云端通道**双向皆断**（状态不上报、命令也不执行），断线时还每分钟狂发指令 = 雪上加霜。**真要强制踢回**：必须前置"电视 `last_updated` < 2 分钟才发"的门禁（失联时不发），且间隔 ≥5 分钟。
16. **ADB 补 App 盲区的方案，当年实测后放弃（2026-10 已重新启用）**：主路由常驻 ADB 守护读前台 App，违规就发 webhook 让 HA 切回——思路通，但**当年小米电视一重启「网络调试」就自动关闭**，守护随之失效，遂撤掉。后来把端口写进持久属性（`setprop persist.adb.tcp.port 5555`）并确认系统设置里调试常开，重启后 `5555` 仍在监听，才重新启用。**用前必须确认三件事**：① `getprop persist.adb.tcp.port` 有值；② `settings get global adb_enabled` 为 1；③ 真重启一次，`adb connect` 仍然成功。三者缺一，ADB 守护迟早会在某次重启后静默失效。
17. **（定位用）电视真实 IP 可能在旁路由 / 二级网段，别认错设备**：我家电电视真实 IP 是 `192.168.8.103`（在旁路由下的二级网段），而 `192.168.3.191` 是**另一台**小米设备。做 ADB 调试或写 IP 白名单前，先用 `adb connect <真IP>:5555` 确认连的是电视本身，别把指令发到别人设备上（同理，HA 里同名实体也要核对 `entity_id`，别张冠李戴）。
18. **⛔ 红线：禁用旧桌面（`com.mitv.tvhome`）会连带弄丢遥控「信号源 / Input」键**：mulan 固件的 Input 键（`KEYCODE_TV_INPUT=178`）是**前台应用**接管的，旧桌面是唯一接管方；而 Google TV 那套 `com.android.tv.action.VIEW_INPUTS` 在这固件上**没有接收方**，真正的信号源面板藏在 `com.droidlogic.tvinput/.shortcut.ShortCutActivity`（Amlogic 输入框架）里。给电视做任何「系统级优化」（去广告 / 换桌面）前，**先确认旧桌面状态**——本次实测：禁用它 → 信号源键失灵；恢复启用 → 立即复原。要换桌面就得自己补这个键（做法见[附录 A.5](#a5-信号源键修复本次核心)）。
19. **小米电视（mulan）走的是有线网卡，`dumpsys wifi` 显示 DISCONNECTED 是正常现象**：这台机器根本没有 `wlan0`，网络走 `eth0`。排障时看到「Wi-Fi 已断开」别顺着查网络——`ip addr` 一眼就能确认。同理 ADB 偶发超时也未必是故障，ping 通 + 5555 重连即可，多为瞬时抖动。
20. **云 `source` 属性会停更，这才是「切几次就锁不住」的真正原因（2026-10 定论）**：实测米家云实体 `media_player.xiaomi_cn_...` 的 `last_updated` 停在几个小时前，期间电视切 App、切 HDMI，`source` 一直报旧值。而锁的唯一判据就是这个属性 → 属性不动，自动化永远不触发。表现极具迷惑性：**刚上锁时有效，切两三次之后就怎么切都不管**。修法见 6.3——用 `androidtv` 集成走局域网 ADB，每 10 秒读 `dumpsys activity activities`（前台是不是 `ExternalSourceActivity`）+ `settings get global mitv.tvplayer.hdmi.last.source`（在哪个 HDMI），电视自己判断、自己切回；云通道降为 5 分钟兜底。**别再靠提高云指令频率去救，那只会打爆通道（踩坑 15）。**
21. **⛔ `androidtv.adb_command` 的 `entity_id` 必须写 `media_player`，写 `remote.*` 会「静默不执行」（2026-10-27 实测）**：androidtv 集成会同时注册 `media_player.android_tv_<ip>`（真正带 ADB 通道的那个）和 `remote.*`。把 `adb_command` 指向 `remote.*` 时：服务调用**返回 success**、自动化 `last_triggered` 照常更新，但电视端什么都没发生——只在 `home-assistant.log` 留一句 `Referenced entities remote.xxx are missing or not currently available`（因为该 remote 实体状态是 `unknown`，服务调用被直接跳过）。排查时极易误判成"电视 ADB 掉线"。**验收方法**：看目标 `media_player` 的 `adb_response` 属性有没有电视返回的文本（见 6.3）。
22. **一次只能连一个 adb 客户端，电脑连着 HA 就连不上（反之亦然）**：小电视的 adbd 是单会话的。电脑 `adb connect` 上去之后，HA 的 androidtv 立刻刷 `Connection reset by peer`，实体变 `unknown`、所有 ADB 自动化空转。**排障或抓 dumpsys 用完立刻 `adb kill-server`**，别让电脑的 adb 常驻；HA 侧重连一般 10 秒左右自愈，不行就 `homeassistant.reload_config_entry`（androidtv 条目）重载一次。
23. **云实体的 `state` 也会停更，别拿它判断开关机（2026-10-27 实测，代价是 385 分钟的假数据）**：一直以为云实体只有 `source` 会停更、`on/off` 是准的——**错**。实测电视关机后 `media_player.xiaomi_cn_...` 仍长期停在 `on`（`last_updated` 停在几小时前不动）。后果两个：①「今日观看分钟累计」在电视关机后继续每分钟 +1，一天下来虚高到 385 分钟；②「关机暂停观看倒计时 / 开机继续」永远不触发，锁定倒计时和续杯倒计时关机后照常空跑。修法：**所有开关机判据统一改成 androidtv 实体** `media_player.android_tv_<ip>` 的状态白名单 `['playing','paused','idle','on']`——正常开机是 `playing/paused/idle`（实测待机桌面 = `paused`），关机是 `off/standby`，ADB 断了是 `unavailable`，后三种都落在白名单外。**触发要用 template trigger，`to: 'off'` 抓不到 `unavailable`。**
    顺带一个显示坑：「本周累计」比「今日已看」还小，不是数据坏了——`ben_zhou` 只累计到昨天，0:01 才把今天并进去。周报文案要算 `ben_zhou + jin_ri`，别只报 `ben_zhou`。
    这个白名单不是拍脑袋定的，有源码依据：`site-packages/androidtv/androidtv/base_androidtv.py` 里 `screen_on is None` → state 为 None（HA 显示 `unavailable`）、`not screen_on` → `STATE_OFF`、`not awake` → `STATE_STANDBY`，只有「屏幕亮且已唤醒」才可能落到 `playing/paused/idle`。所以三种非开机态全在白名单外，**关机必然停表**。

## 十三、可以继续扩展的方向

- **断电/断网逃逸防护**：孩子拔插头或关路由器——加一条"电视恢复在线时，若处于管控时段或配额已用尽 → 自动重新上锁"；
- ~~**倒计时可暂停**：电视关机即暂停计时、开机继续，统计的是"真看了多久"而不是"过了多久"~~（已在 [7.4](#74-倒计时可暂停关机暂停开机继续) 实现）；
- **观影模式联动**：电视一开 → 灯调暗、空调设 26°、窗帘关上，关机自动恢复；
- **无人自动关机**：手机离家或房间人体传感器 10 分钟无动静且电视还开着 → 自动关机省电；
- **解锁要家长批准**：孩子喊解锁 → IM 推一条带「批准/拒绝」按钮的消息，点了才解锁；
- **锁面板本身**：HA 的面板编辑权限可以按用户控制，孩子登录的账号设为只读即可。

> 其中「倒计时暂停」「每日配额」「分时段模式」「续杯」「周报」本教程均已实现，见 [7.4](#74-倒计时可暂停关机暂停开机继续)、第八～十章。

## 附录 A：小米电视系统级折腾实录（去广告 / 换极简桌面 / 修信号源键）

> 本章是**电视本体**的优化记录（不是 HA 侧），与踩坑 16 / 17 / 18 / 19 互为印证。所有操作走 ADB，全部可逆。

### A.1 设备与结论速览

| 项 | 值 |
|---|---|
| 电视 | 小米电视 MiTV-ASTP0（mulan 固件，Amlogic 平台） |
| 网络 | **有线** `eth0`，IP `192.168.8.103/24`（旁路由 `192.168.192.11` 下的二级网段；主路由有该网段路由，局域网内可直连，不需要 zerotier） |
| 工具 | [Gathin23/cleanyourtv](https://github.com/Gathin23/cleanyourtv)（MIT，纯 Python ADB 客户端，免装 adb） |
| 去广告 | 26 个包 `pm disable-user` 禁用（可逆，非卸载） |
| 桌面 | **最终用 [Aerial](https://github.com/pengchujin/aerial-tv)（`app.orchard.tv`）**：靠无障碍守护 `HomeGuardService` 接管 Home 键（实体遥控按 Home → Aerial）；系统默认 Home 仍是旧桌面（冷开机显示小米原桌面）。早期试装的极简桌面 `com.gathin.tvhome` 已卸载（见 A.6） |
| 信号源键 | 仅 gathin 方案需要自修（见 A.5）；Aerial 方案不涉及此问题 |

### A.2 ADB 连接（mulan 固件特性，重要）

1. 电视：设置里打开「USB 调试」→ `adb connect 192.168.8.103:5555`；
2. **重启后网络 ADB 实测会自起**：`persist.adb.tcp.port=5555` 已固化（重启后仍在，2026-10-26 重启验证 ~10s 恢复连接）；若某次连不上再手动开「USB/网络调试」。旧笔记「重启必关 ADB」是基于未固化 persist 属性的情况，已不再成立。
3. 不想装 adb：用 cleanyourtv 的 `tools/tv.py`（纯 Python 实现 ADB 协议）；
4. **文件传输的坑**：这台电视 `adb pull` 报 `open failed`、往 `/sdcard` 写进去是 0 字节 → 实用解法是 `cat` + base64 分块读回。注意每个 64KB 块 base64 后自带 `==` 补位，**必须按行独立解码再拼字节**，直接拼字符串解码会错位。

### A.3 去广告（`pm disable-user`）

- **保留**：小爱语音助手、投屏（miplay / airkan / wfdsink / mi_connect）、米家、桌面、屏保、锁屏、输入法、`com.mitv.videoplayer`（本地/USB 播放引擎）、`com.mitv.tvplayer`（直播/信号源）等；
- **小米视频（`com.mitv.mivideoplayer`）已改为禁用**（2026-10-26）：用户不再使用（收费贵 + 应用内贴片广告），它是唯一残留广告源；禁用不影响投屏与本地 USB 播放（走 `com.mitv.videoplayer`）。如需恢复：`pm enable com.mitv.mivideoplayer`。
- **禁用**：统计 / analytics、商城 / 支付、tvpush 推送、天气、日历、卡拉OK、小米音乐、相册、云控、升级、短信推送等共 26 个；
- 脚本自带 **Home 归属保护**（候选名单里即使出现 `com.mitv.tvhome` 也不会真禁）；
- 随时可还原：`tools/restore.sh` + 定制清单，恢复记录在 `~/.cleanyourtv/recovery-<IP>-<id>.json`；
- ⚠️ 最大代价见踩坑 18：**别手动禁旧桌面**。

### A.4 换极简桌面（可选，附云端构建）

- 上游 launcher（`com.gathin.tvhome`）不提供预编译 APK，但仓库自带完整 gradle 工程 → **GitHub Actions 云端构建**（本机没有 JDK / Android SDK 时最省事）：建私库推 workflow，约 2 分钟出包；
- 新版 runner 的坑：`sdkmanager` 不在 PATH，`android-actions/setup-android@v3` 自身有 bug 会挂 → 直接用 runner 自带 `/usr/local/lib/android/sdk`；
- **CI 每次构建会生成新的 debug 密钥** → 覆盖安装必被拒（签名不匹配），必须先 `pm uninstall` 再装；
- 设默认 Home 的真相：`pm set-home-activity` 在这固件**写了但不生效**（旧桌面带 `HOME_STATE` 自定义权限压场，`am start` HOME 意图解析仍走旧桌面；`resolve-activity` 在这固件上也不读首选登记，别被它误导）；
- 最终采用的行为（零风险，推荐）：**按 Home → 旧桌面**；新桌面从应用列表打开，进去后按 Home 随时回去。

> ⚠️ **gathin 已被 Aerial 取代并卸载**（2026-10-26）：gathin 作为默认桌面在 mulan 上不生效（`set-home-activity` 写了不生效），且信号源键得自己打补丁；最终改投 **Aerial**（体验更完整、Home 接管更稳），详见 [A.6](#a6-最终桌面aerial取代-gathin)。

### A.5 信号源键修复（本次核心）

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

### A.6 最终桌面：Aerial（取代 gathin）

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

### A.7 收尾：去广告收口 + 免 ROOT 兜底拦截

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

### A.8 重启后拦截自动恢复：交给 Home Assistant 看门狗（2026-10-26 终版）

> ⛔ **同样已废弃（2026-10-27）**：看门狗自动化 `automation.dian_shi_yan_gao_lan_jie_shou_hu_personaldnsfilter_zi_hui_fu` 已删除，电视上也早已没有 `dnsfilter.android`，所以它一直显示 `unavailable`。本节仅作为「用 HA 盯电视端 ADB 状态」的实现范式保留——**里面的 ADB 集成接入方式、`adb_response` 回显拿法、`automation.reload` 热重载这些经验仍然有效**（信号源锁、App 锁都在用），只是不要再去拉起 dnsfilter。

> 痛点：personalDNSfilter 的 **VPN 在电视重启后不会自起**——小米系统级「自启动白名单」会杀它的 `BootUpReceiver`（监听 `BOOT_COMPLETED`），加上 Android 9 后台启动限制、本固件又无 `cmd vpn` 的 always-on 能力。结果：每次重启，拦截静默失效，要人手动进 App 点一下。
> 试过两条路，最终选 **HA**：

**① 电脑看门狗（否决）**：在 Windows 控制机放脚本每 5 分钟拉起过滤。用户明确「看门狗不能装电脑上」，弃。

**② Home Assistant 看门狗（采用）**：HA 常驻、且本就接管这台电视，最自然。

- **前置：重新加回 Android TV(ADB) 集成**。之前因为「重启 ADB 自动关闭」移除了 ADB 遥控器；现在 `persist.adb.tcp.port=5555` 已固化、ADB 重启自起，顾虑消除，遂在 HA 加回 Android TV 集成（host `192.168.8.103:5555`，device_class `androidtv`）。生成实体：
  - `media_player.android_tv_192_168_8_103`（friendly=小米电视）—— **所有 adb 命令都发给它**（看门狗、信号源锁都一样）；
  - `remote.android_tv_192_168_8_103` —— 当年移除的 ADB 遥控回来了，但它的状态常是 `unknown`，**不能用它发 adb 命令**（会静默不执行，见踩坑 21）。
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

## 更新日志

> 按日期倒序，只记「改了什么 + 为什么」，细节点进对应章节。

- **2026-10-27｜修「电视关机后还在计时」**：查出来云实体 **`state` 也会停更**（不只是 `source`），关机后长期报 `on`，导致今日观看分钟虚高到 392 分钟、关机暂停倒计时永不触发。把 4 条自动化的开关机判据统一换成 ADB 实体白名单（观看累计 / 关机暂停 / 开机继续 / 睡眠模式防偷看），触发改 template trigger。详见踩坑 23。
- **2026-10-27｜信号源锁 ADB 校验 30s → 10s**：切走后最多 10 秒拉回，消掉"切几次就锁不住"的观感。
- **2026-10-27｜删除广告拦截看门狗**：personalDNSfilter 方案终止（见 A.7 / A.8 顶部废弃说明），残留的 `unavailable` 自动化实体已从注册表清掉。
- **2026-10-27｜周报「本周累计」改为含今日**：`ben_zhou` 只含到昨天，面板上会出现「今日 385 > 本周 295」的怪数。
- **2026-10-27｜禁用名单补 CIBN 酷喵 `com.cibn.tv`**：共 7 个娱乐包（B站小电视 / 云视听极光 / QQ音乐 / 西瓜鲜时光 / 电视家 / 当贝市场 / 酷喵）。
- **2026-10-27｜新增 6.4「锁 App」**：只锁信号源锁不住内容 App，改为锁开时 `pm disable-user` 娱乐类包、锁关 `pm enable` 恢复，2 分钟对账兜底。
- **2026-10-26｜Aerial 接管桌面 + 系统包精简 28 个 + 去广告收口**（Aerial + 无障碍守护 + 包禁用清单）。

## Star 历史

如果这篇教程帮你省了折腾时间，欢迎点个 Star ⭐ 也欢迎 Issue 交流。
