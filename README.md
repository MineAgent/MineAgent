# MineAgent

用 LLM 端到端游玩 Minecraft 的一套完整方案：**一个把游戏变成 HTTP 接口的模组 + 一组替代 GUI 操作的指令 + 一份写给模型看的操作手册。**

LLM 不需要看屏幕、不需要模拟器，只要会发 `curl`，就能从空手一路玩到击败末影龙。

## 组成

| 仓库 | 作用 |
| --- | --- |
| [**mcctl**](https://github.com/MineAgent/mcctl) | Fabric 客户端模组，在 `127.0.0.1:3420` 起一个 HTTP 服务：按键 / 鼠标 / 视角 / 滚轮 / Baritone / 聊天 / 截图 / 玩家信息 |
| [**cmdCraft**](https://github.com/MineAgent/cmdCraft) | Fabric 客户端模组，加两条聊天指令：`/craft` 合成、`/furnace` 冶炼 |
| [**playbook.md**](playbook.md) | 给模型看的操作手册：接口、实测结论、坐标表、流程、坑 |

三者都是客户端侧，**服务端不需要装任何东西**，可以在原版 / Fabric / Paper 服务器上用。

## 它是怎么"玩"的

一个循环：

```
GET  /info        我在哪、面朝哪、背包有什么
   ↓  决定下一步（挖矿 / 合成 / 赶路 / 冶炼）
POST /            发命令
   ↓
GET  /prtsc       看一眼画面，GET /info 对账
   ↓  回到第一步
```

实际的请求长这样：

```bash
curl -s http://127.0.0.1:3420/info                                          # 读状态
curl -s -X POST --data-binary 'bt mine iron_ore'   http://127.0.0.1:3420    # 让 Baritone 去挖
curl -s -X POST --data-binary 'chat /craft iron_pickaxe' http://127.0.0.1:3420
curl -s -o shot.png http://127.0.0.1:3420/prtsc                             # 截图
```

## 为什么是这三个部分

* **mcctl 是"身体"** —— 把游戏变成可编程接口。按键和鼠标走原版输入管线（`KeyMapping` / `Screen` 事件），视角旋转会正常同步给服务器，不抢占真实键鼠。
* **cmdCraft 是"手"** —— GUI 对 LLM 极不友好：光标起始位置读不到、点击有延迟、容易点错格子。所以把最高频的两件事变成指令，直接驱动 `ServerboundContainerClickPacket`，由服务端校验每一步。
* **Baritone 是"腿"** —— `bt mine` / `bt goto` 负责寻路与挖矿，比逐步按键高效得多。

## 快速开始

```bash
# 1. 需要 Minecraft 26.2 + Fabric Loader 0.19.5+
# 2. 把两个模组丢进 .minecraft/mods/
cp mcctl-*.jar cmdCraft-*.jar ~/.minecraft/mods/
# 3. 建议同时装 Baritone（mcctl 的 bt 命令会调用它的 API）
# 4. 启动游戏、进入存档，然后：
curl http://127.0.0.1:3420/        # 能打印出使用说明就通了
curl http://127.0.0.1:3420/info    # 玩家 / 维度 / 坐标 / 朝向 / 背包
```

## 手册

[`playbook.md`](playbook.md) 是真正交给模型的那份文档，只写"正式游玩时该怎么做"：

* 三个接口的用法与 `/info` 字段含义
* **输入通道实测结论**：哪些键通过 HTTP 无效（`E` 开背包、`1`-`9` 切格、`Q`、`T`），以及怎么绕开
* `/craft`、`/furnace` 的前置条件、参数与常用物品 id
* 固定分辨率下的坐标表与转向换算
* 从空手到末影龙的路线

## 已验证

在真实客户端（Minecraft 26.2 + Fabric + Baritone）里跑通过：

| 能力 | 结果 |
| --- | --- |
| `GET /` `GET /info` `GET /prtsc` | 正常返回，`/info` 与游戏内一致 |
| 移动 / 转向 / 视角 | `W 1500`、`mouse move +300 -60` 截图对比确认生效 |
| Baritone | `bt mine oak_log`、`bt mine stone`、`bt mine iron_ore` 自动寻路挖掘并拾取（实测把玩家从 y=48 带到 y=18） |
| 合成 | 原木 → 木板 → 工作台 → 木镐 → 石镐 |
| 冶炼 | `/furnace put raw raw_iron`、`put fuel`、`get product` 取出铁锭 |
| 端到端 | 空手 → 原木 → 工作台 → 木镐，全部由 HTTP 请求驱动完成 |

目标是把这条链走完：**挖矿 → 铁器 → 钻石 → 黑曜石 → 下界烈焰棒 → 末影珍珠 → 末地传送门 → 末影龙。**

## 许可证

[mcctl](https://github.com/MineAgent/mcctl) 与 [cmdCraft](https://github.com/MineAgent/cmdCraft) 均为 **LGPL-3.0-only**。
