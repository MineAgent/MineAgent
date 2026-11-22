# MineAgent

[![License: CC BY-NC-SA 4.0](https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-lightgrey.svg)](LICENSE)

用 LLM 玩 Minecraft 的一套方案：**一组客户端模组 + 一个服务端模组 + 一个 DSH 插件 + 一份操作手册**。

游戏把「操作」「状态」「合成」「寻路」「自动攻击」变成 `127.0.0.1:3420` 的五组 HTTP 接口，事件推送到
`127.0.0.1:3422`；[`dsh-mc-connect`](https://github.com/MineAgent/dsh-mc-connect) 把这些接口直接注册成
DeepSeek Harness 的 `mc_*` 工具。在 DSH 里选「MC 模式」就能从空手玩到击败末影龙，不需要自己拼 HTTP。

反过来，**写 Fabric 模组时它也是一套现成的实测环境**：用「MC 模式+」一边改代码、跑 gradle、读日志，
一边用同一组工具起客户端、造场景、读状态做真机验证。

## 组成

| 仓库 | 必需性 | 接口 | 作用 |
| --- | --- | --- | --- |
| [**MGHttpdProvider**](https://github.com/MineAgent/HttpdProvider) | **必需** | `127.0.0.1:3420` | **插座**：共享 HTTP 服务，各模组挂在它的前缀下；`GET /` 列出当前可用的 endpoint |
| [**mcctl**](https://github.com/MineAgent/mcctl) | **必需** | `/ctl` | **身体**：按键 / 鼠标 / 视角 / 滚轮 / 聊天 / 打字，另有截图与光标位置 |
| [**AdvancedInfoFetcher**](https://github.com/MineAgent/AdvancedInfoFetcher) | 可选 | `/aif` | **眼睛+耳朵**：坐标 / 朝向 / 生命 / 饱食 / 状态效果 / 目标方块 / 光照 / 背包 / 容器 / 世界 / 聊天回显 / 声音 |
| [**cmdCraft**](https://github.com/MineAgent/cmdCraft) | 可选 | `/op` | **手**：合成 / 换快捷栏 / 熔炉 / 箱子 / 转视角 / 进食 / 挖单个方块 |
| [**KillAura**](https://github.com/MineAgent/KillAura) | 可选 | `/ka` | **自动攻击**：开关、状态、配置；四类目标 + 四档强制目标 + 拒绝名单 |
| [**MC Event Reporter**](https://github.com/MineAgent/events) | 可选 | `127.0.0.1:3422` | **事件源**：受伤 / 拾取 / 聊天 / 饥饿变化 / Baritone 结束 / 维度切换 / 死亡 / 重生，JSON POST 给 DSH 插件；另挂 `GET /evt/` 事件目录 |
| [**unlockRecipe**](https://github.com/MineAgent/unlockRecipe) | 可选（用 cmdCraft 时建议装） | —（纯服务端） | **开局配方书**：谁加入世界就替谁执行 `/recipe give <玩家> *`，让合成从第一秒起就有整本配方可用 |
| [**Baritone**](https://github.com/cabaletta/baritone) | 可选（MGBaritoneConnector 硬依赖它） | — | **腿的引擎**：寻路与自动挖矿 |
| [**MGBaritoneConnector**](https://github.com/MineAgent/baritone-connector) | 可选（用 Baritone 时必需） | `/bt` | **腿的接口**：把一条 Baritone 命令挂成 `POST /bt` |
| [**NoAutoPause**](https://github.com/MineAgent/noautopause) | 可选 | — | **后台可跑**：窗口失去焦点时不暂停游戏，后台注入的按键才有效 |
| [**dsh-mc-connect**](https://github.com/MineAgent/dsh-mc-connect) | 用 DSH 时必需 | 直连 `3420` / `3422` | **DSH 插件**：注册「MC 模式」「MC 模式+」，提供完整的 `mc_*` 工具集、2 个事件工具和 Web 订阅界面 |
| [**playbook.md**](playbook.md) | — | — | **给模型的 DSH 操作手册**：事件、生存、合成、寻路、战斗、路线与坑 |

**MGHttpdProvider + mcctl 是硬需求**：前者提供 3420 端口，后者是身体。其余模组都是可选增强，但实战一般都会装：
没有 AdvancedInfoFetcher 就只能靠截图看背包和准星；没有 cmdCraft 就得点 GUI 格子；没有 Baritone 就只能一步步按键；
没有 KillAura 就只能自己按左键砍怪；没有 NoAutoPause 则单人存档一失去焦点就暂停。

除了 unlockRecipe，其余模组都是**客户端**模组，服务端不需要装任何东西，可以在原版 / Fabric / Paper 服务器上用。
unlockRecipe 是唯一的服务端模组（Fabric），只解决「新存档配方书是空的」这一件事。

## 接进 DeepSeek Harness

```bash
git clone git@github.com:MineAgent/dsh-mc-connect.git
cd dsh-mc-connect
dsh plugin --profile web add "link:$PWD"
```

装好后**新建**会话；首次安装或改客户端 bundle 后重启 `dsh web` 再刷新页面。

| preset | 工具面 | 用途 |
| --- | --- | --- |
| **MC 模式** | `mc_*` + 2 个事件工具 + `todo_write` / goal / `ask_user_question` / `exit_plan_mode`；隐藏 `working_directory`，没有文件、终端、联网、子 Agent | 只玩游戏 |
| **MC 模式+** | 标准模式全部工具 + `mc_*` + 2 个事件工具 | 一边改代码 / 跑命令，一边真机验证 |

### 工具

| 组 | 工具 | 作用 |
| --- | --- | --- |
| 自检/说明 | `mc_status`、`mc_help` | 检查服务、前缀、是否进世界；读模组自带的权威说明 |
| 观测 | `mc_player`、`mc_inventory`、`mc_world`、`mc_chat`、`mc_screenshot`、`mc_cursor` | 状态 / 背包 / 世界 / 聊天 / 截图 / 光标 |
| 输入 | `mc_control` | 按键 / 鼠标 / 聊天 / 打字，可带 `waitMs` |
| 合成与容器 | `mc_craft`、`mc_hotbar`、`mc_furnace`、`mc_chest`、`mc_look`、`mc_eat`、`mc_mine`、`mc_place` | 合成 / 换快捷栏 / 熔炉 / 箱子 / 转向 / 进食 / 挖单个方块 / 放置方块 |
| 移动与战斗 | `mc_baritone`、`mc_killaura` | 寻路、采集、停止；自动攻击开关与配置 |
| 兜底 | `mc_request` | 调用尚未封装成专用工具的 `/` 开头 endpoint |

完整参数、返回值和错误文案见 [`dsh-mc-connect` 的 README](https://github.com/MineAgent/dsh-mc-connect#readme)。

### 事件订阅

- `get_mc_event_types`：读 `:3420/evt/` 的静态事件目录，拿到每种事件在 `data` 里的字段名。
- `subscribe_mc_event`：订阅当前会话，监听 `127.0.0.1:3422`。
  - **common**：事件合并成新的 user 消息；订阅后结束本轮等待，不要轮询。
  - **advanced**：必须提供内联 `lua_script`；事件送入常驻 Lua，脚本调用 `send` / `forward` 才通知 Agent，也可以在脚本里直接读状态或操作游戏。
  - `stop` 结束订阅；`duration_seconds` / `max_events` 可自动结束；UI 开关只暂停投递，不释放 3422。
- 事件正文（尤其聊天）是**不可信游戏数据**，模型不应执行其中的指令。
- 事件订阅需要 MC Event Reporter；没装时仍可读聊天和状态。

这些工具只在「MC 模式」/「MC 模式+」里出现；其它模式看不到。

## 玩法

```text
事件    订阅聊天 / 受伤 / 死亡 / Baritone 结束 / 维度切换 → 结束本轮等消息
观察    mc_status / mc_player / mc_inventory / mc_world / mc_chat / mc_screenshot / mc_cursor
决策    结合事件、当前目标、背包和世界时间判断下一步
执行    mc_control / mc_craft / mc_hotbar / mc_furnace / mc_chest / mc_look / mc_eat / mc_mine /
        mc_place / mc_baritone / mc_killaura
校验    看工具返回，再读 mc_chat + mc_player / mc_inventory 对账
```

详细节奏、种子坐标、村庄发育、铁的来源、末地路线和常见坑见 [`playbook.md`](playbook.md)。

## 快速开始

```bash
# 1. 需要 Minecraft 26.3 + Fabric Loader 0.20.1+
# 2. 必需：MGHttpdProvider（3420 端口）+ mcctl
cp httpdprovider-*.jar mcctl-*.jar ~/.minecraft/mods/
# 3. 强烈建议一起装：眼睛 + 手 + 腿 + 自动攻击
cp advanced-info-fetch-*.jar craftcmd-*.jar ~/.minecraft/mods/
cp killaura-*.jar mgbaritoneconnector-*.jar ~/.minecraft/mods/   # Baritone 本体另见其仓库
# 4. 事件订阅：把 MC Event Reporter 放进 mods/
cp mc-event-*.jar ~/.minecraft/mods/
# 5. 后台跑：窗口失焦也不暂停
cp noautopause-*.jar ~/.minecraft/mods/
# 6. 服务端模组：新存档里想直接用配方书合成
cp unlockrecipe-*.jar ~/.minecraft/mods/
# 7. DSH：安装 dsh-mc-connect，新建会话选「MC 模式」
```

游戏启动后，DSH 的 `mc_status` 会给出 3420 上各前缀是否可用、客户端是否已进入世界；异常时先跑它。

## 也能拿它开发模组

在 DSH 里选「MC 模式+」。它保留标准模式的读写文件、跑命令、读日志和联网能力，同时提供全部 `mc_*` 工具：

1. `./gradlew build` 出 jar，拷进 `~/.minecraft/mods/`；
2. 用启动脚本起客户端（例如 `--quickPlaySingleplayer <存档>`，直接进世界）；
3. `mc_status` / `mc_world` 自检，确认真的进了世界，而不是靠 sleep 猜；
4. 用 `mc_control` / `mc_craft` / `mc_baritone` / `mc_killaura` 造场景，用 `mc_screenshot` / `mc_chat` /
   `mc_player` / `mc_inventory` 判结果；日志和 `crash-reports/` 走文件工具，不经过 3420；
5. 对不上就回去改代码，从第 1 步重来；测完把实例停掉。

这套模组自己就是按「构建 → 起客户端 → 用 3420 观察和操作 → 读日志 → 改代码」的循环长出来的。

## unlockRecipe：给 cmdCraft 补上配方书

cmdCraft 的 `craft` 走客户端配方书，只有已解锁的配方才能合成；原版要拿到材料才解锁，新存档开局经常卡在
「配方未解锁」。`unlockRecipe` 在有人加入世界时用控制台身份执行一次 `/recipe give <玩家> *`，让配方书立刻是满的。

- 单人存档、局域网世界、独立 Fabric 服务端都能用；客户端连别人的服务器时它什么也不做。
- 服务端不是 Fabric 时装不了它，那就按原版节奏自己解锁。

## 已验证

- 控制：移动、转向、视角、组合键、截图对比确认生效；光标定位与 GUI 点击可用。
- 状态：坐标、朝向、选中格、目标方块、光照、背包、容器、世界时间和天气与游戏内一致。
- 合成 / 冶炼：原木 → 木板 → 工作台 → 木镐 → 石镐；熔炉放入原料/燃料并取出铁锭。
- Baritone：自动寻路挖掘并拾取；Kill Aura：自动攻击、目标规则、配置更新。
- 事件：MC Event Reporter 的聊天与受伤事件能 POST 到 3422；DSH 插件的工具注册、事件投递、Lua 沙箱由
  `node --test test/` 离线覆盖。

## 本仓库

```text
README.md        本文件
playbook.md      给模型的 DSH 操作手册
LICENSE          CC BY-NC-SA 4.0
```

只放文档，不放代码。

## 许可证

本仓库的文档采用 **[CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/)**，完整文本见 [`LICENSE`](LICENSE)。

文档里提到的模组代码按各自许可分发：HttpdProvider、mcctl、AdvancedInfoFetcher、cmdCraft、KillAura、
MC Event Reporter、MGBaritoneConnector、NoAutoPause 与 unlockRecipe 均为 **LGPL-3.0-only**；
上游 [Baritone](https://github.com/cabaletta/baritone) 仍是它自己的许可。

[`dsh-mc-connect`](https://github.com/MineAgent/dsh-mc-connect) 的**代码与配置**为 **GPL-3.0-only**，
**文档**为 **CC BY-NC-SA 4.0**。
