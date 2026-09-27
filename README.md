# 电视「家长管控」教程：音量上限锁 + 信号源锁 + 儿童观看定时锁（Home Assistant）

> 不改电视系统、不装电视端 App，管控放在电视**外面**——孩子拿遥控器无解。
>
> 方案与品牌无关：小米、索尼、TCL、Apple TV、各类电视盒子……只要电视能接入 Home Assistant，这套教程就能直接套用。**本文以小米电视为例**，其他品牌只是接入方式不同（见「三、准备工作」对照表），自动化部分完全通用。

## 相关仓库（同一系列教程）

- [ha-midea-hualing-ac](https://github.com/cbywyz/ha-midea-hualing-ac) —— 美的/华凌空调接入 Home Assistant（云端集成教程+折腾实录）
- [ha-hualing-fan-broadlink](https://github.com/cbywyz/ha-hualing-fan-broadlink) —— 华凌风扇红外遥控接入 HA（Broadlink + 8 键编码库）
- [gree-yapqf-broadlink-smartir](https://github.com/cbywyz/gree-yapqf-broadlink-smartir) —— 格力空调 Broadlink+SmartIR 接入 HA
- [phicomm-aircat-m1](https://github.com/cbywyz/phicomm-aircat-m1) —— 斐讯悟空 M1 空气检测仪复活记（自建集成）

---

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
| 信号源 | 锁定开关打开**无条件强切** + 换台拦截 + 每分钟兜底 | 不信任电视上报的 source 属性（有缓存） |
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

### 6.2 换台拦截 + 每分钟兜底

锁定期间任何人切走信号源，电视状态一变就自动切回；再加每分钟兜底重发，双保险：

```yaml
- id: tv_source_lock
  alias: "[电视] 信号源锁定"
  description: 锁定开启时信号源被切走自动切回；云指令偶发限流，每分钟兜底重发直到切回。
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
      minutes: "/1"
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

## 七、第四步：儿童观看定时锁

玩法：给孩子填「儿童观看时长」= 30 → 30 分钟后自动上锁；「自动解除锁定时长」填 30 → 再过 30 分钟自动解锁（填 0 = 一直锁着，家长手动解）。**手动开的锁定永不被自动解除**。

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

## 八、面板卡片建议

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

## 九、踩坑记录（省你半天排错）

1. **云通道指令偶发丢失/限流**：走云的集成（如小米 Miot）`select_source`、`volume_set` 单发有时不生效。对策：动作里加 `continue_on_error: true`，关键切换 5 秒后补发一次，再加每分钟兜底。局域网直连的集成没有这个问题。
2. **`source` 属性有缓存，不可全信**：实际换了信号源，属性可能还报旧值。所以「锁定生效那一刻」必须无条件强切，不能先判断 `cur != want` 再切——否则永远不触发。这也是本教程 6.1 与 6.2 分成两条自动化的原因。
3. **音量锁防自触发**：拉回动作本身会再触发一次音量变化事件，条件里用 `cur - target > 0.005`（0.5% 死区）掐掉回弹，不然会无限循环拉回。
4. **锁定开关（input_boolean）本身不触发任何东西**：只开开关不切源是新手最常见的坑，务必有一条「开关打开 → 执行切换」的自动化。
5. **手动锁与倒计时锁要区分**：用一个内部标记 `input_boolean`（7.2 里第 2 个动作）标记"这次是定时上的锁"，自动解锁前检查它——手动上的锁就永远不会被定时器偷偷解掉。

## 十、可以继续扩展的方向

- **定时到点发通知**：上锁/解锁时接上手机或 IM 推送，家长远程知情；
- **每天累计观看时长**：用 `history_stats` 统计当天电视开启时长，超了自动上锁；
- **时段白名单**：只在某些时段允许解锁（比如作业写完之前锁定不可解除）；
- **锁面板本身**：HA 的面板编辑权限可以按用户控制，孩子登录的账号只读即可。

## Star 历史

如果这篇教程帮你省了折腾时间，欢迎点个 Star ⭐ 也欢迎 Issue 交流。
