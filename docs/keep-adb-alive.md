# 让小米电视的 ADB 一直可用（重启不丢 / 运行时常开）

> 本文是「ADB 增强」路径的配套文档。基础三道锁不依赖 ADB 也能用；但**信号源锁的秒级 ADB 校验**和**锁 App（pm disable-user）**都靠电视 ADB 在线——一旦 ADB 掉线，这两层就静默失效。所以启用它们前，先把「ADB 一直可用」这件事钉死。
>
> 正文教程见 [../README.md](../README.md) 第五章；ADB 通道的接入与实体命名见[系统折腾文档 A.8](xiaomi-system-tweaks.md#a8-重启后拦截自动恢复交给-home-assistant-看门狗已废弃)。

---

## 为什么这件事值得单独讲

- 小米**云端实体**（`media_player.xiaomi_cn_*`）的 `source` / `state` 会**停更**（[踩坑 14 / 20 / 23](troubleshooting.md)），信号源锁不能只靠云通道，得用局域网 ADB 每 10 秒读电视真实状态。
- 旧版经验是「**重启必关 ADB**」，导致早年放弃了 ADB 守护。后来把端口固化进持久属性，重启才自起——所以**方法对不对，一定要真重启验证**，别想当然。

---

## 一、固化端口（重启后自动开 ADB）

电视上（或 HA 的 `androidtv.adb_command` 里）执行：

```bash
# 固化 TCP 调试端口，写进 persist 属性（重启不丢）
setprop persist.adb.tcp.port 5555
# 确保全局调试开关是开的
settings put global adb_enabled 1
```

- 端口固化后，**真重启电视，`adb connect <IP>:5555` 约 10 秒自动恢复**（2026-10-26 实测）。
- 旧的「重启必关 ADB」结论，是基于**没固化 persist 属性**的年代，现已不成立。

---

## 二、启用前的三步验收（必做）

任一缺失，ADB 守护迟早会在某次重启后**静默失效**——届时锁还在跑，但 ADB 那层早已失效，你却看不出异常。

```bash
# ① 持久端口确实有值
getprop persist.adb.tcp.port          # 期望：5555
# ② 全局调试开关是 1
settings get global adb_enabled       # 期望：1
# ③ 真·重启一次，再连
adb connect <IP>:5555                 # 重启后仍能成功
```

> 这台机器走**有线网卡 `eth0`**（没有 `wlan0`），ADB 走有线 5555 即可；排障看到 `dumpsys wifi` 显示 DISCONNECTED 是正常现象（[踩坑 19](troubleshooting.md)），别顺着查 Wi-Fi。

---

## 三、运行时常开（不只是重启）

- **已装 Aerial 桌面**：它的无障碍守护 `HomeGuardService` 会**定期发 `keep_adb` 广播保持网络 ADB 常开**，无需你额外操作。Aerial 开机自启、进程自动拉起，所以「运行时常开」在装了 Aerial 后自动成立（详见[系统折腾文档 A.6](xiaomi-system-tweaks.md#a6-最终桌面aerial取代-gathin)）。
- **没装 Aerial**：仅靠第二步的 persist 属性 + `adb_enabled=1`，在 mulan 固件上运行期 ADB 也保持监听；若发现待机/长时间空闲后偶发连不上，可让 HA 用一条 `time_pattern` 自动化周期性下发 `settings put global adb_enabled 1` 或重连作兜底（按需）。

---

## 四、HA 侧接入与单会话陷阱

### 加回 Android TV(ADB) 集成

host 填 `192.168.8.103:5555`，device_class 选 `androidtv`。生成两个实体：

| 实体 | 用途 |
|------|------|
| `media_player.android_tv_192_168_8_103` | **所有 adb 命令都发给它**（信号源锁、App 锁、守护都一样） |
| `remote.android_tv_192_168_8_103` | 当年移除的 ADB 遥控回来了，但状态常是 `unknown`，**别用它发 adb 命令**（会静默不执行，[踩坑 21](troubleshooting.md)） |

- 首次连接电视会弹「允许 USB 调试吗」，**用遥控器点「确定」**一次性授权（新主机密钥，和电脑 adb 不同）。
- 命令回显读目标 `media_player` 的 **`adb_response` 属性**（[踩坑 21 / A.8](troubleshooting.md)），不必再写「自包含黑盒命令」。

### ⚠️ 单会话陷阱（最容易踩）

小电视的 `adbd` 是**单会话**的（[踩坑 22](troubleshooting.md)）：

- 电脑 `adb connect` 上去后，HA 的 androidtv 立刻 `Connection reset by peer`，实体变 `unknown`、所有 ADB 自动化空转。
- **排障或抓 `dumpsys` 用完，立刻 `adb kill-server`**，别让电脑的 adb 常驻。
- HA 侧重连一般 10 秒左右自愈；不行就 `homeassistant.reload_config_entry`（androidtv 条目）重载一次。

---

## 五、一句话 checklist

启用信号源锁 ADB 校验 / 锁 App 前，确认：

- [ ] `persist.adb.tcp.port=5555` 已固化
- [ ] `settings get global adb_enabled` = 1
- [ ] 真重启后 `adb connect` 仍成功
- [ ] HA 的 Android TV 集成指向 `media_player.android_tv_*`（不是 `remote.*`）
- [ ] 电脑排障后已 `adb kill-server`，没占着单会话
