# MineAgent

[![License: CC BY-NC-SA 4.0](https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-lightgrey.svg)](LICENSE)

用 LLM 玩 Minecraft 的一套方案：**三个客户端模组 + 一份写给模型的操作手册**（再配上 Baritone 更省事）。

不需要模拟器，也不需要训练视觉模型——游戏里挂上这些模组，把「操作」「状态」「合成」分别变成
HTTP 接口和聊天指令，LLM 只要会发请求，就能从空手一路玩到击败末影龙。

## 组成

| 仓库 | 必需性 | 接口 | 作用 |
| --- | --- | --- | --- |
| [**mcctl**](https://github.com/MineAgent/mcctl) | **必需** | `127.0.0.1:3420` | **身体**：按键 / 鼠标 / 视角 / 滚轮 / Baritone / 聊天，另有 `GET /prtsc` 截图、`GET /mods` 模组列表、`GET /mouse` 光标位置（配合 `mouse goto` 绝对定位） |
| [**AdvancedInfoFetcher**](https://github.com/MineAgent/AdvancedInfoFetcher) | 可选 | `127.0.0.1:3421` | **眼睛+耳朵**：`GET /info` 坐标 / 朝向 / 生命 / 饱食 / 状态效果，`GET /inventory` 背包 / 副手 / 盔甲 / 熔炉 / 箱子，`GET /world` 维度 / 时间 / 天气，`GET /msg` 聊天栏回显，`GET /sound` 播放过的声音 ID（`GET /keysnd` 只看重要声音） |
| [**cmdCraft**](https://github.com/MineAgent/cmdCraft) | 可选 | 聊天指令 | **手**：`/craft` 合成、`/inventory` 换快捷栏、`/furnace` 冶炼、`/chest` 存取箱子 |
| [**Baritone**](https://github.com/cabaletta/baritone) | 可选 | `bt` 命令 | **腿**：寻路与自动挖矿 |
| [**playbook.md**](playbook.md) | — | — | 给模型看的操作手册：接口、实测结论、坐标、流程、坑 |

**mcctl 是唯一的硬需求**——没有它就没有任何接口可用。其余三个都是可选增强，但实际游玩时一般都会装上：
没有 AdvancedInfoFetcher 就只能靠截图猜背包，没有 cmdCraft 就得去点 GUI 格子（可以用
`GET :3420/mouse` + `mouse goto` 兜底，但仍然容易点错），没有 Baritone 就只能一步步按键走路。

全部是**客户端**模组，服务端不需要装任何东西，可以在原版 / Fabric / Paper 服务器上用。

## 它是怎么玩的

```
        ┌────────────────────────────────┐
        │  GET :3421/info                │  我在哪、面朝哪、还剩多少血
        │  GET :3421/inventory           │  背包 / 副手 / 盔甲 / 熔炉 / 箱子
        │  GET :3421/world               │  维度 / 时间 / 天气
        │  GET :3421/msg                 │  聊天栏回显：指令输出 / Baritone / 报错
        │  GET :3421/sound  /keysnd      │  播放过的声音 ID（/keysnd 只看重要的）
        └───────────────┬────────────────┘
                        │ ① 观察
                        ▼
                  ┌───────────┐
                  │    LLM    │  ② 决定下一步
                  └─────┬─────┘
                        │ ③ 执行
        ┌───────────────▼────────────────┐
        │  POST :3420  'bt mine iron_ore'     腿
        │  POST :3420  'chat /craft ...'      手
        │  POST :3420  'W 500' / 'mouse left' 身体
        └───────────────┬────────────────┘
                        │ ④ 校验
                        ▼
              GET :3421/msg 读回显 + /keysnd 听动静 + GET :3420/prtsc 看一眼画面，回到 ①
```

实际的请求：

```bash
curl -s http://127.0.0.1:3421/info                                        # 坐标/朝向/生命
curl -s http://127.0.0.1:3421/inventory                                   # 背包/副手/盔甲/熔炉/箱子
curl -s http://127.0.0.1:3421/world                                       # 维度/时间/天气
curl -s http://127.0.0.1:3421/msg                                         # 上次读之后的聊天回显
curl -s http://127.0.0.1:3421/keysnd                                      # 上次读之后的重要声音
curl -s -X POST --data-binary 'bt mine iron_ore'          http://127.0.0.1:3420
curl -s -X POST --data-binary 'chat /craft iron_pickaxe'  http://127.0.0.1:3420
curl -s -X POST --data-binary 'chat /inventory torch 2'   http://127.0.0.1:3420
curl -s -o shot.png http://127.0.0.1:3420/prtsc
```

## 各自解决什么问题

* **mcctl = 身体（必需）**：把游戏变成可编程接口。输入走原版管线（`KeyMapping` / `Screen` 事件），
  视角旋转会正常同步给服务器，不抢占真实键鼠。只装它一个，LLM 就已经能玩了——只是玩得很笨。
* **AdvancedInfoFetcher = 眼睛 + 耳朵（可选）**：LLM 不需要看懂画面，坐标、朝向、血量、背包、熔炉燃烧进度全部是纯文本；
  `GET /msg` 还把聊天栏回显也变成纯文本——指令报错、Baritone 的输出不用再靠截图去认；
  `GET /sound` / `GET /keysnd` 更进一步，把播放过的声音 ID（破坏方块、怪物、爆炸；`/keysnd` 已滤掉脚步/音乐/环境音/UI/天气）也变成文本，
  "刚才发生了什么"不用截图就能判断；`GET /world` 则给出维度、时间和天气。
* **cmdCraft = 手（可选）**：GUI 对 LLM 很不友好——点击有延迟、容易点错格子（mcctl 1.5.0 起虽然能读光标位置、
  也能 `mouse goto` 绝对定位，但手点 GUI 依然只是**没有 cmdCraft 时的兜底**，不推荐）。
  所以把合成、冶炼、开箱子这些高频动作变成指令，直接发 `ServerboundContainerClickPacket`，
  和玩家亲手点格子完全等价，服务端照常校验。
* **Baritone = 腿（可选）**：`bt mine` / `bt goto` 负责寻路和挖矿，比逐步按键高效得多。

## 快速开始

```bash
# 1. 需要 Minecraft 26.2 + Fabric Loader 0.19.5+
# 2. 必需的：mcctl
cp mcctl-*.jar ~/.minecraft/mods/
# 3. 强烈建议一起装的三个（可选，但少了会难受）
cp advanced-info-fetch-*.jar craftcmd-*.jar ~/.minecraft/mods/   # Baritone 另见其仓库
# 4. 启动游戏、进入存档，然后：
curl http://127.0.0.1:3420/        # mcctl 使用说明
curl http://127.0.0.1:3420/mods    # 确认模组都加载了
curl http://127.0.0.1:3421/info    # 玩家状态（装了 AdvancedInfoFetcher 才有）
```

## 手册

[`playbook.md`](playbook.md) 是真正交给模型的那份文档，只写「正式游玩时该怎么做」：

* 两个端口（3420 控制 / 3421 信息）的接口与字段（`GET :3420/`、`GET :3421/` 全文的精简版）
* **状态与回显**：`/info`、`/inventory`、`/world` 的字段含义，`/msg` 的增量聊天回显
  （指令输出 / Baritone / 报错，读取即清空），以及 `/sound`·`/keysnd` 的增量声音回显
  （`<声音ID> <音量> <音高>`，每播放一次一行；`/keysnd` 与 `/sound` 共用队列，只给重要声音）
* **输入通道实测结论**：`E` 开背包、`1`-`9` 切格、`Q` 丢弃都通过 HTTP 有效；
  发文本一律走 `chat <文本>` / `bt <命令>`，不通过聊天框打字；
  聊天框开着时 `E`/`Q`/`1`-`9` 会自动先关掉它，`W` 等移动键不受影响（详见手册 §4）
* `cmdCraft` 四条指令的前置条件、参数与常用物品 id
* 「观察 → 决策 → 执行 → 校验」的循环节奏，以及截图延迟、对账、记坐标这些注意事项
* 固定分辨率下的坐标表（仅在**没有 cmdCraft**、必须手点时兜底：先 `GET :3420/mouse` 读光标，再 `mouse goto`）
* 指定种子与那座已激活的末地传送门，以及从空手到末影龙的路线

## 已验证

在真实客户端（Minecraft 26.2 + Fabric + Baritone）里跑通过，**平台为 Linux + X11/Xwayland**：

> **Linux 上的 Minecraft 永远跑在 X11/Xwayland 下**：26.2 的 `GLX` 里写死了
> `glfwInitHint(GLFW_PLATFORM, GLFW_PLATFORM_X11)`，会话是 Wayland 也一样（走 Xwayland）。
> 原生 Wayland 只存在于 `-DMC_DEBUG_ENABLED=true -DMC_DEBUG_PREFER_WAYLAND=true` 这个调试开关下，
> 而实测这么开之后 **MC 自己会卡死在 `glfwSwapBuffers`**（打开存档时冻结、客户端线程不再响应，
> 也不写崩溃报告）。所以实际部署只有 X11/Xwayland 一条路径，Wayland 那段留档备查。
> 光标相关的行为（`GET :3420/mouse` 的「不在窗口内」判断、`mouse goto` 的绝对定位）**只在 Linux 上实测过**；
> Windows、macOS 没有实机验证——接口本身是 GLFW 的跨平台接口，代码里没有平台分支，
> 但换平台后建议先自测一遍 `/mouse` → `mouse goto` → `mouse left`。

| 能力 | 结果 |
| --- | --- |
| 控制接口 | 移动 / 转向 / 视角 / 组合键，截图对比确认生效 |
| 光标接口 | `GET :3420/mouse` 在暂停菜单给出 `光标：427.0 240.0`、`抓取：否`、`界面：PauseScreen`；`mouse goto 321 202` + `mouse left` 点中「进度」按钮 |
| 光标在窗口外 | 把指针移到窗口外后 `/mouse` 输出 `光标：不在窗口内，请使用 mouse goto <x> <y>`（用跨平台的 `GLFW_HOVERED` 判断）；照常 `mouse goto` + `mouse left` 仍能点中，且物理指针被拉回窗口内 |
| 状态接口 | `/info` 的坐标、朝向、选中格与游戏内一致 |
| 世界接口 | `/world` 的维度、时间、天数、游戏刻和天气跟随游戏（`/time set` 后 `时间` 立即变化） |
| 聊天回显 | `/msg` 增量返回玩家聊天、指令输出、Baritone 输出与报错，逐条与客户端日志的 `[CHAT]` 一致，读完再次请求为空 |
| 声音回显 | `/sound` 增量返回播出音效的命名空间 ID、请求音量与音高；挖石头得到 `minecraft:block.stone.break`，未知/被静音跳过的音效不出现 |
| 重要声音 | `/keysnd` 过滤掉脚步/音乐/ambient/ui/天气，只留破坏方块、怪物、爆炸等；与 `/sound` 共用一个队列 |
| Baritone | `bt mine oak_log`、`bt mine stone`、`bt mine iron_ore` 自动寻路挖掘并拾取（实测把玩家从 y=48 带到 y=18） |
| 合成 | 原木 → 木板 → 工作台 → 木镐 → 石镐 |
| 冶炼 | `/furnace put raw raw_iron`、`put fuel`、`get product` 取出铁锭 |
| 端到端 | 空手 → 原木 → 工作台 → 木镐 → 石镐 → 熔炉 → 铁矿 → 铁锭 → **铁镐**，全程只用 HTTP（`E` 开背包 + `/craft`，不点 GUI） |

目标是走完整条链：**挖矿 → 铁器 → 末地 → 末影龙。**

手册针对一个固定世界：种子 **`-818810680825819701`**，并且在 **1697 -2 1124** 有一座**已经激活的末地传送门**。
所以不必去下界打烈焰人，也不必攒末影珍珠和末影之眼——拿到基础装备后直接赶过去就能进末地。

## 本仓库

```
README.md      本文件
playbook.md    给模型看的操作手册
LICENSE        CC BY-NC-SA 4.0
```

只放文档，不放代码。

## 许可证

本仓库的文档采用 **[CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/)**
（署名—非商业性使用—相同方式共享 4.0 国际），完整文本见 [`LICENSE`](LICENSE)：
可以自由复制、分发、改编，但必须署名、不得用于商业用途、且衍生作品需以相同许可分发。

文档里提到的三个模组是代码，仍按各自的许可分发——
[mcctl](https://github.com/MineAgent/mcctl)、[AdvancedInfoFetcher](https://github.com/MineAgent/AdvancedInfoFetcher)
与 [cmdCraft](https://github.com/MineAgent/cmdCraft) 均为 **LGPL-3.0-only**。
