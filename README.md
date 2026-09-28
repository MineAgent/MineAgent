# MineAgent

[![License: CC BY-NC-SA 4.0](https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-lightgrey.svg)](LICENSE)

用 LLM 玩 Minecraft 的一套方案：**三个客户端模组 + 一个服务端模组 + 一份写给模型的操作手册**（再配上 Baritone 更省事）。

不需要模拟器，也不需要训练视觉模型——游戏里挂上这些模组，把「操作」「状态」「合成」分别变成
`/ctl`、`/aif`、`/op` 三组 HTTP 接口，LLM 只要会发请求，就能从空手一路玩到击败末影龙。

## 组成

| 仓库 | 必需性 | 接口 | 作用 |
| --- | --- | --- | --- |
| [**MGHttpdProvider**](https://github.com/MineAgent/HttpdProvider) | **必需** | `127.0.0.1:3420` | **插座**：共享 HTTP 服务，各模组挂在它的前缀下；`GET /` 列出当前可用的 endpoint |
| [**mcctl**](https://github.com/MineAgent/mcctl) | **必需** | `127.0.0.1:3420/ctl` | **身体**：按键 / 鼠标 / 视角 / 滚轮 / Baritone / 聊天，另有 `GET /ctl/prtsc` 截图、`GET /ctl/mouse` 光标位置（配合 `mouse goto` 绝对定位） |
| [**AdvancedInfoFetcher**](https://github.com/MineAgent/AdvancedInfoFetcher) | 可选 | `127.0.0.1:3420/aif` | **眼睛+耳朵**：`GET /aif/info` 坐标 / 朝向 / 生命 / 饱食 / 状态效果，`GET /aif/inventory` 背包 / 副手 / 盔甲 / 熔炉 / 箱子，`GET /aif/world` 维度 / 时间 / 天气，`GET /aif/msg` 聊天栏回显，`GET /aif/sound` 播放过的声音 ID（`GET /aif/keysnd` 只看重要声音） |
| [**cmdCraft**](https://github.com/MineAgent/cmdCraft) | 可选 | `127.0.0.1:3420/op` | **手**：一条 `POST /op` 下辖 `craft` 合成、`inventory` 换快捷栏、`furnace` 冶炼、`chest` 存取箱子、`look` 转视角 |
| [**unlockRecipe**](https://github.com/MineAgent/unlockRecipe) | 可选（用 cmdCraft 时建议装） | —（纯服务端，无 HTTP 接口） | **开局配方书**：谁加入世界就替谁执行 `/recipe give <玩家> *`，让 cmdCraft 从第一秒起就有整本配方可用（见下文） |
| [**Baritone**](https://github.com/cabaletta/baritone) | 可选 | `bt` 命令 | **腿**：寻路与自动挖矿 |
| [**playbook.md**](playbook.md) | — | — | 给模型看的操作手册：接口、实测结论、坐标、流程、坑 |

**MGHttpdProvider + mcctl 是硬需求**——前者提供 3420 端口，后者是身体；没有它们就没有任何接口可用。其余几个都是可选增强，但实际游玩时一般都会装上：
没有 AdvancedInfoFetcher 就只能靠截图猜背包，没有 cmdCraft 就得去点 GUI 格子（可以用
`GET :3420/ctl/mouse` + `mouse goto` 兜底，但仍然容易点错），没有 Baritone 就只能一步步按键走路。

除了 unlockRecipe，其余模组都是**客户端**模组，服务端不需要装任何东西，可以在原版 / Fabric / Paper 服务器上用；
[**unlockRecipe**](https://github.com/MineAgent/unlockRecipe) 则是**唯一的服务端模组**（Fabric），
它只解决一件事：让新存档 / 新玩家的配方书一进来就是满的——原因见下一节。

## unlockRecipe：给 cmdCraft 补上「配方书」

cmdCraft 的 `craft` 走的是**客户端配方书**：只有已经解锁的配方才能合成，找不到就返回
`400 找不到可以合成 <物品> 的配方（配方未解锁或不存在）`，什么都不做。
而原版是「拿到材料才解锁配方」，所以**每开一个新存档，配方书都要从头攒**——
对 LLM 来说，开局那一串 `craft`（木板、工作台、木棍、熔炉、铁镐……）只能等材料凑齐才逐个解锁，
前面几步经常直接卡在那里。

原来的绕法是**非常规手段**：把存档「对局域网开放」并允许其它玩家使用作弊命令（或者建存档时直接勾上允许作弊），
再用**另一个账号**进房间执行 `/recipe give <玩家> *`——单人存档里房主自己不一定有 OP，
所以还得有第二个号来发这条命令；每次新建存档都要重来一遍。

[**unlockRecipe**](https://github.com/MineAgent/unlockRecipe) 就是把这段补上：这是一个
**Fabric 服务端模组**，**只要有人加入世界，就用控制台身份（权限等级 4）替他执行一次
`/recipe give <玩家> *`**，不记录谁来过、每次加入都执行。

* 存档第一秒起配方书就是满的（实测 1561/1585，剩下 24 个是配方书本来就不收的特殊配方），
  cmdCraft 的 `craft` 立刻可用——**不用开作弊、不用第二个账号、不用打开局域网**。
* 单人存档、局域网世界、独立 Fabric 服务端都能用；客户端连别人的服务器时它什么也不做。
* 服务端不是 Fabric（比如 Paper）时装不了它，那就仍然得用上面的非常规手段，
  或者按原版节奏自己解锁配方，再不然只能做已经解锁的那些。

## 它是怎么玩的

```
        ┌────────────────────────────────┐
        │  GET :3420/aif/info                │  我在哪、面朝哪、还剩多少血
        │  GET :3420/aif/inventory           │  背包 / 副手 / 盔甲 / 熔炉 / 箱子
        │  GET :3420/aif/world               │  维度 / 时间 / 天气
        │  GET :3420/aif/msg                 │  聊天栏回显：指令输出 / Baritone / 报错
        │  GET :3420/aif/sound  /keysnd      │  播放过的声音 ID（/keysnd 只看重要的）
        └───────────────┬────────────────┘
                        │ ① 观察
                        ▼
                  ┌───────────┐
                  │    LLM    │  ② 决定下一步
                  └─────┬─────┘
                        │ ③ 执行
        ┌───────────────▼────────────────┐
        │  POST :3420/ctl  'bt mine iron_ore'     腿
        │  POST :3420/op   'craft iron_pickaxe'   手
        │  POST :3420/ctl  'W 500' / 'mouse left' 身体
        └───────────────┬────────────────┘
                        │ ④ 校验
                        ▼
              GET :3420/aif/msg 读回显 + /keysnd 听动静 + GET :3420/ctl/prtsc 看一眼画面，回到 ①
```

实际的请求：

```bash
curl -s http://127.0.0.1:3420/aif/info                                        # 坐标/朝向/生命
curl -s http://127.0.0.1:3420/aif/inventory                                   # 背包/副手/盔甲/熔炉/箱子
curl -s http://127.0.0.1:3420/aif/world                                       # 维度/时间/天气
curl -s http://127.0.0.1:3420/aif/msg                                         # 上次读之后的聊天回显
curl -s http://127.0.0.1:3420/aif/keysnd                                      # 上次读之后的重要声音
curl -s -X POST --data-binary 'bt mine iron_ore'          http://127.0.0.1:3420/ctl
curl -s -X POST --data-binary 'craft iron_pickaxe'        http://127.0.0.1:3420/op
curl -s -X POST --data-binary 'inventory torch 2'         http://127.0.0.1:3420/op
curl -s -X POST --data-binary 'look yaw 90'               http://127.0.0.1:3420/op
curl -s -o shot.png http://127.0.0.1:3420/ctl/prtsc
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
  所以把合成、冶炼、开箱子这些高频动作变成 `POST :3420/op` 的一条命令，直接发 `ServerboundContainerClickPacket`，
  和玩家亲手点格子完全等价，服务端照常校验（1.3.2 起从聊天指令 `/cmdop` 改成了 HTTP 接口，命令语法与报错不变）。
  `look` 再把「转视角」从鼠标像素换算变成一个精确命令：
  绝对角度直接给，相对角度写 `~`，改的就是 `/info` 里的 `yaw` / `pitch`。
* **unlockRecipe = 开局配方书（服务端，可选）**：cmdCraft 只认已经解锁的配方，新存档里配方书要自己攒；
  它让玩家一加入世界就拿到整本配方书（`/recipe give <玩家> *`），是 cmdCraft 在「新存档 / 新玩家」
  场景下的补丁，省掉了开作弊 + 第二个账号的绕路。
* **Baritone = 腿（可选）**：`bt mine` / `bt goto` 负责寻路和挖矿，比逐步按键高效得多。

## 快速开始

```bash
# 1. 需要 Minecraft 26.2 + Fabric Loader 0.19.5+
# 2. 必需的：MGHttpdProvider（3420 端口）+ mcctl
cp httpdprovider-*.jar mcctl-*.jar ~/.minecraft/mods/
# 3. 强烈建议一起装的三个（可选，但少了会难受）
cp advanced-info-fetch-*.jar craftcmd-*.jar ~/.minecraft/mods/   # Baritone 另见其仓库
# 4. 服务端模组：新存档里想直接用 cmdCraft 合成，就把它也放进 mods/（单人存档同样有效）
cp unlockrecipe-*.jar ~/.minecraft/mods/
# 5. 启动游戏、进入存档，然后：
curl http://127.0.0.1:3420/            # 当前可用的 endpoint 列表
curl http://127.0.0.1:3420/ctl/        # mcctl 使用说明
curl http://127.0.0.1:3420/op/         # cmdCraft 使用说明（装了 cmdCraft 才有）
curl http://127.0.0.1:3420/aif/info    # 玩家状态（装了 AdvancedInfoFetcher 才有）
```

## 手册

[`playbook.md`](playbook.md) 是真正交给模型的那份文档，只写「正式游玩时该怎么做」：

* 一个端口（3420：`/ctl` 控制 / `/aif` 信息 / `/op` 容器操作）的接口与字段（`GET :3420/`、`GET :3420/ctl/`、`GET :3420/aif/`、`GET :3420/op/` 全文的精简版）
* **状态与回显**：`/info`、`/inventory`、`/world` 的字段含义，`/msg` 的增量聊天回显
  （指令输出 / Baritone / 报错，读取即清空），以及 `/sound`·`/keysnd` 的增量声音回显
  （`<声音ID> <音量> <音高>`，每播放一次一行；`/keysnd` 与 `/sound` 共用队列，只给重要声音）
* **输入通道实测结论**：`E` 开背包、`1`-`9` 切格、`Q` 丢弃都通过 HTTP 有效；
  发文本一律走 `chat <文本>` / `bt <命令>`，不通过聊天框打字；
  聊天框开着时 `E`/`Q`/`1`-`9` 会自动先关掉它，`W` 等移动键不受影响（详见手册 §4）
* `cmdCraft` 五条 `POST /op` 子命令（`craft` / `inventory` / `furnace` / `chest` / `look`）的前置条件、参数与常用物品 id
* 「观察 → 决策 → 执行 → 校验」的循环节奏，以及截图延迟、对账、记坐标这些注意事项
* 固定分辨率下的坐标表（仅在**没有 cmdCraft**、必须手点时兜底：先 `GET :3420/ctl/mouse` 读光标，再 `mouse goto`）
* 指定种子与那座已激活的末地传送门，以及从空手到末影龙的路线

## 已验证

在真实客户端（Minecraft 26.2 + Fabric + Baritone）里跑通过，**平台为 Linux + X11/Xwayland**：

> **Linux 上的 Minecraft 永远跑在 X11/Xwayland 下**：26.2 在 `GLX` 里写死
> `glfwInitHint(GLFW_PLATFORM, GLFW_PLATFORM_X11)`，会话是 Wayland 也一样（走 Xwayland），
> 所以实际部署只有这一条路径。Wayland 相关的完整调查（怎么强制、为什么跑不起来、GLFW 各后端的差异）
> 见 [mcctl 的 `Wayland.md`](https://github.com/MineAgent/mcctl/blob/main/Wayland.md)。
> 光标相关的行为（`GET :3420/ctl/mouse` 的「不在窗口内」判断、`mouse goto` 的绝对定位）**只在 Linux 上实测过**；
> Windows、macOS 没有实机验证——接口本身是 GLFW 的跨平台接口，代码里没有平台分支，
> 但换平台后建议先自测一遍 `/mouse` → `mouse goto` → `mouse left`。

| 能力 | 结果 |
| --- | --- |
| 控制接口 | 移动 / 转向 / 视角 / 组合键，截图对比确认生效 |
| 视角指令 | `POST :3420/op 'look yaw 90'`、`'look pitch ~-5'` 之后 `/info` 的 `yaw` / `pitch` 与之一致；多人服务端 `data get entity … Rotation` 也是同一组值 |
| 光标接口 | `GET :3420/ctl/mouse` 在暂停菜单给出 `光标：427.0 240.0`、`抓取：否`、`界面：PauseScreen`；`mouse goto 321 202` + `mouse left` 点中「进度」按钮 |
| 光标在窗口外 | 把指针移到窗口外后 `/mouse` 输出 `光标：不在窗口内，请使用 mouse goto <x> <y>`（用跨平台的 `GLFW_HOVERED` 判断）；照常 `mouse goto` + `mouse left` 仍能点中，且物理指针被拉回窗口内 |
| 状态接口 | `/info` 的坐标、朝向、选中格与游戏内一致 |
| 世界接口 | `/world` 的维度、时间、天数、游戏刻和天气跟随游戏（`/time set` 后 `时间` 立即变化） |
| 聊天回显 | `/msg` 增量返回玩家聊天、指令输出、Baritone 输出与报错，逐条与客户端日志的 `[CHAT]` 一致，读完再次请求为空 |
| 声音回显 | `/sound` 增量返回播出音效的命名空间 ID、请求音量与音高；挖石头得到 `minecraft:block.stone.break`，未知/被静音跳过的音效不出现 |
| 重要声音 | `/keysnd` 过滤掉脚步/音乐/ambient/ui/天气，只留破坏方块、怪物、爆炸等；与 `/sound` 共用一个队列 |
| Baritone | `bt mine oak_log`、`bt mine stone`、`bt mine iron_ore` 自动寻路挖掘并拾取（实测把玩家从 y=48 带到 y=18） |
| 合成 | 原木 → 木板 → 工作台 → 木镐 → 石镐 |
| 冶炼 | `POST :3420/op 'furnace put raw raw_iron'`、`'furnace put fuel coal'`、`'furnace get product'` 取出铁锭 |
| 端到端 | 空手 → 原木 → 工作台 → 木镐 → 石镐 → 熔炉 → 铁矿 → 铁锭 → **铁镐**，全程只用 HTTP（`E` 开背包 + `POST :3420/op 'craft …'`，不点 GUI） |

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

文档里提到的四个模组是代码，仍按各自的许可分发——
[mcctl](https://github.com/MineAgent/mcctl)、[AdvancedInfoFetcher](https://github.com/MineAgent/AdvancedInfoFetcher)、
[cmdCraft](https://github.com/MineAgent/cmdCraft) 与 [unlockRecipe](https://github.com/MineAgent/unlockRecipe)
均为 **LGPL-3.0-only**。
