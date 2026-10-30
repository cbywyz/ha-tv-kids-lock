# 踩坑记录（23 条，省你半天排错）

> 本文件从 README 拆出，专放排错经验。正文教程在 [../README.md](../README.md)。
> 标 ⛔ 的是「红线」——踩过会出事，照着做之前先看。

## 踩坑 1：云通道指令偶发丢失 / 限流

走云的集成（如小米 Miot）`select_source`、`volume_set` 单发有时不生效。对策：动作里加 `continue_on_error: true`，关键切换 5 秒后补发一次，再加 5 分钟兜底（别用每分钟，见踩坑 15）。局域网直连的集成没有这个问题。

## 踩坑 2：source 属性有缓存，不可全信

实际换了信号源，属性可能还报旧值。所以「锁定生效那一刻」必须**无条件强切**，不能先判断 `cur != want` 再切——否则永远不触发。这也是 README 里「信号源锁定」拆成「立即强切」+「换台拦截」两条自动化的原因。

## 踩坑 3：音量锁防自触发

拉回动作本身会再触发一次音量变化事件，条件里用 `cur - target > 0.005`（0.5% 死区）掐掉回弹，不然会无限循环拉回。

## 踩坑 4：锁定开关（input_boolean）本身不触发任何东西

只开开关不切源是新手最常见的坑，务必有一条「开关打开 → 执行切换」的自动化。

## 踩坑 5：手动锁与倒计时锁要区分

用一个内部标记 `input_boolean`（README「到点自动上锁」里第 2 个动作）标记"这次是定时上的锁"，自动解锁前检查它——手动上的锁就永远不会被定时器偷偷解掉。

## 踩坑 6：Jinja 里 `{{ set }}` 是错的，必须 `{% set %}`

写错的那条自动化会被判为 invalid **并自动 disabled**（状态变 `unavailable`），界面上不报错，只能从 `home-assistant.log` 里翻。改完 YAML 先本地搜一遍有没有 `{{ set`。

## 踩坑 7：自动化被自动禁用后，改 id 重装会留 residual 实体

旧的 `automation.xxx` 会一直是 `unavailable`，要重启 HA 才清得掉（或用 WS `config/entity_registry/remove`）。所以改自动化时尽量保持 id 不变。

## 踩坑 8：sensor 域（history_stats、template sensor 里的部分平台）没有 reload 服务

这是本教程坚持用 `input_number` 做累加器的原因——能热重载的才是家庭里敢动的东西。

## 踩坑 9：timer.start 带不带 duration 是两种完全不同的语义

带 = 重新开始计时；不带 = 从 `paused` 的剩余时间继续。做暂停功能时千万别顺手补上 duration。详见 README「倒计时可暂停」。

## 踩坑 10：timer.pause / timer.start 对状态不符的 timer 会报错

`idle` 的 timer 调 `pause` 会在日志里刷红。动作外面套一层 `condition: state`（暂停只看 `active`、继续只看 `paused`）就干净了。

## 踩坑 11：区分"该暂停的时间"和"不该暂停的时间"

观看类倒计时可以随关机暂停，但上锁后的冷却 / 惩罚时间必须按墙钟走——否则孩子关个电视就能把惩罚无限期冻结。

## 踩坑 12：Bubble Card 的弹窗 card_type 是 pop-up（带连字符）

写成 `popup` 不报错，就是点了没反应，非常隐蔽。另外触发弹窗别用 bubble 自带的 button 卡（部分版本点了没反应），用普通卡片 `tap_action: navigate` 到 `#hash` 最稳（见 README 面板章的「使用说明弹窗」）。

## 踩坑 13：信号源锁锁不住「电视内 App」（云视听 / 爱奇艺等）—— 机制盲区

锁只认 `media_player` 的 `source` 属性（物理信号源 HDMI / TV）。孩子在电视桌面里打开云视听这类内容 App，**不会切换信号源**，`source` 属性不变 → HA 判定"还在锁定的 HDMI 1"，不触发切回。所以单靠信号源锁**只能锁物理信号源（机顶盒 / 游戏机所在的 HDMI 口），锁不住电视自带的流媒体 App**。

要在 App 层也管住，得另走 ADB / 应用层方案：**锁开时 `pm disable-user` 娱乐 App**，详见 README「锁 App」一节。

## 踩坑 14：小米 Miot 云端通道极不稳定

「切回」依赖它，断线锁就暂时失效：实测一晚断 3 次（其中一次长达 85 分钟零上报）。家里局域网 ping 电视 0 丢包、0.36ms，排除了自家网络——断的是小米云。后果：云端断开期间孩子切走信号源，HA 既收不到变化、也下不了切回指令，锁等于暂时失效。**局域网直连的电视（索尼 / TCL / 雷鸟等走 Android TV 集成）没有这问题**，这是"走云"必须接受的代价。要缓解，只能在锁定时加飞书告警（失联即通知家长），而不是硬扛云端。

## 踩坑 15：⛔ 红线：别对小米云实体做 time_pattern 高频 select_source（会打爆电视）

为补第 13 条 App 盲区的漏，我曾加"每分钟无条件切回 HDMI 1"。结果：每分钟一条云端指令把小米电视（经 xiaomi_home 云端通道）打爆 → **遥控器失灵、电视卡死**。实测当时手动 `select_source(HDMI 2)` 下发返回 200，电视毫无反应，HA 侧 `source` 110+ 秒不更新 → 云端通道**双向皆断**（状态不上报、命令也不执行），断线时还每分钟狂发指令 = 雪上加霜。**真要强制踢回**：必须前置"电视 `last_updated` < 2 分钟才发"的门禁（失联时不发），且间隔 ≥ 5 分钟。

## 踩坑 16：ADB 补 App 盲区的方案，当年实测后放弃（后已重新启用）

主路由常驻 ADB 守护读前台 App，违规就发 webhook 让 HA 切回——思路通，但**当年小米电视一重启「网络调试」就自动关闭**，守护随之失效，遂撤掉。后来把端口写进持久属性（`setprop persist.adb.tcp.port 5555`）并确认系统设置里调试常开，重启后 `5555` 仍在监听，才重新启用。**用前必须确认三件事**：① `getprop persist.adb.tcp.port` 有值；② `settings get global adb_enabled` 为 1；③ 真重启一次，`adb connect` 仍然成功。三者缺一，ADB 守护迟早会在某次重启后静默失效。

## 踩坑 17：（定位用）电视真实 IP 可能在旁路由 / 二级网段，别认错设备

我家电电视真实 IP 是 `192.168.8.103`（在旁路由下的二级网段），而 `192.168.3.191` 是**另一台**小米设备。做 ADB 调试或写 IP 白名单前，先用 `adb connect <真IP>:5555` 确认连的是电视本身，别把指令发到别人设备上（同理，HA 里同名实体也要核对 `entity_id`，别张冠李戴）。

## 踩坑 18：⛔ 红线：禁用旧桌面（com.mitv.tvhome）会连带弄丢遥控「信号源 / Input」键

mulan 固件的 Input 键（`KEYCODE_TV_INPUT=178`）是**前台应用**接管的，旧桌面是唯一接管方；而 Google TV 那套 `com.android.tv.action.VIEW_INPUTS` 在这固件上**没有接收方**，真正的信号源面板藏在 `com.droidlogic.tvinput/.shortcut.ShortCutActivity`（Amlogic 输入框架）里。给电视做任何「系统级优化」（去广告 / 换桌面）前，**先确认旧桌面状态**——本次实测：禁用它 → 信号源键失灵；恢复启用 → 立即复原。要换桌面就得自己补这个键（做法见[系统折腾文档 A.5](xiaomi-system-tweaks.md#a5-信号源键修复本次核心)）。

## 踩坑 19：小米电视（mulan）走的是有线网卡，dumpsys wifi 显示 DISCONNECTED 是正常现象

这台机器根本没有 `wlan0`，网络走 `eth0`。排障时看到「Wi-Fi 已断开」别顺着查网络——`ip addr` 一眼就能确认。同理 ADB 偶发超时也未必是故障，ping 通 + 5555 重连即可，多为瞬时抖动。

## 踩坑 20：云 source 属性会停更，这才是「切几次就锁不住」的真正原因

实测米家云实体 `media_player.xiaomi_cn_...` 的 `last_updated` 停在几个小时前，期间电视切 App、切 HDMI，`source` 一直报旧值。而锁的唯一判据就是这个属性 → 属性不动，自动化永远不触发。表现极具迷惑性：**刚上锁时有效，切两三次之后就怎么切都不管**。

修法见 README「ADB 增强校验」——用 `androidtv` 集成走局域网 ADB，每 10 秒读 `dumpsys activity activities`（前台是不是 `ExternalSourceActivity`）+ `settings get global mitv.tvplayer.hdmi.last.source`（在哪个 HDMI），电视自己判断、自己切回；云通道降为 5 分钟兜底。**别再靠提高云指令频率去救，那只会打爆通道（踩坑 15）。**

## 踩坑 21：⛔ androidtv.adb_command 的 entity_id 必须写 media_player，写 remote.* 会「静默不执行」

androidtv 集成会同时注册 `media_player.android_tv_<ip>`（真正带 ADB 通道的那个）和 `remote.*`。把 `adb_command` 指向 `remote.*` 时：服务调用**返回 success**、自动化 `last_triggered` 照常更新，但电视端什么都没发生——只在 `home-assistant.log` 留一句 `Referenced entities remote.xxx are missing or not currently available`（因为该 remote 实体状态是 `unknown`，服务调用被直接跳过）。排查时极易误判成"电视 ADB 掉线"。**验收方法**：看目标 `media_player` 的 `adb_response` 属性有没有电视返回的文本。

## 踩坑 22：一次只能连一个 adb 客户端，电脑连着 HA 就连不上（反之亦然）

小电视的 adbd 是单会话的。电脑 `adb connect` 上去之后，HA 的 androidtv 立刻刷 `Connection reset by peer`，实体变 `unknown`、所有 ADB 自动化空转。**排障或抓 dumpsys 用完立刻 `adb kill-server`**，别让电脑的 adb 常驻；HA 侧重连一般 10 秒左右自愈，不行就 `homeassistant.reload_config_entry`（androidtv 条目）重载一次。

## 踩坑 23：云实体的 state 也会停更，别拿它判断开关机

一直以为云实体只有 `source` 会停更、`on/off` 是准的——**错**。实测电视关机后 `media_player.xiaomi_cn_...` 仍长期停在 `on`（`last_updated` 停在几小时前不动）。后果两个：①「今日观看分钟累计」在电视关机后继续每分钟 +1，一天下来虚高到 385 分钟；②「关机暂停观看倒计时 / 开机继续」永远不触发，锁定倒计时和续杯倒计时关机后照常空跑。

修法：**所有开关机判据统一改成 androidtv 实体** `media_player.android_tv_<ip>` 的状态白名单 `['playing','paused','idle','on']`——正常开机是 `playing/paused/idle`（实测待机桌面 = `paused`），关机是 `off/standby`，ADB 断了是 `unavailable`，后三种都落在白名单外。**触发要用 template trigger，`to: 'off'` 抓不到 `unavailable`。**

顺带一个显示坑：「本周累计」比「今日已看」还小，不是数据坏了——`ben_zhou` 只累计到昨天，0:01 才把今天并进去。周报文案要算 `ben_zhou + jin_ri`，别只报 `ben_zhou`。

这个白名单不是拍脑袋定的，有源码依据：`site-packages/androidtv/androidtv/base_androidtv.py` 里 `screen_on is None` → state 为 None（HA 显示 `unavailable`）、`not screen_on` → `STATE_OFF`、`not awake` → `STATE_STANDBY`，只有「屏幕亮且已唤醒」才可能落到 `playing/paused/idle`。所以三种非开机态全在白名单外，**关机必然停表**。
